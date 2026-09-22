# ADR-005: Garantia de Entrega At-Least-Once com Header X-Event-Id e Snapshot Imutável de Payload

* **Status:** Aceito
* **Data:** 2026-09-22
* **Decisores:** Diego (Engenheiro de Plataforma), Bruno (Engenheiro de Pedidos), Larissa (Tech Lead), Sofia (Engenheira de Segurança), Marcos (Product Manager)
* **Origem na Reunião:** `[09:24] Diego`, `[09:25] Bruno`, `[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`, `[09:26] Larissa`, `[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`
* **Origem no Código:** `prisma/schema.prisma` (padrão de UUID com `@db.Char(36)`), `src/modules/orders/order.service.ts`

## Contexto

Em sistemas distribuídos sobre redes TCP/IP e HTTP, garantir que uma notificação seja entregue exatamente uma única vez (*exactly-once*) é um problema classicamente complexo que exige coordenação bilateral (two-phase commit) entre remetente e destinatário. Se o worker do OMS emitir a requisição HTTP, o cliente processar a notificação com sucesso, mas a conexão cair antes do retorno do HTTP 200 (ou se a resposta demorar mais que o timeout de 10s), o worker interpretará como falha de rede e agendará um reenvio.

Além disso, como o ciclo de vida do pedido pode sofrer mudanças subsequentes rápidas (ex: `PENDING` -> `PAID` -> `PROCESSING`), precisava-se definir se o evento na outbox armazenaria os dados do pedido já renderizados ou se buscaria o pedido do banco apenas no momento do disparo.

## Decisão

1. **Semântica At-Least-Once:** Assumir formalmente a semântica de entrega **ao menos uma vez** (*at-least-once*). O cliente receptor deve ser projetado para ser idempotente (`[09:24] Diego`, `[09:25] Diego`).
2. **Identificador Único (`X-Event-Id`):** Cada evento gerado na outbox receberá um identificador único universal (UUID v4), persistido na coluna `id` da tabela `webhook_outbox`. Esse mesmo UUID será enviado no cabeçalho HTTP `X-Event-Id` em todas as tentativas e retentativas daquele evento. O cliente utilizará esse identificador como chave de deduplicação em sua base de dados (`[09:25] Diego`).
3. **Snapshot Imutável do Payload na Inserção:** O corpo do evento será renderizado e serializado em JSON no momento exato em que a transação do `changeStatus` ocorrer (`[09:52] Larissa`, `[09:52] Diego`). Isso garante que, se o pedido for alterado novamente antes que o worker dispare a primeira mensagem, o evento representará com precisão o estado histórico da transição ocorrida (`from_status` e `to_status`), evitando efeitos colaterais de mutação concorrente.
4. **Comunicação e Documentação:** A responsabilidade de deduplicação e idempotência pelo consumidor será documentada com destaque no portal do desenvolvedor da empresa (`[09:26] Marcos`).

## Alternativas Consideradas

### 1. Garantia de Entrega Exactly-Once
* **Descrição:** Implementar protocolo de handshake bilateral ou transações distribuídas para certificar que nenhum evento seja processado duas vezes.
* **Motivo do Descarte:** Praticamente inviável em integrações heterogêneas sobre HTTP público entre empresas distintas. Exige sincronização de estado extremamente frágil e complexa. Provedores líderes de mercado (Stripe, GitHub, Twilio) operam todos sob semântica at-least-once com cabeçalho de evento (`[09:25] Diego`).

### 2. Renderização Tardia (Lazy Loading) do Payload no Envio
* **Descrição:** Gravar apenas `order_id` na outbox e fazer `prisma.order.findUnique` no momento em que o worker for realizar o disparo HTTP.
* **Motivo do Descarte:** Se o status do pedido mudar múltiplas vezes em sequência rápida (ex: `PAID` e logo após `PROCESSING`), o worker lendo tardiamente poderia enviar dois eventos com payloads idênticos refletindo o estado mais recente, perdendo a fotografia fiel da transição intermediária (`[09:51] Bruno`, `[09:52] Larissa`).

## Consequências

### Positivas
* **Confiabilidade Extrema:** Nenhum evento é perdido por falha transitória de comunicação.
* **Consistência Histórica dos Dados:** O snapshot preserva a auditoria e integridade do evento independentemente de mutações posteriores na tabela `orders`.
* **Idempotência Simples para o Consumidor:** O consumidor precisa apenas armazenar os `X-Event-Id` já processados em cache ou índice único no banco de dados.

### Negativas e Mitigações
* **Possibilidade de Duplicação no Cliente:** Se houver oscilação de rede no retorno HTTP, o cliente receberá a mesma mensagem novamente.
  * *Mitigação:* O cabeçalho `X-Event-Id` é constante entre retentativas do mesmo evento, tornando trivial a verificação e descarte de duplicatas pelo receptor.
* **Consumo de Armazenamento:** Salvar o snapshot do payload em JSON na outbox ocupa mais espaço do que salvar apenas chaves estrangeiras.
  * *Mitigação:* O payload foi desenhado de forma enxuta (dados básicos do pedido, sem a lista detalhada de itens `OrderItem`), mantendo o tamanho médio em ~1KB, bem abaixo do teto de 64KB (`[09:43] Diego`, `[09:44] Bruno`).
