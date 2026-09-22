# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Status** | Pronto para implementação |
| **Data** | 2026-09-22 |
| **Autores** | Bruno (Eng. Pleno / Pedidos), Diego (Eng. Sênior / Plataforma), Larissa (Tech Lead) |
| **Revisão de segurança** | Sofia (Eng. Segurança) — 2 dias úteis reservados antes do deploy ([09:46]) |
| **Documentos relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Contexto e motivação técnica

O OMS hoje muda o status de um pedido dentro de uma única transação em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`): valida a transição contra a máquina de estados de `src/modules/orders/order.status.ts`, atualiza `orders`, insere em `order_status_history` e debita ou repõe `stockQuantity` dos produtos.

Não existe hoje nenhum ponto de extensão para notificação externa: nenhum evento, nenhuma fila, nenhum processo de background. O projeto tem um único entry-point (`src/server.ts`).

Três clientes B2B precisam saber dessas mudanças em menos de 10 segundos ([09:02] Marcos) sem ficar batendo em `GET /orders`. O desafio técnico central é acoplar a notificação à mudança de status **sem** acoplar a disponibilidade do cliente à transação mais pesada do sistema.

A solução aprovada no [RFC](RFC.md) é o padrão Outbox ([ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md)) com worker separado ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

| # | Objetivo |
|---|---|
| OT-1 | Registrar o evento de mudança de status atomicamente com a própria mudança, sem transação distribuída |
| OT-2 | Entregar o evento ao endpoint do cliente em menos de 10 segundos no caminho feliz, com latência de polling de 2 segundos |
| OT-3 | Nunca bloquear nem falhar `changeStatus` por indisponibilidade do cliente |
| OT-4 | Garantir autenticidade e integridade do payload para o receptor via HMAC-SHA256 |
| OT-5 | Sobreviver a indisponibilidades de cliente de até cerca de 15 horas sem intervenção humana |
| OT-6 | Não perder evento: toda falha definitiva termina em DLQ inspecionável e reprocessável |
| OT-7 | Não introduzir nenhuma dependência de infraestrutura nova |
| OT-8 | Seguir integralmente os padrões de módulo, erro, validação e log já existentes no repositório |

## 3. Escopo

### 3.1 Incluído

- Model e migração de `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
- Módulo `src/modules/webhooks/` (controller, service, repository, routes, schemas).
- CRUD de configuração de endpoints de webhook, com filtro por status assinados.
- Geração e rotação de secret com grace period de 24 horas.
- Publicação do evento na outbox dentro da transação de `changeStatus`.
- Worker (`src/worker.ts` + `src/modules/webhooks/webhook.worker.ts`) com polling, envio HTTP assinado, retry e DLQ.
- Endpoint de histórico de entregas.
- Endpoint administrativo de replay de DLQ, restrito a role `ADMIN`.
- Métricas, logs estruturados e tracing do fluxo de entrega.

### 3.2 Fora de escopo (decidido na reunião)

| Item | Motivo | Origem |
|---|---|---|
| Webhooks inbound (cliente enviando para nós) | Escopo é estritamente outbound | [09:02] Marcos |
| Notificação por e-mail ao cliente quando o webhook falha | Adiado para a próxima fase, depois de medir impacto | [09:37] Larissa |
| Dashboard visual para o cliente | Projeto separado do time de frontend; agora só endpoints | [09:40] Larissa |
| Rate limiting de saída por cliente | Observar em produção e decidir depois | [09:39] Diego e Larissa |
| Arquivamento de linhas entregues da outbox | Explicitamente fora desta feature | [09:08] Diego |
| Escala horizontal do worker (particionamento / lock) | Problema do futuro | [09:13] Diego |
| Eventos além de mudança de status de pedido | Nada além de `order.status_changed` foi pedido | [09:43] Diego |

## 4. Modelagem de dados

Todos os models seguem o padrão do `prisma/schema.prisma`: PK `String @id @default(uuid()) @db.Char(36)` ([09:51] Larissa).

### `webhook_endpoints`

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid | PK |
| `customerId` | char(36) | FK para `customers` |
| `url` | varchar(2048) | **Obrigatoriamente `https`** ([09:23] Sofia) |
| `secret` | varchar(255) | Secret ativa, armazenada de forma recuperável (precisa assinar a cada envio) |
| `previousSecret` | varchar(255) nullable | Secret anterior durante o grace period |
| `previousSecretExpiresAt` | datetime nullable | `rotatedAt + 24h` ([09:21] Sofia) |
| `subscribedStatuses` | json | Lista de `OrderStatus` que este endpoint quer receber ([09:33] Marcos) |
| `active` | boolean | Endpoint pode ser desativado sem ser removido |
| `createdAt` / `updatedAt` | datetime | Padrão do schema |

