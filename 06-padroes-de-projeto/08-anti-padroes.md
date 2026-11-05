# Anti-padrões (God Class, Anemic Domain Model, field injection, `@Transactional` mal usado, OSIV e outros)

> **Resumo:** um **anti-padrão** é uma solução que parece razoável, é muito repetida e **produz mais problemas do que resolve**. Conhecê-los é a outra metade dos padrões: você os reconhece em revisão de código, sabe explicar por que quebram e sabe a saída. Esta página cobre os anti-padrões mais comuns em Java e Spring, cada um com **sintoma, causa, consequência medida e correção**, e inclui o debate honesto em torno do *Anemic Domain Model*.
> **Pré-requisitos:** [01-introducao](01-introducao.md) (SOLID), [05-padroes-enterprise](05-padroes-enterprise.md), [06-padroes-arquiteturais](06-padroes-arquiteturais.md), [02-ioc-e-injecao-de-dependencia](../02-spring-core/02-ioc-e-injecao-de-dependencia.md), [09-transacoes](../02-spring-core/09-transacoes.md).
> **Versões:** Java 25, Spring Framework 7.0.8, Spring Boot 4.1.2. As demonstrações das §5.1–5.4 foram compiladas e executadas (`/tmp/validacao/s15`); as de `@Transactional`, OSIV e N+1 foram executadas nas sessões 6 e 8 e são referenciadas.

## 1. Definição

O termo vem de *AntiPatterns* (Brown, Malveau, McCormick e Mowbray, 1998) e da ideia de Andrew Koenig (1995). Um bom anti-padrão tem:

1. uma **forma recorrente** (reconhecível no código);
2. uma **consequência ruim** previsível;
3. uma **refatoração** conhecida.

Não confunda com **bug** (comportamento incorreto pontual) nem com **code smell** (indício de problema, como método longo). Um anti-padrão é o problema estrutural que costuma estar por trás de vários *smells*.

| Anti-padrão | Categoria | Sintoma típico |
|---|---|---|
| God Class / Blob | projeto | classe com milhares de linhas e dezenas de dependências |
| Anemic Domain Model | projeto | entidades só com getters/setters; regras nos serviços |
| Field injection | Spring/DI | `@Autowired` em campo `private` |
| `@Transactional` mal usado | Spring/transação | anotação em método privado, chamada interna, `catch` que engole |
| Open Session in View | Spring/JPA | consultas disparadas pela view; conexão presa na resposta |
| N+1 queries | JPA | uma consulta por item de lista |
| Entidade JPA exposta na API | camadas | `LazyInitializationException`, vazamento de campos |
| Singleton com estado mutável | concorrência | contador/lista compartilhados sem sincronização |
| Exceções engolidas / `catch (Exception)` | tratamento de erros | falha silenciosa |
| Primitive obsession | projeto | `String`/`BigDecimal` por todo lado em vez de tipos de domínio |
| Service Locator / `ApplicationContext.getBean` | DI | dependências escondidas |
| Distributed Monolith | arquitetura | microsserviços que só funcionam juntos |
| Golden Hammer, Premature Optimization, Cargo Cult | processo | mesmo padrão em tudo, sem medir |

## 2. Conceitos fundamentais

### 2.1 God Class (Blob)

Uma classe que sabe e faz demais: valida, calcula, persiste, formata, envia e-mail. Viola o **SRP** e o **OCP** ([01-introducao §5.1](01-introducao.md)). Sinais: nome genérico (`OrderManager`, `Utils`), centenas de linhas, muitos campos, dezenas de dependências injetadas, testes enormes com muitos mocks. Atrai mudanças de todos os motivos, então todo PR conflita nela.

### 2.2 Anemic Domain Model (e o debate)

Descrito por Martin Fowler em 2003: o domínio tem classes com os nomes certos (`Order`, `Customer`), mas elas são **sacos de dados** (getters e setters) e **toda a lógica** está em classes `*Service` que manipulam esses dados. Fowler chama de anti-padrão porque parece orientado a objetos, mas na prática é programação procedural com os custos de OO (mapeamento ORM) sem os benefícios (encapsulamento e invariantes).

**O debate, com honestidade:**

