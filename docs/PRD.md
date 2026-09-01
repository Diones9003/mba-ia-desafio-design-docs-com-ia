# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O Order Management System (OMS) é uma plataforma B2B que gerencia pedidos de clientes. Três clientes — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — solicitaram formalmente serem notificados em tempo real quando o status dos pedidos deles muda na plataforma. Hoje eles fazem polling no `GET /orders` periodicamente, o que torna a integração lenta e cara.

Esta feature entrega um **sistema de webhooks de notificação de pedidos (outbound)**: quando o status de um pedido muda, o OMS notifica automaticamente o endpoint do cliente. A solução usa o padrão Outbox no MySQL, um worker separado em polling, assinatura HMAC-SHA256 e garantia at-least-once, conforme as decisões registradas nos ADRs.

## Problema e motivação

- **Problema**: clientes B2B dependem de polling manual no `GET /orders` para detectar mudanças de status, o que é lento, caro e não escala.
- **Motivação**: os três clientes pediram notificação em tempo real. A Atlas chegou a sugerir que, se a entrega não ocorrer até o fim do trimestre, pode migrar para o concorrente. Para os clientes, "tempo real" é qualquer coisa **abaixo de 10 segundos**.
- **Direção**: os webhooks são **outbound** (só saem do OMS para os clientes; os clientes não enviam eventos para o OMS).

## Público-alvo e cenários de uso

**Público-alvo:**

- Clientes B2B do OMS que integram seus sistemas com a plataforma (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo).
- Equipe de engenharia do OMS (implementação e operação).
- Equipe de segurança (revisão de autenticação e assinatura).

**Cenários de uso:**

1. Um cliente quer ser notificado quando um pedido dele muda para `SHIPPED` ou `DELIVERED`, para disparar a logística/entrega no sistema dele.
2. Um cliente quer acompanhar o histórico de entregas de webhooks (sucesso/falha, payload, response, tempo de resposta) para depurar integrações.
3. Um cliente precisa rotacionar a secret do webhook por segurança, mantendo a antiga válida por 24h para migração.
4. Um operador (ADMIN) reprocessa manualmente um evento que falhou permanentemente (DLQ).

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|----------|---------|------|
| Notificar clientes em tempo real | Latência entre a mudança de status e a entrega do evento | **< 10 segundos** (p95) |
| Garantir entrega confiável | Taxa de eventos entregues com sucesso (após retries) | **≥ 99%** dos eventos entregues ou em DLQ rastreável |
| Reduzir dependência de polling dos clientes | Adoção dos webhooks pelos 3 clientes iniciais | **3/3 clientes** integrados via webhook até o fim do trimestre |
| Manter consistência de dados | Eventos registrados na mesma transação da mudança de status | **0** casos de status mudado sem evento registrado |

## Escopo

### Incluso

- CRUD de configuração de webhooks (criar, listar, editar, remover).
- Rotação de secret com grace period de 24h.
- Outbox no MySQL, worker separado em polling (2s), retry com backoff e DLQ.
- Histórico de entregas (`deliveries`).
- Filtro de eventos por status na inserção da outbox.
- Assinatura HMAC-SHA256, secret por endpoint, TLS obrigatório.
- Garantia at-least-once com `X-Event-Id`.

### Fora de escopo

1. **Email como fallback** quando o webhook falha 3 vezes seguidas — **adiado** para a próxima fase, depois de medir o impacto ([09:37] Marcos, [09:38] Larissa).
2. **Rate limiting de envio para o cliente** — **adiado**; a equipe observa e decide depois se vira problema ([09:38] Diego, [09:39] Larissa).
3. **Dashboard visual** para o cliente gerenciar webhooks — **fora de escopo**; é projeto separado do time de frontend ([09:39] Marcos, [09:40] Larissa).
4. **Arquivamento de linhas entregues** da outbox após 30 dias — **fora de escopo** desta feature ([09:08] Diego).
5. **Escala para múltiplos workers** em paralelo — **adiado**; hoje é single-worker ([09:13] Diego).

## Requisitos funcionais

