# Tarefa: gerar uma base de estudo em Markdown sobre Java, Spring e o ecossistema Spring

## Seu papel
Aja como engenheiro Java sênior e instrutor técnico. Escreva uma base de conhecimento em arquivos `.md`, em **português do Brasil**. Ela começa nos conceitos iniciais de Java (sem lógica de programação básica) e vai até os padrões de projeto. Também cobre o Spring Framework e as variações do Spring mais usadas no mercado.

## Público
Pessoa que já sabe lógica de programação (variáveis, condicionais, laços, funções) e quer entender **como** e **por que** Java e Spring funcionam, além de saber usá-los.

---

## ⚠️ REGRA PRINCIPAL: trabalho dividido em sessões
O conteúdo é grande demais para uma sessão só. Fazer tudo de uma vez deixa os textos rasos e com erros. Por isso:

1. **Cada sessão executa APENAS UMA etapa** da tabela "Plano de sessões" abaixo. Nunca adiante a etapa seguinte, mesmo que ainda haja espaço.
2. **O estado fica em `PROGRESSO.md`**, na raiz. Ele contém:
   - as versões de referência confirmadas (Java, Spring Framework, Spring Boot, Spring Cloud);
   - a tabela de sessões com status (`pendente` / `em andamento` / `concluída`) e a data;
   - os termos já definidos em cada arquivo (para não repetir conteúdo e para criar links);
   - as pendências e pontos marcados com `⚠️ VERIFICAR`;
   - decisões de estilo tomadas (ex.: projeto-exemplo usado nos códigos, convenções de nome).
3. **No início de cada sessão:** leia `PROGRESSO.md` e o `00-README.md`, identifique a próxima etapa pendente, diga em uma linha o que vai fazer e comece.
4. **No fim de cada sessão:** atualize `PROGRESSO.md`, mostre um resumo curto (arquivos criados, pendências) e **pare**. Termine com a frase: *"Para continuar, abra uma nova sessão e diga: `continue a partir do PROGRESSO.md`."*
5. Se a etapa ficar grande demais e o contexto começar a pesar, salve o que já estiver pronto, registre em `PROGRESSO.md` onde parou e encerre a sessão. **Qualidade vem antes de cumprir a etapa inteira.**

## Plano de sessões
| Sessão | Etapa |
|---|---|
| 0 | Planejamento: estrutura de pastas, `00-README.md`, `PROGRESSO.md`, confirmação das versões. **Mostre o índice e espere aprovação.** |
| 1 | `01-java/00-evolucao-do-java.md` (página histórica, ver especificação abaixo) |
| 2 | `01-java/` arquivos 01 a 05 |
| 3 | `01-java/` arquivos 06 a 10 |
| 4 | `01-java/` arquivos 11 a 15 |
| 5 | `02-spring-core/` arquivos 01 a 05 |
| 6 | `02-spring-core/` arquivos 06 a 10 |
| 7 | `03-spring-boot/` arquivos 01 a 04 |
| 8 | `03-spring-boot/` arquivos 05 a 08 |
| 9 | `03-spring-boot/` arquivos 09 a 13 |
| 10 | `04-spring-cloud/` arquivos 01 a 05 |
| 11 | `04-spring-cloud/` arquivos 06 a 09 |
| 12 | `05-outros-projetos-spring/` |
| 13 | `06-padroes-de-projeto/` arquivos 01 a 03 |
| 14 | `06-padroes-de-projeto/` arquivo 04 |
| 15 | `06-padroes-de-projeto/` arquivos 05 a 08 |
| 16 | `07-projeto-pratico/`, `GLOSSARIO.md` e revisão final (links relativos, consistência, pendências `⚠️ VERIFICAR`) |

---

## Processo obrigatório para CADA arquivo (profundidade e precisão)
Siga estes passos na ordem, para todos os arquivos:

