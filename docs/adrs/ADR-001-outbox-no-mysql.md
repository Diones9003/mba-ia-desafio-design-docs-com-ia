# ADR-001 — Padrão Outbox no MySQL

## Status

Aceito.

## Contexto

O Order Management System (OMS) precisa notificar clientes B2B em tempo real quando o status de um pedido muda. Hoje a mudança de status acontece dentro de uma transação SQL que atualiza a tabela `orders`, insere em `order_status_history` e, dependendo da transição, debita ou repõe `stock_quantity` dos produtos (ver `src/modules/orders/order.service.ts`, método `changeStatus`).

A primeira alternativa discutida foi disparar a notificação de forma **síncrona** dentro do `OrderService`, fazendo uma chamada HTTP ao cliente no meio da transação. Isso foi descartado porque:

- A transação de mudança de status já é pesada; um cliente lento travaria a mudança de status de outros pedidos.
- Se o cliente estiver fora do ar, não há como dar rollback na mudança de status por causa de uma notificação.

A segunda alternativa discutida foi usar **Redis Streams** (ou fila similar). Foi descartada por exigir subir infraestrutura adicional (Redis Cluster), considerada *overengineering* para um time pequeno.

## Decisão

Adotar o **padrão Outbox no MySQL existente**. Quando o status do pedido muda, dentro da **mesma transação SQL** que atualiza `orders` e `order_status_history`, também é inserida uma linha na tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP.

Garantias do padrão:

- Se a transação principal fez commit, o evento foi registrado.
- Se a transação deu rollback, o evento some junto.
- Não existe inconsistência possível entre a mudança de status e o registro do evento.

A tabela `webhook_outbox` terá índice nos campos de status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`. O worker lê apenas os eventos pendentes em batch pequeno, processa e marca como entregue. Linhas entregues são arquivadas após 30 dias, fora do escopo desta feature.

## Alternativas Consideradas

1. **Notificação síncrona dentro do `OrderService`** — descartada: acopla a latência do cliente à transação de status, e não há como tratar cliente fora do ar sem comprometer a mudança de status.
2. **Redis Streams / fila externa** — descartada: exige subir e operar infraestrutura adicional (Redis Cluster), considerada *overengineering* para o tamanho do time.
3. **Trigger de banco** — descartada (ver ADR-002): MySQL não tem listener nativo tipo `NOTIFY/LISTEN` do Postgres; trigger só executa SQL e não notifica processo externo.

## Consequências

**Positivas:**

- Garantia de consistência entre a mudança de status e o registro do evento (transação atômica).
- Não adiciona latência à transação de status (a chamada HTTP acontece fora dela).
- Reutiliza o MySQL já existente, sem nova infraestrutura.
- Permite retry e DLQ de forma desacoplada (ver ADR-003).

**Negativas / trade-offs:**

- A entrega do evento não é imediata: depende do polling do worker (latência mínima de ~2s no pior caso, ver ADR-002).
- A tabela `webhook_outbox` cresce e precisa de política de arquivamento (adiada para fora do escopo).
- Não há garantia de ordering global quando houver múltiplos workers (limitação conhecida, ver ADR-002).
