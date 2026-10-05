# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Webhooks de notificação de mudança de status de pedidos |
| **Status** | Pronto para implementação |
| **Data** | 2026-10-05 |
| **Público-alvo** | Time de engenharia (Pedidos e Plataforma) |
| **Documentos relacionados** | [RFC-001](RFC.md) · [PRD](PRD.md) · [ADRs](adrs/) · [TRACKER](TRACKER.md) |

> **Como ler este documento.** O [RFC](RFC.md) fecha a abordagem e as alternativas; os [ADRs](adrs/) registram cada decisão isolada com seu trade-off; **este FDD é o "como construir"**. Aqui estão os fluxos, o esquema de dados, os contratos de API e os pontos exatos de integração com o código existente.

---

## Contexto e motivação técnica

A aplicação é um OMS em Node.js + TypeScript, com Express, Prisma e MySQL. Não existe hoje **nenhum** mecanismo de evento, fila ou notificação externa. O único ponto do sistema em que o status de um pedido muda é `changeStatus`, em `src/modules/orders/order.service.ts`, que executa dentro de uma transação interativa do Prisma (`this.prisma.$transaction(async (tx) => { ... })`) três efeitos: `tx.order.update`, `tx.orderStatusHistory.create` e débito ou reposição de `stock_quantity` via `debitStock` / `replenishStock`.

A feature precisa notificar clientes B2B quando esse status muda, sem comprometer a transação existente e sem permitir o caso em que o status muda e o evento não é registrado ([09:40] Bruno).

A restrição técnica estruturante é que **não há listener nativo de banco no MySQL** ([09:09] Diego) e que o time **não quer subir infraestrutura nova** ([09:07] Diego). Isso empurra a solução para o padrão outbox sobre o MySQL existente, com consumo por polling — decisões registradas em [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).

---

## Objetivos técnicos

| # | Objetivo | Origem |
| --- | --- | --- |
| OT-1 | Garantir atomicidade entre mudança de status e registro do evento — sem janela de inconsistência | [09:06] Diego, [09:41] Diego |
| OT-2 | Não adicionar I/O de rede ao caminho crítico da transação de pedidos | [09:04] Bruno |
| OT-3 | Entregar eventos com latência máxima de 2 segundos no caminho feliz | [09:09] Diego, [09:10] Larissa |
| OT-4 | Sobreviver a indisponibilidade do cliente por até ~15 horas sem perder o evento | [09:17] Diego |
| OT-5 | Permitir que o cliente valide origem e integridade de cada requisição recebida | [09:19] Sofia |
| OT-6 | Reaproveitar integralmente os padrões transversais existentes (erro, log, validação, autorização) | [09:30] Larissa |
| OT-7 | Manter a outbox enxuta, preservando o desempenho da leitura por polling | [09:18] Diego |

---

## Escopo e exclusões

### Incluso

- CRUD de configuração de webhook por customer, com filtro de status por endpoint ([09:31]–[09:33] Marcos, Bruno).
- Rotação de secret com grace period de 24 horas ([09:21] Sofia).
- Inserção transacional do evento na outbox a partir de `changeStatus` ([09:40] Bruno).
- Worker em processo separado com polling de 2 segundos ([09:09], [09:11] Diego, Larissa).
- Retry com backoff exponencial em 5 tentativas ([09:17] Diego).
- DLQ em tabela separada com replay manual via endpoint admin ([09:18] Diego).
- Assinatura HMAC-SHA256 com secret por endpoint ([09:20], [09:21] Sofia).
- Histórico de entregas por webhook ([09:34] Marcos).

### Exclusões

| Item excluído | Motivo | Origem |
| --- | --- | --- |
| Notificação por e-mail ao cliente em caso de falhas consecutivas | Explicitamente adiado para próxima fase, depois de medir o impacto | [09:37] Larissa |
| Dashboard visual para o cliente | Projeto separado do time de frontend; esta fase entrega apenas endpoints | [09:39] Larissa, [09:40] Larissa |
| Rate limiting de saída por cliente | Não entra nesta fase; observar e decidir depois | [09:39] Diego, [09:39] Larissa |
| Arquivamento das linhas entregues da outbox após ~30 dias | Mencionado como necessário, mas fora do escopo desta feature | [09:08] Diego |
| Webhooks inbound (cliente → plataforma) | Fluxo é somente outbound | [09:02] Sofia, [09:02] Marcos |
| Garantia de ordering global entre pedidos | Nunca foi pedido pelos clientes; só ordering por `order_id` e enquanto o worker for único | [09:13] Larissa, [09:14] Marcos |
| Escala horizontal do worker mantendo ordering | Adiado como "problema do futuro" | [09:13] Diego |

---

## Modelo de dados

Modelagem proposta. Segue o padrão do `prisma/schema.prisma` existente: PK `String @id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, índices explícitos.

> **Nota de proveniência.** Os **campos e comportamentos** abaixo vêm da transcrição (colunas citadas em [09:21] Bruno, estados em [09:08] Diego, payload em [09:43] Diego, snapshot em [09:52] Larissa, UUID em [09:51] Larissa). A **estrutura exata das tabelas** é uma decisão de modelagem deste FDD, a ser validada em revisão.

```prisma
enum WebhookEventStatus {
  PENDING      // aguardando envio
  PROCESSING   // em envio pelo worker
  DELIVERED    // entregue com sucesso
  FAILED       // esgotou tentativas, aguardando movimentação para DLQ
}

/// Configuração de um endpoint de webhook de um customer.
/// Colunas url + secret + customer_id + estado ativo: [09:21] Bruno
model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)   // HTTPS obrigatório: [09:23] Sofia
  secret                  String    @db.VarChar(255)    // gerada pela plataforma: [09:31] Marcos
  previousSecret          String?   @db.VarChar(255)    // grace period 24h: [09:21] Sofia
  previousSecretExpiresAt DateTime?
  subscribedStatuses      Json                          // filtro de eventos: [09:33] Marcos
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer  Customer          @relation(fields: [customerId], references: [id])
  outbox    WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

