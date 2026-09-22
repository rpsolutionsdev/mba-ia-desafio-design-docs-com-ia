# PRD-001: Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) gerencia o fluxo operacional e o ciclo de vida dos pedidos da empresa. Atualmente, os clientes B2B que realizam compras de grande escala e dependem de automação logística precisam realizar consultas recorrentes (*polling*) na API REST (`GET /orders`) para saber se o status de seus pedidos foi alterado.

Esta feature introduz um subsistema de **Webhooks de Notificação de Pedidos** (*outbound webhooks*). A partir desta entrega, o OMS passa a emitir notificações HTTP instantâneas para os servidores dos clientes sempre que houver uma transição de status em seus pedidos, eliminando a dependência de polling, reduzindo latência nas operações dos parceiros e blindando os clientes contra o risco de churn.

---

## 2. Problema e Motivação

Três clientes B2B de grande faturamento da plataforma — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — emitiram solicitações formais para receber notificações automáticas em tempo real sobre seus pedidos (`[09:00] Marcos`).

O modelo atual de integração baseado em polling causa graves dores operacionais e comerciais:
1. **Ineficiência Técnica:** Centenas de requisições por minuto batendo no banco MySQL apenas para constatar que nenhum pedido mudou.
2. **Latência de Negócio:** Intervalos de até 15 minutos entre a alteração do pedido no OMS e a percepção do cliente, atrasando faturamentos e despachos.
3. **Risco Iminente de Perda de Receita (Churn):** A Atlas Comercial comunicou que a falta de webhooks em tempo real inviabiliza suas operações integradas e que migrará para o concorrente direto se a funcionalidade não estiver em produção até o fim do trimestre vigente (`[09:00] Marcos`).

---

## 3. Público-Alvo e Cenários de Uso

### 3.1. Público-Alvo
* **Desenvolvedores e Engenheiros de Integração B2B:** Profissionais responsáveis pelas integrações de ERP, WMS e TMS dos clientes corporativos (Atlas, MaxDistribuição, Nova Cargo).
* **Operadores e Suporte Técnico Interno:** Equipe de operações do OMS que monitora a saúde das entregas e realiza suporte técnico.

### 3.2. Cenários de Uso
* **Cenário 1 — Notificação Automática de Expedição:** Um pedido muda de `PROCESSING` para `SHIPPED`. O servidor WMS da Atlas Comercial recebe um HTTP POST em menos de 5 segundos, acionando a frota de transporte automaticamente sem intervenção humana.
* **Cenário 2 — Configuração Seletiva de Eventos:** A MaxDistribuição cadastra um endpoint informando interesse apenas nos status `SHIPPED` e `DELIVERED`, ignorando transições intermediárias para poupar recursos computacionais de sua infraestrutura.
* **Cenário 3 — Rotação Preventiva de Chave de Segurança:** Um engenheiro de segurança do cliente gera uma nova secret pela API e dispõe de uma janela de 24 horas para alterar a chave em suas variáveis de ambiente sem perda de mensagens.
* **Cenário 4 — Resiliência Durante Manutenção do Cliente:** O servidor da Nova Cargo fica temporariamente fora do ar por 2 horas para manutenção programada. O OMS realiza retentativas automáticas espaçadas e entrega os eventos assim que o cliente restabelece o serviço.

---

## 4. Objetivos e Métricas de Sucesso

### 4.1. Objetivos de Negócio e Produto
* Reter 100% dos clientes B2B críticos sob risco de migração (Atlas Comercial, MaxDistribuição e Nova Cargo) até o final do trimestre vigente.
* Habilitar arquitetura orientada a eventos para integrações B2B escaláveis.