1. **Pesquisar antes de escrever:** se WebFetch/WebSearch estiverem disponíveis, leia a seção correspondente da documentação oficial (reference docs do Spring, JEPs, tutoriais do dev.java, JLS/JVMS) antes de escrever. Baseie-se no que leu, não só na memória.
2. **Esboçar:** monte o esqueleto do arquivo com os subtópicos e a ideia central de cada um. Confira se o tópico está coberto em profundidade, e não só listado.
3. **Escrever com profundidade:** para cada conceito, responda **o que é, por que existe (que problema resolve), como funciona por dentro e quando NÃO usar**. Prefira explicar o mecanismo a listar recursos.
4. **Validar o código:** se houver JDK/Maven/Gradle no ambiente, compile e, quando fizer sentido, execute os exemplos numa pasta temporária fora da base (ex.: `/tmp/validacao`). Corrija o que não compilar. Se não for possível validar, escreva acima do exemplo: `<!-- não compilado -->`.
5. **Revisar com o checklist abaixo** antes de considerar o arquivo pronto.

### Checklist de revisão por arquivo
- [ ] Segue o modelo obrigatório (todas as seções presentes)
- [ ] Nenhuma API, anotação, propriedade ou flag inventada. Na dúvida, verifique ou marque `⚠️ VERIFICAR`
- [ ] Recursos dependentes de versão indicam "a partir do Java X" / "Spring Boot X"
- [ ] Recursos em preview/incubator estão identificados como tal
- [ ] Exemplos coerentes entre si e com o restante da base
- [ ] Links relativos apontam para arquivos existentes ou planejados no índice
- [ ] Links externos verificados, ou trocados pela página raiz da doc oficial
- [ ] Diagramas mermaid com sintaxe válida
- [ ] Sem repetir explicações já dadas em outro arquivo (use link)

---

## Regras gerais de conteúdo
1. **Versões atuais:** use as versões confirmadas na Sessão 0 (LTS mais recente do Java, versões atuais do Spring Framework, Spring Boot e do release train do Spring Cloud).
2. **Links oficiais:** cada arquivo termina com "Referências oficiais", com links para:
   - Java: `docs.oracle.com/en/java`, `dev.java`, `openjdk.org/jeps`, `docs.oracle.com/javase/specs`
   - Spring: `docs.spring.io` (reference e Javadoc), `spring.io/projects`
   - Padrões: GoF, `martinfowler.com`, `microservices.io`
   Nunca invente uma URL. Se não conseguir verificar o link exato, use a página raiz da documentação oficial.
3. **Código:** Java moderno, completo o bastante para compilar (imports quando não forem óbvios), identificadores em inglês, comentários em português. Mostre o jeito "antigo" e o "moderno" quando isso ajudar a entender a evolução.
4. **Diagramas:** blocos ```mermaid para fluxos, ciclos de vida, arquiteturas e diagramas de classe.
5. **Projeto-exemplo comum:** sempre que possível, use o mesmo domínio nos exemplos (ex.: sistema de pedidos com `Customer`, `Order`, `Product`). Registre isso em `PROGRESSO.md`.
6. **Tamanho:** de 250 a 700 linhas por arquivo. Se passar disso, divida (`-parte-1`, `-parte-2`) e atualize o índice.

## Modelo obrigatório de cada arquivo
```
# <Título>

> **Resumo:** 2–3 linhas sobre o que é e por que importa.
> **Pré-requisitos:** links para os arquivos necessários.
> **Versões:** a partir de qual versão existe / versão usada nos exemplos.

