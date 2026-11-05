# Glossário

> **Resumo:** termos usados na base, em ordem alfabética, com definição curta e link para onde são explicados em profundidade.
> **Versões:** Java 25, Spring Framework 7.0.x, Spring Boot 4.1.x, Spring Cloud 2025.1.x.

## A
- **AOP (Programação Orientada a Aspectos):** separa preocupações transversais (transação, segurança, log) da lógica de negócio. → [07-aop-e-proxies](02-spring-core/07-aop-e-proxies.md)
- **AOT (Ahead-of-Time):** processamento no build que gera código e *hints* no lugar de reflection em tempo de execução. → [10-aot-e-native](02-spring-core/10-aot-e-native.md)
- **Auto-configuração:** beans registrados pelo Boot conforme classpath, propriedades e beans existentes (`@Conditional*`). → [02-auto-configuracao](03-spring-boot/02-auto-configuracao.md)
- **Autoboxing:** conversão automática entre primitivo e wrapper (`int` ↔ `Integer`). → [03-tipos-e-memoria](01-java/03-tipos-e-memoria.md)

## B
- **Bean:** objeto instanciado, montado e gerenciado pelo contêiner Spring. → [03-application-context-e-beans](02-spring-core/03-application-context-e-beans.md)
- **BeanDefinition:** metadados de um bean (classe, escopo, dependências) antes da instanciação. → [03](02-spring-core/03-application-context-e-beans.md)
- **BeanFactoryPostProcessor:** altera definições de beans antes de qualquer instanciação. → [04-ciclo-de-vida-e-escopos](02-spring-core/04-ciclo-de-vida-e-escopos.md)
- **BeanPostProcessor:** intercepta cada bean criado, antes/depois da inicialização; base dos proxies. → [04](02-spring-core/04-ciclo-de-vida-e-escopos.md)
- **BOM (Bill of Materials):** POM que fixa versões compatíveis de dependências. → [14-modulos-e-build](01-java/14-modulos-e-build.md)
- **Bytecode:** instruções da JVM contidas nos `.class`. → [01-plataforma-java](01-java/01-plataforma-java.md)

## C
- **Circuit Breaker:** corta chamadas a um serviço falho para evitar cascata. → [06-resiliencia](04-spring-cloud/06-resiliencia.md)
- **ClassLoader:** carrega classes na JVM, com delegação ao pai. → [02-jvm-por-dentro](01-java/02-jvm-por-dentro.md)
- **CGLIB:** técnica de proxy por subclasse; não funciona com classes/métodos `final`. → [11-proxies](01-java/11-proxies-dinamicos-e-bytecode.md)
- **Config Server:** serviço que entrega configuração centralizada. → [02-config-server](04-spring-cloud/02-config-server.md)
- **Contêiner (IoC):** objeto que cria e liga os beans (`ApplicationContext`). → [02-ioc](02-spring-core/02-ioc-e-injecao-de-dependencia.md)
- **CRaC:** checkpoint/restore da JVM para partida rápida. → [00-evolucao](01-java/00-evolucao-do-java.md)

## D
- **DI (Injeção de Dependência):** receber colaboradores de fora em vez de criá-los; preferir construtor. → [02-ioc](02-spring-core/02-ioc-e-injecao-de-dependencia.md)
- **Discovery (descoberta de serviços):** registro e consulta dinâmica de instâncias. → [03-service-discovery](04-spring-cloud/03-service-discovery.md)
- **DTO:** objeto de transferência de dados entre camadas/APIs; em Java moderno, costuma ser um `record`. → [05-padroes-enterprise](06-padroes-de-projeto/05-padroes-enterprise.md)

## E
- **Eureka:** servidor de discovery do Netflix OSS usado no Spring Cloud. → [03](04-spring-cloud/03-service-discovery.md)
- **Entidade (JPA):** classe mapeada a uma tabela. → [06-spring-data-jpa](03-spring-boot/06-spring-data-jpa.md)
- **`equals`/`hashCode` (contrato):** objetos iguais devem ter o mesmo hash. → [03-tipos-e-memoria](01-java/03-tipos-e-memoria.md)
- **Escopo de bean:** ciclo de vida da instância (`singleton`, `prototype`, `request`...). → [04](02-spring-core/04-ciclo-de-vida-e-escopos.md)