- **Contra o anêmico (Fowler/Evans):** invariantes ficam espalhadas; qualquer código pode deixar o objeto em estado inválido (§5.2 mostra "pago e vazio"); regras duplicadas em vários serviços; difícil testar a regra isolada.
- **A favor de modelos anêmicos (defesa legítima):** (1) em CRUD simples, ou sistemas *data-centric*, não há regra para encapsular; (2) em *transaction script* (Fowler, PoEAA), um serviço procedural é **mais simples** e perfeitamente adequado; (3) programação funcional separa dados imutáveis de funções de propósito (um `record` + funções puras não é "anêmico" no sentido ruim); (4) equipes grandes às vezes preferem separar dados de comportamento por motivos operacionais.
- **Conclusão prática:** o problema não é "ter getters", é **ter regras de negócio com invariantes e deixá-las fora do objeto que as deveria proteger**. Se o domínio tem regras ricas, mova-as para as entidades; se é CRUD, aceitar um modelo simples e chamá-lo de *transaction script* é honesto e correto.

### 2.3 Field injection

`@Autowired private PaymentGateway gateway;`. Funciona, mas:

- a dependência fica **invisível** na assinatura do construtor;
- o campo não pode ser `final` → a classe não é imutável nem *thread-safe* por construção;
- sem container, o campo é `null` (teste unitário sem Spring dá `NullPointerException`, §5.1);
- permite um **bean com dezenas de dependências** sem que isso incomode (esconde a God Class);
- permite **ciclos** sem aviso (o Spring Framework resolve ciclo por campo/setter por padrão; ver [04-ciclo-de-vida-e-escopos §3.3](../02-spring-core/04-ciclo-de-vida-e-escopos.md)), enquanto o ciclo por construtor falha na hora, o que é bom (validado em [02-ioc](../02-spring-core/02-ioc-e-injecao-de-dependencia.md));
- acopla o código ao framework (precisa de reflection para injetar).

A recomendação da documentação do Spring é **injeção por construtor** para dependências obrigatórias (construtor único dispensa `@Autowired`).

### 2.4 `@Transactional` mal usado

A anotação funciona por **proxy** ([07-aop-e-proxies](../02-spring-core/07-aop-e-proxies.md), [09-transacoes](../02-spring-core/09-transacoes.md)); os usos errados vêm de esquecer isso:

| Erro | Efeito | Referência |
|---|---|---|
| Chamar método `@Transactional` pela própria classe (*self-invocation*) | o proxy não intercepta; sem transação nova | validada na sessão 6 |
| Exceção **checked** esperando rollback | por padrão só `RuntimeException`/`Error` fazem rollback; checked **faz commit** | matriz de 9 cenários validada |
| `try/catch` que engole a exceção dentro do método | o proxy não vê a falha → commit | idem |
| Transação ampla demais (chamada HTTP/e-mail dentro) | conexão do pool presa durante I/O lento | — |
| `@Transactional` em controlador / em repositório | fronteira errada | [05 §8](05-padroes-enterprise.md) |
| Método `private`/`final` em proxy de classe | não é interceptado | [07-aop](../02-spring-core/07-aop-e-proxies.md) |
| Propagação `REQUIRES_NEW` sem entender | outra conexão, outro commit; pode causar deadlock com a transação externa | idem |
| `readOnly = true` achando que é segurança | é uma dica de otimização, não impede escrita em todos os provedores | — |

### 2.5 Open Session in View (OSIV)

Com `spring.jpa.open-in-view=true` (**o padrão no Boot**), o `EntityManager` fica aberto durante toda a requisição, inclusive a serialização da resposta. Parece conveniente: associações lazy "funcionam" no controlador/serialização. O custo: consultas SQL acontecem **fora do serviço e fora da transação**, difíceis de ver; a conexão do pool pode ser retida durante a renderização; N+1 escondido. O Boot registra na inicialização o aviso `spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning` (texto conferido em `spring-boot-jpa-4.1.2.jar`; aparece só em aplicação web e quando a propriedade não está definida). Correção: `spring.jpa.open-in-view=false`, carregar o necessário no serviço (`join fetch`, `@EntityGraph`, projeções/DTO) ([06-spring-data-jpa](../03-spring-boot/06-spring-data-jpa.md)).

### 2.6 Outros anti-padrões frequentes

