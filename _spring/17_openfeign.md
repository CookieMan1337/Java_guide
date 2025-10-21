---
layout: page
title: "OpenFeign: интеграции с другими сервисами/клиентами"
permalink: /spring/openfeign
---

# 0. Что такое OpenFeign

## Что это

OpenFeign — это **декларативный HTTP-клиент**: вы описываете интерфейс Java, аннотируете методы, а библиотека генерирует реализацию, которая собирает HTTP-запрос, отправляет его и превращает ответ в нужный вам тип. В связке со Spring мы чаще говорим про **Spring Cloud OpenFeign** — адаптацию Feign под экосистему Spring (аннотации `@FeignClient`, `@GetMapping`, интеграция с Jackson, Bean Validation, конфигурацией Boot, и т. п.). По сравнению с ручным использованием `RestTemplate`/`WebClient`, Feign снижает «клейкий» код: меньше конструирования URI, сериализации/десериализации и повторяющихся try/catch.

Важная мысль: Feign — это не «магический транспорт», а всего лишь удобный слой поверх **выбранного HTTP-стека**. Под капотом может работать JDK HttpClient (Java 11+), Apache HttpClient 5 или OkHttp. Это даёт гибкость: вы выбираете стек по требованиям к производительности, TLS/HTTP/2-возможностям и удобству настройки пулов.

Ещё одна ценность Feign — **совместимость с Spring**. Вы можете использовать привычные аннотации `@RequestParam`, `@PathVariable`, `@RequestHeader`, `@RequestBody` (через MVC-контракт) и привычные `HttpMessageConverters` (Jackson). Это снижает когнитивную нагрузку: интерфейс клиента выглядит так же, как контроллер.

Наконец, Feign хорош для **контрактного дизайна**: интерфейс клиента — это явный контракт на уровне кода. В сочетании с OpenAPI-генерацией это превращается в воспроизводимый, легко сопровождаемый слой интеграций между сервисами.

Простейший пример (Spring Cloud OpenFeign):

```kotlin
// build.gradle.kts (фрагмент)
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}
```

```java
@EnableFeignClients     // включает сканирование @FeignClient
@SpringBootApplication
public class App { public static void main(String[] args){ SpringApplication.run(App.class, args); } }
```

```java
@FeignClient(name = "countries", url = "${clients.countries.url}", path = "/v1")
interface CountryClient {

  @GetMapping("/countries/{code}")
  CountryDto one(@PathVariable String code, @RequestHeader("X-Request-Id") String rid);

  @GetMapping("/countries")
  PageDto<CountryDto> list(@RequestParam int page, @RequestParam int size);
}
```

## Зачем это

Главная причина использовать Feign — **снижение связующего кода** и повышение читабельности. Вместо ручной сборки URI, заголовков, сериализации и обработки ошибок вы объявляете метод и его параметры; инфраструктура делает остальное. Это экономит время, выравнивает стиль кода по всей команде и уменьшает вероятность «мелких» багов.

Вторая причина — **централизуемая конфигурация**: таймауты, пулы соединений, компрессия, логирование, пер-клиентные политики (ретраи, circuit-breaker) конфигурируются единообразно через `application.yml` или Java-бины. Это облегчает DevOps: можно менять параметры, не переписывая бизнес-код.

Третья — **расширяемость**. Feign поддерживает перехватчики запросов (`RequestInterceptor`) для авторизации (Bearer/JWT, API-Keys), корреляции (traceId, tenant), локализации (`Accept-Language`). Вы также можете подменить кодировщики/декодеры (например, добавить CBOR/ProtoBuf), и внедрить единый обработчик ошибок (`ErrorDecoder`).

Наконец, Feign хорошо вписывается в **DDD-архитектуру**: он естественный кандидат на роль **адаптера внешнего сервиса** в слое инфраструктуры. Внутри вашего домена вы оперируете чистыми моделями и портами, а Feign-клиенты остаются за анти-коррупционным слоем, преобразуя внешние DTO к внутренним.

## Где используется

Feign используют в **микросервисах**, где много S2S-вызовов: каталоги товаров, профили пользователей, платёжные шлюзы, геосервисы, биллинги. Особенно полезен там, где много однотипных интеграций: один интерфейс — один внешний сервис; легко поддерживать десятки клиентов с едиными правилами.

В интеграциях с внешними платформами (Stripe, PayPal, Google APIs) Feign помогает быстро описать ресурсные эндпоинты, типы запросов/ответов и повторяемые заголовки (например, `Idempotency-Key`). При этом остаётся гибкость выбора HTTP-стека: OkHttp — для продвинутых TLS/HTTP/2 и connection reuse; Apache HC5 — для зрелых корпоративных сетевых настроек.

Внутри **BFF-слоя** (backend-for-frontend) Feign позволяет красиво агрегировать данные из нескольких сервисов, не раздувая контроллеры вручную написанными вызовами. Поверх Feign удобно навесить правила устойчивости: `Retryer`, circuit breaker, rate limit.

Также Feign часто применяют как **«мост»** в проектах, где команда ещё не готова к полноценному переходу на реактивный стек: вы получаете лаконичный API-клиент в «классическом» синхронном мире Spring MVC.

## Какие задачи/проблемы решает

Во-первых, Feign решает проблему **дублирования кода** при HTTP-интеграциях: конструирование URL, сериализация тел, стандартные заголовки, обработка типовых ошибок. Всё это выносится в инфраструктуру, а бизнес-код концентрируется на смысле операций.

Во-вторых, Feign упрощает **консистентные настройки**: таймауты, пулы, gzip, логирование/маскирование полей, трейсинг. В результате уменьшаются «сюрпризы» в проде: клиенты ведут себя одинаково, и вы быстро находите девиации через конфигурацию.

В-третьих, Feign даёт **хуки для безопасности**: `RequestInterceptor` для токенов, подписи запросов, mTLS — всё на уровне клиента. Это закрывает частые «дыры», возникающие при ручных вызовах (забыли заголовок, не обновили токен, залогировали пароль).

Наконец, Feign — это **платформа расширения**. Вы можете добавить `ErrorDecoder`, чтобы маппить Problem Details из внешнего API в доменные исключения, подменить `Decoder` под нестандартные форматы, подключить OpenTelemetry для трассировки — и сделать всё это единообразно для множества клиентов.

---

# 1. Архитектура и роль OpenFeign

## Что такое Feign: декларативный клиент поверх OkHttp/Apache HC5/JDK11

Архитектурно Feign — это **динамический прокси**: при старте он строит реализацию интерфейса, используя выбранный **`Client`** (OkHttp/Apache/JDK). Каждый метод превращается в объект «шаблона запроса» (HTTP-метод, путь, параметры, заголовки, тело) и «стратегии декодирования» ответа. Дальше — дело транспорта: `Client` отправляет запрос, `Decoder` превращает тело в нужный тип.

Выбор клиента влияет на возможности. **OkHttp** даёт эффективный HTTP/2, connection pooling, TLS-фичи и хороший контроль таймаутов/keep-alive. **Apache HttpClient 5** — классика корпоративных настроек: прокси, тонкие TCP-параметры, mature-экосистема. **JDK HttpClient** (Java 11+) — без внешних зависимостей, c HTTP/2 и decent производительностью, но с чуть менее богатой конфигурацией.

