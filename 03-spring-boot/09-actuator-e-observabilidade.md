# Actuator e Observabilidade

> **Resumo:** o Actuator expõe o estado interno de uma aplicação Boot em produção (saúde, métricas, configuração, beans, tarefas agendadas, migrações) por HTTP ou JMX. Combinado com Micrometer (métricas), logs estruturados e traces (OpenTelemetry), forma a base da observabilidade: responder "o que está acontecendo lá dentro?" sem precisar de debugger.
> **Pré-requisitos:** [01-o-que-e-spring-boot](01-o-que-e-spring-boot.md), [02-auto-configuracao](02-auto-configuracao.md), [03-configuracao-externalizada](03-configuracao-externalizada.md), [08-spring-security](08-spring-security.md) (proteger endpoints).
> **Versões:** Spring Boot 4.1.2, Micrometer 1.x/2.x ⚠️ VERIFICAR a versão no BOM, Java 25. Exemplos de §5 executados com MockMvc (`/tmp/validacao/s9` e `s9b`): health, probes, info, flyway, caches, metrics, scheduledtasks, env, `HealthIndicator` próprio e `Counter` próprio. Prometheus, OTLP e traces **não executados**.

## 1. Definição

**Actuator** é o módulo `spring-boot-starter-actuator`: um conjunto de *endpoints* operacionais (`/actuator/health`, `/actuator/metrics`...) mais a infraestrutura para criar os seus. **Observabilidade** é a propriedade de um sistema permitir inferir seu estado interno a partir de suas saídas, tradicionalmente os três pilares:

| Pilar | Pergunta | Ferramenta no ecossistema Spring |
|---|---|---|
| **Métricas** | quanto? com que frequência? quão lento? | Micrometer → Prometheus, etc. |
| **Logs** | o que aconteceu neste evento? | Logback/Log4j2, logs estruturados (ECS, GELF, Logstash) |
| **Traces** | por onde esta requisição passou? | Micrometer Tracing / OpenTelemetry |

Problema que resolve: sem isso, descobrir por que uma aplicação está lenta exige adivinhação ou acesso ao servidor. Com isso, o orquestrador (Kubernetes) sabe quando reiniciar ou retirar a instância do balanceador, e o time vê a latência por rota antes que o cliente reclame.

## 2. Conceitos fundamentais

### 2.1 Endpoints

Cada endpoint tem um **id** e pode ser **habilitado** (existe como bean) e **exposto** (acessível por HTTP/JMX). São duas decisões independentes.

| id | O que mostra | Observação |
|---|---|---|
| `health` | estado agregado UP/DOWN/OUT_OF_SERVICE | o único exposto por padrão via HTTP |
| `info` | metadados (`info.*`, git, build, java, os) | |
| `metrics` | métricas do Micrometer | lista e detalha por nome/tag |
| `env` | `Environment` e `PropertySource`s | valores sensíveis são mascarados (`******`) |
| `configprops` | beans `@ConfigurationProperties` | |
| `beans`, `conditions`, `mappings` | beans, relatório de auto-configuração, rotas | úteis no diagnóstico |
| `loggers` | níveis de log; **aceita POST para mudar em runtime** | |
| `threaddump`, `heapdump` | diagnóstico da JVM | `heapdump` contém dados em memória: nunca exponha |
| `flyway`, `liquibase` | migrações aplicadas | ver [12-migracoes-de-banco](12-migracoes-de-banco.md) |
| `caches` | caches e `CacheManager`s | ver [11](11-cache-agendamento-e-async.md) |
| `scheduledtasks` | `@Scheduled` registrados | |
| `shutdown` | encerra a aplicação | desabilitado por padrão |
| `prometheus` | formato de scraping Prometheus | requer o registro Prometheus no classpath |

### 2.2 Exposição: o padrão é restritivo

Executado em `s9b` (sem nenhuma configuração):

```text
GET /actuator        → {"_links":{"self":..., "health":..., "health-path":...}}
GET /actuator/health → {"groups":["liveness","readiness"],"status":"UP"}
GET /actuator/metrics → 404
```

Só `health` é exposto, e **sem detalhes** (`show-details=never`). Para ampliar:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus   # lista explícita é melhor que "*"
  endpoint:
    health:
      show-details: when-authorized               # never | when-authorized | always
