---
layout: page
title: "Spring Data JPA: Основы"
permalink: /spring/jpa_basic
---

# 0. Введение

## Что такое JDBC

JDBC (Java Database Connectivity) — это низкоуровневый API из стандартной библиотеки Java, который определяет контракт для работы с реляционными базами данных через драйверы. Он предоставляет интерфейсы `Connection`, `PreparedStatement`, `ResultSet` и др., позволяя выполнять SQL-запросы, управлять транзакциями и получать результаты построчно. JDBC — «общий знаменатель» для всех SQL-СУБД, от Postgres и MySQL до Oracle и MS SQL Server.

Исторически JDBC появился как единый способ подключаться к любым драйверам БД без переписывания кода под конкретную СУБД. Практически это означает: вы пишете SQL руками и вызываете методы API, а объект драйвера переводит вызовы в протокол базы. Такой подход максимально прозрачен и предсказуем — вы точно знаете, какой SQL уходит.

Сильная сторона JDBC — контроль. Вы сами формируете SQL, выбираете индексы, задаёте батчи, управляете курсорами и хинтами. Для «узких мест» и отчётов это часто лучший путь: никакой магии, только ваш SQL и измеримая производительность. Но за контроль приходится платить большим количеством «клеевого» кода.

Слабая сторона JDBC — объём шаблонного кода: открытие соединений, подготовка выражений, настройка параметров, чтение колонок, преобразование типов, закрытие ресурсов. Ошибиться легко, особенно при обработке `null`, дат, чисел и больших коллекций. Поэтому поверх JDBC часто используют утилиты (например, Spring `JdbcTemplate`), но даже с ними остаётся ручной маппинг.

JDBC — не только про запросы, но и про транзакции. Вы включаете `autoCommit=false`, группируете изменения и явно делаете `commit/rollback`. Это мощно, но требует дисциплины: нужно чётко понимать границы транзакций, а в многопоточном коде — избегать утечек соединений.

В экосистеме Java JDBC — фундамент. Любая ORM (например, Hibernate) в итоге строится поверх JDBC и использует драйверы. Это значит: даже если вы выбрали JPA/Hibernate, полезно понимать основы JDBC, чтобы правильно настраивать пулы, таймауты и диагностировать проблемы.

На практике команды нередко комбинируют подходы: JPA для CRUD и бизнес-моделей, `JdbcTemplate`/JDBC — для тяжёлых отчётов, массовых вставок и «узких» SQL. Такое разделение обязанностей помогает соблюдать баланс читаемости кода и производительности.

**Java — минимальный пример JDBC (PostgreSQL, SELECT 1):**

```java
package com.example.intro.jdbc;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class JdbcHello {
    public static void main(String[] args) throws Exception {
        String url = "jdbc:postgresql://localhost:5432/app";
        String user = "app";
        String pass = "app";
        try (Connection cn = DriverManager.getConnection(url, user, pass);
             PreparedStatement ps = cn.prepareStatement("select 1");
             ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                System.out.println("OK: " + rs.getInt(1));
            }
        }
    }
}
```

**Kotlin — тот же пример:**

```kotlin
package com.example.intro.jdbc

import java.sql.DriverManager

fun main() {
    val url = "jdbc:postgresql://localhost:5432/app"
    val user = "app"
    val pass = "app"
    DriverManager.getConnection(url, user, pass).use { cn ->
        cn.prepareStatement("select 1").use { ps ->
            ps.executeQuery().use { rs ->
                if (rs.next()) println("OK: ${rs.getInt(1)}")
            }
        }
    }
}
```

*Gradle (драйвер Postgres)*
*Groovy DSL:*

```groovy
dependencies {
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

---

## Что такое Spring JPA

Spring Data JPA — это надстройка над JPA (Java Persistence API) и её реализациями (чаще всего Hibernate), которая генерирует стандартный CRUD-код за вас. Она предоставляет интерфейсы репозиториев (`JpaRepository`), механизм построения запросов по имени метода, декларативную пагинацию/сортировку и интеграцию с транзакциями Spring.

Внутри Spring Data JPA использует `EntityManager` (ядро JPA) через прокси и управляет «контекстом персистентности» в границах транзакции. Это значит, что загруженные сущности становятся «управляемыми», и изменения их полей отслеживаются автоматически (dirty checking), а при коммите/`flush` преобразуются в SQL.

Ключевая идея Spring Data — декларативность. Вы описываете доменную модель как `@Entity` и интерфейсы репозиториев, а фреймворк создаёт реализацию в рантайме. Это резко снижает количество шаблонного кода и ускоряет разработку: меньше ручных `INSERT/UPDATE/SELECT`, меньше маппинга строк из `ResultSet`.

Spring Data JPA органично интегрирован со всей экосистемой Spring: `@Transactional`, конвертеры типов, валидация, обработка исключений (`DataAccessException`), WebMVC/WebFlux и т. д. Это даёт единый способ думать о данных в приложении: от REST-контроллера до БД.

При этом Spring Data JPA не «запирает» вас: вы можете использовать JPQL, Criteria API и нативные SQL-запросы там, где нужно. Репозитории поддерживают `@Query`, проекции на DTO и «частичные» выборки — хороший компромисс между удобством и производительностью.

Важно понимать, что Spring Data JPA — это слой приложения. Он не отменяет необходимости продуманной схемы БД, индексов, миграций и измерений производительности. Но он помогает сфокусироваться на домене, а не на рутинном доступе к данным.

В большинстве корпоративных сервисов Spring Data JPA — «дефолт» для CRUD и бизнес-логики. Он снижает издержки поддержки, упрощает онбординг разработчиков и создаёт консистентный стиль работы с данными.

**Java — простейшая сущность и репозиторий:**

```java
package com.example.intro.jpa;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.web.bind.annotation.*;

@Entity
@Table(name = "customers")
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    protected Customer() {}
    public Customer(String name) { this.name = name; }
    public Long getId() { return id; }
    public String getName() { return name; }
}

interface CustomerRepository extends JpaRepository<Customer, Long> {
    Customer findByName(String name);
}

@RestController
@RequestMapping("/api/customers")
class CustomerController {
    private final CustomerRepository repo;
    CustomerController(CustomerRepository repo) { this.repo = repo; }

    @PostMapping public Customer create(@RequestBody Customer in) { return repo.save(new Customer(in.getName())); }
    @GetMapping("/{id}") public Customer get(@PathVariable Long id) { return repo.findById(id).orElseThrow(); }
}
```

**Kotlin — аналог (плагины `kotlin-jpa`/`kotlin-spring` рекомендуются):**

```kotlin
package com.example.intro.jpa

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.web.bind.annotation.*

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = ""
)

interface CustomerRepository : JpaRepository<Customer, Long> {
    fun findByName(name: String): Customer?
}

@RestController
@RequestMapping("/api/customers")
class CustomerController(private val repo: CustomerRepository) {
    @PostMapping fun create(@RequestBody inDto: Customer) = repo.save(Customer(name = inDto.name))
    @GetMapping("/{id}") fun get(@PathVariable id: Long) = repo.findById(id).orElseThrow()
}
```

*Gradle (стартеры JPA + Postgres)*
*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

---

## Зачем нужно Spring JPA

Spring JPA позволяет сосредоточиться на предметной области и сценариях использования, а не на технических деталях CRUD. Код становится короче и чище: вместо десятков строк JDBC вы пишете одну строку `repo.save(entity)` или чистую декларацию интерфейса. Это снижает стоимость поддержки и скорость появления новых фич.

Контекст персистентности (Persistence Context) реализует «единицу работы» (Unit of Work): все изменения управляемых сущностей агрегируются в рамках транзакции и в нужный момент «сбрасываются» в БД оптимальной последовательностью SQL. Возникают гарантии идентичности (один объект на один ряд в сессии) и консистентности графов объектов.

Каскадирование и отслеживание изменений снимают рутину. Когда вы добавляете дочерние сущности в коллекцию родителя и сохраняете родителя, ORM сама сформирует INSERT/UPDATE/DELETE согласно конфигурации каскадов. Это особенно полезно в бизнес-моделях с богатой объектной структурой.

Портируемость — ещё один мотив. JPA предоставляет стандартный API, и при аккуратном использовании (без vendor-specific фич) вы сможете переключить провайдер или СУБД с меньшими усилиями. На практике до полного «переключения одним тумблером» далеко, но унификация кода ощутима.

Интеграция с транзакциями Spring упрощает работу с границами консистентности. `@Transactional` атомарно оборачивает операции, объединяя несколько вызовов репозиториев и сервисов в единый ACID-блок. Это снижает риск частично применённых изменений и ошибок при ручном управлении `commit/rollback`.

Наблюдаемость и поддержка: стандартные исключения (`DataAccessException`), конвертеры и хуки позволяют централизованно обрабатывать ошибки, добавлять аудит, включать метрики и логирование SQL. Вы получаете предсказуемый «скелет» вокруг доступа к данным.

При всех плюсах важно понимать границы: Spring JPA — не серебряная пуля. Для отчётных запросов со сложной агрегацией или для массовых ETL часто будет эффективнее нативный SQL или `JdbcTemplate`. Хорошая архитектура допускает сосуществование подходов.

**Java — демонстрация dirty checking под транзакцией:**

```java
package com.example.intro.why;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String owner;
    Long balance;
    protected Account() {}
    Account(String owner, Long balance) { this.owner = owner; this.balance = balance; }
}

interface AccountRepository extends JpaRepository<Account, Long> {}

@Service
class TransferService {
    private final AccountRepository repo;
    TransferService(AccountRepository repo) { this.repo = repo; }

