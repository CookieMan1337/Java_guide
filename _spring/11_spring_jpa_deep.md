---
layout: page
title: "Spring Data JPA: Углубленно в JDBC"
permalink: /spring/jpa_deep
---
# 1. Производительность загрузки и N+1

*Зависимости и базовые настройки для всех подпунктов ниже*

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

## Причины N+1, диагностика по SQL-логам и статистике Hibernate

Проблема N+1 возникает, когда вы загружаете N родительских сущностей одним запросом, но для каждой из них ORM отправляет ещё по одному запросу, чтобы подтянуть связанную сторону. Чаще всего это происходит из-за ленивых ассоциаций (`LAZY`) и неосознанного доступа к ним в цикле. На небольших данных это незаметно, но под нагрузкой превращается в шквал коротких запросов, убивающих пулы соединений и кэши.

Корневая причина — **порядок доступа** к данным несогласован с планом выборки. Вы сначала получаете список, а затем, проходясь по нему, обращаетесь к `order.getCustomer().getName()` или `order.getLines().size()`. Для ORM это сигнал: «нужно догрузить ассоциацию прямо сейчас», и она делает отдельный `SELECT` на каждую итерацию. Так, всего за один «невинный» `for` можно получить сотни запросов.

Разработчик может не заметить проблему, потому что всё «магически» работает: коллекции подгружаются прозрачно, бизнес-логика корректна, а в профиле локальной базы данные малы. Но на продакшене, где у вас десятки тысяч строк, подобное поведение резко меняет профиль нагрузки. Поэтому диагностика N+1 — обязательная часть ревью и нагрузочного профилирования.

Первое, что стоит включить — **SQL-лог** Hibernate. Это позволит увидеть реальную последовательность запросов, их параметры и частоту. Лучше выводить и время выполнения, и bind-параметры, иначе вы не отличите одинаковые шаблоны запросов друг от друга. На дев-профиле это дешёво и невероятно полезно.

Второй инструмент — **статистика Hibernate** (`hibernate.generate_statistics=true`). Она считает количество запросов, попаданий в кэш первого уровня, ленивых инициализаций, загрузок коллекций. С её помощью можно зафиксировать, что на конкретный HTTP-эндпоинт уходит, например, 1 запрос на список и 50 запросов на `customer` — явный симптом N+1 для связи «многие-к-одному».

Третий подход — простые эвристики в коде тестов: замерять количество запросов через перехватчик или логгер и падать, если лимит превышен. В e2e-тестах такого стража легко реализовать, подписавшись на `org.hibernate.SQL` и подсчитав строки. Это дисциплинирует и защищает от регресса.

Важно понимать, что причина N+1 не в `LAZY` самом по себе. Ленивость — нормальный выбор по умолчанию, чтобы не тащить «полгорода». Проблема — в **неосознанном** доступе к ассоциациям вне подходящего запроса. Решение — согласовать **выборку** и **потребление**: либо заранее подтянуть нужные связи, либо работать на плоских DTO без ассоциаций.

Диагностика должна идти рука об руку с пониманием **границ транзакции**. Когда `open-in-view=false`, доступ к ленивой связи вне транзакции приведёт к `LazyInitializationException`. Это «случайная защита» от N+1: вы вынуждены либо подгружать связи там, где нужно, либо конвертировать в DTO. Но с `open-in-view=true` проблема маскируется — и поэтому этот флаг не стоит использовать на публичных API.

Симптомы N+1 легко увидеть по графикам БД: много коротких запросов, низкая средняя латентность, но большой `qps` и рост CPU на парсинге. Если добавить сетевую задержку (между приложением и БД), издержки умножаются. Поэтому даже «быстрые» запросы в количестве N+1 опасны — они бьют по коннекторам и сетям.

Наконец, держите в голове, что N+1 — поведенческая проблема, а не только про ассоциации. Похожим образом можно «настрелять» в БД вызовами справочных репозиториев внутри цикла. Решение то же: сгруппировать запросы, использовать `IN (:ids)` или `JOIN` и собрать всё нужное за 1–2 хода.

**application.yml — включаем SQL-лог и статистику**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true
logging:
  level:
    org.hibernate.SQL: debug
    org.hibernate.orm.jdbc.bind: trace   # параметры запросов
```

**Java — демонстрация N+1 и подсчёт запросов через Statistics**

```java
package com.example.nplus1;

import jakarta.persistence.*;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String name;
    protected Customer() {}
    Customer(String name){ this.name = name; }
}

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    Customer customer;
    @Column(nullable = false) Long totalCents;
    protected Order() {}
    Order(Customer c, long total){ this.customer = c; this.totalCents = total; }
}

@Repository
interface OrderRepository extends JpaRepository<Order, Long> { }

@Service
class OrderReportService {
    private final OrderRepository repo;
    private final SessionFactory sessionFactory;
    OrderReportService(OrderRepository repo, EntityManagerFactory emf) {
        this.repo = repo;
        this.sessionFactory = emf.unwrap(SessionFactory.class);
    }

    @Transactional(readOnly = true)
    public long naiveSumByCustomerNamePrefix(String prefix) {
        Statistics st = sessionFactory.getStatistics();
        st.clear(); // обнуляем счётчики на время вызова
        List<Order> orders = repo.findAll();            // 1 запрос
        long sum = 0;
        for (Order o : orders) {
            // для каждого заказа — отдельный SELECT customer -> N запросов
            if (o.customer.name.startsWith(prefix)) {
                sum += o.totalCents;
            }
        }
        long executed = st.getPrepareStatementCount();
        System.out.println("Executed SQL statements: " + executed);
        return sum;
    }
}
```

**Kotlin — та же демонстрация**

```kotlin
package com.example.nplus1

import jakarta.persistence.*
import org.hibernate.SessionFactory
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = ""
)

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

@Repository
interface OrderRepository : JpaRepository<Order, Long>

@Service
class OrderReportService(
    private val repo: OrderRepository,
    emf: EntityManagerFactory
) {
    private val sessionFactory: SessionFactory = emf.unwrap(SessionFactory::class.java)

    @Transactional(readOnly = true)
    fun naiveSumByCustomerNamePrefix(prefix: String): Long {
        val st = sessionFactory.statistics
        st.clear()
        val orders = repo.findAll()
        var sum = 0L
        for (o in orders) {
            if (o.customer!!.name.startsWith(prefix)) {
                sum += o.totalCents
            }
        }
        println("Executed SQL statements: ${st.prepareStatementCount}")
        return sum
    }
}
```

---

## Инструменты: `JOIN FETCH` (ограничения: пагинация с коллекциями, «несколько мешков»), `@EntityGraph`/`@NamedEntityGraph`, `@BatchSize` и `hibernate.default_batch_fetch_size`, `SUBSELECT`

Первый и самый прямой инструмент против N+1 — **`JOIN FETCH`**. Он говорит провайдеру JPA подтянуть указанную ассоциацию в том же запросе. Для связей «многие-к-одному» (`@ManyToOne`) это почти всегда безопасно: одна строка расширяется полями другой таблицы без умножения результата. Для коллекций (`@OneToMany`) вступают ограничения: строки множатся, пагинация ломается, нужно `distinct`, и есть риск «несколько мешков».

Под «несколькими мешками» (multiple bag fetch) Hibernate понимает ситуацию, когда вы пытаетесь фетчить **несколько** коллекций-`List` у одной сущности за раз. Фреймворк выбрасывает исключение, потому что не умеет корректно расплющивать и собирать такой декартов продукт. Решения: фетчить по одной коллекции за раз, использовать `Set` вместо `List` (иногда помогает), либо строить два отдельных запроса.

Пагинация и `JOIN FETCH` коллекций — давно известная боль. SQL-лимит применяется **к умноженному набору**, и вы получаете меньше уникальных «родителей» на странице. Типовой паттерн: сначала выбрать `id` родителей с пагинацией, затем за один запрос загрузить сущности `WHERE id IN (:ids)` и уже к ним догрузить нужные коллекции (ещё одним запросом или батч-фетчем). Это немного сложнее, но стабильно.

Второй инструмент — **`@EntityGraph`** и `@NamedEntityGraph`. Это способ декларативно указать «граф» атрибутов для подгрузки: какие `@ManyToOne` и `@OneToMany` подтянуть по пути. В отличие от `JOIN FETCH`, граф — часть метаданных, и вы можете применять его к стандартным методам репозитория (`findById`, `findAll`) без явного JPQL. Он уважает LAZY/EAGER и может работать как с `LOAD`, так и с `FETCH` семантикой.

Третий — **батч-фетчинг**: `@BatchSize` на ассоциации/сущности и/или глобально `hibernate.default_batch_fetch_size`. Когда вы проходите по списку заказов и обращаетесь к `order.customer`, Hibernate соберёт «пачку» идентификаторов и сделает **один** запрос `select * from customer where id in (...)` вместо N отдельных. Это прекрасно сглаживает N+1 для множества «многие-к-одному» и не ломает пагинацию.

Четвёртый — **`SUBSELECT`** (Hibernate-специфичный `@Fetch(FetchMode.SUBSELECT)` для коллекций). Он говорит: «когда я впервые обращусь к коллекции в этом наборе родительских сущностей, подгрузи все соответствующие элементы одной общей подвыборкой». Это хорошо работает, когда вы отображаете страницу, на которой нужны **все** дочерние элементы для всех родителей. Но будьте осторожны с очень большими выборками — субзапрос может стать тяжёлым.

Комбинируйте инструменты: например, для списков используйте батч-фетч для `@ManyToOne` и не фетчьте коллекции вовсе, а показывайте агрегаты (счётчики) через проекции. Для карточек — точечный `JOIN FETCH` или `@EntityGraph` ровно нужных связей. Для «богатых» экранов с несколькими коллекциями — два отдельных запроса вместо одного гигантского.

Ещё один практический нюанс — **`distinct`** в JPQL с `join fetch`. Он устраняет дубликаты «родителей» после расширения строк, но делает это на уровне ORM, а не SQL (часто). На больших объёмах это лишняя аллокация; лучше соблюдать умеренность в `fetch` и, если нужно, переходить на подход «ids + IN».

Также обращайте внимание на индексы. Любой `JOIN FETCH` — это реальный JOIN в базе. Если у вас нет индекса по внешнему ключу, запросы будут тормозить, и никакая магия ORM не спасёт. Проверьте планы выполнения и добавьте индексы по FK и сортируемым полям, особенно если делаете сортировку по полям из присоединённой таблицы.

И помните: `@EntityGraph` — не серебряная пуля. Он не всегда приводит к `join fetch`; часто это набор **дополнительных селектов**, но сгруппированных батчем. Это нормально: цель — минимизировать общее количество раунд-трипов и держать граф контролируемым, а не «затащить всё одной SQL-портянкой».

**Java — примеры JOIN FETCH, EntityGraph, BatchSize, SUBSELECT**

```java
package com.example.fetch;

import jakarta.persistence.*;
import org.hibernate.annotations.Fetch;
import org.hibernate.annotations.FetchMode;
import org.hibernate.annotations.BatchSize;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String name;
    protected Customer() {}
    Customer(String name){ this.name = name; }
}

@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "order.withCustomer",
    attributeNodes = @NamedAttributeNode("customer")
)
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    @BatchSize(size = 50) // батч-догрузка customer по IN
    Customer customer;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
    @Fetch(FetchMode.SUBSELECT) // подтянуть все lines для набора заказов одним субзапросом
    List<OrderLine> lines;

    protected Order() {}
}

@Entity
@Table(name = "order_lines")
class OrderLine {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    Order order;

    @Column(nullable = false) String sku;
    @Column(nullable = false) int qty;

    protected OrderLine() {}
    OrderLine(Order order, String sku, int qty){ this.order = order; this.sku = sku; this.qty = qty; }
}

@Repository
interface OrderRepository extends JpaRepository<Order, Long> {

    // Безопасный FETCH для ManyToOne
    @Query("""
       select o from Order o
       join fetch o.customer
       where o.id = :id
    """)
    Optional<Order> findWithCustomer(@Param("id") Long id);

    // c EntityGraph: можно применять к findAll/findById
    @EntityGraph(value = "order.withCustomer", type = EntityGraph.EntityGraphType.FETCH)
    @Query("select o from Order o where o.id = :id")
    Optional<Order> byIdWithGraph(@Param("id") Long id);
}
```

**Kotlin — те же приёмы**

```kotlin
package com.example.fetch

import jakarta.persistence.*
import org.hibernate.annotations.BatchSize
import org.hibernate.annotations.Fetch
import org.hibernate.annotations.FetchMode
import org.springframework.data.jpa.repository.EntityGraph
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import java.util.*

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = ""
)

@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "order.withCustomer",
    attributeNodes = [NamedAttributeNode("customer")]
)
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    @BatchSize(size = 50)
    var customer: Customer? = null,

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY, cascade = [CascadeType.ALL], orphanRemoval = true)
    @Fetch(FetchMode.SUBSELECT)
    var lines: MutableList<OrderLine> = mutableListOf()
)

@Entity
@Table(name = "order_lines")
class OrderLine(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    var order: Order? = null,
    @Column(nullable = false) var sku: String = "",
    @Column(nullable = false) var qty: Int = 0
)

interface OrderRepository : JpaRepository<Order, Long> {

    @Query(
        """
        select o from Order o
        join fetch o.customer
        where o.id = :id
        """
    )
    fun findWithCustomer(@Param("id") id: Long): Optional<Order>

    @EntityGraph(value = "order.withCustomer", type = EntityGraph.EntityGraphType.FETCH)
    @Query("select o from Order o where o.id = :id")
    fun byIdWithGraph(@Param("id") id: Long): Optional<Order>
}
```

**application.yml — глобальный батч-фетч**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50
```

---

## Стратегии: «тонкие» DTO-проекции для списков, точечные графы для карточек, разделение чтения/деталей

Первая стратегия — **тонкие DTO для списков**. Идея проста: списочный экран (лента) не должен тянуть графы сущностей. Он показывает несколько полей: идентификатор, заголовок, имя клиента, суммарную стоимость. Всё это можно получить одним JPQL c конструкторной проекцией или интерфейсной проекцией — быстро, дёшево и без участия контекста персистентности.

Вторая — **точечные графы/`JOIN FETCH` для карточек**. Экран детали «заказа» действительно требует подтянуть строки заказа и, возможно, платежи и доставки. Но и здесь нужно избегать «монолитного» запроса: чаще корректнее сделать 1 запрос на заказ + клиента и 1 запрос на строки (или `SUBSELECT`). Это уменьшит декартовы умножения и упростит пагинацию вложенных коллекций.

Третья — **разделение чтения и модификаций**. В `QueryService` возвращайте DTO, в `CommandService` — работайте с сущностями и транзакциями на запись. Это упрощает контроль за ленивыми ассоциациями, устраняет риск `LazyInitializationException` в веб-слое и улучшает производительность: DTO не попадают в контекст персистентности.

Четвёртая — **агрегации в списках**. Вместо того чтобы подтягивать коллекции ради счётчиков, используйте агрегаты (`count`, `sum`) в JPQL/SQL и кладите их в поля DTO. Так вы избежите N+1 и получите точные значения без лишнего трафика. Для сложных агрегатов (например, подсчёт по статусу) используйте CTE/подзапросы или нативный SQL с аккуратным маппингом.

Пятая — **стабильная пагинация**. В списочных запросах на DTO обязательно задавайте `order by` по индексируемым полям и вторичный «тай-брейкер» по `id`. Это не только предотвращает «пляску» элементов, но и помогает батч-фетчу: набор родительских id получается детерминированным, и `IN (:ids)` попадает в кэш планов.

Шестая — **entity graph как контракт**. Если вы всё-таки возвращаете сущности (например, во внутренних сервисах), закрепите набор подгружаемых атрибутов `@NamedEntityGraph` и используйте его в репозиториях. Это документирует ожидания и предотвращает случайные N+1 из-за новых обращений к ассоциациям в коде.

Седьмая — **порог «всё в один запрос»**. Если у карточки несколько коллекций (строки, платежи, трекинги), `JOIN FETCH` на всё сразу приведёт к multiple-bag-fetch и/или взрывному умножению. Дешевле выполнить 2–3 отдельных запроса, чем один «монстр». Измеряйте: профилирование часто показывает, что «несколько простых запросов» быстрее и устойчивее.

Восьмая — **кэширование границ**. Для неизменяемых справочников (статусы, типы, страны) используйте L2-кэш или локальный кэш сервиса. Тогда DTO для списков можно собирать без джоинов по справочникам, подставляя значения из кэша по кодам. Это снижает нагрузку на БД и упрощает запросы.

Девятая — **миграция «тяжёлых» экранов**. Если какой-то список неизбежно требует десятка связей и вычислений, рассмотрите подготовленные представления/материализованные таблицы и чтение через JDBC/jOOQ. Это честная «инфра-оптимизация», и её лучше сделать сознательно, чем мучить ORM.

Десятая — **проверяйте N+1 в тестах**. На каждый публичный эндпоинт со списком заведите тест, который проверяет количество SQL-запросов (через перехватчик/логгер). Это не про микро-оптимизации, а про «охрану периметра» от случайного доступа к ассоциациям в цикле.

**Java — DTO для списка + детальная загрузка с графом**

```java
package com.example.strategy;

import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.Optional;

@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "order.detail",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "lines")
    }
)
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    Customer customer;
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
    java.util.List<OrderLine> lines = new java.util.ArrayList<>();
    @Column(nullable = false) Instant createdAt = Instant.now();
    @Column(nullable = false) Long totalCents = 0L;
    protected Order() {}
}

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String name;
    protected Customer() {}
    Customer(String name){ this.name = name; }
}

@Entity
@Table(name = "order_lines")
class OrderLine {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    Order order;
    @Column(nullable = false) String sku;
    @Column(nullable = false) int qty;
    protected OrderLine() {}
}

record OrderListDto(Long id, String customerName, Long totalCents, Instant createdAt) { }

@Repository
interface OrderRepository extends JpaRepository<Order, Long> {

    // Списочный запрос — только нужные поля
    @Query("""
       select new com.example.strategy.OrderListDto(
           o.id, o.customer.name, o.totalCents, o.createdAt
       )
       from Order o
       where (:q is null or lower(o.customer.name) like lower(concat('%', :q, '%')))
       order by o.createdAt desc, o.id desc
    """)
    Page<OrderListDto> list(@Param("q") String q, Pageable pageable);

    // Деталь — граф (customer + lines)
    @EntityGraph(value = "order.detail", type = EntityGraph.EntityGraphType.FETCH)
    @Query("select o from Order o where o.id = :id")
    Optional<Order> detail(@Param("id") Long id);
}

@Service
class OrderQueryService {
    private final OrderRepository repo;
    OrderQueryService(OrderRepository repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public Page<OrderListDto> findPage(String q, int page, int size) {
        Pageable p = PageRequest.of(page, size, Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id")));
        return repo.list(q, p);
    }

    @Transactional(readOnly = true)
    public Order detail(Long id) {
        return repo.detail(id).orElseThrow();
    }
}
```

**Kotlin — те же стратегии**

```kotlin
package com.example.strategy

import jakarta.persistence.*
import org.springframework.data.domain.*
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.Instant
import java.util.*

@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "order.detail",
    attributeNodes = [
        NamedAttributeNode("customer"),
        NamedAttributeNode("lines")
    ]
)
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    var customer: Customer? = null,
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY, cascade = [CascadeType.ALL], orphanRemoval = true)
    var lines: MutableList<OrderLine> = mutableListOf(),
    @Column(nullable = false) var createdAt: Instant = Instant.now(),
    @Column(nullable = false) var totalCents: Long = 0
)

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = ""
)

@Entity
@Table(name = "order_lines")
class OrderLine(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    var order: Order? = null,
    @Column(nullable = false) var sku: String = "",
    @Column(nullable = false) var qty: Int = 0
)

data class OrderListDto(
    val id: Long,
    val customerName: String,
    val totalCents: Long,
    val createdAt: Instant
)

@Repository
interface OrderRepository : JpaRepository<Order, Long> {

    @Query(
        """
        select new com.example.strategy.OrderListDto(
            o.id, o.customer.name, o.totalCents, o.createdAt
        )
        from Order o
        where (:q is null or lower(o.customer.name) like lower(concat('%', :q, '%')))
        order by o.createdAt desc, o.id desc
        """
    )
    fun list(@Param("q") q: String?, pageable: Pageable): Page<OrderListDto>

    @EntityGraph(value = "order.detail", type = EntityGraph.EntityGraphType.FETCH)
    @Query("select o from Order o where o.id = :id")
    fun detail(@Param("id") id: Long): Optional<Order>
}

@Service
class OrderQueryService(private val repo: OrderRepository) {

    @Transactional(readOnly = true)
    fun findPage(q: String?, page: Int, size: Int): Page<OrderListDto> {
        val sort = Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"))
        return repo.list(q, PageRequest.of(page, size, sort))
    }

    @Transactional(readOnly = true)
    fun detail(id: Long): Order = repo.detail(id).orElseThrow()
}
```

# 2. Проекции и сложные запросы

*Зависимости и базовые настройки для всей подтемы (как в прошлой главе, но повторю для самодостаточности примеров).*

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

## Проекции: интерфейсные/DTO-конструкторы в Spring Data, закрытые vs открытые (SpEL) и их стоимость

Проекции — это способ читать **только нужные поля**, не загружая целые сущности и их графы. Они уменьшают I/O, память и давление на Persistence Context (который вообще можно обойти), поэтому критичны для списков и публичных API. В Spring Data есть два основных вида: **интерфейсные проекции** (закрытые) и **конструкторные DTO** (через `select new ...`). Оба типа «тоньше» сущностей и дисциплинируют вас не тянуть ленивые ассоциации.

Закрытые интерфейсные проекции — это интерфейсы с геттерами, имена которых соответствуют алиасам/свойствам в запросе. Spring создает прокси, который маппит значения по имени. Прелесть в том, что такие проекции легковесны: не нужен конструктор, нет лишних аллокаций, их можно использовать и с derived-запросами, и с `@Query` (JPQL/SQL, если аккуратно проставить алиасы). Они хорошо подходят для 90% списочных экранов.

Конструкторные DTO-проекции — это «жёсткий» вариант: вы объявляете класс/record с конструктором, а в JPQL пишете `select new pkg.Dto(a.id, a.title) from ...`. Такой подход прозрачен и типобезопасен: компилятор проверит наличие конструктора, IDE — переименования полей. Минус — это строго JPQL (для native нужны другие техники) и чуть больше бойлерплейта в коде.

Есть ещё **открытые проекции** с `@Value` и SpEL внутри геттеров интерфейса. Они позволяют вычислять поля на лету, обращаться к другим бинам и «склеивать» строки (`@Value("#{target.firstName + ' ' + target.lastName}")`). Это мощно, но **дорого**: SpEL-компиляция/рефлексия на каждом объекте. На больших списках это ощутимо бьёт по CPU и GC, поэтому используйте открытые проекции экономно и только там, где вычисление действительно нужно на стороне приложения.

Ключевая деталь — соответствие имен. Для интерфейсных проекций геттер `getCustomerName()` будет ожидать либо свойство `customerName` у сущности, либо алиас `as customerName` в запросе. Для вложенных путей можно объявлять вложенные интерфейсы (`CustomerView { String getName(); }`) и возвращать их в основном интерфейсе — Spring умеет раскладывать «через точку».

Проекции не являются managed-сущностями. Это плюс (не попадают в Persistence Context, не грузят L1-кэш), но и ограничение: вы не можете на них опираться для dirty checking или использовать `entityManager.refresh`. Они **read-only**. Пытайтесь держать границу: **QueryService → проекции (DTO)**, **CommandService → сущности**.

Пагинация с проекциями работает идеально: `Page<Dto>`/`Slice<Dto>` возвращается ровно с теми полями, что вам нужны. Но не забывайте про `countQuery` в `@Query`, если ваш JPQL сложный — иначе Spring попытается сгенерировать `count` сам и часто ошибётся, особенно с `distinct` и `join fetch`.

Проекции отлично комбинируются с агрегациями. В списке заказов вместо загрузки `lines` достаточно отдать `OrderListDto(id, customerName, totalCents, linesCount)`, где `linesCount` — это `count(l)` в JPQL. Такой подход «убивает» N+1 и даёт пользователю ровно то, что ему надо на списке. Подробности — в карточке, отдельным запросом.

В Kotlin интерфейсные проекции работают так же, но чаще приятнее писать **data class** и конструкторную проекцию — получается явный контракт и дружелюбная к сериализации структура. Следите за nullability: если поле может быть `null` в базе/выборке, объявляйте его как nullable тип в DTO, иначе вы получите NPE уже в маппере.

Ещё одна тонкость — сортировка по вычисляемым полям. Открытые проекции с `@Value` не участвуют в `order by` на уровне БД. Если вам нужна сортировка по «полному имени», формируйте её в SQL/JPQL (`order by c.firstName, c.lastName`) и маппьте результат в поле `fullName` в DTO. Иначе сортировка будет на приложении и может «прыгать» между страницами.

И, наконец, думайте о стабильности контрактов. DTO — это интерфейс вашего чтения между слоями. Меняя имена/типы полей, вы ломаёте клиенты. Зафиксируйте `equals/hashCode/toString` (в Java — record, в Kotlin — data class) и прикройте критичные запросы интеграционными тестами.

**Java — интерфейсная (закрытая) и открытая проекции + DTO-конструктор**

```java
package com.example.projection;

import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.Instant;

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    Customer customer;

    @Column(nullable = false) Long totalCents;
    @Column(nullable = false) Instant createdAt = Instant.now();
    protected Order() {}
}

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, length = 120) String firstName;
    @Column(nullable = false, length = 120) String lastName;
    protected Customer() {}
    Customer(String f, String l){ this.firstName=f; this.lastName=l; }
}

// Закрытая интерфейсная проекция: имена = алиасы/пути
public interface OrderSummary {
    Long getId();
    String getCustomerFirstName();
    String getCustomerLastName();
    Long getTotalCents();
    Instant getCreatedAt();
}

// Открытая (дороже): SpEL/конкатенация
public interface OrderSummaryOpen {
    Long getId();
    Long getTotalCents();
    @org.springframework.beans.factory.annotation.Value("#{target.customer.firstName + ' ' + target.customer.lastName}")
    String getCustomerFullName();
}

// Конструкторная DTO-проекция через JPQL
public record OrderSummaryDto(Long id, String customerFullName, Long totalCents, Instant createdAt) {}

@Repository
interface OrderRepository extends JpaRepository<Order, Long> {

    // Интерфейсная (закрытая) — важно выставить алиасы
    @Query("""
        select o.id as id,
               c.firstName as customerFirstName,
               c.lastName  as customerLastName,
               o.totalCents as totalCents,
               o.createdAt  as createdAt
        from Order o
        join o.customer c
        where (:q is null or lower(c.lastName) like lower(concat('%', :q, '%')))
        order by o.createdAt desc, o.id desc
    """)
    Page<OrderSummary> pageClosed(@Param("q") String lastNameLike, Pageable pageable);

    // Открытая — на больших списках может быть дорогой
    @Query("select o from Order o join fetch o.customer c where c.lastName like concat('%', :q, '%')")
    Page<OrderSummaryOpen> pageOpen(@Param("q") String q, Pageable p);

    // Конструкторная DTO — типобезопасна и шустра
    @Query("""
        select new com.example.projection.OrderSummaryDto(
            o.id,
            concat(c.firstName, ' ', c.lastName),
            o.totalCents,
            o.createdAt
        )
        from Order o join o.customer c
        where (:q is null or lower(c.lastName) like lower(concat('%', :q, '%')))
        order by o.createdAt desc, o.id desc
    """)
    Page<OrderSummaryDto> pageDto(@Param("q") String q, Pageable p);
}
```

**Kotlin — те же идеи (интерфейс + data class)**

```kotlin
package com.example.projection

import jakarta.persistence.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.Query
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.query.Param
import org.springframework.beans.factory.annotation.Value
import java.time.Instant

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    var customer: Customer? = null,
    @Column(nullable = false) var totalCents: Long = 0,
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var firstName: String = "",
    @Column(nullable = false) var lastName: String = ""
)

interface OrderSummary {
    fun getId(): Long
    fun getCustomerFirstName(): String
    fun getCustomerLastName(): String
    fun getTotalCents(): Long
    fun getCreatedAt(): Instant
}

interface OrderSummaryOpen {
    fun getId(): Long
    fun getTotalCents(): Long
    @Value("#{target.customer.firstName + ' ' + target.customer.lastName}")
    fun getCustomerFullName(): String
}

data class OrderSummaryDto(
    val id: Long,
    val customerFullName: String,
    val totalCents: Long,
    val createdAt: Instant
)

interface OrderRepository : JpaRepository<Order, Long> {

    @Query(
        """
        select o.id as id,
               c.firstName as customerFirstName,
               c.lastName  as customerLastName,
               o.totalCents as totalCents,
               o.createdAt  as createdAt
        from Order o
        join o.customer c
        where (:q is null or lower(c.lastName) like lower(concat('%', :q, '%')))
        order by o.createdAt desc, o.id desc
        """
    )
    fun pageClosed(@Param("q") lastNameLike: String?, p: Pageable): Page<OrderSummary>

    @Query("select o from Order o join fetch o.customer c where c.lastName like concat('%', :q, '%')")
    fun pageOpen(@Param("q") q: String, p: Pageable): Page<OrderSummaryOpen>

    @Query(
        """
        select new com.example.projection.OrderSummaryDto(
            o.id,
            concat(c.firstName, ' ', c.lastName),
            o.totalCents,
            o.createdAt
        )
        from Order o join o.customer c
        where (:q is null or lower(c.lastName) like lower(concat('%', :q, '%')))
        order by o.createdAt desc, o.id desc
        """
    )
    fun pageDto(@Param("q") q: String?, p: Pageable): Page<OrderSummaryDto>
}
```

---

## Нативные SQL: маппинг на DTO/интерфейсы, `SqlResultSetMapping` (где оправдано)

Нативные SQL-запросы уместны там, где **возможностей JPQL недостаточно**: оконные функции, CTE, `jsonb`/массивы в Postgres, `ON CONFLICT`/апсерты, специфичные плановые хинты. В таких случаях мы осознанно жертвуем переносимостью в пользу мощности БД, но хотим сохранить удобство маппинга результатов в понятные DTO/интерфейсы.

Простейший путь — интерфейсные проекции поверх `@Query(nativeQuery = true)`. Здесь важно корректно проставить **алиасы колонок**, чтобы они совпали с именами геттеров. Так Spring сможет собрать интерфейсную проекцию напрямую из результата `ResultSet` без промежуточных сущностей. Этот подход хорошо работает с пагинацией (`Page<T>`), если вы дополнительно зададите `countQuery`.

Если нужна конструкторная DTO-проекция с native SQL, у Spring Data «из коробки» нет `select new ...` для native. Тогда есть два пути: маппить в интерфейс (как выше) **либо** использовать стандарт JPA — `@SqlResultSetMapping` с `@ConstructorResult` и `@NamedNativeQuery`. Это чуть более многословно, зато надёжно и типобезопасно — JPA сам вызовет нужный конструктор DTO для каждой строки.

`@SqlResultSetMapping` особенно уместен, когда вы возвращаете «богатую» структуру: агрегаты, вычисляемые поля, поля из JSON. Вы явно описываете соответствие колонок конструктору и получаете стабильную схему. Добавьте сюда `@NamedNativeQuery` с параметрами — и репозиторий сможет ссылаться на запрос по имени, не дублируя текст.

С JSONB (Postgres) удобнее сразу приводить типы в SQL: `data->>'phone' as phone`, `coalesce((data->>'age')::int,0) as age`. Тогда DTO получает уже «правильные» Java-типы, без ручного парсинга. Индексы GIN/GiST по JSONB работают через операторную семантику (`@>`, `?`, `#>>`) — это важная часть производительности сложных запросов.

Помните о пагинации. В `@Query(nativeQuery = true)` для `Page<T>` почти всегда нужен отдельный `countQuery` — простой и без `order by`/CTE, если возможно. Не пытайтесь «оборачивать» CTE в `select count(*) from (...)` без необходимости — иногда проще завести второй запрос без тяжёлых join’ов, который эквивалентен условиям фильтрации.

Ещё один нюанс — **управление контекстом**. Результаты native-запроса, смапленные на DTO/интерфейсы, не попадают в Persistence Context, и это хорошо. Но если вы маппите native на сущности, помните, что значения могут «переехать» в L1-кэш, и последующие обращения к тем же id вернут **внутренние managed-экземпляры**. Это легко порождает рассинхрон при смешивании native-апдейтов и чтений. Для отчётных чтений держитесь DTO/интерфейсов.

Тестируйте native через Testcontainers той же версии БД, что и прод. Многие особенности (например, `jsonb_path_query` или `generated as identity`) по-разному работают в версиях 13/14/15/16. Локальный H2 «в Postgres-режиме» здесь не спасёт: он не поддерживает ни jsonb, ни операторные индексы.

Не перегибайте с native: начните с JPQL/проекций/EntityGraph, измерьте, и только потом переходите к SQL, если выгода очевидна. Чем меньше у вас «нестандартного» SQL в коде сервиса, тем проще сопровождать, обновлять и обучать команду.

И наконец, документируйте контракт native-запросов рядом с DTO (комментарии + тесты с параметрами и «золотыми» значениями). Это избавит от регрессов при изменениях схемы/индексов и поможет коллегам быстро понять, зачем и где используется конкретный запрос.

**Java — native → интерфейсная проекция + `@SqlResultSetMapping` конструктор**

```java
package com.example.nativeproj;

import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.Instant;

public interface CustomerReportRow {
    Long getId();
    String getFullName();
    Integer getAge();
    String getPhone();
    Instant getCreatedAt();
}

@Entity
@Table(name = "customers")
@NamedNativeQuery(
    name = "Customer.nativeReportDto",
    query = """
        with base as (
            select c.id,
                   (c.first_name || ' ' || c.last_name) as full_name,
                   (c.data->>'age')::int as age,
                   (c.data->>'phone') as phone,
                   c.created_at
            from customers c
            where (:q is null or c.last_name ilike concat('%', :q, '%'))
        )
        select * from base
        order by created_at desc, id desc
    """,
    resultSetMapping = "CustomerReportDtoMapping"
)
@SqlResultSetMapping(
    name = "CustomerReportDtoMapping",
    classes = @ConstructorResult(
        targetClass = com.example.nativeproj.CustomerReportDto.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "full_name", type = String.class),
            @ColumnResult(name = "age", type = Integer.class),
            @ColumnResult(name = "phone", type = String.class),
            @ColumnResult(name = "created_at", type = java.time.Instant.class)
        }
    )
)
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(name = "first_name", nullable = false) String firstName;
    @Column(name = "last_name", nullable = false) String lastName;
    @Column(columnDefinition = "jsonb") String data; // для примера
    @Column(name = "created_at", nullable = false) Instant createdAt = Instant.now();
    protected Customer() {}
}

public record CustomerReportDto(Long id, String fullName, Integer age, String phone, Instant createdAt) { }

@Repository
interface CustomerRepository extends JpaRepository<Customer, Long> {

    // 1) Native + интерфейсная проекция (обратите внимание на алиасы)
    @Query(
        value = """
            select c.id as id,
                   (c.first_name || ' ' || c.last_name) as fullName,
                   (c.data->>'age')::int as age,
                   (c.data->>'phone') as phone,
                   c.created_at as createdAt
            from customers c
            where (:q is null or c.last_name ilike concat('%', :q, '%'))
            order by c.created_at desc, c.id desc
        """,
        countQuery = """
            select count(*) from customers c
            where (:q is null or c.last_name ilike concat('%', :q, '%'))
        """,
        nativeQuery = true
    )
    Page<CustomerReportRow> pageNativeIface(@Param("q") String q, Pageable p);

    // 2) NamedNativeQuery + SqlResultSetMapping → конструктор DTO
    @Query(name = "Customer.nativeReportDto", nativeQuery = true)
    Page<CustomerReportDto> pageNativeDto(@Param("q") String q, Pageable p);
}
```

**Kotlin — аналогичный пример (интерфейс + data class через @SqlResultSetMapping)**

```kotlin
package com.example.nativeproj

import jakarta.persistence.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.Query
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.query.Param
import java.time.Instant

interface CustomerReportRow {
    fun getId(): Long
    fun getFullName(): String
    fun getAge(): Int?
    fun getPhone(): String?
    fun getCreatedAt(): Instant
}

@Entity
@Table(name = "customers")
@NamedNativeQuery(
    name = "Customer.nativeReportDto",
    query = """
        with base as (
            select c.id,
                   (c.first_name || ' ' || c.last_name) as full_name,
                   (c.data->>'age')::int as age,
                   (c.data->>'phone') as phone,
                   c.created_at
            from customers c
            where (:q is null or c.last_name ilike concat('%', :q, '%'))
        )
        select * from base
        order by created_at desc, id desc
    """,
    resultSetMapping = "CustomerReportDtoMapping"
)
@SqlResultSetMapping(
    name = "CustomerReportDtoMapping",
    classes = [
        ConstructorResult(
            targetClass = CustomerReportDto::class,
            columns = [
                ColumnResult(name = "id", type = java.lang.Long::class),
                ColumnResult(name = "full_name", type = java.lang.String::class),
                ColumnResult(name = "age", type = java.lang.Integer::class),
                ColumnResult(name = "phone", type = java.lang.String::class),
                ColumnResult(name = "created_at", type = java.time.Instant::class)
            ]
        )
    ]
)
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(name = "first_name", nullable = false)
    var firstName: String = "",
    @Column(name = "last_name", nullable = false)
    var lastName: String = "",
    @Column(columnDefinition = "jsonb")
    var data: String? = null,
    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now()
)

data class CustomerReportDto(
    val id: Long,
    val fullName: String,
    val age: Int?,
    val phone: String?,
    val createdAt: Instant
)

interface CustomerRepository : JpaRepository<Customer, Long> {

    @Query(
        value = """
            select c.id as id,
                   (c.first_name || ' ' || c.last_name) as fullName,
                   (c.data->>'age')::int as age,
                   (c.data->>'phone') as phone,
                   c.created_at as createdAt
            from customers c
            where (:q is null or c.last_name ilike concat('%', :q, '%'))
            order by c.created_at desc, c.id desc
        """,
        countQuery = """
            select count(*) from customers c
            where (:q is null or c.last_name ilike concat('%', :q, '%'))
        """,
        nativeQuery = true
    )
    fun pageNativeIface(@Param("q") q: String?, p: Pageable): Page<CustomerReportRow>

    @Query(name = "Customer.nativeReportDto", nativeQuery = true)
    fun pageNativeDto(@Param("q") q: String?, p: Pageable): Page<CustomerReportDto>
}
```

---

## Частичные обновления: `@Modifying` JPQL/SQL, возвращаемые счётчики, осторожно с кешами и L1-контекстом

Частичные обновления — это DML-запросы `update ... where ...`/`delete ...` на уровне БД, которые минуют загрузку сущностей и dirty checking. Они полезны для массовых/типовых операций: смена статуса по условию, инкремент счётчика, «закрыть все просроченные». В Spring Data JPA такие методы объявляются через `@Modifying` на `@Query` и возвращают **количество затронутых строк**.

Важно понимать семантику: bulk-операции **обходят** Persistence Context. Hibernate не знает, что строки изменились, и managed-экземпляры остаются со старыми полями. Поэтому типичная практика — `@Modifying(clearAutomatically = true, flushAutomatically = true)`. Первая опция очищает L1-кэш после DML, вторая — гарантирует, что ваши незакомиченные изменения будут «сброшены» до выполнения bulk, чтобы не перезаписать их старым состоянием.

