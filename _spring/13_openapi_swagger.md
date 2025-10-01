---
layout: page
title: "OpenAPI & Swagger"
permalink: /spring/openapi_swagger
---

# 0. OpenAPI и Swagger

**Что это такое.** OpenAPI — это спецификация (YAML/JSON), описывающая контракт вашего HTTP API: пути, параметры, тела запросов, модели данных, ответы, ошибки, безопасность. Swagger — семейство инструментов вокруг спецификации: Swagger UI (интерактивная HTML-документация), Swagger Codegen/ OpenAPI Generator (генерация клиентов/серверов), валидаторы/линтеры. В современной терминологии “OpenAPI — стандарт, Swagger — инструменты”. Начиная с OpenAPI 3.1, спецификация синхронизирована с JSON Schema (2020-12), что делает модели данных выразительнее и привычнее.

**Где используется.** Повсеместно: от внутренних микросервисов до публичных интеграций. Командам это даёт единый «язык» общения между бэком, фронтом, QA и внешними партнёрами. В DevEx-практиках спецификацию подключают к CI/CD: валидируют, проверяют breaking-changes, генерируют SDK (TypeScript/Java/Kotlin/Go), поднимают мок-серверы для контрактного тестирования. Для Ops она полезна как источник ссылок (externalDocs), описаний версий/сред и требований безопасности.

**Почему это важно.** Контракт снимает неопределённость: «что именно принимает/возвращает API», «какие коды ошибок и в каких случаях», «какой формат дат/денег». Это экономит недели переписок и доработок. Автогенерация UI упрощает онбординг новых разработчиков и внешних потребителей, а кодогенерация снижает количество ручного, потенциально багового клиентского кода (особенно при сложных аутентификациях/скоупах OAuth2).

**Минимальный вид.** Даже крошечная спецификация уже полезна — её можно открыть в Swagger UI или прогнать линтером:

```yaml
openapi: 3.1.0
info:
  title: Sample Shop API
  version: 1.0.0
servers:
  - url: https://api.example.com
paths:
  /health:
    get:
      summary: Health check
      responses:
        '200':
          description: OK
```

---

# 1. Структура спецификации OpenAPI

**Верхний уровень (`info`, `servers`, `tags`, `externalDocs`).** На верхнем уровне вы задаёте идентичность и «ландшафт» сервиса. `info` содержит `title`, `version`, `description` (часто сюда кладут политику версионирования и SLA). `servers` описывает базовые URL окружений: prod/stage/dev. `tags` группируют операции для удобной навигации; у каждого тега может быть `description` и даже `externalDocs` на внутреннюю wiki. Это позволяет людям быстро понять назначение API и найти нужный раздел.

```yaml
openapi: 3.1.0
info:
  title: Shop API
  version: 1.2.0
  description: Каталог, корзина, заказы. Версионирование через заголовок X-API-Version.
servers:
  - url: https://api.example.com
    description: Production
tags:
  - name: Catalog
    description: Работа с товарами
externalDocs:
  description: Архитектурная wiki
  url: https://wiki.example.com/apis/shop
```

**`paths` и операции (summary/description/operationId/parameters/requestBody/responses).** Раздел `paths` — сердце спецификации. У каждого пути есть HTTP-методы с уникальным `operationId` (важно для генерации клиентов), кратким `summary` и развёрнутым `description`. Параметры (`in: path|query|header|cookie`) объявляются со схемами/валидацией, тела запросов описываются через `requestBody`, а ответы — через `responses` с явными медиа-типами и схемами. В реальной жизни полезно стандартизовать пагинацию/сортировку и описывать их один раз в `components.parameters`.

```yaml
paths:
  /products:
    get:
      tags: [Catalog]
      summary: Поиск товаров
      operationId: searchProducts
      parameters:
        - $ref: '#/components/parameters/Page'
        - $ref: '#/components/parameters/Size'
        - name: q
          in: query
          schema: { type: string, minLength: 2 }
      responses:
        '200':
          description: Страница товаров
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Page_Product' }
```

**`components` (schemas/parameters/headers/responses/requestBodies/securitySchemes/examples/links/callbacks).** Всё переиспользуемое — сюда. В `schemas` живут модели (DTO), в `parameters` — стандартные параметры (например, `Page`, `Size`, `Sort`), в `responses` — типовые ответы (например, `Problem` по RFC-7807), в `securitySchemes` — схемы авторизации (Basic, Bearer JWT, OAuth2, API Key, mTLS). Это избавляет от дублирования и повышает единообразие across сервисы.

