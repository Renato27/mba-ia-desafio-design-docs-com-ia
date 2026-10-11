# ADR-007: Publicação do evento na mesma transação do changeStatus

**ID:** ADR-007

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53)
- **Data do registro:** 2026-10-11
- **Decisores:** Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Marcos (PM, filtro de eventos)

## Status

Aceito. Proposto por [09:40] Bruno, com concordância de [09:41] Diego ("Essencial. Se ficar fora da transação, perde a garantia toda.") e [09:41] Diego sobre a função recebendo o `tx`. O filtro na inserção foi proposto em [09:34] Bruno, com [09:34] Diego concordando. O resumo de [09:48] Larissa registra "transação atômica com mudança de status".

## Contexto

O padrão outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) só garante consistência se a linha da outbox for gravada na mesma transação SQL que muda o pedido ([09:06] Diego). No código, `changeStatus(id, input, userId)` em `src/modules/orders/order.service.ts` roda tudo em `this.prisma.$transaction(async (tx) => {...})`:

- busca o pedido com itens (`tx.order.findUnique`, linhas 132 a 135) e valida `from === to` e `canTransition` (linhas 138 a 149; `canTransition` e a máquina de estados `transitions` vêm de `src/modules/orders/order.status.ts`);
- debita ou repõe estoque (linhas 151 a 156);
- `tx.order.update` e `tx.orderStatusHistory.create` (linhas 158 a 167);
- relê o pedido com relações e retorna (linhas 169 a 177).

[09:04] Bruno já descreveu essa transação como "pesada", e uma chamada HTTP no meio travaria outros pedidos. Faltava definir onde e como o evento entra nessa transação sem acoplar `OrderService` ao módulo de webhooks, e que eventos entram na outbox.

## Decisão

1. **Inserção atômica.** A inserção na `webhook_outbox` acontece **dentro** da transação de `changeStatus`. Se a inserção falhar, a transação inteira sofre rollback: "Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno, [09:41] Diego).
2. **Função de publicação.** Será criada `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `tx` da transação atual ([09:41] Bruno). É "função pura recebendo o tx. Não precisa injetar repository inteiro" ([09:41] Diego); o `OrderService` apenas a chama. Isso evita passar um repository de webhook ao construtor `OrderService(orders, prisma)` (`src/app.ts`, `buildControllers`).
3. **Ponto de chamada.** Dentro do callback de `$transaction`, depois de `tx.orderStatusHistory.create` e antes do `tx.order.findUnique` final. **Hipótese** do FDD sobre a posição exata, derivada de [09:40] Bruno ("update na order, insere no history e atualiza estoque").
4. **Filtro de eventos na inserção.** Cada webhook tem uma lista de status que quer ouvir ([09:33] Marcos). O filtro é aplicado na hora de inserir: se nenhum webhook do customer quer aquele status, nenhuma linha entra na outbox ([09:34] Bruno, [09:34] Diego). **Hipótese**: `publishWebhookEvent` consulta as configurações do customer usando o próprio `tx` (a reunião não detalha), e uma linha de outbox por endpoint interessado. A forma exata fica no FDD.
5. **Escopo do evento.** O evento só se aplica a `changeStatus`. A reunião não tratou o `create` do pedido (`create` em `src/modules/orders/order.service.ts`, linhas 50 a 124, cria o pedido já `PENDING` com histórico de `fromStatus: null` e não passa por `changeStatus`). **Ponto em aberto**, não decisão: se a criação do pedido deve emitir evento, a ser tratado no FDD e no RFC.

## Alternativas Consideradas

### Chamada HTTP síncrona dentro do `changeStatus`

Descartada. Um cliente lento travaria mudanças de status de outros pedidos, e cliente fora do ar não poderia causar rollback ([09:04] Bruno, [09:04] Larissa, [09:06] Diego: "Síncrono está fora de questão").

### Publicar o evento depois do commit (fora da transação)

Não discutida como alternativa formal; rejeitada na hora por [09:41] Diego: "Se ficar fora da transação, perde a garantia toda". Inferência deste ADR, não dita na reunião: o processo poderia cair entre o commit e a gravação, deixando status mudado sem evento.

### Injetar o repository de webhook (ou serviço de enqueue) no `OrderService`

Considerada e preterida: [09:41] Bruno apontou que isso obrigaria passar um repository ou função de enqueue ao `OrderService`; [09:41] Diego preferiu a função recebendo o `tx`, sem injetar o repository inteiro.

### Filtrar só na hora do envio pelo worker

Considerada em [09:34] Diego ("Filtra na inserção do outbox ou na hora de mandar?") e rejeitada por [09:34] Bruno: filtrar na inserção "economiza linha na tabela".

## Consequências

### Positivas

- Sem inconsistência: commit implica evento registrado, rollback implica evento descartado ([09:06] Diego).
- `OrderService` fica pouco acoplado ao módulo de webhooks: só importa a função de publicação, sem novo parâmetro de construtor (`src/app.ts`, `buildControllers`).
- A outbox cresce menos, pois status não desejados nem entram ([09:34] Bruno).

### Negativas

- Alteração crítica em código existente e testado (`src/modules/orders/order.service.ts`): exige testes ponta a ponta ([09:46] Larissa).
- A transação de `changeStatus` fica mais longa: o `tx` passa a incluir leitura de configuração e escrita da outbox.
- Qualquer falha na inserção (inclusive erro por payload acima do limite, ver [ADR-008](ADR-008-payload-snapshot-na-insercao-da-outbox.md)) causa rollback da mudança de status ([09:40] Bruno). **Hipótese**: se um erro de tamanho de payload deve de fato abortar a mudança de status não foi decidido; fica para o FDD e o RFC.

### Trade-off

Troca-se um acoplamento leve e pontual entre `changeStatus` e a função de publicação, mais carga na transação crítica, por garantia forte de consistência entre mudança de status e evento ([09:41] Diego). Aceita-se que a falha de webhook possa derrubar a mudança de status em vez de aceitar eventos perdidos.

## Referências

- [09:04] Bruno, [09:04] Larissa, [09:06] Diego, [09:33] Marcos, [09:34] Diego, [09:34] Bruno, [09:40] Bruno, [09:41] Diego, [09:41] Bruno, [09:46] Larissa, [09:48] Larissa em `TRANSCRICAO.md`.
- `src/modules/orders/order.service.ts`: `changeStatus` (linhas 126 a 179, transação em 131 a 178), `create` (linhas 50 a 124).
- `src/modules/orders/order.status.ts`: `transitions`, `canTransition`.
- `src/app.ts`: `buildControllers` instancia `OrderService(orderRepository, prisma)`.
- `src/middlewares/error.middleware.ts` e `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError`, `ConflictError`): erros do `changeStatus` continuam passando pelo `errorMiddleware` ([ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)).
- `src/routes/index.ts` e `src/modules/orders/order.routes.ts`: `PATCH /:id/status` com `validate({ params, body })` chama `controller.changeStatus`.

## Relacionados

- [ADR-001: Outbox no MySQL](ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker separado em polling](ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005: Entrega at-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões do projeto](ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-008: Payload snapshot na inserção da outbox](ADR-008-payload-snapshot-na-insercao-da-outbox.md)