Ещё одна критическая деталь — оптимистическая блокировка. JPQL bulk update **не учитывает `@Version`** автоматически и не инкрементирует версию. Если вам важна корректность с версиями, добавьте её в `where` и увеличьте вручную: `set version = version + 1 where id = :id and version = :expected`. В противном случае вы можете перезаписать изменения параллельных транзакций.

Возвращаемые счётчики — отличный инструмент для контроля бизнес-инвариантов. Например, вы ожидаете, что сменится статус ровно у одной строки — проверяйте, что метод вернул 1; если 0, значит, запись не найдена или версия не совпала — возвращайте 404/409. Это превращает bulk-обновление в атомарную команду с понятным исходом.

С native SQL вы получаете больше свободы: `updated_at = now()`, `coalesce`/`case` и т. п. В Postgres есть «сахар» `returning`, но Spring Data `@Modifying` по умолчанию вернёт только число затронутых строк. Если вам нужны сами id/значения «после», либо используйте `JdbcTemplate`/jOOQ, либо маппьте результат `returning` отдельным методом с `nativeQuery=true` и типом `List<Long>`/DTO (без `@Modifying`, так как это уже «select» с точки зрения драйвера).

Работа с L2-кэшем при bulk-операциях требует явной инвалидации регионов. Если у вас включён кэш 2-го уровня для сущности `Order`, после `update Order o set ...` его данные устареют. Либо делайте `entityManager.getEntityManagerFactory().getCache().evict(Order.class)`, либо используйте поставщика кэша с встроенной инвалидацией по событиям (дороже). В противном случае вы прочитаете «старое» из кэша.

Не забывайте про таймауты: bulk-операции могут трогать много строк. Задавайте `@Transactional(timeout = ...)` или хинты для запроса (в Hibernate — `org.hibernate.jpa.HibernateHints.HINT_TIMEOUT`) и мониторьте планы выполнения. Часто выгоднее резать операцию на порции (например, по id-диапазонам) и выполнять в цикле, чем держать долгую транзакцию.

Bulk-операции не вызывают entity listeners (`@PreUpdate/@PostUpdate`), не срабатывают каскады JPA. Если у вас важны побочные эффекты (журналы, аудиты, доменные правила), либо реализуйте их в самой SQL-команде/триггерах БД, либо делайте изменения через управляемые сущности (дороже, но корректнее).

Ещё один паттерн — «апсерты» и «инкременты в базе». Вместо «прочитай-суммируй-запиши», делайте `update ... set counter = counter + :delta where id = :id`. Это атомарно и дружит с конкуренцией. Возвращаемый счётчик строк подскажет, была ли запись. Для upsert в Postgres используйте `insert ... on conflict (key) do update set ...`, но это уже зона native/JDBC/jOOQ.

И наконец, профилируйте. Часто «пять точечных update» по индексу быстрее, чем «один гигантский update» без подходящего индекса. Добавьте where-индексы, измерьте на боевых объёмах (с Testcontainers и репрезентативным набором данных), а потом решайте, оставаться ли на JPA bulk или спускаться на JDBC.

**Java — `@Modifying` JPQL и native, работа со счётчиком и L1-кэшем**

```java
package com.example.bulk;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Entity
@Table(name = "tickets")
class Ticket {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @Column(nullable = false) String status; // NEW, IN_PROGRESS, CLOSED
    @Column(nullable = false) Instant updatedAt = Instant.now();

    @Version
    Long version;

    protected Ticket() {}
}

@Repository
interface TicketRepository extends JpaRepository<Ticket, Long> {

    // JPQL bulk: учитываем версию вручную, чистим L1-кэш
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
        update Ticket t
           set t.status = :status,
               t.updatedAt = :now,
               t.version = t.version + 1
         where t.id = :id
           and (t.version is null or t.version = :expectedVersion)
    """)
    int updateStatus(@Param("id") Long id,
                     @Param("expectedVersion") Long expectedVersion,
                     @Param("status") String status,
                     @Param("now") Instant now);

    // Native bulk с возможностью использовать диалектные фичи
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(value = """
        update tickets
           set status = :status,
               updated_at = now()
         where status = :fromStatus
    """, nativeQuery = true)
    int massClose(@Param("fromStatus") String fromStatus,
                  @Param("status") String toStatus);
}

@Service
class TicketService {
    private final TicketRepository repo;
    private final EntityManagerFactory emf;

    TicketService(TicketRepository repo, EntityManagerFactory emf) {
        this.repo = repo; this.emf = emf;
    }

    @Transactional
    public void close(Long id, Long expectedVersion) {
        int changed = repo.updateStatus(id, expectedVersion, "CLOSED", Instant.now());
        if (changed == 0) {
            throw new IllegalStateException("Optimistic conflict or not found: id=" + id);
        }
        // При L2-кэше стоит ещё и его почистить:
        emf.getCache().evict(Ticket.class, id);
    }

    @Transactional
    public int closeAllInProgress() {
        int n = repo.massClose("IN_PROGRESS", "CLOSED");
        // Можно инвалидировать весь регион L2:
        emf.getCache().evict(Ticket.class);
        return n;
    }
}
```

**Kotlin — `@Modifying`, счётчики и инвалидация кэша**

```kotlin
package com.example.bulk

import jakarta.persistence.*
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

@Entity
@Table(name = "tickets")
class Ticket(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var status: String = "NEW",
    @Column(nullable = false) var updatedAt: Instant = Instant.now(),
    @Version var version: Long? = null
)

@Repository
interface TicketRepository : JpaRepository<Ticket, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        """
        update Ticket t
           set t.status = :status,
               t.updatedAt = :now,
               t.version = t.version + 1
         where t.id = :id
           and (t.version is null or t.version = :expectedVersion)
        """
    )
    fun updateStatus(
        @Param("id") id: Long,
        @Param("expectedVersion") expectedVersion: Long?,
        @Param("status") status: String,
        @Param("now") now: Instant
    ): Int

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        value = """
        update tickets
           set status = :toStatus,
               updated_at = now()
         where status = :fromStatus
        """,
        nativeQuery = true
    )
    fun massClose(
        @Param("fromStatus") fromStatus: String,
        @Param("toStatus") toStatus: String
    ): Int
}

@Service
class TicketService(
    private val repo: TicketRepository,
    private val emf: EntityManagerFactory
) {

    @Transactional
    fun close(id: Long, expectedVersion: Long?) {
        val changed = repo.updateStatus(id, expectedVersion, "CLOSED", Instant.now())
        if (changed == 0) error("Optimistic conflict or not found: id=$id")
        emf.cache.evict(Ticket::class.java, id)
    }

    @Transactional
    fun closeAllInProgress(): Int {
        val n = repo.massClose("IN_PROGRESS", "CLOSED")
        emf.cache.evict(Ticket::class.java)
        return n
    }
}
```

# 3. Вставки/обновления батчами и генерация идентификаторов

*Зависимости и базовые настройки для всей подтемы*

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

**application.yml (важные параметры для батчей)**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app?reWriteBatchedInserts=true
    username: app
    password: app
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true     # разрешить батчить UPDATE версионируемых сущностей
        generate_statistics: true
```

---

## Влияние стратегий `@GeneratedValue` на батчи (SEQUENCE vs IDENTITY; оптимизаторы `pooled/pooled-lo`)

Первое, о чём нужно договориться команде: **батч-вставка в Hibernate полноценно работает только с SEQUENCE**, а не с IDENTITY. Причина в том, что при `IDENTITY` (AUTO_INCREMENT/SERIAL/IDENTITY) провайдер **должен немедленно выполнить INSERT**, чтобы получить сгенерированный ключ (`getGeneratedKeys`/`RETURNING`) и привязать его к объекту. Это вынуждает выполнять каждую вставку отдельно, и батч распадается. В реальной жизни вы увидите один INSERT на каждую сущность, пусть даже драйвер и умеет переписывать батч — Hibernate не даст его собрать.

При стратегии `SEQUENCE` выдача идентификаторов **разделена** от самой вставки. Провайдер может заранее «выдернуть» диапазон значений из последовательности и потом свободно накапливать INSERT-ы в батчи, отправляя их пачками 50/100 и т. п. В результате round-trip’ов меньше, перегрузка пула снижается, а пропускная способность растёт в разы. Поэтому если у вас есть требования к массовым вставкам — выбирайте `SEQUENCE`.

Дальше вступают в игру **оптимизаторы** последовательностей. `pooled` и `pooled-lo` позволяют брать **блок** значений из последовательности, чтобы реже ходить в БД за `nextval`. Схематично: приложение получает «корзину» из N id и раздаёт их локально. Это важная оптимизация под высокую нагрузку, особенно в микросервисах с несколькими инстансами.

Разница между `pooled` и `pooled-lo`. В `pooled` сам хранимый в БД счётчик увеличивается на 1, а Hibernate держит «окно» из `allocationSize`, вычисляя hi/lo внутри. В `pooled-lo` БД-последовательность **реально прыгает** на шаг `allocationSize` (INCREMENT BY N), а Hibernate на стороне приложения раздаёт значения «внутри» блока. В современных проектах чаще используют `pooled-lo`, потому что он проще согласуется между несколькими узлами и лучше восстанавливается после рестартов.

Чтобы `pooled-lo` работал эффективно, **увеличьте инкремент последовательности** в схеме ровно до `allocationSize` и укажите тот же `allocationSize` в аннотации генератора. Это синхронизирует БД и ORM. Если оставить по умолчанию (JPA `allocationSize=50`, а в БД `INCREMENT BY 1`), всё будет корректно, но вы потеряете часть выгоды, делая лишние вызовы `nextval()`.

При работе с SEQUENCE обратите внимание на кэшируемость последовательностей в СУБД. В Postgres последовательность сама по себе быстрая, но под большой нагрузкой упрётесь именно в количество round-trip’ов. Здесь `pooled/pooled-lo` и высокая `allocationSize` (например, 100/200) дают хороший выигрыш, если вы много вставляете.

Что если у вас уже стоит `IDENTITY` из «Основ»? Для массовых вставок можно завести **отдельную сущность с SEQUENCE** и преобразовывать данные после ETL. Но чаще лучше **мигрировать** стратегию: добавить последовательность, перенастроить колонку и генератор. Да, id могут «скакнуть» при смене стратегии, но для непрозрачного внешнего мира это обычно не важно.

И ещё: Spring Data `saveAll()` не делает магии — он просто вызывает `persist`/`merge` по очереди. Работать в батче он начнёт **только** при корректной стратегии id и включённых настройках Hibernate/драйвера. Проверяйте SQL-лог: вы должны увидеть «пачки» INSERT-ов, а не сотни одиночных запросов.

**Flyway (PostgreSQL) — последовательность под pooled-lo**

```sql
-- V1__seq_invoice.sql
create sequence if not exists invoice_seq increment by 50 start with 1; -- allocationSize=50

create table invoices(
    id bigint primary key,
    customer_id bigint not null,
    amount_cents bigint not null,
    created_at timestamptz not null default now()
);
```

**Java — сущность с SEQUENCE + pooled-lo и сущность с IDENTITY (для контраста)**

```java
package com.example.batchid;

import jakarta.persistence.*;
import org.hibernate.annotations.GenericGenerator;

@Entity
@Table(name = "invoices")
@SequenceGenerator(
        name = "invoice_seq",
        sequenceName = "invoice_seq",
        allocationSize = 50  // согласовано с миграцией
)
public class Invoice {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "invoice_seq")
    private Long id;

    @Column(name = "customer_id", nullable = false)
    private Long customerId;

    @Column(name = "amount_cents", nullable = false)
    private long amountCents;

    protected Invoice() {}
    public Invoice(Long customerId, long amountCents) {
        this.customerId = customerId;
        this.amountCents = amountCents;
    }
    // getters/setters ...
}

@Entity
@Table(name = "tags")
class Tag {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // НЕ батчится как вставка в Hibernate
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    protected Tag() {}
    public Tag(String name){ this.name = name; }
    // getters/setters ...
}
```

**Kotlin — те же сущности**

```kotlin
package com.example.batchid

import jakarta.persistence.*

@Entity
@Table(name = "invoices")
@SequenceGenerator(
    name = "invoice_seq",
    sequenceName = "invoice_seq",
    allocationSize = 50
)
class Invoice(
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "invoice_seq")
    var id: Long? = null,

    @Column(name = "customer_id", nullable = false)
    var customerId: Long = 0,

    @Column(name = "amount_cents", nullable = false)
    var amountCents: Long = 0
)

@Entity
@Table(name = "tags")
class Tag(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true)
    var name: String = ""
)
```

**Java — сервис массовой вставки (батч виден только с SEQUENCE)**

```java
package com.example.batchid;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class InvoiceBatchService {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void saveInvoices(List<Invoice> invoices) {
        int batch = 50; // совпадает с jdbc.batch_size
        int i = 0;
        for (Invoice inv : invoices) {
            em.persist(inv);
            if (++i % batch == 0) {
                em.flush();
                em.clear(); // снижает рост L1-кэша
            }
        }
    }
}
```

**Kotlin — аналог сервиса**

```kotlin
package com.example.batchid

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class InvoiceBatchService(
    @PersistenceContext private val em: EntityManager
) {
    @Transactional
    fun saveInvoices(invoices: List<Invoice>) {
        val batch = 50
        var i = 0
        for (inv in invoices) {
            em.persist(inv)
            if (++i % batch == 0) {
                em.flush()
                em.clear()
            }
        }
    }
}
```

---

## Тюнинг батчей: `hibernate.jdbc.batch_size`, `order_inserts/updates`, драйверные оптимизации (для PG — `reWriteBatchedInserts`)

Начинаем с базового параметра `hibernate.jdbc.batch_size`. Он определяет, сколько однотипных DML-операций (INSERT/UPDATE/DELETE) Hibernate **буферизует** перед отправкой в JDBC как батч. Типичные значения — 25–100. Слишком маленькое — мало эффекта; слишком большое — рост памяти и риск попасть в большие RTT/таймаут. На практике 50 — хороший старт.

Опции `hibernate.order_inserts=true` и `hibernate.order_updates=true` позволяют ORM **перегруппировать** операции по типу сущности и ключам, чтобы максимизировать батчи. Без сортировки вы легко получите чередование разных типов и «разобьёте» батч на одиночные запросы. С сортировкой ORM сначала вставит все `Invoice`, затем все `Payment` и т. п., что уменьшит количество статементов.

Для **версионируемых** сущностей (с `@Version`) Hibernate по умолчанию осторожничает и может не батчить UPDATE-ы. Параметр `hibernate.jdbc.batch_versioned_data=true` разрешает это, если вы понимаете свои шаблоны конкурентности и готовы ловить конфликты оптимистической блокировки — в обмен вы получите существенно меньше round-trip’ов на массовых изменениях.

Драйверные оптимизации. В Postgres ключевой флаг — `reWriteBatchedInserts=true` в JDBC URL. Он позволяет драйверу **переписать** серию одинаковых `INSERT INTO t(a,b) values (?, ?)` в один multi-values `INSERT ... VALUES (...), (...), ...`, что уменьшает парсинг и ускоряет запись. Это особенно ощутимо при сотнях/тысячах строк. Для UPDATE/DELETE этот флаг не помогает — но там и так работает JDBC батч.

Не забывайте, что batched вставки работают корректно только при стратегии SEQUENCE (см. предыдущий пункт). С `IDENTITY` Hibernate вынужден выполнять INSERT немедленно — драйверу просто нечего переписывать «в пачку» от Hibernate. Исключения из этого правила бывают в кастомных сценариях, но рассчитывать на них нельзя.

Важно контролировать **память**. Батчи «висят» в L1-кэше до `flush()`. Если вы вставляете десятки тысяч строк, обязательно делайте периодический `flush/clear`. Это не только освобождает память, но и сбрасывает JDBC-буфер, давая БД начать фактическую работу. Наблюдайте за GC: батчирование может кратковременно поднять давление на heap.

Следите за **порядком колонок** в INSERT. Hibernate генерирует SQL с перечнем полей; если вы вручную меняете их в `@Column`/`@DynamicInsert`, не нарушайте согласованность — иначе драйвер не сможет эффективно переписать батч. Обычно это редкость, но при тонкой тюнинговой работе нюанс выстреливает.

Проверяйте итоговый SQL и статистику. Включите `org.hibernate.SQL` и `hibernate.generate_statistics=true`. В логах должны появляться повторяющиеся INSERT-ы группами, а в статистике — малое количество `prepareStatement`/`closeStatement`. Если видите сотни одиночных — значит, где-то мешает стратегия id или порядок операций.

И наконец, у тюнинга есть потолок. Если вам нужно заливать миллионы строк (ETL/миграции), подумайте про bulk-загрузки на уровне СУБД (COPY в Postgres), `StatelessSession` в Hibernate или уход в JDBC/jOOQ. Классический ORM предназначен для бизнес-транзакций, а не для «шлангов» данных.

**Java — сравнение настроек батча на одной операции**

```java
package com.example.batchtune;

import com.example.batchid.Invoice;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.List;

