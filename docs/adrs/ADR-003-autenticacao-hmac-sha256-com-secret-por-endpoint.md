# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period

- **Status:** Aceito
- **Data:** 2026-10-26
- **Decisores:** Sofia (Engenheira de Segurança), Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (Product Manager)

---

## Contexto

O Sistema de Webhooks enviará notificações HTTP `POST` contendo dados de pedidos de clientes B2B para endpoints externos fora da infraestrutura do OMS (`[09:19] Sofia`). Sem um mecanismo criptográfico de autenticação e integridade:
1. Um atacante poderia forjar requisições falsas para o endpoint do cliente se passando pelo OMS (*spoofing*).
2. O conteúdo do payload poderia ser adulterado em trânsito (*tampering*).
3. Requisições legítimas interceptadas poderiam ser reenviadas maliciosamente (*replay attack*) (`[09:19] Sofia`, `[09:44] Diego`).
4. Caso uma chave compartilhada vaze nos logs ou na infraestrutura de um cliente (situação que já ocorreu anteriormente na empresa, conforme relatado em `[09:22] Diego`), o cliente precisa ser capaz de rotacionar sua credencial sem sofrer indisponibilidade ou perda de webhooks durante a janela de atualização dos seus servidores (`[09:21] Sofia`, `[09:22] Diego`).

---

## Decisão

Adotar assinatura criptográfica **HMAC-SHA256** sobre o corpo da requisição HTTP, utilizando uma **secret exclusiva por endpoint de webhook** gerada pelo OMS e suporte a **rotação de secret com grace period de 24 horas**:

1. **Algoritmo e Header de Assinatura:** O worker assinará o corpo (payload JSON serializado) da requisição utilizando **HMAC-SHA256** e enviará a assinatura no header HTTP `X-Signature`, acompanhada dos headers `X-Timestamp` (timestamp do envio para mitigação de *replay attacks*), `X-Event-Id` (UUID do evento) e `X-Webhook-Id` (identificador da configuração do webhook) (`[09:20] Sofia`, `[09:22] Sofia`, `[09:44] Diego`, `[09:44] Sofia`).
2. **Secret Única por Endpoint de Webhook:** Cada registro de configuração de webhook terá sua própria `secret` gerada automaticamente pelo OMS no momento do cadastro (`POST`) e devolvida na resposta de criação (`[09:21] Sofia`, `[09:21] Bruno`, `[09:31] Marcos`). Não haverá secret global compartilhada entre endpoints ou clientes (`[09:21] Sofia`).
3. **Rotação de Secret com Grace Period de 24 Horas:** O sistema disponibilizará um endpoint autenticado para o cliente solicitar a rotação da secret de um webhook. Ao rotacionar, uma nova secret é gerada imediatamente e a secret anterior permanece válida em paralelo por **24 horas** (*grace period*), dando tempo para o cliente atualizar seus sistemas antes da invalidação definitiva da chave antiga (`[09:21] Sofia`, `[09:22] Sofia`, `[09:48] Larissa`).
4. **Requisitos Complementares de Segurança no Transporte:**
   - Exigir obrigatoriamente URLs com esquema `https://` na validação de schema Zod (`[09:23] Sofia`).
   - Recusar com erro (sem truncar) qualquer evento cujo payload ultrapasse **64KB** (`[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`).

---

## Alternativas Consideradas

### 1. Secret Global Única para Toda a Plataforma OMS
- **Descrição:** Utilizar uma única chave secreta global da plataforma para assinar todos os webhooks enviados a todos os clientes (`[09:21] Sofia`).
- **Motivo do Descarte:** Risco severo de segurança (*blast radius* total). Se um único cliente vazar a secret em seus logs ou código, todas as integrações de todos os clientes da plataforma estariam comprometidas e precisariam ser reconfiguradas simultaneamente (`[09:21] Sofia`, `[09:22] Diego`).

### 2. Rotação Imediata de Secret (Sem Grace Period)
- **Descrição:** Invalidar instantaneamente a secret anterior no exato momento em que uma nova secret é gerada na API (`[09:21] Sofia`).
- **Motivo do Descarte:** Causaria falha de validação de assinatura em todos os webhooks disparados entre o momento da chamada de rotação na API do OMS e o deploy/atualização da nova secret nos servidores do cliente. O grace period de 24 horas permite migração sem downtime (`[09:21] Sofia`).

