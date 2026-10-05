# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Status** | Aprovado para desenvolvimento |
| **Data** | 2026-10-05 |
| **Product Manager** | Marcos |
| **Documentos relacionados** | [RFC-001](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [TRACKER](TRACKER.md) |

---

## Resumo e contexto da feature

O OMS passa a oferecer **webhooks outbound**: quando o status de um pedido muda, a plataforma notifica automaticamente o sistema do cliente por HTTP, em vez de exigir que ele consulte a API repetidamente.

A feature nasce de um pedido formal de três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — que hoje fazem polling no `GET /orders` para descobrir mudanças de status, o que torna a integração lenta e cara para eles ([09:00] Marcos).

A decisão técnica foi tomada em reunião entre Tech Lead, PM, engenheiros e segurança. A solução é um **padrão outbox no MySQL existente**, com **worker em processo separado** consumindo por polling, **retry com backoff** e **DLQ**, autenticação **HMAC-SHA256** e entrega **at-least-once**. O detalhamento técnico está no [RFC](RFC.md) e no [FDD](FDD.md).

**Esforço estimado:** 3 sprints, incluindo a revisão de segurança ([09:46] Larissa).

---

## Problema e motivação

### O problema

Os clientes B2B precisam saber quando o status dos pedidos deles muda. Sem um mecanismo de notificação, a única forma de descobrir é **consultar a API repetidamente**. Isso produz três efeitos indesejados:

1. **Integração lenta.** A latência percebida pelo cliente é o intervalo entre suas consultas, não o tempo real de mudança de status ([09:00] Marcos).
2. **Custo para o cliente.** O polling contínuo consome recursos dos dois lados sem entregar informação nova na maior parte das chamadas ([09:00] Marcos).
3. **Risco de churn.** A Atlas Comercial sinalizou que, **se a entrega não ocorrer até o fim do trimestre, pode migrar para o concorrente** ([09:00] Marcos).

### Por que agora

O pedido é formal e vem de três clientes simultaneamente ([09:00] Marcos). O prazo de fim de novembro é uma expectativa declarada pelo cliente, com risco comercial associado ([09:45] Marcos).

### O que "tempo real" significa para o cliente

O PM perguntou especificamente qual latência seria aceitável. A resposta foi que **qualquer coisa abaixo de 10 segundos já é considerada "tempo real"**. O que importa é que o cliente não precise ficar atualizando manualmente ([09:02] Marcos). Essa definição é o que permite uma solução assíncrona com polling, em vez de exigir entrega síncrona.

---

## Público-alvo e cenários de uso

### Público-alvo

| Persona | Descrição | O que precisa |
| --- | --- | --- |
| **Integrador B2B** (Atlas, MaxDistribuição, Nova Cargo) | Time de tecnologia do cliente, responsável por consumir os eventos | Receber notificação confiável, verificar autenticidade, deduplicar e consultar histórico |
| **Operador da plataforma** (role `OPERATOR`) | Usuário autenticado que administra a configuração de webhooks dos clientes | Cadastrar, listar, editar e remover webhooks; consultar entregas ([09:37] Sofia) |
| **Administrador** (role `ADMIN`) | Responsável por operar falhas de entrega | Reprocessar eventos presos na DLQ ([09:36] Sofia) |

### Cenários de uso

**C1 — Cadastro e recebimento no caminho feliz.** O integrador da Atlas cadastra um webhook informando a URL HTTPS e a lista de status que quer acompanhar — por exemplo, apenas `SHIPPED` e `DELIVERED`. A plataforma devolve a secret gerada. Quando um pedido da Atlas muda para `SHIPPED`, o sistema da Atlas recebe a notificação em poucos segundos, valida a assinatura e processa ([09:31] Marcos, [09:33] Marcos, [09:02] Marcos).

**C2 — Filtro evita ruído.** A MaxDistribuição só quer saber de `DELIVERED`. Um pedido dela muda para `PAID`; como nenhum webhook dela subscreve esse status, **nenhuma notificação é gerada** ([09:33] Marcos, [09:34] Bruno).

**C3 — Cliente em manutenção planejada.** O sistema da Nova Cargo fica indisponível por duas horas durante uma janela de manutenção. As primeiras tentativas falham. O backoff espaça as tentativas e a entrega acontece com sucesso quando o cliente volta — sem intervenção humana ([09:16] Diego).

**C4 — Endpoint desativado permanentemente.** Um cliente descomissiona a URL cadastrada sem avisar. As 5 tentativas se esgotam ao longo de ~15 horas. O evento é preservado na DLQ. Um administrador investiga e, depois que o cliente corrige o endpoint, faz o replay do evento ([09:15] Diego, [09:18] Diego).

**C5 — Suspeita de vazamento de secret.** O cliente identifica que pode ter exposto a secret em log próprio. Ele solicita rotação pela API; a plataforma devolve uma secret nova e mantém a antiga válida por 24 horas, dando tempo de migrar os sistemas dele sem interrupção ([09:21] Sofia, [09:22] Diego).

**C6 — Auditoria de entregas.** O integrador precisa provar internamente que recebeu todas as notificações. Ele consulta o histórico de entregas e obtém sucesso/falha, payload enviado, resposta recebida e tempo de resposta de cada tentativa ([09:34] Marcos).

---

## Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| **O1** | Notificar mudanças de status em tempo real para o cliente | Latência entre a mudança de status e a entrega no cliente | **Abaixo de 10 segundos** | [09:02] Marcos |
| **O2** | Entregar a feature dentro da janela esperada pelo cliente | Data de disponibilidade em produção | **Até o fim de novembro** | [09:45] Marcos |
| **O3** | Eliminar a necessidade de polling do lado do cliente | Número de clientes B2B migrados para webhook | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) | [09:00] Marcos |
| **O4** | Absorver indisponibilidades temporárias do cliente sem perder eventos | Janela de retry antes de considerar falha permanente | **~15 horas** | [09:17] Diego |
| **O5** | Não degradar a operação de pedidos existente | Latência da transação de mudança de status | **Sem impacto perceptível** — nenhuma chamada de rede adicionada à transação | [09:04] Bruno |

