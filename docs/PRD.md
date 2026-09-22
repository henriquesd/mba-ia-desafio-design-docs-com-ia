# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado para implementação |
| **Data** | 2026-09-22 |
| **Origem** | Reunião técnica de quinta-feira, 09:00–09:53 ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto

O OMS passa a notificar ativamente clientes B2B sempre que o status de um pedido deles muda, entregando o evento por HTTP no endpoint que o próprio cliente cadastra. Hoje esses clientes descobrem mudanças fazendo requisições repetidas a `GET /orders`; com a feature, passam a receber a informação sem perguntar.

A demanda chegou como pedido formal de três clientes na semana anterior à reunião, e tem pressão comercial explícita associada.

## 2. Problema e motivação

### O problema

> [09:00] Marcos: Hoje eles ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles.

O polling impõe custo aos dois lados: o cliente gasta infraestrutura perguntando e ainda descobre a mudança com atraso; nós servimos requisições que, na maioria das vezes, não trazem novidade.

### A motivação de negócio

O pedido é formal e vem com risco de receita associado:

> [09:00] Marcos: A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente.

### A lacuna técnica

O OMS controla rigorosamente o ciclo de vida do pedido — máquina de estados, controle transacional de estoque e auditoria de mudanças de status — mas **não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks**. A feature preenche exatamente essa lacuna.

## 3. Público-alvo e cenários de uso

### Público-alvo

| Persona | Quem é | O que precisa |
|---|---|---|
| **Cliente B2B integrador** | Atlas Comercial, MaxDistribuição, Nova Cargo ([09:00] Marcos) | Saber, em menos de 10 segundos, que o status de um pedido dele mudou |
| **Desenvolvedor do cliente** | Time técnico que integra com a nossa API | Cadastrar endpoint, validar autenticidade da requisição, depurar entregas que falharam |
| **Operador interno** | Usuário com role `OPERATOR` no OMS | Gerenciar a configuração de webhooks dos clientes |
| **Administrador da plataforma** | Usuário com role `ADMIN` | Reprocessar eventos que falharam definitivamente |

### Cenários de uso

**C1 — Acompanhamento de expedição.** A Atlas cadastra um endpoint assinando apenas `SHIPPED` e `DELIVERED`. Quando um pedido dela é despachado, o sistema dela recebe o evento em poucos segundos e dispara a comunicação com o cliente final, sem nenhum polling ([09:33] Marcos).

**C2 — Cliente temporariamente indisponível.** A MaxDistribuição entra em manutenção planejada de duas horas. Os eventos gerados no período são retentados com espaçamento crescente e entregues quando o endpoint volta, sem perda e sem intervenção ([09:16] Diego).

**C3 — Depuração de integração.** O desenvolvedor da Nova Cargo suspeita que não recebeu um evento. Ele consulta o histórico de entregas do endpoint dele e vê tentativa, status de resposta, corpo e tempo de resposta ([09:34] Marcos).

**C4 — Rotação de credencial.** A Atlas suspeita que a secret vazou em log. Pede rotação pela API, recebe a nova secret e tem 24 horas para migrar os sistemas dela enquanto as duas continuam válidas ([09:21] Sofia).

**C5 — Recuperação administrativa.** Um endpoint ficou fora do ar por mais de 15 horas e eventos caíram na DLQ. Depois de o cliente confirmar que voltou, um `ADMIN` reprocessa manualmente cada evento ([09:18] Diego).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
|---|---|---|---|
| OBJ-1 | Entregar a notificação em tempo percebido como real pelo cliente | Latência p95 entre commit da mudança de status e entrega bem-sucedida | **< 10 segundos** ([09:02] Marcos) |
| OBJ-2 | Eliminar a necessidade de polling dos clientes integrados | Redução do volume de `GET /orders` dos três clientes | **Queda de pelo menos 80%** em 30 dias após a adoção |
| OBJ-3 | Não perder eventos | Eventos gerados que terminam entregues ou visíveis na DLQ | **100%** — nenhum evento desaparece |
| OBJ-4 | Não degradar o fluxo crítico de pedidos | Variação da latência p95 de `PATCH /orders/:id/status` | **Aumento inferior a 5%** após o deploy |
| OBJ-5 | Entregar no prazo comercial acordado | Data de disponibilização em produção | **Fim de novembro**, em três sprints ([09:45]–[09:46] Marcos e Larissa) |
| OBJ-6 | Reter a conta em risco | Renovação da Atlas Comercial | **Contrato renovado** no trimestre |

