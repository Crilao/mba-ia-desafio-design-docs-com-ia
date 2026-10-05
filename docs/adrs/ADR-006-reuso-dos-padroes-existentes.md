# ADR-006: Reuso máximo dos padrões existentes do projeto

## Status

**Aceito** — 2026-10-05

- **Decisores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma)
- **Origem na transcrição:** [09:27]–[09:30], [09:51]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#integração-com-o-sistema-existente), [ADR-002](ADR-002-worker-separado-em-polling.md)

## Contexto

A feature de webhooks introduz um módulo novo, uma entry point nova e um conjunto novo de erros e contratos. A pergunta colocada foi se esse módulo deveria trazer convenções próprias ou se deveria se submeter aos padrões já estabelecidos na codebase.

A codebase atual já tem convenções fortes e consistentes:

- Cada domínio é um módulo em `src/modules/<dominio>/` com `controller`, `service`, `repository`, `routes` e `schemas` — ver `src/modules/orders/`.
- Erros derivam de `AppError` (`src/shared/errors/app-error.ts`) com subclasses em `src/shared/errors/http-errors.ts`, usando códigos em `SCREAMING_SNAKE_CASE`.
- O middleware centralizado `src/middlewares/error.middleware.ts` já traduz `AppError`, `ZodError` e erros conhecidos do Prisma para respostas HTTP.
- O logger é Pino, exportado como singleton em `src/shared/logger/index.ts`.
- Validação é feita com Zod via `src/middlewares/validate.middleware.ts`.
- Autorização usa `requireRole`, de `src/middlewares/auth.middleware.ts`.

## Decisão

**Reuso máximo do que já existe.** O módulo de webhooks é um módulo igual aos outros, sem stack própria e sem convenções paralelas ([09:30] Larissa).

Concretamente:

- **Estrutura de módulo:** `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts` ([09:27] Bruno).
- **Processamento:** a lógica do worker fica dentro do módulo, em `src/modules/webhooks/webhook.processor.ts`, com `src/worker.ts` como entry point separada ([09:28] Bruno).
- **Erros:** novas classes derivadas de `AppError`, seguindo o padrão de `InsufficientStockError` e `InvalidStatusTransitionError`. Códigos com prefixo **`WEBHOOK_`** para tudo do módulo — `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, etc. ([09:28] Bruno, [09:29] Larissa).
- **Logger:** Pino, o mesmo do projeto. Nenhuma ferramenta nova ([09:29] Bruno).
- **Middleware de erro:** nenhuma alteração. Ele já trata `AppError`, `ZodError` e Prisma, e vai capturar os erros do módulo sem modificação ([09:29] Bruno).
- **Validação:** schemas Zod, consumidos pelo `validate.middleware.ts` existente.
- **Autorização:** `requireRole('ADMIN')` reutilizado para o endpoint de replay de DLQ ([09:36] Larissa).
- **Identificadores:** UUID como chave primária das novas tabelas, seguindo o padrão do resto do projeto, onde tudo é UUID ([09:51] Larissa).
- **Instanciação:** o worker usa uma instância própria de `PrismaClient`, mas construída com o mesmo `src/config/database.ts` e a mesma `DATABASE_URL` — `PrismaClient` é por processo ([09:30] Bruno).

## Alternativas Consideradas

### 1. Módulo isolado com stack própria

Tratar webhooks como um subsistema à parte, com estrutura de pastas, tratamento de erro e logger próprios.

- **Trade-off que motivou o descarte:** duplicaria infraestrutura transversal já resolvida (erro, log, validação, autorização) e criaria duas formas de fazer a mesma coisa na mesma codebase. Um desenvolvedor que conhece o módulo de orders passaria a precisar aprender convenções novas para ler o de webhooks. O custo de manutenção cresce sem contrapartida, já que não há requisito que justifique tratamento diferenciado ([09:30] Larissa).

### 2. Extrair uma biblioteca compartilhada antes de implementar

Refatorar os padrões comuns para um pacote interno e então construir webhooks sobre ele.

- **Trade-off que motivou o descarte:** introduziria refatoração de código existente numa feature que é puramente aditiva. O escopo do desafio e da feature é adicionar webhooks, não reorganizar a base. O reuso dos arquivos existentes já entrega o benefício sem tocar no que funciona.

### 3. Injetar o repository de webhooks no `OrderService`

Fazer o `OrderService` receber `WebhookRepository` no construtor para publicar eventos.

- **Trade-off que motivou o descarte:** acoplaria o módulo de pedidos ao módulo de webhooks no nível de construção de objetos, exigindo alteração em `buildControllers`, em `src/app.ts`. A alternativa aceita é uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação corrente — "função pura recebendo o tx, não precisa injetar repository inteiro" ([09:41] Bruno, [09:41] Diego).

## Consequências

### Positivas

- **Curva de aprendizado zero.** Quem conhece `src/modules/orders/` sabe navegar `src/modules/webhooks/`.
- **Tratamento de erro consistente de graça.** Como as classes derivam de `AppError`, `src/middlewares/error.middleware.ts` formata as respostas do módulo sem nenhuma alteração ([09:29] Bruno).
- **Observabilidade uniforme.** Os logs do worker saem no mesmo formato Pino do resto da aplicação, com a mesma redação de campos sensíveis ([09:29] Bruno).
- **Menos superfície de revisão de segurança.** A Sofia revisa HMAC e geração de secret, não uma stack paralela ([09:46] Sofia).

### Negativas

- **Acoplamento do módulo de pedidos à função de publicação.** `changeStatus` passa a chamar `publishWebhookEvent` dentro da sua transação, criando uma dependência direta de `order.service.ts` para o módulo de webhooks ([09:40] Bruno). É um acoplamento aceito, porque a atomicidade entre mudança de status e registro do evento é justamente o que [ADR-001](ADR-001-outbox-no-mysql.md) exige.
- **Convenções existentes viram restrição.** Decisões futuras do módulo de webhooks ficam limitadas ao que a codebase já suporta. Por exemplo, UUID como PK é mantido por consistência mesmo que auto-incremento pudesse ser mais eficiente para uma tabela de fila ([09:51] Larissa).
- **O middleware de erro não distingue contexto de worker.** Ele foi desenhado para o ciclo de requisição HTTP. O worker não tem `req` nem `res`, então o tratamento de erro no processamento precisa ser feito com log direto via Pino, sem depender do middleware ([09:29] Bruno).
