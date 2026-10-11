# ADR-005: Entrega at-least-once com X-Event-Id para deduplicação no cliente

**ID:** ADR-005

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53)
- **Data do registro:** 2026-10-11
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Sofia (Segurança), Bruno (Eng. Pleno), Marcos (PM)

## Status

Aceito. O fechamento específico é de [09:26] Larissa ("At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."). O resumo geral de [09:48] Larissa o reapresentou ("Idempotência por X-Event-Id, garantia at-least-once"), e o aceite de [09:49] Diego, [09:49] Bruno, [09:49] Sofia e [09:49] Marcos é do resumo como um todo.

## Contexto

O worker separado ([ADR-002](ADR-002-worker-separado-em-polling.md)) lê a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e faz a chamada HTTP ao cliente, com retry e DLQ ([ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md)). O timeout de 10 s ([09:42] Diego) é tratado como falha e marcado para retry. Que um timeout leve a reenvio de um evento que o cliente talvez já tenha processado é inferência deste documento (**Hipótese**). A reunião afirma que o mesmo evento pode chegar duas vezes ao cliente ([09:24] Diego: "Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."). Foi preciso definir a garantia de entrega e como o cliente distingue duplicatas ([09:25] Bruno: "E como ele diferencia?").

## Decisão

1. A plataforma garante entrega **at-least-once** ([09:24] Diego).
2. Cada requisição leva o header `X-Event-Id`, um UUID gerado quando o evento entra na outbox, único por evento ([09:25] Diego). O payload também traz um campo `event_id` ([09:43] Diego); que seja o mesmo UUID do header é a leitura natural, mas a reunião não o afirmou (ver **Hipótese** abaixo).
3. A **deduplicação é responsabilidade do cliente**, que usa o `X-Event-Id` ([09:25] Diego, [09:25] Sofia: "Isso joga responsabilidade pro cliente."). Isso será documentado de forma destacada no portal de desenvolvedor ([09:26] Marcos).
4. **Exactly-once foi descartado**: exigiria coordenação dos dois lados e seria muito mais complexo ([09:25] Diego).

**Hipótese** (não decidida na reunião, a fechar no FDD): se o `X-Event-Id`, o `event_id` do payload e o `id` da linha na outbox são o mesmo valor ou campos separados. A reunião só define que o UUID é gerado na entrada da outbox e que os ids do projeto são UUID ([09:51] Larissa).

## Alternativas Consideradas

### Exactly-once

Descartada. Exigiria coordenação entre plataforma e cliente e é "muito mais complexo" ([09:25] Diego).

### At-most-once (sem retry de entregas incertas)

Não discutida na reunião. Plausível, mas incompatível com o retry com backoff e DLQ decididos em [09:17] Larissa, pois eventos seriam perdidos em falhas transitórias do cliente.

### At-least-once sem identificador de evento

Não discutida na reunião. Plausível, mas deixaria o cliente sem meio de distinguir duplicatas, o que contraria [09:25] Bruno e [09:25] Diego.

## Consequências

### Positivas

- Padrão de mercado, citado como usado por Stripe e GitHub ([09:25] Diego); "resolve 99% dos casos" ([09:25] Diego).
- Combina com o retry e a DLQ: evento committado não se perde silenciosamente por falha do cliente, pois vai para retry e depois DLQ ([09:17] Larissa, [09:18] Diego; análise deste ADR).
- Evita coordenação de estado entre os dois lados.

### Negativas

- Clientes podem receber duplicatas e precisam implementar deduplicação ([09:24] Diego).
- Transfere responsabilidade ao cliente ([09:25] Sofia), o que exige documentação clara no portal ([09:26] Marcos).

### Trade-off

Evita a complexidade e a coordenação do exactly-once; aceita duplicatas e transfere a dedup ao cliente ([09:25] Diego, [09:25] Sofia), que a faz por `X-Event-Id`. A decisão foi fechada em [09:26] Larissa.

## Referências

- [09:24] Diego, [09:25] Bruno, [09:25] Diego, [09:25] Sofia, [09:26] Marcos, [09:26] Larissa, [09:44] Diego, [09:48] Larissa em `TRANSCRICAO.md`.
- `prisma/schema.prisma`: ids do projeto são `String @id @default(uuid()) @db.Char(36)`, convenção que o UUID do evento acompanha ([09:51] Larissa).
- `package.json`: dependência `uuid` 11.0.3 já presente.

## Relacionados

- [ADR-001: Outbox no MySQL](ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker separado em polling](ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](ADR-003-retry-backoff-exponencial-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-006: Reuso dos padrões do projeto](ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007: Publicação do evento na transação do changeStatus](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md)
- [ADR-008: Payload snapshot na inserção da outbox](ADR-008-payload-snapshot-na-insercao-da-outbox.md)
