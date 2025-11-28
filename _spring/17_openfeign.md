---
layout: page
title: "OpenFeign: интеграции с другими сервисами/клиентами"
permalink: /spring/openfeign
---

<!-- TOC -->
* [0. Введение](#0-введение)
  * [Что такое Spring Cloud OpenFeign: декларативный HTTP-клиент поверх Feign, интегрированный со Spring Boot](#что-такое-spring-cloud-openfeign-декларативный-http-клиент-поверх-feign-интегрированный-со-spring-boot)
  * [Зачем это](#зачем-это)
  * [Где используется](#где-используется)
  * [Какие проблемы и задачи решает](#какие-проблемы-и-задачи-решает)
* [1. Позиционирование OpenFeign](#1-позиционирование-openfeign)
  * [Когда выбирать Feign vs RestTemplate/WebClient (синхронность, простота интерфейсов, интеграция с облачной экосистемой)](#когда-выбирать-feign-vs-resttemplatewebclient-синхронность-простота-интерфейсов-интеграция-с-облачной-экосистемой)
  * [Типовые сценарии: микросервисы, BFF, интеграция со сторонними API, генерация SDK не требуется](#типовые-сценарии-микросервисы-bff-интеграция-со-сторонними-api-генерация-sdk-не-требуется)
  * [Ограничения: блокирующая модель по умолчанию, внимание к тайм-аутам/пулу соединений](#ограничения-блокирующая-модель-по-умолчанию-внимание-к-тайм-аутампулу-соединений)
  * [Экосистема вокруг: Spring Cloud LoadBalancer, Resilience4j, Micrometer/OTel](#экосистема-вокруг-spring-cloud-loadbalancer-resilience4j-micrometerotel)
  * [Альтернативы/дополнения: WebClient для реактивных/стриминговых кейсов](#альтернативыдополнения-webclient-для-реактивныхстриминговых-кейсов)
* [2. Установка и базовая конфигурация](#2-установка-и-базовая-конфигурация)
  * [Зависимость: `spring-cloud-starter-openfeign`, включение `@EnableFeignClients`](#зависимость-spring-cloud-starter-openfeign-включение-enablefeignclients)
  * [Общее и per-client конфигурирование через `feign.client.config.default|{name}.*`](#общее-и-per-client-конфигурирование-через-feignclientconfigdefaultname)
  * [Выбор HTTP-клиента: Apache HttpClient или OkHttp (`feign.httpclient.enabled`/`feign.okhttp.enabled`)](#выбор-http-клиента-apache-httpclient-или-okhttp-feignhttpclientenabledfeignokhttpenabled)
  * [Базовые тайм-ауты: `connectTimeout`, `readTimeout`, дефолты и переопределения](#базовые-тайм-ауты-connecttimeout-readtimeout-дефолты-и-переопределения)
  * [Сжатие: `feign.compression.request/response.enabled`, лимиты размера](#сжатие-feigncompressionrequestresponseenabled-лимиты-размера)
  * [Переопределение бинов: `Encoder/Decoder/Contract/Logger/Retryer` через конфиг-класс](#переопределение-бинов-encoderdecodercontractloggerretryer-через-конфиг-класс)
* [3. Декларация клиентов (@FeignClient)](#3-декларация-клиентов-feignclient)
  * [Атрибуты: `name`, `url`, `path`, `configuration`, `decode404`](#атрибуты-name-url-path-configuration-decode404)
  * [Поддержка Spring MVC аннотаций: `@Get/Post/Put/DeleteMapping`, `@RequestParam`, `@PathVariable`, `@RequestHeader`, `@RequestBody`](#поддержка-spring-mvc-аннотаций-getpostputdeletemapping-requestparam-pathvariable-requestheader-requestbody)
  * [Динамические параметры: `@SpringQueryMap/@QueryMap`, `Map<String, ?>` заголовков и квери](#динамические-параметры-springquerymapquerymap-mapstring--заголовков-и-квери)
  * [Мультимедиа: `consumes/produces`, файлы (`MultipartFile`, `@RequestPart`)](#мультимедиа-consumesproduces-файлы-multipartfile-requestpart)
  * [Возвраты: `ResponseEntity<T>`, `byte[]`, `InputStream/Resource` для стриминга](#возвраты-responseentityt-byte-inputstreamresource-для-стриминга)
  * [Версионирование/базовые пути: константы, `path` на уровне клиента + относительные пути методов](#версионированиебазовые-пути-константы-path-на-уровне-клиента--относительные-пути-методов)
* [4. Серилизация/десериализация и схемы данных](#4-серилизациядесериализация-и-схемы-данных)
  * [Jackson encoder/decoder по умолчанию, настройка ObjectMapper (JavaTime, `PropertyNamingStrategy`)](#jackson-encoderdecoder-по-умолчанию-настройка-objectmapper-javatime-propertynamingstrategy)
  * [Явные модели ошибок (Problem+JSON) и десериализация тела в `ErrorDecoder`](#явные-модели-ошибок-problemjson-и-десериализация-тела-в-errordecoder)
  * [Кастомные `Encoder/Decoder` (например, protobuf/CBOR) и выбор per-client](#кастомные-encoderdecoder-например-protobufcbor-и-выбор-per-client)
  * [Multipart/формы: `SpringFormEncoder`, корректные `Content-Type` и границы](#multipartформы-springformencoder-корректные-content-type-и-границы)
  * [Большие ответы и стриминг: `InputStream`/`Response` без загрузки в память](#большие-ответы-и-стриминг-inputstreamresponse-без-загрузки-в-память)
  * [Контент-негациация, локализация сообщений и примеры для документации](#контент-негациация-локализация-сообщений-и-примеры-для-документации)
* [5. Тайм-ауты, пулы и ретраи](#5-тайм-ауты-пулы-и-ретраи)
  * [Настройка соединений: размеры пулов, keep-alive, HTTP/2 (через OkHttp где уместно)](#настройка-соединений-размеры-пулов-keep-alive-http2-через-okhttp-где-уместно)
  * [`connectTimeout` vs `readTimeout` vs socket timeouts на уровне HTTP-клиента](#connecttimeout-vs-readtimeout-vs-socket-timeouts-на-уровне-http-клиента)
  * [Повторы: Feign `Retryer` (экспоненциальная пауза) vs Resilience4j Retry (более гибкий контроль)](#повторы-feign-retryer-экспоненциальная-пауза-vs-resilience4j-retry-более-гибкий-контроль)
  * [Идемпотентность: безопасные методы/тела, ключи идемпотентности для POST](#идемпотентность-безопасные-методытела-ключи-идемпотентности-для-post)
  * [Пер-клиентные политики (per-name в `application.yml`) и «жёсткие» ограничения](#пер-клиентные-политики-per-name-в-applicationyml-и-жёсткие-ограничения)
  * [Тайм-ауты на уровне бизнес-логики (Future/CompletableFuture) — когда НЕ нужно](#тайм-ауты-на-уровне-бизнес-логики-futurecompletablefuture--когда-не-нужно)
* [6. Ошибки и устойчивость](#6-ошибки-и-устойчивость)
  * [`ErrorDecoder`: маппинг HTTP-статусов и тел ошибок в доменные исключения](#errordecoder-маппинг-http-статусов-и-тел-ошибок-в-доменные-исключения)
  * [Circuit Breaker: `spring-cloud-starter-circuitbreaker-resilience4j` + Feign (альтернатива устаревшему Hystrix)](#circuit-breaker-spring-cloud-starter-circuitbreaker-resilience4j--feign-альтернатива-устаревшему-hystrix)
  * [Fallback/FallbackFactory: деградация сервисов с доступом к причине ошибки](#fallbackfallbackfactory-деградация-сервисов-с-доступом-к-причине-ошибки)
  * [Категоризация: сетевые/временные ошибки vs бизнес-ошибки (4xx) — разные реакции](#категоризация-сетевыевременные-ошибки-vs-бизнес-ошибки-4xx--разные-реакции)
  * [Политика повторов/тайм-аутов/CB для разных эндпоинтов (пер-метод через конфиг)](#политика-повторовтайм-аутовcb-для-разных-эндпоинтов-пер-метод-через-конфиг)
  * [Единый обработчик и телеметрия ошибок, correlation-id в исключениях](#единый-обработчик-и-телеметрия-ошибок-correlation-id-в-исключениях)
* [7. Безопасность и пропагация контекста](#7-безопасность-и-пропагация-контекста)
  * [OAuth2 Client Credentials: `OAuth2AuthorizedClientManager` + `RequestInterceptor` для Bearer-токена](#oauth2-client-credentials-oauth2authorizedclientmanager--requestinterceptor-для-bearer-токена)
  * [Пропагация заголовков: `Authorization`, `X-Request-Id`, трассировка (B3/W3C `traceparent`)](#пропагация-заголовков-authorization-x-request-id-трассировка-b3w3c-traceparent)
  * [mTLS: конфиг SSL-контекста для Apache/OkHttp, хранилища сертификатов и пины](#mtls-конфиг-ssl-контекста-для-apacheokhttp-хранилища-сертификатов-и-пины)
  * [Basic/API-Key: `RequestInterceptor` или свойства клиента](#basicapi-key-requestinterceptor-или-свойства-клиента)
  * [Маскирование чувствительных данных в логах (`Logger.Level`, custom Logger)](#маскирование-чувствительных-данных-в-логах-loggerlevel-custom-logger)
  * [CORS/CSRF не релевантны клиенту, но важно соответствие политике провайдера](#corscsrf-не-релевантны-клиенту-но-важно-соответствие-политике-провайдера)
* [8. Сервис-дискавери и балансировка](#8-сервис-дискавери-и-балансировка)
  * [Разрешение `name` через Spring Cloud LoadBalancer (Eureka/Consul, без Ribbon)](#разрешение-name-через-spring-cloud-loadbalancer-eurekaconsul-без-ribbon)
  * [Политики выбора инстанса: round-robin/Random/Zone-aware, кэш TTL](#политики-выбора-инстанса-round-robinrandomzone-aware-кэш-ttl)
  * [Health-aware вызовы и отказоустойчивость при частичных сбоях](#health-aware-вызовы-и-отказоустойчивость-при-частичных-сбоях)
  * [Совместимость с Gateway: токен-релэй, маршрутизация/переписывание путей](#совместимость-с-gateway-токен-релэй-маршрутизацияпереписывание-путей)
  * [Статический `url` для внешних API и отключение LB при необходимости](#статический-url-для-внешних-api-и-отключение-lb-при-необходимости)
  * [Разделение конфигураций для внутренних и внешних клиентов](#разделение-конфигураций-для-внутренних-и-внешних-клиентов)
* [9. Тестирование и контракты](#9-тестирование-и-контракты)
  * [Локальные тесты с WireMock/MockWebServer: сценарии 2xx/4xx/5xx/тайм-ауты](#локальные-тесты-с-wiremockmockwebserver-сценарии-2xx4xx5xxтайм-ауты)
  * [Срезы: загрузка контекста с конкретным `@FeignClient`, моки зависимостей](#срезы-загрузка-контекста-с-конкретным-feignclient-моки-зависимостей)
  * [Контрактные тесты (Spring Cloud Contract/Pact) для провайдеров/консьюмеров](#контрактные-тесты-spring-cloud-contractpact-для-провайдеровконсьюмеров)
  * [Тестирование безопасности: стабы IdP, подмена токена в интерсепторе](#тестирование-безопасности-стабы-idp-подмена-токена-в-интерсепторе)
  * [Интеграционные тесты: Testcontainers для провайдера (если это ваш сервис)](#интеграционные-тесты-testcontainers-для-провайдера-если-это-ваш-сервис)
  * [Golden-files/snapshot-подход для стабилизации полезных нагрузок](#golden-filessnapshot-подход-для-стабилизации-полезных-нагрузок)
* [10. Наблюдаемость, производительность и эксплуатация](#10-наблюдаемость-производительность-и-эксплуатация)
  * [Логи: `feign.Logger` уровни, корреляция запрос-ответ, маскирование секретов](#логи-feignlogger-уровни-корреляция-запрос-ответ-маскирование-секретов)
  * [Метрики: Micrometer таймеры/счётчики per-client/method/status, export в Prometheus](#метрики-micrometer-таймерысчётчики-per-clientmethodstatus-export-в-prometheus)
  * [Трейсинг: OpenTelemetry автоинструментация, propagation context](#трейсинг-opentelemetry-автоинструментация-propagation-context)
  * [Производительность: пулы, re-use соединений, gzip, минимизация JSON (модули Jackson)](#производительность-пулы-re-use-соединений-gzip-минимизация-json-модули-jackson)
  * [Паттерны: bulkhead (ограничение параллелизма), очереди/батчинг, кэширование ответов](#паттерны-bulkhead-ограничение-параллелизма-очередибатчинг-кэширование-ответов)
  * [Чек-листы продакшна: тайм-ауты, retries, CB, алерты по SLA, политика ротации ключей](#чек-листы-продакшна-тайм-ауты-retries-cb-алерты-по-sla-политика-ротации-ключей)
<!-- TOC -->

# 0. Введение

## Что такое Spring Cloud OpenFeign: декларативный HTTP-клиент поверх Feign, интегрированный со Spring Boot

Spring Cloud OpenFeign — это надстройка над библиотекой Feign, которая позволяет описывать HTTP-клиентов в виде обычных Java/Kotlin-интерфейсов с аннотациями в стиле Spring MVC. Вы пишете сигнатуры методов, добавляете `@GetMapping`, `@PostMapping`, `@RequestParam`, и Spring во время старта генерирует прокси-реализацию: она сериализует запрос, устанавливает заголовки, вызывает удалённый HTTP-сервис и десериализует ответ обратно в ваши DTO.

Ключевая ценность — декларативность и интеграция с Spring Boot 3.x и Spring Cloud 2023.x: вы получаете автоконфигурацию, «склеивание» с контекстом, единые `ObjectMapper`, валидаторы, логгеры, бины retry/circuit-breaker, а также типизированные ошибки. Это позволяет сосредоточиться на контракте и не писать рутину для каждого клиента.

OpenFeign сам по себе не «реактивный» — он использует выбранный блокирующий HTTP-клиент (Apache HttpClient или OkHttp). Взамен вы получаете простые интерфейсы и понятное поведение потоков: вызов — это обычная синхронная функция, которую легко читать, профилировать и ограничивать тайм-аутами. Для реактивных кейсов уместнее `WebClient`, но в подавляющем большинстве интеграций CRUD-стиля Feign отлично подходит.

Интеграция со Spring Cloud добавляет удобства на уровне облачной экосистемы: сервис-дискавери, балансировка (Spring Cloud LoadBalancer), устойчивость (Resilience4j), метрики (Micrometer), трейсинг (OTel), единые политики тайм-аутов и повторов, маскирование секретов в логах. Всё это конфигурируется через `application.yml` по шаблону «per-client».

Декларативная модель уменьшает «скрытые» дефекты. Вместо того чтобы руками собирать URL и заголовки в десятках мест, вы стандартизируете это в одном интерфейсе и конфиге. Это облегчает ревью и аудит безопасности: понятно, какой метод ходит куда, какими методами, с какими заголовками.

OpenFeign поддерживает богатые данные: JSON (Jackson), формы, multipart (загрузка файлов), байтовые потоки для больших ответов, а также «низкоуровневый» доступ к `Response`, если нужно тонко управлять телом и заголовками. Вы можете подключить альтернативные кодеки (например, protobuf/CBOR) через собственные `Encoder/Decoder`.

Уровень логирования и перехватчики (`RequestInterceptor`) позволяют централизованно внедрять заголовки (токены, correlation-id), политики идемпотентности, а также маскирование чувствительных значений. Это особенно важно в продакшне, чтобы логи помогали, а не становились источником утечек.

Поскольку Feign — это «контракт-первый» подход, он хорошо сочетается с API-дизайном по OpenAPI/Swagger: вы можете генерировать DTO из спецификаций и использовать их прямо в сигнатурах клиентов. Генерацию самого клиента тоже можно автоматизировать, но во многих командах интерфейсы пишут вручную — из-за гибкости и читаемости.

Эволюция контрактов проще, когда клиенты типизированы: рефакторинг и статический анализ ловят несовпадения сигнатур и DTO во время компиляции. В случае «сырого» HTTP-кода или строковых билдеров многие ошибки проявляются только на рантайме.

Порог входа для Junior/Middle-разработчиков невысокий: знание Spring MVC аннотаций и базовой сетевой семантики (методы, коды статусов, заголовки) достаточно, чтобы уверенно писать и сопровождать клиентов. Это ускоряет delivery и снижает когнитивную нагрузку в микросервисных системах.

Наконец, OpenFeign — это стандарт де-факто в Spring-микросервисах для синхронной межсервисной коммуникации. Он не заменяет все инструменты, но закрывает «80% повседневной интеграции» простым и безопасным способом.

**Gradle зависимости (оба DSL)**
*Groovy DSL*

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
}
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.3"
    }
}
```

*Kotlin DSL*

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.24" apply false
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}
dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.3")
    }
}
```

**application.yml — минимальный скелет**

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
        response:
          enabled: true
```

**Java — простой клиент и включение OpenFeign**

```java
package com.example.clients;

import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@SpringBootApplication
@EnableFeignClients
public class App { }

@FeignClient(name = "users", url = "${clients.users.url}")
interface UsersClient {
    @GetMapping("/users/{id}")
    UserDto findById(@PathVariable("id") Long id);
}

record UserDto(Long id, String name, String username) { }
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.cloud.openfeign.EnableFeignClients
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@SpringBootApplication
@EnableFeignClients
class App

@FeignClient(name = "users", url = "\${clients.users.url}")
interface UsersClient {
    @GetMapping("/users/{id}")
    fun findById(@PathVariable("id") id: Long): UserDto
}

data class UserDto(val id: Long, val name: String, val username: String)
```

---

## Зачем это

Главная причина использовать OpenFeign — снижение бойлерплейта и повышение читаемости. Вместо ручной сборки HTTP-запросов в каждом сервисе вы оформляете контракт единожды, а реализацию доверяете фреймворку. Это не только экономит время, но и делает код предсказуемым: любой разработчик моментально понимает, что делает метод клиента.

Вторая причина — согласованность конфигурации. Тайм-ауты, пулы соединений, повторные попытки, circuit-breaker, логирование и метрики вы централизуете. При инцидентах это позволяет быстро «подкрутить» политику без переписывания кода, просто меняя `application.yml` или небольшие конфиг-классы.

Третья — типобезопасность. DTO и сигнатуры методов проверяются компилятором. Если провайдер API поменял поле или формат, вы увидите ошибку на этапе сборки (или тестов), а не в продакшне. Это резко повышает надёжность интеграций в системах с десятками внешних/внутренних зависимостей.

Четвёртая — интегрируемость. OpenFeign «из коробки» дружит с Spring Cloud компонентами: балансировщик, трейсинг, метрики. Вы не изобретаете велосипед по корреляции запросов, а получаете наблюдаемость уровня «приложение ↔ зависимость» почти бесплатно.

Пятая — тестируемость. Класс-интерфейс легко подменить моками, а end-to-end поведение удобно проверять с WireMock/MockWebServer. Декларативная форма, опять же, уменьшает количество скрытых зависимостей, и тесты становятся короче и стабильнее.

Шестая — контроль производительности. Блокирующая модель не всегда минус; она хороша там, где latency умеренная, а пиковая конкуррентность контролируема. Понимание «один запрос — один поток» упрощает профилирование и планирование ресурсов в JVM.

Седьмая — безопасность. Политики добавления токенов/заголовков, маскировка секретов и запрещённых логов централизуются через `RequestInterceptor` и `Logger`. Это снижает риск случайной утечки токенов или персональных данных при отладке.

Восьмая — дисциплина контрактов. Интерфейсы — хорошее место для документации и примеров: описывайте `produces/consumes`, фиксируйте версии путей, оставляйте Javadoc с ссылкой на схему OpenAPI. Это превращает клиента в «живую» документацию.

Девятая — ускорение онбординга. Новичку проще открыть пакет `clients.*` и увидеть все интеграции как на ладони, чем разбираться в зоопарке утилитных HTTP-вызовов, разбросанных по слоям.

Десятая — трансформация в масштабируемую практику. Когда все клиенты оформлены одинаково, переезд на новые версии Spring/Cloud, изменение политики ретраев или балансировки — это массовое, но простое изменение, а не сотни уникальных патчей.

**Java — сравнение: Feign против RestTemplate (идея «зачем»)**

```java
package com.example.clients;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "catalog", url = "${clients.catalog.url}")
interface CatalogClient {
    @GetMapping("/api/v1/items")
    ItemsPage list();
}

// Вариант без Feign требовал бы ручной сборки URL, Headers и парсинга JSON каждой командой.
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "catalog", url = "\${clients.catalog.url}")
interface CatalogClient {
    @GetMapping("/api/v1/items")
    fun list(): ItemsPage
}

data class ItemsPage(val items: List<String>)
```

---

## Где используется

Чаще всего OpenFeign применяют в микросервисной архитектуре для синхронных запросов «бэкенд ↔ бэкенд»: создание/чтение сущностей, проверка статусов, расчёты, получение справочников. Здесь важна понятность, трассируемость и единая конфигурация тайм-аутов — OpenFeign отвечает этим требованиям.

В BFF-слое (Backend For Frontend) Feign удобен для агрегации данных из нескольких внутренних сервисов. BFF часто живёт близко к клиентам и выполняет модель «собери карточку за один запрос», вызывая 2–4 внутренних API параллельно. Простые интерфейсы клиентов делают код BFF лаконичным.

Интеграции со сторонними API — ещё один типичный сценарий: платёжные шлюзы, геокодеры, SMS/Push-провайдеры, антифрод. С Feign удобно инкапсулировать нюансы заголовков, подписей, форматов тел и ретраев в одном месте, а сервисам отдать чистый доменный интерфейс.

Внутри корпоративной сети Feign хорошо сочетается со Spring Cloud LoadBalancer/Eureka/Consul: вы указываете `name`, а резолвинг инстансов происходит автоматически. Это уменьшает жёсткие привязки к URL, упрощает blue/green и canary-релизы.

В системах, где нужен строгий контроль SLA, Feign помогает прогнать «сквозные» тайм-ауты и метрики: вы видите на дашбордах долю 4xx/5xx, p95/p99 на метод и клиента, и можете сравнительно просто применять политику деградации (fallback) при проблемах у провайдера.

В связке с Spring Security/OAuth2 клиенты получают Bearer-токены автоматически и прокидывают correlation-id/request-id. Это упрощает аудит и расследование инцидентов: у вас есть «сквозной след» от фронта до провайдеров.

Даже если у вас есть OpenAPI-спецификации и генерация SDK, Feign остаётся удобным «ручным» слоем: часто generated-клиенты громоздки, а Feign-интерфейсы — компактны и читабельны. Комбинация «генерим DTO + пишем тонкий Feign-интерфейс» — практичный баланс.

В монорепозиториях Feign помогает унифицировать интеграции: общие конфиги/интерсепторы, единый логгер, единый принцип именования клиентов и пакетов. Это снижает стоимость сопровождения и облегчает массовые миграции.

В сценариях потребления больших ответов (экспорт, отчёты) Feign можно настроить на потоковое чтение `InputStream` и передачу дальше в S3/MinIO, не загружая всё в память. Это критично для устойчивости под нагрузкой.

И наконец, OpenFeign часто используют как «синхронный слой» даже там, где часть трафика асинхронна (Kafka/RabbitMQ): вам всё равно нужен удобный способ сделать запрос-ответ, и Feign органично дополняет события.

**Java — BFF-агрегатор с двумя клиентами**

```java
package com.example.bff;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.stereotype.Service;

@FeignClient(name = "users", url = "${clients.users.url}")
interface UsersClient { @GetMapping("/users/me") User me(); }

@FeignClient(name = "orders", url = "${clients.orders.url}")
interface OrdersClient { @GetMapping("/orders/recent") Orders recent(); }

@Service
class ProfileService {
    private final UsersClient users; private final OrdersClient orders;
    ProfileService(UsersClient u, OrdersClient o) { this.users = u; this.orders = o; }
    Profile aggregate() { return new Profile(users.me(), orders.recent()); }
}

record User(Long id, String name) { }
record Orders(java.util.List<String> items) { }
record Profile(User user, Orders orders) { }
```

**Kotlin — эквивалент**

```kotlin
package com.example.bff

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "users", url = "\${clients.users.url}")
interface UsersClient { @GetMapping("/users/me") fun me(): User }

@FeignClient(name = "orders", url = "\${clients.orders.url}")
interface OrdersClient { @GetMapping("/orders/recent") fun recent(): Orders }

@Service
class ProfileService(private val users: UsersClient, private val orders: OrdersClient) {
    fun aggregate(): Profile = Profile(users.me(), orders.recent())
}

data class User(val id: Long, val name: String)
data class Orders(val items: List<String>)
data class Profile(val user: User, val orders: Orders)
```

---

## Какие проблемы и задачи решает

Первая проблема — рассеянность HTTP-логики по коду. Feign концентрирует контракт в одном интерфейсе и подключаемых конфигурациях. В результате проще видеть, какие интеграции существуют, и как они устроены: сигнатуры методов — это «карта» внешних связей сервиса.

Вторая — управление тайм-аутами и ретраями. Без стандарта один разработчик ставит `5s`, другой — `30s`, кто-то вовсе забывает. OpenFeign даёт единый слой для тайм-аутов, «пер-клиентных» политик и централизованных бинов `Retryer`/Resilience4j, что резко уменьшает риск «цепной реакции» при деградации провайдера.

Третья — качество ошибок. «Сырые» HTTP-ошибки неудобны для бизнес-логики. С `ErrorDecoder` вы превращаете 4xx/5xx (и их тела) в доменные исключения с кодами и контекстом, а значит — внятно обрабатываете их наверху и формируете стабильный ответ потребителю.

Четвёртая — безопасность и пропагация контекста. Токены, `X-Request-Id`, заголовки трассировки — всё это добавляется перехватчиками и живёт в одном месте. Вам не нужно помнить «в каждом вызове» добавить нужные заголовки — это делает инфраструктурный слой.

Пятая — наблюдаемость. Feign интегрируется с Micrometer/OTel «из коробки»: у вас появляются таймеры с тегами client/method/status и спаны, связанные с входящим запросом. Это облегчает SRE-практики: видны p95/p99 по методам и всплески ошибок.

Шестая — стандартизация сериализации. Единый `ObjectMapper` (модули JavaTime, стратегии именования), единые правила контент-негациации, единые DTO — меньше сюрпризов на конвертации и меньше «ручных» патчей форматов.

Седьмая — управление размерами и потоками. Для больших ответов вы можете переключиться на потоковое чтение `InputStream` и избежать OOM. Для загрузки файлов — использовать корректный `SpringFormEncoder`. Эти практики сложно поддерживать «вручную», а в Feign это часть стандарта.

Восьмая — совместимость и эволюция. Переезд на новый HTTP-клиент (OkHttp/Apache) — это изменение пропертей, а не переписывание всего слоя. Добавление компрессии, HTTP/2, keep-alive — тоже вопрос конфигурации, а не повсеместного рефакторинга.

Девятая — контроль идемпотентности. Для повторов вы можете внедрять ключи идемпотентности через перехватчик и применять ретраи только для безопасных методов. Это решает класс проблем «дублированные списания/платежи» при сетевых сбоях.

Десятая — ускорение разработки. В итоге команда тратит время на доменную логику, а не на вязание `HttpEntity`, `Headers` и JSON-парсинг. Стоимость сопровождения падает, предсказуемость растёт — это и есть практическая цель.

**application.yml — задачи, решаемые конфигом (timeouts, логирование)**

```yaml
feign:
  httpclient:
    enabled: true
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
        loggerLevel: BASIC
      users:
        connectTimeout: 500
        readTimeout: 1500
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          min-request-size: 1024
        response:
          enabled: true
```

**Java — ErrorDecoder и перехватчик заголовков**

```java
package com.example.clients.config;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import feign.codec.ErrorDecoder;
import feign.Response;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignSupportConfig {

    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            if (response.status() == 404) return new ResourceNotFound("Not found: " + methodKey);
            return new FeignDefaultException("HTTP " + response.status());
        };
    }

    @Bean
    public RequestInterceptor correlation() {
        return new RequestInterceptor() {
            @Override public void apply(RequestTemplate template) {
                template.header("X-Request-Id", java.util.UUID.randomUUID().toString());
            }
        };
    }

    public static class ResourceNotFound extends RuntimeException {
        public ResourceNotFound(String m) { super(m); }
    }
    public static class FeignDefaultException extends RuntimeException {
        public FeignDefaultException(String m) { super(m); }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients.config

import feign.RequestInterceptor
import feign.RequestTemplate
import feign.codec.ErrorDecoder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.UUID

@Configuration
class FeignSupportConfig {

    @Bean
    fun errorDecoder(): ErrorDecoder = ErrorDecoder { methodKey, response ->
        if (response.status() == 404) ResourceNotFound("Not found: $methodKey")
        else FeignDefaultException("HTTP ${response.status()}")
    }

    @Bean
    fun correlation(): RequestInterceptor = object : RequestInterceptor {
        override fun apply(template: RequestTemplate) {
            template.header("X-Request-Id", UUID.randomUUID().toString())
        }
    }
}

class ResourceNotFound(message: String) : RuntimeException(message)
class FeignDefaultException(message: String) : RuntimeException(message)
```

# 1. Позиционирование OpenFeign

## Когда выбирать Feign vs RestTemplate/WebClient (синхронность, простота интерфейсов, интеграция с облачной экосистемой)

Если у вас классический синхронный бэкенд-к-бэкенду вызов (CRUD, справочники, проверки статусов) и нет требований к потоковой передаче или десяткам тысяч параллельных коннектов, **Feign — первое, что стоит рассмотреть**. Он даёт декларативные интерфейсы наподобие Spring MVC контроллеров: вместо ручной сборки URL/заголовков вы описываете контракт, а инфраструктура берёт остальное на себя. Это снижает бойлерплейт и повышает читаемость — особенно в микросервисах, где таких клиентов десятки.

**RestTemplate** исторически решал похожую задачу, но он императивен и низкоуровневее: много кода с `HttpEntity`, `UriComponentsBuilder`, явной (де)сериализацией. Он стабилен, но в новых проектах считается «уступающим» Feign по эргономике и интеграции с облачной экосистемой Spring Cloud (балансировка, CB, ретраи). Если в кодовой базе уже есть RestTemplate, миграция на Feign обычно проходит безболезненно за счёт интерфейсов и `@FeignClient`.

**WebClient** — реактивный инструмент. Он нужен, когда у вас потоковые ответы (SSE, WebSocket мосты), тысячи одновременных исходящих соединений, «лотки» параллельных запросов к нескольким бэкендам, где важна эффективность на уровне event-loop. Цена — повышенная сложность: вы должны управлять подписками, backpressure и тайм-аутами, избегая блокировок. Для 80% синхронных интеграций WebClient избыточен, а Feign проще и безопаснее.

С точки зрения **интеграции с Spring Cloud**, Feign выигрывает: у него «из коробки» дружба со Spring Cloud LoadBalancer (по `name:`), поддержка Spring Cloud CircuitBreaker (Resilience4j), общие `ObjectMapper`/валидация/логгинг, единая конфигурация через `application.yml` c `feign.client.config`. Для RestTemplate/WebClient приходится больше собирать руками или подтягивать стартеры/билдеры.

По **операционной модели** Feign — блокирующий. Это понятнее для профайлинга и планирования ресурсов: «один поток — один запрос». При небольших пиках и обычных SLA это плюс: проще держать под контролем пул, выставлять понятные тайм-ауты и предсказуемо реагировать на деградации. В реактивной модели проще масштабируемость, но сложнее диагностика.

Не забывайте о **границах применимости**. Если вы агрегируете десятки источников и отдаёте стрим клиенту, имеет смысл комбинировать: Feign для CRUD-части и WebClient для «длинных»/стриминговых историй. Это нормальная практика: выбирайте инструмент под задачу, а не «один молоток на всё».

С точки зрения **поддержки командой** (онбординг, ревью, аудит безопасности) декларативные интерфейсы Feign — отличная документация: в пакете `clients.*` видно, «кто куда ходит» и какими методами. Это ускоряет ревью и упрощает внедрение обязательных заголовков (авторизация, корреляция) через единый `RequestInterceptor`.

При выборе учитывайте и **экосистему вокруг провайдера**. Если на вашей платформе принят Eureka/Consul и Spring Cloud, Feign почти всегда будет «правильным» соответствием стандартам. Если в компании доминируют реактивные API и стримы, логичнее начинать с WebClient, а Feign оставить для нишевых синхронных вызовов.

**Производительность** в типовом CRUD-кейсе у Feign упирается в провайдера/сеть, а не в клиента. Репрезентативные нагрузки показывают, что при корректных пулах и keep-alive Feign обслуживает тысячи RPS при умеренных латентностях. Если нужна экстремальная конкуррентность — переключайтесь на WebClient/Netty и реактивную композицию.

**Управляемость** — ещё один плюс Feign: конфиг из `application.yml` (пер-клиент, пер-метод через `configKey`) позволяет быстро «прикрутить» тайм-ауты, ретраи, fallback и маскирование логов без рефакторинга. При инцидентах вы можете оперативно уменьшить пулы, сократить тайм-ауты или полностью «замкнуть» неустойчивого провайдера.

Итог прост: если не нужны реактивные особенности, **берите Feign**. Если нужны — **берите WebClient**. RestTemplate оставьте для legacy и постепенной миграции. Комбинируйте инструменты, не бойтесь гетерогенного стека, но стандартизируйте конфиги и телеметрию.

**Gradle зависимости (используются во всех примерах ниже)**
*Groovy DSL*

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
}
dependencyManagement {
    imports { mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.3" }
}
```

*Kotlin DSL*

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.24" apply false
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}
dependencyManagement {
    imports { mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.3") }
}
```

**Java — Feign vs WebClient короткая иллюстрация**

```java
package com.example.positioning;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;

@FeignClient(name = "users", url = "${clients.users.url}")
interface UsersFeign {
    @GetMapping("/users/me")
    User me();
}

record User(Long id, String name) {}

@Service
class ProfileService {
    private final UsersFeign feign;
    private final WebClient webClient = WebClient.create();

    ProfileService(UsersFeign feign) { this.feign = feign; }

    public User viaFeign() { return feign.me(); } // синхронно, декларативно

    public User viaWebClient() {                  // реактивно, больше контроля
        return webClient.get().uri("https://api.example.com/users/me")
            .retrieve().bodyToMono(User.class).block();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.reactive.function.client.WebClient

@FeignClient(name = "users", url = "\${clients.users.url}")
interface UsersFeign {
    @GetMapping("/users/me")
    fun me(): User
}

data class User(val id: Long, val name: String)

@Service
class ProfileService(private val feign: UsersFeign) {
    private val webClient = WebClient.create()

    fun viaFeign(): User = feign.me()
    fun viaWebClient(): User =
        webClient.get().uri("https://api.example.com/users/me")
            .retrieve().bodyToMono(User::class.java).block()!!
}
```

---

## Типовые сценарии: микросервисы, BFF, интеграция со сторонними API, генерация SDK не требуется

В микросервисной архитектуре сервисы часто обращаются к «соседям» с короткими запросами: получить статус, создать заказ, проверить лимит. **Feign даёт быстрый путь**: написать интерфейс, прикрутить свойства, готово. Это сокращает TTM и упрощает ревью — особенно когда десятки сервисов делятся единообразными клиентами.

В BFF-слое удобно агрегировать данные из нескольких внутренних API. Здесь ценится лаконичность и типизация: сигнатура метода «говорит», что нужен пользователь и его последние заказы. С Feign это два вызова без «плетения» HTTP-рутины; композицию и тайм-ауты берёт на себя сервисный слой.

При интеграциях со сторонними системами (платёжки, геокодинг, почтовые шлюзы) Feign инкапсулирует нюансы заголовков, подписи, особенностей тел и кодировок. Внутренним сервисам вы отдаёте чистый доменный интерфейс, скрывая «грязь» интеграции за клиентом и его конфигом.

Генерация SDK по OpenAPI — полезный инструмент, но на практике **ручные Feign-интерфейсы часто удобнее**: меньше слоёв абстракций, проще адаптировать специфические заголовки/аутентификацию/логирование. Комбинируйте: генерируйте DTO из схем, а интерфейсы пишите сами.

Типовой кейс — **fan-out** к двум-трём зависимостям, где важна предсказуемость. Вы ставите тайм-ауты, ограничиваете параллелизм (через пул/Executor), слепляете результат и отдаёте клиенту. С Feign это естественно: блокирующая модель и декларативные интерфейсы помогают держать процесс под контролем.

В anti-fraud/идентификационных сценариях часто требуется логировать каждый вызов и хранить корреляционные ID. В Feign это решается одним `RequestInterceptor`: добавили `X-Request-Id`, маскируете чувствительные заголовки — и проблема закрыта без модификации десятков мест.

Если нужна **частичная деградация**, Feign легко сочетается с CircuitBreaker/Retry. Для не-критичных разделов (например, рекомендации) можно вернуть «пустые» данные или кэш, не валя весь запрос. Это важный практический паттерн для BFF.

Когда внешние API **ограничивают RPS** или взимают плату за каждый запрос, полезно добавить кэширование на уровне сервисного метода. Feign-клиент остаётся тонким, а кэш — прозрачным для остального кода. Это экономит бюджет и повышает стабильность.

Для больших ответов (экспорты, отчёты) Feign поддерживает **стриминг** через `InputStream`/`Resource`. Вы можете «прокачать» файл в S3/MinIO без загрузки в память. Такой сценарий встречается в отчётности и миграциях данных.

Наконец, у корпоративных команд ценится **унификация**: Feign-клиенты в одном пакете, единые конфиги, стандартные заголовки, чек-листы качества. Это превращает интеграции из «индивидуального творчества» в воспроизводимую инженерную практику.

**Java — BFF-агрегатор над двумя Feign-клиентами**

```java
package com.example.bff;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "users", url = "${clients.users.url}")
interface UsersClient { @GetMapping("/users/me") User me(); }

@FeignClient(name = "orders", url = "${clients.orders.url}")
interface OrdersClient { @GetMapping("/orders/recent") Orders recent(); }

record User(Long id, String name) {}
record Orders(java.util.List<String> items) {}
record Profile(User user, Orders orders) {}

@Service
class ProfileService {
    private final UsersClient users; private final OrdersClient orders;
    ProfileService(UsersClient u, OrdersClient o) { this.users = u; this.orders = o; }
    Profile aggregate() { return new Profile(users.me(), orders.recent()); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.bff

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "users", url = "\${clients.users.url}")
interface UsersClient { @GetMapping("/users/me") fun me(): User }

@FeignClient(name = "orders", url = "\${clients.orders.url}")
interface OrdersClient { @GetMapping("/orders/recent") fun recent(): Orders }

data class User(val id: Long, val name: String)
data class Orders(val items: List<String>)
data class Profile(val user: User, val orders: Orders)

@Service
class ProfileService(private val users: UsersClient, private val orders: OrdersClient) {
    fun aggregate(): Profile = Profile(users.me(), orders.recent())
}
```

---

## Ограничения: блокирующая модель по умолчанию, внимание к тайм-аутам/пулу соединений

Первое ограничение Feign — **блокирующая модель**. Каждый вызов занимает поток на время сетевого обмена. Это не «плохо», но требует дисциплины: ограничивайте количество исходящих запросов, настраивайте пулы и тайм-ауты, не допускайте бесконечного ожидания зависимостей.

Второе — **тайм-ауты**: недостаток или отсутствие тайм-аутов приводит к «зависшим» потокам, росту очередей и лавинообразным отказам. Минимум: connect/read timeout на уровне клиента и общий «бизнес-тайм-аут» на метод (если нужно). Значения подбирайте на стендах по SLA провайдера.

Третье — **пулы соединений**. Apache HttpClient/OkHttp экономят на установке TCP/TLS, но при слишком маленьком пуле получите конкуренцию и p99; при слишком большом — распухшие ресурсы и «скрытые» хвосты. Следите за метриками пула и ищите баланс, а не максимум.

Четвёртое — **keep-alive**. С ним хорошо, но он должен иметь ограничение idle-времени. «Мёртвые» коннекты занимают дескрипторы; слишком агрессивные idle-тайм-ауты — наоборот, приведут к частому re-connect и росту CPU/RTT.

Пятое — **ретраи**. Они оправданы только для идемпотентных операций (GET, HEAD) и некоторых POST с ключом идемпотентности. «Слепые» ретраи на 5xx/тайм-аут без circuit-breaker и лимита конкуррентности способны убить и вас, и провайдера.

Шестое — **большие тела**. По умолчанию JSON читается в память. Для гигабайтных ответов это риск OOM. Используйте `Response`/`InputStream` и передавайте поток дальше (в файловое хранилище) без буферизации.

Седьмое — **логирование**. Уровни `Logger.Level` выше BASIC полезны, но опасны: легко уронить производительность и «утечь» секретами в логи. Маскируйте токены и персональные данные кастомным логгером или фильтрами.

Восьмое — **TLS и сертификаты**. При работе с внешним интернетом корректно настройте trust-store и hostname verification. На OkHttp проще включить HTTP/2, но убедитесь в поддержке со стороны провайдера.

Девятое — **обновления библиотек**. Смена версии Apache/OkHttp может менять поведение пула/тайм-аутов. Тестируйте на стендах, следите за регрессами перфа и корректируйте конфиги.

Десятое — **профилирование и алертинг**. Без метрик вы не увидите проблемы вовремя. Включайте счётчики ошибок/тайм-аутов, p95/p99, насыщение пулов и держите алерты на отклонения — это ваша страховка.

**application.yml — базовые тайм-ауты и HTTP-клиент**

```yaml
feign:
  httpclient:
    enabled: true        # или feign.okhttp.enabled: true
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
        loggerLevel: BASIC
```

**Java — настройка Apache HttpClient для Feign**

```java
package com.example.limits;

import feign.Client;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignApacheConfig {

    @Bean
    public Client feignClient() {
        var req = RequestConfig.custom()
                .setConnectTimeout(1000)
                .setResponseTimeout(2000)
                .build();
        var http = HttpClientBuilder.create()
                .setDefaultRequestConfig(req)
                .evictExpiredConnections()
                .evictIdleConnections(java.time.Duration.ofSeconds(30))
                .setMaxConnTotal(200)
                .setMaxConnPerRoute(50)
                .build();
        return new feign.httpclient.ApacheHttpClient(http);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.limits

import feign.Client
import feign.httpclient.ApacheHttpClient
import org.apache.hc.client5.http.config.RequestConfig
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class FeignApacheConfig {

    @Bean
    fun feignClient(): Client {
        val req = RequestConfig.custom()
            .setConnectTimeout(1000)
            .setResponseTimeout(2000)
            .build()
        val http = HttpClientBuilder.create()
            .setDefaultRequestConfig(req)
            .evictExpiredConnections()
            .evictIdleConnections(Duration.ofSeconds(30))
            .setMaxConnTotal(200)
            .setMaxConnPerRoute(50)
            .build()
        return ApacheHttpClient(http)
    }
}
```

---

## Экосистема вокруг: Spring Cloud LoadBalancer, Resilience4j, Micrometer/OTel

Spring Cloud LoadBalancer позволяет указывать **логическое имя** сервиса в `@FeignClient(name="catalog")`, а не `url`. Дальше резолвинг идёт через реестр (Eureka/Consul) или статическую конфигурацию, а запросы балансируются по политикам (round-robin, random и т. п.). Это устраняет жёсткие URL и упрощает canary/blue-green.

С **Resilience4j** вы получаете CircuitBreaker/Retry/Bulkhead для защиты от деградаций. Feign интегрируется через Spring Cloud CircuitBreaker: на сервисном методе вешаете `@CircuitBreaker(name="catalog")`, а конфиг задаёте в `application.yml`. При «плохой погоде» вызов отстреливается быстро, а не висит в очередях.

**Micrometer** даёт метрики per-client/method/status (таймеры, счётчики ошибок/тайм-аутов), а **OpenTelemetry** — спаны, автоматически связанные с входящим запросом. Это критично для SRE: вы видите p95/p99 по каждому методу и можете строить алерты по SLA.

В связке с **Gateway** удобно делать токен-релэй/переписывание путей, а Feign-клиент видит «нормальный» адрес внутреннего сервиса. Это упрощает безопасность и сегментацию периметра.

При работе с **внешними API** иногда LB не нужен — используйте `url:` и отключайте балансировку. Для внутренних — наоборот: лучше опираться на имя, чтобы не трогать клиентов при масштабировании провайдера.

Для телеметрии придерживайтесь **умеренной кардинальности тегов** (не кладите сырой ID заказа в метки): это сбережёт Prometheus/OTel-бэкенд от взрывного роста рядов. Имейте дашборды на каждый клиент с p95/p99, error rate и насыщением пула.

С точки зрения **трассировки** важно пропагировать заголовки `traceparent`/`b3`, а для платёжных/PII-сервисов маскировать содержимое тел/заголовков. Делайте это в одном `RequestInterceptor`, чтобы не размазывать по коду.

В больших системах полезно держать **единый starter-модуль** с общими перехватчиками (корреляция, токены, маскирование), логгером Feign и автоконфигом телеметрии. Тогда каждая команда просто добавляет зависимость и получает «правильные» клиенты по стандарту.

И, наконец, **инфра-чек-листы**: у каждого клиента должны быть тайм-ауты, политики ретраев/CB (или осознанное их отсутствие), метрики, логгер уровня BASIC и маскирование секретов. Это повышает предсказуемость и снижает MTTR.

**Gradle — доп. зависимости для экосистемы**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.39.0'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-loadbalancer")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.39.0")
}
```

**Java — CB и метрики вокруг Feign-вызова**

```java
package com.example.ecosystem;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Service;

@Service
class CatalogService {
    private final CatalogClient client;
    private final MeterRegistry meter;

    CatalogService(CatalogClient client, MeterRegistry meter) {
        this.client = client; this.meter = meter;
    }

    @CircuitBreaker(name = "catalog")
    public Item getById(Long id) {
        long t0 = System.nanoTime();
        try {
            return client.findById(id);
        } finally {
            meter.timer("feign.catalog", "method", "findById").record(System.nanoTime() - t0,
                    java.util.concurrent.TimeUnit.NANOSECONDS);
        }
    }
}

@org.springframework.cloud.openfeign.FeignClient(name = "catalog")
interface CatalogClient { @org.springframework.web.bind.annotation.GetMapping("/items/{id}") Item findById(@org.springframework.web.bind.annotation.PathVariable Long id); }
record Item(Long id, String name) {}
```

**Kotlin — эквивалент**

```kotlin
package com.example.ecosystem

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import java.util.concurrent.TimeUnit

@FeignClient(name = "catalog")
interface CatalogClient {
    @GetMapping("/items/{id}") fun findById(@PathVariable id: Long): Item
}
data class Item(val id: Long, val name: String)

@Service
class CatalogService(private val client: CatalogClient, private val meter: MeterRegistry) {

    @CircuitBreaker(name = "catalog")
    fun getById(id: Long): Item {
        val t0 = System.nanoTime()
        return try { client.findById(id) }
        finally { meter.timer("feign.catalog", "method", "findById")
            .record(System.nanoTime() - t0, TimeUnit.NANOSECONDS) }
    }
}
```

---

## Альтернативы/дополнения: WebClient для реактивных/стриминговых кейсов

Если у вас **стриминговые ответы** (SSE, большие выгрузки по chunked-transfer), либо сотни одновременных исходящих коннектов, **WebClient** будет уместнее Feign. Он неблокирующий, эффективно использует event-loop и поддерживает backpressure. Цена — усложнение кода и необходимость понимать реактивную модель.

Хороший паттерн — **комбинация**: Feign для «обычных» CRUD-вызовов, WebClient для потоков/долгих коннектов. Так вы держите простые вещи простыми, а сложные — эффективными. Например, BFF может получать карточку пользователя через Feign, а подписку на события — через WebClient SSE.

В потоковых сценариях важны **тайм-ауты и лимиты**. В WebClient их больше и тоньше: `responseTimeout`, `readTimeout` на уровне Netty, `timeout()` в реактивной цепочке, ограничение количества одновременно читаемых чанков. Без этого легко получить «чёрные дыры» памяти.

Не забывайте про **обработку 4xx/5xx** и унификацию ошибок: `onStatus` должен собирать доменные исключения, а не пропускать «сырые» `WebClientResponseException`. Тогда ваш сервисный слой остаётся одинаковым и при Feign, и при WebClient.

При **массовых fan-out** вызовах к множеству источников WebClient может дать выигрыш по p99, но только если все звенья действительно неблокирующие (DNS, TLS, сериализация на стороне клиента и, по возможности, сервер). Иначе получите «полуреактивный» Frankenstein с блокировками.

С точки зрения **телеметрии** WebClient уже интегрирован с Observations/Micrometer/OTel. С Feign вы тоже её получаете, но в WebClient проще добавить доменные теги фильтром. Следите за кардинальностью — не кладите сырые ID в метки.

Если у вас **скачивания** файлов гигабайтами, WebClient позволяет легко «прокачать» поток на диск/S3 без буферизации. Feign тоже может, но там чаще удобнее явная работа с `Response/Body` и кастомными декодерами.

Есть и «между мирами» подход: использовать Feign для запроса **URL выгрузки**, а сам файл скачивать WebClient-ом реактивно. Это часто встречается в отчётных сервисах и экспортах.

Не путайте **реактивность с магией**. Если ваш провайдер медленный, ни WebClient, ни Feign не ускорят ответы. Правильные тайм-ауты, деградация (CB), кэширование и разумный параллелизм — важнее выбора клиента.

И наконец, не стремитесь «переписать всё на реактивное», если это не несёт ценности: увеличится когнитивная нагрузка команды и стоимость поддержки. Будьте прагматиками: **Feign для простого, WebClient для сложного**.

**Gradle — WebFlux/WebClient дополнение**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
}
```

**Java — совместное использование Feign и WebClient (SSE)**

```java
package com.example.combo;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Flux;

@FeignClient(name="users", url="${clients.users.url}")
interface UsersClient { @GetMapping("/users/me") User me(); }
record User(Long id, String name) {}

@Service
public class CompositeService {
    private final UsersClient users;
    private final WebClient webClient = WebClient.create("https://stream.example.com");

    public CompositeService(UsersClient users) { this.users = users; }

    public User profile() { return users.me(); }

    public Flux<String> notifications() {
        return webClient.get().uri("/sse/notifications")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(String.class);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.combo

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.http.MediaType
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.reactive.function.client.WebClient
import reactor.core.publisher.Flux

@FeignClient(name = "users", url = "\${clients.users.url}")
interface UsersClient { @GetMapping("/users/me") fun me(): User }
data class User(val id: Long, val name: String)

@Service
class CompositeService(private val users: UsersClient) {
    private val webClient = WebClient.create("https://stream.example.com")

    fun profile(): User = users.me()

    fun notifications(): Flux<String> =
        webClient.get().uri("/sse/notifications")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(String::class.java)
}
```



# 2. Установка и базовая конфигурация

## Зависимость: `spring-cloud-starter-openfeign`, включение `@EnableFeignClients`

OpenFeign подключается как стартер Spring Cloud и активируется аннотацией `@EnableFeignClients`. Стартер подтянет Feign Core и автоконфигурацию, «склеит» его со Spring Boot (ObjectMapper, валидаторы, логирование), а `@EnableFeignClients` запустит сканирование интерфейсов с `@FeignClient`.

В типовом проекте версии согласуются через BOM Spring Cloud. Для линейки Spring Boot 3.3.x берите совместимую ветку Spring Cloud 2023.x. Это исключает конфликт артефактов Feign/Apache/OkHttp и стабилизирует поведение тайм-аутов и логирования.

`@EnableFeignClients` следует размещать в главном приложении или в конфигурации, которая видит пакеты клиентов. По умолчанию она сканирует пакет приложения; если клиенты лежат в другом модуле/пакете, укажите `basePackages` или `basePackageClasses`.

Интерфейсы с `@FeignClient` — это «контракт». В них описываются методы, пути, параметры и формат тел. Spring генерирует прокси, которая при вызове метода делает HTTP-запрос и возвращает десериализованный ответ.

Если проект мультимодульный, держите клиентов в отдельном модуле `:clients` и подключайте его там, где они нужны. Это уменьшает время сборки и улучшает повторное использование.

Для тестов аннотация не мешает: вы можете подменять `@FeignClient` моками через `@TestConfiguration` или использовать WireMock/MockWebServer. Ничего особенного в настройке — это тот же Spring Bean.

При первом подключении проверьте, что в classpath нет одновременно Apache и OkHttp клиентов с включёнными флагами — выберите один. Иначе поведение может зависеть от порядка автоконфигураций.

В production окружениях клиенты обычно не создаются при старте, если нет зависимостей (LB/SSL). Однако лучше валидировать конфигурацию на стендах и помнить о тайм-аутах.

Важно: Feign — синхронный клиент. Если вы ранее использовали WebClient и случайно мигрируете потоковые сценарии, пересмотрите архитектуру — Feign не для стриминга.

И наконец, в Native/AOT-сборках (GraalVM) используйте актуальные версии Spring Cloud; для нестандартных кодеков/логгеров может понадобиться рефлекшн-хинты.

**Gradle: подключение OpenFeign и BOM**
*Groovy DSL*

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
}
dependencyManagement {
    imports { mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.3" }
}
```

*Kotlin DSL*

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.24"
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}
dependencyManagement {
    imports { mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.3") }
}
```

**Java — включение и простой клиент**

```java
package com.example.app;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.clients")
public class Application { }
```

```java
package com.example.clients;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "time", url = "${clients.time.url}")
public interface TimeClient {
    @GetMapping("/api/v1/now")
    String now();
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.app

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.cloud.openfeign.EnableFeignClients

@SpringBootApplication
@EnableFeignClients(basePackages = ["com.example.clients"])
class Application
```

```kotlin
package com.example.clients

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "time", url = "\${clients.time.url}")
interface TimeClient {
    @GetMapping("/api/v1/now")
    fun now(): String
}
```

---

## Общее и per-client конфигурирование через `feign.client.config.default|{name}.*`

OpenFeign позволяет задавать политику «по умолчанию» и переопределять её для конкретного клиента по имени — это ключ к управляемости. Вы объявляете базовые тайм-ауты, логирование, retry и т.д., а для «чувствительных» клиентов задаёте отдельные значения.

Приоритет простой: настройки `feign.client.config.{clientName}` перекрывают `feign.client.config.default`. Если свойства не заданы у клиента — берутся из default. Это работает и на уровне профилей `spring.profiles`.

Для сложных случаев вы можете привязать конфигурационный класс к клиенту через атрибут `configuration` аннотации `@FeignClient`. Такой класс подменит бины (Encoder/Decoder/Logger/Retryer) только для конкретного клиента.

Метод-уровневые политики (например, тайм-аут на один метод) задаются через `configKey` у Feign (путь «интерфейс#метод»). Spring Cloud автоматически формирует ключи, и их можно адресно переопределить в yaml.

Такой подход решает «дрейф» настроек: все тайм-ауты/логирование/ретраи лежат в одном месте, понятны и документируемы. Это важно для SRE/эксплуатации и аудита.

Полезно держать «жёсткие» дефолты (короче тайм-ауты, BASIC логгинг) и осознанно расширять их там, где это действительно нужно. Иначе дефолты разъедутся и потеряют смысл.

Если клиентов много, заведите «таблицу клиентов» (md/Confluence), где фиксируете SLA, тайм-ауты, retry/CB и бизнес-контакт. Это ускоряет инциденты.

Проверяйте, что имя в `@FeignClient(name="...")` совпадает с секцией в `application.yml`. Опечатки приводят к молчаливому применению default-политики.

При рефакторинге имен клиентов помните, что ключи в yaml завязаны на имя. Меняйте их синхронно и валидируйте автотестами.

Для секретов (ключи, базовая авторизация) используйте внешние Secret-хранилища/переменные окружения, а не хардкод — клиенты легко читают свойства из `SPRING_...`.

**application.yml — default + per-client + per-method**

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
        loggerLevel: BASIC
      time:
        connectTimeout: 500
        readTimeout: 1500
        loggerLevel: HEADERS
      orders#OrdersClient#getById(Long):
        readTimeout: 3000
```

**Java — клиент с привязанной конфигурацией**

```java
package com.example.clients;

import com.example.clients.config.OrdersFeignConfig;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "orders", configuration = OrdersFeignConfig.class)
public interface OrdersClient {
    @GetMapping("/api/v1/orders/{id}")
    OrderDto getById(@PathVariable("id") Long id);
}

record OrderDto(Long id, String status) { }
```

```java
package com.example.clients.config;

import feign.Logger;
import feign.Retryer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OrdersFeignConfig {
    @Bean Logger.Level feignLoggerLevel() { return Logger.Level.BASIC; }
    @Bean Retryer feignRetryer() { return new Retryer.Default(100, 500, 2); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import com.example.clients.config.OrdersFeignConfig
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@FeignClient(name = "orders", configuration = [OrdersFeignConfig::class])
interface OrdersClient {
    @GetMapping("/api/v1/orders/{id}")
    fun getById(@PathVariable id: Long): OrderDto
}

data class OrderDto(val id: Long, val status: String)
```

```kotlin
package com.example.clients.config

import feign.Logger
import feign.Retryer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OrdersFeignConfig {
    @Bean fun feignLoggerLevel(): Logger.Level = Logger.Level.BASIC
    @Bean fun feignRetryer(): Retryer = Retryer.Default(100, 500, 2)
}
```

---

## Выбор HTTP-клиента: Apache HttpClient или OkHttp (`feign.httpclient.enabled`/`feign.okhttp.enabled`)

Spring Cloud OpenFeign может использовать Apache HttpClient (hc5) или OkHttp в качестве транспорта. Оба — зрелые клиенты с пулами соединений, тайм-аутами, keep-alive и TLS. Выбор зависит от предпочтений команды и некоторых особенностей.

Apache HttpClient традиционно популярен в корпоративной Java: богатые настройки пула, прокси, маршрутизация, conn-mgr. Он чуть тяжелее, но предсказуем и широко документирован. Для большинства внутренних API этого достаточно.

OkHttp — лёгкий и быстрый клиент с дружелюбным API и родной поддержкой HTTP/2. Если вы активно используете H2, gRPC-шлюзы или мобильные бэкенды, OkHttp даст удобство и производительность. Однако потребуется добавить зависимость `okhttp` в classpath.

Включение транспорта делается через свойства: `feign.httpclient.enabled=true` для Apache, `feign.okhttp.enabled=true` для OkHttp. Не включайте оба одновременно — используйте ровно один флаг и соответствующую зависимость.

Для Apache полезно настраивать `MaxConnTotal/MaxConnPerRoute`, idle-evict и DNS-кеш. Для OkHttp — `connectionPool(maxIdle, keepAliveDuration)` и `dispatcher.maxRequests{,PerHost}` под ваши пиковые нагрузки.

HTTP/2: OkHttp поддерживает H2 «из коробки» поверх TLS (ALPN). В Apache придётся убедиться, что сборка и TLS-стек позволяют H2 (в корпоративной среде это чаще реже). Для внутренних сервисов H2 не всегда нужен, keep-alive на HTTP/1.1 часто достаточен.

TLS: и Apache, и OkHttp читают trust-store/keystore из стандартных настроек JVM или явно заданных в коде. Проверьте hostname verification и набор шифров под ваши требования безопасности.

Логирование: OkHttp имеет собственные интерсепторы; Apache — нет, но Feign логгирует запрос/ответ на уровне клиента (BASIC/HEADERS/FULL). Маскируйте секреты кастомным логгером, независимо от транспорта.

Стабильность: обе опции стабильны; выбирайте одну и стандартизируйте в платформенном модуле, чтобы командам не приходилось спорить на каждом проекте.

И наконец, измеряйте. Если разницы нет — выбирайте тот, что проще сопровождать в вашей инфраструктуре.

**Gradle: добавление транспорта**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    // Apache HttpClient5 (если нужен явный контроль)
    implementation 'org.apache.httpcomponents.client5:httpclient5:5.3.1'
    // или OkHttp:
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.apache.httpcomponents.client5:httpclient5:5.3.1")
    // or
    // implementation("com.squareup.okhttp3:okhttp:4.12.0")
}
```

**application.yml — выбор транспорта**

```yaml
feign:
  httpclient:
    enabled: true      # Apache HttpClient (hc5)
  okhttp:
    enabled: false     # переключите на true, если используете OkHttp
```

**Java — явная инъекция Apache клиента**

```java
package com.example.http;

import feign.Client;
import feign.httpclient.ApacheHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignApacheClientConfig {
    @Bean
    public Client feignClient() {
        return new ApacheHttpClient(HttpClientBuilder.create().build());
    }
}
```

**Kotlin — явная инъекция OkHttp**

```kotlin
package com.example.http

import feign.Client
import feign.okhttp.OkHttpClient
import okhttp3.ConnectionPool
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.TimeUnit

@Configuration
class FeignOkHttpClientConfig {
    @Bean
    fun feignClient(): Client {
        val pool = ConnectionPool(100, 30, TimeUnit.SECONDS)
        return OkHttpClient(okhttp3.OkHttpClient.Builder().connectionPool(pool).build())
    }
}
```

---

## Базовые тайм-ауты: `connectTimeout`, `readTimeout`, дефолты и переопределения

Тайм-ауты — первый рубеж устойчивости. `connectTimeout` ограничивает установку TCP/TLS-соединения, `readTimeout` — ожидание данных после отправки запроса. Без них поток «висит» и отнимает ресурсы у JVM и ОС.

Задавайте дефолты консервативно (например, 1s/2s) и увеличивайте для конкретных клиентов, чьи SLA требуют большего окна. Подбирайте значения на стендах с реальной сетью и нагрузкой.

Пер-клиентное переопределение в `application.yml` помогает держать настройки в одном месте. Для особо критичных методов можно использовать метод-уровневый ключ `configKey` (см. пример выше).

Помните, что `readTimeout` — это не «весь запрос», а ожидание одной операции чтения. На больших телах он может «продлеваться» по мере поступления данных. Если нужен «общий потолок времени», добавьте тайм-аут на уровне бизнес-операции.

Тайм-ауты клиента должны быть согласованы с балансировщиками/прокси и серверными тайм-аутами. Несогласованность порождает «слепые» ретраи и обрывы.

Сжатие, TLS и JSON-сериализация увеличивают время — закладывайте запас. Но не делайте тайм-ауты «навсегда» — это ухудшит восстановление под отказами.

На Apache HttpClient можно также задать сокетные тайм-ауты и evict idle connections. На OkHttp — `callTimeout`, `readTimeout`, `writeTimeout` и настройки пула.

Логируйте тайм-ауты как отдельную категорию ошибок. Ваши алерты должны поднимать сигнал при всплесках timeout-ов.

И, наконец, проверяйте тайм-ауты тестами (WireMock `fixedDelay`/`chunkedDribbleDelay`) — без этого легко решить, что «всё настроено», но в реальности оно не работает.

**application.yml — дефолт и переопределения**

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
      reports:
        connectTimeout: 2000
        readTimeout: 5000
```

**Java — тонкая настройка Apache HttpClient5**

```java
package com.example.timeout;

import feign.Client;
import feign.httpclient.ApacheHttpClient;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ApacheTimeoutConfig {
    @Bean
    public Client feignClient() {
        var req = RequestConfig.custom()
                .setConnectTimeout(java.time.Duration.ofMillis(1000))
                .setResponseTimeout(java.time.Duration.ofMillis(2000))
                .build();
        var http = HttpClientBuilder.create()
                .setDefaultRequestConfig(req)
                .evictIdleConnections(java.time.Duration.ofSeconds(30))
                .setMaxConnTotal(200).setMaxConnPerRoute(50)
                .build();
        return new ApacheHttpClient(http);
    }
}
```

**Kotlin — OkHttp тайм-ауты**

```kotlin
package com.example.timeout

import feign.Client
import feign.okhttp.OkHttpClient
import okhttp3.ConnectionPool
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.TimeUnit

@Configuration
class OkHttpTimeoutConfig {

    @Bean
    fun feignClient(): Client {
        val ok = okhttp3.OkHttpClient.Builder()
            .connectTimeout(1, TimeUnit.SECONDS)
            .readTimeout(2, TimeUnit.SECONDS)
            .callTimeout(3, TimeUnit.SECONDS)
            .connectionPool(ConnectionPool(100, 30, TimeUnit.SECONDS))
            .build()
        return OkHttpClient(ok)
    }
}
```

---

## Сжатие: `feign.compression.request/response.enabled`, лимиты размера

Сжатие запросов/ответов экономит трафик и снижает latency на «толстых» JSON. Feign в Spring Cloud поддерживает включение gzip на запросах и автоматическую обработку сжатых ответов.

Включайте сжатие только там, где это оправдано: маленькие JSON в несколько сот байт не выиграют, а CPU потратится. Используйте порог `min-request-size`, чтобы не сжимать крохи.

Проверьте, что сервер объявляет поддержку gzip/deflate/br и корректно проставляет `Content-Encoding`. Внешние провайдеры иногда «забывают», из-за чего ответы читаются некорректно.

Следите за компрессией на уровне балансировщиков и прокси — двойное сжатие бесполезно и может ломать метрики.

Не сжимайте уже сжатые форматы (JPEG/PNG/ZIP/PROTOBUF) — это бессмысленно и тормозит. На такие маршруты либо отключайте сжатие, либо фильтруйте типы контента.

На OkHttp/Apache можно включить транспортное сжатие. Но в большинстве кейсов достаточно настроек Feign/Spring Cloud.

В логах FULL-уровня не показывайте тела сжатых запросов/ответов — они бесполезны и загромождают логи. Оставьте HEADERS или BASIC.

В нагрузочных тестах сравните p95/p99 с и без компрессии. На некоторых профилях выигрыш существенный, на других — нет.

И наконец, документируйте политику: где включено, пороги, исключения. Это пригодится при инцидентах.

**application.yml — включение компрессии**

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          min-request-size: 1024
        response:
          enabled: true
```

**Java — клиент с большим телом запроса**

```java
package com.example.compress;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(name = "reports", url = "${clients.reports.url}")
public interface ReportsClient {
    @PostMapping(path = "/api/v1/report")
    ReportResponse generate(@RequestBody ReportRequest req);
}

record ReportRequest(java.util.Map<String, Object> payload) { }
record ReportResponse(String id, String status) { }
```

**Kotlin — эквивалент**

```kotlin
package com.example.compress

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody

@FeignClient(name = "reports", url = "\${clients.reports.url}")
interface ReportsClient {
    @PostMapping("/api/v1/report")
    fun generate(@RequestBody req: ReportRequest): ReportResponse
}

data class ReportRequest(val payload: Map<String, Any?>)
data class ReportResponse(val id: String, val status: String)
```

---

## Переопределение бинов: `Encoder/Decoder/Contract/Logger/Retryer` через конфиг-класс

Feign в Spring Cloud по умолчанию использует Spring-совместимые энкодеры/декодеры (через `HttpMessageConverters`) и контракт Spring MVC. Но вы можете заменить любые из них для конкретного клиента или глобально.

`Encoder/Decoder` — это сериализация/десериализация тел. Часто достаточно Jackson через Spring, но для protobuf/CBOR/MsgPack нужен кастом. Реализацию вешают в конфиг-класс и привязывают к клиенту.

`Contract` задаёт семантику аннотаций. По умолчанию — Spring MVC (`@GetMapping`, `@RequestParam` и т.д.). Менять его нужно редко (например, если вы хотите использовать чистый Feign-контракт).

`Logger` управляет тем, что пишет Feign. Уровни: NONE/BASIC/HEADERS/FULL. В проде обычно BASIC, а для отладки конкретного клиента — HEADERS. FULL — осторожно, легко утечь секретами и нагрузить CPU/IO.

`Retryer` — простая стратегия повторов Feign. Часто вместо неё берут Resilience4j Retry (более гибок и согласуется с CircuitBreaker). Но для некоторых клиентов достаточно и `Retryer.Default`.

Конфиг-класс должен быть виден только нужному клиенту. Не кладите его в `@ComponentScan` без необходимости — иначе он может примениться глобально.

В конфиг-классе можно также задать `RequestInterceptor` (пропагация заголовков, токены) и `Capability` (расширение функциональности Feign).

Для AOT/Graal при кастомных декодерах/энкодерах возможно потребуются рефлекшн-хинты — проверяйте сборку.

И наконец, документируйте: какие клиенты переопределяют что и зачем. Это облегчит ревью и поддержку.

**Java — конфиг с Logger/Retryer/Decoder (SpringDecoder)**

```java
package com.example.clients.config;

import feign.Logger;
import feign.Retryer;
import feign.codec.Decoder;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringDecoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.beans.factory.ObjectFactory;

@Configuration
public class CustomFeignConfig {

    private final ObjectFactory<HttpMessageConverters> converters;

    public CustomFeignConfig(ObjectFactory<HttpMessageConverters> converters) {
        this.converters = converters;
    }

    @Bean
    public Logger.Level feignLoggerLevel() { return Logger.Level.HEADERS; }

    @Bean
    public Retryer retryer() { return new Retryer.Default(100, 500, 2); }

    @Bean
    public Decoder decoder() { return new SpringDecoder(converters); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients.config

import feign.Logger
import feign.Retryer
import feign.codec.Decoder
import org.springframework.beans.factory.ObjectFactory
import org.springframework.boot.autoconfigure.http.HttpMessageConverters
import org.springframework.cloud.openfeign.support.SpringDecoder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class CustomFeignConfig(
    private val converters: ObjectFactory<HttpMessageConverters>
) {
    @Bean fun feignLoggerLevel(): Logger.Level = Logger.Level.HEADERS
    @Bean fun retryer(): Retryer = Retryer.Default(100, 500, 2)
    @Bean fun decoder(): Decoder = SpringDecoder(converters)
}
```

**Java — привязка конфига к клиенту**

```java
package com.example.clients;

import com.example.clients.config.CustomFeignConfig;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "catalog", configuration = CustomFeignConfig.class)
public interface CatalogClient {
    @GetMapping("/api/v1/items")
    ItemsPage list();
}

record ItemsPage(java.util.List<String> items) { }
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import com.example.clients.config.CustomFeignConfig
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "catalog", configuration = [CustomFeignConfig::class])
interface CatalogClient {
    @GetMapping("/api/v1/items")
    fun list(): ItemsPage
}

data class ItemsPage(val items: List<String>)
```

# 3. Декларация клиентов (@FeignClient)

## Атрибуты: `name`, `url`, `path`, `configuration`, `decode404`

Первое, что определяет клиент, — это **`name`**. Оно служит логическим идентификатором бина и точкой интеграции со Spring Cloud LoadBalancer/Service Discovery. Когда вы указываете только `name`, резолвинг адресов выполняет балансировщик: это удобно для внутренних микросервисов и уменьшает количество «жёстких» URL в конфигурации.

Атрибут **`url`** нужен для статических внешних API или когда сервис-дискавери не используется. В этом случае адрес подставляется напрямую, балансировщик отключается. Хорошая практика — хранить `url` в `application.yml` и подтягивать его из `ENV`, чтобы не коммитить реальные хосты.

Атрибут **`path`** добавляет общий префикс ко всем методам клиента. Это удобный способ зафиксировать версию API (`/api/v1`) и базовый путь; далее в методах используют относительные сегменты. Такой подход снижает дублирование и облегчает массовую смену версии.

Атрибут **`configuration`** позволяет привязать к клиенту конфигурационный класс с переопределёнными бинами (`Encoder/Decoder/Logger/Retryer/RequestInterceptor`). Конфиг применяется **только** к этому клиенту и не влияет на остальных — это важно для инкапсуляции особых требований внешнего провайдера.

Атрибут **`decode404`** управляет тем, как обрабатывать статус 404. По умолчанию 404 приводит к `FeignException`. Если включить `decode404 = true`, тело 404 будет десериализовано обычным декодером — это удобно для API, в которых «не найдено» — это нормальный ответ c полезным телом (например, `{ "exists": false }`).

Иногда встречается **`contextId`** — альтернативный идентификатор бина, если в одном приложении есть несколько клиентов с одинаковым `name` (например, один внутренний LB-клиент и один внешний по `url`). Это устраняет конфликты имён при регистрации бинов.

Параметры **`name`** и **`url`** можно комбинировать, но функционально используется либо одно, либо другое: если задан `url`, то балансировка по `name` не участвует. Тем не менее `name` останется ключом для `feign.client.config.{name}` в `application.yml`.

Для внешних API полезно разделять конфиги на «боевые» и «песочницу», переключая `url` и учётные данные профилями. Это позволяет тестировать интеграции без риска затронуть продакшн.

В монорепозиториях имеет смысл стандартизировать соглашение об именовании: клиент → `name=каталог-сервиса`, пакет → `com.company.clients.<service>`, конфиг → `application-<env>.yml` c секцией `feign.client.config.<name>`. Это ускоряет ревью и расследование инцидентов.

Не злоупотребляйте глобальными конфигами «на все клиенты», когда разные провайдеры требуют разные тайм-ауты/ретраи. Проще и безопаснее держать дефолты строгими, а исключения — адресными.

**Java — атрибуты @FeignClient (name/url/path/configuration/decode404)**

```java
package com.example.clients;

import feign.Logger;
import feign.Retryer;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(
    name = "users",
    url = "${clients.users.url}",         // для внешнего API; уберите, если используете LB
    path = "/api/v1",                    // общий префикс для всех методов
    configuration = UsersClient.Config.class,
    decode404 = true
)
public interface UsersClient {
    @GetMapping("/users/{id}")
    UserDto findById(@PathVariable("id") Long id);

    record UserDto(Long id, String name, String username) {}

    @Configuration
    class Config {
        @Bean Logger.Level feignLoggerLevel() { return Logger.Level.BASIC; }
        @Bean Retryer retryer() { return new Retryer.Default(100, 500, 2); }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import feign.Logger
import feign.Retryer
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@FeignClient(
    name = "users",
    url = "\${clients.users.url}",
    path = "/api/v1",
    configuration = [UsersClient.Config::class],
    decode404 = true
)
interface UsersClient {
    @GetMapping("/users/{id}")
    fun findById(@PathVariable("id") id: Long): UserDto
    data class UserDto(val id: Long, val name: String, val username: String)

    @Configuration
    class Config {
        @Bean fun feignLoggerLevel(): Logger.Level = Logger.Level.BASIC
        @Bean fun retryer(): Retryer = Retryer.Default(100, 500, 2)
    }
}
```

---

## Поддержка Spring MVC аннотаций: `@Get/Post/Put/DeleteMapping`, `@RequestParam`, `@PathVariable`, `@RequestHeader`, `@RequestBody`

Интерфейсы Feign-клиентов в Spring Cloud используют **контракт Spring MVC**, поэтому привычные аннотации применяются так же, как в контроллерах. Это снижает когнитивную нагрузку: одна и та же модель вх/вых по обе стороны HTTP.

Методы помечаются `@GetMapping/@PostMapping/...` с относительными путями относительно `path` клиента. Переменные пути описываются `@PathVariable`, причём имя параметра должно совпадать с плейсхолдером или быть явно указано в атрибуте.

Квери-параметры передаются `@RequestParam`. Значения могут быть простыми, коллекциями и картами. Коллекции сериализуются как `?tags=a&tags=b`, если не настроено иначе. `required=false` позволяет передавать опциональные параметры по `null`.

Заголовки добавляются `@RequestHeader`. Это полезно для идемпотентности (например, `Idempotency-Key`) и корреляции (`X-Request-Id`). Можно комбинировать «фиксированные» заголовки на уровне метода и «сквозные» — через `RequestInterceptor`.

Тела запросов оформляются `@RequestBody`. По умолчанию используется Jackson и общий `ObjectMapper` из контекста. Подключите модуль JavaTime, если сериализуете даты/время.

Важно помнить, что **валидация** на стороне клиента (например, `@Valid`) не выполняет «магии» — она проверит DTO до отправки. Это может быть полезно для раннего выявления ошибок до похода в сеть.

При работе с формами (`application/x-www-form-urlencoded`) используйте DTO с полями, а не `Map`, — это повышает читаемость и позволяет прикрутить валидацию. Для multipart — отдельный разговор в следующем пункте.

Содержимое и типы контента (`consumes/produces`) желательно фиксировать в аннотации, чтобы избежать неожиданных «415 Unsupported Media Type» при смене серверной логики.

Если ответ — пустой (204/201 без тела), используйте `void`/`ResponseEntity<Void>` и проверяйте только статус/заголовки. Это разгружает парсер и избавляет от лишних аллокаций.

Для тонкого контроля статуса/заголовков возвращайте `ResponseEntity<T>` — Spring-декодер корректно соберёт все части ответа для бизнес-логики.

**Java — примеры MVC-аннотаций в клиенте**

```java
package com.example.clients;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@FeignClient(name = "orders", url = "${clients.orders.url}", path = "/api/v1")
public interface OrdersClient {

    @GetMapping(value = "/orders/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    OrderDto get(@PathVariable("id") Long id, @RequestHeader("X-Request-Id") String rid);

    @GetMapping("/orders")
    OrdersPage list(@RequestParam(value = "status", required = false) String status,
                    @RequestParam(value = "page", defaultValue = "0") int page);

    @PostMapping(value = "/orders", consumes = MediaType.APPLICATION_JSON_VALUE)
    ResponseEntity<Void> create(@RequestBody CreateOrder cmd);

    @DeleteMapping("/orders/{id}")
    void delete(@PathVariable Long id);

    record OrderDto(Long id, String status) {}
    record OrdersPage(java.util.List<OrderDto> items, int page, int totalPages) {}
    record CreateOrder(String sku, int qty) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@FeignClient(name = "orders", url = "\${clients.orders.url}", path = "/api/v1")
interface OrdersClient {

    @GetMapping("/orders/{id}", produces = [MediaType.APPLICATION_JSON_VALUE])
    fun get(@PathVariable("id") id: Long, @RequestHeader("X-Request-Id") rid: String): OrderDto

    @GetMapping("/orders")
    fun list(
        @RequestParam("status", required = false) status: String?,
        @RequestParam("page", defaultValue = "0") page: Int
    ): OrdersPage

    @PostMapping("/orders", consumes = [MediaType.APPLICATION_JSON_VALUE])
    fun create(@RequestBody cmd: CreateOrder): ResponseEntity<Void>

    @DeleteMapping("/orders/{id}")
    fun delete(@PathVariable id: Long)

    data class OrderDto(val id: Long, val status: String)
    data class OrdersPage(val items: List<OrderDto>, val page: Int, val totalPages: Int)
    data class CreateOrder(val sku: String, val qty: Int)
}
```

---

## Динамические параметры: `@SpringQueryMap/@QueryMap`, `Map<String, ?>` заголовков и квери

Для сложных квери-параметров удобнее использовать **объекты-контейнеры** вместо длинной россыпи `@RequestParam`. В Spring Cloud OpenFeign применяют аннотацию **`@SpringQueryMap`** (рекомендация), которая раскладывает поля бина в квери-строку с учётом конвертеров Spring.

Аннотация `@QueryMap` из «чистого» Feign также работает, но в экосистеме Spring лучше `@SpringQueryMap`, потому что она дружит со `ConversionService` и привычной конверсией типов (например, `LocalDate`).

Сложные типы (коллекции/мапы) сериализуются в несколько одноимённых параметров (`?tag=a&tag=b`), если не указано иначе. Если нужен «CSV»-стиль, придётся добавить кастомный конвертер для нужного поля.

Параметры-`null` обычно пропускаются, а опциональные поля не попадают в запрос — это удобно для «фильтров». Если сервер ожидает пустую строку вместо отсутствующего параметра, настройте конвертер.

Динамические **заголовки** можно передать картой `@RequestHeader Map<String, String>` — это полезно для редких, но адресных заголовков (`X-Provider-Feature`). Сквозные заголовки (Authorization, correlation) лучше внедрять `RequestInterceptor`, чтобы не дублировать по методам.

Если провайдер принимает «пакет» квери-параметров с префиксом (`filter[name]`, `filter[status]`), оформите DTO с таким неймингом полей или переопределите сериализацию (например, кастомный `Param.Expander` для конкретных параметров).

Осторожнее с кодировкой: пробелы и нестандартные символы нужно корректно экранировать. `@SpringQueryMap` делает это автоматически, но ручные строки и «самосбор» URL часто ломаются на пробелах и «+».

Для многоязычности и локализации удобно держать поле `locale`/`lang` в DTO фильтров и передавать его централизованно, не «дописывая» каждый раз `@RequestParam`.

Динамические параметры повышают читаемость, но не заменяют документацию. Описывайте в Javadoc, какие поля поддерживает фильтр, и приводите пример результата сериализации — это сэкономит часы отладки.

Для приватных API (внутренняя шина) стандартизируйте названия квери-параметров и формат значений. Это упрощает кросс-командную интеграцию и позволяет переиспользовать фильтры между клиентами.

**Java — `@SpringQueryMap` и динамические заголовки**

```java
package com.example.clients;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.cloud.openfeign.SpringQueryMap;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestHeader;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

@FeignClient(name = "catalog", url = "${clients.catalog.url}", path = "/api/v1")
public interface CatalogClient {

    @GetMapping("/items")
    ItemsPage list(@SpringQueryMap Filters filters,
                   @RequestHeader Map<String, String> extraHeaders);

    record Filters(String q, List<String> tag, LocalDate from, LocalDate to, Integer page) {}
    record ItemsPage(java.util.List<String> items, int page, int totalPages) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.cloud.openfeign.SpringQueryMap
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestHeader
import java.time.LocalDate

@FeignClient(name = "catalog", url = "\${clients.catalog.url}", path = "/api/v1")
interface CatalogClient {

    @GetMapping("/items")
    fun list(
        @SpringQueryMap filters: Filters,
        @RequestHeader extraHeaders: Map<String, String>
    ): ItemsPage

    data class Filters(
        val q: String?,
        val tag: List<String>?,
        val from: LocalDate?,
        val to: LocalDate?,
        val page: Int?
    )

    data class ItemsPage(val items: List<String>, val page: Int, val totalPages: Int)
}
```

---

## Мультимедиа: `consumes/produces`, файлы (`MultipartFile`, `@RequestPart`)

Работа с файлами требует **multipart/form-data** и специального энкодера. В Spring Cloud OpenFeign поддержка multipart подключается через `SpringFormEncoder`, который оборачивает стандартный `SpringEncoder` и умеет собирать части формы.

На стороне клиента для загрузки лучше использовать **`Resource`** (`ByteArrayResource/FileSystemResource`) вместо `MultipartFile`. Последний тип — это модель **контроллера**, а клиенту удобнее передавать поток/ресурс как часть.

Метод помечают `@PostMapping(..., consumes = MULTIPART_FORM_DATA_VALUE)` и параметры — `@RequestPart("file") Resource file`. Дополнительные части (например, JSON-метаданные) можно передать отдельным `@RequestPart("meta")` с DTO или строкой.

Для скачивания бинарей используйте `byte[]` или `Resource` в возврате. Второй вариант предпочтителен для больших файлов: его можно «протрубить» в S3/диск без буферизации памяти.

Не забывайте про тип контента частей (`Content-Type`). Для файла обычно достаточно `application/octet-stream`. Для JSON-части — `application/json`. `SpringFormEncoder` сформирует корректные границы и заголовки, если типы указаны.

Комбинированные формы (файл + JSON) полезны для загрузки вместе с метаданными. Их удобно описывать двумя `@RequestPart`. На серверной стороне используйте тот же контракт — это упростит e2e-тесты.

Крупные файлы качайте/загружайте потоково и проверяйте тайм-ауты/лимиты размеров: `spring.servlet.multipart.max-file-size` на сервере и разумные `readTimeout`/`callTimeout` на клиенте.

Логирование FULL для multipart запрещено практикой — слишком объёмно и небезопасно. Держите уровень BASIC/HEADERS и маскируйте имена/размеры, если это PII.

Тестируйте multipart через WireMock с `multipart`-матчерами и контролируйте заголовки — мелкие ошибки в `Content-Type` быстро превращаются в 415 на проде.

**Gradle (оба DSL) — поддержка multipart (обычно уже в стартере)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.boot:spring-boot-starter-web' // конвертеры
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

**Java — конфиг SpringFormEncoder и клиент с multipart**

```java
package com.example.clients.upload;

import feign.codec.Encoder;
import feign.form.spring.SpringFormEncoder;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestPart;

@FeignClient(name = "files", url = "${clients.files.url}",
        path = "/api/v1", configuration = FileUploadClient.Config.class)
public interface FileUploadClient {

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    UploadResponse upload(@RequestPart("file") Resource file,
                          @RequestPart("meta") Meta meta);

    record Meta(String owner, String comment) {}
    record UploadResponse(String id, long size) {}

    @Configuration
    class Config {
        private final org.springframework.beans.factory.ObjectFactory<HttpMessageConverters> converters;
        public Config(org.springframework.beans.factory.ObjectFactory<HttpMessageConverters> converters) {
            this.converters = converters;
        }
        @Bean Encoder feignFormEncoder() { return new SpringFormEncoder(new SpringEncoder(converters)); }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients.upload

import feign.codec.Encoder
import feign.form.spring.SpringFormEncoder
import org.springframework.beans.factory.ObjectFactory
import org.springframework.boot.autoconfigure.http.HttpMessageConverters
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.cloud.openfeign.support.SpringEncoder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.Resource
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestPart

@FeignClient(
    name = "files",
    url = "\${clients.files.url}",
    path = "/api/v1",
    configuration = [FileUploadClient.Config::class]
)
interface FileUploadClient {

    @PostMapping("/upload", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
    fun upload(@RequestPart("file") file: Resource,
               @RequestPart("meta") meta: Meta): UploadResponse

    data class Meta(val owner: String, val comment: String)
    data class UploadResponse(val id: String, val size: Long)

    @Configuration
    class Config(private val converters: ObjectFactory<HttpMessageConverters>) {
        @Bean fun feignFormEncoder(): Encoder = SpringFormEncoder(SpringEncoder(converters))
    }
}
```

---

## Возвраты: `ResponseEntity<T>`, `byte[]`, `InputStream/Resource` для стриминга

Если вам нужны **код статуса и заголовки**, возвращайте `ResponseEntity<T>`. Spring-декодер соберёт всё, и вы сможете прочитать, например, заголовок `Retry-After` или `ETag` вместе с телом.

Для небольших бинарных ответов достаточно **`byte[]`**. Это просто и эффективно, когда речь о килобайтах/десятках килобайт. Для мегабайт и гигабайт лучше переходить к потокам, иначе велик риск OOM.

Стриминговый сценарий покрывают **`org.springframework.core.io.Resource`** или «низкоуровневый» **`feign.Response`**. С `Resource` удобнее работать в Spring-мире: его можно сразу писать на диск или прокидывать дальше без полной загрузки в память.

`InputStream` как возврат возможен, но потребует осторожности с закрытием. Без try-with-resources легко утечь дескрипторами. При использовании `Resource` закрытие, как правило, корректно управляется декодером.

Когда ответ может быть «пустым» (204) или «создан ресурс» (201 с `Location`), `ResponseEntity<Void>` — идеальный тип: вы проверяете только статус и нужные заголовки, экономя аллокации.

Если нужен «сырый» доступ ко всем байтам/заголовкам/статусу, используйте **`feign.Response`** и разбирайте тело вручную. Это оправдано для нестандартных форматов или диагностики проблем.

Не забывайте закрывать поток/ресурс в finally — особенно в службах, которые занимаются файловыми выгрузками. Иначе ошибки проявятся не сразу и будут «плыть» от нагрузки.

Для больших скачиваний добавляйте логирование прогресса и ограничение времени. Внешние провайдеры не всегда корректно закрывают соединение — тайм-ауты защитят от «вечных» висевших потоков.

Метрики по объёму/длительности скачиваний помогут держать под контролем производительность и стоимость. Счётчики — дешёвая страховка.

**Java — разные типы возврата (Entity/bytes/Resource)**

```java
package com.example.clients.downloads;

import feign.Response;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.core.io.Resource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "reports", url = "${clients.reports.url}", path = "/api/v1")
public interface ReportsClient {

    @GetMapping("/reports/{id}/meta")
    ResponseEntity<Meta> meta(@PathVariable("id") String id);

    @GetMapping("/reports/{id}/preview")
    byte[] preview(@PathVariable("id") String id);

    @GetMapping("/reports/{id}/file")
    Resource download(@PathVariable("id") String id);

    record Meta(String id, String type, long size) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients.downloads

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.core.io.Resource
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@FeignClient(name = "reports", url = "\${clients.reports.url}", path = "/api/v1")
interface ReportsClient {

    @GetMapping("/reports/{id}/meta")
    fun meta(@PathVariable("id") id: String): ResponseEntity<Meta>

    @GetMapping("/reports/{id}/preview")
    fun preview(@PathVariable("id") id: String): ByteArray

    @GetMapping("/reports/{id}/file")
    fun download(@PathVariable("id") id: String): Resource

    data class Meta(val id: String, val type: String, val size: Long)
}
```

**Java — пример безопасного чтения ресурса в файл**

```java
package com.example.clients.downloads;

import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;

@Service
public class DownloadService {
    private final ReportsClient client;

    public DownloadService(ReportsClient client) { this.client = client; }

    public Path save(String id, Path target) throws Exception {
        Resource res = client.download(id);
        try (InputStream in = res.getInputStream()) {
            Files.copy(in, target, java.nio.file.StandardCopyOption.REPLACE_EXISTING);
        }
        return target;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients.downloads

import org.springframework.stereotype.Service
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.StandardCopyOption

@Service
class DownloadService(private val client: ReportsClient) {

    fun save(id: String, target: Path): Path {
        client.download(id).inputStream.use { input ->
            Files.copy(input, target, StandardCopyOption.REPLACE_EXISTING)
        }
        return target
    }
}
```

---

## Версионирование/базовые пути: константы, `path` на уровне клиента + относительные пути методов

Версионирование — часть контракта. Удобно фиксировать **базовый префикс** через атрибут `path` клиента (`/api/v1`). Это избавляет методы от дублирования версии и облегчает переход на `v2` — меняется одно место.

Для читабельности и консистентности полезно держать **константы путей** рядом с клиентом. Это уменьшает риск опечаток и облегчает ревью; особенно важно для редко вызываемых маршрутов.

Комбинируйте `path` клиента и **относительные** пути методов. Следите за слэшами: `path="/api/v1"` и метод `@GetMapping("/items")` → итог `/api/v1/items`. Двойные слэши — частый источник 404 при неверном склеивании.

Версию API удобно вынести в конфигурацию (`clients.catalog.base-path=/api/v1`) и использовать как значение `path`. Тогда переключение версий произойдёт без перекомпиляции кода.

Старайтесь **не смешивать** версии в одном клиенте. Если нужно ходить в `/v1` и `/v2`, разделите интерфейсы. Это сделает контракты ясными и упростит удаление устаревшей версии.

В путях используйте **говорящие сегменты** и единый стиль именования. Это поможет и серверам, и клиентам. Консистентность — это скорость командной разработки.

Если провайдер поддерживает **content negotiation** по заголовкам (`Accept: application/vnd.example.v2+json`), фиксируйте это в `produces`/`consumes` методов и в тестах. Это предотвратит неожиданные 406.

Для внутренних BFF-клиентов удобно выносить базовые пути и версии в отдельный модуль-константник. Тогда фронтовые и админские клиенты будут синхронизированы по путям.

Переход на новую версию делайте «по учебнику»: параллельные клиенты, теневая нагрузка в тестах, фичи-флаги, метрики ошибок/латентности разделены по `v1/v2`. Это уменьшит риски в релизе.

Пишите Javadoc рядом с константами и методами. Документы со временем стареют, а Javadoc в коде — живой и проходящий ревью артефакт.

**Java — базовые пути и версии как константы/свойства**

```java
package com.example.clients.versioned;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(
    name = "catalog",
    url = "${clients.catalog.url}",
    path = "${clients.catalog.base-path:/api/v1}"
)
public interface CatalogV1Client {
    String ITEMS = "/items";

    @GetMapping(ITEMS)
    ItemsPage list();

    record ItemsPage(java.util.List<String> items, int page, int totalPages) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.clients.versioned

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(
    name = "catalog",
    url = "\${clients.catalog.url}",
    path = "\${clients.catalog.base-path:/api/v1}"
)
interface CatalogV1Client {
    companion object { const val ITEMS = "/items" }

    @GetMapping(ITEMS)
    fun list(): ItemsPage

    data class ItemsPage(val items: List<String>, val page: Int, val totalPages: Int)
}
```

**application.yml — смена версии без перекомпиляции**

```yaml
clients:
  catalog:
    url: https://api.example.com
    base-path: /api/v2
```

# 4. Серилизация/десериализация и схемы данных

## Jackson encoder/decoder по умолчанию, настройка ObjectMapper (JavaTime, `PropertyNamingStrategy`)

По умолчанию Spring Cloud OpenFeign использует Spring-совместимый `Encoder/Decoder`, который опирается на `HttpMessageConverters` и один общий `ObjectMapper`. Это означает, что ваши настройки Jackson из контекста автоматически применяются к сериализации/десериализации тел Feign-запросов и ответов. Такой подход упрощает поддержку: достаточно один раз сконфигурировать `ObjectMapper` «как для всего приложения».

Модуль `jackson-datatype-jsr310` обязателен, если вы передаёте даты/время (`LocalDate`, `Instant`). Без него Jackson пишет/читает их как непонятные структуры или таймстемпы, а это ломает контракты с внешними API. Подключайте модуль и явно задавайте формат (ISO-8601) для предсказуемости.

Стратегия именования полей (`PropertyNamingStrategies.SNAKE_CASE`/`LOWER_CAMEL_CASE`) должна совпадать с тем, что ждёт провайдер. Если сервер требует snake_case, настройте её глобально либо создайте отдельный `ObjectMapper` на клиента, чтобы не менять весь сервис.

Не игнорируйте флаги совместимости: `FAIL_ON_UNKNOWN_PROPERTIES=false` часто уместен на чтении (сервер может прислать лишние поля), а на записи — наоборот, строгие правила помогают не «утекать» мусор. Отдельная опция — `WRITE_DATES_AS_TIMESTAMPS=false`, чтобы писать даты строками.

Разделяйте «глобальный» `ObjectMapper` и «per-client», когда конкретный провайдер требует особого формата (даты без зоны, другое именование полей). Для этого привяжите конфигурационный класс к клиенту через `configuration=` и создайте специфический `Decoder/Encoder`.

В многоязычных приложениях учитывайте локаль и зону времени. На уровне полезной нагрузки лучше договориться о UTC и ISO-8601, а локализацию оставлять для фронта. Это уменьшит погрешности и исключит плавающие ошибки.

Тестируйте сериализацию контрактными тестами: фикстуры JSON «как у провайдера» → DTO и обратно. Это ловит несовпадения имен/форматов до продакшна. В WireMock удобно закрепить «золотые» примеры ответов.

Старайтесь избегать «магии» Jackson через глобальные миксины «на всё». Лучше локальные `@JsonProperty`/`@JsonFormat` и аккуратный `ObjectMapper` для клиента — так меньше сюрпризов при апгрейдах библиотек.

Большие вложенные модели держите стабильными: DTO — это контракт, не храните в них бизнес-логику. Любая «хитрость» усложняет поддержку изменений схемы.

И наконец, документируйте отличия: если один клиент живёт на snake_case, а другой — на camelCase, зафиксируйте это в README модуля с примерами JSON.

**Gradle (оба DSL)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
}
```

**Java — глобальная настройка Jackson + привязка к Feign**

```java
package com.example.feign.jackson;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import feign.codec.Decoder;
import feign.codec.Encoder;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.support.SpringDecoder;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.beans.factory.ObjectFactory;

@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
            .findAndRegisterModules();
    }

    @Bean
    public Decoder springDecoder(ObjectFactory<HttpMessageConverters> converters) {
        return new SpringDecoder(converters);
    }

    @Bean
    public Encoder springEncoder(ObjectFactory<HttpMessageConverters> converters) {
        return new SpringEncoder(converters);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.jackson

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.PropertyNamingStrategies
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import feign.codec.Decoder
import feign.codec.Encoder
import org.springframework.beans.factory.ObjectFactory
import org.springframework.boot.autoconfigure.http.HttpMessageConverters
import org.springframework.cloud.openfeign.support.SpringDecoder
import org.springframework.cloud.openfeign.support.SpringEncoder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class JacksonConfig {

    @Bean
    fun objectMapper(): ObjectMapper =
        ObjectMapper()
            .registerModule(JavaTimeModule())
            .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
            .findAndRegisterModules()

    @Bean fun springDecoder(converters: ObjectFactory<HttpMessageConverters>): Decoder = SpringDecoder(converters)
    @Bean fun springEncoder(converters: ObjectFactory<HttpMessageConverters>): Encoder = SpringEncoder(converters)
}
```

---

## Явные модели ошибок (Problem+JSON) и десериализация тела в `ErrorDecoder`

Стандарт RFC 7807 (Problem Details for HTTP APIs) описывает формат ошибок `application/problem+json`. Он удобен тем, что несёт `type`, `title`, `status`, `detail`, `instance` и расширяемые поля; такой «конверт» легко сериализовать/десериализовать и логировать.

С Feign правильная практика — читать тело ошибки в `ErrorDecoder` и маппить его на доменные исключения. Это позволяет на уровне сервиса принимать решения по коду/типу ошибки, а не по «сырому» статусу. Например, 404 `user-not-found` — это одна реакция, 404 `order-deleted` — другая.

Будьте осторожны с потоком тела: `Response.body()` — одноразовый. Прочитали — закройте. Если используете SpringDecoder в `ErrorDecoder`, аккуратно оборачивайте входной поток, чтобы не ломать downstream-логирование. Проще — читать как строку и парсить `ObjectMapper`-ом.

Различайте «сетевые»/временные ошибки (нет ответа, тайм-аут, 503) и бизнес-ошибки (валидаторы, не найдено). Первая категория обычно идёт в Retry/CB, вторая — обрабатывается логикой домена и не ретраится.

Если сервер возвращает `application/json` вместо `problem+json`, договоритесь о поле `code`/`message` и опишите собственную модель «конверта» ошибки. Важно иметь стабильный контракт и не полагаться на «любые» поля.

Возвращайте наверх понятные исключения: `UserNotFoundException`, `PaymentDeclinedException`. Это улучшает читаемость и позволяет строить консистентную обработку на контроллерах/фасадах.

Логируйте `instance`/correlation-id из ответа. Это сильно ускорит расследование инцидентов: у вас появится «сквозной» идентификатор в логах обеих сторон.

Не показывайте внутренние детали провайдера клиентам фронта. «Переведите» их в свои коды/сообщения, а исходные — логируйте и сохраняйте для диагностики.

Покройте `ErrorDecoder` юнит-тестами с примерами разных тел и статусов. Это дешёво и спасает от регрессов на апгрейдах.

Если используете Spring 6+, можно унифицировать формат на уровне контроллеров через `ProblemDetail`. На клиенте просто парсите его как обычный DTO.

**Gradle (оба DSL)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("com.fasterxml.jackson.core:jackson-databind")
}
```

**Java — `ErrorDecoder` с парсингом Problem+JSON**

```java
package com.example.feign.error;

import com.fasterxml.jackson.databind.ObjectMapper;
import feign.Response;
import feign.codec.ErrorDecoder;
import java.io.IOException;
import java.io.InputStream;

public class ProblemErrorDecoder implements ErrorDecoder {
    private final ObjectMapper om;

    public ProblemErrorDecoder(ObjectMapper om) { this.om = om; }

    @Override
    public Exception decode(String methodKey, Response response) {
        try (InputStream is = response.body() != null ? response.body().asInputStream() : null) {
            Problem p = is != null ? om.readValue(is, Problem.class) : new Problem(response.status(), "no-body");
            return mapToDomain(p, response.status());
        } catch (IOException e) {
            return new RemoteCallException("Failed to decode error: " + methodKey, e);
        }
    }

    private RuntimeException mapToDomain(Problem p, int status) {
        if (status == 404 && "user-not-found".equals(p.type())) return new UserNotFoundException(p.detail());
        if (status == 409 && "payment-declined".equals(p.type())) return new PaymentDeclinedException(p.detail());
        return new RemoteCallException(p.detail());
    }

    public record Problem(int status, String detail, String type) { public Problem(int s, String d){ this(s,d,null);} }
    public static class UserNotFoundException extends RuntimeException { public UserNotFoundException(String m){ super(m);} }
    public static class PaymentDeclinedException extends RuntimeException { public PaymentDeclinedException(String m){ super(m);} }
    public static class RemoteCallException extends RuntimeException { public RemoteCallException(String m){ super(m);} public RemoteCallException(String m, Throwable t){ super(m,t);} }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.error

import com.fasterxml.jackson.databind.ObjectMapper
import feign.Response
import feign.codec.ErrorDecoder

class ProblemErrorDecoder(private val om: ObjectMapper) : ErrorDecoder {
    override fun decode(methodKey: String, response: Response): Exception {
        return try {
            val body = response.body()?.asInputStream()
            val p = body?.use { om.readValue(it, Problem::class.java) } ?: Problem(response.status(), "no-body", null)
            mapToDomain(p, response.status())
        } catch (e: Exception) {
            RemoteCallException("Failed to decode error: $methodKey", e)
        }
    }

    private fun mapToDomain(p: Problem, status: Int): RuntimeException =
        when {
            status == 404 && p.type == "user-not-found" -> UserNotFoundException(p.detail)
            status == 409 && p.type == "payment-declined" -> PaymentDeclinedException(p.detail)
            else -> RemoteCallException(p.detail)
        }

    data class Problem(val status: Int, val detail: String, val type: String?)
    class UserNotFoundException(message: String) : RuntimeException(message)
    class PaymentDeclinedException(message: String) : RuntimeException(message)
    class RemoteCallException(message: String, cause: Throwable? = null) : RuntimeException(message, cause)
}
```

---

## Кастомные `Encoder/Decoder` (например, protobuf/CBOR) и выбор per-client

Не всегда JSON — лучший формат. Для телеметрии, плотных сообщений или ограниченных каналов альтернативные форматы (CBOR/MessagePack/Protobuf) снижают размер и ускоряют парсинг. В Feign это решается подменой `Encoder/Decoder` у конкретного клиента.

Если вам нужен **CBOR**, проще всего завести отдельный `ObjectMapper` с `CBORFactory` и Content-Type `application/cbor`. Это даст экономию без полной миграции стека. Важно, чтобы провайдер тоже понимал CBOR.

Для **Protobuf** логика похожа: используйте DTO из `.proto`, `application/x-protobuf` и кастомные `Encoder/Decoder` вокруг `MessageLite`. Это потребует отдельного пайплайна сборки (protoc) и договорённости о backward-compat.

Связывайте конфигурацию только с нужным клиентом через `configuration=` — так вы не «сломаете» остальных JSON-клиентов. Это принцип «локального отклонения от стандарта».

Не смешивайте форматы в одном интерфейсе без крайней необходимости. Лучше два клиента: `CatalogJsonClient` и `CatalogCborClient`. Это уменьшит путаницу и облегчит тестирование.

Протоколы со схемой (Avro/Protobuf) дисциплинируют эволюцию контрактов. Но за это платят сложностью билдов и tooling. Оцените выгоду прежде чем идти туда.

Всегда отправляйте корректный `Content-Type` и принимайте только ожидаемые `Accept`. Контент-негациация — ваш друг для явного контроля форматов.

Покрывайте конвертеры тестами на совместимость: «тот же объект → JSON/CBOR → обратно». И держите golden-файлы в репозитории.

С точки зрения метрик на формат — заведите отдельные таймеры/счётчики по `content_type`. Это позволит различать поведение форматов на проде.

Учтите, что библиотеки форматов инициализируют буферы/кэши; меряйте аллокации и cold-start на стендах под нагрузкой.

В монорепозиториях введите «платформенный» модуль с готовыми конвертерами, чтобы не дублировать код по проектам.

**Gradle — CBOR (оба DSL)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'com.fasterxml.jackson.dataformat:jackson-dataformat-cbor'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-cbor")
}
```

**Java — CBOR Encoder/Decoder и клиент**

```java
package com.example.feign.cbor;

import com.fasterxml.jackson.dataformat.cbor.databind.CBORMapper;
import feign.codec.Decoder;
import feign.codec.Encoder;
import feign.Feign;
import feign.RequestLine;
import feign.Headers;
import feign.Param;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CborFeignConfig {
    @Bean public CBORMapper cborMapper() { return new CBORMapper(); }
    @Bean public Encoder cborEncoder(CBORMapper m) { return (obj, bodyType, template) -> {
        template.header("Content-Type", "application/cbor");
        template.body(m.writeValueAsBytes(obj));
    }; }
    @Bean public Decoder cborDecoder(CBORMapper m) { return (resp, type) -> m.readValue(resp.body().asInputStream(), m.constructType(type)); }
}
```

```java
package com.example.feign.cbor;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

@FeignClient(name = "metrics", url = "${clients.metrics.url}", configuration = CborFeignConfig.class)
public interface MetricsClient {
    @PostMapping(value = "/v1/ingest", consumes = "application/cbor", produces = "application/cbor")
    MetricAck send(@RequestBody MetricPoint p);
    record MetricPoint(String name, long ts, double value) {}
    record MetricAck(String status) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.cbor

import com.fasterxml.jackson.dataformat.cbor.databind.CBORMapper
import feign.codec.Decoder
import feign.codec.Encoder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody

@Configuration
class CborFeignConfig {
    @Bean fun cborMapper(): CBORMapper = CBORMapper()
    @Bean fun cborEncoder(m: CBORMapper): Encoder = Encoder { obj, _, template ->
        template.header("Content-Type", "application/cbor")
        template.body(m.writeValueAsBytes(obj))
    }
    @Bean fun cborDecoder(m: CBORMapper): Decoder = Decoder { resp, type ->
        m.readValue(resp.body().asInputStream(), m.constructType(type))
    }
}

@FeignClient(name = "metrics", url = "\${clients.metrics.url}", configuration = [CborFeignConfig::class])
interface MetricsClient {
    @PostMapping("/v1/ingest", consumes = ["application/cbor"], produces = ["application/cbor"])
    fun send(@RequestBody p: MetricPoint): MetricAck
}
data class MetricPoint(val name: String, val ts: Long, val value: Double)
data class MetricAck(val status: String)
```

---

## Multipart/формы: `SpringFormEncoder`, корректные `Content-Type` и границы

Отправка файлов и смешанных данных (файл + JSON) в Feign делается через `SpringFormEncoder`. Он строит корректный `multipart/form-data` с нужными границами и заголовками частей. Без него сервер часто отвечает `415` или «файл пуст».

На стороне метода используйте `@RequestPart` и тип `Resource`/`byte[]` для файла. `MultipartFile` — это тип для контроллеров, а не для клиентов. Для JSON-части — обычный DTO с аннотацией `@RequestPart("meta")`.

Для классических форм `application/x-www-form-urlencoded` достаточно DTO с полями и `consumes` соответствующего типа. Конвертеры Spring соберут тело автоматически.

Следите за размером и тайм-аутами: загрузка больших файлов требует увеличенных `readTimeout`/`callTimeout` и, возможно, другой транспорт (OkHttp c HTTP/2). Проверьте и серверные лимиты на размер части и запроса.

Не логируйте тела multipart на FULL-уровне. Маскируйте названия и размер, если это PII/секреты. На HEADERS/BASIC будет достаточно информации для трассировки.

Если провайдер требует конкретные имена частей и их порядок, зафиксируйте это в коде клиента и тестах WireMock. Мелкое несоответствие часто ломает интеграцию незаметно.

Устанавливайте явный `Content-Type` у частей, если сервер этого требует: файл — `application/octet-stream`, JSON — `application/json`. `SpringFormEncoder` подставит корректные boundary и длины.

Старайтесь не «смешивать» multipart с нестандартными кодировками. Чем проще форма, тем меньше шансов на расхождения между библиотеками.

Если загрузка — частая операция, вынесите клиента и конфиг в отдельный модуль, чтобы переиспользовать практики и избежать дублирования.

Покройте юнит-тестами сериализацию multipart: WireMock имеет матчеры на части, это позволит закрепить контракт.

**Gradle (оба DSL)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

**Java — конфигурация `SpringFormEncoder` и клиент**

```java
package com.example.feign.multipart;

import feign.codec.Encoder;
import feign.form.spring.SpringFormEncoder;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.cloud.openfeign.support.SpringEncoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.core.io.Resource;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestPart;

@FeignClient(name = "uploader", url = "${clients.uploader.url}", configuration = FileUploadConfig.class)
public interface FileUploadClient {
    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    UploadAck upload(@RequestPart("file") Resource file, @RequestPart("meta") Meta meta);

    record Meta(String owner, String comment) {}
    record UploadAck(String id, long size) {}
}

@Configuration
class FileUploadConfig {
    private final ObjectFactory<HttpMessageConverters> converters;
    FileUploadConfig(ObjectFactory<HttpMessageConverters> converters) { this.converters = converters; }
    @Bean Encoder feignFormEncoder() { return new SpringFormEncoder(new SpringEncoder(converters)); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.multipart

import feign.codec.Encoder
import feign.form.spring.SpringFormEncoder
import org.springframework.beans.factory.ObjectFactory
import org.springframework.boot.autoconfigure.http.HttpMessageConverters
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.cloud.openfeign.support.SpringEncoder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.Resource
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestPart

@FeignClient(name = "uploader", url = "\${clients.uploader.url}", configuration = [FileUploadConfig::class])
interface FileUploadClient {
    @PostMapping("/upload", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
    fun upload(@RequestPart("file") file: Resource, @RequestPart("meta") meta: Meta): UploadAck

    data class Meta(val owner: String, val comment: String)
    data class UploadAck(val id: String, val size: Long)
}

@Configuration
class FileUploadConfig(private val converters: ObjectFactory<HttpMessageConverters>) {
    @Bean fun feignFormEncoder(): Encoder = SpringFormEncoder(SpringEncoder(converters))
}
```

---

## Большие ответы и стриминг: `InputStream`/`Response` без загрузки в память

При скачивании отчётов/экспортов размером в сотни МБ нельзя читать весь ответ в память. Вместо `byte[]` используйте `Resource` или «сырое» `feign.Response` и копируйте поток в хранилище (S3/диск) по частям. Это защищает от OOM и стабилизирует p99.

`Response` даёт полный контроль: статус, заголовки и поток тела. Но вы отвечаете за закрытие потока и корректную обработку кодировок. Для большинства сценариев удобнее `Resource`, который Spring создаёт поверх тела.

Согласуйте тайм-ауты: большие ответы требуют увеличенного `readTimeout`/`callTimeout`. Иначе вы получите ложные тайм-ауты на «медленных» сетях, хотя всё работает корректно.

Старайтесь передавать файл дальше «трубой»: `InputStream` → `Files.copy()`/S3 `putObject`. Избегайте промежуточных `ByteArrayOutputStream`, они быстро съедают память.

Следите за заголовками `Content-Length`/`Transfer-Encoding`. При chunked-передаче не полагайтесь на длину тела — читайте до конца потока.

Используйте `ETag`/`If-None-Match`, если скачивания повторяются. Это уменьшит трафик и ускорит обработку на стороне провайдера.

Проверяйте контрольные суммы (MD5/SHA-256) после скачивания, если это важно для бизнеса. Заголовки часто несут хеши — вы можете валидировать целостность.

Логируйте длительность и объём скачивания. Эти метрики помогут заранее увидеть деградацию провайдеров и проблемы сети.

В тестах применяйте WireMock с `chunkedDribbleDelay`, чтобы эмулировать реальную сеть. Это помогает подобрать тайм-ауты и убедиться, что поток действительно стримится.

При обработке ошибок не читайте всё тело в лог. Для больших ответов пишите только заголовки и первые байты, чтобы не «зафлудить» систему.

**Java — скачивание `Resource` и запись на диск**

```java
package com.example.feign.stream;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.core.io.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.stereotype.Service;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;

@FeignClient(name = "reports", url = "${clients.reports.url}", path = "/api/v1")
interface ReportsClient {
    @GetMapping("/files/{id}")
    Resource download(@PathVariable("id") String id);
}

@Service
class DownloadService {
    private final ReportsClient client;
    DownloadService(ReportsClient client) { this.client = client; }

    public Path save(String id, Path target) throws Exception {
        try (var in = client.download(id).getInputStream()) {
            Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
        }
        return target;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.stream

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.core.io.Resource
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.StandardCopyOption

@FeignClient(name = "reports", url = "\${clients.reports.url}", path = "/api/v1")
interface ReportsClient {
    @GetMapping("/files/{id}")
    fun download(@PathVariable("id") id: String): Resource
}

@Service
class DownloadService(private val client: ReportsClient) {
    fun save(id: String, target: Path): Path {
        client.download(id).inputStream.use { input ->
            Files.copy(input, target, StandardCopyOption.REPLACE_EXISTING)
        }
        return target
    }
}
```

---

## Контент-негациация, локализация сообщений и примеры для документации

Контент-негациация — это согласование формата через `Accept/Content-Type`. Клиент должен явно объявить, что он отправляет и что хочет получить, чтобы избежать «промахов» при изменении серверной логики. В Feign это задаётся в аннотациях методов (`produces/consumes`) и/или заголовках.

Если вы поддерживаете несколько форматов (JSON/CBOR), укажите `produces`/`consumes` и, при необходимости, заводите разные методы/клиенты под форматы. Это сделает контракт явным и облегчит отладку.

Локализация — это не про формат тела, а про язык сообщений и ошибок. Передавайте `Accept-Language` (или собственный `X-Locale`) через `RequestInterceptor`, и сервер вернёт тексты на нужном языке. Важно задокументировать поддерживаемые языки и fallback.

Для документации удобно хранить «эталонные» примеры запросов и ответов (curl/HTTPie) рядом с кодом клиента. Это ускоряет ревью и онбординг, а также помогает SRE при инцидентах.

При генерации Swagger/OpenAPI DTO клиентов должны совпадать с серверными схемами. Если используете один и тот же модуль DTO — обновляйте версии синхронно и прогоняйте контрактные тесты.

Избегайте избыточной кардинальности заголовков в метриках. Логируйте `content-type`, но не включайте «случайные» заголовки в теги Prometheus — это распухнет.

Если сервер поддерживает версионирование через медиа-типы (`application/vnd.example.v2+json`), фиксируйте этот тип в `produces`. Это защищает от «тихого» переключения формата.

В тестах WireMock/MockWebServer убеждайтесь, что клиент реально посылает `Accept` и `Content-Type` такие, какие вы ожидаете. Ошибка здесь — частая причина 406/415.

Для кэшируемых GET важно, чтобы `Accept` не мешал кэшированию. Согласуйте политику с API-провайдером и фронтами.

Проверяйте, как сервер ведёт себя при неизвестном `Accept-Language`: корректный fallback на дефолтный язык — это UX и устойчивость.

**Java — интерсептор языка и явные `produces/consumes`**

```java
package com.example.feign.i18n;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@FeignClient(name = "catalog", url = "${clients.catalog.url}", configuration = I18nConfig.class)
public interface CatalogClient {
    @GetMapping(value = "/api/v1/items", produces = MediaType.APPLICATION_JSON_VALUE)
    ItemsPage list(@RequestParam("q") String q);

    record ItemsPage(java.util.List<String> items) {}
}

@Configuration
class I18nConfig {
    @Bean
    RequestInterceptor acceptLanguage() {
        return new RequestInterceptor() {
            @Override public void apply(RequestTemplate template) {
                template.header("Accept-Language", "ru-RU");
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.i18n

import feign.RequestInterceptor
import feign.RequestTemplate
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestParam

@FeignClient(name = "catalog", url = "\${clients.catalog.url}", configuration = [I18nConfig::class])
interface CatalogClient {
    @GetMapping("/api/v1/items", produces = [MediaType.APPLICATION_JSON_VALUE])
    fun list(@RequestParam("q") q: String): ItemsPage
    data class ItemsPage(val items: List<String>)
}

@Configuration
class I18nConfig {
    @Bean
    fun acceptLanguage(): RequestInterceptor = RequestInterceptor { tmpl: feign.RequestTemplate ->
        tmpl.header("Accept-Language", "ru-RU")
    }
}
```

**Gradle (оба DSL)**
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.cloud:spring-cloud-starter-openfeign' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.cloud:spring-cloud-starter-openfeign") }
```

# 5. Тайм-ауты, пулы и ретраи

## Настройка соединений: размеры пулов, keep-alive, HTTP/2 (через OkHttp где уместно)

В блокирующей модели OpenFeign «один исходящий запрос занимает один поток», поэтому устойчивость и латентность напрямую зависят от правильной конфигурации HTTP-клиента. На практике это сводится к настройке пула соединений, политик keep-alive и, при необходимости, включению HTTP/2 для мультиплексирования на одном TCP-соединении. Игнорирование этих настроек приводит к росту p99, «залипанию» потоков под нагрузкой и лавинообразным отказам при деградации провайдера.

Пул соединений экономит накладные расходы на TCP/TLS-рукопожатие, но его размер нельзя выбирать «чем больше — тем лучше». Слишком маленький пул создаёт очередь ожидания (рост p95/p99 при всплесках), слишком большой — держит много idling-коннектов, расходуя память/FD и мешая честной эвакуации «умерших» соединений после сетевых скачков. Для типичных CRUD-интеграций безопасный старт — `MaxConnTotal` 100–300 и `MaxConnPerRoute` 20–50 с последующей калибровкой по метрикам.

Политика keep-alive важна так же, как и размер пула: вы хотите переиспользовать соединения, но не держать их бесконечно. На Apache HttpClient настраивайте eviction idle-коннектов раз в 20–60 секунд и ограничивайте TTL соединения, чтобы переустанавливать TCP после длительных простоев. На OkHttp настраивайте `ConnectionPool(maxIdle, keepAliveDuration)` и следите, чтобы keep-alive не противоречил настройкам апстрима/балансировщиков.

HTTP/2 уместен там, где провайдер его поддерживает и вы получаете выгоду от мультиплексирования (много мелких запросов к одному хосту). OkHttp даёт H2 «из коробки» (ALPN поверх TLS), что снижает латентность на установке коннекта и позволяет параллельно гонять несколько запросов по одной трубе. Внутренние корпоративные шлюзы не всегда дружат с H2 — проверяйте реальную трассу и fallback на HTTP/1.1.

DNS и маршрутизация тоже влияют на пулы. При агрессивном кешировании DNS вы рискуете «залипнуть» на плохом инстансе провайдера. Если используете дискавери/балансировку на уровне Spring Cloud, контролируйте TTL кеша и периодически пересобирайте адресное множество. На OkHttp можно подключить кастомный `Dns` со своей стратегией TTL.

Idle-eviction критичен для долгоживущих сервисов. После аварий и failover’ов старые TCP-коннекты принимают RST/FIN с задержкой, и без регулярного «выметания» idling-коннектов вы будете ловить загадочные `IOException`/`NoHttpResponseException`. Включайте `evictIdleConnections` (Apache) или задавайте небольшой `keepAliveDuration` (OkHttp), чтобы уменьшить хвосты.

Важная детализация: пулы считаются на процесс, но лимиты часто разумнее считать «на хост» (per-route). Если у вас один горячий провайдер и несколько второстепенных, не давайте ему выжрать весь пул — оставьте запас на остальные, иначе «второстепенные» начнут падать из-за голода по соединениям.

Следите за метриками соединений. Для Apache это насыщение пула (busy vs idle), очередь запросов и ошибки на установке TCP/TLS. Для OkHttp — размер пула/idle/active и частота re-connect. Без наблюдаемости вы будете «на ощупь» ставить значения и промахиваться на проде.

Прокси и MTLS меняют картину. Под корпоративным прокси вы теряете контроль над некоторыми тайм-аутами и keep-alive — тестируйте значения в стендовой среде, максимально похожей на прод. Для MTLS убедитесь, что переиспользование соединений не ломается переустановкой сессии и что пул не «убегает» в утечки при ошибках проверки цепочки.

Наконец, выбирайте транспорт и его конфиг централизованно. В платформенном модуле сделайте автоконфиг Apache/OkHttp с «жёсткими» дефолтами и не позволяйте проектам разъезжаться в ручных настройках. Это улучшит предсказуемость и упростит инцидент-менеджмент.

Для HTTP/2 на OkHttp не забывайте о серверной стороне: некоторым балансировщикам нужен ALPN и корректный список шифров. Всегда проверяйте `curl --http2 -v` на прод-хосте, чтобы подтвердить реальную поддержку H2 именно по вашей трассе.

**Gradle зависимости (если добавляете явный транспорт)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.apache.httpcomponents.client5:httpclient5:5.3.1' // Apache
    // или OkHttp:
    // implementation 'com.squareup.okhttp3:okhttp:4.12.0'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.apache.httpcomponents.client5:httpclient5:5.3.1")
    // or
    // implementation("com.squareup.okhttp3:okhttp:4.12.0")
}
```

**Java — Apache HttpClient: пул и eviction**

```java
package com.example.http;

import feign.Client;
import feign.httpclient.ApacheHttpClient;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class FeignApacheConfig {
    @Bean
    public Client feignClient() {
        var req = RequestConfig.custom()
                .setConnectTimeout(Duration.ofSeconds(1))
                .setResponseTimeout(Duration.ofSeconds(2))
                .build();
        var http = HttpClientBuilder.create()
                .setDefaultRequestConfig(req)
                .evictExpiredConnections()
                .evictIdleConnections(Duration.ofSeconds(30))
                .setMaxConnTotal(200)
                .setMaxConnPerRoute(50)
                .build();
        return new ApacheHttpClient(http);
    }
}
```

**Kotlin — OkHttp: пул и HTTP/2**

```kotlin
package com.example.http

import feign.Client
import feign.okhttp.OkHttpClient
import okhttp3.ConnectionPool
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.TimeUnit

@Configuration
class FeignOkHttpConfig {
    @Bean
    fun feignClient(): Client {
        val pool = ConnectionPool(100, 30, TimeUnit.SECONDS)
        val ok = okhttp3.OkHttpClient.Builder()
            .connectionPool(pool)
            .followRedirects(true)
            .retryOnConnectionFailure(true)
            // HTTP/2 включится автоматически при поддержке ALPN/TLS
            .build()
        return OkHttpClient(ok)
    }
}
```

---

## `connectTimeout` vs `readTimeout` vs socket timeouts на уровне HTTP-клиента

`connectTimeout` ограничивает время на установку TCP/TLS-соединения. Если сеть нестабильна или адрес недоступен, этот тайм-аут предотвращает «залипание» потока на стадии hand-shake. Его разумно держать коротким (0.5–2 секунды), чтобы быстро переключаться на другие инстансы через LB или отдавать ошибку.

`readTimeout` регулирует ожидание данных после отправки запроса. Он не равен «всему бюджету операции» — при длинных телах чтение может продлеваться, если новые байты приходят до истечения окна. Увеличение `readTimeout` оправдано для «тяжёлых» ответов (экспорт, отчёты), но для типичных CRUD-ответов достаточно 1–2 секунд сверх p95 провайдера.

Socket/response timeouts в Apache HttpClient 5 настраиваются как `ResponseTimeout`. Дополнительно есть «время ожидания соединения из пула» — если пул исчерпан, запрос будет стоять в очереди до этого тайм-аута. Его нужно держать небольшим (например, 200–500 мс), чтобы не накапливать лавину ожиданий при внезапных всплесках.

В OkHttp помимо `connectTimeout` и `readTimeout` есть `writeTimeout` — полезен при загрузке больших тел (multipart), чтобы не зависнуть на медленном uplink’е, и `callTimeout` — абсолютный потолок всей операции (соединение+запрос+ответ). `callTimeout` хорош для «жёсткого» бизнес-бюджета, но убедитесь, что он согласован с ретраями/CB, иначе получите двойные тайм-ауты и путаницу.

Тайм-ауты клиента должны быть **короче** тайм-аутов апстрима/шлюза (Nginx/Envoy/API Gateway). Иначе вы дождётесь ошибки от шлюза позже, чем среагируете сами, и в момент деградации упрётесь в массовые зависшие потоки. Согласование тайм-аутов — часть эксплуатационного чек-листа.

Задавайте дефолты консервативно в `feign.client.config.default`, а специфичные увеличения — только точечно. Распространённая ошибка — поставить «всем» `readTimeout: 30s`, что замедляет деградацию при проблеме у провайдера и мешает CB корректно «стрелять».

Тестируйте тайм-ауты на стенде с искусственными задержками и дри́бблингом (chunked задержка). WireMock/MockWebServer позволяют задать `fixedDelay`/`chunkedDribbleDelay` и проверить, что клиент реально «умирает» по `connectTimeout`/`readTimeout`, а не висит дольше.

Не путайте тайм-ауты HTTP-клиента с «тайм-аутами бизнес-логики». Первые — сетевые, вторые — про SLA агрегирующей операции. Обычно достаточно сетевых тайм-аутов + CB/Retry, и только для сложных fan-out сценариев добавляют общий «бюджет» операции на уровне сервисного метода.

В логах/метриках делайте явную категорию «timeout» и бейте по типам: connect vs read vs pool-acquire. Это ускоряет RCA: вы сразу видите, где именно теряете время.

И наконец, не забывайте про TLS-handshake и DNS. Иногда «подключение» тратит время на OCSP/CRL проверку или медленный DNS. Если видите аномально большой `connectTimeout`, проверьте цепочку TLS и кэширование DNS в рантайме.

**application.yml — разные окна по клиентам**

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
      reports:
        connectTimeout: 2000
        readTimeout: 5000
      heavyUpload:
        connectTimeout: 2000
        readTimeout: 10000
```

**Java — Apache: pool-acquire и timeouts**

```java
package com.example.timeout;

import feign.Client;
import feign.httpclient.ApacheHttpClient;
import org.apache.hc.client5.http.config.ConnectionConfig;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class ApacheTimeoutConfig {
    @Bean
    public Client feignClient() {
        var request = RequestConfig.custom()
                .setConnectionRequestTimeout(Duration.ofMillis(300))
                .setConnectTimeout(Duration.ofMillis(1000))
                .setResponseTimeout(Duration.ofMillis(2000))
                .build();
        var conn = ConnectionConfig.custom()
                .setTimeToLive(Duration.ofMinutes(5))
                .build();
        var http = HttpClientBuilder.create()
                .setDefaultRequestConfig(request)
                .setDefaultConnectionConfig(conn)
                .evictIdleConnections(Duration.ofSeconds(30))
                .setMaxConnTotal(200).setMaxConnPerRoute(50)
                .build();
        return new ApacheHttpClient(http);
    }
}
```

**Kotlin — OkHttp: read/write/call timeouts**

```kotlin
package com.example.timeout

import feign.Client
import feign.okhttp.OkHttpClient
import okhttp3.ConnectionPool
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.TimeUnit

@Configuration
class OkHttpTimeoutConfig {
    @Bean
    fun feignClient(): Client {
        val ok = okhttp3.OkHttpClient.Builder()
            .connectTimeout(1, TimeUnit.SECONDS)
            .readTimeout(2, TimeUnit.SECONDS)
            .writeTimeout(5, TimeUnit.SECONDS)
            .callTimeout(3, TimeUnit.SECONDS)
            .connectionPool(ConnectionPool(100, 30, TimeUnit.SECONDS))
            .build()
        return OkHttpClient(ok)
    }
}
```

---

## Повторы: Feign `Retryer` (экспоненциальная пауза) vs Resilience4j Retry (более гибкий контроль)

`Retryer` из Feign — лёгкая перехватная стратегия: она повторяет вызов при сетевых исключениях/тайм-аутах и, при необходимости, на некоторых кодах ответа, используя фиксированную или экспоненциальную задержку. Плюсы — простота и нулевая зависимость от внешних библиотек; минусы — ограниченная выразительность (сложно задавать предикаты по телу ответа/исключениям), отсутствие тесной связи с Circuit Breaker и метриками.

Resilience4j Retry — компонент уровня бизнес-логики: он оборачивает ваш сервисный метод, может принимать решения «повторять/нет» по классу исключения или по содержимому результата (например, повторять 502/503/504, но не 400/422), поддерживает джиттер, backoff и хорошо интегрируется с Circuit Breaker и Bulkhead. Плюс — подробные метрики попыток и причин повторов.

Правило практики: **сетевые, кратковременные ошибки** (transient) можно повторять; **бизнес-ошибки 4xx** — нельзя. `Retryer` в Feign проще включить для «всех GET», но для POST лучше использовать Retry на уровне Resilience4j с предикатами и idempotency-ключами. Это даёт тонкий контроль и избегает дублирования бизнес-операций.

Backoff с джиттером обязателен под деградацией провайдера: без случайной составляющей вы создадите «стадный инстинкт», когда все инстансы одновременно бьют в стену с одинаковой периодичностью. В Resilience4j джиттер настраивается свойствами, в Feign — только через кастомную реализацию `Retryer`.

Не забывайте ограничивать **общее окно** ретраев. Если `readTimeout=2s` и 3 попытки, то суммарно вы уже ждёте до 6 секунд плюс backoff — это может съесть SLA фронта. Иногда разумнее «стрелять» Circuit Breaker после одной неудачи и сразу деградировать.

В системах с «дорогими» вызовами (платёжки) ретраи должны быть осознанны и редки: максимум одна повторная попытка со строгим журналированием причин. Непродуманная политика легко приводит к дублированным списаниям.

Учитывайте семантику методов. `GET/HEAD` обычно безопасны для повторов; `PUT` — условно идемпотентны; `POST` повторяют только при наличии ключа идемпотентности и явной поддержки на сервере. Это не «формальность», а защита от дубликатов и гонок.

Метрики успехов после ретрая ценны для SRE: если большая доля запросов «проходит со второй попытки», провайдер находится на грани деградации, и надо действовать проактивно. Resilience4j из коробки отдаёт такие метрики в Micrometer.

Разносите зоны ответственности: `Retryer` — для очень простых клиентов без тяжёлой логики, Resilience4j — для большинства продакшн-кейсов, где есть SLА, CB, Bulkhead и наблюдаемость. Микс допустим, но избегайте двойных ретраев на одном вызове.

И наконец, тестируйте ретраи на стендах с fault-injection (e.g., `timeout`, `disconnect after headers`). Это единственный способ проверить, что поведение соответствует ожиданиям, а не «кажется правильным».

**Gradle (добавляем Resilience4j)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j")
}
```

**Java — Feign Retryer и Resilience4j @Retry**

```java
package com.example.retry;

import feign.Logger;
import feign.Retryer;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.web.bind.annotation.GetMapping;
import java.util.concurrent.ThreadLocalRandom;

@FeignClient(name = "catalog", configuration = CatalogRetryConfig.class)
interface CatalogClient {
    @GetMapping("/api/v1/items")
    String list();
}

@Configuration
class CatalogRetryConfig {
    @Bean Logger.Level feignLoggerLevel() { return Logger.Level.BASIC; }
    @Bean Retryer feignRetryer() { return new Retryer.Default(100, 500, 2); } // 2 повтора
}

@Service
class CatalogService {
    private final CatalogClient client;
    CatalogService(CatalogClient c) { this.client = c; }

    @Retry(name = "catalog-retry") // Resilience4j Retry c предикатами в конфиге
    public String safeList() { return client.list(); }

    // пример нестабильности для тестов
    public static void randomDelay() { try { Thread.sleep(ThreadLocalRandom.current().nextInt(50, 200)); } catch (InterruptedException ignored) {} }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.retry

import feign.Logger
import feign.Retryer
import io.github.resilience4j.retry.annotation.Retry
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "catalog", configuration = [CatalogRetryConfig::class])
interface CatalogClient {
    @GetMapping("/api/v1/items")
    fun list(): String
}

@Configuration
class CatalogRetryConfig {
    @Bean fun feignLoggerLevel(): Logger.Level = Logger.Level.BASIC
    @Bean fun feignRetryer(): Retryer = Retryer.Default(100, 500, 2)
}

@Service
class CatalogService(private val client: CatalogClient) {
    @Retry(name = "catalog-retry")
    fun safeList(): String = client.list()
}
```

**application.yml — Retry с джиттером**

```yaml
resilience4j:
  retry:
    instances:
      catalog-retry:
        maxAttempts: 3
        waitDuration: 200ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5
        retryExceptions:
          - org.springframework.web.client.ResourceAccessException
          - feign.RetryableException
        ignoreExceptions:
          - com.example.feign.error.ProblemErrorDecoder.RemoteCallException
```

---

## Идемпотентность: безопасные методы/тела, ключи идемпотентности для POST

Идемпотентность — это свойство операции давать один и тот же эффект при повторном выполнении. В HTTP семантически безопасны `GET/HEAD/OPTIONS`; `PUT` считается идемпотентным по смыслу ресурса; `POST` по умолчанию неидемпотентен. Ретрая `POST` без мер предосторожности — прямой путь к дубликатам и дорогим бизнес-ошибкам.

Чтобы сделать `POST` безопаснее, вводят **ключ идемпотентности** (например, заголовок `Idempotency-Key`). Клиент генерирует стабильный уникальный ключ для конкретной бизнес-операции (например, `orderId + attempt 1`), сервер хранит результат первого успешного выполнения и возвращает его при повторе того же ключа. Так ретраи не создают дубликаты.

Ключ должен жить достаточно долго, чтобы покрыть окно ретраев и ожидаемую задержку сети/провайдера. В платёжных системах — минуты/часы; в CRUD-операциях — секунды/минуты. Слишком короткий TTL убьёт пользу, слишком длинный — раздует сторедж и риск коллизий.

Генерация ключа — ответственность клиента. Он должен быть детерминированно повторяемым для одной бизнес-операции, но не переиспользоваться для разных. Практика — UUID v4, привязанный к бизнес-идентификатору, либо хэш детерминированного payload. Главное — стабильность на ретраях.

Ключ нужен **только** для операций с побочными эффектами. Не надо вешать его на `GET/HEAD`: вы сгенерируете шум без пользы. Хорошее правило — добавлять ключ перехватчиком только на `POST/PUT/PATCH`, когда он действительно нужен.

Ключ — это не «серебряная пуля»: сервер обязан уметь корректно дедуплицировать и возвращать сохранённый результат. Клиент со своей стороны должен ожидать 409/422 и уметь осмысленно их обрабатывать. Контракт надо зафиксировать в документации.

Идемпотентность тесно связана с ретраями: разрешайте повтор только если у вас есть ключ (или операция гарантированно идемпотентна по природе). Иначе при «полууспехе» (тайм-аут после фактического выполнения) вы создадите дубликат.

Отслеживайте идемпотентные ключи в логах и метриках. Это помогает расследовать спорные ситуации «прошёл ли первый вызов». Без корреляции по ключу вы будете слепы.

Ключи — чувствительная информация: они позволяют соотносить попытки и бизнес-операцию. Маскируйте их в логах, если в вашей отрасли это требуется политиками безопасности.

Дубликаты — не единственная угроза. Неправильная семантика ключа может «склеить» разные операции, если он генерируется слишком грубо. Убедитесь, что ключ действительно уникален в рамках ***одной*** операции и не пересекается со следующими.

**Java — RequestInterceptor для Idempotency-Key только на POST**

```java
package com.example.idem;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.UUID;

@Configuration
public class IdempotencyInterceptorConfig {
    @Bean
    public RequestInterceptor idempotency() {
        return new RequestInterceptor() {
            @Override public void apply(RequestTemplate template) {
                if ("POST".equalsIgnoreCase(template.method())) {
                    template.header("Idempotency-Key", UUID.randomUUID().toString());
                }
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.idem

import feign.RequestInterceptor
import feign.RequestTemplate
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.UUID

@Configuration
class IdempotencyInterceptorConfig {
    @Bean
    fun idempotency(): RequestInterceptor = RequestInterceptor { t: RequestTemplate ->
        if ("POST".equals(t.method(), ignoreCase = true)) {
            t.header("Idempotency-Key", UUID.randomUUID().toString())
        }
    }
}
```

**Пример клиента и команды (Java)**

```java
package com.example.idem;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(name = "payments", url = "${clients.payments.url}", configuration = IdempotencyInterceptorConfig.class)
public interface PaymentsClient {
    @PostMapping("/api/v1/pay")
    PaymentResult pay(@RequestBody PaymentCommand cmd);
    record PaymentCommand(String orderId, long amount) {}
    record PaymentResult(String status, String paymentId) {}
}
```

**Kotlin — клиент**

```kotlin
package com.example.idem

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody

@FeignClient(name = "payments", url = "\${clients.payments.url}", configuration = [IdempotencyInterceptorConfig::class])
interface PaymentsClient {
    @PostMapping("/api/v1/pay")
    fun pay(@RequestBody cmd: PaymentCommand): PaymentResult
}

data class PaymentCommand(val orderId: String, val amount: Long)
data class PaymentResult(val status: String, val paymentId: String)
```

---

## Пер-клиентные политики (per-name в `application.yml`) и «жёсткие» ограничения

OpenFeign позволяет задавать дефолтную политику и переопределять её для каждого клиента по имени. Этим надо пользоваться системно: «жёсткие» дефолты (короткие тайм-ауты, BASIC-логгинг) + адресные послабления для нуждающихся клиентов. Такой подход удерживает систему предсказуемой и защищает от «разъезда» настроек.

Пер-клиентные настройки — это не только тайм-ауты. Это уровни логирования, поведение `Retryer`, настройки декодеров, сжатия, а также выбор транспорта (Apache/OkHttp). В документации команды должен быть листинг всех клиентов с SLA и параметрами — иначе в инциденте никто не вспомнит, почему у `billing` `readTimeout=10s`.

Удобно поддерживать метод-уровневые исключения через `configKey` (интерфейс#метод(аргументы)). Например, «все методы 1s/2s, но `downloadReport` — 10s». Это фиксируется одной строкой в yaml и помогает избежать хардкода времени в бизнес-слое.

«Жёсткие» ограничения — это ещё и лимиты на размер пула и уровень конкуррентности на сторону провайдера. Если известно, что апстрим выдерживает не более N RPS, имеет смысл ограничить параллелизм на уровне вызова (bulkhead) и/или на уровне пулов, чтобы не создавать шипы нагрузки в пиках.

Для внешних API (почта, геокодинг) вводите более короткие тайм-ауты и жёсткие ретраи (или без них), чтобы не блокировать рабочие потоки при деградации провайдера. Внутренние сервисы обычно терпимее и ближе по сети — параметры могут быть мягче.

Параметры в yaml должны совпадать с `@FeignClient(name="...")`. Опечатки приводят к тихому применению `default` и трудноуловимым проблемам. Автотест на соответствие имён быстро окупается.

Храните конфигурацию в профилях: `application-dev.yml` с мягкими окнами для отладки и `application-prod.yml` с боевыми ограничениями и выключенным FULL-логированием. Профили должны переключаться переменной окружения на CI/CD.

Настраивайте компрессию выборочно. Нет смысла сжимать маленькие JSON, но есть — большие отчёты. Полезно держать `min-request-size` и включать компрессию ответов глобально, если провайдеры её корректно отдают.

Не смешивайте в одном клиенте разные версии API и разные политики. Если у вас `/v1` и `/v2` с отличающимися SLO, вынесите в отдельные клиенты с разными именами и конфигами. Это снизит когнитивную нагрузку.

И наконец, автоматизируйте «санити-чек» конфигов: линтер на отсутствие FULL-логирования в `prod`, отсутствие `retry` на неидемпотентных POST и т. п. Такие проверки ловят опасные места до релиза.

**application.yml — дефолты, пер-клиент и пер-метод**

```yaml
feign:
  httpclient:
    enabled: true
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
        loggerLevel: BASIC
      catalog:
        connectTimeout: 800
        readTimeout: 1500
        loggerLevel: HEADERS
      reports#ReportsClient#download(String):
        readTimeout: 10000
spring:
  cloud:
    openfeign:
      compression:
        response:
          enabled: true
```

**Java — клиент с «длинным» методом**

```java
package com.example.perclient;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.core.io.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "reports", url = "${clients.reports.url}")
public interface ReportsClient {
    @GetMapping("/api/v1/reports/{id}/file")
    Resource download(@PathVariable("id") String id); // пер-метод: readTimeout=10s
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.perclient

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.core.io.Resource
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable

@FeignClient(name = "reports", url = "\${clients.reports.url}")
interface ReportsClient {
    @GetMapping("/api/v1/reports/{id}/file")
    fun download(@PathVariable("id") id: String): Resource
}
```

---

## Тайм-ауты на уровне бизнес-логики (Future/CompletableFuture) — когда НЕ нужно

Желание добавить «общий тайм-аут» на метод сервиса поверх сетевых тайм-аутов кажется логичным, но часто приводит к **двойным тайм-аутам** и сложной диагностике. Если Feign уже ограничивает connect/read и вы используете Retry/CB, дополнительный `orTimeout()` на `CompletableFuture` в большинстве случаев не нужен.

Где действительно нужен общий «бюджет»? В агрегаторах (BFF), которые делают параллельные запросы к нескольким зависимостям и обязаны уложиться в SLA фронта (например, 300 мс). Здесь уместно давать общий дедлайн и отменять медленные ветки, возвращая частичный ответ/заглушку. Но это — **исключение**, а не правило.

Двойной тайм-аут усложняет RCA: вы получите смесь `TimeoutException` из бизнес-слоя и `ReadTimeout` из HTTP-клиента. Логи будут противоречивыми, а метрики — грязными. Лучше полагаться на сетевые тайм-ауты и CB: они дают понятные причины отказа и стандартные пути восстановления.

Если всё-таки нужен общий дедлайн, обеспечьте **одиночность** механизма: либо `callTimeout` в OkHttp, либо `orTimeout()` на агрегирующем `CompletableFuture`, но не оба сразу. И документируйте, что это бюджет конкретного эндпоинта, а не «всех вызовов».

При отмене «медленных веток» важно корректно освобождать ресурсы. В блокирующей модели отмена будущего не отменяет сетевой вызов внутри — он продолжит выполняться и занимать поток/соединение. Это ещё один аргумент против массового использования Business-timeouts в Feign-мире.

Для типичных CRUD-операций достаточно: `connectTimeout`/`readTimeout` + Resilience4j Retry (по transient-ошибкам) + Circuit Breaker (быстрый отказ при деградации) + Bulkhead (лимит параллелизма). Этого набора хватает, чтобы сервис предсказуемо деградировал и восстанавливался.

Если вы всё же применяете общий дедлайн, логируйте его отдельно как «business_timeout», чтобы не смешивать с сетевыми. Это поможет быстро отличать «медленный провайдер» от «слишком тяжёлого агрегатора».

Не используйте бизнес-тайм-ауты как замену оптимизации. Если p95 упирается в сериализацию JSON или медленный фильтр на стороне провайдера, «ножницы времени» лишь замаскируют проблему под 499/504, но не решат её.

И наконец, помните о пользовательском опыте. Жёсткий общий бюджет хорош, когда у вас есть fallback (кэш, заглушки) и UX терпит частичную деградацию. Если fallback’а нет — лучше дать честную ошибку быстро, чем ждать до общего дедлайна.

В реактивных стэках (WebClient) общий бюджет реализуется естественнее (`timeout()` + отмена подписки). В блокирующем Feign это менее эффективно, поэтому применяйте только адресно и осознанно.

**Java — пример агрегатора с общим бюджетом (адресно, не везде)**

```java
package com.example.budget;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.*;

@Service
public class ProfileFacade {
    private final UsersClient users;
    private final OrdersClient orders;
    private final ExecutorService pool = Executors.newFixedThreadPool(8);

    public ProfileFacade(UsersClient u, OrdersClient o) { this.users = u; this.orders = o; }

    public Profile getProfile() throws Exception {
        CompletableFuture<User> fu = CompletableFuture.supplyAsync(users::me, pool);
        CompletableFuture<Orders> fo = CompletableFuture.supplyAsync(orders::recent, pool);
        CompletableFuture<Profile> all = fu.thenCombine(fo, Profile::new);
        try { return all.orTimeout(300, TimeUnit.MILLISECONDS).join(); }
        catch (Exception e) { return fallback(); } // частичная деградация/заглушка
    }

    private Profile fallback() { return new Profile(new User(-1L, "unknown"), new Orders(java.util.List.of())); }
}
record User(Long id, String name) {}
record Orders(java.util.List<String> items) {}
record Profile(User user, Orders orders) {}
```

**Kotlin — эквивалент (адресный дедлайн)**

```kotlin
package com.example.budget

import org.springframework.stereotype.Service
import java.util.concurrent.CompletableFuture
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit

@Service
class ProfileFacade(
    private val users: UsersClient,
    private val orders: OrdersClient
) {
    private val pool = Executors.newFixedThreadPool(8)

    fun getProfile(): Profile {
        val fu = CompletableFuture.supplyAsync({ users.me() }, pool)
        val fo = CompletableFuture.supplyAsync({ orders.recent() }, pool)
        val all = fu.thenCombine(fo) { u, o -> Profile(u, o) }
        return try {
            all.orTimeout(300, TimeUnit.MILLISECONDS).join()
        } catch (_: Exception) {
            Profile(User(-1, "unknown"), Orders(emptyList()))
        }
    }
}

data class User(val id: Long, val name: String)
data class Orders(val items: List<String>)
data class Profile(val user: User, val orders: Orders)
```

# 6. Ошибки и устойчивость

## `ErrorDecoder`: маппинг HTTP-статусов и тел ошибок в доменные исключения

Правильная обработка ошибок в клиентах — основа предсказуемости. По умолчанию Feign бросает общее `FeignException` с кодом статуса, но бизнес-слою редко достаточно голого статуса. `ErrorDecoder` позволяет прочитать тело ответа, распарсить поля ошибки и выбросить осмысленное доменное исключение. Это резко повышает ясность логов и упрощает ветвление поведения в коде.

Полезная практика — договориться о формате ошибок. Если провайдер возвращает `application/problem+json` (RFC 7807), его удобно десериализовать в простой DTO и на основе полей `type`, `status`, `detail` принять решение. Если формат нестандартный, всё равно стоит завести «конверт» с кодом и сообщением, чтобы не разбирать «сырые» строки.

Важно корректно расходовать поток тела. Объект `Response.body()` читается один раз; после чтения поток нужно закрыть. Если тело отсутствует, следует бросать «сетевое» исключение с кратким описанием, не придумывая сообщение. Пустые тела — частая история на 5xx через прокси.

Рекомендую различать «бизнес-ошибки» 4xx и «технические» 5xx/сети. Для 4xx мы выбрасываем чёткие доменные исключения, не подлежащие ретраям. Для 5xx и тайм-аутов — отдельные исключения, которые могут быть обработаны Retry/Circuit Breaker. Это граница ответственности между ErrorDecoder и политикой устойчивости.

При парсинге тела не забудьте про `Content-Type`. Некоторые провайдеры присылают JSON с `text/plain`. В таких случаях безопасно пытаться парсить JSON, а в случае неудачи — сохранить первые N байт как «сырой фрагмент» для логов. Это улучшит диагностику, не рискуя OOM.

Строки сообщений исключений не должны содержать PII/секретов. Если провайдер присылает токены или номера карт в теле ошибки, маскируйте их. В логах всегда выводите корреляционный идентификатор запроса, а не пользовательские данные.

ErrorDecoder — не место для ретраев. Он должен быть детерминированным маппером статуса/тела в исключения. Любая «автоматика» внутри декодера усложнит поведение и запутает метрики.

Тестируйте декодер на фикстурах. Держите пару золотых примеров тел ошибок и прогоняйте через юнит-тесты, чтобы при обновлении зависимостей не сломать обратную совместимость. Это дешёво и спасает прод.

В больших системах часто появляется «общая библиотека ошибок». Вынесите туда базовые типы исключений и декодер, чтобы команды не плодили разные трактовки одних и тех же кодов. Это критично для единого мониторинга.

Наконец, интегрируйте декодер только на уровне нужных клиентов. Глобальный декодер допустим, если формат ошибок везде унифицирован; иначе используйте `configuration` в `@FeignClient` и подключайте точечно.

**Java — ErrorDecoder с поддержкой Problem+JSON и фоллбэком на текст**

```java
package com.example.feign.error;

import com.fasterxml.jackson.databind.ObjectMapper;
import feign.Response;
import feign.codec.ErrorDecoder;

import java.io.InputStream;
import java.nio.charset.StandardCharsets;

public class ProblemErrorDecoder implements ErrorDecoder {
    private final ObjectMapper om;

    public ProblemErrorDecoder(ObjectMapper om) {
        this.om = om;
    }

    @Override
    public Exception decode(String methodKey, Response response) {
        int status = response.status();
        try (InputStream in = response.body() != null ? response.body().asInputStream() : null) {
            Problem problem = null;
            if (in != null) {
                try {
                    problem = om.readValue(in, Problem.class);
                } catch (Exception parse) {
                    // fallback к тексту первых байт
                    String raw = new String(response.body().asInputStream().readAllBytes(), StandardCharsets.UTF_8);
                    problem = new Problem(status, raw.length() > 512 ? raw.substring(0, 512) : raw, null, null);
                }
            }
            if (status == 404) return new ResourceNotFoundException(detail(problem, "Not found"));
            if (status == 409) return new ConflictException(detail(problem, "Conflict"));
            if (status == 422) return new ValidationException(detail(problem, "Unprocessable entity"));
            return new UpstreamServerException("Upstream error " + status + ": " + detail(problem, "no-detail"));
        } catch (Exception e) {
            return new UpstreamServerException("Failed to decode error " + status, e);
        }
    }

    private static String detail(Problem p, String def) {
        return p != null && p.detail() != null ? p.detail() : def;
    }

    public record Problem(Integer status, String detail, String type, String instance) {}
    public static class ResourceNotFoundException extends RuntimeException { public ResourceNotFoundException(String m){super(m);} }
    public static class ConflictException extends RuntimeException { public ConflictException(String m){super(m);} }
    public static class ValidationException extends RuntimeException { public ValidationException(String m){super(m);} }
    public static class UpstreamServerException extends RuntimeException { public UpstreamServerException(String m){super(m);} public UpstreamServerException(String m, Throwable c){super(m,c);} }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.error

import com.fasterxml.jackson.databind.ObjectMapper
import feign.Response
import feign.codec.ErrorDecoder

class ProblemErrorDecoder(private val om: ObjectMapper) : ErrorDecoder {
    override fun decode(methodKey: String, response: Response): Exception {
        val status = response.status()
        return try {
            val problem = response.body()?.asInputStream()?.use {
                runCatching { om.readValue(it, Problem::class.java) }.getOrNull()
            } ?: Problem(status, "no-detail", null, null)

            when (status) {
                404 -> ResourceNotFoundException(problem.detail)
                409 -> ConflictException(problem.detail)
                422 -> ValidationException(problem.detail)
                else -> UpstreamServerException("Upstream error $status: ${problem.detail}")
            }
        } catch (e: Exception) {
            UpstreamServerException("Failed to decode error $status", e)
        }
    }

    data class Problem(val status: Int?, val detail: String?, val type: String?, val instance: String?)
    class ResourceNotFoundException(message: String?) : RuntimeException(message)
    class ConflictException(message: String?) : RuntimeException(message)
    class ValidationException(message: String?) : RuntimeException(message)
    class UpstreamServerException(message: String?, cause: Throwable? = null) : RuntimeException(message, cause)
}
```

---

## Circuit Breaker: `spring-cloud-starter-circuitbreaker-resilience4j` + Feign (альтернатива устаревшему Hystrix)

Circuit Breaker защищает сервис от «бесконечных» неудачных попыток к деградировавшему апстриму. Он считает ошибки и задержки, и при превышении порогов переводится в «open», немедленно отказывая новые вызовы до «охлаждения». Это разгружает систему и даёт апстриму время восстановиться. В экосистеме Spring актуальная реализация — Spring Cloud Circuit Breaker поверх Resilience4j; Hystrix устарел.

Состояния брейкера интуитивны. «Closed» — всё идёт напрямую. «Open» — все вызовы немедленно завершаются ошибкой `CallNotPermittedException`. «Half-Open» — пробные вызовы с ограниченной конкуррентностью; при успехах брейкер закрывается, при неуспехах снова открывается. Эта модель предохраняет от «дозаваливания» апстрима.

Привязка брейкера к Feign проста. Вы либо аннотируете фасадный метод `@CircuitBreaker(name="clientName", fallbackMethod="...")`, либо используете программную обёртку `CircuitBreakerFactory`/`decorateSupplier`. Важно, что брейкер живёт **над** клиентом и видит итог исключений, в том числе выброшенных `ErrorDecoder`.

Пороговые настройки определяют чувствительность. Скользящее окно по количеству/времени, порог доли ошибок, минимальное число вызовов перед срабатыванием — всё это нужно калибровать по метрикам. Слишком чувствительный брейкер будет «трещать» от случайных всплесков, слишком грубый — не успеет защитить.

Интеграция с Retry позволяет пытаться «промахнуться» в окно деградации. Однако ретраи при открытом брейкере бессмысленны. Типовая цепочка — сначала Retry (по transient исключениям), затем CB. Это даёт шанс на «быструю победу», но защищает от лавины.

Метрики Resilience4j автоматически публикуются в Micrometer. Это источник правды для SRE: процент открытого времени, длина half-open, доля успешных после ретрая. Используйте алерты по долгому «open» или по взрывному росту отказов.

Брейкер — не замена хорошим тайм-аутам. Если `readTimeout` слишком велик, брейкер «увидит» проблему слишком поздно, и деградация пройдёт сквозь вас. Согласуйте тайм-ауты клиента и пороги брейкера, иначе смысл теряется.

Для отдельных эндпоинтов пороги стоит отделять именами инстансов брейкера. Например, `reports-download` потребует большее окно и иной порог ошибок, чем `catalog-list`. Это достигается разными `name` и секциями в конфигурации.

Важно не маскировать ошибки фоллбэком без логирования. Открытый брейкер должен прозрачно сигнализировать наблюдаемости. Иначе вы потеряете сигнал и получите «тихую деградацию».

Проверяйте работу брейкера fault-injection тестами. Эмулируйте 5xx и тайм-ауты на стенде и убедитесь, что метрики и поведение соответствуют ожиданию. Только так можно калибровать пороги.

**Gradle зависимости**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — аннотация @CircuitBreaker над фасадом, вызывающим Feign**

```java
package com.example.circuit;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;

@Service
public class CatalogFacade {
    private final CatalogClient client;
    public CatalogFacade(CatalogClient client) { this.client = client; }

    @CircuitBreaker(name = "catalog-cb", fallbackMethod = "fallback")
    public String list() {
        return client.list();
    }

    private String fallback(Throwable t) {
        return "[]"; // безопасная деградация
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.circuit

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import org.springframework.stereotype.Service

@Service
class CatalogFacade(private val client: CatalogClient) {

    @CircuitBreaker(name = "catalog-cb", fallbackMethod = "fallback")
    fun list(): String = client.list()

    @Suppress("unused")
    fun fallback(t: Throwable): String = "[]"
}
```

**application.yml — пороги брейкера**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      catalog-cb:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 50
        minimumNumberOfCalls: 20
        failureRateThreshold: 50
        slowCallDurationThreshold: 1000ms
        slowCallRateThreshold: 50
        waitDurationInOpenState: 5s
        permittedNumberOfCallsInHalfOpenState: 5
```

---

## Fallback/FallbackFactory: деградация сервисов с доступом к причине ошибки

Fallback — это «план Б», когда всё плохо. В контексте Feign есть два механизма: `fallback` и `fallbackFactory`. Первый просто подменяет реализацию интерфейса при ошибках; второй, помимо подмены, даёт доступ к причине ошибки, что полезно для логирования и выбора стратегии деградации. На практике почти всегда нужен именно `fallbackFactory`.

Назначение фоллбэка — защитить UX или критичный путь, вернув безопасную заглушку. Для «необязательных» данных это пустые коллекции, кэш, последние актуальные значения. Важно, чтобы фоллбэк был быстрым и без побочных эффектов; иначе он сам станет источником проблем.

Фоллбэк — это не «поглотитель» ошибок. Он должен логировать причины с ключевыми полями запроса и корреляционным идентификатором, а также увеличивать счётчики деградации в метриках. Иначе вы получите «тихий отказ»: пользователю вроде вернули заглушку, а сигнал SRE потеряется.

Опасно использовать «универсальный» фоллбэк на все методы. Разные операции требуют разной деградации: список товаров можно вернуть пустым, а оформление платежа — нельзя. Делайте фоллбэк на уровне клиента, а внутри — на уровне метода, с явной семантикой.

Фоллбэк и Circuit Breaker дополняют друг друга. Брейкер «быстро отказывает», фоллбэк «быстро деградирует». Но если фоллбэк активируется часто, это сигнал для алертов и разбирательства; не относитесь к нему как к «нормальному состоянию».

Нельзя делать внутри фоллбэка повторные сетевые вызовы без ограничений. Это формирует «скрытые зависимости», которые тяжело отлаживать и которые бьют по устойчивости. Если нужно сходить в кэш или вторичный источник — изолируйте это отдельным брейкером.

Фоллбэк не должен менять контракт. Если метод возвращает DTO определённой формы, заглушка должна придерживаться этой формы, даже если поля пустые или нулевые. Это защищает верхние уровни от NPE и неожиданных ветвлений.

Хорошая практика — инжектировать Clock/TimeSource и помечать результат фоллбэка «stale=true» или меткой времени. Это позволит фронту/агрегатору отличить живые данные от деградации и принять решение об отображении.

Все фоллбэки должны быть покрыты тестами. На уровне WireMock можно эмулировать 5xx/тайм-ауты и убеждаться, что фабрика фоллбэка получает оригинальную причину и возвращает корректную заглушку. Это защищает от регрессов.

Наконец, документируйте деградацию. В README модуля опишите, какие методы как деградируют и что это значит для UX/SLA. В инциденте это экономит драгоценные минуты.

**Java — FallbackFactory с доступом к Throwable**

```java
package com.example.feign.fallback;

import feign.hystrix.FallbackFactory; // если без Hystrix, используйте org.springframework.cloud.openfeign.FallbackFactory
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "catalog", url = "${clients.catalog.url}", fallbackFactory = CatalogFallback.Factory.class)
public interface CatalogClient {
    @GetMapping("/api/v1/items")
    String list();

    @Component
    class Factory implements org.springframework.cloud.openfeign.FallbackFactory<CatalogClient> {
        @Override
        public CatalogClient create(Throwable cause) {
            return () -> {
                // логирование причины + возврат безопасной заглушки
                return "[]";
            };
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.fallback

import org.springframework.cloud.openfeign.FallbackFactory
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.stereotype.Component
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "catalog", url = "\${clients.catalog.url}", fallbackFactory = CatalogFallbackFactory::class)
interface CatalogClient {
    @GetMapping("/api/v1/items")
    fun list(): String
}

@Component
class CatalogFallbackFactory : FallbackFactory<CatalogClient> {
    override fun create(cause: Throwable): CatalogClient = object : CatalogClient {
        override fun list(): String = "[]"
    }
}
```

---

## Категоризация: сетевые/временные ошибки vs бизнес-ошибки (4xx) — разные реакции

Ключ к устойчивости — корректная категоризация отказов. Сетевые и временные ошибки — это тайм-ауты, `IOException`, `503 Service Unavailable`, «разрыв соединения»; они подлежат ретраю и попадают под Circuit Breaker. Бизнес-ошибки — это «контракт нарушен» или «сущность не найдена»: 400/401/403/404/409/422; ретраи здесь вредны и создают лишнюю нагрузку.

ErrorDecoder — место, где мы материализуем эту границу. Он должен выбрасывать **разные** типы исключений: `TransientUpstreamException` для временных сбоев и `BusinessException`/производные для 4xx. Дальше Retry/CB настраиваются на повтор только по `TransientUpstreamException`.

Есть «серые зоны». Код 429 (`Too Many Requests`) может быть временным — тут уместен один повтор с джиттером и уважением `Retry-After`. Код 408 (`Request Timeout`) зависит от контекста, но чаще относится к временным проблемам канала. Эти исключения разумно включить в список ретраев.

Ошибки аутентификации/авторизации (401/403) — это всегда бизнес-ошибки для клиента. Повторять их бессмысленно; нужно обновлять токены или проверять роли. Полезно включать в исключение краткий контекст: audience токена, скоупы, ресурс.

Валидационные ошибки 422 часто несут подробные поля `errors`. Их нужно логировать структурированно, а не как строку, чтобы позже агрегировать причины и улучшать клиента. Это не про устойчивость, а про качество интеграции.

Граница «бизнес/техника» должна быть одинакова во всех командах. Иначе при миграциях/рефакторингах поведение ретраев и брейкера будет непредсказуемым. Внесите базовые классы исключений в общий модуль и используйте единообразно.

Никогда не ретрайте небезопасные POST без идемпотентности. Даже если сервер ответил 500, операция могла выполниться. Ретрая без ключа создаст дубликаты. Для POST действуют отдельные правила — либо ключ, либо ручной разбор кейса.

Категоризацию полезно отражать в метриках. Отдельные счётчики `business_error_total{code=...}` и `transient_error_total{kind=...}` дадут прозрачность, а алерты по «всплеску transient» будут ранним индикатором деградации апстрима.

В WireMock-тестах эмулируйте обе категории и проверяйте, что Retry действительно срабатывает только на transient, а бизнес-ошибки проходят без повтора и корректно маппятся. Это убережёт от «случайно включили ретраи на всё».

И наконец, документируйте исключения публичных фасадов. Верхние уровни должны понимать, что `UserNotFoundException` не ретраится, а `UpstreamTimeoutException` — ретраится в пределах бюджета.

**Java — базовая иерархия исключений + ErrorDecoder категоризации**

```java
package com.example.classify;

import feign.Response;
import feign.codec.ErrorDecoder;

public class ClassifyingErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        int s = response.status();
        if (s == 404) return new BusinessException.NotFound("not found");
        if (s == 409) return new BusinessException.Conflict("conflict");
        if (s == 422) return new BusinessException.Validation("validation failed");
        if (s == 429 || s == 503 || s >= 500) return new TransientUpstreamException("transient " + s);
        return new BusinessException.Generic("http " + s);
    }

    public static class TransientUpstreamException extends RuntimeException { public TransientUpstreamException(String m){super(m);} }
    public static sealed class BusinessException extends RuntimeException permits BusinessException.NotFound, BusinessException.Conflict, BusinessException.Validation, BusinessException.Generic {
        public BusinessException(String m){super(m);}
        public static final class NotFound extends BusinessException { public NotFound(String m){super(m);} }
        public static final class Conflict extends BusinessException { public Conflict(String m){super(m);} }
        public static final class Validation extends BusinessException { public Validation(String m){super(m);} }
        public static final class Generic extends BusinessException { public Generic(String m){super(m);} }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.classify

import feign.Response
import feign.codec.ErrorDecoder

class ClassifyingErrorDecoder : ErrorDecoder {
    override fun decode(methodKey: String, response: Response): Exception =
        when (val s = response.status()) {
            404 -> BusinessException.NotFound("not found")
            409 -> BusinessException.Conflict("conflict")
            422 -> BusinessException.Validation("validation failed")
            429, 503 -> TransientUpstreamException("transient $s")
            in 500..599 -> TransientUpstreamException("transient $s")
            else -> BusinessException.Generic("http $s")
        }
}

class TransientUpstreamException(message: String) : RuntimeException(message)
sealed class BusinessException(message: String) : RuntimeException(message) {
    class NotFound(message: String) : BusinessException(message)
    class Conflict(message: String) : BusinessException(message)
    class Validation(message: String) : BusinessException(message)
    class Generic(message: String) : BusinessException(message)
}
```

---

## Политика повторов/тайм-аутов/CB для разных эндпоинтов (пер-метод через конфиг)

Не все методы равны. «Список товаров» быстрый и частый, «выгрузка отчёта» редкая и тяжёлая. Пытаться укрыть всё набором общих значений — ошибка. В OpenFeign есть «крючок» `configKey` для пер-методной настройки тайм-аутов, а в Resilience4j легко завести отдельные инстансы Retry/CB с собственными именами.

Пер-методный `readTimeout` позволяет не растягивать дефолты. Пусть у клиента дефолт 2 секунды, а `downloadReport` получит 10. Это сохраняет отзывчивость остальных вызовов и не «топит» пул потоков. В yaml вы настраиваете `feign.client.config.{Interface#method(args)}`.

Resilience4j именует инстансы политик строкой. Дайте методам говорящие имена `catalog-list-retry`, `reports-download-cb`. Тогда пороги можно править без перекомпиляции, просто меняя конфигурацию, и метрики будут читабельны. Это важно для быстрой калибровки на инцидентах.

Опасайтесь дублирования ретраев. Если вы включили Feign `Retryer` и ещё навесили Resilience4j Retry на фасад, получите двойные попытки и непредсказуемые задержки. Выберите один уровень и документируйте это.

Под разные эндпоинты могут требоваться разные `connectTimeout`. Например, внешний геокодер через интернет — 2 секунды на connect, а внутренний сервис — 300 мс. Пер-метод такого не сделать, но пер-клиент — да. Если очень нужно, разделите интерфейсы по «классам» вызовов.

Согласовывайте пер-методные настройки с брейкером. Если «длинный» метод имеет `slowCallDurationThreshold=10s`, `readTimeout` должен быть не меньше; иначе брейкер начнёт считать «медленные» вызовы, а клиент будет их раньше обрывать — и поведение станет рваным.

Метрики и алерты на пер-метод. В прометеевских лейблах держите `client`, `method`, `status`. Это позволит видеть конкретную деградацию, а не загадочный «рост отказов» по всему клиенту. Грубые агрегаты бесполезны в RCA.

Покройте пер-методные ключи тестами конфигурации. Опечатка в `Interface#method` молча отключает настройку и включает дефолт. Дешёвый интеграционный тест с чтением `Environment` спасает часы поиска.

Для REST-клиентов, генерирующих код OpenAPI, пер-метод можно зашить на слой над сгенерённым клиентом, чтобы не редактировать автогенерат. Это баланс между гибкостью и поддерживаемостью.

И, наконец, помните о простоте. Если пер-методных различий слишком много, стоит задуматься о разбиении клиента или о пересмотре SLA. Чрезмерная кастомизация усложняет сопровождение.

**application.yml — пер-методные тайм-ауты и политики**

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 1000
        readTimeout: 2000
      reports#ReportsClient#download(String):
        readTimeout: 10000

resilience4j:
  retry:
    instances:
      catalog-list-retry:
        maxAttempts: 3
        waitDuration: 200ms
        retryExceptions:
          - feign.RetryableException
          - com.example.classify.TransientUpstreamException
  circuitbreaker:
    instances:
      reports-download-cb:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        failureRateThreshold: 50
        slowCallDurationThreshold: 10s
        slowCallRateThreshold: 50
        waitDurationInOpenState: 10s
```

**Java — разные политики на методы в фасаде**

```java
package com.example.permethod;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

@Service
public class ReportsFacade {
    private final ReportsClient client;
    public ReportsFacade(ReportsClient client) { this.client = client; }

    @Retry(name = "catalog-list-retry")
    public String listCatalog() { return client.catalog(); }

    @CircuitBreaker(name = "reports-download-cb", fallbackMethod = "fallbackDownload")
    public Resource download(String id) { return client.download(id); }

    private Resource fallbackDownload(String id, Throwable t) {
        return org.springframework.core.io.ByteArrayResource.EMPTY;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.permethod

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.retry.annotation.Retry
import org.springframework.core.io.Resource
import org.springframework.stereotype.Service

@Service
class ReportsFacade(private val client: ReportsClient) {

    @Retry(name = "catalog-list-retry")
    fun listCatalog(): String = client.catalog()

    @CircuitBreaker(name = "reports-download-cb", fallbackMethod = "fallbackDownload")
    fun download(id: String): Resource = client.download(id)

    @Suppress("unused")
    fun fallbackDownload(id: String, t: Throwable): Resource =
        org.springframework.core.io.ByteArrayResource(ByteArray(0))
}
```

---

## Единый обработчик и телеметрия ошибок, correlation-id в исключениях

Единый обработчик ошибок в веб-слое и согласованная телеметрия замыкают картину устойчивости. Клиентские исключения должны «переводиться» в стандартный формат ответа (например, `application/problem+json`) и сопровождаться корреляционным идентификатором. Это позволяет быстро найти связанный кусок логов клиента и провайдера.

Корреляционный идентификатор («request id») должен путешествовать по всей трассе: приходить из фронта или генерироваться на краю, прокидываться в Feign через `RequestInterceptor` и возвращаться в ответы. Тогда любое исключение можно увязать с исходным запросом и записью у апстрима.

Добавляйте correlation-id в сообщение исключения нежелательно, лучше — в поле исключения и обязательно — в теги метрик и логи. Так вы избежите протечки в пользовательские ответы и сохраните структурированность.

Micrometer — стандартный путь публиковать счётчики/таймеры. При отработке ErrorDecoder или фоллбэка инкрементируйте счётчик с лейблами `client`, `code`, `category`. Для брейкера и ретраев метрики уже есть из Resilience4j; остаётся их собрать на дашборде.

`@ControllerAdvice` в Web-слое может маппить доменные исключения в `ProblemDetail`. Добавляйте туда `instance` с correlation-id и короткое `type`. Это даст фронтам и саппорту единый язык общения при ошибках.

На уровне логов используйте MDC для `X-Request-Id`. В интерсепторе Feign читайте его из MDC и прокидывайте в заголовки. Тогда все логи, включая сетевой стек, будут помечены одним и тем же id.

Не засоряйте метрики high-кардинальностью. Корреляционный id в метриках не нужен, но полезен в логах. В метриках оставляйте агрегирующие теги: клиент, метод, код ответа, категория.

Проверяйте, что mask-фильтры применяются и к ошибкам. Если провайдер прислал PII в теле ошибки, ваш обработчик не должен её вернуть наружу. В ответ мы публикуем безопасные сообщения, детали — в логи.

Наконец, энд-ту-энд тест проверьте: сгенерируйте `X-Request-Id`, пройдите через контроллер → фасад → Feign, спровоцируйте ошибку и убедитесь, что id есть в заголовках, логах и в ProblemDetail. Это гарантия наблюдаемости.

**Java — RequestInterceptor для X-Request-Id + ControllerAdvice c ProblemDetail**

```java
package com.example.telemetry;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import io.micrometer.core.instrument.MeterRegistry;
import org.slf4j.MDC;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import java.util.UUID;

public class CorrelationInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String rid = MDC.get("X-Request-Id");
        if (rid == null || rid.isBlank()) {
            rid = UUID.randomUUID().toString();
            MDC.put("X-Request-Id", rid);
        }
        template.header("X-Request-Id", rid);
    }
}

@ControllerAdvice
class GlobalExceptionHandler {
    private final MeterRegistry meter;

    GlobalExceptionHandler(MeterRegistry meter) { this.meter = meter; }

    @ExceptionHandler(com.example.feign.error.ProblemErrorDecoder.ResourceNotFoundException.class)
    public ProblemDetail notFound(Exception ex) {
        meter.counter("client_business_error_total", "code", "404").increment();
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("Resource not found");
        pd.setDetail("Запрошенный ресурс не найден");
        pd.setProperty("requestId", MDC.get("X-Request-Id"));
        return pd;
    }

    @ExceptionHandler(com.example.classify.TransientUpstreamException.class)
    public ProblemDetail upstream(Exception ex) {
        meter.counter("client_transient_error_total", "code", "5xx").increment();
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.SERVICE_UNAVAILABLE);
        pd.setTitle("Upstream temporarily unavailable");
        pd.setDetail("Сервис временно недоступен, попробуйте позже");
        pd.setProperty("requestId", MDC.get("X-Request-Id"));
        return pd;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.telemetry

import feign.RequestInterceptor
import feign.RequestTemplate
import io.micrometer.core.instrument.MeterRegistry
import org.slf4j.MDC
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.ControllerAdvice
import org.springframework.web.bind.annotation.ExceptionHandler
import java.util.UUID

class CorrelationInterceptor : RequestInterceptor {
    override fun apply(template: RequestTemplate) {
        var rid = MDC.get("X-Request-Id")
        if (rid.isNullOrBlank()) {
            rid = UUID.randomUUID().toString()
            MDC.put("X-Request-Id", rid)
        }
        template.header("X-Request-Id", rid)
    }
}

@ControllerAdvice
class GlobalExceptionHandler(private val meter: MeterRegistry) {

    @ExceptionHandler(com.example.feign.error.ProblemErrorDecoder.ResourceNotFoundException::class)
    fun notFound(ex: Exception): ProblemDetail {
        meter.counter("client_business_error_total", "code", "404").increment()
        return ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
            title = "Resource not found"
            detail = "Запрошенный ресурс не найден"
            setProperty("requestId", MDC.get("X-Request-Id"))
        }
    }

    @ExceptionHandler(com.example.classify.TransientUpstreamException::class)
    fun upstream(ex: Exception): ProblemDetail {
        meter.counter("client_transient_error_total", "code", "5xx").increment()
        return ProblemDetail.forStatus(HttpStatus.SERVICE_UNAVAILABLE).apply {
            title = "Upstream temporarily unavailable"
            detail = "Сервис временно недоступен, попробуйте позже"
            setProperty("requestId", MDC.get("X-Request-Id"))
        }
    }
}
```

**Gradle — Micrometer Prometheus (оба DSL)**
*Groovy DSL*

```groovy
dependencies { implementation 'io.micrometer:micrometer-registry-prometheus' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("io.micrometer:micrometer-registry-prometheus") }
```

# 7. Безопасность и пропагация контекста

## OAuth2 Client Credentials: `OAuth2AuthorizedClientManager` + `RequestInterceptor` для Bearer-токена

В интеграциях сервис-к-сервису самый частый вариант аутентификации — поток OAuth2 Client Credentials. Приложение получает access-token у Authorization Server по паре `client_id/client_secret` и добавляет заголовок `Authorization: Bearer …` ко всем исходящим запросам клиента. В Spring экосистеме это реализуется через `spring-security-oauth2-client` и тонкую интеграцию с Feign через `RequestInterceptor`.

Ключевая деталь — не держать токен «на руках», а делегировать его жизненный цикл `OAuth2AuthorizedClientManager`. Он отвечает за получение и обновление токена, учитывает `expires_in` и `scope`, и безопасно кэширует значения в памяти. Это снимает необходимость «изобретать» собственный кэш и бороться с гонками при истечении.

Конфигурация начинается с регистрации клиента по `spring.security.oauth2.client.registration.*` и `provider.*` в `application.yml`. Важны корректные `token-uri`, набор `scope` и тип потока `client_credentials`. Для продуктивной среды секреты подтягивают из Vault/KMS, а в приложении используют только алиасы.

Далее мы создаём `OAuth2AuthorizedClientManager` на основе `ClientRegistrationRepository` и `OAuth2AuthorizedClientService` или `OAuth2AuthorizedClientRepository`. Для серверных приложений без пользовательских сессий подходит `AuthorizedClientServiceOAuth2AuthorizedClientManager`.

Интеграция с Feign делается через `RequestInterceptor`. На каждом запросе мы запрашиваем у менеджера авторизованного клиента по `clientRegistrationId`, получаем `OAuth2AccessToken` и добавляем заголовок `Authorization`. Если токен протух — менеджер прозрачно освежит его.

Нужно учесть отказоустойчивость: если `token-uri` недоступен, вы не должны блокировать все исходящие вызовы. Имеет смысл оборачивать токено-получение в короткие тайм-ауты и включать Circuit Breaker на путь к IdP, чтобы не устроить лавину.

Логи не должны содержать значения токенов. Уровень логирования Feign оставляйте BASIC/HEADERS с маскированием `Authorization`. Метрики полезно помечать флагом «токен свеж/из кэша», но конкретные токены — никогда.

С точки зрения производительности важно избегать синхронной гонки за токеном. Менеджер уже сериализует обновления, но если у вас десятки клиентских бинов с одним и тем же `registrationId`, объединяйте их либо разделяйте кэш на уровень регистрации.

Для нескольких провайдеров можно завести по одному интерсептору на регистрацию и вешать его на нужные `@FeignClient(configuration=…)`. Это лучше, чем один универсальный интерсептор с `if/else` по URL.

Проверяйте аудиторию и набор скоупов токена на стороне ресурс-сервера. Клиент не должен обладать правами шире, чем ему необходимо по принципу наименьших привилегий.

Наконец, тесты: подменяйте `OAuth2AuthorizedClientManager` стабом, возвращающим детерминированный токен. Это позволяет стабильно проверять заголовки без реального IdP и гонок по времени.

**Gradle (Groovy)**

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
}
```

**application.yml**

```yaml
spring:
  security:
    oauth2:
      client:
        provider:
          myidp:
            token-uri: https://idp.example.com/oauth2/token
        registration:
          client-a:
            provider: myidp
            authorization-grant-type: client_credentials
            client-id: ${OAUTH_CLIENT_ID}
            client-secret: ${OAUTH_CLIENT_SECRET}
            scope: [read, write]
```

**Java — Bearer для Feign через OAuth2AuthorizedClientManager**

```java
package com.example.security.oauth;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.*;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.core.OAuth2AccessToken;

@Configuration
public class FeignOAuth2Config {

    @Bean
    OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository repo,
            OAuth2AuthorizedClientService service) {
        var manager = new AuthorizedClientServiceOAuth2AuthorizedClientManager(repo, service);
        manager.setAuthorizedClientProvider(
                OAuth2AuthorizedClientProviderBuilder.builder().clientCredentials().build());
        return manager;
    }

    @Bean
    RequestInterceptor oauth2Interceptor(OAuth2AuthorizedClientManager manager) {
        return new RequestInterceptor() {
            @Override public void apply(RequestTemplate template) {
                var req = OAuth2AuthorizeRequest.withClientRegistrationId("client-a")
                        .principal("feign-client")
                        .build();
                var client = manager.authorize(req);
                if (client == null) return;
                OAuth2AccessToken token = client.getAccessToken();
                template.header("Authorization", "Bearer " + token.getTokenValue());
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.oauth

import feign.RequestInterceptor
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.oauth2.client.*
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository

@Configuration
class FeignOAuth2Config {

    @Bean
    fun authorizedClientManager(
        repo: ClientRegistrationRepository,
        service: OAuth2AuthorizedClientService
    ): OAuth2AuthorizedClientManager =
        AuthorizedClientServiceOAuth2AuthorizedClientManager(repo, service).apply {
            setAuthorizedClientProvider(
                OAuth2AuthorizedClientProviderBuilder.builder().clientCredentials().build()
            )
        }

    @Bean
    fun oauth2Interceptor(manager: OAuth2AuthorizedClientManager): RequestInterceptor =
        RequestInterceptor { template ->
            val request = OAuth2AuthorizeRequest.withClientRegistrationId("client-a")
                .principal("feign-client")
                .build()
            val client = manager.authorize(request) ?: return@RequestInterceptor
            template.header("Authorization", "Bearer ${client.accessToken.tokenValue}")
        }
}
```

---

## Пропагация заголовков: `Authorization`, `X-Request-Id`, трассировка (B3/W3C `traceparent`)

В микросервисной цепочке важны сквозные контексты — корреляционный идентификатор запроса, текущая трасса, иногда пользовательский `Authorization` при токен-релэе. Клиент обязан прозрачно переносить эти заголовки дальше, чтобы логи и трассы сходились.

Стандарт де-факто для распределённого трейсинга сегодня — W3C Trace Context (`traceparent`/`tracestate`). В старых кластерах ещё встречается B3 (`X-B3-TraceId` и др.). В идеале инфраструктура (OTel/Sleuth) добавляет эти заголовки автоматически, но безопаснее иметь тонкий `RequestInterceptor`, который заберёт идентификаторы из MDC и дослёт их.

`X-Request-Id` помогает связать логи фронта, шлюза и бэкенда. Генерировать id лучше на краю (API-шлюз), а внутри только переносить. Если заголовка нет — можно создать, но важно, чтобы он не подавлял уже существующий.

С `Authorization` соблюдайте осторожность. Переносить его стоит только в сценариях token relay, когда downstream доверяет токену исходного клиента и это согласовано политикой безопасности. Часто для внутренних хопов берут сервисный токен Client Credentials вместо пользовательского.

Перенос кастомных заголовков делайте по allow-list, а не по «копировать всё». Это предотвращает утечки чувствительных значений и конфликты с инфраструктурой (например, `Host`, `Content-Length`).

При наличии нескольких форматов трейсинга выбирайте приоритет W3C и добавляйте B3 только для совместимости. Двойной стэк увеличивает кардинальность метрик и усложняет RCA.

Интеграция с Micrometer/OTel обычно сама наполняет MDC. В интерсепторе просто считывайте `traceId`, `spanId` и кладите корректный формат. Не изобретайте генерацию — полагайтесь на инструментирование.

Не включайте корреляционные идентификаторы в теги метрик с высокой кардинальностью. Они нужны в логах и трейсах, а в метриках достаточно `client`/`method`/`status`.

Учтите, что перенос `Authorization` может ломать кэширование на прокси. Если вы делаете кэшируемые GET, держите это в уме и согласовывайте с API-шлюзом.

Тесты на пропагацию — это чёрный ящик с WireMock/MockWebServer: послали запрос и убедились, что нужные заголовки реально ушли. Это защита от регрессов в связке логирования/трассировки.

**Java — интерсептор пропагации из MDC**

```java
package com.example.security.propagation;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.slf4j.MDC;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class PropagationConfig {

    @Bean
    RequestInterceptor contextPropagation() {
        return new RequestInterceptor() {
            @Override public void apply(RequestTemplate template) {
                var rid = MDC.get("X-Request-Id");
                if (rid != null && !rid.isBlank()) template.header("X-Request-Id", rid);
                var traceparent = MDC.get("traceparent"); // OTel bridge
                if (traceparent != null) template.header("traceparent", traceparent);
                var b3 = MDC.get("X-B3-TraceId");
                if (b3 != null) template.header("X-B3-TraceId", b3);
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.propagation

import feign.RequestInterceptor
import org.slf4j.MDC
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class PropagationConfig {
    @Bean
    fun contextPropagation(): RequestInterceptor = RequestInterceptor { t ->
        MDC.get("X-Request-Id")?.takeIf { it.isNotBlank() }?.let { t.header("X-Request-Id", it) }
        MDC.get("traceparent")?.let { t.header("traceparent", it) }
        MDC.get("X-B3-TraceId")?.let { t.header("X-B3-TraceId", it) }
    }
}
```

---

## mTLS: конфиг SSL-контекста для Apache/OkHttp, хранилища сертификатов и пины

При межсервисном трафике в закрытых сетях часто применяют взаимную аутентификацию TLS (mTLS), где и клиент, и сервер предъявляют сертификаты. В Java это настраивается через `KeyStore`/`TrustStore` и `SSLContext`, а для Feign — через конкретный HTTP-клиент (Apache или OkHttp).

Первый шаг — безопасно хранить ключи. В проде ключевые материалы подтягивают из секрет-хранилищ, а в рантайме загружают в `KeyStore`. Пароли на хранилища и ключи не должны попадать в логи и переменные окружения без политики.

Для Apache HttpClient 5 создаётся `SSLConnectionSocketFactory` на базе `SSLContext`, где `KeyManager` берёт приватный ключ клиента, а `TrustManager` — цепочку доверенных CA. Клиент присоединяется к Feign через `new ApacheHttpClient(httpClient)`.

В OkHttp mTLS активируется методом `sslSocketFactory(SSLSocketFactory, X509TrustManager)` и `hostnameVerifier` при необходимости. Следите, чтобы `TrustManager` совпадал с тем, на базе которого построен `SSLSocketFactory`, иначе будет `IllegalArgumentException`.

Certificate pinning даёт дополнительную защиту от подмены CA в цепочке. В OkHttp есть встроенный `CertificatePinner`, в Apache — реализуют проверку в `HostnameVerifier`/`TrustStrategy`. Пины требуют процесса ротации — планируйте dual-pin.

Следите за версией протокола и наборами шифров. Короткие ключи и TLS 1.0/1.1 должны быть выключены. Разрешайте TLS 1.2/1.3 и актуальные шифры, совместимые с инфраструктурой.

Пулы соединений с mTLS имеют большую стоимость рукопожатия. Выигрыш от keep-alive особенно заметен — не «убивайте» соединения слишком агрессивными eviction-параметрами.

Обновление сертификатов должно происходить без рестартов. Для этого держите срок действия ключей разумным и используйте rolling-релиз. Некоторые клиенты позволяют периодически пересоздавать `SSLContext`, но это сложнее.

Логируйте только метаданные сертификатов (issuer/subject/serial), но никогда — приватные ключи. Диагностика mTLS часто упирается в несоответствие цепочки — метаданные помогут RCA.

**application.yml (пути к хранилищам через секреты)**

```yaml
security:
  mtls:
    key-store: ${MTLS_KEYSTORE_PATH}
    key-store-password: ${MTLS_KEYSTORE_PASSWORD}
    trust-store: ${MTLS_TRUSTSTORE_PATH}
    trust-store-password: ${MTLS_TRUSTSTORE_PASSWORD}
```

**Java — Apache HttpClient с mTLS для Feign**

```java
package com.example.security.mtls;

import feign.Client;
import feign.httpclient.ApacheHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.apache.hc.core5.ssl.SSLContexts;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.net.ssl.SSLContext;
import java.io.FileInputStream;
import java.security.KeyStore;

@Configuration
public class FeignMtlsApacheConfig {

    @Bean
    public Client feignClient(
            @Value("${security.mtls.key-store}") String ksPath,
            @Value("${security.mtls.key-store-password}") String ksPass,
            @Value("${security.mtls.trust-store}") String tsPath,
            @Value("${security.mtls.trust-store-password}") String tsPass) throws Exception {

        KeyStore ks = KeyStore.getInstance("PKCS12");
        try (var in = new FileInputStream(ksPath)) { ks.load(in, ksPass.toCharArray()); }
        KeyStore ts = KeyStore.getInstance("JKS");
        try (var in = new FileInputStream(tsPath)) { ts.load(in, tsPass.toCharArray()); }

        SSLContext ssl = SSLContexts.custom()
                .loadKeyMaterial(ks, ksPass.toCharArray())
                .loadTrustMaterial(ts, null)
                .build();

        var http = HttpClientBuilder.create()
                .setSSLContext(ssl)
                .build();
        return new ApacheHttpClient(http);
    }
}
```

**Kotlin — OkHttp с mTLS и pinning**

```kotlin
package com.example.security.mtls

import feign.Client
import feign.okhttp.OkHttpClient
import okhttp3.CertificatePinner
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.io.FileInputStream
import java.security.KeyStore
import javax.net.ssl.*

@Configuration
class FeignMtlsOkHttpConfig(
    @Value("\${security.mtls.key-store}") private val ksPath: String,
    @Value("\${security.mtls.key-store-password}") private val ksPass: String,
    @Value("\${security.mtls.trust-store}") private val tsPath: String,
    @Value("\${security.mtls.trust-store-password}") private val tsPass: String
) {
    @Bean
    fun feignClient(): Client {
        val ks = KeyStore.getInstance("PKCS12").apply {
            FileInputStream(ksPath).use { load(it, ksPass.toCharArray()) }
        }
        val ts = KeyStore.getInstance("JKS").apply {
            FileInputStream(tsPath).use { load(it, tsPass.toCharArray()) }
        }
        val kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm()).apply {
            init(ks, ksPass.toCharArray())
        }
        val tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm()).apply {
            init(ts)
        }
        val ssl = SSLContext.getInstance("TLS").apply { init(kmf.keyManagers, tmf.trustManagers, null) }
        val x509Tm = tmf.trustManagers.first { it is X509TrustManager } as X509TrustManager

        val pinner = CertificatePinner.Builder()
            .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .build()

        val ok = okhttp3.OkHttpClient.Builder()
            .sslSocketFactory(ssl.socketFactory, x509Tm)
            .certificatePinner(pinner)
            .build()

        return OkHttpClient(ok)
    }
}
```

---

## Basic/API-Key: `RequestInterceptor` или свойства клиента

Некоторые провайдеры пускают по простым схемам — Basic Auth или статический API-ключ в заголовке/квери. Это быстро, но требует дисциплины: ключи нужно хранить как секреты и никогда не писать в логи.

Для Basic существует готовый `BasicAuthRequestInterceptor`, который добавит `Authorization: Basic …`. Его можно подключить бином в конфигурации конкретного клиента. Для API-ключа обычно делают тонкий `RequestInterceptor`, который добавляет заголовок `X-Api-Key` или параметр `api_key`.

Иногда провайдер просит передавать ключ в квери. Это удобно для curl, но опасно: ключ попадёт в логи прокси и APM. По возможности добивайтесь заголовка. Если выбора нет — маскируйте квери в логах.

Переключение ключей и ротация — эксплуатационный процесс. Храните версии и дедлайны в документации, чтобы не ловить внезапные 401. В коде держите ключи в алисах переменных окружения.

Для нескольких клиентов с разными ключами делайте отдельные `configuration=` классы и не используйте «универсальный» интерсептор. Это снижает риск перепутать ключи и отправить не туда.

Важный момент — стараться не смешивать схемы. Если часть методов требует API-ключа, а часть — OAuth2, лучше разделить интерфейсы. Сложные `if` в интерсепторе — источник скрытых багов.

Логи должны маскировать значения ключей. Даже если вы пишете только HEADERS-уровень, обязательно заменяйте значение заголовка ключа на `***`. Это защищает от утечек в централизованные хранилища логов.

Тестируйте интерсепторы на перехват запросов и отсутствие дублей заголовков. Некоторые серверы чувствительны к нескольким `Authorization` — убедитесь, что добавляете один.

Наконец, при миграции с API-ключей на OAuth2 держите клиентов в параллели и включайте фиче-флаги, чтобы переключаться, не перекладывая ответственность на релиз.

**Java — Basic и API-Key интерсепторы**

```java
package com.example.security.basickey;

import feign.RequestInterceptor;
import feign.auth.BasicAuthRequestInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BasicApiKeyConfig {

    @Bean
    public BasicAuthRequestInterceptor basicAuth() {
        return new BasicAuthRequestInterceptor(System.getenv("BASIC_USER"), System.getenv("BASIC_PASS"));
    }

    @Bean
    public RequestInterceptor apiKey() {
        return template -> template.header("X-Api-Key", System.getenv("API_KEY"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.basickey

import feign.RequestInterceptor
import feign.auth.BasicAuthRequestInterceptor
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class BasicApiKeyConfig {

    @Bean
    fun basicAuth(): BasicAuthRequestInterceptor =
        BasicAuthRequestInterceptor(System.getenv("BASIC_USER"), System.getenv("BASIC_PASS"))

    @Bean
    fun apiKey(): RequestInterceptor = RequestInterceptor { t ->
        t.header("X-Api-Key", System.getenv("API_KEY"))
    }
}
```

---

## Маскирование чувствительных данных в логах (`Logger.Level`, custom Logger)

Логи Feign помогают RCA, но по умолчанию могут раскрывать приватные данные. Стратегия проста: минимальный уровень по умолчанию (BASIC), маскирование чувствительных заголовков и отключение FULL в проде. Когда нужен подробный трейс — поднимайте уровень адресно, через per-client конфиг и коротким TTL.

Маскировать нужно `Authorization`, `Cookie`, `Set-Cookie`, любые `X-…-Token`, `X-Api-Key`, а также квери-параметры с ключами. Проще всего написать собственный `Logger`, который перед печатью заголовков проходит allow/deny-list и заменяет значение на `***`.

Для multipart и бинарных ответов включение BODY-логирования — источник проблем. Вы получите гигабайты логов и риск утечек. Даже на DEV лучше HEADERS. Если BODY нужен для отладки JSON — включайте на конкретный клиент и снимаете быстро.

Не забывайте про GDPR/ПДн. ФИО/телефоны/адреса тоже подлежат маскированию или исключению из логов. Часть API придётся диагностировать на основе метаданных (тип ресурса, id), не печатая полезную нагрузку.

Критично сделать защиту «по умолчанию». Не полагайтесь на дисциплину разработчиков — один забытый `loggerLevel: FULL` в prod-профиле испортит вам неделю. Автоматические проверки в CI и unit-тесты конфигурации обязаны это ловить.

Метрики логов не заменяют логи, но полезны: количество «маскированных» заголовков, частота включения FULL, размер логируемых сообщений. Это помогает находить «болтливые» клиенты.

В распределённых системах логируйте также correlation-id и короткий digest тела (например, хеш), если это допустимо политиками. Это помогает «сшивать» события без раскрытия данных.

Custom Logger должен быть простым и быстрым — не форматируйте строки тяжёлым образом, избегайте больших аллокаций. Помните, что логгер живёт в горячем пути каждого вызова.

Тестируйте маскирование отдельно: давайте на вход набор заголовков, проверяйте ожидаемый вывод. Это гарантирует неизменность поведения при апгрейде.

Документируйте список маскируемых заголовков и запрещённых уровней в проде. Это часть чек-листа безопасности релиза.

**Java — кастомный Feign Logger с маскированием**

```java
package com.example.logging;

import feign.Logger;
import feign.Request;
import feign.Response;

import java.io.IOException;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class RedactingLogger extends Logger {
    private static final Set<String> SENSITIVE = Set.of(
            "authorization", "cookie", "set-cookie", "x-api-key", "x-auth-token");

    @Override
    protected void logRequest(String configKey, Level logLevel, Request request) {
        var redacted = request.toBuilder().headers(redact(request.headers())).build();
        super.logRequest(configKey, Level.HEADERS, redacted);
    }

    @Override
    protected Response logAndRebufferResponse(String configKey, Level logLevel, Response response, long elapsedTime) throws IOException {
        var redacted = response.toBuilder().headers(redact(response.headers())).build();
        return super.logAndRebufferResponse(configKey, Level.HEADERS, redacted, elapsedTime);
    }

    private Map<String, List<String>> redact(Map<String, List<String>> headers) {
        return headers.entrySet().stream().collect(java.util.stream.Collectors.toMap(
                Map.Entry::getKey,
                e -> SENSITIVE.contains(e.getKey().toLowerCase())
                        ? List.of("***")
                        : e.getValue()
        ));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.logging

import feign.Logger
import feign.Request
import feign.Response

class RedactingLogger : Logger() {
    private val sensitive = setOf("authorization", "cookie", "set-cookie", "x-api-key", "x-auth-token")

    override fun logRequest(configKey: String, logLevel: Level, request: Request) {
        val redacted = request.toBuilder().headers(redact(request.headers())).build()
        super.logRequest(configKey, Level.HEADERS, redacted)
    }

    override fun logAndRebufferResponse(configKey: String, logLevel: Level, response: Response, elapsedTime: Long): Response {
        val redacted = response.toBuilder().headers(redact(response.headers())).build()
        return super.logAndRebufferResponse(configKey, Level.HEADERS, redacted, elapsedTime)
    }

    private fun redact(headers: Map<String, Collection<String>>): Map<String, Collection<String>> =
        headers.mapValues { (k, v) -> if (sensitive.contains(k.lowercase())) listOf("***") else v }
}
```

**application.yml — только HEADERS в prod**

```yaml
feign:
  client:
    config:
      default:
        loggerLevel: HEADERS
```

---

## CORS/CSRF не релевантны клиенту, но важно соответствие политике провайдера

Важно не путать понятия: CORS и CSRF — это механизмы браузера и серверной защиты от межсайтовых атак. Feign — серверная библиотека; ограничения происхождения и CSRF-токены к нему неприменимы. Однако клиент должен соответствовать политике провайдера: правильные Origin, корректная аутентификация и отсутствие лишних кросс-доменных заголовков.

CORS контролирует браузер: он добавляет `Origin` и ожидает `Access-Control-Allow-*` в ответе. Серверные клиенты CORS не касаются, и попытки «подстроить» заголовки CORS в Feign бессмысленны. Если внешний API вдруг требует `Origin`, это нестандартная политика, и её следует документировать отдельно.

CSRF защищает stateful-сессии и формы от подделки запросов. В сценариях сервис-к-сервису мы используем stateless-аутентификацию (Bearer/mTLS/API-Key); CSRF-токены не участвуют. Если вы всё-таки вызываете старый stateful-сервер, то либо получите 403, либо вам придётся реализовать клиентское получение токена и отправку его заголовком — но это «запах» архитектуры.

Строгие политики заголовков у провайдера всё равно могут влиять: например, запрет неразрешённых заголовков или обязательные security-заголовки на ответ. Клиент должен быть готов обрабатывать ошибки валидации с понятной диагностикой и не «засорять» запрос произвольными полями.

Если ваш сервис является BFF для браузера, CORS настраивается на стороне вашего сервера, а не клиента. При ошибках CORS пользователю кажется, что «не ходит клиент», хотя на самом деле браузер блокирует ответ. Разделяйте зоны ответственности.

Логирование и трейсинг по-прежнему актуальны. Даже если CORS к вам не относится, добавляйте `X-Request-Id` и trace-заголовки — это поможет быстро найти, почему провайдер ответил 403/401.

В документации API провайдера раздел «клиент-сервер» обычно не упоминает CORS. Любые требования к `Origin` или рефереру должны вызывать вопросы и согласование, поскольку они ломают headless-сценарии.

На уровне безопасности не отправляйте лишние заголовки: `Referer`, `Origin` и т. п. это не ваша забота. Список переносимых полей должен быть минимальным и утверждённым.

В тестах воспроизводите только то, что контролируете. Нет смысла пытаться «эмулировать» preflight OPTIONS — Feign не браузер, он пошлёт сразу рабочий метод. Разницы в поведении быть не должно.

Если в архитектуре встречается SPA→BFF→провайдер, то CORS живёт между браузером и BFF. Между BFF и провайдером — аутентификация и сети. Делайте чек-лист: что проверять на каком уровне, чтобы не путать ошибки.

Наконец, обучайте команду. Ошибки «про CORS» часто попадают в бэклог команды интеграций, хотя решаются на фронте/шлюзе. Ясное разграничение сэкономит недели.

**Java — демонстрационный интерсептор, который не добавляет CORS/CSRF**

```java
package com.example.security.cors;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CleanHeadersConfig {
    @Bean
    public RequestInterceptor cleanHeaders() {
        return new RequestInterceptor() {
            @Override public void apply(RequestTemplate template) {
                template.header("Origin"); // не устанавливаем значение
                template.header("X-CSRF-Token"); // не добавляем без явного требования
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.cors

import feign.RequestInterceptor
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class CleanHeadersConfig {
    @Bean
    fun cleanHeaders(): RequestInterceptor = RequestInterceptor { t ->
        // умышленно не добавляем CORS/CSRF заголовки
    }
}
```
# 8. Сервис-дискавери и балансировка

## Разрешение `name` через Spring Cloud LoadBalancer (Eureka/Consul, без Ribbon)

Внутренние HTTP-вызовы в микросервисах удобнее адресовать не по конкретным URL, а по логическим именам сервиса. В `@FeignClient(name = "inventory")` это имя резолвится через Spring Cloud LoadBalancer (SCLB), который запрашивает реализацию `DiscoveryClient` (Eureka/Consul/Kubernetes) и подставляет актуальный хост/порт инстанса в момент вызова. Такой подход убирает «хардкод» адресов и естественно переживает масштабирование.

SCLB — клиентская балансировка: выбор узла делается в вашем процессе, без внешнего балансировщика уровня L7. Это даёт контроль над политиками (round-robin, zone-preference), кэшированием списка инстансов и health-фильтрацией. За надёжность (тайм-ауты/ретраи/CB) по-прежнему отвечаете вы.

Регистрация в реестре — обязанность провайдера сервиса. Экземпляр публикует адрес, метаданные (зона, версия, теги) и поддерживает «сердцебиение». Потребитель получает список и выбирает инстанс на каждый запрос. Если список пуст — корректное поведение клиента: выбросить исключение и сработать через fallback/CB.

Важно различать `name` и `url` в `@FeignClient`. Явно заданный `url` отключает discovery: резолв не нужен, будет прямой вызов. Это удобно для внешних провайдеров, тестов и аварийных обходов, но для внутренних вызовов предпочтительнее `name`, чтобы сохранять гибкость.

В Kubernetes есть два пути. Первый — Spring Cloud Kubernetes Discovery: SCLB получает список подов/эндпоинтов и применяет свои политики. Второй — вообще без Discovery, по DNS `http://service.namespace:port` (просто, но без health-aware и зонных предпочтений). Для тонкой настройки выбирают первый.

SCLB имеет кэш списка инстансов, чтобы не дергать реестр на каждый вызов. TTL выбирается компромиссом: десятки секунд обычно хватает, чтобы быть «достаточно свежим» и не создавать лишний трафик к реестру. Кэш не отменяет тайм-ауты — списки всё равно могут устареть.

Отсутствие инстансов не должно приводить к «вечному ожиданию». Сетевые окна Feign/HTTP-клиента и бизнес-дедлайны должны быть настроены так, чтобы отказ фиксировался быстро и предсказуемо. Это критично для каскадов ретраев.

Наблюдаемость: логируйте имя сервиса, итоговый хост/порт, результат балансировки и статус ответа. В метриках полезны таймеры per-client/per-method и счётчики ошибок. Для трассировки переносите `traceparent`/`X-Request-Id` через интерсептор.

Безопасность: name-based вызовы часто используются внутри периметра и сопровождаются сервисными токенами (client-credentials). Не путайте с token-relay пользовательских JWT — это другой сценарий и другой клиент.

Для локальной разработки discovery можно отключить и задать `url` на WireMock, не меняя кода клиента. На CI это ускоряет тесты и убирает зависимость от внешнего стенда с Eureka/Consul.

Ribbon устарел: современные релизы Spring Cloud используют SCLB. Если у вас всё ещё есть ribbon-зависимости, выведите их — это упростит граф зависимостей и обновления.

**Gradle (Groovy)**

```groovy
dependencies {
    implementation platform("org.springframework.cloud:spring-cloud-dependencies:2024.0.0")
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client' // или consul-discovery
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    implementation(platform("org.springframework.cloud:spring-cloud-dependencies:2024.0.0"))
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.cloud:spring-cloud-starter-loadbalancer")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
}
```

**application.yml (Eureka пример)**

```yaml
spring:
  application:
    name: catalog
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka:8761/eureka/
```

**Java — резолв по имени сервиса**

```java
package com.example.discovery;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "inventory")
public interface InventoryClient {
    @GetMapping("/api/v1/stock/summary")
    String stockSummary();
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.discovery

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping

@FeignClient(name = "inventory")
interface InventoryClient {
    @GetMapping("/api/v1/stock/summary")
    fun stockSummary(): String
}
```

---

## Политики выбора инстанса: round-robin/Random/Zone-aware, кэш TTL

Round-robin — базовая стратегия: вызовы равномерно распределяются по списку инстансов. Она проста и предсказуема, хорошо работает при однородных узлах. В метриках видно ровное распределение, что помогает SRE в инцидентах.

Random иногда полезен при больших кластерах и «рваных» очередях, где случайность лучше сглаживает локальные всплески. Но объяснимость хуже: доли могут временно заметно гулять, и анализ перекосов сложнее.

Zone-aware добавляет локальность: сначала выбираем инстансы своей availability-zone, затем остальные. Это снижает межзонные RTT и стоимость трафика. Нужны метаданные зоны у инстансов и настройка `spring.cloud.loadbalancer.zone`.

SCLB позволяет подменять балансировщик пер-клиент, не затрагивая других. Это сделайте через `@LoadBalancerClient(name="...", configuration=...)`, где объявите `ReactorLoadBalancer<ServiceInstance>`. Так вы можете использовать RR для одного сервиса и Random — для другого.

Кэширование списка инстансов важно под высокой нагрузкой. Без кэша каждый вызов бьёт в реестр, растёт латентность и создаётся избыточный сетевой чаттер. TTL порядка 10–30 секунд — практичный баланс между свежестью и нагрузкой.

Weighted-подходов «из коробки» нет, но можно реализовать взвешивание в `ServiceInstanceListSupplier`: дублировать «тяжёлые» инстансы или сортировать по весу. Прежде чем усложнять — измерьте, действительно ли RR не справляется.

Переопределение списка инстансов — место для фильтрации: исключить «канареек», выбрать только нужную версию, учесть теги. Это полезно для поэтапных релизов и A/B. Фильтруйте до балансировки — так вы избегаете «слепых» попаданий.

Неудачный выбор политики проявляется как устойчивые перекосы нагрузки или рост p95/p99. Добавьте в дашборд граф распределения трафика по инстансам, чтобы вовремя увидеть проблему.

Стратегии не заменяют отказоустойчивость. Медленный инстанс также будет получать долю трафика; здесь срабатывают тайм-ауты, ретраи и circuit breaker по методам. Балансировка — лишь первый шаг.

Документируйте, кто и где использует нестандартную стратегию, чтобы команда понимала распределение. Это уменьшит сюрпризы при расследовании инцидентов.

**application.yml — зона и кэш**

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 20s
      zone: ru-msk-a
      configurations: zone-preference
```

**Java — RR и Random per-client**

```java
package com.example.lb;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.RoundRobinLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;

@LoadBalancerClient(name = "inventory", configuration = InventoryLbCfg.class)
@LoadBalancerClient(name = "billing", configuration = BillingLbCfg.class)
public class LbClients {}

@Configuration
class InventoryLbCfg {
    @Bean
    ReactorLoadBalancer<ServiceInstance> rr(Environment env, LoadBalancerClientFactory f) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}

@Configuration
class BillingLbCfg {
    @Bean
    ReactorLoadBalancer<ServiceInstance> rnd(Environment env, LoadBalancerClientFactory f) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.lb

import org.springframework.cloud.client.ServiceInstance
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer
import org.springframework.cloud.loadbalancer.core.RoundRobinLoadBalancer
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.env.Environment

@LoadBalancerClient(name = "inventory", configuration = [InventoryLbCfg::class])
@LoadBalancerClient(name = "billing", configuration = [BillingLbCfg::class])
class LbClients

@Configuration
class InventoryLbCfg {
    @Bean
    fun rr(env: Environment, f: LoadBalancerClientFactory): ReactorLoadBalancer<ServiceInstance> {
        val name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME)!!
        return RoundRobinLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier::class.java), name)
    }
}

@Configuration
class BillingLbCfg {
    @Bean
    fun rnd(env: Environment, f: LoadBalancerClientFactory): ReactorLoadBalancer<ServiceInstance> {
        val name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME)!!
        return RandomLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier::class.java), name)
    }
}
```

---

## Health-aware вызовы и отказоустойчивость при частичных сбоях

Даже с хорошей стратегией часть инстансов может деградировать: 5xx, долгие GC-паузы, проблемы сети. Health-aware фильтрация позволяет исключать такие инстансы из пула кандидатов. SCLB умеет периодически опрашивать путь `readiness` (обычно `/actuator/health`) и исключать «красные»/«желтые» узлы.

Проверки периодические, поэтому на них нельзя полагаться как на абсолютную истину. Состояние меняется между опросами, а значит тайм-ауты и брейкеры всё равно должны оставаться в силе. Health — лишь первый барьер.

Путь здоровья должен быть лёгким и кэшируемым. Если `health` тянет тяжёлые зависимости без кэша, вы создадите дополнительную нагрузку и получите ложные негативы. Разделяйте liveness и readiness; первый — максимально простой, второй — с лёгким кэшем.

Health-aware влияет на холодный старт. Новые поды попадут в пул только после «зелёного» статуса. В Kubernetes это управляется `readinessProbe`, в Eureka/Consul — задержкой регистрации. Планируйте прогрев, чтобы не ловить 404/connection refused.

Иногда деградация частичная: один эндпоинт «падает», другие здоровы. Health обычно бинарен и не знает про конкретные методы. Поэтому дополняйте его брейкерами per-endpoint и метриками p95/p99 по методам.

Интервал опроса — компромисс. Слишком частый — лишний трафик и колебания; слишком редкий — долго держите «мёртвые» узлы. Практично 10–30 секунд, в зависимости от динамики кластера.

Метаданные discovery можно использовать для фильтрации до health: версия релиза, теги «canary», зона. Это экономит запросы здоровья и повышает предсказуемость трафика на нужные группы.

Тестирование health-aware делайте с эмуляцией «плохих» ответов на `/actuator/health`: «доказать», что такие инстансы реально исключаются и вызов уходит к здоровым. Это ловит ошибки конфигурации пути/TTL.

Health-aware не отменяет алертов. Держите сигналы на «пустой список инстансов», «высокий процент исключений health-фильтра» и «рост отказов по методам». Это ранние индикаторы инцидента.

В отчётах отделяйте причины отказов: «исключён health-фильтром», «тайм-аут чтения», «брейкер открыт». Это помогает RCA и снижает время восстановления.

**application.yml — включить проверки здоровья**

```yaml
spring:
  cloud:
    loadbalancer:
      health-check:
        enabled: true
        interval: 20s
        path:
          default: /actuator/health
```

**Java — поставщик со здоровьем**

```java
package com.example.lb.health;

import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class InventorySupplierCfg {
    @Bean
    ServiceInstanceListSupplier inventorySupplier(ConfigurableApplicationContext ctx) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withHealthChecks()
                .build(ctx);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.lb.health

import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier
import org.springframework.context.ConfigurableApplicationContext
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class InventorySupplierCfg {
    @Bean
    fun inventorySupplier(ctx: ConfigurableApplicationContext): ServiceInstanceListSupplier =
        ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withHealthChecks()
            .build(ctx)
}
```

---

## Совместимость с Gateway: токен-релэй, маршрутизация/переписывание путей

Spring Cloud Gateway часто стоит на «севере», а Feign-вызовы идут «юг-юг». Если BFF должен сохранять пользовательский контекст, возможен token relay: перенос `Authorization` из входящего запроса во внутренний вызов. Делайте это только там, где это согласовано политиками безопасности.

Gateway нередко переписывает пути (например, `/api/v1/catalog/**` наружу и `/catalog/**` внутрь). Feign-клиенты, обращающиеся **напрямую** к бэкендам, должны использовать внутренние пути, не зависящие от внешних префиксов. Это снижает связанность с API-шлюзом.

Иногда выгодно ходить к бэкендам **через** Gateway (единая аутентификация, кэш, WAF). Тогда `@FeignClient(name="gateway", path="/internal/...")` и маршрутизация на `lb://service`. Плюсы — стандартные фильтры и наблюдаемость; минусы — лишний хоп и SPOF. Влияет на SLO и латентность — учитывайте.

Не путайте два вида токенов: пользовательский JWT и сервисный client-credentials. Первый используют для «сквозного» доступа от имени пользователя, второй — для межсервисной аутентификации. Лучше разделить клиентов и их конфигурации, чтобы не смешивать контексты.

CORS/CSRF живут между браузером и Gateway. Межсервисные вызовы Feign к ним нерелевантны. Не добавляйте «браузерных» заголовков в серверные вызовы — это лишь создаёт шум.

Трассировка должна проходить через Gateway. Совместимость форматов (W3C `traceparent`) упростит склейку. В интерсепторе Feign не затирайте заголовки, уже проставленные шлюзом.

Держите возможность быстро переключать маршрут: «через шлюз» ↔ «напрямую». Это делается профилями и двумя вариантами клиентов, чтобы во время инцидента обойти проблемную точку без релиза кода.

Логи — источник правды. Для путей, переписанных шлюзом, полезно логировать исходный и внутренний путь, чтобы легче сопоставлять трассы и понимать, куда фактически ушёл запрос.

Тестирование: для relay — установите `JwtAuthenticationToken` в `SecurityContext` и проверьте заголовок `Authorization` в исходящем запросе. Для маршрутизации — используйте WireMock и убедитесь, что путь «внутренний», а не внешний.

Наконец, помните про ограничения провайдера: читаемость путей, лимиты на заголовки, требования к авторизации. Gateway может смягчить углы, но стандарты downstream важнее.

**Gateway routes (переписывание и TokenRelay)**

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay
      routes:
        - id: catalog
          uri: lb://catalog
          predicates:
            - Path=/api/v1/catalog/**
          filters:
            - RewritePath=/api/v1/catalog/(?<p>.*), /${p}
```

**Java — user-token relay перехватчик**

```java
package com.example.gateway.relay;

import feign.RequestInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;

@Configuration
public class UserTokenRelayFeignConfig {
    @Bean
    RequestInterceptor userTokenRelay() {
        return template -> {
            var auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth instanceof JwtAuthenticationToken jwt) {
                template.header("Authorization", "Bearer " + jwt.getToken().getTokenValue());
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.gateway.relay

import feign.RequestInterceptor
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken

@Configuration
class UserTokenRelayFeignConfig {
    @Bean
    fun userTokenRelay(): RequestInterceptor = RequestInterceptor { t ->
        val auth = SecurityContextHolder.getContext().authentication
        if (auth is JwtAuthenticationToken) {
            t.header("Authorization", "Bearer ${auth.token.tokenValue}")
        }
    }
}
```

---

## Статический `url` для внешних API и отключение LB при необходимости

Для внешних провайдеров обычно не используют discovery: у них собственный DNS/CDN/Anycast. В `@FeignClient(name="geo", url="\${clients.geo.url}")` вы задаёте базовый адрес, а балансировка ложится на инфраструктуру провайдера. Это снижает сложность конфигурации и упрощает эксплуатацию.

Одновременное указание `name` и `url` делает клиента статическим: SCLB игнорируется. Это удобно как «рубильник» для обхода проблем с discovery или для тестов, когда вы перенаправляете клиента на WireMock.

DNS-кэш и тайм-ауты критичны для внешних сетей. Настройте `connectTimeout` и `readTimeout` консервативно, ограничьте размер пула соединений и включите keep-alive. Для OkHttp можно включить HTTP/2, если провайдер поддерживает — это улучшает мультиплексирование.

Региональность: держите `url` параметризуемым (`eu-api`, `us-api`) и переключаемым без релиза. Это помогает при региональных инцидентах и оптимизации латентности.

Схемы аутентификации у внешних API часто другие: API-Key, Basic, mTLS. Разделяйте интерсепторы и конфиги, чтобы случайно не отправить внутренний сервисный токен наружу.

Загрузка больших тел: используйте `Response`/`InputStream`, чтобы не выгружать всё в память. Стриминг критичен при работе с файлами/медиа — иначе рискуете OOM под нагрузкой.

Логирование: маскируйте секреты и не включайте BODY в проде. Для отладки включайте уровни адресно и на короткое время. Это снижает риск утечек.

Тесты: статический `url` удобно перенаправлять на локальные стабы через `DynamicPropertySource`. Это ускоряет CI и делает тесты детерминированными.

Failover: держите запасные адреса провайдера (если есть) и фиче-флаг для переключения. Внешние инциденты случаются чаще внутренних — нужно уметь быстро «переключить шланг».

Документация: опишите SLO/квоты провайдера и ваши тайм-ауты/ретраи. Это помогает согласовать ожидания с бизнесом и SRE.

**application.yml — адрес внешнего провайдера**

```yaml
clients:
  geo:
    url: https://api.example.com
```

**Java — статический URL**

```java
package com.example.external.geo;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(name = "geo", url = "${clients.geo.url}")
public interface GeoClient {
    @GetMapping("/v1/geocode")
    GeoResult geocode(@RequestParam("q") String q);

    record GeoResult(double lat, double lon) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.external.geo

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestParam

@FeignClient(name = "geo", url = "\${clients.geo.url}")
interface GeoClient {
    @GetMapping("/v1/geocode")
    fun geocode(@RequestParam("q") q: String): GeoResult
}

data class GeoResult(val lat: Double, val lon: Double)
```

---

## Разделение конфигураций для внутренних и внешних клиентов

Чёткое разделение «internal» и «external» клиентов уменьшает ошибки и упрощает эксплуатацию. Внутренние — discovery+SCLB, сервисные токены (client-credentials), корреляция/трейсинг, health-aware список, короткие окна. Внешние — статический `url`, API-Key/mTLS, маскирование логов, другие тайм-ауты и пределы параллелизма.

Структурируйте код: разные пакеты клиентов и два `@Configuration`, подключаемые через `configuration=`. Так вы изолируете интерсепторы и не перепутаете схемы аутентификации. Это особенно важно при одновременном наличии OAuth2 и API-Key.

Резилиентность различается. Внутренним обычно достаточно 1–2 ретраев и коротких тайм-аутов; внешним — аккуратный единичный retry или вовсе без него, плюс более длинный `readTimeout` и строгий circuit breaker. Это отражайте в `application.yml` per-client.

Секреты делите по ролям в Vault/KMS. Доступ к внешним ключам ограничивайте сильнее. Никогда не используйте один и тот же «универсальный» интерсептор для разных клиентов — это источник утечек.

Наблюдаемость: помечайте метрики тегом `integration_type=internal|external`, добавляйте per-client/per-method таймеры и счётчики ошибок. Дашборды и алерты разделяйте — у внешних другие SLO и причины отказов.

Профили облегчают жизнь: `dev` направляет внешние клиенты на WireMock, внутренние — на docker-compose/minikube; `prod` — на реальных провайдеров и discovery. Переключение — свойствами, без изменений кода.

Тесты покрывают оба стека. Для внутренних — эмуляция Discovery/health и проверка балансировки. Для внешних — моки авторизации, mTLS и маскирование логов. Это предотвращает регрессии при апгрейдах Spring Cloud.

Платформенный модуль с «шаблонами» конфигураций (InternalFeignConfig/ExternalFeignConfig) сокращает дублирование и стандартизирует практики. Конкретные сервисы наследуют и донастраивают.

Архитектурный гайдлайн должен описывать критерии выбора корзины. Новые интеграции сразу попадают «куда надо», а не «как получится». Это экономит недели на переупаковку.

При миграции внешнего провайдера «внутрь» вы просто меняете `url` на `name`, включаете discovery и подхватываете внутренние политики. Это ещё один плюс такого разделения.

**Java — две конфигурации и два клиента**

```java
package com.example.split;

import feign.RequestInterceptor;
import feign.auth.BasicAuthRequestInterceptor;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.RoundRobinLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.context.annotation.*;
import org.springframework.core.env.Environment;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient(name = "orders", configuration = InternalFeignConfig.class)
interface OrdersClient {
    @GetMapping("/api/v1/orders/me")
    String myOrders();
}

@FeignClient(name = "payments-ext", url = "${clients.payments.url}", configuration = ExternalFeignConfig.class)
interface PaymentsClient {
    @GetMapping("/v1/ping")
    String ping();
}

@Configuration
@LoadBalancerClient(name = "orders", configuration = OrdersLbConfig.class)
class InternalFeignConfig {
    @Bean RequestInterceptor correlation() { return t -> t.header("X-Request-Id", java.util.UUID.randomUUID().toString()); }
}

@Configuration
class OrdersLbConfig {
    @Bean ReactorLoadBalancer<ServiceInstance> rr(Environment env, LoadBalancerClientFactory f) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
    @Bean ServiceInstanceListSupplier supplier(org.springframework.context.ConfigurableApplicationContext ctx) {
        return ServiceInstanceListSupplier.builder().withDiscoveryClient().withHealthChecks().build(ctx);
    }
}

@Configuration
class ExternalFeignConfig {
    @Bean BasicAuthRequestInterceptor basic() {
        return new BasicAuthRequestInterceptor(System.getenv("PAY_USER"), System.getenv("PAY_PASS"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.split

import feign.RequestInterceptor
import feign.auth.BasicAuthRequestInterceptor
import org.springframework.cloud.client.ServiceInstance
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer
import org.springframework.cloud.loadbalancer.core.RoundRobinLoadBalancer
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.env.Environment
import org.springframework.web.bind.annotation.GetMapping
import java.util.UUID
import org.springframework.context.ConfigurableApplicationContext

@FeignClient(name = "orders", configuration = [InternalFeignConfig::class])
interface OrdersClient {
    @GetMapping("/api/v1/orders/me")
    fun myOrders(): String
}

@FeignClient(name = "payments-ext", url = "\${clients.payments.url}", configuration = [ExternalFeignConfig::class])
interface PaymentsClient {
    @GetMapping("/v1/ping")
    fun ping(): String
}

@Configuration
@LoadBalancerClient(name = "orders", configuration = [OrdersLbConfig::class])
class InternalFeignConfig {
    @Bean
    fun correlation(): RequestInterceptor = RequestInterceptor { t ->
        t.header("X-Request-Id", UUID.randomUUID().toString())
    }
}

@Configuration
class OrdersLbConfig {
    @Bean
    fun rr(env: Environment, f: LoadBalancerClientFactory): ReactorLoadBalancer<ServiceInstance> {
        val name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME)!!
        return RoundRobinLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier::class.java), name)
    }
    @Bean
    fun supplier(ctx: ConfigurableApplicationContext): ServiceInstanceListSupplier =
        ServiceInstanceListSupplier.builder().withDiscoveryClient().withHealthChecks().build(ctx)
}

@Configuration
class ExternalFeignConfig {
    @Bean
    fun basic(): BasicAuthRequestInterceptor =
        BasicAuthRequestInterceptor(System.getenv("PAY_USER"), System.getenv("PAY_PASS"))
}
```

# 9. Тестирование и контракты

**Gradle (Groovy DSL)**

```groovy
dependencies {
    implementation platform("org.springframework.cloud:spring-cloud-dependencies:2024.0.0")
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'com.github.tomakehurst:wiremock-jre8:2.35.1'
    testImplementation 'com.squareup.okhttp3:mockwebserver:4.12.0'
    testImplementation 'org.springframework.cloud:spring-cloud-starter-contract-verifier'
    testImplementation 'au.com.dius.pact.consumer:junit5:4.6.9'
    testImplementation 'org.skyscreamer:jsonassert:1.5.1'
    testImplementation 'org.testcontainers:junit-jupiter:1.20.2'
    testImplementation 'org.testcontainers:testcontainers:1.20.2'
}
test { useJUnitPlatform() }
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    implementation(platform("org.springframework.cloud:spring-cloud-dependencies:2024.0.0"))
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("com.github.tomakehurst:wiremock-jre8:2.35.1")
    testImplementation("com.squareup.okhttp3:mockwebserver:4.12.0")
    testImplementation("org.springframework.cloud:spring-cloud-starter-contract-verifier")
    testImplementation("au.com.dius.pact.consumer:junit5:4.6.9")
    testImplementation("org.skyscreamer:jsonassert:1.5.1")
    testImplementation("org.testcontainers:junit-jupiter:1.20.2")
    testImplementation("org.testcontainers:testcontainers:1.20.2")
}
tasks.test { useJUnitPlatform() }
```

---

## Локальные тесты с WireMock/MockWebServer: сценарии 2xx/4xx/5xx/тайм-ауты

Локальное тестирование клиентов — самый дешёвый и быстрый способ поймать несоответствие контракту: мы поднимаем stub-сервер у себя в процессе, на лету описываем ответы и наблюдаем, что делает клиент при успехах и сбоях. Такой подход не зависит от стендов, сетей и реестров, что резко сокращает флейки и ускоряет обратную связь. Ключевая цель — не только «позелинить» 200 OK, но и системно проверить деградации: пустое тело, неверный `Content-Type`, задержки, разрывы сокета, 4xx/5xx.

WireMock удобен, когда нас интересует корректность HTTP-семантики и заголовков. Он декларативно описывает стабы, валидирует, что пришло в запросе (path, query, headers), и отдаёт то, что мы задумали. В типичных тестах мы держим тайм-ауты маленькими (сотни миллисекунд), чтобы негативные сценарии «падали» быстро и предсказуемо. Важный нюанс: в `test`-профиле специально занижаем `readTimeout`, чтобы проверка тайм-аутов была детерминированной.

MockWebServer из OkHttp даёт тонкий контроль над байтовым уровнем и временем: можно оборвать соединение после заголовков, дробить тело на чанки, искусственно задерживать отдельные этапы. Это позволяет ловить ошибки в обработке потока и в ретраях клиента. Если транспорт у Feign — OkHttp, MockWebServer идеально показывает «нативное» поведение сокета и тайм-аутов.

Минимальный набор сценариев, который должен быть в каждом проекте: успешный 2xx с корректным JSON; бизнес-ошибка 4xx с телом Problem+JSON и правильным маппингом через `ErrorDecoder`; техническая 5xx с повтором (если разрешено политикой); тайм-аут чтения; разрыв во время тела; неожиданный `Content-Type` при JSON-теле; пустое тело на 204/404. Эти кейсы покрывают большую часть производственных инцидентов, связанных с интеграциями.

Особое внимание стоит уделить заголовкам и квери-параметрам. Многие баги связаны с пропагонацией `Authorization`, корреляционного `X-Request-Id` и контрактных флагов; правильный тест явно проверяет их присутствие и значения. Если у вас есть перехватчики (`RequestInterceptor`), добавьте отдельные проверки, что они не дублируют заголовки при ретраях и не «теряют» их на исключениях.

Разделяйте позитивные и негативные сценарии по разным тестам — так причина падения будет сразу очевидна, а логи стаба проще анализировать. Дополнительно сохраняйте на CI логи WireMock/MockWebServer как артефакты — это экономит часы при разборе нестабильностей. При этом не злоупотребляйте большими задержками: тесты должны оставаться быстрыми и локальными.

Стоит выработать соглашение по базовым адресам: в тестах мы обычно «перебиваем» `clients.*.url` через `@DynamicPropertySource`, направляя клиента на `wm.baseUrl()` или `mws.url("/")`. Это позволяет не менять сам интерфейс `@FeignClient` и его конфигурации — только свойства окружения. Такой приём полезен и для интеграционных тестов, и для локального запуска IDE.

Следите, чтобы тайм-ауты бизнес-уровня не маскировали тайм-ауты HTTP-клиента. Если у вас в фасаде есть «дедлайн» на операцию, его тестируют отдельно; в тестах Feign целимся именно в сетевые окна (`connectTimeout`, `readTimeout`), иначе диагностика причин отказа станет мутной. Отдельно проверяйте, что на повторяемых методах повтор действительно происходит, а на небезопасных — нет.

Наконец, проверьте поведение на «грязных» типах контента и на пустых телах. Десериализация по умолчанию в Jackson не любит `text/plain` с JSON-строкой внутри, а HTTP-клиенты иногда возвращают пустые тела с кодами, где вы их не ждёте. Грамотный `ErrorDecoder` и аккуратные `Decoder`/`Encoder` избавят вас от внезапных `MismatchedInputException` в проде.

**Java — WireMock: 200/422/тайм-аут**

```java
package com.example.feign.local;

import com.github.tomakehurst.wiremock.WireMockServer;
import org.junit.jupiter.api.*;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

class LocalWireMockTest {
    static WireMockServer wm = new WireMockServer(0);

    @BeforeAll static void up() { wm.start(); }
    @AfterAll static void down() { wm.stop(); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("clients.catalog.url", wm::baseUrl);
    }

    @Test void ok200() {
        wm.stubFor(get("/api/v1/items").willReturn(okJson("[\"A\",\"B\"]")));
        // вызываем Feign-клиент и проверяем тело (опущено ради краткости)
    }

    @Test void business422() {
        wm.stubFor(get("/api/v1/items").willReturn(aResponse()
            .withStatus(422).withHeader("Content-Type","application/json")
            .withBody("{\"type\":\"https://example/validation\",\"title\":\"Unprocessable\",\"detail\":\"bad\"}")));
        // assert: ErrorDecoder -> ValidationException
    }

    @Test void readTimeout() {
        wm.stubFor(get("/api/v1/items").willReturn(aResponse().withFixedDelay(1500).withBody("[]")));
        // при readTimeout=500ms ожидаем тайм-аут/RetryableException
    }
}
```

**Kotlin — MockWebServer: разрыв соединения**

```kotlin
package com.example.feign.local

import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import okhttp3.mockwebserver.SocketPolicy
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test

class LocalMwsTest {
    companion object {
        private val mws = MockWebServer()
        @JvmStatic @BeforeAll fun up() { mws.start() }
        @JvmStatic @AfterAll fun down() { mws.shutdown() }
    }

    @Test
    fun disconnectDuringBody() {
        mws.enqueue(MockResponse().setResponseCode(200).setBody("[]")
            .setSocketPolicy(SocketPolicy.DISCONNECT_DURING_RESPONSE_BODY))
        mws.enqueue(MockResponse().setResponseCode(200).setBody("[]"))
        // вызов клиента с retry; 1-я попытка обрывается, 2-я успешна
    }
}
```

---

## Срезы: загрузка контекста с конкретным `@FeignClient`, моки зависимостей

Срезы нужны, когда полный `@SpringBootTest` слишком тяжёлый: мы поднимаем только то, что критично для клиента, и ничего лишнего. Это даёт быстрые и стабильные тесты, где ясно видно, что проверяется: конфигурация Feign, перехватчики, декодеры, тайм-ауты и типы исключений. Результат — минуты экономии на каждом прогоне и меньше ложных падений из-за посторонних автоконфигураций.

Минимальный срез обычно включает автоконфигурацию OpenFeign и сам интерфейс клиента. Базовый URL мы переопределяем динамически, указывая на локальный стаб, а все сторонние зависимости клиента, вроде `ErrorDecoder` или `RequestInterceptor`, подключаем явными бинами. Так тест становится самодостаточным и легко читается разработчиком, который впервые видит проект.

Если в проекте есть OAuth2-интеграция и перехватчик достаёт токены из `OAuth2AuthorizedClientManager`, в срез подставляем тестовый бин, возвращающий фиктивный `OAuth2AccessToken`. Это избавляет от необходимости тянуть в тестах реальные настройки security, а поведение клиента при этом остаётся идентичным продовому: заголовок будет установлен, а downstream-стаб сможет это проверить.

Через `@DynamicPropertySource` удобно не только URL ставить, но и тайм-ауты: в тестовом профиле задаём короткие окна `connectTimeout/readTimeout`, чтобы негативные сценарии проходили быстро. Параллельно отключаем ненужные автоконфиги (например, LoadBalancer для статического URL) — иначе контекст будет искать discovery и штормить логи.

Срез — отличное место, чтобы протестировать `ErrorDecoder` без тяжёлых сценариев. Мы программируем стабы с 4xx/5xx и проверяем, что правильные доменные исключения выбрасываются и содержат всю нужную диагностику (код, detail, correlation-id). Это избавляет от неожиданностей, когда потребитель API вдруг начинает по-другому реагировать на ту же ошибку.

Не забывайте про перехватчики корреляции и трассировки: именно в срезах легко проверить, что `X-Request-Id`/`traceparent` действительно проставляются, а не теряются на ретраях. Эти мелочи регулярно всплывают в RCA, потому что недостаточно покрыты тестами.

Отдельный плюс срезов — явность зависимостей: тестовый класс показывает весь «композиционный корень» клиента. Когда новому разработчику нужно понять, что и где настраивается, он открывает тест и видит полный путь — без копания в десятках конфигов.

Помните, что срез — это не альтернатива локальным stub-тестам, а следующий слой. Сначала убедитесь, что сам HTTP-уровень работает как задумано, затем покажите, что wiring внутри Spring Boot корректен. Такая последовательность минимизирует стоимость диагностики.

Важно закрыть и сценарий с пустыми телами/необычными типами медиа. В срезе это проверяется особенно удобно: мы можем перехватить вызов декодера и убедиться, что ошибок по сериализации нет, а поведение соответствует ожиданиям для `204 No Content` и `404 Not Found`.

И наконец, документируйте срезы: небольшая секция в README с командами запуска и назначением тестов поможет команде использовать их как «живую документацию» клиента.

**Java — «минимальный контекст» для Feign**

```java
package com.example.feign.slice;

import com.github.tomakehurst.wiremock.WireMockServer;
import org.junit.jupiter.api.*;
import org.springframework.boot.autoconfigure.ImportAutoConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.cloud.openfeign.FeignAutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

@EnableFeignClients(clients = CatalogClient.class)
@ImportAutoConfiguration(FeignAutoConfiguration.class)
class CatalogClientSliceTest {
    static WireMockServer wm = new WireMockServer(0);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        wm.start();
        r.add("clients.catalog.url", wm::baseUrl);
    }

    @TestConfiguration
    static class TestCfg {
        @Bean CatalogFeignConfig config() { return new CatalogFeignConfig(); }
    }

    @AfterAll static void down() { wm.stop(); }
    @Test void loads() { /* проверяем, что бин клиента подхватился */ }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.slice

import com.github.tomakehurst.wiremock.WireMockServer
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Test
import org.springframework.boot.autoconfigure.ImportAutoConfiguration
import org.springframework.boot.test.context.TestConfiguration
import org.springframework.cloud.openfeign.EnableFeignClients
import org.springframework.cloud.openfeign.FeignAutoConfiguration
import org.springframework.context.annotation.Bean
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource

@EnableFeignClients(clients = [CatalogClient::class])
@ImportAutoConfiguration(FeignAutoConfiguration::class)
class CatalogClientSliceTest {
    companion object {
        private val wm = WireMockServer(0)
        @JvmStatic @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) { wm.start(); r.add("clients.catalog.url") { wm.baseUrl() } }
        @JvmStatic @AfterAll fun down() { wm.stop() }
    }

    @TestConfiguration
    class TestCfg {
        @Bean fun config() = CatalogFeignConfig()
    }

    @Test fun loads() { /* бин клиента на месте */ }
}
```

---

## Контрактные тесты (Spring Cloud Contract/Pact) для провайдеров/консьюмеров

Контракты «застёгивают» взаимодействие поставщика и потребителя: любой breaking-change всплывает на ревью кода, а не в проде. В экосистеме Spring есть два проверенных пути. Spring Cloud Contract ориентирован на провайдера: вы описываете контракты (Groovy/YAML), из них генерируются юнит-тесты для сервиса-поставщика и пакуются WireMock-ста бы как артефакт, который подтягивают потребители. Pact ориентирован на потребителя: он пишет ожидаемое взаимодействие, генерирует pact-файл, а провайдер верифицирует его у себя в CI.

Выбор между подходами зависит от процесса. Если команда провайдера централизована и задаёт правила, удобнее SCC: он естественно встраивается в билд и публикует стабы в артефактори. Если потребителей много и они автономны, предпочтителен Pact с брокером: потребители быстро фиксируют ожидания, а провайдер видит, кого затронут изменения, и может катить миграции мягко.

Важная мысль: контракт проверяет протокол (путь, метод, статус, схему тела), а не бизнес-логику. Не стоит пытаться «вшить» инварианты предметной области — это сделает контракт хрупким и привязанным к деталям реализации. Бизнес-правила покрывайте отдельными модульными/интеграционными тестами. Контракт же должен быть стабильной основой для совместимости.

Контрактные тесты ценны не только позитивными сценариями, но и негативными. Явный контракт на `422 Unprocessable Entity` со списком ошибок валидации поможет всем потребителям корректно обрабатывать отказ, а провайдеру — не «сломать» формат случайно при рефакторинге. Аналогично полезны контракты на `409 Conflict` и `401/403` с минимальным телом.

Храните контракты версионированно и не публикуйте «ломающие» без договорённости. В мире Pact это решается тегами/ветками в брокере, в SCC — версиями артефактов стаба, которые потребители подтягивают в своих тестах. Это делает миграции контролируемыми: сначала двоичная совместимость, затем удаление устаревшего.

Не включайте в контракты реальные секреты: токены должны быть фиктивными, заголовки авторизации проверяются по форме, а не по значениям. Это снижает риск утечки и упрощает распространение контрактов между командами и окружениями.

Интеграция в CI стандартна: на стороне провайдера — генерация и прогон тестов от SCC, публикация стабов/пактов; на стороне потребителя — прогон своих тестов с подтянутыми стабами или верификация pact-файлов. Провал любого шага — блокер для слияния/деплоя. Это и есть настоящая «сеть безопасности».

Старайтесь держать контракты компактными и читаемыми. Большие payload’ы усложняют ревью и повышают стоимость изменений. Там, где нужно версионирование схемы, используйте явные типы и комментируйте смысл полей, чтобы новым участникам команды было легче ориентироваться.

И, конечно, дополните контракты хорошей документацией: краткое описание кейса, ссылка на бизнес-правило, ожидания по заголовкам. Контракт — это ещё и «живая документация» API, которой реально пользуются разработчики.

**Java — Pact consumer-тест**

```java
package com.example.contract.pact;

import au.com.dius.pact.consumer.MockServer;
import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.*;
import au.com.dius.pact.core.model.RequestResponsePact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "catalog", port = "0")
class CatalogPactTest {
    @Pact(consumer = "bff")
    RequestResponsePact pact(PactDslWithProvider p) {
        return p.given("items exist")
            .uponReceiving("GET /api/v1/items")
            .path("/api/v1/items").method("GET")
            .willRespondWith().status(200).body("[\"A\",\"B\"]")
            .toPact();
    }

    @Test
    void run(MockServer ms) {
        assertThat(ms.getUrl()).isNotBlank();
        // здесь можно создать Feign-клиент с url=ms.getUrl() и проверить вызов
    }
}
```

**Kotlin — Pact consumer-тест**

```kotlin
package com.example.contract.pact

import au.com.dius.pact.consumer.MockServer
import au.com.dius.pact.consumer.dsl.PactDslWithProvider
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt
import au.com.dius.pact.consumer.junit5.PactTestFor
import au.com.dius.pact.core.model.RequestResponsePact
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith

@ExtendWith(PactConsumerTestExt::class)
@PactTestFor(providerName = "catalog", port = "0")
class CatalogPactTest {
    @au.com.dius.pact.consumer.junit5.Pact(consumer = "bff")
    fun pact(p: PactDslWithProvider): RequestResponsePact =
        p.given("items exist")
            .uponReceiving("GET /api/v1/items")
            .path("/api/v1/items").method("GET")
            .willRespondWith().status(200).body("[\"A\",\"B\"]")
            .toPact()

    @Test
    fun run(ms: MockServer) {
        assertThat(ms.url).isNotBlank
    }
}
```

---

## Тестирование безопасности: стабы IdP, подмена токена в интерсепторе

Проверка безопасности клиентов — это про два аспекта: правильно ли мы формируем/передаём аутентификацию и как реагируем на ответы `401/403`. Нельзя тянуть реальный IdP в юнит-тесты, поэтому мы либо подменяем токены в перехватчике, либо даём тестовую реализацию менеджера авторизованных клиентов. Это гарантирует, что в продуктовой конфигурации код не «расползётся» и не начнёт пропускать заголовки.

Для сценария Client Credentials подменяем `OAuth2AuthorizedClientManager` тестовым бином, возвращающим `OAuth2AccessToken` с коротким сроком. Перехватчик берёт этот токен и добавляет `Authorization: Bearer ...` в каждый запрос. WireMock при этом проверяет наличие заголовка, не требуя реальной подписи. Такой тест надёжен и прост: мы уверены, что интеграция с security не «сломается» после рефакторинга клиента.

Если нужен token relay (проброс пользовательского контекста), используем `SecurityContextHolder` и загружаем туда `JwtAuthenticationToken` в тесте. Интерсептор читает токен и добавляет его в исходящий запрос. Это критично для BFF-паттернов, где downstream-сервис авторизует пользователя по скопам из JWT, а не по сервисному токену. Проверяем и позитив (валидный токен), и негатив (истёкший или с неверным скоупом).

Особенно важно убедиться, что при `401 Unauthorized` клиент не ретраит запросы бесконечно, и что `403 Forbidden` маппится в доменное исключение без дополнительных попыток. Эти категории ошибок часто путают с техническими и оборачивают ретраями; в результате нагрузка на IdP/ресурс-сервер растёт лавинообразно. В юнитах эта грань должна быть зафиксирована.

Логирование — отдельная зона риска. При отладке легко включить вывод HEADERS/BODY и засветить секреты. В тестах мы включаем детальный лог на очень коротких сценариях и проверяем, что `Authorization` маскируется кастомным логгером (или вовсе не логируется), а `X-Request-Id`/trace-заголовки присутствуют. Такой тест — маленький, но выручает в проде.

В сценариях Basic/API-Key действуем аналогично: перехватчик добавляет заголовок, а стаб проверяет его наличие и формат. Важно проверять отсутствие дублирования заголовков на ретраях: некоторые клиенты могут «накладывать» одно и то же поле повторно, что вызывает загадочные 401 у провайдера.

mTLS на юнит-уровне обычно не тестируют — это зона интеграции; но можно завести тест, проверяющий, что SSL-контекст инициализируется с тестовыми кейсторами, чтобы не ходить в систему. В e2e тестах вопрос закрывается контейнером с самоподписанными сертификатами и Nginx.

Не забывайте про `audience` и `scope` в негативных сценариях. 403 без тела — распространённый ответ, и клиент должен уметь превратить его в понятное доменное исключение, которое увидит бизнес-слой. Для этого ErrorDecoder обязан читать заголовки/минимальные поля и добавлять их в сообщение об ошибке.

И наконец, держите тестовые токены короткими и заведомо «нереальными», чтобы их нельзя было перепутать с настоящими. Это простое правило спасает от утечек и недоразумений на ревью.

**Java — тест перехватчика с фиктивным Bearer**

```java
package com.example.security.feign;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class OAuth2InterceptorTest {
    @Test
    void addsBearerHeader() {
        RequestInterceptor interceptor = (RequestTemplate t) ->
                t.header("Authorization", "Bearer test-token");
        RequestTemplate t = new RequestTemplate();
        interceptor.apply(t);
        assertThat(t.headers().get("Authorization")).containsExactly("Bearer test-token");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.feign

import feign.RequestInterceptor
import feign.RequestTemplate
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class OAuth2InterceptorTest {
    @Test
    fun addsBearerHeader() {
        val i: RequestInterceptor = RequestInterceptor { t: RequestTemplate ->
            t.header("Authorization", "Bearer test-token")
        }
        val t = RequestTemplate()
        i.apply(t)
        assertThat(t.headers()["Authorization"]).containsExactly("Bearer test-token")
    }
}
```

---

## Интеграционные тесты: Testcontainers для провайдера (если это ваш сервис)

Когда и клиент, и сервер — ваши артефакты, e2e-проверка через контейнер — лучший способ поймать конфигурационные ошибки: порты, пути, CORS, security-политику. Мы запускаем реальный образ провайдера (или «тонкий» контейнер с MockServer), ждём готовность по `/actuator/health` и направляем Feign на опубликованный `host:port`. Это ближе всего к продовой траектории запроса, а значит — меньше сюрпризов после деплоя.

Правильно организованный интеграционный тест изолирован: он поднимает всё необходимое сам и не зависит от внешних стендов. Тайм-ауты в нём умеренные (секунды, не минуты), а падение «умирает» быстро и оставляет подробные логи контейнера в артефактах CI. Это ключ к быстрой диагностике: когда сервис не поднялся, видно почему — миграции не применились, порт занят, секрет не найден.

Если провайдер зависит от БД/кэша, поднимите минимальные окружения рядом (PostgreSQL, Redis) и прогоните миграции Flyway/Liquibase. Так вы поймаете несовместимость схем на ранней стадии. При mTLS готовьте самоподписанные сертификаты в pre-step, монтируйте их в контейнер и указывайте через переменные окружения. Не кладите рабочие ключи в тесты — это антипрактика.

В большинстве случаев достаточно одного «счастливого» и одного «ошибочного» сценария: успешный чтение списка, например, и 422 при неверных параметрах. Цель — проверить wiring, сериализацию/десериализацию, маппинг ошибок и безопасность «по-настоящему», а не переписывать все юнит-кейсы ещё раз.

Не гоните такие тесты массово параллельно — контейнеры прожорливы, и CI-агент легко уйдёт в swap. Лучше держать их в отдельном job’е или как «медленный» набор, запускаемый по nightly или перед релизом. Это дисциплинирует и держит пайплайн быстрым.

Кэш образов на CI — must-have. Подтягивание базовых образов в каждом прогоне может добавить минуты. С локальным кэшем и фиксированными тегами вы сведёте накладные расходы к минимуму и избежите странных регрессов из-за внезапно изменившихся `latest`.

В логах полезно иметь корреляционные id, чтобы склеивать клиентские и серверные трассы. Добавьте в перехватчик Feign `X-Request-Id`, а в провайдер — логирование этого заголовка; разбор падений станет на порядок быстрее. А ещё включите в ответ сервера Problem-детали, чтобы клиентский тест мог ассертом вытащить причину.

И наконец, держите эти тесты рядом с кодом клиента, а не только провайдера. Падение клиента на изменении контракта должно сработать на PR провайдера — для этого используйте артефакты/регистри с образами и «склейку» пайплайнов.

**Java — GenericContainer с ожиданием health**

```java
package com.example.feign.it;

import org.junit.jupiter.api.*;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.wait.strategy.HttpWaitStrategy;
import org.testcontainers.utility.DockerImageName;

import static org.assertj.core.api.Assertions.assertThat;

class ProviderIT {
    static GenericContainer<?> provider = new GenericContainer<>(DockerImageName.parse("registry.local/catalog:1.2.3"))
            .withExposedPorts(8080)
            .waitingFor(new HttpWaitStrategy().forPath("/actuator/health").forStatusCode(200));

    @BeforeAll static void up() { provider.start(); }
    @AfterAll static void down() { provider.stop(); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("clients.catalog.url", () -> "http://" + provider.getHost() + ":" + provider.getMappedPort(8080));
    }

    @Test void containerRunning() {
        assertThat(provider.isRunning()).isTrue();
        // здесь можно вызвать Feign-клиент и проверить реальный ответ
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.it

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.GenericContainer
import org.testcontainers.containers.wait.strategy.HttpWaitStrategy
import org.testcontainers.utility.DockerImageName

class ProviderIT {
    companion object {
        private val provider = GenericContainer(DockerImageName.parse("registry.local/catalog:1.2.3"))
            .withExposedPorts(8080)
            .waitingFor(HttpWaitStrategy().forPath("/actuator/health").forStatusCode(200))

        @JvmStatic @BeforeAll fun up() { provider.start() }
        @JvmStatic @AfterAll fun down() { provider.stop() }

        @JvmStatic @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("clients.catalog.url") { "http://${provider.host}:${provider.getMappedPort(8080)}" }
        }
    }

    @Test
    fun containerRunning() {
        assertThat(provider.isRunning).isTrue
    }
}
```

---

## Golden-files/snapshot-подход для стабилизации полезных нагрузок

Когда полезные нагрузки «толстые» и часто меняются, обычные ассерты быстро превращаются в хрупкую лапшу. Подход snapshot («золотые файлы») фиксирует канонический JSON/DTO в ресурсах, а тесты сравнивают фактический результат с эталоном. Любое расхождение — видимый дифф, который осознанно принимается или отклоняется на ревью. Это дисциплинирует эволюцию схем и даёт уверенность, что формат не «поплыл» незаметно.

Чтобы снапшоты были полезны, сериализация должна быть детерминированной: фиксированный порядок полей, одни и те же правила форматирования дат/чисел. Для Jackson включают упорядочивание ключей и одинаковые модули для времени (`JavaTimeModule`). Тогда дифф в MR действительно отражает смысловую разницу, а не случайную перестановку.

В «золотых» файлах не должно быть нестабильных полей: времён, случайных id, корреляционных заголовков. Их либо маскируют перед сериализацией, либо исключают из сравнения с помощью JSONPath. Иначе тесты будут ложнопозитивными и их начнут отключать — то есть терять пользу. Удобно завести маленький утиль, который «нормализует» DTO перед записью в эталон.

Снапшоты особенно хороши для ошибок: Problem+JSON на `422/409` быстро «плывёт» при рефакторинге, и потребители потом ломаются неожиданно. Фиксируя эталонные ошибки, мы держим формат под контролем и не даём «переводить» технические детали наружу. Параллельно стоит документировать поля проблемы в README — снапшот служит и проверкой, и примером для документации.

Процесс обновления снапшотов должен быть осознанным действием разработчика: флаг `-DupdateSnapshots=true` или отдельная задача Gradle, которая перезаписывает эталон. На CI перезапись запрещена — иначе тест перестанет выполнять свою охранную функцию. Такой «ритуал» прививает правильную культуру изменений.

Размер эталона тоже важен: гигантские JSON трудны для ревью. Имеет смысл делить по смысловым блокам и сравнивать только то, что критично для контракта. Например, для заказа можно держать отдельные снапшоты для «шапки» и «позиции», а не один монолит.

Снапшоты не заменяют контрактов: первый — инструмент на стороне клиента, второй — договор между сервисами. Комбинация даёт максимальную страховку: контракт ломается — PR не проходит; снапшот отличается — команда понимает, как именно изменился формат и стоит ли это принимать.

При снятии снапшота в тесте задействуйте те же сериализаторы, что и в проде. Разница в конфигурации `ObjectMapper` между `main` и `test` бесполезна: тест будет зелёный, а потребители получат другой JSON. Лучший способ — вынести mapper в отдельный модуль и импортировать его в тесты.

И наконец, снапшоты удобны для отката: после возврата на старую версию вы сразу видите, что фактический JSON снова совпал с золотым. Это экономит время RCA и снижает риск повторных ошибок.

**Java — сравнение JSON с «золотым»**

```java
package com.example.snapshot;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import org.junit.jupiter.api.Test;

import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class SnapshotTest {
    ObjectMapper om = new ObjectMapper().enable(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS);

    @Test
    void payloadMatchesGolden() throws Exception {
        String actual = om.writeValueAsString(new Payload("A", 42));
        String golden = Files.readString(Path.of("src/test/resources/golden/payload.json"));
        assertThat(actual).isEqualToIgnoringWhitespace(golden);
    }

    record Payload(String name, int value) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.snapshot

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.SerializationFeature
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.nio.file.Files
import java.nio.file.Path

class SnapshotTest {
    private val om = ObjectMapper().enable(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS)

    @Test
    fun payloadMatchesGolden() {
        val actual = om.writeValueAsString(Payload("A", 42))
        val golden = Files.readString(Path.of("src/test/resources/golden/payload.json"))
        assertThat(actual).isEqualToIgnoringWhitespace(golden)
    }

    data class Payload(val name: String, val value: Int)
}
```

# 10. Наблюдаемость, производительность и эксплуатация

## Логи: `feign.Logger` уровни, корреляция запрос-ответ, маскирование секретов

Грамотное логирование Feign-вызовов начинается с понимания уровней `feign.Logger.Level`. Уровень `NONE` подходит для «шумных» прод-сервисов по умолчанию, когда метрики и трейсинг уже дают достаточную картину. `BASIC` фиксирует метод, URL и статус — это минимально полезный уровень для диагностики инцидентов. `HEADERS` добавляет заголовки, а `FULL` выводит ещё и тела запросов/ответов. На продакшне «полные» логи включают только адресно и на короткое окно, чтобы не утонуть в объёме и не утечь секретами. В тестах и разработке можно позволить себе `FULL`, но при этом сразу включить маскирование приватных полей.

Корреляция — обязательна. Каждый исходящий вызов должен нести `X-Request-Id` и/или стандартный `traceparent`, чтобы бэкенды и аналитика могли склеить цепочку. Если используете Micrometer Tracing / OpenTelemetry — propagation сделают автоконфигурации, но свой корреляционный ID полезно логировать отдельно: он стабилен для человека и читается в тикетах. Перехватчик Feign — удобное место, чтобы проставить недостающие заголовки и одновременно положить их в Mapped Diagnostic Context (MDC).

Маскирование секретов — не «приятная опция», а требование безопасности. Любой логгер, который пишет заголовки, обязан редактировать `Authorization`, `Cookie`, `Set-Cookie`, API-ключи и токены. Простой регэксп, заменяющий значение на `***`, спасает от бессмысленных расследований утечек. Для тела запросов применяют белые списки — логируем только ожидаемые поля (например, публичные идентификаторы), а всё остальное редактируем.

Логи должны быть структурированными: JSON-строки с ключами `direction`, `service`, `method`, `url`, `status`, `duration_ms`, `request_id`. Это экономит часы при поиске по Kibana/Logbook/Grafana Loki. Текстовые «простыни» пригодны для отладки локально, но в проде побеждает машина, которой нужны ключ-значение.

Сторона сервиса обязана помнить о PII. Если Feign отправляет e-mail или телефон, в логах храните только хэш/маску. Будьте предсказуемы: одинаковое поле — одинаковая маска. Это упростит дашборды и отчёты, где нежелательно светить реальные данные.

Объём логов — это деньги. Если сервис делает десятки тысяч вызовов в минуту, любой переход с `BASIC` на `FULL` может утроить расходы на хранение. «Тумблер» через переменные окружения и Feature Flags позволит быстро включить подробности на нужном клиенте и так же быстро выключить.

Корректный тайминг — половина успеха. В логах фиксируйте время DNS-резолва, установки соединения, TLS-рукопожатия и чтения тела. Это можно сделать через расширенный логгер или за счёт метрик (см. ниже). Во многих инцидентах «медленный» вызов — это не сервер, а TLS или сеть.

Логи ошибок должны отличаться от информационных. Присваивайте им отдельный логгер/категорию (`com.example.feign.error`) и добавляйте поля `error_category` (network/business/timeout) и `downstream` (имя клиента). Это сильно ускорит triage, когда кнопка «красная» и времени мало.

Старайтесь не логировать тела двоичных ответов: PDF, ZIP, изображения мгновенно забьют диски и повалят логопровод. Если такие ответы критичны для расследований, логируйте только хэши и длину. А для JSON используйте лимиты на размер: усекли — пометили флагом.

И наконец, логирование — это часть контракта команды. Задокументируйте, где и в каком формате искать исходящие запросы, какие поля обязательны, что маскируется. Новый инженер должен без вопросов найти «след» любого вызова из тикета.

**Java — кастомный Logger с маскированием и корреляцией**

```java
package com.example.feign.logging;

import feign.Logger;
import feign.Request;
import feign.Response;
import org.slf4j.MDC;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Collection;
import java.util.Map;

public class RedactingFeignLogger extends Logger {

    @Override
    protected void log(String configKey, String format, Object... args) {
        // Делегируйте в SLF4J: один канал для запросов
        org.slf4j.LoggerFactory.getLogger("feign.http").info(String.format(format, args));
    }

    @Override
    protected void logRequest(String configKey, Level logLevel, Request request) {
        String requestId = MDC.get("requestId");
        String url = request.url();
        log(configKey, "--> %s %s (requestId=%s)", request.httpMethod().name(), url, requestId);
        if (logLevel.ordinal() >= Level.HEADERS.ordinal()) {
            for (Map.Entry<String, Collection<String>> e : request.headers().entrySet()) {
                String key = e.getKey();
                for (String v : e.getValue()) {
                    log(configKey, "%s: %s", key, redact(key, v));
                }
            }
        }
        if (request.body() != null && logLevel.ordinal() >= Level.FULL.ordinal()) {
            String body = new String(request.body(), StandardCharsets.UTF_8);
            log(configKey, "Body: %s", redactBody(body));
        }
        log(configKey, "--> END %s", request.httpMethod());
    }

    @Override
    protected Response logAndRebufferResponse(String configKey, Level logLevel, Response response, long elapsedTime) throws IOException {
        String requestId = MDC.get("requestId");
        int status = response.status();
        log(configKey, "<-- %d (%d ms) (requestId=%s)", status, elapsedTime, requestId);
        if (logLevel.ordinal() >= Level.HEADERS.ordinal()) {
            for (Map.Entry<String, Collection<String>> e : response.headers().entrySet()) {
                for (String v : e.getValue()) {
                    log(configKey, "%s: %s", e.getKey(), redact(e.getKey(), v));
                }
            }
        }
        if (response.body() != null && logLevel.ordinal() >= Level.FULL.ordinal()) {
            byte[] bodyData = response.body().asInputStream().readAllBytes();
            String body = new String(bodyData, StandardCharsets.UTF_8);
            log(configKey, "Body: %s", redactBody(body));
            return response.toBuilder().body(bodyData).build();
        }
        return response;
    }

    private String redact(String key, String value) {
        String k = key.toLowerCase();
        if (k.equals("authorization") || k.equals("cookie") || k.equals("set-cookie") || k.contains("api-key")) {
            return "***";
        }
        return value;
    }

    private String redactBody(String body) {
        // Упростим: маскируем очевидные токены
        return body.replaceAll("(?i)\"access_token\"\\s*:\\s*\"[^\"]+\"", "\"access_token\":\"***\"");
    }
}
```

**Kotlin — перехватчик корреляции + настройка уровня логирования**

```kotlin
package com.example.feign.logging

import feign.RequestInterceptor
import feign.RequestTemplate
import org.slf4j.MDC
import java.util.UUID

class CorrelationInterceptor : RequestInterceptor {
    override fun apply(template: RequestTemplate) {
        val rid = MDC.get("requestId") ?: UUID.randomUUID().toString().also { MDC.put("requestId", it) }
        template.header("X-Request-Id", rid)
    }
}
```

**Gradle (Groovy/Kotlin) — подключаем логгер на клиента**

```groovy
dependencies {
    implementation 'org.slf4j:slf4j-api'
}
```

```kotlin
dependencies {
    implementation("org.slf4j:slf4j-api")
}
```

**application.yml — уровни логгера на окружениях**

```yaml
feign:
  client:
    config:
      default:
        loggerLevel: BASIC  # prod по умолчанию
logging:
  level:
    feign.http: INFO
    com.example.feign.logging.RedactingFeignLogger: INFO
```

---

## Метрики: Micrometer таймеры/счётчики per-client/method/status, export в Prometheus

Метрики дают агрегированную, дешёвую по объёму и очень информативную картину: где латентность, где ошибки, какой процент ретраев и на каких методах. В мире Spring Boot 3 наблюдаемость стандартно строится вокруг Micrometer. В связке с OpenFeign есть два пути: модуль `feign-micrometer` добавляет наблюдения непосредственно в Feign, а Spring Cloud OpenFeign 4+ прокидывает `ObservationRegistry`, превращая каждый вызов в Observation с таймером и тегами клиента/метода/статуса.

Базовая идея проста: на каждый исходящий вызов пишется таймер с лейблами `client`, `method`, `uri-template` (без параметров), `status`. Такие метрики позволяют строить p95/p99 и quickly spot проблемные downstream. Счётчики ошибок по категориям (`network`, `timeout`, `business`) помогают при алёртинге: «ошибки сети > X% за N минут» — один тип реакции, «бизнес-ошибки 4xx всплеснули» — совсем другой.

Prometheus остаётся де-факто стандартом для экспорта. Включив `micrometer-registry-prometheus`, вы получаете `/actuator/prometheus`, который скрапится сервером. Выносите теги на «верхний» уровень: `application`, `instance`, `env`. Будьте аккуратны с кардинальностью — не кладите в теги сырые URL/ID, только шаблоны.

Если вам не хватает из коробки того, что собирает `feign-micrometer`, можно добавить обёртку вокруг клиента и вручную писать дополнительные счётчики: количество ретраев, доля circuit-breaker opens, размер ответа. Делают это либо через кастомную Capability, либо через `RequestInterceptor` + `ResponseMapper`, но главная мысль — не плодить метрик со сверхкардинальностью.

Гистограммы и percentiles — инструмент, которым легко выстрелить себе в ногу. Включайте `percentiles-histogram` только для нескольких «горячих» клиентов/методов; иначе хранилище взорвётся. Для остальных достаточно «грубых» p95/p99, вычисляемых на стороне Prometheus из таймеров.

Алёртинг должен быть практичным. «Всё пропало» по одному таймеру — плохая идея. Формируйте SLO-алёрты: `error_ratio(client=X, method=Y) > threshold` и `latency_p95(client=X, method=Y) > target`. Дополнительно полезны алёрты на «отсутствие трафика» — это ловит проблемы в маршрутизации или discovery.

Не забывайте кореллировать метрики клиента и сервера. Если downstream — ваш же сервис, сравните p95 на стороне клиента и сервера: расхождение часто указывает на сеть/TLS, а не на код. Для внешних провайдеров полезно хранить их статус в отдельном лейбле, если SLA различаются.

Выкатывая новые клиенты, сразу добавляйте их в дашборды. «Родившийся» без метрик клиент часто забывается, а проблемы обнаруживаются уже в продакшне. Шаблонные панели Grafana с переменными по клиенту/методу — быстрый и удобный путь.

Наконец, метрики — это контракт команд. Задокументируйте имена, теги, где искать панели, какие алёрты настроены. Это снижает «входной порог» и делает расследования предсказуемыми.

**Gradle (Groovy/Kotlin) — Micrometer, Prometheus, Feign Micrometer**

```groovy
dependencies {
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.github.openfeign:feign-micrometer:13.2'
}
```

```kotlin
dependencies {
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.github.openfeign:feign-micrometer:13.2")
}
```

**Java — кастомные теги и таймер вокруг клиента**

```java
package com.example.feign.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

import java.util.concurrent.Callable;

@Component
public class FeignMetricsHelper {
    private final MeterRegistry registry;

    public FeignMetricsHelper(MeterRegistry registry) {
        this.registry = registry;
    }

    public <T> T time(String client, String operation, Callable<T> c) throws Exception {
        Timer t = Timer.builder("feign.client.request")
                .tag("client", client)
                .tag("operation", operation)
                .publishPercentileHistogram(false)
                .register(registry);
        return t.recordCallable(c);
    }
}
```

**Kotlin — вызов клиента с ручной обёрткой**

```kotlin
package com.example.feign.metrics

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Component

@Component
class FeignMetricsHelper(private val registry: MeterRegistry) {
    fun <T> time(client: String, op: String, f: () -> T): T {
        val t = Timer.builder("feign.client.request")
            .tag("client", client)
            .tag("operation", op)
            .register(registry)
        return t.recordCallable { f() }
    }
}
```

**application.yml — экспортер Prometheus**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

---

## Трейсинг: OpenTelemetry автоинструментация, propagation context

Современный стек Spring Boot 3 использует Micrometer Tracing (Brave/OTel bridge) и OpenTelemetry. Идея проста: каждый исходящий Feign-вызов становится «клиентским» спаном, автоматически получает контекст от входящего запроса и передаёт его далее через заголовки `traceparent`/`baggage`. Это позволяет видеть в одной трассе путь запроса через gateway, BFF, downstream-сервисы и обратно.

Автоинструментация работает с минимальной конфигурацией: добавляете зависимости `micrometer-tracing-bridge-otel` и OTLP-экспортёр, указываете `OTEL_EXPORTER_OTLP_ENDPOINT` на коллектор (Tempo/OTel Collector/Jaeger), и спаны начинают «лететь». Spring Cloud OpenFeign 4 дополнительно использует Observation API, так что вручная обвязка почти не нужна.

Семантика спанов важна: имя должно отражать «что делает клиент», а не только HTTP-метод. Используйте шаблоны URI без параметров (`/orders/{id}`), тег `http.method`, `http.status_code`, `net.peer.name`. Чем меньше кардинальность — тем дешевле хранение и тем проще отлавливать «красные» участки.

Propagation — сердце распределённого трейсинга. Для межъязыковых сетей выбирайте W3C Trace Context, он уже по умолчанию в OTel. `baggage` используйте экономно и только для тех ключей, которые действительно нужны downstream (например, `tenant`, `locale`). Помните, что каждый байт заголовка увеличивает латентность на каналах с MTU-ограничениями.

Сэмплирование — компромисс между ценой и пользой. Для «шумных» путей достаточно 5–10% сэмпла с «высвечиванием» ошибок до 100%. Для критических бизнес-операций иногда включают 100% трассы на короткий промежуток. Конфигурация задаётся на коллекторе; приложение должно просто правильно размечать спаны.

Ошибки в спане должны быть отмечены атрибутами и статусом. Если `ErrorDecoder` вернул бизнес-исключение, пометьте спан статусом `Error`, добавьте `error.type=business` и `error.code`. Для сетевых проблем — `error.type=network`, чтобы алёртинг мог различать категории.

Связка метрик и трейсинга даёт максимальный эффект. Наблюдая всплеск p99 у клиента, вы кликаете прямо в «медленную» трассу, видите конкретный downstream-эндпоинт и время на каждом этапе (DNS, TLS, чтение тела). Это закрывает 80% RCA за минуты.

Экспортируйте трейс-ID в логи. Тогда любой лог-сообщение можно приложить к трассе, а трассу — к логу; и вы видите обе стороны — агрегированную и детальную. Это особенно важно при «дрожи» латентности, когда метрик мало, но логи подскажут редкие аномалии.

Не включайте BODY в спаны: это не место для персональных данных. Если нужно понять payload — делайте это в локальной отладке или через выборочный лог с маскированием. Хранить тела в трассах — риск и деньги.

И наконец, договоритесь в команде о минимальном наборе атрибутов спана. «Дикий» зоопарк тегов делает панели и фильтры бесполезными. Консистентность — первичнее количества.

**Gradle (Groovy/Kotlin) — Micrometer Tracing + OTLP**

```groovy
dependencies {
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
}
```

```kotlin
dependencies {
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

**application.yml — OTLP эндпоинт**

```yaml
management:
  tracing:
    enabled: true
otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
```

**Java — кастомный атрибут на Observation (при необходимости)**

```java
package com.example.feign.tracing;

import io.micrometer.observation.ObservationFilter;
import io.micrometer.observation.Observation;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignObservationConfig {
    @Bean
    ObservationFilter addLowCardinality() {
        return (context) -> {
            if (context.getName().startsWith("http.client.requests")) {
                context.addLowCardinalityKeyValue("component", "feign");
            }
            return context;
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.tracing

import io.micrometer.observation.ObservationFilter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FeignObservationConfig {
    @Bean
    fun addLowCardinality(): ObservationFilter = ObservationFilter { ctx ->
        if (ctx.name.startsWith("http.client.requests")) {
            ctx.addLowCardinalityKeyValue("component", "feign")
        }
        ctx
    }
}
```

---

## Производительность: пулы, re-use соединений, gzip, минимизация JSON (модули Jackson)

Производительность Feign упирается не в сам Feign, а в транспортный клиент и в полезную нагрузку. Ключевые рычаги — connection pooling, keep-alive, разумные тайм-ауты и сжатие. Для высоконагруженных сервисов обычно выбирают OkHttp: у него эффективный пул, понятная модель тайм-аутов, HTTP/2 «из коробки». Apache HttpClient всё ещё уместен, если нужна тонкая кастомизация прокси/аутентификации или специфические TLS-настройки.

Connection pool сокращает издержки на TCP/TLS-установку. Следите, чтобы лимиты по пулу соответствовали степени конкуррентности вашего сервиса: слишком маленький — очереди и рост p95; слишком большой — распухшие сокеты и давление на downstream. Для OkHttp имеет смысл 50–200 keep-alive соединений на крупный сервис; точные цифры — только после профиля. Keep-alive должен быть согласован с downstream (idle timeout). Несогласованность приводит к «half-open» и обрывам при чтении.

Сжатие gzip/deflate экономит трафик и иногда заметно снижает p95, если сеть узкое место. Но CPU не бесплатен: на маленьких телах компрессия может, наоборот, ухудшить метрики. Включайте сжатие для «тяжёлых» запросов/ответов и проверяйте профилем. Для JSON-ответов также полезно уменьшить размер: отключить ненужные поля, использовать компактные именования, сериализовать даты в строках без избыточности.

Jackson — частый «бутылочный горлышко». Подключайте `Afterburner` для ускорения сериализации/десериализации, используйте `ObjectReader`/`ObjectWriter` как долгоживущие объекты (кешируемые), а не создавайте новый маппер на каждый вызов. Форматируйте даты через `JavaTimeModule` и ISO-8601 без таймзоны там, где это допустимо, чтобы уменьшить работу парсера.

HTTP/2 при наличии OkHttp и поддержки у провайдера зачастую даёт выигрыш на множественных мелких запросах за счёт мультиплексирования. Но важна правильная настройка keep-alive и отсутствие head-of-line blocking в сети. Тестируйте A/B — не всегда H2 быстрее H1.1 в реальной инфраструктуре.

Равновесие тайм-аутов — искусство. `connectTimeout` — очень короткий (сотни миллисекунд), `readTimeout` — функция от SLO downstream-метода. Не поддавайтесь «искушению увеличить» — лучше деградировать быстро и ретраить по правилам, чем держать поток и забивать пул.

Сериализация больших структур часто ускоряется переходом на потоковую обработку (`InputStream` на ответ вместо `byte[]`) и пообъектным чтением (Jackson streaming API). Это снижает пиковое потребление памяти и уменьшает GC-паузы.

Наконец, профилируйте. Flame-графы на прод-трафике в условиях боевых данных нередко показывают неожиданное: TLS-handshake из-за отсутствия session resumption, неочевидные аллокации в логгере, невыгодные конвертации дат. Любая общая рекомендация — только гипотеза до тех пор, пока вы её не подтвердите профилем.

**Java — кастомный OkHttpClient для Feign с пулом и тайм-аутами**

```java
package com.example.feign.perf;

import feign.Feign;
import feign.Logger;
import feign.okhttp.OkHttpClient;
import okhttp3.ConnectionPool;
import okhttp3.Dispatcher;
import okhttp3.Protocol;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;
import java.util.concurrent.TimeUnit;

@Configuration
public class OkHttpPerfConfig {

    @Bean
    public okhttp3.OkHttpClient okHttp() {
        return new okhttp3.OkHttpClient.Builder()
                .connectionPool(new ConnectionPool(200, 60, TimeUnit.SECONDS))
                .dispatcher(new Dispatcher() {{ setMaxRequests(256); setMaxRequestsPerHost(128); }})
                .protocols(List.of(Protocol.HTTP_2, Protocol.HTTP_1_1))
                .retryOnConnectionFailure(true)
                .connectTimeout(500, TimeUnit.MILLISECONDS)
                .readTimeout(1500, TimeUnit.MILLISECONDS)
                .build();
    }

    @Bean
    public Feign.Builder feignBuilder(okhttp3.OkHttpClient client) {
        return Feign.builder()
                .client(new OkHttpClient(client))
                .logger(new Logger.NoOpLogger())
                .logLevel(Logger.Level.BASIC);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.perf

import feign.Feign
import feign.Logger
import feign.okhttp.OkHttpClient
import okhttp3.ConnectionPool
import okhttp3.Dispatcher
import okhttp3.Protocol
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.TimeUnit

@Configuration
class OkHttpPerfConfig {
    @Bean
    fun okHttp(): okhttp3.OkHttpClient =
        okhttp3.OkHttpClient.Builder()
            .connectionPool(ConnectionPool(200, 60, TimeUnit.SECONDS))
            .dispatcher(Dispatcher().apply {
                maxRequests = 256
                maxRequestsPerHost = 128
            })
            .protocols(listOf(Protocol.HTTP_2, Protocol.HTTP_1_1))
            .retryOnConnectionFailure(true)
            .connectTimeout(500, TimeUnit.MILLISECONDS)
            .readTimeout(1500, TimeUnit.MILLISECONDS)
            .build()

    @Bean
    fun feignBuilder(client: okhttp3.OkHttpClient): Feign.Builder =
        Feign.builder()
            .client(OkHttpClient(client))
            .logger(Logger.NoOpLogger())
            .logLevel(Logger.Level.BASIC)
}
```

**Gradle — ускорение Jackson**

```groovy
dependencies {
    implementation 'com.fasterxml.jackson.module:jackson-module-parameter-names'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
    implementation 'com.fasterxml.jackson.module:jackson-module-afterburner'
}
```

```kotlin
dependencies {
    implementation("com.fasterxml.jackson.module:jackson-module-parameter-names")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    implementation("com.fasterxml.jackson.module:jackson-module-afterburner")
}
```

**application.yml — gzip**

```yaml
feign:
  compression:
    request:
      enabled: true
      mime-types: application/json,application/xml
      min-request-size: 1024
    response:
      enabled: true
```

---

## Паттерны: bulkhead (ограничение параллелизма), очереди/батчинг, кэширование ответов

Bulkhead («переборки») ограничивает параллелизм исходящих вызовов, чтобы один «тяжёлый» downstream не съедал весь пул потоков и не валил весь сервис. В Java-мире удобно использовать Resilience4j Bulkhead или semaphor-bulkhead из Spring Cloud CircuitBreaker. Настройте «потолок» одновременных вызовов и «очередь ожидания». Это спасёт при ухудшении downstream: вместо лавинообразного роста латентности вы получите контролируемые отказы «сразу».

Очереди на входе — второй уровень защиты. Если фронтовой поток запросов выше, чем downstream способен переварить, очередь сгладит пики и снимет «резонансы». Реализовать можно через `TaskExecutor`/`Scheduler` с ограничением параллелизма или через реактивные очереди (если стек позволяет). Важно не делать «черные дыры» — лимитируйте и длину очереди.

Батчинг — классика. Когда API downstream позволяет «мульти-get» («отдайте сразу 100 объектов по ID»), агрегируйте мелкие запросы. Этим вы сокращаете накладные расходы HTTP и сериализации. В сервисах уровня BFF батчинг часто даёт двукратный выигрыш по p95 без потерь семантики. Главное — удостовериться в идемпотентности и в разумных лимитах тела.

Кэширование ответов на короткие окна (секунды/десятки секунд) «гасит» шум от часто повторяющихся запросов (например, справочники или конфигурации). В проде используют Caffeine/Redis в зависимости от объёма и «горячести». Учтите ключи инвалидции (tenant, locale, версия) — плохой ключ разрушает пользу кэша.

Не путайте кэширование и повторные попытки. Ретрай — это реакция на временную сетевую ошибку, кэш — на повторяемый запрос с одинаковыми параметрами. Кэш снижает нагрузку постоянно; ретрай в худшем случае её удваивает. Разделяйте ответственность конфигурацией и кодом.

Композиция паттернов важна. Часто работает связка: bulkhead ограничивает параллелизм, ретраи аккуратно «подхватывают» transient-ошибки, кэш убирает дубли, а батчинг уменьшает число вызовов. Начинайте с простого и измеряйте: лишние «слои» иногда вредят больше, чем помогают.

Считайте экономику. Любой паттерн — это потребление памяти и CPU. Кэш может «съесть» гигабайты, очередь — задержать время ответа. Стоимость и SLA — ваш ориентир, а не «идеальность» по книжке. Нужен профайл и дашборд.

Ошибки проектирования кэш-ключей и ретраев приводят к «шторму» downstream. Например, ретрай без джиттера на огромной аудитории, «пробивающий» сразу весь флот. Добавляйте джиттер и ограничивайте ретраи идемпотентными методами.

И наконец, документируйте паттерны в ADR (architecture decision record): где включили bulkhead, почему выбрали такие лимиты, какой кэш и на сколько. Это ускорит RCA и даст уверенность при апгрейдах.

**Java — Resilience4j Bulkhead вокруг Feign**

```java
package com.example.feign.resilience;

import io.github.resilience4j.bulkhead.*;
import org.springframework.stereotype.Service;

import java.util.concurrent.Callable;

@Service
public class BulkheadWrappedClient {
    private final OrdersClient client;
    private final Bulkhead bulkhead;

    public BulkheadWrappedClient(OrdersClient client) {
        this.client = client;
        this.bulkhead = Bulkhead.of("orders", BulkheadConfig.custom()
                .maxConcurrentCalls(64)
                .maxWaitDuration(java.time.Duration.ofMillis(50))
                .build());
    }

    public String getMyOrders() throws Exception {
        Callable<String> c = Bulkhead.decorateCallable(bulkhead, () -> client.myOrders());
        return c.call();
    }
}
```

**Kotlin — кэширование Caffeine поверх Feign**

```kotlin
package com.example.feign.cache

import com.github.benmanes.caffeine.cache.Caffeine
import org.springframework.stereotype.Service
import java.util.concurrent.TimeUnit

@Service
class CachedCatalogClient(private val client: CatalogClient) {
    private val cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(30, TimeUnit.SECONDS)
        .build<String, String>()

    fun getItem(id: String): String =
        cache.get(id) { client.getItem(id) }
}
```

**Gradle — зависимости Resilience4j и Caffeine**

```groovy
dependencies {
    implementation 'io.github.resilience4j:resilience4j-bulkhead'
    implementation 'com.github.ben-manes.caffeine:caffeine:3.1.8'
}
```

```kotlin
dependencies {
    implementation("io.github.resilience4j:resilience4j-bulkhead")
    implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
}
```

---

## Чек-листы продакшна: тайм-ауты, retries, CB, алерты по SLA, политика ротации ключей

Эксплуатация начинается с «жёстких» дефолтов. Для каждого клиента зафиксируйте `connectTimeout` (сотни мс), `readTimeout` (согласованное с SLO downstream), максимальное число ретраев (обычно 0–1 для идемпотентных методов), джиттер, и включите circuit-breaker по методу. Эти настройки записывают в `application.yml` per-client и подтверждают на нагрузочном тесте.

Секреты и ключи — под ротацию. Для внешних провайдеров заведите политику смены API-ключей/сертификатов: дата, кто меняет, как проверяем. Feign-интерсепторы должны читать секреты из Secret Store (Vault/KMS), а не из файлов в контейнере. Пропишите в чек-листе «что делаем при компрометации»: где отключить ключ мгновенно и как переключить на запасной.

Алёрты строятся вокруг SLO. «5xx от клиента > 2% за 5 минут» — сигнал, но не паника. «p95 > целевого» — повод копать. Разделяйте алёрты на «пейджерные» (ночью звонит телефон) и «тикетные» (обратить внимание завтра). Избегайте алёртов «на каждый чих» — выгорите сами и обесцените мониторинг.

Управление доступом к логам — обязательный пункт. Кто может видеть `HEADERS/BODY`, где хранится история, как выгрузить кусок для расследования без секретов. Стандартизованные политики и шаблоны запросов в SIEM сэкономят часы при инциденте.

Документируйте договорённости с провайдерами: rate limit, квоты, контакт при инциденте, окно SLA. Это не только юридические бумажки, но и инженерная практика: клиент должен уважать лимиты и «не штормить» ретраями под нагрузкой.

Управляйте «feature flags» вокруг клиентов. Возможность быстро переключить URL на стабы, выключить конкретный ретрай/брейкер, изменить readTimeout — это «красная кнопка», которая экономит деньги и нервы. Такие флаги должны быть управляемы без релиза, через конфиг-сервис/консоль.

Планируйте disaster-drills. Раз в квартал симулируйте обрыв сети, рост latency, 429/503 у провайдера. Смотрите, что происходит с вашими очередями, bulkhead, ретраями и алёртами. Практика выявляет неожиданные связи и «тёмные углы».

Не забывайте про supply-chain. Пиньте версии зависимостей (`spring-cloud-dependencies`, `openfeign`), генерируйте SBOM, проверяйте уязвимости (OSV/Snyk). Обновление транспорта (OkHttp/HttpClient) не должно «приезжать» внезапно через транзитивные зависимости.

И, наконец, сделайте «pre-flight» чек-лист перед релизом: проверка заголовков безопасности, тайм-аутов, ретраев, circuit-breaker, метрик и трассировки, логов и маскировки. Короткий, но обязательный ритуал избавляет от большинства регрессов.

**application.yml — жёсткие дефолты per-client**

```yaml
feign:
  httpclient:
    enabled: false
  okhttp:
    enabled: true
  client:
    config:
      default:
        connectTimeout: 500
        readTimeout: 1500
        loggerLevel: BASIC
      orders:
        readTimeout: 1200
        connectTimeout: 300
```

**Java — Resilience4j CircuitBreaker + Retry вокруг Feign**

```java
package com.example.feign.ops;

import io.github.resilience4j.circuitbreaker.*;
import io.github.resilience4j.retry.*;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.function.Supplier;

@Service
public class GuardedClient {
    private final CatalogClient client;
    private final CircuitBreaker cb;
    private final Retry retry;

    public GuardedClient(CatalogClient client) {
        this.client = client;
        this.cb = CircuitBreaker.of("catalog", CircuitBreakerConfig.custom()
                .failureRateThreshold(50f)
                .slidingWindowSize(50)
                .waitDurationInOpenState(Duration.ofSeconds(10))
                .build());
        this.retry = Retry.of("catalog", RetryConfig.custom()
                .maxAttempts(2)
                .waitDuration(Duration.ofMillis(100))
                .build());
    }

    public String list() {
        Supplier<String> base = () -> client.list();
        return Retry.decorateSupplier(retry, CircuitBreaker.decorateSupplier(cb, base)).get();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.feign.ops

import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig
import io.github.resilience4j.retry.Retry
import io.github.resilience4j.retry.RetryConfig
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class GuardedClient(private val client: CatalogClient) {
    private val cb = CircuitBreaker.of("catalog", CircuitBreakerConfig.custom()
        .failureRateThreshold(50.0f)
        .slidingWindowSize(50)
        .waitDurationInOpenState(Duration.ofSeconds(10))
        .build())

    private val retry = Retry.of("catalog", RetryConfig.custom()
        .maxAttempts(2)
        .waitDuration(Duration.ofMillis(100))
        .build())

    fun list(): String =
        Retry.decorateSupplier(retry, CircuitBreaker.decorateSupplier(cb) { client.list() }).get()
}
```

**Prometheus alert (пример идеи, адаптируйте под свои метрики)**

```yaml
groups:
  - name: feign-slo
    rules:
      - alert: FeignHighErrorRate
        expr: sum(rate(feign_client_requests_seconds_count{status=~"5.."}[5m])) by (client,method)
              / sum(rate(feign_client_requests_seconds_count[5m])) by (client,method) > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "High 5xx for {{ $labels.client }} {{ $labels.method }}"
          description: "Error ratio >5% for 10m"
```





