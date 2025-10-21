---
layout: page
title: "Spring MVC: контроллеры и маршрутизация"
permalink: /spring/mvc
---

# 0. Введение

## Что это

Spring MVC — это веб-стек Spring для построения HTTP-приложений по модели «контроллер — представление/ресурс». Он реализует шаблон Front Controller: все запросы попадают в `DispatcherServlet`, который находит обработчик, сериализует/десериализует данные и формирует ответ. Это фундамент для REST API и серверных HTML-приложений.
В контексте Spring Boot 3.x Spring MVC приходит через стартер `spring-boot-starter-web` и конфигурируется авто-конфигурациями; большинство дефолтов «разумные» и готовы к продакшну.
Поддерживаются и классический servlet-стек (Tomcat/Jetty/Undertow), и современная экосистема HTTP: контент-негациация, валидация, CORS, локализация.
Сильная сторона — расширяемость: почти каждый этап (резолвинг хэндлеров, биндинг, конвертация, обработка ошибок) — это SPI/интерфейсы, которые можно заменить или дополнить.
Важно понимать, что Spring MVC — это **императивный** стек; для реактивных нагрузок есть Spring WebFlux, но архитектурные принципы контроллеров и маршрутизации очень похожи.
В продакшне Spring MVC работает десятилетиями, и вокруг него — огромная экосистема инструментов, стратеров, гайдов и практик наблюдаемости/безопасности.

**Код (Java): минимальный «Hello, MVC»**

```java
package intro.mvc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;

@SpringBootApplication
@RestController
class IntroApp {
  @GetMapping("/hello")
  public String hello(@RequestParam(defaultValue = "world") String name) {
    return "Hello, " + name;
  }
  public static void main(String[] args) { SpringApplication.run(IntroApp.class, args); }
}
```

**Код (Kotlin): минимальный «Hello, MVC»**

```kotlin
package intro.mvc

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
@RestController
class IntroApp {
    @GetMapping("/hello")
    fun hello(@RequestParam(defaultValue = "world") name: String) = "Hello, $name"
}
fun main(args: Array<String>) = runApplication<IntroApp>(*args)
```

**Gradle (Groovy/Kotlin DSL)**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

---

## Зачем это

Нам нужен унифицированный способ принимать HTTP-запросы, валидировать входные данные, вызывать бизнес-логику и возвращать корректные статусы/тела. Spring MVC решает это из коробки, снижая «клеевой» код.
Он сокращает time-to-market: контроллер — это только сигнатура и аннотации, всё остальное — default инфраструктура. Это особенно важно для команд, которые быстро развивают API.
Стек интегрирован с экосистемой Spring: валидация (Jakarta Validation), конфиг-свойства, безопасность (Spring Security), наблюдаемость (Actuator/Micrometer), что даёт целостную платформу.
Framework помогает удерживать архитектурную дисциплину: «тонкие контроллеры», отделение DTO от домена, централизованная обработка ошибок. Это делает код читаемым и поддерживаемым.
MVC предоставляет инструменты для «правильной HTTP-семантики»: коды, заголовки, кэширование, ETag/Last-Modified, что напрямую влияет на производительность и UX клиентов.
Наконец, Spring MVC масштабируется организационно: легко выделять модули, версии API, общие компоненты (конвертеры/интерсепторы), что важно в микросервисных ландшафтах.

**Код (Java): контроллер как тонкий фасад к сервису**

```java
package intro.why;

import org.springframework.web.bind.annotation.*;
import jakarta.validation.constraints.*;

@RestController
@RequestMapping("/api/users")
public class UserController {
  private final UserService service;
  public UserController(UserService service) { this.service = service; }

  @PostMapping
  public UserDto create(@RequestBody @Valid CreateUserDto in) {
    return service.create(in);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package intro.why

import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserController(private val service: UserService) {
    @PostMapping
    fun create(@RequestBody @Valid in: CreateUserDto): UserDto = service.create(in)
}
```

---

## Где используется

Spring MVC — стандарт де-факто для REST API в корпоративных Java-проектах: от внутренних CRUD-сервисов до публичных шлюзов.
Он подходит и для серверного HTML (Thymeleaf/Freemarker), если нужен рендер страниц в бэкенде (админки, письма, отчёты).
В микросервисах контроллеры — граница между сетью и доменом: здесь мы переводим HTTP-аргументы в валидированные DTO, а ответы — в контрактные структуры.
Spring MVC эффективно используется для интеграции с внешними системами (webhook-приёмники, callback-эндпоинты), где важны корректные коды и быстрые таймауты.
Также MVC — удобная «песочница» для прототипов API: мало кода, быстрый старт, сразу есть встроенный сервер и автоматическая сериализация JSON.
В комбинации со Spring Security легко реализуются аутентификация/авторизация, CSRF-защита, CORS-политика и другие аспекты веб-безопасности.

**Код (Java): HTML-view с Thymeleaf**

```java
package intro.where;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
class PageController {
  @GetMapping("/welcome")
  public String welcome(Model model) {
    model.addAttribute("title", "Welcome Page");
    return "welcome"; // templates/welcome.html
  }
}
```

**Код (Kotlin): HTML-view**

```kotlin
package intro.where

import org.springframework.stereotype.Controller
import org.springframework.ui.Model
import org.springframework.web.bind.annotation.GetMapping

@Controller
class PageController {
    @GetMapping("/welcome")
    fun welcome(model: Model): String {
        model.addAttribute("title", "Welcome Page")
        return "welcome"
    }
}
```

---

## Какие задачи/проблемы решает

MVC убирает «ручной парсинг» HTTP: вместо чтения `HttpServletRequest` мы описываем сигнатуру, а фреймворк делает биндинг/валидацию. Это минимизирует ошибки и дубли.
Он нормализует обработку ошибок: централизованные `@ControllerAdvice` дают стабильный формат ответов 4xx/5xx и корректные заголовки.
Он решает кросс-срезы — локализация, временные зоны, CORS, контент-негациация — единообразно и расширяемо.
MVC позволяет жить по REST-принципам: правильные методы/коды/идемпотентность/кэширование, что улучшает производительность и предсказуемость клиентов.
Стек облегчает тестирование: `@WebMvcTest` позволяет изолированно тестировать маршруты/валидацию/формат ответов без поднимаемой БД.
Наконец, за счёт SPI вы можете адаптировать стек под домен: типы денег/идентификаторы, кастомные форматтеры/конвертеры — без «вставления костылей» в контроллеры.

**Код (Java): скелет единого формата ошибок через `ControllerAdvice`**

```java
package intro.problems;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;

@RestControllerAdvice
public class GlobalErrors {
  @ExceptionHandler(MethodArgumentTypeMismatchException.class)
  public ResponseEntity<String> badRequest(Exception e) {
    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Invalid parameter");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package intro.problems

import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException

@RestControllerAdvice
class GlobalErrors {
    @ExceptionHandler(MethodArgumentTypeMismatchException::class)
    fun badRequest(e: Exception): ResponseEntity<String> =
        ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Invalid parameter")
}
```

---

# 1. Архитектура Spring MVC

## Поток запроса: `DispatcherServlet` → `HandlerMapping` → `HandlerAdapter` → контроллер → `HttpMessageConverters` → ответ

Запрос попадает в `DispatcherServlet` — это «фронт-контроллер», отвечающий за маршрутизацию и оркестрацию. Он не знает про ваш домен: его задача — найти подходящий обработчик и запустить правильный адаптер.
`HandlerMapping` решает, **какой** обработчик (метод контроллера) соответствует пути/методу/медиатайпу; он учитывает аннотации `@RequestMapping` и приоритеты.
`HandlerAdapter` знает, **как вызвать** найденный обработчик: подготовить аргументы (биндинг параметров, тела, заголовков), вызвать метод и получить результат.
Если метод возвращает объект/`ResponseEntity`, `HttpMessageConverters` берут на себя сериализацию/десериализацию (JSON через Jackson и т.д.). Для view — в ход идёт `ViewResolver`.
На выходе `DispatcherServlet` формирует `HttpServletResponse`: статус, заголовки, тело — строго в соответствии с результатом обработчика и конвертеров.
Любой из этапов расширяем: можно добавить собственные резолверы аргументов, конвертеры или перехватчики, не меняя бизнес-код.

**Код (Java): перехватчик, логирующий ключевые этапы**

```java
package arch.flow;

import org.springframework.lang.Nullable;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.*;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

@Component
public class TracingInterceptor implements HandlerInterceptor {
  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    System.out.println("preHandle: " + request.getRequestURI());
    return true;
  }
  @Override
  public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) {
    System.out.println("afterCompletion: status=" + response.getStatus());
  }
}
```

**Код (Kotlin): регистрация перехватчика**

```kotlin
package arch.flow

import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.InterceptorRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class WebConfig(private val tracing: TracingInterceptor) : WebMvcConfigurer {
    override fun addInterceptors(registry: InterceptorRegistry) {
        registry.addInterceptor(tracing)
    }
}
```

---

## Компоненты: `ViewResolver`, `LocaleResolver/TimeZone`, `HandlerExceptionResolver`, `CorsConfiguration`

`ViewResolver` нужен, когда вы возвращаете имя view из `@Controller` (не `@RestController`): он мапит имя на шаблон (например, Thymeleaf) и рендерит HTML. Его можно заменить/добавить ещё один, если часть страниц генерируется иначе.
`LocaleResolver` и `TimeZone` управляют локалью/часовым поясом запроса: откуда брать (кука, заголовок), как переключать (`LocaleChangeInterceptor`). Это важно для форматирования дат/валют и сообщений валидации.
`HandlerExceptionResolver` централизует перевод исключений в HTTP-ответы: статусы, заголовки, тело. Spring имеет дефолтный резолвер, но в проде почти всегда добавляют свой через `@ControllerAdvice`.
`CorsConfiguration` определяет, кто может обращаться к API из браузера: домены-источники (origins), разрешённые методы/заголовки, время кэширования preflight. Правильная CORS-политика — защита от случайных утечек.
Все эти компоненты подключаются авто-конфигом, но легко настраиваются через `WebMvcConfigurer` — это точка кастомизации «по месту».
Важно документировать решения: локаль по умолчанию, список разрешённых origins, формат ошибок, чтобы фронт и партнёры не гадали.

**Код (Java): базовая конфигурация локали и CORS**

```java
package arch.components;

import org.springframework.context.annotation.*;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.*;
import org.springframework.web.servlet.config.annotation.*;

import java.util.Locale;

@Configuration
public class MvcComponents implements WebMvcConfigurer {
  @Bean public LocaleResolver localeResolver() {
    SessionLocaleResolver r = new SessionLocaleResolver();
    r.setDefaultLocale(Locale.US);
    return r;
  }
  @Bean public LocaleChangeInterceptor lci() {
    LocaleChangeInterceptor i = new LocaleChangeInterceptor();
    i.setParamName("lang");
    return i;
  }
  @Override public void addInterceptors(InterceptorRegistry registry) { registry.addInterceptor(lci()); }
  @Override public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**").allowedOrigins("https://app.example.com").allowedMethods("GET","POST");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package arch.components

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.LocaleResolver
import org.springframework.web.servlet.config.annotation.CorsRegistry
import org.springframework.web.servlet.config.annotation.InterceptorRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor
import org.springframework.web.servlet.i18n.SessionLocaleResolver
import java.util.Locale

@Configuration
class MvcComponents : WebMvcConfigurer {
    @Bean fun localeResolver(): LocaleResolver = SessionLocaleResolver().apply { setDefaultLocale(Locale.US) }
    @Bean fun lci() = LocaleChangeInterceptor().apply { paramName = "lang" }
    override fun addInterceptors(registry: InterceptorRegistry) { registry.addInterceptor(lci()) }
    override fun addCorsMappings(registry: CorsRegistry) {
        registry.addMapping("/api/**").allowedOrigins("https://app.example.com").allowedMethods("GET","POST")
    }
}
```

---

## Конвенция vs конфигурация: `@EnableWebMvc` и кастомизация через `WebMvcConfigurer`

По умолчанию Spring Boot **не** требует `@EnableWebMvc`: он включает MVC-инфраструктуру автоматически и предоставляет расширяемые дефолты. Это «конвенция поверх конфигурации», подходящая для большинства.
Аннотация `@EnableWebMvc` **отключает** авто-конфиг по части MVC и включает «ручной режим». Используйте её только когда осознанно хотите взять на себя ответственность за полный набор MVC-бинов.
Правильный путь — кастомизировать поведение через `WebMvcConfigurer`: добавлять форматтеры/конвертеры, перехватчики, CORS, message-конвертеры, не ломая общий автоконфиг.
Такой подход минимизирует boilerplate и позволяет подключать будущие улучшения Boot без конфликтов.
Если у вас библиотека, предоставляющая MVC-расширения, упаковывайте их в авто-конфигурацию и избегайте «жёсткого» `@EnableWebMvc` в приложении потребителя.
Всегда оставляйте комментарий в коде «почему нам **нужно** `@EnableWebMvc`», если вдруг применяете — это редкий, но допустимый случай.

**Код (Java): добавляем кастомный `Converter` без `@EnableWebMvc`**

```java
package arch.convention;

import org.springframework.core.convert.converter.Converter;
import org.springframework.stereotype.Component;
import java.util.UUID;

@Component
public class UuidConverter implements Converter<String, UUID> {
  @Override public UUID convert(String source) { return UUID.fromString(source); }
}
```

**Код (Kotlin): регистрация конвертера через `WebMvcConfigurer`**

```kotlin
package arch.convention

import org.springframework.context.annotation.Configuration
import org.springframework.format.FormatterRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer
import java.util.UUID

@Configuration
class ConverterConfig(private val uuidConverter: UuidConverter) : WebMvcConfigurer {
    override fun addFormatters(registry: FormatterRegistry) {
        registry.addConverter(uuidConverter)
    }
}
```

---

# 2. Маршрутизация и шаблоны путей