## F
- **Flyway / Liquibase:** ferramentas de migração versionada de banco. → [12-migracoes](03-spring-boot/12-migracoes-de-banco.md)
- **Fail-fast (iterador):** falha com `ConcurrentModificationException` ao detectar modificação. → [07-colecoes](01-java/07-colecoes.md)

## G
- **Garbage Collector (GC):** libera memória de objetos inalcançáveis (G1, ZGC, Shenandoah...). → [02-jvm-por-dentro](01-java/02-jvm-por-dentro.md)
- **Gateway:** ponto único de entrada que roteia e aplica filtros. → [04-api-gateway](04-spring-cloud/04-api-gateway.md)
- **Generics:** parametrização de tipos; apagados em tempo de execução (*type erasure*). → [06-generics](01-java/06-generics.md)
- **GoF:** "Gang of Four", autores do catálogo de 23 padrões de 1994. → [01-introducao](06-padroes-de-projeto/01-introducao.md)

## H
- **Happens-before:** garantia de visibilidade/ordem do Java Memory Model. → [12-concorrencia](01-java/12-concorrencia.md)
- **Heap:** área de memória dos objetos. → [02-jvm-por-dentro](01-java/02-jvm-por-dentro.md)
- **HATEOAS:** hipermídia como mecanismo de estado da API REST. → [08-spring-hateoas](05-outros-projetos-spring/08-spring-hateoas.md)

## I
- **Idempotência:** repetir a operação produz o mesmo efeito; essencial em retry e mensageria. → [07-padroes-de-microsservicos](06-padroes-de-projeto/07-padroes-de-microsservicos.md)
- **IoC (Inversão de Controle):** o framework controla a criação e o fluxo, não o seu código. → [02-ioc](02-spring-core/02-ioc-e-injecao-de-dependencia.md)

## J
- **Jakarta EE:** sucessor do Java EE; pacotes `jakarta.*` no lugar de `javax.*`. → [00-evolucao](01-java/00-evolucao-do-java.md)
- **JDK / JRE / JVM:** kit de desenvolvimento / ambiente de execução / máquina virtual. → [01-plataforma-java](01-java/01-plataforma-java.md)
- **JEP / JSR / JCP:** proposta de melhoria do JDK / requisição de especificação / processo da comunidade. → [00-evolucao](01-java/00-evolucao-do-java.md)
- **JIT:** compilação em tempo de execução de bytecode para código nativo (C1/C2). → [01](01-java/01-plataforma-java.md)
- **JPMS:** sistema de módulos do Java 9 (`module-info.java`). → [14-modulos-e-build](01-java/14-modulos-e-build.md)
- **JWT:** token assinado com claims, usado em autenticação sem estado. → [08-spring-security](03-spring-boot/08-spring-security.md)

## L
- **Lambda:** função anônima; compilada via `invokedynamic`. → [09-programacao-funcional](01-java/09-programacao-funcional.md)
- **Lazy loading:** carrega associações JPA sob demanda; exige sessão aberta. → [06](03-spring-boot/06-spring-data-jpa.md)

## M
- **Metaspace:** área de metadados de classes (fora do heap). → [02-jvm-por-dentro](01-java/02-jvm-por-dentro.md)
- **MVC:** padrão Model-View-Controller; `Spring Web MVC` é a pilha servlet. → [04-web-mvc](03-spring-boot/04-web-mvc.md)
- **Micrometer:** fachada de métricas e tracing do Boot. → [09-actuator](03-spring-boot/09-actuator-e-observabilidade.md)
- **Modulith:** monólito com módulos e fronteiras verificadas. → [03-spring-modulith](05-outros-projetos-spring/03-spring-modulith.md)

## N
- **N+1:** uma consulta mais N consultas por associação carregada preguiçosamente. → [06](03-spring-boot/06-spring-data-jpa.md), [08-anti-padroes](06-padroes-de-projeto/08-anti-padroes.md)
- **Native image:** executável gerado pelo GraalVM, sem JVM. → [10-aot-e-native](02-spring-core/10-aot-e-native.md)

