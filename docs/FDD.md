# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O OMS (Node.js + TypeScript, Express, MySQL via Prisma) não possui nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Carga) precisam ser notificados em tempo real (abaixo de 10 segundos) quando o status dos pedidos deles muda. Hoje eles fazem polling no `GET /orders`, o que é lento e caro.

A mudança de status de um pedido já é uma transação pesada: atualiza `orders`, insere em `order_status_history` e, dependendo da transição, debita ou repõe `stock_quantity` (ver `src/modules/orders/order.service.ts`, método `changeStatus`). Adicionar uma chamada HTTP síncrona a essa transação é inviável. A solução adotada é o **padrão Outbox no MySQL** com um **worker separado em polling**, conforme os ADRs 001 e 002.

Este documento detalha **como implementar** a feature. As decisões arquiteturais e suas justificativas estão nos ADRs; a proposta de revisão está no RFC.

## Objetivos técnicos

- Registrar o evento de mudança de status na `webhook_outbox` **dentro da mesma transação** da mudança de status, garantindo consistência (status mudou ⇒ evento registrado).
- Entregar o evento ao endpoint do cliente em **menos de 10 segundos** (polling de 2s).
- Garantir **at-least-once** com `X-Event-Id` para deduplicação do lado do cliente.
- Assinar o payload com **HMAC-SHA256** (secret por endpoint) e exigir **TLS**.
- Implementar **retry com backoff exponencial** (5 tentativas) e **DLQ** com replay manual.
- Reutilizar os padrões existentes do projeto (AppError, Pino, error middleware, módulos, Zod, auth).

## Escopo e exclusões

**Incluso:**

- CRUD de configuração de webhooks (criar, listar, editar, remover).
- Rotação de secret com grace period de 24h.
- Outbox, worker, retry, DLQ e replay manual.
- Histórico de entregas (`deliveries`).
- Filtro de eventos por status na inserção da outbox.

**Excluso (descartado ou adiado na reunião):**

- Email como fallback quando o webhook falha 3 vezes seguidas (adiado para próxima fase).
- Rate limiting de envio para o cliente (observar e decidir depois).
- Dashboard visual para o cliente (projeto separado do time de frontend).
- Arquivamento de linhas entregues após 30 dias (fora do escopo desta feature).
- Escala para múltiplos workers em paralelo (problema do futuro).

## Fluxos detalhados

### Fluxo 1 — Criação do evento na outbox (dentro da transação de mudança de status)

1. O cliente chama `PATCH /api/v1/orders/:id/status` com `{ toStatus, reason }`.
2. `OrderService.changeStatus` abre uma transação (`this.prisma.$transaction`).
3. Dentro da transação: valida a transição (`canTransition`), debita/recompõe estoque se aplicável (`shouldDebitStock`/`shouldReplenishStock`), atualiza `orders`, insere em `order_status_history`.
4. Ainda dentro da transação, chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`:
   - Consulta os webhooks ativos do `customerId` do pedido.
   - Filtra pelos status que cada webhook quer ouvir (se nenhum webhook do cliente quer aquele status, **não insere** — economiza linha).
   - Para cada webhook que deve receber o evento, renderiza o payload (snapshot) e insere uma linha em `webhook_outbox` com status `pendente`.
5. Se qualquer inserção na outbox falhar, a transação inteira dá rollback (status não muda sem evento).

### Fluxo 2 — Processamento pelo worker

1. O worker (`src/worker.ts`) roda em processo separado, com polling a cada 2 segundos.
2. A cada ciclo, busca os eventos `pendente` mais antigos em batch pequeno (ex.: 10), ordenados por `created_at`.
3. Para cada evento:
   - Marca como `processando`.
   - Monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`).
   - Faz a chamada HTTP ao endpoint do cliente com timeout de 10 segundos.
   - Se sucesso (2xx): marca como `entregue` e registra a entrega em `webhook_delivery`.
   - Se falha: incrementa o contador de tentativas e agenda o próximo retry conforme o backoff; ao esgotar as 5 tentativas, move para a DLQ.

### Fluxo 3 — Retry com backoff

- Progressão: **1 min, 5 min, 30 min, 2h, 12h** (5 tentativas no total, ~15h de janela).
- O worker, ao processar um evento com tentativas anteriores, só o reprocessa se o intervalo de backoff já tiver decorrido desde a última tentativa.
- Timeout da chamada HTTP: **10 segundos**; cliente que não responde em 10s é tratado como falha e marcado para retry.

### Fluxo 4 — DLQ e replay

