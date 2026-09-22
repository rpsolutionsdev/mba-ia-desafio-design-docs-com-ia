# ADR-002: Worker em Processo Separado em Polling

* **Status:** Aceito
* **Data:** 2026-09-22
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro de Plataforma), Bruno (Engenheiro de Pedidos), Marcos (Product Manager)
* **Origem na Reunião:** `[09:08] Diego`, `[09:09] Diego`, `[09:10] Larissa`, `[09:11] Diego`, `[09:11] Larissa`, `[09:12] Diego`, `[09:28] Bruno`, `[09:30] Bruno`
* **Origem no Código:** `src/server.ts`, `src/config/database.ts`

## Contexto

Para esvaziar a tabela `webhook_outbox` e disparar as notificações HTTP outbound para os clientes B2B, é necessário um mecanismo consumidor. A equipe precisou definir a arquitetura de execução desse consumidor: se rodaria em segundo plano acoplado ao servidor HTTP da API (`src/server.ts`) ou como um processo independente, bem como a estratégia de detecção de novos eventos (polling versus eventos acionados por triggers de banco).

Adicionalmente, os clientes necessitam de previsibilidade e entrega rápida (abaixo de 10 segundos, conforme alinhamento de produto).

## Decisão

1. **Processo Independente:** O worker será executado em um processo Node.js dedicado, com ponto de entrada próprio em `src/worker.ts` e comando de inicialização `npm run worker`. A lógica de processamento de lote residirá no módulo de webhooks (`src/modules/webhooks/webhook.worker.ts`).
2. **Conexão de Banco Isolada:** O worker utilizará sua própria instância de `PrismaClient` com pool de conexões separado, compartilhando as mesmas credenciais e variáveis de ambiente (`DATABASE_URL`).
3. **Estratégia de Polling:** O worker executará um loop contínuo de polling a cada 2 segundos, consultando lotes reduzidos de eventos com status `PENDING` ordenados por `created_at ASC`.
4. **Instância Única (Single-Worker):** Na fase inicial, operará como single-worker para garantir a ordenação de entrega cronológica por pedido sem necessidade de locks distribuídos complexos.

## Alternativas Consideradas

### 1. Worker Executado no Mesmo Processo da API HTTP
* **Descrição:** Inicializar um timer (`setInterval`) dentro de `src/server.ts` compartilhando o ciclo de vida do Express.
* **Motivo do Descarte:** Se a API reiniciar por deploy, rotação de pods ou crash decorrente de requisições web, a execução do worker é interrompida. Além disso, tarefas pesadas de rede de outbound poderiam competir pelo Event Loop com o tráfego da API pública (`[09:11] Diego`).

### 2. Triggers de Banco de Dados com Notificação Externa
* **Descrição:** Criar triggers no MySQL para notificar o consumidor imediatamente após a inserção na outbox.
* **Motivo do Descarte:** O MySQL não possui mecanismos nativos de publicação/subscrição pub-sub como o `LISTEN`/`NOTIFY` do PostgreSQL. Triggers no MySQL só executam SQL e não notificam processos externos sem gambiarras arriscadas (como escrita em disco ou chamada externa). Polling de 2 segundos atende com ampla folga o requisito de entrega em menos de 10 segundos (`[09:09] Diego`).

## Consequências

### Positivas
* **Resiliência e Desacoplamento:** Falhas, restarts e oscilações na API HTTP não interrompem a entrega de webhooks, e vice-versa.
* **Simplicidade de Manutenção:** O mecanismo de polling a cada 2s é determinístico, simples de instrumentar e não exige nenhuma extensão proprietária de banco.
* **Ordenação Natural por Pedido:** O processamento sequencial por lote pelo single-worker preserva a ordem cronológica de eventos para o mesmo pedido (`[09:12] Diego`).

### Negativas e Mitigações
* **Overhead de Consultas ao MySQL:** O polling a cada 2 segundos executa consultas regulares mesmo quando não há eventos pendentes.
  * *Mitigação:* A query é altamente eficiente, filtrando por `status = 'PENDING'` coberto por índice composto em `(status, created_at)` e limitando o batch (`LIMIT N`), gerando carga insignificante no banco.
* **Limitação de Escalabilidade Horizontal:** Rodar múltiplos workers concorrentes no futuro exigirá particionamento de pedidos ou lock pessimista (`SELECT FOR UPDATE SKIP LOCKED`).
  * *Mitigação:* Aceito como limitação conhecida da fase 1; o throughput atual de transições de pedidos no OMS é confortavelmente suportado por um único worker (`[09:13] Diego`).