## O
- **OSIV (Open Session in View):** mantém a sessão JPA aberta durante a requisição. → [06](03-spring-boot/06-spring-data-jpa.md)
- **Outbox:** grava o evento na mesma transação do dado e o publica depois. → [07-padroes-de-microsservicos](06-padroes-de-projeto/07-padroes-de-microsservicos.md)

## P
- **Pattern matching:** teste de tipo/estrutura com extração (`instanceof`, `switch`, record patterns). → [05-recursos-modernos](01-java/05-recursos-modernos-da-linguagem.md)
- **POJO:** objeto Java simples, sem dependência de framework. → [01-o-que-e-spring](02-spring-core/01-o-que-e-spring.md)
- **Preview / incubator:** recurso ainda não final; exige `--enable-preview` ou módulo incubador. → [00-evolucao](01-java/00-evolucao-do-java.md)
- **Profile:** conjunto de beans/propriedades ativado por ambiente. → [05-configuracao](02-spring-core/05-configuracao.md)
- **Propagação (transacional):** como `@Transactional` se comporta ao encontrar transação existente. → [09-transacoes](02-spring-core/09-transacoes.md)
- **Proxy:** objeto substituto que intercepta chamadas (JDK ou CGLIB). → [11-proxies](01-java/11-proxies-dinamicos-e-bytecode.md), [07-aop](02-spring-core/07-aop-e-proxies.md)

## R
- **Reactive Streams / Reactor:** fluxo assíncrono com *backpressure* (`Mono`, `Flux`). → [05-webflux](03-spring-boot/05-webflux-e-reativo.md)
- **Record:** classe de dados imutável (Java 16). → [05-recursos-modernos](01-java/05-recursos-modernos-da-linguagem.md)
- **Reflection:** inspeção e invocação de classes em tempo de execução. → [10-anotacoes-e-reflection](01-java/10-anotacoes-e-reflection.md)
- **Repository:** abstração de coleção de agregados persistidos. → [05-padroes-enterprise](06-padroes-de-projeto/05-padroes-enterprise.md)

## S
- **Saga:** transação distribuída como sequência de transações locais com compensações. → [07-padroes-de-microsservicos](06-padroes-de-projeto/07-padroes-de-microsservicos.md)
- **Sealed:** hierarquia fechada de tipos permitidos. → [05-recursos-modernos](01-java/05-recursos-modernos-da-linguagem.md)
- **Self-invocation:** chamada interna que ignora o proxy, e portanto o aspecto. → [07-aop](02-spring-core/07-aop-e-proxies.md)
- **Slice (teste):** teste que carrega só parte do contexto (`@WebMvcTest`, `@DataJpaTest`). → [10-testes](03-spring-boot/10-testes-no-spring-boot.md)
- **SpEL:** Spring Expression Language. → [05-configuracao](02-spring-core/05-configuracao.md)
- **Starter:** dependência agregadora do Boot para um caso de uso. → [01-o-que-e-spring-boot](03-spring-boot/01-o-que-e-spring-boot.md)
- **Stereotype (estereótipo):** `@Component`, `@Service`, `@Repository`, `@Controller`. → [03](02-spring-core/03-application-context-e-beans.md)
- **Stream (API):** pipeline funcional de dados; não confundir com I/O streams. → [09-programacao-funcional](01-java/09-programacao-funcional.md)

## T
- **Testcontainers:** contêineres Docker descartáveis em testes. → [10-testes](03-spring-boot/10-testes-no-spring-boot.md)
- **Tracing distribuído:** acompanha uma requisição entre serviços (trace/span). → [08-tracing](04-spring-cloud/08-tracing-distribuido.md)
- **Type erasure:** remoção dos argumentos de tipo na compilação. → [06-generics](01-java/06-generics.md)

## V
- **Virtual thread:** thread leve gerenciada pela JVM (Java 21). → [12-concorrencia](01-java/12-concorrencia.md)
- **`volatile`:** garante visibilidade, não atomicidade composta. → [12](01-java/12-concorrencia.md)

## W
- **WebFlux:** pilha web reativa do Spring. → [05-webflux](03-spring-boot/05-webflux-e-reativo.md)
