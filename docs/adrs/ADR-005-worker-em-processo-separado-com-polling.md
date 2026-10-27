# ADR-005: Worker em Processo Separado Utilizando Polling no MySQL

- **Status:** Aceito
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (Product Manager)

---

## Contexto

Com a adoção do padrão Transactional Outbox no MySQL ([ADR-001](./ADR-001-padrao-transactional-outbox-no-mysql.md)), os eventos de mudança de status de pedidos são gravados de forma síncrona na tabela `webhook_outbox`, mas o disparo efetivo das chamadas HTTP `POST` para os endpoints dos clientes deve ocorrer de forma assíncrona (`[09:06] Diego`).

O requisito de negócio levantado pelos clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) define que "tempo real" significa receber a notificação em **menos de 10 segundos** após a mudança de status (`[09:01] Bruno`, `[09:02] Marcos`). Precisamos decidir:
1. Qual mecanismo o consumidor (worker) utilizará para detectar novos eventos na tabela `webhook_outbox` do MySQL (`[09:08] Larissa`).
2. Como esse worker será executado em relação ao processo da API HTTP Express (`src/server.ts`) e como gerenciará sua conexão com o banco de dados (`[09:11] Diego`, `[09:29] Diego`).

---

## Decisão

Implementar o consumidor da outbox como um **worker em processo Node.js separado** (`src/worker.ts`), executando **polling em loop a cada 2 segundos** sobre o banco de dados MySQL:

