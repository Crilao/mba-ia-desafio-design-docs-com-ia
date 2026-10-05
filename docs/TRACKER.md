# Tracker de Rastreabilidade

Este documento mapeia **cada item registrado** no pacote de design docs à sua origem — a transcrição da reunião técnica (`TRANSCRICAO.md`) ou o código-fonte da aplicação.

**Como usar.** Para qualquer afirmação nos documentos, esta tabela responde "de onde veio isso?". Se um item não tem linha correspondente aqui, ele **não tem origem identificável** e não deveria estar na documentação.

| Legenda | Significado |
| --- | --- |
| **Fonte** | `TRANSCRICAO` (falado na reunião) ou `CODIGO` (existe no repositório) |
| **Localização** | Para `TRANSCRICAO`: `[hh:mm] Falante`. Para `CODIGO`: caminho do arquivo |

---

## PRD — Requisitos Funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e lista de status; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listar configurações de webhook de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar configuração de webhook existente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover configuração de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtrar eventos por status na geração; status não subscrito não gera evento | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotação de secret pela API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Consultar histórico de entregas com sucesso/falha, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Notificar mudança de status por HTTP POST com payload JSON | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Retry automático com backoff exponencial em até 5 tentativas | TRANSCRICAO | [09:15] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Mover para DLQ os eventos que esgotarem tentativas, preservando payload e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay manual de evento da DLQ por administrador | TRANSCRICAO | [09:18] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Assinar requisições com HMAC-SHA256 e secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Enviar identificador único e estável do evento para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Recusar cadastro de webhook com URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Restringir replay de DLQ a role ADMIN e auditar o autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Registrar cada tentativa de entrega com resultado | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-17 | docs/PRD.md | Restrição | CRUD de webhook autenticado com JWT do sistema; `customer_id` vem do body/path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-18 | docs/PRD.md | Requisito Funcional | CRUD de configuração acessível a qualquer role autenticada nesta fase | TRANSCRICAO | [09:37] Sofia |

## PRD — Requisitos Não Funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Intervalo de polling do worker de 2 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos na chamada HTTP ao cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64 KB; erro se exceder, sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre mudança de status e geração do evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordenação por `order_id` apenas, com worker único | TRANSCRICAO | [09:12] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Grace period de 24h na rotação de secret | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Nenhuma chamada de rede no caminho crítico da transação de pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Reuso de AppError, Pino, error middleware, Zod e requireRole | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-12 | docs/PRD.md | Restrição | Revisão de segurança obrigatória antes do deploy | TRANSCRICAO | [09:46] Sofia |

## PRD — Objetivos e Métricas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência de notificação abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Disponibilidade em produção até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Métrica | 3 de 3 clientes B2B migrados para webhook | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Janela de retry de ~15 horas antes de falha permanente | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Sem impacto perceptível na latência da transação de mudança de status | TRANSCRICAO | [09:04] Bruno |

## PRD — Escopo e Fora de Escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-ESC-01 | docs/PRD.md | Escopo | CRUD de configuração de webhook por customer | TRANSCRICAO | [09:33] Bruno |
| PRD-ESC-02 | docs/PRD.md | Escopo | Filtro de status por endpoint aplicado na geração | TRANSCRICAO | [09:33] Marcos |
| PRD-ESC-03 | docs/PRD.md | Escopo | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-ESC-04 | docs/PRD.md | Escopo | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-ESC-05 | docs/PRD.md | Escopo | Notificação HTTP com payload JSON e headers de segurança | TRANSCRICAO | [09:44] Diego |
| PRD-ESC-06 | docs/PRD.md | Escopo | Assinatura HMAC-SHA256 por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-ESC-07 | docs/PRD.md | Escopo | Retry com backoff e DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-ESC-08 | docs/PRD.md | Escopo | Replay manual de DLQ por administrador | TRANSCRICAO | [09:18] Diego |
| PRD-ESC-09 | docs/PRD.md | Escopo | Histórico de entregas consultável | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-10 | docs/PRD.md | Escopo | At-least-once com identificador de evento | TRANSCRICAO | [09:24] Diego |
| PRD-ESC-11 | docs/PRD.md | Escopo | Exigência de HTTPS na URL cadastrada | TRANSCRICAO | [09:23] Sofia |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | Notificação por e-mail em caso de falhas repetidas — adiada para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Dashboard visual — projeto separado do time de frontend | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Arquivamento das linhas entregues da outbox após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Webhooks inbound — fluxo é somente outbound | TRANSCRICAO | [09:02] Sofia |
| PRD-OOS-06 | docs/PRD.md | Fora de Escopo | Garantia de ordering global entre pedidos | TRANSCRICAO | [09:14] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de Escopo | Escala horizontal do worker mantendo ordenação — adiada | TRANSCRICAO | [09:13] Diego |

