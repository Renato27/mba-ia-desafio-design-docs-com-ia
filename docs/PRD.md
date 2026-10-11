# PRD: Order Management System - Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| Versão | 1.0 |
| Data | 2026-10-11 |
| Autoria | Larissa (Tech Lead) e Marcos (PM) [09:50] Larissa (abre o doc de design); Marcos é o dono do problema de negócio [09:00] |
| Status | Em revisão |
| Documentos relacionados | [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/README.md), [Tracker](TRACKER.md) |

Como ler este PRD: ele responde "por que e o quê". O "como construir" fica no [FDD](FDD.md), a proposta técnica e as alternativas ficam no [RFC](RFC.md) e cada decisão fechada tem um ADR. Aqui não se repete detalhe técnico.

Convenções:

- Todo item identificável tem ID em negrito (ex: **PRD-FR-01**) e origem entre colchetes: `[hh:mm] Nome` para a reunião técnica (`TRANSCRICAO.md`) ou caminho de arquivo para o código.
- **Hipótese (PRD)** marca o que NÃO veio da reunião nem do código e foi assumido aqui para fechar o documento. Hipóteses não são decisões e precisam ser confirmadas na revisão.
- **Hipótese (FDD)** marca detalhes que o FDD propôs sem origem na reunião; o PRD só os cita para remeter ao FDD.
- Nenhum item descartado ou adiado na reunião vira requisito; eles aparecem apenas em "Fora de escopo".
- Os caminhos HTTP aparecem como foram ditos na reunião, sem o prefixo `/api/v1` que a API usa (ver [FDD](FDD.md), seção 6).

---

## Resumo e contexto da feature

- **PRD-RES-01**: a feature entrega notificações de saída (webhooks) para clientes B2B sempre que o status de um pedido muda no Order Management System (OMS). O cliente cadastra uma URL, escolhe os status que quer ouvir e passa a receber um aviso assinado, sem precisar consultar a API em loop [09:00] Marcos [09:33] Marcos.
- **PRD-RES-02**: o pedido veio formalmente de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo [09:00] Marcos. "Tempo real" para eles significa qualquer coisa abaixo de 10 segundos [09:02] Marcos.
- **PRD-RES-03**: o fluxo é apenas outbound: a plataforma envia, o cliente recebe. Não existe webhook de entrada [09:02] Marcos [09:03] Sofia.
- **PRD-RES-04**: a abordagem acordada na reunião é gravar o evento no próprio banco, na mesma transação da mudança de status, e entregá-lo por um processo separado, com re-tentativas e fila de falhas (resumo em [09:48] Larissa). O raciocínio está no [RFC](RFC.md) (RFC-TLDR) e nos ADRs; o desenho detalhado está no [FDD](FDD.md).
- **PRD-RES-05**: prazo combinado: 3 sprints, incluindo a revisão de segurança, para atender o pedido da Atlas de ter a feature no fim de novembro [09:45] Marcos [09:46] Larissa [09:47] Larissa.

## Problema e motivação

- **PRD-PROB-01** (prioridade: Alta; impacto: Alto): hoje os 3 clientes B2B descobrem mudanças de status consultando `GET /orders` de tempos em tempos. Isso deixa a integração lenta e cara para eles [09:00] Marcos.
- **PRD-PROB-02** (prioridade: Alta; impacto: Alto, comercial): risco de churn. A Atlas Comercial sugeriu que, se a plataforma não entregar a notificação até o fim do trimestre, pode migrar para um concorrente [09:00] Marcos. A Atlas pede a feature para o fim de novembro [09:45] Marcos.
- **PRD-PROB-03** (prioridade: Média; impacto: Médio): o atraso percebido pelo cliente. O importante para eles é "não ficar pendurado" nem precisar atualizar manualmente [09:02] Marcos.
- **PRD-PROB-04** (prioridade: Alta; impacto: Alto, técnico): o sistema atual não tem nenhum mecanismo de notificação externa, evento, fila ou processo em background (`src/modules/orders/order.service.ts`, `package.json`). E a mudança de status já é uma transação pesada, em que um cliente lento ou fora do ar não pode travar outros pedidos nem causar rollback [09:04] Bruno.
- **PRD-PROB-05** (prioridade: Média; impacto: Médio, segurança): enviar dados de pedidos para fora da infraestrutura exige que o cliente consiga validar a origem e a integridade de cada requisição [09:19] Sofia.

## Público-alvo e cenários de uso

Público-alvo:

- clientes B2B integradores, que consomem a API do OMS por sistema (Atlas Comercial, MaxDistribuição, Nova Cargo) e recebem os webhooks [09:00] Marcos.
- usuários da plataforma que representam o cliente. O cliente cadastra o webhook pela API da plataforma, autenticado com JWT do sistema; o JWT é de um usuário operador, não do cliente [09:32] Marcos [09:32] Bruno.
- administradores (role ADMIN), responsáveis pelo reprocessamento manual de eventos que esgotaram as tentativas [09:18] Diego [09:36] Sofia.
- Marcos, que documentará no portal de desenvolvedor como integrar pela API e como tratar eventos duplicados [09:26] Marcos [09:40] Marcos.

Cenários de uso (ilustrativos; os exemplos de nomes e valores são **Hipótese (PRD)**, sustentados pelos comportamentos citados na origem):

- um usuário que representa a Atlas cadastra uma URL https e escolhe ouvir só SHIPPED e DELIVERED. A plataforma devolve a secret na criação [09:31] Marcos [09:33] Marcos.
- um pedido da Atlas vai de PROCESSING para SHIPPED. Em menos de 10 segundos o endpoint da Atlas recebe um evento assinado, com os campos básicos do pedido [09:02] Marcos [09:43] Diego.
- o endpoint do cliente fica fora do ar por 2 horas, em manutenção planejada. A plataforma re-tenta com intervalos crescentes e a janela de re-tentativas cobre a manutenção [09:16] Diego [09:17] Diego.
- o cliente recebe o mesmo evento duas vezes e o descarta pelo `X-Event-Id` que já processou [09:24] Diego [09:25] Diego.
- o cliente suspeita que a secret vazou em um log dele (já aconteceu antes [09:22] Diego). Ele pede uma nova secret pela API e tem 24 horas para migrar os sistemas, com a antiga ainda válida [09:21] Sofia.
- o cliente quer conferir o que foi enviado: consulta o histórico dos últimos 100 envios, com sucesso ou falha, payload, resposta e tempo de resposta [09:34] Marcos.
- um evento esgota as tentativas e vai para a fila de falhas. Um ADMIN o reprocessa por endpoint próprio, e o sistema registra quem fez o replay [09:18] Diego [09:36] Sofia.

## Onde a feature será implantada