/// Outbox transacional. Inserida na MESMA transação de changeStatus: [09:06] Diego
model WebhookOutbox {
  id            String             @id @default(uuid()) @db.Char(36)
  eventId       String             @unique @db.Char(36)  // X-Event-Id: [09:25] Diego
  webhookId     String             @db.Char(36)
  orderId       String             @db.Char(36)
  customerId    String             @db.Char(36)
  eventType     String             @db.VarChar(64)       // "order.status_changed": [09:43] Diego
  payload       Json                                     // snapshot renderizado: [09:52] Larissa
  status        WebhookEventStatus @default(PENDING)     // [09:08] Diego
  attempts      Int                @default(0)           // [09:17] Diego
  nextAttemptAt DateTime           @default(now())       // backoff: [09:17] Diego
  lastError     String?            @db.Text
  createdAt     DateTime           @default(now())
  updatedAt     DateTime           @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([status, nextAttemptAt])  // leitura do worker: [09:08] Diego
  @@index([createdAt])              // ordering por created_at: [09:12] Diego
  @@index([orderId])
  @@map("webhook_outbox")
}

/// Registro de cada tentativa de entrega. Sustenta GET /webhooks/:id/deliveries,
/// que precisa devolver sucesso/falha, payload, response e tempo de resposta: [09:34] Marcos
model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  orderId        String   @db.Char(36)
  success        Boolean
  httpStatus     Int?
  responseBody   String?  @db.Text
  durationMs     Int
  attempt        Int
  errorCode      String?  @db.VarChar(64)
  payload        Json
  createdAt      DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