## PRD — Riscos e Critérios de Aceitação

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RISK-01 | docs/PRD.md | Risco | Perda do cliente Atlas por atraso; sinalizou migração para concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Regressão na mudança de status de pedidos em produção | TRANSCRICAO | [09:04] Bruno |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente não implementa deduplicação e processa eventos duplicados | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-04 | docs/PRD.md | Risco | Vazamento de secret do lado do cliente — já ocorreu antes | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Eventos presos na DLQ sem detecção, por não haver e-mail nem reprocesso automático | TRANSCRICAO | [09:18] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Crescimento da outbox degradando a leitura do worker | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-07 | docs/PRD.md | Risco | Worker único como ponto único de falha | TRANSCRICAO | [09:11] Diego |
| PRD-CA-01 | docs/PRD.md | Critério de Aceite | Cliente cadastra webhook e recebe a secret gerada | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de Aceite | Mudança de status entregue em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-03 | docs/PRD.md | Critério de Aceite | Status não subscrito não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-04 | docs/PRD.md | Critério de Aceite | Indisponibilidade temporária do cliente não perde o evento | TRANSCRICAO | [09:16] Diego |
| PRD-CA-05 | docs/PRD.md | Critério de Aceite | Evento que falha 5 vezes é preservado na DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-CA-06 | docs/PRD.md | Critério de Aceite | Administrador reprocessa evento da DLQ com sucesso | TRANSCRICAO | [09:18] Diego |
| PRD-CA-07 | docs/PRD.md | Critério de Aceite | Usuário sem role ADMIN não reprocessa DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de Aceite | Cliente valida a assinatura com a secret recebida | TRANSCRICAO | [09:20] Sofia |
| PRD-CA-09 | docs/PRD.md | Critério de Aceite | Rotação de secret sem interrupção de entrega | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-10 | docs/PRD.md | Critério de Aceite | Cliente consulta histórico das últimas entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-11 | docs/PRD.md | Critério de Aceite | Cadastro com URL `http://` é recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-12 | docs/PRD.md | Critério de Aceite | Mesmo `X-Event-Id` em reenvios, permitindo deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-CA-13 | docs/PRD.md | Critério de Aceite | Operação normal de pedidos não é impactada | TRANSCRICAO | [09:04] Bruno |

---

## RFC

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-META-01 | docs/RFC.md | Metadado | Revisores do RFC são os cinco participantes da reunião técnica | TRANSCRICAO | [09:00] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Três clientes B2B fizeram pedido formal: Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Clientes fazem polling no `GET /orders`, tornando a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-03 | docs/RFC.md | Restrição | Não pode existir status alterado sem evento correspondente | TRANSCRICAO | [09:40] Bruno |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox transacional no MySQL, evento inserido na mesma transação | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o tx corrente | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Decisão | Worker em `src/worker.ts` com script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão | Processamento em `webhook.processor.ts` dentro do módulo | TRANSCRICAO | [09:28] Bruno |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono em `changeStatus` — cliente lento travaria mudanças de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / broker externo — overengineering para um time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco notificando worker — MySQL não tem listener nativo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Garantia exactly-once — exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Retry indefinido — evento ficaria pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Retry com 3 tentativas — cobriria janela curta demais | TRANSCRICAO | [09:16] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | Escalar worker mantendo ordering — particionar por `order_id` ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Retenção e arquivamento das linhas entregues da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em Aberto | Endurecer autorização do CRUD no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Estimativa de 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| RFC-IMP-02 | docs/RFC.md | Impacto | Atlas espera entrega até fim de novembro | TRANSCRICAO | [09:45] Marcos |

