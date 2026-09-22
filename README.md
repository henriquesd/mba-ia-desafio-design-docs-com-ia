# Da Reunião ao Documento — Pacote de Design Docs gerado com IA

Entrega do desafio técnico **"Da Reunião ao Documento: Design Docs Gerados por IA"** do MBA em Engenharia de Software com IA.

O repositório base ([devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)) contém um Order Management System em Node.js + TypeScript e a transcrição de uma reunião técnica. Este fork acrescenta o pacote de documentação técnica produzido a partir desses dois insumos. **Nenhuma linha de código da aplicação foi alterada.**

---

## Sobre o desafio

A tarefa foi transformar a transcrição literal de uma reunião de 55 minutos ([`TRANSCRICAO.md`](TRANSCRICAO.md)) em um pacote de design docs acionável o suficiente para um time começar a implementar. A feature discutida é um **Sistema de Webhooks de Notificação de Pedidos**: clientes B2B querem ser avisados quando o status dos pedidos deles muda, em vez de ficar fazendo polling na API.

O trabalho não foi "pedir para a IA gerar documentos". Foi definir o recorte de cada documento, dirigir a IA com prompts específicos e, principalmente, **revisar criticamente** o que voltou — inclusive a saída da própria IA usada para auditar o resultado, que produziu uma boa quantidade de falso positivo.

O ponto mais delicado é a fronteira entre o que **entra** e o que **não entra**. A reunião descarta ideias explicitamente (dashboard, e-mail de alerta, rate limiting), e uma IA desatenta transforma qualquer coisa mencionada em requisito.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (Opus)** no terminal | Produção. Leu a transcrição e o código-fonte direto do repositório e escreveu os 12 documentos |
| **Google Gemini** via LangChain (`langchain-google-genai`) | Auditoria independente. Rodou os 3 prompts customizados contra o pacote já pronto, como segundo par de olhos |
| **Script Python de verificação**, escrito pelo Claude Code | Checagem determinística: 63 asserções contra os critérios de aceite do enunciado |

O diferencial de usar um agente com acesso ao repositório é que a seção "Integração com o sistema existente" do FDD pôde ser escrita a partir do código real — `OrderService.changeStatus`, as classes de `src/shared/errors/`, o `requireRole` do middleware de autenticação — em vez de suposições sobre como o projeto provavelmente é.

Usar um modelo **diferente** para auditar foi decisão consciente: pedir para a mesma IA revisar o que ela acabou de escrever tende a produzir concordância, não revisão.

## Workflow adotado

A ordem seguiu a lógica de "decisão primeiro, documento depois":

1. **Contextualização.** Leitura completa da transcrição e dos arquivos-chave: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/shared/errors/`, `prisma/schema.prisma`, `src/middlewares/`, `src/app.ts`, `src/routes/index.ts`, `package.json`.
2. **Extração das decisões.** Mapeamento das decisões fechadas, alternativas descartadas, itens adiados e ganchos com o código — cada um com timestamp e falante.
3. **ADRs (7 arquivos).** As decisões viraram o esqueleto. Escrever ADR primeiro força a explicitar o trade-off antes de descrever a solução.
4. **RFC.** Proposta em nível de arquitetura, referenciando os ADRs já escritos. Curto por design.
5. **FDD.** O documento mais longo: modelagem, fluxos, contratos HTTP, matriz de erros, observabilidade e integração com o código real.
6. **PRD.** Escrito por último entre os grandes, virando quase uma consolidação em nível de produto.
7. **Tracker.** Varredura dos documentos prontos, item por item, atribuindo origem a cada um.
8. **Verificação determinística.** Script conferindo critérios do enunciado, IDs duplicados, caminhos inexistentes e timestamps inválidos.
9. **Auditoria por IA independente.** Os 3 prompts rodados no Gemini contra o pacote pronto.
10. **Triagem da auditoria.** Separar achado real de falso positivo.
11. **README.** Por último, com o processo já completo.

### Fronteira entre os documentos

Para evitar duplicação, cada documento ficou preso à sua altura:

| Documento | Responde | Não faz |
|---|---|---|
| PRD | Por que e o quê | Não descreve solução técnica |
| RFC | Como pretendemos resolver, o que está em aberto | Não desce a payload, header ou schema |
| ADR | Por que decidimos exatamente assim | Não descreve implementação |
| FDD | Como construir, em detalhe | Não rediscute decisão já fechada |
| Tracker | De onde veio cada coisa | Não acrescenta conteúdo novo |

## Prompts customizados

Os três prompts abaixo foram escritos para este desafio e **rodados de fato** contra o pacote pronto, via API do Gemini, como auditoria independente.

### Prompt 1 — Extração dirigida, separando o que entra do que não entra

O prompt genérico ("extraia os requisitos da transcrição") produz uma lista achatada em que ideia descartada e decisão fechada aparecem lado a lado. Este força a classificação e exige prova literal:

```
Leia a transcricao abaixo inteira. Classifique CADA ponto tecnico discutido em
exatamente uma destas categorias:

  (A) DECISAO FECHADA   (B) REQUISITO   (C) DESCARTADO na reuniao
  (D) ADIADO            (E) EM ABERTO   (F) RUIDO