```

`include: "*"` expõe tudo que está habilitado. Em desenvolvimento é cômodo; em produção, combinado com ausência de segurança, é um incidente esperando acontecer (`env`, `heapdump`, `loggers`).

### 2.3 Health: indicadores, grupos e probes

O `/actuator/health` agrega vários `HealthIndicator`s. O status final é o "pior" entre eles (ordem padrão: `DOWN` > `OUT_OF_SERVICE` > `UP` > `UNKNOWN`). Com `show-details: always`, executado:

```json
{"components":{
  "db":{"status":"UP","details":{"database":"H2","validationQuery":"isValid()"}},
  "diskSpace":{"status":"UP","details":{"total":10059448320,"free":9970757632,"threshold":10485760}},
  "livenessState":{"status":"UP"}, "readinessState":{"status":"UP"},
  "ping":{"status":"UP"}, "ssl":{"status":"UP","details":{...}},
  "stock":{"status":"UP","details":{"items":42}}},
 "groups":["liveness","readiness"],"status":"UP"}
```

Os indicadores `db`, `diskSpace`, `ping`, `ssl` vieram de auto-configuração condicional ([02-auto-configuracao](02-auto-configuracao.md)): existe um `DataSource`, então existe `db`. O `stock` é nosso (§5.2).

**Liveness e readiness (probes):** com `management.endpoint.health.probes.enabled=true` (o metadado do Boot 4.1.2 mostra padrão `true`; em versões anteriores só era ativado automaticamente em Kubernetes) existem `/actuator/health/liveness` e `/actuator/health/readiness`:

| Probe | Pergunta | Se falhar, o Kubernetes... |
|---|---|---|
| **liveness** | o processo está num estado irrecuperável? | **reinicia** o contêiner |
| **readiness** | consegue atender tráfego agora? | **tira do balanceador** (não reinicia) |

Regra de ouro: **não ponha dependências externas (banco, broker) no liveness.** Se o banco cair e o liveness falhar, o Kubernetes reiniciaria todas as instâncias em cascata sem resolver nada. Dependências externas, se houver, ficam no readiness (ou fora das probes).

### 2.4 Métricas (Micrometer)

Micrometer é uma fachada de métricas independente de fornecedor (como o SLF4J para logs). Você registra `Counter`, `Gauge`, `Timer`, `DistributionSummary` num `MeterRegistry`; o *registry* concreto (Prometheus, OTLP, Datadog...) traduz para o formato do backend.

| Tipo | Uso | Exemplo |
|---|---|---|
| `Counter` | só cresce | pedidos criados |
| `Gauge` | valor instantâneo que sobe e desce | tamanho de fila |
| `Timer` | duração + contagem + máximo | latência de chamada |
| `DistributionSummary` | distribuição de tamanhos | bytes por resposta |

**Tags (dimensões)** permitem fatiar (`channel=web`), mas cada combinação distinta de valores cria uma série temporal. Tag com valor de alta cardinalidade (id de cliente, URL com parâmetro) explode a memória da aplicação e a conta do backend. Isso é o erro de métricas mais caro de cometer.

O Boot instrumenta sozinho: JVM (`jvm.memory.used`, `jvm.gc.*`, `jvm.threads.*`), CPU, `http.server.requests`, pool do Hikari, caches, executores, Kafka/Rabbit quando presentes. Executado: `/actuator/metrics/jvm.memory.used` retornou `availableTags` com `area` e `id`.

### 2.5 Logs, traces e a API de Observation

- **Logs estruturados** (Boot 3.4+): `logging.structured.format.console=ecs` (ou `logstash`, `gelf`) emite JSON em vez de texto. ⚠️ VERIFICAR os valores aceitos no 4.1.
- **Traces:** o `traceId`/`spanId` entram no MDC dos logs e viajam em cabeçalhos entre serviços (W3C `traceparent`). No Boot 4 o caminho recomendado é o starter `spring-boot-starter-opentelemetry` (existe no repositório Maven na versão 4.1.2), que exporta métricas, traces e logs via OTLP.
- **Observation API:** uma única instrumentação (`Observation`) gera métrica **e** span. Frameworks (MVC, `RestClient`, Data, Kafka) já a usam; no seu código, `@Observed` (com o aspecto) ou `ObservationRegistry` direto.

## 3. Como funciona por dentro

### 3.1 Um endpoint é um bean com operações

Um endpoint é uma classe anotada com `@Endpoint(id = "...")` cujos métodos `@ReadOperation`, `@WriteOperation` e `@DeleteOperation` são **adaptados** para cada tecnologia (um *web extension* transforma em rota HTTP; outro em operação JMX). Por isso o mesmo código aparece em `GET /actuator/x`, em JMX e (via anotações específicas) em servlet ou WebFlux.

```mermaid
flowchart LR
    E["@Endpoint(id='stock')<br/>@ReadOperation"] --> D[EndpointDiscoverer]
    D --> W["WebEndpointDiscoverer<br/>(filtra por exposure.include)"]
    D --> J[JmxEndpointDiscoverer]
    W --> H["HandlerMapping do Actuator<br/>/actuator/stock"]
    J --> M[MBean]
