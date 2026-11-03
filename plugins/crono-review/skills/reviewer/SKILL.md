---
name: reviewer
description: Revisa Pull Requests dos repositórios do Crono (petr-op-crono-backend, petr-op-crono-frontend) aplicando as convenções do time, as regras de camada e um checklist de correção/segurança, e publica no PR o resultado com o status (corrigir, responder ou pronto para merge). Use quando o usuário pedir "revisar PR", "review do PR 123", "reviewer", "/crono-review:reviewer", colar uma URL de PR do github.com/edgebr/petr-op-crono-*, ou quiser revisar a branch atual antes de abrir o PR.
---

# Reviewer — revisão de PR do Crono

Revisão de PR com foco em **achados verificados**, não em lista de opiniões. Cada achado precisa apontar arquivo:linha, descrever o cenário concreto que quebra e sugerir a correção.

## Entrada

`/crono-review:reviewer <alvo> [--no-post] [--run-tests]`

- `<alvo>`: número do PR (`123`), URL (`https://github.com/edgebr/<repo>/pull/123`) ou vazio (= branch atual do repo em que se está trabalhando, comparada com a base).
- `--no-post`: não publicar no PR; o relatório fica só no chat. Sem a flag, toda revisão de PR é publicada no PR (Passo 7).
- `--run-tests`: além da leitura, rodar testes e build localmente.

Repositórios (em `edgebr/` no GitHub, clones locais na raiz do workspace `crono`):

| Pasta local | Stack | Base padrão |
|---|---|---|
| `petr-op-crono-backend` | Spring Boot 4.1.1 / Java 17 / Maven, Flyway, Envers, Lombok, depende do `petr-op-core` | `develop` |
| `petr-op-crono-frontend` | Angular 21, Vitest, Playwright, `@edgebr/edge-components` | `develop` |

Se o alvo for só um número e não der para inferir o repo, perguntar qual dos dois.

> O `CLAUDE.md` do backend cita Spring Boot 3.4 e "JaCoCo com mínimo 0.00"; o `pom.xml` diz 4.1.1 e 80% de linhas. O próprio `CLAUDE.md` manda preferir o `pom.xml`, então esta skill segue o `pom.xml`.

## Passo 1 — Coletar o PR (sem mexer no working tree)

Nunca fazer `checkout` na branch do PR no clone do usuário: ele pode ter trabalho em andamento.

```bash
R=edgebr/<repo>; N=<numero>
gh pr view $N -R $R --json number,title,body,author,baseRefName,headRefName,headRefOid,additions,deletions,changedFiles,files,labels,reviews,comments
gh pr diff $N -R $R > "$SCRATCH/pr-$N.diff"
gh api repos/$R/pulls/$N/comments --paginate   # comentários inline anteriores
git -C <pasta> fetch origin pull/$N/head:review/pr-$N <baseRefName>
```

**Se o `gh` não estiver autenticado** (`gh auth status` falha), não pedir token: seguir só com git, que usa as credenciais do Git Credential Manager. Faça o `fetch` acima, gere o diff com `git -C <pasta> diff origin/<base>...review/pr-$N` e use a mensagem dos commits como descrição. No relatório, avisar que descrição do PR e comentários anteriores não foram lidos, e que a publicação no PR fica indisponível até o usuário rodar `gh auth login`.

Para ler um arquivo inteiro na versão do PR: `git -C <pasta> show review/pr-$N:<caminho>`.
Para alvo = branch atual: `git -C <pasta> diff origin/<base>...HEAD`.

Guardar o diff no scratchpad, não no repositório.

## Passo 2 — Entender a intenção

Antes de julgar o código:
- Ler título, descrição (o frontend usa o template `.github/pull_request_template.md`: Objetivo, Descrição, Depende de, Test plan) e a chave Jira (`CRONO-xxx`) na branch/título/commits.
- Se houver regra de negócio documentada em `petr-op-crono-frontend/.Codex/knowledge-base/business-rules/` ou em `docs/` de cada repo, usar como referência do comportamento esperado.
- Se a seção "Depende de" citar outro PR (backend, core, edge-components), conferir o estado dele.
- Resumir em 2–3 frases o que o PR deveria fazer. Achados de "intenção vs implementação" são os mais valiosos.

## Passo 3 — Revisar

Ler cada arquivo alterado **no contexto** (arquivo inteiro na versão do PR, e chamadores/chamados relevantes), não só o hunk.

