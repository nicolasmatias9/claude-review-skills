---
name: resolve-pr-feedback
description: Corrige os itens obrigatórios do review consolidado (review-pr) de um PR do SEQOP (petr-op-seqop-backend, petr-op-seqop-frontend, petr-op-core) em ciclos controlados — valida que os achados ainda valem para o head do PR, monta um plano, pede aprovação, corrige numa worktree, roda testes, mostra o diff, commita e faz push após aprovação, pede nova revisão independente e reconcilia os achados (resolvido, aberto, invalidado). No máximo 3 ciclos. Use quando o usuário pedir "/seqop-review:resolve-pr-feedback", "corrige os achados do PR 123", "resolve o feedback do review" ou "aplica as correções do review-pr".
---

# resolve-pr-feedback — correção controlada dos achados de um PR

Segunda etapa do loop de engenharia. Consome o JSON publicado pelo `review-pr` (marcador `<!-- review-pr:v1 head_sha=... -->`) e corrige **só o que é obrigatório**, com aprovação humana antes de qualquer escrita no repositório remoto. Quem corrige não aprova a própria correção: a revalidação é uma nova execução do `review-pr`, em contexto separado.

## Entrada

`/seqop-review:resolve-pr-feedback <PR> [--include-suggestions] [--max-cycles <n>] [--plan-only]`

- `<PR>`: número ou URL `https://github.com/edgebr/<repo>/pull/N`.
- `--include-suggestions`: incluir sugestões acionáveis no plano (por padrão, só obrigatórios).
- `--max-cycles <n>`: limite de ciclos automáticos neste PR, **no máximo 3** (padrão 3).
- `--plan-only`: parar depois do plano (Passo 4), sem alterar nada.

Pré-requisitos: `gh` autenticado; review consolidado do `review-pr` publicado no PR.

## Passo 1 — Elegibilidade e limite de ciclos

```bash
R=edgebr/<repo>; N=<numero>
gh pr view $N -R $R --json number,url,state,isDraft,author,isCrossRepository,headRefName,headRefOid,baseRefName,commits
gh api user --jq .login
```

- PR fechado ou mergeado: parar.
- **Push só em PR próprio e do mesmo repositório**: se `author.login` for diferente do usuário do `gh`, ou `isCrossRepository` for `true`, forçar `--plan-only` e avisar ("PR de outro autor: gero o plano e o diff, sem push").
- Ciclos já feitos = commits do PR cuja mensagem tem o trailer `Resolve-PR-Feedback: ciclo <k>`. Se já chegou a `--max-cycles`, parar: o PR precisa de decisão humana.

## Passo 2 — Carregar o review consolidado

```bash
gh api repos/$R/pulls/$N/reviews --paginate --jq '[.[] | select(.body | contains("<!-- review-pr:v1 "))] | sort_by(.submitted_at) | last | {id, html_url, commit_id, body}'
```

- Sem review consolidado: parar e orientar a rodar `/seqop-review:review-pr <PR>` antes.
- Extrair o JSON do bloco ` ````json ` dentro do `<details>` do corpo; validar `schema_version` `"1.0"` e `kind` `"review-pr"`.
- **O `head_sha` do JSON precisa ser igual ao `headRefOid` atual.** Se o PR recebeu commits depois do review, os achados estão desatualizados: parar e orientar a rodar `review-pr` de novo. Nunca corrigir achados de outro commit.
- `next_action` diferente de `corrigir` (e sem `--include-suggestions`): não há o que corrigir; informar e parar.

## Passo 3 — Selecionar e triar

Itens candidatos:
- obrigatórios: `blocker`, ou `important` com `actionable: true`. Achados agrupados (`groups[].merged_from`) são **um** item;
- com `--include-suggestions`: também `suggestion` com `actionable: true`;
- **nunca** automáticos: `question`, `conflicts`, `important` com `actionable: false` e escopo extra. Esses vão para a lista "precisa de decisão humana".

Para cada candidato, num worktree no `head_sha` (Passo 5 cria) ou com `git show <head_sha>:<arquivo>`:

