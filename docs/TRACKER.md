# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado nos documentos desta entrega e sua origem: a transcrição da reunião ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) ou o código-fonte da aplicação.

**Como ler:**

- **Fonte `TRANSCRICAO`** → coluna Localização traz o timestamp e o nome do falante, no formato `[hh:mm] Nome`.
- **Fonte `CODIGO`** → coluna Localização traz o caminho real do arquivo no repositório.

Nenhum requisito, decisão ou restrição foi registrado sem origem identificável. Itens sem rastreabilidade foram removidos durante a revisão (ver [README](../README.md)).

---

## PRD — `docs/PRD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em `GET /orders`, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição de negócio | Atlas pode migrar para o concorrente se não entregarmos até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | Sistema atual não tem notificação externa, eventos, filas ou webhooks | CODIGO | src/routes/index.ts |
| PRD-CTX-05 | docs/PRD.md | Contexto | OMS tem máquina de estados, estoque transacional e auditoria de status | CODIGO | src/modules/orders/order.status.ts |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência p95 de entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Restrição de prazo | Entrega até o fim de novembro, estimada em três sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-PER-01 | docs/PRD.md | Público-alvo | Administrador com role ADMIN reprocessa eventos mortos | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | docs/PRD.md | Cenário de uso | Cliente assina apenas SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário de uso | Cliente em manutenção planejada de duas horas | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário de uso | Desenvolvedor consulta histórico de entregas para depurar | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-04 | docs/PRD.md | Cenário de uso | Cliente rotaciona secret após suspeita de vazamento em log | TRANSCRICAO | [09:22] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint de webhook; secret gerada por nós e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Filtro de status assinados por endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | PATCH para editar endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | DELETE para remover endpoint | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | GET para listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico de entregas com sucesso/falha, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Entrega de evento a cada mudança de status assinada | TRANSCRICAO | [09:06] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-FR-09b | docs/PRD.md | Requisito Funcional | X-Webhook-Id acrescentado para cliente com múltiplos cadastros | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Retry automático, 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ via endpoint admin | TRANSCRICAO | [09:35] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Recusar URL que não seja https | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | CRUD autenticado por qualquer role; replay exige ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre evento e mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Indisponibilidade do cliente não pode travar mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com dedup pelo cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, sem secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, com erro em vez de truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordering apenas por order_id e enquanto single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Prefixo WEBHOOK_ em todos os códigos de erro | TRANSCRICAO | [09:29] Larissa |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | Notificação por e-mail ao cliente adiada para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual descartado nesta fase | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída fora do escopo, observar depois | TRANSCRICAO | [09:39] Diego |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Webhooks inbound fora de escopo, fluxo é só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Arquivamento de linhas entregues fora desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança bloqueante, 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal de desenvolvedor a cargo do PM | TRANSCRICAO | [09:26] Marcos |
| PRD-RI-01 | docs/PRD.md | Risco | Regressão no fluxo crítico de mudança de status | CODIGO | src/modules/orders/order.service.ts |
| PRD-RI-03 | docs/PRD.md | Risco | Cliente não deduplicar e processar evento duas vezes | TRANSCRICAO | [09:25] Sofia |

## RFC — `docs/RFC.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Metadado | Revisores são os cinco participantes da reunião | TRANSCRICAO | [09:00] Larissa |
| RFC-PROP-01 | docs/RFC.md | Decisão | Padrão Outbox no MySQL como base da solução | TRANSCRICAO | [09:08] Larissa |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado com polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry com backoff e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | Entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | docs/RFC.md | Decisão | Filtragem de status na inserção da outbox, não no envio | TRANSCRICAO | [09:34] Bruno |
| RFC-PROP-07 | docs/RFC.md | Decisão | Gancho da feature é o método changeStatus dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo HTTP síncrono descartado: travaria a transação de status | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado: infra nova e overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco descartada: MySQL não tem LISTEN/NOTIFY | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | 3 tentativas descartadas: janela curta demais para indisponibilidade real | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Trade-off | Secret global descartada: vazamento único comprometeria todos | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Trade-off | Exactly-once descartado: exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Trade-off | Payload renderizado no envio descartado: refletiria estado errado | TRANSCRICAO | [09:52] Larissa |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Notificação por e-mail adiada para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Escala horizontal do worker e ordering: problema do futuro | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Política de arquivamento da outbox não definida | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | Endurecer permissão do CRUD de configuração no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Novos models e migração no schema Prisma | CODIGO | prisma/schema.prisma |
| RFC-IMP-02 | docs/RFC.md | Impacto | Registro do novo router na composição da API | CODIGO | src/routes/index.ts |
| RFC-IMP-03 | docs/RFC.md | Impacto | Novas classes de erro exportadas pelo barrel de erros | CODIGO | src/shared/errors/index.ts |
| RFC-RISK-01 | docs/RFC.md | Risco | Regressão em changeStatus mitigada por função pura recebendo tx | TRANSCRICAO | [09:41] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Falha no HMAC mitigada por revisão de segurança dedicada | TRANSCRICAO | [09:46] Sofia |

