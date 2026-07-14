# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos)

---

## Contexto

Os webhooks entregam eventos com dados de pedidos (status, valores, customer_id) para endpoints HTTP externos fora da infraestrutura da empresa. O cliente precisa conseguir validar duas coisas:

1. **Autenticidade:** a requisição realmente veio da nossa plataforma, e não de um atacante.
2. **Integridade:** o payload não foi adulterado em trânsito.

Sem um mecanismo de assinatura, um atacante que descubra a URL do webhook do cliente poderia enviar eventos falsos, potencialmente causando decisões de negócio incorretas (ex: liberar mercadoria com base em um evento forjado de "pago").

Além disso, diferentes clientes não devem compartilhar o mesmo segredo — se um vazar, todos os outros ficam comprometidos.

## Decisão

**Cada endpoint de webhook terá uma secret única gerada pelo servidor. As requisições HTTP de webhook serão assinadas com HMAC-SHA256 sobre o corpo do request, e a assinatura será enviada no header `X-Signature`.**

Detalhes:

- **Algoritmo:** HMAC-SHA256. É o padrão de mercado (Stripe, GitHub, Twilio usam variantes) e todo cliente corporativo tem biblioteca para verificá-lo ([09:20] Sofia).
- **Secret por endpoint:** cada registro na tabela de configuração de webhook (`webhook_endpoints`) armazena `url`, `secret` (gerada pelo servidor na criação), `customer_id` e `active`. Se um segredo vazar, apenas aquele endpoint é comprometido ([09:21] Sofia).
- **Rotação de secret:** o cliente pode solicitar nova secret via `POST /api/v1/webhooks/:id/rotate-secret`. Durante 24 horas (grace period), tanto a secret antiga quanto a nova são aceitas para verificação de assinatura. Após esse período, a antiga é invalidada ([09:21] Sofia, [09:22] Diego).
- **TLS obrigatório:** toda URL de webhook cadastrada deve usar `https`. URLs com `http` são rejeitadas na validação (schema Zod) ([09:23] Sofia).

### Headers da requisição de webhook

| Header | Conteúdo |
|--------|----------|
| `X-Signature` | HMAC-SHA256 do corpo do request, hex-encoded |
| `X-Event-Id` | UUID do evento (para deduplicação) |
| `X-Timestamp` | Timestamp ISO 8601 do envio (para detecção de replay attack) |
| `X-Webhook-Id` | ID do endpoint de webhook cadastrado |
| `Content-Type` | `application/json` |

## Alternativas Consideradas

### Alternativa 1: Secret global única para toda a plataforma (descartada)

**Trade-off:** Mais simples de implementar (uma variável de ambiente, sem tabela de secrets), mas o blast radius de um vazamento é total. Sofia rejeitou com o argumento "se vaza uma, vaza tudo" ([09:21] Sofia). O histórico de vazamento de secret em log de aplicação de cliente ([09:22] Diego) reforça que o risco é real.

### Alternativa 2: JWT assinado como prova de autenticidade (descartada)

**Trade-off:** Resolveria autenticidade, mas exigiria que o cliente implementasse validação de JWT (biblioteca específica, gerenciamento de chave pública). HMAC-SHA256 é simétrico e mais simples de verificar: o cliente só precisa repetir a operação de hash com a secret que já possui. Menor barreira de adoção.

## Consequências

### Positivas

- **Segurança por endpoint:** isolamento de secrets entre clientes.
- **Padrão de mercado:** clientes B2B já conhecem HMAC-SHA256, reduzindo fricção na integração.
- **Rotação segura:** grace period de 24h evita janela de indisponibilidade durante a troca de secret.
- **Verificável pelo cliente:** qualquer cliente pode validar a assinatura com uma operação de HMAC-SHA256 padrão.

### Negativas

- **Complexidade de estado:** secrets antigas precisam ser armazenadas durante o grace period (campo `previous_secret` + `secret_rotated_at` na tabela).
- **Custo de verificação no cliente:** embora pequeno, o cliente precisa implementar a verificação de assinatura. Mitigação: documentação clara com exemplos de código no portal do desenvolvedor.
- **Sem suporte a algoritmos assimétricos:** HMAC é simétrico; se o cliente expuser a secret, um atacante pode tanto verificar quanto gerar assinaturas. Algoritmos assimétricos (Ed25519, RSA) evitariam isso, mas foram considerados complexidade excessiva para a fase atual.

## Referências

- Transcrição: [09:19-09:24] Sofia, [09:44-09:45] Diego/Sofia/Diego
- Código relacionado:
  - `src/modules/orders/order.schemas.ts` — padrão de schemas Zod com validação
  - `src/middlewares/validate.middleware.ts` — middleware de validação reutilizável
