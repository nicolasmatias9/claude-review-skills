# Regras de arquitetura do petr-op-seqop

Regras de camada da arquitetura que o projeto **realmente usa**: camadas tradicionais do Spring no backend e componente → service no frontend. **Não** é Clean Architecture: o projeto não adota casos de uso isolados, domínio independente de framework nem ports & adapters, então nada disso é cobrado.

Cada regra foi medida no código em 2026-11-03. Só virou **violação** o que o código já segue quase sempre. O que o código atual descumpre com frequência ficou como **heurística**, para não marcar como erro um padrão que o próprio projeto pratica. Se o padrão do código mudar, medir de novo antes de mudar a classificação.

Todas valem **só para linhas adicionadas ou alteradas no diff**. Código antigo que já descumpre a regra não entra.

## Backend (`petr-op-seqop-backend`)

Camadas: `controller` → `service` → `repository`, com entidades em `domain/model` e DTOs em `domain/dto`.

### Violações

| Regra | Medição | Sinal no diff |
|---|---|---|
| **ARQ-B1** Service não depende de controller. | 0 de 36 services importam `controller`. | `import ...sequence.controller.` num arquivo de `service/`. |
| **ARQ-B2** Repository não depende de service nem de controller. | 0 ocorrências. | `import ...sequence.service.` ou `...controller.` em `repository/`. |
| **ARQ-B3** Controller não acessa repository direto: passa por um service. | 19 de 24 controllers seguem. | Controller novo, ou método novo de controller, que injeta/chama `*Repository`. |
| **ARQ-B5** Acesso a banco só em repository. | 0 controllers usam `EntityManager`/`JdbcTemplate`. | `EntityManager`, `JdbcTemplate` ou SQL montado fora de `repository/`. |

### Heurísticas

| Regra | Medição | Sinal no diff |
|---|---|---|
| **ARQ-B6** Controller novo recebe e devolve DTO, não entidade JPA. | 19 de 24 controllers importam entidade de `domain/model` (indicativo de uso direto; não confere endpoint a endpoint). Hoje não é regra. | Endpoint novo com entidade no `@RequestBody` ou no `ResponseEntity<...>`. Sugerir DTO, principalmente se a entidade tem relacionamento lazy ou dado sensível. |
| **ARQ-B4** Controller sem regra de negócio: valida entrada, delega ao service e monta a resposta. | Não medido (não dá para medir com busca textual), por isso fica como heurística. | Método de controller com laço, cálculo ou decisão de negócio, ou orquestrando mais de uma escrita. |
| **ARQ-B7** Injeção por construtor (`@RequiredArgsConstructor` + `final`) em classe nova. | `@Autowired` em campo ainda é maioria (41 arquivos contra 14). | `@Autowired` em campo numa classe criada no diff. |

## Frontend (`petr-op-seqop-frontend`)

Camadas: componente (apresentação e estado de tela) → service em `shared/` ou na pasta da feature (HTTP e estado compartilhado) → `util`/`.helper` (lógica pura).

### Violações

| Regra | Medição | Sinal no diff |
|---|---|---|
| **ARQ-F1** Componente não faz HTTP: chama um service. | 163 de 165 componentes seguem (exceções: `home` e `release-notes`). | `HttpClient` injetado ou usado num `*.component.ts`. |
| **ARQ-F2** Service não depende de componente, exceto o componente de um modal que o service abre. | 76 de 77 services não importam componente; o único que importa (`checklist-operation.service`) abre um modal. | `import` de `*.component` num `*.service.ts` que não seja de modal. |

As demais regras de frontend (componentes enxutos, lógica pura em helper, reuso do edge-components) já estão no `CLAUDE.md` do repositório e não são repetidas aqui.

## Como reportar

- **Violação:** citar este arquivo e o código da regra (ex.: `arquitetura.md — ARQ-B3`) mais o hunk literal. No `/seqop-review:reviewer` entra como **Importante**; na `standards-spec-review`, em **Violações**.
- **Heurística:** citar o código da regra mais o hunk. Entra como **Sugestão** / **Heurística**, nunca bloqueia o merge.