```yaml
components:
  parameters:
    Page: { name: page, in: query, schema: { type: integer, minimum: 0 }, description: Номер страницы }
    Size: { name: size, in: query, schema: { type: integer, minimum: 1, maximum: 200, default: 20 }, description: Размер страницы }
  schemas:
    Money:
      type: object
      required: [amount, currency]
      properties:
        amount:   { type: number, multipleOf: 0.01, minimum: 0 }
        currency: { type: string, pattern: '^[A-Z]{3}$', example: 'USD' }
    Product:
      type: object
      required: [id, name, price]
      properties:
        id:   { type: string, format: uuid }
        name: { type: string, minLength: 1, maxLength: 200 }
        price:{ $ref: '#/components/schemas/Money' }
    Page_Product:
      type: object
      properties:
        content: { type: array, items: { $ref: '#/components/schemas/Product' } }
        page:    { type: integer, minimum: 0 }
        size:    { type: integer, minimum: 1 }
        total:   { type: integer, minimum: 0 }
```

**Модульность и `$ref`.** Крупные спецификации стоит дробить: схемы в `schemas/*.yaml`, блоки путей в `paths/*.yaml`. Ссылки на внешние файлы делаются через `$ref: 'schemas/product.yaml#/Product'`. Часто делают «мастер-файл» с верхним уровнем и включениями путей, а каждую предметную область (каталог, заказы, пользователи) ведут в отдельном файле. Это упрощает ревью и уменьшает конфликты при работе параллельных команд.

```yaml
paths:
  /products: { $ref: 'paths/catalog.yaml#/products' }
components:
  schemas:
    Product: { $ref: 'schemas/product.yaml#/Product' }
```

**JSON Schema в OAS 3.1.** Версия 3.1 выровнена с JSON Schema 2020-12, поэтому доступны `if/then/else`, `unevaluatedProperties`, строгие типы, `const`, `enum` с `title/description`. Это позволяет выражать сложные инварианты прямо в контракте. Для дат используйте `format: date`/`date-time` (UTC/ISO-8601), для денег — `number` + `multipleOf: 0.01` (а в коде — `BigDecimal`). Для идентификаторов — `format: uuid`, для валют — `pattern` `^[A-Z]{3}$`.

```yaml
OrderItem:
  type: object
  properties:
    type: { enum: [physical, digital] }
    downloadUrl: { type: string, format: uri }
    shipmentId:  { type: string }
  if: { properties: { type: { const: digital } } }
  then: { required: [downloadUrl] }
  else: { required: [shipmentId] }
```

---

# 2. Моделирование данных

**DTO vs Entity.** JPA-сущности — внутренний слой: ленивые связи, технические поля (`@Version`, аудиты), инварианты, которые клиенту знать не нужно. Экспортировать их «как есть» — значит протекание деталей ORM и повышенную хрупкость API (любая внутреняя рефакторинга ломает внешний контракт). Поэтому наружу — **DTO** с чёткими полями, валидаторами и стабильными именами; внутри — маппинг (MapStruct/ручной) между Entity и DTO.

```java
public record ProductDto(
    @Schema(description="UUID товара") @NotNull UUID id,
    @Schema(minLength=1, maxLength=200) @NotBlank String name,
    @Valid MoneyDto price,
    @Schema(description="Время создания", format="date-time") @NotNull OffsetDateTime createdAt
) {}

public record MoneyDto(
    @Schema(description="Сумма, 2 знака") @NotNull @Digits(integer=18, fraction=2) BigDecimal amount,
    @Schema(description="ISO 4217") @Pattern(regexp="^[A-Z]{3}$") String currency
) {}
```

**Наследование и полиморфизм (`oneOf`/`allOf`/`anyOf`, discriminator).** Для вариантов ответа/запроса (разные виды платежей/доставок) используйте `oneOf` с `discriminator`. Это даёт клиентам понятный полиморфный контракт и корректную генерацию моделей на их стороне. В Java удобно сделать поле `type` и маппить его на конкретные DTO; в Kotlin — применять `sealed interface` (springdoc это корректно разворачивает в `oneOf`).

```yaml
Payment:
  oneOf:
    - $ref: '#/components/schemas/CardPayment'
    - $ref: '#/components/schemas/CashPayment'
  discriminator:
    propertyName: type
    mapping:
      card: '#/components/schemas/CardPayment'
      cash: '#/components/schemas/CashPayment'
```

```java
sealed interface PaymentDto { String type(); }
record CardPaymentDto(String type, String last4, String holder){ }
record CashPaymentDto(String type, String receiptNo){ }
```

