# Da Reunião ao Documento: Design Docs gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (OMS) Node.js + TypeScript + Prisma/MySQL, produzido a partir de uma transcrição de reunião técnica (`TRANSCRICAO.md`) e do código existente.

O enunciado original do desafio está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia). Este README descreve o **processo** de produção.

## Sobre o desafio

A tarefa era transformar a transcrição de uma reunião de 53 minutos (tech lead, PM, dois engenheiros e uma engenheira de segurança) em documentação acionável: PRD, RFC, FDD, ADRs, um Tracker de rastreabilidade e este README. O sistema já existe em produção e não tem eventos, filas nem webhooks; a feature decidida na reunião foi um webhook de saída com padrão outbox no MySQL, worker separado em polling, retry com backoff e DLQ, assinatura HMAC-SHA256 por endpoint e entrega at-least-once.

O ponto central não é "resumir a reunião", e sim separar o que foi **decidido**, o que foi **descartado ou adiado** e o que **nunca foi dito**. Toda informação nos documentos precisa ter origem em uma fala (`[hh:mm] Nome`) ou em um arquivo do código; o que a análise precisou completar (rota de rotação de secret, formato do `X-Signature`, métricas, nomes de colunas) está rotulado como **Hipótese**. A entrega é puramente documental: `src/`, `prisma/`, `tests/`, configs e `TRANSCRICAO.md` não foram alterados.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (modelo Claude Sonnet 5.5) | Ferramenta principal: leitura do código e da transcrição, orquestração do trabalho, geração, revisão e correção dos documentos. |
| **Subagentes do Claude Code** (agentes `general-purpose` em paralelo) | Um agente por unidade de trabalho independente: ADRs 001-004, ADRs 005-008, FDD, RFC, PRD, Tracker (2 partes), revisão crítica e 3 agentes de correção. Cada um recebeu um briefing autocontido e escreveu só os seus arquivos. |
| **Plugin do Notion (MCP)** | Acesso aos prompts do curso (entrevista de PRD, FDD e HLD), usados como ponto de partida para os esqueletos de saída do PRD e do FDD. O conteúdo desses prompts não está reproduzido neste repositório. |
| **Scripts Python de validação** | Conferência mecânica: todo `[hh:mm] Nome` existe na transcrição com o falante correto, todo caminho de código citado existe, todo ID dos documentos tem linha no Tracker, ausência de travessão, links relativos. |

Observações honestas: o prompt de RFC não existe no Notion, então o RFC seguiu o formato exigido pelo enunciado. Os plugins de ADR do curso (`adrs-management`) não foram usados: os ADRs foram escritos direto em MADR por agentes, porque o fluxo do plugin (mapear o código e identificar ADRs candidatos) parte de um código a documentar, e aqui as decisões já vinham prontas da reunião. Os diagramas Mermaid não foram renderizados por ferramenta, só revisados por leitura.

## Workflow adotado

1. **Contextualização.** Leitura integral do enunciado, da transcrição e do código relevante (`order.service.ts`, `order.status.ts`, erros, middlewares, `app.ts`, `server.ts`, `schema.prisma`, `package.json`, testes). Em vez de repassar tudo a cada agente, consolidei uma **base de fatos compartilhada**, arquivo de trabalho que não foi versionado neste repositório (decisões com timestamp, requisitos, itens fora de escopo, questões em aberto, mapa do código com caminhos verificados, convenção de IDs, nomes fixos dos ADRs). Todos os agentes partiram dela e da transcrição, o que manteve nomes, números e IDs consistentes entre documentos escritos em paralelo.
2. **ADRs primeiro** (2 agentes em paralelo, em paralelo com o FDD): 8 ADRs em MADR cobrindo as 6 decisões principais mais duas decisões de integração.
3. **RFC** (depois dos ADRs, para linkar e manter coerência) e, em paralelo, o **FDD** com as seções do prompt do curso mais "Integração com o sistema existente".
4. **PRD** por último entre os grandes documentos, como consolidação de alto nível.
5. **Tracker** em duas partes em paralelo (PRD+RFC+ADRs e FDD), mesclado e conferido por script.
6. **Revisão crítica independente** por um agente que não escreveu nada (só leitura), seguida de correções em paralelo por arquivo, e revalidação por script.
7. **README** por último.

Ordem de dependência: transcrição e código, base de fatos, ADRs, FDD e RFC, PRD, Tracker, revisão, correções, README.

### Fronteira entre os documentos

