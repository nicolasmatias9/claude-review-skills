# Contrato de achados (saída estruturada das revisões)

Formato JSON que `reviewer` e `standards-spec-review` gravam quando recebem `--json <arquivo>`, e que o `review-pr` consolida. É o contrato que permite automatizar as próximas etapas (correção e reconciliação); por isso os nomes de campo e os valores dos enums não mudam sem subir `schema_version`.

O relatório humano de cada skill continua existindo. O JSON é uma saída **adicional**, gerada a partir dos mesmos achados que sobreviveram à filtragem/verificação. Nunca incluir no JSON um achado que não está no relatório, nem o contrário.

## Arquivo produzido por uma skill

```json
{
    "schema_version": "1.0",
    "reviewer": "reviewer",
    "repository": "edgebr/petr-op-seqop-frontend",
    "pull_request": 301,
    "base_ref": "develop",
    "base_sha": "69ff9806f3f9038142a1ef157ab00d9d8792b0f0",
    "head_sha": "f76cf091c3b7a75e788f6959b47f3bb79038fad4",
    "generated_at": "2026-11-03T15:20:00Z",
    "merge_status": "PRONTO_PARA_MERGE",
    "verification": {
        "run_tests": true,
        "checks": ["vitest drop-list: 151/151", "ng build development: ok"],
        "not_verified": ["comportamento visual no navegador"],
        "discarded_in_verification": 0
    },
    "spec": null,
    "requirements": [],
    "findings": [
        {
            "id": "REV-001",
            "source": "reviewer",
            "category": "bug",
            "severity": "suggestion",
            "file": "src/app/operational-sequence/document/drop-list/draggable/draggable.component.html",
            "line": 331,
            "symbol": "grab-handle (mouseup)",
            "rule": null,
            "scenario": "Mouse pressionado no handle e solto fora dele sem iniciar drag deixa canDrag=true e esconde toolbar e navegação.",
            "evidence": "(mouseup)=\"canDrag = false\"",
            "suggested_action": "Resetar canDrag no onSelect ou ouvir mouseup no document enquanto canDrag for true.",
            "actionable": true,
            "confidence": "medium",
            "requirement_id": null,
            "fingerprint": "reviewer|bug|src/app/operational-sequence/document/drop-list/draggable/draggable.component.html|grab-handle|candrag-preso-apos-mouseup-externo",
            "status": "open",
            "resolution_commit": null
        }
    ]
}
```

### Campos do arquivo

| Campo | Obrigatório | Valor |
|---|---|---|
| `schema_version` | sim | `"1.0"` |
| `reviewer` | sim | `"reviewer"` ou `"standards-spec-review"` |
| `repository` | sim | `owner/repo` |
| `pull_request` | sim, se o alvo for PR | número; `null` quando o alvo for branch/commit |
| `base_ref`, `base_sha` | sim | branch base e o SHA do merge-base usado no diff three-dot (`git merge-base origin/<base> <head_sha>`) |
| `head_sha` | **sim** | SHA completo (40 caracteres) do commit revisado. Nunca abreviado, nunca nome de branch |
| `generated_at` | sim | ISO-8601 em UTC |
| `merge_status` | sim | `CORRIGIR_ANTES_DO_MERGE`, `RESPONDER_ANTES_DO_MERGE` ou `PRONTO_PARA_MERGE`, pelo mesmo critério do relatório humano (`publicacao-pr.md`) |
| `verification` | sim | o que foi conferido (`checks`), o que não foi (`not_verified`), se rodou testes, quantos achados foram descartados na verificação |
| `spec` | só `standards-spec-review` | origem da spec usada (caminho, `"pr-description"`, `"ticket-colado"`) ou `null` se o eixo Spec foi pulado |
| `requirements` | só `standards-spec-review` | todos os requisitos avaliados, inclusive os atendidos: `{ "id": "REQ-01", "quote": "<citação literal>", "status": "met|partial|missing|divergent", "where": "<arquivo:linha ou o que falta>" }` |
| `findings` | sim | lista de achados (pode ser vazia) |

## Campos de um achado

