---
name: review-pr
description: Orquestra a revisão completa de um PR do Crono (petr-op-crono-backend, petr-op-crono-frontend) — fixa o commit, roda /crono-review:reviewer e /crono-review:standards-spec-review de forma independente sobre o mesmo SHA, valida, consolida e deduplica os achados e publica um único review no PR com status de merge e os dados estruturados. Use quando o usuário pedir "/crono-review:review-pr", "revisão completa do PR", "roda as duas revisões no PR 123" ou "review-pr". Só revisa, nunca altera código.
---

# review-pr — revisão orquestrada de PR

Executa as duas revisões especialistas **sobre o mesmo commit**, sem que uma veja a outra, e transforma os resultados num relatório único e num JSON consolidado. É a primeira etapa do loop de engenharia: o JSON publicado no PR é o que a etapa de correção (`resolve-pr-feedback`, futura) vai consumir.

| Especialista | Pergunta | Como roda aqui |
|---|---|---|
| `crono-review:reviewer` | Esse PR quebra alguma coisa? | Num subagente isolado |
| `crono-review:standards-spec-review` | Está bem construído? É a coisa certa? | Neste contexto; ela mesma dispara seus dois subagentes (Standards e Spec) |

Esta skill **não** altera código, não commita, não aprova e não faz merge.

## Entrada

`/crono-review:review-pr <PR> [--no-post] [--run-tests] [--force] [spec=<caminho>]`

- `<PR>`: número ou URL `https://github.com/edgebr/<repo>/pull/N`. Com só o número, inferir o repo pela pasta atual; se não der, perguntar (backend ou frontend).
- `--no-post`: consolidar e mostrar no chat, sem publicar no PR.
- `--run-tests`: repassado ao `reviewer`.
- `--force`: revisar de novo mesmo se o SHA atual já tiver review consolidado.
- `spec=<caminho>`: repassado à `standards-spec-review`.

Pré-requisito: `gh` autenticado (`gh auth status`). Sem ele, parar e orientar `gh auth login`: sem `gh` não há como fixar o SHA nem publicar.

## Passo 1 — Fixar o commit

```bash
R=edgebr/<repo>; N=<numero>
gh pr view $N -R $R --json number,url,state,isDraft,title,headRefName,headRefOid,baseRefName
git -C <pasta> fetch origin pull/$N/head <baseRefName>      # traz os objetos sem criar ref
git -C <pasta> cat-file -e <headRefOid>^{commit}
git -C <pasta> merge-base origin/<baseRefName> <headRefOid>  # base_sha
```

- `SHA` = `headRefOid` (completo). Daqui em diante, **todo** passo usa esse SHA, nunca "o head atual".
- PR fechado ou mergeado: avisar e parar, a menos que o usuário peça mesmo assim (nesse caso, forçar `--no-post`).
- Não criar a ref `review/pr-N`: as duas skills recebem `--sha` e trabalham direto no SHA.
- Pasta de trabalho: `$SCRATCH/review-pr-<N>-<SHA7>/`.

## Passo 2 — Não revisar duas vezes o mesmo SHA

```bash
gh api repos/$R/pulls/$N/reviews --paginate --jq '.[] | select(.body | contains("<!-- review-pr:v1 ")) | {id, html_url, commit_id, submitted_at}'
```

- Já existe review consolidado com `head_sha=<SHA>` e não há `--force`: informar o link e **parar**.
- Existem reviews consolidados de SHAs anteriores: estão **desatualizados**. Guardar o SHA e o link do mais recente para citar no relatório novo.

## Passo 3 — Ler o CI (autoridade objetiva)

```bash
gh pr checks $N -R $R --json name,state,bucket,link
```

Classificar em `pass` (todos `pass`/`skipping`), `fail` (algum `fail`/`cancel`), `pending` (algum `pending`) ou `none` (o comando não retorna checks). Esta skill **não** declara que o build passa: só repassa o que o CI informa. Se o `reviewer` rodar testes (`--run-tests`), isso é evidência local, não substitui o CI.

## Passo 4 — Rodar as duas revisões, isoladas

Na **mesma mensagem**:

1. `Agent` com `subagent_type: general-purpose`, com este brief (preencher os `<>`):

   ```
   Use a ferramenta Skill para executar a skill `crono-review:reviewer` com os argumentos:
   <URL do PR> --no-post --sha <SHA> --json <pasta de trabalho>/reviewer.json [--run-tests]
   Siga a skill até o fim, inclusive a validação do JSON.
   Não invoque `crono-review:review-pr` nem `crono-review:standards-spec-review`. Não publique nada no PR.
   Se não conseguir criar subagentes, revise diretamente.
   Ao terminar, responda só: caminho do JSON, status de merge e contagem por severidade. Não repita os achados.
   ```

