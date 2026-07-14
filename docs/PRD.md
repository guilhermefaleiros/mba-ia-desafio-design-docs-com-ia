# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** 2026-07-14
**Autor:** Marcos (Product Manager)
**Stakeholders:** Atlas Comercial, MaxDistribuição, Nova Cargo (clientes B2B)

---

## 1. Resumo e Contexto da Feature

Três clientes B2B da plataforma — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente um mecanismo de notificação em tempo real sobre mudanças de status de pedidos. Hoje, para saber se um pedido mudou de status, eles precisam fazer polling periódico no endpoint `GET /orders`. Esse modelo é caro para ambas as partes: gera carga desnecessária na API e impõe latência percebida elevada para os clientes.

A Atlas Comercial, maior dos três, sinalizou que pode migrar para um concorrente se a funcionalidade não for entregue até o fim do trimestre.

A feature proposta é um **Sistema de Webhooks de Notificação de Pedidos**: sempre que um pedido mudar de status, a plataforma envia uma notificação HTTP (webhook) para endpoints cadastrados pelos clientes, de forma assíncrona, autenticada e resiliente.

---

## 2. Problema e Motivação

**Problema atual:** clientes B2B dependem de polling manual via API REST para detectar mudanças de status de pedidos. Isso causa:

- **Latência percebida:** o cliente só descobre a mudança no próximo ciclo de polling (que pode ser de minutos).
- **Carga desnecessária:** chamadas repetidas a `GET /orders` que retornam "sem mudanças" consomem recursos da API e do banco.
- **Experiência ruim de integração:** o cliente precisa implementar lógica de polling, cache, diff de status — trabalho que deveria ser da plataforma.
- **Risco de churn:** a Atlas Comercial condicionou a permanência à entrega da feature.

**Solução proposta:** webhooks outbound que empurram eventos de mudança de status para os clientes em até 10 segundos, eliminando a necessidade de polling.

---

## 3. Público-Alvo e Cenários de Uso

### Público-alvo primário

- **Desenvolvedores/engenheiros de integração** dos clientes B2B, que vão cadastrar endpoints de webhook, receber eventos e implementar a lógica de consumo.
- **Operadores do OMS** (interno), que precisam de visibilidade sobre o status de entrega dos webhooks e capacidade de reprocessar eventos com falha.

### Cenários de uso

1. **Cadastro de webhook:** um desenvolvedor do cliente Atlas acessa o portal do desenvolvedor, aprende o contrato da API e usa `POST /api/v1/webhooks` para cadastrar o endpoint `https://api.atlas.com.br/webhooks/orders` com os eventos `["PAID", "SHIPPED", "DELIVERED"]`. A API retorna a secret `whsec_...` que ele armazena seguramente.

2. **Notificação em tempo real:** um operador do OMS muda o status de um pedido da Atlas de `PROCESSING` para `SHIPPED`. Em até 10 segundos, o sistema de webhooks envia um POST para `https://api.atlas.com.br/webhooks/orders` com o payload do evento. O sistema da Atlas recebe, valida a assinatura HMAC e atualiza o status no ERP deles.

3. **Diagnóstico de falha:** um desenvolvedor da MaxDistribuição percebe que não recebeu notificações nas últimas 2 horas. Ele consulta `GET /api/v1/webhooks/:id/deliveries`, vê que as últimas 5 entregas falharam com "Connection timeout", identifica que o endpoint deles estava fora do ar e aciona o time de infra.

