# ADR-001: Padrão Outbox no MySQL para disparo de webhooks

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)

---

## Contexto

A feature de Webhooks de Notificação de Pedidos exige que, a cada mudança de status de um pedido, um evento seja enviado via HTTP para endpoints externos cadastrados pelos clientes. O método `changeStatus` do `OrderService` (`src/modules/orders/order.service.ts`) já executa uma transação que envolve:

1. Validação da transição de status (máquina de estados em `src/modules/orders/order.status.ts`)
2. Débito ou reposição de estoque conforme a transição
3. `UPDATE` na tabela `orders`
4. `INSERT` na tabela `order_status_history`

Acrescentar uma chamada HTTP síncrona dentro dessa transação traria dois problemas: (a) degradação de performance — um cliente lento travaria todas as mudanças de status subsequentes; (b) inconsistência transacional — se o HTTP falhar, não há como decidir entre dar rollback na mudança de status (efeito colateral de negócio inaceitável) ou prosseguir sem notificar (quebra de contrato com o cliente).

## Decisão

**Usaremos o padrão Transactional Outbox implementado diretamente no MySQL existente.** Nenhuma infraestrutura adicional de mensageria (Redis Streams, RabbitMQ, Kafka, SQS) será introduzida.

O funcionamento:

1. Dentro da mesma transação Prisma que atualiza `orders` e `order_status_history`, o `OrderService.changeStatus` insere uma linha na tabela `webhook_outbox` com o payload do evento já renderizado (snapshot).
2. Um worker separado (`src/worker.ts`) lê a tabela `webhook_outbox` em polling a cada 2 segundos e dispara as chamadas HTTP.
3. Se a transação principal der rollback, o registro na outbox some junto — garantia de consistência sem coordenação externa.

## Alternativas Consideradas

### Alternativa 1: Redis Streams (descartada)

**Trade-off:** Redis Streams ofereceria latência mais baixa (push vs. polling), mas exigiria provisionar e operar um cluster Redis. O time é pequeno e a aplicação não usa Redis hoje. O ganho de latência (de ~2s para ~100ms) não justifica o custo operacional, já que o requisito de negócio é "abaixo de 10 segundos" ([09:02] Marcos). A complexidade adicional de garantir atomicidade entre a transação MySQL e a publicação no Redis (sem dois-phase commit) também pesou contra.

### Alternativa 2: Chamada HTTP síncrona dentro da transação (descartada)

**Trade-off:** Seria a implementação mais simples, mas acopla a latência e disponibilidade do cliente externo à operação local de mudança de status. Se o endpoint do cliente estiver lento ou fora do ar, a API de pedidos inteira degrada. Bruno e Diego foram unânimes em rejeitar ([09:04] Bruno, [09:06] Diego).

## Consequências

### Positivas

- **Consistência garantida:** evento registrado se e somente se a transação de negócio commitar. Sem mensagens fantasmas ou perdidas.
- **Zero infra adicional:** opera sobre o MySQL existente, mesmo pool de conexões, sem novos serviços para gerenciar.
- **Simplicidade operacional:** mesmo time que opera o MySQL opera a outbox.
- **Observabilidade integrada:** métricas e logs do worker usam o mesmo stack Pino do resto da aplicação.

### Negativas

- **Latência mínima de ~2 segundos:** o polling introduz um delay entre a mudança de status e o envio do webhook. Aceitável para o requisito de negócio (<10s), mas impede uso em cenários que exijam latência sub-segundo.
- **Crescimento da tabela:** a `webhook_outbox` acumula linhas processadas. Mitigação: arquivamento de linhas entregues após 30 dias (fora do escopo desta feature, mas previsto).
- **Single-worker, single-point:** enquanto houver apenas um worker, não há paralelismo de envio. Se o worker cair, os envios param até ele ser reiniciado. Mitigação: o worker é um processo separado com healthcheck; em caso de crash, o processo é reiniciado pelo orchestrator (Docker, systemd, etc.).
- **Polling em banco relacional:** carga adicional de leitura no MySQL. Com índice em `(status, created_at)` e batch pequeno, o impacto é mínimo.

## Referências

- Transcrição: [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:08] Diego
- Código relacionado:
  - `src/modules/orders/order.service.ts:126-179` — método `changeStatus` com transação Prisma
  - `src/modules/orders/order.status.ts` — máquina de estados e funções de transição
  - `src/config/database.ts` — configuração do PrismaClient
  - `prisma/schema.prisma` — schema do banco de dados