```

A *exposição* é um filtro aplicado depois da descoberta: o endpoint continua sendo um bean mesmo quando não exposto.

### 3.2 Health: composição

`HealthEndpoint` consulta um `HealthContributorRegistry`. Cada `@Component` que implementa `HealthIndicator` (ou `ReactiveHealthIndicator`) é registrado com o nome do bean sem o sufixo `HealthIndicator` (o nosso bean `stock` virou o componente `stock`). Um `StatusAggregator` calcula o status final. **Grupos** (`management.endpoint.health.group.<nome>.include`) são subconjuntos com seu próprio agregador, e é assim que `liveness` e `readiness` existem.

Cuidado de desempenho: o indicador é executado **a cada requisição** ao `/health`. Chamar um serviço lento ali faz o probe do Kubernetes estourar o timeout e derrubar a instância por uma dependência que ela nem usa naquele instante.

### 3.3 Métricas: do `MeterRegistry` ao backend

```mermaid
flowchart LR
    C["Seu código<br/>Counter / Timer"] --> R["MeterRegistry<br/>(composite)"]
    I["Instrumentação do Boot<br/>JVM, HTTP, Hikari"] --> R
    R --> P["PrometheusMeterRegistry<br/>(pull: /actuator/prometheus)"]
    R --> O["OtlpMeterRegistry<br/>(push)"]
    R --> S["SimpleMeterRegistry<br/>(em memória: o que /metrics lê)"]
```

Sem um registro de produção no classpath, o Boot usa o `SimpleMeterRegistry` em memória: é o que alimenta `/actuator/metrics` mesmo sem Prometheus. Os `MeterBinder`s (JVM, Hikari...) são aplicados a **todos** os registros. Em testes com `@SpringBootTest`, o Boot desliga a exportação (vimos `management.defaults.metrics.export.enabled` no `/env` do teste), porque exportar métricas de teste para um backend real seria ruído.

### 3.4 Mascaramento no `/env`

No `/actuator/env` do teste, quase todos os valores apareceram como `******`. No Boot 3.x+ o padrão é **mascarar tudo** (`management.endpoint.env.show-values=never`); é preciso optar por `always` ou `when-authorized`. Chaves que contêm `password`, `secret`, `token` etc. eram mascaradas nas versões antigas, mas padrão por nome nunca foi garantia: `jdbc:postgresql://user:senha@host/db` passa por qualquer lista de palavras-chave.

## 4. Evolução histórica

| Versão | Mudança |
|---|---|
| Boot 1.0 (2014) | Actuator nasce com endpoints `/health`, `/metrics`, `/env`, `/beans`, sem prefixo, fora do Spring MVC |
| Boot 2.0 (2018) | Reescrita: modelo `@Endpoint` independente de tecnologia; prefixo `/actuator`; **Micrometer** substitui o sistema de métricas próprio; só `health` e `info` expostos |
| Boot 2.2–2.3 | probes `liveness`/`readiness` e grupos de health |
| Boot 3.0 (2022) | **Observation API**, Micrometer Tracing (substitui o Spring Cloud Sleuth), suporte a OTel |
| Boot 3.4 | logs estruturados nativos |
| Boot 4.0 | Boot modularizado; Actuator separado em `spring-boot-actuator`/`-autoconfigure`; `HealthIndicator` migrou para `org.springframework.boot.health.contributor`; starters para OpenTelemetry e Micrometer |

⚠️ VERIFICAR as versões exatas dos marcos 2.2/2.3/3.4 (de memória). **Armadilha de migração executada:** o import clássico `org.springframework.boot.actuate.health.HealthIndicator` **não compila no 4.1.2**: a classe agora está em `org.springframework.boot.health.contributor` (confirmado listando o jar `spring-boot-health`). Veja a evolução geral em [01-o-que-e-spring-boot §4](01-o-que-e-spring-boot.md).

## 5. Exemplos práticos

### 5.1 Dependências e configuração

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,flyway,caches,scheduledtasks,loggers
  endpoint.health.show-details: always       # só em dev; em prod: when-authorized
  endpoint.health.probes.enabled: true
  info.env.enabled: true                     # expõe info.* do Environment