Para diffs grandes (> ~400 linhas ou > 15 arquivos), dividir por área e rodar subagentes em paralelo (ex.: `compound-engineering:review:correctness-reviewer`, `security-reviewer`, `testing-reviewer`, `kieran-typescript-reviewer` para o frontend), passando a eles o caminho do diff, os arquivos da área e o checklist da seção correspondente abaixo. Depois consolidar e verificar cada achado pessoalmente.

### 3.1 Geral (todos os repos)
- **Correção**: lógica invertida, null/undefined, off-by-one, casos de borda (lista vazia, data limite, fuso), tratamento de erro engolido, condição de corrida.
- **Segurança**: dados de outro contexto/usuário vazando, SQL/JPQL concatenado, input sem validação, segredos/credenciais no código, logs com dados sensíveis.
- **Escopo**: mudanças não relacionadas ao ticket, arquivos de debug/evidência commitados, dumps (`dump.sql`, `update.sql`), código comentado, `console.log`/`System.out`.
- **Testes**: comportamento novo/alterado tem teste? Os asserts verificam o resultado ou só que "não quebrou"?
- **Commits**: padrão `tipo(escopo): descrição`, com a chave quando houver ticket (ex.: `feat(CRONO-1030): ...`, `fix(CRONO-966): ...`, `fix(schedule): ...`).

### 3.2 Backend (`petr-op-crono-backend`)
Ler `petr-op-crono-backend/CLAUDE.md`: é a fonte das convenções. Pontos que mais aparecem em revisão:
- **Migrations**: SQL novo em `src/main/resources/db/migration/` com nome `V<yyyyMMddHHmmss>__Descricao.sql`; transformação com lógica em `src/main/java/db/migration/` (`V<yyyyMMddHHmmss>__NomeDaClasse.java`). Nunca criar no formato legado `V<n>_2__`. `ddl-auto=none`: toda mudança de schema precisa de migration. Conferir se já existe migration programática tocando a mesma tabela. Nunca editar migration já mergeada. Tabela com `@Audited` precisa da `*_aud`.
- **Código gerado**: nada em `br.com.petrobras.fwca.**` pode ser commitado (gerado pelo `jaxws-maven-plugin`).
- **Changelog**: mudança com impacto funcional atualiza `CHANGELOG.CRONO.md` (Keep a Changelog, com `#CRONO-XXX`) e `schedule.version-number` em `application.properties`. Ausência disso é **Importante**.
- **Segurança/permissão**: autorização é imperativa no controller (`securityService.hasPermission(SecurityPermissions.X)`), não `@PreAuthorize`. Hoje poucos endpoints checam (4 chamadas em 3 controllers), então: endpoint novo deve seguir o que os endpoints vizinhos do mesmo controller fazem; se a feature tem permissão própria e o endpoint não checa, é **Pergunta** ao autor. Permissão nova vira constante em `security/constants/SecurityPermissions` no formato `CRONO:<MÓDULO>:<AÇÃO>:<OPERAÇÃO>`.
- **Sincronização onshore/offshore**: mudança em entidade que estende `SynchronizableEntity` ou em `ScheduleSynchronizationService` deve respeitar a flag `synchronizing`.
- **Datas**: a aplicação roda em UTC; conversões de fuso explícitas devem ser justificadas.
- **Entidades**: `equals`/`hashCode` por `uuid`, não por `id`.
- **JPA**: N+1 (lazy em loop), `@Transactional` ausente em escrita ou presente em método privado/self-invocation.
- **Camadas**: regras ARQ-B* de [../standards-spec-review/references/arquitetura.md](../standards-spec-review/references/arquitetura.md). As da seção "Violações" entram como **Importante**; as "Heurísticas" (ex.: controller acessando repository, que hoje é o caso de 14 de 25 controllers) só como **Sugestão**.
- **Testes**: unitário `*Test` com `MockitoExtension` no mesmo pacote; integração `*IT` no pacote `integration` (H2); controller com `MockMvcBuilders.standaloneSetup`; método `should[Comportamento]_when[Condição]()` com `@DisplayName` em inglês.
- **Cobertura**: o JaCoCo exige 80% de linhas e 10% de branches **no projeto inteiro** no `verify` (exclui `lgc`, `schedule`, `domain`, `job`, `migration`, `fwca`). Código novo em `service/` ou `controller/` sem teste é **Importante**.
- **API**: mudança de contrato (campo removido/renomeado, status code) que quebra o frontend ou integrações (SEQOP, Atlas, PocoWeb/PWO).

