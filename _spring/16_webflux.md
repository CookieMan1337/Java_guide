---
layout: page
title: "Spring WebFlux: Реактивный веб-фреймворк от Spring"
permalink: /spring/webflux
---

# 0. Введение: что такое WebFlux и когда он нужен

## Что это

Spring WebFlux — это **неблокирующий** веб-стек на базе реактивной модели **Reactive Streams** (интерфейсы `Publisher`, `Subscriber`, `Subscription`, backpressure). В отличие от классического Spring MVC, где один поток сервера «занят» пока идёт I/O, WebFlux работает на **event-loop** (событийных циклах) и реактивных типах **Reactor** — `Mono<T>` (0..1 значение) и `Flux<T>` (0..N значений). Это позволяет обслуживать большое количество одновременных запросов при интенсивном I/O.

Важная особенность — **ленивость** и «протекание сигналов»: данные обрабатываются по мере готовности, а не «скопом». Когда downstream-подписчик выражает «спрос» (backpressure), источник выдаёт следующий элемент. Такой подход снижает давление на память и повышает устойчивость под нагрузкой.

WebFlux поддерживает две программные модели: **аннотационную** (как в MVC: `@RestController`, `@GetMapping`, но возвращаем `Mono/Flux`) и **функциональную** (декларативные маршруты через `RouterFunction` и обработчики `HandlerFunction`). Обе модели равноправны; часто выбирают аннотационную для привычности, функциональную — для «чистого» типа потоков и удобства композиции.

Минимальный пример контроллера WebFlux (аннотационная модель):

```java
@RestController
@RequestMapping("/api")
class HelloController {

  @GetMapping("/hello")
  Mono<Map<String, Object>> hello() {
    return Mono.just(Map.of("msg", "hello webflux"));
  }

  @GetMapping(value = "/ticks", produces = MediaType.TEXT_EVENT_STREAM_VALUE) // SSE
  Flux<Long> ticks() {
    return Flux.interval(Duration.ofSeconds(1)); // бесконечный поток
  }
}
```

## На чём строится

В основе WebFlux — **Project Reactor** (ядро реактивных типов и операторов) и контракт **Reactive Streams** (backpressure). Серверная часть по умолчанию работает на **Reactor Netty** — это неблокирующий HTTP-сервер/клиент на базе Netty с event-loop-моделью и пулами соединений. Также WebFlux может работать поверх **Servlet 3.1+ контейнеров** (Tomcat/Jetty) в режиме неблокирующих I/O (NIO), но выигрыш по сравнению с Netty обычно меньше.

В экосистеме есть поддержка альтернативных кодеков (JSON через Jackson, Protobuf, CBOR, byte streams), реактивных адаптеров к DataBuffer, multipart-загрузкам. Модель фильтров — **`WebFilter`** (реактивный аналог `javax.servlet.Filter`), а точка входа — **`HttpHandler`** → **`WebHandler`** → роутер/контроллер.

Важно понимать, что реактивность не магия: если где-то внутри вы вызываете **блокирующий** код (например, JDBC), вы «останавливаете» event-loop и теряете преимущества. В таких случаях оборачивайте блокирующие вызовы в **`Schedulers.boundedElastic()`** или выносите в отдельный сервис/пул (подробнее в разделе 1).

Минимальная конфигурация WebFlux (Boot поднимет Reactor Netty автоматически):

```java
@SpringBootApplication
public class App {
  public static void main(String[] args) {
    SpringApplication.run(App.class, args);
  }
}
```

## Зачем нужен

WebFlux нужен там, где **I/O-нагрузка** доминирует над CPU: множество одновременных запросов к БД/кешам/внешним HTTP, стриминг событий (SSE), **WebSocket**, **long-polling**, «фан-аут»/«фан-ин» интеграции (запрос → много параллельных подзапросов → агрегирование). В таких сценариях неблокирующая модель позволяет лучшей утилизации потоков и памяти.

Ещё одна сильная сторона — **потоковые ответы**: передавать данные по мере готовности без буферизации (SSE, NDJSON, byte-стримы). Это уменьшает латентность «первого байта» и позволяет клиенту сразу начинать обработку, пока сервер продолжает вычисления.

Реактивная модель хорошо ложится на **микросервисные** системы, где каждый сервис — I/O-прокси между несколькими зависимостями (БД, другие HTTP, брокеры сообщений). В таких системах WebFlux с `WebClient` помогает держать низкую латентность и высокую степень параллелизма при умеренном количестве ядер.

Ключевая выгода проявляется под **пиковыми нагрузками**: один узел способен обслужить существенно больше одновременных соединений (keep-alive, HTTP/2), чем блокирующий стек, без линейного роста пулов потоков.

Пример сервер-sent events эндпоинта (стриминг):

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> events() {
  return Flux.interval(Duration.ofSeconds(1))
      .map(i -> ServerSentEvent.builder("tick-" + i).build());
}
```

## Где используется

WebFlux логичен в **API-шлюзах**, «агрегаторах» данных, **чатах/уведомлениях**, стриминге метрик/логов, интеграциях «1 запрос → много подзапросов» (fan-out). Он подходит и для внутренних административных панелей, где важны быстрые SSE-обновления.

Также WebFlux применяют для **reactive data** стеков: R2DBC (Postgres/MySQL), reactive MongoDB/Cassandra/Redis, где весь путь — неблокирующий. Такой «сквозной» подход даёт максимальную отдачу от реактивной модели.

В области IoT/финтех/игровых серверов реактивность помогает удерживать **десятки тысяч** открытых соединений с небольшими сообщениями и регламентированными SLA по задержкам.

Наконец, WebFlux часто используют как «клей» между внешними API: с `WebClient` можно эффективно мультиплексировать сотни одновременных вызовов, тонко настраивая пулы, таймауты и backpressure.

Пример функциональной маршрутизации (RouterFunction):

```java
@Configuration
class Routes {
  @Bean
  RouterFunction<ServerResponse> httpRoutes() {
    return RouterFunctions.route()
      .GET("/fn/hello", req -> ServerResponse.ok().bodyValue(Map.of("ok", true)))
      .build();
  }
}
```

---

# 1. Когда выбирать WebFlux (и когда — нет)

## Цели: параллелизм, I/O, стриминг, long-polling/SSE/WebSocket

Если ваша цель — **высокая степень параллелизма** при ограниченных ресурсах, WebFlux — верный выбор. В event-loop модели десятки тысяч одновременных соединений не требуют десятков тысяч потоков; переключения контекста минимальны, что уменьшает overhead и GC-давление.

Для **стриминга** WebFlux даёт естественные primitives: `Flux` в ответе, SSE, **NDJSON** (newline-delimited JSON), «сквозное» кодирование без буферизации. Клиент начинает чтение раньше, сервер продолжает «подливать» данные по мере готовности.

**Long-polling** и **WebSocket** также проще и «нативнее» в WebFlux: `ReactiveWebSocketHandler` и `WebSocketClient` дают реактивные потоки сообщений с backpressure. Вы можете объединять, фильтровать и переиспользовать реактивные последовательности едва ли не так же, как коллекции.

Пример WebSocket-эхо хэндлера:

```java
@Configuration
@EnableWebFlux
class WebSocketCfg implements WebSocketConfigurer {
  @Override public void registerWebSocketHandlers(WebSocketHandlerRegistry r) {
    r.addHandler(session ->
        session.send(session.receive().map(WebSocketMessage::getPayloadAsText)
              .map(s -> "echo: " + s)
              .map(session::textMessage)),
      "/ws/echo").setAllowedOrigins("*");
  }
}
```

## Сравнение с Spring MVC: неблокирующий vs блокирующий; влияние на архитектуру/DevOps

Spring MVC строится на **блокирующей** модели: под каждый запрос — поток, который «занят» во время I/O (чтение/запись сокета, JDBC, HTTP-клиент). Это просто и предсказуемо, хорошо для CPU-интенсивных задач и небольшого числа одновременных соединений. WebFlux — **неблокирующий**: один поток может «переключаться» между тысячами операций I/O.

Архитектурно это влияет на **границы**: нельзя бездумно «звать JDBC прямо здесь» — нужно или реактивный драйвер, или оффлоад в `boundedElastic`. Модель ошибок/таймаутов также меняется: вместо бросков исключений предпочтительнее `onErrorResume`, `timeout`, `retryWhen`.

С точки зрения **DevOps**, WebFlux требует иного профилирования: важны event-loop метрики (queue length, pending tasks), пулы соединений, R2DBC-пулы, иные паттерны GC. Логи и трейсы желательно «проклеивать» через Reactor Context (см. раздел 9).

MVC остаётся отличным выбором для CRUD-сервисов с относительно небольшим уровнем параллелизма, богатой экосистемой блокирующих драйверов (JPA, JDBC), и когда команда не готова к реактивной парадигме. WebFlux — инструмент под **конкретные** задачи высокой I/O-конкурентности.

Пример различий в сигнатурах:

```java
// MVC
@GetMapping("/users/{id}")
UserDto one(@PathVariable long id) { return service.find(id); }