## FDD — `docs/FDD.md`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-CTX-01 | docs/FDD.md | Contexto | changeStatus roda em transação com update, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| FDD-CTX-02 | docs/FDD.md | Contexto | Projeto tem um único entry-point hoje | CODIGO | src/server.ts |
| FDD-MOD-01 | docs/FDD.md | Decisão | PK uuid em todos os models novos, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-MOD-02 | docs/FDD.md | Restrição | Padrão de PK uuid char(36) já vigente no schema | CODIGO | prisma/schema.prisma |
| FDD-MOD-03 | docs/FDD.md | Decisão | Índices em status e createdAt na outbox para o worker | TRANSCRICAO | [09:08] Diego |
| FDD-MOD-04 | docs/FDD.md | Decisão | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-MOD-05 | docs/FDD.md | Decisão | Endpoint armazena url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-MOD-06 | docs/FDD.md | Decisão | Histórico de entregas guarda payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-FLOW-01 | docs/FDD.md | Decisão | publishWebhookEvent(tx, order, fromStatus, toStatus) recebe o tx | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Restrição | Falha na inserção da outbox faz rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Decisão | Filtragem por status assinados acontece na inserção | TRANSCRICAO | [09:34] Diego |
| FDD-FLOW-04 | docs/FDD.md | Decisão | Payload é snapshot renderizado no momento da inserção | TRANSCRICAO | [09:52] Diego |
| FDD-FLOW-05 | docs/FDD.md | Decisão | Payload sem items; cliente busca detalhe em GET /orders/:id | TRANSCRICAO | [09:43] Diego |
| FDD-FLOW-06 | docs/FDD.md | Decisão | Worker faz polling de 2s lendo pendentes mais antigos em batch | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-07 | docs/FDD.md | Decisão | Worker usa PrismaClient próprio, mesmo banco e DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| FDD-FLOW-08 | docs/FDD.md | Restrição | Worker roda como processo separado da API | TRANSCRICAO | [09:11] Diego |
| FDD-FLOW-09 | docs/FDD.md | Decisão | Backoff fixo 1m/5m/30m/2h/12h em 5 tentativas | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-10 | docs/FDD.md | Decisão | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-11 | docs/FDD.md | Restrição | Replay exige role ADMIN e registra quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-HDR-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-HDR-02 | docs/FDD.md | Contrato | Header X-Webhook-Id com o id do endpoint | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks cadastra endpoint e devolve a secret gerada | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks lista endpoints de um customer, paginado | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id edita endpoint | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id remove endpoint | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/secret/rotate rotaciona a secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries devolve histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay reprocessa evento morto | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Restrição | customer_id vem do body ou do path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-09 | docs/FDD.md | Restrição | Paginação de listagens reusa o helper existente | CODIGO | src/shared/http/response.ts |

