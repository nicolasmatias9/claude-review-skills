---
name: reviewer
description: Revisa Pull Requests dos repositórios do petr-op-seqop (petr-op-seqop-backend, petr-op-seqop-frontend, petr-op-core) aplicando as convenções do time, o gate de cobertura do backend e um checklist de correção/segurança, e publica no PR o resultado com o status (corrigir, responder ou pronto para merge). Use quando o usuário pedir "revisar PR", "review do PR 123", "reviewer", "/seqop-review:reviewer", colar uma URL de PR do github.com/edgebr, ou quiser revisar a branch atual antes de abrir o PR.
---

# Reviewer — revisão de PR do petr-op-seqop

Revisão de PR com foco em **achados verificados**, não em lista de opiniões. Cada achado precisa apontar arquivo:linha, descrever o cenário concreto que quebra e sugerir a correção.

## Entrada

`/seqop-review:reviewer <alvo> [--no-post] [--run-tests] [--sha <sha>] [--json <arquivo>]`

- `<alvo>`: número do PR (`123`), URL (`https://github.com/edgebr/<repo>/pull/123`) ou vazio (= branch atual do repo em que se está trabalhando, comparada com a base).
- `--no-post`: não publicar no PR; o relatório fica só no chat. Sem a flag, toda revisão de PR é publicada no PR (Passo 7).
- `--run-tests`: além da leitura, rodar os testes/gate de cobertura localmente.
- `--sha <sha>`: revisar exatamente esse commit (SHA completo). Usado pelo `review-pr` para garantir que as duas revisões olham o mesmo código. Ver Passo 1.
- `--json <arquivo>`: além do relatório, gravar os achados nesse arquivo no formato de [references/contrato-achados.md](references/contrato-achados.md) (Passo 6).

Repositórios (todos em `edgebr/` no GitHub, clones locais na raiz do workspace):

| Pasta local | Stack | Base padrão |
|---|---|---|
| `petr-op-seqop-backend` | Spring Boot 3 / Java 17 / Maven, Flyway, Envers, Lombok, Spring Security | `develop` |
| `petr-op-seqop-frontend` | Angular 19, Vitest, `@edgebr/edge-components` | `develop` |
| `petr-op-core` | biblioteca compartilhada | `master` |

Se o alvo for só um número e não der para inferir o repo, perguntar qual dos três.

## Passo 1 — Coletar o PR (sem mexer no working tree)

Nunca fazer `checkout` na branch do PR no clone do usuário: ele pode ter trabalho em andamento.

```bash
R=edgebr/<repo>; N=<numero>
gh pr view $N -R $R --json number,title,body,author,baseRefName,headRefName,headRefOid,additions,deletions,changedFiles,files,labels,reviews,comments
gh pr diff $N -R $R --name-only                 # só a lista de arquivos; o diff vem do git abaixo
gh api repos/$R/pulls/$N/comments --paginate   # comentários inline anteriores
git -C <pasta> fetch origin pull/$N/head:review/pr-$N <baseRefName>
git -C <pasta> diff --output="$SCRATCH/pr-$N.diff" origin/<baseRefName>...review/pr-$N   # ou ...<sha> com --sha
```

**Se o `gh` não estiver autenticado** (`gh auth status` falha), não pedir token: seguir só com git, que usa as credenciais do Git Credential Manager. Faça o `fetch` acima, gere o diff com `git -C <pasta> diff origin/<base>...review/pr-$N` e use a mensagem dos commits como descrição. No relatório, avisar que descrição do PR e comentários anteriores não foram lidos, e que a publicação no PR fica indisponível até o usuário rodar `gh auth login`.

Para ler um arquivo inteiro na versão do PR: `git -C <pasta> show review/pr-$N:<caminho>`.
Para alvo = branch atual: `git -C <pasta> diff origin/<base>...HEAD`.