## Аннотации: `@RequestMapping`, `@Get/Post/Put/Delete/PatchMapping`, `consumes/produces`

Маршрутизация в Spring MVC декларативна: вы помечаете классы/методы аннотациями, описывая путь, метод, загружаемый/выгружаемый медиатайп. Это делает контроллер самодокументируемым.
Сокращённые аннотации (`@GetMapping` и др.) — сахар над `@RequestMapping(method=…)` и ими удобнее пользоваться.
`produces` ограничивает тип контента ответа и участвует в контент-негациации (заголовок `Accept` клиента).
`consumes` говорит, какой `Content-Type` вы ожидаете во входном теле, и позволяет безопасно отвергнуть неожиданный формат.
Аннотации можно вешать на класс (общий префикс/настройки) и метод (частные настройки), и они складываются.
Если один и тот же путь должен обслуживать разные медиатайпы/версии — на помощь приходят условия по заголовкам и `produces`.

**Код (Java): базовые маршруты с `produces/consumes`**

```java
package routes.basic;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping(value = "/api/items", produces = MediaType.APPLICATION_JSON_VALUE)
public class ItemController {
  @GetMapping("/{id}")
  public ItemDto get(@PathVariable long id) { return new ItemDto(id, "name"); }

  @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
  public ItemDto create(@RequestBody ItemCreate in) { return new ItemDto(1L, in.name()); }
}
```

**Код (Kotlin): аналог**

```kotlin
package routes.basic

import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/items", produces = [MediaType.APPLICATION_JSON_VALUE])
class ItemController {
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long) = ItemDto(id, "name")
    @PostMapping(consumes = [MediaType.APPLICATION_JSON_VALUE])
    fun create(@RequestBody inDto: ItemCreate) = ItemDto(1, inDto.name)
}
```

---

## Шаблоны: переменные пути, regex, приоритеты, «хвостовые» слэши, `PathPattern` vs Ant-паттерны

Переменные пути (`/users/{id}`) автоматически биндуются к аргументам метода и валидируются по типу (например, `Long`). Это удобнее, чем читать `request.getParameter`.
Поддерживаются regex-ограничения: `/files/{name:.+}` — классический пример «захватить точки». Это полезно, когда часть пути должна соответствовать маске.
Приоритеты маршрутов определяются спецификой: точные пути выше шаблонов; более длинные конкретные сегменты выше коротких. Это важно при перекрывающихся маппингах.
Хвостовые слэши по умолчанию «гибкие», но лучше не полагаться на это и нормализовать правила (или явно включить/выключить «трейлинг слэш»).
В Boot 3 по умолчанию используется новый `PathPatternMatcher` (быстрее и безопаснее, чем старые Ant-паттерны); он иначе трактует некоторые случаи (`**`), поэтому при миграции стоит проверить маппинги.
Тестируйте «словари» путей: конфликтующие шаблоны — частая причина неожиданных хэндлеров.

**Код (Java): regex и «хвостовые» слэши**

```java
package routes.patterns;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/files")
public class FileController {
  @GetMapping("/{name:.+}") // захватываем точки в имени
  public String get(@PathVariable String name) { return "file=" + name; }

  @GetMapping({"/list","/list/"}) // явное разрешение слэша
  public String list() { return "ok"; }
}
```

**Код (Kotlin): включение PathPattern (если нужно явно)**

```kotlin
package routes.patterns

import org.springframework.boot.web.servlet.server.Encoding
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.PathMatchConfigurer
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class PathPatternConfig : WebMvcConfigurer {
    override fun configurePathMatch(configurer: PathMatchConfigurer) {
        // в Boot 3 PathPattern включён по умолчанию; пример для наглядности
        configurer.setUseCaseSensitiveMatch(true)
    }
}
```

---

## Версионирование маршрутов: в URI, через заголовки/`produces`, через медиатайпы

Версионирование в URI (`/v1/...`) — самое явное и простое; хорошо работает для публичных API и позволяет жить нескольким версиям параллельно.
Версионирование заголовком (`X-API-Version`) оставляет чистый URI, но требует от клиентов правильной установки заголовков и усложняет кэширование/прокси.
Версионирование через медиатайпы (vendor MIME, `application/vnd.example.v1+json`) интегрируется с контент-негациацией, но менее очевидно для людей.
Подход можно смешивать: например, крупные несовместимости — `v2` в пути, минорные — через media type. Важно: неизменность контрактов внутри версии.
В Spring MVC всё это выражается в аннотациях: `@RequestMapping(headers=...)`, `produces="application/vnd...."`, разные базовые префиксы на уровне класса.
Документируйте стратегию версии и сроки EOL для старых версий — это управляет ожиданиями клиентов.

**Код (Java): версии в пути и через медиатайп**

```java
package routes.versioning;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/v1/users")
class UserV1Controller {
  @GetMapping("/{id}")
  public UserV1 get(@PathVariable long id) { return new UserV1(id, "Alice"); }
}

@RestController
@RequestMapping(value="/users", produces="application/vnd.demo.user.v2+json")
class UserV2Controller {
  @GetMapping("/{id}")
  public UserV2 get(@PathVariable long id) { return new UserV2(id, "Alice", "Smith"); }
}
```

**Код (Kotlin): версия заголовком**

```kotlin
package routes.versioning

import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/users")
class UserHeaderVersionedController {
    @GetMapping("/{id}", headers = ["X-API-Version=3"])
    fun getV3(@PathVariable id: Long) = UserV3(id, "Alice", "Smith", listOf("admin"))
}
```

---

## CORS: per-handler и глобальная конфигурация; preflight, кэширование, безопасные дефолты

CORS регулирует, может ли браузер фронтенда из домена A обращаться к вашему API в домене B. По умолчанию браузер блокирует «чужие» домены; сервер должен явно разрешить нужные origin/методы/заголовки.
Preflight-запрос (`OPTIONS`) проверяет, позволите ли вы кросс-доменные методы/заголовки. Его можно и нужно кэшировать заголовком `Access-Control-Max-Age`, чтобы снизить нагрузку.
В Spring MVC можно поставить `@CrossOrigin` на класс/метод — это per-handler настройка; для общих правил используйте `WebMvcConfigurer#addCorsMappings`.
Безопасные дефолты: не используйте `*` в проде; перечисляйте явные origins, ограничивайте методы/заголовки, включайте креденшелы только при необходимости.
Помните, что CORS — про браузеры; бэкенд-клиенты его не проверяют. Не путайте с аутентификацией — CORS не «даёт доступ», а только разрешает браузеру сделать запрос.
Тестируйте CORS инструментами браузера/интеграционными тестами — опечатки в заголовках/методах часто приводят к «неочевидным» сбоям на фронте.

**Код (Java): глобальный CORS + точечный `@CrossOrigin`**

```java
package routes.cors;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.config.annotation.*;

@RestController
@RequestMapping("/api/public")
@CrossOrigin(origins = "https://app.example.com", maxAge = 3600)
class PublicController {
  @GetMapping("/ping") public String ping() { return "ok"; }
}

@Configuration
class CorsGlobal implements WebMvcConfigurer {
  @Override public void addCorsMappings(CorsRegistry r) {
    r.addMapping("/api/**")
     .allowedOrigins("https://app.example.com")
     .allowedMethods("GET","POST","PUT","DELETE")
     .maxAge(3600);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package routes.cors

import org.springframework.context.annotation.Configuration
import org.springframework.web.bind.annotation.*
import org.springframework.web.servlet.config.annotation.CorsRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@RestController
@RequestMapping("/api/public")
@CrossOrigin(origins = ["https://app.example.com"], maxAge = 3600)
class PublicController {
    @GetMapping("/ping") fun ping() = "ok"
}

@Configuration
class CorsGlobal : WebMvcConfigurer {
    override fun addCorsMappings(r: CorsRegistry) {
        r.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET","POST","PUT","DELETE")
            .maxAge(3600)
    }
}
```

---

# 3. Дизайн контроллеров

## `@Controller` vs `@RestController`: когда нужен view, когда чистый JSON

`@RestController` = `@Controller + @ResponseBody` по умолчанию: метод возвращает объект → он сериализуется в тело ответа. Это выбор по умолчанию для REST API.
`@Controller` используется, когда вы возвращаете имя представления (view) и хотите рендерить HTML/фрагменты/шаблоны писем. Здесь в помощь `Model` и `ViewResolver`.
Смешивать в одном классе можно, но лучше разделять: REST-ресурсы отдельно, страницы отдельно. Это упрощает тесты и читаемость.
Если в `@Controller` нужен JSON-ответ на часть методов, используйте `@ResponseBody` на этих методах или возвращайте `ResponseEntity<?>`.
Для REST важно явно задавать `produces` и придерживаться «чистой» модели DTO, не протаскивая ORM-сущности в web-слой.
Стандартизируйте подход в проекте: разные типы контроллеров — разные пакеты (`api`, `pages`) и общие конвенции.

**Код (Java): пара контроллеров — HTML и REST**

```java
package design.ctrl;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.bind.annotation.RestController;

@Controller
class PageCtrl {
  @GetMapping("/home")
  public String home(Model m) { m.addAttribute("title","Home"); return "home"; }
}

@RestController
@RequestMapping("/api/ping")
class PingCtrl {
  @GetMapping public java.util.Map<String,String> ping() { return java.util.Map.of("status","ok"); }
}
```

**Код (Kotlin): аналог**

```kotlin
package design.ctrl

import org.springframework.stereotype.Controller
import org.springframework.ui.Model
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@Controller
class PageCtrl {
    @GetMapping("/home")
    fun home(model: Model): String { model.addAttribute("title", "Home"); return "home" }
}

@RestController
@RequestMapping("/api/ping")
class PingCtrl {
    @GetMapping fun ping() = mapOf("status" to "ok")
}
```

---

## Сигнатуры методов: вход (DTO/`@RequestParam`/`@PathVariable`/`@RequestHeader`) и выход (`ResponseEntity`, DTO, `HttpEntity`, `StreamingResponseBody`)

Сигнатура — это контракт: что и откуда мы читаем, и что возвращаем. Входные аргументы аннотируются источником — путь, query, заголовки, тело.
Для тела используйте DTO с валидацией — так вы защищены от «грязных» полей и получаете понятные ошибки.
Выходом удобно делать `ResponseEntity<T>` — он даёт полный контроль над статусом/заголовками/телом. Если достаточно дефолта 200 + сериализуемый объект — можно возвращать DTO напрямую.
Для стриминга больших тел используйте `StreamingResponseBody` — ответ пойдёт по мере записи без буферизации в памяти.
`HttpEntity` — редкий гость; чаще достаточно `ResponseEntity`. Но он полезен для «проброса» заголовков без статуса.
Не перегружайте контроллер бизнес-логикой — сигнатура должна оставаться читаемой, а маппинг простым.

**Код (Java): все виды входа/выхода**

```java
package design.sig;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;
import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/books")
public class BookCtrl {
  @GetMapping("/{id}")
  public ResponseEntity<BookDto> get(@PathVariable long id, @RequestHeader(value="X-Req-Id", required=false) String rid) {
    return ResponseEntity.ok(new BookDto(id, "DDD", "Evans"));
  }

  @PostMapping
  public ResponseEntity<BookDto> create(@Valid @RequestBody BookCreate in) {
    return ResponseEntity.status(HttpStatus.CREATED).body(new BookDto(1,"New","A"));
  }

  @GetMapping(value="/export", produces=MediaType.APPLICATION_OCTET_STREAM_VALUE)
  public StreamingResponseBody export() {
    return os -> { os.write("id,title\n1,DDD\n".getBytes()); os.flush(); };
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package design.sig

import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody

@RestController
@RequestMapping("/api/books")
class BookCtrl {
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long, @RequestHeader("X-Req-Id", required = false) rid: String?)
        = ResponseEntity.ok(BookDto(id, "DDD", "Evans"))

    @PostMapping
    fun create(@Valid @RequestBody inDto: BookCreate)
        = ResponseEntity.status(HttpStatus.CREATED).body(BookDto(1, "New", "A"))

    @GetMapping("/export", produces = [MediaType.APPLICATION_OCTET_STREAM_VALUE])
    fun export(): StreamingResponseBody = StreamingResponseBody { it.write("id,title\n1,DDD\n".toByteArray()) }
}
```

---

## Правило «тонких контроллеров»: бизнес-логика в сервисах; маппинг DTO↔доменные модели

Контроллер — это адаптер HTTP↔домен: он валидирует вход, вызывает сервис и описывает HTTP-часть (статусы/заголовки). Бизнес-правила живут в сервисах.
Это освобождает контроллер от сложной логики и делает тестирование проще: контроллеры — через `MockMvc`, сервисы — юнит-тестами.
Маппинг DTO↔модель — явный и «тупой»: без нежданчиков по lazy-полям, без протаскивания ORM-сущностей наружу.
Сервис должен принимать «чистые» значения (или доменные объекты), а не `HttpServletRequest`; это упрощает переиспользование и переносимость.
Контроллеры становятся стабильными фасадами; изменения бизнес-логики не требуют изменения контрактов, если DTO не меняются.
Облегчается наблюдаемость: в контроллере мы логируем/трассируем, а сервисы концентрируются на правилах.

**Код (Java): тонкий контроллер + сервис**

```java
package design.thin;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderCtrl {
  private final OrderService service;
  public OrderCtrl(OrderService s) { this.service = s; }

  @PostMapping
  public ResponseEntity<OrderDto> create(@RequestBody CreateOrderDto in) {
    var out = service.create(in.toDomain());
    return ResponseEntity.ok(OrderDto.from(out));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package design.thin

import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
class OrderCtrl(private val service: OrderService) {
    @PostMapping
    fun create(@RequestBody inDto: CreateOrderDto): ResponseEntity<OrderDto> =
        ResponseEntity.ok(OrderDto.from(service.create(inDto.toDomain())))
}
```

---

## Идемпотентность и семантика HTTP: правильные коды статуса, `Location`/`ETag`/`Cache-Control`

