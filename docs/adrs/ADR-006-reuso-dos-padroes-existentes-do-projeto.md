# ADR-006 — Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Bruno (Eng. Pleno / Pedidos), Larissa (Tech Lead)
- **Consultados:** Diego (Eng. Sênior / Plataforma), Sofia (Eng. Segurança)
- **Decisão relacionada a:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)

> Este é o ADR que ancora o desenho no código existente. Todos os caminhos citados abaixo existem no repositório.

## Contexto

O OMS já tem um padrão arquitetural estabelecido e consistente em cinco módulos (`auth`, `users`, `customers`, `products`, `orders`). A feature de webhooks é grande o suficiente para tentar trazer convenções próprias, e isso criaria duas formas de fazer a mesma coisa dentro do mesmo repositório.

> [09:27] Bruno: A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual.

## Decisão

O módulo de webhooks segue **exatamente** as convenções já em uso. Nada de infraestrutura nova é introduzido.

> [09:30] Larissa: Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros.

### O que é reutilizado, com o ponto exato do código

| Padrão existente | Onde está hoje | Como o módulo de webhooks usa |
|---|---|---|
| Estrutura de módulo (controller / service / repository / routes / schemas) | `src/modules/orders/` | `src/modules/webhooks/` com os mesmos cinco arquivos ([09:27] Bruno) |
| Hierarquia de erros | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Novas classes estendendo `AppError` / `NotFoundError` / `ConflictError`, exportadas por `src/shared/errors/index.ts` |
| Convenção de código de erro | `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` em `src/shared/errors/http-errors.ts` | Prefixo `WEBHOOK_` em todos os códigos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa) |
| Tratamento centralizado de erro | `src/middlewares/error.middleware.ts` | Nenhuma alteração necessária: já trata `AppError`, `ZodError` e `PrismaClientKnownRequestError` de forma genérica ([09:29] Bruno) |
| Validação de entrada | `src/middlewares/validate.middleware.ts` + `src/modules/orders/order.schemas.ts` | `validate({ body, params, query })` com schemas Zod, incluindo o refinamento de URL `https` ([09:23] Sofia) |
| Autenticação e autorização | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` | `authenticate` no router inteiro; `requireRole('ADMIN')` apenas no replay de DLQ ([09:36] Larissa) |
| Logging | Pino em `src/shared/logger/index.ts` | Mesmo logger, mesma configuração de `redact` ([09:29] Bruno) |
| Paginação de listagem | `paginated()` em `src/shared/http/response.ts` | Listagens de webhooks e de deliveries devolvem `PaginatedResponse` |
| Montagem do router | `buildApiRouter` em `src/routes/index.ts` | `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` |
| Injeção de dependência | `buildControllers` em `src/app.ts` | Repository, service e controller instanciados no mesmo ponto, sem container de DI |
| Persistência | `prisma/schema.prisma`, `src/config/database.ts` | Novos models no mesmo schema, `uuid` como PK ([09:51] Larissa) |
| Entry-point de processo | `src/server.ts` | `src/worker.ts` _(novo)_ espelhando bootstrap, shutdown e logger ([09:11] Larissa) |

### O que explicitamente **não** é introduzido

- Nenhuma biblioteca nova de logging, validação, HTTP ou fila.
- Nenhum container de injeção de dependência.
- Nenhuma alteração no `errorMiddleware`.

## Alternativas Consideradas

### 1. Módulo de webhooks como serviço/pacote separado

Isolar webhooks fora de `src/modules`, com convenções próprias, mirando extração futura para outro repositório.

- **Trade-off que motivou o descarte:** a feature depende de participar da **mesma transação** de `OrderService.changeStatus` ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)). Separar o módulo tornaria isso impossível sem transação distribuída, destruindo a garantia que justifica o outbox.

### 2. Introduzir uma camada de eventos de domínio genérica

Criar um event bus interno reutilizável, com webhooks como um dos consumidores.

- **Trade-off que motivou o descarte:** abstração cara para um único caso de uso concreto e três clientes. Segue o mesmo raciocínio que descartou o Redis para um time pequeno ([09:07] Diego). Nada na reunião pediu generalidade além de `order.status_changed`.

### 3. Injetar o `WebhookRepository` inteiro no `OrderService`

- **Trade-off que motivou o descarte:** aumentaria o acoplamento e o construtor do serviço mais crítico do sistema. A alternativa escolhida é uma função que recebe o `tx`:

> [09:41] Bruno: Vou propor uma função publishWebhookEvent(tx, order, fromStatus, toStatus) que aceita o tx client da transação atual.

> [09:41] Diego: Boa, função pura recebendo o tx. Não precisa injetar repository inteiro.

## Consequências

### Positivas

- Curva de aprendizado nula: quem conhece `src/modules/orders/` sabe navegar `src/modules/webhooks/`.
- O `errorMiddleware` já traduz os novos erros para HTTP sem nenhuma mudança, porque despacha por `instanceof AppError` ([09:29] Bruno).
- Testes seguem a estrutura de `tests/orders.test.ts` e as factories de `tests/helpers/factories.ts`.
- Superfície de revisão de segurança menor: a Sofia revisa o que é novo (HMAC, geração de secret), não a plumbing ([09:46] Sofia).

### Negativas

- Herdamos também as limitações do padrão atual: sem container de DI, `buildControllers` em `src/app.ts` cresce a cada módulo.
- O padrão de módulo foi desenhado para CRUD sobre entidade; o worker não encaixa perfeitamente nele, exigindo um arquivo `webhook.worker.ts` que foge da tríade controller/service/repository ([09:28] Bruno).
- `src/shared/errors/index.ts` e `src/routes/index.ts` viram pontos de conflito recorrentes de merge à medida que módulos são adicionados.
- Reuso do `redact` do Pino exige revisão: a lista atual em `src/shared/logger/index.ts` cobre `*.token` e `*.password`, mas não um campo de secret de webhook.