@Service
public class TuningDemoService {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void insertThousandInvoices() {
        List<Invoice> list = new ArrayList<>();
        for (int i = 0; i < 1_000; i++) {
            list.add(new Invoice(100L + (i % 10), 10_00L + i));
        }
        int i = 0;
        for (Invoice inv : list) {
            em.persist(inv);
            if (++i % 50 == 0) { // совпадает с jdbc.batch_size
                em.flush();
                em.clear();
            }
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.batchtune

import com.example.batchid.Invoice
import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class TuningDemoService(
    @PersistenceContext private val em: EntityManager
) {
    @Transactional
    fun insertThousandInvoices() {
        val list = MutableList(1_000) { i ->
            Invoice(customerId = 100L + (i % 10), amountCents = 1_000L + i)
        }
        var i = 0
        for (inv in list) {
            em.persist(inv)
            if (++i % 50 == 0) {
                em.flush()
                em.clear()
            }
        }
    }
}
```

---

## Массовые операции: `bulk update/delete` через JPQL — обход 1-го уровня, необходимость ручной синхронизации

`Bulk update/delete` — это JPQL-команды вида `update Entity e set e.status = :s where ...` и `delete from Entity e where ...`, которые Hibernate транслирует в **один SQL** без загрузки отдельных сущностей и без dirty checking. Это инструмент высокой мощности: вы за одно действие меняете/удаляете тысячи строк. Но плата — они **обходят** Persistence Context (кэш 1-го уровня).

Что означает «обход»? Если в текущей транзакции у вас уже есть managed-экземпляры этой сущности, их поля **не обновятся** в памяти после bulk-операции. Вы можете тут же прочитать «старые» данные, а в БД уже «новые». Чтобы избежать рассинхрона, после bulk-операций нужно **очищать** контекст (`clear`) или хотя бы `refresh` затронутые сущности. В Spring Data JPA есть удобные флаги `@Modifying(clearAutomatically = true, flushAutomatically = true)`.

Важно помнить и про **версионирование**. JPQL bulk не учитывает `@Version` автоматически. Если для вас критично защищаться от конкуренции, вручную добавьте в запрос `where e.version = :expected` и увеличьте `e.version = e.version + 1` в SET. И обязательно проверяйте возвращаемый счётчик строк: 0 — значит, конфликт версий/условий, реагируйте 409/404.

Массовые операции — тяжёлые для БД: они держат блокировки дольше обычного апдейта одной строки, могут затронуть много страниц данных и переписать индексы. Поэтому закладывайте **таймауты** и делайте операции **порционно** (по ключевым диапазонам/статусам), если счёт идёт на миллионы. Часто выгоднее делать несколько маленьких апдейтов, чем один гигантский, особенно если на поле нет подходящего индекса.

Интеграция с кэшем 2-го уровня и кешом запросов: после bulk-операции их содержимое устаревает. Либо запретите L2 для активно меняемой сущности, либо сразу инвалидируйте регион (`EntityManagerFactory.getCache().evict(Entity.class)`), либо используйте поставщик, умеющий реагировать на DML-сигналы. Кеш запросов (query cache) в таких системах вообще лучше не включать.

Наблюдайте за планами: `bulk update` без селективности (без индексов в `where`) превращается в full-scan и массовую перезапись. Если у вас часто «закрываются» тысячи тикетов по статусу, добавьте индекс на `status`, и операция станет в разы дешевле. Иногда имеет смысл разбить: «отметить id в вспомогательной таблице» → «обновить по join» — так вы избежите долгих блокировок горячих таблиц.

Ещё одна типичная ловушка — делать bulk в том же методе, где вы до/после читаете эти же сущности. Даже с `clearAutomatically` есть риск логической ошибки: вы предполагаете, что «статус обновился», а следом читаете коллекцию, которая была загружена раньше и ещё в памяти. Надёжнее разделять «команды» и «запросы» по разным транзакциям/методам.

И помните, что bulk-операции в JPA **не вызывают** `@PreUpdate/@PostUpdate` и другие entity-listeners. Если у вас есть аудит/журналы, реализуйте их отдельно (триггеры БД, аудит-таблицы) или обновляйте сущности поштучно (дороже) там, где слушатели критичны.

Если вы опираетесь на «ретраи» (повтор апдейта при конфликте — характерно для оптимистической блокировки), оборачивайте bulk-команду в цикл с ограниченным количеством попыток. Но чаще проще повторить операцию на уровне бизнес-команды («закрыть тикет»), чем на уровне общей массовой команды.

**Java — репозиторий с bulk update/delete и корректной синхронизацией**

```java
package com.example.bulk;

import jakarta.persistence.EntityManagerFactory;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Repository
public interface TicketBulkRepository extends JpaRepository<Ticket, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
        update Ticket t
           set t.status = :toStatus,
               t.updatedAt = CURRENT_TIMESTAMP,
               t.version = t.version + 1
         where t.status = :fromStatus
    """)
    int transition(@Param("fromStatus") String fromStatus,
                   @Param("toStatus") String toStatus);

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("delete from Ticket t where t.status = :status")
    int deleteByStatus(@Param("status") String status);
}

@Service
class TicketBulkService {
    private final TicketBulkRepository repo;
    private final EntityManagerFactory emf;
    TicketBulkService(TicketBulkRepository repo, EntityManagerFactory emf) {
        this.repo = repo; this.emf = emf;
    }

    @Transactional
    public int closeAllNew() {
        int changed = repo.transition("NEW", "CLOSED");
        // L2-кэш, если включён:
        emf.getCache().evict(Ticket.class);
        return changed;
    }

    @Transactional
    public int purgeClosed() {
        int deleted = repo.deleteByStatus("CLOSED");
        emf.getCache().evict(Ticket.class);
        return deleted;
    }
}
```

**Kotlin — тот же bulk с очисткой кэшей**

```kotlin
package com.example.bulk

import jakarta.persistence.EntityManagerFactory
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Repository
interface TicketBulkRepository : JpaRepository<Ticket, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        """
        update Ticket t
           set t.status = :toStatus,
               t.updatedAt = CURRENT_TIMESTAMP,
               t.version = t.version + 1
         where t.status = :fromStatus
        """
    )
    fun transition(
        @Param("fromStatus") fromStatus: String,
        @Param("toStatus") toStatus: String
    ): Int

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("delete from Ticket t where t.status = :status")
    fun deleteByStatus(@Param("status") status: String): Int
}

@Service
class TicketBulkService(
    private val repo: TicketBulkRepository,
    private val emf: EntityManagerFactory
) {
    @Transactional
    fun closeAllNew(): Int {
        val changed = repo.transition("NEW", "CLOSED")
        emf.cache.evict(Ticket::class.java)
        return changed
    }

    @Transactional
    fun purgeClosed(): Int {
        val deleted = repo.deleteByStatus("CLOSED")
        emf.cache.evict(Ticket::class.java)
        return deleted
    }
}
```

**Подсказки эксплуатации (резюме):**

1. Для массовых **вставок** используйте `SEQUENCE + pooled-lo + batch_size + reWriteBatchedInserts`.
2. Для массовых **обновлений/удалений** — `@Modifying` + `clearAutomatically/flushAutomatically` + контроль счётчиков.
3. Следите за L1/L2 и таймаутами, делайте операции порциями и покрывайте их интеграционными тестами на реальной СУБД.

# 4. Транзакции, изоляция и блокировки

*Зависимости (универсально для примеров этой подтемы).*

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

## Границы и изоляция: поведение `readOnly=true` как хинт, влияние `FlushMode` (COMMIT/MANUAL)

`@Transactional(readOnly = true)` — это не «магический щит», а **набор подсказок** (hints) инфраструктуре. На уровне Spring он помечает транзакцию как «только чтение» и передаёт этот флаг в провайдер JPA и драйвер JDBC. На уровне Hibernate это обычно переключает **режим сброса** (flush) так, чтобы провайдер **не пытался** синхронизировать изменения сущностей с БД в конце транзакции. На уровне драйвера устанавливается `Connection.setReadOnly(true)`, что в некоторых СУБД действительно запрещает DML, а в других лишь оптимизирует планы.

Важно понимать, что `readOnly=true` — **операционный контракт** внутри команды. Мы обещаем «в этом методе не писать», а инфраструктура помогает это обещание выполнить быстрее и безопаснее. Но это не защита уровня безопасности: если вы умышленно вызовете нативный `UPDATE` внутри такой транзакции, часть СУБД всё равно его выполнит. Руководствуйтесь правилом: чтение — только чтение, и никаких «случайных» записей.

Теперь о `FlushMode`. В спецификации JPA есть два значения: `AUTO` и `COMMIT`. В `AUTO` провайдер может выполнять `flush` перед выполнением JPQL/Criteria-запроса, чтобы обеспечить консистентность (вы увидите собственные изменения в том же юните работы). В `COMMIT` синхронизация откладывается до конца транзакции. У Hibernate есть также собственный `MANUAL`: ORM **вообще** не делает `flush` автоматически. В Spring при `@Transactional(readOnly = true)` для Hibernate обычно активируется эквивалент `MANUAL`, что снижает накладные расходы на dirty checking.

Эти режимы имеют прямые последствия для производительности. Если вы выполняете «тяжёлое» чтение (потоковый экспорт, отчёт), то лишний `flush` перед каждым `select` — пустая работа (никаких записей всё равно нет). Поэтому `readOnly=true` + `COMMIT/MANUAL` экономят CPU и уменьшают количество служебных сопоставлений. При этом в обычных командных операциях (`create/update`) оставляйте `AUTO`: так JPA гарантирует видимость ваших изменений в пределах транзакции.

Есть нюанс: `FlushModeType.COMMIT` — режим JPA, а `FlushMode.MANUAL` — расширение Hibernate. Если вы пишете код на «чистом» JPA API, меняйте режим через `entityManager.setFlushMode(FlushModeType.COMMIT)`. Если вы уверены, что под капотом именно Hibernate, можно «уровнем ниже» получить `Session` и включить `MANUAL` — это даёт максимальную экономию на больших чтениях.

Поведение драйвера `readOnly` тоже неоднородно. Для PostgreSQL установка read-only транзакции позволяет оптимизировать планировщик и **запретит** DML в явной «read-only» транзакции, но JDBC-флаг не всегда конвертируется в `SET TRANSACTION READ ONLY` автоматически — это зависит от реализации. Рассчитывать на него как на «брейк» не стоит; относитесь к нему как к оптимизационному сигналу (и хорошей самодисциплине).

Типичный анти-паттерн — аннотировать весь сервис `@Transactional(readOnly = true)` «ради ускорения», а потом добавлять «маленький апдейт». Так вы получаете неожиданные ошибки или, что хуже, неоптимальное поведение. Правильнее разделять чтения/записи по методам, а «толстые» чтения выносить в отдельный `QueryService`.

Не забывайте, что `readOnly` отключает не только `flush`, но и **снятие версий**/dirty checking: Hibernate просто не будет трекать изменения. Если вы всё-таки меняете сущности в таком методе (что уже нарушение контракта), изменения, скорее всего, не попадут в БД. Это хорошо: мы рано узнаём о нарушении инвариантов по тестам.

Ещё один практический приём — отмечать сами запросы как «только чтение». В Hibernate есть хинт `org.hibernate.readOnly` для `Query`: он помечает возвращаемые сущности как **read-only**, исключая их из dirty checking. Это особенно полезно для длинных потоковых выборок, уменьшая давление на L1-кэш.

Для тестовой валидации режима отслеживайте статистику Hibernate и смотрите, сколько раз сработал `flush`. В `readOnly`-методе счётчик должен быть нулевой; если нет — скорее всего, где-то в коде есть запись или вы переопределили `FlushMode`.

Резюме: для чтений — `@Transactional(readOnly=true)` + `COMMIT/MANUAL` + при необходимости хинты на запрос; для команд — дефолтный `AUTO`. Чётко разделяйте «команды» и «запросы» (CQS), и транзакционные границы будут прозрачнее, а производительность — предсказуемее.

**Java — чтение с ручным управлением flush и read-only хинтами**

```java
package com.example.tx;

import jakarta.persistence.*;
import org.hibernate.Session;
import org.hibernate.FlushMode;
import org.hibernate.jpa.HibernateHints;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Entity
@Table(name = "products")
class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String name;
    @Column(nullable = false) Long priceCents;
    protected Product() {}
    Product(String name, long price){ this.name = name; this.priceCents = price; }
}

@Service
class ProductReadService {

    @PersistenceContext
    private EntityManager em;

    @Transactional(readOnly = true)
    public List<Product> topExpensive(int limit) {
        // Переключим JPA flush на COMMIT (на всякий случай)
        em.setFlushMode(FlushModeType.COMMIT);

        // Для Hibernate можно жёстко поставить MANUAL и read-only
        Session session = em.unwrap(Session.class);
        session.setHibernateFlushMode(FlushMode.MANUAL);

        TypedQuery<Product> q = em.createQuery(
            "select p from Product p order by p.priceCents desc", Product.class);
        q.setMaxResults(limit);
        q.setHint(HibernateHints.HINT_READ_ONLY, true); // сущности read-only, без dirty checking
        return q.getResultList();
    }
}
```

**Kotlin — аналог с хинтами и MANUAL**

```kotlin
package com.example.tx

import jakarta.persistence.*
import org.hibernate.FlushMode
import org.hibernate.Session
import org.hibernate.jpa.HibernateHints
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "products")
class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = "",
    @Column(nullable = false) var priceCents: Long = 0
)

@Service
class ProductReadService(
    @PersistenceContext private val em: EntityManager
) {

    @Transactional(readOnly = true)
    fun topExpensive(limit: Int): List<Product> {
        em.flushMode = FlushModeType.COMMIT
        val session = em.unwrap(Session::class.java)
        session.hibernateFlushMode = FlushMode.MANUAL

        val q = em.createQuery(
            "select p from Product p order by p.priceCents desc",
            Product::class.java
        )
        q.maxResults = limit
        q.setHint(HibernateHints.HINT_READ_ONLY, true)
        return q.resultList
    }
}
```

---

## Оптимистическая блокировка `@Version`: обработка конфликтов, повтор операций

Оптимистическая блокировка — это способ защититься от **тихих перезаписей** при конкурентных обновлениях одной записи. Идея проста: у сущности есть поле версии (`@Version`), которое изменяется при каждом успешном `UPDATE`. Когда две транзакции читают одну и ту же строку и пытаются её обновить, у второй `UPDATE` не пройдёт по условию версии — ORM бросит исключение. Мы не блокируем никого заранее, зато обнаруживаем конфликт на записи.

Поле `@Version` может быть `long`/`int`, `Instant` или `Timestamp`. Числовые версии инкрементируются, временные — сравниваются на равенство/неравенство. С точки зрения операционных свойств числовая версия предсказуемее и дешевле: никаких проблем с точностью и таймзонами. На старте лучше выбирать `long`.

Важно различать два источника исключений. JPA провайдер может бросить `OptimisticLockException`, а Spring переведёт его в `OptimisticLockingFailureException`. Обрабатывать всегда нужно «верхний» спринговый тип — он одинаков для JDBC/JPA/JOOQ при честной транзакционной абстракции и легче мокается в тестах.

Стратегия обработки конфликтов зависит от бизнес-кейса. В большинстве приложений вы либо сообщаете пользователю «запись изменилась, обновите форму», либо автоматически повторяете операцию (retry) несколько раз с пересчитыванием данных. Автоповторы особенно удобны в «командах без побочных эффектов» (например, «увеличить счётчик» или «сменить статус по правилам»).

Повторы должны быть **идемпотентными** и ограниченными по числу. Типичный паттерн: попытаться 3 раза с короткой паузой; если не получилось — отдать 409 (Conflict) или специфичную бизнес-ошибку. Важно не «слепо» повторять `merge`, а пересчитывать предметные поля на свежем состоянии (прочитал → изменил → записал).

Оптимистическая блокировка требует **коротких транзакций**. Чем дольше вы держите managed-объект в памяти, тем больше вероятность конфликта. Старайтесь не захватывать сущности «на час» во внешнем слое; закончите команду быстро, а для длительных процессов используйте сага-паттерны или очереди.

Обратите внимание на массовые апдейты. JPQL `update ...` **не учитывает** `@Version` автоматически. Если у вас активна оптимистическая блокировка для сущности, а вы делаете bulk update, добавьте ручное увеличение версии и условие `where version = :expected` — иначе вы обойдёте защиту и можете перетереть чужую работу.

Оптимистическая блокировка сочетается с DTO-паттерном «версия на клиенте». В ответах API отдавайте версию вместе с данными, а при `PUT/PATCH` требуйте её обратно. Так вы получаете «CAS-подобное» поведение: сервер принимает изменения только если клиент видел последнюю версию. Это особенно прозрачно при React/SPA.

С тестовой стороны проверяйте, что второй конкурентный апдейт действительно не проходит. Либо разнесите апдейты по двум транзакциям в одном тесте, либо используйте два потока с барьером. Искусственно увеличивайте задержки, чтобы воспроизвести гонку, и проверяйте, что сервис либо повторяет команду, либо бросает нужное исключение.

И наконец, помните, что оптимистическая блокировка **не защищает** от логических конфликтов уровня бизнес-инвариантов («сумма не должна уйти в минус»). Здесь поможет только проверка условий в транзакции и, возможно, пессимистическая блокировка строки, если правила действительно критичны.

**Java — `@Version`, сервис с ретраями на конфликте**

```java
package com.example.version;

import jakarta.persistence.*;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "wallets")
class Wallet {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @Column(nullable = false) Long balanceCents = 0L;

    @Version
    Long version;

    protected Wallet() {}
    Wallet(long balance){ this.balanceCents = balance; }

    void add(long delta) {
        long next = balanceCents + delta;
        if (next < 0) throw new IllegalArgumentException("Negative balance");
        balanceCents = next;
    }
}

interface WalletRepository extends org.springframework.data.jpa.repository.JpaRepository<Wallet, Long> {}

@Service
class WalletService {
    private final WalletRepository repo;
    WalletService(WalletRepository repo){ this.repo = repo; }

    @Transactional
    public void deposit(Long id, long delta) {
        int attempts = 0;
        while (true) {
            try {
                Wallet w = repo.findById(id).orElseThrow();
                w.add(delta);              // dirty checking
                // flush будет на commit
                return;
            } catch (OptimisticLockingFailureException e) {
                if (++attempts >= 3) throw e;
                // небольшой backoff; в демо просто крутим цикл
            }
        }
    }
}
```

**Kotlin — то же с ретраями**

```kotlin
package com.example.version

import jakarta.persistence.*
import org.springframework.dao.OptimisticLockingFailureException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "wallets")
class Wallet(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var balanceCents: Long = 0,
    @Version var version: Long? = null
) {
    fun add(delta: Long) {
        val next = balanceCents + delta
        require(next >= 0) { "Negative balance" }
        balanceCents = next
    }
}

interface WalletRepository : JpaRepository<Wallet, Long>

@Service
class WalletService(private val repo: WalletRepository) {

    @Transactional
    fun deposit(id: Long, delta: Long) {
        var attempts = 0
        while (true) {
            try {
                val w = repo.findById(id).orElseThrow()
                w.add(delta)
                return
            } catch (e: OptimisticLockingFailureException) {
                if (++attempts >= 3) throw e
            }
        }
    }
}
```

---

## Пессимистические блокировки: `LockModeType.PESSIMISTIC_READ/WRITE`, таймауты/эскалации, риск дедлоков

Пессимистические блокировки — это «жёсткое» средство: мы не допускаем конкурентных изменений, удерживая блокировки на уровне БД. В JPA они выражаются через `LockModeType`: `PESSIMISTIC_READ` (обычно shared lock), `PESSIMISTIC_WRITE` (exclusive lock), а также `PESSIMISTIC_FORCE_INCREMENT` (пишем версию, чтобы «пнуть» конкурентов). Использовать их стоит там, где цена логического конфликта неприемлемо высока: денежные переводы, уникальные слоты бронирования, единичные ресурсы.

Самый распространённый сценарий — загрузить запись «для изменения» и держать `FOR UPDATE` до конца транзакции. Это гарантирует эксклюзивный доступ, но увеличивает вероятность **взаимных блокировок** (deadlocks) при сложном порядке доступа. Поэтому важно дисциплинировать порядок: все операции берут сущности в одном и том же порядке ключей (например, сначала «меньший id»).

`PESSIMISTIC_READ` в разных СУБД ведёт себя по-разному: где-то это shared lock, блокирующий writers, где-то он транслируется в «обычный SELECT ... FOR SHARE». Его смысл — позволить конкурентное чтение, но не давать никому обновить строку до конца транзакции. Для экраников отчётов он мало полезен; чаще вам нужен либо обычный read, либо write-lock на короткое время перед апдейтом.

Таймауты — обязательная часть практики. В JPA есть хинт `jakarta.persistence.lock.timeout` (миллисекунды). Если не удалось взять блокировку за отведённое время, драйвер бросит исключение, и вы сможете решить: повторить, отдать 409/503 или пойти альтернативным путём. Никогда не оставляйте блокировки «ждать вечно» — это поведение под нагрузкой обречёт пул соединений.

Ещё один хинт — «no wait». В некоторых диалектах (например, Postgres через `FOR UPDATE NOWAIT/SKIP LOCKED`) вы можете сказать «не жди». Это отличный инструмент для конкурентных workers: выбираем пачку задач `for update skip locked` и обрабатываем, не мешая друг другу. В JPA это обычно делают нативным SQL, потому что стандартные lock-моды не описывают `skip locked`.

Будьте аккуратны с эскалациями: длительные транзакции и множество заблокированных строк могут привести к эскалациям на уровне страниц/таблиц (в некоторых СУБД) и к «залипанию» системы. Держите транзакции короткими, а блокировки — локальными и минимальными по времени.

Комбинация с оптимистической блокировкой (`@Version`) полезна: вы можете читать без блокировок, а перед записью — брать `PESSIMISTIC_WRITE` на строку и всё равно иметь версию как «сетку безопасности». Это повышает предсказуемость поведения, особенно если часть кода иногда пишет без пессимистической блокировки.

Тесты на блокировки пишите в два потока. Один берёт `PESSIMISTIC_WRITE` и «держит» некоторое время, второй пытается взять `PESSIMISTIC_WRITE` и должен упасть по таймауту. Это позволяет рано поймать конфигурационные ошибки (например, отсутствие таймаута или неверный перевод исключений).

И самое важное — **порядок доступа**. Если у вас есть сценарии «перевода денег» между двумя счетами, **всегда** берите блокировки в порядке id: сначала меньший, затем больший. Это драматически снижает вероятность дедлоков при конкуренции.

**Java — `@Lock` и таймаут, плюс нативный `skip locked` для очереди**

```java
package com.example.lock;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Entity
@Table(name = "jobs")
class Job {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String status; // NEW, RUNNING, DONE
    protected Job() {}
    Job(String s){ this.status = s; }
}

@Repository
interface JobRepository extends JpaRepository<Job, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("select j from Job j where j.id = :id")
    Optional<Job> lockById(@Param("id") Long id);

    // Очередь: берём пачку задач без ожидания конкурентов (Postgres)
    @Query(value = """
        select * from jobs
        where status = 'NEW'
        order by id
        for update skip locked
        limit :n
        """, nativeQuery = true)
    List<Job> pollBatch(@Param("n") int n);
}

@Service
class JobService {
    private final JobRepository repo;
    JobService(JobRepository repo){ this.repo = repo; }

    @Transactional
    public void runExclusive(Long id) {
        Job j = repo.lockById(id).orElseThrow();
        j.status = "RUNNING";
        // делаем работу...
        j.status = "DONE";
    }

    @Transactional
    public List<Job> claimNext(int n) {
        List<Job> jobs = repo.pollBatch(n);
        for (Job j : jobs) j.status = "RUNNING";
        return jobs;
    }
}
```

**Kotlin — то же самое**

```kotlin
package com.example.lock

import jakarta.persistence.*
import org.springframework.data.jpa.repository.*
import org.springframework.data.jpa.repository.Lock
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "jobs")
class Job(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var status: String = "NEW"
)

@Repository
interface JobRepository : JpaRepository<Job, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    @Query("select j from Job j where j.id = :id")
    fun lockById(@Param("id") id: Long): java.util.Optional<Job>

    @Query(
        value = """
        select * from jobs
        where status = 'NEW'
        order by id
        for update skip locked
        limit :n
        """,
        nativeQuery = true
    )
    fun pollBatch(@Param("n") n: Int): List<Job>
}

@Service
class JobService(private val repo: JobRepository) {

    @Transactional
    fun runExclusive(id: Long) {
        val j = repo.lockById(id).orElseThrow()
        j.status = "RUNNING"
        // работа...
        j.status = "DONE"
    }

    @Transactional
    fun claimNext(n: Int): List<Job> {
        val jobs = repo.pollBatch(n)
        jobs.forEach { it.status = "RUNNING" }
        return jobs
    }
}
```

---

## Консистентность чтения: repeatable-read vs snapshot семантика СУБД, где важны «снимки»

Изоляция транзакций определяет, **какие изменения других транзакций** вы видите, пока выполняется ваша. Классические уровни — `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`. Но за этими именами скрываются разные реализации. Например, в PostgreSQL `REPEATABLE_READ` — это **snapshot isolation**: транзакция видит «снимок» базы на момент начала и не видит чужие коммиты до завершения. В MySQL/InnoDB поведение похожее, но детали фанттомов отличаются.

`READ_COMMITTED` — дефолт для большинства OLTP систем: вы видите только подтверждённые данные, но разные запросы в одной транзакции могут видеть **разные** состояния (каждый — свой снимок на момент выполнения). Это нормально для коротких команд, но не подходит для отчётов и операций «прочитал → посчитал → записал», если критично «всё считать на одном снимке».

`REPEATABLE_READ` обеспечивает стабильность чтений: «прочитал раз — вижу то же самое до конца». В PostgreSQL вы также не увидите фантомов в привычном смысле, потому что снимок фиксируется. Это удобно для отчётов, экспорта, расчётов агрегатов: вы получите **консистентный** результат на момент старта транзакции, даже если параллельно идут записи.

`SERIALIZABLE` идёт дальше: база гарантирует такой результат, как если бы транзакции выполнялись по очереди. В PostgreSQL это достигается через детекцию конфликтов и возможные «serialization failures». Это дорогой режим: вы чаще будете ловить исключения и повторять транзакции. Используйте его, когда нужен строгий консистентный порядок (например, сложные взаимосвязанные суммы).

В Spring уровень изоляции можно указать на методе: `@Transactional(isolation = Isolation.REPEATABLE_READ)`. Это отправит хинт менеджеру транзакций; конкретная реализация (JDBC) попытается установить уровень на соединении. Убедитесь, что ваш пула соединений и СУБД разрешают такой переключатель; иногда проще иметь **стабильный дефолт** на уровне DataSource/БД и переопределять только редко.

Выбор изоляции — компромисс между консистентностью и **конкурентностью**. Чем выше изоляция, тем больше риск конфликтов и откатов. Если у вас много чтений и мало записей — `REPEATABLE_READ` на отчётных операциях — хороший выбор. Если наоборот — держитесь `READ_COMMITTED` и локальных блокировок только там, где нужно.

Не путайте изоляцию с блокировками. `REPEATABLE_READ` не «запирает» строки для других писателей; он лишь даёт вам снимок. Чтобы исключить логические гонки, комбинируйте уровень изоляции с пессимистическим `FOR UPDATE` в точках критических изменений.

Прослойка ORM добавляет ещё один слой — **кэш первого уровня**. Даже на `READ_COMMITTED` повторный `find()` в одной и той же транзакции вернёт объект из L1-кэша, а не читает заново. Это «локальная repeatable-read». Если вам нужно увидеть «свежее» состояние в той же транзакции, используйте `refresh()`.

В отчётных сервисах полезно делать явные «снимки» логически: фиксировать момент времени `asOf` и использовать его в запросах (например, фильтровать по `created_at <= :asOf`). Это даёт воспроизводимость результатов при повторении отчёта позже и позволяет масштабировать чтения без долгих транзакций.

И, наконец, тестируйте изоляцию. Пишите тесты в два потока: один коммитит изменения, другой внутри `REPEATABLE_READ` проверяет, что «снимок» стабилен; под `READ_COMMITTED` — что повторные запросы видят новые данные. Это даст уверенность, что ваша конфигурация действительно работает, как ожидается.

**Java — выбор изоляции и refresh для «свежего» чтения**

```java
package com.example.isolation;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "counters")
class Counter {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) Long value = 0L;
    protected Counter() {}
}

interface CounterRepository extends org.springframework.data.jpa.repository.JpaRepository<Counter, Long> {}

@Service
class CounterService {
    private final CounterRepository repo;
    @PersistenceContext private EntityManager em;

    CounterService(CounterRepository repo){ this.repo = repo; }

    // Консистентное чтение на одном «снимке»
    @Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
    public long readConsistent(Long id) {
        Counter c1 = repo.findById(id).orElseThrow();
        // ... здесь могут идти другие запросы ...
        Counter c2 = repo.findById(id).orElseThrow(); // тот же L1-экземпляр, repeatable внутри транзакции
        return c2.value;
    }

    // Принудительно увидеть свежие данные в текущей транзакции
    @Transactional
    public long readFresh(Long id) {
        Counter c = em.find(Counter.class, id);
        em.refresh(c); // перечитать из БД, игнорируя L1
        return c.value;
    }
}
```

**Kotlin — изоляция и refresh**

```kotlin
package com.example.isolation

import jakarta.persistence.*
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Isolation
import org.springframework.transaction.annotation.Transactional
import org.springframework.data.jpa.repository.JpaRepository

@Entity
@Table(name = "counters")
class Counter(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var value: Long = 0
)

interface CounterRepository : JpaRepository<Counter, Long>

@Service
class CounterService(
    private val repo: CounterRepository,
    @PersistenceContext private val em: EntityManager
) {

    @Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
    fun readConsistent(id: Long): Long {
        val c1 = repo.findById(id).orElseThrow()
        val c2 = repo.findById(id).orElseThrow()
        return c2.value
    }

    @Transactional
    fun readFresh(id: Long): Long {
        val c = em.find(Counter::class.java, id)
        em.refresh(c)
        return c.value
    }
}
```

**application.yml — пример глобального таймаута запросов (PostgreSQL)**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc.time_zone: UTC
  datasource:
    hikari:
      connection-timeout: 30000
      # В Postgres можно управлять statement_timeout на уровне сессии:
      data-source-properties:
        options: '-c statement_timeout=5000'
```

# 5. Управление Persistence Context

*Зависимости для примеров (самодостаточно).*

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

## Когда уместно `clear()/detach()`, «длинные» сессии и утечки памяти

Persistence Context (PC) — это «единица работы» JPA и **кэш первого уровня** (identity map): все загруженные и созданные в рамках одной транзакции сущности живут здесь, Hibernate следит за их изменениями (dirty checking) и синхронизирует с БД на `flush`. Это удобно, но память не бесконечна: чем больше вы держите managed-объектов, тем выше давление на heap и GC.

«Длинные» сессии — антипаттерн, когда одна транзакция «таскается» через множество шагов/страниц/сообщений, а PC копит сотни тысяч объектов. Часто так происходит в батч-процессинге: читаем миллион строк, преобразуем и сохраняем. Без периодического «сбрасывания» PC вы быстро упрётесь в память, а `flush` начнёт занимать секунды.

`EntityManager.clear()` очищает **весь** PC: все managed-экземпляры становятся detached, dirty-пометки забыты. Это основной инструмент циклических ETL: «пакет (N записей) → `flush()` → `clear()` → следующий пакет». Такой ритм обеспечивает стабильное потребление памяти и предсказуемые паузы.

`EntityManager.detach(entity)` — хирургический вариант: убрать **конкретный** объект из PC, оставив остальных. Это полезно, когда вы маппите сущность в DTO, отдали наружу и не хотите, чтобы дальнейшие изменения по ошибке затронули ту же ссылку, либо когда читаете огромный поток сущностей, обрабатываете и не хотите держать их в PC до конца транзакции.

Важно понимать, что после `detach` JPA перестаёт отслеживать объект; любые изменения в нём **не попадут** в БД без `merge`. Поэтому `detach` уместен там, где вы точно завершили работу с объектом (например, записали в файл), либо где вы сознательно хотите разорвать связь с PC.

Ещё одна причина применять `clear/detach` — **утечки ссылок из PC** в долгоживущие структуры (кеши, статические поля, синглтоны). Managed-объекты часто содержат ссылки на другие сущности/коллекции (графы), и удержание даже одной сущности может удерживать «целый город». Лучший приём — на границе слоя конвертировать в **плоские DTO** и `detach` исходник (или чистить весь PC).

Open-Session-In-View (`spring.jpa.open-in-view`) соблазняет «удобством» — ленивые ассоциации будут доступными в веб-слое. Но это продлевает жизнь PC до конца web-запроса, увеличивает риск N+1 и «длинных» транзакций. Для публичных API **отключайте** OSIV и управляйте загрузкой данных осознанно в сервисном слое.

В батчах `flush()` обязателен: он синхронизирует накопленный dirty state с БД. Делать только `clear()` без `flush()` — значит «забыть» изменения. Типичный рисунок: каждые 50–100 операций — `flush()` и затем `clear()`. Размер порции подбирайте экспериментально по профилю памяти и времени отклика БД.

`clear()` также помогает избежать «ложных» повторных SELECT: когда вы читаете ту же сущность повторно в рамках **другого** шага, PC может возвращать «старую» версию. Если вы хотите точно читать «свежее» состояние при длинном процессе — либо делите процесс на транзакции, либо периодически очищайте PC, либо используйте `refresh`.

Не забывайте, что `clear()` обнуляет не только сущности, но и контекст каскадов. Если вы держали bidirectional-граф и полагались на автоматическую синхронизацию, после `clear` это перестанет работать — манипулируйте графом **до** очистки.

Наконец, проверяйте себя тестами с профилированием (например, Java Flight Recorder). Запустите ETL с 100k записей, добавьте `flush/clear`, сравните память и латентность. Правильный ритм почти всегда заметно улучшает профиль GC и стабильность.

**Java — батч-обработка с `flush()/clear()` и точечным `detach()`**

```java
package com.example.pc;

import jakarta.persistence.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Entity
@Table(name = "events")
class Event {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String type;
    @Column(nullable = false) Instant createdAt = Instant.now();
    protected Event() {}
    Event(String type){ this.type = type; }
}

interface EventRepository extends org.springframework.data.jpa.repository.JpaRepository<Event, Long> {
    @org.springframework.data.jpa.repository.Query("""
        select e from Event e where e.createdAt < :before order by e.id
    """)
    List<Event> findChunk(@org.springframework.data.repository.query.Param("before") Instant before,
                          org.springframework.data.domain.Pageable pageable);
}

@Service
class EventExportService {
    @PersistenceContext private EntityManager em;
    private final EventRepository repo;
    EventExportService(EventRepository repo){ this.repo = repo; }

    @Transactional
    public void exportOld(Instant before) {
        int page = 0;
        while (true) {
            var chunk = repo.findChunk(before, org.springframework.data.domain.PageRequest.of(page, 200));
            if (chunk.isEmpty()) break;
            for (Event e : chunk) {
                // ... маппим в CSV/DTO ...
                em.detach(e); // не держим в PC после использования
            }
            em.flush(); // если что-то сохраняли по пути
            em.clear(); // гарантированно очищаем графы
            page++;
        }
    }
}
```

**Kotlin — тот же паттерн**

```kotlin
package com.example.pc

import jakarta.persistence.*
import org.springframework.data.domain.PageRequest
import org.springframework.data.jpa.repository.Query
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

@Entity
@Table(name = "events")
class Event(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var type: String = "",
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

interface EventRepository : JpaRepository<Event, Long> {
    @Query("select e from Event e where e.createdAt < :before order by e.id")
    fun findChunk(@Param("before") before: Instant, pageable: org.springframework.data.domain.Pageable): List<Event>
}

@Service
class EventExportService(
    private val repo: EventRepository,
    @PersistenceContext private val em: EntityManager
) {
    @Transactional
    fun exportOld(before: Instant) {
        var page = 0
        while (true) {
            val chunk = repo.findChunk(before, PageRequest.of(page, 200))
            if (chunk.isEmpty()) break
            chunk.forEach { e ->
                // ... пишем в файл/канал ...
                em.detach(e)
            }
            em.flush()
            em.clear()
            page++
        }
    }
}
```

---

## Read-only оптимизации: `Query.setReadOnly(true)`, «immutable»-сущности, `StatelessSession` (узкие случаи)

Read-only — это про экономию: если вы **точно** не будете изменять загруженные сущности, не нужно тратить CPU на dirty checking и bookkeeping. В Hibernate для запроса можно указать хинт `HINT_READ_ONLY`, который пометит возвращённые сущности как «только чтение»; провайдер не будет отслеживать их изменения и будет экономить на снимках состояний.

Помимо хинта на уровне запроса, есть настройка уровня сессии: `Session.setDefaultReadOnly(true)` — все загруженные далее сущности считаются read-only. Это полезно для больших отчётных операций, где вы выполняете несколько запросов подряд и уверены, что записи не модифицируются.

Для неизменяемых справочников хорошо подходит **аннотация Hibernate** `@org.hibernate.annotations.Immutable` на сущности. Такая сущность рассматривается как «read-only» всегда; попытка изменить её поля игнорируется с точки зрения dirty checking (но конечно, вы можете сделать `native update` вручную — ORM тут не спасёт). Это снижает накладные расходы и защищает от случайных обновлений справочников.

`StatelessSession` — это «облегчённая» сессия Hibernate без PC и L1-кэша. Она не делает dirty checking, не поддерживает ассоциации и каскады, работает ближе к JDBC. Чтение через неё минимально нагружает память, а массовые вставки/обновления идут без «обрастания» PC. Но вы теряете все удобства ORM: нет «магии» графов, нет автоматической генерации id (кроме прямых значений/sequence), нет listeners.

Когда уместен `StatelessSession`? В узких случаях: огромные однотипные выборки на выгрузку, массовые вставки/апдейты без загрузки существующих сущностей, миграции/ETL. В «обычной» бизнес-логике он избыточен и опасен: легко нарушить инварианты, забыть про ключи или обработку связей.

Read-only режим стоит сочетать с отключением OSIV и `@Transactional(readOnly = true)`. В таком профиле Hibernate часто переключает `FlushMode` на MANUAL, и вы получаете минимальную цену за чтение. Хорошо также добавить хинт `setReadOnly(true)` на каждый `Query` — это документирует намерение и защищает от случайных модификаций.

В отчётах и потоковой выгрузке добавляйте `setFetchSize`/`setHint(HibernateHints.HINT_FETCH_SIZE, ...)` и используйте forward-only курсоры. Так вы не материализуете весь результат в память. В Postgres это особенно полезно, если у вас длинные транзакции: сервер отдаёт строки порциями.

Не забывайте о сериализации. Если вы всё равно будете превращать результат в DTO/CSV/JSON — подумайте, нужен ли вам вообще ORM на этом пути. Часто `JdbcTemplate`/jOOQ даёт ещё более тонкий контроль и меньшие накладные расходы. Read-only JPA — компромисс, когда у вас уже есть маппинг сущностей и нужно быстро собрать отчёт.

И ещё: read-only «означает договор». Не меняйте поля сущностей, помеченных как read-only, даже «в шутку». Такие правки не сохранятся, а в тестах вы легко перепутаете, почему значение «не доехало».

**Java — read-only запрос, `@Immutable` и `StatelessSession` чтение**

```java
package com.example.ro;

import jakarta.persistence.*;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.StatelessSession;
import org.hibernate.Transaction;
import org.hibernate.jpa.HibernateHints;
import org.hibernate.annotations.Immutable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Entity
@Table(name = "countries")
@Immutable // справочник: никогда не меняем
class Country {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, unique = true) String code;
    @Column(nullable = false) String name;
    protected Country() {}
    Country(String code, String name){ this.code=code; this.name=name; }
}

interface CountryRepository extends org.springframework.data.jpa.repository.JpaRepository<Country, Long> { }

@Service
class ReadOnlyService {
    @PersistenceContext private EntityManager em;
    private final SessionFactory sf;

    ReadOnlyService(EntityManagerFactory emf) {
        this.sf = emf.unwrap(SessionFactory.class);
    }

    @Transactional(readOnly = true)
    public List<Country> topCountries() {
        var q = em.createQuery("select c from Country c order by c.name", Country.class);
        q.setHint(HibernateHints.HINT_READ_ONLY, true); // сущности read-only
        return q.getResultList();
    }

    // Используем StatelessSession для очень большого чтения
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void exportHugeTable() {
        try (StatelessSession ss = sf.openStatelessSession()) {
            Transaction tx = ss.beginTransaction();
            var query = ss.createQuery("select c from Country c order by c.id", Country.class);
            query.setHint(HibernateHints.HINT_FETCH_SIZE, 500);
            for (Country c : query.list()) {
                // пишем c в поток/файл; PC отсутствует, память не растёт
            }
            tx.commit();
        }
    }
}
```

**Kotlin — те же приёмы**

```kotlin
package com.example.ro

import jakarta.persistence.*
import org.hibernate.SessionFactory
import org.hibernate.StatelessSession
import org.hibernate.Transaction
import org.hibernate.jpa.HibernateHints
import org.hibernate.annotations.Immutable
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Propagation
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "countries")
@Immutable
class Country(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true) var code: String = "",
    @Column(nullable = false) var name: String = ""
)

interface CountryRepository : org.springframework.data.jpa.repository.JpaRepository<Country, Long>

@Service
class ReadOnlyService(
    @PersistenceContext private val em: EntityManager,
    emf: EntityManagerFactory
) {
    private val sf: SessionFactory = emf.unwrap(SessionFactory::class.java)

    @Transactional(readOnly = true)
    fun topCountries(): List<Country> {
        val q = em.createQuery("select c from Country c order by c.name", Country::class.java)
        q.setHint(HibernateHints.HINT_READ_ONLY, true)
        return q.resultList
    }

    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    fun exportHugeTable() {
        sf.openStatelessSession().use { ss: StatelessSession ->
            val tx: Transaction = ss.beginTransaction()
            val query = ss.createQuery("select c from Country c order by c.id", Country::class.java)
            query.setHint(HibernateHints.HINT_FETCH_SIZE, 500)
            query.list().forEach { c ->
                // пишем c в поток
            }
            tx.commit()
        }
    }
}
```

---

## Кэш 1-го уровня как защита от двойных SELECT, но источник «устаревших» данных — критерии и баланс

Кэш первого уровня (PC) — ваш друг: он гарантирует **идентичность** экземпляров в рамках транзакции (identity map) и защищает от повторных SELECT того же объекта по id. Если вы дважды вызываете `find()` с тем же ключом, второй раз Hibernate вернёт объект из PC без похода в БД — экономия на лицe.

Однако у этой медали есть оборотная сторона: PC может стать источником **устаревших данных**, если состояние в БД изменилось «мимо» (native update, триггеры, другая транзакция) и вы продолжаете работать с кэшированным экземпляром. В этом случае поможет `em.refresh(entity)` — он перечитает запись из БД и обновит поля объекта.

Смешивание `native update` и чтения в одной транзакции — частая ловушка. Вы делаете `UPDATE ...` через нативный `Query`, а затем читаете сущность через `repo.findById` — получите старое значение из PC. Правильная практика: `em.flush()` перед native, а после — либо `em.clear()`, либо `em.refresh()` конкретной сущности. Идеально — разделяйте запись и чтение по разным методам/транзакциям.

Поведение JPQL-запросов относительно PC тоже важно: провайдер обязан возвращать **тот же управляемый экземпляр** для строк с id, которые уже в PC. Даже если он выполнит SELECT, результат будет «сшит» с PC. Поэтому иногда вы не увидите «новое» состояние, даже делая query, — снова выручат `refresh/clear`.

L1-кэш помогает и в другом: при сборке сложной карточки, если вы уже загружали `Customer` по id, все дальшеe ссылки на этого клиента из `Order` будут указывать на **тот же** объект из PC, без дополнительных SELECT. Это уменьшает вероятность N+1 в простых путях, хотя полностью проблему не решает (для коллекций всё сложнее).

С точки зрения производительности L1 — бесплатный, но он **растёт** с числом загруженных сущностей. В длинных потоках чтения его нужно «подстри¬гать» `clear()/detach()`. Иначе PC станет «кладбищем» ссылок и начнёт мешать сборщику мусора.

Не путайте L1 и кэш второго уровня. L1 — всегда включён, локален транзакции/сессии, потокобезопасен за счёт границ транзакции и никак не реплицируется. L2 — опционально, разделяется между сессиями и инстансами (в зависимости от провайдера), требует строгой стратегии инвалидации. Здесь мы говорим только про L1.

В тестах удобно смотреть **статистику Hibernate** (`generate_statistics`). Счётчик `prepareStatementCount` и раздел `entityFetchCount`/`secondLevelCacheHitCount` помогут подтвердить, что вторые чтения по id действительно не ходят в БД. Это мощный «охранник» от регресса N+1.

Памятка: «читать → обрабатывать → возможно записывать → `flush()` → при необходимости `clear()`». Если между шагами данные могут меняться другими транзакциями, не стесняйтесь делать `refresh()` выборочно — это дешево по сравнению с расследованием инцидентов.

И наконец, health-check для себя: если вы видите в логах повторные одинаковые SELECT по id внутри одной транзакции — это признак, что вы делаете `clear()` слишком часто или обращаетесь к данным из разных контекстов. Проверьте границы `@Transactional` и места, где вы вручную очистили PC.

**Java — демонстрация L1: двойное чтение, native-update и `refresh()`**

```java
package com.example.l1;

import jakarta.persistence.*;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "articles")
class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String title;
    protected Article() {}
    Article(String t){ this.title = t; }
}

interface ArticleRepository extends org.springframework.data.jpa.repository.JpaRepository<Article, Long> { }

@Service
class L1DemoService {
    private final ArticleRepository repo;
    @PersistenceContext private EntityManager em;
    private final SessionFactory sf;

    L1DemoService(ArticleRepository repo, EntityManagerFactory emf) {
        this.repo = repo;
        this.sf = emf.unwrap(SessionFactory.class);
    }

    @Transactional
    public void demo(Long id) {
        Statistics st = sf.getStatistics(); st.clear();

        Article a1 = repo.findById(id).orElseThrow(); // 1-й SELECT
        Article a2 = repo.findById(id).orElseThrow(); // из L1, без SELECT

        // Нативно изменим заголовок
        em.createNativeQuery("update articles set title = :t where id = :id")
          .setParameter("t", "New Title")
          .setParameter("id", id)
          .executeUpdate();

        // Всё ещё старый title в L1:
        System.out.println("Before refresh: " + a1.title);
        em.refresh(a1); // перечитываем из БД
        System.out.println("After  refresh: " + a1.title);

        System.out.println("SQL statements total: " + st.getPrepareStatementCount());
    }
}
```

**Kotlin — аналогичная демонстрация**

```kotlin
package com.example.l1

import jakarta.persistence.*
import org.hibernate.SessionFactory
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "articles")
class Article(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = ""
)

interface ArticleRepository : JpaRepository<Article, Long>

@Service
class L1DemoService(
    private val repo: ArticleRepository,
    @PersistenceContext private val em: EntityManager,
    emf: EntityManagerFactory
) {
    private val sf: SessionFactory = emf.unwrap(SessionFactory::class.java)

    @Transactional
    fun demo(id: Long) {
        val st = sf.statistics; st.clear()

        val a1 = repo.findById(id).orElseThrow() // первый SELECT
        val a2 = repo.findById(id).orElseThrow() // кэш 1-го уровня

        em.createNativeQuery("update articles set title = :t where id = :id")
            .setParameter("t", "New Title")
            .setParameter("id", id)
            .executeUpdate()

        println("Before refresh: ${a1.title}")
        em.refresh(a1)
        println("After  refresh: ${a1.title}")

        println("SQL statements total: ${st.prepareStatementCount}")
    }
}
```

# 6. Сложные маппинги и модель

*Зависимости и базовые настройки для всей подтемы (самодостаточно для компиляции примеров).*

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
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
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
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
```

---

## Наследование: SINGLE_TABLE vs JOINED vs TABLE_PER_CLASS — компромиссы производительности/нормализации

Наследование в JPA — инструмент для моделирования семейства сущностей с общим API и общими полями. На практике оно влияет не только на структуру кода, но и на физическую схему, скорость запросов, размер индексов и сложность миграций. Три стратегии — `SINGLE_TABLE`, `JOINED` и `TABLE_PER_CLASS` — представляют три разных компромисса между производительностью и нормализацией.

`SINGLE_TABLE` складывает всех наследников в **одну таблицу** с колонкой-дискриминатором (`DTYPE`). Это максимально быстро на чтение: полиморфный запрос — это один `SELECT` без джоинов и `UNION`. Но вы платите **разрежённостью**: поля, присутствующие только у части наследников, становятся `NULL`-колонками. При большом количестве таких полей таблица «распухает», индексы становятся тяжелее, а `UPDATE` затрагивает больше страниц.

`JOINED` хранит общие поля в базовой таблице, а специфичные — в таблицах-наследниках, соединяемых по PK=FK. Запросы к конкретному наследнику быстрые (один `JOIN`), а полиморфные читаются через серию `OUTER JOIN` или `UNION` (в зависимости от диалекта и JPQL). Это хорошо нормализовано, но даёт накладные расходы на джоины и требует аккуратных индексов по внешним ключам.

`TABLE_PER_CLASS` создаёт **по таблице на наследника** и выполняет полиморфные выборки через `UNION ALL`. Для сущности с десятком наследников полиморфный `SELECT` превращается в десяток под-запросов — дорого и тяжело для планировщика. Стратегия уместна редко: когда вы почти не делаете полиморфных запросов и вам нужна физическая изоляция данных наследников.

С точки зрения миграций у `SINGLE_TABLE` проще всего **добавлять** колонки, но сложнее выкидывать «мусор»: null-колонок становится много. У `JOINED` проще контролировать эволюцию отдельных веток, но при переносе полей между базой и наследником приходится переписывать данные. `TABLE_PER_CLASS` чаще всего упирается в сложность отчётных запросов и индексации.

Ещё один аспект — «горячие» индексы. В `SINGLE_TABLE` один большой индекс на `DTYPE`+«бизнес-поля» работает как универсальный, но может конфликтовать по структуре доступа у разных наследников. В `JOINED` индексы на дочерних таблицах компактнее и ориентированы на конкретные запросы; общий PK/DTYPE остаётся на базе. Это часто даёт выигрыш под OLTP-нагрузкой.

Правило большого пальца: если у вас немного наследников, а полиморфные запросы — частое явление (например, «все платежи разных типов, отсортированные по времени»), берите **`SINGLE_TABLE`**. Если наследники сильно различаются по набору колонок и полиморфные запросы редки, — **`JOINED`**. `TABLE_PER_CLASS` — только когда вы почти всегда обращаетесь к конкретным типам и хотите жёсткой физической сегрегации.

Избегайте перегиба: иногда наследование в модели — избыточно, и лучше использовать **композицию** (встроенные значения, стратегии) или «тип поля + JSON с параметрами» (для редких расширений). Наследование — мощное, но делает SQL и миграции менее очевидными для команды.

Проверяйте планы запросов и реальный SQL, который генерирует Hibernate. В `SINGLE_TABLE` не забывайте про `@DiscriminatorColumn` и явные `@DiscriminatorValue` — это улучшает читаемость SQL и защищает от сюрпризов при рефакторинге имён классов. В `JOINED` обязательно индексируйте FK-колонки дочерних таблиц, иначе любое чтение превратится в nested loop без индекса.

И наконец, имейте в виду, что наследование усложняет кэширование L2 и сериализацию. DTO-проекции по месту — лёгкий способ упростить контракт наружу и не тянуть за собой полиморфные графы в JSON.

**Java — пример `SINGLE_TABLE` для иерархии платежей**

```java
package com.example.inheritance;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "ptype", length = 16)
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false) private Long amountCents;
    @Column(nullable = false) private Instant createdAt = Instant.now();

    protected Payment() {}
    protected Payment(long amountCents){ this.amountCents = amountCents; }

    // getters
    public Long getId() { return id; }
    public Long getAmountCents() { return amountCents; }
    public Instant getCreatedAt() { return createdAt; }
}

@Entity
@DiscriminatorValue("CARD")
class CardPayment extends Payment {
    @Column(name = "card_last4", length = 4)
    private String cardLast4;

    protected CardPayment() {}
    public CardPayment(long amountCents, String cardLast4) {
        super(amountCents);
        this.cardLast4 = cardLast4;
    }
}

@Entity
@DiscriminatorValue("CASH")
class CashPayment extends Payment {
    @Column(name = "cashbox")
    private String cashbox;

    protected CashPayment() {}
    public CashPayment(long amountCents, String cashbox) {
        super(amountCents);
        this.cashbox = cashbox;
    }
}

interface PaymentRepository extends org.springframework.data.jpa.repository.JpaRepository<Payment, Long> {
    // полиморфный запрос — одна таблица
    java.util.List<Payment> findTop10ByOrderByCreatedAtDesc();
}
```

**Kotlin — тот же пример, но с `JOINED` для контраста**

```kotlin
package com.example.inheritance

import jakarta.persistence.*
import java.time.Instant

@Entity
@Table(name = "payments2")
@Inheritance(strategy = InheritanceType.JOINED)
open class Payment2(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    open var id: Long? = null,
    @Column(nullable = false) open var amountCents: Long = 0,
    @Column(nullable = false) open var createdAt: Instant = Instant.now()
)

@Entity
@Table(name = "card_payments2")
class CardPayment2(
    id: Long? = null,
    amountCents: Long = 0,
    createdAt: Instant = Instant.now(),
    @Column(name = "card_last4", length = 4) var cardLast4: String = ""
) : Payment2(id, amountCents, createdAt)

@Entity
@Table(name = "cash_payments2")
class CashPayment2(
    id: Long? = null,
    amountCents: Long = 0,
    createdAt: Instant = Instant.now(),
    @Column(name = "cashbox") var cashbox: String = ""
) : Payment2(id, amountCents, createdAt)

interface Payment2Repository : org.springframework.data.jpa.repository.JpaRepository<Payment2, Long>
```

---

## Связи: избегаем «грузных» `@ManyToMany` — предпочтительна явная сущность связи (join-entity)

`@ManyToMany` кажется заманчиво коротким: две коллекции — и Hibernate сам создаст промежуточную таблицу. Но в продакшн-сценариях это почти всегда оборачивается проблемами. Связь «многие-ко-многим» редко бывает «просто связью»: у связи появляются атрибуты (роль, дата, порядок, флаги), а операции над ней требуют тонкого контроля (soft-delete, аудит, уникальность). У JPA-коллекций `@ManyToMany` нет места для атрибутов — и вы быстро упираетесь в потолок.

Даже если атрибутов нет, «грязные» операции коллекций вызывают каскады `DELETE/INSERT` в таблицу связей при каждом изменении набора. Это дорого, сложно кешируется и плохо объяснимо по SQL. Любая попытка «подобрать» связи точечным `UPDATE` идёт вразрез с моделью `@ManyToMany`, потому что промежуточная таблица не явлена в коде.

Явная сущность связи (join-entity) решает обе проблемы. Мы моделируем связь как обычную сущность с PK (часто составным) и двумя `@ManyToOne` на «края», добавляем атрибуты и индексы, управляем жизненным циклом, как нам нужно. Это немного больше кода, но в разы больше прозрачности, контроля и производительности.

Отдельная боль — сериализация. Бидирекционная `@ManyToMany` легко превращается в рекурсивную JSON-структуру. С join-entity вы сериализуете плоские DTO, а края добавляете по необходимости. Это лучше для API и снимает риски `LazyInitializationException` при ленивых коллекциях.

Ещё один плюс join-entity — простые **уникальные ограничения**. В промежуточной таблице вы ставите `unique (a_id, b_id)` и уверены, что дубликаты не появятся. С `@ManyToMany` Hibernate может «насовывать» дубликаты в коллекцию в памяти, если вы неправильно определили `equals/hashCode`. С явной сущностью у вас есть полный контроль.

С точки зрения производительности join-entity помогает оптимизировать чтения. Вы можете сначала выбрать набор связей по критериям (и с пагинацией), затем догрузить «края» батчем — без гигантских `JOIN`. Это особенно полезно для «каталогов» и «подписок», когда связей очень много.

Для корректности всегда пишите **вспомогательные методы** на сторонах, чтобы поддерживать обе стороны в консистентности (`addEnrollment`, `removeEnrollment`). Это защищает от рассинхронизаций между памятью и БД и делает код читаемым в сервисах.

Наконец, миграция: начинать с `@ManyToMany` «на время» почти всегда плохо — любые реальные требования быстро заставят переписывать схему. Лучше сразу проектировать через join-entity, даже если сейчас атрибутов нет. Вы выиграете на поддержке и избежите сложных миграций.

И последнее — индексы. На таблице связи ставьте `unique(a_id, b_id)` и отдельные BTREE-индексы на `a_id` и `b_id` (если запросы асимметричны). Это даёт селективность и ускоряет джоины. Для мягкого удаления добавляйте составные индексы с `deleted_at is null`.

**Java — наивный `@ManyToMany` (для контраста, использовать с осторожностью)**

```java
package com.example.mtm;

import jakarta.persistence.*;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "students")
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false) private String name;

    @ManyToMany
    @JoinTable(name = "enrollments_mtm",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();

    protected Student() {}
    public Student(String name){ this.name = name; }

    public void enroll(Course c){ courses.add(c); c.getStudents().add(this); }
    public void drop(Course c){ courses.remove(c); c.getStudents().remove(this); }

    public Set<Course> getCourses() { return courses; }
}

@Entity
@Table(name = "courses")
class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false) private String title;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();

    protected Course() {}
    public Course(String title){ this.title = title; }
    public Set<Student> getStudents() { return students; }
}
```

**Kotlin — рекомендуемая join-entity с атрибутами**

```kotlin
package com.example.mtm

import jakarta.persistence.*
import java.time.Instant

@Entity
@Table(name = "students2")
class Student2(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = ""
) {
    @OneToMany(mappedBy = "student", cascade = [CascadeType.ALL], orphanRemoval = true)
    var enrollments: MutableSet<Enrollment> = mutableSetOf()

    fun enroll(course: Course2, role: String = "REGULAR") {
        val e = Enrollment(student = this, course = course, role = role)
        enrollments += e
        course.enrollments += e
    }

    fun drop(course: Course2) {
        val e = enrollments.firstOrNull { it.course == course } ?: return
        enrollments.remove(e)
        course.enrollments.remove(e)
        e.student = null; e.course = null
    }
}

@Entity
@Table(name = "courses2")
class Course2(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var title: String = ""
) {
    @OneToMany(mappedBy = "course", cascade = [CascadeType.ALL], orphanRemoval = true)
    var enrollments: MutableSet<Enrollment> = mutableSetOf()
}

@Entity
@Table(
    name = "enrollments",
    uniqueConstraints = [UniqueConstraint(columnNames = ["student_id", "course_id"])]
)
class Enrollment(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "student_id", nullable = false)
    var student: Student2? = null,

    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "course_id", nullable = false)
    var course: Course2? = null,

    @Column(nullable = false) var role: String = "REGULAR",
    @Column(nullable = false) var joinedAt: Instant = Instant.now()
)
```

---

## Встроенные значения: `@Embeddable` для value-объектов; `@MappedSuperclass` для общих полей, не для связей

`@Embeddable` позволяет моделировать **value-объекты**: неизменяемые (по смыслу) кусочки данных, которые живут только в составе сущности. Классическая примета — «нет собственного идентификатора и жизненного цикла». Адрес, деньги, интервал времени — отличные кандидаты. Встраивание даёт читаемую доменную модель и избавляет от плоских префиксов в коде (`userStreet`, `userZip`), оставляя их только в схеме через `@AttributeOverride`.

Value-объекты хороши для инвариантов. Вы можете валидировать состояние в конструкторе/билдере `@Embeddable` и быть уверены, что в БД не попадут «битые» значения. В JPA это обычные поля, поэтому индексы и уникальные ограничения объявляются на колонках владельца, что удобно для поиска.

`@MappedSuperclass` — другой инструмент: вынос **общих полей и маппингов** в базовый класс, который сам **не сущность** и таблицы не имеет. Типичный кейс — аудит: `createdAt`, `updatedAt`, `createdBy`. Все наследники получают колонки и аннотации. Важно: `@MappedSuperclass` не подходит для **связей** (`@ManyToOne` и коллекций) — это создаёт неявные графы и затрудняет анализ схемы. Связи лучше описывать в конкретных сущностях.

С точки зрения производительности `@Embeddable` бесплатен: это те же колонки в таблице, и Hibernate не создаёт лишних SQL-джоинов. Главное — не делать огромных вложенных структур с десятками колонок: это уже «впечатанные» денормализации и риск превратить таблицу в «широкую».

Ещё одна практика — **повторное использование** value-объектов. Если в нескольких сущностях встречается `Money` (валюта + сумма) или `Period` (от/до), их удобно переиспользовать как тип поля. Это сокращает количество дефектов — валидаторы и сериализация определены в одном месте и не дублируются.

С `@AttributeOverride` легко контролировать имена колонок и типы. Например, у адреса можно задать префикс `home_`/`billing_`, не дублируя сам класс. Для JSON-сериализации в REST ничего дополнительного не нужно: Jackson прекрасно сериализует `@Embeddable` как вложенный объект.

Про неизменяемость: JPA технически требует пустой конструктор, но вы можете скрыть сеттеры и предоставлять методы, которые создают **новый** value-объект. Это сильнее отражает доменную идею и снижает риск частичной модификации.

По тестируемости `@Embeddable` приятны: их можно создавать и валидировать отдельно от БД. Unit-тесты защищают инварианты и облегчают ревью.

И наконец, не путайте `@Embeddable` и «вложенные сущности». Если у части системы потребуется ссылаться на адрес отдельно (связи, ACL, аудит), значит адрес — **сущность** со своим PK, а не `@Embeddable`. На ранних этапах проще стартовать с value-объектов и «поднимать» их в сущности, когда реально появятся ссылочные требования.

**Java — `@Embeddable` Address + `@MappedSuperclass` Auditable**

```java
package com.example.embedded;

import jakarta.persistence.*;
import java.time.Instant;

@Embeddable
public class Address {
    @Column(name = "addr_city", nullable = false, length = 100)
    private String city;

    @Column(name = "addr_street", nullable = false, length = 200)
    private String street;

    @Column(name = "addr_zip", nullable = false, length = 12)
    private String zip;

    protected Address() {}
    public Address(String city, String street, String zip) {
        if (city == null || street == null || zip == null) throw new IllegalArgumentException("addr");
        this.city = city; this.street = street; this.zip = zip;
    }
    // getters...
}

@MappedSuperclass
public abstract class Auditable {
    @Column(name = "created_at", nullable = false)
    protected Instant createdAt = Instant.now();

    @Column(name = "updated_at", nullable = false)
    protected Instant updatedAt = Instant.now();

    @PreUpdate
    protected void touch() { this.updatedAt = Instant.now(); }
}

@Entity
@Table(name = "customers3")
public class Customer3 extends Auditable {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String name;

    @Embedded
    private Address address;

    protected Customer3() {}
    public Customer3(String name, Address address){ this.name = name; this.address = address; }
}
```

**Kotlin — те же идеи (встроенный адрес, базовый аудит)**

```kotlin
package com.example.embedded

import jakarta.persistence.*
import java.time.Instant

@Embeddable
class Address(
    @Column(name = "addr_city", nullable = false, length = 100)
    var city: String = "",
    @Column(name = "addr_street", nullable = false, length = 200)
    var street: String = "",
    @Column(name = "addr_zip", nullable = false, length = 12)
    var zip: String = ""
)

@MappedSuperclass
abstract class Auditable {
    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now()

    @Column(name = "updated_at", nullable = false)
    var updatedAt: Instant = Instant.now()

    @PreUpdate
    protected fun touch() { updatedAt = Instant.now() }
}

@Entity
@Table(name = "customers3")
class Customer3(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, length = 120)
    var name: String = "",

    @Embedded
    var address: Address = Address()
) : Auditable()
```

---

## События/слушатели: `@PrePersist/@PreUpdate` и entity-listener’ы — минимум логики, без I/O

События жизненного цикла сущности — удобный крючок для поддержания **инвариантов**: выставить временные метки, нормализовать формат, сгенерировать человекочитаемый `slug`. JPA предоставляет аннотации `@PrePersist`, `@PreUpdate`, `@PostLoad` и др., а также механизм внешних слушателей через `@EntityListeners`. Это мощный инструмент — и потому требует дисциплины.

Главное правило: **никакого I/O** в слушателях. Ни сетевых вызовов, ни обращений к файловой системе, ни даже к другим репозиториям. Почему? Эти методы выполняются внутри `flush`, часто — в самый неподходящий момент для внешних зависимостей. Любая задержка или ошибка приведёт к трудноотлавливаемым инцидентам. Ограничьтесь чистыми, быстрыми преобразованиями в памяти.

Второе правило — отсутствие скрытой бизнес-логики. Слушатели должны делать только технические вещи: timestamps, нормализация текстов, транслитерация для `slug`, «обрезка» длинных строк. Настоящая предметная логика должна жить в сервисах — её проще тестировать и контролировать транзакционные границы.

Внешние entity-listeners полезны, когда вы хотите переиспользовать одинаковое поведение между сущностями или держать доменную модель «чистой» от инфраструктурных аннотаций. Класс-слушатель получает сущность и может изменить её поля, но не должен знать про `EntityManager`.

Следите за производительностью: `@PreUpdate` вызывается при каждом `flush` у грязной сущности. Если вы, например, пересчитываете `slug` на основе заголовка при каждом апдейте, добавьте короткую проверку «изменился ли заголовок» — иначе получите лишнюю работу.

При генерации значений учитывайте **уникальность**. Если `slug` должен быть уникальным, слушатель — не место для поиска коллизий в БД (это I/O). Лучше хранить временный «сырой slug» и разрешать коллизии в сервисе (или через БД-констрейнт + обработку исключения), а слушатель пусть только нормализует.

Если вам нужны «доменные события» (например, «заказ создан»), не генерируйте их из `@PostPersist`. Этот момент неудобен: у вас нет инфраструктуры для публикации сообщений, и вы завяжетесь на контекст JPA. Лучше в сервисе после успешного сохранения опубликуйте событие через ApplicationEventPublisher / Kafka продюсер — это явнее и контролируемо.

Тестируйте слушатели простыми unit-тестами на сущности (без БД), а также интеграционными тестами на репозитории. В unit-тестах вызывайте методы слушателя напрямую — так вы быстрее отлавливаете регресс. В интеграционных убедитесь, что поля выставляются при `save`.

И наконец, помните, что слушатели — это «скрытая магия». Документируйте их поведение рядом с сущностями и избегайте чрезмерной «автоматики». Код, который «сам что-то сделал» при сохранении, легко удивляет нового разработчика.

**Java — entity-listener для slug и таймстемпов (без I/O)**

```java
package com.example.listener;

import jakarta.persistence.*;
import java.text.Normalizer;
import java.time.Instant;

@Entity
@EntityListeners(ArticleListener.class)
@Table(name = "articles3", uniqueConstraints = @UniqueConstraint(columnNames = "slug"))
public class Article3 {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 160)
    private String title;

    @Column(nullable = false, length = 200)
    private String slug;

    @Column(nullable = false) private Instant createdAt;
    @Column(nullable = false) private Instant updatedAt;

    protected Article3() {}
    public Article3(String title) { this.title = title; }

    // getters/setters
    public String getTitle() { return title; }
    public void setSlug(String slug) { this.slug = slug; }
    public String getSlug() { return slug; }
    public void setCreatedAt(Instant t){ this.createdAt = t; }
    public void setUpdatedAt(Instant t){ this.updatedAt = t; }
}

class ArticleListener {

    @PrePersist
    public void prePersist(Article3 a) {
        if (a.getSlug() == null || a.getSlug().isBlank()) {
            a.setSlug(slugify(a.getTitle()));
        }
        Instant now = Instant.now();
        a.setCreatedAt(now);
        a.setUpdatedAt(now);
    }

    @PreUpdate
    public void preUpdate(Article3 a) {
        // пересчитываем slug только при смене заголовка — в простом виде сравнивать негде,
        // поэтому обычно сервис сам выставляет slug. Здесь — сохранение updatedAt.
        a.setUpdatedAt(Instant.now());
    }

    private String slugify(String s) {
        String n = Normalizer.normalize(s, Normalizer.Form.NFD)
                .replaceAll("\\p{InCombiningDiacriticalMarks}+", "")
                .toLowerCase();
        n = n.replaceAll("[^a-z0-9\\s-]", "").trim().replaceAll("\\s+", "-");
        return n.substring(0, Math.min(200, n.length()));
    }
}
```

**Kotlin — тот же подход с `@EntityListeners`**

```kotlin
package com.example.listener

import jakarta.persistence.*
import java.text.Normalizer
import java.time.Instant
import kotlin.math.min

@Entity
@Table(name = "articles3", uniqueConstraints = [UniqueConstraint(columnNames = ["slug"])])
@EntityListeners(ArticleKListener::class)
class Article3(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, length = 160)
    var title: String = "",

    @Column(nullable = false, length = 200)
    var slug: String = "",

    @Column(nullable = false)
    var createdAt: Instant? = null,

    @Column(nullable = false)
    var updatedAt: Instant? = null
)

class ArticleKListener {

    @PrePersist
    fun prePersist(a: Article3) {
        if (a.slug.isBlank()) a.slug = slugify(a.title)
        val now = Instant.now()
        a.createdAt = now
        a.updatedAt = now
    }

    @PreUpdate
    fun preUpdate(a: Article3) {
        a.updatedAt = Instant.now()
    }

    private fun slugify(s: String): String {
        var n = Normalizer.normalize(s, Normalizer.Form.NFD)
            .replace("\\p{InCombiningDiacriticalMarks}+".toRegex(), "")
            .lowercase()
        n = n.replace("[^a-z0-9\\s-]".toRegex(), "").trim().replace("\\s+".toRegex(), "-")
        return n.substring(0, min(200, n.length))
    }
}
```

# 7. Кеширование 2-го уровня и кеш запросов

*Ниже — практическое руководство по включению и безопасной эксплуатации L2-кеша Hibernate и кеша запросов. Покажу рабочие конфиги и код (Java и Kotlin), а также нюансы инвалидации и распределённости.*

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    // JCache (JSR-107) + Ehcache 3 как провайдер L2
    implementation 'org.ehcache:ehcache:3.10.8'
    implementation 'javax.cache:cache-api:1.1.1'
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
    implementation("org.ehcache:ehcache:3.10.8")
    implementation("javax.cache:cache-api:1.1.1")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**application.yml — включаем L2 и кеш запросов (Hibernate + JCache/Ehcache)**

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        # Укажем провайдера JCache (Ehcache 3)
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
        generate_statistics: true
  cache:
    type: jcache
    jcache:
      config: classpath:ehcache.xml  # конфигурация регионов
logging:
  level:
    org.hibernate.SQL: warn
    org.hibernate.orm.cache: debug
```

**ehcache.xml — регионы L2 и кеша запросов (classpath:ehcache.xml)**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns='http://www.ehcache.org/v3'
        xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
        xmlns:jcache='http://www.ehcache.org/v3/jsr107'
        xsi:schemaLocation="
           http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.0.xsd
           http://www.ehcache.org/v3/jsr107 http://www.ehcache.org/schema/ehcache-107-ext-3.0.xsd">

    <cache alias="country">
        <key-type>java.lang.Long</key-type>
        <value-type>com.example.cache.Country</value-type>
        <resources>
            <heap unit="entries">10000</heap>
            <offheap unit="MB">64</offheap>
        </resources>
        <expiry><ttl unit="minutes">0</ttl></expiry> <!-- 0 => без TTL (read-only справочник) -->
    </cache>

    <cache alias="country.byName"> <!-- регион для кеша запросов -->
        <key-type>java.lang.Object</key-type>
        <value-type>java.util.List</value-type>
        <resources>
            <heap unit="entries">1000</heap>
        </resources>
        <expiry><ttl unit="minutes">10</ttl></expiry>
    </cache>

    <!-- Включаем интеграцию JCache (имена регионов = alias) -->
    <jcache:defaults enable-management="true" enable-statistics="true"/>
</config>
```

---

## Когда имеет смысл L2: справочники/редко меняемые сущности; регионы, режимы (read-only/non-strict/transactional)

L2-кеш (второй уровень) хранит **снимки сущностей** за пределами Persistence Context’а и переживает границы транзакций и сессий. Это снижает количество SELECT по «холодным» данным и снимает нагрузку с БД. Лучшие кандидаты — **справочники и редко меняемые записи**: страны, валюты, статусы, тарифные планы. Там, где запись «живёт» долго и почти не обновляется, L2 даёт стабильный выигрыш и минимальные риски несогласованности.

L2 работает по регионам. Для каждой сущности можно указать **стратегию конкурентности**:

* `READ_ONLY` — максимально дёшево и безопасно, **только** для неизменяемых данных.
* `NONSTRICT_READ_WRITE` — допускает «окно» несогласованности, быстрее `READ_WRITE`.
* `READ_WRITE` — обеспечивает «мягкую» согласованность с блокировкой кэш-записей.
* `TRANSACTIONAL` — требует транзакционный провайдер кэша (редко в проектной практике).

Правило выбора простое: **если сущность меняется редко — READ_ONLY**. Если меняется редко, но нужно обновлять «сейчас» — `READ_WRITE` (дороже). Для активно меняемых сущностей L2 чаще **отключают**, чтобы не раздувать память и не ловить частые инвалидации.

Следите за бюджетом памяти и GC: L2 — это реальные байты в heap/off-heap у провайдера кэша. Распределяйте регионы, задавайте явную ёмкость (entries/MB), мониторьте hit/miss. «Бездонные» кэши превращаются в паузы GC и регресс производительности.

Ещё нюанс: L2 кеширует **по id**. Если у вас часто идут чтения «по бизнес-ключу» (например, код страны), отдавайте DTO через **кеш запросов** (см. следующий раздел) или заведите маленький «индекс»-кэш (map код→id) в памяти сервиса.

Наконец, не путайте L2 с Spring Cache. Spring Cache аннотации (`@Cacheable`) — абстракция над любыми кэшами; L2 — встроенная функция Hibernate. Их можно использовать **вместе**, но отвечают они за разное.

**Java — сущность-справочник с L2 и статистика попаданий**

```java
package com.example.cache;

import jakarta.persistence.*;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;

import java.util.Optional;
import java.util.List;

@Entity
@Table(name = "countries")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "country")
public class Country {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true, length = 2)
    private String code;
    @Column(nullable = false) private String name;
    protected Country() {}
    public Country(String code, String name){ this.code=code; this.name=name; }
    public Long getId(){ return id; } public String getCode(){ return code; } public String getName(){ return name; }
}

interface CountryRepository extends org.springframework.data.jpa.repository.JpaRepository<Country, Long> {
    Optional<Country> findByCode(String code);
    List<Country> findTop10ByOrderByNameAsc();
}

@Service
class CountryService {
    private final CountryRepository repo;
    private final SessionFactory sf;
    CountryService(CountryRepository repo, EntityManagerFactory emf) {
        this.repo = repo; this.sf = emf.unwrap(SessionFactory.class);
    }

    @Transactional(readOnly = true)
    public Country byId(Long id) {
        Statistics st = sf.getStatistics(); st.clear();
        Country c1 = repo.findById(id).orElseThrow(); // 1й раз — miss -> БД -> put в L2
        Country c2 = repo.findById(id).orElseThrow(); // 2й раз — hit из L2
        System.out.println("L2 hits=" + st.getSecondLevelCacheHitCount()
                + ", puts=" + st.getSecondLevelCachePutCount()
                + ", misses=" + st.getSecondLevelCacheMissCount());
        return c2;
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.cache

import jakarta.persistence.*
import org.hibernate.annotations.Cache
import org.hibernate.annotations.CacheConcurrencyStrategy
import org.hibernate.SessionFactory
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "countries")
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "country")
class Country(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 2)
    var code: String = "",
    @Column(nullable = false)
    var name: String = ""
)

interface CountryRepository : JpaRepository<Country, Long> {
    fun findByCode(code: String): java.util.Optional<Country>
    fun findTop10ByOrderByNameAsc(): List<Country>
}

@Service
class CountryService(repo: CountryRepository, emf: EntityManagerFactory) {
    private val repo = repo
    private val sf: SessionFactory = emf.unwrap(SessionFactory::class.java)

    @Transactional(readOnly = true)
    fun byId(id: Long): Country {
        val st = sf.statistics; st.clear()
        val c1 = repo.findById(id).orElseThrow()
        val c2 = repo.findById(id).orElseThrow()
        println("L2 hits=${st.secondLevelCacheHitCount}, puts=${st.secondLevelCachePutCount}, misses=${st.secondLevelCacheMissCount}")
        return c2
    }
}
```

---

## Кеш запросов: риск «несогласованности», ключи инвалидации, осторожно с частыми изменениями

Кеш запросов (query cache) хранит **результаты** конкретных JPQL/Criteria/native-запросов (обычно — список id/rowset) по ключу «SQL + параметры + регион». Это удобно для часто повторяющихся **однотипных** чтений: «топ-10 стран по имени», «список статусов», «популярные товары дня». Но у query cache хрупкая природа: любая модификация затрагиваемых таблиц требует инвалидации соответствующих регионов, иначе вы увидите «вчерашний» список.

Правила безопасного применения:

1. Кешируйте **стабильные** запросы над редкими изменениями.
2. Всегда задавайте **регион** и TTL (в провайдере), чтобы старые ответы не жили вечно.
3. При записях — **инвалидируйте** нужные регионы (или все регионы сущности), если результат зависит от этих данных.
4. Не кешируйте параметризованные запросы с высокой кардинальностью (много разных значений параметров) — кэш будет «забит» уникальными ключами и перестанет окупаться.

**Java — кешируемый запрос с регионом + инвалидация при записи**

```java
package com.example.qcache;

import jakarta.persistence.*;
import org.hibernate.jpa.HibernateHints;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Entity
@Table(name = "tags2")
class Tag2 {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false, unique = true) String name;
    protected Tag2() {}
    Tag2(String n){ this.name=n; }
}

@Repository
interface TagRepository extends org.springframework.data.jpa.repository.JpaRepository<Tag2, Long> {
    // можно и через @Query + Pageable, но для демонстрации используем EntityManager напрямую в сервисе
}

@Service
class TagService {
    @PersistenceContext private EntityManager em;
    private final SessionFactory sf;
    private final TagRepository repo;

    TagService(EntityManagerFactory emf, TagRepository repo) {
        this.sf = emf.unwrap(SessionFactory.class);
        this.repo = repo;
    }

    @Transactional(readOnly = true)
    public List<Tag2> topAlphabetical(int limit) {
        var st = sf.getStatistics(); st.clear();
        TypedQuery<Tag2> q = em.createQuery(
            "select t from Tag2 t order by t.name asc", Tag2.class);
        q.setMaxResults(limit);
        q.setHint(HibernateHints.HINT_CACHEABLE, true);
        q.setHint(HibernateHints.HINT_CACHE_REGION, "country.byName"); // регион из ehcache.xml
        List<Tag2> first = q.getResultList();       // 1-й раз -> miss/put
        List<Tag2> second = q.getResultList();      // 2-й раз -> hit (query cache)
        System.out.println("Query cache hits=" + st.getQueryCacheHitCount()
            + ", puts=" + st.getQueryCachePutCount());
        return second;
    }

    @Transactional
    public Tag2 create(String name) {
        Tag2 t = new Tag2(name);
        repo.save(t);
        // Инвалидация: менялись данные таблицы tags2, выбьем регион кеша запросов.
        em.getEntityManagerFactory().getCache().evict(Tag2.class); // L2 региона сущности
        // Для query cache regions — через SessionFactory API:
        sf.getCache().evictQueryRegion("country.byName");
        return t;
    }
}
```

**Kotlin — кешируемый запрос + очистка регионов**

```kotlin
package com.example.qcache

import jakarta.persistence.*
import org.hibernate.jpa.HibernateHints
import org.hibernate.SessionFactory
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "tags2")
class Tag2(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true) var name: String = ""
)

@Repository
interface TagRepository : JpaRepository<Tag2, Long>

@Service
class TagService(
    emf: EntityManagerFactory,
    private val repo: TagRepository,
    @PersistenceContext private val em: EntityManager
) {
    private val sf: SessionFactory = emf.unwrap(SessionFactory::class.java)

    @Transactional(readOnly = true)
    fun topAlphabetical(limit: Int): List<Tag2> {
        val st = sf.statistics; st.clear()
        val q = em.createQuery("select t from Tag2 t order by t.name asc", Tag2::class.java)
        q.maxResults = limit
        q.setHint(HibernateHints.HINT_CACHEABLE, true)
        q.setHint(HibernateHints.HINT_CACHE_REGION, "country.byName")
        val first = q.resultList
        val second = q.resultList
        println("Query cache hits=${st.queryCacheHitCount}, puts=${st.queryCachePutCount}")
        return second
    }

    @Transactional
    fun create(name: String): Tag2 {
        val t = Tag2(name = name)
        repo.save(t)
        em.entityManagerFactory.cache.evict(Tag2::class.java)
        sf.cache.evictQueryRegion("country.byName")
        return t
    }
}
```

Комментарий: в примерах выше мы **сознательно** держим query cache только для стабильного «топ-N». Для динамических поисков по множеству параметров лучше использовать L2 для связанных справочников или вовсе обойтись без кеша запросов.

---

## Распределённость: репликация/инвалидация между инстансами, влияние на память и GC

В кластере (несколько инстансов приложения) L2-кеш нужно **согласовывать** между узлами. Есть два пути:

1. **Локальный L2 + внешняя инвалидация.** Каждый инстанс держит локальный Ehcache, а при записях вы выполняете **широковещательную** инвалидацию (через брокер событий, Redis pub/sub, Kafka). Плюсы: минимальная латентность чтения, независимость от сети на hot-path. Минусы: сложность инвалидации, риск «окна» несогласованности на время доставки события.

2. **Распределённый JCache-провайдер.** Подставляете провайдер с кластером (например, Hazelcast/Infinispan в режиме JCache). Тогда регионы L2 и query cache **общие** для всех инстансов (репликация/инвалидация на уровне провайдера). Плюсы: прозрачность. Минусы: зависимость от сети/кластерного кэша, дополнительные ресурсы, нюансы настройки.

Любой распределённый вариант усиливает требования к **ёмкости** и **мониторингу**. Регионы должны иметь чёткие лимиты (entries/MB), а вы — метрики hit/miss, время «get/put», количество инвалидаций. Память кэша напрямую влияет на GC (если heap) и на задержки сетевых операций (если кластерный).

Отдельная практика — **инвалидация при записи**. Даже с распределённым провайдером полезно явно чистить регионы, зависящие от изменённых данных. Для Hibernate это:

* `EntityManagerFactory.getCache().evict(Entity.class)` / `evict(Entity.class, id)` — сущностные регионы.
* `SessionFactory.getCache().evictQueryRegion("region")` / `evictQueryRegions()` — регионы кеша запросов.
  Такая явность уменьшает «случайные» несогласованности и облегчает поиск проблем.

Если вы **часто обновляете** данные (горячие заказы, корзины), L2 может стать скорее **вредным**: много инвалидаций, мало попаданий, большой расход памяти. На таких сущностях L2 лучше отключать и опираться на точные чтения из БД (возможно, с батч-фетчем, проекциями и грамотными индексами).

И наконец, держите транзакции короткими. Долгие транзакции + L2 + кластер — отличная почва для «странных» гонок и задержек инвалидации. «Сначала подумай — потом кэшируй»: включайте L2 точечно, по зрелым горячим точкам чтения.

**Java — пример «инстанс-нейтральной» инвалидации при записи**

```java
package com.example.dist;

import jakarta.persistence.EntityManagerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.hibernate.SessionFactory;

@Service
public class InvalidationService {
    private final EntityManagerFactory emf;
    private final SessionFactory sf;
    public InvalidationService(EntityManagerFactory emf) {
        this.emf = emf; this.sf = emf.unwrap(SessionFactory.class);
    }

    @Transactional
    public void afterCountryChanged(Long id) {
        // Вызывайте после commit'а (TransactionSynchronization) — здесь для простоты внутри транзакции.
        emf.getCache().evict(com.example.cache.Country.class, id);     // сущностный регион
        sf.getCache().evictQueryRegion("country.byName");               // регион кеша запросов
        // При наличии внешнего pub/sub — отправьте «invalidate(country,id)»
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.dist

import jakarta.persistence.EntityManagerFactory
import org.hibernate.SessionFactory
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import com.example.cache.Country

@Service
class InvalidationService(emf: EntityManagerFactory) {
    private val emf = emf
    private val sf: SessionFactory = emf.unwrap(SessionFactory::class.java)

    @Transactional
    fun afterCountryChanged(id: Long) {
        emf.cache.evict(Country::class.java, id)
        sf.cache.evictQueryRegion("country.byName")
        // при необходимости — опубликовать событие в брокер
    }
}
```

**(Опционально) Hazelcast в роли распределённого провайдера JCache**
*Если нужно общий L2 между инстансами, можно подключить JCache-провайдер Hazelcast:*

* зависимости: `com.hazelcast:hazelcast:5.4.0`, `com.hazelcast:hazelcast-client:5.4.0` (если клиент-сервер), `javax.cache:cache-api:1.1.1`;
* `application.yml`:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: com.hazelcast.client.cache.HazelcastClientCachingProvider
  cache:
    type: jcache
    jcache:
      config: classpath:hazelcast-client.xml
```

* в `hazelcast-client.xml` описать кластеры и карты с теми же именами регионов (`country`, `country.byName`).

---

**Итоги практики**

* Включайте L2 **точечно** и в первую очередь для справочников → `READ_ONLY` регионы.
* Кеш запросов — только для **стабильных однотипных** выборок, всегда задавайте **регион** и TTL, чистите его при записи.
* В кластере либо используйте распределённый JCache-провайдер, либо шлите **инвалидации** между инстансами.
* Мониторинг — обязателен: hit/miss, размер регионов, GC/heap и задержки.

# 8. Чтение больших объёмов и стриминг

*Цель этой подтемы — показать, как безопасно и экономно по памяти читать десятки/сотни тысяч строк через JPA/Hibernate и (при необходимости) через JDBC. Сфокусируемся на правильных транзакционных границах, `Stream<T>`, хинтах `fetchSize/readOnly`, курсорной передаче данных и выгрузке «мимо памяти».*

**Базовые зависимости для примеров (актуальны во всей подтеме)**

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
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
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
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
```

**application.yml (настройки, помогающие стримингу)**

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc.fetch_size: 1000        # дефолтный fetch size для запросов Hibernate
        generate_statistics: true
  datasource:
    url: jdbc:postgresql://localhost:5432/app?defaultRowFetchSize=1000
    username: app
    password: app
    hikari:
      auto-commit: false             # для серверных курсоров в PG автокоммит должен быть выключен
logging:
  level:
    org.hibernate.SQL: warn
```

---

## Потоки из репозиториев (`Stream<T>`), требования к `@Transactional` и закрытию

Spring Data JPA умеет отдавать результаты как `java.util.stream.Stream<T>`. Это позволяет обрабатывать строки **по мере поступления** (lazily), не материализуя весь результат в память. Но у стримов есть два условия: 1) необходима активная транзакция, чтобы держать соединение и курсор открытыми, 2) стрим **обязательно закрывать** (try-with-resources), иначе соединение «подвиснет».

Внутри транзакции Hibernate/JDBC создают курсор (в Postgres — server-side cursor при `fetchSize>0` и `autocommit=false`). Чтение идёт порциями. Если транзакция завершится раньше, чем вы дочитали поток, — получите `LazyInitializationException`/`SQLException`. Поэтому экспортирующие методы помечаем `@Transactional(readOnly = true)` и читаем поток **внутри** этого метода.

Ещё один важный хинт — помечать запрос и/или сущности как read-only: Hibernate перестанет делать снимки для dirty checking, что существенно экономит CPU и heap. В Spring Data это можно задать через `@QueryHints` с `HibernateHints.HINT_READ_ONLY = true`. Комбинация `readOnly TX + read-only Query + fetchSize` даёт оптимальный режим «чистого чтения».

Не забывайте про `flush/clear` — если в рамках экспорта вы всё же **что-то пишете** (логируете в БД, помечаете прогресс), периодически сбрасывайте PC и очищайте его, иначе L1-кэш начнёт расти. Но в большинстве экспортов лучше не мешать чтение и запись в одну транзакцию.

При маппинге сущностей в плоские строки/DTO после использования можно делать `em.detach(entity)`, чтобы не держать граф в PC. Это особенно заметно, когда сущность тянет ленивые коллекции, которые вы не планируете использовать — `detach` исключит случайные обращения.

На стороне Postgres не включайте `fetch join` огромных коллекций в таком запросе: объём строки кратно раздуется, а курсор потеряет смысл. Для «толстой карточки» лучше отдельный запрос с `JOIN FETCH` по конкретному id.

И последнее: удачно использовать проекции (DTO / интерфейсные) прямо в стриме — это сокращает размер передаваемых данных и освобождает от PC (DTO не «managed»). Такой подход критичен, если в JSON/CSV вам нужны только 3–5 полей от сущности.

**Java — репозиторий со стримом + хинты и сервис-экспорт в CSV**

```java
package com.example.streams;

import jakarta.persistence.*;
import org.hibernate.jpa.HibernateHints;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.BufferedWriter;
import java.io.OutputStream;
import java.io.OutputStreamWriter;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.stream.Stream;

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) Long customerId;
    @Column(nullable = false) Long totalCents;
    @Column(nullable = false) Instant createdAt = Instant.now();
    protected Order() {}
}

interface OrderRow {
    Long getId();
    Long getCustomerId();
    Long getTotalCents();
    Instant getCreatedAt();
}

@Repository
interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
        select o.id as id, o.customerId as customerId, o.totalCents as totalCents, o.createdAt as createdAt
        from Order o
        where o.createdAt >= :from
        order by o.id
    """)
    @QueryHints({
        @QueryHint(name = HibernateHints.HINT_FETCH_SIZE, value = "1000"),
        @QueryHint(name = HibernateHints.HINT_READ_ONLY, value = "true")
    })
    Stream<OrderRow> streamSince(@Param("from") Instant from);
}

@Service
class OrderExportService {
    private final OrderRepository repo;
    @PersistenceContext private EntityManager em;

