# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**.

Cada decisão arquitetural relevante é registrada em um arquivo individual, nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`.

## Índice

| ADR | Título | Status |
|-----|--------|--------|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado em polling | Aceito |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Política de retry com backoff e DLQ | Aceito |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | Autenticação HMAC-SHA256 com secret por endpoint | Aceito |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Garantia at-least-once com X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado na inserção | Aceito |
| [ADR-008](ADR-008-uuid-como-identificador-da-outbox.md) | UUID como identificador da outbox | Aceito |
