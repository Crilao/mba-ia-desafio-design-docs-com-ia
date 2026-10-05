# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Cristiano Furlan |
| **Status** | Em revisão |
| **Data** | 2026-10-05 |
| **Revisores** | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança) |
| **Origem** | Reunião técnica de quinta-feira, ~55 min (`TRANSCRICAO.md`) |
| **Documentos relacionados** | [FDD](FDD.md) · [PRD](PRD.md) · [ADRs](adrs/) |

---

## Resumo executivo (TL;DR)

Propomos implementar notificação de mudança de status de pedidos por **webhook outbound**, usando o padrão **transactional outbox no MySQL já existente**.

O evento é gravado na tabela `webhook_outbox` **dentro da mesma transação** que hoje atualiza o pedido, insere no histórico e mexe no estoque. Um **worker em processo separado**, em **polling de 2 segundos**, lê essa tabela e dispara as chamadas HTTP, com **retry exponencial em 5 tentativas** e **DLQ** para falhas permanentes. A autenticação é **HMAC-SHA256 com secret por endpoint**, a entrega é **at-least-once** com deduplicação por `X-Event-Id`, e o módulo **reaproveita os padrões já existentes** na codebase.

A decisão central é não introduzir infraestrutura nova: a garantia de consistência vem da transação SQL, não de um broker.

**Esforço estimado:** 3 sprints, incluindo a revisão de segurança ([09:46] Larissa).

---

## Contexto e problema

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente notificação em tempo real de mudança de status dos pedidos deles ([09:00] Marcos). Hoje eles fazem polling no `GET /orders`, o que torna a integração lenta e cara do lado deles. A Atlas sinalizou risco de migração para o concorrente caso a entrega não ocorra até o fim do trimestre ([09:00] Marcos).

"Tempo real", para esses clientes, significa **qualquer latência abaixo de 10 segundos** ([09:02] Marcos).

Do lado da plataforma, a aplicação **não possui nenhum mecanismo de evento, fila ou webhook**. O único ponto de mudança de status é o método `changeStatus` em `src/modules/orders/order.service.ts`, cuja transação já executa três efeitos: atualiza `orders`, insere em `order_status_history` e debita ou repõe `stock_quantity` dos produtos.

A restrição que domina o desenho é de **consistência**: não pode existir o caso em que o status muda e o evento correspondente não é registrado ([09:40] Bruno). A restrição secundária é de **não impactar a transação de pedidos**, que já é pesada ([09:04] Bruno).

Escopo confirmado na abertura: o fluxo é **outbound apenas** — a plataforma envia, os clientes recebem, não há webhook de entrada ([09:02] Sofia, [09:02] Marcos).

---

## Proposta técnica

### Visão geral

```
changeStatus (transação)                    worker (processo separado)
┌──────────────────────────────┐            ┌─────────────────────────────┐
│ 1. valida transição          │            │ 1. polling 2s               │
│ 2. debita/repõe estoque      │            │ 2. lê pendentes (created_at)│
│ 3. update orders             │            │ 3. assina e envia HTTP      │
│ 4. insert order_status_history│  outbox   │ 4. sucesso → entregue       │
│ 5. insert webhook_outbox  ───┼──────────► │    falha  → backoff         │
└──────────────────────────────┘            │ 5. 5 falhas → DLQ           │
        commit atômico                      └─────────────────────────────┘
```

### Os quatro pilares

**1. Outbox transacional.** O evento nasce dentro da transação de `changeStatus`, por meio de uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` corrente em vez de um repository injetado ([09:41] Bruno). O payload é gravado **já renderizado** — um snapshot do estado no momento da transição ([09:52] Larissa). A filtragem por interesse do cliente acontece **na inserção**: se nenhum webhook do customer quer aquele status, nenhuma linha é criada ([09:34] Bruno). Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-008](adrs/ADR-008-snapshot-do-payload-na-insercao.md).

**2. Worker separado em polling.** Entry point própria em `src/worker.ts` com script `npm run worker` ([09:11] Larissa), processamento em `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno). O worker abre sua própria instância de `PrismaClient` — mesmo banco, mesma stack, processo distinto ([09:30] Bruno). Ver [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).