Índices: `customerId`, `active`.

### `webhook_outbox`

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid | Também é o `event_id` enviado em `X-Event-Id` ([09:25] Diego) |
| `webhookEndpointId` | char(36) | FK; uma linha por endpoint destinatário |
| `orderId` | char(36) | FK para `orders`, para consulta e correlação |
| `eventType` | varchar(64) | `order.status_changed` |
| `payload` | json | **Snapshot renderizado na inserção** ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md)) |
| `status` | enum | `PENDING`, `PROCESSING`, `DELIVERED`, `FAILED` |
| `attempts` | int | Contador de tentativas já realizadas (0 a 5) |
| `nextAttemptAt` | datetime | Quando a linha volta a ser elegível |
| `lastError` | varchar(500) nullable | Último motivo de falha |
| `createdAt` / `updatedAt` | datetime | `createdAt` define a ordem de processamento |

Índices: `(status, nextAttemptAt)` para a query do worker, `createdAt` para ordering, `orderId` ([09:08] Diego).

### `webhook_deliveries`

Histórico de tentativas, base do endpoint de consulta pedido em [09:34] por Marcos.

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid | PK |
| `outboxEventId` | char(36) | Evento que originou a tentativa |
| `webhookEndpointId` | char(36) | FK |
| `attempt` | int | Número da tentativa (1 a 5) |
| `requestPayload` | json | O que foi enviado |
| `responseStatus` | int nullable | Null em caso de timeout ou erro de conexão |
| `responseBody` | text nullable | Truncado para armazenamento |
| `durationMs` | int | Tempo de resposta ([09:34] Marcos) |
| `success` | boolean | 2xx dentro do timeout |
| `createdAt` | datetime | — |

Índices: `webhookEndpointId`, `outboxEventId`, `createdAt`.

### `webhook_dead_letter`

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid | PK |
| `originalEventId` | char(36) | `id` da linha original da outbox |
| `webhookEndpointId` | char(36) | FK |
| `payload` | json | Preservado para replay ([09:18] Diego) |
| `failureReason` | varchar(500) | Motivo da última falha |
| `attempts` | int | Sempre 5 no fluxo normal |
| `movedAt` | datetime | Timestamp da morte do evento |
| `replayedAt` | datetime nullable | Preenchido quando um ADMIN reprocessa |
| `replayedById` | char(36) nullable | Usuário que executou o replay, para auditoria ([09:36] Sofia) |

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro da transação)

Ponto de entrada: `PATCH /api/v1/orders/:id/status` → `OrderController.changeStatus` → `OrderService.changeStatus`.

Sequência dentro do `prisma.$transaction` existente, após a escrita em `order_status_history`:

