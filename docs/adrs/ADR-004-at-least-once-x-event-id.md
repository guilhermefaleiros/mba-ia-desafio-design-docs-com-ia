# ADR-004: Garantia At-Least-Once com X-Event-Id para Deduplicação

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Diego (Eng. Plataforma), Sofia (Eng. Segurança), Larissa (Tech Lead), Marcos (PM)

---

## Contexto

Em sistemas distribuídos com retry, é inevitável que um mesmo evento seja entregue mais de uma vez. Os cenários que produzem duplicação incluem:

- O worker envia o HTTP com sucesso, mas a resposta do cliente não chega (timeout de rede).
- O worker envia o HTTP, o cliente processa e persiste, mas o worker crasha antes de marcar o evento como entregue na outbox.
- O endpoint de replay da DLQ é acionado manualmente para um evento que, na verdade, já havia sido entregue.

Nesses casos, o cliente receberá o mesmo evento duas (ou mais) vezes. Sem um mecanismo de deduplicação, isso pode causar processamento duplicado de negócio (ex: enviar duas confirmações de "pedido pago" e o cliente despachar mercadoria em duplicidade).

## Decisão

**Garantia de entrega at-least-once. Cada evento carrega um identificador único (`X-Event-Id`, UUID v4) gerado no momento da inserção na outbox. A responsabilidade de deduplicação é do cliente.**

O `event_id` é um UUID v4 gerado quando o evento é inserido na tabela `webhook_outbox` (dentro da transação de mudança de status). Ele identifica unicamente aquele evento. Se o cliente receber duas requisições com o mesmo `X-Event-Id`, deve ignorar a segunda.

Essa é a abordagem adotada por plataformas como Stripe e GitHub ([09:25] Diego). A alternativa — garantia exactly-once — exigiria coordenação entre os dois lados (two-phase commit ou idempotency key armazenada e verificada pelo servidor antes de cada entrega), complexidade desproporcional ao benefício.

## Alternativas Consideradas

### Alternativa 1: Exactly-once delivery com idempotency key no servidor (descartada)

**Trade-off:** O servidor manteria um registro de quais `event_id` já foram confirmados como entregues por cada endpoint e não reenviaria duplicatas. Isso exigiria: (a) o cliente responder com confirmação explícita; (b) o servidor armazenar e consultar esse estado antes de cada envio; (c) lidar com o caso em que a confirmação do cliente se perde. A complexidade adicional não se justifica quando o padrão de mercado é at-least-once com dedup do lado do cliente. Stripe, GitHub, Twilio e SendGrid todos operam assim.

### Alternativa 2: Best-effort (sem garantia de entrega) (descartada)

**Trade-off:** O worker tentaria enviar uma vez e, se falhar, abandonaria o evento. Inaceitável para um sistema de notificações de pedidos, onde a perda de um evento de "pago" ou "entregue" tem impacto financeiro direto. Nunca foi seriamente considerada; a discussão partiu direto para at-least-once.

## Consequências

### Positivas

- **Simplicidade do servidor:** não há estado adicional para gerenciar além da outbox.
- **Sem ponto único de coordenação:** o servidor não precisa saber se o cliente já recebeu o evento.
- **Alinhado com mercado:** clientes B2B que já integram com Stripe ou GitHub entendem o modelo.

### Negativas

- **Responsabilidade transferida ao cliente:** o cliente precisa implementar deduplicação por `event_id`. Isso foi reconhecido e aceito; Marcos se comprometeu a documentar de forma destacada no portal do desenvolvedor ([09:26] Marcos).
- **Risco de má implementação no cliente:** se o cliente não implementar dedup corretamente, processará eventos duplicados. Mitigação: documentação com exemplos de código e menção explícita no contrato de integração.
- **Sem garantia de ordering global:** com single worker, a ordering é por `order_id` (eventos do mesmo pedido são processados em ordem de `created_at`). Mas entre pedidos diferentes, não há garantia. Essa limitação foi documentada como conhecida ([09:13] Larissa).

## Referências

- Transcrição: [09:24-09:26] Diego, Bruno, Sofia, Marcos, Larissa
- Código relacionado:
  - `src/middlewares/request-logger.middleware.ts` — já usa UUID v4 via lib `uuid` (disponível no projeto)
  - `package.json` — dependência `uuid` versão 11.0.3 já presente