Para cada item: categoria | timestamp [hh:mm] | falante | frase LITERAL de apoio | resumo.
Se voce nao consegue colar a frase literal, o item NAO existe. Nao inclua.
Nao deduza requisito a partir de "como webhooks normalmente funcionam".

Depois, escreva a secao "DIVERGENCIAS" comparando com o PRD entregue:
(1) itens que o PRD trata como requisito mas voce classificou como C ou D;
(2) itens A ou B que o PRD nao cobre.
Se nao houver divergencia, escreva "nenhuma".
```

### Prompt 2 — Ancoragem da integração no código real

A seção "Integração com o sistema existente" é onde a IA mais alucina, inventando caminhos de arquivo plausíveis. Este prompt entrega o código de verdade e proíbe inferência:

```
Abaixo estao arquivos REAIS do repositorio e a secao "Integracao com o sistema
existente" do FDD entregue. Audite a secao. Para cada afirmacao sobre o codigo:
  - o arquivo citado aparece no codigo abaixo?
  - o metodo/classe/funcao citado existe de fato ali?
  - a descricao do comportamento atual esta correta?

Observacao: caminhos marcados com "(novo)" ou "(arquivo novo)" sao arquivos que a
feature vai CRIAR. Nao os trate como erro; ignore-os.

Liste SOMENTE os problemas. Se nao houver, escreva "nenhum problema encontrado".
```

### Prompt 3 — Auditoria do tracker contra alucinação

```
Para CADA linha da tabela, responda SIM ou NAO:
- Fonte = TRANSCRICAO: a frase que sustenta o item existe no timestamp indicado,
  dita pela pessoa indicada, na transcricao abaixo?
- Fonte = CODIGO: o arquivo indicado existe na lista de arquivos abaixo?

