# ADR-006: Reuso dos Padrões Existentes do Projeto no Módulo de Webhooks

**Status:** Aceito

**Data:** 2026-07-14

**Decisores:** Bruno (Eng. Pedidos), Larissa (Tech Lead), Diego (Eng. Plataforma)

---

## Contexto

O código base do OMS estabelece convenções fortes em todos os módulos existentes (`auth`, `users`, `customers`, `products`, `orders`):

- **Estrutura modular:** cada domínio em `src/modules/<nome>/` com `controller.ts`, `service.ts`, `repository.ts`, `routes.ts` e `schemas.ts`.
- **Classes de erro:** hierarquia baseada em `AppError` (`src/shared/errors/app-error.ts`) com subclasses como `NotFoundError`, `ConflictError`, `InvalidStatusTransitionError`, `InsufficientStockError`. Cada erro tem `errorCode` string padronizado.
- **Logging:** Pino (`src/shared/logger/index.ts`) com redação de dados sensíveis e log estruturado em JSON.
- **Validação:** Zod schemas por módulo, aplicados via middleware `validate` (`src/middlewares/validate.middleware.ts`).
- **Middleware de erro centralizado:** `errorMiddleware` em `src/middlewares/error.middleware.ts` que trata `AppError`, `ZodError` e `PrismaClientKnownRequestError` uniformemente.
- **Autenticação/autorização:** JWT com `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`.
- **Resposta paginada:** utilitário `paginated()` em `src/shared/http/response.ts`.

Criar o módulo de webhooks com padrões diferentes dos demais introduziria inconsistência e aumentaria a carga cognitiva de manutenção.

## Decisão

**O módulo de webhooks seguirá rigidamente os mesmos padrões, convenções e infraestrutura compartilhada dos módulos existentes.** Nenhuma nova biblioteca de logging, erro, validação ou autenticação será introduzida.

Especificamente:

| Aspecto | Reuso |
|---------|-------|
| Estrutura de diretórios | `src/modules/webhooks/` com `controller.ts`, `service.ts`, `repository.ts`, `routes.ts`, `schemas.ts` |
| Classes de erro | Estender `AppError` com prefixo `WEBHOOK_` (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`) |
| Logging | Usar o logger Pino já instanciado em `src/shared/logger/index.ts` |
| Validação | Zod schemas em `webhooks.schemas.ts`, aplicados via middleware `validate` |
| Error handling | O `errorMiddleware` existente trata automaticamente `AppError`, `ZodError` e `Prisma` |
| Autenticação | `authenticate` para todos os endpoints de webhook; `requireRole('ADMIN')` para endpoints administrativos (DLQ replay) |
| Paginação | `paginated()` do `src/shared/http/response.ts` para listagem de deliveries |
| UUID | Biblioteca `uuid` já presente no `package.json` (v11.0.3) |
| Prisma | Mesmo `PrismaClient`, mesmo `DATABASE_URL` |

## Alternativas Consideradas

### Alternativa 1: Módulo com padrões próprios (descartada)

**Trade-off:** Liberdade para escolher padrões diferentes que poderiam ser mais adequados ao domínio de webhooks (ex: usar uma biblioteca de fila, logger diferente, padrão de erro diferente). Mas a inconsistência entre módulos tornaria o código mais difícil de manter e onboard. Bruno defendeu "seguir igual pra webhook" ([09:28] Bruno) e Larissa formalizou como decisão ([09:30] Larissa).

### Alternativa 2: Microserviço separado para webhooks (descartada)

**Trade-off:** Isolaria completamente o domínio de webhooks, com stack independente. Mas o time é pequeno e opera um monólito modular. Um microserviço adicional significaria: novo repositório, novo pipeline de CI/CD, comunicação entre serviços, autenticação serviço-a-serviço. A complexidade adicional não se justifica para uma feature que é essencialmente um consumidor de eventos internos + chamadas HTTP. O padrão de módulo dentro do mesmo repositório atinge o isolamento necessário sem o custo operacional de um serviço separado.

## Consequências

### Positivas

- **Consistência total:** qualquer desenvolvedor que conhece o módulo de `orders` entende o de `webhooks` imediatamente.
- **Zero curva de aprendizado para infra compartilhada:** erro, log, validação e auth já estão prontos e testados.
- **Revisão de código mais rápida:** padrões previsíveis facilitam CR.
- **Erros do módulo de webhooks são automaticamente tratados:** o `errorMiddleware` centralizado não precisa de alteração.

### Negativas

- **Acoplamento ao monólito:** o módulo de webhooks compartilha o ciclo de deploy da API. Se no futuro o volume de webhooks crescer a ponto de precisar escalar independentemente, a extração para serviço separado será mais trabalhosa do que se tivesse sido construído separado desde o início. Esse risco foi aceito — a extração futura é possível (o módulo já é bem delimitado) e o volume atual não justifica a complexidade adicional.
- **Convenções podem não ser ideais para webhooks:** por exemplo, o padrão controller-service-repository é voltado para CRUD, enquanto o processamento de webhooks é mais orientado a eventos. Adaptações serão necessárias (ex: `webhook.processor.ts` como arquivo adicional), mas a espinha dorsal permanece a mesma.

## Referências

- Transcrição: [09:27-09:30] Bruno, Diego, Larissa
- Código relacionado:
  - `src/modules/orders/` — estrutura modular de referência (controller, service, repository, routes, schemas)
  - `src/shared/errors/app-error.ts` — classe base `AppError`
  - `src/shared/errors/http-errors.ts` — hierarquia de erros HTTP
  - `src/middlewares/error.middleware.ts` — error handler centralizado
  - `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`
  - `src/middlewares/validate.middleware.ts` — validação Zod
  - `src/shared/logger/index.ts` — logger Pino
  - `src/shared/http/response.ts` — utilitário de paginação
  - `src/routes/index.ts` — composição de rotas
  - `src/app.ts` — composição de dependências (`buildControllers`)
