# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|-------|-------|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão (proposta submetida à equipe) |
| **Data** | 2026-09-01 |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Sofia (Eng. de Segurança) |

## Resumo executivo (TL;DR)

Clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) precisam ser notificados em tempo real quando o status dos pedidos deles muda no OMS. Hoje eles fazem polling no `GET /orders`, o que é lento e caro.

Proposta: um **sistema de webhooks outbound** baseado no **padrão Outbox no MySQL**. Quando o status de um pedido muda, dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, é inserido um evento na tabela `webhook_outbox`. Um **worker em processo separado** faz polling a cada 2 segundos, dispara a chamada HTTP ao cliente com assinatura **HMAC-SHA256** (secret por endpoint) e garante **at-least-once** com `X-Event-Id`. Falhas seguem **retry com backoff exponencial** (5 tentativas) e, ao esgotar, vão para uma **DLQ** em tabela separada, com reprocessamento manual via endpoint admin.

A solução reutiliza ao máximo os padrões existentes do projeto (AppError, Pino, error middleware, módulos em `src/modules`, schemas Zod, `authenticate`/`requireRole`). Prazo estimado: **3 sprints**, com revisão de segurança da Sofia antes do deploy.

## Contexto e problema

O OMS (Node.js + TypeScript, Express, MySQL via Prisma) não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Três clientes B2B pediram notificação em tempo real de mudança de status. Para eles, "tempo real" é qualquer coisa abaixo de **10 segundos**. A Atlas ameaçou migrar para o concorrente se a entrega não ocorrer até o fim do trimestre.

O ciclo de vida do pedido tem máquina de estados controlada (`src/modules/orders/order.status.ts`), controle transacional de estoque e auditoria de mudanças de status (`src/modules/orders/order.service.ts`, método `changeStatus`). A mudança de status já é uma transação pesada; adicionar uma chamada HTTP síncrona a ela é inviável.

## Proposta técnica (visão geral)

1. **Outbox no MySQL (ADR-001)**: tabela `webhook_outbox`; o evento é inserido na mesma transação da mudança de status. Consistência garantida entre status e evento.
2. **Worker separado em polling (ADR-002)**: `src/worker.ts` (script `npm run worker`), polling a cada 2s, mesmo banco e mesmo Prisma client (instância separada por processo). Single-worker dá ordering implícito por `order_id`.
3. **Retry com backoff e DLQ (ADR-003)**: 5 tentativas (1m/5m/30m/2h/12h); falha permanente vai para `webhook_dead_letter`; replay manual via `POST /admin/webhooks/dead-letter/:id/replay` (role ADMIN).
4. **Segurança (ADR-004)**: HMAC-SHA256 sobre o corpo, header `X-Signature`, secret única por endpoint, rotação com grace period de 24h, TLS obrigatório (https).
5. **Entrega (ADR-005)**: at-least-once com `X-Event-Id` (UUID por evento) para deduplicação do lado do cliente.
6. **Padrões do projeto (ADR-006)**: módulo `src/modules/webhooks`, prefixo `WEBHOOK_` nos códigos de erro, reuso de AppError, Pino, error middleware, Zod e auth.
7. **Payload (ADR-007)**: snapshot renderizado na inserção; JSON enxuto com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` (sem items).
8. **Identificadores (ADR-008)**: UUID, seguindo o padrão do projeto.

O detalhamento de implementação (fluxos, contratos, matriz de erros, observabilidade, integração com o código) está no **FDD**.

## Alternativas consideradas

1. **Notificação síncrona dentro do `OrderService`** — descartada. A transação de mudança de status já é pesada (atualiza `orders`, insere em `order_status_history`, debita/recompõe estoque). Uma chamada HTTP no meio travaria a mudança de status de outros pedidos se o cliente estivesse lento, e não há como dar rollback na mudança de status se o cliente estiver fora do ar. *Trade-off que motivou o descarte: acoplamento da latência do cliente à transação de status.*
2. **Redis Streams / fila externa** — descartada. Exigiria subir e operar infraestrutura adicional (Redis Cluster), considerada *overengineering* para um time pequeno. *Trade-off: custo operacional de nova infraestrutura vs. benefício marginal sobre o Outbox no MySQL existente.*
3. **Trigger de banco para reatividade** — descartada. MySQL não tem listener nativo tipo `NOTIFY/LISTEN` do Postgres; trigger só executa SQL e não notifica processo externo. *Trade-off: reatividade vs. improviso frágil para avisar o worker.*

## Questões em aberto

1. **Rate limiting de envio para o cliente** — levantado por Diego ([09:38]): se um cliente tem 50 pedidos mudando de status em um minuto, o OMS bombardeia o endpoint com 50 chamadas. Decidido **observar e decidir depois**; não entra no escopo desta fase.
2. **Escala para múltiplos workers** — levantado por Bruno ([09:13]): escalar para múltiplos workers em paralelo quebraria a garantia de ordering. Particionamento por `order_id` ou lock pessimista ficam para o futuro; hoje é single-worker.
3. **Endurecimento de permissões do CRUD de webhook** — levantado por Sofia ([09:37]): por enquanto qualquer role autenticada pode fazer o CRUD de configuração; endurecer é considerado para o futuro.
4. **Email como fallback** — levantado por Marcos ([09:37]): avisar o cliente por email quando o webhook falhar 3 vezes seguidas. **Adiado** para a próxima fase, depois de medir o impacto.

## Impacto e riscos

- **Impacto no `OrderService.changeStatus`**: a transação passa a inserir também na `webhook_outbox`. Se a inserção falhar, rollback — não pode haver status mudado sem evento.
- **Risco de latência**: polling de 2s atende o requisito de <10s; aceito.
- **Risco de segurança**: exposição de dados de pedidos a endpoints externos; mitigado por HMAC-SHA256, secret por endpoint, TLS obrigatório e revisão de segurança da Sofia.
- **Risco de indisponibilidade do cliente**: mitigado por retry com backoff (até ~15h) e DLQ com replay manual.
- **Risco de duplicidade**: at-least-once; cliente deduplica via `X-Event-Id`.

## Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Snapshot do payload na inserção](adrs/ADR-007-snapshot-do-payload-na-insercao.md)
- [ADR-008 — UUID como identificador da outbox](adrs/ADR-008-uuid-como-identificador-da-outbox.md)
