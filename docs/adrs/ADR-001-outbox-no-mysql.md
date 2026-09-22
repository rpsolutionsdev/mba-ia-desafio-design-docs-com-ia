# ADR-001: Padrão Outbox no MySQL para Notificação de Eventos de Pedido

* **Status:** Aceito
* **Data:** 2026-09-22
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro de Plataforma), Bruno (Engenheiro de Pedidos), Sofia (Engenheira de Segurança), Marcos (Product Manager)
* **Origem na Reunião:** `[09:03] Larissa`, `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa`, `[09:08] Diego`
* **Origem no Código:** `src/modules/orders/order.service.ts` (método `changeStatus`)

## Contexto

O Order Management System (OMS) precisa notificar sistemas externos de clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido é alterado. A alteração de status é uma operação transacional crítica gerenciada por [OrderService.changeStatus](file:///c:/Users/USER/workspace-antiigravity/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts) usando Prisma Client (`$transaction`), responsável por atualizar o pedido na tabela `orders`, registrar histórico em `order_status_history` e decrementar estoque em `products`.

Se a chamada HTTP de notificação fosse realizada de forma síncrona dentro dessa transação ou logo após ela, qualquer lentidão, instabilidade ou indisponibilidade no endpoint do cliente reteria a transação no banco de dados MySQL ou causaria inconsistência de estado (ex: pedido com status atualizado, mas evento perdido; ou rollback de negócio indevido decorrente de falha de rede externa).

## Decisão

Adotar o **Transactional Outbox Pattern** utilizando o próprio banco de dados MySQL existente:
1. Uma nova tabela relacional `webhook_outbox` armazenará os eventos gerados.
2. A inserção do evento na tabela `webhook_outbox` ocorrerá **dentro da mesma transação SQL** que atualiza o status do pedido em `src/modules/orders/order.service.ts`.
3. O payload do evento será gravado já renderizado (snapshot imutável no momento da transição de status).
4. Um worker assíncrono independente consumirá periodicamente os registros pendentes para realizar os disparos HTTP outbound.

## Alternativas Consideradas

### 1. Disparo HTTP Síncrono no OrderService
* **Descrição:** Executar a chamada HTTP diretamente durante o fluxo de `changeStatus`.
* **Motivo do Descarte:** Clientes lentos aumentariam o tempo de retenção de locks no MySQL, degradando o throughput do sistema. Falhas no cliente externo poderiam induzir rollbacks indevidos de transações de negócio ou gerar perda silenciosa de notificações. (`[09:04] Bruno`, `[09:06] Diego`).

### 2. Mensageria Externa (Redis Streams / RabbitMQ / Kafka)
* **Descrição:** Publicar mensagens em um cluster de mensageria externo dedicado.
* **Motivo do Descarte:** A complexidade operacional de manter e monitorar uma nova infraestrutura distribuída (Redis Cluster ou RabbitMQ) representa overengineering para o tamanho atual do time e volume de clientes B2B da empresa. Além disso, a publicação em mensageria externa fora da transação de banco introduz o clássico problema de dual-write. (`[09:07] Larissa`, `[09:07] Diego`).

## Consequências

### Positivas
* **Atomicidade garantida:** O evento só existe se a transação do pedido for efetivamente confirmada (commit). Se houver rollback na alteração do pedido, o evento é revertido junto.
* **Isolamento de falhas:** Falhas, timeouts ou quedas nos serviços dos clientes externos não impactam o ciclo de vida nem a performance das transações de pedidos do OMS.
* **Simplicidade operacional:** Reutiliza a instância e infraestrutura existente de MySQL e o Prisma Client, sem requerer novos serviços de infraestrutura.

### Negativas e Mitigações
* **Acúmulo de dados na tabela outbox:** A tabela `webhook_outbox` pode acumular milhões de linhas com o tempo.
  * *Mitigação:* Criação de índices estratégicos em `(status, created_at)` e planejamento de política de retenção/arquivamento para registros entregues após 30 dias (`[09:08] Diego`).
* **Latência não estritamente instantânea:** A entrega depende do ciclo de leitura do worker.
  * *Mitigação:* Polling ajustado para 2 segundos, atendendo com folga o SLA acordado com o produto de entrega em menos de 10 segundos (`[09:02] Marcos`, `[09:10] Diego`).