HTTP-методы имеют семантику: `GET` — безопасен и идемпотентен, `PUT` — идемпотентен, `POST` — может создавать (не идемпотентен), `DELETE` — идемпотентен. Соблюдайте её — это влияет на ретраи/кэширование.
Создание ресурса возвращает `201 Created` и заголовок `Location` на новый ресурс; это облегчает клиентам работу и соответствует стандарту.
Для условных запросов используйте `ETag`/`Last-Modified`: это экономит трафик и синхронизирует состояние клиента/сервера.
Пагинации/поиску полезен `Cache-Control` (например, короткий `max-age`) — дешёвые ответы станут быстрее.
Ошибки должны быть корректными: `400` для валидации, `404` — нет ресурса, `409` — конфликт версии, `429` — лимиты. Это помогает клиентам принимать решение о ретраях.
Идемпотентные «создания» можно оформлять как `PUT /resource/{id}` или `POST` с идемпотентным ключом (Idempotency-Key) в заголовке — особенно при проблемных сетях.

**Код (Java): `201 Created + Location` и `ETag`**

```java
package design.http;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/docs")
public class DocsCtrl {
  @PostMapping
  public ResponseEntity<Void> create(@RequestBody DocCreate in) {
    String id = "42";
    return ResponseEntity.created(URI.create("/api/docs/" + id)).build();
  }

  @GetMapping("/{id}")
  public ResponseEntity<DocDto> get(@PathVariable String id, @RequestHeader(value="If-None-Match", required=false) String inm) {
    String etag = "\"v1\"";
    if (etag.equals(inm)) return ResponseEntity.status(HttpStatus.NOT_MODIFIED).eTag(etag).build();
    return ResponseEntity.ok().eTag(etag).body(new DocDto(id, "content"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package design.http

import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.net.URI

@RestController
@RequestMapping("/api/docs")
class DocsCtrl {
    @PostMapping
    fun create(@RequestBody inDto: DocCreate): ResponseEntity<Void> {
        val id = "42"
        return ResponseEntity.created(URI.create("/api/docs/$id")).build()
    }

    @GetMapping("/{id}")
    fun get(@PathVariable id: String, @RequestHeader("If-None-Match", required = false) inm: String?): ResponseEntity<DocDto> {
        val etag = "\"v1\""
        if (etag == inm) return ResponseEntity.status(HttpStatus.NOT_MODIFIED).eTag(etag).build()
        return ResponseEntity.ok().eTag(etag).body(DocDto(id, "content"))
    }
}
```

# 4. Связывание данных и валидация

## Биндинг: конвертеры и форматтеры (`Converter`, `Formatter`), `@InitBinder`, кастомные типы (деньги/даты/enum)

Связывание (data binding) — это преобразование «сырого» HTTP-входа в типизированные аргументы метода контроллера и DTO. По умолчанию Spring MVC умеет работать со строками, числами, коллекциями и датами базовых форматов. В продакшне почти всегда возникают доменные типы: «деньги», «идентификатор заказа», «код валюты», — которые хочется принимать прямо в сигнатурах, не размазывая парсинг по контроллерам. Это делает код чище и уменьшает количество ошибок.

`Converter<S,T>` — самый лёгкий путь объяснить контейнеру, как превратить строку из URI/параметра в ваш тип. Например, `String → Currency` или `String → OrderId`. Конвертеры регистрируются в `FormatterRegistry`, и дальше MVC будет применять их автоматически при биндинге `@PathVariable`/`@RequestParam`.

`Formatter<T>` дополняет `Converter` поддержкой локализации и симметричным форматированием «туда и обратно». Он полезен для типов, чьё строковое представление зависит от локали и шаблона: «красивые» даты, суммы денег, проценты. Для JSON это реже нужно (за сериализацию отвечает Jackson), но для form-data и HTML-представлений это золотой стандарт.

Иногда требуется «подправить» вход до биндинга: обрезать пробелы, запретить неожиданные поля, настроить кастомный валидатор поля. Для этого у контроллера или `@ControllerAdvice` есть хук `@InitBinder(WebDataBinder)`. Здесь можно указать `setDisallowedFields`/`setAllowedFields`, включить триммеры (`StringTrimmerEditor`) и привязать `PropertyEditor`.

Кастомные типы благотворно влияют на читаемость сигнатуры методов. Вместо десятка `@RequestParam("amountMinor") long` и `@RequestParam("currency") String` мы принимаем `Money`. Ошибки парсинга становятся валидационными ошибками, а не NPE в недрах сервиса. Это ещё и точка для централизованной валидации формата и диапазона.

Важно помнить: для JSON-пейлоадов преобразованием занимается Jackson, поэтому «конвертеры MVC» задействуются для путей/параметров/форм. Для денег в теле запроса добавьте кастомный `JsonSerializer/JsonDeserializer` или используйте value-объекты с «естественной» формой (например, `amountMinor + currency`).

**Gradle (Groovy/Kotlin)**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java: Converter + Formatter + InitBinder**

```java
package mvc.binding;

import org.springframework.core.convert.converter.Converter;
import org.springframework.format.Formatter;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.InitBinder;

import java.math.BigDecimal;
import java.text.ParseException;
import java.util.Currency;
import java.util.Locale;

// Доменный тип денег
public record Money(BigDecimal amount, Currency currency) {}

// Строка "12.34:USD" -> Money
@Component
class MoneyConverter implements Converter<String, Money> {
  @Override public Money convert(String s) {
    String[] parts = s.split(":");
    return new Money(new BigDecimal(parts[0]), Currency.getInstance(parts[1]));
  }
}

// Локальный формат "12,34 EUR" <-> Money (для форм/страниц)
@Component
class MoneyFormatter implements Formatter<Money> {
  @Override public Money parse(String text, Locale locale) throws ParseException {
    String[] p = text.trim().split("\\s+");
    return new Money(new BigDecimal(p[0].replace(',','.')), Currency.getInstance(p[1]));
  }
  @Override public String print(Money m, Locale locale) {
    return m.amount() + " " + m.currency().getCurrencyCode();
  }
}

// Пример InitBinder на контроллере
@Component
class BinderHooks {
  @InitBinder
  public void tune(WebDataBinder binder) {
    binder.setDisallowedFields("**.hack", "password"); // защита
  }
}
```

**Kotlin: Converter + InitBinder**

```kotlin
package mvc.binding

import org.springframework.core.convert.converter.Converter
import org.springframework.stereotype.Component
import org.springframework.web.bind.WebDataBinder
import org.springframework.web.bind.annotation.InitBinder
import java.math.BigDecimal
import java.util.Currency

data class Money(val amount: BigDecimal, val currency: Currency)

@Component
class MoneyConverter : Converter<String, Money> {
    override fun convert(source: String): Money {
        val (num, cur) = source.split(":")
        return Money(BigDecimal(num), Currency.getInstance(cur))
    }
}

@Component
class BinderHooks {
    @InitBinder
    fun tune(binder: WebDataBinder) {
        binder.setDisallowedFields("**.hack", "password")
    }
}
```

---

## Валидация: `@Valid/@Validated`, `BindingResult`, группы, локализованные сообщения

Валидация — следующий слой защиты после парсинга. Аннотации Jakarta Validation (`@NotBlank`, `@Email`, `@Size`) на DTO позволяют гарантировать корректность данных ещё до вызова сервисов. Это превращает «грязный» ввод в понятные клиенту 400-ответы, а не в случайные исключения глубоко в стеке.

В контроллере достаточно поставить `@Valid` перед `@RequestBody`/`@ModelAttribute`, и Spring вызовет валидатор, соберёт ошибки в `BindingResult` и, если есть `@ResponseBody`, вернёт стандартное представление об ошибке (или то, что вы настроили в `@ControllerAdvice`). Для ручного контроля можно принять `BindingResult` аргументом следом за DTO и реагировать самостоятельно.

Группы валидации (`@Validated(Group.class)`) позволяют использовать разные наборы правил в одном DTO для разных операций: создание/обновление/модерация. Это избавляет от дублирования и делает контракты стабильнее. Например, ИНН обязателен при создании, но опционален при обновлении — просто повесьте аннотацию с группой.

Сообщения валидации должны быть локализуемыми. Выносите их в `messages.properties`/`ValidationMessages.properties` и используйте плейсхолдеры. Тогда фронт и операторы увидят понятные тексты, а не «size must be between 1 and 255». Это ещё и юридически важно на публичных API.

Валидация — не только про вход. Через кастомные констрейнты можно проверять согласованность полей (например, `start < end`), принадлежность к whitelisted значениям, корректность форматов, которые не покрываются стандартными аннотациями (IBAN, BIC, PAN).

Интегрируйте валидацию с логированием/метриками: увеличивайте счётчик «валидационные ошибки» с тегом кода ошибки. Это позволит быстро увидеть, когда клиенты массово ошибаются (например, после релиза мобильного приложения).

**Gradle (Groovy/Kotlin)**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java: DTO + группы + контроллер**

```java
package mvc.validation;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;

interface Create {}
interface Update {}

record UserIn(
  @NotBlank(groups = {Create.class}) String email,
  @Size(min=8, groups = {Create.class}) String password,
  @Min(value=0, groups = {Create.class, Update.class}) int age
) {}

@RestController
@RequestMapping("/api/users")
class UserCtrl {
  @PostMapping
  public Object create(@RequestBody @Validated(Create.class) UserIn in) { return in; }

  @PutMapping("/{id}")
  public Object update(@PathVariable long id, @RequestBody @Validated(Update.class) UserIn in, BindingResult br) {
    if (br.hasErrors()) return br.getAllErrors();
    return in;
  }
}
```

**Kotlin: DTO + локализация**

```kotlin
package mvc.validation

import jakarta.validation.Valid
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Size
import org.springframework.validation.BindingResult
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*

interface Create
interface Update

data class UserIn(
    @field:NotBlank(groups = [Create::class]) val email: String?,
    @field:Size(min = 8, groups = [Create::class]) val password: String?,
    @field:Min(0, groups = [Create::class, Update::class]) val age: Int = 0
)

@RestController
@RequestMapping("/api/users")
class UserCtrl {
    @PostMapping fun create(@RequestBody @Validated(Create::class) inDto: UserIn) = inDto
    @PutMapping("/{id}")
    fun update(@PathVariable id: Long, @RequestBody @Validated(Update::class) inDto: UserIn, br: BindingResult) =
        if (br.hasErrors()) br.allErrors else inDto
}
```

---

## Защита от «грязных» данных: whitelisting полей, чтение только нужных свойств, размеры коллекций

Классическая проблема — «пере-биндинг»: клиент прислал JSON с лишними полями, а Jackson по умолчанию их проигнорирует, и вы об этом даже не узнаете. Хуже, когда `WebDataBinder` в form-data пытается связать нежелательные свойства. В MVC есть два слоя защиты: whitelisting/blacklisting полей на уровне `WebDataBinder` и запрет неизвестных свойств на уровне Jackson.

Whitelisting в `@InitBinder` через `setAllowedFields` позволяет разрешить только ожидаемые поля. Любой посторонний ключ будет проигнорирован и может попасть в `BindingResult` как ошибка — это честнее, чем молча его съесть. Этот приём особенно полезен для form-data и частичных обновлений.

Для JSON включите Jackson-флаг «fail on unknown properties» — пусть парсер падёт с 400 при неизвестном поле. Это дисциплинирует клиентов и предотвращает «тихий дроп» важных данных. На уровне DTO можно указать `@JsonIgnoreProperties(ignoreUnknown = false)` — это локальный контроль для конкретных классов.

Ограничивайте размеры коллекций в DTO: `@Size(max=...)` на списках и картах. Без этого атакующий может прислать мегабайты массива, и сериализация/десериализация станет узким местом. Аналогично, ограничьте длину строк и глубину вложенности (см. раздел про Jackson constraints ниже).

Часть полей вообще не должна приходить снаружи (например, статусы и внутренние флаги). Не мапьте их из входного DTO; назначайте на стороне сервиса. Это исключает горизонтальную эскалацию, когда клиент пытается выставить себе «isAdmin=true».

Наконец, включите чёткую обработку валидационных ошибок и логируйте шаблоны «лишних полей», чтобы видеть, кто именно шлёт мусор: ваша же старая мобильная версия или внешний партнёр.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies { implementation 'com.fasterxml.jackson.core:jackson-databind' }
```

```kotlin
dependencies { implementation("com.fasterxml.jackson.core:jackson-databind") }
```

**Java: Binder whitelist + Jackson строгий режим**

```java
package mvc.sanitize;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;

import jakarta.validation.constraints.Size;
import java.util.List;

@JsonIgnoreProperties(ignoreUnknown = false)
record ProductIn(@Size(max=64) String name, @Size(max=5) List<String> tags) {}

@RestController
@RequestMapping("/api/products")
class ProductCtrl {
  @InitBinder
  public void allowOnly(WebDataBinder b) { b.setAllowedFields("name","tags"); }

  @PostMapping public ProductIn create(@RequestBody ProductIn in) { return in; }
}
```

**Kotlin: аналог**

```kotlin
package mvc.sanitize

import com.fasterxml.jackson.annotation.JsonIgnoreProperties
import jakarta.validation.constraints.Size
import org.springframework.web.bind.WebDataBinder
import org.springframework.web.bind.annotation.*

@JsonIgnoreProperties(ignoreUnknown = false)
data class ProductIn(
    @field:Size(max = 64) val name: String,
    @field:Size(max = 5) val tags: List<String> = emptyList()
)

@RestController
@RequestMapping("/api/products")
class ProductCtrl {
    @InitBinder
    fun allowOnly(b: WebDataBinder) { b.setAllowedFields("name", "tags") }

