# Empacotamento e Deploy

> **Resumo:** como uma aplicação Boot sai do código e vira algo que roda em produção: o *fat jar* executável, a extração em camadas para imagens Docker eficientes, o caminho com Buildpacks, as otimizações de startup (AOT cache da JVM, CDS, native image), o encerramento gracioso e os pontos de contato com Kubernetes (probes, configuração, recursos).
> **Pré-requisitos:** [01-o-que-e-spring-boot](01-o-que-e-spring-boot.md) (JarLauncher), [03-configuracao-externalizada](03-configuracao-externalizada.md), [09-actuator-e-observabilidade](09-actuator-e-observabilidade.md) (probes), [10-aot-e-native](../02-spring-core/10-aot-e-native.md), [14-modulos-e-build](../01-java/14-modulos-e-build.md).
> **Versões:** Spring Boot 4.1.2, Java 25, Maven 3.9.12, Docker disponível no ambiente de validação. Executado em `/tmp/validacao/s9`: `repackage`, `-Djarmode=tools list-layers/extract`, Dockerfile multi-estágio com `eclipse-temurin:25-jre`, execução do contêiner com usuário não-root, medição de startup com AOT cache da JVM. **Não executados:** `build-image` (Buildpacks), native image, Kubernetes, systemd.

## 1. Definição

**Empacotar** é gerar o artefato que contém a aplicação e suas dependências. **Deploy** é colocá-lo para rodar em um ambiente (VM, contêiner, orquestrador, nuvem). No ecossistema Boot o artefato padrão é o **jar executável** ("fat jar"): um único `.jar` com as classes da aplicação, todas as dependências aninhadas e um lançador, que roda com `java -jar`.

