# ADR-006: Reutilização dos Padrões Arquiteturais e Convenções Existentes no Projeto

- **Status:** Aceito
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno — Pedidos), Diego (Engenheiro Sênior — Plataforma), Sofia (Engenheira de Segurança), Marcos (Product Manager)

---

## Contexto

O repositório do OMS já possui uma arquitetura em camadas bem estabelecida, testada e consistente em todos os seus domínios (`auth`, `users`, `customers`, `products` e `orders`). A introdução do Sistema de Webhooks envolve novos endpoints REST (CRUD de webhooks, rotação de secret, listagem de histórico de entregas e replay administrativo de DLQ), validações de entrada, tratamento de erros de domínio, autenticação/autorização, logging estruturado e acesso a dados via Prisma (`[09:27] Larissa`, `[09:27] Bruno`).

Surgiu a necessidade de definir como estruturar o novo módulo de webhooks e suas integrações transversais (erros, logs, validação, autenticação e injeção de dependências) para evitar fragmentação arquitetural, bibliotecas duplicadas ou contratos de erro inconsistentes na API (`[09:27] Bruno`, `[09:28] Bruno`, `[09:29] Bruno`, `[09:30] Larissa`).

---

## Decisão

Reutilizar integralmente os **padrões arquiteturais, middlewares, utilitários e convenções de código existentes no repositório**, estruturando a feature como um módulo padrão em `src/modules/webhooks/`:

1. **Estrutura Modular Padronizada (`src/modules/webhooks/`):**
   Seguindo a mesma organização de `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/` e `src/modules/users/`, o módulo de webhooks será dividido em (`[09:27] Bruno`, `[09:28] Bruno`):
   - `webhook.routes.ts` — Definição de rotas Express com middlewares `authenticate`, `requireRole` e `validate`.
   - `webhook.controller.ts` — Handlers HTTP repassando erros para `next(err)`.
   - `webhook.service.ts` — Regras de negócio de cadastro, geração/rotação de secret, filtro de status, consulta de deliveries e replay de DLQ.
   - `webhook.repository.ts` — Encapsulamento das consultas Prisma (`PrismaClient`).
   - `webhook.schemas.ts` — Schemas Zod para validação de `body`, `params` e `query` (incluindo validação de URL `https://`).
   - `webhook.worker.ts` (ou `webhook.processor.ts`) — Lógica de processamento da outbox invocada pelo entrypoint `src/worker.ts`.
   - Registro dos controllers em `buildControllers` (`src/app.ts`) e montagem das rotas em `buildApiRouter` (`src/routes/index.ts`).

2. **Hierarquia de Erros e Prefixo `WEBHOOK_*`:**
   Reutilizar a classe base `AppError` (`src/shared/errors/app-error.ts`), as classes de erro HTTP (`src/shared/errors/http-errors.ts`) e o middleware global `errorMiddleware` (`src/middlewares/error.middleware.ts`). Todos os códigos de erro específicos do domínio de webhooks utilizarão obrigatoriamente o prefixo **`WEBHOOK_`** (como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, entre outros), seguindo o padrão de `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` (`[09:28] Bruno`, `[09:29] Larissa`, `[09:29] Bruno`).

3. **Logging Estruturado com Pino:**
   Utilizar a instância existente do logger **Pino** em `src/shared/logger/index.ts` e o middleware `requestLogger` em `src/middlewares/request-logger.middleware.ts` tanto na API quanto no worker e na auditoria de replay da DLQ, sem adicionar nenhuma nova biblioteca de log (`[09:29] Bruno`, `[09:36] Sofia`).

