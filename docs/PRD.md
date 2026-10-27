# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| :--- | :--- |
| **Produto / Módulo** | Order Management System (OMS) — Notificações de Pedidos via Webhooks (*Outbound*) |
| **Product Manager** | Marcos (`[09:00] Marcos`) |
| **Tech Lead & Engenharia** | Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Sofia (Engenheira de Segurança) |
| **Status** | Alinhado em Reunião Técnica / Em Revisão Documental (`[09:48]–[09:50] Larissa`) |
| **Prazo Alvo** | 3 Sprints — Entrega até o fim de novembro / fim do trimestre (`[09:00] Marcos`, `[09:45]–[09:47] Marcos/Larissa`) |
| **Documentos Relacionados** | [RFC](./RFC.md) · [FDD](./FDD.md) · [ADR-001](./adrs/ADR-001-padrao-transactional-outbox-no-mysql.md) · [ADR-002](./adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md) · [ADR-003](./adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md) · [ADR-004](./adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) · [ADR-005](./adrs/ADR-005-worker-em-processo-separado-com-polling.md) · [ADR-006](./adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md) |

---

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) opera em produção gerenciando o ciclo de vida completo de pedidos B2B (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED`), incluindo controle transacional de estoque e histórico de auditoria (`prisma/schema.prisma`, `src/modules/orders/order.service.ts`). Contudo, a plataforma atualmente funciona de maneira exclusivamente passiva: não há qualquer mecanismo de notificação ativa quando o status de um pedido sofre alteração.

Esta feature introduz o **Sistema de Webhooks de Notificação de Pedidos (*Outbound*)**, permitindo que clientes B2B cadastrem endpoints HTTPS próprios para receber notificações automáticas em tempo próximo ao real (< 10 segundos) sempre que seus pedidos mudarem de status, com garantias de autenticidade criptográfica (`HMAC-SHA256`), retentativas automáticas em caso de indisponibilidade temporária e visibilidade sobre o histórico de entregas (`[09:00]–[09:03] Marcos/Sofia`, `[09:48] Larissa`).

---

## 2. Problema e Motivação

### 2.1 O Problema Atual
Na ausência de notificações *push* (webhooks), os clientes B2B integrados ao OMS precisam consultar repetidamente o endpoint `GET /api/v1/orders` (*polling* externo) para descobrir se algum pedido teve mudança de status (`[09:00] Marcos`, `src/modules/orders/order.routes.ts`). Esse modelo gera três dores diretas para os clientes e para o negócio:
1. **Lentidão operacional no cliente:** Os sistemas de ERP/logística dos clientes só tomam conhecimento das atualizações de pedidos no intervalo de suas rotinas manuais ou agendadas de consulta (`[09:00] Marcos`, `[09:02] Marcos`).
2. **Alto custo de integração:** Chamadas constantes e redundantes ao `GET /api/v1/orders` consomem recursos tanto dos clientes quanto da API do OMS, mesmo quando nenhuma mudança de status ocorreu (`[09:00] Marcos`).
3. **Risco iminente de *Churn* B2B:** Na semana anterior à reunião de definição, três clientes B2B (**Atlas Comercial**, **MaxDistribuição** e **Nova Cargo**) formalizaram o pedido dessa funcionalidade. A **Atlas Comercial** (cadastrada como `Logística Atlas Comercial Ltda` em `prisma/seed.ts`) alertou que poderá migrar para um concorrente caso a solução não seja entregue até o fim do trimestre / fim de novembro (`[09:00] Marcos`, `[09:45] Marcos`).

### 2.2 Por Que Resolver Agora
Entregar o sistema de webhooks dentro de **3 sprints** (até o fim de novembro) é essencial para reter a conta da Atlas Comercial, atender à demanda conjunta da MaxDistribuição e da Nova Cargo e elevar a competitividade da plataforma B2B do OMS com um padrão de integração moderno e seguro (`[09:00] Marcos`, `[09:25] Diego`, `[09:45]–[09:47] Marcos/Larissa`).

---

## 3. Público-Alvo e Cenários de Uso

### 3.1 Público-Alvo
1. **Clientes B2B Integradores (ex.: Atlas Comercial, MaxDistribuição, Nova Cargo) e suas equipes de engenharia:** Sistemas externos de clientes representados por usuários autenticados na API do OMS que configuram endpoints para receber eventos de pedidos e consultam o histórico de entregas (`[09:00] Marcos`, `[09:31]–[09:34] Marcos`).
2. **Administradores da Plataforma OMS (Role `ADMIN`):** Equipe interna de engenharia/operações responsável por monitorar falhas permanentes na Dead Letter Queue (DLQ) e acionar o reprocessamento (*replay*) manual após o restabelecimento de clientes (`[09:18] Diego`, `[09:35]–[09:36] Larissa/Sofia`).
3. **Operadores do OMS (Role `OPERATOR` / `ADMIN`):** Usuários que operam mudanças de status de pedidos (`PATCH /api/v1/orders/:id/status`) sem que sua operação sofra lentidão decorrente das notificações externas (`[09:04] Bruno`, `[09:32] Bruno`).

### 3.2 Cenários de Uso Principais
- **Cenário 1 — Acompanhamento Logístico Seletivo:** Um cliente B2B cadastra seu endpoint `https://...` filtrando apenas os status `SHIPPED` e `DELIVERED` para acionar seu sistema de recebimento somente quando a mercadoria sai para entrega ou é entregue (`[09:31] Marcos`, `[09:33] Marcos`).
- **Cenário 2 — Manutenção Planejada no Sistema do Cliente:** O servidor de um cliente B2B entra em manutenção planejada por 2 horas pela manhã. Durante esse período, o OMS tenta entregar os eventos e agenda novas tentativas automaticamente via *backoff* exponencial (até ~15h), entregando todas as notificações assim que o cliente volta ao ar, sem perda de dados (`[09:16]–[09:17] Diego`).
- **Cenário 3 — Resposta a Vazamento de Credencial (Rotação de Secret):** A equipe de TI de um cliente identifica que a `secret` do webhook foi exposta acidentalmente em um log interno e aciona a rotação via API; o cliente recebe uma nova `secret` imediatamente enquanto a antiga permanece válida por 24 horas para que ele atualize seus servidores sem interrupção (`[09:21]–[09:22] Sofia/Diego`).
- **Cenário 4 — Auditoria e Troubleshooting de Integração:** O desenvolvedor do cliente consulta `GET /api/v1/webhooks/:id/deliveries` para inspecionar os últimos 100 envios realizados pelo OMS, verificando payload enviado, resposta HTTP devolvida pelo servidor dele e tempo de resposta (`[09:34] Marcos`).
- **Cenário 5 — Recuperação Pós-Indisponibilidade Prolongada (> 15h):** Após uma queda severa no ambiente do cliente que excedeu as 5 tentativas automáticas, o cliente avisa o suporte e um usuário `ADMIN` do OMS aciona `POST /api/v1/admin/webhooks/dead-letter/:id/replay` para reenfileirar os eventos retidos na DLQ (`[09:17] Marcos`, `[09:18] Diego`, `[09:35]–[09:36] Diego/Sofia`).

---

## 4. Objetivos e Métricas de Sucesso

Todas as metas quantitativas abaixo foram extraídas diretamente das definições e restrições acordadas na reunião técnica (`TRANSCRICAO.md`):

| ID | Objetivo de Produto / Negócio | Métrica Indicadora | Meta Quantitativa | Fonte (`TRANSCRICAO.md`) |
| :--- | :--- | :--- | :--- | :--- |
| **OBJ-01** | Entregar notificações em "tempo real" aos clientes B2B sem necessidade de consulta manual | Latência entre a mudança de status do pedido e o disparo da notificação inicial (em condições normais sem falha do cliente) | **< 10 segundos** (viabilizado pelo ciclo de *polling* do worker de **2 segundos**) | `[09:01]–[09:02] Bruno/Marcos`, `[09:09]–[09:10] Diego/Marcos/Larissa` |
| **OBJ-02** | Evitar *churn* e atender à demanda formal dos clientes B2B solicitantes dentro da janela comercial | Entrega em produção para os clientes solicitantes (Atlas Comercial, MaxDistribuição e Nova Cargo) | **3 clientes B2B atendidos** em até **3 sprints** (até o **fim de novembro / fim do trimestre**), reservando **>= 2 dias úteis** para revisão de segurança | `[09:00] Marcos`, `[09:45]–[09:47] Marcos/Larissa/Sofia` |
| **OBJ-03** | Garantir alta tolerância a indisponibilidades temporárias e manutenções nos servidores dos clientes | Janela de cobertura automática de retentativas antes de considerar falha permanente (DLQ) | Até **5 tentativas** cobrindo janela de **~15 horas** (`1m, 5m, 30m, 2h, 12h`), absorvendo manutenções de **2 horas** sem perda de eventos | `[09:15]–[09:17] Diego/Marcos/Larissa` |
| **OBJ-04** | Proteger a estabilidade do OMS contra clientes lentos e garantir autoatendimento de diagnóstico | Timeout máximo por tentativa de entrega e profundidade do histórico de entregas consultável pelo cliente | Timeout HTTP estrito de **10 segundos** por chamada e consulta das **últimas 100 entregas** por webhook (`GET /webhooks/:id/deliveries`) | `[09:34] Marcos`, `[09:42] Sofia/Diego` |
| **OBJ-05** | Viabilizar migração segura de credenciais sem downtime no cliente | Janela de validade paralela (*grace period*) da secret antiga após rotação | **24 horas** de validade simultânea entre a secret antiga e a nova | `[09:21]–[09:22] Sofia` |

---

## 5. Escopo da Feature

### 5.1 Escopo Incluído (In Scope — Fase Atual / MVP)
1. **Notificações Outbound de Mudança de Status de Pedido:** Disparo automático de eventos `"order.status_changed"` do OMS para os endpoints cadastrados pelos clientes (`[09:02]–[09:03] Marcos/Sofia`, `[09:43] Diego`).
2. **Gestão de Configuração de Webhooks via API (CRUD):** Endpoints autenticados para criar (`POST`), listar (`GET`), editar (`PATCH`) e remover (`DELETE`) configurações de webhooks vinculadas a um `customer_id`, com filtro seletivo de quais status (`OrderStatus`) o endpoint deseja receber (`[09:31]–[09:33] Marcos/Larissa/Bruno`).
3. **Segurança HMAC-SHA256 e Rotação de Secret:** Geração automática de `secret` única por endpoint na criação, assinatura `HMAC-SHA256` enviada no header `X-Signature` e endpoint para rotação de secret com *grace period* de 24 horas (`[09:20]–[09:22] Sofia`, `[09:31] Marcos`).
4. **Entrega Confiável (Transactional Outbox + Retry + DLQ):** Persistência atômica na mudança de status do pedido, processamento assíncrono em worker separado (polling de 2s), 5 tentativas com *backoff* exponencial (`1m/5m/30m/2h/12h`) e armazenamento de falhas definitivas em tabela de Dead Letter Queue (`[09:06]–[09:18] Diego/Larissa`).
5. **Histórico de Entregas para o Cliente:** Endpoint `GET /api/v1/webhooks/:id/deliveries` retornando as últimas 100 entregas com status, payload, response e tempo de resposta (`[09:34] Marcos`).
6. **Replay Administrativo de DLQ:** Endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` restrito à role `ADMIN` com log de auditoria (`[09:18] Diego`, `[09:35]–[09:36] Diego/Sofia/Larissa`).
7. **Documentação no Portal de Desenvolvedor:** Guia para clientes sobre integração via API, validação de assinatura HMAC e deduplicação via `X-Event-Id` (`[09:26] Marcos`, `[09:40] Marcos`).

### 5.2 Fora de Escopo (Out of Scope — Descartados ou Adiados na Reunião)
Os itens abaixo foram explicitamente discutidos e **excluídos ou adiados** durante a reunião técnica:
1. **Alertas por E-mail ao Cliente quando o Webhook Falhar (Adiado para próxima fase):** A sugestão de enviar um e-mail automático ao cliente após 3 falhas consecutivas foi descartada para esta entrega e postergada para uma fase futura após medição de impacto (`[09:37] Marcos`, `[09:37] Larissa`, `[09:38] Marcos`, `[09:48] Larissa`).
2. **Dashboard / Painel Visual para Gestão de Webhooks (Fora de escopo):** Não haverá interface gráfica nesta entrega; o escopo contempla exclusivamente os endpoints de API, sendo o painel visual um projeto separado do time de frontend (`[09:39] Marcos`, `[09:40] Larissa`, `[09:48] Larissa`).
3. **Webhooks Inbound — Recebimento de Webhooks Enviados pelos Clientes (Descartado):** O fluxo é estritamente unidirecional (*outbound*, do OMS para o cliente) (`[09:02] Sofia`, `[09:02] Marcos`, `[09:03] Sofia`).
4. **Rate Limiting de Saída por Cliente (Adiado / Em observação):** O controle de limite de chamadas por minuto para um mesmo cliente não será construído agora; o time observará o comportamento em produção antes de decidir (`[09:38] Diego`, `[09:39] Larissa/Diego`, `[09:48] Larissa`).
5. **Rotina de Arquivamento / Expurgo de Eventos Entregues após 30 Dias (Fora de escopo):** O arquivamento automático de registros antigos da `webhook_outbox` após 30 dias ficou explicitamente fora do escopo desta feature (`[09:08] Diego`).
6. **Garantia de Ordenação Global e Múltiplos Workers Paralelos (Adiado):** Não há requisito de ordenação global por parte dos clientes; a operação inicial será *single-worker* com ordenação implícita por `order_id` (`[09:12]–[09:14] Diego/Larissa/Marcos`).

---

## 6. Requisitos Funcionais

| ID | Requisito Funcional | Descrição de Produto | Origem (`TRANSCRICAO.md` / Código) |
| :--- | :--- | :--- | :--- |
| **RF-01** | **Cadastro de Webhook por Cliente (`POST`)** | O sistema deve permitir cadastrar um endpoint de webhook autenticado informando o cliente (`customer_id`), a `url` de destino (`https://`) e a lista de status de pedido que deseja receber (observando o limite mencionado de até 5 webhooks ativos por cliente). O OMS deve gerar uma `secret` exclusiva para o endpoint e devolvê-la na resposta de criação. | `[09:21] Sofia/Bruno`, `[09:23] Sofia`, `[09:31] Marcos`, `[09:32] Larissa`, `[09:44] Sofia` |
| **RF-02** | **Listagem de Webhooks de um Cliente (`GET`)** | O sistema deve permitir listar todos os webhooks configurados para um determinado cliente (`customer_id`). | `[09:32] Larissa`, `[09:33] Bruno` |
| **RF-03** | **Edição de Configuração de Webhook (`PATCH`)** | O sistema deve permitir atualizar um webhook existente (alterar URL `https://`, modificar a lista de status de pedidos monitorados e ativar/desativar o endpoint). | `[09:21] Bruno`, `[09:33] Bruno/Marcos` |
| **RF-04** | **Remoção de Webhook (`DELETE`)** | O sistema deve permitir excluir um endpoint de webhook cadastrado. | `[09:33] Bruno` |
| **RF-05** | **Filtro Seletivo de Eventos por Status** | Ao ocorrer uma mudança de status em um pedido, o sistema deve verificar quais webhooks ativos daquele cliente estão inscritos no novo status; se nenhum webhook escutar aquele status, nenhuma notificação deve ser gerada nem enfileirada. | `[09:33] Marcos`, `[09:34] Bruno/Diego` |
| **RF-06** | **Geração Consistente de Notificação de Mudança de Status** | Quando houver webhook ativo inscrito no novo status do pedido, o sistema deve registrar o evento de forma consistente junto à mudança de status, preservando a fotografia (*snapshot*) enxuta dos dados básicos do pedido no momento exato da transição (sem a lista de itens do pedido, conforme contrato detalhado no [FDD](./FDD.md)). | `[09:06] Diego`, `[09:40] Bruno`, `[09:43] Diego`, `[09:51]–[09:52] Bruno/Larissa/Diego` |
| **RF-07** | **Envio Assíncrono com Cabeçalhos de Segurança e Rastreabilidade** | O sistema deve entregar os eventos pendentes via HTTP `POST` para a URL do cliente incluindo assinatura criptográfica (`X-Signature`), identificador único do evento (`X-Event-Id`), carimbo de tempo (`X-Timestamp`) e identificador do webhook (`X-Webhook-Id`). | `[09:09] Diego`, `[09:20] Sofia`, `[09:25] Diego`, `[09:44]–[09:45] Diego/Sofia` |
| **RF-08** | **Rotação de Secret com Grace Period de 24 Horas** | O cliente deve poder solicitar uma nova `secret` para seu endpoint via API; ao rotacionar, a `secret` antiga deve permanecer válida em paralelo por **24 horas** antes de ser invalidada. | `[09:21] Sofia`, `[09:22] Sofia`, `[09:48] Larissa` |
| **RF-09** | **Retentativa com Backoff Exponencial e Encaminhamento à DLQ** | Caso o envio falhe ou exceda 10s de timeout, o sistema deve retentar até **5 vezes** nos intervalos de **1m, 5m, 30m, 2h e 12h** (~15h). Após esgotar as 5 tentativas de retry sem sucesso, o evento deve ser movido para a Dead Letter Queue (DLQ) com payload, motivo da falha e timestamp. | `[09:15]–[09:18] Diego/Larissa`, `[09:42] Diego` |
| **RF-10** | **Consulta ao Histórico das Últimas 100 Entregas (`GET /webhooks/:id/deliveries`)** | O cliente deve poder consultar as últimas 100 entregas disparadas para um webhook específico, visualizando sucesso/falha, payload enviado, response recebido e tempo de resposta. | `[09:34] Marcos`, `[09:35] Larissa` |
| **RF-11** | **Replay Manual de DLQ Restrito a Admin (`POST /admin/webhooks/dead-letter/:id/replay`)** | Usuários com perfil `ADMIN` devem poder reenfileirar manualmente um evento retido na DLQ para novo envio, registrando em log de auditoria quem executou a operação. | `[09:18] Diego`, `[09:35]–[09:36] Diego/Sofia/Larissa` |

---

## 7. Requisitos Não Funcionais

| ID | Categoria | Especificação | Origem (`TRANSCRICAO.md` / Código) |
| :--- | :--- | :--- | :--- |
| **RNF-01** | **Latência / Tempo Real** | O atraso entre a confirmação da mudança de status do pedido e o disparo inicial do webhook deve ser inferior a **10 segundos**, viabilizado por verificação de novos eventos a cada **2 segundos**. | `[09:02] Marcos`, `[09:09]–[09:10] Diego/Larissa` |
| **RNF-02** | **Isolamento e Disponibilidade da API** | O disparo de webhooks jamais deve bloquear ou atrasar a mudança de status de pedidos na API; o processamento de envios deve ocorrer de forma desacoplada para que lentidões de clientes externos ou reinicializações não impactem a operação do OMS. | `[09:04] Bruno`, `[09:06] Diego`, `[09:11] Diego/Larissa` |
| **RNF-03** | **Segurança de Transporte (TLS)** | Apenas URLs com protocolo seguro `https://` devem ser aceitas no cadastro ou edição de webhooks; URLs `http://` devem ser recusadas. | `[09:23] Sofia`, `[09:28] Bruno` |
| **RNF-04** | **Limite de Tamanho de Payload** | O payload de cada evento não pode exceder **64KB**; caso ultrapasse esse limite, o sistema deve retornar erro sem truncar nem enviar o evento. | `[09:23] Sofia`, `[09:24] Diego/Larissa` |
| **RNF-05** | **Timeout de Chamada Externa** | Toda requisição HTTP disparada para o endpoint de um cliente deve ter timeout máximo de **10 segundos**. | `[09:42] Sofia/Diego` |
| **RNF-06** | **Semântica de Entrega e Idempotência** | Garantia de entrega **at-least-once**, enviando `X-Event-Id` (`UUID` único por evento) para permitir deduplicação idempotente pelo cliente. Ordenação garantida apenas por `order_id` na topologia inicial de worker único. | `[09:12]–[09:13] Diego/Larissa`, `[09:24]–[09:26] Diego/Larissa` |
| **RNF-07** | **Autorização e Auditoria** | Os endpoints de configuração de webhook exigem usuário autenticado na API; o endpoint de replay de DLQ é restrito ao perfil `ADMIN` e exige registro de auditoria do autor da ação. | `[09:32] Larissa`, `[09:36]–[09:37] Sofia/Larissa` |
| **RNF-08** | **Padronização de Erros da API** | Todos os códigos de erro retornados pelo módulo de webhooks devem utilizar o prefixo semântico `WEBHOOK_*`, mantendo o padrão de respostas da API do OMS (detalhado no [FDD](./FDD.md)). | `[09:28]–[09:30] Bruno/Larissa` |

---

## 8. Decisões e Trade-offs Principais

| Decisão | Escolha Adotada | Trade-off Assumido | Referência |
| :--- | :--- | :--- | :--- |
| **Mecanismo de Enfileiramento** | **Transactional Outbox no MySQL existente** em vez de chamada HTTP síncrona ou Redis Streams. | Adiciona escrita na transação do MySQL, mas elimina risco de inconsistência entre status do pedido e notificação e evita o custo operacional de manter um cluster Redis para um time pequeno (`[09:06]–[09:07] Diego`). | [ADR-001](./adrs/ADR-001-padrao-transactional-outbox-no-mysql.md) |
| **Política de Retentativas** | **5 tentativas com backoff (`1m, 5m, 30m, 2h, 12h` = ~15h)** + tabela `webhook_dead_letter` separada, em vez de 3 tentativas (~30m) ou retry infinito. | Mantém eventos falhos agendados por até 15h antes de ir para a DLQ, mas garante que clientes em manutenção planejada (ex.: 2h) não percam notificações e evita eventos pendurados eternamente (`[09:15]–[09:18] Diego`). | [ADR-002](./adrs/ADR-002-politica-de-retry-com-backoff-e-dlq.md) |
| **Segurança e Credenciais** | **HMAC-SHA256 com secret única por endpoint e grace period de 24h**, em vez de secret global ou rotação abrupta. | Exige gerenciar duas chaves durante a janela de 24h de rotação, mas isola o impacto caso um cliente vaze sua secret e permite rotação sem downtime (`[09:20]–[09:22] Sofia/Diego`). | [ADR-003](./adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| **Semântica de Entrega** | **At-Least-Once com `X-Event-Id`**, em vez de *Exactly-Once*. | Transfere ao cliente a responsabilidade de deduplicar eventos repetidos pelo `X-Event-Id` (documentado no portal de desenvolvedor), evitando a complexidade de coordenação bilateral e seguindo o padrão de mercado (Stripe, GitHub) (`[09:24]–[09:26] Diego/Marcos`). | [ADR-004](./adrs/ADR-004-garantia-at-least-once-com-x-event-id.md) |
| **Execução do Worker** | **Processo separado (`src/worker.ts`) em polling de 2s**, em vez de rodar no processo da API ou usar triggers no MySQL. | Introduz até 2s de latência mínima de leitura e requer operar um segundo processo Node.js, mas cumpre folgado o SLA < 10s sem gambiarras em triggers do MySQL e sem acoplar a estabilidade da API (`[09:09]–[09:11] Diego/Larissa`). | [ADR-005](./adrs/ADR-005-worker-em-processo-separado-com-polling.md) |
| **Conteúdo do Payload** | **Snapshot enxuto na inserção (sem `items`)**, em vez de incluir itens ou renderizar no envio. | Caso o cliente precise dos itens do pedido, deve consultar `GET /api/v1/orders/:id`, mas garante payloads leves (< 64KB) e fidelidade histórica ao momento exato da transição (`[09:43] Diego`, `[09:52] Larissa`). | [ADR-001](./adrs/ADR-001-padrao-transactional-outbox-no-mysql.md), [ADR-006](./adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md) |

---

## 9. Dependências

1. **Fluxo de Mudança de Status de Pedidos:** A captura de eventos de notificação depende da integração transacional com o fluxo existente de atualização de status de pedidos do OMS, conforme detalhado no [FDD](./FDD.md) (`[09:40]–[09:41] Bruno/Diego`).
2. **Persistência e Processamento Assíncrono (Sprint 1 e Sprint 2):** Criação das estruturas de banco de dados para configuração de webhooks, fila de saída (*Outbox*), *Dead Letter Queue* e histórico de entregas (Sprint 1), seguida da disponibilização do worker de processamento assíncrono (Sprint 2) (`[09:06] Diego`, `[09:11] Larissa`, `[09:18] Diego`, `[09:34] Marcos`, `[09:46] Larissa`).
3. **Infraestrutura de Autenticação, Autorização e Logs do OMS:** Reutilização do controle de acesso existente (diferenciando perfis `OPERATOR` e `ADMIN`) e da infraestrutura de logs e tratamento de erros da plataforma (`[09:28]–[09:30] Bruno/Larissa`, `[09:36] Larissa`).
4. **Revisão de Segurança Obrigatória (Sofia):** Reserva de **pelo menos 2 dias úteis** antes do deploy para revisão dedicada da implementação de assinatura HMAC-SHA256 e geração/rotação de secrets (`[09:46] Sofia`, `[09:47] Larissa`, `[09:49] Sofia`).
5. **Documentação no Portal do Desenvolvedor (Marcos):** Publicação da documentação externa orientando os clientes sobre os endpoints de API, verificação de `X-Signature` e deduplicação por `X-Event-Id` (`[09:26] Marcos`, `[09:40] Marcos`).

---

## 10. Riscos e Mitigação

| ID | Risco | Probabilidade | Impacto | Estratégia de Mitigação | Origem (`TRANSCRICAO.md`) |
| :--- | :--- | :---: | :---: | :--- | :--- |
| **RSK-01** | **Indisponibilidade ou lentidão nos servidores dos clientes B2B degradar a mudança de status de pedidos no OMS** | **Alta** (já houve clientes com 2h de indisponibilidade em manutenção) | **Alto** | Desacoplar completamente o envio HTTP da transação do pedido via `webhook_outbox`, executar o worker em processo separado (`src/worker.ts`), impor timeout HTTP de **10 segundos** e aplicar 5 retries com backoff (`1m/5m/30m/2h/12h`) seguidos de isolamento em `webhook_dead_letter`. | `[09:04] Bruno`, `[09:11] Diego`, `[09:16]–[09:18] Diego`, `[09:42] Diego` |
| **RSK-02** | **Vazamento de `secret` de webhook em logs da aplicação do cliente** | **Média** (já ocorreu anteriormente com cliente da plataforma) | **Alto** | Gerar uma `secret` exclusiva por endpoint (evitando secret global), disponibilizar endpoint de rotação de secret com *grace period* de **24 horas** e realizar revisão de segurança de **2 dias úteis** antes do deploy (além de manter o padrão de redaction de credenciais no logger Pino do OMS). | `[09:21] Sofia`, `[09:22] Diego`, `[09:46] Sofia`, `src/shared/logger/index.ts` |
| **RSK-03** | **Cliente processar o mesmo evento duas vezes devido à entrega *at-least-once* (retries/timeouts)** | **Média** | **Médio** | Incluir `UUID` único por evento no header `X-Event-Id` e no campo `event_id` do payload JSON, e destacar explicitamente a obrigatoriedade de deduplicação no portal de desenvolvedor. | `[09:24]–[09:26] Diego/Sofia/Marcos` |
| **RSK-04** | **Atraso na entrega das 3 sprints gerar *churn* da Atlas Comercial** | **Baixa / Média** | **Alto** | Manter escopo enxuto (reutilizando MySQL, Prisma, Pino e padrões existentes sem subir nova infra como Redis, e deixando painel visual, alertas por e-mail e rate limiting para fases futuras) e alinhar o cronograma de 3 sprints com o cliente. | `[09:00] Marcos`, `[09:07] Diego`, `[09:37]–[09:40] Larissa`, `[09:45]–[09:47] Marcos/Larissa` |
| **RSK-05** | **Sobrecarga no endpoint do cliente durante picos de atualização de pedidos (ex.: 50 pedidos em 1 minuto)** | **Baixa** | **Médio** | Monitorar o volume de envios por cliente via histórico de entregas e logs em produção e, caso se confirme como problema real, implementar *rate limiting* de saída em iteração futura. | `[09:38]–[09:39] Diego/Larissa` |

---

## 11. Critérios de Aceitação

1. **Cadastro e Segurança de Endpoint:** Um usuário autenticado consegue cadastrar (`POST`), listar (`GET`), editar (`PATCH`) e remover (`DELETE`) webhooks para um `customer_id`; URLs `http://` são rejeitadas com erro `WEBHOOK_INVALID_URL` e a criação retorna uma `secret` exclusiva gerada pelo sistema (`[09:21] Sofia`, `[09:23] Sofia`, `[09:28] Bruno`, `[09:31]–[09:33] Marcos/Bruno`).
2. **Notificação em Tempo Real (< 10s) e Filtro de Status:** Ao alterar o status de um pedido via `PATCH /api/v1/orders/:id/status` para um status assinado pelo webhook ativo do cliente, o endpoint do cliente recebe o `POST` JSON (sem `items`) em menos de 10 segundos contendo os headers `X-Event-Id`, `X-Signature` (HMAC-SHA256 válido), `X-Timestamp` e `X-Webhook-Id`. Se o novo status não estiver assinado por nenhum webhook ativo do cliente, nenhum evento é criado na outbox (`[09:02] Marcos`, `[09:09] Diego`, `[09:34] Bruno`, `[09:43]–[09:45] Diego/Sofia`).
3. **Garantia Transacional:** Se a gravação do evento na `webhook_outbox` falhar, a mudança de status do pedido sofre rollback completo (`[09:06] Diego`, `[09:40] Bruno`).
4. **Rotação de Secret sem Downtime:** Ao acionar a rotação de secret de um webhook, uma nova secret é gerada e a secret anterior continua válida em paralelo por 24 horas (`[09:21]–[09:22] Sofia`).
5. **Resiliência, Backoff e DLQ:** Quando o endpoint do cliente falha ou demora mais de 10 segundos para responder, o sistema agenda até 5 retentativas com intervalos de `1m, 5m, 30m, 2h e 12h` (~15h); após esgotar as 5 tentativas sem sucesso, o evento é movido para a Dead Letter Queue com payload, motivo da falha e timestamp (`[09:15]–[09:18] Diego/Larissa`, `[09:42] Diego`).
6. **Histórico de Entregas:** O endpoint `GET /api/v1/webhooks/:id/deliveries` retorna as últimas 100 entregas daquele webhook informando sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos`).
7. **Replay Administrativo Auditado:** O endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` bloqueia usuários `OPERATOR` (`403 Forbidden`), permite que usuários `ADMIN` reenfileirem o evento para reprocessamento e registra em log de auditoria quem executou o replay (`[09:18] Diego`, `[09:35]–[09:36] Diego/Sofia/Larissa`).

---

## 12. Estratégia de Testes e Validação

Conforme planejado na estimativa de 3 sprints (`[09:46] Larissa`, `[09:46] Sofia`), a validação da feature seguirá quatro etapas antes da liberação aos clientes B2B:

1. **Testes Automatizados Ponta a Ponta da API:**
   - Validar o fluxo completo de configuração e disparo: cadastro, listagem, atualização e remoção de webhooks (incluindo rejeição de URLs `http://` com erro `WEBHOOK_INVALID_URL`), mudança de status de pedido gerando evento apenas para os status assinados pelo cliente, e restrição de perfil `ADMIN` vs `OPERATOR` no endpoint de replay da DLQ (`[09:28] Bruno`, `[09:34] Bruno`, `[09:36] Larissa`, `[09:46] Larissa`).
2. **Testes de Resiliência, Simulação de Falhas e Backoff:**
   - Validar o processamento assíncrono de envios, verificando os cabeçalhos de segurança e rastreabilidade (`X-Signature`, `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`), o corte por timeout de 10 segundos, o agendamento nos 5 intervalos de *backoff* (`1m, 5m, 30m, 2h, 12h`), a transferência para a DLQ após esgotar as 5 tentativas e a listagem em `GET /api/v1/webhooks/:id/deliveries` (`[09:17]–[09:18] Diego`, `[09:34] Marcos`, `[09:42]–[09:45] Diego/Sofia`).
3. **Revisão Dedicada de Segurança (Gate Pré-Deploy de 2 Dias Úteis):**
   - Revisão obrigatória conduzida por Sofia focada na implementação criptográfica do `HMAC-SHA256`, entropia da geração de `secret`, funcionamento do *grace period* de 24 horas na rotação, validação de payload máximo de 64KB e ausência de vazamento de secrets em logs (`[09:21]–[09:24] Sofia`, `[09:46] Sofia`, `[09:49] Sofia`).
4. **Documentação e Homologação com Clientes B2B (Portal de Desenvolvedor):**
   - Publicação da documentação de integração no portal de desenvolvedor detalhando os contratos de API e a obrigatoriedade de deduplicação via `X-Event-Id`, acompanhada do alinhamento de entrega com Atlas Comercial, MaxDistribuição e Nova Cargo (`[09:00] Marcos`, `[09:26] Marcos`, `[09:40] Marcos`, `[09:47] Marcos`).