1. `publishWebhookEvent(tx, order, fromStatus, toStatus)` é chamada com o `Prisma.TransactionClient` corrente ([09:41] Bruno e Diego).
2. A função consulta `webhook_endpoints` por `customerId = order.customerId`, `active = true` e `toStatus ∈ subscribedStatuses`.
3. **Se nenhum endpoint casar, nada é inserido.** A filtragem acontece na inserção, não no envio, para economizar linha na tabela ([09:34] Bruno e Diego).
4. Para cada endpoint elegível, monta o snapshot do payload (§5.2) e insere uma linha em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`.
5. Qualquer erro na inserção propaga e **aborta a transação inteira**: a mudança de status faz rollback.

> [09:40] Bruno: A gente vai inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair.

Assinatura proposta:

```ts
// src/modules/webhooks/webhook.publisher.ts  (arquivo novo)
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>
```

Não recebe o repository inteiro, apenas o `tx`, para não acoplar o `OrderService` ao módulo de webhooks ([09:41] Diego).

### 5.2 Payload do evento

Conteúdo definido em [09:43] por Diego. Itens do pedido **não** entram, para manter o corpo enxuto; quem precisar de detalhe consulta `GET /orders/:id`.

```json
{
  "event_id": "b1e2c3d4-5f6a-7b8c-9d0e-1f2a3b4c5d6e",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-22T12:34:56.789Z",
  "order_id": "9f8e7d6c-5b4a-3c2d-1e0f-a9b8c7d6e5f4",
  "order_number": "ORD-000123",
  "customer_id": "3a2b1c0d-9e8f-7a6b-5c4d-3e2f1a0b9c8d",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "total_cents": 149900
}
```

O `timestamp` é ISO 8601, coerente com `pino.stdTimeFunctions.isoTime` usado em `src/shared/logger/index.ts`.

### 5.3 Processamento pelo worker

Loop em `src/modules/webhooks/webhook.worker.ts`, iniciado por `src/worker.ts`:

1. A cada **2 segundos** ([09:09] Diego), seleciona em batch pequeno as linhas com `status IN (PENDING, FAILED)` e `nextAttemptAt <= now()`, ordenadas por `createdAt` ascendente.
2. Marca as linhas selecionadas como `PROCESSING`.
3. Para cada evento: carrega o endpoint, resolve a secret ativa, calcula `HMAC-SHA256(payload, secret)` e monta os headers (§5.4).
4. `POST` no `url` do endpoint com **timeout de 10 segundos** ([09:42] Diego).
5. Registra a tentativa em `webhook_deliveries` com status, corpo truncado e `durationMs`.
6. **2xx** → `status = DELIVERED`. Qualquer outro código, timeout ou erro de conexão → fluxo de retry (§5.5).

Antes de enviar, o worker valida o tamanho do corpo: acima de **64KB**, o evento falha imediatamente com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem tentar entregar e sem consumir retries ([09:23] Sofia, [09:24] Diego e Larissa).

O worker roda como processo único e reusa `createPrismaClient()` de `src/config/database.ts` com a mesma `DATABASE_URL` ([09:30] Bruno), e registra shutdown em SIGINT/SIGTERM no mesmo formato de `src/server.ts`.

### 5.4 Headers do envio

Definidos em [09:44] por Diego, com o `X-Webhook-Id` acrescentado pela Sofia no mesmo ponto.

| Header | Conteúdo |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento, chave de deduplicação do cliente ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)) |
| `X-Signature` | `sha256=<hex>` do HMAC-SHA256 do corpo com a secret do endpoint |
| `X-Timestamp` | Timestamp ISO 8601 do envio, para o cliente detectar replay attack se quiser |
| `X-Webhook-Id` | `id` do endpoint cadastrado, para clientes com vários cadastros saberem qual caiu naquele envio ([09:44] Sofia) |

Durante o grace period de rotação, `X-Signature` carrega as duas assinaturas separadas por vírgula (`sha256=<nova>,sha256=<antiga>`), para o cliente validar com qualquer uma enquanto migra ([09:21] Sofia).

### 5.5 Retry com backoff

Ao falhar, incrementa `attempts` e calcula `nextAttemptAt` pela tabela fixa ([09:17] Diego e Larissa):

| Tentativa concluída | Próxima tentativa em | Acumulado desde a 1ª falha |
|---|---|---|
| 1 | +1 minuto | 1m |
| 2 | +5 minutos | 6m |
| 3 | +30 minutos | 36m |
| 4 | +2 horas | 2h36m |
| 5 | — (DLQ) | ~14h36m |

A linha volta para `status = FAILED` com `nextAttemptAt` agendado. Depois da 5ª falha, vai para DLQ.

### 5.6 DLQ e replay

1. Esgotadas as 5 tentativas, o worker insere em `webhook_dead_letter` com `payload`, `failureReason` e `movedAt`, e remove ou marca definitivamente a linha da outbox ([09:18] Diego).
2. Nada é reprocessado automaticamente.
3. Um usuário com role `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`. O handler recria a linha na outbox com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`, e preenche `replayedAt` e `replayedById` para auditoria ([09:36] Sofia).
4. O log do replay é emitido via Pino com `userId`, `deadLetterId` e `newOutboxEventId`.

## 6. Contratos públicos

Todos os endpoints ficam sob `/api/v1`, montados em `src/routes/index.ts` via `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`. Todo o router aplica `authenticate` (`src/middlewares/auth.middleware.ts`); apenas o replay de DLQ acrescenta `requireRole('ADMIN')` ([09:36] Larissa).

O `customerId` vem no corpo ou no path, **não do JWT**, porque o token atual representa o usuário operador do nosso sistema, não o cliente ([09:32] Larissa e Bruno).

### 6.1 `POST /api/v1/webhooks` — cadastrar endpoint

Origem: [09:31] Marcos.

Request:

```json
{
  "customerId": "3a2b1c0d-9e8f-7a6b-5c4d-3e2f1a0b9c8d",
  "url": "https://atlas-comercial.example.com/hooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` — **a secret é gerada por nós e devolvida apenas aqui e na rotação**:

```json
{
  "id": "7c6b5a49-3821-4f0e-9d1c-2b3a4c5d6e7f",
  "customerId": "3a2b1c0d-9e8f-7a6b-5c4d-3e2f1a0b9c8d",
  "url": "https://atlas-comercial.example.com/hooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f2a...c81d",
  "createdAt": "2026-09-22T12:00:00.000Z"
}
```