    @Transactional
    public void deposit(Long id, long amount) {
        Account a = repo.findById(id).orElseThrow();
        a.balance += amount; // dirty checking: UPDATE выполнится автоматически при коммите
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.intro.why

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
class Account(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var owner: String = "",
    var balance: Long = 0
)

interface AccountRepository : JpaRepository<Account, Long>

@Service
class TransferService(private val repo: AccountRepository) {
    @Transactional
    fun deposit(id: Long, amount: Long) {
        val a = repo.findById(id).orElseThrow()
        a.balance += amount // dirty checking
    }
}
```

---

## Где используется

Spring Data JPA уместен в сервисах, где доменная модель не тривиальна: от CRUD-форм до сложных агрегатов и инвариантов. Это подавляющее большинство внутренних бизнес-приложений: биллинг, заказы, учёт, справочники, каталоги, право доступа. Там ценится читаемость, предсказуемость и скорость разработки.

В микросервисной архитектуре JPA часто используется в каждом сервисе, у которого есть собственная БД (паттерн Database per Service). Репозитории инкапсулируют доступ к данным, а внешние контракты — REST/gRPC/события — не протекают внутрь. Это облегчает эволюцию схемы и фич-флагов.

В связке со Spring MVC/WebFlux JPA формирует «ядро» слоя данных. Контроллеры получают/отдают DTO, сервисы управляют транзакциями и доменными инвариантами, а репозитории — доступ к БД. Такая стратификация хорошо тестируется и масштабируется командно.

Для Postgres JPA — комфортный выбор: типы соответствуют ожиданиям, драйвер зрелый, транзакционность надёжная. Для Oracle и MS SQL картина аналогична, но больше нюансов с диалектами и функциями — важно придерживаться «чистого» JPQL и аккуратных нативных запросов.

В e-commerce и финтехе JPA часто сочетают с очередями (Kafka) и асинхронными обработчиками: транзакция обрамляет изменение состояния и публикацию события (через outbox/relay). Тесная интеграция Spring + JPA помогает удерживать согласованность.

JPA применяется и для read-моделей, но осторожно. Для высоконагруженных, сложных чтений иногда выбор падает на специализированные решения (материализованные представления, отдельные денормализованные таблицы, Elastic/ClickHouse). Важно трезво оценивать, что вы строите.

Для прототипов и внутренних админок JPA даёт «скорость света». Создаёте сущности, репозитории, контроллер — и уже можно работать. Главное — не забыть позже навести порядок: миграции, индексы, пагинация, ограничения.

**`application.yml` — базовая конфигурация Postgres + JPA:**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app
    username: app
    password: app
  jpa:
    hibernate:
      ddl-auto: validate   # для продакшна: validate/none; create/update только для dev
    properties:
      hibernate:
        format_sql: true
        jdbc:
          time_zone: UTC
  sql:
    init:
      mode: never
```

**Docker Compose для локальной БД:**

```yaml
version: '3.9'
services:
  pg:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 20
```

**Java — минимальный REST поверх JPA:**

```java
package com.example.intro.where;

import org.springframework.web.bind.annotation.*;
import org.springframework.data.jpa.repository.JpaRepository;
import jakarta.persistence.*;

@Entity class Product { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id; String name; protected Product(){} public Product(String n){name=n;} }
interface ProductRepo extends JpaRepository<Product, Long> { }
@RestController @RequestMapping("/api/products")
class ProductApi {
    private final ProductRepo repo;
    ProductApi(ProductRepo repo){this.repo=repo;}
    @PostMapping public Product create(@RequestBody Product p){ return repo.save(new Product(p.name)); }
    @GetMapping public java.util.List<Product> list(){ return repo.findAll(); }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.intro.where

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.web.bind.annotation.*

@Entity
class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = ""
)

interface ProductRepo : JpaRepository<Product, Long>

@RestController
@RequestMapping("/api/products")
class ProductApi(private val repo: ProductRepo) {
    @PostMapping fun create(@RequestBody p: Product) = repo.save(Product(name = p.name))
    @GetMapping fun list(): List<Product> = repo.findAll()
}
```

---

## Какие проблемы решает

Первая проблема, которую решает JPA, — «болезнь маппинга». Ручное извлечение колонок из `ResultSet` и трансляция в объекты чревата ошибками и дублированием. Аннотации `@Entity`, `@Column`, `@Embedded` формализуют соответствие «колонка ↔ поле», а провайдер берёт на себя сериализацию/десериализацию.

Вторая — управление жизненным циклом и транзакциями. Вместо ручного `autoCommit=false` и `commit/rollback` вы объявляете `@Transactional`, а `EntityManager` сам координирует изменения, каскады и очередность SQL. Это уменьшает когнитивную нагрузку и количество «забытых» коммитов.

Третья — консистентность графов объектов. В ORM есть понятие идентичности (одна строка таблицы — один объект в контексте), и изменения разносятся по графу корректно. Это снимает класс ошибок, когда два разных экземпляра «одной и той же» сущности конфликтуют между собой.

Четвёртая — портируемость и стандартизация. JPA — спецификация, и поверх неё можно менять провайдера (Hibernate, EclipseLink), держать общий стиль кода и переиспользовать знания между проектами. Это снижает риски «vendor lock-in» в части кода приложения.

Пятая — расширяемость типов. Конвертеры (`@Converter`) позволяют хранить сложные value-объекты (Money, JSON, списки) в колонках, не засоряя код руками парсерами. Это особенно удобно с Postgres, где есть `jsonb`, `uuid`, диапазоны — а конвертеры дают типобезопасную работу в Java/Kotlin.

Шестая — интеграция с остальными слоями. Валидация (Jakarta Validation), события, слушатели, аудит (created/modified) — всё это есть «из коробки» или включается малой кровью. В JDBC это пришлось бы писать заново.

Седьмая — тестируемость. С `@DataJpaTest` можно поднимать срез persistence-слоя быстро и воспроизводимо, а `TestEntityManager` облегчает подготовку данных. Это формирует культуру покрывать доступ к данным тестами, что уменьшает число «тихих» регрессий.

**Java — встраиваемые типы и конвертер JSON:**

```java
package com.example.intro.problems;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.*;

import java.io.IOException;
import java.util.Map;

@Embeddable
class Address {
    public String city;
    public String street;
    protected Address() {}
    public Address(String city, String street) { this.city = city; this.street = street; }
}

@Converter(autoApply = true)
class JsonMapConverter implements AttributeConverter<Map<String, Object>, String> {
    private static final ObjectMapper om = new ObjectMapper();
    @Override public String convertToDatabaseColumn(Map<String, Object> attribute) {
        try { return attribute == null ? null : om.writeValueAsString(attribute); }
        catch (JsonProcessingException e) { throw new IllegalArgumentException(e); }
    }
    @Override public Map<String, Object> convertToEntityAttribute(String dbData) {
        try { return dbData == null ? null : om.readValue(dbData, Map.class); }
        catch (IOException e) { throw new IllegalArgumentException(e); }
    }
}

@Entity
class Warehouse {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Embedded Address address;
    @Column(columnDefinition = "text") Map<String, Object> meta;
}
```

**Kotlin — аналог:**

```kotlin
package com.example.intro.problems

import com.fasterxml.jackson.databind.ObjectMapper
import jakarta.persistence.*
import java.lang.IllegalArgumentException

@Embeddable
class Address(
    var city: String = "",
    var street: String = ""
)

@Converter(autoApply = true)
class JsonMapConverter : AttributeConverter<Map<String, Any>?, String?> {
    companion object { private val om = ObjectMapper() }
    override fun convertToDatabaseColumn(attribute: Map<String, Any>?): String? =
        attribute?.let { om.writeValueAsString(it) }

    override fun convertToEntityAttribute(dbData: String?): Map<String, Any>? =
        dbData?.let { om.readValue(it, Map::class.java) as Map<String, Any> }
}

@Entity
class Warehouse(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Embedded
    var address: Address? = null,
    @Column(columnDefinition = "text")
    @Convert(converter = JsonMapConverter::class)
    var meta: Map<String, Any>? = null
)
```

---

## Что выбирать: JDBC или JPA?

Выбор — это компромисс между скоростью разработки, контролем и производительностью. Если доменная модель богата, а большая часть операций — CRUD, то JPA/Spring Data даст лучшую читаемость и скорость поставки. Если же у вас доминируют «тяжёлые» отчёты, агрегаты, window-функции и сложные join-ы, нативный SQL/`JdbcTemplate` будет честнее и быстрее.

Правильный ответ часто: **и то, и другое**. Используйте JPA для бизнес-сущностей и инвариантов, но не стесняйтесь опускаться до JDBC там, где это оправдано. Spring прекрасно поддерживает оба слоя в одном приложении: `spring-boot-starter-data-jpa` и `spring-boot-starter-jdbc` не конфликтуют.

Важно установить правила. В команде должно быть понятно, когда «можно/нужно» писать `@Query(nativeQuery=true)` или использовать `JdbcTemplate`. Хорошая метрика — доля кода, где JPA «ломается»: если таких мест много, возможно, JPA не ваш базовый подход для конкретного сервиса.

Не забывайте о профилировании и планах выполнения. Даже «простой» код репозиториев может генерировать неудачный SQL. Включайте логирование SQL, анализируйте планы (`EXPLAIN ANALYZE`), работайте с индексами. Инструмент — не замена инженерному мышлению.

С точки зрения операционного сопровождения JDBC иногда даёт более явные точки контроля: вы видите конкретный SQL, батчи, курсоры. Но с ним больше ответственности: ручное управление ресурсами, обработка исключений, восстановление после ошибок.

Композиция подходов позволяет строить «двухскоростной» слой данных: 80% — удобный, безопасный, декларативный JPA; 20% — узкоспециализированный высокоэффективный SQL. Это уменьшает суммарные риски и даёт лучшую TTM.

Наконец, учитывайте требования переносимости. Если продукт планирует поддержку нескольких СУБД, избегайте vendor-specific SQL и расширений везде, где это разумно. JPA и JPQL здесь помогают, но не отменяют необходимости тестировать «на настоящей БД».

**Java — совместное использование JPA и JdbcTemplate:**

```java
package com.example.intro.choice;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Entity
class Item { @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id; String name; Long price; }

interface ItemRepository extends JpaRepository<Item, Long> { }

@Repository
class ReportDao {
    private final NamedParameterJdbcTemplate jdbc;
    ReportDao(NamedParameterJdbcTemplate jdbc) { this.jdbc = jdbc; }
    List<Map<String,Object>> topItems(int limit) {
        String sql = "select name, sum(price) as total from item group by name order by total desc limit :lim";
        return jdbc.queryForList(sql, Map.of("lim", limit));
    }
}

@Service
class CatalogService {
    private final ItemRepository repo;
    private final ReportDao report;
    CatalogService(ItemRepository repo, ReportDao report) { this.repo = repo; this.report = report; }
    public Item add(String name, long price) { return repo.save(new Item(){ { this.name=name; this.price=price; } }); }
    public List<Map<String,Object>> top(int n) { return report.topItems(n); }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.intro.choice

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service

@Entity
class Item(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = "",
    var price: Long = 0
)

interface ItemRepository : JpaRepository<Item, Long>

@Repository
class ReportDao(private val jdbc: NamedParameterJdbcTemplate) {
    fun topItems(limit: Int): List<Map<String, Any>> =
        jdbc.queryForList(
            "select name, sum(price) as total from item group by name order by total desc limit :lim",
            mapOf("lim" to limit)
        )
}

@Service
class CatalogService(private val repo: ItemRepository, private val report: ReportDao) {
    fun add(name: String, price: Long): Item = repo.save(Item(name = name, price = price))
    fun top(n: Int): List<Map<String, Any>> = report.topItems(n)
}
```

*Gradle — подключаем оба стартера*
*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-jdbc")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```
# 1. Зачем JPA поверх «ручного» JDBC

## Слой абстракции над SQL и маппингом объектов, единый API для разных СУБД, меньше «клеевого» кода

JPA даёт слой абстракции над JDBC: вы моделируете домен через `@Entity`, а фреймворк берёт на себя генерацию SQL, маппинг колонок к полям и управление жизненным циклом объектов. Это резко уменьшает количество «клеевого» кода: меньше шаблонных `try-with-resources`, ручных `setString`/`getLong`, циклов по `ResultSet`. В результате вы фокусируетесь на бизнес-логике.

Единый API JPA работает поверх разных СУБД. На одном и том же коде `repo.save()` вы можете сменить Postgres на MySQL, не переписывая слой доступа к данным. Диалект Hibernate адаптирует нюансы синтаксиса. Да, «магии» не стоит злоупотреблять, но в 80% CRUD-кейсов переносимость действительно работает.

Снижение количества кода — не только про скорость разработки, но и про надёжность. Чем меньше ручной работы с JDBC, тем меньше рисков утечек соединений, незакрытых курсоров, ошибок типов при чтении/записи. Spring Data JPA инкапсулирует все эти рутинные аспекты.

Абстракция не отменяет контроля. В любой момент можно спуститься на JPQL или даже на нативный SQL через `@Query(nativeQuery = true)` либо `EntityManager#createNativeQuery`. Это позволяет комбинировать удобство и точечную оптимизацию там, где это необходимо.

Модель «сущность ↔ таблица» делает код самодокументируемым. В аннотациях видны ограничения (`nullable`, `unique`), длины, типы. Эти же декларации служат подсказкой для миграций и тестов, сокращая рассинхронизацию между схемой и кодом.

Spring Data репозитории встраиваются в архитектуру слоями: контроллеры → сервисы (бизнес-правила) → репозитории (доступ к данным). Это дисциплинирует кодовую базу и облегчает тестирование, особенно в больших командах.

Наконец, абстракция помогает on-boarding’у. Новичку проще понять `CustomerRepository#findByEmail` и `@Entity Customer`, чем пачку SQL-фрагментов и ручной маппинг. Это снижает когнитивную нагрузку и стоимость сопровождения.

**Gradle (единые зависимости для примеров этого подпункта)**
*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**Java — «как было бы» на JdbcTemplate и «как стало» на JPA:**

```java
package com.example.jpa.vs.jdbc;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.web.bind.annotation.*;
import jakarta.persistence.*;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

// JDBC вручную
@RestController
@RequestMapping("/api/jdbc/customers")
class CustomerJdbcApi {
    private final JdbcTemplate jdbc;
    CustomerJdbcApi(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @GetMapping
    List<CustomerDto> all() {
        return jdbc.query("select id, name, email from customers", new RowMapper<>() {
            @Override public CustomerDto mapRow(ResultSet rs, int rowNum) throws SQLException {
                return new CustomerDto(rs.getLong("id"), rs.getString("name"), rs.getString("email"));
            }
        });
    }

    record CustomerDto(Long id, String name, String email) {}
}

// JPA/Repositories
@Entity @Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String name; String email;
    protected Customer() {}
    Customer(String name, String email){ this.name = name; this.email = email; }
}
interface CustomerRepository extends JpaRepository<Customer, Long> {
    List<Customer> findByEmailContainingIgnoreCase(String fragment);
}
@RestController
@RequestMapping("/api/jpa/customers")
class CustomerJpaApi {
    private final CustomerRepository repo;
    CustomerJpaApi(CustomerRepository repo){ this.repo = repo; }

    @GetMapping public List<Customer> all(){ return repo.findAll(); }
    @GetMapping("/search") public List<Customer> search(@RequestParam String q){
        return repo.findByEmailContainingIgnoreCase(q);
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.vs.jdbc

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/jdbc/customers")
class CustomerJdbcApi(private val jdbc: JdbcTemplate) {
    data class CustomerDto(val id: Long, val name: String, val email: String)
    @GetMapping
    fun all(): List<CustomerDto> =
        jdbc.query("select id, name, email from customers") { rs, _ ->
            CustomerDto(rs.getLong("id"), rs.getString("name"), rs.getString("email"))
        }
}

@Entity @Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = "",
    var email: String = ""
)

interface CustomerRepository : JpaRepository<Customer, Long> {
    fun findByEmailContainingIgnoreCase(fragment: String): List<Customer>
}

@RestController
@RequestMapping("/api/jpa/customers")
class CustomerJpaApi(private val repo: CustomerRepository) {
    @GetMapping fun all(): List<Customer> = repo.findAll()
    @GetMapping("/search") fun search(@RequestParam q: String) = repo.findByEmailContainingIgnoreCase(q)
}
```

---

## Persistence Context и Unit of Work: автоматическое отслеживание изменений, каскадные операции

Контекст персистентности (Persistence Context) — это «первый уровень кеша» и **identity map**: в рамках транзакции каждая строка таблицы представлена единственным Java-объектом. Это гарантирует идентичность и упрощает логику — сравнение по ссылке отражает сравнение по ключу в БД.

Unit of Work — шаблон, в котором изменения объектов копятся и «снимаются» в БД одной пачкой при `flush`/commit. JPA реализует его через dirty checking: вы изменили поле управляемой сущности — Hibernate заметил дифф и при коммите сгенерирует `UPDATE`. Не нужно писать `UPDATE` вручную для каждого поля.

Каскады (`CascadeType`) позволяют транслировать операции с корневой сущностью на связанные объекты. Например, `CascadeType.PERSIST` на `@OneToMany` вызовет `INSERT` для детей при сохранении родителя; `CascadeType.REMOVE` — удалит их при удалении родителя. Это безопасно, когда коллекция — действительно «владение» (aggregate).

Важно помнить, что каскад — не магия, а инструмент. Если связь — **не** владение (например, ссылка на справочник), каскады на `REMOVE` могут удалить «общий» справочник — этого чаще не хочется. В таких случаях каскад ограничивают `PERSIST`/`MERGE` или вовсе отключают.

Persistence Context отслеживает только **управляемые** объекты — те, что получены через `find`/`getReference`/загружены запросом, либо сохранены через `persist`/`save`. Объекты, созданные снаружи и не «привязанные» к контексту, называются `detached`; их изменения не попадут в БД без `merge`.

Автоматический `flush` происходит при коммите транзакции и в некоторых точках исполнения запросов (при `FlushMode.AUTO`). Это значит: сначала все накопленные изменения переводятся в SQL, затем выполняется следующий `SELECT`. Неверное понимание этих правил может приводить к неожиданной нагрузке; тонкую настройку обсудим позже.

С точки зрения архитектуры Persistence Context живёт в границах `@Transactional`. За пределами транзакции ленивые связи не инициализируются, а изменения не фиксируются. Это дисциплинирует границы бизнес-операций: одна транзакция — одна единица работы.

**Java — каскадные операции и dirty checking в сервисе:**

```java
package com.example.jpa.uow;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.List;

@Entity
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String customer;
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    List<OrderLine> lines = new ArrayList<>();
    protected Order() {}
    Order(String customer){ this.customer = customer; }
    void addLine(String sku, long qty) {
        OrderLine line = new OrderLine(this, sku, qty);
        lines.add(line);
    }
}

@Entity
class OrderLine {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "order_id")
    Order order;
    String sku; long qty;
    protected OrderLine() {}
    OrderLine(Order order, String sku, long qty){ this.order = order; this.sku = sku; this.qty = qty; }
}

interface OrderRepository extends JpaRepository<Order, Long> { }

@Service
class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo){ this.repo = repo; }

    @Transactional
    public Long create(String customer) {
        Order o = new Order(customer);
        o.addLine("A-1", 2);
        o.addLine("B-2", 1);
        return repo.save(o).id; // каскад PERSIST сохранит и строки
    }

    @Transactional
    public void increaseFirstLine(Long orderId) {
        Order o = repo.findById(orderId).orElseThrow();
        o.lines.get(0).qty += 1; // dirty checking => UPDATE order_line set qty=...
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.uow

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var customer: String = "",
    @OneToMany(mappedBy = "order", cascade = [CascadeType.ALL], orphanRemoval = true)
    var lines: MutableList<OrderLine> = mutableListOf()
) {
    fun addLine(sku: String, qty: Long) {
        val line = OrderLine(order = this, sku = sku, qty = qty)
        lines.add(line)
    }
}

@Entity
class OrderLine(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "order_id")
    var order: Order? = null,
    var sku: String = "",
    var qty: Long = 0
)

interface OrderRepository : JpaRepository<Order, Long>

@Service
class OrderService(private val repo: OrderRepository) {

    @Transactional
    fun create(customer: String): Long {
        val o = Order(customer = customer)
        o.addLine("A-1", 2)
        o.addLine("B-2", 1)
        return repo.save(o).id!!
    }

    @Transactional
    fun increaseFirstLine(orderId: Long) {
        val o = repo.findById(orderId).orElseThrow()
        o.lines[0].qty += 1 // dirty checking
    }
}
```

---

## Где JPA уместна/неуместна: CRUD и бизнес-модели vs тяжёлые отчёты/массовые ETL (об этом — в следующей теме)

JPA блестяще справляется с бизнес-моделями: создание/обновление агрегатов, инварианты, каскады, транзакции. Когда сущности отражают реальные объекты предметной области, ORM экономит сотни строк кода и делает его выразительным. Особенно ценна идентичность: один объект на одну строку в рамках транзакции.

Для чтений типа «получить карточку сущности с несколькими связанными объектами» JPA даёт JPQL/`JOIN FETCH`, проекции и пагинацию. Это закрывает 80% UI/CRUD задач, не ломая переносимость между СУБД. В таких сценариях нет смысла вручную собирать SQL и маппить DTO.

Где JPA начинает «скрипеть» — отчёты с тяжёлыми агрегациями, оконными функциями, CTE и специфичными оптимизациями под конкретную СУБД. Такие запросы удобнее и эффективнее писать нативным SQL и выполнять через `JdbcTemplate`/`EntityManager#createNativeQuery`, возвращая DTO/`Map`. Это нормально: не надо пытаться «упихать» любой отчёт в ORM.

Массовые ETL/батчи — отдельная история. JPA поддерживает batch inserts/updates, но при больших объёмах быстрее и дешевле по памяти может быть `COPY`/bulk API драйвера, особенно в Postgres. Выгрузки/загрузки через чистый JDBC-соединитель без контекста персистентности часто выигрывают.

Чтение огромных выборок через JPA требует стриминга/скролла и аккуратной работы с контекстом (`clear()`), иначе вы упираетесь в рост памяти из-за identity map. Для чистых отчётов проще маппить «плоские» строки сразу в DTO, не проводя их через сущности.

Отдельный сигнал — когда вы делаете «read-only» сервис, который не изменяет данные, а только агрегирует их из нескольких источников. Там JPA даёт мало преимуществ; `JdbcTemplate` + ручные мапперы будут проще, предсказуемее и быстрее.

Ключ — прагматичность. В одном сервисе можно иметь JPA для домена и `JdbcTemplate` для отчётов. Главное — держать чистую границу и общий договор по форматам и транзакциям. Не бойтесь миксовать инструменты.

**Java — сочетание JPA для домена и JDBC для отчёта:**

```java
package com.example.jpa.mixed;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Entity @Table(name = "sales")
class Sale {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String region; long amount;
    protected Sale() {}
    Sale(String region, long amount){ this.region = region; this.amount = amount; }
}
interface SaleRepository extends JpaRepository<Sale, Long> { }

@Repository
class ReportsDao {
    private final NamedParameterJdbcTemplate jdbc;
    ReportsDao(NamedParameterJdbcTemplate jdbc){ this.jdbc = jdbc; }
    List<Map<String,Object>> totalsByRegion() {
        String sql = """
          select region, sum(amount) total
          from sales group by region order by total desc
        """;
        return jdbc.queryForList(sql, Map.of());
    }
}

@Service
class SalesService {
    private final SaleRepository repo; private final ReportsDao reports;
    SalesService(SaleRepository repo, ReportsDao reports){ this.repo = repo; this.reports = reports; }
    public Sale add(String region, long amount){ return repo.save(new Sale(region, amount)); }
    public List<Map<String,Object>> report(){ return reports.totalsByRegion(); }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.mixed

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service

@Entity
@Table(name = "sales")
class Sale(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var region: String = "",
    var amount: Long = 0
)
interface SaleRepository : JpaRepository<Sale, Long>

@Repository
class ReportsDao(private val jdbc: NamedParameterJdbcTemplate) {
    fun totalsByRegion(): List<Map<String, Any>> =
        jdbc.queryForList(
            """
            select region, sum(amount) total
            from sales group by region order by total desc
            """.trimIndent(),
            emptyMap<String, Any>()
        )
}

@Service
class SalesService(private val repo: SaleRepository, private val reports: ReportsDao) {
    fun add(region: String, amount: Long): Sale = repo.save(Sale(region = region, amount = amount))
    fun report(): List<Map<String, Any>> = reports.totalsByRegion()
}
```

---

# 2. Архитектура JPA и роль EntityManager

## `EntityManager` как фасад к Persistence Context (1st-level cache/identity map)

`EntityManager` — центральный объект JPA, фасад к контексту персистентности. Через него выполняются операции загрузки, сохранения, удаления, создание запросов (JPQL/Criteria), управление `flush` и жизненным циклом сущностей. В Hibernate под капотом это обёртка вокруг `Session`.

Контекст персистентности — это первый уровень кеша, связанный с `EntityManager`. Он гарантирует identity map: в пределах одного контекста один и тот же ряд таблицы имеет один Java-объект. Это избавляет от «двойников» и неконсистентности в пределах транзакции.

Когда вы вызываете `em.find(User.class, id)`, сущность становится управляемой (managed). Повторный `find` того же id вернёт **тот же** объект из кеша, даже без похода в БД (пока не было `clear`). Это снижает количество запросов и упрощает логику.

`EntityManager` умеет создавать «ленивые прокси» (`getReference`). Это «пустышки» с id, которые инициализируются при первом доступе к полям. Такой приём уменьшает количество загрузок и откладывает их до реальной необходимости.

Через `EntityManager` вы управляете жизненным циклом: `persist` делает объект управляемым и планирует `INSERT`; `remove` — помечает на удаление; `merge` — копирует состояние из detached-объекта в управляемый. Это и есть Unit of Work в действии.

Важно понимать границы: один `EntityManager` — один контекст. В типичном Spring-приложении он ассоциирован с текущей транзакцией/потоком запроса. Это значит, что логика «по умолчанию» безопасна: каждый HTTP-запрос живёт в собственном контексте.

`EntityManager` — это не пул соединений. Он использует соединение, предоставляемое из пула (HikariCP), когда нужно выполнить SQL (на `flush`/запросе). Управлять пулом и таймаутами — задача DataSource, а не EM.

**Java — явная работа с `EntityManager`:**

```java
package com.example.jpa.em;

import jakarta.persistence.*;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

@Repository
class UserEmDao {
    @PersistenceContext
    private EntityManager em;

    @Transactional
    public Long create(String name) {
        User u = new User(name);
        em.persist(u);      // планируем INSERT
        return u.id;        // id заполнится после flush
    }

    @Transactional(readOnly = true)
    public User get(Long id) {
        return em.find(User.class, id); // возьмём из PC или загрузим
    }
}

@Entity @Table(name = "users")
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String name;
    protected User(){}
    User(String name){ this.name = name; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.em

import jakarta.persistence.*
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional

@Repository
class UserEmDao(
    @PersistenceContext private val em: EntityManager
) {
    @Transactional
    fun create(name: String): Long {
        val u = User(name = name)
        em.persist(u)
        return u.id!!
    }

    @Transactional(readOnly = true)
    fun get(id: Long): User = em.find(User::class.java, id)
}

@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = ""
)
```

---

## Операции: `persist/merge/find/remove`, `flush/clear`, выборочно — `refresh`

`persist` переводит transient-объект в managed и планирует `INSERT`. До фактического `flush`/commit запись в БД не сделана, но объект уже «живёт» в контексте. Поле id при `IDENTITY`-генерации заполняется сразу на вставке (требуется SQL), при SEQUENCE — может быть получено заранее.

`merge` — это «склейка»: вы передаёте detached-объект, а JPA ищет (или создаёт) managed-копию и копирует в неё состояние. Возвращаемое значение — **новый** управляемый объект; исходный остаётся detached. Ошибка новичков — менять исходный detached после `merge` и ожидать сохранения.

`find` загружает и/или берёт из PC сущность по ключу. Если объект уже в PC, запроса в БД не будет. `getReference` возвращает ленивый прокси и может бросить `EntityNotFoundException` при первой инициализации, если строки нет.

`remove` помечает управляемую сущность на удаление. Удаление произойдёт при `flush`. Если у вас настроен `orphanRemoval=true` на коллекциях, удаление ребёнка из коллекции тоже приведёт к `DELETE` для него.

`flush` синхронизирует PC с БД: все pending `INSERT/UPDATE/DELETE` превращаются в SQL в правильном порядке. Обычно `flush` происходит автоматически (commit/выполнение запроса), но иногда уместен явный `em.flush()` — например, чтобы гарантировать уникальность до дальнейших шагов.

`clear` очищает контекст: все сущности становятся detached. Это полезно в батч-процессинге, чтобы не накапливать тысячи объектов в памяти. После `clear` любые изменения объектов не будут сохранены без повторной загрузки/`merge`.

`refresh` перезагружает состояние сущности из БД, сбрасывая локальные несохранённые изменения. Это редкая, но полезная операция, когда вы точно хотите «истину» с диска (например, после внешнего обновления той же строки).

**Java — демонстрация ключевых операций:**

```java
package com.example.jpa.ops;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
class OpsService {
    @PersistenceContext EntityManager em;

    @Transactional
    public void demo() {
        // persist
        Product p = new Product("Pen", 100);
        em.persist(p); // INSERT pending
        em.flush();    // INSERT now

        // find and dirty change
        Product found = em.find(Product.class, p.id);
        found.price = 120;  // UPDATE pending

        // clear & merge
        em.clear();         // found -> detached
        found.price = 130;  // detached change (не сохранится)
        Product managedAgain = em.merge(found); // копируем состояние, UPDATE pending

        // refresh: вернуть данные из БД
        em.refresh(managedAgain);

        // remove
        em.remove(managedAgain); // DELETE pending
    }
}

@Entity @Table(name = "products")
class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String name; int price;
    protected Product(){}
    Product(String name, int price){ this.name = name; this.price = price; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.ops

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OpsService(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun demo() {
        val p = Product(name = "Pen", price = 100)
        em.persist(p)
        em.flush()

        val found = em.find(Product::class.java, p.id)
        found.price = 120

        em.clear()
        found.price = 130 // detached
        val managedAgain = em.merge(found)

        em.refresh(managedAgain)
        em.remove(managedAgain)
    }
}

@Entity
@Table(name = "products")
class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = "",
    var price: Int = 0
)
```

---

## Как это в Spring: `@Transactional`, прокси EM на поток, перевод исключений в `DataAccessException`

В Spring `EntityManager` управляется контейнером: `@PersistenceContext` инжектирует **прокси**, который связывается с реальным `EntityManager` для текущего потока/транзакции. Это называется «контейнер-управляемый» EM (container-managed). Вам не нужно открывать/закрывать его вручную.

Аннотация `@Transactional` определяет границы бизнес-операции. Для методов чтения используйте `readOnly = true`: это подсказывает провайдеру, что `flush` не нужен, а некоторые оптимизации (например, хинт read-only транзакции) могут быть включены. Для модификаций — `@Transactional` без `readOnly`.

Spring переводит исключения JPA/Hibernate в иерархию `DataAccessException` (через `@Repository`/переводчик исключений). Это унифицирует обработку на верхних слоях. Например, `EntityNotFoundException`/`NoResultException` будет превращён в `EmptyResultDataAccessException`.

Transaction synchronization: при входе в `@Transactional` Spring открывает транзакцию на соединении; при выходе — делает commit/rollback. `flush` JPA происходит до коммита, чтобы SQL успел выполниться. Это означает, что «видимость» изменений в рамках текущего EM выше, чем в БД до `flush`.

Прокси `EntityManager` привязан к текущему потоку. Если вы уходите в `@Async`, там уже другой поток и другой контекст. В таких сценариях отдельно планируйте транзакции и понимайте, что «протащить» EM в другой поток нельзя — только открыть новый.

Комбинация `@Transactional` на сервисах и репозиториев Spring Data JPA — проверенный паттерн. Репозитории обычно **не** аннотируют `@Transactional` (кроме, возможно, `@Transactional(readOnly = true)` по умолчанию), а сервисы определяют границы бизнеса, внутри которых вызываются несколько репозиториев.

Важно: по умолчанию Spring откатывает транзакцию на **unchecked** исключениях (наследниках `RuntimeException`). Если вы бросаете checked-исключение и хотите откат — добавьте `rollbackFor = Exception.class` или оборачивайте в runtime.

**Java — сервисные границы, перевод исключений и уровни транзакций:**

```java
package com.example.jpa.spring;

import org.springframework.dao.DataAccessException;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import jakarta.persistence.*;

import java.util.Optional;

@Entity
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String email; String name;
    protected Customer(){}
    Customer(String email, String name){ this.email=email; this.name=name; }
}

@Repository
interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByEmail(String email);
}

@Service
class CustomerService {
    private final CustomerRepository repo;
    CustomerService(CustomerRepository repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public Customer getOrThrow(Long id) {
        return repo.findById(id).orElseThrow(() -> new EntityNotFoundException("Customer " + id + " missing"));
    }

    @Transactional
    public Long register(String email, String name) {
        repo.findByEmail(email).ifPresent(c -> { throw new IllegalStateException("Email exists"); });
        Customer c = new Customer(email, name);
        return repo.save(c).id;
    }

    public Customer tryFind(Long id) {
        try {
            return getOrThrow(id);
        } catch (DataAccessException | EntityNotFoundException e) {
            // унифицированная обработка ошибок доступа к данным
            throw e;
        }
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.spring

import jakarta.persistence.*
import org.springframework.dao.DataAccessException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var email: String = "",
    var name: String = ""
)

@Repository
interface CustomerRepository : JpaRepository<Customer, Long> {
    fun findByEmail(email: String): java.util.Optional<Customer>
}

@Service
class CustomerService(private val repo: CustomerRepository) {

    @Transactional(readOnly = true)
    fun getOrThrow(id: Long): Customer =
        repo.findById(id).orElseThrow { EntityNotFoundException("Customer $id missing") }

    @Transactional
    fun register(email: String, name: String): Long {
        repo.findByEmail(email).ifPresent { throw IllegalStateException("Email exists") }
        val c = Customer(email = email, name = name)
        return repo.save(c).id!!
    }

    fun tryFind(id: Long): Customer =
        try { getOrThrow(id) }
        catch (e: DataAccessException) { throw e }
        catch (e: EntityNotFoundException) { throw e }
}
```

**`application.yml` — включим перевод исключений и базовые настройки JPA (достаточно стартера):**

```yaml
spring:
  jpa:
    open-in-view: false   # рекомендуем выключать, чтобы не было утечек контекста за пределы транзакции
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
```
# 3. Сущности и базовые маппинги

## `@Entity/@Table`, поле первичного ключа `@Id`, стратегии `@GeneratedValue` (обзор без тюнинга: `IDENTITY/SEQUENCE/TABLE/UUID`)

Аннотация `@Entity` помечает класс как управляемый JPA-движком (например, Hibernate). Такой класс должен иметь открытый или защищённый конструктор без аргументов (Hibernate использует его через рефлексию), а также идентификатор — поле с `@Id`. По соглашению имя таблицы берётся из имени класса, но надёжнее явно указать его через `@Table(name = "...")`, чтобы не зависеть от стратегий именования.

Поле `@Id` задаёт первичный ключ. Для большинства доменных моделей практичен тип `Long`/`UUID`. Важно: **не** используйте примитивы (`long`) — `null`-значение полезно, чтобы различать “новые” и “уже сохранённые” экземпляры. Для Kotlin это обычно `var id: Long? = null` или `var id: UUID? = null`.

`@GeneratedValue` определяет стратегию генерации ключей. `IDENTITY` перекладывает генерацию на БД (auto-increment/serial). Плюс — простота; минус — Hibernate вынужден выполнять `INSERT` сразу, чтобы получить id, что мешает оптимизациям батча и может приводить к лишним `flush`-ам. В Postgres `IDENTITY`/`SERIAL` — привычная дорожка, но не всегда оптимальная.

`SEQUENCE` использует серверные последовательности (`CREATE SEQUENCE`). Hibernate может получать id заранее, до фактической вставки, что дружит с батч-вставками и часто даёт лучшую производительность. Требует объявления `@SequenceGenerator` (минимально) и уместен в СУБД с хорошей поддержкой последовательностей (Postgres, Oracle).

`TABLE` хранит счётчик в специальной таблице — историческая стратегия для БД без последовательностей. Сегодня почти не используется из-за накладных расходов и блокировок; приводим её для полноты. Если у вас современная СУБД, избегайте `TABLE`.

`UUID` — в Hibernate 6+ доступна стратегия `GenerationType.UUID`. Это удобно в распределённых системах, где хочется генерировать ключи без похода в БД. Плюсы — независимость от последовательностей и предсказуемость; минусы — больший размер индексов и менее плотная локальность данных. Для Postgres обычно хранят в типе `uuid`.

Ещё один момент — “естественные” ключи. Иногда у сущности есть уникальный бизнес-идентификатор (например, ISO-код валюты). Его можно хранить в отдельном уникальном столбце, но не стоит делать его `@Id`: изменение такого кода (хоть и редкое) становится болезненным. Используйте суррогатный ключ (`Long/UUID`) + уникальный индекс на “естественный” атрибут.

**Java — примеры стратегий `IDENTITY`, `SEQUENCE`, `UUID`:**

```java
package com.example.jpa.mapping;

import jakarta.persistence.*;
import java.util.UUID;

@Entity
@Table(name = "customers")
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // автоинкремент
    private Long id;

    @Column(nullable = false, unique = true, length = 120)
    private String email;

    protected Customer() { }
    public Customer(String email) { this.email = email; }

    public Long getId() { return id; }
    public String getEmail() { return email; }
}

@Entity
@SequenceGenerator(name = "ord_seq", sequenceName = "order_seq", allocationSize = 1)
@Table(name = "orders")
class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "ord_seq")
    private Long id;

    @Column(nullable = false)
    private String number;

    protected Order() { }
    public Order(String number) { this.number = number; }
}

@Entity
@Table(name = "documents")
class Document {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(columnDefinition = "uuid")
    private UUID id;

    @Column(nullable = false)
    private String title;

    protected Document() { }
    public Document(String title) { this.title = title; }
}
```

**Kotlin — аналогичные сущности:**

```kotlin
package com.example.jpa.mapping

import jakarta.persistence.*
import java.util.*

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, unique = true, length = 120)
    var email: String = ""
)

@Entity
@SequenceGenerator(name = "ord_seq", sequenceName = "order_seq", allocationSize = 1)
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "ord_seq")
    var id: Long? = null,

    @Column(nullable = false)
    var number: String = ""
)

@Entity
@Table(name = "documents")
class Document(
    @Id @GeneratedValue(strategy = GenerationType.UUID)
    @Column(columnDefinition = "uuid")
    var id: UUID? = null,

    @Column(nullable = false)
    var title: String = ""
)
```

*Gradle — зависимости и плагины (для Kotlin — плагины `spring`/`jpa`):*
*Groovy DSL:*

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
    // Для Kotlin-проекта дополнительно:
    // id 'org.jetbrains.kotlin.jvm' version '1.9.24'
    // id 'org.jetbrains.kotlin.plugin.spring' version '1.9.24'
    // id 'org.jetbrains.kotlin.plugin.jpa' version '1.9.24'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL:*

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.24" apply false
    kotlin("plugin.spring") version "1.9.24" apply false
    kotlin("plugin.jpa") version "1.9.24" apply false
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

---

## Примитивы маппинга: `@Column` (nullability/uniqueness/length), `@Enumerated`, `@Lob`

`@Column` задаёт свойства столбца: `nullable`, `length`, `unique`, `precision/scale` для чисел. Важно различать: `nullable=false` — это **DDL/SQL-ограничение** на уровне БД; аннотации валидации (`@NotNull`, `@Size`) — это валидация на уровне приложения. В продакшне нужны **оба**: БД — последняя линия обороны, приложение — ранний фидбек и дружелюбные сообщения.

Поле `unique=true` на `@Column` создаёт одиночный уникальный индекс, но для составных уникальностей используйте `@Table(uniqueConstraints = {...})`. Такой подход документирует бизнес-инварианты и гарантирует их соблюдение независимо от багов в приложении.

`@Enumerated` управляет хранением `enum`. По умолчанию JPA — `ORDINAL` (числовой порядок), но это опасно: при перестановке констант данные “поедут”. Без лишних раздумий ставьте `@Enumerated(EnumType.STRING)`: хранится имя константы, что чуть больше по размеру, но гораздо безопаснее и читабельнее.

`@Lob` используется для больших данных: `CLOB` (текст) и `BLOB` (байты). В Postgres обычно это маппится на `text` и `bytea`. Хранить “большие” файлы (мегабайты+) в таблицах можно, но внимательнее с бэкапами и индексами; иногда выгоднее складывать бинарники в объектное хранилище (S3/MinIO), а в БД хранить ссылки и метаданные.

Для LOB-ов можно задать `@Basic(fetch = FetchType.LAZY)`, чтобы откладывать загрузку. В Hibernate это фактически лениво-загружаемые поля. Но помните, что ленивость работает **внутри транзакции**; вне её получите `LazyInitializationException`. В MVC-приложениях это ещё один аргумент против `open-in-view`.

Числовые поля требуют `precision/scale` для денежных значений: `@Column(precision = 19, scale = 4)` на `BigDecimal` — разумный дефолт для финансов, но согласуйте это с доменной командой. Неправильный scale ведёт к неожиданным округлениям и отказам индексов.

`columnDefinition` пригодится, когда вы хотите закрепить конкретный тип БД (например, `uuid`, `jsonb`, `text`). Не злоупотребляйте — это снижает переносимость, но для Postgres-специфичных типов это честный контракт.

**Java — маппинг колонок, enum и LOB:**

```java
package com.example.jpa.columns;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "items", uniqueConstraints = @UniqueConstraint(columnNames = {"sku"}))
public class Item {

    public enum Status { DRAFT, ACTIVE, ARCHIVED }

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 64)
    private String sku;

    @Column(nullable = false, length = 255)
    private String name;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 16)
    private Status status = Status.DRAFT;

    @Column(precision = 19, scale = 4, nullable = false)
    private BigDecimal price = BigDecimal.ZERO;

    @Lob
    @Basic(fetch = FetchType.LAZY)
    @Column(columnDefinition = "text")
    private String description; // крупные тексты

    @Lob
    @Basic(fetch = FetchType.LAZY)
    private byte[] image; // бинарные данные

    protected Item() {}
    public Item(String sku, String name) { this.sku = sku; this.name = name; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.columns

import jakarta.persistence.*
import java.math.BigDecimal

@Entity
@Table(name = "items", uniqueConstraints = [UniqueConstraint(columnNames = ["sku"])])
class Item(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, unique = true, length = 64)
    var sku: String = "",

    @Column(nullable = false, length = 255)
    var name: String = "",

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 16)
    var status: Status = Status.DRAFT,

    @Column(precision = 19, scale = 4, nullable = false)
    var price: BigDecimal = BigDecimal.ZERO,

    @Lob
    @Basic(fetch = FetchType.LAZY)
    @Column(columnDefinition = "text")
    var description: String? = null,

    @Lob
    @Basic(fetch = FetchType.LAZY)
    var image: ByteArray? = null
) {
    enum class Status { DRAFT, ACTIVE, ARCHIVED }
}
```

---

## Встроенные типы: `@Embeddable/@Embedded` для value-объектов

`@Embeddable` описывает **встраиваемый** тип — объект-значение (value object), чьи поля будут разложены в столбцы родительской таблицы, без отдельной таблицы и собственного первичного ключа. Это хороший способ выразить концептуально связанные поля (адрес, период, деньги), сделать их повторно используемыми и инкапсулировать валидацию.

`@Embedded` вставляет такой тип в сущность. Для нескольких встраиваний одного и того же embeddable используйте `@AttributeOverrides`, чтобы задать разные имена столбцов (например, `billing_*` и `shipping_*`). Это повышает выразительность модели и избавляет от префиксов в именах полей класса.

Embeddable хорош тем, что он «по значению»: два адреса считаются равными по содержанию, а не по идентификатору. В Java переопределяйте `equals/hashCode`, в Kotlin — это естественно для `data class`. Такая семантика помогает правильно сравнивать и хранить их в коллекциях.

Иммутабельность — желательное свойство value-object. В JPA для этого можно сделать поля `final` и только геттеры (Hibernate умеет работать с **field access**), но потребуется конструктор для инициализации. В Kotlin — `data class` c `val`-полями. Если нужна ленивость/прокси, помните, что Hibernate иногда требует открытые классы — используйте плагин `kotlin-jpa`/`all-open`.

Embeddable поддерживает вложенность (embeddable внутри embeddable), что позволяет моделировать сложные составные значения. Главное — вовремя остановиться и не превращать одну таблицу в «суперобъект» с сотней колонок: мониторьте ширину строки и индексы.

Value-объекты часто несут инварианты (например, `Period(start, end)` где `end > start`). Их удобно проверять в конструкторе или через Jakarta Validation на полях embeddable. Это держит модель корректной **до** попадания в БД.

С точки зрения производительности embeddable — «бесплатный» маппинг: никаких join-ов, только более широкие строки. Вы платите лишь ростом числа колонок и нагрузкой на сеть/диск при выборках. Балансируйте, исходя из реального профиля запросов.

**Java — адрес как embeddable, два встраивания в сущность:**

```java
package com.example.jpa.embed;

import jakarta.persistence.*;
import java.util.Objects;

@Embeddable
public class Address {
    @Column(length = 120, nullable = false) private String city;
    @Column(length = 200, nullable = false) private String street;
    @Column(length = 16, nullable = false) private String zip;

    protected Address() { }
    public Address(String city, String street, String zip) {
        this.city = city; this.street = street; this.zip = zip;
    }

    // equals/hashCode по значению
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Address a)) return false;
        return Objects.equals(city, a.city) && Objects.equals(street, a.street) && Objects.equals(zip, a.zip);
    }
    @Override public int hashCode() { return Objects.hash(city, street, zip); }
}

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;

    @Embedded
    @AttributeOverrides({
            @AttributeOverride(name = "city", column = @Column(name = "billing_city")),
            @AttributeOverride(name = "street", column = @Column(name = "billing_street")),
            @AttributeOverride(name = "zip", column = @Column(name = "billing_zip"))
    })
    private Address billing;

    @Embedded
    @AttributeOverrides({
            @AttributeOverride(name = "city", column = @Column(name = "shipping_city")),
            @AttributeOverride(name = "street", column = @Column(name = "shipping_street")),
            @AttributeOverride(name = "zip", column = @Column(name = "shipping_zip"))
    })
    private Address shipping;

    protected Customer() { }
    public Customer(Address billing, Address shipping) { this.billing = billing; this.shipping = shipping; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.embed

import jakarta.persistence.*
import java.util.*

@Embeddable
data class Address(
    @Column(length = 120, nullable = false) var city: String = "",
    @Column(length = 200, nullable = false) var street: String = "",
    @Column(length = 16, nullable = false) var zip: String = ""
)

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Embedded
    @AttributeOverrides(
        value = [
            AttributeOverride(name = "city", column = Column(name = "billing_city")),
            AttributeOverride(name = "street", column = Column(name = "billing_street")),
            AttributeOverride(name = "zip", column = Column(name = "billing_zip"))
        ]
    )
    var billing: Address? = null,

    @Embedded
    @AttributeOverrides(
        value = [
            AttributeOverride(name = "city", column = Column(name = "shipping_city")),
            AttributeOverride(name = "street", column = Column(name = "shipping_street")),
            AttributeOverride(name = "zip", column = Column(name = "shipping_zip"))
        ]
    )
    var shipping: Address? = null
)
```

---

## Конвертеры: `@Converter` для кастомных типов (например, Money, JSON-строки)

Иногда доменная модель использует типы, которым не соответствует «прямой» SQL-тип: `Money`, `Set<String>`, сложные структуры настроек и т. п. Аннотация `@Converter` позволяет определить **AttributeConverter<X, Y>**, где `X` — тип в сущности, `Y` — тип столбца БД. Конвертер прозрачно сериализует/десериализует значение при записи/чтении.

Ограничение: конвертер работает с **одним** столбцом. Если вам нужно разложить объект на несколько колонок — используйте `@Embeddable`. Для `Money` это значит: либо хранить в одном столбце (например, `jsonb`), либо выбрать один из атрибутов (что редко приемлемо).

Конвертеры хороши для хранения JSON благодаря типам Postgres `jsonb`. Вы сохраняете объект в строку JSON и обратно маппите через Jackson. Плюсы — гибкость и минимальный усилия; минусы — индексация и валидация ложатся на вас: для часто фильтруемых полей лучше делать нормализованные колонки.

`@Converter(autoApply = true)` включает конвертер для **всех** подходящих полей во всех сущностях. Это удобно, если у вас единый тип (например, `Money`) и вы везде хотите хранить его одинаково. Если нужно точечно — не ставьте `autoApply`, а помечайте поле `@Convert(converter = ...)`.

Учтите, что в JPQL вы оперируете колонкой (`Y`), а не объектом (`X`). Например, `where settings ->> 'theme' = 'dark'` — это уже нативный SQL для Postgres; JPQL такого не умеет. Поэтому для запросов по данным внутри JSON подойдёт `@Query(nativeQuery = true)`.

Конвертер — это простой класс без DI (по спецификации). Если вам нужен `ObjectMapper`, создайте его статически или используйте библиотечный по умолчанию. Не пытайтесь инжектить бины Spring — это ломает жизненный цикл. Альтернатива — Hibernate Types/`@JdbcTypeCode(SqlTypes.JSON)` (вне рамок этой главы).

В тестах убедитесь, что схема БД создаёт нужный тип столбца (`jsonb`, `text`) — иначе Hibernate может выбрать `text`, и вы потеряете преимущества JSON-операторов Postgres. Это можно закрепить `@Column(columnDefinition = "jsonb")`.

**Java — `Money` и JSON-конвертер настроек:**

```java
package com.example.jpa.converter;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.*;

import java.io.IOException;
import java.math.BigDecimal;
import java.util.Map;

public record Money(String currency, BigDecimal amount) { }

@Converter(autoApply = false)
class SettingsJsonConverter implements AttributeConverter<Map<String, Object>, String> {
    private static final ObjectMapper om = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Map<String, Object> attribute) {
        try { return attribute == null ? null : om.writeValueAsString(attribute); }
        catch (JsonProcessingException e) { throw new IllegalArgumentException(e); }
    }

    @Override
    @SuppressWarnings("unchecked")
    public Map<String, Object> convertToEntityAttribute(String dbData) {
        try { return dbData == null ? null : om.readValue(dbData, Map.class); }
        catch (IOException e) { throw new IllegalArgumentException(e); }
    }
}

@Entity
@Table(name = "accounts")
class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;

    @Column(nullable = false, length = 3)
    private String currency;

    @Column(precision = 19, scale = 4, nullable = false)
    private BigDecimal amount;

    @Convert(converter = SettingsJsonConverter.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> settings;

    protected Account() { }
    public Account(Money money) {
        this.currency = money.currency();
        this.amount = money.amount();
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.converter

import com.fasterxml.jackson.databind.ObjectMapper
import jakarta.persistence.*
import java.math.BigDecimal

data class Money(val currency: String, val amount: BigDecimal)

@Converter(autoApply = false)
class SettingsJsonConverter : AttributeConverter<Map<String, Any>?, String?> {
    companion object { private val om = ObjectMapper() }

    override fun convertToDatabaseColumn(attribute: Map<String, Any>?): String? =
        attribute?.let { om.writeValueAsString(it) }

    @Suppress("UNCHECKED_CAST")
    override fun convertToEntityAttribute(dbData: String?): Map<String, Any>? =
        dbData?.let { om.readValue(it, Map::class.java) as Map<String, Any> }
}

@Entity
@Table(name = "accounts")
class Account(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, length = 3)
    var currency: String = "",

    @Column(precision = 19, scale = 4, nullable = false)
    var amount: BigDecimal = BigDecimal.ZERO,

    @Convert(converter = SettingsJsonConverter::class)
    @Column(columnDefinition = "jsonb")
    var settings: Map<String, Any>? = null
) {
    constructor(money: Money) : this(currency = money.currency, amount = money.amount)
}
```

*Gradle — добавим Jackson, если у вас нет web-стартера:*
*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.2'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.fasterxml.jackson.core:jackson-databind:2.17.2")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

---

# 4. Жизненный цикл сущности и dirty checking

## Состояния: `transient → managed → detached → removed`; кто и когда переводит между ними

Состояние **transient** — это обычный новый объект Java, о котором JPA ещё не знает. Вы создали `new User("...")`, у него `id = null`, и он живёт отдельно от контекста персистентности. Любые изменения такого объекта не попадут в БД, пока вы явно не «привяжете» его к `EntityManager`.

Состояние **managed** возникает после `em.persist(entity)` (новая сущность) или после загрузки `em.find(...)`. Управляемые сущности “сидят” в Persistence Context (1-й уровень кеша), и Hibernate отслеживает их изменения. В пределах одной транзакции каждый ряд таблицы представлен **ровно одним** managed-экземпляром — это и есть identity map.

**detached** — это сущность, которая когда-то была managed, но больше не связана с текущим контекстом: вы вышли из транзакции, сделали `em.clear()`/`em.detach(entity)` или передали объект за пределы слоя данных (например, в веб-слой после коммита). Изменения detached-объекта не сохраняются автоматически.

Состояние **removed** — сущность помечена на удаление (`em.remove(entity)`). Физический `DELETE` произойдёт при `flush`/commit. До этого момента объект остаётся управляемым, но любая попытка его повторно «сохранить» потребует явной логики.

Переходы между состояниями управляются операциями `persist`, `find/getReference`, `merge`, `remove`, `clear/detach`, границами транзакций и каскадами. Например, `persist` родителя с `CascadeType.PERSIST` на коллекции переведёт детей из transient в managed автоматически.

В MVC-приложениях частая картина: в сервисе под транзакцией вы грузите entity (managed), возвращаете DTO наружу, коммитите транзакцию — и managed превращается в detached, если кто-то удерживает ссылку. Если затем где-то меняете поля такого detached-объекта — это не попадёт в БД. Это источник многих “пропадающих” изменений.

Явная операция `merge` переводит detached обратно в managed (точнее — создаёт/возвращает **другой** managed-экземпляр с тем же id и копирует в него состояние). Поэтому после `merge` используйте возвращённую ссылку, а не исходный объект — это один из самых частых анти-паттернов.

**Java — пошаговый переход состояний:**

```java
package com.example.jpa.lifecycle;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LifecycleService {
    @PersistenceContext private EntityManager em;

    @Transactional
    public Long createUser() {
        User u = new User("Neo"); // transient
        em.persist(u);            // -> managed (INSERT pending)
        return u.getId();
    }

    @Transactional
    public void detachAndMerge(Long id) {
        User u = em.find(User.class, id); // managed
        em.detach(u);                     // -> detached
        u.setName("Neo Updated");         // сменили detached
        User managedAgain = em.merge(u);  // managed copy получает изменения
        // исходный u остаётся detached
    }

    @Transactional
    public void removeUser(Long id) {
        User u = em.find(User.class, id); // managed
        em.remove(u);                     // -> removed (DELETE pending)
    }
}

@Entity
@Table(name = "users")
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false) private String name;
    protected User() { }
    public User(String name) { this.name = name; }
    public Long getId() { return id; }
    public void setName(String name) { this.name = name; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.lifecycle

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class LifecycleService(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun createUser(): Long {
        val u = User(name = "Neo") // transient
        em.persist(u)              // -> managed
        return u.id!!
    }

    @Transactional
    fun detachAndMerge(id: Long) {
        val u = em.find(User::class.java, id) // managed
        em.detach(u)                          // -> detached
        u.name = "Neo Updated"                // меняем detached
        val managedAgain = em.merge(u)        // получаем managed-копию с изменениями
    }

    @Transactional
    fun removeUser(id: Long) {
        val u = em.find(User::class.java, id)
        em.remove(u) // -> removed
    }
}

@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var name: String = ""
)
```

---

## Dirty checking: что считается изменением, когда происходит `flush`, влияние `FlushMode`

Механизм **dirty checking** сравнивает снимок состояния управляемой сущности с её текущими полями. Если находят различия — при `flush` генерируется `UPDATE` только изменённых колонок (если включён `@DynamicUpdate`, иначе — все колонки маппинга). Это позволяет писать бизнес-код на уровне объектов, не заботясь о низкоуровневых SQL.

`flush` вызывается автоматически при коммите транзакции и перед некоторыми запросами (в `FlushModeType.AUTO`). Например, если вы вызываете `em.createQuery("select ...")`, Hibernate, чтобы обеспечить консистентный результат запроса, предварительно синхронизирует незаписанные изменения. Это может породить лишние `UPDATE`-ы в неожиданных местах.

`FlushModeType.COMMIT` откладывает синхронизацию до коммита. Это экономит лишние `flush` перед `SELECT`, но будьте осторожны: запросы в рамках той же транзакции **не увидят** несинхронизированных изменений. Такой режим уместен для сценариев «много апдейтов — мало чтений» внутри одной транзакции.

Что считается изменением? Для простых полей — сравнение значений. Для коллекций — операции добавления/удаления/замены. Важно правильно реализовать `equals/hashCode` у embeddable и идентичность элементов коллекций, чтобы Hibernate корректно рассчитал дельту. Изменения ленивых прокси учитываются после инициализации.

Побочные эффекты: «грязные» изменения в больших графах (бидирекционные связи) могут вызвать каскадные `UPDATE` даже если вы тронули “другую” сторону. Дисциплинируйте вспомогательные методы, чтобы коллекции менялись консистентно (об этом — в теме про связи).

Для операций «только чтение» включайте `@Transactional(readOnly = true)` и по возможности используйте хинты `org.hibernate.readOnly=true` или соответствующие методы репозитория: это подсказывает провайдеру отключить dirty checking и экономит накладные расходы.

Если вам нужно немедленно “увидеть” side-effects (например, получить сгенерированный столбец/уникальность), вызывайте явный `em.flush()`. Это форсирует SQL до конца метода. Помните, что это может повлиять на производительность — не делайте `flush` слишком часто.

**Java — управление flush mode и read-only хинтами:**

```java
package com.example.jpa.flush;

import jakarta.persistence.*;
import org.hibernate.Session;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Repository
public class FlushDemoDao {
    @PersistenceContext private EntityManager em;

    @Transactional
    public void heavyWritesThenSelect() {
        em.setFlushMode(FlushModeType.COMMIT); // не флешить перед select
        for (int i = 0; i < 1000; i++) {
            em.persist(new Note("n" + i));
        }
        // select, который не триггерит flush
        List<Note> notes = em.createQuery("select n from Note n", Note.class)
                .setHint(org.hibernate.annotations.QueryHints.READ_ONLY, true)
                .getResultList();
        // commit вызовет flush один раз
    }

    @Transactional(readOnly = true)
    public List<Note> readOnlyList() {
        Session session = em.unwrap(Session.class);
        session.setDefaultReadOnly(true); // отключаем dirty checking
        return em.createQuery("select n from Note n", Note.class).getResultList();
    }
}

@Entity
@Table(name = "notes")
class Note {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String title;
    protected Note() { }
    Note(String title){ this.title = title; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.flush

import jakarta.persistence.*
import org.hibernate.Session
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional

@Repository
class FlushDemoDao(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun heavyWritesThenSelect() {
        em.flushMode = FlushModeType.COMMIT
        repeat(1000) { em.persist(Note(title = "n$it")) }
        val notes = em.createQuery("select n from Note n", Note::class.java)
            .setHint(org.hibernate.annotations.QueryHints.READ_ONLY, true)
            .resultList
    }

    @Transactional(readOnly = true)
    fun readOnlyList(): List<Note> {
        val session = em.unwrap(Session::class.java)
        session.isDefaultReadOnly = true
        return em.createQuery("select n from Note n", Note::class.java).resultList
    }
}

@Entity
@Table(name = "notes")
class Note(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var title: String = ""
)
```

---

## `merge` vs `persist`: типичные сценарии и подводные камни со «склейкой» графов

`persist` — для **новых** объектов. Он делает сущность managed и планирует `INSERT`. Если сущность уже существует (у неё не-null id, и такой ряд есть в БД), `persist` выбросит исключение. Поэтому проверяйте, что новые объекты действительно новые (`id == null`), а не «полусохранённые».

`merge` — для **detached** объектов (обычно приходящих из веб-слоя как DTO, преобразованные в сущность). Он **копирует** состояние в managed-экземпляр (найдённый по id или созданный) и возвращает ссылку на него. Исходный объект остаётся detached; дальнейшие изменения в нём игнорируются JPA — это критический нюанс.

При `merge` графов с коллекциями возможны «стирания»: если в detached-объекте коллекция не содержит некоторых детей, Hibernate решит, что их нужно удалить (при `orphanRemoval=true`) или разлинковать. Поэтому безопаснее применять “патч” к загруженному из БД managed-объекту: загрузить entity, затем аккуратно перенести изменения по полям и коллекциям.

Ещё одна ловушка — «слепой merge» со смежными сущностями. Если в графе вы передали детей с только id (без остальных полей), Hibernate может произвести лишние загрузки или попытаться вставить дубликаты, если каскады настроены неаккуратно. В целом, чем сложнее граф, тем осторожнее нужно использовать `merge`.

Нередко проще отказаться от `merge` в пользу явного сценария: загрузить managed-сущность, применить изменения из DTO (только разрешённые поля), добавить/удалить элементы коллекций через вспомогательные методы, сохранить. Это прозрачно и предсказуемо с точки зрения SQL.

`merge` всё же полезен для простых «плоских» сущностей (без сложных коллекций), для синхронизации detached-экземпляров в кэшах/фонах, а также в интеграциях, где приходит полный снимок состояния и его можно безболезненно “наложить”.

Наконец, помните про конкуренцию. При параллельных обновлениях используйте оптимистическую блокировку (`@Version`), чтобы `merge` не перезаписал чужие изменения. Версия — верный способ обнаружить конфликт и сообщить клиенту, что нужно перечитать/повторить.

**Java — сравнение `persist` (create) и “патч вместо merge” (update):**

```java
package com.example.jpa.merge;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProfileService {
    @PersistenceContext private EntityManager em;

    @Transactional
    public Long create(ProfileDto dto) {
        Profile p = new Profile(dto.email, dto.name);
        em.persist(p); // новый -> INSERT
        return p.getId();
    }

    @Transactional
    public void update(Long id, ProfileDto dto) {
        Profile p = em.find(Profile.class, id); // managed, актуальная версия из БД
        p.setName(dto.name);
        // коллекции/связи меняем через вспомогательные методы, чтобы избежать "стереть всё"
    }
}

@Entity
@Table(name = "profiles")
class Profile {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false, unique = true, length = 120) private String email;
    @Column(nullable = false, length = 120) private String name;

    protected Profile() { }
    public Profile(String email, String name) { this.email = email; this.name = name; }
    public Long getId() { return id; }
    public void setName(String name) { this.name = name; }
}

class ProfileDto {
    public String email;
    public String name;
}
```

**Kotlin — аналог:**

```kotlin
package com.example.jpa.merge

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ProfileService(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun create(dto: ProfileDto): Long {
        val p = Profile(email = dto.email, name = dto.name)
        em.persist(p)
        return p.id!!
    }

    @Transactional
    fun update(id: Long, dto: ProfileDto) {
        val p = em.find(Profile::class.java, id)
        p.name = dto.name
        // связи/коллекции — через вспомогательные методы
    }
}

@Entity
@Table(name = "profiles")
class Profile(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 120)
    var email: String = "",
    @Column(nullable = false, length = 120)
    var name: String = ""
)

data class ProfileDto(val email: String, val name: String)
```

# 5. Связи между сущностями (основы)

*Зависимости (для всех подпунктов ниже одинаковые)*

**Gradle Groovy**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

**Gradle Kotlin**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

## Виды: `@ManyToOne`, `@OneToMany`, `@OneToOne`, `@ManyToMany`; где хранится внешний ключ

Связи в JPA описывают, как объекты отражают реляционные отношения. В базе данных связь реализуется через внешний ключ (FK) или промежуточную таблицу; в JPA — через аннотации на полях сущностей. Понимание, **где физически лежит FK**, помогает правильно выбрать сторону владения и избежать лишних апдейтов.

`@ManyToOne` — самый частый случай «многие к одному», например, строка заказа относится к одному заказу. В реляционной модели FK хранится на стороне «многие», то есть в таблице дочерних записей. В JPA это логично маппится полем в дочерней сущности с `@JoinColumn`, и **владелец** связи — именно эта сторона. Это важно: именно владелец формирует SQL для поддержания FK.

`@OneToMany` без промежуточной таблицы — зеркальная часть той же связи: один заказ — много строк. Если вы делаете её «односторонней», Hibernate вынужден создавать join table, потому что FK-то на стороне детей. Поэтому в продакшене мы почти всегда делаем **двустороннюю** пару `@ManyToOne` + `@OneToMany(mappedBy="...")`, где владеет ребёнок. Это даёт правильный и эффективный SQL.

`@OneToOne` хранит FK на одной из сторон. Вариантов два: отдельная колонка FK или «общий PK» (shared primary key). Первый вариант проще и гибче; второй экономит колонку и иногда удобен для «расширений» сущности (одна запись — ровно одно доп.поле). Для `@OneToOne` важно заранее решить, где будет FK, чтобы избежать каскадных лишних апдейтов.

`@ManyToMany` почти всегда реализуется через таблицу связей (join table) с двумя FK, указывающими на обе стороны. Эта таблица хранит только пары ключей (и, при необходимости, дополнительные атрибуты, но тогда правильнее превратить её в полноценную сущность «ассоциации» и маппить как две `@ManyToOne`). На уровне JPA вы получите коллекции с обеих сторон и необходимость аккуратно поддерживать консистентность.

Односторонние связи иногда полезны для упрощения модели. Односторонний `@ManyToOne` — нормальная практика. Односторонний `@OneToMany` — почти всегда признак неоптимального решения: Hibernate создаст join table, и вы потеряете простоту и производительность. Двусторонние связи дают больше контроля над SQL.

Наконец, помните, что каждая связь — это потенциальный join. Критически важно указывать `fetch`-тип (подробнее в следующей теме) и продумывать стратегии загрузки. Начните с **LAZY** где можно, и подгружайте явно (`JOIN FETCH`) там, где требуется.

**Java — пример всех видов связей (минимально воспроизводимый):**

```java
package com.example.relations.basic;

import jakarta.persistence.*;
import java.util.*;

@Entity @Table(name = "authors")
public class Author {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false) private String name;

    @ManyToMany
    @JoinTable(name = "author_book",
            joinColumns = @JoinColumn(name = "author_id"),
            inverseJoinColumns = @JoinColumn(name = "book_id"))
    private Set<Book> books = new HashSet<>();

    protected Author() {}
    public Author(String name) { this.name = name; }
    public void addBook(Book b){ books.add(b); b.getAuthors().add(this); }
    public Set<Book> getBooks(){ return books; }
}

@Entity @Table(name = "books")
class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false) private String title;

    @ManyToMany(mappedBy = "books")
    private Set<Author> authors = new HashSet<>();

    @OneToOne(mappedBy = "book", cascade = CascadeType.ALL, orphanRemoval = true)
    private BookDetails details;

    protected Book() {}
    public Book(String title){ this.title = title; }
    public Set<Author> getAuthors(){ return authors; }
    public void setDetails(BookDetails d){ this.details = d; d.setBook(this); }
}

@Entity @Table(name = "book_details")
class BookDetails {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    private String isbn;

    @OneToOne @JoinColumn(name = "book_id", unique = true)
    private Book book;

    public void setBook(Book book){ this.book = book; }
}

@Entity @Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    List<OrderLine> lines = new ArrayList<>();
    public void addLine(OrderLine l){ l.order = this; lines.add(l); }
}

@Entity @Table(name = "order_lines")
class OrderLine {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(optional = false) @JoinColumn(name = "order_id")
    Order order;
    @Column(nullable = false) String sku;
    @Column(nullable = false) long qty;
    protected OrderLine() {}
    public OrderLine(String sku, long qty){ this.sku = sku; this.qty = qty; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.relations.basic

import jakarta.persistence.*
import java.util.*

@Entity @Table(name = "authors")
class Author(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var name: String = ""
) {
    @ManyToMany
    @JoinTable(
        name = "author_book",
        joinColumns = [JoinColumn(name = "author_id")],
        inverseJoinColumns = [JoinColumn(name = "book_id")]
    )
    var books: MutableSet<Book> = mutableSetOf()

    fun addBook(b: Book) { books.add(b); b.authors.add(this) }
}

@Entity @Table(name = "books")
class Book(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var title: String = ""
) {
    @ManyToMany(mappedBy = "books")
    var authors: MutableSet<Author> = mutableSetOf()

    @OneToOne(mappedBy = "book", cascade = [CascadeType.ALL], orphanRemoval = true)
    var details: BookDetails? = null

    fun setDetails(d: BookDetails) { details = d; d.book = this }
}

@Entity @Table(name = "book_details")
class BookDetails(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var isbn: String? = null
) {
    @OneToOne @JoinColumn(name = "book_id", unique = true)
    var book: Book? = null
}

@Entity @Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
) {
    @OneToMany(mappedBy = "order", cascade = [CascadeType.ALL], orphanRemoval = true)
    var lines: MutableList<OrderLine> = mutableListOf()
    fun addLine(l: OrderLine) { l.order = this; lines.add(l) }
}

@Entity @Table(name = "order_lines")
class OrderLine(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(optional = false) @JoinColumn(name = "order_id")
    var order: Order? = null,
    @Column(nullable = false) var sku: String = "",
    @Column(nullable = false) var qty: Long = 0
)
```

---

## Владелец связи и `mappedBy`: почему важно и чем грозит неверная сторона

В JPA «владелец связи» — это сторона, чье поле содержит фактический FK или чья коллекция управляет строками в join table. Только **владелец** формирует `INSERT/UPDATE` FK-колонки. Вторая сторона с `mappedBy` — «обратная», она отражает связь, но не генерирует SQL для поддержания FK. Ошибка со стороной владения приводит к странным багам: JPA меняет «не ту» сущность или делает лишние апдейты.

Для `@ManyToOne`/`@OneToMany` владелец — всегда `@ManyToOne`, потому что FK хранится в дочерней таблице. Значит, изменения коллекции на стороне `@OneToMany` **без** синхронизации соответствующих полей `@ManyToOne` ни к чему не приведут: JPA не знает, что именно нужно поменять в колонке FK. Вы получите «залипшие» связи или «доп. апдейты» в неожиданных местах.

Для `@OneToOne` владелец — та сторона, где находится `@JoinColumn`. Если вы ошиблись и поставили `mappedBy` не там, где нужно, Hibernate начнёт вставлять/обновлять нулевые FK, чтобы «согласовать» модель. В логах SQL это видно как лишние `update ... set fk = ?`.

Для `@ManyToMany` владелец — сторона, не имеющая `mappedBy`; именно она управляет строками в join table. Если вы добавили связь только на обратной стороне (с `mappedBy`), строки в join table **не появятся**, пока не обновите владельца.

Вывод: всегда добавляйте **вспомогательные методы** на владельце, которые поддерживают обе стороны: `parent.addChild(child)` должен выставлять `child.setParent(parent)` и наоборот. Это убережёт от рассинхронизаций и «немых» ошибок.

Неверная сторона владения ещё и производительная проблема. Hibernate может сгенерировать **два** апдейта там, где достаточно одного: например, сначала поставить FK в `NULL`, затем новый FK. При большом объёме это заметный лишний трафик и блокировки.

Тестируйте связи: пишите небольшие интеграционные тесты, которые создают сущности, добавляют/удаляют элементы и проверяют итоговые строки в БД. Это дешёвый способ поймать неправильный `mappedBy` до продакшна.

**Java — правильный владелец и вспомогательные методы:**

```java
package com.example.relations.owner;

import jakarta.persistence.*;
import java.util.*;

@Entity @Table(name = "departments")
class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();

    public void addEmployee(Employee e) { employees.add(e); e.setDepartment(this); }
    public void removeEmployee(Employee e) { employees.remove(e); e.setDepartment(null); }
}

@Entity @Table(name = "employees")
class Employee {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "department_id")
    private Department department;

    void setDepartment(Department d){ this.department = d; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.relations.owner

import jakarta.persistence.*

@Entity @Table(name = "departments")
class Department(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
) {
    @OneToMany(mappedBy = "department", cascade = [CascadeType.ALL], orphanRemoval = true)
    var employees: MutableList<Employee> = mutableListOf()

    fun addEmployee(e: Employee) { employees.add(e); e.department = this }
    fun removeEmployee(e: Employee) { employees.remove(e); e.department = null }
}

@Entity @Table(name = "employees")
class Employee(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
) {
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "department_id")
    var department: Department? = null
}
```

---

## Каскады `CascadeType` и `orphanRemoval`: когда уместно включать, когда лучше вручную

Каскады определяют, какие операции на родителе автоматически применяются к зависимым. `PERSIST` вставляет детей при сохранении родителя, `MERGE` обновляет, `REMOVE` удаляет, `REFRESH` перечитывает, `DETACH` отсоединяет, `ALL` включает всё перечисленное. Правило практики: включайте **минимально достаточный** набор, избегайте `ALL` «по привычке».

`PERSIST` уместен для агрегатов: когда жизненный цикл детей полностью принадлежит родителю (строки заказа, адреса пользователя как value). Тогда `repo.save(parent)` создаст и детей без дополнительного кода. Это снижает бойлерплейт и делает код выразительным.

`REMOVE` полезен там же, но требует осторожности. Никогда не ставьте `REMOVE` на ссылку на «справочник» (`@ManyToOne` на `Country`, `Role` и т. п.), иначе удаление пользователя может удалить всю страну/роль. Для таких ссылок каскады обычно **выключены**.

`orphanRemoval = true` удаляет «сирот» — элементы коллекции, удалённые из `@OneToMany`. Это удобно для агрегатов: удалили строку из `order.lines` — Hibernate сгенерировал `DELETE` для неё. Но orphanRemoval — это **не** каскад `REMOVE`; он срабатывает именно при «разлинковке». На больших коллекциях массовая замена может привести к множеству `DELETE/INSERT`.

`MERGE` часто включают для удобства, но с ним осторожно: слепой merge detached-графов способен «стереть» коллекции (см. предыдущую тему). В критичных местах лучше обновлять managed-объект явными методами, а не полагаться на merge.

На уровне производительности каскады дают больше SQL, чем вы видите в коде. В логах отслеживайте, что реально происходит. Возможно, лучше сначала сохранить родителя, затем батчем — детей через `saveAll`, если это отдельный агрегат.

Если вы сомневаетесь — отключайте каскады и пишите явный сервисный код. Это вернёт контроль и предсказуемость. Каскады — удобны, но не «автопилот».

**Java — каскады и orphanRemoval на агрегате «Заказ → Строки»:**

```java
package com.example.relations.cascade;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Entity @Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @OneToMany(mappedBy = "order", cascade = {CascadeType.PERSIST, CascadeType.MERGE}, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();
    public void addLine(String sku, long qty){ OrderLine l = new OrderLine(this, sku, qty); lines.add(l); }
    public void removeLine(int idx){ OrderLine l = lines.remove(idx); l.setOrder(null); }
}

@Entity @Table(name = "order_lines")
class OrderLine {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "order_id")
    private Order order;
    @Column(nullable = false) String sku; @Column(nullable = false) long qty;
    protected OrderLine(){ }
    OrderLine(Order order, String sku, long qty){ this.order = order; this.sku = sku; this.qty = qty; }
    void setOrder(Order o){ this.order = o; }
}

@Service
class OrderService {
    @PersistenceContext private EntityManager em;

    @Transactional
    public Long createWithLines() {
        Order o = new Order();
        o.addLine("A", 1); o.addLine("B", 2);
        em.persist(o); // каскад PERSIST сохранит строки
        return o.id;
    }

    @Transactional
    public void removeFirstLine(Long orderId) {
        Order o = em.find(Order.class, orderId);
        o.removeLine(0); // orphanRemoval -> DELETE строки
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.relations.cascade

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity @Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
) {
    @OneToMany(mappedBy = "order", cascade = [CascadeType.PERSIST, CascadeType.MERGE], orphanRemoval = true)
    var lines: MutableList<OrderLine> = mutableListOf()

    fun addLine(sku: String, qty: Long) { lines.add(OrderLine(order = this, sku = sku, qty = qty)) }
    fun removeLine(idx: Int) { val l = lines.removeAt(idx); l.order = null }
}

@Entity @Table(name = "order_lines")
class OrderLine(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "order_id")
    var order: Order? = null,
    @Column(nullable = false) var sku: String = "",
    @Column(nullable = false) var qty: Long = 0
)

@Service
class OrderService(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun createWithLines(): Long {
        val o = Order()
        o.addLine("A", 1); o.addLine("B", 2)
        em.persist(o)
        return o.id!!
    }

    @Transactional
    fun removeFirstLine(orderId: Long) {
        val o = em.find(Order::class.java, orderId)
        o.removeLine(0)
    }
}
```

---

## Бидирекционные связи: вспомогательные методы для поддержания консистентности обеих сторон

Бидирекционная связь требует синхронизации обеих сторон: коллекции родителя и ссылочного поля ребёнка. Если вы обновите только коллекцию, FK в дочерней строке не изменится, потому что владелец — ребёнок. Если измените только ссылку у ребёнка — коллекция родителя «отстанет», и следующий доступ внутри транзакции увидит рассинхрон.

Решение — **вспомогательные методы**, которые одновременно обновляют обе стороны. На родителе — `addChild/removeChild`, на ребёнке — `setParent`. Эти методы должны быть единственным способом менять связь из бизнес-кода, а прямые манипуляции коллекцией — запрещены (можно даже вернуть «неизменяемую» коллекцию наружу).

Ещё один аспект — `equals/hashCode`. Для сущностей, у которых ещё нет `id`, сравнение по `id` может «ломаться». Рекомендация: до появления `id` сравнивать по ссылке (дефолт), а `hashCode` не использовать для mutable-коллекций. В children лучше не полагаться на `equals` для поиска/удаления до сохранения.

С коллекциями всегда следите за «двойными» вставками/удалениями. Два вызова `addChild` на один и тот же объект могут привести к дублям (особенно в `List`). В `Set` потребуется корректный `equals/hashCode` после присвоения `id`. Для стабильности можно хранить children в `List`, а уникальность обеспечивать в домене.

При удалении «сирот» через `orphanRemoval` вспомогательные методы должны **разлинковывать** обе стороны. Удаление из коллекции без `child.setParent(null)` может оставить мёртвую ссылку в кеше 1-го уровня и сгенерировать неожиданный SQL.

Если связь редко используется «с той стороны», не делайте её двусторонней — это уменьшит риск рассинхрона и снизит стоимость загрузки. Но если вы решились — дисциплинируйте обновления только через методы.

Покрывайте вспомогательные методы тестами. Это «мелочь», но ломается она часто и незаметно. Небольшой тест, который проверяет, что `add/remove` меняют обе стороны и порождают ожидаемый SQL, окупается многократно.

**Java — вспомогательные методы и защита от рассинхрона:**

```java
package com.example.relations.helpers;

import jakarta.persistence.*;
import java.util.*;

@Entity @Table(name = "projects")
class Project {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @OneToMany(mappedBy = "project", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Task> tasks = new ArrayList<>();

    public List<Task> getTasks() { return Collections.unmodifiableList(tasks); }

    public void addTask(Task t){
        if (!tasks.contains(t)) {
            tasks.add(t);
            t.setProject(this);
        }
    }
    public void removeTask(Task t){
        if (tasks.remove(t)) {
            t.setProject(null);
        }
    }
}

@Entity @Table(name = "tasks")
class Task {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "project_id")
    private Project project;
    @Column(nullable = false) String title;

    protected Task(){ }
    public Task(String title){ this.title = title; }
    public void setProject(Project p){ this.project = p; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.relations.helpers

import jakarta.persistence.*
import java.util.*

@Entity @Table(name = "projects")
class Project(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
) {
    @OneToMany(mappedBy = "project", cascade = [CascadeType.ALL], orphanRemoval = true)
    private val _tasks: MutableList<Task> = mutableListOf()
    val tasks: List<Task> get() = Collections.unmodifiableList(_tasks)

    fun addTask(t: Task) {
        if (!_tasks.contains(t)) {
            _tasks.add(t)
            t.project = this
        }
    }
    fun removeTask(t: Task) {
        if (_tasks.remove(t)) {
            t.project = null
        }
    }
}

@Entity @Table(name = "tasks")
class Task(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = ""
) {
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "project_id")
    var project: Project? = null
}
```

---

# 6. Загрузка данных: LAZY/EAGER и прокси

*Зависимости (общие для подпунктов)*

**Gradle Groovy**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

**Gradle Kotlin**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

## Дефолты JPA: `@ManyToOne/@OneToOne` — EAGER, `@OneToMany/@ManyToMany` — LAZY; почему почти всегда явно ставим LAZY

По спецификации JPA загрузка по умолчанию для `@ManyToOne` и `@OneToOne` — **EAGER** (жадная), а для коллекций (`@OneToMany`, `@ManyToMany`) — **LAZY** (ленивая). В реальной жизни дефолт EAGER почти всегда приводит к **N+1** и «снежным комам» join-ов: загрузили 20 заказов — внезапно вытянули 20 связанных клиентов и их адреса, а затем ещё и их роли. Поэтому в продакшене мы **почти всегда явно ставим LAZY** везде, где это поддерживается.

Для `@ManyToOne` LAZY поддерживается без доп.настроек: Hibernate подставит прокси по ссылке и инициализирует её при первом доступе. Это снижает число джоинов в базовой выборке и даёт вам контроль над точками загрузки. Для `@OneToOne` ленивость сложнее: полноценная LAZY обычно требует байткод-энхансера Hibernate или специальных аннотаций (`@LazyToOne`). Если вы не включаете энхансер, `@OneToOne(fetch = LAZY)` может быть проигнорирован — учитывайте это.

EAGER — это «опасная глобальная настройка». Она тянет данные «за вас», независимо от того, нужны они сейчас или нет. В небольших формах это незаметно, но в списках/таблицах с пагинацией приводит к взрывному росту SQL и объёма передаваемых данных. Жадные связи также усложняют кэш 1-го уровня и вызывают лишние `flush` перед `SELECT`.

Ставя LAZY, вы берёте ответственность за загрузку на уровень сервисов/репозиториев. Это хорошо: вы явно решаете, **что и когда** подгружать (через `JOIN FETCH`, `EntityGraph` и т. п.), и контролируете форму SQL. Такой код легче профилировать и оптимизировать.

При этом ленивость не «бесплатна»: она создаёт прокси-объекты и требует активного контекста персистентности при инициализации. Вы должны чётко держать границы транзакций и не «таскать» сущности во внешний мир. DTO-маппинг внутри транзакции — здоровая практика.

В больших графах полезно начинать с LAZY везде, затем для часто используемых «срезов» добавить целевые `JOIN FETCH`/графы. В обратную сторону — сложнее: убрать EAGER, который успели разнести по коду, больно и долго.

Для отчётных запросов и агрегаций связность часто не нужна вовсе. Возвращайте проекции/DTO без сущностей — это «сухой» подход, который обходит вопросы ленивости в принципе.

**Java — явный LAZY на to-one, дефолт LAZY на коллекции:**

```java
package com.example.fetch.defaults;

import jakarta.persistence.*;
import java.util.*;

@Entity @Table(name = "invoices")
public class Invoice {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

    @ManyToOne(fetch = FetchType.LAZY)  // по умолчанию EAGER, мы делаем LAZY
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "invoice", cascade = CascadeType.ALL, orphanRemoval = true) // коллекции и так LAZY
    private List<InvoiceLine> lines = new ArrayList<>();

    // геттеры/сеттеры опущены ради краткости
}

@Entity @Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String name;
}

@Entity @Table(name = "invoice_lines")
class InvoiceLine {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "invoice_id")
    Invoice invoice;
    String sku; long qty;
}
```

**Kotlin — аналог:**

```kotlin
package com.example.fetch.defaults

import jakarta.persistence.*

@Entity @Table(name = "invoices")
class Invoice(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
) {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    var customer: Customer? = null

    @OneToMany(mappedBy = "invoice", cascade = [CascadeType.ALL], orphanRemoval = true)
    var lines: MutableList<InvoiceLine> = mutableListOf()
}

@Entity @Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var name: String = ""
)

@Entity @Table(name = "invoice_lines")
class InvoiceLine(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "invoice_id")
    var invoice: Invoice? = null,
    var sku: String = "",
    var qty: Long = 0
)
```

---

## Прокси-объекты и `LazyInitializationException`: откуда берётся и базовые способы избежать (транзакционные границы, fetch join)

Ленивая загрузка работает через **прокси** — подменные объекты, которые держат только id и знают, как загрузить настоящую сущность при первом доступе к полям. Прокси нужен активный `EntityManager` (контекст персистентности). Если вы обращаетесь к ленивому полю **после** закрытия контекста, Hibernate не может инициализировать прокси и выбрасывает `LazyInitializationException`.

Наиболее частый сценарий: `spring.jpa.open-in-view=false` (правильно), контроллер возвращает сущности во «внешний мир», а сериализация JSON пытается «залезть» в ленивые связи уже **после** выхода из транзакции. Итог — LIE. Это не баг, а сигнал о том, что вы смешали слои и утащили модель БД в REST.

Избежать LIE можно несколькими путями. Первый — **делать всё нужное внутри транзакции**: сервисный метод под `@Transactional` загружает сущности, «распаковывает» нужные ленивые поля в DTO и возвращает DTO наружу. Контроллер больше не трогает сущности. Это архитектурно чисто.

Второй — заранее подгружать нужные связи через `JOIN FETCH` или `EntityGraph`. Тогда даже при использовании сущностей в контроллере (что я не рекомендую в общем случае) необходимые данные уже будут инициализированы, и сериализация не встретит прокси. Но помните о рисках N+1 и пагинации — см. следующий подпункт.

Третий — включить `open-in-view=true`. Тогда `EntityManager` «протягивается» до сериализации, и LIE не случится. Но это компромисс: вы размываете границы транзакции, получаете скрытые запросы в слое представления и сложности с нагрузкой. В большинстве серьёзных сервисов его отключают.

Иногда LIE возникает при логировании сущностей вне транзакции (toString/hashCode equals, которые лезут в ленивые поля). Не делайте этого. Убедитесь, что ваши методы `toString` не трогают ленивые коллекции/ссылки.

Проверяйте инициализацию явно, если нужно: `Hibernate.isInitialized(obj)` или «доступ к id» (он доступен без инициализации) вместо доступа к полям. Но лучше вообще не носить сущности за пределы сервисов.

В реактивных/асинхронных сценариях проблема усугубляется из-за смены потоков и жизненного цикла контекста. Ещё один аргумент за DTO-маппинг внутри транзакции.

**Java — сервис с DTO и fetch join для избежания LIE:**

```java
package com.example.fetch.lie;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class InvoiceService {
    @PersistenceContext private EntityManager em;

    public record InvoiceDto(Long id, String customerName, long linesCount) {}

    @Transactional(readOnly = true)
    public List<InvoiceDto> list() {
        // Подгружаем клиента, а коллекцию считаем в SQL (пример, без коллекций в DTO)
        List<Object[]> rows = em.createQuery("""
            select i.id, c.name, count(l.id)
            from Invoice i
            join i.customer c
            left join i.lines l
            group by i.id, c.name
        """, Object[].class).getResultList();

        return rows.stream()
                .map(r -> new InvoiceDto((Long) r[0], (String) r[1], (Long) r[2]))
                .toList();
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.fetch.lie

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class InvoiceService(@PersistenceContext private val em: EntityManager) {

    data class InvoiceDto(val id: Long, val customerName: String, val linesCount: Long)

    @Transactional(readOnly = true)
    fun list(): List<InvoiceDto> {
        val rows = em.createQuery(
            """
            select i.id, c.name, count(l.id)
            from Invoice i
            join i.customer c
            left join i.lines l
            group by i.id, c.name
            """.trimIndent(),
            Array<Any>::class.java
        ).resultList

        return rows.map { InvoiceDto(it[0] as Long, it[1] as String, it[2] as Long) }
    }
}
```

---

## Базовая настройка выборок: когда уместен `JOIN FETCH` (детали оптимизаций — в следующей теме)

`JOIN FETCH` — способ сказать JPA: «подгрузи связанную сущность/коллекцию **в рамках этого запроса**». Он заменяет ленивую инициализацию последующим запросом на один объединённый SQL. Это лекарство от N+1 при аккуратном применении.

Уместные случаи: подгрузка **to-one** (`@ManyToOne`, `@OneToOne`) почти всегда безопасна — один join, ограниченный рост строк. Например, грузим список заказов и сразу подтягиваем клиента: `select o from Order o join fetch o.customer where ...`. Это снизит N+1 без накладных расходов.

С коллекциями (`@OneToMany`, `@ManyToMany`) `JOIN FETCH` опаснее. Джоин умножает строки: один заказ с 50 строками превратится в 50 строк результата. Если вы подгружаете ещё и другие коллекции, получите «декартово умножение». Списки с пагинацией ломаются: база пагинирует **по умноженным строкам**, а не по уникальным заказам. Поэтому «коллекционный» fetch join используйте **точечно** и без пагинации, либо дожимайте до DTO с агрегированием.

Если всё же нужен fetch коллекции, добавляйте `distinct` в JPQL, чтобы Hibernate отфильтровал дубликаты сущностей на уровне памяти. Это не меняет SQL, но защищает от повторов в результате. Однако пагинацию это не спасёт — страница отрежется «по строкам».

Альтернатива — `EntityGraph`. Он позволяет описать набор связей для подгрузки, не вмешиваясь в текст JPQL. Удобно для «типовых» срезов UI. Однако ограничения с коллекциями и пагинацией остаются: магии нет.

Для сложных экранов чаще выгодно возвращать **DTO-проекции**, собирая ровно те поля, которые нужны, одним хорошо спланированным SQL. Это упрощает код и снимает вопросы с ленивостью. ORM хорош для доменных операций, а не для произвольных отчётов.

Под капотом Hibernate может оптимизировать подгрузки (batch fetching, subselect), но это тема тонкой настройки и следующей главы. На старте достаточно дисциплины: LAZY везде, `JOIN FETCH` — на to-one, DTO — для коллекций с пагинацией.

**Java — репозиторий с `JOIN FETCH` и предостережение для коллекций:**

```java
package com.example.fetch.join;

import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import java.util.*;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("select o from Order o join fetch o.customer where o.id = :id")
    Optional<Order> findWithCustomer(@Param("id") Long id);

    @Query("select distinct o from Order o left join fetch o.lines where o.id in :ids")
    List<Order> findAllWithLines(@Param("ids") List<Long> ids); // без пагинации!
}
```

**Kotlin — аналог:**

```kotlin
package com.example.fetch.join

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import java.util.*

@Repository
interface OrderRepository : JpaRepository<Order, Long> {

    @Query("select o from Order o join fetch o.customer where o.id = :id")
    fun findWithCustomer(@Param("id") id: Long): Optional<Order>

    @Query("select distinct o from Order o left join fetch o.lines where o.id in :ids")
    fun findAllWithLines(@Param("ids") ids: List<Long>): List<Order> // не использовать с Pageable
}
```

# 7. Запросы: JPQL и базовые альтернативы

*Зависимости (общие для всех подпунктов)*

**Gradle Groovy**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle Kotlin**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## JPQL как язык над сущностями; именованные и типизированные запросы, параметризация

JPQL (Java Persistence Query Language) — декларативный язык запросов поверх **модели сущностей**, а не таблиц. В нём вы оперируете классами `@Entity` и их полями/связями, а провайдер JPA (обычно Hibernate) транслирует JPQL в конкретный SQL под используемую СУБД. Это позволяет писать переносимые запросы и не привязываться к диалекту БД в простых CRUD/поисковых сценариях. JPQL поддерживает `select`, `from`, `where`, `join`, агрегаты, подзапросы и выражения по свойствам сущностей.

Ключевая особенность JPQL — **путь по ассоциациям**. Вместо явного `JOIN` по ключам вы пишете `join o.customer c` и обращаетесь к `c.name`, а не к `customer_id`. Такой синтаксис тесно связан с моделью, уменьшает бойлерплейт и делает код самодокументируемым, особенно когда сущности имеют разумные названия полей. При этом важно помнить про стоимость join-ов — они всё равно превращаются в SQL-джоины.

Параметризация в JPQL защищает от SQL-инъекций и повышает повторноиспользуемость запроса. Есть **позиционные** (`?1`) и **именованные** (`:email`) параметры; на практике именованные удобнее и надёжнее при эволюции запроса. Параметры поддерживают коллекции (`in :ids`), перечисления и даже значения embeddable-типов, а провайдер корректно занимается конвертацией типов.

Типизированные запросы (`TypedQuery<T>`) — способ усилить безопасность на этапе компиляции: вы указываете ожидаемый тип результата (сущность или DTO-конструктор), и дальше работаете со строго типизированной коллекцией. Это проще тестировать и рефакторить, чем «сырые» `Query` с массивами объектов. В Spring Data типизация ещё удобнее через сигнатуры методов репозитория.

Именованные запросы (`@NamedQuery`) — способ закрепить JPQL на уровне сущности с «логическим» именем. Их парсинг и валидация происходят при старте приложения, что позволяет поймать ошибки раньше, чем в рантайме. Именованные запросы хорошо подходят для стабильных и часто используемых выборок, особенно если вы не хотите держать строки JPQL в репозиториях.

JPQL умеет возвращать не только сущности, но и **проекции**: через `select new com.example.dto.CustomerView(c.id, c.name)` вы сразу создаёте DTO «на лету». Это снижает трафик и нагрузку на Persistence Context (меньше управляемых сущностей), особенно в списочных экранах. Ограничение — конструктор DTO должен совпадать по аргументам, и вы не сможете «лениво» дотянуть ассоциации после.

Ещё одна полезная деталь: функции JPQL (`upper`, `lower`, `size`, `current_date`, `coalesce`) и доступ к enum-ам как к значениям (например, `where u.status = :status`). Часть функций провайдер маппит на диалект-специфичные выражения, но для сложных вещей разумнее уходить в нативный SQL.

В продакшн-коде JPQL чаще всего живёт либо в сигнатурах Spring Data репозиториев через `@Query`, либо в кастомных DAO/репозиториях, которые используют `EntityManager`. Во всех случаях придерживайтесь параметризации, избегайте конкатенации строк и логируйте итоговый SQL на dev-среде, чтобы понимать, что реально уходит в БД.

**Java — JPQL: именованный, типизированный и параметризованный запросы**

```java
package com.example.jpql.basic;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

// --- Сущности ---
@Entity
@Table(name = "customers")
@NamedQuery(
        name = "Customer.findActiveByEmailLike",
        query = "select c from Customer c where c.active = true and lower(c.email) like lower(:pattern)"
)
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true, length = 120)
    private String email;
    @Column(nullable = false) private String name;
    @Column(nullable = false) private boolean active = true;

    protected Customer() { }
    public Customer(String email, String name) { this.email = email; this.name = name; }
    public Long getId() { return id; }
    public String getEmail() { return email; }
    public String getName() { return name; }
    public boolean isActive() { return active; }
}

// --- Spring Data: JPQL в аннотации и конструкторная проекция ---
record CustomerView(Long id, String email) { }

@Repository
interface CustomerRepository extends JpaRepository<Customer, Long> {
    @Query("select new com.example.jpql.basic.CustomerView(c.id, c.email) " +
           "from Customer c where c.active = true and c.email like :pattern")
    List<CustomerView> findViewsByEmailLike(@org.springframework.data.repository.query.Param("pattern") String pattern);

    Optional<Customer> findByEmail(String email); // деривед-метод без JPQL
}

// --- Использование EntityManager: типизированный запрос + именованный ---
@Service
class CustomerFinder {
    @PersistenceContext private EntityManager em;

    @Transactional(readOnly = true)
    public List<Customer> findActiveByNamed(String pattern) {
        TypedQuery<Customer> q = em.createNamedQuery("Customer.findActiveByEmailLike", Customer.class);
        q.setParameter("pattern", pattern);
        return q.getResultList();
    }
}
```

**Kotlin — JPQL: именованный, типизированный и параметризованный запросы**

```kotlin
package com.example.jpql.basic

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "customers")
@NamedQuery(
    name = "Customer.findActiveByEmailLike",
    query = "select c from Customer c where c.active = true and lower(c.email) like lower(:pattern)"
)
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 120)
    var email: String = "",
    @Column(nullable = false)
    var name: String = "",
    @Column(nullable = false)
    var active: Boolean = true
)

data class CustomerView(val id: Long, val email: String)

@Repository
interface CustomerRepository : JpaRepository<Customer, Long> {
    @Query(
        "select new com.example.jpql.basic.CustomerView(c.id, c.email) " +
        "from Customer c where c.active = true and c.email like :pattern"
    )
    fun findViewsByEmailLike(@org.springframework.data.repository.query.Param("pattern") pattern: String): List<CustomerView>

    fun findByEmail(email: String): java.util.Optional<Customer>
}

@Service
class CustomerFinder(@PersistenceContext private val em: EntityManager) {
    @Transactional(readOnly = true)
    fun findActiveByNamed(pattern: String): List<Customer> =
        em.createNamedQuery("Customer.findActiveByEmailLike", Customer::class.java)
            .setParameter("pattern", pattern)
            .resultList
}
```

---

## Criteria API (кратко) и нативные SQL — где применимы и почему не злоупотреблять

Criteria API — это **программный конструктор JPQL**: вместо строки запроса вы строите дерево выражений через `CriteriaBuilder`, `CriteriaQuery`, `Root`, `Predicate`. Главный плюс — удобство динамики: вы можете условно добавлять фильтры, комбинировать предикаты, менять сортировку без конкатенации строк. Код становится длиннее, но безопаснее при рефакторинге (IDE подскажет имена полей).

Типичный сценарий для Criteria — поисковые экраны с кучей опциональных фильтров: от диапазонов дат до подстрочных совпадений и принадлежности множествам. С `CriteriaBuilder` вы собираете список предикатов `List<Predicate>` и передаёте их в `where(cb.and(...))`. Это читаемо и легко тестируется, если выделить конструктор предикатов в отдельный метод/класс.

Минусы Criteria — многословность и сложность отладки. Запрос «расползается» на десяток строк кода, и не всегда очевидно, какой SQL родится в итоге. Поэтому принято выносить Criteria в небольшие «спеки» (в Spring Data JPA — `Specification<T>`), покрывать их тестами и логировать SQL на dev-среде. Для простых запросов строковый JPQL часто понятнее.

Нативный SQL — ваш «выход в аварийный люк». Он нужен, когда JPQL не умеет требуемую конструкцию: оконные функции, CTE (`WITH`), специфичные функции СУБД, `jsonb`-операторы Postgres, `ON CONFLICT DO UPDATE` и т. п. В таких случаях честнее написать SQL и сразу получить нужную производительность, чем пытаться обойти ограничение ORM.

Но злоупотреблять нативом не стоит: вы теряете переносимость, сложнее тестировать и поддерживать маппинг результатов. Кроме того, нативные запросы легко «обходят» Persistence Context, и управляемые сущности могут остаться в неактуальном состоянии, если вы меняете данные в обход ORM. Минимизируйте побочные эффекты и используйте `clear()` при необходимости.

Для маппинга результатов нативного запроса есть опции: `createNativeQuery(sql)` возвращает `Object[]`, но лучше объявить `@SqlResultSetMapping` с `@ConstructorResult` или сразу маппить на интерфейс/DTO через Spring Data (`@Query(nativeQuery=true)`). Если вы возвращаете сущности — используйте `createNativeQuery(sql, Entity.class)`, но убедитесь, что набор колонок соответствует маппингу.

Натив и транзакции: «update native» попадёт под ту же транзакцию, что и ORM-операции, но JPA может не знать про изменения кэша 1-го уровня. В сценариях «native update» + «чтение тех же строк» добавьте `em.flush()` перед native и `em.clear()` после — или просто двигайте такие операции в отдельную транзакцию.

Итого: **Criteria** — когда нужна динамика и типобезопасность; **JPQL** — когда читабельность важнее; **native SQL** — когда нужен полный контроль и фичи СУБД. Не смешивайте всё подряд; пусть у каждого подхода будет понятная зона ответственности.

**Java — Criteria для динамического фильтра + нативный SQL с DTO**

```java
package com.example.query.criteria;

import jakarta.persistence.*;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

@Entity @Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String number;
    @Column(nullable = false) LocalDate createdAt;
    @Column(nullable = false) Long totalCents;
    protected Order() { }
    Order(String number, LocalDate createdAt, Long totalCents) {
        this.number = number; this.createdAt = createdAt; this.totalCents = totalCents;
    }
}

record OrderFilter(String numberLike, LocalDate from, LocalDate to, Long minTotalCents) { }

record OrderStat(String day, Long total) { }

@Repository
class OrderQueryDao {
    @PersistenceContext private EntityManager em;

    @Transactional(readOnly = true)
    public List<Order> findByFilter(OrderFilter f) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> root = cq.from(Order.class);

        List<Predicate> ps = new ArrayList<>();
        if (f.numberLike() != null) ps.add(cb.like(cb.lower(root.get("number")), "%" + f.numberLike().toLowerCase() + "%"));
        if (f.from() != null) ps.add(cb.greaterThanOrEqualTo(root.get("createdAt"), f.from()));
        if (f.to() != null) ps.add(cb.lessThanOrEqualTo(root.get("createdAt"), f.to()));
        if (f.minTotalCents() != null) ps.add(cb.ge(root.get("totalCents"), f.minTotalCents()));

        cq.where(cb.and(ps.toArray(new Predicate[0])));
        cq.orderBy(cb.desc(root.get("createdAt")));
        TypedQuery<Order> q = em.createQuery(cq);
        return q.getResultList();
    }

    @Transactional(readOnly = true)
    public List<OrderStat> dayStats(LocalDate from, LocalDate to) {
        String sql = """
            select to_char(created_at, 'YYYY-MM-DD') as day, sum(total_cents) as total
            from orders
            where created_at between :from and :to
            group by 1 order by 1
        """;
        @SuppressWarnings("unchecked")
        List<Object[]> rows = em.createNativeQuery(sql)
                .setParameter("from", from)
                .setParameter("to", to)
                .getResultList();
        return rows.stream()
                .map(r -> new OrderStat((String) r[0], ((Number) r[1]).longValue()))
                .toList();
    }
}
```

**Kotlin — Criteria для динамического фильтра + нативный SQL с DTO**

```kotlin
package com.example.query.criteria

import jakarta.persistence.*
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDate

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var number: String = "",
    @Column(nullable = false) var createdAt: LocalDate = LocalDate.now(),
    @Column(nullable = false) var totalCents: Long = 0
)

data class OrderFilter(
    val numberLike: String? = null,
    val from: LocalDate? = null,
    val to: LocalDate? = null,
    val minTotalCents: Long? = null
)

data class OrderStat(val day: String, val total: Long)

@Repository
class OrderQueryDao(@PersistenceContext private val em: EntityManager) {

    @Transactional(readOnly = true)
    fun findByFilter(f: OrderFilter): List<Order> {
        val cb = em.criteriaBuilder
        val cq = cb.createQuery(Order::class.java)
        val root = cq.from(Order::class.java)

        val ps = mutableListOf<Predicate>()
        f.numberLike?.let { ps += cb.like(cb.lower(root.get("number")), "%${it.lowercase()}%") }
        f.from?.let { ps += cb.greaterThanOrEqualTo(root.get("createdAt"), it) }
        f.to?.let { ps += cb.lessThanOrEqualTo(root.get("createdAt"), it) }
        f.minTotalCents?.let { ps += cb.ge(root.get("totalCents"), it) }

        cq.where(cb.and(*ps.toTypedArray()))
        cq.orderBy(cb.desc(root.get<Comparable<*>>("createdAt")))
        return em.createQuery(cq).resultList
    }

    @Transactional(readOnly = true)
    fun dayStats(from: LocalDate, to: LocalDate): List<OrderStat> {
        val sql = """
            select to_char(created_at, 'YYYY-MM-DD') as day, sum(total_cents) as total
            from orders
            where created_at between :from and :to
            group by 1 order by 1
        """.trimIndent()
        @Suppress("UNCHECKED_CAST")
        val rows = em.createNativeQuery(sql)
            .setParameter("from", from)
            .setParameter("to", to)
            .resultList as List<Array<Any>>
        return rows.map { OrderStat(it[0] as String, (it[1] as Number).toLong()) }
    }
}
```

---

## Пагинация: `setFirstResult/setMaxResults`, специфика подсчёта total-ов и сортировок

Пагинация нужна не только для UX, но и для производительности: отдавать по 10–50 записей на страницу — нормально, отдавать тысячи — путь к OOM и медленным ответам. В JPA есть два базовых механизма: «ручной» через `TypedQuery#setFirstResult/#setMaxResults` и высокоуровневый Spring Data `Pageable`. Оба в итоге превращаются в `LIMIT/OFFSET` (или аналоги) на уровне SQL.

Ключевая тонкость пагинации — **стабильная сортировка**. Если вы не зададите `ORDER BY`, база имеет право возвращать строки в любом порядке, и «следующая страница» будет гулять. Даже при сортировке по не-уникальному столбцу (например, `created_at`) добавьте «тай-брейкер» (обычно `id`) для детерминизма: `order by created_at desc, id desc`. Это защитит UI от «прыгающих» элементов между страницами.

Подсчёт общего количества (`total`) — отдельный запрос. В Spring Data для `@Query` можно указать `countQuery`, иначе фреймворк попытается «обернуть» ваш JPQL в `select count(...)` и может промахнуться (особенно с `distinct`, `group by`, `join fetch`). Вручной EM-подходе вы просто делаете второй JPQL/SQL с `count(*)`, без `fetch` и без сортировки.

Пагинация и `JOIN FETCH` почти не дружат для коллекций: join умножает строки, и `LIMIT/OFFSET` применяется к **умноженному** набору, а не к уникальным родителям. В итоге на странице окажется меньше родителей, чем `pageSize`, а общий `total` посчитать корректно сложно. Решение — либо отдельно загружать ids и потом догружать коллекции (`WHERE id IN (...)`), либо возвращать DTO.

При больших оффсетах (`offset = page*size`) производительность падает: база вынуждена «пропускать» множество строк. Для бесконечных лент используйте **keyset pagination** (a.k.a. seek): вместо номера страницы передавайте «последний видимый ключ» и делайте `where (created_at, id) < (:lastCreatedAt, :lastId) order by ... limit :size`. В Spring Data можно возвращать `Slice<T>` без `total`.

В транзакционном слое для «только чтение» сценариев используйте `@Transactional(readOnly = true)`, отключайте `open-in-view` и отдавайте наружу DTO. Это уменьшит накладные расходы на dirty checking и снизит риск ленивых инициализаций во вне-транзакционных местах.

Ещё одна практика — не возвращать сущности на списочных экранах вовсе. Делайте конструкторные проекции или интерфейсные — меньше данных, меньше работы контекста персистентности, проще жизнь GC. Сложные экраны можно закрывать нативным SQL с аккуратной проекцией.

И, наконец, не забывайте тестировать пагинацию на реальных объёмах и со стрессовыми сортировками (по полям с низкой селективностью). Профилируйте планы выполнения (`EXPLAIN ANALYZE`), добавляйте необходимые индексы по сортируемым колонкам, учитывайте локали/коллации при `ORDER BY name`.

**Java — пагинация через Spring Data (`Pageable`) и вручную через EM**

```java
package com.example.query.paging;

import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.util.List;

@Entity @Table(name = "articles")
class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String title;
    @Column(nullable = false) LocalDate publishedAt;
    protected Article() { }
    Article(String title, LocalDate publishedAt){ this.title=title; this.publishedAt=publishedAt; }
}

record ArticleView(Long id, String title, LocalDate publishedAt) { }

@Repository
interface ArticleRepository extends JpaRepository<Article, Long> {

    @Query(
        value = "select new com.example.query.paging.ArticleView(a.id, a.title, a.publishedAt) " +
                "from Article a where lower(a.title) like lower(concat('%', :q, '%'))",
        countQuery = "select count(a) from Article a where lower(a.title) like lower(concat('%', :q, '%'))"
    )
    Page<ArticleView> search(@Param("q") String query, Pageable pageable);
}

@Service
class ArticleService {
    private final ArticleRepository repo;
    @PersistenceContext private EntityManager em;
    ArticleService(ArticleRepository repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public Page<ArticleView> searchPaged(String q, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Order.desc("publishedAt"), Sort.Order.desc("id")));
        return repo.search(q, pageable);
    }

    @Transactional(readOnly = true)
    public List<ArticleView> lastN(LocalDate before, int size) {
        // Keyset-like: вручную через EM, без total
        String jpql = """
            select new com.example.query.paging.ArticleView(a.id, a.title, a.publishedAt)
            from Article a
            where a.publishedAt < :before
            order by a.publishedAt desc, a.id desc
        """;
        return em.createQuery(jpql, ArticleView.class)
                .setParameter("before", before)
                .setMaxResults(size)
                .getResultList();
    }
}
```

**Kotlin — пагинация через Spring Data (`Pageable`) и вручную через EM**

```kotlin
package com.example.query.paging

import jakarta.persistence.*
import org.springframework.data.domain.*
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDate

@Entity
@Table(name = "articles")
class Article(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = "",
    @Column(nullable = false) var publishedAt: LocalDate = LocalDate.now()
)

data class ArticleView(val id: Long, val title: String, val publishedAt: LocalDate)

@Repository
interface ArticleRepository : JpaRepository<Article, Long> {

    @Query(
        value = "select new com.example.query.paging.ArticleView(a.id, a.title, a.publishedAt) " +
                "from Article a where lower(a.title) like lower(concat('%', :q, '%'))",
        countQuery = "select count(a) from Article a where lower(a.title) like lower(concat('%', :q, '%'))"
    )
    fun search(@Param("q") query: String, pageable: Pageable): Page<ArticleView>
}

@Service
class ArticleService(
    private val repo: ArticleRepository,
    @PersistenceContext private val em: EntityManager
) {
    @Transactional(readOnly = true)
    fun searchPaged(q: String, page: Int, size: Int): Page<ArticleView> {
        val sort = Sort.by(Sort.Order.desc("publishedAt"), Sort.Order.desc("id"))
        val pageable = PageRequest.of(page, size, sort)
        return repo.search(q, pageable)
    }

    @Transactional(readOnly = true)
    fun lastN(before: LocalDate, size: Int): List<ArticleView> {
        val jpql = """
            select new com.example.query.paging.ArticleView(a.id, a.title, a.publishedAt)
            from Article a
            where a.publishedAt < :before
            order by a.publishedAt desc, a.id desc
        """.trimIndent()
        return em.createQuery(jpql, ArticleView::class.java)
            .setParameter("before", before)
            .setMaxResults(size)
            .resultList
    }
}
```

# 8. Spring Data JPA: репозитории

*Зависимости для всех примеров этой подтемы*

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
    // Для Kotlin-проектов добавьте нужные плагины Kotlin
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    java
    // Для Kotlin-проектов добавьте kotlin("jvm"), kotlin("plugin.spring"), kotlin("plugin.jpa")
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## Базовые интерфейсы: `CrudRepository`, `JpaRepository` — что дают «из коробки»

Первое, что даёт Spring Data JPA, — это готовые базовые интерфейсы репозиториев, реализованные за вас в рантайме. `CrudRepository<T, ID>` покрывает базовые CRUD-операции: `save`, `findById`, `existsById`, `findAll`, `count`, `deleteById`, `delete`. Эти методы работают одинаково для любой сущности и СУБД, снимая с вас обязанность писать однотипный код на JDBC/EntityManager вручную.

`JpaRepository<T, ID>` расширяет `CrudRepository` и `PagingAndSortingRepository`, добавляя JPA-специфику и удобства: `findAll(Pageable)`, `findAll(Sort)`, `saveAll`, `flush`, `saveAndFlush`, `deleteAllInBatch`, `getReferenceById`. Это особенно полезно, когда вам нужно принудительно синхронизировать контекст персистентности (`flush`) или работать с ленивыми прокси через `getReferenceById` без немедленной загрузки из базы.

Стоит понимать семантику `save`. Для сущности с `id == null` (новой) будет выполнен `INSERT`, а для сущности с ненулевым `id` — `UPDATE` (или, если в контексте есть managed-экземпляр, произойдёт merge). Это удобно, но накладывает ответственность: «полу-заполненные» объекты с id могут перезаписать данные. Часто безопаснее разделять команды на «create» и «update» явно в бизнес-слое.

Ещё одна удобная деталь — поддержка `Optional` и коллекций. Методы репозиториев возвращают `Optional<T>` для «одной» записи и `List<T>`/`Page<T>` для множественных. Это формирует понятный контракт API и заставляет явно обрабатывать «не найдено». В Kotlin можно возвращать `T?` и `List<T>` — Spring Data корректно конвертирует.

Репозитории в Spring — бины, участвующие в переводе исключений в `DataAccessException`. Это значит, что вы получаете унифицированную иерархию ошибок независимо от конкретной СУБД и провайдера JPA. Поверх этого удобно строить централизованный хэндлинг ошибок и метрики.

Класс `SimpleJpaRepository` — дефолтная реализация, которую Spring генерирует за вас. Если нужно поведение «по-особенному», можно создать кастомный базовый класс для всех репозиториев через `@EnableJpaRepositories(repositoryBaseClass=...)` и, например, добавить общий метод аудита или хинты для read-only запросов.

Наконец, инфраструктура Spring Data упрощает тестируемость. Вы можете поднимать срез `@DataJpaTest`, инжектить репозитории и проверять контракты методов без поднятия всего приложения. В сочетании с Testcontainers легко получить предсказуемые интеграционные тесты против реального Postgres.

**Java — минимальная сущность и репозиторий на `JpaRepository`**

```java
package com.example.repo.basic;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Entity
@Table(name = "customers")
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 120)
    private String email;

    @Column(nullable = false, length = 120)
    private String name;

    protected Customer() { }
    public Customer(String email, String name) { this.email = email; this.name = name; }

    public Long getId() { return id; }
    public String getEmail() { return email; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}

@Repository
interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByEmail(String email);
    List<Customer> findAllByName(String name);
}

class CustomerService {
    private final CustomerRepository repo;
    CustomerService(CustomerRepository repo) { this.repo = repo; }

    @Transactional
    public Long create(String email, String name) {
        return repo.save(new Customer(email, name)).getId();
    }

    @Transactional(readOnly = true)
    public Optional<Customer> findByEmail(String email) {
        return repo.findByEmail(email);
    }
}
```

**Kotlin — аналогичный пример**

```kotlin
package com.example.repo.basic

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.util.*

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 120)
    var email: String = "",
    @Column(nullable = false, length = 120)
    var name: String = ""
)

@Repository
interface CustomerRepository : JpaRepository<Customer, Long> {
    fun findByEmail(email: String): Optional<Customer>
    fun findAllByName(name: String): List<Customer>
}

class CustomerService(private val repo: CustomerRepository) {
    @Transactional
    fun create(email: String, name: String): Long = repo.save(Customer(email = email, name = name)).id!!

    @Transactional(readOnly = true)
    fun findByEmail(email: String): Optional<Customer> = repo.findByEmail(email)
}
```

---

## Derived queries по имени метода: конвенции, операторы (`And/Or/Between/In/Like/Exists/OrderBy`)

Derived queries — это механизм построения запросов по имени метода. Spring разбирает имя, выделяет предикаты и операторы и строит JPQL/SQL автоматически. Например, `findByEmail` превратится в `where email = :email`, а `findByNameAndActiveTrueOrderByCreatedAtDesc` — в сложный запрос с несколькими условиями и сортировкой. Это резко ускоряет разработку «поисковых» методов без ручного JPQL.

Грамматика имён богата: `And`, `Or` комбинируют предикаты; `Between` работает для сравнимых типов (даты/числа); `In` принимает коллекцию; `Like`, `Containing`, `StartingWith`, `EndingWith` дают разные варианты поиска по подстроке; `IsNull`/`IsNotNull` — для пустых значений; `True`/`False` — для булевых флагов без параметров; `Before`/`After` — для дат. Для `Like` вы сами отвечаете за `%` в параметре, а `Containing` вставляет его за вас.

Существуют количественные модификаторы: `Top`, `First`, например, `findTop3ByOrderByScoreDesc`. Они генерируют `limit` в SQL и удобны для «топ-N» виджетов. Есть и `Distinct`, если нужно устранить дубликаты на уровне запроса. За детерминизм результата отвечайте сортировкой в имени метода через `OrderBy...Asc/Desc`.

Для проверки наличия записей используется `existsBy...`. Это проще и эффективнее, чем `countBy... > 0`, потому что фреймворк строит оптимизированный `exists`-запрос. Для подсчёта — `countBy...` возвращает `long`. Возвращаемые типы могут быть `Optional<T>`, `List<T>`, `Stream<T>` (с осторожностью), `Slice<T>`/`Page<T>` при наличии `Pageable`.

Свойства можно «сквозь» связи: `findByDepartmentName` будет джоинить `department` и фильтровать по `name`. Но имена должны совпадать с полями сущности; IDE с автодополнением здесь очень помогает. Если свойство называется «жаргонно» или регулярно меняется, derived-подход становится хрупким — тогда лучше `@Query`.

Подводные камни: пустые коллекции в `In` (поведение зависит от провайдера; лучше защищать кодом и не вызывать такие методы с пустым списком), локали и регистр в `Like`/`Containing` (используйте `IgnoreCase`: `findByEmailContainingIgnoreCase`), и длинные имена методов, которые трудно читать. Разумная грань — простой поиск и фильтры; сложные отчёты — в JPQL/SQL.

`Pageable` и `Sort` можно передавать параметром даже в derived-методах. Например, `findByActiveTrue(Pageable pageable)` позволит клиенту решать, как сортировать и какую страницу получать. Важно предусмотреть стабильную сортировку по умолчанию, если `Sort` не передан, — желательно через перегруженный метод с заранее заданным `Sort`.

**Java — репозиторий с derived-методами**

```java
package com.example.repo.derived;

import jakarta.persistence.*;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Slice;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDate;
import java.util.Collection;
import java.util.List;
import java.util.Optional;

@Entity
@Table(name = "users")
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false, unique = true, length = 120) String email;
    @Column(nullable = false, length = 80) String name;
    @Column(nullable = false) boolean active = true;
    @Column(nullable = false) LocalDate createdAt = LocalDate.now();
    protected User() { }
    User(String email, String name){ this.email = email; this.name = name; }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    List<User> findByNameContainingIgnoreCaseAndActiveTrueOrderByCreatedAtDesc(String name);

    List<User> findTop3ByActiveTrueOrderByCreatedAtDesc();

    List<User> findByCreatedAtBetween(LocalDate from, LocalDate to);

    List<User> findByEmailIn(Collection<String> emails);

    boolean existsByEmailIgnoreCase(String email);

    Slice<User> findByActiveTrue(Pageable pageable);
}
```

**Kotlin — аналогичный репозиторий**

```kotlin
package com.example.repo.derived

import jakarta.persistence.*
import org.springframework.data.domain.Pageable
import org.springframework.data.domain.Slice
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.time.LocalDate
import java.util.*

@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 120)
    var email: String = "",
    @Column(nullable = false, length = 80)
    var name: String = "",
    @Column(nullable = false)
    var active: Boolean = true,
    @Column(nullable = false)
    var createdAt: LocalDate = LocalDate.now()
)

@Repository
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): Optional<User>
    fun findByNameContainingIgnoreCaseAndActiveTrueOrderByCreatedAtDesc(name: String): List<User>
    fun findTop3ByActiveTrueOrderByCreatedAtDesc(): List<User>
    fun findByCreatedAtBetween(from: LocalDate, to: LocalDate): List<User>
    fun findByEmailIn(emails: Collection<String>): List<User>
    fun existsByEmailIgnoreCase(email: String): Boolean
    fun findByActiveTrue(pageable: Pageable): Slice<User>
}
```

---

## `@Query` (JPQL/SQL) и проекции: интерфейсные/DTO (базово)

Когда derived-методов становится мало, подключаем `@Query`. Она позволяет явно написать JPQL или нативный SQL. JPQL хорош тем, что оперирует сущностями и переносим между СУБД, нативный SQL — когда нужны особенности конкретной базы (оконные функции, `jsonb`, CTE). В обоих случаях параметры должны быть именованными: это безопаснее и читаемее.

Проекции нужны, чтобы не тащить целые сущности, когда достаточно части полей. Есть **интерфейсные проекции**: вы объявляете интерфейс с геттерами (`getId()`, `getEmail()`), и Spring маппит результат запроса по именам колонок. Есть **конструкторные DTO-проекции** («select new ...Dto(...)») — строгие и быстрые; подходят, когда вы заранее знаете набор полей. Оба подхода уменьшают давление на Persistence Context и экономят память/время.

Интерфейсные проекции бывают закрытыми (только геттеры) и открытыми (можно добавить SpEL-выражения, но это дороже). На практике лучше использовать закрытые: они быстрее и предсказуемее. Если имена полей в результате не совпадают с геттерами, используйте `as alias` в JPQL/SQL.

При пагинации кастомных запросов не забывайте про `countQuery`. Если не указать, Spring попытается «обернуть» JPQL в `select count(...)`, но с `distinct`, `group by`, `fetch` это часто неправильно. Явно задайте простой `countQuery` без сортировок и `fetch` — так и быстрее, и точнее.

Для нативных запросов `@Query(nativeQuery=true)` позволяет вернуть либо массивы объектов/мапы, либо проекции (при аккуратных алиасах). Для обновляющих запросов добавляйте `@Modifying` и оборачивайте в транзакцию. Помните, что native-update может устарить кэш 1-го уровня — после него иногда уместно `clear()`.

Хорошая практика — держать читабельность. Если JPQL/SQL становится длинным, выносите его в многострочные литералы Java/Kotlin (как в примерах) и давайте понятные имена методам. Для совсем сложных вещей часто приятнее завести «кастомный репозиторий» с реализацией на `EntityManager`.

И не забывайте о стабильности контрактов. Проекции — часть вашего API между слоями; изменение полей или их названий ломает вызовов. Покрывайте такие методы интеграционными тестами и держите SQL/JPQL рядом с DTO/интерфейсом.

**Java — `@Query` c JPQL, интерфейсная и DTO-проекция, плюс нативный запрос**

```java
package com.example.repo.query;

import jakarta.persistence.*;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Modifying;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;

@Entity
@Table(name = "articles")
class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String title;
    @Column(nullable = false) LocalDate publishedAt = LocalDate.now();
    protected Article() { }
    Article(String title){ this.title = title; }
}

public interface ArticleView {
    Long getId();
    String getTitle();
    LocalDate getPublishedAt();
}

record ArticleDto(Long id, String title, LocalDate publishedAt) { }

@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {

    // Интерфейсная проекция (JPQL)
    @Query("""
            select a.id as id, a.title as title, a.publishedAt as publishedAt
            from Article a
            where lower(a.title) like lower(concat('%', :q, '%'))
            """)
    Page<ArticleView> searchViews(@Param("q") String query, Pageable pageable);

    // Конструкторная проекция (JPQL)
    @Query("""
            select new com.example.repo.query.ArticleDto(a.id, a.title, a.publishedAt)
            from Article a
            where a.publishedAt >= :from
            order by a.publishedAt desc, a.id desc
            """)
    Page<ArticleDto> recentDtos(@Param("from") LocalDate from, Pageable pageable);

    // Нативный SQL (например, специфичный сорт)
    @Query(
        value = """
            select id, title, published_at
            from articles
            where title ilike concat('%', :q, '%')
            order by published_at desc, id desc
            """,
        countQuery = """
            select count(*) from articles
            where title ilike concat('%', :q, '%')
            """,
        nativeQuery = true
    )
    Page<Object[]> nativeSearch(@Param("q") String query, Pageable pageable);

    // Обновление (нужны @Modifying и транзакция)
    @Modifying
    @Transactional
    @Query("update Article a set a.title = :title where a.id = :id")
    int rename(@Param("id") Long id, @Param("title") String newTitle);
}
```

**Kotlin — те же идеи**

```kotlin
package com.example.repo.query

import jakarta.persistence.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Modifying
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDate

@Entity
@Table(name = "articles")
class Article(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = "",
    @Column(nullable = false) var publishedAt: LocalDate = LocalDate.now()
)

interface ArticleView {
    fun getId(): Long
    fun getTitle(): String
    fun getPublishedAt(): LocalDate
}

data class ArticleDto(val id: Long, val title: String, val publishedAt: LocalDate)

@Repository
interface ArticleRepository : JpaRepository<Article, Long> {

    @Query(
        """
        select a.id as id, a.title as title, a.publishedAt as publishedAt
        from Article a
        where lower(a.title) like lower(concat('%', :q, '%'))
        """
    )
    fun searchViews(@Param("q") query: String, pageable: Pageable): Page<ArticleView>

    @Query(
        """
        select new com.example.repo.query.ArticleDto(a.id, a.title, a.publishedAt)
        from Article a
        where a.publishedAt >= :from
        order by a.publishedAt desc, a.id desc
        """
    )
    fun recentDtos(@Param("from") from: LocalDate, pageable: Pageable): Page<ArticleDto>

    @Query(
        value = """
            select id, title, published_at
            from articles
            where title ilike concat('%', :q, '%')
            order by published_at desc, id desc
        """,
        countQuery = """
            select count(*) from articles
            where title ilike concat('%', :q, '%')
        """,
        nativeQuery = true
    )
    fun nativeSearch(@Param("q") query: String, pageable: Pageable): Page<Array<Any>>

    @Modifying
    @Transactional
    @Query("update Article a set a.title = :title where a.id = :id")
    fun rename(@Param("id") id: Long, @Param("title") newTitle: String): Int
}
```

---

## Пагинация и сортировка: `Pageable/Sort`, стабильные сортировки и контракт API

Пагинация в Spring Data строится вокруг `Pageable` и возвращаемых типов `Page<T>`/`Slice<T>`. `Page` содержит не только элементы, но и метаданные (`totalElements`, `totalPages`, `number`, `size`, `sort`). Это удобно для UI, но создаёт дополнительный «count»-запрос. `Slice` ограничивается знанием «есть ли следующая страница» и экономит на подсчёте — полезно для бесконечных лент и скролла.

`Pageable` создаётся через `PageRequest.of(page, size, Sort...)`. Индексация страниц начинается с нуля — это важно проговорить в контракте API. Размер страницы должен быть ограничен сервером (например, максимум 200), иначе клиент может запросить «слишком много» и положить базу. Ограничение задайте в контроллере/сервисе и зафиксируйте в документации.

Сортировка задаётся объектом `Sort` или прямо в `PageRequest`. Важно обеспечить **стабильный порядок**: если сортируете по полю с повторяющимися значениями (например, `createdAt`), добавляйте вторичный ключ — чаще всего `id`. Так вы избежите «пляшущих» записей между страницами при одинаковых значениях основного поля.

При кастомных `@Query` методы, возвращающие `Page`, требуют отдельного `countQuery`. Без него Spring пытается «вычислить» `count` автоматически, что не всегда корректно (особенно с `distinct`/`group by`). Простое правило: если у вас есть `value=...` — почти всегда добавляйте `countQuery=...` без сортировок и с минимальным набором join-ов.

Пагинация плохо сочетается с `JOIN FETCH` коллекций: join умножает строки, и лимит применяется к умноженному набору, из-за чего на «странице» окажется меньше уникальных родительских записей. Решения — двухшаговая загрузка (сначала id родителей, затем записи по `IN (:ids)`), либо возвращение сразу DTO без fetch-join. Это нужно учитывать ещё на стадии проектирования API.

Контракт API должен быть предсказуем: какие поля разрешены для сортировки, как обрабатываются невалидные поля/направления, какое максимальное значение `size`, как интерпретируется `page` (0-based). Лучше явно валидировать `Sort` и маппить публичные имена полей к доменным, чтобы не раскрывать внутреннюю структуру.

Ещё одна полезная опция — «бесконечный скролл» через keyset-пагинацию (seek). Это не `Pageable`, а отдельный контракт: клиент отправляет «курсоры» (`lastCreatedAt`, `lastId`), а вы отдаёте следующий блок `where (created_at, id) < (:lastCreatedAt, :lastId) order by created_at desc, id desc`. Он стабилен и быстр на больших объёмах, но требует отдельной реализации.

**Java — пагинация и сортировка через `Pageable` и `Sort`**

```java
package com.example.repo.paging;

import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;

@Entity
@Table(name = "tickets")
class Ticket {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String subject;
    @Column(nullable = false) LocalDate createdAt = LocalDate.now();
    @Column(nullable = false) boolean open = true;
    protected Ticket() { }
    Ticket(String subject){ this.subject = subject; }
}

record TicketView(Long id, String subject, LocalDate createdAt, boolean open) { }

@Repository
interface TicketRepository extends JpaRepository<Ticket, Long> {

    Page<Ticket> findByOpenTrue(Pageable pageable);

    @Query(
        value = """
            select new com.example.repo.paging.TicketView(t.id, t.subject, t.createdAt, t.open)
            from Ticket t
            where (:q is null or lower(t.subject) like lower(concat('%', :q, '%')))
            """,
        countQuery = """
            select count(t)
            from Ticket t
            where (:q is null or lower(t.subject) like lower(concat('%', :q, '%')))
            """
    )
    Page<TicketView> search(@Param("q") String query, Pageable pageable);
}

@Service
class TicketService {
    private final TicketRepository repo;
    TicketService(TicketRepository repo) { this.repo = repo; }

    @Transactional(readOnly = true)
    public Page<TicketView> searchPaged(String q, int page, int size) {
        int safeSize = Math.min(Math.max(size, 1), 100);
        Sort sort = Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"));
        Pageable pageable = PageRequest.of(Math.max(page, 0), safeSize, sort);
        return repo.search(q, pageable);
    }

    @Transactional(readOnly = true)
    public Page<Ticket> listOpen(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("id").descending());
        return repo.findByOpenTrue(pageable);
    }
}
```

**Kotlin — аналогичный пример**

```kotlin
package com.example.repo.paging

import jakarta.persistence.*
import org.springframework.data.domain.*
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDate

@Entity
@Table(name = "tickets")
class Ticket(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var subject: String = "",
    @Column(nullable = false) var createdAt: LocalDate = LocalDate.now(),
    @Column(nullable = false) var open: Boolean = true
)

data class TicketView(val id: Long, val subject: String, val createdAt: LocalDate, val open: Boolean)

@Repository
interface TicketRepository : JpaRepository<Ticket, Long> {

    fun findByOpenTrue(pageable: Pageable): Page<Ticket>

    @Query(
        value = """
            select new com.example.repo.paging.TicketView(t.id, t.subject, t.createdAt, t.open)
            from Ticket t
            where (:q is null or lower(t.subject) like lower(concat('%', :q, '%')))
        """,
        countQuery = """
            select count(t)
            from Ticket t
            where (:q is null or lower(t.subject) like lower(concat('%', :q, '%')))
        """
    )
    fun search(@Param("q") query: String?, pageable: Pageable): Page<TicketView>
}

@Service
class TicketService(private val repo: TicketRepository) {

    @Transactional(readOnly = true)
    fun searchPaged(q: String?, page: Int, size: Int): Page<TicketView> {
        val safeSize = size.coerceIn(1, 100)
        val sort = Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"))
        val pageable = PageRequest.of(page.coerceAtLeast(0), safeSize, sort)
        return repo.search(q, pageable)
    }

    @Transactional(readOnly = true)
    fun listOpen(page: Int, size: Int): Page<Ticket> {
        val pageable = PageRequest.of(page, size, Sort.by("id").descending())
        return repo.findByOpenTrue(pageable)
    }
}
```

# 9. Транзакции и границы консистентности

*Зависимости (для всех подпунктов ниже одинаковые)*

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    java
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**application.yml (базовые рекомендации)**

```yaml
spring:
  jpa:
    open-in-view: false        # держим транзакционные границы узкими
    properties:
      hibernate:
        format_sql: true
        jdbc.time_zone: UTC
```

---

## `@Transactional` на сервисном слое: чтение (`readOnly=true`) и модификации, когда нужен отдельный транзакционный метод

Транзакции — это единицы атомарной работы с данными. В Spring с JPA за них отвечает `JpaTransactionManager`; вы объявляете границы через `@Transactional`, а инфраструктура берёт на себя открытие/фиксацию/откат. Правильная практика — аннотировать **сервисный слой**: именно он знает бизнес-инварианты и когда набор операций должен быть атомарным.

Для операций чтения используйте `@Transactional(readOnly = true)`. Этот флаг даёт два эффекта: во-первых, Spring подсказывает провайдеру не выполнять `flush` (это экономит накладные расходы на dirty checking), во-вторых, некоторые драйверы/БД могут оптимизировать план, помечая соединение как read-only. Это не «защита от записей», но хороший намёк на оптимизацию.

Для операций записи применяйте «полноценный» `@Transactional` без `readOnly`. Внутри такой транзакции вы можете изменять управляемые сущности, полагаясь на dirty checking и каскады. Снаружи — вы получаете атомарность: либо всё коммитится, либо откатывается при ошибке. Это радикально упрощает бизнес-код: не нужно вручную думать о порядке `INSERT/UPDATE/DELETE`.

Границы транзакции должны совпадать с бизнес-операцией. Например, «создать заказ» включает в себя сохранение заказа, строк, проверку лимитов и резервирование — всё это одна транзакция. «Отправить email» по итогам создания заказа — уже не часть этой транзакции; такие побочные эффекты лучше выносить в отдельные механизмы (outbox, события) или хотя бы в `REQUIRES_NEW`.

Ключевая ловушка — **self-invocation**: прокси транзакций создаются вокруг бинов Spring, поэтому вызов `this.inner()` внутри того же класса **не** активирует новую транзакцию с особыми атрибутами. Если вам нужен отдельный транзакционный метод с другими свойствами (например, `REQUIRES_NEW`), вынесите его в другой бин и вызывайте через DI.

Поведение по умолчанию для `@Transactional` — `propagation = REQUIRED`: если транзакция уже есть, метод войдёт в неё; если нет — будет создана. Это удобно в 90% случаев. Исключения — аудиты/логирование/интеграции, которые должны зафиксироваться независимо (используйте `REQUIRES_NEW`) или, наоборот, не должны участвовать в транзакции (`NOT_SUPPORTED`).

`readOnly = true` не запрещает писать в БД, но Hibernate в таком режиме может оптимистично отключать отслеживание грязных изменений. Если вы случайно модифицируете сущность в read-only транзакции, поведение будет зависеть от версии и настроек; не рассчитывайте на сохранение. Это дополнительный аргумент «не смешивать» чтения и записи в одном методе.

Изолированность (isolation) в JPA задаётся на уровне менеджера транзакций/БД. По умолчанию для Postgres это `READ COMMITTED`: внутри одной транзакции вы видите только зафиксированные чужие изменения. Для JPA это редко настраивают в аннотациях, но знать уровень полезно — особенно при отчётах с долгими чтениями.

Ещё одна практика — отделять «командные» методы (изменяющие состояние) от «запросных» (читающих). Это снижает риски случайного `flush` и помогает сформировать правильные API. В названиях методов и классах сервиса это отражается как `*Service` и `*QueryService`.

Наконец, думайте о времени жизни транзакции. Чем длиннее транзакция, тем выше вероятность блокировок, конфликтов и деградации пула соединений. Делайте бизнес-операции короткими, не включайте в транзакцию внешний I/O (HTTP, файловая система) и не выносите сущности за её пределы.

**Java — сервисный слой: чтение/запись и отдельная транзакция для аудита**

```java
package com.example.txn.service;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.*;

import java.time.Instant;
import java.util.Optional;

@Entity
@Table(name = "accounts")
class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String owner;
    @Column(nullable = false) Long balance;
    protected Account() {}
    Account(String owner, long balance){ this.owner = owner; this.balance = balance; }
}

@Repository
interface AccountRepository extends JpaRepository<Account, Long> { }

@Entity
@Table(name = "audit_log")
class AuditLog {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) Instant ts = Instant.now();
    @Column(nullable = false) String action;
    @Column(nullable = false) String details;
    protected AuditLog() {}
    AuditLog(String action, String details){ this.action = action; this.details = details; }
}

@Repository
interface AuditRepository extends JpaRepository<AuditLog, Long> { }

@Service
class AuditService {
    private final AuditRepository auditRepo;
    AuditService(AuditRepository auditRepo){ this.auditRepo = auditRepo; }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void write(String action, String details) {
        auditRepo.save(new AuditLog(action, details)); // фиксируется отдельно
    }
}

@Service
class AccountService {
    private final AccountRepository repo;
    private final AuditService audit;
    AccountService(AccountRepository repo, AuditService audit){ this.repo = repo; this.audit = audit; }

    @Transactional(readOnly = true)
    public Optional<Account> get(Long id) {
        return repo.findById(id); // без flush, только чтение
    }

    @Transactional
    public void deposit(Long id, long amount) {
        Account a = repo.findById(id).orElseThrow();
        a.balance += amount; // dirty checking
        audit.write("DEPOSIT", "account=" + id + ", amount=" + amount); // отдельная транзакция
    }
}
```

**Kotlin — аналогичный сервисный слой**

```kotlin
package com.example.txn.service

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.*
import java.time.Instant
import java.util.*

@Entity
@Table(name = "accounts")
class Account(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var owner: String = "",
    @Column(nullable = false) var balance: Long = 0
)

@Repository
interface AccountRepository : JpaRepository<Account, Long>

@Entity
@Table(name = "audit_log")
class AuditLog(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var ts: Instant = Instant.now(),
    @Column(nullable = false) var action: String = "",
    @Column(nullable = false) var details: String = ""
)

@Repository
interface AuditRepository : JpaRepository<AuditLog, Long>

@Service
class AuditService(private val auditRepo: AuditRepository) {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun write(action: String, details: String) {
        auditRepo.save(AuditLog(action = action, details = details))
    }
}

@Service
class AccountService(
    private val repo: AccountRepository,
    private val audit: AuditService
) {
    @Transactional(readOnly = true)
    fun get(id: Long): Optional<Account> = repo.findById(id)

    @Transactional
    fun deposit(id: Long, amount: Long) {
        val a = repo.findById(id).orElseThrow()
        a.balance += amount
        audit.write("DEPOSIT", "account=$id, amount=$amount")
    }
}
```

---

## Видимость изменений внутри сессии/контекста, когда происходит `flush` и как это связано с commit

Внутри транзакции действует **контекст персистентности** (Persistence Context): управляемые сущности живут в первом уровне кеша, и любые изменения их полей немедленно «видны» остальному коду, который держит на них ссылки. Эта видимость локальна: пока не произошло `flush`, БД ничего не знает об изменениях, но в памяти у вас уже «новое» состояние.

`flush` — это синхронизация контекста с БД: Hibernate вычисляет дельту (dirty checking) и выполняет `INSERT/UPDATE/DELETE` в правильном порядке. По умолчанию (`FlushModeType.AUTO`) `flush` происходит **перед выполнением JPQL/SQL-запроса**, чтобы запрос увидел актуальные данные, и, конечно, перед фиксацией транзакции. Это значит, что даже внутри одного метода `SELECT` может спровоцировать лишние `UPDATE`-ы, если вы до этого меняли сущности.

`commit` — это завершение транзакции на уровне БД: все изменения, отправленные `flush`, фиксируются и становятся видимыми другим транзакциям. Важно помнить, что `flush` может происходить много раз внутри одной транзакции, а `commit` — один раз в конце. Поэтому «ошибки уникальности» или «нарушения внешнего ключа» вы поймаете уже на `flush`, а не на `commit`.

Иногда полезно отложить синхронизацию до конца транзакции: `em.setFlushMode(COMMIT)` или `@Transactional` со стратегией, где вы сами вызываете `em.flush()` в нужной точке. Это уменьшает количество неожиданных `flush` перед `SELECT`, но и влечёт риск прочитать «старые» данные в рамках той же транзакции, если вы рассчитывали увидеть свои изменения через запрос. Такой режим используют там, где сначала делают много записей, а потом один раз коммитят.

Видимость внутри контекста и в БД может расходиться. Пример: вы изменили баланс счёта и тут же сделали `repo.findById(id)`. Если `flush` не случился, `findById` вернёт **тот же управляемый экземпляр** из контекста (без похода в БД) — вы увидите новые поля. Но если вы выполните JPQL-запрос `select sum(balance) from Account`, то при `AUTO` произойдёт `flush`, и запрос увидит изменения уже с диска.

Важно понимать разницу между **refresh** и **clear**. `refresh(entity)` перечитает состояние сущности из БД, отбрасывая локальные несохранённые изменения (спасает после внешнего апдейта или при желании убедиться в текущих данных). `clear()` очищает весь контекст: все сущности становятся detached, и повторные чтения пойдут в БД. Эти операции редко нужны в CRUD, но незаменимы в батч-процессинге.

Ещё одна тонкость — смешивание ORM и нативного SQL. Если вы делаете `createNativeQuery("update ...")` в той же транзакции, Persistence Context не узнает про изменения, пока вы не сделаете `clear()` или `refresh`. Поэтому после массовых апдейтов нативом имеет смысл очистить контекст, чтобы не работать со «старыми» объектами.

Изоляция транзакций задаёт, что вы видите из «внешнего мира». В Postgres с `READ COMMITTED` в одной транзакции два последовательных `SELECT` могут вернуть разные данные, если кто-то закоммитил изменения между ними. Но внутри вашего контекста управляемые сущности сохраняют свои значения до `refresh`/`clear`.

`flush` дорог: это реальное выполнение SQL. Частые `flush` внутри цикла (например, из-за запросов) могут ударить по производительности. Если вы точно знаете, что читать будете позже, установите `COMMIT` и вызывайте `em.flush()` вручную в конце, либо аккуратно перестройте код, чтобы не вызывать `SELECT` до нужного момента.

В логике бизнес-операций полезно помнить, что «поймать» ошибки ограничений лучше раньше. Явный `em.flush()` в середине метода позволяет выявить конфликт уникальности до внешнего http-запроса или публикации сообщения. Это делает откаты более предсказуемыми и сокращает время удержания блокировок.

**Java — демонстрация flush против commit и локальной видимости**

```java
package com.example.txn.flush;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class FlushDemoService {
    @PersistenceContext private EntityManager em;

    @Transactional
    public void writeManyThenQuery() {
        em.setFlushMode(FlushModeType.COMMIT); // откладываем flush до конца
        for (int i = 0; i < 3; i++) {
            em.persist(new Note("n" + i));
        }
        // Этот select НЕ спровоцирует flush — увидим только старые строки из БД
        long countBefore = (Long) em.createQuery("select count(n) from Note n").getSingleResult();

        // Принудительно синхронизируемся и увидим новые записи в БД
        em.flush();
        long countAfter = (Long) em.createQuery("select count(n) from Note n").getSingleResult();

        // countBefore < countAfter — демонстрация влияния flush
    }
}

@Entity @Table(name = "notes")
class Note {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String title;
    protected Note() {}
    Note(String title){ this.title = title; }
}
```

**Kotlin — аналогичный пример**

```kotlin
package com.example.txn.flush

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class FlushDemoService(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun writeManyThenQuery() {
        em.flushMode = FlushModeType.COMMIT
        repeat(3) { em.persist(Note(title = "n$it")) }

        val countBefore = em.createQuery("select count(n) from Note n", java.lang.Long::class.java)
            .singleResult

        em.flush()

        val countAfter = em.createQuery("select count(n) from Note n", java.lang.Long::class.java)
            .singleResult

        // countBefore < countAfter
    }
}

@Entity
@Table(name = "notes")
class Note(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = ""
)
```

---

## Исключения и rollback-правила: unchecked по умолчанию; почему не ловить и «глотать» ошибки

По умолчанию Spring откатывает транзакцию на **unchecked** исключениях (`RuntimeException` и их наследниках) и на `Error`. Checked-исключения (наследники `Exception`, но не `RuntimeException`) транзакцию **не** откатывают, если вы явно не попросили. Это историческая договорённость, и о ней важно помнить при проектировании контрактов иерархий ошибок.

Отсюда простой принцип: бизнес-ошибки, которые требуют отката, оформляйте как runtime-исключения. Если вы хотите использовать checked-исключения по стилистическим причинам, укажите `@Transactional(rollbackFor = Exception.class)` на методе/классе — иначе получите «тихий коммит» при ошибке и неконсистентное состояние.

Не «глотайте» исключения. Конструкция `try { ... } catch (Exception e) { log.warn(...); }` внутри `@Transactional` приведёт к **коммиту** — фреймворк решит, что всё хорошо, раз исключение обработано. Если вам нужно добавить контекст, **оборачивайте** ошибку и бросайте дальше (`throw new IllegalStateException("...", e)`), или явно помечайте транзакцию как «rollback-only».

Существует API `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`, но злоупотреблять им не стоит. Гораздо понятнее довериться декларативной модели: бросили runtime — будет откат; хотите не откатывать — перехватили и обработали вне транзакции или использовали `noRollbackFor`.

`noRollbackFor` полезен, когда бизнес-исключение нужно вернуть клиенту как 4xx, но изменения всё равно должны сохраниться (редкий случай; чаще наоборот). Например, вы выполняете две независимые операции внутри одной транзакции, и падение второй — не повод откатывать первую. В таких случаях лучше разбить на две транзакции (`REQUIRES_NEW`) или пересмотреть инварианты.

Перевод исключений слоём данных (`DataAccessException`) — мощная фича Spring. Например, нарушение уникальности превратится в `DataIntegrityViolationException`, от которого удобно плясать в контроллерах/хэндлерах ошибок. Но помните: исключение приходит **после** `flush`. Если вы «ждёте» ошибку раньше, сделайте явный `em.flush()`.

Не используйте checked-исключения для «control flow» внутри транзакции: это ломает предсказуемость отката и усложняет код. Если нужно вернуть пользователю «ошибка бизнес-валидации», бросьте своё `DomainException extends RuntimeException` и обработайте его на уровне Web через `@ControllerAdvice`.

Внешние операции (HTTP-вызовы, Kafka) нельзя «включать» в одну транзакцию JPA. Если вы шлёте событие после коммита, используйте outbox-паттерн или `TransactionSynchronization` (`afterCommit`) — это гарантирует отправку только после фиксации. Пытаться «перехватить исключение и всё равно отправить» — анти-паттерн.

В тестах полезно проверять, что транзакции действительно откатываются. В `@DataJpaTest` по умолчанию всё откатывается в конце теста — это удобно для изоляции. Для сервиса же пишите интеграционные тесты, в которых проверяете, что на `RuntimeException` записи не остаются, а на checked при отсутствии `rollbackFor` — остаются (чтобы осознанно выбрать модель).

**Java — правила отката, checked/unchecked и «не глотать» исключения**

```java
package com.example.txn.rollback;

import jakarta.persistence.*;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "users", uniqueConstraints = @UniqueConstraint(columnNames = "email"))
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false, unique = true) String email;
    protected User() {}
    User(String email){ this.email = email; }
}

@Repository
interface UserRepository extends JpaRepository<User, Long> { }

class BusinessException extends RuntimeException {
    public BusinessException(String message) { super(message); }
}

@Service
class UserService {
    private final UserRepository repo;
    UserService(UserRepository repo){ this.repo = repo; }

    // Откатится на BusinessException (runtime)
    @Transactional
    public void createOrFail(String email) {
        repo.save(new User(email));
        if (email.endsWith("@blocked.com")) {
            throw new BusinessException("Domain is blocked");
        }
    }

    // Checked исключение НЕ откатит транзакцию без rollbackFor
    @Transactional(rollbackFor = Exception.class)
    public void createOrThrowChecked(String email) throws Exception {
        repo.save(new User(email));
        if (email.contains("x")) {
            throw new Exception("Checked exception should rollback due to rollbackFor");
        }
    }

    // Не глотаем исключение: добавляем контекст и пробрасываем дальше
    @Transactional
    public void createHandlingDuplicate(String email) {
        try {
            repo.saveAndFlush(new User(email)); // заставим flush, чтобы поймать уникальность здесь
        } catch (DataIntegrityViolationException e) {
            throw new BusinessException("Email already exists: " + email);
        }
    }
}
```

**Kotlin — те же идеи**

```kotlin
package com.example.txn.rollback

import jakarta.persistence.*
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "users", uniqueConstraints = [UniqueConstraint(columnNames = ["email"])])
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true) var email: String = ""
)

@Repository
interface UserRepository : JpaRepository<User, Long>

class BusinessException(message: String) : RuntimeException(message)

@Service
class UserService(private val repo: UserRepository) {

    @Transactional
    fun createOrFail(email: String) {
        repo.save(User(email = email))
        if (email.endsWith("@blocked.com")) {
            throw BusinessException("Domain is blocked")
        }
    }

    @Transactional(rollbackFor = [Exception::class])
    @Throws(Exception::class)
    fun createOrThrowChecked(email: String) {
        repo.save(User(email = email))
        if (email.contains("x")) {
            throw Exception("Checked exception should rollback due to rollbackFor")
        }
    }

    @Transactional
    fun createHandlingDuplicate(email: String) {
        try {
            repo.saveAndFlush(User(email = email))
        } catch (e: DataIntegrityViolationException) {
            throw BusinessException("Email already exists: $email")
        }
    }
}
```

# 10. Ограничения и инварианты данных

*Зависимости (для всех подпунктов ниже одинаковые)*

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    java
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**application.yml (рекомендуемые настройки)**

```yaml
spring:
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate  # проверяем соответствие маппинга схеме
    properties:
      hibernate:
        format_sql: true
        jdbc.time_zone: UTC
```

---

## Ограничения БД как последняя линия обороны: PK/FK/UNIQUE/CHECK; синхронизация с моделью JPA

Первое, что стоит зафиксировать в голове: **инварианты домена должны быть защищены не только кодом приложения, но и схемой БД**. Даже если у вас идеальные DTO-валидации и сервисные проверки, всё равно есть гонки, интеграции, ручные SQL-правки и миграции. Поэтому ограничения уровня базы — это последняя и самая надёжная линия обороны, которая гарантирует корректность данных независимо от ошибок в Java-коде.

Базовые кирпичи ограничений — первичные ключи (PK), внешние ключи (FK), уникальные индексы/ограничения (UNIQUE) и произвольные логические условия (CHECK). PK обеспечивает уникальную идентификацию строк и задаёт физическую «опору» для ссылок. FK сохраняет ссылочную целостность между таблицами — без него «висячие» ссылки и драматические потери времени в расследованиях. UNIQUE фиксирует бизнес-уникальности (например, email). CHECK — универсальный механизм, которым можно запретить отрицательный баланс или ограничить допустимые значения статуса.

С JPA важно синхронизировать аннотации маппинга и реальные ограничения. Пример: `@Column(nullable = false)` должен соответствовать `NOT NULL` в схеме; `@Table(uniqueConstraints = ...)` — реальному уникальному индексу. Но помните: **DDL, генерируемый Hibernate, не должен применяться в продакшне** — он удобен на деве, а в проде используем миграции (Flyway/Liquibase). Мы держим `ddl-auto: validate`, чтобы приложение падало при расхождениях.

Уникальные ограничения требуют внимания к «частичным» условиям. Частый кейс — уникальный email среди активных пользователей, но не среди удалённых (soft delete). В Postgres это решается **частичным индексом** `where deleted_at is null`. В JPA нет прямой аннотации для частичных индексов — их создаём миграцией, а в коде документируем поведение через комментарии/тесты.

CHECK-ограничения дают гибкость и производительность. Например, можно гарантировать, что `total_cents >= 0` и `jsonb_typeof(settings) = 'object'`. Валидация на уровне БД загорится в самый поздний момент — при `flush`. Это не отменяет ранние сообщения об ошибках пользователю, но предотвращает «тихий» занос мусора при байпассах.

FK — не только про `ON DELETE`. Важно договориться о политике каскадов: `ON DELETE RESTRICT` чаще всего безопаснее, чем `CASCADE`, чтобы случайное удаление родителя не уничтожило «полгорода». Каскады JPA — одна история, а каскады БД — другая; они не обязаны совпадать. В критичных местах лучше запретить каскад на уровне БД и управлять удалениями явно.

Согласованность типов — ещё один аспект синхронизации. Если в коде `@Column(precision=19, scale=4)` на `BigDecimal`, в схеме должно быть `numeric(19,4)`. Несоответствие приводит к округлениям, silent-триммингам и ошибкам индексов (например, `varchar(120)` vs фактические 255). Включённый `ddl-auto: validate` поможет поймать это на старте.

Договоритесь в команде: **ограничения на уровне БД — обязательны**, даже если кажется, что «мы всё проверяем в коде». Нельзя полагаться только на JPA-валидации и исключения: их можно обойти другими процессами. Привяжите миграции к CI/CD, запускайте smoke-проверку `validate` на старте и не пропускайте ворнинги.

В тестах полезно симулировать нарушения ограничений, чтобы убедиться, что ваши `@ControllerAdvice`/ошибки слоя сервисов корректно переводят `DataIntegrityViolationException` в понятные ответы API (например, 409 Conflict для уникальности). Это делает поведение предсказуемым и улучшает DX клиентов.

Наконец, документируйте инварианты прямо в сущностях и миграциях. Короткий комментарий «UNIQUE(email) partial (deleted_at is null)» и Javadoc на поле избавят следующего разработчика от гадания. В моделях, ориентированных на долгую жизнь, такие «подписи на чертеже» бесценны.

**SQL (Flyway V1__init.sql) — PK/FK/UNIQUE/CHECK, частичный уникальный индекс**

```sql
-- V1__init.sql  (PostgreSQL)
create table users
(
    id           bigserial primary key,
    email        varchar(120) not null,
    name         varchar(120) not null,
    deleted_at   timestamptz,
    created_at   timestamptz not null default now(),
    constraint chk_user_email_format check (position('@' in email) > 1)
);
-- частичный уникальный индекс: email уникален среди не удалённых
create unique index uq_users_email_active on users (lower(email)) where deleted_at is null;

create table orders
(
    id          bigserial primary key,
    user_id     bigint not null,
    total_cents bigint not null,
    constraint fk_orders_user foreign key (user_id) references users(id) on delete restrict,
    constraint chk_order_total check (total_cents >= 0)
);
```

**Java — сущности, соответствующие ограничениям, и репозиторий**

```java
package com.example.constraints;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.Instant;
import java.util.Optional;

@Entity
@Table(name = "users",
       uniqueConstraints = {
           // В БД у нас частичный индекс; аннотация фиксирует намерение,
           // но реальное условие реализовано миграцией.
           @UniqueConstraint(name = "uq_users_email_annot", columnNames = {"email"})
       })
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String email;

    @Column(nullable = false, length = 120)
    private String name;

    @Column(name = "deleted_at")
    private Instant deletedAt;

    protected User() {}
    public User(String email, String name) { this.email = email; this.name = name; }

    public Long getId(){ return id; }
    public String getEmail(){ return email; }
    public void setEmail(String email){ this.email = email; }
    public String getName(){ return name; }
    public void setName(String name){ this.name = name; }
    public Instant getDeletedAt(){ return deletedAt; }
    public void softDelete(){ this.deletedAt = Instant.now(); }
}

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false, foreignKey = @ForeignKey(name = "fk_orders_user"))
    private User user;

    @Column(name = "total_cents", nullable = false)
    private long totalCents;

    protected Order() {}
    public Order(User user, long totalCents) { this.user = user; this.totalCents = totalCents; }

    public Long getId(){ return id; }
    public User getUser(){ return user; }
    public long getTotalCents(){ return totalCents; }
    public void setTotalCents(long v){ this.totalCents = v; }
}

@Repository
interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmailIgnoreCaseAndDeletedAtIsNull(String email);
}
```

**Kotlin — те же сущности**

```kotlin
package com.example.constraints

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import java.time.Instant
import java.util.*

@Entity
@Table(
    name = "users",
    uniqueConstraints = [UniqueConstraint(name = "uq_users_email_annot", columnNames = ["email"])]
)
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, length = 120)
    var email: String = "",

    @Column(nullable = false, length = 120)
    var name: String = "",

    @Column(name = "deleted_at")
    var deletedAt: Instant? = null
) {
    fun softDelete() { deletedAt = Instant.now() }
}

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false, foreignKey = ForeignKey(name = "fk_orders_user"))
    var user: User? = null,

    @Column(name = "total_cents", nullable = false)
    var totalCents: Long = 0
)

@Repository
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmailIgnoreCaseAndDeletedAtIsNull(email: String): Optional<User>
}
```

---

## Валидация сущностей: аннотации Jakarta Validation, группы и сообщения

Валидация на уровне приложения — это **первая линия обороны** и дружелюбная обратная связь пользователю. Спецификация Jakarta Bean Validation (ранее JSR-303/JSR-380) предоставляет декларативные аннотации (`@NotNull`, `@Size`, `@Email`, `@Pattern`, `@Min`, `@Past`, …) и инфраструктуру проверки через `Validator`. В экосистеме Spring проверка автоматически запускается при использовании `@Valid` или `@Validated` на аргументах контроллера/сервиса.

Аннотации ставятся на поля/геттеры сущностей и DTO. Это позволяет выразить инварианты «как код»: длина имени, формат email, допустимые границы чисел. В отличие от ограничений БД, здесь мы можем вернуть понятные сообщения и привязать их к конкретным полям формы. Валидация происходит раньше, чем попытка `flush`, что экономит ресурсы и улучшает UX.

Группы валидации — механизм, позволяющий **включать разные подмножества правил** в зависимости от сценария (создание vs обновление). Например, при создании пароль обязателен, а при обновлении — опционален. Мы объявляем интерфейсы групп (`OnCreate`, `OnUpdate`) и указываем их в аннотациях и на точке валидации (`@Validated(OnCreate.class)`).

Сообщения об ошибках можно настраивать. По умолчанию используется бандл `ValidationMessages.properties`. Мы можем задавать placeholders (`{javax.validation.constraints.Size.message}`) и собственные коды (`user.email.invalid`). В Spring коды попадают в `MessageSource`, поддерживается локализация. Это особенно важно для публичных API и UI.

Существует **каскадная валидация** через `@Valid` на вложенных объектах/коллекциях. Если у пользователя есть список адресов, аннотации на полях `Address` будут проверены автоматически. Это помогает держать корректными большие графы данных без ручного обхода.

Для доменных правил, которые нельзя выразить стандартными аннотациями, пишется **кастомное ограничение**: аннотация + `ConstraintValidator`. Например, «сильный пароль», «дата окончания позже даты начала», «телефон в международном формате». В валидатор можно инжектить сервисы (в Spring) и выполнять внешние проверки — но внимательно к производительности.

Методная валидация (`@Validated` на классе сервиса) запускает проверку на **входных и выходных** параметрах методов. Это удобно для доменных сервисов: вы декларируете контракты (`@NotNull` на возвращаемом значении, диапазоны на параметрах) и ловите нарушения раньше, чем они попадут в глубокие слои. Для этого нужно включить `MethodValidationPostProcessor` (в Boot он уже подключён).

Валидация — не замена ограничениям в БД. Даже если вы всё проверили, в гонке два запроса могут пройти проверки и оба попытаться вставить одинаковый email. Поэтому мы комбинируем: валидации для UX, ограничения БД — для консистентности, обработчики исключений — для аккуратного перевода `DataIntegrityViolationException` в 409/422.

Не забывайте о перформансе. Каскадная валидация больших коллекций может быть дорогой; иногда для внутренних сервисов достаточно валидации DTO на входе, а сущности внутри не аннотировать (или ограничить набор правил). Правило простое: валидируйте границы системы и места, где данные меняются.

Итоговое правило: **валидация, затем попытка записи, затем обработка ошибок БД**. Такой «тройной барьер» минимизирует мусор в логах и даёт пользователю понятные причины отказа. Тестируйте сценарии: пустые поля, неправильный формат, дубликаты, каскадные ошибки — всё это должно быть покрыто.

**Java — сущность/DTO с валидациями, группы, кастомный валидатор «сильного пароля»**

```java
package com.example.validation;

import jakarta.persistence.*;
import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

import java.lang.annotation.*;
import java.util.List;

public interface OnCreate {}
public interface OnUpdate {}

@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Email(message = "{user.email.invalid}")
    @NotBlank(groups = OnCreate.class)
    @Column(nullable = false, unique = true, length = 120)
    private String email;

    @NotBlank(groups = OnCreate.class)
    @Size(min = 8, max = 100, groups = OnCreate.class)
    @StrongPassword(groups = OnCreate.class)
    @Transient // пароль не храним как есть; это поле для DTO-слияния
    private String rawPassword;

    @NotBlank @Size(max = 120)
    private String name;

    @Valid
    @ElementCollection
    @CollectionTable(name = "user_phones", joinColumns = @JoinColumn(name = "user_id"))
    @Column(name = "phone", length = 20)
    private List<@Pattern(regexp = "\\+\\d{10,15}") String> phones;

    protected User(){}

    // геттеры/сеттеры опущены для краткости
}

@Documented
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
@interface StrongPassword {
    String message() default "{user.password.weak}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    @Override public boolean isValid(String value, ConstraintValidatorContext c) {
        if (value == null) return true; // @NotBlank ловит null/empty
        boolean hasUpper = value.chars().anyMatch(Character::isUpperCase);
        boolean hasLower = value.chars().anyMatch(Character::isLowerCase);
        boolean hasDigit = value.chars().anyMatch(Character::isDigit);
        boolean hasSpec  = value.chars().anyMatch(ch -> "!@#$%^&*()-_+=[]{}".indexOf(ch) >= 0);
        return hasUpper && hasLower && hasDigit && hasSpec;
    }
}
```

**Kotlin — аналог с группами и кастомным валидатором**

```kotlin
package com.example.validation

import jakarta.persistence.*
import jakarta.validation.Valid
import jakarta.validation.Payload
import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.constraints.*

interface OnCreate
interface OnUpdate

@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @field:Email(message = "{user.email.invalid}")
    @field:NotBlank(groups = [OnCreate::class])
    @Column(nullable = false, unique = true, length = 120)
    var email: String = "",

    @Transient
    @field:NotBlank(groups = [OnCreate::class])
    @field:Size(min = 8, max = 100, groups = [OnCreate::class])
    @field:StrongPassword(groups = [OnCreate::class])
    var rawPassword: String? = null,

    @field:NotBlank
    @field:Size(max = 120)
    var name: String = "",

    @ElementCollection
    @CollectionTable(name = "user_phones", joinColumns = [JoinColumn(name = "user_id")])
    @Column(name = "phone", length = 20)
    var phones: MutableList<@Pattern(regexp = "\\+\\d{10,15}") String> = mutableListOf()
)

@MustBeDocumented
@Target(AnnotationTarget.FIELD, AnnotationTarget.PROPERTY_GETTER, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [StrongPasswordValidator::class])
annotation class StrongPassword(
    val message: String = "{user.password.weak}",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class StrongPasswordValidator : ConstraintValidator<StrongPassword, String?> {
    override fun isValid(value: String?, context: ConstraintValidatorContext): Boolean {
        val v = value ?: return true
        val hasUpper = v.any { it.isUpperCase() }
        val hasLower = v.any { it.isLowerCase() }
        val hasDigit = v.any { it.isDigit() }
        val hasSpec = v.any { "!@#$%^&*()-_+=[]{}".contains(it) }
        return hasUpper && hasLower && hasDigit && hasSpec
    }
}
```

---

## Оптимистическая блокировка `@Version`: базовое применение для защиты от перезаписей

Оптимистическая блокировка — это способ защититься от **потерянных обновлений** при параллельных изменениях. Вместо тяжёлых блокировок на уровне БД мы добавляем версионное поле в сущность (`@Version`) и включаем в `UPDATE` условие по этой версии. Если кто-то уже поменял запись, наш `UPDATE` затронет 0 строк, и JPA выбросит `OptimisticLockException` (в Spring — `ObjectOptimisticLockingFailureException`).

Семантика проста: при чтении вы получаете сущность с `version = N`. При коммите Hibernate генерирует `update ... set ..., version = N+1 where id = ? and version = N`. Если условие не выполняется (кто-то уже сделал `N+1`), значит, данные устарели. Это честный и масштабируемый способ координации конкурентных обновлений без эксклюзивных блокировок.

Тип поля версии — `int`, `long`, `Integer`, `Long` или `Instant`/`Timestamp` (Hibernate 6 поддерживает версию как время). Числовые типы — самый понятный вариант. Значение заполняется и инкрементируется провайдером автоматически; вручную трогать поле не нужно.

Где применять: в агрегатах, которые часто редактируются многими пользователями или потоками — карточка заказа, профиль, настройки. Особенно важно при «редактировании формы» в UI: два оператора открыли одну запись, первый сохранил, второй поверх переписал — без версий вы потеряете изменения. С версиями второй запрос получит 409/412, перечитает данные и предложит слить.

Как обрабатывать конфликт: на сервисном уровне ловим `ObjectOptimisticLockingFailureException` и транслируем в понятный ответ (обычно 409 Conflict с указанием текущей версии и, возможно, диффом полей). На уровне UI — показываем баннер «запись изменилась, перезагрузите» или применяем стратегию merge.

Оптимистическая блокировка работает и для графов с каскадами: изменение «детей» также поднимет версию родителя, если обновляется сам объект. Но важно понимать: версия висит на **конкретной сущности**, а не на всём графе. Если у вас много взаимосвязанных таблиц в одном агрегате, часто удобно держать версию только на корне и обновлять через него.

Пара слов о перформансе: `@Version` добавляет одно поле в таблицу и одно условие в `WHERE`. Это дёшево. Основные накладные расходы — обработка конфликтов. Поэтому если запись редактируют редко и последовательно, оптимистическая блокировка почти «не стоит ничего». При постоянных конфликтах, возможно, нужен другой дизайн (например, очереди команд, pessimistic lock).

Отличайте оптимистические и пессимистические блокировки (`@Lock(PESSIMISTIC_WRITE)`). Пессимистическая захватывает блокировку в БД («держит» строку), защищая от параллельных апдейтов ценой снижения конкурентности и рисков дедлоков. Это крайняя мера и нужна значительно реже, чем кажется.

Наконец, помните о тестах. Смоделируйте гонку: загрузите одну и ту же запись в двух транзакциях, обновите в первой, попытайтесь сохранить во второй — убедитесь, что ловите исключение и переводите его в корректный ответ. Это типичная регрессия при рефакторинге.

**Java — пример `@Version`, сервис и обработка конфликта**

```java
package com.example.versioning;

import jakarta.persistence.*;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "profiles")
class Profile {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @Version
    Long version;

    @Column(nullable = false, length = 120)
    String name;

    @Column(nullable = false, length = 120)
    String email;

    protected Profile(){}
    Profile(String name, String email){ this.name=name; this.email=email; }
}

@Repository
interface ProfileRepository extends JpaRepository<Profile, Long> { }

@Service
class ProfileService {
    private final ProfileRepository repo;
    ProfileService(ProfileRepository repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public Profile get(Long id) { return repo.findById(id).orElseThrow(); }

    @Transactional
    public void rename(Long id, Long expectedVersion, String newName) {
        Profile p = repo.findById(id).orElseThrow();
        if (!p.version.equals(expectedVersion)) {
            throw new OptimisticLockingFailureException("Version mismatch: expected=" + expectedVersion + ", actual=" + p.version);
        }
        p.name = newName; // dirty checking + @Version -> WHERE ... AND version=?
    }
}

// Пример хэндлера (фрагмент): перевод в 409 Conflict
@org.springframework.web.bind.annotation.ControllerAdvice
class ErrorAdvice {
    @org.springframework.web.bind.annotation.ExceptionHandler(OptimisticLockingFailureException.class)
    @org.springframework.web.bind.annotation.ResponseStatus(HttpStatus.CONFLICT)
    @org.springframework.web.bind.annotation.ResponseBody
    public java.util.Map<String,Object> onOptimistic(OptimisticLockingFailureException e){
        return java.util.Map.of("type","optimistic_conflict","detail", e.getMessage());
    }
}
```

**Kotlin — аналог `@Version` и сервис**

```kotlin
package com.example.versioning

import jakarta.persistence.*
import org.springframework.dao.OptimisticLockingFailureException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "profiles")
class Profile(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Version
    var version: Long? = null,

    @Column(nullable = false, length = 120)
    var name: String = "",

    @Column(nullable = false, length = 120)
    var email: String = ""
)

@Repository
interface ProfileRepository : JpaRepository<Profile, Long>

@Service
class ProfileService(private val repo: ProfileRepository) {

    @Transactional(readOnly = true)
    fun get(id: Long): Profile = repo.findById(id).orElseThrow()

    @Transactional
    fun rename(id: Long, expectedVersion: Long, newName: String) {
        val p = repo.findById(id).orElseThrow()
        if (p.version != expectedVersion) {
            throw OptimisticLockingFailureException("Version mismatch: expected=$expectedVersion, actual=${p.version}")
        }
        p.name = newName
    }
}

@org.springframework.web.bind.annotation.ControllerAdvice
class ErrorAdvice {
    @org.springframework.web.bind.annotation.ExceptionHandler(OptimisticLockingFailureException::class)
    @org.springframework.web.bind.annotation.ResponseStatus(HttpStatus.CONFLICT)
    @org.springframework.web.bind.annotation.ResponseBody
    fun onOptimistic(e: OptimisticLockingFailureException): Map<String, Any> =
        mapOf("type" to "optimistic_conflict", "detail" to e.message.orEmpty())
}
```

# 11. Генерация схемы и миграции

## `spring.jpa.hibernate.ddl-auto` (обзор) — когда *нельзя* использовать в проде

В экосистеме Spring Boot параметр `spring.jpa.hibernate.ddl-auto` управляет тем, как Hibernate синхронизирует модель сущностей со схемой БД при старте приложения. Фактически это «тонкая обёртка» вокруг свойства Hibernate `hibernate.hbm2ddl.auto`. Оно удобно на этапе прототипирования, но опасно в продуктивной среде, где схема — контракт между версиями приложения, инструментами аналитики и внешними интеграциями. В этой части разберём значения, типичные ловушки и безопасные альтернативы.

Значение `create` заставляет Hibernate выкинуть существующие таблицы и создать их заново из аннотаций. Этот режим имеет смысл только на локальных стендах и временных окружениях, где данные не представляют ценности. Любой запуск с `create` на общей БД приведёт к потере данных. Даже если у вас есть бэкапы, восстановление займёт время и нарушит SLA.

Значение `create-drop` похоже на `create`, но дополнительно удалит схему при остановке приложения. Это чуть удобнее для быстрых интеграционных стендов или демо, где нужно «начисто» пересобрать структуру по окончании работы. В продакшне `create-drop` столь же опасен, как и `create`: потеря данных гарантирована при рестарте или падении контейнера.

Значение `update` кажется заманчивым: Hibernate «мягко» пытается донастроить схему под аннотации — добавить колонки, создать отсутствующие таблицы. Проблема в том, что он не умеет корректно выполнять разрушающие изменения: сокращение `length`, изменение `precision/scale`, переименование столбцов, пересоздание индексов, перенос внешних ключей. Более того, поведение зависит от диалекта БД и версии Hibernate — на одном стенде «прокатило», на другом — нет. Итог: «дрейф схемы», когда у разных окружений структура расходится.

Значение `validate` — единственный вариант, который можно рассматривать в продакшне. Он **не меняет** схему, а только проверяет соответствие маппинга текущей БД и падает при расхождениях. Это полезно, чтобы не пропустить версию приложения, где сущности и миграции «разошлись». Но `validate` — не панацея: он не проверяет всё (например, частичные индексы и многие нюансы), поэтому должен идти **вместе** с управляемыми миграциями.

Значение `none` в Boot просто отключает установку `ddl-auto`, передавая ответственность вам (и Hibernate остаётся без hbm2ddl вообще). На проде это окей, но на деве неудобно, если вы рассчитываете на автоматическое создание структуры. В любом случае, для командной разработки лучше выбрать управляемые миграции вместо магии `create`/`update`.

Есть ещё скрытые риски. Во-первых, «случайный» профиль: разработчик собрал артефакт с `application-dev.yml` внутри, а в контейнере активировался `dev` — и БД «обнулилась». Во-вторых, разные диалекты: на локальном H2 `update` сработал «правильно», а на Postgres — иначе. В-третьих, аннотации часто меняют колонку «незаметно» (например, `@Enumerated` ORDINAL→STRING), а `update` создаёт несовместимые значения. Всё это аргументы против «самогенерации» схемы на проде.

Правильная привычка — держать `ddl-auto: validate` в продакшн-профилях, а на деве пользоваться миграциями так же, как и на проде. Если очень хочется ускорить R&D, можно включать `create-drop` только в юнит/слайс-тестах `@DataJpaTest` и в локальных запускалках с изолированной БД. Чем меньше у вас «особых правил для дева», тем надёжнее релизы.

Иногда в проекте смешивают `update` и миграции: «пусть Hibernate создаёт новые колонки, а Flyway делает сложное». Это неверная практика: вы теряете детерминизм — какая именно часть схемы получилась и за счёт чего. В итоге один стенд «чуть отличается» от другого, а баги носят непредсказуемый характер. Делайте выбор: **либо** управляемые миграции, **либо** жизненный цикл схемы должен быть полностью под контролем DBA. В типичном приложении выбор — миграции.

Наконец, не забывайте про контейнеризацию. Если у вас `docker-compose` поднимает Postgres с томом, и вы запускаете приложение с `create`, таблицы будут пересозданы в томе — данные потеряются. В Kubernetes `initContainers` и миграции должны управлять схемой до старта приложения, а `ddl-auto` стоит оставить на `validate` или вовсе выключить.

**application.yml/профили: безопасные настройки**

```yaml
# application-prod.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate   # только проверка соответствия
    properties:
      hibernate:
        format_sql: true
        jdbc.time_zone: UTC
```

```yaml
# application-dev.yml
spring:
  jpa:
    hibernate:
      ddl-auto: none       # на деве тоже лучше миграции; см. подтему про Flyway/Liquibase
    properties:
      hibernate:
        format_sql: true
  datasource:
    url: jdbc:postgresql://localhost:5432/app
    username: app
    password: app
```

**Java — программное задание ddl-режима по профилям (для демонстрации, не обязательно):**

```java
package com.example.ddlmode;

import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer;
import org.springframework.context.annotation.*;

import java.util.Map;

@Configuration
@Profile("dev")
public class DevHibernateDdlConfig implements HibernatePropertiesCustomizer {
    @Override
    public void customize(Map<String, Object> hibernateProps) {
        // Только для локального R&D, не для общего dev/qa
        hibernateProps.put("hibernate.hbm2ddl.auto", "create-drop");
    }
}

@Configuration
@Profile("prod")
class ProdHibernateDdlConfig implements HibernatePropertiesCustomizer {
    @Override
    public void customize(Map<String, Object> hibernateProps) {
        hibernateProps.put("hibernate.hbm2ddl.auto", "validate");
    }
}
```

**Kotlin — то же самое:**

```kotlin
package com.example.ddlmode

import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile

@Configuration
@Profile("dev")
class DevHibernateDdlConfig : HibernatePropertiesCustomizer {
    override fun customize(hibernateProps: MutableMap<String, Any>) {
        hibernateProps["hibernate.hbm2ddl.auto"] = "create-drop"
    }
}

@Configuration
@Profile("prod")
class ProdHibernateDdlConfig : HibernatePropertiesCustomizer {
    override fun customize(hibernateProps: MutableMap<String, Any>) {
        hibernateProps["hibernate.hbm2ddl.auto"] = "validate"
    }
}
```

**Gradle (Groovy/Kotlin) — базовые зависимости:**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

---

## Правильный путь: миграции через Flyway/Liquibase, согласование типов и индексов с маппингом

Миграции — это управляемые изменения схемы БД, оформленные в версионированных скриптах, которые применяются последовательно и одинаково на всех окружениях. Ключевой эффект — детерминизм: любая версия приложения знает, какую схему она ожидает, а процесс деплоя прозрачен и воспроизводим. В мире Spring Boot два де-факто стандарта — **Flyway** (SQL-скрипты) и **Liquibase** (описание в YAML/XML/JSON или SQL). Оба инструмента интегрированы в Boot и легко конфигурируются.

Начнём с Flyway. Он ищет скрипты в `classpath:db/migration` с именованием `V<версия>__<описание>.sql` и ведёт служебную таблицу `flyway_schema_history`. При старте приложения Flyway сравнивает текущую историю с ресурсами и применяет недостающие миграции в алфавитно-версионном порядке. Скрипты — обычный SQL, что удобно для использования всех фишек Postgres (частичные индексы, `jsonb`, CTE, `generated as identity`, `index concurrently` и т. д.).

Liquibase ближе к декларативной модели: вы описываете изменения как «changeSet» в YAML/JSON/XML или даёте «formatted SQL». Оно строит диалектоспецифичный SQL и так же ведёт служебную таблицу. Преимущество — вынос повторяющихся шаблонов и более выразительные «изменения», недостаток — иногда сложнее добиться тонких оптимизаций (например, `CONCURRENTLY` в индексах). На практике команды часто выбирают Flyway за прозрачность SQL, либо Liquibase, если ценят декларативность.

Ключ к прочному контракту — **согласование типов** между миграциями и маппингом JPA. Если в сущности `@Column(precision = 19, scale = 4) BigDecimal`, в миграции должен быть `numeric(19,4)`. Если вы храните UUID — используйте тип `uuid` и не забывайте расширение `uuid-ossp` (или генерируйте в приложении). Для временных меток предпочтительнее `timestamptz` с явным UTC в JDBC. Любая рассинхронизация приведёт к «немым» округлениям, триммингам и странным багам.

Не менее важно **согласование индексов**. Параметр `@Column(unique = true)` полезен как документация, но надёжнее создавать индексы миграцией: так вы можете задать **частичные условия** (`where deleted_at is null`), коллации, выражения (`(lower(email))`), опции `CONCURRENTLY`. Именно миграции определяют, где и какие индексы действительно есть в продакшн-схеме.

Процесс деплоя должен запускать миграции **до** старта основного приложения. В Boot это достигается автоматически (Flyway/Liquibase запускается как автоконфигурация), но в Kubernetes часто делают отдельный `initContainer`, который выполняет миграции и лишь затем поднимают приложение. Такой подход устраняет гоночные условия при нескольких репликах первого старта.

Управление версиями включает опции `baselineOnMigrate` (если схема уже есть, но не под контролем Flyway) и `outOfOrder` (разрешить применять «пропущенные» миграции). Используйте их осознанно: baseline — при «подсаживании» на существующую БД, out-of-order — только в исключительных случаях (горячие фиксы), иначе теряется детерминизм.

Проверяйте миграции на реальных объёмах. Например, `CREATE INDEX CONCURRENTLY` безопасен для продакшна, потому что не блокирует запись, но его нельзя выполнять в транзакции — значит, в Flyway-скрипте нужно отключить транзакцию (`flyway:executeInTransaction=false` в Java API или просто писать отдельные скрипты, если используете чистый SQL в Postgres — Flyway сам не будет оборачивать DDL в транзакцию для `CONCURRENTLY`). Для тяжёлых ALTER-ов планируйте «двухшаговые» миграции: добавить новую колонку/индекс, заполнить данные, потом переключить код.

Для тестов и локалки полезно запускать те же миграции, что и на проде. Это легко: просто подключите Flyway как зависимость, положите скрипты в `db/migration`, и Boot применит их при старте. Никаких `ddl-auto=create`; вся схема живёт в SQL. Это дисциплинирует команду и убирает класс багов «на проде другая схема».

Конфигурируйте очистку и ремонт истории осознанно. `flyway.clean` должен быть выключен на продакшене политики безопасности, чтобы никто случайно не «снес» схему. `flyway.repair` чинит «битую» историю (например, при ручной правке скрипта) — полезно, но запускайте его только как управляемую операцию.

Наконец, держите миграции рядом с кодом и покрывайте критичные изменения интеграционными тестами с Testcontainers. Тогда вы будете уверены, что новая версия сущностей действительно работает на схеме «как на проде», а не на «рандомной» структуре, которую нагенерил `update`.

**Gradle (Groovy/Kotlin) — Flyway и/или Liquibase:**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.flywaydb:flyway-core:10.17.0'
    // Либо Liquibase:
    // implementation 'org.liquibase:liquibase-core:4.29.2'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.flywaydb:flyway-core:10.17.0")
    // или Liquibase:
    // implementation("org.liquibase:liquibase-core:4.29.2")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**application.yml — включаем Flyway и настраиваем поведение:**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true  # если садимся на уже существующую БД (первый раз)
    clean-disabled: true       # безопасность на проде
```

**Flyway V1__init.sql — типы и индексы, согласованные с JPA:**

```sql
-- PostgreSQL

create extension if not exists "uuid-ossp";

create table customers
(
    id          uuid primary key default uuid_generate_v4(),
    email       varchar(120) not null,
    name        varchar(120) not null,
    status      varchar(16)  not null,
    balance     numeric(19,4) not null default 0,
    created_at  timestamptz not null default now()
);

-- частичная уникальность по активным
create unique index uq_customers_email_active
    on customers (lower(email)) where status = 'ACTIVE';

create table orders
(
    id           bigserial primary key,
    customer_id  uuid not null references customers(id) on delete restrict,
    total_cents  bigint not null check (total_cents >= 0),
    created_at   timestamptz not null default now()
);

-- индекс по дате для лент
create index idx_orders_created_at on orders (created_at desc);
```

**Java — программное управление Flyway (опционально, например, в init-утилите):**

```java
package com.example.migration;

import org.flywaydb.core.Flyway;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayInit {
    @Bean
    CommandLineRunner migrate(Flyway flyway) {
        return args -> {
            // По умолчанию Spring сам вызывает migrate() на старте.
            // Здесь показано, как это сделать вручную при необходимости.
            flyway.migrate();
        };
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.migration

import org.flywaydb.core.Flyway
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayInit {
    @Bean
    fun migrate(flyway: Flyway) = CommandLineRunner {
        flyway.migrate()
    }
}
```

**Liquibase (альтернатива) — YAML changelog c типами:**

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: you
      changes:
        - createTable:
            tableName: product
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: sku, type: VARCHAR(64), constraints: { nullable: false } }
              - column: { name: price, type: NUMERIC(19,4), constraints: { nullable: false } }
        - addUniqueConstraint:
            tableName: product
            columnNames: sku
            constraintName: uq_product_sku
```

---

## Именование объектов: стратегии `PhysicalNamingStrategy`/`ImplicitNamingStrategy` (только базово)

Hibernate применяет две стратегии именования. **ImplicitNamingStrategy** отвечает за «вывод» имён, когда вы их **не указали** (например, класс `CustomerOrder` без `@Table` превратится в таблицу `customer_order`). **PhysicalNamingStrategy** преобразует уже выведенные или указанные имена к «физическому» виду БД (snake_case, префиксы/суффиксы, экранирование). Spring Boot по умолчанию включает стратегию, которая превращает `camelCase` в `snake_case` и делает имена строчными — это то поведение, которое вы обычно видите без явной настройки.

Если вы подключаетесь к legacy-схеме, где используются иные конвенции (префиксы, особые схемы, нестандартные имена join-таблиц), вам почти наверняка потребуется кастомная PhysicalNamingStrategy. Она позволит централизованно преобразовывать имена таблиц/колонок/индексов и не размазывать `@Table(name=...)` по всем сущностям. Важно помнить: стратегия применяется **ко всем** именам, поэтому старайтесь писать её детерминированной и «без сюрпризов».

ImplicitNamingStrategy полезна, если вы хотите изменить правила вывода имён по умолчанию. Например, вместо `order_lines` для `OrderLine` вы хотите `order_line` (без множественного числа), или особые правила для join-таблиц. Но чем больше «магии» в implicit-стратегии, тем сложнее ориентироваться новичкам. В большинстве команд достаточно стандартной implicit-стратегии и явных `@Table/@Column` там, где это важно.

Главный практический совет — **явно именуйте стратегические объекты**. Таблицы агрегатов (`orders`, `customers`), ключевые индексы (`uq_customers_email_active`), внешние ключи — лучше назвать руками в миграциях и повторить в аннотациях как документацию. Стратегии — для «мелкой пыли»: привести camelCase к snake_case, добавить префикс `app_`, выбрать схему по умолчанию.

Обратите внимание на взаимодействие с кавычками. Если вы генерируете имена, совпадающие с ключевыми словами SQL (например, `order` в Postgres), без кавычек диалект может ругаться. В кастомной PhysicalNamingStrategy можно централизованно оборачивать такие имена в `Identifier#toIdentifier(name, true)` (второй аргумент — «quoted»). Но злоупотреблять квотингом не стоит: он делает SQL шумнее и чувствительным к регистру.

Ещё один тонкий момент — имена join-таблиц и FK-колонок. По умолчанию Hibernate выводит их из имён сущностей и полей, но в сложных графах получаются длинные названия. Если вам важны короткие и предсказуемые имена (для DBA и мониторинга), дайте их явно через `@JoinTable(name=...)`, `@JoinColumn(name=...)`, а в стратегиях лишь задайте общее правило для «прочих» мест.

Связь стратегий и миграций: **во что бы то ни стало** добейтесь того, чтобы ваши миграции и стратегия говорили на одном языке. Если PhysicalNamingStrategy добавляет префикс `app_`, а в миграции таблица называется просто `orders`, `ddl-auto: validate` свалится. Самый простой путь — фиксировать реальные имена в аннотациях (`@Table(name="orders")`) и писать миграции под них, а стратегию использовать минимально.

Базовую настройку стратегий в Boot можно сделать свойствами. Если вам достаточно дефолтной имплицитной стратегии, но нужна своя физическая, укажите только `spring.jpa.hibernate.naming.physical-strategy`. Для нестандартной имплицитной — `spring.jpa.hibernate.naming.implicit-strategy`. Имейте в виду: классы должны быть доступны как бины (не обязательно), но чаще это простые классы без DI.

Наконец, тестируйте на холодном старте. Любое изменение стратегии сразу затрагивает все сущности; если у вас `ddl-auto: validate`, приложение упадёт при несовпадении имён — и это хорошо. Так вы обнаружите рассинхроны раньше, чем новая версия попытается писать в «не те» таблицы. Держите отдельные интеграционные тесты, которые поднимают контекст и проверяют базовые CRUD-операции на реальной БД.

**application.yml — подключаем свою стратегию:**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  jpa:
    properties:
      hibernate.physical_naming_strategy: com.example.naming.PrefixPhysicalNamingStrategy
      # при необходимости:
      # hibernate.implicit_naming_strategy: org.hibernate.boot.model.naming.ImplicitNamingStrategyJpaCompliantImpl
```

**Java — пример простой PhysicalNamingStrategy с префиксом и snake_case:**

```java
package com.example.naming;

import org.hibernate.boot.model.naming.Identifier;
import org.hibernate.boot.model.naming.PhysicalNamingStrategy;
import org.hibernate.engine.jdbc.env.spi.JdbcEnvironment;

public class PrefixPhysicalNamingStrategy implements PhysicalNamingStrategy {
    private static final String PREFIX = "app_";

    private Identifier apply(Identifier name, JdbcEnvironment jdbc) {
        if (name == null) return null;
        String text = toSnakeCase(name.getText());
        String withPrefix = text.startsWith(PREFIX) ? text : PREFIX + text;
        return Identifier.toIdentifier(withPrefix, name.isQuoted());
    }

    private String toSnakeCase(String in) {
        return in.replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase();
    }

    @Override
    public Identifier toPhysicalCatalogName(Identifier name, JdbcEnvironment jdbc) { return name; }

    @Override
    public Identifier toPhysicalSchemaName(Identifier name, JdbcEnvironment jdbc) { return name; }

    @Override
    public Identifier toPhysicalTableName(Identifier name, JdbcEnvironment jdbc) { return apply(name, jdbc); }

    @Override
    public Identifier toPhysicalSequenceName(Identifier name, JdbcEnvironment jdbc) { return apply(name, jdbc); }

    @Override
    public Identifier toPhysicalColumnName(Identifier name, JdbcEnvironment jdbc) { return apply(name, jdbc); }
}
```

**Kotlin — аналогичная стратегия:**

```kotlin
package com.example.naming

import org.hibernate.boot.model.naming.Identifier
import org.hibernate.boot.model.naming.PhysicalNamingStrategy
import org.hibernate.engine.jdbc.env.spi.JdbcEnvironment

class PrefixPhysicalNamingStrategy : PhysicalNamingStrategy {
    private val prefix = "app_"

    private fun apply(name: Identifier?, jdbc: JdbcEnvironment): Identifier? {
        if (name == null) return null
        val snake = name.text.replace(Regex("([a-z])([A-Z])"), "$1_$2").lowercase()
        val withPrefix = if (snake.startsWith(prefix)) snake else prefix + snake
        return Identifier.toIdentifier(withPrefix, name.isQuoted)
    }

    override fun toPhysicalCatalogName(name: Identifier?, jdbc: JdbcEnvironment) = name
    override fun toPhysicalSchemaName(name: Identifier?, jdbc: JdbcEnvironment) = name
    override fun toPhysicalTableName(name: Identifier?, jdbc: JdbcEnvironment) = apply(name, jdbc)
    override fun toPhysicalSequenceName(name: Identifier?, jdbc: JdbcEnvironment) = apply(name, jdbc)
    override fun toPhysicalColumnName(name: Identifier?, jdbc: JdbcEnvironment) = apply(name, jdbc)
}
```

**Демонстрационная сущность (Java/Kotlin) и соответствующая таблица:**

```java
package com.example.naming;

import jakarta.persistence.*;

@Entity
// Имена не задаём — стратегия превратит CustomerOrder -> app_customer_order, поля -> app_created_at и т.п.
public class CustomerOrder {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String customerEmail;

    @Column(nullable = false)
    private java.time.Instant createdAt = java.time.Instant.now();
}
```

```kotlin
package com.example.naming

import jakarta.persistence.*
import java.time.Instant

@Entity
class CustomerOrder(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, length = 120)
    var customerEmail: String = "",
    @Column(nullable = false)
    var createdAt: Instant = Instant.now()
)
```

**Flyway-миграция, согласованная со стратегией (snake_case + префикс):**

```sql
-- V2__customer_order.sql
create table app_customer_order
(
    id               bigserial primary key,
    customer_email   varchar(120) not null,
    created_at       timestamptz  not null default now()
);
create index idx_app_customer_order_created_at on app_customer_order (created_at desc);
```

# 12. Тестирование persistence-слоя (база)

*Зависимости (общие варианты для всех подпунктов; выбирайте по ситуации)*

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

    // Вариант A: встраиваемая БД (H2)
    testImplementation 'com.h2database:h2:2.3.232'

    // Вариант B: Testcontainers для Postgres
    testImplementation 'org.testcontainers:junit-jupiter:1.20.3'
    testImplementation 'org.testcontainers:postgresql:1.20.3'
}
test {
    useJUnitPlatform()
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    java
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")

    // Вариант A: H2
    testImplementation("com.h2database:h2:2.3.232")

    // Вариант B: Testcontainers
    testImplementation("org.testcontainers:junit-jupiter:1.20.3")
    testImplementation("org.testcontainers:postgresql:1.20.3")
}
tasks.test { useJUnitPlatform() }
```

---

## `@DataJpaTest` и `TestEntityManager`: быстрые slice-тесты репозиториев

`@DataJpaTest` — это «срез» контекста Spring, который поднимает только слой JPA: `EntityManagerFactory`, `DataSource`, репозитории Spring Data, переводчик исключений и пару полезных бинов. Такой подход ускоряет тесты, уменьшает шум и делает их более надёжными: вы изолируетесь от веб-слоя, безопасности и прочего. По умолчанию каждый тест оборачивается транзакцией и откатывается после выполнения, оставляя БД чистой.

Вместе с `@DataJpaTest` Spring Boot предоставляет `TestEntityManager` — тонкую обёртку над `EntityManager`, упрощающую типичные операции: `persist`/`persistAndFlush`, `find`, `flush`, `clear`. Она особенно удобна для подготовки данных и точечной синхронизации с БД в моменты, когда важно поймать ошибки ограничений (например, уникальности) раньше, чем репозиторий сделает внутренний `flush`.

С точки зрения практики полезно мыслить «через контракты» ваших репозиториев. Выделите на них тесты: happy-path (создание/поиск/обновление/удаление), негативные сценарии (нарушение уникальности, валидации на уровне БД), границы пагинации/сортировок. Такие тесты создают «охранный контур» вокруг наиболее чувствительной части приложения — постоянного состояния.

Скорость — важный аргумент. Slice-тесты стартуют в считанные сотни миллисекунд и выполняются быстро, что мотивирует команду реально их запускать локально. В отличие от полноценных интеграционных сценариев, здесь нет лишних бинов, а база часто in-memory. Это позволяет спокойно держать десятки/сотни тестов, которые срабатывают на каждый PR.

При этом `@DataJpaTest` не отменяет интеграции: вам всё равно нужны несколько тестов «сквозь слой» (например, `@SpringBootTest` с Testcontainers) для проверки «связки» профилей, миграций и диалектных нюансов. Но именно slice закрывают 80–90% будничной логики репозиториев и экономят время.

Тонкость — конфигурация профилей. По умолчанию `@DataJpaTest` подменяет `DataSource` на встраиваемую БД, если в classpath есть H2/Derby/… и если вы не запретили замену `@AutoConfigureTestDatabase(replace = NONE)`. Это удобно для быстрого старта, но если вы тестируете специфичные типы (UUID, `jsonb`), лучше явно отключить подмену и использовать Testcontainers.

Ещё один момент — управляемость `flush`. Порог обмена между контекстом персистентности и БД важен для тестов, где вы хотите увидеть «настоящие» SQL-ошибки. Методы `persistAndFlush` или явный `entityManager.flush()` помогут «поднять» исключение в момент сохранения, а не отложить до конца теста.

Наконец, держите фабрики тестовых данных рядом: билдэры или «mother»-классы, создающие валидные сущности. Они снижают дублирование и делают чтение тестов приятнее. В сочетании с `TestEntityManager` такая фабрика превращает подготовку состояния в пару строк.

**Java — `@DataJpaTest` + `TestEntityManager`**

```java
package com.example.datajpatest;

import jakarta.persistence.*;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.test.context.ActiveProfiles;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;

@Entity
@Table(name = "customers", uniqueConstraints = @UniqueConstraint(columnNames = "email"))
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false, unique = true, length = 120) String email;
    @Column(nullable = false, length = 120) String name;
    protected Customer() {}
    Customer(String email, String name){ this.email = email; this.name = name; }
}

import org.springframework.data.jpa.repository.JpaRepository;
interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByEmailIgnoreCase(String email);
}

@DataJpaTest
@ActiveProfiles("test") // application-test.yml при необходимости
class CustomerRepositoryTest {

    @Autowired private CustomerRepository repo;
    @Autowired private TestEntityManager tem;

    @Test
    @DisplayName("findByEmailIgnoreCase возвращает сохранённого клиента")
    void findByEmail() {
        Customer c = new Customer("a@example.com", "Alice");
        tem.persistAndFlush(c); // гарантируем insert, видим SQL-исключения сразу

        Optional<Customer> found = repo.findByEmailIgnoreCase("A@EXAMPLE.COM");
        assertThat(found).isPresent();
        assertThat(found.get().name).isEqualTo("Alice");
    }

    @Test
    @DisplayName("Нарушение уникальности email приводит к DataIntegrityViolationException")
    void uniqueEmail() {
        tem.persistAndFlush(new Customer("dup@example.com", "A"));
        assertThatThrownBy(() -> {
            repo.saveAndFlush(new Customer("dup@example.com", "B"));
        }).isInstanceOf(DataIntegrityViolationException.class);
    }
}
```

**Kotlin — `@DataJpaTest` + `TestEntityManager`**

```kotlin
package com.example.datajpatest

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.test.context.ActiveProfiles
import java.util.*

@Entity
@Table(name = "customers", uniqueConstraints = [UniqueConstraint(columnNames = ["email"])])
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 120)
    var email: String = "",
    @Column(nullable = false, length = 120)
    var name: String = ""
)

interface CustomerRepository : JpaRepository<Customer, Long> {
    fun findByEmailIgnoreCase(email: String): Optional<Customer>
}

@DataJpaTest
@ActiveProfiles("test")
class CustomerRepositoryTest(
    @Autowired val repo: CustomerRepository,
    @Autowired val testEntityManager: org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager
) {

    @Test
    @DisplayName("findByEmailIgnoreCase возвращает сохранённого клиента")
    fun findByEmail() {
        testEntityManager.persistAndFlush(Customer(email = "a@example.com", name = "Alice"))
        val found = repo.findByEmailIgnoreCase("A@EXAMPLE.COM")
        assertThat(found).isPresent
        assertThat(found.get().name).isEqualTo("Alice")
    }

    @Test
    @DisplayName("Нарушение уникальности email -> DataIntegrityViolationException")
    fun uniqueEmail() {
        testEntityManager.persistAndFlush(Customer(email = "dup@example.com", name = "A"))
        assertThatThrownBy {
            repo.saveAndFlush(Customer(email = "dup@example.com", name = "B"))
        }.isInstanceOf(DataIntegrityViolationException::class.java)
    }
}
```

---

## Встроенные БД vs Testcontainers: совместимость типов и поведения

Выбор инфраструктуры для тестов значительно влияет на надёжность. Встроенные БД (H2, HSQL, Derby) дают молниеносный старт и простоту — они живут в памяти, легко конфигурируются и не требуют Docker. Однако их диалект SQL и поведение типов **не идентичны** вашему продакшн-Postgres. Если вы используете `jsonb`, `uuid`, оконные функции, `generated as identity`, частичные индексы или специфичные коллации — H2 начнёт «сглаживать» отличия, и часть багов вы увидите только на «живой» базе.

Testcontainers обеспечивает «точно такой же» Postgres (или любую другую СУБД) в Docker-контейнере. Тесты стартуют медленнее (секунды), но вы получаете полную совместимость типов, функций и планов выполнения. Особенно это важно, когда вы опираетесь на миграции Flyway/Liquibase: ваши тесты реально прогоняют те же скрипты против того же диалекта, что и в проде.

Компромиссный вариант — использовать H2 в «PostgreSQL-режиме» (compatibility mode) и писать осторожный SQL. Это снимает часть несовместимостей, но не все: например, нет `jsonb`, другое поведение индексов, отличия в `timestamp with time zone`. Это лучше, чем «чистая» H2, но хуже, чем Testcontainers. Для новичковых пет-проектов — норм, для продакшн-команд — лучше контейнеры.

Надо учитывать и время ранка. Для быстрого фидбэка используйте два уровня: slice-тесты на H2 для репозиториев без диалектных трюков (они бегут быстро), и слой «сквозных» интеграций (`@SpringBootTest`) на Testcontainers, который запускается в CI и на локали по кнопке. Такая пирамида тестов даёт баланс скорости и уверенности.

Spring Boot 3.1+ упростил работу с контейнерами: аннотация `@ServiceConnection` автоматически интегрирует Testcontainers в контекст, а для JUnit можно использовать `@Testcontainers` и `@Container`. Вариант «на бине» хорош тем, что контейнер поднимается один раз на весь класс тестов, а подключения раздаются через свойства.

Также важно помнить о транзакциях. Testcontainers — это «настоящая» БД; ваши тесты должны по-прежнему быть изолированными. `@DataJpaTest` по умолчанию откатывает изменения, и это прекрасно работает и с контейнерами. Но если вы используете `@SpringBootTest`, подумайте о чистке данных между тестами: `@DirtiesContext` (дорого), `@Sql` (быстро), или truncate в `@BeforeEach`.

Не забывайте про время зон и локали. На H2 можно «не заметить» проблемы с `timestamptz`, а на Postgres при Testcontainers всплывёт расхождение в сдвигах. Ставьте `hibernate.jdbc.time_zone=UTC` и проверяйте сериализацию/десериализацию дат на ваших DTO.

Наконец, внимательно следите за версиями контейнера: используйте ту же основную ветку, что и в проде (например, `postgres:16-alpine`). Это минимизирует сюрпризы при апгрейдах. Testcontainers вытаскивает образы из Docker Registry — убедитесь, что у разработчиков настроен доступ к сети и докеру; в CI добавьте кеш реестра.

**Java — H2 (быстро) и PostgreSQL Testcontainers (совместимо)**

*H2 профиль (`src/test/resources/application-test.yml`):*

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE;DEFAULT_NULL_ORDERING=HIGH
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        format_sql: true
        jdbc.time_zone: UTC
```

*Testcontainers класс:*

```java
package com.example.tc;

import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.*;

import javax.sql.DataSource;
import java.sql.Connection;

@Testcontainers
@SpringBootTest
@TestPropertySource(properties = {
        "spring.jpa.hibernate.ddl-auto=update" // ТОЛЬКО для примера; в реале — миграции
})
class PostgresContainerTest {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired DataSource ds;

    @Test
    void ping() throws Exception {
        try (Connection c = ds.getConnection()) {
            Assertions.assertTrue(c.isValid(2));
        }
    }
}
```

**Kotlin — аналог Testcontainers**

```kotlin
package com.example.tc

import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import javax.sql.DataSource
import org.springframework.beans.factory.annotation.Autowired

@Testcontainers
@SpringBootTest
class PostgresContainerTest(
    @Autowired val dataSource: DataSource
) {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:16-alpine")
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app")

        @JvmStatic
        @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("spring.datasource.url") { pg.jdbcUrl }
            r.add("spring.datasource.username") { pg.username }
            r.add("spring.datasource.password") { pg.password }
        }
    }

    @Test
    fun ping() {
        dataSource.connection.use { conn ->
            assertTrue(conn.isValid(2))
        }
    }
}
```

---

## Фикстуры данных: SQL-скрипты/@Sql и «чистые» фабрики сущностей; изоляция тестов по транзакциям

Фикстуры — это подготовленные данные, необходимые для теста. Их можно загружать SQL-скриптами (`@Sql`), миграциями, либо создавать программно через фабрики («builders»). Идея проста: тест должен быть самодостаточным и понятным — по его коду видно, «что на входе» и «что ожидаем на выходе». Лучшие фикстуры — те, которые **локальны** тесту и минимальны по объёму.

`@Sql` удобен для «табличных» сценариев: вы можете описать десяток строк в нескольких таблицах и использовать их в ряде тестов. Скрипты храните рядом (`src/test/resources/sql/...`), используйте явные поля и комментарии. Важно: не перегружайте скрипт данными, которые не относятся к тесту — это усложняет чтение и поддержание.

Программные фабрики работают лучше там, где сущности богато связаны, а данные динамичны. Они позволяют описывать доменные инварианты на языке кода: `OrderMother.withLines(n)` вернёт валидный заказ, а `UserBuilder().withEmail(...).build()` — валидного пользователя. Вы избегаете сырого SQL и хрупкой зависимости от схемы, тест становится декларативным и «говорящим».

Изоляция — ключ к надёжности. По умолчанию `@DataJpaTest` делает каждый тест транзакционным и откатывает изменения после завершения метода. Это прекрасно: фикстуры не «протекают» между тестами. Если вы используете `@Sql`, по умолчанию скрипт выполняется **до** теста в той же транзакции, и данные также будут откатаны. Чтобы сохранить данные между тестами — используйте `@SqlConfig(transactionMode = ISOLATED)` (редко нужно).

Иногда нужно выполнить проверку в другой транзакции, чем фикстура. Например, протестировать оптимистическую блокировку или эффект `flush`. Для этого можно пометить метод `@Transactional(propagation = NOT_SUPPORTED)`, вручную управляя транзакциями через сервисы, или запускать вторую транзакцию через отдельный бин. Это усложняет код, но делает поведение ближе к реальному.

Порядок очистки важен. Если вы используете `@SpringBootTest` и общий контекст, чистите данные в `@BeforeEach`/`@AfterEach`: либо `truncate` нужных таблиц (быстро, но требует прав), либо удаление через репозитории (медленнее), либо `@Sql` с `DELETE`. Для Postgres удобно иметь утилитарный скрипт `truncate_all.sql`, который очищает таблицы, учитывая FK (через `cascade`).

Локали и часовые пояса тоже относятся к фикстурам. При вставке `timestamptz` через SQL и через JPA вы можете получить расхождение в ожидаемых значениях. Держите `hibernate.jdbc.time_zone=UTC` и вставляйте временные значения в ISO-8601 с Z-суффиксом, чтобы избежать двусмысленности.

Наконец, документируйте принципы в `CONTRIBUTING.md`: где лежат SQL-фикстуры, какие есть фабрики, как добавлять новый набор данных. Отсутствие правил быстро превращает тестовую базу в «свалку», где сложно понять, кто и зачем создал те или иные строки.

**Java — пример @Sql и фабрики сущностей, плюс изоляция транзакций**

*SQL-файл `src/test/resources/sql/users.sql`:*

```sql
insert into users(id, email, name) values
(1001, 'u1@example.com', 'U1'),
(1002, 'u2@example.com', 'U2');
```

*Сущности и репозиторий (минимум):*

```java
package com.example.fixtures;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

@Entity @Table(name = "users")
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false, unique = true) String email;
    @Column(nullable = false) String name;
    protected User() {}
    User(String email, String name){ this.email = email; this.name = name; }
}
interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