    @PostMapping fun create(@RequestBody inDto: ProductIn) = inDto
}
```

---

# 5. Сериализация/десериализация и контент-негациация

## `HttpMessageConverters`: JSON (Jackson), XML (опционально), byte[]/стримы

`HttpMessageConverter` — это слой, который превращает тело запроса/ответа в объекты и обратно. Spring Boot автоматически регистрирует набор конвертеров: Jackson для JSON, (опционально) JAXB/Jackson XML, строковый, байтовый, ресурсный, и т.д. Вы почти никогда не создаёте их вручную — достаточно объявить зависимость, и автоконфиг всё подключит.

Рабочая связка для JSON — `MappingJackson2HttpMessageConverter`. Он учитывает `Content-Type`/`Accept` и вызывает сконфигурированный `ObjectMapper`. Для файлов и потоков есть `ByteArrayHttpMessageConverter` и `ResourceHttpMessageConverter`: можно вернуть `Resource`/`InputStreamResource`, и фреймворк выставит правильные заголовки и отправит байты.

Своё расширение уместно, когда нужен «нестандартный» формат: CSV, protobuf, фиксированные записи. Тогда реализуйте `AbstractHttpMessageConverter<T>` и зарегистрируйте его через `WebMvcConfigurer#extendMessageConverters`. Постарайтесь держать такие конвертеры в инфраструктурном модуле, а не в каждом сервисе.

Помните, что конвертеры работают *после* выборки хэндлера и *до* сериализации: значит, ошибки парсинга попадут в `HandlerExceptionResolver`, и вы сможете ответить `400` с дружелюбным описанием. Не перехватывайте их «вручную» в контроллерах — это ломает единообразие.

Не смешивайте задачу конвертера и задачу бизнес-валидации: конвертер отвечает за синтаксис (JSON корректен, типы совпадают), а валидация — за семантику (значение в диапазоне, комбинация полей допустима). Это помогает правильно строить ответы и метрики.

Для производительности используйте стриминговые ответы (`StreamingResponseBody`) и ресурсные конвертеры, чтобы не буферизовать всё целиком в памяти. Особенно это критично для выгрузки отчётов и архивов.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies { implementation 'com.fasterxml.jackson.core:jackson-databind' }
```

```kotlin
dependencies { implementation("com.fasterxml.jackson.core:jackson-databind") }
```

**Java: простой CSV-конвертер**

```java
package mvc.conv;

import org.springframework.http.*;
import org.springframework.http.converter.*;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.List;

record CsvRow(String a, String b) {}

@Component
class CsvMessageConverter extends AbstractHttpMessageConverter<List<CsvRow>> {
  public CsvMessageConverter() { super(new MediaType("text", "csv", StandardCharsets.UTF_8)); }
  @Override protected boolean supports(Class<?> clazz) { return List.class.isAssignableFrom(clazz); }
  @Override protected List<CsvRow> readInternal(Class<? extends List<CsvRow>> clazz, HttpInputMessage input) { throw new UnsupportedOperationException(); }
  @Override protected void writeInternal(List<CsvRow> rows, HttpOutputMessage output) throws IOException {
    var out = output.getBody();
    for (CsvRow r : rows) out.write((r.a()+","+r.b()+"\n").getBytes(StandardCharsets.UTF_8));
  }
}
```

**Kotlin: регистрация конвертера**

```kotlin
package mvc.conv

import org.springframework.context.annotation.Configuration
import org.springframework.http.converter.HttpMessageConverter
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class ConverterConfig(private val csv: CsvMessageConverter) : WebMvcConfigurer {
    override fun extendMessageConverters(converters: MutableList<HttpMessageConverter<*>>) {
        converters.add(0, csv) // приоритетнее
    }
}
```

---

## Настройка Jackson: `ObjectMapper`, модуль JSR-310 (JavaTime), Kotlin-модуль, нуллабельность и точности

Jackson — основной «двигатель» JSON в Spring MVC. В Boot его настраивают через бин `ObjectMapper` или через `Jackson2ObjectMapperBuilderCustomizer`. Минимум — включить поддержку JavaTime (`jackson-datatype-jsr310`), иначе локальные даты/время будут сериализоваться числами (epoch millis) или вы получите ошибки.

Для Kotlin-проектов подключайте `jackson-module-kotlin`: он понимает data-классы, non-null по типам и default-параметры конструктора. Это резко снижает количество «проколотов» мест с nullable и делает DTO-код компактнее.

Важный аспект — точности и форматы. Денежные и идентификаторы лучше сериализовать как строки, чтобы не потерять точность в JS-клиентах. Настраивайте `SerializationFeature.WRITE_BIGDECIMAL_AS_PLAIN` и кастомные сериализаторы для BigDecimal, если требуется.

По умолчанию Boot 3 мягко относится к неизвестным полям, но мы выше договорились ужесточить это: `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES = true`. Аналогично, можно включить `FAIL_ON_NUMBERS_FOR_ENUMS`, чтобы Enum не принимал «случайные» числа.

Для защиты от «больших» JSON используйте `StreamReadConstraints` Jackson: ограничьте глубину вложенности, максимальную длину строки и число имён полей. Это даст базовую защиту от JSON-бомб и чрезмерной вложенности.

Не забывайте про модуль параметров `jackson-module-parameter-names` для корректной работы с Java record/конструкторами без аннотаций. В Java 17 и Boot 3 сериализация/десериализация record работает из коробки, но модуль не повредит.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies {
  implementation 'com.fasterxml.jackson.core:jackson-databind'
  implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
  implementation 'com.fasterxml.jackson.module:jackson-module-parameter-names'
  implementation 'com.fasterxml.jackson.module:jackson-module-kotlin'
}
```

```kotlin
dependencies {
  implementation("com.fasterxml.jackson.core:jackson-databind")
  implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
  implementation("com.fasterxml.jackson.module:jackson-module-parameter-names")
  implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}
```

**Java: кастомизация `ObjectMapper`**

```java
package mvc.jackson;

import com.fasterxml.jackson.core.StreamReadConstraints;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JacksonConfig {
  @Bean ObjectMapper objectMapper() {
    ObjectMapper m = JsonMapper.builder()
      .enable(MapperFeature.ACCEPT_CASE_INSENSITIVE_ENUMS)
      .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true)
      .build();
    m.registerModule(new JavaTimeModule());
    m.getFactory().setStreamReadConstraints(
        StreamReadConstraints.builder().maxNestingDepth(200).maxStringLength(256_000).build());
    return m;
  }
}
```

**Kotlin: `Jackson2ObjectMapperBuilderCustomizer`**

```kotlin
package mvc.jackson

import com.fasterxml.jackson.databind.DeserializationFeature
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class JacksonConfig {
    @Bean
    fun customize(): Jackson2ObjectMapperBuilderCustomizer = Jackson2ObjectMapperBuilderCustomizer {
        it.featuresToEnable(com.fasterxml.jackson.databind.MapperFeature.ACCEPT_CASE_INSENSITIVE_ENUMS)
        it.featuresToDisable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES) // или включить — по политике
        it.modules(JavaTimeModule())
    }
}
```

---

## `produces/consumes`, выбор медиатайпа, корректные `Accept`/`Content-Type`

Контент-негациация — механизм согласования формата между клиентом и сервером. Клиент объявляет, что «умеет» принимать (`Accept`), сервер — что «умеет» отдавать (`produces`). Если пересечения нет — 406 Not Acceptable. Это честнее, чем молча отдавать «что-то одно», и помогает клиентам избежать сюрпризов.

В аннотациях контроллера указывайте `produces` и `consumes` явно. Это документирует контракт и защищает от неожиданных конвертеров. Например, «вход только JSON, выход — JSON и CSV по запросу». Для загрузки файлов используйте `multipart/form-data` и отдельные эндпоинты.

Для версионирования через медиатайпы используйте «vendor» типы (`application/vnd.company.v1+json`). Они участвуют в той же негациации, и сервер подберёт нужный метод по `produces`. Это удобно, если не хотите плодить `/v1`, `/v2`.

Ошибки негациации — хорошая почва для 400/406/415 ответов: «не тот `Content-Type`/`Accept`». Давайте детальные сообщения, чтобы клиенту было понятно, как исправить вызов. Это снижает накладные расходы на поддержку.

В логах сохраняйте `Accept`/`Content-Type` хотя бы для ошибок: это сильно ускоряет расследование. Часто проблемы связаны с пропавшими заголовками после прокси.

В тестах проверяйте негациацию `MockMvc` или WebTestClient: шлите разные заголовки и убеждайтесь, что получаете 406/415 там, где должны. Так вы закрепите дисциплину контента.

**Java: разные `produces` на одном ресурсе**

```java
package mvc.neg;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

import java.util.List;

record ReportRow(String id, long count) {}

@RestController
@RequestMapping("/api/report")
class ReportCtrl {
  @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
  public List<ReportRow> json() { return List.of(new ReportRow("A", 10)); }

  @GetMapping(produces = "text/csv")
  public String csv() { return "id,count\nA,10\n"; }

  @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
  public void acceptJson(@RequestBody ReportRow row) {}
}
```

**Kotlin: проверка 406/415 в тесте (фрагмент)**

```kotlin
// иллюстрация использования медиатайпов
@RestController
@RequestMapping("/api/ping")
class PingCtrl {
    @GetMapping(produces = [MediaType.APPLICATION_JSON_VALUE]) fun ok() = mapOf("ok" to true)
    @PostMapping(consumes = [MediaType.APPLICATION_JSON_VALUE]) fun accept(@RequestBody body: Map<String,Any>) {}
}
```

---

## Ограничение размера тела, защита от «больших» JSON (макс. глубина/длина)

Большие тела запросов бьют по памяти, CPU и GC. Грубый, но эффективный способ — ограничить размер тела на периметре (Ingress/Nginx: `client_max_body_size`, API Gateway: лимиты). В самом приложении полезно быстро отвергать запросы с огромным `Content-Length`, не передавая их на парсинг Jackson.

В MVC удобно поставить `OncePerRequestFilter`, который сравнит `Content-Length` с порогом (например, 2 МБ) и вернёт 413 Payload Too Large. Для «chunked» можно считать первые байты в обёртке запроса и остановиться, если порог превышен — но это уже дороже.

На уровне Jackson включите **ограничения чтения** (`StreamReadConstraints`): максимальная глубина вложенности (чтобы защититься от JSON с тысячами массивов) и максимальная длина строки (чтобы не держать мегабайты в heap). Это дешёвая страховка.

Для multipart загрузок размеры настраиваются отдельными свойствами Spring (`spring.servlet.multipart.max-file-size`, `max-request-size`). Не путайте с JSON: на них эти свойства не влияют.

Помните, что «большие» ответы тоже опасны: их лучше стримить (`StreamingResponseBody`) и кэшировать на стороне клиента/прокси. Добавляйте `Content-Length` и `Content-Disposition`, чтобы клиенты корректно воспринимали скачивания.

Наконец, логируйте отказы по размеру с минимумом деталей (пусть не будут «публичными» данными). Отдельная метрика «отвергнуто по размеру» поможет увидеть злоупотребления и настроить периметр.

**Java: фильтр, режущий большие тела**

```java
package mvc.limit;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class ContentLengthGuard extends GenericFilter {
  private static final long MAX = 2L * 1024 * 1024; // 2 MB
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    HttpServletResponse w = (HttpServletResponse) res;
    long len = r.getContentLengthLong();
    if (len > 0 && len > MAX) {
      w.setStatus(HttpStatus.PAYLOAD_TOO_LARGE.value());
      w.getWriter().write("Payload too large");
      return;
    }
    chain.doFilter(req, res);
  }
}
```

**Kotlin: свойства multipart и Jackson constraints**

```kotlin
# application.yml (фрагмент)
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

```kotlin
package mvc.limit

import com.fasterxml.jackson.core.StreamReadConstraints
import com.fasterxml.jackson.databind.ObjectMapper
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class LimitConfig {
    @Bean
    fun hardenedMapper(): ObjectMapper =
        ObjectMapper().apply {
            factory.streamReadConstraints = StreamReadConstraints.builder()
                .maxNestingDepth(200)
                .maxStringLength(256_000)
                .build()
        }
}
```

---

# 6. Параметры, заголовки, куки и сессии

## `@RequestParam` (обязательные/дефолты), `@PathVariable`, `@RequestHeader`, `@CookieValue`

Сигнатура метода контроллера должна явно показывать источник каждого значения. `@PathVariable` — из сегмента пути, `@RequestParam` — из query/form, `@RequestHeader` — из заголовков, `@CookieValue` — из кук. Чёткая аннотация экономит время чтения и исключает сюрпризы.

У `@RequestParam` есть `required` и `defaultValue`. Если параметр обязателен — пусть будет `required=true` (по умолчанию), и фреймворк сам вернёт 400 при отсутствии. Если необязателен — поставьте `defaultValue` и работайте со значением без `Optional`.

`@RequestHeader` также может быть обязательным — это удобно для технических заголовков (`X-Request-Id`, `If-None-Match`). В ответе полезно зеркалить часть этих заголовков (например, correlation id), чтобы клиенту было проще дебажить.

Куки — инструмент браузерного мира. Для API предпочтительнее заголовки, но иногда куки неизбежны (сессии, CSRF). Читайте их явно через `@CookieValue`, не полагайтесь на глобальное состояние.

Для неизменяемых сложных типов используйте конвертеры: `@PathVariable OrderId id`. Тогда «семантика» видна в сигнатуре, а не прячется в `UUID.fromString()` в теле метода.

Наконец, помните про безопасность: всё, что пришло извне, нужно валидировать (диапазоны, форматы) и логировать с осторожностью (без секретов/PII). Не используйте бездумно `Map<String,String>` «всё подряд».

**Java: примеры всех источников**

```java
package mvc.params;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/articles")
class ArticleCtrl {
  @GetMapping("/{id}")
  public ResponseEntity<String> get(@PathVariable long id,
                                    @RequestParam(defaultValue = "false") boolean withComments,
                                    @RequestHeader(value="If-None-Match", required=false) String inm,
                                    @CookieValue(value="SESSION", required=false) String session) {
    return ResponseEntity.ok("id=" + id + ", withComments=" + withComments + ", inm=" + inm);
  }
}
```

**Kotlin: аналог**

```kotlin
package mvc.params