Problema que resolve: no modelo antigo, a aplicação era um `.war` instalado num servidor de aplicações (Tomcat, JBoss) configurado à parte. Um jar autocontido, com servidor embutido, torna a aplicação **um processo comum**, idêntico em desenvolvimento e produção, o que casa com contêineres e a abordagem de [12-factor app](https://12factor.net).

## 2. Conceitos fundamentais

### 2.1 Formas de entrega

| Forma | O que é | Quando |
|---|---|---|
| **Fat jar** | `java -jar app.jar` | VM, scripts, desenvolvimento |
| **Jar em camadas dentro de imagem OCI** | imagem Docker com camadas separadas por frequência de mudança | produção em contêiner (caminho mais comum) |
| **Buildpacks** (`spring-boot:build-image`) | imagem OCI gerada sem Dockerfile, pelo Paketo | equipes que preferem convenção |
| **Native image** | executável nativo (GraalVM) | startup em milissegundos, pouca memória; build lento ([02-spring-core/10](../02-spring-core/10-aot-e-native.md)) |
| **WAR** | para contêiner de servlets externo | legado; `SpringBootServletInitializer` |

### 2.2 O plugin de build

Com o `spring-boot-starter-parent`, basta declarar:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

Goals confirmados no `plugin.xml` do plugin 4.1.2: `repackage` (liga ao `package`), `run`, `start`/`stop` (testes de integração), `build-image`, `build-info` (alimenta `/actuator/info`), `process-aot`, `process-test-aot`, `test-run`. No Gradle, o plugin `org.springframework.boot` oferece `bootJar`, `bootRun`, `bootBuildImage`, `bootWar` (Gradle não executado neste ambiente).

### 2.3 Camadas (layers)

O jar é um ZIP; a imagem Docker é uma pilha de camadas, cada uma cacheada separadamente. Se tudo (dependências + código) entra numa camada, **cada alteração de uma linha de código reenvia ~27 MB** para o registro e para cada nó. Separando por frequência de mudança:

| Camada (no Boot) | Conteúdo | Muda |
|---|---|---|
| `dependencies` | jars de terceiros liberados | raramente |
| `spring-boot-loader` | classes do lançador | quase nunca |
| `snapshot-dependencies` | dependências `-SNAPSHOT` | às vezes |
| `application` | suas classes e recursos | a cada commit |

Executado: `java -Djarmode=tools -jar target/s9-1.jar list-layers` listou exatamente `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application`. Após `extract --layers`:

```text
 64K  ex/application              ← o que muda a cada deploy
 26M  ex/dependencies
  0   ex/snapshot-dependencies
604K  ex/spring-boot-loader
```

A camada que muda a cada commit tem **64 KB**, contra 26 MB do conjunto: cada deploy reenvia 64 KB em vez de 27 MB.

### 2.4 Encerramento gracioso e ciclo de vida

Desde o Boot 2.3 o servidor web sabe encerrar com elegância. Confirmado nos metadados do 4.1.2: `server.shutdown = graceful` (**padrão**) e `spring.lifecycle.timeout-per-shutdown-phase = 30s`. Ao receber `SIGTERM`, o servidor para de aceitar novas conexões, **espera as requisições em andamento terminarem** (até 30 s por fase) e só então fecha o contexto. Combinado com `@PreDestroy`/`SmartLifecycle`, executores e pools têm chance de terminar. Há uma sequência a respeitar no Kubernetes (§3.3).

## 3. Como funciona por dentro

### 3.1 Estrutura do fat jar e o JarLauncher

Um jar comum não carrega jars aninhados. O Boot resolve com um lançador:

```text
app.jar
├── META-INF/MANIFEST.MF        Main-Class: org.springframework.boot.loader.launch.JarLauncher
│                               Start-Class: com.example.orders.App
├── org/springframework/boot/loader/...   (classes do lançador, na raiz)
└── BOOT-INF/
    ├── classes/                suas classes e application.yml
    ├── lib/                    jars das dependências (nested jars)
    ├── classpath.idx           ordem do classpath
    └── layers.idx              mapa das camadas
```

`java -jar` carrega a `Main-Class` do manifesto (`JarLauncher`), que cria um `ClassLoader` capaz de ler os jars aninhados diretamente do arquivo (sem extrair para disco) e invoca a `Start-Class` por reflection ([10-anotacoes-e-reflection](../01-java/10-anotacoes-e-reflection.md)). Visto em [01 §3](01-o-que-e-spring-boot.md). Nos stack traces aparece `jar:nested:/.../s9-1.jar/!BOOT-INF/classes/!/com/example/orders/Api.class` (visto nos logs de erro desta validação).

Consequência: **não dá para adicionar um jar ao classpath de um fat jar com `-cp`** (o `-jar` ignora `-cp`). Para classpath externo use `-Dloader.path` com o `PropertiesLauncher` (`Main-Class` alternativo, `-Dloader.main`).

### 3.2 O modo "extract" e por que ele é melhor para contêiner

`-Djarmode=tools extract` descompacta o jar em diretórios, e com `--launcher` mantém o `JarLauncher` na execução (como no Dockerfile abaixo); sem ele (comportamento descrito na documentação, **não executado**) gera um jar fino com `Class-Path` no manifesto, sem o lançador, o que a documentação do Boot recomenda para CDS/AOT cache ⚠️ VERIFICAR. A imagem executada abaixo usa `--layers`, e o AOT cache medido em §3.4 foi gerado diretamente sobre o fat jar.

### 3.3 Kubernetes: onde cada peça se encaixa

```mermaid
sequenceDiagram
    participant K as Kubernetes
    participant P as Pod (Boot)
    K->>P: inicia contêiner
    K->>P: startupProbe / readinessProbe GET /actuator/health/readiness
    P-->>K: 503 enquanto sobe → 200 quando pronta
    K->>K: adiciona o pod ao Service
    Note over K,P: ... tráfego ...
    K->>P: remove do Service (endpoints) e envia SIGTERM
    P->>P: readiness → REFUSING_TRAFFIC; shutdown gracioso
    P-->>K: termina em ≤ timeout-per-shutdown-phase
    K->>P: SIGKILL se passar de terminationGracePeriodSeconds
```

Pontos de atenção:

- `terminationGracePeriodSeconds` do pod deve ser **maior** que `spring.lifecycle.timeout-per-shutdown-phase` (30 s), senão o Kubernetes mata o processo (`SIGKILL`) antes do término gracioso.
- A remoção do pod do `Service` e o envio do `SIGTERM` acontecem em paralelo, e pode haver conexões novas por alguns segundos depois do `SIGTERM`. Um `preStop` com `sleep` curto (ou o próprio estado `REFUSING_TRAFFIC` do Boot) reduz erros 502 no rolling update.
- Probes: usar `/actuator/health/liveness` e `/readiness` ([09 §2.3](09-actuator-e-observabilidade.md)); aplicações lentas para subir usam `startupProbe` para não serem reiniciadas durante o boot.
- **Memória:** a JVM respeita limites de contêiner (cgroups) em dimensionamento do heap (`-XX:MaxRAMPercentage`, ver [02-jvm-por-dentro](../01-java/02-jvm-por-dentro.md)). Heap + metaspace + threads + off-heap precisam caber no `limits.memory`, senão `OOMKilled`.
- **Configuração:** `ConfigMap`/`Secret` como variáveis de ambiente ou *config tree* (`spring.config.import=configtree:/etc/secrets/`; [03 §2.3](03-configuracao-externalizada.md)).

### 3.4 Otimização de startup

| Técnica | Como funciona | Custo | Estado |
|---|---|---|---|
| **AOT cache da JVM** (JEP 483, Java 24+; Project Leyden) | a JVM grava classes já carregadas/verificadas/ligadas num arquivo e o reutiliza | passo extra de "treino" no build; arquivo ~70 MB | executado abaixo |
| **CDS** (class-data sharing) | versão mais antiga: compartilha classes pré-processadas | idem; Boot documenta o fluxo com `-Djarmode=tools` | não executado |
| **Spring AOT** (`process-aot`) | gera código em vez de reflection em tempo de build | build mais complexo | [02-spring-core/10](../02-spring-core/10-aot-e-native.md) |
| **Native image** (GraalVM) | compila tudo antes da execução | build lento, reflection fechada, otimização de pico menor | não executado |
| **CRaC** (checkpoint/restore) | snapshot do processo em execução | depende de JDK com CRaC | ⚠️ VERIFICAR suporte atual |
| `spring.main.lazy-initialization=true` | cria beans só no primeiro uso | adia o custo e esconde erros de fiação até o runtime | evite como regra |

**Medido** em `s9` (Boot 4.1.2, Java 25, tempo de parede até o contexto atualizar e encerrar com `-Dspring.context.exit=onRefresh`, H2 em memória, 3 execuções cada):

| Configuração | Tempo |
|---|---|
| fat jar sem cache | 2,66 / 2,73 / 2,71 s |
| `-XX:AOTCache=app.aot` | 1,72 / 1,79 / 1,76 s |

Cerca de **35% menos** no startup, sem mexer no código. Passos usados:

```bash
# 1) treino: executa a aplicação até o contexto atualizar e grava o cache
java -XX:AOTCacheOutput=app.aot -Dspring.context.exit=onRefresh -jar app.jar
# 2) uso
java -XX:AOTCache=app.aot -jar app.jar
```

A criação do cache emitiu avisos `Preload Warning: Verification failed for ...reactive.TransactionalOperator` (classes de Reactor ausentes no classpath que o cache tentou pré-carregar); sem efeito na execução. O cache só vale para **o mesmo JDK e o mesmo jar/classpath**: gere-o no `Dockerfile` (no build da imagem), nunca o copie entre versões. ⚠️ VERIFICAR o fluxo recomendado na documentação do Boot 4.1 (que descreve o passo com `extract`).

## 4. Evolução histórica

| Versão | Mudança |
|---|---|
| Boot 1.0 (2014) | fat jar executável, `spring-boot-maven-plugin` |
| Boot 2.3 (2020) | camadas (`layers.idx`), Buildpacks (`build-image`), shutdown gracioso, probes |
| Boot 2.4 | Dockerfile em camadas oficial (`-Djarmode=layertools`) |
| Boot 3.0 (2022) | native image com GraalVM oficial; Java 17 mínimo |
| Boot 3.2 | virtual threads (`spring.threads.virtual.enabled`), suporte a CRaC |
| Boot 3.3 | CDS; `-Djarmode=tools` substitui `layertools` ⚠️ VERIFICAR a versão |
| Java 24+ / Boot 3.5–4.x | AOT cache da JVM (JEP 483) utilizável com o Boot; `JarLauncher` em `org.springframework.boot.loader.launch` (visto no MANIFEST em [01](01-o-que-e-spring-boot.md)) ⚠️ VERIFICAR desde qual versão do Boot e o fluxo recomendado |

⚠️ VERIFICAR as datas intermediárias (de memória). Nota: o modo `layertools` foi substituído por `tools`: o comando que **funciona** na versão validada é `java -Djarmode=tools -jar app.jar list-layers|extract|help` (a ajuda executada mostrou `extract`, `list-layers`, `help`).

## 5. Exemplos práticos

### 5.1 Build e execução simples

```bash
mvn -DskipTests package           # gera target/orders-1.jar (executado: 27 MB)
java -jar target/orders-1.jar --spring.profiles.active=prod
```

O jar de 27 MB vem do Tomcat, Jackson, Hibernate, Flyway, H2, Micrometer etc. Verifique o conteúdo: `jar tf app.jar | grep BOOT-INF/lib | sort`.

### 5.2 Dockerfile multi-estágio com camadas (executado)

```dockerfile
FROM eclipse-temurin:25-jre AS builder
WORKDIR /builder
COPY app.jar application.jar
RUN java -Djarmode=tools -jar application.jar extract --layers --destination extracted

FROM eclipse-temurin:25-jre
WORKDIR /application
COPY --from=builder /builder/extracted/dependencies/ ./
COPY --from=builder /builder/extracted/spring-boot-loader/ ./
COPY --from=builder /builder/extracted/snapshot-dependencies/ ./
COPY --from=builder /builder/extracted/application/ ./
RUN useradd -r app
USER app
ENTRYPOINT ["java", "-jar", "application.jar"]
```

Executado: `docker build` OK; `docker history` mostrou camadas de **27 MB** (dependencies), 4,1 kB (loader), 4,1 kB (snapshot-deps) e **20,5 kB** (application), ou seja, o que muda a cada commit pesa ~20 kB. O contêiner subiu; `GET /actuator/health` retornou `UP` com `db`, `diskSpace`, etc.; `GET /orders/count` retornou `1` (Flyway aplicou as migrações); `docker exec s9c id` retornou `uid=999(app)`, **não root**.

Observações do Dockerfile:
- Usa a imagem **JRE** (`eclipse-temurin:25-jre`, 480 MB no ambiente), menor que o JDK; para menores ainda, `jlink` ([14-modulos-e-build](../01-java/14-modulos-e-build.md)) ou imagens distroless.
- Ordem das `COPY` do mais estável ao mais volátil, para o cache de camadas funcionar.
- `USER app` (não root): se o processo for comprometido, o atacante não é root no contêiner.
- Opções da JVM por `JAVA_TOOL_OPTIONS` (ex.: `-XX:MaxRAMPercentage=75`), sem alterar o `ENTRYPOINT`.
- `ENTRYPOINT` na forma *exec* (array JSON): o processo Java vira PID 1 e recebe `SIGTERM`; a forma *shell* (`ENTRYPOINT java -jar ...`) coloca um `sh` na frente que **não repassa sinais**, e o encerramento gracioso nunca ocorre.

### 5.3 Buildpacks (não executado)

```bash
mvn spring-boot:build-image -Dspring-boot.build-image.imageName=orders:1
```

Sem Dockerfile: o Paketo detecta o jar, escolhe o JRE, configura memória em tempo de execução e organiza as camadas. Requer Docker local. Ganho: padronização e atualização de segurança da base pela plataforma; perda: menos controle fino. Não foi executado porque baixa várias imagens grandes.

### 5.4 Manifestos Kubernetes (trecho, não executado)

```yaml
spec:
  terminationGracePeriodSeconds: 45          # > timeout-per-shutdown-phase (30s)
  containers:
    - name: orders
      image: registry.example.com/orders:1
      env:
        - { name: JAVA_TOOL_OPTIONS, value: "-XX:MaxRAMPercentage=75" }
        - { name: SPRING_PROFILES_ACTIVE, value: prod }
      resources:
        requests: { cpu: "500m", memory: 768Mi }
        limits:   { memory: 768Mi }
      readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
      livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 } }
      startupProbe:
        httpGet: { path: /actuator/health/liveness, port: 8080 }
        failureThreshold: 30
        periodSeconds: 2
```

<!-- não aplicado em cluster -->

Um detalhe: **CPU limit baixo faz o startup demorar** (a JIT e o classpath scanning são intensivos); muitas equipes definem `requests` de CPU sem `limits`, ou aceitam um startup mais lento com `startupProbe` generoso.

### 5.5 Variáveis de ambiente e ordem de precedência

Em contêiner, a configuração vem de fora ([03 §2.1](03-configuracao-externalizada.md)): variáveis de ambiente `SPRING_DATASOURCE_URL` (relaxed binding) sobrepõem o `application.yml` empacotado. **Nunca coloque segredo no `application.yml` do jar nem na imagem**; imagens são distribuídas e analisadas.

### 5.6 Registrando a versão no `/actuator/info`

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution><goals><goal>build-info</goal><goal>repackage</goal></goals></execution>
  </executions>
</plugin>
```

Gera `META-INF/build-info.properties` (nome, versão, hora do build) e o `/actuator/info` passa a mostrá-los ([09](09-actuator-e-observabilidade.md)). Essencial para saber **qual versão** está rodando em cada ambiente.

## 6. Configurações e anotações relevantes

| Item | Para que serve | Exemplo |
|---|---|---|
| `spring-boot-maven-plugin:repackage` | gera o fat jar | automático com o parent |
| `spring-boot:run` / `spring-boot:build-image` | rodar / gerar imagem | |
| `spring-boot:build-info`, `git-commit-id` | metadados de versão | `/info` |
| `java -Djarmode=tools -jar app.jar extract` | extrai camadas / jar fino | `--layers --launcher --destination dir` |
| `list-layers` | lista as camadas | |
| `server.shutdown` | `graceful` (padrão confirmado) ou `immediate` | |
| `spring.lifecycle.timeout-per-shutdown-phase` | espera do encerramento (padrão `30s`) | `45s` |
| `server.port` | porta HTTP | `8080` |
| `management.server.port` | porta do Actuator | `9090` |
| `-XX:MaxRAMPercentage`, `-XX:+ExitOnOutOfMemoryError` | JVM em contêiner | em `JAVA_TOOL_OPTIONS` |
| `-XX:AOTCacheOutput` / `-XX:AOTCache` | cache AOT da JVM (Java 24+) | §3.4 |
| `-Dloader.path` | classpath externo (`PropertiesLauncher`) | |
| `spring.main.lazy-initialization` | beans sob demanda | evite em produção |
| `spring.threads.virtual.enabled` (padrão `false`, confirmado) | virtual threads | `true` |
| `SpringBootServletInitializer` | WAR em servidor externo | legado |

## 7. Quando usar e quando NÃO usar

**Jar executável / imagem em camadas:** padrão para quase tudo.
**Buildpacks:** quando se quer convenção e atualizações da base geridas pela plataforma; ruim quando se precisa de otimizações específicas no Dockerfile.
**AOT cache/CDS:** quando o startup importa (escala automática, funções, `CronJob`) e o passo de treino é aceitável.
**Native image:** quando o startup em milissegundos e o baixo consumo de memória compensam um build longo, menor throughput de pico e restrições de reflection; **não use** com bibliotecas pesadas em reflection sem teste, ou quando precisa de ferramentas de diagnóstico da JVM (JFR é limitado).
**WAR:** só se a organização impõe um servidor de aplicações compartilhado.
**Não** baixe o JDK completo e Maven na imagem final; use multi-estágio.
**Não** confie em `latest` para imagens base: fixe a versão e o digest, e reconstrua periodicamente (CVEs).

## 8. Erros comuns e boas práticas

1. **`ENTRYPOINT` em forma shell**: `SIGTERM` não chega ao Java; sem encerramento gracioso.
2. **Rodar como root** no contêiner: use `USER` não-root (executado `uid=999`).
3. **Imagem com tudo numa camada**: reenvio de dezenas de MB por commit.
4. **`terminationGracePeriodSeconds` menor que o timeout do Spring**: processo morto no meio.
5. **Limite de memória do contêiner sem ajustar a JVM**: `OOMKilled` sem stack trace Java. Use `MaxRAMPercentage` e deixe folga para metaspace/threads.
6. **Segredos na imagem ou no `application.yml`.**
7. **Cache AOT/CDS copiado entre JDKs ou entre builds**: ignorado ou rejeitado; gere no mesmo ambiente do runtime.
8. **Migração longa no startup** que ultrapassa o `startupProbe`: pod reiniciado no meio ([12 §7](12-migracoes-de-banco.md)).
9. **`lazy-initialization` ligado para "acelerar"**: erro de configuração só aparece na primeira requisição.
10. **Esquecer a versão no `/info`**: impossível saber o que está no ar.
11. **Exigir `-cp` com `-jar`**: ignorado; use `PropertiesLauncher`.
12. **Treinar o AOT cache com perfil/configuração diferente da produção**: o benefício cai (as classes carregadas diferem).

## 9. Perguntas de revisão

1. Por que o `ENTRYPOINT` deve usar a forma exec no Docker?
<details><summary>Resposta</summary>Na forma shell o PID 1 é um `sh` que não repassa `SIGTERM`; o Java não recebe o sinal e o encerramento gracioso não acontece. Na forma exec o Java é o PID 1.</details>

2. Qual o benefício das camadas na imagem, com números do exemplo?
<details><summary>Resposta</summary>As dependências (27 MB) mudam raramente e ficam em camada cacheada; o código (≈20 kB) é a única camada que muda por commit, reduzindo o tráfego e o tempo de deploy.</details>

3. Qual a relação entre `server.shutdown`, `timeout-per-shutdown-phase` e `terminationGracePeriodSeconds`?
<details><summary>Resposta</summary>`graceful` (padrão) espera requisições em andamento até 30 s; o Kubernetes precisa conceder mais tempo que isso, senão envia `SIGKILL` antes do término.</details>

4. Por que o liveness probe não deve ser o endpoint `/health` completo?
<details><summary>Resposta</summary>O `/health` agrega dependências externas; se o banco cair, o Kubernetes reiniciaria todas as instâncias em cascata. Use `/health/liveness` e `/health/readiness` ([09](09-actuator-e-observabilidade.md)).</details>

5. Como o AOT cache da JVM reduz o startup, e quais são suas restrições?
<details><summary>Resposta</summary>Guarda classes já carregadas, verificadas e ligadas em um arquivo reaproveitado na próxima execução (≈35% menos tempo medido). Vale só para o mesmo JDK e o mesmo classpath; exige um passo de treino.</details>

## 10. Referências oficiais

- Spring Boot Reference, Packaging / Efficient Deployments / Container Images: <https://docs.spring.io/spring-boot/reference/packaging/index.html> ⚠️ VERIFICAR o caminho exato
- Spring Boot Maven Plugin: <https://docs.spring.io/spring-boot/maven-plugin/index.html> ⚠️ VERIFICAR
- JEP 483: Ahead-of-Time Class Loading & Linking: <https://openjdk.org/jeps/483>
- Paketo Buildpacks: <https://paketo.io>
- Kubernetes: <https://kubernetes.io/docs/>
- The Twelve-Factor App: <https://12factor.net>
- Anterior: [12-migracoes-de-banco](12-migracoes-de-banco.md) · Próximo: [04-spring-cloud/](../04-spring-cloud/) (Sessão 10)
