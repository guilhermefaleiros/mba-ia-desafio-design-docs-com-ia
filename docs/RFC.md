# RFC-001: Sistema de Webhooks de Notificação de Pedidos

**Autor:** Larissa (Tech Lead)
**Status:** Proposto — aguardando revisão
**Data:** 2026-07-14
**Revisores:** Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Sofia (Eng. Segurança), Marcos (PM)

---

## Resumo Executivo (TL;DR)

Proposta para implementação de um sistema de webhooks outbound que notifica clientes B2B em tempo real (<10s) sobre mudanças de status de pedidos, usando padrão Transactional Outbox no MySQL existente, worker separado com retry e backoff, autenticação HMAC-SHA256 por endpoint e garantia at-least-once com deduplicação via `X-Event-Id`. Nenhuma infraestrutura adicional é necessária. Feature estimada em 3 sprints.

---

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) solicitaram formalmente notificações em tempo real quando o status de seus pedidos muda na plataforma. Hoje eles fazem polling via `GET /orders`, o que é lento e caro. A Atlas Comercial sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre.

A aplicação atual é um OMS (Order Management System) em Node.js/TypeScript com MySQL (Prisma), com módulos de autenticação, usuários, clientes, produtos e pedidos. O ciclo de vida do pedido é controlado por máquina de estados, com transações atômicas para mudança de status, controle de estoque e auditoria. A aplicação **não possui** nenhum mecanismo de notificação externa, eventos, filas ou webhooks.

O desafio técnico é enviar notificações para endpoints externos sem degradar a operação local de mudança de status, sem introduzir inconsistências (status mudou mas evento não foi enviado, ou vice-versa) e sem adicionar infraestrutura que o time não tem capacidade de operar.

---

## Proposta Técnica

### Visão geral

A solução segue o padrão **Transactional Outbox** com quatro componentes principais:

```
┌──────────────┐    ┌──────────────────┐    ┌────────────┐    ┌─────────────┐
│ OrderService │───▶│ webhook_outbox   │◀───│  Worker    │───▶│ Cliente B2B │
│ changeStatus │    │ (mesma transação) │    │ (polling)  │    │ (HTTPS)     │
└──────────────┘    └──────────────────┘    └────────────┘    └─────────────┘
                            │                      │
                            ▼                      ▼
                    ┌──────────────────┐    ┌─────────────┐
                    │ webhook_dead_    │◀───│ 5 falhas?   │
                    │ letter (DLQ)     │    └─────────────┘
                    └──────────────────┘
```

1. **Inserção na outbox**: quando `OrderService.changeStatus` é chamado, dentro da mesma transação Prisma que atualiza `orders`, `order_status_history` e estoque, uma linha é inserida em `webhook_outbox` com o payload JSON completo (snapshot) do evento. Se a transação der rollback, o evento some junto — atomicidade garantida sem two-phase commit.

2. **Worker**: processo Node.js separado (`src/worker.ts`, `npm run worker`) faz polling da outbox a cada 2 segundos, busca eventos pendentes cujo `next_retry_at <= NOW()`, envia HTTP POST para os endpoints cadastrados e marca como entregue ou agenda retry.

3. **Retry e DLQ**: 5 tentativas com backoff exponencial (1m → 5m → 30m → 2h → 12h). Após a 5ª falha, o evento é movido para `webhook_dead_letter`, preservando payload e motivo da falha. Reprocessamento via `POST /admin/webhooks/dead-letter/:id/replay` (restrito a role `ADMIN`).

4. **Segurança**: cada endpoint de webhook tem secret única. Requisições são assinadas com HMAC-SHA256 sobre o corpo (header `X-Signature`). TLS obrigatório (URLs `http` são rejeitadas). Rotação de secret com grace period de 24h.

### Decisões arquiteturais