import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/articles")
class ArticleCtrl {
    @GetMapping("/{id}")
    fun get(
        @PathVariable id: Long,
        @RequestParam(defaultValue = "false") withComments: Boolean,
        @RequestHeader("If-None-Match", required = false) inm: String?,
        @CookieValue("SESSION", required = false) session: String?
    ): ResponseEntity<String> =
        ResponseEntity.ok("id=$id, withComments=$withComments, inm=$inm")
}
```

---

## Мульти-значные параметры, коллекции и карты; пагинация/сортировка как контракт

Query-параметры могут повторяться: `?tag=a&tag=b&tag=c`. Spring автоматически свяжет их в `List<String>`/`Set<String>`. Это удобно для фильтров и поиска. Для пар «ключ=значение» подойдёт `Map<String,String>` или более строгий DTO.

Пагинация — часть контракта API. Даже если вы используете Spring Data, **контракт** лучше задокументировать: `page` (0..N), `size` (1..100), `sort` (поле,направление). В контроллере можно принять эти значения и передать вниз абстракцию «страничного запроса». Обязательно поставьте пределы `size`, чтобы не выстрелить себе в ногу.

Сортировка — потенциальная дыра: не давайте клиенту сортировать по *любому* полю; whitelist’ьте допустимые поля и направления. Иначе злоумышленник будет пытаться сортировать по тяжёлым вычисляемым колонкам.

Не забывайте про согласованность пагинации: `Link`-заголовки (`next`, `prev`) или явные поля `page/size/total/pages`. Это улучшает DX и упрощает клиентские SDK.

При больших ответах верните только то, что нужно (`fields=` или «лёгкие» представления), чтобы не грузить сеть. Но не уходите в «RPC over HTTP»: контракт должен оставаться стабильным и предсказуемым.

Документируйте код состояния: пустая страница — это 200 с пустым списком, а не 404. Ошибки фильтров (неизвестный `sort`) — 400 с детальным описанием.

**Java: коллекции и пагинация**

```java
package mvc.page;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

record Page<T>(List<T> content, int page, int size, long total) {}
record Item(String id) {}

@RestController
@RequestMapping("/api/items")
class ItemsCtrl {
  @GetMapping
  public Page<Item> list(@RequestParam(defaultValue="0") int page,
                         @RequestParam(defaultValue="20") int size,
                         @RequestParam(required=false) List<String> tag,
                         @RequestParam(defaultValue="id,asc") String sort) {
    int safeSize = Math.min(size, 100);
    return new Page<>(List.of(new Item("1")), page, safeSize, 1);
  }
}
```

**Kotlin: аналог**

```kotlin
package mvc.page

import org.springframework.web.bind.annotation.*

data class Page<T>(val content: List<T>, val page: Int, val size: Int, val total: Long)
data class Item(val id: String)

@RestController
@RequestMapping("/api/items")
class ItemsCtrl {
    @GetMapping
    fun list(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(required = false) tag: List<String>?,
        @RequestParam(defaultValue = "id,asc") sort: String
    ): Page<Item> = Page(listOf(Item("1")), page, minOf(size, 100), 1)
}
```

---

## `@SessionAttributes` и работа с сессией осторожно; CSRF-токен (в связке со Spring Security)

Сессии в MVC — мощный инструмент, но в микросервисных API почти не используются: это состояние на сервере, которое плохо масштабируется и усложняет отказоустойчивость. Если всё же используете сессии (например, в server-side HTML), делайте это осознанно: храните минимум, задавайте таймауты, не слагайте большие объекты.

`@SessionAttributes` позволяет пометить атрибуты `Model`, которые должны сохраняться между запросами. Это удобно для «мастеров» (wizard) в web-формах. Но в REST этот подход неуместен — состояние должно быть на клиенте или в явном сторадже (кеш/БД).

Доступ к сессии возможен через `HttpSession`. Не храните там секреты в чистом виде и не используйте сессию как кеш. Для кластеров сессии нужно либо «липкое» балансирование, либо распределённый стор (Spring Session + Redis), что добавляет сложность.

CSRF — защита от подделки запросов из браузера. В REST без куков сессии (stateless) она не нужна, но для форм и страниц с куками — обязательна. Spring Security публикует `CsrfToken` как аргумент/атрибут модели; его нужно рендерить в форму и отправлять обратно в заголовке/параметре.

Если ваш API всё-таки использует куки (например, SPA с cookie-based auth), включите у фронта отправку токена в заголовке `X-CSRF-Token` и проверьте его на сервере. Для REST желательно использовать bearer-токены и отключать CSRF (он не про эти сценарии).

Не забудьте про безопасность сессии: флаги `Secure`, `HttpOnly`, `SameSite` для cookie, регенерация идентификатора после логина, ограничение времени жизни. Это базовая гигиена.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-security' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-security") }
```

**Java: SessionAttributes + чтение CSRF**

```java
package mvc.session;

import org.springframework.security.web.csrf.CsrfToken;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Controller
@SessionAttributes("wizard")
@RequestMapping("/wizard")
public class WizardCtrl {
  @ModelAttribute("wizard") public WizardState init() { return new WizardState(); }

  @GetMapping public String step1(@ModelAttribute("wizard") WizardState s, Model m, CsrfToken token) {
    m.addAttribute("_csrf", token.getToken());
    return "step1";
  }

  @PostMapping public String save(@ModelAttribute("wizard") WizardState s) { return "redirect:/wizard/next"; }
}

class WizardState { public String field; }
```

**Kotlin: HttpSession и осторожное использование**

```kotlin
package mvc.session

import jakarta.servlet.http.HttpSession
import org.springframework.security.web.csrf.CsrfToken
import org.springframework.stereotype.Controller
import org.springframework.ui.Model
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping

@Controller
@RequestMapping("/profile")
class ProfileCtrl {
    @GetMapping
    fun page(session: HttpSession, model: Model, token: CsrfToken): String {
        session.maxInactiveInterval = 900 // 15 минут
        model.addAttribute("_csrf", token.token)
        return "profile"
    }
}
```

# 7. Перехватчики, фильтры и аргумент-резолверы

## `HandlerInterceptor`: аутентификация/авторизация на уровне MVC, миддлвары (логирование, метрики, аудит)

`HandlerInterceptor` — лёгкая «прослойка» вокруг вызова метода контроллера. Он предоставляет три хука: `preHandle` (до биндинга аргументов), `postHandle` (после выполнения, до сериализации) и `afterCompletion` (всегда, даже при исключении). Для задач вроде тонкой аутентификации, аудита ключевых атрибутов запроса или простейшей метрики времени ответа — это идеальное место. В отличие от фильтра, перехватчик знает о «хэндлере» (методе контроллера) и может читать аннотации — удобно для правил «на основании метаданных».

Принцип применения прост: регистрируете перехватчик через `WebMvcConfigurer#addInterceptors`, задаёте путь/исключения (например, не трогаем `/actuator/**`) и реализуете нужную логику. Для измерения времени используйте `System.nanoTime()` и кладите значение как атрибут запроса; в `afterCompletion` посчитайте длительность. Ошибки внутри перехватчика нельзя «глотать»: если нарушили контракт безопасности — возвращайте 401/403 и останавливайте цепочку (`return false`).

Аудит и логирование через перехватчик позволяют концентрировать корреляцию (traceId, userId, ip) в одном месте. Это снимает дублирование из контроллеров и гарантирует одинаковые поля в логах. Не перегружайте перехватчик парсингом тел — к этому месту тело может быть ещё не прочитано/связано; для access-логов фиксируйте только заголовки и путь.

Интерсепторы полезны и для бизнес-правил, выраженных аннотациями. Например, вы можете объявить `@TenantRequired` на методе и в `preHandle` проверить, что хедер `X-Tenant` присутствует и валиден. Так вы держите «авторизационные» инварианты у границы HTTP, прежде чем вход попал в сервис.

Важно различать «что лучше — фильтр или интерсептор?». Если вам нужны чисто HTTP-технические аспекты (сжатие, CORS, безопасность до MVC) — используйте фильтр. Если нужна осознанность о хэндлере и его аннотациях/аргументах — используйте перехватчик.

Наконец, не забывайте про порядок: регистрируйте перехватчик **после** тех, кто должен отработать раньше (например, перехватчик локализации), и защищайте «служебные» пути (actuator, static) от лишней работы — это уменьшит шум и улучшит производительность.

**Java: интерсептор аудита + регистрация**

```java
package mvc.interceptors;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class AuditInterceptor implements HandlerInterceptor {
  private static final String T0 = "t0";

  @Override
  public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
    req.setAttribute(T0, System.nanoTime());
    MDC.put("path", req.getRequestURI());
    MDC.put("method", req.getMethod());
    if (handler instanceof HandlerMethod hm) {
      MDC.put("handler", hm.getMethod().getDeclaringClass().getSimpleName() + "#" + hm.getMethod().getName());
    }
    return true;
  }

  @Override
  public void afterCompletion(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex) {
    long tookMs = (System.nanoTime() - (long) req.getAttribute(T0)) / 1_000_000;
    org.slf4j.LoggerFactory.getLogger("access").info("status={} tookMs={}", res.getStatus(), tookMs);
    MDC.clear();
  }
}
```

```java
package mvc.interceptors;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.*;

@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
  private final AuditInterceptor audit;
  public InterceptorConfig(AuditInterceptor audit) { this.audit = audit; }
  @Override public void addInterceptors(InterceptorRegistry r) {
    r.addInterceptor(audit).addPathPatterns("/api/**").excludePathPatterns("/actuator/**");
  }
}
```

**Kotlin: интерсептор и конфигурация**

```kotlin
package mvc.interceptors

import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.context.annotation.Configuration
import org.springframework.web.method.HandlerMethod
import org.springframework.web.servlet.HandlerInterceptor
import org.springframework.web.servlet.config.annotation.InterceptorRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

class AuditInterceptor : HandlerInterceptor {
    override fun preHandle(req: HttpServletRequest, res: HttpServletResponse, handler: Any): Boolean {
        req.setAttribute("t0", System.nanoTime())
        MDC.put("path", req.requestURI); MDC.put("method", req.method)
        if (handler is HandlerMethod) MDC.put("handler", "${handler.beanType.simpleName}#${handler.method.name}")
        return true
    }
    override fun afterCompletion(req: HttpServletRequest, res: HttpServletResponse, handler: Any, ex: Exception?) {
        val tookMs = ((System.nanoTime() - (req.getAttribute("t0") as Long)) / 1_000_000)
        LoggerFactory.getLogger("access").info("status={} tookMs={}", res.status, tookMs)
        MDC.clear()
    }
}

@Configuration
class InterceptorConfig(private val audit: AuditInterceptor) : WebMvcConfigurer {
    override fun addInterceptors(r: InterceptorRegistry) {
        r.addInterceptor(audit).addPathPatterns("/api/**").excludePathPatterns("/actuator/**")
    }
}
```

Gradle (Groovy/Kotlin) — достаточно Web:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

---

## Servlet-фильтры: технические аспекты до MVC (компрессия, заголовки), порядок исполнения

Servlet-фильтры работают **до** попадания запроса в Spring MVC и видят «сырой» `HttpServletRequest/Response`. Они идеальны для технических задач «уровня контейнера»: добавление/удаление заголовков, ограничение размера тела, защита от небезопасных методов, базовое rate-limiting. Порядок исполнения фильтров контролируется через `@Order` или `FilterRegistrationBean.setOrder()`.

Простое правило выбора: если нужен доступ к хэндлеру/аннотациям/аргументам — используйте `HandlerInterceptor`. Если задача не зависит от MVC и должна работать даже на статике/actuator — используйте фильтр. Например, выставление `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `X-Frame-Options` — фильтровая история.

Ещё одна частая задача — трассировка/корреляция. Вы считываете или генерируете `X-Request-Id`/`traceparent` в фильтре, кладёте его в MDC и в заголовок ответа. Это работает для **всех** ответов, включая 404/500 до MVC, что невозможно перехватчиком.

Сжатие обычно делегируют контейнеру (Tomcat/Nginx), но если вы хотите принудительно добавить `Vary`/`Cache-Control`/безопасные заголовки — тоже фильтр. Главное — не «закрывать» поток ответа раньше времени и соблюдать порядок: сначала фильтр безопасности, затем корреляция, затем ваши дополнительные заголовки.

Будьте осторожны с чтением тела запроса: если вы прочитали InputStream, MVC уже не сможет связать его повторно. Используйте обёртки (`ContentCachingRequestWrapper`), если анализ тела неизбежен, и учитывайте влияние на память.

Наконец, фильтры «видны» всему приложению. Старайтесь держать их короткими и предсказуемыми, иначе расследования перформанса превращаются в боль — тяжёлый фильтр тормозит вообще всё.

**Java: фильтр корреляции + порядок**

```java
package mvc.filters;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationFilter implements Filter {
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    HttpServletResponse w = (HttpServletResponse) res;
    String cid = r.getHeader("X-Request-Id");
    if (cid == null || cid.isBlank()) cid = UUID.randomUUID().toString();
    MDC.put("traceId", cid);
    w.setHeader("X-Request-Id", cid);
    try { chain.doFilter(req, res); } finally { MDC.clear(); }
  }
}
```

**Kotlin: регистрация через `FilterRegistrationBean`**

```kotlin
package mvc.filters

import org.springframework.boot.web.servlet.FilterRegistrationBean
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FilterRegConfig(private val corr: CorrelationFilter) {
    @Bean
    fun reg(): FilterRegistrationBean<CorrelationFilter> =
        FilterRegistrationBean(corr).apply {
            order = Integer.MIN_VALUE
            addUrlPatterns("/*")
        }
}
```

---

## `HandlerMethodArgumentResolver`: собственные аргументы (корреляция, текущий пользователь/тенант)

`HandlerMethodArgumentResolver` позволяет «научить» MVC заполнять аргументы контроллера осмысленными доменными объектами. Например, вместо того чтобы в каждом методе руками читать заголовок `X-User-Id`, вы создаёте аннотацию `@CurrentUser` и резолвер, который превращает заголовок в объект `CurrentUser`.

Резолвер состоит из двух методов: `supportsParameter` — решает, «умею ли я» для данного аргумента; `resolveArgument` — возвращает значение (или кидает исключение для 401/403). Регистрируется он через `WebMvcConfigurer#addArgumentResolvers`. Это делает сигнатуры контроллеров чище и документирует зависимость: «этот метод требует текущего пользователя».

