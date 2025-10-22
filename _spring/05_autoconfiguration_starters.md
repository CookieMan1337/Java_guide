---
layout: page
title: "Spring Boot «под капотом»: авто-конфигурация и стартеры"
permalink: /spring/context
---
# 0. Введение

## Что это

Авто-конфигурация Spring Boot — это набор правил и конфигурационных классов, которые автоматически создают бины и настраивают инфраструктуру приложения, ориентируясь на то, что находится в classpath и какие свойства заданы в среде. Иначе говоря, Boot «собирает» приложение из готовых кубиков, если видит нужные зависимости, и не трогает то, что вы определили вручную. Это каркас с разумными дефолтами и предсказуемыми точками выключения.

Ключ к магии — условия (`@Conditional*`). Они решают, включать ли конкретный автоконфиг: есть ли класс в classpath, активен ли веб-стек (servlet/reactive), включён ли флаг свойства. Поэтому один и тот же jar с кодом ведёт себя по-разному на разных окружениях — без ветвлений в вашем коде.

Стартеры — тонкие зависимости, которые подтягивают набор библиотек и сам «autoconfigure»-модуль. Подключили `spring-boot-starter-web` — и у вас уже есть Tomcat, `DispatcherServlet`, конвертеры, Jackson; подключили `spring-boot-starter-data-jpa` — и поднялись `EntityManagerFactory`, транзакции, `JpaRepository`.

Важно понимать, что авто-конфигурация — это инфраструктурный слой. Он не подменяет бизнес-логику и не вводит «скрытые» правила; он создаёт и связывает бины «проволоки»: источники данных, HTTP-клиенты, метрики, безопасность. Ваша предметная область остаётся «чистой» и зависит от интерфейсов/сервисов, а не от деталей wiring’а.

В Boot 3.x автоконфигурации регистрируются через файл `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Это делает процесс явным и пригодным для AOT/Native: генератор может заранее понять, что реально нужно приложению, и выбросить мёртвый код.

Разумеется, у любой «магии» должны быть рычаги управления. У Boot это: явное объявление своего `@Bean` (бэкоф дефолта), явный флаг свойства (`xxx.enabled=false`), и жёсткое исключение (`spring.autoconfigure.exclude`). Поэтому «под капотом» — предсказуемость: вы всегда сможете объяснить себе, почему появился конкретный бин.

**Код (Java): минимальное приложение + ручной бин, чтобы показать бэкоф**

```java
package intro.boot;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication // включает @EnableAutoConfiguration
public class App {
  public static void main(String[] args) {
    SpringApplication.run(App.class, args);
  }

  // Пользовательский ObjectMapper заставит Jackson-автоконфиг «отступить»
  @Bean
  public ObjectMapper objectMapper() {
    return new ObjectMapper().findAndRegisterModules();
  }
}
```

**Код (Kotlin): то же самое**

```kotlin
package intro.boot

import com.fasterxml.jackson.databind.ObjectMapper
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.context.annotation.Bean

@SpringBootApplication
class App {
    @Bean
    fun objectMapper(): ObjectMapper = ObjectMapper().findAndRegisterModules()
}
fun main(args: Array<String>) = runApplication<App>(*args)
```

Gradle (Groovy/Kotlin):

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  // Jackson приедет со стартером web
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
}
```

---

## Зачем это

Авто-конфигурация экономит время и снижает шум «проволоки». Вместо сотен строк фабрик и XML/Java-конфигов вы подключаете стартеры — и получаете рабочую инфраструктуру. Это ускоряет первые шаги и делает типовые решения (web, data, security, metrics) единообразными в разных командах.

Более того, это снижает риск ошибок. Дефолтные настройки, выбранные Boot, годами шлифовались сообществом: пулы, таймауты, кодеки, форматтеры — всё выставлено на «разумно безопасные» значения. Вы можете их переопределить, но базовая траектория — сразу продукционная.

Автоконфиг из коробки соблюдает 12-фактор: артефакт неизменяем, параметры — снаружи, поведение — через свойства/профили. Это даёт быстрые релизы и простые откаты: поменяли env — получили другую конфигурацию без пересборки.

Наконец, авто-конфигурация — стабильный контракт между платформенной/библиотечной командой и продуктовой. Первые поставляют «готовую интеграцию» (например, корпоративный HTTP-клиент с трейсингом и ретраями), вторые — просто подключают стартер и настраивают префикс `corp.http.*`. Это уменьшает разброс практик и облегчает поддержку.

С точки зрения тестирования выигрывает изоляция. Ваш доменный код не зависит от конкретного wiring’а — он получает уже сконструированные сервисы. Интеграционные тесты поднимают контекст ровно с теми автоконфигами, что нужны, а unit-тесты работают без Spring.

И, что не менее важно, авто-конфиги легко выключить. Не нужен MVC — исключили; нужен альтернативный `ObjectMapper` — объявили свой `@Bean`; нужна «тонкая настройка» — подкрутили свойства. Всё прозрачно и декларативно.

**Код (Java): показать, как просто переопределить дефолт MVC-настройку**

```java
package intro.why;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.servlet.config.annotation.*;

@SpringBootApplication
public class WhyApp {
  @Bean
  public WebMvcConfigurer corsConfigurer() {
    return new WebMvcConfigurer() {
      @Override
      public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**").allowedOrigins("https://app.example.com");
      }
    };
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package intro.why

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.context.annotation.Bean
import org.springframework.web.servlet.config.annotation.CorsRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@SpringBootApplication
class WhyApp {
    @Bean
    fun corsConfigurer() = WebMvcConfigurer {
        it.addCorsMappings(CorsRegistry().addMapping("/**").allowedOrigins("https://app.example.com"))
    }
}
```

Gradle:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

---

## Где используется

Авто-конфигурация покрывает едва ли не все слои типичного микросервиса: веб (MVC/WebFlux), данные (JPA/Jdbc/R2DBC), безопасность (Spring Security), интеграции (Kafka/AMQP), наблюдаемость (Actuator/Micrometer/OpenTelemetry), кэш (Caffeine/Redis), e-mail, шаблоны, сериализацию. Каждая область — отдельный автоконфиг или их семейство, подключаемые через соответствующий стартер.

В «клиентских» сервисах особенно ценны стартеры наблюдаемости: вы включаете зависимость — и у вас появляются health-пробы, метрики HTTP/DB, экспортер Prometheus, трассировка запросов. Это снимет с команды львиную долю инфраструктурных задач.

Во внутренних библиотеках компаний часто встречаются собственные стартеры: «корпоративный HTTP», «логирование по стандарту», «авторизация с IDP», «шаблон Kafka-продюсера». Это делает платформу самодокументируемой: достаточно открыть README стартера и настроить свойства.

В CLI/batch-утилитах авто-конфиги так же применимы: вы тянете только то, что нужно (например, `spring-boot-starter-jdbc`), и получаете готовый DataSource с миграциями Flyway. Никаких веб-серверов и лишних бинов — условия не сработают, и они не будут созданы.

В монорепозиториях с десятками микросервисов единообразие конфигов — критично. Стартеры — лучший способ «зашить» конвенции и лучшие практики внутрь зависимостей, а не в устные договорённости.

Даже в десктопных/встраиваемых сценариях (plain Boot без web) автоконфиги полезны: они дают `ConversionService`, биндинг конфигов, события старта/остановки, scheduler — и всё это можно включать/выключать свойствами.

**Код (Java): добавили наблюдаемость «одной строкой»**

```java
// build.gradle
/*
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
}
*/
// application.yml
/*
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
*/
```

**Код (Kotlin): тот же набор зависимостей (Gradle Kotlin)**

```kotlin
// build.gradle.kts
/*
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  runtimeOnly("io.micrometer:micrometer-registry-prometheus")
}
*/
```

---

## Какие проблемы и задачи решает

Первая боль, которую снимает Boot, — «церемония» конфигурации. Вручную создавать `DataSource`, `EntityManagerFactory`, `TransactionManager`, прописывать `@EnableWebMvc`, регистрировать конвертеры/кодеки — долго и подвержено ошибкам. Авто-конфиги делают это за вас и одинаково для всех приложений.

Вторая — согласованность. Когда команды на одном стеке, различия должны сводиться к бизнес-коду, а не к тому, как подключены метрики или выставлены таймауты. Стартеры задают стандарт, который можно точечно переопределять.

Третья — скорость доставки. Чем меньше вы пишете инфраструктурного кода, тем быстрее получаете рабочий сервис. Это особенно чувствуется в прототипировании и при масштабировании команды.

Четвёртая — предсказуемость поведения. Дефолты Boot годятся для продакшна, а условия включения/выключения прозрачны. Даже если что-то «поднялось не так», отчёт условий покажет причину, и вы быстро исправите конфиг.

Пятая — разделение ответственности. Платформенная команда отдаёт стартер, продуктовые — подключают и задают свойства. Никаких копипаст и «снежинок», меньше рисков при апгрейдах.

Шестая — поддержка AOT/Native. Тонкие автоконфиги, условия и хинты позволяют собирать нативные образы быстро и с минимальной памятью. Это новая база для быстрых CLI и «холодного» старта в FaaS.

**Код (конфиг-пример): мгновенное выключение ненужного автоконфига**

```yaml
# application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

**Код (Java/Kotlin): ручное перекрытие бина — дефолт отступает**

```java
package intro.solve;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestClient;

@SpringBootApplication
public class SolveApp {
  @Bean RestClient restClient() { return RestClient.create(); } // перекрываем дефолт
}
```

```kotlin
package intro.solve

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.context.annotation.Bean
import org.springframework.web.client.RestClient

@SpringBootApplication
class SolveApp {
    @Bean fun restClient(): RestClient = RestClient.create()
}
```

Gradle:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

---

# 1. Что такое авто-конфигурация и как она включается

## Роль `@SpringBootApplication` и `@EnableAutoConfiguration`: сборка приложения из «кубиков» в зависимости от classpath и настроек

Аннотация `@SpringBootApplication` — это композиция трёх: `@Configuration` (класс конфигурации), `@ComponentScan` (сканирование компонентов) и `@EnableAutoConfiguration` (включение механизма автоконфигов). Последняя и даёт зелёный свет всей «магии»: Boot найдёт список доступных автоконфигураций в classpath и применит те, чьи условия выполнились.

`@EnableAutoConfiguration` не жёстко «подключает всё» — она включает процессор, который читает ресурсы `AutoConfiguration.imports` и последовательно пытается применить каждый конфиг. На каждом шаге срабатывают условия: есть ли необходимые классы, активен ли web-стек, нет ли уже пользовательского бина и т.д.

Если вы объявляете свой `@Bean` того же типа/имени, что и дефолт, автоконфиг «отступает» — это правило бэкофа. Таким образом Boot никогда молча не перекроет ваш бин, а вы — можете перекрыть его дефолт.

В редких случаях полезно отключить конкретный автоконфиг целиком: для этого `@SpringBootApplication(exclude = …)` или свойство `spring.autoconfigure.exclude`. Это крайняя мера, обычно достаточно флага `feature.enabled=false` или объявления своего бина.

Разрешение автоконфигов зависит от classpath. Если у вас нет `spring-boot-starter-web`, MVC-автоконфиги не активируются; если есть `webflux`, поднимется реактивная ветка. Это даёт тонкую модулярность: вы тянете только нужные куски, остальное остаётся «мертвым».

Наконец, порядок сканирования пользовательских компонентов и автоконфигов продуман так, чтобы ваши бины были видны условиям (например, `@ConditionalOnMissingBean`) — и дефолты не создавались поверх. Это позволяет свободно заменять части инфраструктуры без «борьбы» с Boot.

**Код (Java): включение/исключение автоконфигов**

```java
package ac.on;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration;
import org.springframework.boot.SpringApplication;

@SpringBootApplication(exclude = { WebMvcAutoConfiguration.class }) // пример исключения
public class OnApp {
  public static void main(String[] args) { SpringApplication.run(OnApp.class, args); }
}
```

**Код (Kotlin): аналог**

```kotlin
package ac.on

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
import org.springframework.boot.runApplication

@SpringBootApplication(exclude = [WebMvcAutoConfiguration::class])
class OnApp
fun main(args: Array<String>) = runApplication<OnApp>(*args)
```

Gradle:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

## Идея «разумных дефолтов с бэкофом»: автоконфиг создаёт бины только если вы их не определили сами

Принцип бэкофа — столп доверия к Boot. Каждый бин в автоконфиге почти всегда помечен `@ConditionalOnMissingBean`: если пользователь уже объявил совместимый тип, дефолт не создаётся. Это избавляет от «драк» за контекст и делает поведение очевидным.

Такой подход стимулирует переопределять точечно. Хотите особый `ObjectMapper` — объявите свой; хотите другой `RestClient` — создайте бин с нужными таймаутами; хотите заменить кэш — подключите другой стартер/реализацию. Остальная часть стека останется прежней.

Бэкоф работает и на уровне целых автоконфигов: условие `@ConditionalOnClass` просто не даст им активироваться без нужных зависимостей. Это защищает от «случайных» бинов в контексте, если вы подтянули библиотеку «мимо кассы».

Ещё одна грань — явные «выключатели» через свойства. Автоконфиги часто добавляют `xxx.enabled` и включают тяжёлые компоненты только по флагу. Это делает поведение управляемым на окружениях без пересборки.

Стоит помнить, что «ручной» `@Bean` часто проще и надёжнее, чем рефакторинг дефолтного бина через свойства. Если меняется более одной-двух опций, вероятно, пора объявить свой бин и дать автоконфигу отступить.

Наконец, принцип бэкофа — договор. Пишете собственные автоконфиги — обязательно ставьте `@ConditionalOnMissingBean` на создаваемые бины, иначе вторжение в пользовательский граф станет неожиданным.

**Код (Java): пользовательский бин перекрывает дефолт RestClient**

```java
package ac.backoff;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestClient;

