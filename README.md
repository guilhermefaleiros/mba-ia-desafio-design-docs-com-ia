# Da Reunião ao Documento: Design Docs Gerados por IA

> **Desafio:** MBA em Inteligência Artificial — Design Docs com IA
> **Aluno:** Guilherme Faleiros
> **Data de entrega:** 2026-07-14

---

## Sobre o desafio

Este repositório contém o pacote de design docs produzido para a feature **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (OMS). A tarefa foi transformar a transcrição de uma reunião técnica de 55 minutos em um conjunto completo de documentos de engenharia de software, usando IA como ferramenta principal de produção.

O cenário é realista: uma empresa que opera um OMS em produção decide construir uma nova feature, mas a única coisa registrada da decisão técnica é a transcrição da call. Meu papel foi atuar como "maestro" — definir o que precisava ser documentado, formular prompts dirigidos, revisar criticamente o que a IA gerou e refinar até obter documentos consistentes, acionáveis e rastreáveis.

O pacote cobre 7 níveis de documentação: PRD (produto), RFC (proposta técnica), FDD (implementação), 7 ADRs (decisões arquiteturais), Tracker (rastreabilidade) e este README (processo de produção).

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|-----------|-------|
| **Claude Code (DeepSeek V4 Pro)** | Ferramenta principal. Usei o Claude Code para ler o código fonte, analisar a transcrição, estruturar e redigir todos os documentos, e iterar sobre revisões. O modelo foi usado em sessão interativa com acesso ao repositório completo. |
| **Claude Agent (subagentes)** | Usei subagentes do Claude Code para exploração simultânea de múltiplos arquivos do código fonte, acelerando o mapeamento dos padrões do projeto (estrutura modular, classes de erro, middlewares, logger). |

A escolha de usar o Claude Code diretamente (em vez de prompts manuais no ChatGPT ou Cursor) foi motivada pelo acesso completo ao código fonte e à transcrição em uma única sessão, permitindo que a IA fizesse referências cruzadas entre a transcrição e o código sem eu precisar copiar e colar trechos.

---

## Workflow adotado

### Ordem de produção

Segui a ordem sugerida no enunciado do desafio, com uma adaptação: escrevi os ADRs e o Tracker em paralelo aos demais documentos.

1. **Contextualização com IA (30 min):** forneci ao Claude Code acesso ao repositório completo. Pedi uma exploração sistemática: listar todos os arquivos, ler a transcrição, mapear a estrutura modular, identificar classes de erro, middlewares, logger, padrão de rotas. A IA gerou um mapa mental do código que usei como referência.

2. **ADRs primeiro (1h):** produzi os 7 ADRs antes de qualquer outro documento. As decisões formam o esqueleto do "como implementar". Cada ADR foi escrito com referências explícitas à transcrição e ao código. Usei prompts específicos para cada decisão, em vez de um prompt genérico "gere todos os ADRs".

3. **RFC (30 min):** com as decisões formalizadas, o RFC consolidou a proposta técnica. As alternativas descartadas e questões em aberto da transcrição encontraram lugar natural aqui. Referenciei todos os 7 ADRs com links.

4. **FDD (1h30):** o documento mais extenso e técnico. Detalhei fluxos, contratos de API com payloads de exemplo, matriz de erro com prefixo `WEBHOOK_`, estratégias de resiliência, observabilidade e — crucialmente — a seção "Integração com o sistema existente" com 12 referências a arquivos reais do código.

5. **PRD (45 min):** produzi por último entre os grandes documentos. Com RFC, FDD e ADRs prontos, o PRD foi praticamente uma consolidação em linguagem de produto.

6. **Tracker (45 min):** montei a tabela de rastreabilidade varrendo cada documento e ligando cada item ao timestamp da transcrição ou ao arquivo de código. Isso funcionou como uma auditoria reversa: itens sem origem identificável eram sinal de alucinação da IA e foram corrigidos ou removidos.