1. Após 5 tentativas sem sucesso, o evento é movido para `webhook_dead_letter` com a payload, o motivo da falha e o timestamp.
2. Um operador com role `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
3. O sistema recoloca o evento na `webhook_outbox` como `pendente` (novo ciclo de retry) e registra quem fez o replay (auditoria).

## Contratos públicos

Todos os endpoints abaixo são autenticados com JWT Bearer (`authenticate`). O CRUD de configuração aceita qualquer role autenticada; o replay de DLQ exige role `ADMIN` (`requireRole('ADMIN')`).

### 1. Criar webhook — `POST /api/v1/webhooks`

**Request body:**

```json
{
  "customerId": "3f2b1a9c-0000-0000-0000-000000000001",
  "url": "https://cliente.example.com/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"]
}
```

**Response 201 Created:**

```json
{
  "id": "a1b2c3d4-0000-0000-0000-000000000001",
  "customerId": "3f2b1a9c-0000-0000-0000-000000000001",
  "url": "https://cliente.example.com/webhooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "secret": "sk_webhook_9f8e7d6c5b4a39281706f5e4d3c2b1a0",
  "active": true,
  "createdAt": "2026-09-01T12:00:00.000Z"
}
```

> A `secret` é gerada pelo OMS e devolvida **apenas na criação** (e na rotação). Não é retornada nas listagens.

**Erros:** `WEBHOOK_INVALID_URL` (400, url não é https), `WEBHOOK_INVALID_STATUS` (400, status inválido), `CUSTOMER_NOT_FOUND` (404), `VALIDATION_ERROR` (400).

### 2. Listar webhooks de um customer — `GET /api/v1/webhooks?customerId=:id`

**Response 200 OK:**

```json
{
  "data": [
    {
      "id": "a1b2c3d4-0000-0000-0000-000000000001",
      "customerId": "3f2b1a9c-0000-0000-0000-000000000001",
      "url": "https://cliente.example.com/webhooks/orders",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-01T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

> A `secret` **não** é retornada na listagem.

### 3. Editar webhook — `PATCH /api/v1/webhooks/:id`

**Request body (parcial):**

```json
{
  "url": "https://cliente.example.com/webhooks/orders/v2",
  "statuses": ["SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Response 200 OK:** objeto do webhook atualizado (sem `secret`).

**Erros:** `WEBHOOK_NOT_FOUND` (404), `WEBHOOK_INVALID_URL` (400), `VALIDATION_ERROR` (400).

### 4. Remover webhook — `DELETE /api/v1/webhooks/:id`

**Response 204 No Content.**

**Erros:** `WEBHOOK_NOT_FOUND` (404).

### 5. Rotacionar secret — `POST /api/v1/webhooks/:id/rotate-secret`

**Response 200 OK:**

```json
{
  "id": "a1b2c3d4-0000-0000-0000-000000000001",
  "secret": "sk_webhook_7a6b5c4d3e2f1098a7b6c5d4e3f2a1b0",
  "previousSecretExpiresAt": "2026-09-02T12:00:00.000Z"
}
```

> A secret anterior permanece válida por **24 horas** (grace period) para migração do cliente.

**Erros:** `WEBHOOK_NOT_FOUND` (404).

### 6. Histórico de entregas — `GET /api/v1/webhooks/:id/deliveries`

**Response 200 OK:**

```json
{
  "data": [
    {
      "id": "d1e2f3a4-0000-0000-0000-000000000001",
      "webhookId": "a1b2c3d4-0000-0000-0000-000000000001",
      "eventId": "e5f6a7b8-0000-0000-0000-000000000001",
      "status": "delivered",
      "httpStatus": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 120,
      "attempt": 1,
      "deliveredAt": "2026-09-01T12:00:05.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

> Retorna os últimos 100 registros por padrão (sucesso/falha, payload, response, tempo de resposta).

**Erros:** `WEBHOOK_NOT_FOUND` (404).

### 7. Replay de DLQ — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role ADMIN)

**Response 200 OK:**

```json
{
  "replayed": true,
  "deadLetterId": "f1a2b3c4-0000-0000-0000-000000000001",
  "replayedBy": "user-uuid-admin"
}
```

**Erros:** `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404), `FORBIDDEN` (403, se não for ADMIN).

### Payload do evento (enviado ao cliente)

```json
{
  "event_id": "e5f6a7b8-0000-0000-0000-000000000001",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-01T12:00:00.000Z",
  "order_id": "3f2b1a9c-0000-0000-0000-000000000010",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "3f2b1a9c-0000-0000-0000-000000000001",
  "total_cents": 15000
}
```

> Payload enxuto: **não** envia `items`. Se o cliente quiser detalhes, ele consulta `GET /orders/:id` depois.

### Headers do request de entrega

| Header | Valor |
|--------|-------|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (deduplicação do lado do cliente) |
| `X-Signature` | HMAC-SHA256 do corpo, hex, com a secret do endpoint |
| `X-Timestamp` | timestamp do envio (detecção de replay attack) |
| `X-Webhook-Id` | id do endpoint webhook (cliente com vários cadastros identifica a origem) |

## Matriz de erros previstos

Códigos no padrão `WEBHOOK_*`, seguindo o padrão de códigos de erro do projeto (ex.: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`).

| Código | HTTP | Cenário |
|--------|------|---------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook de configuração não encontrado |
| `WEBHOOK_INVALID_URL` | 400 | URL não é `https` (TLS obrigatório) |
| `WEBHOOK_INVALID_STATUS` | 400 | Status na lista de eventos não é um `OrderStatus` válido |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret ausente/obrigatória em operação que a exige |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload do evento ultrapassa o limite de 64KB |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Registro de DLQ não encontrado para replay |
| `WEBHOOK_DELIVERY_FAILED` | 502 | Falha permanente de entrega (após retries) — usado em logs/observabilidade |
| `CUSTOMER_NOT_FOUND` | 404 | Customer informado no cadastro não existe |
| `FORBIDDEN` | 403 | Role insuficiente (ex.: replay sem ADMIN) |
| `VALIDATION_ERROR` | 400 | Falha de validação Zod |

## Estratégias de resiliência

- **Timeout**: chamada HTTP do worker com timeout de **10 segundos**; cliente que não responde em 10s é tratado como falha.
- **Retry com backoff exponencial**: 5 tentativas, progressão 1m/5m/30m/2h/12h (~15h de janela).
- **DLQ**: após 5 tentativas, evento vai para `webhook_dead_letter` (payload, motivo, timestamp) para debug e reprocessamento manual.
- **Fallback**: não há fallback automático nesta fase (email como fallback foi adiado). O fallback operacional é o replay manual da DLQ.
- **Consistência**: evento registrado na mesma transação da mudança de status (rollback conjunto).
- **Limite de payload**: 64KB; se ultrapassar, o envio **erra** (`WEBHOOK_PAYLOAD_TOO_LARGE`) em vez de truncar.

## Observabilidade

- **Métricas** (a instrumentar no worker e nos endpoints):
  - `webhook_events_created_total` (por status de evento).
  - `webhook_deliveries_total` (por resultado: sucesso/falha).
  - `webhook_delivery_duration_ms` (histograma).
  - `webhook_retries_total` (por tentativa).
  - `webhook_dead_letter_total`.
  - `webhook_outbox_pending` (gauge).
- **Logs** (Pino, `src/shared/logger/index.ts`):
  - Log estruturado por entrega: `eventId`, `webhookId`, `customerId`, `httpStatus`, `durationMs`, `attempt`.
  - Log de falha com motivo e stack.
  - Log de replay de DLQ com `replayedBy` (auditoria).
  - Redação automática de secrets via `redact` do Pino (paths `*.secret`, `*.signature`).
- **Tracing**:
  - Propagar `requestId`/`X-Request-Id` (já usado no `request-logger.middleware.ts`) para correlacionar a mudança de status com a entrega do evento.
  - Registrar `eventId` como span/contexto para rastrear o ciclo de vida do evento (criação → processamento → entrega → DLQ).

## Dependências e compatibilidade

- **Banco**: MySQL existente via Prisma. Novos modelos: `WebhookConfig`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDelivery` (ver `prisma/schema.prisma`).
- **Stack**: Node.js ≥ 20, TypeScript, Express, Prisma 5.22, Zod, Pino. Nenhuma dependência nova obrigatória para o núcleo (HMAC via `crypto` nativo do Node; HTTP via `fetch`/`undici` ou `axios` a decidir na implementação).
- **Compatibilidade**: o módulo segue o padrão de módulos existente (`src/modules/*`), o middleware de erro centralizado já trata `AppError`, Zod e Prisma, e o `buildApiRouter` em `src/routes/index.ts` precisa registrar a rota de webhooks.
- **Configuração**: novas variáveis de ambiente (ex.: `WEBHOOK_POLL_INTERVAL_MS`, `WEBHOOK_HTTP_TIMEOUT_MS`, `WEBHOOK_MAX_ATTEMPTS`) com defaults, adicionadas ao schema Zod em `src/config/env.ts`.

## Critérios de aceite técnicos

- [ ] Evento é inserido na `webhook_outbox` na mesma transação da mudança de status; se a inserção falhar, rollback.
- [ ] Filtro de eventos por status aplicado na inserção (webhook que não quer o status não gera linha).
- [ ] Worker em processo separado (`src/worker.ts`, `npm run worker`) faz polling a cada 2s e entrega em <10s.
- [ ] Payload assinado com HMAC-SHA256; headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` presentes.
- [ ] Retry com backoff 1m/5m/30m/2h/12h (5 tentativas); após esgotar, evento vai para a DLQ.
- [ ] Replay de DLQ exige role `ADMIN` e registra quem fez o replay.
- [ ] URL de webhook deve ser `https`; `http` é recusado com `WEBHOOK_INVALID_URL`.
- [ ] Payload > 64KB erra com `WEBHOOK_PAYLOAD_TOO_LARGE`.
- [ ] Histórico de entregas disponível em `GET /webhooks/:id/deliveries`.
- [ ] Códigos de erro com prefixo `WEBHOOK_`; erros tratados pelo middleware centralizado.
- [ ] Testes ponta a ponta cobrindo: mudança de status gera evento, retry, DLQ, replay, validação de URL, HMAC.

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Cliente offline por longos períodos | Média | Alto | Retry com backoff (~15h) + DLQ + replay manual |
| Vazamento de secret | Baixa | Alto | Secret por endpoint, rotação com grace period de 24h, redação em logs |
| Duplicidade de entrega | Média | Médio | At-least-once documentado; cliente deduplica via `X-Event-Id` |
| Payload grande inflando a outbox | Baixa | Médio | Limite de 64KB + payload enxuto (sem items) |
| Inserção na outbox falhar e travar mudança de status | Baixa | Alto | Transação atômica; teste de rollback obrigatório |
| Sobrecarga de envio (muitos eventos por minuto) | Média | Médio | Observar; rate limiting adiado (questão em aberto) |

## Integração com o sistema existente

Esta seção nomeia os caminhos de arquivo reais do código base e descreve como o módulo de webhooks se integra com cada um.

1. **`src/modules/orders/order.service.ts`** — o método `changeStatus` é o ponto crítico. Hoje a transação faz `tx.order.update`, `tx.orderStatusHistory.create` e, se aplicável, `debitStock`/`replenishStock`. A integração adiciona, **dentro da mesma transação**, a chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` (função pura do módulo de webhooks que recebe o `tx` da transação atual). Se a inserção na outbox falhar, a transação inteira dá rollback — não pode haver status mudado sem evento. O `OrderService` não recebe o repository inteiro de webhook; apenas a função de enqueue.

2. **`src/modules/orders/order.status.ts`** — define `canTransition`, `shouldDebitStock` e `shouldReplenishStock`, usados no `changeStatus`. O módulo de webhooks reutiliza o enum `OrderStatus` (via `@prisma/client`) para validar a lista de status que cada webhook quer ouvir (`WEBHOOK_INVALID_STATUS`) e para filtrar eventos na inserção da outbox.

3. **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`** — o módulo de webhooks estende `AppError` e as classes existentes (`NotFoundError`, `BadRequestError`, `UnprocessableEntityError`, `ForbiddenError`) para criar os erros com prefixo `WEBHOOK_*` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_PAYLOAD_TOO_LARGE`). O middleware de erro centralizado (`src/middlewares/error.middleware.ts`) já trata `AppError`, Zod e Prisma, capturando esses erros sem mudança.

4. **`src/middlewares/auth.middleware.ts`** — o CRUD de configuração de webhooks usa `authenticate` (qualquer role autenticada); o replay de DLQ usa `authenticate` + `requireRole('ADMIN')`, reaproveitando o middleware existente. O `customerId` é passado no body/path (não vem do JWT), conforme decidido na reunião.

5. **`src/routes/index.ts` e `src/app.ts`** — o `buildApiRouter` registra os routers dos módulos; o módulo de webhooks adiciona `buildWebhookRouter` (e a rota admin) em `src/routes/index.ts`, e `buildControllers` em `src/app.ts` instancia `WebhookRepository`, `WebhookService` e `WebhookController`. O `src/server.ts` serve de referência para a nova entry `src/worker.ts`.

6. **`src/config/database.ts` e `src/config/env.ts`** — o worker abre um `PrismaClient` separado (mesmo banco, mesma `DATABASE_URL`), usando `createPrismaClient`. Novas variáveis de ambiente de webhook são adicionadas ao schema Zod em `src/config/env.ts`.

7. **`prisma/schema.prisma`** — novos modelos `WebhookConfig`, `WebhookOutbox`, `WebhookDeadLetter` e `WebhookDelivery`, com `id String @id @default(uuid()) @db.Char(36)` (padrão UUID do projeto, ADR-008), índices em status e `created_at` na outbox, e relação com `Customer`/`Order`.

8. **`src/shared/logger/index.ts`** — o worker e o módulo usam o mesmo logger Pino, com paths de redação adicionados para secrets (`*.secret`, `*.signature`).

9. **`src/shared/http/response.ts`** — o `paginated`/`buildPagination` é reutilizado nas listagens (`GET /webhooks`, `GET /webhooks/:id/deliveries`).

10. **`src/middlewares/validate.middleware.ts`** — os schemas Zod de webhook (`webhook.schemas.ts`) são validados com o middleware `validate` existente, incluindo a validação de URL `https`.