@SpringBootApplication
public class BackoffApp {
  @Bean
  RestClient customRestClient() {
    return RestClient.builder().baseUrl("https://api.example.com").build();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package ac.backoff

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.context.annotation.Bean
import org.springframework.web.client.RestClient

@SpringBootApplication
class BackoffApp {
    @Bean
    fun customRestClient(): RestClient =
        RestClient.builder().baseUrl("https://api.example.com").build()
}
fun main(args: Array<String>) = runApplication<BackoffApp>(*args)
```

Gradle:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

---

## Где лежит список авто-конфигураций: файл `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` в артефактах автоконфига

С Boot 2.7+/3.x список автоконфигов перестал собираться через «старые» `spring.factories` и хранится в отдельном файле `AutoConfiguration.imports`. Каждый autoconfigure-jar содержит его в `META-INF/spring/` и перечисляет FQCN классов с `@AutoConfiguration`. Это ускоряет старт и делает регистрацию явной для AOT.

Вы можете заглянуть в этот файл прямо из кода/тестов и увидеть, какие автоконфиги приносит зависимость. Это удобно при диагностике «почему появился бин X» и при ревью сторонних стартеров.

Правило простое: **стартер** (например, `spring-boot-starter-web`) тянет **autoconfigure-модуль** (`spring-boot-autoconfigure`) и нужные библиотеки (Jackson, Tomcat и т.д.). Сам список автоконфигов лежит именно в autoconfigure-артефакте.

При написании собственного автоконфига вы тоже добавляете такой файл и указываете там свои классы с `@AutoConfiguration`. Тогда `@EnableAutoConfiguration` увидит их и применит при старте.

Полезно помнить, что один артефакт может включать десятки автоконфигов, но условия «ограничат» их до реально нужных для вашего приложения. Если сомневаетесь — включайте отчёт условий (см. следующую подтему) и смотрите, что сработало.

Для библиотеки важно документировать, какие именно бины и при каких условиях вы регистрируете. Тогда по `AutoConfiguration.imports` и README пользователь сразу понимает границы влияния на контекст.

**Код (Java): выводим содержимое всех `AutoConfiguration.imports` из classpath**

```java
package ac.list;

import org.springframework.boot.CommandLineRunner;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.stereotype.Component;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;

@Component
public class ImportsPrinter implements CommandLineRunner {
  @Override
  public void run(String... args) throws Exception {
    var resolver = new PathMatchingResourcePatternResolver();
    var resources = resolver.getResources("classpath*:/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports");
    for (var res : resources) {
      System.out.println("== " + res.getURL());
      try (var br = new BufferedReader(new InputStreamReader(res.getInputStream(), StandardCharsets.UTF_8))) {
        br.lines().filter(l -> !l.isBlank() && !l.startsWith("#")).forEach(System.out::println);
      }
    }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package ac.list

import org.springframework.boot.CommandLineRunner
import org.springframework.core.io.support.PathMatchingResourcePatternResolver
import org.springframework.stereotype.Component
import java.io.BufferedReader
import java.io.InputStreamReader
import java.nio.charset.StandardCharsets

@Component
class ImportsPrinter : CommandLineRunner {
    override fun run(vararg args: String?) {
        val resolver = PathMatchingResourcePatternResolver()
        val resources = resolver.getResources("classpath*:/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports")
        resources.forEach { res ->
            println("== ${res.url}")
            BufferedReader(InputStreamReader(res.inputStream, StandardCharsets.UTF_8)).use { br ->
                br.lines().filter { it.isNotBlank() && !it.startsWith("#") }.forEach { println(it) }
            }
        }
    }
}
```

Gradle:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

# 2. Жизненный цикл авто-конфигураций при старте

## Фазы: сбор `AutoConfiguration.imports` → применение условий → регистрация бинов → post-processors → готовый контекст

Старт Boot проходит несколькими чёткими фазами. Сначала формируется `Environment` и загружаются конфиги (Config Data API). Затем `@EnableAutoConfiguration` собирает все `AutoConfiguration.imports` и строит список кандидатов. На этом этапе ничего ещё не создано — только «планы».

Далее для каждого кандидата оцениваются условия: класс в classpath, профиль, `@ConditionalOnBean/OnMissingBean`, флаги свойств. Положительный матч — конфигурация допускается к регистрации, отрицательный — выбывает. Это экономит время/память: ненужные куски даже не доходят до создания бинов.

Следом идёт регистрация бинов из «пропущенных» автоконфигов и пользовательских конфигураций. Здесь работают `BeanFactoryPostProcessor` и `BeanDefinitionRegistryPostProcessor` — они могут модифицировать дефиниции до реального создания объектов (например, добавлять прокси, валидировать граф).

Когда начинается создание singleton-бинов, вступают `BeanPostProcessor` (AOP, валидация, биндинг конфигов). Именно здесь появятся прокси для транзакций/кэша/трассировки. Порядок важен: сначала инфраструктура, затем ваши сервисы.

После инициализации всех singleton-бинов публикуется `ApplicationStartedEvent`, затем, когда веб-сервер готов принимать трафик, — `ApplicationReadyEvent`. На этих хуках удобно делать прогревы и раннеры.

При остановке — обратная последовательность: `ContextClosedEvent`, `SmartLifecycle.stop` и вызовы `@PreDestroy`. Автоконфиги, как и ваши бины, должны корректно завершать фоновые задачи, закрывать пулы, сбрасывать буферы.

**Код (Java): логируем фазы старта «снаружи»**

```java
package ac.lifecycle;

import org.springframework.boot.context.event.*;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class PhaseLogger {
  @EventListener(ApplicationStartingEvent.class)    public void s(ApplicationStartingEvent e){ System.out.println("Phase: starting"); }
  @EventListener(ApplicationEnvironmentPreparedEvent.class) public void ep(ApplicationEnvironmentPreparedEvent e){ System.out.println("Phase: env prepared"); }
  @EventListener(ApplicationPreparedEvent.class)    public void p(ApplicationPreparedEvent e){ System.out.println("Phase: context prepared"); }
  @EventListener(ApplicationStartedEvent.class)     public void st(ApplicationStartedEvent e){ System.out.println("Phase: started"); }
  @EventListener(ApplicationReadyEvent.class)       public void r(ApplicationReadyEvent e){ System.out.println("Phase: ready"); }
}
```

**Код (Kotlin): аналог**

```kotlin
package ac.lifecycle

import org.springframework.boot.context.event.*
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class PhaseLogger {
    @EventListener(ApplicationStartingEvent::class) fun s(e: ApplicationStartingEvent) { println("Phase: starting") }
    @EventListener(ApplicationEnvironmentPreparedEvent::class) fun ep(e: ApplicationEnvironmentPreparedEvent) { println("Phase: env prepared") }
    @EventListener(ApplicationPreparedEvent::class) fun p(e: ApplicationPreparedEvent) { println("Phase: context prepared") }
    @EventListener(ApplicationStartedEvent::class) fun st(e: ApplicationStartedEvent) { println("Phase: started") }
    @EventListener(ApplicationReadyEvent::class) fun r(e: ApplicationReadyEvent) { println("Phase: ready") }
}
```

Gradle:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

## Отчёт условий (Condition Evaluation Report): как понять, почему конкретная авто-конфигурация сработала/не сработала

Boot умеет объяснять свои решения. Если включить `debug: true` или поднять лог-уровень для пакета условий, при старте вы увидите отчёт «Positive/Negative matches»: какие автоконфиги прошли, какие — нет, и по какой причине. Это главный инструмент диагностики «куда делся бин?» или «откуда он взялся?».

Отчёт полезен и для ревью. Вы обновили зависимость — и вдруг появился новый фильтр в цепочке. Откройте отчёт: возможно, в библиотеке добавили автоконфиг, условие `OnClass` сработало, и теперь в контексте новый бин. Так вы быстро решите, выключить его флагом или переопределить.

Ещё один способ — получить отчёт программно и вывести компактно в логи. Это уместно в dev/stage, когда нужно «подсветить» активные автоконфиги для службы поддержки. Главное — не забыть выключить «болтливость» на проде.

Помните, что отчёт относится к моменту построения контекста. Если вы затем вручную регистрируете бины — в отчёте это не отразится. Но для авто-конфигов и условий он исчерпывающий.

Для тонкой настройки логов используйте `logging.level.org.springframework.boot.autoconfigure.condition=DEBUG` — это печатает причины матчей, не включая прочий `debug`. На сложных приложениях это заметно уменьшает шум.

И наконец, договоритесь в команде: при непонятном старте первым делом включаем отчёт условий и `/actuator/conditions` (если доступен), прикладываем к тикету — и только потом разбираем код.

**Код (Java): выводим отчёт условий в конце старта**

```java
package ac.report;

import org.springframework.boot.autoconfigure.condition.ConditionEvaluationReport;
import org.springframework.context.ApplicationContext;
import org.springframework.context.event.EventListener;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.stereotype.Component;

@Component
public class ConditionsDumper {
  private final ApplicationContext ctx;
  public ConditionsDumper(ApplicationContext ctx){ this.ctx = ctx; }

  @EventListener(ApplicationReadyEvent.class)
  public void dump() {
    var report = ConditionEvaluationReport.get(ctx.getAutowireCapableBeanFactory());
    report.getConditionAndOutcomesBySource().forEach((source, outcomes) -> {
      boolean matched = outcomes.isFullMatch();
      System.out.println((matched ? "[+]" : "[-]") + " " + source + " :: " + outcomes);
    });
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package ac.report

import org.springframework.boot.autoconfigure.condition.ConditionEvaluationReport
import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.ApplicationContext
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class ConditionsDumper(private val ctx: ApplicationContext) {
    @EventListener(ApplicationReadyEvent::class)
    fun dump() {
        val report = ConditionEvaluationReport.get(ctx.autowireCapableBeanFactory)
        report.conditionAndOutcomesBySource.forEach { (source, outcomes) ->
            val matched = outcomes.isFullMatch
            println("${if (matched) "[+]" else "[-]"} $source :: $outcomes")
        }
    }
}
```

`application.yml` (включить детальный отчёт):

```yaml
debug: true
logging:
  level:
    org.springframework.boot.autoconfigure.condition: DEBUG
```

---

## Диагностика: включение подробных логов условий, просмотр причин «позитивных/негативных» матчей

Подробные логи условий — ваш «рентген» старта. Включив `DEBUG` на пакете `org.springframework.boot.autoconfigure.condition`, вы увидите, какие именно `@Conditional…` вернули true/false: найден ли класс, присутствует ли бин, какое значение у свойства. Это ускоряет поиск «ломающих» зависимостей и конфликтов кандидатов.

Если отчёт слишком большой, фильтруйте его ключевыми словами (например, по имени интересующего автоконфига). Полезно и временно исключать подозрительные автоконфиги через `spring.autoconfigure.exclude`, чтобы проверить гипотезу без правки кода.

Для библиотек/стартеров готовьте «микро-сценарии» с `ApplicationContextRunner` (см. подтему 8 позже): он печатает мини-отчёты и позволяет проверять условия точечно, без запуска всего приложения. Это быстрее, чем перебирать комбинации профилей свойствами.

Отдельно настройте `/actuator/conditions` и `/actuator/beans` в dev/stage: это безопасный способ посмотреть реальный контекст онлайн. Выведите ссылки в README сервиса: поддержке так проще.

Не забывайте выключать «болтливый» DEBUG на проде. Он мешает анализу инцидентов и расходует I/O. Оставьте INFO/ERROR, а условия включайте по требованию (feature-flag в конфиге) и только на репродукции.

И ещё одна практика: логируйте «своё». Если вы пишете автоконфиг, оставьте краткое сообщение при включении/выключении (на уровне DEBUG) и перечислите ключевые свойства. Это становится частью отчёта и помогает интеграторам.

**Код (Java/Kotlin): точечная настройка лог-уровня через свойства**

```yaml
logging:
  level:
    org.springframework.boot.autoconfigure.condition: DEBUG
    com.mycompany.mystarter: DEBUG
management:
  endpoints.web.exposure.include: conditions,beans
```

---

# 3. Условия авто-конфигурации (Conditional*)

## Базовые: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnBean`, `@ConditionalOnProperty`

Основная четвёрка условий покрывает 90% сценариев. `@ConditionalOnClass` — активируйся, если класс есть в classpath (например, включаем интеграцию с Redis, только если зависимость присутствует). `@ConditionalOnMissingBean` — создай дефолтный бин, если пользователь не объявил свой. `@ConditionalOnBean` — включись, только если уже есть другой бин (например, декоратор поверх клиента). `@ConditionalOnProperty` — управляем включением/выключением флагом.

Комбинации условий разрешены и обычны: «если есть RestClient *и* включён флаг *и* нет пользовательского бина — создать наш клиент с метриками». Важно не перегружать логику: лучше раздробить конфиг на несколько классов с понятными зонами ответственности.

Всегда помните про «бэкоф»: любые создаваемые бины должны быть под `@ConditionalOnMissingBean`. Это гарантия, что пользователь может переопределить поведение без форка вашего стартера.

`@ConditionalOnProperty` удобно снабжать разумным дефолтом: тяжёлые вещи — выключены по умолчанию (`enabled=false`), лёгкие — включены (`enabled=true`). Документируйте поведение в метаданных и README.

`@ConditionalOnBean` полезно использовать для «надстроек»: если есть `DataSource`, регистрируем `JdbcTemplate`; если есть `MeterRegistry`, навешиваем метрики. Так вы не создадите «висящие» бины без инфраструктуры.

Тестируйте условия изолированно: `ApplicationContextRunner` позволит имитировать наличие/отсутствие классов/бинов/свойств и проверить, что ваши конфиги ведут себя корректно.

**Код (Java): минимальный автоконфиг с базовыми условиями**

```java
package cond.basic;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;

public interface Greeting { String hi(String name); }

@AutoConfiguration
@ConditionalOnClass(name = "org.springframework.web.client.RestClient")
@ConditionalOnProperty(prefix = "greeting", name = "enabled", havingValue = "true", matchIfMissing = true)
public class GreetingAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean
  public Greeting defaultGreeting() {
    return name -> "Hello, " + name;
  }

  @Bean
  @ConditionalOnBean(Greeting.class)
  public Greeting decorated(Greeting base) { // пример «надстройки»
    return name -> "[v1] " + base.hi(name);
  }
}
```

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
cond.basic.GreetingAutoConfiguration
```

**Код (Kotlin): аналог**

```kotlin
package cond.basic

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean

fun interface Greeting { fun hi(name: String): String }

@AutoConfiguration
@ConditionalOnClass(name = ["org.springframework.web.client.RestClient"])
@ConditionalOnProperty(prefix = "greeting", name = ["enabled"], havingValue = "true", matchIfMissing = true)
class GreetingAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    fun defaultGreeting(): Greeting = Greeting { name -> "Hello, $name" }

    @Bean
    @ConditionalOnBean(Greeting::class)
    fun decorated(base: Greeting): Greeting = Greeting { name -> "[v1] ${base.hi(name)}" }
}
```

Gradle (добавьте autoconfigure-модуль в зависимости приложения):

```groovy
dependencies { implementation project(':my-autoconfigure') }
```

```kotlin
dependencies { implementation(project(":my-autoconfigure")) }
```

---

## Среда выполнения: `@ConditionalOnWebApplication` (servlet/reactive), `@ConditionalOnCloudPlatform`, профили

Иногда поведение зависит от «типа» приложения. `@ConditionalOnWebApplication(type = SERVLET)` — только для MVC; `type = REACTIVE` — только для WebFlux. Это важно для фильтров, кодеков, Interceptor’ов, которые не имеют смысла в другом стеке.

`@ConditionalOnCloudPlatform` позволяет включать конфиги только в конкретном облаке (Cloud Foundry, Heroku, Kubernetes). В Boot 3.x есть распознавание среды, но для Kubernetes чаще используют профиль/свойство вместо этого условия, чтобы не завязываться на эвристику.

Профили (`@Profile`) — ещё один измеритель. Бины для `dev`/`prod`, заглушки для `test`, особые политики для `eu/us`. В автоконфиге их применяют редко (чаще в приложении), но для корпоративных стартеров это способ аккуратно «достраивать» поведение.

Комбинации: «включить логирование доступа только в servlet-приложении *и* только если `feature.accesslog.enabled=true`», «поднять реактивный клиент только при `webflux` и наличии `ReactorNetty`». Условия читаются сверху вниз; старайтесь, чтобы каждое было самостоятельным и простым.

Проверяйте отсутствие «ложных матчей». Если вы смотрите на класс, который может попасть транзитивно, ограничьте условие и по флагу свойства. Тогда библиотека в classpath не приведёт к неожиданным включениям.

Не забывайте про «бэкоф»: даже если условия для среды прошли, бины должны уважать пользовательские переопределения.

**Код (Java): условие по типу веб-приложения + флаг**

```java
package cond.env;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.web.servlet.HandlerInterceptor;

@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnProperty(prefix = "accesslog", name = "enabled", havingValue = "true")
public class AccessLogAutoConfig {

  @Bean
  @ConditionalOnMissingBean
  public HandlerInterceptor accessLogInterceptor() {
    return new HandlerInterceptor() { /* ... */ };
  }
}
```

**Код (Kotlin): аналог для WebFlux**

```kotlin
package cond.env

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication
import org.springframework.context.annotation.Bean
import org.springframework.web.server.WebFilter

@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.REACTIVE)
@ConditionalOnProperty(prefix = "accesslog", name = ["enabled"], havingValue = "true")
class AccessLogReactiveAutoConfig {
    @Bean
    @ConditionalOnMissingBean
    fun accessLogFilter(): WebFilter = WebFilter { exchange, chain -> chain.filter(exchange) }
}
```

`application.yml`:

```yaml
accesslog:
  enabled: true
```

---

## Правила «бэкофа»: любые создаваемые бины — под `@ConditionalOnMissingBean`, явные точки выключения — через `@ConditionalOnProperty`

Золотое правило автора автоконфига: **никогда** не регистрируйте бин без `@ConditionalOnMissingBean`, если он может пересечься с пользовательским. Иначе вы рискуете «забетонировать» поведение и лишить приложение гибкости.

Второе правило — добавляйте «выключатели». Если компонент тяжёлый (фоновая нитка, большой кэш, сеть), пусть по умолчанию он выключен или включает минимум поведения, а полное включение — только флагом `xxx.enabled=true`. Это защищает от сюрпризов на проде.

Третье — делайте условия максимально узкими: «есть класс *и* задан флаг *и* есть инфраструктурный бин». Тогда дефолт не появится «сам по себе» в неподходящей среде.

Четвёртое — имена бинов/квалификаторы. Если ваш бин потенциально конкурирует с «общим» типом, дайте ему понятное имя и документируйте, как переопределить. В коллекциях (`List<T>`) порядок задавайте `@Order`, чтобы пользователь мог вставить свою реализацию на нужную позицию.

Пятое — не забывайте про метаданные свойств: IDE-подсказки и javadoc снижают риск неправильного включения/конфигурации. Пишите «что делает флаг» и «каков дефолт» прямо в классе свойств.

Шестое — тесты бэкофа: проверяйте кейс «пользователь объявил свой бин» — ваш автоконфиг должен промолчать.

**Код (Java): пример «вежливого» включателя**

```java
package cond.backoff;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;

public interface AuditSink { void emit(String msg); }

@AutoConfiguration
@ConditionalOnProperty(prefix = "audit", name = "enabled", havingValue = "true", matchIfMissing = false)
public class AuditAutoConfig {

  @Bean
  @ConditionalOnMissingBean
  AuditSink stdOutAudit() { return msg -> System.out.println("AUDIT " + msg); }
}
```

**Код (Kotlin): тестовый оверрайд бина (пользовательский `@Bean`)**

```kotlin
package cond.backoff

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

interface AuditSink { fun emit(msg: String) }

@Configuration
class UserOverrideConfig {
    @Bean
    fun auditSink(): AuditSink = AuditSink { /* отправка в Kafka */ }
}
```

Gradle:

```groovy
dependencies { implementation project(':my-autoconfigure') }
```

```kotlin
dependencies { implementation(project(":my-autoconfigure")) }
```
# 4. Порядок применения и зависимость авто-конфигураций

## `@AutoConfiguration(before/after = …)` и `@AutoConfigureOrder`: управление очередностью, когда есть перекрытия

Порядок применения авто-конфигураций важен в случаях, когда одна конфигурация полагается на бины, созданные другой, или должна выполниться раньше, чтобы подменить дефолты. В Spring Boot это решается двумя механизмами: атрибутами `before/after` у `@AutoConfiguration` и аннотацией `@AutoConfigureOrder`. Первый задаёт относительные зависимости между конкретными классами автоконфигов, второй — числовой приоритет наподобие `@Order` (меньше — раньше).
Когда вы контролируете оба конца зависимости (свои автоконфиги), предпочтительнее использовать `before/after`, так как это декларативная, читаемая связь «кто перед кем». Числовые приоритеты уместны в общих библиотеках, где вы не знаете имён потенциальных соседей, но хотите располагаться «пораньше» или «позже» в общем списке.
Частая задача — расширить настройки существующего дефолта: например, добавить перехватчик в `RestClient`. Правильный подход — выполнить ваш конфиг «после» стандартного `RestClient`-конфига, чтобы иметь готовый бин для декорирования. Обратная ситуация — вы хотите заблокировать «лишние» дефолты, тогда ваш конфиг должен выполниться «до» и выставить пользовательский бин, из-за чего дефолт «отступит».
Важно помнить, что порядок влияет только на регистрацию дефиниций бинов, а не на фактическое создание singleton-ов: создание происходит позже согласно обычным правилам контейнера. Тем не менее, если ваш конфиг зависит от уже зарегистрированных `BeanDefinition`, задайте `after` и используйте условия `@ConditionalOnBean`.
Ошибкой будет «силовой» порядок без условий. Даже если вы выполнитесь раньше, но не проверяете наличие пользовательского бина, вы можете создать нежелательный дефолт. Поэтому порядок всегда идёт в паре с условиями бэкофа (`OnMissingBean`).
Отлаживайте порядок через отчёт условий и, при необходимости, с помощью минимальных тестов на `ApplicationContextRunner`: так вы увидите, какой конфиг применился первым, и как повлиял на доступность бинов для последующих конфигов.

**Код (Java): автоконфиг, который должен выполниться после стандартного**

```java
package order.sample;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.AutoConfigureOrder;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.core.Ordered;
import org.springframework.web.client.RestClient;

@AutoConfiguration(after = org.springframework.boot.autoconfigure.web.client.RestClientAutoConfiguration.class)
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE) // на всякий случай позже большинства
public class RestClientDecoratorAutoConfiguration {

  @Bean
  @ConditionalOnBean(RestClient.class)
  @ConditionalOnMissingBean(name = "metricsRestClient")
  RestClient metricsRestClient(RestClient base) {
    return RestClient.builder().baseUrl("https://metrics-wrapper")
        .requestFactory(base::getRequestFactory).build();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package order.sample

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.AutoConfigureOrder
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.core.Ordered
import org.springframework.context.annotation.Bean
import org.springframework.web.client.RestClient

@AutoConfiguration(after = [org.springframework.boot.autoconfigure.web.client.RestClientAutoConfiguration::class])
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
class RestClientDecoratorAutoConfiguration {
    @Bean
    @ConditionalOnBean(RestClient::class)
    @ConditionalOnMissingBean(name = ["metricsRestClient"])
    fun metricsRestClient(base: RestClient): RestClient =
        RestClient.builder().requestFactory(base::getRequestFactory).build()
}
```

Gradle (Groovy/Kotlin) — модуль с автоконфигом:

```groovy
dependencies {
  api 'org.springframework.boot:spring-boot-autoconfigure'
}
```

```kotlin
dependencies {
  api("org.springframework.boot:spring-boot-autoconfigure")
}
```

---

## Дробление на несколько конфиг-классов по зонам ответственности: web-слой, клиенты, data-слой, наблюдаемость

Монолитные автоконфиги на сотни строк тяжело читать, тестировать и развивать. Хорошая практика — дробить по функциональным «срезам»: отдельный класс для web (фильтры, кодеки), отдельный для клиентов (RestClient, gRPC), отдельный для data (DataSource, репозитории), отдельный для наблюдаемости (MeterRegistry, обвязки). Это улучшает локализацию условий и облегчает выбор порядка между независимыми частями.
Каждый такой класс получает собственные условия (`OnClass`, `OnWebApplication`, `OnBean`) и не знает о «соседях» напрямую. Если нужно «соединить» слои, используйте `@ConditionalOnBean`(инфраструктурный бин) вместо прямых вызовов методов соседней конфигурации. Так вы избегаете жёсткой сцепки и облегчаете переиспользование кусков.
Дробление также помогает при AOT/Native: ненужные классы не попадут в итоговый список, если условия не сработают. Мелкие конфиги быстрее анализируются и точнее описывают зависимость от среды.
С точки зрения тестов вы получите маленькие изолированные сценарии: web-конфиг можно протестировать без базы, а data-конфиг — без веба. Это ускоряет CI и делает отчёт условий компактнее.
При именовании придерживайтесь ясности: `<Feature>WebAutoConfiguration`, `<Feature>ClientAutoConfiguration`, `<Feature>DataAutoConfiguration`, `<Feature>ObservabilityAutoConfiguration`. Такой паттерн сразу объясняет назначение класса.
Помните и про свойства: у каждого «среза» может быть свой префикс, но базовый префикс продукта лучше един: `my.feature.web.*`, `my.feature.client.*`. Так IDE-подсказки выглядят предсказуемо.

**Код (Java): два независимых автоконфига одного модуля**

```java
package split.sample;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.client.RestClient;

@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
class MyFeatureWebAutoConfiguration {
  @Bean @ConditionalOnMissingBean HandlerInterceptor auditInterceptor() { return new HandlerInterceptor(){}; }
}

@AutoConfiguration
@ConditionalOnClass(RestClient.class)
class MyFeatureClientAutoConfiguration {
  @Bean @ConditionalOnMissingBean RestClient featureClient() { return RestClient.create(); }
}
```

**Код (Kotlin): аналог**

```kotlin
package split.sample

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication
import org.springframework.context.annotation.Bean
import org.springframework.web.client.RestClient
import org.springframework.web.servlet.HandlerInterceptor

@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
class MyFeatureWebAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    fun auditInterceptor(): HandlerInterceptor = HandlerInterceptor { _, _, _ -> true }
}

@AutoConfiguration
@ConditionalOnClass(RestClient::class)
class MyFeatureClientAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    fun featureClient(): RestClient = RestClient.create()
}
```

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
split.sample.MyFeatureWebAutoConfiguration
split.sample.MyFeatureClientAutoConfiguration
```

---

## Изоляция от пользовательских бинов: не считайте их «данностью», проверяйте наличие через условия

Автоконфиг не должен «предполагать», что в приложении есть конкретные пользовательские бины. Если вам нужно взаимодействовать с ними, делайте это через условия `@ConditionalOnBean` и опциональную инъекцию (`ObjectProvider`). Это позволяет автоконфигу быть «вежливым»: если бин есть — мы подстраиваемся; если нет — молчим.
Например, ваш модуль может обогащать существующий `RestClient` перехватчиками. Делайте два пути: если пользователь объявил `RestClient` — декорируйте его; если нет — создайте свой дефолтный (но только если он уместен). В обоих случаях `@ConditionalOnMissingBean` гарантирует бэкоф.
Изоляция нужна и в обратную сторону: не тяните в автоконфиг зависимость от конкретных доменных бинов — это нарушает границы и делает библиотеку непереносимой. Автоконфиг должен опираться на инфраструктуру (DataSource, MeterRegistry, Environment) и собственные свойства.
Старайтесь избегать прямых вызовов `ApplicationContext.getBean(...)` — это Service Locator и ломает тестируемость. Если вам нужно «необязательное» участие внешнего бина, используйте `ObjectProvider<T>`/`Provider<T>`, которые безопасно выдают 0..N кандидатов без жёсткой связности.
Опциональные точки расширения можно оформить как интерфейсы (SPI) и искать их в контексте коллекцией: `List<MyCustomizer>`. Тогда пользователь добавит свой кастомизатор, и вы примените его в порядке `@Order`.
Всё это делает автоконфиг предсказуемым: он не «ломает» чужой граф и не высасывает из воздуха зависимости, а аккуратно встраивается, если есть условия.

**Код (Java): безопасная опциональная интеграция через ObjectProvider**

```java
package isolation.sample;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestClient;

public interface RestClientCustomizer { void customize(RestClient.Builder b); }

@AutoConfiguration
@ConditionalOnClass(RestClient.class)
class RestClientAutoConfig {
  @Bean RestClient restClient(ObjectProvider<RestClientCustomizer> customizers) {
    RestClient.Builder b = RestClient.builder();
    customizers.orderedStream().forEach(c -> c.customize(b));
    return b.build();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package isolation.sample

import org.springframework.beans.factory.ObjectProvider
import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.context.annotation.Bean
import org.springframework.core.annotation.Order
import org.springframework.web.client.RestClient

fun interface RestClientCustomizer { fun customize(b: RestClient.Builder) }

@AutoConfiguration
@ConditionalOnClass(RestClient::class)
class RestClientAutoConfig {
    @Bean
    fun restClient(customizers: ObjectProvider<RestClientCustomizer>): RestClient {
        val b = RestClient.builder()
        customizers.orderedStream().forEach { it.customize(b) }
        return b.build()
    }
}

@Order(100)
class TimeoutCustomizer : RestClientCustomizer { override fun customize(b: RestClient.Builder) { /* set timeouts */ } }
```

---

# 5. Как написать свою авто-конфигурацию

## Структура модулей: `my-lib-autoconfigure` (аннотации, условия, бины) и `my-lib-spring-boot-starter` (только зависимости)

Классическая упаковка — разделить «логику» автоконфига и «тонкий» стартер. В модуле `*-autoconfigure` лежат классы `@AutoConfiguration`, `@ConfigurationProperties`, тесты `ApplicationContextRunner`, метаданные и файл `AutoConfiguration.imports`. В модуле `spring-boot-starter-*` — только `pom/gradle` с зависимостями: сам `*-autoconfigure` и внешние библиотеки (например, HTTP-клиент), плюс — *никакого кода*.
Зачем делить? Приложение может подтянуть автоконфиг напрямую (без стартера) и более гибко управлять зависимостями. Стартер же помогает разработчикам: подключил одну зависимость — получил всё нужное. Это стандарт Spring Boot и его экосистемы.
Такую структуру проще версионировать и тестировать. Автоконфиг можно гонять изолированно, не подтягивая весь список транзитивных библиотек. Стартер — по сути «alias» нескольких зависимостей — проверяется минимально.
При публикации в Maven/Bintray оба артефакта получают согласованные версии. CI может выпускать `*-autoconfigure` чаще (когда меняются условия/свойства), а стартер — только когда меняется набор зависимостей.
В документации обязательно объясните, что даёт стартер, какие свойства доступны, как выключить/переопределить. Это снимает вопросы у потребителей и делает интеграцию быстрой.
Наконец, не забывайте про BOM: стартер не должен пинить версии напрямую — пусть ими управляет BOM Spring Boot или ваш корпоративный BOM. Это предотвращает «ад зависимостей».

**Код (Java): простейший автоконфиг и стартер — структура**

```java
// module: my-hello-autoconfigure
package my.hello.autoconf;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;

public interface Hello { String hi(); }

@AutoConfiguration
public class HelloAutoConfiguration {
  @Bean @ConditionalOnMissingBean Hello hello() { return () -> "Hello"; }
}
```

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
my.hello.autoconf.HelloAutoConfiguration
```

**Код (Kotlin): потребитель использует стартер**

```kotlin
@SpringBootApplication
class DemoApp

fun main(args: Array<String>) = runApplication<DemoApp>(*args)
```

Gradle (Groovy) — стартер:

```groovy
// module: my-hello-spring-boot-starter
dependencies {
  api project(':my-hello-autoconfigure')
  api 'org.springframework.boot:spring-boot-autoconfigure'
}
```

Gradle (Kotlin) — потребитель:

```kotlin
dependencies { implementation("com.acme:my-hello-spring-boot-starter:1.0.0") }
```

---

## Класс с `@AutoConfiguration` + конфиг-бины `@Bean` с `@Conditional*`; регистрация в `AutoConfiguration.imports`

Сердце — класс, помеченный `@AutoConfiguration`. Внутри — методы `@Bean`, каждый «вежливо» закрыт условиями: `@ConditionalOnClass` (внешняя библиотека доступна), `@ConditionalOnMissingBean` (бэкоф), `@ConditionalOnProperty` (включатель/выключатель), `@ConditionalOnBean` (инфраструктура есть). Список таких классов регистрируется в `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
Названия бинов держите стабильными и документируйте, если ожидаете индивидуальные переопределения «по имени». В противном случае полагайтесь на типы и `@Primary`/`@Order`.
Внутри автоконфига допустимы малые вспомогательные методы, но не стоит выносить туда бизнес-логику или тяжёлые вычисления. Цель — определить и связать бины. Всё сложное — внутри самих компонентов, которые вы создаёте.
Свойства читайте через `@ConfigurationProperties` и инжектируйте в фабрики `@Bean`. Это даёт типобезопасность, валидацию и подсказки IDE, а также «снимок» конфигурации на старте.
Если автоконфиг предлагает «настройщики» (`Customizer`/`Configurer` интерфейсы), инжектируйте коллекции: `List<MyCustomizer>`, применяйте по `@Order`. Это открытый SPI для пользователей.
Проверяйте себя тестами `ApplicationContextRunner`: сценарии «флаг включён», «класса нет», «пользовательский бин существует» — минимальный набор.

**Код (Java): полный пример с условием и свойствами**

```java
package my.http.autoconf;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.Min;
import org.springframework.web.client.RestClient;

@Validated
@ConfigurationProperties("my.http")
class MyHttpProps {
  @Min(100) private int timeoutMs = 1000;
  public int getTimeoutMs(){return timeoutMs;} public void setTimeoutMs(int v){this.timeoutMs=v;}
}

@AutoConfiguration
@ConditionalOnClass(RestClient.class)
@ConditionalOnProperty(prefix = "my.http", name = "enabled", havingValue = "true", matchIfMissing = true)
public class MyHttpAutoConfiguration {
  @Bean MyHttpProps myHttpProps(){ return new MyHttpProps(); }

  @Bean
  @ConditionalOnMissingBean
  RestClient myRestClient(MyHttpProps p) {
    return RestClient.builder()/* .set timeouts via client factory */.build();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package my.http.autoconf

import jakarta.validation.constraints.Min
import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Bean
import org.springframework.validation.annotation.Validated
import org.springframework.web.client.RestClient

@Validated
@ConfigurationProperties("my.http")
data class MyHttpProps(@field:Min(100) val timeoutMs: Int = 1000)

@AutoConfiguration
@ConditionalOnClass(RestClient::class)
@ConditionalOnProperty(prefix = "my.http", name = ["enabled"], havingValue = "true", matchIfMissing = true)
class MyHttpAutoConfiguration {
    @Bean fun myHttpProps() = MyHttpProps()
    @Bean @ConditionalOnMissingBean
    fun myRestClient(p: MyHttpProps): RestClient = RestClient.create()
}
```

`AutoConfiguration.imports`:

```
my.http.autoconf.MyHttpAutoConfiguration
```

---

## Валидация и свойства: `@ConfigurationProperties` по конструктору; дефолты и однозначные точки выключения

Классы свойств — лучший способ описать конфигурационный контракт модуля. В Boot 3.x по умолчанию используется конструкторный биндинг, что делает экземпляры неизменяемыми и валидируемыми при старте. Отказывайтесь от «россыпи» `@Value` — это не прозрачно и плохо тестируется.
Дайте префикс, говорящий сам за себя (`my.http.*`, `corp.mail.*`). Пропишите разумные дефолты и комментарии (Javadoc) — они попадут в метаданные и IDE-подсказки. Для «тяжёлых» фич заведите флаги `*.enabled=false` по умолчанию, чтобы включение было осознанным.
Валидация через Jakarta Validation (`@NotBlank`, `@Min`, `@DurationMin`, и пр.) обеспечивает fail-fast: приложение не стартует, если конфиг нарушает инварианты. Это особенно важно для таймаутов, лимитов, URL.
Свойства должны влиять только на инфраструктуру. Не протаскивайте `Environment` в домен; лучше инжектируйте уже настроенные клиенты/политики. Так модуль остаётся чистым и переносимым.
Помните о метаданных: подключите `spring-boot-configuration-processor`, чтобы IDE подсказывала ключи и типы. Это снижает число ошибок на вводе и ускоряет онбординг.
Для выключателей используйте `@ConditionalOnProperty(..., matchIfMissing = false)` там, где фича «опасна» по умолчанию, и `true` там, где она «безопасна». Документируйте это в README.

**Код (Java): свойства + валидация + выключатель**

```java
package my.mail.autoconf;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

@Validated
@Component
@ConfigurationProperties("my.mail")
public class MailProps {
  @NotBlank private String host;
  @Min(1) @Max(65535) private int port = 25;
  private boolean enabled = false;
  // getters/setters
  public String getHost(){return host;} public void setHost(String v){this.host=v;}
  public int getPort(){return port;} public void setPort(int v){this.port=v;}
  public boolean isEnabled(){return enabled;} public void setEnabled(boolean v){this.enabled=v;}
}
```

**Код (Kotlin): аналог**

```kotlin
package my.mail.autoconf

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated

@Validated
@Component
@ConfigurationProperties("my.mail")
data class MailProps(
    @field:NotBlank val host: String = "localhost",
    @field:Min(1) @field:Max(65535) val port: Int = 25,
    val enabled: Boolean = false
)
```

Gradle:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-validation'
  annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-validation")
  annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```

---

## Runtime hints для native-сборок: ресурсы/рефлексия/прокси через `RuntimeHints`/`@ImportRuntimeHints`

Для GraalVM Native Image динамика (рефлексия, ресурсы, прокси) должна быть описана заранее. Автоконфиг — отличное место, чтобы зарегистрировать хинты, ведь именно он знает, какие классы и ресурсы вы используете. В Boot 3.x предусмотрены `RuntimeHints` и `@ImportRuntimeHints`, которые позволяют без `reflect-config.json` задекларировать нужные доступы.
Если ваши свойства биндятся на record/класс без рефлексии — ничего делать не нужно. Если же библиотека внутри использует рефлексию (например, для (де)сериализации), зарегистрируйте типы через `hints.reflection().registerType`. Для ресурсов (шаблоны, файлы в `classpath:`) — `hints.resources().registerPattern`. Для JDK-прокси — `hints.proxies().registerJdkProxy`.
Хинты должны быть «узкими»: регистрируйте только то, что реально используется при сработавших условиях. Это уменьшит размер бинарника и ускорит образ.
Тестируйте нативную сборку на CI хотя бы для smoke-приложения, использующего ваш стартер: так вы поймаете недостающие хинты до релиза.
Помните, что условия (`OnClass`, `OnProperty`) влияют и на хинты: вы можете генерировать их программно, проверяя наличие классов/флагов в `RuntimeHintsRegistrar`.
Документируйте ограничения нативной сборки (например, отсутствие поддержки динамических скриптов или SPI), чтобы потребители знали, чего ожидать.

**Код (Java): регистрация runtime-хинтов**

```java
package my.nativehints;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.context.annotation.ImportRuntimeHints;

@ImportRuntimeHints(MyHints.class)
public class MyNativeAwareAutoConfiguration { /* @AutoConfiguration + бины как обычно */ }

class MyHints implements RuntimeHintsRegistrar {
  @Override public void registerHints(RuntimeHints hints, ClassLoader cl) {
    hints.resources().registerPattern("my/templates/*.txt");
    hints.reflection().registerType(com.fasterxml.jackson.databind.ObjectMapper.class);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package my.nativehints

import org.springframework.aot.hint.RuntimeHints
import org.springframework.aot.hint.RuntimeHintsRegistrar
import org.springframework.context.annotation.ImportRuntimeHints

@ImportRuntimeHints(MyHints::class)
class MyNativeAwareAutoConfiguration

class MyHints : RuntimeHintsRegistrar {
    override fun registerHints(hints: RuntimeHints, classLoader: ClassLoader?) {
        hints.resources().registerPattern("my/templates/*.txt")
        hints.reflection().registerType(com.fasterxml.jackson.databind.ObjectMapper::class.java)
    }
}
```

---

# 6. Стартеры: назначение, структура, правила

## Что такое «стартер»: тонкая obвязка зависимостей + `*-autoconfigure` модуль

Стартер — это удобная зависимость для потребителя: один координат — и у вас приезжают нужные библиотеки и соответствующий `*-autoconfigure`. В стартере **нет кода**: только `pom/gradle`-манифест, управляющий зависимостями и (иногда) опциональными модулями. Это делает стартер лёгким, предсказуемым и безопасным для апгрейдов.
Стартер снимает когнитивную нагрузку: не нужно помнить, что к `web` нужен Jackson, валидатор и Tomcat; что к `data-jpa` нужен Hibernate и драйвер. Вы подключаете `spring-boot-starter-web` — и получаете всё необходимое.
Для корпоративных платформ стартер — способ инкапсулировать конвенции: «наш HTTP» + «наш трейсинг» + «наши политики таймаутов». Команда продукта подключает один артефакт и получает унифицированное поведение.
Стартер также фиксирует «минимальный» набор зависимостей, исключая ленивые «модные» библиотеки. Чем короче список, тем меньше рисков конфликтов версий. Всё, что может быть опционально, выносится в `optional`/`runtimeOnly`.
Важно не смешивать роли: логика автоконфига — только в `*-autoconfigure`. Стартер не должен включать классов, которые «влезают» в контекст. Так вы избегаете сюрпризов и держите архитектуру чистой.
Документация стартера — список зависимостей, версия BOM, ссылка на свойства автоконфига, инструкция по отключению/переопределению. Это единственное место, куда смотрит потребитель при интеграции.

**Код (Java/Kotlin): потребительский код — просто инжектит бин, принесённый стартером**

```java
package starter.use;

import my.hello.autoconf.Hello;
import org.springframework.stereotype.Component;

@Component
class Greeter {
  Greeter(Hello hello){ System.out.println(hello.hi()); }
}
```

```kotlin
package starter.use

import my.hello.autoconf.Hello
import org.springframework.stereotype.Component

@Component
class Greeter(hello: Hello) { init { println(hello.hi()) } }
```

Gradle (Groovy/Kotlin) — подключение стартера:

```groovy
dependencies { implementation 'com.acme:my-hello-spring-boot-starter:1.0.0' }
```

```kotlin
dependencies { implementation("com.acme:my-hello-spring-boot-starter:1.0.0") }
```

---

## Нейминг: `spring-boot-starter-<name>` для потребителей; автоконфиг — `<name>-autoconfigure`

Имена важны для предсказуемости. Общее правило экосистемы: для потребительских артефактов используйте префикс `spring-boot-starter-<name>`. Для внутренних/корпоративных — `<company>-spring-boot-starter-<name>`. Автоконфиг лежит в соседнем артефакте `<name>-autoconfigure`.
Такой нейминг сразу подсказывает, что этот артефакт «подключаемый» и совместим с Boot; что автоконфиг можно тянуть отдельно; что «логика» не смешана с «обвязкой». Это снижает порог входа для новых разработчиков.
Не злоупотребляйте подстановками в имени: «starter-all» — анти-паттерн. Лучше несколько маленьких стартеров (web, data, observability), чем один огромный, тянущий «всё».
Пакетные имена классов автоконфига отражают зону ответственности: `com.company.feature.web`, `com.company.feature.client`. Это облегчает поиск в отчёте условий.
Файлы ресурсов (`AutoConfiguration.imports`) кладите в `META-INF/spring/` — это конвенция Boot 3.x. Для метаданных свойств соблюдайте префикс и документируйте переходы (deprecated-ключи).
Поддерживайте семантику версий (semver) и совместимость стартера/автоконфига между минорными релизами: потребители не должны «ломаться» от апгрейда.

**Код (Java/Kotlin): демонстрационный пакет и аннотация**

```java
package com.acme.pay.client; // ясная зона: pay + client
public interface PayClient { /* ... */ }
```

```kotlin
package com.acme.pay.web // web-часть той же фичи
class PayTracingFilter
```

Gradle:

```groovy
group = 'com.acme'
version = '1.2.0'
```

```kotlin
group = "com.acme"
version = "1.2.0"
```

---

## Управление версиями через BOM (`spring-boot-dependencies`); никаких фиксированных версий в стартере

Правильный способ управлять версиями — импортировать BOM Spring Boot (`spring-boot-dependencies`) в потребительском проекте и/или вашем корпоративном BOM. Тогда стартер объявляет зависимости **без версий** — их подтянет BOM. Это предотвращает «заморозку» библиотек на старых версиях и конфликт артефактов при апгрейде Boot.
Если вы публикуете корпоративный BOM, он может «прикладывать» ваши стартеры и их версии к конкретной версии Boot. Потребитель импортирует один BOM и получает согласованный набор артефактов.
В самом стартере не указывайте `<version>` у зависимостей, которые управляются BOM. Исключение — редкие внешние артефакты вне BOM; но лучше добавить их в ваш BOM, чем пинить в стартере.
Gradle с плагином dependency-management делает это просто: подключаете `io.spring.dependency-management` и импортируете BOM. Дальше можно писать `implementation("org.springframework.boot:spring-boot-starter-web")` без версий.
Это также упрощает поддержку: апгрейд Boot на минор → новые версии библиотек подтянулись автоматически; ваши стартеры остались неизменными.
Документируйте минимально поддерживаемую версию Boot для вашего стартера, чтобы потребители понимали границы.

**Gradle (Groovy/Kotlin): импорт BOM**

```groovy
plugins { id 'io.spring.dependency-management' version '1.1.6' }
dependencyManagement {
  imports { mavenBom "org.springframework.boot:spring-boot-dependencies:3.3.4" }
}
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

```kotlin
plugins { id("io.spring.dependency-management") version "1.1.6" }
dependencyManagement {
  imports { mavenBom("org.springframework.boot:spring-boot-dependencies:3.3.4") }
}
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

**Код (Java/Kotlin): обычное использование — без указания версий**

```java
// Никакого кода здесь не нужно; показано выше в Gradle.
// Потребитель продолжает инжектить бины, пришедшие со стартером.
```

```kotlin
// Аналогично: код потребителя не меняется от BOM-настройки.
```

---

## Минимизация транзитивных зависимостей и отсутствие кода в стартере

Чем меньше транзитивных зависимостей тянет стартер, тем меньше вероятность конфликтов и дубликатов классов. Подключайте только действительно необходимые артефакты, остальное — как `optional`/`runtimeOnly`, чтобы потребитель добавил при необходимости.
Избегайте «комфортных» пакетов вроде `*-all`, которые подтягивают пол-интернета. Сформируйте аккуратный список: для клиента — HTTP и сериализация; для наблюдаемости — actuator и реестр метрик; для БД — JDBC/JPA, но не конкретный драйвер (пусть потребитель выберет).
Повторим: **никакого кода в стартере**. Как только вы добавляете класс, стартер превращается в «непредсказуемый» источник бинов. Логика и условия — только в `*-autoconfigure`.
Если есть условно-подключаемые интеграции (например, с тремя разными трейсинг-бэкендами), делайте суб-стартеры: `starter-observability-otel`, `starter-observability-statsd`. Потребитель выберет ровно то, что ему нужно.
Документируйте, какие зависимости транзитивны и зачем: это ускоряет ревью безопасности и помогает DevSecOps команде. Список должен быть коротким и понятным.
Поддерживайте тест на «гигиену»: скрипт/плагин, проверяющий, что в стартере нет классов, а список зависимостей не «распух».

**Код (Java/Kotlin): потребитель добавляет драйвер отдельно (минимализм стартера)**

```groovy
dependencies {
  implementation 'com.acme:corp-jdbc-spring-boot-starter:1.0.0'
  runtimeOnly 'org.postgresql:postgresql' // потребитель сам выбирает драйвер
}
```

```kotlin
dependencies {
  implementation("com.acme:corp-jdbc-spring-boot-starter:1.0.0")
  runtimeOnly("org.postgresql:postgresql")
}
```

---

# 7. Кастомизация и отключение авто-конфигураций

## Точечное выключение: `spring.autoconfigure.exclude` или `@EnableAutoConfiguration(exclude = …)`

Иногда проще полностью исключить автоконфиг: например, если вы строите чистый WebFlux и не хотите, чтобы MVC-ветка даже пыталась активироваться, или когда штатный конфиг мешает корпоративной политике. Для этого есть два эквивалентных механизма: свойство `spring.autoconfigure.exclude` и параметр `exclude` у `@EnableAutoConfiguration`/`@SpringBootApplication`.
Свойство удобно для окружений: вы можете выключить конфиг на dev-стендах или в конкретном сервисе без перепаковки кода. Аннотация — для постоянных контрактов: «этот сервис никогда не поднимает MVC». Оба решения читаемы и легко ищутся.
Выключение работает на уровне класса автоконфига: вы указываете его FQCN. Чтобы узнать нужное имя, смотрите отчёт условий или файл `AutoConfiguration.imports` в зависимостях.
Не переусердствуйте: часто достаточно флага `*.enabled=false` или объявления своего `@Bean`, чтобы дефолт отступил. Полное исключение полезно, когда автоконфиг делает слишком много «за вас» и вы намерены управлять областью полностью.
Имейте в виду, что исключение автоконфига может вырубить и транзитивные бины, на которые вы рассчитывали (например, `Jackson` настройки). Проверьте `/actuator/beans` или отчёт условий, чтобы не остаться без критичных компонентов.
Добавьте smoke-тест на старт контекста после исключения: `ApplicationContextRunner` быстро покажет, что контекст поднимается и содержит ожидаемые бины.

**Код (Java/Kotlin): выключение через свойство и через аннотацию**

```yaml
# application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

```java
@SpringBootApplication(exclude = {org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.class})
public class NoJdbcApp { public static void main(String[] args){ org.springframework.boot.SpringApplication.run(NoJdbcApp.class,args);} }
```

```kotlin
@SpringBootApplication(exclude = [org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration::class])
class NoJdbcApp
fun main(args: Array<String>) = org.springframework.boot.runApplication<NoJdbcApp>(*args)
```

---

## Перекрытие бинов: объявите свой `@Bean` совместимого типа — автоконфиг «отступит»

Наиболее частый способ кастомизации — предоставить собственный бин того же контракта. Благодаря `@ConditionalOnMissingBean` в автоконфиге ваш бин «побеждает», а дефолт не создаётся. Это точечная, безопасная и прозрачная замена.
Важный момент — тип и имя. По умолчанию условия проверяют тип; если автоконфиг объявляет бин по имени, можно использовать то же имя (`@Bean(name = "…")`) или `@Primary`, когда кандидатов несколько. Документация автоконфига обычно указывает, как именно выполняется бэкоф.
Перекрывать имеет смысл, когда меняется не пару свойств, а «философия»: особая политика ретраев, альтернативная сериализация, иной пул соединений. Когда различия минимальны, сначала попробуйте свойства модуля.
В коллекциях (например, список конвертеров/перехватчиков) перекрытие — это добавление, а не замена. Порядок задают `@Order`/`Ordered`; вы можете вставить свою реализацию выше дефолтов.
Следите, чтобы ваш бин оставался совместимым с остальной экосистемой: например, кастомный `ObjectMapper` должен иметь зарегистрированные модули JavaTime/ParameterNames, иначе ломаются стандартные сценарии.
Сделайте тест на старт контекста с вашим перекрытием: это дешёвая гарантия, что дефолт отступил, а остальной граф остался здоровым.

**Код (Java): замена дефолтного `ObjectMapper` и добавление кастомайзера**

```java
package override.sample;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class JsonOverrideConfig {
  @Bean ObjectMapper objectMapper() {
    return new ObjectMapper().registerModule(new JavaTimeModule());
  }
}
```

**Код (Kotlin): аналог + пример списка кастомайзеров**

```kotlin
package override.sample

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order

@Configuration
class JsonOverrideConfig {
    @Bean fun objectMapper(): ObjectMapper = ObjectMapper().registerModule(JavaTimeModule())
}

fun interface RestClientCustomizer { fun customize(b: org.springframework.web.client.RestClient.Builder) }

@Order(0)
class RetryCustomizer : RestClientCustomizer { override fun customize(b: org.springframework.web.client.RestClient.Builder) { /* set retries */ } }
```

---

## Тюнинг через свойства: включение/выключение подсекций автоконфига, явные дефолты

Многие автоконфиги дают десятки «тонких» настроек через `@ConfigurationProperties`. Это лучший канал кастомизации: безопасные дефолты уже подобраны, вы меняете только нужные параметры. Например, таймауты в `RestClient`, буферы в кодеках, пути health-проб в Actuator.
Используйте IDE-подсказки (метаданные), чтобы не охотиться за ключами в исходниках. Хорошие библиотеки документируют префиксы и дают javadoc в метаданных.
Флаги `*.enabled` — явные переключатели: включают/выключают части автоконфига (фильтры, интеграции, фоновые задачи). Предпочтительно отключать «тяжёлые» подсекции по умолчанию и включать осознанно.
Свойства работают на всех окружениях одинаково: baseline в `application.yml`, profile-оверрайды в `application-prod.yml`, разовые изменения через переменные окружения/CLI. Это даёт удобный и безопасный процесс эксплуатации.
Следите за коллизиями: если свойство перекрывается в нескольких местах, выигрывает источник с большим приоритетом. Включите отчёт условий/источников при отладке или используйте методический «трассировщик» `Environment` на старте.
Договоритесь в команде о «реестре» ключей: какие считаются обязательными, какие секретны, где лежат дефолты. Это уменьшает сюрпризы при апгрейдах.

**Код (Java/Kotlin): настройка через свойства**

```yaml
my:
  http:
    enabled: true
    timeout-ms: 1500
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

```java
// бин из вашего автоконфига уже читает MyHttpProps; меняем только YAML.
```

```kotlin
// аналогично: код не меняется, вся кастомизация — в свойствах.
```

# 4. Порядок применения и зависимость авто-конфигураций

## `@AutoConfiguration(before/after = …)` и `@AutoConfigureOrder`: управление очередностью, когда есть перекрытия

Порядок применения авто-конфигураций важен в случаях, когда одна конфигурация полагается на бины, созданные другой, или должна выполниться раньше, чтобы подменить дефолты. В Spring Boot это решается двумя механизмами: атрибутами `before/after` у `@AutoConfiguration` и аннотацией `@AutoConfigureOrder`. Первый задаёт относительные зависимости между конкретными классами автоконфигов, второй — числовой приоритет наподобие `@Order` (меньше — раньше).
Когда вы контролируете оба конца зависимости (свои автоконфиги), предпочтительнее использовать `before/after`, так как это декларативная, читаемая связь «кто перед кем». Числовые приоритеты уместны в общих библиотеках, где вы не знаете имён потенциальных соседей, но хотите располагаться «пораньше» или «позже» в общем списке.
Частая задача — расширить настройки существующего дефолта: например, добавить перехватчик в `RestClient`. Правильный подход — выполнить ваш конфиг «после» стандартного `RestClient`-конфига, чтобы иметь готовый бин для декорирования. Обратная ситуация — вы хотите заблокировать «лишние» дефолты, тогда ваш конфиг должен выполниться «до» и выставить пользовательский бин, из-за чего дефолт «отступит».
Важно помнить, что порядок влияет только на регистрацию дефиниций бинов, а не на фактическое создание singleton-ов: создание происходит позже согласно обычным правилам контейнера. Тем не менее, если ваш конфиг зависит от уже зарегистрированных `BeanDefinition`, задайте `after` и используйте условия `@ConditionalOnBean`.
Ошибкой будет «силовой» порядок без условий. Даже если вы выполнитесь раньше, но не проверяете наличие пользовательского бина, вы можете создать нежелательный дефолт. Поэтому порядок всегда идёт в паре с условиями бэкофа (`OnMissingBean`).
Отлаживайте порядок через отчёт условий и, при необходимости, с помощью минимальных тестов на `ApplicationContextRunner`: так вы увидите, какой конфиг применился первым, и как повлиял на доступность бинов для последующих конфигов.

**Код (Java): автоконфиг, который должен выполниться после стандартного**

```java
package order.sample;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.AutoConfigureOrder;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.core.Ordered;
import org.springframework.web.client.RestClient;

@AutoConfiguration(after = org.springframework.boot.autoconfigure.web.client.RestClientAutoConfiguration.class)
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE) // на всякий случай позже большинства
public class RestClientDecoratorAutoConfiguration {

  @Bean
  @ConditionalOnBean(RestClient.class)
  @ConditionalOnMissingBean(name = "metricsRestClient")
  RestClient metricsRestClient(RestClient base) {
    return RestClient.builder().baseUrl("https://metrics-wrapper")
        .requestFactory(base::getRequestFactory).build();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package order.sample

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.AutoConfigureOrder
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.core.Ordered
import org.springframework.context.annotation.Bean
import org.springframework.web.client.RestClient

@AutoConfiguration(after = [org.springframework.boot.autoconfigure.web.client.RestClientAutoConfiguration::class])
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
class RestClientDecoratorAutoConfiguration {
    @Bean
    @ConditionalOnBean(RestClient::class)
    @ConditionalOnMissingBean(name = ["metricsRestClient"])
    fun metricsRestClient(base: RestClient): RestClient =
        RestClient.builder().requestFactory(base::getRequestFactory).build()
}
```

Gradle (Groovy/Kotlin) — модуль с автоконфигом:

```groovy
dependencies {
  api 'org.springframework.boot:spring-boot-autoconfigure'
}
```

```kotlin
dependencies {
  api("org.springframework.boot:spring-boot-autoconfigure")
}
```

---

## Дробление на несколько конфиг-классов по зонам ответственности: web-слой, клиенты, data-слой, наблюдаемость

Монолитные автоконфиги на сотни строк тяжело читать, тестировать и развивать. Хорошая практика — дробить по функциональным «срезам»: отдельный класс для web (фильтры, кодеки), отдельный для клиентов (RestClient, gRPC), отдельный для data (DataSource, репозитории), отдельный для наблюдаемости (MeterRegistry, обвязки). Это улучшает локализацию условий и облегчает выбор порядка между независимыми частями.
Каждый такой класс получает собственные условия (`OnClass`, `OnWebApplication`, `OnBean`) и не знает о «соседях» напрямую. Если нужно «соединить» слои, используйте `@ConditionalOnBean`(инфраструктурный бин) вместо прямых вызовов методов соседней конфигурации. Так вы избегаете жёсткой сцепки и облегчаете переиспользование кусков.
Дробление также помогает при AOT/Native: ненужные классы не попадут в итоговый список, если условия не сработают. Мелкие конфиги быстрее анализируются и точнее описывают зависимость от среды.
С точки зрения тестов вы получите маленькие изолированные сценарии: web-конфиг можно протестировать без базы, а data-конфиг — без веба. Это ускоряет CI и делает отчёт условий компактнее.
При именовании придерживайтесь ясности: `<Feature>WebAutoConfiguration`, `<Feature>ClientAutoConfiguration`, `<Feature>DataAutoConfiguration`, `<Feature>ObservabilityAutoConfiguration`. Такой паттерн сразу объясняет назначение класса.
Помните и про свойства: у каждого «среза» может быть свой префикс, но базовый префикс продукта лучше един: `my.feature.web.*`, `my.feature.client.*`. Так IDE-подсказки выглядят предсказуемо.

**Код (Java): два независимых автоконфига одного модуля**

```java
package split.sample;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.client.RestClient;

@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
class MyFeatureWebAutoConfiguration {
  @Bean @ConditionalOnMissingBean HandlerInterceptor auditInterceptor() { return new HandlerInterceptor(){}; }
}

@AutoConfiguration
@ConditionalOnClass(RestClient.class)
class MyFeatureClientAutoConfiguration {
  @Bean @ConditionalOnMissingBean RestClient featureClient() { return RestClient.create(); }
}
```

**Код (Kotlin): аналог**

```kotlin
package split.sample

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication
import org.springframework.context.annotation.Bean
import org.springframework.web.client.RestClient
import org.springframework.web.servlet.HandlerInterceptor

@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
class MyFeatureWebAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    fun auditInterceptor(): HandlerInterceptor = HandlerInterceptor { _, _, _ -> true }
}

@AutoConfiguration
@ConditionalOnClass(RestClient::class)
class MyFeatureClientAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    fun featureClient(): RestClient = RestClient.create()
}
```

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
split.sample.MyFeatureWebAutoConfiguration
split.sample.MyFeatureClientAutoConfiguration
```

---

## Изоляция от пользовательских бинов: не считайте их «данностью», проверяйте наличие через условия

Автоконфиг не должен «предполагать», что в приложении есть конкретные пользовательские бины. Если вам нужно взаимодействовать с ними, делайте это через условия `@ConditionalOnBean` и опциональную инъекцию (`ObjectProvider`). Это позволяет автоконфигу быть «вежливым»: если бин есть — мы подстраиваемся; если нет — молчим.
Например, ваш модуль может обогащать существующий `RestClient` перехватчиками. Делайте два пути: если пользователь объявил `RestClient` — декорируйте его; если нет — создайте свой дефолтный (но только если он уместен). В обоих случаях `@ConditionalOnMissingBean` гарантирует бэкоф.
Изоляция нужна и в обратную сторону: не тяните в автоконфиг зависимость от конкретных доменных бинов — это нарушает границы и делает библиотеку непереносимой. Автоконфиг должен опираться на инфраструктуру (DataSource, MeterRegistry, Environment) и собственные свойства.
Старайтесь избегать прямых вызовов `ApplicationContext.getBean(...)` — это Service Locator и ломает тестируемость. Если вам нужно «необязательное» участие внешнего бина, используйте `ObjectProvider<T>`/`Provider<T>`, которые безопасно выдают 0..N кандидатов без жёсткой связности.
Опциональные точки расширения можно оформить как интерфейсы (SPI) и искать их в контексте коллекцией: `List<MyCustomizer>`. Тогда пользователь добавит свой кастомизатор, и вы примените его в порядке `@Order`.
Всё это делает автоконфиг предсказуемым: он не «ломает» чужой граф и не высасывает из воздуха зависимости, а аккуратно встраивается, если есть условия.

**Код (Java): безопасная опциональная интеграция через ObjectProvider**

```java
package isolation.sample;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestClient;

public interface RestClientCustomizer { void customize(RestClient.Builder b); }

@AutoConfiguration
@ConditionalOnClass(RestClient.class)
class RestClientAutoConfig {
  @Bean RestClient restClient(ObjectProvider<RestClientCustomizer> customizers) {
    RestClient.Builder b = RestClient.builder();
    customizers.orderedStream().forEach(c -> c.customize(b));
    return b.build();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package isolation.sample

import org.springframework.beans.factory.ObjectProvider
import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.context.annotation.Bean
import org.springframework.core.annotation.Order
import org.springframework.web.client.RestClient

fun interface RestClientCustomizer { fun customize(b: RestClient.Builder) }

@AutoConfiguration
@ConditionalOnClass(RestClient::class)
class RestClientAutoConfig {
    @Bean
    fun restClient(customizers: ObjectProvider<RestClientCustomizer>): RestClient {
        val b = RestClient.builder()
        customizers.orderedStream().forEach { it.customize(b) }
        return b.build()
    }
}

@Order(100)
class TimeoutCustomizer : RestClientCustomizer { override fun customize(b: RestClient.Builder) { /* set timeouts */ } }
```

---

# 5. Как написать свою авто-конфигурацию

## Структура модулей: `my-lib-autoconfigure` (аннотации, условия, бины) и `my-lib-spring-boot-starter` (только зависимости)

Классическая упаковка — разделить «логику» автоконфига и «тонкий» стартер. В модуле `*-autoconfigure` лежат классы `@AutoConfiguration`, `@ConfigurationProperties`, тесты `ApplicationContextRunner`, метаданные и файл `AutoConfiguration.imports`. В модуле `spring-boot-starter-*` — только `pom/gradle` с зависимостями: сам `*-autoconfigure` и внешние библиотеки (например, HTTP-клиент), плюс — *никакого кода*.
Зачем делить? Приложение может подтянуть автоконфиг напрямую (без стартера) и более гибко управлять зависимостями. Стартер же помогает разработчикам: подключил одну зависимость — получил всё нужное. Это стандарт Spring Boot и его экосистемы.
Такую структуру проще версионировать и тестировать. Автоконфиг можно гонять изолированно, не подтягивая весь список транзитивных библиотек. Стартер — по сути «alias» нескольких зависимостей — проверяется минимально.
При публикации в Maven/Bintray оба артефакта получают согласованные версии. CI может выпускать `*-autoconfigure` чаще (когда меняются условия/свойства), а стартер — только когда меняется набор зависимостей.
В документации обязательно объясните, что даёт стартер, какие свойства доступны, как выключить/переопределить. Это снимает вопросы у потребителей и делает интеграцию быстрой.
Наконец, не забывайте про BOM: стартер не должен пинить версии напрямую — пусть ими управляет BOM Spring Boot или ваш корпоративный BOM. Это предотвращает «ад зависимостей».

**Код (Java): простейший автоконфиг и стартер — структура**

```java
// module: my-hello-autoconfigure
package my.hello.autoconf;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;

public interface Hello { String hi(); }

@AutoConfiguration
public class HelloAutoConfiguration {
  @Bean @ConditionalOnMissingBean Hello hello() { return () -> "Hello"; }
}
```

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
my.hello.autoconf.HelloAutoConfiguration
```

**Код (Kotlin): потребитель использует стартер**

```kotlin
@SpringBootApplication
class DemoApp

fun main(args: Array<String>) = runApplication<DemoApp>(*args)
```

Gradle (Groovy) — стартер:

```groovy
// module: my-hello-spring-boot-starter
dependencies {
  api project(':my-hello-autoconfigure')
  api 'org.springframework.boot:spring-boot-autoconfigure'
}
```

Gradle (Kotlin) — потребитель:

```kotlin
dependencies { implementation("com.acme:my-hello-spring-boot-starter:1.0.0") }
```

---

## Класс с `@AutoConfiguration` + конфиг-бины `@Bean` с `@Conditional*`; регистрация в `AutoConfiguration.imports`

Сердце — класс, помеченный `@AutoConfiguration`. Внутри — методы `@Bean`, каждый «вежливо» закрыт условиями: `@ConditionalOnClass` (внешняя библиотека доступна), `@ConditionalOnMissingBean` (бэкоф), `@ConditionalOnProperty` (включатель/выключатель), `@ConditionalOnBean` (инфраструктура есть). Список таких классов регистрируется в `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
Названия бинов держите стабильными и документируйте, если ожидаете индивидуальные переопределения «по имени». В противном случае полагайтесь на типы и `@Primary`/`@Order`.
Внутри автоконфига допустимы малые вспомогательные методы, но не стоит выносить туда бизнес-логику или тяжёлые вычисления. Цель — определить и связать бины. Всё сложное — внутри самих компонентов, которые вы создаёте.
Свойства читайте через `@ConfigurationProperties` и инжектируйте в фабрики `@Bean`. Это даёт типобезопасность, валидацию и подсказки IDE, а также «снимок» конфигурации на старте.
Если автоконфиг предлагает «настройщики» (`Customizer`/`Configurer` интерфейсы), инжектируйте коллекции: `List<MyCustomizer>`, применяйте по `@Order`. Это открытый SPI для пользователей.
Проверяйте себя тестами `ApplicationContextRunner`: сценарии «флаг включён», «класса нет», «пользовательский бин существует» — минимальный набор.

**Код (Java): полный пример с условием и свойствами**

```java
package my.http.autoconf;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.Min;
import org.springframework.web.client.RestClient;

@Validated
@ConfigurationProperties("my.http")
class MyHttpProps {
  @Min(100) private int timeoutMs = 1000;
  public int getTimeoutMs(){return timeoutMs;} public void setTimeoutMs(int v){this.timeoutMs=v;}
}

@AutoConfiguration
@ConditionalOnClass(RestClient.class)
@ConditionalOnProperty(prefix = "my.http", name = "enabled", havingValue = "true", matchIfMissing = true)
public class MyHttpAutoConfiguration {
  @Bean MyHttpProps myHttpProps(){ return new MyHttpProps(); }

  @Bean
  @ConditionalOnMissingBean
  RestClient myRestClient(MyHttpProps p) {
    return RestClient.builder()/* .set timeouts via client factory */.build();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package my.http.autoconf

import jakarta.validation.constraints.Min
import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Bean
import org.springframework.validation.annotation.Validated
import org.springframework.web.client.RestClient

@Validated
@ConfigurationProperties("my.http")
data class MyHttpProps(@field:Min(100) val timeoutMs: Int = 1000)

@AutoConfiguration
@ConditionalOnClass(RestClient::class)
@ConditionalOnProperty(prefix = "my.http", name = ["enabled"], havingValue = "true", matchIfMissing = true)
class MyHttpAutoConfiguration {
    @Bean fun myHttpProps() = MyHttpProps()
    @Bean @ConditionalOnMissingBean
    fun myRestClient(p: MyHttpProps): RestClient = RestClient.create()
}
```

`AutoConfiguration.imports`:

```
my.http.autoconf.MyHttpAutoConfiguration
```

---

## Валидация и свойства: `@ConfigurationProperties` по конструктору; дефолты и однозначные точки выключения

Классы свойств — лучший способ описать конфигурационный контракт модуля. В Boot 3.x по умолчанию используется конструкторный биндинг, что делает экземпляры неизменяемыми и валидируемыми при старте. Отказывайтесь от «россыпи» `@Value` — это не прозрачно и плохо тестируется.
Дайте префикс, говорящий сам за себя (`my.http.*`, `corp.mail.*`). Пропишите разумные дефолты и комментарии (Javadoc) — они попадут в метаданные и IDE-подсказки. Для «тяжёлых» фич заведите флаги `*.enabled=false` по умолчанию, чтобы включение было осознанным.
Валидация через Jakarta Validation (`@NotBlank`, `@Min`, `@DurationMin`, и пр.) обеспечивает fail-fast: приложение не стартует, если конфиг нарушает инварианты. Это особенно важно для таймаутов, лимитов, URL.
Свойства должны влиять только на инфраструктуру. Не протаскивайте `Environment` в домен; лучше инжектируйте уже настроенные клиенты/политики. Так модуль остаётся чистым и переносимым.
Помните о метаданных: подключите `spring-boot-configuration-processor`, чтобы IDE подсказывала ключи и типы. Это снижает число ошибок на вводе и ускоряет онбординг.
Для выключателей используйте `@ConditionalOnProperty(..., matchIfMissing = false)` там, где фича «опасна» по умолчанию, и `true` там, где она «безопасна». Документируйте это в README.

**Код (Java): свойства + валидация + выключатель**

```java
package my.mail.autoconf;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

@Validated
@Component
@ConfigurationProperties("my.mail")
public class MailProps {
  @NotBlank private String host;
  @Min(1) @Max(65535) private int port = 25;
  private boolean enabled = false;
  // getters/setters
  public String getHost(){return host;} public void setHost(String v){this.host=v;}
  public int getPort(){return port;} public void setPort(int v){this.port=v;}
  public boolean isEnabled(){return enabled;} public void setEnabled(boolean v){this.enabled=v;}
}
```

**Код (Kotlin): аналог**

```kotlin
package my.mail.autoconf

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated

@Validated
@Component
@ConfigurationProperties("my.mail")
data class MailProps(
    @field:NotBlank val host: String = "localhost",
    @field:Min(1) @field:Max(65535) val port: Int = 25,
    val enabled: Boolean = false
)
```

Gradle:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-validation'
  annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-validation")
  annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```

---

## Runtime hints для native-сборок: ресурсы/рефлексия/прокси через `RuntimeHints`/`@ImportRuntimeHints`

Для GraalVM Native Image динамика (рефлексия, ресурсы, прокси) должна быть описана заранее. Автоконфиг — отличное место, чтобы зарегистрировать хинты, ведь именно он знает, какие классы и ресурсы вы используете. В Boot 3.x предусмотрены `RuntimeHints` и `@ImportRuntimeHints`, которые позволяют без `reflect-config.json` задекларировать нужные доступы.
Если ваши свойства биндятся на record/класс без рефлексии — ничего делать не нужно. Если же библиотека внутри использует рефлексию (например, для (де)сериализации), зарегистрируйте типы через `hints.reflection().registerType`. Для ресурсов (шаблоны, файлы в `classpath:`) — `hints.resources().registerPattern`. Для JDK-прокси — `hints.proxies().registerJdkProxy`.
Хинты должны быть «узкими»: регистрируйте только то, что реально используется при сработавших условиях. Это уменьшит размер бинарника и ускорит образ.
Тестируйте нативную сборку на CI хотя бы для smoke-приложения, использующего ваш стартер: так вы поймаете недостающие хинты до релиза.
Помните, что условия (`OnClass`, `OnProperty`) влияют и на хинты: вы можете генерировать их программно, проверяя наличие классов/флагов в `RuntimeHintsRegistrar`.
Документируйте ограничения нативной сборки (например, отсутствие поддержки динамических скриптов или SPI), чтобы потребители знали, чего ожидать.

**Код (Java): регистрация runtime-хинтов**

```java
package my.nativehints;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.context.annotation.ImportRuntimeHints;

@ImportRuntimeHints(MyHints.class)
public class MyNativeAwareAutoConfiguration { /* @AutoConfiguration + бины как обычно */ }

class MyHints implements RuntimeHintsRegistrar {
  @Override public void registerHints(RuntimeHints hints, ClassLoader cl) {
    hints.resources().registerPattern("my/templates/*.txt");
    hints.reflection().registerType(com.fasterxml.jackson.databind.ObjectMapper.class);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package my.nativehints

import org.springframework.aot.hint.RuntimeHints
import org.springframework.aot.hint.RuntimeHintsRegistrar
import org.springframework.context.annotation.ImportRuntimeHints

@ImportRuntimeHints(MyHints::class)
class MyNativeAwareAutoConfiguration

class MyHints : RuntimeHintsRegistrar {
    override fun registerHints(hints: RuntimeHints, classLoader: ClassLoader?) {
        hints.resources().registerPattern("my/templates/*.txt")
        hints.reflection().registerType(com.fasterxml.jackson.databind.ObjectMapper::class.java)
    }
}
```

---

# 6. Стартеры: назначение, структура, правила

## Что такое «стартер»: тонкая obвязка зависимостей + `*-autoconfigure` модуль

Стартер — это удобная зависимость для потребителя: один координат — и у вас приезжают нужные библиотеки и соответствующий `*-autoconfigure`. В стартере **нет кода**: только `pom/gradle`-манифест, управляющий зависимостями и (иногда) опциональными модулями. Это делает стартер лёгким, предсказуемым и безопасным для апгрейдов.
Стартер снимает когнитивную нагрузку: не нужно помнить, что к `web` нужен Jackson, валидатор и Tomcat; что к `data-jpa` нужен Hibernate и драйвер. Вы подключаете `spring-boot-starter-web` — и получаете всё необходимое.
Для корпоративных платформ стартер — способ инкапсулировать конвенции: «наш HTTP» + «наш трейсинг» + «наши политики таймаутов». Команда продукта подключает один артефакт и получает унифицированное поведение.
Стартер также фиксирует «минимальный» набор зависимостей, исключая ленивые «модные» библиотеки. Чем короче список, тем меньше рисков конфликтов версий. Всё, что может быть опционально, выносится в `optional`/`runtimeOnly`.
Важно не смешивать роли: логика автоконфига — только в `*-autoconfigure`. Стартер не должен включать классов, которые «влезают» в контекст. Так вы избегаете сюрпризов и держите архитектуру чистой.
Документация стартера — список зависимостей, версия BOM, ссылка на свойства автоконфига, инструкция по отключению/переопределению. Это единственное место, куда смотрит потребитель при интеграции.

**Код (Java/Kotlin): потребительский код — просто инжектит бин, принесённый стартером**

```java
package starter.use;

import my.hello.autoconf.Hello;
import org.springframework.stereotype.Component;

@Component
class Greeter {
  Greeter(Hello hello){ System.out.println(hello.hi()); }
}
```

```kotlin
package starter.use

import my.hello.autoconf.Hello
import org.springframework.stereotype.Component

@Component
class Greeter(hello: Hello) { init { println(hello.hi()) } }
```

Gradle (Groovy/Kotlin) — подключение стартера:

```groovy
dependencies { implementation 'com.acme:my-hello-spring-boot-starter:1.0.0' }
```

```kotlin
dependencies { implementation("com.acme:my-hello-spring-boot-starter:1.0.0") }
```

---

## Нейминг: `spring-boot-starter-<name>` для потребителей; автоконфиг — `<name>-autoconfigure`

Имена важны для предсказуемости. Общее правило экосистемы: для потребительских артефактов используйте префикс `spring-boot-starter-<name>`. Для внутренних/корпоративных — `<company>-spring-boot-starter-<name>`. Автоконфиг лежит в соседнем артефакте `<name>-autoconfigure`.
Такой нейминг сразу подсказывает, что этот артефакт «подключаемый» и совместим с Boot; что автоконфиг можно тянуть отдельно; что «логика» не смешана с «обвязкой». Это снижает порог входа для новых разработчиков.
Не злоупотребляйте подстановками в имени: «starter-all» — анти-паттерн. Лучше несколько маленьких стартеров (web, data, observability), чем один огромный, тянущий «всё».
Пакетные имена классов автоконфига отражают зону ответственности: `com.company.feature.web`, `com.company.feature.client`. Это облегчает поиск в отчёте условий.
Файлы ресурсов (`AutoConfiguration.imports`) кладите в `META-INF/spring/` — это конвенция Boot 3.x. Для метаданных свойств соблюдайте префикс и документируйте переходы (deprecated-ключи).
Поддерживайте семантику версий (semver) и совместимость стартера/автоконфига между минорными релизами: потребители не должны «ломаться» от апгрейда.

**Код (Java/Kotlin): демонстрационный пакет и аннотация**

```java
package com.acme.pay.client; // ясная зона: pay + client
public interface PayClient { /* ... */ }
```

```kotlin
package com.acme.pay.web // web-часть той же фичи
class PayTracingFilter
```

Gradle:

```groovy
group = 'com.acme'
version = '1.2.0'
```

```kotlin
group = "com.acme"
version = "1.2.0"
```

---

## Управление версиями через BOM (`spring-boot-dependencies`); никаких фиксированных версий в стартере

Правильный способ управлять версиями — импортировать BOM Spring Boot (`spring-boot-dependencies`) в потребительском проекте и/или вашем корпоративном BOM. Тогда стартер объявляет зависимости **без версий** — их подтянет BOM. Это предотвращает «заморозку» библиотек на старых версиях и конфликт артефактов при апгрейде Boot.
Если вы публикуете корпоративный BOM, он может «прикладывать» ваши стартеры и их версии к конкретной версии Boot. Потребитель импортирует один BOM и получает согласованный набор артефактов.
В самом стартере не указывайте `<version>` у зависимостей, которые управляются BOM. Исключение — редкие внешние артефакты вне BOM; но лучше добавить их в ваш BOM, чем пинить в стартере.
Gradle с плагином dependency-management делает это просто: подключаете `io.spring.dependency-management` и импортируете BOM. Дальше можно писать `implementation("org.springframework.boot:spring-boot-starter-web")` без версий.
Это также упрощает поддержку: апгрейд Boot на минор → новые версии библиотек подтянулись автоматически; ваши стартеры остались неизменными.
Документируйте минимально поддерживаемую версию Boot для вашего стартера, чтобы потребители понимали границы.

**Gradle (Groovy/Kotlin): импорт BOM**

```groovy
plugins { id 'io.spring.dependency-management' version '1.1.6' }
dependencyManagement {
  imports { mavenBom "org.springframework.boot:spring-boot-dependencies:3.3.4" }
}
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

```kotlin
plugins { id("io.spring.dependency-management") version "1.1.6" }
dependencyManagement {
  imports { mavenBom("org.springframework.boot:spring-boot-dependencies:3.3.4") }
}
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

**Код (Java/Kotlin): обычное использование — без указания версий**

```java
// Никакого кода здесь не нужно; показано выше в Gradle.
// Потребитель продолжает инжектить бины, пришедшие со стартером.
```

```kotlin
// Аналогично: код потребителя не меняется от BOM-настройки.
```

---

## Минимизация транзитивных зависимостей и отсутствие кода в стартере

Чем меньше транзитивных зависимостей тянет стартер, тем меньше вероятность конфликтов и дубликатов классов. Подключайте только действительно необходимые артефакты, остальное — как `optional`/`runtimeOnly`, чтобы потребитель добавил при необходимости.
Избегайте «комфортных» пакетов вроде `*-all`, которые подтягивают пол-интернета. Сформируйте аккуратный список: для клиента — HTTP и сериализация; для наблюдаемости — actuator и реестр метрик; для БД — JDBC/JPA, но не конкретный драйвер (пусть потребитель выберет).
Повторим: **никакого кода в стартере**. Как только вы добавляете класс, стартер превращается в «непредсказуемый» источник бинов. Логика и условия — только в `*-autoconfigure`.
Если есть условно-подключаемые интеграции (например, с тремя разными трейсинг-бэкендами), делайте суб-стартеры: `starter-observability-otel`, `starter-observability-statsd`. Потребитель выберет ровно то, что ему нужно.
Документируйте, какие зависимости транзитивны и зачем: это ускоряет ревью безопасности и помогает DevSecOps команде. Список должен быть коротким и понятным.
Поддерживайте тест на «гигиену»: скрипт/плагин, проверяющий, что в стартере нет классов, а список зависимостей не «распух».

**Код (Java/Kotlin): потребитель добавляет драйвер отдельно (минимализм стартера)**

```groovy
dependencies {
  implementation 'com.acme:corp-jdbc-spring-boot-starter:1.0.0'
  runtimeOnly 'org.postgresql:postgresql' // потребитель сам выбирает драйвер
}
```

```kotlin
dependencies {
  implementation("com.acme:corp-jdbc-spring-boot-starter:1.0.0")
  runtimeOnly("org.postgresql:postgresql")
}
```

---

# 7. Кастомизация и отключение авто-конфигураций

## Точечное выключение: `spring.autoconfigure.exclude` или `@EnableAutoConfiguration(exclude = …)`

Иногда проще полностью исключить автоконфиг: например, если вы строите чистый WebFlux и не хотите, чтобы MVC-ветка даже пыталась активироваться, или когда штатный конфиг мешает корпоративной политике. Для этого есть два эквивалентных механизма: свойство `spring.autoconfigure.exclude` и параметр `exclude` у `@EnableAutoConfiguration`/`@SpringBootApplication`.
Свойство удобно для окружений: вы можете выключить конфиг на dev-стендах или в конкретном сервисе без перепаковки кода. Аннотация — для постоянных контрактов: «этот сервис никогда не поднимает MVC». Оба решения читаемы и легко ищутся.
Выключение работает на уровне класса автоконфига: вы указываете его FQCN. Чтобы узнать нужное имя, смотрите отчёт условий или файл `AutoConfiguration.imports` в зависимостях.
Не переусердствуйте: часто достаточно флага `*.enabled=false` или объявления своего `@Bean`, чтобы дефолт отступил. Полное исключение полезно, когда автоконфиг делает слишком много «за вас» и вы намерены управлять областью полностью.
Имейте в виду, что исключение автоконфига может вырубить и транзитивные бины, на которые вы рассчитывали (например, `Jackson` настройки). Проверьте `/actuator/beans` или отчёт условий, чтобы не остаться без критичных компонентов.
Добавьте smoke-тест на старт контекста после исключения: `ApplicationContextRunner` быстро покажет, что контекст поднимается и содержит ожидаемые бины.

**Код (Java/Kotlin): выключение через свойство и через аннотацию**

```yaml
# application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

```java
@SpringBootApplication(exclude = {org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.class})
public class NoJdbcApp { public static void main(String[] args){ org.springframework.boot.SpringApplication.run(NoJdbcApp.class,args);} }
```

```kotlin
@SpringBootApplication(exclude = [org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration::class])
class NoJdbcApp
fun main(args: Array<String>) = org.springframework.boot.runApplication<NoJdbcApp>(*args)
```

---

## Перекрытие бинов: объявите свой `@Bean` совместимого типа — автоконфиг «отступит»

Наиболее частый способ кастомизации — предоставить собственный бин того же контракта. Благодаря `@ConditionalOnMissingBean` в автоконфиге ваш бин «побеждает», а дефолт не создаётся. Это точечная, безопасная и прозрачная замена.
Важный момент — тип и имя. По умолчанию условия проверяют тип; если автоконфиг объявляет бин по имени, можно использовать то же имя (`@Bean(name = "…")`) или `@Primary`, когда кандидатов несколько. Документация автоконфига обычно указывает, как именно выполняется бэкоф.
Перекрывать имеет смысл, когда меняется не пару свойств, а «философия»: особая политика ретраев, альтернативная сериализация, иной пул соединений. Когда различия минимальны, сначала попробуйте свойства модуля.
В коллекциях (например, список конвертеров/перехватчиков) перекрытие — это добавление, а не замена. Порядок задают `@Order`/`Ordered`; вы можете вставить свою реализацию выше дефолтов.
Следите, чтобы ваш бин оставался совместимым с остальной экосистемой: например, кастомный `ObjectMapper` должен иметь зарегистрированные модули JavaTime/ParameterNames, иначе ломаются стандартные сценарии.
Сделайте тест на старт контекста с вашим перекрытием: это дешёвая гарантия, что дефолт отступил, а остальной граф остался здоровым.

**Код (Java): замена дефолтного `ObjectMapper` и добавление кастомайзера**

```java
package override.sample;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class JsonOverrideConfig {
  @Bean ObjectMapper objectMapper() {
    return new ObjectMapper().registerModule(new JavaTimeModule());
  }
}
```

**Код (Kotlin): аналог + пример списка кастомайзеров**

```kotlin
package override.sample

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order

@Configuration
class JsonOverrideConfig {
    @Bean fun objectMapper(): ObjectMapper = ObjectMapper().registerModule(JavaTimeModule())
}

fun interface RestClientCustomizer { fun customize(b: org.springframework.web.client.RestClient.Builder) }

@Order(0)
class RetryCustomizer : RestClientCustomizer { override fun customize(b: org.springframework.web.client.RestClient.Builder) { /* set retries */ } }
```

---

## Тюнинг через свойства: включение/выключение подсекций автоконфига, явные дефолты

Многие автоконфиги дают десятки «тонких» настроек через `@ConfigurationProperties`. Это лучший канал кастомизации: безопасные дефолты уже подобраны, вы меняете только нужные параметры. Например, таймауты в `RestClient`, буферы в кодеках, пути health-проб в Actuator.
Используйте IDE-подсказки (метаданные), чтобы не охотиться за ключами в исходниках. Хорошие библиотеки документируют префиксы и дают javadoc в метаданных.
Флаги `*.enabled` — явные переключатели: включают/выключают части автоконфига (фильтры, интеграции, фоновые задачи). Предпочтительно отключать «тяжёлые» подсекции по умолчанию и включать осознанно.
Свойства работают на всех окружениях одинаково: baseline в `application.yml`, profile-оверрайды в `application-prod.yml`, разовые изменения через переменные окружения/CLI. Это даёт удобный и безопасный процесс эксплуатации.
Следите за коллизиями: если свойство перекрывается в нескольких местах, выигрывает источник с большим приоритетом. Включите отчёт условий/источников при отладке или используйте методический «трассировщик» `Environment` на старте.
Договоритесь в команде о «реестре» ключей: какие считаются обязательными, какие секретны, где лежат дефолты. Это уменьшает сюрпризы при апгрейдах.

**Код (Java/Kotlin): настройка через свойства**

```yaml
my:
  http:
    enabled: true
    timeout-ms: 1500
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

```java
// бин из вашего автоконфига уже читает MyHttpProps; меняем только YAML.
```

```kotlin
// аналогично: код не меняется, вся кастомизация — в свойствах.
```

# 8. Тестирование авто-конфигураций

## Изолированные тесты без полного контекста: `ApplicationContextRunner` + `AutoConfigurations.of(...)`

Изолированные тесты автоконфигов — способ проверить поведение без запуска «всего приложения». Класс `ApplicationContextRunner` из Spring Boot Test поднимает крошечный контекст по нужной конфигурации, а затем даёт fluently-API для ассертов. Главное преимущество — скорость и управляемость: вы создаёте только те бины, которые действительно нужны кейсу.

Типичный сценарий: «если включён флаг и класс в classpath — бин появился». Для этого у `ApplicationContextRunner` есть методы `withPropertyValues(...)` и `withConfiguration(AutoConfigurations.of(...))`. Первый задаёт свойства, второй подключает тестируемый `@AutoConfiguration`. Результат — предсказуемые условия, которые легко варьировать по кейсам.

Такой подход уничтожает флейки. Вы изолируете влияние внешних конфигов, случайных профилей и «поднятых» компонентов. Любая зависимость указывается явно: либо через свойства, либо через подмешивание конфигурационных классов. В итоге тест концентрируется на условиях и бинах, а не на побочных эффектах.

Важно и то, что runner даёт понятные сообщения: если бин не найден — печатается список доступных. Это помогает быстро понять, что именно «не доехало». В связке с отчётом условий (в Dev-режиме) отладка становится почти механической.

Пишите «микро-тесты» на каждую ветку условий: «включено по флагу», «выключено по флагу», «нет нужного класса», «есть пользовательский бин (бэкоф)». Такие тесты короткие, читаемые и хорошо локализуют регресс.

Под конец — практический совет. Не подключайте случайно `@SpringBootTest` для автоконфигов. Это втягивает много лишнего и размывает фокус. Для автоконфигов почти всегда достаточно `ApplicationContextRunner`.

**Код (Java): изолированный тест автоконфига**

```java
package testing.ac;

import my.http.autoconf.MyHttpAutoConfiguration;
import my.http.autoconf.MyHttpProps;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.web.client.RestClient;

import static org.assertj.core.api.Assertions.assertThat;

class MyHttpAutoConfigurationTest {

  private final ApplicationContextRunner runner = new ApplicationContextRunner()
      .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration.class));

  @Test void enabledByDefault() {
    runner.run(ctx -> {
      assertThat(ctx).hasSingleBean(RestClient.class);
      assertThat(ctx).hasSingleBean(MyHttpProps.class);
    });
  }

  @Test void disabledByProperty() {
    runner.withPropertyValues("my.http.enabled=false").run(ctx -> {
      assertThat(ctx).doesNotHaveBean(RestClient.class);
    });
  }
}
```

**Код (Kotlin): аналог с AssertJ**

```kotlin
package testing.ac

import my.http.autoconf.MyHttpAutoConfiguration
import my.http.autoconf.MyHttpProps
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.autoconfigure.AutoConfigurations
import org.springframework.boot.test.context.runner.ApplicationContextRunner
import org.springframework.web.client.RestClient

class MyHttpAutoConfigurationKtTest {

    private val runner = ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration::class.java))

    @Test
    fun `enabled by default`() {
        runner.run { ctx ->
            assertThat(ctx).hasSingleBean(RestClient::class.java)
            assertThat(ctx).hasSingleBean(MyHttpProps::class.java)
        }
    }

    @Test
    fun `disabled by property`() {
        runner.withPropertyValues("my.http.enabled=false").run { ctx ->
            assertThat(ctx).doesNotHaveBean(RestClient::class.java)
        }
    }
}
```

Gradle (Groovy/Kotlin) — зависимости для раннера:

```groovy
dependencies {
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

```kotlin
dependencies {
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## Проверка условий: присутствие классов, значений свойств, бэкоф при наличии пользовательского бина

Условия «по классам» проверяются с помощью `FilteredClassLoader`: так мы имитируем отсутствие библиотеки в classpath. Это надёжно воспроизводит «тонкие» сценарии и предотвращает случайные регрессы: если завтра транситивная зависимость скрытно вернётся, тест вас предупредит.

Проверка `@ConditionalOnProperty` — простая: разверните тест с `withPropertyValues(...)` и без них. Хорошая практика — указывать дефолт в тесте явно, чтобы не зависеть от текущих начальных значений (например, `matchIfMissing`).

Для «бэкофа» используем подмешивание пользовательского `@Configuration` с «своим» бином. Runner имеет метод `withUserConfiguration(...)`, который добавляет конфиг в контекст до автоконфигов. Если ваш автоконфиг корректно обёрнут `@ConditionalOnMissingBean`, дефолтный бин не создастся.

Проверяйте и «сочетания» условий: класс есть, но флаг выключен; класс отсутствует, но флаг включен; объявлен пользовательский бин — автоконфиг отступил. Покрытие этих случаев — гарантия предсказуемости.

Ещё один полезный приём — проверка коллекций: если ваш автоконфиг собирает `List<Customizer>`, добавьте свой кастомайзер пользователем и убедитесь в нужном порядке (`@Order`). Это документирует SPI вашего модуля.

Помните, что runner не тянет весь автоконфиг Boot’а. Если ваш модуль зависит от инфраструктуры (например, `MeterRegistry`), либо создайте её «заглушку» в тесте, либо подключите соответствующую автоконфигурацию в `withConfiguration(...)`.

**Код (Java): FilteredClassLoader + бэкоф**

```java
package testing.cond;

import my.http.autoconf.MyHttpAutoConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.FilteredClassLoader;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;

import static org.assertj.core.api.Assertions.assertThat;

class ConditionsTest {

  @Test void noRestClientClass_skipsConfig() {
    new ApplicationContextRunner()
      .withClassLoader(new FilteredClassLoader(RestClient.class))
      .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration.class))
      .run(ctx -> assertThat(ctx).doesNotHaveBean(RestClient.class));
  }

  @Test void userBeanBackoff() {
    new ApplicationContextRunner()
      .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration.class))
      .withUserConfiguration(UserOverride.class)
      .run(ctx -> {
        assertThat(ctx).hasSingleBean(RestClient.class);
        assertThat(ctx).getBean(RestClient.class).isSameAs(ctx.getBean("userRestClient"));
      });
  }

  @Configuration static class UserOverride {
    @Bean RestClient userRestClient() { return RestClient.create(); }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package testing.cond

import my.http.autoconf.MyHttpAutoConfiguration
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.autoconfigure.AutoConfigurations
import org.springframework.boot.test.context.FilteredClassLoader
import org.springframework.boot.test.context.runner.ApplicationContextRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.client.RestClient

class ConditionsKtTest {

    @Test
    fun `no RestClient class - skips config`() {
        ApplicationContextRunner()
            .withClassLoader(FilteredClassLoader(RestClient::class.java))
            .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration::class.java))
            .run { ctx -> assertThat(ctx).doesNotHaveBean(RestClient::class.java) }
    }

    @Test
    fun `user bean backoff`() {
        ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration::class.java))
            .withUserConfiguration(UserOverride::class.java)
            .run { ctx ->
                assertThat(ctx).hasSingleBean(RestClient::class.java)
                assertThat(ctx.getBean(RestClient::class.java)).isSameAs(ctx.getBean("userRestClient"))
            }
    }

    @Configuration
    class UserOverride {
        @Bean fun userRestClient(): RestClient = RestClient.create()
    }
}
```

---

## Контрактные тесты модулей: стабильные имена свойств, метаданные, отсутствие неожиданных бинов в контексте

Контрактные тесты фиксируют «внешний» API вашего стартера: префиксы свойств, дефолты, состав бинов. Их задача — не допустить «тихих» изменений, ломающих потребителей при апгрейде. Простая форма — smoke-тест на «поднялся контекст и есть ровно такие бины».

Проверьте набор экспортируемых бинов: «в приложении без пользовательских переопределений мои автоконфиги создают A, B, C — и ничего лишнего». Это полезно, чтобы случайно не добавить «тяжёлый» компонент. Если вы добавляете новый бин — тест падает и заставляет вас обновить контракт осознанно.

Стабильные имена свойств можно проверить «через обратную сторону»: runner с `withPropertyValues("my.http.timeout-ms=1500")` должен успешно подняться, а попытка установить «переименованный» ключ — вызывать ошибку валидации. Это особенно важно после рефакторинга классов свойств.

Метаданные (`spring-configuration-metadata.json`) тоже стоит проверять: интеграционный тест, который распаковывает JAR и проверяет, что ключи присутствуют, а описания не пустые. Это подстраховка против «забыли подключить configuration-processor».

Полезна и негативная проверка: «без моего стартера в чистом приложении этих бинов быть не должно». Так вы заменяете «магическое» появление компонентов явной зависимостью на стартер.

Контрактные тесты — ваш «документ в коде». В README пишете «стартёр даёт X и Y, ключи такие-то», а тест гарантирует, что это правда.

**Код (Java): контракт — ровно один бин и корректный префикс**

```java
package testing.contract;