- **N+1:** `findAll()` de pedidos e, em cada um, `getCustomer().getName()`. Medido na sessão 8: **6 consultas** viram **1** com `join fetch`/`@EntityGraph`.
- **Entidade JPA como DTO:** ver [05 §2.3](05-padroes-enterprise.md).
- **Singleton com estado mutável:** o bean Spring é singleton por padrão e **compartilhado entre threads**; guardar dados de uma requisição em campo do bean causa *race condition* (§5.4).
- **`catch (Exception e) {}`:** engole `InterruptedException`, `NullPointerException`, tudo ([08-excecoes](../01-java/08-excecoes.md), [12-concorrencia](../01-java/12-concorrencia.md)).
- **Primitive obsession:** `String email`, `BigDecimal amount`, `long customerId` trocáveis entre si; use `record Email`, `Money`, `CustomerId`.
- **Service Locator:** `context.getBean(X.class)` dentro da lógica, escondendo dependências; vence-se com DI por construtor. Exceção legítima: `ObjectProvider` para dependências opcionais ou *prototype* em singleton ([04-ciclo-de-vida](../02-spring-core/04-ciclo-de-vida-e-escopos.md)).
- **Distributed Monolith:** [07 §8](07-padroes-de-microsservicos.md).
- **Cargo cult / Golden Hammer:** usar Hexagonal, CQRS ou microsserviços "porque é moderno" ([06 §7](06-padroes-arquiteturais.md)).

## 3. Como funciona por dentro

### 3.1 Por que a field injection quebra fora do Spring

O `AutowiredAnnotationBeanPostProcessor` percorre os campos anotados e usa **reflection** (`Field.setAccessible(true)` + `set`) para gravar a dependência depois do `new` ([06-como-java-e-spring-conversam](../02-spring-core/06-como-java-e-spring-conversam.md), [10-anotacoes-e-reflection](../01-java/10-anotacoes-e-reflection.md)). Sem o contêiner, `new FieldInjected()` produz um objeto com o campo `null`. Com construtor, o compilador **exige** a dependência, e `final` garante que ela nunca mude.

### 3.2 Por que o singleton compartilhado corrompe dados

Um bean singleton é **uma** instância com **um** conjunto de campos, acessada por todas as threads do servidor (um pool de dezenas ou, com virtual threads, milhares). `n++` num `int` é leitura-modificação-escrita, não atômica; sem sincronização, o *Java Memory Model* não garante visibilidade nem atomicidade ([12-concorrencia](../01-java/12-concorrencia.md)). O resultado deixa de ser determinístico (§5.4).

### 3.3 Por que `catch` que engole "desliga" o rollback

O interceptor transacional envolve a chamada do método com `try/catch`: se uma exceção que atende às regras de rollback **sai** do método, ele marca/realiza rollback; se o código interno a capturou, o método retorna normalmente e o proxy faz **commit**. A exceção que o `catch` engoliu nunca chegou ao proxy ([09-transacoes](../02-spring-core/09-transacoes.md)).

### 3.4 OSIV no ciclo da requisição

O Boot registra (`JpaBaseConfiguration$JpaWebConfiguration`, conferido no JAR) um interceptor (`org.springframework.orm.jpa.support.OpenEntityManagerInViewInterceptor`) no MVC que abre o `EntityManager` no `preHandle` e o fecha no `afterCompletion` ([04-web-mvc](../03-spring-boot/04-web-mvc.md): `HandlerInterceptor`). Tudo entre os dois, inclusive a serialização para JSON, pode disparar `SELECT`s por associações lazy. Com OSIV desligado, o mesmo acesso fora da transação lança `LazyInitializationException`, o que é uma **proteção**: obriga a decidir o que carregar.

## 4. Evolução histórica

- **1995–1998:** Koenig e o livro *AntiPatterns* (1998) nomeiam o conceito; God Class, Spaghetti Code, Golden Hammer.
- **1999:** *Refactoring* (Fowler) cataloga *code smells* e as refatorações.
- **2003:** Fowler publica o post *AnemicDomainModel*. Evans lança o DDD no mesmo ano.
- **Spring 2.5 (2007):** `@Autowired` em campos populariza a field injection. Spring 4.3 permite construtor único sem `@Autowired`; a documentação passa a recomendar construtor.
- **Boot 1.x–4.x:** OSIV ligado por padrão desde o início do Spring Boot; o aviso na inicialização veio depois (versão a confirmar).
- **Java 14–21:** `record` e `sealed` reduzem a *primitive obsession* e a boilerplate de DTO; veja [00-evolucao-do-java](../01-java/00-evolucao-do-java.md).