    OrderExportService(OrderRepository repo) { this.repo = repo; }

    @Transactional(readOnly = true)
    public void exportSince(Instant from, OutputStream out) throws Exception {
        try (Stream<OrderRow> s = repo.streamSince(from);
             BufferedWriter w = new BufferedWriter(new OutputStreamWriter(out, StandardCharsets.UTF_8))) {
            w.write("id,customer_id,total_cents,created_at\n");
            final int[] i = {0};
            s.forEach(r -> {
                try {
                    w.write(r.getId() + "," + r.getCustomerId() + "," + r.getTotalCents() + "," + r.getCreatedAt() + "\n");
                    if (++i[0] % 5000 == 0) w.flush(); // не копим большой буфер
                } catch (Exception e) { throw new RuntimeException(e); }
            });
            w.flush();
        }
    }
}
```

**Kotlin — аналогичный стрим + экспорт**

```kotlin
package com.example.streams

import jakarta.persistence.*
import org.hibernate.jpa.HibernateHints
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.io.BufferedWriter
import java.io.OutputStream
import java.io.OutputStreamWriter
import java.nio.charset.StandardCharsets
import java.time.Instant
import java.util.stream.Stream

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var customerId: Long = 0,
    @Column(nullable = false) var totalCents: Long = 0,
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

interface OrderRow {
    fun getId(): Long
    fun getCustomerId(): Long
    fun getTotalCents(): Long
    fun getCreatedAt(): Instant
}

@Repository
interface OrderRepository : JpaRepository<Order, Long> {

    @Query(
        """
        select o.id as id, o.customerId as customerId, o.totalCents as totalCents, o.createdAt as createdAt
        from Order o
        where o.createdAt >= :from
        order by o.id
        """
    )
    @QueryHints(
        value = [
            QueryHint(name = HibernateHints.HINT_FETCH_SIZE, value = "1000"),
            QueryHint(name = HibernateHints.HINT_READ_ONLY, value = "true")
        ]
    )
    fun streamSince(@Param("from") from: Instant): Stream<OrderRow>
}

@Service
class OrderExportService(private val repo: OrderRepository) {

    @Transactional(readOnly = true)
    fun exportSince(from: Instant, out: OutputStream) {
        repo.streamSince(from).use { s ->
            BufferedWriter(OutputStreamWriter(out, StandardCharsets.UTF_8)).use { w ->
                w.write("id,customer_id,total_cents,created_at\n")
                var i = 0
                s.forEach { r ->
                    w.write("${r.id},${r.customerId},${r.totalCents},${r.createdAt}\n")
                    i++
                    if (i % 5000 == 0) w.flush()
                }
                w.flush()
            }
        }
    }
}
```

---

## Скролл/курсор: forward-only, `fetchSize`/`setFetchSize`, минимизация памяти

Стримы — частный случай курсорного чтения. Базовая идея: держать курсор **forward-only** и забирать данные порциями у драйвера. В PostgreSQL курсоры включаются автоматически, когда **(а)** транзакция активна (autocommit=false) и **(б)** `fetchSize > 0`. В примерах выше мы задали и глобальное `hibernate.jdbc.fetch_size`, и хинт на сам запрос — так надёжнее.

Если нужен полный контроль, можно использовать `JdbcTemplate` и настроить `fetchSize` напрямую на `PreparedStatement`. Это обходит ORM и гарантирует, что драйвер отдаёт не более N строк за раз. Также у PG есть параметр `defaultRowFetchSize` в JDBC URL — он выставляет значение по умолчанию для всех запросов.

Forward-only курсор означает, что нельзя «мотать» результат назад/вперёд и нельзя рассчитывать на `ResultSet.last()`. Это плата за низкое потребление памяти. В обмен вы можете обрабатывать миллионы строк при стабильном heap, если не копите большие структуры в коллекции.

Важно фильтровать и сортировать по индексируемым полям: курсор не спасёт от full scan по горячей таблице, который будет «пережёвывать» тонны данных на стороне БД. Всегда начинайте с селективного `where` и `order by` по индексам, чтобы сервер быстро строил план.

Не забывайте про таймауты. Долгие курсоры — это долго удерживаемые соединения и транзакции. Выставляйте `statement_timeout`/`queryTimeout` и логируйте длительность выгрузок. Если выгрузка может длиться десятки минут, лучше её бить на **логические чанки** (по id/времени) и делать по одной транзакции на чанк.

И последнее — не «увлекайтесь» гигантским `fetchSize`. Сладкое место обычно 200–2000. Слишком маленький — много round-trip’ов; слишком большой — большие пакеты в сети и всплески памяти.

**Java — курсорное чтение через `JdbcTemplate` с `fetchSize` и обработчиком строк**

```java
package com.example.cursor;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowCallbackHandler;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.BufferedWriter;
import java.io.OutputStream;
import java.io.OutputStreamWriter;
import java.nio.charset.StandardCharsets;
import java.sql.ResultSet;
import java.sql.SQLException;

@Service
public class CursorExportService {
    private final JdbcTemplate jdbc;

    public CursorExportService(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
        this.jdbc.setFetchSize(1000); // дефолт для всех запросов через этот JdbcTemplate
    }

    @Transactional(readOnly = true)
    public void export(OutputStream out) throws Exception {
        try (BufferedWriter w = new BufferedWriter(new OutputStreamWriter(out, StandardCharsets.UTF_8))) {
            w.write("id,customer_id,total_cents,created_at\n");
            RowCallbackHandler rch = new RowCallbackHandler() {
                @Override public void processRow(ResultSet rs) throws SQLException {
                    try {
                        w.write(rs.getLong("id") + "," + rs.getLong("customer_id") + ","
                                + rs.getLong("total_cents") + "," + rs.getTimestamp("created_at").toInstant() + "\n");
                    } catch (Exception e) { throw new RuntimeException(e); }
                }
            };
            jdbc.query(
                "select id, customer_id, total_cents, created_at from orders where created_at >= now() - interval '30 days' order by id",
                rch
            );
            w.flush();
        }
    }
}
```

**Kotlin — то же, курсорное чтение через `JdbcTemplate`**

```kotlin
package com.example.cursor

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.RowCallbackHandler
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.io.BufferedWriter
import java.io.OutputStream
import java.io.OutputStreamWriter
import java.nio.charset.StandardCharsets
import java.sql.ResultSet

@Service
class CursorExportService(private val jdbc: JdbcTemplate) {

    init { jdbc.fetchSize = 1000 }

    @Transactional(readOnly = true)
    fun export(out: OutputStream) {
        BufferedWriter(OutputStreamWriter(out, StandardCharsets.UTF_8)).use { w ->
            w.write("id,customer_id,total_cents,created_at\n")
            val handler = RowCallbackHandler { rs: ResultSet ->
                w.write("${rs.getLong("id")},${rs.getLong("customer_id")},${rs.getLong("total_cents")},${rs.getTimestamp("created_at").toInstant()}\n")
            }
            jdbc.query(
                "select id, customer_id, total_cents, created_at from orders where created_at >= now() - interval '30 days' order by id",
                handler
            )
            w.flush()
        }
    }
}
```

---

## Read-only транзакции + хинты драйвера; выгрузка в файлы/каналы без материализации всего списка

`@Transactional(readOnly = true)` — необходимый контекст для долгого чтения: отключает `flush`, даёт драйверу оптимизационный хинт и держит соединение/курсор открытыми. Дополнительно на уровне запроса/сессии указывайте **read-only** хинты (HibernateHints.HINT_READ_ONLY, `Session.setDefaultReadOnly(true)`), чтобы Hibernate не занимался dirty checking.

Выгрузку лучше писать **строго потоково**: читаете строку — сразу сериализуете/пишете в `OutputStream`/`WritableByteChannel`. Не формируйте `List<T>`/`StringBuilder` на миллионы элементов. Для CSV используйте буферизированные writer’ы и периодический `flush`; для JSON — стриминговые API (`JsonGenerator` Jackson’а) или хотя бы «по-строчному» NDJSON (один объект — одна строка).

На уровне драйвера используйте параметры, ускоряющие «бестелесную» выгрузку. В Postgres можно задать `stringtype=unspecified` (снижает конвертации в некоторых сценариях), `defaultRowFetchSize`, а также `readOnly=true` на соединении (в некоторых базах меняет планы). Но главный выигрыш даёт именно `fetchSize` и отсутствие ненужных преобразований на стороне приложения.

Если экспорт длится долго, добавьте **heartbeat-логи** (каждые N тысяч строк) и обработку отмены (проверка флага/Thread.interrupted). Так вы избежите 30-минутных «чёрных ящиков», когда непонятно, жив ли процесс.

При необходимости шифрования/компрессии не пишите сначала на диск, а потом в архив: используйте **каскад потоков** (`GZIPOutputStream` поверх исходного `OutputStream`) — это сохраняет потоковую природу и экономит диск/IO.

Не забывайте про «чистоту» API: экспорт должен быть **идемпотентным** и воспроизводимым. Фиксируйте границы чтения (`from`/`to`/`asOf`) и версию схемы в заголовке файла. Это поможет повторять экспорт и отлаживать отчёты.

И, наконец, измеряйте. Разница между «наивным» `findAll()` и курсорным `fetchSize=1000` — порядок величины по памяти и по времени GC. Добавьте метрики (скорость строк/сек, общее время, средний размер чанка) и алерты на деградацию.

**Java — потоковая выгрузка в NDJSON через Jackson JsonGenerator**

```java
package com.example.streamingjson;

import com.fasterxml.jackson.core.JsonFactory;
import com.fasterxml.jackson.core.JsonGenerator;
import jakarta.persistence.*;
import org.hibernate.jpa.HibernateHints;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.OutputStream;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.stream.Stream;

@Entity
@Table(name = "events")
class Event {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String type;
    @Column(nullable = false) Instant createdAt = Instant.now();
    protected Event() {}
}

interface EventView {
    Long getId();
    String getType();
    Instant getCreatedAt();
}

@Repository
interface EventRepository extends JpaRepository<Event, Long> {
    @Query("""
        select e.id as id, e.type as type, e.createdAt as createdAt
        from Event e
        where e.createdAt between :from and :to
        order by e.id
    """)
    @QueryHints({
        @QueryHint(name = HibernateHints.HINT_FETCH_SIZE, value = "1000"),
        @QueryHint(name = HibernateHints.HINT_READ_ONLY, value = "true")
    })
    Stream<EventView> streamRange(@Param("from") Instant from, @Param("to") Instant to);
}

@Service
class EventExportJsonService {
    private final EventRepository repo;
    EventExportJsonService(EventRepository repo) { this.repo = repo; }