**Métrica primária:** O1. É a definição de sucesso dada diretamente pelo cliente e o critério que diferencia a feature do polling atual. É também a métrica que o FDD instrumenta diretamente (`webhook_end_to_end_latency_ms`).

**Nota sobre metas não quantificadas.** Metas como taxa de sucesso de entrega ou disponibilidade do worker **não foram discutidas na reunião** e não são fixadas aqui. O produto acompanha O1–O5 nesta fase; metas adicionais devem ser definidas depois de observar tráfego real.

---

## Escopo

### Incluso

| # | Item | Origem |
| --- | --- | --- |
| E1 | Cadastro, listagem, edição e remoção de configuração de webhook por customer | [09:31]–[09:33] Marcos, Bruno |
| E2 | Filtro de status por endpoint, aplicado na geração do evento | [09:33] Marcos, [09:34] Bruno |
| E3 | Secret gerada pela plataforma, devolvida na criação | [09:31] Marcos |
| E4 | Rotação de secret pela API, com grace period de 24 horas | [09:21] Sofia |
| E5 | Notificação HTTP de mudança de status com payload JSON e headers de segurança | [09:43], [09:44] Diego |
| E6 | Assinatura HMAC-SHA256 com secret por endpoint | [09:20], [09:21] Sofia |
| E7 | Retry com backoff exponencial e DLQ | [09:15], [09:17] Diego |
| E8 | Replay manual de DLQ por administrador | [09:18] Diego, [09:36] Sofia |
| E9 | Histórico de entregas consultável pelo cliente | [09:34] Marcos |
| E10 | Garantia at-least-once com identificador de evento para deduplicação | [09:24], [09:25] Diego |
| E11 | Exigência de HTTPS na URL cadastrada | [09:23] Sofia |

