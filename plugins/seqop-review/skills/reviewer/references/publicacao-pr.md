# Regra: publicar o resultado da revisão no PR

Vale para `/seqop-review:reviewer` e `/seqop-review:standards-spec-review`. **Toda revisão cujo alvo é um PR termina com o resultado publicado no próprio PR**, para o dev saber, sem precisar perguntar, se deve corrigir algo ou se pode aceitar e mergear.

É regra do time que toda revisão de PR seja publicada: **não pedir confirmação** antes de postar. A exceção é a flag `--no-post`, que gera o relatório só no chat.

Se o alvo não for um PR (branch, commit, `HEAD~5`…), não há onde publicar: o relatório fica só no chat.

## Status

Todo review publicado começa com **exatamente um** destes status, decidido pelos achados que sobreviveram à verificação:

| Status | Quando |
|---|---|
| **CORRIGIR ANTES DO MERGE** | Há pelo menos um achado que precisa de correção (tabela abaixo). |
| **RESPONDER ANTES DO MERGE** | Nada a corrigir, mas há pergunta ao autor cuja resposta pode mudar o código. |
| **PRONTO PARA MERGE** | Nada a corrigir nem a responder. Sugestões e heurísticas podem existir, e são opcionais. |

O que conta como "precisa de correção":

| Skill | Exige correção | Opcional |
|---|---|---|
| `/seqop-review:reviewer` | Bloqueantes e Importantes | Sugestões |
| `/seqop-review:standards-spec-review` | Violações de Standards; requisitos Parciais, Ausentes ou Divergentes na Spec | Heurísticas (smells) e escopo extra |

Na dúvida entre dois status, usar o mais restritivo.

## Formato do review

```markdown
## Status: <CORRIGIR ANTES DO MERGE | RESPONDER ANTES DO MERGE | PRONTO PARA MERGE>

<Uma frase dizendo o que o dev precisa fazer: "Corrija os 2 itens obrigatórios abaixo e peça nova revisão." | "Responda a pergunta abaixo; se a resposta não exigir mudança, pode mergear." | "Pode aceitar e mergear. As sugestões abaixo são opcionais.">

### Obrigatório
1. `arquivo:linha` — <problema> — <correção sugerida>
(ou "Nada.")

### Perguntas
(só se houver)

### Opcional
(sugestões / heurísticas; ou omitir a seção)

### Verificado
- <o que foi conferido: testes, build, cobertura, backend, spec usada>
- Não verificado: <o que ficou de fora, ex.: comportamento visual>

---
<sub>Revisão automática via `/<nome-da-skill>`. Achados conferidos antes da publicação; em caso de discordância, responda neste PR.</sub>
```

Regras de escrita:
- Português, direto, sem emojis.
- O dev precisa conseguir agir só com o texto do review: cada item obrigatório traz `arquivo:linha`, o problema e a correção.
- Não mencionar `@usuario`.
- Só afirmar como fato o que foi verificado nesta revisão. O que é inferência vai como risco ("não há garantia de…") ou em "Não verificado". Reler o texto final procurando afirmações sem verificação antes de publicar, porque o review é público e o dev vai agir com base nele.

## Como publicar

1. Usar sempre `event: "COMMENT"`. O status vai no texto. Não usar `APPROVE` nem `REQUEST_CHANGES`: o GitHub não deixa o autor aprovar o próprio PR, e o merge é decisão do time.
2. Itens com linha presente no diff viram também **comentário inline** (problema + correção). Os demais ficam só no corpo.
3. Montar um JSON no scratchpad e enviar:
   ```bash
   gh api repos/<owner>/<repo>/pulls/<N>/reviews --method POST --input <arquivo.json>
   ```
   ```json
   {
       "commit_id": "<headRefOid>",
       "event": "COMMENT",
       "body": "<corpo no formato acima>",
       "comments": [{ "path": "<arquivo>", "line": <linha no lado novo>, "side": "RIGHT", "body": "<problema + correção>" }]
   }
   ```
   Sem comentários inline, `gh pr review <N> -R <owner>/<repo> --comment --body-file <arquivo.md>` basta.

   Gravar o JSON com a ferramenta Write. No Windows PowerShell 5.1, `Out-File -Encoding utf8` grava BOM e o GitHub responde "Problems parsing JSON". Se precisar gerar pelo PowerShell, usar `[IO.File]::WriteAllText(<arquivo>, <texto>, (New-Object System.Text.UTF8Encoding($false)))`.

   Para corrigir um review já publicado: `PUT repos/<owner>/<repo>/pulls/<N>/reviews/<review_id>` com `{"body": ...}`. Para um comentário inline: `PATCH repos/<owner>/<repo>/pulls/comments/<comment_id>`.
4. Se o `gh` não estiver autenticado, não publicar: avisar o usuário que precisa rodar `gh auth login` e mostrar o texto que seria postado.
5. Depois de publicar, informar ao usuário o status e o link do review (`html_url` da resposta).