Status: `201` criado · `400` `VALIDATION_ERROR` ou `WEBHOOK_INVALID_URL` (URL não-`https`) · `401` sem token · `404` `NOT_FOUND` (customer inexistente).

### 6.2 `GET /api/v1/webhooks?customerId=...&page=1&pageSize=20` — listar

Origem: [09:33] Bruno. Usa `paginated()` de `src/shared/http/response.ts`. A secret **nunca** aparece em listagem.

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "7c6b5a49-3821-4f0e-9d1c-2b3a4c5d6e7f",
      "customerId": "3a2b1c0d-9e8f-7a6b-5c4d-3e2f1a0b9c8d",
      "url": "https://atlas-comercial.example.com/hooks/oms",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-22T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200` · `400` query inválida · `401`.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar

Origem: [09:33] Bruno.

Request (campos opcionais):

```json
{ "url": "https://atlas-comercial.example.com/hooks/oms-v2", "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK` — recurso atualizado, sem a secret:

```json
{
  "id": "7c6b5a49-3821-4f0e-9d1c-2b3a4c5d6e7f",
  "customerId": "3a2b1c0d-9e8f-7a6b-5c4d-3e2f1a0b9c8d",
  "url": "https://atlas-comercial.example.com/hooks/oms-v2",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-09-22T14:10:00.000Z"
}
```

Status: `200` · `400` `WEBHOOK_INVALID_URL` ou `VALIDATION_ERROR` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover

Origem: [09:33] Bruno. Response `204 No Content`, sem corpo, no mesmo padrão de `OrderController.delete`.

Status: `204` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret

Origem: [09:21] Sofia. Sem corpo de request.

Response `200 OK`:

```json
{
  "id": "7c6b5a49-3821-4f0e-9d1c-2b3a4c5d6e7f",
  "secret": "whsec_4d7e...aa93",
  "previousSecretExpiresAt": "2026-09-23T12:00:00.000Z"
}
```

Durante as 24 horas de grace period, os envios carregam as duas assinaturas ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)).

Status: `200` · `401` · `404` `WEBHOOK_NOT_FOUND` · `409` `WEBHOOK_ROTATION_IN_PROGRESS` (rotação pedida com grace period anterior ainda aberto).

### 6.6 `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=100` — histórico de entregas

Origem: [09:34] Marcos — "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta".

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "e1d2c3b4-a596-4837-9a0b-1c2d3e4f5a6b",
      "eventId": "b1e2c3d4-5f6a-7b8c-9d0e-1f2a3b4c5d6e",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "durationMs": 342,
      "requestPayload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "responseBody": "{\"ok\":true}",
      "createdAt": "2026-09-22T12:01:04.120Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Status: `200` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ (ADMIN)

Origem: [09:18] e [09:35] Diego; restrição de role em [09:36] Sofia. Sem corpo de request.

Response `202 Accepted`:

```json
{
  "deadLetterId": "5a4b3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d",
  "newOutboxEventId": "0f1e2d3c-4b5a-4968-9788-1a2b3c4d5e6f",
  "status": "PENDING",
  "replayedBy": "d4c3b2a1-0f9e-4d8c-7b6a-5e4f3a2b1c0d",
  "replayedAt": "2026-09-22T15:20:00.000Z"
}
```

Status: `202` · `401` · `403` `FORBIDDEN` (role diferente de `ADMIN`, erro já emitido por `requireRole`) · `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND` · `409` `WEBHOOK_ALREADY_REPLAYED`.

### 6.8 Contrato do lado do cliente (o que recebemos de volta)

Esperamos qualquer `2xx` para considerar entrega bem-sucedida, dentro de 10 segundos. Corpo da resposta é ignorado (apenas armazenado truncado em `webhook_deliveries`).

## 7. Matriz de erros

Todos os erros do módulo usam o prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa) e são classes que estendem `AppError` (`src/shared/errors/app-error.ts`), exportadas por `src/shared/errors/index.ts`. Nenhuma mudança é necessária em `src/middlewares/error.middleware.ts`: ele já serializa qualquer `AppError` para `{ error: { code, message, details } }` ([09:29] Bruno).

