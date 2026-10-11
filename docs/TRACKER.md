# Tracker de Rastreabilidade

Este documento liga cada item registrado nos documentos do pacote (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-*.md`) à sua origem na transcrição (`TRANSCRICAO.md`) ou no código existente. Se uma informação não tem linha aqui, ela não tem origem identificável e deve ser ajustada ou removida do documento.

**Como ler**
- **ID**: o identificador em negrito que aparece no próprio documento (ex: `PRD-FR-03`, `FDD-CONTRATO-01`). Rótulos locais do FDD: `FDD-OT-NN` são os objetivos técnicos (seção 2, `OT-NN` no texto) e `FDD-Q-NN` as questões em aberto (seção 14, `Q-NN` no texto). Os ADRs têm uma linha por decisão (`ADR-00N`) e linhas extras de consequências (`ADR-00N-CONS-NN`); só o ADR-007 tem linhas extras de alternativas (`ADR-007-ALT-NN`), as alternativas dos demais ADRs estão rastreadas nas linhas `RFC-ALT-NN`.
- **Fonte**: `TRANSCRICAO` (localização no formato `[hh:mm] Nome`) ou `CODIGO` (caminho de arquivo existente no repositório).
- **Tipo `Hipótese`**: o documento rotula o item como hipótese, mas ele deriva de uma fala ou de um arquivo; a Fonte e a Localização apontam essa derivação, não uma decisão da reunião.
- Itens que são proposta pura, sem fala nem arquivo de origem, não entram na tabela principal: ficam na seção "Itens sem origem identificável" ao final, para ficarem visíveis e serem decididos na revisão.

## Tabela de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RES-01 | docs/PRD.md | Contexto | Feature entrega webhooks de saída a clientes B2B quando o status do pedido muda; cliente cadastra URL e escolhe os status | TRANSCRICAO | [09:00] Marcos; [09:33] Marcos |
| PRD-RES-02 | docs/PRD.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo; tempo real significa menos de 10 segundos | TRANSCRICAO | [09:00] Marcos; [09:02] Marcos |
| PRD-RES-03 | docs/PRD.md | Restrição | Fluxo apenas outbound: a plataforma envia, o cliente recebe, sem webhook de entrada | TRANSCRICAO | [09:02] Marcos; [09:03] Sofia |
| PRD-RES-04 | docs/PRD.md | Decisão | Abordagem acordada: gravar o evento no banco na mesma transação do status e entregar por processo separado com retry e DLQ | TRANSCRICAO | [09:48] Larissa |
| PRD-RES-05 | docs/PRD.md | Restrição | Prazo de 3 sprints incluindo revisão de segurança para atender a Atlas no fim de novembro | TRANSCRICAO | [09:47] Larissa; [09:45] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Clientes B2B fazem polling em GET /orders para saber de mudanças, o que deixa a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Risco | Risco de churn: a Atlas pode migrar para concorrente se não receber a feature até o fim do trimestre | TRANSCRICAO | [09:00] Marcos; [09:45] Marcos |
| PRD-PROB-03 | docs/PRD.md | Problema | Atraso percebido pelo cliente: não pode ficar pendurado nem exigir atualização manual | TRANSCRICAO | [09:02] Marcos |
| PRD-PROB-04 | docs/PRD.md | Problema | Sistema atual não tem notificação externa, evento, fila nem processo em background; changeStatus já é transação pesada | CODIGO | src/modules/orders/order.service.ts |
| PRD-PROB-05 | docs/PRD.md | Problema | Dados de pedidos saem da infraestrutura: cliente precisa validar origem e integridade de cada requisição | TRANSCRICAO | [09:19] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Notificar o cliente em tempo real: menos de 10 s entre a mudança de status e o primeiro envio | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Latência dentro da meta com solução simples: polling de 2 s, latência mínima de 2 s no pior caso aceita | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Zero eventos perdidos ou fantasmas em relação às mudanças de status | TRANSCRICAO | [09:06] Diego; [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Métrica | 5 tentativas com backoff 1m/5m/30m/2h/12h (14h36m) e depois fila de falhas; contagem da 1ª entrega é Hipótese (RFC-OPEN-07) | TRANSCRICAO | [09:17] Larissa; [09:48] Larissa |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Timeout de 10 s por chamada ao cliente; sem resposta vira falha e entra em retry | TRANSCRICAO | [09:42] Diego |
| PRD-OBJ-06 | docs/PRD.md | Métrica | 100% dos envios assinados com HMAC-SHA256, secret única por endpoint e rotação com 24 h de convivência | TRANSCRICAO | [09:22] Sofia |
| PRD-OBJ-07 | docs/PRD.md | Métrica | Cumprir o prazo da Atlas: 3 sprints com revisão de segurança de no mínimo 2 dias úteis, entrega no fim de novembro | TRANSCRICAO | [09:46] Larissa; [09:46] Sofia |
| PRD-ESC-01 | docs/PRD.md | Escopo | Cadastro, edição, remoção e listagem de webhooks por customer, com URL https e lista de status | TRANSCRICAO | [09:33] Bruno; [09:31] Marcos |
| PRD-ESC-02 | docs/PRD.md | Escopo | Secret gerada pela plataforma, uma por endpoint, devolvida na criação, com rotação por API e 24 h de convivência | TRANSCRICAO | [09:21] Sofia; [09:31] Marcos |
| PRD-ESC-03 | docs/PRD.md | Escopo | Evento order.status_changed registrado de forma atômica com a mudança de status, com filtro por status na inserção | TRANSCRICAO | [09:40] Bruno; [09:34] Bruno |
| PRD-ESC-04 | docs/PRD.md | Escopo | Entrega assíncrona por worker separado, polling de 2 s, payload assinado e headers de identificação | TRANSCRICAO | [09:09] Diego; [09:11] Diego |
| PRD-ESC-05 | docs/PRD.md | Escopo | Retry com backoff exponencial e DLQ, com replay manual por ADMIN e registro de quem executou | TRANSCRICAO | [09:17] Larissa; [09:36] Sofia |
| PRD-ESC-06 | docs/PRD.md | Escopo | Histórico das últimas 100 entregas de um webhook, por endpoint | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-07 | docs/PRD.md | Escopo | Garantia at-least-once com X-Event-Id para o cliente deduplicar | TRANSCRICAO | [09:25] Diego; [09:26] Larissa |
| PRD-ESC-08 | docs/PRD.md | Escopo | Reuso dos padrões do projeto: módulo webhooks, AppError, prefixo WEBHOOK_, Pino, error middleware, Zod, requireRole | TRANSCRICAO | [09:30] Larissa; [09:27] Bruno |
| PRD-ESC-09 | docs/PRD.md | Escopo | Documentação no portal de desenvolvedor a cargo do PM: como integrar via API e tratar duplicatas | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-OUT-01 | docs/PRD.md | Fora de Escopo | Email ou notificação ao cliente quando o webhook falha; adiado para possível próxima fase | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| PRD-OUT-02 | docs/PRD.md | Fora de Escopo | Rate limiting de saída por cliente; observar e decidir depois | TRANSCRICAO | [09:39] Larissa; [09:38] Diego |
| PRD-OUT-03 | docs/PRD.md | Fora de Escopo | Dashboard visual ou painel do cliente; projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa; [09:39] Marcos |
| PRD-OUT-04 | docs/PRD.md | Fora de Escopo | Webhooks de entrada (inbound) descartados | TRANSCRICAO | [09:02] Marcos; [09:03] Sofia |
| PRD-OUT-05 | docs/PRD.md | Fora de Escopo | Arquivamento de eventos entregues após cerca de 30 dias fica fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de Escopo | Ordering global e múltiplos workers adiados como problema do futuro e limitação conhecida | TRANSCRICAO | [09:13] Larissa; [09:13] Diego |
| PRD-OUT-07 | docs/PRD.md | Alternativa Descartada | Entrega exactly-once descartada por exigir coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| PRD-OUT-08 | docs/PRD.md | Alternativa Descartada | Retry indefinido descartado: deixaria evento pendurado se o cliente sumisse | TRANSCRICAO | [09:15] Diego |
| PRD-OUT-09 | docs/PRD.md | Alternativa Descartada | Apenas 3 tentativas descartado: não cobre manutenção planejada de 2 h | TRANSCRICAO | [09:16] Diego; [09:16] Bruno |
| PRD-OUT-10 | docs/PRD.md | Alternativa Descartada | Redis Streams ou fila externa descartado: mais infraestrutura para time pequeno | TRANSCRICAO | [09:07] Diego; [09:07] Larissa |
| PRD-OUT-11 | docs/PRD.md | Alternativa Descartada | Trigger do banco ou NOTIFY descartado: MySQL não tem listener nativo | TRANSCRICAO | [09:09] Diego; [09:09] Bruno |
| PRD-OUT-12 | docs/PRD.md | Alternativa Descartada | Chamada HTTP síncrona dentro da mudança de status descartada | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| PRD-OUT-13 | docs/PRD.md | Alternativa Descartada | Id auto incremental na tabela de eventos descartado em favor de UUID | TRANSCRICAO | [09:51] Larissa; [09:51] Diego |
| PRD-OUT-14 | docs/PRD.md | Alternativa Descartada | Truncar payload acima do limite descartado; evento acima de 64 KB falha com erro | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| PRD-OUT-15 | docs/PRD.md | Alternativa Descartada | Itens do pedido (items) fora do payload; cliente consulta GET /orders/:id | TRANSCRICAO | [09:43] Diego; [09:44] Bruno |
| PRD-OUT-16 | docs/PRD.md | Fora de Escopo | Endurecer a permissão do CRUD de webhooks além da autenticação fica para o futuro | TRANSCRICAO | [09:37] Sofia |
| PRD-OUT-17 | docs/PRD.md | Alternativa Descartada | Worker dentro do processo da API descartado: reinício da API derrubaria o worker | TRANSCRICAO | [09:11] Diego |
| PRD-OUT-18 | docs/PRD.md | Alternativa Descartada | Secret global da plataforma descartada: se vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| PRD-OUT-19 | docs/PRD.md | Alternativa Descartada | Guardar só order_id e montar o payload no envio descartado em favor de snapshot | TRANSCRICAO | [09:52] Larissa; [09:51] Bruno |
| PRD-OUT-20 | docs/PRD.md | Hipótese | Evento na criação do pedido não foi tratado na reunião; create não passa por changeStatus; hipótese: fora desta fase | CODIGO | src/modules/orders/order.service.ts |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL https e lista de status; plataforma gera a secret e a devolve só na criação | TRANSCRICAO | [09:31] Marcos; [09:23] Sofia |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | customerId vem no body ou na consulta e não do JWT, pois o JWT é do usuário operador | TRANSCRICAO | [09:32] Larissa; [09:32] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook existente por PATCH | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook por DELETE | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer por GET | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status aplicado na inserção da outbox; sem webhook interessado não insere | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico de entregas GET /webhooks/:id/deliveries com últimos 100 envios, resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com secret antiga válida por 24 h (rota exata é hipótese do FDD) | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay manual da DLQ por POST /admin/webhooks/dead-letter/:id/replay, só ADMIN, com log de quem executou | TRANSCRICAO | [09:18] Diego; [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Worker envia POST com payload assinado HMAC-SHA256 e headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Evento registrado na mesma transação do changeStatus; falha no evento reverte o status | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Payload JSON enxuto sem items, em snapshot no momento do registro do evento | TRANSCRICAO | [09:43] Diego; [09:52] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Notificação em menos de 10 s com o endpoint do cliente saudável | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Polling do worker a cada 2 s; latência mínima de 2 s no pior caso aceita | TRANSCRICAO | [09:09] Diego; [09:10] Larissa |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada HTTP ao cliente; sem resposta é falha e marca para retry | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Payload de no máximo 64 KB com erro acima do limite, sem truncar | TRANSCRICAO | [09:24] Larissa; [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API; reiniciar a API não derruba o worker | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once com X-Event-Id estável para deduplicação no cliente | TRANSCRICAO | [09:24] Diego; [09:25] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Retry com 5 tentativas e backoff 1m/5m/30m/2h/12h, sem retry indefinido; contagem da 1ª entrega é Hipótese (RFC-OPEN-07) | TRANSCRICAO | [09:17] Larissa; [09:48] Larissa |
| PRD-NFR-08 | docs/PRD.md | Restrição | Ordering só por order_id e só com um único worker; sem garantia de ordering global | TRANSCRICAO | [09:13] Larissa; [09:12] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Evento e mudança de status atômicos: zero eventos perdidos ou fantasmas | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Ids em UUID como no resto do projeto e payload como snapshot do momento da mudança | TRANSCRICAO | [09:51] Larissa; [09:52] Larissa |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 sobre o corpo, secret única por endpoint gerada pela plataforma, rotação com 24 h | TRANSCRICAO | [09:22] Sofia; [09:31] Marcos |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | TLS obrigatório: URL do webhook precisa ser https e http é recusado na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-14 | docs/PRD.md | Requisito Não Funcional | CRUD aceita qualquer role autenticada por enquanto; replay exige ADMIN via requireRole | TRANSCRICAO | [09:36] Sofia; [09:37] Sofia |
| PRD-NFR-15 | docs/PRD.md | Requisito Não Funcional | Secret nunca deve aparecer em log nem em consultas; regras de redação no Pino são hipótese do FDD | TRANSCRICAO | [09:22] Diego |
| PRD-NFR-16 | docs/PRD.md | Requisito Não Funcional | Observabilidade com o Pino já existente, sem ferramenta nova; métricas e tracing não discutidos | TRANSCRICAO | [09:29] Bruno |
| PRD-NFR-17 | docs/PRD.md | Requisito Não Funcional | Módulo no padrão dos demais: AppError, prefixo WEBHOOK_ e error middleware sem mudança | TRANSCRICAO | [09:29] Bruno; [09:29] Larissa |
| PRD-NFR-18 | docs/PRD.md | Requisito Não Funcional | Replay da DLQ registra em log quem o executou, para auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-DEC-01 | docs/PRD.md | Decisão | Padrão outbox no MySQL existente, trocando simplicidade do HTTP direto por consistência | TRANSCRICAO | [09:08] Larissa; [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker em processo separado com polling de 2 s; trade-off de latência mínima de 2 s e ordering limitado | TRANSCRICAO | [09:10] Larissa; [09:11] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h, 5 tentativas e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com 24 h | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão | Entrega at-least-once com X-Event-Id; cliente precisa deduplicar | TRANSCRICAO | [09:26] Larissa; [09:25] Diego |
| PRD-DEC-06 | docs/PRD.md | Decisão | Reuso máximo dos padrões do projeto para ganhar consistência e velocidade | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Decisão | Evento publicado dentro da transação do changeStatus por publishWebhookEvent(tx, ...) com filtro na inserção | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| PRD-DEC-08 | docs/PRD.md | Decisão | Payload como snapshot na inserção, sem items, com limite de 64 KB e ids em UUID | TRANSCRICAO | [09:52] Larissa; [09:43] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (HMAC e geração de secret) com no mínimo 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia; [09:49] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Portal de desenvolvedor documentado pelo Marcos: integração via API e aviso sobre at-least-once | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Marcos confirma o prazo de fim de novembro com a Atlas e atualiza os clientes | TRANSCRICAO | [09:47] Marcos; [09:49] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Sessão de revisão do design doc com Bruno e Diego antes de começar a codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Time de frontend apenas para futuro painel do cliente, fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-DEP-06 | docs/PRD.md | Dependência | Banco MySQL existente hospeda também os eventos e a fila de falhas | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-07 | docs/PRD.md | Dependência | Prisma como acesso ao banco, com PrismaClient próprio no worker e mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno; [09:11] Bruno |
| PRD-DEP-08 | docs/PRD.md | Dependência | Nova entry-point src/worker.ts e script npm run worker como processo separado | TRANSCRICAO | [09:11] Larissa; [09:11] Diego |
| PRD-DEP-09 | docs/PRD.md | Dependência | Componentes existentes reaproveitados: AppError, error middleware, Pino, requireRole, Zod e transação do changeStatus | TRANSCRICAO | [09:30] Larissa; [09:29] Bruno |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente não deduplica por X-Event-Id e processa duplicatas | TRANSCRICAO | [09:25] Sofia; [09:24] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Prazo apertado de 3 sprints com revisão de segurança no fim | TRANSCRICAO | [09:46] Larissa; [09:47] Larissa |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret, que já ocorreu com um cliente | TRANSCRICAO | [09:22] Diego; [09:21] Sofia |
| PRD-RISK-04 | docs/PRD.md | Risco | Ordering ao escalar o worker para múltiplas instâncias | TRANSCRICAO | [09:13] Diego; [09:13] Larissa |
| PRD-RISK-05 | docs/PRD.md | Risco | Atraso na revisão de segurança da Sofia atrasa o deploy | TRANSCRICAO | [09:46] Sofia; [09:49] Sofia |
| PRD-RISK-06 | docs/PRD.md | Risco | Falha na outbox derruba a mudança de status; mitigar com testes de atomicidade | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| PRD-RISK-07 | docs/PRD.md | Risco | Worker parado ou lento viola a meta de 10 segundos | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-08 | docs/PRD.md | Risco | CRUD aberto a qualquer role autenticada com customerId no body, aceito por enquanto | TRANSCRICAO | [09:37] Sofia; [09:32] Larissa |
| PRD-RISK-09 | docs/PRD.md | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de Aceite | Primeiro envio chega em menos de 10 s com o endpoint saudável | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceite | Falha no registro do evento reverte status, histórico e estoque | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-03 | docs/PRD.md | Critério de Aceite | Um evento por webhook ativo do customer que ouve o novo status, nenhum para os demais | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-04 | docs/PRD.md | Critério de Aceite | Payload com campos básicos, sem items, imutável depois de registrado | TRANSCRICAO | [09:43] Diego; [09:52] Larissa |
| PRD-AC-05 | docs/PRD.md | Critério de Aceite | Payload acima de 64 KB falha com erro e nunca é truncado | TRANSCRICAO | [09:24] Larissa |
| PRD-AC-06 | docs/PRD.md | Critério de Aceite | Todo envio leva X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e Content-Type; assinatura HMAC reproduzível | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de Aceite | Endpoint sem resposta em 10 s encerra a tentativa como falha | TRANSCRICAO | [09:42] Diego |
| PRD-AC-08 | docs/PRD.md | Critério de Aceite | Retries 1m/5m/30m/2h/12h e, esgotados, evento vai para a DLQ sem nova tentativa automática | TRANSCRICAO | [09:17] Larissa; [09:18] Diego |
| PRD-AC-09 | docs/PRD.md | Critério de Aceite | Replay exige ADMIN, recoloca o evento como pendente e registra em log quem o executou | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| PRD-AC-10 | docs/PRD.md | Critério de Aceite | Rotação devolve nova secret uma vez; por 24 h valem as duas assinaturas e depois só a nova | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| PRD-AC-11 | docs/PRD.md | Critério de Aceite | Nenhuma resposta de consulta nem log de teste contém a secret | TRANSCRICAO | [09:22] Diego |
| PRD-AC-12 | docs/PRD.md | Critério de Aceite | URL http recusada com erro de validação e URL https aceita | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-13 | docs/PRD.md | Critério de Aceite | CRUD e rotação funcionam para qualquer role autenticada, com customerId vindo do body ou da consulta | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos |
| PRD-AC-14 | docs/PRD.md | Critério de Aceite | Histórico de entregas devolve no máximo 100 envios com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-15 | docs/PRD.md | Critério de Aceite | Worker roda por npm run worker como processo separado e a API muda status com o worker parado | TRANSCRICAO | [09:11] Larissa; [09:11] Diego |
| PRD-AC-16 | docs/PRD.md | Critério de Aceite | Testes existentes continuam passando e a resposta de PATCH /orders/:id/status não muda | CODIGO | src/modules/orders/order.routes.ts |
| PRD-AC-17 | docs/PRD.md | Critério de Aceite | Revisão de segurança da Sofia concluída antes do deploy, com no mínimo 2 dias úteis | TRANSCRICAO | [09:46] Sofia; [09:47] Larissa |
| PRD-AC-18 | docs/PRD.md | Critério de Aceite | Portal destaca at-least-once, deduplicação por X-Event-Id e como integrar pela API | TRANSCRICAO | [09:26] Marcos; [09:40] Marcos |
| PRD-AC-19 | docs/PRD.md | Critério de Aceite | Itens fora de escopo PRD-OUT-01 a PRD-OUT-20 não são implementados nesta fase | TRANSCRICAO | [09:48] Larissa |
| PRD-TEST-01 | docs/PRD.md | Hipótese | Testes de integração com supertest para CRUD, rotação, histórico e replay, no padrão da suíte existente | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Critério de Aceite | Testes de atomicidade do changeStatus com falha injetada (rollback) e sucesso (commit), incluindo filtro | TRANSCRICAO | [09:40] Bruno; [09:46] Larissa |
| PRD-TEST-03 | docs/PRD.md | Critério de Aceite | Testes ponta a ponta do PATCH /orders/:id/status até o envio ao endpoint do cliente | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-04 | docs/PRD.md | Hipótese | Testes de unidade de backoff, limite de 64 KB, HMAC e janela de 24 h, derivados das falas sobre esses valores | TRANSCRICAO | [09:17] Larissa; [09:24] Larissa |
| PRD-TEST-05 | docs/PRD.md | Hipótese | Testes do worker: at-least-once, esgotamento para DLQ e leitura do mais antigo ao mais novo | TRANSCRICAO | [09:12] Diego; [09:25] Diego |
| PRD-TEST-06 | docs/PRD.md | Hipótese | Teste de regressão: suíte existente intacta e resposta do PATCH de status inalterada | CODIGO | tests/orders.test.ts |
| PRD-TEST-07 | docs/PRD.md | Hipótese | Validação de latência: medir commit até primeiro envio para confirmar a meta de 10 s (método é hipótese) | TRANSCRICAO | [09:02] Marcos |
| PRD-TEST-08 | docs/PRD.md | Critério de Aceite | Validação de segurança: revisão da Sofia sobre HMAC e geração de secret, mínimo de 2 dias úteis | TRANSCRICAO | [09:46] Sofia; [09:47] Larissa |
| RFC-TLDR | docs/RFC.md | Decisão | Resumo executivo: webhooks de saída em menos de 10 s via outbox MySQL, worker com polling, retry, DLQ, HMAC e at-least-once | TRANSCRICAO | [09:48] Larissa; [09:47] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Atlas, MaxDistribuição e Nova Cargo pediram notificação de mudança de status e hoje fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Tempo real para esses clientes é qualquer coisa abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Escopo só outbound, sem webhook de entrada | TRANSCRICAO | [09:02] Marcos; [09:03] Sofia |
| RFC-CTX-04 | docs/RFC.md | Contexto | Código atual sem eventos, filas ou background; changeStatus é transação pesada | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-05 | docs/RFC.md | Restrição | Pressão de prazo (Atlas, fim de novembro) e time pequeno que evita infraestrutura nova | TRANSCRICAO | [09:00] Marcos; [09:07] Diego |
| RFC-PROP-01 | docs/RFC.md | Decisão | publishWebhookEvent(tx, ...) insere na webhook_outbox dentro da transação do changeStatus, filtrando por status | TRANSCRICAO | [09:06] Diego; [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Nova entry-point src/worker.ts e npm run worker, processo separado, PrismaClient próprio, worker único com polling de 2 s | TRANSCRICAO | [09:11] Larissa; [09:30] Bruno |
| RFC-PROP-03 | docs/RFC.md | Decisão | Timeout de 10 s, retry 1m/5m/30m/2h/12h e DLQ separada com replay restrito a ADMIN e auditado | TRANSCRICAO | [09:17] Larissa; [09:42] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256 sobre o corpo, secret única por endpoint gerada pela plataforma, rotação com grace de 24 h | TRANSCRICAO | [09:22] Sofia; [09:31] Marcos |
| RFC-PROP-05 | docs/RFC.md | Decisão | Garantia at-least-once com deduplicação pelo X-Event-Id no cliente | TRANSCRICAO | [09:25] Diego |
| RFC-PROP-06 | docs/RFC.md | Decisão | Payload enxuto em snapshot, limite de 64 KB com erro e URL https obrigatória | TRANSCRICAO | [09:43] Diego; [09:52] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | Módulo src/modules/webhooks com erros WEBHOOK_* sobre AppError, Pino, errorMiddleware e requireRole | TRANSCRICAO | [09:27] Bruno; [09:30] Larissa |
| RFC-PROP-08 | docs/RFC.md | Requisito Funcional | Superfície de API: CRUD por customer, filtro por status, rotação de secret, deliveries e replay | TRANSCRICAO | [09:33] Bruno; [09:34] Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Webhook HTTP síncrono no changeStatus: cliente lento travaria outros pedidos e fora do ar não pode causar rollback | TRANSCRICAO | [09:04] Bruno; [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams ou fila externa: mais infraestrutura, overengineering para time pequeno | TRANSCRICAO | [09:07] Larissa; [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger do banco estilo LISTEN/NOTIFY: MySQL não tem listener nativo e polling de 2 s cumpre a meta | TRANSCRICAO | [09:09] Diego; [09:09] Bruno |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Worker dentro do processo da API: reinício da API derrubaria o worker | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Retry indefinido com backoff: evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Apenas 3 tentativas: janela de cerca de 30 min mataria evento de cliente com 2 h de manutenção | TRANSCRICAO | [09:16] Diego; [09:16] Bruno |
| RFC-ALT-07 | docs/RFC.md | Alternativa Descartada | DLQ como status failed na própria outbox: tabela separada deixa a leitura limpa e serve de evidência | TRANSCRICAO | [09:18] Diego; [09:17] Larissa |
| RFC-ALT-08 | docs/RFC.md | Alternativa Descartada | Entrega exactly-once: exigiria coordenação dos dois lados; at-least-once com X-Event-Id cobre 99% dos casos | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-09 | docs/RFC.md | Alternativa Descartada | Secret global da plataforma: se vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-10 | docs/RFC.md | Alternativa Descartada | Guardar só order_id na outbox e renderizar no envio: evento refletiria estado posterior do pedido | TRANSCRICAO | [09:52] Larissa; [09:51] Bruno |
| RFC-ALT-11 | docs/RFC.md | Alternativa Descartada | Id auto incremental na outbox: foge do padrão UUID do projeto | TRANSCRICAO | [09:51] Larissa; [09:51] Diego |
| RFC-ALT-12 | docs/RFC.md | Alternativa Descartada | Truncar payload acima do limite: evento grande indica problema, preferido falhar com erro | TRANSCRICAO | [09:23] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída por cliente: fora do escopo, observar e decidir depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Email ou alerta ao cliente quando o webhook falha repetidamente, talvez na próxima fase | TRANSCRICAO | [09:37] Larissa; [09:38] Marcos |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Escala do worker e ordering com múltiplos workers por particionamento ou lock pessimista | TRANSCRICAO | [09:13] Diego; [09:12] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | Arquivamento e retenção da outbox após cerca de 30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em Aberto | Endurecer a permissão do CRUD de configuração, hoje qualquer role autenticada | TRANSCRICAO | [09:37] Sofia; [09:36] Marcos |
| RFC-OPEN-06 | docs/RFC.md | Questão em Aberto | Nome do arquivo de processamento do worker: webhook.worker.ts ou webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| RFC-OPEN-07 | docs/RFC.md | Questão em Aberto | Interpretação de 5 tentativas: hipótese de 5 re-tentativas após a primeira falha, soma 14h36m | TRANSCRICAO | [09:17] Diego |
| RFC-OPEN-08 | docs/RFC.md | Questão em Aberto | Formato do valor de X-Signature e se X-Timestamp entra no material assinado; reunião só diz HMAC sobre o corpo | TRANSCRICAO | [09:22] Sofia |
| RFC-OPEN-09 | docs/RFC.md | Questão em Aberto | Rota de rotação de secret: reunião só fala em endpoint para o cliente pedir nova secret | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-10 | docs/RFC.md | Questão em Aberto | Armazenamento da secret (cifrada ou hash); hipótese de que hash não serve pois a plataforma precisa dela para assinar | TRANSCRICAO | [09:21] Bruno; [09:21] Sofia |
| RFC-OPEN-11 | docs/RFC.md | Questão em Aberto | Evento na criação do pedido: create cria o pedido PENDING e não passa por changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-OPEN-12 | docs/RFC.md | Questão em Aberto | Erro de payload acima de 64 KB dentro da transação aborta a mudança de status? Reunião não decidiu | TRANSCRICAO | [09:24] Larissa; [09:40] Bruno |
| RFC-OPEN-13 | docs/RFC.md | Questão em Aberto | Relação entre X-Event-Id, event_id do payload e id da linha da outbox | TRANSCRICAO | [09:25] Diego; [09:43] Diego |
| RFC-IMP-01 | docs/RFC.md | Dependência | changeStatus ganha chamada a publishWebhookEvent; novos src/worker.ts, módulo webhooks e tabelas; sem mudança de contrato | TRANSCRICAO | [09:41] Bruno; [09:29] Bruno |
| RFC-IMP-02 | docs/RFC.md | Risco | Processo novo para deploy, restart e monitoramento, com pool próprio e consulta ao MySQL a cada 2 s | TRANSCRICAO | [09:11] Diego; [09:30] Bruno |
| RFC-IMP-03 | docs/RFC.md | Restrição | Cronograma de 3 sprints com revisão de segurança de no mínimo 2 dias úteis no fim | TRANSCRICAO | [09:47] Larissa; [09:46] Sofia |
| RFC-IMP-04 | docs/RFC.md | Restrição | Limitação conhecida: sem ordering global, só por order_id e com um único worker | TRANSCRICAO | [09:13] Larissa; [09:14] Marcos |
| RFC-IMP-05 | docs/RFC.md | Risco | Falha na outbox derruba a mudança de status; mitigação com testes ponta a ponta | TRANSCRICAO | [09:40] Bruno; [09:46] Larissa |
| RFC-IMP-06 | docs/RFC.md | Risco | Crescimento da outbox sem arquivamento; mitigação com índices e batch pequeno | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-07 | docs/RFC.md | Risco | Vazamento de secret já ocorrido; mitigação com secret por endpoint, rotação e revisão da Sofia | TRANSCRICAO | [09:22] Diego |
| RFC-IMP-08 | docs/RFC.md | Risco | Duplicatas chegam ao cliente; mitigação com X-Event-Id e documentação no portal | TRANSCRICAO | [09:26] Marcos; [09:24] Diego |
| RFC-IMP-09 | docs/RFC.md | Risco | Cliente fora do ar por mais de cerca de 15 horas só recupera o evento por replay manual | TRANSCRICAO | [09:17] Marcos |
| RFC-IMP-10 | docs/RFC.md | Hipótese | Evento em retry pode ser ultrapassado por evento posterior do mesmo pedido, pois o ordering vale só para a leitura | TRANSCRICAO | [09:12] Diego |
| RFC-IMP-11 | docs/RFC.md | Fora de Escopo | Fora de escopo: email ao cliente, rate limiting de saída, dashboard, webhooks de entrada e arquivamento | TRANSCRICAO | [09:37] Larissa; [09:39] Diego |
| RFC-ADR-01 | docs/RFC.md | Decisão | Referência ao ADR-001: padrão outbox no MySQL existente | TRANSCRICAO | [09:08] Larissa |
| RFC-ADR-02 | docs/RFC.md | Decisão | Referência ao ADR-002: worker em processo separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-ADR-03 | docs/RFC.md | Decisão | Referência ao ADR-003: retry com backoff exponencial e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| RFC-ADR-04 | docs/RFC.md | Decisão | Referência ao ADR-004: HMAC-SHA256 com secret por endpoint e rotação com grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| RFC-ADR-05 | docs/RFC.md | Decisão | Referência ao ADR-005: entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-ADR-06 | docs/RFC.md | Decisão | Referência ao ADR-006: reuso dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| RFC-ADR-07 | docs/RFC.md | Decisão | Referência ao ADR-007: publicação do evento na transação do changeStatus | TRANSCRICAO | [09:41] Diego |
| RFC-ADR-08 | docs/RFC.md | Decisão | Referência ao ADR-008: payload como snapshot na inserção da outbox | TRANSCRICAO | [09:52] Bruno |
| RFC-OPEN-14 | docs/RFC.md | Questão em Aberto | Papel de WEBHOOK_INVALID_URL versus VALIDATION_ERROR do Zod para URL http; a reunião não reconciliou os dois | TRANSCRICAO | [09:23] Sofia; [09:28] Bruno |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL existente: evento gravado na mesma transação e lido por worker separado | TRANSCRICAO | [09:08] Larissa; [09:06] Diego |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Consistência entre status e evento em troca de tabela extra, worker e mais carga no MySQL, preferindo MySQL a broker | TRANSCRICAO | [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado com polling de 2 s, PrismaClient próprio e worker único | TRANSCRICAO | [09:10] Larissa; [09:11] Diego |
| ADR-002-CONS-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Latência mínima de 2 s e ordering só por order_id em troca de simplicidade e sem infraestrutura nova | TRANSCRICAO | [09:10] Larissa; [09:13] Larissa |
| ADR-002-CONS-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Hipótese | Novas variáveis de ambiente do worker, se necessárias, entrariam em src/config/env.ts | CODIGO | src/config/env.ts |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h, 5 tentativas, DLQ webhook_dead_letter e replay admin | TRANSCRICAO | [09:17] Larissa; [09:15] Diego |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Trade-off | Teto finito de 5 tentativas em vez de retry indefinido, aceitando replay manual para indisponibilidades longas | TRANSCRICAO | [09:15] Diego; [09:16] Diego |
| ADR-003-CONS-02 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Consequência | Evento de cliente fora do ar por mais de cerca de 15 horas só é entregue por replay manual | TRANSCRICAO | [09:17] Marcos |
| ADR-003-CONS-03 | docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md | Hipótese | Interação entre retry e ordering: evento em retry e evento posterior do mesmo pedido, risco a analisar no FDD | TRANSCRICAO | [09:13] Larissa; [09:12] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo com secret única por endpoint e rotação com grace de 24 h | TRANSCRICAO | [09:22] Sofia; [09:20] Sofia |
| ADR-004-CONS-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Segurança por endpoint com rotação em duas fases em troca de mais complexidade e prazo de revisão de segurança | TRANSCRICAO | [09:21] Sofia; [09:46] Sofia |
| ADR-004-CONS-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Hipótese | Adicionar redact de secret e assinatura no Pino, motivado pelo vazamento relatado; lista atual de redact não cobre secret | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com X-Event-Id (UUID gerado na outbox) e deduplicação pelo cliente | TRANSCRICAO | [09:26] Larissa; [09:25] Diego |
| ADR-005-CONS-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Entrega mais simples do nosso lado em troca de o cliente implementar a deduplicação por X-Event-Id | TRANSCRICAO | [09:25] Diego; [09:26] Larissa |
| ADR-005-CONS-02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Dependência | Dependência uuid 11.0.3 já presente no package.json para gerar o UUID do evento | CODIGO | package.json |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões do projeto: módulo, AppError, prefixo WEBHOOK_, Pino, error middleware, Zod e requireRole | TRANSCRICAO | [09:30] Larissa; [09:27] Bruno |
| ADR-006-CONS-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | Consistência e velocidade de entrega em vez de liberdade para convenções ideais, herdando limitações atuais | TRANSCRICAO | [09:30] Larissa; [09:47] Larissa |
| ADR-006-CONS-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Consequência | NotFoundError fixa o código NOT_FOUND, então WEBHOOK_NOT_FOUND exige outra subclasse ou AppError direto | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CONS-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Consequência | errorMiddleware só trata AppError, ZodError e Prisma P2002 e P2025; erros do worker não passam por ele | CODIGO | src/middlewares/error.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md | Decisão | Evento publicado na mesma transação do changeStatus por publishWebhookEvent(tx, ...) com filtro na inserção | TRANSCRICAO | [09:41] Diego; [09:40] Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md | Alternativa Descartada | Injetar o repository de webhook no OrderService; preferida função recebendo o tx | TRANSCRICAO | [09:41] Diego; [09:41] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md | Alternativa Descartada | Filtrar só na hora do envio pelo worker; rejeitada em favor de filtrar na inserção | TRANSCRICAO | [09:34] Bruno; [09:34] Diego |
| ADR-007-CONS-01 | docs/adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md | Trade-off | Acoplamento leve e mais carga na transação crítica em troca de consistência forte entre status e evento | TRANSCRICAO | [09:41] Diego |
| ADR-007-CONS-02 | docs/adrs/ADR-007-publicacao-do-evento-na-transacao-do-change-status.md | Consequência | Alteração crítica em código existente exige testes ponta a ponta | TRANSCRICAO | [09:46] Larissa |
| ADR-008 | docs/adrs/ADR-008-payload-snapshot-na-insercao-da-outbox.md | Decisão | Payload como snapshot renderizado na inserção da outbox, sem items, limite de 64 KB e id em UUID | TRANSCRICAO | [09:52] Larissa; [09:52] Bruno |
| ADR-008-CONS-01 | docs/adrs/ADR-008-payload-snapshot-na-insercao-da-outbox.md | Trade-off | Fidelidade ao momento da mudança e evento autocontido em troca de mais dados armazenados e trabalho na transação | TRANSCRICAO | [09:52] Larissa; [09:43] Diego |
| FDD-OT-01 | docs/FDD.md | Requisito Não Funcional | Atomicidade: evento na outbox gravado na mesma transação do changeStatus, rollback se o insert falhar | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-OT-02 | docs/FDD.md | Requisito Não Funcional | Latência ponta a ponta abaixo de 10 s, com polling de 2 s | TRANSCRICAO | [09:02] Marcos; [09:09] Diego; [09:10] Larissa |
| FDD-OT-03 | docs/FDD.md | Requisito Não Funcional | Nenhuma chamada HTTP de saída dentro da transação nem da API; worker em processo separado | TRANSCRICAO | [09:04] Bruno; [09:11] Diego; [09:11] Larissa |
| FDD-OT-04 | docs/FDD.md | Requisito Não Funcional | Entrega at-least-once com event_id UUID único em X-Event-Id, estável em retries e replay | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Larissa |
| FDD-OT-05 | docs/FDD.md | Requisito Não Funcional | Retry limitado com backoff 1m/5m/30m/2h/12h e DLQ, nunca retry indefinido | TRANSCRICAO | [09:15] Diego; [09:17] Larissa; [09:18] Diego |
| FDD-OT-06 | docs/FDD.md | Requisito Não Funcional | HMAC-SHA256 sobre o corpo, secret única por endpoint e rotação com grace de 24 h | TRANSCRICAO | [09:20] Sofia; [09:21] Sofia; [09:22] Sofia |
| FDD-OT-07 | docs/FDD.md | Requisito Não Funcional | URL obrigatoriamente https e payload de no máximo 64KB, com erro e nunca truncamento | TRANSCRICAO | [09:23] Sofia; [09:24] Larissa |
| FDD-OT-08 | docs/FDD.md | Requisito Não Funcional | Reuso máximo do projeto: AppError, Pino, error middleware sem mudança, sem dependência de runtime nova | TRANSCRICAO | [09:29] Bruno; [09:30] Larissa |
| FDD-OT-09 | docs/FDD.md | Requisito Não Funcional | Operabilidade mínima: logs estruturados Pino com eventId, webhookId e orderId, sem secret | TRANSCRICAO | [09:29] Bruno |
| FDD-FLX-01 | docs/FDD.md | Fluxo | Publicação transacional do evento: publishWebhookEvent(tx, ...) chamada dentro do $transaction do changeStatus | TRANSCRICAO | [09:40] Bruno; [09:41] Diego; [09:06] Diego |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Filtro de eventos na inserção da outbox: sem webhook interessado no to_status, nada é inserido | TRANSCRICAO | [09:33] Marcos; [09:34] Bruno; [09:34] Diego |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Loop de polling de 2 s no worker, leitura de pendentes em batch pequeno e claim por guarda de status | TRANSCRICAO | [09:09] Diego; [09:10] Larissa; [09:11] Diego; [09:08] Diego |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Montagem do request, assinatura HMAC dos bytes do corpo e POST com timeout de 10 s | TRANSCRICAO | [09:42] Diego; [09:44] Diego; [09:44] Sofia; [09:20] Sofia |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Resultado de sucesso: outbox DELIVERED e registro de entrega com resposta e tempo | TRANSCRICAO | [09:08] Diego; [09:34] Marcos |
| FDD-FLX-06 | docs/FDD.md | Fluxo | Falha e reagendamento com backoff 1m/5m/30m/2h/12h, interpretação de 5 re-tentativas após a falha original | TRANSCRICAO | [09:15] Diego; [09:17] Diego; [09:17] Larissa; [09:42] Diego |
| FDD-FLX-07 | docs/FDD.md | Fluxo | Esgotamento das tentativas: outbox FAILED e cópia com payload, motivo e timestamp em webhook_dead_letter | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-FLX-08 | docs/FDD.md | Fluxo | Replay manual da DLQ por ADMIN, recolocando o evento como pendente e logando quem fez o replay | TRANSCRICAO | [09:18] Diego; [09:35] Diego; [09:36] Sofia; [09:36] Larissa |
| FDD-FLX-09 | docs/FDD.md | Fluxo | Rotação de secret com grace de 24 h em que a secret anterior segue válida em paralelo (rota é hipótese do FDD) | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia; [09:22] Diego |
| FDD-FLX-11 | docs/FDD.md | Hipótese | Encerramento gracioso do worker em SIGINT/SIGTERM com prisma.$disconnect, espelhando o server.ts (proposta do FDD) | CODIGO | src/server.ts |
| FDD-DATA-01 | docs/FDD.md | Modelo de Dados | Tabela webhooks de configuração: url, secret, customer_id, estado ativo, lista de status e secret anterior com grace | TRANSCRICAO | [09:21] Bruno; [09:33] Marcos; [09:21] Sofia |
| FDD-DATA-02 | docs/FDD.md | Modelo de Dados | Tabela webhook_outbox com id uuid, event_id uuid, payload snapshot, status e índices em status e created_at | TRANSCRICAO | [09:08] Diego; [09:51] Larissa; [09:52] Larissa |
| FDD-DATA-03 | docs/FDD.md | Modelo de Dados | Tabela webhook_dead_letter com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DATA-04 | docs/FDD.md | Modelo de Dados | Tabela webhook_deliveries com histórico por tentativa: sucesso ou falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-05 | docs/FDD.md | Modelo de Dados | Enum de status da outbox (PENDING, PROCESSING, DELIVERED, FAILED) e enum de resultado da entrega (hipótese) | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-06 | docs/FDD.md | Modelo de Dados | Índices obrigatórios em status e created_at da outbox; demais índices são hipótese de desempenho | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-10 | docs/FDD.md | Modelo de Dados | Alterações em models existentes limitadas a campos de relação em Customer e User, sem mudar colunas atuais | CODIGO | prisma/schema.prisma |
| FDD-DATA-11 | docs/FDD.md | Restrição | Retenção e arquivamento das tabelas de webhook fora do escopo da feature | TRANSCRICAO | [09:08] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks cadastra webhook com customerId, url https e events; secret gerada pela plataforma e devolvida só na resposta | TRANSCRICAO | [09:31] Marcos; [09:32] Larissa; [09:23] Sofia |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks?customerId= lista os webhooks de um customer, sem secret na resposta | TRANSCRICAO | [09:33] Bruno; [09:32] Larissa |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id edita url, events ou estado ativo | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id remove o webhook, com 204 e remoção lógica (hipótese) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries devolve os últimos 100 envios com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Hipótese | POST /webhooks/:id/rotate-secret como rota proposta para a rotação de secret, cuja capacidade foi pedida na reunião | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay exige role ADMIN e registra quem executou | TRANSCRICAO | [09:18] Diego; [09:35] Diego; [09:36] Sofia; [09:36] Larissa |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers de saída: Content-Type, X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id | TRANSCRICAO | [09:44] Diego; [09:44] Sofia; [09:45] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Corpo order.status_changed em snake_case, snapshot sem items e de até 64KB | TRANSCRICAO | [09:43] Diego; [09:44] Bruno; [09:52] Larissa |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Assinatura de publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx da transação atual | TRANSCRICAO | [09:41] Bruno; [09:41] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Hipótese | Formato do X-Signature (sha256=hex, timestamp fora do material assinado) não decidido na reunião, que só definiu HMAC-SHA256 sobre o corpo | TRANSCRICAO | [09:22] Sofia; [09:20] Sofia |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Convenções da API de configuração: JWT em qualquer role, customerId no body ou query e nunca do JWT, replay só ADMIN | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos; [09:37] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND (404) para webhook inexistente ou removido | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (400) para URL inválida do módulo | TRANSCRICAO | [09:28] Bruno; [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED (422) para webhook sem secret utilizável | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (422) para corpo acima de 64KB, com rollback do changeStatus (código proposto) | TRANSCRICAO | [09:24] Larissa; [09:23] Sofia |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT: sem resposta em 10 s conta como falha da tentativa e vai para retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED: resposta não 2xx ou erro de rede conta como falha, com retry e depois DLQ (código proposto) | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro | FORBIDDEN (403) existente no replay por usuário não ADMIN, reaproveitando requireRole | TRANSCRICAO | [09:36] Larissa |
| FDD-ERR-11 | docs/FDD.md | Erro | VALIDATION_ERROR (400) existente para body, query ou params inválidos via validate() | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERR-12 | docs/FDD.md | Erro | NOT_FOUND (404) existente quando o customerId do POST não existe, padrão do OrderService.create | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERR-13 | docs/FDD.md | Erro | UNAUTHORIZED (401) existente para JWT ausente ou inválido | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10 s por chamada HTTP de saída; sem resposta é falha e vai para retry | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Requisito Não Funcional | 5 tentativas com backoff exponencial 1m/5m/30m/2h/12h; retry indefinido e 3 tentativas descartados | TRANSCRICAO | [09:15] Diego; [09:16] Diego; [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Decisão | Fallback por DLQ com reprocessamento manual por ADMIN, sem email ou alerta ao cliente | TRANSCRICAO | [09:17] Diego; [09:18] Diego; [09:37] Larissa |
| FDD-RES-04 | docs/FDD.md | Restrição | Circuit breaker não previsto e rate limiting de saída fora do escopo (observar e decidir depois) | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| FDD-RES-05 | docs/FDD.md | Decisão | Atomicidade: evento gravado na mesma transação SQL da mudança de status, commit registra e rollback apaga | TRANSCRICAO | [09:06] Diego; [09:40] Bruno |
| FDD-RES-06 | docs/FDD.md | Decisão | Entrega at-least-once com deduplicação pelo cliente via X-Event-Id; exactly-once descartado | TRANSCRICAO | [09:24] Diego; [09:25] Diego; [09:26] Marcos |
| FDD-RES-07 | docs/FDD.md | Restrição | Ordering só por order_id e apenas com worker único; sem ordering global (limitação conhecida) | TRANSCRICAO | [09:12] Diego; [09:13] Larissa; [09:14] Marcos |
| FDD-RES-08 | docs/FDD.md | Requisito Não Funcional | Proteção contra double-claim por UPDATE condicional com guarda de status, premissa de worker único | TRANSCRICAO | [09:12] Diego |
| FDD-RES-09 | docs/FDD.md | Requisito Não Funcional | Transações curtas no worker, sem conexão ou lock aberto durante o POST de até 10 s | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-11 | docs/FDD.md | Hipótese | Loop do worker não morre por erro de tick e bootstrap falho segue o padrão logger.fatal e exit(1) do server.ts | CODIGO | src/server.ts |
| FDD-RES-12 | docs/FDD.md | Requisito Não Funcional | Isolamento de falhas: API e worker são processos separados e um não derruba o outro | TRANSCRICAO | [09:11] Diego |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Logs estruturados com o Pino existente e eventos nomeados em snake_case | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Nunca logar secret, signature nem corpo de resposta do cliente; redact do Pino como reforço (hipótese) | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-05 | docs/FDD.md | Hipótese | Métrica de idade do evento PENDING mais antigo, sinal direto do requisito de menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-06 | docs/FDD.md | Hipótese | Métricas de tentativas e duração das entregas a partir do tempo de resposta registrado em webhook_deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-09 | docs/FDD.md | Hipótese | Propagação de requestId não é possível sem mudar a assinatura de publishWebhookEvent; elo por orderId e horário | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node.js >=20 como runtime da API e do worker, com fetch nativo (hipótese) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8.0 do docker-compose para outbox, DLQ, config e deliveries | CODIGO | docker-compose.yml |
| FDD-DEP-03 | docs/FDD.md | Dependência | Prisma 5.22.0 para models novos, migration, transação e claim com updateMany | CODIGO | package.json |
| FDD-DEP-04 | docs/FDD.md | Dependência | Express 4.21.1 para as rotas /webhooks e /admin/webhooks | CODIGO | package.json |
| FDD-DEP-05 | docs/FDD.md | Dependência | Zod 3.23.8 para os schemas do módulo (https, enums, uuid) | CODIGO | package.json |
| FDD-DEP-06 | docs/FDD.md | Dependência | Pino 9.5.0 reaproveitado para logs do worker e do módulo | TRANSCRICAO | [09:29] Bruno |
| FDD-DEP-07 | docs/FDD.md | Dependência | uuid 11.0.3 para o event_id e demais ids na aplicação | CODIGO | package.json |
| FDD-DEP-09 | docs/FDD.md | Restrição | Compatibilidade: nenhum contrato de pedidos muda e nenhuma coluna existente é alterada | TRANSCRICAO | [09:30] Larissa |
| FDD-DEP-10 | docs/FDD.md | Dependência | jsonwebtoken 9.0.2 usado pelo authenticate existente, sem mudança | CODIGO | package.json |
| FDD-DEP-11 | docs/FDD.md | Dependência | tsx, Vitest e supertest para o script worker e os testes | CODIGO | package.json |
| FDD-INT-01 | docs/FDD.md | Integração | order.service.ts alterado: changeStatus chama publishWebhookEvent(tx, order, from, to) dentro do $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | order.status.ts sem mudança: transições definem os pares from/to válidos de cada evento | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | http-errors.ts ganha classes WEBHOOK_* no estilo de InsufficientStockError, com AppError como base | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | error.middleware.ts sem mudança: erros WEBHOOK_* saem pelo ramo de AppError | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | auth.middleware.ts sem mudança: authenticate nas rotas e requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate.middleware.ts sem mudança, com schemas Zod novos do módulo (https, events, uuid) | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | routes/index.ts alterado: Controllers ganha webhooks e monta /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | app.ts alterado: buildControllers instancia repository, service e controller de webhooks | CODIGO | src/app.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Entry-point src/worker.ts replica o bootstrap do server.ts e package.json ganha o script worker | CODIGO | src/server.ts |
| FDD-INT-10 | docs/FDD.md | Integração | database.ts sem mudança: o worker obtém seu próprio PrismaClient por processo com a mesma DATABASE_URL | CODIGO | src/config/database.ts |
| FDD-INT-11 | docs/FDD.md | Integração | env.ts ganha chaves opcionais do worker com default (hipótese do FDD) | CODIGO | src/config/env.ts |
| FDD-INT-12 | docs/FDD.md | Integração | logger/index.ts reaproveitado, com redact de secret e signature como reforço opcional (hipótese) | CODIGO | src/shared/logger/index.ts |
| FDD-INT-13 | docs/FDD.md | Integração | response.ts sem mudança: paginated formata as listagens de webhooks e deliveries | CODIGO | src/shared/http/response.ts |
| FDD-INT-14 | docs/FDD.md | Integração | schema.prisma alterado com 4 models e 2 enums novos mais migration nova, sem mudar colunas existentes | CODIGO | prisma/schema.prisma |
| FDD-INT-15 | docs/FDD.md | Integração | tests/setup.ts e factories ajustados para limpar e criar as tabelas novas (hipótese do FDD) | CODIGO | tests/setup.ts |
| FDD-INT-16 | docs/FDD.md | Integração | request-logger.middleware.ts sem mudança: rotas novas herdam o log http_request com userId | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-17 | docs/FDD.md | Integração | order.controller.ts e order.routes.ts sem mudança: o evento é efeito colateral do service | CODIGO | src/modules/orders/order.controller.ts |
| FDD-AC-01 | docs/FDD.md | Critério de Aceite | Primeiro POST ao cliente em menos de 10 s após o commit do changeStatus com endpoint saudável | TRANSCRICAO | [09:02] Marcos; [09:10] Larissa |
| FDD-AC-02 | docs/FDD.md | Critério de Aceite | Falha injetada no insert da outbox faz o PATCH falhar sem alterar status, history nem estoque | TRANSCRICAO | [09:40] Bruno; [09:41] Diego |
| FDD-AC-03 | docs/FDD.md | Critério de Aceite | Após o 200 do PATCH existe 1 linha PENDING na outbox por webhook ativo interessado no to_status | TRANSCRICAO | [09:06] Diego |
| FDD-AC-04 | docs/FDD.md | Critério de Aceite | Customer sem webhook ativo para o to_status gera 0 linhas na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-05 | docs/FDD.md | Critério de Aceite | Alterar o pedido depois do evento não muda o payload já gravado (snapshot) | TRANSCRICAO | [09:52] Larissa |
| FDD-AC-06 | docs/FDD.md | Critério de Aceite | Payload com exatamente os campos acordados e sem items | TRANSCRICAO | [09:43] Diego |
| FDD-AC-07 | docs/FDD.md | Critério de Aceite | Payload de 65536 bytes é aceito; 65537 bytes falha com WEBHOOK_PAYLOAD_TOO_LARGE e nunca é truncado | TRANSCRICAO | [09:24] Larissa; [09:23] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério de Aceite | Todo POST de saída leva Content-Type, X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id | TRANSCRICAO | [09:44] Diego; [09:44] Sofia |
| FDD-AC-09 | docs/FDD.md | Critério de Aceite | Recalcular HMAC-SHA256 do corpo bruto com a secret reproduz o X-Signature | TRANSCRICAO | [09:22] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério de Aceite | Endpoint que demora mais de 10 s é abortado e registrado como FAILURE por timeout | TRANSCRICAO | [09:42] Diego |
| FDD-AC-11 | docs/FDD.md | Critério de Aceite | Backoff exato +1m, +5m, +30m, +2h e +12h com relógio controlado | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| FDD-AC-12 | docs/FDD.md | Critério de Aceite | Após esgotar as tentativas a outbox fica FAILED e há 1 linha na DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:17] Diego; [09:18] Diego |
| FDD-AC-13 | docs/FDD.md | Critério de Aceite | Replay: ADMIN recebe 202 e a linha volta a PENDING com o mesmo event_id; OPERATOR recebe 403 | TRANSCRICAO | [09:36] Sofia; [09:36] Larissa |
| FDD-AC-14 | docs/FDD.md | Critério de Aceite | Cada replay gera log webhook_dead_letter_replayed com o id do ADMIN em replayedBy | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-16 | docs/FDD.md | Critério de Aceite | Rotação devolve a nova secret uma vez e por 24 h o X-Signature cobre a nova e a anterior | TRANSCRICAO | [09:21] Sofia; [09:22] Sofia |
| FDD-AC-17 | docs/FDD.md | Critério de Aceite | Nenhuma resposta nem log nos testes contém o valor da secret | TRANSCRICAO | [09:22] Diego |
| FDD-AC-18 | docs/FDD.md | Critério de Aceite | URL http retorna 400 VALIDATION_ERROR e https é aceita | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-19 | docs/FDD.md | Critério de Aceite | CRUD funciona para OPERATOR e ADMIN e customerId vem do body ou query, nunca do JWT | TRANSCRICAO | [09:32] Larissa; [09:36] Marcos; [09:37] Sofia |
| FDD-AC-20 | docs/FDD.md | Critério de Aceite | GET deliveries retorna no máximo 100 itens, do mais recente ao mais antigo, com resultado, payload e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-21 | docs/FDD.md | Critério de Aceite | Dois claims concorrentes sobre a mesma linha PENDING resultam em um só vencedor e uma só chamada HTTP | TRANSCRICAO | [09:12] Diego |
| FDD-AC-22 | docs/FDD.md | Critério de Aceite | Falha do worker após o POST reenvia o evento com o mesmo X-Event-Id (at-least-once) | TRANSCRICAO | [09:24] Diego; [09:25] Diego |
| FDD-AC-23 | docs/FDD.md | Critério de Aceite | npm run worker sobe processo próprio e a API segue aceitando mudanças de status com o worker parado | TRANSCRICAO | [09:11] Diego; [09:11] Larissa |
| FDD-AC-24 | docs/FDD.md | Critério de Aceite | A migration cria índices em status e createdAt de webhook_outbox | TRANSCRICAO | [09:08] Diego |
| FDD-AC-25 | docs/FDD.md | Critério de Aceite | Testes existentes de pedidos e auth passam sem alteração e a resposta do PATCH de status não muda | CODIGO | tests/orders.test.ts |
| FDD-AC-26 | docs/FDD.md | Critério de Aceite | Pelo menos 2 dias úteis de revisão de segurança da Sofia sobre HMAC e geração de secret antes do deploy | TRANSCRICAO | [09:46] Sofia; [09:47] Larissa |
| FDD-RISK-01 | docs/FDD.md | Risco | Duplicidade de entrega por at-least-once, mitigada com X-Event-Id estável e documentação no portal | TRANSCRICAO | [09:24] Diego; [09:25] Sofia; [09:26] Marcos |
| FDD-RISK-02 | docs/FDD.md | Risco | Ordering quebrado com múltiplos workers ou por retries com backoff diferente por evento | TRANSCRICAO | [09:12] Diego; [09:13] Diego; [09:13] Larissa |
| FDD-RISK-03 | docs/FDD.md | Risco | Vazamento de secret, limitado por secret única por endpoint e rotação com grace de 24 h | TRANSCRICAO | [09:21] Sofia; [09:22] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Outbox, deliveries e DLQ crescendo sem arquivamento, que está fora do escopo | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Worker parado ou lento acumula eventos PENDING e viola o requisito de menos de 10 s | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Polling concorrente e double-claim por subida acidental de segunda instância do worker | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | Atraso na revisão de segurança da Sofia (2 dias úteis) frente ao prazo de fim de novembro da Atlas | TRANSCRICAO | [09:46] Sofia; [09:45] Marcos; [09:47] Larissa |
| FDD-RISK-08 | docs/FDD.md | Risco | Transação do changeStatus mais pesada com a leitura de webhooks e o createMany da outbox | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-09 | docs/FDD.md | Risco | Cliente lento atrasando outros no batch e ausência de rate limit de saída | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| FDD-RISK-10 | docs/FDD.md | Risco | CRUD aberto a qualquer role e customerId vindo do body, aceito por enquanto | TRANSCRICAO | [09:32] Larissa; [09:37] Sofia |
| FDD-RISK-11 | docs/FDD.md | Risco | Semântica das 5 tentativas divergente (5 retries versus 5 totais) mudaria os números de teste | TRANSCRICAO | [09:17] Larissa; [09:17] Diego |
| FDD-Q-01 | docs/FDD.md | Questão em Aberto | Semântica de 5 tentativas: 1 original mais 5 retries (14h36m) a confirmar na revisão do RFC | TRANSCRICAO | [09:17] Diego; [09:17] Larissa |
| FDD-Q-02 | docs/FDD.md | Questão em Aberto | Formato do X-Signature, timestamp no material assinado e armazenamento da secret ficaram sem decisão | TRANSCRICAO | [09:46] Sofia; [09:49] Sofia |
| FDD-Q-03 | docs/FDD.md | Questão em Aberto | Papel de WEBHOOK_INVALID_URL frente ao VALIDATION_ERROR que o Zod já produz para http | TRANSCRICAO | [09:23] Sofia; [09:28] Bruno |
| FDD-Q-04 | docs/FDD.md | Questão em Aberto | event_id por par pedido/webhook (FDD) versus um único event_id por mudança de status | TRANSCRICAO | [09:25] Diego |
| FDD-Q-05 | docs/FDD.md | Questão em Aberto | Evento na criação do pedido: a reunião só trata mudança de status e create não passa por changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-Q-07 | docs/FDD.md | Questão em Aberto | Como emitir WEBHOOK_NOT_FOUND: NotFoundError hoje fixa o código NOT_FOUND no construtor | CODIGO | src/shared/errors/http-errors.ts |
| FDD-Q-08 | docs/FDD.md | Questão em Aberto | Rate limiting de saída por cliente: observar e decidir depois | TRANSCRICAO | [09:38] Diego; [09:39] Larissa |
| FDD-Q-10 | docs/FDD.md | Questão em Aberto | Vínculo de requestId com o evento exigiria mudar a assinatura de publishWebhookEvent (métricas e tracing sem origem) | TRANSCRICAO | [09:41] Bruno |
| FDD-Q-11 | docs/FDD.md | Questão em Aberto | Arquivamento e retenção da outbox (cerca de 30 dias) | TRANSCRICAO | [09:08] Diego |
| FDD-Q-12 | docs/FDD.md | Questão em Aberto | Email ou alerta ao cliente quando o webhook falha: próxima fase, após medir impacto | TRANSCRICAO | [09:37] Larissa |
| FDD-Q-13 | docs/FDD.md | Questão em Aberto | Estratégia de escala do worker e ordering com múltiplos workers (particionar por order_id ou lock pessimista) | TRANSCRICAO | [09:13] Diego |
| FDD-Q-14 | docs/FDD.md | Questão em Aberto | Endurecer a permissão do CRUD de webhooks no futuro | TRANSCRICAO | [09:37] Sofia |
| FDD-Q-15 | docs/FDD.md | Questão em Aberto | Nome do arquivo do processor: webhook.worker.ts ou webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| FDD-Q-17 | docs/FDD.md | Hipótese | Despacho sequencial (default) ou concorrente entre pedidos no batch do worker; ordering depende de worker único em ordem de created_at | TRANSCRICAO | [09:12] Diego; [09:13] Larissa |
| FDD-Q-18 | docs/FDD.md | Questão em Aberto | Payload acima de 64KB dentro da transação aborta a mudança de status? FDD propõe 422 com rollback (RFC-OPEN-12) | TRANSCRICAO | [09:24] Larissa; [09:40] Bruno |
| FDD-Q-19 | docs/FDD.md | Questão em Aberto | Relação entre X-Event-Id, event_id do corpo e id da linha da outbox (RFC-OPEN-13); Diego só disse UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| FDD-Q-20 | docs/FDD.md | Hipótese | Regra de onDelete (Restrict nas tabelas de evidência, Cascade de Customer para Webhook), alinhada ao schema atual onde Order referencia Customer sem cascade | CODIGO | prisma/schema.prisma |

## Itens sem origem identificável (propostas dos documentos, a decidir na revisão)

Estes itens aparecem nos documentos rotulados como **Hipótese** ou proposta e **não** têm fala da reunião nem arquivo de código que os sustente integralmente. Não são requisitos aprovados: devem ser aceitos, ajustados ou removidos na revisão do RFC.

| ID | Documento | Observação |
| --- | --- | --- |
| PRD-OBJ-08 | `docs/PRD.md` | Meta de reduzir em 80% as chamadas de polling a `GET /orders`; o problema nasce em [09:00] Marcos, mas o número é proposta do PRD sem fala de origem. |
| PRD-OBJ-09 | `docs/PRD.md` | Meta qualitativa "Atlas permanece cliente"; o risco de churn vem de [09:00] Marcos, a meta é proposta do PRD. |
| PRD-NFR-09 | `docs/PRD.md` | Meta de disponibilidade em percentual; não foi discutida na reunião nem existe no código. |
| PRD-TEST-09 | `docs/PRD.md` | Validação com a Atlas em ambiente de teste; proposta do PRD, a reunião só trata de confirmar o prazo ([09:47] Marcos). |
| FDD-FLX-10 | `docs/FDD.md` | Recuperação de evento PROCESSING órfão por lease de 60 s; proposta do FDD. |
| FDD-RES-10 | `docs/FDD.md` | Resiliência: repete a recuperação por lease de 60 s (FDD-FLX-10); proposta do FDD. |
| FDD-RES-13 | `docs/FDD.md` | Não seguir redirects, truncar resposta armazenada em 4 KB e mitigar SSRF; proposta do FDD. |
| FDD-DATA-08 | `docs/FDD.md` | Armazenamento da secret cifrada com AES-256-GCM; proposta do FDD (a reunião só pede que a secret exista, [09:21] Sofia, e que Sofia revise a geração, [09:46] Sofia). |
| FDD-OBS-04 | `docs/FDD.md` | Métrica de contagem da outbox por status; proposta do FDD, a reunião não discutiu métricas. |
| FDD-OBS-07 | `docs/FDD.md` | Métrica de tamanho da DLQ; proposta do FDD. |
| FDD-OBS-08 | `docs/FDD.md` | Tracing por correlação de ids e `attemptId`; proposta do FDD, a reunião não discutiu tracing. |
| FDD-OBS-10 | `docs/FDD.md` | Nomes de spans no estilo OpenTelemetry; proposta do FDD. |
| FDD-Q-09 | `docs/FDD.md` | Questão sobre SSRF nas URLs cadastradas; levantada pela análise, não discutida na reunião. |
| FDD-Q-16 | `docs/FDD.md` | Lista consolidada de hipóteses do FDD (batch de 10, lease de 60 s, 202 no replay e outras). |
| FDD-ERR-07 | `docs/FDD.md` | Código WEBHOOK_DEAD_LETTER_NOT_FOUND para replay de id inexistente; o conceito de replay vem de [09:18] Diego, o código é proposta do FDD. |
| FDD-ERR-08 | `docs/FDD.md` | Código WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409) para replay repetido; proposta do FDD. |
| FDD-ERR-09 | `docs/FDD.md` | Código WEBHOOK_INACTIVE (409) para replay ou rotação em webhook removido ou inativo; proposta do FDD. |
| FDD-AC-15 | `docs/FDD.md` | Critério de aceite de replay repetido e de webhook inativo; deriva de FDD-ERR-08 e FDD-ERR-09, que são propostas do FDD. |
| FDD-DATA-07 | `docs/FDD.md` | Sem FK de webhook_outbox.orderId para orders e regra de onDelete das tabelas de evidência; proposta do FDD (o snapshot independente do pedido vem de [09:52] Larissa, mas a regra de FK não). |
| FDD-DATA-09 | `docs/FDD.md` | Soft delete de webhooks, preservando histórico; proposta do FDD ([09:33] Bruno só fala em DELETE para remover). |
| FDD-DEP-08 | `docs/FDD.md` | Uso do módulo node:crypto para HMAC e geração de secret; proposta do FDD (a reunião só fixa HMAC-SHA256, [09:20] Sofia). |
| FDD-OBS-03 | `docs/FDD.md` | Log estruturado com eventId, webhookId e orderId, sem secret; proposta do FDD, a reunião só fixa o reuso do Pino ([09:29] Bruno). |
| FDD-Q-06 | `docs/FDD.md` | Questão sobre rotação de secret em webhook inativo; levantada pela análise, não discutida na reunião. |

## Cobertura

- Linhas na tabela principal: 359. Itens sem origem listados à parte: 23.
- Linhas com Fonte `TRANSCRICAO`: 312 (86.9%). Linhas com Fonte `CODIGO`: 47.
- Todo ID em negrito dos documentos (e os rótulos `OT-NN` e `Q-NN` do FDD) aparece na tabela principal ou na seção de itens sem origem. A conferência foi feita por script, descrita no README, seção "Iterações e ajustes".