7. **README do processo (20 min):** este documento, escrito quando o processo já estava completo.

### Estratégia de interação com a IA

- **Prompts dirigidos, não genéricos:** em vez de "gere um PRD", usei prompts que especificavam seções exatas, mencionavam timestamps da transcrição e arquivos do código.
- **Uma decisão por vez:** cada ADR foi escrito com um prompt focado naquela decisão específica, com o contexto relevante da transcrição já extraído.
- **Revisão cruzada:** o Tracker funcionou como ferramenta de verificação contra alucinações — se um item não tinha timestamp ou arquivo correspondente, era falso.
- **Sem mexer no código:** o código fonte serviu apenas como contexto e referência. Nenhum arquivo em `src/`, `prisma/`, `tests/` ou configurações foi alterado.

---

## Prompts customizados

### Prompt 1: Exploração inicial do código base

Usei este prompt para mapear sistematicamente os padrões do projeto antes de começar a escrever qualquer documento:

```
Você é um engenheiro de software fazendo code review de um projeto
Node.js + TypeScript. Preciso que você mapeie os padrões arquiteturais
do código. Especificamente:

1. Como os módulos são estruturados? (pasta, arquivos, convenções)
2. Como as classes de erro são organizadas? (hierarquia, códigos, uso)
3. Como funciona o middleware de erro centralizado?
4. Como funciona a autenticação e autorização?
5. Como é o padrão de logging?
6. Como schemas Zod são usados para validação?
7. Como o Prisma é configurado e usado?
8. Como o OrderService.changeStatus implementa a transação atômica?

Para cada item, forneça o caminho do arquivo, um resumo do padrão
e um trecho de código representativo.
```

Esse prompt produziu o mapeamento que usei como base para todos os ADRs e para a seção de integração do FDD.

### Prompt 2: Extração de decisões da transcrição

Usei este prompt para identificar e classificar decisões na transcrição antes de escrever os ADRs:

```
Analise a transcrição abaixo e extraia todas as decisões técnicas
mencionadas. Para cada decisão, classifique como:

- DECIDIDO: houve consenso e a decisão foi fechada
- ADIADO: mencionado mas deixado para depois
- DESCARTADO: considerado e explicitamente rejeitado

Para cada item DECIDIDO, forneça:
1. O que foi decidido (uma frase)
2. Timestamps onde foi discutido
3. Quem argumentou a favor e contra
4. O trade-off principal
5. Se há código existente que será impactado

Para cada item DESCARTADO, forneça:
1. O que foi descartado
2. Por que foi descartado (a razão dada na reunião)
3. O que foi escolhido no lugar

Filtre: NÃO inclua menções passageiras ou comentários laterais
que não representam decisões.
```

Esse prompt foi fundamental para separar o que entrava nos ADRs do que ia para "fora de escopo" e "questões em aberto".

### Prompt 3: Verificação de consistência contra a transcrição

Usei este prompt ao final, como passo de revisão:

```
Compare cada requisito funcional listado abaixo com a transcrição
e me diga se o requisito TEM ou NÃO TEM origem na reunião. Para
cada um, cite o timestamp exato onde foi discutido.

Se um requisito NÃO TEM origem, classifique como ALUCINAÇÃO e sugira
removê-lo.

[Lista de requisitos do PRD]
```

Esse prompt me ajudou a identificar e remover itens que a IA havia inferido mas não estavam na transcrição.

---

## Iterações e ajustes

### Iteração 1: Primeira geração (completa)

Gerei todos os documentos em sequência: ADRs → RFC → FDD → PRD → Tracker. A primeira geração já veio com boa estrutura porque os prompts foram dirigidos, mas identifiquei problemas:

- **Problema:** alguns ADRs repetiam o mesmo contexto da transcrição palavra por palavra, sem análise. **Correção:** pedi para reescrever cada ADR com foco no trade-off e nas consequências, não na transcrição.
- **Problema:** o RFC inicial tinha detalhes de implementação que pertenciam ao FDD (ex: schemas Prisma, pseudocódigo do worker). **Correção:** movi esses detalhes para o FDD e mantive o RFC em nível de arquitetura.

