# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

**Feature:** Webhook de Notificação de Pedidos
**Versão:** 1.0.0
**Data:** 2026-07-14
**Estimativa:** 3 sprints

---

## 1. Contexto e Motivação Técnica

O Order Management System (OMS) atual não possui mecanismo de notificação externa. Clientes B2B dependem de polling via `GET /api/v1/orders` para detectar mudanças de status, gerando carga desnecessária e latência percebida elevada. A feature de webhooks resolve isso empurrando eventos para endpoints cadastrados pelos clientes assim que ocorrem mudanças de status.

Do ponto de vista técnico, o desafio é introduzir notificações assíncronas em uma aplicação síncrona (REST API com Express), sem degradar a operação existente e sem introduzir inconsistências entre estado de negócio e notificações. A solução adota o padrão Transactional Outbox, acoplado à transação existente do `changeStatus`, e um worker separado para envio HTTP.

---

## 2. Objetivos Técnicos

1. **Atomicidade:** evento de webhook registrado se e somente se a transação de mudança de status commitar.
2. **Desacoplamento:** envio HTTP não bloqueia nem impacta a latência da API de pedidos.
3. **Resiliência:** retry automático com backoff para falhas transitórias; DLQ para falhas permanentes.
4. **Segurança:** assinatura HMAC-SHA256 por endpoint; TLS obrigatório; secret rotacionável.
5. **Rastreabilidade:** cada evento tem ID único; log estruturado em todas as etapas.
6. **Consistência com o código existente:** reuso de AppError, Pino, Zod, padrão de módulos, middlewares de auth e erro.

---

## 3. Escopo e Exclusões

### Dentro do escopo

- CRUD de configuração de endpoints de webhook por customer
- Geração e rotação de secrets HMAC-SHA256
- Inserção atômica de eventos na outbox durante `changeStatus`
- Worker em polling (2s) para envio HTTP
- Retry com backoff (1m/5m/30m/2h/12h, 5 tentativas)
- DLQ com endpoint admin de replay
- Histórico de deliveries (últimos 100) por endpoint
- Filtro de eventos por status na inscrição do webhook
- Limite de payload de 64KB

### Fora de escopo (explicitamente)

- **Notificação por email** ao cliente sobre falhas de webhook — adiado para fase futura ([09:37] Larissa)
- **Dashboard visual** para gestão de webhooks — projeto separado do time de frontend ([09:40] Larissa)
- **Rate limiting de saída** — observar primeiro, implementar se necessário ([09:39] Diego)
- **Arquivamento automático** de eventos entregues após 30 dias — previsto, mas fora do escopo inicial ([09:08] Diego)
- **Escala horizontal do worker** — single worker é suficiente para o volume atual ([09:13] Larissa)

---

## 4. Fluxos Detalhados

### 4.1 Criação do evento na outbox (dentro de `changeStatus`)

**Trigger:** `PATCH /api/v1/orders/:id/status`

**Fluxo:**

```
1. OrderController.changeStatus recebe a requisição
2. OrderService.changeStatus inicia transação Prisma ($transaction)
3. Dentro da transação:
   a. Valida transição (canTransition)
   b. Se aplicável, debita ou repõe estoque
   c. UPDATE订单 SET status = toStatus
   d. INSERT INTO order_status_history
   e. [NOVO] Para cada webhook_endpoint ativo do customer que aceita o toStatus:
      - Gera event_id (UUID v4)
      - Renderiza payload JSON (snapshot)
      - Calcula next_retry_at = NOW()
      - INSERT INTO webhook_outbox (id, event_id, endpoint_id, order_id, customer_id,
        event_type, payload, status='PENDING', attempt=0, next_retry_at=NOW())
   f. Se a inserção na outbox falhar, toda a transação dá rollback
4. Resposta HTTP 200 com o pedido atualizado
```

**Ponto de atenção:** a consulta aos `webhook_endpoints` ativos do customer deve ser feita dentro da transação para evitar race condition com desativação de endpoint, mas como endpoints são raramente desativados, um cache de curta duração pode ser usado para evitar o JOIN extra em toda mudança de status. A versão inicial fará a consulta dentro da transação; otimização com cache é melhoria futura.

**Arquivo impactado:** `src/modules/orders/order.service.ts:126-179` — método `changeStatus`

