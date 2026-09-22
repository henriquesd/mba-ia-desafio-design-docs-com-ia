# ADR-005 — Garantia de entrega at-least-once com deduplicação por X-Event-Id

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Diego (Eng. Sênior / Plataforma), Larissa (Tech Lead)
- **Consultados:** Sofia (Eng. Segurança), Bruno (Eng. Pleno / Pedidos), Marcos (PM)
- **Decisão relacionada a:** [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

A combinação de outbox ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)) com retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) cria um cenário inevitável de entrega duplicada: se o cliente processa a requisição mas a resposta se perde, ou se o timeout de 10 segundos estoura depois de o cliente já ter processado, o worker marca falha e reenvia o mesmo evento.

Não existe forma de distinguir, do nosso lado, "o cliente não recebeu" de "o cliente recebeu mas não conseguimos confirmar".

## Decisão

**Assumimos e documentamos a garantia de entrega como at-least-once.** Cada evento carrega um identificador único e imutável, gerado no momento em que entra na outbox, enviado no header `X-Event-Id`. A deduplicação é responsabilidade do receptor.

> [09:24] Diego: a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado.

> [09:25] Diego: A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele.

> [09:26] Larissa: Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.

O identificador é um **UUID**, coerente com o padrão de chave primária de todo o schema (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`), confirmado explicitamente ao final da reunião:

> [09:51] Larissa: UUID, segue o padrão do resto do projeto. Tudo é uuid.

O mesmo `event_id` aparece **também no corpo do payload**, junto de `event_type`, `timestamp` ISO 8601 e dados do pedido ([09:43] Diego), para que o cliente possa deduplicar sem depender de headers.

A obrigação do cliente de deduplicar será documentada com destaque no portal de desenvolvedor:

> [09:26] Marcos: Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema.

## Alternativas Consideradas

### 1. Exactly-once com protocolo de confirmação

Handshake de dois lados: cliente confirma o processamento e nós só então marcamos como entregue de forma definitiva.

- **Trade-off que motivou o descarte:** exige coordenação e estado dos dois lados da integração, elevando muito a complexidade e impondo mudança de contrato a três clientes B2B que só querem receber um POST. Além disso, é uma garantia que nenhum sistema distribuído entrega de verdade sem idempotência no receptor de qualquer forma.

> [09:25] Diego: Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos.

### 2. Deduplicação do nosso lado, por chave de negócio

Suprimir reenvio quando `(order_id, from_status, to_status)` já foi entregue.

- **Trade-off que motivou o descarte:** não resolve o caso real. A duplicata nasce justamente quando não sabemos se o cliente recebeu; suprimir o reenvio nesse cenário trocaria duplicata por **perda de evento**, que é falha pior. Também não foi a direção tomada na reunião, que atribuiu a dedup ao receptor ([09:25] Diego).

### 3. At-most-once (enviar uma vez e desistir)

- **Trade-off que motivou o descarte:** contradiz frontalmente a decisão de retry com backoff ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) e o motivo pelo qual a feature existe: os clientes querem parar de fazer polling justamente porque não confiam em perder mudança de status.

## Consequências

### Positivas

- Nenhum evento é perdido por falha transitória de rede ou de disponibilidade do cliente.
- Contrato simples e alinhado ao mercado: Stripe e GitHub usam o mesmo modelo, então os clientes já conhecem o padrão ([09:25] Diego).
- O `event_id` serve duplamente como chave de dedup para o cliente e como identificador de correlação nos nossos logs, métricas e no histórico de entregas.
- Não exige estado extra nem coordenação do nosso lado.

### Negativas

- **Transfere responsabilidade para o cliente**, ponto levantado explicitamente na reunião:

> [09:25] Sofia: Isso joga responsabilidade pro cliente.

  Cliente que não deduplicar pode processar o mesmo evento duas vezes, com efeito colateral no sistema dele. Mitigação é documental (portal de desenvolvedor) e não técnica.
- A documentação externa passa a ser parte do contrato da feature, não um extra opcional.
- Suporte tende a receber tickets de "recebi o mesmo evento duas vezes" de clientes que ignoraram a orientação.
- Combinado com a ausência de ordering global ([ADR-002](ADR-002-worker-separado-em-polling.md)), o cliente precisa de lógica defensiva: deduplicar por `event_id` e não assumir sequência entre eventos de pedidos distintos.