- o OMS existente, uma aplicação Node.js + TypeScript com módulos em `src/modules` (autenticação, usuários, clientes, produtos e pedidos), banco MySQL acessado via Prisma (`prisma/schema.prisma`, `package.json`) [09:27] Bruno.
- a feature entra como um módulo novo no mesmo padrão dos demais, `src/modules/webhooks` (a criar) [09:27] Bruno, e como uma segunda entry-point do projeto, `src/worker.ts` (a criar), ao lado de `src/server.ts` [09:11] Larissa.
- o ponto de contato com o código existente é a mudança de status do pedido em `src/modules/orders/order.service.ts` (`changeStatus`) [09:40] Bruno. O detalhamento por arquivo está no FDD, seção 11 ([FDD](FDD.md), FDD-INT-01 a FDD-INT-17).
- não há infraestrutura nova: o banco é o mesmo MySQL e a stack é a mesma [09:07] Diego [09:11] Diego.

## Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| **PRD-OBJ-01** | Notificar o cliente em "tempo real" | Tempo entre a mudança de status e o primeiro envio ao endpoint do cliente (com o endpoint saudável) | Menos de 10 s | [09:02] Marcos |
| **PRD-OBJ-02** | Manter a latência dentro da meta com uma solução simples | Intervalo de leitura da fila de eventos pelo worker | Polling de 2 s (latência mínima de 2 s no pior caso aceita) | [09:09] Diego [09:10] Larissa [09:10] Marcos |
| **PRD-OBJ-03** | Não perder nem inventar eventos | Eventos perdidos ou fantasmas em relação às mudanças de status (commit sem evento, ou evento sem commit) | 0 (**Hipótese (PRD)**: meta numérica derivada da regra de atomicidade, que veio da reunião) | [09:06] Diego [09:40] Bruno [09:41] Diego |
| **PRD-OBJ-04** | Entregar mesmo com o cliente instável | Re-tentativas por evento e intervalos | 5 tentativas com backoff 1m/5m/30m/2h/12h (a reunião não disse se a 1ª entrega conta; leitura adotada: 5 re-envios após a 1ª falha, 14h36m, **Hipótese** a confirmar, ver RFC-OPEN-07); depois, fila de falhas | [09:17] Diego [09:17] Larissa |
| **PRD-OBJ-05** | Não prender a plataforma a clientes lentos | Tempo máximo de espera por uma chamada ao cliente | 10 s por chamada; sem resposta vira falha e entra em re-tentativa | [09:42] Diego |
| **PRD-OBJ-06** | Garantir autenticidade e integridade do que é enviado | Envios assinados com HMAC-SHA256, com secret única por endpoint | 100% dos envios assinados (**Hipótese (PRD)**: meta numérica derivada da regra de HMAC, que veio da reunião); rotação com 24 h de convivência | [09:22] Sofia |
| **PRD-OBJ-07** | Cumprir o prazo da Atlas | Duração do projeto e data de entrega | 3 sprints, com revisão de segurança de no mínimo 2 dias úteis ao final, entrega no fim de novembro | [09:45] Marcos [09:46] Larissa [09:46] Sofia [09:47] Larissa |
| **PRD-OBJ-08** | Reduzir a dependência dos clientes do polling em `GET /orders` | Chamadas de polling dos 3 clientes a `GET /orders` para detectar mudança de status | **Hipótese (PRD)**: queda de pelo menos 80% depois da adoção do webhook. A reunião não fixou número; a meta deve ser acordada por Marcos com os clientes | [09:00] Marcos (problema); meta: Hipótese (PRD) |
| **PRD-OBJ-09** | Evitar a perda da conta da Atlas | Atlas ativa na plataforma após a entrega | Sem número na reunião. **Hipótese (PRD)**: meta qualitativa, "Atlas permanece cliente"; Marcos confirma o prazo com a Atlas | [09:00] Marcos [09:47] Marcos |

## Escopo

### Incluso

- **PRD-ESC-01**: cadastro, edição, remoção e listagem de webhooks por customer, com URL https e lista de status desejados [09:31] Marcos [09:33] Bruno [09:23] Sofia.
- **PRD-ESC-02**: geração de secret pela plataforma, uma por endpoint, devolvida na criação, com rotação por API e 24 h de convivência da secret antiga [09:21] Sofia [09:31] Marcos.
- **PRD-ESC-03**: evento `order.status_changed` registrado de forma atômica com a mudança de status, com filtro por status aplicado na inserção [09:40] Bruno [09:33] Marcos [09:34] Bruno.
- **PRD-ESC-04**: entrega assíncrona por worker em processo separado, em polling de 2 s, com payload assinado (HMAC-SHA256) e headers de identificação [09:09] Diego [09:11] Diego [09:44] Diego [09:44] Sofia.
- **PRD-ESC-05**: re-tentativas com backoff exponencial e fila de falhas (DLQ), com reprocessamento manual por ADMIN e registro de quem o executou [09:17] Larissa [09:18] Diego [09:36] Sofia.
- **PRD-ESC-06**: histórico das últimas 100 entregas de um webhook, acessível por endpoint [09:34] Marcos.
- **PRD-ESC-07**: garantia de entrega at-least-once, com `X-Event-Id` para o cliente deduplicar [09:25] Diego [09:26] Larissa.
- **PRD-ESC-08**: reuso dos padrões do projeto: módulo `src/modules/webhooks`, `AppError` e códigos de erro com prefixo `WEBHOOK_`, Pino, error middleware, Zod e `requireRole` [09:27] Bruno [09:29] Larissa [09:30] Larissa [09:36] Larissa.
- **PRD-ESC-09**: documentação para o cliente no portal de desenvolvedor, a cargo do PM: como integrar via API e como tratar duplicatas [09:26] Marcos [09:40] Marcos.

### Fora de escopo

Nada abaixo vira requisito. "Adiado" significa que pode voltar em fase futura; "descartado" significa que foi rejeitado na reunião.