| Decisão | ADR |
|---------|-----|
| Outbox no MySQL, sem infra adicional | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| 5 retries com backoff e DLQ em tabela separada | [ADR-002](adrs/ADR-002-retry-backoff-dlq.md) |
| HMAC-SHA256 com secret por endpoint e rotação | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` para dedup | [ADR-004](adrs/ADR-004-at-least-once-x-event-id.md) |
| Worker em processo separado com polling 2s | [ADR-005](adrs/ADR-005-worker-processo-separado-polling.md) |
| Reuso de AppError, Pino, Zod, auth middleware | [ADR-006](adrs/ADR-006-reuso-padroes-existentes.md) |
| Payload renderizado como snapshot na inserção | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |

### Estrutura do módulo

O módulo de webhooks segue o padrão dos módulos existentes:

```
src/modules/webhooks/
├── webhook.controller.ts    # Handlers HTTP
├── webhook.service.ts       # Lógica de negócio (CRUD de endpoints)
├── webhook.repository.ts    # Acesso a dados (endpoints, outbox, DLQ)
├── webhook.processor.ts     # Lógica de envio HTTP (worker)
├── webhook.schemas.ts       # Schemas Zod
└── webhook.routes.ts        # Definição de rotas
src/worker.ts                # Entry point do worker
```

---

## Alternativas Consideradas

### Alternativa 1: Redis Streams como mecanismo de fila (descartada)

**O que era:** publicar eventos em Redis Streams no lugar da outbox em MySQL. O worker consumiria do Redis em vez de fazer polling no banco.

**Por que foi descartada:** introduziria Redis como dependência de infraestrutura — provisionamento, alta disponibilidade, monitoramento, backups — para um time pequeno que não opera Redis hoje. O ganho de latência (de ~2s de polling para <100ms push) não justifica o custo operacional, dado que o requisito de negócio é "abaixo de 10 segundos". Além disso, garantir atomicidade entre a transação MySQL e a publicação no Redis exigiria o padrão Outbox de qualquer forma, anulando parte do benefício.

### Alternativa 2: Serviço separado de notificações (microserviço) (descartada)

**O que era:** extrair toda a funcionalidade de webhooks para um microserviço independente, com seu próprio banco, comunicando-se com o OMS via eventos ou API.

**Por que foi descartada:** time pequeno, baixo volume atual de eventos. Um microserviço adicional significa: novo repositório, pipeline de CI/CD, autenticação serviço-a-serviço, contratos de API, debugging distribuído. O isolamento proporcionado pelo módulo dentro do monólito (com worker em processo separado) é suficiente para o desacoplamento necessário. Se no futuro o volume justificar, a extração é facilitada pelo fato de o módulo já ser bem delimitado.

---

## Questões em Aberto

1. **Rate limiting de saída:** se um cliente tiver 50 pedidos mudando de status em um minuto, o worker bombardeará o endpoint dele com 50 chamadas HTTP? Diego levantou essa preocupação ([09:38-09:39]). A decisão foi "observar e decidir depois". Se a observação mostrar degradação nos endpoints dos clientes, rate limiting por endpoint será implementado (ex: max 10 requisições/segundo por endpoint, com buffer).

2. **Notificação proativa ao cliente sobre falhas:** Marcos perguntou se o cliente pode ser avisado por email quando o webhook dele falhar 3 vezes seguidas ([09:37]). A decisão foi "fora de escopo dessa fase, talvez próxima fase". O cliente só descobre que houve falha consultando proativamente o endpoint `GET /webhooks/:id/deliveries`. Email de alerta é uma melhoria futura.

3. **Escala horizontal do worker:** com single worker, o throughput de envio é limitado pelo loop sequencial. A escalabilidade futura (múltiplos workers com `SELECT ... FOR UPDATE SKIP LOCKED`) foi discutida e considerada "problema do futuro" ([09:13] Diego). Essa decisão será reavaliada quando o volume de eventos se aproximar da capacidade de processamento de um único worker.

4. **Dashboard visual para clientes:** Marcos perguntou sobre painel para o cliente visualizar webhooks ([09:39]). Foi descartado para esta fase — "projeto separado do time de frontend" ([09:40] Larissa). Os clientes terão apenas os endpoints de API para gerenciar webhooks e consultar deliveries.

---

## Impacto e Riscos

### Impacto

- **Código:** novo módulo `src/modules/webhooks/` (~6 arquivos), novo entry point `src/worker.ts`, alteração pontual em `OrderService.changeStatus` para inserir na outbox, duas novas tabelas no schema Prisma, novas classes de erro com prefixo `WEBHOOK_`.
- **Infraestrutura:** novo serviço no `docker-compose.yml` para o worker. Mesmo banco MySQL, sem novas dependências.
- **Operação:** novo processo para monitorar (healthcheck do worker), novas métricas (latência de entrega, taxa de falha, tamanho da DLQ).

### Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker cair e eventos acumularem | Média | Alto — atraso nas notificações | Healthcheck + restart automático (Docker `restart: unless-stopped`); monitorar lag da outbox |
| Cliente não implementar dedup e processar duplicado | Média | Médio — processamento duplicado no cliente | Documentação destacada no portal dev; `X-Event-Id` documentado como obrigatório para idempotência |
| Vazamento de secret de webhook | Baixa | Alto — atacante pode forjar eventos | Rotação de secret com grace period; secret única por endpoint limita blast radius |
| Volume de eventos crescer além da capacidade do single worker | Baixa (curto prazo) | Médio — latência >10s | Monitorar métricas de throughput; plano de escala horizontal documentado (ADR-005) |

---

## Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Política de Retry com Backoff Exponencial e DLQ](adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004: Garantia At-Least-Once com X-Event-Id](adrs/ADR-004-at-least-once-x-event-id.md)
- [ADR-005: Worker em Processo Separado com Polling](adrs/ADR-005-worker-processo-separado-polling.md)
- [ADR-006: Reuso dos Padrões Existentes do Projeto](adrs/ADR-006-reuso-padroes-existentes.md)
- [ADR-007: Payload como Snapshot na Inserção](adrs/ADR-007-payload-snapshot-na-insercao.md)

---

## Referências

- Transcrição da reunião técnica: `TRANSCRICAO.md` (55 min, 5 participantes)
- Código base: `src/modules/orders/` (estrutura de referência), `src/shared/errors/` (hierarquia de erros), `src/middlewares/` (auth, error, validate)