*Фабрика (Mother/Builder):*

```java
package com.example.fixtures;

class UserMother {
    static User random() { return new User("user"+System.nanoTime()+"@example.com", "User"); }
    static User withEmail(String email) { return new User(email, "User"); }
}
```

*Tests:*

```java
package com.example.fixtures;

import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.*;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
class FixtureTests {

    @Autowired UserRepository repo;
    @Autowired TestEntityManager tem;

    @Test
    @Sql("/sql/users.sql") // загрузим фикс-данные
    void findsUserFromSqlFixture() {
        assertThat(repo.findByEmail("u1@example.com")).isPresent();
        assertThat(repo.findByEmail("u2@example.com")).isPresent();
    }

    @Test
    void createsUserViaFactory() {
        User u = UserMother.withEmail("factory@example.com");
        tem.persistAndFlush(u);
        assertThat(repo.findByEmail("factory@example.com")).isPresent();
    }

    @Test
    @Transactional(propagation = Propagation.NOT_SUPPORTED) // проверим видимость между транзакциями
    void visibilityAcrossTransactions() {
        // транзакции нет; сохраним в отдельной
        Long id = saveInNewTx("x@example.com");
        // читаем заново — должно быть видно
        assertThat(repo.findByEmail("x@example.com")).isPresent();
    }

    @org.springframework.beans.factory.annotation.Autowired
    private org.springframework.transaction.PlatformTransactionManager tm;

    private Long saveInNewTx(String email) {
        org.springframework.transaction.support.TransactionTemplate tpl =
                new org.springframework.transaction.support.TransactionTemplate(tm);
        return tpl.execute(status -> repo.save(new User(email, "X")).id);
    }
}
```