Liste SO as linhas que receberam NAO, dizendo se devem ser CORRIGIDAS (indique a
origem certa) ou REMOVIDAS. Nao justifique as que passaram.
Se todas passaram, escreva "todas as linhas deste lote passaram".
```

## Iterações e ajustes

### 1. A primeira extração ia transformar item descartado em requisito

A reunião registra três pedidos **recusados na hora**: alerta por e-mail quando o webhook falha ([09:37] Larissa), dashboard visual para o cliente ([09:40] Larissa) e rate limiting de saída ([09:39] Diego). São exatamente o tipo de coisa que vira requisito quando se lê a transcrição procurando pedidos em vez de decisões.

A correção foi o Prompt 1, com as categorias C (descartado) e D (adiado) **obrigatórias**. O resultado virou a seção "Fora de escopo" do PRD, um dos pedaços mais valiosos do pacote.

### 2. Revisão contra o enunciado achou três furos

Com o pacote pronto, uma passada critério por critério sobre o enunciado encontrou:

- **`PATCH /webhooks/:id` sem exemplo de resposta.** Tinha o request, faltava o response. Adicionado em `docs/FDD.md` §6.3.
- **`docs/adrs/README.md` contradizia a própria entrega.** Era o arquivo do repositório base, mandando nomear ADRs como `0001-titulo.md`, enquanto os entregues usam `ADR-001-titulo.md`. Reescrito como índice dos 7 ADRs.
- **Arquivos novos pareciam alucinação.** Os documentos citam `src/worker.ts`, `webhook.publisher.ts`, `webhook.worker.ts` e `webhook.schemas.ts`, que não existem hoje porque são o que a feature vai criar. Um revisor leria como arquivo inventado. Todos ganharam a marca _(novo)_, e o FDD ganhou uma nota de convenção no início da §11.

### 3. O script de auditoria mentia sobre o próprio resultado

A primeira execução dos prompts no Gemini imprimiu `ok` nas 6 chamadas — e o relatório saiu com 6 blocos de erro HTTP 429. O script capturava a exceção e imprimia sucesso do mesmo jeito. Bug no código de auditoria, não nos documentos.

Ficou a lição: **ferramenta de verificação também precisa ser verificada.** O script foi reescrito para imprimir `OK - N chars` ou `FALHOU`, e para listar ao final quantas chamadas falharam.

### 4. Cota da API forçou fallback entre modelos

O Gemini free tem limite **por modelo**: 5 requisições por minuto e 20 por dia. O modelo configurado (`gemini-3.7-flash`) esgotou a cota diária, e duas alternativas óbvias (`gemini-2.5-flash`, `gemini-2.0-flash`) não existiam naquela conta.

O script passou a tentar uma cadeia de modelos, caindo para o próximo quando a cota diária estoura ou o modelo não existe. A auditoria acabou rodando no `gemini-2.5-flash-lite`.

### 5. A auditoria por IA produziu mais falso positivo que achado

A iteração mais instrutiva. O relatório do Gemini precisou ser triado item por item:

| O que o Gemini apontou | Veredito |
|---|---|
| "Nenhum item A ou B da reunião ficou fora do PRD" | **Achado válido.** Confirma cobertura |
| 8 "divergências" entre PRD e transcrição | **Descartado.** Todas eram discussão de rótulo: "o PRD chama de requisito não funcional, a transcrição é uma decisão fechada". Nenhuma era ideia descartada virando requisito |
| 10 linhas do tracker "a corrigir" | **Descartado.** Leu a pergunta errada: conferiu se o arquivo *contém uma frase*, em vez de se o arquivo *existe*. Os 10 arquivos existem |
| Lote 2 do tracker, 78 linhas | **Descartado.** Devolveu as linhas sem veredito nenhum |
| Arquivos `(novo)` "não encontrados" no código | **Descartado.** O próprio prompt mandava ignorá-los |

Ou seja: **zero defeito real encontrado pela auditoria por IA**, e quatro categorias de ruído que precisaram ser reconhecidas e descartadas. O modelo leve não deu conta da tarefa.

A conclusão prática: para este tipo de verificação, o **script determinístico vale mais que o LLM**. Conferir se um caminho existe no filesystem e se uma string aparece na transcrição é trabalho de `os.path.exists()` e de busca literal, não de modelo de linguagem. O LLM ficou útil só para a pergunta genuinamente semântica — "ficou algo de fora?".

### Verificação final

O script determinístico roda 63 asserções: seções obrigatórias de cada documento, contagens mínimas do enunciado, IDs duplicados no tracker, caminhos inexistentes, timestamps inválidos e `git status` de `src/`, `prisma/` e `tests/`.

```
63 OK   |   0 FALHA
```

No tracker: **158 linhas**, 0 IDs duplicados, 0 caminhos quebrados, 0 timestamps inventados, 80% com fonte na transcrição e 31 linhas apontando para código real.

## Como navegar a entrega

Ordem de leitura sugerida:

| # | Arquivo | O que é |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que a feature existe, para quem, o que entra e o que fica de fora |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica em nível de arquitetura, alternativas descartadas e questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | As 7 decisões arquiteturais, uma por arquivo, com trade-off explícito |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: modelagem, fluxos, contratos HTTP, erros, observabilidade e integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item: timestamp da transcrição ou caminho de arquivo |
| — | [`TRANSCRICAO.md`](TRANSCRICAO.md) | A fonte primária (não alterada) |

### Estrutura dos arquivos entregues

```
.
├── README.md                 (este arquivo)
├── TRANSCRICAO.md            (não alterado)
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── README.md         (índice)
        ├── ADR-001-padrao-outbox-no-mysql.md
        ├── ADR-002-worker-separado-em-polling.md
        ├── ADR-003-retry-com-backoff-exponencial-e-dlq.md
        ├── ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-entrega-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
        └── ADR-007-snapshot-do-payload-na-outbox.md
```

Os diretórios `src/`, `prisma/` e `tests/` permanecem exatamente como no repositório base.