### Fora de escopo

| # | Item excluído | Motivo do descarte ou adiamento | Origem |
| --- | --- | --- | --- |
| **X1** | **Notificação por e-mail ao cliente** quando o webhook falha repetidamente | Descartado nesta fase. "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." | [09:37] Larissa |
| **X2** | **Dashboard visual** para o cliente gerenciar e visualizar seus webhooks | Descartado. Esta fase entrega apenas endpoints; painel é projeto separado do time de frontend | [09:39] Larissa, [09:40] Larissa |
| **X3** | **Rate limiting de envio por cliente** (evitar rajadas de chamadas) | Adiado. "A gente observa e implementa se virar problema" — registrado como ponto em aberto | [09:38] Diego, [09:39] Larissa |
| **X4** | **Arquivamento das linhas entregues da outbox** após ~30 dias | Mencionado como necessário, mas explicitamente fora do escopo desta feature | [09:08] Diego |
| **X5** | **Webhooks inbound** (cliente enviando eventos para a plataforma) | Descartado na definição de escopo. O fluxo é somente outbound | [09:02] Sofia, [09:02] Marcos |
| **X6** | **Garantia de ordering global** entre pedidos diferentes | Nunca foi pedido pelos clientes. A garantia é por `order_id` e apenas com worker único | [09:13] Larissa, [09:14] Marcos |
| **X7** | **Escala horizontal do worker** mantendo a ordenação | Adiado como "problema do futuro, não agora" | [09:13] Diego |

> **Atenção para a implementação:** X1, X2, X3 e X4 foram **explicitamente descartados ou adiados** na reunião. Se algum deles aparecer como requisito em qualquer documento ou código desta feature, é sinal de desalinhamento com a decisão tomada.

---

## Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **FR-01** | O sistema deve permitir cadastrar uma configuração de webhook para um customer, informando URL e lista de status subscritos. A secret é gerada pela plataforma e devolvida na resposta de criação. | [09:31] Marcos |
| **FR-02** | O sistema deve permitir listar as configurações de webhook de um customer. | [09:33] Bruno |
| **FR-03** | O sistema deve permitir editar uma configuração de webhook existente (URL, lista de status subscritos e estado ativo). | [09:33] Bruno |
| **FR-04** | O sistema deve permitir remover uma configuração de webhook. | [09:33] Bruno |
| **FR-05** | O sistema deve filtrar eventos por status **no momento da geração**: se nenhum webhook do customer subscreve o novo status, nenhum evento é gerado. | [09:33] Marcos, [09:34] Bruno |
| **FR-06** | O sistema deve permitir que o cliente rotacione a secret do seu webhook pela API, mantendo a secret anterior válida por 24 horas em paralelo. | [09:21] Sofia |
| **FR-07** | O sistema deve permitir consultar o histórico de entregas de um webhook, incluindo sucesso/falha, payload, resposta do cliente e tempo de resposta. | [09:34] Marcos |
| **FR-08** | O sistema deve notificar o cliente por HTTP POST quando o status de um pedido seu mudar, enviando payload JSON com identificador do evento, tipo, timestamp, dados do pedido, status de origem e destino. | [09:43] Diego |
| **FR-09** | O sistema deve retentar automaticamente entregas que falharem, com backoff exponencial em até 5 tentativas. | [09:15], [09:17] Diego |
| **FR-10** | O sistema deve mover para uma fila de mensagens mortas os eventos que esgotarem as tentativas, preservando payload e motivo da falha. | [09:15], [09:18] Diego |
| **FR-11** | O sistema deve permitir que um administrador reprocesse manualmente um evento da DLQ, recolocando-o na fila de entrega. | [09:18] Diego, [09:36] Sofia |
| **FR-12** | O sistema deve assinar cada requisição enviada ao cliente com HMAC-SHA256 sobre o corpo, usando uma secret única por endpoint. | [09:20], [09:21] Sofia |
| **FR-13** | O sistema deve enviar um identificador único e estável do evento em cada tentativa de entrega, permitindo que o cliente deduplique notificações repetidas. | [09:25] Diego |
| **FR-14** | O sistema deve recusar o cadastro de webhook cuja URL não utilize HTTPS. | [09:23] Sofia |
| **FR-15** | O sistema deve restringir o reprocessamento de DLQ a usuários com role `ADMIN` e registrar quem executou a ação, para auditoria. | [09:36] Sofia |
| **FR-16** | O sistema deve registrar cada tentativa de entrega com o resultado, permitindo a consulta prevista em FR-07. | [09:34] Marcos |

