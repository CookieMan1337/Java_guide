---
layout: page
title: "Контекст, профили и конфигурационные свойства"
permalink: /spring/context
---
# 0. Введение

## Что это такое

Контекст приложения Spring (`ApplicationContext`) — это «жизненная среда» вашего кода: контейнер управляемых объектов (бинов), их зависимостей, жизненного цикла и инфраструктуры. Вместе с ним живёт `Environment` — абстракция над конфигурацией (набором источников свойств, динамических параметров и профилей). В современных приложениях на Spring Boot эта пара образует сердцевину: Boot собирает контекст, наполняет его автоконфигурациями и подмешивает конфиги из `Environment`, а затем отдаёт готовое, прогретое приложение.

Это не только про «где хранить настройки». Контекст умеет публиковать события, доставлять переводы/сообщения (`MessageSource`), конвертировать типы, сканировать компоненты и создавать прокси для аспектов (транзакции, кеш, ретраи). `Environment` решает «кто победит», если свойство определено в нескольких местах, и гарантирует детерминированное разрешение значений.

Важная деталь — **единообразие**: неважно, откуда приходит конфигурация (переменные окружения, `application.yml`, параметры командной строки, секреты из K8s) — вы читаете её одинаково, через типобезопасные классы свойств или `Environment`. Это позволяет собирать immutable-образы и конфигурировать поведение исключительно «снаружи».

В Spring Boot 3.x (Java 17+) уже зашито разумное поведение по умолчанию: единый корневой контекст, автоматическое подключение `application.yml`/`application-<profile>.yml`, предсказуемые приоритеты источников, инициализация до события `ApplicationReady`. Это упрощает ментальную модель: мы говорим о **настройках среды** и **графе бинов**, а не о «горстке XML-файлов».

Контекст — это и **границы**: он отделяет домен от инфраструктуры. Мы инжектим уже сконфигурированные клиенты/репозитории/сервисы, а не создаём их руками; конфиги — снаружи, логика — внутри. Такая композиция прямо ведёт к тестируемости и предсказуемому развёртыванию.

**Код (Java): минимальное приложение, печатающее активные профили и одно свойство**

```java
package demo.intro;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@SpringBootApplication
public class IntroApp {
  public static void main(String[] args) {
    SpringApplication.run(IntroApp.class, args);
  }
}

@Component
class PrintEnvRunner implements ApplicationRunner {
  private final Environment env;
  public PrintEnvRunner(Environment env) { this.env = env; }
  @Override public void run(ApplicationArguments args) {
    System.out.println("Active profiles: " + String.join(",", env.getActiveProfiles()));
    System.out.println("app.name=" + env.getProperty("app.name", "demo"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.intro

import org.springframework.boot.ApplicationArguments
import org.springframework.boot.ApplicationRunner
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@SpringBootApplication
class IntroApp
fun main(args: Array<String>) = runApplication<IntroApp>(*args)

@Component
class PrintEnvRunner(private val env: Environment) : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        println("Active profiles: ${env.activeProfiles.joinToString()}")
        println("app.name=${env.getProperty("app.name", "demo")}")
    }
}
```

`application.yml` (для наглядности):

```yaml
app:
  name: spring-guide
```

---

## Зачем это нужно

Главная причина — **разделение конфигурации и кода**. Мы собираем один неизменяемый образ приложения и изменяем его поведение параметрами среды. Это ускоряет релизы, снижает ошибки «несоответствия окружения», да и просто делает жизнь легче: один и тот же артефакт едет на dev, stage, prod.

Вторая — **качество и безопасность запуска**. `Environment` формируется раньше бинов, так что всё, что зависит от настроек, может упасть fail-fast при старте, а не в середине рабочего дня. Сюда же — валидация конфигурационных свойств: неверное значение таймаута, пустой URL — и приложение не стартует.

Третья — **тестируемость**. Через `@ConfigurationProperties` вы получаете типобезопасные классы, которые легко подменить в тестах или создать вручную с фиктивными значениями. Никаких хрупких `@Value("${...}")` по всему коду.

Наконец, **наблюдаемость**: единая точка управления свойствами позволяет прозрачно логировать активные профили, печатать отчёты условий (почему включилась/не включилась автоконфигурация), подключать разные политики кеша/транзакций/ретраев через флаги.

**Код (Java): fail-fast проверка критически важного свойства**

```java
package demo.why;

import org.springframework.boot.context.event.ApplicationStartingEvent;
import org.springframework.context.event.EventListener;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.stereotype.Component;

@Component
class CriticalCheck {
  @EventListener(ApplicationStartingEvent.class)
  void onStarting(ApplicationStartingEvent e) {
    ConfigurableEnvironment env = e.getSpringApplication().getEnvironment();
    String baseUrl = env.getProperty("client.base-url");
    if (baseUrl == null || baseUrl.isBlank()) {
      throw new IllegalStateException("client.base-url is required");
    }
  }
}
```

**Код (Kotlin): тот же подход**

```kotlin
package demo.why

import org.springframework.boot.context.event.ApplicationStartingEvent
import org.springframework.context.event.EventListener
import org.springframework.core.env.ConfigurableEnvironment
import org.springframework.stereotype.Component

@Component
class CriticalCheck {
    @EventListener(ApplicationStartingEvent::class)
    fun onStarting(e: ApplicationStartingEvent) {
        val env: ConfigurableEnvironment = e.springApplication.environment
        val baseUrl = env.getProperty("client.base-url")
        require(!baseUrl.isNullOrBlank()) { "client.base-url is required" }
    }
}
```

---

## Где используется

Контекст и `Environment` в действии — это любой слой: веб (порт/хост, сжатие), клиентские SDK (таймауты, TLS), интеграции (Kafka, Postgres), бизнес-фичи (включить/выключить эксперимент), профили (dev/test/prod). Практически каждый бин так или иначе зависит от настроек: напрямую или через автоконфигурации.

В тестах — особый случай. Мы включаем профиль `test`, подменяем свойства и получаем лёгкий, детерминированный запуск. В интеграционных тестах добавляем динамические параметры (например, URL из Testcontainers) — и опять же, через `Environment`, а не ручные «if» в коде.

В платформенных проектах (общие стартеры) `Environment` используется для **условий**: включить бин, если есть библиотека X и свойство `feature.x.enabled=true`. Так библиотеки ведут себя «вежливо», не навязывая поведение.

**Код (Java): использование профиля для dev-инструмента**

```java
package demo.where;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("dev")
class DevBanner {
  DevBanner() { System.out.println("== DEV MODE =="); }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.where

import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

@Component
@Profile("dev")
class DevBanner {
    init { println("== DEV MODE ==") }
}
```

---

## Какие проблемы/задачи решает

Типичные боли без контекста/`Environment`: разбросанные `System.getenv()`, хрупкие чтения `Properties`, ручное переключение окружений, непредсказуемые старты (когда часть зависимостей появляется в момент первого запроса). Модель Spring решает это детерминированным порядком, единым доступом к конфигурации и сильными контрактами (валидация, типы).

Она же помогает соблюсти 12-фактор: один билд — разные окружения, внешняя конфигурация, отсутствие локальных «if (prod)». Вы получаете воспроизводимость и быстрый rollback: поменяли переменную — не пересобирали.

И ещё — **безопасность**: секреты можно подтягивать из внешних хранилищ, а не хранить в VCS; логирование значений можно отключить; значения — маскировать. Всё это делается на уровне конфигурации, без вторжения в бизнес-код.

**Код (Java): типобезопасные свойства с валидацией — отказ от старта при ошибке**

```java
package demo.problems;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

@Validated
@Component
@ConfigurationProperties("client")
class ClientProps {
  @NotBlank String baseUrl;
  @Min(100) @Max(10_000) int timeoutMs = 500;
  public String getBaseUrl(){ return baseUrl; }
  public void setBaseUrl(String v){ this.baseUrl = v; }
  public int getTimeoutMs(){ return timeoutMs; }
  public void setTimeoutMs(int v){ this.timeoutMs = v; }
}
```

**Код (Kotlin): то же, через data-класс**

```kotlin
package demo.problems

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated

@Validated
@Component
@ConfigurationProperties("client")
data class ClientProps(
    @field:NotBlank val baseUrl: String = "",
    @field:Min(100) @field:Max(10_000) val timeoutMs: Int = 500
)
```

---

# 1. ApplicationContext и Environment: роли и границы

## Что такое `ApplicationContext`: контейнер бинов + инфраструктурные сервисы (ресолвер сообщений, событийная шина, конвертеры)

`ApplicationContext` расширяет «сырую» `BeanFactory` и добавляет инфраструктуру: международные сообщения (`MessageSource`), публикацию событий (`ApplicationEventPublisher`), преобразование типов (`ConversionService`), доступ к ресурсам (`ResourceLoader`). Он знает, **как создавать** бины (через дефиниции), **когда их инициализировать**, **какие пост-процессоры** применить и **как уничтожать** их корректно.

Важная мысль: большинство ваших классов не должны напрямую зависеть от `ApplicationContext`. Вместо этого вы просите нужные абстракции — `MessageSource`, `ApplicationEventPublisher`, `ConversionService` — через конструктор. Так вы сохраняете тестируемость и изоляцию домена от инфраструктуры.

Контекст также является «шиной событий», но только инфраструктурной. Это место для событий старта/остановки, технических сигналов; доменные события лучше оформить собственными типами и публиковать через тот же `ApplicationEventPublisher`, но не использовать это как «скрытый транспорт» между несвязанными модулями.

`MessageSource` помогает отделить **технические** и **локализованные** сообщения. Вы описываете bundle ресурсов (например, `messages.properties`), а затем просите переводы по ключу и локали. Это особенно удобно в пользовательских приложениях и при формировании ошибок API.

Наконец, `ConversionService` — невидимый герой: он конвертирует строки в типы при биндинге свойств, парсит `Duration`, `DataSize`, enum’ы, URL. Благодаря этому `@ConfigurationProperties` просто «взлетает» без ручного парсинга.

**Код (Java): инжектим инфраструктурные сервисы, не тянем ApplicationContext**

```java
package demo.ctx;

import org.springframework.context.MessageSource;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

import java.util.Locale;

@Service
public class GreetingService {
  private final MessageSource messages;
  private final ApplicationEventPublisher events;

  public GreetingService(MessageSource messages, ApplicationEventPublisher events) {
    this.messages = messages; this.events = events;
  }

  public String greet(String name, Locale locale) {
    String template = messages.getMessage("greet", null, "Hello, {0}", locale);
    events.publishEvent("greeted:" + name);
    return java.text.MessageFormat.format(template, name);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.ctx

import org.springframework.context.ApplicationEventPublisher
import org.springframework.context.MessageSource
import org.springframework.stereotype.Service
import java.text.MessageFormat
import java.util.Locale

@Service
class GreetingService(
    private val messages: MessageSource,
    private val events: ApplicationEventPublisher
) {
    fun greet(name: String, locale: Locale): String {
        val template = messages.getMessage("greet", null, "Hello, {0}", locale)
        events.publishEvent("greeted:$name")
        return MessageFormat.format(template, name)
    }
}
```

`src/main/resources/messages.properties`:

```
greet=Hello, {0}
```

`src/main/resources/messages_ru.properties`:

```
greet=Привет, {0}
```

---

## Типы контекстов в Boot 3.x: для servlet, reactive и «голого» приложения; единственный корневой контекст как норма

Spring Boot сам выберет «тип» приложения по зависимостям: если подключён `spring-boot-starter-web`, будет создан `WebApplicationContext` (servlet, Tomcat/Jetty/Undertow); если `spring-boot-starter-webflux` — `ReactiveWebApplicationContext` (Netty/WebFlux); если ни того ни другого — «голый» `ApplicationContext`. Для большинства микросервисов достаточно **одного корневого контекста** — это упрощает порядок старта и остановки.

Раньше часто использовали иерархии контекстов (родитель/дочерний) для сложных приложений. В Boot 3.x это редкость: единственный контекст проще диагностировать, а модулям достаточно логического разделения пакетов/бинов. Исключения — многохостовые web-приложения или особые сценарии много-контекстных тестов.

Выбор стека влияет на автоконфиги: servlet-приложение поднимет `DispatcherServlet`, фильтры, MVC-инфраструктуру; reactive — `HttpHandler`, функциональные эндпойнты, иные фильтры безопасности. Но и там, и там — единый `Environment`, единые профили, единый порядок разрешения свойств.

Смешивать оба веб-стартера в одном приложении нельзя: Boot ругнётся и попросит определиться (или отделить в разные процессы). Для блокирующих зависимостей и драйверов используйте servlet-ветку, для end-to-end реактивности — WebFlux.

Понимание «какой контекст у меня» полезно при настройке мониторинга, фильтров, CORS, безопасности — у стеков разные расширения и точки входа.

**Код (Java): три «вида» приложений — выбор по зависимостям**

