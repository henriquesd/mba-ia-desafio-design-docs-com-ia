# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos)
- **Consultados:** Marcos (PM), Sofia (Eng. Segurança)
- **Decisão relacionada a:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md)

## Contexto

Os clientes B2B precisam ser notificados quando o status de um pedido muda. A mudança de status já é uma operação transacional pesada hoje: `OrderService.changeStatus` abre um `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e debita ou repõe `stockQuantity` dos produtos (`src/modules/orders/order.service.ts`).

Disparar a chamada HTTP de forma síncrona dentro dessa transação foi descartado logo no início:

> [09:04] Bruno: Síncrono não rola. A transação de mudança de status hoje já é pesada [...] Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos.

> [09:04] Bruno: Sem falar que se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá.

O problema central é de consistência: precisamos garantir que **se a mudança de status commitou, o evento existe**, e que **se a transação deu rollback, o evento não existe**. Nenhuma fila externa oferece isso sem coordenação de dois recursos distintos.

## Decisão

Adotamos o **padrão Outbox sobre o MySQL já existente**. Na mesma transação SQL que atualiza `orders` e `order_status_history`, inserimos uma linha na tabela `webhook_outbox` com o evento a ser entregue.

> [09:06] Diego: quando o status do pedido muda, dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox com o evento. [...] Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto.

> [09:08] Larissa: Tá decidido então: outbox em MySQL.

Características da decisão:

- A tabela `webhook_outbox` vive no mesmo schema Prisma (`prisma/schema.prisma`), com chave primária `uuid` seguindo o padrão do projeto ([09:51] Larissa).
- Índices em `status` (PENDING, PROCESSING, FAILED, DELIVERED) e em `createdAt`, para o worker varrer apenas pendentes ([09:08] Diego).
- A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `TransactionClient` da transação corrente, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno e Diego).
- Se a inserção na outbox falhar, a transação inteira faz rollback ([09:40] Bruno).
- Filtragem por status assinado acontece **na inserção**: se nenhum webhook ativo do customer escuta aquele status, a linha não é criada ([09:34] Bruno e Diego).

## Alternativas Consideradas

### 1. Disparo HTTP síncrono dentro de `changeStatus`

Chamar o endpoint do cliente diretamente dentro da transação.

- **Trade-off que motivou o descarte:** acopla a latência da mudança de status à disponibilidade do cliente. Um cliente lento seguraria a transação, e um cliente fora do ar forçaria a escolha impossível entre dar rollback numa mudança de status válida ou perder o evento ([09:04] Bruno).

### 2. Redis Streams (ou message broker dedicado)

Publicar os eventos numa stream Redis consumida por um worker.

- **Trade-off que motivou o descarte:** exige subir e operar infraestrutura nova (Redis Cluster) para um time pequeno, e reintroduz o problema de dupla escrita — a transação MySQL e a publicação no Redis não são atômicas entre si.

> [09:07] Diego: a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve.

### 3. Trigger de banco de dados

Usar trigger no MySQL para reagir à mudança em `orders`.

- **Trade-off que motivou o descarte:** o MySQL não tem `LISTEN/NOTIFY` como o PostgreSQL. A trigger só executa SQL e não consegue acordar um processo externo sem gambiarra (escrever em arquivo, bater em endpoint) ([09:09] Diego).

## Consequências

### Positivas

- Atomicidade real entre mudança de status e registro do evento, sem transação distribuída.
- Zero infraestrutura nova: reaproveita MySQL, Prisma e o pool de conexão já configurados em `src/config/database.ts`.
- A outbox é auditável: fica registro em banco de todo evento gerado, com histórico de tentativas.
- Rollback natural — um pedido cuja mudança de status falhou nunca gera notificação fantasma.

### Negativas

- A tabela `webhook_outbox` cresce com o volume de pedidos e precisa de política de arquivamento. Ficou explicitamente **fora do escopo desta feature**: "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature" ([09:08] Diego).
- Introduz carga de escrita adicional na transação crítica de `changeStatus`, que já é a mais pesada do sistema.
- A entrega não é imediata: existe latência inerente entre o commit e a leitura pelo worker (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- A leitura em polling gera consultas recorrentes ao MySQL mesmo em períodos sem eventos.

