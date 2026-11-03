# Baseline de code smells e SOLID

Doze smells do capítulo 3 de *Refactoring* (Martin Fowler, 2ª ed.) e os cinco princípios SOLID (seção no fim). Valem **só onde o repositório não documenta regra própria**; a regra do repo sempre vence.

Cada smell é uma **heurística**, nunca uma violação: reportar com o nome do smell, o hunk literal e uma sugestão de correção. Na dúvida, não reportar.

| Smell | Sinal no diff | Correção típica |
|---|---|---|
| **Mysterious Name** | Nome que não diz o que a coisa é ou faz: abreviação obscura, metáfora (`haystack`), genérico (`data`, `aux`, `handle2`). | Renomear para o conceito do domínio. |
| **Duplicated Code** | O mesmo trecho (ou quase) aparece duas vezes no diff, ou repete código que já existe no repo. | Extrair função/helper e reutilizar. |
| **Feature Envy** | Método que mexe mais nos dados de outro objeto do que nos seus próprios (vários `outro.getX()`). | Mover o método (ou parte dele) para a classe dona dos dados. |
| **Data Clumps** | O mesmo grupo de parâmetros/campos viaja junto (ex.: `sequenceId, revision, contextId` em várias assinaturas). | Agrupar num tipo/DTO/record. |
| **Primitive Obsession** | Conceito do domínio representado por `string`/`number`/`Long` crus (status como string, ID composto em string). | Introduzir enum, tipo ou value object. |
| **Repeated Switches** | O mesmo `switch`/`if` encadeado sobre o mesmo tipo/enum em vários lugares. | Polimorfismo, mapa de estratégias ou um único ponto de decisão. |
| **Shotgun Surgery** | Uma mudança conceitual pequena obrigou edições espalhadas em muitos arquivos. | Concentrar a responsabilidade num lugar só. |
| **Divergent Change** | Uma mesma classe recebe mudanças por motivos não relacionados (ex.: componente que já faz PDF, formatação e layout ganha mais um papel). | Separar por responsabilidade (componente próprio, service, helper). |
| **Speculative Generality** | Parâmetros, abstrações, flags ou hooks "para o futuro" que nenhum chamador usa. | Remover até existir um uso real. |
| **Message Chains** | Navegação longa `a.getB().getC().getD()` ou `x?.y?.z?.w` repetida. | Esconder a navegação atrás de um método/propriedade. |
| **Middle Man** | Classe/método que só repassa a chamada para outro, sem lógica própria. | Chamar o destino direto ou fundir as camadas. |
| **Refused Bequest** | Subclasse ou implementação que herda/implementa algo e ignora ou anula boa parte (métodos vazios, `throw new UnsupportedOperationException`). | Trocar herança por composição ou ajustar a hierarquia. |

## SOLID

Mesmas regras dos smells: **heurística**, nunca violação, sempre com hunk literal e correção, e só onde o repositório não documenta regra própria. Reportar com o nome do princípio (ex.: "SOLID-D"). Quando o caso já é um smell da tabela acima, reportar uma vez só, com o nome do smell.

| Princípio | Sinal no diff | Correção típica |
|---|---|---|
| **SOLID-S** (Responsabilidade única) | Service ou componente ganha um método de outro assunto (ex.: service de sequência passa a montar e-mail e gerar PDF; componente passa a formatar datas e chamar API). Equivale a *Divergent Change* quando a classe já é grande. | Extrair para um service/helper/componente do assunto certo. |
| **SOLID-O** (Aberto/fechado) | Adicionar um tipo exige mais um `case`/`if (tipo == X)` num `switch` que já existe e que se repete em outros lugares. | Mapa de estratégias, polimorfismo ou um único ponto de decisão por tipo. |
| **SOLID-L** (Substituição de Liskov) | Implementação/subclasse que lança `UnsupportedOperationException`, retorna vazio ou ignora parâmetros de um método herdado; ou exige pré-condição mais forte que a do contrato. | Ajustar a hierarquia ou trocar herança por composição. |
| **SOLID-I** (Segregação de interfaces) | Interface nova (ou método novo em interface existente) que algumas implementações deixam vazio ou não usam; `@Input` que recebe um objeto inteiro quando o componente usa um campo. | Dividir a interface por papel; passar só o dado necessário. |
| **SOLID-D** (Inversão de dependência) | Classe de negócio instancia a própria dependência de infraestrutura (`new RestTemplate()`, `new ObjectMapper()` em método, `new XxxService()`); chamada estática a infraestrutura dentro de regra de negócio; `@Autowired` em campo em classe nova (o padrão novo é construtor via `@RequiredArgsConstructor`). | Receber a dependência por injeção (construtor/`inject()`), declarando bean quando preciso. |