---

## FDD — Fluxos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserção na outbox como último passo da transação de `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Se a inserção na outbox falhar, toda a transação sofre rollback | TRANSCRICAO | [09:41] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Filtro de status aplicado na inserção, economizando linhas | TRANSCRICAO | [09:34] Bruno |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Worker lê pendentes em ordem de `created_at`, em batch pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Worker usa instância própria de PrismaClient, mesmo banco | TRANSCRICAO | [09:30] Bruno |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | Cálculo do próximo agendamento por `nextAttemptAt` com backoff | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-07 | docs/FDD.md | Fluxo | Movimentação transacional da outbox para a DLQ ao esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-08 | docs/FDD.md | Fluxo | Replay recoloca na outbox com o mesmo `eventId` e `attempts = 0` | TRANSCRICAO | [09:25] Diego |

## FDD — Contratos Públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks` — cria configuração e devolve a secret gerada | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/webhooks?customerId=` — lista webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` — edita URL, status subscritos e estado ativo | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` — remove configuração | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /api/v1/webhooks/:id/rotate-secret` — rotaciona com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` — histórico com sucesso/falha, payload, response e duração | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ, role ADMIN | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Autenticação de todos os endpoints por JWT Bearer do sistema | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-09 | docs/FDD.md | Restrição | `secret` só é devolvida na criação e na rotação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-10 | docs/FDD.md | Restrição | Paginação do histórico com `pageSize` máximo de 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-HEADER-01 | docs/FDD.md | Contrato | Header `X-Event-Id` com UUID do evento | TRANSCRICAO | [09:25] Diego |
| FDD-HEADER-02 | docs/FDD.md | Contrato | Header `X-Signature` com o HMAC-SHA256 do corpo | TRANSCRICAO | [09:20] Sofia |
| FDD-HEADER-03 | docs/FDD.md | Contrato | Header `X-Timestamp` para o cliente detectar replay attack | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-04 | docs/FDD.md | Contrato | Header `X-Webhook-Id` para identificar qual cadastro originou o envio | TRANSCRICAO | [09:44] Sofia |
| FDD-HEADER-05 | docs/FDD.md | Contrato | `Content-Type: application/json` | TRANSCRICAO | [09:44] Diego |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload JSON com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-02 | docs/FDD.md | Restrição | Itens do pedido **não** vão no payload, para não inflar | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-03 | docs/FDD.md | Contrato | `event_type` com valor `order.status_changed` | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-04 | docs/FDD.md | Contrato | `timestamp` em formato ISO 8601 | TRANSCRICAO | [09:43] Diego |

## FDD — Erros, Resiliência e Observabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Todos os códigos do módulo usam prefixo `WEBHOOK_` | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-05 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (422) — payload acima de 64 KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-06 | docs/FDD.md | Erro | `WEBHOOK_ENDPOINT_INACTIVE` — endpoint desativado antes do envio, vai direto para DLQ | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-07 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` — cliente não respondeu em 10s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-08 | docs/FDD.md | Erro | `WEBHOOK_RETRIES_EXHAUSTED` — 5 tentativas esgotadas, move para DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-09 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_NOT_FOUND` — `:id` inexistente na DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-10 | docs/FDD.md | Erro | `WEBHOOK_INVALID_STATUS_FILTER` — lista de status vazia ou inválida | TRANSCRICAO | [09:33] Marcos |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Timeout HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | Progressão de backoff 1m / 5m / 30m / 2h / 12h | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-03 | docs/FDD.md | Decisão | Janela total de ~15 horas entre primeira falha e última tentativa | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-04 | docs/FDD.md | Decisão | Máximo de 5 tentativas antes de considerar falha permanente | TRANSCRICAO | [09:15] Diego |
| FDD-RESIL-05 | docs/FDD.md | Trade-off | Evento em backoff pode ser ultrapassado por eventos mais novos do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-01 | docs/FDD.md | Decisão | Reuso do logger Pino existente, sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Restrição | Secret e `X-Signature` não devem aparecer em log | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Decisão | Log de auditoria do replay com o identificador de quem executou | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-04 | docs/FDD.md | Proposta | Métricas e tracing propostos pelo FDD — não discutidos na reunião | CODIGO | src/shared/logger/index.ts |
| FDD-DEP-01 | docs/FDD.md | Restrição | Nenhuma dependência nova; HMAC via `node:crypto` nativo | TRANSCRICAO | [09:20] Sofia |
| FDD-DEP-02 | docs/FDD.md | Restrição | Redis/broker não é dependência — explicitamente descartado | TRANSCRICAO | [09:07] Diego |