Ещё один кейс — мультитенантность: резолвер берёт `X-Tenant-Id` и кладёт в `TenantContext`. Это удобно для аудита/логов и для дао-слоя, который может читать контекст и добавлять ограничения.

Пишите резолверы **узко**: одна ответственность, минимум работы. Не прячьте тяжёлую бизнес-логику — это усложнит тестирование и уменьшит прозрачность. И обязательно возвращайте корректные HTTP-статусы: «нет токена» — 401, «токен есть, но доступ запрещён» — 403.

Тестируйте резолвер отдельным юнит-тестом (через мок `MethodParameter/NativeWebRequest`) и интеграционно — `@WebMvcTest` с контроллером, использующим аргумент. Это набивает руку и быстро ловит регрессы.

Не перегибайте палку: резолвер — замечательный инструмент, но не заставляйте команду учить «магические» аргументы десятками. Согласуйте короткий список стандартных: `@CurrentUser`, `@Tenant`, `Correlation`.

**Java: @CurrentUser + резолвер**

```java
package mvc.resolvers;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.core.MethodParameter;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.MissingRequestHeaderException;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.*;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

import java.util.List;

public record CurrentUser(String id, String role) {}

@java.lang.annotation.Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@java.lang.annotation.Target({java.lang.annotation.ElementType.PARAMETER})
@interface Current {}

@Component
class CurrentUserResolver implements HandlerMethodArgumentResolver {
  @Override public boolean supportsParameter(MethodParameter p) {
    return p.hasParameterAnnotation(Current.class) && p.getParameterType().equals(CurrentUser.class);
  }
  @Override public Object resolveArgument(MethodParameter p, ModelAndViewContainer m, NativeWebRequest w, WebDataBinderFactory b) {
    HttpServletRequest r = w.getNativeRequest(HttpServletRequest.class);
    String id = r.getHeader("X-User-Id");
    if (id == null) throw new MissingRequestHeaderException("X-User-Id", p);
    String role = r.getHeader("X-User-Role");
    return new CurrentUser(id, role == null ? "user" : role);
  }
}

@Component
class ResolverConfig implements WebMvcConfigurer {
  private final CurrentUserResolver cur;
  ResolverConfig(CurrentUserResolver cur){ this.cur = cur; }
  @Override public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) { resolvers.add(cur); }
}
```

**Kotlin: использование аргумента в контроллере**

```kotlin
package mvc.resolvers

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/me")
class MeCtrl {
    @GetMapping fun me(@Current user: CurrentUser) = mapOf("id" to user.id(), "role" to user.role())
}
```

---

## Локализация и временная зона: `LocaleResolver/LocaleChangeInterceptor`

Локализация повышает UX и уменьшает недоразумения. В MVC локаль и часовой пояс задаются через `LocaleResolver`/`LocaleContextResolver`. Чаще всего это `AcceptHeaderLocaleResolver` (берёт локаль из `Accept-Language`) либо `SessionLocaleResolver` (хранит выбор пользователя в сессии).

Если приложение поддерживает ручное переключение языка, подключайте `LocaleChangeInterceptor` с параметром, например, `lang`. Тогда `GET /page?lang=ru` сменит локаль сессии на русский. Для REST этот подход реже нужен, но на серверных HTML-страницах — нормальная практика.

Часовой пояс — больное место. Лучше всегда хранить и логировать время в UTC, а отображать — в пользовательской зоне. Если вам всё же нужна зона запроса в серверной логике, используйте `TimeZoneAwareLocaleContext` и резолвер, который читает заголовок `Time-Zone`/`X-Timezone`.

Сообщения валидации и ошибки MVC берут локаль из `LocaleResolver`. Поэтому правильно настроенный резолвер — это не только UI, но и корректные тексты 400/422 для пользователей из разных стран.

Не забывайте, что локаль — часть контекста. Не кладите её в статическое состояние; извлекайте из контекста запроса в нужных местах. Это избавит от багов при параллельной обработке.

И, конечно, тестируйте локализацию: `MockMvc` позволяет задавать `locale(Locale.FRANCE)` и проверять, что формат даты/валюты и тексты сообщений корректны.

**Java: AcceptHeaderLocaleResolver**

```java
package mvc.i18n;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver;

import java.util.List;
import java.util.Locale;

@Configuration
public class LocaleConfig {
  @Bean LocaleResolver localeResolver() {
    AcceptHeaderLocaleResolver r = new AcceptHeaderLocaleResolver();
    r.setSupportedLocales(List.of(Locale.US, new Locale("ru")));
    r.setDefaultLocale(Locale.US);
    return r;
  }
}
```

**Kotlin: LocaleChangeInterceptor**

```kotlin
package mvc.i18n

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.config.annotation.InterceptorRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor

@Configuration
class I18nConfig : WebMvcConfigurer {
    @Bean fun localeChangeInterceptor() = LocaleChangeInterceptor().apply { paramName = "lang" }
    override fun addInterceptors(registry: InterceptorRegistry) {
        registry.addInterceptor(localeChangeInterceptor())
    }
}
```

---

# 8. Ошибки и исключения

## `@ControllerAdvice` + `@ExceptionHandler`: единый формат ошибок (например, `ProblemDetail`)

Единый формат ошибок — база для наблюдаемости и DX клиентов. В Spring 6+/Boot 3+ рекомендовано использовать `org.springframework.http.ProblemDetail` (RFC 7807). Он стандартен для REST, легко расширяем и прекрасно ложится в JSON. Через `@RestControllerAdvice` вы описываете правила: «какое исключение во что маппить» и с каким статусом.

Преимущество `ProblemDetail` — поля `type`, `title`, `status`, `detail`, `instance` и возможность добавлять «расширения» (`setProperty`). Это удобно для кодов домена (`errorCode`) и корреляции (`traceId`). Из коробки Boot 3 сериализует `ProblemDetail` в JSON.

Хороший совет: маппируйте «бедные» исключения (например, `MethodArgumentNotValidException`) на человеческие сообщения, а не отдавайте клинику из десяти полей. «detail» должен отвечать на вопрос «что клиент сделал не так и как исправить». Для системных сбоев не раскрывайте внутренности — пишите «временная ошибка», а детали оставляйте в логах.

Стройте иерархию доменных исключений: `DomainException` → конкретные типы. В хэндлере по базовому классу добавляйте общие поля (код, severity). Это уменьшает дублирование и фиксирует норматив.

Не забудьте о медиатайпах: для HTML-страниц ошибки могут быть отданы как view. `@ExceptionHandler` можно перегрузить по `produces`, чтобы на `text/html` отдавать шаблон, а на JSON — `ProblemDetail`. Так вы поддержите оба канала с единым центром знаний.

И наконец, пишите метрики по ошибкам: счётчик с тегами `http.status`, `errorCode`, `exception`. Это обеспечит быстрый «топ ошибок» в Grafana и облегчит расследование.

**Java: базовый `@RestControllerAdvice` c ProblemDetail**

```java
package mvc.errors;

import org.slf4j.MDC;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

@RestControllerAdvice
public class GlobalErrorHandler {

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ProblemDetail validation(MethodArgumentNotValidException ex) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setType(URI.create("https://docs.example.com/errors/validation"));
    pd.setProperty("traceId", MDC.get("traceId"));
    pd.setProperty("violations", ex.getBindingResult().getFieldErrors().stream()
        .map(fe -> fe.getField() + ": " + fe.getDefaultMessage()).toList());
    return pd;
  }

  @ExceptionHandler(ResourceNotFound.class)
  public ProblemDetail notFound(ResourceNotFound ex) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    pd.setDetail(ex.getMessage());
    pd.setProperty("traceId", MDC.get("traceId"));
    return pd;
  }
}

class ResourceNotFound extends RuntimeException {
  public ResourceNotFound(String msg){ super(msg); }
}
```

**Kotlin: аналог с расширениями**

```kotlin
package mvc.errors

import org.slf4j.MDC
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.net.URI

@RestControllerAdvice
class GlobalErrorHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun validation(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            title = "Validation failed"
            type = URI.create("https://docs.example.com/errors/validation")
            setProperty("traceId", MDC.get("traceId"))
            setProperty("violations", ex.bindingResult.fieldErrors.map { "${it.field}: ${it.defaultMessage}" })
        }

    @ExceptionHandler(ResourceNotFound::class)
    fun notFound(ex: ResourceNotFound): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
            detail = ex.message
            setProperty("traceId", MDC.get("traceId"))
        }
}

class ResourceNotFound(msg: String) : RuntimeException(msg)
```

---

## Статусы и заголовки: корректные 4xx/5xx, `Retry-After`, `WWW-Authenticate`

HTTP — это протокол договорённостей. Если вы возвращаете 401, вы **обязаны** добавить `WWW-Authenticate` с описанием схемы (например, `Bearer realm="api", error="invalid_token"`). Для 429 (rate limit) полезно добавить `Retry-After` и кастомные лимитные заголовки (`X-RateLimit-Remaining`). Для 503 при деградации — также `Retry-After`, чтобы клиенты не долбили ядро.

Заголовки важны для «умных» клиентов и прокси. Например, `Cache-Control` при 304/ETag экономит трафик, а `Location` при 201 облегчает навигацию. В ошибках не стесняйтесь быть многословными в заголовках и теле: это всё равно дешевле, чем переписка в саппорте.

Не путайте «ошибку клиента» и «ошибку сервера». Неверная валидация/отсутствие обязательного параметра — 400, а не 500. Отсутствие авторизации — 401 (не 403), запрет доступа — 403. Конфликт версий при `If-Match` — 412. Эти мелочи сильно улучшают DX и кэшируемость.

Если вы используете idempotency-key на POST, при повторе целесообразно отдавать 409/200 с указанием статуса исходной операции. Это помогаете клиенту «слепить» сетевые сбои.

Берегитесь утечек: не кладите в заголовки PII/секреты. Транзакционные идентификаторы/корреляция — ок, но никакой email/номер карты.

И обязательно тестируйте заголовки: `MockMvc` позволяет проверять их присутствие/значение. Это фиксирует контракт и защищает от регрессов.

**Java: 401/429 с заголовками**

```java
package mvc.headers;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class AuthRateAdvice {

  @ExceptionHandler(Unauthorized.class)
  public ResponseEntity<ProblemDetail> unauthorized(Unauthorized e) {
    HttpHeaders h = new HttpHeaders();
    h.add(HttpHeaders.WWW_AUTHENTICATE, "Bearer realm=\"api\", error=\"invalid_token\"");
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNAUTHORIZED);
    pd.setDetail("Authentication required");
    return new ResponseEntity<>(pd, h, HttpStatus.UNAUTHORIZED);
  }

  @ExceptionHandler(TooManyRequests.class)
  public ResponseEntity<ProblemDetail> tooMany(TooManyRequests e) {
    HttpHeaders h = new HttpHeaders();
    h.add(HttpHeaders.RETRY_AFTER, "60");
    h.add("X-RateLimit-Remaining", "0");
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.TOO_MANY_REQUESTS);
    pd.setDetail("Try again later");
    return new ResponseEntity<>(pd, h, HttpStatus.TOO_MANY_REQUESTS);
  }
}

class Unauthorized extends RuntimeException {}
class TooManyRequests extends RuntimeException {}
```

**Kotlin: аналог**

```kotlin
package mvc.headers

import org.springframework.http.HttpHeaders
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class AuthRateAdvice {
    @ExceptionHandler(Unauthorized::class)
    fun unauthorized(e: Unauthorized): ResponseEntity<ProblemDetail> {
        val h = HttpHeaders().apply {
            add(HttpHeaders.WWW_AUTHENTICATE, "Bearer realm=\"api\", error=\"invalid_token\"")
        }
        val pd = ProblemDetail.forStatus(HttpStatus.UNAUTHORIZED).apply { detail = "Authentication required" }
        return ResponseEntity(pd, h, HttpStatus.UNAUTHORIZED)
    }

    @ExceptionHandler(TooManyRequests::class)
    fun tooMany(e: TooManyRequests): ResponseEntity<ProblemDetail> {
        val h = HttpHeaders().apply { add(HttpHeaders.RETRY_AFTER, "60"); add("X-RateLimit-Remaining", "0") }
        val pd = ProblemDetail.forStatus(HttpStatus.TOO_MANY_REQUESTS).apply { detail = "Try again later" }
        return ResponseEntity(pd, h, HttpStatus.TOO_MANY_REQUESTS)
    }
}
class Unauthorized : RuntimeException()
class TooManyRequests : RuntimeException()
```

---

## Маппинг доменных ошибок → HTTP; логирование без утечки PII, корреляция по traceId

Доменные исключения — мост между бизнес-правилами и HTTP. «Недостаточно средств», «заказ уже оплачен», «нарушена инвариантная проверка» — всё это должно быть отражено в коде и маппиться в разумные статусы. Часто подходит `409 Conflict` или `422 Unprocessable Entity`. Главное — стабильный `errorCode`, чтобы клиенты могли программно реагировать.

Логи ошибок должны содержать контекст (traceId, userId, tenant), но обходиться без PII. Вместо «email=ivan@…» логируйте «userId=123». Исключения форматируйте с аккуратным стеком; не пишите в логи тела запросов для 4xx — там могут быть секреты.

Карта «исключение→HTTP» должна жить в одном месте (`@ControllerAdvice`), а не размазываться по контроллерам. Это дисциплинирует и упрощает аудит. В этом же месте увеличивайте счётчик ошибок и отдавайте `ProblemDetail` с `errorCode`.

