---
layout: page
title: "Spring Data JPA: Углубленно в JDBC"
permalink: /spring/jpa_deep
---

# Spring JPA: Углубленно в JDBC

Spring Data JPA прячет низкоуровневую работу с БД, но на проде неизбежны кейсы, где нужно тоньше управлять соединениями, транзакциями и SQL. Иногда это для производительности (батчи, потоковая выборка), иногда — для корректности (isolation, блокировки), а иногда — для прозрачности (точный контроль над запросом). Хорошая новость: весь этот «низкий уровень» в Spring интегрирован и дополняет JPA, а не конкурирует с ним.

Практический паттерн такой: **чтение** — чаще через проекции/JPQL/Specification, **командные операции** — через JPA с аккуратными транзакциями и настройками батча; **узкие места** и «тяжёлые» отчёты — через `JdbcTemplate`/нативный SQL. Ниже — инструменты, из которых вы будете собирать это решение.

---

# JDBC

JDBC — это базовый стандарт работы с реляционными БД в Java: подключение, подготовленные выражения, чтение `ResultSet`, управление транзакцией. В «чистом» виде он многословен: try-with-resources, ручное закрытие ресурсов, конвертация типов, обработка исключений. Но **он максимально предсказуем** и даёт полный контроль над SQL и курсорами.

Понимать JDBC нужно даже если вы пишете через JPA. Hibernate в итоге тоже выполняет SQL через драйвер JDBC; зная его поведение (авто-commit, fetch size, подготовка батчей), вы легче настраиваете JPA/Boot. Например, серверные курсоры в PostgreSQL начинают работать только **в транзакции** и при положительном `fetchSize` у стейтмента — иначе драйвер заберёт все строки в память.

Ниже — учебный пример «голого» JDBC: так вы увидите, что делает за вас `JdbcTemplate`. В прод-коде напрямую к `DriverManager` вы почти не обратитесь — вместо этого будет `DataSource` и пул соединений (HikariCP).

```java
public List<Book> findBooksRawJdbc(String titlePart) throws SQLException {
    String sql = "select id, title from books where lower(title) like lower(?)";
    List<Book> result = new ArrayList<>();
    try (Connection con = DriverManager.getConnection(
             "jdbc:postgresql://localhost:5432/app", "user", "pass");
         PreparedStatement ps = con.prepareStatement(sql)) {

        ps.setString(1, "%" + titlePart + "%");
        try (ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                Book b = new Book();
                b.setId(rs.getLong("id"));
                b.setTitle(rs.getString("title"));
                result.add(b);
            }
        }
    }
    return result;
}
```

Ключевые выводы: (1) **всегда** используйте `PreparedStatement` и параметры — это безопасность и план кэширования; (2) не забывайте закрывать ресурсы; (3) для длинных селектов на PostgreSQL используйте транзакции и `setFetchSize`, чтобы не проглатывать весь набор в память.

---

# DataSource

`DataSource` предоставляет соединения и обычно обёрнут пулом. В Spring Boot по умолчанию — **HikariCP**: быстрый и экономный. От качества настройки пула зависит стабильность и пропускная способность приложения под нагрузкой. Минимальные параметры: размер пула (`maximumPoolSize`), таймаут получения соединения (`connectionTimeout`), таймаут простоя (`idleTimeout`), время жизни (`maxLifetime`).

Настраивать лучше через `application.yml`, а не кодом — так проще варьировать по окружениям. В PostgreSQL есть критически важные свойства драйвера: `reWriteBatchedInserts=true` ускоряет батчи, а `preparedStatementCacheQueries`/`preparedStatementCacheSizeMiB` помогают кэшировать планы (актуально при PgJDBC 42.2+).

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app?reWriteBatchedInserts=true
    username: app
    password: secret
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000
      idle-timeout: 300000
      max-lifetime: 1800000
  jpa:
    properties:
      hibernate.jdbc.batch_size: 100
      hibernate.order_inserts: true
      hibernate.order_updates: true