import my.hello.autoconf.Hello;
import my.hello.autoconf.HelloAutoConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;

import static org.assertj.core.api.Assertions.assertThat;

class HelloContractTest {
  @Test void exactlyOneHelloBean() {
    new ApplicationContextRunner()
      .withConfiguration(AutoConfigurations.of(HelloAutoConfiguration.class))
      .run(ctx -> {
        assertThat(ctx).hasSingleBean(Hello.class);
        assertThat(ctx.getBeanNamesForType(Hello.class)).containsExactly("hello");
      });
  }
}
```

**Код (Kotlin): smoke + свойство**

```kotlin
package testing.contract

import my.http.autoconf.MyHttpAutoConfiguration
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.autoconfigure.AutoConfigurations
import org.springframework.boot.test.context.runner.ApplicationContextRunner
import org.springframework.web.client.RestClient

class HttpContractKtTest {
    @Test
    fun `property prefix works`() {
        ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(MyHttpAutoConfiguration::class.java))
            .withPropertyValues("my.http.timeout-ms=2000")
            .run { ctx -> assertThat(ctx).hasSingleBean(RestClient::class.java) }
    }
}
```

Gradle (Groovy/Kotlin) — проверка наличия метаданных (простой task):

```groovy
tasks.register('checkConfigMetadata') {
  dependsOn 'jar'
  doLast {
    def jarFile = file("${buildDir}/libs/${project.name}-${project.version}.jar")
    assert new java.util.zip.ZipFile(jarFile).entries().any { it.name.endsWith('spring-configuration-metadata.json') }
  }
}
```

```kotlin
tasks.register("checkConfigMetadata") {
    dependsOn("jar")
    doLast {
        val jar = file("${buildDir}/libs/${project.name}-${project.version}.jar")
        java.util.zip.ZipFile(jar).use { z ->
            require(z.entries().toList().any { it.name.endsWith("spring-configuration-metadata.json") }) {
                "No configuration metadata found"
            }
        }
    }
}
```

---

# 9. Взаимодействие с AOT/Native Image и производительностью старта

## Почему условия важнее, чем «жёсткая» регистрация: меньше бинов — быстрее старт/меньше память

Скорость старта — это не только GraalVM. Даже на JVM основную цену платят за создание бинов, инициализацию и проксирование. Чем меньше бинов вы регистрируете, тем быстрее поднимается контекст и тем меньше памяти тратится на структуры контейнера. Поэтому условия в автоконфигах — не «бюрократия», а ключевой инструмент производительности.

Жёсткая регистрация «на всякий случай» приводит к «жирным» контекстам. Бины создаются, держат ссылки, вешают хуки shutdown — и всё это без реальной пользы. Условия `OnClass/OnProperty/OnBean` заранее отсеивают ненужные куски и уменьшают работу контейнера. Вы выигрываете и во времени, и в размерах heap’а.

Ещё один аспект — постпроцессоры. Каждый созданный бин проходит через цепочку `BeanPostProcessor` (AOP, валидация, биндинг), что умножает издержки на количество. Сократив число бинов, вы автоматически снижаете стоимость постобработки.

Особенно заметно это на микросервисах, где «переподъём» случается часто (автоскейлинг, blue/green). Секунды старта складываются в минуты недоступности под нагрузкой. «Тонкие» автоконфиги — прямо вклад в SLO.

Наконец, условия отлично сочетаются с профилями. Вы можете держать «набор тяжёлых» бинов для dev (например, детальная диагностика, доступные эндпоинты) и выключать их в prod, не меняя код. Это и быстрее, и безопаснее.

Практический вывод: любые «дорогие» фичи — под флаг `*.enabled=false` и `@ConditionalOnClass`. Всё, что зависит от внешней инфраструктуры, — под `@ConditionalOnBean`. Жёсткая регистрация — только для крошечных и действительно базовых компонентов.

**Код (Java): дорогой компонент включаем только по условиям**

```java
package perf.lean;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnProperty(prefix = "heavy", name = "enabled", havingValue = "true", matchIfMissing = false)
@ConditionalOnClass(name = "com.acme.expensive.Engine")
public class HeavyFeatureAutoConfig {

