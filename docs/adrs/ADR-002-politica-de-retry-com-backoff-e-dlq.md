# ADR-002: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ)

- **Status:** Aceito
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

Na entrega de webhooks *outbound* para sistemas externos de clientes B2B, falhas transitórias são inevitáveis devido a instabilidades de rede, lentidão dos servidores de destino ou janelas de manutenção planejada nos ambientes dos clientes — historicamente, clientes da plataforma já apresentaram janelas de indisponibilidade de até 2 horas em manutenções programadas (`[09:14] Larissa`, `[09:16] Diego`).

Ao mesmo tempo, o sistema não pode manter tentativas de reenvio infinitamente caso o endpoint de um cliente seja desativado ou fique permanentemente indisponível, nem pode poluir a leitura de eventos ativos na tabela `webhook_outbox` com eventos mortos (`[09:15] Diego`, `[09:18] Diego`). Além disso, a equipe de operação precisa de visibilidade sobre as entregas e de um mecanismo seguro e auditável para reprocessar eventos que esgotaram as tentativas após o cliente restabelecer seu serviço (`[09:18] Bruno`, `[09:18] Diego`, `[09:34] Marcos`, `[09:36] Sofia`).

---

## Decisão

Estabelecer uma política determinística de **5 tentativas com backoff exponencial** (~15 horas de janela total), timeout de **10 segundos** por requisição HTTP e isolamento de falhas permanentes em uma tabela dedicada de **Dead Letter Queue (`webhook_dead_letter`)** com endpoint administrativo de replay:

1. **Timeout por Chamada HTTP:** Cada tentativa de envio realizada pelo worker terá timeout máximo de **10 segundos**. Caso o servidor do cliente não responda dentro de 10 segundos (ou retorne erro HTTP/rede), a entrega é considerada falha e agendada para retry (`[09:42] Sofia`, `[09:42] Diego`).
2. **Limite de Tentativas e Progressão do Backoff:** O sistema realizará no máximo **5 tentativas** de retry após a falha, seguindo os intervalos progressivos de:
   - 1ª retentativa: **1 minuto**
   - 2ª retentativa: **5 minutos**
   - 3ª retentativa: **30 minutos**
   - 4ª retentativa: **2 horas**
   - 5ª retentativa: **12 horas**
   - **Janela total acumulada:** 14 horas e 36 minutos (~15 horas entre a primeira falha e a última tentativa), cobrindo com folga manutenções planejadas de clientes (`[09:15] Diego`, `[09:16] Larissa`, `[09:17] Diego`, `[09:17] Marcos`, `[09:17] Larissa`).