| Código | HTTP | Classe base | Quando ocorre |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Endpoint de webhook inexistente no path |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL ausente, malformada ou não-`https` ([09:23] Sofia) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `BadRequestError` | Operação que exige secret sem secret resolvível |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `BadRequestError` | `subscribedStatuses` com valor fora do enum `OrderStatus` |
| `WEBHOOK_DUPLICATE_ENDPOINT` | 409 | `ConflictError` | Mesma `url` já cadastrada e ativa para o mesmo `customerId` |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Rotação pedida com grace period anterior ainda aberto |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | `id` de DLQ inexistente no replay |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `ConflictError` | Linha de DLQ já reprocessada (`replayedAt` preenchido) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Corpo do evento acima de 64KB; falha sem tentar entregar ([09:24] Diego e Larissa) |
| `WEBHOOK_ENDPOINT_INACTIVE` | 422 | `UnprocessableEntityError` | Tentativa de operar sobre endpoint desativado |

Erros internos do worker (não expostos por HTTP, apenas logados e gravados em `lastError` / `failureReason`):

| Código | Significado |
|---|---|
| `WEBHOOK_DELIVERY_TIMEOUT` | Cliente não respondeu dentro dos 10 segundos ([09:42] Diego) |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Resposta fora da faixa 2xx |
| `WEBHOOK_DELIVERY_CONNECTION_ERROR` | DNS, TLS ou conexão recusada |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | 5 tentativas esgotadas, evento movido para DLQ |

Os erros de validação de schema continuam saindo como `VALIDATION_ERROR` pelo caminho do `ZodError` já tratado no `errorMiddleware`.

## 8. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
|---|---|---|
| **Timeout** | 10 segundos por chamada HTTP; estouro conta como falha e agenda retry | [09:42] Diego |
| **Retries** | 5 tentativas no total | [09:15]–[09:17] Diego e Larissa |
| **Backoff** | Progressão fixa 1m / 5m / 30m / 2h / 12h | [09:17] Diego |
| **Fallback terminal** | DLQ em `webhook_dead_letter`, replay manual por ADMIN | [09:18] Diego |
| **Isolamento de processo** | Worker fora da API: crash de um não derruba o outro | [09:11] Diego |
| **Atomicidade** | Evento inserido na mesma transação; falha de inserção faz rollback da mudança de status | [09:40] Bruno |
| **Proteção de payload** | Corte em 64KB antes do envio | [09:23]–[09:24] Sofia, Diego, Larissa |
| **Isolamento entre clientes** | Um endpoint lento não bloqueia outro: cada evento tem timeout próprio e batch é pequeno | [09:08] Diego |
| **Graceful shutdown** | SIGINT/SIGTERM finalizam o ciclo corrente antes de sair, no padrão de `src/server.ts` | `src/server.ts` |

**Limitação conhecida:** não há garantia de ordering global. A ordem é preservada por `order_id` enquanto houver um único worker ([09:13] Larissa e Diego).

## 9. Observabilidade

### 9.1 Métricas

| Métrica | Tipo | Labels | Uso |
|---|---|---|---|
| `webhook_events_enqueued_total` | contador | `event_type`, `to_status` | Volume de eventos gerados |
| `webhook_delivery_attempts_total` | contador | `result` (`success`/`http_error`/`timeout`/`connection_error`), `attempt` | Taxa de sucesso e perfil de falha |
| `webhook_delivery_duration_seconds` | histograma | `webhook_endpoint_id` | Tempo de resposta do cliente; alimenta `durationMs` do histórico |
| `webhook_outbox_pending_gauge` | gauge | — | Backlog da outbox; dispara alerta se crescer |
| `webhook_outbox_lag_seconds` | gauge | — | `now() - min(createdAt)` dos pendentes; guarda o SLO de 10 segundos |
| `webhook_dead_letter_total` | contador | `webhook_endpoint_id` | Eventos mortos por cliente; sinal de integração quebrada |

Alertas mínimos: lag acima de 10 segundos por mais de 1 minuto, e qualquer entrada em DLQ.

### 9.2 Logs

Pino já configurado em `src/shared/logger/index.ts`, com `base: { service, env }` e timestamp ISO. O worker usa o mesmo logger, com `service` distinguindo o processo.

Eventos logados, em formato estruturado no estilo `snake_case` já usado (`server_started`, `shutdown_initiated`):

- `webhook_event_enqueued` — `event_id`, `order_id`, `webhook_endpoint_id`, `to_status`
- `webhook_delivery_attempt` — `event_id`, `attempt`, `response_status`, `duration_ms`
- `webhook_delivery_failed` — `event_id`, `attempt`, `error_code`, `next_attempt_at`
- `webhook_moved_to_dead_letter` — `event_id`, `webhook_endpoint_id`, `failure_reason`
- `webhook_dead_letter_replayed` — `dead_letter_id`, `new_outbox_event_id`, `user_id` (auditoria exigida em [09:36] por Sofia)

