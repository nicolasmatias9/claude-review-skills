---
name: standards-spec-review
description: Revisa o diff entre HEAD e um ponto fixo (commit, branch, tag, origin/develop, HEAD~5 ou um PR) em dois eixos independentes — Standards ("está bem construído?", contra as convenções documentadas do repo e um baseline de code smells) e Spec ("é a coisa certa?", contra o ticket/spec que originou a mudança). Cada eixo roda num subagente isolado e os relatórios nunca são fundidos num veredito único; em PR, publica o resultado no PR com o status de merge. Use quando o usuário pedir "/crono-review:standards-spec-review", "revisar contra o padrão e a spec", "revisar a branch desde X", "a mudança atende a spec?" ou "review em dois eixos". Não é caçador de bugs — para isso use /code-review ou /crono-review:reviewer.
---

# Standards & Spec review

Revisa **o diff entre `HEAD` e um ponto fixo** em dois eixos que não se enxergam:

| Eixo | Pergunta | Fonte da verdade |
|---|---|---|
| **Standards** | Está bem construído? | Convenções documentadas do repo e as regras de camada em [references/arquitetura.md](references/arquitetura.md); sem regra documentada, o baseline de smells e SOLID em [references/smells.md](references/smells.md) |
| **Spec** | É a coisa certa? | O ticket, PR ou spec que originou a mudança |

Os eixos rodam em subagentes separados para que um não contamine o julgamento do outro. O resultado são **dois relatórios lado a lado**, sem veredito combinado e sem re-ranquear um contra o outro.

Esta skill **não procura bugs**. Bugs, segurança e testes quebrados são o trabalho do `/crono-review:reviewer` (PRs) ou do `/code-review` nativo.

## Entrada

`/crono-review:standards-spec-review <ponto-fixo | PR> [spec=<caminho>] [repo=<pasta>] [--no-post] [--sha <sha>] [--json <arquivo>]`

- **Ponto fixo** (obrigatório): commit, branch, tag, `origin/develop`, `HEAD~5`…
- **PR**: número ou URL `https://github.com/edgebr/<repo>/pull/N`. Vira ponto fixo = `origin/<base do PR>` e HEAD = ref temporária do PR.
- `spec=`: caminho de um arquivo de spec, para pular a busca.
- `repo=`: `petr-op-crono-backend` ou `petr-op-crono-frontend`. Se o diretório atual já for um deles, não precisa.
- `--no-post`: com alvo PR, não publicar no PR (o relatório fica só no chat). Sem a flag, revisão de PR é sempre publicada (Passo 7).
- `--sha <sha>`: com alvo PR, revisar exatamente esse commit (SHA completo) em vez do head atual. Usado pelo `review-pr`.
- `--json <arquivo>`: além do relatório, gravar os achados nesse arquivo no formato de [../reviewer/references/contrato-achados.md](../reviewer/references/contrato-achados.md) (Passo 6).

Se o ponto fixo não for informado, **perguntar**. Não adivinhar `develop`.

## Passo 1 — Validar antes de gastar subagentes

1. Resolver o repositório (o diretório atual, `repo=` ou o repo do PR). A raiz do workspace **não** é um repositório git.
2. Se for PR, buscar sem fazer checkout no clone do usuário:
   ```bash
   gh pr view N -R edgebr/<repo> --json title,body,baseRefName,headRefName
   git -C <pasta> fetch origin pull/N/head:review/pr-N <baseRefName>
   ```
   Nesse caso o HEAD é `review/pr-N` e o ponto fixo é `origin/<baseRefName>`.

   **Com `--sha <sha>`:** não criar nem apagar a ref `review/pr-N` (outra revisão pode estar usando o mesmo PR em paralelo). O HEAD é o próprio `<sha>`. Se o objeto não existir localmente (`git -C <pasta> cat-file -e <sha>^{commit}` falha), rodar `git -C <pasta> fetch origin pull/N/head <baseRefName>` sem criar ref e conferir de novo; se continuar faltando, abortar sem gerar relatório nem JSON. Sem `--sha`, anotar o SHA completo de `review/pr-N` (`git rev-parse`): ele é o `head_sha` do relatório e do JSON.