```java
// 1) web (servlet)
@SpringBootApplication
public class MvcApp { public static void main(String[] a){ org.springframework.boot.SpringApplication.run(MvcApp.class,a);} }

// 2) reactive (WebFlux)
// Зависимость: spring-boot-starter-webflux
@SpringBootApplication
public class ReactiveApp { public static void main(String[] a){ org.springframework.boot.SpringApplication.run(ReactiveApp.class,a);} }

// 3) plain (no web)
// Зависимость: только spring-boot-starter
@SpringBootApplication
public class CliApp { public static void main(String[] a){ org.springframework.boot.SpringApplication.run(CliApp.class,a);} }
```

**Код (Kotlin): аналог**

```kotlin
@SpringBootApplication
class MvcApp
fun main(args: Array<String>) = runApplication<MvcApp>(*args)

// Для WebFlux просто меняется зависимость
```

Gradle (Groovy):

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }                 // plain
// dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }        // servlet
// dependencies { implementation 'org.springframework.boot:spring-boot-starter-webflux' }    // reactive
```

Gradle (Kotlin):

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
// implementation("org.springframework.boot:spring-boot-starter-web")
// implementation("org.springframework.boot:spring-boot-starter-webflux")
```

---

## `Environment` и `PropertySources`: где живут свойства, как происходит резолвинг и приоритеты

`Environment` хранит список `PropertySource` в **строгом порядке**: сначала аргументы командной строки, затем переменные окружения, системные свойства JVM, затем — файлы конфигурации (`application.yml/.properties`) и дополнительные источники (импорты, Config Server, `configtree:`). Поиск идёт сверху вниз, первое найденное значение «побеждает».

Каждый `PropertySource` может быть доступен напрямую (например, добавить свой на ранней фазе старта), но чаще всё делается автоматически через **Config Data API** (подробнее ниже). Вы можете в раннем слушателе добавить свою «прослойку» значений, и они станут выше, чем `application.yml`.

Есть важная деталь про **форматы**: Boot поддерживает «расслабленные» имена. Свойство `my.service-timeout` равно `my.serviceTimeout` равно `MY_SERVICE_TIMEOUT` (из окружения). Это часть стратегии унификации и переносимости.

На практике в коде вы редко просите `Environment` напрямую — лучше типобезопасные классы свойств. Но `Environment` полезен в ранних событиях (проверить наличие критического значения) и в общих утилитах (логирование активных профилей, диагностика).

**Код (Java): добавляем свой PropertySource ранним слушателем**

```java
package demo.env;

import org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.core.env.MapPropertySource;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
class ExtraSourceListener implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {
  @Override
  public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    var env = event.getEnvironment();
    var props = Map.of("banner.greeting", "Hello from extra source");
    env.getPropertySources().addFirst(new MapPropertySource("extra", props));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.env

import org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent
import org.springframework.context.ApplicationListener
import org.springframework.core.env.MapPropertySource
import org.springframework.stereotype.Component

@Component
class ExtraSourceListener : ApplicationListener<ApplicationEnvironmentPreparedEvent> {
    override fun onApplicationEvent(event: ApplicationEnvironmentPreparedEvent) {
        val env = event.environment
        val props = mapOf("banner.greeting" to "Hello from extra source")
        env.propertySources.addFirst(MapPropertySource("extra", props))
    }
}
```

---

## События старта: когда уже есть `Environment`, но ещё нет всех бинов; где можно вмешаться

Жизненный цикл Boot богат ранними событиями. Самые полезные для конфигурации:

* `ApplicationStartingEvent` — самый ранний хук: ещё нет `Environment`, можно подключить логирование, баннеры.
* `ApplicationEnvironmentPreparedEvent` — **уже есть `Environment`**, можно модифицировать/добавить `PropertySource`, проверить критические значения.
* `ApplicationPreparedEvent` — контекст создан, но бины ещё не инициализированы; можно регистрировать `BeanFactoryPostProcessor`.
* Далее — `ApplicationStartedEvent` и `ApplicationReadyEvent` (поздние фазы).

Вмешиваться стоит **минимально** и осознанно: ранний доступ — мощный, но легко превратить запуск в «магический». Проверки «обязательных» свойств, регистрация доп. локаций, включение профилей — нормальные сценарии. Сетевые вызовы, heavy-IO — нет.

Помните, что слушатели можно регистрировать как бины (как выше), так и через `SpringApplication.addListeners(...)` — второй вариант особенно удобен для библиотек/стартеров.

Для отладки включайте `debug: true` — Boot выведет отчёт условий и источники конфигурации, это часто снимает вопросы «откуда взялось это значение?».

**Код (Java): ранний слушатель, включающий профиль при условии**

```java
package demo.start;

import org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.stereotype.Component;

@Component
class AutoProfileActivator implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {
  @Override
  public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    ConfigurableEnvironment env = event.getEnvironment();
    if ("true".equals(env.getProperty("feature.experimental"))) {
      env.addActiveProfile("exp");
    }
  }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.start

import org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent
import org.springframework.context.ApplicationListener
import org.springframework.core.env.ConfigurableEnvironment
import org.springframework.stereotype.Component

@Component
class AutoProfileActivator : ApplicationListener<ApplicationEnvironmentPreparedEvent> {
    override fun onApplicationEvent(event: ApplicationEnvironmentPreparedEvent) {
        val env: ConfigurableEnvironment = event.environment
        if (env.getProperty("feature.experimental") == "true") {
            env.addActiveProfile("exp")
        }
    }
}
```

---

# 2. Источники конфигурации и порядок разрешения

## Конфигурация «снаружи»: аргументы командной строки, переменные окружения, системные свойства JVM, файлы `application.yml/.properties`

Spring Boot по умолчанию собирает `Environment` из нескольких источников. **Аргументы командной строки** (`--key=value`) имеют самый высокий приоритет — удобно для одноразовых оверрайдов. **Переменные окружения** (например, `APP_NAME=prod`) — основной механизм в контейнерах/CI. **Системные свойства JVM** (`-Dkey=value`) — полезны для локальной отладки/IDE. Затем — **файлы конфигурации** (`application.yml`/`.properties`) из стандартных локаций.

При коллизии побеждает более высокий слой (аргумент CLI «перекроет» значение из `application.yml`). Это позволяет иметь «базовый» набор в репозитории и точечные override’ы на окружениях без правки артефакта. Все значения доступны одинаково — через `Environment` или типобезопасный биндинг.

Для Docker/K8s чаще всего используются именно переменные окружения — они маппятся в свойства по правилам (см. ниже в теме 8): точки → подчёркивания, верхний регистр, преобразование имён Boot.

**Код (Java): читаем значения из разных источников**

```java
package demo.sources;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
class PrintProps implements ApplicationRunner {
  private final Environment env;
  public PrintProps(Environment env){ this.env = env; }
  @Override public void run(ApplicationArguments args) {
    System.out.println("name=" + env.getProperty("app.name"));
    System.out.println("debug=" + env.getProperty("app.debug", Boolean.class, false));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.sources

import org.springframework.boot.ApplicationArguments
import org.springframework.boot.ApplicationRunner
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
class PrintProps(private val env: Environment) : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        println("name=${env.getProperty("app.name")}")
        println("debug=${env.getProperty("app.debug", Boolean::class.java, false)}")
    }
}
```

Запуск:

```
java -jar app.jar --app.name=cli-name -Dapp.debug=true
# или APP_NAME=env-name java -jar app.jar
```

---

## «Config Data API» в Boot: автопоиск `application.yml` в classpath и в `config/`, профилированные файлы `application-<profile>.yml`

Начиная с Boot 2.4+, конфиги грузятся через **Config Data API**. Это означает:

* Boot ищет `application.yml`/`.properties` в `classpath:/`, `classpath:/config/`, `file:./`, `file:./config/`.
* Профильные файлы `application-<profile>.yml` автоматически подмешиваются при активном профиле.
* Порядок и мердж значений — предсказуемый: профильные файлы **поверх** базового.

Это также открывает путь к расширенным источникам (`spring.config.import`). Но даже в базовом виде вы получаете гибкость: можно положить общие настройки в `config/`, а локальные — рядом с приложением, не изменяя jar.

Для одного файла `application.yml` можно использовать профильные **секции** (ниже, в теме 3). Выберите то, что вам удобнее по команде/репозиторию.

**Код (Java): демонстрация чтения профилей через простую печать**

```java
package demo.configdata;

import org.springframework.boot.CommandLineRunner;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
class PrintProfiled implements CommandLineRunner {
  private final Environment env;
  PrintProfiled(Environment env){ this.env = env; }
  @Override public void run(String... args) {
    System.out.println("profiled.url=" + env.getProperty("demo.url"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.configdata

import org.springframework.boot.CommandLineRunner
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
class PrintProfiled(private val env: Environment) : CommandLineRunner {
    override fun run(vararg args: String?) {
        println("profiled.url=${env.getProperty("demo.url")}")
    }
}
```

`application.yml`:

```yaml
demo:
  url: http://localhost:8080
```

`application-prod.yml`:

```yaml
demo:
  url: https://service.internal
```

---

## Расширение локаций: `spring.config.additional-location`, импорт: `spring.config.import` (classpath:, file:, configtree:, configserver: и т.п.)

Когда стандартных локаций мало, используйте `spring.config.additional-location` (CLI/env) — Boot подмешает указанные файлы/каталоги (высокий приоритет). Для сложных случаев — директива `spring.config.import` внутри `application.yml`: можно импортировать пакеты конфигов с classpath, из файловой системы, из K8s `configtree:`, из Spring Cloud Config Server.

`configtree:` особенно удобен в Docker/K8s: когда секреты смонтированы как набор файлов (каждый ключ — отдельный файл), Boot сможет собрать из них свойства. Это надёжный способ не хранить секреты в одном плоском YAML и контролировать права на файловой системе.

Импорты выполняются **ранно** и участвуют в общем порядке разрешения. Конфигурация становится модульной: «базовые» настройки в одном файле, региональные/тенантные — через отдельные импорты в зависимости от профилей.

**Код (Java): печать значения, пришедшего из импортированного источника**

```java
package demo.imports;

import org.springframework.boot.CommandLineRunner;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
class PrintImported implements CommandLineRunner {
  private final Environment env;
  PrintImported(Environment env){ this.env = env; }
  @Override public void run(String... args) {
    System.out.println("ext.region=" + env.getProperty("ext.region", "n/a"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.imports

import org.springframework.boot.CommandLineRunner
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
class PrintImported(private val env: Environment) : CommandLineRunner {
    override fun run(vararg args: String?) {
        println("ext.region=${env.getProperty("ext.region", "n/a")}")
    }
}
```

`application.yml` с импортами:

```yaml
spring:
  config:
    import:
      - "classpath:shared.yml"
      - "file:/etc/app/tenant.yml"
      - "configtree:/etc/secrets/"
```

---

## Приоритеты свойств и переопределения: кто «побеждает» при коллизиях; типичные сценарии прод-оверайдов

Правило простое: чем «ближе» к запуску и оператору, тем выше приоритет. В типичном проде порядок такой:

1. аргументы `--key=value` (one-off override),
2. переменные окружения,
3. `-Dkey=value`,
4. импортированные источники (в т.ч. `configtree:`/Config Server),
5. `application-<profile>.yml`,
6. базовый `application.yml`.

Поэтому обычно «базовые» значения живут в репозитории, «профильные» — рядом, а секреты и чувствительные override’ы — в окружении/секрет-хранилище. В docker/k8s активные значения приезжают через env или смонтированные файлы.

Осторожнее с дублированием ключей. Старайтесь держать «источник истины» в одном месте и документировать, где ожидается override. Включите `debug: true`, чтобы увидеть **Condition Evaluation Report** и список `PropertySource` — это помогает разобраться, почему взялось именно это значение.

**Код (Java): отчёт о первом источнике значения (диагностика)**

```java
package demo.prio;

import org.springframework.boot.CommandLineRunner;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.PropertySource;
import org.springframework.stereotype.Component;

@Component
class TraceSource implements CommandLineRunner {
  private final ConfigurableEnvironment env;
  TraceSource(ConfigurableEnvironment env){ this.env = env; }
  @Override public void run(String... args) {
    String key = "demo.flag";
    for (PropertySource<?> ps : env.getPropertySources()) {
      if (ps.containsProperty(key)) {
        System.out.println("Found '"+key+"' in: " + ps.getName() + " = " + ps.getProperty(key));
        break;
      }
    }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.prio

import org.springframework.boot.CommandLineRunner
import org.springframework.core.env.ConfigurableEnvironment
import org.springframework.core.env.PropertySource
import org.springframework.stereotype.Component

@Component
class TraceSource(private val env: ConfigurableEnvironment) : CommandLineRunner {
    override fun run(vararg args: String?) {
        val key = "demo.flag"
        for (ps: PropertySource<*> in env.propertySources) {
            if (ps.containsProperty(key)) {
                println("Found '$key' in: ${ps.name} = ${ps.getProperty(key)}")
                break
            }
        }
    }
}
```

---

# 3. Профили: активация, группировка, условия

## Активация профилей: `spring.profiles.active` (env/args), `SPRING_PROFILES_ACTIVE` (env), профили по умолчанию

Профили позволяют переключать наборы бинов и конфигов. Активировать их можно:

* через аргументы `--spring.profiles.active=dev,local`,
* через переменную окружения `SPRING_PROFILES_ACTIVE=prod`,
* через системное свойство `-Dspring.profiles.active=...`.

