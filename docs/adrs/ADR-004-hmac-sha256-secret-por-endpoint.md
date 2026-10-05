# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

**Aceito** — 2026-10-05

- **Decisores:** Sofia (Engenheira de Segurança), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
- **Origem na transcrição:** [09:19]–[09:22]
- **Documentos relacionados:** [RFC](../RFC.md#proposta-técnica), [FDD](../FDD.md#contratos-públicos), [ADR-001](ADR-001-outbox-no-mysql.md)

## Contexto

A feature expõe eventos contendo dados de pedidos para endpoints **fora da infraestrutura da empresa** ([09:19] Sofia). O cliente precisa conseguir verificar duas coisas: que a requisição veio realmente da plataforma e que o payload não foi adulterado no caminho ([09:19] Sofia).

Há um histórico concreto de incidente que pesa na decisão: a empresa já teve cliente que vazou secret em log de aplicação própria ([09:22] Diego). Isso significa que o modelo de credencial precisa assumir que uma secret **vai** vazar eventualmente, e que o estrago precisa ser contido.

## Decisão

**HMAC-SHA256 sobre o corpo do request**, com **secret única por endpoint de webhook** e **suporte a rotação com grace period de 24 horas** ([09:20] Sofia, [09:21] Sofia, [09:22] Sofia).

Detalhamento:

- A assinatura é enviada no header `X-Signature` ([09:20] Sofia).
- A secret **não é global da plataforma**: cada endpoint de webhook do cliente tem a sua ([09:21] Sofia). Se uma vaza, as demais permanecem válidas.
- A tabela de configuração de webhook armazena `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno).
- A secret é **rotacionável pelo cliente via API**. Ao rotacionar, a secret antiga permanece válida **em paralelo por 24 horas**, dando tempo ao cliente de migrar seus sistemas; depois disso, a antiga é invalidada ([09:21] Sofia).
- A secret é **gerada pela plataforma** e devolvida na criação do webhook, não escolhida pelo cliente ([09:31] Marcos).

Complemento relacionado, registrado aqui por ser da mesma frente de segurança: a URL do webhook precisa ser **HTTPS**. Cadastro com `http` é recusado com erro de validação, implementado como validação de schema Zod — explicitamente classificado como validação, não como decisão arquitetural ([09:23] Sofia).

## Alternativas Consideradas

### 1. Secret global única da plataforma

Todos os endpoints de clientes compartilhando a mesma credencial.

- **Trade-off que motivou o descarte:** o vazamento de uma única secret comprometeria **todos** os clientes simultaneamente ([09:21] Sofia). Dado o histórico de vazamento em log de cliente ([09:22] Diego), o raio de impacto de uma credencial global foi considerado inaceitável.

### 2. Não assinar o payload (confiar apenas em TLS)

- **Trade-off que motivou o descarte:** TLS protege o transporte, mas não permite ao cliente verificar a **origem** da requisição nem detectar adulteração por qualquer parte que termine o TLS. O requisito explícito é que o cliente consiga validar que a requisição veio da plataforma e que o payload não foi alterado ([09:19] Sofia).

### 3. mTLS ou OAuth client credentials por cliente

- **Trade-off que motivou o descarte:** exige gestão de certificados ou fluxo de tokens do lado do cliente, com custo de integração muito maior. HMAC-SHA256 foi escolhido por ser "o padrão de mercado, todo cliente sério tem biblioteca pra isso" ([09:20] Sofia) — ou seja, minimiza o esforço de adoção dos três clientes B2B que motivam a feature.

## Consequências

### Positivas

- **Raio de impacto contido.** O vazamento de uma secret compromete um único endpoint de um único cliente ([09:21] Sofia).
- **Rotação sem downtime para o cliente.** O grace period de 24 horas permite migrar sistemas sem interrupção de entrega ([09:21] Sofia).
- **Verificação trivial para o cliente.** HMAC-SHA256 tem biblioteca consolidada em qualquer stack relevante ([09:20] Sofia).
- **Integridade além da origem.** A assinatura cobre o corpo, então alteração de payload é detectável.

### Negativas

- **A secret é um material sensível armazenado no banco.** Precisa ser protegida na leitura, não exposta em logs e não devolvida em consultas subsequentes além do momento da criação e da rotação. O logger Pino já possui redação configurada para campos sensíveis como `*.token` e `*.password` ([09:29] Bruno), mas o campo da secret exige cuidado adicional na modelagem e nos contratos.
- **Duas secrets válidas durante a janela de rotação.** O worker precisa assinar considerando o estado de rotação, e a validação do cliente precisa aceitar ambas durante 24 horas. Adiciona estado e complexidade ao fluxo de envio.
- **Responsabilidade de guarda é do cliente.** A plataforma não controla se o cliente armazena a secret corretamente — o incidente passado mostra que isso é um risco real ([09:22] Diego).
- **Revisão de segurança obrigatória.** A Tech Lead reservou explicitamente pelo menos dois dias úteis para revisão do código de segurança (HMAC e geração de secret) antes do deploy ([09:46] Sofia, [09:46] Larissa).