### Iteração 2: Refinamento do FDD

- **Problema:** a seção "Integração com o sistema existente" inicial tinha apenas 2 referências ao código. **Correção:** pedi para a IA listar todos os arquivos que seriam tocados e como, resultando em 12 referências com caminhos reais.
- **Problema:** a matriz de erros usava códigos genéricos (`400_BAD_REQUEST`). **Correção:** pedi para usar o padrão `WEBHOOK_*` consistente com o que foi discutido na reunião ([09:28] Bruno).

### Iteração 3: Verificação cruzada com o Tracker

- **Problema:** ao montar o Tracker, encontrei 3 itens no PRD sem timestamp correspondente (detalhes genéricos que a IA adicionou). **Correção:** removi os itens sem origem ou os substituí por equivalentes com origem na transcrição.
- **Problema:** 2 ADRs referenciaram arquivos que não existem no repositório (ex: `src/shared/utils/crypto.ts`). **Correção:** corrigi para apontar apenas arquivos reais.

### Iteração 4: Revisão final

- Revisão completa da checklist de critérios de aceite do enunciado.
- Verificação de que todos os links entre documentos funcionam (RFC → ADRs).
- Verificação de que os nomes de participantes, timestamps e paths de arquivo estão corretos.

**Total de ciclos:** 1 geração inicial + 3 iterações de refinamento = 4 ciclos.

---

## Como navegar a entrega

### Arquivos entregues

```
.
├── README.md                                    ← Este arquivo (processo de produção)
├── TRANSCRICAO.md                               ← Transcrição original (não alterada)
├── docs/
│   ├── PRD.md                                   ← Product Requirements Document
│   ├── RFC.md                                   ← Request for Comments (proposta técnica)
│   ├── FDD.md                                   ← Feature Design Document
│   ├── TRACKER.md                               ← Rastreabilidade (96 itens)
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md           ← Padrão Outbox no MySQL
│       ├── ADR-002-retry-backoff-dlq.md         ← Retry com backoff e DLQ
│       ├── ADR-003-hmac-sha256-secret-por-endpoint.md ← HMAC-SHA256 por endpoint
│       ├── ADR-004-at-least-once-x-event-id.md  ← At-least-once com X-Event-Id
│       ├── ADR-005-worker-processo-separado-polling.md ← Worker em polling
│       ├── ADR-006-reuso-padroes-existentes.md  ← Reuso de padrões do projeto
│       └── ADR-007-payload-snapshot-na-insercao.md ← Snapshot na inserção
├── src/                                         ← Código fonte (não alterado)
├── prisma/                                      ← Schema Prisma (não alterado)
└── tests/                                       ← Testes (não alterados)
```

### Ordem sugerida de leitura

1. **Comece pela transcrição** (`TRANSCRICAO.md`) — 10 minutos de leitura que dão todo o contexto.
2. **Depois os ADRs** (`docs/adrs/`) — cada um em 2-3 minutos. Entenda as decisões fundamentais.
3. **RFC** (`docs/RFC.md`) — 5-7 minutos. Veja como as decisões se consolidam em uma proposta.
4. **FDD** (`docs/FDD.md`) — 10-15 minutos. O mergulho técnico: fluxos, contratos, erros.
5. **PRD** (`docs/PRD.md`) — 5-7 minutos. A visão de produto, no topo da pirâmide.
6. **Tracker** (`docs/TRACKER.md`) — referência. Não precisa ler linearmente; use para buscar a origem de qualquer item.
7. **README** — este arquivo, com o processo de produção.

---

## Referência ao enunciado original

O enunciado original do desafio está disponível no repositório base:
https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia

---

*Documentação produzida inteiramente a partir da transcrição (`TRANSCRICAO.md`) e do código fonte existente (`src/`), usando IA como ferramenta de produção.*
