# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos**, produzido a partir da transcrição de uma reunião técnica de ~55 minutos e do código de um Order Management System em produção.

---

## Sobre o desafio

O ponto de partida era uma decisão já tomada e nada registrado: cinco pessoas (Tech Lead, PM, dois engenheiros e uma engenheira de segurança) discutiram por ~55 minutos como construir uma feature de webhooks, e o único vestígio dessa conversa era a transcrição literal da call. Minha tarefa foi transformar isso em um pacote de documentação técnica acionável — PRD, RFC, FDD, ADRs e um tracker de rastreabilidade — usando IA como ferramenta principal de produção.

O trabalho real não foi gerar texto: foi **decidir o que entra e o que fica de fora**. A transcrição contém decisões fechadas, requisitos explícitos, alternativas descartadas, itens adiados para fases futuras e detalhes secundários misturados na mesma conversa. Separar essas categorias, garantir que cada afirmação nos documentos tivesse origem identificável e impedir que a IA "completasse" lacunas com invenções foi o que consumiu a maior parte do esforço.

A restrição era absoluta: **entrega puramente documental**. Nenhum arquivo em `src/`, `prisma/`, `tests/` ou de configuração foi alterado — o código serviu apenas como contexto e referência.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **Claude Code** (CLI, modo agente) | Ferramenta principal. Leu o repositório inteiro diretamente (sem colar trechos), mapeou os padrões da codebase, extraiu e classificou os itens da transcrição, e gerou os documentos. O acesso direto ao sistema de arquivos foi decisivo: permitiu verificar caminhos, assinaturas de método e configurações reais em vez de confiar em descrições. |
| **Exploração via subagente** | Varredura paralela do repositório para mapear estrutura de módulos, taxonomia de erros, middlewares, schema Prisma e padrões de teste, enquanto eu lia os arquivos críticos (`order.service.ts`, `schema.prisma`, `error.middleware.ts`) em detalhe. |
| **Web search / documentação externa** | Não utilizado. Todas as informações técnicas dos documentos vêm da transcrição ou do código — nenhuma fonte externa foi necessária, e usar uma teria introduzido conteúdo sem origem rastreável. |

Não usei geração de imagens, transcrição de áudio nem outras ferramentas. A escolha foi deliberada: com uma única ferramenta com acesso ao repositório, o risco de inconsistência entre documentos cai, e a verificação de cada afirmação contra o código fica direta.

---

## Workflow adotado

Segui a ordem sugerida no enunciado, porque ela respeita a dependência natural entre os documentos: **as decisões são o esqueleto do resto**.

```
1. Contextualização
   └─ Leitura da transcrição completa (323 linhas)
   └─ Mapeamento do código: estrutura, padrões, pontos de integração
                    │
2. Extração estruturada
   └─ Classificar cada fala em: decisão fechada | requisito | restrição |
      alternativa descartada | item adiado | detalhe secundário
                    │
3. ADRs (8)  ──────────► o esqueleto: cada decisão isolada com seu trade-off
                    │
4. RFC  ───────────────► consolida a proposta em cima das decisões;
                          alternativas descartadas e questões em aberto
                    │
5. FDD  ───────────────► detalha o "como construir" em cima do RFC + ADRs
                    │
6. PRD  ───────────────► altura de produto; com o resto pronto, é consolidação
                    │
7. TRACKER  ───────────► varredura dos documentos prontos, item por item
                    │
8. README  ────────────► documentação do processo
                    │
9. Revisão final  ─────► checklist de critérios de aceite item por item
```

**Por que ADRs primeiro.** O enunciado lista seis decisões principais. Escrevê-las antes de qualquer outro documento forçou o esclarecimento de cada trade-off isoladamente — e revelou que duas delas, embora discutidas juntas na reunião, tinham alternativas e consequências distintas: a *política de retry* ([ADR-003](docs/adrs/ADR-003-retry-com-backoff-exponencial.md)) e o *destino dos eventos que esgotam as tentativas* ([ADR-007](docs/adrs/ADR-007-dlq-em-tabela-separada.md)). Separá-las produziu dois ADRs melhores do que um ADR genérico sobre "retry e DLQ".