1. **Mecanismo de Leitura via Polling (Intervalo de 2 Segundos):** O worker consultará a tabela `webhook_outbox` em loop a cada **2 segundos**, buscando os eventos pendentes mais antigos (ordenados por `created_at` e filtrados pelo índice de `status` e agendamento de retry) em pequenos lotes (*small batches*), processando os envios HTTP e atualizando o status correspondente (`[09:08] Diego`, `[09:09] Diego`, `[09:10] Larissa`).
2. **Execução em Processo Separado (`src/worker.ts`):** O worker não rodará dentro da mesma instância/processo da API HTTP. Será criado um novo *entrypoint* em `src/worker.ts` acionado por um script dedicado `"worker"` no `package.json` (`npm run worker`), espelhando a estrutura de inicialização e *graceful shutdown* existente em `src/server.ts`, enquanto a lógica de processamento residirá dentro do módulo de webhooks (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`) (`[09:11] Diego`, `[09:11] Larissa`, `[09:28] Bruno`).
3. **Instância Própria de `PrismaClient`:** Por executar em outro processo do sistema operacional, o worker instanciará seu próprio `PrismaClient` (reutilizando a fábrica `createPrismaClient()` de `src/config/database.ts` e a mesma variável `DATABASE_URL` de `src/config/env.ts`) conectado ao mesmo banco MySQL (`[09:11] Bruno`, `[09:11] Diego`, `[09:29] Diego`, `[09:30] Bruno`).
4. **Topologia Inicial Single-Worker:** Na fase atual, haverá uma única instância do processo worker em execução (*single-worker*), garantindo processamento sequencial por `created_at` (`[09:12] Diego`, `[09:13] Larissa`).

> **Nota sobre Escopo (Em Aberto / Adiado):**
> - **Escalonamento Horizontal de Workers:** A execução de múltiplos workers em paralelo com particionamento por `order_id` ou lock pessimista foi adiada para o futuro caso a demanda exija (`[09:12] Diego`, `[09:13] Diego`).
> - **Rate Limiting de Saída:** O controle de taxa de envio por cliente (ex.: quando 50 pedidos de um mesmo cliente mudam de status em 1 minuto) não será implementado agora; foi registrado como questão em aberto para observar em produção e decidir depois (`[09:38] Diego`, `[09:39] Diego`, `[09:39] Larissa`).

---

## Alternativas Consideradas

### 1. Executar o Worker Dentro do Mesmo Processo da API (`src/server.ts`)
- **Descrição:** Iniciar o loop de polling (`setInterval`) dentro do próprio processo da API Express em `src/server.ts` (`[09:11] Diego`).
- **Motivo do Descarte:** Acopla o ciclo de vida do processador de background ao servidor HTTP: se a API reiniciar, o worker em andamento é interrompido abruptamente (`[09:11] Diego`). Além disso, isolar em processo separado permite escalar ou reiniciar a API e o worker de forma independente e evita competição direta no event loop durante picos de requisições HTTP.

### 2. Utilizar Triggers do MySQL para Acionar o Worker Reativamente
- **Descrição:** Usar uma trigger no banco de dados MySQL na inserção da `webhook_outbox` para notificar o worker imediatamente sem polling (`[09:09] Bruno`).
- **Motivo do Descarte:** Ao contrário do PostgreSQL (que possui `NOTIFY/LISTEN` nativo), o MySQL 8.0 não dispõe de mecanismo nativo para notificar processos externos a partir de triggers — uma trigger no MySQL apenas executa instruções SQL internas. Para notificar o worker a partir de uma trigger no MySQL, seria necessário improvisar soluções frágeis e anti-padrão (como escrever em arquivo local ou acionar um endpoint via extensão externa). O polling a cada 2 segundos é simples, nativo e atende com folga o SLA de < 10 segundos (`[09:09] Diego`, `[09:10] Marcos`, `[09:10] Larissa`).

---

## Consequências

### Positivas
- **Atendimento Folgado ao SLA de Latência (< 10s):** O intervalo de polling de 2 segundos garante que novos eventos comecem a ser processados em no máximo 2 segundos em condições normais, muito abaixo do limite de 10 segundos acordado com os clientes (`[09:02] Marcos`, `[09:09] Diego`, `[09:10] Larissa`).
- **Isolamento de Falhas e Deploys:** Reinicializações ou picos de carga na API (`src/server.ts`) não interrompem o worker (`src/worker.ts`), e lentidões em chamadas HTTP externas no worker não afetam a responsividade da API (`[09:11] Diego`).
- **Mesma Stack e Zero Infraestrutura Nova:** O worker compartilha a mesma base de código TypeScript, schemas, logger Pino e configuração Prisma (`src/config/database.ts`, `src/config/env.ts`) (`[09:11] Bruno`, `[09:11] Diego`, `[09:30] Bruno`).

### Negativas
- **Latência Base de Polling (Até 2s):** Mesmo com o sistema ocioso, um evento pode aguardar até 2 segundos entre o commit da transação e o ciclo de leitura do worker (`[09:10] Larissa`).
- **Queries Periódicas Constantes no MySQL:** O worker executa uma consulta `SELECT` na `webhook_outbox` a cada 2 segundos (30 queries/minuto), mitigada pelo índice composto em `(status, created_at)` (`[09:08] Diego`, `[09:09] Diego`).
- **Gerenciamento de Dois Processos em Produção:** O ambiente de execução passa a monitorar dois processos Node.js (`npm start` para a API e `npm run worker` para o worker) e duas conexões/pools de `PrismaClient` no MySQL (`[09:11] Larissa`, `[09:30] Bruno`).

### Trade-offs
- **Simplicidade do Polling de 2s vs. Reatividade Instantânea (Sub-segundo):** Aceita-se uma latência mínima de até 2 segundos e uma leve carga constante de leitura indexada no MySQL em troca de evitar gambiarras com triggers no MySQL ou a introdução de um message broker externo (`[09:07] Diego`, `[09:09] Diego`, `[09:10] Larissa`).

---

## Referências

### Transcrição (`TRANSCRICAO.md`)
- `[09:01] Bruno` / `[09:02] Marcos` — Definição de "tempo real" pelos clientes como qualquer atraso abaixo de 10 segundos.
- `[09:08] Diego` / `[09:08] Larissa` / `[09:09] Diego` — Índices em `status` e `created_at` e proposta de polling em loop a cada 2 segundos.
- `[09:09] Bruno` / `[09:09] Diego` / `[09:10] Marcos` / `[09:10] Larissa` — Descarte de trigger no MySQL (ausência de `NOTIFY/LISTEN` nativo) e confirmação da decisão de polling de 2 segundos.
- `[09:11] Diego` / `[09:11] Larissa` / `[09:11] Bruno` — Decisão de rodar o worker em processo separado (`src/worker.ts` + script `"npm run worker"`) conectando no mesmo banco via Prisma.
- `[09:12] Diego` / `[09:13] Larissa` — Operação inicial como *single-worker*.
- `[09:28] Diego` / `[09:28] Bruno` — Organização de `src/worker.ts` como entrypoint e `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) para a lógica.
- `[09:29] Diego` / `[09:30] Bruno` — Instância separada de `PrismaClient` para o processo do worker usando a mesma `DATABASE_URL`.
- `[09:38] Diego` / `[09:39] Larissa` — Rate limiting de saída deixado em aberto para observação futura.

### Código-Fonte do Repositório
- `src/server.ts` (linhas 1–27) — Entrypoint atual da API HTTP com tratamento de sinais `SIGINT` e `SIGTERM` e desconexão limpa do Prisma (`await prisma.$disconnect()`), modelo para `src/worker.ts`.
- `src/config/database.ts` (linhas 4–10) — Função `createPrismaClient()` que permite instanciar o cliente Prisma no processo do worker.
- `src/config/env.ts` (linhas 3–10) — Validação de `DATABASE_URL`, `NODE_ENV` e `LOG_LEVEL` via Zod.
- `package.json` (linhas 10–21) — Seção `"scripts"` onde os comandos do worker (`dev:worker` / `worker`) serão adicionados ao lado de `dev`, `build` e `start`.
