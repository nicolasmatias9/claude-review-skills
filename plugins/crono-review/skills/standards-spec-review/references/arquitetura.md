# Regras de arquitetura do Crono

Regras de camada da arquitetura que o projeto **realmente usa**: camadas tradicionais do Spring no backend ("camada no topo, subdomínio como subpacote", conforme o `CLAUDE.md` do backend) e componente → service no frontend. **Não** é Clean Architecture: o projeto não adota casos de uso isolados, domínio independente de framework nem ports & adapters, então nada disso é cobrado.

Cada regra foi medida no código em 2026-11-03. Só virou **violação** o que o código já segue quase sempre ou o que o `CLAUDE.md` do backend documenta explicitamente. O que o código atual descumpre com frequência e não está documentado ficou como **heurística**. Se o padrão do código mudar, medir de novo antes de mudar a classificação.

Todas valem **só para linhas adicionadas ou alteradas no diff**. Código antigo que já descumpre a regra não entra.

## Backend (`petr-op-crono-backend`)

Pacote raiz `br.com.petrobras.operacao.crono`. Camadas: `controller` → `service` → `repository`, com entidades em `domain/model` e DTOs em `domain/dto`, cada camada com subpacote por subdomínio (`service/dashboard`, `repository/anomaly`…).

### Violações

| Regra | Medição | Sinal no diff |
|---|---|---|
| **ARQ-B1** Service não depende de controller. | 0 de 32 services importam `controller`. | `import ...crono.controller.` num arquivo de `service/`. |
| **ARQ-B2** Repository não depende de service nem de controller. | 0 de 26 repositories. | `import ...crono.service.` ou `...crono.controller.` em `repository/`. |
| **ARQ-B5** Acesso a banco só em repository (ou nas configurações de datasource, como `AidaDataSourceConfiguration`). | 0 de 25 controllers usam `EntityManager`/`JdbcTemplate`. | `EntityManager`, `JdbcTemplate` ou SQL montado em controller ou service. |
| **ARQ-B8** Classe nova usa injeção por construtor. | Documentado no `CLAUDE.md` ("injeção por construtor"), embora o código antigo tenha `@Autowired` em campo em pelo menos 46 lugares. Por ser regra escrita do time, vale como violação **só para classes criadas no diff**. | `@Autowired` em campo numa classe nova. Em classe antiga que só ganhou um campo, é heurística. |
| **ARQ-B9** Organização "camada no topo, subdomínio como subpacote". | Documentado no `CLAUDE.md`. | Arquivo novo fora da camada certa, ou camada criada dentro de subdomínio (ex.: `dashboard/service/` em vez de `service/dashboard/`). |

### Heurísticas

| Regra | Medição | Sinal no diff |
|---|---|---|
| **ARQ-B3** Controller acessa banco via service, não via repository. | 14 de 25 controllers importam repository: não é regra hoje. | Endpoint novo que chama `*Repository` direto quando já existe service do subdomínio. |
| **ARQ-B4** Controller sem regra de negócio: valida entrada, checa permissão, delega ao service e monta a resposta. | Não medido (não dá para medir com busca textual). | Método de controller com laço, cálculo ou decisão de negócio, ou orquestrando mais de uma escrita. |
| **ARQ-B6** Controller novo recebe e devolve DTO, não entidade JPA. | 16 de 25 controllers importam entidade de `domain/model` (indicativo de uso direto; não confere endpoint a endpoint). | Endpoint novo com entidade no `@RequestBody` ou no `ResponseEntity<...>`. Sugerir DTO, principalmente se a entidade tem relacionamento lazy ou dado sensível. |

## Frontend (`petr-op-crono-frontend`)

Camadas: componente (apresentação e estado de tela) → service (HTTP e estado compartilhado) → utilitários de lógica pura.

### Violações

| Regra | Medição | Sinal no diff |
|---|---|---|
| **ARQ-F1** Componente não faz HTTP: chama um service. | 127 de 130 componentes seguem (exceções: `home`, `intervention` e `old-arara-chart`). | `HttpClient` injetado ou usado num `*.component.ts`. |
| **ARQ-F2** Service não depende de componente, exceto o componente de um modal/popover que o service abre. | 54 de 57 services não importam componente; os 3 que importam abrem modal ou popover (`bop-popover`, `anomaly-modal`, `deviations-justifications-modal`). | `import` de `*.component` num `*.service.ts` que não seja de modal/popover. |

## Como reportar

- **Violação:** citar este arquivo e o código da regra (ex.: `arquitetura.md — ARQ-B8`) mais o hunk literal. No `/crono-review:reviewer` entra como **Importante**; na `standards-spec-review`, em **Violações**.
- **Heurística:** citar o código da regra mais o hunk. Entra como **Sugestão** / **Heurística**, nunca bloqueia o merge.