---

## Requisitos não funcionais

| ID | Requisito | Valor / critério | Origem |
| --- | --- | --- | --- |
| **NFR-01** | **Latência de entrega** | Abaixo de 10 segundos no caminho feliz | [09:02] Marcos |
| **NFR-02** | **Intervalo de polling do worker** | 2 segundos | [09:09] Diego |
| **NFR-03** | **Timeout da chamada HTTP ao cliente** | 10 segundos; excedido, é tratado como falha e entra em retry | [09:42] Diego |
| **NFR-04** | **Tamanho máximo do payload** | 64 KB. Acima disso, a entrega falha — o payload **não é truncado** | [09:24] Diego, [09:24] Larissa |
| **NFR-05** | **Garantia de entrega** | At-least-once: duplicatas são possíveis e o cliente deve deduplicar | [09:24] Diego |
| **NFR-06** | **Atomicidade** | A geração do evento ocorre na mesma transação da mudança de status. Se a transação falhar, o evento não existe | [09:06] Diego, [09:40] Bruno |
| **NFR-07** | **Isolamento de processo** | O worker roda em processo separado da API; reinício de um não afeta o outro | [09:11] Diego |
| **NFR-08** | **Ordenação** | Garantida por `order_id` apenas, e somente enquanto houver um único worker | [09:12], [09:13] Larissa |
| **NFR-09** | **Grace period de rotação de secret** | 24 horas com a secret anterior válida em paralelo | [09:21] Sofia |
| **NFR-10** | **Sem impacto na transação de pedidos** | Nenhuma chamada de rede adicionada ao caminho crítico da mudança de status | [09:04] Bruno |
| **NFR-11** | **Reuso de padrões** | Erros derivados de `AppError`, logger Pino, middleware de erro, validação Zod e `requireRole` reaproveitados sem alteração | [09:30] Larissa |
| **NFR-12** | **Revisão de segurança** | HMAC e geração de secret revisados pela engenharia de segurança antes do deploy | [09:46] Sofia |

---

## Decisões e trade-offs principais

| Decisão | Trade-off aceito | Referência |
| --- | --- | --- |
| Outbox no MySQL existente, em vez de broker externo | Aceita-se latência de polling em troca de não subir infraestrutura nova ([09:07] Diego) | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker em processo separado com polling de 2s | Aceita-se latência mínima de 2s em toda entrega, mesmo com o cliente saudável ([09:10] Larissa) | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 tentativas com backoff até ~15h | Aceita-se que o cliente só descubra a perda do evento após ~15 horas ([09:17] Marcos) | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial.md) |
| Secret por endpoint com rotação | Aceita-se complexidade de manter duas secrets válidas durante 24h ([09:21] Sofia) | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| Entrega at-least-once | Aceita-se transferir a responsabilidade de deduplicação ao cliente ([09:25] Sofia) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões existentes | Aceita-se acoplar `changeStatus` à função de publicação de evento ([09:40] Bruno) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| DLQ com replay manual | Aceita-se que a recuperação dependa de intervenção humana ([09:18] Diego) | [ADR-007](adrs/ADR-007-dlq-em-tabela-separada.md) |
| Snapshot do payload na geração | Aceita-se payload congelado no formato da época ([09:52] Larissa) | [ADR-008](adrs/ADR-008-snapshot-do-payload-na-insercao.md) |

---

## Dependências