4. **Reprocessamento de evento:** um evento foi para a DLQ após 5 falhas (cliente offline por 15h). Um operador do OMS com role ADMIN acessa `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, o evento volta para a fila e é entregue com sucesso na próxima tentativa.

5. **Rotação de secret:** o time de segurança da Nova Cargo identifica que a secret de webhook vazou em um log interno. O desenvolvedor chama `POST /api/v1/webhooks/:id/rotate-secret`, recebe a nova secret e atualiza o sistema deles. A secret antiga continua funcionando por 24h, garantindo zero downtime durante a migração.

---

## 4. Objetivos e Métricas de Sucesso

### Objetivo 1: Eliminar dependência de polling para clientes B2B

**Métrica:** Número de chamadas a `GET /orders` originadas de clientes B2B (identificadas por User-Agent ou customer_id).

**Meta:** Redução de 80% nas chamadas de polling em 60 dias após o lançamento, assumindo que os 3 clientes iniciais adotem webhooks e desliguem seus cron jobs de polling.

**Medição:** logs de acesso da API, filtrados por `customer_id` e endpoint `GET /orders`.

### Objetivo 2: Notificar clientes em até 10 segundos após mudança de status

**Métrica:** p95 da latência entre `order_status_history.changed_at` e o timestamp de chegada da requisição HTTP no endpoint do cliente (estimado pelo timestamp de resposta 2xx do cliente).

**Meta:** p95 ≤ 10 segundos em condições normais de operação (cliente online, sem retry).

**Medição:** `webhook_delivery_latency_ms` (histograma), capturado no momento em que o worker recebe a resposta HTTP do cliente.

### Objetivo 3: Garantir entrega dos eventos com resiliência

**Métrica:** taxa de entrega bem-sucedida (eventos que alcançaram o cliente com 2xx) sobre o total de eventos gerados, incluindo eventos que passaram por retry.

**Meta:** ≥ 99,5% de taxa de entrega bem-sucedida em 30 dias (excluindo eventos em que o endpoint do cliente ficou offline por mais de 15h, que vão para DLQ).

**Medição:** `webhook_delivery_success_rate` (counter), agregado por semana.

---

## 5. Escopo

### Incluso (v1)

- **F1:** CRUD de endpoints de webhook por customer (criar, listar, editar, remover)
- **F2:** Geração automática de secret única por endpoint (formato `whsec_<random>`)
- **F3:** Configuração de filtro de eventos por status (ex: "só quero SHIPPED e DELIVERED")
- **F4:** Inserção atômica de eventos na outbox durante mudança de status do pedido
- **F5:** Worker de envio HTTP assíncrono com retry e backoff (5 tentativas)
- **F6:** Dead Letter Queue para eventos com falha permanente, com endpoint admin de replay
- **F7:** Histórico de deliveries por endpoint (últimas 100 entregas)
- **F8:** Assinatura HMAC-SHA256 das requisições de webhook
- **F9:** Rotação de secret com grace period de 24 horas
- **F10:** Validação de TLS obrigatório (URLs http rejeitadas)
- **F11:** Autenticação JWT em todos os endpoints de gerenciamento de webhooks
- **F12:** Restrição de role ADMIN para operações de replay de DLQ
- **F13:** Limite de 64KB de payload por evento
- **F14:** Log estruturado (Pino) de todas as etapas do ciclo de vida do webhook

### Fora de escopo (v1)

1. **Notificação por email ao cliente sobre falhas consecutivas de webhook.** Discutido na reunião ([09:37] Marcos), adiado para fase futura após medição de impacto. O cliente pode consultar proativamente o histórico de deliveries via API.

2. **Dashboard visual / painel administrativo para gestão de webhooks.** Solicitado por Marcos ([09:39]), classificado como "projeto separado do time de frontend" ([09:40] Larissa). A V1 expõe apenas os endpoints de API.

3. **Rate limiting de saída por endpoint.** Diego levantou a preocupação ([09:38-09:39]). A decisão foi "observar e decidir depois". Se o volume de eventos por cliente causar degradação, rate limiting será implementado como melhoria.

4. **Arquivamento automático de eventos entregues.** Previsto na discussão técnica ([09:08] Diego — "fora do escopo dessa feature"), será implementado em fase subsequente. Na V1, linhas entregues permanecem na outbox (a tabela é pequena para o volume inicial).

---

## 6. Requisitos Funcionais

### RF01 — Criar configuração de webhook

O sistema deve permitir que um usuário autenticado cadastre um endpoint de webhook para um customer, informando URL (obrigatoriamente HTTPS) e lista de status de pedido que deseja receber. O sistema gera automaticamente uma secret única (`whsec_<uuid>`) e a retorna na resposta. O endpoint é criado com `active: true`.

**Origem:** [09:31] Marcos, [09:21] Sofia

### RF02 — Listar webhooks de um customer

O sistema deve permitir consultar todos os endpoints de webhook cadastrados para um customer, com paginação.

**Origem:** [09:33] Bruno, [09:31] Marcos

### RF03 — Editar configuração de webhook

O sistema deve permitir alterar URL, lista de eventos e status ativo/inativo de um endpoint de webhook existente.

**Origem:** [09:33] Bruno

### RF04 — Remover configuração de webhook

O sistema deve permitir excluir um endpoint de webhook. Eventos pendentes na outbox para o endpoint removido devem ser marcados como `FAILED` para não serem processados.

**Origem:** [09:33] Bruno

### RF05 — Rotacionar secret de webhook

O sistema deve permitir que o cliente solicite uma nova secret para um endpoint. A secret anterior permanece válida por 24 horas (grace period) para permitir migração sem downtime. Após 24h, apenas a nova secret é aceita.

**Origem:** [09:21-09:22] Sofia, Diego

### RF06 — Disparar evento na mudança de status do pedido

Sempre que o status de um pedido for alterado via `PATCH /orders/:id/status`, o sistema deve gerar um evento de webhook para cada endpoint ativo do customer que esteja inscrito no novo status. O evento é registrado atomicamente com a transação de mudança de status (padrão Outbox).

**Origem:** [09:06-09:08] Diego, Larissa; [09:33-09:34] Marcos, Bruno, Diego

### RF07 — Enviar webhook para o endpoint do cliente

Um worker separado deve ler eventos pendentes da outbox e enviá-los via HTTP POST para a URL cadastrada, com timeout de 10 segundos, headers de segurança (`X-Signature`, `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`) e payload JSON com os dados do evento.

**Origem:** [09:08-09:09] Larissa, Diego; [09:42-09:45] Diego, Sofia

### RF08 — Retry automático com backoff

Em caso de falha na entrega (não-2xx, timeout, erro de rede), o sistema deve reagendar o envio com backoff exponencial: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas — total de 5 tentativas.

**Origem:** [09:15-09:17] Diego, Bruno, Larissa

### RF09 — Mover para DLQ após exceder tentativas

Após a 5ª falha consecutiva, o evento deve ser movido para uma tabela de Dead Letter Queue, preservando payload e motivo da falha. O evento não será mais processado automaticamente.

**Origem:** [09:17-09:18] Larissa, Diego

### RF10 — Reprocessar evento da DLQ (admin)

Um usuário com role ADMIN deve poder reprocessar um evento da DLQ via `POST /admin/webhooks/dead-letter/:id/replay`. O evento é reinserido na outbox como pendente. A ação deve ser logada com o userId do operador.

**Origem:** [09:18-09:19] Diego, Larissa; [09:35-09:36] Sofia

### RF11 — Histórico de deliveries

O sistema deve expor um endpoint `GET /webhooks/:id/deliveries` que retorna as últimas entregas (sucesso e falha) de um endpoint de webhook, com paginação, incluindo status HTTP de resposta, duração e mensagem de erro quando aplicável.

**Origem:** [09:34] Marcos

### RF12 — Validação de URL HTTPS

O sistema deve rejeitar (400) o cadastro de webhook com URL que não use o protocolo `https://`.