```

Если нужна тонкая конфигурация — определите бин `DataSource` явно. Плюс: можно повесить метрики Micrometer и health-check’и, настроить «лейблы» пула, добавить валидатор соединений.

```java
@Bean
DataSource dataSource(DataSourceProperties props) {
    HikariConfig cfg = new HikariConfig();
    cfg.setJdbcUrl(props.getUrl());
    cfg.setUsername(props.getUsername());
    cfg.setPassword(props.getPassword());
    cfg.setMaximumPoolSize(20);
    cfg.setMinimumIdle(5);
    cfg.addDataSourceProperty("reWriteBatchedInserts", "true");
    return new HikariDataSource(cfg);
}
```

---

# JDBC Template

`JdbcTemplate` — тонкая обёртка над JDBC: управление ресурсами, обработка исключений в иерархию Spring (`DataAccessException`), удобные методы `query`, `update`, `batchUpdate`, `queryForObject`. Он не «ORM»: вы сами пишете SQL и маппинг. Для именованных параметров есть `NamedParameterJdbcTemplate`.

Простой запрос списка с лямбда-`RowMapper`:

```java
@Repository
public class BookJdbcDao {
  private final JdbcTemplate jdbc;

  public BookJdbcDao(JdbcTemplate jdbc) {
    this.jdbc = jdbc;
  }

  public List<Book> findByTitle(String q) {
    return jdbc.query("""
        select id, title
        from books
        where lower(title) like lower(?)
        """,
        (rs, i) -> new Book(rs.getLong("id"), rs.getString("title")),
        "%" + q + "%"
    );
  }

  public int updateTitle(long id, String title) {
    return jdbc.update("update books set title=? where id=?", title, id);
  }
}
```

Именованные параметры читаемее, особенно при длинных списках:

```java
@Repository
public class BookNamedDao {
  private final NamedParameterJdbcTemplate named;

  public BookNamedDao(NamedParameterJdbcTemplate named) { this.named = named; }

  public Optional<Long> insert(String title) {
    var params = new MapSqlParameterSource().addValue("title", title);
    KeyHolder kh = new GeneratedKeyHolder();
    named.update("insert into books(title) values(:title)", params, kh, new String[]{"id"});
    return Optional.ofNullable(kh.getKey()).map(Number::longValue);
  }
}
```

Плюсы `JdbcTemplate`: (1) полный контроль над SQL; (2) лёгкая батч-обработка; (3) потоковые чтения с `fetchSize`; (4) предсказуемость. Минусы: руками писать маппинг и следить за дублированием запросов — используйте репозитории там, где нужен доменный уровень и объектные графы.

---

# Маппинг строк (RowMapper)

`RowMapper<T>` — контракт для преобразования строки `ResultSet` в объект. Вы можете писать его лямбдой, отдельным классом или использовать рефлективный `BeanPropertyRowMapper` (он маппит по совпадению имён столбцов и свойств). Для простых DTO это удобно, для сложных — лучше явно.

Классический RowMapper:

```java
public class BookRowMapper implements RowMapper<Book> {
  @Override
  public Book mapRow(ResultSet rs, int rowNum) throws SQLException {
    Book b = new Book();
    b.setId(rs.getLong("id"));
    b.setTitle(rs.getString("title"));
    return b;
  }
}
```

Использование:

```java
List<Book> books = jdbc.query("select id, title from books", new BookRowMapper());
```

Для плоских DTO удобно использовать Java 16+ `record`:

```java
public record BookView(Long id, String title) {}

