# ADR-006: Reuso dos padrões do projeto para o módulo de webhooks

**ID:** ADR-006

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53)
- **Data do registro:** 2026-10-11
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Segurança), Marcos (PM)

## Status

Aceito. Fechado em [09:30] Larissa: "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros." Reconfirmado no resumo de [09:48] Larissa, com aceite em [09:49] Diego e [09:49] Bruno.

## Contexto

A feature adiciona API de configuração, worker e endpoint admin de replay. Havia o risco de introduzir novas convenções de erro, log e estrutura de módulo. [09:27] Bruno apontou que "a gente tem um padrão claro na codebase". O código confirma o padrão:

- Cada domínio é um módulo em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas` (exemplo: `src/modules/orders/order.routes.ts`, `src/modules/orders/order.schemas.ts`).
- A composição repository, service, controller é feita em `buildControllers(prisma)` em `src/app.ts`, e os routers são montados em `buildApiRouter(controllers)` em `src/routes/index.ts` (tipo `Controllers`, `router.use('/orders', ...)`), sob o prefixo `/api/v1` (`app.use('/api/v1', ...)` em `src/app.ts`).
- Erros: `AppError(message, statusCode, errorCode, details?)` em `src/shared/errors/app-error.ts`, com subclasses em `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError(message, code, details)`, `UnprocessableEntityError(message, code, details)`, `BadRequestError(message, code, details)`, `InvalidStatusTransitionError`, `InsufficientStockError`).
- `errorMiddleware` em `src/middlewares/error.middleware.ts` converte `AppError`, `ZodError` e erros Prisma `P2002`/`P2025` em `{ error: { code, message, details? } }` e registra o restante com `logger.error`; é montado por último em `src/app.ts`.
- Validação: `validate({ body, query, params })` em `src/middlewares/validate.middleware.ts` converte `ZodError` em `ValidationError`.
- Autorização: `authenticate` e `requireRole(...roles)` em `src/middlewares/auth.middleware.ts`; hoje `requireRole('ADMIN')` só é usado em `src/modules/users/user.routes.ts`.
- Logger: Pino em `src/shared/logger/index.ts`, com `redact` de `authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken`.

## Decisão

Reusar, sem criar nada paralelo ([09:30] Larissa):

1. **Módulo** `src/modules/webhooks` (a criar) com controller, service, repository, routes e schemas, igual aos outros domínios ([09:27] Bruno, [09:28] Diego). O router entra em `buildApiRouter` (`src/routes/index.ts`) e o controller em `buildControllers` (`src/app.ts`). **Hipótese**: a forma exata de montagem, inclusive da rota admin, fica para o FDD. A reunião cita os paths sem o prefixo `/api/v1` (ex: [09:34] Marcos, [09:18] Diego).
2. **Erros** seguem `AppError` e suas subclasses, com prefixo `WEBHOOK_` em todos os códigos do módulo ([09:28] Bruno, [09:29] Larissa). Exemplos citados: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, "etc" ([09:28] Bruno). Seguem o estilo de `InvalidStatusTransitionError extends ConflictError` em `src/shared/errors/http-errors.ts`. **Hipótese**: lista completa de códigos e classe base de cada um, a fechar no FDD.
3. **Logger**: Pino existente, "não vamos botar nada novo" ([09:29] Bruno). **Hipótese** do FDD: incluir `secret` e assinatura nos `redact` de `src/shared/logger/index.ts`, motivada por [09:22] Diego (cliente já vazou secret em log), mas não decidida na reunião.
4. **Middleware de erro**: `errorMiddleware` "vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno). Nenhuma alteração em `src/middlewares/error.middleware.ts`.
5. **Schemas Zod e validação** com `validate(...)` ([09:30] Larissa). A regra de HTTPS obrigatório para a URL é só uma validação no schema Zod, "nem é decisão arquitetural" ([09:23] Sofia).
6. **Autorização**: o replay admin exige role `ADMIN` reaproveitando `requireRole` ([09:36] Sofia, [09:36] Larissa); o resto do CRUD usa só `authenticate` ([09:36] Marcos, [09:37] Sofia).
7. **Banco e Prisma**: mesmo banco e mesma `DATABASE_URL`; o worker usa um `PrismaClient` próprio porque "PrismaClient é por processo" ([09:30] Bruno), o que é viável com `createPrismaClient()` em `src/config/database.ts`. O worker segue o modelo do `src/server.ts` ([09:11] Larissa, ver [ADR-002](ADR-002-worker-separado-em-polling.md)).

## Alternativas Consideradas

### Nova convenção de erros/logs para o worker e a API de webhooks

Não discutida na reunião como alternativa formal. Rejeitada de forma implícita por [09:30] Larissa ("reuso máximo") e [09:29] Bruno ("Não vamos botar nada novo"): criaria dois padrões de erro e de log para o mesmo serviço.

### Biblioteca ou serviço externo de webhooks

Não discutida na reunião. Plausível, mas contraria o objetivo de manter a stack atual de time pequeno ([09:07] Diego, [09:11] Diego: "mesmo banco, mesma stack").

### Estrutura própria para o worker, fora de `src/modules`

Parcialmente discutida: [09:28] Bruno propôs `src/worker.ts` (a criar) como entry separada e a lógica de processamento dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`; o nome não foi fechado, [09:28] Diego: "Beleza."). Uma pasta isolada fora do módulo não foi proposta; manter a lógica no módulo mantém o padrão.

