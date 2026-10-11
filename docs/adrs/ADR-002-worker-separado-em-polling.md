# ADR-002: Worker em processo separado com polling de 2 segundos

**ID:** ADR-002

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53)
- **Data:** 2026-10-11 (data do registro)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM, aceitou a latência)

## Status

Aceito. Larissa registrou a decisão em [09:10] ("Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos.") e a confirmou no resumo de [09:48].

## Contexto

Com o outbox no MySQL ([ADR-001](ADR-001-outbox-no-mysql.md)), é preciso definir como o worker descobre eventos novos e onde ele roda. O requisito de negócio é entregar em menos de 10 segundos ([09:02] Marcos).

No código atual, o processo da API é iniciado por `src/server.ts`: `buildApp({ prisma })`, `app.listen`, e shutdown por SIGINT/SIGTERM com `prisma.$disconnect()`. O cliente Prisma é um singleton por processo em `src/config/database.ts` (`prisma = createPrismaClient()`). Não existe hoje nenhum processo de background; os scripts de `package.json` são `dev`, `build`, `start`, `db:*`, `test`, `lint` e `format`.

## Decisão

1. **Polling em loop a cada 2 segundos**: o worker busca os eventos pendentes mais antigos, processa e marca ([09:09] Diego). A latência adicionada é de 2 s no pior caso, aceita por Larissa e Marcos ([09:10]).
2. **Processo separado da API**: se o worker rodasse dentro da API, um restart da API derrubaria o worker ([09:11] Diego). Nova entry-point `src/worker.ts` (a criar), no mesmo modelo de `src/server.ts`, e script `npm run worker` ([09:11] Larissa). Mesmo banco e mesma stack ([09:11] Diego).
3. **PrismaClient próprio do worker**: "PrismaClient é por processo", mesma `DATABASE_URL`, instância nova ([09:30] Bruno).
4. **Worker único (single-worker)**: ordenação por `created_at` da outbox, o que dá ordering implícita por `order_id`. Isso é **limitação conhecida**: não há ordering global, só por `order_id` e enquanto houver um único worker ([09:12] Diego, [09:13] Larissa).

O arquivo com a lógica de processamento ficará no módulo de webhooks (`src/modules/webhooks`, a criar); o nome (`webhook.worker.ts` ou `webhook.processor.ts`) **não foi fechado** ([09:28] Bruno). Novas variáveis de ambiente do worker, se necessárias, entrariam em `src/config/env.ts` (**Hipótese**, ver FDD).

## Alternativas Consideradas

1. **Trigger do banco para notificar o worker (reatividade).** Rejeitada. O MySQL não tem listener nativo como o NOTIFY/LISTEN do Postgres; trigger só executa SQL e não notifica processo externo, e avisar o worker exigiria improviso (escrever em arquivo ou chamar um endpoint). Polling de 2 s já atende "abaixo de 10 segundos". Fonte: [09:09] Bruno (propôs) e [09:09] Diego (rejeitou).
2. **Worker dentro do mesmo processo/instância da API.** Excluída como requisito por Diego ([09:11]), não proposta por ninguém: "o worker tem que rodar como processo separado", pois se a API reinicia, o worker é perdido. Fonte: [09:11] Diego.
3. **Escalar para múltiplos workers em paralelo.** Adiada, não decidida. Perderia a garantia de ordering; particionar por `order_id` ou usar lock pessimista é "problema do futuro". Fonte: [09:12] Diego, [09:13] Diego, [09:13] Larissa.

## Consequências

### Positivas

- Atende o requisito de latência (menos de 10 s) com mecanismo simples, sem componente novo de infraestrutura ([09:09] Diego).
- Isolamento operacional: reiniciar ou fazer deploy da API não afeta o worker ([09:11] Diego).
- Reuso da stack e do padrão de entry-point já existente em `src/server.ts` ([09:11] Larissa, [09:11] Diego).
- Ordering por `order_id` sem coordenação extra enquanto houver um worker ([09:12] Diego). Marcos informa que clientes nunca pediram ordering global ([09:14] Marcos).

### Negativas

- Mais um processo, com seu próprio PrismaClient e pool de conexões ([09:30] Bruno). Que isso implique operar e monitorar (deploy, restart, shutdown próprios) é análise deste ADR (**Hipótese**).
- Consultas periódicas ao MySQL a cada 2 s, mesmo sem eventos (consequência direta do polling, [09:09] Diego).
- Single-worker é ponto único de processamento (análise deste ADR, **Hipótese**); escalar exige decisão futura ([09:13] Diego). Estratégia de escala continua em aberto.

### Trade-off

Aceitamos **latência mínima de 2 s no pior caso e uma limitação de ordering (só por `order_id`, só com um worker)** em troca de uma solução simples, sem infraestrutura nova e dentro da meta de 10 s ([09:10] Larissa, [09:13] Larissa). A reatividade imediata de um mecanismo de notificação foi trocada por simplicidade.

## Referências

- Transcrição: [09:02] Marcos, [09:09] Diego, [09:09] Bruno, [09:10] Larissa, [09:10] Marcos, [09:11] Diego, [09:11] Larissa, [09:11] Bruno, [09:12] Diego, [09:13] Diego, [09:13] Larissa, [09:14] Marcos, [09:28] Bruno, [09:30] Bruno, [09:48] Larissa.
- Código: `src/server.ts` (bootstrap e shutdown, modelo para `src/worker.ts`, a criar), `src/config/database.ts` (`createPrismaClient`, `prisma` singleton por processo), `src/config/env.ts` (`envSchema`), `package.json` (scripts), `prisma/schema.prisma` (mesmo banco MySQL).
- Documentos do desafio: `docs/FDD.md` (fluxo do worker), `docs/RFC.md` (questões em aberto).

## Relacionados

- [ADR-001](ADR-001-outbox-no-mysql.md): outbox lida pelo worker.
- [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md): retry, timeout e DLQ executados pelo worker.
- [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md): assinatura aplicada no envio feito pelo worker.
- [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md): garantia de entrega do worker.
- [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md): padrões reaproveitados (Pino, módulo `webhooks`).
- [ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md) e [ADR-008](ADR-008-payload-snapshot-na-insercao-da-outbox.md): origem dos eventos que o worker consome.