List<BookView> views = jdbc.query(
  "select id, title from books",
  (rs, i) -> new BookView(rs.getLong(1), rs.getString(2))
);
```

Если объём данных огромный и вы не хотите держать всё в памяти — вместо `RowMapper` используйте `RowCallbackHandler` и обрабатывайте строки «на лету»:

```java
jdbc.setFetchSize(1000);
jdbc.query("select id, title from books", (RowCallbackHandler) rs -> {
  // обработка каждой строки, например запись в файл/очередь
});
```

---

# Транзакции

Транзакции в Spring декларируются аннотацией `@Transactional` на **публичных** методах сервисов (важно: self-invocation не сработает из-за прокси). По умолчанию — `Propagation.REQUIRED` и `Isolation.DEFAULT`. Для чтений ставьте `@Transactional(readOnly = true)` — это снижает накладные расходы ORM и выставляет read-only флаг соединению у некоторых БД.

Пример сервисного слоя, где читаем через `JdbcTemplate`, а пишем через JPA:

```java
@Service
public class BookService {
  private final BookRepository repo;
  private final JdbcTemplate jdbc;

  public BookService(BookRepository repo, JdbcTemplate jdbc) {
    this.repo = repo;
    this.jdbc = jdbc;
  }

  @Transactional
  public Book createFromTitle(String title) {
    // Валидация через запрос
    Integer exists = jdbc.queryForObject(
        "select 1 from books where lower(title)=lower(?) limit 1",
        Integer.class, title);
    if (exists != null) throw new IllegalArgumentException("Duplicate title");

    Book b = new Book();
    b.setTitle(title);
    return repo.save(b);
  }

  @Transactional(readOnly = true)
  public List<Book> search(String q) {
    return repo.findByTitleContainingIgnoreCase(q, Sort.by("id"));
  }
}
```

Программная модель (когда нужна тонкая логика) — `TransactionTemplate`:

```java
@Service
public class BillingService {
  private final TransactionTemplate tx;

  public BillingService(PlatformTransactionManager tm) {
    this.tx = new TransactionTemplate(tm);
    this.tx.setReadOnly(false);
  }

  public void doWork() {
    tx.executeWithoutResult(status -> {
      // работа в транзакции
    });
  }
}
```

---

# Исключения в Spring JPA и rollback транзакций

Spring по умолчанию делает rollback при **unchecked** исключениях (`RuntimeException`, `Error`). Checked-исключения **не** приводят к откату, если явно не указать `rollbackFor = ...`. Это важно помнить: `IOException` в транзакции по умолчанию не откатит изменения в БД.

Аннотация `@Transactional` поддерживает `rollbackFor`/`noRollbackFor`:

```java
@Transactional(rollbackFor = Exception.class) // откатываем даже на checked
public void importFile(Path path) throws Exception {
  // ... если здесь вылетит IOException, будет rollback
}
```

Ещё один слой — перевод исключений поставщика JPA/JDBC в иерархию Spring (`DataAccessException`). Это делает `@Repository` + `PersistenceExceptionTranslationPostProcessor`. Вы увидите, например, `DuplicateKeyException` вместо сырого `PSQLException` — удобно ловить бизнес-ошибки (уникальные ключи, FK) типово.

Осторожно с **самовызовом** (`this.methodInside()`): прокси не сработает, и транзакция/rollback не применятся. Выделяйте транзакционные методы в отдельный бин или инжектите self-proxy:

```java
@Service
public class A {
  private final A self;
  public A(A self) { this.self = self; }

  public void outer() { self.innerTx(); }

  @Transactional
  public void innerTx() { /* ... */ }
}
```

---

# Propagation

**`REQUIRED`** — стандарт: использовать текущую транзакцию, если есть, иначе создать новую. **`REQUIRES_NEW`** — всегда открыть новую, приостановив текущую. **`NESTED`** — создать вложенную (savepoint) внутри текущей, если платформа поддерживает (в PostgreSQL savepoint есть, но Spring JPA может опираться на DataSourceTxManager).

Типичный сценарий `REQUIRES_NEW` — аудит/логирование, которое не должно откатываться вместе с основной бизнес-операцией:

```java
@Service
public class AuditService {
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void writeAudit(String event) { /* insert audit */ }
}