## 5. Escopo

### 5.1 Incluído

- Cadastro, listagem, edição e remoção de endpoints de webhook por cliente.
- Assinatura seletiva de status: o cliente escolhe quais transições quer receber.
- Geração e rotação de secret com período de convivência de 24 horas.
- Entrega assinada de eventos de mudança de status de pedido.
- Retentativas automáticas em caso de falha e fila de eventos mortos.
- Consulta do histórico de entregas por endpoint.
- Reprocessamento manual de eventos mortos, restrito a administradores.

### 5.2 Fora de escopo

Itens descartados ou adiados explicitamente durante a reunião:

| Item | Decisão | Origem |
|---|---|---|
| **Notificação por e-mail ao cliente** quando o webhook dele falha repetidamente | Adiado para a próxima fase, depois de medir o impacto | [09:37] Larissa: "Email tá fora de escopo dessa fase" |
| **Dashboard visual** para o cliente acompanhar os webhooks dele | Descartado nesta fase; é projeto do time de frontend. Nesta entrega, só endpoints | [09:40] Larissa: "Não, agora não. Só endpoints" |
| **Rate limiting de saída** por cliente | Fora do escopo; observar em produção e decidir depois | [09:39] Diego e Larissa |
| **Webhooks inbound** (cliente enviando eventos para nós) | Fora de escopo: o fluxo é apenas outbound | [09:02] Marcos: "Só saindo da gente pra eles" |
| **Arquivamento das linhas entregues** | Intenção de arquivar após ~30 dias, mas fora desta feature | [09:08] Diego |
| **Eventos além de mudança de status de pedido** | Nenhum outro tipo de evento foi pedido | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| **RF-01** | O cliente deve poder **cadastrar um endpoint de webhook**, informando a URL de destino e o `customerId`. A secret é gerada pela plataforma e devolvida na resposta da criação | [09:31] Marcos |
| **RF-02** | O cliente deve poder **escolher quais status quer receber** por endpoint. Se nenhum webhook do customer assina aquele status, o evento não é gerado | [09:33] Marcos, [09:34] Bruno |
| **RF-03** | O cliente deve poder **editar** um endpoint já cadastrado (URL, status assinados, estado ativo) | [09:33] Bruno |
| **RF-04** | O cliente deve poder **remover** um endpoint cadastrado | [09:33] Bruno |
| **RF-05** | O cliente deve poder **listar os endpoints de webhook de um customer** | [09:33] Bruno |
| **RF-06** | O cliente deve poder **rotacionar a secret** de um endpoint pela API, com a secret anterior permanecendo válida por 24 horas | [09:21] Sofia |
| **RF-07** | O cliente deve poder **consultar o histórico de entregas** de um endpoint, vendo sucesso ou falha, payload enviado, resposta recebida e tempo de resposta | [09:34] Marcos |
| **RF-08** | O sistema deve **entregar um evento sempre que o status de um pedido mudar** para um status assinado por algum endpoint ativo do customer | [09:00] Marcos, [09:06] Diego |
| **RF-09** | Cada entrega deve ser **assinada com HMAC-SHA256** e acompanhada dos headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` | [09:20] Sofia, [09:44] Diego e Sofia |
| **RF-10** | O sistema deve **retentar automaticamente** entregas que falharem, até 5 vezes, com intervalos de 1m, 5m, 30m, 2h e 12h | [09:17] Diego e Larissa |
| **RF-11** | Eventos que falharem nas 5 tentativas devem ser **movidos para uma fila de eventos mortos (DLQ)**, preservando payload, motivo da falha e timestamp | [09:18] Diego |
| **RF-12** | Um usuário com role **`ADMIN`** deve poder **reprocessar manualmente** um evento da DLQ, recolocando-o na fila de envio. O replay é registrado para auditoria | [09:18] e [09:35] Diego, [09:36] Sofia |
| **RF-13** | O sistema deve **recusar o cadastro de URL que não seja `https`** | [09:23] Sofia |
| **RF-14** | Todos os endpoints de gerenciamento devem ser **autenticados**; apenas o reprocessamento de DLQ exige role `ADMIN` | [09:32] Marcos, [09:36] Sofia e Larissa, [09:37] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| **RNF-01** | Latência de entrega abaixo de 10 segundos no caminho feliz | [09:02] Marcos |
| **RNF-02** | A geração do evento deve ser **atômica** com a mudança de status: se uma falha, nenhuma acontece | [09:06] Diego, [09:40] Bruno |
| **RNF-03** | A indisponibilidade de um cliente **não pode** bloquear nem reverter a mudança de status de um pedido | [09:04] Bruno |
| **RNF-04** | Garantia de entrega **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e deve deduplicar pelo `X-Event-Id` | [09:24]–[09:26] Diego e Larissa |
| **RNF-05** | Cada endpoint tem **secret própria**; não existe secret global da plataforma | [09:21] Sofia |
| **RNF-06** | O payload de um evento é limitado a **64KB**; acima disso o envio falha em vez de truncar | [09:23] Sofia, [09:24] Diego e Larissa |
| **RNF-07** | Timeout de **10 segundos** por tentativa de entrega | [09:42] Diego |
| **RNF-08** | O processamento roda em **processo separado** da API, com ciclo de vida independente | [09:11] Diego |
| **RNF-09** | Ordem de entrega garantida apenas **por pedido** e enquanto houver um único worker; não há ordering global | [09:12]–[09:13] Diego e Larissa |
| **RNF-10** | A feature **não introduz infraestrutura nova**: mesmo MySQL, mesmo Prisma, mesma stack | [09:07] Diego, [09:30] Larissa |
| **RNF-11** | Todos os códigos de erro do módulo usam o prefixo **`WEBHOOK_`**, seguindo o padrão existente de `AppError` | [09:28] Bruno, [09:29] Larissa |

## 8. Decisões e trade-offs principais

Detalhamento completo nos [ADRs](adrs/); resumo do que foi decidido e do preço pago:

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Padrão Outbox no MySQL existente | Ganha atomicidade e zero infra nova; paga com tabela que cresce e precisa de arquivamento futuro | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) |
| Worker separado com polling de 2 segundos | Ganha isolamento de falhas e simplicidade; paga com até 2 segundos de latência e consultas ociosas ao banco | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 tentativas com backoff até 12h, depois DLQ | Ganha tolerância a indisponibilidade longa; paga com evento podendo chegar até cerca de 15h depois | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint | Ganha raio de explosão mínimo em vazamento; paga com armazenamento recuperável da secret e duas assinaturas durante a rotação | [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega at-least-once com `X-Event-Id` | Ganha simplicidade e alinhamento com o mercado; paga transferindo a deduplicação ao cliente | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso integral dos padrões do projeto | Ganha velocidade e revisão menor; paga herdando as limitações do padrão atual | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Snapshot do payload na inserção | Ganha fidelidade ao momento do evento; paga com duplicação de dados e payload possivelmente desatualizado na entrega | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-outbox.md) |

## 9. Dependências

| Dependência | Natureza | Observação |
|---|---|---|
| Módulo de pedidos (`changeStatus`) | Interna, bloqueante | É o gatilho de todo evento; a feature depende de estender a transação existente ([09:40] Bruno) |
| Banco MySQL via Prisma | Interna, bloqueante | Outbox e DLQ vivem no mesmo banco ([09:07] Diego) |
| Revisão de segurança da Sofia | Interna, bloqueante para o deploy | 2 dias úteis reservados antes de subir ([09:46] Sofia) |
| Endpoint HTTPS do cliente | Externa | Fora do nosso controle; a disponibilidade dele define a taxa de retry |
| Documentação no portal de desenvolvedor | Externa ao time de engenharia | Marcos documenta a obrigação de deduplicar e o modelo de integração ([09:26] e [09:40] Marcos) |
| Capacidade do time | Organizacional | Três sprints estimados, incluindo a revisão de segurança ([09:46] Larissa) |

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| **RI-1** | Regressão no fluxo de mudança de status, o caminho mais crítico do OMS | Média | Alto | Extensão mínima e isolada via função que recebe o `tx` ([09:41] Diego); testes de integração cobrindo rollback; medição de latência antes e depois (OBJ-4) |
| **RI-2** | Falha de segurança na assinatura ou na geração e guarda da secret | Média | Alto | Secret por endpoint ([09:21] Sofia); revisão de segurança bloqueante com 2 dias úteis reservados ([09:46] Sofia); redação da secret nos logs |
| **RI-3** | Cliente não implementar deduplicação e processar o mesmo evento duas vezes | Alta | Médio | Documentação destacada no portal de desenvolvedor ([09:26] Marcos); `event_id` no header e no corpo |
| **RI-4** | Atraso na entrega além do fim de novembro, com risco de perder a Atlas | Média | Alto | Escopo já cortado (sem e-mail, sem dashboard, sem rate limiting) e fatiado em três sprints ([09:46] Larissa); acompanhamento semanal do PM |
| **RI-5** | Rajada de mudanças de status sobrecarregar o endpoint do cliente | Média | Médio | Lotes pequenos no worker; observação em produção antes de implementar rate limiting ([09:39] Diego) |
| **RI-6** | Worker parar silenciosamente e eventos deixarem de sair sem ninguém notar | Média | Alto | Alerta sobre o atraso da fila e sobre entradas na DLQ; processo supervisionado com reinício automático |

## 11. Critérios de aceitação

| # | Critério |
|---|---|
| **AC-1** | Um cliente consegue cadastrar, listar, editar e remover endpoints de webhook pela API, autenticado |
| **AC-2** | A secret é devolvida na criação e na rotação, e em nenhum outro lugar |
| **AC-3** | Cadastro com URL `http://` é recusado com erro de validação |
| **AC-4** | Mudança de status de um pedido gera entrega apenas para endpoints ativos do customer que assinam aquele status |
| **AC-5** | A entrega chega ao endpoint do cliente em menos de 10 segundos no p95 |
| **AC-6** | A requisição entregue traz `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`, e a assinatura valida com a secret do endpoint |
| **AC-7** | Endpoint indisponível recebe 5 tentativas nos intervalos definidos antes de o evento ir para a DLQ |
| **AC-8** | Evento na DLQ preserva payload e motivo da falha e pode ser reprocessado apenas por `ADMIN`, com registro de quem reprocessou |
| **AC-9** | Rotação de secret mantém a anterior válida por 24 horas e a invalida depois |
| **AC-10** | O histórico de entregas mostra tentativa, sucesso ou falha, status da resposta e tempo de resposta |
| **AC-11** | Falha na geração do evento reverte a mudança de status por completo |
| **AC-12** | Nenhum evento é perdido: todo evento gerado termina entregue ou visível na DLQ |