  @Bean
  @ConditionalOnMissingBean
  public Object heavyEngine() {
    // создание дорогого объекта (кэш, планировщик и т.д.)
    return new Object();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package perf.lean

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean

@AutoConfiguration
@ConditionalOnProperty(prefix = "heavy", name = ["enabled"], havingValue = "true", matchIfMissing = false)
@ConditionalOnClass(name = ["com.acme.expensive.Engine"])
class HeavyFeatureAutoConfig {
    @Bean @ConditionalOnMissingBean
    fun heavyEngine(): Any = Any()
}
```

---

## AOT-генерация: влияние автоконфига на подсказки рефлексии/ресурсов/динамических прокси

В нативной сборке GraalVM запрещает «сюрпризы» во время выполнения. Всё, что вы делаете рефлексией или через динамические прокси, должно быть объявлено заранее. Автоконфиг — лучшее место для этих деклараций, потому что именно он решает, будет ли фича активна.

Механизм Spring AOT предоставляет `RuntimeHints` и `@ImportRuntimeHints`. В `RuntimeHintsRegistrar` вы регистрируете типы для рефлексии (конструкторы/поля/методы), шаблоны ресурсов (например, `classpath:/my/templates/*.ftl`) и JDK-прокси для интерфейсов. Делайте хинты «узкими»: только то, что реально используется при сработавших условиях.

Отсутствие хинтов проявляется в нативной сборке падением во время рантайма или во время image-build. Лучше ловить это на CI: заведите sample-приложение, которое подключает ваш стартер, и собирайте его нативно хотя бы как smoke. Так вы заметите, что забыли зарегистрировать, например, модели Jackson или ресурсные шаблоны.

АOT также «уплощает» конфигурации: многие рефлексивные операции (в т.ч. биндинг свойств) становятся статически сгенерированными. Отсюда ещё один плюс «тонких» автоконфигов: чем меньше динамики (SpEL, класслоадинг, анонимные прокси), тем проще AOT.

Учитывайте условия при регистрации хинтов. В `registerHints(...)` доступен `ClassLoader`, можно проверять наличие класса и регистрировать хинты условно. Это позволяет не загрязнять нативный образ лишними декларациями.

Документируйте ограничение: «эта часть работает только на JVM, потому что библиотека X не совместима с Native». Иногда честно выключить фичу под `native.enabled=false` лучше, чем тащить боль в прод.

**Код (Java): hints для ресурсов и типов**

```java
package aot.hints;

import org.springframework.aot.hint.*;
import org.springframework.context.annotation.ImportRuntimeHints;

@ImportRuntimeHints(MyHints.class)
public class MyNativeAutoConfiguration { /* @AutoConfiguration + бины */ }

class MyHints implements RuntimeHintsRegistrar {
  @Override public void registerHints(RuntimeHints hints, ClassLoader cl) {
    hints.resources().registerPattern("my/templates/*.ftl");
    hints.reflection().registerType(
      com.fasterxml.jackson.databind.ObjectMapper.class,
      MemberCategory.INTROSPECT_PUBLIC_METHODS
    );
    hints.proxies().registerJdkProxy(aot.hints.MySpi.class);
  }
}

interface MySpi { void apply(); }
```

**Код (Kotlin): аналог**

```kotlin
package aot.hints

import org.springframework.aot.hint.MemberCategory
import org.springframework.aot.hint.RuntimeHints
import org.springframework.aot.hint.RuntimeHintsRegistrar
import org.springframework.context.annotation.ImportRuntimeHints

@ImportRuntimeHints(MyHints::class)
class MyNativeAutoConfiguration

class MyHints : RuntimeHintsRegistrar {
    override fun registerHints(hints: RuntimeHints, classLoader: ClassLoader?) {
        hints.resources().registerPattern("my/templates/*.ftl")
        hints.reflection().registerType(
            com.fasterxml.jackson.databind.ObjectMapper::class.java,
            MemberCategory.INTROSPECT_PUBLIC_METHODS
        )
        hints.proxies().registerJdkProxy(MySpi::class.java)
    }
}

interface MySpi { fun apply() }
```

Gradle (Groovy/Kotlin) — GraalVM native (пример):

```groovy
plugins { id 'org.graalvm.buildtools.native' version '0.10.2' }
```

```kotlin
plugins { id("org.graalvm.buildtools.native") version "0.10.2" }
```

---

## Практика «тонких» авто-конфигов: без тяжёлой логики, ранняя проверка доступности зависимостей, ленивые бины только там, где оправдано

«Тонкий» автоконфиг — это про декларации, а не про бизнес-логику. В нём описываются **условия** и **связи**: какие бины и как создавать. Любые тяжёлые операции (скан каталога, сетевые пинги, длительный warmup) должны жить в самих компонентах и запускаться контролируемо (после `ApplicationReadyEvent`, по таймеру, вручную).

Ранняя проверка доступности — это условия `OnClass/OnBean/OnProperty` и, при необходимости, валидация свойств. Если нет `DataSource`, не создавайте JdbcTemplate; если `enabled=false`, не поднимайте планировщик. Это уважает ресурсы и снижает шанс «висеть на старте».

Ленивые бины (`@Lazy`) — инструмент, а не панацея. Хорошо подходит для редкопользуемых интеграций (например, дополнительный экспорт метрик), но вреден для базовой инфраструктуры (клиент БД/HTTP). Ленивая инициализация сдвигает проблемы «в рантайм» и усложняет диагностику — включайте её осознанно.

Ещё один приём — прогрев «медленных» частей фоново после старта: `ApplicationRunner` или `@EventListener(ApplicationReadyEvent)`. Так вы не задерживаете готовность сервиса, но достигаете быстрых ответов на первые запросы.

Не пытайтесь «спасти» тяжёлый автоконфиг `@DependsOn` и хаками порядка. В 99% случаев решение — разделить конфиг, ужесточить условия и убрать побочные эффекты из инициализации.

Финально — измеряйте. Включите тайминги старта (логируйте время на фазах, счётчики созданных бинов) и держите регрессионный тест на «время старта» для образца приложения. Падение на 20–30% — сигнал, что какой-то автоконфиг стал «толстым».

**Код (Java): тяжёлая часть уезжает в раннер, автоконфиг — тонкий**

```java
package thin.good;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.boot.ApplicationRunner;

@AutoConfiguration
@ConditionalOnProperty(prefix = "feature.cache", name = "enabled", havingValue = "true")
public class CacheAutoConfig {