    @Transactional(readOnly = true)
    public void exportNdjson(Instant from, Instant to, OutputStream out) throws Exception {
        JsonFactory f = new JsonFactory();
        try (Stream<EventView> s = repo.streamRange(from, to);
             JsonGenerator g = f.createGenerator(out, com.fasterxml.jackson.core.JsonEncoding.UTF8)) {
            s.forEach(ev -> {
                try {
                    g.writeStartObject();
                    g.writeNumberField("id", ev.getId());
                    g.writeStringField("type", ev.getType());
                    g.writeStringField("createdAt", ev.getCreatedAt().toString());
                    g.writeEndObject();
                    g.writeRaw('\n'); // NDJSON: один объект — одна строка
                } catch (Exception e) { throw new RuntimeException(e); }
            });
            g.flush();
        }
    }
}
```

**Kotlin — потоковая NDJSON-выгрузка**

```kotlin
package com.example.streamingjson

import com.fasterxml.jackson.core.JsonEncoding
import com.fasterxml.jackson.core.JsonFactory
import com.fasterxml.jackson.core.JsonGenerator
import jakarta.persistence.*
import org.hibernate.jpa.HibernateHints
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.io.OutputStream
import java.time.Instant
import java.util.stream.Stream

@Entity
@Table(name = "events")
class Event(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var type: String = "",
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

interface EventView {
    fun getId(): Long
    fun getType(): String
    fun getCreatedAt(): Instant
}

@Repository
interface EventRepository : JpaRepository<Event, Long> {
    @Query(
        """
        select e.id as id, e.type as type, e.createdAt as createdAt
        from Event e
        where e.createdAt between :from and :to
        order by e.id
        """
    )
    @QueryHints(
        value = [
            QueryHint(name = HibernateHints.HINT_FETCH_SIZE, value = "1000"),
            QueryHint(name = HibernateHints.HINT_READ_ONLY, value = "true")
        ]
    )
    fun streamRange(@Param("from") from: Instant, @Param("to") to: Instant): Stream<EventView>
}

@Service
class EventExportJsonService(private val repo: EventRepository) {

    @Transactional(readOnly = true)
    fun exportNdjson(from: Instant, to: Instant, out: OutputStream) {
        val factory = JsonFactory()
        repo.streamRange(from, to).use { s ->
            factory.createGenerator(out, JsonEncoding.UTF8).use { g: JsonGenerator ->
                s.forEach { ev ->
                    g.writeStartObject()
                    g.writeNumberField("id", ev.id)
                    g.writeStringField("type", ev.type)
                    g.writeStringField("createdAt", ev.createdAt.toString())
                    g.writeEndObject()
                    g.writeRaw('\n')
                }
                g.flush()
            }
        }
    }
}
```

**Итоги практики для больших чтений**

* Держите **транзакцию read-only** и используйте `Stream<T>`/`RowCallbackHandler` для курсорного чтения.
* Настраивайте `fetchSize` (1000±) на уровне Hibernate/запроса/`JdbcTemplate`.
* Помечайте запросы **read-only**, избегайте материализации в память, пишите сразу в поток/файл.
* Дробите выгрузку на **чанки** по диапазонам ключей/времени, логируйте прогресс и ставьте таймауты.


# 9. DB-специфика и нестандартные типы

*В этой подтеме сосредоточимся на том, чего нет «в учебниках по чистому JPA»: типы и приёмы, специфичные для СУБД (на примере PostgreSQL), и как их безопасно и эффективно использовать в Spring Data JPA. Покажу варианты на «чистом» JPA через `@Converter` и через специализированные типы Hibernate (hibernate-types), а также практики индексации и «виртуальных» таблиц.*

**Зависимости (общие для примеров подтемы)**

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

    // Hibernate Types для JSON/массивов/диапазонов/hstore (под Hibernate 6)
    implementation 'com.vladmihalcea:hibernate-types-60:2.21.1'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
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
    implementation("com.vladmihalcea:hibernate-types-60:2.21.1")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
```

---

## JSON/JSONB, массивы, диапазоны, `hstore`: `@Converter` vs Hibernate Types; индексы (например, GIN для JSONB)

PostgreSQL даёт богатую палитру «нестандартных» типов: `jsonb` для полуструктурированных данных, массивы (`text[]`, `int[]`), диапазоны (`int4range`, `tstzrange`) и ключ-значение (`hstore`). На уровне JPA их можно представить двумя путями. Первый — «чистый» JPA через `@Converter`, где мы сами превращаем объект в строку (или `PGobject`) и обратно. Второй — воспользоваться библиотекой **Hibernate Types**, в которой уже реализованы маппинги и диалектные оптимизации под Hibernate 6. Оба пути валидны: конвертер — полностью под вашим контролем, Hibernate Types — быстрее стартует и богаче по фичам.

Начнём с **JSONB**. Если данные полуструктурированные и меняются, `jsonb` удобен: он бинарно нормализует JSON, умеет операторы `@>`, `?`, `#>>`, поддерживает индексы GIN. В JPA через `@Converter` вы можете хранить поле как `String` и сериализовать объект Jackson’ом, но теряете типовую безопасность и часть операторов в критериях. Hibernate Types позволяет объявить поле типа `JsonNode`/`Map<String,Object>` и работать с ним прямо как с объектом, а в запросах использовать native SQL с операторными индексами.

**Массивы** в PG (`text[]`, `uuid[]`) пригодятся, если нужна компактная коллекция скаляров без отдельной таблицы связей. Они хорошо индексируются GIN/GiST, подходят для «тегов»/«ролей»/«флагов». В Hibernate Types есть `StringArrayType`/`UUIDArrayType`, что снимает боль ручного парсинга. Но не путайте массивы с отношениями: если элементы — сущности, лучше нормализовать.

**Диапазоны** (`int4range`, `numrange`, `tstzrange`) позволяют в БД выразить интервальные ограничения: проверки пересечений, включения, соседства. Для расписаний и бронирований это мощный инструмент. Hibernate Types предоставляет тип `Range` и `PostgreSQLRangeType`, что позволяет сохранять/читать интервалы как Java-объекты и проверять пересечения на стороне SQL.

`hstore` — лёгкий кей-вэлью тип. Он экономичнее JSONB для «плоских» словарей строк→строк, также индексируется GIN, но лишён вложенности/типов. Подойдёт для «слабоструктурированных» атрибутов, которые нужно быстро фильтровать/искать по ключам.

Ключ к производительности всех этих типов — **индексы**. Для `jsonb` почти всегда нужен `GIN` (обычный или `jsonb_path_ops`) по колонке, иначе запросы по операторам превращаются во full scan. Для массивов — `GIN` по `col` или `col gin__int_ops`. Для диапазонов — `GiST` по колонке диапазона. Для `hstore` — `GIN`. Индексы следует проектировать под ваши операторы (`@>`/`?`/`&&`), иначе выгоды не будет.

Если вы выбираете `@Converter`, выигрываете в зависимости (нет внешней библиотеки), но платите ручным кодом сериализации/десериализации и отсутствием «нативных» типов на уровне ORM. Если берёте Hibernate Types, получаете много готового, но «привязываетесь» к конкретной библиотеке. На практике гибрид: «простые» поля — конвертером, сложные/массовые — Hibernate Types.

При проектировании схемы оцените частоту изменений поля: `jsonb` удобен, но **не злоупотребляйте** «схемой в поле». Поведенческие/ключевые атрибуты лучше вынести в колонки — это упростит индексацию, миграции и тесты. JSONB оставьте для «редких»/«дополнительных» свойств и логов.

Тестируйте операторные запросы интеграционно. H2 не понимает JSONB/массивы/диапазоны, даже «в PG-режиме». Для этой зоны используйте **Testcontainers** с реальным Postgres и накатывайте полноценные миграции Flyway/Liquibase, включая индексы и расширения (`CREATE EXTENSION`).

Не забывайте про **валидацию** JSON. Если поле — `jsonb`, убедитесь, что сериализация в `@Converter`/сервисе не пишет «битые» документы. Jackson прекрасно валидирует структуру при десериализации, так что декодируйте хотя бы в `JsonNode`, прежде чем записать.

**SQL (Flyway) — таблица с jsonb/массивами/диапазоном/hstore + индексы**

```sql
-- V10__complex_types.sql
create extension if not exists hstore;

create table products_ext (
    id           bigserial primary key,
    sku          text not null unique,
    attrs        jsonb not null default '{}'::jsonb,  -- произвольные атрибуты
    tags         text[] not null default '{}',        -- массив тегов
    active_range tstzrange,                           -- период активности
    meta         hstore                               -- плоский словарь
);

create index if not exists products_ext_attrs_gin on products_ext using gin (attrs jsonb_path_ops);
create index if not exists products_ext_tags_gin  on products_ext using gin (tags);
create index if not exists products_ext_range_gist on products_ext using gist (active_range);
create index if not exists products_ext_meta_gin  on products_ext using gin (meta);
```

**Java — маппинг через Hibernate Types и альтернативно через `@Converter`**

```java
package com.example.pgtypes;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.vladmihalcea.hibernate.type.array.StringArrayType;
import com.vladmihalcea.hibernate.type.json.JsonType;
import com.vladmihalcea.hibernate.type.range.Range;
import com.vladmihalcea.hibernate.type.range.PostgreSQLRangeType;
import com.vladmihalcea.hibernate.type.basic.PostgreSQLHStoreType;
import jakarta.persistence.*;
import org.hibernate.annotations.Type;
import org.hibernate.annotations.TypeDef;
import org.hibernate.annotations.TypeDefs;

import java.io.IOException;
import java.time.Instant;
import java.util.Map;

@Entity
@Table(name = "products_ext")
@TypeDefs({
    @TypeDef(name = "json", typeClass = JsonType.class),
    @TypeDef(name = "string-array", typeClass = StringArrayType.class),
    @TypeDef(name = "tsrange", typeClass = PostgreSQLRangeType.class),
    @TypeDef(name = "hstore", typeClass = PostgreSQLHStoreType.class)
})
public class ProductExt {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true) private String sku;

    // JSONB через Hibernate Types
    @Type(type = "json")
    @Column(columnDefinition = "jsonb", nullable = false)
    private JsonNode attrs;

    // Массив строк
    @Type(type = "string-array")
    @Column(columnDefinition = "text[]", nullable = false)
    private String[] tags = new String[0];

    // Диапазон времени
    @Type(type = "tsrange")
    @Column(name = "active_range", columnDefinition = "tstzrange")
    private Range<Instant> activeRange;

    // hstore как Map<String,String>
    @Type(type = "hstore")
    @Column(columnDefinition = "hstore")
    private Map<String, String> meta;

    protected ProductExt() {}
    public ProductExt(String sku, JsonNode attrs) { this.sku = sku; this.attrs = attrs; }

    // getters/setters ...
}

/** Альтернатива: JSONB через @Converter (String <-> JsonNode) */
@Converter(autoApply = false)
class JsonNodeConverter implements AttributeConverter<JsonNode, String> {
    private static final ObjectMapper MAPPER = new ObjectMapper();
    @Override public String convertToDatabaseColumn(JsonNode attribute) {
        return attribute == null ? "{}" : attribute.toString();
    }
    @Override public JsonNode convertToEntityAttribute(String dbData) {
        try { return dbData == null ? MAPPER.nullNode() : MAPPER.readTree(dbData); }
        catch (IOException e) { throw new IllegalArgumentException("Bad JSON", e); }
    }
}

@Entity
@Table(name = "products_ext_conv")
class ProductExtConv {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, unique = true) String sku;

    @Convert(converter = JsonNodeConverter.class)
    @Column(columnDefinition = "jsonb", nullable = false)
    JsonNode attrs;

    protected ProductExtConv() {}
    public ProductExtConv(String sku, JsonNode attrs){ this.sku = sku; this.attrs = attrs; }
}
```

**Kotlin — те же маппинги**

```kotlin
package com.example.pgtypes

import com.fasterxml.jackson.databind.JsonNode
import com.fasterxml.jackson.databind.ObjectMapper
import com.vladmihalcea.hibernate.type.array.StringArrayType
import com.vladmihalcea.hibernate.type.json.JsonType
import com.vladmihalcea.hibernate.type.range.PostgreSQLRangeType
import com.vladmihalcea.hibernate.type.range.Range
import com.vladmihalcea.hibernate.type.basic.PostgreSQLHStoreType
import jakarta.persistence.*
import org.hibernate.annotations.Type
import org.hibernate.annotations.TypeDef
import org.hibernate.annotations.TypeDefs
import java.time.Instant

@Entity
@Table(name = "products_ext")
@TypeDefs(
    TypeDef(name = "json", typeClass = JsonType::class),
    TypeDef(name = "string-array", typeClass = StringArrayType::class),
    TypeDef(name = "tsrange", typeClass = PostgreSQLRangeType::class),
    TypeDef(name = "hstore", typeClass = PostgreSQLHStoreType::class)
)
class ProductExt(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false, unique = true)
    var sku: String = "",

    @Type(type = "json")
    @Column(columnDefinition = "jsonb", nullable = false)
    var attrs: JsonNode? = null,

    @Type(type = "string-array")
    @Column(columnDefinition = "text[]", nullable = false)
    var tags: Array<String> = emptyArray(),

    @Type(type = "tsrange")
    @Column(name = "active_range", columnDefinition = "tstzrange")
    var activeRange: Range<Instant>? = null,

    @Type(type = "hstore")
    @Column(columnDefinition = "hstore")
    var meta: Map<String, String>? = null
)

@Converter(autoApply = false)
class JsonNodeConverter : AttributeConverter<JsonNode, String> {
    private val mapper = ObjectMapper()
    override fun convertToDatabaseColumn(attribute: JsonNode?): String =
        attribute?.toString() ?: "{}"

    override fun convertToEntityAttribute(dbData: String?): JsonNode =
        if (dbData == null) mapper.nullNode() else mapper.readTree(dbData)
}

@Entity
@Table(name = "products_ext_conv")
class ProductExtConv(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true)
    var sku: String = "",
    @Convert(converter = JsonNodeConverter::class)
    @Column(columnDefinition = "jsonb", nullable = false)
    var attrs: JsonNode? = null
)
```

**Пример выборок с операторами (native) и проекциями**
— это показывает, зачем нам индексы GIN/GiST и как извлекать значения из JSONB/`hstore`.

```java
package com.example.pgtypes;

import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.Instant;
import java.util.List;

public interface ProductRow {
    String getSku();
    String getBrand();
    String[] getTags();
    Instant getFrom(); Instant getTo();
}

@Repository
interface ProductExtRepository extends JpaRepository<ProductExt, Long> {

    // JSONB: фильтр по бренду в attrs {"brand":"..."} и по ключу в hstore meta
    @Query(value = """
        select p.sku as sku,
               p.attrs->>'brand' as brand,
               p.tags as tags,
               lower(p.active_range) as from,
               upper(p.active_range) as to
        from products_ext p
        where (p.attrs @> cast(:attrs as jsonb))
          and (p.meta ? :metaKey)
        order by p.sku
        """,
        countQuery = "select count(*) from products_ext p where (p.attrs @> cast(:attrs as jsonb)) and (p.meta ? :metaKey)",
        nativeQuery = true)
    Page<ProductRow> findByAttrsAndMeta(@Param("attrs") String attrsJson,
                                        @Param("metaKey") String metaKey,
                                        Pageable pageable);

    // Диапазоны: пересечение интервалов NOW..NOW+7d
    @Query(value = """
        select p.* from products_ext p
        where p.active_range && tstzrange(now(), now() + interval '7 days')
        """, nativeQuery = true)
    List<ProductExt> activeInNextWeek();
}
```

---

## Денежные/точные типы: `BigDecimal` + scale/rounding; контроль сериализации в JSON

Деньги — зона, где «плавающая точка» недопустима. В Java мы используем `BigDecimal` с фиксированной **масштабностью** (`scale`) и явным **округлением**. В Postgres — тип `numeric(precision, scale)` или, альтернативно, хранение суммы в **центах** как `bigint`. Первый вариант удобен для ад-хок SQL/агрегаций, второй — максимально быстрый и безошибочный при арифметике в приложении.

Если выбираете `numeric`, в JPA укажите `precision/scale` в `@Column`, а все операции производите через `BigDecimal` с контролируемым `RoundingMode`. Запрещено полагаться на дефолтное округление: оно может отличаться и породить копеечные рассогласования. Общая рекомендация: фиксируйте масштаб как **2** или **4** (валюта/измерения) и используйте `RoundingMode.HALF_UP` (или бизнес-правило).

Для сериализации в JSON важны два момента. Во-первых, не допускать **научной нотации** («1E+6»): включите `WRITE_BIGDECIMAL_AS_PLAIN` или используйте кастомный сериализатор, который форматирует число как строку. Во-вторых, избежать потери нулей масштаба при сериализации («10.00» vs «10»): задайте шаблон или свой сериализатор. Если вы храните суммы в **центах**, сериализуйте наружу как «деньги.две_десятичные» — и обратно в сущность принимайте строго две цифры.

Альтернативный паттерн — **value object Money** с валютой и суммой. Его можно хранить как `numeric(19,4)` (или `bigint` центов) через `@Embeddable` и `@AttributeOverrides`. Это упрощает инварианты: конструктор `Money` валидирует знак, масштаб и округление, а сервисам отдаёт готовые методы `add/subtract/multiply`.

В расчётах помните о **накоплении ошибок**: умножения/деления сначала выполняйте с «высоким» масштабом (например, 8), а результат **нормализуйте** к нужному масштабу в конце. Это снизит риск копеечной разницы на «длинных» формулах. В отчётах используйте SQL-агрегации по `numeric`, но финальный формат округляйте в приложении.

Если ваш фронт принимает суммы строками, валидируйте локаль: десятичный разделитель — точка, не запятая. На уровне DTO используйте `@Pattern` для формата и `@JsonCreator` для явного парсинга. Никогда не доверяйте «свободной строке» — это проблемы в проде и инциденты.

Для выгоды производительности и упрощения индексации многие команды выбирают **cents as BIGINT**. Тогда в БД на колонку «центов» ставятся обычные BTREE-индексы, а в приложении — тип `long`. Внешние API всё равно оперируют «денежными строками/BigDecimal» — а внутренняя запись выполняется без потерь и регрессов.

Тестируйте деньги особенно строго: «золотые» кейсы на добавление/умножение/распределение, и сравнение сериализованных значений. В CI держите тесты, что `10.00` не превращается в `10` и не выводится «1E+1».

**application.yml — контроль сериализации BigDecimal в Jackson**

```yaml
spring:
  jackson:
    serialization:
      write-bigdecimal-as-plain: true
```

**Java — два подхода: `numeric(19,4)` и «центы как BIGINT», плюс сериализация**

```java
package com.example.money;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.fasterxml.jackson.databind.ser.std.StdScalarSerializer;
import jakarta.persistence.*;
import jakarta.validation.constraints.NotNull;
import java.io.IOException;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.Instant;

@Entity
@Table(name = "invoices_money")
class InvoiceMoney {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @Column(name = "amount", nullable = false, precision = 19, scale = 4)
    BigDecimal amount; // numeric(19,4)

    @Column(nullable = false)
    Instant createdAt = Instant.now();

    protected InvoiceMoney() {}
    public InvoiceMoney(BigDecimal amount) {
        this.amount = normalize(amount);
    }

    public static BigDecimal normalize(BigDecimal src) {
        if (src == null) throw new IllegalArgumentException("amount");
        return src.setScale(4, RoundingMode.HALF_UP);
    }
}

@Entity
@Table(name = "invoices_cents")
class InvoiceCents {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @Column(name = "amount_cents", nullable = false)
    Long amountCents; // BIGINT cents

    @Column(nullable = false)
    Instant createdAt = Instant.now();

    protected InvoiceCents() {}
    public InvoiceCents(long cents){ this.amountCents = cents; }
}

/** Value Object для API: деньги с сериализацией как строка фиксированного масштаба */
class MoneyVO {
    @NotNull
    private final BigDecimal amount;

    public MoneyVO(BigDecimal amount) {
        this.amount = InvoiceMoney.normalize(amount);
    }
    @JsonSerialize(using = MoneySerializer.class)
    public BigDecimal getAmount() { return amount; }
}

class MoneySerializer extends StdScalarSerializer<BigDecimal> {
    protected MoneySerializer() { super(BigDecimal.class); }
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toPlainString());
    }
}
```

**Kotlin — аналогичные сущности и сериализация**

```kotlin
package com.example.money

import com.fasterxml.jackson.core.JsonGenerator
import com.fasterxml.jackson.databind.SerializerProvider
import com.fasterxml.jackson.databind.annotation.JsonSerialize
import com.fasterxml.jackson.databind.ser.std.StdScalarSerializer
import jakarta.persistence.*
import jakarta.validation.constraints.NotNull
import java.io.IOException
import java.math.BigDecimal
import java.math.RoundingMode
import java.time.Instant

@Entity
@Table(name = "invoices_money")
class InvoiceMoney(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(name = "amount", nullable = false, precision = 19, scale = 4)
    var amount: BigDecimal = BigDecimal.ZERO,

    @Column(nullable = false)
    var createdAt: Instant = Instant.now()
) {
    init { amount = amount.setScale(4, RoundingMode.HALF_UP) }
}

@Entity
@Table(name = "invoices_cents")
class InvoiceCents(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(name = "amount_cents", nullable = false)
    var amountCents: Long = 0L,

    @Column(nullable = false)
    var createdAt: Instant = Instant.now()
)

class MoneyVO(@field:NotNull private val amount: BigDecimal) {
    @JsonSerialize(using = MoneySerializer::class)
    fun getAmount(): BigDecimal = amount.setScale(4, RoundingMode.HALF_UP)
}

class MoneySerializer : StdScalarSerializer<BigDecimal>(BigDecimal::class.java) {
    override fun serialize(value: BigDecimal, gen: JsonGenerator, provider: SerializerProvider) {
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toPlainString())
    }
}
```

**SQL (Flyway) — варианты хранения денег**

```sql
-- V11__money.sql
create table invoices_money (
    id bigserial primary key,
    amount numeric(19,4) not null,
    created_at timestamptz not null default now()
);

create table invoices_cents (
    id bigserial primary key,
    amount_cents bigint not null,
    created_at timestamptz not null default now()
);
create index on invoices_cents (amount_cents);
```

---

## Частичные/функциональные индексы, `UNIQUE` под soft-delete, «виртуальные» столбцы/представления (`@Immutable`, `@Subselect`)

Частичные индексы — мощная фича PostgreSQL: индекс строится **по условию**, а не по всей таблице. Это идеальный инструмент для **soft-delete** и «активных» записей. Например, уникальность email только среди «живых» пользователей: `unique (lower(email)) where deleted_at is null`. Такой индекс меньше по размеру и не мешает создавать «дубликаты» среди удалённых записей.

Функциональные индексы позволяют индексировать результат функции: `lower(email)`, `coalesce(phone,'')`, выражения из JSONB (`(attrs->>'brand')`). Это снимает потребность в «служебных» колонках для сортировки/поиска без учёта регистра и ускоряет фильтры в `where`.

Под soft-delete на уровне ORM удобно использовать `@SQLDelete` (перехватывает `DELETE` и делает `UPDATE ... set deleted_at=now()`) и `@Where(clause = "deleted_at is null")` — чтобы все чтения по умолчанию скрывали удалённые записи. Но тогда уникальные ограничения через обычный `unique` не подойдут — нужен именно **частичный уникальный индекс**.

«Виртуальные» табличные представления через Hibernate — это `@Subselect`: сущность маппится не на таблицу, а на подзапрос/представление. Сочетайте с `@Immutable` (Hibernate) и, при необходимости, `@Synchronize` (список таблиц, от которых зависит подзапрос). Это удобный способ «собрать» материализованный отчетный вид без отдельного ETL, но помнить нужно: запись через такую сущность **нельзя** выполнять (только чтение).

Для стабильной работы `@Subselect` важно, чтобы подзапрос возвращал **уникальный первичный ключ**. Часто это агрегированные данные с `row_number()` или «готовый id» из базовой таблицы. Добавляйте индексы на опорные таблицы — `@Subselect` не делает SQL быстрее, он лишь меняет точку входа.

Частичные индексы и `@Where` требуют внимательности на миграциях. Если вы меняете условие «живости» (`deleted_at is null` → `deleted=false`), синхронизируйте и аннотации, и индексы. Иначе оптимизатор перестанет использовать индекс и производительность просядет.

В запросах учитывайте, что `@Where` автоматически дописывается к JPQL/Criteria. Если вам **нужно** прочитать удалённые, пишите **native** и указывайте условие сами или заводите отдельный репозиторий/метод без `@Where` (на отдельной сущности-проекции).

Функциональные индексы удобны и с JSONB: часто обращаемые атрибуты (например, `attrs->>'brand'`) можно вынести в функциональный индекс, не дублируя колонку. Но помните: такие индексы «ломаются» при смене ключа/структуры JSON — это технический долг, который надо осознанно обслуживать.

`@Immutable` на отчётных сущностях защищает от случайных модификаций в коде и уменьшает накладные расходы ORM, исключая dirty checking. Для стабильных представлений это бесплатная надёжность.

И, наконец, проверяйте планы (`explain analyze`) под частичные/функциональные индексы. Оптимизатор использует их, только если условие и функция **эквивалентны** объявлению индекса (например, `where lower(email)=?` — идеально; `where email ilike ?` — может уже понадобиться другой индекс или выражение).

**SQL (Flyway) — soft-delete, частичный UNIQUE, функциональные индексы и представление**

```sql
-- V12__indexes_and_views.sql
create table users_soft (
    id bigserial primary key,
    email text not null,
    name text not null,
    deleted_at timestamptz
);

-- Уникальность только среди «живых» (email сравниваем без регистра)
create unique index ux_users_soft_email_alive
    on users_soft (lower(email))
    where deleted_at is null;

-- Функциональный индекс по JSONB-ключу brand в products_ext.attrs
create index if not exists products_ext_brand_idx
    on products_ext ((attrs->>'brand'));

-- Виртуальный отчёт: суммы по sku, только активные
create or replace view v_sku_totals as
select p.sku,
       count(*) as cnt,
       sum( (p.attrs->>'price_cents')::bigint ) as total_cents
from products_ext p
where p.active_range @> now()
group by p.sku;
```

**Java — soft-delete через `@SQLDelete/@Where` и чтение `@Subselect`-представления**

```java
package com.example.indexes;

import jakarta.persistence.*;
import org.hibernate.annotations.Immutable;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;

@Entity
@Table(name = "users_soft")
@SQLDelete(sql = "update users_soft set deleted_at = now() where id = ?")
@Where(clause = "deleted_at is null")
public class UserSoft {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @Column(nullable = false) String email;
    @Column(nullable = false) String name;

    protected UserSoft() {}
    public UserSoft(String email, String name){ this.email=email; this.name=name; }
}

@Entity
@Immutable
@org.hibernate.annotations.Subselect("""
    select row_number() over ()::bigint as id, v.sku, v.cnt, v.total_cents
    from v_sku_totals v
""")
@org.hibernate.annotations.Synchronize({"products_ext"})
class SkuTotalsView {
    @Id Long id; // искусственный PK на основе row_number() — только чтение
    String sku;
    Long cnt;
    Long totalCents;
}

interface UserSoftRepository extends org.springframework.data.jpa.repository.JpaRepository<UserSoft, Long> {
    // Поиск «живых» пользователей — @Where добавится автоматически
    java.util.Optional<UserSoft> findByEmailIgnoreCase(String email);
}

interface SkuTotalsRepository extends org.springframework.data.jpa.repository.JpaRepository<SkuTotalsView, Long> {
    java.util.List<SkuTotalsView> findTop10ByOrderByTotalCentsDesc();
}
```

**Kotlin — те же приёмы**

```kotlin
package com.example.indexes

import jakarta.persistence.*
import org.hibernate.annotations.Immutable
import org.hibernate.annotations.SQLDelete
import org.hibernate.annotations.Where

@Entity
@Table(name = "users_soft")
@SQLDelete(sql = "update users_soft set deleted_at = now() where id = ?")
@Where(clause = "deleted_at is null")
class UserSoft(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var email: String = "",
    @Column(nullable = false) var name: String = ""
)

@Entity
@Immutable
@org.hibernate.annotations.Subselect(
    """
    select row_number() over ()::bigint as id, v.sku, v.cnt, v.total_cents
    from v_sku_totals v
    """
)
@org.hibernate.annotations.Synchronize("products_ext")
class SkuTotalsView(
    @Id var id: Long? = null,
    var sku: String? = null,
    var cnt: Long? = null,
    var totalCents: Long? = null
)

interface UserSoftRepository : org.springframework.data.jpa.repository.JpaRepository<UserSoft, Long> {
    fun findByEmailIgnoreCase(email: String): java.util.Optional<UserSoft>
}

interface SkuTotalsRepository : org.springframework.data.jpa.repository.JpaRepository<SkuTotalsView, Long> {
    fun findTop10ByOrderByTotalCentsDesc(): List<SkuTotalsView>
}
```

**Советы эксплуатации и тестирования (короткое резюме)**
Держите миграции с расширениями/индексами рядом с маппингами; покрывайте интеграционными тестами (Testcontainers) операторы и индексы; не злоупотребляйте JSONB — выносите «горячие» поля в колонки; для денег фиксируйте масштаб/округление и формат JSON; для soft-delete используйте частичный UNIQUE и не забывайте, что `@Where` скрывает записи — отдельные методы читайте native, если нужно «видеть всё».

# 10. Аудит и «мягкое удаление»

## Auditing: `@CreatedDate/@LastModifiedDate`, `AuditingEntityListener`, пользователи/тенанты

Аудит — это системный слой, который фиксирует «кто и когда» создал/изменил запись. Он не про бизнес-логику как таковую, а про трассируемость и соответствие организационным требованиям. В Spring Data JPA аудит реализуется почти «из коробки» через аннотации `@CreatedDate`, `@LastModifiedDate`, а также `@CreatedBy` и `@LastModifiedBy`. Эти поля заполняются инфраструктурой автоматически, но для этого нам нужно включить аудит и предоставить **источники текущего пользователя и (если нужно) текущего тенанта**.

Ключевая деталь: аннотации сами по себе ничего не сделают без слушателя жизненного цикла `AuditingEntityListener`. Он подписывается на `@PrePersist` и `@PreUpdate` и перед коммитом расставляет метки времени и пользователей. Spring включает этот слушатель через `@EnableJpaAuditing`; под капотом регистрируется `AuditingHandler`, который использует `AuditorAware<T>` — наш боб, возвращающий «кто сейчас действует». Тип `T` — ваш выбор: имя пользователя (`String`), id (`Long`) или доменный объект.

Источником «кто» часто выступает `SecurityContextHolder`. Но в микросервисах не всегда используется Spring Security; иногда контекст приходит в заголовках (например, `X-User-Id`, `X-Tenant-Id`) и складывается в ThreadLocal. Важно, чтобы `AuditorAware` был **детерминированным**: для фоновых задач и миграций возвращайте «system»/`0`, чтобы отличать машинные операции от пользовательских.

Тенант — это «логическая организация» данных: в многотенантных системах мы храним `tenantId` в каждой строке. Аудит и тенант переплетены: одни компании хотят видеть «кто из их пользователей» изменил запись, другие — только сервисные аккаунты. Хорошая практика — хранить оба: `createdBy`, `lastModifiedBy`, `tenantId`. Причём `tenantId` — это **часть инварианта**: он должен выставляться всегда и не меняться. Для него не используются `@CreatedBy`, а обычная колонка + код на входе транзакции (фильтр, интерцептор).

Бонус аудита — **обратимость**: мы можем быстро разобраться, почему запись в таком состоянии. Когда инцидент уже случился, поля аудита сужают круг поиска (какой сервис, чья авторизация, из какой зоны). Это особенно важно в распределённых системах, где одно изменение может пройти через несколько слоёв.

Рассинхронизация дат — типичная ловушка. Если приложение не работает в UTC, а база — да, вы получите «плавающие» таймстемпы. Вывод прост: придерживайтесь **UTC-везде**, а форматирование в локальное время — ответственность UI. В Spring укажите `spring.jpa.properties.hibernate.jdbc.time_zone: UTC`.

Стоит обсудить и «вставки с прошлой датой». Иногда нужно импортировать исторические данные; `@CreatedDate` перезапишет их «сейчас». Решение — либо временно отключать аудит на этот импорт (отдельный профиль), либо явно задавать поле и запретить слушателю затирать не-null значения. Такой «опт-аут» можно реализовать через маркеры в `AuditorAware` или флаги в `TransactionSynchronizationManager`.

Где хранить аудитные поля? В каждой сущности, где важна трассируемость. Не обязательно везде. Для справочников «страны/валюты» часто достаточно `@Immutable` и одной даты создания; для операций денег — полный набор. Подумайте о производительности индексов: `createdAt` — хороший кандидат для сортировки и TTL-архивирования (партционирование по дате).

И наконец: **тестируйте аудит**. В slice-тестах `@DataJpaTest` задайте фейковый `AuditorAware`, создайте/обновите сущность и убедитесь, что поля расставлены. Это дешёво и даёт уверенность, что прод-инцидент «кто поменял сумму» вы разберёте по полям, а не по логам.

**Gradle (Groovy DSL)**

```groovy
plugins {
  id 'org.springframework.boot' version '3.3.4'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'java'
}
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.springframework.boot:spring-boot-starter-security' // если берете из SecurityContext
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
  implementation("org.springframework.boot:spring-boot-starter-security")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**application.yml — единая тайм-зона и выключенный OSIV**

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc.time_zone: UTC
```

**Java — включение аудита, текущий пользователь/тенант и сущность с полями аудита**

```java
package com.example.audit;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.context.annotation.Bean;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Optional;

@Configuration
@EnableJpaAuditing
public class AuditConfig {

    @Bean
    AuditorAware<String> auditorAware() {
        return () -> {
            var ctx = SecurityContextHolder.getContext();
            if (ctx != null && ctx.getAuthentication() != null && ctx.getAuthentication().isAuthenticated()) {
                return Optional.ofNullable(ctx.getAuthentication().getName());
            }
            return Optional.of("system");
        };
    }
}
```

```java
package com.example.audit;

import jakarta.persistence.*;
import org.springframework.data.annotation.*;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;

@Entity
@EntityListeners(AuditingEntityListener.class)
@Table(name = "orders_aud")
public class OrderAud {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false) private Long customerId;
    @Column(nullable = false) private Long totalCents;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(nullable = false, updatable = false, length = 120)
    private String createdBy;

    @LastModifiedBy
    @Column(nullable = false, length = 120)
    private String lastModifiedBy;

    @Column(nullable = false, length = 64)
    private String tenantId; // выставляйте через сервис/фильтр

    protected OrderAud() {}
    public OrderAud(Long customerId, Long totalCents, String tenantId) {
        this.customerId = customerId; this.totalCents = totalCents; this.tenantId = tenantId;
    }
    // getters/setters …
}
```

**Kotlin — включение аудита и сущность**

```kotlin
package com.example.audit

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.domain.AuditorAware
import org.springframework.data.jpa.repository.config.EnableJpaAuditing
import org.springframework.security.core.context.SecurityContextHolder
import java.util.*

@Configuration
@EnableJpaAuditing
class AuditConfig {

    @Bean
    fun auditorAware(): AuditorAware<String> = AuditorAware {
        val auth = SecurityContextHolder.getContext()?.authentication
        Optional.ofNullable(if (auth != null && auth.isAuthenticated) auth.name else "system")
    }
}
```

```kotlin
package com.example.audit

import jakarta.persistence.*
import org.springframework.data.annotation.CreatedBy
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedBy
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import java.time.Instant

@Entity
@Table(name = "orders_aud")
@EntityListeners(AuditingEntityListener::class)
class OrderAud(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    @Column(nullable = false) var customerId: Long = 0,
    @Column(nullable = false) var totalCents: Long = 0,

    @CreatedDate @Column(nullable = false, updatable = false)
    var createdAt: Instant? = null,

    @LastModifiedDate @Column(nullable = false)
    var updatedAt: Instant? = null,

    @CreatedBy @Column(nullable = false, updatable = false, length = 120)
    var createdBy: String? = null,

    @LastModifiedBy @Column(nullable = false, length = 120)
    var lastModifiedBy: String? = null,

    @Column(nullable = false, length = 64)
    var tenantId: String = ""
)
```

---

## Soft-delete: флаг + глобальные фильтры (`@Where/@SQLDelete` или фильтры Hibernate), последствия для уникальных ключей/джойнов

«Мягкое удаление» — это практика не удалять строки физически, а помечать их признаком «удалено» (флагом `deleted` или `deleted_at`). Плюсы: можно восстановить, можно хранить историю ссылок и не получать «висячие» FK. Минусы: усложняется уникальность и запросы «видеть всё». В JPA самый удобный приём — комбинация `@SQLDelete` (перехватывает `DELETE` и делает `UPDATE ...`) и `@Where` (по умолчанию прячет удалённые строки из всех запросов ORM).

`@SQLDelete` гарантирует, что даже вызов `repo.delete(entity)` не приведёт к физическому удалению: Hibernate выполнит SQL, который вы указали. Часто это `update table set deleted_at = now() where id = ?`. Вместо флага времени можно использовать boolean-колонку `deleted = true`, но временная метка удобнее для аудита и TTL-архивирования.

`@Where(clause = "deleted_at is null")` добавляет условие во все JPQL/Criteria-запросы к сущности и её коллекциям. Это «глобальный фильтр»: никто в команде не забудет дописать `where ... is null`, ORM сделает это сам. Цена — помнить, что иногда вам **нужно** увидеть удалённые; тогда используйте native SQL или отдельную сущность-проекцию без `@Where`.

Проблема уникальности: обычный `UNIQUE(email)` не позволит создать «такого же» пользователя после удаления, потому что старая строка всё ещё лежит в таблице. Решение на стороне Postgres — **частичный уникальный индекс**: `unique (lower(email)) where deleted_at is null`. Он обеспечивает уникальность только среди «живых» записей. Этот паттерн критичен для e-mail/логинов/код-значений.

Джойны и `@Where` — источник неожиданных эффектов. В `INNER JOIN` на «правую» сущность `@Where` исчезающе прозрачно «съест» удалённые строки — и это, как правило, хорошо. Но в `LEFT JOIN` вы тоже не увидите «удалённого» правого края, хотя логически могли ожидать `NULL`-колонки. Документируйте это поведение для аналитиков и отчётов.

Альтернатива `@Where` — **фильтры Hibernate** (`@FilterDef` + `@Filter`). Преимущество фильтра — его можно включать/выключать на сессии: `session.enableFilter("alive")`. Это гибко, если вам нужны разные политики видимости в разных частях системы («оператор видит всё», «клиент — только живое»). Недостаток — надо не забыть включить фильтр каждой транзакции (или сделать перехватчик, который включит его автоматически).

Учтите каскады. Если у вас `@OneToMany` на «детей» с `orphanRemoval = true`, вызов `parent.getChildren().remove(x)` вызовет удаление **строки детей**. При soft-delete это может быть не то, что вы хотели. Используйте отдельные методы `trash()` на детях и избегайте «неявной» семантики orphan removal для soft-удалений.

И ещё про FK: если вы оставляете FK «как есть», soft-delete родителя будет блокироваться из-за детей (если ON DELETE не настроен). Правильно — либо запретить удаление (требуется ручная очистка), либо перевести FK в `ON DELETE SET NULL` (когда бизнес это допускает), либо каскадно «мягко удалять» детей.

Наконец, мониторинг. Мягко удалённые записи — это «грязь», растущая бесконечно. Нужны фоновый TTL-процесс (псевдоархив) и метрики: сколько процентов таблицы «мёртвое». Если вы используете партиции по дате — переносите партиции в холодное хранилище и физически очищайте по SLA.

**SQL (Flyway) — схема soft-delete и частичный UNIQUE**

```sql
-- V20__users_soft.sql
create table users_soft (
  id bigserial primary key,
  email text not null,
  name text not null,
  deleted_at timestamptz
);

-- Уникальность среди «живых»
create unique index ux_users_soft_email_alive
  on users_soft (lower(email))
  where deleted_at is null;
```

**Java — сущность с `@SQLDelete/@Where` и опциональным фильтром**

```java
package com.example.soft;

import jakarta.persistence.*;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;
import org.hibernate.annotations.FilterDef;
import org.hibernate.annotations.Filter;

@Entity
@Table(name = "users_soft")
@SQLDelete(sql = "update users_soft set deleted_at = now() where id = ?")
@Where(clause = "deleted_at is null")
@FilterDef(name = "alive") // без параметров
@Filter(name = "alive", condition = "deleted_at is null")
public class UserSoft {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false) private String name;

    @Column(nullable = false) private String email;

    @Column private java.time.Instant deleted_at;

    protected UserSoft() {}
    public UserSoft(String name, String email){ this.name = name; this.email = email; }
    // getters/setters …
}
```

```java
package com.example.soft;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.Session;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserSoftService {
    @PersistenceContext private EntityManager em;

    @Transactional
    public void removeSoft(Long id) {
        UserSoft u = em.find(UserSoft.class, id);
        em.remove(u); // выполнится UPDATE через @SQLDelete
    }

    @Transactional(readOnly = true)
    public java.util.List<UserSoft> findAliveWithFilter() {
        Session session = em.unwrap(Session.class);
        session.enableFilter("alive");
        try {
            return em.createQuery("select u from UserSoft u order by u.id", UserSoft.class)
                     .getResultList();
        } finally {
            session.disableFilter("alive");
        }
    }
}
```

**Kotlin — тот же приём**

```kotlin
package com.example.soft

import jakarta.persistence.*
import org.hibernate.annotations.Filter
import org.hibernate.annotations.FilterDef
import org.hibernate.annotations.SQLDelete
import org.hibernate.annotations.Where
import org.hibernate.Session
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "users_soft")
@SQLDelete(sql = "update users_soft set deleted_at = now() where id = ?")
@Where(clause = "deleted_at is null")
@FilterDef(name = "alive")
@Filter(name = "alive", condition = "deleted_at is null")
class UserSoft(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = "",
    @Column(nullable = false) var email: String = "",
    var deleted_at: java.time.Instant? = null
)

@Service
class UserSoftService(
    @PersistenceContext private val em: EntityManager
) {
    @Transactional
    fun removeSoft(id: Long) {
        val u = em.find(UserSoft::class.java, id)
        em.remove(u)
    }

    @Transactional(readOnly = true)
    fun findAliveWithFilter(): List<UserSoft> {
        val session = em.unwrap(Session::class.java)
        session.enableFilter("alive")
        return try {
            em.createQuery("select u from UserSoft u order by u.id", UserSoft::class.java).resultList
        } finally {
            session.disableFilter("alive")
        }
    }
}
```

---

## История изменений: версионирование записей vs отдельные audit-таблицы; компромисс сложность/польза

Аудитные поля отвечают на вопросы «кто/когда», но не показывают «что именно поменялось». Для этого есть два направления: **версионирование** (храним все версии записи) и **аудит-журнал** (отдельная таблица событий изменений). Версионирование удобно для «срезов на момент времени» («покажи состояние заказа на вчера 12:00»), а журнал — для лаконичного ответа «старое значение → новое значение» и последующего анализа.

На стороне Hibernate готовое решение — **Envers**. Он перехватывает INSERT/UPDATE/DELETE и записывает снимки сущности в таблицы `*_AUD` вместе с номером «ревизии» и метаданными (время, пользователь, тип операции). Достоинства: минимум кода, поддержка сложных графов, запросы по «состоянию на ревизии». Недостатки: объём данных растёт быстро, сложнее выборки для специфических отчётов.

Альтернатива — **свой журнал**: отдельная таблица `audit_log` с полями `entity_name`, `entity_id`, `event`, `old_data`, `new_data`, `actor`, `at`. Заполняется слушателем или сервисом. Плюс — полный контроль (например, можно логировать только ключевые поля, а не всю сущность). Минус — больше кода и риск «забыть» залогировать.

Компромисс в реальных системах часто такой: для «критичных» сущностей (деньги, заказы) — Envers или «снимки», для остальных — только поля аудита или компактный журнал «изменённые ключевые поля». Ещё один критерий — «воспроизводимость»: если вам нужен отчёт «как видел клиент в тот день», храните версии. Если нужен «кто поменял поле X» — хватит журнала.

Учитывайте GDPR/retention. История изменений — это тоже персональные данные. Нужны политики хранения: сколько времени держим, как анонимизируем. Envers поддерживает «очистку ревизий», но чаще это реализуют фоновыми задачами/архивами.

С производительности стороны любой аудит добавляет I/O на запись. Envers вешает триггеры на ORM-уровне; если вы делаете `bulk update` (см. предыдущие подтемы), Envers его **не увидит** — придётся логировать вручную или через триггеры БД. Это важный нюанс при массовых операциях.

Тестируйте историю. Заводите интеграционный тест, который делает: create → update → delete, а затем читает ревизии и проверяет ожидаемые состояния. Так вы зафиксируете контракт «какие поля аудируются» и убережётесь от регрессов после миграций.

Наконец, не путайте версионирование для аудита и `@Version` для оптимистической блокировки. Это разные механизмы: первое — хранение прошлых состояний, второе — защита от конфликтов записи. Они могут жить вместе: Envers создаёт свои `_AUD`, а `@Version` остаётся в основной таблице.

**Gradle — добавить Hibernate Envers**

*Groovy DSL*

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.hibernate.orm:hibernate-envers' // версия подтянется из BOM Spring Boot
  runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL*

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.hibernate.orm:hibernate-envers")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**Java — сущность с `@Audited` и чтение ревизий через Envers**

```java
package com.example.envers;

import jakarta.persistence.*;
import org.hibernate.envers.Audited;
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Entity
@Table(name = "products_a")
@Audited
public class ProductA {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false) private String sku;
    @Column(nullable = false) private Long priceCents;
    protected ProductA() {}
    public ProductA(String sku, Long priceCents){ this.sku=sku; this.priceCents=priceCents; }
    // getters/setters …
}

interface ProductARepository extends org.springframework.data.jpa.repository.JpaRepository<ProductA, Long> {}

@Service
public class ProductAuditService {
    private final ProductARepository repo;
    @jakarta.persistence.PersistenceContext private jakarta.persistence.EntityManager em;
    public ProductAuditService(ProductARepository repo){ this.repo = repo; }

    @Transactional
    public ProductA changePrice(Long id, long newPrice) {
        ProductA p = repo.findById(id).orElseThrow();
        p.setPriceCents(newPrice);
        return p;
    }

    @Transactional(readOnly = true)
    public List<Number> revisions(Long id) {
        AuditReader reader = AuditReaderFactory.get(em);
        return reader.getRevisions(ProductA.class, id);
    }

    @Transactional(readOnly = true)
    public ProductA stateAtRevision(Long id, Number rev) {
        AuditReader reader = AuditReaderFactory.get(em);
        return reader.find(ProductA.class, id, rev);
    }
}
```

**Kotlin — `@Audited` и чтение ревизий**

```kotlin
package com.example.envers

import jakarta.persistence.*
import org.hibernate.envers.Audited
import org.hibernate.envers.AuditReaderFactory
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "products_a")
@Audited
class ProductA(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var sku: String = "",
    @Column(nullable = false) var priceCents: Long = 0
)

interface ProductARepository : JpaRepository<ProductA, Long>

@Service
class ProductAuditService(
    private val repo: ProductARepository,
    @PersistenceContext private val em: EntityManager
) {

    @Transactional
    fun changePrice(id: Long, newPrice: Long): ProductA {
        val p = repo.findById(id).orElseThrow()
        p.priceCents = newPrice
        return p
    }

    @Transactional(readOnly = true)
    fun revisions(id: Long): List<Number> =
        AuditReaderFactory.get(em).getRevisions(ProductA::class.java, id)

    @Transactional(readOnly = true)
    fun stateAtRevision(id: Long, rev: Number): ProductA? =
        AuditReaderFactory.get(em).find(ProductA::class.java, id, rev)
}
```

**Альтернатива — простой «ручной» журнал изменений (идея)**

```java
package com.example.auditlog;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "audit_log")
public class AuditLog {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String entityName;
    @Column(nullable = false) String entityId;
    @Column(nullable = false) String event;   // CREATED/UPDATED/DELETED
    @Column(columnDefinition = "jsonb") String oldData;
    @Column(columnDefinition = "jsonb") String newData;
    @Column(nullable = false) String actor;
    @Column(nullable = false) Instant at = Instant.now();
    protected AuditLog() {}
    public AuditLog(String e, String id, String ev, String o, String n, String a){
        this.entityName=e; this.entityId=id; this.event=ev; this.oldData=o; this.newData=n; this.actor=a;
    }
}
```

Этот «ручной» журнал можно заполнять в сервисе перед изменением: сериализовать «до/после» ключевых полей и сохранить `AuditLog`. Простоты ради здесь JSON хранится как текст; с Postgres разумно задать `jsonb` и индекс GIN, если вы планируете сложные отчёты.

**Вывод по аудиту/soft-delete/истории изменений**

1. «Кто/когда» — включайте Spring Data Auditing с `AuditingEntityListener` и дисциплинированным `AuditorAware`.
2. Soft-delete — делайте через `@SQLDelete + @Where` или через фильтры Hibernate; для уникальности — частичный UNIQUE. Проговаривайте поведение джойнов.
3. История изменений — Envers для «срезов состояний», кастомный журнал для лаконичных событий. Обязательно учтите retention, массовые операции и тесты «create→update→delete→read history».

# 11. Спецификации, Criteria и QueryDSL

## `Specification<T>`: динамические фильтры, композиция условий, пагинация+сортировка

Подход на базе `Specification<T>` хорош там, где количество фильтров и их комбинаций растёт вместе с бизнес-требованиями. В отличие от строкового JPQL, вы собираете запрос из маленьких, переиспользуемых «кирпичиков»: каждый `Specification` отвечает за одно условие, а композиция через `and/or/where` даёт итоговый предикат. Такой стиль делает код декларативным: сервис говорит «хочу заказы статуса X, от даты Y, с суммой больше Z», а детали выразятся в отдельном наборе функций.

Важная особенность `Specification` — она знает о JPA-модели: в лямбде вы получаете `Root<T>` (корень запроса), `CriteriaQuery<?>` и `CriteriaBuilder`. Это позволяет не только строить `where`, но и добавлять `fetch`/`join` или менять `distinct`. Использовать fetch в спеках нужно осторожно: при пагинации (`Pageable`) коллекционные `fetch join` ломают подсчёт `count` и стабилизацию строк; для «карточек» конкретной сущности fetch уместен, а для списков лучше обойтись проекциями.

Композиция условий — сильная сторона Specification. Пусть у вас 10 опциональных полей фильтра. Вместо сложных `if` в одном запросе вы создаёте 10 мини-спеков: «по статусу», «по дате от», «по сумме», «по текстовому поиску» и т.д. Затем складываете их в цепочку `where(spec1).and(spec2)...`. Спеки легко тестируются по отдельности; ошибки локализуются быстрее.

Пагинация и сортировка в Spring Data сочетаются с спеками «из коробки»: метод `findAll(Specification, Pageable)` вернёт `Page<T>` и корректно применит `Sort`. Важно помнить о стабильной сортировке: если сортируете по полю, где возможны дубликаты (например, `createdAt`), добавьте вторичный ключ (обычно `id`) — иначе строки с одинаковыми значениями могут «прыгать» между страницами.

Списки лучше отдавать **тонкими DTO** — это снижает нагрузку на ORM и БД. Спеки используют сущности как исходные данные, но наружу вы маппите только нужные поля. Для больших страниц это ключ к производительности: нет N+1, нет массивных графов, меньше JSON. Если нужен подсчёт «итого», делайте отдельный агрегатный запрос (спека тоже может помочь), не пытайтесь «как-нибудь» прикрутить его к списку.

Взаимодействие со связями легко выражается в спеках через `join`. Типично: фильтровать заказы по городу покупателя или тегам товара. Но не забывайте индексы: любое условие по «правой» таблице будет эффективным только при наличии селективных индексов (например, `customer.city`, `order.customer_id`). Спека — это не волшебство; она лишь декларация для Criteria.

Ещё один совет — делайте «чистые» спеки. Пусть каждая возвращает только `Predicate`, не трогая `select`, `groupBy` и `orderBy`. Тогда вы свободно комбинируете их в любых сценариях. Если всё же нужен `fetch` (например, для карточки), вынесите его в отдельную спеку «для карточки», и не используйте её вместе с пагинацией.

Обрабатывайте пустые значения входного фильтра. Хорошая спека возвращает `cb.conjunction()` (истину) или `null`, когда критерий не задан. Так композиция не «ломается», а итоговый запрос остаётся корректным. Но следите, чтобы вы не возвращали `null` из всего билдера — используйте `Specification.where(...)`.

С точки зрения тестов удобно проверять содержание SQL (через логгер) и корректность результатов на маленьких фикстурах. Для сложных предикатов (например, поиск по нескольким словам в `LIKE`) заведите уголковые кейсы: пустая строка, спецсимволы, регистр.

И, наконец, помните про «границы ПК» (`Persistence Context`). Если вы собираете крупные страницы сущностей, не комбинируйте это с записью в той же транзакции: PC раздуется. Для листингов лучше `readOnly`-транзакции и проекции; для команд — отдельные методы.

**Gradle (Groovy DSL) — зависимости для спецификаций**

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
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
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
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
```

**Java — сущности, репозиторий со спеками и билдер условий**

```java
package com.example.spec;

import jakarta.persistence.*;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.data.jpa.repository.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.domain.*;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

@Entity
@Table(name = "customers")
class Customer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String name;
    @Column(nullable = false) String city;
    protected Customer() {}
    Customer(String name, String city){ this.name=name; this.city=city; }
}

@Entity
@Table(name = "orders")
class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "customer_id", nullable = false)
    Customer customer;
    @Column(nullable = false) String status;          // NEW, PAID, SHIPPED...
    @Column(nullable = false) Long totalCents;
    @Column(nullable = false) Instant createdAt = Instant.now();
    protected Order() {}
    Order(Customer c, String status, long total){ this.customer=c; this.status=status; this.totalCents=total; }
}