**Ponto de atenção de segurança:** a lista de `redactPaths` em `src/shared/logger/index.ts` cobre `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, mas **não** cobre a secret do webhook. Adicionar `*.secret` e `*.previousSecret` à lista é pré-requisito da revisão da Sofia.

### 9.3 Tracing

O `requestId` já propagado por `src/middlewares/request-logger.middleware.ts` é persistido na linha da outbox como atributo de correlação, para que a mudança de status na API e a entrega no worker apareçam na mesma trace. O `event_id` (`X-Event-Id`) funciona como chave de correlação de ponta a ponta: aparece no log da API, no log do worker, no histórico de entregas e no header recebido pelo cliente.

## 10. Dependências e compatibilidade

### 10.1 Dependências de runtime

Nenhuma dependência nova de infraestrutura ([09:07] Diego). A feature usa o que já está declarado em `package.json`:

| Dependência | Versão no projeto | Uso na feature |
|---|---|---|
| `@prisma/client` / `prisma` | 5.22.0 | Models novos, migração, `TransactionClient` |
| `zod` | 3.23.8 | Schemas de validação, incluindo o refinamento de `https` |
| `pino` | 9.5.0 | Logs estruturados do módulo e do worker |
| `express` | 4.21.1 | Router do módulo |
| `jsonwebtoken` | 9.0.2 | Reuso indireto via `authenticate` |
| `uuid` | 11.0.3 | Geração do `event_id` |
| `node:crypto` | nativo | `createHmac('sha256', ...)` e geração de secret |
| `fetch` | nativo (Node >= 20) | Chamada HTTP do worker com `AbortSignal.timeout(10_000)` |

`node:crypto` e `fetch` são nativos do Node 20, versão mínima já declarada em `engines` no `package.json`. Nenhum pacote HTTP novo é necessário.

### 10.2 Mudanças de configuração

Novas variáveis a acrescentar ao schema de `src/config/env.ts` e ao `.env.example`, todas com default:

| Variável | Default | Origem |
|---|---|---|
| `WEBHOOK_POLL_INTERVAL_MS` | `2000` | [09:09] Diego |
| `WEBHOOK_HTTP_TIMEOUT_MS` | `10000` | [09:42] Diego |
| `WEBHOOK_MAX_ATTEMPTS` | `5` | [09:17] Larissa |
| `WEBHOOK_MAX_PAYLOAD_BYTES` | `65536` | [09:24] Diego e Larissa |
| `WEBHOOK_SECRET_GRACE_PERIOD_HOURS` | `24` | [09:21] Sofia |
| `WEBHOOK_BATCH_SIZE` | `20` | [09:08] Diego (batch pequeno) |

### 10.3 Compatibilidade

- **Retrocompatível.** Nenhum endpoint existente muda de contrato. Clientes sem webhook cadastrado não sofrem alteração alguma: sem endpoint elegível, nenhuma linha é inserida na outbox ([09:34] Bruno).
- **Migração é aditiva.** Apenas `CREATE TABLE`; nenhuma tabela existente é alterada.
- **Deploy.** A migração e a API podem subir antes do worker. Eventos acumulam na outbox e são drenados quando o worker sobe.
- **Rollback.** Desligar o worker interrompe as entregas sem afetar a API; eventos ficam pendentes na outbox.

## 11. Integração com o sistema existente

Esta seção nomeia os pontos exatos do código base que a feature toca.

**Convenção de leitura:** todo caminho citado sem marcação **existe hoje** no repositório e pode ser aberto para conferência. Os quatro arquivos a serem **criados** por esta feature estão sempre marcados com _(novo)_: `src/worker.ts` _(novo)_, `src/modules/webhooks/webhook.publisher.ts` _(novo)_, `src/modules/webhooks/webhook.worker.ts` _(novo)_ e `src/modules/webhooks/webhook.schemas.ts` _(novo)_, além dos demais arquivos do módulo `src/modules/webhooks/` _(novo)_.

### 11.1 `src/modules/orders/order.service.ts` — o gancho crítico

`OrderService.changeStatus` hoje executa, dentro de `this.prisma.$transaction(async (tx) => { ... })`: busca do pedido, validação via `canTransition`, `debitStock`/`replenishStock`, `tx.order.update`, `tx.orderStatusHistory.create` e recarga do pedido com `include`.

A extensão é **uma linha**, inserida após `tx.orderStatusHistory.create(...)` e antes da recarga final:

```ts
await publishWebhookEvent(tx, order, from, to);
```

Nada mais no método muda. A função recebe o `Prisma.TransactionClient` já tipado como `TxClient` no topo do arquivo, e não é injetada no construtor do serviço ([09:41] Bruno e Diego). Falha na publicação propaga a exceção e o `$transaction` reverte tudo ([09:40] Bruno).

### 11.2 `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` — reuso das classes de erro

As classes de webhook seguem exatamente o padrão de `InsufficientStockError` e `InvalidStatusTransitionError`: estendem a classe HTTP apropriada e fixam o `errorCode`.

```ts
export class WebhookNotFoundError extends NotFoundError {
  constructor() { super('Webhook endpoint'); }
}