// WebFlux
@GetMapping("/users/{id}")
Mono<UserDto> one(@PathVariable long id) { return service.findReactive(id); }
```

## Ограничения: блокирующие драйверы/SDK; изоляция через boundedElastic/вынесение

Главное ограничение — **блокирующие** SDK: JDBC/JPA, LDAP-клиенты без NIO, синхронные S3/HTTP/FS-клиенты. Если их вызывать на event-loop, вы «заморозите» сервер. Решение 1: оборачивать вызовы в `publishOn(Schedulers.boundedElastic())`, что выполнит работу в выделенном эластичном пуле потоков (ограниченный по размеру).

Решение 2 — **анти-коррупционный слой**: вынести блокирующие взаимодействия в **отдельный сервис** (микросервис) на MVC/блокирующем стеке и общаться с ним по реактивному HTTP. Это чище архитектурно и снимает риски случайных блокировок.

Использование `boundedElastic` нужно контролировать: это «спасательный круг», но при массовом offload вы вернётесь к классической модели «много потоков». Мониторьте счётчики задач/очередей, ставьте таймауты и лимиты конкурентности (`flatMap(..., concurrency)`).

Пример изоляции блокирующего JDBC-звонка:

```java
Mono<User> findUserBlocking(long id) {
  return Mono.fromCallable(() -> jdbcTemplate.queryForObject(SQL, rowMapper, id))
             .subscribeOn(Schedulers.boundedElastic()) // offload
             .timeout(Duration.ofSeconds(1));
}
```

## Гибрид: сосуществование MVC + WebFlux и границы применения

Технически в одном приложении могут **сосуществовать** MVC и WebFlux, но рекомендуется **не смешивать** в одном `DispatcherServlet`: у них разные механизмы. В Spring Boot обычно выбирают один стартер: `spring-boot-starter-web` (MVC) **или** `spring-boot-starter-webflux`. Возможен гибрид через **разные порты/контексты** (например, UI на MVC, API на WebFlux в отдельном модуле/процессе).

Более частый гибрид — **WebFlux сервер + блокирующие участки на boundedElastic** (см. выше). Это компромисс, который даёт выгоду на I/O-интеграциях и позволяет постепенно мигрировать на реактивные драйверы.

Границы применения удобно фиксировать архитектурно: «контроллеры — реактивные», «доступ к данным — реактивный (R2DBC/Reactive Mongo/Redis), а где нет — отдельный модуль». Такой раздел помогает избежать «реактивно-блокирующей мешанины» и сделать поведение предсказуемым.

Пример Gradle-зависимостей для чистого WebFlux:

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-webflux")
  implementation("org.springframework.boot:spring-boot-starter-validation")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

# 2. Базовые концепции Reactor

## Reactive Streams: Publisher/Subscriber/Subscription, backpressure

**Reactive Streams** — это спецификация интерфейсов, описывающих реактивное взаимодействие с **контролем давления** (backpressure). `Publisher<T>` выдаёт элементы, `Subscriber<T>` их получает, а `Subscription` управляет запросом (`request(n)`), отменой (`cancel()`). Контракт гарантирует, что производитель **не «затопит»** потребителя.

В Reactor типы `Flux` и `Mono` реализуют `Publisher`. Когда вы вызываете `.subscribe()`, формируется связь «источник → подписчик» и начинается передача. Пока подписчик не запросил данные (`request(n)`), источник ждать; многие операторы управляют этим прозрачно.

Backpressure позволяет строить **устойчивые** пайплайны: например, отправка SSE клиенту «с той скоростью, с которой он читает». Если потребитель не успевает — `onBackpressureBuffer`, `onBackpressureDrop` и другие стратегии регулируют поведение.

Ниже — иллюстрация ручной подписки (в приложениях обычно не нужно, это делает WebFlux):

```java
Flux.range(1, 10).subscribe(new BaseSubscriber<>() {
  @Override protected void hookOnSubscribe(Subscription s) { request(1); }
  @Override protected void hookOnNext(Integer value) {
    System.out.println("got " + value);
    request(1);
  }
});
```

## Типы: Mono/Flux, горячие/холодные источники, «не завершать стрим»

`Mono<T>` — максимум один элемент (или ошибка/завершение без элемента), `Flux<T>` — 0..N элементов. Источники делятся на **холодные** (каждая подписка запускает вычисление заново, например `Flux.range`, `Mono.fromCallable`) и **горячие** (генерируют данные независимо от подписок: `Sinks`, `Flux.interval`).

Важное правило: многие HTTP-эндпоинты должны **завершаться** (end of stream), иначе соединение «повиснет». Исключение — намеренно бесконечные стримы (SSE, WebSocket), где вы самостоятельно управляете жизненным циклом и heartbeat’ами.

«Горячие» источники часто используют с **SSE/WebSocket** для мультикаста на несколько клиентов; для этого есть `Sinks.many().multicast().onBackpressureBuffer()`. «Холодные» — чаще для вычислений/запросов к БД: каждый клиент получает свой экземпляр последовательности.

Пример горячего «тикающего» источника + SSE:

```java
Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream() { return sink.asFlux(); }

// где-то в другом месте
sink.tryEmitNext("update-" + Instant.now());
```

## Операторы: создание, трансформации, объединение, onError*/retry/timeout

Reactor предлагает **сотни операторов**: создание (`just`, `fromCallable`, `defer`), трансформации (`map`, `flatMap`, `concatMap`), объединение (`merge`, `zip`, `concat`), фильтрация (`filter`, `take`), ошибки (`onErrorReturn`, `onErrorResume`, `retryWhen`), контроль времени (`timeout`, `delayElements`).

Выбор оператора влияет на **семантику**: `flatMap` даёт параллелизм и «перемешивание» элементов, `concatMap` — последовательную обработку, `flatMapSequential` — гибрид. Для ошибок — `onErrorResume` позволяет «переключиться» на запасной источник, `retryWhen` — задать стратегию повторов (например, экспоненциальную с jitter).

Для HTTP/БД-интеграций всегда ставьте **таймауты** (`.timeout(Duration.ofSeconds(1))`) и обрабатывайте исключения, чтобы не «зависнуть» в вечных ожиданиях. Важно завершать поток корректно и возвращать клиенту понятный статус.

Пример «фан-аута» к двум сервисам и слияния результатов:

```java
Mono<User> user = client.get().uri("/user/{id}", id).retrieve().bodyToMono(User.class);
Mono<List<Order>> orders = client.get().uri("/orders/{id}", id).retrieve().bodyToFlux(Order.class).collectList();

Mono<UserWithOrders> dto =
    Mono.zip(user, orders)
        .map(t -> new UserWithOrders(t.getT1(), t.getT2()))
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(e -> Mono.just(UserWithOrders.fallback()));
```

## Планировщики: immediate, parallel, boundedElastic; «не блокировать event-loop»

**Schedulers** управляют, где выполняется работа. По умолчанию Reactor исполняет операторы **там же**, где идёт подписка (в WebFlux — это event-loop Netty). Поэтому правило №1: **никогда не блокируйте** event-loop (никаких `Thread.sleep`, JDBC, синхронных HTTP). Если нужно заблокировать — используйте `subscribeOn/ publishOn` с `Schedulers.boundedElastic()`.

`boundedElastic` — пул для блокирующих/долгих операций (размер ограничен, растёт по необходимости). `parallel()` — фиксированный пул для CPU-задач (обычно `#cores`). `immediate()` — без переключений (текущий поток). Всегда измеряйте, не злоупотребляйте оффлоадом: иначе вы потеряете преимущество реактивной модели.

Для безопасной обработки больших файлов/архивов комбинируйте **чтение в реактивный стрим** и «узкие» участки на `boundedElastic`, обязательно с **таймаутами** и **лимитами размерностей** (buffer/данные/конкурентность).

Пример оффлоада блокирующего JSON-парсинга:

```java
Mono<MyObj> parse(Path path) {
  return Mono.fromCallable(() -> Files.readString(path))
      .subscribeOn(Schedulers.boundedElastic())
      .map(json -> new ObjectMapper().readValue(json, MyObj.class));
}
```

---

# 3. Архитектура WebFlux

## Серверы: Reactor Netty (дефолт), Servlet 3.1 контейнер (Tomcat/Jetty)

