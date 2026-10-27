# ADR-001: Padrão Transactional Outbox no MySQL

- **Status:** Aceito
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

O Order Management System (OMS) precisa notificar clientes B2B (como Atlas Comercial, MaxDistribuição e Nova Cargo) em tempo próximo ao real (latência inferior a 10 segundos) sempre que o status de um pedido mudar (`[09:00] Marcos`, `[09:02] Marcos`).

Atualmente, a transição de status de um pedido ocorre no método `OrderService.changeStatus` em `src/modules/orders/order.service.ts` (linhas 126–179), executada dentro de uma transação interativa do Prisma (`this.prisma.$transaction(async (tx) => { ... })`) sobre o banco de dados MySQL 8.0 (`prisma/schema.prisma`, `docker-compose.yml`). Essa transação realiza múltiplas operações críticas de domínio:
1. Busca o pedido e seus itens (`tx.order.findUnique`).
2. Valida a máquina de estados definida em `src/modules/orders/order.status.ts` (`canTransition`).
3. Debita ou estorna o estoque de produtos (`tx.product.update` com `decrement` ou `increment` em `stockQuantity`).
4. Atualiza o status do pedido (`tx.order.update`).
5. Registra a trilha de auditoria na tabela `order_status_history` (`tx.orderStatusHistory.create`).

O problema arquitetural central é garantir atomicidade e consistência entre a mudança de status no banco de dados e o registro do evento de notificação para envio externo via HTTP, sem degradar a performance ou a disponibilidade da transação de pedidos nem introduzir infraestrutura desproporcional ao tamanho da equipe (`[09:03] Larissa`, `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Diego`).

---

## Decisão

Adotar o padrão **Transactional Outbox** persistido no próprio banco de dados **MySQL** existente, por meio de uma tabela dedicada `webhook_outbox` mapeada no `prisma/schema.prisma`.

Os detalhes confirmados da decisão são:
1. **Atomicidade Transacional:** Durante a execução de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro da mesma transação SQL (`tx: Prisma.TransactionClient`) que atualiza `orders`, `products` e `order_status_history`, o sistema invocará uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:06] Diego`, `[09:40] Bruno`, `[09:41] Bruno`, `[09:41] Diego`). Se a inserção na `webhook_outbox` falhar, toda a transação de mudança de status sofre rollback (`[09:40] Bruno`).
2. **Filtragem na Inserção:** Antes de inserir na `webhook_outbox`, o sistema verifica quais endpoints de webhook ativos do `customerId` do pedido estão inscritos para o novo status (`toStatus`). Caso nenhum webhook do cliente escute aquele status, nenhuma linha é inserida na outbox (`[09:33] Marcos`, `[09:34] Bruno`, `[09:34] Diego`).
3. **Snapshot do Payload na Inserção:** O evento gravado na `webhook_outbox` armazena o payload JSON já renderizado (snapshot do momento da transição contendo `event_id`, `event_type: "order.status_changed"`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`, sem a lista de `items`), garantindo que alterações subsequentes no pedido não distorçam o evento histórico (`[09:43] Diego`, `[09:51] Bruno`, `[09:52] Larissa`, `[09:52] Diego`).
4. **Identificadores e Índices:** A tabela `webhook_outbox` utilizará chave primária em formato `UUID` (`@id @default(uuid()) @db.Char(36)`), seguindo o padrão de todas as entidades de domínio em `prisma/schema.prisma` (`[09:51] Diego`, `[09:51] Larissa`), e terá índices nos campos `status` (pendente, processando, falhou, entregue) e `created_at` para otimizar a leitura em pequenos lotes pelo worker (`[09:08] Diego`).

> **Nota sobre Escopo (Adiado):** O arquivamento/expurgo de linhas com status entregue após ~30 dias foi mencionado como prática operacional futura, mas está explicitamente **fora do escopo** desta fase (`[09:08] Diego`).

---

## Alternativas Consideradas

### 1. Disparo HTTP síncrono dentro de `OrderService.changeStatus`
- **Descrição:** Realizar a chamada HTTP `POST` para o endpoint do cliente diretamente dentro do fluxo síncrono de mudança de status do pedido (`[09:03] Larissa`).
- **Motivo do Descarte:** A transação atual em `src/modules/orders/order.service.ts` já executa múltiplas leituras e escritas (atualização de `orders`, inserção em `order_status_history` e atualização de `stock_quantity` em `products`). Incluir uma chamada de rede externa no meio da transação reteria locks no MySQL e faria com que qualquer cliente lento travasse a mudança de status de outros pedidos (`[09:04] Bruno`). Além disso, caso o endpoint do cliente estivesse indisponível, não faria sentido dar rollback na mudança de status do pedido nem perder a notificação (`[09:04] Bruno`, `[09:06] Diego`).

### 2. Mensageria / Fila externa (ex.: Redis Streams / Redis Cluster)
- **Descrição:** Publicar os eventos de mudança de status em uma infraestrutura dedicada de fila ou stream, como Redis Streams (`[09:07] Larissa`).
- **Motivo do Descarte:** Exigiria provisionar, manter e operar mais um componente de infraestrutura para um time pequeno, configurando *overengineering* para o volume atual (`[09:07] Diego`). Ademais, publicar no Redis após o commit do MySQL sem um outbox local ainda deixaria uma janela de inconsistência (dual-write) caso o processo caísse entre o commit no MySQL e o envio ao Redis.