⚠️ VERIFICAR: anos de Koenig (1995), *AntiPatterns* (1998) e *Refactoring* (1999); versão do Spring em que a recomendação passou a ser o construtor e em que surgiu o aviso do OSIV.

## 5. Exemplos práticos

### 5.1 Field injection vs. construtor (executado)

```java
@interface Autowired {}           // anotação ilustrativa
static class Mailer { void send(String s) { System.out.println("mail: " + s); } }

static class FieldInjected {
    @Autowired private Mailer mailer;               // sem container: null
    void notifyUser() { mailer.send("oi"); }
}

static class CtorInjected {
    private final Mailer mailer;                    // obrigatória e imutável
    CtorInjected(Mailer m) { mailer = Objects.requireNonNull(m); }
    void notifyUser() { mailer.send("oi"); }
}

try { new FieldInjected().notifyUser(); }
catch (NullPointerException e) { System.out.println("field injection sem container: NPE"); }
new CtorInjected(new Mailer()).notifyUser();
try { new CtorInjected(null); } catch (NullPointerException e) { System.out.println("construtor: falha cedo"); }
```

Saída:

```
field injection sem container: NPE
mail: oi
construtor: falha cedo (NPE no new)
```

**Correção no Spring:**

```java
@Service
class NotificationService {
    private final Mailer mailer;                    // construtor único: @Autowired dispensável
    NotificationService(Mailer mailer) { this.mailer = mailer; }
}
```

Em teste unitário: `new NotificationService(new FakeMailer())`, sem contêiner. No Boot, `@MockitoBean` apenas em testes de contexto ([10-testes-no-spring-boot](../03-spring-boot/10-testes-no-spring-boot.md)).

### 5.2 Anemic vs. rico (executado)

```java
// Anêmico: dados soltos; a regra está no serviço
class AnemicOrder { BigDecimal total; String status; /* getters e setters */ }
class AnemicService {
    void pay(AnemicOrder o) { if (o.total.signum() > 0) o.status = "PAID"; }
}
class OtherPlace {                                      // outro módulo, outro dev
    void forcePaid(AnemicOrder o) { o.status = "PAID"; }   // ninguém impede
}

// Rico: o objeto protege suas invariantes
final class RichOrder {
    private BigDecimal total = BigDecimal.ZERO; private String status = "NEW";
    void add(BigDecimal p) { if (!status.equals("NEW")) throw new IllegalStateException("fechado"); total = total.add(p); }
    void pay() { if (total.signum() == 0) throw new IllegalStateException("vazio"); status = "PAID"; }
    String status() { return status; }
}
```

Executado: `forcePaid` num pedido de total 0 deixa `status=PAID total=0` (**pago e vazio**: invariante violada, e o `AnemicService` nem foi chamado); no rico, `pay()` num pedido vazio lança `IllegalStateException: vazio`. Observe que com JPA a entidade rica ainda precisa de construtor sem argumentos (`protected`) e, em geral, de acesso por campo, o que é compatível com ela expor poucos setters ([06-spring-data-jpa](../03-spring-boot/06-spring-data-jpa.md)).

### 5.3 Exceção engolida (executado)

```java
static void swallow() {
    try { Integer.parseInt("x"); } catch (Exception e) { /* ignora */ }
    System.out.println("swallow: seguiu como se nada");
}
```

Saída: `swallow: seguiu como se nada`. Mesmo com `@Transactional` em volta, o método retorna normalmente e há commit com dados possivelmente incompletos. **Correção:** capture só o que sabe tratar, registre com contexto (`log.warn("falha ao converter {}", valor, e)`), relance ou traduza ([08-excecoes](../01-java/08-excecoes.md)). Se precisar de rollback com exceção capturada, use `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` ou, melhor, não capture.

### 5.4 Bean singleton com estado mutável (executado)

```java
static class Counter { int n; }
static final Counter SHARED = new Counter();     // equivale a um bean singleton com um campo mutável

var pool = Executors.newFixedThreadPool(8);
for (int i = 0; i < 8; i++) pool.submit(() -> { for (int k = 0; k < 100_000; k++) SHARED.n++; });
pool.shutdown(); pool.awaitTermination(10, TimeUnit.SECONDS);
System.out.println("contador compartilhado: " + SHARED.n + " (esperado 800000)");
```

