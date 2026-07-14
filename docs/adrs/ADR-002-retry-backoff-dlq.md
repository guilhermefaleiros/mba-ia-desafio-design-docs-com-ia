# ADR-002: Política de Retry com Backoff Exponencial e DLQ

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)

---

## Contexto

O worker de webhooks envia requisições HTTP para endpoints externos que podem estar temporariamente indisponíveis (manutenção, picos de carga, falhas de rede). Se a entrega falhar, o sistema precisa reenviar o evento sem intervenção manual, mas também precisa de um limite: eventos não podem ficar pendentes indefinidamente nem congestionar a fila de envios ativos.

A discussão na reunião ponderou entre 3 ou 5 tentativas ([09:15] Diego, [09:16] Bruno) e entre diferentes progressões de intervalo. O histórico de indisponibilidade de clientes — incluindo janelas de manutenção planejada de até 2 horas ([09:16] Diego) — foi o fator determinante.

## Decisão

**Adotaremos 5 tentativas com backoff exponencial progressivo: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas.**

Após a quinta falha consecutiva, o evento é movido para uma tabela separada de **Dead Letter Queue (DLQ)**: `webhook_dead_letter`. O registro na DLQ preserva o payload original, o motivo da última falha e o timestamp, permitindo diagnóstico e reprocessamento manual.

A progressão cobre uma janela total de aproximadamente 15 horas entre a primeira tentativa e o descarte para DLQ, o que acomoda janelas de manutenção de clientes e falhas de rede prolongadas.

O reprocessamento de eventos da DLQ é feito via endpoint administrativo `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Este endpoint exige role `ADMIN` (reaproveitando o middleware `requireRole` de `src/middlewares/auth.middleware.ts`).

## Alternativas Consideradas

### Alternativa 1: 3 tentativas (descartada)

**Trade-off:** Mais agressiva, encerraria o ciclo de retry em cerca de 36 minutos (1m/5m/30m). Bruno defendeu essa opção ([09:16] Bruno), mas Diego contra-argumentou que clientes já tiveram indisponibilidade de 2 horas em manutenção planejada ([09:16] Diego). Com 3 tentativas, uma manutenção programada de 2h resultaria em perda definitiva do evento. O custo adicional de 2 tentativas extras é baixo (apenas linhas na outbox aguardando `next_retry_at`) e o benefício de não perder eventos é alto.

### Alternativa 2: Retry indefinido com backoff (descartada)

**Trade-off:** Evitaria completamente a perda de eventos, mas eventos de clientes que descontinuaram o serviço ou de endpoints permanentemente quebrados congestionariam a outbox para sempre. A DLQ com reprocessamento manual oferece um meio-termo: eventos não se perdem, mas também não poluem a fila ativa indefinidamente.

## Consequências

### Positivas

- **Cobertura de indisponibilidade longa:** 15h de janela cobre manutenções planejadas e falhas de rede prolongadas.
- **Fila ativa limpa:** eventos que excederam o limite não ficam pendurados na outbox principal, mantendo a leitura do worker eficiente.
- **Reprocessamento controlado:** endpoint admin com exigência de role `ADMIN` garante que só usuários autorizados reativem eventos.
- **Rastreabilidade:** a DLQ mantém payload e motivo da falha para debug.

### Negativas

- **Perda definitiva se não houver reprocessamento manual:** após 5 falhas, se ninguém fizer replay pela API admin, o evento não será entregue. Mitigação: futuramente, um alerta pode ser configurado para notificar operadores quando a DLQ crescer.
- **Complexidade adicional:** duas tabelas (outbox + dead_letter) e lógica de transição entre elas. Justifica-se pela separação clara de responsabilidades.
- **Sem notificação proativa ao cliente:** o cliente não é avisado quando um evento vai para DLQ. Email de alerta foi discutido e explicitamente adiado para fase futura ([09:37] Larissa, [09:38] Marcos).

## Referências

- Transcrição: [09:15] Diego, [09:16] Bruno, [09:17] Diego, [09:18] Diego, [09:37-09:38] Larissa/Marcos
- Código relacionado:
  - `src/middlewares/auth.middleware.ts:49-61` — `requireRole` middleware
  - `src/shared/errors/app-error.ts` — classe base `AppError`