В Spring Cloud OpenFeign всё это завернуто в авто-конфигурацию: достаточно добавить зависимость, пометить интерфейсы `@FeignClient`, и Boot поднимет нужное окружение. При необходимости вы можете заменить `Client` на per-client основе — например, одному клиенту дать OkHttp, другому HC5, третьему — JDK.

Важно понимать границы: Feign **не «умнее» сервера**. Если внешний сервис нарушает HTTP-контракт (неправильный `Content-Type`, битый JSON), задача «разрулить» это ложится на ваш `Decoder`/`ErrorDecoder` и политику устойчивости, а не на Feign как таковой.

Пример переключения на OkHttp:

```kotlin
dependencies {
  implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
  implementation("io.github.openfeign:feign-okhttp")
}
```

```java
@Bean
feign.Client feignOkHttpClient(okhttp3.OkHttpClient ok) {
  return new feign.okhttp.OkHttpClient(ok);
}

@Bean
okhttp3.OkHttpClient okHttp() {
  return new okhttp3.OkHttpClient.Builder()
      .connectTimeout(java.time.Duration.ofMillis(500))
      .readTimeout(java.time.Duration.ofSeconds(1))
      .build();
}
```

## Синхронная природа по умолчанию; когда нужен реактив

По умолчанию Feign — **синхронный**: метод клиента блокирует поток до получения ответа. В большинстве MVC-сервисов это нормально: у вас есть пул рабочих потоков, запросы относительно короткие, нагрузка умеренная. Но в **реактивном** приложении (WebFlux) блокировки ломают масштабируемость: один «зависший» HTTP-вызов блокирует event-loop.

Если вы в реактивном стеке — используйте **`WebClient`** или `feign-reactor` (альтернативные адаптеры Feign, возвращающие `Mono/Flux`). Но выбирать «Feign-reactor вместо WebClient» имеет смысл только если вы очень хотите сохранить декларативный интерфейс и общую экосистему. В противном случае «чистый» `WebClient` проще и богаче по реактивным возможностям (backpressure, стриминг, кодеки).

В смешанных проектах придерживайтесь правила: **блокирующие клиенты — в MVC**, **реактивные — в WebFlux**. Если по каким-то причинам вам нужен Feign в WebFlux, выносите его вызовы на `Schedulers.boundedElastic()` (оффлоад в отдельный пул) и ставьте короткие таймауты, чтобы не «залипать».

И наконец, не смешивайте в одном фасаде реактивные и синхронные вызовы без явной изоляции: это создаёт трудноотлавливаемые паузы и локи. Чётко разделите модули, и пусть каждый модуль использует «свой» стек.

## Место в слоистой архитектуре: адаптер внешнего сервиса, маппинг DTO, анти-коррупционный слой

В DDD мы выделяем **порты** (интерфейсы домена) и **адаптеры** (реализации под конкретные технологии). Feign-клиент — типичный **входной адаптер наружу**: он знает, «как говорить» с внешним API, но не «тащит» его модели в домен. Между Feign и доменом — **анти-коррупционный слой** (ACL): мапперы DTO → доменные объекты и обратно.

Такой слой защищает домен от **дрожащих контрактов** внешнего мира. Если внешний сервис добавил поле/поменял enum — вы правите маппер и внешние DTO, не касаясь доменной логики. Это же упрощает тестирование: можно мокировать порт домена, не поднимая весь Feign.

Отдельно храните **внешние модели** (например, пакет `com.example.integrations.geo.dto`) и **внутренние** (`com.example.domain.country`). Никогда не «протягивайте» внешние DTO до контроллеров вашего API — иначе вы привяжетесь к чужой эволюции.

Наконец, фасад поверх нескольких Feign-клиентов — удобное место для **сценарной логики** (fan-out/fan-in, кэширование, агрегация). Пусть контроллеры вызывают фасад, а фасад — клиентов, применяя политику таймаутов/ретраев и собирая результат в единый ответ.

Мини-пример фасада:

```java
@Service
class GeoFacade {
  private final CountryClient countries;
  private final CityClient cities;
  private final CountryMapper mapper;

  GeoFacade(CountryClient countries, CityClient cities, CountryMapper mapper) {
    this.countries = countries; this.cities = cities; this.mapper = mapper;
  }

  CountryDetails getCountry(String code, String rid) {
    var c = countries.one(code, rid);
    var top = cities.topByCountry(code, 5, rid);
    return mapper.toDetails(c, top);
  }
}
```

## Контракты: чистый OpenFeign (@RequestLine) vs Spring Cloud OpenFeign (MVC-аннотации)

У Feign есть «родной» контракт — аннотация **`@RequestLine`** (`"GET /countries/{code}"`) и `@Param` для подстановки. Spring Cloud OpenFeign по умолчанию использует **Spring MVC-контракт** — привычные `@GetMapping`, `@PostMapping`, `@RequestParam`, `@PathVariable`, `@RequestHeader`. Это удобно для команд, привыкших к Spring.

Если вы хотите «чистый Feign» (например, в библиотеках вне Spring), используйте `feign-core` и `@RequestLine`. В Spring-проектах практически всегда логичнее придерживаться MVC-контракта: меньше переключений контекста и отлично работает автогенерация Jackson/Validation.

Иногда полезно **явно** указать контракт (например, если вы смешиваете разные клиенты):

```java
@Configuration
class FeignContractsConfig {
  @Bean
  feign.Contract feignContract() {
    return new org.springframework.cloud.openfeign.support.SpringMvcContract();
  }
}
```

Пример «чистого» Feign (не Spring):

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);
}
```

---

# 2. Подключение и базовая конфигурация (Spring Cloud OpenFeign)

## Зависимости и `@EnableFeignClients`; организация пакетов

Чтобы включить OpenFeign в Spring Boot, добавьте стартер и включите сканирование клиентов аннотацией **`@EnableFeignClients`**. По умолчанию будут отсканированы пакеты приложения; если клиенты лежат в отдельном модуле/пакете, укажите `basePackages` или `basePackageClasses`.

Рекомендуем выделять **отдельный пакет/модуль** под внешние интеграции, например:
`com.example.integrations.<system>.client` — интерфейсы `@FeignClient`,
`com.example.integrations.<system>.dto` — внешние DTO,
`com.example.integrations.<system>.config` — per-client конфиг,
`com.example.integrations.<system>.impl` — мапперы/фасады.

Такой расклад облегчает миграции (например, заменить Feign на WebClient в одном месте), не затрагивая остальную кодовую базу. А ещё позволяет применять разные профили/билды для «внешних» модулей.

```kotlin
dependencies {
  implementation(platform("org.springframework.cloud:spring-cloud-dependencies:2023.0.3"))
  implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}
```

```java
@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.integrations")
public class App { /* ... */ }
```

## `@FeignClient(name/url/path)` и переопределение URL (config/Discovery)

Аннотация **`@FeignClient`** регистрирует интерфейс как бин и связывает его с именем (**`name`**). Атрибуты: `url` (базовый адрес) и `path` (общий префикс для всех методов). Если вы используете **Service Discovery** (Eureka/Consul), оставляйте `name`, а `url` не задавайте — резолвинг адресов сделает Ribbon/LoadBalancer (в новых версиях — Spring Cloud LoadBalancer).

URL удобно переопределять через конфиг (profiles/окружения), не меняя код. Для локальных тестов можно задать `http://localhost:8081`, для prod — DNS-имя/ingress. `path` полезен, когда у внешнего API общий префикс `"/api/v1"`.