**Como organizei a interação com a IA.** Em vez de pedir documentos prontos, usei prompts de **extração com formato de saída definido** e depois prompts de **geração com restrição de origem**. Toda afirmação gerada precisava apontar para um timestamp `[hh:mm] Falante` ou um caminho de arquivo. Quando um item não tinha origem, ele saía — e foi isso que produziu a lista de exclusões explícitas do PRD.

**Fronteira entre documentos.** O maior risco era produzir três documentos que dissessem a mesma coisa com palavras diferentes. A regra aplicada: o RFC **decide** e abre questões; os ADRs **justificam** cada decisão isolada; o FDD **especifica**. Concretamente, o RFC não contém payload de endpoint nem matriz de erros — esses detalhes existem só no FDD, e o RFC aponta para lá.

---

## Prompts customizados

Três prompts foram centrais. Todos partem de uma ideia: **restringir a saída a conteúdo com origem verificável**.

### Prompt 1 — Extração classificada da transcrição

Usado antes de qualquer documento. O ponto crítico é a classificação explícita do que **não** entra.

```text
Leia TRANSCRICAO.md integralmente. Produza uma tabela com TODAS as
falas que carregam informação técnica ou de produto, classificando cada
uma em exatamente uma destas categorias:

- DECISAO_FECHADA    → algo que o grupo explicitamente fechou
- REQUISITO          → comportamento que o sistema deve ter
- RESTRICAO          → limite ou regra que condiciona a solução
- ALTERNATIVA_DESCARTADA → opção levantada e rejeitada, COM o motivo
- ADIADO             → mencionado e deixado para depois
- FORA_DE_ESCOPO     → explicitamente excluído desta fase
- QUESTAO_ABERTA     → levantado e não decidido
- CONTEXTO           → informação de negócio (clientes, prazos, motivação)
- DETALHE            → parâmetro técnico secundário (timeout, header, campo)

Colunas: timestamp | falante | categoria | conteúdo (1 linha) | citação literal

REGRAS:
1. Não infira categoria pelo tom. Use o que foi dito e o que o grupo
   confirmou depois. Se alguém propôs algo e outro refutou, é
   ALTERNATIVA_DESCARTADA, não REQUISITO.
2. Se um item foi mencionado e depois classificado pelo grupo como
   secundário ("não vejo como decisão arquitetural separada"), registre
   a classificação do grupo, não a sua.
3. Citação literal é obrigatória. Sem citação, o item não entra.
4. Não agrupe falas. Uma linha por item identificável.
```

**O que esse prompt evitou.** Sem a classificação, a IA trata toda menção como requisito. Isso teria colocado notificação por e-mail, dashboard visual e rate limiting dentro do escopo — os três foram explicitamente descartados ou adiados ([09:37], [09:39] Larissa).

### Prompt 2 — Mapeamento do código com caminhos reais

Usado para construir a seção obrigatória de integração do FDD.

```text
Explore o repositório e mapeie os pontos de integração da feature de
webhooks com o código existente. Para CADA item, informe o caminho de
arquivo exato e um trecho curto do código (assinatura, não corpo inteiro):

1. O método changeStatus: assinatura, o que a transação executa, tipo do
   client transacional, erros que lança.
2. A taxonomia de erros: classe base, subclasses, convenção de código,
   como o middleware central os traduz para HTTP.
3. Autenticação e autorização: como requireRole é implementado e onde já
   é usado.
4. Configuração de ambiente: como as variáveis são validadas.
5. Logger: como está configurado e o que já é redigido (redact).
6. Schema Prisma: convenções de PK, @@map e índices.
7. Wiring: como controllers e routers são construídos e registrados.

REGRAS:
- Todo caminho de arquivo deve existir. Se não tiver certeza, verifique
  antes de reportar. Não reporte caminho "provável".
- Não descreva o que o código "deveria" fazer. Descreva o que ele faz.
- Para cada ponto, diga explicitamente como o módulo de webhooks se
  acopla: altera, estende ou apenas reutiliza?
```

**O que esse prompt produziu de concreto.** Verificar o logger real contra a decisão de HMAC revelou uma lacuna concreta: `src/shared/logger/index.ts` redige `*.token` e `*.password`, mas **não** `*.secret` nem `x-signature`. Isso virou um requisito específico no FDD em vez de um genérico "cuidar de secrets".