**Kotlin — те же идеи с @Sql и фабриками**

```kotlin
package com.example.fixtures

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.test.context.jdbc.Sql
import org.springframework.transaction.PlatformTransactionManager
import org.springframework.transaction.support.TransactionTemplate

@Entity
@Table(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true) var email: String = "",
    @Column(nullable = false) var name: String = ""
)

interface UserRepository : org.springframework.data.jpa.repository.JpaRepository<User, Long> {
    fun findByEmail(email: String): java.util.Optional<User>
}

object UserMother {
    fun random(): User = User(email = "user${System.nanoTime()}@example.com", name = "User")
    fun withEmail(email: String): User = User(email = email, name = "User")
}

@DataJpaTest
class FixtureTests(
    @Autowired val repo: UserRepository,
    @Autowired val tem: org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager,
    @Autowired val txManager: PlatformTransactionManager
) {

    @Test
    @Sql("/sql/users.sql")
    fun findsUserFromSqlFixture() {
        assertThat(repo.findByEmail("u1@example.com")).isPresent
        assertThat(repo.findByEmail("u2@example.com")).isPresent
    }

    @Test
    fun createsUserViaFactory() {
        tem.persistAndFlush(UserMother.withEmail("factory@example.com"))
        assertThat(repo.findByEmail("factory@example.com")).isPresent
    }

    @Test
    fun visibilityAcrossTransactions() {
        val tpl = TransactionTemplate(txManager)
        val id = tpl.execute { repo.save(UserMother.withEmail("x@example.com")).id }!!
        assertThat(repo.findByEmail("x@example.com")).isPresent
    }
}
```