| Dependência | Tipo | Situação | Observação |
| --- | --- | --- | --- |
| MySQL / Prisma | Técnica | Existente | Nenhuma infraestrutura nova ([09:07] Diego) |
| Stack Express + Zod + Pino + JWT | Técnica | Existente | Reaproveitada integralmente ([09:30] Larissa) |
| `node:crypto` | Técnica | Nativo | HMAC-SHA256 sem dependência nova ([09:20] Sofia) |
| Revisão da engenharia de segurança | Processo | Bloqueante para deploy | Dois dias úteis reservados ([09:46] Sofia) |
| Documentação no portal do desenvolvedor | Produto | Sob responsabilidade do PM | Instruções de integração, incluindo deduplicação ([09:26] Marcos, [09:40] Marcos) |
| Confirmação de prazo com os clientes | Negócio | Sob responsabilidade do PM | PM confirma prazo com a Atlas ([09:47] Marcos) |
| Implementação do lado do cliente | Externa | Fora do controle da plataforma | Cliente precisa aceitar HTTPS, validar HMAC e deduplicar por `X-Event-Id` |

---

## Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **R1** | **Perda do cliente Atlas** por atraso na entrega. A Atlas sinalizou migração para o concorrente caso a feature não chegue até o fim do trimestre | Média | Alto | Estimativa de 3 sprints ([09:46] Larissa); prazo confirmado com o cliente pelo PM ([09:47] Marcos) |
| **R2** | **Regressão na mudança de status de pedidos** em produção, já que `changeStatus` é caminho crítico e passa a ter um passo adicional | Média | Alto | Inserção como último passo da transação, sem I/O de rede ([09:04] Bruno); teste de rollback dedicado ([09:40] Bruno) |
| **R3** | **Cliente não implementa deduplicação** e processa o mesmo evento mais de uma vez, gerando efeitos duplicados no sistema dele | Média | Médio | Documentação destacada no portal do desenvolvedor ([09:26] Marcos); `X-Event-Id` em header e no corpo ([09:25] Diego) |
| **R4** | **Vazamento de secret** do lado do cliente. Já ocorreu com outro cliente, que expôs a secret em log de aplicação | Média | Alto | Secret por endpoint limita o raio de impacto ([09:21] Sofia); rotação com grace period de 24h ([09:21] Sofia); revisão de segurança obrigatória ([09:46] Sofia) |
| **R5** | **Eventos presos na DLQ sem ninguém perceber**, já que não há reprocessamento automático nem notificação por e-mail | Alta | Médio | Métrica de volume da DLQ com alerta; replay manual documentado ([09:18] Diego). Notificação por e-mail adiada (X1) |
| **R6** | **Crescimento da outbox** degradando o desempenho da leitura do worker, já que o arquivamento ficou fora de escopo | Média | Médio | Índices em `status` e `created_at`, leitura em batch pequeno ([09:08] Diego). Risco residual aceito conscientemente |
| **R7** | **Worker único como ponto único de falha**: se o processo cair, nenhum evento é entregue | Média | Médio | Processo separado com restart automático ([09:11] Diego). Escala horizontal adiada por quebrar a ordenação (X7) |

---

## Critérios de aceitação