| Documento | Papel | Onde está |
| --- | --- | --- |
| PRD | Por que e o quê | [`docs/PRD.md`](docs/PRD.md) |
| RFC | Proposta técnica aberta a revisão: abordagem, alternativas, questões em aberto | [`docs/RFC.md`](docs/RFC.md) |
| ADRs | Cada decisão fechada, com contexto e consequências | [`docs/adrs/`](docs/adrs/) |
| FDD | Como construir, em detalhe | [`docs/FDD.md`](docs/FDD.md) |
| Tracker | De onde veio cada item | [`docs/TRACKER.md`](docs/TRACKER.md) |

## Prompts customizados

### 1. Briefing de ADRs (versão resumida do prompt enviado aos agentes)

```text
Você escreve ADRs (formato MADR, pt-BR) para o desafio de design docs. Leia PRIMEIRO,
por inteiro, a base de fatos (decisões, timestamps, mapa do código, estilo e nomes de
arquivo). Leia também TRANSCRICAO.md para conferir citações e os arquivos de código que
for citar (confirme que existem).

Tarefa: criar EXATAMENTE estes 4 arquivos em docs/adrs/ ...
Cada ADR deve ter, com estes títulos exatos: ## Status, ## Contexto, ## Decisão,
## Alternativas Consideradas (pelo menos 2 alternativas reais discutidas, cada uma com o
motivo da rejeição e a fonte [hh:mm] Nome), ## Consequências (### Positivas, ### Negativas
e ### Trade-off, com o trade-off explícito), ## Referências, ## Relacionados.
Regras: toda afirmação com origem em [hh:mm] Nome ou arquivo de código; nada inventado
(detalhe não decidido na reunião => rotule **Hipótese** e remeta ao FDD); NUNCA transforme
itens descartados ou adiados em decisão; sem travessão; ADR conciso, sem repetir detalhe
de implementação do FDD.
NÃO altere nenhum outro arquivo. Ao final reporte inconsistências entre a base de fatos
e a transcrição.
```

### 2. Revisão crítica independente (versão resumida do prompt enviado, só leitura)

```text
Você é revisor crítico e independente do pacote de design docs. NÃO edite arquivos: só
reporte. Fontes da verdade: TRANSCRICAO.md e o código. Procure, com evidência (arquivo,
ID do item e trecho da transcrição/código):
1. CONTRADIÇÕES com a transcrição ou o código (valores como 2s, 10s, 64KB, 24h,
   1m/5m/30m/2h/12h, nomes de headers, roles, rotas, falante errado, caminhos ou classes
   inexistentes ou descritos errado).
2. ITENS DESCARTADOS OU ADIADOS virando requisito (email, rate limiting, dashboard,
   inbound, arquivamento, exactly-once, retry indefinido, ordering global, Redis, trigger,
   truncar payload, items no payload).
3. INFORMAÇÃO INVENTADA apresentada como se viesse da reunião, ou com citação de origem
   que não sustenta a afirmação.
4. INCONSISTÊNCIAS ENTRE DOCUMENTOS (nº de tentativas, nomes de tabelas, códigos
   WEBHOOK_*, rotas, links quebrados).
5. VIOLAÇÃO DAS FRONTEIRAS: RFC com detalhe de FDD, PRD com detalhe técnico, duplicação.
6. Checklist de critérios de aceite do enunciado, item por item.
Entregue relatório priorizado (CRÍTICO / MÉDIO / BAIXO) com no máximo 25 achados, cada um
com arquivo, ID, problema, evidência e correção sugerida. Diga também o que verificou e
estava correto.
```

### Prompts do curso reaproveitados

Os esqueletos de saída de **PRD** e **FDD** dos prompts do curso foram adaptados e incorporados aos briefings dos agentes, junto com suas regras de estilo: não inventar detalhe técnico sem rotular como hipótese, não usar travessão, e em "Observabilidade" especificar métricas, logs e tracing.

## Iterações e ajustes

Foram **4 ciclos principais** de geração, revisão crítica e correção. A primeira geração saiu fiel em números e falantes, mas com os problemas abaixo, que exigiram correção.

**Ciclo 1: geração em paralelo.** Os 8 ADRs, o FDD, o RFC e o PRD saíram em uma passada, a partir da base de fatos. Já aqui os próprios agentes devolveram inconsistências que eu corrigi antes de seguir:
- O ADR-008 afirmava que o UUID do evento "é o usado em `X-Event-Id`", enquanto o ADR-005 tratava a igualdade entre `X-Event-Id`, `event_id` do payload e `id` da linha da outbox como hipótese. Corrigi o ADR-008 para a mesma posição.
- O PRD dizia que o `customerId` vinha "na consulta (query)". A transcrição diz "body ou no path" ([09:32] Larissa), depois de [09:31] Marcos ter dito "implícito do JWT". Reescrevi: body ou path, com a query na listagem rotulada como hipótese do FDD.
- Na minha própria base de fatos, a classe base de `WEBHOOK_NOT_FOUND` estava como `NotFoundError`, mas esse construtor fixa o código `NOT_FOUND` e só aceita o nome do recurso. O FDD passou a registrar duas opções (estender `AppError` ou dar um parâmetro `code` opcional à classe) como questão em aberto.
- Os agentes do Tracker devolveram a lista de itens que eram proposta pura do FDD e do PRD (AES-256-GCM, lease de 60 s, batch de 10, metas de 80% de redução de polling). Em vez de forçar uma fala de origem, criei a seção "Itens sem origem identificável" no Tracker.