### Prompt 3 — ADR com alternativa obrigatória

Usado para cada uma das oito decisões.

```text
Escreva um ADR em formato MADR sobre: <decisão>.

Seções obrigatórias: Status, Contexto, Decisão, Alternativas Consideradas,
Consequências (positivas e negativas).

REGRAS:
1. A seção Contexto deve explicar o problema ANTES de revelar a decisão,
   citando a situação concreta que a motivou.
2. Alternativas Consideradas: pelo menos uma alternativa REAL discutida na
   reunião, com o trade-off específico que motivou o descarte. Não invente
   alternativas plausíveis genéricas.
3. Consequências negativas são obrigatórias e devem ser desconfortáveis.
   Um ADR sem custo real não registra uma decisão, registra um anúncio.
4. Toda afirmação entre aspas ou entre colchetes deve referenciar um
   timestamp [hh:mm] Falante da transcrição.
5. Não use "boas práticas" como justificativa. A justificativa é o que foi
   dito na reunião ou o que existe no código.
```

**O que esse prompt evitou.** A regra 3 é a mais importante. Sem ela, a IA escreve ADRs que só listam benefícios — e um ADR que não registra o custo da decisão é inútil seis meses depois, quando alguém pergunta por que a latência mínima é de 2 segundos.

---

## Iterações e ajustes

Registrei os três ajustes mais significativos, incluindo o raciocínio por trás de cada correção.

### Iteração 1 — ADR de retry e DLQ: de um documento genérico para dois específicos

**O que a IA gerou primeiro.** Um único ADR cobrindo "política de retry com backoff e DLQ", tratando a fila de mensagens mortas como detalhe da política de retry.

**Por que estava errado.** As duas decisões têm contextos, alternativas e consequências distintos. A política de retry responde "quantas vezes e com qual espaçamento"; o destino dos eventos que esgotam as tentativas responde "onde guardar e como recuperar". As alternativas descartadas são diferentes (retry indefinido e 3 tentativas vs. status `failed` na outbox e reprocesso automático), e as consequências negativas também — a primeira produz uma janela de ~15h, a segunda produz dependência de intervenção humana.

**Correção.** Dividido em [ADR-003](docs/adrs/ADR-003-retry-com-backoff-exponencial.md) (política de retry) e [ADR-007](docs/adrs/ADR-007-dlq-em-tabela-separada.md) (armazenamento e recuperação). O resultado foram oito ADRs cobrindo as seis decisões principais mais duas secundárias, em vez de sete com uma sobrecarregada.

### Iteração 2 — Observabilidade: detectando uma alucinação em formação

**O que a IA gerou primeiro.** Uma seção de observabilidade com métricas, logs e tracing apresentados como requisitos da feature, no mesmo tom das demais seções.

**Por que estava errado.** **A reunião nunca discutiu métricas nem tracing.** O único gancho real é o reuso do Pino ([09:29] Bruno) e a exigência de auditoria do replay ([09:36] Sofia). Apresentar métricas e tracing como se tivessem sido decididos era exatamente o tipo de invenção que o tracker existe para pegar — e que o enunciado proíbe.

**Correção.** A seção foi reescrita em dois blocos com proveniência explícita: **logs** ancorados na decisão real de reuso do Pino, e **métricas e tracing** marcados como *proposta deste FDD, não discutida na reunião, sujeita a confirmação do time*. A distinção foi registrada no tracker (linha `FDD-OBS-04`). Isso preserva o valor do documento — uma feature com DLQ sem observabilidade é inoperável — sem inventar decisões.

### Iteração 3 — Modelagem de dados: separando o que foi dito do que foi desenhado

**O que a IA gerou primeiro.** Os modelos Prisma apresentados como se cada tabela, coluna e índice tivesse sido decidido na reunião.

**Por que estava errado.** A reunião cita campos e comportamentos — `url + secret + customer_id + estado ativo` ([09:21] Bruno), estados pendente/processando/falhou/entregue ([09:08] Diego), índice em `status` e `created_at` ([09:08] Diego), a DLQ com payload/motivo/timestamp ([09:18] Diego). Mas o desenho concreto das tabelas, a criação de uma tabela `webhook_deliveries` para sustentar o histórico de entregas e a escolha de `nextAttemptAt` como coluna são decisões de implementação, não falas da reunião.

