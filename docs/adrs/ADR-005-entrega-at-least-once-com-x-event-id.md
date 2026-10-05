# ADR-005: Entrega at-least-once com deduplicação por X-Event-Id

## Status

**Aceito** — 2026-10-05

- **Decisores:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead), Sofia (Engenheira de Segurança), Marcos (Product Manager)
- **Origem na transcrição:** [09:24]–[09:26]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#contratos-públicos), [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial.md)

## Contexto

O worker marca um evento como entregue após receber resposta de sucesso do cliente. Existe uma janela de incerteza inerente: o cliente pode ter processado a requisição e a resposta ter se perdido antes de chegar ao worker, ou o worker pode falhar entre o envio e a marcação. Nesses casos o evento será reenviado.

Isso levanta a pergunta de qual garantia de entrega o sistema oferece — e, por consequência, qual responsabilidade é transferida ao cliente.

O requisito de negócio não exige processamento único. Os clientes querem saber que o pedido mudou, não que receberam exatamente uma notificação por mudança ([09:14] Marcos).

## Decisão

Garantia de **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso ([09:24] Diego).

Para permitir a deduplicação do lado do cliente, todo envio carrega o header **`X-Event-Id`** com um **UUID gerado no momento em que o evento entra na outbox** ([09:25] Diego). O identificador é único por evento; se o cliente receber o mesmo `event_id` duas vezes, deduplica do seu lado ([09:25] Diego).

O mesmo UUID é usado como `event_id` dentro do payload ([09:43] Diego), de modo que o cliente possa deduplicar tanto pelo header quanto pelo corpo.

A responsabilidade pela deduplicação é explicitamente do cliente ([09:25] Sofia), e isso será documentado de forma destacada no portal de desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

### 1. Exactly-once

- **Trade-off que motivou o descarte:** exigiria coordenação dos dois lados — a plataforma e cada cliente — e fica muito mais complexo ([09:25] Diego). Garantir exactly-once de ponta a ponta sobre HTTP não é atingível sem protocolo de confirmação em duas fases com o receptor. At-least-once com `event_id` resolve a grande maioria dos casos ([09:25] Diego).

### 2. At-most-once (não retentar)

- **Trade-off que motivou o descarte:** elimina duplicatas ao custo de perder eventos silenciosamente em qualquer falha transitória de rede. É incompatível com o objetivo da feature, que existe justamente porque os clientes não querem descobrir mudanças de status por conta própria. Contradiz também a política de retry de [ADR-003](ADR-003-retry-com-backoff-exponencial.md).

### 3. Precedente de mercado como validação

Não é uma alternativa descartada, mas foi o argumento de sustentação: Stripe e GitHub adotam at-least-once com identificador de evento ([09:25] Diego). Isso indica que o modelo é aceito por clientes B2B sem atrito relevante de integração.

## Consequências

### Positivas

- **Simplicidade de implementação.** Não requer protocolo de confirmação em duas fases nem coordenação com o cliente ([09:25] Diego).
- **Nenhum evento perdido em falha transitória.** Combinado com o retry de [ADR-003](ADR-003-retry-com-backoff-exponencial.md), o evento só se perde após esgotar as 5 tentativas, quando vai para a DLQ.
- **Alinhamento com o mercado.** O modelo é familiar para clientes B2B, reduzindo atrito de integração ([09:25] Diego).
- **Identificador com origem única e estável.** O `event_id` nasce no outbox e é preservado em todos os reenvios, o que torna a deduplicação do cliente confiável.

### Negativas

- **Transfere responsabilidade ao cliente.** A plataforma garante entrega, mas a corretude do processamento passa a depender de o cliente deduplicar. Reconhecido explicitamente como ponto de atrito na reunião ([09:25] Sofia).
- **Duplicatas são esperadas em produção.** Qualquer cliente que não implemente deduplicação terá efeitos duplicados em seus sistemas. Mitigado por documentação destacada no portal ([09:26] Marcos), mas não eliminado.
- **Requer que o `event_id` seja imutável e persistido.** O UUID precisa ser gravado na outbox no momento da inserção e reutilizado em cada tentativa, nunca regerado. É uma restrição de implementação que precisa estar clara no FDD.
- **Não cobre deduplicação do lado da plataforma.** O sistema não tem visão de se o cliente processou o evento, então não há como oferecer uma janela de deduplicação server-side.