## FDD — Integração com o Sistema Existente

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-INT-01 | docs/FDD.md | Integração | Chamada a `publishWebhookEvent` dentro do `$transaction` de `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reuso do tipo `TxClient` (`Prisma.TransactionClient`) já declarado no service | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas classes de erro derivam de `AppError` / `NotFoundError` / `ValidationError` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | `AppError` já aceita `(message, statusCode, errorCode, details?)` — base não muda | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Novos erros reexportados pelo barrel de erros | CODIGO | src/shared/errors/index.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Error middleware trata `AppError` sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | `requireRole('ADMIN')` reutilizado no replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Padrão de uso `authenticate, requireRole('ADMIN')` replicado | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-09 | docs/FDD.md | Integração | `validate({ body, query, params })` reutilizado com schemas Zod novos | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Novas variáveis de ambiente no schema Zod de configuração | CODIGO | src/config/env.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Worker usa `createPrismaClient()` para instância própria | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração | `buildControllers` instancia repository → service → controller do módulo | CODIGO | src/app.ts |
| FDD-INT-13 | docs/FDD.md | Integração | `buildApiRouter` monta `/webhooks` e `/admin/webhooks` | CODIGO | src/routes/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Router factory segue o formato de `buildOrderRouter` | CODIGO | src/modules/orders/order.routes.ts |
| FDD-INT-15 | docs/FDD.md | Integração | `src/worker.ts` replica o bootstrap e o shutdown gracioso do server | CODIGO | src/server.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Novos modelos seguem PK UUID e `@@map` snake_case do schema | CODIGO | prisma/schema.prisma |
| FDD-INT-17 | docs/FDD.md | Integração | Envelope `data` + `pagination` reutiliza o helper `paginated` | CODIGO | src/shared/http/response.ts |
| FDD-INT-18 | docs/FDD.md | Integração | Enum `OrderStatus` é a fonte dos valores de `subscribedStatuses` | CODIGO | prisma/schema.prisma |
| FDD-INT-19 | docs/FDD.md | Integração | `canTransition` / `shouldDebitStock` permanecem inalterados na transação | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-20 | docs/FDD.md | Integração | Padrão de schema Zod com `z.coerce.number().int().min(1).max(100)` na paginação | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-21 | docs/FDD.md | Integração | Request logger existente segue como referência de log por requisição | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-22 | docs/FDD.md | Integração | `OrderWithRelations` é o modelo de tipo composto do repository | CODIGO | src/modules/orders/order.repository.ts |

## FDD — Critérios Técnicos e Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CA-01 | docs/FDD.md | Critério de Aceite | Rollback da outbox implica rollback da mudança de status | TRANSCRICAO | [09:41] Diego |
| FDD-CA-02 | docs/FDD.md | Critério de Aceite | Entrega abaixo de 10 segundos no caminho feliz | TRANSCRICAO | [09:02] Marcos |
| FDD-CA-03 | docs/FDD.md | Critério de Aceite | Backoff segue a progressão definida | TRANSCRICAO | [09:17] Diego |
| FDD-CA-04 | docs/FDD.md | Critério de Aceite | Após 5 falhas o evento sai da outbox e aparece na DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de Aceite | Replay preserva o mesmo `eventId` | TRANSCRICAO | [09:25] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de Aceite | Replay sem role ADMIN retorna 403 | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-07 | docs/FDD.md | Critério de Aceite | Assinatura HMAC verificável com a secret recebida | TRANSCRICAO | [09:20] Sofia |
| FDD-CA-08 | docs/FDD.md | Critério de Aceite | URL `http://` recusada com `WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de Aceite | Secret antiga válida por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério de Aceite | Status não subscrito não gera linha na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-11 | docs/FDD.md | Critério de Aceite | Payload acima de 64 KB gera erro e não é truncado | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-12 | docs/FDD.md | Critério de Aceite | Reinício da API não interrompe o worker | TRANSCRICAO | [09:11] Diego |
| FDD-CA-13 | docs/FDD.md | Critério de Aceite | Códigos de erro do módulo usam prefixo `WEBHOOK_` | TRANSCRICAO | [09:29] Larissa |
| FDD-CA-14 | docs/FDD.md | Critério de Aceite | `secret` não é retornada fora de criação e rotação | TRANSCRICAO | [09:31] Marcos |
| FDD-RISK-01 | docs/FDD.md | Risco | Regressão na transação de pedidos | TRANSCRICAO | [09:40] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox degradando a leitura | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Secret exposta em log | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Worker único como ponto único de falha | TRANSCRICAO | [09:11] Diego |

