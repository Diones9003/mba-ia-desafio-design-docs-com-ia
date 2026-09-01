# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint

## Status

Aceito.

## Contexto

Os webhooks expõem eventos com dados de pedidos para endpoints fora da infraestrutura do OMS. O cliente precisa conseguir validar que a requisição veio realmente do OMS e que ninguém adulterou o payload no meio do caminho. Também é preciso definir como as secrets são gerenciadas e rotacionadas.

## Decisão

- **HMAC-SHA256 sobre o corpo do request**: o OMS assina o payload com uma secret compartilhada entre o OMS e o cliente e envia a assinatura no header `X-Signature`. O cliente verifica do lado dele. HMAC-SHA256 é o padrão de mercado.
- **Secret única por endpoint de webhook**: não é uma secret global da plataforma. Se uma secret vazar, apenas aquele endpoint é comprometido.
- **Rotação de secret com grace period de 24h**: o cliente pode pedir uma nova secret pela API. Quando rotaciona, a antiga permanece válida por 24 horas em paralelo, para o cliente ter tempo de migrar os sistemas dele. Depois disso, a antiga morre.
- **TLS obrigatório**: a URL do webhook deve ser `https`. Se o cliente cadastrar `http`, a requisição é recusada com erro de validação (validação no schema Zod, não decisão arquitetural separada).

## Alternativas Consideradas

1. **Sem assinatura (payload em texto puro)** — descartada: o cliente não conseguiria validar autenticidade nem integridade.
2. **Secret global da plataforma** — descartada: se uma secret vazar, vaza tudo. Secret por endpoint limita o impacto.
3. **Sem rotação de secret** — descartada: já houve cliente que vazou secret em log de aplicação; a rotação com grace period de 24h permite migração segura.

## Consequências

**Positivas:**

- Autenticidade e integridade do payload verificáveis pelo cliente.
- Compartimentação de risco: vazamento de uma secret afeta apenas um endpoint.
- Rotação com grace period de 24h permite migração sem downtime.

**Negativas / trade-offs:**

- Responsabilidade de verificação recai sobre o cliente (precisa implementar HMAC do lado dele).
- Gerenciamento de múltiplas secrets (uma por endpoint) e de janelas de rotação em paralelo.
- Requer armazenamento seguro das secrets no banco (não em texto puro em logs).