**Com `--sha <sha>`** (SHA fixado, normalmente pelo `review-pr`):
- Não criar nem apagar a ref `review/pr-$N`: outra revisão pode estar usando o mesmo PR em paralelo. Usar o próprio SHA como "versão do PR" em todos os comandos (`git show <sha>:<caminho>`, `git diff origin/<base>...<sha>`, worktree do Passo 4).
- Se o objeto não existir localmente (`git -C <pasta> cat-file -e <sha>^{commit}` falha), rodar `git -C <pasta> fetch origin pull/$N/head <baseRefName>` (sem criar ref) e conferir de novo.
- Gerar o diff com `git -C <pasta> diff --output=<arquivo> origin/<baseRefName>...<sha>` (o `gh pr diff` sempre mostra o head atual).
- Se o `headRefOid` atual do PR for diferente de `<sha>`, registrar isso no relatório ("PR recebeu commits depois do SHA revisado"), mas revisar o `<sha>` pedido.

Sem `--sha`, anotar o `headRefOid` lido no início: ele é o `head_sha` do relatório e do JSON.

Guardar o diff no scratchpad, não no repositório.

**Gravar diff e log sempre com `--output=<arquivo>` do próprio git, nunca com `>`.** Neste ambiente o Bash passa `git` e `gh` por um wrapper (`rtk`) que compacta a saída mesmo redirecionada: o arquivo sai como resumo, sem as linhas `diff --git`, e a revisão fica incompleta sem aviso. Conferir antes de revisar: o arquivo precisa ter uma linha `diff --git` por arquivo alterado.

## Passo 2 — Entender a intenção

Antes de julgar o código:
- Ler título, descrição e a chave Jira (`SEQOP-xxx`, `CRONO-xxx`) na branch/título/commits.
- Se houver spec relacionada em `openspec/changes/`, `openspec/specs/` ou `.specs/features/`, ler e usar como referência do comportamento esperado.
- Resumir em 2–3 frases o que o PR deveria fazer. Achados de "intenção vs implementação" são os mais valiosos.

## Passo 3 — Revisar

Ler cada arquivo alterado **no contexto** (arquivo inteiro na versão do PR, e chamadores/chamados relevantes), não só o hunk.

Para diffs grandes (> ~400 linhas ou > 15 arquivos), dividir por área e rodar subagentes em paralelo (ex.: `compound-engineering:review:correctness-reviewer`, `security-reviewer`, `testing-reviewer`, `kieran-typescript-reviewer` para o frontend), passando a eles o caminho do diff, os arquivos da área e o checklist da seção correspondente abaixo. Depois consolidar e verificar cada achado pessoalmente.

### 3.1 Geral (todos os repos)
- **Correção**: lógica invertida, null/undefined, off-by-one, casos de borda (lista vazia, data limite, fuso), tratamento de erro engolido, condição de corrida.
- **Segurança**: autorização ausente em endpoint novo, dados de outro contexto/usuário vazando, SQL/JPQL concatenado, input sem validação, segredos/credenciais no código, logs com dados sensíveis.
- **Escopo**: mudanças não relacionadas ao ticket, arquivos de debug/evidência commitados (`*.png`, `mcp-*.json`, dumps), código comentado, `console.log`/`System.out`.
- **Testes**: comportamento novo/alterado tem teste? Os asserts verificam o resultado ou só que "não quebrou"?
- **Commits**: padrão `tipo(CHAVE-JIRA): descrição` (ex.: `feat(CRONO-1030): ...`, `fix(SEQOP-895): ...`).

### 3.2 Backend (`petr-op-seqop-backend`)
- **Gate de cobertura**: 80% de cobertura no código alterado vs `origin/develop` (plugin `diff-coverage`). `service/` e `controller/` contam; `dto/`, `domain/`, `enums/`, `configuration/`, `filter/`, `mapper/`, `validation/`, `exceptions/`, `security/`, `event/`, `job/` são excluídos. Métodos só exercitados via `@Mock` + `verify(...)` contam como **não cobertos** — apontar quando a cobertura de um método novo depende só disso.
- **Flyway**: migration nova com versão única e maior que a última existente em `src/main/resources/db/migration/` (scripts só de dev vão em `db/migration-dev/`); nunca editar migration já mergeada; DDL compatível com Postgres e com o H2 dos testes de integração; tabelas auditadas pelo Envers precisam da `*_aud` correspondente.
- **JPA**: N+1 (lazy em loop), `@Transactional` ausente em escrita ou presente em método privado/self-invocation, `equals/hashCode` do Lombok (`@Data`) em entidade com relacionamento.
- **Camadas**: regras ARQ-B* de [../standards-spec-review/references/arquitetura.md](../standards-spec-review/references/arquitetura.md). As da seção "Violações" (ex.: controller acessando repository direto) entram como **Importante**; as "Heurísticas" (ex.: entidade JPA exposta no controller; 19 de 24 controllers importam entidade) entram só como **Sugestão**.
- **Spring Security**: endpoint novo com a mesma checagem de permissão/contexto dos vizinhos.
- **API**: mudança de contrato (campo removido/renomeado, status code) que quebra o frontend ou integrações (Cronoweb, APR).