export class WebhookInvalidUrlError extends BadRequestError {
  constructor(url: string) {
    super('Webhook URL must use https', 'WEBHOOK_INVALID_URL', { url });
  }
}

export class WebhookPayloadTooLargeError extends UnprocessableEntityError {
  constructor(sizeBytes: number) {
    super('Webhook payload exceeds size limit', 'WEBHOOK_PAYLOAD_TOO_LARGE', { sizeBytes, limitBytes: 65536 });
  }
}
```

Todas são reexportadas por `src/shared/errors/index.ts`, mantendo o import único usado pelos demais módulos. Como o `errorMiddleware` despacha por `instanceof AppError`, a serialização HTTP funciona sem nenhuma alteração ([09:29] Bruno).

### 11.3 `src/middlewares/auth.middleware.ts` — autenticação e role no replay

O router de webhooks aplica `router.use(authenticate)` no topo, exatamente como `buildOrderRouter` faz em `src/modules/orders/order.routes.ts`. O endpoint de replay acrescenta o `requireRole` já existente:

```ts
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  requireRole('ADMIN'),
  validate({ params: deadLetterIdParamSchema }),
  controller.replayDeadLetter,
);
```

`requireRole` já emite `ForbiddenError` com código `FORBIDDEN` quando a role não bate, então o contrato de erro do 403 vem de graça ([09:36] Larissa).

### 11.4 `src/modules/orders/order.status.ts` — fonte da verdade dos status

O filtro `subscribedStatuses` é validado contra o enum `OrderStatus` do Prisma, o mesmo usado pela máquina de estados (`transitions`) deste arquivo. O worker não reimplementa nenhuma regra de transição: ele apenas transporta `from_status` e `to_status` que o `OrderService` já validou via `canTransition`. Transições inválidas nunca chegam a gerar evento, porque `InvalidStatusTransitionError` aborta a transação antes da inserção na outbox.

### 11.5 `src/server.ts` e `package.json` — novo entry-point

`src/worker.ts` espelha a estrutura de `src/server.ts`: função `bootstrap()`, logger Pino, `prisma.$disconnect()` no shutdown e handlers de `SIGINT`/`SIGTERM`. A diferença é que, no lugar de `app.listen`, inicia o loop de polling e, no shutdown, aguarda o ciclo corrente terminar.

Em `package.json`, dois scripts novos no mesmo estilo dos existentes:

```json
"worker": "node --env-file=.env dist/worker.js",
"worker:dev": "tsx watch --env-file=.env src/worker.ts"
```

### 11.6 Outros pontos de contato

| Arquivo | Alteração |
|---|---|
| `prisma/schema.prisma` | Quatro models novos e um enum de status da outbox |
| `src/routes/index.ts` | `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e o tipo em `Controllers` |
| `src/app.ts` | Instanciação de repository, service e controller em `buildControllers` |
| `src/config/env.ts` | Seis variáveis novas no `envSchema`, todas com default |
| `src/shared/logger/index.ts` | Acrescentar `*.secret` e `*.previousSecret` a `redactPaths` |
| `src/shared/http/response.ts` | Reuso de `paginated()` nas listagens, sem alteração |
| `src/middlewares/validate.middleware.ts` | Reuso direto, sem alteração |
| `tests/helpers/factories.ts` | Factories novas para endpoint e evento de outbox |

## 12. Critérios de aceite técnicos

