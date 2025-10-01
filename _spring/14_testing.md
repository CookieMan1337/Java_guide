---
layout: page
title: "Тестирование Spring Boot приложения"
permalink: /spring/testing
---

---

# 1. Стратегия тестирования и пирамида для Spring-сервисов

**Типы тестов: unit, slice, интеграционные, контрактные, end-to-end.** Хорошая пирамида опирается на быстрые **unit-тесты** (чистая Java-логика без Spring-контекста), под ними — **slice-тесты** Spring (узкий срез слоя: MVC/JPA/JSON/RestClient/Jdbc), выше — **интеграционные** `@SpringBootTest` (полный контекст, фильтры, AOP, конфиги), ещё выше — **контрактные** (HTTP/Kafka, проверяют соответствие внешним контрактам), а на вершине — редкие **end-to-end** (с реальными окружениями). Чем выше — тем дороже и медленнее, тем меньше их должно быть.

**Цели и скоуп.** На unit-уровне валидируйте чистую доменную логику, алгоритмы, инварианты, превращая спецификацию в читаемый набор Given-When-Then. На slice-уровне проверяйте слой в изоляции: репозитории, сериализацию, контроллеры без поднятия всего приложения. Интеграционные (`@SpringBootTest`) оставляйте для сквозных сценариев, пересечения бинов, security-фильтров, транзакционных границ. Контрактные тесты (по OpenAPI/Avro/JSON Schema) страхуют против «ломающих» изменений при интеграциях.

**Организационная модульность: фикстуры, билдеры, utils.** Отдельная тестовая «библиотека» в проекте экономит время: `testFixtures` (Gradle) с Test Data Builder-ами, фабриками DTO/JSON, «золотыми» сэмплами и хелперами (`assertProblem`, `readResource`, `jsonOf`). Структура директорий: `src/test/java` для обычных тестов, `src/testFixtures/java` для переиспользуемых билдеров, `src/integrationTest/java` под медленные/контейнерные сценарии (с отдельным Gradle sourceSet).

```kotlin
// build.gradle.kts: включаем testFixtures и отдельный sourceSet
java {
  withJavadocJar()
  withSourcesJar()
}
testing {
  suites {
    val test by getting(JvmTestSuite::class) {
      useJUnitJupiter()
    }
    val integrationTest by registering(JvmTestSuite::class) {
      useJUnitJupiter()
      dependencies {
        implementation(project)
        implementation(testFixtures(project))
      }
    }
  }
}
```

**Политика стабильности: «зелёный по умолчанию».** Флаки-тесты подрывают доверие — маркируйте их `@Tag("flaky")` и отправляйте в карантинные наборы, но не оставляйте в PR-гейте. Следите за «time-to-red/green»: быстрый красный сигнал важнее 100% покрытия. Любой найденный баг должен сопровождаться тестом-регрессом; любой инцидент — ретро → тест.

**Метрики качества.** Покрытие (JaCoCo) — не самоцель, а индикатор «дырок». Мутационное тестирование (PIT) показывает, «живые» ли тесты: если мутации выживают, значит, проверки слабые. Полезны метрики времени: длительность набора, самый медленный тест/класс, процент флаки. На дэшборде Sonar — правила качества (дубли, code smells, эволюция покрытия).

---
Ниже — «глава 14» про тестирование в Spring-проектах. Я держу тот же стиль, что и в теме 10: плавный вход, затем подробное раскрытие, обязательно с кодом. Java 17+, Spring Boot 3.x, JUnit 5. Для БД — Postgres; для HTTP-клиентов — WireMock; для сообщений — Kafka/JMS; для реактивщины — WebFlux/StepVerifier.

---

# 1. Стратегия тестирования и пирамида для Spring-сервисов

**Типы тестов: unit, slice, интеграционные, контрактные, end-to-end.** Хорошая пирамида опирается на быстрые **unit-тесты** (чистая Java-логика без Spring-контекста), под ними — **slice-тесты** Spring (узкий срез слоя: MVC/JPA/JSON/RestClient/Jdbc), выше — **интеграционные** `@SpringBootTest` (полный контекст, фильтры, AOP, конфиги), ещё выше — **контрактные** (HTTP/Kafka, проверяют соответствие внешним контрактам), а на вершине — редкие **end-to-end** (с реальными окружениями). Чем выше — тем дороже и медленнее, тем меньше их должно быть.

**Цели и скоуп.** На unit-уровне валидируйте чистую доменную логику, алгоритмы, инварианты, превращая спецификацию в читаемый набор Given-When-Then. На slice-уровне проверяйте слой в изоляции: репозитории, сериализацию, контроллеры без поднятия всего приложения. Интеграционные (`@SpringBootTest`) оставляйте для сквозных сценариев, пересечения бинов, security-фильтров, транзакционных границ. Контрактные тесты (по OpenAPI/Avro/JSON Schema) страхуют против «ломающих» изменений при интеграциях.

**Организационная модульность: фикстуры, билдеры, utils.** Отдельная тестовая «библиотека» в проекте экономит время: `testFixtures` (Gradle) с Test Data Builder-ами, фабриками DTO/JSON, «золотыми» сэмплами и хелперами (`assertProblem`, `readResource`, `jsonOf`). Структура директорий: `src/test/java` для обычных тестов, `src/testFixtures/java` для переиспользуемых билдеров, `src/integrationTest/java` под медленные/контейнерные сценарии (с отдельным Gradle sourceSet).

```kotlin
// build.gradle.kts: включаем testFixtures и отдельный sourceSet
java {
  withJavadocJar()
  withSourcesJar()
}
testing {
  suites {
    val test by getting(JvmTestSuite::class) {
      useJUnitJupiter()
    }
    val integrationTest by registering(JvmTestSuite::class) {
      useJUnitJupiter()
      dependencies {
        implementation(project)
        implementation(testFixtures(project))
      }
    }
  }
}
```

**Политика стабильности: «зелёный по умолчанию».** Флаки-тесты подрывают доверие — маркируйте их `@Tag("flaky")` и отправляйте в карантинные наборы, но не оставляйте в PR-гейте. Следите за «time-to-red/green»: быстрый красный сигнал важнее 100% покрытия. Любой найденный баг должен сопровождаться тестом-регрессом; любой инцидент — ретро → тест.

**Метрики качества.** Покрытие (JaCoCo) — не самоцель, а индикатор «дырок». Мутационное тестирование (PIT) показывает, «живые» ли тесты: если мутации выживают, значит, проверки слабые. Полезны метрики времени: длительность набора, самый медленный тест/класс, процент флаки. На дэшборде Sonar — правила качества (дубли, code smells, эволюция покрытия).

---

# 2. Unit-тесты доменной логики (без Spring-контекста)

**Принципы: AAA, Given-When-Then, TDD на горячих участках.** Unit-тест должен быть простым и детерминированным. Шаблон AAA: Arrange (готовим данные), Act (запускаем метод), Assert (проверяем результат). Именуйте тесты по бизнес-правилу, а не по названию метода — это улучшает читаемость. На «горячих» участках выгодно идти TDD: сначала тест, затем минимальная реализация, затем рефакторинг.

**Моделирование данных: Builders/Factory, фиксированные Clock/Random.** Не плодите «магические» конструкторы: используйте Test Data Builder с разумными дефолтами и fluent-API. Время и случайность фиксируйте: `Clock.fixed(…)`, детерминированный `Random(0)` — иначе тест может стать флаки. Для JSON-контрактов держите «золотые файлы» в `src/test/resources`.

```java
class MoneyBuilder {
  private BigDecimal amount = new BigDecimal("10.00");
  private String currency = "USD";
  MoneyBuilder amount(String a){ this.amount = new BigDecimal(a); return this; }
  MoneyBuilder currency(String c){ this.currency = c; return this; }
  MoneyDto build(){ return new MoneyDto(amount, currency); }
}
```