interface OrderRepository extends JpaRepository<Order, Long>, JpaSpecificationExecutor<Order> { }

record OrderFilter(String status, String city, Instant from, Instant to, Long minTotal, String q) {}

final class OrderSpecs {
    static Specification<Order> hasStatus(String status) {
        return (root, cq, cb) -> status == null ? cb.conjunction() : cb.equal(root.get("status"), status);
    }
    static Specification<Order> customerCity(String city) {
        return (root, cq, cb) -> {
            if (city == null || city.isBlank()) return cb.conjunction();
            var join = root.join("customer");
            return cb.equal(cb.lower(join.get("city")), city.toLowerCase());
        };
    }
    static Specification<Order> createdFrom(Instant from) {
        return (root, cq, cb) -> from == null ? cb.conjunction() : cb.greaterThanOrEqualTo(root.get("createdAt"), from);
    }
    static Specification<Order> createdTo(Instant to) {
        return (root, cq, cb) -> to == null ? cb.conjunction() : cb.lessThan(root.get("createdAt"), to);
    }
    static Specification<Order> minTotal(Long minTotal) {
        return (root, cq, cb) -> minTotal == null ? cb.conjunction() : cb.ge(root.get("totalCents"), minTotal);
    }
    static Specification<Order> query(String q) {
        return (root, cq, cb) -> {
            if (q == null || q.isBlank()) return cb.conjunction();
            var like = "%" + q.toLowerCase().trim() + "%";
            var join = root.join("customer");
            return cb.or(
                cb.like(cb.lower(join.get("name")), like),
                cb.like(cb.lower(root.get("status")), like)
            );
        };
    }
    static Specification<Order> fromFilter(OrderFilter f) {
        return Specification.where(hasStatus(f.status()))
                .and(customerCity(f.city()))
                .and(createdFrom(f.from()))
                .and(createdTo(f.to()))
                .and(minTotal(f.minTotal()))
                .and(query(f.q()));
    }
}

@Service
class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public Page<Order> find(OrderFilter f, int page, int size) {
        Sort sort = Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"));
        Pageable pageable = PageRequest.of(page, size, sort);
        return repo.findAll(OrderSpecs.fromFilter(f), pageable);
    }
}
```

**Kotlin — те же сущности/спеки и сервис**

```kotlin
package com.example.spec

import jakarta.persistence.*
import org.springframework.data.jpa.domain.Specification
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.JpaSpecificationExecutor
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Pageable
import org.springframework.data.domain.Sort
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

@Entity
@Table(name = "customers")
class Customer(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = "",
    @Column(nullable = false) var city: String = ""
)

@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "customer_id", nullable = false)
    var customer: Customer? = null,
    @Column(nullable = false) var status: String = "",
    @Column(nullable = false) var totalCents: Long = 0,
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

interface OrderRepository : JpaRepository<Order, Long>, JpaSpecificationExecutor<Order>

data class OrderFilter(
    val status: String? = null,
    val city: String? = null,
    val from: Instant? = null,
    val to: Instant? = null,
    val minTotal: Long? = null,
    val q: String? = null
)

object OrderSpecs {
    fun hasStatus(status: String?) = Specification<Order> { root, _, cb ->
        if (status.isNullOrBlank()) cb.conjunction() else cb.equal(root.get<String>("status"), status)
    }
    fun customerCity(city: String?) = Specification<Order> { root, _, cb ->
        if (city.isNullOrBlank()) cb.conjunction()
        else cb.equal(cb.lower(root.join<Any, Any>("customer").get("city")), city.lowercase())
    }
    fun createdFrom(from: Instant?) = Specification<Order> { root, _, cb ->
        if (from == null) cb.conjunction() else cb.greaterThanOrEqualTo(root.get("createdAt"), from)
    }
    fun createdTo(to: Instant?) = Specification<Order> { root, _, cb ->
        if (to == null) cb.conjunction() else cb.lessThan(root.get("createdAt"), to)
    }
    fun minTotal(min: Long?) = Specification<Order> { root, _, cb ->
        if (min == null) cb.conjunction() else cb.ge(root.get("totalCents"), min)
    }
    fun query(q: String?) = Specification<Order> { root, _, cb ->
        if (q.isNullOrBlank()) cb.conjunction() else run {
            val like = "%${q.trim().lowercase()}%"
            val c = root.join<Any, Any>("customer")
            cb.or(
                cb.like(cb.lower(c.get("name")), like),
                cb.like(cb.lower(root.get("status")), like)
            )
        }
    }
    fun fromFilter(f: OrderFilter) =
        Specification.where(hasStatus(f.status))
            ?.and(customerCity(f.city))
            ?.and(createdFrom(f.from))
            ?.and(createdTo(f.to))
            ?.and(minTotal(f.minTotal))
            ?.and(query(f.q))
}

@Service
class OrderService(private val repo: OrderRepository) {

    @Transactional(readOnly = true)
    fun find(f: OrderFilter, page: Int, size: Int): Page<Order> {
        val sort = Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"))
        val pageable: Pageable = PageRequest.of(page, size, sort)
        return repo.findAll(OrderSpecs.fromFilter(f)!!, pageable)
    }
}
```

---

## Criteria API: типобезопасность vs шум кода — где помогает

Criteria API — низкоуровневый, но мощный способ формировать запросы программно. В отличие от `Specification`, вы управляете *всем* запросом: `select`, `where`, `order by`, `group by`, конструктор-проекции, `distinct`. Это полезно для отчётных запросов, агрегатов, сложных вычислений и тех мест, где хочется типобезопасности без строкового JPQL. Цена — «шум кода» и бойлерплейт: билдер, корни, джоины, список предикатов, отдельный `count` для пагинации.

С типобезопасностью всё неоднозначно. «Из коробки» вы пишете `root.get("status")` — строка, не типобезопасно. Чтобы получить настоящую типобезопасность, используйте **статический метамодел** (классы `Order_`, `Customer_`) — их генерирует аннотационный процессор `hibernate-jpamodelgen`. Тогда вы пишете `root.get(Order_.status)` и компилятор ловит опечатки. Минус — нужно настроить `annotationProcessor` и держать генерацию в сборке.

Criteria удобен для **проекций**: `cb.construct(Dto.class, ...)` позволяет вернуть DTO без промежуточной сущности. Это экономит PC и делает список «тонким». Часто это лучший выбор для страниц-таблиц с десятками тысяч строк/час: вы не тянете графы, не держите сущности управляемыми, формируете JSON мгновенно.

Пагинация требует двух запросов: основной и `count`. Если у вас `distinct` или `join` на «многие», важно аккуратно считать `count` — либо по подзапросу, либо с `countDistinct`. В Hibernate 6 лучше явно строить отдельный `CriteriaQuery<Long>` для счётчика, повторяя `where`-часть.

Критерии легко выражают условные фильтры: набираете `List<Predicate>`, добавляете по мере наличия параметров, потом `query.where(cb.and(preds...))`. Это удобнее, чем «склеивать» строки JPQL. При этом сохраняется контроль над `order by`: можно применять сортировку из `Pageable` вручную, преобразовав `Sort.Order` в `Order` (Criteria).

Если нужны агрегаты — `cb.sum`, `cb.count`, `cb.avg` и `groupBy`/`having`. Критерии становятся особенно полезными, когда отчёт меняется «по кнопке» (добавился новый столбец, разрез) — вы программно собираете разные формы запросов без риска SQL-инъекций и опечаток.

Но не всё стоит делать Criteria. Для простых `findBy...` репозитории Spring Data и спеки короче и читаемее. Для действительно сложных фильтров и подзапросов QueryDSL (см. следующий пункт) нередко даёт более чистый код. Считайте Criteria «интермедиатным» инструментом: мощно, но шумно.

Производительность определяется не критерием, а качеством SQL. Проверяйте планы, ставьте индексы, следите за `distinct`. Не используйте коллекционные fetch-join в страницах: либо «карточка по id», либо проекции.

И ещё — транзакции. Критерии — просто запрос; все правила (readOnly, fetchSize, L1-кэш) применимы. Возвращайте DTO, а не сущности, если это таблица.

**Gradle — добавить генерацию метамодели (по желанию)**

*Groovy DSL*

```groovy
dependencies {
    annotationProcessor 'org.hibernate.orm:hibernate-jpamodelgen:6.5.2.Final'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    annotationProcessor("org.hibernate.orm:hibernate-jpamodelgen:6.5.2.Final")
}
```

**Java — Criteria с динамическими фильтрами, проекцией и отдельным count**

```java
package com.example.criteria;

import jakarta.persistence.*;
import jakarta.persistence.criteria.*;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.domain.*;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders_c")
class OrderC {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String status;
    @Column(nullable = false) Long totalCents;
    @Column(nullable = false) Instant createdAt = Instant.now();
    protected OrderC() {}
    OrderC(String status, long total){ this.status=status; this.totalCents=total; }
}

record OrderRow(Long id, String status, Long totalCents, Instant createdAt){}

@Repository
class OrderCriteriaRepo {
    @PersistenceContext EntityManager em;

    public Page<OrderRow> search(String status, Instant from, Instant to, Pageable pageable) {
        CriteriaBuilder cb = em.getCriteriaBuilder();

        // Основной запрос на DTO
        CriteriaQuery<OrderRow> cq = cb.createQuery(OrderRow.class);
        Root<OrderC> root = cq.from(OrderC.class);
        List<Predicate> preds = new ArrayList<>();
        if (status != null) preds.add(cb.equal(root.get("status"), status));
        if (from != null) preds.add(cb.greaterThanOrEqualTo(root.get("createdAt"), from));
        if (to != null) preds.add(cb.lessThan(root.get("createdAt"), to));

        cq.select(cb.construct(OrderRow.class,
                root.get("id"), root.get("status"), root.get("totalCents"), root.get("createdAt")))
          .where(cb.and(preds.toArray(Predicate[]::new)));

        // Сортировка из Pageable
        List<jakarta.persistence.criteria.Order> orders = new ArrayList<>();
        for (Sort.Order o : pageable.getSort()) {
            Path<?> p = root.get(o.getProperty());
            orders.add(o.isAscending() ? cb.asc(p) : cb.desc(p));
        }
        if (!orders.isEmpty()) cq.orderBy(orders);

        TypedQuery<OrderRow> q = em.createQuery(cq)
            .setFirstResult((int) pageable.getOffset())
            .setMaxResults(pageable.getPageSize());
        List<OrderRow> content = q.getResultList();

        // Отдельный count
        CriteriaQuery<Long> countQ = cb.createQuery(Long.class);
        Root<OrderC> countRoot = countQ.from(OrderC.class);
        countQ.select(cb.count(countRoot))
              .where(cb.and(preds.stream().map(p -> p)  // пересоздать на countRoot
                      .toArray(Predicate[]::new))); // для краткости демонстрации оставим упрощённо
        long total = em.createQuery(countQ).getSingleResult();

        return new PageImpl<>(content, pageable, total);
    }
}

@Service
class OrderCriteriaService {
    private final OrderCriteriaRepo repo;
    OrderCriteriaService(OrderCriteriaRepo repo){ this.repo = repo; }

    @Transactional(readOnly = true)
    public Page<OrderRow> find(String status, Instant from, Instant to, int page, int size) {
        return repo.search(status, from, to, PageRequest.of(page, size, Sort.by("createdAt").descending().and(Sort.by("id").descending())));
    }
}
```

**Kotlin — Criteria с DTO-проекцией**

```kotlin
package com.example.criteria

import jakarta.persistence.*
import jakarta.persistence.criteria.*
import org.springframework.data.domain.*
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

@Entity
@Table(name = "orders_c")
class OrderC(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var status: String = "",
    @Column(nullable = false) var totalCents: Long = 0,
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

data class OrderRow(val id: Long, val status: String, val totalCents: Long, val createdAt: Instant)

@Repository
class OrderCriteriaRepo(@PersistenceContext private val em: EntityManager) {

    fun search(status: String?, from: Instant?, to: Instant?, pageable: Pageable): Page<OrderRow> {
        val cb = em.criteriaBuilder

        val cq: CriteriaQuery<OrderRow> = cb.createQuery(OrderRow::class.java)
        val root: Root<OrderC> = cq.from(OrderC::class.java)
        val preds = mutableListOf<Predicate>()
        if (status != null) preds += cb.equal(root.get<String>("status"), status)
        if (from != null) preds += cb.greaterThanOrEqualTo(root.get("createdAt"), from)
        if (to != null) preds += cb.lessThan(root.get("createdAt"), to)

        cq.select(cb.construct(OrderRow::class.java,
            root.get<Long>("id"), root.get<String>("status"),
            root.get<Long>("totalCents"), root.get<Instant>("createdAt")
        )).where(cb.and(*preds.toTypedArray()))

        val orders = pageable.sort.map { o ->
            val path = root.get<Any>(o.property)
            if (o.isAscending) cb.asc(path) else cb.desc(path)
        }.toList()
        if (orders.isNotEmpty()) cq.orderBy(orders)

        val content = em.createQuery(cq)
            .setFirstResult(pageable.offset.toInt())
            .setMaxResults(pageable.pageSize)
            .resultList

        val countQ: CriteriaQuery<Long> = cb.createQuery(Long::class.java)
        val countRoot = countQ.from(OrderC::class.java)
        // Для краткости: в реальном коде пересоберите предикаты на countRoot
        countQ.select(cb.count(countRoot)).where(cb.and(*preds.toTypedArray()))
        val total = em.createQuery(countQ).singleResult

        return PageImpl(content, pageable, total)
    }
}

@Service
class OrderCriteriaService(private val repo: OrderCriteriaRepo) {
    @Transactional(readOnly = true)
    fun find(status: String?, from: Instant?, to: Instant?, page: Int, size: Int): Page<OrderRow> =
        repo.search(status, from, to, PageRequest.of(page, size, Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"))))
}
```

---

## QueryDSL: предикаты, join-ы, подзапросы, «тяжёлые» фильтры — когда лучше, чем строковый JPQL

QueryDSL даёт типобезопасный DSL-язык поверх JPA с генерацией `Q`-классов по вашим сущностям. В коде вы оперируете полями как свойствами (`QOrder.order.status.eq("PAID")`), собираете сложные предикаты через `BooleanBuilder`, пишете подзапросы и join-ы с минимальным шумом. В итоге получается компактнее и безопаснее, чем Criteria, и выразительнее, чем строковый JPQL — особенно для «тяжёлых» поисков, где динамики много.

Сильная сторона QueryDSL — **композиция предикатов**. Вы легко добавляете условия при наличии параметра (`if (p != null) builder.and(...);`) и переиспользуете их в разных запросах. Для страниц — отдельный плюс: есть готовые `offset/limit`, быстрое сопоставление со `Sort` (можно написать адаптер), проекции в DTO через `Projections.constructor/fields`.

Подзапросы и агрегации выглядят естественно: создаёте `JPAExpressions.select(...)`, вкладываете в `where` или используете в `select`. Для кейсов «покажи заказы с максимальной суммой клиента», «все клиенты с количеством заказов > N» QueryDSL пишет читаемо, без строковых конкатенаций и без длинных билдеров Criteria.

С fetch join тоже приятно: `join(order.customer, customer).fetchJoin()` — и всё. Но действуют те же правила, что и в JPA: коллекционные fetch-join и пагинация не дружат; выполняйте их только для «карточек» или применяйте проекции. Для листингов предпочитайте `select(...)` на DTO/интерфейс — это «тонко» и быстро.

Интеграция со Spring удобна: регистрируете `JPAQueryFactory` как бин (на основе `EntityManager`), пишете кастомный репозиторий или компонент-DAO. Есть и `QuerydslPredicateExecutor` у Spring Data, но в проде чаще берут «ручной» `JPAQueryFactory`: он гибче и не навязывает глобальную схему фильтров.

Про версии: с Hibernate 6 и Spring Boot 3 нужны артефакты `:jakarta`. Генерация `Q`-классов — через `annotationProcessor` (Java) или `kapt` (Kotlin). Не забудьте добавить `jakarta.annotation-api` и `jakarta.persistence-api` как `annotationProcessor`/`kapt`, иначе процессор не увидит аннотаций.

Пагинация в QueryDSL 5+ делается «вручную»: сначала `fetch()` содержимое с `offset/limit`, затем отдельный `count()` (или оптимизированный подсчёт, если фильтр «несложный»). Это честнее и прозрачнее, чем единый «fetchResults» из старых версий, и даёт шанс оптимизировать count (например, не делать его, если страница не полная).

Наблюдаемость и тесты: логируйте SQL Hibernate’ом, проверяйте планы и следите за N+1. QueryDSL легко подталкивает к fetch join «на всякий случай» — не делайте так в списках. Помните правило: списки — проекции; карточки — fetch join. И держите интеграционные тесты на самые тяжёлые фильтры.

Наконец, типизация — это не только удобство, но и защита от регрессов. Переименование поля в сущности сломает сборку, а не прод. Для больших команд это ценно: меньше «тихих» падений в рантайме.

**Gradle — зависимости для QueryDSL (Hibernate 6 / Jakarta)**

*Groovy DSL*

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jakarta'
    annotationProcessor 'jakarta.annotation:jakarta.annotation-api:2.1.1'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api:3.1.0'
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL*

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("kapt") version "1.9.24"
    java
}
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.querydsl:querydsl-jpa:5.0.0:jakarta")
    kapt("com.querydsl:querydsl-apt:5.0.0:jakarta")
    kapt("jakarta.annotation:jakarta.annotation-api:2.1.1")
    kapt("jakarta.persistence:jakarta.persistence-api:3.1.0")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**Java — JPAQueryFactory бин, запрос c динамическими предикатами, DTO-проекция и отдельный count**

```java
package com.example.qdsl;

import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.Order;
import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.core.types.Projections;
import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Entity
@Table(name = "customers_q")
class CustomerQ {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false) String name;
    @Column(nullable = false) String city;
}

@Entity
@Table(name = "orders_q")
class OrderQ {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "customer_id", nullable = false)
    CustomerQ customer;
    @Column(nullable = false) String status;
    @Column(nullable = false) Long totalCents;
    @Column(nullable = false) Instant createdAt = Instant.now();
}

record OrderView(Long id, String customerName, String status, Long totalCents, Instant createdAt) {}

@Configuration
class QuerydslConfig {
    @PersistenceContext EntityManager em;
    @Bean JPAQueryFactory jpaQueryFactory() { return new JPAQueryFactory(em); }
}

@Repository
class OrderQdslRepo {
    private final JPAQueryFactory qf;
    OrderQdslRepo(JPAQueryFactory qf){ this.qf = qf; }

    @Transactional(readOnly = true)
    public Page<OrderView> search(String status, String city, Instant from, Instant to, Pageable pageable) {
        QOrderQ o = QOrderQ.orderQ;
        QCustomerQ c = QCustomerQ.customerQ;

        BooleanBuilder where = new BooleanBuilder();
        if (status != null) where.and(o.status.eq(status));
        if (city != null && !city.isBlank()) where.and(c.city.equalsIgnoreCase(city));
        if (from != null) where.and(o.createdAt.goe(from));
        if (to != null) where.and(o.createdAt.lt(to));

        List<OrderSpecifier<?>> orders = pageable.getSort().stream().map(s -> {
            var path = switch (s.getProperty()) {
                case "createdAt" -> o.createdAt;
                case "id" -> o.id;
                case "totalCents" -> o.totalCents;
                case "customerName" -> c.name;
                default -> o.id;
            };
            return new OrderSpecifier<>(s.isAscending() ? Order.ASC : Order.DESC, path);
        }).toList();

        var content = qf.select(Projections.constructor(OrderView.class,
                            o.id, c.name, o.status, o.totalCents, o.createdAt))
                .from(o)
                .join(o.customer, c)
                .where(where)
                .orderBy(orders.toArray(OrderSpecifier[]::new))
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();

        long total = qf.select(o.id.count())
                .from(o).join(o.customer, c)
                .where(where)
                .fetchOne();

        return new PageImpl<>(content, pageable, total == null ? 0 : total);
    }

    @Transactional(readOnly = true)
    public OrderView showCard(Long id) {
        QOrderQ o = QOrderQ.orderQ;
        QCustomerQ c = QCustomerQ.customerQ;
        return qf.select(Projections.constructor(OrderView.class,
                        o.id, c.name, o.status, o.totalCents, o.createdAt))
                .from(o)
                .join(o.customer, c).fetchJoin() // карточка — fetch join уместен
                .where(o.id.eq(id))
                .fetchOne();
    }
}
```

**Kotlin — тот же запрос на QueryDSL с динамикой и пагинацией**

```kotlin
package com.example.qdsl

import com.querydsl.core.BooleanBuilder
import com.querydsl.core.types.Order
import com.querydsl.core.types.OrderSpecifier
import com.querydsl.core.types.Projections
import com.querydsl.jpa.impl.JPAQueryFactory
import jakarta.persistence.*
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.domain.*
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

@Entity
@Table(name = "customers_q")
class CustomerQ(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = "",
    @Column(nullable = false) var city: String = ""
)

@Entity
@Table(name = "orders_q")
class OrderQ(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "customer_id", nullable = false)
    var customer: CustomerQ? = null,
    @Column(nullable = false) var status: String = "",
    @Column(nullable = false) var totalCents: Long = 0,
    @Column(nullable = false) var createdAt: Instant = Instant.now()
)

data class OrderView(
    val id: Long,
    val customerName: String,
    val status: String,
    val totalCents: Long,
    val createdAt: Instant
)

@Configuration
class QuerydslConfig {
    @PersistenceContext lateinit var em: EntityManager
    @Bean fun jpaQueryFactory() = JPAQueryFactory(em)
}

@Repository
class OrderQdslRepo(private val qf: JPAQueryFactory) {

    @Transactional(readOnly = true)
    fun search(status: String?, city: String?, from: Instant?, to: Instant?, pageable: Pageable): Page<OrderView> {
        val o = QOrderQ.orderQ
        val c = QCustomerQ.customerQ

        val where = BooleanBuilder().apply {
            if (!status.isNullOrBlank()) and(o.status.eq(status))
            if (!city.isNullOrBlank()) and(c.city.equalsIgnoreCase(city))
            if (from != null) and(o.createdAt.goe(from))
            if (to != null) and(o.createdAt.lt(to))
        }

        val orders: List<OrderSpecifier<*>> = pageable.sort.map { s ->
            val path = when (s.property) {
                "createdAt" -> o.createdAt
                "id" -> o.id
                "totalCents" -> o.totalCents
                "customerName" -> c.name
                else -> o.id
            }
            OrderSpecifier(if (s.isAscending) Order.ASC else Order.DESC, path)
        }.toList()

        val content = qf.select(
                Projections.constructor(
                    OrderView::class.java,
                    o.id, c.name, o.status, o.totalCents, o.createdAt
                )
            )
            .from(o)
            .join(o.customer, c)
            .where(where)
            .orderBy(*orders.toTypedArray())
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetch()

        val total = qf.select(o.id.count())
            .from(o).join(o.customer, c)
            .where(where)
            .fetchOne() ?: 0L

        return PageImpl(content, pageable, total)
    }

