# ADR-007 — Payload renderizado como snapshot no momento da inserção na outbox

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno / Pedidos), Diego (Eng. Sênior / Plataforma)
- **Decisão relacionada a:** [ADR-001](ADR-001-padrao-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Definido o outbox ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), resta decidir **o que a linha da outbox guarda**: o payload JSON já montado, ou apenas uma referência (`order_id`, `from_status`, `to_status`) a ser resolvida em consulta no momento do envio.

A pergunta foi levantada literalmente no fim da reunião:

> [09:51] Bruno: o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?

O cenário problemático é concreto: com retry de até 12 horas de espera ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), um pedido pode ter mudado de status várias vezes entre a geração do evento e a entrega efetiva.

## Decisão

**O payload é renderizado e persistido como snapshot no instante da inserção na outbox**, dentro da mesma transação de `changeStatus`.

> [09:52] Larissa: Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito.

> [09:52] Diego: Concordo, snapshot na inserção.

> [09:52] Bruno: Beleza, snapshot. Decidido.

O conteúdo do snapshot foi definido em [09:43] por Diego: `event_id`, `event_type` (`order.status_changed`), `timestamp` em ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. **Itens do pedido não entram**, para manter o payload enxuto; o cliente que precisar de detalhe consulta `GET /orders/:id` depois.

## Alternativas Consideradas

### 1. Guardar apenas referências e renderizar na hora do envio

Armazenar `order_id`, `from_status`, `to_status` e montar o JSON consultando `orders` no momento da chamada HTTP.

- **Trade-off que motivou o descarte:** o payload passaria a refletir o estado **atual** do pedido, não o estado no momento do evento. Um evento de `PAID` reentregue 12 horas depois chegaria descrevendo um pedido já `SHIPPED` — exatamente o "caso esquisito" apontado pela Larissa ([09:52]). Economiza espaço em disco ao custo de correção semântica.

### 2. Snapshot completo do pedido, incluindo itens

- **Trade-off que motivou o descarte:** infla a linha da outbox e o corpo da requisição sem demanda. A decisão explícita foi manter o payload enxuto ([09:43] Diego, [09:44] Bruno), e o limite de 64KB por envio ([09:24] Diego, [09:24] Larissa) reforça a escolha.

## Consequências

### Positivas

- O evento entregue é fiel ao momento em que o status mudou, independentemente de quantos retries ocorreram ou de quanto o pedido mudou depois.
- O envio não depende de consulta ao banco além da leitura da própria linha da outbox, simplificando e acelerando o worker.
- O corpo assinado por HMAC ([ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)) é estável entre tentativas: a mesma assinatura vale para o mesmo `event_id`.
- Replay de DLQ reenvia exatamente o que deveria ter sido enviado originalmente, já que a linha de dead letter carrega o payload ([09:18] Diego).

### Negativas

- Duplicação de dados: o JSON do evento coexiste com a fonte da verdade em `orders`, aumentando o tamanho da tabela `webhook_outbox` e o custo da política de arquivamento.
- Payload congelado pode ficar **desatualizado em relação à realidade** quando entregue com horas de atraso. É intencional, mas precisa estar documentado no portal do desenvolvedor para não confundir o cliente.
- Mudar o formato do payload não afeta retroativamente eventos já enfileirados: durante um deploy, eventos com formato antigo e novo convivem na outbox.

