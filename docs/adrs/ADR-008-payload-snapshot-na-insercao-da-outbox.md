# ADR-008: Payload como snapshot renderizado na inserção da outbox

**ID:** ADR-008

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53)
- **Data do registro:** 2026-10-11
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior), Bruno (Eng. Pleno); limite de 64KB com Sofia (Segurança)

## Status

Aceito. O snapshot foi decidido em [09:52] Larissa ("Eu prefiro renderizado já, na hora da inserção"), [09:52] Diego ("snapshot na inserção") e [09:52] Bruno ("Beleza, snapshot. Decidido."). O UUID como id foi decidido em [09:51] Larissa. O limite de 64KB foi registrado em [09:24] Larissa, como requisito não funcional. O formato enxuto sem itens veio de [09:43] Diego e [09:44] Bruno.

## Contexto

A outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) guarda o evento até o worker enviá-lo ([ADR-002](ADR-002-worker-separado-em-polling.md)), o que pode levar de 2 segundos a muitas horas com retry ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)). Em [09:51] Bruno perguntou se a linha guarda o payload já renderizado ou só `order_id`, renderizando na hora do envio. Também faltava definir o conteúdo do payload, o tamanho máximo ([09:23] Sofia) e o tipo do id da tabela ([09:51] Diego). No código, os ids do projeto são `String @id @default(uuid()) @db.Char(36)` (`prisma/schema.prisma`) e o `Order` tem `orderNumber`, `customerId`, `status` e `totalCents`.

## Decisão

1. **Snapshot na inserção.** O payload é renderizado e gravado na linha da outbox no momento da inserção, que ocorre na transação do `changeStatus` ([ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)). Assim, "se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou" ([09:52] Larissa). A outbox não guarda só `order_id`.
2. **Conteúdo, enxuto e sem `items`.** JSON com `event_id`, `event_type` ("order.status_changed"), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order, como `total_cents` ([09:43] Diego). Não inclui `items`: o cliente consulta `GET /orders/:id` se quiser detalhes ([09:43] Diego). **Hipótese**: lista exata dos "campos básicos" além de `total_cents` e formato dos nomes, a fechar no FDD.
3. **Limite de 64KB com erro.** O evento acima de 64KB não é enviado e gera erro; **não trunca** ([09:23] Sofia, [09:24] Diego, [09:24] Larissa). É um requisito não funcional, não uma decisão arquitetural separada ([09:24] Larissa). **Hipótese**: em que ponto o erro é levantado e se ele aborta a mudança de status (a inserção ocorre dentro da transação, [ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)); a reunião não decidiu, fica para o FDD e o RFC.
4. **UUID como id** das linhas da outbox, não auto incremento, seguindo o padrão do projeto: "Tudo é uuid" ([09:51] Larissa). O UUID do evento é gerado na entrada da outbox e alimenta o `X-Event-Id` ([09:25] Diego, [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)); **Hipótese** a fechar no FDD: se esse UUID coincide com o `id` da linha da outbox ou é uma coluna separada.

HTTPS obrigatório para a URL do webhook é só uma validação no schema Zod, "nem é decisão arquitetural" ([09:23] Sofia), e portanto não é parte desta decisão.

## Alternativas Consideradas

### Guardar só `order_id` e renderizar o payload na hora do envio

Rejeitada. Se o pedido mudar entre a inserção e o envio, o evento refletiria um estado posterior, o que gera "caso esquisito" ([09:52] Larissa). [09:52] Diego concordou.

### Truncar o payload quando exceder o limite

Rejeitada. [09:23] Sofia defendeu erro: "Se chegou nesse tamanho, tem algo errado". [09:24] Larissa registrou erro caso ultrapasse.

### Incluir `items` no payload

Registrada no [RFC](../RFC.md) como RFC-ALT-13. Rejeitada para evitar inflar o evento: "Não manda items pra não inflar" ([09:43] Diego), "mantém payload enxuto" ([09:44] Bruno).

### Id auto incremental na outbox

Rejeitada. [09:51] Diego perguntou; [09:51] Larissa escolheu UUID por consistência com o projeto.

## Consequências

### Positivas

- O evento representa de forma fiel o estado do pedido no instante da mudança, independentemente de retries tardios ([09:52] Larissa).
- O worker não precisa consultar o pedido ao enviar, o que simplifica o envio. **Hipótese**: não depender de leituras no momento do envio, derivada de [09:51] Bruno.
- Payload enxuto reduz tamanho e risco de ultrapassar 64KB ([09:43] Diego, [09:24] Diego: "Nenhum evento nosso vai chegar perto disso").
- Ids UUID mantêm a convenção do projeto (`prisma/schema.prisma`).

### Negativas

- A renderização acontece na transação de `changeStatus`, aumentando seu custo ([ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)).
- O payload fica armazenado em duplicidade na outbox (e depois na DLQ, [09:18] Diego), ocupando espaço; arquivamento de linhas entregues está fora do escopo ([09:08] Diego).
- Clientes que precisam dos itens fazem uma segunda chamada a `GET /orders/:id` ([09:43] Diego).
- Um erro por excesso de 64KB pode abortar a mudança de status, caso o FDD mantenha o erro dentro da transação (ponto em aberto, ver Decisão item 3).

### Trade-off

Escolhe-se fidelidade ao momento da mudança, previsibilidade de tamanho e um evento autocontido, em troca de mais dados armazenados e de trabalho extra na transação, e em troca de o cliente precisar buscar os itens por outra chamada ([09:43] Diego, [09:52] Larissa).

## Referências

- [09:23] Sofia, [09:24] Diego, [09:24] Larissa, [09:43] Diego, [09:44] Bruno, [09:51] Diego, [09:51] Larissa, [09:51] Bruno, [09:52] Larissa, [09:52] Diego, [09:52] Bruno em `TRANSCRICAO.md`.
- `prisma/schema.prisma`: ids `String @id @default(uuid()) @db.Char(36)`, model `Order` (`orderNumber`, `customerId`, `totalCents`).
- `src/modules/orders/order.service.ts`: `changeStatus`, transação em que o snapshot é gravado ([ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)).
- `src/modules/orders/order.schemas.ts`: `orderIdParamSchema` (uuid), padrão de schemas Zod ([ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md)).

## Relacionados

- [ADR-001: Outbox no MySQL](ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker separado em polling](ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005: Entrega at-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões do projeto](ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007: Publicação do evento na transação do changeStatus](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)