    @Transactional(readOnly = true)
    fun showCard(id: Long): OrderView? {
        val o = QOrderQ.orderQ
        val c = QCustomerQ.customerQ
        return qf.select(
                Projections.constructor(
                    OrderView::class.java,
                    o.id, c.name, o.status, o.totalCents, o.createdAt
                )
            )
            .from(o)
            .join(o.customer, c).fetchJoin()
            .where(o.id.eq(id))
            .fetchOne()
    }
}
```

**Практические выводы.**

1. Для «мозаичных» фильтров и быстрого роста требований используйте `Specification<T>`; держите спеки мелкими и чистыми, страницы — проекциями.
2. Criteria берите, когда нужно управлять всей формой запроса (select/aggregate/group/having) и когда метамодель/типобезопасность важнее бойлерплейта.
3. QueryDSL — выбор для сложной динамики, подзапросов и join-ов; он короче Criteria и безопаснее строкового JPQL. Списки — через DTO/проекции; карточки — через fetch join.


# 12. JDBC и jOOQ: когда уходить ниже

*Ниже — практическое продолжение «Основ» с фокусом на те случаи, где JPA перестаёт быть лучшим выбором. Покажу, когда и почему стоит опуститься на уровень JDBC или взять jOOQ, как настроить потоковое чтение, батчи, upsert/CTE/окна, и как безопасно сочетать это со Spring Data JPA.*

**Зависимости (общие, можно добавить к существующему проекту):**

**Gradle (Groovy DSL)**

```groovy
plugins {
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
    id 'java'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'org.springframework.boot:spring-boot-starter-jooq' // для jOOQ-пунктов
    runtimeOnly 'org.postgresql:postgresql:42.7.3'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
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
    implementation("org.springframework.boot:spring-boot-starter-jdbc")
    implementation("org.springframework.boot:spring-boot-starter-jooq")
    runtimeOnly("org.postgresql:postgresql:42.7.3")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
```

---

## Где JPA неэффективна: отчёты, агрегаты/окна, сложные CTE, апсерты/`ON CONFLICT`, bulk-операции

JPA оптимальна для «моделирования предметной области» и CRUD-а на сущностях. Там, где вы оперируете графами объектов, жизненным циклом и инвариантами, ORM экономит время и снижает связность. Но как только задача превращается в отчёт, агрегации по большим объёмам, окна и CTE — преимущества JPA сходят на нет: вам нужен **точный SQL** и контроль плана запроса, а не граф сущностей.

Оконные функции (`row_number()`, `sum() over (partition by ...)`, `lag/lead`) — каркас более 70% реальных отчётов. JPQL их не выражает переносимо; Hibernate 6 кое-где помогает нативом или функциями, но это уже «выход наружу» и потеря переносимости. На практике такие запросы пишут либо через **native SQL** (JDBC), либо через **jOOQ** — типобезопасный DSL поверх SQL конкретной СУБД.

CTE (`with ... as (...)`) — ещё один «стержень» сложной аналитики и пошаговых преобразований. В JPQL CTE отсутствуют, в Criteria — нет удобного аналога. Снова выигрывают JDBC/jOOQ: читабельнее, предсказуемее, легче оптимизировать и профилировать `EXPLAIN ANALYZE`.

Апсерты — «вставь или обнови при конфликте». В Postgres это `INSERT ... ON CONFLICT (key) DO UPDATE SET ...`. Через JPA это превращается в хрупкую «попробуй найти → если нет — persist → иначе — merge», что медленно и гонкоопасно. Нативный апсерт **одним запросом** — быстрее и безопаснее. Точно так же массовые операции `bulk update/delete` в JPQL обходят Persistence Context и часто требуют ручной синхронизации — проще сразу писать точный SQL.

Bulk-вставки — любимая зона JPA-«боли». Даже с `hibernate.jdbc.batch_size` и упорядочиванием `order_inserts` вы завязаны на стратегию генерации идентификаторов, и реальный throughput может сильно уступать **чистому JDBC батчу**. Сырые батчи легче дозировать, логировать и оборачивать в «швабры» (retry, chunking).

В ETL/репортинге часто требуется «временная таблица»/`UNLOGGED`/`ON COMMIT DROP`, загрузка «как есть», с последующим `INSERT INTO ... SELECT` — это «низкоуровневые» приёмы, в которых ORM просто не участвует. Их честнее выразить SQL-ом и выполнить через JDBC.

Важно и то, что JPA тянет за собой Persistence Context. При выводе десятков тысяч строк это превращается в лишний расход памяти, если вы не используете проекции. А в чистом JDBC вы формируете DTO сразу на чтении и не держите ORM-«хвост».

Хорошее правило: **читайте там, где «тонко», а не «толсто»**. Для отчётных таблиц и агрегатов берите JDBC/jOOQ; для карточек/команд — JPA. И не бойтесь смешивать: один сервис может иметь JPA-репозитории и рядом DAO на jOOQ/JdbcTemplate.

Безопасность и тесты не страдают. Вы так же проходите через Spring-транзакции, так же используете Testcontainers и миграции Flyway/Liquibase. Вы просто говорите базе «вот ровно такой SQL», и это хорошо.

А теперь — небольшой, но рабочий пример честного апсерта через JDBC. Он показывает, как одним запросом вставить или обновить запись в Postgres, избежав гонок и двух походов.

**Java — upsert `ON CONFLICT` через `JdbcTemplate`**

```java
package com.example.sql;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

@Repository
public class ProductUpsertDao {
    private final JdbcTemplate jdbc;

    public ProductUpsertDao(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Transactional
    public int upsert(String sku, long priceCents) {
        String sql = """
            insert into products (sku, price_cents)
            values (?, ?)
            on conflict (sku) do update
              set price_cents = excluded.price_cents
            """;
        return jdbc.update(sql, ps -> {
            ps.setString(1, sku);
            ps.setLong(2, priceCents);
        });
    }
}
```

**Kotlin — тот же апсерт**

```kotlin
package com.example.sql

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional

@Repository
class ProductUpsertDao(private val jdbc: JdbcTemplate) {

    @Transactional
    fun upsert(sku: String, priceCents: Long): Int {
        val sql = """
            insert into products (sku, price_cents)
            values (?, ?)
            on conflict (sku) do update
              set price_cents = excluded.price_cents
        """.trimIndent()
        return jdbc.update(sql) { ps ->
            ps.setString(1, sku)
            ps.setLong(2, priceCents)
        }
    }
}
```

---

## `JdbcTemplate/NamedParameterJdbcTemplate`: `RowMapper`, батчи `batchUpdate`, `SimpleJdbcInsert`, таймауты и `fetchSize`

`JdbcTemplate` — рабочая лошадка Spring: он упрощает JDBC без потери контроля. Бинц-менеджмент, освобождение ресурсов, удобные коллбеки и мапперы — и вы получаете «чистый SQL» с минимумом бойлерплейта. Когда важна читаемость параметров по имени, берите `NamedParameterJdbcTemplate` — особенно удобно для длинных `IN`/`VALUES` и «семантических» апдейтов.

`RowMapper<T>` — простой способ собирать DTO напрямую из `ResultSet`. Это экономит память: вы не создаёте сущности, не держите их в PC, а формируете тонкий объект «по дороге». Для больших выборок используйте `RowCallbackHandler` (обработка построчно) — он не накапливает результаты в список и дружит с курсорами.

Батчи (`batchUpdate`) — основной способ ускорить множество однотипных вставок/обновлений. JDBC отправляет их пачкой, и драйвер/СУБД оптимизируют сетевые round-trip’ы и планы. Важно «подбирать» размер батча (обычно 500–2000), следить за ошибками в отдельной строке (возвращаемые counts), и выставлять драйверные оптимизации (для PG — `reWriteBatchedInserts=true`).

`SimpleJdbcInsert` удобен, когда вы не хотите писать `INSERT` руками: описываете таблицу и набор колонок, передаёте `Map<String,Object>` — и всё. Он умеет возвращать сгенерированные ключи. Для массовых вставок всё же лучше явный `batchUpdate`.

Таймауты — защита от «зависших» запросов. Вы можете выставить `queryTimeout` на `JdbcTemplate`/`DataSource` или на отдельном `PreparedStatement`. Для длинных отчётов это must-have: один «заблудившийся» запрос не должен повесить пул.

`fetchSize` — хинт драйверу о размере «порций» результата. В Postgres, если `autocommit=false` и `fetchSize>0`, будет открыт server-side cursor, и строки поедут чанками. Это ключ к потоковой обработке (ниже), но и в «обычных» выборках помогает не раздувать память.

Логируйте SQL — но осторожно с параметрами (PII). Включайте логгер на `org.springframework.jdbc.core` для отладки, а в проде оставляйте только тайминги/метрики (Micrometer, p95/p99).

Следите за типами. JDBC маппит по именам/индексам столбцов; опечатка — рантайм-ошибка. Тесты с Testcontainers покрывают риск: вы проверяете SQL против реальной БД, а не «эмулятора».

Ниже — небольшой, но самодостаточный пример: `RowMapper` для DTO, батчевая вставка и `SimpleJdbcInsert`. Он показывает, как собрать списки без JPA и с хорошим контролем.

**Java — `RowMapper`, `batchUpdate`, `SimpleJdbcInsert`, таймауты**

```java
package com.example.jdbc;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.core.simple.SimpleJdbcInsert;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.Instant;
import java.util.*;

record OrderRow(Long id, Long customerId, Long totalCents, Instant createdAt) {}

@Repository
public class OrderJdbcDao {
    private final JdbcTemplate jdbc;
    private final NamedParameterJdbcTemplate named;

    public OrderJdbcDao(JdbcTemplate jdbc, NamedParameterJdbcTemplate named) {
        this.jdbc = jdbc; this.named = named;
        this.jdbc.setQueryTimeout(30);       // сек
        this.jdbc.setFetchSize(500);         // дефолтный fetchSize
    }

    private static final RowMapper<OrderRow> ORDER_MAPPER = (rs, rowNum) -> new OrderRow(
        rs.getLong("id"),
        rs.getLong("customer_id"),
        rs.getLong("total_cents"),
        rs.getTimestamp("created_at").toInstant()
    );

    @Transactional(readOnly = true)
    public List<OrderRow> findRecent(int limit) {
        return jdbc.query("""
            select id, customer_id, total_cents, created_at
            from orders
            order by id desc
            limit ?
        """, ORDER_MAPPER, limit);
    }

    @Transactional
    public int[] insertBatch(List<OrderRow> rows) {
        return jdbc.batchUpdate("""
            insert into orders (customer_id, total_cents, created_at)
            values (?, ?, ?)
        """, rows, 1000, (ps, r) -> {
            ps.setLong(1, r.customerId());
            ps.setLong(2, r.totalCents());
            ps.setTimestamp(3, java.sql.Timestamp.from(r.createdAt()));
        });
    }

    @Transactional
    public Number insertOneSimple(long customerId, long totalCents, Instant createdAt) {
        SimpleJdbcInsert insert = new SimpleJdbcInsert(jdbc)
            .withTableName("orders")
            .usingGeneratedKeyColumns("id");
        Map<String, Object> params = Map.of(
            "customer_id", customerId,
            "total_cents", totalCents,
            "created_at", java.sql.Timestamp.from(createdAt)
        );
        return insert.executeAndReturnKey(params);
    }

    @Transactional
    public int updateNamed(List<Long> ids, long delta) {
        var sql = """
            update orders set total_cents = total_cents + :d
            where id in (:ids)
        """;
        var params = new MapSqlParameterSource()
            .addValue("d", delta)
            .addValue("ids", ids);
        return named.update(sql, params);
    }
}
```

**Kotlin — те же приёмы**

```kotlin
package com.example.jdbc

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.RowMapper
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
import org.springframework.jdbc.core.simple.SimpleJdbcInsert
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

data class OrderRow(val id: Long?, val customerId: Long, val totalCents: Long, val createdAt: Instant)

@Repository
class OrderJdbcDao(
    private val jdbc: JdbcTemplate,
    private val named: NamedParameterJdbcTemplate
) {
    init {
        jdbc.queryTimeout = 30
        jdbc.fetchSize = 500
    }

    private val mapper = RowMapper { rs, _ ->
        OrderRow(
            id = rs.getLong("id"),
            customerId = rs.getLong("customer_id"),
            totalCents = rs.getLong("total_cents"),
            createdAt = rs.getTimestamp("created_at").toInstant()
        )
    }

    @Transactional(readOnly = true)
    fun findRecent(limit: Int): List<OrderRow> =
        jdbc.query(
            """
            select id, customer_id, total_cents, created_at
            from orders
            order by id desc
            limit ?
            """.trimIndent(),
            mapper,
            limit
        )

    @Transactional
    fun insertBatch(rows: List<OrderRow>): IntArray =
        jdbc.batchUpdate(
            """
            insert into orders (customer_id, total_cents, created_at)
            values (?, ?, ?)
            """.trimIndent(),
            rows,
        ) { ps, r ->
            ps.setLong(1, r.customerId)
            ps.setLong(2, r.totalCents)
            ps.setTimestamp(3, java.sql.Timestamp.from(r.createdAt))
        }

    @Transactional
    fun insertOneSimple(customerId: Long, totalCents: Long, createdAt: Instant): Number {
        val insert = SimpleJdbcInsert(jdbc)
            .withTableName("orders")
            .usingGeneratedKeyColumns("id")
        val params = mapOf(
            "customer_id" to customerId,
            "total_cents" to totalCents,
            "created_at" to java.sql.Timestamp.from(createdAt)
        )
        return insert.executeAndReturnKey(params)
    }

    @Transactional
    fun updateNamed(ids: List<Long>, delta: Long): Int {
        val sql = """
            update orders set total_cents = total_cents + :d
            where id in (:ids)
        """.trimIndent()
        val params = MapSqlParameterSource()
            .addValue("d", delta)
            .addValue("ids", ids)
        return named.update(sql, params)
    }
}
```

---

## Потоковая обработка: курсоры/стримы, контроль памяти, работа с LOB

Потоковая обработка — ключ к стабильной памяти при больших выборках. Идея проста: не загружать всё в список, а читать и обрабатывать **порциями**. В Postgres это реализуется server-side cursor при `autocommit=false` и `fetchSize>0`. В Spring это достигается транзакцией `readOnly=true` + `JdbcTemplate` с `fetchSize` и обработчиком `RowCallbackHandler`.

Курсор — forward-only: вы идёте вперёд и не откручиваете назад. Это накладывает стиль на код: «прочитал строку → сразу записал в поток/файл/очередь». Так вы не держите данные в памяти дольше, чем нужно. Для CSV/JSON-стриминга используйте буферизированные writer’ы и периодический flush.

Контроль памяти — не только про «коллекции». Избегайте создания временных больших строк (`StringBuilder` на десятки мегабайт), сериализуйте по мере чтения (например, NDJSON — «объект на строку»). Если вы всё же собираете пачку (например, на отправку в Kafka), ограничивайте размер чанка (1–5 тыс. строк) и очищайте.

С LOB (BLOB/CLOB) потоки особенно важны. Никогда не грузите файл целиком в память. Вставка — через `PreparedStatement.setBinaryStream`, чтение — `getBinaryStream` и копирование в `OutputStream` с небольшим буфером. Старайтесь, чтобы и на стороне клиента (HTTP) был поток, а не буфер.

Таймауты и «heartbeat» логи — обязательны. Длинный курсор держит соединение. Выставляйте `statement_timeout`/`queryTimeout`, логируйте «прочитали N строк» каждые, скажем, 5k. Это помогает эксплуатации понимать, что процесс жив и что скорость нормальная.

Если отчёт может продолжаться десятки минут, **бейте его на логические чанки** (по `id` или по дате). Транзакция на чанк + фиксация прогресса позволяет рестартовать процесс и не держать курсор вечность.

LOB-и в Postgres — отдельная тема (`oid`/large objects), но в большинстве приложений хватает `bytea` (обычная колонка байтов). Она прекрасно работает с потоками, индексируется (хэши/префиксы) и упрощает миграции.

При чтении очень больших объектов («видео/архивы») не забывайте про ограничение полосы (throttling), иначе вы забьёте сетевой стек приложения. Отдавайте это на балансировщик/прокси или используйте NIO/Reactive, если это действительно streaming API, а не служебный экспорт.

Ниже — пример потоковой выгрузки через `RowCallbackHandler` и чтение/запись LOB. Он безопасен по памяти и легко переносим.

**Java — курсорное чтение и LOB-потоки**

```java
package com.example.stream;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowCallbackHandler;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.*;
import java.sql.ResultSet;
import java.sql.SQLException;

@Service
public class StreamingService {
    private final JdbcTemplate jdbc;

    public StreamingService(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
        this.jdbc.setFetchSize(1000);
        this.jdbc.setQueryTimeout(60);
    }

    @Transactional(readOnly = true)
    public void exportCsv(OutputStream out) throws IOException {
        try (BufferedWriter w = new BufferedWriter(new OutputStreamWriter(out))) {
            w.write("id,customer_id,total_cents,created_at\n");
            RowCallbackHandler rch = rs -> {
                try {
                    w.write(rs.getLong("id") + "," + rs.getLong("customer_id") + ","
                            + rs.getLong("total_cents") + "," + rs.getTimestamp("created_at").toInstant() + "\n");
                } catch (IOException e) { throw new UncheckedIOException(e); }
            };
            jdbc.query("""
                select id, customer_id, total_cents, created_at
                from orders where created_at >= now() - interval '30 days'
                order by id
            """, rch);
            w.flush();
        }
    }

    @Transactional
    public void saveBlob(long docId, InputStream content) {
        jdbc.update("insert into docs(id, data) values(?, ?)",
            ps -> { ps.setLong(1, docId); ps.setBinaryStream(2, content); });
    }

    @Transactional(readOnly = true)
    public void loadBlob(long docId, OutputStream target) throws IOException {
        jdbc.query("select data from docs where id = ?", rs -> {
            try (InputStream in = rs.getBinaryStream(1)) {
                in.transferTo(target);
            }
        }, docId);
    }
}
```

**Kotlin — те же подходы**

```kotlin
package com.example.stream

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.RowCallbackHandler
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.io.*

@Service
class StreamingService(private val jdbc: JdbcTemplate) {

    init {
        jdbc.fetchSize = 1000
        jdbc.queryTimeout = 60
    }

    @Transactional(readOnly = true)
    fun exportCsv(out: OutputStream) {
        BufferedWriter(OutputStreamWriter(out)).use { w ->
            w.write("id,customer_id,total_cents,created_at\n")
            val handler = RowCallbackHandler { rs ->
                w.write("${rs.getLong("id")},${rs.getLong("customer_id")},${rs.getLong("total_cents")},${rs.getTimestamp("created_at").toInstant()}\n")
            }
            jdbc.query(
                """
                select id, customer_id, total_cents, created_at
                from orders where created_at >= now() - interval '30 days'
                order by id
                """.trimIndent(),
                handler
            )
            w.flush()
        }
    }

    @Transactional
    fun saveBlob(docId: Long, content: InputStream) {
        jdbc.update("insert into docs(id, data) values(?, ?)") { ps ->
            ps.setLong(1, docId)
            ps.setBinaryStream(2, content)
        }
    }

    @Transactional(readOnly = true)
    fun loadBlob(docId: Long, target: OutputStream) {
        jdbc.query("select data from docs where id = ?",
            { rs -> rs.getBinaryStream(1).use { it.transferTo(target) } },
            docId
        )
    }
}
```

---

## jOOQ как «типобезопасный SQL»: генерация DSL из схемы, тонкая оптимизация под конкретную СУБД; совместное использование с JPA в одном сервисе

jOOQ — это «SQL как Java/Kotlin-DSL». В отличие от ORM, он не скрывает SQL, а делает его **типобезопасным**: поля, таблицы и функции — это классы/методы, проверяемые компилятором. jOOQ отлично подходит для CTE, окон, апсертов, сложных `JOIN/UNION`, когда важна читаемость и контроль. При этом вы остаётесь в экосистеме Spring (`DSLContext` как бин, транзакции Spring).

Сердце jOOQ — **генерация кода** из схемы: на основе БД создаются классы `Tables`, `Routines`, `Keys`. Это даёт настоящую типизацию (например, `ORDERS.CREATED_AT` — это `Field<Timestamp>`). Генератор можно запускать при билде (плагин Gradle) или отдельной задачей в CI. Если генерация пока не настроена, вы можете писать и «динамический DSL» через `DSL.table("...")`, но сила jOOQ именно в сгенерированных типах.

jOOQ «понимает» диалект. Запись `insertInto(...).onConflict(...).doUpdate()` сгенерирует корректный `ON CONFLICT` в Postgres и «эквивалент» в других СУБД, где возможно. То же касается оконных функций и CTE (`with(...)`). Это упрощает переносимость и снимает страх «завязать» проект на один SQL-диалект — хотя честно, для продвинутых фич вы всё равно будете «под Postgres».

Сочетание с JPA — обычная практика. Вы храните агрегаты/отчёты в jOOQ, а доменную модель — в JPA. В одном транзакционном методе вы можете читать через jOOQ и сохранять сущности через JPA: Spring обеспечивает общий `Connection` (через `DataSourceTransactionManager` или JPA-менеджер). Главное — не мешать «толстые» графы и «большие» отчёты в одну транзакцию без необходимости.

Производительность jOOQ хороша «из коробки»: нет PC, вы превращаете строки сразу в DTO/Record. Но ответственность за «правильный SQL» — на вас: индексы, планы, таймауты. К счастью, jOOQ логирует SQL и помогает строить сложные конструкции безопасно.

Тестирование — как у JDBC: интеграционные тесты с Testcontainers, миграции прикладываются перед запуском, `DSLContext` инжектится как бин. Вы можете писать фикстуры SQL и проверять, что запрос возвращает ожидаемые строки.

В больших системах jOOQ становится «языком данных», на котором пишут отчётные DAO. Там, где тревожит JPA N+1 или нужно сделать хитрый `WITH RECURSIVE`, jOOQ читабельнее и безопаснее, чем «нативные строки» в коде.

Ниже — примеры: CTE + оконная функция, апсерт. Я сознательно не использую сгенерированные классы, чтобы пример был самодостаточным; в комментарии — минимальная конфигурация codegen-плагина для прод-проекта.

**Java — `DSLContext`: CTE, окно, upsert**

```java
package com.example.jooq;

import org.jooq.*;
import org.jooq.impl.DSL;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.time.OffsetDateTime;
import java.util.List;

@Repository
public class ReportDao {

    private final DSLContext dsl;

    public ReportDao(DSLContext dsl) { this.dsl = dsl; }

    @Transactional(readOnly = true)
    public List<Record3<Long, String, Long>> topCustomersBySum(int limit) {
        // WITH o AS (select customer_id, total_cents from orders where created_at >= now() - interval '30 days')
        // select customer_id, sum(total_cents) as sum, row_number() over (order by sum desc) as rn from o ...
        Name o = DSL.name("o");
        Table<?> cte = DSL.name("o").as(DSL.table(
                DSL.select(DSL.field("customer_id"), DSL.field("total_cents"))
                   .from(DSL.table("orders"))
                   .where(DSL.field("created_at").ge(DSL.field("now() - interval '30 days'")))
        ));
        Field<Long> customerId = DSL.field(DSL.name("customer_id"), Long.class);
        Field<Long> total = DSL.sum(DSL.field(DSL.name("total_cents"), Long.class)).as("sum");
        Field<Integer> rn = DSL.rowNumber().over().orderBy(total.desc()).as("rn");

        return dsl.with(o).as(cte)
                .select(customerId, total, rn)
                .from(DSL.table(o))
                .groupBy(customerId)
                .orderBy(total.desc())
                .limit(limit)
                .fetch();
    }

    @Transactional
    public int upsertOrder(long id, long customerId, long totalCents, OffsetDateTime createdAt) {
        // INSERT ... ON CONFLICT (id) DO UPDATE
        return dsl.insertInto(DSL.table("orders"))
                .columns(DSL.field("id"), DSL.field("customer_id"), DSL.field("total_cents"), DSL.field("created_at"))
                .values(id, customerId, totalCents, createdAt)
                .onConflict(DSL.field("id"))
                .doUpdate()
                .set(DSL.field("customer_id"), customerId)
                .set(DSL.field("total_cents"), totalCents)
                .set(DSL.field("created_at"), createdAt)
                .execute();
    }
}
```

**Kotlin — `DSLContext` с CTE, окном и upsert**

```kotlin
package com.example.jooq

import org.jooq.DSLContext
import org.jooq.Field
import org.jooq.Name
import org.jooq.Record3
import org.jooq.impl.DSL
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.time.OffsetDateTime

@Repository
class ReportDao(private val dsl: DSLContext) {

    @Transactional(readOnly = true)
    fun topCustomersBySum(limit: Int): List<Record3<Long, String, Long>> {
        val o: Name = DSL.name("o")
        val cte = DSL.name("o").`as`(
            DSL.table(
                DSL.select(DSL.field("customer_id"), DSL.field("total_cents"))
                    .from(DSL.table("orders"))
                    .where(DSL.field("created_at").ge(DSL.field("now() - interval '30 days'")))
            )
        )
        val customerId: Field<Long> = DSL.field(DSL.name("customer_id"), Long::class.java)
        val total: Field<Long> = DSL.sum(DSL.field(DSL.name("total_cents"), Long::class.java)).`as`("sum")
        val rn = DSL.rowNumber().over().orderBy(total.desc()).`as`("rn")

        return dsl.with(o).`as`(cte)
            .select(customerId, total, rn)
            .from(DSL.table(o))
            .groupBy(customerId)
            .orderBy(total.desc())
            .limit(limit)
            .fetch()
    }

    @Transactional
    fun upsertOrder(id: Long, customerId: Long, totalCents: Long, createdAt: OffsetDateTime): Int =
        dsl.insertInto(DSL.table("orders"))
            .columns(DSL.field("id"), DSL.field("customer_id"), DSL.field("total_cents"), DSL.field("created_at"))
            .values(id, customerId, totalCents, createdAt)
            .onConflict(DSL.field("id"))
            .doUpdate()
            .set(DSL.field("customer_id"), customerId)
            .set(DSL.field("total_cents"), totalCents)
            .set(DSL.field("created_at"), createdAt)
            .execute()
}
```

**Совместное использование с JPA в одном сервисе (одна транзакция)**
— читаем отчёт jOOQ-ом и обновляем доменные сущности JPA. Это легально и часто удобно.

**Java — jOOQ + JPA под одной транзакцией**

```java
package com.example.mixed;

import com.example.jooq.ReportDao;
import com.example.domain.Order; // ваша JPA-сущность
import com.example.domain.OrderRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class MixedService {
    private final ReportDao reportDao;
    private final OrderRepository orderRepo;

    public MixedService(ReportDao reportDao, OrderRepository orderRepo) {
        this.reportDao = reportDao; this.orderRepo = orderRepo;
    }

    @Transactional
    public void recomputeTopAndMarkFeatured() {
        var top = reportDao.topCustomersBySum(10); // jOOQ
        top.forEach(rec -> {
            Long id = (Long) rec.get(0); // пример; в реале — свой DTO
            orderRepo.findById(id).ifPresent(o -> o.setStatus("FEATURED")); // JPA
        });
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.mixed

import com.example.jooq.ReportDao
import com.example.domain.OrderRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class MixedService(
    private val reportDao: ReportDao,
    private val orderRepo: OrderRepository
) {
    @Transactional
    fun recomputeTopAndMarkFeatured() {
        val top = reportDao.topCustomersBySum(10)
        top.forEach { rec ->
            val id = rec.get(0, Long::class.java)
            orderRepo.findById(id).ifPresent { it.status = "FEATURED" }
        }
    }
}
```

**(Опционально) Минимальная конфигурация codegen для jOOQ (Gradle Kotlin DSL, плагин nu.studer.jooq)**
*Этот блок — справочный. Его можно добавить позже; он не нужен для компиляции приведённого кода.*

```kotlin
plugins {
    id("nu.studer.jooq") version "9.0"
}
dependencies {
    jooqGenerator("org.postgresql:postgresql:42.7.3")
}
jooq {
    version.set("3.19.9") // актуальную версию уточните в документации
    configurations {
        create("main") {
            generateSchemaSourceOnCompilation.set(true)
            jooqConfiguration.apply {
                jdbc.apply {
                    driver = "org.postgresql.Driver"
                    url = "jdbc:postgresql://localhost:5432/app"
                    user = "app"
                    password = "app"
                }
                generator.apply {
                    name = "org.jooq.codegen.DefaultGenerator"
                    database.apply {
                        name = "org.jooq.meta.postgres.PostgresDatabase"
                        inputSchema = "public"
                    }
                    target.apply {
                        packageName = "org.jooq.generated"
                        directory = "$projectDir/src/generated/jooq"
                    }
                }
            }
        }
    }
}
sourceSets.main {
    java.srcDir("src/generated/jooq")
}
```

**Итоги практики по JDBC/jOOQ**

1. Сложные отчёты, окна, CTE, апсерты и bulk — это **зона JDBC/jOOQ**.
2. `JdbcTemplate` даёт контроль: `RowMapper`, `RowCallbackHandler`, батчи, таймауты, `fetchSize`.
3. LOB — только потоками. Вставка/чтение через стрим без буферизации в память.
4. jOOQ — «SQL как код»: типобезопасно и прозрачно. С JPA уживается отлично; держите транзакции короткими и роль каждого инструмента понятной.



# 13. Репозитории: расширение и кастомизация

Ниже разберём практики, которые начинают понадобиться, когда возможностей «голых» `JpaRepository` уже не хватает: общий базовый репозиторий с полезными утилитами, точечные (surgical) кастомные реализации для отдельных агрегатов, а также здравые границы ответственности между репозиторием и сервисом. Всё — с рабочими примерами на Java и Kotlin, и с акцентом на производительность и предсказуемость поведения.

**Gradle (Groovy DSL)**

```groovy
plugins {
  id 'org.springframework.boot' version '3.3.4'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'java'
}
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.springframework.boot:spring-boot-starter-cache' // для спец-кешей при желании
  implementation 'com.github.ben-manes.caffeine:caffeine:3.1.8'      // пример локального кеша
  runtimeOnly 'org.postgresql:postgresql:42.7.3'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
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
  implementation("org.springframework.boot:spring-boot-starter-cache")
  implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
```

**application.yml (минимум для примеров)**

```yaml
spring:
  jpa:
    open-in-view: false
```

## Свой базовый интерфейс репозитория: общие методы, хук на `EntityManager`

Первый импульс «вынести общее» в репозитории возникает, когда по проекту начинают копироваться однотипные приёмы: пакетные вставки/обновления, принудительный `flush/clear`, поиск по коллекции id **с сохранением порядка**, безопасные read-only выборки без dirty checking и т.п. Делать всё это в каждом конкретном интерфейсе неудобно; правильнее — определить **свой базовый репозиторий**, расширяющий `JpaRepository`, и подключить собственную базовую реализацию на основе `SimpleJpaRepository`.

Технически это два шага: объявить `@NoRepositoryBean`-интерфейс `BaseRepository<T, ID>` c дополнительными методами, и написать `BaseRepositoryImpl<T, ID>` (наследник `SimpleJpaRepository`), который умеет обращаться к `EntityManager` и Hibernate `Session`. Затем в конфигурации репозиториев указать `repositoryBaseClass = BaseRepositoryImpl.class`, чтобы Spring Data JPA использовал именно вашу реализацию как «скелет» для всех репозиториев домена.

Хук на `EntityManager` даёт доступ к «тонким ручкам»: можно временно переключить `FlushModeType`, вызвать `flush()` в определённой точке, использовать `setHint(READ_ONLY,true)`/`Session.setDefaultReadOnly(true)` для дешёвого чтения, вызвать `unwrap(Session.class)` и зачитаться forward-only-курсор с `fetchSize`. Когда это собрано в базовый класс, команда пользуется единым и проверенным API, а не изобретает велосипед заново.

Полезный метод — `findAllByIdsOrdered(Collection<ID> ids)`, который возвращает сущности строго в порядке входных id. Это спасает в слоях, где порядок важен (например, выдача карточек по «желаемому» списку id). Ещё один частый утилитарный метод — `saveInBatch(List<T> entities, int batchSize)`, который дозирует нагрузку на L1-кэш и вызывает `flush/clear` между порциями.

Часто хочется и «правильного» read-only чтения без слепков в PC. В базовой реализации можно сделать `streamReadOnly(Query)`/`listReadOnly(TypedQuery<T>)`, которые выставляют хинты `READ_ONLY`, `fetchSize`, `cacheable=false` (если нужно), и выдают результат в потоковом режиме. Так вы снизите нагрузку на память и GC в списках.

Осторожнее с «универсализацией»: не нужно тянуть в базовый репозиторий бизнес-логику, кеши конкретных сущностей, специфичные нативные запросы. Всё, что привязано к одной модели — оставляем на уровне кастомных имплементаций или сервисов. База — только общий инфраструктурный «инструментарий».

С точки зрения тестирования базовый репозиторий неплохо покрыть мини-набором интеграционных тестов (Testcontainers), которые проверят: `findAllByIdsOrdered` сохраняет порядок, `saveInBatch` действительно делает `flush/clear`, а `streamReadOnly` не накапливает сущности в PC. Это инвестиция, которая окупится, когда репозиториев станет десятки.

Производительноcть базового класса зависит от аккуратного обращения с `flush()`. Он не должен вызываться «про запас». Для пакетных операций — да; для обычного `save()` — нет. Хорошая эвристика: в базовом методе при `batchSize > 0` делать `flush/clear` каждые N записей, иначе — не трогать.

Важный момент — совместимость с транзакциями. Базовый репозиторий не должен «сам» открывать/закрывать транзакции, это ответственность сервисного слоя. Но он может использовать `@Transactional(readOnly = true)` на методах «чтения», если вы явно вызываете их из сервисов без аннотаций. В большинстве проектов аннотации ставят на сервисах — и это нормально.

Наконец, не забывайте про `@NoRepositoryBean`: без него Spring попытается создать бин для интерфейса базового репозитория — и вы получите ошибку. Это мелочь, но встречается часто.

**Java — базовый репозиторий и реализация с хуком на EntityManager**

```java
package com.example.repo.base;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.persistence.TypedQuery;
import org.hibernate.Session;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.jpa.repository.support.SimpleJpaRepository;

import java.util.*;
import java.util.stream.Collectors;

@NoRepositoryBean
public interface BaseRepository<T, ID> extends JpaRepository<T, ID> {
    List<T> findAllByIdsOrdered(Collection<ID> ids);
    void saveInBatch(List<T> entities, int batchSize);
    <X> List<X> listReadOnly(TypedQuery<X> query, Integer fetchSize);
}

public class BaseRepositoryImpl<T, ID> extends SimpleJpaRepository<T, ID> implements BaseRepository<T, ID> {

    @PersistenceContext
    private final EntityManager em;

    public BaseRepositoryImpl(Class<T> domainClass, EntityManager em) {
        super(domainClass, em);
        this.em = em;
    }

    @Override
    @Transactional(readOnly = true)
    public List<T> findAllByIdsOrdered(Collection<ID> ids) {
        if (ids == null || ids.isEmpty()) return List.of();
        List<T> list = this.findAllById(ids);
        // вернуть в порядке входных ids
        Map<Object, T> byId = list.stream().collect(Collectors.toMap(this::getId, e -> e));
        List<T> ordered = new ArrayList<>(ids.size());
        for (ID id : ids) {
            T t = byId.get(id);
            if (t != null) ordered.add(t);
        }
        return ordered;
    }

    @Override
    @Transactional
    public void saveInBatch(List<T> entities, int batchSize) {
        Session session = em.unwrap(Session.class);
        int i = 0;
        for (T e : entities) {
            session.persist(e);
            i++;
            if (batchSize > 0 && i % batchSize == 0) {
                session.flush();
                session.clear();
            }
        }
    }

    @Override
    @Transactional(readOnly = true)
    public <X> List<X> listReadOnly(TypedQuery<X> query, Integer fetchSize) {
        Session session = em.unwrap(Session.class);
        boolean prev = session.isDefaultReadOnly();
        try {
            session.setDefaultReadOnly(true);
            if (fetchSize != null) query.setHint(org.hibernate.jpa.HibernateHints.HINT_FETCH_SIZE, fetchSize);
            query.setHint(org.hibernate.jpa.HibernateHints.HINT_READ_ONLY, true);
            return query.getResultList();
        } finally {
            session.setDefaultReadOnly(prev);
        }
    }

    @SuppressWarnings("unchecked")
    private Object getId(T entity) {
        return em.getEntityManagerFactory().getPersistenceUnitUtil().getIdentifier(entity);
    }
}
```

```java
package com.example.repo.base;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.repo",
        repositoryBaseClass = BaseRepositoryImpl.class
)
class JpaRepoConfig { }
```

**Kotlin — базовый репозиторий и реализация**

```kotlin
package com.example.repo.base

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import jakarta.persistence.TypedQuery
import org.hibernate.Session
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.repository.NoRepositoryBean
import org.springframework.data.jpa.repository.support.SimpleJpaRepository
import org.springframework.transaction.annotation.Transactional

@NoRepositoryBean
interface BaseRepository<T, ID> : JpaRepository<T, ID> {
    fun findAllByIdsOrdered(ids: Collection<ID>): List<T>
    fun <X> listReadOnly(query: TypedQuery<X>, fetchSize: Int? = null): List<X>
    fun saveInBatch(entities: List<T>, batchSize: Int = 1000)
}

class BaseRepositoryImpl<T, ID>(
    domainClass: Class<T>,
    @PersistenceContext private val em: EntityManager
) : SimpleJpaRepository<T, ID>(domainClass, em), BaseRepository<T, ID> {

    @Transactional(readOnly = true)
    override fun findAllByIdsOrdered(ids: Collection<ID>): List<T> {
        if (ids.isEmpty()) return emptyList()
        val list = findAllById(ids)
        val byId = list.associateBy { em.entityManagerFactory.persistenceUnitUtil.getIdentifier(it) }
        return ids.mapNotNull { byId[it] }
    }

    @Transactional
    override fun saveInBatch(entities: List<T>, batchSize: Int) {
        val session = em.unwrap(Session::class.java)
        var i = 0
        for (e in entities) {
            session.persist(e)
            i++
            if (batchSize > 0 && i % batchSize == 0) {
                session.flush()
                session.clear()
            }
        }
    }

    @Transactional(readOnly = true)
    override fun <X> listReadOnly(query: TypedQuery<X>, fetchSize: Int?): List<X> {
        val session = em.unwrap(Session::class.java)
        val prev = session.isDefaultReadOnly
        try {
            session.isDefaultReadOnly = true
            fetchSize?.let { query.setHint(org.hibernate.jpa.HibernateHints.HINT_FETCH_SIZE, it) }
            query.setHint(org.hibernate.jpa.HibernateHints.HINT_READ_ONLY, true)
            return query.resultList
        } finally {
            session.isDefaultReadOnly = prev
        }
    }
}
```

Для примеров использования нам достаточно любой сущности; ниже в следующих пунктах появится `Product`/`Order` — они автоматически унаследуют возможности из базового репозитория благодаря конфигурации `repositoryBaseClass`.

## Кастомные имплементации для отдельных репозиториев (surgical-SQL, спец-кеши)

Помимо общего «скелета», иногда нужна **точечная**, «хирургическая» оптимизация: нативный запрос на конкретную таблицу, быстрый «апсерт», локальный кеш по бизнес-ключу, специальная пагинация с хинтами. В Spring Data JPA это решается **фрагментами репозитория**: вы объявляете интерфейс `XxxRepositoryCustom` и реализацию `XxxRepositoryImpl` (важно совпадение суффиксов), а затем ваш основной `XxxRepository` расширяет и `JpaRepository`, и `XxxRepositoryCustom`.

Такой фрагмент идеально подходит для «surgical SQL», когда JPA-абстракция мешает: нужно `INSERT ... ON CONFLICT`, `WITH ...` (CTE), специфичный индекс-хинт, или банально — получить **две колонки** без подъёма всей сущности. Внутри кастомной реализации у вас есть `EntityManager`, можно инжектировать и `JdbcTemplate`, и локальные кеши (например, Caffeine).

Кеш в уровне репозитория — решение спорное, но иногда оправданное: например, часто дергается «getIdBySku» по стабильному справочнику. Делать для этого отдельный сервис и палить сеть/базу в каждом вызове — накладно; достаточно локального `Cache<K,V>` с управляемой инвалидацией при изменении данных. Главное — **место** инвалидации: обновляете `Product` — почистите кеш по соответствующему ключу.

Опасные места кастомных реализаций — рассогласование с L1-кешем и транзакциями. Если вы делаете нативный `UPDATE`/`DELETE`, Hibernate **не знает** об изменениях управляемых сущностей: либо делайте `clear()`/evict после операции, либо используйте такие методы вне сессий, где сущности уже загружены и будут переиспользованы. В нашем примере после апсерта мы не держим `Product` в PC — и это безопасно.

Ещё один нюанс — маппинг DTO из `createNativeQuery`. Либо используйте JPA `SqlResultSetMapping`, либо возвращайте «плоские» типы (`Object[]`) и вручную склеивайте DTO. Для компактности примера ниже покажу «вручную» c `Object[]`, а также альтернативу через `JdbcTemplate` (она короче и чаще приятнее).

Локальные кеши: если выбираете Caffeine, определите TTL/ёмкость, продумайте инвалидацию. В критичных местах не полагайтесь на «вечные» значения; даже для справочников лучше короткий TTL или подписка на «инвалидацию» через событие. В примере — мини-кеш по `sku → id`.

Каскады и кеши не дружат «из коробки»: если вы мягко удалили запись (`@SQLDelete`), а кеш хранит id по ключу — метод «найди id» начнёт возвращать «скрытую» запись. Решение — включить в ключ кеша параметр «только живые/со всеми» или всегда читать только «живые» (и инвалидировать при удалении).

Журналируйте «попадания/промахи» кеша — это покажет, окупается ли он вообще. В микросервисах с несколькими инстансами помните: Caffeine локален, может дать разные ответы на разных узлах при гонках. Если нужна единая картинка — идите в распределённый кеш (Redis) или откажитесь от кеша на этом уровне.

И последнее: не превращайте кастомный репозиторий в «монолит всего». Выносите сюда только то, что **реально** относится к данным этого агрегата и даёт измеримый выигрыш. Всё иное — в сервис.

**Java — кастомный фрагмент репозитория с surgical SQL и локальным кешем (Caffeine)**

```java
package com.example.repo.product;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;

import java.time.Instant;
import java.util.Optional;
import java.util.concurrent.TimeUnit;

// --- сущность для примера ---
@Entity
@Table(name = "products", indexes = { @Index(columnList = "sku", unique = true) })
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true, length = 64) private String sku;
    @Column(nullable = false) private Long priceCents;
    @Column(nullable = false) private Instant createdAt = Instant.now();
    protected Product() {}
    public Product(String sku, long priceCents){ this.sku=sku; this.priceCents=priceCents; }
    public Long getId(){ return id; }
    public String getSku(){ return sku; }
    public Long getPriceCents(){ return priceCents; }
    public void setPriceCents(Long v){ this.priceCents = v; }
}

// --- фрагмент интерфейса ---
interface ProductRepositoryCustom {
    Optional<Long> findIdBySkuCached(String sku);
    int upsertPrice(String sku, long priceCents);
    Optional<ProductPriceView> findPriceViewNative(String sku);
}

record ProductPriceView(Long id, String sku, Long priceCents) {}

// --- основное API репозитория ---
@Repository
interface ProductRepository extends JpaRepository<Product, Long>, ProductRepositoryCustom { }

// --- реализация фрагмента (важно имя ...Impl) ---
class ProductRepositoryImpl implements ProductRepositoryCustom {

    @PersistenceContext private EntityManager em;
    private final Cache<String, Long> idBySku = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .build();

    @Override
    @Transactional(readOnly = true)
    public Optional<Long> findIdBySkuCached(String sku) {
        Long cached = idBySku.getIfPresent(sku);
        if (cached != null) return Optional.of(cached);
        Long id = em.createQuery("select p.id from Product p where p.sku = :sku", Long.class)
                .setParameter("sku", sku)
                .setHint(org.hibernate.jpa.HibernateHints.HINT_READ_ONLY, true)
                .getResultStream().findFirst().orElse(null);
        if (id != null) idBySku.put(sku, id);
        return Optional.ofNullable(id);
    }

    @Override
    @Transactional
    public int upsertPrice(String sku, long priceCents) {
        // surgical SQL: Postgres ON CONFLICT
        int updated = em.createNativeQuery("""
            insert into products(sku, price_cents, created_at)
            values (:sku, :price, now())
            on conflict (sku) do update set price_cents = excluded.price_cents
        """)
        .setParameter("sku", sku)
        .setParameter("price", priceCents)
        .executeUpdate();

        idBySku.invalidate(sku); // инвалидация локального кеша
        return updated;
    }

    @Override
    @Transactional(readOnly = true)
    public Optional<ProductPriceView> findPriceViewNative(String sku) {
        var rows = em.createNativeQuery("""
            select id, sku, price_cents from products where sku=:sku
        """).setParameter("sku", sku).getResultList();

        if (rows.isEmpty()) return Optional.empty();
        Object[] r = (Object[]) rows.get(0);
        return Optional.of(new ProductPriceView(((Number) r[0]).longValue(), (String) r[1], ((Number) r[2]).longValue()));
    }
}
```

**Kotlin — тот же кастомный фрагмент**

```kotlin
package com.example.repo.product

import com.github.benmanes.caffeine.cache.Caffeine
import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.time.Instant
import java.util.Optional
import java.util.concurrent.TimeUnit

@Entity
@Table(name = "products", indexes = [Index(columnList = "sku", unique = true)])
class Product(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true, length = 64)
    var sku: String = "",
    @Column(nullable = false)
    var priceCents: Long = 0,
    @Column(nullable = false)
    var createdAt: Instant = Instant.now()
)

