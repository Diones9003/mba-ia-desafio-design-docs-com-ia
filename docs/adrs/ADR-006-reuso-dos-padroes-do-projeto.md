# ADR-006 — Reuso dos padrões existentes do projeto

## Status

Aceito.

## Contexto

O OMS já tem padrões consolidados de arquitetura, tratamento de erros, logging, validação e autenticação. A feature de webhooks deve seguir esses padrões para manter consistência e reduzir manutenção. A decisão é reutilizar ao máximo o que já existe em vez de introduzir convenções novas.

## Decisão

O módulo de webhooks segue os padrões existentes do projeto:

- **Estrutura de módulo**: cada domínio é um módulo em `src/modules` com `controller`, `service`, `repository`, `routes` e `schemas`. Webhook segue igual, em `src/modules/webhooks`. O worker fica em `src/worker.ts` (entry separada) e a lógica de processamento em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`).
- **Erros**: reutiliza a classe `AppError` (`src/shared/errors/app-error.ts`) e as classes específicas (`src/shared/errors/http-errors.ts`), com códigos de erro no padrão existente. Para webhook, prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).
- **Logger**: Pino (`src/shared/logger/index.ts`), já usado no projeto inteiro. Nada novo.
- **Middleware de erro centralizado**: `src/middlewares/error.middleware.ts` já trata `AppError`, Zod e Prisma; captura os erros de webhook sem mudança.
- **Validação**: schemas Zod com o middleware `validate` (`src/middlewares/validate.middleware.ts`).
- **Autenticação**: `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts`). O CRUD de configuração de webhook aceita qualquer role autenticada; o replay de DLQ exige role `ADMIN`.
- **PrismaClient**: instância separada por processo (mesmo banco, mesma `DATABASE_URL`), conforme `src/config/database.ts`.

## Alternativas Consideradas

1. **Criar convenções novas para o módulo de webhooks** — descartada: aumentaria a superfície de manutenção e quebraria a consistência com o restante do projeto.
2. **Injetar o repository inteiro de webhook no `OrderService`** — descartada: a reunião preferiu uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação atual, sem acoplar o service (ver FDD).

## Consequências

**Positivas:**

- Consistência com o restante da codebase.
- Reuso de infraestrutura já testada (erros, logging, validação, auth).
- Menor esforço de manutenção e revisão de segurança.

**Negativas / trade-offs:**

- O módulo fica acoplado às convenções atuais do projeto (mudanças futuras nos padrões afetam o módulo).
- A função `publishWebhookEvent` precisa ser chamada explicitamente dentro da transação do `OrderService`, exigindo cuidado para não esquecer a chamada em novos fluxos de mudança de status.
