# ADR-003: Política de Retry com Backoff Exponencial e Tabela Dedicada de Dead Letter Queue (DLQ)

* **Status:** Aceito
* **Data:** 2026-09-22
* **Decisores:** Larissa (Tech Lead), Diego (Engenheiro de Plataforma), Bruno (Engenheiro de Pedidos), Sofia (Engenheira de Segurança), Marcos (Product Manager)
* **Origem na Reunião:** `[09:14] Larissa`, `[09:15] Diego`, `[09:15] Bruno`, `[09:16] Diego`, `[09:17] Diego`, `[09:18] Diego`, `[09:19] Larissa`, `[09:35] Larissa`, `[09:36] Sofia`, `[09:37] Larissa`
* **Origem no Código:** `src/middlewares/auth.middleware.ts` (função `requireRole`), `src/shared/errors/app-error.ts`

## Contexto

Endpoints receptores operados por clientes B2B estão sujeitos a falhas temporárias de rede, lentidão, manutenções programadas ou incidentes internos. Se o worker de webhooks tentar entregar um evento e receber um erro de rede, timeout (acima de 10s) ou status HTTP 4xx/5xx, o sistema precisa reagir de forma resiliente, sem descartar o evento prematuramente nem sobrecarregar o cliente com retentativas agressivas.

Além disso, após o esgotamento das tentativas, é fundamental manter evidência auditável do evento com a causa da falha, permitindo suporte operacional e reprocessamento posterior sem poluir a tabela de outbox ativa.

## Decisão

1. **Tentativas e Backoff Exponencial:** Estabelecer um limite de **5 tentativas de reenvio**, seguindo intervalos progressivos de backoff:
   * 1ª retentativa: após **1 minuto**
   * 2ª retentativa: após **5 minutos**
   * 3ª retentativa: após **30 minutos**
   * 4ª retentativa: após **2 horas**
   * 5ª retentativa: após **12 horas**
   * Janela total acumulada de resiliência: aproximadamente **15 horas** desde o primeiro erro.
2. **Timeout por Requisição:** Fixar timeout estrito de **10 segundos** por chamada HTTP outbound (`[09:42] Diego`). Respostas além desse tempo são abortadas e computadas como falha de tentativa.
3. **Tabela Dedicada de DLQ (`webhook_dead_letter`):** Esgotadas as 5 tentativas sem sucesso, o registro é movido da `webhook_outbox` para a tabela `webhook_dead_letter`, gravando o payload completo original, código de erro/motivo da falha, contador de tentativas e timestamp de descarte.
4. **Replay Manual Auditado:** Disponibilizar o endpoint `POST /admin/webhooks/dead-letter/:id/replay` para permitir que a equipe de suporte/operações reenfileire eventos na outbox como `PENDING`.
5. **Controle de Acesso ao Replay:** O endpoint de replay é estritamente restrito a usuários com a role `ADMIN` (utilizando o middleware existente `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`), com registro obrigatório de log de auditoria via Pino com o ID do administrador responsável.
6. **Fora de Escopo da Fase Atual:** O envio de e-mails automáticos ao cliente alertando sobre falhas consecutivas de webhook foi explicitamente descartado para esta fase (`[09:37] Larissa`).

## Alternativas Consideradas

### 1. Política de 3 Tentativas Rápidas (Backoff Curto)
* **Descrição:** Retentar 3 vezes em intervalos curtos (ex: 1m, 5m, 15m), cobrindo no máximo 30 minutos.
* **Motivo do Descarte:** Clientes B2B frequentemente realizam janelas de manutenção de 1 a 2 horas. Três tentativas rápidas declarariam falha definitiva enquanto a janela de manutenção do cliente ainda estaria em andamento, descartando notificações legítimas (`[09:16] Diego`).

### 2. Retentativas Indefinidas (Retry Infinito)
* **Descrição:** Continuar retentando infinitamente com backoff fixo até o endpoint do cliente responder.
* **Motivo do Descarte:** Endpoints permanentemente desativados ou clientes cancelados causariam acúmulo contínuo de registros e desperdício permanente de recursos de rede do worker (`[09:15] Diego`).

### 3. DLQ Marcada apenas como Coluna de Status na Própria Outbox
* **Descrição:** Deixar registros com status `DEAD_LETTER` dentro da tabela `webhook_outbox`.
* **Motivo do Descarte:** Polui a tabela operacional de alta rotatividade com registros mortos, prejudicando queries do worker e complicando rotinas de retenção. Uma tabela separada oferece isolamento físico, clareza para consultas de suporte e facilidade de arquivamento (`[09:18] Diego`).

## Consequências

### Positivas
* **Alta Tolerância a Falhas:** Cobre interrupções de curta e média duração de clientes (até 15 horas) sem intervenção humana.
* **Isolamento e Segurança Operacional:** A tabela `webhook_outbox` permanece enxuta e focada em eventos quentes, enquanto a `webhook_dead_letter` serve como repositório de auditoria para o time de suporte.
* **Rastreabilidade Administrativa:** Apenas administradores auditados podem reprocessar eventos mortos.

### Negativas e Mitigações
* **Complexidade no Cálculo de Next Retry:** A tabela de outbox precisa armazenar `attempts` e `next_retry_at`, exigindo atualização após cada tentativa frustrada.
  * *Mitigação:* A query do worker selecionará apenas `status = 'PENDING' AND next_retry_at <= NOW()`, utilizando índice eficiente.
* **Necessidade de Replay Manual em Quedas Longas:** Quedas superiores a 15 horas demandam ação de suporte via API.
  * *Mitigação:* Documentado como procedimento operacional padrão (SOP) para o time de suporte via endpoint de replay.