**Ciclo 2: revisão crítica independente.** Um agente só de leitura conferiu tudo contra a transcrição e o código (tinha 0 erros de falante ou timestamp, e valores numéricos corretos) e achou 2 pontos críticos, 11 médios e 8 baixos. Os principais:
- **"5 tentativas"**: [09:17] Larissa fechou "5 tentativas" e [09:48] resumiu "total 5 tentativas", mas os cinco intervalos somam 14h36m, o que só bate com "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego) se a primeira entrega **não** conta (1 original + 5 re-envios). Os documentos adotavam essa leitura sem citar [09:48] e o PRD a apresentava como fato. Agora todos os documentos citam as duas falas e marcam a interpretação como **Hipótese** a confirmar (RFC-OPEN-07).
- **Hipóteses sem rótulo ancoradas em falas que não as sustentam**: limite de 2048 caracteres da URL atribuído a Sofia, "a secret só é devolvida uma vez" (Marcos só disse "devolvida na criação"), "não se corta a revisão para cumprir prazo" atribuído a Larissa. Rotulei como hipótese ou removi a citação.
- **Ordem dos sprints do FDD divergia de [09:46] Larissa** (a integração no `order.service.ts` estava no sprint 1, a revisão de segurança no fim do sprint 2). Reescrevi na ordem da reunião, com a revisão de Sofia ao final do sprint 3.
- **Despacho paralelo dentro do worker** (hipótese do FDD) enfraquecia a premissa de ordering de [09:12] Diego. O default passou a ser sequencial, com concorrência configurável como questão em aberto.
- **PRD com detalhe técnico demais** (códigos HTTP, `publishWebhookEvent`, `PrismaClient`) e duplicação de listas entre PRD, RFC e FDD. Reduzi o PRD a linguagem de produto com remissão ao FDD.
- **Design em tensão com a alternativa descartada**: o RFC rejeitava "failed na própria outbox", mas o FDD mantém a linha `FAILED` na outbox e copia para a DLQ. Documentei a escolha (a linha `FAILED` é espelho; a `webhook_dead_letter` é a fonte de replay) como hipótese do FDD.
- Exemplo de timestamps incoerente no FDD, grafias divergentes (`previousSecretExpiresAt` na API versus coluna `secretPreviousExpiresAt`), e `onDelete: Cascade` contradizendo o soft delete (passou a `Restrict` nas tabelas de evidência, com o trade-off registrado).
- O Tracker ainda marcava como `TRANSCRICAO` itens que os próprios documentos rotulam como hipótese, com falas só tangenciais (ex: soft delete ancorado em "DELETE para remover", códigos de erro propostos ancorados no replay). Movi os que eram proposta pura para a seção "sem origem" e ajustei os resumos dos demais.
- A pasta `docs/adrs/` tinha um `README.md` além dos 8 ADRs; movi o índice para `docs/INDEX.md` para a pasta conter só arquivos `ADR-NNN-*`.

**Ciclo 3: correção e revalidação.** Três agentes aplicaram as correções em paralelo, um por conjunto de arquivos (FDD; PRD; RFC e ADRs), e eu atualizei o Tracker (novos IDs, itens reclassificados como sem origem, resumos ajustados) e reexecutei os scripts de validação: 0 erros de falante ou timestamp, 0 caminhos de código inexistentes na coluna `CODIGO`, 100% dos IDs com linha ou listados como sem origem, nenhum travessão, links relativos resolvendo.