4. **Autenticação JWT, RBAC e Passagem Explícita de `customerId`:**
   - Reutilizar o middleware `authenticate` (`src/middlewares/auth.middleware.ts`) em todas as rotas de webhooks (`[09:32] Larissa`, `[09:36] Marcos`, `[09:37] Sofia`).
   - Como o payload do JWT atual (`JwtPayload` em `src/middlewares/auth.middleware.ts`, linhas 19–25) representa o usuário operador/admin (`sub`, `email`, `role: 'ADMIN' | 'OPERATOR'`) e não possui `customerId`, o identificador do cliente (`customerId`) será informado explicitamente no `body` ou `query`/`path` das requisições de configuração de webhook (`[09:31] Marcos`, `[09:32] Bruno`, `[09:32] Marcos`, `[09:32] Larissa`).
   - Reutilizar o middleware `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`, linhas 49–61) para restringir o endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` exclusivamente a administradores (`[09:35] Larissa`, `[09:36] Sofia`, `[09:36] Larissa`).

5. **Desacoplamento na Integração com `OrderService`:**
   Para não acoplar a classe `OrderService` (`src/modules/orders/order.service.ts`) à instância completa de `WebhookRepository`, a integração dentro da transação de `changeStatus` será feita por meio de uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` (`Prisma.TransactionClient`) (`[09:41] Bruno`, `[09:41] Diego`).

> **Nota sobre Escopo (Questão em Aberto / Futuro):** O endurecimento futuro das permissões de role no CRUD de configuração de webhooks (atualmente aberto a qualquer role autenticada, `ADMIN` ou `OPERATOR`) foi deixado para evolução posterior (`[09:36] Marcos`, `[09:37] Sofia`).

---

## Alternativas Consideradas

### 1. Extrair `customer_id` Implícito Diretamente do Token JWT Atual
- **Descrição:** Assumir que o `customer_id` viria embutido nas claims do token JWT do usuário autenticado (`[09:31] Marcos`).
- **Motivo do Descarte:** Na implementação real do sistema (`src/modules/auth/auth.service.ts` e `src/middlewares/auth.middleware.ts`), o token JWT pertence à entidade `User` (`ADMIN` ou `OPERATOR`) e contém apenas `{ sub, email, role }`, sem vínculo direto de schema entre `User` e `Customer` em `prisma/schema.prisma`. Alterar a estrutura de autenticação quebraria o modelo atual; portanto, decidiu-se receber `customerId` explicitamente no `body` ou `path`/`query` (`[09:32] Bruno`, `[09:32] Larissa`).

### 2. Injetar `WebhookRepository` Completo no Construtor de `OrderService`
- **Descrição:** Passar a classe `WebhookRepository` inteira como dependência no construtor de `OrderService` em `src/app.ts` (`[09:41] Bruno`).
- **Motivo do Descarte:** Aumentaria o acoplamento entre os módulos `orders` e `webhooks` e exigiria lidar com estado de repositório dentro de uma transação interativa (`tx`). O uso de uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo apenas o `tx` mantém a integração limpa e testável (`[09:41] Bruno`, `[09:41] Diego`).

### 3. Introduzir Novas Bibliotecas de Logging, Validação ou Tratamento de Erros
- **Descrição:** Adotar ferramentas externas adicionais para validação, logs ou formatação de erros no módulo de webhooks (`[09:28] Bruno`, `[09:29] Bruno`).
- **Motivo do Descarte:** O projeto já conta com `AppError`, `errorMiddleware`, `validate` (Zod) e `logger` (Pino) cobrindo todos os cenários necessários. Introduzir padrões paralelos geraria inconsistência nas respostas HTTP e nos logs da aplicação (`[09:29] Bruno`, `[09:30] Larissa`).

---

## Consequências

### Positivas
- **Consistência de Contratos na API:** Todas as respostas de erro do módulo de webhooks seguem automaticamente o envelope `{ error: { code, message, details } }` processado por `src/middlewares/error.middleware.ts`, com códigos claros sob o namespace `WEBHOOK_*` (`[09:28] Bruno`, `[09:29] Larissa`).
- **Velocidade de Desenvolvimento e Manutenibilidade:** Desenvolvedores familiarizados com `orders`, `customers` ou `products` navegam imediatamente por `src/modules/webhooks/` sem curva de aprendizado (`[09:27] Bruno`, `[09:30] Larissa`).
- **Compatibilidade com Testes Existentes:** A injeção via `buildApp({ prisma })` em `src/app.ts` permite testar o módulo de webhooks com `supertest` e `vitest` exatamente como em `tests/orders.test.ts` e `tests/auth.test.ts`.