  @Bean @ConditionalOnMissingBean CacheClient cacheClient() { return new CacheClient(); }

  @Bean
  @ConditionalOnBean(CacheClient.class)
  ApplicationRunner cacheWarmup(CacheClient c) {
    return args -> c.warmup(); // фоновый прогрев после старта
  }

  static class CacheClient { void warmup() {/*...*/} }
}
```

**Код (Kotlin): аналог**

```kotlin
package thin.good

import org.springframework.boot.ApplicationRunner
import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean

@AutoConfiguration
@ConditionalOnProperty(prefix = "feature.cache", name = ["enabled"], havingValue = "true")
class CacheAutoConfig {
    @Bean @ConditionalOnMissingBean
    fun cacheClient() = CacheClient()

    @Bean
    @ConditionalOnBean(CacheClient::class)
    fun cacheWarmup(c: CacheClient) = ApplicationRunner { c.warmup() }
}

class CacheClient { fun warmup() { /* ... */ } }
```

---

# 10. Анти-паттерны и гайдлайны по дизайну

## Не подменяйте пользовательские бины молча; всегда делайте «бэкоф» и явные свойства-переключатели

Главный анти-паттерн — регистрировать бины «поверх» пользовательских. Если ваш автоконфиг создаёт бин без `@ConditionalOnMissingBean`, он может неожиданно перезаписать реализацию, от которой зависит приложение. Итог — трудные для диагностики баги и невозможность кастомизации.

Используйте «вежливые» условия везде, где есть шанс конкуренции: `OnMissingBean` по типу и/или имени. Если ваш дефолт должен легко отключаться, добавьте `@ConditionalOnProperty(prefix="...", name="enabled", havingValue="true")`. Это явная точка выключения в конфиге.

Документируйте, как переопределять. Если автоконфиг ожидает бин с конкретным именем — скажите об этом в README. Если достаточно типа — так и укажите. Прозрачные правила экономят часы интеграции.

Иногда хочется «ушлёпать» все пользовательские бины своим «совершенным» по умолчанию. Не делайте так. Если вы хотите «улучшить» поведение, оформляйте это как декоратор по `@ConditionalOnBean` или SPI-кастомайзер: коллекция `List<Customizer>` применится к уже существующим бинам.

Для «опасных» фич — выключатель по умолчанию. Логирование тела запросов, тяжёлые планировщики, экспортеры секретов — всё это должно включаться осознанно флагом, а не приезжать «в нагрузку».

Тест «бэкофа» обязателен. В контрактных тестах проверяйте кейс «пользовательский бин уже есть — дефолт не создаётся». Это страховка на будущее.

**Код (Java): плохо/хорошо**

```java
// ПЛОХО: перезапишет пользовательский бин
//@AutoConfiguration
//public class BadAutoConfig {
//  @Bean public ObjectMapper objectMapper() { return new ObjectMapper(); }
//}

// ХОРОШО: вежливый бэкоф + выключатель
@org.springframework.boot.autoconfigure.AutoConfiguration
@org.springframework.boot.autoconfigure.condition.ConditionalOnProperty(prefix="json", name="enabled", havingValue="true", matchIfMissing = true)
public class GoodAutoConfig {
  @org.springframework.context.annotation.Bean
  @org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
  com.fasterxml.jackson.databind.ObjectMapper objectMapper() {
    return new com.fasterxml.jackson.databind.ObjectMapper().findAndRegisterModules();
  }
}
```

**Код (Kotlin): аналог**

```kotlin
@org.springframework.boot.autoconfigure.AutoConfiguration
@org.springframework.boot.autoconfigure.condition.ConditionalOnProperty(prefix = "json", name = ["enabled"], havingValue = "true", matchIfMissing = true)
class GoodAutoConfigKt {
    @org.springframework.context.annotation.Bean
    @org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
    fun objectMapper(): com.fasterxml.jackson.databind.ObjectMapper =
        com.fasterxml.jackson.databind.ObjectMapper().findAndRegisterModules()
}
```

---

## Не тащите «всё и сразу»: дробите на части и включайте по условиям; избегайте циклов зависимостей

Автоконфиг «комбайн» с десятками бинов и условной логикой трудно читать и поддерживать. Гораздо лучше разнести по зонам ответственности: web, data, clients, observability. Каждый класс — маленький, с простыми условиями и минимальными связями.

Дробление снижает риск циклов. Циклы часто появляются, когда два бина из одного большого конфига зависят друг от друга через конструкторы, а автор пытается «лечить» это `@Lazy`. Правильнее пересмотреть дизайн: вынести общую часть, заменить жёсткую зависимость на `ObjectProvider` или интерфейс SPI.

Условия — естественный способ «подсушить» граф: если нет `DataSource`, не создавайте репозитории; если приложение реактивное, не поднимайте MVC-фильтры. Это автоматически предотвращает «левые» связи.

Стремитесь к **отсутствию** `@DependsOn` в автоконфиге. Порядок регистрации обеспечивается `before/after` на уровнях конфигов, а сами бины должны быть устойчивы к порядку — через условия и опциональную инъекцию.

Не добавляйте скрытые зависимости из пользовательского графа. Если бин реально опционален — ищите его через `ObjectProvider` и действуйте, только если он есть. Иначе автоконфиг «залипнет» на чужих деталях.

В тестах держите «наборы сценариев» для каждого куска: так вы заметите, что добавление новой фичи неожиданно потащило веб-бин в сервис без web-зависимости.

**Код (Java): разрыв цикла через `ObjectProvider`**

```java
package antipattern.cycle;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Component;

// Было: ServiceA -> ServiceB -> ServiceA (конструкторный цикл)
// Стало: одна сторона опциональная, обращение по требованию

@Component class ServiceA {
  private final ObjectProvider<ServiceB> b;
  ServiceA(ObjectProvider<ServiceB> b){ this.b = b; }
  void doA(){ b.ifAvailable(ServiceB::assist); }
}
@Component class ServiceB {
  private final ServiceA a;
  ServiceB(ServiceA a){ this.a = a; }
  void assist(){ /* ... */ }
}
```

**Код (Kotlin): аналог**

```kotlin
package antipattern.cycle

import org.springframework.beans.factory.ObjectProvider
import org.springframework.stereotype.Component

@Component
class ServiceA(private val b: ObjectProvider<ServiceB>) {
    fun doA() { b.ifAvailable { it.assist() } }
}

@Component
class ServiceB(private val a: ServiceA) {
    fun assist() { /* ... */ }
}
```

---

## Не скрывайте побочные эффекты (миграции/сетевые подключения) в статических инициализаторах; всё через управляемые бины

Статические инициализаторы и блоки `init` — плохое место для работы с сетью или диском. Они запускаются при загрузке класса, ещё до того, как Spring сможет управлять жизненным циклом, обработать ошибки и корректно остановиться. Результат — «фантомные» подключения и проблемные откаты.

Все побочные эффекты делайте в управляемых бинах: через конструктор/`@PostConstruct`/инит-метод — если действительно нужно до `ready`; или через `ApplicationRunner`/события — если можно отложить. Так вы получите контроль над порядком, логированием и ошибками.

Миграции БД — не в статике и не в автоконфиге произвольно. Либо используйте стандартные инструменты (Flyway/Liquibase автоконфиги), либо оформляйте отдельный бин-джобу с понятным включателем и политикой запуска. Это делает поведение прозрачным для эксплуатации.

Выделяйте сети в отдельные компоненты с таймаутами и ретраями. Статический вызов «сходить в конфиг-сервер» при загрузке класса ломает старт на ноутбуке, в офлайне и в нативной сборке (AOT не умеет симулировать такие произвольные вызовы).

В отчётах условий не должно быть «побочек». Автоконфиг лишь описывает, какие бины создать. Все операции — в runtime по управляемым правилам (init/runner/scheduler). Это облегчает и AOT, и диагностику.

Если нужна «синхронная» предподготовка, используйте `SmartLifecycle` с фазами: это дало Spring’у возможность корректно остановить компонент и дождаться его завершения.

**Код (Java): плохо/хорошо**

```java
// ПЛОХО
class Bad {
  static {
    try { new java.net.URL("https://config.example").openStream().close(); }
    catch (Exception e) { throw new RuntimeException(e); }
  }
}

// ХОРОШО
@org.springframework.stereotype.Component
class GoodInit implements org.springframework.boot.ApplicationRunner {
  @Override public void run(org.springframework.boot.ApplicationArguments args) throws Exception {
    try (var s = new java.net.URL("https://config.example").openStream()) { /* ... */ }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
// Плохо — статический побочный эффект
class Bad {
    companion object {
        init { java.net.URL("https://config.example").openStream().use { } }
    }
}

// Хорошо — управляемый раннер
@org.springframework.stereotype.Component
class GoodInit : org.springframework.boot.ApplicationRunner {
    override fun run(args: org.springframework.boot.ApplicationArguments) {
        java.net.URL("https://config.example").openStream().use { /* ... */ }
    }
}
```

---

## Документируйте: префиксы свойств, дефолты, сценарии отключения, ожидаемая интеграция с другими автоконфигами

Хорошая документация стартера экономит десятки часов потребителям. Минимум: префикс свойств, таблица ключей с типами и дефолтами, примеры YAML, способы отключения/переопределения (`*.enabled`, `spring.autoconfigure.exclude`, пользовательский `@Bean`). Всё это дублируется и в метаданных, и в README.

Опишите «границы влияния»: какие бины регистрируются, какие зависят от других автоконфигов (например, «если есть `MeterRegistry`, регистрируются метрики»). Укажите, как ваш модуль реагирует на профили и тип web-приложения (MVC/WebFlux).

Метаданные конфигурации генерируйте обязательно: `spring-boot-configuration-processor` в зависимостях, Javadoc на полях — всё это прямо попадает в подсказки IDE. Потребитель увидит правильные ключи и допустимые значения ещё в YAML.

Поддерживайте «changelog для ключей»: при переименовании — добавляйте deprecated-ключи и миграционные подсказки. Контрактные тесты на «старый ключ не работает без явной миграции» защитят от непреднамеренных ломающих изменений.

Покажите «рецепты» интеграции: короткие сниппеты «как заменить бин», «как выключить подсекцию», «как добавить кастомайзер». Это снижает порог входа и уменьшает поток вопросов.

И наконец, держите sample-приложение в репозитории стартера. Запустив его, потребитель увидит ожидаемые бины (`/actuator/beans`) и условия (`/actuator/conditions`), а вы — получите «живой» стенд для регрессии.

**Код (Java): свойства с Javadoc → метаданные**

```java
package docs.props;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.util.unit.DataSize;

@ConfigurationProperties("corp.http")
public record CorpHttpProps(
  /** Таймаут соединения, миллисекунды. Должен быть >=100. */
  int connectTimeoutMs,
  /** Максимальный размер тела ответа для логирования. Пример: 512KB. */
  DataSize logBodyLimit,
  /** Включить метрики клиента (Micrometer). */
  boolean metricsEnabled
) { }
```

**Код (Kotlin): аналог**

```kotlin
package docs.props

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.util.unit.DataSize

/** Настройки корпоративного HTTP-клиента. */
@ConfigurationProperties("corp.http")
data class CorpHttpProps(
    /** Таймаут соединения, мс (>=100). */
    val connectTimeoutMs: Int = 1000,
    /** Максимальный размер логируемого тела. */
    val logBodyLimit: DataSize = DataSize.ofKilobytes(512),
    /** Включить метрики клиента. */
    val metricsEnabled: Boolean = true
)
```

Gradle (Groovy/Kotlin) — процессор метаданных:

```groovy
dependencies { annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor' }
```

```kotlin
dependencies { annotationProcessor("org.springframework.boot:spring-boot-configuration-processor") }
```