### 4.2 Processamento pelo worker

**Entry point:** `src/worker.ts`, script `npm run worker`

**Loop principal (pseudocódigo):**

```
while (true) {
  const batch = await prisma.webhookOutbox.findMany({
    where: {
      status: 'PENDING',
      next_retry_at: { lte: new Date() }
    },
    orderBy: { created_at: 'asc' },
    take: BATCH_SIZE (20)
  })

  for (const event of batch) {
    await processEvent(event)
  }

  await sleep(2000)
}
```

**`processEvent(event)` — algoritmo:**

```
1. Marca evento como PROCESSING (UPDATE status = 'PROCESSING')
2. Busca webhook_endpoint associado
3. Se endpoint inativo, marca como FAILED com reason='endpoint_disabled'
4. Monta headers HTTP:
   - X-Event-Id: event.event_id
   - X-Signature: HMAC-SHA256(payload, endpoint.secret)
   - X-Timestamp: ISO 8601 now
   - X-Webhook-Id: endpoint.id
   - Content-Type: application/json
5. Envia POST para endpoint.url com timeout de 10s
6. Se sucesso (2xx):
   a. INSERT INTO webhook_delivery (event_id, endpoint_id, status='SUCCESS',
      response_status, response_body, duration_ms)
   b. UPDATE webhook_outbox SET status='DELIVERED', delivered_at=NOW()
7. Se falha (não-2xx, timeout, erro de rede):
   a. Se attempt < 5:
      - Calcula next_retry_at com backoff (1m/5m/30m/2h/12h)
      - UPDATE webhook_outbox SET status='PENDING', attempt++,
        next_retry_at=<calculado>, last_error=<erro>
   b. Se attempt >= 5:
      - INSERT INTO webhook_dead_letter (event_id, endpoint_id, payload,
        last_error, failed_at)
      - UPDATE webhook_outbox SET status='FAILED'
   c. INSERT INTO webhook_delivery (event_id, endpoint_id, status='FAILED',
      response_status, error_message, duration_ms)
```

### 4.3 Retry com backoff

| Tentativa | Intervalo após falha | Timestamp acumulado (aproximado) |
|-----------|---------------------|-----------------------------------|
| 1 (envio inicial) | imediato | t+0 |
| 2 | +1 minuto | t+1m |
| 3 | +5 minutos | t+6m |
| 4 | +30 minutos | t+36m |
| 5 | +2 horas | t+2h36m |
| DLQ | +12 horas | t+14h36m |

O campo `next_retry_at` na outbox controla quando o evento volta a ser elegível para processamento. O worker só busca eventos com `next_retry_at <= NOW()`.

### 4.4 Dead Letter Queue (DLQ)