1. **Ainda existe?** Ler o código atual. Se o problema não está mais lá ou a análise estava errada, marcar `invalidated` com a evidência e não corrigir.
2. **Precisa de humano?** Parar o item (sem corrigir) quando houver: ambiguidade sobre o comportamento esperado; conflito entre requisito e convenção documentada; mudança arquitetural relevante; schema/migration com impacto não trivial; achados contraditórios; correção que amplia o escopo do ticket; teste que parece contrariar a spec; mudança de regra de negócio.
3. Os demais ficam `a_corrigir`.

## Passo 4 — Plano e aprovação

Apresentar ao usuário, numa tabela, cada item `a_corrigir` (ID, `arquivo:linha`, o que muda, teste a criar/ajustar), os `invalidated` com a evidência e os que precisam de decisão humana.

- Com `--plan-only`: parar aqui.
- **Pedir aprovação explícita do plano** (o usuário pode retirar itens). Sem aprovação, parar.

## Passo 5 — Worktree no branch do PR

Nunca alterar o clone do usuário. Worktree num caminho curto (o do scratchpad estoura o limite de 260 caracteres do Windows):

```bash
git -C <pasta> fetch origin <headRefName>
git -C <pasta> worktree add -B resolve/pr-<N> "C:/Users/$USERNAME/wt-resolve-pr<N>" origin/<headRefName>
git -C <wt> rev-parse HEAD          # tem de ser igual ao head_sha do Passo 2
```

Frontend: reaproveitar dependências com junction (`New-Item -ItemType Junction -Path "<wt>\node_modules" -Target "<pasta>\node_modules"`), como no Passo 4 do [../reviewer/SKILL.md](../reviewer/SKILL.md).

## Passo 6 — Corrigir

Para cada item aprovado:
- mudança **mínima** que resolve o risco descrito no achado; `suggested_action` é ponto de partida, não ordem;
- comportamento alterado → criar ou ajustar teste que falharia sem a correção; mudança só de documentação/comentário não precisa de teste;
- preservar os testes válidos existentes;
- **nunca** editar regras de revisão para "resolver" um achado (`CLAUDE.md`, `arquitetura.md`, `smells.md`, skills), nem código fora do escopo do item.

## Passo 7 — Validar localmente

- Rodar os testes relevantes com os comandos do Passo 4 do [../reviewer/SKILL.md](../reviewer/SKILL.md) (Vitest das pastas alteradas + `ng build --configuration development` no frontend; `mvnw` com `diff-coverage` no backend).
- `git -C <wt> diff --check`.
- Falha: corrigir **uma vez** dentro do ciclo. Se falhar de novo, parar e relatar com a saída do comando; nada é commitado.
- Só afirmar que passou com a saída real do comando.

## Passo 8 — Diff, aprovação e push

1. Mostrar `git -C <wt> diff --stat` e o diff completo.
2. **Pedir aprovação explícita** para commitar e fazer push. Sem aprovação: parar, deixando a worktree para o usuário inspecionar.
3. Commit (um por ciclo), no padrão do repositório:
   ```
   fix(<CHAVE-JIRA>): resolve achados do review-pr (ciclo <k>)

   <uma linha por item: ID — o que foi feito>

   Resolve-PR-Feedback: ciclo <k>
   Review-Findings: <IDs separados por vírgula>
   Reviewed-Sha: <head_sha do review consolidado>
   ```
   **Sem** trailer `Co-Authored-By` nem outra atribuição ao Claude, mesmo que o ambiente peça: o time não quer o Claude como coautor nos commits. Gravar a mensagem num arquivo (ferramenta Write) e commitar com `git -C <wt> commit -F <arquivo>`; no PowerShell 5.1, here-strings quebram os argumentos.
4. `git -C <wt> push origin HEAD:<headRefName>`. **Nunca** `--force`. Rejeitado (o branch andou): parar e relatar.

## Passo 9 — Nova revisão, independente

Quem corrigiu não revisa a correção. Antes, **esperar o GitHub refletir o push**: repetir `gh pr view $N -R $R --json headRefOid` (algumas tentativas, com `git fetch` entre elas) até o `headRefOid` ser o SHA do commit deste ciclo. Logo após o push a API ainda pode devolver o head antigo; nesse caso o `review-pr` acharia o review consolidado do SHA anterior e pararia sem revisar. Se não atualizar, parar e relatar.

