# ADR-004: Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id

- **Status:** Aceito
- **Data:** 2026-10-26
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior — Plataforma), Bruno (Engenheiro Pleno — Pedidos), Sofia (Engenheira de Segurança), Marcos (Product Manager)

---

## Contexto

Em sistemas distribuídos que realizam notificações via HTTP sobre redes não confiáveis, falhas de comunicação ambíguas podem ocorrer — por exemplo, quando o endpoint do cliente recebe e processa o webhook com sucesso, mas a resposta HTTP 2xx sofre queda de conexão ou ultrapassa o timeout de 10 segundos do worker (`[09:24] Diego`, `[09:42] Diego`). Nesse cenário, o worker interpretará a tentativa como falha e reenviará o evento conforme a política de retry ([ADR-002](./ADR-002-politica-de-retry-com-backoff-e-dlq.md)), ou um administrador poderá acionar o replay manual da DLQ (`[09:18] Diego`, `[09:24] Diego`).

Além disso, mudanças rápidas e sucessivas de status de um mesmo pedido (ex.: `PAID -> PROCESSING -> SHIPPED`) levantam a questão de como o sistema trata a ordenação e a deduplicação das mensagens entregues aos clientes B2B (`[09:12] Larissa`, `[09:25] Bruno`).

---

## Decisão

Adotar a semântica de entrega **At-Least-Once (pelo menos uma vez)** combinada com identificação única por evento via **UUID no header `X-Event-Id` e no corpo do payload (`event_id`)**, delegando a deduplicação (idempotência) ao consumidor do webhook:

1. **Semântica At-Least-Once:** O OMS garante que todo evento registrado na `webhook_outbox` será entregue pelo menos uma vez (respeitando o limite de 5 tentativas e o mecanismo de replay da DLQ), assumindo explicitamente que o cliente pode receber o mesmo evento mais de uma vez (`[09:24] Diego`, `[09:26] Larissa`).
2. **Identificador Único de Evento (`X-Event-Id` / `event_id`):** Cada evento recebe um `UUID` único gerado no momento em que entra na `webhook_outbox` (`[09:25] Diego`, `[09:51] Larissa`). Esse mesmo `UUID` é enviado em todas as tentativas de entrega daquele evento tanto no header HTTP `X-Event-Id` quanto na propriedade `event_id` do payload JSON (`[09:25] Diego`, `[09:43] Diego`, `[09:44] Diego`).
3. **Deduplicação no Cliente e Documentação:** O cliente utiliza o `X-Event-Id` (`event_id`) para deduplicar entregas repetidas do seu lado. Essa responsabilidade será destacada na documentação do portal de desenvolvedores voltada aos clientes (`[09:25] Diego`, `[09:25] Sofia`, `[09:26] Marcos`, `[09:26] Larissa`).
4. **Ordenação (Ordering) como Limitação Conhecida:** Não há garantia de ordenação global entre eventos. Enquanto o sistema operar com um único worker (*single-worker*), o processamento em ordem de `created_at` da `webhook_outbox` proporciona ordenação implícita por `order_id`. Essa condição é registrada como limitação conhecida da arquitetura atual (`[09:12] Diego`, `[09:13] Larissa`, `[09:14] Marcos`).

> **Nota sobre Escopo (Adiado):** Mecanismos para manter ordenação caso o sistema escale para múltiplos workers em paralelo no futuro (como particionamento por `order_id` ou lock pessimista) foram identificados como problema futuro e **não fazem parte do escopo atual** (`[09:12] Diego`, `[09:13] Bruno`, `[09:13] Diego`).

---

## Alternativas Consideradas

### 1. Garantia de Entrega Exactly-Once (Exatamente Uma Vez)
- **Descrição:** Implementar um protocolo coordenado entre o OMS e os servidores dos clientes para garantir que nenhum evento seja entregue mais de uma vez (`[09:25] Diego`).
- **Motivo do Descarte:** Exigiria coordenação distribuída em duas fases (handshake/ack transacional) entre a plataforma e cada cliente B2B, elevando drasticamente a complexidade de implementação e integração de ambos os lados. O padrão consolidado de mercado (adotado por provedores como Stripe e GitHub) é *at-least-once* com `event_id` para deduplicação no destinatário, o que resolve 99% dos cenários com baixa complexidade (`[09:25] Diego`).