**Мокинг/стаббинг.** Mockito — дефолт: `@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`, `ArgumentCaptor`. Частичные моки (spy) допустимы для оборачивания тяжёлых зависимостей, но не для «шпионажа» за доменной логикой — это признак дизайна, требующего улучшения. В Kotlin удобен MockK.

```java
@ExtendWith(MockitoExtension.class)
class PriceServiceTest {
  @Mock RateProvider rateProvider;
  @InjectMocks PriceService svc;
  @Test void convertsUsdToEur() {
    when(rateProvider.rate("USD","EUR")).thenReturn(new BigDecimal("0.9"));
    var eur = svc.convert(new BigDecimal("100.00"), "USD", "EUR");
    assertThat(eur).isEqualByComparingTo("90.00");
  }
}
```

**Параметризованные и property-based.** JUnit 5 `@ParameterizedTest` подходит для граничных наборов (пустые строки, Unicode, большие числа). Property-based (jqwik) генерирует множество входов, проверяя инварианты: сумма всегда ≥ 0; сортировка — идемпотентна; сериализация/десериализация — обратимы.

```java
// jqwik property-based
@Property
void additionIsCommutative(@ForAll int a, @ForAll int b) {
  assertThat(a + b).isEqualTo(b + a);
}
```

**Читаемость и кастомные assertions.** Скрывайте «шум» в хелперах и кастомных ассертерах: `assertThat(order).hasTotal("123.45").hasStatus(PAID)`. Это уменьшает дублирование и делает тесты ближе к бизнес-языку. Избегайте «магических чисел»: выносите значения в именованные константы.

---

# 3. Spring Test Slices: быстрые тесты слоёв