### Negativas
- **Permissão Ampla Inicial no CRUD de Webhooks:** Como qualquer usuário autenticado (`OPERATOR` ou `ADMIN`) pode informar um `customerId` nos endpoints de configuração nesta primeira fase, não há isolamento multi-tenant no nível do token JWT até que o modelo de permissões seja endurecido no futuro (`[09:32] Larissa`, `[09:36] Marcos`, `[09:37] Sofia`).

### Trade-offs
- **Aderência Estrita ao Modelo Atual (Reuso Total) vs. Refatoração de Identidade/Auth:** Optou-se por reutilizar 100% da infraestrutura de autenticação (`authenticate`/`requireRole`), erros (`AppError`) e camadas existentes — passando `customerId` explicitamente na requisição — em vez de redesenhar o sistema de usuários/JWT da aplicação, garantindo entrega segura dentro do prazo de 3 sprints (`[09:30] Larissa`, `[09:32] Larissa`, `[09:37] Sofia`, `[09:46] Larissa`).

---

## Referências

### Transcrição (`TRANSCRICAO.md`)
- `[09:27] Bruno` / `[09:28] Diego` / `[09:28] Bruno` — Estrutura do módulo em `src/modules/webhooks` (`controller`, `service`, `repository`, `routes`, `schemas`) e `src/worker.ts` + `webhook.worker.ts` / `webhook.processor.ts`.
- `[09:28] Bruno` / `[09:29] Larissa` — Reuso de `AppError` e padronização de códigos de erro com prefixo `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
- `[09:29] Bruno` / `[09:30] Larissa` — Reuso de Pino, `errorMiddleware`, schemas Zod e confirmação da decisão de reuso máximo dos padrões existentes.
- `[09:31] Marcos` / `[09:32] Bruno` / `[09:32] Marcos` / `[09:32] Larissa` — Esclarecimento sobre o JWT atual ser de usuário operador e decisão de passar `customer_id` no `body` ou `path`.
- `[09:35] Larissa` / `[09:36] Sofia` / `[09:36] Larissa` / `[09:37] Sofia` — Uso de `requireRole('ADMIN')` no replay da DLQ e qualquer role autenticada no CRUD inicial.
- `[09:41] Bruno` / `[09:41] Diego` — Integração com `OrderService` via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`.

### Código-Fonte do Repositório
- `src/app.ts` (linhas 26–75) — Composição dos módulos em `buildControllers` e montagem do Express em `buildApp`.
- `src/routes/index.ts` (linhas 13–31) — Registro centralizado de rotas sob `/api/v1` em `buildApiRouter`.
- `src/shared/errors/app-error.ts` (linhas 1–16) e `src/shared/errors/http-errors.ts` (linhas 1–63) — Classes `AppError`, `InvalidStatusTransitionError`, `InsufficientStockError`, etc.
- `src/middlewares/error.middleware.ts` (linhas 14–65) — Tratamento centralizado de `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError`.
- `src/middlewares/auth.middleware.ts` (linhas 6–61) — Tipos `AuthUser`, `JwtPayload` e middlewares `authenticate` e `requireRole`.
- `src/middlewares/validate.middleware.ts` (linhas 11–37) — Middleware `validate({ body, query, params })` baseado em Zod.
- `src/shared/logger/index.ts` (linhas 1–32) — Logger estruturado Pino.
- `src/shared/http/response.ts` (linhas 1–24) — Utilitário de resposta paginada `paginated()`.
- `src/modules/orders/order.service.ts` (linhas 126–179) — Método `OrderService.changeStatus`.