/// DLQ. Tabela separada com payload, motivo da falha e timestamp: [09:18] Diego
model WebhookDeadLetter {
  id          String   @id @default(uuid()) @db.Char(36)
  eventId     String   @db.Char(36)
  webhookId   String   @db.Char(36)
  orderId     String   @db.Char(36)
  customerId  String   @db.Char(36)
  payload     Json
  failureReason String @db.Text
  attempts    Int
  failedAt    DateTime @default(now())
  replayedAt  DateTime?
  replayedById String? @db.Char(36)   // auditoria do replay: [09:36] Sofia

  @@index([webhookId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

---

## Fluxos detalhados

### Fluxo 1: Inserção na outbox dentro da transação de mudança de status

Este é o ponto de integração mais sensível da feature.

```
PATCH /api/v1/orders/:id/status
  │
  ▼
OrderService.changeStatus(id, input, userId)
  └─ this.prisma.$transaction(async (tx) => {
       1. tx.order.findUnique  ────────────► NotFoundError('Order')
       2. valida from === to ──────────────► ConflictError('INVALID_STATUS_TRANSITION')
       3. canTransition(from, to) ─────────► InvalidStatusTransitionError
       4. shouldDebitStock  → debitStock(tx, items)     ──► InsufficientStockError
          shouldReplenishStock → replenishStock(tx, items)
       5. tx.order.update({ status: to })
       6. tx.orderStatusHistory.create({ ... })
       7. ★ publishWebhookEvent(tx, order, from, to)     ◄── NOVO
       8. tx.order.findUnique (refresh) → return
     })
```

**Passo 7 — detalhamento.** `publishWebhookEvent` recebe o `tx` da transação corrente, não um repository ([09:41] Bruno, [09:41] Diego). Comportamento:

1. Busca os `WebhookEndpoint` do `customerId` do pedido com `active = true`.
2. **Filtra por interesse:** mantém apenas endpoints cujo `subscribedStatuses` contenha o `toStatus`. Se a lista ficar vazia, **nenhuma linha é inserida** — economia de linha na tabela ([09:34] Bruno, [09:34] Diego).
3. Para cada endpoint elegível, monta o payload **já renderizado** — snapshot do estado atual ([09:52] Larissa) — e insere uma linha em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`, `eventId` = novo UUID ([09:25] Diego, [09:51] Larissa).

**Garantia de atomicidade.** Se qualquer passo de 1 a 7 falhar, a transação inteira sofre rollback, incluindo a mudança de status. Se a transação commitar, o evento está persistido. Não existe caminho em que o status mude sem o evento ([09:06] Diego, [09:41] Diego).

**Consequência de custo:** o passo 7 é CPU em memória e uma inserção SQL. Não há I/O de rede, então o requisito OT-2 é preservado ([09:04] Bruno).

### Fluxo 2: Processamento pelo worker

Entry point `src/worker.ts`, lógica em `src/modules/webhooks/webhook.processor.ts` ([09:11] Larissa, [09:28] Bruno). Processo Node separado, com instância própria de `PrismaClient` sobre a mesma `DATABASE_URL` ([09:30] Bruno).

```
loop (a cada WORKER_POLL_INTERVAL_MS, default 2000ms):     // [09:09] Diego
  │
  ├─ SELECT * FROM webhook_outbox
  │    WHERE status = 'PENDING'
  │      AND nextAttemptAt <= NOW()
  │    ORDER BY createdAt ASC                              // ordering: [09:12] Diego
  │    LIMIT WEBHOOK_BATCH_SIZE                            // batch pequeno: [09:08] Diego
  │
  ├─ para cada evento:
  │    1. marca status = 'PROCESSING'                      // evita processamento duplo
  │    2. carrega WebhookEndpoint (url, secret, previousSecret)
  │    3. se endpoint inativo → move para DLQ (WEBHOOK_ENDPOINT_INACTIVE)
  │    4. valida tamanho do payload ≤ 64KB                 // [09:24] Larissa
  │    5. assina: HMAC-SHA256(secret, rawBody)             // [09:20] Sofia
  │    6. POST url, timeout 10s                            // [09:42] Diego
  │    7. registra WebhookDelivery (sucesso/falha, status, response, duração)
  │    8. sucesso (2xx) → status = 'DELIVERED'
  │       falha          → attempts++, calcula nextAttemptAt
  │                        se attempts >= 5 → move para DLQ
  │
  └─ sleep(WORKER_POLL_INTERVAL_MS)
```

**Ordenação.** Lendo em `ORDER BY createdAt ASC` com um único worker, eventos de um mesmo pedido são processados na ordem em que foram gerados, e o cliente os recebe nessa ordem ([09:12] Diego). Isso vale **apenas por `order_id` e enquanto houver um único worker** — é limitação conhecida, não garantia ([09:13] Larissa). Ver [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) e a QA-2 do [RFC](RFC.md#qa-2--escalar-o-worker-mantendo-ordering).

**Encerramento gracioso.** `src/worker.ts` deve seguir o padrão já existente em `src/server.ts`: registrar handlers de `SIGINT` e `SIGTERM`, interromper o loop de polling, aguardar o evento em processamento terminar e então chamar `prisma.$disconnect()` antes de `process.exit(0)`.

### Fluxo 3: Retry e backoff

Política registrada em [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md). Máximo de **5 tentativas**, com a progressão abaixo ([09:17] Diego):

| Tentativa | Espera antes da tentativa | Tempo acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 | imediata (primeiro envio) | 0 |
| 2 | 1 minuto | ~1 min |
| 3 | 5 minutos | ~6 min |
| 4 | 30 minutos | ~36 min |
| 5 | 2 horas | ~2h36min |
| — | 12 horas | ~15h (janela total) |

Cálculo do próximo agendamento: `nextAttemptAt = now() + backoff[attempts]`, gravado na própria linha da outbox. O worker só considera eventos com `nextAttemptAt <= NOW()`, então um evento em backoff não é reprocessado antes da hora.

Uma falha é contabilizada para **qualquer** um destes casos: timeout de 10s, resposta HTTP fora da faixa 2xx, ou erro de conexão/DNS.

### Fluxo 4: DLQ e replay

Esgotadas as 5 tentativas, o evento é movido para `webhook_dead_letter` ([09:18] Diego):

```
transação:
  1. INSERT webhook_dead_letter (payload, failureReason, attempts, failedAt, ...)
  2. DELETE webhook_outbox WHERE id = <evento>
```

O `eventId` é preservado, o que mantém a deduplicação do cliente consistente caso o evento seja reenviado depois ([09:25] Diego).

**Replay manual** — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego), com role `ADMIN` ([09:36] Sofia) e registro de quem executou ([09:36] Sofia):

```
transação:
  1. SELECT ... FROM webhook_dead_letter WHERE id = :id  ──► WEBHOOK_DELIVERY_NOT_FOUND
  2. INSERT webhook_outbox (mesmo eventId, status = PENDING, attempts = 0, nextAttemptAt = now())
  3. UPDATE webhook_dead_letter SET replayedAt = now(), replayedById = <userId>
  4. logger.info({ deadLetterId, eventId, replayedById: userId }, 'webhook_dlq_replayed')  // auditoria
```

O replay é **manual e deliberado**. Não há reprocessamento automático da DLQ — ver [ADR-007](adrs/ADR-007-dlq-em-tabela-separada.md).

---

## Contratos públicos

Base path: `/api/v1`. Todos os endpoints exigem `Authorization: Bearer <token>`.

### Convenções de erro

Formato de erro herdado do `error.middleware.ts` existente:

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found",
    "details": { "id": "..." }
  }
}
```

### C1 — `POST /api/v1/webhooks` — Criar configuração de webhook

Autenticado com JWT comum ([09:32] Larissa). A secret é **gerada pela plataforma** e devolvida **apenas nesta resposta** ([09:31] Marcos).

**Request**

```http
POST /api/v1/webhooks
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "customerId": "3f8a1c2e-9b4d-4a71-8e6f-2d5c7b0a1f93",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Response — 201 Created**

```json
{
  "id": "b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20",
  "customerId": "3f8a1c2e-9b4d-4a71-8e6f-2d5c7b0a1f93",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "secret": "whsec_9f4b2c7e8a1d3f6b5e0c9a2d4f7b8e1c",
  "createdAt": "2026-10-05T14:32:07.412Z"
}
```

**Erros:** `WEBHOOK_INVALID_URL` (400), `WEBHOOK_CUSTOMER_NOT_FOUND` (404), `WEBHOOK_INVALID_STATUS_FILTER` (400), `VALIDATION_ERROR` (400).

**Semântica:** o campo `secret` é devolvido **somente na criação e na rotação**. Nenhum `GET` subsequente o retorna.

---

### C2 — `GET /api/v1/webhooks` — Listar webhooks de um customer

**Request**

```http
GET /api/v1/webhooks?customerId=3f8a1c2e-9b4d-4a71-8e6f-2d5c7b0a1f93&page=1&pageSize=20
Authorization: Bearer <token>
```

**Response — 200 OK**

```json
{
  "data": [
    {
      "id": "b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20",
      "customerId": "3f8a1c2e-9b4d-4a71-8e6f-2d5c7b0a1f93",
      "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
      "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-10-05T14:32:07.412Z",
      "updatedAt": "2026-10-05T14:32:07.412Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**Erros:** `VALIDATION_ERROR` (400). Envelope `data` + `pagination` conforme `src/shared/http/response.ts` (`paginated`), já usado por `OrderService.list`.

---

### C3 — `PATCH /api/v1/webhooks/:id` — Editar configuração

Permite alterar `url`, `subscribedStatuses` e `active` ([09:33] Bruno).

**Request**

```http
PATCH /api/v1/webhooks/b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response — 200 OK**

```json
{
  "id": "b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20",
  "customerId": "3f8a1c2e-9b4d-4a71-8e6f-2d5c7b0a1f93",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-10-05T15:04:55.108Z"
}
```

**Erros:** `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INVALID_URL` (400), `WEBHOOK_INVALID_STATUS_FILTER` (400).

**Semântica:** alterar `subscribedStatuses` afeta **apenas eventos futuros**. Eventos já inseridos na outbox não são reprocessados nem descartados — consequência direta do snapshot em [ADR-008](adrs/ADR-008-snapshot-do-payload-na-insercao.md).

---

### C4 — `DELETE /api/v1/webhooks/:id` — Remover configuração

**Request**

```http
DELETE /api/v1/webhooks/b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20
Authorization: Bearer <token>
```

**Response — 204 No Content**

**Erros:** `WEBHOOK_NOT_FOUND` (404).

**Semântica:** remoção lógica recomendada (`active = false`) para preservar o histórico de entregas associado. Eventos já pendentes na outbox para esse endpoint são movidos para a DLQ com `WEBHOOK_ENDPOINT_INACTIVE`.

---

### C5 — `POST /api/v1/webhooks/:id/rotate-secret` — Rotacionar secret

A secret antiga permanece válida por **24 horas** em paralelo, para o cliente migrar seus sistemas ([09:21] Sofia).

**Request**

```http
POST /api/v1/webhooks/b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20/rotate-secret
Authorization: Bearer <token>
```

**Response — 200 OK**

```json
{
  "id": "b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20",
  "secret": "whsec_1a8c3e5f7b9d2a4c6e8f0b1d3a5c7e9f",
  "previousSecretExpiresAt": "2026-10-06T15:10:00.000Z"
}
```

**Erros:** `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INACTIVE_ENDPOINT` (409).

**Semântica:** durante a janela de grace period o worker **assina com a secret nova**; o cliente deve aceitar ambas ao validar. Depois de `previousSecretExpiresAt` a secret anterior é invalidada ([09:21] Sofia).

---

### C6 — `GET /api/v1/webhooks/:id/deliveries` — Histórico de entregas

Devolve as últimas entregas com sucesso/falha, payload, response e tempo de resposta ([09:34] Marcos).

**Request**

```http
GET /api/v1/webhooks/b7d2e4f1-0a3c-4d58-9e12-6c8b5a4f7d20/deliveries?page=1&pageSize=100
Authorization: Bearer <token>
```

**Response — 200 OK**

```json
{
  "data": [
    {
      "id": "c9e1f3a5-7b2d-4e60-8a14-3f5b7d9c1e20",
      "eventId": "5d7f9a1c-3e5b-4d72-9f80-1a3c5e7b9d20",
      "orderId": "8a2c4e6f-1b3d-4f59-8e70-2c4a6e8b0d13",
      "success": true,
      "httpStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 143,
      "attempt": 1,
      "createdAt": "2026-10-05T15:22:41.882Z"
    },
    {
      "id": "d0f2a4b6-8c3e-4f71-9b25-4a6c8e0d2f31",
      "eventId": "6e8a0b2d-4f6c-5e83-a091-2b4d6f8a0c31",
      "orderId": "9b3d5f7a-2c4e-4a60-9f81-3d5b7f9c1e24",
      "success": false,
      "httpStatus": 503,
      "responseBody": "{\"error\":\"service unavailable\"}",
      "durationMs": 10012,
      "attempt": 2,
      "errorCode": "WEBHOOK_DELIVERY_HTTP_ERROR",
      "createdAt": "2026-10-05T15:18:02.331Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

**Erros:** `WEBHOOK_NOT_FOUND` (404).

**Semântica:** `pageSize` máximo de 100, alinhado ao padrão de `listOrdersQuerySchema` em `src/modules/orders/order.schemas.ts` (`z.coerce.number().int().min(1).max(100)`). O histórico é a evidência que o cliente usa para confirmar que os eventos foram entregues ([09:34] Marcos).

---

### C7 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — Replay de DLQ

Exige role `ADMIN` ([09:36] Sofia, [09:36] Larissa), reaproveitando `requireRole` ([09:36] Larissa). A ação é registrada com o autor para auditoria ([09:36] Sofia).

**Request**

```http
POST /api/v1/admin/webhooks/dead-letter/e4b6d8f0-2a4c-4e60-8b71-5c7d9f1a3b42/replay
Authorization: Bearer <token-de-admin>
```

**Response — 200 OK**

```json
{
  "deadLetterId": "e4b6d8f0-2a4c-4e60-8b71-5c7d9f1a3b42",
  "eventId": "7f9b1d3e-5a7c-4f82-b103-3c5e7a9b1d42",
  "outboxId": "f5c7e9a1-3b5d-4f70-9c82-6d8e0a2b4c53",
  "replayedAt": "2026-10-05T16:02:19.774Z",
  "replayedById": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "status": "PENDING"
}
```

**Erros:** `WEBHOOK_DELIVERY_NOT_FOUND` (404), `FORBIDDEN` (403 — role insuficiente), `UNAUTHORIZED` (401).

**Semântica:** o replay recoloca o evento na outbox com `attempts = 0` e o **mesmo `eventId`**, preservando a deduplicação do cliente ([09:25] Diego). O evento volta a ser processado pelo worker no próximo ciclo de polling.

---

### Headers das requisições de saída (plataforma → cliente)

Enviados em todo POST para a URL do cliente ([09:44] Diego, [09:44] Sofia):

| Header | Valor | Origem |
| --- | --- | --- |
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento, gerado na inserção na outbox | [09:25] Diego, [09:44] Diego |
| `X-Signature` | `sha256=<hex>` — HMAC-SHA256 sobre o corpo do request | [09:20] Sofia, [09:22] Sofia |
| `X-Timestamp` | ISO 8601 do momento do envio, para o cliente detectar replay attack | [09:44] Diego |
| `X-Webhook-Id` | `id` do `WebhookEndpoint`, para o cliente saber qual cadastro originou o envio | [09:44] Sofia |

### Payload do evento

Campos definidos em [09:43] Diego. **Itens do pedido não são enviados** — o cliente consulta `GET /orders/:id` se precisar de detalhe ([09:43] Diego).

```json
{
  "event_id": "5d7f9a1c-3e5b-4d72-9f80-1a3c5e7b9d20",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-05T15:22:41.512Z",
  "order_id": "8a2c4e6f-1b3d-4f59-8e70-2c4a6e8b0d13",
  "order_number": "ORD-000482",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "3f8a1c2e-9b4d-4a71-8e6f-2d5c7b0a1f93",
  "total_cents": 24990
}
```

**Limite de tamanho:** 64 KB. Um payload acima disso **não é enviado** — é tratado como erro, não truncado ([09:24] Larissa, [09:24] Diego).

---

## Matriz de erros previstos

Todos os códigos do módulo usam o prefixo `WEBHOOK_` ([09:29] Larissa), seguindo o padrão de `SCREAMING_SNAKE_CASE` de `src/shared/errors/http-errors.ts` ([09:28] Bruno).

### Erros da API (retornados ao chamador)

| Código | HTTP | Classe base | Quando ocorre | Origem |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | `:id` de webhook inexistente em GET/PATCH/DELETE/rotate/deliveries | [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | 400 | `ValidationError` | URL cadastrada não é HTTPS | [09:23] Sofia, [09:28] Bruno |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `ValidationError` | Operação exige secret ativa e o endpoint não possui uma válida | [09:28] Bruno |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `NotFoundError` | `customerId` informado não existe | [09:31] Marcos |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `ValidationError` | `subscribedStatuses` vazio ou com valor fora do enum `OrderStatus` | [09:33] Marcos |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload do evento excede 64 KB | [09:24] Larissa |
| `WEBHOOK_INACTIVE_ENDPOINT` | 409 | `ConflictError` | Rotação de secret solicitada em endpoint inativo | [09:21] Sofia |
| `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | `NotFoundError` | `:id` inexistente na DLQ no replay | [09:18] Diego |
| `WEBHOOK_INVALID_STATUS_TRANSITION` | 409 | `ConflictError` | Transição de status inválida — **reaproveita** o código existente `INVALID_STATUS_TRANSITION` | [09:28] Bruno |

### Erros de entrega (registrados na outbox / delivery, não retornados via HTTP)

| Código | Quando ocorre | Ação | Origem |
| --- | --- | --- | --- |
| `WEBHOOK_DELIVERY_TIMEOUT` | Cliente não respondeu em 10 segundos | Retry | [09:42] Diego |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Resposta HTTP fora da faixa 2xx | Retry | [09:44] Diego |
| `WEBHOOK_DELIVERY_CONNECTION_ERROR` | Erro de conexão, DNS ou TLS | Retry | [09:19] Sofia |
| `WEBHOOK_ENDPOINT_INACTIVE` | Endpoint desativado ou removido antes do envio | DLQ direto, sem retry | [09:21] Sofia |
| `WEBHOOK_RETRIES_EXHAUSTED` | 5 tentativas esgotadas | DLQ | [09:15] Diego |

**Tratamento.** As classes de erro derivam de `AppError` (`src/shared/errors/app-error.ts`) e são capturadas por `src/middlewares/error.middleware.ts` **sem qualquer alteração** nesse arquivo ([09:29] Bruno). Os erros de entrega não passam pelo middleware — são registrados pelo worker via logger Pino, porque não existe ciclo requisição/resposta no processo do worker.

---

## Estratégias de resiliência

| Mecanismo | Valor | Justificativa | Origem |
| --- | --- | --- | --- |
| **Timeout HTTP** | 10 segundos | Cliente lento que não responde em 10s é tratado como falha e entra em retry | [09:42] Diego |
| **Retry** | 5 tentativas | Cobre janela longa sem deixar evento pendurado para sempre | [09:15] Diego |
| **Backoff** | 1m / 5m / 30m / 2h / 12h (~15h) | Absorve indisponibilidade de manutenção planejada de algumas horas | [09:17] Diego |
| **DLQ** | Tabela `webhook_dead_letter` | Ponto final: preserva evidência sem reprocessamento automático | [09:18] Diego |
| **Replay** | Manual, via endpoint ADMIN | Mantém decisão humana no loop de recuperação | [09:18] Diego, [09:36] Sofia |
| **Tamanho máximo de payload** | 64 KB, erro se exceder | Nenhum evento real chega perto; acima disso há algo errado | [09:24] Diego, [09:24] Larissa |
| **Isolamento de processo** | Worker separado da API | Reinício da API não interrompe entrega | [09:11] Diego |
| **Atomicidade** | Inserção na outbox dentro da transação | Elimina a janela de inconsistência | [09:06] Diego |

**Fallback.** Não existe fallback alternativo de entrega nesta fase: o evento ou é entregue por HTTP, ou vai para a DLQ. A notificação por e-mail em caso de falha foi explicitamente adiada ([09:37] Larissa).

**Ponto de atenção — retry vs. ordering.** Um evento em backoff pode ser ultrapassado por eventos mais novos do mesmo pedido, quebrando a ordenação justamente no caminho de falha. É consequência aceita da combinação entre [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) e [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md), e reforça a necessidade do `X-Event-Id` para o cliente reconciliar.

---

## Observabilidade

### Logs

O projeto já usa **Pino**, exportado como singleton em `src/shared/logger/index.ts`, com `redact` configurado para `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`, e `base: { service: 'order-management-api', env }` ([09:29] Bruno). O módulo de webhooks **não introduz ferramenta nova** ([09:29] Bruno).

Eventos de log propostos para o worker e o módulo:

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook_event_enqueued` | `info` | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus` |
| `webhook_delivery_attempt` | `info` | `eventId`, `webhookId`, `attempt`, `durationMs`, `httpStatus` |
| `webhook_delivery_failed` | `warn` | `eventId`, `attempt`, `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dlq` | `error` | `eventId`, `webhookId`, `attempts`, `failureReason` |
| `webhook_dlq_replayed` | `info` | `deadLetterId`, `eventId`, `replayedById` |
| `webhook_worker_started` / `webhook_worker_stopped` | `info` | `pollIntervalMs`, `batchSize` |

> **Atenção de segurança.** O campo `secret` e o header `X-Signature` **não devem aparecer em log**. Os `redact.paths` atuais cobrem `*.token` e `*.password`, mas não `*.secret` nem `x-signature` — a implementação precisa estender essa lista ou garantir que esses valores nunca sejam passados ao logger. Ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).

### Métricas

> **Proveniência.** Métricas e tracing **não foram discutidos na reunião**. O que segue é proposta deste FDD para tornar observáveis os comportamentos que **foram** decididos — em particular a DLQ, cuja saída depende de intervenção humana ([ADR-007](adrs/ADR-007-dlq-em-tabela-separada.md)), e o backoff, cuja janela de ~15h precisa ser visível. Sujeito a confirmação do time.

| Métrica | Tipo | Para que serve |
| --- | --- | --- |
| `webhook_outbox_pending_total` | gauge | Detecta acúmulo na outbox — sinal de worker parado ou cliente degradado |
| `webhook_delivery_duration_ms` | histogram | Acompanha a latência de entrega contra o teto de 10s de timeout |
| `webhook_delivery_total{result}` | counter | Taxa de sucesso/falha por endpoint |
| `webhook_retry_total{attempt}` | counter | Distribuição das tentativas — valida se a progressão de backoff é adequada |
| `webhook_dlq_total` | counter | **Métrica mais importante.** Sem ela, eventos falhos ficam invisíveis, já que não há saída automática da DLQ |
| `webhook_end_to_end_latency_ms` | histogram | Mede `createdAt` da outbox → resposta do cliente, contra a meta de 10s ([09:02] Marcos) |

### Tracing

Proposta: propagar um `trace_id` do enqueue ao envio. Como o worker é um **processo separado** ([09:11] Diego) e o processamento acontece minutos ou horas depois da inserção, o trace não pode ser um span contínuo — o `eventId` funciona como chave de correlação entre o log da transação de `changeStatus` e o log da tentativa de entrega. Isso permite reconstruir a jornada completa de um evento a partir do `X-Event-Id` ([09:25] Diego).

O projeto não possui instrumentação de tracing hoje; adotá-la seria decisão nova, não coberta pela reunião.

---

## Integração com o sistema existente

Esta seção nomeia os pontos exatos de acoplamento com o código base. **A alteração em código existente é mínima e localizada** — o restante é aditivo ([09:30] Larissa).

**Inventário de arquivos.** Para evitar ambiguidade entre o que já existe e o que a feature cria:

| Situação | Arquivos |
| --- | --- |
| **Existentes — alterados** | `src/modules/orders/order.service.ts` (uma chamada a `publishWebhookEvent` dentro da transação) · `src/config/env.ts` (novas variáveis) · `src/app.ts` e `src/routes/index.ts` (registro do módulo) · `prisma/schema.prisma` (novos modelos e relação reversa em `Customer`) |
| **Existentes — reutilizados sem alteração** | `src/shared/errors/app-error.ts` · `src/shared/errors/http-errors.ts` · `src/shared/errors/index.ts` · `src/middlewares/error.middleware.ts` · `src/middlewares/auth.middleware.ts` · `src/middlewares/validate.middleware.ts` · `src/shared/logger/index.ts` · `src/config/database.ts` · `src/shared/http/response.ts` · `src/modules/orders/order.status.ts` · `src/modules/orders/order.routes.ts` · `src/modules/orders/order.schemas.ts` · `src/modules/orders/order.repository.ts` · `src/modules/users/user.routes.ts` · `src/middlewares/request-logger.middleware.ts` |
| **Novos — a criar pela feature** | `src/worker.ts` (entry point do worker) · `src/modules/webhooks/webhook.controller.ts` · `src/modules/webhooks/webhook.service.ts` · `src/modules/webhooks/webhook.repository.ts` · `src/modules/webhooks/webhook.routes.ts` · `src/modules/webhooks/webhook.schemas.ts` · `src/modules/webhooks/webhook.processor.ts` · `src/modules/webhooks/webhook.errors.ts` · `src/modules/webhooks/webhook.crypto.ts` |

> Os caminhos da terceira linha **não existem hoje** — são os arquivos que a implementação deve criar. Os caminhos das duas primeiras linhas existem no repositório e foram verificados.

### 1. `src/modules/orders/order.service.ts` — a única alteração em código existente

O método `changeStatus` (linhas 126–179) é o ponto de integração crítico ([09:40] Bruno).

- A chamada `publishWebhookEvent(tx, order, from, to)` entra **dentro do callback de `this.prisma.$transaction`**, como último passo antes do `findUnique` de refresh.
- O `tx` passado é o mesmo `Prisma.TransactionClient` já usado por `debitStock` e `replenishStock` — o tipo `TxClient` já está declarado no topo do arquivo (`type TxClient = Prisma.TransactionClient`).
- A variável `order` disponível no escopo já contém `items` (o `findUnique` inicial inclui `{ items: true }`), e `from` / `to` já estão calculados.
- **Nenhuma alteração no construtor** de `OrderService`: a função recebe `tx`, não um repository injetado ([09:41] Diego). Isso evita tocar em `buildControllers` em `src/app.ts`.
- **Rollback é automático:** se `publishWebhookEvent` lançar, a exceção propaga para fora do callback do `$transaction`, o Prisma faz rollback e a mudança de status não persiste ([09:40] Bruno).

### 2. `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — extensão da taxonomia

As novas classes de erro ficam em `src/modules/webhooks/webhook.errors.ts` e seguem o padrão já estabelecido ([09:28] Bruno). O helper de assinatura fica em `src/modules/webhooks/webhook.crypto.ts`, isolado para permitir teste unitário do HMAC sem subir o worker:

- `WebhookNotFoundError extends NotFoundError` → `WEBHOOK_NOT_FOUND`
- `WebhookInvalidUrlError extends ValidationError` → `WEBHOOK_INVALID_URL`
- `WebhookSecretRequiredError extends ValidationError` → `WEBHOOK_SECRET_REQUIRED`
- `WebhookPayloadTooLargeError extends UnprocessableEntityError` → `WEBHOOK_PAYLOAD_TOO_LARGE`
- `WebhookInactiveEndpointError extends ConflictError` → `WEBHOOK_INACTIVE_ENDPOINT`

O modelo é `InsufficientStockError` em `http-errors.ts`, que estende `UnprocessableEntityError` e passa o código como segundo argumento. O construtor de `AppError` já aceita `(message, statusCode, errorCode, details?)`, então **a classe base não precisa mudar**. Novos erros devem ser reexportados pelo barrel `src/shared/errors/index.ts`, que hoje já reexporta `app-error.js` e `http-errors.js`.

### 3. `src/middlewares/error.middleware.ts` — reuso sem alteração

O middleware já trata `err instanceof AppError` e responde com `{ error: { code, message, details? } }` usando `err.statusCode` e `err.errorCode` ([09:29] Bruno). Como todos os erros do módulo derivam de `AppError`, **este arquivo não é alterado**. Erros de validação Zod continuam caindo no ramo `ZodError` do mesmo middleware.

### 4. `src/middlewares/auth.middleware.ts` — reuso de `requireRole`

O endpoint de replay de DLQ usa `requireRole('ADMIN')`, já exportado por este arquivo ([09:36] Larissa). O tipo `AuthUser` define `role: 'ADMIN' | 'OPERATOR'`, e `requireRole` lança `ForbiddenError('Insufficient permissions')` quando a role não bate — comportamento já correto para o requisito de [09:36] Sofia. O padrão de uso é o de `src/modules/users/user.routes.ts`, que aplica `authenticate, requireRole('ADMIN')` na mesma rota.

### 5. `src/middlewares/validate.middleware.ts` — reuso da validação Zod

Os schemas de webhook em `src/modules/webhooks/webhook.schemas.ts` são consumidos pelo `validate({ body, query, params })` existente. A validação de HTTPS na URL ([09:23] Sofia) é implementada como refinamento Zod no schema — `z.string().url().refine(u => u.startsWith('https://'))` — e portanto **não requer código novo de validação**, apenas o schema. O `validate` converte `ZodError` em `ValidationError('Validation failed', details)` com `{ path, message }[]`.

### 6. `src/config/env.ts` — novas variáveis de ambiente

O schema Zod de ambiente em `src/config/env.ts` valida `process.env` e chama `process.exit(1)` em caso de falha. Novas variáveis propostas:

```ts
WORKER_POLL_INTERVAL_MS: z.coerce.number().int().positive().default(2000),  // [09:09] Diego
WORKER_BATCH_SIZE: z.coerce.number().int().positive().default(20),          // [09:08] Diego
WEBHOOK_TIMEOUT_MS: z.coerce.number().int().positive().default(10000),      // [09:42] Diego
WEBHOOK_MAX_PAYLOAD_BYTES: z.coerce.number().int().positive().default(65536), // [09:24] Diego
WEBHOOK_SECRET_GRACE_PERIOD_HOURS: z.coerce.number().int().positive().default(24), // [09:21] Sofia
```

Os valores da reunião viram **defaults**, e não constantes hard-coded — assim a política de backoff e os limites são ajustáveis sem deploy de código.

### 7. `src/config/database.ts` — instância de PrismaClient para o worker

`src/config/database.ts` exporta `createPrismaClient()` e um singleton `prisma`. O worker importa `createPrismaClient()` para obter sua **própria instância**, com o mesmo `DATABASE_URL` ([09:30] Bruno). Reutilizar o singleton do módulo não é adequado porque `PrismaClient` é por processo e o worker é outro processo Node ([09:30] Bruno, [09:30] Diego).

### 8. `src/app.ts` e `src/routes/index.ts` — registro do módulo

O padrão de wiring é `buildControllers(prisma)` em `src/app.ts`, que instancia repository → service → controller por módulo e devolve o objeto `Controllers`, consumido por `buildApiRouter(controllers)` em `src/routes/index.ts`.

Para webhooks:

- `buildControllers` passa a instanciar `WebhookRepository`, `WebhookService` e `WebhookController` e a incluí-los no objeto `Controllers`.
- O tipo `Controllers` em `src/routes/index.ts` ganha a chave `webhooks`.
- `buildApiRouter` monta `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e, para o replay, uma rota administrativa sob `/admin/webhooks`.
- O router de webhooks segue o formato de `buildOrderRouter` em `src/modules/orders/order.routes.ts`: uma factory function que recebe o controller e aplica `router.use(authenticate)` no topo.

### 9. `src/server.ts` — modelo para `src/worker.ts`

`src/server.ts` é o modelo estrutural da nova entry point: `bootstrap()`, handler de `SIGINT`/`SIGTERM` que faz shutdown gracioso, `prisma.$disconnect()` e `process.exit(0)`, e um `bootstrap().catch()` que loga `bootstrap_failed` via `logger.fatal` e sai com código 1 ([09:11] Larissa). `src/worker.ts` replica esse esqueleto trocando `app.listen` pelo loop de polling.

### 10. `prisma/schema.prisma` — novos modelos

Os modelos da seção [Modelo de dados](#modelo-de-dados) são adicionados a `prisma/schema.prisma`, seguindo as convenções do arquivo: `@id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, `@@index` explícito para toda coluna usada em filtro. O enum `WebhookEventStatus` é novo e não conflita com `OrderStatus` nem `UserRole`. As relações com `Customer` exigem adicionar o campo reverso no modelo `Customer` existente.

---

## Critérios de aceite técnicos

| # | Critério | Verificação |
| --- | --- | --- |
| CA-1 | Mudança de status e inserção na outbox são atômicas: rollback de uma implica rollback da outra | Teste que força erro na inserção da outbox e confirma que `orders.status` não mudou |
| CA-2 | Evento é entregue ao cliente em menos de 10 segundos no caminho feliz | Teste ponta a ponta medindo de `PATCH /orders/:id/status` até o recebimento no servidor de teste |
| CA-3 | Falha na entrega dispara retry com a progressão 1m/5m/30m/2h/12h | Teste com cliente que responde 503 e relógio controlado |
| CA-4 | Após 5 falhas, o evento sai da outbox e aparece em `webhook_dead_letter` com payload e motivo | Inspeção das duas tabelas após esgotar tentativas |
| CA-5 | Replay recoloca o evento na outbox com o **mesmo** `eventId` | Comparação do `eventId` antes e depois do replay |
| CA-6 | Replay sem role `ADMIN` retorna 403 | Requisição com token de `OPERATOR` |
| CA-7 | Assinatura HMAC-SHA256 é verificável pelo cliente com a secret recebida | Cálculo independente do HMAC sobre o corpo e comparação com `X-Signature` |
| CA-8 | URL `http://` é recusada na criação com `WEBHOOK_INVALID_URL` | `POST /webhooks` com URL sem TLS |
| CA-9 | Secret antiga continua válida por 24h após rotação | Rotação e verificação de assinatura com a secret anterior dentro da janela |
| CA-10 | Status não subscrito por nenhum endpoint do customer não gera linha na outbox | Mudança de status e contagem de linhas em `webhook_outbox` |
| CA-11 | Payload acima de 64 KB gera erro e não é truncado | Evento forçado acima do limite |
| CA-12 | Worker não derruba a API e vice-versa | Reinício da API durante processamento; o worker continua entregando |
| CA-13 | Códigos de erro do módulo usam prefixo `WEBHOOK_` | Inspeção dos códigos nas respostas de erro |
| CA-14 | Nenhum endpoint retorna a `secret` fora de criação e rotação | `GET /webhooks` e `GET /webhooks/:id/deliveries` não contêm o campo |

---

## Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| **Regressão na transação de pedidos** por causa da inserção na outbox | Média | Alto | Inserção como último passo, sem I/O de rede; CA-1 cobre o rollback; revisão focada no diff de `order.service.ts` ([09:40] Bruno) |
| **Crescimento da outbox** degradando a leitura do worker | Média | Médio | Índices em `[status, nextAttemptAt]` e `createdAt`; batch pequeno ([09:08] Diego). Arquivamento de 30 dias fora de escopo — risco residual aceito ([09:08] Diego) |
| **Eventos presos na DLQ sem ninguém perceber** | Alta | Médio | Métrica `webhook_dlq_total` e alerta sobre volume; replay manual documentado ([09:18] Diego) |
| **Secret exposta em log** | Baixa | Alto | `redact.paths` estendido para `*.secret` e `x-signature`; revisão de segurança obrigatória antes do deploy ([09:46] Sofia) |
| **Worker único vira ponto único de falha** | Média | Médio | Reinício automático do processo; a limitação de ordering sob escala é conhecida e documentada ([09:13] Larissa) |
| **Cliente não implementa deduplicação** e processa eventos duplicados | Média | Médio | Documentação destacada no portal do desenvolvedor ([09:26] Marcos); `X-Event-Id` em header e no corpo ([09:25] Diego) |
| **Payload congelado no formato da época** após evolução do contrato | Baixa | Baixo | Consequência aceita do snapshot ([ADR-008](adrs/ADR-008-snapshot-do-payload-na-insercao.md)); exige janela de transição no cliente |

---

## Dependências e compatibilidade

| Dependência | Situação | Observação |
| --- | --- | --- |
| MySQL | Existente | Nenhuma migração de infraestrutura; apenas novas tabelas ([09:07] Diego) |
| Prisma / `PrismaClient` | Existente | Nova instância por processo no worker ([09:30] Bruno) |
| Express 4 | Existente | Rotas novas seguem `buildXRouter(controller)` |
| Pino | Existente | Reuso do singleton; possível extensão de `redact.paths` ([09:29] Bruno) |
| Zod | Existente | Schemas novos consumidos pelo `validate.middleware.ts` |
| `jsonwebtoken` | Existente | `requireRole('ADMIN')` reaproveitado ([09:36] Larissa) |
| `node:crypto` | Nativo | `createHmac('sha256', secret)` — **sem dependência nova** para HMAC ([09:20] Sofia) |
| Servidor de teste HTTP | Novo, apenas para testes | Necessário para os testes ponta a ponta de entrega |
| Notificação por e-mail | **Não é dependência** | Fora de escopo nesta fase ([09:37] Larissa) |
| Redis / broker | **Não é dependência** | Explicitamente descartado ([09:07] Diego) |

**Compatibilidade retroativa.** A feature é **puramente aditiva** para clientes existentes. Nenhum endpoint atual muda de contrato ou comportamento. A única alteração em código existente é a linha adicional na transação de `changeStatus` — invisível para consumidores da API de pedidos, exceto por um custo marginal de latência na transação.