### 4.2. Metas Quantitativas e Métricas de Sucesso
* **Latência de Notificação:** 95% das notificações entregues em menos de 5 segundos e 99% em menos de 10 segundos a partir do commit da transação do pedido (`[09:02] Marcos`, `[09:09] Diego`).
* **Redução de Polling Ineficiente:** Redução de pelo menos **80%** no volume de chamadas HTTP `GET /orders` originadas pelos clientes B2B nos primeiros 30 dias após a adoção dos webhooks.
* **Taxa de Sucesso na Entrega:** Alcançar um SLA de entrega com sucesso de pelo menos **99.5%** dos eventos válidos emitidos.
* **Tolerância a Indisponibilidade Externa:** Capacidade de reter e retentar eventos por uma janela de até **15 horas** consecutivas em casos de instabilidade no endpoint do parceiro.

---

## 5. Escopo da Solução

### 5.1. Incluso no Escopo (In-Scope)
* Cadastro, listagem, atualização e exclusão de configurações de webhooks por cliente B2B via API REST autenticada (`POST`, `GET`, `PATCH`, `DELETE`).
* Filtragem granular de eventos por lista de status de pedido no momento da inserção (`OrderStatus`).
* Assinatura criptográfica obrigatória HMAC-SHA256 (`X-Signature`) gerada a partir de secret exclusiva por endpoint.
* Endpoint para rotação de secret com tolerância de convivência (grace period) de 24 horas para a chave anterior.
* Garantia de entrega *at-least-once* com identificador único UUID de deduplicação (`X-Event-Id`).
* Consulta paginada do histórico de entregas de cada webhook (`GET /webhooks/:id/deliveries`).
* Política automática de retry com 5 tentativas em backoff exponencial (1m, 5m, 30m, 2h, 12h).
* Armazenamento de falhas esgotadas em Dead Letter Queue (DLQ) persistida em tabela dedicada.
* Endpoint administrativo para reprocessamento manual (replay) de eventos da DLQ restrito a usuários com perfil `ADMIN`.

### 5.2. Fora de Escopo (Out-of-Scope)
* **Notificações por E-mail em Falha:** Disparo de e-mails de alerta para clientes após falhas consecutivas de webhook (descartado na reunião para manter o foco na entrega core do trimestre; `[09:37] Larissa`).
* **Interface Gráfica / Dashboard Frontend:** Criação de telas e painéis visuais no portal web do cliente (descartado pelo tech lead; o escopo é estritamente via API REST; `[09:40] Larissa`).
* **Webhooks Inbound:** Recepção de requisições externas enviadas pelos clientes para o OMS (a reunião definiu escopo puramente outbound; `[09:02] Sofia`, `[09:03] Sofia`).
* **Brokers de Mensageria Dedicados (Redis/Kafka):** Instalação e manutenção de novas ferramentas de mensageria distribuída (descartado por overengineering para o time atual; `[09:07] Diego`).
* **Rate Limiting de Saída Outbound:** Mecanismos de controle de vazão de disparos por cliente (postergado para análise e monitoramento em produção; `[09:39] Diego`, `[09:39] Larissa`).

---

## 6. Requisitos Funcionais

A tabela abaixo sintetiza os requisitos funcionais discutidos e acordados na reunião técnica:

