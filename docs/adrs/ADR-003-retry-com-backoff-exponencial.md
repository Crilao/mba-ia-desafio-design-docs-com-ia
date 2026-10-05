# ADR-003: Política de retry com backoff exponencial em 5 tentativas

## Status

**Aceito** — 2026-10-05

- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos), Marcos (Product Manager)
- **Origem na transcrição:** [09:14]–[09:17]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#fluxo-3-retry-e-backoff), [ADR-007](ADR-007-dlq-em-tabela-separada.md)

## Contexto

O cliente pode estar indisponível no momento do envio. A política de retry precisa responder a duas perguntas: **quantas tentativas** e **com qual espaçamento entre elas**.

O problema a evitar foi enunciado explicitamente: retry indefinido com backoff faz com que um evento fique "pendurado para sempre" se o cliente desapareceu ([09:15] Diego). Por outro lado, uma política agressiva demais mata eventos que seriam entregues com sucesso poucas horas depois — o time já teve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

Há um requisito de produto relevante: o cliente aceita que uma indisponibilidade longa dele signifique perda, desde que a janela seja razoável ([09:17] Marcos).

## Decisão

**5 tentativas** com **backoff exponencial** na progressão **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas** ([09:15] Diego, [09:17] Diego).

A janela total entre a primeira falha e a última tentativa é de aproximadamente **15 horas** ([09:17] Diego). Esgotadas as 5 tentativas, o evento é considerado falha permanente e movido para a DLQ ([09:15] Diego), conforme [ADR-007](ADR-007-dlq-em-tabela-separada.md).

## Alternativas Consideradas

### 1. Retry indefinido com backoff

Defendida por "algumas pessoas" no debate ([09:15] Diego).

- **Trade-off que motivou o descarte:** um evento cujo cliente desapareceu fica pendurado para sempre, consumindo processamento e poluindo a leitura da outbox. Não há critério natural de encerramento ([09:15] Diego).

### 2. 3 tentativas

Proposta por Bruno como política mais agressiva ([09:16] Bruno).

- **Trade-off que motivou o descarte:** 3 tentativas cobrem uma janela curta demais. Se o cliente teve indisponibilidade pela manhã, seriam três tentativas em cerca de 30 minutos e o evento seria descartado. O time já teve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).

### 3. Número maior de tentativas (6 ou mais) com progressão mais longa

- **Trade-off que motivou o descarte:** estenderia a janela além do que o cliente considera útil. Foi avaliado que 15 horas já é o limite do aceitável — "se um cliente meu cair por 15 horas, ele já está com problema sério dele" ([09:17] Marcos).

## Consequências

### Positivas

- **Cobre indisponibilidades realistas.** A janela de ~15 horas absorve manutenções planejadas de algumas horas, que foi o caso concreto que motivou o descarte das 3 tentativas ([09:16] Diego).
- **Encerramento garantido.** Todo evento sai da outbox: ou é entregue, ou vai para a DLQ. Não há acúmulo indefinido ([09:15] Diego).
- **Custo decrescente.** As tentativas ficam espaçadas, então um cliente em manutenção longa não é martelado com requisições.
- **Progressão alinhada ao requisito de latência.** Como só o caminho de falha usa backoff, o caminho feliz continua dentro dos 2 segundos de polling.

### Negativas

- **Janela de até 15 horas para o cliente descobrir que perdeu um evento.** Aceita explicitamente pelo PM ([09:17] Marcos).
- **Números arbitrários.** A progressão 1m/5m/30m/2h/12h foi escolhida por julgamento, não por medição de comportamento real dos clientes. É um candidato natural a revisão depois de observar tráfego de produção.
- **Requer persistência de estado de retry.** A outbox precisa armazenar contador de tentativas e a próxima data de tentativa, o que adiciona campos e índice à modelagem.
- **Interação com ordering.** Um evento em backoff fica "preso" enquanto eventos mais novos do mesmo pedido podem ser entregues antes, quebrando a ordenação por `order_id` justamente no caminho de falha. Ver [ADR-002](ADR-002-worker-separado-em-polling.md).
