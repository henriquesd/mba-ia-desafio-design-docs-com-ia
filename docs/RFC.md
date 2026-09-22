# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-09-22 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| **Origem** | Reunião técnica de quinta-feira, 09:00–09:53 ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) |

---

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B sobre mudanças de status de pedidos via **webhooks outbound**, construídos sobre o **padrão Outbox no MySQL existente**. A inserção do evento acontece na mesma transação de `OrderService.changeStatus`, e um **worker em processo separado**, com polling de 2 segundos, faz a entrega HTTP.

A entrega é **at-least-once**, autenticada por **HMAC-SHA256** com secret por endpoint, com **5 tentativas** em backoff exponencial (1m/5m/30m/2h/12h) e **DLQ em tabela separada** com replay manual por endpoint administrativo.

Nenhuma infraestrutura nova é introduzida: mesmo MySQL, mesmo Prisma, mesmos padrões de módulo, erro e logging já em uso no repositório. Estimativa: **três sprints**, incluindo a revisão de segurança.

## 2. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente notificação em tempo real de mudanças de status dos pedidos deles ([09:00] Marcos). Hoje eles resolvem isso batendo repetidamente em `GET /orders`, o que torna a integração lenta e cara do lado deles. A Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre.

"Tempo real", para o cliente, significa **abaixo de 10 segundos** ([09:02] Marcos). O escopo é estritamente **outbound**: eventos saem da nossa plataforma para eles, nunca o contrário ([09:02] Marcos e Sofia).