@Service
public class OrderService {
  private final AuditService audit;
  // ...
  @Transactional
  public void placeOrder(Order o) {
    // бизнес-логика
    audit.writeAudit("ORDER_CREATED"); // сохранится даже при последующем падении
    // потенциально упадём здесь -> аудит уже зафиксирован
  }
}
```

`NESTED` уместен, когда вы хотите «частичный откат»: если подэтап не удался — откатить его к savepoint, но оставить внешнюю транзакцию живой. Однако убедитесь, что ваш `PlatformTransactionManager` реально поддерживает nested (для JPA чаще — нет; нужен JDBC/`DataSourceTransactionManager`).

Не злоупотребляйте `REQUIRES_NEW`: слишком частые «подтранзакции» увеличивают нагрузку на БД и затрудняют понимание консистентности. Это инструмент для редких, изолированных побочных записей (лог, outbox, метрики).

---

# Isolation

Уровни изоляции управляют феноменами чтения: **dirty read**, **non-repeatable read**, **phantom read**. В PostgreSQL доступны `READ COMMITTED` (по умолчанию), `REPEATABLE READ` и `SERIALIZABLE`. `READ UNCOMMITTED` мапится к `READ COMMITTED` — грязных чтений в Pg нет.

В Spring можно задать изоляцию на метод:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Invoice computeInvoice(long orderId) { ... }
```

`READ COMMITTED` — хороший баланс: каждое чтение видит только зафиксированные данные, но повторные чтения могут возвращать новые версии строк. `REPEATABLE READ` в PostgreSQL предотвращает non-repeatable reads и phantom reads благодаря снимкам (MVCC), но может бросать serialization errors в сложных сценариях. `SERIALIZABLE` добавляет сильную сериализацию — безопасно, но дороже и чаще конфликтует.

Важно помнить: изоляция — не серебряная пуля против логических гонок. Часто правильнее использовать **оптимистическую блокировку** (`@Version` в JPA) или пессимистическую (`SELECT FOR UPDATE`, `@Lock(PESSIMISTIC_WRITE)`), чем завышать уровень изоляции «на всякий случай».

---

# Batch-обработка (batchUpdate)

Для массивных вставок/обновлений `JdbcTemplate.batchUpdate` очень эффективен: один парсинг, один сетевой раунд (или немного), меньше накладных расходов. Для PostgreSQL добавьте в URL `reWriteBatchedInserts=true`.

```java
public int[] saveAll(List<Book> books) {
  return jdbc.batchUpdate("insert into books(title) values(?)",
      new BatchPreparedStatementSetter() {
        public void setValues(PreparedStatement ps, int i) throws SQLException {
          ps.setString(1, books.get(i).getTitle());
        }
        public int getBatchSize() { return books.size(); }
      }
  );
}
```

С именованными параметрами:

```java
public int[] saveAllNamed(List<Book> books) {
  SqlParameterSource[] batch = SqlParameterSourceUtils.createBatch(
      books.stream().map(b -> Map.of("title", b.getTitle())).toArray(Map[]::new));
  return named.batchUpdate("insert into books(title) values(:title)", batch);
}
```

Через JPA тоже можно батчить: включите `hibernate.jdbc.batch_size`, `order_inserts/updates`, используйте `SEQUENCE` и периодически `flush/clear`, чтобы не раздувать контекст:

```java
@Transactional
public void saveMany(List<Book> books) {
  int i = 0;
  for (Book b : books) {
    em.persist(b);
    if (++i % 100 == 0) {
      em.flush();
      em.clear();
    }
  }
}
```

---

# Работа с большим набором данных

Стриминговое чтение через `JdbcTemplate`: важно выставить `fetchSize` и держать транзакцию открытой (для Pg это включает серверный курсор). Иначе драйвер заберёт весь `ResultSet` в память.