Корреляция — ключ к расследованию. Возвращайте `X-Request-Id`/`traceId` в ответе и включайте его в `ProblemDetail`. Тогда клиент сразу сможет назвать id инцидента в тикете.

Если вы интегрируете внешний сервис, заведите слой трансляции исключений: сетевые таймауты → `503` (с `Retry-After`), бизнесовые 4xx партнёра → `502/424/409` по политике. Не тяните «сырые» коды наружу.

И, конечно, пишите договор с фронтом/партнёрами: список `errorCode`, статусы, поля `ProblemDetail`. Это ваш «контракт ошибок».

**Java: доменное исключение → 422**

```java
package mvc.domainerrors;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class DomainAdvice {
  @ExceptionHandler(InsufficientFunds.class)
  public ProblemDetail insufficient(InsufficientFunds e) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
    pd.setTitle("Insufficient funds");
    pd.setProperty("errorCode", "PAY-001");
    pd.setProperty("balanceMinor", e.balanceMinor);
    return pd;
  }
}
class InsufficientFunds extends RuntimeException {
  public final long balanceMinor;
  public InsufficientFunds(long bal){ this.balanceMinor = bal; }
}
```

**Kotlin: аналог**

```kotlin
package mvc.domainerrors

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class DomainAdvice {
    @ExceptionHandler(InsufficientFunds::class)
    fun insufficient(e: InsufficientFunds): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Insufficient funds"
            setProperty("errorCode", "PAY-001")
            setProperty("balanceMinor", e.balanceMinor)
        }
}
class InsufficientFunds(val balanceMinor: Long) : RuntimeException()
```

---

## Страница ошибок для HTML и JSON-ответы для API (раздельные хэндлеры)

Иногда одно приложение обслуживает и API, и HTML-страницы. Тогда важно отдавать ошибки **в соответствующем формате**, не смешивая JSON и HTML. Подход №1: два метода `@ExceptionHandler` для одного исключения — с разным `produces`. MVC выберет нужный по `Accept` заголовку.

Подход №2: кастомизировать стандартный `BasicErrorController` (Spring Boot) и `ErrorViewResolver` для HTML. Это удобно, когда вы хотите красивую страницу 404/500 без «рукописного» advice для каждого исключения.

Для API оставляйте `ProblemDetail`/JSON. Для HTML — отдавайте `ModelAndView` с дружелюбным текстом, но без технических подробностей. В обоих случаях не забывайте проставить корректный статус.

Встроенный механизм /error можно оставить для «не перехваченных» исключений; для предсказуемых доменных — используйте `@ControllerAdvice`. Такой дуализм закрывает 100% сценариев.

Не забудьте тест: «при Accept: text/html получаю HTML-страницу, при Accept: application/json — JSON с `application/problem+json`». Это защитит от регрессов при рефакторинге.

И ещё: если фронт — SPA, лучше вообще не отдавать HTML-страницы из API-домена, чтобы не путать клиентов. Держите SPA и API на разных origin/пути.

**Java: два хэндлера, разные `produces`**

```java
package mvc.dual;

import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.ModelAndView;

@RestControllerAdvice
public class DualAdvice {
  @ExceptionHandler(ResourceNotFound.class)
  @ResponseStatus(HttpStatus.NOT_FOUND)
  @RequestMapping(produces = MediaType.APPLICATION_JSON_VALUE)
  public ProblemDetail json(ResourceNotFound e) {
    ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    pd.setDetail(e.getMessage());
    return pd;
  }

  @ExceptionHandler(ResourceNotFound.class)
  @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
  public ModelAndView html(ResourceNotFound e) {
    ModelAndView mv = new ModelAndView("error/404");
    mv.setStatus(HttpStatus.NOT_FOUND);
    mv.addObject("message", e.getMessage());
    return mv;
  }
}
class ResourceNotFound extends RuntimeException { public ResourceNotFound(String m){ super(m);} }
```

**Kotlin: `ErrorViewResolver` для HTML**

```kotlin
package mvc.dual

import jakarta.servlet.http.HttpServletRequest
import org.springframework.boot.web.servlet.error.ErrorAttributeOptions
import org.springframework.boot.web.servlet.error.ErrorAttributes
import org.springframework.boot.web.servlet.error.ErrorViewResolver
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpStatus
import org.springframework.web.servlet.ModelAndView

@Configuration
class ErrorPageConfig(private val errorAttributes: ErrorAttributes) {
    @Bean
    fun htmlErrorResolver(): ErrorViewResolver = ErrorViewResolver { req: HttpServletRequest, status: HttpStatus, _ ->
        val attrs = errorAttributes.getErrorAttributes(req, ErrorAttributeOptions.defaults())
        val mv = ModelAndView("error/${status.value()}")
        mv.addObject("path", attrs["path"])
        mv.addObject("message", attrs["error"])
        mv.status = status
        mv
    }
}
```

---

# 9. Файлы, стриминг и асинхронность в MVC

## Загрузка: `MultipartFile`, лимиты, временные директории, валидация контента/расширений

Загрузка файлов в MVC делается через `MultipartFile` (или `Part` на уровне сервлета). Включение multipart — из коробки в Boot, достаточно `spring.servlet.multipart.*` в `application.yml`. Лимиты ставьте обязательно: размер файла и суммарный размер запроса.

Валидация — не опция. Проверяйте **и** `Content-Type` (может врать), и расширение, и желательно — сигнатуру файла (magic numbers). Всегда сохраняйте файл во временную директорию и только потом перемещайте в «постоянное» место — так вы контролируете время жизни и уборку.

Не читайте файл целиком в память — пользуйтесь `getInputStream().transferTo(...)` и ограничивайте размер. Для вирус-сканирования или метаданных сделайте out-of-band обработку (очередь), чтобы не держать HTTP-соединение.

Возвращайте понятный 400 при нарушении ограничений; Spring сам бросит `MaxUploadSizeExceededException`, перехватите его в `@ControllerAdvice` и превратите в `ProblemDetail`.

И помните про безопасность путей: запрещайте относительные пути/обход директорий; очищайте имена; храните оригинальное имя файла только как метаданные — путь должен быть вашим, не пользовательским.

**`application.yml` (фрагмент)**

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
      location: ${TMP_DIR:/tmp}
```

**Java: приём файла с валидацией**

```java
package mvc.multipart;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.*;
import java.nio.file.*;

@RestController
@RequestMapping("/api/upload")
public class UploadCtrl {
  @PostMapping(consumes = "multipart/form-data")
  public ResponseEntity<String> upload(@RequestPart("file") MultipartFile file) throws IOException {
    if (file.isEmpty()) return ResponseEntity.badRequest().body("Empty file");
    String ext = StringUtils.getFilenameExtension(file.getOriginalFilename());
    if (ext == null || !(ext.equalsIgnoreCase("png") || ext.equalsIgnoreCase("jpg"))) {
      return ResponseEntity.status(HttpStatus.UNSUPPORTED_MEDIA_TYPE).body("Only png/jpg");
    }
    Path tmp = Files.createTempFile("upload-", "." + ext);
    try (InputStream in = file.getInputStream(); OutputStream out = Files.newOutputStream(tmp)) {
      in.transferTo(out);
    }
    return ResponseEntity.ok(tmp.toAbsolutePath().toString());
  }
}
```

**Kotlin: аналог**

```kotlin
package mvc.multipart

import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.util.StringUtils
import org.springframework.web.bind.annotation.*
import org.springframework.web.multipart.MultipartFile
import java.io.InputStream
import java.nio.file.Files
import java.nio.file.Path

@RestController
@RequestMapping("/api/upload")
class UploadCtrl {
    @PostMapping(consumes = ["multipart/form-data"])
    fun upload(@RequestPart("file") file: MultipartFile): ResponseEntity<String> {
        if (file.isEmpty) return ResponseEntity.badRequest().body("Empty file")
        val ext = StringUtils.getFilenameExtension(file.originalFilename ?: "")
        if (ext == null || !(ext.equals("png", true) || ext.equals("jpg", true))) {
            return ResponseEntity.status(HttpStatus.UNSUPPORTED_MEDIA_TYPE).body("Only png/jpg")
        }
        val tmp: Path = Files.createTempFile("upload-", ".$ext")
        file.inputStream.use { input: InputStream -> Files.newOutputStream(tmp).use { input.transferTo(it) } }
        return ResponseEntity.ok(tmp.toAbsolutePath().toString())
    }
}
```

---

## Выгрузка: `Resource`/`InputStreamResource`, `Content-Disposition`, диапазоны (Range) и кэширование

Для скачиваний удобно возвращать `Resource` или `InputStreamResource`. Это даёт MVC управлять потоком, расставлять `Content-Length` и корректно работать с Byte-Range при поддержке клиента. Обязательно задавайте `Content-Disposition` (`attachment; filename="..."`), чтобы браузер не пытался отобразить бинарник.

Поддержка диапазонов позволяет возобновлять прерванные скачивания и стримить медиа. В MVC можно распарсить `Range` заголовок через `HttpRange.parseRanges` и вернуть `206 Partial Content` с нужным диапазоном. В простых кейсах `ResourceHttpRequestHandler` делает это за вас при статике.

Кэширование для неизменяемых файлов — must-have: `ETag`, `Last-Modified`, `Cache-Control: public, max-age=...`. Так вы снизите нагрузку на сервис и ускорите повторные загрузки. Для приватных файлов ставьте `Cache-Control: no-store`.

Не забывайте указать корректный `Content-Type`: определяйте по расширению (`Files.probeContentType`) либо храните в метаданных. Неверный тип ломает UX и может быть рискован (например, SVG может исполнять скрипты).

Для больших файлов отдавайте поток (`InputStreamResource`) и не буферизуйте всё в памяти. Это спасает от OutOfMemory и бережёт GC.

**Java: скачивание с `Content-Disposition` + поддержка Range**

```java
package mvc.download;

import org.springframework.http.*;
import org.springframework.util.MimeTypeUtils;
import org.springframework.web.bind.annotation.*;
import org.springframework.core.io.*;

import java.io.IOException;
import java.nio.file.*;
import java.util.List;

@RestController
@RequestMapping("/api/files")
public class DownloadCtrl {
  @GetMapping("/{name}")
  public ResponseEntity<Resource> download(@PathVariable String name, @RequestHeader(value="Range", required=false) String range) throws IOException {
    Path p = Path.of("/tmp", name).normalize();
    long length = Files.size(p);
    Resource res = new FileSystemResource(p);
    String ctype = Files.probeContentType(p);
    HttpHeaders h = new HttpHeaders();
    h.setContentDisposition(ContentDisposition.attachment().filename(name).build());

    if (range != null) {
      List<HttpRange> ranges = HttpRange.parseRanges(range);
      HttpRange r = ranges.get(0);
      ResourceRegion region = new ResourceRegion(res, r.getRangeStart(length), r.getRangeEnd(length) - r.getRangeStart(length) + 1);
      return ResponseEntity.status(HttpStatus.PARTIAL_CONTENT).headers(h)
          .contentType(MediaType.parseMediaType(ctype != null ? ctype : MimeTypeUtils.APPLICATION_OCTET_STREAM_VALUE))
          .body(region.getResource()); // простая демонстрация; полноценная отдача region — через HttpMessageWriter в WebFlux, в MVC чаще отдают весь файл или используют статику
    }
    return ResponseEntity.ok().headers(h)
        .contentType(MediaType.parseMediaType(ctype != null ? ctype : MimeTypeUtils.APPLICATION_OCTET_STREAM_VALUE))
        .contentLength(length)
        .body(res);
  }
}
```

**Kotlin: простой `InputStreamResource`**

```kotlin
package mvc.download

import org.springframework.core.io.InputStreamResource
import org.springframework.http.ContentDisposition
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.nio.file.Files
import java.nio.file.Path

@RestController
@RequestMapping("/api/files")
class DownloadCtrl {
    @GetMapping("/raw/{name}")
    fun download(@PathVariable name: String): ResponseEntity<InputStreamResource> {
        val path: Path = Path.of("/tmp", name).normalize()
        val isr = InputStreamResource(Files.newInputStream(path))
        val cd = ContentDisposition.attachment().filename(name).build()
        val type = Files.probeContentType(path) ?: MediaType.APPLICATION_OCTET_STREAM_VALUE
        return ResponseEntity.ok()
            .header("Content-Disposition", cd.toString())
            .contentType(MediaType.parseMediaType(type))
            .contentLength(Files.size(path))
            .body(isr)
    }
}
```

---

## Стриминг: `StreamingResponseBody`; server-sent events через `SseEmitter` (ограниченно)

`StreamingResponseBody` даёт «поточный» вывод: вы пишете в `OutputStream`, а ответ уходит клиенту частями, без буферизации целиком. Это незаменимо для отчётов, больших экспортов, серверных генераций. Поток пишется в отдельном треде task executor’а MVC.

Server-Sent Events (`SseEmitter`) — способ пушить клиенту однонаправленные события по HTTP/1.1. Это проще, чем WebSocket, но имеет ограничения (один канал, прокси). Для «лайтовых» обновлений статуса — норм. Не забывайте таймауты и heartbeats, иначе соединения висят.

В обоих случаях **обязательно** ставьте таймауты и проверяйте backpressure со стороны клиента (исключения записи). Логируйте закрытия/ошибки, чтобы не накапливать «зомби»-коннекты.

Не используйте стриминг для того, что естественно бьётся на страницы — лучше отдавать страницу за страницей. Стриминг — для «большого непрерывного» или событий.

С точки зрения безопасности, не вкладывайте в поток секреты; учитывайте, что промежуточные прокси могут буферизовать/обрезать ответы.

И наконец, тестируйте: `MockMvc` стриминг не исполняет полностью; интеграционно проверяйте через RestAssured/HTTP-клиент.

**Java: CSV стриминг**

```java
package mvc.stream;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