- Tabela: `webhook_dead_letter`
- Colunas: `id`, `event_id`, `endpoint_id`, `order_id`, `customer_id`, `payload` (JSON), `last_error` (text), `attempts` (int), `failed_at` (timestamp), `created_at`
- Reprocessamento: `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
  - Valida role `ADMIN` via `requireRole('ADMIN')`
  - Busca registro na DLQ
  - Reinsere na `webhook_outbox` com `status='PENDING'`, `attempt=0`, `next_retry_at=NOW()`
  - Remove da DLQ (ou marca como `REPLAYED`)
  - Loga userId que fez o replay para auditoria

### 4.5 Filtro de eventos por status

Na criação do webhook endpoint, o campo `events` (array de `OrderStatus`) define quais status o cliente quer receber. Exemplo: `["SHIPPED", "DELIVERED"]`. Durante a inserção na outbox (passo 3e do fluxo 4.1), o sistema verifica se o `toStatus` da transição está na lista de eventos do endpoint. Se nenhum endpoint do customer estiver inscrito naquele status, nenhuma linha é inserida na outbox para aquele customer.

---

## 5. Contratos Públicos

### 5.1 Criar Webhook Endpoint

**`POST /api/v1/webhooks`**

**Headers:** `Authorization: Bearer <JWT>`

**Request Body:**
```json
{
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://api.cliente.com.br/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`:**
```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://api.cliente.com.br/webhooks/orders",
  "secret": "whsec_xK9mP2vL8nQ4wR7y",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-07-14T10:30:00.000Z",
  "updatedAt": "2026-07-14T10:30:00.000Z"
}
```

**Erros possíveis:** `400` (URL inválida/http), `404` (Customer não encontrado), `401`/`403` (autenticação)

### 5.2 Listar Webhooks de um Customer

**`GET /api/v1/webhooks?customerId=<uuid>`**

**Headers:** `Authorization: Bearer <JWT>`

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "url": "https://api.cliente.com.br/webhooks/orders",
      "events": ["PAID", "SHIPPED"],
      "active": true,
      "createdAt": "2026-07-14T10:30:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

### 5.3 Atualizar Webhook Endpoint

**`PATCH /api/v1/webhooks/:id`**

**Headers:** `Authorization: Bearer <JWT>`

**Request Body:**
```json
{
  "url": "https://api.cliente.com.br/webhooks/v2/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `200 OK`:**
```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "url": "https://api.cliente.com.br/webhooks/v2/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-14T11:00:00.000Z"
}
```

### 5.4 Rotacionar Secret

**`POST /api/v1/webhooks/:id/rotate-secret`**

**Headers:** `Authorization: Bearer <JWT>`

**Response `200 OK`:**
```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "secret": "whsec_nE8wR3tY6uI1oP5aS",
  "previousSecretValidUntil": "2026-07-15T11:00:00.000Z"
}
```

**Regras:**
- Gera nova secret (UUID sem hífens + prefixo `whsec_`)
- Armazena secret anterior em `previous_secret` e timestamp em `secret_rotated_at`
- Durante 24h após `secret_rotated_at`, ambas as secrets são válidas para verificação
- Após 24h, `previous_secret` é limpo e só a nova secret é aceita

### 5.5 Remover Webhook Endpoint

**`DELETE /api/v1/webhooks/:id`**

**Response `204 No Content`**

Eventos pendentes na outbox para este endpoint continuam existindo, mas não serão processados (worker verifica `active` antes de enviar). Alternativa: ao deletar, marcar todos os eventos pendentes do endpoint como `FAILED` com reason `endpoint_deleted`.

### 5.6 Histórico de Deliveries

**`GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20`**

**Response `200 OK`:**
```json
{
  "data": [
    {
      "id": "d4e5f6a7-b8c9-0123-def4-567890abcdef",
      "eventId": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
      "status": "SUCCESS",
      "responseStatus": 200,
      "durationMs": 342,
      "createdAt": "2026-07-14T10:35:00.000Z"
    },
    {
      "id": "f6a7b8c9-d0e1-2345-f678-90abcdef0123",
      "eventId": "a7b8c9d0-e1f2-3456-7890-abcdef012345",
      "status": "FAILED",
      "responseStatus": null,
      "errorMessage": "Connection timeout after 10000ms",
      "durationMs": 10002,
      "createdAt": "2026-07-14T10:32:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

### 5.7 Reprocessar DLQ (Admin)

**`POST /api/v1/admin/webhooks/dead-letter/:id/replay`**

**Headers:** `Authorization: Bearer <JWT>` (requer role ADMIN)

**Response `200 OK`:**
```json
{
  "message": "Event replayed successfully",
  "eventId": "a7b8c9d0-e1f2-3456-7890-abcdef012345",
  "webhookEndpointId": "b2c3d4e5-f6a7-8901-bcde-f12345678901"
}
```

**Erro `404`:** Evento não encontrado na DLQ
**Erro `403`:** Role diferente de ADMIN

### 5.8 Payload do Webhook (enviado ao cliente)

**`POST <cliente_url>`**

**Headers:**
```
Content-Type: application/json
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
X-Signature: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0
X-Timestamp: 2026-07-14T10:35:00.123Z
X-Webhook-Id: b2c3d4e5-f6a7-8901-bcde-f12345678901
```

**Request Body:**
```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "order.status_changed",
  "timestamp": "2026-07-14T10:35:00.123Z",
  "data": {
    "orderId": "c3d4e5f6-a7b8-9012-cdef-123456789012",
    "orderNumber": "ORD-000042",
    "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "fromStatus": "PROCESSING",
    "toStatus": "SHIPPED",
    "totalCents": 15990
  }
}
```

### 5.9 Contrato de verificação de assinatura (para clientes)

O cliente deve verificar a assinatura da seguinte forma:

```
assinatura_recebida = request.headers['X-Signature']
payload_bytes = request.body (raw bytes)
assinatura_calculada = HMAC-SHA256(payload_bytes, secret)
comparação segura = timing_safe_compare(assinatura_recebida, assinatura_calculada)
```

Exemplo em Node.js:
```js
const crypto = require('crypto');
const received = req.headers['x-signature'];
const computed = crypto
  .createHmac('sha256', webhookSecret)
  .update(JSON.stringify(req.body))
  .digest('hex');
if (crypto.timingSafeEqual(Buffer.from(received), Buffer.from(computed))) {
  // assinatura válida
}
```

---

## 6. Matriz de Erros Previstos

Todos os erros do módulo de webhooks usam o prefixo `WEBHOOK_` e estendem `AppError` (`src/shared/errors/app-error.ts`).

| Código | HTTP Status | Mensagem | Quando ocorre |
|--------|-------------|----------|---------------|
| `WEBHOOK_NOT_FOUND` | 404 | `Webhook endpoint not found` | GET/PATCH/DELETE com id inexistente |
| `WEBHOOK_INVALID_URL` | 400 | `Webhook URL must use HTTPS` | URL cadastrada usa `http://` |
| `WEBHOOK_URL_INVALID_FORMAT` | 400 | `Invalid webhook URL format` | URL não é URL válida (Zod) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `Secret is required for webhook endpoint` | Tentativa de criar endpoint sem secret (não deve ocorrer, secret é gerada) |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `Customer not found` | customerId informado não existe |
| `WEBHOOK_MAX_ENDPOINTS` | 422 | `Customer has reached maximum number of webhook endpoints` | Limite de endpoints por customer excedido (ex: 10) |
| `WEBHOOK_EVENT_EMPTY` | 400 | `At least one event type must be specified` | Array `events` vazio |
| `WEBHOOK_INVALID_EVENT` | 400 | `Invalid event type: <value>` | Status no array `events` não é um `OrderStatus` válido |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 | `Webhook payload exceeds 64KB limit` | Payload serializado > 65536 bytes |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | `Dead letter event not found` | Replay de evento inexistente na DLQ |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `Event has already been replayed` | Tentativa de replay de evento já reprocessado |
| `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | `Delivery not found` | GET de delivery específico inexistente |

---

## 7. Estratégias de Resiliência

| Mecanismo | Configuração | Descrição |
|-----------|-------------|-----------|
| **Timeout HTTP** | 10 segundos | Conexão + resposta do cliente. Após 10s, trata como falha e agenda retry. |
| **Retry com backoff** | 1m/5m/30m/2h/12h, 5 tentativas | Backoff exponencial progressivo. `next_retry_at` controla elegibilidade. |
| **DLQ** | Tabela `webhook_dead_letter` | Eventos que excederam 5 tentativas. Não poluem a outbox principal. |
| **Idempotência** | `X-Event-Id` UUID v4 | Cliente deduplica pelo event_id. Worker nunca remove eventos da outbox antes de confirmar entrega. |
| **TLS obrigatório** | Validação Zod no schema de criação | URLs `http://` são rejeitadas na camada de validação (400). |
| **Grace period de secret** | 24 horas | Durante rotação, ambas as secrets são aceitas. Evita falsos positivos de falha. |
| **Healthcheck do worker** | Endpoint `GET /health` no worker | Permite monitoramento externo (Docker healthcheck). |
| **Graceful shutdown** | SIGTERM/SIGINT | Worker termina o batch atual antes de sair. |
| **Payload size limit** | 64KB (65536 bytes) | Eventos com payload maior são rejeitados com erro `WEBHOOK_PAYLOAD_TOO_LARGE`. |

---

## 8. Observabilidade

### 8.1 Métricas (expostas via endpoint `/metrics` ou logs estruturados)

| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `webhook_outbox_size` | Gauge | Número de eventos pendentes na outbox |
| `webhook_delivery_latency_ms` | Histogram | Latência de entrega (inserção na outbox → confirmação do cliente) |
| `webhook_delivery_success_rate` | Counter | Taxa de sucesso (2xx) vs. falha |
| `webhook_delivery_duration_ms` | Histogram | Duração da chamada HTTP individual |
| `webhook_retry_count` | Counter | Número de retries executados |
| `webhook_dlq_size` | Gauge | Tamanho atual da DLQ |
| `webhook_worker_loop_duration_ms` | Histogram | Duração de cada ciclo de polling |
| `webhook_hmac_validation_errors` | Counter | Falhas de validação HMAC reportadas por clientes (se houver callback) |

### 8.2 Logs

Logger Pino (`src/shared/logger/index.ts`) com log estruturado em JSON. Eventos chave:

| Evento | Nível | Campos |
|--------|-------|--------|
| `webhook_event_created` | info | `event_id`, `order_id`, `to_status`, `endpoint_id` |
| `webhook_delivery_attempt` | info | `event_id`, `attempt`, `endpoint_url` |
| `webhook_delivery_success` | info | `event_id`, `duration_ms`, `response_status` |
| `webhook_delivery_failure` | warn | `event_id`, `attempt`, `error_message`, `duration_ms` |
| `webhook_dlq_inserted` | warn | `event_id`, `endpoint_id`, `last_error` |
| `webhook_dlq_replay` | info | `event_id`, `replayed_by_user_id` |
| `webhook_secret_rotated` | info | `endpoint_id`, `rotated_by_user_id` |
| `webhook_worker_started` | info | `pid` |
| `webhook_worker_stopped` | info | `signal` |
| `webhook_payload_too_large` | error | `order_id`, `payload_size_bytes` |

### 8.3 Tracing

O header `X-Request-Id` já é gerado pelo `requestLogger` middleware (`src/middlewares/request-logger.middleware.ts`). Para o worker, cada evento carrega um `event_id` (UUID) que serve como identificador de trace. Logs do worker incluem `event_id` para correlação. Se tracing distribuído (OpenTelemetry) for adicionado no futuro, o `event_id` é propagado como span attribute.

---

## 9. Dependências e Compatibilidade

### Dependências existentes (já no `package.json`)

| Dependência | Uso no módulo de webhooks |
|-------------|---------------------------|
| `express` 4.21.1 | Rotas HTTP do CRUD de webhooks |
| `@prisma/client` 5.22.0 | Acesso a dados (outbox, endpoints, DLQ, deliveries) |
| `pino` 9.5.0 | Log estruturado no worker e nas rotas |
| `zod` 3.23.8 | Validação de schemas (criação/edição de endpoints) |
| `uuid` 11.0.3 | Geração de `event_id` e secrets |
| `jsonwebtoken` 9.0.2 | Autenticação (reuso do middleware `authenticate`) |
| `bcrypt` 5.1.1 | (não usado diretamente, mas presente) |

### Novas dependências? Nenhuma.

HMAC-SHA256 usa o módulo nativo `crypto` do Node.js. Nenhuma lib adicional é necessária.

### Compatibilidade

- **Node.js:** >=20 (mesmo requisito do projeto)
- **MySQL:** mesmo banco existente. As tabelas `webhook_outbox`, `webhook_endpoints`, `webhook_dead_letter` e `webhook_delivery` são criadas via migration Prisma.
- **Schema Prisma:** novas models adicionadas ao `prisma/schema.prisma`. Migration não afeta tabelas existentes.

---

## 10. Critérios de Aceite Técnicos

1. **Atomicidade:** um teste de integração deve demonstrar que, se a transação de `changeStatus` der rollback (ex: `InsufficientStockError`), NENHUMA linha é inserida em `webhook_outbox`.
2. **Desacoplamento:** a latência do `PATCH /orders/:id/status` não deve aumentar em mais de 5ms (p95) em relação ao baseline atual — medido com e sem a inserção na outbox.
3. **Retry e backoff:** um teste com endpoint HTTP mockado deve demonstrar que o worker retenta exatamente 5 vezes com os intervalos corretos (tolerância de ±10% no tempo) e move para DLQ após a 5ª falha.
4. **DLQ replay:** `POST /admin/webhooks/dead-letter/:id/replay` deve reinserir o evento na outbox, e o worker deve processá-lo no próximo ciclo.
5. **HMAC-SHA256:** um teste unitário deve calcular a assinatura sobre um payload conhecido e comparar com um valor esperado (teste de vetor). O header `X-Signature` deve estar presente em todas as requisições de webhook.
6. **Rotação de secret:** um teste deve verificar que, durante o grace period de 24h, ambas as secrets (atual e anterior) são aceitas para assinatura.
7. **Filtro de eventos:** ao criar um endpoint com `events: ["SHIPPED"]`, uma transição para `PAID` não deve gerar linha na outbox.
8. **Validação de URL:** `POST /webhooks` com `url: "http://..."` deve retornar 400 com código `WEBHOOK_INVALID_URL`.
9. **Role ADMIN obrigatório:** `POST /admin/webhooks/dead-letter/:id/replay` sem role ADMIN deve retornar 403.
10. **Payload size limit:** um evento cujo payload serializado exceda 64KB deve gerar erro `WEBHOOK_PAYLOAD_TOO_LARGE` e não ser inserido na outbox.

---

## 11. Integração com o Sistema Existente

Esta seção descreve como o módulo de webhooks se integra com componentes específicos do código base existente.

### 11.1 `src/modules/orders/order.service.ts:126-179` — `changeStatus`

**Integração:** o método `changeStatus` será estendido para receber uma função de enqueue de evento (`publishWebhookEvent`) como dependência ou parâmetro. Dentro da transação Prisma (linha 131), após o `INSERT` em `order_status_history` (linhas 158-167) e antes do `findUnique` de refresh (linhas 169-177), a função `publishWebhookEvent` será chamada passando o `tx` (transaction client), o `order`, `fromStatus` e `toStatus`.

**Forma proposta:**

```typescript
// Nova assinatura do construtor ou parâmetro
constructor(
  private readonly orders: OrderRepository,
  private readonly prisma: PrismaClient,
  private readonly publishWebhookEvent?: PublishWebhookEventFn, // opcional para backward compat
) {}
```

A função `publishWebhookEvent` consulta `webhook_endpoints` ativos do `order.customerId` cujo array `events` contenha `toStatus`, renderiza o payload e insere em `webhook_outbox` — tudo dentro da mesma transação `tx`.

### 11.2 `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`

**Integração:** novas classes de erro do módulo de webhooks estenderão `AppError` diretamente ou suas subclasses existentes (`NotFoundError`, `ConflictError`, `ValidationError`). Exemplo:

```typescript
export class WebhookNotFoundError extends NotFoundError {
  constructor() {
    super('Webhook endpoint');
    this.errorCode = 'WEBHOOK_NOT_FOUND'; // sobrescreve o 'NOT_FOUND' padrão
  }
}
```

Alternativamente, usar `AppError` diretamente com código customizado:

```typescript
throw new AppError('Webhook URL must use HTTPS', 400, 'WEBHOOK_INVALID_URL');
```

A segunda abordagem é mais consistente com o uso atual no código (`ConflictError` com código customizado, como `INVALID_STATUS_TRANSITION` em `src/shared/errors/http-errors.ts:45-53`).

### 11.3 `src/middlewares/error.middleware.ts` — error handler centralizado

**Integração:** nenhuma alteração necessária. O middleware já trata:
- `AppError` (status code, error code, message, details)
- `ZodError` (validação)
- `Prisma.PrismaClientKnownRequestError` (unique constraint, not found)

Todos os erros do módulo de webhooks que estendem `AppError` serão automaticamente serializados no formato `{ error: { code, message, details? } }`.

### 11.4 `src/middlewares/auth.middleware.ts:27-61` — `authenticate` e `requireRole`

**Integração:** os endpoints CRUD de webhook usarão `authenticate` (linha 27) como todos os outros módulos. O endpoint de replay de DLQ usará `requireRole('ADMIN')` (linha 49), que verifica `req.user.role`. O middleware já está disponível e testado; bastará compor na definição de rotas:

```typescript
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  authenticate,
  requireRole('ADMIN'),
  controller.replayDeadLetter
);
```

### 11.5 `src/middlewares/validate.middleware.ts` — validação Zod

**Integração:** os schemas Zod do módulo de webhooks (`src/modules/webhooks/webhook.schemas.ts`) serão aplicados via middleware `validate`, exatamente como os módulos existentes:

```typescript
router.post(
  '/',
  authenticate,
  validate({ body: createWebhookSchema }),
  controller.create
);
```

O middleware já converte erros `ZodError` em `ValidationError`, que por sua vez são tratados pelo `errorMiddleware`.

### 11.6 `src/shared/logger/index.ts` — logger Pino

**Integração:** o worker e o módulo de webhooks importam e usam o mesmo `logger` instanciado. Nenhuma configuração adicional necessária. O `requestLogger` middleware (`src/middlewares/request-logger.middleware.ts`) já loga automaticamente todas as requisições HTTP com `requestId`, `method`, `path`, `statusCode` e `durationMs`.

### 11.7 `src/app.ts:26-53` — `buildControllers` e `buildApp`

**Integração:** o `WebhookController` será instanciado e registrado em `buildControllers`, e suas rotas serão adicionadas em `buildApiRouter` (`src/routes/index.ts`):

```typescript
// Em buildControllers:
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository);
const webhookController = new WebhookController(webhookService);

// Em Controllers type (src/routes/index.ts):
export type Controllers = {
  // ... existing
  webhooks: WebhookController;
};

// Em buildApiRouter:
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks));
```

### 11.8 `src/server.ts:6-27` — bootstrap da API

**Integração:** o worker terá seu próprio entry point (`src/worker.ts`), análogo ao `src/server.ts`, mas sem Express. Ambos compartilham `createPrismaClient()` de `src/config/database.ts` e o logger.

---

## 12. Modelo de Dados (Novas Tabelas)

### `webhook_endpoints`

```prisma
model WebhookEndpoint {
  id            String   @id @default(uuid()) @db.Char(36)
  customerId    String   @db.Char(36)
  url           String   @db.VarChar(2048)
  secret        String   @db.VarChar(255)
  previousSecret String? @db.VarChar(255)
  secretRotatedAt DateTime?
  events        Json     // OrderStatus[] ex: ["PAID", "SHIPPED"]
  active        Boolean  @default(true)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}
```

### `webhook_outbox`

```prisma
model WebhookOutbox {
  id           String   @id @default(uuid()) @db.Char(36)
  eventId      String   @unique @db.Char(36)
  endpointId   String   @db.Char(36)
  orderId      String   @db.Char(36)
  customerId   String   @db.Char(36)
  eventType    String   @db.VarChar(100)
  payload      Json
  status       WebhookOutboxStatus @default(PENDING)
  attempt      Int      @default(0)
  lastError    String?  @db.Text
  nextRetryAt  DateTime @default(now())
  deliveredAt  DateTime?
  createdAt    DateTime @default(now())

  @@index([status, nextRetryAt])
  @@index([endpointId])
  @@index([orderId])
  @@map("webhook_outbox")
}

enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}
```

### `webhook_dead_letter`

```prisma
model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  eventId      String   @unique @db.Char(36)
  endpointId   String   @db.Char(36)
  orderId      String   @db.Char(36)
  customerId   String   @db.Char(36)
  payload      Json
  attempts     Int
  lastError    String?  @db.Text
  failedAt     DateTime @default(now())
  replayed     Boolean  @default(false)
  replayedAt   DateTime?
  replayedBy   String?  @db.Char(36)
  createdAt    DateTime @default(now())

  @@index([customerId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

### `webhook_delivery`

```prisma
model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  eventId        String   @db.Char(36)
  endpointId     String   @db.Char(36)
  status         String   @db.VarChar(20)
  responseStatus Int?
  responseBody   String?  @db.Text
  errorMessage   String?  @db.Text
  durationMs     Int
  createdAt      DateTime @default(now())

  @@index([endpointId, createdAt])
  @@index([eventId])
  @@map("webhook_delivery")
}
```

---

## 13. Riscos e Mitigação (Técnicos)

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker crash entre HTTP bem-sucedido e UPDATE de status | Baixa | Médio — evento reenviado (duplicata) | `X-Event-Id` garante idempotência no cliente; at-least-once é o contrato |
| Deadlock na outbox com alta concorrência | Baixa | Baixo — latência adicional | Índice em `(status, next_retry_at)` otimiza leitura; `SELECT ... LIMIT` reduz lock range |
| Payload muito grande (array de itens, etc.) | Muito baixa | Baixo | Payload fixo de ~500 bytes; validação de 64KB como safety net |
| Ataque de replay com evento antigo | Baixa | Médio | `X-Timestamp` permite que o cliente rejeite eventos com timestamp muito antigo (se implementar) |