*Опционально: «чистка» между тестами через SQL (если используете @SpringBootTest)*

```sql
-- src/test/resources/sql/cleanup.sql
truncate table orders cascade;
truncate table users cascade;
```

```java
@Sql(scripts = "/sql/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
```

# 13. Частые ошибки новичков (и как их избежать)

*Зависимости для примеров этой подтемы (универсально):*

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    java
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## Неверная сторона владения в связях; EAGER по умолчанию; отсутствие вспомогательных методов в двусторонних связях

Первый типичный промах — неправильно определённая **сторона владения** (owning side) в двусторонних связях. В JPA обновление внешнего ключа выполняется **только** со стороны владельца. Если вы «толкаете» изменения в сторону `mappedBy`, Hibernate не сделает `UPDATE` нужной колонки, и связь окажется несинхронизированной. Это приводит к неожиданным «потерянным» связям и загадочным SQL.

Чтобы понимать, кто «владеет», ориентируйтесь на внешний ключ в БД. Там, где хранится FK-колонка (`customer_id` в таблице `orders`), — там и должна быть сторона-владелец (`@ManyToOne` у `Order`). На обратной стороне (`Customer.orders`) ставится `mappedBy="customer"`, что прямо указывает: «я не владелец, изменения не пишу».

Второй промах — оставить дефолтный `EAGER` на `@ManyToOne`/`@OneToOne`. По спецификации они жадные, и Hibernate подтянет связанные сущности «по умолчанию», что часто ведёт к **N+1** и жирным графам в памяти. В большинстве доменных моделей безопаснее **явно** ставить `fetch = FetchType.LAZY` и контролировать выборку через запросы (`JOIN FETCH`, EntityGraph).