3. Conferir que o ponto fixo resolve: `git -C <pasta> rev-parse --verify <ponto-fixo>`.
4. Gerar o diff **three-dot**, que mede a partir do merge-base e ignora staged e working tree:
   ```bash
   git -C <pasta> diff --output="$SCRATCH/ssr.diff" <ponto-fixo>...<head>
   git -C <pasta> log --output="$SCRATCH/ssr-commits.txt" --format='%h %s%n%b' <ponto-fixo>..<head>
   ```
5. Usar sempre `--output=<arquivo>` do git, nunca `>`: neste ambiente o Bash passa `git` por um wrapper (`rtk`) que compacta a saída redirecionada, e os subagentes revisariam um resumo em vez do diff. Conferir que o `ssr.diff` tem uma linha `diff --git` por arquivo alterado. Se o diff estiver vazio, **parar aqui** e dizer isso. Se `git status` mostrar mudanças não commitadas, avisar que elas ficam de fora da revisão.

## Passo 2 — Reunir as fontes de Standards

Listar (não resumir) os documentos de convenção que valem para o repo. Os mais específicos vencem os mais gerais:

| Repo | Documentos |
|---|---|
| `petr-op-crono-backend` | `CLAUDE.md` (arquitetura, migrations, segurança, testes, convenções; onde ele contradiz o `pom.xml`, vale o `pom.xml`, como o próprio arquivo orienta), `docs/prompts/generate-unit-and-integration-tests.prompt.md` (padrão de testes) |
| `petr-op-crono-frontend` | `.editorconfig` (4 espaços, aspas simples em `.ts`), `README.md`, `docs/TESTING-TOOLS.md` (cita Karma, mas o `npm test` atual é Vitest), `.github/pull_request_template.md` |
| backend e frontend | [references/arquitetura.md](references/arquitetura.md): regras de camada (ARQ-B*, ARQ-F*), já classificadas em violação ou heurística |
| se existir | um `CLAUDE.md` em pasta acima do workspace (estilo geral da máquina do dev) |

Incluir também qualquer `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md` ou `CODING_STANDARDS.md` encontrado na raiz do repo ou na pasta dos arquivos alterados.

**Linter:** nenhum dos dois repos tem linter configurado. O que o `.editorconfig` já define não deve ser reportado.

## Passo 3 — Localizar a spec

Tentar nesta ordem e parar no primeiro que encontrar:

1. **`spec=<caminho>`** passado na chamada.
2. **Descrição do PR**, se o alvo for um PR e a descrição tiver requisitos (no template do frontend: "Objetivo", "Comportamento esperado após merge", "Test plan").
3. **Documento no workspace** cujo nome combine com a branch, a chave Jira (`CRONO-xxx`) ou palavras do título dos commits:
   - `petr-op-crono-frontend/.Codex/knowledge-base/business-rules/`;
   - `petr-op-crono-frontend/docs/` (ex.: `PR-CRONO-xxx-*.md`, `GUIA-*.md`);
   - `petr-op-crono-backend/docs/`;
   - arquivos `.md` na raiz do workspace `crono` (ex.: `sequence-links-projections.md`).
4. **Perguntar ao usuário.** O Jira não está acessível daqui: pedir que cole o texto do ticket ou indique o arquivo.

Se o usuário disser que não há spec, o eixo Spec é **pulado**. Nunca inferir os requisitos a partir do próprio código.

## Passo 4 — Rodar os dois eixos em paralelo

Disparar **na mesma mensagem** dois `Agent` com `subagent_type: general-purpose` (o de Spec só se houver spec). Cada brief é autocontido. **Não** passar a um eixo nada do raciocínio do outro.

### Brief do eixo Standards