2. Neste contexto, a ferramenta `Skill` com `crono-review:standards-spec-review` e os argumentos `<URL do PR> --no-post --sha <SHA> --json <pasta de trabalho>/standards.json [spec=<caminho>]`. Seguir a skill até o fim (ela dispara os eixos Standards e Spec em subagentes e verifica as citações).

Plugin não instalado: se a ferramenta Skill não encontrar a skill (por exemplo, só o repositório `claude-review-skills` foi clonado), o brief manda o subagente **ler e seguir** o arquivo `../reviewer/SKILL.md` relativo ao diretório base desta skill, com os mesmos argumentos; neste contexto, ler e seguir `../standards-spec-review/SKILL.md` da mesma forma. Registrar no relatório que a execução foi por leitura do arquivo.

Independência:
- Não ler o `reviewer.json` nem o retorno do subagente do `reviewer` antes de terminar a `standards-spec-review` por completo (incluindo o JSON dela). A verificação de citações da `standards-spec-review` acontece neste contexto e não pode ser influenciada pelos achados do `reviewer`.
- Não passar ao subagente do `reviewer` nada da análise de Standards/Spec.
- O ambiente pode entregar o retorno do subagente do `reviewer` antes de a `standards-spec-review` terminar. Nesse caso, não usar o conteúdo até gravar o `standards.json`, e registrar no relatório se isso aconteceu depois da verificação de citações (sem efeito) ou antes dela.

Se uma das duas falhar (erro, JSON ausente ou inválido): não publicar nada. Relatar o que rodou e o que falhou, e parar.

## Passo 5 — Validar os resultados

Para cada arquivo (`reviewer.json`, `standards.json`):

- JSON válido (comando em [../reviewer/references/contrato-achados.md](../reviewer/references/contrato-achados.md)) com `schema_version` `"1.0"` e o campo `reviewer` esperado;
- `head_sha` **igual** ao SHA do Passo 1.

SHA divergente em qualquer um: a consolidação é inválida. Rodar de novo **só a skill divergente** com o mesmo `--sha`, uma vez. Se divergir de novo, parar sem publicar e relatar.

Depois, reler o `headRefOid` do PR. Se mudou durante a revisão, o resultado continua válido para o SHA revisado, mas o relatório precisa dizer: "o PR recebeu commits depois desta revisão; rode `review-pr` de novo".

## Passo 6 — Consolidar

Os achados originais **não** são reescritos: o consolidado referencia os IDs de origem (`REV-*`, `STD-*`, `SPC-*`).

**Deduplicação (só com fundamento):** dois achados formam um grupo apenas quando estão no **mesmo arquivo**, no **mesmo símbolo** (ou em linhas a até ~5 de distância) **e** descrevem o **mesmo risco**, conferido lendo `scenario` e `evidence` dos dois. Semelhança de texto ou de fingerprint não basta. Na dúvida, manter separados.

- Grupo: um item consolidado `CON-001`… com `merged_from` (IDs de origem), `severity` = a mais grave do grupo, e uma frase dizendo por que foram agrupados.
- Achado de Spec só agrupa com achado do `reviewer` se for literalmente o mesmo defeito.
- **Conflito** (ex.: um especialista sugere o oposto do outro para o mesmo trecho): não escolher um lado. Virar pergunta ao autor (`severity: question`, `conflict: true`) citando os dois IDs.

## Passo 7 — Decidir o status e a próxima ação

Na ordem, o primeiro que se aplica:

| Situação | `merge_status` | `next_action` |
|---|---|---|
| CI `fail` | `CORRIGIR_ANTES_DO_MERGE` (item obrigatório "CI falhando: <checks>") | `corrigir` |
| Algum `blocker`, ou `important` com `actionable: true` | `CORRIGIR_ANTES_DO_MERGE` | `corrigir` |
| Algum `question`, `conflict`, ou `important` com `actionable: false` | `RESPONDER_ANTES_DO_MERGE` | `responder` |
| Só `suggestion`, ou nada | `PRONTO_PARA_MERGE` | `aprovacao_humana` |

- Com `PRONTO_PARA_MERGE` e CI `pending`/`none`, o texto diz explicitamente "só mergear depois que os checks passarem" (ou "o PR não tem checks de CI").
- Sugestões nunca disparam correção automática.
- Aprovação e merge são sempre humanos.

## Passo 8 — JSON consolidado

Gravar `<pasta de trabalho>/consolidado.json` (ferramenta Write, depois validar):