```java
@FeignClient(name = "billing", url = "${clients.billing.url}", path = "/api/v1")
interface BillingClient {
  @PostMapping("/invoices")
  InvoiceDto create(@RequestBody CreateInvoiceDto body, @RequestHeader("Idempotency-Key") String key);
}
```

```yaml
# application.yml
clients:
  billing:
    url: https://billing.example.com
```

Если вы используете Discovery:

```yaml
feign:
  httpclient:
    enabled: true
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false
# И не задаём clients.billing.url — будет резолв через name=billing
```

## Таймауты/пулы: connect/read, maxConnections, keep-alive; GZIP

Feign сам по себе не управляет сокетами — это делает выбранный `Client`. В Spring Cloud есть **упрощённые свойства** для таймаутов:

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 500         # мс
        readTimeout: 1500           # мс
        loggerLevel: BASIC
        decode404: true
      billing:
        connectTimeout: 300
        readTimeout: 1000
```

Для **OkHttp**/Apache HC5 задайте **connection pool** и keep-alive:

```java
@Bean
okhttp3.OkHttpClient okHttp() {
  return new okhttp3.OkHttpClient.Builder()
      .connectionPool(new ConnectionPool(200, 30, java.util.concurrent.TimeUnit.SECONDS))
      .connectTimeout(java.time.Duration.ofMillis(500))
      .readTimeout(java.time.Duration.ofMillis(1500))
      .retryOnConnectionFailure(false)
      .build();
}
```

GZIP-сжатие можно включить двумя способами: (1) глобально (`feign.compression.response.enabled=true`, `feign.compression.request.enabled=true`), (2) отправлять/принимать с заголовками `Accept-Encoding: gzip` и `Content-Encoding: gzip`. Следите за MTU и CPU-ценой: сжатие выгодно для «толстых» JSON-ов, но избыточно для крошечных ответов.

## Per-client настройки: `configuration=…`, свойства `feign.client.config.<name>`

У `@FeignClient` есть атрибут **`configuration`**, куда вы можете передать класс(ы) с бинами, которые **будут видны только этому клиенту**: например, свой `Encoder/Decoder`, `ErrorDecoder`, `RequestInterceptor`, `Retryer`. Это позволяет иметь разные политики для разных интеграций.

Альтернатива — свойства с префиксом `feign.client.config.<name>` (как выше). Комбинируйте: общие дефолты — в `default`, индивидуальные — в секции клиента, а сложные случаи (нестандартная сериализация) — через Java-конфигурацию.

```java
@Configuration
class BillingFeignConfig {

  @Bean
  feign.codec.ErrorDecoder billingErrorDecoder() {
    return (methodKey, response) -> new BillingException("HTTP " + response.status());
  }

  @Bean
  feign.RequestInterceptor authInterceptor(TokenProvider tokens) {
    return tpl -> tpl.header("Authorization", "Bearer " + tokens.get());
  }
}

@FeignClient(name = "billing", configuration = BillingFeignConfig.class)
interface BillingClient { /* ... */ }
```

---

# 3. Контракты запросов/ответов

## Аннотации: `@GetMapping/@PostMapping/@RequestParam/@PathVariable/@RequestHeader/@RequestPart`

Spring Cloud OpenFeign поддерживает **MVC-аннотации** для декларации HTTP-контракта. Это удобно и привычно: вы описываете метод почти так же, как контроллер, только вместо `@RestController` — `@FeignClient`. Параметры пути помечаются `@PathVariable`, query-параметры — `@RequestParam`, заголовки — `@RequestHeader`, тело — `@RequestBody`.

Важная тонкость — **совпадение имён**: имя в `@PathVariable("id")` должно соответствовать `{id}` в пути, иначе маппинг не произойдёт. Для заголовков используйте точные названия, учитывая регистр (в HTTP заголовки case-insensitive, но лучше придерживаться одного стиля).

Для multipart-загрузок используйте `@RequestPart` и типы `org.springframework.core.io.Resource`/`MultipartFile` (в клиенте обычно `Resource`). Для скачивания файлов верните `feign.Response` или `ResponseEntity<byte[]>`/`InputStream` (см. ниже про стриминг и закрытие).

```java
@FeignClient(name = "files", url = "${clients.files.url}", path = "/api")
interface FileClient {

  @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
  UploadResult upload(@RequestPart("file") org.springframework.core.io.Resource file,
                      @RequestPart("meta") MetaDto meta);

  @GetMapping(value = "/download/{id}")
  feign.Response download(@PathVariable String id, @RequestHeader("X-Request-Id") String rid);
}
```

## Сериализация: Jackson (encoders/decoders), дата/время, BigDecimal, null-handling

По умолчанию Spring Cloud OpenFeign использует **Jackson** через Spring-конвертеры. Это значит, что ваши настройки `ObjectMapper` (модули JavaTime, правила для `BigDecimal`, форматы дат) будут применяться и к клиентам. Обязательно подключайте `jackson-datatype-jsr310` (идёт в стартере) и настраивайте форматы ISO-8601 (`OffsetDateTime`, `LocalDate`) — это снизит расхождения с внешними API.

Для денег используйте `BigDecimal` и явные **скейлы** в схемах/DTO, не полагайтесь на двоичную точность типов `double/float`. Если внешний API ожидает строки (например, `"price":"12.34"`), настройте сериализацию под это правило (кастомный сериализатор либо `@JsonFormat(shape=STRING)`).

`null`-handling: по умолчанию Jackson не сериализует `null` (если вы это включили в конфиге), но внешние API могут ожидать **пустые поля**. Для пер-клиентной настройки можно сделать кастомный `Encoder` с отдельным `ObjectMapper` (только для одного клиента), или держать DTO с корректными default-значениями.

```java
@Configuration
class FeignJacksonConfig {
  @Bean
  feign.codec.Encoder feignEncoder(ObjectFactory<HttpMessageConverters> converters) {
    return new org.springframework.cloud.openfeign.support.SpringEncoder(converters);
  }
  @Bean
  feign.codec.Decoder feignDecoder(ObjectFactory<HttpMessageConverters> converters) {
    return new org.springframework.cloud.openfeign.support.SpringDecoder(converters);
  }
}
```

## Multipart и файлы: загрузка, скачивание — безопасный стриминг и закрытие

При загрузке файлов **не буферизуйте** огромные байтовые массивы в памяти. Используйте `Resource` (например, `FileSystemResource`/`ByteArrayResource`) и пусть Feign/HTTP-клиент сам стримит данные. Добавляйте ограничения размера на стороне сервера и клиента (если поддерживаются).

При скачивании больших файлов лучше возвращать **`feign.Response`** и читать тело потоком, не держа всё в памяти. Обязательно **закрывайте** `Response` (или используйте try-with-resources), иначе потечёт соединение/память. Учитывайте заголовки `Content-Type`/`Content-Disposition`, чтобы корректно обрабатывать скачивание.

```java
try (feign.Response resp = fileClient.download(id, rid)) {
  if (resp.status() != 200) throw new RuntimeException("Failed " + resp.status());
  try (var in = resp.body().asInputStream();
       var out = Files.newOutputStream(Path.of("/tmp/file.bin"))) {
    in.transferTo(out);
  }
}
```

## Пагинация/фильтры: общие модели, `@QueryMap` для динамики

Для унификации запросов с фильтрами удобно завести **общий DTO** (например, `PageRequest { page, size, sort }`) и передавать его как набор query-параметров. Когда набор параметров динамический (например, произвольный фильтр), удобна аннотация **`@QueryMap`** — она преобразует POJO/Map в query-строку.

Следите за соглашениями: сортировка (`sort=field,asc` или повторяемые параметры), многозначные фильтры (`tags=a&tags=b`) и кодирование значений (URL-encode). Лучше сразу описать это в контракте/документации и поддерживать единый формат по всем клиентам.

```java
record SearchParams(Integer page, Integer size, String sort, String q) {}

