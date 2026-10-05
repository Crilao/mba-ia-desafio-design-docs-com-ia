# ADR-007: DLQ em tabela separada com replay manual via endpoint admin

## Status

**Aceito** — 2026-10-05

- **Decisores:** Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos), Larissa (Tech Lead), Sofia (Engenheira de Segurança)
- **Origem na transcrição:** [09:18]–[09:19], [09:35]–[09:36]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#fluxo-4-dlq-e-replay), [ADR-003](ADR-003-retry-com-backoff-exponencial.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Esgotadas as 5 tentativas de [ADR-003](ADR-003-retry-com-backoff-exponencial.md), o evento é considerado falha permanente. É preciso decidir **onde** esse evento falho é armazenado e **como** ele pode ser recuperado.

Dois usos foram identificados para o evento falho: servir de evidência para debug e permitir reprocessamento ([09:18] Diego).

## Decisão

Criar uma tabela **separada**, `webhook_dead_letter`, contendo a **payload**, o **motivo da falha** e o **timestamp** ([09:18] Diego).

O reprocessamento é **manual, via endpoint administrativo**: `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego, [09:19] Larissa).

O endpoint de replay exige **role `ADMIN`** ([09:36] Sofia, [09:36] Larissa), reaproveitando o `requireRole` já existente em `src/middlewares/auth.middleware.ts` ([09:36] Larissa). A ação de replay **registra quem a executou**, para fins de auditoria ([09:36] Sofia).

## Alternativas Consideradas

### 1. Marcar como `failed` na própria tabela de outbox

Manter o evento na `webhook_outbox` com um status terminal de falha, sem tabela nova.

- **Trade-off que motivou o descarte:** poluiria a leitura da tabela principal. O worker lê os pendentes em batch pequeno a cada 2 segundos ([09:08] Diego); acumular registros terminais na mesma tabela degrada essa leitura ao longo do tempo. A separação mantém a outbox enxuta ([09:18] Diego).

### 2. Reprocessamento automático periódico da DLQ

Um job que tentasse reenviar eventos da DLQ automaticamente.

- **Trade-off que motivou o descarte:** reabriria o problema que a política de retry existe para resolver — um evento cujo cliente desapareceu voltaria a circular indefinidamente. A DLQ precisa ser um ponto final, não um segundo retry ([09:15] Diego). O replay manual mantém a decisão humana no loop.

### 3. Descartar eventos falhos (sem DLQ)

- **Trade-off que motivou o descarte:** eliminaria a evidência para debug e a possibilidade de recuperação após o cliente corrigir o problema do lado dele. A DLQ foi escolhida justamente por ser "evidence pra debug e reprocessamento" ([09:18] Diego).

## Consequências

### Positivas

- **Outbox principal permanece enxuta.** Só eventos em processamento ou entregues vivem nela, o que preserva o desempenho da leitura por polling ([09:18] Diego).
- **Evidência preservada.** Payload, motivo e timestamp ficam registrados para investigação de falhas ([09:18] Diego).
- **Recuperação controlada.** Um cliente que corrigiu o endpoint dele pode ter os eventos reenviados sem esperar nova mudança de status.
- **Auditoria do replay.** O registro de quem executou o replay atende à exigência de segurança ([09:36] Sofia).

### Negativas

- **Recuperação depende de intervenção humana.** Não há caminho automático de saída da DLQ. Se ninguém olhar, o evento fica lá indefinidamente. Mitigável por observabilidade sobre o volume da DLQ — ver a seção de observabilidade do [FDD](../FDD.md#observabilidade).
- **Endpoint administrativo é superfície de ataque.** Exige role `ADMIN` e auditoria ([09:36] Sofia). O replay recoloca o evento na outbox, o que significa que uma ação administrativa pode gerar tráfego para o cliente.
- **Estado duplicado entre outbox e DLQ.** O evento precisa ser removido da outbox ao ir para a DLQ e reinserido no replay, o que exige cuidado transacional para não perder nem duplicar registros. Como a entrega é at-least-once ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)), uma duplicata em replay é tolerável, mas a perda não é.
- **Sem política de retenção definida.** Não foi discutido por quanto tempo eventos ficam na DLQ. O arquivamento da outbox após 30 dias foi mencionado para a outbox, mas explicitamente deixado fora de escopo ([09:08] Diego), e a DLQ não teve regra definida.