3. **Tabela Separada de Dead Letter Queue (`webhook_dead_letter`):** Após esgotar as 5 tentativas sem sucesso, o evento é considerado falha permanente e movido para uma tabela separada `webhook_dead_letter` contendo o payload, o motivo da falha e o timestamp (`[09:15] Diego`, `[09:18] Diego`).
4. **Replay Manual Restrito a Administradores:** Eventos na DLQ poderão ser reprocessados manualmente por meio do endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na `webhook_outbox` com status pendente (`[09:18] Diego`, `[09:19] Larissa`, `[09:35] Diego`). Este endpoint exige obrigatoriamente autenticação JWT e autorização `requireRole('ADMIN')` (reutilizando `src/middlewares/auth.middleware.ts`), além de registrar em log estruturado (Pino) o usuário administrador responsável pela ação para fins de auditoria (`[09:35] Larissa`, `[09:36] Sofia`, `[09:36] Larissa`).
5. **Histórico de Entregas para o Cliente:** Todas as tentativas de entrega são registradas para permitir que o cliente consulte as últimas 100 entregas via `GET /api/v1/webhooks/:id/deliveries`, inspecionando sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos`, `[09:35] Larissa`).

> **Nota sobre Escopo (Fora de Escopo / Adiado):** O envio de alertas por e-mail ao cliente quando um webhook falha repetidamente (ex.: após 3 falhas consecutivas) foi discutido e explicitamente **descartado para esta fase** (adiado para uma fase futura) (`[09:37] Marcos`, `[09:37] Larissa`, `[09:48] Larissa`).

---

## Alternativas Consideradas

### 1. Retry Indefinido com Backoff
- **Descrição:** Continuar retentando o envio do evento indefinidamente até obter sucesso (`[09:15] Diego`).
- **Motivo do Descarte:** Se a URL do cliente deixar de existir ou ficar quebrada permanentemente, os eventos ficariam acumulados e pendurados para sempre no banco de dados, consumindo recursos de processamento indefinidamente (`[09:15] Diego`).

### 2. Política Mais Agressiva de Apenas 3 Tentativas
- **Descrição:** Limitar a política de retry a 3 tentativas rápidas (ex.: encerrando o ciclo em ~30 minutos) (`[09:16] Bruno`).
- **Motivo do Descarte:** Janela curta demais. Como a empresa já teve clientes em produção com indisponibilidade de 2 horas devido a manutenções planejadas, 3 tentativas esgotariam os retries antes do término da manutenção do cliente, enviando eventos prematuramente para a DLQ (`[09:16] Diego`).

### 3. Manter Eventos Falhos na Própria Tabela `webhook_outbox` (Status `failed`) sem Tabela DLQ Separada
- **Descrição:** Não criar a tabela `webhook_dead_letter` e apenas marcar os eventos esgotados com status `failed` na própria tabela `webhook_outbox` (`[09:17] Larissa`).
- **Motivo do Descarte:** Degradaria a limpeza e a eficiência de leitura da tabela `webhook_outbox` principal pelo worker. Isolar na tabela `webhook_dead_letter` mantém a outbox enxuta e organiza as evidências de falha permanente (payload, motivo da falha e timestamp) para diagnóstico e replay (`[09:18] Diego`, `[09:18] Bruno`).

---

## Consequências

### Positivas
- **Resiliência a Manutenções e Falhas Transitórias:** A progressão `1m -> 5m -> 30m -> 2h -> 12h` absorve tanto oscilações rápidas de rede (primeiros 6 minutos) quanto manutenções prolongadas de várias horas (`[09:16] Diego`, `[09:17] Diego`).
- **Proteção da Tabela `webhook_outbox`:** Eventos permanentemente falhos são removidos do caminho crítico de polling do worker e isolados em `webhook_dead_letter` (`[09:18] Diego`).
- **Recuperação Operacional Segura e Auditável:** Administradores podem reenfileirar eventos da DLQ sem intervenção manual via banco de dados (SQL direto), contando com controle de acesso RBAC (`ADMIN`) e trilha de auditoria em log (`[09:18] Diego`, `[09:36] Sofia`, `[09:36] Larissa`).
- **Previsibilidade de Recursos no Worker:** O timeout de 10s evita que conexões HTTP abertas com clientes lentos bloqueiem o worker indefinidamente (`[09:42] Diego`).

### Negativas
- **Modelagem e Movimentação Adicional de Dados:** Requer manter uma tabela adicional (`webhook_dead_letter`) no `prisma/schema.prisma` e implementar a lógica transacional de transferência da outbox para a DLQ ao esgotar as 5 tentativas de retry (`[09:18] Diego`).
- **Operação Manual para Recuperação Pós-15h:** Caso o sistema de um cliente fique fora do ar por mais de 15 horas e não haja alerta automático por e-mail nesta fase, o reprocessamento dependerá do acionamento do endpoint de replay por um usuário `ADMIN` (`[09:17] Marcos`, `[09:18] Diego`, `[09:37] Larissa`).

### Trade-offs
- **Janela Longa de Retentativa (~15h) vs. Retenção Temporária na Outbox:** Priorizou-se minimizar a perda de notificações em manutenções de clientes B2B (até 15h de tolerância automática) em troca de manter eventos com falha transitória agendados na outbox durante esse período, transferindo apenas as falhas definitivas para a `webhook_dead_letter` (`[09:16] Diego`, `[09:17] Diego`, `[09:18] Diego`).

---

## Referências

### Transcrição (`TRANSCRICAO.md`)
- `[09:14] Larissa` / `[09:15] Diego` — Definição de backoff exponencial com teto de 5 tentativas e encaminhamento para DLQ.
- `[09:16] Bruno` / `[09:16] Diego` — Discussão e descarte de 3 tentativas devido ao histórico de manutenções planejadas de 2 horas de clientes.
- `[09:17] Diego` / `[09:17] Marcos` / `[09:17] Larissa` — Aprovação da progressão `1m/5m/30m/2h/12h` (~15 horas).
- `[09:17] Larissa` / `[09:18] Diego` / `[09:18] Bruno` — Decisão por tabela separada `webhook_dead_letter` com payload, motivo da falha e timestamp.
- `[09:18] Diego` / `[09:19] Larissa` / `[09:35] Diego` — Definição do endpoint `POST /admin/webhooks/dead-letter/:id/replay`.
- `[09:34] Marcos` — Histórico das últimas 100 entregas em `GET /webhooks/:id/deliveries`.
- `[09:35] Larissa` / `[09:36] Sofia` / `[09:36] Larissa` — Exigência da role `ADMIN` via `requireRole` e log de auditoria no replay da DLQ.
- `[09:37] Marcos` / `[09:37] Larissa` — Descarte de notificações por e-mail para esta fase.
- `[09:42] Sofia` / `[09:42] Diego` — Timeout de 10 segundos para a chamada HTTP do worker.

### Código-Fonte do Repositório
- `src/middlewares/auth.middleware.ts` (linhas 27–61) — Implementação existente de `authenticate` e `requireRole('ADMIN')` que protegerá o endpoint de replay da DLQ (como já utilizado em `src/modules/users/user.routes.ts`, linha 15).
- `src/shared/logger/index.ts` (linhas 13–32) — Logger Pino estruturado utilizado para registrar a auditoria de quem acionou o replay.
- `prisma/schema.prisma` (linhas 11–14) — Enum `UserRole` (`ADMIN`, `OPERATOR`).