@FeignClient(name = "search", url = "${clients.search.url}")
interface SearchClient {
  @GetMapping("/items")
  PageDto<ItemDto> find(@QueryMap SearchParams params);
}
```

отлично — продолжаю «Интеграции: OpenFeign» и подробно раскрываю **подтемы 4–7**. Для каждого подпункта — 4+ абзацев и практический код (Java 17+, Spring Boot 3.x, Spring Cloud 2023.x).

---

# 4. Надёжность и устойчивость

## Retryer: отключить дефолтные ретраи на небезопасных методах, настроить backoff

По умолчанию OpenFeign может ретраить некоторые сетевые ошибки. Это удобно при «флаппи» сети, но **опасно для небезопасных методов** (POST/PUT/PATCH/DELETE), где повтор может создать дубликаты. Первая практика — **выключить дефолтные ретраи** и включать их точечно только для **идемпотентных** операций (GET/HEAD) и конкретных ошибок (408/429/5xx). Настройка делается глобально и/или per-client через `Retryer` и свойства `feign.client.config.*`.

Экспоненциальный бэкофф снижает «громыхание» по внешнему API и помогает избежать «штормов» ретраев. Но важно держать **жёсткую «крышу»** по времени (time limiter/общий таймаут на операцию), чтобы запрос не «жил» бесконечно. Ретраить следует только ошибки, где есть шанс на успех при повторе: таймауты, 502/503/504, 429 (с уважением к `Retry-After`).

Ещё одна тонкость — **дедупликация** на стороне сервера. Даже для GET полезно реплеить запросы с одинаковым `X-Request-Id`: если повтор — сервер сможет связать два запроса и вернуть кэшированный ответ. Для POST — это **Idempotency-Key** (см. ниже).

Пер-клиентная конфигурация ретраев:

```yaml
feign:
  client:
    config:
      default:
        retryer: com.example.feign.NoRetryer  # свой retryer: вообще без ретраев
      catalog:
        connectTimeout: 500
        readTimeout: 1200
        loggerLevel: BASIC
        # можно также указать класс кастомного Retryer для конкретного клиента
```

```java
public class NoRetryer implements feign.Retryer {
  @Override public void continueOrPropagate(RetryableException e) { throw e; }
  @Override public Retryer clone() { return this; }
}
```

## Circuit breaker/bulkhead/time limiter (Resilience4j), fallbacks и их границы

**Circuit breaker** (предохранитель) размыкает «цепь» при частых ошибках, предотвращая лавину неудачных вызовов. В Spring Cloud OpenFeign интеграция с **Spring Cloud CircuitBreaker** включается свойством `feign.circuitbreaker.enabled=true`, а реализацию даёт **Resilience4j**. Помимо CB используйте **bulkhead** (ограничение параллельности) и **time limiter** (жёсткая «крышка» на время операции).

**Fallback** — запасной сценарий при ошибке: вернуть заглушку, пустой ответ, кешированное значение либо поднять доменное исключение. Но у fallback есть границы: его не стоит злоупотреблять для критичных путей, чтобы не «маскировать» реальные инциденты. Для небезопасных операций (создание платежа) безопаснее возвращать ошибку и позволять оркестратору принять решение (повтор/компенсация).

Подключение CB для Feign и локальный fallback:

```yaml
feign:
  circuitbreaker:
    enabled: true
resilience4j:
  circuitbreaker:
    instances:
      billing:
        registerHealthIndicator: true
        slidingWindowSize: 20
        failureRateThreshold: 50
  timelimiter:
    instances:
      billing:
        timeoutDuration: 1s
```

```java
@FeignClient(name = "billing", configuration = BillingFeignConfig.class, fallbackFactory = BillingFallbacks.class)
interface BillingClient {
  @PostMapping("/api/v1/invoices")
  InvoiceDto create(@RequestBody CreateInvoiceDto body, @RequestHeader("Idempotency-Key") String key);
}

@Component
class BillingFallbacks implements feign.hystrix.FallbackFactory<BillingClient> {
  @Override public BillingClient create(Throwable cause) {
    return (body, key) -> { throw new BillingUnavailableException("billing down", cause); };
  }
}
```

> Примечание: начиная с современных версий Spring Cloud, fallback работает через CircuitBreaker API; для новых проектов используйте `feign.circuitbreaker.enabled` и `FallbackFactory` из OpenFeign адаптера (без Hystrix).

## Идемпотентность и повторная отправка: `Idempotency-Key`, safe-мутаторы, дедупликация

Для **мутаторов** (например, создание счёта) обязательно используйте **Idempotency-Key** — уникальный ключ операции, по которому сервер распознает повтор и вернёт **тот же ответ**, не создавая дубль. Ключ генерируйте на клиенте (UUIDv4/v7), храните в логах вместе с `traceId` и прокидывайте в заголовке.

Там, где допустимо, делайте операции **идемпотентными по дизайну**: `PUT` «создай или обнови», `DELETE` «удали, если есть» — и тогда ретраи будут безопаснее. У сервера должна быть **хранилка ключей** (Redis/БД) с TTL, чтобы обслуживать повторные запросы.

В клиенто-фасаде полезно инкапсулировать «генерацию ключа + вызов клиента», чтобы команда не забывала добавлять заголовок. Пример:

```java
@Service
class BillingGateway {
  private final BillingClient client;
  BillingGateway(BillingClient client) { this.client = client; }

  public InvoiceDto createInvoice(CreateInvoiceDto dto) {
    String idem = java.util.UUID.randomUUID().toString();
    return client.create(dto, idem);
  }
}
```

## Ограничение скорости (rate limiting) и «вежливость» к внешним API

Даже самый надёжный предохранитель не спасёт, если вы **перегружаете** внешнее API. Реализуйте **client-side rate limiting**: глобальный лимит на клиента и/или per-route. С Resilience4j это делается `RateLimiter`, которым можно обернуть вызов feign-клиента в фасаде.

Если внешний сервис возвращает **429/`Retry-After`**, уважайте его — распарсьте заголовок и подождите. Внутри компании согласуйте «контракты» по лимитам, чтобы избежать банов.

```java
@Service
class SearchGateway {
  private final SearchClient client;
  private final io.github.resilience4j.ratelimiter.RateLimiter rl;

  SearchGateway(SearchClient client, io.github.resilience4j.ratelimiter.RateLimiterRegistry reg) {
    this.client = client;
    this.rl = reg.rateLimiter("search");
  }