### 3.3 Frontend (`petr-op-crono-frontend`)
O repositório não tem `CLAUDE.md` próprio; as fontes são `.editorconfig` (4 espaços, aspas simples em `.ts`), `README.md` e `docs/TESTING-TOOLS.md` (atenção: esse doc ainda cita Karma, mas o `npm test` atual é `vitest run`). Checar:
- Regras de camada ARQ-F* de [../standards-spec-review/references/arquitetura.md](../standards-spec-review/references/arquitetura.md) (componente não faz HTTP; service não depende de componente fora de modal/popover), como **Importante**.
- Specs Vitest para lógica nova; tipagem sem `any` desnecessário; template sem chamada de função pesada em binding.
- Inscrições encerradas (`take(1)`, `takeUntilDestroyed` ou equivalente); sem vazamento.
- Reuso de `@edgebr/edge-components` antes de recriar controle de UI (Sugestão: não há regra escrita no repo).
- Dependências declaradas na seção "Depende de" do PR (versões de `@edgebr/*`, PR do backend).

### 3.4 Design (todos os repos)
Princípios SOLID e code smells **não** são o foco desta skill (ela caça bugs). Só reportar um deles quando causar um problema concreto neste PR, como **Sugestão**, citando a seção SOLID de [../standards-spec-review/references/smells.md](../standards-spec-review/references/smells.md). Para uma passada completa de design, usar `/crono-review:standards-spec-review`. Clean Architecture não é cobrada: o projeto usa camadas tradicionais do Spring.

## Passo 4 — (opcional, `--run-tests`) Verificar localmente

Rodar em uma **worktree separada** para não tocar no clone do usuário:

```powershell
$wt = "C:\Users\$env:USERNAME\wt-crono-pr$N"   # caminho curto: o do scratchpad estoura o limite de 260 caracteres do Windows
git -C <pasta> worktree add $wt review/pr-$N
# Frontend: reaproveitar as dependências do clone em vez de rodar npm install
New-Item -ItemType Junction -Path "$wt\node_modules" -Target "<pasta>\node_modules"
```

- Backend (PowerShell): `$env:JAVA_HOME='C:\Program Files\Java\jdk-17'; .\mvnw.cmd -o verify -Pdev`. O perfil `-Pdev` é obrigatório (sem ele o build tenta baixar o WSDL do CAV4 pela rede Petrobras). Um teste: `.\mvnw.cmd -o test -Pdev -Dtest=Classe`. Relatório em `target/site/jacoco-aggregate/`.
- Frontend (PowerShell): `npx vitest run <pastas alteradas>` e `npx ng build --configuration development` para checar templates.

Ao final, **primeiro** remover a junction (`cmd /c rmdir "$wt\node_modules"`, que apaga só o link), e só depois a worktree (`git worktree remove --force $wt`) e a ref `review/pr-$N`. Na ordem inversa, o `--force` pode apagar o `node_modules` do clone original.

## Passo 5 — Filtrar

Antes de reportar, cada achado precisa passar por:
- **Está no código alterado** (ou é diretamente causado por ele)? Problemas antigos fora do diff só entram como nota, nunca como bloqueante.
- **Consigo descrever entrada/estado → resultado errado?** Se não, é especulação: descartar ou rebaixar a "pergunta".
- **Já foi apontado** em comentário anterior do PR? Se sim, só dizer se foi resolvido ou não.
- Não reportar estilo que o `.editorconfig` já pega nem preferência pessoal sem regra do time.

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
- **Bloqueante**: bug, falha de segurança, perda de dados, quebra de contrato, migration incorreta, código gerado (`fwca`) commitado.
- **Importante**: violação das regras documentadas do time (`CLAUDE.md` do backend, regras ARQ de violação), changelog ausente em mudança funcional, teste faltando para comportamento novo, risco de performance real.
- **Sugestão**: legibilidade, simplificação, nomes, heurísticas.

Se não houver achados, dizer isso claramente — não inventar problema para preencher seção.

## Passo 7 — Publicar o resultado no PR

**Obrigatório** quando o alvo é um PR e não houver `--no-post`. Seguir [references/publicacao-pr.md](references/publicacao-pr.md): status (CORRIGIR / RESPONDER / PRONTO PARA MERGE), formato do review, comentários inline e comando de publicação. Publicar sem pedir confirmação: é regra do time que toda revisão de PR seja publicada no PR.

Para esta skill, **Bloqueantes e Importantes** exigem correção; **Sugestões** são opcionais.

## Regras

- Nunca commitar, fazer push, aprovar ou fazer merge.
- Não alterar arquivos do PR; a skill só revisa. Se o usuário quiser as correções aplicadas, isso é um passo separado.
- Limpar refs/worktrees temporárias ao final.