## 12. Estratégia de testes e validação

### Antes do deploy

- Testes automatizados nos três níveis descritos no [FDD](FDD.md): unitário (backoff, montagem do payload, HMAC, validação de schema), integração (atomicidade e rollback em `changeStatus`) e ponta a ponta (ciclo completo de entrega, retry e DLQ contra um endpoint local simulado).
- Revisão de segurança da Sofia sobre HMAC, geração e guarda de secret, com 2 dias úteis reservados ([09:46] Sofia).
- Medição da latência de `PATCH /orders/:id/status` antes e depois do deploy, para validar OBJ-4.

### Validação com cliente

- Piloto com um dos três clientes, preferencialmente a Atlas, que é a conta com pressão de prazo.
- Acompanhamento de OBJ-1 (latência p95) e OBJ-2 (queda do volume de `GET /orders`) nos 30 dias seguintes à adoção.
- Monitoramento de entradas na DLQ como sinal precoce de integração quebrada do lado do cliente.

### Critério de sucesso da fase

Os três clientes integrados, latência p95 abaixo de 10 segundos, nenhum evento perdido e nenhuma degradação mensurável no fluxo de pedidos. Só depois disso a fase seguinte — notificação por e-mail e rate limiting de saída — entra em discussão ([09:37] e [09:39] Larissa).

