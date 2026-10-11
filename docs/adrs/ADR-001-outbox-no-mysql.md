# ADR-001: Padrão outbox no MySQL existente para eventos de webhook

**ID:** ADR-001

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53)
- **Data:** 2026-10-11 (data do registro)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)

## Status

Aceito. A decisão foi fechada por Larissa em [09:08] ("Tá decidido então: outbox em MySQL") e reconfirmada no resumo de [09:48] ("Padrão outbox no MySQL, transação atômica com mudança de status").

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status de um pedido muda. Hoje eles fazem polling em `GET /orders`, o que deixa a integração lenta e cara ([09:00] Marcos). "Tempo real" para eles significa menos de 10 segundos ([09:02] Marcos). O envio é só de saída (outbound) ([09:02] Marcos, [09:03] Sofia).

Larissa abriu a questão: disparar o webhook de forma síncrona no service de orders ou usar fila/outbox ([09:03] Larissa). No código atual, `OrderService.changeStatus` em `src/modules/orders/order.service.ts` executa tudo dentro de `this.prisma.$transaction(async (tx) => { ... })`: lê o pedido, valida a transição, ajusta estoque, faz `tx.order.update` e `tx.orderStatusHistory.create`. O banco é MySQL (`prisma/schema.prisma`, `datasource db`), com cliente Prisma criado em `src/config/database.ts` (`createPrismaClient`).

## Decisão

Adotar o padrão **outbox no MySQL já existente**. Na mesma transação SQL que atualiza `orders` e `order_status_history`, o sistema insere uma linha na tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego). Se a transação comita, o evento está registrado; se dá rollback, o evento some junto ([09:06] Diego).

Pontos fechados na reunião:

- A tabela tem índice no campo de status (pendente, processando, falhou, entregue) e em `created_at`; o worker lê só os pendentes, em batch pequeno ([09:08] Diego).
- No código, o ponto de integração é o método `changeStatus` de `src/modules/orders/order.service.ts`, dentro do callback de `$transaction` ([09:40] Bruno). O detalhe da chamada (`publishWebhookEvent(tx, ...)`) é decisão registrada no ADR-007 e detalhada no FDD.
- Os ids da outbox seguem o padrão do projeto, UUID ([09:51] Larissa). No `prisma/schema.prisma` os ids são `String @id @default(uuid()) @db.Char(36)`.

Arquivamento das linhas entregues (cerca de 30 dias) fica **fora do escopo** desta feature ([09:08] Diego); é questão em aberto, não decisão.

## Alternativas Consideradas

1. **Disparo HTTP síncrono dentro de `changeStatus`.** Rejeitada. A transação de status já é pesada (atualiza `orders`, insere em `order_status_history`, decrementa `stock_quantity`), e um cliente lento travaria a mudança de status de outros pedidos. Se o cliente estiver fora do ar, a plataforma não pode dar rollback no status. Fonte: [09:04] Bruno, [09:04] Larissa (concordou) e [09:06] Diego ("Síncrono está fora de questão").
2. **Redis Streams ou fila externa semelhante.** Rejeitada. Exigiria subir mais infraestrutura; para um time pequeno, Redis Cluster seria overengineering, e o outbox no MySQL existente resolve. Fonte: [09:07] Larissa (levantou a alternativa) e [09:07] Diego.

## Consequências

### Positivas

- Atomicidade: não existe caso de status mudar sem evento registrado, nem de evento registrado com rollback do status ([09:06] Diego, [09:40] Bruno, [09:41] Diego).
- Nenhuma infraestrutura nova: reaproveita o MySQL, o Prisma e a stack existentes ([09:07] Diego).
- A mudança de status do pedido não depende da disponibilidade do endpoint do cliente ([09:04] Bruno).

### Negativas

- A tabela cresce e disputa o mesmo MySQL da API (análise deste ADR, **Hipótese**; a reunião só trata do crescimento e dos índices). Mitigação parcial por índices e batch pequeno ([09:07] Bruno perguntou sobre performance, [09:08] Diego respondeu). A retenção/arquivamento não está definida (fora do escopo, [09:08] Diego).
- A transação de `changeStatus` ganha mais uma escrita, e uma falha na inserção da outbox faz rollback da mudança de status ([09:40] Bruno).
- O evento só sai depois que o worker o lê, o que depende do ADR-002.

### Trade-off

Trocamos a simplicidade de um HTTP direto (sem tabela nem worker) por **consistência garantida entre o status do pedido e o evento**, ao custo de manter uma tabela extra, um processo worker e mais carga no MySQL. Foi aceito também preferir o MySQL existente a uma fila dedicada, em nome de menos infraestrutura ([09:07] Diego). Que isso signifique abrir mão das capacidades de um broker é análise deste ADR (**Hipótese**).

## Referências

- Transcrição: [09:00] Marcos, [09:03] Larissa, [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:07] Diego, [09:08] Diego, [09:08] Larissa, [09:40] Bruno, [09:41] Diego, [09:48] Larissa, [09:51] Larissa.
- Código: `src/modules/orders/order.service.ts` (método `changeStatus`, bloco `$transaction`), `src/config/database.ts` (`createPrismaClient`, `prisma`), `prisma/schema.prisma` (MySQL, ids UUID, `OrderStatusHistory`, `@@map` em snake_case), `src/server.ts` (modelo de bootstrap por processo).
- Documentos do desafio: `docs/FDD.md` (modelo de dados e fluxos), `docs/RFC.md`.

## Relacionados

- [ADR-002](ADR-002-worker-separado-em-polling.md): worker que lê a outbox.
- [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md): retry e DLQ sobre a outbox.
- [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md): `X-Event-Id` gerado quando o evento entra na outbox.
- [ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md): integração transacional no `changeStatus`.
- [ADR-008](ADR-008-payload-snapshot-na-insercao-da-outbox.md): payload em snapshot gravado na outbox.
- [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) e [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md): contexto de segurança e de padrões do projeto.