**Correção.** Adicionei uma nota de proveniência antes dos modelos, separando explicitamente **campos e comportamentos** (rastreados à transcrição) de **estrutura das tabelas** (decisão do FDD, a validar em revisão). As linhas de código da seção de integração foram marcadas como `Fonte: CODIGO` no tracker, com caminho de arquivo real.

### Ajustes menores

- **Exclusões como seção de primeira classe.** A seção "Fora de escopo" do PRD ganhou uma nota de alerta explícita, porque o risco de a IA reintroduzir um item descartado como requisito em outro documento é real e recorrente.
- **RFC enxugado.** A primeira versão do RFC continha tabelas de headers e exemplos de payload — conteúdo que pertence ao FDD. Removido, e adicionada uma seção final "O que este RFC não decide" apontando para o FDD.
- **Requisitos funcionais expandidos.** A primeira extração produziu 9 requisitos funcionais. Revisando a transcrição com o filtro de categoria `REQUISITO`, cheguei a 16 — os endpoints de CRUD estavam sendo agrupados em um único item quando são quatro requisitos distintos.

**Total: 3 iterações estruturais principais** e um conjunto de ajustes de escopo, acima do intervalo de 3 a 5 ciclos sugerido pelo enunciado.

---

## Como navegar a entrega

### Arquivos entregues

```
├── README.md                                  ← este documento (processo de produção)
├── TRANSCRICAO.md                             ← fonte primária (não alterado)
├── docs/
│   ├── PRD.md                                 ← problema, escopo, requisitos, métricas
│   ├── RFC.md                                 ← proposta técnica, alternativas, questões abertas
│   ├── FDD.md                                 ← especificação de implementação
│   ├── TRACKER.md                             ← rastreabilidade item → origem
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-retry-com-backoff-exponencial.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-entrega-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-existentes.md
│       ├── ADR-007-dlq-em-tabela-separada.md
│       └── ADR-008-snapshot-do-payload-na-insercao.md
├── src/ · prisma/ · tests/                    ← código da aplicação (não alterado)
```

### Ordem de leitura sugerida

**Para entender o produto** — quem decide se a feature vale a pena:
1. [PRD](docs/PRD.md) → o problema, o escopo e as métricas. Comece pela seção "Fora de escopo": ela delimita o que a feature **não** é.

**Para revisar a proposta técnica** — quem vai aprovar a abordagem:
2. [RFC](docs/RFC.md) → a proposta em 2 a 4 páginas. As seções "Alternativas consideradas" e "Questões em aberto" são o núcleo da revisão.
3. Os 8 [ADRs](docs/adrs/) → cada decisão isolada com seu trade-off. Leia em ordem numérica; os links entre eles formam o grafo de dependências.

**Para implementar** — quem vai escrever o código:
4. [FDD](docs/FDD.md) → comece por "Fluxos detalhados", depois "Contratos públicos", depois a seção **"Integração com o sistema existente"**, que nomeia os arquivos reais a tocar. O único ponto de alteração em código existente é `src/modules/orders/order.service.ts`.

**Para auditar a documentação** — quem quer verificar se algo foi inventado:
5. [TRACKER](docs/TRACKER.md) → cada item dos documentos com sua origem em `[hh:mm] Falante` ou caminho de arquivo. Se um item não está lá, não tem origem identificável.

### Verificação rápida de consistência

```bash
# Confirmar que a entrega é puramente documental (nenhuma alteração em código)
git diff --stat HEAD -- src/ prisma/ tests/

# Confirmar que os caminhos de arquivo citados nos documentos existem
grep -oE 'src/[a-z/.-]+\.ts|prisma/[a-z.]+' docs/FDD.md | sort -u | while read f; do
  [ -e "$f" ] && echo "OK   $f" || echo "FALTA $f"
done
```

---

## Referência ao enunciado original

Este repositório é um fork do repositório base do desafio **"Da Reunião ao Documento: Design Docs Gerados por IA"** do MBA. O enunciado original foi substituído por este documento, conforme previsto nas instruções. A transcrição da reunião técnica que originou toda a documentação está preservada em [`TRANSCRICAO.md`](TRANSCRICAO.md) e não foi alterada.
