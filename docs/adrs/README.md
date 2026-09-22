# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**.

Cada decisão arquitetural é registrada em um arquivo individual, nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`, seguindo o formato MADR: **Status, Contexto, Decisão, Alternativas Consideradas e Consequências** (positivas e negativas, com trade-off explícito).

## Índice

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period | Aceito |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-outbox.md) | Payload renderizado como snapshot na inserção | Aceito |

Contexto de produto em [`../PRD.md`](../PRD.md), proposta técnica em [`../RFC.md`](../RFC.md), detalhamento de implementação em [`../FDD.md`](../FDD.md) e rastreabilidade em [`../TRACKER.md`](../TRACKER.md).