### 3. Filtrar eventos apenas no momento do envio pelo Worker
- **Descrição:** Inserir todas as mudanças de status na `webhook_outbox` indiscriminadamente e deixar o worker decidir se descarta ou envia o evento (`[09:34] Diego`).
- **Motivo do Descarte:** Geraria escrita e armazenamento desnecessários na tabela `webhook_outbox` para transições que nenhum webhook do cliente deseja receber (`[09:34] Bruno`, `[09:34] Diego`).

### 4. Armazenar apenas `order_id` na Outbox e renderizar o payload na hora do disparo
- **Descrição:** Salvar somente a referência do pedido na `webhook_outbox` e montar o JSON consultando a tabela `orders` quando o worker fosse enviar o webhook (`[09:51] Bruno`).
- **Motivo do Descarte:** Se o pedido sofresse outra alteração rapidamente antes do envio pelo worker, o payload enviado refletiria um estado futuro em vez do estado exato da transição que originou o evento (`[09:52] Larissa`, `[09:52] Diego`).

---

## Consequências

### Positivas
- **Consistência Transacional Forte (ACID):** Garantia absoluta de que um evento só existe na `webhook_outbox` se a transição de status em `orders` e `order_status_history` tiver sido efetivada com sucesso (e vice-versa) (`[09:06] Diego`, `[09:40] Bruno`).
- **Simplicidade Operacional:** Reutiliza integralmente o banco MySQL 8.0 e o `PrismaClient` já configurados no projeto (`prisma/schema.prisma`, `src/config/database.ts`), sem introduzir novos brokers (`[09:07] Diego`).
- **Isolamento de Latência na API:** O endpoint `PATCH /api/v1/orders/:id/status` continua respondendo rapidamente apenas com o custo de inserts locais no MySQL, sem depender da disponibilidade ou latência de rede dos clientes (`[09:04] Bruno`).
- **Imutabilidade Histórica do Evento:** O snapshot do payload garante fidelidade aos dados do pedido no exato instante da transição (`[09:52] Larissa`).

### Negativas
- **Aumento de Escrita e Armazenamento no MySQL:** Cada transição de status relevante adiciona linhas de snapshot JSON na tabela `webhook_outbox` dentro da mesma transação do pedido (`[09:07] Bruno`, `[09:40] Bruno`).
- **Necessidade Futura de Expurgo/Arquivamento:** Como os registros entregues permanecem na tabela e o expurgo de 30 dias ficou fora de escopo desta entrega, a tabela crescerá continuamente até que a rotina de limpeza seja implementada em fase futura (`[09:08] Diego`).
- **Acoplamento Transacional com a Gravação da Outbox:** Uma falha de banco ao inserir na `webhook_outbox` aborta a mudança de status do pedido (`[09:40] Bruno`).

### Trade-offs
- **Consistência + Simplicidade de Infraestrutura vs. Carga Adicional no Banco Relacional:** Aceita-se concentrar a fila de saída (outbox) e o armazenamento de snapshots JSON no MySQL transacional para eliminar o risco de perda silenciosa de eventos (*dual-write*) e evitar o custo operacional de manter um cluster Redis/broker externo (`[09:06] Diego`, `[09:07] Diego`, `[09:40] Bruno`).

---

## Referências

### Transcrição (`TRANSCRICAO.md`)
- `[09:03] Larissa` / `[09:04] Bruno` — Discussão inicial e rejeição da chamada HTTP síncrona na transação de `orders`.
- `[09:06] Diego` — Proposta e explicação do padrão Transactional Outbox no SQL junto à atualização de `orders` e `order_status_history`.
- `[09:07] Larissa` / `[09:07] Diego` — Descarte de Redis Streams / Redis Cluster por *overengineering*.
- `[09:08] Diego` / `[09:08] Larissa` — Definição dos índices (`status`, `created_at`), leitura em batch pequeno e confirmação da decisão do Outbox em MySQL (arquivamento de 30 dias fora de escopo).
- `[09:33] Marcos` / `[09:34] Bruno` / `[09:34] Diego` — Decisão de filtrar eventos por status na hora da inserção na outbox.
- `[09:40] Bruno` / `[09:41] Diego` / `[09:41] Bruno` — Integração atômica no `OrderService.changeStatus` via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
- `[09:43] Diego` / `[09:44] Bruno` — Estrutura enxuta do payload JSON sem os itens do pedido.
- `[09:51] Diego` / `[09:51] Larissa` — Uso de `UUID` como chave primária da outbox.
- `[09:51] Bruno` / `[09:52] Larissa` / `[09:52] Diego` — Decisão de armazenar o payload renderizado (snapshot) no momento da inserção.

### Código-Fonte do Repositório
- `src/modules/orders/order.service.ts` (linhas 24, 126–179) — Tipo `TxClient = Prisma.TransactionClient` e método transacional `OrderService.changeStatus`.
- `src/modules/orders/order.status.ts` (linhas 1–37) — Regras de transição de status (`OrderStatus`) e gatilhos de movimentação de estoque.
- `prisma/schema.prisma` (linhas 16–23, 74–131) — Enum `OrderStatus` e models `Order`, `OrderItem` e `OrderStatusHistory` com padrão `@id @default(uuid()) @db.Char(36)`.
- `src/config/database.ts` (linhas 1–10) — Instanciação e configuração do `PrismaClient`.
