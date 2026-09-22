# ADR-004 — Autenticação por HMAC-SHA256 com secret por endpoint e rotação com grace period

- **Status:** Aceito
- **Data:** 2026-09-22
- **Decisores:** Sofia (Eng. Segurança), Larissa (Tech Lead)
- **Consultados:** Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos)
- **Decisão relacionada a:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

## Contexto

Estamos enviando dados de pedidos para endpoints HTTP fora da nossa infraestrutura. O cliente precisa conseguir provar duas coisas ao receber uma requisição: que ela **veio realmente de nós** e que o corpo **não foi adulterado no caminho**.

> [09:19] Sofia: a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio.

Há também precedente interno de vazamento de credencial pelo lado do cliente:

> [09:22] Diego: A gente já teve cliente que vazou secret em log de aplicação dele uma vez.

## Decisão

1. **HMAC-SHA256 sobre o corpo do request**, com a assinatura enviada no header `X-Signature`.

> [09:20] Sofia: Padrão é HMAC. A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature.

> [09:20] Sofia: SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso.

2. **Uma secret única por endpoint cadastrado**, não uma secret global da plataforma. A secret é gerada por nós e devolvida uma única vez, na resposta de criação do webhook ([09:31] Marcos).

> [09:21] Sofia: cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo.

3. **Rotação de secret via API, com grace period de 24 horas.** Durante a janela, as duas secrets são válidas em paralelo; depois, a antiga morre.

> [09:21] Sofia: a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre.

> [09:22] Sofia: Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h.

4. **HTTPS obrigatório na URL cadastrada.** URL `http` é recusada com erro de validação, implementado como refinamento de schema Zod em `src/modules/webhooks/webhook.schemas.ts` _(arquivo novo)_, no mesmo padrão de `src/modules/orders/order.schemas.ts` _(existente)_ executado pelo `validate` de `src/middlewares/validate.middleware.ts`.

> [09:23] Sofia: TLS obrigatório. URL do webhook tem que ser https. Se o cliente cadastrar http, recusamos com erro de validação. Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod.

Durante o grace period, o envio carrega **as duas assinaturas** — a da secret nova e a da antiga — de modo que o cliente valide com qualquer uma das duas enquanto migra.

## Alternativas Consideradas

### 1. Secret global da plataforma, compartilhada entre todos os endpoints

- **Trade-off que motivou o descarte:** raio de explosão total. Um único vazamento comprometeria a autenticidade de todos os envios para todos os clientes ([09:21] Sofia). O custo de armazenar uma secret por endpoint é desprezível perto disso.

### 2. Token estático em header (Bearer / API key do cliente)

- **Trade-off que motivou o descarte:** prova identidade, mas não prova **integridade do payload** — um intermediário poderia alterar o corpo mantendo o token válido. O requisito explícito da Sofia cobre os dois eixos, "veio da gente" e "não adulteraram" ([09:19] Sofia). HMAC resolve ambos com o mesmo mecanismo.

### 3. Rotação imediata, sem grace period

- **Trade-off que motivou o descarte:** quebraria a integração do cliente no instante da rotação, transformando um procedimento de higiene de segurança em incidente. As 24 horas existem exatamente para o cliente migrar os sistemas dele ([09:21] Sofia).

### 4. Assinatura assimétrica (RSA / Ed25519)

- **Trade-off que motivou o descarte:** dispensaria secret compartilhada, mas HMAC-SHA256 foi escolhido por ser o padrão de mercado com suporte universal de bibliotecas do lado do cliente ([09:20] Sofia), e o modelo de ameaça aqui não exige não-repúdio.

## Consequências

### Positivas

- Autenticidade e integridade garantidas com um mecanismo padrão de mercado, barato de verificar e trivial de implementar do lado do cliente.
- Vazamento de uma secret compromete apenas um endpoint de um cliente.
- Rotação é uma operação segura e auto-atendida, sem downtime da integração.
- HTTPS obrigatório protege confidencialidade dos dados do pedido em trânsito.

### Negativas

- A secret precisa ser armazenada de forma recuperável (não pode ser hash, pois precisamos assiná-la a cada envio), o que exige cuidado de acesso ao banco e redação nos logs. O `redact` de `src/shared/logger/index.ts` cobre `*.token` e `*.password`, e precisará de path adicional para o campo de secret de webhook.
- O grace period de 24h implica manter **duas secrets ativas** por endpoint no schema, com campo de expiração da antiga, e calcular duas assinaturas por envio durante a janela.
- A secret só é exibida na criação e na rotação; cliente que a perder precisa rotacionar de novo.
- O cliente é responsável por implementar a verificação corretamente (comparação em tempo constante, por exemplo). Cabe ao portal de desenvolvedor documentar isso ([09:26] Marcos).

