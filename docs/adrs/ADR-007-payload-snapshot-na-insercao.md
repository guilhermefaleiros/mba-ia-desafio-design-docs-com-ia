# ADR-007: Payload do Evento Renderizado como Snapshot na Inserção

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)

---

## Contexto

Quando um evento de mudança de status é registrado na outbox, há duas abordagens possíveis para o payload que será enviado ao cliente:

1. **Snapshot na inserção:** renderizar o JSON completo do evento no momento em que a linha é inserida na `webhook_outbox` e armazená-lo na tabela.
2. **Renderização tardia:** armazenar apenas `order_id`, `from_status` e `to_status` na outbox; montar o payload completo no momento do envio, consultando a order atual no banco.

A diferença é relevante porque o pedido pode sofrer novas alterações entre a inserção na outbox e o envio pelo worker (ex: o pedido muda de `PROCESSING` para `SHIPPED` enquanto o evento de `PAID` ainda está na fila de retry). Além disso, a ordem pode ser alterada por um operador (correção de valor, cancelamento de item).

## Decisão

**O payload completo do webhook será renderizado no momento da inserção na outbox (snapshot).** O campo `payload` (JSON) na tabela `webhook_outbox` conterá o corpo exato que será enviado ao cliente, sem necessidade de consultas adicionais no momento do envio.

O payload inclui: `event_id`, `event_type` (`order.status_changed`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`. Itens do pedido não são incluídos para manter o payload enxuto; se o cliente quiser detalhes, consulta `GET /orders/:id` ([09:43] Diego).

## Alternativas Consideradas

### Alternativa 1: Renderização tardia (descartada)

**Trade-off:** Armazenar apenas referências (`order_id`, `from_status`, `to_status`) economizaria espaço na outbox e garantiria que o payload sempre refletisse o estado mais atual do pedido. No entanto:
- Se o pedido for alterado (ex: correção de valor) ou tiver itens removidos entre a inserção e o envio, o evento passaria a refletir um estado que não corresponde ao momento da transição.
- O worker precisaria fazer um `JOIN` com `orders` para cada envio, adicionando latência e complexidade.
- Se o pedido for deletado (possível apenas em `PENDING` ou `CANCELLED`, mas ainda assim um risco), a renderização tardia falharia com `NOT_FOUND`.

Larissa resumiu: "se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" ([09:52] Larissa).

### Alternativa 2: Event sourcing com log de eventos (não considerada na reunião)

**Trade-off:** Em vez de snapshots, armazenar apenas a intenção do evento (`order_id`, transição) e derivar o payload de uma projeção. Isso é o padrão em sistemas de event sourcing puro, mas exigiria reconstruir o estado do pedido no momento do evento, o que é complexo e desnecessário para o escopo de webhooks de notificação.

## Consequências

### Positivas

- **Imutabilidade do evento:** o payload registrado nunca muda, independentemente do que acontecer com o pedido depois. Facilita debugging e auditoria.
- **Independência do worker:** o worker não precisa consultar `orders` para montar o payload — só lê a outbox e envia. Mais rápido, menos queries.
- **Sobrevive a deleção de pedido:** se o pedido for deletado após a transição, o evento ainda tem payload completo.

### Negativas

- **Espaço em disco:** cada linha da outbox armazena o payload JSON completo (estimado em ~500 bytes por evento). Para 100.000 eventos/mês, são ~50 MB, desprezível para MySQL. As linhas processadas serão arquivadas após 30 dias.
- **Inconsistência potencial:** se um campo do pedido for corrigido (ex: `total_cents` estava errado e foi ajustado), o webhook enviará o valor do momento da transição, que pode estar incorreto. Esse risco foi aceito: se o valor estava errado no momento da transição, o evento deve refletir exatamente o que foi registrado. Correções posteriores gerariam novos eventos de status, se aplicável.
- **Sem atualização retroativa:** se o formato do payload evoluir (ex: adicionar novo campo), eventos antigos na outbox ou em retry continuarão com o formato antigo. Isso é uma característica, não um bug: o snapshot é imutável por definição.

## Referências

- Transcrição: [09:43] Diego (formato do payload), [09:51-09:52] Bruno/Larissa/Diego (snapshot vs renderização tardia)
- Código relacionado:
  - `src/modules/orders/order.service.ts:126-179` — `changeStatus`, onde a inserção na outbox ocorrerá
  - `prisma/schema.prisma` — modelo `Order`, que contém os campos incluídos no payload