```
Você revisa UM eixo de code review: Standards ("está bem construído?"). Não procure bugs nem avalie se a mudança atende requisitos — outro revisor cuida disso.

Repositório: <pasta absoluta>
Diff a revisar (three-dot, só código commitado): <caminho do ssr.diff>
Para ler um arquivo inteiro na versão revisada: git -C <pasta> show <head>:<caminho>

Fontes de convenção, da mais específica para a mais geral (a mais específica vence):
<lista de caminhos do Passo 2>

Regras de arquitetura (camadas): <caminho absoluto de references/arquitetura.md>. Respeite a classificação de cada regra: as da seção "Violações" são violação; as de "Heurísticas", heurística.

Baseline de smells e SOLID (só onde o repo não documenta regra): <caminho absoluto de references/smells.md>. Não cobre Clean Architecture: o projeto não a adota.

Regras:
- Revise só linhas adicionadas ou alteradas no diff. Código antigo fora do diff não entra.
- Não reporte o que o .editorconfig já define.
- Todo achado PRECISA de citação:
  - violação de regra do repo: arquivo da regra + trecho literal da regra + hunk literal do diff;
  - smell: nome do smell do baseline + hunk literal do diff.
  Achado sem citação deve ser descartado.
- Violação de regra documentada = "violação". Smell = "heurística", nunca violação, sempre com sugestão de correção.
- No máximo 10 achados, os mais relevantes primeiro.
- Não invoque /crono-review:standards-spec-review, /crono-review:reviewer, /code-review nem nenhuma skill, e não crie outros agentes: faça a revisão diretamente.

Responda em português, neste formato, sem preâmbulo:
### Violações
1. `arquivo:linha` — <problema em uma frase>
   - Regra: <arquivo da regra> — "<trecho literal da regra>"
   - Trecho: ```<hunk literal>```
   - Correção: <sugestão concreta>
### Heurísticas (smells)
1. `arquivo:linha` — <Nome do smell>: <problema em uma frase>
   - Trecho: ```<hunk literal>```
   - Correção: <sugestão concreta>
Se não houver achados numa seção, escreva "Nenhum."
```

### Brief do eixo Spec

```
Você revisa UM eixo de code review: Spec ("é a coisa certa?"). Não avalie estilo, convenções nem qualidade de código — outro revisor cuida disso.

Repositório: <pasta absoluta>
Diff a revisar (three-dot, só código commitado): <caminho do ssr.diff>
Commits: <caminho do ssr-commits.txt>
Spec: <caminho(s) da spec, ou o texto do ticket/PR salvo no scratchpad>
Para ler um arquivo inteiro na versão revisada: git -C <pasta> show <head>:<caminho>

Para cada requisito, critério de aceite ou cenário da spec, classifique:
- Atendido — aponte onde (arquivo:linha).
- Parcial — o que falta.
- Ausente — nada no diff implementa.
- Divergente — implementado de forma diferente do pedido.
Depois liste o escopo extra: mudanças do diff que nenhum requisito pede.

Regras:
- Todo achado PRECISA citar a linha literal da spec. Requisito sem citação não existe.
- Um requisito pode estar atendido por código que já existia: confira o arquivo inteiro antes de dizer "Ausente".
- Escopo extra não é automaticamente ruim: descreva, não condene.
- Não invoque /crono-review:standards-spec-review, /crono-review:reviewer, /code-review nem nenhuma skill, e não crie outros agentes: faça a revisão diretamente.

Responda em português, neste formato, sem preâmbulo:
### Requisitos
| # | Requisito (citação literal da spec) | Status | Onde / o que falta |
### Divergentes e ausentes
1. "<citação da spec>" — <o que o código faz em vez disso> (`arquivo:linha`)
### Escopo extra
- `arquivo` — <o que mudou sem requisito correspondente>
```

## Passo 5 — Verificar as citações

Os subagentes produzem hipóteses. Antes de apresentar, conferir **cada** achado:

- o hunk citado existe no diff (`grep -F` no `ssr.diff`) e o `arquivo:linha` aponta para ele;
- o trecho de regra ou de spec citado existe literalmente no arquivo de origem; a comparação ignora só a ênfase markdown (`**`, `__`), porque o subagente costuma citar o texto como aparece renderizado; qualquer outra diferença invalida a citação;
- em "Ausente", dar uma olhada rápida no arquivo inteiro para confirmar que não estava implementado antes.