| # | Critério |
|---|---|
| CA-1 | Mudança de status de pedido cujo customer tem endpoint ativo assinando aquele status gera exatamente uma linha em `webhook_outbox` por endpoint elegível, na mesma transação |
| CA-2 | Se a inserção na outbox falhar, `orders`, `order_status_history` e `stockQuantity` permanecem inalterados (rollback completo) |
| CA-3 | Customer sem endpoint ativo, ou com endpoint que não assina aquele status, não gera nenhuma linha na outbox |
| CA-4 | O worker entrega o evento em até 2 segundos após o commit no caminho feliz, medido por `webhook_outbox_lag_seconds` |
| CA-5 | Toda requisição enviada carrega `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` |
| CA-6 | `X-Signature` verifica corretamente como `HMAC-SHA256(corpo_exato_enviado, secret_do_endpoint)` |
| CA-7 | Durante o grace period de 24h, o envio valida tanto com a secret nova quanto com a antiga; após expirar, só com a nova |
| CA-8 | Resposta não-2xx ou timeout de 10s agenda nova tentativa exatamente em 1m, 5m, 30m, 2h e 12h |
| CA-9 | Após a 5ª falha, o evento aparece em `webhook_dead_letter` com payload e motivo, e não é mais reprocessado automaticamente |
| CA-10 | `POST /admin/webhooks/dead-letter/:id/replay` com role `OPERATOR` retorna 403; com `ADMIN`, retorna 202 e recria a linha na outbox |
| CA-11 | Replay grava `replayedById` e emite log `webhook_dead_letter_replayed` com o `user_id` |
| CA-12 | Cadastro com URL `http://` é recusado com 400 e código `WEBHOOK_INVALID_URL` |
| CA-13 | Payload acima de 64KB falha com `WEBHOOK_PAYLOAD_TOO_LARGE` sem consumir tentativas de retry |
| CA-14 | A secret aparece apenas nas respostas de criação e rotação, nunca em listagem, leitura ou log |
| CA-15 | `GET /webhooks/:id/deliveries` retorna as tentativas com `success`, `responseStatus`, `durationMs` e payload, paginadas |
| CA-16 | Matar o processo do worker não afeta a API; matar a API não afeta o worker |
| CA-17 | Todos os códigos de erro do módulo começam com `WEBHOOK_` e são serializados pelo `errorMiddleware` sem alteração nele |
| CA-18 | Para um mesmo `order_id`, com worker único, as entregas saem na ordem de `createdAt` |

### Estratégia de testes

- **Unitários:** cálculo de backoff, montagem do payload, cálculo de HMAC e assinatura dupla no grace period, validação de schema (`https`, enum de status).
- **Integração:** `changeStatus` gerando outbox dentro da transação e revertendo em caso de falha, no estilo de `tests/orders.test.ts` com as factories de `tests/helpers/factories.ts`.
- **Ponta a ponta:** ciclo completo com endpoint HTTP local simulando 2xx, 5xx, timeout e corpo grande, cobrindo retry e DLQ.
- **Segurança:** revisão dedicada da Sofia sobre geração de secret, HMAC e redação de log, com 2 dias úteis reservados antes do deploy ([09:46] Sofia).

## 13. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| R-1 | Regressão em `changeStatus`, caminho mais crítico do OMS | Média | Alto | Alteração de uma linha, função pura recebendo `tx`; testes de integração cobrindo rollback (CA-2) |
| R-2 | Erro na implementação do HMAC ou entropia fraca na geração de secret | Média | Alto | `node:crypto` com `randomBytes`; revisão de segurança bloqueante antes do deploy ([09:46] Sofia) |
| R-3 | Secret vazar em log da aplicação | Baixa | Alto | Acrescentar `*.secret` e `*.previousSecret` ao `redactPaths` de `src/shared/logger/index.ts`; secret exibida apenas na criação e rotação |
| R-4 | Cliente não deduplicar por `X-Event-Id` e processar o evento duas vezes | Alta | Médio | Documentação destacada no portal de desenvolvedor ([09:26] Marcos); `event_id` também no corpo |
| R-5 | Rajada de mudanças de status bombardeando um cliente | Média | Médio | Batch pequeno e timeout curto limitam a vazão; rate limiting fica como questão em aberto ([09:39] Diego) |
| R-6 | Crescimento descontrolado da `webhook_outbox` degradando o polling | Alta | Médio | Índice composto `(status, nextAttemptAt)`; gauge de backlog com alerta; arquivamento fora de escopo mas monitorado |
| R-7 | Worker morrer silenciosamente e eventos pararem de sair | Média | Alto | Alerta sobre `webhook_outbox_lag_seconds` acima de 10s; restart policy do supervisor |
| R-8 | Perda de ordering ao escalar o worker no futuro | Baixa (nesta fase) | Médio | Limitação documentada; escala horizontal exigirá particionamento por `order_id` ou lock pessimista ([09:13] Diego) |
| R-9 | Prazo apertado: Atlas espera entrega para fim de novembro | Média | Alto | Três sprints estimados com fatiamento acordado ([09:46] Larissa), revisão de segurança incluída no fim |