### 3.3 Frontend (`petr-op-seqop-frontend`)
Aplicar **todas** as regras de `petr-op-seqop-frontend/CLAUDE.md` (ler o arquivo — ele é a fonte da verdade). Hoje elas são:
1. Reutilizar `@edgebr/edge-components` em vez de recriar controles com HTML cru.
2. Identificadores em inglês (textos de UI e comentários podem ficar em português).
3. Nenhuma cor hardcoded em SCSS — usar `_variables.scss`, `var(--...)` ou variável semântica.
4. OnPush: nada de mutação in-place de arrays/objetos; criar novas referências.
5. Inscrições pontuais encerradas com `take(1)` (ou equivalente); sem vazamento.
6. Lógica pura (formatação, parsing) em `util`/`.helper`, checando se já existe.
7. Comentário curto no início de métodos **complexos**; nomes descritivos. Complexo = corpo com mais de ~10 linhas, **ou** mais de um ponto de decisão além de uma guarda inicial (`if`/`else`, `switch`, ternário encadeado, laço, `try`/`catch`), **ou** cálculo/conversão não óbvio (índices, datas, posições), **ou** efeito colateral que o nome não revela. Getter/setter, delegação de uma linha e handler que só repassa um valor são **simples**: sem comentário é no máximo **Sugestão**, nunca Importante.
8. Componentes já grandes (ex.: `operational-sequence.component`) não recebem lógica nova — extrair componente ou helper.

Além disso: specs Vitest (não Jest) para lógica nova; tipagem sem `any` desnecessário; template sem chamada de função pesada em binding; regras de camada ARQ-F* de [../standards-spec-review/references/arquitetura.md](../standards-spec-review/references/arquitetura.md) (componente não faz HTTP; service não depende de componente), como **Importante**.

### 3.4 Design (todos os repos)
Princípios SOLID e code smells **não** são o foco desta skill (ela caça bugs). Só reportar um deles quando causar um problema concreto neste PR, como **Sugestão**, citando o nome da seção SOLID de [../standards-spec-review/references/smells.md](../standards-spec-review/references/smells.md). Para uma passada completa de design, usar `/seqop-review:standards-spec-review`. Clean Architecture não é cobrada: o projeto usa camadas tradicionais do Spring.

### 3.5 `petr-op-core`
Mudança em API pública da lib afeta backend e outros consumidores: apontar quebra de compatibilidade e se a versão foi incrementada.

## Passo 4 — (opcional, `--run-tests`) Verificar localmente

Rodar em uma **worktree separada** para não tocar no clone do usuário:

```powershell
$wt = "C:\Users\$env:USERNAME\wt-pr$N"   # caminho curto: o do scratchpad estoura o limite de 260 caracteres do Windows
git -C <pasta> worktree add --detach $wt <sha ou review/pr-$N>
# Frontend: reaproveitar as dependências do clone em vez de rodar npm install
New-Item -ItemType Junction -Path "$wt\node_modules" -Target "<pasta>\node_modules"
```

O `ng build` leva menos de 1 minuto e pode rodar em background enquanto as specs rodam.

- Backend (PowerShell): `$env:JAVA_HOME='C:\Program Files\Java\jdk-17'; .\mvnw.cmd -o clean test "diff-coverage:diffCoverage"`. Se houver falha de teste, repetir com `-Dmaven.test.failure.ignore=true` para ainda gerar o relatório (`target/delta-coverage-summary.json`). Erro `Unknown data type: "TINYINT"` em `*IntegrationTest` é do ambiente H2/Envers, não do PR.
- Frontend (PowerShell): `npx vitest run --config vitest.unit.config.ts <pastas alteradas>` e `npx ng build --configuration development` para checar templates.