  public PageDto<ItemDto> find(SearchParams params) {
    java.util.function.Supplier<PageDto<ItemDto>> sup = () -> client.find(params);
    return io.github.resilience4j.ratelimiter.RateLimiter.decorateSupplier(rl, sup).get();
  }
}
```

---

# 5. Ошибки и исключения

## Поведение по умолчанию: `FeignException` для 4xx/5xx; `decode404`

По умолчанию, если сервер вернул 4xx/5xx, OpenFeign выбрасывает **`FeignException`** (подклассы: `FeignException.BadRequest`, `NotFound`, `InternalServerError` и т. п.). Это разумная базовая семантика, но беда в том, что она **не доменная**: бизнес-слою сложно отличить «недостаточно средств» от «временная ошибка шлюза». Для 404 можно включить `decode404=true`, чтобы декодер попытался вернуть `null`/пустой результат вместо исключения (аккуратно, это может скрыть проблемы).

Лучшая практика — подключить **`ErrorDecoder`** и маппировать статусы/тела на **свои исключения**, где код и поля понятны домену. Так вы перестанете «течь» HTTP-семантикой вверх по слоям, а фасад сможет принимать осмысленные решения.

Держите в голове: исключение — это тоже «контракт». Если вы будете бросать голые `RuntimeException`, отладка станет болью. Делайте чёткую иерархию: `RemoteClientException` ← `BillingUnavailableException`, `InsufficientFundsException`, `RateLimitedException`…

## `ErrorDecoder`: маппинг Problem Details/тел на доменные исключения

Часто внешние API возвращают **Problem Details (RFC-7807)**. Удобно распарсить его в модель и бросить доменное исключение с сохранением `type/title/detail` и полезных `properties`. `ErrorDecoder` получает `Response`: прочитайте `body` и попробуйте десериализовать; если не удалось — добавьте в исключение «сырое» тело (по обрезке до N символов).

Важно закрыть `Response.body()` — используйте try-with-resources. Старайтесь не логировать **полное** тело ошибки, если оно может содержать персональные данные; логируйте `traceId`, `problem.type`, `status`.

```java
class ProblemDetails {
  public String type, title, detail;
  public Integer status;
  public Map<String,Object> properties;
}

@Configuration
class CommonFeignErrorHandling {

  @Bean
  feign.codec.ErrorDecoder problemDecoder(com.fasterxml.jackson.databind.ObjectMapper om) {
    return (methodKey, response) -> {
      try (var body = response.body() != null ? response.body().asInputStream() : null) {
        if (body != null && response.headers().containsKey("Content-Type")) {
          ProblemDetails p = om.readValue(body, ProblemDetails.class);
          return switch (response.status()) {
            case 400 -> new BadRemoteRequestException(p.title, p.detail);
            case 401, 403 -> new RemoteAuthException(p.title);
            case 404 -> new RemoteNotFoundException(p.title);
            case 429 -> new RemoteRateLimitedException(p.title);
            default -> new RemoteServerException("Upstream error " + response.status() + ": " + p.title);
          };
        }
      } catch (Exception ignore) { /* fallback ниже */ }
      return new RemoteServerException("HTTP " + response.status());
    };
  }
}
```

## Ретрай только на «временных» ошибках; увязка с идемпотентностью

Ретрай — это **про стратегия** ошибок. Не ретрайте 4xx, кроме 408/429 (и то с уважением к `Retry-After`). Ретрайте ограниченное число раз **только идемпотентные** операции. Для POST — используйте `Idempotency-Key` и убедитесь, что сервер действительно поддерживает его семантику.

Комбинируйте Retryer с CircuitBreaker/TimeLimiter. Если операция «горит» в time limiter — ретрай дальше редко поможет. Сначала убедитесь, что **timeout** адекватен (не слишком мал для реальных SLO), и только потом настройте ретраи на сетевые «дрожания».

Хорошая практика — вести **метрики** попыток/успехов после ретраев. Если половина запросов успешны только со второй/третьей попытки, проблема шире и её надо решать с владельцем внешнего API.

## Трассировка причин: корреляция `requestId/traceId/spanId`

Чтобы расследовать ошибки, вам нужен путь «сквозной» корреляции. Передавайте **`X-Request-Id`** (или аналог) и **трейсинговые заголовки** (`traceparent`, `b3`) во все исходящие запросы. Записывайте эти значения в логи и прикладывайте в исключения (минимум — `requestId` и код ответа).

В ответах сервер может возвращать собственный `requestId`. Сохраняйте его и прокидывайте выше — это поможет поддержке внешней стороны быстрее найти нужную запись. В ErrorDecoder можно положить `remoteRequestId` в доменное исключение как поле/причину.

```java
@Component
class CorrelationInterceptor implements feign.RequestInterceptor {
  @Override public void apply(feign.RequestTemplate tpl) {
    String rid = java.util.Optional.ofNullable(tpl.headers().get("X-Request-Id"))
        .filter(v -> !v.isEmpty()).map(v -> v.iterator().next()).orElseGet(() -> java.util.UUID.randomUUID().toString());
    tpl.header("X-Request-Id", rid);
  }
}
```

---

# 6. Безопасность и заголовки

## `RequestInterceptor`: авторизация (Bearer/JWT/OAuth2 client credentials)

Чаще всего защищённые внешние API требуют **Bearer JWT**. Если сервис-сервис коммуникация идёт по **client credentials** (машинная учётка), используйте `OAuth2AuthorizedClientManager`: он сам получит/обновит токен и отдаст актуальный. Реализуйте `RequestInterceptor`, который добавляет `Authorization: Bearer <token>` в каждый запрос клиента.

Если вы принимаете запрос от пользователя и хотите **«ретранслировать»** его токен во внешний сервис (token relay), извлеките его из `SecurityContextHolder` (например, `JwtAuthenticationToken`) и установите в заголовок. Обязательно следите за **истечением срока**: если токен пользователя «вот-вот» истечёт, внешний сервис может отказать.

```java
@Configuration
class OAuth2FeignConfig {

  @Bean
  public feign.RequestInterceptor clientCredentialsInterceptor(
      org.springframework.security.oauth2.client.OAuth2AuthorizedClientManager manager) {
    return tpl -> {
      var req = org.springframework.security.oauth2.client.OAuth2AuthorizeRequest
          .withClientRegistrationId("billing-cc")
          .principal("feign") // технический principal
          .build();
      var client = manager.authorize(req);
      String token = client.getAccessToken().getTokenValue();
      tpl.header("Authorization", "Bearer " + token);
    };
  }
}
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          billing-cc:
            provider: keycloak
            client-id: billing-service
            client-secret: ${BILLING_CLIENT_SECRET}
            authorization-grant-type: client_credentials
        provider:
          keycloak:
            token-uri: https://idp.example.com/realms/acme/protocol/openid-connect/token
