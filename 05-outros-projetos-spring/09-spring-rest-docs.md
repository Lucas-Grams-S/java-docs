# Spring REST Docs

> **Resumo:** gera trechos (*snippets*) de documentação de API a partir dos **testes**: exemplos de requisição/resposta (curl, HTTP, corpos) que são montados em um documento Asciidoctor. Se o teste falha, a documentação não é gerada, então ela não envelhece em silêncio.
> **Pré-requisitos:** [10-testes-no-spring-boot](../03-spring-boot/10-testes-no-spring-boot.md), [04-web-mvc](../03-spring-boot/04-web-mvc.md).
> **Versões:** Spring REST Docs **4.0.2** (BOM do Boot 4.1.2). Executado em `/tmp/validacao/s12/c`.

## 1. Definição

Alternativa à documentação gerada por anotações (Swagger/OpenAPI via springdoc). Aqui o **teste** é a fonte: ele chama a API (MockMvc, WebTestClient ou REST Assured) e registra o que viu. Campos documentados que não existem na resposta **quebram o teste**.

## 2. Conceitos fundamentais

- `document("identificador", ...)` após cada chamada gera os snippets em `target/snippets/<id>/`.
- Snippets padrão (observados): `curl-request.adoc`, `http-request.adoc`, `http-response.adoc`, `httpie-request.adoc`, `request-body.adoc`, `response-body.adoc`.
- Snippets sob demanda: `requestFields`, `responseFields`, `pathParameters`, `queryParameters`, `requestHeaders`, `links`.
- Os `.adoc` são incluídos em um documento Asciidoctor (`include::{snippets}/...[]`) e viram HTML/PDF.

## 3. Como funciona por dentro

Um `RestDocumentationContextProvider` (JUnit 5 extension) controla o diretório de saída. O `MockMvcRestDocumentationConfigurer` instala um interceptor na requisição de teste; ao executar `document(...)`, um `RestDocumentationResultHandler` captura a requisição e a resposta reais e aplica os snippets (templates Mustache) para gerar os arquivos. Documentar campos usa um *payload* parseado e compara com a resposta: campo não documentado ou ausente falha por padrão.

## 4. Evolução histórica

Criado em 2014/2015 como resposta ao viés "anotações poluem o código" do Swagger; ganhou suporte a WebTestClient, REST Assured e OpenAPI/Postman via extensões da comunidade. A 4.x acompanha Framework 7. ⚠️ VERIFICAR datas (de memória).

## 5. Exemplos práticos

Dependências de teste no Boot 4.1.2: `spring-boot-starter-restdocs` **mais** `spring-restdocs-mockmvc` (o starter sozinho não trazia as classes de MockMvc; foi preciso adicioná-las). Teste executado:

```java
@SpringBootTest @AutoConfigureMockMvc
@AutoConfigureRestDocs(outputDir = "target/snippets")      // org.springframework.boot.restdocs.test.autoconfigure
class OrderDocsTest {
  @Autowired MockMvc mvc;

  @Test void getOrder() throws Exception {
    mvc.perform(get("/orders/9"))
       .andExpect(status().isOk())
       .andDo(document("get-order"));                       // MockMvcRestDocumentation.document
  }
}
```

Resultado: criada a pasta `target/snippets/<id>/` com os seis arquivos listados acima. Documentação de campos (**não executado**):

<!-- não compilado -->
```java
.andDo(document("get-order",
    responseFields(
        fieldWithPath("id").description("Identificador do pedido"),
        fieldWithPath("status").description("PAID, SHIPPED..."))));
```

Montagem do HTML com `asciidoctor-maven-plugin` NÃO foi executada.

## 6. Configurações e anotações relevantes

| Item | Para que serve | Exemplo |
|---|---|---|
| `@AutoConfigureRestDocs` | configura REST Docs no teste (Boot) | `outputDir`, `uriHost` |
| `document(id, snippets...)` | gera snippets | acima |
| `responseFields`, `requestFields` | contrato de corpo | `fieldWithPath("id")` |
| `preprocessRequest/Response` | `prettyPrint`, mascarar valores | `modifyHeaders` |
| `asciidoctor-maven-plugin` | HTML final | `include::{snippets}/...` |

## 7. Quando usar e quando NÃO usar

- Use: documentação **precisa**, vinculada aos testes, com exemplos reais; equipe que valoriza código de produção sem anotações de doc.
- Não use: quando quer UI interativa para experimentar a API ("Try it out") ou contrato OpenAPI gerado automaticamente: springdoc/OpenAPI atendem melhor (REST Docs gera trechos, não um OpenAPI nativo).

## 8. Erros comuns e boas práticas

- Esquecer um campo → teste falha (é o objetivo); use `relaxedResponseFields` só quando justificado.
- Dados sensíveis nos snippets: aplique pré-processadores.
- Identificadores duplicados em `document(...)` sobrescrevem snippets.
- `/target` limpo antes do build de documentação para evitar lixo.

## 9. Perguntas de revisão

1. Qual a diferença entre REST Docs e springdoc?
<details><summary>Resposta</summary>REST Docs documenta a partir de testes (exemplos reais, verificados); springdoc gera OpenAPI a partir de anotações e do código.</details>

2. O que acontece se um campo da resposta não for documentado?
<details><summary>Resposta</summary>O teste falha, a menos que se use a variante *relaxed*.</details>

3. Onde ficam os snippets?
<details><summary>Resposta</summary>Em `target/snippets/<id>/` (configurável) e são incluídos em arquivos Asciidoctor.</details>

4. Por que a documentação é "sempre atual"?
<details><summary>Resposta</summary>É gerada pela execução dos testes; se a API divergir, o teste falha.</details>

5. Quando preferir OpenAPI?
<details><summary>Resposta</summary>Para UI interativa e geração de clientes/SDKs a partir de contrato.</details>

## 10. Referências oficiais

- Documentação: <https://docs.spring.io/spring-restdocs/docs/current/reference/htmlsingle/>
- Projeto: <https://spring.io/projects/spring-restdocs>
- Asciidoctor: <https://asciidoctor.org/>

Anterior: [08-spring-hateoas](08-spring-hateoas.md) · Voltar: [README](README.md) · Próximo da base: [../06-padroes-de-projeto/](../06-padroes-de-projeto/)
