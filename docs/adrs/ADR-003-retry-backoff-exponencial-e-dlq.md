# ADR-003: Retry com backoff exponencial (5 re-tentativas, Hipótese, ver RFC-OPEN-07) e DLQ em tabela separada

**ID:** ADR-003

- **Status:** Aceito (decisão da reunião); a interpretação do contador de tentativas está Em aberto (RFC-OPEN-07)
- **Data:** 2026-10-11 (data do registro)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM), Sofia (Segurança, regras do replay)

## Status

Aceito (decisão da reunião; a interpretação do contador de tentativas está Em aberto, RFC-OPEN-07). Larissa registrou em [09:17] ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h") e confirmou o replay de DLQ com role ADMIN em [09:36]. Resumo em [09:48].

## Contexto

Quando o endpoint do cliente está fora do ar ou lento, o worker ([ADR-002](ADR-002-worker-separado-em-polling.md)) precisa decidir quando tentar de novo e quando desistir ([09:14] Larissa). Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). A plataforma só envia eventos (outbound) e o evento está registrado na outbox ([ADR-001](ADR-001-outbox-no-mysql.md)).

## Decisão

1. **Retry com backoff exponencial e teto de tentativas.** Após o teto, a falha é permanente e o evento vai para a DLQ ([09:15] Diego).
2. **Cinco tentativas, com progressão 1 min, 5 min, 30 min, 2 h e 12 h** ([09:16] Larissa, [09:17] Diego, [09:17] Larissa). Marcos considerou aceitável o total de quase 15 horas ([09:17] Marcos).
3. **Interpretação do contador em aberto (RFC-OPEN-07):** [09:17] Larissa fechou "5 tentativas" e [09:48] Larissa resumiu "total 5 tentativas"; a reunião não definiu se a 1ª entrega conta. Os 5 intervalos 1m/5m/30m/2h/12h somam 14h36m, compatíveis com "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego), o que só ocorre se a 1ª entrega NÃO conta (1 original + 5 retries). **Hipótese** a confirmar com Diego e Larissa.
4. **Timeout do HTTP do worker: 10 segundos.** Cliente que não responde em 10 s é tratado como falha e marcado para retry ([09:42] Diego).
5. **DLQ em tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego). No desenho do FDD a linha FAILED permanece na outbox como espelho e a `webhook_dead_letter` é a fonte de evidência e replay (**Hipótese** (FDD), ver FDD-FLX-07).
6. **Reprocessamento manual** por endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego). Exige **role ADMIN** e **loga quem fez o replay** (auditoria) ([09:36] Sofia). Reaproveita o `requireRole` existente ([09:36] Larissa).

Fora desta decisão: aviso por email ao cliente quando o webhook falha, que ficou fora de escopo ("talvez próxima fase", [09:37] Larissa), e rate limiting de saída, que fica como ponto em aberto ([09:39] Larissa). Como o agendamento da próxima tentativa é guardado na outbox (por exemplo, uma coluna de próxima tentativa) não foi definido; **Hipótese** a detalhar no FDD.

## Alternativas Consideradas

1. **Retry indefinido com backoff.** Rejeitada. Deixa evento pendurado para sempre se o cliente sumiu. Fonte: [09:15] Diego.
2. **Apenas 3 tentativas ("mais agressivo").** Rejeitada. Três tentativas em cerca de 30 minutos descartariam o evento de um cliente com indisponibilidade de manhã ou de 2 horas em manutenção planejada. Fonte: [09:16] Bruno (propôs) e [09:16] Diego (rejeitou).
3. **Marcar como "failed" na própria outbox, sem tabela de DLQ.** Rejeitada em favor de tabela separada: leitura da outbox principal mais limpa e evidência para debug e reprocessamento. Fonte: [09:17] Larissa (levantou a opção) e [09:18] Diego. No desenho do FDD a linha FAILED permanece na outbox como espelho e a `webhook_dead_letter` é a fonte de evidência e replay (**Hipótese** (FDD), ver FDD-FLX-07).

## Consequências

### Positivas

- Janela de recuperação de cerca de 14h36m cobre manutenções planejadas de 2 h, como a que um cliente já teve ([09:16] Diego, [09:17] Diego).
- Eventos não somem nem ficam pendurados: ou são entregues, ou ficam na DLQ para análise e replay ([09:18] Diego).
- A outbox continua simples de ler (só pendentes) e a DLQ serve de evidência de debug ([09:18] Diego).
- Replay restrito a ADMIN e auditado reduz risco operacional ([09:36] Sofia).

### Negativas

- Evento de um cliente fora do ar por mais de cerca de 15 horas só será entregue por replay manual ([09:17] Marcos aceitou esse limite).
- O cliente não é avisado automaticamente das falhas: email está fora de escopo desta fase ([09:37] Larissa).
- Uma tabela a mais (`webhook_dead_letter`) e um endpoint admin a manter ([09:18] Diego).
- Manter FAILED como espelho na outbox (**Hipótese** (FDD)) custa um estado a mais na outbox, em troca de rastrear na própria outbox o desfecho de cada evento, enquanto a DLQ guarda a evidência.
- A reunião não tratou a interação entre retry e ordering (um evento em retry e um evento posterior do mesmo pedido). **Hipótese** de risco, a analisar no FDD, dado que o ordering é só por `order_id` e single-worker ([09:12] Diego, [09:13] Larissa).

### Trade-off

Escolhemos um **teto finito de 5 re-tentativas (Hipótese, ver RFC-OPEN-07; cerca de 14h36m)** em vez de retry indefinido: evitamos eventos pendurados para sempre, mas aceitamos que indisponibilidades mais longas exijam replay manual por um ADMIN. Cinco tentativas (contagem em aberto), e não três, protegem contra manutenções longas ao custo de manter eventos em retry por muitas horas ([09:15] Diego, [09:16] Diego).

## Referências

- Transcrição: [09:14] Larissa, [09:15] Diego, [09:15] Bruno, [09:16] Bruno, [09:16] Diego, [09:16] Larissa, [09:17] Diego, [09:17] Marcos, [09:17] Larissa, [09:18] Diego, [09:18] Bruno, [09:19] Larissa, [09:35] Diego, [09:36] Sofia, [09:36] Larissa, [09:37] Larissa, [09:39] Larissa, [09:42] Diego, [09:48] Larissa.
- Código: `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`), `src/modules/users/user.routes.ts` (único uso atual de `requireRole('ADMIN')`), `src/routes/index.ts` (`buildApiRouter`, onde a rota admin entraria; no código as rotas ficam sob `/api/v1`, **Hipótese** de montagem), `src/shared/logger/index.ts` (Pino, base do log de auditoria do replay), `prisma/schema.prisma` (novas tabelas `webhook_outbox` e `webhook_dead_letter` ainda não existem, a criar).
- Documentos do desafio: `docs/FDD.md`, `docs/RFC.md` (RFC-OPEN-07, confirmação da interpretação de "5 tentativas").

## Relacionados

- [ADR-001](ADR-001-outbox-no-mysql.md): outbox para onde o replay recoloca o evento.
- [ADR-002](ADR-002-worker-separado-em-polling.md): worker que executa as tentativas.
- [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md): cada tentativa é assinada.
- [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md): retries e replay geram reentregas, por isso o `X-Event-Id` para dedup.
- [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md): reuso de `requireRole` e Pino.
- [ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md) e [ADR-008](ADR-008-payload-snapshot-na-insercao-da-outbox.md): origem e conteúdo do evento reprocessado.
