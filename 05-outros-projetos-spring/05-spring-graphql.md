# Spring for GraphQL

> **Resumo:** suporte do Spring a APIs GraphQL sobre o GraphQL Java. Em vez de vários endpoints REST, um único endpoint em que o cliente escolhe os campos. Controllers anotados fazem a ponte entre o esquema e o código Java.
> **Pré-requisitos:** [04-web-mvc](../03-spring-boot/04-web-mvc.md), [06-spring-data-jpa](../03-spring-boot/06-spring-data-jpa.md) (N+1).
> **Versões:** Spring for GraphQL **2.0.6** (BOM do Boot 4.1.2), Java 25. Executado em `/tmp/validacao/s12/b` (MVC + `HttpGraphQlTester`).

## 1. Definição

GraphQL tem **esquema tipado** (SDL), operações `Query`, `Mutation` e `Subscription`, e o cliente descreve o formato da resposta. O Spring carrega arquivos `*.graphqls` e liga cada campo a um *data fetcher* escrito como método de `@Controller`.

## 2. Conceitos fundamentais

- **Esquema** em `src/main/resources/graphql/*.graphqls`.
- `@QueryMapping`, `@MutationMapping`, `@SubscriptionMapping`: campos raiz. `@Argument` liga argumentos.
- `@SchemaMapping`: resolve um campo de um tipo (ex.: `Order.customer`).
- `@BatchMapping`: resolve um campo para vários pais de uma vez (**resolve o N+1**, via DataLoader).
- Transportes: HTTP (`/graphql`), WebSocket, RSocket; MVC ou WebFlux.
- GraphiQL (IDE no navegador) opcional por propriedade.

## 3. Como funciona por dentro

Uma requisição chega ao handler HTTP, vira `ExecutionInput` e o `GraphQlService` executa o GraphQL Java. Para cada campo selecionado ele chama o *DataFetcher*. Como cada pedido pode ter `customer`, uma consulta de lista sem cuidado faz 1 + N chamadas: o `@BatchMapping` registra um `DataLoader` que agrupa as chaves e chama o método uma vez com a lista de pais. É o mesmo problema N+1 de [06-spring-data-jpa](../03-spring-boot/06-spring-data-jpa.md), no nível da API.

```mermaid
flowchart LR
  Q[POST /graphql] --> E[GraphQlService]
  E --> F1[@QueryMapping orders]
  E --> F2[@BatchMapping customer]
  F2 --> DL[DataLoader agrupa N chaves]
```

## 4. Evolução histórica

GraphQL foi aberto pelo Facebook em 2015. O Spring for GraphQL surgiu em 2021 (1.0 em maio de 2022, acompanhando o Boot 2.7/3.0) e foi absorvendo recursos do GraphQL Java. A série 2.x acompanha Framework 7 / Boot 4. ⚠️ VERIFICAR datas (de memória).

## 5. Exemplos práticos

Dependência: `spring-boot-starter-graphql` (+ `spring-boot-starter-webmvc`). Esquema:

```graphql
type Query { orderById(id: ID!): Order }
type Order { id: ID! status: String! }
```

Controller (executado; consulta `{ orderById(id: "7") { id status } }` devolveu `PAID`):

```java
@Controller
public class OrderGraphQl {
  public record Order(String id, String status) {}

  @QueryMapping
  Order orderById(@Argument String id) { return new Order(id, "PAID"); }
}
```

Teste (Boot 4: starter `spring-boot-starter-graphql-test`):

```java
@SpringBootTest @AutoConfigureHttpGraphQlTester   // org.springframework.boot.graphql.test.autoconfigure.tester
class OrderGraphQlTest {
  @Autowired HttpGraphQlTester gql;
  @Test void byId() {
    gql.document("{ orderById(id: \"7\") { id status } }").execute()
       .path("orderById.status").entity(String.class).isEqualTo("PAID");
  }
}
```

Evitando N+1 (**não compilado**):

<!-- não compilado -->
```java
@BatchMapping
Map<Order, Customer> customer(List<Order> orders) { /* uma consulta para todos */ }
```

## 6. Configurações e anotações relevantes

| Item | Para que serve | Exemplo |
|---|---|---|
| `spring.graphql.path` | endpoint HTTP | `/graphql` (padrão) |
| `spring.graphql.graphiql.enabled` | IDE de testes | `true` ⚠️ VERIFICAR |
| `spring.graphql.schema.locations` | onde ficam os esquemas | `classpath:graphql/**` |
| `@SchemaMapping` | campo de um tipo | `@SchemaMapping(typeName="Order")` |
| `@BatchMapping` | N+1 | acima |

## 7. Quando usar e quando NÃO usar

- Use: clientes variados (web, mobile) com necessidades de dados diferentes; BFF; grafos de entidades.
- Não use: API simples e estável (REST basta, com cache HTTP trivial); upload/streaming de arquivos; quando a equipe não pode cuidar de custo de consultas.

## 8. Erros comuns e boas práticas

- Consultas profundas/caras: configure limites de profundidade/complexidade.
- Esquecer o N+1 em campos aninhados (use `@BatchMapping`).
- Expor o esquema inteiro sem autorização por campo; combine com [Security](../03-spring-boot/08-spring-security.md).
- Erros retornam HTTP 200 com `errors` no corpo; monitore de acordo.

## 9. Perguntas de revisão

1. Onde ficam os esquemas por padrão?
<details><summary>Resposta</summary>Em `classpath:graphql/`, arquivos `*.graphqls`.</details>

2. Diferença entre `@QueryMapping` e `@SchemaMapping`?
<details><summary>Resposta</summary>`@QueryMapping` mapeia campos da raiz `Query`; `@SchemaMapping` mapeia um campo de qualquer tipo.</details>

3. Como o `@BatchMapping` evita N+1?
<details><summary>Resposta</summary>Usa um DataLoader que agrupa chaves e chama o método uma vez com a lista.</details>

4. Por que GraphQL costuma retornar 200 em erros?
<details><summary>Resposta</summary>O protocolo coloca erros no campo `errors`; a requisição HTTP em si foi processada.</details>

5. Quando REST é melhor?
<details><summary>Resposta</summary>Recursos simples, cache HTTP, arquivos, equipes sem necessidade de consultas flexíveis.</details>

## 10. Referências oficiais

- Documentação: <https://docs.spring.io/spring-graphql/reference/>
- Projeto: <https://spring.io/projects/spring-graphql>
- GraphQL: <https://graphql.org/learn/>

Anterior: [04-spring-authorization-server](04-spring-authorization-server.md) · Próximo: [06-spring-ai](06-spring-ai.md)
