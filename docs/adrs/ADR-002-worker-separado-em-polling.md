# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos)
- **Consultados:** Marcos (PM)
- **Decisão relacionada a:** [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Com a outbox decidida ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), falta definir **quem lê a tabela e dispara as chamadas HTTP**, e com que frequência.

O requisito de latência veio do PM e é explicitamente frouxo:

> [09:02] Marcos: Pra eles, qualquer coisa abaixo de 10 segundos já é "tempo real".

O sistema atual tem um único entry-point (`src/server.ts`), que sobe o Express via `buildApp` e registra shutdown em SIGINT/SIGTERM. Não existe nenhum processo de background hoje.

## Decisão

1. **Polling em loop, a cada 2 segundos.** O worker busca os eventos `PENDING` mais antigos em batch pequeno, processa e marca o resultado.

> [09:09] Diego: Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca.

> [09:10] Larissa: Vamos registrar isso como uma decisão. Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos.

2. **Processo do SO separado da API.** Um novo entry-point `src/worker.ts` _(arquivo novo)_, espelhando `src/server.ts` _(existente)_, exposto por um script `npm run worker` no `package.json`. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` _(arquivo novo)_.

> [09:11] Diego: o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker.

> [09:11] Larissa: Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script "npm run worker".

3. **`PrismaClient` próprio, mesmo banco.** O worker instancia seu próprio client via `createPrismaClient()` (`src/config/database.ts`), usando a mesma `DATABASE_URL` validada em `src/config/env.ts`.

> [09:30] Bruno: Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node.

4. **Single-worker por ora.** Roda uma única instância. Isso dá ordenação implícita por `createdAt` dentro de cada `order_id`, mas **não** é garantia de ordering global.

> [09:13] Larissa: Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker.

## Alternativas Consideradas

### 1. Worker dentro do processo da API (`setInterval` no `src/server.ts`)

- **Trade-off que motivou o descarte:** o ciclo de vida do worker ficaria amarrado ao da API. Reinício, deploy ou crash do servidor HTTP derrubaria o processamento de eventos ([09:11] Diego). Também competiria por event loop com o tráfego de requisições.

### 2. Trigger de banco reagindo a `orders` / `order_status_history`

- **Trade-off que motivou o descarte:** MySQL não tem `NOTIFY/LISTEN`. A trigger executa SQL e não notifica processo externo; acordar o worker exigiria improviso (arquivo, endpoint interno). O ganho de reatividade não se justifica quando 2 segundos já cabem folgado no orçamento de 10 segundos ([09:09] Diego).

### 3. Múltiplos workers em paralelo desde o início

- **Trade-off que motivou o descarte:** exigiria particionamento por `order_id` ou lock pessimista para preservar ordem, complexidade sem demanda atual. Foi classificado como problema futuro ([09:13] Diego).

## Consequências

### Positivas

- Isolamento de falhas: um crash do worker não derruba a API, e vice-versa.
- Deploy e escala independentes entre API e processamento de eventos.
- Simplicidade operacional: nenhum componente novo além de um processo Node com a mesma stack.
- Polling de 2s dá margem confortável (5x) contra o SLO de 10 segundos acordado com os clientes.

### Negativas

- **Latência mínima de até 2 segundos** entre o commit da mudança de status e o início do envio. Aceito explicitamente ([09:10] Larissa).
- Consultas recorrentes ao MySQL mesmo sem eventos pendentes (custo de polling ocioso).
- **Ordering não é garantido globalmente**, apenas por `order_id` e apenas enquanto houver um único worker. Escalar horizontalmente quebra essa propriedade e exigirá particionamento por `order_id` ou lock pessimista ([09:13] Diego).
- Novo artefato operacional para monitorar, alertar e manter vivo (supervisor, restart policy).