Para conferir texto com acento (regras e specs em português), usar a ferramenta Grep ou `grep -F` no Bash. O `Get-Content` do Windows PowerShell 5.1 lê UTF-8 sem BOM como ANSI e dá falso "não encontrado".

Achado com citação que não confere: descartar e contar quantos foram descartados. Não corrigir nem reescrever achados por conta própria: se algo parecer errado, descartar.

## Passo 6 — Relatório

Dois blocos separados, nesta ordem, cada um com o conteúdo verificado do seu eixo:

```markdown
# Standards & Spec review — <repo> (<ponto-fixo>...<head>)
<N> arquivos, +<a>/-<d>. Commit revisado: <head_sha completo>. Spec usada: <caminho/PR/ticket ou "nenhuma">.

## Standards
<relatório verificado do eixo Standards>

## Spec
<relatório verificado do eixo Spec, ou "Nenhuma spec disponível — eixo não avaliado.">

## Resumo
- Pior ponto de Standards: <um item, ou "nenhum">
- Pior ponto de Spec: <um item, ou "nenhum">
- Achados descartados na verificação: <n>
```

O resumo **não** elege um eixo vencedor nem compara os eixos entre si. O status de merge do Passo 7 é outra coisa: ele não pondera os eixos, só diz se **algum** deles tem item obrigatório.

### Saída estruturada (`--json <arquivo>`)

Gravar, com a ferramenta Write, um arquivo no formato de [../reviewer/references/contrato-achados.md](../reviewer/references/contrato-achados.md) com `"reviewer": "standards-spec-review"`, o `head_sha` revisado e **só os achados que sobreviveram ao Passo 5**, convertidos pelo de-para do contrato:
- eixo Standards → `source: "standards"`, IDs `STD-001`…; Violação → `important`, Heurística → `suggestion`; `rule` com o código citado (`ARQ-B8`, `CLAUDE.md#Migrations`, `smell:<Nome>`, `SOLID-<letra>`);
- eixo Spec → `source: "spec"`, IDs `SPC-001`…; Ausente/Parcial/Divergente → `important` com `category` `spec_missing`/`spec_partial`/`spec_divergence`; escopo extra → `suggestion`, `scope_creep`, `actionable: false`;
- `requirements` com **todos** os requisitos da tabela do eixo Spec, inclusive os atendidos (`met`), e `requirement_id` ligando cada achado de Spec ao requisito;
- `spec` com a origem da spec, ou `null` se o eixo foi pulado; `verification.discarded_in_verification` com o número do Passo 5.

Validar o JSON (comando no contrato) antes de terminar. Com `--json`, responder no fim com o caminho do arquivo e uma linha de resumo por eixo.

## Passo 7 — Publicar o resultado no PR

**Obrigatório** quando o alvo é um PR e não houver `--no-post`. Seguir [../reviewer/references/publicacao-pr.md](../reviewer/references/publicacao-pr.md): status (CORRIGIR / RESPONDER / PRONTO PARA MERGE), formato do review, comentários inline e comando de publicação. Publicar sem pedir confirmação: é regra do time que toda revisão de PR seja publicada no PR.

Para esta skill:
- **Exigem correção:** Violações de Standards; requisitos Parciais, Ausentes ou Divergentes na Spec.
- **Opcionais:** Heurísticas (smells) e escopo extra.
- Na seção "Obrigatório" do review, manter a separação por eixo (`Standards` / `Spec`) para o dev saber de onde vem cada item.

## Regras

- Rodar uma vez por pedido. Não repetir a revisão até "zerar" os achados: rodadas novas sempre trazem algo novo, e só achados citados merecem ação.
- Nunca editar código, commitar, aprovar ou fazer merge. A única escrita permitida é o review do Passo 7.
- Ao final, apagar a ref `review/pr-N`, se tiver sido criada por esta execução (com `--sha`, nenhuma ref é criada).
- Chamada pelo `review-pr`: não invocar `review-pr` nem `reviewer`; os dois eixos continuam nos subagentes do Passo 4.
