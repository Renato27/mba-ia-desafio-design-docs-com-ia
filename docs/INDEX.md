# Índice da documentação: Webhooks de Notificação de Pedidos

Pacote de design docs da feature, gerado a partir de `TRANSCRICAO.md` e do código existente. Ordem sugerida de leitura: PRD, RFC, ADRs, FDD e, por fim, o Tracker.

- [PRD](PRD.md): por que e o quê.
- [RFC](RFC.md): proposta técnica para revisão.
- [FDD](FDD.md): como construir, em detalhe.
- [TRACKER](TRACKER.md): de onde veio cada item.

## ADRs (`docs/adrs/`)

Cada decisão arquitetural fechada na reunião técnica é registrada em um arquivo individual, no formato MADR, nomeado `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Decisão | Origem principal |
| --- | --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | [09:06] Diego, [09:08] Larissa |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, em polling de 2s | [09:10] Larissa, [09:11] Diego |
| [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial e DLQ | [09:17] Larissa, [09:18] Diego |
| [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação | [09:22] Sofia |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` | [09:26] Larissa |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto | [09:30] Larissa |
| [ADR-007](adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md) | Publicação do evento na transação do `changeStatus` | [09:40] Bruno, [09:41] Diego |
| [ADR-008](adrs/ADR-008-payload-snapshot-na-insercao-da-outbox.md) | Payload em snapshot na inserção da outbox | [09:52] Larissa |