Третий промах — отсутствие **вспомогательных методов** для двусторонних связей. Когда вы добавляете заказ в коллекцию клиента, нужно одновременно проставить обратную сторону `order.setCustomer(this)`. Если этого не сделать, в памяти будет одно, а в БД — другое, и при `flush` зависимость может просто не сохраниться. Маленький метод `addOrder` решает проблему и делает код выразительным.

Ещё одна распространённая ловушка — `cascade` и `orphanRemoval` «наудачу». Включённый повсюду `CascadeType.ALL` может неожиданно удалить «детей» при удалении «родителя», а `orphanRemoval=true` подчистит записи при отцеплении из коллекции. Эти флаги работают мощно и быстро; включайте их только там, где модель действительно подразумевает **жизненный цикл** дочерних объектов, управляемый родителем.

Следите за равенством и хэш-кодом у сущностей, которые лежат в Set-коллекциях. Если вы определите `equals/hashCode` по изменяемым полям, коллекция легко «потеряет» элемент при изменении значения. Для сущностей с БД чаще всего достаточно равенства по идентификатору (с аккуратной обработкой `null`), а для embeddable-типов по всем полям-значениям.

Проблемы владения часто маскируются тестовыми данными. Когда у вас один заказ и один клиент, всё «как будто работает». Но при множестве записей начинают всплывать «висячие» строки и дубли. Привычка — проверять SQL-лог, писать интеграционные тесты и просматривать схему миграций — спасает от долгих расследований.

