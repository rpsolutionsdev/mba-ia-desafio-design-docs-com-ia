# Matriz de Rastreabilidade (Tracker)

A tabela abaixo estabelece a referência cruzada completa de cada decisão, requisito, restrição e contrato especificado no pacote de design docs (`PRD.md`, `RFC.md`, `FDD.md` e ADRs), mapeando sua exata origem na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte do sistema existente.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PRD-MOTIV-01** | `docs/PRD.md` | Motivação | Demanda de clientes B2B (Atlas, MaxDistribuição, Nova Cargo) e risco de churn | `TRANSCRICAO` | `[09:00] Marcos` |
| **PRD-METRIC-01** | `docs/PRD.md` | Métrica | Definição de tempo real aceitável como inferior a 10 segundos | `TRANSCRICAO` | `[09:02] Marcos` |
| **PRD-ESCOPO-01** | `docs/PRD.md` | Escopo | Notificações outbound apenas (do OMS para os clientes) | `TRANSCRICAO` | `[09:02] Marcos` |
| **PRD-FR-01** | `docs/PRD.md` | Requisito Funcional | Endpoint de cadastro de webhook recebendo URL e eventos ouvidos | `TRANSCRICAO` | `[09:31] Marcos` |
| **PRD-FR-02** | `docs/PRD.md` | Requisito Funcional | Geração de secret aleatória segura devolvida na criação do webhook | `TRANSCRICAO` | `[09:31] Marcos` |
| **PRD-FR-03** | `docs/PRD.md` | Requisito Funcional | Filtragem na inserção da outbox por status de pedido de interesse | `TRANSCRICAO` | `[09:33] Marcos` |
| **PRD-FR-04** | `docs/PRD.md` | Requisito Funcional | Operações de CRUD para gerenciar webhooks cadastrados | `TRANSCRICAO` | `[09:33] Bruno` |
| **PRD-FR-05** | `docs/PRD.md` | Requisito Funcional | Consulta de histórico de entregas com payload, status e tempo de resposta | `TRANSCRICAO` | `[09:34] Marcos` |
| **PRD-FR-06** | `docs/PRD.md` | Requisito Funcional | Rotação de secret com tolerância de convivência (grace period) de 24h | `TRANSCRICAO` | `[09:21] Sofia` |
| **PRD-FR-07** | `docs/PRD.md` | Requisito Funcional | Inserção atômica de evento na outbox durante a mudança de status | `TRANSCRICAO` | `[09:06] Diego` |
| **PRD-FR-08** | `docs/PRD.md` | Requisito Funcional | Assinatura HMAC-SHA256 no header X-Signature sobre o payload | `TRANSCRICAO` | `[09:20] Sofia` |
| **PRD-FR-09** | `docs/PRD.md` | Requisito Funcional | Header X-Event-Id com UUID fixo para deduplicação pelo cliente | `TRANSCRICAO` | `[09:25] Diego` |
| **PRD-FR-10** | `docs/PRD.md` | Requisito Funcional | Política de 5 tentativas com backoff progressivo (1m/5m/30m/2h/12h) | `TRANSCRICAO` | `[09:17] Diego` |
| **PRD-FR-11** | `docs/PRD.md` | Requisito Funcional | Redirecionamento de falhas esgotadas para tabela dedicada de DLQ | `TRANSCRICAO` | `[09:18] Diego` |
| **PRD-FR-12** | `docs/PRD.md` | Requisito Funcional | Endpoint de replay manual de DLQ protegido para role ADMIN | `TRANSCRICAO` | `[09:18] Diego` |
| **PRD-RNF-01** | `docs/PRD.md` | Requisito Não Funcional | Exigência mandatória de HTTPS/TLS para URLs cadastradas | `TRANSCRICAO` | `[09:23] Sofia` |
| **PRD-RNF-02** | `docs/PRD.md` | Requisito Não Funcional | Timeout de chamada HTTP outbound fixado em 10 segundos | `TRANSCRICAO` | `[09:42] Diego` |
| **PRD-RNF-03** | `docs/PRD.md` | Requisito Não Funcional | Limite estrito de 64 KB para o payload serializado do webhook | `TRANSCRICAO` | `[09:24] Diego` |
| **PRD-RNF-04** | `docs/PRD.md` | Requisito Não Funcional | Garantia de entrega at-least-once | `TRANSCRICAO` | `[09:24] Diego` |
| **PRD-OUT-01** | `docs/PRD.md` | Fora de Escopo | Disparo de alertas por e-mail em caso de falha de entrega | `TRANSCRICAO` | `[09:37] Larissa` |
| **PRD-OUT-02** | `docs/PRD.md` | Fora de Escopo | Criação de dashboard visual ou tela de frontend | `TRANSCRICAO` | `[09:40] Larissa` |
| **PRD-OUT-03** | `docs/PRD.md` | Fora de Escopo | Rate limiting outbound de envio para os clientes | `TRANSCRICAO` | `[09:39] Diego` |
| **RFC-PROP-01** | `docs/RFC.md` | Proposta Técnica | Adoção do Transactional Outbox Pattern no MySQL | `TRANSCRICAO` | `[09:06] Diego` |
| **RFC-PROP-02** | `docs/RFC.md` | Proposta Técnica | Worker assíncrono executando em processo separado | `TRANSCRICAO` | `[09:11] Diego` |
| **RFC-PROP-03** | `docs/RFC.md` | Proposta Técnica | Polling a cada 2 segundos no MySQL coberto por índices | `TRANSCRICAO` | `[09:09] Diego` |
| **RFC-ALT-01** | `docs/RFC.md` | Alternativa Rejeitada | Disparo síncrono no OrderService rejeitado por travar transação | `TRANSCRICAO` | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Alternativa Rejeitada | Uso de Redis Streams / RabbitMQ descartado por overengineering | `TRANSCRICAO` | `[09:07] Diego` |
| **RFC-ALT-03** | `docs/RFC.md` | Alternativa Rejeitada | Triggers de banco rejeitados por ausência de NOTIFY nativo no MySQL | `TRANSCRICAO` | `[09:09] Diego` |
| **RFC-ALT-04** | `docs/RFC.md` | Alternativa Rejeitada | Semântica Exactly-Once descartada pela complexidade distribuída | `TRANSCRICAO` | `[09:25] Diego` |
| **RFC-OPEN-01** | `docs/RFC.md` | Questão em Aberto | Avaliação de rate limiting outbound pós-lançamento sob monitoramento | `TRANSCRICAO` | `[09:39] Larissa` |
| **RFC-OPEN-02** | `docs/RFC.md` | Questão em Aberto | Particionamento ou lock pessimista para escala de múltiplos workers | `TRANSCRICAO` | `[09:13] Diego` |
| **RFC-PRAZO-01** | `docs/RFC.md` | Planejamento | Estimativa de 3 sprints com 2 dias de revisão da segurança por Sofia | `TRANSCRICAO` | `[09:46] Larissa` |
| **ADR-001-DEC** | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Uso de tabela webhook_outbox gravada na transação de pedidos | `TRANSCRICAO` | `[09:08] Larissa` |
| **ADR-002-DEC** | `docs/adrs/ADR-002-worker-processo-separado-polling.md` | Decisão | Worker em processo Node.js dedicado com polling a cada 2s | `TRANSCRICAO` | `[09:10] Larissa` |
| **ADR-003-DEC** | `docs/adrs/ADR-003-politica-retry-backoff-dlq.md` | Decisão | Janela de retry de 15h com 5 tentativas e tabela webhook_dead_letter | `TRANSCRICAO` | `[09:17] Larissa` |
| **ADR-004-DEC** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-rotacao-secret.md` | Decisão | HMAC-SHA256, secret única por endpoint e grace period de 24h | `TRANSCRICAO` | `[09:22] Sofia` |
| **ADR-005-DEC** | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Decisão | At-least-once com cabeçalho X-Event-Id e snapshot serializado | `TRANSCRICAO` | `[09:26] Larissa` |
| **ADR-005-SNAP** | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Decisão | Snapshot imutável de payload na inserção da outbox | `TRANSCRICAO` | `[09:52] Larissa` |
| **ADR-006-DEC** | `docs/adrs/ADR-006-reuso-padroes-existentes-projeto.md` | Decisão | Reuso integral de AppError, Pino, Zod e estrutura modular | `TRANSCRICAO` | `[09:30] Larissa` |
| **FDD-PAYLOAD-01** | `docs/FDD.md` | Especificação | Payload enxuto sem relação de items para respeitar limite de tamanho | `TRANSCRICAO` | `[09:43] Diego` |
| **FDD-HEADERS-01** | `docs/FDD.md` | Especificação | Headers X-Event-Id, X-Signature, X-Timestamp e X-Webhook-Id | `TRANSCRICAO` | `[09:44] Diego` |
| **FDD-ERRORS-01** | `docs/FDD.md` | Padrão | Prefixo WEBHOOK_ em todos os códigos de erro do módulo | `TRANSCRICAO` | `[09:29] Larissa` |
| **FDD-ROLE-01** | `docs/FDD.md` | Segurança | Requisito de role ADMIN para o endpoint de replay manual de DLQ | `TRANSCRICAO` | `[09:36] Sofia` |
| **INT-ORDERS-01** | `docs/FDD.md` | Integração de Código | Transação do changeStatus estendida com inserção na outbox | `CODIGO` | `src/modules/orders/order.service.ts` |
| **INT-ERROR-01** | `docs/FDD.md` | Integração de Código | Subclasses de erro de webhook derivadas da classe base AppError | `CODIGO` | `src/shared/errors/app-error.ts` |
| **INT-AUTH-01** | `docs/FDD.md` | Integração de Código | Autenticação JWT e proteção de rotas com requireRole | `CODIGO` | `src/middlewares/auth.middleware.ts` |
| **INT-LOGGER-01** | `docs/FDD.md` | Integração de Código | Instrumentação de logs estruturados utilizando o logger Pino | `CODIGO` | `src/shared/logger/index.ts` |
| **INT-MW-ERR-01** | `docs/FDD.md` | Integração de Código | Interceptação e serialização de erros no middleware centralizado | `CODIGO` | `src/middlewares/error.middleware.ts` |
| **INT-SCHEMA-01** | `docs/FDD.md` | Integração de Código | Modelagem de dados com UUIDs e enum OrderStatus no schema Prisma | `CODIGO` | `prisma/schema.prisma` |
| **INT-ENTRY-01** | `docs/FDD.md` | Integração de Código | Padrão de ponto de entrada paralelo a server.ts para worker.ts | `CODIGO` | `src/server.ts` |
| **INT-SCHEMA-ZOD** | `docs/FDD.md` | Integração de Código | Validação declarativa de entrada seguindo o padrão de schemas | `CODIGO` | `src/modules/orders/order.schemas.ts` |
