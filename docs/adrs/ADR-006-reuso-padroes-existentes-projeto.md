# ADR-006: Reuso dos Padrões Arquiteturais e Estruturais Existentes do Projeto

* **Status:** Aceito
* **Data:** 2026-09-22
* **Decisores:** Bruno (Engenheiro de Pedidos), Larissa (Tech Lead), Diego (Engenheiro de Plataforma), Sofia (Engenheira de Segurança)
* **Origem na Reunião:** `[09:27] Bruno`, `[09:28] Bruno`, `[09:28] Diego`, `[09:29] Bruno`, `[09:29] Larissa`, `[09:30] Larissa`, `[09:36] Sofia`, `[09:36] Larissa`, `[09:40] Bruno`, `[09:41] Diego`
* **Origem no Código:** `src/shared/errors/app-error.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/config/logger.ts`, `src/modules/orders/order.service.ts`

## Contexto

A base de código do OMS já possui uma arquitetura bem delineada e consistente em Node.js com TypeScript, estruturada em camadas bem definidas e convenções consolidadas:
* Arquitetura modular por domínio dentro de `src/modules/` (composto por controller, service, repository, routes e schemas).
* Tratamento centralizado de exceções baseado na classe `AppError` (`src/shared/errors/app-error.ts`) interceptado pelo `errorHandler` (`src/middlewares/error.middleware.ts`).
* Validação declarativa de entrada via bibliotecas de schema Zod.
* Logging estruturado em formato JSON padronizado com a biblioteca Pino (`src/config/logger.ts`).
* Controle de acesso baseado em papéis via middleware `requireRole` (`src/middlewares/auth.middleware.ts`).
* Persistência relacional transacional gerenciada pelo Prisma Client no MySQL.

Ao introduzir a funcionalidade de Webhooks, a equipe avaliou se seria necessário introduzir novos frameworks, bibliotecas alternativas de log/validação ou estruturas arquiteturais diferenciadas.

## Decisão

Adotar o **reuso integral e estrito de todos os padrões e convenções já existentes** no projeto:
1. **Estrutura Modular:** Criar o módulo `src/modules/webhooks/` respeitando rigorosamente a divisão de responsabilidades da aplicação:
   * `webhook.controller.ts`: manipulação de requisições HTTP e serialização de respostas.
   * `webhook.service.ts`: regras de negócio, gerenciamento de status e rotação de chaves.
   * `webhook.repository.ts`: abstração de queries no Prisma para tabelas de configuração, outbox, deliveries e DLQ.
   * `webhook.routes.ts`: definição de rotas e acoplamento dos middlewares.
   * `webhook.schemas.ts`: validações Zod para criação, atualização e parâmetros.
   * `webhook.worker.ts`: rotina de processamento assíncrono de lotes e disparos de rede.
2. **Hierarquia de Erros e Padrão de Códigos:** Reutilizar a classe base `AppError` para todas as falhas de domínio e regras de negócio do novo módulo, adotando o prefixo obrigatório `WEBHOOK_*` nos códigos de erro (`[09:28] Bruno`, `[09:29] Larissa`):
   * `WEBHOOK_NOT_FOUND` (404)
   * `WEBHOOK_INVALID_URL` (400)
   * `WEBHOOK_SECRET_REQUIRED` (400)
   * `WEBHOOK_PAYLOAD_TOO_LARGE` (413)
   * `WEBHOOK_DELIVERY_FAILED` (502)
   * `WEBHOOK_REPLAY_INVALID` (400)
3. **Tratamento Centralizado de Erros:** Manter o `error.middleware.ts` sem alterações estruturais, uma vez que ele já captura nativamente instâncias de `AppError`, erros de validação do Zod (`ZodError`) e falhas de banco do Prisma (`PrismaClientKnownRequestError`).
4. **Logging com Pino:** Instrumentar todos os pontos do ciclo de vida dos webhooks (inserção no outbox, tentativas do worker, assinaturas HMAC e replays) com o logger centralizado `src/config/logger.ts`.
5. **Controle de Acesso com `requireRole`:** Utilizar o middleware `authMiddleware` existente para validar o JWT em todas as rotas e `requireRole(UserRole.ADMIN)` especificamente para a rota de replay manual da DLQ (`[09:36] Sofia`, `[09:36] Larissa`).
6. **Integração Transacional Leve:** Integrar o disparo de webhooks com o serviço `OrderService.changeStatus` por meio de uma função publicadora pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o cliente de transação existente (`Prisma.TransactionClient`), sem necessidade de injetar o repositório inteiro de pedidos ou alterar o contrato do repositório (`[09:41] Bruno`, `[09:41] Diego`).

## Alternativas Consideradas

### 1. Criar um Microsserviço Independente para Webhooks
* **Descrição:** Desenvolver um serviço autônomo em outro repositório ou subpasta com stack própria para cuidar de webhooks.
* **Motivo do Descarte:** Aumentaria drasticamente a complexidade de deploy, pipeline de CI/CD e governança sem necessidade técnica justificável. Manter como módulo do monólito modular existente compartilha modelos Prisma e evita duplicação de boilerplate (`[09:27] Bruno`, `[09:30] Larissa`).

### 2. Introduzir Nova Biblioteca de Fila em Memória / BullMQ
* **Descrição:** Instalar BullMQ / BeeQueue com Redis para orquestrar as retentativas.
* **Motivo do Descarte:** Requereria infraestrutura adicional (Redis) rejeitada no ADR-001. A base existente com MySQL + Prisma suporta nativamente o padrão Outbox e controle de retry sem dependências externas (`[09:07] Diego`).

## Consequências

### Positivas
* **Curva de Aprendizado Zero:** Qualquer engenheiro do time que já conhece a aplicação existente consegue ler, manter e evoluir o módulo de webhooks imediatamente.
* **Alta Coesão e Baixo Acoplamento:** O acoplamento com o domínio de pedidos limita-se à invocação de `publishWebhookEvent` dentro da transação existente.
* **Segurança e Auditoria Uniformes:** Reaproveita autenticação JWT, permissões e sanitização de erros já consolidadas em produção.

### Negativas e Mitigações
* **Crescimento do Monólito Modular:** Adiciona novos modelos e rotas dentro da mesma base de código.
  * *Mitigação:* O isolamento estrito dentro de `src/modules/webhooks` garante fronteiras limpas, facilitando futura extração caso a aplicação venha a ser decomposta em microsserviços no futuro.