**Origem:** [09:23] Sofia

### RF13 — Limite de tamanho de payload

O sistema deve rejeitar eventos cujo payload serializado exceda 64KB (65536 bytes). O erro `WEBHOOK_PAYLOAD_TOO_LARGE` deve ser registrado.

**Origem:** [09:23-09:24] Sofia, Diego, Larissa

---

## 7. Requisitos Não Funcionais

### RNF01 — Latência de notificação

A latência entre a mudança de status (commit da transação) e o envio da requisição HTTP ao cliente deve ter p95 ≤ 10 segundos em condições normais (cliente online, sem retry).

**Origem:** [09:02] Marcos ("abaixo de 10 segundos"), [09:10] Larissa

### RNF02 — Atomicidade

O registro do evento de webhook deve ser atômico com a transação de mudança de status: se a transação commitar, o evento é registrado; se der rollback, o evento não é registrado.

**Origem:** [09:06-09:07] Diego ("Garante que se a transação principal commitou, o evento foi registrado")

### RNF03 — Isolamento de falhas

Falhas no envio de webhooks (timeout, endpoint offline) não devem impactar a disponibilidade ou latência da API REST de pedidos.

**Origem:** [09:04] Bruno ("qualquer cliente lento vai travar mudança de status pra outros pedidos")