### 2. Garantia de Ordenação Global com Múltiplos Workers Particionados desde o Dia 1
- **Descrição:** Implementar desde já particionamento de filas por `order_id` ou locks pessimistas complexos para garantir ordenação estrita em execução paralela (`[09:12] Larissa`, `[09:13] Bruno`, `[09:13] Diego`).
- **Motivo do Descarte:** Os clientes B2B nunca solicitaram garantia de ordenação global (`[09:14] Marcos`), e a operação inicial com *single-worker* lendo por `created_at` já preserva a ordem natural de criação na outbox sem complexidade extra (`[09:12] Diego`, `[09:13] Larissa`).

---

## Consequências

### Positivas
- **Simplicidade de Protocolo HTTP:** Os clientes precisam apenas expor um endpoint HTTPS padrão que receba `POST` JSON e armazene os `X-Event-Id` já processados (`[09:25] Diego`).
- **Tolerância a Retries e Replay de DLQ:** Como o `event_id` é gerado na inserção da `webhook_outbox` e preservado em todas as retentativas e replays de DLQ, o reenvio após timeouts de rede é seguro para clientes idempotentes (`[09:25] Diego`).
- **Alinhamento com Padrões de Mercado:** Segue exatamente a convenção esperada por desenvolvedores acostumados com webhooks de plataformas como Stripe e GitHub (`[09:25] Diego`).

### Negativas
- **Responsabilidade de Idempotência Transferida ao Cliente:** Clientes que não implementarem deduplicação por `X-Event-Id` poderão processar a mesma mudança de status mais de uma vez em cenários de falha de rede ou timeout na resposta (`[09:24] Diego`, `[09:25] Sofia`).
- **Ordenação Condicionada a Single-Worker:** Caso um evento específico falhe e entre em backoff enquanto um evento subsequente do mesmo pedido seja processado, ou caso múltiplos workers sejam ativados no futuro sem particionamento, eventos podem chegar fora de ordem (`[09:12] Diego`, `[09:13] Larissa`).

### Trade-offs
- **Confiabilidade de Entrega + Simplicidade (At-Least-Once) vs. Deduplicação no Consumidor:** Abre-se mão de uma coordenação distribuída *exactly-once* no servidor em troca de um modelo simples, resiliente e padrão de mercado, exigindo em contrapartida que os clientes tratem idempotência via `X-Event-Id` (`[09:25] Sofia`, `[09:25] Diego`, `[09:26] Larissa`).

---

## Referências

### Transcrição (`TRANSCRICAO.md`)
- `[09:12] Larissa` / `[09:12] Diego` / `[09:13] Bruno` / `[09:13] Diego` / `[09:13] Larissa` / `[09:14] Marcos` — Discussão sobre ordering em mudanças rápidas de status, definição de single-worker por `created_at` e registro como limitação conhecida.
- `[09:24] Diego` / `[09:25] Bruno` / `[09:25] Diego` — Definição da garantia *at-least-once* e geração de `UUID` único enviado no header `X-Event-Id`.
- `[09:25] Sofia` / `[09:25] Diego` / `[09:26] Marcos` / `[09:26] Larissa` — Trade-off da responsabilidade de deduplicação no cliente, comparação com Stripe/GitHub, documentação no portal de desenvolvedor e confirmação da decisão.
- `[09:43] Diego` / `[09:44] Diego` — Inclusão de `event_id` no payload JSON e `X-Event-Id` nos headers HTTP.

### Código-Fonte do Repositório
- `package.json` (linha 33) e `src/middlewares/request-logger.middleware.ts` (linha 2) — Biblioteca `uuid` (`v4 as uuidv4`) já presente nas dependências do projeto e utilizada para geração de identificadores únicos.
- `prisma/schema.prisma` (linhas 26, 41, 57, 75, 100, 117) — Padrão `@id @default(uuid()) @db.Char(36)` utilizado em todo o banco de dados.