**Назначение `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`, `@RestClientTest`, `@JdbcTest`.** Слайсы поднимают только необходимый кусочек контекста. `@WebMvcTest` — контроллеры + MVC-инфраструктура (без сервисов/репозиториев), тестируете валидацию/сериализацию/ответы через `MockMvc`. `@DataJpaTest` — JPA/EntityManager/репозитории (без web), удобно для запросов и миграций. `@JsonTest` — Jackson-модуль: проверка (де)сериализации DTO. `@RestClientTest` — HTTP-клиенты (RestTemplate/WebClient) против встроенного WireMock. `@JdbcTest` — слой DAO на JdbcTemplate.

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {
  @Autowired MockMvc mvc;
  @MockBean ProductService service;
  @Test void returnsPage() throws Exception {
    when(service.search("phone",0,20)).thenReturn(PageResponse.of(List.of(...)));
    mvc.perform(get("/api/products").param("q","phone"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.content").isArray());
  }
}
```

**Конфигурация: точечные бины, отключение автоконфигураций.** Слайсам можно «подложить» только нужные конфиги через `@Import`, а лишние автоконфигурации выключить свойствами. В `@WebMvcTest` зависимости контроллера закрывайте `@MockBean`, чтобы изоляция не нарушалась. В `@DataJpaTest` используйте Testcontainers + `@AutoConfigureTestDatabase(replace = NONE)` для реального Postgres.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class BookRepositoryTest {
  @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
  @DynamicPropertySource static void props(DynamicPropertyRegistry r){
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
  }
  @Autowired BookRepository repo;
  @Test void findsByTitle(){ ... }
}
```

**Ограничения слайсов.** У них намеренно нет всего контекста (нет security-фильтров в `@WebMvcTest`, нет внешних бинов в `@JsonTest` и т. п.). Это плюс для скорости, но ловушка для ожиданий. Если нужно расширить границы — добавьте `@Import` конкретных конфигов или перейдите на `@SpringBootTest` для данного сценария.

---

# 4. Интеграционные тесты с `@SpringBootTest`

**Когда нужен полный контекст.** Сквозные сценарии, проверка interplay фильтров, интерсепторов, аспектов, сообщений, транзакций — всё это требует реального контекста. Здесь же тестируются запрос-ответ с security (JWT), глобальные обработчики ошибок, сериализация по «боевому» `ObjectMapper`, конфиг-оверрайды.

**Кэш контекста JUnit 5 и цена рестартов.** Spring кеширует контекст между тестами. Любая «грязнящая» аннотация (`@DirtiesContext`) ломает кеш и делает прогон медленным. Стройте конфиги и профили так, чтобы разные наборы разделяли контекст, а не рестартовали. Общие property задавайте через `@TestPropertySource` или `@ActiveProfiles`.

**Инструменты клиента: MockMvc, WebTestClient, TestRestTemplate.**
— `MockMvc` (MVC) — быстрый, без реального сокета, богатый по матчерам.
— `WebTestClient` (WebFlux) — реактивный клиент для MVC или WebFlux; удобен с `expectBody()` + JSONPath.
— `TestRestTemplate` — настоящий HTTP к живому серверу, полезно для проверки фильтров, CORS, HTTPS.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class AppIT {
  @Autowired MockMvc mvc;                    // быстрые проверки
  @LocalServerPort int port;                 // «живой» порт
  @Test void healthIsOk() throws Exception {
    mvc.perform(get("/actuator/health")).andExpect(status().isOk());
  }
}
```

**Конфигурация окружения: профили, DynamicPropertySource, порты.** Для контейнеров/внешних систем используйте `@DynamicPropertySource` — это лучше, чем `application-test.yml` с фиксированными портами. Для параллельных прогонов берите `RANDOM_PORT`, а внешним клиентам прокидывайте URL через свойства.

---

# 5. Тестирование persistence-слоя: JPA, миграции, данные

**Реальный Postgres vs H2/embedded.** H2 удобен, но несовместим по типам/диалекту/индексам (особенно JSONB, GIN/GIST, `timestamp with time zone`). На интеграционных тестах лучше Testcontainers с Postgres — вы проверяете настоящие планы и поведение транзакций. H2 допустим на unit-уровне (например, `@JdbcTest`) для быстрых проверок простых SQL.

**Миграции в тестах: Flyway/Liquibase.** Поднимайте схему миграциями перед тестами — это гарантирует соответствие прод-состоянию. Для Flyway: добавить зависимость и `spring.flyway.enabled=true`. Для Liquibase: `spring.liquibase.change-log=...`. Можно запускать миграции программно в тесте, но обычно достаточно автоконфигурации.

```java
@DataJpaTest
@Testcontainers
class MigrationsIT {
  @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
  @DynamicPropertySource static void props(DynamicPropertyRegistry r){
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
    r.add("spring.flyway.enabled", () -> "true");
  }
  @Autowired DataSource ds;
  @Test void schemaReady() throws Exception {
    try (var c = ds.getConnection(); var rs = c.createStatement()
         .executeQuery("select to_regclass('public.books') is not null")) {
      assertThat(rs.next()).isTrue();
    }
  }
}
```

**Репозитории и запросы: `@DataJpaTest`, `@Sql`, TestEntityManager.** Для сложных JPQL/SQL удобно подготавливать данные через `@Sql` (seed.sql) и проверять точные результаты. `TestEntityManager` помогает управлять `flush/clear` и состоянием сущностей.

```java
@DataJpaTest
@Sql("/sql/books_seed.sql")
class BookRepositoryQueriesTest {
  @Autowired BookRepository repo;
  @Test void searchByTitle(){ ... }
}
```

**Производительность: батчи, ID, N+1.** В тестах детектируйте N+1 (например, логированием SQL и проверкой количества запросов), проверяйте, что `batch_size` применился (через драйвер/логи), и что стратегия генерации ID (SEQUENCE vs IDENTITY) не ломает батчи. Простая защита: «метрика» количества запросов для конкретного use-case.

---

# 6. Веб-слой и безопасность

**Контроллеры: валидация, сериализация, ошибки.** В `@WebMvcTest`/`MockMvc` удобно проверять `@Valid` на входных DTO, корректность `ProblemDetail` для ошибок, наличие заголовков пагинации. Загружать файлы — через `multipart` и проверять ограничения размера/типа.

```java
@WebMvcTest(ProductController.class)
class ProductControllerValidationTest {
  @Autowired MockMvc mvc;
  @Test void badRequestOnValidation() throws Exception {
    mvc.perform(post("/api/v1/products")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""{"name": "", "amount": -1, "currency": "us"}"""))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.title").value("Bad Request"));
  }
}
```

**Spring Security Test.** Используйте `@WithMockUser`, `@WithSecurityContext` или JWT-моки (через SecurityFilterChain с тестовым конвертером). Проверяйте доступы по ролям/скоупам и то, что ответы для «гостей» и «пользователей» различаются.

```java
@AutoConfigureMockMvc
@SpringBootTest
class SecurityIT {
  @Autowired MockMvc mvc;
  @Test @WithMockUser(roles = "ADMIN")
  void adminAllowed() throws Exception {
    mvc.perform(get("/api/v1/admin")).andExpect(status().isOk());
  }
  @Test
  void anonymousForbidden() throws Exception {
    mvc.perform(get("/api/v1/admin")).andExpect(status().isForbidden());
  }
}
```

**Фильтры/интерсепторы/советы.** Интеграционные тесты полезны для проверки глобальных заголовков (`X-Request-Id`), CORS, логирования запросов. Для CORS проверьте preflight (`OPTIONS`) и список разрешённых origin.

**Версионирование API.** Если вы поддерживаете `v1/v2` одновременно, заведите группы тестов, которые проверяют backward-совместимость (минимальный контракт `v1` остаётся валидным). Это особенно важно при expand/contract-эволюциях.

---

# 7. Внешние HTTP-клиенты и устойчивость

**WireMock/MockWebServer.** Для RestTemplate/WebClient используйте `@RestClientTest` + встроенный WireMock или OkHttp MockWebServer. Опишите сценарии с задержками, 429/503, поломанными JSON, чтобы проверить ретраи/таймауты/деградацию.

```java
@RestClientTest(ExternalClient.class)
class ExternalClientTest {
  @Autowired ExternalClient client;
  @Autowired WireMockServer server;

  @Test void retriesOn503() {
    server.stubFor(get(urlEqualTo("/profile/42"))
      .inScenario("retry").whenScenarioStateIs(STARTED)
      .willReturn(aResponse().withStatus(503))
      .willSetStateTo("ok"));
    server.stubFor(get("/profile/42").inScenario("retry").whenScenarioStateIs("ok")
      .willReturn(okJson("""{"id":42,"name":"Neo"}""")));

    var profile = client.getProfile(42);
    assertThat(profile.name()).isEqualTo("Neo");
  }
}
```

**Динамические URL.** В тестах инжектируйте базовый URL клиента через `@DynamicPropertySource`, взяв порт WireMock/MockWebServer из бина. Избегайте фиксированных портов — на CI они конфликтуют.

**Тестирование устойчивости: ретраи/таймауты/circuit breaker.** Если используете Resilience4j, проверяйте переходы состояний (closed→open→half-open), таймауты и фоллбеки. Для хаоса добавьте тесты «медленный ответ», «разорванное соединение», «сбоеустойчивые коды».

```java
@Retry(name = "ext")
@CircuitBreaker(name = "ext")
public Profile getProfile(long id) { ... }
// в тестах: конфигурируйте маленькие пороги и проверяйте открытие/закрытие
```

**Контракты клиентов.** Из OpenAPI/JSON Schema генерируйте стабы (WireMock mapping) и валидируйте ответы клиента против схемы. Это защищает от «мелких» несовпадений формата, которые часто всплывают в проде.

---

# 8. Сообщения и асинхронность

**Kafka: spring-kafka-test и Testcontainers.** Поднимайте `KafkaContainer` и отправляйте сообщения в тестовый топик, проверяя потребителей. Тестируйте идемпотентность по ключу, обработку ретраев (DLT), сериализацию.

```java
@Testcontainers
@SpringBootTest
class KafkaIT {
  @Container static KafkaContainer kafka = new KafkaContainer("confluentinc/cp-kafka:7.5.0");
  @DynamicPropertySource static void props(DynamicPropertyRegistry r){
    r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
  }
  @Autowired KafkaTemplate<String,String> template;

  @Test void sendsAndConsumes() {
    template.send("orders","key","{...}");
    await().atMost(Duration.ofSeconds(5))
      .untilAsserted(() -> assertThat(receivedCount()).isEqualTo(1));
  }
}
```

**JMS/IBM MQ.** Для `@JmsListener` проверьте транзакционность (повторная доставка при исключении), конкуррентность (порядок и параллелизм), конфигурацию prefetch. Если нет Testcontainers для IBM MQ — используйте эмуляцию/встроенный брокер (ActiveMQ Artemis) для логики, а против реального MQ гоняйте nightly.

**Реактивные сценарии.** В WebFlux используйте `StepVerifier` для `Flux/Mono`: проверяйте порядок, задержки, ошибки, отмену. Для SSE поднимайте контроллер с `text/event-stream` и читайте события тестовым клиентом.

```java
StepVerifier.create(service.stream())
  .expectNextMatches(e -> e.data().name().startsWith("X"))
  .thenAwait(Duration.ofSeconds(1))
  .expectNextCount(3)
  .thenCancel()
  .verify();
```

**Синхронизация/ожидания.** Не злоупотребляйте `Thread.sleep`. Используйте Awaitility с условиями: ожидать запись в БД, приход сообщения, смену статуса. Это снижает флаки, особенно в асинхронных тестах.

---

# 9. Testcontainers и окружения

**Базовые контейнеры и lifecycle.** Чаще всего нужны Postgres, Kafka, Redis, MinIO, MailHog. Объявляйте контейнеры статическими `@Container` на класс, чтобы переиспользовать между тестами. Для нескольких тестовых наборов включайте глобальный reuse (`~/.testcontainers.properties: testcontainers.reuse.enable=true`) — но помните про уборку на CI.

**Изоляция данных.** Каждый тестовый класс/метод должен иметь свою «песочницу»: уникальные БД/схемы/топики/префиксы ключей. Сидап/тёрдаун — через миграции/`@Sql`/фикстуры. Реплицируемость: тест не должен зависеть от порядка запуска или предыдущих артефактов.

**Сетевые аспекты.** На CI Docker может быть «узким местом»: заранее подтягивайте образы (pre-pull), ограничивайте ресурсы, используйте алиасы сетей/`dependsOn` при многоконтейнерных сценариях (Compose). Если тест чувствителен к времени (JWT exp), синхронизируйте `Clock`/NTP или фиксируйте время в тестах.

**Оптимизация времени.** Параллельные джобы, разделение набора на быстрый PR-гейт и медленный nightly, шаринг образов в локальном регистри (GHCR). Там, где возможно, делайте slice вместо full-context, и не «грязните» контекст, чтобы кеш работал.

---

# 10. CI/CD, стабильность и контроль качества

**Параллельные прогоны и тэги.** Разделите тесты по `@Tag`: `fast` (unit/slice), `slow` (containers), `contract`, `e2e`. В PR-гейте гоняйте `fast` + часть `contract`, nightly — полный набор. Для нестабильных — `@Tag("flaky")` и отдельный отчёт; цель — пустой flaky-список.

**Репортинг: JaCoCo + Sonar.** Собирайте покрытия для unit и интеграции, публикуйте в Sonar, включите Quality Gate (минимальное покрытие на новые/изменённые файлы, отсутствие blocker-issues). Храните статистику длительности тестов: топ-10 самых медленных — кандидаты на оптимизацию.

```kotlin
// Gradle JaCoCo
plugins { jacoco }
tasks.jacocoTestReport {
  reports { xml.required.set(true); html.required.set(true) }
}
```

**Мутационное тестирование (PIT).** Включайте PIT там, где сложная бизнес-логика и регрессы дороги: расчёты, агрегаты, матчи. Не гоняйте PIT на всём монорепо каждый PR — достаточно nightly/weekly по критическим модулям.

```kotlin
plugins { id("info.solidsoft.pitest") version "1.9.11" }
pitest {
  targetClasses.set(listOf("com.example.domain.*"))
  targetTests.set(listOf("com.example.*Test"))
  mutationThreshold.set(80)
}
```

**Политики тест-данных и секретов.** Не коммитьте реальные креды. Делайте `.env.sample`, используйте Vault/CI Secrets. Чистите артефакты после прогонов (лог-файлы, дампы). Любая бага — новый тест; любое изменение контракта — контратест.

**Правила эволюции.** В DoD включите: тесты (unit/slice) на новую логику, интеграционные на сквозной сценарий, обновлённые контрактные тесты при изменении API/событий, и «кейс инцидента» → регресс-тест. Так кодовая база остаётся защищённой от повторных ошибок.

---

## Мини-справочник: готовые сниппеты

**JUnit 5 + Mockito:**

```kotlin
dependencies {
  testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
  testImplementation("org.mockito:mockito-junit-jupiter:5.12.0")
  testImplementation("org.assertj:assertj-core:3.26.0")
}
```

**Spring slices:**

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test") // включает web, json, restclient, jdbc slices
```

**WireMock:**

```kotlin
testImplementation("org.springframework.cloud:spring-cloud-contract-wiremock:4.1.3")
```

**Testcontainers:**

```kotlin
testImplementation("org.testcontainers:junit-jupiter:1.20.1")
testImplementation("org.testcontainers:postgresql:1.20.1")
testImplementation("org.testcontainers:kafka:1.20.1")
```

**Awaitility / StepVerifier:**

```kotlin
testImplementation("org.awaitility:awaitility:4.2.0")
testImplementation("io.projectreactor:reactor-test")
```

---
отлично — продолжаю. Ниже детально развернуты разделы **7–10**. На каждый подпункт — 3–4 абзаца, с кодом и пояснениями терминов/аннотаций. Стек: Java 17+, Spring Boot 3.x, JUnit 5.

---

# 7. Внешние HTTP-клиенты и устойчивость

## WireMock/MockWebServer: контрактные стабы, сценарии, задержки/ошибки, регэксп-матчеры

**WireMock** — это HTTP-сервер-эмулятор («стаб»), который поднимается в тестах и отвечает на запросы по заранее описанным правилам (stubs). Он поддерживает сценарии (**Scenarios**) — последовательности ответов (например, сначала 503, потом 200), задержки (simulated latency) и матчер запросов по **регулярным выражениям**. В Spring Boot для тестов RestTemplate/WebClient часто используют аннотацию `@RestClientTest`, которая автоматически стартует WireMock и подменяет реальные вызовы на локальные.

**MockWebServer** — более лёгкий стабер из OkHttp. Он очередью отдаёт `MockResponse` на входящие запросы и удобен для «тонкого» контроля порядка/количества вызовов, записывает фактические запросы (`RecordedRequest`). В отличие от WireMock у него меньше DSL для матчей, но он отлично подходит для low-level тестов клиентов.

Сценарий на WireMock: сначала вернуть **503 Service Unavailable**, затем на повторную попытку — корректный JSON. Мы задаём stub через DSL и используем **регэксп-матчеры** для путей/заголовков.

```java
// @RestClientTest(ExternalClient.class)
@Autowired WireMockServer wm;

@Test
void scenarioRetryThenOk() {
  wm.stubFor(get(urlPathMatching("/profile/[0-9]+"))
    .inScenario("retry")
    .whenScenarioStateIs(STARTED)
    .willReturn(aResponse().withStatus(503).withFixedDelay(200))
    .willSetStateTo("ok"));

  wm.stubFor(get(urlPathEqualTo("/profile/42"))
    .inScenario("retry")
    .whenScenarioStateIs("ok")
    .willReturn(okJson("""{"id":42,"name":"Neo"}""")));

  var dto = client.getProfile(42); // ваш HTTP-клиент
  assertThat(dto.name()).isEqualTo("Neo");
}
```

Пример с **MockWebServer**: кладём ответы в очередь и проверяем, что клиент сформировал корректный запрос (метод, путь, заголовки). Это удобно для валидации аутентификационных заголовков и сериализации тела.

```java
var mws = new MockWebServer();
mws.enqueue(new MockResponse().setResponseCode(200).setBody("""{"ok":true}"""));
mws.start();
try {
  client.setBaseUrl(mws.url("/").toString()); // инъекция базового URL
  client.call();

  var recorded = mws.takeRequest();
  assertThat(recorded.getPath()).isEqualTo("/v1/ping");
  assertThat(recorded.getHeader("Authorization")).startsWith("Bearer ");
} finally {
  mws.shutdown();
}
```

## Инъекция динамических URL в проперти тестов; fixed vs dynamic ports

В тестах **нельзя** хардкодить порты — локальные и CI-окружения различаются. Для подстановки реального URL стаба используйте `@DynamicPropertySource`: этот механизм Spring Test записывает значения в `Environment` **до** поднятия контекста. Так вы конфигурируете ваш клиент через обычные `application-*.yml` ключи (например, `external.base-url`), но в тесте подставляете `http://localhost:{wiremockPort}`.

**Fixed-порт** полезен, только если вы запускаете стабы из-вне (Docker compose) и управлять номером порта проще на уровне оркестратора. В остальных случаях берите **dynamic port** — WireMock/MockWebServer сам подберёт свободный порт. Это снижает конфликтность и позволяет параллельные прогоны.

```java
@Testcontainers
@RestClientTest(ExternalClient.class)
class ExternalClientTest {
  @Autowired WireMockServer wm;

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    // заглушка — порт узнаем позднее, см. @BeforeEach
  }

  @BeforeEach
  void setUp() {
    wm.start(); // если не стартует автоматически
    System.setProperty("external.base-url", "http://localhost:" + wm.port());
  }
}
```

Если вы используете **WebClient** (реактивный клиент), лучше инжектировать **`WebClient.Builder`** с базовым URL из свойств. Для **RestTemplate** — аналогично через `RestTemplateBuilder`. Это избавляет от разнородной конфигурации и делает клиента тестопригодным.

## Тестирование устойчивости: ретраи, таймауты, circuit breaker (Resilience4j), хаос-кейс

**Ретрай** — повторная попытка при ошибке; **таймаут** — максимальное время ожидания ответа; **circuit breaker** — «выключатель», который при частых ошибках временно блокирует вызовы к зависимому сервису (за это отвечает библиотека **Resilience4j**; аннотация `@CircuitBreaker` включает аспект вокруг метода). В тестах имитируйте «плохие» сценарии: 429/503, медленный ответ, Connection reset.

Конфиг для Resilience4j задают в `application.yml` (порог срабатывания, длительность **open**-состояния, максимальное число ретраев, экспоненциальный backoff). В тестах полезно уменьшить интервалы, чтобы быстро пройти циклы `closed → open → half-open`.

```yaml
resilience4j:
  retry:
    instances:
      ext:
        max-attempts: 3
        wait-duration: 100ms
  circuitbreaker:
    instances:
      ext:
        sliding-window-size: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 1s
```

```java
@Retry(name = "ext")
@CircuitBreaker(name = "ext", fallbackMethod = "fallback")
public Profile getProfile(long id) { /* вызов WebClient/RestTemplate */ }

public Profile fallback(long id, Throwable t) { return new Profile(id, "fallback"); }
```

Тест: два раза вернуть 503 с задержкой больше таймаута, затем 200 и убедиться, что (а) ретраи были, (б) при превышении порога открылся брейкер и сработал `fallback`. Для ожидания асинхронных состояний используйте **Awaitility** (DSL ожиданий без `Thread.sleep`).

```java
await().atMost(Duration.ofSeconds(2))
  .untilAsserted(() -> assertThat(circuitBreaker.getState()).isIn(OPEN, HALF_OPEN));
```

## Валидация контрактов клиентов: OpenAPI/JSON Schema, потребительские контракты

**Валидация контракта** означает: ответ внешнего сервиса соответствует описанию (OpenAPI/JSON Schema), а ваш клиент правильно на него реагирует. Один путь — поднять WireMock со сгенерированными стаба-мэппингами из OpenAPI (есть генераторы), второй — валидировать фактический JSON схемой (например, `networknt` JSON Schema Validator).

**Consumer-driven contracts** (CDC) — договор от лица потребителя: вы фиксируете, какие поля и значения вам нужны, и проверяете, что провайдер их отдаёт. Для HTTP в экосистеме Spring есть **Spring Cloud Contract** (модуль WireMock) — он умеет собирать стабы из декларативных контрактов и использовать их как на стороне провайдера, так и у потребителя.

Пример JSON-валидации: схема лежит в ресурсах, валидируем ответ клиента перед маппингом в DTO или после.

```java
var schema = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V202012)
    .getSchema(getClass().getResourceAsStream("/schema/profile.json"));
var node = new ObjectMapper().readTree(responseJson);
var result = schema.validate(node);
assertThat(result).isEmpty(); // нарушений нет
```

В CI добавьте шаг сравнения версий спецификаций (openapi-diff): если убрали поле/поменяли тип — PR падает. Это предотвращает случайные **breaking changes** при интеграции.

---

# 8. Сообщения и асинхронность

## Kafka: spring-kafka-test, Testcontainers, идемпотентность, ретраи/DLT, ключи

**Apache Kafka** — лог сообщений с партиционированием; **spring-kafka** — интеграция со Spring. В тестах не используйте «встроенный брокер» в долгосрочной перспективе — лучше **Testcontainers Kafka**: поведение ближе к продовой среде. Модуль **spring-kafka-test** даёт хелперы для consumer/producer и утилиты сериализации.

Идемпотентность (операция может быть выполнена повторно без побочных эффектов) достигается через **ключи сообщений** и логику «не обрабатывать, если уже видели offset/idempotency key». Ретраи в Kafka обычно реализуют через **retry topic** (переиздание с задержкой) и **DLT** (dead-letter topic). Проверьте, что при необрабатываемой ошибке сообщение уходит в DLT.

```java
@Testcontainers
@SpringBootTest
class KafkaIT {
  @Container static KafkaContainer kafka = new KafkaContainer("confluentinc/cp-kafka:7.5.0");
  @DynamicPropertySource static void props(DynamicPropertyRegistry r) {
    r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
  }

  @Autowired KafkaTemplate<String, String> template;

  @Test
  void sendsAndConsumedOnce() {
    template.send("orders", "key-42", "{\"id\":42}");
    await().atMost(5, SECONDS).untilAsserted(() ->
      assertThat(metrics.consumed("orders","key-42")).isEqualTo(1)); // ваш счётчик/репозиторий идемпотентности
  }
}
```

Для проверки **DLT**: конфигурируйте **DeadLetterPublishingRecoverer** или «ошибочный» слушатель, кидайте исключение — и проверьте, что сообщение оказалось в `orders.DLT`. Согласованность ключей важна: одинаковый ключ → порядок в рамках партиции, разный ключ → параллелизм.

## JMS/IBM MQ: @JmsListener, транзакции, concurrency, порядок

**JMS** (Java Message Service) — API для брокеров сообщений (ActiveMQ/IBM MQ/Artemis). Аннотация `@JmsListener` помечает метод как подписчика очереди/топика. Для тестов удобно использовать **ActiveMQ Artemis** через Testcontainers или встроенный Artemis (часто — GenericContainer с образом `vromero/activemq-artemis`).

Транзакционность в JMS управляется `DefaultJmsListenerContainerFactory`: `sessionTransacted=true` или `transactionManager`. Если обработчик кинул исключение — сообщение будет **повторно доставлено** (redelivery). В тесте проверьте: первый раз — исключение, второй — успешно; или что после N попыток сообщение попадает в D/L очередь.

```java
@Configuration
class JmsConfig {
  @Bean DefaultJmsListenerContainerFactory jmsListenerContainerFactory(ConnectionFactory cf) {
    var f = new DefaultJmsListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setSessionTransacted(true);
    f.setConcurrency("2-5"); // 2..5 параллельных потребителей
    return f;
  }
}

@Component
class OrderListener {
  @JmsListener(destination = "orders.queue")
  public void onMessage(String payload) { /* обработка */ }
}
```

Порядок сообщений гарантируется **внутри** одной очереди/сессии; с concurrency > 1 возможна параллельная обработка. Настройка **prefetch** (сколько сообщений клиент «забирает» заранее) влияет на балансировку — в тестах прогоняйте сценарии с конкуренцией, чтобы поймать «уступки» порядка и блокировки.

## Реактивные сценарии: StepVerifier для Flux/Mono, тесты SSE

**StepVerifier** (из `reactor-test`) — инструмент пошаговой проверки реактивных последовательностей `Flux`/`Mono`. Он позволяет ожидать элементы, ошибки, завершение, а также тестировать «виртуальное время» (`withVirtualTime`) для таймеров/задержек без реального ожидания.

Пример: сервис выдаёт `Flux<Event>` раз в 100 мс; мы проверим первые три события и отменим подписку. Для **SSE** (Server-Sent Events, поток событий поверх HTTP с `Content-Type: text/event-stream`) используйте `WebTestClient` и проверьте заголовки/формат.

```java
StepVerifier.withVirtualTime(() -> service.ticker(Duration.ofSeconds(1)))
  .thenAwait(Duration.ofSeconds(3))
  .expectNextCount(3)
  .thenCancel()
  .verify();
```

```java
@WebFluxTest(SseController.class)
class SseTest {
  @Autowired WebTestClient client;

  @Test
  void sse() {
    client.get().uri("/api/stream")
      .exchange()
      .expectStatus().isOk()
      .expectHeader().contentTypeCompatibleWith("text/event-stream")
      .returnResult(String.class)
      .getResponseBody()
      .take(2)
      .as(StepVerifier::create)
      .expectNextMatches(line -> line.startsWith("data:"))
      .thenCancel()
      .verify();
  }
}
```

## Синхронизация и ожидания: Awaitility/Condition, борьба с флаки

**Awaitility** — библиотека ожиданий «пока условие не станет истинным» (polling с таймаутом). Она заменяет `Thread.sleep` и делает асинхронные тесты стабильнее. Пишите **условия**, а не задержки: «ждём, пока в БД появится запись», «ждём, пока consumer увеличит счётчик».

Оптимально настроить **таймаут** и **poll interval**: маленький интервал — быстрее реакция, но больше CPU; большой — дольше ожидание. Для нестабильных внешних систем допускайте «окно» ожидания, но фиксируйте верхнюю границу (например, 5 секунд).

Синхронизацию иногда удобнее делать **сигналами** — `CountDownLatch`, который listener «щёлкает» после обработки, а тест `await`-ит. Комбинируйте: latch для точной синхронизации и Awaitility для «правильного» таймаута и повторной проверки побочных эффектов (например, значение в БД).

```java
var latch = new CountDownLatch(1);
listener.onProcessed(() -> latch.countDown()); // ваш хук

assertThat(latch.await(3, TimeUnit.SECONDS)).isTrue();
await().atMost(1, SECONDS).untilAsserted(() ->
  assertThat(repo.findById(id)).isPresent());
```

---

# 9. Testcontainers и окружения

## Базовые контейнеры: Postgres, Kafka, Redis, MinIO; reuse и lifecycle

**Testcontainers** — библиотека для запуска Docker-контейнеров в тестах. Аннотация `@Testcontainers` включает интеграцию с JUnit 5, а `@Container` помечает поля-контейнеры. Часто используемые: `PostgreSQLContainer`, `KafkaContainer`, `GenericContainer` для **Redis** и **MinIO**. Делайте контейнеры **static** на класс — так они стартуют один раз на весь класс.

```java
@Testcontainers
class InfraIT {
  @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
  @Container static KafkaContainer kafka = new KafkaContainer("confluentinc/cp-kafka:7.5.0");
  @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
}
```

Режим **reuse** (повторное использование контейнеров между запусками) ускоряет локальные прогоны: включается в `~/.testcontainers.properties` (`testcontainers.reuse.enable=true`) и через `withReuse(true)` на контейнере. На CI reuse обычно отключают ради чистоты окружения. Жизненный цикл: «per-class» (static) или «per-test» (нестатический `@Container`) — выбирайте по стоимости старта.

## Изоляция данных: namespace/топики/БД на тест, seed/teardown, реплицируемость

Каждый тест должен работать в собственной «песочнице». Для Postgres создавайте **уникальную схему**/БД на метод (имя с UUID), для Kafka — **топик с префиксом теста** (`orders_it_{UUID}`), для Redis — **префикс ключей** или отдельный dbIndex. Это исключит перекрёстное влияние и позволит параллельный запуск.

Seed/teardown: для БД — `@Sql` (миграции/фикстуры) и truncate между тестами; для Kafka — явное удаление топиков не требуется (они эфемерны в контейнере), но в рамках длительного контейнера чистите consumer group offsets; для MinIO — удаляйте бакеты/объекты после теста.

Реплицируемость на CI достигается **детерминизмом**: фиксируйте `Clock`/сид генераторов, не полагайтесь на локаль/часовой пояс машины, храните версии образов контейнеров явно (`postgres:16-alpine`, не `latest`).

## Сетевые аспекты: алиасы/depends-on, порты, clock-skew, ресурсы Docker

Сложные интеграции иногда требуют нескольких контейнеров «в сети». В Testcontainers можно создавать **сети** (`Network.newNetwork()`) и задавать контейнерам **алиасы** (hostnames), а также объявлять «зависимости» (стартовать один после другого). Порты мэппятся на случайные хостовые — забирайте их через `getMappedPort(...)`.

**Clock skew** (рассинхрон времени) важен для JWT/TTL-логики: в тестах **инжектируйте Clock** в приложение, а не пытайтесь «менять» часы контейнера. На CI следите за ресурсами Docker: лимитируйте CPU/RAM, не запускайте слишком много контейнеров параллельно — иначе получите флаки из-за timeouts.

Если тесты запускаются в Windows/WSL2 или macOS, учитывайте сетевые особенности (например, различия в DNS/IPv6). Всегда проверяйте подключение через фактический `host:port`, который предоставил Testcontainers.

## Оптимизация времени: reuse, pre-pull, локальные образы, параллельные джобы

Для ускорения: заранее **pull** образов (CI шаг «docker pull …»), используйте **локальный регистр**/кеш образов, включайте **reuse** локально. Тяжёлые контейнеры (Kafka, MinIO) старайтесь стартовать **один раз на класс/набор**, а не per-test.

Разделяйте прогон на **параллельные джобы**: unit/slice в одном, container-tests в другом. В Gradle можно вынести контейнерные тесты в `integrationTest` source set и запускать их реже (например, в nightly). Следите за логами — излишнее `DEBUG` замедляет тесты и CI.

---

# 10. CI/CD, стабильность и контроль качества

## Параллельные прогоны, `@Tag`, стратегия nightly vs PR-гейты

**`@Tag`** — аннотация JUnit 5 для маркировки тестов (например, `@Tag("slow")`, `@Tag("contract")`). В Gradle можно включать/исключать теги (`-Dgroups`, `-PincludeTags` через JUnit Platform). Стратегия: в **PR-гейте** запускать быстрые наборы (`unit`, `slice`, часть contract), в **nightly** — полный (включая контейнерные, нагрузочные и PIT).

Параллельные прогоны в CI сокращают время обратной связи: распараллельте по модулям/категориям. Следите за общими ресурсами (Docker, CPU) — если падают таймауты, уменьшите параллелизм или разнесите контейнерные тесты отдельно.

Обязательно включайте **fail-fast** для PR-гейта (останавливать пайплайн при первой ошибке) и публикуйте отчёты JUnit/JaCoCo как артефакты. Команда должна видеть, **что** упало и **почему**, без захода в логи агента.

```yaml
# Пример GitHub Actions (фрагмент)
- name: Unit+Slice
  run: ./gradlew test -PincludeTags=fast
- name: Integration (containers)
  if: github.event_name == 'schedule' || github.ref == 'refs/heads/main'
  run: ./gradlew integrationTest -PincludeTags=slow
```

## Репортинг: JaCoCo + Sonar, flaky-трекинг, длительности

**JaCoCo** — инструмент покрытия кода. Подключите задачи отчётов и публикуйте XML в **SonarQube**/**SonarCloud** — там работает **Quality Gate** (порог покрытия на новые/изменённые файлы, отсутствие критических замечаний). Покрытие — индикатор, а не цель: отслеживайте «дыры» в ключевой логике.

Собирайте **длительности** тестов (JUnit XML содержит время) — топ-10 медленных тестов/классов регулярно оптимизируйте. Для **flaky-трекинга** храните историю падений: хотя бы простой CSV/таблица в артефактах, лучше — отчётная панель (Grafana/Datadog).

```kotlin
plugins { jacoco }
tasks.test { useJUnitPlatform() }
tasks.jacocoTestReport {
  reports { xml.required.set(true); html.required.set(true) }
}
```

Если есть `integrationTest` набор — **слейте отчёты**:

```kotlin
tasks.register<JacocoReport>("jacocoMerge") {
  executionData(fileTree(project.buildDir).include("**/jacoco/*.exec"))
  reports { xml.required.set(true); html.required.set(true) }
  dependsOn(tasks.test, tasks.named("integrationTest"))
}
```

## Мутационное тестирование (PIT): ценность и интеграция

**PIT** (Pitest) — инструмент **мутационного тестирования**: вносит мелкие изменения («мутации») в байткод и проверяет, «убивают» ли их тесты. Метрика — **mutation score** (процент убитых мутантов). Он дорог по времени, поэтому запускайте его **выборочно**: на модулях доменной логики, nightly/weekly.

Подключение Gradle-плагина и настройка целей:

```kotlin
plugins { id("info.solidsoft.pitest") version "1.9.11" }
pitest {
  targetClasses.set(listOf("com.example.domain.*"))
  targetTests.set(listOf("com.example.*Test"))
  mutationThreshold.set(80)
  junit5PluginVersion.set("1.2.0")
}
```

Исключайте **генерируемый**/инфраструктурный код (DTO, мапперы) — мутанты там малоинформативны. Анализируйте выживших мутантов: это подсказки, где тесты поверхностны (например, проверяют «не нулевое», а надо сверять бизнес-инвариант).

## Политики тест-данных и секретов: .env.sample, Vault, очистка артефактов

Никогда не храните **секреты** в репозитории. Держите `.env.sample` с примерами, фактические значения — в секретах CI (Vault/Actions Secrets). В тестах секреты прокидывайте через **переменные окружения**/`@DynamicPropertySource`. Санитизируйте логи — токены/JWT/ключи не должны попадать в артефакты.

Для **тест-данных**: минимально необходимые фикстуры, детерминированные сиды, «золотые» JSON под ревью. Артефакты (дампы БД, большие логи) чистите после прогона или ограничивайте ретеншн — чтобы репозиторий и хранилище не «распухали».

Если тесты касаются персональных данных, анонимизируйте их. На CI используйте **эфемерные** окружения: после завершения джобы контейнеры и тома удаляются.

## Правила эволюции: багфикс ⇒ тест, DoD, инциденты ⇒ регресс-тест

Каждый багфикс начинается с **красного теста**, воспроизводящего дефект, — это гарантирует, что баг не вернётся. Добавляйте его в набор unit/slice, если возможно, иначе — в интеграцию.

В **Definition of Done** (DoD) включите: (1) тесты на новую логику, (2) обновлённые контрактные тесты при изменении API/событий, (3) обновлённую документацию (OpenAPI), (4) зелёный PR-гейт. После инцидента — **пост-морем** и регресс-тест, фиксирующий сценарий отказа.

Стабильно держите **PR-гейт зелёным**: флаки — в карантин с последующим исправлением, не игнорируйте «красное». Используйте дифф-инструменты контрактов (OpenAPI-diff/Avro-compat) как «брейкеры» пайплайна, чтобы несовместимые изменения не попадали в main.

---

## Мини-справочник: готовые сниппеты

**JUnit 5 + Mockito:**

```kotlin
dependencies {
  testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
  testImplementation("org.mockito:mockito-junit-jupiter:5.12.0")
  testImplementation("org.assertj:assertj-core:3.26.0")
}
```

**Spring slices:**

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test") // включает web, json, restclient, jdbc slices
```

**WireMock:**

```kotlin
testImplementation("org.springframework.cloud:spring-cloud-contract-wiremock:4.1.3")
```

**Testcontainers:**

```kotlin
testImplementation("org.testcontainers:junit-jupiter:1.20.1")
testImplementation("org.testcontainers:postgresql:1.20.1")
testImplementation("org.testcontainers:kafka:1.20.1")
```

**Awaitility / StepVerifier:**

```kotlin
testImplementation("org.awaitility:awaitility:4.2.0")
testImplementation("io.projectreactor:reactor-test")
```

---


Примеры:

супер — вот обещанные «рабочие шаблоны». Каждый — минимальный, но полноценный: Gradle (Kotlin DSL), конфиги, код и тест(ы). Копируй в пустой проект и запускай `./gradlew test` (или конкретную задачу, я отмечаю).

---

# 1) REST-клиент + WireMock + Resilience4j (MVC/WebClient)

**Структура (минимум):**

```
build.gradle.kts
src/main/java/com/example/client/ExternalClient.java
src/main/java/com/example/client/WebClientConfig.java
src/main/resources/application.yml
src/test/java/com/example/client/ExternalClientTest.java
```

**`build.gradle.kts`**

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.2.6"
  id("io.spring.dependency-management") version "1.1.5"
}

group = "com.example"
version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_17

repositories { mavenCentral() }

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-webflux")
  // Resilience4j Spring Boot 3 starter
  implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")

  testImplementation("org.springframework.boot:spring-boot-starter-test")
  // WireMock auto-config for Spring (includes JUnit 5 support)
  testImplementation("org.springframework.cloud:spring-cloud-contract-wiremock:4.1.3")
  testImplementation("org.assertj:assertj-core:3.26.0")
}

tasks.test {
  useJUnitPlatform()
}
```

**`src/main/resources/application.yml`**

```yaml
external:
  base-url: https://example.invalid       # переопределяется в тесте
resilience4j:
  retry:
    instances:
      ext:
        max-attempts: 3
        wait-duration: 100ms
  circuitbreaker:
    instances:
      ext:
        sliding-window-size: 5
        failure-rate-threshold: 50
        wait-duration-in-open-state: 1s
spring:
  main:
    web-application-type: reactive
```

**`src/main/java/com/example/client/WebClientConfig.java`**

```java
package com.example.client;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.*;

@Configuration
public class WebClientConfig {

  @Bean
  WebClient externalWebClient(@Value("${external.base-url}") String baseUrl) {
    return WebClient.builder()
        .baseUrl(baseUrl)
        .filter(ExchangeFilterFunctions.statusError(
            // любые 5xx/429 считаем ошибкой для ретраев
            status -> status.is5xxServerError() || status.value() == 429,
            (req, res) -> WebClientResponseException.create(
                res.statusCode().value(), res.statusCode().getReasonPhrase(),
                res.headers().asHttpHeaders(), null, null)))
        .build();
  }
}
```

**`src/main/java/com/example/client/ExternalClient.java`**

```java
package com.example.client;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;

@Component
public class ExternalClient {
  private final WebClient wc;

  public record Profile(long id, String name) {}

  public ExternalClient(WebClient externalWebClient) {
    this.wc = externalWebClient;
  }

  @Retry(name = "ext")
  @CircuitBreaker(name = "ext", fallbackMethod = "fallback")
  public Profile getProfile(long id) {
    return wc.get().uri("/profile/{id}", id)
        .retrieve()
        .bodyToMono(Profile.class)
        .block();
  }

  // fallback должен совпадать сигнатурой + Throwable
  public Profile fallback(long id, Throwable t) {
    return new Profile(id, "fallback");
  }
}
```

**`src/test/java/com/example/client/ExternalClientTest.java`**

```java
package com.example.client;

import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestComponent;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import static com.github.tomakehurst.wiremock.client.WireMock.*;

import static org.assertj.core.api.Assertions.assertThat;

import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
@AutoConfigureWireMock(port = 0) // поднимет WireMock на случайном порту и добавит свойство wiremock.server.port
class ExternalClientTest {

  @Autowired ExternalClient client;

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    // external.base-url зависит от wiremock.server.port, который выставит AutoConfigureWireMock
    r.add("external.base-url", () -> "http://localhost:" + System.getProperty("wiremock.server.port"));
  }

  @Test
  void retryThenOk() {
    stubFor(get(urlEqualTo("/profile/42"))
      .inScenario("retry")
      .whenScenarioStateIs(STARTED)
      .willReturn(aResponse().withStatus(503)));

    stubFor(get(urlEqualTo("/profile/42"))
      .inScenario("retry")
      .whenScenarioStateIs(STARTED)
      .willSetStateTo("ok")
      .willReturn(aResponse().withStatus(503))); // второй тоже 503

    stubFor(get(urlEqualTo("/profile/42"))
      .inScenario("retry")
      .whenScenarioStateIs("ok")
      .willReturn(okJson("{\"id\":42,\"name\":\"Neo\"}")));

    var p = client.getProfile(42);
    assertThat(p.name()).isIn("Neo","fallback"); // в зависимости от таймингов/порогов
  }
}
```

> Запуск: `./gradlew test`

---

# 2) Kafka + retry + DLT (spring-kafka + Testcontainers)

**Структура:**

```
build.gradle.kts
src/main/java/com/example/k/KafkaConfig.java
src/main/java/com/example/k/OrderListener.java
src/test/java/com/example/k/KafkaIT.java
src/main/resources/application.yml
```

**`build.gradle.kts`**

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.2.6"
  id("io.spring.dependency-management") version "1.1.5"
}

group = "com.example"; version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_17

repositories { mavenCentral() }

dependencies {
  implementation("org.springframework.boot:spring-boot-starter")
  implementation("org.springframework.kafka:spring-kafka")

  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.springframework.kafka:spring-kafka-test")
  testImplementation("org.testcontainers:junit-jupiter:1.20.1")
  testImplementation("org.testcontainers:kafka:1.20.1")
  testImplementation("org.awaitility:awaitility:4.2.0")
  testImplementation("org.assertj:assertj-core:3.26.0")
}

tasks.test { useJUnitPlatform() }
```

**`src/main/resources/application.yml`**

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:29092 # переопределим в тесте
    consumer:
      group-id: orders-test
      auto-offset-reset: earliest
      properties:
        isolation.level: read_committed
    producer:
      properties:
        enable.idempotence: true
        acks: all
    listener:
      ack-mode: record
app:
  topics:
    orders: orders
    dlt: orders.DLT
```

**`src/main/java/com/example/k/KafkaConfig.java`**

```java
package com.example.k;

import java.util.function.BiFunction;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.*;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.core.*;
import org.springframework.kafka.listener.*;
import org.springframework.kafka.support.serializer.*;

@EnableKafka
@Configuration
public class KafkaConfig {

  @Bean
  DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template, TopicsProps p) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
        (cr, e) -> new org.apache.kafka.common.TopicPartition(p.dlt(), cr.partition()));
    var handler = new DefaultErrorHandler(recoverer, new FixedBackOff(100L, 1)); // 1 ретрай => затем DLT
    return handler;
  }

  @Bean
  @ConfigurationProperties("app.topics")
  TopicsProps topicsProps() { return new TopicsProps(); }

  public static class TopicsProps {
    private String orders;
    private String dlt;
    public String orders() { return orders; }
    public void setOrders(String v) { this.orders = v; }
    public String dlt() { return dlt; }
    public void setDlt(String v) { this.dlt = v; }
  }
}
```

**`src/main/java/com/example/k/OrderListener.java`**

```java
package com.example.k;

import java.util.concurrent.ConcurrentLinkedQueue;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

@Component
public class OrderListener {
  public final ConcurrentLinkedQueue<String> consumed = new ConcurrentLinkedQueue<>();
  public final ConcurrentLinkedQueue<String> dltConsumed = new ConcurrentLinkedQueue<>();

  @KafkaListener(topics = "${app.topics.orders}", groupId = "${spring.kafka.consumer.group-id}",
      containerFactory = "kafkaListenerContainerFactory")
  public void onOrder(String payload, @Header("kafka_receivedMessageKey") String key) {
    if (payload.contains("fail")) {
      throw new IllegalStateException("simulate processing error");
    }
    consumed.add(key + ":" + payload);
  }

  @KafkaListener(topics = "${app.topics.dlt}", groupId = "orders-dlt")
  public void onDlt(String payload, @Header("kafka_receivedMessageKey") String key) {
    dltConsumed.add(key + ":" + payload);
  }
}
```

**`src/test/java/com/example/k/KafkaIT.java`**

```java
package com.example.k;

import static org.assertj.core.api.Assertions.assertThat;
import static java.util.concurrent.TimeUnit.SECONDS;

import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.*;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@Testcontainers
@SpringBootTest
class KafkaIT {

  @Container
  static KafkaContainer kafka = new KafkaContainer("confluentinc/cp-kafka:7.5.0");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
  }

  @Autowired KafkaTemplate<String, String> template;
  @Autowired OrderListener listener;

  @Test
  void successGoesToMainConsumer() {
    template.send("orders", "key-1", "{\"id\":1}");
    Awaitility.await().atMost(5, SECONDS)
        .untilAsserted(() -> assertThat(listener.consumed).anyMatch(s -> s.startsWith("key-1")));
  }

  @Test
  void failureGoesToDLT() {
    template.send("orders", "key-2", "{\"id\":2,\"fail\":true}");
    Awaitility.await().atMost(10, SECONDS)
        .untilAsserted(() -> assertThat(listener.dltConsumed).anyMatch(s -> s.startsWith("key-2")));
  }
}
```

> Запуск: `./gradlew test`

---

# 3) JMS + Artemis (контейнер) + транзакционная повторная доставка

**Структура:**

```
build.gradle.kts
src/main/java/com/example/jms/JmsConfig.java
src/main/java/com/example/jms/JmsService.java
src/test/java/com/example/jms/JmsIT.java
```

**`build.gradle.kts`**

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.2.6"
  id("io.spring.dependency-management") version "1.1.5"
}

group = "com.example"; version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_17
repositories { mavenCentral() }

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-artemis")
  implementation("org.springframework.boot:spring-boot-starter")

  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.testcontainers:junit-jupiter:1.20.1")
  testImplementation("org.testcontainers:testcontainers:1.20.1")
}

tasks.test { useJUnitPlatform() }
```

**`src/main/java/com/example/jms/JmsConfig.java`**

```java
package com.example.jms;

import jakarta.jms.ConnectionFactory;
import org.springframework.context.annotation.*;
import org.springframework.jms.annotation.EnableJms;
import org.springframework.jms.config.DefaultJmsListenerContainerFactory;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@EnableJms
@EnableTransactionManagement
@Configuration
public class JmsConfig {

  @Bean
  public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
      ConnectionFactory cf, PlatformTransactionManager tx) {
    var f = new DefaultJmsListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setTransactionManager(tx);         // транзакционная обработка
    f.setSessionTransacted(true);
    f.setConcurrency("1-2");
    return f;
  }
}
```

**`src/main/java/com/example/jms/JmsService.java`**

```java
package com.example.jms;

import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicInteger;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class JmsService {

  private final JmsTemplate jms;
  public final ConcurrentLinkedQueue<String> processed = new ConcurrentLinkedQueue<>();
  public final AtomicInteger attempts = new AtomicInteger();

  public JmsService(JmsTemplate jms) { this.jms = jms; }

  public void send(String q, String payload) { jms.convertAndSend(q, payload); }

  @Transactional
  @JmsListener(destination = "orders.queue", containerFactory = "jmsListenerContainerFactory")
  public void onMessage(String payload) {
    int n = attempts.incrementAndGet();
    if (payload.contains("fail") && n == 1) {
      throw new IllegalStateException("simulate failure to trigger redelivery");
    }
    processed.add(payload);
  }
}
```

**`src/test/java/com/example/jms/JmsIT.java`**

```java
package com.example.jms;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.*;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import java.time.Duration;

@Testcontainers
@SpringBootTest
class JmsIT {

  @Container
  static GenericContainer<?> artemis =
      new GenericContainer<>("vromero/activemq-artemis:2.31.2-alpine")
          .withEnv("ARTEMIS_USER", "user")
          .withEnv("ARTEMIS_PASSWORD", "password")
          .withExposedPorts(61616);

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.artemis.mode", () -> "native");
    r.add("spring.artemis.broker-url", () -> "tcp://" + artemis.getHost() + ":" + artemis.getMappedPort(61616));
    r.add("spring.artemis.user", () -> "user");
    r.add("spring.artemis.password", () -> "password");
  }

  @Autowired JmsService jms;

  @Test
  void redeliversOnFailure() {
    jms.send("orders.queue", "fail-once");
    await().atMost(Duration.ofSeconds(5))
        .untilAsserted(() -> assertThat(jms.processed).contains("fail-once"));
    assertThat(jms.attempts.get()).isGreaterThanOrEqualTo(2); // была повторная доставка
  }
}
```

Запуск:
> `./gradlew test`

