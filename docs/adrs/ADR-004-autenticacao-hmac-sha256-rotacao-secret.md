# ADR-004: Autenticação por Assinatura HMAC-SHA256, Secret Única por Endpoint e Rotação com Grace Period

* **Status:** Aceito
* **Data:** 2026-09-22
* **Decisores:** Sofia (Engenheira de Segurança), Diego (Engenheiro de Plataforma), Bruno (Engenheiro de Pedidos), Larissa (Tech Lead), Marcos (Product Manager)
* **Origem na Reunião:** `[09:19] Sofia`, `[09:20] Sofia`, `[09:21] Sofia`, `[09:21] Bruno`, `[09:22] Sofia`, `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`, `[09:44] Diego`, `[09:46] Sofia`
* **Origem no Código:** `src/modules/orders/order.schemas.ts` (padrão de validação Zod), `src/shared/errors/app-error.ts`

## Contexto

As notificações de pedidos transmitem dados comerciais e cadastrais sensíveis (ex: identificadores de pedido, clientes, valores totais e transições de status) para servidores externos dos clientes B2B via internet pública. Os clientes precisam garantir:
1. **Autenticidade da Origem:** Certeza técnica de que a requisição partiu legitimamente dos servidores do OMS e não de um atacante.
2. **Integridade da Carga:** Garantia de que o corpo da mensagem (payload JSON) não sofreu adulteração ou corrupção em trânsito (man-in-the-middle).
3. **Não Repúdio e Prevenção contra Replay:** Mecanismo para atestar o momento do envio e evitar reenvios maliciosos.

Adicionalmente, caso uma credencial seja comprometida ou precise ser renovada periodicamente, o cliente deve conseguir rotacionar a chave sem causar indisponibilidade em suas integrações produtivas.

## Decisão

1. **Assinatura HMAC-SHA256:** Toda requisição outbound incluirá o cabeçalho `X-Signature` contendo a assinatura hexadecimal do corpo do request (`JSON stringified`) calculada utilizando o algoritmo HMAC-SHA256 com a secret do endpoint (`[09:20] Sofia`).
2. **Secret Única por Endpoint:** Cada endpoint de webhook cadastrado terá sua própria secret criptograficamente aleatória (gerada pelo sistema na criação e nunca fornecida pelo cliente). É vedado o uso de chave simétrica global compartilhada entre múltiplos clientes ou endpoints (`[09:21] Sofia`).
3. **Mecanismo de Rotação com Grace Period:**
   * A API fornecerá endpoint de rotação (`POST /webhooks/:id/rotate-secret`).
   * Ao rotacionar, uma nova secret primária é gerada. A secret anterior permanece temporariamente válida para validação por um **período de tolerância (grace period) de 24 horas**, após o qual é revogada em definitivo (`[09:21] Sofia`, `[09:22] Sofia`).
4. **HTTPS / TLS Estritamente Obrigatório:** Endpoints com protocolo inseguro `http://` serão sumariamente rejeitados no momento do cadastro ou edição via validação de schema Zod (`[09:23] Sofia`).
5. **Cabeçalhos de Segurança Adicionais:**
   * `X-Timestamp`: Timestamp Unix / ISO 8601 do envio para mitigação de replay attacks (`[09:44] Diego`).
   * `X-Webhook-Id`: Identificador do cadastro do webhook receptor para clientes com múltiplos endpoints (`[09:45] Sofia`).
6. **Limite Estrito de Tamanho de Payload:** O payload JSON gerado é limitado a no máximo **64 KB**. Mensagens que excedam essa dimensão não serão truncadas nem enviadas; gerarão erro de validação e descarte com alerta (`[09:24] Sofia`, `[09:24] Diego`).
7. **Revisão Formal de Segurança:** O time de segurança (Sofia) terá uma janela de 2 dias úteis reservada para auditoria do código de criptografia e gerenciamento de secrets antes do deploy em produção (`[09:46] Sofia`).

## Alternativas Consideradas

### 1. Secret Simétrica Global da Plataforma
* **Descrição:** Utilizar uma única secret configurada no OMS para assinar todos os webhooks de todos os clientes.
* **Motivo do Descarte:** Se a secret for vazada ou exposta nos logs de um único cliente, toda a integridade de notificações da plataforma é comprometida (`[09:21] Sofia`, `[09:22] Diego`).

### 2. Autenticação Básica (HTTP Basic Auth) ou Bearer Token Estático
* **Descrição:** Enviar um token estático no header `Authorization`.
* **Motivo do Descarte:** Tokens estáticos não protegem o conteúdo da mensagem contra adulteração (não fornecem integridade) e são vulneráveis a vazamento e interceptação direta se houver falhas intermediárias.

### 3. Truncamento Automático de Payloads Grandes
* **Descrição:** Se o payload ultrapassasse o limite seguro, truncar campos ou itens para forçar o envio.
* **Motivo do Descarte:** Truncar corrompe a estrutura do JSON e gera dados incompletos ou inválidos para os parsers do cliente. O correto para integridade de dados é rejeitar formalmente com erro (`[09:24] Sofia`).

## Consequências

### Positivas
* **Conformidade com Padrões de Mercado:** Padrão alinhado ao adotado por referências globais de API (Stripe, GitHub, Shopify).
* **Zero Downtime em Rotação:** O cliente tem 24 horas para atualizar suas variáveis de ambiente sem perder nenhuma notificação em trânsito.
* **Confinamento de Risco:** O comprometimento de uma chave afeta exclusivamente o endpoint correspondente daquele cliente específico.

### Negativas e Mitigações
* **Custo Computacional de Criptografia no Worker:** Assinar cada payload antes do envio consome ciclos de CPU.
  * *Mitigação:* O algoritmo HMAC-SHA256 em Node.js utiliza bindings nativos do OpenSSL (`crypto.createHmac`), executando em submilissegundos para payloads menores que 64KB.
* **Responsabilidade de Validação no Cliente:** O cliente consumidor é obrigado a codificar a validação do HMAC em seu receptor.
  * *Mitigação:* Documentação detalhada no portal do desenvolvedor com exemplos de código em várias linguagens (`[09:26] Marcos`).