Critérios verificáveis do ponto de vista de produto. Os critérios técnicos detalhados estão no [FDD](FDD.md#critérios-de-aceite-técnicos).

| # | Critério | Como validar |
| --- | --- | --- |
| **CA-P1** | Um cliente consegue cadastrar um webhook informando URL HTTPS e lista de status, e recebe a secret gerada na resposta | Fluxo completo de cadastro contra a API |
| **CA-P2** | Uma mudança de status é entregue ao cliente em **menos de 10 segundos** | Teste ponta a ponta com cronometragem |
| **CA-P3** | Um status não subscrito por nenhum webhook do customer **não gera notificação** | Mudança de status e verificação de ausência de entrega |
| **CA-P4** | Uma indisponibilidade temporária do cliente **não perde o evento**: a entrega ocorre quando ele volta | Simulação de cliente fora do ar por período curto |
| **CA-P5** | Um evento que falha 5 vezes é preservado na DLQ com payload e motivo da falha | Esgotar tentativas e inspecionar a DLQ |
| **CA-P6** | Um administrador consegue reprocessar um evento da DLQ e ele é entregue | Replay e verificação de recebimento |
| **CA-P7** | Um usuário sem role `ADMIN` **não consegue** reprocessar DLQ | Requisição com token de operador |
| **CA-P8** | O cliente consegue validar a assinatura da requisição com a secret recebida | Verificação independente do HMAC |
| **CA-P9** | O cliente consegue rotacionar a secret sem interrupção de entrega | Rotação e verificação de que ambas as secrets funcionam por 24h |
| **CA-P10** | O cliente consegue consultar o histórico das últimas entregas com resultado, payload, resposta e tempo | Consulta ao endpoint de entregas |
| **CA-P11** | Cadastro de webhook com URL `http://` é recusado | Tentativa de cadastro sem TLS |
| **CA-P12** | O cliente recebe a mesma notificação duas vezes apenas em cenário de falha, com o mesmo `X-Event-Id` | Inspeção do identificador em reenvios |
| **CA-P13** | A operação normal de pedidos **não é impactada**: mudanças de status continuam funcionando com a mesma fluidez | Comparação de latência antes e depois |

---

## Estratégia de testes e validação

O projeto já usa **Vitest** com **supertest** (`vitest.config.ts`, `tests/`), com `fileParallelism: false` e pool de fork único por rodar contra MySQL real. A estratégia abaixo estende esse setup.

### Camadas de teste

| Camada | Escopo | Exemplos |
| --- | --- | --- |
| **Unitário** | Cálculo de backoff, geração e validação de HMAC, validação de URL HTTPS, filtro de status | Progressão 1m/5m/30m/2h/12h; assinatura verificável; `http://` recusado |
| **Integração (service + banco)** | Transação de `changeStatus` com inserção na outbox; atomicidade; movimentação para DLQ | Rollback forçado na inserção do evento; evento que esgota tentativas e sai da outbox |
| **Ponta a ponta** | `PATCH /orders/:id/status` até o recebimento no servidor de teste do cliente | Latência abaixo de 10s; payload e headers corretos; assinatura validável |
| **Segurança** | Rotação de secret, grace period, autorização do replay | Secret antiga válida por 24h; replay com `OPERATOR` retorna 403 |

### Pontos de validação obrigatórios

1. **Atomicidade (CA-P... / NFR-06).** O teste mais importante: forçar falha na inserção da outbox e confirmar que o status do pedido **não** mudou ([09:40] Bruno).
2. **Latência ponta a ponta (CA-P2).** Servidor HTTP de teste que registra o instante de recebimento, comparado ao instante da chamada de mudança de status.
3. **Ciclo completo de retry (CA-P5).** Cliente que responde erro e relógio controlado, para não esperar 15 horas reais.
4. **Assinatura (CA-P8).** Verificação independente do HMAC sobre o corpo, com a secret devolvida no cadastro.
5. **Autorização (CA-P7).** Replay com token de `OPERATOR` deve retornar 403.

### Validação de aceite

- **Revisão de segurança obrigatória** antes do deploy: pelo menos dois dias úteis reservados para a engenharia de segurança revisar HMAC e geração de secret ([09:46] Sofia, [09:46] Larissa).
- **Validação com cliente real:** entrega validada com a Atlas, que é o cliente com risco de churn declarado ([09:00] Marcos).
- **Sem requisito de validação por e-mail ou dashboard:** ambos fora de escopo (X1, X2), portanto não há critério de aceite associado.

---

## Rastreabilidade

Cada requisito, decisão e exclusão deste PRD tem origem identificável na transcrição da reunião técnica ou no código da aplicação. O mapeamento completo está em [TRACKER.md](TRACKER.md).
