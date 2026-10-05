# ADR-002: Worker em processo separado com polling de 2 segundos

## Status

**Aceito** — 2026-10-05

- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
- **Origem na transcrição:** [09:08]–[09:13]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#fluxo-2-processamento-pelo-worker), [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Com o outbox decidido em [ADR-001](ADR-001-outbox-no-mysql.md), a linha na tabela `webhook_outbox` existe, mas alguém precisa lê-la e executar a chamada HTTP ao cliente. Duas perguntas precisavam de resposta: **onde** esse consumidor roda e **como** ele descobre que há trabalho novo.

Sobre "onde": o projeto hoje tem uma única entry point, `src/server.ts`, que sobe a API HTTP. Sobre "como": o MySQL não oferece mecanismo nativo de notificação de processo externo ([09:09] Diego), o que elimina a opção reativa mais direta.

O requisito de latência é de "abaixo de 10 segundos" ([09:02] Marcos), o que é folgado em relação a qualquer estratégia de polling razoável.

## Decisão

O consumidor do outbox é um **processo separado da API**, com entry point própria em `src/worker.ts` e script `npm run worker` ([09:11] Larissa).

- A lógica de processamento fica dentro do módulo de webhooks, em `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno).
- O worker abre sua **própria instância de `PrismaClient`**. `PrismaClient` é por processo; o worker usa o mesmo banco e a mesma `DATABASE_URL`, mas uma instância nova porque é outro processo Node ([09:30] Bruno, [09:30] Diego).
- A descoberta de trabalho é por **polling em loop a cada 2 segundos**: busca os eventos pendentes mais antigos, processa e marca ([09:09] Diego).
- A leitura é feita **em batch pequeno**, apoiada nos índices de `status` e `created_at` ([09:08] Diego).

**Ordering:** com um único worker lendo em ordem de `created_at`, o cliente recebe os eventos de um mesmo pedido na ordem correta. Isso é registrado como **limitação conhecida**, não como garantia: não há garantia de ordering global, apenas por `order_id` e enquanto o worker for único ([09:12] Diego, [09:13] Larissa).

## Alternativas Consideradas

### 1. Worker dentro do processo da API

Manter o loop de processamento na mesma instância que serve HTTP.

- **Trade-off que motivou o descarte:** se a API reinicia, o worker morre junto ([09:11] Diego). O ciclo de vida dos dois é diferente — a API escala e reinicia por tráfego HTTP, o worker por volume de eventos.

### 2. Trigger de banco notificando o worker

Levantada por Bruno buscando maior reatividade ([09:09] Bruno).

- **Trade-off que motivou o descarte:** o MySQL não tem `NOTIFY`/`LISTEN` como o Postgres. Uma trigger executa SQL, mas não notifica processo externo; para avisar o worker seria preciso improvisar escrita em arquivo ou chamada a endpoint, o que foi considerado "esquisito" ([09:09] Diego). O polling de 2 segundos atende ao requisito de 10 segundos com folga ([09:09] Diego, [09:10] Marcos).

### 3. Polling com intervalo maior (ou consumo reativo por broker)

- **Trade-off que motivou o descarte:** intervalos maiores aproximariam a latência do teto de 10 segundos sem ganho relevante de carga; o consumo reativo por broker reabriria a discussão de infraestrutura extra já descartada em [ADR-001](ADR-001-outbox-no-mysql.md).

## Consequências

### Positivas

- **Ciclos de vida independentes.** Reinício ou deploy da API não interrompe a entrega de eventos ([09:11] Diego).
- **Escala independente no futuro.** O worker pode ser escalado sem escalar a API, ainda que isso exija revisitar a garantia de ordering.
- **Latência previsível e suficiente.** Pior caso de 2 segundos, bem abaixo dos 10 segundos aceitos pelo cliente ([09:10] Larissa).
- **Sem infraestrutura nova.** Mesmo banco, mesma stack ([09:11] Diego).

### Negativas

- **Latência mínima de 2 segundos** em toda entrega, mesmo quando o cliente está saudável. Aceita como trade-off explícito ([09:10] Larissa).
- **Ordering não garantida sob múltiplos workers.** Escalar horizontalmente o worker quebra a ordenação por `order_id`. As saídas mencionadas — particionar por `order_id` ou usar lock pessimista — foram explicitamente adiadas como "problema do futuro" ([09:13] Diego).
- **Carga de polling constante.** O worker consulta o banco a cada 2 segundos mesmo sem eventos pendentes. Mitigado pelo índice em `status` e pelo batch pequeno ([09:08] Diego).
- **Duas entry points para operar.** Deploy, observabilidade e shutdown precisam considerar `src/server.ts` e `src/worker.ts` como processos distintos, com o mesmo cuidado de encerramento gracioso que hoje existe em `src/server.ts`.