data class ProductPriceView(val id: Long, val sku: String, val priceCents: Long)

interface ProductRepositoryCustom {
    fun findIdBySkuCached(sku: String): Optional<Long>
    fun upsertPrice(sku: String, priceCents: Long): Int
    fun findPriceViewNative(sku: String): Optional<ProductPriceView>
}

@Repository
interface ProductRepository : JpaRepository<Product, Long>, ProductRepositoryCustom

class ProductRepositoryImpl(
    @PersistenceContext private val em: EntityManager
) : ProductRepositoryCustom {

    private val idBySku = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build<String, Long>()

    @Transactional(readOnly = true)
    override fun findIdBySkuCached(sku: String): Optional<Long> {
        idBySku.getIfPresent(sku)?.let { return Optional.of(it) }
        val id = em.createQuery("select p.id from Product p where p.sku = :sku", java.lang.Long::class.java)
            .setParameter("sku", sku)
            .setHint(org.hibernate.jpa.HibernateHints.HINT_READ_ONLY, true)
            .resultStream.findFirst().orElse(null)
        if (id != null) idBySku.put(sku, id)
        return Optional.ofNullable(id)
    }

    @Transactional
    override fun upsertPrice(sku: String, priceCents: Long): Int {
        val updated = em.createNativeQuery(
            """
            insert into products(sku, price_cents, created_at)
            values (:sku, :price, now())
            on conflict (sku) do update set price_cents = excluded.price_cents
            """.trimIndent()
        )
            .setParameter("sku", sku)
            .setParameter("price", priceCents)
            .executeUpdate()
        idBySku.invalidate(sku)
        return updated
    }

    @Transactional(readOnly = true)
    override fun findPriceViewNative(sku: String): Optional<ProductPriceView> {
        val rows = em.createNativeQuery("select id, sku, price_cents from products where sku=:sku")
            .setParameter("sku", sku)
            .resultList
        if (rows.isEmpty()) return Optional.empty()
        val r = rows[0] as Array<*>
        return Optional.of(ProductPriceView((r[0] as Number).toLong(), r[1] as String, (r[2] as Number).toLong()))
    }
}
```

Такой подход (фрагмент + имплементация) сохраняет декларативный стиль Spring Data для «обычных» операций и даёт точечную мощь там, где она нужна. А кеш показывает, как экономно закрыть частый «lookup» без перегрева БД — при условии аккуратной инвалидации.

## «Граница ответственности» репозитория: тонкие методы vs «толстые» сервисы — баланс читаемости и производительности

Классический соблазн — начать «накачивать» репозиторий бизнес-логикой: валидацией входных данных, агрегациями из нескольких агрегатов, вызовами внешних систем и т.д. Это ведёт к «толстым» репозиториям, трудно тестируемым и плохо сочетаемым между собой. Правильная граница проста: **репозиторий оперирует данными одной модели и близкими к ней проекциями; сервис оркестрирует сценарий** из нескольких репозиториев, внешних клиентов и доменных политик.

«Тонкий» репозиторий — это набор чётких контрактов: найти по бизнес-ключу, сохранить партию, получить проекцию для списка, выполнить спец-запрос по этой таблице. Такой код стабилен, хорошо тестируется slice-тестами `@DataJpaTest` и редко ломается при изменениях бизнес-логики. Он не знает о «кто вызвал» и «какой use-case», он про данные.

Сервис «толстый» — но это правильно: именно сервис знает, как связать операции разных агрегатов, какие инварианты применить, какие транзакции нужны (`REQUIRES_NEW` для аудита, `readOnly` для чтения), как устроен ретрай и как логировать шаги. Здесь уместны и кеши уровня сценария (например, кэшировать результат сложного отчёта, зависящего от нескольких таблиц).

Производительность страдает, когда репозитории начинают дергать друг друга (скрытые циклы) или когда в одном методе репозитория происходит и чтение, и запись в несвязанные агрегаты. Рецепт — держать коммуникацию на уровне сервиса, а в репозитории не использовать «чужие» репозитории напрямую. Это снижает риск взаимных зависимостей и «скрытых» транзакций.

Ещё одно правило — явная композиция: если для страницы нужен список и «итого», не прячьте второй запрос внутри метода репозитория «найти список». Пусть сервис читает список через один метод, а «итого» — через другой, чтобы было видно, сколько запросов реально выполняется и где узкие места.

Проверка границ на практике простая: задайте себе вопрос, можно ли переиспользовать этот метод репозитория **в другом сценарии** без изменения. Если нет — вероятно, это логика уровня сервиса (или даже контроллера), и ей не место в репозитории. Репозиторий — библиотека данных; сервис — сценарий.

Разумеется, есть исключения. Например, «инкремент склада» — мелкая бизнес-операция строго на одном агрегате `Product`. Её удобно разместить как кастомный метод в репозитории (atomic `UPDATE ... SET stock = stock + ? WHERE id=?`). Это «узкоспециализированный» метод, но он **в рамках** модели данных. Сценарии «создай заказ и зарезервируй склад» — уже сервис.

В плане тестов: репозитории — slice `@DataJpaTest` на реальной БД (Testcontainers), сервисы — интеграционные тесты с моками внешних клиентов и настоящими транзакциями. Такой раскол ускоряет обратную связь и помогает ловить регрессы отдельно в хранилище и в оркестрации.

Не забывайте про «потолок» репозитория: как только у вас пошли агрегации с CTE/окнами через jOOQ — держите их в **отдельном DAO**, не смешивая с JPA-репозиторием. Это тоже граница ответственности между «ORM-частью» и «репортинг-частью». Общее — транзакции и миграции; разное — подход к данным и оптимизации.

В результате «тонкие репозитории, толстые сервисы» дают читаемость и предсказуемость. Сервис становится «источником правды» сценариев, репозиторий — «надёжным драйвером данных». И с ростом проекта эта архитектура помогает распределять зоны ответственности между командами.

**Java — тонкий репозиторий + «толстый» сервис (оркестрация нескольких операций)**

```java
package com.example.boundary;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name = "customers_b")
class CustomerB {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String email;
    protected CustomerB() {}
    CustomerB(String email){ this.email = email; }
}

@Entity
@Table(name = "orders_b")
class OrderB {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) Long customerId;
    @Column(nullable = false) Long totalCents;
    protected OrderB() {}
    OrderB(Long customerId, Long totalCents){ this.customerId = customerId; this.totalCents = totalCents; }
}

@Repository interface CustomerBRepository extends JpaRepository<CustomerB, Long> {
    boolean existsByEmailIgnoreCase(String email);
}

@Repository interface OrderBRepository extends JpaRepository<OrderB, Long> { }

@Service
class CheckoutService {
    private final CustomerBRepository customers;
    private final OrderBRepository orders;

    CheckoutService(CustomerBRepository customers, OrderBRepository orders) {
        this.customers = customers; this.orders = orders;
    }

    @Transactional
    public Long checkout(String email, long totalCents) {
        // Валидация и сценарий — здесь (сервис), не в репозитории
        if (!customers.existsByEmailIgnoreCase(email)) {
            CustomerB c = customers.save(new CustomerB(email));
            OrderB o = orders.save(new OrderB(c.id, totalCents));
            return o.id;
        } else {
            // Упростим: существующему создаём заказ
            CustomerB c = customers.findAll().stream()
                    .filter(u -> u.email.equalsIgnoreCase(email)).findFirst().orElseThrow();
            OrderB o = orders.save(new OrderB(c.id, totalCents));
            return o.id;
        }
    }
}
```

**Kotlin — тот же принцип**

```kotlin
package com.example.boundary

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "customers_b")
class CustomerB(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var email: String = ""
)

@Entity
@Table(name = "orders_b")
class OrderB(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false)
    var customerId: Long = 0,
    @Column(nullable = false)
    var totalCents: Long = 0
)

@Repository
interface CustomerBRepository : JpaRepository<CustomerB, Long> {
    fun existsByEmailIgnoreCase(email: String): Boolean
}

@Repository
interface OrderBRepository : JpaRepository<OrderB, Long>

@Service
class CheckoutService(
    private val customers: CustomerBRepository,
    private val orders: OrderBRepository
) {
    @Transactional
    fun checkout(email: String, totalCents: Long): Long {
        val cid = if (!customers.existsByEmailIgnoreCase(email)) {
            customers.save(CustomerB(email = email)).id!!
        } else {
            customers.findAll().first { it.email.equals(email, ignoreCase = true) }.id!!
        }
        return orders.save(OrderB(customerId = cid, totalCents = totalCents)).id!!
    }
}
```

В этом примере репозитории — «тонкие»: в них нет сценарной логики; всё принятие решений и последовательность шагов — в сервисе. Если завтра добавится аудит, резервы, публикация в Kafka — это появится в сервисе и не сломает репозитории. Производительность контролируется точечно: при необходимости вы добавите кастомные методы в отдельные `*RepositoryCustom`, не мешая общему контракту.


# 14. Тестирование продвинутого persistence

## Testcontainers с реальной СУБД, прогон миграций перед тестами, сидирование данных

Тестировать persistence слой «по-настоящему» — значит запускать тесты на **реальной СУБД** с реальными типами данных, планами выполнения и поведением транзакций. Встраиваемые БД («под PG») часто ведут себя иначе: иначе считают индексы, не знают `jsonb`, не поддерживают `ON CONFLICT`, по-другому реализуют блокировки. Поэтому базовый инструмент для интеграционных тестов сегодня — **Testcontainers**: он поднимает контейнер Postgres (или другой СУБД) на время тестов, выдаёт реквизиты подключения и убирает за собой. Вы получаете предсказуемость и верность поведения, а команда — уверенность, что код пройдёт на проде.

Важно включить **миграции перед тестами**. Схему нельзя полагать на `ddl-auto=create` — вы рискуете рассинхронизировать код и DDL. Гораздо надёжнее запускать Flyway/Liquibase в тестах ровно так же, как это делается в приложении при старте. Тогда ваши тесты становятся «детектором рассинхронизации»: если новая сущность забыта в миграции, тесты упадут на `DataAccessException` при первом `persist`.

**Сидирование (fixtures)** — данные для тестов. Есть три основных подхода. Первый: SQL-скрипты (`@Sql`) — быстрые, декларативные, хорошо подходят для справочников и больших наборов данных. Второй: «чистые фабрики» в коде — билдеры сущностей, которые создают минимально валидные объекты для данного сценария. Третий: комбинированный — базовые справочники через `@Sql`, доменные экземпляры через фабрики. Важно, чтобы тесты не зависели друг от друга — то есть сами готовили нужный набор данных и не полагались на «порядок выполнения».

Чтобы **ускорить старт** контейнера, используйте reusability (перезапускаемый контейнер между сессиями разработчика) и статический singleton для всего сьюта. В Testcontainers это делается либо через `~/.testcontainers.properties` (`testcontainers.reuse.enable=true`), либо через паттерн «одиночки» с `@Testcontainers` и статическим `@Container`. Для CI лучше поднимать контейнер на сьют (а не на каждый тест-класс), чтобы не тратить минуты на каждый запуск.

Тайм-зона и локаль в тестах — тоже не мелочи. Приводите всё к **UTC** (и в БД, и в Hibernate), иначе сравнение времён превратится в случайный набор «±1 час» и будет ломать тесты в феврале и октябре. В `application-test.yml` укажите `hibernate.jdbc.time_zone: UTC`. Это также исключит расхождения при сериализации `Instant`.

При работе с Testcontainers удобно **прокидывать свойства** в Spring через `@DynamicPropertySource`: так Boot сам создаст DataSource на базе параметров контейнера (`jdbcUrl`, `username`, `password`). И — обязательно — отключите автоподмену DataSource у `@DataJpaTest`: `@AutoConfigureTestDatabase(replace = NONE)`. Иначе Boot поставит H2 и весь смысл потеряется.

С точки зрения структуры тестов полезно отделять **slice-тесты** (`@DataJpaTest`) от **полных интеграционных** (`@SpringBootTest`). Первые — быстрые проверки репозиториев/DAO на реальной СУБД с минимальным контекстом. Вторые — сценарные тесты сервисов/транзакций/блокировок. Обе группы работают на одном контейнере Postgres в рамках сьюта, что экономит время.

Наконец, не забывайте про **миграции расширений**. Если используете `jsonb`, массивы, `hstore`, убедитесь, что в миграции есть `CREATE EXTENSION ...`. В H2 такого нет, а в реальном PG — есть. Тесты должны «ловить» такие пропуски. Именно поэтому интеграционные тесты на Testcontainers — это не «роскошь», а «страховка» от инцидентов.

**Gradle зависимости (тесты)**

*Groovy DSL*

```groovy
dependencies {
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
  testImplementation 'org.testcontainers:junit-jupiter:1.20.2'
  testImplementation 'org.testcontainers:postgresql:1.20.2'
  testImplementation 'org.flywaydb:flyway-core'
  runtimeOnly 'org.postgresql:postgresql:42.7.3'
}
```

*Kotlin DSL*

```kotlin
dependencies {
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.testcontainers:junit-jupiter:1.20.2")
  testImplementation("org.testcontainers:postgresql:1.20.2")
  testImplementation("org.flywaydb:flyway-core")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**application-test.yml (UTC + OSIV off)**

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc.time_zone: UTC
```

**Java — базовая настройка Testcontainers + Flyway + сидирование**

```java
package com.example.testcontainers;

import org.junit.jupiter.api.*;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.containers.PostgreSQLContainer;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class BasicJpaSliceIT {

  @Container
  static final PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine")
      .withDatabaseName("app")
      .withUsername("app")
      .withPassword("app");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
    r.add("spring.flyway.enabled", () -> true);
  }

  @Autowired JdbcTemplate jdbc;

  @BeforeEach
  void seed() {
    jdbc.update("insert into customers(id, email) values(1, 'seed@example.com')");
  }

  @Test
  void containerStartsAndMigrationsApplied() {
    Integer cnt = jdbc.queryForObject("select count(*) from customers", Integer.class);
    Assertions.assertNotNull(cnt);
  }
}
```

**Kotlin — тот же slice-тест**

```kotlin
package com.example.testcontainers

import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class BasicJpaSliceIT {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:16-alpine")
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app")

        @JvmStatic
        @DynamicPropertySource
        fun props(reg: DynamicPropertyRegistry) {
            reg.add("spring.datasource.url", pg::getJdbcUrl)
            reg.add("spring.datasource.username", pg::getUsername)
            reg.add("spring.datasource.password", pg::getPassword)
            reg.add("spring.flyway.enabled") { true }
        }
    }

    @Autowired lateinit var jdbc: JdbcTemplate

    @BeforeEach
    fun seed() {
        jdbc.update("insert into customers(id, email) values(1, 'seed@example.com')")
    }

    @Test
    fun containerStartsAndMigrationsApplied() {
        val cnt = jdbc.queryForObject("select count(*) from customers", Int::class.java)
        assertNotNull(cnt)
    }
}
```

---

## Проверки на N+1/пагинацию/графы загрузки; сбор статистики Hibernate в тестах

Проблема **N+1** редко видна на локальном стенде, но на проде превращается в лавину запросов. В тестах эту ловушку можно ловить **счётчиком SQL**. Hibernate предоставляет `Statistics` (если включить `hibernate.generate_statistics=true`), где есть метрики запросов. Мы можем обнулить статистику, выполнить метод репозитория и проверить, сколько SQL улетело. Такой тест не «ломкий» и не зависит от формулировок SQL — ему важна только **кардинальность**.

Для списков важно проверить **пагинацию** и **стабильную сортировку**. Если вы делаете `Page<T>`, но сортируете по полю с повторяющимися значениями без вторичного ключа, строки будут «перепрыгивать» между страницами. Тест должен это ловить: запросите первую страницу два раза и проверьте, что порядки совпадают. При `JOIN FETCH` коллекций пагинация может ломаться совсем (дубликаты строк) — такой тест тоже обязателен.

Когда вы используете **графы загрузки** (`@EntityGraph`/`@NamedEntityGraph`), тест легко подтверждает, что N+1 устранён: один запрос на корневую сущность и (в идеале) ни одного дополнительного на «детей». Если граф сложный и вы не уверены, включите логгер SQL на DEBUG только в тестах и проверьте вручную, но предпочтительнее — автоматические проверки статистики.

Для корректных измерений надо **обнулять и «прогревать»**. Часто первый запрос включает `prepare`/инициализацию; чтобы не привязывать тест к этому, либо прогрейте репозиторий «холостым» вызовом, либо снимайте статистику в повторном вызове. Также важно выгрузить L1-кеш: `em.clear()` перед замером, иначе часть чтения пройдёт из PC и тест «соврёт».

Ещё один аспект — **глубокие графы**. Если вы тянете `Order -> items -> product`, проверяйте не только количество запросов, но и **размер результата** (rows). При неаккуратном `JOIN FETCH` на «многие» результат раздувается, и Hibernate делает `distinct`. Это может повлиять на `count`. Тесты должны удостовериться, что «карточка» загружается без дубликатов, а «списки» — через проекции.

Важный сценарий — **EntityGraph в репозитории**. Тест показывает, что метод с графом делает меньше запросов, чем базовый. Это сильный сигнал для команды: «для карточки используйте метод X, для списка — Y (DTO)». Документируйте это прямо в Javadoc метода репозитория и подтверждайте тестом.

Можно использовать и внешние инструменты — например, datasource-proxy, которые логируют каждый запрос. Но с Hibernate Statistics проще и быстрее. Добавьте утиль «счётчик запросов» с функциональным интерфейсом и используйте его во всех тестах на производительность загрузки.

Наконец, всё это нужно **в CI**. Если тест ловит N+1 и падает при регрессе, вы обнаружите проблему на PR, а не на проде. Старайтесь писать такие тесты как **«контракты производительности загрузки»**.

**application-test.yml — статистика Hibernate**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

**Java — утиль счётчика запросов + тест N+1 и EntityGraph**

```java
package com.example.nplus1;

import jakarta.persistence.*;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Entity
class Author {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
  String name;
  @OneToMany(mappedBy="author", fetch=FetchType.LAZY)
  List<Book> books;
}
@Entity
class Book {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
  String title;
  @ManyToOne(fetch=FetchType.LAZY) @JoinColumn(name="author_id")
  Author author;
}

interface AuthorRepository extends JpaRepository<Author, Long> {
  @EntityGraph(attributePaths = "books")
  @Query("select a from Author a")
  List<Author> findAllWithBooksGraph();

  @Query("select a from Author a")
  List<Author> findAllPlain();
}

@DataJpaTest
@EnableJpaRepositories(considerNestedRepositories = true)
class NPlusOneIT {

  @Autowired EntityManager em;
  @Autowired AuthorRepository authors;

  private Statistics stats() {
    return em.getEntityManagerFactory().unwrap(SessionFactory.class).getStatistics();
  }

  @BeforeEach
  @Transactional
  void seed() {
    for (int i=0;i<5;i++) {
      Author a = new Author(); a.name = "A"+i; em.persist(a);
      for (int j=0;j<3;j++) { Book b = new Book(); b.title="B"+i+"_"+j; b.author=a; em.persist(b); }
    }
    em.flush(); em.clear();
  }

  @Test
  void plainQuery_hasNPlusOne() {
    stats().clear();
    var list = authors.findAllPlain();
    list.forEach(a -> a.books.size()); // инициализация LAZY
    long q = stats().getPrepareStatementCount();
    Assertions.assertTrue(q >= 1 + list.size(), "Ожидали N+1 запросов");
  }

  @Test
  void entityGraph_eliminatesNPlusOne() {
    stats().clear();
    var list = authors.findAllWithBooksGraph();
    list.forEach(a -> a.books.size());
    long q = stats().getPrepareStatementCount();
    Assertions.assertTrue(q <= 2, "Должно быть 1-2 запроса, а не N+1");
  }
}
```

**Kotlin — аналогичный тест**

```kotlin
package com.example.nplus1

import jakarta.persistence.*
import org.hibernate.SessionFactory
import org.hibernate.stat.Statistics
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.data.jpa.repository.EntityGraph
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.jpa.repository.config.EnableJpaRepositories
import org.springframework.transaction.annotation.Transactional

@Entity
class Author(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String? = null,
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    var books: MutableList<Book> = mutableListOf()
)
@Entity
class Book(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var title: String? = null,
    @ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "author_id")
    var author: Author? = null
)

interface AuthorRepository : JpaRepository<Author, Long> {
    @EntityGraph(attributePaths = ["books"])
    @Query("select a from Author a")
    fun findAllWithBooksGraph(): List<Author>

    @Query("select a from Author a")
    fun findAllPlain(): List<Author>
}

@DataJpaTest
@EnableJpaRepositories(considerNestedRepositories = true)
class NPlusOneIT {

    @Autowired lateinit var em: EntityManager
    @Autowired lateinit var authors: AuthorRepository

    private fun stats(): Statistics =
        em.entityManagerFactory.unwrap(SessionFactory::class.java).statistics

    @BeforeEach
    @Transactional
    fun seed() {
        repeat(5) { i ->
            val a = Author(name = "A$i")
            em.persist(a)
            repeat(3) { j ->
                em.persist(Book(title = "B${i}_$j", author = a))
            }
        }
        em.flush(); em.clear()
    }

    @Test
    fun plainQuery_hasNPlusOne() {
        stats().clear()
        val list = authors.findAllPlain()
        list.forEach { it.books.size }
        val q = stats().prepareStatementCount
        assertTrue(q >= 1 + list.size, "Ожидали N+1 запросов")
    }

    @Test
    fun entityGraph_eliminatesNPlusOne() {
        stats().clear()
        val list = authors.findAllWithBooksGraph()
        list.forEach { it.books.size }
        val q = stats().prepareStatementCount
        assertTrue(q <= 2, "Должно быть 1-2 запроса, а не N+1")
    }
}
```

---

## Инварианты БД: уникальные/`check`/FK — негативные тесты на нарушения

Никакая валидация на уровне бинов не заменит **жёстких ограничений БД**. Именно они — последняя линия обороны от гонок и «недобросовестных» клиентов. В тестах нам нужно удостовериться, что **уникальные ключи**, **CHECK-ограничения** и **внешние ключи** работают как задумано, а приложение корректно переводит SQL-ошибки в исключения Spring (`DataIntegrityViolationException`, `ConstraintViolationException` и т.д.).

Проверка **уникальности** должна покрывать и обычные, и «частичные» уникальные индексы (например, `where deleted_at is null` при soft-delete). В тесте мы вставляем две строки с одинаковым бизнес-ключом и убеждаемся, что вторая операция падает на `flush()`/`commit()`. Важно именно **форсировать flush**, потому что JPA буферизует вставки; без `flush()` исключение может прилететь вне теста.

**CHECK** — это доменные инварианты на уровне БД: «сумма >= 0», «дата окончания >= даты начала». Тесты на CHECK полезны, когда в доменной модели много путей изменения полей (несколько сервисов) и высок риск «забыть» валидацию. Мы намеренно сохраняем «плохую» сущность и убеждаемся, что БД её отвергает.

**FK** гарантирует ссылочную целостность. В тестах мы пытаемся сохранить «дитя» без родителя или удалить родителя при существующих детях (если FK без `ON DELETE ...`). Мы ожидаем, что БД откажет, и что исключение будет перехвачено переводчиком исключений Spring (в `@DataJpaTest` он активен).

Отдельный сценарий — **дефолты** и `NOT NULL`. Если столбец объявлен `NOT NULL`, а вы забыли поставить значение, JPA может попытаться вставить `null`. Тест ловит это на `flush()`. То же для «серверных» дефолтов: в миграции `default now()`, а в сущности — `null`. Тест даёт сигнал, что модель и DDL рассинхронизированы.

Удобно держать **фикстурные фабрики** для валидных сущностей и «инвалидов». Тогда тесты читаются: «добавляем валидного клиента → повторяем вставку с тем же email → ждём падения». Старайтесь не строить негативные тесты на «магических» id — сначала явно убедитесь, что «первый» записан.

Наконец, учтите **задержку проверки**. Некоторые ограничения могут быть отложенными (DEFERRABLE INITIALLY DEFERRED). В Postgres это опционально и по умолчанию `NOT DEFERRABLE`. Если вы используете отложенные FK/unique, вызовите `SET CONSTRAINTS ALL IMMEDIATE` перед `flush`, чтобы получить ошибку в тесте «здесь и сейчас».

**Java — негативные тесты: UNIQUE, CHECK, FK (с `flush()`)**

```java
package com.example.constraints;

import jakarta.persistence.*;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.transaction.annotation.Transactional;

@Entity
@Table(name="users_u", uniqueConstraints = @UniqueConstraint(columnNames = "email"))
class UserU {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
  @Column(nullable=false) String email;
  @Column(nullable=false) Long balanceCents; // CHECK (balance_cents >= 0) в DDL
}

@Entity
@Table(name="orders_fk")
class OrderFK {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
  @Column(nullable=false) Long userId; // FK на users_u(id)
}

@DataJpaTest
class ConstraintsIT {

  @PersistenceContext EntityManager em;

  @Test
  @Transactional
  void uniqueViolation_onDuplicateEmail() {
    var a = new UserU(); a.email="dup@example.com"; a.balanceCents=0L; em.persist(a);
    var b = new UserU(); b.email="dup@example.com"; b.balanceCents=0L; em.persist(b);
    Assertions.assertThrows(DataIntegrityViolationException.class, () -> {
      em.flush(); // выбросит исключение из-за UNIQUE
    });
  }

  @Test
  @Transactional
  void checkViolation_onNegativeBalance() {
    var u = new UserU(); u.email="bad@example.com"; u.balanceCents=-10L; em.persist(u);
    Assertions.assertThrows(DataIntegrityViolationException.class, () -> em.flush());
  }

  @Test
  @Transactional
  void fkViolation_onMissingParent() {
    var o = new OrderFK(); o.userId=999_999L; em.persist(o);
    Assertions.assertThrows(DataIntegrityViolationException.class, () -> em.flush());
  }
}
```

**Kotlin — негативные тесты**

```kotlin
package com.example.constraints

import jakarta.persistence.*
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.transaction.annotation.Transactional

@Entity
@Table(name = "users_u", uniqueConstraints = [UniqueConstraint(columnNames = ["email"])])
class UserU(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var email: String = "",
    @Column(nullable = false) var balanceCents: Long = 0
)

@Entity
@Table(name = "orders_fk")
class OrderFK(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var userId: Long = 0
)

@DataJpaTest
class ConstraintsIT(
    @PersistenceContext private val em: EntityManager
) {
    @Test
    @Transactional
    fun uniqueViolation_onDuplicateEmail() {
        em.persist(UserU(email = "dup@example.com", balanceCents = 0))
        em.persist(UserU(email = "dup@example.com", balanceCents = 0))
        assertThrows(DataIntegrityViolationException::class.java) { em.flush() }
    }

    @Test
    @Transactional
    fun checkViolation_onNegativeBalance() {
        em.persist(UserU(email = "bad@example.com", balanceCents = -10))
        assertThrows(DataIntegrityViolationException::class.java) { em.flush() }
    }

    @Test
    @Transactional
    fun fkViolation_onMissingParent() {
        em.persist(OrderFK(userId = 999_999))
        assertThrows(DataIntegrityViolationException::class.java) { em.flush() }
    }
}
```

---

## Контроль таймаутов/блокировок: сценарии дедлоков, пессимистические lock-тесты, «грязные» чтения

Производственные инциденты часто связаны не с «ошибками в коде», а с **подвисшими транзакциями**, **дедлоками** и слишком долгими запросами. Эти истории можно и нужно воспроизводить в тестах. В Postgres есть два важных параметра: `lock_timeout` (сколько ждать блокировку) и `statement_timeout` (сколько ждать выполнение запроса). В тестах мы можем установить их **локально на сессию** (`SET LOCAL ...`) и ожидать предсказуемый таймаут/исключение.

Сценарий **пессимистической блокировки**: поток A открывает транзакцию и делает `SELECT ... FOR UPDATE` по строке, поток B пытается сделать то же — и блокируется. Если `lock_timeout` короткий, поток B получит исключение `PessimisticLockingFailureException`/`LockTimeoutException`. Такой тест доказывает, что наш код правильно конфигурирован (мы не ждём вечность) и что у нас есть корректная обработка ошибок.

**Дедлок** воспроизводится, если два потока берут **разные строки в разном порядке** и пытаются взять вторую — в пересекающемся порядке. Постгрес обнаружит дедлок и убьёт одну транзакцию (выпадет `DeadlockLoserDataAccessException`). Важно синхронизировать потоки (через `CountDownLatch`/`CyclicBarrier`), чтобы гарантировать пересечение.

Про **«грязные» чтения**: в Postgres их нет — `READ UNCOMMITTED` фактически ведёт себя как `READ COMMITTED`. Это тоже полезно зафиксировать тестом: поток A обновляет значение и не коммитит, поток B читает и видит **старое** значение. Такой тест — «предохранитель» против иллюзий команды о «заглянуть в неподтверждённое».

Организационно такие тесты удобнее писать **на сервисе**, а не в тестовом классе напрямую. Сделайте методы `tx1()`/`tx2()` с нужной изоляцией и аннотациями `@Transactional`, и вызовите их из теста в отдельных потоках через `ExecutorService`. Так вы используете тот же прокси/аспект транзакций, что и в проде.

Чтобы **не виснуть**, перед блокирующей операцией выставляйте `SET LOCAL lock_timeout='1s'` и/или `SET LOCAL statement_timeout='2s'`. Это делается через `EntityManager.createNativeQuery(...).executeUpdate()` в начале транзакционного метода. Тогда даже если что-то пошло не по плану, тест завершится быстро и предсказуемо.

Наконец, держите такие тесты **детерминированными**. Категорически избегайте `Thread.sleep` как синхронизации; используйте барьеры/защёлки. И помните: эти тесты не должны гоняться в десятки потоков — достаточно двух потоков, чтобы воспроизвести нужный граф блокировок.

**Java — сервис для сценариев блокировок и тест с тайм-аутами/дедлоком**

```java
package com.example.locks;

import jakarta.persistence.*;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

@Entity
@Table(name="acc")
class Account {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
  @Column(nullable=false) Long balance;
}

interface AccountRepository extends org.springframework.data.jpa.repository.JpaRepository<Account, Long> { }

@Service
class LockService {
  @PersistenceContext EntityManager em;

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void lockRow(Long id, long lockMs) {
    em.createNativeQuery("set local lock_timeout = :t").setParameter("t", lockMs + "ms").executeUpdate();
    em.createQuery("select a from Account a where a.id=:id", Account.class)
      .setParameter("id", id)
      .setLockMode(LockModeType.PESSIMISTIC_WRITE)
      .getSingleResult(); // держим блокировку до конца транзакции
  }

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void tryUpdate(Long id, long lockMs) {
    em.createNativeQuery("set local lock_timeout = :t").setParameter("t", lockMs + "ms").executeUpdate();
    Account a = em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE);
    a.balance += 10;
  }
}

@SpringBootTest
class LockingIT {

  @Autowired LockService svc;
  @Autowired AccountRepository repo;

  @BeforeEach
  void init() { repo.deleteAll(); repo.save(new Account(){ { balance=100L; } }); }

  @Test
  void lockTimeout_whenRowAlreadyLocked() throws Exception {
    Long id = repo.findAll().get(0).id;

    ExecutorService es = Executors.newFixedThreadPool(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<?> f1 = es.submit(() -> { start.countDown(); svc.lockRow(id, 5000); }); // удерживает
    start.await();

    assertThatThrownBy(() -> svc.tryUpdate(id, 500))
      .isInstanceOfAny(org.springframework.dao.PessimisticLockingFailureException.class,
                       jakarta.persistence.LockTimeoutException.class);

    f1.get(2, TimeUnit.SECONDS);
    es.shutdown();
  }

  @Test
  void deadlock_oneLoser() throws Exception {
    // создадим две строки
    var a = repo.save(new Account(){ { balance=1L; } });
    var b = repo.save(new Account(){ { balance=2L; } });

    ExecutorService es = Executors.newFixedThreadPool(2);
    CyclicBarrier barrier = new CyclicBarrier(2);

    Callable<Void> t1 = () -> {
      svc.lockRow(a.id, 5000);
      barrier.await();
      svc.tryUpdate(b.id, 2000); // пытаемся взять вторую
      return null;
    };
    Callable<Void> t2 = () -> {
      svc.lockRow(b.id, 5000);
      barrier.await();
      svc.tryUpdate(a.id, 2000);
      return null;
    };

    Future<Void> r1 = es.submit(t1);
    Future<Void> r2 = es.submit(t2);

    int failures = 0;
    for (Future<Void> r : new Future[]{r1, r2}) {
      try { r.get(); } catch (ExecutionException ex) { failures++; }
    }
    Assertions.assertEquals(1, failures, "Ожидали одного проигравшего в дедлоке");
    es.shutdown();
  }
}
```

**Kotlin — тот же сценарий**

```kotlin
package com.example.locks

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Propagation
import org.springframework.transaction.annotation.Transactional
import java.util.concurrent.*

@Entity
@Table(name = "acc")
class Account(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var balance: Long = 0
)

interface AccountRepository : JpaRepository<Account, Long>

@Service
class LockService(
    @PersistenceContext private val em: EntityManager
) {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun lockRow(id: Long, lockMs: Long) {
        em.createNativeQuery("set local lock_timeout = :t").setParameter("t", "${lockMs}ms").executeUpdate()
        em.createQuery("select a from Account a where a.id=:id", Account::class.java)
            .setParameter("id", id)
            .setLockMode(LockModeType.PESSIMISTIC_WRITE)
            .singleResult
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun tryUpdate(id: Long, lockMs: Long) {
        em.createNativeQuery("set local lock_timeout = :t").setParameter("t", "${lockMs}ms").executeUpdate()
        val a = em.find(Account::class.java, id, LockModeType.PESSIMISTIC_WRITE)
        a.balance += 10
    }
}

@SpringBootTest
class LockingIT(
    private val svc: LockService,
    private val repo: AccountRepository
) {
    @BeforeEach
    fun init() { repo.deleteAll(); repo.save(Account(balance = 100)) }

    @Test
    fun lockTimeout_whenRowAlreadyLocked() {
        val id = repo.findAll().first().id!!

        val start = CountDownLatch(1)
        val es = Executors.newFixedThreadPool(2)
        val f1 = es.submit<Void?> {
            start.countDown()
            svc.lockRow(id, 5000)
            null
        }
        start.await()

        assertThatThrownBy { svc.tryUpdate(id, 500) }
            .isInstanceOfAny(
                org.springframework.dao.PessimisticLockingFailureException::class.java,
                jakarta.persistence.LockTimeoutException::class.java
            )

        f1.get(2, TimeUnit.SECONDS)
        es.shutdown()
    }

    @Test
    fun deadlock_oneLoser() {
        val a = repo.save(Account(balance = 1))
        val b = repo.save(Account(balance = 2))

        val es = Executors.newFixedThreadPool(2)
        val barrier = CyclicBarrier(2)

        val t1 = Callable {
            svc.lockRow(a.id!!, 5000)
            barrier.await()
            svc.tryUpdate(b.id!!, 2000)
            null
        }
        val t2 = Callable {
            svc.lockRow(b.id!!, 5000)
            barrier.await()
            svc.tryUpdate(a.id!!, 2000)
            null
        }

        val r1 = es.submit(t1)
        val r2 = es.submit(t2)

        var failures = 0
        listOf(r1, r2).forEach {
            try { it.get() } catch (e: ExecutionException) { failures++ }
        }
        assertEquals(1, failures, "Ожидали одного проигравшего в дедлоке")
        es.shutdown()
    }
}
```

**Проверка «грязных» чтений (для Postgres — отсутствие dirty read) — идея теста**

*Суть: Тx1 начало, обновило баланс, не закоммитило. Тx2 читает и видит старое значение. В Postgres READ UNCOMMITTED == READ COMMITTED.*

```java
// В сервисе:
@Transactional(propagation = Propagation.REQUIRES_NEW, isolation = Isolation.READ_UNCOMMITTED)
public long readBalance(Long id) {
  return em.find(Account.class, id).balance;
}
```

В тесте: `tx1` меняет баланс и «спит» до команды коммита; `tx2` читает и сравнивает со старым значением; убеждаемся, что грязного чтения **нет** (видим старое). Это закрепляет знание о семантике PG.

---

### Резюме по подтеме

1. Берите **Testcontainers** и гоняйте тесты на реальной СУБД с реальными миграциями.
2. Ловите **N+1** и проверяйте **пагинацию/графы** через Hibernate Statistics; фиксируйте стабильную сортировку.
3. Тестируйте **инварианты БД** негативными сценариями и форсируйте `flush()` для детерминизма.
4. Репродуцируйте **блокировки/дедлоки** в двух потоках, ставьте локальные таймауты, и документируйте отсутствие **dirty read** в PG.
















