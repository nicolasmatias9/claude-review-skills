# claude-review-skills

Marketplace de plugins do Claude Code com as skills de revisão de PR dos projetos **SEQOP** e **Crono**.

| Plugin | Repositórios cobertos | Comandos |
|---|---|---|
| `seqop-review` | `petr-op-seqop-backend`, `petr-op-seqop-frontend`, `petr-op-core` | `/seqop-review:review-pr`, `/seqop-review:reviewer`, `/seqop-review:standards-spec-review` |
| `crono-review` | `petr-op-crono-backend`, `petr-op-crono-frontend` | `/crono-review:review-pr`, `/crono-review:reviewer`, `/crono-review:standards-spec-review` |

## O que cada skill faz

- **`review-pr`** (recomendado) — orquestra a revisão completa: fixa o commit do PR, roda as duas skills abaixo **de forma independente sobre o mesmo SHA**, valida, consolida e deduplica os achados, lê o CI e publica **um único review** no PR, com os dados estruturados (JSON) num bloco recolhido. Não revisa duas vezes o mesmo SHA (use `--force` para isso). Só revisa: não altera código.
- **`reviewer`** — caça bugs no PR: correção, segurança, regras do time, regras de camada e testes. Opcionalmente roda testes e build (`--run-tests`).
- **`standards-spec-review`** — revisa em dois eixos independentes, cada um num subagente: **Standards** (convenções documentadas, regras de camada, code smells e SOLID) e **Spec** (o PR entrega o que o ticket/spec pediu?). Não procura bugs.

As duas especialistas aceitam `--sha <sha>` (revisar um commit exato) e `--json <arquivo>` (gravar os achados no formato de `skills/reviewer/references/contrato-achados.md`). O `review-pr` usa as duas flags; no uso direto, são opcionais.

Quando o alvo é um PR, o resultado é **publicado no PR** (review do tipo Comment) com um destes status:

| Status | Quando |
|---|---|
| **CORRIGIR ANTES DO MERGE** | Há item obrigatório |
| **RESPONDER ANTES DO MERGE** | Nada a corrigir, mas há pergunta que pode mudar o código |
| **PRONTO PARA MERGE** | Nada a corrigir nem a responder (sugestões são opcionais) |

Use `--no-post` para receber o relatório só no chat. Nenhuma skill aprova, faz merge, commita ou altera o código do PR.

## Loop engineering

Este repositório é a base do loop de engenharia (planejar → implementar → revisar → corrigir → revalidar). Etapas:

| Etapa | Situação |
|---|---|
| 1. Contrato JSON dos achados (`contrato-achados.md`) e saída estruturada nas duas especialistas | feito (1.1.0) |
| 2. `review-pr`: orquestração, SHA fixo, consolidação e publicação única | feito (1.1.0), validar com PRs reais |
| 3. `resolve-pr-feedback`: correção dos achados com limite de ciclos e revisão independente | a fazer |
| 4. CI como autoridade independente (GitHub Actions + proteção de branch) | a fazer |
| 5. Disparo por eventos do GitHub | a fazer |
| 6. Observabilidade e aprendizado controlado das regras | a fazer |

O JSON consolidado publicado pelo `review-pr` (marcador `<!-- review-pr:v1 head_sha=... -->`) é o que a etapa 3 vai consumir.

## Instalação

Pré-requisitos: Claude Code e a [GitHub CLI](https://cli.github.com/) autenticada (`gh auth login`) com acesso a este repositório e aos repositórios revisados.

No Claude Code:

```
/plugin marketplace add nicolasmatias9/claude-review-skills
/plugin install seqop-review@claude-review-skills
/plugin install crono-review@claude-review-skills
```

Instale só o plugin do projeto em que você trabalha. Para receber atualizações: `/plugin marketplace update claude-review-skills`.

## Uso

Abra o Claude Code na **pasta do workspace** que contém os clones dos repositórios (ex.: a pasta com `petr-op-seqop-backend/` e `petr-op-seqop-frontend/` lado a lado). As skills localizam o repositório pelo nome da pasta.

```
/seqop-review:review-pr https://github.com/edgebr/petr-op-seqop-frontend/pull/301
/crono-review:review-pr 75 --no-post
/seqop-review:reviewer https://github.com/edgebr/petr-op-seqop-frontend/pull/301
/seqop-review:reviewer 171 --run-tests
/crono-review:standards-spec-review https://github.com/edgebr/petr-op-crono-backend/pull/75
/crono-review:standards-spec-review origin/develop --no-post
```

Também funciona pedindo em linguagem natural ("revisa o PR 301 do frontend do seqop").

## Ajustes por máquina

- `--run-tests` no backend usa `JAVA_HOME=C:\Program Files\Java\jdk-17` e Maven offline (`-o`). Se o seu JDK estiver em outro lugar ou as dependências não estiverem em cache, ajuste o comando quando o Claude pedir.
- Os comandos de worktree assumem Windows + PowerShell (limite de 260 caracteres em caminhos, junction para `node_modules`).

## Estrutura

```
.claude-plugin/marketplace.json
plugins/
├── seqop-review/
│   ├── .claude-plugin/plugin.json
│   └── skills/
│       ├── review-pr/             (SKILL.md — orquestrador)
│       ├── reviewer/              (SKILL.md, references/publicacao-pr.md, references/contrato-achados.md)
│       └── standards-spec-review/ (SKILL.md, references/arquitetura.md, references/smells.md)
└── crono-review/                  (mesma estrutura, regras do Crono)
```

As regras de camada em `references/arquitetura.md` foram medidas no código de cada projeto em 2026-11-03. Se o padrão do código mudar, meça de novo antes de mudar uma regra de heurística para violação (ou o contrário).

Para mudar uma regra: edite o arquivo, suba a `version` no `plugin.json` do plugin e no `marketplace.json`, e abra um PR neste repositório.