| ID | Item | Situação | Origem |
| --- | --- | --- | --- |
| **PRD-OUT-01** | Email ou notificação ao cliente quando o webhook falha (ex: 3 falhas seguidas) | Adiado: fora desta fase, "talvez próxima fase, depois que a gente medir o impacto"; Marcos registrou como "futuro" | [09:37] Larissa [09:38] Marcos |
| **PRD-OUT-02** | Rate limiting de saída por cliente | Adiado: "observar e decidir depois"; fica como ponto em aberto | [09:38] Diego [09:39] Diego [09:39] Larissa |
| **PRD-OUT-03** | Dashboard visual ou painel do cliente | Adiado: agora só endpoints; painel é projeto separado do time de frontend | [09:39] Marcos [09:40] Larissa |
| **PRD-OUT-04** | Webhooks de entrada (inbound) | Descartado: só saída | [09:02] Marcos [09:03] Sofia |
| **PRD-OUT-05** | Arquivamento de eventos entregues após cerca de 30 dias | Adiado: fora do escopo desta feature | [09:08] Diego |
| **PRD-OUT-06** | Ordering global e múltiplos workers (particionar por `order_id` ou lock pessimista) | Adiado: "problema do futuro"; limitação conhecida. A garantia é só por pedido e só com um worker | [09:12] Diego [09:13] Diego [09:13] Larissa [09:14] Marcos |
| **PRD-OUT-07** | Entrega exactly-once | Descartado: exigiria coordenação dos dois lados e muito mais complexidade | [09:25] Diego |
| **PRD-OUT-08** | Retry indefinido | Descartado: deixaria evento pendurado para sempre se o cliente sumisse | [09:15] Diego |
| **PRD-OUT-09** | Apenas 3 tentativas | Descartado: 3 tentativas em 30 min não cobrem uma manutenção planejada de 2 h | [09:16] Bruno [09:16] Diego |
| **PRD-OUT-10** | Redis Streams ou fila externa | Descartado: mais infraestrutura, desproporcional para um time pequeno | [09:07] Larissa [09:07] Diego |
| **PRD-OUT-11** | Trigger do banco ou NOTIFY para acordar o worker | Descartado: o MySQL não tem listener nativo e a trigger só executa SQL | [09:09] Bruno [09:09] Diego |
| **PRD-OUT-12** | Chamada HTTP síncrona dentro da mudança de status | Descartado: travaria outros pedidos e um cliente fora do ar não pode causar rollback | [09:04] Bruno [09:06] Diego |
| **PRD-OUT-13** | Id auto incremental na tabela de eventos | Descartado: o padrão do projeto é UUID | [09:51] Larissa |
| **PRD-OUT-14** | Truncar payload acima do limite | Descartado: o evento acima de 64 KB falha com erro | [09:23] Sofia [09:24] Larissa |
| **PRD-OUT-15** | Itens do pedido (`items`) dentro do payload | Descartado: payload enxuto; o cliente consulta `GET /orders/:id` | [09:43] Diego [09:44] Bruno |
| **PRD-OUT-16** | Endurecer a permissão do CRUD de webhooks além da autenticação | Adiado: "por enquanto sim; mais pra frente a gente pode endurecer" | [09:37] Sofia |
| **PRD-OUT-17** | Worker rodando dentro do processo da API | Descartado: o reinício da API derrubaria o worker | [09:11] Diego |
| **PRD-OUT-18** | Secret global da plataforma, em vez de uma por endpoint | Descartado: se vazasse uma, vazaria tudo | [09:21] Sofia |
| **PRD-OUT-19** | Guardar só o `order_id` e montar o payload no envio | Descartado: o evento deve refletir o estado do pedido no momento da mudança | [09:51] Bruno [09:52] Larissa |
| **PRD-OUT-20** | Evento na criação do pedido | Não tratado na reunião (só se falou de mudança de status). **Hipótese (PRD)**: fora desta fase; questão em aberto no RFC (RFC-OPEN-11) | [09:40] Bruno; `src/modules/orders/order.service.ts` |

## Requisitos funcionais

Cada requisito indica: descrição, fluxo principal, fluxos alternativos e exceções, erros previstos, prioridade e origem. Os erros são descritos em linguagem de produto, com uma linha de remissão à matriz de erros do [FDD](FDD.md) (seção 7); contratos, payloads de exemplo e códigos HTTP estão no FDD (seções 6 e 7). Os códigos `WEBHOOK_*` mencionados na reunião são `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` [09:28] Bruno; os demais códigos foram propostos no FDD (**Hipótese (FDD)**). As prioridades são avaliação deste PRD (**Hipótese (PRD)**).

### **PRD-FR-01** Cadastrar webhook

Descrição: o cliente cadastra um webhook informando a URL e a lista de status que deseja receber. A plataforma gera a secret e a devolve apenas na criação.

- **Fluxo principal**: usuário autenticado envia `POST` de cadastro com `customerId`, URL https e lista de status; o sistema valida, grava e responde com os dados do webhook e a secret gerada.
- **Fluxos alternativos e exceções**: URL com `http` é recusada com erro de validação; customer inexistente é recusado; **Hipótese (PRD)**: a secret não é devolvida de novo depois (a reunião só disse que ela é "devolvida na criação" [09:31] Marcos); para obter outra, ver PRD-FR-08.
- **Erros previstos**: ver FDD-ERR-02, FDD-ERR-11, FDD-ERR-12 e FDD-ERR-13 (FDD, seção 7, Matriz de erros); o papel de `WEBHOOK_INVALID_URL` frente ao erro de validação genérico segue em aberto (Q-03 do FDD).
- **Prioridade**: Alta.
- Origem: [09:31] Marcos [09:21] Bruno [09:23] Sofia [09:28] Bruno. Contrato: FDD-CONTRATO-01.

### **PRD-FR-02** Identificar o customer no cadastro e na consulta

Descrição: o `customerId` identifica de quem é o webhook e NÃO vem do JWT, porque o JWT é do usuário operador, não do cliente.

- **Fluxo principal**: o `customerId` é enviado no body (cadastro) ou no path, como definido em [09:32] Larissa; na listagem ele é enviado como parâmetro de consulta (**Hipótese (FDD)**, FDD-CONTRATO-02, a reunião só fala em body ou path), e o endpoint é autenticado normalmente.
- **Fluxos alternativos e exceções**: listagem sem `customerId` é recusada. Marcos disse inicialmente "customer_id implícito do JWT" [09:31] e foi corrigido logo depois [09:32]; vale a correção.
- **Erros previstos**: ver FDD-ERR-11 e FDD-ERR-13.
- **Prioridade**: Alta.
- Origem: [09:32] Bruno [09:32] Marcos [09:32] Larissa. Contrato: FDD-CONTRATO-12.

### **PRD-FR-03** Editar webhook

Descrição: o cliente altera um webhook existente (por exemplo, a URL e a lista de status).

- **Fluxo principal**: `PATCH` do webhook com os campos alterados; o sistema valida e devolve o webhook atualizado, sem a secret (**Hipótese (PRD)**, ver PRD-NFR-15).
- **Fluxos alternativos e exceções**: webhook inexistente; nova URL `http`; as mesmas regras de URL do cadastro valem na edição.
- **Erros previstos**: ver FDD-ERR-01, FDD-ERR-02 e FDD-ERR-11.
- **Prioridade**: Média.
- Origem: [09:33] Bruno. Contrato: FDD-CONTRATO-03.

### **PRD-FR-04** Remover webhook

Descrição: o cliente remove um webhook que não quer mais usar.

- **Fluxo principal**: `DELETE` do webhook; o sistema confirma a remoção e deixa de gerar eventos para ele.
- **Fluxos alternativos e exceções**: webhook inexistente. O destino dos eventos pendentes de um webhook removido ou inativo é detalhe do FDD (FDD-FLX-03 e Q-06), não decidido na reunião.
- **Erros previstos**: ver FDD-ERR-01.
- **Prioridade**: Média.
- Origem: [09:33] Bruno. Contrato: FDD-CONTRATO-04.

### **PRD-FR-05** Listar webhooks de um customer

Descrição: o cliente lista os webhooks cadastrados de um customer.

- **Fluxo principal**: `GET` com o `customerId`; o sistema devolve a lista, sem expor secrets (**Hipótese (PRD)**, ver PRD-NFR-15).
- **Fluxos alternativos e exceções**: customer sem webhooks recebe lista vazia; `customerId` ausente é recusado.
- **Erros previstos**: ver FDD-ERR-11 e FDD-ERR-13.
- **Prioridade**: Média.
- Origem: [09:33] Bruno. Contrato: FDD-CONTRATO-02.

