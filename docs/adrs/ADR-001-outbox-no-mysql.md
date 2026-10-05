# ADR-001: Adoção do padrão Outbox no MySQL existente

## Status

**Aceito** — 2026-10-05

- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
- **Origem na transcrição:** [09:03]–[09:08]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#fluxo-1-inserção-na-outbox-dentro-da-transação-de-mudança-de-status), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-008](ADR-008-snapshot-do-payload-na-insercao.md)

## Contexto

Os clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) precisam ser notificados quando o status de um pedido muda. Hoje eles fazem polling no `GET /orders`, o que torna a integração lenta e cara para eles.

A aplicação não possui nenhum mecanismo de evento, fila ou notificação externa. O único ponto do sistema onde o status de um pedido muda é o método `changeStatus` em `src/modules/orders/order.service.ts`, que executa uma transação com três efeitos: atualiza `orders`, insere em `order_status_history` e, conforme a transição, debita ou repõe `stock_quantity` dos produtos.

A pergunta colocada na reunião foi: o disparo da notificação acontece de forma síncrona dentro dessa transação, ou passa por algum mecanismo de fila/outbox?

O requisito de latência é frouxo — os clientes consideram "tempo real" qualquer coisa abaixo de 10 segundos ([09:02] Marcos) —, o que abre espaço para uma solução assíncrona. A restrição real é de consistência: não pode existir o caso em que o status muda e o evento não é registrado ([09:40] Bruno).

## Decisão

Adotar o padrão **transactional outbox** com a tabela de outbox no **MySQL já existente**.

Quando o status do pedido muda, dentro da **mesma transação SQL** que atualiza `orders` e insere em `order_status_history`, também é inserida uma linha na tabela `webhook_outbox` com o evento já renderizado. Um worker separado lê essa tabela e dispara as chamadas HTTP.

A garantia que se busca é de atomicidade: se a transação principal commitou, o evento foi registrado; se ela deu rollback, o evento desaparece junto ([09:06] Diego).

A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação corrente, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Alternativas Consideradas

### 1. Disparo síncrono dentro de `changeStatus`

Foi a primeira hipótese levantada ([09:03] Larissa) e descartada na sequência.

- **Trade-off que motivou o descarte:** a transação de mudança de status já é pesada — atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity`. Inserir uma chamada HTTP no meio dela faria com que qualquer cliente lento travasse a mudança de status de outros pedidos ([09:04] Bruno).
- **Problema secundário:** a semântica de falha é indefinida. Se o cliente estiver fora do ar, não há resposta razoável — não se dá rollback na mudança de status por causa de um webhook ([09:04] Bruno).

### 2. Redis Streams (ou broker externo equivalente)

Levantada por Larissa como contraponto ao outbox ([09:07] Larissa).

- **Trade-off que motivou o descarte:** exigiria subir infraestrutura adicional. O time é pequeno e subir um Redis Cluster para esse volume é overengineering ([09:07] Diego). O outbox resolve no MySQL existente.

### 3. Trigger de banco para notificar o worker

Levantada por Bruno ao discutir a reatividade do worker ([09:09] Bruno).

- **Trade-off que motivou o descarte:** o MySQL não tem listener nativo como o `NOTIFY`/`LISTEN` do Postgres. Uma trigger executa SQL, mas não notifica um processo externo; para avisar o worker seria preciso improvisar mecanismos como escrever em arquivo ou bater em um endpoint ([09:09] Diego). O descarte dessa alternativa é o que sustenta a decisão de polling registrada em [ADR-002](ADR-002-worker-separado-em-polling.md).

## Consequências

### Positivas

- **Atomicidade entre mudança de status e registro do evento.** Não existe janela em que o status muda sem o evento correspondente, nem evento sem mudança de status ([09:06] Diego, [09:41] Diego).
- **Nenhuma infraestrutura nova.** Reaproveita o MySQL e o Prisma Client já presentes no projeto ([09:07] Diego).
- **Desacoplamento da latência do cliente.** Um cliente lento não impacta a transação de pedidos ([09:04] Bruno).
- **Base para reprocessamento.** A linha persistida é a evidência necessária para retry e para a DLQ ([09:18] Diego).

### Negativas

- **Latência mínima de 2 segundos** entre a mudança de status e a entrega, imposta pelo ciclo de polling do worker. Aceita explicitamente ([09:10] Larissa).
- **Crescimento da tabela de outbox.** Exige índice em `status` e `created_at` e leitura em batch pequeno. O arquivamento das linhas entregues após 30 dias foi mencionado, mas ficou **fora do escopo desta feature** ([09:08] Diego).
- **Garantia de ordering limitada.** Com o outbox lido em ordem de `created_at` por um único worker há ordering por `order_id`; ao escalar para múltiplos workers a garantia se perde ([09:12] Diego). Ver [ADR-002](ADR-002-worker-separado-em-polling.md).
- **Complexidade de teste.** Passa a existir um caminho assíncrono entre a mudança de status e a entrega, o que exige testes ponta a ponta e não apenas testes do service.