**Коллекции, дженерики и обёртки.** В OAS нет дженериков, поэтому универсальные обёртки описывают конкретными типами: `Page_Product`, `Envelope_Product`. Часто используют «базовую» схему + `allOf` для специализации, или просто руками создают по одному типу на сущность (это прозрачнее для клиентов). В Java обёртки выражаются как `record PageResponse<T>`; в спецификации — конкретизированные варианты.

```yaml
Envelope_Product:
  type: object
  properties:
    data: { $ref: '#/components/schemas/Product' }
    time: { type: string, format: date-time }
    requestId: { type: string }
```

**Деньги: BigDecimal/валюта/точность.** Денежные суммы хранятся как `BigDecimal` в Java, в OAS — как `number` с `multipleOf: 0.01` (или 0.0001 для 4 знаков). Валюта — ISO 4217, три заглавные буквы. Никогда не используйте `double` в DTO/сущностях для денежных расчётов. При сериализации Jackson убедитесь, что `BigDecimal` не превращается в научную нотацию.

```yaml
Money:
  type: object
  required: [amount, currency]
  properties:
    amount: { type: number, multipleOf: 0.01, minimum: 0, example: 1234.56 }
    currency: { type: string, pattern: '^[A-Z]{3}$', example: 'EUR' }
```

**Даты/время: Instant/OffsetDateTime/LocalDate.** Во внешнем API используйте `OffsetDateTime` (ISO-8601 с зоной) для моментов времени и `LocalDate` для дат без времени. В Spring/Jackson выключите таймстемпы и включите ISO-формат — это автоматически подхватит springdoc.

```yaml
properties:
  createdAt: { type: string, format: date-time, example: '2025-09-25T14:25:00Z' }
  birthday:  { type: string, format: date }
```

**Пагинация/фильтрация/сортировка.** Согласуйте единый формат: `page`, `size`, `sort` (`field,asc|desc`), а фильтры — через явные параметры (`status`, `from`, `to`) или массив `filter[status]=...`. Описание параметров вынесите в `components.parameters` и подключайте `$ref` — документация будет единообразной.

**Контракт ошибок (RFC-7807 / Problem Details).** В Spring Boot 3 есть `ProblemDetail`. В спецификации опишите типовой ответ `application/problem+json` и подключайте его ко всем операциям. Это дисциплинирует форматы ошибок и упрощает клиентам обработку.

```java
@ControllerAdvice
class ApiErrors {
  @ExceptionHandler(EntityNotFoundException.class)
  ResponseEntity<ProblemDetail> notFound(EntityNotFoundException ex) {
    var pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    pd.setTitle("Entity not found");
    pd.setProperty("errorCode", "ENTITY_NOT_FOUND");
    return ResponseEntity.status(pd.getStatus()).body(pd);
  }
}
```

```yaml
components:
  schemas:
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type:   { type: string, format: uri }
        title:  { type: string }
        status: { type: integer, minimum: 100, maximum: 599 }
        detail: { type: string }
        instance: { type: string, format: uri }
```

---

# 3. Документация в Spring Boot (MVC и WebFlux)

**Библиотеки и подключение.** Для Spring Boot 3.x актуален `springdoc-openapi` 2.x. Есть два стартера: `springdoc-openapi-starter-webmvc-ui` (для MVC) и `springdoc-openapi-starter-webflux-ui` (для WebFlux). Подключаем один из них — и автоматически получаем эндпоинты `/v3/api-docs(.yaml)` и `/swagger-ui.html`.

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web") // или -webflux
  implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
  implementation("jakarta.validation:jakarta.validation-api:3.0.2")
}
```

**Эндпоинты и базовая настройка.** Springdoc сканирует контроллеры, модели, аннотации (Jakarta Validation) и строит спецификацию на лету. По умолчанию:
— JSON: `/v3/api-docs`
— YAML: `/v3/api-docs.yaml`
— UI: `/swagger-ui.html`
Параметрами можно включить/отключить UI, сменить путь, разбить на группы.

```java
@Configuration
class OpenApiGroups {
  @Bean GroupedOpenApi publicApi() {
    return GroupedOpenApi.builder()
      .group("public")
      .pathsToMatch("/api/v1/**")
      .build();
  }
  @Bean GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
      .group("admin")
      .packagesToScan("com.example.admin")
      .build();
  }
}
```

**Аннотации: `@Operation`, `@Parameter`, `@Schema`, `@Tag`, `@Hidden`.** Аннотации помогают уточнить то, чего фреймворк не догадает автоматически: `summary`, `description`, `operationId`, обязательность параметров, примеры (`@ExampleObject`). `@Tag` группирует контроллер, `@Hidden` скрывает внутренние операции.

```java
@RestController
@RequestMapping("/api/v1/products")
@Tag(name="Catalog", description="Работа с товарами")
class ProductController {
  private final ProductService service;
  ProductController(ProductService service){ this.service = service; }