```json
{
    "schema_version": "1.0",
    "kind": "review-pr",
    "repository": "edgebr/<repo>",
    "pull_request": 0,
    "base_sha": "<40 chars>",
    "head_sha": "<40 chars>",
    "generated_at": "<ISO-8601 UTC>",
    "ci": { "state": "pass|fail|pending|none", "checks": [{ "name": "", "state": "" }] },
    "merge_status": "CORRIGIR_ANTES_DO_MERGE|RESPONDER_ANTES_DO_MERGE|PRONTO_PARA_MERGE",
    "next_action": "corrigir|responder|aprovacao_humana",
    "sources": {
        "reviewer": { "merge_status": "", "counts": { "blocker": 0, "important": 0, "suggestion": 0, "question": 0 } },
        "standards-spec-review": { "merge_status": "", "spec": null, "counts": { "blocker": 0, "important": 0, "suggestion": 0, "question": 0 }, "discarded_in_verification": 0 }
    },
    "groups": [{ "id": "CON-001", "merged_from": ["REV-002", "STD-001"], "severity": "important", "reason": "" }],
    "conflicts": [{ "id": "CON-002", "between": ["REV-003", "STD-002"], "question": "" }],
    "findings": [],
    "requirements": [],
    "previous_review": { "head_sha": null, "url": null }
}
```

- `findings`: **todos** os achados das duas fontes, no formato do contrato, sem alteração (inclusive os que estão em grupos).
- `requirements`: copiados do `standards.json`.

## Passo 9 — Publicar (exceto com `--no-post`)

Um único review `event: "COMMENT"` com `commit_id` = SHA, seguindo as regras de escrita e de envio de [../reviewer/references/publicacao-pr.md](../reviewer/references/publicacao-pr.md) (status no texto, nada de `APPROVE`/`REQUEST_CHANGES`, só afirmar o que foi verificado, JSON sem BOM). Corpo:

`````markdown
## Status: <CORRIGIR ANTES DO MERGE | RESPONDER ANTES DO MERGE | PRONTO PARA MERGE>

<Uma frase com o que o dev precisa fazer.>

**Commit revisado:** `<SHA>` · **CI:** <pass | fail: checks | pending | sem checks>
<Se houver: "Revisão anterior (`<sha7>`) ficou desatualizada: <link>". Se o PR mudou durante a revisão: aviso para rodar de novo.>

| | Bloqueante | Importante | Sugestão | Pergunta |
|---|---|---|---|---|
| reviewer | n | n | n | n |
| standards-spec-review | n | n | n | n |
| **Consolidado** | n | n | n | n |

### Obrigatório
1. `[reviewer REV-001]` `arquivo:linha` — <problema> — <correção>
2. `[CON-001 = REV-002 + STD-001]` `arquivo:linha` — <problema> — <correção>
(ou "Nada.")

### Perguntas
### Opcional
### Spec
<x de y requisitos atendidos; parciais/ausentes/divergentes já listados em Obrigatório. Ou "Spec não avaliada: <motivo>".>

### Verificado
- <checks de cada especialista; descartados na verificação>
- Não verificado: <...>

<details><summary>Dados estruturados (review-pr v1)</summary>

````json
<conteúdo do consolidado.json>
````

</details>

<!-- review-pr:v1 head_sha=<SHA> -->

---
<sub>Revisão automática via `/crono-review:review-pr` (reviewer + standards-spec-review). Achados conferidos antes da publicação; em caso de discordância, responda neste PR.</sub>
`````

O bloco JSON dentro do `<details>` usa cerca de **quatro** crases porque `evidence` pode conter trechos com três.

- O marcador `<!-- review-pr:v1 head_sha=<SHA> -->` é obrigatório: é ele que o Passo 2 procura.
- Comentários inline: um por item obrigatório ou pergunta **com linha no diff** (um só por grupo), começando pelo ID (`[CON-001]`, `[REV-001]`…).
- Limite do GitHub: corpo até 65.536 caracteres. Se passar de ~60.000, resumir a seção "Opcional" para a contagem por arquivo, mantendo o JSON completo.

## Passo 10 — Responder ao usuário

Status, próxima ação, contagens (por fonte e consolidado), grupos e conflitos, link do review publicado (ou "não publicado: --no-post") e o caminho do `consolidado.json`. Com `next_action: corrigir`, **não** iniciar correção: dizer quais itens são obrigatórios e parar.

## Regras

- Uma execução por pedido. Não repetir a revisão até "zerar" os achados.
- Nunca editar código, commitar, aprovar ou fazer merge. A única escrita é o review do Passo 9.
- Nunca publicar consolidação de SHAs diferentes, nem consolidação parcial (com uma das revisões faltando).
- Não invocar `review-pr` de dentro dos especialistas, nem os especialistas de dentro dos subagentes deles.