По умолчанию Spring Boot с `spring-boot-starter-webflux` поднимает **Reactor Netty** как HTTP-сервер. Это наиболее производительный вариант для типичных реактивных нагрузок: он глубоко интегрирован с Reactor, поддерживает HTTP/2, efficient keep-alive, connection pooling.

Альтернатива — запуск WebFlux поверх **Servlet 3.1 контейнера** (Tomcat/Jetty) с неблокирующим I/O. Это актуально, если у вас уже есть инфраструктура/фильтры вокруг Servlet, но вы хотите реализовать часть эндпоинтов реактивно. Однако путь данных будет длиннее, а перформанс — ниже, чем у «чистого» Netty.

Выбор сервера влияет на **тюнинг**: для Netty — размер event-loop, pOOL соединений, TCP-настройки; для Servlet-контейнеров — NIO-пулы, max-коннекты, очереди. В любом случае, старайтесь избегать смешения MVC и WebFlux в одном приложении, если нет веских причин.

Переход на WebFlux (Gradle):

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-webflux")
  // НЕ добавляйте spring-boot-starter-web одновременно, чтобы не смешать стеки
}
```

## Слои: HttpHandler → WebHandler → WebFilter → Router/Handler → codecs

Путь запроса: вход в **`HttpHandler`** (низкоуровневый HTTP), затем **`WebHandler`**, и дальше включаются ваши **`WebFilter`** (реактивные фильтры до/после), после чего управление передаётся либо в **RouterFunction/HandlerFunction** (функциональная модель), либо в **`@Controller`** (аннотационная модель).

Сериализация/десериализация тела происходит через **кодеки** (см. ниже): WebFlux использует `ServerCodecConfigurer` для регистрации JSON/CBOR/Protobuf/byte-кодеков. Для больших тел доступен низкоуровневый API `DataBuffer`, который избегает лишних копирований.

Модель **фильтров** (`WebFilter`) — место для кросс-сечений: логирование, корреляция, авторизация, пер-запросные таймауты. Отличие от Servlet Filter — реактивные сигнатуры и отсутствие блокировок/commit до завершения цепочки.

Пример `WebFilter` для корреляции:

```java
@Component
class CorrelationFilter implements WebFilter {
  @Override
  public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    String id = Optional.ofNullable(exchange.getRequest().getHeaders().getFirst("X-Request-Id"))
                        .orElse(UUID.randomUUID().toString());
    return chain.filter(exchange)
        .contextWrite(ctx -> ctx.put("traceId", id));
  }
}
```

## Модель контроллеров: аннотационная и функциональная; плюсы/минусы

**Аннотационная модель** (`@RestController`) знакома большинству Java-разработчиков: те же аннотации маршрутизации (`@GetMapping`, `@PostMapping`), те же механизмы валидации/советов, но методы возвращают `Mono/Flux`. Плюс — низкий порог входа; минус — влияние магии аннотаций и меньшая «композируемость» маршрутов.

**Функциональная модель** строится на `RouterFunction` (определяет маршруты) и `HandlerFunction` (обработчик запроса). Она даёт явный DSL, лучше подходит для **SSE/streaming**, и хорошо «складывается» при сборке сложных маршрутов. Минус — придётся привыкнуть, меньше примеров в сети.

Обе модели можно смешивать в одном приложении, но на проекте лучше выбрать **одну основной** для единообразия. Для API «классического» вида чаще берут аннотационную модель; для «stream-heavy» — функциональную.

Пример функционального роутера:

```java
@Configuration
class FunctionalRoutes {

  @Bean
  RouterFunction<ServerResponse> routes(MyHandler h) {
    return RouterFunctions.route()
        .GET("/fn/users/{id}", h::user)
        .build();
  }
}

@Component
class MyHandler {
  Mono<ServerResponse> user(ServerRequest req) {
    return Mono.just(new User(req.pathVariable("id"), "Neo"))
        .flatMap(u -> ServerResponse.ok().bodyValue(u));
  }
}
```

## Кодеки: Jackson/JSON, Protobuf, CBOR, byte streams; ServerCodecConfigurer

WebFlux использует **`HttpMessageReader`** и **`HttpMessageWriter`** для чтения/записи тел. По умолчанию подключён JSON через **Jackson**. Можно добавить **CBOR** (`Jackson2CborEncoder/Decoder`), **Smile**, **Protobuf** (`ProtobufEncoder/Decoder`) — это полезно для бинарных протоколов и экономии трафика.

Для потоковой передачи файлов и крупных данных используйте **byte-кодеки** (`DataBuffer` и `DataBufferUtils`), чтобы избежать полной буферизации. При необходимости меняйте лимиты (maxInMemorySize) — дефолты могут быть недостаточны/избыточны.

Конфигурация кодеков через `ServerCodecConfigurer`:

```java
@Configuration
class CodecConfig implements WebFluxConfigurer {
  @Override
  public void configureHttpMessageCodecs(ServerCodecConfigurer c) {
    // увеличить лимит в памяти (например, для больших JSON)
    c.defaultCodecs().maxInMemorySize(4 * 1024 * 1024);

    // включить CBOR
    c.customCodecs().register(new Jackson2CborDecoder());
    c.customCodecs().register(new Jackson2CborEncoder());
  }
}
```

Пример отдачи файла «в поток»:

```java
@GetMapping(value = "/download/{name}", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
public Mono<ResponseEntity<DataBuffer>> download(@PathVariable String name) {
  Path path = storage.resolve(name);
  return DataBufferUtils.read(path, new DefaultDataBufferFactory(), 8192)
      .reduce(DataBuffer::write) // склеим в один буфер (или пишите Flux<DataBuffer> как есть)
      .map(buf -> ResponseEntity.ok()
              .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + name + "\"")
              .body(buf));
}
```
отлично — продолжаю «Тему 16: WebFlux». Ниже подробно раскрыты **подтемы 4–6**. Для каждого подпункта — 4+ абзацев с пояснениями и кодом. Стек: **Java 17+, Spring Boot 3.x, Spring WebFlux, Reactor**.

---

# 4. Контроллеры и обработка запросов

## Возврат `Mono`/`Flux`: сериализация, завершение/ошибки, статусные ответы

В аннотационной модели WebFlux методы контроллеров возвращают `Mono<T>` (0..1) или `Flux<T>` (0..N), которые WebFlux «подпишет» и сериализует через соответствующие **кодеки** (по умолчанию Jackson → JSON). Поток завершения важен: **HTTP-ответ будет закрыт только когда Publisher завершится** (onComplete/onError). Если поток бесконечный (например, `Flux.interval`), это сознательный выбор (SSE/streaming), иначе вы рискуете «повисшим» соединением.

Ошибки в реактивном пайплайне всплывают как **onError** и транслируются в HTTP-ответ (по умолчанию 500). Рекомендуется централизовать маппинг в формат **Problem Details (RFC-7807)** через `@ControllerAdvice` (см. подпункт про валидацию). Внутри пайплайна используйте `onErrorResume/return` для «локальной» деградации — например, вернуть «пустой список» вместо ошибки внешнего сервиса.

Статусы ответа задаются несколькими способами. Самый простой — вернуть `ResponseEntity<T>` в `Mono<ResponseEntity<T>>` и управлять `status/headers`. Для «пустых» успешных ответов используйте `Mono<Void>` (например, при стриминге файлов напрямую в Netty-буфер). Для ошибок бросайте специализированные исключения (`ResponseStatusException`) или свои и маппируйте их в advice.

Минимальный контроллер:

```java
@RestController
@RequestMapping("/api/users")
class UserController {

  private final UserService service;

  UserController(UserService service) { this.service = service; }

  @GetMapping("/{id}")
  Mono<ResponseEntity<UserDto>> get(@PathVariable long id) {
    return service.findById(id)
        .map(ResponseEntity::ok)
        .switchIfEmpty(Mono.just(ResponseEntity.notFound().build()));
  }

  @GetMapping
  Flux<UserDto> list() {
    return service.findAll(); // сериализуется как JSON-массив по мере готовности элементов
  }
}
```

## Валидация: реактивный Bean Validation, `@Valid/@Validated`, ошибки и ProblemDetail

В WebFlux поддерживается **Bean Validation** (Jakarta Validation) — аннотации `@NotBlank`, `@Min`, `@Email` и др. Для тела запроса (`@RequestBody`) просто пометьте аргумент `@Valid`; для параметров (`@RequestParam`, `@PathVariable`) включите `@Validated` на классе/методе. В случае нарушения валидации WebFlux бросит `WebExchangeBindException`, которую удобно маппировать на **Problem Details**.

Важно: валидатор сработает **после** десериализации JSON → DTO. Если ваш DTO реактивный (например, `Mono<DTO>`), используйте `@Valid @RequestBody Mono<DTO>` и валидируйте через `flatMap`, либо разместите аннотацию на поле DTO, а контроллеру достаточно `Mono<DTO>`.

Глобальный обработчик ошибок в стиле Problem Details позволяет унифицировать ответы (для фронта и интеграций). В WebFlux есть `ProblemDetail` (Spring 6): создайте его и верните из `@ExceptionHandler`.

```java
@RestControllerAdvice
class GlobalErrors {