### RNF04 — Segurança na entrega

Toda requisição de webhook deve ser assinada com HMAC-SHA256 usando secret única por endpoint. O header `X-Signature` deve permitir que o cliente verifique autenticidade e integridade do payload.

**Origem:** [09:19-09:21] Sofia

### RNF05 — Idempotência

O sistema garante entrega at-least-once. Cada evento carrega um `X-Event-Id` único (UUID v4). O cliente é responsável por deduplicar eventos com mesmo ID.

**Origem:** [09:24-09:26] Diego, Sofia, Larissa

### RNF06 — Segurança na rotação de secrets

A rotação de secret deve ter grace period de 24 horas, durante o qual tanto a secret nova quanto a anterior são aceitas para assinatura.

**Origem:** [09:21-09:22] Sofia, Diego

### RNF07 — Escalabilidade

A solução deve suportar o volume atual de mudanças de status sem degradação. Single worker é suficiente para a V1; a arquitetura (polling com `SELECT ... FOR UPDATE SKIP LOCKED`) suporta escala horizontal futura sem redesenho.

**Origem:** [09:12-09:13] Diego, Larissa

### RNF08 — Observabilidade

Todas as etapas do ciclo de vida do webhook (inserção na outbox, envio, retry, DLQ) devem ser registradas em log estruturado (Pino). Métricas de latência, taxa de sucesso e tamanho da DLQ devem estar disponíveis.

**Origem:** [09:29] Larissa ("O logger, que é Pino, já tá no projeto inteiro")

### RNF09 — Consistência com o código existente

O módulo de webhooks deve seguir os mesmos padrões do projeto: AppError, Pino, Zod, middleware de erro centralizado, middleware de autenticação, estrutura de módulos em `src/modules/`.

**Origem:** [09:27-09:30] Bruno, Diego, Larissa

---

## 8. Decisões e Trade-offs Principais

| Decisão | Trade-off |
|---------|-----------|
| Outbox no MySQL em vez de Redis/SQS | Simplicidade operacional (zero infra nova) vs. latência de ~2s (aceitável para <10s) |
| 5 retries com backoff (1m/5m/30m/2h/12h) | Cobertura de 15h de indisponibilidade vs. eventos ocupando outbox por até 15h |
| HMAC-SHA256 simétrico por endpoint | Simplicidade de implementação e verificação vs. secret compartilhada (se vazar, ambos os lados comprometidos) |
| At-least-once (não exactly-once) | Simplicidade do servidor vs. responsabilidade de dedup no cliente |
| Single worker, single process | Simplicidade operacional vs. sem paralelismo de envio |
| Payload snapshot na inserção | Imutabilidade e independência do worker vs. potencial envio de dado "corrigido depois" |

Para detalhamento completo de cada decisão, consulte os ADRs em `docs/adrs/`.

---

## 9. Dependências

### Dependências técnicas (já existentes no projeto)

- **MySQL + Prisma:** banco de dados e ORM. As tabelas de outbox e DLQ são criadas via migration Prisma.
- **Express 4.21:** framework HTTP para os endpoints de CRUD de webhooks.
- **Pino 9.5:** logger estruturado usado no worker e nos controllers.
- **Zod 3.23:** validação de schemas de entrada.
- **JWT + middleware `authenticate`:** autenticação dos endpoints de gerenciamento.
- **Node.js crypto (nativo):** HMAC-SHA256 para assinatura de webhooks.

### Dependências externas

- Nenhuma. A feature não depende de serviços de terceiros, brokers de mensageria ou APIs externas.

### Dependências de produto

- **Portal do desenvolvedor:** documentação da API de webhooks para clientes (responsabilidade de Marcos, [09:26]).
- **Revisão de segurança:** Sofia reservou 2 dias úteis para revisão do código de HMAC e geração de secrets antes do deploy ([09:46-09:47]).