### 3. Outros Algoritmos de Hash ou Sem Assinatura Criptográfica
- **Descrição:** Enviar requisições sem assinatura ou com algoritmos obsoletos/não padronizados (`[09:19] Sofia`, `[09:20] Bruno`).
- **Motivo do Descarte:** Sem assinatura, o cliente não consegue validar autenticidade nem integridade do payload. O `HMAC-SHA256` foi escolhido por ser o padrão consolidado de mercado e possuir suporte nativo em praticamente qualquer linguagem/biblioteca utilizada pelos clientes (`[09:20] Sofia`).

---

## Consequências

### Positivas
- **Autenticidade e Integridade Fim a Fim:** Os clientes podem verificar criptograficamente que o payload veio do OMS e não foi modificado em trânsito (`[09:19] Sofia`, `[09:20] Sofia`).
- **Isolamento de Falhas de Segurança (Menor Privilégio):** O comprometimento da secret de um endpoint afeta exclusivamente aquele endpoint específico (`[09:21] Sofia`).
- **Rotação Zero-Downtime:** O período de graça de 24 horas viabiliza resposta a incidentes (como vazamento de secret em logs do cliente) e políticas periódicas de rotação sem perda de eventos (`[09:21] Sofia`, `[09:22] Diego`).
- **Proteção contra Replay Attacks:** O envio do header `X-Timestamp` junto ao `X-Signature` permite que o cliente rejeite requisições antigas retransmitidas (`[09:44] Diego`).

### Negativas
- **Complexidade Adicional no Modelo de Dados e Assinatura:** A tabela de configuração de webhooks precisa armazenar tanto a secret atual quanto a secret anterior (com validade paralela de +24h), além de exigir suporte durante a janela de transição (`[09:21] Sofia`).
- **Necessidade de Revisão Dedicada de Segurança e Cuidado com Credenciais:** Como já houve caso real de cliente que vazou secret no log da própria aplicação (`[09:22] Diego`), o fluxo de geração/rotação de secret e cálculo de HMAC exigirá reserva de pelo menos **2 dias úteis** para revisão dedicada de segurança antes do deploy (`[09:46] Sofia`), além de manter as boas práticas de redaction do logger Pino no OMS (`src/shared/logger/index.ts`).

### Trade-offs
- **Segurança e Continuidade Operacional (Grace Period 24h) vs. Simplicidade de Chave Única Estática:** Aceita-se maior complexidade no armazenamento de estado das credenciais (gerenciar chave ativa + chave anterior temporária por endpoint) em troca de isolamento criptográfico por endpoint e rotação sem janela de indisponibilidade nos clientes (`[09:21] Sofia`, `[09:22] Sofia`).

---

## Referências

### Transcrição (`TRANSCRICAO.md`)
- `[09:19] Sofia` / `[09:20] Sofia` / `[09:20] Bruno` — Necessidade de validar origem/integridade e escolha do algoritmo `HMAC-SHA256` enviado no header `X-Signature`.
- `[09:21] Sofia` / `[09:21] Bruno` — Decisão de secret única por endpoint de webhook e rotação de secret via API com validade paralela por 24 horas.
- `[09:22] Diego` / `[09:22] Sofia` — Histórico real de vazamento de secret em log de cliente e confirmação da decisão (HMAC-SHA256, secret por endpoint, grace period de 24h).
- `[09:23] Sofia` / `[09:24] Diego` / `[09:24] Larissa` — TLS obrigatório (`https://`) validado no Zod e limite de payload de 64KB.
- `[09:31] Marcos` — Secret gerada pelo OMS e devolvida na criação do webhook.
- `[09:44] Diego` / `[09:44] Sofia` / `[09:45] Diego` — Definição dos headers `X-Signature`, `X-Timestamp`, `X-Event-Id` e `X-Webhook-Id`.
- `[09:46] Sofia` — Reserva de 2 dias úteis para revisão de código de segurança (HMAC e geração de secret).

### Código-Fonte do Repositório
- `src/shared/logger/index.ts` (linhas 4–20) — Configuração `redactPaths` do Pino (`req.headers.authorization`, `*.password`, `*.token`, etc.), padrão que deve ser preservado para nunca vazar secrets em logs.
- `src/middlewares/validate.middleware.ts` (linhas 11–37) — Middleware de validação Zod onde a exigência de URL `https://` será aplicada.