  @GetMapping
  @Operation(summary="Поиск товаров", operationId="searchProducts")
  public PageResponse<ProductDto> search(
      @Parameter(description="Поисковая строка", example="phone") @RequestParam(required=false) String q,
      @RequestParam(defaultValue="0") int page,
      @RequestParam(defaultValue="20") int size) {
    return service.search(q, page, size);
  }

  @PostMapping
  @Operation(summary="Создать товар", operationId="createProduct")
  public ProductDto create(@Valid @RequestBody CreateProductRequest req){
    return service.create(req);
  }
}
```

**Генерация схем из валидаторов.** Аннотации Jakarta Validation (`@NotNull`, `@Size`, `@Pattern`, `@Digits`, `@Email`) конвертируются в ограничения JSON Schema. Добавляйте их на DTO — и в UI сразу появятся «минимальная длина», «паттерн», «обязательность». Это бесплатно улучшает качество контракта и UX потребителей.

**Jackson: форматы, nullability, примеры.** Включите ISO-даты и отключите таймстемпы, чтобы `OffsetDateTime` сериализовался как `2025-09-25T12:00:00Z`. Для примеров можно использовать `@Schema(example="...")` на полях.

```yaml
spring:
  jackson:
    serialization:
      WRITE_DATES_AS_TIMESTAMPS: false
    default-property-inclusion: non_null