Если ничего не задано, активен «пустой» профиль и `spring.profiles.default` (если указан). Хорошая практика — определять **дефолт** как «локальная разработка»: `spring.profiles.default=dev`, а в CI/проде активировать нужные через env.

Сами профили — логические метки; они не обязаны соответствовать окружениям. Часто выделяют технологические (`kafka`, `s3`) и поведенческие (`exp`, `perf`) профили, комбинируя их по ситуации. Главное — договориться о конвенции в команде.

Профили влияют на: выбор бинов (`@Profile`), загрузку профильных файлов конфигурации, условия автоконфигов. Это мощный инструмент, но его не стоит превращать в «карусель if-ов»: дробите разумно и документируйте.

**Код (Java): печать активных профилей и проверка флага**

```java
package demo.profiles;

import org.springframework.boot.CommandLineRunner;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
class PrintProfiles implements CommandLineRunner {
  private final Environment env;
  PrintProfiles(Environment env){ this.env = env; }
  @Override public void run(String... args) {
    System.out.println("Active: " + String.join(",", env.getActiveProfiles()));
    System.out.println("Default: " + String.join(",", env.getDefaultProfiles()));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.profiles

import org.springframework.boot.CommandLineRunner
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
class PrintProfiles(private val env: Environment) : CommandLineRunner {
    override fun run(vararg args: String?) {
        println("Active: ${env.activeProfiles.joinToString()}")
        println("Default: ${env.defaultProfiles.joinToString()}")
    }
}
```

`application.yml`:

```yaml
spring:
  profiles:
    default: dev
```

---

## Профильные файлы и секции: `application-<profile>.yml`, а также `spring.config.activate.on-profile` внутри одного файла

Есть два способа задавать профильные конфиги. Первый — отдельные файлы `application-dev.yml`, `application-prod.yml`. Второй — **секции** в одном `application.yml` с ключом `spring.config.activate.on-profile`. Секции удобны, когда вы хотите держать всё в одном файле, файлы — когда конфигов много и их проще разнести.

Профильные фрагменты **мерджатся** поверх базовых значений. Если ключ повторяется — побеждает профильный. Для коллекций/карт — тоже мердж, если не указано явное переопределение (внимательнее с YAML-якорями).

Секции можно комбинировать: одна секция — для `dev|test`, другая — для `prod`. Это экономит дублирование. Но не превращайте `application.yml` в «роман»: если файл пухнет, вынесите профили отдельно.

**Код (Java): печать профильного значения (секции)**

```java
package demo.profiled;

import org.springframework.boot.CommandLineRunner;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
class PrintSection implements CommandLineRunner {
  private final Environment env;
  PrintSection(Environment env){ this.env = env; }
  @Override public void run(String... args) {
    System.out.println("db.url=" + env.getProperty("db.url"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.profiled

import org.springframework.boot.CommandLineRunner
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
class PrintSection(private val env: Environment) : CommandLineRunner {
    override fun run(vararg args: String?) {
        println("db.url=${env.getProperty("db.url")}")
    }
}
```

`application.yml` (секции):

```yaml
db:
  url: jdbc:postgresql://localhost:5432/app

---
spring:
  config:
    activate:
      on-profile: prod
db:
  url: jdbc:postgresql://db.prod:5432/app
```

---

## Группы профилей: логические «сборки» (`spring.profiles.group.*`) для включения нескольких сразу

Иногда удобно включать целый набор профилей одним именем. Для этого есть **группы**: `spring.profiles.group.<groupName>=a,b,c`. Активируя `groupName`, вы получаете все перечисленные. Пример — `prod-eu` включает `prod`, `eu`, `kafka`, `s3`. Это уменьшает длину переменных и защищает от забывчивости.

Группы композиционны: групповой профиль может включать другие группы. Главное — избегать циклов и чётко документировать состав. На ревью достаточно глянуть в `application.yml`, чтобы понять, что именно «входит» в окружение.

Группы влияют и на загрузку профильных файлов: активировав `prod-eu`, вы фактически активируете `prod` и `eu`, а значит подхватятся `application-prod.yml` и `application-eu.yml`.

**Код (Java): демонстрация — два бина на разные профили, один на группу**

```java
package demo.group;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component @Profile("kafka")
class KafkaClient {}

@Component @Profile("s3")
class S3Client {}

@Component @Profile("prod-eu")
class Marker {}
```

**Код (Kotlin): аналог**

```kotlin
package demo.group

import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

@Component @Profile("kafka")
class KafkaClient

@Component @Profile("s3")
class S3Client

@Component @Profile("prod-eu")
class Marker
```

`application.yml`:

```yaml
spring:
  profiles:
    group:
      prod-eu:
        - prod
        - eu
        - kafka
        - s3
```

Запуск:

```
SPRING_PROFILES_ACTIVE=prod-eu java -jar app.jar
```

---

## Условия на бины: `@Profile`, `@ConditionalOnProperty`/`OnClass`/`OnMissingBean` — когда что выбирать

`@Profile` — простой и понятный способ сказать «этот бин живёт только в таких-то окружениях». Он отлично подходит для фейковых адаптеров (dev/test), специфичных для стека реализаций, включения экспериментальных компонентов.

`@ConditionalOnProperty` — тонкая настройка по флагам: «включить, если `feature.x.enabled=true`». Этот подход лучше, когда вам нужно управлять функцией независимо от профиля, например, включать/выключать ретраи, альтернативные клиенты, дополнительные фильтры.

`@ConditionalOnClass` (и `OnMissingBean`) — инструменты **автоконфигураций**. «Есть класс X в classpath — подключим интеграцию». «Нет пользовательского бина Y — создадим свой по умолчанию». В прикладном коде их реже применяют, но они незаменимы в стартерных библиотеках.

Комбинируйте условия аккуратно: «и профиль, и флаг» — нормально; «или профиль, или класс» — уже трудно читать. Всегда оставляйте дефолтное, безопасное состояние (например, `enabled=false` для тяжёлых интеграций).

**Код (Java): условия — профиль и флаг**

```java
package demo.conds;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("prod")
@ConditionalOnProperty(prefix = "feature.mail", name = "enabled", havingValue = "true")
class ProdMailer {}
```

**Код (Kotlin): аналог**

```kotlin
package demo.conds

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

@Component
@Profile("prod")
@ConditionalOnProperty(prefix = "feature.mail", name = ["enabled"], havingValue = "true")
class ProdMailer
```

`application-prod.yml`:

```yaml
feature:
  mail:
    enabled: true
```

---

### Мини-зависимости для примеров

Gradle (Groovy):

