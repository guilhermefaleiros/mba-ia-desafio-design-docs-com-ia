# ADR-005: Worker em Processo Separado com Polling de 2 Segundos

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Diego (Eng. Plataforma), Larissa (Tech Lead), Bruno (Eng. Pedidos)

---

## Contexto

O envio de webhooks é uma operação de I/O bound (HTTP calls) que não deve compartilhar o mesmo processo da API REST. Se o worker rodasse no mesmo processo:

1. Um crash no worker derrubaria a API junto.
2. A reinicialização da API (deploy, restart) interromperia o processamento de webhooks.
3. O event loop da API competiria com as chamadas HTTP do worker.

A alternativa de usar mecanismos reativos (triggers de banco, NOTIFY/LISTEN) não está disponível no MySQL, e introduzir PostgreSQL só para isso seria desproporcional.

## Decisão

**O worker será um processo Node.js separado, iniciado via entry point `src/worker.ts` com script `npm run worker`. Ele fará polling da tabela `webhook_outbox` a cada 2 segundos, buscando eventos com status `PENDING` e `next_retry_at <= NOW()`.**

Características:

- **Entry point separado:** `src/worker.ts`, análogo ao `src/server.ts` da API, mas sem escutar porta HTTP.
- **Instância própria do PrismaClient:** mesmo banco (mesma `DATABASE_URL`), mas instância separada porque é outro processo Node ([09:30] Bruno).
- **Polling, não push:** o worker executa um loop: `SELECT ... WHERE status = 'PENDING' AND next_retry_at <= NOW() ORDER BY created_at LIMIT $BATCH_SIZE`, processa o batch, repete após 2 segundos.
- **Latência pior caso:** 2 segundos entre a inserção na outbox e a próxima leitura do worker. Alinhado com o requisito de negócio de <10 segundos ([09:02] Marcos).
- **Single worker, sem concorrência:** no primeiro momento, um único worker processa os eventos, garantindo ordering implícita por `order_id` ([09:12] Diego).

### Alternativa 1: Trigger de banco + notificação externa (descartada)

**Trade-off:** MySQL não tem `NOTIFY/LISTEN` como o PostgreSQL. Usar triggers para escrever em arquivo ou chamar um endpoint interno seria um improviso frágil. Também não traria benefício real de latência que justificasse a complexidade. Diego descreveu como "esquisito" ([09:09] Diego).

### Alternativa 2: Worker no mesmo processo da API (descartada)

**Trade-off:** Simplicidade de deploy (um só processo), mas acopla o ciclo de vida do worker ao da API. Se a API reiniciar por deploy, o worker morre junto. Se o worker travar por erro não tratado, a API cai. Diego foi taxativo: "não pode ser o mesmo processo" ([09:11] Diego).

## Consequências

### Positivas

- **Desacoplamento de ciclo de vida:** API e worker podem ser reiniciados, escalados e monitorados independentemente.
- **Isolamento de falhas:** um crash no worker não afeta a API REST.
- **Simplicidade:** polling em loop é trivial de implementar e debugar. Não requer bibliotecas de fila, nem conexão com broker externo.
- **Reuso de stack:** mesmo Prisma, mesmo Pino, mesmas classes de erro. Nenhuma dependência nova.

### Negativas

- **Latência de polling:** 2 segundos de latência mínima. Aceitável para o requisito, mas impede cenários de "tempo real" sub-segundo no futuro.
- **Single worker = sem paralelismo:** um só worker processa um batch por vez. Se o volume de eventos crescer muito, o throughput fica limitado. Mitigação futura: escalar para múltiplos workers com particionamento por `order_id` ou lock pessimista (`SELECT ... FOR UPDATE SKIP LOCKED` no MySQL 8.0+).
- **Carga de polling constante:** mesmo sem eventos pendentes, o worker consulta a outbox a cada 2s. Com índice adequado, a consulta é leve, mas é uma carga constante. Mitigação: se a outbox estiver vazia por N ciclos consecutivos, aumentar dinamicamente o intervalo (adaptive polling), embora isso esteja fora do escopo inicial.
- **Deploy e operação:** mais um processo para gerenciar no Docker Compose, healthcheck, monitoramento. O `docker-compose.yml` precisará de um serviço adicional.

## Referências

- Transcrição: [09:08] Larissa, [09:09] Diego/Bruno, [09:10] Marcos/Larissa, [09:11] Diego/Bruno, [09:12-09:13] Diego/Larissa
- Código relacionado:
  - `src/server.ts` — entry point da API, modelo para `src/worker.ts`
  - `src/config/database.ts` — `createPrismaClient()`, reutilizável pelo worker
  - `docker-compose.yml` — orquestração de containers