info.app.name: orders
```

Resultado executado: `GET /actuator/info` → `{"app":{"name":"orders"}}`.

### 5.2 `HealthIndicator` próprio (executado)

```java
package com.example.orders;

import org.springframework.boot.health.contributor.*;   // Boot 4 (era ...actuate.health)
import org.springframework.stereotype.Component;

@Component("stock")
class StockHealth implements HealthIndicator {
    volatile boolean ok = true;

    @Override
    public Health health() {
        return ok
            ? Health.up().withDetail("items", 42).build()
            : Health.down().withDetail("reason", "empty").build();
    }
}
```

Com `ok = false`, executado: `GET /actuator/health` → **503**. O status HTTP segue o status agregado (`UP`→200, `DOWN`→503), que é o que o balanceador e o Kubernetes olham.

### 5.3 Métrica de negócio (executado)

```java
@Service
class OrderService {
    private final Counter placed;

    OrderService(MeterRegistry registry) {
        this.placed = Counter.builder("orders.placed")
                .tag("channel", "web")          // baixa cardinalidade: poucos valores
                .register(registry);
    }

    void place() { placed.increment(); }
}
```

Executado após duas chamadas, com `exposure.include=health,metrics`:

```json
GET /actuator/metrics/orders.placed
{"name":"orders.placed","measurements":[{"statistic":"COUNT","value":2.0}],
 "availableTags":[{"tag":"channel","values":["web"]}]}
```

Para medir duração, use `Timer.builder("orders.pricing").register(registry).record(() -> ...)` ou `@Timed` (precisa do `TimedAspect`).

### 5.4 Inspecionando migrações, caches e tarefas (executado)

```json
GET /actuator/flyway
{"contexts":{"application":{"flywayBeans":{"flyway":{"migrations":[
  {"version":"1","description":"create orders","script":"V1__create_orders.sql","state":"SUCCESS","installedRank":1,...},
  {"version":"2","description":"add status","script":"V2__add_status.sql","state":"SUCCESS","installedRank":2,...}]}}}}}

GET /actuator/scheduledtasks
{"fixedRate":[{"interval":200,"runnable":{"target":"com.example.orders.Jobs.tick"},
               "lastExecution":{"status":"SUCCESS",...}}], "cron":[],...}

GET /actuator/caches
{"cacheManagers":{"cacheManager":{"caches":{"prices":{"target":"java.util.concurrent.ConcurrentHashMap"}}}}}
```

Observação executada: o endpoint `caches` só listou `prices` depois do primeiro uso, porque o `ConcurrentMapCacheManager` cria caches **sob demanda**. Para listar desde o início, declare `spring.cache.cache-names`.

### 5.5 Mudar nível de log em runtime

```bash
curl -X POST localhost:8080/actuator/loggers/com.example.orders \
     -H 'Content-Type: application/json' -d '{"configuredLevel":"DEBUG"}'
