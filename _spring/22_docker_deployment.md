---
layout: page
title: "Spring + Docker/K8s, деплой приложения"
permalink: /spring/observability
---

# 21. Spring Boot + Docker/K8s: деплой приложения

# 0. Введение

## Что это

Контейнеризация — подход к упаковке приложения и его зависимостей в стандартизованный образ, который одинаково запускается на ноутбуке разработчика, в CI и в продакшне. Для Java-микросервисов это означает: один и тот же Spring Boot JAR или native-бинарь вместе с JVM/рутФС в образе Docker. Kubernetes (K8s) — система оркестрации контейнеров: она разворачивает, скейлит и лечит ваши Pod’ы, следит за здоровьем, подменяет конфиги и секреты. В современном Java-проекте эти две технологии образуют основу поставки: «собери образ → задеплой чартом/манифестами».

Контейнеры решают проблему «работает у меня» за счёт изоляции среды выполнения. В образ уже входит нужная версия JDK, нужные СА-сертификаты, timezone, системные библиотеки — вы перестаёте зависеть от того, как настроен сервер. Это критично для повторяемости и быстрого восстановления после инцидентов: вы всегда можете поднять ту же версию образа.

С Kubernetes мы получаем автоматическое перезапускание процессов при падении, скейлинг по метрикам, выкладки без простоя и «самоисцеление». Это особенно важно для сервисов с непредсказуемой нагрузкой (например, событийные системы на Kafka): платформа держит нужное число реплик и умеет аккуратно останавливать инстансы, дренируя трафик.

В Java-мире контейнеризация сочетается с инструментами сборки артефактов: Dockerfile (multi-stage), Cloud Native Buildpacks (Gradle `bootBuildImage`) и Jib. Каждый из них формирует OCI-совместимый образ, но с разными компромиссами по скорости, требованиям к окружению и прозрачности слоёв.

Идея «иммутабельного артефакта» («immutable artifact») означает, что образ после сборки не меняется: вы не «правите» файлы на сервере, а выпускаете новый образ и выкатываете его. В связке с GitOps/Helm это даёт надёжную трассируемость — можно всегда сказать, какой коммит/чарт/образ сейчас в проде.

**Код: минимальное Spring Boot приложение**
Java:

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
public class DemoApplication {
  public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
  }
}

@RestController
@RequestMapping("/api")
class HelloController {
  @GetMapping("/hello")
  public String hello() {
    return "Hello, containers!";
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class DemoApplication

fun main(args: Array<String>) {
    runApplication<DemoApplication>(*args)
}

@RestController
@RequestMapping("/api")
class HelloController {
    @GetMapping("/hello")
    fun hello(): String = "Hello, containers!"
}
```

Gradle Groovy:

```groovy
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.3.5'
  id 'io.spring.dependency-management' version '1.1.6'
}

java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

repositories { mavenCentral() }

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

Gradle Kotlin:

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.5"
    id("io.spring.dependency-management") version "1.1.6"
}

java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

repositories { mavenCentral() }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## Зачем это

Главная причина — **повторяемость**. Один и тот же образ гарантирует одинаковую среду: шрифты/локаль, OpenJDK, CA-bundle. Ошибки «на проде другая JDK» исчезают. Контейнер упаковывает «всё необходимое», а инфраструктура занимается только запуском.

Вторая причина — **быстрое масштабирование**. Вместо медленной настройки VM вы стартуете ещё N Pod’ов. Kubernetes сам распределит их по нодам, учтёт лимиты CPU/памяти и запускает readiness-пробы, чтобы не слать трафик в холодные инстансы. Для событийной нагрузки можно динамически менять число потребителей.

Третья — **изоляция и безопасность**. Непривилегированный пользователь внутри контейнера, ограниченные capabilities, readOnly RootFS — это минимизирует ущерб даже при уязвимости в приложении. Плюс легко применять подписи образов и политику «только доверенные реестры».

Четвёртая — **скорость поставки**. CI конвейер «build → test → image → scan → sign → push → deploy» становится стандартом. Появляется «single source of truth» — тэг образа. Helm/ArgoCD/GitOps накатывают один артефакт на все окружения.

Пятая — **простая операционка**. Логи — в stdout/stderr; метрики — по HTTP/Prometheus; трассировка — через Otel-агенты. Никаких rsyslog/cron на инстансах. Всё это удобно агрегировать централизованными инструментами.

**Код: пример контроллера и actuator-пинга**
Java:

```java
package com.example.demo;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@Component
class AppHealth implements HealthIndicator {
  @Override public Health health() {
    return Health.up().withDetail("status", "ok").build();
  }
}

@RestController
class PingController {
  @GetMapping("/ping")
  public String ping() { return "pong"; }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.stereotype.Component
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@Component
class AppHealth : HealthIndicator {
    override fun health(): Health = Health.up().withDetail("status", "ok").build()
}

@RestController
class PingController {
    @GetMapping("/ping")
    fun ping(): String = "pong"
}
```

`application.yml`:

```yaml
server:
  port: 8080
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

---

## Где используется

Контейнеры применяются на всех стадиях: локальная разработка (Docker Compose для Postgres/Kafka), интеграционное тестирование (Testcontainers), стенды dev/test/stage (Helm-релизы), продакшн (K8s кластеры on-prem/в облаке). Один и тот же образ катится по всем средам.

В банкинге/финтехе контейнеризация стандартизует требования безопасности: non-root, скан уязвимостей, подписи (Cosign/Sigstore), политика обновления базовых образов. Это реально упрощает прохождение аудитов и ускоряет внедрение патчей.

В микросервисных архитектурах каждый сервис — отдельный образ, с собственным lifecycle. Это позволяет выкатывать фичи независимо, масштабировать «тяжёлые» компоненты отдельно и гибко обновлять только нужные части системы.

В монолитах контейнеры тоже полезны: «упаковали один большой сервис» и получили такие же преимущества: воспроизводимость, простое масштабирование, проверяемые health-пробы, единый способ логирования/метрик.

Data-площадки и ETL: контейнеры удобно используют для батч-джобов и init-контейнеров (миграции Liquibase/Flyway), где важны изоляция зависимостей (psql/jdbc-драйверы) и управляемые ресурсы.

**Код: конфиг подключения к Postgres (чтобы сразу проверить на локальном Compose)**
Java:

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.sql.DataSource;
import com.zaxxer.hikari.HikariDataSource;

@Configuration
public class DbConfig {
  @Bean
  public DataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl(System.getenv().getOrDefault("DB_URL", "jdbc:postgresql://localhost:5432/app"));
    ds.setUsername(System.getenv().getOrDefault("DB_USER", "app"));
    ds.setPassword(System.getenv().getOrDefault("DB_PASSWORD", "app"));
    return ds;
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import com.zaxxer.hikari.HikariDataSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class DbConfig {
    @Bean
    fun dataSource(): DataSource =
        HikariDataSource().apply {
            jdbcUrl = System.getenv("DB_URL") ?: "jdbc:postgresql://localhost:5432/app"
            username = System.getenv("DB_USER") ?: "app"
            password = System.getenv("DB_PASSWORD") ?: "app"
        }
}
```

`docker-compose.yml` (фрагмент):

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports: ["5432:5432"]
```

---

## Какие виды бывают

Варианты сборки образов: ручной Dockerfile (полный контроль), Buildpacks (Gradle `bootBuildImage`, без Dockerfile), Jib (сборка образа без Docker-демона). Варианты базовых образов: «толстые» дистрибутивные (Ubuntu/Debian с shell/менеджером пакетов), облегчённые Alpine (musl) и **distroless** (только рантайм-файлы, без shell). Варианты артефактов: «fat jar», **layered jar**, native-бинарь.

Варианты деплоя: одиночный хост/Compose, Kubernetes (Helm/классические манифесты), serverless-контейнеры (Cloud Run/Azure Container Apps). В корпоративной среде почти всегда K8s + Helm.

Есть варианты упаковки конфигов/секретов: env-переменные, монтируемые файлы (ConfigMap/Secret), внешние vault’ы (HashiCorp Vault, AWS Secrets Manager). Подход зависит от политики безопасности и требований к ротации.

Набор health-проб: liveness/readiness/startup — это не один «/health», а разноцелевые проверки. И, наконец, варианты выкладок: `RollingUpdate`, blue-green, canary.

**Код: Gradle Buildpacks и Jib**
Groovy:

```groovy
tasks.named('bootBuildImage') {
  imageName = "registry.example.com/demo:${version}"
  environment = ["BP_JVM_VERSION":"17"]
}

plugins {
  id 'com.google.cloud.tools.jib' version '3.4.3'
}
jib {
  to { image = "registry.example.com/demo:${version}" }
  container {
    jvmFlags = ['-XX:MaxRAMPercentage=75.0','-XX:+UseG1GC']
    ports = ['8080']
  }
}
```

Kotlin:

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootBuildImage>("bootBuildImage") {
    imageName.set("registry.example.com/demo:${project.version}")
    environment.set(mapOf("BP_JVM_VERSION" to "17"))
}

plugins { id("com.google.cloud.tools.jib") version "3.4.3" }
jib {
    to { image = "registry.example.com/demo:${project.version}" }
    container {
        jvmFlags = listOf("-XX:MaxRAMPercentage=75.0", "-XX:+UseG1GC")
        ports = listOf("8080")
    }
}
```

---

## Какие задачи и проблемы решает

Контейнеры и K8s закрывают боль совместимости (разные JDK/сертификаты/локали), обеспечивают управление ресурсами (CPU/память/FD-лимиты), дают стандартизованную наблюдаемость (логи/метрики/трейсинг) и безопасный жизненный цикл (graceful shutdown, пробы, перезапуски).

Они помогают выстраивать **12-факторную конфигурацию**: неизменяемые образы и параметры «снаружи». Это дисциплина — но именно она позволяет раскатывать десятки сервисов без дрейфа окружений.

С безопасностью тоже проще: централизованный скан образов, политика базовых образов и регулярные патчи, отказ от root, ограничение capabilities, read-only root FS. Добавьте подписи артефактов и политику admission-контроллеров — и вы серьёзно повышаете надёжность контура.

Отдельная задача — **инициализация**: миграции БД, прогрев кэшей/соединений. Контейнеры дают init-контейнеры/Job’ы и чёткую последовательность запуска.

И, наконец, **быстрые откаты**. Если релиз не удался, вы просто переключаете тэг образа/версии чарта, не откручивая вручную изменения на серверах.

**Код: настройка health-проб и метрик**
Java:

```java
package com.example.demo;

import io.micrometer.core.instrument.MeterRegistry;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
class MetricsConfig {
  private final MeterRegistry registry;
  MetricsConfig(MeterRegistry registry) { this.registry = registry; }

  @PostConstruct
  void init() {
    registry.counter("app.start.count").increment();
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import io.micrometer.core.instrument.MeterRegistry
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration

@Configuration
class MetricsConfig(private val registry: MeterRegistry) {
    @PostConstruct
    fun init() { registry.counter("app.start.count").increment() }
}
```

`application.yml`:

```yaml
management:
  endpoints.web.exposure.include: health,info,prometheus
  metrics.tags:
    application: demo
```

---

# 1. Образ приложения: стратегии сборки

## Подходы: Dockerfile multi-stage, Buildpacks (bootBuildImage), Jib. Когда что выбирать по скорости/кешированию/доступу к Docker.

**Dockerfile multi-stage** даёт полный контроль: вы явно определяете слои, базовые образы, копируете только нужные артефакты. Это лучший выбор, если нужны тонкие оптимизации (distroless с кастомным truststore, tini, дополнительные сертификаты). Недостаток — чуть больше поддержки (писать/обновлять Dockerfile), потенциально дольше локально, если нет хорошего кеша.

**Buildpacks (`bootBuildImage`)** удобно тем, что «нет Dockerfile». Плагины анализируют проект, автоматически формируют слои, выбирают подходящую JVM, подмешивают агенты. Работает быстро в CI с кэшированием слоёв билдпаков. Минусы — меньшая прозрачность и зависимость от стека buildpack’ов; тонкие оптимизации сложнее.

**Jib** собирает образ без Docker-демона, напрямую в реестр. Отличный выбор для CI, где нет Docker socket. Jib умеет умно раскладывать слои (зависимости отдельно от классов), что ускоряет инкрементальные релизы. Минус — тоже ограниченная возможность «тонкой ручной сборки» рантайм-образа.

Скорость и кеширование: Jib и Buildpacks часто быстрее при частых небольших изменениях кода; Dockerfile быстрее там, где слои подстроены вручную (например, разделение `build.gradle` и `settings.gradle`, pre-cache зависимостей). Требования: Dockerfile/Buildpacks обычно требуют Docker-демона на машине, Jib — нет.

Рекомендация: начинайте с Buildpacks или Jib (минимум инфраструктуры). Переходите на Dockerfile, когда появляется потребность в distroless, tini, ручном HEALTHCHECK или специфичном управлении truststore/локалью.

**Код и конфиг:**
Java/Kotlin — сам сервис (см. «Введение»). Ниже — сборка.

Dockerfile (multi-stage):

```dockerfile
# syntax=docker/dockerfile:1.7
FROM eclipse-temurin:17-jdk AS build
WORKDIR /src
COPY gradlew gradlew
COPY gradle gradle
COPY build.gradle settings.gradle ./
COPY src src
RUN ./gradlew --no-daemon clean bootJar

FROM gcr.io/distroless/java17-debian12:latest
WORKDIR /app
COPY --from=build /src/build/libs/*-SNAPSHOT.jar /app/app.jar
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","-XX:+UseG1GC","-jar","/app/app.jar"]
```

Gradle Groovy (Buildpacks + Jib):

```groovy
tasks.named('bootBuildImage') {
  imageName = "registry.local/demo:${version}"
  environment = ["BP_JVM_VERSION":"17.*"]
}

plugins { id 'com.google.cloud.tools.jib' version '3.4.3' }
jib {
  to { image = "registry.local/demo:${version}" }
}
```