## 1. Definição
## 2. Conceitos fundamentais
## 3. Como funciona por dentro          (mecanismo interno: JVM, reflection, proxies, etc.)
## 4. Evolução histórica                 (como o recurso surgiu e mudou entre versões; link para 01-java/00-evolucao-do-java.md ou para a evolução do Spring)
## 5. Exemplos práticos                  (do simples ao realista)
## 6. Configurações e anotações relevantes  (tabela: anotação/propriedade | para que serve | exemplo)
## 7. Quando usar e quando NÃO usar
## 8. Erros comuns e boas práticas
## 9. Perguntas de revisão               (5 perguntas, respostas dentro de <details>)
## 10. Referências oficiais
```
A seção 4 pode ser curta (2 a 3 linhas) quando o tópico não tiver história relevante, mas não pode faltar.

---

## Estrutura de pastas e conteúdo

### Raiz
- `00-README.md`: índice geral, ordem de estudo, versões de referência e mapa mental (mermaid) do ecossistema.
- `PROGRESSO.md`: controle das sessões (ver regra principal).
- `GLOSSARIO.md`: gerado na última sessão.

### `01-java/` — Como o Java funciona
- `00-evolucao-do-java.md`: **página histórica (especificação abaixo)**
- `01-plataforma-java.md`: JDK vs JRE vs JVM, compilação para bytecode, JIT, HotSpot, GraalVM, distribuições (Oracle JDK, OpenJDK, Temurin, Corretto etc.)
- `02-jvm-por-dentro.md`: ClassLoaders e delegação, áreas de memória (heap, stack, metaspace), Garbage Collectors (G1, ZGC, Shenandoah, Parallel, Serial), noções de tuning e ferramentas (JFR, jcmd, VisualVM)
- `03-tipos-e-memoria.md`: primitivos vs referência, wrappers, autoboxing, pool de String, imutabilidade, `equals`/`hashCode`, passagem por valor
- `04-orientacao-a-objetos.md`: encapsulamento, herança, polimorfismo, abstração, interfaces (default/static/private), classes abstratas, composição vs herança, classes internas e anônimas
- `05-recursos-modernos-da-linguagem.md`: `var`, records, sealed classes, pattern matching (`instanceof`, switch, record patterns), text blocks, switch expressions
- `06-generics.md`: wildcards, PECS, type erasure e suas consequências
- `07-colecoes.md`: hierarquia, implementações e complexidade, coleções imutáveis, Sequenced Collections
- `08-excecoes.md`: checked vs unchecked, try-with-resources, hierarquia Throwable, boas práticas
- `09-programacao-funcional.md`: lambdas, interfaces funcionais, method references, Streams API (incluindo coletores e streams paralelos), Optional
- `10-anotacoes-e-reflection.md`: anotações, `@Retention`, `@Target`, anotações customizadas, Reflection API, MethodHandles e **por que frameworks dependem disso**
- `11-proxies-dinamicos-e-bytecode.md`: `java.lang.reflect.Proxy`, geração de bytecode (CGLIB/ByteBuddy, visão geral), Class-File API (se já estiver disponível na versão de referência)
- `12-concorrencia.md`: threads, `ExecutorService`, `CompletableFuture`, `synchronized`, locks, `volatile`, Java Memory Model, coleções concorrentes, virtual threads, structured concurrency e scoped values (com status de cada um)
- `13-io-nio-e-http.md`: I/O clássico, NIO.2, HttpClient
- `14-modulos-e-build.md`: JPMS, classpath vs module path, Maven e Gradle (ciclo de vida, dependências, escopos, BOM)
- `15-testes-em-java.md`: JUnit 5, AssertJ, Mockito

#### Especificação de `01-java/00-evolucao-do-java.md`
Esta página tem estrutura própria, diferente do modelo padrão:
1. **Linha do tempo (mermaid `timeline`)** da primeira versão até a mais recente.
2. **Eras do Java:** era Sun (1.0 a 6), aquisição pela Oracle e Java 7, a virada do Java 8, a modularização no Java 9, o modelo de releases a cada 6 meses (desde o Java 10) e a mudança da cadência de LTS (de 3 para 2 anos).
3. **Tabela por versão** (todas, de 1.0 até a atual): `Versão | Ano | LTS? | Principais novidades | JEPs relevantes | Impacto prático`. Dê mais destaque a 1.2, 1.4, 5, 7, 8, 9, 11, 17, 21 e à LTS mais recente.
4. **Seção detalhada por marco**, com exemplo de código "antes vs depois": generics/enums/annotations (5), try-with-resources/diamond (7), lambdas/streams/java.time (8), módulos/JShell (9), `var` (10), HttpClient (11), switch expressions/text blocks/records/sealed/pattern matching (14 a 21), virtual threads (21) e as novidades da LTS mais recente.
5. **Como funciona o processo de evolução:** JCP, JSR, JEP, OpenJDK, recursos em preview e incubator e como habilitá-los (`--enable-preview`).
6. **Mudanças de ecossistema que afetam o Spring:** licenciamento do Oracle JDK vs OpenJDK, migração `javax.*` → `jakarta.*` (Java EE → Jakarta EE) e versão mínima de Java exigida por cada geração do Spring Boot.
7. **Projetos em andamento no OpenJDK** (Valhalla, Panama, Loom, Leyden, Amber): o que cada um busca e o que já entregou.
8. **Referências oficiais:** página de JEPs por release em `openjdk.org`, release notes da Oracle e roadmap do Java SE.

### `02-spring-core/` — Como o Spring funciona
- `01-o-que-e-spring.md`: problema que resolve, módulos, Spring vs Boot vs Cloud, **evolução do Spring** (das versões 1.x/2.x baseadas em XML até a versão atual: anotações, Java Config, Boot, reativo, Jakarta EE, AOT) e o papel do Java mínimo em cada versão
- `02-ioc-e-injecao-de-dependencia.md`: IoC, DI por construtor/setter/campo, `@Autowired`, `@Qualifier`, `@Primary`
- `03-application-context-e-beans.md`: `BeanFactory` vs `ApplicationContext`, `BeanDefinition`, estereótipos, `@Bean`, `@Configuration` (modo full vs lite)
- `04-ciclo-de-vida-e-escopos.md`: ciclo de vida (mermaid), `BeanPostProcessor`, `BeanFactoryPostProcessor`, `@PostConstruct`/`@PreDestroy`, escopos, dependências circulares
- `05-configuracao.md`: XML → Java Config → anotações, scanning, `@Import`, `@Profile`, `Environment`, `@Value`, `@PropertySource`, SpEL
- `06-como-java-e-spring-conversam.md`: **arquivo central.** O que acontece passo a passo quando o contexto sobe: reflection, scanning, `BeanDefinition`s, instanciação, injeção, proxies (JDK vs CGLIB), eventos. Inclua um **mini-container de DI didático em Java puro** (~100 a 150 linhas), compilado e testado.
- `07-aop-e-proxies.md`: conceitos de AOP, Spring AOP vs AspectJ, como `@Transactional`/`@Async`/`@Cacheable` funcionam por proxy, self-invocation
- `08-eventos-validacao-e-conversao.md`: `ApplicationEvent`, `@EventListener`, Bean Validation, conversores e formatadores
- `09-transacoes.md`: `PlatformTransactionManager`, `@Transactional`, propagação, isolamento, rollback rules
- `10-aot-e-native.md`: AOT do Spring, GraalVM native image, limitações

### `03-spring-boot/`
- `01-o-que-e-spring-boot.md`: convenção sobre configuração, starters, servidor embutido, `@SpringBootApplication` decomposta, evolução das versões principais do Boot
- `02-auto-configuracao.md`: funcionamento interno (`AutoConfiguration.imports`, `@Conditional*`), depuração (condition evaluation report), criação de starter próprio
- `03-configuracao-externalizada.md`: properties/yml, ordem de precedência, profiles, `@ConfigurationProperties`
- `04-web-mvc.md`: DispatcherServlet (mermaid), `@RestController`, `ResponseEntity`, `@ControllerAdvice`, ProblemDetail, filtros vs interceptors, RestClient
- `05-webflux-e-reativo.md`: Reactor, quando usar, comparação com virtual threads
- `06-spring-data-jpa.md`: JPA/Hibernate, entidades, relacionamentos, repositories, query methods, `@Query`, paginação, N+1, projeções
- `07-spring-data-outros.md`: JdbcClient, MongoDB, Redis (visão geral)
- `08-spring-security.md`: SecurityFilterChain, autenticação vs autorização, `UserDetailsService`, password encoding, JWT, OAuth2 Resource Server/Client, segurança por método
- `09-actuator-e-observabilidade.md`: Actuator, Micrometer (métricas e tracing), health checks
- `10-testes-no-spring-boot.md`: `@SpringBootTest`, test slices, MockMvc, Testcontainers, `@MockitoBean`
- `11-cache-agendamento-e-async.md`: `@Cacheable`, `@Scheduled`, `@Async`
- `12-migracoes-de-banco.md`: Flyway e Liquibase
- `13-empacotamento-e-deploy.md`: fat jar, layered jars, Docker/Buildpacks, native image

### `04-spring-cloud/` — Microsserviços
- `01-visao-geral.md`: release train, compatibilidade com Boot, quando usar e quando não usar microsserviços
- `02-config-server.md`
- `03-service-discovery.md`: Eureka, Consul, discovery do Kubernetes
- `04-api-gateway.md`: Spring Cloud Gateway
- `05-comunicacao-entre-servicos.md`: OpenFeign, HTTP Interfaces (`@HttpExchange`), RestClient/WebClient, load balancing
- `06-resiliencia.md`: Circuit Breaker + Resilience4j
- `07-mensageria.md`: Spring Cloud Stream, Kafka, RabbitMQ
- `08-tracing-distribuido.md`: Micrometer Tracing (sucessor do Sleuth), OpenTelemetry, Zipkin
- `09-spring-cloud-e-kubernetes.md`

### `05-outros-projetos-spring/`
Uma página por projeto, mais curta (definição, quando usar, exemplo mínimo, links): Spring Batch, Spring Integration, Spring Modulith, Spring Authorization Server, Spring GraphQL, Spring AI, Spring Session, Spring HATEOAS, Spring REST Docs. Confirme em `spring.io/projects` se cada um continua ativo.

### `06-padroes-de-projeto/`
- `01-introducao.md`: o que são padrões, SOLID, DRY, KISS, YAGNI, acoplamento e coesão
- `02-criacionais.md`: Singleton, Factory Method, Abstract Factory, Builder, Prototype
- `03-estruturais.md`: Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy
- `04-comportamentais.md`: Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor
  → Para **cada** padrão GoF: problema, solução, `classDiagram` mermaid, implementação em Java puro, **onde o JDK ou o Spring o usam** (ex.: `JdbcTemplate` = Template Method, `ApplicationEvent` = Observer, `HandlerInterceptor` = Chain of Responsibility, `BeanFactory` = Factory) e a implementação "do jeito Spring" (ex.: Strategy com `Map<String, MinhaInterface>` injetado)
- `05-padroes-enterprise.md`: Repository, Service Layer, DTO, Mapper, Unit of Work, Specification
- `06-padroes-arquiteturais.md`: camadas, MVC, Hexagonal, Clean Architecture, DDD tático, CQRS, Event Sourcing, monólito modular, com exemplos de estrutura de pacotes em Spring Boot
- `07-padroes-de-microsservicos.md`: API Gateway, Service Discovery, Circuit Breaker, Saga, Outbox, Database per Service, Strangler Fig, BFF, ligando cada um ao projeto Spring correspondente
- `08-anti-padroes.md`: God Class, Anemic Domain Model (e o debate em torno dele), field injection, `@Transactional` mal usado, Open Session in View, entre outros

### `07-projeto-pratico/`
- `README.md`: roteiro de um projeto incremental (API de pedidos, usando o domínio comum dos exemplos), com checkpoints que apontam para os arquivos de teoria.

---

## Comece agora
Execute **somente a Sessão 0**: crie a estrutura, o `00-README.md` e o `PROGRESSO.md`, confirme as versões, mostre o índice e espere minha aprovação.