Старайтесь, чтобы модель была **асимметричной** там, где это возможно. Многие связи в реальных системах — это «много-к-одному» и «один-ко-многим», и достаточно экспонировать в API только сторону, с которой вы реально работаете. Чем меньше у вас двусторонних связей, тем меньше риска рассинхронизаций.

В REST-слое помните, что сериализация двусторонних связей без осторожности приведёт к рекурсивным JSON-циклами. Для DTO используйте плоские проекции. Если по какой-то причине отдаёте сущности, применяйте `@JsonIgnore` на обратной стороне или `@JsonManagedReference/@JsonBackReference`.

И наконец, не забывайте про ленивые прокси. Даже если вы аккуратно поставили `LAZY` на `@ManyToOne`, доступ к полю вне транзакции приведёт к `LazyInitializationException`. Решение — не тянуть сущности в контроллеры, а конвертировать их в DTO внутри сервисной транзакции.

**Java — правильная сторона владения и вспомогательные методы (LAZY)**

```java
package com.example.ownership;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable=false, length=120)
    private String name;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.PERSIST, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    protected Customer() {}
    public Customer(String name) { this.name = name; }

    public void addOrder(Order o) {
        orders.add(o);
        o.setCustomer(this); // поддерживаем обе стороны
    }
    public void removeOrder(Order o) {
        orders.remove(o);
        o.setCustomer(null);
    }
}

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @Column(nullable=false)
    private long totalCents;

    protected Order() {}
    public Order(long totalCents) { this.totalCents = totalCents; }
    public void setCustomer(Customer c) { this.customer = c; }
}
```