**Ciclo 4: auditoria anti-alucinação, afirmação por afirmação.** Os scripts do ciclo anterior só provavam que o falante existia naquele minuto e que o arquivo existia; não provavam que a fala **sustentava** a afirmação. Para fechar essa lacuna, quatro agentes só de leitura (PRD, FDD seções 1 a 7, FDD seções 8 a 14, RFC e ADRs) conferiram cada item com ID contra a transcrição e o código, classificando em suportado, parcial, não suportado ou hipótese rotulada. Resultado: **nenhum item descartado ou adiado virou requisito, decisão ou escopo incluso** e todos os números (2 s, 10 s, 64KB, 24 h, backoff, 3 sprints, 100 entregas) e referências de código conferiram, mas dezenas de itens estavam parcialmente sem sustentação. Os principais:
- Hipóteses de desenho do FDD apresentadas como se viessem de uma fala: o mecanismo de claim do worker atribuído ao "lock pessimista" de [09:13] Diego, o `timestamp` do payload como "instante da mudança de status", o "event_id estável no replay", "uma linha na outbox por webhook", 64KB lido como 65536 bytes, a secret "devolvida uma única vez". Todas passaram a carregar **Hipótese (FDD)** no próprio item.
- Atribuições imprecisas: o aceite geral de [09:49] tratado como fechamento de uma decisão específica, "velocidade de entrega" atribuída a Larissa no ADR-006, "testes ponta a ponta [09:46]" tratado como mitigação quando era só estimativa de esforço, Sofia listada na sessão de revisão do design que [09:50] Larissa marcou só para Bruno e Diego.
- Quatro **citações entre aspas que eram paráfrases** (ex: "worker sequencial por created_at", "leitura da outbox mais limpa", "endpoint pro cliente pedir nova secret"). Passaram a ser literais, e um script confere as aspas próximas de `[hh:mm]` contra a transcrição.
- **Defeitos reais de desenho** que a leitura cruzada expôs: a métrica "idade do evento devido" usava `createdAt` e dispararia falso positivo em todo retry de 12 h (passou a usar `nextAttemptAt`); o exemplo de replay reaproveitava o id de um evento já entregue com sucesso e uma data anterior ao fim da janela de retries; a descrição do diagrama de estados dizia que todo evento `FAILED` ia para a DLQ, contradizendo o fluxo de webhook inativo; um link do PRD apontava para um arquivo que eu mesmo tinha movido.
- Alternativas tratadas como "propostas e rejeitadas" na reunião que na verdade ninguém propôs (worker dentro da API, secret global): foram excluídas como requisito por Diego e Sofia, e o texto agora diz isso. Criei também a alternativa que faltava (`items` no payload, RFC-ALT-13).
- O Tracker foi reconciliado com o texto corrigido: itens cuja parte central é hipótese ganharam o Tipo `Hipótese` com a fala ou arquivo de derivação, e propostas puras (métricas, tracing, lease do worker, cifra da secret, códigos de erro adicionais) foram para a seção "sem origem".

Pontos que **continuam abertos de propósito** (e estão registrados como questões em aberto no RFC e no FDD): o significado final de "5 tentativas", o formato do `X-Signature`, a rota de rotação de secret, o armazenamento da secret, o evento na criação do pedido (`create` não passa por `changeStatus`) e o papel do código `WEBHOOK_INVALID_URL` diante do `VALIDATION_ERROR` do Zod.

## Como navegar a entrega

```text
.
├── README.md                  este arquivo (processo)
├── TRANSCRICAO.md             reunião (não alterada)
└── docs/
    ├── INDEX.md               índice do pacote e dos ADRs
    ├── PRD.md                 por que e o quê
    ├── RFC.md                 proposta técnica para revisão
    ├── FDD.md                 como construir, em detalhe
    ├── TRACKER.md             rastreabilidade item a item
    └── adrs/
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-separado-em-polling.md
        ├── ADR-003-retry-backoff-exponencial-e-dlq.md
        ├── ADR-004-hmac-sha256-secret-por-endpoint.md
        ├── ADR-005-entrega-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-do-projeto.md
        ├── ADR-007-publicacao-do-evento-na-transacao-do-change-status.md
        └── ADR-008-payload-snapshot-na-insercao-da-outbox.md
```

**Ordem sugerida de leitura**

1. [`docs/PRD.md`](docs/PRD.md): o problema, o público, o escopo (incluindo o que ficou **fora**) e as metas.
2. [`docs/RFC.md`](docs/RFC.md): a proposta, as alternativas descartadas e as questões em aberto (2 a 4 páginas).
3. [`docs/adrs/`](docs/adrs/): cada decisão fechada, com contexto e consequências (começar pelo ADR-001).
4. [`docs/FDD.md`](docs/FDD.md): fluxos, modelo de dados, contratos HTTP, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção "Integração com o sistema existente".
5. [`docs/TRACKER.md`](docs/TRACKER.md): para auditar a origem de qualquer item e ver os itens que são só proposta.

**Convenções**: cada item identificável tem um ID em negrito (`PRD-FR-03`, `RFC-ALT-02`, `FDD-CONTRATO-01`, `ADR-004`), citações de origem no formato `[hh:mm] Nome` (transcrição) ou caminho de arquivo (código), e **Hipótese** marca tudo que não veio da reunião nem do código. Arquivos que ainda não existem (como `src/worker.ts` e `src/modules/webhooks/`) aparecem marcados como "(novo)" ou "(a criar)".
