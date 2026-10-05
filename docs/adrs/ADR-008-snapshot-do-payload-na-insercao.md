# ADR-008: Snapshot do payload renderizado no momento da inserção na outbox

## Status

**Aceito** — 2026-10-05

- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
- **Origem na transcrição:** [09:51]–[09:52]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#fluxo-1-inserção-na-outbox-dentro-da-transação-de-mudança-de-status), [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto

A linha da outbox precisa, em algum momento, virar o JSON que será enviado ao cliente. Isso levanta uma escolha de modelagem: a outbox guarda o **payload já renderizado**, ou guarda apenas uma referência (por exemplo `order_id`) e renderiza o JSON **no momento do envio**?

A questão foi levantada por Bruno já no encerramento da reunião ([09:51] Bruno), depois que as decisões principais já estavam fechadas.

O ponto sensível é a defasagem temporal entre a inserção e o envio. Entre esses dois instantes podem decorrer horas — no caminho de falha, até 15 horas de backoff ([09:17] Diego). Nesse intervalo o pedido pode ter mudado de novo.

## Decisão

A outbox guarda o **payload renderizado já na inserção** — um **snapshot** do estado do pedido no momento em que o status mudou ([09:52] Larissa, [09:52] Diego).

A justificativa registrada: se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Caso contrário, haveria "caso esquisito" ([09:52] Larissa).

Isso é coerente com o conteúdo do payload definido em [09:43] Diego, que inclui `from_status` e `to_status` — campos que descrevem uma transição específica e que perderiam sentido se renderizados a partir do estado atual do pedido.

## Alternativas Consideradas

### 1. Renderizar o payload no momento do envio

Guardar apenas `order_id` e os campos da transição na outbox, e montar o JSON consultando o pedido no momento de cada tentativa.

- **Trade-off que motivou o descarte:** o evento passaria a refletir o estado **atual** do pedido, não o estado no momento da mudança de status. Um evento inserido quando o pedido virou `PAID` e enviado três horas depois, quando o pedido já está `SHIPPED`, entregaria ao cliente uma notificação de `PAID` com dados de um pedido que já mudou — ou, pior, com `from_status`/`to_status` inconsistentes com o restante do corpo ([09:52] Larissa).
- **Agravante no caminho de falha:** com backoff de até 12 horas entre tentativas ([ADR-003](ADR-003-retry-com-backoff-exponencial.md)), a defasagem entre inserção e envio é a regra, não a exceção.

### 2. Renderizar na inserção e re-renderizar se o pedido mudar

Híbrido: snapshot na inserção, com atualização do payload caso o pedido mude antes do envio.

- **Trade-off que motivou o descarte:** eliminaria a garantia de que o evento é um registro fiel da transição que o originou. O cliente receberia um evento que não corresponde a nenhuma mudança de status real e observável — o evento deixaria de ser um log de transição e passaria a ser um retrato móvel. Além disso, exigiria rastrear e atualizar linhas da outbox a partir de outros fluxos, aumentando o acoplamento.

### 3. Guardar payload e referência lado a lado

Persistir o snapshot e também `order_id`, para permitir consulta posterior.

- **Trade-off que motivou o descarte:** não é exatamente uma alternativa — `order_id` já está previsto no payload ([09:43] Diego) e o `event_id` é o identificador próprio do evento ([09:25] Diego). A modelagem resultante já contempla ambos, então a alternativa não se distingue da decisão.

## Consequências

### Positivas

- **Fidelidade temporal.** O evento é um registro imutável do que aconteceu no momento da transição, independentemente de quanto tempo leve para ser entregue ([09:52] Larissa).
- **Coerência interna do payload.** `from_status`, `to_status` e os demais campos descrevem a mesma transição, sem risco de misturar estados de instantes diferentes.
- **Reprocessamento seguro a partir da DLQ.** Um evento recuperado dias depois via replay ([ADR-007](ADR-007-dlq-em-tabela-separada.md)) entrega exatamente o que deveria ter sido entregue, não uma versão distorcida.
- **Deduplicação confiável.** Como o snapshot é imutável, reenvios do mesmo `event_id` carregam corpo idêntico, o que torna a deduplicação do cliente por `X-Event-Id` ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)) determinística.

### Negativas

- **Duplicação de dados.** O payload renderizado é armazenado na outbox e novamente na DLQ em caso de falha, ocupando mais espaço do que uma simples referência. Relevante porque o arquivamento de linhas antigas ficou fora de escopo ([09:08] Diego).
- **Payload congelado no formato da época.** Se o contrato do payload evoluir, eventos antigos ainda na outbox continuarão sendo enviados no formato em que foram renderizados. Isso é desejável para fidelidade, mas exige que o cliente tolere versões de payload conviventes durante uma janela de transição.
- **Menos flexibilidade para enriquecer o evento depois.** Qualquer informação que se queira adicionar ao payload precisa estar disponível **dentro da transação** de `changeStatus`. Não é possível complementar o corpo no momento do envio com dados buscados fora da transação.
- **A renderização acontece no caminho crítico da transação.** Montar o JSON e serializá-lo ocorre dentro da transação que já é considerada pesada ([09:04] Bruno), embora o custo seja de CPU em memória e não de I/O de rede.