| Campo | Valor |
|---|---|
| `id` | Prefixo por origem + sequência no arquivo: `REV-001` (reviewer), `STD-001` (eixo Standards), `SPC-001` (eixo Spec) |
| `source` | `reviewer`, `standards` ou `spec` |
| `category` | ver taxonomia abaixo |
| `severity` | `blocker`, `important`, `suggestion` ou `question` |
| `file` | caminho relativo à raiz do repositório; `null` se o achado não tem arquivo (ex.: requisito ausente) |
| `line` | linha no lado novo do diff; `null` se não houver. **Não** serve como identificador |
| `symbol` | classe, método, componente ou seletor que contém o problema (ex.: `SequenceEmailNotificationService.sendSequenceRevisionNotification`); `null` se não se aplica |
| `rule` | regra citada, quando houver: `ARQ-B3`, `CLAUDE.md#7`, `smell:Mysterious Name`, `SOLID-D`; `null` para bug sem regra escrita |
| `scenario` | entrada/estado → resultado errado (reviewer); para Standards/Spec, a frase do problema |
| `evidence` | trecho literal do diff, da regra ou da spec que sustenta o achado |
| `suggested_action` | correção sugerida; quem for corrigir não é obrigado a segui-la |
| `actionable` | `true` se existe correção objetiva no código; `false` para perguntas, escopo extra e decisões de produto |
| `confidence` | `high` (verificado no código/backend/teste), `medium` (raciocínio sólido, sem execução), `low` (plausível, a confirmar) |
| `requirement_id` | `REQ-xx` relacionado (eixo Spec); `null` nos demais |
| `previous_comment` | URL do comentário anterior no PR quando o achado já tinha sido apontado e **continua pendente**; `null` nos demais. Achado pendente nunca é omitido |
| `fingerprint` | identificador estável (ver abaixo) |
| `status` | sempre `"open"` na revisão; os demais valores são para a reconciliação |
| `resolution_commit` | `null` na revisão |

## Taxonomia

### Categorias

| `category` | Uso |
|---|---|
| `bug` | lógica errada, null, borda, erro engolido, corrida |
| `security` | autorização, vazamento de dados, injeção, segredo |
| `test` | teste ausente ou que não verifica o comportamento; cobertura abaixo do gate |
| `contract` | quebra de API/contrato com outro sistema ou dependência (inclui versão de dependência) |
| `migration` | Flyway/schema/auditoria Envers |
| `architecture` | regra de camada (`ARQ-*`) |
| `convention` | regra documentada do repositório (`CLAUDE.md`, `.editorconfig`, guia de testes) |
| `code_smell` | smell do baseline |
| `solid` | princípio SOLID do baseline |
| `spec_missing` | requisito ausente |
| `spec_partial` | requisito parcial |
| `spec_divergence` | requisito implementado de forma diferente do pedido |
| `scope_creep` | escopo extra |
| `other` | qualquer outro, com `rule: null` |

### De-para das severidades atuais

| Skill / eixo | Classificação no relatório humano | `severity` | `actionable` |
|---|---|---|---|
| reviewer | Bloqueante | `blocker` | `true` |
| reviewer | Importante | `important` | `true` |
| reviewer | Sugestão | `suggestion` | `true` |
| reviewer | Pergunta ao autor | `question` | `false` |
| Standards | Violação | `important` | `true` |
| Standards | Heurística (smell/SOLID/ARQ heurística) | `suggestion` | `true` |
| Spec | Ausente / Parcial / Divergente | `important` | `true`, ou `false` se a correção depende de decisão de produto |
| Spec | Escopo extra | `suggestion` | `false` |

`question` e `suggestion` nunca bloqueiam merge sozinhos.

## Fingerprint

Identifica **o mesmo problema** entre rodadas e commits, para a reconciliação (`resolved`, `open`…). Não usa número de linha.

Formato, tudo em minúsculas, separado por `|`:

```
<source>|<rule ou category>|<file ou "-">|<symbol ou "-">|<slug>
```

- `rule` quando houver regra citada (é o componente mais estável); senão `category`.
- `slug`: 2 a 6 palavras em kebab-case descrevendo a **natureza** do problema, não a correção (`candrag-preso-apos-mouseup-externo`, `dependencia-snapshot-nao-publicada`). Sem acentos.
- Para achados do eixo Spec, usar `requirement_id` como `symbol`.

Limite conhecido: o `slug` é escrito pelo modelo e pode variar entre rodadas para o mesmo problema. Achados com `rule` preenchida tendem a manter o fingerprint; os demais precisam de conferência semântica na reconciliação (mesmo arquivo, mesmo símbolo, mesmo risco) em vez de só igualdade de texto.

## Regras de integridade

- O JSON é sempre do `head_sha` informado. Se a skill recebeu `--sha` e o código revisado não é esse commit, ela aborta sem gerar JSON.
- Linha não identifica achado; `fingerprint` identifica.
- Achados de uma skill não entram na análise da outra. Só o `review-pr` lê os dois arquivos, e só depois que ambos terminaram.
- Gravar o arquivo com a ferramenta Write (UTF-8 sem BOM). Validar antes de entregar: `python -c "import json,sys; json.load(open(sys.argv[1], encoding='utf-8'))" <arquivo>` ou `node -e "JSON.parse(require('fs').readFileSync(process.argv[1],'utf8'))" <arquivo>`.