  @ExceptionHandler(WebExchangeBindException.class)
  public ResponseEntity<ProblemDetail> onBind(WebExchangeBindException e) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setProperty("errors", e.getFieldErrors().stream()
        .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
        .toList());
    return ResponseEntity.badRequest().body(pd);
  }
}

record CreateUserReq(@NotBlank String name, @Email String email) {}

@RestController
class CreateUserController {
  @PostMapping("/api/users")
  Mono<ResponseEntity<Void>> create(@Valid @RequestBody Mono<CreateUserReq> mono) {
    return mono.flatMap(req -> /* save */ Mono.empty())
               .then(Mono.just(ResponseEntity.status(201).build()));
  }
}
```

## Загрузка/выгрузка файлов: multipart, `DataBuffer`/`Part`, потоковая передача

В WebFlux multipart-данные приходят как реактивные `Part`: `FilePart` (файл), `FormFieldPart` (поле формы). Для сохранения файла **не буферизуйте полностью в память** — используйте `FilePart#transferTo` (самый простой путь) или `DataBufferUtils.write(...)` для тонкого контроля и ограничения скорости/объёма.

При скачивании больших файлов вы можете возвращать **`Flux<DataBuffer>`**, чтобы отдавать контент порциями (chunked) без буферизации всего в памяти. Следите за **`Content-Type`** и `Content-Disposition`, чтобы браузер не пытался «исполнить» файл.

```java
@RestController
@RequestMapping("/api/files")
class FileController {

  private final Path root = Path.of("/tmp/uploads");

  @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
  Mono<ResponseEntity<Void>> upload(@RequestPart("file") Mono<FilePart> fileMono) {
    return fileMono.flatMap(fp -> {
      Path dest = root.resolve(fp.filename()).normalize();
      return fp.transferTo(dest);
    }).thenReturn(ResponseEntity.accepted().build());
  }

  @GetMapping(value = "/{name}", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
  Mono<ResponseEntity<Flux<DataBuffer>>> download(@PathVariable String name) {
    Path p = root.resolve(name).normalize();
    var flux = DataBufferUtils.read(p, new DefaultDataBufferFactory(), 16 * 1024);
    return Mono.just(ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + name + "\"")
        .body(flux));
  }
}
```

## SSE/WebSocket: «горячие» стримы, heartbeat/keep-alive, backpressure

**SSE (Server-Sent Events)** — односторонний поток событий от сервера к клиенту по `text/event-stream`. Отлично подходит для уведомлений, прогресса, обновлений. В WebFlux достаточно указать `produces = MediaType.TEXT_EVENT_STREAM_VALUE` и вернуть `Flux<ServerSentEvent<...>>`. Для долгоживущих стримов добавляйте **heartbeat**: периодически посылайте «пустые» события, чтобы межсетевые экраны не закрывали простои.

**WebSocket** — двунаправленное общение. В WebFlux он представлен через `WebSocketHandler` и реактивные `Flux<WebSocketMessage>`/`Mono<Void>`. Работайте с backpressure: если клиент «медленный», ограничьте `send` (например, `.onBackpressureDrop()`), иначе очередь сообщений может расти.

Также предусмотрите **повторные подключения** на клиенте (SSE: EventSource автоматически переподключается) и **идемпотентность** событий (идентификаторы/offset), чтобы потерянные сообщения могли быть восстановлены.

```java
@GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
Flux<ServerSentEvent<String>> sse() {
  return Flux.interval(Duration.ofSeconds(1))
      .map(i -> ServerSentEvent.builder("tick-" + i).id(i.toString()).build())
      .mergeWith(Flux.interval(Duration.ofSeconds(15))
                     .map(i -> ServerSentEvent.<String>builder().comment("heartbeat").build()));
}
```

---

# 5. `WebClient`: реактивный HTTP-клиент

## Пулы соединений, таймауты (connect/read/write), лимиты буфера, кодеки

`WebClient` — реактивный HTTP-клиент на базе **Reactor Netty**. Под капотом — пул TCP-соединений (keep-alive) и неблокирующая обработка. Важно **обязательно** настраивать таймауты: подключение (`CONNECT`), ожидание ответа (read), запись (write). Иначе «зависшие» зависимые сервисы повиснут и ваш клиент.

Буферизация регулируется max-in-memory размером: большие ответы (например, 50 МБ JSON) могут переполнить память; используйте потоковое чтение (`bodyToFlux<DataBuffer>`) или увеличьте лимит осторожно. Кодеки `WebClient` наследует от WebFlux — добавляйте CBOR/Protobuf при необходимости.

```java
@Configuration
class WebClientCfg {

  @Bean
  WebClient externalClient() {
    var http = HttpClient.create()
        .responseTimeout(Duration.ofSeconds(2))
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000)
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(2))
            .addHandlerLast(new WriteTimeoutHandler(2)));

    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .clientConnector(new ReactorClientHttpConnector(http))
        .codecs(c -> c.defaultCodecs().maxInMemorySize(2 * 1024 * 1024)) // 2MB
        .build();
  }
}
```

## Стриминг ответов, backpressure, `bodyToFlux<DataBuffer>`

Если ответ большой или бесконечный (SSE/NDJSON), читайте **потоком**. `bodyToFlux<DataBuffer>` отдаёт чанки, которые вы можете «перекачивать» дальше (в файл/сокет) без буферизации целиком. Это снижает латентность и память.

Backpressure на клиенте работает аналогично: вы потребляете чанки с той скоростью, которая вам нужна. При записи в файл используйте `DataBufferUtils.write` и **не забывайте релизить** буферы (утечки памяти возможны). Если после декодирования в объекты — `bodyToFlux<MyDto>()`, и помните про `maxInMemorySize`.

```java
Mono<Path> downloadTo(Path dest, WebClient wc) {
  return wc.get().uri("/bigfile")
      .retrieve()
      .bodyToFlux(DataBuffer.class)
      .transform(flux -> DataBufferUtils.write(flux, dest, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING))
      .then(Mono.just(dest));
}
```

## Надёжность: retry/backoff, timeout, circuit breaker (Resilience4j), идемпотентность

Для сетевой надёжности комбинируйте: **timeout** на уровень операции (`.timeout(...)`), **ретраи** с экспоненциальной паузой (`retryWhen(Retry.backoff(...))`), и **circuit breaker** (Resilience4j) для отсечки «ломающегося» сервиса. Важно учитывать **идемпотентность**: безопасно ретраить GET/HEAD и идемпотентные POST (с идемпотентным ключом на сервере). Для небезопасных операций (платёж) ретрай только при гарантированно «неуспешной» отправке (например, connect timeout) и с **идемпотентным ключом**.

В Resilience4j для `WebClient` используйте фильтр `Resilience4jCircuitBreakerOperator` или аннотации на сервисном методе (`@CircuitBreaker`, `@Retry`). Следите за метриками срабатывания и настройте короткие окна на тестовых окружениях.

```java
Mono<MyDto> call(WebClient wc) {
  return wc.get().uri("/resource")
      .retrieve()
      .bodyToMono(MyDto.class)
      .timeout(Duration.ofMillis(800))
      .retryWhen(Retry.backoff(2, Duration.ofMillis(200))
                      .filter(ex -> ex instanceof ReadTimeoutException || ex instanceof TimeoutException));
}
```

## Диагностика: wire-логгинг, заголовки/трейсы, лимитирование скорости

Для отладки сетевых проблем включайте **wiretap** в Reactor Netty (уровень DEBUG): это покажет отправленные/полученные строки HTTP и заголовки (осторожно с продом и чувствительными данными). Прокидывайте **traceId**/`X-Request-Id` через заголовки (Micrometer Tracing/OpenTelemetry сделает это автоматически), логируйте статус, длительность и размер.

Локальное **лимитирование скорости** можно реализовать как фильтр `ExchangeFilterFunction` (token bucket) на клиенте, чтобы не «забить» внешний сервис. Ещё один полезный фильтр — маскирование чувствительных заголовков в логах (`Authorization`, `Cookie`).