```

Útil em incidente; a mudança vale até o reinício e **só para aquela instância**. Exponha `loggers` só com segurança (`POST` altera estado).

### 5.6 Proteger os endpoints

Em conjunto com [08-spring-security](08-spring-security.md), use o matcher dedicado:

```java
@Bean
SecurityFilterChain actuator(HttpSecurity http) throws Exception {
    http.securityMatcher(EndpointRequest.toAnyEndpoint())
        .authorizeHttpRequests(a -> a
            .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
            .anyRequest().hasRole("OPS"))
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

<!-- não compilado: o pacote de `EndpointRequest` no Boot 4 ⚠️ VERIFICAR (moveu junto com a modularização) -->

Alternativa mais robusta: servir o Actuator em **outra porta** (`management.server.port=9090`) que não é publicada para fora do cluster.

### 5.7 Prometheus e OTLP (não executado)

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Com isso (e `prometheus` na exposição) o endpoint `/actuator/prometheus` passa a existir em formato texto do Prometheus. Para OTLP, o starter `spring-boot-starter-opentelemetry`; o destino vem de propriedades `management.otlp.*` ⚠️ VERIFICAR os nomes exatos no 4.1.

## 6. Configurações e anotações relevantes

| Item | Para que serve | Exemplo |
|---|---|---|
| `management.endpoints.web.exposure.include/exclude` | quais endpoints ficam acessíveis | `include: health,info` |
| `management.endpoint.<id>.enabled` ⚠️ VERIFICAR nome no 4.x | liga/desliga o bean do endpoint | `shutdown` |
| `management.endpoint.health.show-details` | detalhes no health | `when-authorized` |
| `management.endpoint.health.probes.enabled` | `liveness`/`readiness` | `true` |
| `management.endpoint.health.group.<g>.include` | grupo customizado | `readiness.include: db` |
| `management.endpoint.env.show-values` | mostrar valores no `/env` (padrão `never`, confirmado) | `when-authorized` |
| `management.server.port` / `.address` | porta/endereço separados | `9090` |
| `management.endpoints.web.base-path` | muda `/actuator` | `/internal` |
| `management.metrics.tags.*` | tags comuns a todas as métricas | `application: orders` |
| `@Endpoint`, `@ReadOperation` | endpoint próprio | §3.1 |
| `HealthIndicator` | contribuinte de health | §5.2 |
| `MeterRegistry`, `@Timed`, `@Observed` | métricas/observações próprias | §5.3 |
| `info.*` / `management.info.<contributor>.enabled` | conteúdo do `/info` | `env`, `git`, `build`, `java` |

## 7. Quando usar e quando NÃO usar

**Use:** em toda aplicação que roda em produção. `health` + probes é o mínimo; métricas HTTP e JVM são quase de graça.

**Não exponha** `env`, `heapdump`, `threaddump`, `loggers`, `beans`, `mappings`, `configprops` na internet. Eles revelam topologia, propriedades e dados em memória.

**Não use Actuator como API de negócio**, nem o `/health` para checar regras de domínio: é contrato com a infraestrutura.

**Não crie tags de alta cardinalidade** (id do pedido, e-mail, URL crua).

## 8. Erros comuns e boas práticas

1. **`exposure.include: "*"` em produção sem segurança.** Prefira lista explícita e porta de gerenciamento separada.
2. **Dependência externa no liveness** causa reinícios em cascata (§2.3).
3. **`HealthIndicator` lento ou que lança exceção**: a exceção vira `DOWN`; o lento estoura o timeout do probe. Use timeout curto e cache do resultado.
4. **Import antigo (`actuate.health`) no Boot 4**: erro de compilação (§4).
5. **Tag de alta cardinalidade**: exemplo clássico, `tag("orderId", id)`.
6. **Esperar `/metrics` sem expor**: 404 por padrão (executado, §2.2).
7. **Agrupar sem normalizar a URL**: `http.server.requests` usa o *template* da rota (`/orders/{id}`), não a URL crua; mantenha as rotas parametrizadas para a cardinalidade ficar controlada.
8. **Tag `application` ausente**: sem `management.metrics.tags.application`, vários serviços se misturam no mesmo painel.

## 9. Perguntas de revisão

1. Qual a diferença entre um endpoint estar *habilitado* e *exposto*?
<details><summary>Resposta</summary>Habilitado = o bean existe; exposto = acessível por HTTP/JMX. A exposição é filtro aplicado depois da descoberta. Por padrão só `health` é exposto por HTTP.</details>

2. Por que o liveness não deve verificar o banco?
<details><summary>Resposta</summary>Se o banco cair, o liveness falharia em todas as instâncias e o Kubernetes as reiniciaria em cascata sem resolver o problema. Dependências externas ficam no readiness (que só retira do balanceador).</details>

3. O que acontece com o status HTTP do `/actuator/health` quando um indicador fica `DOWN`?
<details><summary>Resposta</summary>O status agregado vira `DOWN` e a resposta é 503 (executado em §5.2).</details>

4. Por que tags com id do pedido são um problema?
<details><summary>Resposta</summary>Cada valor distinto cria uma série temporal nova (alta cardinalidade), consumindo memória na aplicação e custo/armazenamento no backend.</details>

5. Como o mesmo `@Endpoint` aparece em HTTP e em JMX?
<details><summary>Resposta</summary>As operações são descobertas por `EndpointDiscoverer`s e adaptadas por tecnologia (web, JMX); o código do endpoint é independente do transporte.</details>

## 10. Referências oficiais

- Spring Boot Reference, Actuator: <https://docs.spring.io/spring-boot/reference/actuator/index.html> ⚠️ VERIFICAR o caminho exato
- Micrometer: <https://micrometer.io>
- Observabilidade no Spring: <https://docs.spring.io/spring-boot/reference/actuator/observability.html> ⚠️ VERIFICAR
- Projetos Spring: <https://spring.io/projects/spring-boot>
- Próximo: [10-testes-no-spring-boot](10-testes-no-spring-boot.md)
