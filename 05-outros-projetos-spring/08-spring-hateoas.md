# Spring HATEOAS

> **Resumo:** modelos e construtores de links para APIs REST hipermídia (HATEOAS): a resposta traz as ações possíveis como links (formato HAL por padrão). O cliente navega seguindo links, sem montar URLs.
> **Pré-requisitos:** [04-web-mvc](../03-spring-boot/04-web-mvc.md).
> **Versões:** Spring HATEOAS **3.1.3** (BOM do Boot 4.1.2). Executado em `/tmp/validacao/s12/b`.

## 1. Definição

HATEOAS (*Hypermedia As The Engine Of Application State*) é o nível 3 do modelo de maturidade de Richardson. Uma resposta de pedido pago pode oferecer o link `cancel`; um pedido enviado não. O servidor controla o fluxo, o cliente apenas segue links.

## 2. Conceitos fundamentais

- `EntityModel<T>`: recurso + links. `CollectionModel<T>`: coleção. `PagedModel<T>`: paginação.
- `RepresentationModel`: base para seus próprios modelos.
- `Link` e *link relations* (`self`, `next`, ou relações próprias como `pay`).
- `WebMvcLinkBuilder.linkTo(methodOn(Controller.class).metodo(...))`: gera a URL **a partir do método do controller**, não de strings.
- Formato **HAL** (`application/hal+json`) por padrão; outros: HAL-FORMS, Collection+JSON, UBER.
- `RepresentationModelAssembler`: centraliza a montagem do modelo.

## 3. Como funciona por dentro

`methodOn(...)` cria um **proxy** (CGLIB/ByteBuddy, veja [07-aop-e-proxies](../02-spring-core/07-aop-e-proxies.md)) do controller que registra qual método foi chamado e com quais argumentos; `linkTo` lê a URL das anotações de mapeamento (`@GetMapping`, `@PathVariable`) e a resolve contra a requisição atual. Por ser proxy, o controller não pode ser `final` e os métodos referenciados precisam ser acessíveis e ter retorno proxiável (um método `void` não serve em `methodOn(...)`: erro "void type not allowed here" na validação; foi trocado por `ResponseEntity<Void>`).

## 4. Evolução histórica

O conceito é da tese de Roy Fielding (2000). O projeto nasceu em 2012 (Spring HATEOAS 0.x), com API reescrita na 1.0 (2019, `RepresentationModel`, `EntityModel`) e suporte reativo (`WebFluxLinkBuilder`). ⚠️ VERIFICAR datas (de memória).

## 5. Exemplos práticos

Dependência: `spring-boot-starter-hateoas`. Controller (executado):

```java
@RestController
public class OrderApi {
  public record Order(String id, String status) {}

  @GetMapping("/orders/{id}")
  EntityModel<Order> get(@PathVariable String id) {
    return EntityModel.of(new Order(id, "PAID"),
        linkTo(methodOn(OrderApi.class).get(id)).withSelfRel(),
        linkTo(methodOn(OrderApi.class).pay(id)).withRel("pay"));
  }

  @PostMapping("/orders/{id}/pay")
  ResponseEntity<Void> pay(@PathVariable String id) { return null; }   // exemplo mínimo
}
```

Resposta observada com `Accept: application/hal+json` (MockMvc):

```json
{"_links":{"self":{"href":"http://localhost/orders/9"},"pay":{"href":"http://localhost/orders/9/pay"}},"id":"9","status":"PAID"}
```

(`linkTo`/`methodOn` vêm de `org.springframework.hateoas.server.mvc.WebMvcLinkBuilder`.) Teste: starter `spring-boot-starter-hateoas-test`.

## 6. Configurações e anotações relevantes

| Item | Para que serve | Exemplo |
|---|---|---|
| `EntityModel.of(obj, links...)` | recurso com links | acima |
| `WebMvcLinkBuilder.linkTo/methodOn` | links por método | acima |
| `RepresentationModelAssembler` | monta modelos | para listas |
| `@EnableHypermediaSupport` | escolhe tipos de hipermídia | `type = HAL` ⚠️ VERIFICAR (Boot configura) |
| `spring.hateoas.*` | opções do Boot | ⚠️ VERIFICAR |

## 7. Quando usar e quando NÃO usar

- Use: APIs públicas de longa vida, fluxos com estados (links aparecem conforme o estado), paginação padronizada.
- Não use: APIs internas e simples; clientes que já "conhecem" as URLs (mobile fortemente acoplado). Na prática a maior parte dos clientes não navega por links, então o custo costuma não compensar. Considere [GraphQL](05-spring-graphql.md) se o problema for flexibilidade.

## 8. Erros comuns e boas práticas

- Montar URLs por string: use `linkTo(methodOn(...))`.
- Links que ignoram permissão/estado: só inclua ações válidas.
- Não versionar os tipos de mídia e relações: o contrato real são as *rels*.
- Retornar entidades JPA dentro de `EntityModel`: use DTOs ([padrões enterprise](../06-padroes-de-projeto/05-padroes-enterprise.md)).

## 9. Perguntas de revisão

1. O que o nível 3 de Richardson acrescenta?
<details><summary>Resposta</summary>Hipermídia: a resposta indica as próximas ações via links.</details>

2. Por que `methodOn` usa proxy?
<details><summary>Resposta</summary>Para registrar qual método/argumentos foram "chamados" e ler suas anotações de mapeamento, gerando a URL sem string.</details>

3. O que é HAL?
<details><summary>Resposta</summary>Formato JSON com `_links` e `_embedded`, `application/hal+json`.</details>

4. Qual a vantagem de links por estado?
<details><summary>Resposta</summary>O servidor controla quais ações estão disponíveis; o cliente não replica regras.</details>

5. Quando é exagero?
<details><summary>Resposta</summary>Em APIs internas pequenas cujo cliente já conhece as URLs.</details>

## 10. Referências oficiais

- Documentação: <https://docs.spring.io/spring-hateoas/docs/current/reference/html/>
- Projeto: <https://spring.io/projects/spring-hateoas>
- Richardson Maturity Model: <https://martinfowler.com/articles/richardsonMaturityModel.html>

Anterior: [07-spring-session](07-spring-session.md) · Próximo: [09-spring-rest-docs](09-spring-rest-docs.md)