```java
@Transactional(readOnly = true)
public void dumpAllBooksTo(OutputStream out) {
  jdbc.setFetchSize(1000);
  jdbc.query("select id, title from books",
      (RowCallbackHandler) rs -> {
        out.write((rs.getLong(1) + "," + rs.getString(2) + "\n").getBytes());
      });
}
```

Через Spring Data JPA можно вернуть `Stream<T>` — он ленивый, но **его нужно закрывать** (лучше try-with-resources) и выполнять в транзакции:

```java
public interface BookRepository extends JpaRepository<Book, Long> {
  @Query("select b from Book b")
  Stream<Book> streamAll();
}

@Transactional(readOnly = true)
public void process() {
  try (Stream<Book> s = repo.streamAll()) {
    s.forEach(this::handle);
  }
}
```

Для особо тяжёлых отчётов проще и надёжнее делать «порционное» чтение по ключу/странице (keyset pagination), чем полагаться на курсоры. Например: «брать по 10k записей, где `id > lastId`, пока не закончится» — это стабильно по памяти и предсказуемо для БД.

Не забывайте про индексы. Любая стратегия «больших чтений» должна опираться на индексируемые фильтры/сортировки (`id`, дата, статус). Без индексов никакой fetch size не спасёт от планов со сканированием таблицы.

---

# Тестирование репозиториев

Для репозиториев и JPA-уровня используйте **`@DataJpaTest`**: он поднимает только JPA-часть (репозитории, EntityManager, ddl), ускоряя тесты. Связка с **Testcontainers** даёт реалистичный PostgreSQL вместо in-memory БД. Обязательно отключайте авто-подмену на H2: `@AutoConfigureTestDatabase(replace = NONE)`.

```java
@Testcontainers
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class BookRepositoryIT {

  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", postgres::getJdbcUrl);
    r.add("spring.datasource.username", postgres::getUsername);
    r.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired BookRepository repo;

  @Test
  void savesAndFinds() {
    Book b = new Book();
    b.setTitle("DDD");
    repo.save(b);

    var found = repo.findByTitleContainingIgnoreCase("dd", Sort.by("id"));
    assertThat(found).extracting(Book::getTitle).contains("DDD");
  }
}
```

Если вы используете миграции (Flyway/Liquibase) — в `@DataJpaTest` они не всегда запускаются автоматически. Включите через свойства или используйте `@SpringBootTest` для полноценных интеграционных сценариев. Альтернатива — подготавливать данные через `@Sql` или `TestEntityManager`.

Для JDBC-слоя пишите отдельные тесты DAO на том же контейнере: это быстрые проверки SQL/маппинга/батчей. В качестве фикстур используйте `@Sql` с seed-данными или `JdbcTemplate` в `@BeforeEach` — так тесты остаются изолированными и воспроизводимыми.

И помните про производительность тестов: один контейнер на весь класс (статический `@Container`), минимальные данные, параллельный запуск при возможности. Интеграционные тесты должны быть «прецизионными», а не «всё покрыть одним махом».

---

# Итог

«Расширенная» часть JPA — это осознанное использование низкоуровневых инструментов Spring: `DataSource`/HikariCP для надёжности, `JdbcTemplate`/SQL для прозрачности и скорости, `RowMapper`/DTO для контроля данных, зрелые транзакции с правильным rollback, propagation и isolation для корректности, батчи и потоковые выборки для масштаба, и, наконец, быстрые и реалистичные тесты с `@DataJpaTest` + Testcontainers.

Применяйте принцип: **JPA — по умолчанию, JDBC — там, где нужно**. Строите репозитории с читабельными запросами и проекциями; включайте батчинг и серверные курсоры, когда объёмы растут; держите транзакции короткими и понятными; изолируйте аудиты/вспомогательные записи через `REQUIRES_NEW`; выбирайте минимальный достаточный уровень изоляции. Тогда ваш код остаётся и быстрым, и предсказуемым, и приятным для ревью.