### **PRD-FR-06** Filtro de eventos por status

Descrição: cada webhook escolhe quais status quer ouvir (ex: só SHIPPED e DELIVERED). O filtro é aplicado quando o evento é registrado: se nenhum webhook do customer ouve aquele status, nenhum evento é criado.

- **Fluxo principal**: a mudança de status ocorre; o sistema identifica os webhooks ativos do customer que ouvem o novo status e registra um evento para cada um.
- **Fluxos alternativos e exceções**: nenhum webhook interessado, então nenhum evento e nenhuma linha nova (economiza armazenamento).
- **Erros previstos**: ver FDD-ERR-11 (lista de status inválida no cadastro ou na edição é recusada com erro de validação).
- **Prioridade**: Alta.
- Origem: [09:33] Marcos [09:34] Bruno [09:34] Diego. Ver [ADR-007](adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md).

### **PRD-FR-07** Histórico de entregas

Descrição: o cliente consulta os últimos 100 envios feitos para um webhook, com sucesso ou falha, payload, resposta recebida e tempo de resposta.

- **Fluxo principal**: `GET /webhooks/:id/deliveries`; o sistema devolve os envios mais recentes, até 100.
- **Fluxos alternativos e exceções**: webhook sem envios devolve lista vazia; o formato exato da paginação é **Hipótese (FDD)**.
- **Erros previstos**: ver FDD-ERR-01, FDD-ERR-11 e FDD-ERR-13.
- **Prioridade**: Alta (sem ele o cliente não consegue diagnosticar falhas e não há email, PRD-OUT-01).
- Origem: [09:34] Marcos. Contrato: FDD-CONTRATO-05.

### **PRD-FR-08** Rotação de secret

Descrição: o cliente pede uma nova secret pela API. A secret antiga continua válida por 24 horas em paralelo e depois deixa de valer.

- **Fluxo principal**: o cliente aciona a rotação; o sistema gera e devolve a nova secret uma única vez; durante 24 h as duas assinaturas são aceitáveis para o cliente; depois só a nova.
- **Fluxos alternativos e exceções**: a reunião definiu a capacidade, mas NÃO a rota. A rota `POST /webhooks/:id/rotate-secret` é **Hipótese (FDD)**. Webhook inexistente ou inativo não rotaciona.
- **Erros previstos**: ver FDD-ERR-01 e FDD-ERR-09 (**Hipótese (FDD)**).
- **Prioridade**: Alta (vazamento de secret já ocorreu com um cliente).
- Origem: [09:21] Sofia [09:22] Diego [09:22] Sofia. Contrato: FDD-CONTRATO-06. Ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).

### **PRD-FR-09** Replay manual da fila de falhas (DLQ)

Descrição: um ADMIN reprocessa manualmente um evento que esgotou as tentativas, recolocando-o como pendente para nova entrega. O sistema registra quem executou o replay.

- **Fluxo principal**: `POST /admin/webhooks/dead-letter/:id/replay` por usuário com role ADMIN; o evento volta para a fila de envio e o log registra o responsável.
- **Fluxos alternativos e exceções**: usuário que não é ADMIN é recusado; id inexistente; replay repetido da mesma linha; webhook removido ou inativo.
- **Erros previstos**: ver FDD-ERR-07 a FDD-ERR-10 e FDD-ERR-13 (os erros específicos do replay foram propostos no FDD, **Hipótese (FDD)**; a recusa por role reaproveita o `requireRole` existente).
- **Prioridade**: Alta.
- Origem: [09:18] Diego [09:35] Diego [09:36] Sofia [09:36] Larissa. Contrato: FDD-CONTRATO-07. Ver [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md).

### **PRD-FR-10** Entrega assíncrona do evento assinado

Descrição: a cada evento registrado, um worker em processo separado envia ao endpoint do cliente um `POST` com o payload JSON de `order.status_changed`, assinado com HMAC-SHA256 e acompanhado de headers de identificação do evento, da assinatura e do webhook, entre eles `X-Event-Id` e `X-Signature` (conjunto completo no FDD, seção 6).

- **Fluxo principal**: o worker lê a cada 2 s os eventos pendentes mais antigos, assina o corpo com a secret do endpoint, envia com timeout de 10 s e marca o evento como entregue quando o cliente responde com sucesso.
- **Fluxos alternativos e exceções**: sem resposta em 10 s ou resposta de falha, a tentativa conta como falha e segue para PRD-FR-11; o mesmo evento pode chegar mais de uma vez, com o mesmo `X-Event-Id` (at-least-once); o formato exato do valor de `X-Signature` e se a data do envio entra na assinatura não foram definidos (FDD Q-02, RFC-OPEN-08).
- **Erros previstos** (falhas internas de entrega, que não chegam ao cliente da API): ver FDD-ERR-03, FDD-ERR-05 e FDD-ERR-06 (**Hipótese (FDD)**).
- **Prioridade**: Alta.
- Origem: [09:09] Diego [09:11] Diego [09:20] Sofia [09:42] Diego [09:44] Diego [09:44] Sofia [09:25] Diego. Ver [ADR-002](adrs/ADR-002-worker-separado-em-polling.md), [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) e [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).

### **PRD-FR-11** Re-tentativas com backoff e DLQ

Descrição: quando a entrega falha, o sistema re-tenta com backoff exponencial de 1 min, 5 min, 30 min, 2 h e 12 h. Esgotadas as tentativas, o evento vai para uma fila de falhas persistida em tabela separada, com payload, motivo da falha e timestamp.

- **Fluxo principal**: falha da entrega, agendamento da próxima tentativa conforme o backoff, nova tentativa; após a última, o evento é copiado para a fila de falhas (ele permanece registrado como falho na outbox, e a cópia na fila de falhas é a base do replay, **Hipótese (FDD)**, ver FDD-FLX-07) e fica disponível para replay (PRD-FR-09).
- **Fluxos alternativos e exceções**: não há retry indefinido nem apenas 3 tentativas (PRD-OUT-08, PRD-OUT-09); não há email ao cliente (PRD-OUT-01). Contagem: 5 tentativas com backoff 1m/5m/30m/2h/12h [09:17] Larissa (a reunião não disse se a 1ª entrega conta; leitura adotada: 5 re-envios após a 1ª falha, 14h36m, que bate com "quase 15 horas" [09:17] Diego, **Hipótese** a confirmar, ver RFC-OPEN-07).
- **Erros previstos**: ver FDD-ERR-05 e FDD-ERR-06 (motivo de falha registrado, **Hipótese (FDD)**).
- **Prioridade**: Alta.
- Origem: [09:15] Diego [09:16] Larissa [09:17] Diego [09:17] Larissa [09:18] Diego. Ver [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md).

### **PRD-FR-12** Evento atômico com a mudança de status