Saída numa execução: `contador compartilhado: 196235 (esperado 800000)`. O valor varia a cada execução e perde mais de 75% das somas. **Correções:** não guardar estado de requisição em bean (use parâmetros e variáveis locais), `AtomicInteger`/`LongAdder` para contadores, `ConcurrentHashMap` para caches, ou escopo `request` quando o estado é por requisição ([12-concorrencia](../01-java/12-concorrencia.md), [04-ciclo-de-vida-e-escopos](../02-spring-core/04-ciclo-de-vida-e-escopos.md)).

### 5.5 Os que já foram executados em outras páginas

- **Self-invocation, checked exception e `catch` em `@Transactional`:** matriz de 9 cenários em H2 ([09-transacoes](../02-spring-core/09-transacoes.md)); `@Cacheable` e `@Async` têm a mesma limitação ([11-cache-agendamento-e-async](../03-spring-boot/11-cache-agendamento-e-async.md)).
- **N+1 e `LazyInitializationException`:** 6 → 1 consulta com `join fetch` ([06-spring-data-jpa](../03-spring-boot/06-spring-data-jpa.md)).
- **`@Enable*` na classe principal quebrando `@WebMvcTest`:** reproduzido e corrigido ([10-testes-no-spring-boot](../03-spring-boot/10-testes-no-spring-boot.md)).
- **`@LoadBalanced RestClient.Builder` único quebrando o registro no Eureka:** [03-service-discovery](../04-spring-cloud/03-service-discovery.md).
- **Evento do Modulith descartado sem `@Transactional`:** [03-spring-modulith](../05-outros-projetos-spring/03-spring-modulith.md).

### 5.6 Como refatorar uma God Class

1. **Cobrir com testes** o comportamento atual (testes de caracterização).
2. **Agrupar** campos e métodos que mudam juntos (coesão) → candidatos a classes.
3. **Extrair** responsabilidades: *Extract Class*; o que é regra vai para o domínio, o que é orquestração fica no serviço, o que é I/O vira adaptador.
4. **Substituir** `if/switch` por tipo por Strategy/`sealed` ([04-comportamentais](04-comportamentais.md)).
5. **Injetar** as novas classes por construtor e deixar a antiga como fachada fina até migrar os chamadores (Strangler em pequena escala).

## 6. Configurações e anotações relevantes

| Item | Para que serve | Exemplo |
|---|---|---|
| `spring.jpa.open-in-view=false` | desliga o OSIV | `application.properties` |
| `@Transactional(rollbackFor = ...)` | fazer rollback também para checked | `rollbackFor = Exception.class` |
| `@Transactional(readOnly = true)` | dica de leitura, evita dirty checking | consultas |
| `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` | rollback sem lançar | em `catch` consciente |
| `spring.main.allow-circular-references` | permite/proíbe ciclos (por padrão, ciclo por campo/setter é resolvido) | `=false` para detectar |
| `@Lazy` na injeção | quebra ciclo (paliativo, não correção) | `@Lazy` no parâmetro |
| `ObjectProvider<T>` | dependência opcional/prototype sem Service Locator | `provider.getIfAvailable()` |
| `@EntityGraph`, `join fetch`, projeções | evitar N+1 e lazy fora da transação | `@EntityGraph(attributePaths = "items")` |
| `-parameters`, `-Xlint:all`, `-Werror` | o compilador acusa parte dos problemas | no `maven-compiler-plugin` |
| ArchUnit / Modulith `verify()` | automatizar regras de arquitetura | teste de arquitetura |
| SonarQube / SpotBugs / Error Prone | detectar *smells* e bugs comuns | no CI |

## 7. Quando usar e quando NÃO usar (aqui: quando o "anti-padrão" é aceitável)

Nem todo anti-padrão é proibido o tempo todo; o importante é **escolher com consciência**.

| Prática | Aceitável quando | Nunca quando |
|---|---|---|
| Modelo anêmico / transaction script | CRUD, relatórios, serviços de integração sem regra | há invariantes de negócio que várias partes violam |
| Field injection | testes (`@Autowired` em classe de teste do Spring), `@Configuration` simples | em código de produção com regras ou dependências obrigatórias |
| OSIV ligado | protótipo, aplicação pequena (com o aviso entendido) | API com alto tráfego, conexão de pool escassa |
| `@Transactional` amplo | operação curta, só banco | com chamadas remotas ou I/O lento dentro |
| Singleton com estado | estado imutável ou thread-safe (cache `ConcurrentHashMap`, `AtomicLong`) | estado mutável por requisição |
| `catch (Exception)` | na **borda** (handler global, `@ControllerAdvice`) que traduz erro | no meio da lógica de negócio |
| Service Locator | código de infraestrutura/framework | código de aplicação |

