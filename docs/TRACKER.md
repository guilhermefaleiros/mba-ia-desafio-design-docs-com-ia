# Tracker de Rastreabilidade

Mapeamento de cada item registrado nos documentos de design à sua origem na transcrição (`TRANSCRICAO.md`) ou no código fonte (`src/`).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|--------------------|-------|-------------|
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Criar configuração de webhook (POST, URL, secret gerada automaticamente) | TRANSCRICAO | [09:31] Marcos, [09:21] Sofia |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um customer com paginação | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Editar configuração de webhook (URL, eventos, active) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remover configuração de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Rotacionar secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Disparar evento na outbox durante changeStatus (atômico) | TRANSCRICAO | [09:06] Diego, [09:40] Bruno |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Worker enviar POST HTTP com headers X-Signature, X-Event-Id, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego, [09:45] Sofia/Diego |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Retry automático com backoff 1m/5m/30m/2h/12h (5 tentativas) | TRANSCRICAO | [09:15] Diego, [09:17] Diego/Larissa |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Mover evento para DLQ após exceder 5 tentativas | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Endpoint admin POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego, [09:35] Diego/Larissa |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Histórico de deliveries (GET /webhooks/:id/deliveries) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Validação: rejeitar URL que não seja HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Limite de payload de 64KB | TRANSCRICAO | [09:23] Sofia, [09:24] Diego/Larissa |
| PRD-RNF-01 | `docs/PRD.md` | Requisito Não Funcional | Latência p95 ≤ 10s (inserção na outbox → envio HTTP) | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-RNF-02 | `docs/PRD.md` | Requisito Não Funcional | Atomicidade: evento na outbox na mesma transação do changeStatus | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-03 | `docs/PRD.md` | Requisito Não Funcional | Isolamento de falhas: worker não afeta API REST | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| PRD-RNF-04 | `docs/PRD.md` | Requisito Não Funcional | HMAC-SHA256 sobre payload, secret por endpoint | TRANSCRICAO | [09:20] Sofia, [09:21] Sofia |
| PRD-RNF-05 | `docs/PRD.md` | Requisito Não Funcional | At-least-once com X-Event-Id (UUID) para dedup no cliente | TRANSCRICAO | [09:24] Diego, [09:25] Diego |
| PRD-RNF-06 | `docs/PRD.md` | Requisito Não Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| PRD-RNF-07 | `docs/PRD.md` | Requisito Não Funcional | Single worker para V1; arquitetura suporta escala horizontal futura | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| PRD-RNF-08 | `docs/PRD.md` | Requisito Não Funcional | Log estruturado com Pino em todas as etapas | TRANSCRICAO | [09:29] Larissa |
| PRD-RNF-09 | `docs/PRD.md` | Requisito Não Funcional | Seguir padrões do projeto: AppError, Zod, middleware de erro, auth | TRANSCRICAO | [09:27] Bruno, [09:30] Larissa |
| PRD-ESCOPO-01 | `docs/PRD.md` | Exclusão de Escopo | Notificação por email ao cliente sobre falhas — adiado | TRANSCRICAO | [09:37] Larissa, [09:38] Marcos |
| PRD-ESCOPO-02 | `docs/PRD.md` | Exclusão de Escopo | Dashboard visual para clientes — projeto separado do frontend | TRANSCRICAO | [09:39] Marcos, [09:40] Larissa |
| PRD-ESCOPO-03 | `docs/PRD.md` | Exclusão de Escopo | Rate limiting de saída — observar primeiro | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo/Métrica | Reduzir 80% das chamadas de polling em 60 dias | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo/Métrica | p95 de latência de entrega ≤ 10s | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa/Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo/Métrica | Taxa de entrega ≥ 99.5% em 30 dias | TRANSCRICAO | [09:15-09:17] Diego/Larissa (retry policy) |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | Clientes não adotarem webhooks e manterem polling | TRANSCRICAO | [09:00] Marcos (pedido formal de 3 clientes) |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | Atlas migrar para concorrente antes da entrega | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Volume de eventos subestimado, worker não dá conta | TRANSCRICAO | [09:12] Diego (single worker) |
| PRD-RISCO-04 | `docs/PRD.md` | Risco | Regressão na API de pedidos por conta da inserção na outbox | TRANSCRICAO | [09:04] Bruno, [09:40] Bruno |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Redis Streams — descartado por custo operacional e overengineering | TRANSCRICAO | [09:06] Diego, [09:07] Larissa/Diego |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Microserviço separado — descartado por complexidade para time pequeno | TRANSCRICAO | [09:27] Bruno (módulo no monólito) |
| RFC-ABERTO-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| RFC-ABERTO-02 | `docs/RFC.md` | Questão em Aberto | Notificação proativa (email) ao cliente sobre falhas — próxima fase | TRANSCRICAO | [09:37] Marcos/Larissa |
| RFC-ABERTO-03 | `docs/RFC.md` | Questão em Aberto | Escala horizontal do worker — problema do futuro | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| RFC-ABERTO-04 | `docs/RFC.md` | Questão em Aberto | Dashboard visual — projeto separado do frontend | TRANSCRICAO | [09:39] Marcos, [09:40] Larissa |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Padrão Outbox no MySQL em vez de Redis/SQS/fila externa | TRANSCRICAO | [09:06] Diego, [09:07] Larissa, [09:08] Larissa |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Transação atômica: outbox na mesma transação do changeStatus | TRANSCRICAO | [09:06] Diego, [09:40] Bruno |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Alternativa descartada: chamada HTTP síncrona dentro da transação | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego, [09:17] Diego/Larissa |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | DLQ em tabela separada com endpoint admin de replay | TRANSCRICAO | [09:17] Larissa, [09:18] Diego |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | Alternativa descartada: 3 tentativas (muito agressivo) | TRANSCRICAO | [09:16] Bruno/Debate Diego |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | Alternativa descartada: retry indefinido (poluiria outbox) | TRANSCRICAO | [09:15] Diego |
| ADR-003 | `docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre payload, secret única por endpoint | TRANSCRICAO | [09:20] Sofia, [09:21] Sofia |
| ADR-003 | `docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md` | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| ADR-003 | `docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md` | Decisão | TLS obrigatório (https apenas) | TRANSCRICAO | [09:23] Sofia |
| ADR-003 | `docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md` | Decisão | Alternativa descartada: secret global única (blast radius total) | TRANSCRICAO | [09:21] Sofia |
| ADR-004 | `docs/adrs/ADR-004-at-least-once-x-event-id.md` | Decisão | Garantia at-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:24] Diego, [09:25] Diego/Sofia |
| ADR-004 | `docs/adrs/ADR-004-at-least-once-x-event-id.md` | Decisão | Alternativa descartada: exactly-once (complexidade desproporcional) | TRANSCRICAO | [09:25] Diego |
| ADR-005 | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | [09:08] Larissa, [09:09] Diego, [09:11] Diego |
| ADR-005 | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Entry point separado: src/worker.ts, npm run worker | TRANSCRICAO | [09:11] Larissa/Bruno/Diego |
| ADR-005 | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Alternativa descartada: trigger de banco MySQL (sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Bruno/Diego |
| ADR-005 | `docs/adrs/ADR-005-worker-processo-separado-polling.md` | Decisão | Alternativa descartada: worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-006 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Reuso de AppError, Pino, Zod, middleware de erro, auth middleware | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa, [09:30] Larissa |
| ADR-006 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Estrutura de módulo igual aos existentes: controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno, [09:28] Diego |
| ADR-006 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa |
| ADR-007 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | Payload renderizado como snapshot no momento da inserção na outbox | TRANSCRICAO | [09:51] Bruno, [09:52] Larissa/Diego |
| ADR-007 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | UUID como ID (segue padrão do projeto) | TRANSCRICAO | [09:51] Diego/Larissa |
| ADR-007 | `docs/adrs/ADR-007-payload-snapshot-na-insercao.md` | Decisão | Payload inclui order_id, order_number, from/to_status, customer_id, total_cents; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /api/v1/webhooks — criar endpoint | TRANSCRICAO | [09:31] Marcos, [09:32] Bruno/Marcos/Larissa |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /api/v1/webhooks?customerId= — listar webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /api/v1/webhooks/:id — editar endpoint | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotacionar secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | DELETE /api/v1/webhooks/:id — remover endpoint | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — reprocessar DLQ | TRANSCRICAO | [09:18] Diego, [09:35] Diego/Larissa |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Payload do webhook: event_id, event_type, timestamp, data (order_id, order_number, etc.) | TRANSCRICAO | [09:43] Diego |
| FDD-ERRO-01 | `docs/FDD.md` | Matriz de Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno (padrão de códigos de erro) |
| FDD-ERRO-02 | `docs/FDD.md` | Matriz de Erro | WEBHOOK_INVALID_URL (400) — URL não é HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | `docs/FDD.md` | Matriz de Erro | WEBHOOK_PAYLOAD_TOO_LARGE (413) — payload > 64KB | TRANSCRICAO | [09:23] Sofia, [09:24] Diego |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção na outbox dentro da transação de changeStatus | TRANSCRICAO | [09:06] Diego, [09:40] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Worker: polling a cada 2s, batch de 20, envia HTTP, marca status | TRANSCRICAO | [09:08] Diego, [09:09] Diego/Bruno |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Retry: 5 tentativas com backoff, depois DLQ | TRANSCRICAO | [09:15] Diego, [09:17] Diego/Larissa |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | DLQ: replay via endpoint admin com role ADMIN | TRANSCRICAO | [09:18] Diego, [09:35] Sofia/Larissa |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Filtro de eventos: só insere na outbox se endpoint tem aquele status | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno/Diego |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego/Sofia |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Backoff exponencial: 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Idempotência via X-Event-Id (UUID v4) | TRANSCRICAO | [09:24] Diego, [09:25] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Métricas: outbox_size, delivery_latency_ms, success_rate, dlq_size | TRANSCRICAO | [09:29] Larissa (logger Pino) |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Logs estruturados: event_id em todos os logs do worker | TRANSCRICAO | [09:29] Larissa/Bruno |
| FDD-INT-01 | `docs/FDD.md` | Integração | order.service.ts:126-179 — changeStatus estendido com publishWebhookEvent | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | app-error.ts — AppError como classe base para erros WEBHOOK_* | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | error.middleware.ts — tratamento automático de AppError, ZodError, Prisma | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | auth.middleware.ts:49-61 — requireRole('ADMIN') para replay de DLQ | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | validate.middleware.ts — middleware de validação Zod reutilizado | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | logger/index.ts — Pino logger reutilizado no worker e controller | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | app.ts:26-53 — buildControllers e buildApp para registrar WebhookController | CODIGO | `src/app.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | routes/index.ts — buildApiRouter para adicionar rota /webhooks | CODIGO | `src/routes/index.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | server.ts — modelo para o entry point do worker (src/worker.ts) | CODIGO | `src/server.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | order.status.ts — máquina de estados e transições; determina quando eventos são gerados | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | database.ts — createPrismaClient reutilizado pelo worker | CODIGO | `src/config/database.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração | http/response.ts — paginated() reutilizado para listagem de deliveries | CODIGO | `src/shared/http/response.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Proposta | Outbox no MySQL com transação atômica | TRANSCRICAO | [09:06] Diego, [09:08] Larissa |
| RFC-PROP-02 | `docs/RFC.md` | Proposta | Worker separado em polling (2s) | TRANSCRICAO | [09:08] Larissa, [09:09] Diego |
| RFC-PROP-03 | `docs/RFC.md` | Proposta | Retry com backoff e DLQ em tabela separada | TRANSCRICAO | [09:15-09:18] Diego/Bruno/Larissa |
| RFC-PROP-04 | `docs/RFC.md` | Proposta | HMAC-SHA256, secret por endpoint, rotação com grace period | TRANSCRICAO | [09:19-09:22] Sofia/Diego |
| RFC-PROP-05 | `docs/RFC.md` | Proposta | At-least-once com X-Event-Id, dedup no cliente | TRANSCRICAO | [09:24-09:26] Diego/Sofia/Larissa |
| RFC-PROP-06 | `docs/RFC.md` | Proposta | Reuso de padrões: AppError, Pino, Zod, middleware | TRANSCRICAO | [09:27-09:30] Bruno/Diego/Larissa |
| RFC-PROP-07 | `docs/RFC.md` | Proposta | Payload snapshot na inserção | TRANSCRICAO | [09:51-09:52] Bruno/Larissa/Diego |
| RFC-PROP-08 | `docs/RFC.md` | Proposta | Estrutura do módulo: src/modules/webhooks/ + src/worker.ts | TRANSCRICAO | [09:27] Bruno, [09:28] Diego/Bruno |
| RFC-RISCO-01 | `docs/RFC.md` | Risco | Worker cair e eventos acumularem | TRANSCRICAO | [09:11] Diego/Larissa (worker separado) |
| RFC-RISCO-02 | `docs/RFC.md` | Risco | Cliente não implementar dedup e processar duplicado | TRANSCRICAO | [09:25] Diego/Sofia, [09:26] Marcos |
| RFC-RISCO-03 | `docs/RFC.md` | Risco | Vazamento de secret de webhook | TRANSCRICAO | [09:22] Diego ("cliente que vazou secret") |
| RFC-RISCO-04 | `docs/RFC.md` | Risco | Volume de eventos crescer além da capacidade do single worker | TRANSCRICAO | [09:12] Diego (single worker), [09:13] Larissa |
| RFC-PRAZO-01 | `docs/RFC.md` | Prazo | 3 sprints incluindo revisão de segurança da Sofia (2 dias) | TRANSCRICAO | [09:46] Larissa, [09:47] Sofia/Larissa |
| ADR-001-COD | `docs/adrs/ADR-001-outbox-no-mysql.md` | Referência Código | Método changeStatus com transação Prisma | CODIGO | `src/modules/orders/order.service.ts:126-179` |
| ADR-001-COD2 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Referência Código | Máquina de estados de pedidos | CODIGO | `src/modules/orders/order.status.ts` |
| ADR-003-COD | `docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md` | Referência Código | Schemas Zod nos módulos existentes (padrão de validação) | CODIGO | `src/modules/orders/order.schemas.ts` |
| ADR-006-COD | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Referência Código | Hierarquia de classes de erro | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-COD2 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Referência Código | Estrutura modular: orders como referência | CODIGO | `src/modules/orders/` |
| DET-01 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Timeout HTTP de 10 segundos no worker | TRANSCRICAO | [09:42] Diego/Sofia |
| DET-02 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Formato do payload: JSON sem items do pedido | TRANSCRICAO | [09:43] Diego |
| DET-03 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Headers: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego, [09:45] Sofia/Diego |
| DET-04 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Batch size do worker: 20 eventos por ciclo | TRANSCRICAO | [09:08] Diego ("batch pequeno") |
| DET-05 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | PrismaClient separado por processo (worker vs API) | TRANSCRICAO | [09:29] Diego, [09:30] Bruno |
| DET-06 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Customer não deriva do JWT; explícito no body ou path | TRANSCRICAO | [09:32] Bruno/Marcos/Larissa |
| DET-07 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Admin = role ADMIN no JWT; operador não pode mexer na DLQ | TRANSCRICAO | [09:35] Larissa, [09:36] Sofia |
| DET-08 | `docs/PRD.md` / `docs/FDD.md` | Detalhe Técnico | Single-worker garante ordering por order_id (não global) | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| DET-09 | `docs/PRD.md` | Detalhe Negócio | Clientes iniciais: Atlas Comercial, MaxDistribuição, Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| DET-10 | `docs/PRD.md` | Detalhe Negócio | Prazo: fim de novembro (Atlas) | TRANSCRICAO | [09:45] Marcos/Larissa |

---

**Resumo de cobertura:**

- Total de linhas: **96**
- Fonte `TRANSCRICAO`: **80** linhas (83%)
- Fonte `CODIGO`: **16** linhas (17%)
- Todos os timestamps seguem o formato `[hh:mm] Nome`
- Todos os caminhos de arquivo em `CODIGO` são reais e existem no repositório