Ao final, **primeiro** remover a junction (`cmd /c rmdir "$wt\node_modules"`, que apaga só o link), e só depois a worktree (`git worktree remove --force $wt`) e a ref `review/pr-$N`. Na ordem inversa, o `--force` pode apagar o `node_modules` do clone original.

## Passo 5 — Filtrar

Antes de reportar, cada achado precisa passar por:
- **Está no código alterado** (ou é diretamente causado por ele)? Problemas antigos fora do diff só entram como nota, nunca como bloqueante.
- **Consigo descrever entrada/estado → resultado errado?** Se não, é especulação: descartar ou rebaixar a "pergunta".
- **Já foi apontado** em comentário anterior do PR? Se foi resolvido, registrar na verificação. Se **continua pendente, manter como achado** (mesma severidade, no relatório e no JSON) com `previous_comment` apontando para o comentário anterior: pendente não é resolvido, e omitir faria o status de merge esconder o problema.
- Não reportar estilo que o lint/formatter já pega nem preferência pessoal sem regra do time.

## Passo 6 — Relatório

Escrever em português, nesta estrutura:

```markdown
## Review — <repo>#<N>: <título>
**Intenção:** <2–3 frases>
**Status:** CORRIGIR ANTES DO MERGE | RESPONDER ANTES DO MERGE | PRONTO PARA MERGE  (critério em references/publicacao-pr.md)

### Bloqueantes
1. `caminho/Arquivo.java:42` — <defeito em uma frase>
   - Cenário: <entrada/estado → resultado errado>
   - Sugestão: <correção concreta, com snippet curto se ajudar>

### Importantes
...
### Sugestões (não bloqueiam)
...
### Perguntas ao autor
...
### Verificação
- Testes/cobertura: <resultado do --run-tests ou "não executado">
- Comentários anteriores: <resolvidos / pendentes>
```

Severidades:
- **Bloqueante**: bug, falha de segurança, perda de dados, quebra de contrato, gate de cobertura reprovado, migration incorreta.
- **Importante**: violação das regras do time (CLAUDE.md do frontend), teste faltando para comportamento novo, risco de performance real.
- **Sugestão**: legibilidade, simplificação, nomes.

Se não houver achados, dizer isso claramente — não inventar problema para preencher seção.

O relatório informa o commit revisado: `**Commit revisado:** <head_sha>` logo abaixo da intenção.

### Saída estruturada (`--json <arquivo>`)

Gravar, com a ferramenta Write, um arquivo no formato de [references/contrato-achados.md](references/contrato-achados.md) com `"reviewer": "reviewer"`, o `head_sha` revisado e **os mesmos achados do relatório**, convertidos pelo de-para do contrato (Bloqueante → `blocker`, Importante → `important`, Sugestão → `suggestion`, Pergunta → `question`). IDs `REV-001`, `REV-002`… na ordem do relatório. Validar o JSON (comando no contrato) antes de terminar. Com `--json`, responder no fim com o caminho do arquivo e uma linha de resumo (status e contagem por severidade).

## Passo 7 — Publicar o resultado no PR

**Obrigatório** quando o alvo é um PR e não houver `--no-post`. Seguir [references/publicacao-pr.md](references/publicacao-pr.md): status (CORRIGIR / RESPONDER / PRONTO PARA MERGE), formato do review, comentários inline e comando de publicação. Publicar sem pedir confirmação: é regra do time que toda revisão de PR seja publicada no PR.

Para esta skill, **Bloqueantes e Importantes** exigem correção; **Sugestões** são opcionais.

## Regras

- Nunca commitar, fazer push, aprovar ou fazer merge.
- Não alterar arquivos do PR; a skill só revisa. Se o usuário quiser as correções aplicadas, isso é um passo separado.
- Limpar ao final as refs e worktrees temporárias **que esta execução criou** (com `--sha`, nenhuma ref é criada).
- Rodando dentro de um subagente (ex.: chamado pelo `review-pr`): não invocar `review-pr` nem `standards-spec-review`. Se não for possível criar subagentes para diffs grandes, revisar diretamente, área por área.