---

## ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL, inserido na mesma transação | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono — travaria mudanças de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams — exigiria infraestrutura adicional | TRANSCRICAO | [09:07] Diego |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Consequência | Crescimento da tabela exige índice em `status` e `created_at` | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Worker dentro da API — morreria no restart da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Trigger de banco — MySQL não tem `NOTIFY`/`LISTEN` | TRANSCRICAO | [09:09] Diego |
| ADR-002-CONS-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Consequência | Ordering só por `order_id` e apenas com worker único | TRANSCRICAO | [09:12] Diego |
| ADR-002-CONS-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Consequência | Latência mínima de 2s aceita explicitamente | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Alternativa Descartada | Retry indefinido — evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Alternativa Descartada | 3 tentativas — cliente com indisponibilidade de 2h seria descartado | TRANSCRICAO | [09:16] Diego |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial.md | Consequência | Cliente pode levar até 15h para saber que perdeu um evento | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | Secret global da plataforma — vazamento comprometeria todos | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa Descartada | mTLS/OAuth — custo de integração muito maior | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CONS-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Consequência | Rotação mantém duas secrets válidas por 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONS-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Consequência | Revisão de segurança obrigatória antes do deploy | TRANSCRICAO | [09:46] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com `X-Event-Id` para deduplicação | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa Descartada | Exactly-once — exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Consequência | Deduplicação vira responsabilidade do cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Injetar repository de webhooks no `OrderService` | TRANSCRICAO | [09:41] Diego |
| ADR-006-REF-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Referência de Código | Padrão de módulo `controller`/`service`/`repository`/`routes`/`schemas` | CODIGO | src/modules/orders/order.service.ts |
| ADR-006-REF-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Referência de Código | Taxonomia de erros derivada de `AppError` | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-REF-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Referência de Código | Middleware centralizado já traduz AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-REF-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Referência de Código | Logger Pino exportado como singleton | CODIGO | src/shared/logger/index.ts |
| ADR-006-REF-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Referência de Código | `requireRole` já existente para autorização por role | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-CONS-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Consequência | UUID como PK mantido por consistência com o resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-007 | docs/adrs/ADR-007-dlq-em-tabela-separada.md | Decisão | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-007-ALT-01 | docs/adrs/ADR-007-dlq-em-tabela-separada.md | Alternativa Descartada | Status `failed` na própria outbox — poluiria a leitura | TRANSCRICAO | [09:18] Diego |
| ADR-007-CONS-01 | docs/adrs/ADR-007-dlq-em-tabela-separada.md | Consequência | Recuperação depende de intervenção humana | TRANSCRICAO | [09:18] Diego |
| ADR-008 | docs/adrs/ADR-008-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado como snapshot no momento da inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-008-ALT-01 | docs/adrs/ADR-008-snapshot-do-payload-na-insercao.md | Alternativa Descartada | Renderizar no envio — evento refletiria estado posterior do pedido | TRANSCRICAO | [09:52] Larissa |
| ADR-008-CONS-01 | docs/adrs/ADR-008-snapshot-do-payload-na-insercao.md | Consequência | Payload congelado no formato da época | TRANSCRICAO | [09:52] Larissa |

---

