# ADR-004: HMAC-SHA256 com secret única por endpoint e rotação com grace de 24h

**ID:** ADR-004

- **Status:** Aceito (reunião técnica, quinta-feira 09:00-09:53), com lacunas de especificação em aberto (ver Decisão)
- **Data:** 2026-10-11 (data do registro)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)

## Status

Aceito. Sofia fechou em [09:22] ("Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h"), confirmado no resumo de [09:48] e aceito por Diego, Bruno e Larissa. Sofia pediu revisão de segurança do código de HMAC e geração de secret antes do deploy ([09:46] Sofia).

## Contexto

Os eventos de pedido saem da infraestrutura da plataforma para endpoints do cliente. O cliente precisa validar que a requisição veio da plataforma e que ninguém adulterou o payload ([09:19] Sofia). A reunião lembrou que um cliente já vazou uma secret no log da própria aplicação ([09:22] Diego). A configuração de cada webhook guarda url, secret, `customer_id` e estado ativo ([09:21] Bruno, confirmado por Sofia).

O código atual não tem nenhum mecanismo de assinatura. Em `src/shared/logger/index.ts`, o Pino faz `redact` apenas de authorization, cookie, password, passwordHash, token e accessToken.

## Decisão

1. **HMAC com SHA-256** sobre o corpo do request, assinatura enviada no header `X-Signature`; o cliente verifica do lado dele ([09:20] Sofia, [09:22] Sofia).
2. **Uma secret única por endpoint de webhook**, não uma secret global da plataforma ([09:21] Sofia).
3. **Secret gerada pela plataforma** e devolvida na criação do webhook ([09:31] Marcos).
4. **Rotação pela API:** o cliente pede nova secret; a antiga continua válida por **24 horas em paralelo (grace period)** e depois deixa de valer ([09:21] Sofia).
5. **Headers do envio** (contexto, [09:44] Diego e [09:44] Sofia): `X-Signature` (HMAC), `X-Event-Id` (UUID do evento, ver [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)), `X-Timestamp` (timestamp do envio, para o cliente detectar replay attack se quiser), `X-Webhook-Id` (id do endpoint, proposto por Sofia e aceito por Diego em [09:45]) e `Content-Type: application/json`.

Lacunas **não decididas na reunião** (rotuladas como Hipótese ou em aberto, a fechar na revisão de segurança de Sofia e no FDD):

- **Formato do valor de `X-Signature`** (por exemplo, prefixo `sha256=` e codificação hex): **em aberto**.
- **Se o `X-Timestamp` entra no material assinado:** **em aberto**. A reunião só diz "HMAC sobre o corpo do request" ([09:22] Sofia).
- **Armazenamento da secret** (cifrada ou em hash): **em aberto**. **Hipótese:** como a plataforma precisa da secret em claro para assinar cada envio, hash unidirecional não serviria; isso deve ser avaliado por Sofia.
- **Rota exata da rotação:** a reunião só fala em "endpoint pro cliente conseguir pedir nova secret pela API" ([09:21] Sofia). **Hipótese** de contrato no FDD (`POST /webhooks/:id/rotate-secret`).
- **Hipótese:** adicionar redact de secret e assinatura no Pino (`src/shared/logger/index.ts`), em linha com o vazamento relatado em [09:22] Diego; proposta do FDD, sem decisão na reunião.

## Alternativas Consideradas

1. **Secret global única da plataforma.** Rejeitada. Se vaza uma, vaza tudo. Fonte: [09:21] Sofia, reforçada por [09:22] Diego (cliente já vazou secret em log).
2. **Rotação sem período de convivência (a secret antiga morre na hora).** Rejeitada de forma implícita: o grace de 24h existe para o cliente ter tempo de migrar os sistemas dele. Fonte: [09:21] Sofia. A reunião não discutiu uma proposta alternativa explícita; este item registra o contraste com a decisão.

## Consequências

### Positivas

- Cliente consegue validar origem e integridade com HMAC-SHA256, padrão de mercado com biblioteca em todo cliente sério ([09:20] Sofia).
- Vazamento de uma secret afeta um endpoint, não todos ([09:21] Sofia).
- Rotação sem janela de indisponibilidade para o cliente, graças ao grace de 24h ([09:21] Sofia).
- `X-Timestamp` e `X-Webhook-Id` ajudam o cliente a detectar replay e a identificar o cadastro de origem ([09:44] Diego, [09:44] Sofia).

### Negativas

- Mais estado e complexidade: a configuração precisa lidar com duas secrets válidas durante o grace de 24h ([09:21] Sofia).
- A secret precisa ser armazenada de forma recuperável para assinar cada envio, o que cria superfície de risco (**Hipótese**, depende da decisão em aberto sobre armazenamento).
- Pontos de especificação (formato de `X-Signature`, timestamp no material assinado, armazenamento da secret, rota de rotação) ainda dependem de fechamento e podem gerar retrabalho.
- Exige revisão de segurança de pelo menos 2 dias úteis antes do deploy, dentro do prazo de 3 sprints ([09:46] Sofia, [09:47] Larissa).

### Trade-off

Escolhemos **segurança por endpoint com rotação em duas fases** (uma secret por endpoint, grace de 24h) em vez de uma secret global simples. Pagamos com mais complexidade de modelo e de rotação, e com um prazo reservado para revisão de segurança, em troca de menor raio de dano em caso de vazamento ([09:21] Sofia, [09:46] Sofia).

## Referências

- Transcrição: [09:19] Sofia, [09:20] Sofia, [09:20] Bruno, [09:21] Sofia, [09:21] Bruno, [09:22] Diego, [09:22] Sofia, [09:31] Marcos, [09:44] Diego, [09:44] Sofia, [09:45] Diego, [09:46] Sofia, [09:47] Larissa, [09:48] Larissa.
- Código: `src/shared/logger/index.ts` (lista `redactPaths` do Pino), `src/modules/orders/order.schemas.ts` (padrão de schemas Zod a seguir), `src/shared/errors/http-errors.ts` (base para erros `WEBHOOK_*`), `prisma/schema.prisma` (tabela de configuração do webhook ainda inexistente).
- Documentos do desafio: `docs/FDD.md` (contratos e segurança), `docs/RFC.md` (lacunas a fechar).

## Relacionados

- [ADR-001](ADR-001-outbox-no-mysql.md): origem dos eventos assinados.
- [ADR-002](ADR-002-worker-separado-em-polling.md): worker que assina e envia.
- [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md): cada tentativa e cada replay é assinado.
- [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md): `X-Event-Id` enviado junto da assinatura.
- [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md): erros `WEBHOOK_*` e schemas Zod.
- [ADR-007](ADR-007-publicacao-do-evento-na-transacao-do-change-status.md) e [ADR-008](ADR-008-payload-snapshot-na-insercao-da-outbox.md): corpo do evento que é assinado.