Descrição: o evento é registrado na mesma transação da mudança de status do pedido. Se a transação der commit, o evento existe; se der rollback, o evento some. Se o registro do evento falhar, a mudança de status também é revertida.

- **Fluxo principal**: o `changeStatus` atualiza o pedido e o histórico, e, ainda na mesma transação, registra os eventos dos webhooks interessados; ao confirmar, ambos persistem juntos.
- **Fluxos alternativos e exceções**: falha ao registrar o evento reverte o status ("não pode ter caso de status mudar e evento não sair"); evento gerado acima de 64 KB falha com erro; se isso também aborta a mudança de status, o FDD assume que sim, mas a reunião não decidiu o caso (RFC-OPEN-12).
- **Erros previstos**: ver FDD-ERR-04 (**Hipótese (FDD)**); uma falha de banco também reverte a mudança de status.
- **Prioridade**: Alta.
- Origem: [09:06] Diego [09:40] Bruno [09:41] Bruno [09:41] Diego; `src/modules/orders/order.service.ts`. Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-007](adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md).

### **PRD-FR-13** Conteúdo do evento (snapshot enxuto)

Descrição: o evento é um JSON com `event_id`, `event_type` (`order.status_changed`), `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. Não inclui `items`. O conteúdo é fixado (snapshot) no momento em que o evento é registrado.

- **Fluxo principal**: ao registrar o evento, o sistema monta o payload com o estado do pedido naquele instante e o guarda; todas as tentativas e replays enviam esse mesmo conteúdo.
- **Fluxos alternativos e exceções**: se o pedido mudar depois, o evento já registrado não é alterado; o cliente que precisa dos itens consulta `GET /orders/:id`; payload acima de 64 KB é erro, nunca é truncado.
- **Erros previstos**: ver FDD-ERR-04 (**Hipótese (FDD)**).
- **Prioridade**: Alta.
- Origem: [09:43] Diego [09:44] Bruno [09:52] Larissa [09:52] Diego [09:24] Larissa. Ver [ADR-008](adrs/ADR-008-payload-snapshot-na-insercao-da-outbox.md).

## Requisitos não funcionais

- **PRD-NFR-01** (performance): notificação em menos de 10 s entre a mudança de status e o primeiro envio, com o endpoint do cliente saudável [09:02] Marcos.
- **PRD-NFR-02** (performance): polling do worker a cada 2 s; a latência mínima de 2 s no pior caso foi aceita [09:09] Diego [09:10] Larissa [09:10] Marcos.
- **PRD-NFR-03** (performance): timeout de 10 s por chamada HTTP ao cliente; sem resposta é falha e marca para retry [09:42] Diego.
- **PRD-NFR-04** (performance): payload de no máximo 64 KB; acima disso há erro, sem truncar. Larissa classificou como requisito não funcional, não como decisão arquitetural [09:23] Sofia [09:24] Diego [09:24] Larissa.
- **PRD-NFR-05** (disponibilidade e confiabilidade): o worker é um processo separado da API; reiniciar a API não derruba o worker [09:11] Diego. Um worker parado não impede a mudança de status de pedidos, e os eventos ficam pendentes e duráveis (consequência do desenho, FDD-RES-12).
- **PRD-NFR-06** (confiabilidade): entrega at-least-once, com `X-Event-Id` estável para deduplicação no cliente [09:24] Diego [09:25] Diego.
- **PRD-NFR-07** (confiabilidade): retry com 5 tentativas e backoff 1m/5m/30m/2h/12h (a reunião não disse se a 1ª entrega conta; leitura adotada: 5 re-envios após a 1ª falha, 14h36m, **Hipótese** a confirmar, ver RFC-OPEN-07); sem retry indefinido [09:17] Larissa.
- **PRD-NFR-08** (confiabilidade, limitação conhecida): ordering só por `order_id` e só enquanto houver um único worker; sem garantia de ordering global [09:12] Diego [09:13] Larissa [09:14] Marcos. O FDD acrescenta que a ordem pode inverter nos retries de um mesmo pedido (**Hipótese (FDD)**, FDD-RES-07).
- **PRD-NFR-09** (disponibilidade, meta numérica): nenhuma meta de disponibilidade em percentual (por exemplo, 99,9%) foi discutida na reunião. **Hipótese (PRD)**: o item fica sem valor e deve ser definido por Marcos e Larissa se os clientes exigirem.
- **PRD-NFR-10** (integridade de dados): evento e mudança de status são atômicos [09:40] Bruno [09:41] Diego; a meta de 0 eventos perdidos ou fantasmas é derivada dessa regra (**Hipótese (PRD)**).
- **PRD-NFR-11** (integridade de dados): ids em UUID, como no resto do projeto; payload como snapshot do momento da mudança [09:51] Larissa [09:52] Larissa.
- **PRD-NFR-12** (segurança): HMAC-SHA256 sobre o corpo do request, com secret única por endpoint, gerada pela plataforma e rotacionável com 24 h de convivência [09:20] Sofia [09:21] Sofia [09:22] Sofia [09:31] Marcos.
- **PRD-NFR-13** (segurança): TLS obrigatório; URL do webhook precisa ser https, e `http` é recusado com erro de validação [09:23] Sofia.
- **PRD-NFR-14** (segurança e autorização): o CRUD, o histórico de entregas e a rotação aceitam qualquer role autenticada, "por enquanto"; o replay exige role ADMIN, por `requireRole` [09:36] Marcos [09:36] Sofia [09:36] Larissa [09:37] Sofia. O `customerId` vem do body ou do path, não do JWT [09:32] Larissa, e isso abre o risco PRD-RISK-08. A extensão de "qualquer role autenticada" ao histórico de entregas e à rotação de secret é **Hipótese (PRD)**: a reunião falou só do "resto do CRUD de configuração" [09:36] Marcos.
- **PRD-NFR-15** (segurança, operação): **Hipótese (PRD)**: a secret nunca deve aparecer em log nem em respostas de consulta. A reunião não definiu essa regra; só relata que um cliente já vazou secret em log de aplicação dele [09:22] Diego (motivação) e que a secret é "devolvida na criação" [09:31] Marcos. As regras de redação no Pino são **Hipótese (FDD)** (FDD-OBS-02).
- **PRD-NFR-16** (observabilidade): logs com o Pino que já existe no projeto, sem ferramenta nova [09:29] Bruno. Métricas e tracing não foram discutidos na reunião; o FDD propõe um conjunto mínimo (**Hipótese (FDD)**, FDD-OBS-04 a FDD-OBS-10), fora do compromisso deste PRD.
- **PRD-NFR-17** (compatibilidade): o módulo segue o padrão dos demais, com `AppError`, código de erro com prefixo `WEBHOOK_` e o error middleware centralizado sem mudança [09:28] Bruno [09:29] Bruno [09:29] Larissa. Os contratos HTTP dos módulos existentes, inclusive `PATCH /orders/:id/status`, não mudam (consequência do desenho, FDD-DEP-09; `src/modules/orders/order.routes.ts`).
- **PRD-NFR-18** (compliance e auditoria): o replay da DLQ registra em log quem o executou, para auditoria [09:36] Sofia. Requisitos regulatórios específicos (por exemplo, retenção de dados) não foram discutidos e não são tratados neste PRD.

## Decisões e trade-offs principais

Cada decisão foi fechada na reunião e tem um ADR. A justificativa e o trade-off abaixo resumem o ADR; o detalhe está no arquivo vinculado.

| ID | Decisão | Justificativa | Trade-off | ADR |
| --- | --- | --- | --- | --- |
| **PRD-DEC-01** | Padrão outbox no MySQL existente | Garante que commit registra o evento e rollback o apaga; evita infraestrutura nova para um time pequeno [09:06] Diego [09:07] Diego [09:08] Larissa | Troca a simplicidade de um HTTP direto por consistência, ao custo de uma tabela extra, um worker e mais carga no MySQL | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **PRD-DEC-02** | Worker em processo separado, com polling de 2 s | Não perde o worker quando a API reinicia; MySQL não tem LISTEN/NOTIFY e 2 s cumpre "menos de 10 s" [09:09] Diego [09:10] Larissa [09:11] Diego | Latência mínima de 2 s no pior caso e ordering só por pedido com um único worker, em troca de simplicidade | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| **PRD-DEC-03** | Retry com backoff 1m/5m/30m/2h/12h, 5 tentativas (leitura adotada: 5 re-envios após a 1ª falha, 14h36m, **Hipótese** a confirmar, ver RFC-OPEN-07) e DLQ em tabela separada | Cobre manutenções longas (2 h) sem deixar evento pendurado para sempre; DLQ serve de evidência e de base para replay [09:15] Diego [09:16] Diego [09:17] Larissa [09:18] Diego | Eventos ficam em retry por até cerca de 15 horas, e indisponibilidades maiores exigem replay manual por ADMIN | [ADR-003](adrs/ADR-003-retry-backoff-exponencial-e-dlq.md) |
| **PRD-DEC-04** | HMAC-SHA256 com secret por endpoint e rotação com 24 h | O cliente valida origem e integridade; secret única limita o dano de um vazamento [09:19] Sofia [09:21] Sofia [09:22] Sofia | Mais complexidade de modelo e de rotação, e prazo reservado para revisão de segurança, em troca de menor raio de dano | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| **PRD-DEC-05** | Entrega at-least-once com `X-Event-Id` | Padrão de mercado (Stripe, GitHub); exactly-once exigiria coordenação dos dois lados [09:25] Diego [09:26] Larissa | O cliente precisa deduplicar; duplicatas são possíveis. Marcos documentará no portal [09:26] Marcos | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| **PRD-DEC-06** | Reuso dos padrões do projeto | Consistência e velocidade dentro de 3 sprints: módulo igual aos outros, `AppError`, Pino, error middleware, Zod e `requireRole` [09:27] Bruno [09:30] Larissa | Herda as limitações dos padrões atuais, em troca de não desenhar convenções novas | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| **PRD-DEC-07** | Evento registrado dentro da transação do `changeStatus` (detalhe da publicação no FDD, seção 11), com filtro de status na inserção | Sem isso "perde a garantia toda"; filtrar na inserção economiza linhas [09:40] Bruno [09:41] Diego [09:34] Bruno | Pequeno acoplamento e mais carga na transação crítica; uma falha do webhook pode derrubar a mudança de status | [ADR-007](adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md) |
| **PRD-DEC-08** | Payload como snapshot renderizado na inserção, sem `items`, com limite de 64 KB e ids em UUID | O evento reflete o pedido no momento da mudança, sem "caso esquisito" depois [09:52] Larissa [09:43] Diego [09:51] Larissa | Mais dados armazenados e trabalho extra na transação; o cliente busca os itens por outra chamada | [ADR-008](adrs/ADR-008-payload-snapshot-na-insercao-da-outbox.md) |

## Dependências

Organizacionais:

- **PRD-DEP-01**: revisão de segurança da Sofia (HMAC e geração de secret) com no mínimo 2 dias úteis reservados antes do deploy, e a Sofia precisa ser agendada [09:46] Sofia [09:47] Larissa [09:49] Sofia.
- **PRD-DEP-02**: portal de desenvolvedor documentado pelo Marcos: como integrar via API, e o aviso destacado sobre at-least-once e deduplicação por `X-Event-Id` [09:26] Marcos [09:40] Marcos.
- **PRD-DEP-03**: confirmação do prazo (fim de novembro) com a Atlas, a cargo do Marcos, e atualização dos clientes sobre a decisão [09:47] Marcos [09:49] Marcos.
- **PRD-DEP-04**: sessão de revisão do design doc com Bruno e Diego, marcada por Larissa antes de começar a codar [09:50] Larissa.
- **PRD-DEP-05**: time de frontend, apenas para um futuro painel do cliente, que está fora de escopo (PRD-OUT-03) [09:40] Larissa.

Técnicas:

- **PRD-DEP-06**: banco MySQL existente, que hospeda também os eventos e a fila de falhas [09:07] Diego; `docker-compose.yml`.
- **PRD-DEP-07**: Prisma como acesso ao banco, com conexão própria do worker e a mesma configuração de banco da API [09:11] Bruno [09:30] Bruno; `prisma/schema.prisma`.
- **PRD-DEP-08**: nova entry-point do worker (a criar; ver FDD, seção 11), rodando como processo separado da API [09:11] Larissa [09:11] Diego.
- **PRD-DEP-09**: componentes existentes reaproveitados: `AppError`, error middleware, Pino, `requireRole`, validação Zod e a transação do `changeStatus` [09:29] Bruno [09:30] Larissa [09:36] Larissa. Detalhes das versões estão no FDD, seção 10 (FDD-DEP-01 a FDD-DEP-11).

## Riscos e mitigação

Probabilidade e impacto são avaliação do PRD (**Hipótese (PRD)**), alinhada aos riscos do FDD (seção 13). Os riscos em si derivam da reunião e do código.

### **PRD-RISK-01** Cliente não deduplica por `X-Event-Id`

- **Probabilidade**: Alta. **Impacto**: Médio.
- Origem: [09:24] Diego [09:25] Sofia ("joga responsabilidade pro cliente"). Ver FDD-RISK-01.
- Mitigação:
  - `X-Event-Id` único e estável em re-tentativas e replay [09:25] Diego.
  - Documentação destacada no portal de desenvolvedor, a cargo do Marcos [09:26] Marcos.
  - **Hipótese (PRD)**: exemplo de deduplicação no portal.
- Plano de contingência: o cliente reconcilia pelo histórico de entregas (PRD-FR-07).

### **PRD-RISK-02** Prazo apertado de 3 sprints, com revisão de segurança no fim

- **Probabilidade**: Média. **Impacto**: Alto (a Atlas pode migrar para um concorrente [09:00] Marcos).
- Origem: [09:46] Larissa [09:47] Larissa [09:45] Marcos.
- Mitigação:
  - Reuso máximo dos padrões do projeto para ganhar velocidade [09:30] Larissa.
  - Ordem de trabalho estimada na reunião [09:46] Larissa: 1 sprint para outbox e DLQ; 1 sprint para worker e retry; meio sprint para CRUD e deliveries; meio sprint para a integração no `order.service` e testes ponta a ponta; HMAC, schemas e validações "mais um pouco"; e a revisão da Sofia (no mínimo 2 dias úteis) no fim, na sprint 3 [09:47] Larissa.
  - Marcos confirma o prazo com a Atlas cedo [09:47] Marcos.
- Plano de contingência: **Hipótese (PRD)**: Marcos renegocia o prazo com a Atlas se algo escorregar. **Hipótese (PRD)**: a revisão de segurança não é cortada para cumprir prazo (Larissa só combinou "três sprints com a revisão da Sofia incluída no fim" [09:47]).

### **PRD-RISK-03** Vazamento de secret

- **Probabilidade**: Média. **Impacto**: Alto.
- Origem: [09:22] Diego (já ocorreu com um cliente) [09:21] Sofia. Ver FDD-RISK-03.
- Mitigação:
  - Secret única por endpoint, limitando o dano a um cadastro [09:21] Sofia.
  - Secret devolvida na criação [09:31] Marcos e na rotação; **Hipótese (PRD)**: nunca em consultas (a reunião não definiu; FDD-CONTRATO-12).
  - Revisão de segurança da Sofia sobre HMAC e geração de secret [09:46] Sofia.
- Plano de contingência: rotação imediata pela API, com 24 h de convivência para o cliente migrar [09:21] Sofia.

### **PRD-RISK-04** Ordering ao escalar o worker

- **Probabilidade**: Média. **Impacto**: Médio.
- Origem: [09:12] Diego [09:13] Diego [09:13] Larissa. Ver FDD-RISK-02 e FDD-RISK-06.
- Mitigação:
  - Operar com um único worker e registrar isso como limitação conhecida [09:13] Larissa.
  - Os clientes nunca pediram ordering global [09:14] Marcos.
  - O payload traz `from_status`, `to_status` e `timestamp`, o que permite ao cliente descartar eventos antigos (FDD-RISK-02, **Hipótese (FDD)**).
- Plano de contingência: particionar por `order_id` ou usar lock pessimista, trabalho futuro fora do escopo [09:13] Diego (PRD-OUT-06).

### **PRD-RISK-05** Atraso na revisão de segurança da Sofia

- **Probabilidade**: Média. **Impacto**: Alto (o deploy depende da revisão).
- Origem: [09:46] Sofia [09:49] Sofia. Ver FDD-RISK-07.
- Mitigação:
  - Agendar a revisão com antecedência, reservando no mínimo 2 dias úteis [09:46] Sofia [09:49] Sofia.
  - Entregar à Sofia o FDD e os critérios de aceite de HMAC e rotação antes do início da revisão (**Hipótese (PRD)**).
- Plano de contingência: se a revisão atrasar, o deploy atrasa junto (a revisão fica no fim dos 3 sprints [09:47] Larissa); **Hipótese (PRD)**: a revisão não é cortada para cumprir prazo.

### **PRD-RISK-06** Falha na outbox derruba a mudança de status

- **Probabilidade**: Baixa. **Impacto**: Alto.
- Origem: [09:40] Bruno [09:41] Diego (aceito conscientemente). Ver ADR-007 e RFC-IMP-05.
- Mitigação:
  - Testes de atomicidade e testes de ponta a ponta em `changeStatus` [09:46] Larissa.
  - Índices e consulta única na inserção do evento (FDD-RISK-08, **Hipótese (FDD)**).
- Plano de contingência: **Hipótese (PRD)**: reverter a integração em `order.service.ts` (uma chamada) e manter a API de pedidos funcionando sem webhooks.

### **PRD-RISK-07** Worker parado ou lento viola os 10 segundos

- **Probabilidade**: Média. **Impacto**: Alto.
- Origem: [09:11] Diego (worker em processo separado). Ver FDD-RISK-05.
- Mitigação:
  - Os eventos são duráveis: reiniciar o worker drena o acúmulo e nada se perde (FDD-RISK-05).
  - Sinal de saúde do worker, por exemplo, idade do evento pendente mais antigo (**Hipótese (FDD)**, FDD-OBS-05).
- Plano de contingência: reinício do worker e, para eventos que esgotaram as tentativas, replay por ADMIN (PRD-FR-09).

### **PRD-RISK-08** CRUD aberto a qualquer role autenticada, com `customerId` no body

- **Probabilidade**: Média. **Impacto**: Alto (um usuário autenticado poderia cadastrar uma URL própria para receber pedidos de outro customer).
- Origem: [09:32] Larissa [09:37] Sofia ("por enquanto sim"). Ver FDD-RISK-10.
- Mitigação:
  - Risco aceito conscientemente nesta fase [09:37] Sofia.
  - Levar o ponto à revisão de segurança (FDD-RISK-10).
- Plano de contingência: endurecer permissões em fase futura (PRD-OUT-16) [09:37] Sofia.

### **PRD-RISK-09** Crescimento da outbox sem arquivamento

- **Probabilidade**: Alta. **Impacto**: Médio.
- Origem: [09:08] Diego (arquivamento fora do escopo). Ver FDD-RISK-04.
- Mitigação:
  - Índices em status e `created_at`, e leitura em lote pequeno [09:08] Diego.
  - Registrar o arquivamento como débito explícito (FDD-RISK-04, RFC-OPEN-04).
- Plano de contingência: **Hipótese (FDD)**: limpeza manual de linhas entregues antigas por script operacional até existir a rotina de arquivamento.

## Critérios de aceitação

Checklist objetivo e verificável no nível do produto. O detalhe técnico (campos, headers, códigos HTTP, formatos) está nos critérios do FDD, seção 12 (FDD-AC-01 a FDD-AC-26), e não é repetido aqui.

- [ ] **PRD-AC-01**: com o endpoint do cliente saudável, o primeiro envio chega em menos de 10 s depois da mudança de status [09:02] Marcos (FDD-AC-01).
- [ ] **PRD-AC-02**: se o registro do evento falhar, a mudança de status é revertida: status, histórico e estoque ficam inalterados [09:40] Bruno (FDD-AC-02).
- [ ] **PRD-AC-03**: após uma mudança de status confirmada, existe um evento para cada webhook ativo do customer que ouve o novo status, e nenhum para os demais [09:34] Bruno (FDD-AC-03, FDD-AC-04).
- [ ] **PRD-AC-04**: o payload traz os campos básicos do pedido e da mudança (pedido, status anterior e novo, customer e total), não contém `items`, e não muda depois de registrado [09:43] Diego [09:52] Larissa (lista de campos: FDD-AC-05, FDD-AC-06).
- [ ] **PRD-AC-05**: payload acima de 64 KB falha com erro e nunca é truncado [09:24] Larissa (FDD-AC-07).
- [ ] **PRD-AC-06**: todo envio leva os headers de identificação e a assinatura HMAC-SHA256 do corpo, que o cliente consegue reproduzir com a secret do webhook [09:44] Diego [09:44] Sofia [09:22] Sofia (headers e formato: FDD-AC-08, FDD-AC-09).
- [ ] **PRD-AC-07**: um endpoint que não responde em 10 s tem a tentativa encerrada como falha [09:42] Diego (FDD-AC-10).
- [ ] **PRD-AC-08**: as 5 tentativas seguem o backoff 1 min, 5 min, 30 min, 2 h e 12 h (leitura adotada: 5 re-envios após a 1ª falha, 14h36m, **Hipótese** a confirmar, ver RFC-OPEN-07); esgotadas, o evento vai para a DLQ e não há nova tentativa automática [09:17] Larissa [09:18] Diego (FDD-AC-11, FDD-AC-12).
- [ ] **PRD-AC-09**: o replay é restrito a ADMIN, recoloca o evento como pendente e registra em log quem o executou [09:36] Sofia [09:36] Larissa (FDD-AC-13, FDD-AC-14).
- [ ] **PRD-AC-10**: a rotação devolve a nova secret uma vez; por 24 h as duas assinaturas são válidas e depois só a nova [09:21] Sofia [09:22] Sofia (FDD-AC-16).
- [ ] **PRD-AC-11**: **Hipótese (PRD)**: nenhuma resposta de consulta traz a secret e nenhum log produzido nos testes a contém (ver PRD-NFR-15; FDD-AC-17).
- [ ] **PRD-AC-12**: URL `http` é recusada com erro de validação e URL https é aceita [09:23] Sofia (FDD-AC-18).
- [ ] **PRD-AC-13**: cadastro, edição, remoção, listagem e rotação funcionam para qualquer role autenticada, com o `customerId` vindo do body ou do path [09:32] Larissa (consulta na listagem: **Hipótese (FDD)**) [09:36] Marcos [09:37] Sofia (FDD-AC-19).
- [ ] **PRD-AC-14**: o histórico de entregas devolve no máximo 100 envios, com resultado, payload, resposta e tempo de resposta [09:34] Marcos (FDD-AC-20).
- [ ] **PRD-AC-15**: o worker roda como processo separado da API, e a API continua mudando status com o worker parado [09:11] Larissa [09:11] Diego (FDD-AC-23).
- [ ] **PRD-AC-16**: os testes existentes continuam passando e o contrato de resposta da mudança de status do pedido não muda (FDD-AC-25).
- [ ] **PRD-AC-17**: a revisão de segurança da Sofia (HMAC e geração de secret, mínimo de 2 dias úteis) está concluída antes do deploy [09:46] Sofia [09:47] Larissa (FDD-AC-26).
- [ ] **PRD-AC-18**: a documentação do portal de desenvolvedor destaca at-least-once e deduplicação por `X-Event-Id` e explica como integrar pela API [09:26] Marcos [09:40] Marcos.
- [ ] **PRD-AC-19**: nenhuma funcionalidade listada em Fora de escopo é entregue (email, rate limiting de saída, dashboard, webhook de entrada, arquivamento automático, ordering global com múltiplos workers); os itens descartados (PRD-OUT-xx de alternativas) estão apenas registrados como decisões.

## Estratégia de testes e validação

O repositório já usa Vitest e supertest em `tests/` (`tests/orders.test.ts`, `tests/auth.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`; `package.json`). A reunião só pede explicitamente "testes ponta a ponta" na integração com o `order.service` [09:46] Larissa. O restante da estratégia é **Hipótese (PRD)**, e os casos detalhados estão nos critérios de aceite técnicos do FDD (seção 12) e em FDD-INT-15.

- **PRD-TEST-01** (testes de integração com supertest): cobrem CRUD de webhooks, rotação, histórico de entregas e replay com os papéis OPERATOR, ADMIN e sem token, validação de URL https e de `customerId`. Origem: **Hipótese (PRD)**, no padrão de `tests/orders.test.ts`.
- **PRD-TEST-02** (testes de atomicidade): mudança de status com falha injetada no registro do evento (rollback) e com sucesso (commit com os eventos esperados), incluindo o filtro por status. Origem: [09:40] Bruno [09:46] Larissa.
- **PRD-TEST-03** (testes de ponta a ponta): do `PATCH /orders/:id/status` até o envio ao endpoint do cliente, com um servidor HTTP de teste que simule sucesso, resposta de falha e atraso maior que 10 s. Origem: [09:46] Larissa; o servidor simulado é **Hipótese (PRD)**.
- **PRD-TEST-04** (testes de unidade): backoff 1m/5m/30m/2h/12h com relógio controlado, limite de 64 KB, cálculo da assinatura HMAC-SHA256 com secret conhecida e janela de 24 h da rotação. Origem: **Hipótese (PRD)**, derivada de [09:17] Larissa, [09:24] Larissa e [09:22] Sofia.
- **PRD-TEST-05** (testes do worker): at-least-once (mesmo `X-Event-Id` ao reenviar), esgotamento para a DLQ e leitura do mais antigo para o mais novo. Origem: **Hipótese (PRD)**, derivada de [09:12] Diego e [09:25] Diego.
- **PRD-TEST-06** (regressão): suíte existente intacta e resposta de `PATCH /orders/:id/status` inalterada. Origem: **Hipótese (PRD)** (FDD-AC-25).
- **PRD-TEST-07** (validação de latência): medir a diferença entre o commit e o primeiro envio em ambiente de teste para confirmar a meta de menos de 10 s. Origem: [09:02] Marcos (meta); método: **Hipótese (PRD)**.
- **PRD-TEST-08** (validação de segurança): revisão da Sofia sobre HMAC e geração de secret, com no mínimo 2 dias úteis, antes do deploy. Origem: [09:46] Sofia [09:47] Larissa.
- **PRD-TEST-09** (validação com o cliente): **Hipótese (PRD)**: Marcos valida com a Atlas, em ambiente de teste, o recebimento e a deduplicação antes da virada.

## Questões em aberto

O detalhe e os revisores estão no [RFC](RFC.md) (RFC-OPEN-01 a RFC-OPEN-13) e no FDD (seção 14). Nenhuma é requisito desta fase.

- Adiadas na reunião: rate limiting de saída, email ao cliente, escala do worker e ordering com múltiplos workers, arquivamento da outbox e permissão do CRUD de webhooks (PRD-OUT-01, PRD-OUT-02, PRD-OUT-05, PRD-OUT-06 e PRD-OUT-16) [09:37] Larissa [09:39] Larissa [09:13] Diego [09:08] Diego [09:37] Sofia.
- Nome do arquivo de processamento do worker (a criar), `webhook.worker.ts` ou `webhook.processor.ts` [09:28] Bruno.
- Lacunas de especificação a fechar na revisão do RFC: semântica de "5 tentativas", formato do `X-Signature`, rota de rotação de secret, armazenamento da secret, evento na criação do pedido, erro de payload acima de 64 KB dentro da transação e relação entre `X-Event-Id`, `event_id` e id da outbox (RFC-OPEN-07 a RFC-OPEN-13).
