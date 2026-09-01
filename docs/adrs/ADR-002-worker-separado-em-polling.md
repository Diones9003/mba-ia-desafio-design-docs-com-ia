# ADR-002 — Worker em processo separado em polling

## Status

Aceito.

## Contexto

Com o padrão Outbox definido (ADR-001), é preciso decidir **como** o worker lê a tabela `webhook_outbox` e dispara as chamadas HTTP. Duas questões foram levantadas na reunião:

1. **Mecanismo de leitura**: polling em loop vs. reatividade via trigger de banco.
2. **Localização do worker**: mesmo processo da API vs. processo separado.

O requisito de negócio é que a notificação chegue em menos de 10 segundos ("tempo real" para os clientes, [09:02] Marcos).

## Decisão

- **Polling em loop a cada 2 segundos**: o worker busca os eventos pendentes mais antigos, processa e marca. A latência mínima no pior caso é de 2 segundos, o que atende o requisito de "abaixo de 10 segundos".
- **Processo separado da API**: o worker roda como uma entry-point nova do projeto, `src/worker.ts`, com script `npm run worker`, análogo ao que já existe em `src/server.ts`. Ele conecta no **mesmo banco** e usa o **mesmo Prisma client** (instância separada, pois `PrismaClient` é por processo — ver `src/config/database.ts` e `src/server.ts`).
- **Ordering**: com um único worker, os eventos são processados em ordem de `created_at` do outbox, dando ordering implícita por `order_id`. Isso **não** é uma garantia de ordering global; é uma limitação conhecida. Escalar para múltiplos workers em paralelo (particionando por `order_id` ou com lock pessimista) é problema do futuro.

## Alternativas Consideradas

1. **Trigger de banco para reatividade** — descartada: MySQL não tem listener nativo tipo `NOTIFY/LISTEN` do Postgres. Trigger só executa SQL e não notifica processo externo; avisar o worker exigiria improviso (escrever em arquivo ou bater em endpoint), considerado esquisito.
2. **Worker dentro da mesma instância da API** — descartada: se a API reinicia, perde o worker.
3. **Múltiplos workers em paralelo** — adiada: quebraria a garantia de ordering; particionamento por `order_id` ou lock pessimista ficam para o futuro.

## Consequências

**Positivas:**

- Atende o requisito de latência (< 10s) com polling de 2s.
- Isolamento de ciclo de vida: reinício da API não derruba o worker.
- Reutiliza a mesma stack (MySQL + Prisma), sem nova infraestrutura.
- Single-worker dá ordering implícito por `order_id`.

**Negativas / trade-offs:**

- Latência mínima de ~2s no pior caso (aceita na reunião).
- Sem garantia de ordering global; escalar para múltiplos workers exigirá particionamento ou lock.
- Polling gera leituras periódicas no banco (batch pequeno, mitigado por índice em status e `created_at`).
