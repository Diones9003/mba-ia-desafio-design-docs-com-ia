# Tracker de Rastreabilidade

Este documento é uma referência cruzada que mapeia cada item registrado nos documentos do pacote (PRD, RFC, FDD e ADRs) à sua origem na **transcrição da reunião** (`TRANSCRICAO.md`) ou no **código fonte** da aplicação. Ele garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código, evitando alucinações da IA.

**Legenda de Fonte:** `TRANSCRICAO` (timestamp + falante) ou `CODIGO` (caminho de arquivo real).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|--------------------|-------|-------------|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram notificação em tempo real de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Hoje os clientes fazem polling no GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Atlas ameaça migrar para o concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Requisito Não Funcional | "Tempo real" para os clientes é qualquer coisa abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Restrição | Webhooks são outbound (só saem do OMS para os clientes) | TRANSCRICAO | [09:02] Sofia, [09:03] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (url, statuses, customerId); secret gerada pelo OMS | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotacionar secret; antiga válida por 24h (grace period) | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimos 100, sucesso/falha, payload, response, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status na inserção da outbox | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Evento registrado na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Worker separado processa eventos em polling de 2s | TRANSCRICAO | [09:09] Diego, [09:10] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Retry com backoff (5 tentativas) e DLQ | TRANSCRICAO | [09:15] Diego, [09:17] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay de DLQ exige role ADMIN com auditoria | TRANSCRICAO | [09:35] Larissa, [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | URL do webhook deve ser https; http recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Payload snapshot enxuto (sem items) com campos definidos | TRANSCRICAO | [09:43] Diego, [09:52] Larissa |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega < 10 segundos | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB; erro se ultrapassar | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout da chamada HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Consistência: evento na mesma transação (rollback conjunto) | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Reuso dos padrões do projeto (AppError, Pino, error middleware, módulos, Zod, auth) | TRANSCRICAO | [09:30] Larissa |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência < 10 segundos (p95) | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Adoção pelos 3 clientes iniciais até o fim do trimestre | TRANSCRICAO | [09:00] Marcos, [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | 0 casos de status mudado sem evento registrado | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| PRD-FORA-01 | docs/PRD.md | Fora de escopo | Email como fallback quando webhook falha 3x seguidas (adiado) | TRANSCRICAO | [09:37] Marcos, [09:38] Larissa |
| PRD-FORA-02 | docs/PRD.md | Fora de escopo | Rate limiting de envio para o cliente (adiado) | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| PRD-FORA-03 | docs/PRD.md | Fora de escopo | Dashboard visual para o cliente (projeto separado do frontend) | TRANSCRICAO | [09:39] Marcos, [09:40] Larissa |
| PRD-FORA-04 | docs/PRD.md | Fora de escopo | Arquivamento de linhas entregues após 30 dias | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-05 | docs/PRD.md | Fora de escopo | Escala para múltiplos workers em paralelo (adiado) | TRANSCRICAO | [09:13] Diego |
| PRD-RISCO-01 | docs/PRD.md | Risco | Cliente offline por longos períodos (prob. média, impacto alto) | TRANSCRICAO | [09:15] Diego, [09:16] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | Vazamento de secret (prob. baixa, impacto alto) | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Duplicidade de entrega (prob. média, impacto médio) | TRANSCRICAO | [09:24] Diego, [09:25] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Inserção na outbox falhar e travar mudança de status | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Sobrecarga de envio (muitos eventos por minuto) | TRANSCRICAO | [09:38] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Prazo de 3 sprints com revisão de segurança da Sofia | TRANSCRICAO | [09:46] Larissa, [09:46] Sofia |
| RFC-META-01 | docs/RFC.md | Decisão | Revisores são os participantes da reunião | TRANSCRICAO | [09:00] Larissa (participantes) |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Notificação síncrona descartada (acopla latência do cliente à transação) | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado (overengineering, time pequeno) | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Trigger de banco descartado (MySQL sem listener nativo) | TRANSCRICAO | [09:09] Bruno, [09:09] Diego |
| RFC-ABERTO-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio (observar e decidir depois) | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| RFC-ABERTO-02 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers (ordering global) | TRANSCRICAO | [09:13] Diego |
| RFC-ABERTO-03 | docs/RFC.md | Questão em aberto | Endurecimento de permissões do CRUD de webhook | TRANSCRICAO | [09:37] Sofia |
| RFC-ABERTO-04 | docs/RFC.md | Questão em aberto | Email como fallback (adiado para próxima fase) | TRANSCRICAO | [09:37] Marcos, [09:38] Larissa |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox no MySQL (ADR-001) | TRANSCRICAO | [09:06] Diego, [09:08] Larissa |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado em polling de 2s (ADR-002) | TRANSCRICAO | [09:09] Diego, [09:11] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry com backoff e DLQ (ADR-003) | TRANSCRICAO | [09:15] Diego, [09:18] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256 com secret por endpoint (ADR-004) | TRANSCRICAO | [09:20] Sofia, [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | At-least-once com X-Event-Id (ADR-005) | TRANSCRICAO | [09:24] Diego, [09:26] Larissa |
| RFC-PROP-06 | docs/RFC.md | Decisão | Reuso dos padrões do projeto (ADR-006) | TRANSCRICAO | [09:30] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | Snapshot do payload na inserção (ADR-007) | TRANSCRICAO | [09:52] Larissa, [09:52] Diego |
| RFC-PROP-08 | docs/RFC.md | Decisão | UUID como identificador da outbox (ADR-008) | TRANSCRICAO | [09:51] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Evento criado na outbox dentro da transação de mudança de status | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro de eventos por status na inserção (economiza linha) | TRANSCRICAO | [09:34] Bruno, [09:34] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Worker processa em batch pequeno, ordenado por created_at | TRANSCRICAO | [09:08] Diego, [09:12] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Retry com backoff 1m/5m/30m/2h/12h (5 tentativas) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Replay manual via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego, [09:35] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks (criar webhook) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId= (listar) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id (editar) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id (remover) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret (rotacionar) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries (histórico) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:18] Diego, [09:35] Diego |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload JSON enxuto (event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents) | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-02 | docs/FDD.md | Contrato | Payload não envia items; cliente consulta GET /orders/:id | TRANSCRICAO | [09:43] Diego, [09:44] Bruno |
| FDD-HEADER-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| FDD-ERRO-01 | docs/FDD.md | Matriz de erros | Códigos de erro com prefixo WEBHOOK_ | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Matriz de erros | WEBHOOK_INVALID_URL (URL não https) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Matriz de erros | WEBHOOK_PAYLOAD_TOO_LARGE (payload > 64KB) | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| FDD-RESIL-01 | docs/FDD.md | Resiliência | Timeout da chamada HTTP de 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Resiliência | Retry com backoff exponencial (5 tentativas) | TRANSCRICAO | [09:15] Diego, [09:17] Diego |
| FDD-RESIL-03 | docs/FDD.md | Resiliência | DLQ com replay manual (fallback operacional) | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-04 | docs/FDD.md | Resiliência | Limite de payload 64KB; errar em vez de truncar | TRANSCRICAO | [09:23] Sofia, [09:24] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados Pino com redação de secrets | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | requestId/X-Request-Id para correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus estendido para chamar publishWebhookEvent na transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reuso de canTransition/shouldDebitStock/shouldReplenishStock e enum OrderStatus | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros WEBHOOK_* estendem AppError e classes existentes | CODIGO | src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Middleware de erro centralizado trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') para auth | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | buildApiRouter registra rota de webhooks; buildControllers instancia módulo | CODIGO | src/routes/index.ts, src/app.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Worker abre PrismaClient separado (mesmo banco) | CODIGO | src/config/database.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Novos modelos com id UUID @db.Char(36) | CODIGO | prisma/schema.prisma |
| FDD-INT-09 | docs/FDD.md | Integração | paginated/buildPagination reutilizado nas listagens | CODIGO | src/shared/http/response.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Schemas Zod validados com middleware validate | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-11 | docs/FDD.md | Integração | src/worker.ts como entry separada (referência: src/server.ts) | TRANSCRICAO | [09:11] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, transação atômica com mudança de status | TRANSCRICAO | [09:06] Diego, [09:08] Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Síncrono descartado (cliente lento trava transação) | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Redis Streams descartado (overengineering) | TRANSCRICAO | [09:07] Diego |
| ADR-001-COD | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Transação atual de changeStatus (orders, history, estoque) | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado em polling de 2s | TRANSCRICAO | [09:09] Diego, [09:11] Diego |
| ADR-002-ALT | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Trigger de banco descartado (MySQL sem listener nativo) | TRANSCRICAO | [09:09] Diego |
| ADR-002-ORD | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Ordering implícita por order_id; sem garantia global | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| ADR-002-COD | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | src/worker.ts e npm run worker (referência src/server.ts) | TRANSCRICAO | [09:11] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Retry com backoff 1m/5m/30m/2h/12h (5 tentativas) | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | 3 tentativas descartado (pouco, cliente com 2h de indisponibilidade) | TRANSCRICAO | [09:16] Diego |
| ADR-003-DLQ | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-REPLAY | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-SECRET | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Secret única por endpoint (não global) | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ROT | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-TLS | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | TLS obrigatório (https); http recusado | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego, [09:26] Larissa |
| ADR-005-ALT | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Exactly-once descartado (complexidade de coordenação) | TRANSCRICAO | [09:25] Diego |
| ADR-005-MKT | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Padrão de mercado (Stripe, GitHub) | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso dos padrões do projeto (AppError, Pino, error middleware, módulos, Zod, auth) | TRANSCRICAO | [09:30] Larissa |
| ADR-006-COD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo em src/modules/webhooks com estrutura padrão | TRANSCRICAO | [09:27] Bruno |
| ADR-006-COD2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Erros estendem AppError; prefixo WEBHOOK_ | CODIGO | src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts |
| ADR-006-COD3 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Logger Pino já no projeto | CODIGO | src/shared/logger/index.ts |
| ADR-006-COD4 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Error middleware centralizado trata AppError, Zod, Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD5 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | PrismaClient separado por processo | CODIGO | src/config/database.ts |
| ADR-006-ADMIN | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Replay de DLQ exige role ADMIN (requireRole) | TRANSCRICAO | [09:36] Sofia, [09:36] Larissa |
| ADR-006-CRUD | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | CRUD de configuração aceita qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Snapshot do payload renderizado na inserção | TRANSCRICAO | [09:52] Larissa, [09:52] Diego |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Guardar só order_id e renderizar na hora descartado | TRANSCRICAO | [09:52] Larissa |
| ADR-008 | docs/adrs/ADR-008-uuid-como-identificador-da-outbox.md | Decisão | UUID como identificador da outbox (padrão do projeto) | TRANSCRICAO | [09:51] Larissa |
| ADR-008-COD | docs/adrs/ADR-008-uuid-como-identificador-da-outbox.md | Decisão | Modelos existentes usam id UUID @db.Char(36) | CODIGO | prisma/schema.prisma |
| FDD-WORKER-01 | docs/FDD.md | Decisão | Worker em src/worker.ts, lógica em src/modules/webhooks/webhook.worker.ts | TRANSCRICAO | [09:28] Bruno |
| FDD-AUTH-01 | docs/FDD.md | Decisão | customerId passado no body/path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-FUNCAO-01 | docs/FDD.md | Decisão | Função publishWebhookEvent(tx, order, fromStatus, toStatus) recebe tx | TRANSCRICAO | [09:41] Bruno, [09:41] Diego |
| FDD-ORDERING-01 | docs/FDD.md | Restrição | Single-worker processa em ordem de created_at | TRANSCRICAO | [09:12] Diego |
| FDD-PRAZO-01 | docs/FDD.md | Dependência | Prazo de 3 sprints | TRANSCRICAO | [09:46] Larissa |
| FDD-SEG-01 | docs/FDD.md | Dependência | Revisão de segurança da Sofia (2 dias úteis) antes do deploy | TRANSCRICAO | [09:46] Sofia |
