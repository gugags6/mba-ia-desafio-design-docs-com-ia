# Tracker de Rastreabilidade da Documentação

Este documento mapeia cada requisito, decisão arquitetural, alternativa descartada, restrição quantitativa, contrato técnico e ponto de integração registrados nos documentos do pacote (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-001` a `ADR-006`) à sua respectiva origem comprovável na transcrição da reunião (`TRANSCRICAO.md`) ou no código-fonte existente do repositório.

---

## 1. Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Demanda formal de 3 clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) e risco de churn da Atlas até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Registro do cliente B2B Logística Atlas Comercial Ltda na base de dados inicial do OMS | CODIGO | prisma/seed.ts |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Notificação em "tempo real" definida pelos clientes como atraso inferior a 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Prazo de entrega estimado em 3 sprints para atender à janela de fim de novembro da Atlas Comercial | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Reserva obrigatória de pelo menos 2 dias úteis para revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Cobertura de indisponibilidades e manutenções de clientes por janela de quase 15 horas (5 tentativas) | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Validade paralela (grace period) de 24 horas para a secret antiga após rotação pelo cliente | TRANSCRICAO | [09:21] Sofia |
| PRD-ESC-01 | docs/PRD.md | Fora de Escopo | Exclusão de webhooks inbound; fluxo restrito exclusivamente a notificações outbound (do OMS para o cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-02 | docs/PRD.md | Fora de Escopo | Arquivamento/expurgo de eventos entregues após 30 dias na outbox fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-03 | docs/PRD.md | Fora de Escopo | Envio de e-mail de alerta ao cliente após 3 falhas seguidas descartado para esta fase (adiado para fase futura) | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-04 | docs/PRD.md | Fora de Escopo | Rate limiting de saída por cliente não implementado agora; mantido em observação para decisão futura | TRANSCRICAO | [09:39] Diego |
| PRD-ESC-05 | docs/PRD.md | Fora de Escopo | Dashboard/painel visual fora do escopo; entrega restrita aos endpoints de API | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-06 | docs/PRD.md | Fora de Escopo | Garantia de ordenação global fora de escopo; operação inicial single-worker com ordem implícita por order_id | TRANSCRICAO | [09:13] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Endpoint POST para cadastro de webhook com URL, filtro de status e secret gerada e devolvida pelo OMS | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Passagem explícita do customer_id no body ou path (não extraído do JWT de usuário operador) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Endpoints GET para listar, PATCH para editar e DELETE para remover configurações de webhook de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro seletivo de status de pedido avaliado na hora da inserção na webhook_outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Inserção atômica do evento na webhook_outbox dentro da mesma transação de mudança de status do pedido | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Persistência do snapshot do payload JSON já renderizado no momento da inserção na outbox | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Envio de headers padronizados X-Event-Id, X-Signature, X-Timestamp e Content-Type application/json | TRANSCRICAO | [09:44] Diego |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Inclusão do header X-Webhook-Id identificando o cadastro do endpoint receptor | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Endpoint de rotação de secret pela API mantendo a secret antiga válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Endpoint GET /webhooks/:id/deliveries retornando histórico das últimas 100 entregas com payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Endpoint POST /admin/webhooks/dead-letter/:id/replay para reprocessar eventos da DLQ como pendentes | TRANSCRICAO | [09:18] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Worker em polling a cada 2 segundos para cumprir meta de latência inferior a 10 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Execução do worker em processo separado da API para evitar perda do worker em reinicializações | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Obrigatoriedade de TLS (URL https://) validada via schema Zod, recusando http:// | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite máximo de 64KB para o payload do evento, retornando erro sem truncar caso ultrapasse | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos para a chamada HTTP realizada pelo worker | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once com deduplicação no cliente via UUID no header X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Exigência de role ADMIN no replay da DLQ com registro de log de auditoria de quem executou a ação | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Padronização de todos os códigos de erro do módulo com o prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| PRD-RSK-01 | docs/PRD.md | Risco | Risco de indisponibilidade de cliente em manutenção planejada (histórico real de 2 horas) mitigado por 5 retries | TRANSCRICAO | [09:16] Diego |
| PRD-RSK-02 | docs/PRD.md | Risco | Risco de vazamento de secret em log de aplicação do cliente mitigado por secret única por endpoint e rotação | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-03 | docs/PRD.md | Risco | Mitigação de entregas duplicadas via documentação destacada de deduplicação no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| RFC-PRB-01 | docs/RFC.md | Contexto | Transação pesada atual em changeStatus (orders, order_status_history e stock_quantity) inviabiliza HTTP síncrono | TRANSCRICAO | [09:04] Bruno |
| RFC-PRP-01 | docs/RFC.md | Decisão | Adoção do padrão Transactional Outbox no MySQL existente com tabela webhook_outbox | TRANSCRICAO | [09:06] Diego |
| RFC-PRP-02 | docs/RFC.md | Decisão | Criação de índices em status e created_at na tabela webhook_outbox e leitura em batch pequeno | TRANSCRICAO | [09:08] Diego |
| RFC-PRP-03 | docs/RFC.md | Decisão | Criação de novo entrypoint src/worker.ts e script npm run worker para o processo separado | TRANSCRICAO | [09:11] Larissa |
| RFC-PRP-04 | docs/RFC.md | Decisão | Política de retry com 5 tentativas e progressão de backoff 1m, 5m, 30m, 2h e 12h | TRANSCRICAO | [09:17] Larissa |
| RFC-PRP-05 | docs/RFC.md | Decisão | Isolamento de falhas permanentes em tabela separada webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| RFC-PRP-06 | docs/RFC.md | Decisão | Assinatura HMAC-SHA256 sobre o corpo da requisição com secret única por endpoint e grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Descarte de disparo HTTP síncrono no service de orders por travar outras mudanças e impossibilitar rollback limpo | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Descarte de Redis Streams / Redis Cluster por exigir subir mais infraestrutura (overengineering para time pequeno) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Descarte de trigger no MySQL por ausência de listener nativo tipo NOTIFY/LISTEN para processos externos | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Descarte de rodar o worker na mesma instância da API para não perder o worker quando a API reiniciar | TRANSCRICAO | [09:11] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Descarte de retry indefinido (eventos pendurados eternamente) e de apenas 3 tentativas (curto para manutenção de 2h) | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Descarte de marcar apenas como failed na própria outbox em favor de tabela webhook_dead_letter separada | TRANSCRICAO | [09:18] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa Descartada | Descarte de secret global da plataforma (se vazar uma, vaza tudo) em favor de secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-08 | docs/RFC.md | Alternativa Descartada | Descarte de garantia exactly-once por exigir coordenação complexa dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-OPN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio para cliente (ex.: 50 chamadas em 1 minuto) deixado em aberto para observar em produção | TRANSCRICAO | [09:38] Diego |
| RFC-OPN-02 | docs/RFC.md | Questão em Aberto | Particionamento por order_id ou lock pessimista para escalar múltiplos workers deixado para o futuro | TRANSCRICAO | [09:13] Diego |
| RFC-OPN-03 | docs/RFC.md | Questão em Aberto | Permissão de qualquer role autenticada no CRUD inicial de webhooks, deixando endurecimento para o futuro | TRANSCRICAO | [09:37] Sofia |
| FDD-MOD-01 | docs/FDD.md | Decisão | Uso de UUID como chave primária na modelagem da outbox e demais tabelas, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-FLX-01 | docs/FDD.md | Decisão | Integração no OrderService via função pura publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx | TRANSCRICAO | [09:41] Bruno |
| FDD-FLX-02 | docs/FDD.md | Decisão | Instanciação de PrismaClient separado no processo do worker conectado à mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| FDD-CON-01 | docs/FDD.md | Contrato | Endpoint POST /api/v1/webhooks para criação de webhook com url, events, customerId e retorno da secret gerada | TRANSCRICAO | [09:31] Marcos |
| FDD-CON-02 | docs/FDD.md | Contrato | Endpoint GET /api/v1/webhooks para listar os webhooks configurados de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CON-03 | docs/FDD.md | Contrato | Endpoint PATCH /api/v1/webhooks/:id para edição de configuração do webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CON-04 | docs/FDD.md | Contrato | Endpoint DELETE /api/v1/webhooks/:id para remoção de webhook cadastrado | TRANSCRICAO | [09:33] Bruno |
| FDD-CON-05 | docs/FDD.md | Contrato | Endpoint POST /api/v1/webhooks/:id/rotate-secret para rotação de secret com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-CON-06 | docs/FDD.md | Contrato | Endpoint GET /api/v1/webhooks/:id/deliveries retornando as últimas 100 entregas com payload, response e durationMs | TRANSCRICAO | [09:34] Marcos |
| FDD-CON-07 | docs/FDD.md | Contrato | Endpoint POST /api/v1/admin/webhooks/dead-letter/:id/replay exigindo role ADMIN para reenfileirar na outbox | TRANSCRICAO | [09:35] Diego |
| FDD-CON-08 | docs/FDD.md | Contrato | Formato enxuto do payload outbound (event_id, event_type, timestamp, order_id, order_number, status, total_cents, sem items) | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Matriz de Erros | Códigos de erro de domínio WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL e WEBHOOK_SECRET_REQUIRED estendendo AppError | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Matriz de Erros | Regra de erro sem truncamento quando o payload ultrapassar o teto de 64KB (WEBHOOK_PAYLOAD_TOO_LARGE) | TRANSCRICAO | [09:23] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão da transação prisma.$transaction no método OrderService.changeStatus para inserir na webhook_outbox | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados de pedidos e enum OrderStatus usados na validação dos filtros de eventos | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Adição dos novos models e relações com Customer e Order mantendo padrão @id @default(uuid()) @db.Char(36) | CODIGO | prisma/schema.prisma |
| FDD-INT-04 | docs/FDD.md | Integração | Classe base AppError reutilizada para estruturar statusCode, errorCode e details dos erros WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Classes de erro HTTP e de domínio existentes (NotFoundError, ConflictError, UnprocessableEntityError) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Tratamento centralizado de instâncias AppError, ZodError e erros Prisma sem necessidade de alterar o middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Reuso dos middlewares authenticate e requireRole('ADMIN') e estrutura JwtPayload (sub, email, role) | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Reuso do middleware validate({ body, query, params }) baseado em schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Registro de WebhookRepository, WebhookService e WebhookController na função buildControllers | CODIGO | src/app.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Montagem das rotas do módulo sob o prefixo /api/v1 na função buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Padrão de bootstrap e graceful shutdown (SIGINT/SIGTERM com prisma.$disconnect) usado como base para src/worker.ts | CODIGO | src/server.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Fábrica createPrismaClient() reutilizada para instanciar o cliente Prisma no processo separado do worker | CODIGO | src/config/database.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Logger estruturado Pino com redactPaths reutilizado na API, no worker e na auditoria de replay da DLQ | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Propagação e geração de X-Request-Id para correlação de requisições e logs HTTP | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Helper paginated(data, page, pageSize, total) reutilizado na listagem paginada de webhooks | CODIGO | src/shared/http/response.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Adição do script de inicialização do worker e uso de dependências nativas/existentes (Node >=20, uuid, zod, pino) | CODIGO | package.json |
| ADR-001-DEC | docs/adrs/ADR-001-padrao-transactional-outbox-no-mysql.md | Decisão | Persistência atômica de eventos na tabela webhook_outbox no MySQL dentro da mesma transação de changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-TRD | docs/adrs/ADR-001-padrao-transactional-outbox-no-mysql.md | Trade-off | Descarte de filtrar eventos no envio e de renderizar payload no envio em favor de filtro e snapshot na inserção | TRANSCRICAO | [09:52] Diego |
| ADR-002-DEC | docs/adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md | Decisão | Política de 5 tentativas com backoff 1m/5m/30m/2h/12h, timeout de 10s, tabela webhook_dead_letter e replay ADMIN | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DEC | docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Autenticação HMAC-SHA256 no header X-Signature com secret única por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:20] Sofia |
| ADR-004-DEC | docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com UUID no header X-Event-Id para deduplicação pelo cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-DEC | docs/adrs/ADR-005-worker-em-processo-separado-com-polling.md | Decisão | Worker executando em processo separado (src/worker.ts) realizando polling no MySQL a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Decisão | Reuso integral de AppError (WEBHOOK_*), Pino, errorMiddleware, Zod, requireRole e estrutura src/modules/webhooks | TRANSCRICAO | [09:30] Larissa |
| ADR-006-MOD | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Decisão | Organização do módulo em src/modules/webhooks com controller, service, repository, routes, schemas e worker | TRANSCRICAO | [09:27] Bruno |

---

## 2. Itens Sinalizados como Pendentes de Definição (Não Inventados)

Durante a revisão de rastreabilidade, os seguintes detalhes de baixo nível presentes no `docs/FDD.md` foram verificados como **não fixados literalmente** na transcrição (embora o requisito/comportamento macro tenha sido decidido) e, portanto, estão explicitamente sinalizados no `docs/FDD.md` como pendentes de definição final na implementação/revisão:

1. **Sufixo literal da rota de rotação de secret (`POST /api/v1/webhooks/:id/rotate-secret`):** A existência do endpoint de rotação com validade paralela de 24h foi decidida em `[09:21] Sofia`, mas o path literal da URL não foi verbalizado na call (diferentemente de `GET /webhooks/:id/deliveries` em `[09:34] Marcos` e `POST /admin/webhooks/dead-letter/:id/replay` em `[09:18] Diego`).
2. **Formato exato do header `X-Signature` durante a janela de 24h do Grace Period:** A validade paralela da secret antiga por 24h foi decidida em `[09:21] Sofia`, e o detalhamento criptográfico passará pela revisão de 2 dias úteis solicitada em `[09:46] Sofia`.
3. **Valor numérico exato do lote de leitura do Worker (`batch pequeno`):** Em `[09:08] Diego` definiu-se leitura em *"batch pequeno"*, ficando a constante numérica exata (ex.: 20 ou 50) parametrizável na implementação.
4. **Nomes literais dos códigos de erro derivados do prefixo `WEBHOOK_*`:** Em `[09:28] Bruno` foram nomeados literalmente `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED`, e em `[09:29] Larissa` foi estabelecida a regra geral *"Prefixo WEBHOOK_ pra tudo do módulo"*. Os códigos `WEBHOOK_PAYLOAD_TOO_LARGE` (para a regra de 64KB de `[09:24] Larissa`) e `WEBHOOK_DEAD_LETTER_NOT_FOUND` / `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` (para o replay de `[09:18] Diego`) derivam diretamente dessa regra de prefixo.
5. **Modelagem física de tabelas/colunas auxiliares (Sprint 1 — `[09:46] Larissa`):** Enquanto `webhook_outbox` (`[09:06] Diego`) e `webhook_dead_letter` (`[09:18] Diego`) tiveram nomes literais definidos na call, os nomes físicos das tabelas de configuração (`webhooks`) e histórico (`webhook_deliveries`) e de colunas internas de controle (`nextAttemptAt`, `previousSecretExpiresAt`, `replayedById`) são propostas de modelagem para a Sprint 1.
6. **Campos adicionais opcionais no payload ("campos básicos da order") e nomes de métricas:** Em `[09:43] Diego` foi citado nominalmente `total_cents` como exemplo de campo básico do pedido; a inclusão de outros escalares de `Order` (`subtotalCents`, `discountCents`) e os identificadores de métricas de observabilidade no `docs/FDD.md` estão sinalizados como pendentes de definição.

---

## 3. Métricas e Estatísticas de Rastreabilidade

| Indicador | Resultado Obtido | Meta Mínima do Desafio | Status |
| :--- | :---: | :---: | :---: |
| **Total de itens identificáveis nos documentos (`PRD`, `RFC`, `FDD`, `ADRs`)** | **90 itens** | — | — |
| **Total de itens rastreados na tabela (`docs/TRACKER.md`)** | **90 itens** | >= 80% dos itens | **100,0% (Atendido)** |
| **Linhas com `Fonte = TRANSCRICAO` (`[hh:mm] Nome` válido)** | **73 de 90 linhas (81,11%)** | >= 70% das linhas | **81,11% (Atendido)** |
| **Linhas com `Fonte = CODIGO` (caminhos reais do repositório)** | **17 de 90 linhas (18,89% — 17 arquivos distintos)** | >= 5 linhas | **17 linhas (Atendido)** |
| **Itens sem fonte comprovável / alucinações** | **0** | 0 | **100% Íntegro** |