| ID | Nome do Requisito | Descrição Detalhada | Origem |
| :--- | :--- | :--- | :--- |
| **PRD-FR-01** | Cadastro de Webhook | A API deve permitir que o cliente cadastre uma URL de destino para receber notificações, vinculada ao seu `customerId`. | `[09:31] Marcos` |
| **PRD-FR-02** | Geração e Devolução da Secret | Na criação do webhook, o sistema deve gerar automaticamente uma secret criptográfica segura e retorná-la apenas uma única vez no payload de resposta. | `[09:31] Marcos`, `[09:21] Sofia` |
| **PRD-FR-03** | Filtragem de Eventos por Status | O cliente pode especificar uma lista de status de pedidos (`OrderStatus`) aos quais deseja reagir. Se nenhum webhook do cliente ouvir o status alterado, o evento não deve ser gravado. | `[09:33] Marcos`, `[09:34] Bruno` |
| **PRD-FR-04** | Gerenciamento de Webhooks (CRUD) | A API deve permitir consulta (`GET`), atualização de URL/eventos (`PATCH`) e exclusão lógica/física (`DELETE`) de endpoints cadastrados. | `[09:33] Bruno` |
| **PRD-FR-05** | Consulta ao Histórico de Entregas | O cliente pode consultar o histórico paginado de tentativas de entrega de um webhook, contendo payload, status code de resposta, sucesso/falha e tempo de resposta. | `[09:34] Marcos` |
| **PRD-FR-06** | Rotação de Secret com Grace Period | A API deve permitir a rotação da secret de um endpoint, mantendo a secret anterior funcional para validação por exatamente 24 horas antes de sua revogação total. | `[09:21] Sofia`, `[09:22] Sofia` |
| **PRD-FR-07** | Captura Atômica de Evento (Outbox) | Ao alterar o status do pedido em `changeStatus`, o sistema deve gravar atomicamente o evento na tabela `webhook_outbox` dentro da mesma transação SQL. | `[09:06] Diego`, `[09:40] Bruno` |
| **PRD-FR-08** | Assinatura HMAC-SHA256 no Envio | Cada requisição outbound emitida pelo worker deve conter o cabeçalho `X-Signature` com o HMAC-SHA256 do corpo JSON assinado com a secret ativa do endpoint. | `[09:20] Sofia` |
| **PRD-FR-09** | Identificador de Deduplicação | Cada notificação deve carregar o header `X-Event-Id` com um UUID fixo único gerado na outbox, permitindo ao receptor dedupicar requisições repetidas. | `[09:25] Diego` |
| **PRD-FR-10** | Política Automática de Retry | Notificações que retornem erro HTTP (não 2xx) ou timeout devem ser retentadas automaticamente 5 vezes nos intervalos de 1m, 5m, 30m, 2h e 12h. | `[09:15] Diego`, `[09:17] Diego` |
| **PRD-FR-11** | Roteamento para Dead Letter Queue | Notificações que esgotarem as 5 tentativas de retentativa devem ser movidas para a tabela `webhook_dead_letter` contendo diagnóstico da falha. | `[09:18] Diego` |
| **PRD-FR-12** | Replay Manual de DLQ por Admin | O sistema deve disponibilizar endpoint para reprocessar manualmente eventos mortos da DLQ, exigindo estritamente a role `ADMIN` e gerando log de auditoria. | `[09:18] Diego`, `[09:36] Sofia` |

---

## 7. Requisitos Não Funcionais

* **RNF-01 (Segurança no Transporte):** É obrigatório o uso de HTTPS/TLS. Qualquer tentativa de cadastro de URL com protocolo não seguro (`http://`) deve ser rejeitada com código 400 (`[09:23] Sofia`).
* **RNF-02 (Tempo Limite de Conexão):** O worker deve abortar qualquer chamada que demore mais de 10 segundos, computando-a como tentativa com falha (`[09:42] Diego`).
* **RNF-03 (Tamanho Máximo de Carga):** O payload JSON do webhook não deve ultrapassar 64 KB. Cargas superiores devem ser rejeitadas com erro (`[09:24] Diego`, `[09:24] Sofia`).
* **RNF-04 (Garantia de Entrega):** O sistema assume semântica *at-least-once*. Clientes devem implementar idempotência a partir do cabeçalho `X-Event-Id` (`[09:24] Diego`, `[09:26] Marcos`).
* **RNF-05 (Desacoplamento e Performance):** O processo consumidor de envio deve rodar de maneira isolada em relação ao processo web da API, com intervalo de ciclo de polling fixado em 2 segundos (`[09:10] Larissa`, `[09:11] Diego`).

---

## 8. Decisões e Trade-offs Principais

