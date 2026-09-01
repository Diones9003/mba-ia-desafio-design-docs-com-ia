# ADR-007 — Snapshot do payload renderizado na inserção

## Status

Aceito.

## Contexto

Ao inserir um evento na `webhook_outbox`, é preciso decidir o que a tabela guarda: apenas o `order_id` (e renderizar o payload na hora do envio) ou o payload já renderizado (snapshot) no momento da inserção.

## Decisão

Guardar o **payload renderizado (snapshot) no momento da inserção** na outbox. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Isso evita casos esquisitos em que o payload renderizado na hora do envio não corresponderia ao estado no momento da transição.

## Alternativas Consideradas

1. **Guardar apenas `order_id` e renderizar na hora do envio** — descartada: se o pedido mudar entre a inserção e o envio, o payload refletiria um estado diferente do momento da transição, gerando inconsistência.

## Consequências

**Positivas:**

- O evento é fiel ao estado no momento da transição de status.
- O worker não precisa consultar o pedido na hora do envio (menos acoplamento e menos leituras).

**Negativas / trade-offs:**

- O payload ocupa espaço na outbox (mitigado pelo limite de 64KB e pelo payload enxuto, ver FDD).
- Se o formato do payload evoluir, eventos antigos na outbox/DLQ mantêm o formato antigo (snapshot).