```

## mTLS/сертификаты клиента, доверенные корни, пиннинг

Для **mTLS** (взаимная аутентификация TLS) клиент должен предъявлять **клиентский сертификат**. На стороне Feign (OkHttp/Apache) настройте SSL-контекст с **KeyManager** (клиентский ключ) и **TrustManager** (доверенные корни). Spring Boot может автоматически сконфигурировать SSL для **`RestTemplate`/`WebClient`**, но для Feign на OkHttp настройте `OkHttpClient` вручную и передайте его в `feign.Client`.

Пиннинг сертификата/публичного ключа (certificate/public-key pinning) — дополнительный уровень защиты против атаки «злой СА». В OkHttp это `CertificatePinner`. Пины обновляйте заранее (двойная публикация), иначе при ротации на стороне сервера вы «отрежете» себя.

```java
@Bean
okhttp3.OkHttpClient mtlsOkHttp(javax.net.ssl.SSLSocketFactory ssl, javax.net.ssl.X509TrustManager tm) {
  return new okhttp3.OkHttpClient.Builder()
      .sslSocketFactory(ssl, tm)
      .certificatePinner(new okhttp3.CertificatePinner.Builder()
          .add("billing.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
          .build())
      .build();
}
```

## Маскирование чувствительных данных в логах, защита от header-инъекций

Не логируйте **Authorization**, **Cookie**, **Set-Cookie**, **API-Keys** и телá с PII. У Feign есть встроенный `Logger`, но он «честно» выведет всё. Решение — **кастомный Logger**, который замаскирует значения чувствительных заголовков и ограничит размер тела. На проде выбирайте `BASIC`/`HEADERS` максимум, **никогда не `FULL`**.

Header injection (CRLF-инъекции в заголовки) предотвратите валидацией/нормализацией значений ключевых заголовков, приходящих снаружи (например, `X-Request-Id`). В перехватчиках не позволяйте raw-подстановки без проверок.

```java
class MaskingFeignLogger extends feign.Logger {
  @Override
  protected void logRequest(String configKey, Level logLevel, feign.Request request) {
    var masked = new java.util.LinkedHashMap<String, java.util.Collection<String>>(request.headers());
    masked.computeIfPresent("Authorization", (k,v) -> java.util.List.of("******"));
    super.logRequest(configKey, logLevel, request.toBuilder().headers(masked).build());
  }
}
```

## Токен-релей в микросервисах: propagation контекста и истечения сроков

В микросервисах часто нужно «ретранслировать» пользовательский токен во внешний сервис. Самый простой способ — `RequestInterceptor`, который берёт текущий `Authentication` и прокидывает `Bearer`. Важно учитывать **scope**: у внешнего сервиса может быть другой набор scope’ов, и пользовательский токен может оказаться «слишком слабым». Тогда используйте **token exchange** (если поддерживает ваш IdP) или делегируйте доступ через **client credentials** с «делегированным» scope.

Следите за **истечением**: если токен вот-вот истечёт, внешний сервис может ответить 401. Примите решение: (а) не ретранслировать просроченный токен, (б) ловить 401 и пытаться сделать **refresh** — но это уже про публичного клиента, не про server-to-server.

```java
@Component
class TokenRelayInterceptor implements feign.RequestInterceptor {
  @Override public void apply(feign.RequestTemplate tpl) {
    var auth = org.springframework.security.core.context.SecurityContextHolder.getContext().getAuthentication();
    if (auth instanceof org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken jwt) {
      tpl.header("Authorization", "Bearer " + jwt.getToken().getTokenValue());
    }
  }
}
```

---

# 7. Наблюдаемость и производительность

## Логирование: `Logger.Level` per-client; не логировать PII/секреты

Feign поддерживает уровни логирования: `NONE`, `BASIC`, `HEADERS`, `FULL`. На **проде** используйте `BASIC` (метод/URL/статус/время) или, максимум, `HEADERS` (без тела). На **dev** временно включайте `FULL` для проблемных клиентов, но маскируйте секреты и ограничивайте размер тела (см. кастомный Logger выше).

Уровень можно задать **per-client** через свойства:

```yaml
feign:
  client:
    config:
      default:
        loggerLevel: BASIC
      billing:
        loggerLevel: HEADERS
```

Также полезно логировать **сжатые** тела (gzip) как «<gzipped N bytes>», не распаковывая — это снизит CPU/память. Логируйте **идентификаторы** (`X-Request-Id`, `traceId/spanId`), но не payload.

## Метрики: Micrometer таймеры/счётчики, распределение задержек

Для метрик добавьте зависимость **`feign-micrometer`**: она включает **Micrometer Observation** вокруг вызовов Feign. Spring Boot подтянет `ObservationRegistry`, и вы получите таймеры по клиенту/методу/статусу. Включайте **гистограммы** p95/p99, чтобы видеть хвосты задержек.

```kotlin
dependencies {
  implementation("io.github.openfeign:feign-micrometer")
  implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

```yaml
management:
  endpoints.web.exposure.include: health,metrics,prometheus
  metrics:
    distribution:
      percentiles-histogram:
        feign.client.requests: true
```

Если хотите тоньше — добавьте `FeignBuilderCustomizer` и зарегистрируйте `MicrometerObservationCapability` вручную (обычно не нужно, Spring Cloud делает это сам).

```java
@Bean
org.springframework.cloud.openfeign.FeignBuilderCustomizer micrometerCustomizer(io.micrometer.observation.ObservationRegistry or) {
  return builder -> builder.addCapability(new feign.micrometer.MicrometerObservationCapability(or));
}
```

## Трейсинг: OpenTelemetry/Micrometer Tracing; корреляция с серверными спанами

Добавьте **Micrometer Tracing + OpenTelemetry** (bridge + exporter), и ваши Feign-вызовы появятся как **клиентские спаны** в трейсерe (Jaeger/Tempo/Zipkin/OTel Collector). Это сильно помогает видеть «бутерброд» запросов: входной серверный спан → клиентский спан Feign → серверный спан внешнего сервиса (если он тоже в трейсинге).

С заголовками всё сделает автоконфигурация: `traceparent`/`b3` пойдут наружу из `Observation`. Главное — **не перетирайте** их своими `RequestInterceptor` и аккуратно добавляйте кастомные заголовки (например, `X-Request-Id`).

```kotlin
dependencies {
  implementation("io.micrometer:micrometer-tracing-bridge-otel")
  runtimeOnly("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

```yaml
management:
  tracing:
    enabled: true
```

## Тюнинг HTTP-клиента: DNS cache, connection reuse, HTTP/2, компрессия

Производительность Feign на 90% — это производительность **клиентского HTTP-стека**. Для **OkHttp** настройте **connection pool** (размер/время жизни), **keep-alive**, **HTTP/2** (включён по умолчанию), **DNS** (кастомный резолвер с кешем, если надо), **pingInterval** для поддержания HTTP/2. Включайте **gzip** для «толстых» JSON и отключайте для мелких ответов.

Для **Apache HC5** — настройте `PoolingAsyncClientConnectionManager`, политику `timeToLive`, `ValidateAfterInactivity`, лимиты на route/total. Для **JDK HttpClient** — меньше ручек, но есть `version(HTTP_2)`, `connectTimeout`, `followRedirects`, «пул» создаётся на клиенте — **переиспользуйте** его, не создавайте на каждый запрос.

OkHttp пример (переподключение к Feign):

```java
@Bean
okhttp3.OkHttpClient tunedOkHttp() {
  return new okhttp3.OkHttpClient.Builder()
      .connectionPool(new okhttp3.ConnectionPool(300, 30, java.util.concurrent.TimeUnit.SECONDS))
      .followRedirects(false)
      .retryOnConnectionFailure(false)
      .connectTimeout(java.time.Duration.ofMillis(300))
      .readTimeout(java.time.Duration.ofMillis(1200))
      .callTimeout(java.time.Duration.ofSeconds(2)) // жёсткая «крышка»
      .build();
}

@Bean
feign.Client feignOkHttpClient(okhttp3.OkHttpClient ok) {
  return new feign.okhttp.OkHttpClient(ok);
}
```

Компрессия (глобально):

```yaml
feign:
  compression:
    request:
      enabled: true
      mime-types: application/json
      min-request-size: 1024
    response:
      enabled: true
```

отлично — завершаем «Интеграции: OpenFeign» подробным разбором **подтем 8–10**. Для каждого подпункта — 4+ абзацев, практические советы и рабочие фрагменты кода (Java 17+, Spring Boot 3.x, Spring Cloud OpenFeign 2023.x).

---

# 8. Расширяемость и продвинутые паттерны

## Интерсепторы цепочкой: корреляция, локализация (`Accept-Language`), кастомные заголовки

`RequestInterceptor` — это «точка расширения» Feign. Хорошая практика — собирать **цепочку** небольших интерсепторов, каждый из которых решает **одну** задачу: корреляция (`X-Request-Id`/`traceId`), локализация (`Accept-Language` на основе текущего пользователя/тенанта), кастомные заголовки домена (например, `X-Tenant-Id`). Так проще тестировать, переиспользовать и выключать куски поведения по профилям.

Корреляция нужна для трассировки и поддержки: генерируем/прокидываем `X-Request-Id`, если его нет, а также не перетираем уже проставленный (внутрисервисные вызовы). Для локализации тянем язык из `LocaleContextHolder` или из `SecurityContext` (профиль пользователя) и формируем стандартный заголовок `Accept-Language: ru-RU,ru;q=0.9`.

```java
@Component
class RequestIdInterceptor implements feign.RequestInterceptor {
  @Override public void apply(feign.RequestTemplate t) {
    var headers = t.headers().get("X-Request-Id");
    if (headers == null || headers.isEmpty()) {
      t.header("X-Request-Id", java.util.UUID.randomUUID().toString());
    }
  }
}

@Component
class LocaleInterceptor implements feign.RequestInterceptor {
  @Override public void apply(feign.RequestTemplate t) {
    var locale = org.springframework.context.i18n.LocaleContextHolder.getLocale();
    if (locale != null) {
      t.header("Accept-Language", locale.toLanguageTag());
    }
  }
}
```

Если нужно **условное** поведение (например, интерсептор включён только для конкретного клиента), используйте `configuration = …` в `@FeignClient` и объявляйте интерсептор в этой конфигурации, а не как `@Component`. Порядок интерсепторов важен (корреляция раньше логирования), поэтому регистрируйте их предсказуемо.

## Версионирование внешних API: media type versioning, разные клиенты на версии

У внешних API часто несколько версий. Два главных подхода: **path versioning** (`/api/v2/...`) и **media type versioning** (`Accept: application/vnd.billing.v2+json`). Для первого удобно задать `path=" /api/v2"` в `@FeignClient`. Для второго — добавить интерсептор `Accept` и, при необходимости, иной `Decoder`/`Encoder` (например, когда меняется формát даты/денег).

Поддерживать «всё в одном» клиенте тяжело: появятся условные поля, сложные мапперы и ветвления. Проще и надёжнее — **два разных клиента** (V1/V2) и чёткий фасад, который решает, кого звать. Так вы сможете независимо выключать старую версию и увидеть «кто ещё её дергает» по зависимостям.

```java
@FeignClient(name = "billingV2", url="${clients.billing.url}", configuration = BillingV2Cfg.class)
interface BillingV2Client { /* методы v2 */ }

@Configuration
class BillingV2Cfg {
  @Bean feign.RequestInterceptor v2Accept() {
    return tpl -> tpl.header("Accept", "application/vnd.billing.v2+json");
  }
}
```

При смене версии не забывайте о **совместимости**: если внешняя сторона меняет семантику статус-кодов/полей, обновляйте `ErrorDecoder` и мапперы. И держите **фича-флаг** в фасаде, чтобы переключать версии по окружениям/тенантам.

## Политика схем: OpenAPI-first → генерация Feign клиентов (openapi-generator), `oneOf`/`nullable`

OpenAPI-first — зрелый процесс для интеграций. Вы держите спецификацию (YAML/JSON) как источник истины, а клиентские интерфейсы и модели генерируете из неё. Это снижает ручные ошибки и держит контракт в синхроне. Для Feign используйте **OpenAPI Generator** с библиотекой `feign`.

Пример Gradle-плагина (OpenAPI Generator):

```kotlin
plugins { id("org.openapi.generator") version "7.5.0" }

openApiGenerate {
  generatorName.set("java")
  inputSpec.set("$rootDir/openapi/billing-v2.yaml")
  outputDir.set("$buildDir/generated/billing")
  apiPackage.set("com.example.integrations.billing.client")
  modelPackage.set("com.example.integrations.billing.dto")
  additionalProperties.set(mapOf(
    "library" to "feign",
    "dateLibrary" to "java8",
    "useBeanValidation" to "true",
    "hideGenerationTimestamp" to "true"
  ))
}
sourceSets["main"].java.srcDir("$buildDir/generated/billing/src/main/java")
```

Сложности: **`oneOf`/`anyOf`/`allOf`** порождают иерархии, которые не всегда удобно маппить в домен. Чаще используйте `discriminator` и избегайте «гипер-полиморфных» моделей; для `nullable` проверьте соответствие Jackson/генератора (иногда нужeн `JsonNullable` и отдельный модуль). И обязательно включайте **валидаторы** схем на CI, чтобы не сломать генерацию.

## Стриминг/SSE/длинные ответы: буферы, чтение кусками, защита от OOM

Feign — **синхронный** и по умолчанию стремится загрузить тело ответа целиком для декодирования в объект. Для **длинных** ответов (скачивание файла, NDJSON-стрим) возвращайте `feign.Response` и читайте **поточнo**, без полного буферинга. Контролируйте размер буфера, закрывайте ресурсы, ставьте **жёсткие таймауты** и лимиты размера (если поддерживает сервер).

SSE в «чистом» Feign — плохая идея: лучше использовать специализированный клиент (WebClient/OkHttp streaming). Если выбора нет — читайте `InputStream` построчно, обрабатывайте и **вовремя отменяйте** операцию. Никогда не логируйте целиком «толстые» тела.

```java
try (var resp = reportClient.download(reportId)) {
  if (resp.status() != 200) throw new RuntimeException();
  try (var in = new java.io.BufferedReader(new java.io.InputStreamReader(resp.body().asInputStream()))) {
    String line; int count = 0;
    while ((line = in.readLine()) != null) {
      // обработать NDJSON строку
      if (++count > 10_000) break; // предохранитель
    }
  }
}
```

---

# 9. Тестирование клиентов

## Unit: мок энкодеров/декодеров, `ErrorDecoder`, `RequestInterceptor`

Юнит-тесты для «инфраструктурных» классов дешевы и полезны. Для `RequestInterceptor` достаточно создать `RequestTemplate`, вызвать `apply()` и проверить заголовки. Для `ErrorDecoder` — сконструировать `Response` (статус + тело) и проверить, что маппится в **нужное доменное исключение**.

Не забывайте краевые случаи: пустое тело, неправильный `Content-Type`, битый JSON — ваш декодер должен вести себя предсказуемо (fallback, безопасное сообщение). И проверьте маскирование в кастомном `Logger` (секреты не попали в логи).

```java
@Test
void interceptorAddsLanguage() {
  var t = new feign.RequestTemplate();
  new LocaleInterceptor().apply(t);
  assertThat(t.headers()).containsKey("Accept-Language");
}

@Test
void errorDecoderMapsProblem() throws Exception {
  var body = "{\"title\":\"Rate limit\",\"status\":429}";
  var resp = feign.Response.builder()
      .status(429).reason("Too Many")
      .request(feign.Request.create(feign.Request.HttpMethod.GET, "/x", java.util.Map.of(), null, feign.Util.UTF_8, null))
      .body(body, feign.Util.UTF_8).build();
  var dec = new CommonFeignErrorHandling().problemDecoder(new com.fasterxml.jackson.databind.ObjectMapper());
  var ex = dec.decode("SearchClient#find", resp);
  assertThat(ex).isInstanceOf(RemoteRateLimitedException.class);
}
```

## Интеграционные: WireMock/MockWebServer, негативные сценарии (timeouts, 5xx, «битые» JSON)

Для интеграционных тестов клиентов используйте **WireMock** или **MockWebServer**. Поднимайте сервер на случайном порту, прокидывайте `@DynamicPropertySource` (`clients.<name>.url`), задавайте стабы: 200 с JSON, 429 с `Retry-After`, 500, «битый» JSON, медленный ответ для проверки таймаута. Проверяйте, что клиент **правильно интерпретирует** статусы и бросает нужные исключения.

```java
@Test
void feignTimesOut() {
  var wm = new com.github.tomakehurst.wiremock.WireMockServer(0);
  wm.start();
  wm.stubFor(com.github.tomakehurst.wiremock.client.WireMock.get("/api/v1/items")
      .willReturn(com.github.tomakehurst.wiremock.client.WireMock.aResponse().withFixedDelay(2_000).withStatus(200)));

  try {
    // настроить FeignClient url на wm.baseUrl()
    var page = searchClient.find(new SearchParams(0,10,null,"q")); // должен бросить Timeout
    fail("expected timeout");
  } catch (Exception e) {
    assertThat(e.getMessage()).contains("timed out");
  } finally { wm.stop(); }
}
```

## Контрактные: проверка по OpenAPI/JSON Schema, снапшоты запросов

Контрактные тесты проверяют **соответствие** клиент ↔ спецификация. Простой путь — валидировать тела ответов по **JSON Schema** (например, networknt json-schema-validator), а запросы — через **снапшоты** (assert, что поля/формат соответствуют, без лишних полей). В идеале — генерируйте клиента из OpenAPI и тестируйте «наживую» против WireMock, который **валидирует запрос** по той же схеме.

Стаб WireMock может проверять заголовки/параметры: полезно, чтобы не «забывали» `Idempotency-Key` или `Authorization`. Для multipart — проверяйте `Content-Type` с границей (`boundary`) и поле `filename`.

```java
wm.stubFor(post(urlEqualTo("/api/v1/invoices"))
  .withHeader("Idempotency-Key", matching(".+"))
  .withHeader("Authorization", matching("Bearer .+"))
  .willReturn(aResponse().withStatus(201).withBody("{\"id\":\"inv_1\"}")));
```

## Тесты устойчивости: флаппи сеть, DNS, медленные ответы, лимиты пула

Часть проблем проявляется только под «шумом»: медленный TCP, закрытие соединений, мелкие потери пакетов. Для этого есть **Toxiproxy** (можно через Testcontainers) — он вставляется между клиентом и WireMock и «портит» сеть (latency, bandwidth, timeouts). Проверьте, что ваши **таймауты** адекватны, **Retryer** не разгоняет шторм, а **CB** отсекает лавину.

Пулы соединений: тестируйте «шипы» — сотни одновременных запросов, лимит коннектов (total/route) и поведение очередей (pending acquire). Старайтесь держать отдельные пулы для «тяжёлых» и «лёгких» клиентов (или хотя бы per-client лимиты).

---

# 10. Governance и best practices

## Нейминг клиентов/методов/DTO, изоляция моделей, анти-коррупционный слой

Договоритесь о **нейминге**: `XxxClient` (по источнику/системе), методы — глагол+сущность (`createInvoice`, `getInvoice`, `searchInvoices`). DTO внешних сервисов держите в пакете `integrations.<system>.dto` и **не** протягивайте их в контроллеры/домен. На границе — **мапперы** (MapStruct/ручные), которые переводят во внутренние модели. Это и есть **анти-коррупционный слой**.

Документируйте «что опасно»: какие поля не надёжны, где возможны *breaking changes*, какие enum могут расширяться. Носите knowledge в коде — комментарии/тесты, а не только в Confluence. Интерфейсы клиентов — «живой» контракт, важно держать их максимально лаконичными и стабильными.

Для больших систем заведите **каталог интеграций** (readme/ADR): кто хозяин, контакты, SLO, лимиты, версии API, схемы аутентификации. Это экономит часы при инцидентах и вводе новых инженеров.

## Политика таймаутов/ретраев/CB per класс операций, чек-листы перед продом

Установите **стандарты**:

* `connectTimeout` сотни миллисекунд;
* `readTimeout` ≈ p95 внешнего API + небольшой запас;
* `callTimeout` (жёсткая «крышка»);
* **без ретраев** на мутаторах, **с ретраями** (1–2 попытки, backoff+jitter) на идемпотентных чтениях;
* **CB** включён, окна и пороги описаны;
* **RateLimiter** там, где внешний лимит строгий.

Соберите **чек-лист перед продом**:

* есть ли `RequestInterceptor` с `Authorization`/`Request-Id`?
* настроен ли `ErrorDecoder` и маппинг Problem Details?
* таймауты/пулы выставлены?
* включены метрики/трейсинг?
* маскирование секретов в логах?
* контрактные тесты зелёные?
* «песочница» (staging) прогнана под нагрузкой?

## Управление секретами (Vault/KMS), ротация ключей/сертификатов, безопасные дефолты

Секреты (client-secret, API-key) не храним в `application.yml`. Используйте **Vault**/KMS и инжектите через env/Config Server. Обеспечьте **ротацию**: короткие TTL токенов, план ротации ключей, двойная публикация сертификатных пинов. Периодически запускайте «секьюрити-скан» репозитория на утечки (git-secrets/TruffleHog).

Безопасные дефолты: логгер на `BASIC`, `retryOnConnectionFailure=false` (OkHttp), `decode404=false` (если 404 — это ошибка домена), минимальный набор заголовков, **строгое** CORS/CSRF на стороне вашего API. Для mTLS — храните приватные ключи в защищённом хранилище, не в контейнере.

## Миграции: OkHttp ↔ HC5, вынос блокирующих интеграций из реактивного стека

Иногда приходится мигрировать между HTTP-стеками (политика безопасности, поддержка HTTP/2, TLS-фичи). Делайте слой **абстракции**: фабрика `feign.Client` и per-client бин конфигурации. Тогда замена OkHttp ↔ HC5 сведётся к смене зависимостей/бинов без модификации интерфейсов.

Если часть приложения — реактивная (WebFlux), **не тащите** Feign в event-loop. Вынесите интеграцию в отдельный модуль/сервис на MVC или оффлоадьте на `boundedElastic` с короткими таймаутами и явными границами. Любая «случайная» блокировка в реактивном пути убьёт масштабируемость.

Поддерживайте **канареечные** миграции: фича-флаг/переключатель клиента по окружению/тенанту, метрики сравнения латентности/ошибок, быстрый rollback. Логи/трейсы должны ясно показывать, кем выполнялся вызов (OkHttp/Hc5/JDK) — это упростит расследование.

---

хочешь — соберу **микро-репозиторий OpenFeign** с:

* двумя клиентами (V1/V2, media type versioning),
* `RequestInterceptor`-цепочкой (request-id, Accept-Language, OAuth2 client credentials),
* `ErrorDecoder` с Problem Details,
* OkHttp-клиентом с тюнингом пула/таймаутов/HTTP/2,
* интеграционными тестами (WireMock) на 200/429/500/таймаут и контрактной валидацией JSON Schema.