* **Outbox no MySQL vs. Broker Externo:** Escolheu-se usar o banco MySQL existente em vez de adicionar Redis/Kafka para economizar custo e complexidade operacional em time enxuto (`[09:07] Diego`).
* **5 Tentativas em 15h vs. 3 Tentativas Rápidas:** Escolheu-se uma janela de 15h para cobrir manutenções programadas dos clientes B2B que chegam a 2h de duração (`[09:16] Diego`).
* **Snapshot Imutável vs. Consulta Dinâmica:** O payload é serializado no momento exato da transação do pedido para evitar discrepâncias caso o pedido sofra alterações rápidas subsequentes (`[09:51] Bruno`, `[09:52] Larissa`).

---

## 9. Dependências

* **Dependências Técnicas:**
  * Banco de dados MySQL 8.0 funcional e pool de conexões Prisma ORM configurado.
  * Runtime Node.js com suporte a ESM/TypeScript e biblioteca nativa `node:crypto`.
* **Dependências Organizacionais / Inter-times:**
  * Janela de 2 dias úteis reservada com a Engenharia de Segurança (Sofia) antes da entrada em produção para homologação formal dos fluxos de assinatura e secret (`[09:46] Sofia`).
  * Atualização da documentação do portal do desenvolvedor pelo time de produto (Marcos) com guias de validação do HMAC e deduplicação (`[09:26] Marcos`).

---

## 10. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Estratégia de Mitigação |
| :--- | :--- | :--- | :--- |
| **Degradação de performance da API por lentidão de clientes** | Média | Alto | Adoção do Transactional Outbox Pattern: nenhuma chamada de rede externa ocorre na thread da API de pedidos (`[09:04] Bruno`). |
| **Vazamento de chaves secretas compartilhadas** | Baixa | Crítico | Secrets únicas por endpoint (proibição de chave global), transporte exclusivamente por HTTPS e mecanismo de rotação com grace period de 24h (`[09:21] Sofia`). |
| **Perda de eventos por manutenção programada no cliente** | Média | Alto | Política com 5 retries espaçados até 12 horas (janela total de 15 horas) e persistência definitiva em DLQ com replay manual (`[09:16] Diego`). |
| **Duplicação de processamento no destino por oscilação de rede** | Alta | Médio | Header fixo `X-Event-Id` enviado em todas as tentativas do mesmo evento e documentação técnica orientando deduplicação pelo cliente (`[09:25] Diego`). |

---

## 11. Critérios de Aceitação de Negócio

* [ ] Os 3 clientes piloto (Atlas Comercial, MaxDistribuição e Nova Cargo) conseguem cadastrar seus endpoints HTTPS e receber eventos em ambiente de homologação.
* [ ] 100% dos eventos gerados por transições válidas de status de pedidos são disparados com cabeçalho `X-Signature` verificável via HMAC-SHA256.
* [ ] A latência medida entre o salvamento do status do pedido e o recebimento pelo cliente permanece inferior a 10 segundos para 99% das notificações.
* [ ] Endpoints indisponíveis são retentados nos intervalos definidos sem travamento do worker.
* [ ] Administradores conseguem reenfileirar eventos da DLQ com sucesso através do endpoint de replay autenticado.

---

## 12. Estratégia de Testes e Validação

* **Testes Unitários:** Validação de geração e cálculo do HMAC-SHA256, cálculo das datas de retentativa do backoff exponencial e validações de URL com Zod (rejeitando `http://`).
* **Testes de Integração:** Testar a atomicidade da transação em `OrderService.changeStatus` com Prisma Mock/In-Memory DB, garantindo rollback da outbox se a transação do pedido falhar.
* **Testes End-to-End (E2E):** Simulação de um servidor HTTP mock (ex: via WireMock ou Express receptor de testes) validando o ciclo completo: alteração de pedido -> gravação na outbox -> leitura pelo worker -> disparo com headers corretos -> validação de deduplicação com `X-Event-Id`.
* **Testes de Resiliência e Simulação de Falhas:** Simulação de endpoints retornando status 500, timeouts superiores a 10s e quedas temporárias de rede, atestando a progressão correta para a tabela `webhook_dead_letter`.
