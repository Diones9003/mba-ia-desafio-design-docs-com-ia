# MBA IA — Desafio: Da Reunião ao Documento — Design Docs Gerados por IA

## Sobre o desafio

Este desafio consiste em transformar a transcrição de uma reunião técnica em um pacote completo de design docs para uma nova feature do Order Management System (OMS): o **Sistema de Webhooks de Notificação de Pedidos**. A decisão técnica já foi tomada em uma reunião entre tech lead, PM, engenheiros e segurança, mas nada foi registrado além da transcrição (`TRANSCRICAO.md`).

A entrega é **puramente documental**: produzir PRD, RFC, FDD, ADRs, Tracker e README a partir da transcrição e do código existente, sem alterar nenhum arquivo de código (`src/`, `prisma/`, `tests/`). O princípio central é a **rastreabilidade**: toda informação registrada nos documentos deve ter origem identificável na transcrição ou no código, e nada pode ser inventado. O Tracker é a ferramenta que garante essa integridade contra alucinações da IA.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|------------|-------|
| **Abacus AI Agent (agente orquestrador)** | Ferramenta principal de produção. Leitura e análise do código, leitura da transcrição, estruturação dos documentos, geração do conteúdo final e revisão crítica. Atuou como "maestro" do processo: definiu o que fazer, formulou os prompts, revisou e refinou o resultado. |
| **Análise de código (via agente)** | Exploração da estrutura do repositório (`src/`, `prisma/`, `tests/`), identificação de padrões (módulos, erros, logging, auth, validação) e dos pontos de integração da feature. |
| **GitHub (via agente)** | Clonagem do fork, versionamento e push final do pacote de documentos. |

> Nota: o processo foi conduzido integralmente pelo agente de IA, que combinou leitura de código, análise da transcrição e geração de documentos em um fluxo único e iterativo.

## Workflow adotado

A produção seguiu a ordem sugerida no enunciado, com a IA como ferramenta principal em cada etapa:

1. **Fork e setup**: o repositório base foi forkado para `https://github.com/Diones9003/mba-ia-desafio-design-docs-com-ia` e clonado localmente.
2. **Contextualização**: a IA leu a `TRANSCRICAO.md` completa e explorou o código (schema Prisma, `OrderService.changeStatus`, máquina de estados, erros, middlewares, logger, rotas, testes) para entender estrutura, padrões e o que a feature precisa endereçar.
3. **ADRs primeiro**: as decisões principais foram identificadas e registradas em 8 ADRs (as 6 decisões principais + 2 decisões secundárias). As decisões formam o esqueleto do "como implementar".
4. **RFC**: a proposta técnica foi consolidada em cima das decisões, com alternativas descartadas e questões em aberto, referenciando os ADRs.
5. **FDD**: o desenho técnico detalhado foi construído sobre as decisões, incluindo a seção obrigatória "Integração com o sistema existente".
6. **PRD**: produzido por último entre os grandes documentos, como consolidação de alto nível.
7. **Tracker**: montado varrendo os documentos prontos, mapeando cada item à origem na transcrição ou no código.
8. **README**: documentado por último, quando o processo já estava completo.
9. **Revisão final**: passagem pela checklist de critérios de aceite item por item antes do push.

## Prompts customizados

Abaixo, dois prompts relevantes usados (e adaptados) durante o processo. O princípio foi sempre **dirigir a IA com contexto específico**, nunca pedidos genéricos.

**Prompt 1 — Filtragem do que NÃO entra (usado na análise da transcrição):**

```text
Analise a transcrição da reunião técnica sobre o Sistema de Webhooks de Notificação de Pedidos.
Identifique e separe em categorias:
1) Decisões fechadas (com o falante e o timestamp de cada uma)
2) Requisitos funcionais explícitos
3) Restrições e requisitos não funcionais
4) Ganchos com o código existente (mencione o arquivo/módulo quando possível)
5) Pontos EXPLICITAMENTE descartados (não devem virar requisito)
6) Pontos ADIADOS para fases futuras (devem aparecer como "fora de escopo" ou "questão em aberto")
7) Detalhes técnicos secundários
Para cada item, registre o timestamp [hh:mm] e o nome do falante. Não invente nada que não esteja na transcrição.
```

**Prompt 2 — Geração do FDD com integração ao código (usado na produção do FDD):**