## 8. Erros comuns e boas práticas

1. **Chamar de anti-padrão sem avaliar o contexto.** O custo do "padrão correto" (mais classes, mapeamentos) pode ser maior que o do problema.
2. **Trocar um anti-padrão por outro:** tirar `@Autowired` de campo e criar construtor com 12 parâmetros esconde a God Class em outro lugar. Construtor grande é **sintoma** de classe grande; divida a classe.
3. **Usar `@Lazy` ou `allow-circular-references=true` para "resolver" ciclos** em vez de rever a modelagem.
4. **Confiar em `readOnly=true`** como controle de acesso.
5. **Tratar OSIV como "bug do Hibernate":** o `LazyInitializationException` é só o sintoma de não ter decidido o que carregar.
6. **Refatorar sem testes.** Sem rede de segurança, mover regra de serviço para entidade pode alterar o comportamento.
7. **Deixar `TODO`/`catch` vazio em produção**; configure o linter para falhar o build.
8. **Revisar código com checklist:** construtor com muitas dependências? `catch` vazio? estado mutável em bean? `@Transactional` com I/O? entidade na resposta?
9. **Medir antes de "otimizar"** (*premature optimization*): perfil com JFR/Actuator ([02-jvm-por-dentro](../01-java/02-jvm-por-dentro.md), [09-actuator-e-observabilidade](../03-spring-boot/09-actuator-e-observabilidade.md)).

## 9. Perguntas de revisão

1. Por que a injeção por construtor é preferida à injeção por campo?
   <details><summary>Resposta</summary>Explicita as dependências, permite `final`/imutabilidade, funciona em teste unitário sem contêiner (o campo vira `null` sem ele: NPE demonstrado), falha cedo e revela classes com dependências demais; o ciclo por construtor falha na inicialização em vez de passar despercebido.</details>
2. Em que condições um modelo de domínio anêmico é uma escolha razoável?
   <details><summary>Resposta</summary>Quando o sistema é essencialmente CRUD ou orientado a dados, sem invariantes ricas (transaction script), ou quando se adota estilo funcional com dados imutáveis e funções puras. O problema ocorre quando há regras de negócio que deveriam ser protegidas pelo objeto e estão espalhadas nos serviços.</details>
3. Um método `@Transactional` captura a exceção num `try/catch` e a registra no log. O que acontece com a transação?
   <details><summary>Resposta</summary>Há commit: a exceção não saiu do método, então o interceptor do proxy não a vê e não faz rollback. Relance, ou marque `setRollbackOnly()` conscientemente.</details>
4. Que problema o OSIV esconde e qual a alternativa?
   <details><summary>Resposta</summary>Esconde consultas lazy disparadas fora do serviço/transação (N+1, conexão presa durante a serialização). Alternativa: `spring.jpa.open-in-view=false` e carregar explicitamente no serviço (`join fetch`, `@EntityGraph`, DTO/projeção).</details>
5. Por que um campo `int` mutável em um bean singleton do Spring é perigoso?
   <details><summary>Resposta</summary>O singleton é compartilhado por todas as threads; `n++` não é atômico, e o resultado perde atualizações (demonstrado: 196235 em vez de 800000). Use variáveis locais, `AtomicInteger`/`LongAdder`, estruturas concorrentes ou escopo `request`.</details>

## 10. Referências oficiais

- *AntiPatterns* (Brown et al., 1998) e <https://sourcemaking.com/antipatterns> (catálogo comunitário)
- Martin Fowler, *AnemicDomainModel*: <https://martinfowler.com/bliki/AnemicDomainModel.html>
- Martin Fowler, *Refactoring*: <https://refactoring.com/catalog/>
- Spring Framework, Dependency Injection (construtor vs. setter): <https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html>
- Spring Framework, Transaction Management: <https://docs.spring.io/spring-framework/reference/data-access/transaction.html>
- Spring Boot, JPA e OSIV: <https://docs.spring.io/spring-boot/reference/data/sql.html>
- Hibernate ORM: <https://hibernate.org/orm/documentation/>
- ArchUnit: <https://www.archunit.org/>
- Anterior: [07-padroes-de-microsservicos](07-padroes-de-microsservicos.md). Seção: [01-introducao](01-introducao.md)