| FDD-ERRO-01 | docs/FDD.md | Restrição | Prefixo WEBHOOK_ em todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Decisão | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Restrição | Novas classes estendem AppError seguindo o padrão existente | CODIGO | src/shared/errors/app-error.ts |
| FDD-ERRO-04 | docs/FDD.md | Restrição | Padrão de código espelha INSUFFICIENT_STOCK e INVALID_STATUS_TRANSITION | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-05 | docs/FDD.md | Restrição | errorMiddleware não precisa de alteração para tratar os novos erros | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERRO-06 | docs/FDD.md | Decisão | WEBHOOK_PAYLOAD_TOO_LARGE quando o corpo passa de 64KB | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-07 | docs/FDD.md | Decisão | WEBHOOK_DELIVERY_TIMEOUT após 10 segundos sem resposta | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Timeout de 10 segundos por tentativa | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Teto de 64KB por payload, com erro em vez de truncar | TRANSCRICAO | [09:24] Diego |
| FDD-RESIL-03 | docs/FDD.md | Trade-off | Sem garantia de ordering global, apenas por order_id | TRANSCRICAO | [09:12] Diego |
| FDD-RESIL-04 | docs/FDD.md | Restrição | Graceful shutdown no padrão de SIGINT/SIGTERM já existente | CODIGO | src/server.ts |
| FDD-OBS-01 | docs/FDD.md | Decisão | Logger Pino reaproveitado sem introduzir nada novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Restrição | redactPaths atual não cobre a secret de webhook, precisa de ajuste | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Decisão | Log de auditoria do replay com o usuário que executou | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-04 | docs/FDD.md | Decisão | X-Event-Id como chave de correlação ponta a ponta | TRANSCRICAO | [09:25] Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Nenhuma dependência nova de infraestrutura | TRANSCRICAO | [09:07] Diego |
| FDD-DEP-02 | docs/FDD.md | Dependência | Node 20 e libs já declaradas cobrem crypto e fetch nativos | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Dependência | Novas variáveis de configuração seguem o envSchema existente | CODIGO | src/config/env.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Chamada de publishWebhookEvent após orderStatusHistory.create | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reuso das classes de erro e do barrel de exportação | CODIGO | src/shared/errors/index.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Reuso de authenticate e requireRole no router do módulo | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Enum OrderStatus e máquina de estados como fonte do filtro | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Novo src/worker.ts espelha a estrutura de bootstrap do servidor | CODIGO | src/server.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Script npm run worker no package.json | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-07 | docs/FDD.md | Integração | Módulo em src/modules/webhooks seguindo o padrão dos demais | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-08 | docs/FDD.md | Integração | Lógica de processamento em webhook.worker.ts dentro do módulo | TRANSCRICAO | [09:28] Bruno |
| FDD-INT-09 | docs/FDD.md | Integração | Router montado junto aos demais módulos da API | CODIGO | src/routes/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Dependências instanciadas em buildControllers | CODIGO | src/app.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Validação de entrada pelo middleware Zod existente | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Schemas seguem o padrão de order.schemas.ts | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Router aplica authenticate no topo, como o de pedidos | CODIGO | src/modules/orders/order.routes.ts |
| FDD-TEST-01 | docs/FDD.md | Restrição | Testes de integração seguem o formato dos testes de pedidos | CODIGO | tests/orders.test.ts |
| FDD-TEST-02 | docs/FDD.md | Restrição | Factories novas no helper de testes existente | CODIGO | tests/helpers/factories.ts |
| FDD-TEST-03 | docs/FDD.md | Dependência | Revisão de segurança com 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |

## ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, inserção na mesma transação | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT1 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Trade-off | Síncrono descartado por acoplar disponibilidade do cliente | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Trade-off | Redis Streams descartado por infra nova e dupla escrita | TRANSCRICAO | [09:07] Diego |
| ADR-001-CONS | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Consequência | Arquivamento da outbox fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Trigger de banco descartada: MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-CONS | docs/adrs/ADR-002-worker-separado-em-polling.md | Consequência | Ordering apenas por order_id enquanto single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | 3 tentativas descartadas por janela curta demais | TRANSCRICAO | [09:16] Diego |
| ADR-003-CONS | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Consequência | Janela de até cerca de 15h aceita pelo PM | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-CTX | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Contexto | Precedente de cliente que vazou secret em log | TRANSCRICAO | [09:22] Diego |
| ADR-004-ALT | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret global descartada: vazamento comprometeria todos | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONS | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Consequência | TLS obrigatório validado no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado por exigir coordenação bilateral | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Consequência | Responsabilidade transferida ao cliente, ponto levantado na revisão | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CTX | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto | Padrão de módulo controller/service/repository/routes/schemas | CODIGO | src/modules/orders/order.controller.ts |
| ADR-006-ALT | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Trade-off | Injetar repository inteiro descartado em favor de função com tx | TRANSCRICAO | [09:41] Diego |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-CTX | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Contexto | Dúvida sobre o que a outbox guarda, levantada no fim da reunião | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-do-payload-na-outbox.md | Trade-off | Snapshot com items descartado para manter o payload enxuto | TRANSCRICAO | [09:44] Bruno |

---

## Cobertura

Todos os caminhos de arquivo citados na coluna Localização existem no repositório base, e todos os timestamps citados existem na transcrição.