## Consequências

### Positivas

- Curva de aprendizado e revisão menores: o módulo é igual aos outros ([09:27] Bruno).
- O contrato de erro `{ error: { code, message, details? } }` fica uniforme para os clientes, sem mudar `src/middlewares/error.middleware.ts` ([09:29] Bruno).
- Reaproveita `requireRole`, `validate`, Pino e `AppError`, reduzindo código novo e risco.

### Negativas

- Acopla o módulo às convenções atuais: `NotFoundError` fixa o código `NOT_FOUND` (`src/shared/errors/http-errors.ts`), então um `WEBHOOK_NOT_FOUND` exige estender `AppError` diretamente ou outra subclasse com código configurável (`ConflictError`, `UnprocessableEntityError`, `BadRequestError`). **Hipótese**: escolha a fechar no FDD.
- O `errorMiddleware` só trata `AppError`, `ZodError` e Prisma `P2002`/`P2025`; erros do worker (fora do ciclo HTTP) não passam por ele. **Hipótese**: tratamento de erro do worker a detalhar no FDD.
- O logger atual não redige `secret` por padrão (ver `redactPaths` em `src/shared/logger/index.ts`).

### Trade-off

Escolhe-se consistência e velocidade de entrega (prazo de 3 sprints, [09:47] Larissa) em vez de liberdade para desenhar convenções ideais para webhooks. O custo é herdar as limitações do padrão atual, aceitas por [09:30] Larissa.

## Referências

- [09:27] Bruno, [09:28] Bruno, [09:28] Diego, [09:29] Larissa, [09:29] Bruno, [09:30] Bruno, [09:30] Larissa, [09:36] Larissa, [09:48] Larissa em `TRANSCRICAO.md`.
- `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`.
- `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`), `src/modules/users/user.routes.ts` (uso atual de `requireRole('ADMIN')`).
- `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`.
- `src/routes/index.ts` (`buildApiRouter`, `Controllers`), `src/app.ts` (`buildControllers`, `buildApp`), `src/config/database.ts`, `src/server.ts`.
- `src/modules/orders/order.routes.ts`, `src/modules/orders/order.schemas.ts` (padrão de módulo).

## Relacionados

- [ADR-001: Outbox no MySQL](ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker separado em polling](ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005: Entrega at-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-007: Publicação do evento na transação do changeStatus](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)
- [ADR-008: Payload snapshot na inserção da outbox](ADR-008-payload-snapshot-na-insercao-da-outbox.md)