```java
@Bean
WebClient loggedClient() {
  var http = HttpClient.create()
      .wiretap("reactor.netty.http.client.HttpClient", LogLevel.DEBUG, AdvancedByteBufFormat.TEXTUAL);

  return WebClient.builder()
      .clientConnector(new ReactorClientHttpConnector(http))
      .filter((req, next) -> {
        long start = System.nanoTime();
        return next.exchange(req).doOnEach(sig -> {
          if (sig.isOnComplete() || sig.isOnError()) {
            long ms = (System.nanoTime() - start) / 1_000_000;
            // log: req.method, req.url, status, ms
          }
        });
      })
      .build();
}
```

---

# 6. Доступ к данным: R2DBC и ко

## R2DBC: транзакции, `ConnectionFactory`, отличия от JPA

**R2DBC** — реактивный драйвер SQL-БД (Postgres, MySQL и др.). Вместо JDBC-`DataSource` здесь используется **`ConnectionFactory`**, а доступ — через `DatabaseClient` или Spring Data R2DBC (`R2dbcEntityTemplate`, реактивные репозитории). Важно: **ленивая загрузка JPA здесь отсутствует** — вы явно формируете запросы (или используете проекции). Кэш первого уровня JPA/контекст персистентности также отсутствует.

Реактивные транзакции поддерживаются: добавьте `R2dbcTransactionManager` и пометьте методы `@Transactional` **на реактивных типах** (Spring «обернёт» Publisher транзакцией). Либо используйте низкоуровневый `TransactionalOperator`. Следите за тем, чтобы внутри транзакции вы не «уходили» в другой поток без контекста. В реактивных транзакциях commit/rollback происходит при onComplete/onError.

```java
@Configuration
class R2dbcConfig {

  @Bean
  ConnectionFactory connectionFactory() {
    return ConnectionFactories.get("r2dbc:postgresql://user:pass@localhost:5432/app");
  }

  @Bean
  R2dbcEntityTemplate template(ConnectionFactory cf) {
    return new R2dbcEntityTemplate(cf);
  }

  @Bean
  ReactiveTransactionManager r2dbcTxManager(ConnectionFactory cf) {
    return new R2dbcTransactionManager(cf);
  }
}

@Table("users")
record User(@Id Long id, String name, String email) {}

@Repository
interface UserRepo extends ReactiveCrudRepository<User, Long> {
  Flux<User> findByNameContains(String q);
}

@Service
class UserService {
  private final UserRepo repo;

  UserService(UserRepo repo) { this.repo = repo; }

  @Transactional
  public Mono<User> create(User u) { return repo.save(u); }

  public Flux<User> search(String q) { return repo.findByNameContains(q); }
}
```

## Reactive Mongo/Cassandra/Redis: паттерны репозиториев/шаблонов

Для **MongoDB** используйте `spring-boot-starter-data-mongodb-reactive`: реактивные репозитории (`ReactiveMongoRepository`) и `ReactiveMongoTemplate`. Mongo — документная БД, где реактивность особенно естественна (tailable cursors, change streams). Для **Cassandra** есть `spring-boot-starter-data-cassandra-reactive`, для **Redis** — `spring-boot-starter-data-redis-reactive` (`ReactiveRedisTemplate`).

Паттерн: репозиторий покрывает «типовые» операции, а для специфичных запросов — шаблоны (Template) с `Criteria/Query` DSL. Не забывайте про индексы: реактивность не спасёт от медленных полноскановых запросов. Для Redis — храните небольшие объекты/кэши и внимательно относитесь к TTL/eviction.

```java
@Document("items")
record Item(@Id String id, String tenant, String name) {}

interface ItemRepo extends ReactiveMongoRepository<Item, String> {
  Flux<Item> findByTenant(String tenant);
}

@Service
class ItemService {
  private final ItemRepo repo; private final ReactiveMongoTemplate tpl;

  ItemService(ItemRepo repo, ReactiveMongoTemplate tpl) { this.repo = repo; this.tpl = tpl; }

  Flux<Item> list(String tenant) { return repo.findByTenant(tenant); }

  Flux<Item> search(String tenant, String q) {
    var query = new Query(new Criteria().andOperator(
        Criteria.where("tenant").is(tenant),
        new Criteria().orOperator(Criteria.where("name").regex(q, "i"))
    ));
    return tpl.find(query, Item.class);
  }
}
```

## Работа с блокирующим JDBC: `boundedElastic`/отдельный модуль/сервис, контроль пулов

Если вам всё же нужен JDBC (например, из-за отсутствия драйвера R2DBC под вашу БД/фичу), **изолируйте** блокирующие вызовы. Оборачивайте `JdbcTemplate`/репозитории в `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` и ставьте таймауты. Контролируйте пул JDBC (Hikari): размер, таймауты, чтобы «блокирующая часть» не отжирала все ресурсы.

При значительном объёме блокирующей логики лучше вынести её в **отдельный сервис** (на MVC/JPA), а в WebFlux-приложении обращаться к нему через `WebClient`. Это чище с точки зрения архитектуры и упрощает управление ресурсами/масштабирование.

```java
@Service
class LegacyJdbcAdapter {
  private final JdbcTemplate jdbc;

  LegacyJdbcAdapter(JdbcTemplate jdbc) { this.jdbc = jdbc; }

  Mono<User> find(long id) {
    return Mono.fromCallable(() ->
        jdbc.queryForObject("select id, name, email from users where id=?",
            (rs, n) -> new User(rs.getLong("id"), rs.getString("name"), rs.getString("email")), id))
      .subscribeOn(Schedulers.boundedElastic())
      .timeout(Duration.ofSeconds(1));
  }
}
```

## Миграции схем: запуск Liquibase/Flyway отдельно, тестирование миграций

И **Flyway**, и **Liquibase** работают через **JDBC**, а не R2DBC. В реактивных приложениях это нормально: миграции выполняются **до** старта при помощи JDBC-подключения (отдельные `spring.flyway.url/user/password`), а рантайм-путь — реактивный R2DBC. Так вы избегаете блокировок в event-loop, инициализируя схему заранее.

Если вы используете Boot, достаточно добавить `spring-boot-starter-jdbc` только для миграций (scope runtime) и настроить `spring.flyway.*`. В докере/CI миграции запускают отдельным шагом до старта сервиса. В тестах (Testcontainers) поднимайте БД, гоняйте миграции JDBC, затем тестируйте R2DBC-код.

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/app
    username: app
    password: pass
  flyway:
    url: jdbc:postgresql://localhost:5432/app
    user: app
    password: pass
    locations: classpath:db/migration
```

```sql
-- db/migration/V1__init.sql
create table users (
  id bigserial primary key,
  name text not null,
  email text not null unique
);
```

```java
// Интеграционный тест
@Testcontainers
@SpringBootTest
class R2dbcIT {