Gradle Kotlin:

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootBuildImage>("bootBuildImage") {
    imageName.set("registry.local/demo:${project.version}")
    environment.set(mapOf("BP_JVM_VERSION" to "17.*"))
}
plugins { id("com.google.cloud.tools.jib") version "3.4.3" }
jib { to { image = "registry.local/demo:${project.version}" } }
```

---

## Базовые образы: distro-full (e.g. eclipse-temurin) vs distroless/alpine. Компромиссы между размером, совместимостью и дебагом.

**Full/distro-based** (Debian/Ubuntu, `eclipse-temurin`) — удобно для дебага: есть shell, можно добавить утилиты, TZ, локали. Образы тяжелее, поверхность атаки шире, но совместимы с большинством native-lib’ов (glibc). Хороший выбор для сборочного этапа (build stage) и иногда для рантайма, если нужен shell.

**Alpine (musl)** — очень лёгкий, но может ломать некоторые библиотеки, собранные под glibc. Java на Alpine работает, но стоит быть осторожнее с JNI/шрифтами/локалями. Плюс — малый размер, минус — иногда «неуловимые» баги.

**Distroless** — только JVM и нужные системные файлы, нет shell/пакетного менеджера. Минимальная поверхность атаки и размер. Минус — «не залезть внутрь»; дебаг делайте через логи/метрики/отладочные эндпойнты, а не через `bash`. Отличен для прод-рантайма.

Подход: сборка (build) — на full-образе, рантайм — distroless. Если нужен TLS с кастомным CA или timezone, подготовьте на этапе сборки и **копируйте** в рантайм.

**Код/конфиг:**
Dockerfile сравнения:

```dockerfile
# Full runtime (удобно, но тяжелее)
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY build/libs/app.jar /app/app.jar
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

```dockerfile
# Distroless runtime (рекомендуется для прод)
FROM gcr.io/distroless/java17-debian12
WORKDIR /app
COPY build/libs/app.jar /app/app.jar
USER nonroot:nonroot
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","-jar","/app/app.jar"]
```

Java/Kotlin — эндпойнт для проверки (тот же `/ping`).

Gradle Groovy/Kotlin — без изменений; важно лишь собирать `bootJar`.

---

## Слои: reproducible build, слой зависимостей vs слой приложения; как ускорять CI кэшами Gradle/Artifactory.

Слои образа должны меняться как можно реже. В Java-сервисах это достигается разделением **зависимостей** (слой, который редко меняется) и **классов/ресурсов приложения** (меняются чаще). Spring Boot поддерживает слойность JAR’а; Buildpacks/Jib умеют автоматически разносить слои.

**Reproducible build**: фиксируйте версии плагинов, используйте `gradle.lockfile`, кэшируйте Gradle-директории (`~/.gradle/caches`, `~/.m2` при Maven) в CI. В Artifactory/Nexus храните зависимости, чтобы не тянуть их из интернета каждый раз.

В Dockerfile можно тянуть только файлы, влияющие на зависимости, **до** копирования кода: сначала `build.gradle`/`settings.gradle` + `gradle` wrapper → команда, качающая зависимости, → потом `src/` и сборка. Так вы максимально используете кеш.

Buildpacks и Jib сами раскладывают слои, поэтому инкрементальные сборки часто очень быстры. Но если вам нужно «разложить» нестандартные артефакты (нативные либы, шрифты) — Dockerfile даёт полный контроль.

**Код/конфиг:**
Gradle Groovy:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  // фиксируем версии
  constraints { implementation('com.fasterxml.jackson.core:jackson-databind:2.17.2') }
}
```

Gradle Kotlin:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    constraints { implementation("com.fasterxml.jackson.core:jackson-databind:2.17.2") }
}
```

Dockerfile (оптимизированный кеш):

```dockerfile
FROM eclipse-temurin:17-jdk AS build
WORKDIR /src
COPY gradle gradle
COPY gradlew settings.gradle build.gradle ./
RUN ./gradlew --no-daemon dependencies || true
COPY src src
RUN ./gradlew --no-daemon bootJar
```

Java/Kotlin код — любой контроллер (как выше), т.к. логика здесь про сборку/слои.

---

## Мультиархитектура: buildx и теги для amd64/arm64; политика тегов (semver, commit-sha, build-metadata).

Поддержка **multi-arch** нужна, если разработчики на Apple Silicon (arm64), а прод — amd64, или вы выкатываете на разные кластеры. Docker Buildx умеет собирать манифест для нескольких архитектур в один тэг.

Политика тегов: используйте **semver** (`1.4.2`), **commit-sha** (`git-abcdef0`) и, при необходимости, **build-metadata** (дата/ветка). Хорошая практика — пушить `:1.4.2`, `:1.4`, `:1`, `:git-abcdef0` и **никогда** не перезаписывать исторические теги.

Храните информацию о версии в приложении (`/actuator/info`) через `buildInfo()`. Это помогает мониторингу и расследованию инцидентов: глядя на метки/логи, вы понимаете, какая версия где крутится.

В CI указывайте платформы `--platform=linux/amd64,linux/arm64`. Следите за базовыми образами: не все тэги бывают multi-arch. Distroless/Temurin — обычно поддерживают обе.

**Код/конфиг:**
Gradle Groovy:

```groovy
springBoot {
  buildInfo()
}
```

Gradle Kotlin:

```kotlin
tasks.named<org.springframework.boot.gradle.tasks.buildinfo.BuildInfo>("bootBuildInfo")
```

Java:

```java
package com.example.demo;

import org.springframework.boot.info.BuildProperties;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class VersionController {
  private final BuildProperties build;
  VersionController(BuildProperties build) { this.build = build; }

  @GetMapping("/version")
  public String version() { return build.getVersion() + " " + build.get("git.commit.id.abbrev"); }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.boot.info.BuildProperties
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class VersionController(private val build: BuildProperties) {
    @GetMapping("/version")
    fun version(): String = "${build.version} ${build["git.commit.id.abbrev"]}"
}
```

Пример buildx:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.local/demo:1.4.2 \
  -t registry.local/demo:1.4 \
  -t registry.local/demo:git-abcdef0 \
  --push .
```

---

# 2. Продовый Dockerfile (best practices)

## Multi-stage: сборка артефакта, затем минимальный runtime. COPY только нужного.

Multi-stage позволяет держать «тяжёлые» инструменты (JDK, Gradle, gcc) только в build-слое и получить минимальный рантайм (distroless/JRE). Это уменьшает размер и поверхность атаки. Копируйте **только** итоговый JAR/бинарь и необходимые файлы (truststore, локаль, ресурсы).

Следите за тем, чтобы случайно не скопировать `~/.m2`, `.git`, папки `test` и прочий «мусор». Используйте `.dockerignore`. В рантайм-слоях не нужны исходники.

Если требуется tini или дополнительные сертификаты — соберите/скачайте их в build-слое и перенесите в рантайм как отдельные файлы. Не ставьте `apt` в рантайм-образ.

**Код/конфиг:**
`.dockerignore`:

```
.git
.gradle
build
out
Dockerfile*
docker-compose*
```

Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM eclipse-temurin:17-jdk AS build
WORKDIR /src
COPY gradlew gradlew
COPY gradle gradle
COPY settings.gradle build.gradle ./
COPY src src
RUN ./gradlew --no-daemon clean bootJar

FROM gcr.io/distroless/java17-debian12
WORKDIR /app
COPY --from=build /src/build/libs/*-SNAPSHOT.jar /app/app.jar
USER nonroot:nonroot
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","-XX:+UseG1GC","-jar","/app/app.jar"]
```

Java/Kotlin — приложение как выше. Gradle — стандартный.

---

## Безопасность: USER non-root, readOnlyRootFilesystem, drop capabilities; отсутствие shell/пакетного менеджера в runtime.

Запускайте процесс **под непривилегированным пользователем**. Это снижает ущерб при компрометации. В Dockerfile — `USER nonroot:nonroot` (distroless), или создайте uid/gid вручную. На уровне оркестратора делайте rootfs read-only, а для нужных каталогов монтируйте `emptyDir`/`tmpfs`. Capabilities можно урезать в K8s `securityContext`.

Отсутствие shell — плюс distroless: атакующему сложнее «превратить» RCE в полноценный взлом. Пакетный менеджер в рантайме не нужен. Сертификаты/локаль/временные каталоги готовьте заранее.

Проверяйте образ на уязвимости (Trivy/Grype), подписывайте (Cosign), применяйте политику (Kyverno/OPA Gatekeeper), чтобы отклонять неподписанные/уязвимые образы.

**Код/конфиг:**
Dockerfile — создание рабочей директории для временных файлов:

```dockerfile
FROM gcr.io/distroless/java17-debian12
WORKDIR /app
USER nonroot:nonroot
# В distroless /tmp доступен; можно использовать его для временных файлов
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

Java:

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.nio.file.*;

@RestController
class TmpController {
  @GetMapping("/tmp")
  public String writeTmp() throws Exception {
    Path p = Files.createTempFile("demo-", ".txt");
    Files.writeString(p, "ok");
    return "wrote to " + p.toString();
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.nio.file.Files

@RestController
class TmpController {
    @GetMapping("/tmp")
    fun writeTmp(): String {
        val p = Files.createTempFile("demo-", ".txt")
        Files.writeString(p, "ok")
        return "wrote to $p"
    }
}
```

K8s (фрагмент, если будете применять Helm позже):

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

---

## JVM-настройки по умолчанию для контейнеров: MaxRAMPercentage, GC (G1/ZGC), утилизация CPU квот.

Современная JVM умеет «видеть» cgroup-лимиты, но всё равно стоит управлять памятью явно: `-XX:MaxRAMPercentage=75.0` оставляет запас под нативную память, стек, metaspace. Для большинства сервисов хорош G1 GC; ZGC — для больших heap’ов и паузы <10мс, но требует профилирования.

CPU-квоты: указывайте пул потоков и параллелизм с учётом `AvailableProcessors()`. В Boot 3.x Tomcat/Jetty учитывают это автоматически, но пулов в приложении может быть много (DB, Kafka, executors) — настраивайте. **Не** ставьте `Xms=Xmx` в контейнере, чтобы JVM могла адаптироваться.

Учитывайте off-heap (Netty, ByteBuffer, caches) — даже при «маленьком Xmx» можно легко словить OOM от нативной памяти, если rootfs read-only и нет свопа.

**Код/конфиг:**
Dockerfile (ENV):

```dockerfile
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
```