## Decisões Transversais (cobrem múltiplos documentos)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| X-01 | docs/RFC.md, docs/FDD.md | Decisão | Fluxo é somente outbound — cliente recebe, não envia | TRANSCRICAO | [09:02] Sofia |
| X-02 | docs/RFC.md, docs/PRD.md | Contexto | Clientes aceitam qualquer latência abaixo de 10 segundos como "tempo real" | TRANSCRICAO | [09:02] Marcos |
| X-03 | docs/FDD.md, docs/adrs/ADR-002 | Decisão | Polling de 2 segundos atende ao requisito de 10 segundos com folga | TRANSCRICAO | [09:09] Diego |
| X-04 | docs/FDD.md, docs/adrs/ADR-006 | Decisão | Módulo em `src/modules/webhooks` seguindo o padrão da codebase | TRANSCRICAO | [09:27] Bruno |
| X-05 | docs/PRD.md, docs/FDD.md | Restrição | `customer_id` vem do body/path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| X-06 | docs/FDD.md, docs/TRACKER.md | Restrição | UUID como chave primária das novas tabelas | TRANSCRICAO | [09:51] Larissa |
| X-07 | docs/PRD.md, docs/FDD.md | Decisão | Requisito de 64 KB é não funcional, não decisão arquitetural separada | TRANSCRICAO | [09:24] Larissa |
| X-08 | docs/FDD.md, docs/adrs/ADR-007 | Decisão | Replay de DLQ exige role ADMIN e registra o autor | TRANSCRICAO | [09:36] Sofia |
| X-09 | docs/RFC.md, docs/PRD.md | Restrição | Ordenação por `order_id` é limitação conhecida, não garantia | TRANSCRICAO | [09:13] Larissa |
| X-10 | docs/PRD.md, docs/RFC.md | Restrição | Documentação de integração no portal do desenvolvedor é responsabilidade do PM | TRANSCRICAO | [09:26] Marcos |
| X-11 | docs/PRD.md, docs/RFC.md | Prazo | Estimativa de 3 sprints com revisão de segurança incluída | TRANSCRICAO | [09:46] Larissa |
| X-12 | docs/PRD.md, docs/RFC.md | Prazo | Atlas espera entrega até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| X-13 | docs/FDD.md | Restrição | PrismaClient é por processo; worker usa instância nova | TRANSCRICAO | [09:30] Bruno |
| X-14 | docs/FDD.md | Decisão | Filtro de eventos aplicado na inserção, não no envio | TRANSCRICAO | [09:34] Bruno |
| X-15 | docs/FDD.md, docs/PRD.md | Decisão | Timeout de 10s na chamada HTTP ao cliente | TRANSCRICAO | [09:42] Diego |

---

## Notas de Manutenção

**Cobertura.** As linhas acima cobrem os itens identificáveis dos documentos entregues: 16 requisitos funcionais, 12 não funcionais, 5 objetivos, 11 itens de escopo, 7 itens fora de escopo, 7 riscos, 13 critérios de aceite de produto, 18 itens de RFC, 34 itens de FDD, 8 ADRs e 15 decisões transversais.

**Itens deliberadamente fora do tracker.** Dois tipos de conteúdo **não** têm linha própria, por não serem decisões ou requisitos:

1. **Estrutura de formatação** — títulos de seção, ordem das seções, formato de tabela, exemplos de payload ilustrativos. São escolhas de redação, não de produto ou arquitetura.
2. **Estrutura exata das tabelas do banco** — a modelagem de `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` é proposta do [FDD](FDD.md#modelo-de-dados), construída sobre os campos citados na reunião ([09:21] Bruno, [09:08] Diego, [09:43] Diego). Os **campos e comportamentos** estão rastreados; o **desenho das tabelas** é decisão de implementação a validar em revisão.

**Métricas e tracing.** A seção de observabilidade do FDD distingue explicitamente o que veio da reunião (logs via Pino, [09:29] Bruno) do que é **proposta do próprio FDD** (métricas e tracing), já que esses tópicos não foram discutidos na call. A linha `FDD-OBS-04` registra essa distinção.

**Contradições verificadas.** Nenhum item registrado contradiz a transcrição ou o código. Os pontos que poderiam gerar contradição foram tratados como exclusões explícitas: notificação por e-mail, dashboard visual, rate limiting, arquivamento da outbox, webhooks inbound, ordering global e escala horizontal do worker — todos fora de escopo com origem rastreada em `PRD-OOS-01` a `PRD-OOS-07`.