**Kotlin — аналогичный маппинг с helper-методами**

```kotlin
package com.example.ownership

import jakarta.persistence.*

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, length = 120)
    var name: String = ""
) {
    @OneToMany(mappedBy = "customer", cascade = [CascadeType.PERSIST], orphanRemoval = true, fetch = FetchType.LAZY)
    var orders: MutableList<Order> = mutableListOf()

    fun addOrder(o: Order) {
        orders += o
        o.customer = this
    }

    fun removeOrder(o: Order) {
        orders -= o
        o.customer = null
    }
}

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    var customer: Customer? = null,

    @Column(nullable = false)
    var totalCents: Long = 0
)
```

---

## Использование `merge` «на всё» вместо `persist` на новые сущности

Ещё одна типовая ошибка — вызывать `EntityManager.merge` для **новых** сущностей «на всякий случай». Важно помнить семантику: `merge` **никогда не делает переданный объект managed**. Он создаёт копию внутри контекста и возвращает **другой** экземпляр. Переданный объект так и останется detached. Это легко приводит к путанице ссылок и «теряющимся» изменениям после метода.

Кроме того, `merge` вынужден проверять, есть ли запись в БД: если у объекта установлен `id`, провайдер делает `SELECT` перед `UPDATE`, а если `id` `null`, он всё равно строит граф копий. Для новых сущностей `persist` проще и дешевле: вы объявляете контексту «вот новый объект», и Hibernate вставит его при `flush`. На графах это критично: `merge` копирует всю коллекцию «как есть», что может привести к лишним удалениями/вставкам.

Часто `merge` используют из-за недопонимания жизненного цикла. Если вы загрузили сущность, вышли из транзакции и хотите в новой транзакции сохранить изменения, кажется логичным «смерджить». Но правильнее — **повторно загрузить** сущность и применить изменения к управляемому экземпляру, либо работать в одной транзакции. `merge` — тяжёлая артиллерия для синхронизации detached-графов, а не дефолтная операция «сохранить всё подряд».

В Spring Data JPA `save` на `JpaRepository` ведёт себя «как merge» для сущностей с не-нулевым id и «как persist» для новых. Это удобно, но не означает, что нужно везде руками вызывать `em.merge`. Наоборот, держите транзакции короткими и работайте с managed-экземплярами. Для «создать» используйте `save(new)`, для «обновить» — загрузить + изменить поля + `flush` по завершении.

У `merge` есть ещё один побочный эффект: он может «перезатереть» поля `null` из DTO, если вы бездумно копируете DTO→Entity и вызываете `merge`. Для частичных обновлений (patch) используйте явную логику с `Optional`/`Nullable` и изменяйте только пришедшие поля. «Склейка» графов через `merge` в таких сценариях делает много лишнего.

Если вы всё же используете `merge` для сложных графов, сначала **очистите** коллекции осторожно. Неправильная «пересборка» детской коллекции часто приводит к каскадным `DELETE` и потере данных. Надёжнее иметь helper-методы на агрегате (как в предыдущем пункте), которые поддерживают консистентность, и заменять элементы по ключу, а не «всё заново».

Ещё важный момент — `IDENTITY`-генерация ключей. При вставке Hibernate получает id только после `INSERT`. Если вы «меняете» id руками на detached-экземпляре и зовёте `merge`, провайдер может не угадать ваши намерения и выполнит дорогую серию SQL. Не подменяйте идентификаторы и не смешивайте политику генерации.

Профилируйте SQL. Один и тот же бизнес-кейс «создать заказ с 10 линиями» через `persist` часто даёт 11 `INSERT`. Через `merge` можно легко получить 20+ запросов (`select exists?`, копирование графа, каскады). В высоконагруженных системах умный выбор операции экономит целые ядра CPU.

И помните про читаемость. `persist` говорит «создать», `remove` — «удалить», «изменить» — это просто присвоение полей managed-объекта. Код с `merge` не отражает намерений и требует от читателя помнить детали JPA; такой код труднее ревьюить и сопровождать.

**Java — чем плох `merge` для новых и как делать правильно**

```java
package com.example.merge;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "tags")
class Tag {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, unique = true) String name;
    protected Tag(){}
    Tag(String name){ this.name = name; }
}

@Service
class TagService {
    @PersistenceContext private EntityManager em;

    // ПЛОХО: merge для нового объекта возвращает другой экземпляр и делает лишний SQL
    @Transactional
    public Long createWithMerge(String name) {
        Tag detached = new Tag(name);
        Tag managed = em.merge(detached);   // detached != managed
        // здесь легко случайно продолжить работать с detached
        return managed.id;
    }

    // ХОРОШО: persist для нового — дёшево и намерение прозрачно
    @Transactional
    public Long createWithPersist(String name) {
        Tag tag = new Tag(name);
        em.persist(tag); // tag становится managed
        return tag.id;   // появится после flush/insert
    }
}
```

**Kotlin — аналогичная демонстрация**

```kotlin
package com.example.merge

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "tags")
class Tag(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true)
    var name: String = ""
)

@Service
class TagService(@PersistenceContext private val em: EntityManager) {

    @Transactional
    fun createWithMerge(name: String): Long? {
        val detached = Tag(name = name)
        val managed = em.merge(detached) // detached !== managed
        return managed.id
    }

    @Transactional
    fun createWithPersist(name: String): Long? {
        val tag = Tag(name = name)
        em.persist(tag)
        return tag.id
    }
}
```

---

## Логика в сущностях, завязанная на ленивые коллекции, за пределами транзакций

Третья ошибка — помещать бизнес-логику **внутрь сущностей**, где она опирается на ленивые коллекции (`@OneToMany`/`@ManyToMany`) и вызывается за пределами транзакции. Пока сессия открыта, всё кажется нормальным. Но как только вы попытаетесь дернуть `entity.getOrders().size()` в контроллере при `open-in-view=false`, получите `LazyInitializationException`. Это классический сценарий для новичков.

Сущность — это состояние и инварианты. Логику, которая требует загрузки данных, лучше держать в **сервисном** слое, где вы контролируете транзакцию и выборку (`JOIN FETCH`, проекции). Нарушение этого принципа делает поведение зависящим от инфраструктурных настроек (есть ли Open Session In View), что плохо переносится между проектами и окружениями.

Если доменная логика действительно должна жить в модели, старайтесь, чтобы она работала только с уже загруженными полями или получала явно необходимые данные параметрами. Не заставляйте метод сущности «самому» лезть в ленивые коллекции — это связывает слой предметной области с ORM.

Хорошая практика — **возвращать DTO** из сервисов наружу и не тянуть сущности в веб-слой. При конвертации в DTO внутри транзакции вы сами решаете, какие ассоциации подгрузить: через `JOIN FETCH`, `EntityGraph` или отдельные запросы. Это убирает ленивые прокси из представления и исключает ошибки инициализации.

Альтернатива — использовать `@EntityGraph` или явные `join fetch` в репозитории там, где вы гарантированно показываете связанные данные. Это делает контракт метода очевидным: «возвращаю `Customer` с заказами». Но всё равно лучше преобразовать в DTO и не отдавать managed-объекты наружу.

Ещё один путь — «компоновка» запросов: сначала получить список id подходящих клиентов, затем отдельным запросом выбрать заказы по этим id. Это стабильно работает с пагинацией и не раздувает графы. Главное — не смешивать подходы внутри одного метода.

Не поддавайтесь искушению включить `spring.jpa.open-in-view=true` «чтобы всё работало». Это анти-паттерн для публичных API: вы расширяете транзакционную границу до веб-слоя, рискуете длинными транзакциями и скрытыми N+1. Лучше исправить слой доступа и внедрить DTO.

Пишите тесты, которые воспроизводят жизненную ситуацию: `open-in-view=false`, контроллер вызывает сервис, сервис — репозиторий. Если код тянет ленивую коллекцию в контроллере — тест упадёт, и вы увидите проблему до релиза. Такая дисциплина экономит часы дебаггинга.

Наконец, не забывайте, что ленивость — свойство не только коллекций, но и `@ManyToOne`. Доступ к `order.getCustomer().getName()` вне транзакции тоже «рванёт». Если нужно имя клиента, вытащите его в DTO заранее.

**Java — неправильный доступ к LAZY и корректная альтернатива с DTO**

```java
package com.example.lazy;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable=false) String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    List<Order> orders;

    protected Customer() {}
    Customer(String name){ this.name = name; }

    // ПЛОХО: обращение к ленивой коллекции может взорваться вне транзакции
    public int ordersCount() { return orders.size(); }
}

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name="customer_id", nullable=false)
    Customer customer;
    @Column(nullable=false) long totalCents;
    protected Order() {}
}

record CustomerDto(Long id, String name, int ordersCount) { }

@Repository
interface CustomerRepository extends JpaRepository<Customer, Long> {

    @Query("""
        select c from Customer c
        left join fetch c.orders
        where c.id = :id
    """)
    Customer findWithOrders(@org.springframework.data.repository.query.Param("id") Long id);
}

@Service
class CustomerService {
    private final CustomerRepository repo;
    CustomerService(CustomerRepository repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public CustomerDto getDto(Long id) {
        Customer c = repo.findWithOrders(id); // загрузили заранее
        int count = c.orders.size();
        return new CustomerDto(c.id, c.name, count);
    }
}
```

**Kotlin — корректный подход с DTO и fetch join**

```kotlin
package com.example.lazy

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = ""
) {
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    var orders: MutableList<Order> = mutableListOf()

    fun ordersCount(): Int = orders.size // плохо вызывать вне транзакции
}

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    var customer: Customer? = null,
    @Column(nullable = false) var totalCents: Long = 0
)

data class CustomerDto(val id: Long, val name: String, val ordersCount: Int)

@Repository
interface CustomerRepository : JpaRepository<Customer, Long> {
    @Query(
        """
        select c from Customer c
        left join fetch c.orders
        where c.id = :id
        """
    )
    fun findWithOrders(@org.springframework.data.repository.query.Param("id") id: Long): Customer
}

@Service
class CustomerService(private val repo: CustomerRepository) {

    @Transactional(readOnly = true)
    fun getDto(id: Long): CustomerDto {
        val c = repo.findWithOrders(id)
        return CustomerDto(c.id!!, c.name, c.orders.size)
    }
}
```

---

## Ранний выбор нативного SQL без необходимости — ломает переносимость и кеш 1-го уровня

Четвёртая ошибка — «сразу писать нативный SQL», даже когда хватает JPQL. Нативные запросы мощны: доступ к CTE, оконным функциям, `jsonb`-операторам, подсказкам планировщика. Но платить приходится переносимостью и сложностью маппинга. Вы теряете преимущества ORM: кросс-СУБД, проверку JPQL при старте, удобное управление графами сущностей.

Кроме того, нативные апдейты могут **обойти** контекст персистентности. Если вы сделали `UPDATE articles SET title = ...` нативкой, а у вас в текущем Persistence Context уже есть `Article` с тем же id, этот managed-экземпляр останется со старым `title` до `refresh`/`clear`. В итоге в сервисе вы видите «старое», а в БД — «новое», что рождает трудноуловимые баги.

JPQL покрывает 80–90% CRUD и поисковых задач. Это язык над сущностями, провайдер валидирует его при старте, а вы получаете типизированные результаты и аккуратную интеграцию со `Pageable`. Если нужен массовый апдейт — есть **JPQL bulk update** с `@Modifying`, который корректно учитывает диалект и проще тестируется.

Когда без нативного никак (специфичный индекс, `jsonb_path_query`, `ON CONFLICT`), старайтесь отделять эти места в инфраструктурных DAO и документировать контракты. После нативного апдейта — **или** `clear()`, **или** `refresh()` для затронутых сущностей, **или** выполняйте такие операции в отдельной транзакции до чтений.

Ещё одна проблема нативного SQL — маппинг результатов. Возвращать `Object[]` неудобно и хрупко, интерфейсные проекции работают только при аккуратных алиасах. В итоге вы получите больше бойлерплейта, чем при DTO-конструкторах в JPQL. Для сложных отчётов это оправдано, но для повседневных списков — избыточно.

Не забывайте про пагинацию. Многие пишут `select ... from (...) as t limit :size` с нативом и удивляются, что `count` не сходится. В Spring Data с `@Query(nativeQuery=true)` нужно явно указывать `countQuery`. Для JPQL фреймворк умеет строить `count` автоматически в простых случаях и даёт вам `Page<T>` без лишней боли.

Поддержка кода — отдельный аспект. Через полгода команда забудет, почему в запросе `jsonb_build_object` и `->>` именно так. JPQL читабельнее для большинства разработчиков Java, а специфичные места всегда привлекают внимание в ревью. Держите «острые» SQL локализованными, покрытыми тестами и сопровождением DBA.

Производительность — не всегда аргумент за нативный SQL. Современный Hibernate генерирует очень приличный SQL, а узкие места почти всегда лечатся правильными индексами и **выборкой ровно того, что нужно** (DTO/проекции, `JOIN FETCH` там, где уместно). Переключаться на native стоит, когда вы **точно** понимаете, что нужно базе.

И помните про кеш 1-го уровня. Если смешиваете native update и чтение тех же сущностей в одном методе, обязательно делайте `em.flush()` перед апдейтом и `em.clear()` после — чтобы не работать со «старыми» данными. Ещё лучше — разделите операции на два метода/транзакции.

**Java — нативный апдейт, «устаревший» кеш и корректная альтернатива на JPQL**

```java
package com.example.nativepit;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "articles")
class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable=false) String title;
    protected Article() {}
    Article(String title){ this.title = title; }
}

@Repository
interface ArticleRepository extends JpaRepository<Article, Long> {

    @Modifying
    @Query("update Article a set a.title = :title where a.id = :id")
    int renameJpql(@org.springframework.data.repository.query.Param("id") Long id,
                   @org.springframework.data.repository.query.Param("title") String title);
}

@Service
class ArticleService {
    @PersistenceContext private EntityManager em;
    private final ArticleRepository repo;
    ArticleService(ArticleRepository repo){ this.repo = repo; }

    @Transactional
    public void renameNative(Long id, String title) {
        Article a = em.find(Article.class, id); // теперь a в кэше 1-го уровня
        em.createNativeQuery("update articles set title = :t where id = :id")
          .setParameter("t", title)
          .setParameter("id", id)
          .executeUpdate();
        // a.title всё ещё старое! Исправляем:
        em.refresh(a); // или em.clear()
    }

    @Transactional
    public void renameJpql(Long id, String title) {
        repo.renameJpql(id, title); // корректный bulk update через JPQL
    }
}
```

**Kotlin — то же самое**

```kotlin
package com.example.nativepit

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "articles")
class Article(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = ""
)

@Repository
interface ArticleRepository : JpaRepository<Article, Long> {

    @Modifying
    @Query("update Article a set a.title = :title where a.id = :id")
    fun renameJpql(id: Long, title: String): Int
}

@Service
class ArticleService(
    @PersistenceContext private val em: EntityManager,
    private val repo: ArticleRepository
) {

    @Transactional
    fun renameNative(id: Long, title: String) {
        val a = em.find(Article::class.java, id)
        em.createNativeQuery("update articles set title = :t where id = :id")
            .setParameter("t", title)
            .setParameter("id", id)
            .executeUpdate()
        em.refresh(a) // или em.clear()
    }

    @Transactional
    fun renameJpql(id: Long, title: String) {
        repo.renameJpql(id, title)
    }
}
```
