| ID | Requisito |
|----|-----------|
| FR-01 | O cliente pode **cadastrar** um webhook informando `url`, lista de status desejados e `customerId`; a `secret` é gerada pelo OMS e devolvida na criação. |
| FR-02 | O cliente pode **listar** os webhooks de um `customerId`. |
| FR-03 | O cliente pode **editar** um webhook (url, statuses, ativo). |
| FR-04 | O cliente pode **remover** um webhook. |
| FR-05 | O cliente pode **rotacionar** a secret de um webhook; a antiga permanece válida por 24h (grace period). |
| FR-06 | O cliente pode consultar o **histórico de entregas** de um webhook (sucesso/falha, payload, response, tempo de resposta). |
| FR-07 | O sistema **filtra eventos por status** na inserção da outbox: se nenhum webhook do cliente quer aquele status, o evento não é inserido. |
| FR-08 | Quando o status de um pedido muda, o evento é registrado na `webhook_outbox` **na mesma transação** da mudança de status. |
| FR-09 | Um **worker separado** processa os eventos pendentes em polling de 2s e dispara a chamada HTTP ao cliente. |
| FR-10 | O envio inclui headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. |
| FR-11 | O sistema aplica **retry com backoff exponencial** (5 tentativas: 1m/5m/30m/2h/12h) e move falhas permanentes para a **DLQ**. |
| FR-12 | Um operador com role **ADMIN** pode **reprocessar** um evento da DLQ via endpoint de replay, com registro de auditoria. |
| FR-13 | A URL do webhook deve ser **https**; `http` é recusado com erro de validação. |
| FR-14 | O payload do evento é um **snapshot renderizado na inserção**, enxuto (sem `items`), com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`. |

## Requisitos não funcionais

| ID | Requisito |
|----|-----------|
| NFR-01 | **Latência**: entrega do evento em menos de 10 segundos (p95). |
| NFR-02 | **Segurança**: assinatura HMAC-SHA256 sobre o corpo; secret única por endpoint; TLS obrigatório. |
| NFR-03 | **Confiabilidade**: garantia at-least-once; deduplicação via `X-Event-Id` do lado do cliente. |
| NFR-04 | **Limite de payload**: eventos acima de 64KB **erram** (não truncam). |
| NFR-05 | **Timeout**: chamada HTTP do worker com timeout de 10 segundos. |
| NFR-06 | **Consistência**: evento registrado na mesma transação da mudança de status (rollback conjunto). |
| NFR-07 | **Observabilidade**: métricas, logs estruturados (Pino) e tracing do ciclo de vida do evento. |
| NFR-08 | **Consistência de código**: reuso dos padrões do projeto (AppError, Pino, error middleware, módulos, Zod, auth). |

## Decisões e trade-offs principais

- **Outbox no MySQL** em vez de notificação síncrona ou Redis Streams: garante consistência transacional sem nova infraestrutura; trade-off é a latência mínima de ~2s do polling (ADR-001, ADR-002).
- **Worker separado em polling de 2s**: atende o requisito de <10s; trade-off é a ausência de ordering global quando houver múltiplos workers (ADR-002).
- **Retry com backoff (5 tentativas) + DLQ**: cobre janelas de indisponibilidade de ~15h; trade-off é que um evento pode demorar até ~15h para ser considerado falho (ADR-003).
- **HMAC-SHA256 com secret por endpoint**: compartimenta risco de vazamento; trade-off é a responsabilidade de verificação recair sobre o cliente (ADR-004).
- **At-least-once com X-Event-Id**: simplicidade e padrão de mercado; trade-off é o cliente precisar deduplicar (ADR-005).
- **Snapshot do payload na inserção**: fidelidade ao estado no momento da transição; trade-off é espaço ocupado na outbox (ADR-007).

## Dependências

- **Banco MySQL existente** via Prisma (novos modelos: `WebhookConfig`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDelivery`).
- **Stack existente**: Node.js ≥ 20, TypeScript, Express, Prisma 5.22, Zod, Pino.
- **Módulo de pedidos**: `OrderService.changeStatus` precisa chamar `publishWebhookEvent` dentro da transação.
- **Infraestrutura**: nenhuma nova dependência externa obrigatória (HMAC via `crypto` nativo; HTTP via `fetch`/`undici` ou `axios`).
- **Prazo**: 3 sprints, com revisão de segurança da Sofia antes do deploy.

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Cliente offline por longos períodos | Média | Alto | Retry com backoff (~15h) + DLQ + replay manual |
| Vazamento de secret | Baixa | Alto | Secret por endpoint, rotação com grace period de 24h, redação em logs |
| Duplicidade de entrega | Média | Médio | At-least-once documentado; cliente deduplica via `X-Event-Id` |
| Inserção na outbox falhar e travar mudança de status | Baixa | Alto | Transação atômica; teste de rollback obrigatório |
| Sobrecarga de envio (muitos eventos por minuto) | Média | Médio | Observar; rate limiting adiado (questão em aberto) |

## Critérios de aceitação

- [ ] Cliente consegue cadastrar, listar, editar, remover e rotacionar webhooks via API.
- [ ] Mudança de status gera evento na outbox na mesma transação; rollback se a inserção falhar.
- [ ] Filtro de eventos por status aplicado na inserção.
- [ ] Evento entregue em <10 segundos (p95) via worker em polling de 2s.
- [ ] Payload assinado com HMAC-SHA256; headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` presentes.
- [ ] Retry com backoff (5 tentativas) e DLQ; replay manual exige role ADMIN com auditoria.
- [ ] URL `http` recusada; payload > 64KB erra.
- [ ] Histórico de entregas disponível.
- [ ] Códigos de erro com prefixo `WEBHOOK_`; erros tratados pelo middleware centralizado.

## Estratégia de testes e validação

- **Testes unitários**: `publishWebhookEvent` (filtro por status, snapshot, inserção na outbox), cálculo de backoff, geração de HMAC, validação de URL https.
- **Testes de integração**: `OrderService.changeStatus` gera evento na outbox na mesma transação; rollback quando a inserção falha.
- **Testes ponta a ponta** (padrão dos testes existentes em `tests/`): criar webhook, mudar status, verificar entrega, retry, DLQ, replay, validação de URL, HMAC.
- **Revisão de segurança** (Sofia): HMAC, geração/rotação de secret, redação em logs — pelo menos 2 dias úteis antes do deploy.
- **Validação com clientes**: documentação no portal de desenvolvedor (Marcos) e integração piloto com os 3 clientes.
