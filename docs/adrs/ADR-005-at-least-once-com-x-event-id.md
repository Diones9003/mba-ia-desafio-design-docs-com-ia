# ADR-005 — Garantia at-least-once com X-Event-Id

## Status

Aceito.

## Contexto

Com o padrão Outbox e o retry (ADRs 001 e 003), pode acontecer de o cliente receber o mesmo evento mais de uma vez (por exemplo, se o worker processa, o cliente responde, mas a resposta se perde e o retry reenvia). É preciso definir a garantia de entrega e como o cliente diferencia eventos duplicados.

## Decisão

- **Garantia at-least-once**: o cliente pode receber o mesmo evento duas vezes e deve estar preparado para isso.
- **`X-Event-Id`**: o OMS envia um `event_id` no header `X-Event-Id`, um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente receber o mesmo evento duas vezes, ele deduplica pelo `event_id` do lado dele.

Garantir exactly-once exigiria coordenação dos dois lados e é muito mais complexo. At-least-once com `event_id` resolve a grande maioria dos casos e é o padrão de mercado (Stripe e GitHub fazem assim).

## Alternativas Consideradas

1. **Exactly-once** — descartada: exigiria coordenação entre OMS e cliente (acknowledgement, idempotência transacional dos dois lados), complexidade desproporcional.
2. **Sem identificador de evento** — descartada: o cliente não teria como deduplicar entregas duplicadas.

## Consequências

**Positivas:**

- Simplicidade e alinhamento com o padrão de mercado.
- O cliente consegue deduplicar de forma determinística via `X-Event-Id`.
- Compatível com o retry/backoff do ADR-003.

**Negativas / trade-offs:**

- Joga responsabilidade de deduplicação para o cliente.
- O cliente precisa armazenar/processar `event_id` para idempotência.
- Não há garantia de entrega exatamente uma vez.