**3. Retry com backoff e DLQ.** 5 tentativas na progressão **1m / 5m / 30m / 2h / 12h**, cobrindo ~15 horas entre a primeira falha e a última tentativa ([09:17] Diego). Esgotadas as tentativas, o evento vai para a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp ([09:18] Diego). A recuperação é **manual**, por endpoint admin ([09:18] Diego). Ver [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md) e [ADR-007](adrs/ADR-007-dlq-em-tabela-separada.md).

**4. Segurança e contrato de entrega.** HMAC-SHA256 sobre o corpo do request, no header `X-Signature`, com **secret única por endpoint** e **rotação com grace period de 24 horas** ([09:21] Sofia). URL obrigatoriamente HTTPS, recusada na validação em caso de `http` ([09:23] Sofia). Entrega **at-least-once**, com `X-Event-Id` carregando um UUID gerado na inserção da outbox para deduplicação do lado do cliente ([09:25] Diego). Ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) e [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md).

### Superfície de API

Quatro grupos de endpoints, detalhados no [FDD](FDD.md#contratos-públicos):

- **CRUD de configuração** de webhook por customer — criar, listar, editar, remover — autenticado com JWT comum ([09:31]–[09:33] Marcos, Bruno).
- **Rotação de secret** pelo cliente ([09:21] Sofia).
- **Histórico de entregas** — `GET /webhooks/:id/deliveries`, com sucesso/falha, payload, response e tempo de resposta ([09:34] Marcos).
- **Replay de DLQ** — `POST /admin/webhooks/dead-letter/:id/replay`, exigindo role `ADMIN` e registrando o autor da ação para auditoria ([09:35]–[09:36] Sofia).

### Integração com o sistema existente

A alteração crítica é cirúrgica e localizada: o passo 5 dentro da transação de `changeStatus`. Fora isso, o módulo de webhooks é **aditivo** — um módulo novo em `src/modules/webhooks/` seguindo o padrão de `src/modules/orders/`, reutilizando `AppError` e derivadas, o logger Pino, o `error.middleware.ts` centralizado, o `validate.middleware.ts` e o `requireRole` ([09:29] Bruno, [09:36] Larissa). Nenhuma dessas peças precisa ser modificada. O detalhamento está em [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) e na seção [Integração com o sistema existente](FDD.md#integração-com-o-sistema-existente) do FDD.

---

## Alternativas consideradas

### 1. Disparo síncrono dentro de `changeStatus`

Foi a primeira hipótese colocada na mesa ([09:03] Larissa) e caiu rapidamente.

**Trade-off que motivou o descarte:** a transação de mudança de status já é pesada — atualiza `orders`, insere em `order_status_history` e decrementa estoque. Inserir uma chamada HTTP no meio dela faria qualquer cliente lento **travar a mudança de status de outros pedidos** ([09:04] Bruno). Havia ainda um problema de semântica: se o cliente estivesse fora do ar, não há resposta razoável — não se dá rollback na mudança de status por causa de um webhook ([09:04] Bruno).

### 2. Redis Streams (ou broker externo)

Levantada como contraponto direto ao outbox ([09:07] Larissa).

**Trade-off que motivou o descarte:** exigiria subir infraestrutura adicional. O time é pequeno e **subir um Redis Cluster para esse volume é overengineering** ([09:07] Diego). O outbox resolve no MySQL existente, sem novo componente para operar.

### 3. Trigger de banco notificando o worker

Levantada buscando reatividade maior que o polling ([09:09] Bruno).

**Trade-off que motivou o descarte:** o MySQL **não tem listener nativo** como o `NOTIFY`/`LISTEN` do Postgres. Uma trigger executa SQL, mas não notifica processo externo; avisar o worker exigiria improvisar escrita em arquivo ou chamada a endpoint, o que foi considerado "esquisito" ([09:09] Diego). O polling de 2s atende ao requisito de 10s com folga.

### 4. Garantia exactly-once

**Trade-off que motivou o descarte:** exigiria **coordenação dos dois lados** — plataforma e cada cliente — e ficaria muito mais complexo ([09:25] Diego). At-least-once com `event_id` resolve a grande maioria dos casos e é o que Stripe e GitHub fazem ([09:25] Diego). O custo aceito é transferir a deduplicação para o cliente ([09:25] Sofia).

### 5. Retry indefinido, e retry com 3 tentativas

Duas variantes da política de retry foram descartadas em sequência. **Retry indefinido** deixaria eventos pendurados para sempre quando o cliente desaparece ([09:15] Diego). **3 tentativas** seria agressivo demais: cobriria cerca de 30 minutos, e o time já teve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

---

## Questões em aberto

Pontos levantados na reunião e **deliberadamente não decididos** ou adiados. Nenhum deles bloqueia o início da implementação.

### QA-1 — Rate limiting de saída por cliente

Se um cliente tem 50 pedidos mudando de status em um minuto, a plataforma faria 50 chamadas HTTP para ele ([09:38] Diego). A decisão foi **observar e decidir depois**: "a gente observa e implementa se virar problema" ([09:39] Diego, [09:39] Larissa). Fica registrado como ponto em aberto, não como requisito desta fase.

### QA-2 — Escalar o worker mantendo ordering

A ordering por `order_id` só existe enquanto houver **um único worker** lendo em ordem de `created_at` ([09:12] Diego). Ao escalar horizontalmente a garantia se perde. As saídas cogitadas — **particionar por `order_id`** ou usar **lock pessimista** — foram explicitamente adiadas como "problema do futuro, não agora" ([09:13] Diego). A limitação é registrada como conhecida, não como garantia ([09:13] Larissa).

### QA-3 — Retenção e arquivamento das linhas entregues

O arquivamento de linhas entregues após ~30 dias foi mencionado ([09:08] Diego), mas classificado como **fora do escopo desta feature**. A outbox vai crescer sem política de retenção definida até que isso seja endereçado.

### QA-4 — Endurecer a autorização do CRUD

O CRUD de configuração de webhook aceita qualquer role autenticada nesta fase. Foi registrado que "mais pra frente a gente pode endurecer" ([09:37] Sofia). Não há requisito de restrição por customer ou por role além do `ADMIN` no replay.

---

## Impacto e riscos

| Risco | Impacto | Mitigação |
| --- | --- | --- |
| **Alteração na transação de `changeStatus`** — é o caminho crítico de pedidos em produção | Alto | A inserção na outbox é o último passo da transação e reaproveita o `tx` corrente, sem I/O de rede. Testes ponta a ponta cobrem o caminho ([09:40] Bruno) |
| **Cliente indisponível por período longo** | Médio | Backoff cobre ~15h; depois disso o evento é preservado na DLQ e recuperável por replay ([09:17] Diego, [09:18] Diego) |
| **Vazamento de secret do lado do cliente** — já ocorreu antes | Médio | Secret por endpoint limita o raio de impacto; rotação com grace period de 24h permite troca sem downtime ([09:21] Sofia, [09:22] Diego) |
| **Crescimento da tabela de outbox** degradando a leitura do worker | Médio | Índices em `status` e `created_at`, leitura em batch pequeno ([09:08] Diego). Arquivamento adiado — ver QA-3 |
| **Múltiplas instâncias do worker** quebrando ordering silenciosamente | Médio | Limitação documentada; single-worker nesta fase ([09:13] Larissa). Ver QA-2 |
| **Dependência de revisão de segurança** antes do deploy | Baixo | Dois dias úteis reservados para a Sofia revisar HMAC e geração de secret ([09:46] Sofia) |

**Impacto no prazo:** 3 sprints, incluindo a revisão de segurança no fim ([09:46] Larissa). A Atlas espera entrega até **fim de novembro** ([09:45] Marcos).

---

## Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Adoção do padrão Outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md) | Política de retry com backoff exponencial em 5 tentativas |
| [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint e rotação |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto |
| [ADR-007](adrs/ADR-007-dlq-em-tabela-separada.md) | DLQ em tabela separada com replay manual via endpoint admin |
| [ADR-008](adrs/ADR-008-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado na inserção na outbox |

---

## O que este RFC não decide

Este documento **não** especifica contratos de endpoint, payloads de exemplo, matriz de erros, esquema de tabelas nem estratégia de observabilidade. Esses detalhes estão no [FDD](FDD.md), que é o documento de implementação. Aqui a intenção é fechar a **abordagem** e expor o que segue em aberto para revisão.