  @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.r2dbc.url", () -> "r2dbc:postgresql://" + pg.getHost() + ":" + pg.getMappedPort(5432) + "/test");
    r.add("spring.r2dbc.username", () -> "test");
    r.add("spring.r2dbc.password", () -> "test");
    r.add("spring.flyway.url", () -> pg.getJdbcUrl());
    r.add("spring.flyway.user", () -> pg.getUsername());
    r.add("spring.flyway.password", () -> pg.getPassword());
  }

  @Autowired UserService service;

  @Test
  void createdAndFound() {
    var saved = service.create(new User(null, "Neo", "neo@matrix.io")).block();
    assertThat(saved.id()).isNotNull();
    assertThat(service.search("Ne").collectList().block()).extracting(User::name).contains("Neo");
  }
}
```

отлично — завершаем «Тему 16: WebFlux» подробным разбором **подтем 7–10**. Для каждого подпункта — 4+ абзацев, практические советы и рабочие фрагменты кода (Java 17+, Spring Boot 3.x, Spring WebFlux/Reactive Security, Reactor).

---

# 7. Безопасность (reactive)

## SecurityWebFilterChain вместо SecurityFilterChain; реактивные WebFilter’ы

В реактивном стеке используется **`SecurityWebFilterChain`** (а не `SecurityFilterChain`). Он конфигурируется через **`ServerHttpSecurity`** (аналог `HttpSecurity` из MVC), а входной слой — это **реактивные `WebFilter`** вместо сервлетных фильтров. Архитектурно это важно: фильтры и обработчики работают на реактивных типах `Mono/Flux`, не блокируют event-loop и возвращают управление, когда источник готов продолжить.

Spring Boot автоматически обнаруживает бины `SecurityWebFilterChain` и подключает их к цепочке обработки (`WebFilterChain`). Вы можете объявить **несколько** цепочек с разными `securityMatcher` для зон `/public/**`, `/api/**`, `/actuator/**` и т. п. Порядок цепочек регулируется `@Order` — сначала более специфичные, потом общие. Это позволяет разделить, например, stateless-API и stateful-UI в одном приложении.

Реактивные кросс-сечения (корреляция, audit, rate limit) реализуются как `WebFilter`: подписывайтесь на «сигналы» (`doOnSuccess`, `doOnError`, `contextWrite`) и **не делайте блокирующих вызовов**. Если нужно «оффлоадить» (редко — например, запись больших логов в медленное хранилище), используйте `publishOn(Schedulers.boundedElastic())`, чётко осознавая цену переключений.

Простейшая конфигурация двух цепочек — публичная и защищённая — в реактивном стиле:

```java
@Configuration
@EnableMethodSecurity // включает @PreAuthorize/@PostAuthorize в реактивном контексте
class ReactiveSecurityConfig {

  @Bean @Order(1)
  SecurityWebFilterChain publicChain(ServerHttpSecurity http) {
    return http.securityMatcher(new PathPatternParserServerWebExchangeMatcher("/public/**"))
        .authorizeExchange(ex -> ex.anyExchange().permitAll())
        .csrf(ServerHttpSecurity.CsrfSpec::disable)
        .build();
  }

  @Bean @Order(2)
  SecurityWebFilterChain apiChain(ServerHttpSecurity http) {
    return http.securityMatcher(new PathPatternParserServerWebExchangeMatcher("/api/**"))
        .authorizeExchange(ex -> ex.anyExchange().authenticated())
        .httpBasic(ServerHttpSecurity.HttpBasicSpec::disable)
        .formLogin(ServerHttpSecurity.FormLoginSpec::disable)
        .csrf(ServerHttpSecurity.CsrfSpec::disable)
        .build();
  }
}
```

## OAuth2/JWT ресурс-сервер (reactive): JWKs, scope→authority, clock-skew

Чтобы сделать WebFlux-приложение **ресурс-сервером** (принимать `Bearer`-токены и проверять их подпись), подключите стартер `spring-boot-starter-oauth2-resource-server` и включите в `ServerHttpSecurity` блок `oauth2ResourceServer().jwt()`. Подписи проверяются по **JWK Set** (JSON Web Key) с вашего IdP. Удобнее всего указывать `issuer-uri` (Boot сам найдёт `jwks_uri` через OIDC-метаданные), альтернатива — прямой `jwk-set-uri`.

По умолчанию авторитеты берутся из claim `scope`/`scp` с префиксом `SCOPE_`. Если нужно учитывать роли из `roles`/`realm_access`, настройте **`ReactiveJwtAuthenticationConverterAdapter`** с кастомным `JwtGrantedAuthoritiesConverter`. Также стоит учесть **clock-skew** (рассинхрон часов) и дополнительно валидировать `iss`/`aud` в `ReactiveJwtDecoder`.

```java
@Configuration
class JwtResourceServerConfig {

  @Bean
  SecurityWebFilterChain api(ServerHttpSecurity http, Converter<Jwt, ? extends Mono<AbstractAuthenticationToken>> conv) {
    return http.securityMatcher(new PathPatternParserServerWebExchangeMatcher("/api/**"))
        .authorizeExchange(ex -> ex
            .pathMatchers("/api/health").permitAll()
            .pathMatchers("/api/orders/**").hasAuthority("SCOPE_orders.read")
            .anyExchange().authenticated())
        .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(conv)))
        .csrf(ServerHttpSecurity.CsrfSpec::disable)
        .build();
  }

  @Bean
  Converter<Jwt, ? extends Mono<AbstractAuthenticationToken>> reactiveJwtConverter() {
    var scopes = new JwtGrantedAuthoritiesConverter();
    scopes.setAuthoritiesClaimName("scp");         // или "scope"
    scopes.setAuthorityPrefix("SCOPE_");

    var delegate = new JwtAuthenticationConverter();
    delegate.setJwtGrantedAuthoritiesConverter(jwt -> {
      var base = scopes.convert(jwt);
      var roles = Optional.ofNullable(jwt.getClaimAsStringList("roles")).orElse(List.of());
      var roleAuths = roles.stream().map(r -> new SimpleGrantedAuthority("ROLE_" + r)).toList();
      return Stream.concat(base.stream(), roleAuths.stream()).toList();
    });
    // адаптер для реактивного мира
    return new ReactiveJwtAuthenticationConverterAdapter(delegate);
  }
}
```

## Метод-уровень: `@PreAuthorize` в reactive, доступ к `SecurityContext` через Reactor Context

Метод-безопасность в реактивном стеке включается **той же** аннотацией `@EnableMethodSecurity`. Аннотация `@PreAuthorize` работает с реактивными методами и выражениями **SpEL**. Важно помнить: если проверка зависит от «реактивной» информации (например, lazy-загрузка пользователя), лучше инкапсулировать это в сервис и возвращать `Mono<Boolean>` с последующим `filterWhen`/`switchIfEmpty`.

`SecurityContext` в WebFlux хранится **в Reactor Context**. Для доступа используйте `ReactiveSecurityContextHolder.getContext()` — это `Mono<SecurityContext>`, который можно встроить в пайплайн. В контроллере удобнее принимать `Authentication`/`Principal` параметром — фреймворк сам извлечёт их из контекста.

```java
@Service
class OrderService {

  // Пример: доступ разрешён владельцу заказа или ROLE_ADMIN
  public Mono<OrderDto> findSecured(String id) {
    return ReactiveSecurityContextHolder.getContext()
      .map(SecurityContext::getAuthentication)
      .flatMap(auth ->
          repo.findById(id)
              .filterWhen(o -> Mono.just(o.ownerId().equals(auth.getName()))
                                  .zipWith(Mono.just(has(auth, "ROLE_ADMIN")), (own, admin) -> own || admin))
              .switchIfEmpty(Mono.error(new ResponseStatusException(HttpStatus.FORBIDDEN))));
  }

  private boolean has(Authentication a, String role) {
    return a.getAuthorities().stream().anyMatch(ga -> ga.getAuthority().equals(role));
  }
}

@RestController
class MeController {
  @GetMapping("/api/me")
  Mono<Map<String,Object>> me(Authentication auth) {
    return Mono.just(Map.of("name", auth.getName(), "authorities", auth.getAuthorities()));
  }
}
```

## CORS/CSRF/headers: политика для API и UI, cookie-настройки (SameSite, Secure)

В реактивном приложении **CORS** включается через `ServerHttpSecurity.cors()` и бин `CorsConfigurationSource` (так же, как в MVC, но типы другие). Для продакшна не используйте `*` с `allowCredentials=true`; перечисляйте допустимые `origins`, заголовки и методы, тестируйте preflight-запросы `OPTIONS`.

**CSRF** нужен для stateful-UI на cookie (WebFlux поддерживает `WebSession`). Для «чистого» статлесс-API на Bearer-токенах CSRF выключайте (`csrf().disable()`), а безопасность обеспечивайте JWT и CORS-политикой. Заголовки безопасности (HSTS, X-Content-Type-Options, CSP) задаются через `headers()`, как и в MVC.

Cookie-параметры важны в Single-Page App сценариях: `SameSite=None; Secure` для кросс-сайтового доступа (только по HTTPS), `HttpOnly` чтобы JS их не читал. В реактивном UI на WebFlux соответственно включите CSRF и корректно прокидывайте CSRF-токены в форму/заголовок.

```java
@Bean
SecurityWebFilterChain ui(ServerHttpSecurity http) {
  return http.securityMatcher(new PathPatternParserServerWebExchangeMatcher("/app/**","/login/**"))
      .authorizeExchange(ex -> ex.anyExchange().authenticated())
      .formLogin(Customizer.withDefaults())
      .csrf(Customizer.withDefaults())   // UI: CSRF включён
      .headers(h -> h
          .httpStrictTransportSecurity(Customizer.withDefaults())
          .contentTypeOptions(Customizer.withDefaults()))
      .cors(Customizer.withDefaults())
      .build();
}

@Bean
CorsConfigurationSource corsSource() {
  var cfg = new CorsConfiguration();
  cfg.setAllowedOrigins(List.of("https://frontend.example.com"));
  cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
  cfg.setAllowedHeaders(List.of("Authorization","Content-Type","X-Requested-With"));
  cfg.setAllowCredentials(true);
  var src = new UrlBasedCorsConfigurationSource();
  src.registerCorsConfiguration("/**", cfg);
  return src;
}
```

---

# 8. Ошибки, глобальные обработчики и политика SLA

## Глобальные ошибки: `@ControllerAdvice` (reactive), функциональные фильтры, ProblemDetail

В WebFlux можно обрабатывать ошибки двумя способами: через **аннотационную** модель (`@RestControllerAdvice` + `@ExceptionHandler`) и через **функциональные** обработчики (`ErrorWebExceptionHandler` или `HandlerFilterFunction` в RouterFunction). Первый путь привычнее и позволяет возвращать **Problem Details (RFC-7807)** единым образом.

Важно, что исключение в реактивном пайплайне проявляется как сигнал **onError**. Если вы бросите `ResponseStatusException` внутри `Mono/Flux`, WebFlux превратит её в HTTP-ответ с нужным статусом; но лучше централизованно формировать `ProblemDetail` — «чисто» и единообразно для фронта и интеграций.

Функциональная модель (`RouterFunction`) даёт возможность повесить **фильтр** на весь роутер: там можно перехватывать ошибки, замерять задержки, добавлять диагностические заголовки (`traceId`). Этот способ удобен для «чистых» файлов роутов, где вы избегаете магии аннотаций.

```java
@RestControllerAdvice
class GlobalErrorHandler {

  @ExceptionHandler(WebExchangeBindException.class)
  ResponseEntity<ProblemDetail> onBind(WebExchangeBindException e) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
    pd.setTitle("Validation failed");
    pd.setProperty("errors", e.getFieldErrors().stream()
        .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
        .toList());
    return ResponseEntity.badRequest().body(pd);
  }

  @ExceptionHandler(ResponseStatusException.class)
  ProblemDetail onRse(ResponseStatusException e) {
    var pd = ProblemDetail.forStatusAndDetail(e.getStatusCode(), e.getReason());
    pd.setTitle("HTTP error");
    return pd;
  }
}
```

## Коды и семантика: timeouts, 429/503, graceful degradation; советы по retry для клиентов

Для **timeouts** уместен код **504 Gateway Timeout** (если вы — прокси) или **503 Service Unavailable** с `Retry-After`, когда зависимость недоступна/медленна. Для **лимитов** — **429 Too Many Requests** с понятным сообщением и, по возможности, `Retry-After`. Важно, чтобы клиентам было ясно — **когда** и **стоит ли** повторить запрос.

Грейсфул деградация делается через `onErrorResume`: например, если внешний профиль-сервис «лежит», верните «усечённый» отчёт без профиля и явно отметьте это в данных. Но не скрывайте системные проблемы: ведите метрики ошибок и выдавайте соответствующие статусы там, где это критично для SLA.

Опубликуйте **гайд по retry** для клиентов: какие коды ретраить (идемпотентные GET на 502/503/504), какую схему backoff использовать, какой **идемпотентный ключ** включать в POST. Это снижает нагрузку в авариях и упорядочивает ретраи.

```java
Mono<Data> fusedCall() {
  return profileClient.getProfile()          // внешний сервис
      .timeout(Duration.ofMillis(800))
      .onErrorResume(e -> Mono.empty())      // деградация: нет профиля
      .zipWith(coreClient.getCoreData().timeout(Duration.ofMillis(800)))
      .map(tuple -> fuse(tuple.getT1().orElse(null), tuple.getT2()))
      .switchIfEmpty(Mono.error(new ResponseStatusException(HttpStatus.SERVICE_UNAVAILABLE, "Dependencies down")));
}
```

## Стриминг и ошибки: корректное завершение SSE/WebSocket, реконнекты, идемпотентность

В **SSE** желательно посылать **heartbeat** (комментарии `: ping\n\n`) раз в 10–30 сек, чтобы прокси/фаерволы не закрывали простаивающее соединение. При ошибке корректно завершайте поток `onError` → клиент сам переподключится (EventSource делает это автоматически). Используйте **id**-поля событий, чтобы клиент мог возобновить с нужного места.

В **WebSocket** следите за закрытием: отправьте **close frame** с кодом/причиной, обработайте `onClose` и очистите ресурсы. Если поток исчерпан — завершите `send`/`receive`. Не копите сообщения в очереди: используйте `onBackpressureDrop()` и квотирование.

Идемпотентность важна для **повторных подключений**: если клиент после обрыва запросил «с 100-го события», сервер должен суметь отдать с этого offset’а (храните последние N событий/offset в памяти/Redis). Это снижает дубликаты и улучшает UX.

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream(@RequestHeader(name = "Last-Event-ID", required = false) String lastId) {
  long from = lastId != null ? Long.parseLong(lastId) + 1 : 0;
  return eventStore.tail(from)
      .map(ev -> ServerSentEvent.builder(ev.payload()).id(Long.toString(ev.id())).build())
      .mergeWith(Flux.interval(Duration.ofSeconds(15)).map(i -> ServerSentEvent.<String>builder().comment("heartbeat").build()));
}
```

## Наблюдаемость ошибок: корреляция traceId/spanId, аудит

Каждая ошибка должна иметь **корреляцию** с трассировкой (`traceId/spanId`) и «деанонимизацию» по пользователю/тенанту (где это правомерно). В реактивном стеке эти атрибуты живут в **Reactor Context**; интеграции Micrometer Tracing/OpenTelemetry автоматически кладут их в контекст и логи (`MDC`).

Аудитируйте **значимые события**: аутентификация/авторизация, доступ к ресурсам, изменение данных. Формируйте минимальный набор полей: `who/when/what/result/traceId`. Не логируйте чувствительные значения (PII/секреты) и используйте маскирование.

Хорошая практика — иметь дашборды 5xx/4xx по эндпоинтам, задержки p95/p99, error budget (SLO) и «карты зависимостей» — так проще понять влияния деградации внешнего сервиса на вашу отказоустойчивость.

---

# 9. Наблюдаемость и производительность

## BlockHound: детект блокировок в event-loop; фиксы и white-list

**BlockHound** — инструмент, который «ловит» **блокирующие вызовы** (`Thread.sleep`, `FileChannel#read`, JDBC и т. п.) в реактивном коде. На dev/CI его полезно включать, чтобы `Reactor`/Netty event-loop не «замораживали» случайно. При обнаружении блокировки бросается `BlockingOperationError`.

Интеграция проста: добавьте зависимость и вызовите `BlockHound.install()` при старте тестов/приложения в dev-профиле. Некоторые библиотеки легитимно блокируют на управляемых потоках — их можно занести в **white-list** (разрешить конкретные методы) или **оффлоадить** в `boundedElastic`.

Важно не превращать white-list в «пылесос»: разрешайте только строго необходимые вызовы и сопровождайте их комментариями. Лучше переписать участок на реактивные API или вынести в выделенный пул.

```java
@TestConfiguration
class BlockHoundTestCfg {
  static {
    reactor.blockhound.BlockHound.builder()
        .allowBlockingCallsInside("org.slf4j.Logger", "info") // пример
        .install();
  }
}
```

## Метрики: Micrometer для Netty/WebClient/R2DBC; гистограммы задержек, пер-эндпоинтные таймеры

В Spring Boot 3 наблюдаемость унифицирована через **Micrometer Observation**. Для WebFlux/Netty, `WebClient`, R2DBC метрики собираются «из коробки», если включён Actuator. Пер-эндпоинтные **таймеры** помогут видеть p95/p99 задержек, коды ответов, размер тел. Включайте **гистограммы** (SLF оптимизация по конфигациям), если используете Prometheus/Grafana.

Настройка через `application.yml`: `management.metrics.distribution.percentiles-histogram.http.server.requests=true` и аналогично для `http.client.requests`/`r2dbc`. Дополнительно можно вешать `@Timed` на конкретные методы (контроллеры/сервисы), чтобы получить метрику с понятным именем и тегами.

```yaml
management:
  endpoints.web.exposure.include: health, metrics, prometheus
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        http.client.requests: true
        r2dbc.connections: true
```

```java
@RestController
class TimedController {
  @Timed(value = "api.users.get", histogram = true)
  @GetMapping("/api/users/{id}")
  Mono<UserDto> get(@PathVariable long id) { /* ... */ return service.get(id); }
}
```

## Трейсинг: Reactor Context, propagation для логгеров/трэйсов (OpenTelemetry)

Для сквозной трассировки используйте **Micrometer Tracing** с **OpenTelemetry** (экспорт в OTLP/Jaeger/Tempo/Zipkin). В реактивном мире контекст трассировки лежит в **Reactor Context** и автоматически пропагируется через операторы. Для логов подключите **MDC bridge** — значимые атрибуты (`traceId`, `spanId`, `user`, `tenant`) будут в каждом log-сообщении.

Если вы вручную создаёте реактивные последовательности, не теряйте контекст: используйте `deferContextual`/`contextWrite`. Для `WebClient` и WebFlux серверной части участие в Observations включено по умолчанию — заголовки `traceparent`/`b3` пройдут дальше.

```java
Mono<Void> doWork() {
  return Mono.deferContextual(ctx -> {
    String traceId = ctx.getOrDefault("traceId", "n/a");
    log.info("traceId={}", traceId);
    return Mono.empty();
  });
}