O sistema atual não tem nenhum mecanismo de notificação externa, eventos ou filas. O gancho natural é `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que já roda dentro de um `prisma.$transaction` atualizando `orders`, `order_status_history` e o estoque dos produtos.

## 3. Proposta técnica

### 3.1 Visão geral do fluxo

1. `PATCH /api/v1/orders/:id/status` chega no `OrderController`.
2. `OrderService.changeStatus` abre a transação: valida a transição via `order.status.ts`, atualiza `orders`, insere em `order_status_history`, ajusta estoque e — nova etapa — chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que insere as linhas em `webhook_outbox`.
3. Commit. A partir daqui o evento está garantido.
4. O worker (`src/worker.ts`, processo separado) faz polling a cada 2 segundos buscando eventos elegíveis, monta os headers assinados e faz `POST` no endpoint do cliente com timeout de 10 segundos.
5. Resposta 2xx marca `DELIVERED`. Qualquer outra coisa agenda a próxima tentativa segundo o backoff.
6. Esgotadas as 5 tentativas, a linha vai para `webhook_dead_letter`, de onde só sai por replay manual de um usuário `ADMIN`.

### 3.2 Pilares da proposta

**Outbox transacional.** O evento é inserido na mesma transação da mudança de status. Se a transação commitou, o evento existe; se deu rollback, o evento não existe. Isso elimina a possibilidade de notificação fantasma ou de mudança de status sem notificação ([09:06] Diego). Detalhe em [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md).

**Worker separado em polling.** Processo próprio (`src/worker.ts` _(novo)_, `npm run worker`), fora do ciclo de vida da API, lendo pendentes a cada 2 segundos. Latência de pior caso de 2 segundos, dentro dos 10 segundos acordados. Ver [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).

**Resiliência.** Timeout de 10 segundos por chamada, 5 tentativas com backoff 1m/5m/30m/2h/12h, depois DLQ em tabela dedicada com replay manual restrito a role `ADMIN`. Ver [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md).

**Segurança.** HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com grace period de 24 horas, HTTPS obrigatório. Ver [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md).

**Semântica de entrega.** At-least-once, com `X-Event-Id` (UUID) para o cliente deduplicar. Ver [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).

**Superfície de API.** CRUD de configuração de webhook (criar, listar, editar, remover), rotação de secret, consulta de histórico de entregas e replay administrativo de DLQ. Todos autenticados; apenas o replay exige `ADMIN` ([09:36] Sofia e Larissa).

**Reuso.** Módulo `src/modules/webhooks/` no mesmo formato dos demais, erros estendendo `AppError` com prefixo `WEBHOOK_`, Pino, `errorMiddleware` e schemas Zod sem alteração. Ver [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md).

**Snapshot do payload.** O JSON é renderizado e gravado no momento da inserção, não no envio, para que o evento reflita o estado do pedido quando o status mudou. Ver [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md).

### 3.3 Filtragem por assinatura de eventos

Cada endpoint declara quais status quer receber. A filtragem acontece **na inserção**: se nenhum webhook ativo do customer escuta aquele status, a linha nem é criada na outbox ([09:34] Bruno e Diego).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que motivou o descarte | Quem levantou |
|---|---|---|---|
| 1 | **Disparo HTTP síncrono** dentro de `changeStatus` | Acopla latência e disponibilidade do cliente à transação mais pesada do sistema; cliente fora do ar forçaria rollback de uma mudança de status válida | [09:04] Bruno |
| 2 | **Redis Streams** como fila de eventos | Exige subir e operar infraestrutura nova para um time pequeno, e reintroduz dupla escrita não atômica entre MySQL e Redis. Classificado como overengineering | [09:07] Diego |
| 3 | **Trigger de banco** para reagir à mudança | MySQL não tem `LISTEN/NOTIFY`; trigger só executa SQL e não acorda processo externo sem gambiarra. Polling de 2s já atende o SLO | [09:09] Diego |
| 4 | **3 tentativas de retry** em vez de 5 | Janela curta demais: mataria eventos de cliente em manutenção planejada de poucas horas | [09:16] Diego |
| 5 | **Secret global** da plataforma | Vazamento único comprometeria todos os clientes: "se vaza uma, vaza tudo" | [09:21] Sofia |
| 6 | **Exactly-once** com confirmação bilateral | Exige coordenação dos dois lados e muda o contrato para três clientes que só querem receber um POST. At-least-once com `event_id` cobre 99% dos casos | [09:25] Diego |
| 7 | **Payload renderizado no envio** (só referência na outbox) | Evento reentregue horas depois descreveria o estado atual do pedido, não o do momento do evento | [09:52] Larissa |

## 5. Questões em aberto

1. **Rate limiting de saída.** Se um cliente tiver 50 pedidos mudando de status no mesmo minuto, disparamos 50 chamadas contra ele. Ficou fora do escopo desta fase, com orientação explícita de observar em produção e decidir depois.
   > [09:39] Diego: Eu acho que não [faz parte do escopo]. A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto.

2. **Notificação ao cliente sobre webhooks com falha.** Marcos pediu alerta por e-mail quando um endpoint falha repetidamente; foi adiado para a fase seguinte, condicionado à medição de impacto.
   > [09:37] Larissa: Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto.

3. **Escala horizontal do worker e garantia de ordering.** Com múltiplos workers, a ordenação por `order_id` se perde. As direções levantadas foram particionamento por `order_id` ou lock pessimista, mas nenhuma foi decidida.
   > [09:13] Diego: Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora.

4. **Política de arquivamento da outbox.** A intenção é arquivar linhas entregues depois de cerca de 30 dias, mas mecanismo, frequência e destino não foram definidos.
   > [09:08] Diego: Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature.

5. **Endurecimento de permissão no CRUD de configuração.** Hoje qualquer role autenticada pode gerenciar webhooks; a Sofia sinalizou revisão futura.
   > [09:37] Sofia: Por enquanto sim. Mais pra frente a gente pode endurecer.

## 6. Impacto e riscos

### 6.1 Impacto no sistema existente

| Área | Impacto |
|---|---|
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, ...)` dentro da transação. É a alteração mais sensível da feature: se a inserção falhar, a mudança de status faz rollback ([09:40] Bruno) |
| `prisma/schema.prisma` | Três novos models (configuração, outbox, dead letter) e a migração correspondente |
| `src/routes/index.ts` e `src/app.ts` | Registro do novo router e das novas dependências em `buildControllers` |
| `src/shared/errors/index.ts` | Exportação das novas classes de erro com prefixo `WEBHOOK_` |
| `package.json` | Novo script `worker` |
| Operação | Novo processo a monitorar, além da API |

### 6.2 Riscos principais

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Regressão em `changeStatus`, caminho crítico do OMS | Média | Alto | Função pura recebendo `tx`, sem injetar repository ([09:41] Diego); testes ponta a ponta no estilo de `tests/orders.test.ts` |
| Falha na implementação do HMAC ou na geração de secret | Média | Alto | Revisão de segurança dedicada, com **2 dias úteis reservados antes do deploy** ([09:46] Sofia) |
| Cliente não deduplicar por `X-Event-Id` e processar evento duas vezes | Alta | Médio | Documentação destacada no portal de desenvolvedor ([09:26] Marcos) |
| Bombardeio de um cliente por rajada de mudanças de status | Média | Médio | Observação em produção; rate limiting é questão em aberto (§5.1) |
| Crescimento não controlado da `webhook_outbox` | Alta | Médio | Índices em `status` e `createdAt` ([09:08] Diego); arquivamento fora de escopo, monitorar tamanho |
| Prazo: Atlas espera entrega para fim de novembro | Média | Alto | Três sprints estimados com fatiamento acordado ([09:46] Larissa) |

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-padrao-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2 segundos](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Entrega at-least-once com X-Event-Id](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Snapshot do payload na outbox](adrs/ADR-007-snapshot-do-payload-na-outbox.md)

O detalhamento de implementação — contratos HTTP, matriz de erros, modelagem de dados, observabilidade e integração com o código existente — está no [FDD](FDD.md).