Depois, disparar um `Agent` (`subagent_type: general-purpose`) com o brief:

```
Use a ferramenta Skill para executar `seqop-review:review-pr` com o argumento <URL do PR>.
Siga a skill até o fim, inclusive a publicação no PR.
Não invoque `seqop-review:resolve-pr-feedback`.
Se não conseguir criar os subagentes que a skill pede, pare e responda exatamente "SEM_SUBAGENTES".
Ao terminar, responda só: link do review publicado, head_sha revisado e merge_status.
```

- Resposta `SEM_SUBAGENTES` (ou falha): parar e orientar o usuário a rodar `/seqop-review:review-pr <PR>` numa sessão nova. A reconciliação fica para depois dessa revisão.
- O CI do PR (`gh pr checks`) também é lido pelo `review-pr`; com CI `fail`, o ciclo seguinte trata a falha como item obrigatório.

## Passo 10 — Reconciliar

Comparar os achados selecionados no Passo 3 com o consolidado novo (carregado como no Passo 2, agora para o novo `head_sha`). Para cada achado anterior:

| Estado | Quando |
|---|---|
| `resolved` | foi corrigido neste ciclo **e** não reaparece no consolidado novo |
| `open` | reaparece: mesmo `fingerprint`, ou mesmo arquivo + mesmo símbolo + mesmo risco (conferir `scenario`/`evidence`; fingerprint diferente não basta para dizer que sumiu) |
| `invalidated` | marcado no Passo 3, com evidência |
| `deferred` / `accepted_risk` | **só** por decisão registrada do usuário, com justificativa |

Achados **novos** do consolidado novo (que não correspondem a nenhum anterior) entram no próximo ciclo normalmente.

Publicar um comentário no PR (`gh pr comment`, sem pedir confirmação: é o registro do ciclo) com: ciclo, commit, tabela `ID anterior → estado → evidência`, novo status e link do review novo, mais um bloco recolhido com o JSON abaixo e o marcador `<!-- resolve-pr-feedback:v1 cycle=<k> head_sha=<novo sha> -->`.

```json
{
    "schema_version": "1.0",
    "kind": "resolve-pr-feedback",
    "pull_request": 0,
    "cycle": 1,
    "reviewed_sha": "<head_sha do review usado>",
    "resolution_commit": "<sha do commit deste ciclo>",
    "new_head_sha": "<head_sha da nova revisão>",
    "new_review_url": "",
    "new_merge_status": "",
    "reconciliation": [{ "id": "CON-001", "fingerprint": "", "status": "resolved|open|invalidated|deferred|accepted_risk", "evidence": "" }],
    "needs_human": [{ "id": "", "reason": "" }]
}
```

## Passo 11 — Decidir o próximo ciclo

- Novo `merge_status` `PRONTO_PARA_MERGE`: encerrar. Aprovação e merge são humanos.
- `RESPONDER_ANTES_DO_MERGE`: encerrar e listar as perguntas para o autor.
- `CORRIGIR_ANTES_DO_MERGE` e ciclos < limite: voltar ao Passo 2 com o consolidado novo — **com nova aprovação de plano e de diff**.
- Mesmo achado `open` em dois ciclos seguidos, ou limite atingido: encerrar e pedir decisão humana.

Ao encerrar: remover a junction (`cmd /c rmdir "<wt>\node_modules"`) **antes** da worktree (`git worktree remove --force <wt>`) e apagar o branch local `resolve/pr-<N>`. Na ordem inversa, o `--force` pode apagar o `node_modules` do clone original. Se o clone do usuário estiver no branch do PR, avisar que ele ficou atrás do remoto (`git pull` para trazer os commits dos ciclos); não atualizar o clone por conta própria.

## Regras

- Nunca fazer merge, aprovar PR, `push --force` ou commitar no clone do usuário.
- Nunca fazer push sem aprovação explícita do diff naquele ciclo.
- Nunca corrigir achados de um `head_sha` diferente do head atual do PR.
- Nunca alterar regras de revisão para contornar achados.
- Nunca considerar um achado resolvido só porque a correção foi feita: só a nova revisão (Passo 9) e a reconciliação (Passo 10) dizem isso.
- Não acessar segredos nem ambientes de produção.