@Bean
WebFilter correlation() {
  return (ex, chain) -> {
    String id = Optional.ofNullable(ex.getRequest().getHeaders().getFirst("X-Request-Id"))
                        .orElse(UUID.randomUUID().toString());
    return chain.filter(ex).contextWrite(c -> c.put("traceId", id));
  };
}
```

## Тюнинг: размер event-loop, connection pool, буферы кодеков, GC-настройки

Reactor Netty по умолчанию создаёт **N** event-loop потоков (обычно = числу процессоров). Если у вас **много I/O** и немного CPU, дефолты обычно хороши. При CPU-насыщенных операциях отделите их в `Schedulers.parallel()` и контролируйте размер пула. Для `WebClient`/Netty клиента настройте **пул соединений**: `ConnectionProvider.builder("http").maxConnections(...).pendingAcquireMaxCount(...)`.

Буферы кодеков — параметр `spring.codec.max-in-memory-size` и `ServerCodecConfigurer.defaultCodecs().maxInMemorySize(...)`. Не ставьте слишком большие значения — лучше стримьте `DataBuffer` и ограничивайте размер входящих тел (`spring.webflux.multipart.max-*`, `http2.max-frame-size` при необходимости).

GC-тюнинг под высокую конкарентность: для JVM 17 часто подходит **G1** (дефолт) или **ZGC** для больших heap/низкой паузы. Следите за **выделениями** (избегайте лишних `.collectList()`), используйте **кэширование** сериализаторов Jackson, и профилируйте flame-графами (Async-profiler) «горячие» пути.

```java
@Bean
ReactorResourceFactory reactorResources() {
  // пример явного пула коннектов для WebClient'ов
  ConnectionProvider provider = ConnectionProvider.builder("http")
      .maxConnections(500)
      .pendingAcquireMaxCount(10000)
      .maxIdleTime(Duration.ofSeconds(30))
      .build();
  ReactorResourceFactory r = new ReactorResourceFactory();
  r.setUseGlobalResources(false);
  r.setConnectionProvider(provider);
  return r;
}
```

---

# 10. Тестирование WebFlux

## WebTestClient для контроллеров/функциональных маршрутов; проверка статус/заголовков/тела/стриминга

**`WebTestClient`** — основной инструмент тестирования WebFlux: он работает как «мини-клиент» поверх реактивного сервера (standalone или с поднятым контекстом). Он удобен для проверки статусов, заголовков, тела (JSON assertions), а также **стриминга** (SSE/NDJSON) — можно читать несколько элементов и отменить подписку.

Для slice-тестов используйте `@WebFluxTest(controllers=...)` (поднимет только веб-слой), для интеграционных — `@SpringBootTest(webEnvironment=RANDOM_PORT)` и инжект `WebTestClient` (Boot его автоконфигурирует). В реактивном мире проверка «части стрима» делается методами `.returnResult()` и работой с `Flux` вручную.

```java
@WebFluxTest(controllers = UserController.class)
class UserControllerTest {