```text
Produza o FDD (Feature Design Document) do Sistema de Webhooks de Notificação de Pedidos.
O FDD é o documento mais técnico e deve ser acionável para um desenvolvedor começar a codar.
Inclua: contexto e motivação técnica, objetivos técnicos, escopo e exclusões, fluxos detalhados
(criação do evento na outbox, processamento pelo worker, retry, DLQ), contratos públicos
(endpoints HTTP com payloads de exemplo, headers, status codes), matriz de erros com códigos
no padrão WEBHOOK_*, estratégias de resiliência, observabilidade, dependências, critérios de
aceite técnicos e riscos.
Seção obrigatória "Integração com o sistema existente": nomeie pelo menos 4 caminhos de arquivo
REAIS do código base (ex.: src/modules/orders/order.service.ts) e descreva como o módulo de
webhooks se integra com cada um. Use apenas arquivos que existem de fato no repositório.
Toda informação deve ser rastreável à transcrição ou ao código. Não invente requisitos.
```

## Iterações e ajustes

O processo demandou **3 ciclos principais de geração, revisão crítica e ajuste**, conforme esperado no enunciado:

1. **Primeira iteração — ADRs**: a primeira versão dos ADRs saiu genérica, sem amarrar cada decisão ao timestamp e ao falante da transcrição. **Ajuste**: reescrevi cada ADR ancorando a decisão ao trecho exato da reunião (ex.: backoff 1m/5m/30m/2h/12h em [09:17] Diego) e ao código (ex.: `changeStatus` em `src/modules/orders/order.service.ts`), e adicionei as alternativas reais discutidas com o trade-off que motivou o descarte.

2. **Segunda iteração — FDD**: a primeira versão do FDD listava endpoints e fluxos, mas a seção "Integração com o sistema existente" citava caminhos de arquivo de forma vaga e alguns códigos de erro não seguiam o padrão `WEBHOOK_*`. **Ajuste**: verifiquei cada caminho de arquivo contra a árvore real do repositório, nomeei 10 caminhos reais e padronizei a matriz de erros com o prefixo `WEBHOOK_`, além de detalhar os fluxos de outbox, worker, retry e DLQ.

3. **Terceira iteração — Tracker e consistência**: ao montar o Tracker, identifiquei itens que não tinham origem clara na transcrição ou no código. **Ajuste**: removi/ajustei esses itens e garanti que cada linha tivesse timestamp válido `[hh:mm] Nome` (para `TRANSCRICAO`) ou caminho de arquivo real (para `CODIGO`), atingindo a cobertura exigida.

O resultado final foi validado contra a checklist de critérios de aceite item por item antes do push.

## Como navegar a entrega

**Arquivos entregues:**

```
.
├── README.md                              (este arquivo — processo de produção)
├── TRANSCRICAO.md                         (fonte primária — não alterado)
├── docs/
│   ├── PRD.md                             (produto/negócio)
│   ├── RFC.md                             (proposta técnica para revisão)
│   ├── FDD.md                             (especificação de implementação)
│   ├── TRACKER.md                         (rastreabilidade de cada item)
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-em-polling.md
│       ├── ADR-003-retry-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│       ├── ADR-005-at-least-once-com-x-event-id.md
│       ├── ADR-006-reuso-dos-padroes-do-projeto.md
│       ├── ADR-007-snapshot-do-payload-na-insercao.md
│       ├── ADR-008-uuid-como-identificador-da-outbox.md
│       └── README.md                      (índice dos ADRs)
├── src/                                   (não alterado)
├── prisma/                                (não alterado)
├── tests/                                 (não alterado)
└── ... (demais arquivos do boilerplate)
```

**Ordem sugerida de leitura:**

1. **`TRANSCRICAO.md`** — a fonte primária; leia primeiro para entender o que foi decidido.
2. **`docs/adrs/`** — as decisões arquiteturais fechadas (o esqueleto do "como implementar").
3. **`docs/RFC.md`** — a proposta técnica consolidada, com alternativas e questões em aberto.
4. **`docs/FDD.md`** — o detalhamento de implementação (fluxos, contratos, erros, integração).
5. **`docs/PRD.md`** — a visão de produto/negócio (por que e o quê).
6. **`docs/TRACKER.md`** — a referência cruzada de onde veio cada item (use para validar a integridade).

---

*Enunciado original do desafio disponível no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).*
