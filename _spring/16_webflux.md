---
layout: page
title: "Spring WebFlux: Реактивный веб-фреймворк от Spring"
permalink: /spring/webflux
---

<!-- TOC -->
* [0. Введение](#0-введение)
  * [Что это такое](#что-это-такое)
  * [Зачем это](#зачем-это)
  * [Где используется](#где-используется)
  * [Какие проблемы и задачи решает](#какие-проблемы-и-задачи-решает)
* [1. Позиционирование WebFlux](#1-позиционирование-webflux)
  * [Чем WebFlux отличается от Spring MVC: неблокирующая модель, event-loop против per-thread](#чем-webflux-отличается-от-spring-mvc-неблокирующая-модель-event-loop-против-per-thread)
  * [Когда выбирать WebFlux: высокая конкуррентность I/O, стриминг, долгие соединения](#когда-выбирать-webflux-высокая-конкуррентность-io-стриминг-долгие-соединения)
  * [Ограничения: «всё должно быть реактивным» (БД, HTTP-клиенты, файловая I/O)](#ограничения-всё-должно-быть-реактивным-бд-http-клиенты-файловая-io)
  * [Стек по умолчанию: Reactor, reactor-netty, codecs, WebClient](#стек-по-умолчанию-reactor-reactor-netty-codecs-webclient)
  * [Совместимость: можно запускать на Netty/Undertow/Servlet-контейнере 3.1+](#совместимость-можно-запускать-на-nettyundertowservlet-контейнере-31)
  * [Типичные use-cases: чат/нотификации (SSE/WS), API-агрегация, BFF для мобильных](#типичные-use-cases-чатнотификации-ssews-api-агрегация-bff-для-мобильных)
* [2. Основы Reactor (Flux/Mono)](#2-основы-reactor-fluxmono)
  * [Модель pull-push и backpressure; холодные/горячие источники](#модель-pull-push-и-backpressure-холодныегорячие-источники)
  * [Типы: `Mono<T>` и `Flux<T>` — композиция vs коллекции](#типы-monot-и-fluxt--композиция-vs-коллекции)
  * [Операторы: `map/flatMap/concatMap/switchMap`, `error/timeout/retry`](#операторы-mapflatmapconcatmapswitchmap-errortimeoutretry)
  * [Планировщики: `Schedulers.parallel()`/`boundedElastic()` — когда и зачем](#планировщики-schedulersparallelboundedelastic--когда-и-зачем)
  * [Context и MDC: перенос корреляционных айди и безопасности](#context-и-mdc-перенос-корреляционных-айди-и-безопасности)
  * [Диагностика: `log()`, checkpoints, Hooks для улучшения стэктрейсов](#диагностика-log-checkpoints-hooks-для-улучшения-стэктрейсов)
* [3. Архитектура WebFlux](#3-архитектура-webflux)
  * [`HttpHandler` → `WebFlux` → `DispatcherHandler` (аннотации) или Router/Handler (функционально)](#httphandler--webflux--dispatcherhandler-аннотации-или-routerhandler-функционально)
  * [Серверная платформа: Reactor Netty, event-loop, non-blocking сокеты](#серверная-платформа-reactor-netty-event-loop-non-blocking-сокеты)
  * [Кодеки: Jackson/JSON, protobuf, byte buffers; ручной контроль `DataBuffer`](#кодеки-jacksonjson-protobuf-byte-buffers-ручной-контроль-databuffer)
  * [Потоки ответов: стриминг, chunked, `ServerSentEvent`](#потоки-ответов-стриминг-chunked-serversentevent)
  * [Модель блокировок: где нельзя блокировать и как оградить `boundedElastic`](#модель-блокировок-где-нельзя-блокировать-и-как-оградить-boundedelastic)
  * [Ограничения памяти/очередей: настройка серверных буферов и timeouts](#ограничения-памятиочередей-настройка-серверных-буферов-и-timeouts)
* [4. Аннотационные контроллеры (MVC-стиль в реактивном мире)](#4-аннотационные-контроллеры-mvc-стиль-в-реактивном-мире)
  * [`@RestController`, `@GetMapping`/`@PostMapping` c `Mono/Flux` телами](#restcontroller-getmappingpostmapping-c-monoflux-телами)
  * [Валидация: Bean Validation в реактивном пайплайне, `@Valid` и обработчики ошибок](#валидация-bean-validation-в-реактивном-пайплайне-valid-и-обработчики-ошибок)
  * [Работа с файлами и формами: `Part`, `FilePart`, многопартовые загрузки](#работа-с-файлами-и-формами-part-filepart-многопартовые-загрузки)
  * [Контент-негациация и сериализация, reactive `HttpMessageReader/Writer`](#контент-негациация-и-сериализация-reactive-httpmessagereaderwriter)
  * [Потоковые ответы: SSE, `application/stream+json`, управление частями](#потоковые-ответы-sse-applicationstreamjson-управление-частями)
  * [Глобальный error handling: `@ControllerAdvice`, `ResponseStatusException`](#глобальный-error-handling-controlleradvice-responsestatusexception)
* [5. Функциональные маршруты (RouterFunction/HandlerFunction)](#5-функциональные-маршруты-routerfunctionhandlerfunction)
  * [Декларация маршрутов: `RouterFunctions.route()` и DSL, группировка по префиксам](#декларация-маршрутов-routerfunctionsroute-и-dsl-группировка-по-префиксам)
  * [Обработчики: `ServerRequest`/`ServerResponse`, паттерн «чистых» хендлеров](#обработчики-serverrequestserverresponse-паттерн-чистых-хендлеров)
  * [Фильтры и перехватчики: `HandlerFilterFunction`, сквозные аспекты (лог/метрики)](#фильтры-и-перехватчики-handlerfilterfunction-сквозные-аспекты-логметрики)
  * [Ошибки/исключения: фильтры, fallback-маршруты, централизованный маппинг](#ошибкиисключения-фильтры-fallback-маршруты-централизованный-маппинг)
  * [Сосуществование аннотационного и функционального стилей в одном приложении](#сосуществование-аннотационного-и-функционального-стилей-в-одном-приложении)
  * [Где функциональный стиль удобнее: простые API, high-perf маршрутизация, тестируемость](#где-функциональный-стиль-удобнее-простые-api-high-perf-маршрутизация-тестируемость)
* [6. WebClient: реактивный HTTP-клиент](#6-webclient-реактивный-http-клиент)
  * [Конфигурация: connection pool, timeouts, codecs, `ExchangeStrategies`](#конфигурация-connection-pool-timeouts-codecs-exchangestrategies)
  * [Маппинг ответов: `retrieve()/exchangeToMono()`, обработка 4xx/5xx, `onStatus`](#маппинг-ответов-retrieveexchangetomono-обработка-4xx5xx-onstatus)
  * [Потоковые запросы/ответы, загрузка файлов, backpressure при чтении тела](#потоковые-запросыответы-загрузка-файлов-backpressure-при-чтении-тела)
  * [Повторные попытки/тайм-ауты: `retryWhen`, `timeout`, идемпотентность](#повторные-попыткитайм-ауты-retrywhen-timeout-идемпотентность)
  * [Интеграция с OAuth2/JWT (через фильтры и `ExchangeFilterFunction`)](#интеграция-с-oauth2jwt-через-фильтры-и-exchangefilterfunction)
  * [Метрики и трассировки: Micrometer, HTTP-тэги, распределённый трейсинг](#метрики-и-трассировки-micrometer-http-тэги-распределённый-трейсинг)
* [7. Доступ к данным: R2DBC и друзья](#7-доступ-к-данным-r2dbc-и-друзья)
  * [Почему JDBC блокирует и как R2DBC решает задачу; драйверы и ограничения](#почему-jdbc-блокирует-и-как-r2dbc-решает-задачу-драйверы-и-ограничения)
  * [Spring Data R2DBC/Reactive Repositories: транзакции, `@Transactional` в реактивном мире](#spring-data-r2dbcreactive-repositories-транзакции-transactional-в-реактивном-мире)
  * [Реактивные хранилища: MongoDB/Cassandra/Redis — особенности моделей](#реактивные-хранилища-mongodbcassandraredis--особенности-моделей)
  * [«Мосты» к блокирующим API: `boundedElastic`, bulk-операции, анти-паттерны](#мосты-к-блокирующим-api-boundedelastic-bulk-операции-анти-паттерны)
  * [Потоковые запросы и пагинация: курсоры/лимиты, backpressure на БД уровне](#потоковые-запросы-и-пагинация-курсорылимиты-backpressure-на-бд-уровне)
  * [Тонкая настройка пула соединений и тайм-аутов для стабильности](#тонкая-настройка-пула-соединений-и-тайм-аутов-для-стабильности)
* [8. Безопасность в WebFlux (reactive)](#8-безопасность-в-webflux-reactive)
  * [`ServerHttpSecurity` и реактивный `SecurityWebFilterChain`](#serverhttpsecurity-и-реактивный-securitywebfilterchain)
  * [JWT/OAuth2 Resource Server в реактивном приложении, маппинг scopes→authorities](#jwtoauth2-resource-server-в-реактивном-приложении-маппинг-scopesauthorities)
  * [CSRF/CORS/headers в контексте SPA и мобильных клиентов](#csrfcorsheaders-в-контексте-spa-и-мобильных-клиентов)
  * [Методная безопасность: `@EnableMethodSecurity` (reactive), `ReactiveAuthorizationManager`](#методная-безопасность-enablemethodsecurity-reactive-reactiveauthorizationmanager)
  * [Перенос аутентификации в Reactor Context (`ReactiveSecurityContextHolder`)](#перенос-аутентификации-в-reactor-context-reactivesecuritycontextholder)
  * [Тестирование безопасности: `WebTestClient` + `@WithMockUser` (reactive), мок JWT](#тестирование-безопасности-webtestclient--withmockuser-reactive-мок-jwt)
* [9. Тестирование и отладка WebFlux](#9-тестирование-и-отладка-webflux)
  * [WebTestClient против MockMvc: чёрный ящик vs сервер-менее](#webtestclient-против-mockmvc-чёрный-ящик-vs-сервер-менее)
  * [`StepVerifier` и виртуальное время для сложных последовательностей](#stepverifier-и-виртуальное-время-для-сложных-последовательностей)
  * [Контроль блокировок: BlockHound, выявление «случайного JDBC»](#контроль-блокировок-blockhound-выявление-случайного-jdbc)
  * [Интеграции: WireMock/MockServer для WebClient, контракты и негативные кейсы](#интеграции-wiremockmockserver-для-webclient-контракты-и-негативные-кейсы)
  * [Наблюдаемость: Micrometer timers, распределённые трейсы, корреляция request-id](#наблюдаемость-micrometer-timers-распределённые-трейсы-корреляция-request-id)
  * [Testcontainers c R2DBC/БД, изоляция и миграции (Flyway/Liquibase) перед тестом](#testcontainers-c-r2dbcбд-изоляция-и-миграции-flywayliquibase-перед-тестом)
* [10. Продакшн-паттерны и производительность](#10-продакшн-паттерны-и-производительность)
  * [Тюнинг reactor-netty: event-loops, размеры пулов/буферов, keep-alive, H2C/H2](#тюнинг-reactor-netty-event-loops-размеры-пуловбуферов-keep-alive-h2ch2)
  * [Управление backpressure: ограничение параллелизма, `limitRate`, оконные операторы](#управление-backpressure-ограничение-параллелизма-limitrate-оконные-операторы)
  * [Долгоживущие соединения: SSE/WebSocket/RSocket — когда что выбрать](#долгоживущие-соединения-ssewebsocketrsocket--когда-что-выбрать)
  * [Резилиентность: тайм-ауты, `retryWhen`, circuit-breaker (Resilience4j reactive)](#резилиентность-тайм-ауты-retrywhen-circuit-breaker-resilience4j-reactive)
  * [Профилирование и перф-аудит: flame-графы, reactor-debug, метрики на операцию](#профилирование-и-перф-аудит-flame-графы-reactor-debug-метрики-на-операцию)
  * [Деплой и окружение: контейнеры, k8s ресурсы/пробы, graceful shutdown и drain](#деплой-и-окружение-контейнеры-k8s-ресурсыпробы-graceful-shutdown-и-drain)
<!-- TOC -->

# 0. Введение

Ниже — единые зависимости и настройки, которые пригодятся для всех пунктов введения. Подключите один раз, а в примерах далее будут только контроллеры, роутеры и конфигурации.

**Gradle (Groovy)**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'io.projectreactor:reactor-tools' // checkpoints/debug
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.projectreactor:reactor-test'
}
test { useJUnitPlatform() }
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("io.projectreactor:reactor-tools")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.projectreactor:reactor-test")
}
tasks.test { useJUnitPlatform() }
```

---

## Что это такое

Spring WebFlux — это реактивный веб-фреймворк в составе Spring, построенный на спецификации Reactive Streams и библиотеке Project Reactor. Он использует неблокирующую модель ввода-вывода и событийные циклы вместо классической схемы «один поток на запрос». Благодаря этому одна машина может эффективно обрабатывать большое количество параллельных подключений.

В терминах API WebFlux предлагает два стиля: аннотационный, очень похожий на Spring MVC, и функциональный, основанный на `RouterFunction`/`HandlerFunction`. Первый удобен знакомством и миграцией, второй — для компактной и высокопроизводительной маршрутизации без рефлексии.

Фреймворк поддерживает популярные серверные движки: по умолчанию это Reactor Netty, но можно работать и поверх сервлетов 3.1+ (Tomcat/Jetty/Undertow) в неблокирующем режиме. Это означает, что переход на WebFlux не всегда требует отказа от привычной инфраструктуры.

Ключевая идея реактивности — управляемый поток данных с поддержкой обратного давления (backpressure). Потребитель сам сообщает источнику, сколько элементов он готов принять, и тем самым система избегает перегрузки буферами и очередями.

WebFlux естественно поддерживает потоковые ответы: Server-Sent Events, chunked-передачу, WebSocket. Это полезно для чатов, нотификаций в реальном времени и любых сценариев «долгоживущих» соединений.

Внутри WebFlux всё опирается на типы `Mono<T>` (один элемент или пусто) и `Flux<T>` (много элементов). Это контейнеры для асинхронных результатов, с сотнями операторов композиции, преобразований и обработки ошибок.

Фреймворк дружит с реактивными драйверами и клиентами: HTTP-клиент `WebClient`, драйверы R2DBC для SQL-баз, реактивные клиенты для MongoDB, Cassandra, Redis. Это важно, потому что блокирующие компоненты в реактивном приложении могут обнулить плюсы неблокирующей модели.

Безопасность реализуется реактивно через `ServerHttpSecurity` и `SecurityWebFilterChain`. Это отдельная реализация, совместимая по идеям со Spring Security, но адаптированная к неблокирующей модели и реактивному контексту.

Философия WebFlux — не «быстрее любой ценой», а «предсказуемая производительность при высокой конкурентности I/O». На чисто CPU-сценариях реактивность не даёт преимуществ, но там, где есть сеть и ожидание, эффект заметен.

Наконец, WebFlux — это не «замена MVC», а параллельный стек. Для классических CRUD-админок и рендеринга серверных шаблонов MVC остаётся нормальным выбором, а WebFlux добавляет инструментарий для I/O-интенсивных задач.

**Java — минимальный аннотационный контроллер WebFlux**

```java
package com.example.intro;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

@RestController
class HelloReactiveController {

    @GetMapping(value = "/hello", produces = MediaType.TEXT_PLAIN_VALUE)
    public Mono<String> hello() {
        return Mono.just("Hello, WebFlux!");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.intro

import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Mono

@RestController
class HelloReactiveController {

    @GetMapping("/hello", produces = [MediaType.TEXT_PLAIN_VALUE])
    fun hello(): Mono<String> = Mono.just("Hello, WebFlux!")
}
```

---

## Зачем это

Реактивная модель помогает выдерживать высокую конкуррентность при ограниченных ресурсах. Когда большинство времени уходит на ожидание удалённых сервисов или БД, неблокирующие event-loop’ы позволяют тем же потокам обслужить больше запросов без линейного роста памяти и контекст-свитчей.

WebFlux облегчает работу с потоками данных и стримингом. Отдавать клиенту результаты по мере готовности — естественный сценарий: SSE, «стрим-JSON», чанки. Это снижает время до первого байта и улучшает UX.

Долгоживущие соединения вроде WebSocket и SSE — типичная боль для «поток на запрос», потому что каждый «зависший» запрос занимает ресурс. В WebFlux «зависшие» соединения — нормальное состояние, и их можно держать тысячами.

Фреймворк упрощает написание BFF-слоя для мобильных/SPA-клиентов. Он эффективен как «агрегатор», который одновременно дергает несколько бэкендов через `WebClient`, объединяет ответы и возвращает клиенту единый DTO.

Неблокирующий клиент `WebClient` удобен для кросс-сервисной коммуникации: общие фильтры, метрики, корреляция трассировок, гибкая обработка статусов. Он позволяет не городить ручные пулы потоков и тайм-ауты для каждого вызова.

Вместе с реактивными драйверами (R2DBC, Mongo Reactive) WebFlux формирует цельный стек, где backpressure и тайм-ауты проходят сквозь все уровни. Это сокращает риск неожиданной перегрузки где-то «посередине».

Реактивность — это ещё и предсказуемость. Когда нет тысячи блокирующих потоков, GC и планировщик ОС ведут себя стабильнее, Tail-латентность падает, а деградации появляются позже и плавнее.

В современном облаке лучше масштабируется «немного CPU + много I/O». WebFlux помогает максимально использовать эту модель, не переплачивая за «лишние» ядра и гигабайты памяти.

Фреймворк даёт инструменты для управляемых ретраев, тайм-аутов, отмены запросов при закрытии соединения. Это критично для устойчивости к сетевым сбоям и капризам зависимостей.

Важно понимать границы. Если ваша нагрузка — CPU-интенсивная обработка в памяти, WebFlux не даст прироста. Но если вы упирались в «закон Мёрфи» сетей, он часто превращает «красную зону» в рабочую.

**Java — простейший SSE-стриминг**

```java
package com.example.intro;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.time.Duration;

@RestController
class SseController {

    @GetMapping(value = "/ticks", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> ticks() {
        return Flux.interval(Duration.ofSeconds(1)).map(i -> "tick-" + i);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.intro

import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Flux
import java.time.Duration

@RestController
class SseController {

    @GetMapping("/ticks", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun ticks(): Flux<String> = Flux.interval(Duration.ofSeconds(1)).map { "tick-$it" }
}
```

---

## Где используется

WebFlux особенно уместен в API-шлюзах и BFF-сервисах, которые агрегируют данные из нескольких источников. Он позволяет параллелить вызовы и отдавать клиенту результат раньше, не дожидаясь «самого медленного» сервиса, если контракт это позволяет.

Сервисы нотификаций и real-time UI часто требуют поддерживать тысячи постоянных подключений. В этой роли WebFlux даёт экономичный и контролируемый по ресурсам сервер для SSE/WebSocket-каналов.

Интеграционные слои и прокси к внешним API выигрывают от неблокирующих клиентов. Пул соединений и событийные циклы держат высокий RPS без взрывного роста потоков и памяти.

Сервисы, работающие с реактивными БД (Mongo, Cassandra) или R2DBC-драйверами, естественным образом сочетаются с WebFlux. Сквозная реактивность снижает вероятность «бутылочного горлышка» между слоями.

В сценариях «длинных» запросов — большой отчёт, конвертация, загрузка/выгрузка — WebFlux позволяет аккуратно отдавать прогресс или порции данных, не занимая поток «на весь запрос».

Микросервисы, которым важно предсказуемо отстреливать ретраи и тайм-ауты, получают встроенную композицию операторов Reactor. Это упрощает внедрение policy-шаблонов без «лесов» вокруг `ExecutorService`.

Лёгкие edge-сервисы, выполняющие трансформации и фильтрацию, можно писать функциональным стилем: кратко и прозрачно. Под нагрузкой такой стиль часто показывает лучшие показатели задержек.

Для IoT/высокочастотных телеметрий WebFlux полезен как серверный «приёмник»: он выдерживает бурсты подключений и даёт тонкий контроль над буферами и давлением.

При необходимости «плавного» перехода можно запускать WebFlux поверх сервлетов 3.1+, сохраняя часть инфраструктуры. Это снижает барьеры миграции.

Смешанные приложения с MVC и WebFlux тоже встречаются: например, MVC для административных страниц и WebFlux для API-аггрегатора и SSE. Отдельные цепочки безопасности и конфигурации позволяют им мирно сосуществовать.

**Java — агрегатор двух сервисов через WebClient**

```java
package com.example.intro;

import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

record A(String value) {}
record B(String value) {}
record AB(String a, String b) {}

@Service
class AggregationService {
    private final WebClient client = WebClient.builder().baseUrl("http://localhost:8081").build();

    Mono<A> a() { return client.get().uri("/a").retrieve().bodyToMono(A.class); }
    Mono<B> b() { return client.get().uri("/b").retrieve().bodyToMono(B.class); }

    Mono<AB> ab() { return Mono.zip(a(), b()).map(t -> new AB(t.getT1().value(), t.getT2().value())); }
}

@RestController
class AggregationController {
    private final AggregationService svc;
    AggregationController(AggregationService svc) { this.svc = svc; }

    @GetMapping("/ab")
    Mono<AB> ab() { return svc.ab(); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.intro

import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.reactive.function.client.WebClient
import reactor.core.publisher.Mono

data class A(val value: String)
data class B(val value: String)
data class AB(val a: String, val b: String)

@Service
class AggregationService {
    private val client = WebClient.builder().baseUrl("http://localhost:8081").build()

    fun a(): Mono<A> = client.get().uri("/a").retrieve().bodyToMono(A::class.java)
    fun b(): Mono<B> = client.get().uri("/b").retrieve().bodyToMono(B::class.java)
    fun ab(): Mono<AB> = Mono.zip(a(), b()).map { (a, b) -> AB(a.value, b.value) }
}

@RestController
class AggregationController(private val svc: AggregationService) {
    @GetMapping("/ab")
    fun ab(): Mono<AB> = svc.ab()
}
```

---

## Какие проблемы и задачи решает

Первая проблема классического стека — масштабирование блокирующих потоков при длинных I/O-операциях. WebFlux решает её event-loop-моделью и неблокирующими клиентами/драйверами, позволяя обрабатывать больше соединений на ядро.

Вторая задача — потоковые ответы и «пакеты по мере готовности». Реактивный конвейер естественно выражает такие сценарии оператором `map/flatMap` и типами `Flux`, не требуя «ручной» работы с буферами и потоками.

Третья — управление обратным давлением. Вместо того чтобы «утопать» в очередях, WebFlux и Reactor позволяют явно ограничивать скорость потребления и параллельность, не теряя данных и не умирая от OOM.

Четвёртая — единообразная обработка ошибок, ретраев, тайм-аутов и отмены операций. Реактивные операторы превращают эти политики из «размазанных по коду» в декларативные цепочки.

Пятая — ресурсоэффективность в облаке. Когда CPU — не узкое место, важно уметь удерживать много соединений и активно работать с сетью, не размножая потоки и не раздувая хип.

Шестая — улучшение Tail-латентности. Отсутствие блокирующих очередей и предсказуемые event-loop’ы сглаживают хвост распределения задержек, что делает SLA надёжнее.

Седьмая — безопасная работа с «шумными» зависимостями. Тайм-ауты, «быстрые отказоустойчивые» маршруты и отмена при закрытии клиента уменьшают удержание ресурсов и каскадные сбои.

Восьмая — явная изоляция блокирующих мест. Если без них нельзя, их можно вынести на `boundedElastic` и контролировать влияние на всю систему, не разрушая event-loop.

Девятая — наблюдаемость. Реактивный стек хорошо дружит с Micrometer/OTel, а контекст можно обогащать метаданными запроса, корреляторами и пользователями.

Десятая — выразительность архитектуры. Когда потоки данных и политика их обработки описаны декларативно, их проще тестировать, читать и сопровождать.

**application.yml — базовые тайм-ауты и размеры пула WebClient (Reactor Netty)**

```yaml
spring:
  main:
    web-application-type: reactive
webclient:
  connect-timeout-ms: 2000
  read-timeout-ms: 3000
```

**Java — ограничение параллелизма и backpressure**

```java
package com.example.intro;

import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;

import java.time.Duration;

public class BackpressureExample {

    public static Flux<Integer> process(Flux<Integer> in) {
        return in
            .limitRate(128)
            .flatMap(i -> Flux.just(i).delayElements(Duration.ofMillis(5)), 32) // max concurrency
            .publishOn(Schedulers.parallel(), 256);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.intro

import reactor.core.publisher.Flux
import reactor.core.scheduler.Schedulers
import java.time.Duration

object BackpressureExample {
    fun process(input: Flux<Int>): Flux<Int> =
        input
            .limitRate(128)
            .flatMap({ i -> Flux.just(i).delayElements(Duration.ofMillis(5)) }, 32)
            .publishOn(Schedulers.parallel(), 256)
}
```

# 1. Позиционирование WebFlux

## Чем WebFlux отличается от Spring MVC: неблокирующая модель, event-loop против per-thread

В классическом Spring MVC одна входящая HTTP-заявка закрепляется за отдельным потоком до отправки ответа. Это просто и прозрачно, но при долгих I/O-операциях поток простаивает и «ест» память под стек и контекст — масштабирование получается дорогим. В неблокирующем мире WebFlux несколько event-loop-потоков обрабатывают множество соединений, переключаясь по готовности данных.

Реактивная модель строится вокруг `Publisher`/`Subscriber` с поддержкой backpressure, чтобы источник не заливал потребителя данными быстрее, чем тот успевает. В MVC backpressure нет — если вы читаете тело запроса/ответа, то либо блокируетесь, либо крутите собственные очереди. В WebFlux эта дисциплина встроена во все уровни — от сокета до сериализации.

Разница чувствуется на длинных соединениях (SSE/WS) и стриминге: в MVC «подвешенный» запрос удерживает поток, а в WebFlux — нет. Поэтому тысячи «висящих» SSE-каналов на одной ноде — нормальный сценарий для WebFlux, а для MVC это быстро упирается в пределы thread-pool’а. Tail-латентность у WebFlux обычно стабильнее под высокими конкарентными I/O.

Однако «реактивность ради реактивности» не нужна на CPU-тяжёлых задачах. Когда доминирует вычисление в памяти, выигрыша от неблокирующих сокетов почти нет. Более того, дополнительная сложность реактивного кода может только мешать.

Ещё одно отличие — контракт контроллера: в MVC вы возвращаете готовый объект/`ResponseEntity`, а в WebFlux — `Mono<T>`/`Flux<T>`. Это не «отложенное вычисление ради моды», это единая модель для тайм-аутов, отмены, ретраев и стриминга. Вам не нужно вручную управлять потоками и коллбэками.

Поскольку WebFlux крутится вокруг небольшой группы event-loop’ов, блокирующие вызовы в них недопустимы. «Случайный» JDBC-вызов или чтение файла заблокирует цикл и заморозит множество клиентов. Если без блокировки нельзя — переносим на `boundedElastic()` и чётко дозируем.

В MVC механика фильтров/интерсепторов основана на `ServletFilter` и синхронной цепочке. В WebFlux — неблокирующие `WebFilter` и функциональные фильтры роутера, которые видят реактивный поток. Это влияет на способы логирования, метрик и обработки ошибок.

С точки зрения экосистемы, MVC поддерживает зрелый стек (JSP/Thymeleaf, классические библиотечные фильтры, сервлет-контейнеры), а WebFlux — реактивные драйверы, Reactor Netty и новый `WebClient`. Оба стека развиваются параллельно; выбор — вопрос сценария.

Важно понимать, что WebFlux — не «ускоритель MVC». Это другой способ управления ресурсами под I/O-нагрузкой. Пытаться выиграть чисто CPU-бенчмарки бессмысленно, но под десятью тысячами keep-alive соединений разница станет очевидной.

Для миграций существует «смешанный» режим: одно приложение может содержать и MVC, и WebFlux — но тогда нужны изолированные цепочки безопасности и осознанное разделение зон. Это путь к постепенному переходу.

**Java — различие контрактов и недопустимая блокировка**

```java
package com.example.positioning;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

@RestController
class DiffController {

    // "Правильный" реактивный метод: никаких block(), возвращаем Mono.
    @GetMapping(value = "/reactive", produces = MediaType.TEXT_PLAIN_VALUE)
    Mono<String> reactive() {
        return Mono.defer(() -> Mono.just("ok")).timeout(java.time.Duration.ofSeconds(2));
    }

    // Если очень нужно выполнить блокирующую I/O — уводим на boundedElastic.
    @GetMapping(value = "/maybe-blocking", produces = MediaType.TEXT_PLAIN_VALUE)
    Mono<String> maybeBlocking() {
        return Mono.fromCallable(() -> {
                Thread.sleep(50); // имитация блокировки: сети/файла
                return "done";
            })
            .subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers
import java.time.Duration

@RestController
class DiffController {

    @GetMapping("/reactive", produces = [MediaType.TEXT_PLAIN_VALUE])
    fun reactive(): Mono<String> = Mono.defer { Mono.just("ok") }.timeout(Duration.ofSeconds(2))

    @GetMapping("/maybe-blocking", produces = [MediaType.TEXT_PLAIN_VALUE])
    fun maybeBlocking(): Mono<String> =
        Mono.fromCallable {
            Thread.sleep(50)
            "done"
        }.subscribeOn(Schedulers.boundedElastic())
}
```

---

## Когда выбирать WebFlux: высокая конкуррентность I/O, стриминг, долгие соединения

Если ваши запросы тратят большую долю времени вне CPU — в сетях, на диске, в ожидании сторонних сервисов, — WebFlux позволит обслужить больше клиентов на том же железе. Это типичные API-шлюзы и BFF, агрегирующие несколько медленных источников. Там выигрывает неблокирующий пул соединений и экономия потоков.

Когда нужно держать тысячи открытых соединений с пользователями — SSE канал для нотификаций, уведомления в реальном времени, чат — классическая модель «поток на запрос» упирается в лимиты. WebFlux проектировался именно под такие сценарии и раскрывается на них.

Стриминг — отдельная причина. Отдавать результат частями по мере готовности снижает TTFB и улучшает UX. В реактивном конвейере это записывается естественно: `Flux` из элементов, кодек, и клиент получает «порции» данных.

Нагрузочные пики и «бурсты» входящих запросов проще переживать, когда backpressure встроен в стек. Вместо того чтобы раздувать очереди и JVM heap, вы регулируете количество одновременно обрабатываемых элементов и не падаете в OOM.

Если у вас «фан-аут» ко множеству бэкендов, реактивный `WebClient` с `zip/merge` позволяет параллелить IO без взрывного роста потоков. Возврат «частичного» результата по готовности — тоже естественный паттерн.

Для мобильных клиентов с нестабильной сетью WebFlux даёт тонкий контроль тайм-аутов/ретраев и отмену вычислений при закрытии соединения. Это снижает «зависшие» работы и устраняет каскадные отказы.

Когда большая часть сервиса — проксирование/обогащение/фильтрация, функциональные маршруты WebFlux дают компактный и быстрый слой. Меньше магии — легче держать периметр производительности и безопасности.

Если вы строите real-time UI и не хотите поднимать Kafka/WebSocket-хаб отдельно, WebFlux-SSE — простой путь начать, а затем масштабироваться горизонтально. Совместно с gateway/ratelimit получается достаточно управляемо.

Но если у вас «толстый» CPU-сервис (например, интенсивная криптография или трансформация изображений), то поклон WebFlux сам по себе чудес не сделает. Там важнее профилирование и параллельные вычисления.

И наконец, выбирайте WebFlux, если готовы прийти к «сквозной реактивности»: HTTP-клиент, хранилище, кэш. Пытаться «встроить» JDBC внутрь реактивного контроллера — путь к блокировкам и регрессиям.

**Java — SSE как типичный use-case**

```java
package com.example.positioning;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

import java.time.Duration;

@RestController
class SseUseCaseController {

    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<String> events() {
        return Flux.interval(Duration.ofSeconds(1)).map(i -> "event-" + i);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Flux
import java.time.Duration

@RestController
class SseUseCaseController {
    @GetMapping("/events", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun events(): Flux<String> = Flux.interval(Duration.ofSeconds(1)).map { "event-$it" }
}
```

---

## Ограничения: «всё должно быть реактивным» (БД, HTTP-клиенты, файловая I/O)

Главное ограничение WebFlux — он не терпит случайных блокировок на event-loop. JDBC, файловая система, LDAP-драйверы, которые блокируют поток, не должны исполняться на этих циклах. Любая блокировка превращается в «заморозку» множества клиентов.

Чтобы взаимодействовать с неизбежно блокирующими API, используйте `Schedulers.boundedElastic()`. Это пул «для блокирующих задач», он имеет ограничение параллелизма и не даст вам «съесть» всю машину. Но применять его нужно осознанно и дозированно.

Для SQL-баз есть R2DBC — реактивный протокол и драйверы (PostgreSQL, MySQL, MSSQL и др.), которые не блокируют. В сочетании с Spring Data R2DBC вы получаете репозитории, очень похожие по стилю на JPA, но возвращающие `Mono/Flux`.

С файлами похожая история: чтение больших файлов из FS в реактивном контроллере должно происходить через неблокирующие адаптеры и кусками (`DataBuffer`), либо через offloading на `boundedElastic`. Пытаться прочитать «всё и сразу» — путь к OOM.

HTTP-клиент должен быть реактивным. `WebClient` построен на Reactor Netty и полностью неблокирующий. Старые `RestTemplate`/Apache HttpClient «как есть» не подходят: они блокируют и требуют отдельных пулов потоков.

Если у вас уже есть много библиотек «под MVC», не спешите их переносить «как есть». Часто потребуется рефакторинг контуров I/O, чтобы вынести блокирующие места в отдельные сервисы или адаптеры. Это инвестиция, но она окупается под нагрузкой.

Ограничения видны и в экосистеме: например, Bean Validation в реактивном контроллере работает иначе (асинхронно), а транзакции в R2DBC — не такие, как в JPA. Нужно ознакомиться с нюансами, чтобы не ожидать «магии из MVC».

Диагностика «случайных блокировок» помогает инструмент BlockHound: он интегрируется с Reactor и падает в тестах, если вы блокируете event-loop. Это быстрый способ дисциплинировать команду и поймать регрессии.

Не забывайте о backpressure: любой «мост» к блокирующему миру должен учитывать скорость потребителя. Иначе упрётесь в очереди на границе и получите всплески GC.

В документации сервисов явно фиксируйте, где допускается блокировка и как вы это ограничиваете. Это часть контракта производительности.

**Java — offloading блокирующего вызова + включение BlockHound в тестах**

```java
package com.example.positioning;

import reactor.blockhound.BlockHound;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

public class BlockingBridge {

    static { BlockHound.install(); } // включите в тестовом профиле

    public Mono<String> readFileBlocking(java.nio.file.Path path) {
        return Mono.fromCallable(() -> java.nio.file.Files.readString(path))
                   .subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import reactor.blockhound.BlockHound
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers
import java.nio.file.Files
import java.nio.file.Path

class BlockingBridge {
    init { BlockHound.install() } // активируйте в тестах/стендах

    fun readFileBlocking(path: Path): Mono<String> =
        Mono.fromCallable { Files.readString(path) }
            .subscribeOn(Schedulers.boundedElastic())
}
```

---

## Стек по умолчанию: Reactor, reactor-netty, codecs, WebClient

Сердце WebFlux — Project Reactor: типы `Mono/Flux`, планировщики, контекст и сотни операторов. Именно он обеспечивает backpressure и композицию. Освоение операторов — ключ к читаемому и предсказуемому коду.

На уровне транспорта по умолчанию используется Reactor Netty — неблокирующий HTTP-движок с connection pool, keep-alive и настройками тайм-аутов. Он лёгок, быстр и тесно интегрирован с Reactor, поэтому лишних адаптаций нет.

Кодеки отвечают за преобразование тел запросов и ответов: Jackson JSON, byte buffers, multipart, SSE. Через `ExchangeStrategies` вы можете настраивать пределы (`maxInMemorySize`), добавлять/заменять кодеки (protobuf/CBOR), контролировать производительность сериализации.

`WebClient` — реактивный HTTP-клиент, который отлично вписывается в стиль «агрегации». У него есть фильтры (`ExchangeFilterFunction`) для логирования, аутентификации, метрик и ретраев. Он понимает backpressure: тело читается «по запросу».

Стек включает и «инструменты разработчика»: `reactor-tools` даёт checkpoints и расширенные стэктрейсы, а `BlockHound` ловит блокировки. В связке это превращает «сложную реактивность» в управляемый конвейер.

Actuator и Micrometer интегрируются с WebFlux из коробки: вы получаете таймеры по маршрутам, теги методов/статусов, распределённый трейсинг (OTel/Zipkin). Для продакшна это критично.

На стороне безопасности есть отдельная реализация Spring Security для WebFlux: `ServerHttpSecurity`, `SecurityWebFilterChain` и реактивные конвертеры JWT (`ReactiveJwtAuthenticationConverter`). Она концептуально похожа на servlet-версию, но не делит с ней классы.

Если вы предпочитаете функциональные маршруты, используйте `RouterFunction/HandlerFunction`. Они дают минимальный оверхед и читаемый DSL. Для сложных проектов часто сочетают оба стиля.

Реактивные клиенты к БД — R2DBC, Mongo Reactive, Redis Lettuce (reactive API), Cassandra — составляют «сквозную» реактивность. Это важно, иначе «бутылочное горлышко» сделает бессмысленными плюсы стека.

Помните про осознанные пределы: буферы, размеры пулов, тайм-ауты. «По умолчанию» хорошо для старта, но в проде числа стоит откалибровать под профиль вашего трафика.

**Java — настройка WebClient и кодеков**

```java
package com.example.positioning;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.*;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import reactor.netty.http.client.HttpClient;
import org.springframework.http.codec.support.DefaultClientCodecConfigurer;

@Configuration
class ClientConfig {

    @Bean
    WebClient webClient() {
        HttpClient http = HttpClient.create()
            .responseTimeout(java.time.Duration.ofSeconds(3));
        ExchangeStrategies strategies = ExchangeStrategies.builder()
            .codecs(cfg -> {
                if (cfg instanceof DefaultClientCodecConfigurer d) {
                    d.defaultCodecs().maxInMemorySize(512 * 1024);
                }
            }).build();
        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(http))
                .exchangeStrategies(strategies)
                .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.codec.support.DefaultClientCodecConfigurer
import org.springframework.web.reactive.function.client.ExchangeStrategies
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.http.client.reactive.ReactorClientHttpConnector
import reactor.netty.http.client.HttpClient
import java.time.Duration

@Configuration
class ClientConfig {

    @Bean
    fun webClient(): WebClient {
        val http = HttpClient.create().responseTimeout(Duration.ofSeconds(3))
        val strategies = ExchangeStrategies.builder().codecs { cfg ->
            (cfg as? DefaultClientCodecConfigurer)?.defaultCodecs()?.maxInMemorySize(512 * 1024)
        }.build()
        return WebClient.builder()
            .clientConnector(ReactorClientHttpConnector(http))
            .exchangeStrategies(strategies)
            .build()
    }
}
```

---

## Совместимость: можно запускать на Netty/Undertow/Servlet-контейнере 3.1+

По умолчанию Spring Boot с `spring-boot-starter-webflux` поднимает Reactor Netty. Это самый прямой путь, где вся цепочка неблокирующая. Для большинства задач этого достаточно — он быстрый и лёгкий.

Иногда нужна интеграция с существующей инфраструктурой сервлетов 3.1+ (Tomcat/Jetty/Undertow). WebFlux умеет работать поверх сервлетов через адаптеры, сохраняя реактивный программный модель. Это полезно, если инфраструктура заточена под сервлеты.

Чтобы запустить WebFlux на сервлет-контейнере, убедитесь, что приложение стартует в режиме `reactive`: `spring.main.web-application-type=reactive`. И добавьте соответствующий стартер контейнера (например, `spring-boot-starter-tomcat`). При этом **не** подключайте MVC-стартер, иначе Boot выберет MVC-стек.

Есть и программный способ выбрать серверную платформу — задать `ReactiveWebServerFactory` бином: Netty, Undertow, Jetty. Это удобно, когда вы хотите одинаковые артефакты для разных сред, а платформу — переключать конфигом.

Помните, что «реактивно на сервлетах» ≠ «блокирую на сервлетах». Блокирующие фильтры/сервлет-фичи в цепочке могут испортить картину, даже если ваш контроллер реактивный. Внимательно проверяйте сторонние фильтры.

По наблюдаемости и метрикам различия минимальны — Micrometer и Actuator работают на обеих платформах. Но тайм-ауты/пулы/keep-alive настраиваются иначе: у Netty свои параметры, у Tomcat/Jetty — свои.

Если вы планируете WebSocket/SSE на сервлет-платформе — проверьте совместимость и нагрузку заранее. Реализации могут отличаться по поведению под бурстами.

И наконец, смешанный стек (MVC+WebFlux) на сервлетах возможен, но потребует явного разделения зон (пути/порты) и аккуратной безопасности. Иначе легко получить неожиданные редиректы/401 в API.

Для развертываний в Kubernetes выбор платформы зависит от операторских требований. Netty чаще проще: меньше настроек, меньше сюрпризов с thread-pools.

**Java — выбор платформы серверной части**

```java
package com.example.positioning;

import org.springframework.context.annotation.*;
import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory;
import org.springframework.boot.web.embedded.undertow.UndertowReactiveWebServerFactory;
import org.springframework.boot.web.reactive.server.ReactiveWebServerFactory;

@Configuration
class ServerPlatformConfig {

    // Выберите нужный бин и профильми/фичами переключайте
    @Bean
    @Profile("netty")
    ReactiveWebServerFactory netty() { return new NettyReactiveWebServerFactory(); }

    @Bean
    @Profile("undertow")
    ReactiveWebServerFactory undertow() { return new UndertowReactiveWebServerFactory(); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory
import org.springframework.boot.web.embedded.undertow.UndertowReactiveWebServerFactory
import org.springframework.boot.web.reactive.server.ReactiveWebServerFactory
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile

@Configuration
class ServerPlatformConfig {

    @Bean
    @Profile("netty")
    fun netty(): ReactiveWebServerFactory = NettyReactiveWebServerFactory()

    @Bean
    @Profile("undertow")
    fun undertow(): ReactiveWebServerFactory = UndertowReactiveWebServerFactory()
}
```

---

## Типичные use-cases: чат/нотификации (SSE/WS), API-агрегация, BFF для мобильных

Нотификации в реальном времени — один из самых частых кейсов. С SSE вы можете простым контроллером поддерживать тысячи подписок и рассылать обновления. Это удобно для дашбордов и мобильных клиентов, где важен низкий TTFB и экономия батареи.

Чаты и двусторонние каналы — территория WebSocket. В WebFlux это реализуется через хендлеры и `WebSocketHandler`, без тяжёлых библиотек. Масштабируемость достигается горизонталью и лёгкими узлами.

BFF-слой для мобильных/SPA часто агрегирует несколько REST/GraphQL источников. С `WebClient` можно параллелить, делать тайм-ауты и возвращать частичные результаты. Это улучшает UX и снижает зависимость от «самого медленного» бэкенда.

В интеграционных сервисах (edge/proxy) WebFlux хорош как «тонкий» слой трансформации: дешёвый по памяти и потокам, с метриками и трейсингом. Он выдержит бурсты трафика от партнёров.

Потоковые выгрузки (большие отчёты, экспорты) удобнее отдавать чанками или как `application/stream+json`. Клиент может начинать обработку раньше, а сервер — дозировать выделение памяти.

Push-каналы к мобильным устройствам можно реализовать как гибрид: BFF держит SSE, а критичные пуши отправляются через внешние каналы (APNS/FCM). WebFlux-узел собирает подтверждения и управление подписками.

В IoT-сценариях тысячи сенсоров пишут телеметрию небольшими порциями. WebFlux-узел как «приёмник» измерений хорошо справляется, если вы настроите пределы буферов и валидацию на входе.

Реактивные хранилища (Mongo/Cassandra) органично вписываются: поток измерений напрямую маппится в `Flux` и пишется батчами, соблюдая backpressure. Это снимает необходимость «ручных» очередей.

Edge-сервисы с простыми правилами авторизации/скоринга тоже удобно писать функциональным роутером. Здесь важна тестируемость и минимальный оверхед — сильная сторона функционального стиля.

Наконец, WebFlux хорош для внутренних API, где важны метрики/трассировки и управляемость. С Micrometer/OTel вы видите каждый этап конвейера и можете точечно оптимизировать узкие места.

**Java — простейший WS-чат-эхо и BFF-агрегация**

```java
package com.example.positioning;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.HandlerMapping;
import org.springframework.web.reactive.handler.SimpleUrlHandlerMapping;
import org.springframework.web.reactive.socket.*;
import reactor.core.publisher.Mono;

import java.util.Map;

@Configuration
class WebSocketConfig {

    @Bean
    HandlerMapping wsMapping() {
        return new SimpleUrlHandlerMapping(Map.of("/ws/echo", (WebSocketHandler) session ->
                session.send(session.receive().map(msg -> session.textMessage("echo:" + msg.getPayloadAsText())))), 10);
    }
}

```

```java
package com.example.positioning;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

record Profile(String name) {}
record Orders(int count) {}
record Screen(Profile profile, Orders orders) {}

@RestController
class BffController {
    private final WebClient client = WebClient.create("http://backend");

    @GetMapping("/screen")
    Mono<Screen> screen() {
        Mono<Profile> p = client.get().uri("/profile").retrieve().bodyToMono(Profile.class);
        Mono<Orders> o = client.get().uri("/orders").retrieve().bodyToMono(Orders.class);
        return Mono.zip(p, o).map(t -> new Screen(t.getT1(), t.getT2()));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.positioning

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.HandlerMapping
import org.springframework.web.reactive.handler.SimpleUrlHandlerMapping
import org.springframework.web.reactive.socket.WebSocketHandler
import org.springframework.web.reactive.socket.WebSocketSession
import reactor.core.publisher.Mono

@Configuration
class WebSocketConfig {

    @Bean
    fun wsMapping(): HandlerMapping =
        SimpleUrlHandlerMapping(mapOf("/ws/echo" to WebSocketHandler { session: WebSocketSession ->
            session.send(session.receive().map { session.textMessage("echo:${it.payloadAsText}") })
        }), 10)
}
```

```kotlin
package com.example.positioning

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.reactive.function.client.WebClient
import reactor.core.publisher.Mono

data class Profile(val name: String)
data class Orders(val count: Int)
data class Screen(val profile: Profile, val orders: Orders)

@RestController
class BffController {
    private val client = WebClient.create("http://backend")

    @GetMapping("/screen")
    fun screen(): Mono<Screen> {
        val p = client.get().uri("/profile").retrieve().bodyToMono(Profile::class.java)
        val o = client.get().uri("/orders").retrieve().bodyToMono(Orders::class.java)
        return Mono.zip(p, o).map { (profile, orders) -> Screen(profile, orders) }
    }
}
```

# 2. Основы Reactor (Flux/Mono)

## Модель pull-push и backpressure; холодные/горячие источники

В реактивной модели источник (Publisher) «толкает» данные (push), но делает это ровно в том объёме, который «запросил» подписчик (Subscriber) через `request(n)`. Это и есть backpressure — встроенный механизм «обратного давления», защищающий систему от переполнения очередей и непредсказуемых задержек.

Классическая синхронная коллекция работает по «pull»: потребитель сам запрашивает следующий элемент. В Reactor `Flux`/`Mono` объединяют оба мира: потребитель сигнализирует спрос (pull через `request`), а источник доставляет элементы push-событиями `onNext`. Такой контракт строго формализован в Reactive Streams.

Backpressure важен там, где производитель «быстрее» потребителя: сетевые адаптеры, файловые источники, очереди. Без него вы либо теряете данные, либо захламляете память буферами. Reactor позволяет ограничивать скорость и «подтягивать» элементы дозами — это делает задержки предсказуемыми.

«Холодные» источники (cold) генерируют данные заново на каждого подписчика. Пример — `Flux.range(1, 3)`: два подписчика получат свои независимые последовательности 1..3. Это удобно для идемпотентных источников, где воспроизведение не стоит ничего.

«Горячие» источники (hot) живут независимо от подписчиков и транслируют «текущий момент». Подписчик, подключившийся позже, не увидит прошлых элементов. Типичный пример — SDA/маркет-фид, WebSocket-канал. В Reactor горячие источники удобно реализовать через `Sinks` или `ConnectableFlux`.

Backpressure у горячих источников особенно важен: «медленный» подписчик не должен «ронять» всех. `Sinks.many().multicast().onBackpressureBuffer()` обеспечит буфер и отсеивание при переполнении — это честное поведение под нагрузкой.

Ещё одна грань — преобразование холодного в горячий через `.publish().autoConnect()`: вы «делитесь» одной подпиской вверх по цепочке между несколькими подписчиками вниз. Это сокращает нагрузку на источник (например, избегаете двойного HTTP-вызова).

В backpressure-цепях не забывайте про «места буферизации»: кодеки HTTP, сериализация JSON, агрегация чанков. Даже если ваш `Flux` ограничен `limitRate`, большой `maxInMemorySize` у кодека может неожиданно «съесть» память.

Реактивный контракт допускает и сценарий «без давления» — когда источник игнорирует `request(n)` и льёт всё сразу. В проде это нежелательно: используйте операторские «предохранители» (`limitRate`, `onBackpressureBuffer/Drop/Latest`) и тестируйте поведение под бурстами.

Проверять корректность backpressure удобнее всего юнит-тестами (`StepVerifier`) и нагрузочными сценариями: искусственно делайте потребителя медленным и смотрите, как ведут себя очереди и задержки. Это находит узкие места заранее.

Если в цепочке неизбежно есть блокирующий участок, отделяйте его `boundedElastic()`: так вы не заблокируете event-loop, а объём параллелизма будет контролируемым. Но это «мост», а не «норма» — старайтесь держать цепь полностью неблокирующей.

**Java — холодный/горячий источник и ручной backpressure**

```java
package com.example.reactor.basics;

import org.reactivestreams.Subscription;
import reactor.core.publisher.BaseSubscriber;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

public class HotColdBackpressureDemo {

    public static void main(String[] args) throws Exception {
        // cold: для каждого подписчика заново
        Flux<Integer> cold = Flux.range(1, 5);
        cold.subscribe(v -> System.out.println("cold-A: " + v));
        cold.subscribe(v -> System.out.println("cold-B: " + v));

        // hot: общий источник для всех подписчиков
        Sinks.Many<Integer> sink = Sinks.many().multicast().onBackpressureBuffer();
        Flux<Integer> hot = sink.asFlux();
        hot.subscribe(v -> System.out.println("hot-A: " + v));
        sink.tryEmitNext(1);
        sink.tryEmitNext(2);
        hot.subscribe(v -> System.out.println("hot-B: " + v)); // B увидит только 3+
        sink.tryEmitNext(3);

        // backpressure: запрашиваем по 3 элемента за раз
        Flux<Integer> fast = Flux.range(1, 10);
        fast.subscribe(new BaseSubscriber<>() {
            Subscription sub;
            int received;

            @Override
            protected void hookOnSubscribe(Subscription subscription) {
                this.sub = subscription;
                request(3);
            }

            @Override
            protected void hookOnNext(Integer value) {
                System.out.println("bp: " + value);
                if (++received % 3 == 0) {
                    request(3);
                }
            }
        });

        Thread.sleep(100);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.basics

import org.reactivestreams.Subscription
import reactor.core.publisher.BaseSubscriber
import reactor.core.publisher.Flux
import reactor.core.publisher.Sinks

object HotColdBackpressureDemo {
    @JvmStatic
    fun main(args: Array<String>) {
        val cold = Flux.range(1, 5)
        cold.subscribe { println("cold-A: $it") }
        cold.subscribe { println("cold-B: $it") }

        val sink = Sinks.many().multicast().onBackpressureBuffer<Int>()
        val hot = sink.asFlux()
        hot.subscribe { println("hot-A: $it") }
        sink.tryEmitNext(1)
        sink.tryEmitNext(2)
        hot.subscribe { println("hot-B: $it") }
        sink.tryEmitNext(3)

        val fast = Flux.range(1, 10)
        fast.subscribe(object : BaseSubscriber<Int>() {
            var sub: Subscription? = null
            var received = 0
            override fun hookOnSubscribe(subscription: Subscription) {
                sub = subscription
                request(3)
            }
            override fun hookOnNext(value: Int) {
                println("bp: $value")
                received++
                if (received % 3 == 0) request(3)
            }
        })
    }
}
```

---

## Типы: `Mono<T>` и `Flux<T>` — композиция vs коллекции

`Mono<T>` — это «0 или 1» элемент асинхронно. Он идеален для запросов «получить один объект» или «вернуть статус». В отличие от `Optional`, `Mono` умеет сигнализировать об ошибке, отменяться и участвовать в тайм-аутах/ретраях.

`Flux<T>` — это «0..N» элементов. Его удобно воспринимать как «реактивную коллекцию», но это больше: элементы приходят во времени, их можно порционировать, объединять, транслировать и отменять, не дожидаясь «полной загрузки».

Композиция — сильная сторона `Mono/Flux`: операторы `zip/merge/concat` позволяют строить конвейеры из независимых операций. Это делает код декларативным: вместо «раскиданных коллбэков» вы видите граф преобразований.

`Mono` отлично сочетается с сетевыми вызовами: `WebClient.get().retrieve().bodyToMono(User.class)` вернёт `Mono<User>`, который легко объединить с другими моно в `zip`. Вы не блокируете поток и не заводите «пулы на каждый вызов».

`Flux` удобен для «страниц», курсоров и стримов: вы можете обработать 10 элементов, а остальные — отбросить по условию (например, пользователь закрыл страницу). Блокирующий подход обычно вынуждает сначала загрузить всё.

Переход `Flux`→`Mono<List<T>>` через `collectList()` — полезный паттерн, когда на границе (например, HTTP-ответ) нужна «готовая коллекция». Главное — контролируйте размер: задайте лимиты и фильтры, чтобы не получить огромный список.

Семантика пустоты различается: `Mono.empty()` — это «нормальная пустота», а `Mono.error(e)` — это «ошибка». Для API это важно: 404 и 500 — разные истории, и реактивная цепь позволит корректно развести эти случаи.

Не стоит превращать всё подряд в `Flux`: если доменная операция возвращает один объект, используйте `Mono<T>`. Это повышает читаемость, упрощает контракты и даёт более точные сигналы об окончании/пустоте.

В обоих типах есть «терминальные» операторы (`subscribe`, `block`, `toFuture`), но в приложении WebFlux вы почти никогда сами не вызываете `subscribe` — это делает фреймворк. Вы описываете цепочку, а платформа исполняет её на нужных планировщиках.

Наконец, `Mono<Void>` — валидное значение для «нет тела, только статус». Это лучше, чем возвращать `Mono<String>` с пустой строкой: контракт становится очевиднее.

**Java — композиция `Mono`+`Flux` в сервисе**

```java
package com.example.reactor.types;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.List;

public class TypesService {

    record User(String id, String name) {}
    record Order(String id, String userId) {}
    record Screen(User user, List<Order> orders) {}

    Mono<User> loadUser(String id) {
        return Mono.just(new User(id, "Alice"));
    }

    Flux<Order> loadOrders(String userId) {
        return Flux.just(new Order("o1", userId), new Order("o2", userId));
    }

    Mono<Screen> screen(String userId) {
        return Mono.zip(loadUser(userId), loadOrders(userId).collectList())
                .map(t -> new Screen(t.getT1(), t.getT2()));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.types

import reactor.core.publisher.Flux
import reactor.core.publisher.Mono

data class User(val id: String, val name: String)
data class Order(val id: String, val userId: String)
data class Screen(val user: User, val orders: List<Order>)

class TypesService {
    fun loadUser(id: String): Mono<User> = Mono.just(User(id, "Alice"))
    fun loadOrders(userId: String): Flux<Order> = Flux.just(Order("o1", userId), Order("o2", userId))
    fun screen(userId: String): Mono<Screen> =
        Mono.zip(loadUser(userId), loadOrders(userId).collectList())
            .map { (u, orders) -> Screen(u, orders) }
}
```

---

## Операторы: `map/flatMap/concatMap/switchMap`, `error/timeout/retry`

`map` — синхронное преобразование «элемент в элемент». Он прост и безопасен: порядок сохраняется, параллельности не добавляется. Хорош для форматирования DTO, валидации, подсчёта полей.

`flatMap` раскрывает «элемент → Publisher», позволяя запускать асинхронные операции для каждого элемента. Он по умолчанию параллелит, а порядок выхода может отличаться от входного. Это мощный, но требовательный оператор: контролируйте степень параллелизма.

`concatMap` ведёт себя как «упорядоченный flatMap»: последовательно обрабатывает элементы и сохраняет порядок. Он безопаснее для внешних API, которым важна последовательность (например, запись в БД с зависимостями).

`switchMap` отменяет предыдущую ветку при приходе нового элемента. Это идеален для «живого поиска»: пользователь печатает — старые запросы отменяются, остаётся последний. В сетевых вызовах это снижает нагрузку и ускоряет отклик UI.

Ошибки в реактивном мире — это сигналы, а не исключения в стеке. Используйте `onErrorReturn`, `onErrorResume`, `onErrorMap` для «мягких» сценариев и ретраев. Так вы чётко контролируете поведение и избежите «падений» цепи.

`timeout` защищает от зависаний: если за N миллисекунд элемент не пришёл — сигнал об ошибке или переход на запасной источник (`timeout(..., fallback)`). В распределённых системах это базовая страховка.

`retry` и `retryWhen` дают повторные попытки. Но ретраи должны быть дозированными и **идемпотентными**: не повторяйте POST с побочными эффектами без гарантии безопасности. Добавляйте backoff и джиттер, чтобы не устроить «штурм» зависимого сервиса.

Комбинируя операторы, записывайте «политику»: `flatMap(...).timeout(...).retryWhen(...)`. Это читаемо и переносимо. Обязательно покрывайте тестами негативные сценарии, чтобы не гладить «зелёные ветки».

Для потоков (`Flux`) помните о `limitRate`/`flatMap(..., concurrency)`: без ограничений вы легко создадите слишком много одновременных вызовов. Параметр «одновременность» — ключевой рычаг управления.

**Java — демонстрация операторов и политики ошибок**

```java
package com.example.reactor.operators;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;

import java.time.Duration;

public class OperatorsDemo {

    static Mono<String> httpCall(String q) {
        return Mono.just("res:" + q).delayElement(Duration.ofMillis(50));
    }

    public static void main(String[] args) {
        // map / flatMap / concatMap
        Flux.just("a", "b", "c")
            .map(String::toUpperCase)
            .flatMap(q -> httpCall(q), 2) // параллелим до 2
            .timeout(Duration.ofMillis(200))
            .retryWhen(Retry.backoff(2, Duration.ofMillis(100)))
            .onErrorResume(ex -> Mono.just("fallback"))
            .subscribe(System.out::println);

        // switchMap: отменяем старые
        Flux.just("spr", "spri", "spring")
            .delayElements(Duration.ofMillis(30))
            .switchMap(OperatorsDemo::httpCall)
            .subscribe(v -> System.out.println("switchMap: " + v));

        try { Thread.sleep(500); } catch (InterruptedException ignored) {}
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.operators

import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import reactor.util.retry.Retry
import java.time.Duration

object OperatorsDemo {

    fun httpCall(q: String): Mono<String> =
        Mono.just("res:$q").delayElement(Duration.ofMillis(50))

    @JvmStatic
    fun main(args: Array<String>) {
        Flux.just("a", "b", "c")
            .map { it.uppercase() }
            .flatMap({ q -> httpCall(q) }, 2)
            .timeout(Duration.ofMillis(200))
            .retryWhen(Retry.backoff(2, Duration.ofMillis(100)))
            .onErrorResume { Mono.just("fallback") }
            .subscribe { println(it) }

        Flux.just("spr", "spri", "spring")
            .delayElements(Duration.ofMillis(30))
            .switchMap(::httpCall)
            .subscribe { println("switchMap: $it") }

        Thread.sleep(500)
    }
}
```

---

## Планировщики: `Schedulers.parallel()`/`boundedElastic()` — когда и зачем

Планировщик (Scheduler) — это «где» выполняется ваша цепь. По умолчанию Reactor использует подходящий пул для сетевых операций (event-loop). Менять его стоит только осознанно: для CPU-работы — `parallel()`, для блокирующих вызовов — `boundedElastic()`.

`subscribeOn` задаёт, где начнётся выполнение источника — «вверх по цепочке». Это полезно, когда сама «поставка» данных блокирующая (`fromCallable`, JDBC-мост). Вы уводите работу на отдельный пул, не трогая event-loop.

`publishOn` меняет планировщик «вниз по цепочке», начиная с места вызова. Так можно отделить, например, CPU-тяжёлую обработку от I/O-части: сетевой слой на event-loop, вычисления — на `parallel()`.

`boundedElastic()` — ограниченный пул для блокирующих задач. Он динамически создаёт потоки в разумных пределах и не даёт «съесть» всю машину. Это безопасный мост к миру блокировок, но злоупотреблять им нельзя — ищите реактивные альтернативы.

`parallel()` — пул фиксированного размера (по числу ядер) для CPU-работ. Он хорош для небольших чистых вычислений без блокировок. Если в нём «повесить» `Thread.sleep`, вы посадите всю машину.

Смена планировщика — это граница производительности. Каждое переключение стоит дорого: лишние планирования, переключение контекста. Старайтесь делать их осмысленно и редко, группируя операции.

С планировщиками имеет смысл работать вместе с backpressure-ограничениями (`flatMap(..., concurrency)`, `limitRate`). Без них вы легко устроите «бурю» из задач на `parallel()` и потеряете предсказуемость.

В реактивном вебе большинство цепочек должно «катиться» на event-loop. Это держит задержки низкими. Только когда «по-другому нельзя» — переключайтесь. Такой стиль дисциплинирует и делает поведение системы понятнее.

Для трассировки/логирования не забывайте, что смена планировщика «роняет» MDC/контекст — используйте контекст Reactor (о нём ниже) или автоматическую пропагацию, если требуется.

И главное: тестируйте. Планировщики отлично видны в flame-графах. Если вы видите неожиданные блокировки на event-loop — немедленно выносите их на `boundedElastic` или ищите реактивную альтернативу.

**Java — offload блокирующего вызова и CPU-обработки**

```java
package com.example.reactor.schedulers;

import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.time.Duration;

public class SchedulersDemo {

    static String blockingIo() {
        try { Thread.sleep(50); } catch (InterruptedException ignored) {}
        return "io";
    }

    public static void main(String[] args) {
        Mono.fromCallable(SchedulersDemo::blockingIo)
            .subscribeOn(Schedulers.boundedElastic())   // уводим блокировку
            .map(v -> v + "-mapped")
            .publishOn(Schedulers.parallel())           // CPU-последовательность
            .map(String::toUpperCase)
            .timeout(Duration.ofMillis(500))
            .subscribe(System.out::println);

        try { Thread.sleep(200); } catch (InterruptedException ignored) {}
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.schedulers

import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers
import java.time.Duration

object SchedulersDemo {

    fun blockingIo(): String {
        Thread.sleep(50)
        return "io"
    }

    @JvmStatic
    fun main(args: Array<String>) {
        Mono.fromCallable(::blockingIo)
            .subscribeOn(Schedulers.boundedElastic())
            .map { "$it-mapped" }
            .publishOn(Schedulers.parallel())
            .map { it.uppercase() }
            .timeout(Duration.ofMillis(500))
            .subscribe { println(it) }

        Thread.sleep(200)
    }
}
```

---

## Context и MDC: перенос корреляционных айди и безопасности

Reactor Context — это сопутствующие метаданные, идущие вместе с сигналами: корреляционный ID, tenant, user-id, флаги безопасности. Он «течёт» по цепи без привязки к потокам, в отличие от `ThreadLocal`/MDC.

Контекст задаётся через `contextWrite(ctx -> ctx.put("cid", ...))` и читается через `deferContextual`. Это удобный способ «провести» корреляционный ID от входа (HTTP-запрос, gRPC-метаданные) до логов и исходящих вызовов.

Так как Reactor часто меняет потоки (event-loop, `parallel`, `boundedElastic`), обычный MDC от SLF4J «теряется». Либо используйте расширения для автоматической пропагации, либо вручную прокладывайте CID в лог-вызовах через `deferContextual`.

Входной кор-ID считывайте из заголовков (`X-Request-Id`, `traceparent`) на самом краю (WebFilter/HandlerFilterFunction) и кладите в контекст. Далее любой сервисный метод сможет его прочитать и добавить в outbound-заголовки.

Безопасность также переносится через контекст: реактивный Spring Security хранит `Authentication` в `ReactiveSecurityContextHolder`. При необходимости вы можете дополнительно продублировать нужные атрибуты (например, tenant) в Reactor Context для удобства SpEL/логов.

Для логов можно временно «закинуть» CID в MDC на время обработки элемента и убрать затем. Делайте это аккуратно в `doOnEach` и обязательно удаляйте ключ в `doFinally`, чтобы не «утекал» между запросами.

Контекст — неизменяемый, каждое `put` создаёт новую «ветку» вниз по цепи. Это безопасно для параллельного исполнения и повышает предсказуемость; просто держите это в уме при сложных комбинациях.

При тестировании контекст легко подложить через `contextWrite`, а затем утверждать, что outbound-клиент положил CID в заголовки. Это делает перенос метаданных воспроизводимым и проверяемым.

Не храните в контексте «тяжёлое» (большие DTO). Это вспомогательные метки, а не кэш. Большие объекты давят на GC и усложняют трассировку.

И наконец, не путайте Reactor Context с HTTP-атрибутами. Это универсальный механизм, который работает и в чистых сервисных слоях, и в клиентских цепочках.

**Java — перенос correlationId через Context и логирование**

```java
package com.example.reactor.context;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import reactor.core.publisher.Mono;

public class ContextDemo {
    private static final Logger log = LoggerFactory.getLogger(ContextDemo.class);

    public static void main(String[] args) {
        Mono.deferContextual(ctx -> {
                String cid = ctx.getOrDefault("cid", "n/a");
                // временно кладём в MDC на время обработки
                try {
                    MDC.put("cid", cid);
                    log.info("Handling with cid={}", cid);
                    return Mono.just("ok");
                } finally {
                    MDC.remove("cid");
                }
            })
            .contextWrite(ctx -> ctx.put("cid", "req-123"))
            .subscribe(v -> log.info("Done: {}", v));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.context

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import reactor.core.publisher.Mono

object ContextDemo {
    private val log = LoggerFactory.getLogger(ContextDemo::class.java)

    @JvmStatic
    fun main(args: Array<String>) {
        Mono.deferContextual { ctx ->
            val cid = ctx.getOrDefault("cid", "n/a")
            try {
                MDC.put("cid", cid)
                log.info("Handling with cid={}", cid)
                Mono.just("ok")
            } finally {
                MDC.remove("cid")
            }
        }
            .contextWrite { it.put("cid", "req-123") }
            .subscribe { log.info("Done: {}", it) }
    }
}
```

---

## Диагностика: `log()`, checkpoints, Hooks для улучшения стэктрейсов

Оператор `log()` печатает жизненный цикл последовательности: `onSubscribe`, `request(n)`, `onNext`, `onComplete/onError`. Это быстрый способ понять, где «застряли» сигналы и как течёт backpressure. Используйте его точечно, иначе зашумите логи.

`checkpoint("name")` вставляет маркер в стек выполнения. Если далее произойдёт ошибка, в стэктрейсе будет видно, через какой участок цепочки она прошла. Это резко улучшает отладку сложных конвейеров.

В проде «развёрнутые» стэктрейсы дорогие. В dev/test включайте расширенную диагностику, в проде — оставляйте только метрики и редкие маркеры. `checkpoint` можно пометить как «force stack capture», но помните о цене.

Библиотека `reactor-tools` добавляет агент `ReactorDebugAgent`, который улучшает стэктрейсы, «прошивая» операторы в байткод. Инициализация в `main` делает все цепочки значительно понятнее при ошибках.

Для сетевых сценариев полезно сочетать `log()` с HTTP-логированием (WebClient filters). Вы получите связную картину «кто позвал кого», где именно отвалился тайм-аут и какая была реакция ретраев.

Диагностика — это ещё и тесты. Используйте `StepVerifier` для воспроизведения сложной последовательности: задержки, ошибки, отмены. Это даёт регрессионную страховку от «незаметных» поломок.

Если у вас «вдруг блокируется» цепь, включайте BlockHound (в тестах/стендах). Он бросит ошибку при попытке блокировки в event-loop и покажет, где именно вы «промахнулись» с планировщиком.

Для длинных цепочек раскладывайте конвейер на именованные методы и вставляйте `checkpoint` на стыках. Это повышает читабельность и ускоряет поиск дефектов.

Помните про объём логов. В реактивных сервисах RPS велик, и «наивное» логирование каждого `onNext` может утопить диск. Логируйте на выборке, сэмплируйте, используйте кор-ID для корреляции.

И наконец, держите «режим диагностики» фичей или профилем. Возможность быстро включить расширенные стэктрейсы на стенде и выключить в проде — большой плюс к эксплуатационной готовности.

**Java — `log()`, `checkpoint()` и ReactorDebugAgent**

```java
package com.example.reactor.debug;

import reactor.core.publisher.Flux;
import reactor.tools.agent.ReactorDebugAgent;

import java.time.Duration;

public class DebugDemo {

    public static void main(String[] args) throws Exception {
        ReactorDebugAgent.init();
        ReactorDebugAgent.processExistingClasses();

        Flux.range(1, 5)
            .map(i -> 10 / (i - 3)) // деление на ноль при i=3
            .checkpoint("after-map")
            .delayElements(Duration.ofMillis(10))
            .log("seq")
            .subscribe(
                v -> System.out.println("v=" + v),
                ex -> ex.printStackTrace()
            );

        Thread.sleep(200);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.debug

import reactor.core.publisher.Flux
import reactor.tools.agent.ReactorDebugAgent
import java.time.Duration

object DebugDemo {
    @JvmStatic
    fun main(args: Array<String>) {
        ReactorDebugAgent.init()
        ReactorDebugAgent.processExistingClasses()

        Flux.range(1, 5)
            .map { 10 / (it - 3) }
            .checkpoint("after-map")
            .delayElements(Duration.ofMillis(10))
            .log("seq")
            .subscribe({ println("v=$it") }, { it.printStackTrace() })

        Thread.sleep(200)
    }
}
```

# 3. Архитектура WebFlux

## `HttpHandler` → `WebFlux` → `DispatcherHandler` (аннотации) или Router/Handler (функционально)

Архитектурно WebFlux строится вокруг минимального контракта `HttpHandler`: это функция «запрос → реактивный ответ». Конкретная серверная платформа (Netty/Undertow/Servlet 3.1+) поставляет адаптер, который умеет принимать соединения и делегировать их `HttpHandler`. Таким образом, ядро не зависит от движка, а движок не знает о ваших контроллерах.

Над `HttpHandler` находится слой Spring WebFlux — инфраструктура маршрутизации, кодеки, конвертеры, резолверы аргументов. Если вы выбираете аннотационный стиль, входная точка — `DispatcherHandler`: это «реактивный брат» `DispatcherServlet`, который, опираясь на маппинги, вызывает нужный контроллер и конвертирует вход/выход.

Аннотационный подход удобен знакомостью: `@RestController`, `@GetMapping`, валидация и `ResponseEntity` — всё похоже на MVC, но типы возвращаемых значений реактивные (`Mono`/`Flux`). Это снижает стоимость миграции команд, привыкших к MVC.

Функциональный стиль обходит рефлексию и аннотации. Вы явно объявляете `RouterFunction<ServerResponse>` и «чистые» обработчики `HandlerFunction<ServerResponse>`. Такой код проще читать как таблицу маршрутов, он минималистичен и даёт чуть меньший оверхед.

Выбор стиля — не взаимоисключающий. В одном приложении можно держать и аннотационные контроллеры для сложной декларативной логики, и функциональные обработчики для тонких, производительных участков или «интеграционных» маршрутов.

`DispatcherHandler` работает реактивно от начала до конца: аргументы метода контроллера резолвятся из потокового тела, результат контроллера — тоже Publisher, который сериализуется по мере готовности. Это и делает возможным стриминг без буферизации всего ответа.

При функциональной маршрутизации в игру вступает `RouterFunctions`: вы строите дерево маршрутов (`nest`, `resources`, предикаты метода/пути/заголовков) и получаете полный контроль над порядком проверки и фильтрами.

«Фильтрация» запроса и сквозные аспекты реализуются через `WebFilter` (аннотационный стиль) или `HandlerFilterFunction` (функциональный стиль). Оба механизма реактивные, и в них можно аккуратно управлять `Context`/MDC и backpressure.

Интеграция с валидацией и обработкой ошибок работает в обоих стилях: `@ControllerAdvice` в аннотационном и функциональный «глобальный» фильтр/роут для исключений — в функциональном. Это позволяет унифицировать ответы и логирование.

Понимание этого стека помогает точечно оптимизировать: если у вас много маршрутов и простая логика — функциональный стиль ускорит холодный старт и маршрутизацию; если богатая «магия» Spring — аннотации дадут гибкость и знакомые расширения.

**Java — два стиля рядом (аннотационный и функциональный)**

```java
package com.example.arch;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

@RestController
class HelloController {
    @GetMapping(value = "/api/hello", produces = MediaType.TEXT_PLAIN_VALUE)
    Mono<String> hello() { return Mono.just("hello-annot"); }
}

@Component
class HelloHandler {
    Mono<ServerResponse> hello(ServerRequest req) {
        return ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("hello-func");
    }
}

@Configuration
class Routes {
    @Bean
    RouterFunction<ServerResponse> helloRoutes(HelloHandler h) {
        return RouterFunctions.route(RequestPredicates.GET("/func/hello"), h::hello);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.stereotype.Component
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.reactive.function.server.*
import reactor.core.publisher.Mono

@RestController
class HelloController {
    @GetMapping("/api/hello", produces = [MediaType.TEXT_PLAIN_VALUE])
    fun hello(): Mono<String> = Mono.just("hello-annot")
}

@Component
class HelloHandler {
    fun hello(req: ServerRequest): Mono<ServerResponse> =
        ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("hello-func")
}

@Configuration
class Routes {
    @Bean
    fun helloRoutes(h: HelloHandler): RouterFunction<ServerResponse> =
        RouterFunctions.route(RequestPredicates.GET("/func/hello"), h::hello)
}
```

---

## Серверная платформа: Reactor Netty, event-loop, non-blocking сокеты

Реализуя HTTP-сервер, WebFlux по умолчанию использует Reactor Netty — тонкую оболочку над Netty с удобным реактивным API. Netty построен на неблокирующих сокетах и мультиплексировании событий (NIO/epoll/kqueue), что позволяет несколькими потоками обслуживать множество соединений.

Модель потоков Netty — это event-loop’ы: небольшой набор «рабочих» потоков крутит селекторы и обрабатывает события. В отличие от «поток на запрос» здесь важна дисциплина — на этих потоках нельзя блокироваться, иначе вы «заморозите» многих клиентов сразу.

Reactor Netty предоставляет удобный способ настройки тайм-аутов, keep-alive, размеров пулов и TCP-опций. Через `HttpServer`/`HttpClient` можно аккуратно задавать idle-timeout и лимиты заголовков/URI, а через Boot — свойства server.netty.*

В режиме Servlet 3.1+ WebFlux использует «асинхронные сервлеты» и неблокирующие каналы, но часть экосистемы может случайно внести блокировки. Поэтому если нужна максимальная предсказуемость, Netty — предпочтительный выбор.

Важно понимать различие времени жизни соединения и запроса. Под долгоживущими потоками SSE/WS Netty держит канал открытым, а ваш реактивный граф выдаёт элементы по мере готовности. Это почти ничего не стоит event-loop’ам, пока вы не блокируетесь.

Настройка количества event-loop’ов обычно оставляется по умолчанию (число CPU). Ручное увеличение редко даёт пользу, а вот уменьшение под «узкое железо» может экономить память. Главный рычаг — ограничение параллелизма в бизнес-цепях, а не количество event-loop.

Reactor Netty логирует соединения, закрытия, timeouts. В проде полезно включить умеренный DEBUG на время инцидента, чтобы понять, где тратится время — в чтении, записи, ожидании.

Шифрование (TLS) также выполняется на event-loop’ах. Длинные рукопожатия или медленные сертификаты могут временно занять цикл — контролируйте тайм-ауты и кэшируйте цепочки сертификатов.

Сетевые буферы — ещё один фактор. Слишком большие — риск OOM под бурстом; слишком маленькие — ухудшение пропускной способности. Настраивайте осознанно и вместе с кодеками сериализации.

Наконец, при горизонтальном масштабировании следите за «свипом» keep-alive: idle-timeouts должны быть согласованы между балансировщиком и сервисом, чтобы не получить лавину SYN/FIN в пиках.

**Java — тонкая настройка Reactor Netty-сервера**

```java
package com.example.arch;

import io.netty.channel.ChannelOption;
import reactor.netty.http.server.HttpServer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;

import java.time.Duration;

@Configuration
class NettyTuning {

    @Bean
    WebServerFactoryCustomizer<NettyReactiveWebServerFactory> nettyCustomizer() {
        return factory -> factory.addServerCustomizers(http -> {
            HttpServer server = http
                .idleTimeout(Duration.ofSeconds(30))
                .compress(true)
                .option(ChannelOption.SO_BACKLOG, 1024);
            return server;
        });
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch

import io.netty.channel.ChannelOption
import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory
import org.springframework.boot.web.server.WebServerFactoryCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import reactor.netty.http.server.HttpServer
import java.time.Duration

@Configuration
class NettyTuning {

    @Bean
    fun nettyCustomizer(): WebServerFactoryCustomizer<NettyReactiveWebServerFactory> =
        WebServerFactoryCustomizer { factory ->
            factory.addServerCustomizers { http: HttpServer ->
                http.idleTimeout(Duration.ofSeconds(30))
                    .compress(true)
                    .option(ChannelOption.SO_BACKLOG, 1024)
            }
        }
}
```

---

## Кодеки: Jackson/JSON, protobuf, byte buffers; ручной контроль `DataBuffer`

Кодеки (`HttpMessageReader/Writer`) выполняют преобразование тел запросов/ответов. По умолчанию активирован JSON через Jackson, а также ряд «бинарных» и текстовых конвертеров. Вы можете менять лимиты памяти, набор кодеков и сериализаторов через `ExchangeStrategies`.

Если у вас потоковые ответы (`application/stream+json`), важно контролировать «агрегацию»: не собирайте весь `Flux` в память, позволяйте кодеку писать чанки по мере готовности. Ограничение `maxInMemorySize` защитит от огромных объектов.

Для бинарных форматов (protobuf/CBOR) добавляются соответствующие кодеки. Важно, чтобы они тоже поддерживали «поштучную» запись элементов. Иначе смысл реактивности теряется при сериализации.

Иногда нужно работать с телом на уровне байтов. В WebFlux для этого есть `DataBuffer`/`DataBufferUtils`: вы можете читать/писать чанки, контролировать выделение и высвобождение буферов. Это повышает производительность при файлах/стримах.

«Ручной» контроль буферов требует дисциплины: забытый `DataBufferUtils.release` может привести к утечке. Используйте готовые утилиты копирования (`write`, `read`) и закрывайте ресурсы в `doFinally`.

Настройку Jackson (модули, `ObjectMapper`, включение/выключение фич) можно сделать через `CodecCustomizer` или `Jackson2ObjectMapperBuilderCustomizer`. Это влияет на производительность и совместимость API.

Для ошибок сериализации держите единый обработчик — возвращайте `Problem Details`/JSON, а не «сырой 500». Кодеки умеют сигнализировать об ошибках — ловите их и маршаллизуйте.

В «гигабайтных» ответах (архивы/экспорты) отдавайте поток байтов (`BodyInserters.fromPublisher`) и явно ставьте `Content-Type`/`Content-Length` (если известна) или chunked. Это избавляет от буферизации и ускоряет клиента.

Если нужен TTFB как можно быстрее, настройте Jackson на «меньше форматирования», избегайте больших вложенных структур, отдавайте элементы по мере подготовки, а не «списком».

И наконец, помните про безопасность: кодеки — часть поверхности атаки. Ограничивайте размеры тел, глубину JSON, проверяйте типы и используйте безопасные десериализаторы.

**Java — настройка кодеков и работа с DataBuffer**

```java
package com.example.arch;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.http.codec.CodecConfigurer;
import org.springframework.http.codec.ServerCodecConfigurer;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Flux;

import java.nio.charset.StandardCharsets;

@Configuration
class CodecConfig {

    @Bean
    ServerCodecConfigurer serverCodecConfigurer() {
        return new ServerCodecConfigurer() {
            final ServerCodecConfigurer delegate = ServerCodecConfigurer.create();
            @Override public void defaultCodecs(CodecConfigurer.DefaultCodecs consumer) {
                delegate.defaultCodecs().maxInMemorySize(512 * 1024);
            }
            @Override public void customCodecs(CodecConfigurer.CustomCodecs customCodecs) {
                delegate.customCodecs().register(new org.springframework.http.codec.StringEncoder(StandardCharsets.UTF_8));
            }
            @Override public CodecConfigurer.DefaultCodecs defaultCodecs() { return delegate.defaultCodecs(); }
            @Override public CodecConfigurer.CustomCodecs customCodecs() { return delegate.customCodecs(); }
        };
    }

    @Bean
    RouterFunction<ServerResponse> streamBytesRoute() {
        Flux<byte[]> bytes = Flux.range(0, 5).map(i -> ("part-" + i + "\n").getBytes(StandardCharsets.UTF_8));
        return RouterFunctions.route(RequestPredicates.GET("/stream/bytes"),
            req -> ServerResponse.ok().contentType(MediaType.APPLICATION_OCTET_STREAM)
                    .body(BodyInserters.fromPublisher(bytes, byte[].class)));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.http.codec.CodecConfigurer
import org.springframework.http.codec.ServerCodecConfigurer
import org.springframework.web.reactive.function.BodyInserters
import org.springframework.web.reactive.function.server.RouterFunction
import org.springframework.web.reactive.function.server.RouterFunctions
import org.springframework.web.reactive.function.server.ServerRequest
import org.springframework.web.reactive.function.server.ServerResponse
import reactor.core.publisher.Flux
import java.nio.charset.StandardCharsets

@Configuration
class CodecConfig {

    @Bean
    fun serverCodecConfigurer(): ServerCodecConfigurer {
        val delegate = ServerCodecConfigurer.create()
        delegate.defaultCodecs().maxInMemorySize(512 * 1024)
        return delegate
    }

    @Bean
    fun streamBytesRoute(): RouterFunction<ServerResponse> {
        val bytes: Flux<ByteArray> = Flux.range(0, 5).map { "part-$it\n".toByteArray(StandardCharsets.UTF_8) }
        return RouterFunctions.route({ req: ServerRequest -> req.path() == "/stream/bytes" }) { _ ->
            ServerResponse.ok().contentType(MediaType.APPLICATION_OCTET_STREAM)
                .body(BodyInserters.fromPublisher(bytes, ByteArray::class.java))
        }
    }
}
```

---

## Потоки ответов: стриминг, chunked, `ServerSentEvent`

Стриминг — естественный режим для WebFlux. Когда тело ответа — `Flux`, кодек пишет элементы по мере их появления, не собирая в память весь результат. Это снижает задержку до первого байта и выравнивает нагрузку.

Chunked-передача (`Transfer-Encoding: chunked`) включается автоматически, когда размер ответа заранее неизвестен. Клиент начинает получать чанки сразу, а коннект живёт столько, сколько выдаёт ваш `Flux`.

Формат `application/stream+json` — это последовательность JSON-элементов, разделённых переводами строк/другими разделителями. Он удобен для потоковой обработки на фронте и совместим с большинством парсеров.

Server-Sent Events (SSE) — стандартный текстовый поток событий. В WebFlux ему соответствует `MediaType.TEXT_EVENT_STREAM` и удобный тип `ServerSentEvent<T>`, позволяющий добавлять id/тип/retry-интервалы.

SSE хорош для односторонних нотификаций, где клиентом часто является браузер. Соединение держится долго, сервер шлёт события, а браузер сам переподключается при обрыве (согласно `retry`).

Для двусторонней связи уместнее WebSocket: протокол экономит заголовки и даёт полнодуплексный канал. Но для многих UI-сценариев SSE проще: один HTTP-поток и минимум инфраструктуры.

В потоковых ответах особенно важны ограничители: если источник быстрее потребителя, используйте `limitRate`, `delayElements` или серверные «шлюзы» событий. Иначе вы рискуете заполнить TCP-буферы и память клиента.

Логируйте потоковые маршруты аккуратно: `log()` на каждом `onNext` при 1000 RPS быстро превратит логи в «белый шум». Сэмплирование и кор-ID — ваш выбор.

Клиентам, которые потребляют поток, объясняйте контракты: как отличить heartbeat от данных, где границы элементов, как обрабатывать reconnect. Это часть DX и эксплуатационной стабильности.

И наконец, тестируйте стримы `WebTestClient`-ом с `returnResult()` и `StepVerifier`: вы сможете читать поток порциями и делать утверждения на первые N элементов/временные окна.

**Java — SSE и stream+json**

```java
package com.example.arch;

import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.time.Duration;

@RestController
class StreamController {

    @GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<ServerSentEvent<String>> sse() {
        return Flux.interval(Duration.ofSeconds(1))
                .map(i -> ServerSentEvent.builder("tick-" + i).id(Long.toString(i)).build());
    }

    @GetMapping(value = "/stream", produces = "application/stream+json")
    Flux<String> streamJson() {
        return Flux.interval(Duration.ofMillis(200)).map(i -> "{\"n\":" + i + "}");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch

import org.springframework.http.MediaType
import org.springframework.http.codec.ServerSentEvent
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Flux
import java.time.Duration

@RestController
class StreamController {

    @GetMapping("/sse", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun sse(): Flux<ServerSentEvent<String>> =
        Flux.interval(Duration.ofSeconds(1))
            .map { ServerSentEvent.builder("tick-$it").id("$it").build() }

    @GetMapping("/stream", produces = ["application/stream+json"])
    fun streamJson(): Flux<String> =
        Flux.interval(Duration.ofMillis(200)).map { """{"n":$it}""" }
}
```

---

## Модель блокировок: где нельзя блокировать и как оградить `boundedElastic`

Ключевое правило — не блокировать event-loop. Любой `Thread.sleep`, JDBC-вызов, блокирующий ввод/вывод на этих потоках остановит обработку многих соединений. Отсюда вытекает стратегия: «всё реактивное» или «мосты к блокирующему миру» на `boundedElastic`.

`boundedElastic()` — специальный пул для блокирующих задач. Он ограничен по размеру и растёт по мере необходимости, предотвращая «смерть от тысяч потоков». Если без блокировки никак, выносите фрагмент через `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`.

Типичные блокирующие места — файловая система, JDBC, старые HTTP-клиенты, криптография без NIO. Для многих есть реактивные альтернативы (R2DBC, WebClient, async FS), и ими стоит пользоваться по умолчанию.

Нельзя забывать и про «скрытые» блокировки: логеры, форматирование больших объектов, синхронизированные коллекции. Они не I/O, но могут «держать» поток достаточно долго, чтобы создать очереди.

BlockHound помогает обнаруживать блокировки рантаймом: в тестах/стендах он бросит ошибку при попытке заблокировать event-loop. Подключайте его рано — он дисциплинирует архитектуру.

Ограждение «мостов» — не только `subscribeOn`, но и ограничение параллелизма: `.flatMap(call, /*concurrency*/ 8)` и backpressure. Иначе вы вынесете блокировки, но создадите 1000 одновременных задач и свалите пул.

Не превращайте `boundedElastic` в «свалку». Определяйте чёткие слои: адаптер к блокирующему миру и тонкие сервисы над ним. Так вы сможете позже заменить их на реактивные доставки без каскадного рефакторинга.

Чётко документируйте, что блокирует. Новички часто добавляют в конвейер «маленький» вызов, а под нагрузкой именно он и становиться бутылочным горлышком. Архитектурные гайды должны это отражать.

Если нужно перейти к реактивной БД, но сейчас только JDBC — выделите отдельный сервис-агрегатор и общайтесь с ним по реактивному HTTP. Пусть он блокируется внутри себя, а WebFlux-узлы останутся чистыми.

Ну и главное — измеряйте. Профили и flame-графы покажут, где реально тратите время. Интуиция в реактивных системах часто подводит.

**Java — «мост» к блокирующему JDBC в отдельном слое**

```java
package com.example.arch;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;

@Service
class BlockingRepo {

    Mono<Integer> countUsers() {
        return Mono.fromCallable(() -> {
            try (Connection c = DriverManager.getConnection("jdbc:h2:mem:test");
                 var st = c.createStatement();
                 ResultSet rs = st.executeQuery("select 42 as n")) {
                rs.next();
                return rs.getInt("n");
            }
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch

import org.springframework.stereotype.Service
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers
import java.sql.DriverManager

@Service
class BlockingRepo {
    fun countUsers(): Mono<Int> =
        Mono.fromCallable {
            DriverManager.getConnection("jdbc:h2:mem:test").use { c ->
                c.createStatement().use { st ->
                    st.executeQuery("select 42 as n").use { rs ->
                        rs.next(); rs.getInt("n")
                    }
                }
            }
        }.subscribeOn(Schedulers.boundedElastic())
}
```

---

## Ограничения памяти/очередей: настройка серверных буферов и timeouts

Реактивность не отменяет законы памяти. Буферы и очереди есть на каждом уровне: TCP, Netty, кодеки, ваши коллекции. Их размеры нужно настраивать под профиль трафика, иначе под бурстами вы упадёте в OOM или начнёте «задушивать» клиентов.

Главный защитный механизм — лимиты размеров тел (`maxInMemorySize` для кодеков), лимиты на заголовки/строку запроса, idle/read timeouts. На входе это защищает от «гигантских» запросов, на выходе — от «безбрежного» стриминга.

Backpressure — ваш друг: `limitRate`, «мягкие» буферы, оконные операторы. Вместе с тайм-аутами на исходящих вызовах это предотвращает накопление незавершённых операций и «снежный ком».

Reactor Netty даёт `idleTimeout`/`responseTimeout` и тюнинг каналов. Правильно настроенные тайм-ауты — это не «строгость», а безопасность: они освобождают ресурсы и дают быстрый фейл, который можно ретраить.

В chunked/stream сценариях контролируйте «глубину» сериализации: большой `maxInMemorySize` может привести к агрегации за пределами вашего внимания. Снижайте его и, если нужно, делайте собственные `BodyInserter`.

На уровне приложения избегайте «собрать всё в List» без верхних ограничений. Если граница нужна (например, на внешнем API) — используйте пагинацию/страницы и агрегируйте только «разрешённый максимум».

При обработке файлов и бинарных потоков применяйте «копирование по частям» и освобождение буферов (`DataBufferUtils.release`), если выходите на низкий уровень. Иначе лики могут проявляться только под нагрузкой.

Наблюдайте за GC/метриками памяти. Micrometer/Actuator помогут увидеть рост хипа и частоту фулл-GC при бурстах. Это часто симптом избыточной буферизации или некорректных лимитов.

Согласовывайте тайм-ауты с балансировщиками/ingress. Если у вас `idleTimeout=30s`, а на периметре — 15s, то клиенты будут видеть неожиданные обрывы. Единая политика времени — часть SLO.

И наконец, автоматизируйте «санити-чек» лимитов на CI/стендах: простые smoke-тесты, дергающие крупные тела/бурсты, быстро выявляют регрессии конфигурации.

**application.yml — базовые лимиты и тайм-ауты**

```yaml
spring:
  codec:
    max-in-memory-size: 512KB
server:
  reactive:
    session:
      timeout: 30s
  netty:
    idle-timeout: 30s
```

**Java — настройка клиентских/серверных тайм-аутов и ограничителей**

```java
package com.example.arch;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.*;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;

@Configuration
class TimeoutsAndLimits {

    @Bean
    WebClient tunedClient() {
        HttpClient http = HttpClient.create()
            .responseTimeout(Duration.ofSeconds(2))
            .compress(true);
        return WebClient.builder()
            .clientConnector(new org.springframework.http.client.reactive.ReactorClientHttpConnector(http))
            .filter(ExchangeFilterFunction.ofRequestProcessor(req -> {
                // пример: принудительный timeout на уровне operator
                return Mono.just(req);
            }))
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.client.reactive.ReactorClientHttpConnector
import org.springframework.web.reactive.function.client.ExchangeFilterFunction
import org.springframework.web.reactive.function.client.WebClient
import reactor.netty.http.client.HttpClient
import java.time.Duration

@Configuration
class TimeoutsAndLimits {

    @Bean
    fun tunedClient(): WebClient {
        val http = HttpClient.create()
            .responseTimeout(Duration.ofSeconds(2))
            .compress(true)
        return WebClient.builder()
            .clientConnector(ReactorClientHttpConnector(http))
            .filter(ExchangeFilterFunction.ofRequestProcessor { Mono.just(it) })
            .build()
    }
}
```

# 4. Аннотационные контроллеры (MVC-стиль в реактивном мире)

## `@RestController`, `@GetMapping`/`@PostMapping` c `Mono/Flux` телами

Реактивные аннотационные контроллеры в WebFlux внешне выглядят так же, как и в Spring MVC, но возвращают не готовые объекты, а `Mono<T>` или `Flux<T>`. Это не «причуда», а способ выразить отложенное вычисление и позволить фреймворку управлять потоком данных, тайм-аутами и отменой работы при закрытии соединения.

Контракт прост: если операция возвращает один результат — `Mono<T>`, если много — `Flux<T>`. На границе HTTP WebFlux сам сериализует элементы по мере готовности, не блокируя event-loop. Это особенно хорошо видно при стриминге, где вы получаете TTFB почти сразу.

Входное тело запроса также может быть реактивным. Для «обычного» JSON-запроса вполне уместно принимать `Mono<Dto>` — фреймворк распарсит тело, а вы останетесь в реактивной модели от начала до конца. Это дешевле, чем сначала блокирующе считать всё тело в память.

Почти всегда лучше возвращать `Mono<ResponseEntity<T>>`, когда нужно управлять кодами статуса/заголовками. Возврат `ResponseEntity<Mono<T>>` в WebFlux не поддерживается — реактивность должна быть «снаружи», чтобы кодеки понимали, когда закрывать ответ.

Для чтения query-параметров и заголовков всё привычно: `@RequestParam`, `@RequestHeader`, `@PathVariable`. Разница лишь в том, что метод не должен блокироваться — любые тяжёлые I/O переносите на `boundedElastic`.

Списки и страницы лучше возвращать как `Flux<T>` и лишь на самом краю (если контракт требует) собирать в `Mono<List<T>>` через `collectList()`. Это даёт вам контроль над памятью и возможность досрочно прервать обработку.

Отмена («клиент закрыл соединение») в реактивной модели приводит к сигналу `cancel`. Если вы используете внешние вызовы (`WebClient`) — они тоже будут отменены; это экономит ресурсы бэкендов.

Обработка ошибок — это не `try/catch` вокруг метода, а сигнал `onError`. Генерируйте `ResponseStatusException` или свои исключения — их перехватит `@ControllerAdvice` и вернёт согласованный ответ.

Не используйте `block()` в контроллерах. Это заблокирует event-loop и приведёт к «заморозке» многих клиентов. Если без блокирующего вызова никак — вынесите в отдельный адаптер и `subscribeOn(Schedulers.boundedElastic())`.

Наконец, помните о контракте сериализации: большие ответы логичнее стримить (`Flux<T>`), а если клиенту нужна «цельная» структура — ограничьте максимальные размеры и контролируйте `maxInMemorySize` кодеков.

**Java — базовые GET/POST с `Mono/Flux`**

```java
package com.example.webflux.anno;

import jakarta.validation.Valid;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.*;

import java.net.URI;

record User(String id, String name) {}
record CreateUser(String name) {}

@RestController
@RequestMapping("/api/users")
class UserController {

    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    Mono<ResponseEntity<User>> get(@PathVariable String id) {
        return Mono.just(new User(id, "Alice"))
                   .map(ResponseEntity::ok);
    }

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    Flux<User> all() {
        return Flux.just(new User("1", "Alice"), new User("2", "Bob"));
    }

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    Mono<ResponseEntity<Void>> create(@Valid @RequestBody Mono<CreateUser> in) {
        return in.map(dto -> "3") // имитация save -> id
                 .map(id -> ResponseEntity.created(URI.create("/api/users/" + id)).build());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.anno

import jakarta.validation.Valid
import org.springframework.http.MediaType
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import java.net.URI

data class User(val id: String, val name: String)
data class CreateUser(val name: String)

@RestController
@RequestMapping("/api/users")
class UserController {

    @GetMapping("/{id}", produces = [MediaType.APPLICATION_JSON_VALUE])
    fun get(@PathVariable id: String): Mono<ResponseEntity<User>> =
        Mono.just(User(id, "Alice")).map { ResponseEntity.ok(it) }

    @GetMapping(produces = [MediaType.APPLICATION_JSON_VALUE])
    fun all(): Flux<User> = Flux.just(User("1", "Alice"), User("2", "Bob"))

    @PostMapping(consumes = [MediaType.APPLICATION_JSON_VALUE])
    fun create(@Valid @RequestBody inDto: Mono<CreateUser>): Mono<ResponseEntity<Void>> =
        inDto.map { "3" }.map { ResponseEntity.created(URI.create("/api/users/$it")).build() }
}
```

---

## Валидация: Bean Validation в реактивном пайплайне, `@Valid` и обработчики ошибок

В WebFlux аннотации Bean Validation (`@NotBlank`, `@Size`, `@Email` и т.д.) работают и для «реактивных» тел. Если параметр метода — `@RequestBody Mono<Dto>`, то валидатор проверит элемент, когда он будет десериализован, и при нарушении правил бросит `WebExchangeBindException`.

Классический `BindingResult` соседствующий с `@RequestBody` в реактивном мире недоступен: тело читается асинхронно, и «двойной» контракт метод-параметров неприменим. Вместо этого используйте централизованную обработку через `@ControllerAdvice` и `@ExceptionHandler`.

При валидации «на уровне пути/параметров» (`@PathVariable`, `@RequestParam`) можно задействовать `@Validated` на классе контроллера и `@Min/@Max` на аргументах; такие проверки синхронны и бросают `ConstraintViolationException`.

Важно учитывать, что валидация больших `Flux<Dto>` может быть «бесконечной». Если контракт требует «первые N элементов» — валидируйте лениво и отбрасывайте лишнее с `take(N)`. Это снизит риск избыточной нагрузки.

Семантика ошибок должна быть единообразной: клиент ожидает один формат для 400/422 и для доменных ошибок. Возьмите `ProblemDetail` (Spring 6) как каноническую форму и заполняйте поля `type`, `title`, `detail`, `instance`.

Для сложных сценариев валидируйте в конвейере после бизнес-обогащения: иногда валидность зависит от данных из БД/внешнего сервиса. В реактивном стиле это просто дополнительный оператор `flatMap` с проверкой и `Mono.error(...)`.

Не перегружайте DTO валидацией, которая зависит от контекста. Разделяйте «структурную» (`@NotBlank`) и «контекстную» (существует ли такой ресурс) валидации — вторую делайте в сервисном слое.

Тестируйте негативные ветки `WebTestClient`-ом: отправляйте некорректные тела и проверяйте формат ошибок. Это защищает от «случайной» смены формата при обновлении зависимостей.

Помните, что сериализация ошибок может «упереться» в лимиты кодеков. Ставьте разумные `maxInMemorySize`, чтобы огромные тела не роняли сервис на этапе формирования ответа об ошибке.

Наконец, не возвращайте «слишком подробные» сообщения для атакующего. Валидация должна быть полезной для честного клиента, но не раскрывать внутренние детали.

**Java — валидация `Mono<Dto>` и `@ControllerAdvice`**

```java
package com.example.webflux.validation;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.Valid;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.bind.support.WebExchangeBindException;
import reactor.core.publisher.Mono;

record CreateUserReq(@NotBlank String name) {}

@RestController
class ValidationController {

    @PostMapping("/v/users")
    Mono<ResponseEntity<Void>> create(@Valid @RequestBody Mono<CreateUserReq> in) {
        return in.thenReturn(ResponseEntity.accepted().build());
    }
}

@RestControllerAdvice
class GlobalErrors {

    @ExceptionHandler(WebExchangeBindException.class)
    Mono<ResponseEntity<ProblemDetail>> handleBind(WebExchangeBindException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(400);
        pd.setTitle("Validation failed");
        pd.setDetail(ex.getAllErrors().stream()
            .map(err -> (err instanceof FieldError fe) ? fe.getField() + ": " + fe.getDefaultMessage()
                                                       : err.getDefaultMessage())
            .reduce((a,b) -> a + "; " + b).orElse("Invalid request"));
        return Mono.just(ResponseEntity.badRequest().body(pd));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.validation

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.validation.FieldError
import org.springframework.web.bind.annotation.*
import org.springframework.web.bind.support.WebExchangeBindException
import reactor.core.publisher.Mono

data class CreateUserReq(@field:NotBlank val name: String)

@RestController
class ValidationController {
    @PostMapping("/v/users")
    fun create(@Valid @RequestBody inDto: Mono<CreateUserReq>): Mono<ResponseEntity<Void>> =
        inDto.thenReturn(ResponseEntity.accepted().build())
}

@RestControllerAdvice
class GlobalErrors {

    @ExceptionHandler(WebExchangeBindException::class)
    fun handleBind(ex: WebExchangeBindException): Mono<ResponseEntity<ProblemDetail>> {
        val pd = ProblemDetail.forStatus(400)
        pd.title = "Validation failed"
        pd.detail = ex.allErrors.joinToString("; ") { err ->
            if (err is FieldError) "${err.field}: ${err.defaultMessage}" else err.defaultMessage ?: "Invalid"
        }
        return Mono.just(ResponseEntity.badRequest().body(pd))
    }
}
```

---

## Работа с файлами и формами: `Part`, `FilePart`, многопартовые загрузки

Загрузка файлов в WebFlux строится на `multipart/form-data`, но вместо `MultipartFile` используется реактивный `FilePart`. Он не читает файл целиком в память, а предоставляет методы для стримовой передачи и сохранения. Это критично для крупных файлов и экономии хипа.

Если в форме есть и поля, и файл, используйте `@RequestPart` для привязки `FilePart` и DTO одновременно. Для наборов файлов подойдёт `Flux<FilePart>` — элементы будут поступать по мере чтения тела запроса.

`FilePart.transferTo(Path)` под капотом выполняет неблокирующую передачу, но конкретный файловый бэкенд всё равно может быть блокирующим. Если вы делаете сложную пост-обработку (антивирус, вычисления) — отделите её и перенесите на `boundedElastic`.

При загрузках контролируйте лимиты: максимальный размер части, общее ограничение запроса и допустимые типы содержимого. Это снижает риск злоупотреблений и «проталкивания» огромных тел.

Чтение файла «ручками» возможно через `DataBuffer` и `DataBufferUtils`. Это нужно для «прокачки» байтов, валидации заголовков, вычисления хэшей «на лету», без буферизации всего файла.

Формы без файлов (url-encoded) также поддерживаются и биндятся в DTO через стандартные аннотации. Но как только появляется файл — переключайтесь на multipart и `@RequestPart`.

В ответах отдавайте ссылки/идентификаторы, а не содержимое файла «в лоб». Для скачивания делайте отдельный GET с валидацией прав и корректными заголовками `Content-Disposition`.

Не забывайте про безопасность путей при сохранении: никогда не используйте имя файла из клиента как путь на сервере. Генерируйте собственные ID и храните безопасно (облако/объектное хранилище).

Для больших файлов отдача тоже должна быть потоковой. Используйте `BodyInserters.fromResource` или `DataBufferUtils.read` из `AsynchronousFileChannel` — клиент начнёт получать байты сразу.

И, наконец, логируйте метаданные, но не содержимое. Размеры, типы, время обработки — важно; сами байты — нет.

**Java — загрузка файла и сохранение**

```java
package com.example.webflux.files;

import org.springframework.http.MediaType;
import org.springframework.http.codec.multipart.FilePart;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.nio.file.Path;
import java.util.UUID;

record UploadResult(String id) {}

@RestController
class UploadController {

    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    Mono<UploadResult> upload(@RequestPart("file") FilePart file) {
        String id = UUID.randomUUID().toString();
        Path target = Path.of(System.getProperty("java.io.tmpdir"), id + "-" + file.filename());
        return file.transferTo(target).thenReturn(new UploadResult(id));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.files

import org.springframework.http.MediaType
import org.springframework.http.codec.multipart.FilePart
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Mono
import java.nio.file.Path
import java.util.UUID

data class UploadResult(val id: String)

@RestController
class UploadController {

    @PostMapping("/upload", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
    fun upload(@RequestPart("file") file: FilePart): Mono<UploadResult> {
        val id = UUID.randomUUID().toString()
        val target = Path.of(System.getProperty("java.io.tmpdir"), "$id-${file.filename()}")
        return file.transferTo(target).thenReturn(UploadResult(id))
    }
}
```

---

## Контент-негациация и сериализация, reactive `HttpMessageReader/Writer`

Контент-негациация (content negotiation) — это выбор представления ответа на основе `Accept` клиента и `produces` метода. В WebFlux это работает так же, как в MVC, но важно помнить о «поштучной» сериализации для `Flux`: кодек должен уметь писать элементы по мере готовности.

JSON по умолчанию обеспечивается Jackson-кодеком. Его можно настроить через `CodecCustomizer` или `ExchangeStrategies`: добавить модули, изменить видимость, включить/отключить фичи. Это влияет и на производительность, и на совместимость схемы.

Если нужны альтернативные форматы (protobuf/CBOR/Smile), их кодеки тоже можно подключить. Убедитесь, что формат подходит для стриминга: некоторые «пакетные» форматы требуют предварительного знания длины.

Для больших ответов ограничьте `maxInMemorySize`: иначе кодек попытается агрегировать слишком много данных. Это особенно актуально при ошибках сериализации — лучше вернуть аккуратную 500, чем упасть с OOM.

Сервер и клиент могут договориться о компрессии. На уровне Netty включайте `compress(true)`, а на периметре — gzip/deflate/brotli. Для стриминга gzip обычно уместен — чанки будут сжиматься по мере появления.

Вы можете явно задавать `produces` на методе, чтобы устранить двусмысленность, и `consumes` — чтобы отклонять неподдерживаемые тела ещё на раннем этапе. Это улучшает DX и снижает лишнюю работу.

Если ответ зависит от `Accept`, тестируйте оба варианта `WebTestClient`-ом: `accept(MediaType.APPLICATION_JSON)` и, например, `accept(MediaType.APPLICATION_CBOR)`. Это защитит от неожиданных регрессий кодеков.

Кодеки — часть поверхности атаки. Отключайте десериализацию «опасных» типов, запрещайте по умолчанию неизвестные свойства, ограничивайте глубину/размер JSON.

В реактивном слое есть низкоуровневые `ServerRequest/ServerResponse` и `BodyInserters` — ими удобно отдавать поток байтов, если вы хотите полностью контролировать сериализацию.

Наконец, держите настройки сериализации одинаковыми между сервисами. Различия в дате/времени, точности чисел и правилах null-ов приводят к «микро-инцидентам» и сложным багам.

**Java — настройка Jackson и явное `produces/consumes`**

```java
package com.example.webflux.codec;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import org.springframework.context.annotation.*;
import org.springframework.http.MediaType;
import org.springframework.http.codec.CodecCustomizer;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

record Item(String id, String name) {}

@Configuration
class JacksonCfg {
    @Bean
    CodecCustomizer codecCustomizer(ObjectMapper mapper) {
        return c -> mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}

@RestController
class CodecController {

    @PostMapping(value = "/items", consumes = MediaType.APPLICATION_JSON_VALUE,
                                  produces = MediaType.APPLICATION_JSON_VALUE)
    Mono<Item> create(@RequestBody Mono<Item> in) {
        return in.map(it -> new Item("gen-1", it.name()));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.codec

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.SerializationFeature
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.http.codec.CodecCustomizer
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Mono

data class Item(val id: String, val name: String)

@Configuration
class JacksonCfg {
    @Bean
    fun codecCustomizer(mapper: ObjectMapper): CodecCustomizer =
        CodecCustomizer { mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS) }
}

@RestController
class CodecController {
    @PostMapping("/items", consumes = [MediaType.APPLICATION_JSON_VALUE], produces = [MediaType.APPLICATION_JSON_VALUE])
    fun create(@RequestBody inDto: Mono<Item>): Mono<Item> =
        inDto.map { Item("gen-1", it.name) }
}
```

---

## Потоковые ответы: SSE, `application/stream+json`, управление частями

Потоковые ответы — сильная сторона WebFlux. Когда вы возвращаете `Flux<T>`, кодек отправляет элементы по мере готовности, а соединение остаётся открытым до завершения потока или отмены клиентом. Это снижает задержку и выравнивает нагрузку.

SSE (Server-Sent Events) — простой способ передавать события от сервера к браузеру. Достаточно поставить `produces = text/event-stream` и возвращать `Flux<ServerSentEvent<T>>`. Браузер сам переподключится при обрыве и продолжит получать события.

`application/stream+json` — удобный для машинных клиентов формат «по одному JSON-объекту за раз». Его легко парсить, а на стороне сервера не нужно держать весь массив в памяти.

Управление частями важно для «шумных» источников. Используйте `limitRate`, `window`, `bufferTimeout`, чтобы не перегружать клиента. Сервер может вставлять heartbeat-события, чтобы прокси не рвали idle-соединение.

Отключайте кэширование потоков: `Cache-Control: no-store`. SSE-клиентам можно задавать `retry` в теле события — браузер будет учитывать эту задержку между переподключениями.

Ошибки в стримах отдавайте аккуратно. Для SSE принято шлёпать диагностическое событие, а потом закрывать канал; для stream+json — завершайте поток и дайте клиенту понять, что данные прекратились не из-за тайм-аута.

Если клиент закрыл соединение, реактивная цепь получит отмену. Обязательно освобождайте ресурсы в `doFinally`, чтобы не «подвешивать» исходящие вызовы и не держать файлы/соединения.

С backpressure на стороне клиента не всё прозрачно: браузерные EventSource буферизуют данные. Поэтому проверяйте поведение под плохой сетью и ограничивайте размер событий.

Для крупных выгрузок подумайте об `NDJSON` (newline-delimited JSON) — он дружит с инструментами парсинга и «сжирает» минимум памяти у клиента.

И наконец, тестируйте стримы `WebTestClient`-ом: `returnResult()` возвращает консьюмера, через которого можно читать первые N элементов и assert-ить интервалы.

**Java — SSE и stream+json с дозированием**

```java
package com.example.webflux.streams;

import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

import java.time.Duration;

@RestController
class StreamController {

    @GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<ServerSentEvent<String>> sse() {
        return Flux.interval(Duration.ofMillis(500))
                   .limitRate(64)
                   .map(i -> ServerSentEvent.builder("tick-" + i).id(Long.toString(i)).build());
    }

    @GetMapping(value = "/orders", produces = "application/stream+json")
    Flux<String> orders() {
        return Flux.range(1, 1000)
                   .delayElements(Duration.ofMillis(10))
                   .map(i -> "{\"id\":" + i + "}");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.streams

import org.springframework.http.MediaType
import org.springframework.http.codec.ServerSentEvent
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Flux
import java.time.Duration

@RestController
class StreamController {

    @GetMapping("/sse", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun sse(): Flux<ServerSentEvent<String>> =
        Flux.interval(Duration.ofMillis(500))
            .limitRate(64)
            .map { ServerSentEvent.builder("tick-$it").id("$it").build() }

    @GetMapping("/orders", produces = ["application/stream+json"])
    fun orders(): Flux<String> =
        Flux.range(1, 1000).delayElements(Duration.ofMillis(10)).map { """{"id":$it}""" }
}
```

---

## Глобальный error handling: `@ControllerAdvice`, `ResponseStatusException`

В аннотационном стиле централизованный обработчик ошибок реализуется через `@RestControllerAdvice`. Он перехватывает исключения и формирует единый формат ответа — например, `ProblemDetail` с нужными полями и ссылками на документацию. Это упрощает клиентов и облегчает эксплуатацию.

`ResponseStatusException` — удобный способ «выстрелить» правильным HTTP-кодом из любого места: `throw new ResponseStatusException(HttpStatus.NOT_FOUND, "User not found")`. В `@ControllerAdvice` вы можете дополнительно обогатить ответ деталями/корреляционным ID.

Для доменных ошибок создайте собственные исключения и сопоставьте им статусы. Например, `BusinessRuleViolation` → 422, `QuotaExceeded` → 429. Это повысит читаемость и убережёт от «магических чисел» по коду.

Обработчик ошибок в реактивном мире должен возвращать `Mono<...>` — он участвует в цепочке и не блокирует. Если выполнение было на event-loop, «тяжёлую» подготовку подробностей не делайте; максимум — сериализация маленького объекта.

Важно унифицировать коды и структуры: 400 — ошибки валидации, 401/403 — аутентификация/авторизация, 404 — ресурсы, 409 — конфликты версий, 422 — бизнес-валидация. Документируйте это в OpenAPI и проверяйте контрактными тестами.

Если используете функциональные роуты параллельно, учтите, что у них «свой» глобальный обработчик (`ErrorWebExceptionHandler`). В аннотационном мире оставайтесь в `@ControllerAdvice`, чтобы не смешивать механики.

Логируйте исключения со сэмплингом и кор-ID. Сырые стектрейсы нужны только для неожиданных 5xx; клиентским ошибкам достаточно краткого сообщения. Это сбережёт логи и не раскроет лишнего.

Не забывайте про локализацию сообщений, если API потребляют люди. Для машин лучше держать стабильные коды причин (`"code": "VALIDATION_FAILED"`) и человекочитаемое `detail` рядом.

При необходимости «перехватывайте» ошибки ниже — в `WebFilter` — чтобы добавлять заголовки корреляции и маскировать приватные детали. Но избегайте двойной сериализации.

И, наконец, тестируйте негативные сценарии так же тщательно, как позитивные: иначе при обновлении Spring вы рискуете незаметно «поменять» формат ошибок.

**Java — единый обработчик и пример доменной ошибки**

```java
package com.example.webflux.errors;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;
import reactor.core.publisher.Mono;

class QuotaExceeded extends RuntimeException {
    QuotaExceeded(String msg) { super(msg); }
}

@RestController
class SampleController {
    @GetMapping("/pay")
    Mono<String> pay() {
        if (true) throw new QuotaExceeded("Daily quota exceeded");
        // unreachable, пример
    }
}

@RestControllerAdvice
class ErrorsAdvice {

    @ExceptionHandler(QuotaExceeded.class)
    Mono<ResponseEntity<ProblemDetail>> quota(QuotaExceeded ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.TOO_MANY_REQUESTS);
        pd.setTitle("Quota exceeded");
        pd.setDetail(ex.getMessage());
        return Mono.just(ResponseEntity.status(429).body(pd));
    }

    @ExceptionHandler(ResponseStatusException.class)
    Mono<ResponseEntity<ProblemDetail>> rse(ResponseStatusException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(ex.getStatusCode());
        pd.setTitle("Request failed");
        pd.setDetail(ex.getReason());
        return Mono.just(ResponseEntity.status(ex.getStatusCode()).body(pd));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.errors

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import org.springframework.web.server.ResponseStatusException
import reactor.core.publisher.Mono

class QuotaExceeded(msg: String) : RuntimeException(msg)

@RestController
class SampleController {
    @GetMapping("/pay")
    fun pay(): Mono<String> {
        throw QuotaExceeded("Daily quota exceeded")
    }
}

@RestControllerAdvice
class ErrorsAdvice {

    @ExceptionHandler(QuotaExceeded::class)
    fun quota(ex: QuotaExceeded): Mono<ResponseEntity<ProblemDetail>> {
        val pd = ProblemDetail.forStatus(HttpStatus.TOO_MANY_REQUESTS)
        pd.title = "Quota exceeded"
        pd.detail = ex.message
        return Mono.just(ResponseEntity.status(429).body(pd))
    }

    @ExceptionHandler(ResponseStatusException::class)
    fun rse(ex: ResponseStatusException): Mono<ResponseEntity<ProblemDetail>> {
        val pd = ProblemDetail.forStatus(ex.statusCode)
        pd.title = "Request failed"
        pd.detail = ex.reason
        return Mono.just(ResponseEntity.status(ex.statusCode).body(pd))
    }
}
```

# 5. Функциональные маршруты (RouterFunction/HandlerFunction)

## Декларация маршрутов: `RouterFunctions.route()` и DSL, группировка по префиксам

Функциональная маршрутизация в WebFlux — это явное описание правил «какой запрос → какой обработчик». В отличие от аннотаций, здесь нет рефлексии: вы строите дерево правил из предикатов пути, метода, заголовков и типа контента. Такой подход даёт полный контроль над порядком сопоставления и облегчает чтение «таблицы маршрутов».

Базовый API — `RouterFunctions.route(RequestPredicate, HandlerFunction)`. Предикаты поставляются из `RequestPredicates`: `GET`, `POST`, `accept`, `contentType`, `path`, `headers` и их комбинации через `.and()`/`.or()`. Чем конкретнее предикат — тем выше правило должно стоять.

Для группировки по префиксам используется `RouterFunctions.nest(...)`: вы объявляете общий предикат (например, `path("/api/v1")`), а внутри — подмаршруты. Это упрощает «модулизацию» и миграции версий API, не дублируя строки.

Есть и Kotlin DSL-надстройка `router { }`, которая компактнее и ближе к декларативному стилю. Внутри `router` можно использовать `accept`, `contentType`, `nest("/prefix") { ... }`, а также регистрировать фильтры для группы маршрутов.

Маршруты можно комбинировать через `.andOther(...)` или оператор `+` (в Kotlin DSL): это позволяет собирать один «главный» роутер из отдельных конфигураций доменных модулей. Такое компонирование похоже на «плагины».

Приоритет важен: проверка идёт сверху вниз. Старайтесь ставить «самые конкретные» правила выше («`GET /orders/{id}`» раньше, чем общий «`/orders/**`»), иначе поймаете не тот обработчик.

Контент-негациация легко выражается предикатами: «`GET("/report").and(accept(MediaType.APPLICATION_PDF))`» отправит PDF-генератор, а `accept(JSON)` — JSON-представление; оба пути физически одинаковы, различается только `Accept`.

Для статических ресурсов удобны `resources("/assets/**", new ClassPathResource("public/"))` — это «хвост» схемы, который отдаёт фронт или документацию, не засоряя контроллеры.

Паттерн именования помогает поддержке: группируйте наборы маршрутов в отдельные методы/бины (`userRoutes()`, `orderRoutes()`), тестируйте их изолированно через `WebTestClient.bindToRouterFunction`.

С точки зрения производительности, функциональная маршрутизация убирает часть «магии» и ускоряет холодный старт/поиск хендлера. Это особенно заметно в простых API и шлюзах с множеством узких маршрутов.

**Java — группировка по префиксу и комбинирование маршрутов**

```java
package com.example.functional.routes;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.server.*;
import static org.springframework.web.reactive.function.server.RequestPredicates.*;

@Configuration
class RoutesConfig {

    @Bean
    RouterFunction<ServerResponse> userRoutes(UserHandler h) {
        return RouterFunctions.nest(path("/api/v1/users"),
                RouterFunctions.route(GET("/{id}").and(accept(MediaType.APPLICATION_JSON)), h::get)
                    .andRoute(GET("").and(accept(MediaType.APPLICATION_JSON)), h::list)
                    .andRoute(POST("").and(contentType(MediaType.APPLICATION_JSON)), h::create));
    }

    @Bean
    RouterFunction<ServerResponse> assetRoutes() {
        return RouterFunctions.resources("/assets/**", new org.springframework.core.io.ClassPathResource("public/"));
    }

    @Bean
    RouterFunction<ServerResponse> mainRouter(UserHandler h) {
        return userRoutes(h).andOther(assetRoutes());
    }
}
```

**Kotlin — эквивалент с DSL `router {}`**

```kotlin
package com.example.functional.routes

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.web.reactive.function.server.router

@Configuration
class RoutesConfig {

    @Bean
    fun mainRouter(h: UserHandler) = router {
        ("/api/v1/users").nest {
            accept(MediaType.APPLICATION_JSON).nest {
                GET("/{id}", h::get)
                GET("", h::list)
            }
            POST("", contentType(MediaType.APPLICATION_JSON), h::create)
        }
        resources("/assets/**", org.springframework.core.io.ClassPathResource("public/"))
    }
}
```

---

## Обработчики: `ServerRequest`/`ServerResponse`, паттерн «чистых» хендлеров

`HandlerFunction<ServerResponse>` — это «чистая» функция: на входе `ServerRequest`, на выходе `Mono<ServerResponse>`. Такой контракт поощряет явную работу с телом/параметрами и делает тестирование простым.

`ServerRequest` содержит путь, параметры, заголовки и тело. Тело читается реактивно: `request.bodyToMono(Dto.class)` или `request.bodyToFlux(Dto.class)`. Следите за типами — парсинг выполнится только когда подпишутся на результат.

`ServerResponse` — билдер HTTP-ответа. Вы управляете кодом, заголовками, типом контента и телом (`bodyValue`, `fromPublisher`). Это эквивалент `ResponseEntity`, но заточенный под функциональный стиль.

Паттерн «чистых» хендлеров: обработчик ничего не знает о внешних фреймворках, кроме контракта WebFlux; доменная логика вынесена в сервисы, которые возвращают `Mono/Flux`. Хендлер маппит вход → сервис → HTTP-ответ.

Такой слой легко мокать: в тестах вы создаёте `ServerRequest` через вспомогательные билдеры, подставляете фейковый сервис и проверяете формирование `ServerResponse`. Это короче и быстрее, чем поднимать контекст.

Чтение path-аргументов — через `request.pathVariable("id")`, query — `request.queryParam("q")`. Выражайте валидацию явно: пустой `Optional` → 400, неверный UUID → 400/422, отсутствующий ресурс → 404.

Если тело большое, не собирайте «всё в память» без нужды: стримьте `Flux` в `ServerResponse.ok().body(fromPublisher(..., Type))`. Кодек будет писать чанки по мере готовности.

Ошибки домена поднимайте как сигналы (`Mono.error`), а не `try/catch`. Централизованный обработчик в функциональном стиле перехватит их и сформирует унифицированный ответ.

Для статусов 201 добавляйте `location` заголовок: это улучшает DX и соответствует REST-обычаям. В билдере есть `created(URI)` и `build()` без тела.

Держите обработчики маленькими: парсинг, делегирование, сборка ответа. Всё остальное — сервисы и адаптеры (кэш, БД, HTTP). Так вы избежите «анемии» и залежей инфраструктуры в веб-слое.

**Java — чистый хендлер с чтением тела и маппингом ошибок**

```java
package com.example.functional.handlers;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

import java.net.URI;

record User(String id, String name) {}
record CreateUser(String name) {}

interface UserService {
    Mono<User> get(String id);
    Mono<User> create(String name);
}

@Component
class UserHandler {

    private final UserService service;
    UserHandler(UserService service) { this.service = service; }

    Mono<ServerResponse> get(ServerRequest req) {
        String id = req.pathVariable("id");
        return service.get(id)
            .flatMap(u -> ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(u))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    Mono<ServerResponse> create(ServerRequest req) {
        return req.bodyToMono(CreateUser.class)
            .flatMap(dto -> service.create(dto.name()))
            .flatMap(u -> ServerResponse.created(URI.create("/api/v1/users/" + u.id())).build());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.functional.handlers

import org.springframework.http.MediaType
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.server.ServerRequest
import org.springframework.web.reactive.function.server.ServerResponse
import reactor.core.publisher.Mono
import java.net.URI

data class User(val id: String, val name: String)
data class CreateUser(val name: String)

interface UserService {
    fun get(id: String): Mono<User>
    fun create(name: String): Mono<User>
}

@Component
class UserHandler(private val service: UserService) {

    fun get(req: ServerRequest): Mono<ServerResponse> =
        service.get(req.pathVariable("id"))
            .flatMap { ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).bodyValue(it) }
            .switchIfEmpty(ServerResponse.notFound().build())

    fun create(req: ServerRequest): Mono<ServerResponse> =
        req.bodyToMono(CreateUser::class.java)
            .flatMap { service.create(it.name) }
            .flatMap { ServerResponse.created(URI.create("/api/v1/users/${it.id}")).build() }
}
```

---

## Фильтры и перехватчики: `HandlerFilterFunction`, сквозные аспекты (лог/метрики)

Для функционального стиля «интерсептор» — это `HandlerFilterFunction<ServerResponse, ServerResponse>`. Он оборачивает вызов хендлера, позволяя реализовать кросс-срезы: логирование, метрики, корреляцию, аутентификацию.

Фильтр получает `ServerRequest` и `HandlerFunction`, и сам решает — вызвать «следующий» (`next.handle(request)`) или прервать цепочку (например, вернуть 401/429). Это удобно для ранних отказов и rate limiting.

Логирование на уровне фильтра даёт доступ к заголовкам/параметрам и времени обработки. Добавив `deferContextual` и MDC, вы получите сквозной кор-ID без привязки к потокам.

Метрики считываются через `Mono.deferContextual` и `Timer` из Micrometer. Оборачивайте вызов `next.handle(...)` `doOnSuccess/doOnError` и записывайте длительность, теги метода/пути/статуса.

Фильтры можно применять к отдельной группе маршрутов: `router.filter(myFilter)`. Так аутентификация/логирование для админ-зоны не «отравит» публичные ресурсы.

Для простого CORS в функциональном мире тоже возможен фильтр: проверяете `Origin`/`Access-Control-Request-Method` и отвечаете на preflight. Но обычно лучше включить встроенную конфигурацию CORS в WebFlux.

Порядок фильтров имеет значение: сначала аутентифицируйтесь/ограничивайте, потом логируйте/метрите. Это экономит ресурсы и не «замазывает» логи шумом отказов.

Помните, что фильтр — реактивный: никакого `block()`. Если нужно обратиться к внешнему сервису (например, токен-интроспекция) — делайте это реактивно и с тайм-аутом.

Отдельный фильтр полезен для унификации заголовков безопасности (`X-Content-Type-Options`, `X-Frame-Options`, CSP) на уровне конкретного «батча» маршрутов.

Фильтры — отличный шов для A/B и фичфлагов: читаете флаг из контекста/заголовка, направляете на другой хендлер (`return altHandler.handle(req)`), не меняя остальной код.

**Java — фильтр логирования/метрик для группы маршрутов**

```java
package com.example.functional.filters;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

@Configuration
class FilteredRoutesConfig {

    @Bean
    RouterFunction<ServerResponse> filtered(UserHandler h, MeterRegistry meters) {
        HandlerFilterFunction<ServerResponse, ServerResponse> metrics = (req, next) -> {
            long start = System.nanoTime();
            return next.handle(req)
                .doOnSuccessOrError((res, ex) -> {
                    long dur = System.nanoTime() - start;
                    String path = req.requestPath().pathWithinApplication().value();
                    Timer.builder("http.server.requests")
                         .tag("path", path)
                         .tag("status", res != null ? String.valueOf(res.statusCode().value()) : "500")
                         .register(meters)
                         .record(dur, java.util.concurrent.TimeUnit.NANOSECONDS);
                });
        };
        return RouterFunctions.route(RequestPredicates.GET("/f/users/{id}"), h::get)
                .andRoute(RequestPredicates.POST("/f/users"), h::create)
                .filter(metrics);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.functional.filters

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.server.ServerResponse
import org.springframework.web.reactive.function.server.router
import java.util.concurrent.TimeUnit

@Configuration
class FilteredRoutesConfig {

    @Bean
    fun filtered(h: UserHandler, meters: MeterRegistry) = router {
        val metrics = org.springframework.web.reactive.function.server.HandlerFilterFunction<ServerResponse, ServerResponse> { req, next ->
            val start = System.nanoTime()
            next.handle(req).doOnSuccessOrError { res, _ ->
                val dur = System.nanoTime() - start
                val path = req.requestPath().pathWithinApplication().value()
                Timer.builder("http.server.requests")
                    .tag("path", path)
                    .tag("status", res?.statusCode()?.value()?.toString() ?: "500")
                    .register(meters)
                    .record(dur, TimeUnit.NANOSECONDS)
            }
        }
        ("/f").nest {
            GET("/users/{id}", h::get)
            POST("/users", h::create)
            filter(metrics)
        }
    }
}
```

---

## Ошибки/исключения: фильтры, fallback-маршруты, централизованный маппинг

В функциональном стиле «глобальный» обработчик ошибок — это `WebExceptionHandler` (или более удобный `AbstractErrorWebExceptionHandler`). Он перехватывает исключения на уровне всей цепочки и формирует унифицированный ответ, например `ProblemDetail`.

Альтернатива — «локальный» фильтр, который оборачивает `next.handle(req)` и маппит известные исключения домена на коды статуса. Это удобно для модуля, где известен набор ошибок и формат.

Fallback-маршрут (`RequestPredicates.all()`) можно использовать как мягкий 404/«маршрут по умолчанию» внутри конкретной ветки (`/admin/**`). Но помните, что у Boot уже есть 404; ваш fallback — скорей для кастомных страниц или мультитенантных схем.

Важно не смешивать два мира: аннотационный `@ControllerAdvice` обработчик не работает для функциональных маршрутов; для них — `WebExceptionHandler`. Если в проекте есть оба стиля, держите два обработчика.

Старайтесь возвращать согласованный формат ошибок. В Spring 6 есть `ProblemDetail`, который легко сериализуется Jackson’ом; в нём есть `type`, `title`, `detail`, `instance`. Это упрощает клиентов и алерты.

Реактивность ошибок сохраняется: обработчик должен вернуть `Mono<Void>` (низкоуровневый `WebExceptionHandler`) или `Mono<ServerResponse>` (внутри `AbstractErrorWebExceptionHandler`). Не делайте блокировок.

Для трассировки добавляйте correlation-id в ответ и логи. Это особенно полезно, когда ошибка возникает уже «внутри» цепочки после нескольких исходящих вызовов.

Не раскрывайте лишние детали в 5xx. Стек-трейсы — только в логи. Клиенту — стабильный код и краткое сообщение. Подробности — по кор-ID внутри APM.

Подумайте о «маскировании» исключений от зависимых сервисов (например, 502 от партнёра → ваша доменная 424). Конвертируйте их в одном месте, чтобы фронт не зависел от деталей интеграций.

Наконец, покройте негативные сценарии `WebTestClient`-тестами: отправляйте некорректные тела, провоцируйте доменные исключения, проверяйте коды/формат. Это защитит от незаметной смены формата при апдейтах.

**Java — централизованный обработчик ошибок для функциональных маршрутов**

```java
package com.example.functional.errors;

import org.springframework.boot.web.error.ErrorAttributeOptions;
import org.springframework.boot.web.reactive.error.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

@Configuration
class FunctionalErrorHandler extends AbstractErrorWebExceptionHandler {

    FunctionalErrorHandler(ErrorAttributes errorAttributes, ApplicationContext applicationContext) {
        super(errorAttributes, new org.springframework.boot.autoconfigure.web.WebProperties.Resources(), applicationContext);
        setMessageWriters(org.springframework.http.codec.support.DefaultServerCodecConfigurer.create().getWriters());
        setMessageReaders(org.springframework.http.codec.support.DefaultServerCodecConfigurer.create().getReaders());
    }

    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(ErrorAttributes errorAttributes) {
        return RouterFunctions.route(RequestPredicates.all(), request -> {
            var attrs = errorAttributes.getErrorAttributes(request, ErrorAttributeOptions.defaults());
            return ServerResponse
                .status((int) attrs.getOrDefault("status", 500))
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(attrs);
        });
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.functional.errors

import org.springframework.boot.web.error.ErrorAttributeOptions
import org.springframework.boot.web.reactive.error.AbstractErrorWebExceptionHandler
import org.springframework.boot.web.reactive.error.ErrorAttributes
import org.springframework.context.ApplicationContext
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.http.codec.support.DefaultServerCodecConfigurer
import org.springframework.web.reactive.function.server.*
import reactor.core.publisher.Mono

@Configuration
class FunctionalErrorHandler(
    errorAttributes: ErrorAttributes,
    applicationContext: ApplicationContext
) : AbstractErrorWebExceptionHandler(
    errorAttributes,
    org.springframework.boot.autoconfigure.web.WebProperties.Resources(),
    applicationContext
) {
    init {
        val cfg = DefaultServerCodecConfigurer.create()
        setMessageReaders(cfg.readers)
        setMessageWriters(cfg.writers)
    }

    override fun getRoutingFunction(errorAttributes: ErrorAttributes): RouterFunction<ServerResponse> =
        RouterFunctions.route(RequestPredicates.all()) { request ->
            val attrs = errorAttributes.getErrorAttributes(request, ErrorAttributeOptions.defaults())
            ServerResponse.status((attrs["status"] as? Int) ?: 500)
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(attrs)
        }
}
```

---

## Сосуществование аннотационного и функционального стилей в одном приложении

Spring допускает одновременное использование аннотационных контроллеров и функциональных маршрутов. Это удобно при миграции: новые «тонкие» участки пишете функционально, старые — оставляете на аннотациях.

Главное — логическая изоляция: разделяйте префиксы (`/api/func/**` и `/api/anno/**`), чтобы маршруты не спорили. Порядок сопоставления учитывает конкретность, но явное разделение снижает риск ловить «не тот» обработчик.

В безопасности также лучше развести цепочки: отдельный `SecurityWebFilterChain` на функциональные пути и отдельный — на аннотационные. Это позволит конфигурировать CORS/CSRF/ролей без побочных эффектов.

Единый формат ошибок можно сохранить: аннотационную часть ведёт `@ControllerAdvice`, функциональную — `WebExceptionHandler`, но оба сериализуют `ProblemDetail` и используют одинаковые коды.

Наблюдаемость делайте общей: Micrometer/OTel одинаково работают и там, и там. В фильтрах/интерсепторах прокладывайте кор-ID в контекст и MDC, чтобы видеть сквозные трассы.

Фича-флаги помогут «переключать» маршрутизацию: фильтр может по флагу направлять на функциональный/аннотационный обработчик. Это мягкая миграция без разветвления кода на длительное время.

Документацию (OpenAPI) придётся собирать из двух источников. Для функциональных маршрутов используйте springdoc-webflux-fn или собственные дескрипторы, для аннотаций — стандартный springdoc.

Контрактные тесты пишите на «публичный» интерфейс: `WebTestClient` к живому приложению. Это держит совместимость независимо от внутреннего стиля.

Остерегайтесь «двух разных» подходов к валидации. Старайтесь вынести правила в общие валидаторы/сервисный слой, чтобы не плодить различий в сообщениях об ошибках.

На уровне сборки/модулей полезно разделить слои: `web-anno`, `web-func`, `domain`, `infra`. Тогда сосуществование — просто составление итогового приложения, а не смешивание в одном пакете.

**Java — сосуществование: контроллер + функциональный роутер**

```java
package com.example.coexist;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/anno")
class HelloAnno {
    @GetMapping(value = "/hello", produces = MediaType.TEXT_PLAIN_VALUE)
    Mono<String> hello() { return Mono.just("hello-anno"); }
}

@Configuration
class FuncPart {

    @Bean
    RouterFunction<ServerResponse> funcRoute() {
        return RouterFunctions.route(RequestPredicates.GET("/api/func/hello"),
            req -> ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("hello-func"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.coexist

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.reactive.function.server.ServerResponse
import org.springframework.web.reactive.function.server.router
import reactor.core.publisher.Mono

@RestController
@RequestMapping("/api/anno")
class HelloAnno {
    @GetMapping("/hello", produces = [MediaType.TEXT_PLAIN_VALUE])
    fun hello(): Mono<String> = Mono.just("hello-anno")
}

@Configuration
class FuncPart {
    @Bean
    fun funcRoute() = router {
        GET("/api/func/hello") { ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("hello-func") }
    }
}
```

---

## Где функциональный стиль удобнее: простые API, high-perf маршрутизация, тестируемость

Функциональные маршруты блистают там, где много простых правил и мало «магии»: тонкие BFF, прокси/агрегаторы, шлюзы. Явные предикаты и отсутствие рефлексии ускоряют поиск маршрута и холодный старт.

В high-perf сценариях важна предсказуемость: минимум аннотаций и рефлексии, максимум «видимого» кода. Функциональный стиль делает горячие пути очевидными и легко профилируемыми.

Тестируемость — сильная сторона: `WebTestClient.bindToRouterFunction(router).build()` даёт быстрые, сервер-less тесты без запуска контейнера. Это ускоряет TDD и повышает покрытие негативных веток.

Слои с минимальной логикой (health, метаданные, технические эндпоинты) логично писать функционально — код короче, зависимостей меньше, изоляция выше.

Маршруты, критичные к порядку/приоритетам, удобнее держать в одном месте: дерево `nest { ... }` показывает «карту» API, а не разрозненные аннотации по пакетам.

Для «фоллбеков» и альтернативных веток функциональный стиль проще: фильтр может вернуть другой хендлер без наследования/проксей вокруг контроллеров.

Сборка экспериментальных API под фичфлагами делает функциональный стиль привлекательным: убираете блок `nest("/exp")` — и фича исчезла, не затрагивая остальной код.

С другой стороны, «богатые» контроллеры с `@InitBinder`, `@ModelAttribute`, шаблонами — территория аннотаций. Смешивать стили по зонам — нормальная практика.

Документация может быть удобнее в аннотациях, но для функциональных маршрутов есть springdoc-webflux-fn и ручные дескрипторы — выбирайте, что проще для вашей команды.

В итоге правило простое: **чем проще и «техничнее» слой, тем больше поводов для функционального стиля**; чем богаче «магию Spring MVC» вы используете, тем удобнее оставаться на аннотациях.

**Java — тестируемость: `WebTestClient` без сервера**

```java
package com.example.functional.tests;

import org.junit.jupiter.api.Test;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;
import static org.assertj.core.api.Assertions.assertThat;

class RouterUnitTest {

    RouterFunction<ServerResponse> router() {
        return RouterFunctions.route(RequestPredicates.GET("/ping"),
            req -> ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("pong"));
    }

    @Test
    void shouldRespondPong() {
        var client = org.springframework.test.web.reactive.server.WebTestClient.bindToRouterFunction(router()).build();
        client.get().uri("/ping").exchange()
              .expectStatus().isOk()
              .expectHeader().contentTypeCompatibleWith(MediaType.TEXT_PLAIN)
              .expectBody(String.class).value(s -> assertThat(s).isEqualTo("pong"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.functional.tests

import org.junit.jupiter.api.Test
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.reactive.function.server.ServerResponse
import org.springframework.web.reactive.function.server.router
import reactor.core.publisher.Mono
import kotlin.test.assertEquals

class RouterUnitTest {

    private fun router() = router {
        GET("/ping") { ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("pong") }
    }

    @Test
    fun shouldRespondPong() {
        val client = WebTestClient.bindToRouterFunction(router()).build()
        val body = client.get().uri("/ping").exchange()
            .expectStatus().isOk
            .expectHeader().contentTypeCompatibleWith(MediaType.TEXT_PLAIN)
            .expectBody(String::class.java).returnResult().responseBody
        assertEquals("pong", body)
    }
}
```

# 6. WebClient: реактивный HTTP-клиент

## Конфигурация: connection pool, timeouts, codecs, `ExchangeStrategies`

Реактивный `WebClient` — это неблокирующий HTTP-клиент на базе Reactor Netty, поддерживающий backpressure и потоковую обработку тела. Его ключевая ценность — возможность масштабировать исходящие вызовы без «взрыва» потоков. При высоком RPS и фан-ауте к нескольким бэкендам это критично для стабильности.

Пулы соединений управляются `ConnectionProvider`. Он ограничивает максимальное число одновременных соединений, держит keep-alive и переиспользует TCP-каналы. Правильные лимиты предотвращают шторм подключений и истощение дескрипторов. Хорошая практика — иметь отдельный `WebClient` (и провайдер) для «медленных» и «быстрых» бэкендов с разными лимитами.

Тайм-ауты должны быть на нескольких уровнях: DNS/handshake/connect/read/response. В Reactor Netty есть `responseTimeout` и отдельные read/write idle-тайм-ауты; дополнительно полезно ставить оператор `timeout(...)` в цепочке, чтобы «обрубать» долгие операции на стороне приложения и освобождать ресурсы.

Кодеки управляют сериализацией/десериализацией тела. Через `ExchangeStrategies` и `CodecCustomizer` вы настраиваете `maxInMemorySize` (защита от гигантских payload’ов), модули Jackson, бинарные кодеки (CBOR/Smile/Protobuf). Для потоковых ответов не собирайте всё в память — пишите чанками.

HTTP-параметры вроде компрессии (`compress(true)`), заголовков по умолчанию, логирующих фильтров (`ExchangeFilterFunction`) стоит задавать на билдере, а не «на лету» в каждом вызове. Это упрощает сопровождение и снижает риск забыть важный заголовок.

TLS — дорогая операция; при массовых вызовах держите keep-alive, включайте session resumption и будьте внимательны к размерам пулов. Для внутренних вызовов может пригодиться HTTP/2 (H2C/H2) — мультиплексирование уменьшит число TCP-соединений.

DNS-резолвинг — ещё один «узел». Для Kubernetes-кластеров и активного DNS-балансирования можно включить Netty-резолверы и кэшировать ответы на разумное время, чтобы не перегружать kube-dns.

Сетевые лимиты («верхние границы») — часть «safety net»: ограничьте максимальный размер заголовков, строк запроса и тела. Это защищает от злоупотреблений и случайных «свалок» памяти при ошибках в интеграциях.

В проде отделяйте «клиенты для критичных зависимостей» в качестве отдельных бинов. Это позволит тюнить их независимо, подключать специфические фильтры (аутентификация, ретраи, кэш) и видеть метрики раздельно.

Наконец, тестируйте настройки под реальную нагрузку: бурсты, медленные ответы, внезапные обрывы. Бумажные тайм-ауты без нагрузочного прогона легко оказываются слишком оптимистичными или избыточно строгими.

**Gradle (зависимости)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'io.projectreactor.netty:reactor-netty-http'
    implementation 'com.fasterxml.jackson.module:jackson-module-parameter-names'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("io.projectreactor.netty:reactor-netty-http")
    implementation("com.fasterxml.jackson.module:jackson-module-parameter-names")
}
```

**Java — настраиваемый `WebClient` с пулом и тайм-аутами**

```java
package com.example.webclient.config;

import io.netty.resolver.DefaultAddressResolverGroup;
import reactor.netty.http.client.HttpClient;
import reactor.netty.resources.ConnectionProvider;
import org.springframework.context.annotation.*;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.*;
import org.springframework.http.codec.ClientCodecConfigurer;
import org.springframework.web.reactive.function.client.ExchangeStrategies;

import java.time.Duration;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient fastBackendClient() {
        ConnectionProvider pool = ConnectionProvider.builder("fast-pool")
                .maxConnections(256)
                .pendingAcquireMaxCount(1000)
                .build();

        HttpClient http = HttpClient.create(pool)
                .resolver(DefaultAddressResolverGroup.INSTANCE)
                .compress(true)
                .responseTimeout(Duration.ofSeconds(2));

        ExchangeStrategies strategies = ExchangeStrategies.builder()
                .codecs((ClientCodecConfigurer cfg) -> cfg.defaultCodecs().maxInMemorySize(512 * 1024))
                .build();

        return WebClient.builder()
                .baseUrl("https://api.fast.local")
                .clientConnector(new ReactorClientHttpConnector(http))
                .exchangeStrategies(strategies)
                .defaultHeader("User-Agent", "app-fast-client/1.0")
                .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.config

import io.netty.resolver.DefaultAddressResolverGroup
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.client.reactive.ReactorClientHttpConnector
import org.springframework.web.reactive.function.client.ExchangeStrategies
import org.springframework.web.reactive.function.client.WebClient
import reactor.netty.http.client.HttpClient
import reactor.netty.resources.ConnectionProvider
import java.time.Duration

@Configuration
class WebClientConfig {

    @Bean
    fun fastBackendClient(): WebClient {
        val pool = ConnectionProvider.builder("fast-pool")
            .maxConnections(256)
            .pendingAcquireMaxCount(1000)
            .build()

        val http = HttpClient.create(pool)
            .resolver(DefaultAddressResolverGroup.INSTANCE)
            .compress(true)
            .responseTimeout(Duration.ofSeconds(2))

        val strategies = ExchangeStrategies.builder()
            .codecs { cfg -> cfg.defaultCodecs().maxInMemorySize(512 * 1024) }
            .build()

        return WebClient.builder()
            .baseUrl("https://api.fast.local")
            .clientConnector(ReactorClientHttpConnector(http))
            .exchangeStrategies(strategies)
            .defaultHeader("User-Agent", "app-fast-client/1.0")
            .build()
    }
}
```

---

## Маппинг ответов: `retrieve()/exchangeToMono()`, обработка 4xx/5xx, `onStatus`

В `WebClient` есть два уровня API. «Высокоуровневый» `retrieve()` упрощает happy-path: он сам бросит исключение при 4xx/5xx, а `bodyToMono/Flux` распарсят тело. «Низкоуровневый» `exchangeToMono/Flux` предоставляет полный доступ к `ClientResponse`: вы вручную решаете, как обрабатывать статусы, заголовки и тело, в том числе потоково.

`onStatus` у `retrieve()` даёт гибкий маппинг ошибок: вы можете прочитать тело ошибки как JSON и сформировать доменное исключение (например, `RemoteQuotaExceeded`), а не общее `WebClientResponseException`. Это повышает читаемость кода обработчиков выше по стеку.

Не все ответы содержат тело: `204 No Content` — нормальный исход. Используйте `toBodilessEntity()` или проверяйте статус в `exchangeToMono`, чтобы не пытаться парсить пустоту.

Сериализация больших массивов может «съесть» память. Если контракт допускает «по одному», используйте `exchangeToFlux` и потоковую обработку, чтобы не собирать всё в `List`.

Частая ошибка — слепо ретраить все 5xx. Некоторые 5xx у партнёра означают «неповторяемую» ошибку (например, валидации), а «под капотом» это просто 500. Корректнее сначала классифицировать через `onStatus`, а уже затем применять политику ретраев.

Важно читать и использовать заголовки (rate limit, request-id, retry-after). Они помогают принимать решения: продолжать/ждать/снижать частоту. С `exchangeToMono` это сделать проще — у вас есть полный доступ к `ClientResponse`.

Для безопасной сериализации ошибок давайте клиентским ошибкам (4xx) свои типы исключений и фиксируйте формат в контракте. Это облегчает тестирование негативных сценариев и не «прячет» смысл за общими эксепшенами.

Маппинг на `ProblemDetail` — хороший стиль: вы получаете единый формат полей (`type/title/detail/instance`) и упрощаете обработку на фронте. Храните «сырой» текст ответа для диагностики, но наружу отдавайте нормализованный объект.

Обязательно покрывайте негативные кейсы тестами `WebTestClient`/WireMock: поднимайте стабы с 4xx/5xx и проверяйте, что ваши мапперы действительно формируют ожидаемые исключения.

И помните про отмену: если верхний потребитель закрыл соединение, `WebClient` прерывает чтение тела. Это экономит сеть и CPU. Не нужно самим «закрывать» — корректно пишите цепочку.

**Java — `retrieve()` с `onStatus` и низкоуровневый `exchangeToMono`**

```java
package com.example.webclient.mapping;

import org.springframework.http.HttpStatus;
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.Mono;

record User(String id, String name) {}
class RemoteQuotaExceeded extends RuntimeException { RemoteQuotaExceeded(String m){super(m);} }

public class MappingService {
    private final WebClient client;

    public MappingService(WebClient client) { this.client = client; }

    public Mono<User> loadUser(String id) {
        return client.get().uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, resp ->
                resp.bodyToMono(String.class).defaultIfEmpty("bad request")
                    .map(body -> new WebClientResponseException("4xx: " + body,
                            resp.statusCode().value(), resp.statusCode().toString(), null, null, null)))
            .onStatus(status -> status.value() == 429, resp ->
                resp.bodyToMono(String.class).map(RemoteQuotaExceeded::new))
            .bodyToMono(User.class);
    }

    public Mono<Boolean> deleteUser(String id) {
        return client.delete().uri("/users/{id}", id)
            .exchangeToMono(resp -> {
                if (resp.statusCode().is2xxSuccessful()) return Mono.just(true);
                if (resp.statusCode() == HttpStatus.NOT_FOUND) return Mono.just(false);
                return resp.createException().flatMap(Mono::error);
            });
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.mapping

import org.springframework.http.HttpStatus
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.WebClientResponseException
import reactor.core.publisher.Mono

data class User(val id: String, val name: String)
class RemoteQuotaExceeded(msg: String) : RuntimeException(msg)

class MappingService(private val client: WebClient) {

    fun loadUser(id: String): Mono<User> =
        client.get().uri("/users/{id}", id)
            .retrieve()
            .onStatus({ it.is4xxClientError }) { resp ->
                resp.bodyToMono(String::class.java).defaultIfEmpty("bad request")
                    .map { body ->
                        WebClientResponseException("4xx: $body", resp.statusCode().value(), resp.statusCode().toString(), null, null, null)
                    }
            }
            .onStatus({ it.value() == 429 }) { resp ->
                resp.bodyToMono(String::class.java).map { RemoteQuotaExceeded(it) }
            }
            .bodyToMono(User::class.java)

    fun deleteUser(id: String): Mono<Boolean> =
        client.delete().uri("/users/{id}", id)
            .exchangeToMono { resp ->
                when {
                    resp.statusCode().is2xxSuccessful -> Mono.just(true)
                    resp.statusCode() == HttpStatus.NOT_FOUND -> Mono.just(false)
                    else -> resp.createException().flatMap { Mono.error(it) }
                }
            }
}
```

---

## Потоковые запросы/ответы, загрузка файлов, backpressure при чтении тела

Потоковый «вход» — это `BodyInserters.fromPublisher(Flux<DataBuffer>, ...)` или `Flux<Part>` для multipart. Вы можете отправлять большие файлы, не буферизуя их целиком в памяти, — данные читаются по частям и пишутся в сеть с уважением к backpressure.

Потоковый «выход» — чтение тела как `Flux<DataBuffer>` и запись на диск/в другой поток через `DataBufferUtils.write(...)`. Это защищает от OOM при скачивании больших файлов и позволяет встраивать контроль скорости, подсчёт хэшей, промежуточную валидацию.

Multipart-загрузка реализуется `BodyInserters.fromMultipartData(...)` или через `MultipartBodyBuilder`. Реактивность сохраняется: файлы читаются и отправляются кусками. Важно указать корректные `Content-Type` и имя части.

При потоковом чтении тела обязательно закрывайте ресурсы: `DataBufferUtils.release(...)` делает это автоматически, если вы используете утилиты записи/чтения. Избегайте ручной работы с `ByteBuffer`, если нет крайней необходимости.

Backpressure при чтении: если ваш downstream (например, файловая система) не успевает, ограничивайте скорость через `limitRate` или «окна». Иначе буферы будут скапливаться в памяти и в TCP-стеке.

Сторонние прокси/ingress могут обрывать idle-соединения. Для долгих потоков держите heartbeats или отправляйте данные небольшими порциями регулярно. На клиентах закладывайте возобновление скачивания (range-запросы).

Учитывайте контроль целостности: контрольные суммы/подписи. В реактивной цепи их удобно считать «на лету», не держа весь файл: `map` по байтам, параллельно считаете hash и пишете в файл.

Существуют пределы `maxInMemorySize` — при сериализации JSON-элементов и multipart-частей фреймворк может «агрегировать» данные; уменьшайте лимит и отдавайте/отправляйте чанками.

Проверяйте коды/заголовки до чтения всего тела: если это ошибка с большим JSON-ом, возможно, стоит прочитать только первые K байт для диагностики и прервать, чтобы не тратить канал впустую.

И наконец, помните про безопасность пути при сохранении файлов: никакого использования исходных имён и относительных путей от клиента — только сгенерированные идентификаторы и безопасные директории/объектные стореджи.

**Java — потоковое скачивание и загрузка файла**

```java
package com.example.webclient.streaming;

import org.springframework.core.io.FileSystemResource;
import org.springframework.http.MediaType;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import org.springframework.core.io.buffer.DataBufferUtils;

import java.nio.file.Path;
import java.time.Duration;

public class StreamingService {
    private final WebClient client;

    public StreamingService(WebClient client) { this.client = client; }

    public Mono<Path> downloadTo(Path target) {
        return client.get().uri("/big/file")
            .accept(MediaType.APPLICATION_OCTET_STREAM)
            .retrieve()
            .bodyToFlux(org.springframework.core.io.buffer.DataBuffer.class)
            .limitRate(64)
            .as(data -> DataBufferUtils.write(data, target))
            .timeout(Duration.ofSeconds(60))
            .then(Mono.just(target));
    }

    public Mono<String> uploadFile(Path file) {
        var parts = new LinkedMultiValueMap<String, Object>();
        parts.add("file", new FileSystemResource(file));
        return client.post().uri("/upload")
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .body(BodyInserters.fromMultipartData(parts))
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.streaming

import org.springframework.core.io.FileSystemResource
import org.springframework.core.io.buffer.DataBuffer
import org.springframework.core.io.buffer.DataBufferUtils
import org.springframework.http.MediaType
import org.springframework.util.LinkedMultiValueMap
import org.springframework.web.reactive.function.BodyInserters
import org.springframework.web.reactive.function.client.WebClient
import reactor.core.publisher.Mono
import java.nio.file.Path
import java.time.Duration

class StreamingService(private val client: WebClient) {

    fun downloadTo(target: Path): Mono<Path> =
        client.get().uri("/big/file")
            .accept(MediaType.APPLICATION_OCTET_STREAM)
            .retrieve()
            .bodyToFlux(DataBuffer::class.java)
            .limitRate(64)
            .let { DataBufferUtils.write(it, target) }
            .timeout(Duration.ofSeconds(60))
            .then(Mono.just(target))

    fun uploadFile(file: Path): Mono<String> {
        val parts = LinkedMultiValueMap<String, Any>().apply {
            add("file", FileSystemResource(file))
        }
        return client.post().uri("/upload")
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .body(BodyInserters.fromMultipartData(parts))
            .retrieve()
            .bodyToMono(String::class.java)
    }
}
```

---

## Повторные попытки/тайм-ауты: `retryWhen`, `timeout`, идемпотентность

Ретраи — мощный инструмент, но без идемпотентности таят риск дублировать операции. Повторять безопаснее GET/HEAD/PUT (в большинстве случаев) и «специальные» POST с идемпотентным ключом. Любой `retryWhen` должен быть ограничен по числу попыток и времени.

Используйте экспоненциальный backoff с джиттером, чтобы избежать синхронных штурмов зависимого сервиса. Reactor предлагает `Retry.backoff(...)` и `jitter(...)`. Комбинация с `timeout` обеспечивает «быстрый фейл» при долгих зависаниях.

Классифицируйте ошибки: сеть/timeout/5xx — кандидаты на повтор; 4xx — обычно нет (исключение — 429/409 с `Retry-After`). Делайте классификацию в `onStatus` или в фильтре, подставляя «метку» в исключение.

Идемпотентный ключ для POST можно передавать в заголовке (`Idempotency-Key`) и хранить у сервиса-получателя. На повторную попытку он обязан вернуть тот же результат. Это снимает риск двойных платежей/заказов.

`timeout` в цепочке — защита приложения, а не только клиента. Он отрывает «зависшую» работу, освобождает конвейер и позволяет быстро деградировать. Ставьте его на каждый исходящий вызов.

Не ретрайте бесконечно «мелкие» элементы в больших потоках: лучше оборачивать отдельный элемент в «результат с ошибкой» и принимать решение наверху (DLQ/запись в «плохую корзину»). Это ускоряет общее завершение.

Логируйте ретраи с кор-ID и счётчиком попыток. Это помогает в разборе инцидентов и оценке «дороговизны» внешних зависимостей. Метрики (количество ретраев, среднее число попыток) дают пищу для тюнинга.

Старайтесь не сочетать `retryWhen` и «широкий» тайм-аут балансировщика: приложение может старательно ретраить внутри, а LB оборвёт соединение. Согласуйте политики.

Тестируйте политику на WireMock/MockServer: сценарии 500→500→200, 429 с `Retry-After`, временной тайм-аут. Проверяйте, что поведение соответствует ожиданиям и не «пробивает» лимиты.

И помните: «меньше ретраев — лучше». Часто эффективнее деградировать (кэш/частичный ответ) и сигнализировать алертами, чем биться о стену.

**Java — политика ретраев с backoff и идемпотентный ключ**

```java
package com.example.webclient.retries;

import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;

import java.time.Duration;
import java.util.UUID;

public class RetryPolicyService {
    private final WebClient client;

    public RetryPolicyService(WebClient client) { this.client = client; }

    public Mono<String> createOrder(String payload) {
        String key = UUID.randomUUID().toString();
        return client.post().uri("/orders")
            .header("Idempotency-Key", key)
            .bodyValue(payload)
            .retrieve()
            .bodyToMono(String.class)
            .timeout(Duration.ofSeconds(2))
            .retryWhen(Retry.backoff(2, Duration.ofMillis(100)).jitter(0.5))
            .onErrorResume(ex -> Mono.just("fallback-id"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.retries

import org.springframework.web.reactive.function.client.WebClient
import reactor.core.publisher.Mono
import reactor.util.retry.Retry
import java.time.Duration
import java.util.UUID

class RetryPolicyService(private val client: WebClient) {

    fun createOrder(payload: String): Mono<String> {
        val key = UUID.randomUUID().toString()
        return client.post().uri("/orders")
            .header("Idempotency-Key", key)
            .bodyValue(payload)
            .retrieve()
            .bodyToMono(String::class.java)
            .timeout(Duration.ofSeconds(2))
            .retryWhen(Retry.backoff(2, Duration.ofMillis(100)).jitter(0.5))
            .onErrorResume { Mono.just("fallback-id") }
    }
}
```

---

## Интеграция с OAuth2/JWT (через фильтры и `ExchangeFilterFunction`)

Для вызовов к защищённым ресурсам `WebClient` можно «научить» автоматически добавлять токены. В Spring Security есть `ServerOAuth2AuthorizedClientExchangeFilterFunction`, который берёт `OAuth2AuthorizedClient` (по `clientRegistrationId`) и подставляет `Authorization: Bearer`. Это работает для client-credentials, auth-code (через хранилище авторизованных клиентов) и device code.

Если у вас уже есть JWT (например, вы — Resource Server и проксируете запрос в другой сервис), используйте кастомный `ExchangeFilterFunction`, который считывает токен из `ReactiveSecurityContextHolder` и переносит его в исходящий вызов («token relay»). Так downstream видит исходную аутентификацию.

Refresh-токены обрабатываются на стороне OAuth2 Client (или отдельного Auth-сервиса). Сам `WebClient` не должен «самопридумывать» ротацию — он лишь потребляет `OAuth2AuthorizedClient` и сигнализирует об истечении/невалидности.

Для разных зависимостей разумно делать отдельные регистрации клиентов (scopes, audiences) и собственные бины `WebClient` — так проще управлять доступами и видеть метрики.

Если потребовался mTLS — настраивается `HttpClient.secure(ssl -> ...)` с truststore/keystore. Это уже не про OAuth2, но часто сочетается (mTLS для межсервисного + JWT для пользователя).

Обязательно валидируйте аудитории/скоупы на стороне downstream и фильтруйте лишние claims. Избыточные payload’ы JWT увеличивают заголовки и мешают кэшам/прокси.

Тестирование: в юнит-тестах подменяйте `ReactiveSecurityContextHolder` нужной аутентификацией или внедряйте заглушку `JwtEncoder/Decoder` и проверяйте, что фильтр реально вставляет `Authorization`.

Локально удобно иметь Keycloak/Testcontainers-Keycloak со сценарием `client-credentials` и `authorization_code (PKCE)`. Контрактные тесты «в реальном железе» ловят много нюансов с редиректами/сессиями.

Логи не должны содержать токены. Маскируйте заголовки в логирующих фильтрах и не печатайте тело ответов, если там могут быть секреты.

При прокидывании токена учитывайте downgrade-риски: если upstream токен слабее, чем нужно downstream (нет нужного scope), то лучше выдавать 403 сразу, чем пересылать заведомо «нерабочий» токен и ждать 403 «изнутри».

**Gradle (OAuth2 client)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
}
```

**application.yml — client-credentials**

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          svc-a:
            provider: keycloak
            client-id: svc-a
            client-secret: ${SVC_A_SECRET}
            authorization-grant-type: client_credentials
        provider:
          keycloak:
            issuer-uri: https://keycloak.local/realms/app
```

**Java — `WebClient` с OAuth2 client и token relay**

```java
package com.example.webclient.oauth;

import org.springframework.context.annotation.*;
import org.springframework.security.oauth2.client.*;
import org.springframework.security.oauth2.client.web.reactive.function.client.ServletOAuth2AuthorizedClientExchangeFilterFunction; // for servlet env
import org.springframework.security.oauth2.client.web.reactive.function.client.ServerOAuth2AuthorizedClientExchangeFilterFunction; // for reactive env
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.Mono;

@Configuration
public class OAuthClients {

    @Bean
    WebClient svcAClient(ReactiveClientRegistrationRepository regs,
                         ReactiveOAuth2AuthorizedClientService svc) {
        var oauth = new ServerOAuth2AuthorizedClientExchangeFilterFunction(regs, svc);
        oauth.setDefaultClientRegistrationId("svc-a");
        return WebClient.builder().filter(oauth).baseUrl("https://svc-a.local").build();
    }

    @Bean
    WebClient tokenRelayClient() {
        return WebClient.builder()
            .filter((request, next) ->
                reactor.core.publisher.Mono.deferContextual(ctx -> {
                    var auth = org.springframework.security.core.context.ReactiveSecurityContextHolder.getContext()
                            .map(security -> security.getAuthentication());
                    return auth.flatMap(a -> {
                        String token = (a instanceof org.springframework.security.oauth2.server.resource.authentication.BearerTokenAuthentication b)
                                ? b.getToken().getTokenValue()
                                : null;
                        ClientRequest r = ClientRequest.from(request)
                                .headers(h -> { if (token != null) h.setBearerAuth(token); })
                                .build();
                        return next.exchange(r);
                    });
                }))
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.oauth

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.core.context.ReactiveSecurityContextHolder
import org.springframework.security.oauth2.client.ReactiveClientRegistrationRepository
import org.springframework.security.oauth2.client.ReactiveOAuth2AuthorizedClientService
import org.springframework.security.oauth2.client.web.reactive.function.client.ServerOAuth2AuthorizedClientExchangeFilterFunction
import org.springframework.web.reactive.function.client.ClientRequest
import org.springframework.web.reactive.function.client.WebClient

@Configuration
class OAuthClients {

    @Bean
    fun svcAClient(regs: ReactiveClientRegistrationRepository,
                   svc: ReactiveOAuth2AuthorizedClientService): WebClient {
        val oauth = ServerOAuth2AuthorizedClientExchangeFilterFunction(regs, svc)
        oauth.setDefaultClientRegistrationId("svc-a")
        return WebClient.builder().filter(oauth).baseUrl("https://svc-a.local").build()
    }

    @Bean
    fun tokenRelayClient(): WebClient =
        WebClient.builder()
            .filter { request, next ->
                ReactiveSecurityContextHolder.getContext()
                    .map { it.authentication }
                    .flatMap { auth ->
                        val token = (auth as? org.springframework.security.oauth2.server.resource.authentication.BearerTokenAuthentication)
                            ?.token?.tokenValue
                        val r = ClientRequest.from(request).headers { h -> token?.let { h.setBearerAuth(it) } }.build()
                        next.exchange(r)
                    }
            }
            .build()
}
```

---

## Метрики и трассировки: Micrometer, HTTP-тэги, распределённый трейсинг

В Spring Boot 3 наблюдаемость «первого класса»: Observability строится на Micrometer/Observation API и интегрирована в WebFlux/`WebClient`. Вы получаете таймеры по методам/статусам/URI и распределённые трейсы (OTel/Zipkin/Jaeger) «из коробки».

Подключив `spring-boot-starter-actuator` и `micrometer-registry-*`, вы увидите метрики `http.client.requests` с тегами `method`, `uri`, `status`, `outcome`, `exception`. Эти метрики полезны для SLO и оповещений (рост латентности, доля 5xx).

Reactor Netty умеет экспортировать сетевые метрики (пулы, байты, коннекты). Это помогает понять, что именно болит: приложение или сеть. Включайте умеренно — метрики тоже стоят CPU.

Трейсинг автоматически прокидывает `traceparent`/`b3` заголовки. С `WebClient` это делает Observations: вы увидите спаны «исходящий HTTP» внутри общего трейса запроса. Это бесценно для поиска узких мест в фан-ауте.

Кастомные теги и измерения можно добавлять фильтром `ExchangeFilterFunction`, оборачивая `next.exchange(request)` в `Timer`. Но чаще правильнее опираться на встроенные Observations — они унифицированы и видны всем интеграциям.

Сэмплирование трейсов — обязательное. Полный трейсинг всех запросов дороже, чем кажется. Используйте 1–10% на проде, повышая для проблемных сервисов/маршрутов на время инцидента.

Ошибки логируйте с кор-ID/trace-id. Так вы связываете логи и трейсы, экономите время расследований. В реактивном мире кор-ID держите в Reactor Context и переносите в MDC на границах логирования.

Метрики ретраев/тайм-аутов — отличный индикатор деградации партнёра. Считайте `retry.count`, `timeouts.total`, строите алерты на аномалии. Опять же, Observations могут помочь тегировать события.

Объединяйте метрики клиента и сервера: рост латентности на клиентах без роста на сервере подскажет сетевые проблемы/пулы/резолвер. Сильный перекос — сигнал к тюнингу пула или TTL DNS.

И не забывайте про дашборды: «читаемый» слой наблюдаемости (метод, путь, статус, партнёр) — это меньше «слепых зон» и быстрее MTTR.

**Gradle (наблюдаемость)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    runtimeOnly 'io.opentelemetry:opentelemetry-exporter-otlp'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    runtimeOnly("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

**application.yml — включаем метрики/трейсинг**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  tracing:
    sampling:
      probability: 0.1
  metrics:
    distribution:
      percentiles-histogram:
        http.client.requests: true
```

**Java — кастомные теги через фильтр (дополнение к встроенным Observations)**

```java
package com.example.webclient.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.context.annotation.*;
import org.springframework.web.reactive.function.client.*;

import java.util.concurrent.TimeUnit;

@Configuration
public class ClientMetricsConfig {

    @Bean
    WebClient meteredClient(MeterRegistry registry) {
        ExchangeFilterFunction metrics = (request, next) -> {
            long start = System.nanoTime();
            return next.exchange(request).doOnSuccess(resp -> {
                long dur = System.nanoTime() - start;
                Timer.builder("http.client.extra")
                        .tag("method", request.method().name())
                        .tag("uri", request.url().getPath())
                        .tag("status", resp != null ? String.valueOf(resp.statusCode().value()) : "ERR")
                        .register(registry)
                        .record(dur, TimeUnit.NANOSECONDS);
            });
        };
        return WebClient.builder().filter(metrics).build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.metrics

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.client.ExchangeFilterFunction
import org.springframework.web.reactive.function.client.WebClient
import java.util.concurrent.TimeUnit

@Configuration
class ClientMetricsConfig {

    @Bean
    fun meteredClient(registry: MeterRegistry): WebClient {
        val metrics = ExchangeFilterFunction { request, next ->
            val start = System.nanoTime()
            next.exchange(request).doOnSuccess { resp ->
                val dur = System.nanoTime() - start
                Timer.builder("http.client.extra")
                    .tag("method", request.method().name())
                    .tag("uri", request.url().path)
                    .tag("status", resp?.statusCode()?.value()?.toString() ?: "ERR")
                    .register(registry)
                    .record(dur, TimeUnit.NANOSECONDS)
            }
        }
        return WebClient.builder().filter(metrics).build()
    }
}
```

# 7. Доступ к данным: R2DBC и друзья

## Почему JDBC блокирует и как R2DBC решает задачу; драйверы и ограничения

JDBC исторически строится на модели «поток на запрос»: вызовы `executeQuery`, чтение `ResultSet` и сетевой ввод-вывод блокируют текущий поток до прихода данных. В реактивном приложении это губительно: блокируется event-loop, растут задержки, а под нагрузкой вы вынуждены плодить тысячи потоков. R2DBC (Reactive Relational Database Connectivity) решает проблему иначе: драйвер общается с БД асинхронно и возвращает `Publisher` (обычно `Flux<Row>`/`Mono<T>`), уважающий backpressure — потребитель сам определяет темп чтения.

В терминах Reactor это значит: чтение из БД — часть реактивной цепи, и если downstream не успевает, driver не будет «заливать» всё в память. В JDBC вы привыкли к «курсорному» API, но фактически драйвер читает данные блокирующе; R2DBC использует неблокирующие сокеты Netty и протокольные порталы (например, PostgreSQL portals) для дозированного чтения.

Поддерживаемые драйверы для популярных СУБД: r2dbc-postgresql, r2dbc-mssql, r2dbc-mysql/mariadb, r2dbc-h2. Зрелость разная: Postgres — самый зрелый; MySQL/MariaDB сильно продвинулись, но нюансы (auth, multi-result) встречаются; MSSQL имеет хорошие драйверы, но больше ограничений в фичах TDS. В любом случае — проверяйте конкретный драйвер на поддержу fetch size, batch, prepared statements, cancel, протоколы SSL.

Важно понимать ограничения R2DBC: это не JPA. Нет ленивых прокси и каскадов ORM, нет `EntityManager` с кешем первого уровня, нет автоматического генератора сложных join-графов. Есть маппинг строк в объекты и удобный `DatabaseClient`/`R2dbcEntityTemplate`, а также Spring Data R2DBC репозитории. Сложные join’ы пишутся руками (SQL), а агрегации делаются в БД/вьюхах — это честнее и предсказуемее под нагрузкой.

Транзакции в R2DBC поддерживаются — на уровне соединения и реактивного менеджера. Но поскольку выполнение «растянуто» во времени и может переключать потоки, транзакция должна «путешествовать» вместе с реактивным контекстом. Spring делает это прозрачно (через `R2dbcTransactionManager`/`@Transactional`), если вся цепочка остаётся реактивной и не содержит `block()`.

Драйверы R2DBC по-разному относятся к backpressure. Postgres драйвер умеет уважать demand и порционно запрашивать ряды; у некоторых других драйверов фактическое поведение ближе к «prefetch», но всё равно без блокировок. Тонкую порционность можно настраивать через `fetchSize` на `Statement` или на уровне connection options.

R2DBC — не серебряная пуля. Если остальной стек у вас блокирующий (JDBC-репозитории, старые HTTP-клиенты), вы построите «реактивный фасад над блокировками», и выгоды будет мало. Смысл появляется, когда весь путь неблокирующий: вход → бизнес → БД/HTTP → выход.

С точки зрения эксплуатации вы выигрываете не только латентность event-loop, но и предсказуемость ресурсов. Пиковые нагрузки в реактивной модели «распластываются» равномерно, потому что драйвер и пул соединений дозируют работу под demand, а не исходят из принципа «чем больше потоков — тем лучше».

Что насчёт миграции? Начинать проще с новых сервисов, где схема данных и запросов уже «заточены» под реактивный стиль. Для монолитов и тяжёлых JPA-домов лучше выделять отдельные реактивные BFF/агрегаторы и постепенно переносить горячие пути.

Схема коннекта отличается: вместо `jdbc:postgresql://…` вы пишете `r2dbc:postgresql://…`. Boot 3 автоматически поднимет `ConnectionFactory`, пул r2dbc-pool и `R2dbcEntityTemplate`, если у вас есть `spring-boot-starter-data-r2dbc` и корректный URL. Остальное — дело ваших репозиториев и SQL.

И напоследок: измеряйте. Даже при R2DBC вы можете уронить сервис ненастроенным пулом или чрезмерными join’ами. Смотрите метрики пула/соединений, лейтенси запросов, исключения из драйвера — и тюньте конфиги.

**Gradle зависимости (R2DBC + Postgres + пул)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
    runtimeOnly 'org.postgresql:postgresql'                // для Flyway/Liquibase миграций
    runtimeOnly 'io.r2dbc:r2dbc-postgresql'
    implementation 'io.r2dbc:r2dbc-pool'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    runtimeOnly("org.postgresql:postgresql")
    runtimeOnly("io.r2dbc:r2dbc-postgresql")
    implementation("io.r2dbc:r2dbc-pool")
}
```

**application.yml (R2DBC + пул)**

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/app
    username: app
    password: secret
  r2dbc:
    pool:
      enabled: true
      initial-size: 10
      max-size: 50
      max-idle-time: 30s
```

**Java — простейший запрос через DatabaseClient (реактивно)**

```java
package com.example.r2dbc.basics;

import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Flux;

@Repository
public class UserR2dbcDao {
    private final DatabaseClient db;

    public UserR2dbcDao(DatabaseClient db) { this.db = db; }

    public Flux<User> findAll() {
        return db.sql("select id, name from users")
                 .map((row, meta) -> new User(row.get("id", Long.class), row.get("name", String.class)))
                 .all();
    }

    public record User(Long id, String name) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.r2dbc.basics

import org.springframework.r2dbc.core.DatabaseClient
import org.springframework.stereotype.Repository
import reactor.core.publisher.Flux

@Repository
class UserR2dbcDao(private val db: DatabaseClient) {

    data class User(val id: Long, val name: String)

    fun findAll(): Flux<User> =
        db.sql("select id, name from users")
            .map { row, _ -> User(row.get("id", java.lang.Long::class.java)?.toLong()!!,
                                  row.get("name", String::class.java)!!) }
            .all()
}
```

---

## Spring Data R2DBC/Reactive Repositories: транзакции, `@Transactional` в реактивном мире

Spring Data R2DBC даёт привычные «репозитории» (`R2dbcRepository`) и `R2dbcEntityTemplate`. Они закрывают базовые CRUD и маппинг, но остаются честно реактивными: методы возвращают `Mono`/`Flux`, и вы свободны писать SQL при необходимости. Это «лёгкая ORM», без ленивых прокси, зато с предсказуемым SQL.

С транзакциями всё немного иначе, чем в JDBC: транзакция должна жить столько, сколько живёт реактивная цепочка, и «переноситься» между потоками. Spring делает это через `R2dbcTransactionManager` и реактивные прокси, поэтому `@Transactional` работает, пока вы не покидаете реактивный мир (никакого `block()` внутри).

Если вам нужна программная транзакция, используйте `TransactionalOperator`. Он оборачивает `Publisher` и гарантирует begin/commit/rollback вокруг вашего потока. Так удобно группировать несколько реактивных вызовов в одну транзакцию.

Распространённая ошибка — вызывать репозиторий, потом `.block()` ради «быстрого» решения. Это остановит event-loop и «сломает» реактивную транзакцию. Всегда возвращайте `Mono/Flux` из сервисов и «склеивайте» цепи операторами.

Важный нюанс — границы транзакции в реактивном коде. Поскольку сигнал `onComplete` определяет момент фиксации, будьте осторожны с параллелизмом (`flatMap` с concurrency). Если две операции должны быть строго последовательными в одной транзакции, используйте `concatMap` или цепочку `then`.

Тонкая настройка изоляции, read-only и тайм-аутов поддерживается: аннотация `@Transactional` имеет поля `isolation`, `readOnly`, `timeout`. Драйвер и СУБД должны это уважить; Postgres, как правило, поддерживает.

Для массовых операций избегайте «поэлементного» `save` в `flatMap` — лучше `DatabaseClient` с batch/unnest и одной транзакцией. Это значительно разгружает пул соединений и уменьшает время фиксирования.

Репозитории Spring Data R2DBC поддерживают `@Query` с нативным SQL и именованные параметры. В сложных проектах это часто лучший путь: вы видите точный SQL, планируете индексы и не теряете контроль из-за автомагии.

Проверяйте реактивные транзакции тестами, в том числе негативными: выбрасывайте ошибку в середине цепочки и убеждайтесь, что данные откатываются. Это спасёт от «полунаписанных» состояний.

И помните про observability: метрики транзакций (время начала-фиксации), размер batch, частота конфликтов — отличный материал для тюнинга.

**Java — репозиторий + реактивная транзакция**

```java
package com.example.r2dbc.tx;

import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.r2dbc.repository.R2dbcRepository;
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Service;
import org.springframework.transaction.ReactiveTransactionManager;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.reactive.TransactionalOperator;
import reactor.core.publisher.Mono;

@Table("accounts")
record Account(@Id Long id, String owner, Long balance) {}

interface AccountRepo extends R2dbcRepository<Account, Long> {}

@Service
class AccountService {
    private final AccountRepo repo;
    private final TransactionalOperator tx;

    AccountService(AccountRepo repo, ReactiveTransactionManager tm) {
        this.repo = repo;
        this.tx = TransactionalOperator.create(tm);
    }

    @Transactional
    public Mono<Void> transfer(Long fromId, Long toId, long amount) {
        return repo.findById(fromId)
            .zipWith(repo.findById(toId))
            .flatMap(tuple -> {
                Account from = tuple.getT1();
                Account to = tuple.getT2();
                if (from.balance() < amount) return Mono.error(new IllegalStateException("insufficient"));
                return repo.save(new Account(from.id(), from.owner(), from.balance() - amount))
                           .then(repo.save(new Account(to.id(), to.owner(), to.balance() + amount)))
                           .then();
            });
    }

    public Mono<Void> transferProgrammatic(Long fromId, Long toId, long amount) {
        return tx.execute(status -> transfer(fromId, toId, amount)).then();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.r2dbc.tx

import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Table
import org.springframework.data.r2dbc.repository.R2dbcRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.ReactiveTransactionManager
import org.springframework.transaction.annotation.Transactional
import org.springframework.transaction.reactive.TransactionalOperator
import reactor.core.publisher.Mono

@Table("accounts")
data class Account(@Id val id: Long?, val owner: String, val balance: Long)

interface AccountRepo : R2dbcRepository<Account, Long>

@Service
class AccountService(repo: AccountRepo, tm: ReactiveTransactionManager) {

    private val repo = repo
    private val tx = TransactionalOperator.create(tm)

    @Transactional
    fun transfer(fromId: Long, toId: Long, amount: Long): Mono<Void> =
        repo.findById(fromId)
            .zipWith(repo.findById(toId))
            .flatMap { (from, to) ->
                if (from.balance < amount) Mono.error(IllegalStateException("insufficient"))
                else repo.save(from.copy(balance = from.balance - amount))
                    .then(repo.save(to.copy(balance = to.balance + amount)))
                    .then()
            }

    fun transferProgrammatic(fromId: Long, toId: Long, amount: Long): Mono<Void> =
        tx.execute { transfer(fromId, toId, amount) }.then()
}
```

---

## Реактивные хранилища: MongoDB/Cassandra/Redis — особенности моделей

Реляционная БД — не единственный вариант. Весомая часть реактивного стека — это хранилища, изначально спроектированные под асинхронные драйверы: MongoDB, Cassandra, Redis. Они дают высокую конкуррентность и низкую латентность I/O, не блокируя event-loop.

MongoDB (spring-boot-starter-data-mongodb-reactive) — документное хранилище с гибкими схемами. Реактивный драйвер — «первого класса гражданин»: коллекции возвращают `Flux<Document>`/`Mono<T>`. Модель «один документ = агрегат» хорошо вписывается в микросервисы, где денормализация допустима.

Cassandra (spring-data-cassandra-reactive) — колоночное, распределённое хранилище, сильное в больших объёмах и предсказуемых ключах. Запросы проектируются «от чтения»: корректная ключевая модель важнее нормализации. Реактивный драйвер DataStax хорошо масштабируется при тысячах параллельных запросов.

Redis (spring-data-redis-reactive) — in-memory ключ-значение. Реактивный API полезен для кэшей, лимитеров, очередей. `ReactiveStringRedisTemplate` и реактивные операторы над структурами данных помогают добиться высокой пропускной способности без блокировок.

Выбор хранилища — компромисс. Документы Mongo — удобно, но без join’ов; Cassandra — феноменальна на write-heavy, но сложнее в сложных выборках; Redis — молниеносный, но volatile и ограничен по моделированию. Реактивность — не «спасающий» фактор от неправильной модели.

Интеграция с WebFlux проста: у всех стартеров методы уже возвращают `Mono/Flux`. Важнее не смешивать блокирующие клиенты: не подключайте «рядом» классический MongoDB Java Driver (синхронный), иначе рискуете случайно заблокироваться.

Транзакции в нереляционных реактивных хранилищах ограничены. У Mongo есть multi-document транзакции, но они дороже и снижают пропускную способность; в Cassandra транзакций нет в ПрО-смысле (есть LWT для условных апдейтов). Думайте о консистентности на уровне агрегатов и бизнес-идемпотентности.

Индексирование и схемы валидируйте заранее. В Mongo включайте валидаторы схем на коллекции, в Cassandra планируйте партиции так, чтобы чтение было «локальным». Наблюдайте tail-latency и анализируйте hot-keys (для Redis/Cassandra).

Не забывайте про backpressure: драйверы возвращают реактивные стримы, но сеть и память не безграничны. Ограничивайте параллелизм и размер ответа; используйте курсоры/лимиты/сканы с шагом.

В проде держите отдельные пула/клиенты под разные профили нагрузки. Например, один `ReactiveMongoTemplate` для «горячих» коллекций, другой — для фонов, с отличающимися тайм-аутами/пулами.

**Gradle зависимости**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb-reactive'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
    implementation 'org.springframework.boot:spring-boot-starter-data-cassandra-reactive'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb-reactive")
    implementation("org.springframework.boot:spring-boot-starter-data-redis-reactive")
    implementation("org.springframework.boot:spring-boot-starter-data-cassandra-reactive")
}
```

**Java — Mongo Reactive Repository и Redis template**

```java
package com.example.nosql.reactive;

import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.repository.ReactiveMongoRepository;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Document("users")
record User(@Id String id, String name) {}

interface UserRepository extends ReactiveMongoRepository<User, String> {}

@Service
class CacheService {
    private final ReactiveStringRedisTemplate redis;

    CacheService(ReactiveStringRedisTemplate redis) { this.redis = redis; }

    public Mono<Void> putUserName(String id, String name) {
        return redis.opsForValue().set("user:name:" + id, name).then();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.nosql.reactive

import org.springframework.data.annotation.Id
import org.springframework.data.mongodb.core.mapping.Document
import org.springframework.data.mongodb.repository.ReactiveMongoRepository
import org.springframework.data.redis.core.ReactiveStringRedisTemplate
import org.springframework.stereotype.Service
import reactor.core.publisher.Mono

@Document("users")
data class User(@Id val id: String?, val name: String)

interface UserRepository : ReactiveMongoRepository<User, String>

@Service
class CacheService(private val redis: ReactiveStringRedisTemplate) {
    fun putUserName(id: String, name: String): Mono<Void> =
        redis.opsForValue().set("user:name:$id", name).then()
}
```

---

## «Мосты» к блокирующим API: `boundedElastic`, bulk-операции, анти-паттерны

Иногда без блокирующего клиента нельзя: устаревший JDBC-драйвер, SOAP, файловая библиотека. В таких случаях мостите их на `boundedElastic()`. Это защищает event-loop, создавая ограниченный пул потоков для блокировок. Но это компромисс, а не норма.

Главная ошибка — «поэлементный offload»: `flux.flatMap(x -> Mono.fromCallable(() -> blocking(x)).subscribeOn(boundedElastic()))` для тысячи элементов породит тысячу задач и перегрузит пул/баблинг GC. Лучше буферизовать и делать bulk: `.buffer(100).flatMap(chunk -> offloadBulk(chunk))`.

Вторая ошибка — смешивать блокирующий JDBC внутри реактивной транзакции R2DBC. Такие «гибриды» ломают границы и делают поведение непредсказуемым. Если вам нужен JDBC — вынесите его в отдельный сервис/процесс, общайтесь реактивным HTTP.

Не используйте `block()`/`toIterable()`/`subscribe()` в бизнес-коде WebFlux. Это перечёркивает реактивность, нарушает перенос контекста и может повесить приложение. Если очень надо получить значение синхронно — значит, вы на неправильном уровне абстракции.

С антивирусами/архиваторами и прочими «тяжёлыми» библиотеками поступайте так: положили файл в S3/минИО, отправили команду на асинхронного воркера (queue), вернули клиенту 202 Accepted. Реактивный сервис остаётся лёгким, тяжёлая работа — вне пути запроса.

Если нужно «временно» использовать JDBC из WebFlux, делайте это явно: отдельный модуль, отдельный пул, чёткий лимит параллелизма, тайм-ауты, циркут-брейкер. И план миграции на R2DBC/реактивный стор.

Будьте осторожны с логированием и сериализацией больших объектов — это тоже блокирующие операции. Логируйте кратко, сэмплируйте, не форматируйте гигантские JSON в горячих путях.

Мосты — место для идемпотентности. Если блокирующий сервис падает/медлит, ретраи и тайм-ауты должны быть дозированы. Иначе очередь задач на `boundedElastic` станет бесконтрольной.

Наблюдайте за `boundedElastic.scheduler.maxPending` (Micrometer) и временем выполнения задач. Если видите рост очереди — срочно снижайте параллелизм и повышайте backpressure вверх по цепи.

Планируйте деградацию: если «мост» недоступен, отдайте кэш/стаб/частичный ответ. Это лучше, чем «держать» соединения до тайм-аута и уронить весь сервис.

**Java — неправильный и правильный offload**

```java
package com.example.bridges;

import org.springframework.jdbc.core.JdbcTemplate;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.util.List;

public class BridgeService {
    private final JdbcTemplate jdbc;

    public BridgeService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    // АНТИ-ПАТТЕРН: поэлементный offload
    public Mono<Integer> countBad(Flux<Long> ids) {
        return ids.flatMap(id -> Mono.fromCallable(() ->
                        jdbc.queryForObject("select count(*) from t where id=?", Integer.class, id))
                .subscribeOn(Schedulers.boundedElastic()))
            .reduce(0, Integer::sum);
    }

    // Лучше: буфер + batch
    public Mono<int[]> countGood(Flux<Long> ids) {
        return ids.buffer(100)
            .flatMap(batch -> Mono.fromCallable(() -> {
                List<Object[]> args = batch.stream().map(id -> new Object[]{id}).toList();
                return jdbc.batchUpdate("select /*+ some hint */ 1 from t where id=?", args);
            }).subscribeOn(Schedulers.boundedElastic()), 4)
            .reduce(new int[0], (acc, arr) -> arr); // пример
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.bridges

import org.springframework.jdbc.core.JdbcTemplate
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers

class BridgeService(private val jdbc: JdbcTemplate) {

    // АНТИ-ПАТТЕРН
    fun countBad(ids: Flux<Long>): Mono<Int> =
        ids.flatMap { id ->
            Mono.fromCallable {
                jdbc.queryForObject("select count(*) from t where id=?", Int::class.java, id)!!
            }.subscribeOn(Schedulers.boundedElastic())
        }.reduce(0) { a, b -> a + b }

    // Буфер + batch
    fun countGood(ids: Flux<Long>): Mono<IntArray> =
        ids.buffer(100)
            .flatMap({ batch ->
                Mono.fromCallable {
                    val args = batch.map { arrayOf<Any>(it) }
                    jdbc.batchUpdate("select /*+ some hint */ 1 from t where id=?", args)
                }.subscribeOn(Schedulers.boundedElastic())
            }, 4)
            .reduce(IntArray(0)) { _, arr -> arr }
}
```

---

## Потоковые запросы и пагинация: курсоры/лимиты, backpressure на БД уровне

Реактивная сила R2DBC — в потоковом чтении: вы можете обрабатывать тысячи строк, не держа их в памяти. Но чтобы это было эффективно, нужно дозировать чтение: `LIMIT / FETCH FIRST`, keyset-пагинация, fetch size, и, главное, осознанный дизайн запросов.

`OFFSET/LIMIT` удобны, но дороже на больших смещениях (перескан строк). Для бесконечных лент лучше keyset: «где id > :lastId order by id limit :pageSize». Это естественно сочетается с `Flux` и позволяет честно делать backpressure.

Некоторые драйверы поддерживают `Statement.fetchSize(int)`, что позволяет читать порциями на уровне протокола (например, PostgreSQL portals). В Spring R2DBC можно добраться до `Statement` через `DatabaseClient`/`GenericExecuteSpec.filter`.

Потоковый API не отменяет индексы. Проекции и покрытия индексов (only needed columns) драматически снижают I/O. Для стримов — это особенно важно: вы долгие минуты «живёте» на одном запросе.

Чтобы не «взорвать» downstream, ограничивайте rate: `limitRate(pageSize)` или `bufferTimeout`. Это даёт пространство для сериализации/отправки в сеть и предотвращает накопление очередей.

При агрегациях (SUM/COUNT) лучше делать их в БД, а не в приложении. Если всё же считаете на лету — применяйте `reduce`/`scan` и следите за размером накопителей.

Будьте осторожны с `collectList()` — это выключатель реактивности. Если нужно собрать «в ответ» всё, ставьте верхний лимит и метрики размера; иначе легко получить 100-мегабайтную страницу и свалиться в GC-паузу.

Для отчётов/экспортов применяйте серверные курсоры, если драйвер умеет, или разбивайте на чанки по id/диапазонам. Параллельная выгрузка по диапазонам (sharding key) часто быстрее, чем один огромный скан.

Тестируйте пагинацию с реальными объёмами. «Маленькие» таблицы скрывают проблемы: план запроса, сортировки, «горячие» партиции. Нагрузочные тесты покажут истинные латентности и хвостовые задержки.

И обязательно наблюдайте: время запроса, ряды/секунду, глубина очередей на сериализации. Эти показатели помогут выбрать разумный page size и ограничители конвейера.

**Java — keyset и fetchSize через DatabaseClient**

```java
package com.example.r2dbc.pagination;

import io.r2dbc.spi.Statement;
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Flux;

@Repository
public class OrderDao {
    private final DatabaseClient db;
    public OrderDao(DatabaseClient db) { this.db = db; }

    public Flux<Order> streamAfter(long lastId, int pageSize) {
        return db.sql("select id, amount from orders where id > :last order by id limit :ps")
            .bind("last", lastId)
            .bind("ps", pageSize)
            .filter((statement, execute) -> {
                Statement st = statement;
                try { st.fetchSize(pageSize); } catch (Throwable ignore) {}
                return execute.execute(st);
            })
            .map((row, meta) -> new Order(row.get("id", Long.class), row.get("amount", Long.class)))
            .all()
            .limitRate(pageSize);
    }

    public record Order(Long id, Long amount) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.r2dbc.pagination

import io.r2dbc.spi.Statement
import org.springframework.r2dbc.core.DatabaseClient
import org.springframework.stereotype.Repository
import reactor.core.publisher.Flux

@Repository
class OrderDao(private val db: DatabaseClient) {

    data class Order(val id: Long, val amount: Long)

    fun streamAfter(lastId: Long, pageSize: Int): Flux<Order> =
        db.sql("select id, amount from orders where id > :last order by id limit :ps")
            .bind("last", lastId)
            .bind("ps", pageSize)
            .filter { statement, execute ->
                try { (statement as Statement).fetchSize(pageSize) } catch (_: Throwable) {}
                execute.execute(statement)
            }
            .map { row, _ -> Order(row.get("id", java.lang.Long::class.java)!!.toLong(),
                                   row.get("amount", java.lang.Long::class.java)!!.toLong()) }
            .all()
            .limitRate(pageSize)
}
```

---

## Тонкая настройка пула соединений и тайм-аутов для стабильности

В реактивном мире пул соединений критичен: слишком маленький — получите очереди на получение коннекта, слишком большой — уроните БД/сеть. `r2dbc-pool` даёт гибкие параметры: `initialSize`, `maxSize`, `maxIdleTime`, `maxCreateConnectionTime`, валидация коннекта и пр.

Boot 3 умеет автоконфигурировать пул по `spring.r2dbc.pool.*`. Начните с умеренных значений (например, `max-size: 50` для средних сервисов) и смотрите на метрики: среднее/максимальное время получения коннекта, количество ожиданий, долю отказов.

Тайм-ауты важны на всех уровнях: connect, socket/read, statement. В R2DBC Postgres можно задавать `connectTimeout` через `ConnectionFactoryOptions`, а тайм-ауты запросов — через стороно-драйверные настройки либо в SQL (`statement_timeout` в Postgres per-session).

Следите за «утечками» коннектов: если цепочка закончилась ошибкой до релиза, коннект должен возвращаться в пул. Неправильное обращение с `Publisher` может оставить сессию «висящей». Автовалидация на отдаче-в-пул снижает риск.

Warmup пула (`initial-size`) улучшает холодный старт, но не стоит держать больше, чем действительно нужно на типичной нагрузке. Idle-коннекты тоже стоят ресурсов на БД.

Сетевые настройки (TCP keep-alive, TLS) влияют на устойчивасть длинных запросов. Выравнивайте тайм-ауты: не должно быть ситуации, когда socket-timeout меньше, чем statement-timeout — вы получите загадочные обрывы.

В multi-tenant сценариях используйте пулы на «тенанта» или хотя бы теги в метриках, чтобы видеть, кто ест соединения. Иначе один «шумный» клиент выжмет все коннекты.

Отдельные пулы под «критичные» и «фоново тяжёлые» операции — хороший приём. Не давайте бэкап-выгрузке съесть коннекты продового пути.

Обновления драйверов меняют дефолты. После апгрейда проверяйте метрики пула и тайм-аутов: «случайные» регрессии здесь обычны. И держите property-override в `application.yml`, чтобы не зависеть от дефолтов.

Наконец, не забывайте о мониторинге: Micrometer экспонирует метрики пула (время acquire, количество активных, idle). Повесьте алерты на рост `acquire` и «почти полный» пул — это ранний сигнал беды.

**application.yml — тонкий тюнинг пула и тайм-аутов (Postgres)**

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/app
    username: app
    password: secret
    properties:
      connectTimeout: PT3S
      ssl: false
  r2dbc:
    pool:
      enabled: true
      initial-size: 8
      max-size: 48
      max-idle-time: 45s
      validation-query: SELECT 1
```

**Java — явная сборка ConnectionFactory с опциями и пулом**

```java
package com.example.r2dbc.pool;

import io.r2dbc.pool.ConnectionPool;
import io.r2dbc.pool.ConnectionPoolConfiguration;
import io.r2dbc.postgresql.PostgresqlConnectionFactoryProvider;
import io.r2dbc.spi.ConnectionFactory;
import io.r2dbc.spi.ConnectionFactoryOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

import static io.r2dbc.spi.ConnectionFactoryOptions.*;

@Configuration
public class R2dbcPoolConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        ConnectionFactoryOptions options = ConnectionFactoryOptions.builder()
            .option(DRIVER, "postgresql")
            .option(HOST, "localhost")
            .option(PORT, 5432)
            .option(DATABASE, "app")
            .option(USER, "app")
            .option(PASSWORD, "secret")
            .option(Option.valueOf("connectTimeout"), Duration.ofSeconds(3))
            .build();

        ConnectionPoolConfiguration cfg = ConnectionPoolConfiguration.builder()
            .connectionFactory(io.r2dbc.postgresql.PostgresqlConnectionFactoryProvider.INSTANCE.create(options))
            .initialSize(8)
            .maxSize(48)
            .maxIdleTime(Duration.ofSeconds(45))
            .maxAcquireTime(Duration.ofSeconds(2))
            .build();

        return new ConnectionPool(cfg);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.r2dbc.pool

import io.r2dbc.pool.ConnectionPool
import io.r2dbc.pool.ConnectionPoolConfiguration
import io.r2dbc.postgresql.PostgresqlConnectionFactoryProvider
import io.r2dbc.spi.ConnectionFactory
import io.r2dbc.spi.ConnectionFactoryOptions
import io.r2dbc.spi.ConnectionFactoryOptions.*
import io.r2dbc.spi.Option
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class R2dbcPoolConfig {

    @Bean
    fun connectionFactory(): ConnectionFactory {
        val options = ConnectionFactoryOptions.builder()
            .option(DRIVER, "postgresql")
            .option(HOST, "localhost")
            .option(PORT, 5432)
            .option(DATABASE, "app")
            .option(USER, "app")
            .option(PASSWORD, "secret")
            .option(Option.valueOf("connectTimeout"), Duration.ofSeconds(3))
            .build()

        val cfg = ConnectionPoolConfiguration.builder()
            .connectionFactory(PostgresqlConnectionFactoryProvider.INSTANCE.create(options))
            .initialSize(8)
            .maxSize(48)
            .maxIdleTime(Duration.ofSeconds(45))
            .maxAcquireTime(Duration.ofSeconds(2))
            .build()

        return ConnectionPool(cfg)
    }
}
```

# 8. Безопасность в WebFlux (reactive)

## `ServerHttpSecurity` и реактивный `SecurityWebFilterChain`

В реактивных приложениях безопасность строится вокруг неблокирующей цепочки фильтров `SecurityWebFilterChain`. Это аналог servlet-модели, но для WebFlux: каждый фильтр — это реактивный компонент, который не блокирует event-loop и умеет корректно «реагировать» на отмену запроса. Базовая идея: вы объявляете единственный (или несколько) бинов `SecurityWebFilterChain` и последовательно настраиваете политику для входящих запросов — от CORS/CSRF и заголовков до правил авторизации по путям.

Конфиг начинается с `ServerHttpSecurity`, который Spring Boot 3.x/6.x подставляет в ваш `@Configuration`. Важно помнить, что по умолчанию политика — «secure by default»: всё закрыто, пока вы явно не откроете конкретные эндпоинты. Это уменьшает вероятность оставить «дырку» в новом контроллере и соответствует принципу наименьших привилегий.

Реактивная авторизация в WebFlux выражается через `authorizeExchange`, где вы настраиваете `pathMatchers`/`method`-матчеры и указываете требуемые роли/authority или разрешаете доступ. Порядок имеет значение: сначала ставьте более конкретные правила, ниже — общие, а в конце — `anyExchange().authenticated()` или `denyAll()` для «страховки».

Способы аутентификации зависят от сценария: для API — Bearer/JWT через `oauth2ResourceServer().jwt()`, для технических эндпоинтов допускается `httpBasic()`, а для браузерных форм — `formLogin()`. Миксуйте осознанно: вы можете создавать несколько `SecurityWebFilterChain` с разными `securityMatcher` под разные «сегменты» приложения.

Стандартные заголовки безопасности (`HSTS`, `X-Content-Type-Options`, `X-Frame-Options`, `CSP`) включаются через `headers()` и в WebFlux работают так же, как в MVC. Лучше не отключать дефолты без крайней необходимости — это ваш «минимальный» щит против распространённых классов атак.

CSRF в WebFlux по умолчанию включён для сценариев, где есть браузер, формы и cookie-сессии. Для чистого статлесс-API (только Bearer) его обычно отключают. Если у вас SPA с cookie-сессией, используйте `CookieServerCsrfTokenRepository` — он совместим с браузерами и не требует дополнительного хранения на клиенте.

CORS — это отдельная тема. В WebFlux есть реактивный `CorsConfigurationSource`, который применяет политику к preflight и основным запросам. Разводите CORS-настройки по средам (dev/test/prod) и не используйте «звёздочку» в проде — это дыра.

Наконец, помните, что `SecurityWebFilterChain` — обычный бин. Его легко тестировать интеграционными тестами через `WebTestClient`, а при необходимости — иметь две цепочки: для публичных API и для внутренних административных маршрутов.

**Gradle зависимости**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-security'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-security")
}
```

**Java — базовый `SecurityWebFilterChain` с разными зонами**

```java
package com.example.security.webflux;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityWebFilterChain apiSecurity(ServerHttpSecurity http) {
        return http
            .securityMatcher(org.springframework.security.web.server.util.matcher.PathPatternParserServerWebExchangeMatcher
                .patterns("/api/**"))
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(ex -> ex
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    @Bean
    SecurityWebFilterChain docsSecurity(ServerHttpSecurity http) {
        return http
            .securityMatcher(org.springframework.security.web.server.util.matcher.PathPatternParserServerWebExchangeMatcher
                .patterns("/actuator/**", "/docs/**"))
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(ex -> ex.anyExchange().permitAll())
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.webflux

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.Customizer
import org.springframework.security.config.web.server.ServerHttpSecurity
import org.springframework.security.web.server.SecurityWebFilterChain
import org.springframework.security.web.server.util.matcher.PathPatternParserServerWebExchangeMatcher

@Configuration
class SecurityConfig {

    @Bean
    fun apiSecurity(http: ServerHttpSecurity): SecurityWebFilterChain =
        http.securityMatcher(PathPatternParserServerWebExchangeMatcher("/api/**"))
            .csrf { it.disable() }
            .authorizeExchange {
                it.pathMatchers("/api/public/**").permitAll()
                    .pathMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyExchange().authenticated()
            }
            .httpBasic(Customizer.withDefaults())
            .build()

    @Bean
    fun docsSecurity(http: ServerHttpSecurity): SecurityWebFilterChain =
        http.securityMatcher(PathPatternParserServerWebExchangeMatcher("/actuator/**", "/docs/**"))
            .csrf { it.disable() }
            .authorizeExchange { it.anyExchange().permitAll() }
            .build()
}
```

---

## JWT/OAuth2 Resource Server в реактивном приложении, маппинг scopes→authorities

Для статлесс-API лучшая практика — «ресурс-сервер» с JWT: проверяем подпись, срок действия, аудиторию, а права маппим из токена. В WebFlux это включается одной строкой: `http.oauth2ResourceServer().jwt()`. Остальное делает авто-конфигурация, если в `application.yml` указан `issuer-uri` или `jwk-set-uri`.

Права внутри токена часто разносятся по claim’ам `scope`/`scp` (строка или массив) и `roles`/`realm_access.roles` (Keycloak). Spring Security по умолчанию преобразует `scope` в `SCOPE_xxx`, но «роли» вам нужно промаппить самостоятельно. Для реактивного стека используйте `JwtGrantedAuthoritiesConverter` + адаптер `ReactiveJwtAuthenticationConverterAdapter`.

Важно решить конвенцию: что считать «ролью» (`ROLE_...`) и что — «authority»/scope (`SCOPE_...`). Рекомендуется не смешивать: урл-правила — по ролям, методная безопасность для API — по scope, чтобы было ясно, «кто пользователь» и «что он может».

Проверка аудитории/issuer — критична. Если сервис доверяет любому токену с валидной подписью, возможна атака подмены аудитории. В Boot 3 можно задать `spring.security.oauth2.resourceserver.jwt.audiences` или проверять аудиторию программно в конвертере и бросать исключение при несовпадении.

При работе с несколькими IdP (например, B2B и внутренняя Auth-служба) удобно создать несколько `SecurityWebFilterChain`, каждая со своим `securityMatcher` и `ReactiveJwtDecoder`. Так вы избежите конфликтов маппинга и различий в claim’ах.

Осознанно задавайте clock skew — допустимое рассогласование часов. Реалистичное значение — 30–60 секунд. Это избавит от ложных 401 при «плохих» часах на клиентах.

Ошибки аутентификации должны быть единообразными: 401 с challenge `WWW-Authenticate: Bearer error="invalid_token" ...`. Boot формирует его сам, но текст причины можно улучшить обработчиком (например, добавлять `error_description` без раскрытия секретов).

Для разработки удобно использовать «симметричный» ключ HS256 и `secret` в переменной окружения; в проде — только асимметрия и JWK-ротация. Следите за kid и планируйте dual-signing на период перевыпуска ключей.

Методная безопасность с JWT — естественное расширение: `@PreAuthorize("hasAuthority('SCOPE_orders.read')")`. Это не дублирует урл-правила, а уточняет доменные операции.

И не забывайте про минимизацию токена: огромные JWT — это большие заголовки, проблемы с прокси и кэшем. Кладите только нужные claim’ы, всё остальное получайте по `userinfo`/`introspection` при необходимости.

**application.yml — автоконфигурация Resource Server**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/app
          jwk-set-uri: https://auth.example.com/realms/app/protocol/openid-connect/certs
```

**Java — маппинг scope/roles в authorities (reactive)**

```java
package com.example.security.jwt;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.server.SecurityWebFilterChain;
import reactor.core.publisher.Mono;

import java.util.*;
import java.util.stream.Stream;

@Configuration
public class JwtSecurity {

    @Bean
    SecurityWebFilterChain rsChain(ServerHttpSecurity http,
                                   Converter<Jwt, ? extends Mono<? extends AbstractAuthenticationToken>> jwtAuthConverter) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(ex -> ex
                .pathMatchers("/public/**").permitAll()
                .pathMatchers("/orders/**").hasAuthority("SCOPE_orders.read")
                .anyExchange().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtAuthConverter)))
            .build();
    }

    @Bean
    Converter<Jwt, ? extends Mono<? extends AbstractAuthenticationToken>> jwtAuthConverter() {
        var scopes = new org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter();
        scopes.setAuthorityPrefix("SCOPE_");
        scopes.setAuthoritiesClaimName("scope");

        var delegate = new org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter();
        delegate.setJwtGrantedAuthoritiesConverter(jwt -> {
            Collection<GrantedAuthority> base = scopes.convert(jwt);
            Collection<GrantedAuthority> roles = extractRoles(jwt);
            return Stream.concat(base.stream(), roles.stream()).toList();
        });

        return new ReactiveJwtAuthenticationConverterAdapter(delegate);
    }

    private Collection<GrantedAuthority> extractRoles(Jwt jwt) {
        var roles = new ArrayList<GrantedAuthority>();
        Map<String, Object> realm = jwt.getClaimAsMap("realm_access");
        if (realm != null && realm.get("roles") instanceof Collection<?> rs) {
            for (Object r : rs) {
                roles.add(new org.springframework.security.core.authority.SimpleGrantedAuthority("ROLE_" + r));
            }
        }
        return roles;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.jwt

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.convert.converter.Converter
import org.springframework.security.authentication.AbstractAuthenticationToken
import org.springframework.security.config.web.server.ServerHttpSecurity
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter
import org.springframework.security.oauth2.server.resource.authentication.ReactiveJwtAuthenticationConverterAdapter
import org.springframework.security.web.server.SecurityWebFilterChain
import reactor.core.publisher.Mono

@Configuration
class JwtSecurity {

    @Bean
    fun rsChain(
        http: ServerHttpSecurity,
        jwtAuthConverter: Converter<Jwt, out Mono<out AbstractAuthenticationToken>>
    ): SecurityWebFilterChain =
        http.csrf { it.disable() }
            .authorizeExchange {
                it.pathMatchers("/public/**").permitAll()
                    .pathMatchers("/orders/**").hasAuthority("SCOPE_orders.read")
                    .anyExchange().authenticated()
            }
            .oauth2ResourceServer { o -> o.jwt { j -> j.jwtAuthenticationConverter(jwtAuthConverter) } }
            .build()

    @Bean
    fun jwtAuthConverter(): Converter<Jwt, out Mono<out AbstractAuthenticationToken>> {
        val scopes = JwtGrantedAuthoritiesConverter().apply {
            setAuthorityPrefix("SCOPE_")
            setAuthoritiesClaimName("scope")
        }
        val delegate = JwtAuthenticationConverter().apply {
            setJwtGrantedAuthoritiesConverter { jwt ->
                val base = scopes.convert(jwt)
                val roles = extractRoles(jwt)
                (base.orEmpty() + roles)
            }
        }
        return ReactiveJwtAuthenticationConverterAdapter(delegate)
    }

    private fun extractRoles(jwt: Jwt): Collection<GrantedAuthority> {
        val realm = jwt.getClaim<Map<String, Any>>("realm_access")
        val result = mutableListOf<GrantedAuthority>()
        val roles = realm?.get("roles") as? Collection<*>
        roles?.forEach { r -> result.add(SimpleGrantedAuthority("ROLE_$r")) }
        return result
    }
}
```

---

## CSRF/CORS/headers в контексте SPA и мобильных клиентов

В SPA-сценариях (браузер + cookie-сессия) CSRF обязателен: браузер автоматически шлёт cookie, и злоумышленник может инициировать запрос с другого домена. Лучший способ в WebFlux — `CookieServerCsrfTokenRepository.withHttpOnlyFalse()`: токен кладётся в cookie и дублируется в заголовок, который SPA читает и отправляет обратно.

Если ваш API строго Bearer-стайл и клиент — мобильное приложение или другое серверное приложение, CSRF не нужен и только усложнит жизнь. В таком случае отключайте его для цепочки API-эндпоинтов, но оставляйте включённым для админ-консолей/форм.

CORS должен быть минимальным: точные `allowedOrigins`, строгое множество методов/заголовков, запрет `credentials`, если не нужны cookie. Для SPA с cookie включайте `allowCredentials=true` и прописывайте конкретный origin. В WebFlux используйте `org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource`.

HTTP-заголовки безопасности — ваш «базовый» щит. `Strict-Transport-Security` фиксирует HTTPS, `X-Content-Type-Options: nosniff` блокирует MIME-sniffing, `X-Frame-Options: DENY` защищает от clickjacking, а `Content-Security-Policy` сокращает поверхность XSS. Включайте их через `headers()`; CSP требует аккуратной настройки под ваш фронт.

Не путайте CORS и CSRF. CORS — про междоменные запросы и браузерную политику; CSRF — про подделку запроса с «чужой» страницы. Часто включают CORS и выключают CSRF для мобильных приложений и включают оба для SPA с cookie.

Preflight-запросы (OPTIONS) должны обрабатываться быстро и без аутентификации. Отдельным правилом разрешите `OPTIONS /**` и не трогайте их авторизацией — это снизит латентность клиента.

Для нескольких сред заведите профили: в `dev` можно открыть CORS на `http://localhost:3000`, а в `prod` — только на боевой домен. Не забывайте, что `*` несовместим с `allowCredentials=true`.

Если отдаёте документацию/статические файлы из того же приложения, держите для них отдельную `SecurityWebFilterChain` с безопасными заголовками и выключенным CSRF.

И наконец, логируйте попытки CORS-доступа и ошибки CSRF умеренно: это помогает отлавливать неверные конфиги фронта, но не засоряйте логи в проде бурстами preflight.

**Java — CSRF (SPA), CORS и заголовки**

```java
package com.example.security.webflux.corscsrf;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.security.web.server.csrf.CookieServerCsrfTokenRepository;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsConfigurationSource;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.time.Duration;
import java.util.List;

@Configuration
public class SpaSecurity {

    @Bean
    CorsConfigurationSource corsSource() {
        CorsConfiguration cfg = new CorsConfiguration();
        cfg.setAllowedOrigins(List.of("https://spa.example.com"));
        cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
        cfg.setAllowedHeaders(List.of("Content-Type","Authorization","X-XSRF-TOKEN"));
        cfg.setAllowCredentials(true);
        cfg.setMaxAge(Duration.ofHours(1).getSeconds());
        UrlBasedCorsConfigurationSource src = new UrlBasedCorsConfigurationSource();
        src.registerCorsConfiguration("/**", cfg);
        return src;
    }

    @Bean
    SecurityWebFilterChain spaChain(ServerHttpSecurity http, CorsConfigurationSource cors) {
        return http
            .cors(c -> c.configurationSource(cors))
            .csrf(csrf -> csrf.csrfTokenRepository(CookieServerCsrfTokenRepository.withHttpOnlyFalse()))
            .headers(h -> h
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(ServerHttpSecurity.HeaderSpec.FrameOptionsSpec::deny)
                .hsts(hsts -> hsts.includeSubdomains(true))
            )
            .authorizeExchange(ex -> ex
                .pathMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .pathMatchers("/login","/logout","/assets/**").permitAll()
                .anyExchange().authenticated()
            )
            .formLogin(ServerHttpSecurity.FormLoginSpec::disable)
            .httpBasic(ServerHttpSecurity.HttpBasicSpec::disable)
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.webflux.corscsrf

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.security.config.web.server.ServerHttpSecurity
import org.springframework.security.web.server.SecurityWebFilterChain
import org.springframework.security.web.server.csrf.CookieServerCsrfTokenRepository
import org.springframework.web.cors.CorsConfiguration
import org.springframework.web.cors.reactive.CorsConfigurationSource
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource
import java.time.Duration

@Configuration
class SpaSecurity {

    @Bean
    fun corsSource(): CorsConfigurationSource {
        val cfg = CorsConfiguration().apply {
            allowedOrigins = listOf("https://spa.example.com")
            allowedMethods = listOf("GET","POST","PUT","DELETE","OPTIONS")
            allowedHeaders = listOf("Content-Type","Authorization","X-XSRF-TOKEN")
            allowCredentials = true
            maxAge = Duration.ofHours(1).seconds
        }
        return UrlBasedCorsConfigurationSource().apply { registerCorsConfiguration("/**", cfg) }
    }

    @Bean
    fun spaChain(http: ServerHttpSecurity, cors: CorsConfigurationSource): SecurityWebFilterChain =
        http.cors { it.configurationSource(cors) }
            .csrf { it.csrfTokenRepository(CookieServerCsrfTokenRepository.withHttpOnlyFalse()) }
            .headers {
                it.contentSecurityPolicy { csp -> csp.policyDirectives("default-src 'self'") }
                    .frameOptions { fo -> fo.deny() }
                    .hsts { h -> h.includeSubdomains(true) }
            }
            .authorizeExchange {
                it.pathMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                    .pathMatchers("/login","/logout","/assets/**").permitAll()
                    .anyExchange().authenticated()
            }
            .formLogin { it.disable() }
            .httpBasic { it.disable() }
            .build()
}
```

---

## Методная безопасность: `@EnableMethodSecurity` (reactive), `ReactiveAuthorizationManager`

Методная безопасность дополняет URL-политику и живёт ближе к домену: «у пользователя есть право читать этот заказ?» В реактивном приложении она работает так же, как в MVC, но учитывает `Mono/Flux`. Аннотация `@EnableMethodSecurity` включает `@PreAuthorize/@PostAuthorize/@Secured` и использует реактивные выражения SpEL с учётом `Mono<Authentication>`.

Практическая схема: грубая фильтрация на уровне URL (`/orders/**` доступна только аутентифицированным), а на уровне сервиса — тонкие правила с `@PreAuthorize("hasAuthority('SCOPE_orders.read') and @perm.canView(#orderId)")`. Бины-помощники для SpEL (например, `@perm`) — обычные компоненты, где можно вызвать БД/кеш реактивно.

При необходимости выносите «политику на URL» из декларативного DSL в реактивный `ReactiveAuthorizationManager`. Это полезно, когда решение зависит от данных запроса (tenant, заголовки, IP) и вычисляется реактивно (например, запрос к ACL-сервису). Такой менеджер подключается через `.access(customManager)`.

Важно помнить о композиции: методные аннотации срабатывают уже после прохождения цепочки фильтров. Значит, если URL-правило пустило пользователя, но метод проверку не прошёл, вернётся 403. Это нормально и повышает наблюдаемость: в логах видно, что доступ запрещён «по бизнес-правилу».

SpEL в `@PreAuthorize` получает доступ к `authentication`, `principal`, параметрам метода (`#id`), возвращаемому значению (в `@PostAuthorize`) и бинам по имени. В реактивном мире он корректно обрабатывает `Mono/Flux`: безопасность применяется до подписки на результат.

Методная безопасность хорошо тестируется юнитами: подставляете `@WithMockUser` или мок-JWT и вызываете метод сервиса напрямую или через контроллер. Для реактивных методов используйте `StepVerifier` и проверяйте, что неавторизованный вызов завершается ошибкой `AccessDeniedException`.

Старайтесь разделять роли и права. Роли — групповые ярлыки (например, `ROLE_MANAGER`), права — конкретные действия (`SCOPE_orders.cancel`). В `@PreAuthorize` используйте права, роли оставьте в `authorizeExchange` для маршрутов. Это делает код читабельнее и убирает «магические» связи между путями и доменом.

Если решение «можно/нельзя» требует длинной реактивной цепочки (HTTP к внешнему PDP/OPA), лучше делать это в `ReactiveAuthorizationManager` на уровне URL и кэшировать на короткое время. Так вы не привяжете доменный слой к инфраструктурным задержкам.

И наконец, не забывайте про аудит отказов: логируйте контекст (пользователь, ресурс, причина) с сэмплингом — это помогает расследовать инциденты и корректно настраивать алерты.

**Java — включение методной безопасности и кастомный `ReactiveAuthorizationManager`**

```java
package com.example.security.methods;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.authorization.AuthorizationDecision;
import org.springframework.security.authorization.ReactiveAuthorizationManager;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.server.authorization.AuthorizationContext;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Configuration
@EnableMethodSecurity
class MethodSecurityConfig { }

@Service
class OrderService {

    @PreAuthorize("hasAuthority('SCOPE_orders.read')")
    public reactor.core.publisher.Mono<String> getOrder(String id) {
        return reactor.core.publisher.Mono.just("order-" + id);
    }
}

class TenantReactiveAuthorizationManager implements ReactiveAuthorizationManager<AuthorizationContext> {
    @Override
    public Mono<AuthorizationDecision> check(Mono<Authentication> authentication, AuthorizationContext context) {
        String tenant = context.getExchange().getRequest().getHeaders().getFirst("X-Tenant");
        return authentication.map(auth -> {
            boolean ok = tenant != null && auth.getAuthorities().stream()
                    .anyMatch(a -> a.getAuthority().equals("TENANT_" + tenant));
            return new AuthorizationDecision(ok);
        });
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.methods

import org.springframework.context.annotation.Configuration
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.security.authorization.AuthorizationDecision
import org.springframework.security.authorization.ReactiveAuthorizationManager
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.core.Authentication
import org.springframework.security.web.server.authorization.AuthorizationContext
import org.springframework.stereotype.Service
import reactor.core.publisher.Mono

@Configuration
@EnableMethodSecurity
class MethodSecurityConfig

@Service
class OrderService {
    @PreAuthorize("hasAuthority('SCOPE_orders.read')")
    fun getOrder(id: String): Mono<String> = Mono.just("order-$id")
}

class TenantReactiveAuthorizationManager : ReactiveAuthorizationManager<AuthorizationContext> {
    override fun check(authentication: Mono<Authentication>, context: AuthorizationContext): Mono<AuthorizationDecision> {
        val tenant = context.exchange.request.headers.getFirst("X-Tenant")
        return authentication.map { auth ->
            val ok = tenant != null && auth.authorities.any { it.authority == "TENANT_$tenant" }
            AuthorizationDecision(ok)
        }
    }
}
```

---

## Перенос аутентификации в Reactor Context (`ReactiveSecurityContextHolder`)

В реактивном приложении «текущий пользователь» хранится не в `ThreadLocal`, а в `Reactor Context`. Spring Security кладёт туда `SecurityContext`, и вы можете получить его через `ReactiveSecurityContextHolder.getContext()`. Это важно: потоки переключаются, а контекст остаётся «приклеенным» к сигналам.

Типичный сценарий — аудит: вам нужно записать `userId` в каждую запись лога или в БД. Берите `ReactiveSecurityContextHolder.getContext().map(ctx -> ctx.getAuthentication())` и доставайте `name/authorities`. Делайте это как можно ближе к месту, где информация реально нужна, чтобы не захламлять сигналы.

Иногда приходится «протолкнуть» аутентификацию вручную — например, при вызове внутренних сервисов из фонового пайплайна. Для этого есть `ReactiveSecurityContextHolder.withAuthentication(auth)`, который добавляет аутентификацию в контекст через `contextWrite`. Это полезно в тестах и при системных задачах.

Для `WebClient` перенос токена делается фильтром: считываете текущую аутентификацию из `ReactiveSecurityContextHolder` и добавляете `Authorization: Bearer`. Так downstream сервисы видят «исходного» пользователя (token relay), а не сервисный идентификатор.

Важно помнить, что контекст «теряется», если вы выходите в блокирующий код. Любой `block()`/`subscribeOn(boundedElastic())` без ручного переноса контекста отрежет безопасность. Избегайте блокировок и держите цепь реактивной от начала до конца.

В асинхронных «огибающих» (например, при использовании `Schedulers`) переносите контекст явно, если библиотека не совместима с Reactor Context. Для этого существует `Hooks.enableAutomaticContextPropagation()`, но используйте его осознанно и тестируйте.

Если вы пишете библиотеку, которая должна работать «под безопасностью», принимайте `Mono<Authentication>` или `Supplier<Mono<Authentication>>` вместо вытягивания контекста внутри — так вы не навязываете стратегию и облегчаете тестирование.

Аудит можно собирать в одном месте — веб-фильтре/методном аспекте. В реактивном мире это удобнее: у вас есть доступ ко всем сигналам, и вы можете аккуратно закрывать ресурсы в `doFinally`.

И наконец, не логируйте токены и чувствительные данные из контекста. Достаточно `principal`/`subject` и корреляционного ID. Остальное — только в системе аутентификации.

**Java — получение/установка аутентификации в реактивной цепочке**

```java
package com.example.security.context;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import reactor.core.publisher.Mono;

public class AuditService {

    public Mono<String> currentUser() {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .defaultIfEmpty("anonymous");
    }

    public Mono<String> runAs(Authentication auth, Mono<String> work) {
        return work.contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.context

import org.springframework.security.core.Authentication
import org.springframework.security.core.context.ReactiveSecurityContextHolder
import reactor.core.publisher.Mono

class AuditService {

    fun currentUser(): Mono<String> =
        ReactiveSecurityContextHolder.getContext()
            .map { it.authentication.name }
            .defaultIfEmpty("anonymous")

    fun runAs(auth: Authentication, work: Mono<String>): Mono<String> =
        work.contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth))
}
```

---

## Тестирование безопасности: `WebTestClient` + `@WithMockUser` (reactive), мок JWT

Тестировать безопасность в WebFlux лучше «чёрным ящиком» через `WebTestClient`. Вместе с `spring-security-test` вы получаете конфигураторы `SecurityMockServerConfigurers`: `mockJwt()` для ресурсо-сервера, `oauth2Login()` для клиентской стороны, `csrf()` для имитации CSRF-токена. Они не требуют реального IdP и JWK.

Позитивные кейсы: создайте `WebTestClient` и «мутирайте» его `mutateWith(mockJwt().jwt(j -> j.claim("scope","orders.read").claim("realm_access", Map.of("roles", List.of("ADMIN")))))`. Дальше проверяйте 200/контент. Это быстро и повторяемо.

Негативные кейсы: без токена ожидаем 401, с токеном без нужного scope — 403. Добавляйте проверки заголовка `WWW-Authenticate` и тела ошибки (если используете `ProblemDetail`), чтобы не допустить регресса формата.

Для методной безопасности можно обойти веб-слой и тестировать сервисы напрямую с `@WithMockUser`/`ReactiveSecurityContextHolder`. Для JWT-стиля также есть `@WithMockJwt` в сторонних аддонах, но обычно хватает конфигураторов WebTestClient.

Если у вас кастомный `JwtDecoder`, на тестах можно переопределить его через `@TestConfiguration` и отдавать заранее подписанные токены, но чаще удобнее `mockJwt()`: он не лезет в валидацию подписи, а просто создаёт `Authentication`.

Не забывайте про CSRF-кейс для SPA: `client.mutateWith(csrf())` должен быть в тесте POST/PUT/DELETE, иначе — 403. Это ловит «забытые» токены на фронте.

Для CORS-проверок используйте `WebTestClient` с заголовками `Origin`/`Access-Control-Request-Method` и убедитесь, что preflight возвращает 200/204 и нужные `Access-Control-*`. Это важно для DX фронтенда.

В e2e вы можете поднять Testcontainers-Keycloak и гонять настоящие токены по реальной схеме. Это дороже, но выявляет проблемы с конфигом IdP, маппингом ролей и ротацией ключей.

И наконец, включите `spring-security` логгер в тестах на DEBUG для быстрой диагностики: видно, кто и на каком шаге отказал (matcher, authn, authz). Это экономит часы отладки.

**Gradle test deps**
*Groovy DSL*

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

**Java — тест `WebTestClient` с mockJwt и csrf**

```java
package com.example.security.tests;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.util.List;
import java.util.Map;

import static org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.*;

@WebFluxTest
class SecurityWebTests {

    @Autowired
    WebTestClient web;

    @Test
    void shouldReturn401WithoutToken() {
        web.get().uri("/orders/1").exchange().expectStatus().isUnauthorized();
    }

    @Test
    void shouldAllowWithScope() {
        var client = web.mutateWith(mockJwt().jwt(jwt -> jwt
            .claim("scope", "orders.read")
            .claim("realm_access", Map.of("roles", List.of("ADMIN")))));
        client.get().uri("/orders/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange().expectStatus().isOk();
    }

    @Test
    void shouldRejectWithoutCsrfOnPost() {
        web.post().uri("/profile").contentType(MediaType.APPLICATION_JSON)
            .bodyValue("{\"name\":\"x\"}")
            .exchange().expectStatus().isForbidden();
    }

    @Test
    void shouldPassWithCsrfOnPost() {
        web.mutateWith(csrf()).post().uri("/profile").contentType(MediaType.APPLICATION_JSON)
            .bodyValue("{\"name\":\"x\"}")
            .exchange().expectStatus().isOk();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.tests

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.*

@WebFluxTest
class SecurityWebTests(

    @Autowired
    val web: WebTestClient
) {

    @Test
    fun shouldReturn401WithoutToken() {
        web.get().uri("/orders/1").exchange().expectStatus().isUnauthorized
    }

    @Test
    fun shouldAllowWithScope() {
        val client = web.mutateWith(mockJwt().jwt { jwt ->
            jwt.claim("scope", "orders.read")
            jwt.claim("realm_access", mapOf("roles" to listOf("ADMIN")))
        })
        client.get().uri("/orders/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange().expectStatus().isOk
    }

    @Test
    fun shouldRejectWithoutCsrfOnPost() {
        web.post().uri("/profile").contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"name":"x"}""")
            .exchange().expectStatus().isForbidden
    }

    @Test
    fun shouldPassWithCsrfOnPost() {
        web.mutateWith(csrf()).post().uri("/profile").contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"name":"x"}""")
            .exchange().expectStatus().isOk
    }
}
```

# 9. Тестирование и отладка WebFlux

## WebTestClient против MockMvc: чёрный ящик vs сервер-менее

`WebTestClient` — «родной» инструмент для WebFlux и реактивного стека; он понимает неблокирующее выполнение, backpressure и кодеки WebFlux. Это значит, что ваши тесты воспроизводят реальную модель исполнения и не дают ложноположительных результатов из-за синхронных абстракций.

В отличие от MockMvc (который завязан на Servlet API и синхронную модель), `WebTestClient` может работать как без сервера (bind-to-application/bind-to-router), так и поверх реального HTTP-стека (bind-to-server), включая Reactor Netty. Это позволяет выстраивать пирамиду тестов от быстрых unit-/slice-слоёв до интеграционных.

Режим **server-less** особенно полезен для TDD и локальной обратной связи: вы поднимаете только контроллеры/роутеры и фильтры, но избегаете накладных расходов на поднятие TCP-стека. При этом проходят все ваши `WebFilter`, конвертеры, валидация и обработчики ошибок.

Режим **bindToServer** важен для проверки конфигурации сети и безопасности: CORS, HSTS/CSP-заголовки, компрессия, шифрование, поведение Netty под тайм-аутами и keep-alive. Такие тесты дороже по времени, но ловят «экосистемные» баги, которые не видны в server-less.

Оба режима дают единый декларативный API проверок: статус, заголовки, тип контента и тело (через `expectBody`/`jsonPath`). Это стимулирует фиксировать контракты ответа (включая формат ошибок), а не ограничиваться «200 ОК» — так регресс по формату сразу станет красным тестом.

Для реактивных потоков `WebTestClient` поддерживает проверку длительности обмена и потоковых тел: можно проверять SSE/stream+json, ограничивать размер буферов и дедлайны, чтобы исключить зависшие тесты и утечки.

В WebFlux-тестах принципиально важно не использовать `block()` в тестируемом коде: `WebTestClient` и так подписывается на реактивную цепочку. Если нужно достать тело — используйте `returnResult()` и потребляйте `Mono/Flux` в тесте контролируемо, не блокируя event-loop.

Интеграция со Spring Security Test делает «чёрный ящик» полноценным: `mutateWith(mockJwt()/csrf()/oauth2Login())` позволяет моделировать аутентификацию любого вида, включая JWT и OAuth2, без реального провайдера.

Практика: держите **быстрые** server-less тесты на каждый контроллер/роутер и несколько **сквозных** bind-to-server на общую конфигурацию (Security/CORS/Actuator/метрики). Это даёт и скорость, и уверенность в продовой готовности.

Наконец, включайте DEBUG-логирование `org.springframework.web` и `reactor.netty` в «плавающих» кейсах. Это даёт видимость маршрутов, кодеков и сетевых событий и экономит часы отладки.

**Java — server-less и bind-to-server**

```java
package com.example.webflux.test;

import org.junit.jupiter.api.Test;
import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;
import static org.springframework.web.reactive.function.server.RouterFunctions.route;
import static org.springframework.web.reactive.function.server.RequestPredicates.GET;

class WebTestClientModesTest {

    RouterFunction<ServerResponse> router() {
        return route(GET("/ping"),
            req -> ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("pong"));
    }

    @Test
    void serverLessMode() {
        var client = org.springframework.test.web.reactive.server.WebTestClient
                .bindToRouterFunction(router()).configureClient().build();
        client.get().uri("/ping").exchange()
              .expectStatus().isOk()
              .expectHeader().contentTypeCompatibleWith(MediaType.TEXT_PLAIN)
              .expectBody(String.class).isEqualTo("pong");
    }

    @Test
    void bindToServerMode() {
        var httpServer = reactor.netty.http.server.HttpServer.create().port(0)
                .handle(new org.springframework.http.server.reactive.HttpHandlerDecorator(
                        org.springframework.web.server.adapter.WebHttpHandlerBuilder
                                .webHandler(new org.springframework.web.reactive.function.server.RouterFunctions
                                        .Builder().add(router()).build().toWebHandler())
                                .build()));
        var disposable = httpServer.bindNow();
        int port = disposable.port();
        try {
            var client = org.springframework.test.web.reactive.server.WebTestClient
                    .bindToServer().baseUrl("http://localhost:" + port).build();
            client.get().uri("/ping").exchange().expectStatus().isOk();
        } finally {
            disposable.disposeNow();
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.test

import org.junit.jupiter.api.Test
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.reactive.function.server.RouterFunctions
import org.springframework.web.reactive.function.server.ServerResponse
import org.springframework.web.reactive.function.server.router
import reactor.netty.http.server.HttpServer

class WebTestClientModesTest {

    private fun router() = router {
        GET("/ping") { ServerResponse.ok().contentType(MediaType.TEXT_PLAIN).bodyValue("pong") }
    }

    @Test
    fun serverLessMode() {
        val client = WebTestClient.bindToRouterFunction(router()).configureClient().build()
        client.get().uri("/ping").exchange()
            .expectStatus().isOk
            .expectHeader().contentTypeCompatibleWith(MediaType.TEXT_PLAIN)
            .expectBody(String::class.java).isEqualTo("pong")
    }

    @Test
    fun bindToServerMode() {
        val httpServer = HttpServer.create().port(0)
            .handle(org.springframework.http.server.reactive.HttpHandlerDecorator(
                org.springframework.web.server.adapter.WebHttpHandlerBuilder
                    .webHandler(RouterFunctions.Builder().add(router()).build().toWebHandler())
                    .build()
            ))
            .bindNow()
        try {
            val client = WebTestClient.bindToServer().baseUrl("http://localhost:${httpServer.port()}").build()
            client.get().uri("/ping").exchange().expectStatus().isOk
        } finally {
            httpServer.disposeNow()
        }
    }
}
```

---

## `StepVerifier` и виртуальное время для сложных последовательностей

`StepVerifier` позволяет пошагово проверять реактивные последовательности: элементы, ошибки, завершение. Это устраняет потребность в `sleep` и случайных ожиданиях, делая тесты детерминированными и быстрыми.

Сложные сценарии — ретраи с backoff, `timeout`, `debounce`, `cache(Duration)` — завязаны на время. `StepVerifier.withVirtualTime` подменяет часы, и вы «прокручиваете» минуты за миллисекунды, точно контролируя моменты повторных попыток и истечения тайм-аутов.

Чтобы виртуальное время работало, создавайте поток лениво (`Mono.defer/Flux.defer`) или передавайте фабрику в `withVirtualTime`. Иначе источник уже «просчитал» реальные таймеры до того, как StepVerifier успел их перехватить.

Отдельный нюанс — явные планировщики: если вы вызываете `publishOn(Schedulers.boundedElastic())` и делаете блокирующие операции, виртуальное время не поможет; такие части остаются привязанными к системным часам. Поэтому для тестируемой бизнес-логики полезно отделять планировщики зависимостями.

При проверке ретраев полезно явно считать количество попыток и фиксировать правило backoff (например, экспонента с джиттером). Это позволяет ловить регресс в политике повторов после обновления зависимостей.

`expectNoEvent(Duration)` и `thenAwait` — удобные средства для проверки «тишины» окна: вы можете утверждать, что до дебаунса событий не будет, а затем ожидать одно событие.

Хорошая практика — подписывать шаги через `as("label")`: в отчёте будет видно, где нарушено ожидание, что упрощает диагностику длинных последовательностей.

Для потоков с буферами и окнами (window/buffer) проверяйте не только содержание, но и **размер** и порядок групп — это ловит ошибки с неправильным управлением backpressure, которые на проде выражаются в пилах латентности.

В тестах ретраев и тайм-аутов избегайте «выпадения» на реальные системные таймеры балансировщиков. Если ваш код сочетает оператор `timeout` и внешний HTTP-тайм-аут, закладывайте это в сценарии StepVerifier.

И наконец, держите тесты краткими: проверяйте главное (число ретраев, время ожидания, тип ошибки/данных), не привязываясь к внутренним деталям реализации, чтобы не ломать тесты при рефакторинге.

**Java — `withVirtualTime` для backoff**

```java
package com.example.reactor.test;

import org.junit.jupiter.api.Test;
import reactor.core.publisher.Mono;
import reactor.test.StepVerifier;
import reactor.util.retry.Retry;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

class RetryVirtualTimeTest {

    Mono<String> flaky() {
        AtomicInteger attempts = new AtomicInteger();
        return Mono.defer(() -> {
            if (attempts.getAndIncrement() < 2) {
                return Mono.error(new RuntimeException("boom"));
            }
            return Mono.just("ok");
        }).retryWhen(Retry.backoff(2, Duration.ofMillis(100)));
    }

    @Test
    void testRetryWithVirtualTime() {
        StepVerifier.withVirtualTime(this::flaky)
            .thenAwait(Duration.ofMillis(100))
            .thenAwait(Duration.ofMillis(200))
            .expectNext("ok")
            .verifyComplete();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.reactor.test

import org.junit.jupiter.api.Test
import reactor.core.publisher.Mono
import reactor.test.StepVerifier
import reactor.util.retry.Retry
import java.time.Duration
import java.util.concurrent.atomic.AtomicInteger

class RetryVirtualTimeTest {

    private fun flaky(): Mono<String> {
        val attempts = AtomicInteger()
        return Mono.defer {
            if (attempts.getAndIncrement() < 2) Mono.error(RuntimeException("boom"))
            else Mono.just("ok")
        }.retryWhen(Retry.backoff(2, Duration.ofMillis(100)))
    }

    @Test
    fun testRetryWithVirtualTime() {
        StepVerifier.withVirtualTime { flaky() }
            .thenAwait(Duration.ofMillis(100))
            .thenAwait(Duration.ofMillis(200))
            .expectNext("ok")
            .verifyComplete()
    }
}
```

---

## Контроль блокировок: BlockHound, выявление «случайного JDBC»

BlockHound — агент, который обнаруживает **блокирующие вызовы** в потоках, где они запрещены (Reactor event-loop, параллельные планировщики). Любой `Thread.sleep/Files.readAllBytes/JDBC` без offload’а превратится в `BlockingOperationError`.

Подключать BlockHound лучше на тестах и, при желании, в dev-профиле рантайма. Это даёт гарантию, что случайный переход на «удобный» JDBC/IO не проскочит незамеченным при рефакторинге.

Важно понимать семантику: BlockHound **не запрещает** блокировки вообще; он запрещает их в «реактивных» потоках. Если вы явно отправили работу на `Schedulers.boundedElastic()`, блокировка допустима и ошибка не будет брошена.

Типичная ловушка — сторонняя библиотека, внутрь которой вы не заглядывали. Она может читать файлы или выполнять DNS-синхронно. Агент поможет обнаружить это в горячих путях — и вы примете решение: заменить библиотеку, обернуть вызовы в offload или внести точечную «поблажку» для контрольного участка инициализации.

Не превращайте конфиг в «белый список на всё»: чем шире whitelist, тем меньше защиты. Отдельные «allowBlockingCallsInside» оправданы для старта приложения, но в запросном пути лучше добиваться чистой реактивности.

Добавьте тест, намеренно вызывающий `Thread.sleep` в реактивном потоке, чтобы убедиться, что агент действительно активен и ловит нарушения. Это «smoke-check» против случайного отключения BlockHound.

В CI держите отдельный профиль, включающий BlockHound на сквозных тестах web-слоя. Это рано обнаружит попадание `block()/toIterable()` в бизнес-цепочку, что часто случается при миграции на новые версии библиотек.

Помните, что некоторые драйверы и интеграции уже имеют встроенные «адаптеры» BlockHound. Обновления могут изменить поведение — регулярный прогон тестов критичен для стабильности.

Для долгих фоновых задач лучше выносить блокирующие участки в отдельный сервис/воркер, а не мостить их на `boundedElastic`: это разгружает «боевое» приложение и предсказуемее под нагрузкой.

В логах старайтесь не печатать большие объёмы данных при ошибках BlockHound: ограничивайте сообщения и добавляйте ссылки на документацию команды, чтобы разработчикам было понятно, как чинить нарушения.

**Java — установка и проверка BlockHound**

```java
package com.example.blockhound;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import static org.junit.jupiter.api.Assertions.assertThrows;

class BlockHoundTests {

    @BeforeAll
    static void setup() {
        reactor.blockhound.BlockHound.install();
    }

    @Test
    void shouldDetectBlockingCall() {
        assertThrows(Error.class, () ->
            Mono.fromRunnable(() -> {
                try { Thread.sleep(10); } catch (InterruptedException ignored) {}
            }).publishOn(Schedulers.parallel()).block()
        );
    }

    @Test
    void allowedOnBoundedElastic() {
        Mono.fromCallable(() -> { Thread.sleep(5); return "ok"; })
            .subscribeOn(Schedulers.boundedElastic())
            .block();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.blockhound

import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import reactor.blockhound.BlockHound
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers
import kotlin.test.assertFailsWith

class BlockHoundTests {

    companion object {
        @JvmStatic @BeforeAll
        fun setup() { BlockHound.install() }
    }

    @Test
    fun shouldDetectBlockingCall() {
        assertFailsWith<Error> {
            Mono.fromRunnable { Thread.sleep(10) }
                .publishOn(Schedulers.parallel())
                .block()
        }
    }

    @Test
    fun allowedOnBoundedElastic() {
        Mono.fromCallable { Thread.sleep(5); "ok" }
            .subscribeOn(Schedulers.boundedElastic())
            .block()
    }
}
```

---

## Интеграции: WireMock/MockServer для WebClient, контракты и негативные кейсы

Изоляция исходящих вызовов — ключ к надёжным тестам: WireMock/MockServer позволяют тестировать `WebClient` без сети, воспроизводя статусы, заголовки, задержки, chunked-ответы и «битые» тела. Это критично для проверки `onStatus`, `timeout`, ретраев и маппинга ошибок.

WireMock прост и популярен для юнит-/интеграционных тестов: поднимается на случайном порту, легко задавать стабы и проверять, какие запросы пришли. MockServer мощнее по верификациям и сценариям, но вход чуть круче.

Важно переопределять базовый URL клиента через `@DynamicPropertySource` или переопределение бина, чтобы тестировалcя реальный `WebClient` со всеми фильтрами, кодеками и наблюдаемостью. Тест, который «бьёт» по старому адресу, будет всегда зелёным и бесполезным.

Негативные сценарии должны быть первоклассными: 401/403 для безопасности, 404/409/422 для валидации и конкуренции, 429 с `Retry-After` для backoff, 5xx и «битый» JSON для устойчивости десериализации. Такие кейсы встречаются чаще, чем кажется.

Задержки (`fixedDelay`, `chunkedDribbleDelay`) — обязательны: без них легко «проглядеть» нерабочие `timeout`. Провоцируйте и отмену чтения тела, чтобы проверить, что вы не буферизуете всё в память.

Контракты можно фиксировать Spring Cloud Contract или просто стабильными JSON-файлами. Контракты дороже в поддержке, но дают гарантии схем/примеров и облегчают взаимодействие команд «клиент ↔ сервер».

TLS-ветки тоже стоит покрывать: включайте self-signed сертификат в стабе и убедитесь, что ваш `HttpClient` корректно сконфигурирован (truststore, hostname verification).

Совмещайте интеграционные тесты с проверкой метрик: после 5xx должен расти счётчик ретраев и тайм-аутов. Это доказывает, что наблюдаемость не «декларативная», а реально работает.

Держите наборы стабов «по слоям»: быстрые — для happy-path, расширенные — для инцидентных сценариев. Так вы не теряете скорость на CI и сохраняете покрытие рисков.

И не забывайте чистить стабы между тестами: остаточные матчинги приводят к флакерам. Правильный lifecycle расширения или `resetAll()` — must.

**Java — WireMock + WebClient**

```java
package com.example.webclient.wiremock;

import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import org.junit.jupiter.api.RegisterExtension;
import org.junit.jupiter.api.Test;
import org.springframework.web.reactive.function.client.WebClient;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

class WireMockTests {

    @RegisterExtension
    static WireMockExtension wm = WireMockExtension.newInstance().options(
        com.github.tomakehurst.wiremock.core.WireMockConfiguration.options().dynamicPort()
    ).build();

    WebClient client() {
        return WebClient.builder().baseUrl(wm.baseUrl()).build();
    }

    @Test
    void shouldMap4xxToDomainError() {
        wm.stubFor(get(urlEqualTo("/users/42"))
            .willReturn(aResponse().withStatus(404).withHeader("Content-Type","application/json")
            .withBody("{\"code\":\"USER_NOT_FOUND\"}")));

        var ex = org.junit.jupiter.api.Assertions.assertThrows(RuntimeException.class, () ->
            client().get().uri("/users/42")
                .retrieve()
                .onStatus(s -> s.value()==404, r -> r.bodyToMono(String.class).map(RuntimeException::new))
                .bodyToMono(String.class)
                .block()
        );
        org.assertj.core.api.Assertions.assertThat(ex.getMessage()).contains("USER_NOT_FOUND");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webclient.wiremock

import com.github.tomakehurst.wiremock.junit5.WireMockExtension
import com.github.tomakehurst.wiremock.client.WireMock.*
import com.github.tomakehurst.wiremock.core.WireMockConfiguration
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.RegisterExtension
import org.springframework.web.reactive.function.client.WebClient

class WireMockTests {

    companion object {
        @JvmField
        @RegisterExtension
        val wm: WireMockExtension = WireMockExtension.newInstance()
            .options(WireMockConfiguration.options().dynamicPort())
            .build()
    }

    private fun client(): WebClient = WebClient.builder().baseUrl(wm.baseUrl()).build()

    @Test
    fun shouldMap4xxToDomainError() {
        wm.stubFor(get(urlEqualTo("/users/42"))
            .willReturn(aResponse().withStatus(404).withHeader("Content-Type","application/json")
                .withBody("""{"code":"USER_NOT_FOUND"}""")))

        val ex = assertThrows(RuntimeException::class.java) {
            client().get().uri("/users/42")
                .retrieve()
                .onStatus({ it.value()==404 }) { r -> r.bodyToMono(String::class.java).map { RuntimeException(it) } }
                .bodyToMono(String::class.java)
                .block()
        }
        assert(ex.message!!.contains("USER_NOT_FOUND"))
    }
}
```

---

## Наблюдаемость: Micrometer timers, распределённые трейсы, корреляция request-id

В Boot 3 наблюдаемость встроена в WebFlux и WebClient через Observations/Micrometer и мосты трейсинга (OTel/Zipkin/Jaeger). Это значит, что без вашей ручной работы появятся таймеры `http.server.requests/http.client.requests` и спаны «сервер/клиент».

Тесты наблюдаемости полезны не реже функциональных: они доказывают, что таймеры реально инкрементятся и содержат ожидаемые теги (`uri`, `status`, `outcome`, `exception`). В юнитах достаточно `SimpleMeterRegistry`; для интеграций — подключайте Prometheus-регистр и проверяйте экспозицию.

Корреляция `trace-id` важна для сквозной отладки: сервер ставит/прокидывает `traceparent`, `WebClient` подхватывает и создаёт дочерний спан. Тесты могут проверять наличие заголовка на исходящем запросе или использовать in-memory экспортер спанов.

Следите за кардинальностью тегов: URL и сырые идентификаторы не должны попадать в теги как есть — иначе вы «взорвёте» TSDB. В тестах фиксируйте whitelist тегов и убеждайтесь, что `uri` нормализован (шаблонный, а не «/orders/12345»).

Метрики ошибок — основа алертинга: доля 5xx/4xx, тайм-ауты, ретраи, отказ сети. Негативные тесты с WireMock, которые генерируют 429/5xx/время, должны приводить к росту соответствующих счётчиков/таймеров.

Для долгих потоков фиксируйте и проверяйте гистограммы: tail-latency (p95/p99) важна в реактивных приложениях из-за backpressure и burst-нагрузки. Даже простая проверка «есть ли histogram» предотвращает случайное отключение.

Request-ID — отдельная, но родственная практика: генерируйте `X-Request-Id` на входе и возвращайте его в ответах. В тестах проверяйте, что заголовок действительно присутствует — это сильно ускоряет разбор инцидентов.

В e2e-режиме можно поднять OTLP no-op экспортер и проверять, что спаны создаются и имеют ожидаемую иерархию (родитель/ребёнок). Это несложно, но даёт уверенность после апгрейдов Spring.

Не забывайте требование безопасности: заголовки/тела с секретами не должны попадать в логи/спаны. В тестах убедитесь, что фильтры маскируют `Authorization/Set-Cookie` и чувствительные поля.

В проде держите сэмплирование трейсинга умеренным (1–10%). В тестах чаще включают 100%, чтобы не потерять спаны в проверках; это нормально для CI.

**Java — проверка кастомной метрики клиента**

```java
package com.example.observability;

import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.Test;
import org.springframework.web.reactive.function.client.*;

import static org.assertj.core.api.Assertions.assertThat;

class ClientMetricsTest {

    @Test
    void shouldRecordTimer() {
        var meters = new SimpleMeterRegistry();
        ExchangeFilterFunction metrics = (req, next) -> {
            long start = System.nanoTime();
            return next.exchange(req).doOnSuccess(resp -> {
                meters.timer("client.partner", "uri", req.url().getPath(),
                        "status", resp != null ? String.valueOf(resp.statusCode().value()) : "ERR")
                    .record(System.nanoTime() - start, java.util.concurrent.TimeUnit.NANOSECONDS);
            });
        };
        var client = WebClient.builder().filter(metrics).baseUrl("http://example.invalid").build();
        try {
            client.get().uri("/x").retrieve().toBodilessEntity().block();
        } catch (Exception ignored) { }

        assertThat(meters.find("client.partner").timer()).isNotNull();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.observability

import io.micrometer.core.instrument.simple.SimpleMeterRegistry
import org.junit.jupiter.api.Test
import org.springframework.web.reactive.function.client.ExchangeFilterFunction
import org.springframework.web.reactive.function.client.WebClient
import kotlin.test.assertNotNull

class ClientMetricsTest {

    @Test
    fun shouldRecordTimer() {
        val meters = SimpleMeterRegistry()
        val metrics = ExchangeFilterFunction { req, next ->
            val start = System.nanoTime()
            next.exchange(req).doOnSuccess { resp ->
                meters.timer("client.partner", "uri", req.url().path, "status",
                    resp?.statusCode()?.value()?.toString() ?: "ERR")
                    .record(System.nanoTime() - start, java.util.concurrent.TimeUnit.NANOSECONDS)
            }
        }
        val client = WebClient.builder().filter(metrics).baseUrl("http://example.invalid").build()
        runCatching { client.get().uri("/x").retrieve().toBodilessEntity().block() }

        assertNotNull(meters.find("client.partner").timer())
    }
}
```

---

## Testcontainers c R2DBC/БД, изоляция и миграции (Flyway/Liquibase) перед тестом

Реальная БД в тестах — лучшая профилактика «сюрпризов» на проде: Testcontainers поднимает Postgres/Mongo/Cassandra на случайном порту и даёт чистое окружение на каждый сьют. Для R2DBC достаточно прокинуть `spring.r2dbc.url/username/password` через `@DynamicPropertySource`.

Миграции должны запускаться **обязательно**: схема тестовой БД обязана соответствовать production. Самый простой путь — подключить `flyway-core`/`liquibase-core` и вызывать миграции в `@BeforeAll`, используя JDBC-URL контейнера.

Хорошая практика — общий базовый класс тестов БД, который поднимает контейнер ровно один раз на класс и делится URL-ами через `DynamicPropertySource`. Это экономит минуты на CI, не жертвуя изоляцией.

Инициализация данных должна быть идемпотентной: либо `truncate + insert`, либо фикстуры, загружаемые реактивно через R2DBC. Избегайте «косвенных» зависимостей между тестами — тесты должны быть независимыми и повторяемыми.

Для производительности включайте `reuse` локально (на CI он отключён): так вы ускорите разработку, но в коммитах это не должно влиять на изоляцию. Следите за версией образа БД (например, `postgres:16-alpine`) — закрепляйте теги.

В тестах, которые проверяют пагинации/стриминг, используйте **реалистичные объёмы**. Маленькие таблички скрывают проблемы планов, индексов и «горячих» партиций; контейнер легко позволяет генерировать десятки тысяч строк.

С R2DBC избегайте `block()` даже в тестах: читайте/пишите реактивно и используйте `StepVerifier`/`WebTestClient`. Блокировки маскируют взаимодействие пула и драйвера и искажают результаты.

Разносите профили: на интеграциях включайте лог SQL и показатели пула (`acquire time, active, idle`). Это помогает найти узкие места (слишком маленький пул, неудачные тайм-ауты, долгие планы).

Если вы комбинируете WebFlux + R2DBC + внешние зависимые сервисы, поднимайте их тоже в контейнерах (Keycloak, Kafka, MinIO). Вертикальная интеграция ловит сложные проблемы конфигов и совместимости.

Не забывайте про TZ/locale: задайте `TZ=UTC` контейнеру и приложению в тестах, чтобы вычисления дат/сортировки были детерминированы. Это особенно важно при сравнении времён и `timestamp with time zone`.

**Java — Postgres Testcontainers + Flyway + R2DBC url**

```java
package com.example.r2dbc.tc;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.TestInstance;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.containers.PostgreSQLContainer;

@Testcontainers
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class R2dbcPostgresBaseTest {

    @Container
    static final PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("app").withUsername("app").withPassword("secret");

    @BeforeAll
    void migrate() {
        Flyway.configure().dataSource(pg.getJdbcUrl(), pg.getUsername(), pg.getPassword())
                .locations("classpath:db/migration").load().migrate();
    }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.r2dbc.url", () ->
                "r2dbc:postgresql://" + pg.getHost() + ":" + pg.getFirstMappedPort() + "/" + pg.getDatabaseName());
        r.add("spring.r2dbc.username", pg::getUsername);
        r.add("spring.r2dbc.password", pg::getPassword);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.r2dbc.tc

import org.flywaydb.core.Flyway
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.TestInstance
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@Testcontainers
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
abstract class R2dbcPostgresBaseTest {

    companion object {
        @JvmField @Container
        val pg = PostgreSQLContainer("postgres:16-alpine")
            .withDatabaseName("app").withUsername("app").withPassword("secret")
    }

    @BeforeAll
    fun migrate() {
        Flyway.configure().dataSource(pg.jdbcUrl, pg.username, pg.password)
            .locations("classpath:db/migration").load().migrate()
    }

    companion object {
        @JvmStatic
        @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("spring.r2dbc.url") {
                "r2dbc:postgresql://${pg.host}:${pg.firstMappedPort}/${pg.databaseName}"
            }
            r.add("spring.r2dbc.username") { pg.username }
            r.add("spring.r2dbc.password") { pg.password }
        }
    }
}
```

# 10. Продакшн-паттерны и производительность

## Тюнинг reactor-netty: event-loops, размеры пулов/буферов, keep-alive, H2C/H2

В продакшне производительность WebFlux во многом определяется настройкой Reactor Netty — неблокирующего HTTP-сервера/клиента под капотом. По умолчанию он берёт число event-loop потоков, равное количеству ядер, что хорошо для большинства нагрузок. Однако на узких CPU или в условиях множественных pod’ов на одном узле иногда полезно ограничить event-loops до 1–2 потоков на pod, чтобы не конкурировать с соседями. Это особенно заметно при высокой сетевой конкуррентности и небольших «единицах работы».

Размеры внутренних очередей и буферов влияют на латентность и потребление памяти. Слишком маленькие буферы приводят к фрагментации потоков и частым переключениям контекста, слишком большие — к росту tail-latency из-за накопления «хвостов». В Reactor Netty можно настраивать `maxInitialLineLength`, `maxHeaderSize`, `initialBufferSize` и стратегию аллокатора ByteBuf, чтобы избежать лишних копирований и перерасхода памяти на большие заголовки.

Поддержание соединений (keep-alive) экономит RTT и CPU на установку TCP/SSL, но требует аккуратного таймаута простаивающих коннектов. Для сервера важно не держать «мертвые» keep-alive слишком долго, чтобы не выгореть по дескрипторам; для клиента — держать пул соединений под контролем, настраивая idle-timeout и max-connections. Грамотно выставленные таймауты сокращают пилообразность латентности под нагрузкой.

HTTP/2 и H2C (HTTP/2 без TLS) дают мультиплексирование поверх одного TCP, уменьшая количество системных ресурсов на коннекты и повышая пропускную способность. У HTTP/2 другой рисунок задержек и свой набор таймаутов (pings), а также требования к TLS-стеку. В продакшне чаще включают H2 поверх TLS, для которого рекомендуется `netty-tcnative-boringssl-static`, чтобы получить быструю и совместимую криптографию.

Сжатие ответов помогает экономить трафик и время передачи, но добавляет CPU-нагрузку и может вредить tail-latency при больших ответах без backpressure. Для API с мелкими JSON полезно включать компрессию с порогом по размеру ответа и с исключениями по типам контента. Комбинируйте это с грамотной сериализацией (без «жирных» Jackson-модулей), чтобы не удваивать расходы на CPU.

Таймауты подключения, ожидания запроса и ответа — ваш второй уровень защиты от зависаний. На сервере имеет смысл задавать read/write timeout и ограничение времени на заголовок; на клиенте — connect/read/response timeout и общий `responseTimeout`. Такое «обрамление» ловит сетевые хвосты и выдерживает SLA под аварийную деградацию зависимостей.

Пулы соединений клиента и сервера должны соответствовать пикам параллелизма ваших сценариев. Для коротких запросов с высокой конкуррентностью лучше меньше conn-per-host и агрессивнее reuse, для долгоживущих потоков — наоборот, ограничивайте коннекты и держите их «закреплёнными» за подписчиками. Следите метриками пула, чтобы не делать «магии» вслепую.

Сетевые опции Netty (TCP_NODELAY, SO_BACKLOG, SO_RCVBUF/SO_SNDBUF) задают фундаментальный характер IO. `TCP_NODELAY` полезен для мелких JSON-ответов, отключая Nagle; `SO_BACKLOG` под пиками защищает от временного отказа accept; буферы стоит подгонять под MTU/RTT реального канала, но не переусердствовать. Эти параметры требовательны к валидации на стендах.

Для крупной инсталляции полезно разделить «горячие» и «админские» цепочки: отдельная `SecurityWebFilterChain`, другой порт/хост, иные таймауты и лимиты. Так вы не дадите тяжёлым админским операциям разогнать GC и сеть, влияя на боевые запросы.

Наконец, любые изменения сетевых настроек должны делаться под наблюдением. Снимайте p50/p95/p99 latency, ошибки, saturation пула/CPU и сопоставляйте с конфигом. Без метрик тюнинг — гадание.

**Gradle зависимости (H2/H2C + TLS; добавить к стандартному WebFlux)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'io.projectreactor.netty:reactor-netty-http'
    runtimeOnly 'io.netty:netty-tcnative-boringssl-static:2.0.65.Final'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("io.projectreactor.netty:reactor-netty-http")
    runtimeOnly("io.netty:netty-tcnative-boringssl-static:2.0.65.Final")
}
```

**Java — кастомизация Reactor Netty сервера (HTTP/2, таймауты, заголовки)**

```java
package com.example.net.tuning;

import io.netty.channel.ChannelOption;
import reactor.netty.http.Http11SslContextSpec;
import reactor.netty.http.server.HttpServer;
import reactor.netty.resources.LoopResources;
import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.net.ssl.SSLException;
import java.time.Duration;

@Configuration
public class NettyServerTuningConfig {

    @Bean
    public WebServerFactoryCustomizer<NettyReactiveWebServerFactory> nettyCustomizer() {
        return factory -> factory.addServerCustomizers(httpServer -> {
            HttpServer tuned = httpServer
                .http2(true)
                .compress(true)
                .idleTimeout(Duration.ofSeconds(30))
                .tcpConfiguration(tcp -> tcp
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                )
                .wiretap(false);
            return tuned;
        });
    }

    // Пример: кастомные event-loops (опционально)
    @Bean(destroyMethod = "dispose")
    public LoopResources appLoops() {
        return LoopResources.create("app-loops", 2, true);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.net.tuning

import io.netty.channel.ChannelOption
import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory
import org.springframework.boot.web.server.WebServerFactoryCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import reactor.netty.http.server.HttpServer
import reactor.netty.resources.LoopResources
import java.time.Duration

@Configuration
class NettyServerTuningConfig {

    @Bean
    fun nettyCustomizer(): WebServerFactoryCustomizer<NettyReactiveWebServerFactory> =
        WebServerFactoryCustomizer { factory ->
            factory.addServerCustomizers { http: HttpServer ->
                http.http2(true)
                    .compress(true)
                    .idleTimeout(Duration.ofSeconds(30))
                    .tcpConfiguration { tcp ->
                        tcp.option(ChannelOption.SO_BACKLOG, 1024)
                           .option(ChannelOption.TCP_NODELAY, true)
                           .option(ChannelOption.SO_KEEPALIVE, true)
                    }
                    .wiretap(false)
            }
        }

    @Bean(destroyMethod = "dispose")
    fun appLoops(): LoopResources = LoopResources.create("app-loops", 2, true)
}
```

**application.yml — общие параметры**

```yaml
server:
  http2:
    enabled: true
  reactive:
    session:
      timeout: 30m
```

---

## Управление backpressure: ограничение параллелизма, `limitRate`, оконные операторы

Backpressure — механизм, позволяющий потребителю диктовать темп производителю, чтобы избежать переполнений и всплесков памяти. В реактивных конвейерах нужно сознательно ограничивать параллелизм и размер очередей: «побыстрее» не всегда лучше, чаще «честно и ровно». Без этого под нагрузкой вы увидите растущий p99 и «зубья» латентности.

Оператор `flatMap` по умолчанию стремится к максимальной параллельности в рамках upstream скорости. Если внутренняя работа I/O-тяжёлая, задайте ограничение `concurrency` и «запас» по предзагрузке `prefetch`. Это сохраняет shape конвейера и не даёт распухать очередям при burst-нагрузках.

`limitRate` на завершении цепочки — быстрый способ сказать «отгружай по N элементов, не больше». Он полезен, когда downstream — сетевой ответ клиенту или медленное сериализующее звено. В паре с ним иногда нужно ограничить size буферов в кодеках/сервере — иначе вы просто переместите давление.

Оконные операторы `window`/`buffer` помогают «резать» поток на управляемые чанки. Это удобно для батчевых вставок/отправок: обрабатываете по 100–500 элементов, гарантируя верхнюю границу памяти и стабильный темп. Старайтесь, чтобы размер окна отвечал реальной I/O-емкости зависимых систем.

Если источник не умеет backpressure (горячий источник, таймер), используйте `onBackpressureBuffer`/`drop`/`latest` по ситуации. Буфер опасен без верхней границы; дроп/latest — осознанная деградация для телеметрии/нотификаций, где лучше потерять событие, чем убить сервис.

Не путайте параллелизм с конкурентной безопасностью. `flatMap` с большим concurrency не означает безопасность доступа к общим структурам. Все мутирующие объекты либо изолируйте, либо защищайте, либо переработайте на неизменяемые модели.

Параллелизм стоит задавать от обратного: «что выдержит зависимость?» Если у вас пул БД на 50 соединений, нет смысла ставить concurrency 1_000 — вы получите лишь рост очереди и p99. Стройте конвейер от SLA внешних систем и профилируйте поведение под разной нагрузкой.

Backpressure — не только про скорость, но и про память. Считайте килобайты на элемент и умножайте на глубину очереди. Даже простой `collectList()` легко превратить 10М маленьких объектов в OOM. Там, где нужен «список», часто нужен стриминг или курсор.

В потоковых ответах клиента (SSE/WS) ограничивайте rate на отправку. Иначе недорогая в целом подписка может стать «дырой» памяти, если получатель не успевает и не закрывает соединение. Применяйте `limitRate`, пульсирующие окна и таймауты неактивности.

В конце дня backpressure — дисциплина. Встраивайте лимиты в каждое звено, проверяйте их тестами под нагрузкой и наблюдайте очереди/буферы. Это и есть «производительность, которая предсказуема».

**Java — осознанный параллелизм и лимитирование темпа**

```java
package com.example.backpressure;

import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;

import java.time.Duration;

public class BackpressureDemo {

    public Flux<String> process(Flux<Integer> source) {
        return source
            .windowTimeout(100, Duration.ofMillis(200))
            .flatMap(batch -> batch.collectList(), 4) // ограничили параллелизм до 4
            .flatMap(list -> Flux.fromIterable(list)
                   .flatMap(this::ioBoundOp, 8, 16))   // 8 параллельно, префетч 16
            .limitRate(256);                           // отдаём по 256 элементов за запрос
    }

    private Flux<String> ioBoundOp(Integer i) {
        return Flux.defer(() -> Flux.just("ok-" + i)).subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.backpressure

import reactor.core.publisher.Flux
import reactor.core.scheduler.Schedulers
import java.time.Duration

class BackpressureDemo {

    fun process(source: Flux<Int>): Flux<String> =
        source
            .windowTimeout(100, Duration.ofMillis(200))
            .flatMap({ batch -> batch.collectList() }, 4)
            .flatMap({ list ->
                Flux.fromIterable(list)
                    .flatMap({ i -> ioBoundOp(i) }, 8, 16)
            })
            .limitRate(256)

    private fun ioBoundOp(i: Int): Flux<String> =
        Flux.defer { Flux.just("ok-$i") }.subscribeOn(Schedulers.boundedElastic())
}
```

---

## Долгоживущие соединения: SSE/WebSocket/RSocket — когда что выбрать

Server-Sent Events (SSE) — однонаправленный канал «сервер → клиент» поверх HTTP, удобный для стриминга обновлений. Он прост, хорошо работает за прокси и масштабируется по числу подписок при скромных требованиях. Для внутренних панелей и пуш-событий SSE часто лучший выбор.

WebSocket — двунаправленный full-duplex канал, уместный для интерактивных приложений: чаты, совместное редактирование, игровые лупы. Он требует аккуратного управления ресурсами: каждый сокет — это держатель дескриптора и буферов, и под сотни тысяч соединений понадобится специфичная архитектура и балансировка.

RSocket — бинарный протокол поверх TCP/WebSocket с поддержкой запрос-ответа, потоков и «каналов», умеющий multiplex и backpressure на уровне протокола. Для микросервисов он хорош, когда нужен реактивный RPC с «живыми» подписками и требуются предсказуемые характеристики на нестабильных сетях.

SSE удобен тем, что проксируется, кэшируется и «понимается» браузером без библиотек. Он переживает перезапуски через «Last-Event-ID» и прост в дебаге. Его главный минус — только «в одну сторону», а также шумный формат для бинарных данных.

WebSocket сложнее для продакшна: sticky sessions, health-пробы, недружелюбные прокси, необходимость продуманного масштабирования по числу коннектов. При этом это единственный простой путь для «настоящего» двустороннего обмена в браузере.

RSocket создаёт новую плоскость: единый транспорт для запросов и стримов между сервисами. Он требует инфраструктурного стандарта и поддержки в наблюдаемости. Для BFF, который агрегирует стримы от нескольких бэкендов, RSocket часто оказывается более экономным, чем парк HTTP подписок.

С точки зрения WebFlux, все три варианта интегрируются «родным» образом. SSE — это `Flux<ServerSentEvent<T>>` из контроллера и корректный `Content-Type`. WebSocket — это `WebSocketHandler`, реагирующий на входящие сообщения и отправляющий исходящие реактивно. RSocket — это `spring-boot-starter-rsocket` и маршруты/`RSocketRequester`.

Выбор делаем по требованиям домена: если нужен только push на фронт — SSE; если двусторонний real-time — WebSocket; если межсервисные стримы и RPC с backpressure — RSocket. Не забывайте про метрики и таймауты — долгоживущие коннекты без наблюдаемости превращаются в «чёрные дыры».

Наконец, любой длительный канал нуждается в стратегии восстановления: heartbeat/ping, реконнект с экспонентой, возобновление с offset/`Last-Event-ID`. Это часть SLA, а не «надстройка сверху».

**Gradle зависимости (SSE/WS/RSocket)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-websocket'
    implementation 'org.springframework.boot:spring-boot-starter-rsocket'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-websocket")
    implementation("org.springframework.boot:spring-boot-starter-rsocket")
}
```

**Java — SSE и WebSocket handler**

```java
package com.example.longconn;

import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.reactive.socket.WebSocketHandler;
import org.springframework.web.reactive.socket.WebSocketSession;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;

@Controller
public class StreamControllers implements WebSocketHandler {

    @GetMapping(path = "/sse/clock", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> sseClock() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(tick -> ServerSentEvent.builder("tick-" + tick).id(String.valueOf(tick)).build());
    }

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        Flux<String> out = session.receive().map(msg -> "echo-" + msg.getPayloadAsText());
        return session.send(out.map(session::textMessage));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.longconn

import org.springframework.http.MediaType
import org.springframework.http.codec.ServerSentEvent
import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.reactive.socket.WebSocketHandler
import org.springframework.web.reactive.socket.WebSocketSession
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import java.time.Duration

@Controller
class StreamControllers : WebSocketHandler {

    @GetMapping("/sse/clock", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun sseClock(): Flux<ServerSentEvent<String>> =
        Flux.interval(Duration.ofSeconds(1))
            .map { tick -> ServerSentEvent.builder("tick-$tick").id("$tick").build() }

    override fun handle(session: WebSocketSession): Mono<Void> {
        val out = session.receive().map { "echo-" + it.payloadAsText }
        return session.send(out.map { session.textMessage(it) })
    }
}
```

**application.yml — RSocket сервер**

```yaml
spring:
  rsocket:
    server:
      port: 7000
      transport: websocket
```

---

## Резилиентность: тайм-ауты, `retryWhen`, circuit-breaker (Resilience4j reactive)

Резилиентность — это заранее спроектированная реакция на неисправности зависимостей. В реактивном мире это удобнее: таймауты, ретраи с backoff и circuit-breaker становятся частью цепочки, не блокируя потоки. Главное — применять их дозированно и осознанно.

Таймауты ставьте на каждом I/O-звене: HTTP-клиент, БД, кэш. На `WebClient` есть `responseTimeout`; на уровне конвейера — `.timeout(Duration)`. Первый таймаут обрывает транспорт, второй — бизнес-операцию, даже если транспорт завис. Это разные контуры защиты и оба нужны.

Повторные попытки применимы только к идемпотентным операциям и с ограниченным числом шагов. Используйте `Retry.backoff`/`fixedDelay` с джиттером, чтобы не синхронизировать штурм отключившегося партнёра. На многие ошибки (4xx) ретраи не нужны; их отфильтровывают предикатом.

Circuit-breaker размыкает «плохое» направление, не давая забить очереди и сжечь потоки. Resilience4j имеет реактивные декораторы, которые оборачивают `Mono/Flux`. В открытом состоянии конвейер быстро «отстреливает» отказ, а в half-open аккуратно пробует восстановиться.

Bulkhead ограничивает параллелизм на чувствительные участки, не давая «прожорливой» зависимости вытеснить остальных. В реактивном API это семантически похоже на backpressure, но с явной политикой отказа при превышении «мест» в отсеке.

Rate-limiter с токен-бакетом выравнивает нагрузку на партнёров. В обвязке `WebClient` это просто фильтр, который при нехватке токенов задерживает/отказывает вызову. Это спасает от банов и защищает SLA.

Ошибка — «слепые» ретраи. Если вы ретраите таймаут без circut-breaker и без лимита конкуррентности, вы лишь усугубляете навал. Правильная композиция: лимит параллелизма → таймаут → circuit-breaker → ретраи на быстрые отказоустойчивые ошибки.

Наблюдаемость — часть резилиентности. Счётчики ретраев, открытий breaker’а, доля timeouts, средняя длительность удачных вызовов — эти метрики должны быть в алертах. И лучше один «шумный» алерт на открытие breaker’а, чем сотни таймаутов на графике.

Серверные ошибки оборачивайте в стабильный формат (Problem Details) с кодами, дружелюбными клиенту. Это меньше нагружает клиента «догадками» и помогает быстрее восстановить сервис.

Наконец, держите «ручной тормоз»: фича-флаги или конфиг, который на лету снижает concurrency/выключает ретраи. В авариях это экономит минуты.

**Gradle зависимости (Resilience4j reactive)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'io.github.resilience4j:resilience4j-reactor'
    implementation 'io.github.resilience4j:resilience4j-spring-boot3'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("io.github.resilience4j:resilience4j-reactor")
    implementation("io.github.resilience4j:resilience4j-spring-boot3")
}
```

**Java — WebClient с таймаутом, retryWhen и circuit-breaker**

```java
package com.example.resilience;

import io.github.resilience4j.circuitbreaker.*;
import io.github.resilience4j.reactor.circuitbreaker.operator.CircuitBreakerOperator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.*;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;

import java.time.Duration;

@Configuration
public class ResilienceConfig {

    @Bean
    WebClient resilientClient() {
        CircuitBreaker cb = CircuitBreaker.ofDefaults("partner");
        return WebClient.builder()
            .baseUrl("https://partner.example.com")
            .filter((req, next) -> next.exchange(req)
                .timeout(Duration.ofSeconds(2))
                .transformDeferred(CircuitBreakerOperator.of(cb))
                .retryWhen(Retry.backoff(2, Duration.ofMillis(100))
                    .filter(ex -> ex instanceof WebClientResponseException.ServiceUnavailable)))
            .build();
    }

    public Mono<String> call(WebClient client) {
        return client.get().uri("/api")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.resilience

import io.github.resilience4j.circuitbreaker.CircuitBreaker
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig
import io.github.resilience4j.reactor.circuitbreaker.operator.CircuitBreakerOperator
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.WebClientResponseException
import reactor.core.publisher.Mono
import reactor.util.retry.Retry
import java.time.Duration

@Configuration
class ResilienceConfig {

    @Bean
    fun resilientClient(): WebClient {
        val cb = CircuitBreaker.ofDefaults("partner")
        return WebClient.builder()
            .baseUrl("https://partner.example.com")
            .filter { req, next ->
                next.exchange(req)
                    .timeout(Duration.ofSeconds(2))
                    .transformDeferred(CircuitBreakerOperator.of(cb))
                    .retryWhen(
                        Retry.backoff(2, Duration.ofMillis(100))
                            .filter { it is WebClientResponseException.ServiceUnavailable }
                    )
            }
            .build()
    }

    fun call(client: WebClient): Mono<String> =
        client.get().uri("/api").retrieve().bodyToMono(String::class.java)
}
```

**application.yml — пример настроек Resilience4j**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      partner:
        slidingWindowSize: 50
        permittedNumberOfCallsInHalfOpenState: 5
        failureRateThreshold: 50
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 50
```

---

## Профилирование и перф-аудит: flame-графы, reactor-debug, метрики на операцию

Профилирование в реактивных приложениях требует инструментов, понимающих асинхронную модель. Для «горячих» путей используйте async-profiler/JFR, которые строят flame-графы по CPU/allocations и видят Netty/Reactor стеки. Это помогает найти самые «дорогие» операторы, сериализацию и конвертацию.

`reactor-tools` и `Hooks.onOperatorDebug()` дают ассембли-трейсы — обратные ссылки от подписки к месту сборки конвейера. В продакшне это дорого, но в dev/стендах помогает понять, откуда рождается ошибка или утечка. Альтернатива — точечные `.checkpoint("label")`, которые почти не бьют по перфу и улучшают диагностику.

Наблюдаемость на уровне операций — не только HTTP-таймеры. Оборачивайте доменные конвейеры в `Observation`/Micrometer timers: вы увидите распределение времени по важным шагам — чтение БД, вызов партнёра, сериализация. Это помогает не гадать, а знать, куда уходит время запроса.

Логирование с корреляцией trace-id/request-id — must-have для расследований. В реактивном мире позаботьтесь о переносе MDC (в Boot 3 это встроено через Observability), чтобы логи всей цепочки имели один и тот же trace. Тогда flame-графы и логи складываются в единую картину.

Веб-сервер и клиент уже снабжены метриками, но вам нужны и собственные: счётчики ретраев, ошибок парсинга, гистограммы размерности тел. Чем ближе метрика к реальной бизнес-операции, тем легче держать SLA.

При анализе GC используйте профилировщики, умеющие JFR, и смотрите не только паузы, но и аллокации на оператор. Часто «починить перф» означает убрать лишние маппинги/конвертации, а не «подкрутить GC». Снижение аллокаций даёт стабильный p99.

В продакшне лучше сэмплировать: включите 1–10% трейсов, соберите метрики, а полный профайлинг включайте эпизодически. Любые тяжёлые флаги (`onOperatorDebug`) — только под фича-флагом и на короткое окно.

Ревью перфа — регулярная практика. Раз в спринт прогоняйте нагрузочный профиль на стенде, снимайте flame-графы до/после оптимизаций, сверяйте метрики. Это не только «срочная помощь», но и плановая профилактика деградаций после апгрейдов библиотек.

Наконец, держите «перф-библиотеку»: каталожные паттерны и анти-паттерны с примерами, чек-листы перед релизом (включая тюнинг netty/буферов). Это экономит время команды и делает знания коллективными.

**Gradle зависимости (reactor-tools для ассембли-трейсов)**
*Groovy DSL*

```groovy
dependencies {
    runtimeOnly 'io.projectreactor:reactor-tools'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    runtimeOnly("io.projectreactor:reactor-tools")
}
```

**Java — включение operator-debug только в dev и checkpoint**

```java
package com.example.perf;

import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Hooks;

@Component
@Profile("dev")
public class ReactorDebug {
    @PostConstruct
    public void enable() {
        Hooks.onOperatorDebug();
    }

    public Flux<String> pipeline(Flux<String> in) {
        return in.map(String::trim)
                 .checkpoint("after-trim")
                 .map(String::toUpperCase);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.perf

import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component
import reactor.core.publisher.Flux
import reactor.core.publisher.Hooks

@Component
@Profile("dev")
class ReactorDebug {

    @PostConstruct
    fun enable() { Hooks.onOperatorDebug() }

    fun pipeline(input: Flux<String>): Flux<String> =
        input.map(String::trim)
            .checkpoint("after-trim")
            .map(String::uppercase)
}
```

---

## Деплой и окружение: контейнеры, k8s ресурсы/пробы, graceful shutdown и drain

Контейнеризация задаёт предсказуемое окружение и облегчает тюнинг JVM/Netty. Для реактивных сервисов выгодно использовать JDK 21 и легковесные образы (alpine/distroless), но с оглядкой на нативные зависимости (TLS/ICU). Переносите тюнинг через переменные окружения и `application.yml`, чтобы не править образ при каждой мелочи.

В Kubernetes ресурсы — часть производительности. Ставьте **requests** на уровень, гарантирующий вам CPU-квоту для event-loop; **limits** — с небольшим запасом, чтобы не ловить throttling под пиками. Реактивный сервис лучше переносит ограниченный CPU, чем «троттлинг каждые 10 мс».

Пробы здоровья должны пониматься приложением. Включайте `readiness` для зависимости от БД/кэша/IdP, чтобы трафик шёл только на готовые поды. `liveness` должна ловить «залипание» и утечки; для Reactor Netty достаточно стандартных `/actuator/health/liveness` и `/readiness`, но внимательно настройте таймауты.

Graceful shutdown — must. В Boot 3 это `server.shutdown=graceful` и таймаут фазы. Kubernetes `preStop` с `sleep` на пару секунд помогает дренировать соединения за счёт удаления pod из эндпоинтов и закрытия keep-alive без резких обрывов. В реактивном мире это особенно эффективно: незавершённые конвейеры успевают дослать последние байты.

Хорошая практика — выключать принимающий трафик заранее: ставить «дверь на выход» — фильтр, который в фазе остановки отвечает `Connection: close` и не стартует новые долгие стримы, оставляя текущие завершиться. Это уменьшает вероятность «обрубленных» SSE/WS.

Сетевые таймауты должны соответствовать k8s-пробам и балансировщикам. Согласуйте idle-timeout Load Balancer’а, keep-alive сервера и таймауты клиентов. Несогласованность рождает странные 499/502 на ровном месте.

Храните конфиги во внешних Secret/ConfigMap и прокидывайте в приложение через `SPRING_` переменные. Так вы сможете на лету корректировать лимиты пула, таймауты, origins для CORS и т. п. — это критично при инцидентах.

При раскатке используйте **rolling update** с минимальным `maxUnavailable` и достаточным `maxSurge`, чтобы не провалить capacity. Для реактивного сервиса перевалка обычно быстрая, но лучше держать запас на прогрев TLS/пула.

Наконец, CI/CD должен валидировать образ и чарты: контейнер стартует, health-пробы отвечают, graceful работает. Несколько минут теста на стенде окупают часы ночных звонков.

**Dockerfile (multi-stage, JDK 21, distroless-like)**

```dockerfile
# build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./gradlew clean bootJar

# run
FROM eclipse-temurin:21-jre-alpine
ENV JAVA_OPTS="-XX:+UseZGC -XX:MaxRAMPercentage=75"
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar"]
```

**application.yml — graceful и management**

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

**Helm values (фрагмент Deployment + пробы)**

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
lifecycle:
  preStop:
    exec:
      command: ["sh","-c","sleep 5"]
```

**Java — «дверь на выход» при остановке (логика дренажа)**

```java
package com.example.deploy;

import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.stereotype.Component;
import java.util.concurrent.atomic.AtomicBoolean;

@Component
public class DrainState implements ApplicationListener<ContextClosedEvent> {
    private static final AtomicBoolean DRAINING = new AtomicBoolean(false);

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        DRAINING.set(true);
    }

    public static boolean isDraining() { return DRAINING.get(); }
}
```

**Kotlin — WebFilter, который закрывает keep-alive при дрене**

```kotlin
package com.example.deploy

import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
class DrainFilter : WebFilter {
    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        if (DrainState.isDraining()) {
            exchange.response.headers.add("Connection", "close")
        }
        return chain.filter(exchange)
    }
}
```