  @Autowired WebTestClient client;

  @Test
  void getOkAndJson() {
    client.get().uri("/api/users/1")
        .exchange()
        .expectStatus().isOk()
        .expectHeader().contentTypeCompatibleWith(MediaType.APPLICATION_JSON)
        .expectBody()
        .jsonPath("$.id").isEqualTo(1);
  }

  @Test
  void sseTakesFirst3() {
    var result = client.get().uri("/api/ticks")
        .accept(MediaType.TEXT_EVENT_STREAM)
        .exchange()
        .expectStatus().isOk()
        .returnResult(Long.class);

    StepVerifier.create(result.getResponseBody().take(3))
        .expectNextCount(3)
        .thenCancel()
        .verify();
  }
}
```

## StepVerifier: виртуальное время, backpressure, тестирование retry/timeout

**Reactor Test** даёт `StepVerifier` — «пошаговый» проверщик реактивных последовательностей. Он поддерживает **виртуальное время** (`withVirtualTime`), что позволяет мгновенно тестировать `delay`, `timeout`, `retryWhen(backoff)` без реального ожидания. Это ускоряет тесты и делает их детерминированными.

Backpressure тестируется через `thenRequest(n)` — вы можете контролировать количество «запрошенных» элементов и проверять поведение оператора `onBackpressureDrop/buffer`. Это полезно в сценариях high-throughput, чтобы убедиться: источник не «затопит» подписчика.

Для `retry`/`timeout` важно проверять **число попыток** и корректное завершение (ошибка/успех). Можно «симулировать» последовательность ошибок и успешных ответов, используя `Flux.concat`/`Mono.error`.

```java
@Test
void retryBackoffWithVirtualTime() {
  var source = Flux.concat(
      Flux.error(new TimeoutException()),
      Flux.error(new TimeoutException()),
      Flux.just("ok"));

  Supplier<Flux<String>> retriable = () -> source
      .retryWhen(Retry.backoff(2, Duration.ofSeconds(1)))
      .timeout(Duration.ofSeconds(5));

  StepVerifier.withVirtualTime(retriable)
      .thenAwait(Duration.ofSeconds(1)) // first backoff
      .thenAwait(Duration.ofSeconds(2)) // second backoff
      .expectNext("ok")
      .expectComplete()
      .verify();
}
```

## Контракты клиентов: WireMock/MockWebServer с реактивным WebClient, негативные кейсы

Для тестирования `WebClient` удобно использовать **WireMock** или **OkHttp MockWebServer**. Оба позволяют поднимать локальный HTTP-сервер, возвращать нужные статусы/задержки/заголовки и проверять, что клиент корректно обрабатывает таймауты/ретраи/контент. С WebFlux это работает прозрачно — клиент неблокирующий, заглушка — обычный HTTP.

Покрывайте **негативные кейсы**: 429/503 с `Retry-After`, «висящие» ответы (чтобы сработал `timeout`), некорректный JSON (проверка `onErrorMap`). Это предотвращает «зависания» в проде и документирует поведение клиента при авариях.

```java
@Test
void clientTimesOutOnSlowEndpoint() throws Exception {
  var server = new MockWebServer();
  server.enqueue(new MockResponse().setBody("...").setBodyDelay(3, TimeUnit.SECONDS));
  server.start();
  try {
    var wc = WebClient.builder()
        .baseUrl(server.url("/").toString())
        .clientConnector(new ReactorClientHttpConnector(HttpClient.create()
            .responseTimeout(Duration.ofMillis(500))))
        .build();

    StepVerifier.create(wc.get().uri("/slow").retrieve().bodyToMono(String.class))
        .expectError(TimeoutException.class)
        .verify();
  } finally {
    server.shutdown();
  }
}
```

## Интеграции с данными: Testcontainers (Postgres R2DBC/Mongo/Redis), фикстуры, флейки и подавление гонок

Для реактивного доступа к данным отлично работают **Testcontainers**: поднимайте Postgres (R2DBC), Mongo, Redis на время теста. Прокидывайте свойства через `@DynamicPropertySource`, миграции схем гоняйте JDBC-инструментом (Flyway/Liquibase) **до** запуска реактивного кода. Так вы гарантируете согласованность схемы и избегаете блокировок в event-loop.

Фикстуры данных загружайте реактивно (`template.insert(...)`). Для изоляции используйте уникальные БД/схемы/префиксы ключей на тест, очищайте после теста или в `@BeforeEach`. «Флейки» в реактивных тестах часто происходят из-за **гонок** (нежёстко заданных ожиданий): применяйте `StepVerifier` вместо «уснуть на секунду», используйте `await` только как крайность, тестируйте **по сигналам**, а не по таймерам.

```java
@Testcontainers
@SpringBootTest
class MongoIT {

  @Container static MongoDBContainer mongo = new MongoDBContainer("mongo:7");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.data.mongodb.uri", mongo::getReplicaSetUrl);
  }

  @Autowired ReactiveMongoTemplate tpl;

  @Test
  void insertAndFind() {
    StepVerifier.create(tpl.insert(new Item(null, "acme", "wrench")))
        .expectNextCount(1).verifyComplete();

    StepVerifier.create(tpl.find(Query.query(Criteria.where("tenant").is("acme")), Item.class))
        .expectNextMatches(i -> i.name().equals("wrench"))
        .verifyComplete();
  }
}
```
