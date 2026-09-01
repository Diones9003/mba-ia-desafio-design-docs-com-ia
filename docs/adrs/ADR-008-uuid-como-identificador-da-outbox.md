# ADR-008 — UUID como identificador da outbox

## Status

Aceito.

## Contexto

Ao modelar a `webhook_outbox`, é preciso escolher o tipo de identificador da linha do evento: id auto incremental ou UUID. A mesma pergunta se aplica aos demais modelos novos do módulo de webhooks.

## Decisão

Usar **UUID** como identificador dos registros da outbox (e dos demais modelos do módulo de webhooks), seguindo o padrão do restante do projeto. Todos os modelos existentes usam `id String @id @default(uuid()) @db.Char(36)` (ver `prisma/schema.prisma`). O `event_id` enviado no header `X-Event-Id` (ADR-005) é o UUID gerado quando o evento entra na outbox.

## Alternativas Consideradas

1. **Id auto incremental** — descartada: foge do padrão do projeto, onde tudo é UUID.

## Consequências

**Positivas:**

- Consistência com o padrão do projeto.
- UUIDs não expõem volume/sequência de eventos e são adequados para identificadores distribuídos.

**Negativas / trade-offs:**

- UUIDs são maiores que inteiros e levemente menos eficientes em indexação (irrelevante para o volume esperado).
- Ordenação por `created_at` (não pelo id) é necessária para garantir a ordem de processamento (ver ADR-002).