---

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Clientes não adotarem webhooks e manterem polling | Média | Baixo — carga de polling continua | Comunicação proativa com os 3 clientes; métrica de adoção para follow-up |
| Atlas migrar para concorrente antes da entrega | Média | Alto — perda de cliente âncora | Prazo de 3 sprints (~6 semanas) atende "fim de trimestre"; milestone checkpoints semanais |
| Volume de eventos subestimado, worker não dá conta | Baixa | Médio — latência de entrega >10s | Métricas de throughput monitoradas; plano de escala documentado (ADR-005) |
| Cliente não proteger a secret e sofrer ataque | Média | Alto — reputação da plataforma | Documentação com boas práticas; rotação de secret simples (1 endpoint); grace period evita downtime |
| Regressão na API de pedidos por conta da inserção na outbox | Baixa | Alto — degradação da operação principal | Inserção na outbox é um INSERT simples, mesma transação; testado com carga; métrica de latência do `PATCH /orders/:id/status` monitorada |
| Complexidade do módulo aumentar tempo de onboarding | Média | Baixo — novos devs demoram mais para entender | Padrão consistente com módulos existentes (ADR-006); documentação interna no FDD |

---

## 11. Critérios de Aceitação

1. Um cliente B2B consegue cadastrar um endpoint de webhook, receber a secret e começar a receber notificações em menos de 5 minutos (assumindo que o endpoint dele esteja implementado).
2. Toda mudança de status de pedido gera um evento de webhook para os endpoints inscritos naquele status, em até 10 segundos (p95).
3. Se o endpoint do cliente estiver offline, o sistema retenta automaticamente por até ~15 horas (5 tentativas), sem intervenção manual.
4. Após 5 falhas consecutivas, o evento vai para DLQ e um ADMIN pode reprocessá-lo manualmente.
5. O cliente consegue ver o histórico de entregas (sucesso/falha) dos últimos webhooks enviados para ele.
6. O cliente consegue rotacionar a secret do webhook sem perder entregas (grace period de 24h).
7. Nenhuma mudança de status de pedido é bloqueada ou atrasada por falha no sistema de webhooks.
8. URLs `http://` são rejeitadas no cadastro.
9. Apenas usuários com role ADMIN conseguem reprocessar eventos da DLQ.
10. Todas as operações de webhook (criação, envio, retry, DLQ, replay) são registradas em log estruturado com os campos definidos no FDD.

---

## 12. Estratégia de Testes e Validação

### Testes unitários

- Cálculo de HMAC-SHA256 com vetor de teste conhecido
- Lógica de backoff (next_retry_at para cada tentativa)
- Validação de schemas Zod (URL https, eventos válidos, limite de payload)
- Máquina de estados da outbox (PENDING → PROCESSING → DELIVERED/FAILED)
- Filtro de eventos (status na lista de eventos do endpoint)

### Testes de integração

- Criação de webhook endpoint → persistência no banco → secret gerada
- `changeStatus` com endpoint ativo → linha na outbox na mesma transação
- `changeStatus` com `InsufficientStockError` → rollback → zero linhas na outbox
- Worker processa evento → HTTP POST chega no endpoint mockado → status atualizado para DELIVERED
- Worker falha 5 vezes → evento movido para DLQ
- Replay de DLQ → evento reinserido na outbox → processado com sucesso
- Rotação de secret → ambas secrets válidas durante grace period

### Testes end-to-end

- Fluxo completo: criar endpoint → mudar status de pedido → worker envia webhook → mock verifica headers e payload → delivery registrada
- Cenário de falha: mock retorna 500 → retry com backoff → após 5 falhas, evento na DLQ → admin faz replay → mock retorna 200 → evento entregue
- Cenário de segurança: requisição sem `X-Signature` → cliente rejeita (teste de contrato)
- Cenário de concorrência: 50 mudanças de status em paralelo → todos os eventos aparecem na outbox → worker processa todos

### Testes de performance

- Medir latência do `PATCH /orders/:id/status` com e sem inserção na outbox (基线 comparison)
- Medir throughput do worker (eventos/segundo) com endpoint mockado respondendo em 100ms
- Medir latência de entrega (p95) em condição normal
- Medir uso de CPU e memória do worker sob carga constante

### Validação de segurança (Sofia)

- Revisão de código do fluxo de geração e armazenamento de secrets
- Revisão da implementação de HMAC-SHA256
- Teste de rotação de secret (grace period, expiração, ambas secrets simultâneas)
- Verificação de que secrets nunca aparecem em logs (Pino redact config)