Java:

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class SysController {
  @GetMapping("/sys")
  public String sys() {
    int cpus = Runtime.getRuntime().availableProcessors();
    long max = Runtime.getRuntime().maxMemory() / (1024*1024);
    return "cpus=" + cpus + " maxMemMb=" + max;
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class SysController {
    @GetMapping("/sys")
    fun sys(): String {
        val cpus = Runtime.getRuntime().availableProcessors()
        val max = Runtime.getRuntime().maxMemory() / (1024*1024)
        return "cpus=$cpus maxMemMb=$max"
    }
}
```

---

## Стартовый скрипт/entrypoint: exec-формат, корректная обработка сигналов; tini как опция.

Команда контейнера должна использовать **exec-формат** JSON, а не `sh -c`: тогда сигналы (`SIGTERM`) приходят прямо вашему процессу PID 1. Если у вас сложный скрипт, используйте `tini` — минимальный init, который проксирует сигналы и ждёт зомби-процессы.

Spring Boot корректно завершает работу при `SIGTERM`: закрывает входящие соединения, ждёт завершения обработчиков. Но важно настроить таймауты и graceful-period на уровне оркестратора.

Если без tini, убедитесь, что вы не используете оболочку в CMD/ENTRYPOINT. Не перехватывайте сигналы «на глазок»; в Java можно добавить `@PreDestroy`/`DisposableBean` для логов.

**Код/конфиг:**
Dockerfile с tini:

```dockerfile
FROM alpine:3.20 AS tini
RUN apk add --no-cache tini

FROM gcr.io/distroless/java17-debian12
COPY --from=tini /sbin/tini /tini
WORKDIR /app
COPY app.jar /app/app.jar
USER nonroot:nonroot
ENTRYPOINT ["/tini","--","java","-jar","/app/app.jar"]
```

Java:

```java
package com.example.demo;

import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class ShutdownHook {
  @PreDestroy
  public void onStop() {
    System.out.println("Graceful shutdown...");
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import jakarta.annotation.PreDestroy
import org.springframework.stereotype.Component

@Component
class ShutdownHook {
    @PreDestroy
    fun onStop() {
        println("Graceful shutdown...")
    }
}
```

---

## HEALTHCHECK: ping actuator/ready; интервалы/таймауты без ложных тревог.

HEALTHCHECK в Docker полезен для одиночного хоста/Compose; в K8s лучше полагаться на readiness/liveness пробы. Если оставляете Docker HEALTHCHECK — не стучитесь в «тяжёлые» эндпойнты, используйте лёгкий `/actuator/health/readiness`.

Не делайте слишком частые проверки: иначе получите ложные срабатывания под нагрузкой и «пинги» будут мешать нормальному трафику. Балансируйте `interval`, `timeout`, `retries`, `start-period`.

В Spring Boot включите **probes**: они учитывают состояние зависимостей (DB/Kafka). Readiness может быть «down», пока пулы не готовы; это нормальное поведение при старте.

**Код/конфиг:**
Dockerfile:

```dockerfile
HEALTHCHECK --interval=30s --timeout=2s --retries=3 --start-period=20s \
  CMD [ "wget", "--spider", "-q", "http://localhost:8080/actuator/health/readiness" ]
```

`application.yml`:

```yaml
management:
  endpoint.health.probes.enabled: true
  health:
    livenessstate.enabled: true
    readinessstate.enabled: true
```

Java/Kotlin — можно добавить кастомный `HealthIndicator`, как в «Введение».

---

## Часовой пояс/локаль/SSL-сертификаты: копирование truststore/ca-bundle, контроль TZ через env.

Часовой пояс лучше задавать **через переменную окружения** (`TZ=Europe/Moscow`) — не меняйте системные файлы. Локаль и шрифты для рендеринга PDF/Excel соберите на build-слое и перенесите.

SSL: если нужно доверять кастомному CA, создайте **отдельный truststore** и передавайте путь/пароль через конфигурацию (`javax.net.ssl.trustStore`). В distroless нет `update-ca-certificates`, поэтому truststore формируйте на сборке.

Проверяйте, что JDBC/HTTP-клиенты используют нужный truststore. Для MTLS добавляйте `keystore` и настраивайте клиента/сервер.

**Код/конфиг:**
Создание truststore (скрипт этапа сборки):

```bash
keytool -importcert -noprompt \
  -alias corp-ca -file corp-ca.crt \
  -keystore truststore.jks -storepass changeit
```

Dockerfile (копируем truststore):

```dockerfile
FROM gcr.io/distroless/java17-debian12
WORKDIR /app
COPY truststore.jks /app/truststore.jks
ENV TZ=Europe/Moscow
ENV JAVA_TOOL_OPTIONS="-Djavax.net.ssl.trustStore=/app/truststore.jks -Djavax.net.ssl.trustStorePassword=changeit"
COPY app.jar /app/app.jar
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

Java:

```java
package com.example.demo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.time.ZoneId;
import java.time.ZonedDateTime;

@RestController
class TimeController {
  @GetMapping("/time")
  public String now() {
    return ZonedDateTime.now(ZoneId.systemDefault()).toString();
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.time.ZoneId
import java.time.ZonedDateTime

@RestController
class TimeController {
    @GetMapping("/time")
    fun now(): String = ZonedDateTime.now(ZoneId.systemDefault()).toString()
}
```

---

# 3. Конфигурация, секреты и профили

## Externalized config: environment, system properties, mounted files; profiles (dev/test/prod).

По «12-фактору» конфиги хранятся **вне** образа. В Spring Boot порядок источников: `application.yml` → профили → env → системные свойства → аргументы. Профили (`spring.profiles.active`) позволяют включать разные блоки конфигурации для dev/test/prod.

Секреты лучше передавать **как файлы** (монтирование) или секрет-менеджером. Для параметров подключения — env-переменные удобнее. В Kubernetes `ConfigMap`/`Secret` монтируются в файлы или попадают в env.

При локальной разработке удобно иметь `application-local.yml` и активировать профиль `local`. В проде ни один профиль, кроме `prod`, не должен попадать в контейнер через bake-вставки.

Проверяйте значения на старте: если обязательный параметр не задан — падать fail-fast, а не работать «как-нибудь».

**Код/конфиг:**
`application.yml`:

```yaml
spring:
  application.name: demo
  profiles:
    group:
      dev: [ "common" ]
      prod: [ "common" ]
---
spring:
  config.activate.on-profile: common
server:
  port: 8080
---
spring:
  config.activate.on-profile: prod
app:
  external-url: https://prod.example.com
```

Java:

```java
package com.example.demo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
class ConfigController {
  @Value("${app.external-url:unset}")
  private String externalUrl;

  @GetMapping("/cfg")
  public String cfg() { return externalUrl; }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.beans.factory.annotation.Value
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class ConfigController(
    @Value("\${app.external-url:unset}") private val externalUrl: String
) {
    @GetMapping("/cfg")
    fun cfg(): String = externalUrl
}
```

Gradle Groovy/Kotlin — добавить `spring-boot-configuration-processor` для подсказок (см. следующий пункт).

---

## Secrets: файловые и env-секреты; шифрование конфигурации, ключи/сертификаты, ротация.

Секреты в K8s — base64-коды, монтируются в файлы (`/var/run/secrets/...`) или env. Для TLS и MTLS чаще используют файловые секреты (keystore/truststore). В Docker Compose есть `secrets`, но в простых сценариях чаще — env.

Шифрование конфигов (например, Spring Cloud Config + Vault) позволяет  хранить только зашифрованные значения, а ключ — в отдельном vault’е. Ротация ключей критична: настраивайте TTL и автоматическое обновление.

Приложение должно **не логировать** секреты. Маскирование в логгере/actuator — обязательная практика. Для программного чтения файлов-секретов удобно указывать путь в `application.yml` и загружать ключ в памяти (без записи на диск).

**Код/конфиг:**
K8s Secret (фрагмент):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-tls
type: Opaque
data:
  keystore.p12: <base64>
```

`application.yml`:

```yaml
server:
  ssl:
    enabled: true
    key-store: /etc/ssl/keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-store-type: PKCS12
```

Java:

```java
package com.example.demo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
class SecretConsumer {
  SecretConsumer(@Value("${server.ssl.key-store}") String path) {
    System.out.println("Keystore path = " + path);
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component

@Component
class SecretConsumer(
    @Value("\${server.ssl.key-store}") path: String
) {
    init { println("Keystore path = $path") }
}
```

Docker Compose (фрагмент):

```yaml
services:
  app:
    environment:
      SSL_PASSWORD: "${SSL_PASSWORD}"
    volumes:
      - ./secrets/keystore.p12:/etc/ssl/keystore.p12:ro
```

---

## 12-фактор: неизменяемый образ, параметры — извне; отдельные билды на «dev-фича» не допускаются.

Смысл 12-фактора — один и тот же образ катится на все окружения, а **параметры** меняются снаружи. Не делайте разные сборки «dev/prod» с разными JAR’ами — это увеличивает риск дрейфа и ловли «окруженческих» багов.

Версионируйте только код/инфраструктуру, а не конфиги. Для dev используйте `docker-compose.override.yml` или профили, но образ — тот же. В K8s — разные values для Helm, один chart, один образ.

Feature-flags выносятся в конфигурацию или сервис фичей. Нельзя bake’ить «на лету» флаги в контейнер и выкатывать «специальный образ» для тестов.

**Код/конфиг:**
Java:

```java
package com.example.demo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
class FeatureFlagController {
  @Value("${feature.newFlow:false}")
  private boolean newFlow;

  @GetMapping("/feature")
  public String flag() {
    return newFlow ? "new flow" : "old flow";
  }
}
```

Kotlin:

```kotlin
package com.example.demo

import org.springframework.beans.factory.annotation.Value
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class FeatureFlagController(
    @Value("\${feature.newFlow:false}") private val newFlow: Boolean
) {
    @GetMapping("/feature")
    fun flag(): String = if (newFlow) "new flow" else "old flow"
}
```

Helm values (dev/prod различаются только параметрами):

```yaml
env:
  - name: FEATURE_NEWFLOW
    value: "true"
```

`application.yml`:

```yaml
feature:
  newFlow: ${FEATURE_NEWFLOW:false}
```

---

## Проверка конфигов на старте: fail-fast, валидация значений, безопасные дефолты.

Лучше упасть на старте, чем работать с неожиданными дефолтами. В Spring Boot используйте `@ConfigurationProperties` + `@Validated` для строгой схемы конфигурации. Опциональные параметры — с безопасными дефолтами; обязательные — без дефолтов, со strict-валидацией.

Проверяйте порты/URL/таймауты на валидность, минимумы/максимумы, что секреты не пустые. В логи выводите **факты** (включена ли фича), но не значения секретов. В CI можно прогонять «dry-run» с реальными values.

Если конфиг критичен (например, обязательная зависимость), в `SmartLifecycle`/`ApplicationRunner` можно выполнить явную проверку доступности (с таймаутом) и упасть, чтобы не «зависнуть» в `CrashLoopBackOff`.

**Код/конфиг:**
Gradle Groovy:

```groovy
dependencies {
  annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

Gradle Kotlin:

```kotlin
dependencies {
    annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```

Java:

```java
package com.example.demo;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties("app")
@Validated
class AppProps {
  @NotBlank
  private String externalUrl;

  @Min(100) @Max(10000)
  private int httpTimeoutMs = 1000;

  public String getExternalUrl() { return externalUrl; }
  public void setExternalUrl(String v) { this.externalUrl = v; }
  public int getHttpTimeoutMs() { return httpTimeoutMs; }
  public void setHttpTimeoutMs(int v) { this.httpTimeoutMs = v; }
}
```

Kotlin:

```kotlin
package com.example.demo

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated

@Component
@ConfigurationProperties("app")
@Validated
class AppProps {
    @field:NotBlank
    var externalUrl: String = ""

    @field:Min(100) @field:Max(10000)
    var httpTimeoutMs: Int = 1000
}
```

`application.yml` (пример корректного старта):

```yaml
app:
  external-url: https://example.org
  http-timeout-ms: 500
```

# 4. Сеть, протоколы и прокси

## Порты и биндинг: server.port, host networking vs bridge; keep-alive/HTTP/2, gzip

В контейнерах приложение обычно слушает на `0.0.0.0`, иначе Kubernetes/Compose не смогут пробросить порт. В Spring Boot это поведение по умолчанию: биндинг на все интерфейсы. Для явности можно зафиксировать `server.address=0.0.0.0` и указать `server.port=8080`. На стороне Docker/K8s порты мапятся наружу через `ports:` (Compose) или `Service` (K8s). Для одиночного хоста есть соблазн использовать `--network host`, но это уменьшает изоляцию и мешает нескольким сервисам делить одни и те же порты; в 99% случаев **bridge/overlay** — правильный выбор.

HTTP keep-alive критичен для производительности: он экономит TCP-рукопожатия и TLS-шифрование. В Boot keep-alive включён; если вы используете реверс-прокси (Nginx/Ingress), следите, чтобы оно не закрывало соединения слишком агрессивно (таймауты должны быть согласованы с сервером приложений). Для высоконагруженных API полезен HTTP/2: мультиплексирование одно соединение/несколько запросов снижает накладные расходы, особенно за TLS.

Gzip/deflate/brotli — сжатие ответов экономит трафик и накладные расходы, но расходует CPU. Лучший компромисс — включать сжатие **на прокси у периметра** (Nginx/Ingress), а в приложении — только для отдельных случаев (например, когда ответ непроксифицируется). Если всё-таки включаете сжатие в Boot, ограничивайте минимальный размер тела и поддерживаемые MIME-типы, чтобы не сжимать мелочь и двоичные форматы (PNG/JPEG).

На уровне Tomcat/Jetty важно контролировать размеры буферов и заголовков (см. пункт об ограничениях). Иначе при нагрузке на длинные заголовки (JWT, cookies) можно попасть в неожиданные 400/431. Также учитывайте, что HTTP/2 в Tomcat требует включённого ALPN; на старых JDK/базовых образах нужны правильные пакеты/флаги.

Сетевые параметры контейнера и JVM пересекаются: если CPU/память задраны «впритык», keep-alive/HTTP/2 могут страдать от GC-пешек/контеншена. Наблюдайте pool’ы соединений, GC-паузы и таймауты — сеть «тянет» за собой тюнинг всей платформы. Для бэкпрешсера обязательно задавайте таймауты клиентам (HTTP/DB/Kafka), чтобы не зависать на зомби-коннектах.

В практике: фиксируем `server.port`, включаем HTTP/2, если у вас TLS на приложении (или h2c за доверенным прокси), и оставляем сжатие на прокси. Для разработчика это пара свойств; для SRE — согласованные таймауты и лимиты на всех слоях.

**Код (Java): минимальная проверка биндинга/HTTP-версии**

```java
package com.example.net;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
public class NetApp {
  public static void main(String[] args) { SpringApplication.run(NetApp.class, args); }
}

@RestController
class NetController {
  @GetMapping("/who")
  public String who(HttpServletRequest req) {
    return "proto=" + req.getProtocol() + " remote=" + req.getRemoteAddr() + " host=" + req.getServerName();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.net

import jakarta.servlet.http.HttpServletRequest
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class NetApp

fun main(args: Array<String>) = runApplication<NetApp>(*args)

@RestController
class NetController {
    @GetMapping("/who")
    fun who(req: HttpServletRequest) = "proto=${req.protocol} remote=${req.remoteAddr} host=${req.serverName}"
}
```

`application.yml` (фрагмент):

```yaml
server:
  address: 0.0.0.0
  port: 8080
  http2:
    enabled: true
# сжатие лучше на прокси; если нужно тут:
#  compression:
#    enabled: true
#    min-response-size: 2KB
#    mime-types: application/json,text/plain,text/html
```

---

## Реверс-прокси/ingress: X-Forwarded-*, доверенные прокси и корректные ссылки/редиректы

За прокси (Nginx/Ingress) приложение видит IP и схему прокси, а не клиента. Чтобы Spring корректно восстанавливал `scheme/host/port` и формировал правильные абсолютные ссылки/редиректы, включите поддержку заголовков `X-Forwarded-*`/`Forwarded`. В Boot 3+ достаточно установить стратегию обработки заголовков и, при необходимости, ограничить доверенные прокси.

Доверять нужно **только** собственным прокси/балансировщикам, иначе злоумышленник может подделать `X-Forwarded-Proto: https` и обмануть вашу генерацию ссылок. В Tomcat это делается через `RemoteIpValve` с шаблоном «внутренних» адресов. В мире Kubernetes доверенными обычно являются IP подсети Ingress-контроллера/Service LB.

Убедитесь, что прокси действительно проксирует нужные заголовки (`X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host`). В Nginx это `proxy_set_header ...`. В Ingress-аннотациях часто достаточно включить `use-forwarded-headers: "true"`. Проверить можно эндпойнтом, который печатает `request.getHeader(...)` — удобно для диагностики.

Неверная настройка ведёт к поломанным редиректам на `http://` вместо `https://`, неправильным absolute URL в ссылках и ошибкам OAuth2 callback’ов (redirect_uri). Если у вас проблемы именно с редиректами через `Spring Security`, всегда сначала проверьте forwarded-заголовки и доверенные прокси.

В микросервисах согласованная передача `X-Request-ID`/`Traceparent` облегчает трассировку запросов через несколько сервисов. Проследите, чтобы прокси не перезаписывал эти заголовки, если они уже пришли от внешнего клиента, но добавлял свои при их отсутствии.

Наконец, помните: заголовки могут быть длинными. Заложите запас в лимитах размера заголовков как на прокси, так и на сервере приложений (см. пункт об ограничениях).

**Код (Java): включаем обработку forwarded-заголовков и доверяем своим прокси**

```java
package com.example.proxy;

import org.apache.catalina.valves.RemoteIpValve;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.*;

import jakarta.servlet.http.HttpServletRequest;

@SpringBootApplication
public class ProxyApp {
  public static void main(String[] args) { SpringApplication.run(ProxyApp.class, args); }

  @Bean
  WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatForwardedCustomizer() {
    return factory -> factory.addEngineValves(remoteIpValve());
  }

  private RemoteIpValve remoteIpValve() {
    RemoteIpValve valve = new RemoteIpValve();
    valve.setProtocolHeader("X-Forwarded-Proto");
    valve.setRemoteIpHeader("X-Forwarded-For");
    // доверенные прокси: подсеть ingress-контроллера/балансировщика
    valve.setInternalProxies("10\\.0\\.\\d+\\.\\d+|192\\.168\\.\\d+\\.\\d+");
    return valve;
  }
}

@RestController
class UrlController {
  @GetMapping("/url")
  public String url(HttpServletRequest r) {
    String scheme = r.getScheme(); // учитывает X-Forwarded-Proto при RemoteIpValve
    String host = r.getHeader("Host");
    return scheme + "://" + host + r.getRequestURI();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.proxy

import jakarta.servlet.http.HttpServletRequest
import org.apache.catalina.valves.RemoteIpValve
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory
import org.springframework.boot.web.server.WebServerFactoryCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class ProxyApp {
    @Bean
    fun tomcatForwardedCustomizer() =
        WebServerFactoryCustomizer<TomcatServletWebServerFactory> { factory ->
            factory.addEngineValves(RemoteIpValve().apply {
                protocolHeader = "X-Forwarded-Proto"
                remoteIpHeader = "X-Forwarded-For"
                internalProxies = "10\\.0\\.\\d+\\.\\d+|192\\.168\\.\\d+\\.\\d+"
            })
        }
}

fun main(args: Array<String>) = runApplication<ProxyApp>(*args)

@RestController
class UrlController {
    @GetMapping("/url")
    fun url(r: HttpServletRequest) = "${r.scheme}://${r.getHeader("Host")}${r.requestURI}"
}
```

`application.yml` (дополнительно):

```yaml
server:
  forward-headers-strategy: framework
```

---

## TLS-терминация: где заканчивается шифрование (edge vs app), mTLS к внутренним зависимостям

TLS можно «закончить» на периметре (LB/Ingress) — тогда трафик до приложения идёт как HTTP внутри кластера. Это проще и дешевле по CPU, но требует доверенной внутренней сети. Второй вариант — TLS до приложения; это нужно, если сеть недоверенная или вы хотите сквозной h2. Комбинированный подход: edge-TLS для внешнего трафика и **mTLS** для внутренних вызовов между сервисами (двусторонняя аутентификация сертификатами).

При mTLS каждый участник предъявляет сертификат; сервис проверяет **клиентский** сертификат (client auth). В Spring Boot серверная сторона включается через `server.ssl.*` с `client-auth=need/want`, а клиент — через `SslContext`/`HttpClient`/`RestClient` с указанием client keystore и truststore. Сертификаты желательно перекатывать и ротировать автоматически (например, через cert-менеджер в K8s).

Если терминируете TLS на прокси, не забудьте пробрасывать схему в заголовках (`X-Forwarded-Proto: https`), иначе приложение будет считать, что протокол — `http` и генерировать неправильные ссылки. Для gRPC/H2 лучше держать TLS до приложения.

Проверка клиентских сертификатов — место, где легко «выстрелить в ногу»: убедитесь, что доверяете **только** нужному CA (корпоративному), а не системному «всему миру». Иначе чужой сертификат может пройти проверку. Для повышения безопасности используйте SAN-ограничения и короткоживущие сертификаты.

Производительность: TLS стоит CPU. На Java 17+ это недорого, но всё же учитывайте при sizing. ALPN обязателен для HTTP/2: убедитесь, что у вас нужные JDK и базовый образ поддерживают его из коробки.

Для отладки TLS в контейнерах полезно логировать включение SSL, печатать «как минимум» используемый протокол/шифр при старте, но **не** логировать содержимое ключей/паролей.

**Код (Java): сервер с TLS и mTLS, клиент с client-cert**

```java
package com.example.tls;

import jakarta.annotation.PostConstruct;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestClient;

@SpringBootApplication
public class TlsApp {
  public static void main(String[] args) { SpringApplication.run(TlsApp.class, args); }
}

@RestController
class TlsController {
  @GetMapping("/secure")
  public String secure() { return "ok"; }
}

@Configuration
class MtlsClientConfig {
  @PostConstruct
  void log() { System.out.println("mTLS client ready"); }

  @Bean
  RestClient mtlsClient() throws Exception {
    var ssl = org.springframework.http.client.JdkClientHttpRequestFactoryBuilder
        .create().ssl((s) -> {
          s.trustStore(new java.io.File("truststore.jks").toPath(), "changeit".toCharArray());
          s.keyStore(new java.io.File("keystore.p12").toPath(), "changeit".toCharArray(), "changeit".toCharArray());
        }).build();
    return RestClient.builder().requestFactory(ssl).baseUrl("https://internal-service:8443").build();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.tls

import jakarta.annotation.PostConstruct
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.client.RestClient

@SpringBootApplication
class TlsApp

fun main(args: Array<String>) = runApplication<TlsApp>(*args)

@RestController
class TlsController {
    @GetMapping("/secure")
    fun secure() = "ok"
}

@Configuration
class MtlsClientConfig {
    @PostConstruct fun log() = println("mTLS client ready")

    @Bean
    fun mtlsClient(): RestClient {
        val factory = org.springframework.http.client.JdkClientHttpRequestFactoryBuilder
            .create().ssl {
                it.trustStore(java.io.File("truststore.jks").toPath(), "changeit".toCharArray())
                it.keyStore(java.io.File("keystore.p12").toPath(), "changeit".toCharArray(), "changeit".toCharArray())
            }.build()
        return RestClient.builder().requestFactory(factory).baseUrl("https://internal-service:8443").build()
    }
}
```

`application.yml` (сервер, mTLS):

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-store-type: PKCS12
    client-auth: need
    trust-store: classpath:truststore.jks
    trust-store-password: ${TRUSTSTORE_PASSWORD}
```

---

## Ограничения: лимиты заголовков/тела, загрузка файлов, защита от slowloris

Лимиты — ваш спасательный круг против злоупотреблений и аварий. На сервере приложений ограничьте размер **заголовков** и **тела**. Большие JWT/кучи cookies легко выбивают сервер ошибками 400/431. В Spring Boot есть `server.max-http-header-size`, multipart-лимиты — `spring.servlet.multipart.*`. На прокси тоже установите соответствующие лимиты, чтобы не пускать избыточный трафик внутрь.

Файлоприём — отдельная история: в Java потоковая обработка предпочтительна, чтобы не держать гигабайты в памяти. Заранее определите максимальные размеры, временные каталоги (в контейнере их может не быть!) и место хранения. Никогда не доверяйте имени файла от клиента и ограничивайте расширения/конвертацию.

Slowloris-атаки (медленная отправка запроса/заголовков) «отъедают» коннекты. На прокси включайте «read timeout» на заголовки/тело, на Tomcat — `connection-timeout`. Для бэкендов, которые долго пишут ответ, выставляйте разумный write-timeout — иначе «залипшие» соединения съедят пул.

Сжатие и большие ответы усиливают влияние лимитов: на прокси и приложении таймауты/буферы должны быть согласованы. Для больших скачиваний лучше отдавать через статику прокси/S3-пресайнд и не держать долгоживущие коннекты в приложении.

Наконец, логируйте причины отказов **без** утечек данных: статус/причина/идентификатор запроса, но не содержимое тела. Это поможет отличить баг в клиенте от атаки.

Регулярно тестируйте лимиты нагрузкой/генератором «плохих» запросов. Лучше найти дисбаланс на стенде, чем в проде.

**Код (Java): загрузка файла + ограничение размеров**

```java
package com.example.limits;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;

@RestController
@RequestMapping("/upload")
public class UploadController {
  @PostMapping(consumes = "multipart/form-data")
  public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) throws IOException {
    if (file.getSize() > 10 * 1024 * 1024) { // дополнительная защита
      return ResponseEntity.badRequest().body("File too large");
    }
    // обработка потоком
    long bytes = file.getInputStream().transferTo(java.io.OutputStream.nullOutputStream());
    return ResponseEntity.ok("received=" + bytes);
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.limits

import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import org.springframework.web.multipart.MultipartFile

@RestController
@RequestMapping("/upload")
class UploadController {
    @PostMapping(consumes = ["multipart/form-data"])
    fun upload(@RequestParam("file") file: MultipartFile): ResponseEntity<String> {
        if (file.size > 10 * 1024 * 1024) return ResponseEntity.badRequest().body("File too large")
        file.inputStream.use { it.transferTo(java.io.OutputStream.nullOutputStream()) }
        return ResponseEntity.ok("received=${file.size}")
    }
}
```

`application.yml` (лимиты/таймауты):

```yaml
server:
  max-http-header-size: 16KB
  tomcat:
    connection-timeout: 10s
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 12MB
```

---

# 5. Логи, метрики и трассировка в контейнерах

## Логи в stdout/stderr, структурированные JSON; ротация лог-драйвером, а не приложением

В контейнерах логи — это **stdout/stderr**. Никаких «каталоги логов внутри контейнера»: так вы теряете централизованный сбор и ротацию. Лог-драйвер Docker/K8s сам собирает вывод и отдаёт его в Fluent Bit/Vector/Logstash. Формат — лучше JSON, чтобы парсить поля без грубого grok.

Ротация — забота платформы (лог-драйвер/агент/Elasticsearch SLM), а не приложения. Если внутри приложения включить собственную ротацию, вы получите дубликаты, проблемы с inode и нетипичное поведение при ротации. Оставьте приложению только форматирование и уровень.

Структурируйте поля: `ts, level, logger, thread, traceId, spanId, message, kv…`. Маскируйте секреты (см. ниже). Для высоконагруженных сервисов подумайте о асинхронном аппендере, но не «перестарайтесь»: при падении процесс может не сбросить буферы.

Уровни логов: в проде разумно держать `INFO/WARN/ERROR`, а `DEBUG/TRACE` включать точечно через `/actuator/loggers` на короткий период. **Запрещайте** логировать тела запросов/ответов для приватных API — это риск утечки.

На локали/стендах включайте «человеческие» паттерны (цвета/короткий формат), но в проде — JSON. Согласуйте формат со сборщиками логов и SRE — чтобы поля прилетали в унифицированную схему.

Для корреляции добавляйте `traceId/spanId` (см. ниже). Это позволит найти все логи по одной операции через весь парк сервисов.

**Код (Java): логируем в stdout, JSON формат, пример логирования**

```java
package com.example.logs;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
public class LogApp { public static void main(String[] args) { SpringApplication.run(LogApp.class, args); } }

@RestController
class LogController {
  private static final Logger log = LoggerFactory.getLogger(LogController.class);

  @GetMapping("/work")
  public String work() {
    log.info("start work taskId={}", java.util.UUID.randomUUID());
    return "ok";
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.logs

import org.slf4j.LoggerFactory
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class LogApp

fun main(args: Array<String>) = runApplication<LogApp>(*args)

@RestController
class LogController {
    private val log = LoggerFactory.getLogger(LogController::class.java)
    @GetMapping("/work")
    fun work(): String {
        log.info("start work taskId={}", java.util.UUID.randomUUID())
        return "ok"
    }
}
```

`logback-spring.xml` (JSON вывод):

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

Gradle зависимости (Groovy/Kotlin):

```groovy
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

```kotlin
implementation("net.logstash.logback:logstash-logback-encoder:7.4")
```

---

## Actuator/Observation/Micrometer: экспонирование метрик, /actuator/health/prometheus/otel

Micrometer — унифицированный API метрик. В Spring Boot вы включаете `spring-boot-starter-actuator`, и получаете `/actuator/health`, `/actuator/metrics/**`, `/actuator/prometheus` (при наличии регистра Prometheus). «Observation» — новый каркас для таймингов и трассировки; аннотация `@Observed` добавляет метрики/спаны поверх методов.

Экспонирование для Prometheus делайте через `/actuator/prometheus` и scrape в Prometheus/Agent. Для OpenTelemetry можно добавить OTLP-экспорт в коллектор. Важно: **не** открывайте все эндпойнты наружу; выставляйте только то, что нужно, и ограничивайте доступ сетево/авторизацией.

Создавайте бизнес-метрики: количество успешно обработанных событий, ретраи, состояние внутренних очередей. Это помогает SRE/дежурным понимать поведение, а не только CPU/GC. Для высоконагруженных путей используйте **гистограммы** с подходящими SLA-биннами, чтобы видеть перцентили.

Health-пробы лучше разделять на liveness/readiness (см. ниже). Не заставляйте liveness зависеть от внешних систем — иначе получите перезапуски из-за временной деградации БД/кэша. Readiness — как раз место, где можно проверять зависимость.

Следите за накладными расходами: тонкая выборка метрик, фильтрация ненужных регистров, выборочная регистрация таймеров. На «шумных» эндпойнтах не включайте подробные метрики без нужды.

В проде полезно иметь `/actuator/info` с версией сборки (см. ранее) и git-метками. Это упрощает расследование инцидентов.

**Код (Java): кастомная метрика и @Observed**

```java
package com.example.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.observation.annotation.Observed;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
public class MetricsApp { public static void main(String[] args) { SpringApplication.run(MetricsApp.class, args); } }

@Service
class Worker {
  private final MeterRegistry meter;
  Worker(MeterRegistry meter) { this.meter = meter; }

  @Observed(name = "worker.process", contextualName = "process-job")
  public String process() {
    meter.counter("worker.process.count").increment();
    return "done";
  }
}

@RestController
class MetricsController {
  private final Worker worker;
  MetricsController(Worker worker) { this.worker = worker; }

  @GetMapping("/do")
  public String doWork() { return worker.process(); }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.metrics

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.observation.annotation.Observed
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class MetricsApp

fun main(args: Array<String>) = runApplication<MetricsApp>(*args)

@Service
class Worker(private val meter: MeterRegistry) {
    @Observed(name = "worker.process", contextualName = "process-job")
    fun process(): String {
        meter.counter("worker.process.count").increment()
        return "done"
    }
}

@RestController
class MetricsController(private val worker: Worker) {
    @GetMapping("/do")
    fun doWork(): String = worker.process()
}
```

`application.yml` (Prometheus/health):

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,loggers
  endpoint:
    health:
      probes:
        enabled: true
```

---

## Корреляция: traceId/spanId в MDC, интеграция с агентами и коллекторами; ресурсоёмкость и сэмплинг

Корреляция — способность связать логи/метрики по одному идентификатору запроса. С Observability в Boot traceId/spanId автоматически попадают в MDC при наличии bridge-зависимостей (например, micrometer-tracing + otel). Если не используете агента — добавляйте `X-Request-ID` вручную и прокидывайте через MDC.

Интеграция с OpenTelemetry: вы можете собрать спаны и отправлять в OTLP-коллектор (в DaemonSet/sidecar). Это даёт end-to-end трассировку через все сервисы. Но имейте в виду накладные: спаны стоят CPU/память/сеть. Настраивайте sampling (например, 0.1-1.0% на проде для «шумных» сервисов).

Сэмплинг может быть динамическим: повышать долю для ошибок (tail sampling), снижать — для «зелёных» путей. В любом случае, traceId должен доходить до логов, чтобы по нему находить редкие случаи. Не забывайте проксировать/сохранять заголовки `traceparent`, `baggage`.

Клиентские библиотеки (WebClient/RestClient) уже умеют прокидывать контекст; для «сырого» HTTP клиента/DB драйвера иногда нужен ручной MDC/labels. Для Kafka/Rabbit стоит включить корреляцию на уровне consumer/producer через headers.

Будьте аккуратны с MDC при использовании реактивных потоков или собственных пулов: контекст «теряется» между тредами; используйте `Context`/hooks или обёртки executors, чтобы переносить MDC.

Главное: корреляция — часть операционки. Согласуйте формат traceId/spanId в логах и в графах трассировки, чтобы их было легко сопоставлять.

**Код (Java): простой фильтр, добавляющий requestId в MDC**

```java
package com.example.trace;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import java.io.IOException;
import java.util.UUID;

@Component
public class RequestIdFilter implements Filter {
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    String id = r.getHeader("X-Request-ID");
    if (id == null || id.isBlank()) id = UUID.randomUUID().toString();
    MDC.put("requestId", id);
    try { chain.doFilter(req, res); } finally { MDC.remove("requestId"); }
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.trace

import jakarta.servlet.Filter
import jakarta.servlet.FilterChain
import jakarta.servlet.ServletRequest
import jakarta.servlet.ServletResponse
import jakarta.servlet.http.HttpServletRequest
import org.slf4j.MDC
import org.springframework.stereotype.Component
import java.util.UUID

@Component
class RequestIdFilter : Filter {
    override fun doFilter(req: ServletRequest, res: ServletResponse, chain: FilterChain) {
        val r = req as HttpServletRequest
        val id = r.getHeader("X-Request-ID")?.takeIf { it.isNotBlank() } ?: UUID.randomUUID().toString()
        MDC.put("requestId", id)
        try { chain.doFilter(req, res) } finally { MDC.remove("requestId") }
    }
}
```

---

## Диагностика на проде: безопасное включение debug-логов и маскирование секретов

В проде иногда нужно «подсветить» конкретный пакет/класс. Делайте это через `/actuator/loggers` (меняя уровень в рантайме) и на короткое время. После — возвращайте в `INFO`. Никогда не включайте глобальный `DEBUG` — это лавина данных и риск утечек.

Маскирование секретов критично: пароли/токены/ключи не должны попадать в логи. Можно использовать `TurboFilter`/`MaskingConverter` в Logback, который заменяет совпадения по regex на `***`. Плюс — дисциплина в коде: не логировать конфигурации целиком, а только факт наличия/статус.

Для проблемных запросов используйте traceId/requestId и выборочную трассировку (см. сэмплинг). Для долгих операций — профилирующие endpoints/async стеки — но аккуратно: не включайте профилировщики на больших процентилях трафика.

«Диагностические» флаги (вроде `?debug=true`) отключайте на проде или подвязывайте под аутентификацию. Если нужно видеть тело запроса, делайте это на стенде/теневом трафике, не в бою.

Удобно иметь «диагностическую» страницу с информацией версии/времени запуска/основных параметров — но без секретов. Это поможет при инцидентах, не заглядывая внутрь контейнера.

Соблюдайте ретеншн-политику логов и PII/персональных данных: минимизируйте сбор, шифруйте хранение, ограничивайте доступ.

**Код (Java): маскирование секретов в Logback через TurboFilter (пример)**

```java
package com.example.mask;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.spi.FilterReply;
import ch.qos.logback.classic.turbo.TurboFilter;
import java.util.regex.Pattern;

public class SecretMaskingTurboFilter extends TurboFilter {
  private final Pattern pattern = Pattern.compile("(password|secret|token)=([^&\\s]+)", Pattern.CASE_INSENSITIVE);
  @Override
  public FilterReply decide(org.slf4j.Marker marker, ch.qos.logback.classic.Logger logger, Level level, String format, Object[] params, Throwable t) {
    if (format != null) {
      var m = pattern.matcher(format);
      if (m.find()) {
        String masked = m.replaceAll("$1=***");
        logger.callAppenders(new ch.qos.logback.classic.spi.LoggingEvent(logger.getName(), logger, level, masked, t, params));
        return FilterReply.DENY;
      }
    }
    return FilterReply.NEUTRAL;
  }
}
```

**Код (Kotlin): простая маскировка в приложении (дополнение к фильтру)**

```kotlin
package com.example.mask

object Masker {
    private val regex = Regex("(password|secret|token)=([^&\\s]+)", RegexOption.IGNORE_CASE)
    fun safe(msg: String) = msg.replace(regex, "$1=***")
}
```

`logback-spring.xml` (подключение TurboFilter):

```xml
<configuration>
  <turboFilter class="com.example.mask.SecretMaskingTurboFilter"/>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>
  <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

---

# 6. Жизненный цикл, здоровье и останов

## Liveness/Readiness/Startup: что именно проверять и какие статусы; зависимость от БД/кеша/очередей

Liveness отвечает на вопрос «жив ли процесс». Если он отвечает — kube не должен его убивать. Нельзя в liveness проверять внешние зависимости: иначе при кратком падении БД процесс начнёт перезапускаться по кругу. Liveness — это «сам себе доктор»: внутренняя петля, deadlock-детектор, базовый ответ приложения.

Readiness — «готов ли принимать трафик». Здесь уже можно проверять доступность БД/кеша/очередей. Если зависимость падает — readiness должен стать `DOWN`, чтобы маршрутизатор перестал слать трафик, но процесс продолжал жить и пытаться восстановиться.

Startup-проба помогает длинно стартующим сервисам. Пока она не `UP`, kube не включает liveness/readiness. Полезно для сервисов с прогревом кэшей/компиляцией шаблонов/подключениями.

Золотое правило: **liveness — лёгкая и автономная**, **readiness — агрессивная и зависимая**. Не путайте их, иначе получите или ложные убийства процесса, или вечный трафик в неготовые инстансы.

В Spring Boot включите «probes»: Boot сам учитывает состояния и отдаёт `livenessState/readinessState`. Добавляйте кастомные `HealthIndicator` для зависимостей, но аккуратно с таймаутами — readiness не должен зависать на минуты.

Проверяйте пробы нагрузкой и отказами: отключайте БД/кэш и смотрите, как быстро readiness падает и восстанавливается. Это важнее теории.

**Код (Java): кастомный readiness HealthIndicator для БД**

```java
package com.example.health;

import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;

@Component("dbReadiness")
public class DbReadinessIndicator implements HealthIndicator {
  private final DataSource ds;
  public DbReadinessIndicator(DataSource ds) { this.ds = ds; }
  @Override public Health health() {
    try (Connection c = ds.getConnection()) {
      return Health.up().withDetail("db", "ok").build();
    } catch (Exception e) {
      return Health.down(e).withDetail("db", "down").build();
    }
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.health

import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.stereotype.Component
import javax.sql.DataSource

@Component("dbReadiness")
class DbReadinessIndicator(private val ds: DataSource) : HealthIndicator {
    override fun health(): Health = try {
        ds.connection.use { Health.up().withDetail("db", "ok").build() }
    } catch (e: Exception) {
        Health.down(e).withDetail("db", "down").build()
    }
}
```

`application.yml`:

```yaml
management:
  endpoint.health.probes.enabled: true
```

---

## Graceful shutdown: Spring Boot graceful-period, обработка SIGTERM, preStop-хуки и дренирование трафика

Graceful shutdown — корректное завершение: перестать принимать новые запросы, дождаться завершения текущих, закрыть ресурсы. В Boot 3 включите `server.shutdown=graceful` и задайте время на останов. Kubernetes сначала отправит `SIGTERM`, затем подождёт `terminationGracePeriodSeconds`, после чего пришлёт `SIGKILL`.

Важно, чтобы прокси «дренировали» трафик: перед SIGTERM полезно иметь `preStop`-хук, который сначала дергает `/sleep`/`/ready=false` или просто спит пару секунд, чтобы убрать Pod из балансировки. И только потом — `SIGTERM`. Так вы избежите 5xx в конце релиза.

Долгие фоновые задачи нужно прерывать. В Java ловите `InterruptedException`, проверяйте флаг `Thread.interrupted()` в петлях. Для consumer’ов (Kafka/Rabbit) закрывайте подписки в `@PreDestroy`.

Не ставьте слишком маленький `terminationGracePeriodSeconds`: сложные сервисы с открытыми коннектами (DB, внешние API) могут не успеть корректно завершиться. Лучше чуть «перебдеть», чем ронять активные запросы.

Тестируйте останов: запускайте сервис, шлите нагрузку, посылайте `SIGTERM` и смотрите, что происходит. Это помогает найти «забытые» фоновые потоки и блокировки.

Логируйте начало/конец graceful shutdown и количество «незавершённых» задач — это ключевые маркеры в расследованиях.

**Код (Java): graceful-хуки и прерывание работы**

```java
package com.example.grace;

import jakarta.annotation.PreDestroy;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.stereotype.Component;
import java.util.concurrent.*;

@SpringBootApplication
public class GraceApp { public static void main(String[] args) { SpringApplication.run(GraceApp.class, args); } }

@Component
class Worker implements Runnable {
  private final ExecutorService pool = Executors.newFixedThreadPool(2);
  public Worker() { pool.submit(this); }
  @Override public void run() {
    try {
      while (!Thread.currentThread().isInterrupted()) {
        // имитация работы
        Thread.sleep(200);
      }
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
  }
  @PreDestroy
  public void stop() throws InterruptedException {
    pool.shutdown();
    if (!pool.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS)) pool.shutdownNow();
    System.out.println("Graceful stopped");
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.grace

import jakarta.annotation.PreDestroy
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.stereotype.Component
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit

@SpringBootApplication
class GraceApp

fun main(args: Array<String>) = runApplication<GraceApp>(*args)

@Component
class Worker : Runnable {
    private val pool = Executors.newFixedThreadPool(2)
    init { pool.submit(this) }
    override fun run() {
        try {
            while (!Thread.currentThread().isInterrupted) Thread.sleep(200)
        } catch (e: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
    @PreDestroy
    fun stop() {
        pool.shutdown()
        if (!pool.awaitTermination(5, TimeUnit.SECONDS)) pool.shutdownNow()
        println("Graceful stopped")
    }
}
```

`application.yml` и K8s preStop:

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 10s
```

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh","-c","sleep 3"]
terminationGracePeriodSeconds: 20
```

---

## Инициализация: Liquibase/Flyway, прогрев кэшей/соединений; отдельные init-контейнеры/джобы

Миграции схемы БД должны выполняться **до** старта приложений, которым нужна новая схема. В Kubernetes это удобно делать отдельной Job (или init-контейнером), чтобы сам сервис не зависел от прав на миграции. Для малых проектов допустимо оставить Liquibase/Flyway внутри сервиса, но аккуратно с параллельным стартом нескольких Pod’ов.

Прогрев кэшей/соединений ускоряет первые запросы. Полезно заранее прогреть Hikari-пул, JIT/кэш шаблонов, подготовить часто используемые данные. Делайте это в `ApplicationRunner`/`SmartLifecycle.start()`, но не держите readiness «UP», пока прогрев не завершился.

Init-контейнеры хороши для загрузки артефактов/сертификатов/генерации конфигов. Они запускаются перед основным контейнером и гарантируют «правильное» состояние файловой системы/секретов.

Если миграций много, разделяйте **DDL-джобу** и **деплой приложения** — так вы сможете скатывать миграции отдельно, проверять их, откатывать без влияния на движок сервиса. В Helm это два шаблона: Job и Deployment.

Важно: миграции должны быть **идемпотентны** и быстры по максимуму. Большие DDL лучше через maintenance-окна/онлайн-алгоритмы.

Наблюдаемость и логи миграций: отправляйте в stdout и собирайте централизованно. Не храните лог-файлы в контейнере.

**Код (Java): прогрев кэша при старте**

```java
package com.example.init;

import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class WarmupConfig {
  @Bean
  ApplicationRunner warmup() {
    return args -> {
      // прогреваем что-то часто используемое
      Thread.sleep(500);
      System.out.println("Warmup done");
    };
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.init

import org.springframework.boot.ApplicationRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class WarmupConfig {
    @Bean
    fun warmup() = ApplicationRunner {
        Thread.sleep(500)
        println("Warmup done")
    }
}
```

`application.yml` (Liquibase внутри приложения, пример):

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

K8s Job для миграций (упрощённо):

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: demo-liquibase }
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: liquibase
          image: registry/demo:1.0.0
          args: ["java","-jar","/app/app.jar","--spring.liquibase.enabled=true","--spring.main.web-application-type=none"]
          envFrom: [ { secretRef: { name: db-secret } } ]
```

---

## Стратегии перезапуска и защита от crashloop: экспоненциальные задержки, быстрая диагностика причин

CrashLoopBackOff — когда контейнер падает сразу после старта. Причины: неверные конфиги/секреты, миграции, порты. Первая линия защиты — **fail-fast** и внятные логи старта. Либо всё Ok и сервис работает, либо он быстро и очевидно падает.

Экспоненциальные backoff’ы на клиентах (HTTP/Kafka/DB) помогают не завалить зависимость штормом ретраев при инциденте. Но не путайте это с рестарт-политиками K8s — они про процесс, а не про сетевые вызовы. В приложении делайте короткий retry с джиттером и чёткими лимитами.

Для сервисов-консьюмеров удобно иметь «safe-mode» профиль, где консьюминг не запускается, пока вы не проверите готовность зависимостей. Это спасает при раскрутке сложных сред.

Диагностика: выводите конфиг-факты при старте (без секретов), подключайтесь к логу контейнера, смотрите события Pod’а/Job’а. Статусы `/actuator/health` должны отражать истинное состояние.

Иногда лучше «умереть» быстро, чем «жить» в полурабочем состоянии. Если обязательная зависимость недоступна при старте и без неё сервис бессмысленен — падайте. Но если зависимость может появиться позже — держите сервис живым с `readiness=DOWN`.

И ещё: тестируйте падения. «Чёрный ящик» старта/остановки должен быть прозрачен SRE и разработчикам.

**Код (Java): fail-fast в ApplicationRunner и простой экспоненциальный retry**

```java
package com.example.crash;

import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class StartupChecks {
  @Bean
  ApplicationRunner checkRequiredEnv() {
    return args -> {
      String url = System.getenv("REQUIRED_URL");
      if (url == null || url.isBlank()) throw new IllegalStateException("REQUIRED_URL is missing");
    };
  }

  public static <T> T retry(java.util.concurrent.Callable<T> call) throws Exception {
    long delay = 100;
    for (int i=0; i<5; i++) {
      try { return call.call(); }
      catch (Exception e) { Thread.sleep(delay); delay *= 2; }
    }
    return call.call();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.crash

import org.springframework.boot.ApplicationRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.Callable

@Configuration
class StartupChecks {
    @Bean
    fun checkRequiredEnv() = ApplicationRunner {
        val url = System.getenv("REQUIRED_URL")
        require(!url.isNullOrBlank()) { "REQUIRED_URL is missing" }
    }
}

@Throws(Exception::class)
fun <T> retry(call: Callable<T>): T {
    var delay = 100L
    repeat(5) {
        try { return call.call() } catch (_: Exception) { Thread.sleep(delay); delay *= 2 }
    }
    return call.call()
}
```

---

# 7. Ресурсы и производительность в контейнере

## Requests/limits и влияние на JVM: нитепулы, реакция на throttling; настройка parallelism

В Kubernetes requests/limits задают «квоты» CPU/памяти. JVM с cgroup-осведомлённостью учитывает эти лимиты, но не магически: если limit CPU низкий, thread-pools, которые вы задали «по умолчанию = N CPU», начнут конкурировать. Настройте размеры пулов (Tomcat, @Async, scheduler, DB) исходя из `availableProcessors()` **и** реальной нагрузки.

CPU-throttling (когда вы упёрлись в limit) виден как задержки: GC/таймеры/пулы начинают «рвано» работать. Симптомы — скачки латентности без роста RPS. Лекарства: поднимите limit, уменьшите параллелизм, распараллеливайте задачи по нескольким Pod’ам.

Requests — база для HPA. Если зададите слишком маленький request, Pod’ов может поместиться слишком много на ноду, и все начнут драться за CPU; слишком большой — недоиспользование кластера. Ищите баланс по метрикам.

Пулы задач должны учитывать и IO-характер — CPU-bound vs IO-bound. Для CPU-bound держите размер ~N CPU; для IO-bound — больше, но с бэкпрешсером и таймаутами. Не забывайте про GC-паузы: много тредов ≠ быстро.

Проверяйте `Runtime.getRuntime().availableProcessors()` в рантайме — это «видимый» JVM CPU после cgroups. Но не слепо умножайте на константы — профилируйте. Иногда уменьшение параллелизма даёт лучший tail latency.

Согласуйте Hikari-пул/HTTP-клиент/экзекьюторы, чтобы они не создавали «бурю» параллельных работ при пиках.

**Код (Java): настраиваем пул @Async от числа «видимых» CPU**

```java
package com.example.res;

import org.springframework.context.annotation.*;
import org.springframework.scheduling.annotation.*;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@EnableAsync
@Configuration
public class AsyncConfig {
  @Bean(name = "ioPool")
  public ThreadPoolTaskExecutor ioPool() {
    int cpus = Runtime.getRuntime().availableProcessors();
    ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(Math.max(2, cpus * 2));
    ex.setMaxPoolSize(Math.max(4, cpus * 4));
    ex.setQueueCapacity(1000);
    ex.setThreadNamePrefix("io-");
    ex.initialize();
    return ex;
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.res

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.annotation.EnableAsync
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor

@EnableAsync
@Configuration
class AsyncConfig {
    @Bean("ioPool")
    fun ioPool(): ThreadPoolTaskExecutor {
        val cpus = Runtime.getRuntime().availableProcessors()
        return ThreadPoolTaskExecutor().apply {
            corePoolSize = maxOf(2, cpus * 2)
            maxPoolSize = maxOf(4, cpus * 4)
            setQueueCapacity(1000)
            setThreadNamePrefix("io-")
            initialize()
        }
    }
}
```

---

## Память: Xms/Xmx vs MaxRAMPercentage; off-heap (Netty, кеши), tmpfs и размер слоёв

В контейнере лучше использовать **процентную модель** памяти (`-XX:MaxRAMPercentage=`), чтобы JVM адаптировалась к лимиту. Ставить `Xms=Xmx` имеет смысл только если вы уверены в профиле нагрузки и хотите избегать роста heap. Не забывайте про metaspace/кодач/стек/буферы — оставляйте запас.

Off-heap: Netty/ByteBuffer/Hazelcast/Caffeine могут забирать память вне heap. Следите за суммарным потреблением и лимитируйте размер кэшей. Иначе словите OOM от нативной памяти, даже если `Xmx` маленький. Для Netty можно ограничивать арену/директ-буферы.

Временные файлы: в distroless их стоит держать в `/tmp` или монтировать `emptyDir`/`tmpfs`. Большие аплоады в память — плохая идея; используйте потоковую обработку и файлы.

Размер слоёв образа — это и скорость раскатки. Убирайте лишние артефакты из рантайм-слоя, чистите менеджеры пакетов на build-слое, используйте `--no-cache`. Layered JAR уменьшает частичные обновления.

Следите за ошибками «Container killed due to OOM»: kube evicts Pod без шанса на дамп. Лучше заранее контролировать память и падать с понятным логом, чем тихо умирать.

Мониторьте `process.memory.usage`, GC метрики, `container_memory_*` на уровне kube, и коррелируйте с RPS/latency, чтобы ловить утечки и неправильный sizing.

**Код (Java): печать видимой памяти/CPU + пример off-heap**

```java
package com.example.mem;

import org.springframework.web.bind.annotation.*;
import java.nio.ByteBuffer;

@RestController
public class MemController {
  @GetMapping("/sys")
  public String sys() {
    long maxMb = Runtime.getRuntime().maxMemory() / (1024*1024);
    int cpus = Runtime.getRuntime().availableProcessors();
    return "maxMemMb=" + maxMb + " cpus=" + cpus;
  }

  @GetMapping("/offheap")
  public String offheap() {
    ByteBuffer buf = ByteBuffer.allocateDirect(1024 * 1024);
    return "allocatedDirect=" + buf.capacity();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.mem

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.nio.ByteBuffer

@RestController
class MemController {
    @GetMapping("/sys")
    fun sys(): String {
        val maxMb = Runtime.getRuntime().maxMemory() / (1024*1024)
        val cpus = Runtime.getRuntime().availableProcessors()
        return "maxMemMb=$maxMb cpus=$cpus"
    }
    @GetMapping("/offheap")
    fun offheap(): String {
        val buf = ByteBuffer.allocateDirect(1024 * 1024)
        return "allocatedDirect=${buf.capacity()}"
    }
}
```

Dockerfile (часть):

```dockerfile
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
```

---

## Стартап/тёплый JIT: CDS/AOT/native-image/CRaC; выбор под профиль нагрузки

Время старта важно в автоскейлинге. Классический JVM-стартап можно ускорить Class Data Sharing (CDS/AppCDS): предзагрузка классов в архив уменьшает время загрузки. В Spring Boot 3.x есть AOT-оптимизации (генерируются при сборке), а также поддержка сборки **native-image** на GraalVM — это ещё быстрее стартует и ест меньше памяти, но имеет компромиссы по совместимости/функциональности (рефлексия, прокси и т.п.).

CRaC (Coordinated Restore at Checkpoint) позволяет «заморозить» прогретую JVM и быстро «восстановить». Это интересно для serverless/быстрых рестартов, но требует аккуратной интеграции (ресурсы, сетевые сокеты).

Выбор: для «долго живущих» сервисов критична **устойчивость в нагрузке**, а не старт; JIT «разгоняется» через прогрев. Для serverless/высокоэластичных путей — native-image/CRaC даёт выигрыш. CDS — недорогая оптимизация почти без минусов.

Наблюдайте **tail latency** при прогреве: первые минуты нагрузка может «гулять» из-за JIT/кэш-прогрева. Хитрый приём — прогревать оффлайн в init-фазе типовые запросы.

При native-image следите за агентами/инструментами: не всё переносится (например, dynamic attach агенты). Проверяйте метрики/трейсинг/логирование.

Учитывайте размер образа: native-бинарь может быть увесистым, но рантайм-образ упрощается (без JVM). Это полезно для холодных запусков.

**Код (Java): AOT/Native базовый сервис**

```java
package com.example.start;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
public class StartApp { public static void main(String[] args) { SpringApplication.run(StartApp.class, args); } }

@RestController
class Hello {
  @GetMapping("/hello") public String hello() { return "hi"; }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.start

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class StartApp

fun main(args: Array<String>) = runApplication<StartApp>(*args)

@RestController
class Hello {
    @GetMapping("/hello") fun hello() = "hi"
}
```

Gradle (Groovy/Kotlin) — GraalVM plugin:

```groovy
plugins { id 'org.graalvm.buildtools.native' version '0.10.3' }
```

```kotlin
plugins { id("org.graalvm.buildtools.native") version "0.10.3" }
```

Сборка native через Buildpacks:

```bash
./gradlew bootBuildImage --imageName registry/demo:native --builder paketobuildpacks/builder-jammy-tiny --environment BP_NATIVE_IMAGE=true
```

---

## IO/FD-лимиты: количество соединений, connection pools (HTTP/DB), таймауты и backpressure

Файловые дескрипторы (FD) — это и сокеты. При высокой параллельности легко упереться в лимиты. В K8s/контейнерах обычно лимит достаточен, но при агрессивных клиентах/сервер-sent events его стоит мониторить. На уровне сервера приложений контролируйте `max-connections` (Tomcat) и очереди.

Пулы соединений должны быть соразмерны нагрузке и ресурсам: HikariCP для БД, HTTP-клиенты (Apache, JDK, Netty). Важно выставить **таймауты** (connect/read/write) и политику ретраев — иначе зависшие коннекты «съедят» пул. Для БД добавьте `connectionTimeout`, `maxLifetime`, `validationTimeout`.

Backpressure — защита от перегрузки: очереди/буферы ограничены, при переполнении — отклоняем запросы или замедляем производителей. Иначе вы просто копите работы в памяти и «умираете поздно и громко».

Сетевые таймауты должны быть согласованы через весь путь: клиент → прокси → приложение → внешняя зависимость. Иначе получится «шахматка» таймаутов и ретраев, которая только усугубляет шторма.

Для загружаемых файлов/стриминга используйте потоковые API, не буферизуйте всё в памяти, и учитывайте, что прокси может рвать соединение — готовьте возобновление/чанкинг.

Наконец, мониторьте: connection pool usage, rejected executions, очередь задач, FD count. Это лучший индикатор реальных проблем с ресурсами.

**Код (Java): Hikari и RestClient с таймаутами**

```java
package com.example.io;

import com.zaxxer.hikari.HikariDataSource;
import org.springframework.context.annotation.*;
import org.springframework.http.client.JdkClientHttpRequestFactoryBuilder;
import org.springframework.web.client.RestClient;

import javax.sql.DataSource;
import java.time.Duration;

@Configuration
public class IoConfig {
  @Bean
  DataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:postgresql://db:5432/app");
    ds.setUsername("app"); ds.setPassword("app");
    ds.setMaximumPoolSize(10);
    ds.setConnectionTimeout(1000);
    ds.setMaxLifetime(30_000);
    return ds;
  }

  @Bean
  RestClient httpClient() {
    var factory = JdkClientHttpRequestFactoryBuilder.create()
        .connectTimeout(Duration.ofMillis(800))
        .readTimeout(Duration.ofMillis(1500))
        .build();
    return RestClient.builder().requestFactory(factory).build();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.io

import com.zaxxer.hikari.HikariDataSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.client.JdkClientHttpRequestFactoryBuilder
import org.springframework.web.client.RestClient
import java.time.Duration
import javax.sql.DataSource

@Configuration
class IoConfig {
    @Bean
    fun dataSource(): DataSource = HikariDataSource().apply {
        jdbcUrl = "jdbc:postgresql://db:5432/app"
        username = "app"; password = "app"
        maximumPoolSize = 10
        connectionTimeout = 1000
        maxLifetime = 30_000
    }

    @Bean
    fun httpClient(): RestClient {
        val factory = JdkClientHttpRequestFactoryBuilder.create()
            .connectTimeout(Duration.ofMillis(800))
            .readTimeout(Duration.ofMillis(1500))
            .build()
        return RestClient.builder().requestFactory(factory).build()
    }
}
```

`application.yml` (Tomcat max connections):

```yaml
server:
  tomcat:
    max-connections: 8192
    accept-count: 200
```

# 8. Docker Compose: одиночный хост и тестовый прод

## Сети/volumes: изоляция сервисов, зависимости (depends_on + healthcheck)

В Compose мы получаем быстрый «тестовый прод»: несколько контейнеров с сетью по имени проекта и предсказуемыми DNS-именами. Практика — разбивать сервисы на отдельные сети: «frontend», «backend», «datastore». Это ограничивает область видимости и упрощает аудит. Изоляция особенно важна, когда вы одновременно поднимаете несколько стеков на одной машине — пересечения портов и коллизии имён прекрасно устраняются сетями Compose. Отдельные тома (volumes) держат состояние БД/кеша между перезапусками, а также позволяют делиться артефактами вроде миграций. В проде обычно томами управляет кластер (PVC), но на одном хосте volumes — «минимальный аналог».

Зависимости между контейнерами не гарантируют фактическую готовность сервиса: `depends_on` ждёт «контейнер запущен», но не «сервис готов принимать трафик». Поэтому добавляйте `healthcheck` на зависимости и управляйте порядком через `depends_on: condition: service_healthy`. Это особенно заметно при старте Postgres/Kafka — без healthcheck консьюмеры могут «проснуться» слишком рано и уйти в бесконечные ретраи.

Внутренние имена сервисов в Compose становятся DNS-именами. Это значит, что в приложении вы можете писать `jdbc:postgresql://db:5432/app` вместо `localhost`. Такой подход не только удобен — он приближает вас к Kubernetes, где сервисы также резолвятся по DNS. Переход с Compose на K8s в этом смысле становится механическим: меняются только значения хостов.

Volumes полезны и для разработчика: монтируйте локальную папку с миграциями/скриптами в контейнер и выполняйте их в «нативной среде». Это избавит от дрейфа «версия psql у меня vs в контейнере». Скучная, но реальная выгода — меньше «случайных» багов между машинами.

Завершая, помните, что «здоровье» сервиса должно быть быстрой и честной метрикой. Не заставляйте healthcheck выполнять тяжёлые запросы; достаточно `pg_isready`/`/actuator/health/readiness`. Цель — не флипать статусом, а защищать потребителей от гонок старта.

**Код (Compose, сети/тома/зависимости/healthcheck)**

```yaml
version: "3.9"
services:
  app:
    image: registry.local/demo:1.0.0
    depends_on:
      db:
        condition: service_healthy
    networks: [ backend ]
    ports: [ "8080:8080" ]
    environment:
      DB_URL: jdbc:postgresql://db:5432/app
      DB_USER: app
      DB_PASSWORD: app
  db:
    image: postgres:16
    networks: [ backend ]
    healthcheck:
      test: ["CMD-SHELL","pg_isready -U app -d app"]
      interval: 5s
      timeout: 2s
      retries: 10
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - dbdata:/var/lib/postgresql/data
networks:
  backend: {}
volumes:
  dbdata: {}
```

**Код (Java): простой ping для healthcheck приложением)**

```java
package com.example.compose;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class ComposeApp {
  public static void main(String[] args) { SpringApplication.run(ComposeApp.class, args); }
}

@RestController
class PingController {
  @GetMapping("/ping")
  public String ping() { return "pong"; }
}
```

**Код (Kotlin): тот же ping)**

```kotlin
package com.example.compose

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class ComposeApp

fun main(args: Array<String>) = runApplication<ComposeApp>(*args)

@RestController
class PingController {
    @GetMapping("/ping") fun ping() = "pong"
}
```

---

## Переменные окружения и секреты: .env, override файлы, режимы restart

Один из главных плюсов Compose — согласованная передача конфигурации через env. Файл `.env` в корне проекта автоматически подхватывается и подставляет значения в `docker-compose.yml`. Это удобно для локали, где секреты «псевдосекреты», но всё же не надо хранить их в Git. Для разных разработчиков значения могут отличаться — вы раздаёте только `.env.example` и каждый делает свой `.env`.

Для «средовых» различий заводите `docker-compose.override.yml`. По умолчанию он применяется автоматически и может заменять порты/переменные/тома. Подход напоминает Helm values и приучает держать общее описание в одном файле, а конкретику — в overrides. В CI можно использовать `-f` со своими файлами, собирая нужный профиль.

Режим `restart` (`no`, `on-failure`, `always`, `unless-stopped`) важен на одиночных хостах. Для приложений уместно `unless-stopped`: контейнер поднимется после перезагрузки машины. Но помните: поломки конфигурации не чинит никакой `restart` — делайте fail-fast с понятной ошибкой.

Секреты в Compose есть как сущность (`secrets:`), но на практике чаще используют env и тома. Если вы монтируете ключи/сертификаты — отдавайте их read-only и не держите в Git. На проде секрет-менеджер обязателен (Vault, KMS), но для локали — файл вполне допустим.

Не забывайте про принцип «не зашивать значения в образ». Образ должен быть одинаков на всех окружениях; .env и overrides дают вам достаточно рычагов для конфигурации без пересборки.

**Код (.env и override)**

```
# .env
DB_USER=app
DB_PASSWORD=app
SSL_PASSWORD=changeit
```

```yaml
# docker-compose.override.yml
services:
  app:
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SSL_PASSWORD: ${SSL_PASSWORD}
    restart: unless-stopped
```

**Код (Java): чтение env для демонстрации внешней конфигурации**

```java
package com.example.env;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
class EnvController {
  EnvController(@Value("${spring.profiles.active:default}") String p,
                @Value("${DB_USER:unset}") String user) {
    this.p = p; this.user = user;
  }
  private final String p, user;

  @GetMapping("/env")
  public String env() { return "profile=" + p + " user=" + user; }
}
```

**Код (Kotlin): то же**

```kotlin
package com.example.env

import org.springframework.beans.factory.annotation.Value
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class EnvController(
    @Value("\${spring.profiles.active:default}") private val p: String,
    @Value("\${DB_USER:unset}") private val user: String
) {
    @GetMapping("/env") fun env() = "profile=$p user=$user"
}
```

---

## Локальные зависимости: Postgres/Redis/Kafka; стратегия «максимально близко к прод»

Чтобы на локали ловить «настоящие» ошибки, тяните те же версии зависимостей, что и в проде. Compose хорош тем, что легко собрать стек: Postgres + Redis + Kafka + schema-registry. Все они подключаются по DNS имён сервисов, и ваш Spring Boot почти не отличит это от k8s-сервиса.

Kafka в связке с Spring Cloud Stream или обычным `spring-kafka` особенно требовательна к порядку старта: брокер должен быть «здоров», и только потом поднимается приложение. Healthcheck на брокере и ретраи консьюмера — обязательны. Для Postgres держите отдельный volume, чтобы не терять тестовые данные между перезапусками, но иногда полезно стартовать с чистой БД — добавьте make-цель «reset».

Старайтесь не разъезжаться по версиям. Простой чек-лист: версии образов в Compose совпадают с образами helm-чартов; переменные окружения указывают те же имена пользователей/баз; порты совпадают с прод-портами (если это безопасно). Тогда CI интеграционные тесты и локальные прогоны максимально предсказуемы.

И ещё: когда вам нужна «нестандартная» настройка, например TLS к Redis или аутентификация в Kafka — добавляйте её уже на локали. Не оставляйте «в бою включим». Потратите больше времени на настройку, но вернёте в надёжности.

Отдельный бонус — Testcontainers. Если Compose «тяжёлый» или конфликтует с текущими портами, тесты могут сами поднимать Postgres/Kafka из Java-кода. Архитектурно важно только одно: везде единые настройки соединений и таймаутов.

**Код (Compose: Postgres + Kafka + приложение)**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports: [ "5432:5432" ]
    healthcheck:
      test: ["CMD-SHELL","pg_isready -U app -d app"]
      interval: 5s
      timeout: 2s
      retries: 20

  kafka:
    image: bitnami/kafka:3.7
    environment:
      KAFKA_ENABLE_KRAFT: "yes"
      KAFKA_CFG_NODE_ID: "1"
      KAFKA_CFG_PROCESS_ROLES: "broker,controller"
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"
      KAFKA_CFG_LISTENERS: "PLAINTEXT://:9092,CONTROLLER://:9093"
      KAFKA_CFG_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:9092"
    ports: [ "9092:9092" ]
    healthcheck:
      test: ["CMD","bash","-lc","kafka-topics.sh --bootstrap-server kafka:9092 --list"]
      interval: 10s
      timeout: 5s
      retries: 10

  app:
    image: registry.local/demo:1.0.0
    depends_on:
      db: { condition: service_healthy }
      kafka: { condition: service_healthy }
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/app
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: app
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports: [ "8080:8080" ]
```

**Код (Java): минимальный Kafka consumer для smoke-теста)**

```java
package com.example.kafka;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class DemoConsumer {
  @KafkaListener(topics = "demo", groupId = "demo-grp")
  public void on(String msg) { System.out.println("Got: " + msg); }
}
```

**Код (Kotlin): тот же consumer)**

```kotlin
package com.example.kafka

import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class DemoConsumer {
    @KafkaListener(topics = ["demo"], groupId = "demo-grp")
    fun on(msg: String) { println("Got: $msg") }
}
```

---

## Makefile/скрипты: удобный билд/апдейты/миграции; экспорт логов и профили dev/test

Командная дисциплина экономит часы. Вместо длинных `docker compose ...` держите Makefile/скрипты: `make up`, `make logs`, `make reset-db`, `make migrate`. Так команде не нужно помнить десятки флагов, а CI использует те же скрипты, что и разработчики.

Миграции БД удобно запускать отдельной целью. Если Liquibase/Flyway встроены в приложение, делайте «headless» запуск (`--spring.main.web-application-type=none`). Для крупных баз лучше иметь отдельный образ/контейнер «мигратора».

Экспорт логов полезен при расследовании инцидентов на стенде: `docker compose logs --no-color > logs.txt` — банально, но снизит трение. Профили `dev/test` удобно включать через переменные Make: `make up PROFILE=test`.

Согласуйте имена целей с CI. Пайплайн будет банально вызывать `make build-image`/`make push`. Это уменьшает дрейф между локалью и конвейером и помогает держать «истину» в одном месте.

И не забывайте `clean`: снос томов и перезапуск «с нуля» часто самый быстрый путь избавиться от странных состояний. В Makefile это одна строка, которую приятно иметь под рукой.

**Код (Makefile)**

```makefile
PROJECT?=demo
PROFILE?=dev

build:
\t./gradlew clean bootJar

image:
\tdocker build -t registry.local/$(PROJECT):$(PROFILE) .

up:
\tdocker compose up -d

down:
\tdocker compose down

logs:
\tdocker compose logs -f --tail=200

reset-db:
\tdocker compose down -v db && docker compose up -d db

migrate:
\tdocker compose run --rm app java -jar /app/app.jar --spring.main.web-application-type=none
```

**Код (Java/Kotlin): запуск миграций как «headless» режима уже показан; дополнительных классов не требуется.**

---

# 9. Kubernetes/Helm: деплой и операционка

## Deployment/Service/Ingress: стратегия обновлений (RollingUpdate), readiness-пробы и минимальный healthy-под

В Kubernetes **Deployment** управляет ReplicaSet и стратегией обновлений. По умолчанию RollingUpdate меняет Pod’ы постепенно, сохраняя доступность. Настройте `maxUnavailable` и `maxSurge`, чтобы балансировать скорость и риск. Для «критичных» сервисов держите `minReadySeconds` — kube не будет считать Pod «готовым», пока он не пробыл Ready столько-то секунд. Это снижает флапы.

**Service** даёт стабильный виртуальный IP/DNS для Pod’ов, а **Ingress** — входной маршрут с TLS, лимитами и правилами. Если приложение отдаёт правильные пробы, трафик будет идти только в здоровые Pod’ы, остальным — 503. Readiness-пробы в проде важнее liveness: первые защищают пользователей, вторые — сам процесс.

Согласуйте тайминги: HTTP-таймауты клиента, Ingress, Service, приложение. Простой закон — длиннейший таймаут на самом дальнем слое не должен быть короче внутренних; иначе клиент бросит запрос раньше, чем бэкенд успеет ответить, и начнутся ретраи.

Не гонитесь за «минимальным» временем выкладки — куда важнее отсутствие 5xx и короткий spike latency. Для этого включайте `progressDeadlineSeconds`, следите за событиями Deployment и не стесняйтесь прерывать rollout при ошибках readiness.

Для тонкой дорожки отладки у Ingress есть аннотации на keep-alive/таймауты и буферы заголовков — не поленитесь их настроить, если вы отдаёте большие JWT и габаритные заголовки.

**Код (Helm Deployment/Service/Ingress, фрагменты шаблонов)**

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: {{ include "demo.fullname" . }} }
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  minReadySeconds: 10
  selector:
    matchLabels: { app: {{ include "demo.name" . }} }
  template:
    metadata:
      labels: { app: {{ include "demo.name" . }} }
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports: [ { containerPort: 8080 } ]
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 5
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
```

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata: { name: {{ include "demo.fullname" . }} }
spec:
  type: ClusterIP
  selector: { app: {{ include "demo.name" . }} }
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

```yaml
# templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "demo.fullname" . }}
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [ {{ .Values.host }} ]
      secretName: {{ .Values.tlsSecret }}
  rules:
    - host: {{ .Values.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "demo.fullname" . }}
                port: { number: 80 }
```

**Код (Java): эндпойнты readiness/liveness уже даны; добавим info)**

```java
package com.example.k8s;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class InfoController {
  @GetMapping("/info-lite")
  public String info() { return "ok"; }
}
```

**Код (Kotlin): то же)**

```kotlin
package com.example.k8s

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class InfoController {
    @GetMapping("/info-lite") fun info() = "ok"
}
```

---

## ConfigMap/Secret: монтирование, hot-reload конфигов; политика перезапуска после изменения

ConfigMap/Secret — два базовых «кирпича» внешней конфигурации. Их можно передавать как env или монтировать файлами. Файлы удобны для больших YAML и ключей/сертификатов; env — для «ключ-значение» простых параметров. Важно помнить: обновление ConfigMap/Secret **не меняет** содержимое уже запущенных Pod’ов автоматически, если не используется projected-volume с обновлением inode. На практике перезапускайте Pod’ы по изменению (например, через аннотацию-хэш в шаблоне pod’а).

«Горячая перезагрузка» конфигов в Java бывает, но в проде редко оправдана. Надёжнее перекатывать Pod’ы, чем ловить диффы конфигов в рантайме и гадать, что именно уже перечитано. Исключение — отдельные фичи/тогглы, где важна секундна реакция (см. feature-flags).

Secrets должны быть ограничены по скоупу и доступу RBAC; не кладите туда мегабайты. Для TLS-ключей используйте тип `kubernetes.io/tls`. Если у вас ротация сертификатов через cert-manager — держите короткий TTL и проверяйте автоматический reload.

Собственный паттерн: хранить только ссылку/путь на секрет в env, а содержимое — монтировать файлом с правами read-only. Это яснее и безопаснее, чем передавать секрет в env и случайно засветить его в /proc.

В Helm принято зашивать «checksum-аннотации» ресурсов конфигураций: при изменении ConfigMap/Secret рестартует Deployment. Это дешёвый и прозрачный способ гарантировать, что Pod всегда соответствует конфигу.

**Код (Helm, checksum-аннотации + монтирование Secret)**

```yaml
# templates/deployment.yaml (фрагмент)
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
spec:
  template:
    spec:
      volumes:
        - name: tls
          secret:
            secretName: {{ .Values.tlsSecret }}
      containers:
        - name: app
          volumeMounts:
            - name: tls
              mountPath: /etc/tls
              readOnly: true
          env:
            - name: SSL_PASSWORD
              valueFrom:
                secretKeyRef: { name: {{ .Values.tlsSecret }}, key: password }
```

**Код (Java): чтение смонтированного файла)**

```java
package com.example.cm;

import java.nio.file.Files;
import java.nio.file.Path;
import org.springframework.web.bind.annotation.*;

@RestController
class FileCfgController {
  @GetMapping("/cert-exists")
  public String certExists() {
    return Files.exists(Path.of("/etc/tls/tls.crt")) ? "yes" : "no";
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package com.example.cm

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.nio.file.Files
import java.nio.file.Path

@RestController
class FileCfgController {
    @GetMapping("/cert-exists") fun certExists() =
        if (Files.exists(Path.of("/etc/tls/tls.crt"))) "yes" else "no"
}
```

---

## HPA/автоскейл: метрики CPU/RAM/кастомные; PodDisruptionBudget, restartPolicy, topology spread

Horizontal Pod Autoscaler (HPA) — ваш «авточокер» по метрикам ресурсов. Стандартные источники — CPU/RAM, но через custom-metrics можно скейлить по RPS/очередям/лагу Kafka. Важно правильно выставить `requests`: HPA сопоставляет текущую загрузку с request, а не с «физическим» CPU. Слишком маленький request — и HPA будет думать, что всё 200% загружено, хотя процесс в порядке.

**PodDisruptionBudget (PDB)** ограничивает количество одновременно «вышибаемых» Pod’ов при эвакуациях/апдейтах нод. Это страховка от случайной потери кворума сервиса. `topologySpreadConstraints` распределяют Pod’ы по зонам/нодам, чтобы одно падение машины не уроняло весь сервис.

`restartPolicy` для Pod’ов в Deployment — всегда `Always`. Если у вас батч-задачи — используйте Job/CronJob. Не пытайтесь «воспроизводить» cron в приложении при наличии расписаний на уровне кластера — Kubernetes делает это лучше и надёжнее.

Для кастомных метрик используйте Prometheus Adapter или внешний источник (KEDA) — особенно удобно для событийных систем (Kafka): скейл по глубине топиков не привязан к CPU. Но обязательно держите «нижние» и «верхние» административные ограничения, чтобы сервис не ушёл в бесконечный рост.

Не злоупотребляйте автоскейлом: плохая конфигурация легко станет «усилителем» проблемы, если ваш сервис не выдерживает резких стартов/остановов (например, при холодном JIT). Прогревайте, держите минимальное число реплик >1 и включайте `startupProbe`.

**Код (HPA и PDB, YAML)**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: demo }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: demo-pdb }
spec:
  minAvailable: 1
  selector:
    matchLabels: { app: demo }
```

**Код (Java/Kotlin): не требуется специальных классов; автоскейл прозрачен для приложения.**

---

## Обновления без простоя: blue-green/canary; миграции БД как отдельные джобы; хранение артефактов Helm

Blue-green — это два Deployment (blue и green) и переключение Service/Ingress между ними. Canary — выпускаем новую версию на часть трафика и наблюдаем. Без специализированных контроллеров (Argo Rollouts) на Nginx Ingress можно делить трафик по весам через аннотации/доп. Ingress. Для большинства команд «ручной» blue-green уже сильно снижает риск, особенно когда в релизе есть миграции и хочется кнопки «откатить service назад».

Миграции БД выносите в Job. Это позволяет катить схему отдельно, проверять её, и только потом переводить трафик на новую версию приложения. В Helm удобно иметь оба артефакта: chart приложения и chart миграций (или один chart с включаемой Job). Храните релизы Helm (ChartMuseum/OCI-реестр) — это ускоряет откаты и делает поставку воспроизводимой.

Сервисные зависимости — тоже кандидаты на canary. Например, новый Kafka consumer с изменённой семантикой можно запустить в «тени» и сравнить результаты (shadow traffic). Это дороже по инфраструктуре, но драматически снижает риск.

Не забывайте про обратную совместимость схемы. Паттерн «expand/contract»: сначала добавили колонку/индекс, приложение стало писать и туда, затем во втором релизе — начали читать, и только потом удаляете старое. Такой подход делает откаты безболезненными.

Для Helm артефактов придерживайтесь иммутабельных тегов. Версия чарта ≠ версия приложения, но храните линк: values должны содержать образ `repo:tag`. GitOps-подходы (ArgoCD/Flux) любят такую прозрачность.

**Код (Helm Job для миграций)**

```yaml
# templates/job-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata: { name: {{ include "demo.fullname" . }}-migrate }
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrator
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          args: ["java","-jar","/app/app.jar","--spring.main.web-application-type=none"]
          envFrom:
            - secretRef: { name: {{ .Values.dbSecret }} }
```

**Код (Java): «feature-gate» для canary маршрута)**

```java
package com.example.canary;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/v2")
class CanaryController {
  private final boolean enabled;
  CanaryController(@Value("${feature.canary:false}") boolean enabled) { this.enabled = enabled; }

  @GetMapping("/calc")
  public String calc() { return enabled ? "new" : "disabled"; }
}
```

**Код (Kotlin): то же)**

```kotlin
package com.example.canary

import org.springframework.beans.factory.annotation.Value
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/v2")
class CanaryController(@Value("\${feature.canary:false}") private val enabled: Boolean) {
    @GetMapping("/calc") fun calc() = if (enabled) "new" else "disabled"
}
```

---

# 10. CI/CD, безопасность образов и продвижение релизов

## Pipeline: build → test → image → scan → sign → push → deploy; карантин/стадии promotion

Зрелый конвейер не заканчивается сборкой образа. После unit-/интеграционных тестов создаём OCI-образ, сканируем его на уязвимости, подписываем (supply chain), пушим в реестр и только потом — деплой. Хорошая практика — «карантин» (staging): образ в реестре помечается как «не прод» (по тегу/репозиторию), проходит автоматические тесты стенда, и лишь затем продвигается (promotion) в «prod» репозиторий или получает иммутабельный prod-тег.

Подписи (Cosign/Sigstore) и политика допуска в кластере (Kyverno/OPA) гарантируют, что в прод попадут только проверенные образы. Это снижает риск подмены артефактов и ошибки человека при ручном пуше. Важная деталь — управление ключами: используйте keyless или защищённые key-vault’ы, не храните ключи в CI как «секреты окружения» без аппаратной защиты.

Deployment должен быть идемпотентным шагом: вы описываете целевое состояние, а не выполняете «скрипт изменений». GitOps усиливает это правило: изменения в Git приводят к изменению кластера, а не наоборот. Это прозрачно и воспроизводимо.

«Promotion» удобно делать изменением только тега изображения в values Helm — без перекомпиляции чарта. Логику «кто и когда может продвигать» фиксируйте политикой: автомат на stage, человек (или автомат с gate тестами) — на prod.

Не забывайте про артефакты тестов/метрик конвейера: отчёты о качестве (coverage, mutation), результаты нагрузочного smoke и SLO-проверки — это не «доп. опция», а часть «зелёного света» для релиза.

**Код (GitHub Actions, упрощённый пример)**

```yaml
name: ci
on: [ push ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: 'temurin' }
      - name: Test
        run: ./gradlew clean test
      - name: Build image
        run: docker build -t registry.local/demo:${{ github.sha }} .
      - name: Scan
        run: trivy image --exit-code 1 --severity CRITICAL,HIGH registry.local/demo:${{ github.sha }}
      - name: Sign
        run: cosign sign --key ${{ secrets.COSIGN_KEY }} registry.local/demo:${{ github.sha }}
      - name: Push
        run: docker push registry.local/demo:${{ github.sha }}
```

**Код (Java/Kotlin): приложению безразлично, где оно собрано; специальных классов не требуется.**

---

## SBOM/подписи/сканирование уязвимостей; политика базовых образов и патч-окно

SBOM (Software Bill of Materials) — перечень зависимостей вашего артефакта. Его генерируют плагины (CycloneDX/Syft) и публикуют вместе с образом. Это упрощает аудит CVE и автоматическую корреляцию: когда выходит новая уязвимость, платформа мгновенно понимает, какие сервисы затронуты. В supply chain это «паспорт» вашего ПО.

Сканирование образа (Trivy/Grype) — не только «галочка». Введите политики «запретить CRITICAL/HIGH» и патч-окна: например, критичные — незамедлительно, high — до следующего релиза, medium — в плановую итерацию. Актуальность базовых образов — отдельная политика: еженедельный/ежемесячный пересбор образов поверх свежих баз, даже без изменений кода.

Подписи образов (Cosign) возвращают доверие: без подписи — доступ в кластер закрыт. Ключевой момент — валидация подписи Admission-контроллером и хранение публичных ключей в конфигурации кластера. Подписывайте и SBOM, и подпись привязывайте к digest образа, а не к тегу.

Документируйте исключения. Иногда нужно временно пропустить HIGH, чтобы закрыть критический бизнес-инцидент. Это должно быть явным, с тикетом/сроком. Так вы избежите «вечных исключений».

Сканируйте не только образы, но и Helm-чарты/манифесты (policies, misconfigurations) — тот же Trivy умеет проверять IaC на опасные default’ы.

**Код (Gradle: CycloneDX SBOM)**

```groovy
plugins { id 'org.cyclonedx.bom' version '1.9.0' }
cyclonedxBom { includeBomSerialNumber = true }
```

```kotlin
plugins { id("org.cyclonedx.bom") version "1.9.0" }
cyclonedxBom { includeBomSerialNumber.set(true) }
```

**Код (Java): печать версии и sha из BuildProperties — для трассируемости)**

```java
package com.example.sbom;

import org.springframework.boot.info.BuildProperties;
import org.springframework.web.bind.annotation.*;

@RestController
class VersionController {
  private final BuildProperties build;
  VersionController(BuildProperties build) { this.build = build; }
  @GetMapping("/version")
  public String v() { return build.getVersion() + " " + build.get("git.commit.id.abbrev"); }
}
```

**Код (Kotlin): то же)**

```kotlin
package com.example.sbom

import org.springframework.boot.info.BuildProperties
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class VersionController(private val build: BuildProperties) {
    @GetMapping("/version") fun v() = "${build.version} ${build["git.commit.id.abbrev"]}"
}
```

---

## Контроль версий и откаты: иммутабельные теги, хранение прошлых чартов/манифестов, быстрый rollback

Иммутабельные теги и хранение артефактов — основа быстрых откатов. Никогда не перезаписывайте теги вида `1.2.3`. Держите рядом «человеческие» метки (`1.2`, `1`) и «технические» (`git-sha`). В реестре храните достаточно историю, чтобы вернуться назад без пересборки.

Helm сам по себе хранит ревизии релиза в кластере; `helm rollback` — мгновенный возврат к прошлому состоянию. Но хранить чарт и values отдельно всё равно полезно: репозиторий артефактов (OCI/ChartMuseum) даст возможность пересобрать/перекатить тот же релиз в новый кластер.

При откатах учитывайте совместимость схемы БД (см. expand/contract). Если приложение уже начало писать в новую колонку, а вы вернулись назад — старый код может падать. Поэтому миграции должны быть двунаправленные по возможности, а откаты — проверены на стенде.

Сценарий «быстрый откат» должен быть частью runbook. Это не «на глаз» в консоли SRE, а записанная команда, понятные критерии, когда и кто её запускает, и как возвращаться обратно. «Промежуточный» откат (только Service указывает на старую ReplicaSet) бывает быстрее, чем полный rollback Helm — в зависимости от инструмента.

Наконец, следите за конфигурациями внешних прокси/ограничителей — откат приложения не всегда откатывает периметр. Храните и его конфиги версионированными, с тем же подходом «иммутабельности».

**Код (Helm, values с иммутабельными тегами)**

```yaml
image:
  repository: registry.example.com/demo
  tag: "1.4.2-gitabcdef0"
```

**Код (Java/Kotlin): уже показанные версии/health пригодятся для проверки после rollback; доп.кода не требуется.**

---

## Release-наблюдаемость: pre/post-deploy проверки SLO, dark-launch, feature-flags, circuit-breakers на периметре

В зрелом CI/CD релиз — это не только «положили yaml». Обязательны pre-checks (staging/tests) и post-deploy проверки: метрики ошибок, латентность, saturation. SLO-гейты могут блокировать продвижение: если 5xx > X% в течение N минут после релиза — автоматический rollback. Это дисциплинирует код и снижает MTTR.

Dark-launch — включение новой логики без маршрутизации внешнего трафика: вы можете активировать код на небольшой доле внутренних запросов, записывать метрики/логи и сравнивать с «старой» веткой. Feature-flags дают гибкость: конфигурируемое включение/отключение без пересборки. Храните флаги централизованно (например, Unleash/FF4J), с аудитом и TTL.

Circuit-breakers на периметре (API-шлюз/Ingress) и на уровне клиента (Resilience4j) защищают от каскадных отказов. При деградации зависимостей вы не «заливаете» их ретраями и даёте системе время восстановиться. Включайте бэкофы c джиттером и лимиты попыток.

После релиза сделайте целенаправленный smoke на ключевые пользовательские сценарии. Это не нагрузочный тест, а «дыхание системы». Часто он находит элементарные ошибки конфигурации раньше пользователей.

Все решения по откату/включению флагов должны оставлять «след» в логах/метриках: кто включил/выключил, в какое время, какой сегмент трафика попал под эксперимент. Это превращает наблюдаемость в реальный инструмент контроля релизов.

**Код (Java: Resilience4j, простой CB + retry)**

```java
package com.example.rel;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;

@Service
public class DownstreamService {
  @CircuitBreaker(name = "ds", fallbackMethod = "fallback")
  @Retry(name = "ds")
  public String call() {
    if (Math.random() < 0.7) throw new RuntimeException("boom");
    return "ok";
  }
  public String fallback(Throwable t) { return "fallback"; }
}
```

**Код (Kotlin): то же)**

```kotlin
package com.example.rel

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.retry.annotation.Retry
import org.springframework.stereotype.Service

@Service
class DownstreamService {
    @CircuitBreaker(name = "ds", fallbackMethod = "fallback")
    @Retry(name = "ds")
    fun call(): String {
        if (Math.random() < 0.7) throw RuntimeException("boom")
        return "ok"
    }
    fun fallback(t: Throwable) = "fallback"
}
```

`application.yml` (Resilience4j конфиг + actuator для наблюдаемости):

```yaml
resilience4j:
  circuitbreaker:
    instances:
      ds:
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
  retry:
    instances:
      ds:
        max-attempts: 3
        wait-duration: 200ms

management.endpoints.web.exposure.include: health,info,prometheus,metrics
```

