# ADR-003 — Política de retry com backoff e DLQ

## Status

Aceito.

## Contexto

Quando o cliente está offline ou o endpoint de webhook falha, o evento não pode ser perdido. É preciso definir a política de retry e o que acontece quando o evento falha de forma permanente. Também foi discutido se a DLQ (dead letter queue) fica em tabela separada ou se o evento é apenas marcado como `failed` na própria outbox.

## Decisão

- **Retry com backoff exponencial**: 5 tentativas no total, com progressão de **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas**. Isso cobre uma janela de quase 15 horas entre a primeira falha e a última tentativa.
- **DLQ em tabela separada** (`webhook_dead_letter`): quando as 5 tentativas se esgotam, o evento é movido para a DLQ com a payload, o motivo da falha e o timestamp. Isso mantém a leitura da outbox principal limpa e serve como evidência para debug e reprocessamento.
- **Reprocessamento manual via endpoint admin**: `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Exige role `ADMIN` (ver ADR-006) e registra quem fez o replay para auditoria.

## Alternativas Consideradas

1. **Retry indefinido com backoff** — descartada: evento ficaria pendurado para sempre se o cliente sumir.
2. **3 tentativas** — descartada: considerado pouco. Se o cliente tiver indisponibilidade de manhã, 3 tentativas em ~30 minutos matariam o evento; já houve cliente com indisponibilidade de 2 horas em manutenção planejada.
3. **Marcar como `failed` na própria outbox** — descartada: a leitura da outbox principal fica mais suja e perde-se a separação clara entre "a processar" e "falhou permanentemente".

## Consequências

**Positivas:**

- Cobre janelas de indisponibilidade de até ~15 horas.
- DLQ separada mantém a outbox principal limpa e facilita debug/reprocessamento.
- Replay manual com auditoria dá controle operacional.

**Negativas / trade-offs:**

- 5 tentativas ao longo de ~15h significa que um evento pode demorar até ~15h para ser considerado falho permanentemente.
- Requer operação manual (endpoint admin) para reprocessar DLQ; não há automação de replay.
- A DLQ cresce e precisa de política de retenção (não definida nesta fase).
