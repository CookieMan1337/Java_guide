---
layout: page
title: "OpenAPI & Swagger"
permalink: /spring/openapi_swagger
---

<!-- TOC -->
* [0. Введение](#0-введение)
  * [Что это такое](#что-это-такое)
  * [Зачем это нужно](#зачем-это-нужно)
  * [Где это используется](#где-это-используется)
  * [Какие задачи и проблемы решает](#какие-задачи-и-проблемы-решает)
* [1. Роль OpenAPI/Swagger в Spring Boot](#1-роль-openapiswagger-в-spring-boot)
  * [Что такое OpenAPI-спецификация, чем она отличается от Swagger UI/аннотаций](#что-такое-openapi-спецификация-чем-она-отличается-от-swagger-uiаннотаций)
  * [Зачем нужна: контракт между сервисом и клиентом, генерация клиентов/серверов, документация](#зачем-нужна-контракт-между-сервисом-и-клиентом-генерация-клиентовсерверов-документация)
  * [Место в SDLC: проектирование, разработка, тесты, релизы, сопровождение](#место-в-sdlc-проектирование-разработка-тесты-релизы-сопровождение)
  * [Подходы: code-first vs contract-first — плюсы и риски каждого](#подходы-code-first-vs-contract-first--плюсы-и-риски-каждого)
  * [Артефакты: YAML/JSON спецификация, интерактивная UI, статически сгенерированные сайты](#артефакты-yamljson-спецификация-интерактивная-ui-статически-сгенерированные-сайты)
  * [Базовые элементы спецификации: paths, operations, components (schemas, responses, security)](#базовые-элементы-спецификации-paths-operations-components-schemas-responses-security)
* [2. Инструменты и установка](#2-инструменты-и-установка)
  * [springdoc-openapi для Spring Boot (MVC/WebFlux), Swagger UI как dev-вью](#springdoc-openapi-для-spring-boot-mvcwebflux-swagger-ui-как-dev-вью)
  * [Подключение зависимостей (starter), минимальная конфигурация через `application.yml`](#подключение-зависимостей-starter-минимальная-конфигурация-через-applicationyml)
  * [Совместимость: версии Spring Boot, Jakarta vs javax, PathPattern vs Ant](#совместимость-версии-spring-boot-jakarta-vs-javax-pathpattern-vs-ant)
  * [Альтернативы UI: ReDoc, RapiDoc; когда и зачем использовать](#альтернативы-ui-redoc-rapidoc-когда-и-зачем-использовать)
  * [Генераторы: OpenAPI Generator / Swagger Codegen (клиенты/серверные заглушки)](#генераторы-openapi-generator--swagger-codegen-клиентысерверные-заглушки)
  * [Профили dev/test/prod для включения/ограничения UI и эндпоинтов спецификации](#профили-devtestprod-для-включенияограничения-ui-и-эндпоинтов-спецификации)
* [3. Базовая конфигурация springdoc](#3-базовая-конфигурация-springdoc)
  * [`OpenAPI` bean: info/contact/license/terms, servers, externalDocs](#openapi-bean-infocontactlicenseterms-servers-externaldocs)
  * [Настройка путей: `springdoc.api-docs.path`, `springdoc.swagger-ui.path`, сортировка/фильтры](#настройка-путей-springdocapi-docspath-springdocswagger-uipath-сортировкафильтры)
  * [Сканирование пакетов и исключения: `packagesToScan`, `pathsToMatch/Exclude`](#сканирование-пакетов-и-исключения-packagestoscan-pathstomatchexclude)
  * [Группы спецификаций: `GroupedOpenApi` для модулей/версий/доменных областей](#группы-спецификаций-groupedopenapi-для-модулейверсийдоменных-областей)
  * [Интеграция с Actuator и включение зависимостей (`/v3/api-docs`)](#интеграция-с-actuator-и-включение-зависимостей-v3api-docs)
  * [Кастомизация конвертеров типов (e.g., `Pageable`, `OffsetDateTime`, `UUID`)](#кастомизация-конвертеров-типов-eg-pageable-offsetdatetime-uuid)
* [4. Документирование контроллеров и операций](#4-документирование-контроллеров-и-операций)
  * [`@Operation`, `summary/description`, `tags`, `deprecated`, `hidden`](#operation-summarydescription-tags-deprecated-hidden)
  * [Параметры: `@Parameter` (path/query/header/cookie), обязательность, формат, примеры](#параметры-parameter-pathqueryheadercookie-обязательность-формат-примеры)
  * [Тело запроса: `@RequestBody`, медиа-типы, required, `@Content`/`@Schema`](#тело-запроса-requestbody-медиа-типы-required-contentschema)
  * [Ответы: `@ApiResponse`/`@ApiResponses`, коды, описания, ссылки на модели](#ответы-apiresponseapiresponses-коды-описания-ссылки-на-модели)
  * [Множественные контент-типы и `produces/consumes`, файловые загрузки/выгрузки](#множественные-контент-типы-и-producesconsumes-файловые-загрузкивыгрузки)
  * [Документация для WebFlux (reactive types) и асинхронных ответов](#документация-для-webflux-reactive-types-и-асинхронных-ответов)
* [5. Модели данных и валидация](#5-модели-данных-и-валидация)
  * [`@Schema`/`@ArraySchema`, описания полей, `format`, `nullable`, дефолты](#schemaarrayschema-описания-полей-format-nullable-дефолты)
  * [Enum-типы, oneOf/anyOf/allOf (наследование и композиция DTO)](#enum-типы-oneofanyofallof-наследование-и-композиция-dto)
  * [Bean Validation → OpenAPI: `@NotNull`, `@Size`, `@Pattern`, диапазоны чисел](#bean-validation--openapi-notnull-size-pattern-диапазоны-чисел)
  * [Примеры: `example`, `examples`, `@ExampleObject` для сложных payload’ов](#примеры-example-examples-exampleobject-для-сложных-payloadов)
  * [Маскирование/скрытие полей (`accessMode`, `writeOnly/readOnly`)](#маскированиескрытие-полей-accessmode-writeonlyreadonly)
  * [Референсы к шареным схемам (`components.schemas`), переиспользование](#референсы-к-шареным-схемам-componentsschemas-переиспользование)
* [6. Ошибки, стандартизация ответов и HATEOAS](#6-ошибки-стандартизация-ответов-и-hateoas)
  * [Единая модель ошибки (например, RFC7807/Problem Details) и глобальные мапперы](#единая-модель-ошибки-например-rfc7807problem-details-и-глобальные-мапперы)
  * [Глобальные ответы: добавление 4xx/5xx через конфигурацию/операционные фильтры](#глобальные-ответы-добавление-4xx5xx-через-конфигурациюоперационные-фильтры)
  * [Контент-негациация, локализация сообщений в описаниях](#контент-негациация-локализация-сообщений-в-описаниях)
  * [Линки и расширения: `links`, `x-*` для нестандартных метаданных](#линки-и-расширения-links-x--для-нестандартных-метаданных)
  * [Документация пагинации/сортировки (`Page`, `Link` заголовки)](#документация-пагинациисортировки-page-link-заголовки)
  * [Версионирование error-контракта и обратная совместимость](#версионирование-error-контракта-и-обратная-совместимость)
* [7. Безопасность и авторизация](#7-безопасность-и-авторизация)
  * [`@SecurityScheme` (HTTP bearer/JWT, API-ключ, OAuth2 flows, OpenID Connect)](#securityscheme-http-bearerjwt-api-ключ-oauth2-flows-openid-connect)
  * [`@SecurityRequirement` на уровне контроллера/операции](#securityrequirement-на-уровне-контроллераоперации)
  * [Описание scopes и ролей, связь с Spring Security](#описание-scopes-и-ролей-связь-с-spring-security)
  * [Документация процесса аутентификации в UI (Authorize, PKCE, device code)](#документация-процесса-аутентификации-в-ui-authorize-pkce-device-code)
  * [Работа с CSRF/сессионной аутентификацией в Swagger UI](#работа-с-csrfсессионной-аутентификацией-в-swagger-ui)
  * [Тестирование защищённых операций прямо из UI и в автотестах](#тестирование-защищённых-операций-прямо-из-ui-и-в-автотестах)
* [8. Версионирование API и мульти-спеки](#8-версионирование-api-и-мульти-спеки)
  * [Подходы к версиям: путь (`/v1`), заголовок/медиа-тип, поддомены](#подходы-к-версиям-путь-v1-заголовокмедиа-тип-поддомены)
  * [`GroupedOpenApi` для раздельных `v1/v2`, модульных bounded contexts](#groupedopenapi-для-раздельных-v1v2-модульных-bounded-contexts)
  * [Совместное существование старых и новых контрактов (sunset headers, deprecations)](#совместное-существование-старых-и-новых-контрактов-sunset-headers-deprecations)
  * [Документация лабораторных/внутренних эндпоинтов через теги/контексты](#документация-лабораторныхвнутренних-эндпоинтов-через-тегиконтексты)
  * [Стабильные URLs спецификаций для клиентов/порталов (per-version)](#стабильные-urls-спецификаций-для-клиентовпорталов-per-version)
  * [Сводные порталы: агрегирование нескольких сервисов/групп](#сводные-порталы-агрегирование-нескольких-сервисовгрупп)
* [9. Contract-first/Code-first, генерация и тестирование](#9-contract-firstcode-first-генерация-и-тестирование)
  * [Contract-first: цикл «редактируем OAS → генерим сервер/клиент → реализуем»](#contract-first-цикл-редактируем-oas--генерим-серверклиент--реализуем)
  * [Code-first: автогенерация OAS из аннотаций; риски расхождения кода и контракта](#code-first-автогенерация-oas-из-аннотаций-риски-расхождения-кода-и-контракта)
  * [Генерация клиентов (Java/Kotlin/TS): Feign/WebClient/Retrofit, типобезопасность](#генерация-клиентов-javakotlints-feignwebclientretrofit-типобезопасность)
  * [Контроль дрейфа: сравнение OAS (Spectral, oas-diff), контрактные тесты](#контроль-дрейфа-сравнение-oas-spectral-oas-diff-контрактные-тесты)
  * [CI: публикуем OAS как артефакт, снапшоты, проверка линтерами](#ci-публикуем-oas-как-артефакт-снапшоты-проверка-линтерами)
  * [Интеграция со Spring REST Docs (restdocs-openapi) для гибридного подхода](#интеграция-со-spring-rest-docs-restdocs-openapi-для-гибридного-подхода)
* [10. Публикация, эксплуатация и UX документации](#10-публикация-эксплуатация-и-ux-документации)
  * [Доступ к UI в проде: защита (Basic/OIDC), IP-allowlist, выключение в проде](#доступ-к-ui-в-проде-защита-basicoidc-ip-allowlist-выключение-в-проде)
  * [Экспорт статической документации (`/v3/api-docs` → HTML/Redoc) и хостинг](#экспорт-статической-документации-v3api-docs--htmlredoc-и-хостинг)
  * [Кастомизация Swagger UI: брендирование, фильтры, сортировка, скрытие служебных тегов](#кастомизация-swagger-ui-брендирование-фильтры-сортировка-скрытие-служебных-тегов)
  * [Порталы разработчика: changelog, примеры Postman/HTTPie, SDK-линки, rate limits](#порталы-разработчика-changelog-примеры-postmanhttpie-sdk-линки-rate-limits)
  * [Управление изменениями: deprecation policy, совместимость, коммуникация с потребителями](#управление-изменениями-deprecation-policy-совместимость-коммуникация-с-потребителями)
  * [Наблюдаемость: метрики наличия/размера OAS, алерты на разрыв контракта](#наблюдаемость-метрики-наличияразмера-oas-алерты-на-разрыв-контракта)
<!-- TOC -->

# 0. Введение

## Что это такое

OpenAPI — это формальный, машинно-читаемый способ описать REST-API: какие есть пути (`paths`), какие у них операции (`operations`), какие принимаются параметры и тела запросов, какие возвращаются ответы и ошибки, какие схемы данных используются и какая требуется авторизация. Это спецификация (документ в YAML/JSON), а не библиотека. На её основе инструменты умеют генерировать клиентские SDK, серверные заглушки, валидаторы, статические сайты и интерактивные UI.

Swagger — историческое имя экосистемы вокруг OpenAPI. Сегодня под «Swagger» чаще всего подразумевают Swagger UI — интерактивную веб-страницу, на которой можно читать документацию к API и сразу дергать методы (с подстановкой токена в `Authorize`). Также под «Swagger» иногда подразумевают аннотации (`io.swagger.v3.oas.annotations.*`), которые помогают сгенерировать спецификацию из кода. Важно различать: **OpenAPI — это сам контракт**, Swagger UI — лишь один из способов его показать.

В Spring Boot миром OpenAPI де-факто стандарт де-вью — библиотека `springdoc-openapi`. Она сканирует ваши контроллеры, модели, аннотации и на лету публикует JSON/YAML спецификацию на эндпоинте вроде `/v3/api-docs`, а также поднимает Swagger UI (обычно `/swagger-ui.html`). То есть вам не нужно «писать» JSON руками, если вы идёте путём code-first: сам код контроллера будет источником правды.

Спецификация OpenAPI описывает не только структуру запросов/ответов, но и безопасность: схемы авторизации (HTTP bearer/JWT, API-ключи, OAuth2, OpenID Connect), требуемые скоупы, заголовки, куки. Это позволяет одному и тому же документу служить и как пользовательская документация, и как контракт для автоматической генерации клиентов.

Ещё одна важная идея — **компоненты** (`components`): вы описываете схемы (`schemas`) и переиспользуете их в разных операциях через `$ref`. Это повышает однообразие и упрощает эволюцию: поправили модель — изменения автоматически отражаются в десятках методов.

OpenAPI — независим от технологии. Описанный контракт одинаково понятен для Java, Kotlin, JavaScript, Go, Python и т. д. Это снимает барьер между командами на разных стеках и упрощает доработки интеграций: контрагентам достаточно документа в одном формате.

Для Spring Boot OpenAPI полезен ещё и как «живой тест»: если спецификация публикуется вместе с приложением, она всегда отражает текущую версию кода. Любые расхождения между ожиданиями клиента и реальностью появляются сразу, на этапе автотестов/ревью.

При этом OpenAPI не навязывает «философию» API-дизайна: вы сами решаете, как версионировать, как именовать ресурсы и какие коды ответов использовать. Спецификация лишь фиксирует принятые вами решения.

У OpenAPI есть жизненный цикл. На старте проекта его используют для проектирования (через редакторы, дизайн-ревью и мок-серверы), в разработке — для генерации DTO/клиентов и автотестов, в эксплуатации — для публикации документации, управления изменениями и обратной совместимостью.

И наконец, OpenAPI — это мост между Dev и Ops: на его основе можно строить порталы разработчика (developer portal), выгружать статические HTML-страницы, выпускать SDK и Postman-коллекции, добавлять changelog’и и ограничивать доступ к UI в проде.

**Gradle (Groovy DSL): базовые зависимости для springdoc-openapi (MVC)**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin DSL): то же самое**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java: минимальный контроллер + аннотации OpenAPI**

```java
package demo.api;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@Tag(name = "Health", description = "Простые проверки живости")
public class HealthController {

    @GetMapping("/api/health")
    @Operation(summary = "Проверка живости", description = "Возвращает 'ok' если сервис жив")
    public String health() {
        return "ok";
    }
}
```

**Kotlin: то же самое**

```kotlin
package demo.api

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.tags.Tag
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@Tag(name = "Health", description = "Простые проверки живости")
class HealthController {

    @GetMapping("/api/health")
    @Operation(summary = "Проверка живости", description = "Возвращает 'ok' если сервис жив")
    fun health(): String = "ok"
}
```

**application.yml: пути к спецификации и UI**

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

---

## Зачем это нужно

OpenAPI решает проблему «устной договорённости» между сервисом и клиентом. Контракт фиксирует факты: какие параметры обязательны, какие форматы дат/ID используются, какие статусы и ошибки возможны. Это резко снижает стоимость интеграций: вместо «попробуй так, не работает — попробуй иначе» у команды есть однозначный источник правды.

Контракт позволяет **генерировать клиентов**. В проекте с несколькими серверами на разных языках не хочется писать HTTP-клиенты руками и ловить расхождения в DTO/валидации. Инструменты (OpenAPI Generator/Swagger Codegen) создают типобезопасный клиент, где параметры и модели строго соответствуют контракту. Это уменьшает баги и ускоряет разработку потребителей API.

OpenAPI помогает автоматизировать **тесты**. Из спецификации можно сгенерировать каркас контрактных тестов, мок-сервер, негативные сценарии (например, отсутствует обязательный параметр). Это превращает документацию в тестируемый артефакт, а не статическую страницу.

Ещё один важный случай — **обратная совместимость**. Сравнивая две версии спецификации (например, через `oas-diff`), легко увидеть «ломающие» изменения: удалённые поля, ужесточённые ограничения, изменённые коды ответов. Это позволяет внедрять deprecation-политику и планомерно переводить клиентов.

Для продуктовых команд OpenAPI — это **UX-артефакт**: Swagger UI/Redoc делают API «самообъясняющимся». Внутренние и внешние команды быстрее понимают возможности сервиса и пробуют запросы прямо в браузере, включая авторизацию (PKCE/OAuth2/OIDC).

В разработке OpenAPI — инструмент **снижения рисков**. Переезд команды, смена ключевых разработчиков или подрядчика не ломают коммуникацию — контракт остаётся. В распределённых организациях это особенно критично.

Для безопасности OpenAPI — способ формализовать **модель авторизации**: схемы, скоупы, обязательность токена. В UI можно пояснить, как получить токен, и сразу проверить защищённые методы, не мимоходя «обходя» защиту.

В DevOps OpenAPI — артефакт **публикации**: спецификацию можно выгружать статикой и хостить на отдельном домене, добавлять версионирование, changelog и метрики наличия. Это устраняет риск «подняли сервис — забыли документацию».

OpenAPI снижает **стоимость коммуникаций**. Вместо длинных писем и созвонов «объясните, что делает ваш метод» — одна ссылка на спецификацию и примеры запросов/ответов. Это особенно ценно при интеграциях с внешними командами и партнёрами.

Наконец, OpenAPI помогает **стандартизовать** ошибки и пагинацию: единая модель (например, RFC7807 Problem Details), единые заголовки и параметры. Это убирает «зоопарк» ответов, который неизбежно возникает в монорепозиториях/мульти-командных системах.

**Gradle (Groovy DSL): подключение OpenAPI Generator (генерация клиента)**

```groovy
plugins {
    id "org.openapi.generator" version "7.6.0"
}

openApiGenerate {
    generatorName = "java"
    inputSpec = "$rootDir/api/openapi.yaml"
    outputDir = "$buildDir/generated"
    library = "webclient"
    configOptions = [ dateLibrary: "java8" ]
}
```

**Gradle (Kotlin DSL): то же**

```kotlin
plugins {
    id("org.openapi.generator") version "7.6.0"
}

openApiGenerate {
    generatorName.set("java")
    inputSpec.set("$rootDir/api/openapi.yaml")
    outputDir.set("$buildDir/generated")
    library.set("webclient")
    configOptions.set(mapOf("dateLibrary" to "java8"))
}
```

**Java: использование сгенерированного клиента WebClient (пример)**

```java
package client.demo;

import org.openapitools.client.ApiClient;
import org.openapitools.client.api.HealthApi;
import org.springframework.web.reactive.function.client.WebClient;

public class UseGeneratedClient {
    public static void main(String[] args) {
        ApiClient api = new ApiClient(WebClient.builder()).setBasePath("http://localhost:8080");
        HealthApi healthApi = new HealthApi(api);
        String ok = healthApi.health(); // имя метода зависит от генерации
        System.out.println(ok);
    }
}
```

**Kotlin: аналогичный вызов**

```kotlin
package client.demo

import org.openapitools.client.ApiClient
import org.openapitools.client.api.HealthApi
import org.springframework.web.reactive.function.client.WebClient

fun main() {
    val api = ApiClient(WebClient.builder()).setBasePath("http://localhost:8080")
    val health = HealthApi(api)
    val ok = health.health()
    println(ok)
}
```

---

## Где это используется

Во внутренних микросервисах OpenAPI — основной способ согласовать контракты между командами. Когда десятки сервисов общаются REST’ом, без единой спецификации быстро начинается дрейф: у одного сервиса поле `id` — строка, у другого — число, у третьего — UUID. OpenAPI фиксирует типы, форматы и позволяет инструментам «подсветить» любые расхождения.

Во внешних интеграциях OpenAPI — визитная карточка. Партнёры оценивают зрелость вашего API по качеству документации, примерам, описанию авторизации, наличию SDK и Postman-коллекции. Хорошая спецификация сокращает время до первой интеграции.

В B2B-порталах OpenAPI — база для «порталов разработчика». На одной странице — интерактивная документация, лимиты, примеры, changelog, «как получить токен», «как воспроизвести ошибку». Командам поддержки становится проще — они ссылаются на единый источник.

В Data-платформах OpenAPI применяют для внутренних API управления данными (каталоги, словари, доступы). Это ускоряет онбординг и снижает количество вопросов. Да, это не «пользовательский продукт», но качество внутренней документации — про скорость всей организации.

В гибридных архитектурах (SPA + API) OpenAPI помогает фронт-командам: из спецификации можно сгенерировать TypeScript-клиент с типами, что заметно повышает надёжность и скорость изменения UI. Схемы DTO становятся общими артефактами для FE/BE.

В тестировании OpenAPI — источник для **контрактных тестов**. По спецификации можно автогенерировать проверки обязательных полей, негативные кейсы, smoke-наборы. Контрактные тесты между сервисами снижают риск «сломали соседа».

В платформенных командах OpenAPI служит **стандартом**: одинаковые паттерны ошибок, пагинации, версионирования. Появляются шаблоны сервисов, которые сразу содержат springdoc, Security схемы, примеры ошибок и пагинации.

В compliance/безопасности OpenAPI — «файл-паспорт» API, который можно анализировать статически: где требуются токены, какие есть публичные методы, какие PII-поля возвращаются. Это полезно для аудитов и регуляторики.

В эксплуатационных практиках OpenAPI — артефакт деплоя. Статическую документацию можно публиковать отдельно от бинарника, чтобы UI не открывался на продакшн-сервисе. Это повышает безопасность и снимает нагрузку.

Наконец, в обучении и найме OpenAPI — удобный материал для новичков: посмотрел спецификацию — понял контекст, эндпоинты, модели и ошибки. Это ускоряет онбординг и снижает стоимость менторинга.

**application.yml: включать UI в dev, отключать в prod**

```yaml
spring:
  profiles:
    group:
      dev: "swagger"
---
spring:
  config:
    activate:
      on-profile: swagger
springdoc:
  swagger-ui:
    enabled: true
---
spring:
  config:
    activate:
      on-profile: prod
springdoc:
  swagger-ui:
    enabled: false
```

**Java: конфиг для статуса UI (подтверждает включение/выключение)**

```java
package demo.config;

import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.OpenAPI;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiInfoConfig {
    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI().info(new Info().title("Demo API").version("v1"));
    }
}
```

**Kotlin: тот же конфиг**

```kotlin
package demo.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Info
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiInfoConfig {
    @Bean
    fun openAPI(): OpenAPI = OpenAPI().info(Info().title("Demo API").version("v1"))
}
```

---

## Какие задачи и проблемы решает

**Снижение интеграционных рисков.** Без спецификации API становится «устным протоколом», где «вчера было так, сегодня иначе». OpenAPI делает контракт явным и проверяемым. На ревью можно сравнить прошлую и новую версию, увидеть ломающее изменение и ввести процесс deprecation.

**Единообразие данных.** Команды часто «изобретают свои DTO» и ошибки. OpenAPI с общими `components.schemas` устраняет дублирование, вырабатывает корпоративные типы (например, `Money`, `Uuid`, `Page`) и повышает повторное использование.

**Генерация клиентов/заглушек.** Клиенты на Java/Kotlin/TS/Go получают одинаковые типы и сигнатуры. Это экономит недели разработки, особенно при большом числе потребителей. Заглушки ускоряют интеграции — можно тестировать до готовности реального сервера.

**Тестируемость контракта.** Автотесты могут валидировать ответы сервиса против OpenAPI: если код возвращает поле вне схемы — тест падает. Это ловит дрейф между командами и предотвращает «молчаливые» изменения.

**Улучшение DX (Developer Experience).** Swagger UI/Redoc позволяют быстро исследовать API, авторизоваться и попробовать запросы. Для поддержки это меньше тикетов «а как вызвать метод?», для разработчиков — меньше времени на чтение исходников.

**Безопасность и комплаенс.** Спецификация явно описывает авторизацию и чувствительные поля. Это облегчает аудит и автоматические проверки (линтеры, «сторожи»), а также обучение новых разработчиков безопасной работе с API.

**Версионирование и обратная совместимость.** Контракт облегчает ведение нескольких версий (`/v1`, `/v2`) и поэтапный вывод старых. Можно документировать sunset-заголовки, помечать `deprecated` и публиковать план миграции для клиентов.

**Порталы и внешние интеграции.** Из OpenAPI можно собрать статический сайт, SDK, примеры для Postman/HTTPie, добавить rate limits и SLA. Это повышает конверсию внешних интеграций и упрощает сопровождение.

**Наблюдаемость документации.** Спецификация — тоже артефакт. Её размер, валидность, наличие на проде можно мониторить и алертить. Это предотвращает «случайно сломали /v3/api-docs».

**Снижение времени онбординга.** Новому разработчику не нужно «копать» проект, чтобы понять API. Достаточно открыть UI или YAML — и картина ясна. Это особенно ценно в больших организациях с десятками сервисов.

**Gradle (Groovy DSL): зависимость для Problem Details и springdoc**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-validation"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin DSL): аналог**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java: единая ошибка RFC7807 + аннотации ответов**

```java
package demo.error;

import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/items")
public class ItemController {

    @GetMapping("/{id}")
    @ApiResponse(responseCode = "200", description = "OK")
    @ApiResponse(responseCode = "404", description = "Not Found",
        content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
    public String get(@PathVariable Long id) {
        throw new ItemNotFound(id);
    }

    @ResponseStatus(HttpStatus.NOT_FOUND)
    static class ItemNotFound extends RuntimeException {
        public ItemNotFound(Long id) { super("Item "+id+" not found"); }
    }

    @ExceptionHandler(ItemNotFound.class)
    public ProblemDetail handle(ItemNotFound ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setDetail(ex.getMessage());
        return pd;
    }
}
```

**Kotlin: тот же подход**

```kotlin
package demo.error

import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/items")
class ItemController {

    @GetMapping("/{id}")
    @ApiResponse(responseCode = "200", description = "OK")
    @ApiResponse(
        responseCode = "404",
        description = "Not Found",
        content = [Content(schema = Schema(implementation = ProblemDetail::class))]
    )
    fun get(@PathVariable id: Long): String {
        throw ItemNotFound(id)
    }

    @ResponseStatus(HttpStatus.NOT_FOUND)
    class ItemNotFound(id: Long) : RuntimeException("Item $id not found")

    @ExceptionHandler(ItemNotFound::class)
    fun handle(ex: ItemNotFound): ProblemDetail =
        ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.message ?: "not found")
}
```

# 1. Роль OpenAPI/Swagger в Spring Boot

## Что такое OpenAPI-спецификация, чем она отличается от Swagger UI/аннотаций

OpenAPI — это формальный контракт REST-API в формате YAML/JSON. В нём описываются пути, операции, параметры, тела, ответы, коды ошибок, безопасность и общие компоненты. Контракт независим от конкретного фреймворка и языка — его понимают генераторы, валидаторы и UI.

Swagger — историческое название экосистемы вокруг OpenAPI. В повседневной речи под «Swagger» часто подразумевают Swagger UI — интерактивную HTML-страницу, которая рендерит OpenAPI и позволяет вызывать методы «вживую». Это средство визуализации, а не сам контракт.

Аннотации `io.swagger.v3.oas.annotations.*` — это вспомогательный способ подсказать генератору (springdoc) описание операций, параметров и схем прямо в коде. Они не заменяют саму спеку, а помогают её собрать «из кода» в режиме code-first.

В Spring Boot популярна библиотека `springdoc-openapi`, которая сканирует контроллеры, модели и аннотации, формирует `/v3/api-docs` и публикует Swagger UI (обычно `/swagger-ui.html`). Это runtime-генерация спек из кода.

OpenAPI можно хранить как отдельный файл (`openapi.yaml`) и проектировать API до кода (contract-first). Swagger UI может рендерить и «живой» эндпоинт `/v3/api-docs`, и статическую YAML-спеку — это гибко.

Важно разделять роли: **OpenAPI** — источник правды, **Swagger UI** — способ чтения/пробы, **аннотации** — подсказки генератору. Путаница приводит к дрейфу: разработчики правят аннотации, а внешние клиенты ориентируются на устаревшую YAML-версию.

Аннотации удобны для локальных нюансов (пример, формат, описание), но сложные сценарии (наследование моделей, комбинированные oneOf/anyOf, сложная безопасность) легче и прозрачнее выразить в YAML.

Даже в code-first подходе полезно периодически выгружать и сдавать в ревью сгенерированную YAML-спеку. Это делает изменения явными и уменьшает зависимость от IDE/рендеринга UI.

Swagger UI — dev-инструмент. В проде UI либо выключают, либо защищают (OIDC/Basic/IP-allowlist). Но `/v3/api-docs` как машинный эндпоинт часто публикуют для порталов и генераторов.

Аннотации — часть кода, они версионируются вместе с сервисом. YAML-спека — артефакт, который можно публиковать независимо, версионировать, хранить в артефакт-репозитории и использовать генераторами клиентов.

**Gradle (Groovy) — зависимости для springdoc**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin) — то же**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — минимальный контроллер + аннотации (рендерится в `/v3/api-docs`)**

```java
package demo.role;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@Tag(name = "Health", description = "Проверки живости")
public class HealthController {
    @GetMapping("/api/health")
    @Operation(summary = "Проверка живости", description = "Возвращает 'ok' при доступности сервиса")
    public String health() { return "ok"; }
}
```

**Kotlin — эквивалент**

```kotlin
package demo.role

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.tags.Tag
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@Tag(name = "Health", description = "Проверки живости")
class HealthController {
    @GetMapping("/api/health")
    @Operation(summary = "Проверка живости", description = "Возвращает 'ok' при доступности сервиса")
    fun health(): String = "ok"
}
```

**application.yml — пути UI и JSON**

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

---

## Зачем нужна: контракт между сервисом и клиентом, генерация клиентов/серверов, документация

Один из главных рисков интеграций — «устные договорённости». OpenAPI делает контракт явным: типы, форматы, коды ответов, ошибки — всё фиксируется и проверяется.

Контракт облегчает общение между командами и контрагентами: вместо скриншотов и переписки — один YAML/JSON. Это снижает время до первой интеграции и количество недопониманий.

Генераторы (OpenAPI Generator, Swagger Codegen) превращают спецификацию в типобезопасных клиентов (Java/Kotlin/TS/Go и др.). Это убирает ручной бойлерплейт HTTP-вызовов и синхронизирует модели.

Серверные заглушки ускоряют старт проектирования: по контракту можно поднять мок-сервер и писать фронт/интеграции параллельно с бэкендом. Это снижает «узкие горлышки» по зависимости от соседней команды.

Swagger UI/Redoc дают интерактивную документацию: описание, параметры, примеры, кнопка «Authorize». Это улучшает DX и разгружает поддержку от типовых вопросов.

Контракт — это тестируемый артефакт. По нему можно запускать контрактные тесты и валидаторы: расхождение между кодом и контрактом будет замечено CI, а не пользователем.

В больших компаниях OpenAPI — источник для порталов разработчика: статические сайты, SDK, Postman-коллекции, changelog и rate limits. Это повышает прозрачность API.

Наличие единого контракта позволяет проводить дифф-анализ между версиями (ломающие/неломающие изменения). Это основа продуманной deprecation-политики и обратной совместимости.

Контракт полезен и для безопасности: описываются схемы авторизации, sсopes, требования к токену. UI даёт понятный поток получения токена и проверки вызова.

Документация «живая»: при подходе code-first springdoc генерирует `/v3/api-docs` прямо из кода, минимизируя дрейф. При contract-first YAML — источник, а код генерируется из него.

**Gradle (Groovy) — подключение OpenAPI Generator**

```groovy
plugins { id "org.openapi.generator" version "7.6.0" }

openApiGenerate {
    generatorName = "java"
    inputSpec = "$rootDir/api/openapi.yaml"
    outputDir = "$buildDir/generated"
    library = "webclient"
    configOptions = [ dateLibrary: "java8" ]
}
```

**Gradle (Kotlin) — то же**

```kotlin
plugins { id("org.openapi.generator") version "7.6.0" }

openApiGenerate {
    generatorName.set("java")
    inputSpec.set("$rootDir/api/openapi.yaml")
    outputDir.set("$buildDir/generated")
    library.set("webclient")
    configOptions.set(mapOf("dateLibrary" to "java8"))
}
```

**Java — вызов сгенерированного клиента (WebClient library)**

```java
package demo.clients;

import org.openapitools.client.ApiClient;
import org.openapitools.client.api.HealthApi;
import org.springframework.web.reactive.function.client.WebClient;

public class UseClient {
    public static void main(String[] args) {
        ApiClient api = new ApiClient(WebClient.builder()).setBasePath("http://localhost:8080");
        HealthApi health = new HealthApi(api);
        System.out.println(health.health()); // метод имени эндпоинта
    }
}
```

**Kotlin — эквивалент**

```kotlin
package demo.clients

import org.openapitools.client.ApiClient
import org.openapitools.client.api.HealthApi
import org.springframework.web.reactive.function.client.WebClient

fun main() {
    val api = ApiClient(WebClient.builder()).setBasePath("http://localhost:8080")
    val health = HealthApi(api)
    println(health.health())
}
```

---

## Место в SDLC: проектирование, разработка, тесты, релизы, сопровождение

На этапе проектирования OpenAPI служит макетом API. Команда согласует ресурсы, статусы и модели; правки в YAML видны всем и проходят через обычный код-ревью.

В разработке контракт — источник генерации клиентов/стабов и/или результат аннотированного кода (springdoc). В обоих случаях спек становится частью CI-потока.

В тестировании на основе контракта строятся мок-серверы, проверяются негативные сценарии и backward-совместимость через дифф-инструменты. Это уменьшает риск «сломали соседа».

На релизе спецификацию публикуют как артефакт, снапшотят, кладут в артефакт-репозиторий и/или выкладывают статический сайт (UI). Это обеспечивает доступ без запуска сервиса.

На сопровождении контракт используют для планирования изменений: помечают `deprecated`, объявляют sunset-даты, ведут changelog и информируют потребителей.

Для поддержки контракт — источник правды по ошибкам и кодам. Единый формат проблем (например, RFC7807) ускоряет диагностику и обработку тикетов.

В безопасности контракт отражает схемы авторизации; линтеры могут проверять, что на публичных маршрутах нет утечек чувствительных полей.

Для аналитики и порталов контракт агрегируют: сводят спецификации нескольких сервисов, обеспечивают стабильные пер-версийные URL и поиск.

В управлении качеством в CI добавляют шаги: линтер (Spectral), валидация (`openapi-cli validate`), дифф с предыдущей версией (oas-diff), публикация артефактов.

В онбординге новый разработчик по спецификации быстро понимает домены, модели, границы контекстов — это снижает входной барьер.

**Gradle (Groovy) — задача экспорта `/v3/api-docs` в файл**

```groovy
tasks.register("exportOpenApi", JavaExec) {
    classpath = sourceSets.main.runtimeClasspath
    mainClass = "demo.sdlc.ExportOpenApi"
    environment "SPEC_URL", "http://localhost:8080/v3/api-docs"
    environment "OUT", "$buildDir/openapi.json"
}
```

**Java — простой экспортёр спецификации (ApplicationRunner/HTTP)**

```java
package demo.sdlc;

import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.net.http.*;
import java.net.URI;
import java.nio.file.*;

@Configuration
public class ExportOpenApi {
    @Bean
    ApplicationRunner dump() {
        return args -> {
            var url = System.getenv().getOrDefault("SPEC_URL", "http://localhost:8080/v3/api-docs");
            var out = System.getenv().getOrDefault("OUT", "build/openapi.json");
            var body = HttpClient.newHttpClient()
                    .send(HttpRequest.newBuilder(URI.create(url)).GET().build(),
                          HttpResponse.BodyHandlers.ofString()).body();
            Files.createDirectories(Path.of("build"));
            Files.writeString(Path.of(out), body);
            System.out.println("OpenAPI exported to " + out);
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package demo.sdlc

import org.springframework.boot.ApplicationRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.net.URI
import java.net.http.HttpClient
import java.net.http.HttpRequest
import java.net.http.HttpResponse
import java.nio.file.Files
import java.nio.file.Path

@Configuration
class ExportOpenApi {
    @Bean
    fun dump(): ApplicationRunner = ApplicationRunner {
        val url = System.getenv("SPEC_URL") ?: "http://localhost:8080/v3/api-docs"
        val out = System.getenv("OUT") ?: "build/openapi.json"
        val body = HttpClient.newHttpClient().send(
            HttpRequest.newBuilder(URI.create(url)).GET().build(),
            HttpResponse.BodyHandlers.ofString()
        ).body()
        Files.createDirectories(Path.of("build"))
        Files.writeString(Path.of(out), body)
        println("OpenAPI exported to $out")
    }
}
```

---

## Подходы: code-first vs contract-first — плюсы и риски каждого

**Code-first**: быстрее стартовать — добавил контроллер, пометил аннотациями, получил `/v3/api-docs`. Меньше артефактов, выше «правдоподобность» — спецификация всегда отражает код.

Риск code-first — дрейф качества: аннотации неполные, описания и примеры отстают, сложные модели выражены «как получилось». Без линтинга и ревью это приводит к слабой документации.

**Contract-first**: дизайн до кода, обсуждение разрывов совместимости, генерация стабов/клиентов. Лучше для внешних API и межкомандных границ — договорились и зафиксировали прежде, чем писать код.

Риск contract-first — «теоретичность»: при слабой дисциплине YAML может уехать от реалий. Нужны автотесты/валидация, чтобы код соответствовал контракту.

Гибридный подход: грубую форму делают в YAML, дальше детали и описания дополняют аннотациями. Или наоборот — старт code-first, затем замораживают YAML как «источник правды».

Важно учитывать зрелость команды: если сильные практики ревью и линтинга — code-first работает отлично. Если много внешних клиентов — выгоднее contract-first, чтобы согласовать заранее.

С точки зрения tooling’а: для contract-first критичны редакторы/линтеры и OpenAPI Generator; для code-first — springdoc и единые гайды по аннотациям.

Про миграции версий: contract-first проще проводить дифф и формализовать deprecation. В code-first это тоже возможно, если выгружать YAML на каждом CI и сравнивать его.

Отлаженность DevEx: code-first проще для разработчиков, contract-first проще для потребителей. Выбор — про приоритеты продукта и зрелость окружения.

Компромисс: начинаем с code-first для скорости, на первом стабильном релизе «замораживаем» YAML и переходим к «контракт — главный».

**Gradle (Groovy) — генерация серверных стабов Spring из YAML (contract-first)**

```groovy
plugins { id "org.openapi.generator" version "7.6.0" }

openApiGenerate {
    generatorName = "spring"
    inputSpec = "$rootDir/api/openapi.yaml"
    outputDir = "$buildDir/gen-server"
    apiPackage = "demo.contract.api"
    modelPackage = "demo.contract.model"
    configOptions = [ interfaceOnly: "true", useSpringBoot3: "true" ]
}
```

**Gradle (Kotlin) — то же**

```kotlin
plugins { id("org.openapi.generator") version "7.6.0" }

openApiGenerate {
    generatorName.set("spring")
    inputSpec.set("$rootDir/api/openapi.yaml")
    outputDir.set("$buildDir/gen-server")
    apiPackage.set("demo.contract.api")
    modelPackage.set("demo.contract.model")
    configOptions.set(mapOf("interfaceOnly" to "true", "useSpringBoot3" to "true"))
}
```

**Java — реализация «сгенерированного» интерфейса (пример contract-first)**

```java
package demo.contract.impl;

import demo.contract.api.HealthApi;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthApiController implements HealthApi {
    @Override
    public ResponseEntity<String> health() {
        return ResponseEntity.ok("ok");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package demo.contract.impl

import demo.contract.api.HealthApi
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.RestController

@RestController
class HealthApiController : HealthApi {
    override fun health(): ResponseEntity<String> = ResponseEntity.ok("ok")
}
```

**Java — пример code-first контроллера с богатыми аннотациями**

```java
package demo.codefirst;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/items")
public class ItemController {
    @GetMapping("/{id}")
    @Operation(summary = "Получить предмет", description = "Возвращает предмет по ID")
    @ApiResponse(responseCode = "200", content = @Content(
            mediaType = "application/json",
            schema = @Schema(implementation = Item.class)))
    public Item get(@PathVariable long id) { return new Item(id, "name"); }

    public record Item(long id, String name) {}
}
```

**Kotlin — эквивалент**

```kotlin
package demo.codefirst

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/items")
class ItemController {
    @GetMapping("/{id}")
    @Operation(summary = "Получить предмет", description = "Возвращает предмет по ID")
    @ApiResponse(
        responseCode = "200",
        content = [Content(mediaType = "application/json", schema = Schema(implementation = Item::class))]
    )
    fun get(@PathVariable id: Long) = Item(id, "name")
    data class Item(val id: Long, val name: String)
}
```

---

## Артефакты: YAML/JSON спецификация, интерактивная UI, статически сгенерированные сайты

Первый артефакт — сама спецификация (`openapi.yaml`/`openapi.json`). Её удобно хранить в репозитории: версия рядом с кодом, ревью, линтеры, дифф-проверки.

Второй — интерактивная документация (Swagger UI/Redoc). Её можно поднимать вместе с приложением или публиковать как статический сайт без доступа к прод-базе.

Третий — сгенерированные SDK/клиенты. Это бинарные/исходные артефакты, которые поставляются потребителям и версионируются вместе с API.

Четвёртый — Postman/HTTPie коллекции, полученные из спецификации. Они помогают QA/интеграторам быстро начать тестирование.

Пятый — статический HTML-портал: брендирование, разделы «Авторизация», «Ограничения», «Changelog», «Поддержка». Основа — OAS + шаблон сайта.

Шестой — «снимки» спек по версиям: `https://docs.example.com/api/v1/openapi.yaml`. Стабильные ссылки для автоматической генерации клиентов в CI сторонних команд.

Седьмой — файлы сгенерированных страниц Redoc/RapiDoc, которые можно положить на CDN и обслуживать без сервиса. Это снижает риск экспонирования UI прод-сервиса наружу.

Восьмой — отчёты линтера/валидации (Spectral, `openapi-cli validate`). Они прикладываются к MR/релизу как доказательство качества.

Девятый — дифф-отчёты между версиями: список ломающих/совместимых изменений, служит основой коммуникаций и deprecation-плана.

Десятый — мини-«паспорт» API: метаданные (контакты, лицензия, terms), список серверов, политика версионирования — всё это часть `openapi.info` и сопровождающих файлов.

**Java — отдаём статический Redoc, связанный с `/v3/api-docs`**

```java
package demo.artifacts;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DocsController {
    @GetMapping("/docs")
    public String docs() {
        return "redirect:/redoc.html"; // redoc.html лежит в src/main/resources/static
    }
}
```

**Kotlin — эквивалент**

```kotlin
package demo.artifacts

import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.GetMapping

@Controller
class DocsController {
    @GetMapping("/docs")
    fun docs(): String = "redirect:/redoc.html"
}
```

**`src/main/resources/static/redoc.html` — минимальная страница**

```html
<!doctype html>
<html>
  <head><meta charset="utf-8"><title>API Docs</title></head>
  <body>
    <redoc spec-url="/v3/api-docs"></redoc>
    <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
  </body>
</html>
```

**Gradle (Groovy/Kotlin) — задачка на копию YAML в артефакты**

```groovy
tasks.register('copyOas', Copy) {
    from("$buildDir/openapi.json"); into("$buildDir/artifacts")
}
```

```kotlin
tasks.register<Copy>("copyOas") {
    from("$buildDir/openapi.json"); into("$buildDir/artifacts")
}
```

---

## Базовые элементы спецификации: paths, operations, components (schemas, responses, security)

`paths` — корневой раздел, где перечислены URL-пути. Каждый путь содержит набор HTTP-операций (`get`, `post`, …) с параметрами, телами и ответами.

`operationId` — уникальный идентификатор операции. Генераторы используют его как имя метода в клиентах. Стабильность `operationId` — залог стабильных SDK.

`parameters` — входы из пути, query, header, cookie. Они описываются типами/форматами/обязательностью и могут ссылаться на общие компоненты.

`requestBody` — структура и медиа-типы тела запроса. Можно сразу задавать примеры и ссылки на схемы.

`responses` — карта «код → описание → content → схема». Обычно фиксируют `200/201`, ошибки `400/401/403/404/409/422/500` и единый формат ошибки.

`components.schemas` — переиспользуемые DTO (record/POJO/дата-классы). Наследование и композицию выражают через `allOf/oneOf/anyOf`.

`components.responses` — заранее описанные ответы, чтобы переиспользовать шаблоны (например, «Страница результатов», «Проблема по RFC7807»).

`securitySchemes` — схемы безопасности (bearer/JWT, API-ключ, OAuth2, OpenID Connect). На уровне операции указывают требования (`security`).

`servers` — базовые URL окружений (dev/stage/prod) и переменные для подстановки. Это облегчает запуск UI и клиентов.

`tags`, `externalDocs` и `info` — организационные элементы навигации и метаданных, улучшают UX документации и порталов.

**Java — модель и контроллер с ответами/схемами**

```java
package demo.elements;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.constraints.NotBlank;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
@Tag(name = "Users", description = "Управление пользователями")
public class UserController {

    @GetMapping("/{id}")
    @Operation(operationId = "getUser", summary = "Получить пользователя")
    @ApiResponse(responseCode = "200", content = @Content(
            mediaType = "application/json",
            schema = @Schema(implementation = UserDto.class)))
    public UserDto get(@PathVariable String id) {
        return new UserDto(id, "Alice");
    }

    public record UserDto(
            @Schema(description = "Идентификатор", example = "u_123") String id,
            @NotBlank @Schema(description = "Имя") String name
    ) {}
}
```

**Kotlin — эквивалент**

```kotlin
package demo.elements

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.tags.Tag
import jakarta.validation.constraints.NotBlank
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
@Tag(name = "Users", description = "Управление пользователями")
class UserController {

    @GetMapping("/{id}")
    @Operation(operationId = "getUser", summary = "Получить пользователя")
    @ApiResponse(
        responseCode = "200",
        content = [Content(mediaType = "application/json", schema = Schema(implementation = UserDto::class))]
    )
    fun get(@PathVariable id: String) = UserDto(id, "Alice")

    data class UserDto(
        @Schema(description = "Идентификатор", example = "u_123") val id: String,
        @field:NotBlank @Schema(description = "Имя") val name: String
    )
}
```

**Фрагмент OpenAPI (YAML) — базовые элементы**

```yaml
openapi: 3.0.3
info: { title: Demo API, version: v1 }
servers: [ { url: https://api.example.com } ]
paths:
  /api/users/{id}:
    get:
      operationId: getUser
      tags: [ Users ]
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, example: u_123 }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
components:
  schemas:
    User:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security: [ { bearerAuth: [ ] } ]
```

# 2. Инструменты и установка

## springdoc-openapi для Spring Boot (MVC/WebFlux), Swagger UI как dev-вью

springdoc-openapi — де-факто стандарт интеграции OpenAPI в Spring Boot. Он считывает ваши контроллеры, модели, аннотации `io.swagger.v3.oas.*` и формирует эндпоинт спецификации (`/v3/api-docs`), а также поднимает Swagger UI как удобный dev-интерфейс. Для Spring MVC используется стартер `springdoc-openapi-starter-webmvc-ui`, для WebFlux — `springdoc-openapi-starter-webflux-ui`. Выбор стартера должен соответствовать выбранному стэку веб-слоя.

В MVC-приложении springdoc интегрируется прозрачно: достаточно добавить зависимость — и появится JSON со спецификацией. Swagger UI рендерит эту спецификацию и позволяет, не покидая браузер, делать запросы к локальному сервису, включая авторизацию (кнопка **Authorize**). Это особенно полезно на стадии разработки и ручной проверки.

В WebFlux-приложениях (reactive) springdoc точно так же публикует спецификацию, но корректно отображает реактивные типы (`Mono`, `Flux`) и их медиа-типы. Параметры и тела запросов описываются одинаково, но важно указывать корректные `produces/consumes`, чтобы UI и генераторы не «теряли» реактивную семантику.

Swagger UI — это dev-вью, а не публичный портал. В продакшене его обычно выключают или защищают. Но даже при выключенном UI машинный эндпоинт `/v3/api-docs` часто остаётся доступным для внутренних систем (портал разработчика, генераторы клиентов), однако с авторизацией и/или IP-фильтрацией.

springdoc уважает структуру вашего приложения: можно разбивать спецификацию на **группы** (например, «public», «admin», «internal») и публиковать отдельные JSON для каждой группы. Это упрощает контроль доступа и разграничение ответственности между командами.

Важная деталь — совместимость с Spring Boot 3.x: springdoc 2.x полностью ориентирован на `jakarta.*` и современные версии Spring. Для старых проектов на Spring Boot 2.x требуется линия 1.x стартера. Смешивать их нельзя.

В MVC-приложениях страница UI обычно доступна по `/swagger-ui.html`. Это не «жёсткий» путь — его можно переопределить и поместить UI под префикс `/docs/**`, чтобы избежать конфликтов маршрутов и соблюдать корпоративные политики URL.

UI поддерживает **параметризацию**: можно подставлять базовый URL сервера, выбирать схему авторизации, менять сортировку тегов. Это делается конфигурацией springdoc без кастомного кода.

Интеграция с Spring Security не требует особых плясок: UI и `/v3/api-docs` — такие же эндпоинты, как и ваши API. Дальше вы решаете, какие правила к ним применить. Главное — не забыть исключить `/v3/api-docs/**` из CSRF для GET-запросов к спецификации.

Если у вас WebFlux-стек, springdoc корректно работает вместе с функциональными роутерами (`RouterFunction`). Но для лучшей читаемости спецификации на функциональные обработчики стоит добавлять аннотации с описаниями, иначе документация получится «скудной».

Наконец, springdoc — это только «сборщик» и «рендерер». Он не навязывает философию дизайна API. Качество документации зависит от того, насколько дисциплинированно вы описываете операции, параметры и модели.

**Gradle (Groovy DSL): MVC и WebFlux стартеры**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-validation"

    // Для MVC
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"

    // Для WebFlux (если нужен реактивный стек)
    // implementation "org.springframework.boot:spring-boot-starter-webflux"
    // implementation "org.springdoc:springdoc-openapi-starter-webflux-ui:2.6.0"
}
```

**Gradle (Kotlin DSL): аналог**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-validation")

    // MVC
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")

    // WebFlux (реактивный стек)
    // implementation("org.springframework.boot:spring-boot-starter-webflux")
    // implementation("org.springdoc:springdoc-openapi-starter-webflux-ui:2.6.0")
}
```

**Java (MVC): минимальный контроллер с аннотацией**

```java
package tools.mvc;

import io.swagger.v3.oas.annotations.Operation;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
class PingController {
    @GetMapping("/api/ping")
    @Operation(summary = "Пинг", description = "Возвращает pong")
    public String ping() { return "pong"; }
}
```

**Kotlin (WebFlux): реактивный контроллер**

```kotlin
package tools.webflux

import io.swagger.v3.oas.annotations.Operation
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Mono

@RestController
class PingHandler {
    @GetMapping("/api/ping-reactive")
    @Operation(summary = "Пинг (reactive)", description = "Возвращает pong реактивно")
    fun ping(): Mono<String> = Mono.just("pong")
}
```

**application.yml: базовые пути UI и JSON**

```yaml
springdoc:
  api-docs.path: /v3/api-docs
  swagger-ui.path: /swagger-ui.html
```

---

## Подключение зависимостей (starter), минимальная конфигурация через `application.yml`

Минимально рабочая установка сводится к добавлению стартера springdoc и одного из веб-стартеров (MVC или WebFlux). Никаких дополнительных `@Bean` не требуется — автоконфигурация поднимет эндпоинт спецификации и UI.

Храните версии централизованно через `spring-boot-dependencies` BOM, чтобы исключить конфликт артефактов. Это важно для Jakarta и современных версий Spring: вы избегаете «зоопарка» несовместимых зависимостей.

В `application.yml` задайте пути к спецификации и UI. Это помогает избежать конфликтов с вашими маршрутами и позволяет спрятать документацию под «служебным» префиксом. В больших организациях это часто корпоративное требование.

Если у вас несколько модулей, подключающих springdoc, контролируйте **порядок сканирования** и `packagesToScan`, чтобы не получить лишние пути в итоговой спецификации. Это особенно актуально в монорепозиториях.

Для включения/отключения UI по профилям удобно использовать секции `---` с `spring.config.activate.on-profile`. В dev UI включён, в prod — выключен. Машинный эндпоинт `/v3/api-docs` можно оставить, но закрыть авторизацией.

Если вы публикуете статический Redoc/Swagger UI отдельно от приложения, всё равно полезно держать springdoc в рантайме — это «источник правды» для генерации YAML в CI. Эндпоинт можно закрыть фаерволом или доступом по VPN.

Ошибки часто возникают из-за неправильной кодировки YAML/JSON: убедитесь, что ваш реверс-прокси не модифицирует ответы `/v3/api-docs` (сжатие/чанкование ок) и не блокирует длинные ответы.

Если вы используете Spring Actuator, учтите, что springdoc не зависит от него, но может публиковать спецификацию и на management-порте. Это удобно, когда приложенческий порт закрыт.

Напоследок — избегайте «собственных» фильтров, которые перехватывают все пути и случайно ломают `/v3/api-docs`. Исключайте служебные пути явно.

**Gradle (Groovy DSL): базовый набор зависимостей**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin DSL): аналог**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**application.yml: минимальная конфигурация и профили**

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /docs

---
spring:
  config:
    activate:
      on-profile: prod
springdoc:
  swagger-ui:
    enabled: false
```

**Java: явный OpenAPI bean (метаданные)**

```java
package tools.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class OpenApiInfoConfig {
    @Bean
    OpenAPI openAPI() {
        return new OpenAPI().info(new Info()
            .title("Demo API").version("v1")
            .license(new License().name("Apache-2.0")));
    }
}
```

**Kotlin: тот же bean**

```kotlin
package tools.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiInfoConfig {
    @Bean
    fun openAPI(): OpenAPI = OpenAPI().info(
        Info().title("Demo API").version("v1").license(License().name("Apache-2.0"))
    )
}
```

---

## Совместимость: версии Spring Boot, Jakarta vs javax, PathPattern vs Ant

С Spring Boot 3.x вся экосистема перешла с `javax.*` на `jakarta.*`. Это значит, что ваши DTO должны импортировать `jakarta.validation.*`, а не `javax.validation.*`. Если смешать аннотации, сборка пройдёт, но рантайм начнёт «сыпаться» на несовместимых пакетах.

springdoc 2.x ориентирован на Spring Boot 3.x и `jakarta.*`. Для проектов на Spring Boot 2.7 следует использовать springdoc 1.7.x. Ошибка «класс не найден» при старте почти всегда признак неправильной ветки стартера.

MVC-маршрутизация в Spring Boot 3.x по умолчанию использует `PathPatternParser` вместо старых Ant-паттернов. springdoc корректно работает с обоими, однако если вы явно настраиваете стратегию `ANT_PATH_MATCHER`, убедитесь, что фильтры безопасности и springdoc смотрят на одинаковую стратегию, иначе правила могут разойтись.

В WebFlux `PathPattern` — единственный вариант, Ant-паттерны там не применяются. Это иногда влияет на ваши `pathsToMatch`/`pathsToExclude` в конфигурации springdoc — указывайте пути детальнее.

При миграции со Spring Boot 2.x проверьте импорты в аннотациях OpenAPI: пакет `io.swagger.v3.oas.annotations.*` остаётся прежним, но ваши модели/валидация должны быть на `jakarta.*`. Это самая частая точка спотыкания.

Если вы используете старые генераторы (Swagger Codegen 2.x), они могут не понимать новые форматы типов времени или особенности Spring 6/Boot 3. Лучше переходить на OpenAPI Generator 7.x.

Для Actuator в Boot 3.x часто используется отдельный порт. springdoc поддерживает публикацию спецификации и на management-порте; важно согласовать CORS и безопасность для него.

В приложениях с несколькими `DispatcherServlet` важно явно ограничить `packagesToScan`, чтобы springdoc не пытался анализировать «админский» или «внутренний» сервлет-контекст.

Если вы держите совместимость с клиентами на старых JVM, обратите внимание на «датовые» библиотеки: генераторы клиентов имеют опции `dateLibrary` (java8, threeTenBP). Неверный выбор приводит к несостыковкам типов в SDK.

И помните, что смешивать javax и jakarta в одном рантайме — плохая идея. Даже если оно «вроде бы работает», в какой-то момент получите несходимость зависимостей.

**Gradle (Groovy DSL): фиксируем ветки для Boot 3.x / jakarta**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-validation" // jakarta.validation
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin DSL): аналог**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java: DTO с jakarta-валидацией**

```java
package compat.dto;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;

public record UserDto(
        @Schema(description = "Имя пользователя") @NotBlank String name
) {}
```

**Kotlin: тот же DTO**

```kotlin
package compat.dto

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.NotBlank

data class UserDto(
    @field:NotBlank @Schema(description = "Имя пользователя")
    val name: String
)
```

**application.yml: стратегия матчей путей (если нужно ANT)**

```yaml
spring:
  mvc:
    pathmatch:
      matching-strategy: ant_path_matcher
```

---

## Альтернативы UI: ReDoc, RapiDoc; когда и зачем использовать

Swagger UI удобен в разработке, но не всегда подходит для публичной документации. Здесь уместны альтернативы — **ReDoc** и **RapiDoc**. Они рендерят ту же OpenAPI-спеку, но позволяют сильнее кастомизировать внешний вид, структуру и навигацию, вплоть до брендирования под вашу компанию.

ReDoc делает акцент на читаемости и структурировании: слева — оглавление, справа — подробная панель с примерами, описаниями, схемами. Он хорошо подходит для «порталов разработчика», где важна навигация и длинные описания.

RapiDoc — веб-компонент с богатыми опциями: тёмная тема, панель авторизации, фильтры тегов, возможность размещать документацию прямо в вашей CMS как `<rapi-doc>` тег. Он часто выигрывает у Swagger UI по UX для внешней аудитории.

Обе альтернативы можно хостить как статические страницы из вашего Spring Boot (в `src/main/resources/static`) или как отдельный статический сайт на CDN. Во втором случае приложение не экспонирует UI наружу, а отдаёт только `/v3/api-docs` для генерации.

С точки зрения безопасности ReDoc/RapiDoc — просто статические HTML. Всё управление доступом к самой спецификации решается на уровне того, кто может прочитать `/v3/api-docs`. UI можно сделать публичным, а JSON защищённым, если вы публикуете предварительно выгруженный статический YAML.

Поддерживайте стабильный URL спецификации. ReDoc/RapiDoc можно настраивать на `/v3/api-docs` текущего сервиса либо на статический `openapi.yaml`, выгруженный в CI. Второй вариант надёжнее для внешних пользователей.

Кастомизация тем/логотипов проще через ReDoc/RapiDoc, чем через Swagger UI. Если ваша команда дизайна предъявляет требования к брендингу портала, альтернативные UI обычно с ними справляются лучше.

Интеграция в Spring проста: положили HTML со скриптом в `static/`, добавили контроллер редиректа на `/docs` — и готово. Ни одного бина писать не нужно.

С ReDoc/RapiDoc легко делать «мульти-спеки»: на странице можно разместить табы со спекой v1 и v2, или разные группы (public/admin). Это помогает мягко вести версии.

Если API внешнее, лучше сразу думать о UX документации: качество портала — это время интеграции клиентов. Сами инструменты рендеринга — бесплатные, критично лишь ваше наполнение.

И наконец, выбор между Swagger UI/Redoc/RapiDoc — не взаимоисключающий. Swagger UI оставьте для девов, ReDoc — для внешней документации, RapiDoc — для кастомных страниц и PoC.

**Java: контроллер-редирект на статическую ReDoc-страницу**

```java
package ui.alt;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
class DocsController {
    @GetMapping("/docs")
    public String docs() { return "redirect:/redoc.html"; }
}
```

**Kotlin: то же**

```kotlin
package ui.alt

import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.GetMapping

@Controller
class DocsController {
    @GetMapping("/docs")
    fun docs(): String = "redirect:/redoc.html"
}
```

**`src/main/resources/static/redoc.html`: минимальная страница**

```html
<!doctype html>
<html>
  <head><meta charset="utf-8"><title>API Docs</title></head>
  <body>
    <redoc spec-url="/v3/api-docs"></redoc>
    <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
  </body>
</html>
```

**`src/main/resources/static/rapidoc.html`: RapiDoc пример**

```html
<!doctype html>
<html>
  <head><meta charset="utf-8"><title>API Docs</title></head>
  <body>
    <rapi-doc spec-url="/v3/api-docs" render-style="read" show-header="true"></rapi-doc>
    <script src="https://unpkg.com/rapidoc/dist/rapidoc-min.js"></script>
  </body>
</html>
```

---

## Генераторы: OpenAPI Generator / Swagger Codegen (клиенты/серверные заглушки)

OpenAPI Generator — активный проект-наследник Swagger Codegen. Он умеет генерировать клиенты и серверные заглушки для десятков языков, поддерживает последние версии OpenAPI и богатые опции конфигурации. Для Java/Kotlin чаще всего генерируют клиентов под WebClient/OkHttp/Retrofit.

Генерация клиента экономит недели рутины: типобезопасные модели, сериализация/десериализация, обработка ошибок, базовые ретраи. Особенно это важно, когда сервисов много, а команд — несколько: «ручные» клиенты быстро расходятся.

Генерация серверных интерфейсов (Spring) полезна в contract-first: вы получаете интерфейсы/DTO из YAML и реализуете только бизнес-логику. Это дисциплинирует контракт и уменьшает вероятность дрейфа.

Генераторы можно запускать из Gradle/Maven и в CI, выпуская SDK как артефакты релиза. Клиентские библиотеки версионируются вместе с API, потребители получают обновления по стандартным каналам.

Swagger Codegen всё ещё используется, но проект развивается медленнее. Если начинаете с нуля — берите OpenAPI Generator. Он поддерживает Spring Boot 3 / Jakarta и актуальные шаблоны.

Важно правильно настроить `dateLibrary`: `java8` (JSR-310) — наиболее удобный выбор. Неверный выбор приведёт к конфликтам типов дат/времени между сервисом и клиентом.

Будьте осторожны с «генерировать всё»: генераторы не знают вашей бизнес-логики. Потребуются ручные обёртки (например, для авторизации, ретраев, телеметрии). Но 80% бойлерплейта они закрывают.

Храните `openapi.yaml` в репозитории и валидируйте его линтерами. Генератор на невалидной спека сгенерирует невалидный SDK — баги будут дорогими.

Если клиенты на других языках (TS/Go), стройте мульти-задачи генерации в одном проекте. Это упростит жизнь потребителям и стандартизует практики.

Генерация серверных заглушек не отменяет springdoc. Даже если вы идёте contract-first, держите springdoc для публикации «живой» спеки — затем используйте её как «каноничную».

**Gradle (Groovy DSL): генерация клиента (WebClient)**

```groovy
plugins { id "org.openapi.generator" version "7.6.0" }

openApiGenerate {
    generatorName = "java"
    inputSpec = "$rootDir/api/openapi.yaml"
    outputDir = "$buildDir/generated"
    library = "webclient"
    packageName = "sdk.demo"
    configOptions = [ dateLibrary: "java8", hideGenerationTimestamp: "true" ]
}
```

**Gradle (Kotlin DSL): аналог**

```kotlin
plugins { id("org.openapi.generator") version "7.6.0" }

openApiGenerate {
    generatorName.set("java")
    inputSpec.set("$rootDir/api/openapi.yaml")
    outputDir.set("$buildDir/generated")
    library.set("webclient")
    packageName.set("sdk.demo")
    configOptions.set(mapOf("dateLibrary" to "java8", "hideGenerationTimestamp" to "true"))
}
```

**Java: использование сгенерированного клиента**

```java
package gen.client;

import org.openapitools.client.ApiClient;
import org.openapitools.client.api.PingApi;
import org.springframework.web.reactive.function.client.WebClient;

public class UseSdk {
    public static void main(String[] args) {
        ApiClient api = new ApiClient(WebClient.builder())
                .setBasePath("http://localhost:8080");
        System.out.println(new PingApi(api).ping());
    }
}
```

**Kotlin: то же**

```kotlin
package gen.client

import org.openapitools.client.ApiClient
import org.openapitools.client.api.PingApi
import org.springframework.web.reactive.function.client.WebClient

fun main() {
    val api = ApiClient(WebClient.builder()).setBasePath("http://localhost:8080")
    println(PingApi(api).ping())
}
```

**Генерация серверного интерфейса Spring (contract-first)**

```groovy
openApiGenerate {
    generatorName = "spring"
    inputSpec = "$rootDir/api/openapi.yaml"
    outputDir = "$buildDir/gen-server"
    apiPackage = "demo.api"
    modelPackage = "demo.model"
    configOptions = [ interfaceOnly: "true", useSpringBoot3: "true" ]
}
```

---

## Профили dev/test/prod для включения/ограничения UI и эндпоинтов спецификации

В dev UI нужен всегда: быстрый «живой» справочник и ручное тестирование. В test — по ситуации: чаще включён, чтобы QA/автотесты могли получать спецификацию. В prod — UI обычно выключают, а `/v3/api-docs` либо закрывают авторизацией, либо публикуют статический экспорт.

Профиль-управление проще делать в `application.yml` через секции `---` и `on-profile`. Это наглядно и не требует бинов. Для продакшна можно задать `springdoc.swagger-ui.enabled: false`.

Если спецификация нужна внешним системам, но UI нельзя, используйте статический экспорт в CI: скачайте `/v3/api-docs` с dev/stage, положите в CDN и отдавайте ReDoc с этим YAML. Так вы исключаете риск экспонирования прод-приложения.

Spring Security должен знать про эти правила: добавьте конфигурацию, где `/v3/api-docs/**` доступен только из внутренней сети/с определённой ролью. Swagger UI можно полностью отрезать правилами.

Для автотестов удобно иметь отдельный профиль `e2e`, где UI включён, а спецификация отдаётся без авторизации. Это ускорит прогон контрактных тестов и не будет мешать всем остальным стендам.

Если в кластере несколько сервисов, заведите консистентные пути: например, всегда `/docs` для UI и `/v3/api-docs` для JSON. Это уменьшит когнитивную нагрузку на команды.

Можно хранить «номер версии схемы» в баннере UI или в `info.version`, а приложение на старте сверять ожидаемую версию. Это добавляет контроля за публикациями.

Для внутреннего портала разработчика пригодится стабильный URL вида `/specs/<service>/<version>/openapi.yaml`. Генерируйте его в CI, а не в рантайме — так портал независим от аптайма сервисов.

Сторонние команды ценят предсказуемость: если UI выключен в prod, сообщите об этом в документации и дайте ссылку на статический портал. Это снизит поток «а где посмотреть Swagger?».

И наконец, любой доступ к документации — это тоже сервис. Мониторьте его доступность и алертите, как делаете это для основного приложения.

**application.yml: профили dev/test/prod**

```yaml
springdoc:
  api-docs.path: /v3/api-docs
  swagger-ui.path: /docs

---
spring:
  config:
    activate:
      on-profile: dev
springdoc:
  swagger-ui.enabled: true

---
spring:
  config:
    activate:
      on-profile: test
springdoc:
  swagger-ui.enabled: true

---
spring:
  config:
    activate:
      on-profile: prod
springdoc:
  swagger-ui.enabled: false
```

**Java: Security-конфигурация — закрыть `/v3/api-docs` в prod**

```java
package prof.sec;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
class SecurityConfig {
    @Bean
    SecurityFilterChain http(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.ignoringRequestMatchers("/v3/api-docs/**"))
           .authorizeHttpRequests(reg -> reg
               .requestMatchers("/docs", "/swagger-ui.html").permitAll() // в dev/test
               .requestMatchers("/v3/api-docs/**").hasRole("DOCS")
               .anyRequest().authenticated())
           .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

**Kotlin: тот же подход**

```kotlin
package prof.sec

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.Customizer
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecurityConfig {
    @Bean
    fun http(http: HttpSecurity): SecurityFilterChain {
        http.csrf { it.ignoringRequestMatchers("/v3/api-docs/**") }
            .authorizeHttpRequests {
                it.requestMatchers("/docs", "/swagger-ui.html").permitAll()
                 .requestMatchers("/v3/api-docs/**").hasRole("DOCS")
                 .anyRequest().authenticated()
            }
            .httpBasic(Customizer.withDefaults())
        return http.build()
    }
}
```

**Gradle (оба DSL): профиль-переменные**

```groovy
tasks.register("runDev") {
    doLast { println "Use: SPRING_PROFILES_ACTIVE=dev" }
}
```

```kotlin
tasks.register("runDev") {
    doLast { println("Use: SPRING_PROFILES_ACTIVE=dev") }
}
```

# 3. Базовая конфигурация springdoc

## `OpenAPI` bean: info/contact/license/terms, servers, externalDocs

Первое, что стоит настроить, — метаданные спецификации. Объект `OpenAPI` позволяет задать `info` (название, версию, описание), контактные данные, лицензию и условия использования. Эти поля потом подхватываются UI и статическими сайтами, помогая внешним и внутренним клиентам понять «кто владелец» и куда идти за помощью.

Поле `info.version` должно соответствовать версии контракта, а не номеру релиза приложения. Если вы версионируете API как `/v1`, логично ставить `v1` и в `info.version`, чтобы SDK и порталы можно было синхронизировать без «угадываний». Это также упрощает мониторинг: версии видны прямо в JSON/YAML.

Контакты (`contact`) — не формальность. Добавьте общий почтовый ящик команды и ссылку на борт/портал с инструкциями. Когда у партнёра что-то сломается, он увидит «куда писать» прямо в спецификации — это ускоряет поддержку и снижает нагрузку на случайные чаты.

Поле `license` полезно не только для публичных API. В больших организациях часто требуется указывать лицензию даже для внутренних сервисов — для единообразия и юридической прозрачности. Укажите хотя бы «Proprietary» или корпоративную формулировку.

`termsOfService` и `externalDocs` связывают спецификацию с внешними документами: SLA, политика deprecation, «как получить токен», примеры Postman/HTTPie. Ссылка в UI — это один клик для разработчика вместо поиска в wiki.

Секция `servers` описывает базовые адреса окружений: dev, stage, prod. Это делает UI и SDK «самообъясняемыми»: клиент сразу видит, куда слать запрос на стенде. Используйте переменные (`{env}`) и описания, чтобы не плодить дубли.

Если у вас мульти-регион, добавьте несколько серверов с понятными описаниями («eu-west prod», «us-east stage»). Это не только про удобство: генераторы клиентов умеют подставлять нужный сервер по умолчанию.

Следите, чтобы описания (`info.description`, `tag.description`) были полноценными, а не «заглушками». Два-три насыщенных абзаца экономят часы чужого времени. В Swagger UI длинные тексты отображаются аккуратно, это не «мешает».

Согласуйте владельца полей `info` в процессе ревью. Хорошая практика — правки метаданных проходят вместе с изменениями эндпоинтов: так никто не забудет обновить версию и контакты при крупном релизе.

Наконец, держите `OpenAPI` bean в отдельном конфиге и покрывайте smoke-тестом, который проверяет наличие ключевых полей. Бывает, что при рефакторинге этот бин «исчезает», а UI теряет заголовок и контакт.

**Java — `OpenAPI` bean с серверами и внешней документацией**

```java
package config.meta;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.*;
import io.swagger.v3.oas.models.servers.Server;
import io.swagger.v3.oas.models.ExternalDocumentation;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiMetaConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Orders API")
                .version("v1")
                .description("REST API для управления заказами")
                .contact(new Contact().name("API Team").email("api-team@example.com"))
                .license(new License().name("Proprietary").url("https://example.com/license"))
                .termsOfService("https://example.com/tos"))
            .addServersItem(new Server().url("https://dev.api.example.com").description("Dev"))
            .addServersItem(new Server().url("https://api.example.com").description("Prod"))
            .externalDocs(new ExternalDocumentation()
                .description("Портал разработчика")
                .url("https://docs.example.com/orders"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package config.meta

import io.swagger.v3.oas.models.ExternalDocumentation
import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Contact
import io.swagger.v3.oas.models.info.Info
import io.swagger.v3.oas.models.info.License
import io.swagger.v3.oas.models.servers.Server
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiMetaConfig {

    @Bean
    fun openAPI(): OpenAPI =
        OpenAPI()
            .info(
                Info()
                    .title("Orders API")
                    .version("v1")
                    .description("REST API для управления заказами")
                    .contact(Contact().name("API Team").email("api-team@example.com"))
                    .license(License().name("Proprietary").url("https://example.com/license"))
                    .termsOfService("https://example.com/tos")
            )
            .addServersItem(Server().url("https://dev.api.example.com").description("Dev"))
            .addServersItem(Server().url("https://api.example.com").description("Prod"))
            .externalDocs(ExternalDocumentation().description("Портал разработчика").url("https://docs.example.com/orders"))
}
```

**Gradle (Groovy/Kotlin) — зависимости (MVC)**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

---

## Настройка путей: `springdoc.api-docs.path`, `springdoc.swagger-ui.path`, сортировка/фильтры

Пути по умолчанию — `/v3/api-docs` для JSON и `/swagger-ui.html` для UI. В реальных системах лучше вынести UI под служебный префикс (`/docs`) и не пересекать его с пользовательскими маршрутами. Это снимает конфликты и делает правила безопасности проще.

Свойство `springdoc.api-docs.path` позволяет переместить эндпоинт спецификации. На продакшне его часто «живут» на management-порту или проксируются через API-шлюз, поэтому продумайте стабильный адрес для порталов и генераторов.

`springdoc.swagger-ui.path` меняет URL страницы Swagger UI. Согласуйте его со стандартами в компании — когда все сервисы используют одинаковый путь, проще обучать и сопровождать.

Swagger UI поддерживает сортировку тегов и операций: `springdoc.swagger-ui.tagsSorter=alpha` и `operationsSorter=method|alpha`. В больших API это резко повышает читабельность: разработчик быстро находит нужный раздел.

Фильтр в UI (`springdoc.swagger-ui.filter=true`) добавляет строку поиска по операциям. На богатых по числу методов API эта мелочь экономит минуты каждый день, особенно на стендах.

Если у вас несколько приложений за одним доменом, обратите внимание на кэширование UI CDN/прокси. Пути к статике Swagger не должны конфликтовать с кэшем других сервисов.

Для среды `prod` UI обычно отключают (`springdoc.swagger-ui.enabled=false`), а JSON закрывают авторизацией. Это не мешает CI выгружать спецификацию заранее и публиковать статический Redoc/Swagger на отдельном домене.

Настройка CORS: если UI открыт в другом домене, убедитесь, что `/v3/api-docs` и сами методы API отвечают с корректными заголовками CORS для GET. Иначе кнопка «Try it out» будет бесполезна.

В кластере полезно иметь префиксы `server.servlet.context-path` — не забывайте, что они влияют и на пути springdoc. Финальные URL должны быть задокументированы.

И, наконец, проверьте, что ваши фильтры безопасности/логгеры не перехватывают `/v3/api-docs` и не изменяют тело ответа. Любая мутация JSON ломает UI и генераторы.

**application.yml — пути и UI-настройки**

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /docs
    tags-sorter: alpha
    operations-sorter: method
    filter: true

---
spring:
  config:
    activate:
      on-profile: prod
springdoc:
  swagger-ui:
    enabled: false
```

**Java — небольшой smoke-тест доступности путей**

```java
package config.paths;

import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.net.http.*;
import java.net.URI;

@Configuration
class OpenApiSmoke {

    @Bean
    ApplicationRunner checkDocs() {
        return args -> {
            var url = "http://localhost:8080/v3/api-docs";
            var code = HttpClient.newHttpClient()
                .send(HttpRequest.newBuilder(URI.create(url)).GET().build(), HttpResponse.BodyHandlers.discarding())
                .statusCode();
            if (code != 200) throw new IllegalStateException("OpenAPI not available: " + code);
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package config.paths

import org.springframework.boot.ApplicationRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.net.URI
import java.net.http.HttpClient
import java.net.http.HttpRequest
import java.net.http.HttpResponse

@Configuration
class OpenApiSmoke {
    @Bean
    fun checkDocs(): ApplicationRunner = ApplicationRunner {
        val url = "http://localhost:8080/v3/api-docs"
        val code = HttpClient.newHttpClient().send(
            HttpRequest.newBuilder(URI.create(url)).GET().build(),
            HttpResponse.BodyHandlers.discarding()
        ).statusCode()
        require(code == 200) { "OpenAPI not available: $code" }
    }
}
```

---

## Сканирование пакетов и исключения: `packagesToScan`, `pathsToMatch/Exclude`

В монорепозиториях или мульти-модульных приложениях важно ограничить области, которые springdoc анализирует. Иначе в спецификацию попадут служебные контроллеры, админ-панели и внутренние API, что создаст шум и риски экспонирования.

Свойство `springdoc.packagesToScan` указывает, какие Java-пакеты сканировать. Старайтесь держать публичные контроллеры в отдельных пакетах (`com.example.api.public`) — это дисциплинирует архитектуру и упрощает конфигурацию.

`springdoc.pathsToMatch` задаёт белый список путей. Это удобно, когда API структурирован по префиксам (`/api/public/**`, `/api/admin/**`). Белые списки легче сопровождать, чем чёрные.

Обратная сторона — исключения (`springdoc.pathsToExclude`). Если вы не готовы к разделению пакетов, исключайте служебные пути явно (`/internal/**`, `/actuator/**`, `/error`, `/health`). Это временная мера, но работающая.

Помните, что фильтры применяются до построения схемы. Если исключить путь, связанная с ним модель не попадёт в `components.schemas`. Это может быть желательным эффектом для «внутренних DTO».

Для функциональных роутов WebFlux фильтры по путям работают так же, но указывайте шаблоны без антипаттернов Ant, если используете `PathPatternParser`. Проще — перечисление конкретных префиксов.

`packagesToScan` и `pathsToMatch` можно комбинировать. Сначала ограничьте пакеты, затем уточните пути. Так вы избежите сюрпризов, когда библиотека подтянет чужой контроллер с тем же URL-префиксом.

Не забудьте про тестовые контроллеры: в интеграционных тестах их иногда кладут в `test`-источники. Исключайте их из сборки финальной спецификации или выносите в тестовые профили.

Всё это — про чистоту контракта. Клиенты не должны видеть того, что им не положено, а ваш UI — оставаться компактным и релевантным.

Периодически проверяйте, что фильтры соответствуют реальности: добавьте автотест, который валидирует отсутствие внутренних путей в спецификации. Это дешёвый, но эффективный предохранитель.

**application.yml — ограничение сканирования и фильтры путей**

```yaml
springdoc:
  packages-to-scan: com.example.api.public
  paths-to-match: /api/public/**, /api/common/**
  paths-to-exclude: /internal/**, /admin/**, /error, /actuator/**, /health
```

**Java — точечная настройка через `GroupedOpenApi` (для конкретной группы)**

```java
package config.scan;

import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ScanConfig {
    @Bean
    GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public")
            .packagesToScan("com.example.api.public")
            .pathsToMatch("/api/public/**")
            .pathsToExclude("/internal/**")
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package config.scan

import org.springdoc.core.models.GroupedOpenApi
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class ScanConfig {
    @Bean
    fun publicApi(): GroupedOpenApi =
        GroupedOpenApi.builder()
            .group("public")
            .packagesToScan("com.example.api.public")
            .pathsToMatch("/api/public/**")
            .pathsToExclude("/internal/**")
            .build()
}
```

---

## Группы спецификаций: `GroupedOpenApi` для модулей/версий/доменных областей

Группы позволяют публиковать несколько спецификаций из одного приложения: по версиям (`v1`, `v2`), по доменам (`orders`, `users`), по уровням доступа (`public`, `admin`). Для каждого набора путей и пакетов вы создаёте бин `GroupedOpenApi`.

Отдельные JSON-эндпоинты для групп упрощают разграничение доступа. Например, `/v3/api-docs/public` можно отдавать наружу, а `/v3/api-docs/admin` — только внутри. Это снижает риск случайной экспозиции внутренних методов.

Группы — это и про UX. В Swagger UI можно переключаться между ними, получая компактную, тематическую документацию. В больших системах один «огромный» документ неудобен, а группировка решает проблему навигации.

Для версионирования удобно заводить группы `/v1` и `/v2`, указывая `pathsToMatch("/v1/**")` и `("/v2/**")`. Это позволяет вести старую и новую версии параллельно и давать клиентам стабильные ссылки на «свою» спеку.

Собираете монолит с несколькими bounded contexts? Сгруппируйте по доменам — ревью и сопровождение станут легче, а команда домена будет владеть «своей» частью контракта.

Не бойтесь перекрытий: один и тот же контроллер может попасть в две группы, если так нужно для порталов. Главное — следите за консистентностью tag’ов и описаний.

Группы хорошо сочетаются с профилями: в dev можно показывать «internal» группу, а в prod — скрывать её. Это двигается свойствами и не требует перекомпиляции.

У групп есть имя (`group`), которым формируется URL JSON: `/v3/api-docs/{group}`. Имя должно быть коротким и стабильным — по нему строятся линки в порталах и генераторы SDK.

Можно комбинировать `packagesToScan` и `pathsToMatch` в группе, чтобы жёстче ограничить область. Это особенно полезно, если библиотечные контроллеры попадают в общий classpath.

И помните: сами контроллеры от групп не зависят. Это «витрины» одного и того же кода, а не разные приложения. В этом их сила и простота.

**Java — три группы: public/admin/v1**

```java
package config.groups;

import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GroupsConfig {

    @Bean
    GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public")
            .pathsToMatch("/api/public/**")
            .build();
    }

    @Bean
    GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
            .group("admin")
            .pathsToMatch("/api/admin/**")
            .build();
    }

    @Bean
    GroupedOpenApi v1Api() {
        return GroupedOpenApi.builder()
            .group("v1")
            .pathsToMatch("/v1/**")
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package config.groups

import org.springdoc.core.models.GroupedOpenApi
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class GroupsConfig {
    @Bean
    fun publicApi(): GroupedOpenApi = GroupedOpenApi.builder().group("public").pathsToMatch("/api/public/**").build()

    @Bean
    fun adminApi(): GroupedOpenApi = GroupedOpenApi.builder().group("admin").pathsToMatch("/api/admin/**").build()

    @Bean
    fun v1Api(): GroupedOpenApi = GroupedOpenApi.builder().group("v1").pathsToMatch("/v1/**").build()
}
```

**application.yml — ссылки на JSON групп (для портала)**

```yaml
# Итоговые эндпоинты:
# /v3/api-docs/public
# /v3/api-docs/admin
# /v3/api-docs/v1
springdoc:
  swagger-ui:
    urls:
      - name: Public
        url: /v3/api-docs/public
      - name: Admin
        url: /v3/api-docs/admin
      - name: v1
        url: /v3/api-docs/v1
```

---

## Интеграция с Actuator и включение зависимостей (`/v3/api-docs`)

Spring Boot Actuator — стандартный способ давать операционные метрики, health и инфо. springdoc умеет дружить с Actuator: можно выставить спецификацию и UI на management-порту и/или описывать actuator-эндпоинты в отдельной группе.

Добавьте `spring-boot-starter-actuator` и, по желанию, стартер `springdoc-openapi-starter-actuator`, чтобы получить дополнительные интеграции (например, actuator-endpoint, который отдаёт OpenAPI). Это удобно для изоляции служебных эндпоинтов от пользовательского порта.

Часто management-порт отделяют: приложение слушает `:8080`, actuator — `:8081`. Тогда и `/v3/api-docs` можно держать на management-порту (через reverse-proxy), чтобы снизить поверхность атаки и логически отделить документацию.

Не забудьте `management.endpoints.web.exposure.include` — по умолчанию сервис отдаёт ограниченный набор эндпоинтов. Добавьте то, что нужно операторам (`health`, `info`, `metrics`), а спеку отдавайте как обычный контроллер springdoc.

Если UI должен быть доступен только операторам, добавьте правила безопасности на management-порт: Basic/OIDC, IP-allowlist. Это легко проверяется на ревью и не мешает разработчикам локально.

Иногда полезно разделить «боевой» и «инструментальный» трафик по TLS-сертификатам. Документация и actuator могут обслуживаться другим Ingress/сертификатом — это устранит лишние риски при обновлении сертификатов пользовательского домена.

Учтите, что Prometheus-скрейп и доступ к `/v3/api-docs` могут конфликтовать по лимитам. Не ставьте избыточные rate-limits на management-порт, если туда же прокинуты метрики.

При публикации статического портала springdoc всё равно полезен в рантайме: CI-задача может сходить на dev/stage management-порт и выгрузить «источник правды» для site-генерации.

Любые изменения Actuator не должны ломать основную спеку. Отдельная группа `GroupedOpenApi` под `/actuator/**` (с доступом только внутри) — чистый, предсказуемый подход.

И последнее: следите за версиями — Actuator и springdoc должны соответствовать вашей версии Spring Boot. Используйте BOM, чтобы избежать «дрейфа».

**Gradle (Groovy/Kotlin) — добавляем Actuator**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-actuator"
    // optional: интеграции с actuator
    implementation "org.springdoc:springdoc-openapi-starter-actuator:2.6.0"
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springdoc:springdoc-openapi-starter-actuator:2.6.0")
}
```

**application.yml — management-порт и экспозиция**

```yaml
management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
springdoc:
  api-docs:
    path: /v3/api-docs
```

**Java — отдельная группа для actuator**

```java
package config.act;

import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class ActuatorGroup {

    @Bean
    GroupedOpenApi actuatorApi() {
        return GroupedOpenApi.builder()
            .group("actuator")
            .pathsToMatch("/actuator/**")
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package config.act

import org.springdoc.core.models.GroupedOpenApi
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class ActuatorGroup {
    @Bean
    fun actuatorApi(): GroupedOpenApi =
        GroupedOpenApi.builder().group("actuator").pathsToMatch("/actuator/**").build()
}
```

---

## Кастомизация конвертеров типов (e.g., `Pageable`, `OffsetDateTime`, `UUID`)

Springdoc неплохо распознаёт типы «из коробки», но для продакшна почти всегда нужны доработки. Классика — `Pageable` из Spring Data: без подсказки он превращается в один объект, а нам нужны три query-параметра (`page`, `size`, `sort`). Решение — аннотация `@ParameterObject` на параметре контроллера.

Временные типы тоже требуют внимания. По умолчанию `OffsetDateTime` мапится в строку с `format: date-time`. Если в компании приняты строгие форматы (RFC 3339/UTC-ISO), закрепите это в `@Schema(format="date-time", example="2025-10-23T16:05:00Z")` и/или через глобальный кастомайзер.

UUID удобно помечать `format: uuid`, чтобы генераторы клиентов создавали правильные типы. Это опять же можно делать либо локально аннотациями, либо написать `PropertyCustomizer`, который добавит формат ко всем `UUID`-полям автоматически.

Иногда требуется переопределить сериализацию/названия параметров пагинации (`pageNumber`, `pageSize` вместо `page`, `size`). Тогда имеет смысл описать собственный `PageRequest`-DTO и использовать его в контракте, а в контроллере по нему строить `Pageable`.

Если вы активно используете коллекции и карты, настраивайте `@ArraySchema`/`additionalProperties` явно. Иначе генераторы клиентов могут решить, что поле «любой тип» и породить слабТипизированный код.

Глобальные кастомизации удобнее делать через `OpenApiCustomiser` — бин, который пост-обрабатывает уже собранную модель OpenAPI. Там можно пройтись по схемам и проставить форматы/примеры по корпоративным правилам.

Более низкоуровневый вариант — `ModelConverter` из `io.swagger.v3.core.converter`. Он позволяет влиять на построение схемы на этапе сканирования моделей. Это мощно, но сложнее в поддержке.

Не переусердствуйте: излишняя «магия» может скрыть реальные проблемы дизайна DTO. Если формат нестандартный — лучше описать его явно на поле через `@Schema`, чем пытаться угадать всё глобальным правилом.

И наконец, покрывайте кастомизации тестами: выгружайте YAML и проверяйте, что нужные поля получили `format`, `example` и т. п. Это предотвратит «тихий» дрейф при обновлениях springdoc.

**Java — `@ParameterObject` для `Pageable` + `UUID`/`OffsetDateTime` в DTO**

```java
package config.types;

import io.swagger.v3.oas.annotations.media.Schema;
import org.springdoc.core.annotations.ParameterObject;
import org.springframework.data.domain.Pageable;
import org.springframework.web.bind.annotation.*;

import java.time.OffsetDateTime;
import java.util.UUID;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    public PageResponse list(@ParameterObject Pageable pageable) {
        return new PageResponse();
    }

    public record OrderDto(
        @Schema(format = "uuid", description = "ID заказа") UUID id,
        @Schema(format = "date-time", example = "2025-10-23T10:00:00Z") OffsetDateTime createdAt
    ) {}

    public static class PageResponse { /* опущено */ }
}
```

**Kotlin — эквивалент**

```kotlin
package config.types

import io.swagger.v3.oas.annotations.media.Schema
import org.springdoc.core.annotations.ParameterObject
import org.springframework.data.domain.Pageable
import org.springframework.web.bind.annotation.*
import java.time.OffsetDateTime
import java.util.UUID

@RestController
@RequestMapping("/api/orders")
class OrderController {

    @GetMapping
    fun list(@ParameterObject pageable: Pageable): PageResponse = PageResponse()
}

data class OrderDto(
    @Schema(format = "uuid", description = "ID заказа") val id: UUID,
    @Schema(format = "date-time", example = "2025-10-23T10:00:00Z") val createdAt: OffsetDateTime
)

class PageResponse
```

**Java — глобальный кастомайзер форматов (`OpenApiCustomiser`)**

```java
package config.types;

import io.swagger.v3.oas.models.media.Schema;
import org.springdoc.core.customizers.OpenApiCustomiser;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class FormatsCustomizer {

    @Bean
    OpenApiCustomiser addFormats() {
        return openApi -> {
            if (openApi.getComponents() == null || openApi.getComponents().getSchemas() == null) return;
            openApi.getComponents().getSchemas().values().forEach(s -> applyFormatsRecursively(s));
        };
    }

    @SuppressWarnings("unchecked")
    private void applyFormatsRecursively(Schema<?> s) {
        if ("string".equals(s.getType()) && "UUID".equalsIgnoreCase(s.getTitle())) {
            s.setFormat("uuid");
        }
        if (s.getProperties() != null) {
            s.getProperties().values().forEach(p -> applyFormatsRecursively((Schema<?>) p));
        }
        if (s.getItems() != null) applyFormatsRecursively(s.getItems());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package config.types

import io.swagger.v3.oas.models.media.Schema
import org.springdoc.core.customizers.OpenApiCustomiser
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FormatsCustomizer {
    @Bean
    fun addFormats(): OpenApiCustomiser = OpenApiCustomiser { openApi ->
        val schemas = openApi.components?.schemas ?: return@OpenApiCustomiser
        schemas.values.forEach { applyFormatsRecursively(it) }
    }

    private fun applyFormatsRecursively(s: Schema<*>) {
        if (s.type == "string" && s.title.equals("UUID", ignoreCase = true)) s.format = "uuid"
        s.properties?.values?.forEach { applyFormatsRecursively(it as Schema<*>) }
        s.items?.let { applyFormatsRecursively(it) }
    }
}
```

**Gradle (оба DSL) — Spring Data для `Pageable`**

```groovy
dependencies { implementation "org.springframework.boot:spring-boot-starter-data-jpa" }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-data-jpa") }
```

# 4. Документирование контроллеров и операций

## `@Operation`, `summary/description`, `tags`, `deprecated`, `hidden`

Аннотация `@Operation` — основной инструмент, которым мы «приклеиваем» человекочитаемую документацию к методу контроллера. Она не меняет поведение кода, но сообщает springdoc, как назвать операцию, какое дать краткое описание и в какие разделы UI её поместить. По сути это заголовок карточки операции и метаданные, которые видит читатель в Swagger UI/Redoc.

`summary` и `description` выполняют разные роли. В `summary` даём короткое действие в повелительном наклонении («Получить заказ», «Создать платёж»), чтобы оно влезало в список методов. В `description` подробно раскрываем нюансы: бизнес-контекст, ограничения, правила валидации, ссылки на внешние регламенты или FAQ. Если описание длинное, это нормально: UI аккуратно сворачивает/разворачивает текст.

`tags` — способ тематически группировать операции. В больших API теги играют роль оглавления: «Orders», «Users», «Admin». Не перегружайте метод множеством тегов и используйте согласованные имена, совпадающие с доменными bounded contexts. Стабильные теги облегчают навигацию и упрощают выделение мульти-спек по группам.

`operationId` в `@Operation` часто оставляют по умолчанию, но лучше задать самостоятельно и стабильно. Это имя станет методом в сгенерированных клиентах; сменив его без необходимости, вы сломаете чужой код. Конвенция «глагол + сущность» (`getOrder`, `createOrder`) хорошо работает и читается во всех SDK.

Флаг `deprecated = true` помогает сообщить потребителям о планируемом выводе метода. В UI он подсветится как устаревший, а в генераторах клиентов появится отметка. Помечайте устаревшие операции и указывайте в `description` план миграции: до какой даты метод будет работать и на что переходить.

Аннотация `@Hidden` скрывает метод или целый контроллер из спецификации. Это полезно для внутренних служебных маршрутов, health-check'ов и тестовых «фикстур». Не злоупотребляйте скрытием: потребителю важнее видеть честную карту API, а скрытое — действительно должно быть недоступно внешне.

Стоит договориться о единообразной локализации. Чаще выбирают один язык для всех описаний (обычно английский), но в внутренних API для русскоязычных команд допустим русский. Микс языков в одном проекте затрудняет поиск и ухудшает UX. Если нужна мультиязычность, лучше хранить тексты в шаблонах и собирать разные сборки документации.

Добавляйте в `description` важные правовые и операционные детали: лимиты, квоты, окна обслуживания. Это снижает число вопросов к поддержке. Если детали часто меняются, укажите ссылку на внешний «живой» документ в Confluence/портале и обновляйте её там.

Не стесняйтесь ставить понятные бизнес-названия в `@Tag(name = "...", description = "...")`. Даже внутренним коллегам проще ориентироваться по доменным терминам, чем по техническим «util», «internal». Описание тега — место для краткого «о чём этот раздел».

Покрывайте наличие базы метаданных тестом. Простой интеграционный тест, который вытягивает `/v3/api-docs` и проверяет, что нужные теги и operationId присутствуют, спасёт от случайного удаления `@Operation` при рефакторинге. Документация — такой же контракт, как и интерфейс метода.

**Gradle (Groovy) — базовые зависимости для аннотирования**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin) — то же**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — пример `@Operation`, теги, deprecated и hidden**

```java
package docs.op;

import io.swagger.v3.oas.annotations.Hidden;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
@Tag(name = "Orders", description = "Операции с заказами")
public class OrderController {

    @GetMapping("/{id}")
    @Operation(
        operationId = "getOrder",
        summary = "Получить заказ",
        description = "Возвращает заказ по идентификатору. Используйте для чтения актуального состояния.",
        deprecated = false
    )
    public String get(@PathVariable String id) {
        return "order:" + id;
    }

    @DeleteMapping("/{id}")
    @Operation(
        operationId = "deleteOrder",
        summary = "Удалить заказ",
        description = "Метод устарел, используйте отмену заказа. Будет удалён 2026-01-01.",
        deprecated = true
    )
    public void delete(@PathVariable String id) { }

    @Hidden
    @GetMapping("/internal/ping")
    public String internalPing() { return "ok"; }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.op

import io.swagger.v3.oas.annotations.Hidden
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.tags.Tag
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
@Tag(name = "Orders", description = "Операции с заказами")
class OrderController {

    @GetMapping("/{id}")
    @Operation(
        operationId = "getOrder",
        summary = "Получить заказ",
        description = "Возвращает заказ по идентификатору. Используйте для чтения актуального состояния.",
        deprecated = false
    )
    fun get(@PathVariable id: String): String = "order:$id"

    @DeleteMapping("/{id}")
    @Operation(
        operationId = "deleteOrder",
        summary = "Удалить заказ",
        description = "Метод устарел, используйте отмену заказа. Будет удалён 2026-01-01.",
        deprecated = true
    )
    fun delete(@PathVariable id: String) { }

    @Hidden
    @GetMapping("/internal/ping")
    fun internalPing(): String = "ok"
}
```

---

## Параметры: `@Parameter` (path/query/header/cookie), обязательность, формат, примеры

Параметры пути — единственные по-настоящему обязательные параметры транспортного уровня: без них маршрут не будет сопоставлен. В OpenAPI их нужно отмечать как `required = true`, а в `schema` указывать тип и формат (`UUID`, `int64`). В `example` удобно дать правдоподобное значение, чтобы кнопка «Try it out» была полезной.

Query-параметры передают фильтры, сортировки и опциональные модификаторы запроса. Здесь важно явно моделировать обязательность: если параметр «по умолчанию» включён, лучше отразить это и в схеме (`default`), и в описании. Для коллекций используйте «множественные» параметры (`?sort=...&sort=...`) или массивы, и фиксируйте стиль кодирования.

Заголовки (`header`) применяйте для метаданных запроса: идемпотентные ключи, кореляционные ID, языковые предпочтения. В описании стоит указать ожидаемый формат (`UUID`, `ISO-639-1`). Если заголовок чувствителен (например, idempotency-key для платежей), отражайте это в `description` и примерах.

Куки (`cookie`) редки в чистых REST-интеграциях, но встречаются при сессионной аутентификации. В документации важно отметить, при каких сценариях кука появляется и как долго живёт. Если API предполагает только токены, лучше полностью отказаться от кук и не путать клиентов.

`@Parameter` поддерживает `schema`, `required`, `description`, `example` и `examples`. Последнее удобно для набора типовых значений. Внутри `schema` ставьте `format` и `pattern`, если у вас нестандартные ID (например, `ord_12345`).

Для сложных наборов query-параметров (пагинация, сортировки) в Spring практичнее использовать `@ParameterObject` с `Pageable`/DTO и добавить локальные аннотации `@Schema` на поля этого DTO. Так вы избежите ручного дублирования одинаковых описаний во всех методах.

Если параметр логически обязательный, но транспортно опциональный (например, `Accept-Language`), это стоит подчеркнуть в тексте. Генераторы не понимают бизнес-обязательность, они лишь видят флаг `required`.

Примеры — это UX. Потратьте минуту и подставьте реалистичные значения, а не `string`. Хороший пример экономит время у каждого, кто впервые пробует API в UI. Особенно это важно для дат и валют — формат нужно показывать явно.

Проверяйте соответствие описанных параметров и фактических. Любое расхождение (переименовали `from` в `start` в коде, но забыли в аннотации) вернётся багами у клиентов. Автотест на дифф спецификации — дешёвый предохранитель.

**Gradle (Groovy) — добавим валидацию для схем параметров**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-validation"
}
```

**Gradle (Kotlin) — то же**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — параметры всех видов с примерами и форматами**

```java
package docs.params;

import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.enums.ParameterIn;
import io.swagger.v3.oas.annotations.media.Schema;
import org.springdoc.core.annotations.ParameterObject;
import org.springframework.data.domain.Pageable;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/search")
public class SearchController {

    @GetMapping("/orders/{id}")
    public String byId(
        @Parameter(in = ParameterIn.PATH, required = true,
                   description = "UUID заказа", example = "550e8400-e29b-41d4-a716-446655440000",
                   schema = @Schema(format = "uuid"))
        @PathVariable UUID id,

        @Parameter(in = ParameterIn.QUERY, description = "Включать отменённые", example = "false")
        @RequestParam(required = false) Boolean includeCanceled,

        @Parameter(in = ParameterIn.HEADER, name = "X-Request-Id",
                   description = "Корреляционный идентификатор запроса", example = "req-12345")
        @RequestHeader(value = "X-Request-Id", required = false) String rid,

        @Parameter(in = ParameterIn.COOKIE, name = "session", description = "Сессионная кука", example = "abc123")
        @CookieValue(value = "session", required = false) String session,

        @ParameterObject Pageable pageable
    ) {
        return "ok";
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.params

import io.swagger.v3.oas.annotations.Parameter
import io.swagger.v3.oas.annotations.enums.ParameterIn
import io.swagger.v3.oas.annotations.media.Schema
import org.springdoc.core.annotations.ParameterObject
import org.springframework.data.domain.Pageable
import org.springframework.web.bind.annotation.*
import java.util.UUID

@RestController
@RequestMapping("/api/search")
class SearchController {

    @GetMapping("/orders/{id}")
    fun byId(
        @Parameter(
            `in` = ParameterIn.PATH, required = true,
            description = "UUID заказа",
            example = "550e8400-e29b-41d4-a716-446655440000",
            schema = Schema(format = "uuid")
        )
        @PathVariable id: UUID,
        @Parameter(in = ParameterIn.QUERY, description = "Включать отменённые", example = "false")
        @RequestParam(required = false) includeCanceled: Boolean?,
        @Parameter(name = "X-Request-Id", `in` = ParameterIn.HEADER, description = "Корреляционный ID", example = "req-12345")
        @RequestHeader(value = "X-Request-Id", required = false) rid: String?,
        @Parameter(name = "session", `in` = ParameterIn.COOKIE, description = "Сессионная кука", example = "abc123")
        @CookieValue(value = "session", required = false) session: String?,
        @ParameterObject pageable: Pageable
    ): String = "ok"
}
```

---

## Тело запроса: `@RequestBody`, медиа-типы, required, `@Content`/`@Schema`

Тело запроса в OpenAPI описывает «форму» данных, которые принимает метод. В `@RequestBody` мы указываем обязательность тела, поддерживаемые медиа-типы и ссылку на схему DTO. Это ключевая часть для методов `POST`/`PUT`/`PATCH`, где без тела метод не имеет смысла.

Медиа-типы важны для контент-негациации. Для JSON указываем `application/json`, для патч-операций может быть `application/merge-patch+json` или `application/json-patch+json`. Чем точнее вы укажете тип, тем меньше сюрпризов у клиентов и тем корректнее работают генераторы.

`required = true` в `@RequestBody` — не просто «сделать красиво». Это формализует контракт: сервер вправе вернуть 400, если тела нет. Если тело опционально (редкий случай), добавьте в описание, при каких условиях оно не нужно.

Схемы описываются через `@Schema` на DTO-классе. Там задайте обязательные поля, форматы (`date-time`, `uuid`, `email`), примеры и ограничения Bean Validation (`@NotNull`, `@Size`). Springdoc подтянет в схему и валидационные аннотации.

Для сложных моделей используйте композицию `oneOf/anyOf/allOf`. Это удобнее выразить в самом DTO через наследование / sealed и в явной OpenAPI-схеме через YAML. Если держите всё аннотациями, можно пометить поле `@Schema(oneOf = {A.class, B.class})`.

Размеры тел и вложенность нужно осмысленно ограничивать: в описании укажите ожидаемые пределы, а в коде проверьте их валидаторами. Это убережёт ваш API от «бомб» в виде многомегабайтных JSON.

При документировании `PATCH` не забывайте уточнять семантику null/отсутствия полей. В контракте можно показать два `example`: «поле удаляем» и «поле оставляем неизменным». Это спасает от неоднозначностей.

Если вы принимаете нестандартные форматы (CSV, XML), обязательно приложите пример и опишите, как обрабатываются экранирование и разделители. Для CSV удобно показать заголовок и одну-две строки.

Не бойтесь добавлять бизнес-правила напрямую в `description` схемы: «сумма > 0», «валюта — ISO-4217». Документация — не только типы, но и смыслы. Эти поля читают инженеры, а не только машины.

Проверяйте, что ваша сериализация (Jackson) соответствует контракту: имена полей, регистры, формат дат. Любая разница между DTO и схемой будет ударять по клиентам и тестам генераторов.

**Gradle (Groovy/Kotlin) — валидация и Jackson уже есть в web-стартере**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-validation"
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — `@RequestBody` с медиа-типами и схемой**

```java
package docs.body;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import jakarta.validation.constraints.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    @PostMapping(consumes = {"application/json", "application/merge-patch+json"})
    @Operation(summary = "Создать платёж", description = "Создаёт платёж. PATCH-подобный JSON допускается для частичных обновлений.")
    public String create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            required = true,
            content = {
                @Content(mediaType = "application/json",
                        schema = @Schema(implementation = CreatePayment.class),
                        examples = @ExampleObject(name = "basic", value = "{\"amount\":100,\"currency\":\"RUB\"}")),
                @Content(mediaType = "application/merge-patch+json",
                        schema = @Schema(implementation = CreatePayment.class))
            }
        )
        @RequestBody CreatePayment body) {
        return "ok";
    }

    public static class CreatePayment {
        @Positive @Schema(description = "Сумма в минорных единицах", example = "100")
        public Long amount;
        @Pattern(regexp = "^[A-Z]{3}$") @Schema(description = "Валюта ISO-4217", example = "RUB")
        public String currency;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.body

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.ExampleObject
import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.Pattern
import jakarta.validation.constraints.Positive
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/payments")
class PaymentController {

    @PostMapping(consumes = ["application/json", "application/merge-patch+json"])
    @Operation(summary = "Создать платёж", description = "Создаёт платёж. PATCH-подобный JSON допускается для частичных обновлений.")
    fun create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            required = true,
            content = [
                Content(
                    mediaType = "application/json",
                    schema = Schema(implementation = CreatePayment::class),
                    examples = [ExampleObject(name = "basic", value = """{"amount":100,"currency":"RUB"}""")]
                ),
                Content(
                    mediaType = "application/merge-patch+json",
                    schema = Schema(implementation = CreatePayment::class)
                )
            ]
        )
        @RequestBody body: CreatePayment
    ): String = "ok"
}

class CreatePayment {
    @field:Positive
    @Schema(description = "Сумма в минорных единицах", example = "100")
    var amount: Long? = null

    @field:Pattern(regexp = "^[A-Z]{3}$")
    @Schema(description = "Валюта ISO-4217", example = "RUB")
    var currency: String? = null
}
```

---

## Ответы: `@ApiResponse`/`@ApiResponses`, коды, описания, ссылки на модели

Хорошая спецификация ответов экономит часы потребителям. Для каждого метода описывайте не только «успех», но и ожидаемые ошибки: `400/401/403/404/409/422/429/500`. Минимум — коды и краткие описания; лучше — ещё и схемы тела ошибок.

`@ApiResponse` позволяет указать `responseCode`, `description` и `content`. В `content` привязываем `mediaType` и `schema` с DTO. Если вы переиспользуете стандартные ответы, вынесите их в `components.responses` и ссылайтесь из методов.

Модели ошибок стоит стандартизовать. В Spring Boot 3 можно использовать `ProblemDetail` (RFC7807) или собственный формат. В обоих случаях опишите поля (`type`, `title`, `detail`, `instance`, `traceId`) и приведите примеры. Потребители экономят время, когда видят единый шаблон.

В ответах важны заголовки. Для создания ресурса указывайте `201` и заголовок `Location` с URL нового объекта. Для кэшируемых GET — `ETag` и `Last-Modified`. В OpenAPI это можно задокументировать через `headers` в `@ApiResponse`.

Не ограничивайтесь JSON. Если метод может вернуть `204 No Content`, добавьте это явно. Отсутствие тела — часть контракта, и его надо зафиксировать. Swagger UI корректно покажет «без контента».

Стриминговые и бинарные ответы описывают как `application/octet-stream` или `text/event-stream`. Для JSON-стримов можно использовать `application/x-ndjson`. Это улучшает автогенерацию клиентов и помогает UI понять, чего ждать.

Ссылки (`links`) в OpenAPI связывают ответы между собой. Это полезно для REST-навигации: например, из `createOrder` дать ссылку `getOrder` с подстановкой `id`. В аннотациях такое выразить сложнее, но в YAML это делается легко.

Глобальные ответы можно добавлять через `OpenApiCustomiser`, чтобы не дублировать `401/403/429/500` в каждом методе. Это повышает единообразие и снижает шанс забыть важные коды.

Не забывайте документировать альтернативные коды успеха. Например, `202 Accepted` для асинхронной постановки задачи в очередь. В описании поясните, как клиент узнает статус (polling, callback, webhook).

Проверяйте соответствие статусов в коде и в аннотациях. Если метод фактически возвращает `201`, а в спецификации `200`, клиенты сгенерируют неправильные ожидания и тесты начнут падать. Автотесты по спецификации — лучшая страховка.

**Java — ответы с 200/201/404 и ProblemDetail**

```java
package docs.responses;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.*;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    @Operation(summary = "Создать пользователя")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Создано",
            headers = @Header(name = "Location", description = "URL нового ресурса", schema = @Schema(type = "string")),
            content = @Content(mediaType = "application/json", schema = @Schema(implementation = User.class))),
        @ApiResponse(responseCode = "400", description = "Неверные данные",
            content = @Content(mediaType = "application/problem+json", schema = @Schema(implementation = org.springframework.http.ProblemDetail.class)))
    })
    public ResponseEntity<User> create(@RequestBody User body) {
        return ResponseEntity.created(URI.create("/api/users/1")).body(new User(1L, body.name()));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Получить пользователя")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "OK",
            content = @Content(mediaType = "application/json", schema = @Schema(implementation = User.class))),
        @ApiResponse(responseCode = "404", description = "Не найден",
            content = @Content(mediaType = "application/problem+json", schema = @Schema(implementation = org.springframework.http.ProblemDetail.class)))
    })
    public User get(@PathVariable Long id) { return new User(id, "Alice"); }

    public record User(Long id, String name) { }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.responses

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.net.URI

@RestController
@RequestMapping("/api/users")
class UserController {

    @PostMapping
    @Operation(summary = "Создать пользователя")
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "201",
                description = "Создано",
                headers = [io.swagger.v3.oas.annotations.headers.Header(
                    name = "Location",
                    description = "URL нового ресурса",
                    schema = Schema(type = "string")
                )],
                content = [Content(mediaType = "application/json", schema = Schema(implementation = User::class))]
            ),
            ApiResponse(
                responseCode = "400",
                description = "Неверные данные",
                content = [Content(mediaType = "application/problem+json", schema = Schema(implementation = org.springframework.http.ProblemDetail::class))]
            )
        ]
    )
    fun create(@RequestBody body: User): ResponseEntity<User> =
        ResponseEntity.created(URI.create("/api/users/1")).body(User(1, body.name))

    @GetMapping("/{id}")
    @Operation(summary = "Получить пользователя")
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "200",
                description = "OK",
                content = [Content(mediaType = "application/json", schema = Schema(implementation = User::class))]
            ),
            ApiResponse(
                responseCode = "404",
                description = "Не найден",
                content = [Content(mediaType = "application/problem+json", schema = Schema(implementation = org.springframework.http.ProblemDetail::class))]
            )
        ]
    )
    fun get(@PathVariable id: Long) = User(id, "Alice")

    data class User(val id: Long, val name: String)
}
```

---

## Множественные контент-типы и `produces/consumes`, файловые загрузки/выгрузки

Контент-негациация — стандарт HTTP, и OpenAPI должен её отражать. Если метод поддерживает несколько представлений, перечислите их в `produces`/`consumes` и в `@Content`. Это даёт генераторам понять, какой сериализатор выбрать, и помогает UI предложить нужный `Content-Type`/`Accept`.

Для файловых загрузок используйте `multipart/form-data` и `@RequestPart`. В OpenAPI схему части помечают `type: string, format: binary`. Если вместе с файлом идут метаданные, добавляйте вторую часть с JSON-DTO и не забывайте указать её медиатип как `application/json`.

Для скачивания бинарных данных указывайте `application/octet-stream` и при необходимости документируйте заголовки `Content-Disposition` и `Content-Length`. Если поддерживаете range-запросы для докачки, добавьте упоминание и пример заголовка `Range` в описании.

CSV и TSV популярны в интеграциях. Укажите `text/csv` и приложите небольшой пример в `examples`. Опишите правила экранирования запятых и переносов строк. Генераторы клиентам не помогут, зато документация сэкономит массу времени интеграторам.

JSON Lines (`application/x-ndjson`) удобен для потоковой выдачи множества объектов. В документации поясните, что каждая строка — отдельный JSON-объект, и приведите мини-пример трёх строк. Это снимет вопросы у QA и аналитиков.

Если метод может возвращать и JSON-ошибку, и бинарный файл, перечислите оба варианта в `responses` с разными `content`. Это редкость, но встречается при экспорте с возможной валидационной ошибкой.

Старайтесь не смешивать загрузку больших файлов с синхронной обработкой. Лучше загрузить файл, сразу вернуть `202 Accepted` и дать клиенту URL статуса. Документацией это описывается как асинхронный контракт, а не «подождите минуту».

Коды `415 Unsupported Media Type` и `406 Not Acceptable` — часть игры. Добавьте их в глобальные ответы, чтобы клиенты понимали, почему сервер отказался обрабатывать запрос. UI тогда тоже покажет подсказку про заголовки.

Для безопасности не забывайте про анти-скан: фильтруйте расширения и контент при загрузке, но в контракте фиксируйте, что сервер принимает именно `multipart/form-data`, а не «всё подряд». Это снижает неожиданности на бэке.

Наконец, примеры — ваше всё. Дайте хотя бы один `curl`-пример в `description`: `curl -F file=@report.csv http://...`. Swagger UI «прикроет» 80% сценариев, но команды часто копируют именно `curl`.

**Java — загрузка и скачивание файлов**

```java
package docs.files;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("/api/files")
public class FileController {

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    @Operation(summary = "Загрузить файл")
    @ApiResponse(responseCode = "204", description = "Загружено")
    public ResponseEntity<Void> upload(
        @RequestPart("file")
        @Schema(type = "string", format = "binary", description = "Файл для загрузки")
        MultipartFile file,
        @RequestPart(value = "meta", required = false)
        @Content(mediaType = "application/json", schema = @Schema(implementation = Meta.class))
        String metaJson
    ) {
        return ResponseEntity.noContent().build();
    }

    @GetMapping(value = "/download/{name}", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
    @Operation(summary = "Скачать файл")
    @ApiResponse(responseCode = "200", description = "Ок",
        content = @Content(mediaType = "application/octet-stream", schema = @Schema(type = "string", format = "binary")))
    public ResponseEntity<ByteArrayResource> download(@PathVariable String name) {
        byte[] bytes = "demo".getBytes();
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + name + "\"")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(new ByteArrayResource(bytes));
    }

    public record Meta(String comment) {}
}
```

**Kotlin — эквивалент**

```kotlin
package docs.files

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.core.io.ByteArrayResource
import org.springframework.http.HttpHeaders
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import org.springframework.web.multipart.MultipartFile

@RestController
@RequestMapping("/api/files")
class FileController {

    @PostMapping("/upload", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
    @Operation(summary = "Загрузить файл")
    @ApiResponse(responseCode = "204", description = "Загружено")
    fun upload(
        @Schema(type = "string", format = "binary", description = "Файл для загрузки")
        @RequestPart("file") file: MultipartFile,
        @Content(mediaType = "application/json", schema = Schema(implementation = Meta::class))
        @RequestPart(value = "meta", required = false) metaJson: String?
    ): ResponseEntity<Void> = ResponseEntity.noContent().build()

    @GetMapping("/download/{name}", produces = [MediaType.APPLICATION_OCTET_STREAM_VALUE])
    @Operation(summary = "Скачать файл")
    @ApiResponse(
        responseCode = "200",
        description = "Ок",
        content = [Content(mediaType = "application/octet-stream", schema = Schema(type = "string", format = "binary"))]
    )
    fun download(@PathVariable name: String): ResponseEntity<ByteArrayResource> {
        val bytes = "demo".toByteArray()
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"$name\"")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(ByteArrayResource(bytes))
    }

    data class Meta(val comment: String?)
}
```

---

## Документация для WebFlux (reactive types) и асинхронных ответов

В реактивном стеке методы возвращают `Mono<T>` или `Flux<T>`. В контракте это по-прежнему «обычный» ответ с типом `T` или массивом `T`. Springdoc автоматически разворачивает реактивные типы, но полезно явно указывать медиа-типы и семантику стриминга, если это не классический JSON.

Server-Sent Events (SSE) — распространённый паттерн для стриминга событий. Документируйте `produces = "text/event-stream"` и описывайте структуру события: какой объект в поле `data`, какие заголовки устанавливаются, нужна ли авторизация для длительного соединения. Это снимет вопросы у фронта.

Асинхронные операции часто возвращают `202 Accepted` и ссылку на ресурс статуса. В описании операции поясните контракт: как часто пуллить, какие коды статуса возможны, что вернётся при завершении. Это «не просто REST», это жизненный цикл.

Для больших потоков данных используйте NDJSON (`application/x-ndjson`) и помечайте схему элемента отдельно. В UI такие ответы не проигрываются, но контракт подскажет интеграторам правильный парсер. Это важный момент для data-команд.

Реактивные контроллеры дружат со Swagger UI, но кнопка «Try it out» не лучший инструмент для стриминга. В описании дайте `curl`-подсказки: `curl -N` для SSE, правильные заголовки `Accept`. Это практичная помощь.

Ошибки в реактивном мире те же самые. RFC7807 прекрасно летит в `Mono<ProblemDetail>` с нужным статусом. Документируйте это, чтобы клиенты не ожидали «особых» форматов только из-за Flux/Mono. Контракт остаётся единым.

Таймауты и отмена — часть контракта. Если сервер закрывает SSE через 60 минут, напишите об этом. Если поддерживается `Last-Event-ID`, добавьте параметр и пример. Такие детали спасают от «непонятных» отвалов в проде.

Backpressure — не предмет OpenAPI, но бизнес-ограничения можно отразить словами: «не чаще, чем 1 событие/сек», «максимум 10K событий за запрос». Генераторам всё равно, а людям — нет.

В WebFlux-приложении используйте стартер `springdoc-openapi-starter-webflux-ui`. Смешивание MVC и WebFlux-стартеров приводит к странным конфликтам бинов. Версии библиотек должны соответствовать Spring Boot 3.x.

Тестируйте спецификацию реактивных методов так же, как синхронных. Выгрузка `/v3/api-docs` не исполняет поток — она описывает форму. Если при изменении метода с `Mono<User>` на `Mono<Void>` забыли обновить описание — CI подскажет.

**Gradle (Groovy/Kotlin) — WebFlux и springdoc**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-webflux"
    implementation "org.springdoc:springdoc-openapi-starter-webflux-ui:2.6.0"
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springdoc:springdoc-openapi-starter-webflux-ui:2.6.0")
}
```

**Java — WebFlux: Mono/Flux, SSE и 202 Accepted**

```java
package docs.webflux;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.http.*;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.time.Duration;

@RestController
@RequestMapping("/api/reactive")
public class ReactiveController {

    @PostMapping("/jobs")
    @Operation(summary = "Запустить задачу", description = "Возвращает 202 и ссылку на статус")
    @ApiResponse(responseCode = "202", description = "Принято",
        headers = @Header(name = "Location", description = "URL статуса", schema = @Schema(type = "string")))
    public ResponseEntity<Void> start() {
        return ResponseEntity.accepted().location(URI.create("/api/reactive/jobs/1")).build();
    }

    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    @Operation(summary = "События SSE",
        description = "Стримит события. Используйте Accept: text/event-stream и curl -N.")
    @ApiResponse(responseCode = "200", description = "OK",
        content = @Content(mediaType = "text/event-stream",
            schema = @Schema(implementation = String.class)))
    public Flux<ServerSentEvent<String>> events() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(i -> ServerSentEvent.builder("tick-" + i).build())
            .take(5);
    }

    @GetMapping(value = "/ndjson", produces = "application/x-ndjson")
    @Operation(summary = "Поток JSON Lines")
    @ApiResponse(responseCode = "200", description = "OK",
        content = @Content(mediaType = "application/x-ndjson", schema = @Schema(implementation = Item.class)))
    public Flux<Item> ndjson() {
        return Flux.just(new Item("a"), new Item("b"));
    }

    public record Item(String value) { }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.webflux

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.http.codec.ServerSentEvent
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Flux
import java.net.URI
import java.time.Duration

@RestController
@RequestMapping("/api/reactive")
class ReactiveController {

    @PostMapping("/jobs")
    @Operation(summary = "Запустить задачу", description = "Возвращает 202 и ссылку на статус")
    @ApiResponse(
        responseCode = "202",
        description = "Принято",
        headers = [io.swagger.v3.oas.annotations.headers.Header(
            name = "Location", description = "URL статуса", schema = Schema(type = "string")
        )]
    )
    fun start(): ResponseEntity<Void> =
        ResponseEntity.accepted().location(URI.create("/api/reactive/jobs/1")).build()

    @GetMapping("/events", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    @Operation(summary = "События SSE", description = "Стримит события. Используйте Accept: text/event-stream и curl -N.")
    @ApiResponse(
        responseCode = "200",
        description = "OK",
        content = [Content(mediaType = "text/event-stream", schema = Schema(implementation = String::class))]
    )
    fun events(): Flux<ServerSentEvent<String>> =
        Flux.interval(Duration.ofSeconds(1))
            .map { ServerSentEvent.builder("tick-$it").build() }
            .take(5)

    @GetMapping("/ndjson", produces = ["application/x-ndjson"])
    @Operation(summary = "Поток JSON Lines")
    @ApiResponse(
        responseCode = "200",
        description = "OK",
        content = [Content(mediaType = "application/x-ndjson", schema = Schema(implementation = Item::class))]
    )
    fun ndjson(): Flux<Item> = Flux.just(Item("a"), Item("b"))

    data class Item(val value: String)
}
```

# 5. Модели данных и валидация

## `@Schema`/`@ArraySchema`, описания полей, `format`, `nullable`, дефолты

Аннотации `@Schema` и `@ArraySchema` — мост между Java/Kotlin-моделями и OpenAPI-схемой. Они позволяют явно описывать тип поля, формат, обязательность, дефолт и человекочитаемое описание, которое увидит разработчик в Swagger UI/Redoc. Чем точнее вы заполните эти атрибуты, тем меньше «серых зон» останется у потребителей.

`description` у поля — это не украшение, а контракт. Здесь стоит кратко указать бизнес-смысл, допустимые значения и пограничные состояния. К примеру: «`externalId` — идентификатор партнёра, уникален в рамках партнёра, не путать с внутренним `id`».

`format` (например, `uuid`, `date`, `date-time`, `email`, `uri`) влияет на генераторы клиентов и валидаторы: они смогут выбрать подходящий тип или предзаполнить пример корректным значением. Если формат нетривиальный (например, денежные суммы), лучше добавить и формат, и пример.

`nullable` важно различать с «опциональностью». В OpenAPI поле может быть опущено (optional) или присутствовать со значением `null` (nullable). В Java это отражается через отсутствие/наличие валидации, в Kotlin — через `T?`. Для внешних клиентов разница принципиальна, фиксируйте её в схеме явно.

`defaultValue`/`default` в `@Schema` показывают клиенту предполагаемое значение по умолчанию. Это не заменяет серверную логику, но улучшает DX: UI подставит дефолт в форму, а SDK может использовать его как значение по умолчанию в конструкторе.

Коллекции описывайте через `@ArraySchema` или `@Schema` с `type="array"` и `implementation`/`items`. Важно указывать элемент (`items`) — без него UI и генераторы покажут «массив чего-то». Если массив может быть пустым — сделайте это очевидным в описании.

Для больших объектов, которые будут переиспользоваться, задавайте имя схемы через `@Schema(name="...")`. Это подскажет springdoc, как назвать компонент в `components.schemas`, и позволит ссылаться на него повторно без дублирования.

Если вы используете сериализацию Джексоном с переименованием полей (`@JsonProperty`), убедитесь, что имена в схеме совпадают с фактическими именами в wire-протоколе. `springdoc` учитывает `@JsonProperty`, но при ручных рефах легко ошибиться.

Старайтесь снабжать поля реалистичными `example`. Пример — это быстрый способ понять формат, особенно для дат и денежных значений. «`string`» как пример — это почти что «пример без примера».

Наконец, помните о локали и единицах измерения: если поле — «вес», укажите килограммы/граммы. Такие вещи лучше фиксировать текстом там же, где потребитель смотрит на поле — в `description`.

**Gradle (Groovy DSL) — зависимости для моделей и генерации схем**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-validation"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin DSL) — то же**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — DTO с `@Schema`/`@ArraySchema`, форматами, nullable и дефолтами**

```java
package oas.models.schema;

import io.swagger.v3.oas.annotations.media.ArraySchema;
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import java.time.OffsetDateTime;
import java.util.List;
import java.util.UUID;

@Schema(name = "Order", description = "Заказ в системе")
public class OrderDto {

    @Schema(description = "Внутренний ID заказа", format = "uuid", example = "550e8400-e29b-41d4-a716-446655440000")
    public UUID id;

    @NotBlank
    @Schema(description = "Внешний ID от партнёра", example = "ext-12345")
    public String externalId;

    @Schema(description = "Дата создания в UTC", format = "date-time", example = "2025-10-24T10:00:00Z", nullable = false)
    public OffsetDateTime createdAt;

    @ArraySchema(arraySchema = @Schema(description = "Список тегов заказа", defaultValue = "[]"),
                 schema = @Schema(implementation = String.class, example = "priority"))
    public List<String> tags;

    @Schema(description = "Комментарий оператора (может отсутствовать)", nullable = true, example = "Позвонить клиенту")
    public String comment;
}
```

**Kotlin — эквивалент DTO (nullability через типы `?`)**

```kotlin
package oas.models.schema

import io.swagger.v3.oas.annotations.media.ArraySchema
import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.NotBlank
import java.time.OffsetDateTime
import java.util.UUID

@Schema(name = "Order", description = "Заказ в системе")
data class OrderDto(
    @Schema(description = "Внутренний ID заказа", format = "uuid", example = "550e8400-e29b-41d4-a716-446655440000")
    val id: UUID?,
    @field:NotBlank
    @Schema(description = "Внешний ID от партнёра", example = "ext-12345")
    val externalId: String,
    @Schema(description = "Дата создания в UTC", format = "date-time", example = "2025-10-24T10:00:00Z")
    val createdAt: OffsetDateTime,
    @field:ArraySchema(
        arraySchema = Schema(description = "Список тегов заказа", defaultValue = "[]"),
        schema = Schema(implementation = String::class, example = "priority")
    )
    val tags: List<String> = emptyList(),
    @Schema(description = "Комментарий оператора (может отсутствовать)", nullable = true, example = "Позвонить клиенту")
    val comment: String?
)
```

---

## Enum-типы, oneOf/anyOf/allOf (наследование и композиция DTO)

Enum в OpenAPI — мощный способ ограничить значения и сделать UI/SDK самодокументируемыми. В Java/Kotlin привычные `enum`-классы автоматически отрендерятся как перечисления, но часто стоит дополнительно описать `description` каждого значения через `@Schema` на константе.

`oneOf` помогает описать поля-полиморфы: например, «детали платежа» могут быть картой или банковским переводом. Для корректного десериализатора и генераторов полезно добавить `discriminator` — поле, по которому клиент/сервер отличают конкретный тип.

`anyOf` используют реже, когда допустимо соответствовать нескольким альтернативным схемам одновременно. Это повышает гибкость, но усложняет валидацию и код SDK — применяйте осознанно и документируйте примеры.

`allOf` выражает композицию/наследование: «базовый» объект + дополнительные поля. В коде это обычное наследование/делегирование; в схеме — объединение нескольких схем. Удобно для паттерна «база + расширения» между версиями.

В Java `@Schema(oneOf = {A.class, B.class})` можно ставить на поле полиморфного типа или на контейнер-класс. Для production-качества добавляйте `discriminatorProperty` (в YAML — `discriminator`) и согласованный «типовой» enum.

В Kotlin хорошо ложатся `sealed class`/`sealed interface` для выражения закрытых иерархий. Они дисциплинируют разработчика: компилятор требует учесть все варианты, а спецификация — задокументировать их.

Не забывайте про сериализацию. Если используете Джексон с `@JsonTypeInfo`/`@JsonSubTypes`, синхронизируйте значения с `discriminator` OpenAPI. Иначе клиенты сгенерируют поля, которых нет в wire-протоколе.

Для enum полезно задать `example` на уровне поля-модели, чтобы UI показывал реальный кейс. Также можно добавить `x-enum-varnames`/`x-enum-descriptions` через расширения, если нужно отразить более богатые метаданные.

Полиморфизм усложняет тесты и backward-совместимость: новый подтип — потенциально ломающее изменение для старых клиентов. Фиксируйте это в changelog и не меняйте семантику `discriminator`.

И наконец, не бойтесь комбинировать подходы: наследование (`allOf`) + поле `oneOf` в том же объекте — нормальный инструмент при сложных моделях, если это оправдано доменной логикой.

**Gradle (Groovy DSL) — без изменений к зависимостям**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — enum + поле `oneOf` с дискриминатором**

```java
package oas.models.polymorph;

import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "Способ оплаты")
public enum PaymentMethod {
    @Schema(description = "Банковская карта") CARD,
    @Schema(description = "Банковский перевод") BANK
}

@Schema(discriminatorProperty = "type", oneOf = {CardDetails.class, BankDetails.class}, description = "Детали платежа")
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = CardDetails.class, name = "CARD"),
    @JsonSubTypes.Type(value = BankDetails.class, name = "BANK")
})
public abstract class PaymentDetails { }

@Schema(description = "Оплата картой")
class CardDetails extends PaymentDetails {
    @Schema(description = "Маскированный PAN", example = "411111******1111")
    public String maskedPan;
}

@Schema(description = "Оплата банковским переводом")
class BankDetails extends PaymentDetails {
    @Schema(description = "BIC/SWIFT банка", example = "SABRRUMMXXX")
    public String bic;
}

@Schema(description = "Платёж")
public class Payment {
    @Schema(description = "Метод оплаты")
    public PaymentMethod method;

    @Schema(description = "Детали платежа", oneOf = {CardDetails.class, BankDetails.class}, required = true)
    public PaymentDetails details;
}
```

**Kotlin — enum + `sealed class` для полиморфизма**

```kotlin
package oas.models.polymorph

import com.fasterxml.jackson.annotation.JsonSubTypes
import com.fasterxml.jackson.annotation.JsonTypeInfo
import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Способ оплаты")
enum class PaymentMethod {
    @Schema(description = "Банковская карта") CARD,
    @Schema(description = "Банковский перевод") BANK
}

@Schema(discriminatorProperty = "type", oneOf = [CardDetails::class, BankDetails::class], description = "Детали платежа")
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes(
    JsonSubTypes.Type(value = CardDetails::class, name = "CARD"),
    JsonSubTypes.Type(value = BankDetails::class, name = "BANK")
)
sealed class PaymentDetails

@Schema(description = "Оплата картой")
class CardDetails(
    @Schema(description = "Маскированный PAN", example = "411111******1111")
    val maskedPan: String
) : PaymentDetails()

@Schema(description = "Оплата банковским переводом")
class BankDetails(
    @Schema(description = "BIC/SWIFT банка", example = "SABRRUMMXXX")
    val bic: String
) : PaymentDetails()

@Schema(description = "Платёж")
data class Payment(
    @Schema(description = "Метод оплаты") val method: PaymentMethod,
    @Schema(description = "Детали платежа", oneOf = [CardDetails::class, BankDetails::class], required = true)
    val details: PaymentDetails
)
```

---

## Bean Validation → OpenAPI: `@NotNull`, `@Size`, `@Pattern`, диапазоны чисел

Связка Bean Validation и OpenAPI — удобный способ автоматически перенести ограничения из кода в контракт. `springdoc` читает аннотации `jakarta.validation.*` и заполняет для полей `minLength`, `maxLength`, `pattern`, `minimum`, `maximum`, `exclusive*` и флаг `required`.

`@NotNull` делает поле обязательным в схеме (попадает в `required`), но это не то же самое, что «нельзя опустить поле» при сериализации. Для коллекций добавляйте `@NotEmpty`/`@Size(min=1)`, иначе пустой список пройдет валидацию.

`@Size` мапится в `minLength`/`maxLength` для строк и в `minItems`/`maxItems` для коллекций. Это важно для SDK и UI: разработчику легче понять, почему сервер отвергает «слишком длинный» инпут.

`@Pattern` кладётся в `pattern` (регулярка). Документируйте коротко человека-понятный формат рядом, regex — это для машины, а человек должен быстро понять, что от него хотят.

Числовые ограничения (`@Min`, `@Max`, `@DecimalMin`, `@DecimalMax`, `@Positive`, `@Negative`) отображаются в `minimum`/`maximum` и/или `exclusiveMinimum`/`exclusiveMaximum`. Выбирайте аккуратно: `@Positive` — это `> 0`, а не `>= 0`.

`@Email`, `@URL`, `@Past`, `@Future` — полезная семантика, которую стоит добавлять валидацией и кратко повторять в `description`. Контракт и валидатор должны говорить одно и то же, иначе поведение будет неожиданным.

На уровне контроллера `@Validated` и `@Valid` включают рекурсивную проверку вложенных объектов. Это не влияет на схему, но влияет на фактическое поведение. В документации добавьте примеры ошибок валидации.

Глобальная модель ошибки (например, RFC7807) помогает потребителям. Укажите, как приходят сообщения о нарушенных ограничениях, какие коды ошибок и поля «подсвечиваются».

Будьте внимательны к локалям и десятичным разделителям, если принимаете числа в строковом виде. Лучше избегать строк для чисел, но если без этого не обойтись — подчеркните формат в `pattern` и примерах.

Наконец, покрывайте валидацию тестами на уровне DTO и интеграционными тестами для контроллеров. Спецификация отражает ограничения, но баги проявятся именно в рантайме на реальных входах.

**Gradle (Groovy/Kotlin) — валидация уже добавлена**

```groovy
dependencies { implementation "org.springframework.boot:spring-boot-starter-validation" }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-validation") }
```

**Java — DTO с валидацией, мэппящейся в схему**

```java
package oas.models.validation;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.util.List;

@Schema(description = "Создание счёта")
public class InvoiceCreate {

    @NotBlank @Schema(description = "Имя клиента", example = "ООО Ромашка")
    public String customerName;

    @Email @Schema(description = "Email для отправки", example = "billing@example.com")
    public String email;

    @Valid @NotEmpty @Schema(description = "Позиции счёта", minItems = 1)
    public List<Line> lines;

    @Positive @Schema(description = "Итого к оплате, в валютных единицах с копейками", example = "1234.56")
    public BigDecimal total;

    public static class Line {
        @NotBlank @Schema(description = "Название позиции", example = "Подписка PRO")
        public String title;

        @Positive @Schema(description = "Количество", example = "1")
        public Integer qty;

        @DecimalMin(value = "0.00") @Schema(description = "Цена за единицу", example = "1234.56")
        public BigDecimal unitPrice;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package oas.models.validation

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.Valid
import jakarta.validation.constraints.*
import java.math.BigDecimal

@Schema(description = "Создание счёта")
data class InvoiceCreate(
    @field:NotBlank @Schema(description = "Имя клиента", example = "ООО Ромашка")
    val customerName: String,
    @field:Email @Schema(description = "Email для отправки", example = "billing@example.com")
    val email: String?,
    @field:Valid @field:NotEmpty @Schema(description = "Позиции счёта", minItems = 1)
    val lines: List<Line>,
    @field:Positive @Schema(description = "Итого к оплате", example = "1234.56")
    val total: BigDecimal
) {
    data class Line(
        @field:NotBlank @Schema(description = "Название позиции", example = "Подписка PRO")
        val title: String,
        @field:Positive @Schema(description = "Количество", example = "1")
        val qty: Int,
        @field:DecimalMin("0.00") @Schema(description = "Цена за единицу", example = "1234.56")
        val unitPrice: BigDecimal
    )
}
```

---

## Примеры: `example`, `examples`, `@ExampleObject` для сложных payload’ов

Примеры — быстрый путь от теории к практике. В UI они становятся готовыми телами запросов и ответов, а в SDK — подсказками для тестов. Используйте `@Schema(example="...")` для простых полей и `@ExampleObject`/`examples` в уровне операции, когда нужно показать целый payload.

`@ExampleObject` поддерживает `name`, `summary`, `description`, `value` (inline) и `externalValue` (ссылка). Практичнее хранить сложные примеры inline рядом с кодом, чтобы ревью видеть контракт в одном месте. Для очень больших payload удобнее давать ссылку на статик-файл.

Для операций с несколькими медиа-типами указывайте примеры для каждого: JSON, CSV, NDJSON. Это сделает UI полезным даже для «не-JSON» сценариев — разработчик увидит формат и сможет быстро собрать `curl`.

Для ошибок тоже нужны примеры. RFC7807-ответ с полями `type`, `title`, `detail`, `instance`, `traceId` — это минимум. Чем проще воспроизвести «типовую» ошибку, тем меньше вопросов к поддержке.

В `examples` можно показать альтернативы одного и того же запроса: «минимальный», «полный», «с флагом X». Это хорошая практика для «богатых» DTO, где клиентам сложно понять обязательные/опциональные части.

Если у вас полиморфизм (`oneOf`), примеры должны покрывать каждый вариант. Иначе потребители поймут только «среднюю температуру» и начнут гадать, чего хочет сервер.

Даты/время — всегда с таймзоной и в одном формате. Унифицированный `2025-10-24T10:00:00Z` лучше, чем «10:00 24/10/2025». В примерах не допускайте неоднозначностей.

Старайтесь размещать примеры там, где их ожидают: на уровне `@RequestBody`, `@ApiResponse` и на полях. Тогда UI соберёт «полную картину»: описание + пример на вход и пример на выход.

Не бойтесь «длинных» примеров: читаемый формат с отступами — это инвестиция в DX. Команды часто копируют payload из UI в Postman/HTTPie — сделайте так, чтобы он работал сразу.

И наконец, тестируйте примеры. Минимальная автоматизация — прогнать их через JSON-валидатор в CI, максимум — крутить через мок-сервер. «Сломанный» пример подрывает доверие к документации.

**Gradle (Groovy/Kotlin) — без изменений к зависимостям**

```groovy
dependencies { implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0" }
```

```kotlin
dependencies { implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0") }
```

**Java — `@ExampleObject` на запросе и ответе**

```java
package oas.models.examples;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.*;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    @Operation(summary = "Создать клиента")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "OK",
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = Customer.class),
                examples = @ExampleObject(name = "created", value = """
                    { "id":"c_1","name":"Alice","email":"alice@example.com" }
                """))),
        @ApiResponse(responseCode = "400", description = "Ошибка валидации",
            content = @Content(mediaType = "application/problem+json",
                examples = @ExampleObject(name = "bad_request", value = """
                    { "type":"https://errors.example.com/validation",
                      "title":"Validation failed",
                      "detail":"email must be a well-formed email address",
                      "status":400 }""")))
    })
    public Customer create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            required = true,
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = CustomerCreate.class),
                examples = {
                    @ExampleObject(name = "minimal", value = """{ "name":"Alice" }"""),
                    @ExampleObject(name = "full", value = """{ "name":"Alice", "email":"alice@example.com" }""")
                }
        )) @RequestBody CustomerCreate body) {
        return new Customer("c_1", body.name, body.email);
    }

    public record Customer(String id, String name, String email) {}
    public static class CustomerCreate { public String name; public String email; }
}
```

**Kotlin — эквивалент**

```kotlin
package oas.models.examples

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.ExampleObject
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import io.swagger.v3.oas.annotations.responses.ApiResponses
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/customers")
class CustomerController {

    @PostMapping(consumes = [MediaType.APPLICATION_JSON_VALUE], produces = [MediaType.APPLICATION_JSON_VALUE])
    @Operation(summary = "Создать клиента")
    @ApiResponses(
        value = [
            ApiResponse(
                responseCode = "200",
                description = "OK",
                content = [Content(
                    mediaType = "application/json",
                    schema = Schema(implementation = Customer::class),
                    examples = [ExampleObject(name = "created", value = """{ "id":"c_1","name":"Alice","email":"alice@example.com" }""")]
                )]
            ),
            ApiResponse(
                responseCode = "400",
                description = "Ошибка валидации",
                content = [Content(
                    mediaType = "application/problem+json",
                    examples = [ExampleObject(name = "bad_request", value = """{
                        "type":"https://errors.example.com/validation",
                        "title":"Validation failed",
                        "detail":"email must be a well-formed email address",
                        "status":400
                    }""")]
                )]
            )
        ]
    )
    fun create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            required = true,
            content = [Content(
                mediaType = "application/json",
                schema = Schema(implementation = CustomerCreate::class),
                examples = [
                    ExampleObject(name = "minimal", value = """{ "name":"Alice" }"""),
                    ExampleObject(name = "full", value = """{ "name":"Alice", "email":"alice@example.com" }""")
                ]
            )]
        )
        @RequestBody body: CustomerCreate
    ): Customer = Customer("c_1", body.name, body.email)

    data class Customer(val id: String, val name: String, val email: String?)
    class CustomerCreate { var name: String = ""; var email: String? = null }
}
```

---

## Маскирование/скрытие полей (`accessMode`, `writeOnly/readOnly`)

Иногда поле присутствует только на записи (пароль) или только на чтении (серверный `id`, `createdAt`). В OpenAPI это отражают флагами `writeOnly` и `readOnly`. В springdoc они настраиваются через `@Schema(accessMode = WRITE_ONLY|READ_ONLY)`.

`readOnly` означает: клиент не должен отправлять это поле в запросах; сервер заполняет его сам. Генераторы SDK помечают такие поля только для чтения, а UI убирает их из формы запроса. Это помогает не смешивать обязанности и не путать потребителя.

`writeOnly` — противоположное. Поле используется для ввода (например, пароль), но никогда не возвращается в ответах. Это критично для безопасности: вы явно показываете, что секреты не «утекут» из API.

Сочетание read/write-only делает контракты самодокументируемыми. Разработчик, глядя на модель, понимает, какие поля он «получает», а какие «задаёт». Это особенно полезно в моделях «создание/чтение» с общими именами.

Для «масок» (например, хранить PAN в маскированном виде) используйте отдельное поле `maskedPan` как `readOnly`, а «сырые» данные вовсе не возвращайте. Так вы не нарушите PCI DSS и не запутаете клиентов.

При миграции моделей важно не перепутать флаги. Если вы пометите `password` как `readOnly`, UI перестанет показывать его в запросе — клиенты не смогут отправить пароль. Автотесты по спеке быстро поймают такие регрессии.

Валидация и флаги независимы. Поле может быть `@NotBlank` и при этом `writeOnly` — валидатор сработает на входе, а в ответе поле просто отсутствует. В схеме это корректная комбинация.

Скрывать поля «совсем» можно через `@Hidden`, но это уже другой инструмент: поле исчезнет из схемы, и потребители не узнают о его существовании. Обычно лучше использовать read/write-only — контракты становятся прозрачнее.

Если у вас разные формы DTO для create/read/update, можно вообще обойтись без флагов, разделив модели. Это упрощает логику, но увеличивает число классов. Выбор — вопрос стиля и дисциплины в проекте.

И наконец, помните: флаги в документации не заменяют серверную логику. Не полагайтесь на UI — сервер не должен возвращать write-only поля «случайно», даже если документация «сказала не возвращать».

**Gradle (Groovy/Kotlin) — те же зависимости**

```groovy
dependencies { implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0" }
```

```kotlin
dependencies { implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0") }
```

**Java — writeOnly/readOnly поля**

```java
package oas.models.access;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import java.time.OffsetDateTime;

@Schema(description = "Пользователь")
public class UserDto {

    @Schema(description = "ID пользователя", readOnly = true, example = "u_123")
    public String id;

    @NotBlank
    @Schema(description = "Email", example = "user@example.com")
    public String email;

    @NotBlank
    @Schema(description = "Пароль (write-only)", accessMode = Schema.AccessMode.WRITE_ONLY, example = "P@ssw0rd!")
    public String password;

    @Schema(description = "Дата создания (read-only)", readOnly = true, format = "date-time", example = "2025-10-24T10:00:00Z")
    public OffsetDateTime createdAt;
}
```

**Kotlin — эквивалент**

```kotlin
package oas.models.access

import io.swagger.v3.oas.annotations.media.Schema
import jakarta.validation.constraints.NotBlank
import java.time.OffsetDateTime

@Schema(description = "Пользователь")
data class UserDto(
    @Schema(description = "ID пользователя", readOnly = true, example = "u_123")
    val id: String?,
    @field:NotBlank
    @Schema(description = "Email", example = "user@example.com")
    val email: String,
    @field:NotBlank
    @Schema(description = "Пароль (write-only)", accessMode = Schema.AccessMode.WRITE_ONLY, example = "P@ssw0rd!")
    val password: String,
    @Schema(description = "Дата создания (read-only)", readOnly = true, format = "date-time", example = "2025-10-24T10:00:00Z")
    val createdAt: OffsetDateTime?
)
```

---

## Референсы к шареным схемам (`components.schemas`), переиспользование

Переиспользование схем — ключ к консистентности. Один «эталонный» `Money`, один `Problem`, одна `Page<T>` — и весь ландшафт API говорит на одном языке. В OpenAPI это достигается через `components.schemas` и `$ref`.

В code-first springdoc сам помещает ваши классы в `components.schemas`, когда они где-то используются. Чтобы явно сослаться на уже объявленную схему, можно указать `@Schema(implementation = Money.class)` на поле или `@Schema(ref="#/components/schemas/Money")`, если нужно именно «ссылка, а не inline».

Переиспользование критично для клиентов: SDK создаёт один класс `Money`, а не 10 похожих типов в разных пакетах. Это снимает боль преобразований между почти-одинаковыми моделями и уменьшает шанс «неправильной» сериализации.

Именуйте схемы стабильно. `@Schema(name="Money")` фиксирует имя в компоненте; без этого springdoc выведет его из имени класса. Если в проекте два разных `Money` в разных пакетах, задайте уникальные `name`, чтобы избежать конфликта.

Общие схемы выносите в отдельный модуль и подключайте его там, где нужно. Это дисциплинирует архитектуру и предотвращает «локальные форки» моделей, которые начинают отличаться и ломать совместимость.

При эволюции схем старайтесь не менять смысл и формат полей. Добавление поля — обычно совместимо, а вот переименование/удаление — нет. Дифф по спеке (oas-diff/Spectral) в CI поможет держать руку на пульсе.

В некоторых случаях полезно описать общую схему в YAML, а в коде лишь сослаться на неё через `ref`. Это уместно для «контрактно-владельческих» моделей, которые не должны «ездить» из-за случайных аннотаций разработчика.

`components.responses`, `components.parameters` и `components.headers` тоже стоит переиспользовать. Например, стандартный `ProblemResponse` или `PaginationParameters` облегчает поддержку и делает документацию компактнее.

Следите за тем, чтобы «общие» схемы не тянули за собой бизнес-зависимости. DTO-коммон не должен знать о доменах, иначе его начнут менять разные команды по своим нуждам.

И наконец, документируйте правила переиспользования в README/Confluence. Архитектурные решения должны быть понятны всем, иначе через год вы снова увидите «зоопарк» моделей.

**Gradle (Groovy/Kotlin) — модуль с общими моделями**

```groovy
dependencies {
    api "org.springframework.boot:spring-boot-starter-validation"
    api "org.springdoc:springdoc-openapi-starter-common:2.6.0"
}
```

```kotlin
dependencies {
    api("org.springframework.boot:spring-boot-starter-validation")
    api("org.springdoc:springdoc-openapi-starter-common:2.6.0")
}
```

**Java — общая схема `Money` и её референс в модели**

```java
package oas.models.shared;

import io.swagger.v3.oas.annotations.media.Schema;
import java.math.BigDecimal;
import java.util.Currency;

@Schema(name = "Money", description = "Денежная сумма в ISO-4217 валюте")
public class Money {
    @Schema(description = "Сумма", example = "123.45")
    public BigDecimal amount;
    @Schema(description = "Код валюты ISO-4217", example = "RUB")
    public String currency;
}

package oas.models.shared;

import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "Позиция корзины")
public class CartLine {
    @Schema(description = "Наименование", example = "Подписка PRO")
    public String title;

    @Schema(implementation = Money.class, description = "Цена с валютой")
    public Money price;
}
```

**Kotlin — эквивалент**

```kotlin
package oas.models.shared

import io.swagger.v3.oas.annotations.media.Schema
import java.math.BigDecimal

@Schema(name = "Money", description = "Денежная сумма в ISO-4217 валюте")
data class Money(
    @Schema(description = "Сумма", example = "123.45") val amount: BigDecimal,
    @Schema(description = "Код валюты ISO-4217", example = "RUB") val currency: String
)

package oas.models.shared

import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Позиция корзины")
data class CartLine(
    @Schema(description = "Наименование", example = "Подписка PRO") val title: String,
    @Schema(implementation = Money::class, description = "Цена с валютой") val price: Money
)
```


# 6. Ошибки, стандартизация ответов и HATEOAS

## Единая модель ошибки (например, RFC7807/Problem Details) и глобальные мапперы

Единая модель ошибки — это фундамент, на котором строится предсказуемость интеграций. Когда все сервисы возвращают ошибки в одном формате, клиентский код становится проще: обработчики одинаковы, логирование унифицировано, алерты предсказуемы. На практике удобнее всего взять за основу RFC 7807 (Problem Details), который описывает поля `type`, `title`, `status`, `detail`, `instance` и допускает расширения для доменных атрибутов, например `traceId` или `errorCode`.

В Spring Boot 3 уже есть класс `ProblemDetail`, который непосредственно реализует идею RFC 7807. Мы можем создавать его вручную или через фабрики `ProblemDetail.forStatus(...)`, а затем обогащать доменными полями через `setProperty`. Это снимает боль сериализации и делает код лаконичным. Важно, что сериализация идёт с `application/problem+json`, и это нужно зафиксировать и в документации, и в глобальных мапперах.

Глобальные мапперы — это `@ControllerAdvice` с `@ExceptionHandler`, где мы ловим доменные исключения и переводим их в `ProblemDetail`. Такой слой не должен протекать бизнес-логикой: он только преобразует исключения в контракт и логирует контекст. В результате контроллеры остаются «чистыми», а формат ошибок — единообразным.

Полезно договориться о стандартных доменных полях. Обычно это `errorCode` (стабильный, пригодный для агрегирования), `traceId` (для корреляции в логах и APM), иногда `violations` для валидационных ошибок. Стабильность имён критична: эти поля будут использоваться в мониторинге и аналитике.

Важно различать «проблемы» уровня транспорта и бизнес-ошибки. Например, `400 Bad Request` с `type` «validation-error» и списком нарушений — это транспортный уровень, а «недостаточно средств» — бизнес-ошибка со своим `type`, `title` и кодом. В обоих случаях контракт единый, но семантика разная, и это видно в `type`.

Отдельная тема — логирование. Формируя `ProblemDetail`, сразу прикладывайте `traceId` из MDC (или `X-Request-Id`) и не забывайте логировать исходное исключение на уровне маппера. Клиенту от этого ни жарко ни холодно, но вам — проще при разборе инцидентов.

Ещё один аспект — валидация Bean Validation. Исключения `MethodArgumentNotValidException` и `ConstraintViolationException` лучше переводить в «богатую» структуру `violations` с указанием поля/сообщения/кода. Это можно сделать одним маппером и переиспользовать во всех сервисах.

С точки зрения безопасности избегайте утечек. Никогда не включайте стектрейсы и внутренние идентификаторы в `detail` для продакшена. Если их хочется видеть в stage/dev, заведите флаг профиля и подмешивайте диагностическую информацию только там.

Для удобства клиентов документируйте типы проблем: короткое описание каждого `type` и стабильный URI (даже если это просто строка-ключ вида «urn:problem:…»). Тогда по одному полю клиент понимает, что за ошибка, без парсинга текстовых сообщений.

Наконец, помните о локализации. `title` и `detail` могут быть локализованы, но `type` и `errorCode` должны оставаться стабильными и не зависящими от языка. Это позволяет строить автоматические правила ретраев и категоризации.

**Gradle (Groovy) — зависимости**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-validation"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin) — то же**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**application.yml — медиа-тип ошибок и включение деталей в dev**

```yaml
server:
  error:
    include-message: always
    include-binding-errors: always
spring:
  profiles:
    group:
      dev: dev
```

**Java — единый маппер ошибок c Problem Details**

```java
package errors.mapping;

import jakarta.validation.ConstraintViolationException;
import org.slf4j.MDC;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.ErrorResponseException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import java.net.URI;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("urn:problem:validation-error"));
        pd.setTitle("Validation failed");
        pd.setDetail("Request contains invalid fields");
        pd.setProperty("violations", ex.getBindingResult().getFieldErrors().stream()
                .map(f -> new Violation(f.getField(), f.getDefaultMessage()))
                .collect(Collectors.toList()));
        pd.setProperty("traceId", MDC.get("traceId"));
        pd.setProperty("errorCode", "VALIDATION_ERROR");
        return pd;
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ProblemDetail handleConstraint(ConstraintViolationException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("urn:problem:constraint-violation"));
        pd.setTitle("Constraint violation");
        pd.setDetail(ex.getMessage());
        pd.setProperty("traceId", MDC.get("traceId"));
        pd.setProperty("errorCode", "CONSTRAINT_VIOLATION");
        return pd;
    }

    @ExceptionHandler(ErrorResponseException.class)
    public ProblemDetail handleSpring(ErrorResponseException ex) {
        var pd = ex.getBody();
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;
    }

    public record Violation(String field, String message) {}
}
```

**Kotlin — эквивалентный маппер**

```kotlin
package errors.mapping

import jakarta.validation.ConstraintViolationException
import org.slf4j.MDC
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.ErrorResponseException
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.net.URI

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            type = URI.create("urn:problem:validation-error")
            title = "Validation failed"
            detail = "Request contains invalid fields"
            setProperty("violations", ex.bindingResult.fieldErrors.map { Violation(it.field, it.defaultMessage) })
            setProperty("traceId", MDC.get("traceId"))
            setProperty("errorCode", "VALIDATION_ERROR")
        }

    @ExceptionHandler(ConstraintViolationException::class)
    fun handleConstraint(ex: ConstraintViolationException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            type = URI.create("urn:problem:constraint-violation")
            title = "Constraint violation"
            detail = ex.message
            setProperty("traceId", MDC.get("traceId"))
            setProperty("errorCode", "CONSTRAINT_VIOLATION")
        }

    @ExceptionHandler(ErrorResponseException::class)
    fun handleSpring(ex: ErrorResponseException): ProblemDetail =
        ex.body.apply { setProperty("traceId", MDC.get("traceId")) }
}

data class Violation(val field: String, val message: String?)
```

---

## Глобальные ответы: добавление 4xx/5xx через конфигурацию/операционные фильтры

Повторять одни и те же ошибки в каждой операции — утомительно и опасно: рано или поздно описания разъедутся. Лучше добавить глобальные ответы программно, пройдясь по всем операциям и приклеив стандартный набор `401/403/429/500` (и при необходимости `400`). Такой подход делает документацию консистентной и снижает риск забыть важный статус.

В springdoc это делается через `OpenApiCustomiser`: бин, который получает уже собранную модель OpenAPI и может модифицировать её. Мы можем пройтись по `paths → operations` и для каждой операции добавить недостающие ответы, не перетирая явно описанные разработчиком.

Важно не навредить. Если у операции уже есть кастомное описание `400`, не трогаем. Если у вас несколько медиа-типов для ошибок, добавляйте оба (`application/problem+json` и `application/json`), но держите единый главный. Консервативный вариант — всегда отдавать `problem+json` и документировать это явно.

Стоит хранить «шаблон» ответа в `components.responses` и переиспользовать через `$ref`. Но в аннотациях это неудобно, поэтому программное добавление через `OpenApiCustomiser` выгоднее: мы создаём один объект `ApiResponse` и вставляем его в каждую операцию.

Не забывайте про заголовки. Для `429` (rate limit) полезно задокументировать `Retry-After`. Для `401` при bearer— `WWW-Authenticate`. Эти мелочи экономят недели на интеграциях, когда потребители понимают, как правильно ретраить.

Глобальные ответы — это ещё и место для единых примеров ошибок. Даже если вы добавляете только «скелет», положите короткий пример `ProblemDetail`, чтобы UI не выглядел пустым. Потребители привыкли копировать-пробовать.

Согласуйте объём глобальных ответов. Избыточный список кодов может перегрузить UI, но отсутствие ключевых — хуже. Практика: `400/401/403/404/409/422/429/500` по умолчанию, а экзотику — локально, на уровне операции.

При желании можно наклеить ответы только на публичные группы (`GroupedOpenApi` для `/api/public/**`). Это снизит шум в «внутренних» спеках, где набор статусов может отличаться.

Проверяйте корректность в CI: выгрузите YAML и убедитесь, что в каждой операции появился набор стандартных ответов. Это легко сделать snapshot-тестами.

Не подменяйте код `500` бизнес-ошибками. Это резервный уровень, сигнализирующий о проблемах сервера. Бизнес-ошибки должны быть `4xx` с осмысленными `type` и `errorCode`.

И, наконец, глобальные ответы не заменяют единый маппер. Они только документируют следствие. Источник правды — ваш `@ControllerAdvice`.

**Java — добавление глобальных ответов через `OpenApiCustomiser`**

```java
package errors.global;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.media.Content;
import io.swagger.v3.oas.models.media.MediaType;
import io.swagger.v3.oas.models.responses.ApiResponse;
import org.springdoc.core.customizers.OpenApiCustomiser;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GlobalResponsesConfig {

    @Bean
    OpenApiCustomiser addGlobalResponses() {
        return (OpenAPI openApi) -> {
            var problem = new ApiResponse()
                .description("Problem Details")
                .content(new Content().addMediaType("application/problem+json", new MediaType()));
            openApi.getPaths().values().forEach(p -> p.readOperations().forEach(op -> {
                op.getResponses().putIfAbsent("401", clone(problem).description("Unauthorized"));
                op.getResponses().putIfAbsent("403", clone(problem).description("Forbidden"));
                op.getResponses().putIfAbsent("429", clone(problem).description("Too Many Requests"));
                op.getResponses().putIfAbsent("500", clone(problem).description("Internal Server Error"));
            }));
        };
    }

    private ApiResponse clone(ApiResponse src) {
        var r = new ApiResponse();
        r.setDescription(src.getDescription());
        r.setContent(src.getContent());
        return r;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package errors.global

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.media.Content
import io.swagger.v3.oas.models.media.MediaType
import io.swagger.v3.oas.models.responses.ApiResponse
import org.springdoc.core.customizers.OpenApiCustomiser
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class GlobalResponsesConfig {

    @Bean
    fun addGlobalResponses(): OpenApiCustomiser = OpenApiCustomiser { openApi: OpenAPI ->
        val problem = ApiResponse()
            .description("Problem Details")
            .content(Content().addMediaType("application/problem+json", MediaType()))
        openApi.paths.values.forEach { path ->
            path.readOperations().forEach { op ->
                op.responses.putIfAbsent("401", problem.clone().description("Unauthorized"))
                op.responses.putIfAbsent("403", problem.clone().description("Forbidden"))
                op.responses.putIfAbsent("429", problem.clone().description("Too Many Requests"))
                op.responses.putIfAbsent("500", problem.clone().description("Internal Server Error"))
            }
        }
    }

    private fun ApiResponse.clone(): ApiResponse = ApiResponse().also {
        it.description = this.description
        it.content = this.content
    }
}
```

**Gradle (оба DSL) — зависимости уже подключены**
(см. предыдущий пункт; дополнительных зависимостей не требуется)

---

## Контент-негациация, локализация сообщений в описаниях

Контент-негациация — это о честном диалоге клиента и сервера: клиент говорит, что он готов принять (`Accept`), сервер — чем может ответить. Если вы возвращаете `Problem Details`, фиксируйте `application/problem+json` и в коде, и в документации. Это избавит потребителей от сюрпризов, когда ошибка приходит «как обычный JSON», а не как `problem+json`.

С локализацией важно быть прагматичным. Локализуйте `title` и пользовательские `detail`, но держите стабильные машинные поля (`type`, `errorCode`) неизменными. Иначе вы потеряете возможность агрегировать метрики и строить автоматические реакции на ошибки.

Источником локализации служит `MessageSource` c бандлами сообщений и `LocaleResolver`/`LocaleContextResolver`. Приходит `Accept-Language` — формируете `ProblemDetail` с локализованными `title/detail`. Если заголовка нет, используйте дефолтную локаль (например, `en`) и фиксируйте это в описании API.

В спецификации полезно задокументировать заголовок `Accept-Language` как необязательный параметр для всех операций. Это можно сделать программно, добавив глобальный header-параметр через `OpenApiCustomiser`, чтобы не размножать аннотации по всем методам.

Помните, что «локализация» ошибок валидации может быть детерминирована аннотациями Bean Validation. Если вы уже используете сообщения из `messages.properties`, ваш маппер просто берёт `messageSource.getMessage(code, args, locale)` и вставляет в `detail`/`violations`. Это убирает дублирование.

Контент-негациация относится и к «нормальным» ответам. Если вы поддерживаете несколько `produces`, явно перечисляйте их в `@ApiResponse` и показывайте, какие поля появляются в каждом варианте. Клиентам не нравится угадывать.

Думайте про «границу ответственности». Нельзя обещать контент-типы, которые фактически не поддерживаются. Автотесты с `Accept: application/problem+json` на ошибочные сценарии помогут: если сервер вернул другой тип, тест упадёт.

Локализация в логах — отдельная тема. В логе пишите английские коды и ключи, а локализацию давайте в ответе клиенту. Это сделает диагностику в командах и ротейшенах понятной, особенно в международных.

Не забывайте про кэш: если вы отдаёте статические ресурсы с локализацией, заголовки Vary (`Vary: Accept-Language`) обязательны. Для API ошибок это редко критично, но полезно помнить в целом.

Наконец, в документации приведите хотя бы два примера `ProblemDetail` на разных языках и явно покажите заголовок запроса. Это снижает «стыдные» вопросы у интеграторов.

**Java — MessageSource + глобальный параметр `Accept-Language` в спецификации**

```java
package errors.i18n;

import io.swagger.v3.oas.models.parameters.Parameter;
import io.swagger.v3.oas.models.parameters.Parameter.In;
import org.springdoc.core.customizers.OpenApiCustomiser;
import org.springframework.context.MessageSource;
import org.springframework.context.annotation.*;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver;

import java.util.Locale;

@Configuration
public class I18nConfig {

    @Bean
    MessageSource messageSource() {
        var ms = new ResourceBundleMessageSource();
        ms.setBasenames("messages");
        ms.setDefaultEncoding("UTF-8");
        ms.setFallbackToSystemLocale(false);
        return ms;
    }

    @Bean
    LocaleResolver localeResolver() {
        var r = new AcceptHeaderLocaleResolver();
        r.setDefaultLocale(Locale.ENGLISH);
        return r;
    }

    @Bean
    OpenApiCustomiser addAcceptLanguageHeader() {
        return openApi -> openApi.getPaths().values().forEach(p -> p.readOperations().forEach(op -> {
            var lang = new Parameter().name("Accept-Language").in(In.HEADER.toString())
                    .description("Locale preference (e.g. en, ru)")
                    .required(false).schema(new io.swagger.v3.oas.models.media.StringSchema());
            op.addParametersItem(lang);
        }));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package errors.i18n

import io.swagger.v3.oas.models.media.StringSchema
import io.swagger.v3.oas.models.parameters.Parameter
import org.springdoc.core.customizers.OpenApiCustomiser
import org.springframework.context.MessageSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.support.ResourceBundleMessageSource
import org.springframework.web.servlet.LocaleResolver
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
import java.util.Locale

@Configuration
class I18nConfig {

    @Bean
    fun messageSource(): MessageSource = ResourceBundleMessageSource().apply {
        setBasenames("messages")
        setDefaultEncoding("UTF-8")
        setFallbackToSystemLocale(false)
    }

    @Bean
    fun localeResolver(): LocaleResolver = AcceptHeaderLocaleResolver().apply {
        setDefaultLocale(Locale.ENGLISH)
    }

    @Bean
    fun addAcceptLanguageHeader(): OpenApiCustomiser = OpenApiCustomiser { openApi ->
        openApi.paths.values.forEach { path ->
            path.readOperations().forEach { op ->
                val lang = Parameter()
                    .name("Accept-Language")
                    .in("header")
                    .description("Locale preference (e.g. en, ru)")
                    .required(false)
                    .schema(StringSchema())
                op.addParametersItem(lang)
            }
        }
    }
}
```

**messages.properties (пример ключей)**

```
problem.validation.title=Validation failed
problem.validation.detail=Request contains invalid fields
```

---

## Линки и расширения: `links`, `x-*` для нестандартных метаданных

Линки (`links`) в OpenAPI позволяют связать ответы и последующие операции: например, после `POST /orders` можно сразу указать ссылку на `GET /orders/{id}` с подстановкой идентификатора из тела ответа. Это делает документацию «живой», а навигацию — естественной, в духе HATEOAS.

В аннотациях это поддерживается через `@Link` и `@LinkParameter`. Мы можем описать, что поле `id` из ответа маппится в `id` пути следующего метода. Swagger UI отобразит эту связь, а некоторые генераторы клиентов — используют для построения вспомогательных вызовов.

Расширения (`x-*`) — способ добавить не-стандартные метаданные к операциям, схемам и т. д. Частые кейсы: `x-rate-limit`, `x-error-codes`, `x-idempotency-hint`. Они не влияют на валидность OpenAPI, но полезны для порталов/плагинов, которые «понимают» ваши расширения.

Линки особенно хороши в асинхронных сценариях. Возвращая `202 Accepted`, можно положить `links` на ресурс статуса и на итоговый ресурс. Документация станет проводником для клиента: «сначала сюда, потом — туда».

Не путайте линки OpenAPI с гипермедиа в самих ответах. Вы можете не возвращать HAL/JSON:API в рантайме, но в документации всё равно описать связи для удобства человека. Однако если вы действительно используете HATEOAS в ответах, имеет смысл показать это и в примерах.

Расширения стоит стандартизовать. Если вы вводите `x-error-code`, договоритесь о формате (например, SCREAMING_SNAKE_CASE) и поддерживайте каталог кодов в отдельном документе. Тогда по одному полю клиент понимает класс ошибки, не читая `detail`.

При добавлении `x-*` думайте о потребителях: UI их почти не показывает, но порталы/линтеры могут использовать. Не кладите туда чувствительные данные, и не дублируйте то, что уже есть стандартным способом.

Старайтесь не злоупотреблять линками. Это полезная «подсказка», но если все операции связаны со всеми, документация превратится в паутину. Выделяйте только реально полезные переходы, которые повторяет большинство клиентов.

Поддерживайте примеры, соответствующие линкам. Если вы пишете, что `id` идёт из тела, покажите это в `example`. Иначе пользователь запутается: теоретически — одно, на практике — другое.

Наконец, тестируйте аннотации линков: при переименовании методов и `operationId` легко «сломать» ссылку. CI-линтеры помогают, но банальный ревью YAML — тоже мощный инструмент.

**Java — POST с link на GET и расширение `x-idempotency`**

```java
package hateoas.links;

import io.swagger.v3.oas.annotations.*;
import io.swagger.v3.oas.annotations.extensions.*;
import io.swagger.v3.oas.annotations.links.*;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
@Tag(name = "Orders")
public class OrderController {

    @PostMapping
    @Operation(
        operationId = "createOrder",
        summary = "Создать заказ",
        description = "Идемпотентно по заголовку X-Idempotency-Key",
        responses = {
            @ApiResponse(responseCode = "201", description = "Создано",
                content = @Content(mediaType = "application/json", schema = @Schema(implementation = Order.class)),
                links = {
                    @Link(name = "getOrder", operationId = "getOrder",
                          parameters = @LinkParameter(name = "id", expression = "$response.body#/id"))
                })
        },
        extensions = @Extension(name = "x-idempotency", properties = @ExtensionProperty(name = "header", value = "X-Idempotency-Key"))
    )
    public ResponseEntity<Order> create(@RequestBody CreateOrder body) {
        return ResponseEntity.status(201).body(new Order("o_1", body.title));
        }

    @GetMapping("/{id}")
    @Operation(operationId = "getOrder", summary = "Получить заказ")
    @ApiResponse(responseCode = "200", description = "OK",
        content = @Content(mediaType = "application/json", schema = @Schema(implementation = Order.class)))
    public Order get(@PathVariable String id) { return new Order(id, "Sample"); }

    public record CreateOrder(String title) {}
    public record Order(String id, String title) {}
}
```

**Kotlin — эквивалент**

```kotlin
package hateoas.links

import io.swagger.v3.oas.annotations.*
import io.swagger.v3.oas.annotations.extensions.Extension
import io.swagger.v3.oas.annotations.extensions.ExtensionProperty
import io.swagger.v3.oas.annotations.links.Link
import io.swagger.v3.oas.annotations.links.LinkParameter
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
@Tag(name = "Orders")
class OrderController {

    @PostMapping
    @Operation(
        operationId = "createOrder",
        summary = "Создать заказ",
        description = "Идемпотентно по заголовку X-Idempotency-Key",
        responses = [
            ApiResponse(
                responseCode = "201",
                description = "Создано",
                content = [Content(mediaType = "application/json", schema = Schema(implementation = Order::class))],
                links = [Link(
                    name = "getOrder",
                    operationId = "getOrder",
                    parameters = [LinkParameter(name = "id", expression = "\$response.body#/id")]
                )]
            )
        ],
        extensions = [Extension(name = "x-idempotency", properties = [ExtensionProperty(name = "header", value = "X-Idempotency-Key")])]
    )
    fun create(@RequestBody body: CreateOrder): ResponseEntity<Order> =
        ResponseEntity.status(201).body(Order("o_1", body.title))

    @GetMapping("/{id}")
    @Operation(operationId = "getOrder", summary = "Получить заказ")
    @ApiResponse(
        responseCode = "200",
        description = "OK",
        content = [Content(mediaType = "application/json", schema = Schema(implementation = Order::class))]
    )
    fun get(@PathVariable id: String) = Order(id, "Sample")

    data class CreateOrder(val title: String)
    data class Order(val id: String, val title: String)
}
```

**Gradle — без дополнительных зависимостей**
(см. общий блок зависимостей выше)

---

## Документация пагинации/сортировки (`Page`, `Link` заголовки)

Пагинация — зона, где контракты часто «плывут». Spring Data `Pageable` удобно для сервера, но для клиента важнее стабильный контракт: как задать страницу и размер, как сортировать, где смотреть «следующую» страницу. Лучше явно задокументировать query-параметры и форму ответа, а также заголовки `Link` и `X-Total-Count`.

Есть два подхода: «параметры в query» (`page`, `size`, `sort`) и «курсорная пагинация» (`cursor`, `limit`). Первый проще и привычнее, второй — устойчивее на больших данных. В любом случае контракт должен быть однозначным, а в документации — примеры запросов и ответов.

Если вы используете `Pageable`, помечайте параметр `@ParameterObject` и документируйте формат `sort` (`field,asc|desc`). Пример: `?page=2&size=50&sort=createdAt,desc&sort=id,asc`. Это оптимально для UI и популярных клиентов.

Ответ лучше завернуть в собственный `PageResult<T>` с `items`, `page`, `size`, `total`, чтобы не зависеть от внутренних деталей Spring. Так вы контролируете контракт и облегчаете генерацию клиентов. Схему сделайте переиспользуемой (`components.schemas.PageResult`).

Заголовок `Link` полезен для навигации: `rel="next"`, `rel="prev"`, `rel="first"`, `rel="last"`. В OpenAPI его можно описать в `@ApiResponse(headers=...)`. Клиентам не придётся строить URL, если вы даёте готовые ссылки.

`X-Total-Count` — практично для UI, но на больших таблицах дорогая операция. Если вы не можете гарантировать быстрое вычисление total, опишите это честно и дайте флаг `partialTotal=true` в ответе.

Сортировка — часть контракта. Ограничьте разрешённые поля и документируйте их список. Иначе клиенты начнут передавать «что угодно», а вы будете удивляться SQL-ошибкам. Минимум — список разрешённых полей и поведение при невалидном значении.

Не забывайте про стабильность. Добавление нового поля сортировки совместимо, а смена дефолтной сортировки — уже риск. В changelog явно указывайте такие изменения и помечайте версии в `info.version`.

Если вы используете курсоры (base64 state), подробно опишите их жизненный цикл: срок жизни, связь с фильтрами, поведение при изменении underlying-данных. Это избавит от «двоения» в выборках и непредсказуемостей.

Наконец, приведите рабочие примеры `curl` и ответов, включая заголовки. Разработчики часто копируют их буквально; сделайте так, чтобы они работали сразу.

**Java — пагинация с `Pageable` и `Link` заголовком**

```java
package paging.docs;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.headers.Header;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springdoc.core.annotations.ParameterObject;
import org.springframework.data.domain.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/users")
public class UsersController {

    @GetMapping
    @Operation(summary = "Список пользователей", description = "Поддерживаются параметры page/size/sort, сорт по полям: id, createdAt.")
    @ApiResponse(responseCode = "200", description = "OK",
        headers = {
            @Header(name = "Link", description = "Навигационные ссылки RFC5988", schema = @Schema(type = "string")),
            @Header(name = "X-Total-Count", description = "Общее количество элементов", schema = @Schema(type = "integer"))
        },
        content = @Content(mediaType = "application/json",
            schema = @Schema(implementation = PageResult.class)))
    public ResponseEntity<PageResult<User>> list(@ParameterObject Pageable pageable) {
        var items = List.of(new User(1L, "Alice"), new User(2L, "Bob"));
        var page = new PageImpl<>(items, PageRequest.of(pageable.getPageNumber(), pageable.getPageSize()), 100);
        var body = PageResult.from(page);
        var next = URI.create("/api/users?page=" + (page.getNumber()+1) + "&size=" + page.getSize());
        return ResponseEntity.ok()
            .header("Link", "<" + next + ">; rel=\"next\"")
            .header("X-Total-Count", String.valueOf(page.getTotalElements()))
            .body(body);
    }

    public record User(Long id, String name) {}

    @Schema(name = "PageResultUser", description = "Страница результатов")
    public static class PageResult<T> {
        @ArraySchema(schema = @Schema(implementation = User.class))
        public List<T> items;
        public int page;
        public int size;
        public long total;

        public static <T> PageResult<T> from(Page<T> p) {
            var r = new PageResult<T>();
            r.items = p.getContent();
            r.page = p.getNumber();
            r.size = p.getSize();
            r.total = p.getTotalElements();
            return r;
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package paging.docs

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.headers.Header
import io.swagger.v3.oas.annotations.media.ArraySchema
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springdoc.core.annotations.ParameterObject
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageImpl
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Pageable
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.net.URI

@RestController
@RequestMapping("/api/users")
class UsersController {

    @GetMapping
    @Operation(summary = "Список пользователей", description = "Поддерживаются параметры page/size/sort, сорт по полям: id, createdAt.")
    @ApiResponse(
        responseCode = "200",
        description = "OK",
        headers = [
            Header(name = "Link", description = "Навигационные ссылки RFC5988", schema = Schema(type = "string")),
            Header(name = "X-Total-Count", description = "Общее количество элементов", schema = Schema(type = "integer"))
        ]
    )
    fun list(@ParameterObject pageable: Pageable): ResponseEntity<PageResult<User>> {
        val items = listOf(User(1, "Alice"), User(2, "Bob"))
        val page: Page<User> = PageImpl(items, PageRequest.of(pageable.pageNumber, pageable.pageSize), 100)
        val body = PageResult.from(page)
        val next = URI.create("/api/users?page=${page.number + 1}&size=${page.size}")
        return ResponseEntity.ok()
            .header("Link", "<$next>; rel=\"next\"")
            .header("X-Total-Count", page.totalElements.toString())
            .body(body)
    }

    data class User(val id: Long, val name: String)

    @Schema(name = "PageResultUser", description = "Страница результатов")
    class PageResult<T> {
        @field:ArraySchema(schema = Schema(implementation = User::class))
        var items: List<T> = emptyList()
        var page: Int = 0
        var size: Int = 0
        var total: Long = 0

        companion object {
            fun <T> from(p: Page<T>) = PageResult<T>().apply {
                items = p.content; page = p.number; size = p.size; total = p.totalElements
            }
        }
    }
}
```

**Gradle (оба DSL)**
(достаточно `spring-boot-starter-web` и `springdoc-openapi-starter-webmvc-ui`)

---

## Версионирование error-контракта и обратная совместимость

Ошибки — такой же контракт, как и успешные ответы. Менять их бездумно опасно: клиенты завязаны на коды и поля. Введите «версию» error-контракта, например через стабильный `type` и (опционально) поле `version` в расширениях. Тогда вы сможете выпускать «v2» с новыми полями, не ломая существующих потребителей.

Первый принцип: не удаляйте и не переименовывайте поля в «минорных» релизах. Добавление новых полей — совместимо; клиенты их игнорируют. Удаление — ломающее изменение (breaking), требующее договорённости о переходе и, возможно, отдельного эндпоинта/версии API.

Поле `type` — идеальный «носитель» версии. Например, `urn:problem:validation-error:v1` и `urn:problem:validation-error:v2`. Это не мешает локализации `title/detail`, но позволяет программно различать поколения ошибок и мигрировать обработчики постепенно.

Если вы хотите оставить `type` стабильным, добавьте расширение `x-problem-version=2` и документируйте его. Это хуже, чем инкапсулировать версию в `type`, но тоже работает и не ломает старые клиенты, которые ожидают прежние URI в `type`.

Версионирование должно быть отражено в OpenAPI. Опишите компоненты `ProblemV1` и `ProblemV2` (или один `Problem` с опциональными полями) и привяжите их к статусам через глобальные ответы. В идеале — отдельные группы спецификаций для v1/v2 API.

Переход планируйте заранее. Пометьте `deprecated` в описаниях старых ошибок, добавьте sunset-даты и план миграции. Коммуникация важна: даже идеальная документация не поможет, если клиенты «не слышали» об изменениях.

Для валидационных ошибок заведите стабильный формат `violations` с полями `field`, `code`, `message`. Добавление новых полей внутри элемента — допустимо; переименование — ломает клиентов. Защитите это линтерами спецификации.

В CI добавьте дифф-валидатор спецификаций (даже простые snapshot’ы), который помечает ломающие изменения. Для ошибок это особенно ценно: обычно изменения в успешных ответах виднее, а «ошибки» забывают.

Наконец, автоматизируйте связь между кодом и спецификацией: глобальный маппер возвращает ровно те поля, что описаны в актуальной версии схемы, а OpenApiCustomiser гарантирует, что все операции с `4xx/5xx` ссылаются на правильную схему.

**Java — компоненты Problem v1/v2 и привязка в глобальных ответах**

```java
package errors.versioning;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.media.*;
import io.swagger.v3.oas.models.responses.ApiResponse;
import org.springdoc.core.customizers.OpenApiCustomiser;
import org.springframework.context.annotation.*;

@Configuration
public class ErrorVersioningConfig {

    @Bean
    OpenApiCustomiser problemSchemasAndResponses() {
        return (OpenAPI oas) -> {
            var components = oas.getComponents();
            if (components.getSchemas() == null) components.setSchemas(new java.util.LinkedHashMap<>());

            var problemV1 = new ObjectSchema()
                    .addProperty("type", new StringSchema())
                    .addProperty("title", new StringSchema())
                    .addProperty("status", new IntegerSchema())
                    .addProperty("detail", new StringSchema())
                    .addProperty("instance", new StringSchema())
                    .addProperty("traceId", new StringSchema())
                    .addProperty("errorCode", new StringSchema());
            components.addSchemas("ProblemV1", problemV1);

            var problemV2 = new ObjectSchema()
                    .addProperty("type", new StringSchema()._default("urn:problem:unknown:v2"))
                    .addProperty("title", new StringSchema())
                    .addProperty("status", new IntegerSchema())
                    .addProperty("detail", new StringSchema())
                    .addProperty("instance", new StringSchema())
                    .addProperty("traceId", new StringSchema())
                    .addProperty("errorCode", new StringSchema())
                    .addProperty("timestamp", new StringSchema().format("date-time")); // новое поле
            components.addSchemas("ProblemV2", problemV2);

            var v2Content = new Content().addMediaType("application/problem+json",
                    new MediaType().schema(new Schema<>().$ref("#/components/schemas/ProblemV2")));
            var v2 = new ApiResponse().description("Problem Details v2").content(v2Content);

            oas.getPaths().values().forEach(p -> p.readOperations().forEach(op -> {
                op.getResponses().put("400", v2);
                op.getResponses().put("500", v2);
            }));
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package errors.versioning

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.media.*
import io.swagger.v3.oas.models.responses.ApiResponse
import org.springdoc.core.customizers.OpenApiCustomiser
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class ErrorVersioningConfig {

    @Bean
    fun problemSchemasAndResponses(): OpenApiCustomiser = OpenApiCustomiser { oas: OpenAPI ->
        val components = oas.components
        if (components.schemas == null) components.schemas = linkedMapOf()

        val problemV1 = ObjectSchema()
            .addProperty("type", StringSchema())
            .addProperty("title", StringSchema())
            .addProperty("status", IntegerSchema())
            .addProperty("detail", StringSchema())
            .addProperty("instance", StringSchema())
            .addProperty("traceId", StringSchema())
            .addProperty("errorCode", StringSchema())
        components.addSchemas("ProblemV1", problemV1)

        val problemV2 = ObjectSchema()
            .addProperty("type", StringSchema()._default("urn:problem:unknown:v2"))
            .addProperty("title", StringSchema())
            .addProperty("status", IntegerSchema())
            .addProperty("detail", StringSchema())
            .addProperty("instance", StringSchema())
            .addProperty("traceId", StringSchema())
            .addProperty("errorCode", StringSchema())
            .addProperty("timestamp", StringSchema().format("date-time"))
        components.addSchemas("ProblemV2", problemV2)

        val v2Content = Content().addMediaType("application/problem+json", MediaType().schema(Schema<Any>().`$ref`("#/components/schemas/ProblemV2")))
        val v2 = ApiResponse().description("Problem Details v2").content(v2Content)

        oas.paths.values.forEach { path ->
            path.readOperations().forEach { op ->
                op.responses["400"] = v2
                op.responses["500"] = v2
            }
        }
    }
}
```

**application.yml — метаданные версии контракта**

```yaml
springdoc:
  api-docs.path: /v3/api-docs
  swagger-ui.path: /docs
info:
  app:
    error-contract-version: v2
```

# 7. Безопасность и авторизация

## `@SecurityScheme` (HTTP bearer/JWT, API-ключ, OAuth2 flows, OpenID Connect)

Первое, что важно усвоить: аннотации OpenAPI, такие как `@SecurityScheme`, **не добавляют безопасность в рантайм** — они только документируют, *какая* аутентификация/авторизация ожидается от клиента. Реальная защита реализуется Spring Security, а `@SecurityScheme` делает её видимой в Swagger UI/Redoc и понятной для генераторов SDK. Поэтому мы всегда думаем в двух плоскостях: «как будет реально проверяться доступ» и «как это покажем в контракте».

Для современных REST API базовый вариант — `HTTP bearer` с JWT. В спецификации это документируется как `type=HTTP`, `scheme=bearer`, `bearerFormat=JWT`. UI тогда отображает кнопку **Authorize**, запрашивает токен и автоматически подставляет `Authorization: Bearer ...` ко всем защищённым вызовам.

Параллельно встречается API Key — простой ключ в заголовке или query. Это **не шифрует** трафик и не даёт тонкой авторизации, но может быть уместно для внутренних сервисов, вебхуков или временных интеграций. В OpenAPI документируется как `type=APIKEY` и атрибуты `in=header|query|cookie`, `paramName`.

OAuth2 — это не «другой тип токена», а протокол *получения* токена. В контракте описывают «потоки» (`authorizationCode`, `clientCredentials` и т. д.) с URL-ами авторизации и токен-эндпоинта, а также *scopes*. Swagger UI умеет проходить Authorization Code (включая PKCE), что удобно для ручного тестирования из браузера.

OpenID Connect — надстройка над OAuth2 с discovery-документом `/.well-known/openid-configuration`. В OpenAPI можно указать `type=OPENIDCONNECT` + `openIdConnectUrl`. Это особенно удобно, если ваш IdP (Keycloak/Auth0/…) поддерживает discovery: UI сам «поймёт», куда ходить за токенами и ключами.

Часто в одном сервисе нужны **несколько схем** аутентификации сразу: например, публичные вебхуки по API Key и приватные REST по JWT. В контракте это нормально: объявляем две `@SecurityScheme` с разными именами, а затем указываем `@SecurityRequirement` там, где нужно (см. следующий пункт).

К именам схем относитесь как к API: они становятся «ключами» в YAML (`components.securitySchemes`). Стабильное имя (`bearer-jwt`, `api_key`, `oauth2`) удобно для клиентов и для ваших тестов, где вы валидируете наличие нужных требований.

Ещё нюанс — формат токена в UI. `bearerFormat=JWT` — подсказка человеку, а не машине. Генераторы не валидируют JWT, но разработчик понимает, что токен можно декодировать, а поля `scope`/`roles` попадут в `SecurityContext` сервера (если вы это настроите).

Наконец, держите спецификацию в фокусе безопасности: закрывайте `/v3/api-docs` и `/docs` авторизацией в проде, не публикуйте тестовые секреты, не вводите «фиктивные» схемы ради красоты UI. Документация должна отражать то, что реально работает.

Последняя практическая рекомендация: для OAuth2/Bearer добавляйте короткое описание того, где получить токен, в `description` схемы. Это экономит часы у потребителей и снижает «шум» в каналах поддержки.

**Gradle (Groovy/Kotlin) — добавляем безопасность и springdoc**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-security"
    implementation "org.springframework.boot:spring-boot-starter-oauth2-resource-server"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
    testImplementation "org.springframework.security:spring-security-test"
}
```

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
    testImplementation("org.springframework.security:spring-security-test")
}
```

**Java — объявляем несколько схем безопасности**

```java
package security.oas;

import io.swagger.v3.oas.annotations.enums.SecuritySchemeIn;
import io.swagger.v3.oas.annotations.enums.SecuritySchemeType;
import io.swagger.v3.oas.annotations.security.OAuthFlow;
import io.swagger.v3.oas.annotations.security.OAuthFlows;
import io.swagger.v3.oas.annotations.security.OAuthScope;
import io.swagger.v3.oas.annotations.security.SecurityScheme;
import org.springframework.context.annotation.Configuration;

@Configuration
@SecurityScheme(
    name = "bearer-jwt",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT",
    description = "Bearer JWT. Получите токен у вашего Identity Provider."
)
@SecurityScheme(
    name = "api_key",
    type = SecuritySchemeType.APIKEY,
    in = SecuritySchemeIn.HEADER,
    paramName = "X-API-Key",
    description = "Простой ключ доступа для вебхуков/интеграций."
)
@SecurityScheme(
    name = "oauth2",
    type = SecuritySchemeType.OAUTH2,
    flows = @OAuthFlows(
        authorizationCode = @OAuthFlow(
            authorizationUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/auth",
            tokenUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/token",
            scopes = {
                @OAuthScope(name = "orders.read", description = "Чтение заказов"),
                @OAuthScope(name = "orders.write", description = "Создание/изменение заказов")
            }
        ),
        clientCredentials = @OAuthFlow(
            tokenUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/token",
            scopes = {
                @OAuthScope(name = "service.read", description = "Технический доступ чтения")
            }
        )
    )
)
@SecurityScheme(
    name = "openid",
    type = SecuritySchemeType.OPENIDCONNECT,
    openIdConnectUrl = "https://auth.example.com/realms/acme/.well-known/openid-configuration",
    description = "OpenID Connect discovery"
)
public class OpenApiSecuritySchemes { }
```

**Kotlin — эквивалент**

```kotlin
package security.oas

import io.swagger.v3.oas.annotations.enums.SecuritySchemeIn
import io.swagger.v3.oas.annotations.enums.SecuritySchemeType
import io.swagger.v3.oas.annotations.security.*
import org.springframework.context.annotation.Configuration

@Configuration
@SecurityScheme(
    name = "bearer-jwt",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT",
    description = "Bearer JWT. Получите токен у вашего Identity Provider."
)
@SecurityScheme(
    name = "api_key",
    type = SecuritySchemeType.APIKEY,
    `in` = SecuritySchemeIn.HEADER,
    paramName = "X-API-Key",
    description = "Простой ключ доступа для вебхуков/интеграций."
)
@SecurityScheme(
    name = "oauth2",
    type = SecuritySchemeType.OAUTH2,
    flows = OAuthFlows(
        authorizationCode = OAuthFlow(
            authorizationUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/auth",
            tokenUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/token",
            scopes = [OAuthScope(name = "orders.read", description = "Чтение заказов"),
                      OAuthScope(name = "orders.write", description = "Создание/изменение заказов")]
        ),
        clientCredentials = OAuthFlow(
            tokenUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/token",
            scopes = [OAuthScope(name = "service.read", description = "Технический доступ чтения")]
        )
    )
)
@SecurityScheme(
    name = "openid",
    type = SecuritySchemeType.OPENIDCONNECT,
    openIdConnectUrl = "https://auth.example.com/realms/acme/.well-known/openid-configuration",
    description = "OpenID Connect discovery"
)
class OpenApiSecuritySchemes
```

---

## `@SecurityRequirement` на уровне контроллера/операции

`@SecurityRequirement` — это «привязка» объявленных схем безопасности к конкретным REST-методам. Здесь мы говорим: «чтобы вызвать этот метод, клиент должен удовлетворять схеме `bearer-jwt` (или `oauth2` с конкретными `scopes`)». В Swagger UI это отображается как замочек, а при клике на **Authorize** UI будет применять подходящий токен.

Размещать требование можно на уровне класса (для всего контроллера) и/или на уровне метода (точечно). На практике удобно вешать базовое требование на класс, а исключения (например, публичные `GET` или health-check) помечать без требований с помощью `@io.swagger.v3.oas.annotations.security.SecurityRequirements({})`, тем самым «обнуляя» наследование.

Если операция допускает **несколько** способов аутентификации («или JWT, или API Key»), мы указываем несколько требований. В OpenAPI это трактуется как «логическое ИЛИ между объектами требований, и логическое И между скоупами внутри одного требования». То есть либо bearer, либо api_key; но если выбрали oauth2 — то все указанные scopes должны присутствовать.

Важно синхронизировать `@SecurityRequirement` с реальной конфигурацией Spring Security. Если в коде метод открыт аннотацией `@PermitAll`, а в контракте указан `bearer-jwt`, вы получите «ложный негатив» в UI и путаницу у клиентов. Правило: **документация должна соответствовать рантайму**, а не «как хочется».

Внутри `@SecurityRequirement` для OAuth2 можно перечислить скоупы (строки), и это станет подсказкой для UI: какой набор scopes запросить у IdP в момент авторизации. Генераторы клиентов также учитывают этот список при сборке пермишенов.

Не забудьте про **внутренние** маршруты: админские/отладочные методы нужно либо исключать из спецификации, либо корректно помечать отдельной схемой (например, Basic для локальной админки) и не публиковать наружу. Здесь хороша группировка `GroupedOpenApi` и разные URL спецификаций.

Если у вас API Key для вебхуков, не размазывайте его по всему контроллеру. Лучше явно пометить только тот метод, который действительно его принимает (`@SecurityRequirement(name="api_key")`), а остальные — оставить на JWT/OAuth2.

В многоарендных системах можно использовать несколько требований одновременно: «JWT + API Key партнёра». В OpenAPI это два требования в **одном** массиве (логическое И). В UI это выглядит сложнее, но контракты становятся честнее, а клиенты не пройдут без обеих частей.

И последнее: если операция действительно публичная (не требует аутентификации), это тоже стоит явно показать — отсутствие `@SecurityRequirement` на неё будет сигналом, что токен не нужен. В больших спецификациях дублируйте это словами в `description`, чтобы снять сомнения.

**Java — контроллер с требованиями безопасности**

```java
package security.requirement;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.security.SecurityRequirements;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
@Tag(name = "Orders")
@SecurityRequirement(name = "bearer-jwt") // по умолчанию весь контроллер требует JWT
public class OrdersController {

    @GetMapping("/{id}")
    @Operation(summary = "Получить заказ")
    public String get(@PathVariable String id) { return "order:" + id; }

    @PostMapping
    @Operation(summary = "Создать заказ (OAuth2 scopes)")
    @SecurityRequirement(name = "oauth2", scopes = {"orders.write"})
    public String create(@RequestBody String body) { return "created"; }

    @PostMapping("/webhook")
    @Operation(summary = "Webhook (API Key)")
    @SecurityRequirements({ @SecurityRequirement(name = "api_key") })
    public void webhook(@RequestBody String payload) { /* ... */ }

    @GetMapping("/public/info")
    @Operation(summary = "Публичная информация")
    @SecurityRequirements({}) // явное отсутствие требований
    public String publicInfo() { return "ok"; }
}
```

**Kotlin — эквивалент**

```kotlin
package security.requirement

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.security.SecurityRequirement
import io.swagger.v3.oas.annotations.security.SecurityRequirements
import io.swagger.v3.oas.annotations.tags.Tag
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
@Tag(name = "Orders")
@SecurityRequirement(name = "bearer-jwt")
class OrdersController {

    @GetMapping("/{id}")
    @Operation(summary = "Получить заказ")
    fun get(@PathVariable id: String): String = "order:$id"

    @PostMapping
    @Operation(summary = "Создать заказ (OAuth2 scopes)")
    @SecurityRequirement(name = "oauth2", scopes = ["orders.write"])
    fun create(@RequestBody body: String): String = "created"

    @PostMapping("/webhook")
    @Operation(summary = "Webhook (API Key)")
    @SecurityRequirements(SecurityRequirement(name = "api_key"))
    fun webhook(@RequestBody payload: String) { }

    @GetMapping("/public/info")
    @Operation(summary = "Публичная информация")
    @SecurityRequirements
    fun publicInfo(): String = "ok"
}
```

---

## Описание scopes и ролей, связь с Spring Security

Scopes в OAuth2 — это **разрешения**, которые клиент запрашивает у IdP. На стороне Spring Security из коробки они превращаются в `GrantedAuthority` с префиксом `SCOPE_`. Например, `orders.read` в токене станет `SCOPE_orders.read`, и выражение доступа `hasAuthority("SCOPE_orders.read")` начнёт работать без дополнительной магии — если вы включили Resource Server (JWT).

Роли (`ROLE_*`) — исторически другой слой, часто «локальные роли приложения». Их можно продолжать использовать, но старайтесь не смешивать: либо вы опираетесь на scopes как на единицу разрешения (рекомендовано для API), либо чётко маппите внешние роли в локальные. В любом случае **одинаковая терминология** в коде и в OAS обязательна, иначе команда начнёт путаться.

Для Keycloak/Okta/… часто требуется поменять, **из какого claim** брать разрешения: `scope`, `scp`, `permissions`, `realm_access.roles`, `resource_access.{client}.roles`. Это настраивается конвертером `JwtGrantedAuthoritiesConverter`: задаём `authoritiesClaimName` и `authorityPrefix`. Если у вас роли без точек, можно префиксировать `ROLE_` и использовать `hasRole("ADMIN")`.

Хорошая практика — **минимизировать логику на контроллерах** и использовать выражения доступа в Security DSL: `authorizeHttpRequests().requestMatchers("/api/orders/**").hasAuthority("SCOPE_orders.read")`. Так требования «меньше дублируются» между аннотациями `@SecurityRequirement` и кодом авторизации.

В OAS можно указать, какие scopes нужны операции: `@SecurityRequirement(name = "oauth2", scopes = {"orders.write"})`. Это станет подсказкой для кнопки **Authorize**: UI запросит именно эти scopes. Внимание: список в контракте — не «истина в последней инстанции». Истина — в `SecurityFilterChain`. Держите их синхронными (ревью/линтинг в CI).

Не переусердствуйте с количеством scopes. Десятки разрешений усложняют жизнь и вам, и клиентам. Часто хватает 5–10 «грубых» scopes: чтение/запись доменных сущностей. Точное разделение на поле-права лучше делать на уровне бизнес-логики и собственных политик, а не превращать scopes в «матрицу из сотен флагов».

Подумайте о **совместимости**: добавление нового scope — совместимо; ужесточение (теперь метод требует больше) — потенциально ломающий шаг. Вводите переходный период: сначала разрешайте оба набора, документируйте депрекацию, готовьте клиентов.

Для machine-to-machine лучше подходит Client Credentials с сервисным набором scopes. Для браузерного UI — Authorization Code + PKCE. Разные клиенты — разные наборы разрешений, и это нормально. Документируйте их в соответствующих группах/профилях.

В JWT не надо держать «горы» данных. Чем меньше и стабильнее claims, тем проще кэшировать ключи/валидировать токены и тем меньше сюрпризов в парсерах. Если у вас сложная атрибутика — отдавайте её из вашего API по защищённому вызову, а не зашивайте в токен.

И ещё: делайте **явные ошибки**. Если scope не хватает — `403 Forbidden` с понятным `type` (`urn:problem:forbidden`) и списком недостающих scopes в расширениях. Это резко ускорит диагностику у клиентов.

**application.yml — ресурс-сервер JWT**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/acme
```

**Java — маппинг scopes/ролей и правила доступа**

```java
package security.jwt;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authorization.AuthorizationDecision;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(reg -> reg
                .requestMatchers("/docs/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/api/orders/**").hasAuthority("SCOPE_orders.read")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth -> oauth.jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter())))
            .csrf(csrf -> csrf.disable())
            .build();
    }

    private JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter();
        scopes.setAuthorityPrefix("SCOPE_");
        scopes.setAuthoritiesClaimName("scope"); // или "scp" / "permissions"

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter((Jwt jwt) -> {
            Collection<GrantedAuthority> authorities = scopes.convert(jwt);
            // Пример: добавить роли из кастомного claim:
            // authorities.add(new SimpleGrantedAuthority("ROLE_ADMIN"));
            return authorities;
        });
        return converter;
    }

    @Bean
    JwtDecoder jwtDecoder() {
        // Обычно не нужен, issuer-uri достаточно; добавлено как заглушка примера
        return JwtDecoders.fromIssuerLocation("https://auth.example.com/realms/acme");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package security.jwt

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.jwt.JwtDecoder
import org.springframework.security.oauth2.jwt.JwtDecoders
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecurityConfig {

    @Bean
    fun api(http: HttpSecurity): SecurityFilterChain =
        http
            .authorizeHttpRequests {
                it.requestMatchers("/docs/**", "/v3/api-docs/**").permitAll()
                    .requestMatchers("/api/orders/**").hasAuthority("SCOPE_orders.read")
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { res ->
                res.jwt { jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()) }
            }
            .csrf { it.disable() }
            .build()

    private fun jwtAuthConverter(): JwtAuthenticationConverter {
        val scopes = JwtGrantedAuthoritiesConverter().apply {
            setAuthorityPrefix("SCOPE_")
            setAuthoritiesClaimName("scope")
        }
        return JwtAuthenticationConverter().apply {
            setJwtGrantedAuthoritiesConverter { jwt: Jwt ->
                val authorities: Collection<GrantedAuthority> = scopes.convert(jwt)
                authorities
            }
        }
    }

    @Bean
    fun jwtDecoder(): JwtDecoder = JwtDecoders.fromIssuerLocation("https://auth.example.com/realms/acme")
}
```

---

## Документация процесса аутентификации в UI (Authorize, PKCE, device code)

Swagger UI способен проходить **Authorization Code** (включая PKCE) и **Client Credentials**. Для этого в OAS должны быть корректно описаны `authorizationUrl`, `tokenUrl` и `scopes` (мы сделали это в `@SecurityScheme(name="oauth2")`). Дальше UI покажет кнопку **Authorize**, предложит выбрать схему и прокредитовать клиента.

Для **PKCE** в браузерных сценариях не нужен клиентский секрет. На стороне Swagger UI включаем флаг `usePkceWithAuthorizationCodeGrant=true`, и UI будет генерировать `code_verifier`/`code_challenge`, подменяя обычный «секрет». Это безболезненно для серверной стороны, которая уже умеет в PKCE.

Сервисные сценарии (**Client Credentials**) полезны для бэкенд-бэкенд интеграций. Swagger UI тоже умеет их, но помните: это **секрет**. В проде UI обычно закрывают, и секреты туда не кладут. Для дев-стендов секреты можно прокинуть через переменные окружения/локальные конфиги и запретить индексирование UI.

**Device Code** — отдельный поток OAuth2, удобный для «умных телевизоров» и CLI. В OpenAPI 3.0/3.1 для него нет «родного» типа flow, поэтому документируйте процесс текстом (в описании схемы/операции) и, при желании, используйте `x-*` расширения. UI его как «кнопку» не поддержит, но разработчик поймёт последовательность шагов.

Если ваш IdP поддерживает **OpenID Connect Discovery**, достаточно `openIdConnectUrl` — UI подтянет метаданные (ендпоинты, сигнатуры, scopes). Это снижает вероятность ошибок при ручном вводе URL и сильно ускоряет настройку.

При описании **scopes** дайте ясные тексты: «`orders.read` — чтение заказов; `orders.write` — создание/изменение». Эти описания UI показывает в модальном окне, и разработчик быстрее выбирает корректный минимальный набор вместо «галка на всё».

Конечно, UI — не единственный потребитель. Выгружайте `/v3/api-docs` и публикуйте статический Redoc/Swagger-страницу, где остаётся кнопка Authorize. Это полезно для партнёров: не нужно доступа к вашему дев-стенду, чтобы посмотреть формальный контракт и сценарии авторизации.

Учтите **redirect URL**. Для локального UI это обычно `/swagger-ui/oauth2-redirect.html`. Пропишите его в конфигурации клиента на IdP, иначе вы увидите ошибку «redirect_uri mismatch». Для каждого стенда (dev/stage/prod) нужен свой зарегистрированный redirect.

И ещё про куки/3rd-party cookies: некоторые IdP используют IFrame для авторизации, что ломается в строгих настройках браузера. Проверьте «ручной» поток на всех целевых браузерах и добавьте советы в документацию (например, открыть авторизацию в новом окне UI умеет).

Безопасность прежде всего: даже в dev не храните секреты в репозитории. Если уж нужно — используйте переменные окружения и `application-local.yml`, который не коммитится.

**application.yml — параметры Swagger UI OAuth & PKCE**

```yaml
springdoc:
  swagger-ui:
    path: /docs
    oauth:
      client-id: demo-swagger-ui
      scopes: openid, profile, orders.read, orders.write
      use-pkce-with-authorization-code-grant: true
    oauth2-redirect-url: http://localhost:8080/docs/oauth2-redirect.html
```

**Java — добавим пояснения в схему через `description` и расширения**

```java
package security.ui;

import io.swagger.v3.oas.annotations.extensions.Extension;
import io.swagger.v3.oas.annotations.extensions.ExtensionProperty;
import io.swagger.v3.oas.annotations.security.SecurityScheme;
import io.swagger.v3.oas.annotations.security.OAuthFlows;
import io.swagger.v3.oas.annotations.security.OAuthFlow;
import io.swagger.v3.oas.annotations.security.OAuthScope;
import io.swagger.v3.oas.annotations.enums.SecuritySchemeType;
import org.springframework.context.annotation.Configuration;

@Configuration
@SecurityScheme(
    name = "oauth2-ui",
    type = SecuritySchemeType.OAUTH2,
    flows = @OAuthFlows(authorizationCode = @OAuthFlow(
        authorizationUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/auth",
        tokenUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/token",
        scopes = { @OAuthScope(name="openid"), @OAuthScope(name="orders.read"), @OAuthScope(name="orders.write") }
    )),
    description = "Авторизуйтесь через Authorization Code (PKCE включён в UI). Для Device Code см. портал.",
    extensions = @Extension(name = "x-docs", properties = @ExtensionProperty(name = "device-code-doc", value = "https://docs.example.com/device-code"))
)
public class SwaggerUiOauthConfig { }
```

**Kotlin — эквивалент**

```kotlin
package security.ui

import io.swagger.v3.oas.annotations.enums.SecuritySchemeType
import io.swagger.v3.oas.annotations.extensions.Extension
import io.swagger.v3.oas.annotations.extensions.ExtensionProperty
import io.swagger.v3.oas.annotations.security.*
import org.springframework.context.annotation.Configuration

@Configuration
@SecurityScheme(
    name = "oauth2-ui",
    type = SecuritySchemeType.OAUTH2,
    flows = OAuthFlows(
        authorizationCode = OAuthFlow(
            authorizationUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/auth",
            tokenUrl = "https://auth.example.com/realms/acme/protocol/openid-connect/token",
            scopes = [OAuthScope(name = "openid"), OAuthScope(name = "orders.read"), OAuthScope(name = "orders.write")]
        )
    ),
    description = "Авторизуйтесь через Authorization Code (PKCE включён в UI). Для Device Code см. портал.",
    extensions = [Extension(name = "x-docs", properties = [ExtensionProperty(name = "device-code-doc", value = "https://docs.example.com/device-code")])]
)
class SwaggerUiOauthConfig
```

---

## Работа с CSRF/сессионной аутентификацией в Swagger UI

Если вы используете **сессионную аутентификацию** (cookie-based) и включён CSRF, Swagger UI из коробки не знает, куда взять CSRF-токен. В Spring Security дефолтный `CookieCsrfTokenRepository` кладёт токен в куку `XSRF-TOKEN`, которую браузер видит, но UI не подставляет в заголовок `X-XSRF-TOKEN`. Нужно либо *отключить CSRF* для REST-эндпоинтов, которые дергает UI, либо настроить UI вставлять заголовок.

Отключение CSRF для JSON-REST (не для форм) — частая практика: REST-клиенты используют Bearer JWT и не уязвимы к CSRF. Тогда Swagger UI просто работает как любой HTTP-клиент. Но если у вас **сессии** и вы защищаете `/api/**` CSRF-механизмом — документируйте это и обеспечьте прокладку заголовка.

Самый чистый путь — оставить CSRF включённым и **научить Swagger UI** тянуть токен из куки и отправлять его. UI поддерживает `requestInterceptor`, к которому можно прицепиться, если вы переопределите `swagger-ui` статикой и добавите пару строк JS.

Альтернатива — **выключить CSRF только для документации** (`/docs/**`, `/v3/api-docs/**`) и оставить для «боевого» `/api/**`. Это уменьшает напильник, но не решает вопрос тестирования защищённых POST/PUT из UI, если они требуют CSRF. Тогда без перехватчика не обойтись.

При сессионной аутентификации в UI удобно включить флаг `withCredentials` (CORS), чтобы куки ходили с запросами. На стороне сервера CORS должен разрешать `credentials=true`, и заголовки `Access-Control-Allow-Origin` не могут быть `*` (должен быть конкретный origin).

Не забывайте, что CSRF-токен нужно **обновлять** после логина. Если вы кешируете UI, убедитесь, что токен попадает в свежую страницу или выдаётся отдельным эндпоинтом (часто UI сам получит его, сделав GET на корневую страницу).

Внутренним сервисам на Spring Boot проще уйти от сессий в сторону JWT/Bearer — тогда Swagger UI из коробки «просто работает». Сессии оставьте для внутренних админок, где заходят люди через браузер.

Документируйте поведение честно: в `description` схемы безопасности напишите, нужен ли CSRF-заголовок для методов изменения состояния. Люди перестанут гадать, почему «оно 403 только из UI».

Помните и про безопасность: если вы включили JS-перехватчик для подстановки токена, не допускайте XSS на странице UI — иначе токен уйдёт злоумышленнику. Держите `/docs` под строгой Content Security Policy.

И наконец, тестируйте локально в «реальном» браузере сценарии логин→авторизоваться→вызвать POST из UI. Много багов всплывает только там, где смешиваются куки, редиректы и CORS.

**Java — конфигурация CSRF и исключения для документации**

```java
package security.csrf;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class CsrfSecurityConfig {

    @Bean
    SecurityFilterChain csrfChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(reg -> reg
                .requestMatchers("/docs/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/v3/api-docs/**", "/docs/**") // UI и спецификацию не защищаем
            )
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package security.csrf

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.csrf.CookieCsrfTokenRepository

@Configuration
class CsrfSecurityConfig {

    @Bean
    fun csrfChain(http: HttpSecurity): SecurityFilterChain =
        http
            .authorizeHttpRequests {
                it.requestMatchers("/docs/**", "/v3/api-docs/**").permitAll()
                    .anyRequest().authenticated()
            }
            .csrf {
                it.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                    .ignoringRequestMatchers("/v3/api-docs/**", "/docs/**")
            }
            .build()
}
```

**`swagger-initializer.js` (ресурс, который подхватывает Swagger UI) — подставляем CSRF-заголовок**

```javascript
// Файл положите в src/main/resources/static/docs/swagger-initializer.js
window.ui = SwaggerUIBundle({
  url: "/v3/api-docs",
  dom_id: '#swagger-ui',
  requestInterceptor: (req) => {
    const m = document.cookie.match(/XSRF-TOKEN=([^;]+)/);
    if (m) req.headers['X-XSRF-TOKEN'] = decodeURIComponent(m[1]);
    return req;
  }
});
```

---

## Тестирование защищённых операций прямо из UI и в автотестах

Swagger UI — удобный способ «пощупать» защищённый эндпоинт руками: нажал **Authorize**, вставил Bearer-токен или прошёл OAuth2, вызвал метод. Это хорошо для первичной проверki, но **не заменяет** автотесты. В CI нужно автоматически проверять, что доступы настроены правильно, а контракты не «поплыли».

Для Spring Security есть `spring-security-test`, который даёт постпроцессоры `jwt()`/`oauth2Login()` для MockMvc и WebTestClient. Так вы можете программно подложить JWT со скоупами/ролями и проверить, что `/api/orders/**` действительно требует `SCOPE_orders.read` и отвечает 403 при его отсутствии.

Полезно тестировать и «отрицательные» сценарии: без токена → 401, с неправильным audience/issuer → 401, с неподходящим scope → 403. Эти тесты служат «предохранителем» от случайных регрессий в Security DSL при рефакторинге.

Если вы используете API Key для вебхуков, добавляйте тесты на отсутствие/неверный ключ и на happy-path с корректным заголовком `X-API-Key`. Это дешево и надёжно фиксирует требования.

Дальше — проверка **OAS против рантайма**. Минимум — интеграционный тест, который скачивает `/v3/api-docs` и проверяет, что у защищённых методов есть `security`-требования. Более продвинутый путь — линтеры (Spectral) и дифф-валидаторы (oas-diff) в пайплайне: они ловят несоответствия между версиями.

Для e2e можно использовать RestAssured с реальным IdP sandbox/тестового realm’а: завели клиента, получили токен по client credentials и сходили в защищённый метод. Такие тесты более хрупкие, но отлично ловят интеграционные проблемы.

Если у вас UI отключён в проде (как обычно), убедитесь, что на dev/stage он работает и сценарий авторизации — «живой». Поднимите smoke-тест, который хотя бы проверяет доступность `/docs` и `/v3/api-docs`.

Наконец, обучайте команду читать 401/403. Возвращайте `Problem Details` с понятными `type`/`detail` и, например, расширением `missing_scopes`. Так время на «почему мне 403?» уменьшается радикально.

И не забывайте: токены в логах — табу. Маскируйте `Authorization` и храните тестовые токены в безопасном хранилище/секрет-менеджере.

**Java — MockMvc: тесты JWT-доступа**

```java
package security.tests;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.http.MediaType;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest
class OrdersSecurityTest {

    @Autowired MockMvc mvc;

    @Test
    void forbiddenWithoutScope() throws Exception {
        mvc.perform(get("/api/orders/1")
                .with(jwt().jwt(jwt -> jwt.claim("scope", "profile"))))
            .andExpect(status().isForbidden());
    }

    @Test
    void okWithScope() throws Exception {
        mvc.perform(get("/api/orders/1")
                .with(jwt().jwt(jwt -> jwt.claim("scope", "orders.read"))))
            .andExpect(status().isOk());
    }

    @Test
    void unauthorizedWithoutToken() throws Exception {
        mvc.perform(get("/api/orders/1").accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isUnauthorized());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package security.tests

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get

@WebMvcTest
class OrdersSecurityTest(@Autowired val mvc: MockMvc) {

    @Test
    fun forbiddenWithoutScope() {
        mvc.get("/api/orders/1") {
            with(jwt().jwt { it.claim("scope", "profile") })
        }.andExpect { status { isForbidden() } }
    }

    @Test
    fun okWithScope() {
        mvc.get("/api/orders/1") {
            with(jwt().jwt { it.claim("scope", "orders.read") })
        }.andExpect { status { isOk() } }
    }

    @Test
    fun unauthorizedWithoutToken() {
        mvc.get("/api/orders/1").andExpect { status { isUnauthorized() } }
    }
}
```

**Gradle (оба DSL) — зависимости для тестов**

```groovy
dependencies {
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testImplementation "org.springframework.security:spring-security-test"
}
```

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

# 8. Версионирование API и мульти-спеки

## Подходы к версиям: путь (`/v1`), заголовок/медиа-тип, поддомены

Версионирование — это способ явным образом зафиксировать контракт во времени и управлять его эволюцией без слома потребителей. В контексте REST самый популярный подход — версионирование в URL: `/v1/orders`, `/v2/orders`. Клиент видит версию сразу, прокси и кеши легко различают ресурсы, а документация становится очевидной. Минус — «засорение» маршрутов и необходимость дублировать много контроллеров при крупном апгрейде.

Альтернатива — версия в заголовке, например пользовательском `X-API-Version: 1` или стандартном `Accept: application/json; version=1`. Такой подход «чище» с точки зрения URI и позволяет менять версию без смены ссылок. Однако кеши, CDN и многие инструменты хуже понимают этот вариант, а ошибку «забыли заголовок» сложно заметить глазами.

Третий путь — **vendor media type**: `Accept: application/vnd.acme.v1+json`. Это подвид заголовочного подхода, но с более формальной семантикой и отличной поддержкой средствами контент-негациации Spring. Плюсы — гибкость, минусы — сложнее «надеть очками» и увидеть версию, и немного тяжелее для людей в отладке.

Поддомены — `/orders` на `v1.api.example.com` против `v2.api.example.com`. Так часто делают крупные публичные API: инфраструктура и лимиты разделены физически, а маршруты внутри остаются прежними. Это добавляет DevOps-сложности (DNS, TLS, маршрутизация), но даёт операционную изоляцию и независимые окна релиза.

Как выбирать? Для внутренних систем и микросервисов путь — почти всегда лучший компромисс: просто, понятно, спокойно кешируется. Для публичных API с жёстким SLA поддомены дают больше контроля. Заголовки и vendor types хороши, когда критично сохранить «красивые» URL и когда команда дисциплинирована в клиентах/прокси.

Не путайте **версию API** и **версию ресурса**. Первое — формат контракта «в целом», второе — конкретное состояние объекта («ETag», «If-Match»). Эти механики дополняют друг друга: версия API управляет схемами и операциями, версия ресурса — оптимистичной блокировкой и кэшированием.

Практика с семвером: «v1» — это «линия», а внутри неё допускаются несовместимые изменения? Нет. Консервативно считайте любой breaking change новым major — `v2`. Внутри `v1` добавляйте только назад-совместимые поля/операции. Это дисциплина, но она спасает интеграции от скрытых поломок.

При заголовочном подходе важно чётко документировать «дефолт». Если клиент не прислал `X-API-Version`, какую версию вы ему дадите? Обычно — самую свежую «стабильную», но лучше возвращать 400 с Problem Details и просьбой указать версию. Такой строгий режим снимает класс недоразумений.

На границе версий обязательно вводите «дорожные знаки»: `Deprecation: true`, `Sunset: <http://…>; date="2026-01-01"` и заметный `@Operation(deprecated = true)` на методах `v1`. Это прозрачная коммуникация с потребителем и формальный повод для миграции.

Сетевые устройства и кеши. URL-версионирование дружит с Varnish/CDN, media-type — реже. Если вы планируете агрессивный кэш на границе, выбирайте путь/поддомены. Иначе CDN будет сложно настроить вариативность по `Accept`, что чревато «грязными» кешами.

**Gradle (Groovy)**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — три варианта маршрутизации: путь, заголовок, vendor media type**

```java
package versioning.pathheader;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class VersionedController {

    // Путь: /api/v1/greeting
    @GetMapping("/v1/greeting")
    public String v1Path() { return "hello v1"; }

    // Заголовок: X-API-Version=2
    @GetMapping(value = "/greeting", headers = "X-API-Version=2")
    public String v2Header() { return "hello v2"; }

    // Vendor media type: Accept: application/vnd.acme.v3+json
    public static final String V3 = "application/vnd.acme.v3+json";

    @GetMapping(value = "/greeting", produces = V3)
    public String v3Vendor() { return "hello v3"; }
}
```

**Kotlin — эквивалент**

```kotlin
package versioning.pathheader

import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api")
class VersionedController {

    @GetMapping("/v1/greeting")
    fun v1Path(): String = "hello v1"

    @GetMapping("/greeting", headers = ["X-API-Version=2"])
    fun v2Header(): String = "hello v2"

    companion object { const val V3 = "application/vnd.acme.v3+json" }

    @GetMapping("/greeting", produces = [V3])
    fun v3Vendor(): String = "hello v3"
}
```

---

## `GroupedOpenApi` для раздельных `v1/v2`, модульных bounded contexts

`GroupedOpenApi` — это бин из springdoc, который собирает отдельные спецификации по правилам: совпадению путей (`pathsToMatch`), пакетам (`packagesToScan`) и исключениям. Он удобен для разруливания нескольких версий: `/v1/**` → группа `v1`, `/v2/**` → группа `v2`. UI покажет переключатель между ними, а JSON/YAML будут доступны по `/v3/api-docs/v1` и `/v3/api-docs/v2`.

Группы работают и как «оглавление» доменов. Если у вас несколько bounded contexts — `catalog`, `billing`, `users` — удобно завести отдельные группы по пакетам и/или префиксам. Это снижает шум и помогает командам «владеть» своей частью контракта, не перетягивая всё в один гигантский файл.

Семантика важнее техники: группа — это не только фильтр, но и единица публикации. Вы можете выставлять `v1` наружу, а `v2` держать только на stage. Или же публиковать `catalog` публично, а `billing` — за VPN. Документацию нужно собирать так, как вы планируете её показывать миру.

Комбинируйте фильтрацию. Часто нужна группа «v1-public», которая берёт `pathsToMatch("/v1/**")` и при этом исключает «внутренние» теги. Это делается через `addOperationCustomizer`, где можно убрать операции по условию (`op.getTags()` содержит `"internal"`).

Группы помогают и в тестировании: интеграционные тесты могут скачивать `/v3/api-docs/v1` и сравнивать с эталоном. Это превращает документацию в артефакт, который контролируется в CI, а не «кнопку, которая иногда открывается».

Не забывайте, что `GroupedOpenApi` собирается из уже поднятого контекста. Если вы включаете/выключаете контроллеры по профилям (`@Profile("v2")`), группа «v1» на профиле `v2` станет пустой. Это нормально, но стоит осознанно управлять профилями при генерации/публикации.

При использовании WebFlux вместо WebMVC просто берите `springdoc-openapi-starter-webflux-ui`: API групп — те же. В монорепозитории микросервисов имеет смысл собрать «общий» модуль, где живут только конфиги групп, чтобы не дублировать правила.

Остерегайтесь пересечений групп. Если один и тот же эндпоинт попал в две группы, вы начнёте получать «плавающие» дубли в UI. Лучше делать группы взаимоисключающими, а для общих операций — осознанно дублировать только там, где это нужно.

Если версий больше двух, не стесняйтесь автоматизировать. Небольшой конфиг на основе списка версий создаст набор бинов циклом. Это проще поддерживать, чем копировать одинаковые методы на 5 групп.

Важное правило: имя группы — часть стабильного URL спецификации. Как только вы выбрали `v1` и `v2`, больше их не меняйте. Иначе клиенты/порталы будут ломаться на 404 при обновлении приложения.

**Java — группы `v1`, `v2` и доменные группы**

```java
package versioning.groups;

import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiGroupsConfig {

    @Bean
    GroupedOpenApi v1() {
        return GroupedOpenApi.builder()
            .group("v1")
            .pathsToMatch("/api/v1/**")
            .build();
    }

    @Bean
    GroupedOpenApi v2() {
        return GroupedOpenApi.builder()
            .group("v2")
            .pathsToMatch("/api/v2/**")
            .build();
    }

    @Bean
    GroupedOpenApi catalog() {
        return GroupedOpenApi.builder()
            .group("catalog")
            .packagesToScan("com.example.catalog")
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package versioning.groups

import org.springdoc.core.models.GroupedOpenApi
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiGroupsConfig {

    @Bean
    fun v1(): GroupedOpenApi =
        GroupedOpenApi.builder().group("v1").pathsToMatch("/api/v1/**").build()

    @Bean
    fun v2(): GroupedOpenApi =
        GroupedOpenApi.builder().group("v2").pathsToMatch("/api/v2/**").build()

    @Bean
    fun catalog(): GroupedOpenApi =
        GroupedOpenApi.builder().group("catalog").packagesToScan("com.example.catalog").build()
}
```

**application.yml — включим UI с переключателем групп**

```yaml
springdoc:
  api-docs:
    enabled: true
  swagger-ui:
    path: /docs
    urls:
      - name: v1
        url: /v3/api-docs/v1
      - name: v2
        url: /v3/api-docs/v2
      - name: catalog
        url: /v3/api-docs/catalog
```

---

## Совместное существование старых и новых контрактов (sunset headers, deprecations)

Переход между версиями — это всегда период сосуществования. Клиенты ещё на `v1`, а сервер уже предлагает `v2`. В этот момент прозрачная коммуникация — ключ к успеху. Два инструмента HTTP помогают формализовать процесс: заголовки `Deprecation` и `Sunset`. Первый сигнализирует, что ресурс устарел, второй указывает дату, когда он перестанет быть доступен.

Вместе с заголовками полезно добавлять `Link` на документ миграции: `Link: <https://developer.example.com/migration/v2>; rel="deprecation"`. Такой «путидатель» экономит десятки часов саппорта — клиент кликает и видит подробный гайд с примерами.

Внутри спецификации помечайте операции как `deprecated = true`. Это делает проблему видимой в UI и попадает в артефакт OpenAPI. Генераторы клиентов тоже учитывают этот флаг — IDE подсветит устаревший метод, что ускорит миграцию.

Скорость отключения старой версии — управляемая величина. Хорошая практика — объявить «срок жизни» устаревшей версии (например, 6–12 месяцев), регулярно напоминать об этом в рассылках/портале и постепенно «закручивать» лимиты (rate limit) на `v1`, чтобы мотивировать клиентов обновиться.

На уровне кода удобно добавлять простой фильтр/интерцептор, который на любые запросы к `/api/v1/**` ставит нужные заголовки. Это не меняет бизнес-логику, но создаёт единый слой коммуникации. Важное условие — не забыть исключить внутренние служебные маршруты, чтобы не шуметь в мониторинге.

Если у вас media-type версионирование, ориентируйтесь не на путь, а на `Accept`. Фильтр должен смотреть `Accept` и ставить заголовки, если там `application/vnd.acme.v1+json`. Это чуть сложнее, но тоже решаемо и централизуемо.

Комбинируйте «жёсткость»: `Deprecation`/`Sunset` + `@Operation(deprecated = true)` + банальные баннеры в Swagger UI (через custom index.html) — и шанс «мы не знали» стремится к нулю. Документация — часть UX, а не отголосок разработки.

На время миграции полезно вести метрики использования версий: доля запросов к `v1`/`v2`, топ клиентов, падение доли `v1` неделя к неделе. Когда падающая кривая приближается к нулю, можно планировать отключение уверенно и с минимальными рисками.

Наконец, не забывайте про каноническую ссылку в ответах `v1`: иногда разумно добавлять `Link: <.../v2/...>; rel="successor-version"`. Это прямой навигатор к эквивалентному ресурсу `v2`, особенно если URI отличаются.

И да, фиксируйте sunset-политику в публичной политике API. Чёткие ожидания — меньше конфликтов. «Мы поддерживаем устаревшие версии не менее 12 месяцев» — простая формула, которую понимают все.

**Java — фильтр, добавляющий Deprecation/Sunset для v1**

```java
package versioning.sunset;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import java.io.IOException;
import java.time.OffsetDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class SunsetFilter implements Filter {

    private static final String SUNSET_DATE = "2026-01-01T00:00:00Z";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {

        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse res = (HttpServletResponse) response;

        if (req.getRequestURI().startsWith("/api/v1/")) {
            res.addHeader("Deprecation", "true");
            res.addHeader("Sunset", OffsetDateTime.parse(SUNSET_DATE).format(DateTimeFormatter.RFC_1123_DATE_TIME));
            res.addHeader("Link", "<https://developer.example.com/migration/v2>; rel=\"deprecation\"");
        }
        chain.doFilter(request, response);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package versioning.sunset

import jakarta.servlet.*
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.stereotype.Component
import java.time.OffsetDateTime
import java.time.format.DateTimeFormatter

@Component
class SunsetFilter : Filter {

    private val sunsetDate = "2026-01-01T00:00:00Z"

    override fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
        val req = request as HttpServletRequest
        val res = response as HttpServletResponse
        if (req.requestURI.startsWith("/api/v1/")) {
            res.addHeader("Deprecation", "true")
            res.addHeader("Sunset", OffsetDateTime.parse(sunsetDate).format(DateTimeFormatter.RFC_1123_DATE_TIME))
            res.addHeader("Link", "<https://developer.example.com/migration/v2>; rel=\"deprecation\"")
        }
        chain.doFilter(request, response)
    }
}
```

**Java — помечаем устаревшую операцию**

```java
package versioning.depr;

import io.swagger.v3.oas.annotations.Operation;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrdersV1Controller {

    @GetMapping("/{id}")
    @Operation(summary = "Получить заказ (v1)", deprecated = true)
    public String get(@PathVariable String id) { return "v1:" + id; }
}
```

**Kotlin — эквивалент**

```kotlin
package versioning.depr

import io.swagger.v3.oas.annotations.Operation
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/orders")
class OrdersV1Controller {

    @GetMapping("/{id}")
    @Operation(summary = "Получить заказ (v1)", deprecated = true)
    fun get(@PathVariable id: String): String = "v1:$id"
}
```

---

## Документация лабораторных/внутренних эндпоинтов через теги/контексты

В реальной жизни есть «лабораторные» и «внутренние» методы: пилоты, A/B, инженерные заглушки. Их нельзя смешивать с публичной документацией. Простейшая дисциплина — помечать такие операции тегом `internal`/`lab` и исключать их из публичной спеки через группу `public`. Для внутренней спеки — наоборот: включать только эти теги плюс публичные.

В springdoc нет «флага тега» на уровне ядра, но есть `OperationCustomizer`, который позволяет программно модифицировать/фильтровать операции. Мы можем пройтись по операциям группы и удалить те, у которых есть нежелательный тег, или наоборот — оставить только нужные. Это гибче, чем поддерживать сложные паттерны путей.

Профили Spring — второй уровень контроля. Поднимайте «лабораторные» контроллеры только на профилях `dev`/`lab`, а в проде они физически отсутствуют. Тогда даже внутренняя спека их не увидит. Это не столько про документацию, сколько про безопасность: нельзя случайно вызвать лабораторный метод в бою.

`@Hidden` пригодится для единичных методов, которые не должны попасть ни в одну спецификацию (например, служебная отладка). Но для «видимых только внутри» лучше теги + группы: людям нужна документация, просто в другом месте.

Третий инструмент — разные Swagger UI: `public-docs` и `internal-docs`, каждая с собственным набором `urls`. Внешний UI отдавайте из публичного приложения/портала, а внутренний — за VPN/OIDC. Это снижает риск утечек и удобно командам.

Контексты окружений важны и для примеров: в «лабораторной» документации можно показывать payload с экспериментальными полями и пометкой «subject to change». В публичной — только стабильные поля. Так меньше соблазна копипастить экспериментальное в прод.

Когда фича переходит из lab в prod, меняйте тег с `lab` на доменный (`Orders`) и переносите операцию в публичную группу. Автотесты, проверяющие отсутствие `lab` в публичной спеки, помогут не забыть это сделать перед релизом.

Старайтесь поддерживать **явный список** внутренних тегов в одном месте (константа/enum), чтобы не размазывать «internal/lab/private/debug» по разным вариантам написания. Фильтр тогда будет простым и не промажет по опечатке.

Если у вас mono-repo и много модулей, храните правила групп/фильтров в «документационном» модуле и подключайте его как зависимость. Команды будут добавлять теги, а инфраструктура — автоматически «раскладывать» операции по спекам.

И наконец, держите процесс «переноса» документально оформленным: шаблон PR «перевести из lab в public», чек-лист «убрать тег, обновить примеры, добавить миграционный гайд, включить в публичную группу». Это снимает хаос.

**Java — группа public без internal-тегов**

```java
package internal.filter;

import org.springdoc.core.customizers.OperationCustomizer;
import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class PublicDocsConfig {

    @Bean
    GroupedOpenApi publicApi() {
        OperationCustomizer withoutInternal = (operation, handler) -> {
            if (operation.getTags() != null && operation.getTags().stream().anyMatch(t -> t.equalsIgnoreCase("internal") || t.equalsIgnoreCase("lab"))) {
                // вернём null — операция будет исключена
                return null;
            }
            return operation;
        };
        return GroupedOpenApi.builder()
            .group("public")
            .pathsToMatch("/api/**")
            .addOperationCustomizer(withoutInternal)
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package internal.filter

import org.springdoc.core.customizers.OperationCustomizer
import org.springdoc.core.models.GroupedOpenApi
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class PublicDocsConfig {

    @Bean
    fun publicApi(): GroupedOpenApi {
        val withoutInternal = OperationCustomizer { operation, _ ->
            val hasInternal = operation.tags?.any { it.equals("internal", true) || it.equals("lab", true) } ?: false
            if (hasInternal) null else operation
        }
        return GroupedOpenApi.builder()
            .group("public")
            .pathsToMatch("/api/**")
            .addOperationCustomizer(withoutInternal)
            .build()
    }
}
```

**Java — контроллер с тегом internal**

```java
package internal.sample;

import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/admin")
@Tag(name = "internal", description = "Внутренние операции")
public class AdminController {
    @GetMapping("/stats")
    public String stats() { return "ok"; }
}
```

**Kotlin — эквивалент**

```kotlin
package internal.sample

import io.swagger.v3.oas.annotations.tags.Tag
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/admin")
@Tag(name = "internal", description = "Внутренние операции")
class AdminController {
    @GetMapping("/stats")
    fun stats(): String = "ok"
}
```

---

## Стабильные URLs спецификаций для клиентов/порталов (per-version)

Потребители любят стабильность: «дай ссылку на OAS, и мы её положим в генератор». В springdoc у каждой группы есть стабильный URL вида `/v3/api-docs/{group}`. Стабилизируйте имена групп (`v1`, `v2`, `public`, `internal`) и публикуйте именно их — не завязанные на hostname или на случайный путь.

Для основного «корня» спецификаций удобно вынести базовый путь: `springdoc.api-docs.path=/_specs`. Тогда `/_specs/v1` и `/_specs/v2` будут жить отдельно от боевых URL. Это облегчает конфигурацию прокси и безопасность (можно чётко «пропускать только /_specs/**»).

Swagger UI поддерживает список `urls`, где каждому элементу можно дать «человеческое» имя и прямой путь к JSON. Это и есть «переключатель версий» на одной странице. Для партнёров вы можете собрать «публичный UI» с только `v1` и `v2`, не показывая внутренние группы вообще.

Стабильные ссылки полезно «замкнуть» на CDN (только GET), а JSON кэшировать на 5–10 минут. Это снижает нагрузку на прод-приложение и защищает от всплесков при генерации SDK. Главное — не кэшируйте навсегда, чтобы потребители получали новые поля.

Если у вас релизный цикл с тегами, публикуйте архивы спецификаций per tag: `/specs/v1@2025-10-24.json`. «Последняя стабильная» остаётся `/specs/v1.json`. Такой двойной доступ позволяет воспроизводить старые сборки и держать «живую» ссылку для интеграторов.

Не забывайте про CORS/доступ. Если портал/генератор забирает спеки с другого домена, включите `Access-Control-Allow-Origin` на JSON. Но в идеале — сервируйте спеки с того же домена, где крутится портал, чтобы не открывать лишние щели.

Формат — YAML или JSON? Генераторы чаще едят JSON, а людям удобнее YAML. Springdoc отдаёт JSON, но вы можете прокинуть его через простую конвертацию в пайплайне (openapi-generator-cli умеет). Главное — чтобы стабильные URL были «под рукой», формат вторичен.

Для **версионирования UI** храните конфиг `urls` в `application.yml` и не трогайте index.html. Так легче параметризовать дев/стейдж и держать всё как код. В редких случаях можно программно регистрировать `urls` через `SwaggerUiConfigParameters`.

И последнее: документируйте для партнёров, как часто может меняться спецификация «v1» (добавления), чтобы они не удивлялись новым полям. Это часть ожиданий по backward-совместимости.

**application.yml — стабильные пути и список версий**

```yaml
springdoc:
  api-docs:
    path: /_specs
  swagger-ui:
    path: /docs
    urls:
      - name: API v1
        url: /_specs/v1
      - name: API v2
        url: /_specs/v2
```

**Java — программно регистрируем URLs Swagger UI (опционально)**

```java
package specs.urls;

import org.springdoc.core.properties.SwaggerUiConfigParameters;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SwaggerUiUrlsConfig {

    @Bean
    SwaggerUiConfigParameters swaggerUiConfigParameters(SwaggerUiConfigParameters params) {
        params.addUrl("/_specs/v1", "API v1");
        params.addUrl("/_specs/v2", "API v2");
        return params;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package specs.urls

import org.springdoc.core.properties.SwaggerUiConfigParameters
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class SwaggerUiUrlsConfig {

    @Bean
    fun swaggerUiConfigParameters(params: SwaggerUiConfigParameters): SwaggerUiConfigParameters =
        params.apply {
            addUrl("/_specs/v1", "API v1")
            addUrl("/_specs/v2", "API v2")
        }
}
```

---

## Сводные порталы: агрегирование нескольких сервисов/групп

Когда микросервисов десятки, разработчикам и партнёрам нужен «портал документации». Самый простой способ — использовать возможности Swagger UI: указать массив `urls`, где каждый элемент — удалённый JSON OpenAPI другого сервиса. UI сам отрисует выпадающий список и подгрузит нужную спеку по клику. Это не «слияние», а удобный каталог.

Если хочется единый «каталог» с группировкой по доменам, заведите небольшой «документационный» сервис, который отдаёт JSON-описание списка доступных спецификаций (название, URL, описание, тэги). UI можно слегка кастомизировать (через `index.html`) и отрисовывать левое меню с группами. Технически просто, а пользы — много.

Слияние спецификаций в одну — спорная идея. Да, есть инструменты «склейки» (compose), но вы потеряете чёткие границы владения и рискуете конфликтами имён. Лучше давать отдельные спеки и общую навигацию/поиск по порталу. Так проще жить и развиваться независимо.

С точки зрения безопасности полезно перед порталом поставить OIDC и дать права группам: «внешние партнёры» видят только публичные спеки, «внутренние» — все. Swagger UI работает за OIDC нормально, главное — разрешить отдачу статики/конфигов после логина.

Хостинг: многие выкатывают портал как отдельный статический сайт (Nginx + статика SwaggerUI/Redoc), который подтягивает JSON по `urls`. Это дешево и легко кешируется. Альтернатива — отдельное Spring Boot приложение, которое в рантайме формирует список `urls` из конфигурации/реестра сервисов.

Содержимое портала — не только интерактивка. Добавьте разделы: changelog API, примеры Postman/HTTPie, ссылки на SDK, rate limits, политики депрекаций. Разработчики ценят «единый вход» со всей контекстной информацией, а не просто список эндпоинтов.

Для мульти-спек важно продумать «стабильные имена» сервисов: `catalog-v1`, `billing-v2`. Эти имена будут ключами в UI и ссылках. Смена имени — ломающее изменение для закладок/скриптов генерации. Зафиксируйте нотацию в гайдлайнах.

Мониторинг портала — метрики доступности каждой спек-ссылки. Периодический джоб, который ходит по `urls` и проверяет статус/размер JSON, помогает ловить «мертвые» сервисы и несоответствия CORS. Алертите команду-владельца сервиса, если их спека недоступна.

Наконец, держите портал как код: репозиторий с `index.html`, `application.yml`, пайплайном публикации и списком `urls`. Правка портала тогда — обычный PR с ревью, а не «кто-то поправил руками на сервере».

И про совместимость: если вы меняете структуру меню/названия, не ломайте прямые ссылки на конкретные спеки. Люди ставят закладки. Пусть старые URL ведут на ту же JSON-спеку, даже если визуально портал выглядит иначе.

**application.yml — Swagger UI, агрегирующий чужие спеки**

```yaml
springdoc:
  swagger-ui:
    path: /portal
    urls:
      - name: Catalog v1
        url: https://catalog.example.com/_specs/v1
      - name: Billing v2
        url: https://billing.example.com/_specs/v2
      - name: Users public
        url: https://users.example.com/_specs/public
```

**Java — программно регистрируем внешние спеки в портале**

```java
package portal.agg;

import org.springdoc.core.properties.SwaggerUiConfigParameters;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class PortalConfig {
    @Bean
    SwaggerUiConfigParameters portal(SwaggerUiConfigParameters params) {
        params.setPath("/portal");
        params.addUrl("https://catalog.example.com/_specs/v1", "Catalog v1");
        params.addUrl("https://billing.example.com/_specs/v2", "Billing v2");
        params.addUrl("https://users.example.com/_specs/public", "Users public");
        return params;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package portal.agg

import org.springdoc.core.properties.SwaggerUiConfigParameters
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class PortalConfig {

    @Bean
    fun portal(params: SwaggerUiConfigParameters): SwaggerUiConfigParameters =
        params.apply {
            path = "/portal"
            addUrl("https://catalog.example.com/_specs/v1", "Catalog v1")
            addUrl("https://billing.example.com/_specs/v2", "Billing v2")
            addUrl("https://users.example.com/_specs/public", "Users public")
        }
}
```

**Gradle (оба DSL) — добавим web и springdoc**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

# 9. Contract-first/Code-first, генерация и тестирование

## Contract-first: цикл «редактируем OAS → генерим сервер/клиент → реализуем»

Подход **contract-first** означает, что источником истины является спецификация OpenAPI (OAS) — файл YAML/JSON, где формально описаны эндпоинты, модели, коды ответов и безопасность. Мы сначала проектируем контракт, согласуем его с потребителями, фиксируем версию, и только потом пишем код сервера/клиентов. Это дисциплинирует интерфейсы и делает их предсказуемыми.

Практический цикл выглядит так: архитектор/разработчик правит OAS (ручками или через редактор), команда обсуждает изменения в MR, CI валидирует и линтует документ, затем из него **генерируются** заготовки серверного кода и SDK для клиентов. Реализация сервера «подгоняется» под контракт, а не наоборот.

Главная выгода — **договорённости до кода**. Клиентская команда может начинать работу, имея стабильный контракт и мок-сервер. Раннее согласование схем снижает риск «переделок по факту», когда сервер уже вкатили, а у клиента иные ожидания по полям и кодам ошибок.

Ключевой артефакт — OAS-файл. В нём фиксируется `info.version`, базовые `servers`, `paths`, `components`. Любое изменение проходит ревью как обычный код. Хорошая практика — держать читаемые описания (`description`, `summary`) и примеры (`examples`): это улучшает DX и сокращает время онбординга.

Генерация — не «волшебная кнопка», а инфраструктурная задача. Мы настраиваем Gradle-плагин OpenAPI Generator, который по YAML создаёт **серверные интерфейсы** (или контроллеры-заглушки) и **клиентские SDK**. Эти артефакты собираются в CI, публикуются как артефакты/бинарники и используются приложениями.

Важно разделить «генерируемое» и «руками написанное». Обычно генерим интерфейсы/DTO в отдельный модуль (или в `build/generated`), а реализацию пишем в своём исходном каталоге. Тогда регенерация не затирает нашу логику, а только обновляет контракты.

Слабое место — «ручное отклонение» от контракта. Если реализация пойдёт в сторону от OAS, мы получим дрейф. Его гасят автотестами и диффом спецификаций: в CI сверяем текущую спецификацию с базовой, запрещая ломающие изменения без смены версии.

Для совместимости в контракт-first особенно критичны **правила эволюции**: добавлять поля — можно (backward-compatible), менять тип/удалять — нельзя в текущем major. При необходимости ломающего изменения вводим новый major (`/v2` или новый файл OAS).

Тонкая настройка генератора — часть работы. Для Spring Boot мы выбираем `generatorName=spring` или `kotlin-spring`, задаём пакет, имя артефакта, предпочитаем интерфейсы с аннотациями, чтобы реализация оставалась под контролем.

Наконец, успех contract-first определяется не инструментами, а культурой: ревью OAS, единые шаблоны ошибок, политики версионирования и наличие людей, которые отвечают за «чистоту контракта». Инструменты лишь помогают это удержать.

**Gradle (Groovy DSL) — генерация server-stub из OAS**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.5'
    id 'org.openapi.generator' version '7.5.0'
    id 'java'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}

openApiGenerate {
    generatorName = 'spring'
    inputSpec = "$rootDir/spec/openapi.yaml"
    outputDir = "$buildDir/generated/server"
    apiPackage = 'com.acme.api'
    modelPackage = 'com.acme.api.model'
    invokerPackage = 'com.acme.api.invoker'
    configOptions = [
        interfaceOnly: 'true',
        useSpringBoot3: 'true',
        useTags      : 'true'
    ]
}

sourceSets {
    main.java.srcDir("$buildDir/generated/server/src/main/java")
}
```

**Gradle (Kotlin DSL) — то же**

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.5"
    id("org.openapi.generator") version "7.5.0"
    java
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
}

openApiGenerate {
    generatorName.set("spring")
    inputSpec.set("$rootDir/spec/openapi.yaml")
    outputDir.set("$buildDir/generated/server")
    apiPackage.set("com.acme.api")
    modelPackage.set("com.acme.api.model")
    invokerPackage.set("com.acme.api.invoker")
    configOptions.set(mapOf(
        "interfaceOnly" to "true",
        "useSpringBoot3" to "true",
        "useTags" to "true"
    ))
}
sourceSets["main"].java.srcDir("$buildDir/generated/server/src/main/java")
```

**Java — реализация сгенерированного интерфейса (пример OrdersApi)**

```java
package com.acme.orders.impl;

import com.acme.api.OrdersApi;
import com.acme.api.model.Order;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.RestController;

import java.time.OffsetDateTime;
import java.util.UUID;

@RestController
public class OrdersApiImpl implements OrdersApi {

    @Override
    public ResponseEntity<Order> getOrderById(UUID id) {
        var o = new Order().id(id).createdAt(OffsetDateTime.now()).title("Sample");
        return ResponseEntity.ok(o);
    }

    @Override
    public ResponseEntity<Order> createOrder(Order body) {
        body.id(UUID.randomUUID());
        body.createdAt(OffsetDateTime.now());
        return ResponseEntity.status(201).body(body);
    }
}
```

**Kotlin — эквивалент реализации**

```kotlin
package com.acme.orders.impl

import com.acme.api.OrdersApi
import com.acme.api.model.Order
import org.springframework.http.ResponseEntity
import org.springframework.stereotype.RestController
import java.time.OffsetDateTime
import java.util.UUID

@RestController
class OrdersApiImpl : OrdersApi {

    override fun getOrderById(id: UUID): ResponseEntity<Order> {
        val o = Order().id(id).createdAt(OffsetDateTime.now()).title("Sample")
        return ResponseEntity.ok(o)
    }

    override fun createOrder(body: Order): ResponseEntity<Order> {
        body.id(UUID.randomUUID())
        body.createdAt(OffsetDateTime.now())
        return ResponseEntity.status(201).body(body)
    }
}
```

---

## Code-first: автогенерация OAS из аннотаций; риски расхождения кода и контракта

Подход **code-first** стартует от рабочего кода контроллеров. Мы пишем REST в Spring Boot, описываем операции и модели аннотациями (`@Operation`, `@Schema`, `@ApiResponse`), а `springdoc-openapi` на старте приложения строит OpenAPI-спеку. Это быстрый старт: не надо сначала писать YAML — документация возникает «из кода».

Плюс подхода — низкий порог входа. Любой разработчик может добавить `@Operation(summary=...)` и получить обновлённую документацию в Swagger UI. Это идеально для внутренних сервисов и быстрых итераций, когда контракт эволюционирует вместе с кодом.

Риск — **дрейф требований**. Аннотациям свойственно устаревать: поменяли поведение — забыли обновить описание/примеры/коды ответов. Спека «красивая», но не отражает реальность или, наоборот, скрывает ломающие изменения, которые попали в код.

Чтобы держать качество, code-first усиливают автоматикой: снапшоты `/v3/api-docs` в репозитории, сравнение со «стабильной» спекой в CI, линтеры, обязательные примеры. В идеале любая правка контроллера, меняющая контракт, должна проходить через MR-ревью текста/диффа спецификации.

Второй риск — «аннотационный шум». Перегрузка кода документационной разметкой снижает читабельность. Решение — выносить крупные описания/примеры в константы/методы-фабрики или в отдельные классы-описатели, а в контроллере оставить только минимум.

Третий аспект — «скрытые» контракты. Без явного файла OAS внешним командам сложнее работать с контрактом вне рантайма (например, для моков или статического портала). Это решается публикацией JSON-файла как артефакта сборки.

Ещё один момент — **покрытие**. Аннотации не «знают», какие ветки кода при каких условиях возвращают определённые ошибки. Поэтому в code-first уделяем особое внимание глобальным ответам и единому мапперу ошибок, чтобы документация не расходилась с практикой.

Code-first хорошо работает в монокоманде: те же люди, кто пишет код, держат и контракт. При множестве внешних потребителей чаще побеждает contract-first, потому что создаёт «точку коммуникации» до кода.

И наконец, компромисс — гибрид: code-first для быстрых фич и внутренних API, но с регулярной выгрузкой OAS в файл и его ревью в MR. Это даёт скорость без потери управляемости.

**Gradle (оба DSL) — зависимости для springdoc**
Groovy:

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

Kotlin:

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — простой контроллер с аннотациями springdoc**

```java
package codefirst.sample;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.*;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/greeting")
public class GreetingController {

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    @Operation(summary = "Приветствие", description = "Возвращает приветствие по имени")
    @ApiResponse(responseCode = "200", description = "OK",
            content = @Content(schema = @Schema(implementation = Greeting.class)))
    public Greeting hello(@RequestParam(defaultValue = "world") String name) {
        return new Greeting("Hello, " + name + "!");
    }

    public record Greeting(String message) {}
}
```

**Kotlin — эквивалент**

```kotlin
package codefirst.sample

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.media.Content
import io.swagger.v3.oas.annotations.media.Schema
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/greeting")
class GreetingController {

    @GetMapping(produces = [MediaType.APPLICATION_JSON_VALUE])
    @Operation(summary = "Приветствие", description = "Возвращает приветствие по имени")
    @ApiResponse(
        responseCode = "200",
        description = "OK",
        content = [Content(schema = Schema(implementation = Greeting::class))]
    )
    fun hello(@RequestParam(defaultValue = "world") name: String) = Greeting("Hello, $name!")

    data class Greeting(val message: String)
}
```

**Java — снапшот-тест спецификации (минимальная проверка)**
Сохраняем предыдущий `openapi-baseline.json` в `src/test/resources`, сверяем, что все старые `paths` не исчезли.

```java
package codefirst.snapshot;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.web.client.RestClient;

import java.util.Iterator;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OpenApiSnapshotTest {

    private final ObjectMapper om = new ObjectMapper();

    @Test
    void oldPathsStillPresent() throws Exception {
        String current = RestClient.create().get().uri("http://localhost:8080/v3/api-docs")
                .retrieve().body(String.class);
        JsonNode cur = om.readTree(current);
        JsonNode base = om.readTree(getClass().getResourceAsStream("/openapi-baseline.json"));
        Iterator<String> it = base.get("paths").fieldNames();
        while (it.hasNext()) {
            String p = it.next();
            assertThat(cur.get("paths").has(p)).as("Path removed: " + p).isTrue();
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package codefirst.snapshot

import com.fasterxml.jackson.databind.ObjectMapper
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.web.client.RestClient

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OpenApiSnapshotTest {

    private val om = ObjectMapper()

    @Test
    fun oldPathsStillPresent() {
        val current = RestClient.create().get().uri("http://localhost:8080/v3/api-docs")
            .retrieve().body(String::class.java)
        val cur = om.readTree(current)
        val base = om.readTree(javaClass.getResourceAsStream("/openapi-baseline.json"))
        base.get("paths").fieldNames().forEachRemaining { p ->
            assertThat(cur.get("paths").has(p)).`as`("Path removed: $p").isTrue
        }
    }
}
```

---

## Генерация клиентов (Java/Kotlin/TS): Feign/WebClient/Retrofit, типобезопасность

Генерация клиентов по OAS даёт **типобезопасные вызовы** с автосборкой URL/параметров, сериализацией и проверкой моделей. Это снижает количество «боевого» кода интеграций и ускоряет их написание. Для JVM популярны целевые библиотеки: **Feign**, **WebClient**, **Retrofit**; для фронта — TypeScript (fetch/axios).

Выбор библиотеки — это компромисс. **Feign** прост и хорошо ложится в Spring-экосистему (интегрируется с Spring Cloud, ретраями, балансировкой). **WebClient** — реактивный, удобен в высоконагруженных интеграциях и при back-pressure. **Retrofit** нравится своей декларативностью и обилием адаптеров, но в Spring используется реже.

OpenAPI Generator поддерживает соответствующие пресеты (`generatorName=java` + `library=feign`, либо `generatorName=java` + `library=webclient`, либо `generatorName=kotlin` + `library=jvm-retrofit2`). Мы конфигурируем базовый пакет, артефакт, схемы авторизации (Bearer) и получаем SDK, который используем как обычную зависимость.

С точки зрения DX важно, чтобы SDK был **независимым**: без `@Component`/`@Autowire` внутри, с явной конфигурацией базового URL и авторизации. Тогда его можно использовать в любом приложении, а не только в «spring-контексте».

Генерация экономит время, но требует дисциплины версионирования. При изменениях контракта регенерация может менять имена методов/DTO. Поэтому «внешний» SDK лучше версионировать отдельно (semver) и публиковать в артефакт-репозитории.

Для авторизации обычно достаточно передать Bearer-токен в ApiClient (генератор создаёт поле/интерцептор). Важно не «зашивать» токен; лучше прокидывать `Supplier<String>` или конфигурировать перехватчик, чтобы токен из свежего кэша попадал в каждый вызов.

В Kotlin-мирах приятно генерировать клиенты под Retrofit/ktor. Они органично вписываются в корутины, а типы ошибок удобно моделировать через sealed-иерархии (часть генераторов это поддерживает опционально).

TypeScript-клиенты — отдельный поток работы: их генерят для фронтов/скриптов, публикуют в npm-реестр. Важно синхронизировать кодоген с релизами сервера: по тэгу API выпускаем тэг SDK.

Паттерн эксплуатации: для каждого сервиса — свой SDK, подключаемый в другие микросервисы. Это дешевый «BFF» для сервис-клиента и эффективная защита от копипасты HTTP-кода.

Наконец, не забывайте про **обработку ошибок**. Клиенты должны уметь читать `Problem Details` и выдавать доменную ошибку, а не «IOException». Часть генераторов позволяет настроить маппинг; иначе — пишем thin-adapter поверх.

**Gradle (Groovy DSL) — генерация Feign/WebClient клиентов**

```groovy
plugins { id 'org.openapi.generator' version '7.5.0' }

task genClientFeign(type: org.openapitools.generator.gradle.plugin.tasks.GenerateTask) {
    generatorName = 'java'
    library = 'feign'
    inputSpec = "$rootDir/spec/openapi.yaml"
    outputDir = "$buildDir/generated/client-feign"
    apiPackage = 'com.acme.sdk.api'
    modelPackage = 'com.acme.sdk.model'
    configOptions = [ dateLibrary: 'java8', useJakartaEe: 'true' ]
}

task genClientWebClient(type: org.openapitools.generator.gradle.plugin.tasks.GenerateTask) {
    generatorName = 'java'
    library = 'webclient'
    inputSpec = "$rootDir/spec/openapi.yaml"
    outputDir = "$buildDir/generated/client-webclient"
    apiPackage = 'com.acme.sdk.api'
    modelPackage = 'com.acme.sdk.model'
    configOptions = [ dateLibrary: 'java8', useJakartaEe: 'true' ]
}
```

**Gradle (Kotlin DSL) — то же**

```kotlin
plugins { id("org.openapi.generator") version "7.5.0" }

tasks.register("genClientFeign", org.openapitools.generator.gradle.plugin.tasks.GenerateTask::class) {
    generatorName.set("java")
    library.set("feign")
    inputSpec.set("$rootDir/spec/openapi.yaml")
    outputDir.set("$buildDir/generated/client-feign")
    apiPackage.set("com.acme.sdk.api")
    modelPackage.set("com.acme.sdk.model")
    configOptions.set(mapOf("dateLibrary" to "java8", "useJakartaEe" to "true"))
}
tasks.register("genClientWebClient", org.openapitools.generator.gradle.plugin.tasks.GenerateTask::class) {
    generatorName.set("java")
    library.set("webclient")
    inputSpec.set("$rootDir/spec/openapi.yaml")
    outputDir.set("$buildDir/generated/client-webclient")
    apiPackage.set("com.acme.sdk.api")
    modelPackage.set("com.acme.sdk.model")
    configOptions.set(mapOf("dateLibrary" to "java8", "useJakartaEe" to "true"))
}
```

**Java — использование сгенерированного Feign-SDK**

```java
package client.usage;

import com.acme.sdk.api.OrdersApi;
import com.acme.sdk.invoker.ApiClient;

public class OrdersClientExample {
    public static void main(String[] args) {
        ApiClient client = new ApiClient();
        client.setBasePath("https://api.example.com");
        client.setBearerToken("eyJhbGciOi..."); // лучше — перехватчик, см. ниже
        OrdersApi api = new OrdersApi(client);
        var order = api.getOrderById(java.util.UUID.fromString("550e8400-e29b-41d4-a716-446655440000"));
        System.out.println(order.getTitle());
    }
}
```

**Kotlin — эквивалент (WebClient-SDK)**

```kotlin
package client.usage

import com.acme.sdk.api.OrdersApi
import com.acme.sdk.invoker.ApiClient
import java.util.UUID

fun main() {
    val client = ApiClient()
    client.setBasePath("https://api.example.com")
    client.setBearerToken("eyJhbGciOi...") // в проде — токен-провайдер
    val api = OrdersApi(client)
    val order = api.getOrderById(UUID.fromString("550e8400-e29b-41d4-a716-446655440000")).block()
    println(order?.title)
}
```

---

## Контроль дрейфа: сравнение OAS (Spectral, oas-diff), контрактные тесты

Дрейф — это несоответствие между тем, что сервер фактически делает, и тем, что заявлено в спецификации. Он возникает и в contract-first (реализация «ушла» от YAML), и в code-first (аннотации «отстали» от кода). Задача — обнаруживать его **раньше релиза**.

Первый барьер — линтеры (например, Spectral): они ловят стилистические и структурные огрехи («нет описания», «нет примера», «неиспользуемая схема»). Это не про дрейф поведения, но отлично повышает базовую гигиену OAS.

Второй барьер — **дифф спецификаций**. Мы сравниваем текущую OAS с предыдущей (baseline). Инструменты типа `oas-diff` умеют классифицировать изменения как ломающие/совместимые. В CI можно «заваливать» сборку при удалении эндпоинтов/полей или ужесточении требований.

Третий слой — **контрактные тесты**. Клиентские тесты, основанные на контракте, проверяют, что реальный сервер отвечает в рамках OAS: коды, заголовки, медиа-типы, модели. Это можно делать HTTP-тестами с фикстурами из OAS (минимальный MVP — проверка ключевых path/методов и обязательных полей).

Полезный приём — снапшоты. Сохраняем `/v3/api-docs.json` как артефакт релиза; в новых MR сравниваем с ним. Если изменился контракт — требуем явной пометки в changelog и обновления версии.

Гранулярные проверки можно писать прямо на JVM, не тянув внешние тулкиты: загрузить два JSON-дерева и убедиться, что старые `paths` не исчезли, старые `responses` всё ещё есть, обязательные поля моделей не удалены. Это не заменит «умный дифф», но ловит 80% риска.

Контракты не должны «врать» про ошибки. Если в коде глобальный маппер отдаёт `application/problem+json`, а в OAS стоит `application/json`, это тоже дрейф. Автотест на ответ с 400/500 ловит такие несоответствия уверенно.

Хорошая практика — **контрактные smoke-тесты** для прод-окружений: раз в N минут ходим на `/v3/api-docs`, валидируем JSON и ключевые операции. Это мониторинг документации, а не только сервиса.

При мульти-версиях OAS (v1/v2) тестируем обе. То, что «последняя» зелёная, не гарантирует корректность устаревшей версии, которой пользуются клиенты.

И наконец, дрейф — не только технический аспект. Нужна культура «не меняем контракт без причины», ревью OAS наравне с кодом и понятные правила «что считается ломающим».

**Java — минимальный «no-breaking-remove» дифф-тест**

```java
package drift.diff;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import java.io.InputStream;
import java.util.Iterator;

import static org.assertj.core.api.Assertions.assertThat;

class NoBreakingRemoveTest {

    private final ObjectMapper om = new ObjectMapper();

    @Test
    void allOldOperationsExist() throws Exception {
        JsonNode oldSpec = load("/openapi-baseline.json");
        JsonNode newSpec = load("/openapi-current.json");
        for (Iterator<String> it = oldSpec.get("paths").fieldNames(); it.hasNext();) {
            String path = it.next();
            assertThat(newSpec.get("paths").has(path)).as("Removed path: " + path).isTrue();
            JsonNode oldOps = oldSpec.get("paths").get(path);
            JsonNode newOps = newSpec.get("paths").get(path);
            for (Iterator<String> it2 = oldOps.fieldNames(); it2.hasNext();) {
                String method = it2.next();
                assertThat(newOps.has(method)).as("Removed operation: " + method + " " + path).isTrue();
            }
        }
    }

    private JsonNode load(String res) throws Exception {
        try (InputStream in = getClass().getResourceAsStream(res)) {
            return om.readTree(in);
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package drift.diff

import com.fasterxml.jackson.databind.ObjectMapper
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class NoBreakingRemoveTest {

    private val om = ObjectMapper()

    @Test
    fun allOldOperationsExist() {
        val oldSpec = load("/openapi-baseline.json")
        val newSpec = load("/openapi-current.json")
        val oldPaths = oldSpec.get("paths")
        val newPaths = newSpec.get("paths")
        oldPaths.fieldNames().forEachRemaining { path ->
            assertThat(newPaths.has(path)).`as`("Removed path: $path").isTrue
            val oldOps = oldPaths.get(path)
            val newOps = newPaths.get(path)
            oldOps.fieldNames().forEachRemaining { method ->
                assertThat(newOps.has(method)).`as`("Removed operation: $method $path").isTrue
            }
        }
    }

    private fun load(res: String) = om.readTree(javaClass.getResourceAsStream(res))
}
```

---

## CI: публикуем OAS как артефакт, снапшоты, проверка линтерами

В CI OAS — полноценный артефакт, как JAR или Docker-образ. Мы валидируем, линтуем, сравниваем спецификацию, публикуем её в артефакт-хранилище и, при необходимости, выкатываем статический сайт (Swagger UI/Redoc), который её читает. Это делает контракт обозримым и версионируемым.

Первая ступень — **валидация**: запускаем OpenAPI Generator/`openapi-generator-cli validate` на YAML, ловим синтаксические ошибки. Дальше — **линт** (Spectral): правила на обязательные `summary`, описания кодов, запрет «пустых» схем.

Вторая — **дифф**. Сравниваем с предыдущей версией в `main` или с последним релизным артефактом. Если найдено удаление эндпоинта/поля или ужесточение ограничений — билд падает, пока не обновим major или не переделаем изменение.

Третья — **публикация**. Складываем `openapi.json` в артефакты CI, пушим в Git-релизы, в S3/CDN или в «портал документации». Для внешних партнёров важно иметь стабильный URL текущей версии и архив версий с датами.

Четвёртая — **генерация SDK**. По тэгу релиза прогоняем задачи генерации клиентов и публикуем бинарники/пакеты (Maven/NPM). Это связывает «контракт» и «библиотеки» в одну цепочку доверия.

Пятая — **снапшоты**. Храним baseline OAS для сравнения в MR. Это может быть JSON-файл в репозитории или артефакт последнего релиза, который скачиваем шагом CI перед диффом.

Шестая — **качество примеров**. Простая автоматизация: валидировать JSON-примеры из `examples` против схемы (есть готовые утилиты, но можно и минимальный скрипт). Это защищает от «битых» примеров в UI.

Седьмая — **пороги**. Можно ввести soft-quality-gate: если линтер нашёл N предупреждений — билд «жёлтый», больше порога — «красный». Это мягко подтягивает качество без тотального стопа.

Восьмая — **проверка доступности**. В CD шагом после деплоя дергаем `/v3/api-docs` и убеждаемся, что UI/JSON доступны, размеры разумны, заголовки CORS корректны. Это не про контракт, а про UX документации.

Девятая — **безопасность**. OAS не должен утекать наружу, если в ней есть внутренние эндпоинты. В публичной публикации — только «public» группа; «internal» — за VPN/OIDC.

Десятая — **прозрачность**. В релиз-нотах публикуем список изменений контракта (генерируемый из диффа). Потребители видят, что поменялось, и планируют обновления клиентов.

**Java — утилита для выгрузки OAS в файл (артефакт CI)**

```java
package ci.dump;

import java.io.IOException;
import java.net.URI;
import java.net.http.*;
import java.nio.file.*;

public class DumpOpenApi {
    public static void main(String[] args) throws IOException, InterruptedException {
        String url = args.length > 0 ? args[0] : "http://localhost:8080/v3/api-docs";
        Path out = Paths.get(args.length > 1 ? args[1] : "build/openapi.json");
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest req = HttpRequest.newBuilder(URI.create(url)).GET().build();
        HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
        Files.createDirectories(out.getParent());
        Files.writeString(out, resp.body());
        System.out.println("Saved: " + out.toAbsolutePath());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package ci.dump

import java.net.URI
import java.net.http.HttpClient
import java.net.http.HttpRequest
import java.net.http.HttpResponse
import java.nio.file.Files
import java.nio.file.Path

fun main(args: Array<String>) {
    val url = if (args.isNotEmpty()) args[0] else "http://localhost:8080/v3/api-docs"
    val out = Path.of(if (args.size > 1) args[1] else "build/openapi.json")
    val client = HttpClient.newHttpClient()
    val req = HttpRequest.newBuilder(URI.create(url)).GET().build()
    val resp = client.send(req, HttpResponse.BodyHandlers.ofString())
    Files.createDirectories(out.parent)
    Files.writeString(out, resp.body())
    println("Saved: ${out.toAbsolutePath()}")
}
```

**(Дополнительно) GitHub Actions — валидация и публикация артефакта**

```yaml
name: openapi-ci
on: [push, pull_request]
jobs:
  oas:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '21' }
      - run: ./gradlew bootRun -Pargs="--spring.main.web-application-type=servlet" &
      - run: |
          sleep 10
          ./gradlew -q run -PmainClass=ci.dump.DumpOpenApi
      - run: npm i -g @stoplight/spectral-cli
      - run: spectral lint build/openapi.json
      - uses: actions/upload-artifact@v4
        with: { name: openapi.json, path: build/openapi.json }
```

---

## Интеграция со Spring REST Docs (restdocs-openapi) для гибридного подхода

**Spring REST Docs** документирует API примерами, «снятыми» с интеграционных тестов. Это даёт феноменальную точность: документация строится на реальных запросах/ответах. В гибридном подходе мы превращаем эти сниппеты в OpenAPI-спеку (через restdocs-api-spec), получая лучшее из обоих миров.

Механика: пишем MockMvc/WebTestClient тесты, которые выполняют запросы и через REST Docs генерируют сниппеты (`curl-request.adoc`, `http-response.adoc`, `request-fields.adoc` и т. п.). Плагин `restdocs-api-spec` собирает их в OpenAPI 3, создавая YAML/JSON.

Сильная сторона — **несходимость невозможна**: если тест зелёный, значит сервер реально так отвечает, и документация ему соответствует. Это снимает большую часть дрейфа, свойственного чистому code-first.

REST Docs требует дисциплины: тесты должны покрывать все варианты (коды, ошибки, пустые списки). За это платим временем написания тестов, но выигрываем качеством документации и устойчивостью к регрессиям.

Генерируемый OAS можно дальше линтить/диффить в CI, публиковать и на его основе собирать SDK. При необходимости вносим ручные правки (например, безопасность) — плагин поддерживает доп. метаданные.

Гибрид уместен там, где контракт важен и где мы всё равно пишем интеграционные тесты. Тогда «стоимость» документации почти нулевая — мы просто включаем REST Docs и собираем OAS.

Практический совет: держите тесты с REST Docs отдельно от «бизнесовых», чтобы не мешать целям. Документационные тесты фокусируются на примерах и кроют краевые случаи.

Осторожность: REST Docs не «видит» все варианты сериализации/локализации автоматически. Если у вас много контент-типов, нужно писать отдельные тесты, чтобы зафиксировать каждый.

Нюанс — версионирование. Генерируйте OAS per-группа/версия (например, `/v1/**`) отдельными наборами тестов. Тогда документация будет разбита так же, как и ваши спецификации.

И последнее: гибрид — это не «всегда лучше». Если у вас стабильный контракт-first с хорошей культурой ревью OAS, можно обойтись без REST Docs. Но если качество примеров — боль, он сильно помогает.

**Gradle (Groovy DSL) — зависимости REST Docs + генерация OAS**

```groovy
dependencies {
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testImplementation "org.springframework.restdocs:spring-restdocs-mockmvc"
    testImplementation "com.epages:restdocs-api-spec-mockmvc:0.17.2"
}

test {
    systemProperty "restdocs.outputDir", "$buildDir/generated-snippets"
}
```

**Gradle (Kotlin DSL) — то же**

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.restdocs:spring-restdocs-mockmvc")
    testImplementation("com.epages:restdocs-api-spec-mockmvc:0.17.2")
}
tasks.test {
    systemProperty("restdocs.outputDir", "$buildDir/generated-snippets")
}
```

**Java — MockMvc тест с REST Docs и OAS-экспортом**

```java
package docs.restdocs;

import com.epages.restdocs.apispec.MockMvcRestDocumentationWrapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.restdocs.payload.PayloadDocumentation;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.document;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class GreetingDocsTest {

    @Autowired MockMvc mvc;

    @Test
    void greet() throws Exception {
        mvc.perform(get("/api/greeting").param("name", "Alice"))
            .andExpect(status().isOk())
            .andDo(MockMvcRestDocumentationWrapper.document("greeting",
                PayloadDocumentation.responseFields(
                    PayloadDocumentation.fieldWithPath("message").description("Приветствие")
                )));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.restdocs

import com.epages.restdocs.apispec.MockMvcRestDocumentationWrapper
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.restdocs.payload.PayloadDocumentation.fieldWithPath
import org.springframework.restdocs.payload.PayloadDocumentation.responseFields
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import javax.annotation.Resource

@SpringBootTest
@AutoConfigureMockMvc
class GreetingDocsTest {

    @Resource
    lateinit var mvc: MockMvc

    @Test
    fun greet() {
        mvc.get("/api/greeting") { param("name", "Alice") }
            .andExpect { status { isOk() } }
            .andDo(
                MockMvcRestDocumentationWrapper.document(
                    "greeting",
                    responseFields(fieldWithPath("message").description("Приветствие"))
                )
            )
    }
}
```

**application.yml — параметры генерации OAS из сниппетов (epages)**

```yaml
restdocs-api-spec:
  output-directory: build/openapi
  snippets-directory: build/generated-snippets
  servers:
    - url: http://localhost:8080
  info:
    title: Demo API
    version: 1.0.0
```

# 10. Публикация, эксплуатация и UX документации

## Доступ к UI в проде: защита (Basic/OIDC), IP-allowlist, выключение в проде

В продакшне Swagger UI и `/v3/api-docs` — это такой же внешний интерфейс, как и ваш API. Их нужно защищать по тем же правилам: минимальные права по умолчанию, явное включение, аудит. Хорошая отправная точка — по умолчанию **выключать** UI на prod и оставлять только на dev/stage.

Когда UI всё-таки нужен в проде (например, для партнёров и саппорта), применяют аутентификацию. Для внутренних порталов часто достаточно **Basic Auth** за VPN. Для внешних интеграторов лучше **OIDC/OAuth2**: пользователи логинятся через IdP, а доступ контролируется группами/ролями.

Помимо аутентификации полезно ограничить **источник трафика**. IP-allowlist на уровне Ingress/NGINX/Cloud LB защищает от случайного публичного доступа. Это не замена аутентификации, а второй рубеж.

Ещё один слой — разделение путей. Перенесите спецификацию и UI в отдельный префикс (`/_specs`, `/docs`) и применяйте к нему строгие правила. Так проще описать политики в обратном прокси и в Spring Security.

Наблюдаемость критична: все запросы к UI и спецификации должны логироваться с идентификатором пользователя/клиента. Это помогает в разборе инцидентов и доказывает, что доступы использовались корректно.

Не забывайте о rate limit. Даже документация может лечь от «бота». Поставьте базовые лимиты на `/v3/api-docs` и UI-статические ресурсы, чтобы избежать DDOS легальными средствами.

В сценариях, где UI в проде не нужен, оставляйте только `/v3/api-docs` **защищённым** эндпоинтом, который забирает ваш портал разработчика или CI. Ручного доступа у людей тогда нет вообще.

При выкладке на Kubernetes удобно использовать отдельный сервис/Ingress для UI и спецификации. Это даёт независимые правила, артефакты и изоляцию рисков.

Для аварийной работы держите фичу-флаг: возможность одним конфигом скрыть UI, не перезапуская приложение. Это банально, но часто экономит время.

Документируйте политику: «UI доступен только во внутренних сетях, под OIDC, логируется, rate-limited, выключаем по флагу». Ясные правила уменьшают споры и улучшат эксплуатацию.

**Gradle (Groovy) — зависимости**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-security"
    implementation "org.springframework.boot:spring-boot-starter-oauth2-client"
    implementation "org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0"
}
```

**Gradle (Kotlin) — зависимости**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**application.yml — выключаем UI на prod, переносим пути**

```yaml
spring:
  profiles:
    group:
      prod: prod
      dev: dev
springdoc:
  api-docs:
    path: /_specs
    enabled: true
  swagger-ui:
    path: /docs
    enabled: false

---
spring.config.activate.on-profile: dev
springdoc.swagger-ui.enabled: true
```

**Java — SecurityConfig: Basic для /docs, OIDC для /_specs**

```java
package docs.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class DocsSecurityConfig {

    @Bean
    SecurityFilterChain docs(HttpSecurity http) throws Exception {
        http
          .securityMatcher("/docs/**", "/_specs/**")
          .authorizeHttpRequests(reg -> reg
              .requestMatchers("/docs/**").authenticated()
              .requestMatchers("/_specs/**").hasAnyAuthority("ROLE_DOCS_VIEWER","SCOPE_docs.read"))
          .httpBasic(basic -> {})                 // для /docs
          .oauth2Login(oauth -> {})              // для /_specs
          .csrf(csrf -> csrf.ignoringRequestMatchers("/_specs/**","/docs/**"));
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.security

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class DocsSecurityConfig {

    @Bean
    fun docs(http: HttpSecurity): SecurityFilterChain =
        http
            .securityMatcher("/docs/**", "/_specs/**")
            .authorizeHttpRequests {
                it.requestMatchers("/docs/**").authenticated()
                  .requestMatchers("/_specs/**").hasAnyAuthority("ROLE_DOCS_VIEWER", "SCOPE_docs.read")
            }
            .httpBasic { }
            .oauth2Login { }
            .csrf { it.ignoringRequestMatchers("/_specs/**", "/docs/**") }
            .build()
}
```

**Kubernetes Ingress (NGINX) — IP allowlist для UI**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-docs
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: 10.0.0.0/8,192.168.0.0/16
spec:
  rules:
    - host: docs.example.com
      http:
        paths:
          - path: /docs
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 8080
```

---

## Экспорт статической документации (`/v3/api-docs` → HTML/Redoc) и хостинг

Статическая документация решает две задачи: отдавать документацию без поднятого приложения и разгружать прод-инстансы от скачиваний спек. Самый простой вариант — выгружать JSON из `/v3/api-docs` при сборке и собирать статический сайт на **Redoc** или «самодельном» Swagger UI.

Redoc хорош своей читаемостью: одна страница, удобная навигация, левое меню. Для публикации достаточно HTML-файла, который указывает на URL JSON или содержит его inline. Swagger UI тоже можно сделать статикой, но чаще его держат «живым» в приложении.

С точки зрения DevOps — статика идеальна для CDN: дешёвый, быстрый, кэшируемый артефакт. Обновляете при релизе API, публикуете в S3/Cloud Storage + CDN, получаете высокую доступность без нагрузки на API-бэкенд.

Экспорт статических спек можно объединить с «архивом версий»: храните `openapi-v1.json`, `openapi-v2.json` и датированные снапшоты. Это упрощает воспроизводимость и отладку «по следам».

Хостинг статической документации следует защищать, как и живой UI. Если там есть внутренние эндпоинты, публикуйте статический сайт за OIDC/Basic или держите его полностью во внутренней сети.

Иногда удобно встроить статическую Redoc-страницу прямо в Spring Boot: отдавать предсобранный HTML из `/public`. Это компромисс между «живым» UI и внешним статическим хостингом.

При экспорте не забывайте о CORS: если статика с одного домена, а JSON — с другого, потребуется `Access-Control-Allow-Origin`. Проще инлайнить JSON в HTML, но это увеличит размер файла.

Для локальной разработки добавьте Gradle-таску, которая скачает `/v3/api-docs` и положит в `build/openapi.json`. Из него уже генерируйте Redoc-страницу. Это делает процесс повторяемым.

Не перегружайте страницу. Большие OAS (5+ МБ) тяжело рендерятся в браузере. Разделение по `GroupedOpenApi` улучшит UX и снизит размер каждого JSON.

И наконец, тестируйте доступность: проверяйте, что статика открывается, ссылки на JSON живые, и кэш не мешает видеть свежую версию после релиза.

**Gradle (Groovy) — задача экспорта OAS и генерация Redoc**

```groovy
plugins { id 'base' }
configurations { redoc }
dependencies { redoc "org.webjars:redoc:2.0.0-rc.56" }

task exportOpenApi(type: Exec) {
    commandLine "bash","-lc","curl -fsSL http://localhost:8080/_specs > build/openapi.json"
}

task redocHtml {
    inputs.file("build/openapi.json")
    outputs.file("build/redoc.html")
    doLast {
        def json = file("build/openapi.json").text.replace("\\","\\\\").replace("`","\\`")
        file("build/redoc.html").text = """
<!doctype html>
<html><head><meta charset="utf-8"><title>API Docs</title>
<script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
</head><body>
<redoc spec-url='openapi.json'></redoc>
</body></html>
"""
        copy { from("build/openapi.json"); into("build"); rename { "openapi.json" } }
    }
}
```

**Gradle (Kotlin) — эквивалент**

```kotlin
plugins { base }
tasks.register<Exec>("exportOpenApi") {
    commandLine("bash","-lc","curl -fsSL http://localhost:8080/_specs > build/openapi.json")
}
tasks.register("redocHtml") {
    inputs.file("build/openapi.json")
    outputs.file("build/redoc.html")
    doLast {
        file("build/redoc.html").writeText("""
<!doctype html>
<html><head><meta charset="utf-8"><title>API Docs</title>
<script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
</head><body>
<redoc spec-url='openapi.json'></redoc>
</body></html>
""".trimIndent())
        file("build").mkdirs()
        file("build/openapi.json").copyTo(file("build/openapi.json"), overwrite = true)
    }
}
```

**Java — контроллер для отдачи статической Redoc-страницы**

```java
package docs.staticredoc;

import org.springframework.core.io.ClassPathResource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;

@RestController
@RequestMapping("/redoc")
public class RedocController {

    @GetMapping(produces = MediaType.TEXT_HTML_VALUE)
    public ResponseEntity<byte[]> redoc() throws IOException {
        var bytes = new ClassPathResource("static/redoc.html").getContentAsByteArray();
        return ResponseEntity.ok().contentType(MediaType.TEXT_HTML).body(bytes);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.staticredoc

import org.springframework.core.io.ClassPathResource
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/redoc")
class RedocController {

    @GetMapping(produces = [MediaType.TEXT_HTML_VALUE])
    fun redoc(): ResponseEntity<ByteArray> {
        val bytes = ClassPathResource("static/redoc.html").inputStream.readAllBytes()
        return ResponseEntity.ok().contentType(MediaType.TEXT_HTML).body(bytes)
    }
}
```

---

## Кастомизация Swagger UI: брендирование, фильтры, сортировка, скрытие служебных тегов

UX документации — это не «косметика», а продуктивность потребителей. В Swagger UI можно включить **фильтр** по операциям, настроить сортировку тегов и операций, включить отображение `operationId`, подсветку синтаксиса и собственный заголовок/логотип.

Брендирование повышает узнаваемость портала для партнёров: логотип, цвета, favicon. Для этого достаточно положить в `src/main/resources/static/docs` кастомный `index.html` и подключить свои стили. Springdoc будет использовать его вместо встроенного.

Сортировка упрощает навигацию. `tagSorter=alpha` и `operationsSorter=alpha` делают UI детерминированным, а `displayOperationId=true` позволяет быстро искать метод по коду клиента, где вызывается `operationId`.

Фильтр (`filter=true`) в UI — мощная вещь: разработчик моментально находит нужный эндпоинт по названию поля или части пути. Это особенно полезно в больших спеках.

Чтобы скрыть «служебные» теги (internal/lab), лучше собрать отдельную «публичную» группу (мы делали выше). Если всё в одной группе, можно программно пометить такие операции `@Hidden` или вырезать их в `OpenApiCustomiser`.

Дополнительно можно включить `docExpansion=none`, чтобы UI открывался «компактным» и не грузил браузер. На больших спеках это разница между «работает» и «тормозит».

При необходимости внедрите «быстрые ссылки»: файл `index.html` UI можно дополнить блоком с ссылками на changelog, SDK и rate-limits. Это простой, но полезный UX-штрих.

Наконец, не забывайте про доступность (a11y): контраст, размеры шрифтов и понятные тексты. Swagger UI уже продуман, но бренд-стили не должны ломать базовые принципы.

Проверяйте кастомизацию на мобильных устройствах. Партнёры часто читают документацию «с дороги», и сломанное меню — реальная боль.

Любую кастомизацию фиксируйте как код и покрывайте smoke-тестом «страница отдаётся 200, заголовок есть», чтобы релизы не возвращали «голый» UI случайно.

**application.yml — параметры UI**

```yaml
springdoc:
  swagger-ui:
    path: /docs
    filter: true
    display-operation-id: true
    tags-sorter: alpha
    operations-sorter: alpha
    doc-expansion: none
    syntaxHighlight:
      activated: true
    tryItOutEnabled: true
```

**Java — вырезаем служебные теги из UI (если остаются в спеках)**

```java
package docs.ui;

import io.swagger.v3.oas.models.OpenAPI;
import org.springdoc.core.customizers.OpenApiCustomiser;
import org.springframework.context.annotation.*;

@Configuration
public class HideInternalTags {

    @Bean
    OpenApiCustomiser removeInternalTag() {
        return (OpenAPI oas) -> {
            if (oas.getTags() == null) return;
            oas.getTags().removeIf(t -> "internal".equalsIgnoreCase(t.getName()) || "lab".equalsIgnoreCase(t.getName()));
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.ui

import io.swagger.v3.oas.models.OpenAPI
import org.springdoc.core.customizers.OpenApiCustomiser
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class HideInternalTags {

    @Bean
    fun removeInternalTag(): OpenApiCustomiser = OpenApiCustomiser { oas: OpenAPI ->
        oas.tags?.removeIf { it.name.equals("internal", true) || it.name.equals("lab", true) }
    }
}
```

**Swagger UI index.html (фрагмент брендирования)**

```html
<!-- src/main/resources/static/docs/index.html -->
<!doctype html>
<html>
<head>
  <meta charset="UTF-8"/>
  <title>Acme API Docs</title>
  <link rel="icon" href="/docs/favicon.ico">
  <style>
    .topbar { background: #0f172a; }
    .topbar-wrapper .link img { content: url('/docs/logo.svg'); height: 30px; }
  </style>
</head>
<body>
<div id="swagger-ui"></div>
<script src="/docs/swagger-ui-bundle.js"></script>
<script>
  window.ui = SwaggerUIBundle({
    url: "/_specs",
    dom_id: '#swagger-ui',
    filter: true,
    displayOperationId: true,
    tagsSorter: "alpha",
    operationsSorter: "alpha",
    docExpansion: "none"
  });
</script>
</body>
</html>
```

---

## Порталы разработчика: changelog, примеры Postman/HTTPie, SDK-линки, rate limits

Полезная документация — это не только OAS. Портал разработчика должен отвечать на практичные вопросы: «что изменилось?», «как быстро попробовать?», «где взять SDK?», «какие лимиты?». Эти элементы экономят десятки часов у интеграторов.

Changelog — артефакт коммуникации. Его проще всего генерировать из MR/релиз-нотов и публиковать как статическую страницу рядом с UI. Формат должен фиксировать тип изменения (feat/fix/breaking), дату и ссылки на спецификацию.

Примеры Postman/HTTPie ускоряют первые шаги. На базе `/v3/api-docs` можно сгенерировать Postman-коллекцию (в CI утилитами), но даже «ручная» коллекция с 5–10 ключевыми запросами — огромная помощь.

SDK-линки — «мост» между контрактом и кодом. Публикуйте клиентские библиотеки (JVM/TS) как артефакты и давайте короткие примеры использования. Версионирование SDK должно соответствовать версии API, а не релизу сервиса.

Rate limits важны для планирования трафика. Документируйте общие лимиты и возможность расширения по запросу. В ответах показывайте `X-RateLimit-*` и `Retry-After`, а в UI — примечание, как эти заголовки читаются.

Для портала можно сделать мини-бэкенд, который отдаёт JSON с «метаданными» документации: версии, ссылки на SDK, дата последнего изменения, URL changelog. UI подтянет этот JSON и отрисует «шапку».

Безопасность портала — как у UI: OIDC/Basic, аудит, rate limit, IP-ограничения. Не публикуйте внутренние эндпоинты наружу никогда.

Дайте поле для обратной связи: ссылку на канал поддержки, SLA ответа, форму запроса расширения лимитов. Коммуникация — часть DX.

Регламентируйте регулярность обновлений changelog. Нельзя «копить» изменения и выкатывать раз в полгода — потребители не успеют сориентироваться.

Наконец, держите портал «как код» — PR-ы, ревью, пайплайны. Документация — часть продукта.

**Java — мини-портал метаданных документации**

```java
package devportal.meta;

import org.springframework.web.bind.annotation.*;
import java.time.OffsetDateTime;
import java.util.List;

@RestController
@RequestMapping("/devportal/meta")
public class DevPortalMetaController {

    @GetMapping
    public Meta meta() {
        return new Meta(
            "1.12.0",
            OffsetDateTime.now().toString(),
            List.of(new Sdk("java-feign","https://repo.example.com/acme-sdk-java")),
            "https://docs.example.com/changelog",
            new RateLimits(1000, 900, 60)
        );
    }

    public record Meta(String apiVersion, String lastUpdated, List<Sdk> sdks, String changelogUrl, RateLimits rateLimits) {}
    public record Sdk(String name, String url) {}
    public record RateLimits(int limitPerMin, int typicalBurst, int retryAfterSec) {}
}
```

**Kotlin — эквивалент**

```kotlin
package devportal.meta

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.time.OffsetDateTime

@RestController
@RequestMapping("/devportal/meta")
class DevPortalMetaController {

    @GetMapping
    fun meta(): Meta = Meta(
        apiVersion = "1.12.0",
        lastUpdated = OffsetDateTime.now().toString(),
        sdks = listOf(Sdk("java-feign", "https://repo.example.com/acme-sdk-java")),
        changelogUrl = "https://docs.example.com/changelog",
        rateLimits = RateLimits(1000, 900, 60)
    )

    data class Meta(
        val apiVersion: String,
        val lastUpdated: String,
        val sdks: List<Sdk>,
        val changelogUrl: String,
        val rateLimits: RateLimits
    )
    data class Sdk(val name: String, val url: String)
    data class RateLimits(val limitPerMin: Int, val typicalBurst: Int, val retryAfterSec: Int)
}
```

**Java — фильтр, выставляющий rate limit заголовки (демо)**

```java
package devportal.ratelimit;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
public class RateLimitHeadersFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        throws IOException, ServletException {
        HttpServletResponse resp = (HttpServletResponse) res;
        resp.setHeader("X-RateLimit-Limit", "1000");
        resp.setHeader("X-RateLimit-Remaining", "998");
        resp.setHeader("Retry-After", "60");
        chain.doFilter(req, res);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package devportal.ratelimit

import jakarta.servlet.*
import jakarta.servlet.http.HttpServletResponse
import org.springframework.stereotype.Component

@Component
class RateLimitHeadersFilter : Filter {
    override fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
        val resp = response as HttpServletResponse
        resp.setHeader("X-RateLimit-Limit", "1000")
        resp.setHeader("X-RateLimit-Remaining", "998")
        resp.setHeader("Retry-After", "60")
        chain.doFilter(request, response)
    }
}
```

---

## Управление изменениями: deprecation policy, совместимость, коммуникация с потребителями

Политика изменений — это «социальный контракт» с потребителями API. Она должна отвечать на вопросы: сколько живёт устаревшая версия, какие изменения считаются ломающими, как мы уведомляем и когда отключаем.

Семвер-дисциплина: любое несовместимое изменение — новый **major** (`v2`). Добавление полей, новых эндпоинтов и статусов — допустимо в пределах `v1`. Переименование, ужесточение валидации, удаление — только через `v2`.

Deprecation policy фиксирует сроки. Типовой шаблон: объявляем депрекацию, даём **Sunset** через 6–12 месяцев, рассылаем уведомления и показываем предупреждения в UI. После даты — включаем «жёсткие» ограничения (rate limit) и удаляем в ближайшее окно.

Коммуникация должна быть много-канальной: changelog, письма, баннер в UI, заголовки `Deprecation/Sunset`, тикеты ключевым клиентам. Пассивного «записали в README» мало.

Для безопасности включайте «канареек»: сначала отключайте старые эндпоинты только для небольшой группы клиентов или в ночные часы. Это снизит риски массового падения.

Автоматизация чертовски помогает: скрипт, который смотрит `@Operation(deprecated=true)` и формирует список для рассылки/баннера, исключает человеческий фактор.

Для совместимости важна **контрактная чистота**: линтеры и `oas-diff` в CI блокируют ломающие изменения без повышения major. Прозрачность — потребители благодарят.

Регулярно проводите «инвентаризацию устаревшего»: список deprecated-операций, кто ими пользуется, прогресс миграции. Без этого депрекация превращается в бесконечность.

Фиксируйте исключения. Бывают стратегические причины «сломать» быстрее (безопасность/стоимость). Тогда — явное решение с согласованием.

Не забывайте про SDK: депрекация в API → депрекация методов в SDK. IDE-подсветка работает на вас.

В конце — «хвосты»: после отключения закройте документацию устаревшей версии и оставьте только архивные YAML с пометкой «readonly».

**Java — баннер о депрекации через HandlerInterceptor**

```java
package changes.banner;

import jakarta.servlet.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class DeprecationBannerInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        if (req.getRequestURI().startsWith("/api/v1/")) {
            res.addHeader("Deprecation", "true");
            res.addHeader("Sunset", "Wed, 01 Jan 2026 00:00:00 GMT");
            res.addHeader("Link", "<https://developer.example.com/migrate/v2>; rel=\"deprecation\"");
        }
        return true;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package changes.banner

import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.stereotype.Component
import org.springframework.web.servlet.HandlerInterceptor

@Component
class DeprecationBannerInterceptor : HandlerInterceptor {
    override fun preHandle(req: HttpServletRequest, res: HttpServletResponse, handler: Any): Boolean {
        if (req.requestURI.startsWith("/api/v1/")) {
            res.addHeader("Deprecation", "true")
            res.addHeader("Sunset", "Wed, 01 Jan 2026 00:00:00 GMT")
            res.addHeader("Link", "<https://developer.example.com/migrate/v2>; rel=\"deprecation\"")
        }
        return true
    }
}
```

**Gradle (оба DSL) — утилиты проверки спецификации в CI**
Groovy:

```groovy
tasks.register("validateOpenApi", Exec) {
    commandLine "bash","-lc","npx -y @redocly/cli lint build/openapi.json"
}
```

Kotlin:

```kotlin
tasks.register<Exec>("validateOpenApi") {
    commandLine("bash","-lc","npx -y @redocly/cli lint build/openapi.json")
}
```

---

## Наблюдаемость: метрики наличия/размера OAS, алерты на разрыв контракта

Документация — часть продакшна, её тоже нужно мониторить. Минимум — **наличие** `/v3/api-docs` (или `/_specs`) и **размер** JSON. Нулевой размер или внезапное уменьшение — сигнал, что что-то отвалилось в генерации или фильтрации.

С Micrometer можно легко зарегистрировать `gauge` и `timer` для выдачи спецификации. Соберите метрику `oas.size.bytes`, обновляйте её при каждом обращении или по расписанию. Алерт: «если размер < X КБ — предупреждение».

Следующий слой — «контрактные» алерты. В CI при диффе фиксируйте ломающие изменения и шлите уведомления владельцам. В проде — проверка, что «обязательные» пути присутствуют. Это найдёт случайные регрессии при конфигурации групп/фильтров.

Логи важны не меньше метрик: все запросы к UI/спецификации — с userId/ip и кодом ответа. На основе их можно строить usage-графики и ловить аномалии активности.

Отдельная метрика — время генерации OAS на старте приложения. Если внезапно стало долго, разработчикам пора «подчистить» аннотации/конвертеры.

Если спек много (v1/v2/public/internal), собирайте метрики для каждой. Так вы увидите, что «public v1» пропала, а «internal» — жива, и сократите время поиска причины.

Сторонние порталы тоже нужно мониторить. Если вы отдаёте статическую Redoc-страницу из S3/CDN, настройте health-чек и алерт на 4xx/5xx или аномальный размер.

Храните «эталонный» размер и список путей в конфиге мониторинга. Это простая, но эффективная проверка целостности.

Генерируйте **traceId** для выдачи `/v3/api-docs` и прокидывайте его в логи. Это поможет связать обращение к документации с проблемами клиентов «на месте».

Не забывайте про SLO: «доступность документации 99.9%» звучит странно, но на практике влияет на скорость интеграций. Укажите SLO рядом с SLO API.

И наконец, раз в квартал делайте аудит: открывается ли UI, корректно ли работает Authorize, не устарели ли инструкции на страницах портала.

**Gradle (оба DSL) — Micrometer**

```groovy
dependencies {
    implementation "io.micrometer:micrometer-core"
    implementation "io.micrometer:micrometer-registry-prometheus"
    implementation "org.springframework.boot:spring-boot-starter-actuator"
}
```

```kotlin
dependencies {
    implementation("io.micrometer:micrometer-core")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

**Java — метрика размера OAS и health-проверка**

```java
package docs.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.boot.actuate.health.*;
import org.springframework.context.annotation.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.client.RestClient;

@Configuration
public class OpenApiMetrics {

    @Bean
    HealthIndicator oasHealth() {
        return () -> {
            try {
                String body = RestClient.create().get().uri("http://localhost:8080/_specs").retrieve().body(String.class);
                if (body == null || body.length() < 1024) return Health.down().withDetail("size", body == null ? 0 : body.length()).build();
                return Health.up().withDetail("size", body.length()).build();
            } catch (Exception e) {
                return Health.down(e).build();
            }
        };
    }

    @Bean
    ApplicationRunner metrics(MeterRegistry registry) {
        return args -> {
            registry.gauge("oas.size.bytes", this, s -> {
                try {
                    String body = RestClient.create().get().uri("http://localhost:8080/_specs").retrieve().body(String.class);
                    return body == null ? 0.0 : body.length();
                } catch (Exception e) {
                    return 0.0;
                }
            });
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package docs.metrics

import io.micrometer.core.instrument.MeterRegistry
import org.springframework.boot.ApplicationRunner
import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.client.RestClient

@Configuration
class OpenApiMetrics {

    @Bean
    fun oasHealth(): HealthIndicator = HealthIndicator {
        try {
            val body = RestClient.create().get().uri("http://localhost:8080/_specs").retrieve().body(String::class.java)
            if (body == null || body.length < 1024) Health.down().withDetail("size", body?.length ?: 0).build()
            else Health.up().withDetail("size", body.length).build()
        } catch (e: Exception) {
            Health.down(e).build()
        }
    }

    @Bean
    fun metrics(registry: MeterRegistry) = ApplicationRunner {
        registry.gauge("oas.size.bytes", this) {
            try {
                val body = RestClient.create().get().uri("http://localhost:8080/_specs").retrieve().body(String::class.java)
                (body?.length ?: 0).toDouble()
            } catch (e: Exception) { 0.0 }
        }
    }
}
```

**application.yml — публикация метрик/health**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus
  endpoint:
    health:
      show-details: when_authorized
```