```

**Кастомизация: `OpenApiCustomiser`/`OperationCustomizer`/`PropertyCustomizer`.**
— `OpenApiCustomiser` меняет весь документ (добавить `servers`, глобальные `responses`).
— `OperationCustomizer` добавляет заголовки ко всем операциям (например, `X-Request-Id`).
— `PropertyCustomizer` настраивает свойства схем (например, сделать все `UUID` с `format: uuid`).

```java
@Bean
OperationCustomizer addRequestIdHeader() {
  return (operation, handler) -> {
    operation.addParametersItem(new Parameter()
      .in("header").name("X-Request-Id").required(false)
      .schema(new StringSchema()).description("Корреляция"));
    return operation;
  };
}
```

**Глобальные ответы/хедеры и ProblemDetail.** Хорошая практика — задокументировать типовые 400/404/409/500 с единой схемой `Problem`. Это можно сделать кастомайзером или повторно ссылаться в аннотациях `@ApiResponse`.

```java
@Bean
OpenApiCustomiser addProblemSchema() {
  return openApi -> {
    var pb = new Schema<>().type("object")
      .addProperties("type", new StringSchema())
      .addProperties("title", new StringSchema())
      .addProperties("status", new IntegerSchema())
      .addProperties("detail", new StringSchema())
      .addProperties("instance", new StringSchema());
    openApi.getComponents().addSchemas("Problem", pb);
  };
}
```

**WebFlux/Reactive: Mono/Flux/SSE.** Springdoc корректно документирует `Mono<T>` как обычный объект, `Flux<T>` — как массив. Для SSE укажите `produces = text/event-stream` и схему события (`ServerSentEvent<T>` можно отразить как `T`, а метаданные — словами в `description`). Реактивные timeouts/кодеки тоже стоит упомянуть в `externalDocs`, если это часть контракта.

```java
@GetMapping(value="/stream", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
@Operation(summary="События каталога (SSE)")
public Flux<ServerSentEvent<ProductDto>> stream(){ ... }
```

**Kotlin-нюансы.** `data class` и nullability в Kotlin автоматически транслируются в `nullable` в OAS. Значения по умолчанию становятся примерами. Для `sealed`-иерархий springdoc делает `oneOf` — удобно для полиморфных payload’ов. Не забудьте `kotlin-reflect` и модуль Jackson Kotlin в зависимостях.

---

# 4. Безопасность и авторизация

**`securitySchemes`: Basic, Bearer (JWT), OAuth2, API Key, mTLS.** В `components.securitySchemes` объявите схемы, а затем примените их глобально (`security` на корне) или точечно на операции. Для JWT укажите `type: http` + `scheme: bearer`, для OAuth2 — нужные флоуы (authorizationCode/clientCredentials), для API-ключей — где искать ключ (`in: header|query|cookie`). Для mTLS в OAS 3.1 есть `type: mutualTLS`.

```yaml
components:
  securitySchemes:
    BearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
    OAuth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            catalog:read: Чтение каталога
            orders:write: Создание заказов
    ApiKeyAuth: { type: apiKey, in: header, name: X-API-Key }
    mTLS: { type: mutualTLS }
security:
  - BearerAuth: []
```

**Scopes и RBAC.** Для OAuth2 перечислите скоупы и кратко опишите их назначение. В описаниях операций упоминайте, какие скоупы требуются (это видно в UI и помогает интеграторам). Если у вас ролевые модели (RBAC), можно описать их словами в `description` и/или дублировать в `externalDocs` с матрицей «роль → доступ к операциям».

**mTLS/Mutual TLS.** При mTLS клиент предъявляет сертификат, а сервер его валидирует: корневой CA, CN/OU-требования, сроки действия. В спецификации зафиксируйте `securitySchemes.mTLS` и словами опишите процесс получения/ротации сертификатов, требования к TLS-версиям/шифрам. Это особенно важно для B2B-интеграций, где mTLS — основной фактор доверия.

**CORS/CSRF для браузерных клиентов.** Спецификация может и должна объяснять фронту, какие Origins/методы/заголовки разрешены (CORS), и как работать с CSRF (если применимо): где брать токен, в какой заголовок класть, какие эндпоинты освобождены. В Spring Security это настраивается кодом, а в OpenAPI — описывается словами и ссылками (`externalDocs`) — иначе интеграторам придётся «угадывать».

**Маскирование чувствительных примеров.** В `examples` не используйте реальные токены, персональные данные, номера карт. Маскируйте или генерируйте фиктивные значения. Если вы публикуете спеки внешне, включите ревью на конфиденциальность: автоматические проверки в CI легко ловят «секретоподобные» строки (JWT-паттерны, base64-похожие).

```java
@Configuration
class SecurityOpenApi {
  @Bean
  OpenApiCustomiser securityReq() {
    return openApi -> {
      var req = new io.swagger.v3.oas.models.security.SecurityRequirement().addList("BearerAuth");
      openApi.addSecurityItem(req);
    };
  }
}
```

---

## Завершающий практикум: минимальный «скелет» проекта

**Gradle (Kotlin DSL):**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web") // или -webflux
  implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
  implementation("jakarta.validation:jakarta.validation-api:3.0.2")
  implementation("org.springframework.boot:spring-boot-starter-security")
}
```

**DTO/валидация + контроллер:**

```java
public record CreateProductRequest(
  @NotBlank String name,
  @NotNull @Digits(integer=18, fraction=2) BigDecimal amount,
  @Pattern(regexp="^[A-Z]{3}$") String currency) {}

@RestController
@RequestMapping("/api/v1/products")
@Tag(name="Catalog")
class ProductController {
  @PostMapping
  @Operation(summary="Создать товар")
  public ProductDto create(@Valid @RequestBody CreateProductRequest req){ ... }

  @GetMapping("/{id}")
  @Operation(summary="Получить товар")
  public ProductDto get(@PathVariable UUID id){ ... }
}
```

**Группы/кастомизация/безопасность в OpenAPI:**

```java
@Configuration
class OpenApiConfig {
  @Bean GroupedOpenApi publicApi() {
    return GroupedOpenApi.builder().group("public").pathsToMatch("/api/v1/**").build();
  }
  @Bean OpenApiCustomiser addServers() {
    return o -> o.addServersItem(new Server().url("https://api.example.com"));
  }
  @Bean OpenApiCustomiser securityReq() {
    return o -> o.addSecurityItem(new SecurityRequirement().addList("BearerAuth"));
  }
}
```

---

# Итог

OpenAPI превращает ваш HTTP API в «живой контракт»: машинно-читаемый документ для CI/CD и удобный портал для людей. Правильная структура (верхний уровень/paths/components), модульность через `$ref`, аккуратное моделирование DTO (деньги/даты/полиморфизм), единый формат ошибок (RFC-7807), и тщательно описанная безопасность (JWT/OAuth2/mTLS) — всё это делает интеграции предсказуемыми, а релизы безопасными. На стороне Spring Boot всё просто: **springdoc-openapi** + аннотации на контроллерах/DTO, кастомайзеры для нюансов, и вы уже публикуете актуальную документацию, из которой можно генерировать клиентов и проверять совместимость на каждом коммите.