@RestController
@RequestMapping("/api/export")
public class ExportCtrl {
  @GetMapping(value="/csv", produces=MediaType.TEXT_PLAIN_VALUE)
  public StreamingResponseBody csv() {
    return out -> {
      out.write("id,name\n".getBytes());
      for (int i=1;i<=1000;i++) {
        out.write((i + ",Item " + i + "\n").getBytes());
        out.flush();
      }
    };
  }
}
```

**Kotlin: SSE**

```kotlin
package mvc.stream

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter
import java.time.Instant
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit

@RestController
class SseCtrl {
    @GetMapping("/api/sse")
    fun sse(): SseEmitter {
        val emitter = SseEmitter(30_000) // 30s timeout
        val exec = Executors.newSingleThreadScheduledExecutor()
        exec.scheduleAtFixedRate({
            try {
                emitter.send("tick ${Instant.now()}")
            } catch (e: Exception) {
                emitter.complete()
                exec.shutdown()
            }
        }, 0, 1, TimeUnit.SECONDS)
        emitter.onCompletion { exec.shutdown() }
        emitter.onTimeout { exec.shutdown() }
        return emitter
    }
}
```

---

## Async-обработка: `Callable`/`DeferredResult` для долгих операций; таймауты и пул

Асинхронность в MVC разгружает сервлетные потоки: вместо того, чтобы «держать» тред во время ожидания I/O, вы возвращаете `Callable` (простая форма) или `DeferredResult` (более гибко). Контейнер освобождает поток, а когда результат готов — снова сериализует ответ.

`Callable<ResponseEntity<?>>` подходит, когда нужно «вынести» работу в executor и просто дождаться результата. `DeferredResult<T>` позволяет завершить запрос из любого места (другой тред, колбэк), а ещё — выставить таймаут и fallback-ответ.

Таймауты — критичны. Выставляйте `spring.mvc.async.request-timeout` или на объекте `DeferredResult`. При таймауте возвращайте 503/504 с подсказкой ретрая. Не держите висящие операции — они съедают ресурсы.

Пул асинхронных задач конфигурируйте: размер, очередь, именование тредов. Через `WebMvcConfigurer#configureAsyncSupport` можно указать `taskExecutor` и общий таймаут. Следите за метриками пула — при истощении начнутся таймауты.

Асинхронность — не серебряная пуля. Для CPU-bound нагрузки она бесполезна; для I/O-bound (внешние запросы, БД с медленными ответами) она экономит треды и повышает throughput.

И, конечно, не используйте асинхронность там, где уместнее реактивный стек (WebFlux). MVC-async — лёгкая надстройка, но не заменяет реактивность.

**`application.yml` (фрагмент)**

```yaml
spring:
  mvc:
    async:
      request-timeout: 10s
```

**Java: `Callable` и `DeferredResult`**

```java
package mvc.async;

import org.springframework.http.ResponseEntity;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.async.DeferredResult;

import java.util.concurrent.*;

@RestController
@RequestMapping("/api/async")
public class AsyncCtrl {
  private final ExecutorService pool = Executors.newFixedThreadPool(10);

  @GetMapping("/callable")
  public Callable<ResponseEntity<String>> callable() {
    return () -> ResponseEntity.ok("done on " + Thread.currentThread().getName());
  }

  @GetMapping("/deferred")
  public DeferredResult<ResponseEntity<String>> deferred() {
    DeferredResult<ResponseEntity<String>> dr = new DeferredResult<>(5000L);
    pool.submit(() -> {
      try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
      dr.setResult(ResponseEntity.ok("async ok"));
    });
    dr.onTimeout(() -> dr.setErrorResult(ResponseEntity.status(503).body("timeout")));
    return dr;
  }
}
```

**Kotlin: конфигурация async-пула**

```kotlin
package mvc.async

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import org.springframework.web.servlet.config.annotation.AsyncSupportConfigurer
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class AsyncConfig : WebMvcConfigurer {
    @Bean
    fun mvcExecutor(): ThreadPoolTaskExecutor = ThreadPoolTaskExecutor().apply {
        corePoolSize = 10; maxPoolSize = 50; setQueueCapacity(200); setThreadNamePrefix("mvc-async-"); initialize()
    }
    override fun configureAsyncSupport(configurer: AsyncSupportConfigurer) {
        configurer.setTaskExecutor(mvcExecutor()); configurer.setDefaultTimeout(10_000)
    }
}
```

---

# 10. Тестирование и эксплуатация MVC-слоя

## Тесты контроллеров: `MockMvc` (MVC), `@WebMvcTest` (slice), проверка негациации/валидации/кодов

`@WebMvcTest` поднимает только MVC-слой (контроллеры, конвертеры, биндинг, валидация), изолируя от БД/сервисов. Для юнитов веб-контрактов это идеальный «срез»: быстро, детерминировано, без шума. Сервисы мокайте через `@MockBean`.

`MockMvc` позволяет программно собирать запрос: путь, метод, заголовки (`Accept`, `Content-Type`), тело, и проверять статус/заголовки/контент. Это удобно для проверки контент-негациации (406/415), корректных кодов и формата ошибок.

Проверяйте валидацию: шлите некорректные DTO и ожидайте 400 с нужными полями в `ProblemDetail`. Так вы защищаете контракт и ловите «случайные» ослабления правил.

Для бинарных ответов проверяйте заголовки (`Content-Disposition`) и длину. Для SSE/стриминга — делайте интеграционные тесты, но хотя бы убедитесь, что endpoint доступен и устанавливает правильный медиатайп.

Старайтесь держать тесты «говорящими»: имя — как спецификация маршрута (`should_return_201_and_location_on_create`). Это облегчает ревью и ускоряет разбор падений CI.

И не забывайте о негативных сценариях: 404 на «чужой id», 403 без прав, 429 при превышении лимита — это такие же контрактные требования, как и «счастливый путь».

**Java: пример `@WebMvcTest` + валидация/негациация**

```java
package mvc.tests;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.*;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = UserCtrl.class)
class UserCtrlTest {
  @Autowired MockMvc mvc;
  @Autowired ObjectMapper om;
  @MockBean UserService userService;

  @Test void create_should_return_400_on_invalid() throws Exception {
    mvc.perform(post("/api/users").contentType(MediaType.APPLICATION_JSON).content("""{"email":""}"""))
       .andExpect(status().isBadRequest());
  }

  @Test void get_should_return_406_when_accept_xml() throws Exception {
    mvc.perform(get("/api/users/1").accept(MediaType.APPLICATION_XML))
       .andExpect(status().isNotAcceptable());
  }
}
```

**Kotlin: аналог**

```kotlin
package mvc.tests

import com.fasterxml.jackson.databind.ObjectMapper
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import org.springframework.test.web.servlet.post

@WebMvcTest(UserCtrl::class)
class UserCtrlTest(
    @Autowired val mvc: MockMvc,
    @Autowired val om: ObjectMapper
) {
    @MockBean lateinit var userService: UserService

    @Test fun `create should return 400 on invalid`() {
        mvc.post("/api/users") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"email":""}"""
        }.andExpect { status { isBadRequest() } }
    }

    @Test fun `get should return 406 for xml`() {
        mvc.get("/api/users/1") { accept = MediaType.APPLICATION_XML }
            .andExpect { status { isNotAcceptable() } }
    }
}
```

---

## Контракт API: стабильные DTO, пагинация/сортировка, обратная совместимость

Контракт — это обещание клиентам. DTO должны быть стабильными: добавляйте поля **только** назад-совместимо (новые nullable/с дефолтами), избегайте переименований. Для «жёстких» изменений — новая версия API. Аннотации Jackson (`@JsonProperty`, `@JsonInclude`) помогают управлять формой.

Пагинация/сортировка — зафиксированный протокол: параметры, лимиты, порядок. Любые изменения — с версией. Отдавайте метаданные (page/size/total) и ссылки (`Link`-заголовки), чтобы клиенты не гадали.

Старайтесь иметь «семейство» DTO: `*Request` и `*Response`. Не таскайте ORM-сущности наружу — это ломает инкапсуляцию и усложняет миграции БД. При необходимости поддерживать старое поведение — применяйте адаптеры/мэпперы.

Депрецируйте аккуратно: помечайте поля в swagger/доках, логируйте использование депрецированных параметров, ставьте конечную дату EOL. Коммуникация важнее кода.

Не завязывайтесь на порядок полей/формат даты «по умолчанию», всегда указывайте ISO-8601 для времени и UTC. Чёткие правила уменьшают количество «серых» багов.

И наконец, держите автотесты контрактов (consumer-driven tests, snapshot-тесты JSON). Это стоп-кран против случайных ломающих изменений.

**Java: DTO с совместимостью**

```java
package mvc.contract;

import com.fasterxml.jackson.annotation.*;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record UserResponse(
  @JsonProperty("id") long id,
  @JsonProperty("email") String email,
  @JsonProperty("display_name") String displayName, // добавлено назад-совместимо
  @JsonProperty("roles") java.util.List<String> roles
) {}
```

**Kotlin: контракт пагинации**

```kotlin
package mvc.contract

import com.fasterxml.jackson.annotation.JsonInclude

@JsonInclude(JsonInclude.Include.NON_NULL)
data class PageResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val total: Long,
    val links: Map<String, String>? = null
)
```

---

## Наблюдаемость: access-логи, метрики `http.server.requests`, трейсинг по заголовкам

Минимум наблюдаемости в MVC — access-логи (метод, путь, статус, tookMs, traceId), метрики `http.server.requests` (они есть из Micrometer «из коробки») и корректная пропагация trace-заголовков (`traceparent`/`X-Request-Id`). Этого достаточно, чтобы «сшить» графы и логи при инциденте.

Access-логи можно делать интерсептором (как выше) или отдельным `Logger` «access». Главное — **не** смешивать с бизнес-логами. Метки `service/env/version` должны совпадать с метриками, чтобы дашборды были единообразными.

Micrometer автоматически собирает тайминги/коды по HTTP. Добавьте полезные теги (component, endpoint pattern), но не поднимайте кардинальность (не тегируйте сырым `path` без шаблона). В Boot 3 для этого используется `server.requests.micrometer.enabled` по умолчанию.

Корреляция: возвращайте `X-Request-Id` в ответах, добавляйте его в каждый лог. Если подключён OpenTelemetry — он сам проставит `traceId/spanId` в MDC, и дальше всё по маслу.

Отдельный граф — «исключения по типам» (метрика `http.server.requests` с `exception`), и «топ-ошибки из логов». Это позволяет быстро понять природу проблемы.

И да, `/actuator/metrics` и `/actuator/health` держите включёнными для каждого сервиса. Это — must-have.

**Java: собственная метрика бизнес-операции**

```java
package mvc.obs;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Service;

@Service
public class BizMetrics {
  private final MeterRegistry reg;
  public BizMetrics(MeterRegistry reg){ this.reg = reg; }
  public void recordOrderCreated(String source) {
    reg.counter("orders.created", "source", source).increment();
  }
}
```

**Kotlin: access-логгер**

```kotlin
package mvc.obs

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse

@Component
class AccessLogFilter : OncePerRequestFilter() {
    private val log = LoggerFactory.getLogger("access")
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        val t0 = System.nanoTime()
        chain.doFilter(req, res)
        val took = (System.nanoTime() - t0) / 1_000_000
        log.info("method={} path={} status={} tookMs={}", req.method, req.requestURI, res.status, took)
    }
}
```

---

## Производительность: кеширование (ETag/Last-Modified), gzip/HTTP/2, ограничение RPS (на периметре)

Производительность HTTP — это прежде всего **не считать лишнее** и **не передавать лишнее**. Для неизменяемых ресурсов и GET-эндпоинтов используйте `ETag`/`Last-Modified` и условные запросы. Это экономит CPU/IO и трафик. Для коллекций применяйте кэширование коротким TTL, если бизнес-допускает.

Сжатие GZIP/Brotli — зона периметра (Ingress/Nginx), но и встроенный Tomcat умеет компрессию. В Boot: `server.compression.enabled=true` и список типов. HTTP/2 ускорит мультиплексирование — включайте на уровне TLS-терминации.

Ограничение RPS (rate-limiting) в приложении возможно (Bucket4j, Resilience4j), но надёжнее — на периметре API Gateway/Ingress Controller. Там проще масштабировать, меньше влияние на GC, и есть централизованная политика.

Старайтесь отвечать «тонко»: не сериализуйте «всё и сразу», предоставьте поля «по требованию» (`fields=`) или облегчённые DTO. Дорогие подсчёты делайте асинхронно или кешируйте.

Для списков используйте пагинацию и разумные лимиты (100–1000). Это и защита, и ускорение. И не забывайте строить профили в проде (JFR/Async-Profiler) при сложных запросах — MVC не отходите, но бизнес-логика может.

Наконец, включите `ShallowEtagHeaderFilter` для API, где тело строится детерминированно. Это «дешёвый» ETag от содержимого ответа; при `If-None-Match` вернётся 304.

**`application.yml` (фрагмент)**

```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,text/plain,text/css,application/javascript
    min-response-size: 1024
  http2:
    enabled: true
```

**Java: ETag-фильтр и Last-Modified**

```java
package mvc.perf;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.*;
import org.springframework.web.filter.ShallowEtagHeaderFilter;

@Configuration
public class PerfConfig {
  @Bean FilterRegistrationBean<ShallowEtagHeaderFilter> etag() {
    FilterRegistrationBean<ShallowEtagHeaderFilter> reg = new FilterRegistrationBean<>(new ShallowEtagHeaderFilter());
    reg.addUrlPatterns("/api/*");
    return reg;
  }
}
```

**Kotlin: контроллер с Last-Modified**

```kotlin
package mvc.perf

import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.time.Instant

@RestController
class CacheCtrl {
    @GetMapping("/api/time")
    fun time(): ResponseEntity<String> =
        ResponseEntity.ok()
            .lastModified(Instant.now().toEpochMilli())
            .eTag("\"v1\"")
            .body("{\"now\":true}")
}
```