```groovy
plugins {
  id 'org.springframework.boot' version '3.3.4'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'java'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

Gradle (Kotlin):

```kotlin
plugins {
  id("org.springframework.boot") version "3.3.4"
  id("io.spring.dependency-management") version "1.1.6"
  kotlin("jvm") version "1.9.25"
  kotlin("plugin.spring") version "1.9.25"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
dependencies {
  implementation("org.springframework.boot:spring-boot-starter")
  implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

# 4. Конфигурационные свойства: типобезопасное биндинг

## Зачем `@ConfigurationProperties`: группировка настроек, автодокументация, валидация, удобство тестирования

Типобезопасные конфигурационные свойства — это класс, который аккумулирует связанный набор опций под общим префиксом (`app.payment.*`, `client.orders.*`) и предоставляется через DI. Такой подход делает конфигурацию «читаемой»: вместо сотни рассыпанных `@Value("${...}")` у вас одна точка входа с понятными именами полей и Javadoc. Это снижает когнитивную нагрузку и ускоряет ревью: видно, какие настройки реально существуют и кто их использует.

Второй сильный аргумент — **валидация на старте**. Пометив класс `@Validated` и поля аннотациями Jakarta Validation (`@NotBlank`, `@Min`, `@Pattern` и т.д.), мы получаем fail-fast: приложение не запустится, если значение некорректно. В проде это спасает от «тихих» ошибок вроде пустого URL или отрицательного таймаута.

Третий — **автодокументация**. Процессор `spring-boot-configuration-processor` генерирует метаданные, которые IDE понимает как подсказки: автодополнение ключей, типы, дефолты, ссылки на Javadoc. Разработчикам проще добавлять/менять параметры, не копаясь в исходниках.

Четвёртый — **тестируемость**. Такой класс удобно подменять в unit-тестах явным конструктором, без подъёма всего контекста. А в интеграционных тестах его легко накормить `TestPropertyValues` или `@TestPropertySource`, не трогая продовый YAML.

Пятый — **инкапсуляция формата**. Конфиг-класс может держать удобные для кода типы (`Duration`, `URI`, `DataSize`), а YAML остаётся человеко-читабельным (`2s`, `512MB`, `https://…`). Встроенный конвертер Spring выполнит преобразование без ручного парсинга.

И наконец, это дисциплина слоёв. Мы инжектим свойства в инфраструктурные/конфигурационные бины, а доменный код зависит уже от «готовых» клиентов/сервисов. Домен не знает ни про YAML, ни про `Environment`, и это прекрасно.

**Код (Java): типобезопасные свойства с валидацией**

```java
package conf.props;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

import java.net.URI;
import java.time.Duration;

@Validated
@Component // (альтернатива: @ConfigurationPropertiesScan на приложении)
@ConfigurationProperties("client.orders")
public class OrdersClientProps {
  @NotNull private URI baseUrl;
  @Min(100) @Max(10000) private int timeoutMs = 1000;
  @NotNull private Duration retryBackoff = Duration.ofMillis(200);
  private boolean enableMetrics = true;

  public URI getBaseUrl() { return baseUrl; }
  public void setBaseUrl(URI baseUrl) { this.baseUrl = baseUrl; }
  public int getTimeoutMs() { return timeoutMs; }
  public void setTimeoutMs(int timeoutMs) { this.timeoutMs = timeoutMs; }
  public Duration getRetryBackoff() { return retryBackoff; }
  public void setRetryBackoff(Duration retryBackoff) { this.retryBackoff = retryBackoff; }
  public boolean isEnableMetrics() { return enableMetrics; }
  public void setEnableMetrics(boolean enableMetrics) { this.enableMetrics = enableMetrics; }
}
```

**Код (Kotlin): то же, через data-класс**

```kotlin
package conf.props

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotNull
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated
import java.net.URI
import java.time.Duration

@Validated
@Component
@ConfigurationProperties("client.orders")
data class OrdersClientProps(
    @field:NotNull val baseUrl: URI = URI.create("http://localhost:8080"),
    @field:Min(100) @field:Max(10_000) val timeoutMs: Int = 1_000,
    val retryBackoff: Duration = Duration.ofMillis(200),
    val enableMetrics: Boolean = true
)
```

`application.yml`:

```yaml
client:
  orders:
    base-url: https://orders.internal
    timeout-ms: 1500
    retry-backoff: 300ms
    enable-metrics: true
```

Gradle (Groovy):

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-validation'
  annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

Gradle (Kotlin):

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-validation")
  annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```

---

## Биндинг: «расслабленные» имена, вложенные объекты, коллекции и карты

Spring Boot поддерживает **расслабленные** имена: `client.orders.base-url`, `client.orders.baseUrl` и `CLIENT_ORDERS_BASE_URL` будут сопоставлены одному полю `baseUrl`. Это обеспечивает переносимость между YAML, env-переменными и Java-кодом. Команда может выбирать стиль в конфиге (чаще kebab-case), не втягивая его в имена полей.

Вложенные объекты полезны для группировки: «пул» внутри клиента, «TLS» внутри сетевых настроек. Так у вас получается «дерево» с естественной навигацией, без длинных префиксов в одном плоском классе. Внутренние классы удобно валидировать независимо и переиспользовать.

Списки и карты — стандартные граждане биндинга. Список стратегий (`List<String>`), карта «код страны → URL» (`Map<String,URI>`) или «имя брокера → конфиг» — всё это задаётся в YAML естественно. Биндинг сохранит порядок списков и корректно соберёт карту по ключам.

Для карт помните, что ключ — это строка, и в YAML точки в ключах нужно экранировать кавычками. Если у вас `eu-west-1` и `us-east-1` — всё просто; если ключ содержит «.» — заключайте `"a.b"` в кавычки.

Коллекции **мерджатся** поверх базовой конфигурации профилями: вы можете задать общий список, а в `prod` добавить/переопределить элементы (для карт — переопределить по ключу). Это удобно для региональных различий.

И, наконец, биндинг понимает массивы примитивов, перечисления и сложные типы через конвертеры. Главное — держать структуру «узкой» и самодокументируемой: вложенности в 3–4 уровня — ок; глубже — признак переразбиения.

**Код (Java): вложенные объекты, список и карта**

```java
package conf.bind;

import jakarta.validation.constraints.NotBlank;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.time.Duration;
import java.util.List;
import java.util.Map;

@Component
@ConfigurationProperties("client.catalog")
public class CatalogProps {
  private URI baseUrl;
  private Pool pool = new Pool();
  private Tls tls = new Tls();
  private List<String> features = List.of();
  private Map<String, URI> regionUrls = Map.of();

  public URI getBaseUrl() { return baseUrl; }
  public void setBaseUrl(URI baseUrl) { this.baseUrl = baseUrl; }
  public Pool getPool() { return pool; }
  public void setPool(Pool pool) { this.pool = pool; }
  public Tls getTls() { return tls; }
  public void setTls(Tls tls) { this.tls = tls; }
  public List<String> getFeatures() { return features; }
  public void setFeatures(List<String> features) { this.features = features; }
  public Map<String, URI> getRegionUrls() { return regionUrls; }
  public void setRegionUrls(Map<String, URI> regionUrls) { this.regionUrls = regionUrls; }

  public static class Pool {
    private int maxSize = 10;
    private Duration idle = Duration.ofSeconds(30);
    public int getMaxSize() { return maxSize; }
    public void setMaxSize(int maxSize) { this.maxSize = maxSize; }
    public Duration getIdle() { return idle; }
    public void setIdle(Duration idle) { this.idle = idle; }
  }
  public static class Tls {
    @NotBlank private String truststorePath = "";
    public String getTruststorePath() { return truststorePath; }
    public void setTruststorePath(String truststorePath) { this.truststorePath = truststorePath; }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.bind

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import java.net.URI
import java.time.Duration

@Component
@ConfigurationProperties("client.catalog")
data class CatalogProps(
    val baseUrl: URI = URI.create("http://localhost:8081"),
    val pool: Pool = Pool(),
    val tls: Tls = Tls(),
    val features: List<String> = emptyList(),
    val regionUrls: Map<String, URI> = emptyMap()
) {
    data class Pool(val maxSize: Int = 10, val idle: Duration = Duration.ofSeconds(30))
    data class Tls(val truststorePath: String = "")
}
```

`application.yml`:

```yaml
client:
  catalog:
    base-url: https://catalog.internal
    pool:
      max-size: 32
      idle: 45s
    tls:
      truststore-path: /etc/certs/ts.jks
    features: [recommendations, trending]
    region-urls:
      eu-west-1: https://catalog.eu
      "us.east": https://catalog.use
```

---

## Иммутабельность: конструкторный биндинг по умолчанию в Boot 3.x; почему лучше, чем сеттеры

В Boot 3.x конструкторный биндинг — **дефолт** для классов свойств (особенно очевиден в Kotlin и Java records). Это делает объекты неизменяемыми: значения устанавливаются один раз при старте, после чего их нельзя мутировать из кода. Плюсы — потокобезопасность, упрощённый reasoning и отсутствие «магии» поздних изменений.

Сеттеры допускают «подмену» значений во время работы (например, ленивые компоненты неожиданно перестраивают конфиг), что ломает инварианты. При immutable-подходе таких сюрпризов нет: если что-то нужно изменить — это новая конфигурация и новый деплой/рестарт. Для микросервисов это норма.

Ещё один плюс конструктора — **обязательность** полей. Если аргумент конструктора не передан, биндинг не состоится, и вы получите понятную ошибку. В setter-модели легко «забыть» заполнить поле: NPE всплывёт позже.

Где иммутабельность неудобна? В редких случаях «горячего» обновления конфигов (Config Server + /refresh). Но и там лучше держать mutable только одну «прослойку», а до домена доносить уже «снимок» через новую зависимость.

Если нужен дефолт на уровне параметра — используйте `@DefaultValue` (Boot) или дефолт аргумента в Kotlin. Это лучше, чем «магические» значения в коде или `Optional` с пост-обработкой.

Наконец, immutable-классы свойств хорошо работают с AOT/Native: меньше рефлексии, проще подсказки, предсказуемые графы.

**Код (Java): record как класс свойств (конструкторный биндинг)**

```java
package conf.immut;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.time.Duration;

@Component
@ConfigurationProperties("client.payments")
public record PaymentsProps(URI baseUrl, Duration timeout, boolean retriesEnabled) { }
```

**Код (Kotlin): data-класс — естественно immutable**

```kotlin
package conf.immut

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import java.net.URI
import java.time.Duration

@Component
@ConfigurationProperties("client.payments")
data class PaymentsProps(
    val baseUrl: URI,
    val timeout: Duration = Duration.ofSeconds(2),
    val retriesEnabled: Boolean = true
)
```

`application.yml`:

```yaml
client:
  payments:
    base-url: https://pay.internal
    timeout: 2s
    retries-enabled: true
```

---

## Валидация: аннотации Jakarta Validation, отказ от старта при некорректных значениях

Валидация свойств — главный механизм fail-fast. Любой класс свойств можно пометить `@Validated` и использовать аннотации на полях/параметрах конструктора: `@NotBlank`, `@Min/@Max`, `@Positive`, `@Pattern`, `@Email` и т.д. Если правило нарушено, Spring выбросит исключение на старте с понятным сообщением «Property … invalid».

С `Duration` и `DataSize` удобно проверять границы через числовые аналоги или дополнительный валидатор-метод с `@AssertTrue`. Например, «таймаут не меньше 100 мс» или «размер буфера не превышает 256 MB». Это лучше, чем хранить всё в long-миллис и терять читабельность YAML.

Валидация особенно важна для небезопасных параметров: лимитов, порогов, URL внешних систем. Опечатка в хосте или ноль в таймауте могут превратиться в час отладки под нагрузкой. Пусть приложение падает сразу — так дешевле.

Связывайте валидацию с документацией: Javadoc на поле/параметре плюс аннотации дают IDE достаточную информацию, чтобы подсветить неправильное значение прямо в YAML (`.json` метаданные помогают, см. ниже).

Если нужно своё правило, используйте кастомную аннотацию и `ConstraintValidator`. Так вы сумеете выразить доменные ограничения (например, «имя проекта — `^[a-z][a-z0-9-]{2,20}$`»), и это будет единообразно применяться везде.

И не забывайте тесты: негативные кейсы на валидацию должны быть частью регрессии; `ApplicationContextRunner` позволяет быстро проверить падение старта при плохих значениях.

**Код (Java): валидация с `@AssertTrue` для `Duration`**

```java
package conf.validate;

import jakarta.validation.constraints.*;
import jakarta.validation.constraints.AssertTrue;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;

@Validated
@Component
@ConfigurationProperties("limits.rate")
public class RateLimitProps {
  @Positive private int maxRequests = 100;
  @NotNull  private Duration per = Duration.ofSeconds(1);

  @AssertTrue(message = "per must be >= 100ms")
  public boolean isPerValid() { return per.compareTo(Duration.ofMillis(100)) >= 0; }

  public int getMaxRequests() { return maxRequests; }
  public void setMaxRequests(int maxRequests) { this.maxRequests = maxRequests; }
  public Duration getPer() { return per; }
  public void setPer(Duration per) { this.per = per; }
}
```

**Код (Kotlin): аналогичная проверка**

```kotlin
package conf.validate

import jakarta.validation.constraints.NotNull
import jakarta.validation.constraints.Positive
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated
import java.time.Duration

@Validated
@Component
@ConfigurationProperties("limits.rate")
data class RateLimitProps(
    @field:Positive val maxRequests: Int = 100,
    @field:NotNull val per: Duration = Duration.ofSeconds(1)
) {
    fun isPerValid() = per >= Duration.ofMillis(100)
}
```

`application.yml`:

```yaml
limits:
  rate:
    max-requests: 200
    per: 500ms
```

---

# 5. `@ConfigurationProperties` vs `@Value`

## Когда использовать `@ConfigurationProperties`: набор связанных параметров, повторное использование, валидация

Если у вас **несколько** параметров с общим смыслом (HTTP-клиент, пул БД, политика ретраев), выбирайте `@ConfigurationProperties`. Это даёт группировку, валидацию и автодокументацию. Один префикс = одна зона ответственности; так конфиги легко искать и обсуждать.

Такой класс переиспользуем между несколькими бинами: один и тот же набор опций может конфигурировать и «низкоуровневый» клиент, и его декоратор (метрики/трассировка). Меняя YAML, вы меняете поведение целого подграфа, а не правите код в каждом месте.

Ещё бонус — простые тесты. Инициализируйте объект конструктором и передайте в фабрику/сервис: никакой среды и контекста. Это особенно удобно для модульных библиотек и стартеров.

Наконец, это уменьшает «шум» в DI. Вместо пяти аргументов-примитивов вы передаёте один объект настроек. Конструктор сервиса чище, сигнатуры стабильнее, а ревью фокусируется на логике, а не на wiring’е.

Именно поэтому «правило команды» часто звучит так: *всё, что больше одного-двух флагов, — в `@ConfigurationProperties`*.

**Код (Java): использование `@ConfigurationProperties` в фабрике клиента**

```java
package conf.vsvalue;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class HttpClientConfig {
  @Bean OrdersClient ordersClient(conf.props.OrdersClientProps props) {
    return new OrdersClient(props.getBaseUrl(), props.getTimeoutMs(), props.isEnableMetrics());
  }
}

class OrdersClient {
  OrdersClient(java.net.URI baseUrl, int timeoutMs, boolean metrics) { /* ... */ }
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.vsvalue

import conf.props.OrdersClientProps
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class HttpClientConfig {
    @Bean
    fun ordersClient(props: OrdersClientProps) =
        OrdersClient(props.baseUrl, props.timeoutMs, props.enableMetrics)
}

class OrdersClient(val baseUrl: java.net.URI, val timeoutMs: Int, val metrics: Boolean)
```

---

## Когда достаточно `@Value`: единичные флаги/константы; почему плохо «размазывать» `@Value` по сервисам

`@Value` уместен для одиночных, **редко меняющихся** флагов: сервисное имя, величина batch-пакета, переключатель фичи, который не тянет за собой целый класс свойств. Это кратко и не требует отдельного типа. Но как только параметров становится 2–3, `@Value` начинает размножаться по коду, ломая читаемость и валидацию.

Главный минус «россыпи `@Value`» — невозможность централизованно валидировать и документировать параметры. Любая опечатка в ключе не ловится компиляцией, а ошибки могут проявиться «внезапно». С `@ConfigurationProperties` IDE подскажет ключи и типы.

Если `@Value` всё-таки используется, держите его в **инфраструктурных** классах (конфиг-фабрики, адаптеры), а домен пусть получает уже настроенные сервисы. Так доменная логика не «знает» о YAML.

И не используйте SpEL-логику внутри `@Value` (кроме простых плейсхолдеров) — вычисления в аннотациях усложняют отладку и мешают AOT.

**Код (Java): уместный `@Value` для одного флага**

```java
package conf.value;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
class FeatureFlags {
  private final boolean newEngine;
  FeatureFlags(@Value("${feature.new-engine:false}") boolean newEngine) { this.newEngine = newEngine; }
  boolean enabled() { return newEngine; }
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.value

import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component

@Component
class FeatureFlags(@Value("\${feature.new-engine:false}") private val newEngine: Boolean) {
    fun enabled() = newEngine
}
```

`application.yml`:

```yaml
feature:
  new-engine: true
```

---

## Плейсхолдеры `${...}` и дефолты (`${X:Y}`), ограничения SpEL; избегаем логики в аннотациях

Плейсхолдер `${key}` подставляет значение свойства в другую строку, `${key:default}` — даёт дефолт, если ключ не найден. Это удобно, чтобы **собрать** сложные URL/строки из атомарных значений (`host`, `port`, `scheme`) и избежать дублей.

SpEL в `@Value` поддерживается, но применяйте его крайне экономно. Любая логика (конкатенация, условия) в аннотации ухудшает читабельность, ломает AOT и маскирует ошибки. Лучше сформировать сложное значение в конфигурационном бине или прямо в классе свойств.

Для секретов используйте плейсхолдеры на части строки, а не храните «склеенные» URI с паролями в одном месте: так легче маскировать и менять. И помните, что значения могут попадать в логи — отключайте «echo» секретов в своих утилитах.

Если всё-таки нужен дефолт в `@Value`, ставьте его как «разумный» и документируйте: комментарий или Javadoc рядом критично важны.

**Код (Java): сборка строки из плейсхолдеров**

```java
// application.yml:
/*
db:
  host: db.prod
  port: 5432
  url: jdbc:postgresql://${db.host}:${db.port}/app
*/

package conf.place;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
class DbUrlHolder {
  private final String url;
  DbUrlHolder(@Value("${db.url}") String url) { this.url = url; }
  public String url() { return url; }
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.place

import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component

@Component
class DbUrlHolder(@Value("\${db.url}") private val url: String) {
    fun url() = url
}
```

---

## Инъекция свойств только в конфигурационные/инфраструктурные бины, а не в доменную логику

Доменные сервисы должны быть **чисты** и детерминированы. Инъекция свойств — это инфраструктурная забота: как настроить клиент, какие таймауты/пути использовать. Если «протащить» `Environment`/`@Value` в домен, тесты станут тяжелее, а код — хрупче к окружению.

Лучше инжектить домену готовые **политики** или **параметризованные** интерфейсы. Например, домен зависит от `PricingPolicy`, а конфиг собирает нужную реализацию в зависимости от свойств. Так произведение остаётся чистым, а конфиг — адаптируемым.

Если всё-таки домену нужен параметр (например, скидка), передайте его **значением через конструктор** из конфиг-слоя. Это по-прежнему DI, но домен ничего не знает о YAML и может быть создан в юнит-тесте простым `new`.

Разделяйте ответственность по пакетам: `*.config`/`*.infrastructure` для классов, читающих свойства, и `*.domain` для логики. Такой порядок облегчает навигацию и ревью.

И, конечно, линтуйте себя: запретить использование `Environment` в `*.domain` через ArchUnit — дешёвый способ не допускать утечек инфраструктуры.

**Код (Java): домен чистый, конфиг инжектит значение**

```java
package domain.clean;

public class DiscountService {
  private final int percent; // обычное значение, пришло "сверху"
  public DiscountService(int percent) { this.percent = percent; }
  public long apply(long price) { return price - (price * percent / 100); }
}
```

```java
package domain.clean;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class DiscountConfig {
  @Bean DiscountService discountService(@Value("${pricing.discount-percent:5}") int p) {
    return new DiscountService(p);
  }
}
```

**Код (Kotlin): то же**

```kotlin
package domain.clean

class DiscountService(private val percent: Int) {
    fun apply(price: Long): Long = price - (price * percent / 100)
}

package domain.clean

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class DiscountConfig {
    @Bean
    fun discountService(@Value("\${pricing.discount-percent:5}") p: Int) = DiscountService(p)
}
```

---

# 6. Типы и форматы свойств

## Встроенные конвертеры: `Duration` (`10s`, `2m`), `DataSize` (`256MB`), `InetAddress`, `URI`, `Enum`

Boot «из коробки» конвертирует строковые значения в богатые типы: `Duration` (поддерживает `ns,ms,s,m,h,d`), `DataSize` (`KB,MB,GB`), `InetAddress`, `URI`, `URL`, `Locale`, `Charset`, `Path`, а также enum-ы. Это позволяет писать выразительный YAML и избегать ручного парсинга/ошибок единиц.

Для `Duration` удобно хранить времена ожидания/TTL; для `DataSize` — лимиты буферов/вложений; для `URI` — адреса внешних сервисов. Enum-ы избавляют от опечаток и документируют допустимые значения в IDE.

Если надо переопределить единицу по умолчанию (например, `timeout: 200` трактовать как миллисекунды), используйте аннотации-подсказки на уровне свойств (в Boot есть `@DurationUnit`). Это снимает двусмысленность и работает одинаково в Java и Kotlin.

При необходимости добавляйте собственные конвертеры (через `Converter<String, T>` и регистратора); но чаще хватает встроенных.

Всегда предпочитайте богатые типы примитивам: `Duration` читабельнее и безопаснее, чем `long timeoutMs`, особенно в миксах профилей.

**Код (Java): все типы в одном классе свойств**

```java
package conf.types;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.net.InetAddress;
import java.net.URI;
import java.time.Duration;
import org.springframework.util.unit.DataSize;

@Component
@ConfigurationProperties("upload")
public class UploadProps {
  private DataSize maxFile = DataSize.ofMegabytes(32);
  private Duration timeout = Duration.ofSeconds(10);
  private URI storageUri;
  private InetAddress bind;

  public DataSize getMaxFile() { return maxFile; }
  public void setMaxFile(DataSize maxFile) { this.maxFile = maxFile; }
  public Duration getTimeout() { return timeout; }
  public void setTimeout(Duration timeout) { this.timeout = timeout; }
  public URI getStorageUri() { return storageUri; }
  public void setStorageUri(URI storageUri) { this.storageUri = storageUri; }
  public InetAddress getBind() { return bind; }
  public void setBind(InetAddress bind) { this.bind = bind; }
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.types

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.util.unit.DataSize
import java.net.InetAddress
import java.net.URI
import java.time.Duration

@Component
@ConfigurationProperties("upload")
data class UploadProps(
    val maxFile: DataSize = DataSize.ofMegabytes(32),
    val timeout: Duration = Duration.ofSeconds(10),
    val storageUri: URI = URI.create("s3://bucket/path"),
    val bind: InetAddress? = null
)
```

`application.yml`:

```yaml
upload:
  max-file: 64MB
  timeout: 15s
  storage-uri: s3://media/cdn
  bind: 0.0.0.0
```

---

## Карты/списки: как задавать в YAML, ключи с точками, «мердж» значений между профилями

Списки в YAML оформляются «пулей» (`-`) или `[]`. Порядок сохраняется — это важно для пайплайнов. Карты — привычное «ключ: значение». Ключи с точками нужно брать в кавычки, иначе YAML воспримет их как вложенность.

Профили **мерджат** коллекции: базовый список может быть дополнен в `prod`, а карта — частично переопределена по ключам. Это позволяет держать общий baseline и варьировать детали, не копируя всё целиком.

Для сложных списков (например, список объектов) просто описывайте вложенный класс; биндинг прекрасно справится. Старайтесь держать имена полей короткими и понятными — в YAML это экономит место.

Проверяйте, что коллекции не пустые там, где это критично (валидация `@NotEmpty`). Это типичная ошибка: пустой список брокеров/реплик заметят только под нагрузкой.

Если требуется фиксированный набор ключей, используйте enum-ключ — биндинг поддерживает карты с `Enum` как ключом (в YAML ключи — имена констант).

**Код (Java): списки и карты с вложенными объектами**

```java
package conf.collections;

import jakarta.validation.constraints.NotEmpty;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.net.URI;
import java.util.List;
import java.util.Map;

@Component
@ConfigurationProperties("routes")
public class RoutesProps {
  @NotEmpty private List<Rule> rules = List.of();
  private Map<String, URI> overrides = Map.of();

  public List<Rule> getRules() { return rules; }
  public void setRules(List<Rule> rules) { this.rules = rules; }
  public Map<String, URI> getOverrides() { return overrides; }
  public void setOverrides(Map<String, URI> overrides) { this.overrides = overrides; }

  public static class Rule {
    private String path;
    private String service;
    public String getPath() { return path; }
    public void setPath(String path) { this.path = path; }
    public String getService() { return service; }
    public void setService(String service) { this.service = service; }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.collections

import jakarta.validation.constraints.NotEmpty
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import java.net.URI

@Component
@ConfigurationProperties("routes")
data class RoutesProps(
    @field:NotEmpty val rules: List<Rule> = emptyList(),
    val overrides: Map<String, URI> = emptyMap()
) {
    data class Rule(val path: String, val service: String)
}
```

`application.yml` и профиль:

```yaml
routes:
  rules:
    - path: /api/users/**
      service: users
    - path: /api/pay/**
      service: payments
  overrides:
    "users.v2": https://users-v2.internal

---
spring:
  config:
    activate:
      on-profile: prod
routes:
  rules:
    - path: /api/analytics/**
      service: analytics
  overrides:
    payments: https://payments-eu.internal
```

---

## Секреты и многократное использование: аккуратное составление строк (например, URL из хоста+порта) через плейсхолдеры

Секреты (пароли, токены) лучше **не зашивать** в единый URL. Разносите на атомарные поля (`username`, `password`, `host`, `port`) и через плейсхолдеры собирайте «рабочие» строки в безопасных местах. Так легче подключить внешние секрет-хранилища и маскировать значения в логах.

В Docker/K8s удобно маппить секреты в env-переменные и/или в `configtree:` (файлы). Тогда YAML остаётся «чистым», а чувствительные куски приезжают из внешней системы. В коде — только типобезопасный доступ.

Для многократного использования выбирайте один «источник истины» и плейсхолдерами подставляйте туда-сюда. Это минимизирует дрейф значений при правках.

Шаблоны URL с параметрами (`jdbc:postgresql://${db.host}:${db.port}/app`) — хороший компромисс: пароль отдельно, адрес отдельно, сборка прозрачна.

И не печатайте секреты в логи. Чаще всего их случайно вываливает «диагностический принтер» конфигов — проследите, чтобы он маскировал известные ключи (`password`, `secret`, `token`), или вообще не логируйте значения по умолчанию.

**Код (Java): свойства для сборки JDBC без секретов в одном поле**

```java
package conf.secrets;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties("db")
public class DbProps {
  private String host; private int port; private String name;
  private String username; private String password;

  public String jdbcUrl() { return "jdbc:postgresql://" + host + ":" + port + "/" + name; }

  // getters/setters
  public String getHost(){return host;} public void setHost(String h){this.host=h;}
  public int getPort(){return port;} public void setPort(int p){this.port=p;}
  public String getName(){return name;} public void setName(String n){this.name=n;}
  public String getUsername(){return username;} public void setUsername(String u){this.username=u;}
  public String getPassword(){return password;} public void setPassword(String p){this.password=p;}
}
```

**Код (Kotlin): аналог**

```kotlin
package conf.secrets

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties("db")
data class DbProps(
    val host: String = "localhost",
    val port: Int = 5432,
    val name: String = "app",
    val username: String = "app",
    val password: String = "secret"
) {
    fun jdbcUrl() = "jdbc:postgresql://$host:$port/$name"
}
```

`application.yml`:

```yaml
db:
  host: ${DB_HOST:db}
  port: ${DB_PORT:5432}
  name: app
  username: ${DB_USER}
  password: ${DB_PASS}
```

---

## Локализация и многоязычность: конфиги сообщений отдельно от технических свойств

Технические свойства (`client.*`, `server.*`) не должны смешиваться с локализуемыми сообщениями пользователя. Для последних используйте `MessageSource` и bundles (`messages_xx.properties`). Это разделяет ответственность: инфраструктура — в YAML, тексты — в ресурсах.

Вы можете задавать базовое имя bundles через `spring.messages.basename` — это облегчает поддержку нескольких наборов сообщений (например, для разных доменов UI). Для форматирования параметров в сообщениях используйте `MessageFormat` и плейсхолдеры `{0}`, `{1}`.

Если локализуются шаблоны email/SMS — держите их рядом, но всё же отдельно от YAML; так проще обновлять тексты без риска «сломать» техническую конфигурацию.

Для сервисов без UI это тоже актуально: сообщения ошибок API (user-facing) и технические логи — разные сущности. Держите ключи сообщений в коде, а переводы — в ресурсах.

И, как всегда, тестируйте бандлы: отсутствие ключа должно давать понятное поведение (дефолт или явную ошибку), а не NPE в глубинах форматтера.

**Код (Java): MessageSource + свойство basename**

```java
// application.yml
/*
spring:
  messages:
    basename: messages,errors
*/

package i18n.demo;

import org.springframework.context.MessageSource;
import org.springframework.stereotype.Service;

import java.util.Locale;

@Service
public class I18nService {
  private final MessageSource ms;
  public I18nService(MessageSource ms){ this.ms = ms; }
  public String greet(String name, Locale locale) {
    return ms.getMessage("greet", new Object[]{name}, locale);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package i18n.demo

import org.springframework.context.MessageSource
import org.springframework.stereotype.Service
import java.util.Locale

@Service
class I18nService(private val ms: MessageSource) {
    fun greet(name: String, locale: Locale) = ms.getMessage("greet", arrayOf(name), locale)
}
```

`src/main/resources/messages.properties`:

```
greet=Hello, {0}!
```

`src/main/resources/messages_ru.properties`:

```
greet=Привет, {0}!
```

---

# 7. Метаданные конфигурации и удобство разработки

## Генерация метаданных (`spring-boot-configuration-processor`) для подсказок IDE и документации

Процессор конфигурации генерирует файл `META-INF/spring-configuration-metadata.json`, где описаны все ваши `@ConfigurationProperties`: префиксы, поля, типы, описания и дефолты. IDE (IntelliJ IDEA) использует его, чтобы подсказывать ключи и подсвечивать ошибки прямо в YAML.

Это не просто «приятно». Метаданные становятся **договором** между командами: какой ключ за что отвечает, какие значения допустимы. Документируйте поля Javadoc’ом — комментарии попадут в метаданные и в подсказки IDE.

Процессор нужно подключить как `annotationProcessor` (Java) или `kapt/ksp` (Kotlin). Он работает на компиляции и не увеличивает runtime-зависимости. Для Kotlin-проектов с `kotlin-spring` достаточно `annotationProcessor`; при использовании KAPT — добавьте `kapt` плагин.

Проверяйте артефакт: `jar tf build/libs/app.jar | grep spring-configuration-metadata.json`. Если файла нет — IDE не увидит подсказки (особенно в multi-module).

В CI полезно иметь задачу, валидирующую, что ключи не «сломались» (например, snapshot-тест метаданных для критичных модулей).

**Gradle (Groovy/Kotlin): подключение процессора**

```groovy
plugins { id 'org.springframework.boot' version '3.3.4'; id 'io.spring.dependency-management' version '1.1.6'; id 'java' }
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter'
  annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
}
```

```kotlin
plugins { id("org.springframework.boot") version "3.3.4"; id("io.spring.dependency-management") version "1.1.6"; kotlin("jvm") version "1.9.25"; kotlin("plugin.spring") version "1.9.25" }
dependencies {
  implementation("org.springframework.boot:spring-boot-starter")
  annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
}
```

**Код (Java): Javadoc попадёт в метаданные**

```java
package meta.demo;

import org.springframework.boot.context.properties.ConfigurationProperties;

/** Настройки upload-сервиса. */
@ConfigurationProperties("upload")
public record UploadMetaProps(
    /** Максимальный размер файла, поддерживает единицы (KB/MB/GB). */
    org.springframework.util.unit.DataSize maxFile,
    /** Таймаут обработки запроса. */
    java.time.Duration timeout
) {}
```

**Код (Kotlin): аналог**

```kotlin
package meta.demo

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.util.unit.DataSize
import java.time.Duration

/** Настройки upload-сервиса. */
@ConfigurationProperties("upload")
data class UploadMetaProps(
    /** Максимальный размер файла, поддерживает единицы (KB/MB/GB). */
    val maxFile: DataSize = DataSize.ofMegabytes(64),
    /** Таймаут обработки запроса. */
    val timeout: Duration = Duration.ofSeconds(10)
)
```

---

## Подключение процессора: `annotationProcessor` (Java) или `kapt`/`ksp` (Kotlin), размещение классов свойств в отдельном модуле «contract»

В больших системах полезно держать классы свойств в отдельном модуле «contract» (без зависимостей на конкретные клиенты). Тогда сервисы и стартеры зависят от одного и того же набора типов, а метаданные генерируются централизованно.

Для Java достаточно `annotationProcessor`. Для Kotlin возможны варианты: `annotationProcessor` тоже работает, но если у вас KAPT-pipeline, добавьте `kapt("org.springframework.boot:spring-boot-configuration-processor")`. С KSP можно использовать community-адаптеры, но стандартный путь — kapt/annotationProcessor.

Следите за публикацией `-metadata.json` внутри JAR модуля-контракта. Это позволит и **другим** проектам видеть подсказки по вашим ключам, просто подключив зависимость (стартера).

В multi-module важно правильно настроить зависимость «compileOnly» для процессора в каждом модуле, где есть `@ConfigurationProperties`. Иначе метаданные будут не полными.

Поддерживайте версионирование контрактов: добавляйте новые ключи **с дефолтами**, помечайте deprecated-поля в Javadoc, чтобы подсказки IDE помогали мигрировать.

**Gradle (Kotlin + KAPT) пример корневого и контракта**

```kotlin
plugins { kotlin("jvm") version "1.9.25"; id("org.springframework.boot") version "3.3.4"; kotlin("kapt") }

subprojects {
  dependencies { annotationProcessor("org.springframework.boot:spring-boot-configuration-processor") }
}

project(":config-contract") {
  plugins.apply("java-library")
  dependencies {
    api("org.springframework.boot:spring-boot")
    annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
    kapt("org.springframework.boot:spring-boot-configuration-processor")
  }
}
```

---

## Описание `@ConstructorBinding`/`@DefaultValue` и Javadoc — как источник помощи в IDE

В Boot 3.x указание `@ConstructorBinding` на классе свойств опционально: конструкторный биндинг используется по умолчанию, особенно очевидно в record/data-классах. Тем не менее аннотация может подсказать намерение и в смешанных случаях (когда есть и сеттеры).

`@DefaultValue` (из пакета биндинга Spring Boot) позволяет задать дефолт непосредственно на параметре конструктора (Java), что делает контракт явным и не требует перегруженных конструкторов. В Kotlin более естественно использовать дефолты параметров.

Javadoc — ваш друг: описания на уровне полей/параметров попадают в метаданные и видны как подсказки в YAML. Это особенно важно для числовых значений: «миллисекунды или секунды?», «какой допустимый диапазон?».

Команда выигрывает от таких «самодокументирующихся» конфигов: меньше вопросов в чатах, меньше ошибок «я думал, что это секунды». Сделайте правило: каждый экспортируемый класс свойств имеет комментарии и разумные дефолты.

Не бойтесь помечать старые ключи `@Deprecated` в Javadoc и оставлять заметки «use xyz instead». IDE подсветит это в YAML, и миграции пройдут мягче.

**Код (Java): `@DefaultValue` на параметрах конструктора**

```java
package meta.defaultv;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;

/** Настройки analytics-клиента. */
@ConfigurationProperties("client.analytics")
public record AnalyticsProps(
    /** Базовый URL сервиса. */
    String baseUrl,
    /** Таймаут запроса (мс). */
    @DefaultValue("1000") int timeoutMs,
    /** Включить метрики клиента. */
    @DefaultValue("true") boolean metrics
) { }
```

**Код (Kotlin): дефолты параметров как контракт**

```kotlin
package meta.defaultv

import org.springframework.boot.context.properties.ConfigurationProperties

/** Настройки analytics-клиента. */
@ConfigurationProperties("client.analytics")
data class AnalyticsProps(
    /** Базовый URL сервиса. */
    val baseUrl: String,
    /** Таймаут запроса (мс). */
    val timeoutMs: Int = 1_000,
    /** Включить метрики клиента. */
    val metrics: Boolean = true
)
```

---

## Названия иерархий свойств: единый префикс на сервис/модуль, предсказуемые имена

Хорошее имя — половина конфигурации. Договоритесь о префиксах на уровне модулей: `client.orders.*`, `client.payments.*`, `db.*`, `security.*`, `feature.*`. Избегайте абстрактных `app.*` для всего подряд: быстро превращается в свалку.

Используйте **kebab-case** для ключей в YAML: он привычен глазу и хорошо читается. В коде поля по конвенции — `camelCase`. Расслабленный биндинг соединит оба мира.

Не смешивайте «поведение» и «окружение» в одном префиксе. Например, `feature.*` — включатели бизнес-функций, `env.*` — инфраструктурные особенности окружения (регион, зона, tenant). Так понятнее, что можно менять часто, а чему нужен change-management.

Старайтесь, чтобы ключ **говорил сам за себя**: `timeout-ms`, `max-file`, `base-url`, `retries-enabled`. Избегайте «коротких, но туманных» вроде `limit` без контекста — через месяц никто не вспомнит, какой именно это лимит.

И, конечно, не плодите синонимов. Один ключ — одно значение. Если API эволюционирует, оставьте старый ключ как deprecated на пару релизов с явным предупреждением.

**Код (Java): пример «словаря» префикса модуля**

```java
package naming.demo;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("security")
public record SecurityProps(
    boolean enableCsrf,
    boolean enableHsts,
    String corsAllowedOrigins
) {}
```

**Код (Kotlin): аналог**

```kotlin
package naming.demo

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties("security")
data class SecurityProps(
    val enableCsrf: Boolean = true,
    val enableHsts: Boolean = true,
    val corsAllowedOrigins: String = "*"
)
```

`application.yml`:

```yaml
security:
  enable-csrf: true
  enable-hsts: true
  cors-allowed-origins: https://app.example.com
```

# 8. Секреты и внешние конфиги: Docker/K8s/облако

## Переменные окружения ↔ имена свойств: правила маппинга (верхний регистр, `_` вместо `.`)

В контейнерах и CI/CD главный носитель настроек — переменные окружения. Spring Boot применяет «расслабленный» биндинг: ключи свойств можно писать разными способами, а движок сведёт их к одному каноническому имени. Это означает, что `client.mail.base-url` из YAML эквивалентен `CLIENT_MAIL_BASE_URL` в окружении и `client.mail.baseUrl` в Java-коде. Такое соответствие снимает боль преобразования между стилями и форматами.

Правило маппинга простое: точки и дефисы превращаются в подчёркивания, а буквы — в верхний регистр. Для индексов списков используется суффикс `_0`, `_1` и т.п. Например, `app.allowed-origins[0]` ↔ `APP_ALLOWED_ORIGINS_0`. Для карт ключ включается в имя: `client.region-urls.eu-west-1` ↔ `CLIENT_REGION_URLS_EU_WEST_1`. Если ключ карты содержит точку, в YAML его берут в кавычки, а в окружении — заменяют точку на подчёркивание.

Приоритет у переменных окружения высокий: значение из env перекрывает то же свойство из `application.yml`. Это позволяет иметь «базовый» конфиг в артефакте и накладывать оверрайды на окружении без пересборки. Такой подход соответствует 12-фактору и особенно удобен в Docker/Kubernetes.

Есть нюанс: некоторые оболочки интерпретируют символы (`$`, `:`), поэтому для сложных строк (URL с паролями) лучше дробить секреты на атомарные части: `DB_HOST`, `DB_USER`, `DB_PASS`, а URI собирать плейсхолдерами в YAML. Выигрывают и безопасность, и читаемость.

В docker-compose переменные можно объявлять в `environment:` или через `.env` — оба способа корректно подхватываются. В Kubernetes — это `env:` в манифесте Pod/Deployment. Там же удобно ссылаться на `Secret`/`ConfigMap` через `valueFrom`, не прописывая значения явно.

В коде не надо «читать» окружение вручную — используйте `@ConfigurationProperties` или `Environment` (только в инфраструктуре). Тогда ваш сервис остаётся чистым, а конфигурация — внешней.

**Код (Java): маппинг ENV → свойства**

```java
package ext.env;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.net.URI;

@Component
@ConfigurationProperties("client.mail")
public class MailProps {
  private URI baseUrl;
  private int timeoutMs = 1000;

  public URI getBaseUrl() { return baseUrl; }
  public void setBaseUrl(URI baseUrl) { this.baseUrl = baseUrl; }
  public int getTimeoutMs() { return timeoutMs; }
  public void setTimeoutMs(int timeoutMs) { this.timeoutMs = timeoutMs; }
}
```

```kotlin
package ext.env

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import java.net.URI

@Component
@ConfigurationProperties("client.mail")
data class MailProps(
    val baseUrl: URI = URI.create("https://mail.local"),
    val timeoutMs: Int = 1000
)
```

`docker-compose.yml` (фрагмент):

```yaml
services:
  app:
    image: my/app:latest
    environment:
      CLIENT_MAIL_BASE_URL: https://mail.prod
      CLIENT_MAIL_TIMEOUT_MS: "1500"
```

`Deployment.yaml` (фрагмент):

```yaml
env:
  - name: CLIENT_MAIL_BASE_URL
    value: https://mail.prod
  - name: CLIENT_MAIL_TIMEOUT_MS
    value: "1500"
```

---

## Docker/K8s secrets/configmaps: монтирование как `configtree:`/файлы, отключение логирования значений

В Kubernetes и Docker часть конфигурации удобно хранить в `ConfigMap` (некритичные параметры) и `Secret` (секреты). Оба ресурса можно монтировать в контейнер как файлы. Spring Boot поддерживает формат «дерева конфигов» — `configtree:`: каждая пара «файл → значение» превращается в свойство, где имя свойства — имя файла. Это избавляет от необходимости перечислять десятки `env`-переменных.

Монтирование secrets как файлов хорош тем, что права на файлы управляются ОС/контейнером, а приложение считывает их стандартным способом. Кроме того, `kubectl` не показывает содержимое файлов при выводе env, что снижает риск случайной утечки в логи CI. Для включения достаточно добавить `spring.config.import=configtree:/etc/secrets/` и смонтировать Secret на этот путь.

Управляйте видимостью значений в Actuator. В Boot 3.x есть настройки `management.endpoint.env.show-values` и `management.endpoint.configprops.show-values` — выставите `NEVER` или `WHEN_AUTHORIZED`, чтобы даже на dev-стендах случайно не вываливать секреты. Дополнительно можно расширить список маскируемых ключей через `management.endpoint.env.keys-to-sanitize`.

Не печатайте свойства «как есть» в логах старта. Если нужен диагностический отчёт, маскируйте известные ключи (`password`, `secret`, `token`). Actuator делает это автоматически, но самописные принтеры должны следовать тем же правилам.

Для ConfigMap (несекретные параметры) можно использовать и `configtree:`, и обычный `spring.config.import=file:/etc/config/application.yml`. Первый способ удобнее при «мельчайшей» гранулярности (много независимых ключей), второй — когда хочется хранить YAML как единый файл, но вне образа.

Секреты и конфиги подлежат ротации. В Kubernetes пересоздайте `Secret/ConfigMap` и перезапустите Pod (или используйте операторы, которые триггерят rolling update). Приложение читает свойства при старте; если нужен hot-reload, рассматривайте специальные решения (Spring Cloud Kubernetes/Config), но осознанно.

**Kubernetes Secret → `configtree:`**

```yaml
apiVersion: v1
kind: Secret
metadata: { name: app-secrets }
type: Opaque
stringData:
  db.username: app
  db.password: s3cr3t
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      volumes:
        - name: secrets
          secret: { secretName: app-secrets }
      containers:
        - name: app
          image: my/app:latest
          volumeMounts:
            - { name: secrets, mountPath: /etc/secrets, readOnly: true }
          env:
            - name: SPRING_CONFIG_IMPORT
              value: "configtree:/etc/secrets/"
```

`application.yml`:

```yaml
management:
  endpoint:
    env:
      show-values: NEVER
      keys-to-sanitize: password,secret,token,apikey
    configprops:
      show-values: NEVER
```

**Код (Java/Kotlin): бин, который использует секреты как обычные свойства**

```java
package ext.secrets;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties("db")
public class DbProps {
  private String username; private String password;
  public String getUsername(){ return username; } public void setUsername(String v){ this.username=v; }
  public String getPassword(){ return password; } public void setPassword(String v){ this.password=v; }
}
```

```kotlin
package ext.secrets

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties("db")
data class DbProps(
    val username: String = "",
    val password: String = ""
)
```

---

## Vault/облачные секреты: отборочный обзор; почему не хранить секреты в гите

Секреты в репозитории — риск и долг. Коммиты живут вечно, кэши CI/зеркала/резервные копии множат поверхность атаки. Правильная стратегия — хранить секреты в выделенных хранилищах с аудитом и ротацией: HashiCorp Vault, AWS Secrets Manager/Parameter Store, GCP Secret Manager, Azure Key Vault. Эти системы предоставляют контроль доступа, TTL, версионирование и нативную интеграцию с облачными идентичностями.

Spring Boot интегрируется через Spring Cloud. Для Vault — `spring.config.import=vault://` и блок `spring.cloud.vault.*` (адрес, метод аутентификации, путь к секретам). Биндинг свойств работает так же, как и с YAML: вы описываете `@ConfigurationProperties`, а значения подтягиваются из хранилища при старте. Это позволяет держать baseline в `application.yml`, а секреты перепоручить Vault.

Облачные хранилища (AWS/GCP/Azure) подключаются по тому же принципу, но используют провайдерные креденшалы (IAM роли, Workload Identity и т.д.). Это упрощает деплой: контейнер получает временные токены от платформы, а приложение — доступ к секретам без хранения ключей.

Ротация — ключевое преимущество: для некоторых типов секретов (DB, API токены) можно применять короткие TTL и автоматическую выдачу нового значения. Приложение лучше перезапускать при ротации (immutable-конфиг), но есть и подходы hot-reload — включайте их точечно и осознанно.

Не смешивайте «секреты» и «настройки» в одном месте. В Vault держите именно секреты; в YAML/ConfigMap — функциональные параметры (таймауты, фичи). Это упрощает управление и аудит.

Наконец, проверяйте отказоустойчивость: при недоступности хранилища старт должен падать (fail-fast), а не продолжаться с пустыми значениями. Подготовьте алерты и «план Б» (например, кэш секретов в init-контейнере), если это критично.

**Gradle (Groovy/Kotlin) — Spring Cloud Vault**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter'
  implementation 'org.springframework.cloud:spring-cloud-starter-vault-config'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter")
  implementation("org.springframework.cloud:spring-cloud-starter-vault-config")
}
```

`application.yml` (Vault, пример):

```yaml
spring:
  config:
    import: "vault://"
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: kubernetes
      kubernetes:
        role: app-role
      generic:
        enabled: true
        application-name: myapp
```

**Код (Java/Kotlin): биндинг секрета из Vault**

```java
package ext.vault;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties("secrets.pay")
public class PaySecretProps {
  private String apiKey;
  public String getApiKey(){ return apiKey; }
  public void setApiKey(String v){ this.apiKey=v; }
}
```

```kotlin
package ext.vault

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties("secrets.pay")
data class PaySecretProps(val apiKey: String = "")
```

---

## Разделение конфигов: общий «baseline» + профильные/региональные/тенантные оверрайды

Хорошая конфигурация — слоистая. В `application.yml` держите baseline, одинаковый для всех окружений: имена сервисов, таймауты «по умолчанию», флаги фич с безопасными значениями. Поверх накладывайте профильные файлы (`application-prod.yml`, `application-dev.yml`) — инфраструктурные различия, параметры производительности. Дальше — региональные (`application-eu.yml`, `application-us.yml`) или тенантные оверрайды. Такая стратификация делает изменения локальными и прогнозируемыми.

Для сложных компаний удобно использовать `spring.profiles.group`: объявите логические сборки вроде `prod-eu = prod,eu,kafka,s3`. Тогда активация одного профиля включает весь набор нужных «сигналов». В результате `SPRING_PROFILES_ACTIVE=prod-eu` подключит и prod-конфиги, и европейские хосты/лимиты, и интеграции.

Отдельные пакеты конфигов можно импортировать через `spring.config.import`: это даст модульность (например, общий YAML для региона и поверх — «тенантные» куски). Монтирование дополнительных локаций (`spring.config.additional-location`) тоже уместно, если конфиги лежат вне jar’а.

Тенантные различия лучше выносить из кода. Если бизнес-логика неизбежно меняется по тенантам — инкапсулируйте политику в стратегии, а выбор стратегии передавайте через конфиг («ключ → реализация»). Это сохраняет чистоту домена и делает поведение прозрачным.

Документируйте слои и приоритеты. В README модуля опишите, что в baseline, что в профилях, где живут региональные значения. Это экономит время всем, кто приходит в проект позже.

И наконец, автоматизируйте проверку конфигов: линтер YAML, тесты на обязательные ключи, smoke-тест контекста с выбранной комбинацией профилей. Конфиг — такой же код, и он заслуживает качества.

**Пример слоёв**

```yaml
# application.yml
client:
  orders:
    base-url: https://orders.default
    timeout-ms: 1000

---
spring:
  config:
    activate:
      on-profile: prod
client:
  orders:
    timeout-ms: 1500

---
spring:
  config:
    activate:
      on-profile: eu
client:
  orders:
    base-url: https://orders.eu.internal
```

`application.yml` (группа):

```yaml
spring:
  profiles:
    group:
      prod-eu: [prod, eu]
```

**Код (Java/Kotlin): сервис получает уже «собранные» значения**

```java
package ext.layered;

import org.springframework.stereotype.Service;
import conf.props.OrdersClientProps;

@Service
public class OrdersClient {
  public OrdersClient(OrdersClientProps props) {
    System.out.println("Orders URL=" + props.getBaseUrl() + " timeout=" + props.getTimeoutMs());
  }
}
```

```kotlin
package ext.layered

import conf.props.OrdersClientProps
import org.springframework.stereotype.Service

@Service
class OrdersClient(props: OrdersClientProps) {
    init { println("Orders URL=${props.baseUrl} timeout=${props.timeoutMs}") }
}
```

---

# 9. Тесты: свойства и профили в тестовой среде

## `@ActiveProfiles` для выбора окружения тестов; `@TestPropertySource` для точечного оверрайда

В интеграционных тестах важно запускать приложение в «правильном» окружении. Аннотация `@ActiveProfiles("test")` активирует профиль `test`, подмешивает `application-test.yml` и включает/отключает тестовые бины. Это избавляет от «ручной» подмены зависимостей и делает поведение детерминированным.

Когда нужен точечный override, используйте `@TestPropertySource`: можно указать inline-пары `key=value` или файл. Эти свойства по приоритету выше, чем YAML — удобно, чтобы подменить один параметр без создания отдельного профильного файла. Такой подход ускоряет локальные эксперименты и делает тесты самодостаточными.

Избегайте смешения «боевых» и «тестовых» конфигов. Если ваш тест зависит от отключения Kafka или имитации внешнего клиента, это должно включаться через профиль `test` и условные бины, а не через случайный `@MockBean` поверх сложного графа. Тогда то же окружение может использоваться в локальном dev-режиме.

Помните про приоритеты: `@TestPropertySource` перекроет `application-test.yml`, а `@DynamicPropertySource` (см. ниже) по факту регистрирует свой `PropertySource` — обычно с самым высоким приоритетом. Это важно, когда вы сочетаете Testcontainers и тестовые YAML’ы.

Для быстрых slice-тестов (`@WebMvcTest`, `@DataJpaTest`) `@ActiveProfiles("test")` тоже работает; но учитывайте, что поднимается урезанный контекст. Не пытайтесь в таких тестах активировать бины, которых там нет по определению.

И не забудьте о чистоте: то, что включено в `test`-профиле, не должно требовать внешних ресурсов. Тесты должны быть воспроизводимыми и автономными.

**Код (Java): профиль + точечные свойства**

```java
package test.env;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.TestPropertySource;

@SpringBootTest
@ActiveProfiles("test")
@TestPropertySource(properties = {
    "client.orders.timeout-ms=2000"
})
class EnvProfileTest {
  @Test void contextLoads() { }
}
```

**Код (Kotlin): аналог**

```kotlin
package test.env

import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.TestPropertySource

@SpringBootTest
@ActiveProfiles("test")
@TestPropertySource(properties = ["client.orders.timeout-ms=2000"])
class EnvProfileTest {
    @Test fun contextLoads() { }
}
```

---

## `DynamicPropertySource` для динамических значений (Testcontainers: URL БД/кафки и т.д.)

Иногда значения нельзя «знать» заранее: URL базы от Testcontainers, случайный порт mock-сервера, путь к временной директории. Аннотация `@DynamicPropertySource` позволяет регистрировать свойства программно, уже во время запуска тестового контекста. Это идеальный партнер Testcontainers: подняли контейнер → записали в `spring.datasource.url`.

Метод с `@DynamicPropertySource` должен быть `static` (Java) или companion (Kotlin) и принимать `DynamicPropertyRegistry`. В него вы добавляете лямбды, которые вернут актуальное значение в момент биндинга. Такой подход гарантирует правильный порядок: контейнер стартует до создания контекста.

С Testcontainers тщательно подбирайте «health» и время ожидания, чтобы не получать флапы. Комбинируйте с `@ActiveProfiles("test")`, где отключены тяжёлые интеграции и включены заглушки.

Для Kafka/Redis/MinIO рецепт тот же: поднимаете контейнер, регистрируете bootstrap URL/порт/credentials, а остальной код «не знает», что он в тесте — он просто читает Spring-свойства.

Если у вас несколько тестов с одинаковыми контейнерами, рассмотрите JUnit-перезапускаемый singleton-контейнер (shared). Это уменьшит время тестов в CI.

И наконец, не забывайте закрывать ресурсы. Testcontainers делает это автоматически по окончании JVM, но аккуратная остановка в `@AfterAll` улучшает стабильность.

**Код (Java): Postgres + `DynamicPropertySource`**

```java
package test.dyn;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;

@SpringBootTest
class PgDynamicTest {
  static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

  @BeforeAll static void start() { pg.start(); }
  @AfterAll  static void stop()  { pg.stop(); }

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
  }

  @Test void ok() { }
}
```

**Код (Kotlin): аналог**

```kotlin
package test.dyn

import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer

@SpringBootTest
class PgDynamicTest {
    companion object {
        private val pg = PostgreSQLContainer("postgres:16-alpine")

        @JvmStatic @BeforeAll fun start() = pg.start()
        @JvmStatic @AfterAll  fun stop()  = pg.stop()

        @JvmStatic
        @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("spring.datasource.url", pg::getJdbcUrl)
            r.add("spring.datasource.username", pg::getUsername)
            r.add("spring.datasource.password", pg::getPassword)
        }
    }

    @Test fun ok() {}
}
```

Gradle (Groovy/Kotlin) — Testcontainers:

```groovy
testImplementation 'org.testcontainers:junit-jupiter'
testImplementation 'org.testcontainers:postgresql'
```

```kotlin
testImplementation("org.testcontainers:junit-jupiter")
testImplementation("org.testcontainers:postgresql")
```

---

## Инициализация данных через профили `test`/`it`: отключение тяжёлых интеграций, включение заглушек

Тестовые профили — лучший способ «облегчить» приложение: отключить Kafka/SMTP/S3, включить фейки и заглушки, изменить таймауты. Бойцовские интеграции не должны мешать тестам, а тестовые — не должны просачиваться в прод. Разделение по профилям решает это прозрачно и воспроизводимо.

Для инициализации данных используйте стандартные механизмы Spring Data: `data.sql`/`schema.sql`, `@Sql` в тестах, профильные бины «загрузчики». В `application-test.yml` можно включить `spring.jpa.hibernate.ddl-auto=create-drop`, чтобы база поднималась чистой. Для больших сценариев лучше писать целевые фикстуры (Test Data Builders) вместо монолитных SQL.

Заглушки оформляйте как полноценные бины под `@Profile("test")`. Тогда доменный код не знает, что его зависимости — фейки; он просто работает через интерфейсы. Это сильно упрощает поддержку и делает тесты читаемыми.

Отдельно выделяют профиль `it` (integration tests), если нужны отличные от `test` настройки (например, включить реальные интеграции, но на «песочнице»). Тогда в CI можно запускать быстрые юниты под `test` и более долгие IT — под `it`.

Следите за идемпотентностью: повторный запуск тестовой инициализации не должен ломать сценарии. Используйте `insert … on conflict`, `merge` или предварительную очистку.

И наконец, отделите «seed» dev-данных от тестовых. Девелоперские данные могут быть богатыми и случайными, в тестах нужны маленькие, детерминированные наборы.

**Код (Java): заглушка под профиль `test`**

```java
package test.stubs;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

public interface Mailer { void send(String to, String body); }

@Component
@Profile("prod")
class SmtpMailer implements Mailer {
  public void send(String to, String body) { /* реальная отправка */ }
}

@Component
@Profile("test")
class FakeMailer implements Mailer {
  public void send(String to, String body) { /* no-op или запись в память */ }
}
```

**Код (Kotlin): аналог**

```kotlin
package test.stubs

import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

interface Mailer { fun send(to: String, body: String) }

@Component @Profile("prod")
class SmtpMailer : Mailer { override fun send(to: String, body: String) { /* real send */ } }

@Component @Profile("test")
class FakeMailer : Mailer { override fun send(to: String, body: String) { /* no-op */ } }
```

`application-test.yml`:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop
```

---

## Контроль детерминизма: фиксированные значения времени/рандома в свойствах

Нестабильные тесты — боль. Часто причина в «подглядывании» к системному времени или в случайных числах без фиксированного seed. Простое лекарство — инжектировать `Clock` и `Random` как бины и конфигурировать их для `test`-профиля: `Clock.fixed(…)`, `Random(seed)`. Тогда код зависит от абстракций, а тесты — детерминированы.

Свойства помогают управлять этими бинами. Например, `time.fixed-enabled=true` включает фиксированные часы, а `random.seed=42` задаёт зерно. В проде эти флаги выключены; в тестах — включены. Это также делает тестовый «режим» документированным и прозрачным.

`Clock` удобен тем, что его можно передавать глубоко в домен — это лучше, чем дёргать `Instant.now()` везде. В коде вы зависите от `Clock`, а в тестах — от его фиктивной реализации. Аналогично с генераторами «случайных» идентификаторов.

Для компонентов, где время — часть контракта (TTL, окна), делайте значение явным параметром, а не скрытой зависимостью. Это уменьшает хрупкость тестов и облегчает reasoning.

Не забывайте проверять «вживание» в проде: фиксированные бины не должны случайно активироваться на stage/prod. Профили и явные свойства — защита от этого.

И наконец, логируйте включение тестовых бинов: по строке в логах старта понятно, что активны «фиктивные часы» — это экономит время при разборе неожиданностей.

**Код (Java): `Clock`/`Random` через свойства и профиль**

```java
package test.determinism;

import org.springframework.context.annotation.*;
import org.springframework.beans.factory.annotation.Value;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.Random;

@Configuration
public class DeterministicConfig {
  @Bean
  @Profile("test")
  Clock clock(@Value("${time.fixed-epoch:2024-01-01T00:00:00Z}") String epoch) {
    return Clock.fixed(Instant.parse(epoch), ZoneOffset.UTC);
  }
  @Bean @Profile("!test") Clock sysClock() { return Clock.systemUTC(); }

  @Bean
  @Profile("test")
  Random rnd(@Value("${random.seed:42}") long seed) { return new Random(seed); }

  @Bean @Profile("!test") Random sysRnd() { return new Random(); }
}
```

**Код (Kotlin): аналог**

```kotlin
package test.determinism

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile
import java.time.Clock
import java.time.Instant
import java.time.ZoneOffset
import java.util.Random

@Configuration
class DeterministicConfig {
    @Bean
    @Profile("test")
    fun clock(@Value("\${time.fixed-epoch:2024-01-01T00:00:00Z}") epoch: String): Clock =
        Clock.fixed(Instant.parse(epoch), ZoneOffset.UTC)

    @Bean @Profile("!test") fun sysClock(): Clock = Clock.systemUTC()

    @Bean
    @Profile("test")
    fun rnd(@Value("\${random.seed:42}") seed: Long): Random = Random(seed)

    @Bean @Profile("!test") fun sysRnd(): Random = Random()
}
```

---

# 10. Практики и анти-паттерны

## Не тащить `Environment` в бизнес-код; предпочитать типобезопасные классы свойств

Прямой доступ к `Environment` внутри доменных сервисов — скрытая связность: сервис завязан на внешний мир и становится труднее тестируемым. К тому же вы теряете типобезопасность и валидацию. Вместо этого опишите класс свойств (`@ConfigurationProperties`) и инжектите его в конфигурационный слой, который собирает инфраструктуру.

Такой подход концентрирует знание о ключах в одном месте, а домен остаётся чистым и переносимым. В тестах можно просто создать объект свойств конструктором и передать в фабрику клиентов. А если ключ переименовали — IDE/процессор метаданных подскажет.

Ещё аргумент: `Environment` подменяет приоритеты источников. Если вы читаете его «на лету», можно случайно схватить неожиданное значение. Класс свойств фиксирует снимок на старте — предсказуемое поведение, соответствующее immutable-подходу.

В редких случаях `Environment` допустим (инфраструктура, ранний старт), но не в домене. Лучшая защита — архитектурные правила/линтер (например, ArchUnit), запрещающие импорт `org.springframework.core.env` в `*.domain`.

Помните, что `@Value` — частный случай той же проблемы. Разумно применять его в конфигурации, но не размазывать по сервисам, где он затрудняет тесты и обзор зависимостей.

И наконец, документируйте это правило в руководстве команды. Соглашение снижает различия стилей и повышает читаемость.

**Код (плохо/хорошо)**

```java
// Плохо
class BadService {
  private final org.springframework.core.env.Environment env;
  BadService(org.springframework.core.env.Environment env) { this.env = env; }
  int limit() { return Integer.parseInt(env.getProperty("feature.limit","10")); }
}

// Хорошо
record FeatureProps(int limit) { }
@org.springframework.boot.context.properties.ConfigurationProperties("feature")
class FeatureCfg {
  private int limit=10;
  public int getLimit(){return limit;}
  public void setLimit(int v){this.limit=v;}
}
class GoodService {
  private final int limit;
  GoodService(FeatureCfg cfg) { this.limit = cfg.getLimit(); }
  int limit() { return limit; }
}
```

```kotlin
// Хорошо (Kotlin)
@org.springframework.boot.context.properties.ConfigurationProperties("feature")
data class FeatureCfg(val limit: Int = 10)

class GoodService(cfg: FeatureCfg) { fun limit() = cfg.limit }
```

---

## Не плодить «глобальные» бины с чтением свойств на лету; читать один раз и инжектить

Иногда встречается «ConfigHolder», который каждое обращение лезет в `Environment` за значением — это создаёт неявные зависимости и нестабильность. Пользователь получает разные значения в разное время, логика превращается в «магический пластилин». Читайте свойства **один раз** (на старте) и инжектите как значения/политики.

Если есть реальная потребность в «горячем» обновлении (редко), отделите это в инфраструктуру: наблюдатель, который обновляет **копию** конфигурации и безопасно публикует новый снапшот зависимостям (через `AtomicReference`). Но даже тогда домен работает с интерфейсом, а не с `Environment`.

Глобальные «читатели конфигов» сложны для тестирования: вам приходится поднимать Spring-контекст, чтобы заглушить поведение. С типобезопасными классами свойств — обычный unit-тест.

Подумайте о производительности. Дешевле держать значения в памяти, чем парсить строку из `Environment` на каждом вызове. Для чисел/времён это особенно заметно под нагрузкой.

И ещё — наблюдаемость. Когда конфиг инжектится явным параметром конструктора, его легко вывести в лог (маскируя секреты) при старте и понимать, с чем живёт инстанс. С динамическими чтениями картинка мутная.

Резюмируя: снимок при старте, инъекция значений, явные зависимости. Простота побеждает.

**Код (Java/Kotlin): «снапшот» конфигурации**

```java
package best.snapshot;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties("limits")
class LimitsProps {
  private int read=100; private int write=50;
  public int getRead(){return read;} public void setRead(int v){this.read=v;}
  public int getWrite(){return write;} public void setWrite(int v){this.write=v;}
}

class QuotaService {
  private final int readLimit, writeLimit;
  QuotaService(LimitsProps p){ this.readLimit=p.getRead(); this.writeLimit=p.getWrite(); }
  boolean canRead(int n){ return n<=readLimit; }
  boolean canWrite(int n){ return n<=writeLimit; }
}
```

```kotlin
package best.snapshot

import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component

@Component
@ConfigurationProperties("limits")
data class LimitsProps(val read: Int = 100, val write: Int = 50)

class QuotaService(p: LimitsProps) {
    private val readLimit = p.read
    private val writeLimit = p.write
    fun canRead(n: Int) = n <= readLimit
    fun canWrite(n: Int) = n <= writeLimit
}
```

---

## Не смешивать профили «поведения» и профили «окружения»; договориться о конвенциях имён

Профили — мощный инструмент, но легко превратить их в «кашу»: `prod-eu-fast-newcache-experiment42`. Такую комбинацию сложно поддерживать и документировать. Разделяйте **окружение** (`dev`, `stage`, `prod`) и **поведение**/интеграции (`kafka`, `s3`, `exp`). Комбинации собирайте группами (`spring.profiles.group`), а не длинными строками.

Ясные конвенции имён экономят время: `feature.*` — флаги поведения, `client.*` — настройки клиентов, `env.*` — параметры среды (регион/тенант). Профили называйте коротко и общеупотребительно. Отдельно — технологические (`webflux`, `jdbc`), если у вас вариативный стек.

Для экспериментов/фич-флагов чаще подходят **свойства**, а не профили. Профиль переключает целые бины, а флаг — конкретное поведение. Смешение ведёт к переинициализации графа там, где можно обойтись конфигом.

Группы профилей помогают включать стабильные связки (например, `prod-eu = [prod, eu, s3, kafka]`). Это документирует композицию и снижает риск забыть ключевую часть.

Условные бины (`@Profile`, `@ConditionalOnProperty`) используйте по назначению: профиль для «живёт/не живёт»; флаг — для «настроить существующее». Такой дизайн проще читать.

И не забывайте проверять комбинации: smoke-тесты контекста под основными группами профилей — дешёвая гарантия работоспособности.

**Код (группы и условия)**

```yaml
spring:
  profiles:
    group:
      prod-eu: [prod, eu, kafka, s3]
feature:
  new-cache:
    enabled: true
```

```java
@org.springframework.context.annotation.Profile("kafka")
@org.springframework.stereotype.Component
class KafkaClient {}

@org.springframework.boot.autoconfigure.condition.ConditionalOnProperty(
  prefix="feature.new-cache", name="enabled", havingValue = "true")
@org.springframework.stereotype.Component
class NewCache {}
```

```kotlin
@org.springframework.context.annotation.Profile("kafka")
@org.springframework.stereotype.Component
class KafkaClient

@org.springframework.boot.autoconfigure.condition.ConditionalOnProperty(
    prefix = "feature.new-cache", name = ["enabled"], havingValue = "true")
@org.springframework.stereotype.Component
class NewCache
```

---

## Документировать контракт конфигов: что обязательно, какие дефолты, что считается секретом; проверка на старте (fail-fast)

Конфиг — это контракт между приложением и средой. Как любой контракт, он должен быть описан: обязательные ключи, их типы и допустимые диапазоны, безопасные дефолты, список секретов. Часть документации можно «поженить» с кодом: Javadoc в `@ConfigurationProperties` попадает в метаданные, а IDE показывает подсказки.

Обязательные параметры валидируйте аннотациями (`@NotBlank`, `@Positive`) и падайте при нарушении — fail-fast дешевле, чем «выстрел в ногу» под нагрузкой. Для составных правил используйте `@AssertTrue` или кастомные валидаторы. Это поднимет уровень доверия к релизу: если приложение стартовало, базовые инварианты соблюдены.

Секреты явно помечайте и не печатайте. Добавьте правила маскировки в Actuator и избегайте «диагностов», которые выводят сырые значения. В README запишите, какие ключи относятся к секретам и где они должны храниться (Vault/Secrets Manager).

Отдельно обозначьте сценарии эволюции: какие ключи deprecated, на что переходить, как долго поддерживается обратная совместимость. Подсказки в метаданных помогут поймать устаревшие ключи прямо в IDE.

В CI можно добавить проверку с `ApplicationContextRunner`: поднять контекст с «пустыми» значениями и убедиться, что старт падает с понятной ошибкой. Это защищает от случайного удаления обязательного ключа.

И наконец, держите примеры: минимальный `application.yml` для dev, шаблон переменных окружения для prod, пример манифеста K8s с `configtree:`. Примеры — лучший ускоритель онбординга.

**Код (Java/Kotlin): валидация + fail-fast)**

```java
package best.contract;

import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;
import java.net.URI;

@Validated
@Component
@ConfigurationProperties("client.pay")
public class PayProps {
  @NotNull URI baseUrl;
  @Min(100) @Max(10_000) int timeoutMs = 1000;
  @NotBlank String apiKey; // секрет

  // getters/setters
  public URI getBaseUrl(){return baseUrl;} public void setBaseUrl(URI v){this.baseUrl=v;}
  public int getTimeoutMs(){return timeoutMs;} public void setTimeoutMs(int v){this.timeoutMs=v;}
  public String getApiKey(){return apiKey;} public void setApiKey(String v){this.apiKey=v;}
}
```

```kotlin
package best.contract

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.NotNull
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.stereotype.Component
import org.springframework.validation.annotation.Validated
import java.net.URI

@Validated
@Component
@ConfigurationProperties("client.pay")
data class PayProps(
    @field:NotNull val baseUrl: URI? = null,
    @field:Min(100) @field:Max(10_000) val timeoutMs: Int = 1000,
    @field:NotBlank val apiKey: String = ""
)
```

`application.yml` (Actuator маскирует):

```yaml
management:
  endpoint:
    env:
      show-values: NEVER
      keys-to-sanitize: password,secret,token,api-key,apikey
    configprops:
      show-values: NEVER
```


