# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos)
- **Consultados:** Marcos (PM), Sofia (Eng. Segurança)
- **Decisão relacionada a:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

## Contexto

O endpoint de destino é infraestrutura de terceiro, fora do nosso controle. Indisponibilidade do cliente é normal e esperada:

> [09:16] Diego: Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada.

Precisamos definir quantas vezes tentar, com que espaçamento, e o que fazer com o evento quando as tentativas se esgotarem. Sem um teto, o evento fica pendurado para sempre se o cliente sumiu de vez.

## Decisão

**Backoff exponencial com 5 tentativas e progressão fixa de 1m / 5m / 30m / 2h / 12h.** Esgotadas as tentativas, o evento é movido para a tabela `webhook_dead_letter`.

> [09:17] Diego: Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa.

> [09:17] Larissa: Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h.

A DLQ é uma **tabela separada**, não um estado terminal na própria outbox:

> [09:18] Diego: Eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento.

O reprocessamento é **manual, via endpoint administrativo**, e recoloca o evento na outbox como pendente:

> [09:18] Diego: Manual via endpoint admin. Tipo um POST /admin/webhooks/dead-letter/:id/replay. Recoloca na outbox como pendente.

O endpoint exige role `ADMIN` e registra quem executou o replay, reutilizando o `requireRole` de `src/middlewares/auth.middleware.ts` e o logger Pino de `src/shared/logger/index.ts`:

> [09:36] Sofia: Tem que ser ADMIN sim. [...] E o endpoint de admin tem que logar quem fez o replay, pra auditoria.

Uma tentativa é considerada falha quando a resposta não é 2xx ou quando o timeout de 10 segundos estoura ([09:42] Diego).

## Alternativas Consideradas

### 1. Três tentativas com janela curta

Proposta pelo Bruno como opção mais agressiva.

- **Trade-off que motivou o descarte:** cobre uma janela pequena demais. "Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria" ([09:16] Diego). Perderia eventos de clientes em manutenção planejada de poucas horas.

### 2. Retry indefinido com backoff

- **Trade-off que motivou o descarte:** eventos ficam pendurados eternamente se o cliente desapareceu, inflando a outbox e mascarando integrações mortas ([09:15] Diego). O teto de 5 tentativas cobre cerca de 15 horas, o que o PM considerou aceitável: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17] Marcos).

### 3. DLQ como estado `FAILED` na própria `webhook_outbox`

- **Trade-off que motivou o descarte:** polui a tabela quente que o worker varre a cada 2 segundos com linhas que nunca mais serão processadas automaticamente, degradando a leitura de pendentes ([09:18] Diego).

### 4. Replay automático da DLQ

- **Trade-off que motivou o descarte:** a decisão explícita foi "manual via endpoint admin" ([09:18] Diego). Um evento na DLQ indica problema que merece inspeção humana antes do reenvio; replay automático só repetiria o mesmo padrão de falha.

## Consequências

### Positivas

- Cobre indisponibilidades reais de cliente (até cerca de 15 horas) sem intervenção humana.
- Nenhum evento é silenciosamente perdido: ou entrega, ou vai para a DLQ com payload, motivo e timestamp preservados.
- A outbox quente fica pequena e rápida de varrer.
- A DLQ funciona como evidência de debug e como métrica de saúde da integração de cada cliente.

### Negativas

- Um evento pode levar até cerca de 15 horas para chegar ao cliente em cenário de falha prolongada, muito acima do SLO de 10 segundos do caminho feliz. É degradação aceita, não garantida.
- Reprocessamento exige ação humana; ninguém é notificado automaticamente quando algo cai na DLQ (notificação por e-mail ficou explicitamente para fase futura, [09:37] Larissa).
- A progressão longa (2h, 12h) faz uma linha da outbox ficar agendada por muito tempo, exigindo campo `nextAttemptAt` indexado para não degradar o polling.
- Duas tabelas para manter em sincronia conceitual (outbox e dead letter), com o replay tendo que reconstruir a linha na outbox.

