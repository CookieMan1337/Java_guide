---
layout: page
title: "Spring Data JPA: Углубленно в JDBC"
permalink: /spring/jpa_deep
---

<!-- TOC -->
* [1. Производительность загрузки и N+1](#1-производительность-загрузки-и-n1)
  * [Причины N+1, диагностика по SQL-логам и статистике Hibernate](#причины-n1-диагностика-по-sql-логам-и-статистике-hibernate)
  * [Инструменты: JOIN FETCH, @EntityGraph, @BatchSize, hibernate.default_batch_fetch_size, SUBSELECT](#инструменты-join-fetch-entitygraph-batchsize-hibernatedefault_batch_fetch_size-subselect)
  * [Стратегии: «тонкие» DTO-проекции для списков, точечные графы для карточек, разделение чтения/деталей](#стратегии-тонкие-dto-проекции-для-списков-точечные-графы-для-карточек-разделение-чтениядеталей)
* [2. Проекции и сложные запросы](#2-проекции-и-сложные-запросы)
  * [Проекции: интерфейсные/DTO-конструкторы в Spring Data, закрытые vs открытые (SpEL) и их стоимость](#проекции-интерфейсныеdto-конструкторы-в-spring-data-закрытые-vs-открытые-spel-и-их-стоимость)
  * [Нативные SQL: маппинг на DTO/интерфейсы, SqlResultSetMapping (где оправдано)](#нативные-sql-маппинг-на-dtoинтерфейсы-sqlresultsetmapping-где-оправдано)
  * [Частичные обновления: @Modifying JPQL/SQL, возвращаемые счётчики, осторожно с кешами и L1-контекстом](#частичные-обновления-modifying-jpqlsql-возвращаемые-счётчики-осторожно-с-кешами-и-l1-контекстом)
* [3. Вставки/обновления батчами и генерация идентификаторов](#3-вставкиобновления-батчами-и-генерация-идентификаторов)
  * [Влияние стратегий @GeneratedValue на батчи (SEQUENCE vs IDENTITY; оптимизаторы pooled/pooled-lo)](#влияние-стратегий-generatedvalue-на-батчи-sequence-vs-identity-оптимизаторы-pooledpooled-lo)
  * [Тюнинг батчей: hibernate.jdbc.batch_size, order_inserts/updates, драйверные оптимизации (для PG — reWriteBatchedInserts)](#тюнинг-батчей-hibernatejdbcbatch_size-order_insertsupdates-драйверные-оптимизации-для-pg--rewritebatchedinserts)
  * [Массовые операции: bulk update/delete через JPQL — обход 1-го уровня, необходимость ручной синхронизации](#массовые-операции-bulk-updatedelete-через-jpql--обход-1-го-уровня-необходимость-ручной-синхронизации)
* [4. Транзакции, изоляция и блокировки](#4-транзакции-изоляция-и-блокировки)
  * [Границы и изоляция: поведение `readOnly=true` как хинт, влияние `FlushMode` (COMMIT/MANUAL)](#границы-и-изоляция-поведение-readonlytrue-как-хинт-влияние-flushmode-commitmanual)
  * [Оптимистическая блокировка `@Version`: обработка конфликтов, повтор операций](#оптимистическая-блокировка-version-обработка-конфликтов-повтор-операций)
  * [Пессимистические блокировки: `LockModeType.PESSIMISTIC_READ/WRITE`, таймауты/эскалации, риск дедлоков](#пессимистические-блокировки-lockmodetypepessimistic_readwrite-таймаутыэскалации-риск-дедлоков)
  * [Консистентность чтения: repeatable-read vs snapshot семантика СУБД, где важны «снимки»](#консистентность-чтения-repeatable-read-vs-snapshot-семантика-субд-где-важны-снимки)
* [5. Управление Persistence Context](#5-управление-persistence-context)
  * [Когда уместно `clear()/detach()`, «длинные» сессии и утечки памяти](#когда-уместно-cleardetach-длинные-сессии-и-утечки-памяти)
  * [Read-only оптимизации: `Query.setReadOnly(true)`, «immutable»-сущности, `StatelessSession` (узкие случаи)](#read-only-оптимизации-querysetreadonlytrue-immutable-сущности-statelesssession-узкие-случаи)
  * [Кэш 1-го уровня как защита от двойных SELECT, но источник «устаревших» данных — критерии и баланс](#кэш-1-го-уровня-как-защита-от-двойных-select-но-источник-устаревших-данных--критерии-и-баланс)
* [6. Сложные маппинги и модель](#6-сложные-маппинги-и-модель)
  * [Наследование: SINGLE_TABLE vs JOINED vs TABLE_PER_CLASS — компромиссы производительности/нормализации](#наследование-single_table-vs-joined-vs-table_per_class--компромиссы-производительностинормализации)
  * [Связи: избегаем «грузных» `@ManyToMany` — предпочтительна явная сущность связи (join-entity)](#связи-избегаем-грузных-manytomany--предпочтительна-явная-сущность-связи-join-entity)
  * [Встроенные значения: `@Embeddable` для value-объектов; `@MappedSuperclass` для общих полей, не для связей](#встроенные-значения-embeddable-для-value-объектов-mappedsuperclass-для-общих-полей-не-для-связей)
  * [События/слушатели: `@PrePersist/@PreUpdate` и entity-listener’ы — минимум логики, без I/O](#событияслушатели-prepersistpreupdate-и-entity-listenerы--минимум-логики-без-io)
* [7. Кеширование 2-го уровня и кеш запросов](#7-кеширование-2-го-уровня-и-кеш-запросов)
  * [Когда имеет смысл L2: справочники/редко меняемые сущности; регионы, режимы (read-only/non-strict/transactional)](#когда-имеет-смысл-l2-справочникиредко-меняемые-сущности-регионы-режимы-read-onlynon-stricttransactional)
  * [Кеш запросов: риск «несогласованности», ключи инвалидации, осторожно с частыми изменениями](#кеш-запросов-риск-несогласованности-ключи-инвалидации-осторожно-с-частыми-изменениями)
  * [Распределённость: репликация/инвалидация между инстансами, влияние на память и GC](#распределённость-репликацияинвалидация-между-инстансами-влияние-на-память-и-gc)
* [8. Чтение больших объёмов и стриминг](#8-чтение-больших-объёмов-и-стриминг)
  * [Потоки из репозиториев (`Stream<T>`), требования к `@Transactional` и закрытию](#потоки-из-репозиториев-streamt-требования-к-transactional-и-закрытию)
  * [Скролл/курсор: forward-only, `fetchSize`/`setFetchSize`, минимизация памяти](#скроллкурсор-forward-only-fetchsizesetfetchsize-минимизация-памяти)
  * [Read-only транзакции + хинты драйвера; выгрузка в файлы/каналы без материализации всего списка](#read-only-транзакции--хинты-драйвера-выгрузка-в-файлыканалы-без-материализации-всего-списка)
* [9. DB-специфика и нестандартные типы](#9-db-специфика-и-нестандартные-типы)
  * [JSON/JSONB, массивы, диапазоны, `hstore`: `@Converter` vs Hibernate Types; индексы (например, GIN для JSONB)](#jsonjsonb-массивы-диапазоны-hstore-converter-vs-hibernate-types-индексы-например-gin-для-jsonb)
  * [Денежные/точные типы: `BigDecimal` + scale/rounding; контроль сериализации в JSON](#денежныеточные-типы-bigdecimal--scalerounding-контроль-сериализации-в-json)
  * [Частичные/функциональные индексы, `UNIQUE` под soft-delete, «виртуальные» столбцы/представления (`@Immutable`, `@Subselect`)](#частичныефункциональные-индексы-unique-под-soft-delete-виртуальные-столбцыпредставления-immutable-subselect)
* [10. Аудит и «мягкое удаление»](#10-аудит-и-мягкое-удаление)
  * [Auditing: `@CreatedDate/@LastModifiedDate`, `AuditingEntityListener`, пользователи/тенанты](#auditing-createddatelastmodifieddate-auditingentitylistener-пользователитенанты)
  * [Soft-delete: флаг + глобальные фильтры (`@Where/@SQLDelete` или фильтры Hibernate), последствия для уникальных ключей/джойнов](#soft-delete-флаг--глобальные-фильтры-wheresqldelete-или-фильтры-hibernate-последствия-для-уникальных-ключейджойнов)
  * [История изменений: версионирование записей vs отдельные audit-таблицы; компромисс сложность/польза](#история-изменений-версионирование-записей-vs-отдельные-audit-таблицы-компромисс-сложностьпольза)
* [11. Спецификации, Criteria и QueryDSL](#11-спецификации-criteria-и-querydsl)
  * [`Specification<T>`: динамические фильтры, композиция условий, пагинация+сортировка](#specificationt-динамические-фильтры-композиция-условий-пагинациясортировка)
  * [Criteria API: типобезопасность vs шум кода — где помогает](#criteria-api-типобезопасность-vs-шум-кода--где-помогает)
  * [QueryDSL: предикаты, join-ы, подзапросы, «тяжёлые» фильтры — когда лучше, чем строковый JPQL](#querydsl-предикаты-join-ы-подзапросы-тяжёлые-фильтры--когда-лучше-чем-строковый-jpql)
* [12. JDBC и jOOQ: когда уходить ниже](#12-jdbc-и-jooq-когда-уходить-ниже)
  * [Где JPA неэффективна: отчёты, агрегаты/окна, сложные CTE, апсерты/`ON CONFLICT`, bulk-операции](#где-jpa-неэффективна-отчёты-агрегатыокна-сложные-cte-апсертыon-conflict-bulk-операции)
  * [`JdbcTemplate/NamedParameterJdbcTemplate`: `RowMapper`, батчи `batchUpdate`, `SimpleJdbcInsert`, таймауты и `fetchSize`](#jdbctemplatenamedparameterjdbctemplate-rowmapper-батчи-batchupdate-simplejdbcinsert-таймауты-и-fetchsize)
  * [Потоковая обработка: курсоры/стримы, контроль памяти, работа с LOB](#потоковая-обработка-курсорыстримы-контроль-памяти-работа-с-lob)
  * [jOOQ как «типобезопасный SQL»: генерация DSL из схемы, тонкая оптимизация под конкретную СУБД; совместное использование с JPA в одном сервисе](#jooq-как-типобезопасный-sql-генерация-dsl-из-схемы-тонкая-оптимизация-под-конкретную-субд-совместное-использование-с-jpa-в-одном-сервисе)
* [13. Репозитории: расширение и кастомизация](#13-репозитории-расширение-и-кастомизация)
  * [Свой базовый интерфейс репозитория: общие методы, хук на `EntityManager`](#свой-базовый-интерфейс-репозитория-общие-методы-хук-на-entitymanager)
  * [Кастомные имплементации для отдельных репозиториев (surgical-SQL, спец-кеши)](#кастомные-имплементации-для-отдельных-репозиториев-surgical-sql-спец-кеши)
  * [«Граница ответственности» репозитория: тонкие методы vs «толстые» сервисы — баланс читаемости и производительности](#граница-ответственности-репозитория-тонкие-методы-vs-толстые-сервисы--баланс-читаемости-и-производительности)
* [14. Тестирование продвинутого persistence](#14-тестирование-продвинутого-persistence)
  * [Testcontainers с реальной СУБД, прогон миграций перед тестами, сидирование данных](#testcontainers-с-реальной-субд-прогон-миграций-перед-тестами-сидирование-данных)
  * [Проверки на N+1/пагинацию/графы загрузки; сбор статистики Hibernate в тестах](#проверки-на-n1пагинациюграфы-загрузки-сбор-статистики-hibernate-в-тестах)
  * [Инварианты БД: уникальные/`check`/FK — негативные тесты на нарушения](#инварианты-бд-уникальныеcheckfk--негативные-тесты-на-нарушения)
  * [Контроль таймаутов/блокировок: сценарии дедлоков, пессимистические lock-тесты, «грязные» чтения](#контроль-таймаутовблокировок-сценарии-дедлоков-пессимистические-lock-тесты-грязные-чтения)
<!-- TOC -->

# 1. Производительность загрузки и N+1

Эта подтема — про то, почему JPA/Hibernate могут внезапно «стрелять себе в ногу» по производительности, откуда берётся N+1, как его увидеть глазами и числами, и какие есть базовые приёмы, чтобы не утонуть в лавине SQL-запросов. Всё это напрямую упирается в JDBC: каждый отдельный SELECT — это отдельный round-trip к базе, отдельные allocation’ы в драйвере, отдельные чтения из сокета.

---

## Причины N+1, диагностика по SQL-логам и статистике Hibernate

Первое, что нужно усвоить про N+1: это не «магия ORM», а следствие ленивой загрузки и того, как вы ходите по графу объектов в коде. Классический сценарий — вы загружаете список сущностей `Order`, а затем в цикле обращаетесь к их ассоциациям `orderItems`. Hibernate по умолчанию делает один запрос на список заказов (это «1»), а затем по одному отдельному запросу на каждую коллекцию (это «N»). На уровне JDBC это выглядит как десятки или сотни SELECT’ов подряд, каждый из которых мог бы быть частью одного более крупного запроса.

Причина такого поведения чаще всего кроется в комбинации `FetchType.LAZY` и факта, что вы проходите по ассоциации уже после того, как основной список был загружен. Ленивая загрузка сама по себе не зло, напротив — это важный инструмент. Проблема начинается тогда, когда вы многократно запускаете ленивую загрузку в одном и том же паттерне: например, пробегаетесь по коллекции сущностей в контроллере, который сериализует их в JSON, и Jackson, дергая геттеры, триггерит N дополнительных запросов.

Диагностировать N+1 без логов почти нереально: на уровне Java-кода всё выглядит красиво, сущности работают как обычные объекты, а под капотом в JDBC крутится карусель. Поэтому первый шаг — включить логирование SQL. В Spring Boot достаточно добавить логгер для `org.hibernate.SQL` и, по желанию, `org.hibernate.type.descriptor.sql.BasicBinder`, чтобы видеть параметры. В `application.yml` это задаётся настройками уровня логов.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
logging:
  level:
    org.hibernate.SQL: debug
    org.hibernate.type.descriptor.sql.BasicBinder: trace
```

Теперь, если у нас есть сущности `Customer` и `Order`, мы легко увидим N+1. Например, вот простое маппирование, которое в вакууме выглядит вполне невинно, но при наивном использовании приведёт к проблеме.

```java
package com.example.jpa.nplusone;

import jakarta.persistence.*;
import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders;

    // getters/setters
}

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    private String description;

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;

    // getters/setters
}
```

```kotlin
package com.example.jpa.nplusone

import jakarta.persistence.*

@Entity
@Table(name = "customers")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = "",

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    var orders: List<Order> = emptyList()
)

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var description: String = "",

    @ManyToOne(fetch = FetchType.LAZY)
    var customer: Customer? = null
)
```

Представьте сервис, который загружает всех клиентов и для каждого печатает количество заказов. В коде это кажется «одной операцией», но из-за ленивых коллекций Hibernate будет поднимать коллекции по одной.

```java
package com.example.jpa.nplusone;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class CustomerService {

    private final CustomerRepository repository;

    public CustomerService(CustomerRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public void printCustomerOrdersCount() {
        List<Customer> customers = repository.findAll();
        customers.forEach(customer -> {
            int count = customer.getOrders().size();
            System.out.println(customer.getName() + " has " + count + " orders");
        });
    }
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class CustomerService(
    private val repository: CustomerRepository
) {

    @Transactional(readOnly = true)
    fun printCustomerOrdersCount() {
        val customers = repository.findAll()
        customers.forEach { customer ->
            val count = customer.orders.size
            println("${customer.name} has $count orders")
        }
    }
}
```

При включенных логах вы увидите сначала один SELECT по таблице `customers`, а затем серию SELECT’ов по `orders`, по одному на каждый `customer`. Чем больше клиентов, тем сильнее лавина запросов. Диагностировать это можно глазами, но как только запросов становится много, лучше включить статистику Hibernate, чтобы получать агрегированные показатели.

Hibernate умеет считать количество запросов, хиты/промахи кэша и пр. Достаточно включить `hibernate.generate_statistics=true` в конфигурации. В Spring Boot это делается всё там же, в `application.yml`, а потом можно инжектить `SessionFactory` и вытаскивать из него `Statistics`.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

```java
package com.example.jpa.nplusone;

import jakarta.persistence.EntityManagerFactory;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.stereotype.Component;

@Component
public class HibernateStatsLogger {

    private final SessionFactory sessionFactory;

    public HibernateStatsLogger(EntityManagerFactory entityManagerFactory) {
        this.sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
    }

    public void logStats() {
        Statistics stats = sessionFactory.getStatistics();
        System.out.println("Queries executed: " + stats.getQueryExecutionCount());
    }
}
```

```kotlin
package com.example.jpa.nplusone

import jakarta.persistence.EntityManagerFactory
import org.hibernate.SessionFactory
import org.springframework.stereotype.Component

@Component
class HibernateStatsLogger(
    entityManagerFactory: EntityManagerFactory
) {

    private val sessionFactory: SessionFactory =
        entityManagerFactory.unwrap(SessionFactory::class.java)

    fun logStats() {
        val stats = sessionFactory.statistics
        println("Queries executed: ${stats.queryExecutionCount}")
    }
}
```

Если вы вызываете сервисный метод, который, казалось бы, просто возвращает список DTO, но статистика показывает сотни запросов — это почти наверняка N+1, вызванный ленивыми ассоциациями. Важно смотреть не только на количество запросов, но и на их структуру: одинаковые SELECT’ы с разными значениям параметров — явный признак N+1.

Отдельный интерес — N+1 на стороне `ManyToOne`: когда у вас список заказов, и каждый тянет своего клиента. Здесь тоже легко получить один SELECT по таблице заказов и N отдельных запросов по клиентам, если вы где-то в коде обращаетесь к `order.getCustomer().getName()`. Диагностируется это так же, логами и статистикой, а лечится похожими приёмами, но важно осознавать, что N+1 может жить и в «обратную сторону» связи.

Ещё один источник проблем — ленивые коллекции в связке с сериализацией JSON. Если вы отдаёте наружу сущности как есть, без DTO, Jackson, дергая геттеры, не видит разницы между обычным списком и лениво загружаемой коллекцией. В ответ на один HTTP-запрос ваш сервис может сделать десятки SQL, которые вы не видите в коде контроллера. Поэтому включённые SQL-логи и статистика Hibernate — обязательный инструмент разработки, а не опция «на случай проблем».

---

## Инструменты: JOIN FETCH, @EntityGraph, @BatchSize, hibernate.default_batch_fetch_size, SUBSELECT

После того как вы увидели N+1, возникает вопрос «что с этим делать». Первый и самый известный инструмент — `JOIN FETCH` в JPQL. В отличие от обычного `JOIN`, который просто присоединяет таблицу, `JOIN FETCH` говорит Hibernate: «загрузи ассоциацию в рамках этого запроса и положи её в граф сущностей». На уровне SQL это просто join, но на уровне ORM — заполнение ассоциации, чтобы позже не было дополнительных SELECT’ов.

Простейший пример — репозиторий, который возвращает клиентов сразу с заказами. В Spring Data JPA это делается через `@Query` и `LEFT JOIN FETCH`. Такой метод вытаскивает весь граф одним запросом, а ленивые коллекции уже не триггерят N+1 при обходе.

```java
package com.example.jpa.nplusone;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface CustomerRepository extends JpaRepository<Customer, Long> {

    @Query("select c from Customer c left join fetch c.orders")
    List<Customer> findAllWithOrders();
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface CustomerRepository : JpaRepository<Customer, Long> {

    @Query("select c from Customer c left join fetch c.orders")
    fun findAllWithOrders(): List<Customer>
}
```

Однако у `JOIN FETCH` есть жёсткие ограничения. Во-первых, такой запрос нельзя нормально сочетать с пагинацией по коллекциям: результирующий SQL дублирует строки по числу элементов коллекции, и `LIMIT/OFFSET` начинает резать не по сущностям, а по строкам. Spring Data прямо запрещает делать `Page<Customer>` с `join fetch` коллекции. Для единичных сущностей (`findById`) это отлично, для списков — нужен более аккуратный подход.

Во-вторых, есть проблема «нескольких мешков» (multiple bag fetch exception): Hibernate не позволяет в одном запросе `JOIN FETCH`ить несколько коллекций типа `List` или `bag`. Всё из-за того же дублирования строк: ORM перестаёт понимать, как восстановить уникальные коллекции. Частично это лечится заменой `List` на `Set` и явными `@OrderColumn`, частично — отказом от нескольких коллекций в одном запросе и использованием батч-стратегий.

`@EntityGraph` и `@NamedEntityGraph` дают более декларативный способ описать «что именно подгружать». Вместо того, чтобы писать `JOIN FETCH` руками в каждом запросе, вы объявляете граф на уровне сущности, а затем используете его в репозитории. Это удобно, когда нужно несколько разных представлений одной и той же сущности: например, «только заказы» или «заказы с товарами и платежами».

```java
package com.example.jpa.nplusone;

import jakarta.persistence.*;
import java.util.List;

@Entity
@Table(name = "customers")
@NamedEntityGraph(
        name = "Customer.withOrders",
        attributeNodes = @NamedAttributeNode("orders")
)
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders;

    // getters/setters
}
```

```kotlin
package com.example.jpa.nplusone

import jakarta.persistence.*

@Entity
@Table(name = "customers")
@NamedEntityGraph(
    name = "Customer.withOrders",
    attributeNodes = [NamedAttributeNode("orders")]
)
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = "",

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    var orders: List<Order> = emptyList()
)
```

В репозитории это используется просто: добавляете `@EntityGraph` на метод. Spring Data сам построит запрос с правильным fetch-планом. Это удобно для «карточек», где вы загружаете одну сущность и хотите сразу получить «хвост» ассоциаций, не думая о JPQL.

```java
package com.example.jpa.nplusone;

import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface CustomerRepository extends JpaRepository<Customer, Long> {

    @EntityGraph(value = "Customer.withOrders", type = EntityGraph.EntityGraphType.LOAD)
    Optional<Customer> findWithOrdersById(Long id);
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.data.jpa.repository.EntityGraph
import org.springframework.data.jpa.repository.JpaRepository
import java.util.Optional

interface CustomerRepository : JpaRepository<Customer, Long> {

    @EntityGraph(value = "Customer.withOrders", type = EntityGraph.EntityGraphType.LOAD)
    fun findWithOrdersById(id: Long): Optional<Customer>
}
```

`@BatchSize` и глобальный `hibernate.default_batch_fetch_size` — другой подход. Они не убирают ленивую загрузку, но уменьшают количество запросов при N+1. Вместо N отдельных SELECT’ов Hibernate делает запросы пачками: когда вы обращаетесь к коллекции у первого клиента, он заодно подгружает коллекции ещё у нескольких клиентов. В итоге N+1 превращается в M+1, где M намного меньше N.

```java
package com.example.jpa.nplusone;

import jakarta.persistence.*;
import org.hibernate.annotations.BatchSize;

import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @BatchSize(size = 20)
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders;

    // getters/setters
}
```

```kotlin
package com.example.jpa.nplusone

import jakarta.persistence.*
import org.hibernate.annotations.BatchSize

@Entity
@Table(name = "customers")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = "",

    @BatchSize(size = 20)
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    var orders: List<Order> = emptyList()
)
```

Глобальная настройка `hibernate.default_batch_fetch_size` работает похожим образом для всех ленивых ассоциаций. Вы просто говорите Hibernate: «если нужно подгрузить ленивые ссылки, делай это пачками по N штук». Эта настройка особенно полезна для `ManyToOne`, когда вы загружаете много заказов и хотите одним запросом подтянуть клиентов.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 20
```

`FetchMode.SUBSELECT` (через `@Fetch(FetchMode.SUBSELECT)`) — ещё один инструмент от Hibernate. Он говорит: «подгрузи коллекции для всех уже загруженных родителей одним большим подзапросом». Алгоритм такой: сначала ORM выполняет SELECT по родителям (например, `select * from customers where ...`), а затем один дополнительный запрос вида `select * from orders where customer_id in (ids...)`. Это особенно эффективно, когда вы разово загружаете большую страницу сущностей и хотите одним махом подтянуть их коллекции.

```java
package com.example.jpa.nplusone;

import jakarta.persistence.*;
import org.hibernate.annotations.Fetch;
import org.hibernate.annotations.FetchMode;

import java.util.List;

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Fetch(FetchMode.SUBSELECT)
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders;

    // getters/setters
}
```

```kotlin
package com.example.jpa.nplusone

import jakarta.persistence.*
import org.hibernate.annotations.Fetch
import org.hibernate.annotations.FetchMode

@Entity
@Table(name = "customers")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = "",

    @Fetch(FetchMode.SUBSELECT)
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    var orders: List<Order> = emptyList()
)
```

У каждого инструмента есть зона применимости. `JOIN FETCH` и `@EntityGraph` хороши для единичных сущностей и «карточек», где нужно получить всё и сразу. `@BatchSize` и `default_batch_fetch_size` — для списков, где вы не хотите тянуть всё, но и N+1 терпеть не готовы. `SUBSELECT` — компромисс для большой страницы, когда вы готовы на один тяжёлый запрос вместо множества мелких. На практике вы обычно комбинируете подходы: для «карточек» — графы и `join fetch`, для списков — батчи и тонкие выборки.

---

## Стратегии: «тонкие» DTO-проекции для списков, точечные графы для карточек, разделение чтения/деталей

Один из ключевых архитектурных выводов из темы N+1: нельзя обрабатывать все сценарии чтения через один и тот же репозиторий и один и тот же набор сущностей. Потребности списка и «карточки» принципиально разные. Для списка вам достаточно лёгкого DTO с несколькими полями, а для карточки — полного графа сущности с ассоциациями. Попытка использовать один и тот же метод `findAll()` для обоих сценариев почти гарантированно приведёт к либо избыточной загрузке, либо к N+1.

Для списков разумно использовать «тонкие» DTO-проекции. В Spring Data JPA это легко делается через интерфейсные или конструкторные проекции. Например, если на списке клиентов вам нужны только id, имя и количество заказов, нет смысла тянуть весь список `Order` как сущности. Можно сделать проекцию, которая агрегирует эти данные на стороне базы. Это уменьшает размер результата, нагрузку на JPA-мэппинг и снижает риск N+1.

```java
package com.example.jpa.nplusone;

public interface CustomerSummary {

    Long getId();

    String getName();

    long getOrdersCount();
}
```

```java
package com.example.jpa.nplusone;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface CustomerSummaryRepository extends JpaRepository<Customer, Long> {

    @Query("""
        select c.id as id,
               c.name as name,
               count(o) as ordersCount
        from Customer c
        left join c.orders o
        group by c.id, c.name
        """)
    List<CustomerSummary> findAllSummaries();
}
```

```kotlin
package com.example.jpa.nplusone

interface CustomerSummary {
    fun getId(): Long
    fun getName(): String
    fun getOrdersCount(): Long
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface CustomerSummaryRepository : JpaRepository<Customer, Long> {

    @Query(
        """
        select c.id as id,
               c.name as name,
               count(o) as ordersCount
        from Customer c
        left join c.orders o
        group by c.id, c.name
        """
    )
    fun findAllSummaries(): List<CustomerSummary>
}
```

Такой репозиторий возвращает уже агрегированные данные, и никакого N+1 здесь быть не может: у вас один запрос с join’ом и group by, и результат маппится напрямую в интерфейс. Для контроллера это просто список лёгких DTO, которые не тащат за собой граф сущностей и не триггерят дополнительные запросы при сериализации.

Для карточки — отдельный путь. Здесь как раз уместны `@EntityGraph` или `JOIN FETCH`, потому что вы хотите получить одного клиента со всеми его заказами (а иногда и с вложенными сущностями вроде платежей). Это делается отдельным методом в репозитории, который не используется в списковых сценариях, чтобы случайно не выстрелить себе в ногу.

```java
package com.example.jpa.nplusone;

import java.util.Optional;

import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CustomerCardRepository extends JpaRepository<Customer, Long> {

    @EntityGraph(attributePaths = {"orders"})
    Optional<Customer> findDetailedById(Long id);
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.data.jpa.repository.EntityGraph
import org.springframework.data.jpa.repository.JpaRepository
import java.util.Optional

interface CustomerCardRepository : JpaRepository<Customer, Long> {

    @EntityGraph(attributePaths = ["orders"])
    fun findDetailedById(id: Long): Optional<Customer>
}
```

Дальнейший шаг — разделить сервисный слой по типам запросов. Один сервис отвечает за списки и использует только DTO-проекции, другой — за карточки и использует графы загрузки. Это не полный CQRS, но уже полезное разделение. Такой подход не только улучшает производительность, но и делает код понятнее: по названию метода (`getCustomerSummaries`, `getCustomerCard`) понятно, что он делает и какой объём данных тащит.

```java
package com.example.jpa.nplusone;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class CustomerReadService {

    private final CustomerSummaryRepository summaryRepository;
    private final CustomerCardRepository cardRepository;

    public CustomerReadService(
            CustomerSummaryRepository summaryRepository,
            CustomerCardRepository cardRepository
    ) {
        this.summaryRepository = summaryRepository;
        this.cardRepository = cardRepository;
    }

    @Transactional(readOnly = true)
    public List<CustomerSummary> listCustomers() {
        return summaryRepository.findAllSummaries();
    }

    @Transactional(readOnly = true)
    public Customer getCard(Long id) {
        return cardRepository.findDetailedById(id)
                .orElseThrow(() -> new IllegalArgumentException("Customer not found"));
    }
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class CustomerReadService(
    private val summaryRepository: CustomerSummaryRepository,
    private val cardRepository: CustomerCardRepository
) {

    @Transactional(readOnly = true)
    fun listCustomers(): List<CustomerSummary> =
        summaryRepository.findAllSummaries()

    @Transactional(readOnly = true)
    fun getCard(id: Long): Customer =
        cardRepository.findDetailedById(id)
            .orElseThrow { IllegalArgumentException("Customer not found") }
}
```

Важно также не скармливать наружу сущности напрямую. Если JSON-контроллер возвращает `Customer`, при сериализации Jackson может залезть во все ассоциации, включая ленивые. Это не только N+1, но и утечки внутренней модели. Лучше делать явные DTO и маппинг из сущности в DTO, а для списков — вообще обходиться проекциями, минуя сущности целиком.

```java
package com.example.jpa.nplusone;

public record CustomerCardDto(Long id, String name, int ordersCount) {

    public static CustomerCardDto from(Customer customer) {
        return new CustomerCardDto(
                customer.getId(),
                customer.getName(),
                customer.getOrders().size()
        );
    }
}
```

```java
package com.example.jpa.nplusone;

import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    private final CustomerReadService service;

    public CustomerController(CustomerReadService service) {
        this.service = service;
    }

    @GetMapping
    public List<CustomerSummary> list() {
        return service.listCustomers();
    }

    @GetMapping("/{id}")
    public CustomerCardDto card(@PathVariable Long id) {
        return CustomerCardDto.from(service.getCard(id));
    }
}
```

```kotlin
package com.example.jpa.nplusone

data class CustomerCardDto(
    val id: Long,
    val name: String,
    val ordersCount: Int
) {
    companion object {
        fun from(customer: Customer): CustomerCardDto =
            CustomerCardDto(
                id = customer.id!!,
                name = customer.name,
                ordersCount = customer.orders.size
            )
    }
}
```

```kotlin
package com.example.jpa.nplusone

import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/customers")
class CustomerController(
    private val service: CustomerReadService
) {

    @GetMapping
    fun list(): List<CustomerSummary> =
        service.listCustomers()

    @GetMapping("/{id}")
    fun card(@PathVariable id: Long): CustomerCardDto =
        CustomerCardDto.from(service.getCard(id))
}
```

Разделение чтения и деталей имеет ещё один приятный эффект: вы легче контролируете индексы и нагрузку на базу. Для списков вы знаете, какие поля участвуют в фильтрации и сортировке, и можете под них строить индексы; для карточек — можете позволить себе более тяжёлые join’ы, потому что запросов будет меньше. Вместо того чтобы пытаться «оптимизировать всё сразу», вы оптимизируете конкретные сценарии.

Наконец, важно помнить, что все эти стратегии — не взаимоисключающие. Для простого сервиса иногда достаточно одного `JOIN FETCH` и парочки `@EntityGraph`. Но как только приложение растёт, без осознанного разделения на тонкие DTO для списков и детализированные графы для карточек вы почти неизбежно получаете смесь из избыточных загрузок, N+1 и сложных для отладки зависимостей. Чем раньше вы проведёте границу «что где читаем и зачем», тем проще будет масштабировать и код, и базу.

# 2. Проекции и сложные запросы

Перед тем как уходить в детали, важно зафиксировать идею: классический JPA-стек (Spring Data + Hibernate + JDBC-драйвер) прекрасно подходит для CRUD и простых выборок по сущностям, но как только речь заходит о тяжёлых отчётах, агрегатах, сложных фильтрах и оптимизации трафика между приложением и базой — вы почти неизбежно приходите к проекциям и более «ручному» SQL. В этой подтеме мы разберём, как в Spring Data JPA делать тонкие проекции (интерфейсные и DTO), когда и как использовать нативный SQL, и как аккуратно реализовывать частичные обновления, чтобы не ломать кеши и контекст персистентности.

Для всех примеров дальше будем считать, что у нас стандартный стек: Spring Boot 3.x, Spring Data JPA, Hibernate и Postgres. Зависимости в Gradle выглядят так и будут использоваться во всех примерах в этой подтеме.

```groovy
// build.gradle (Groovy)
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

```kotlin
// build.gradle.kts (Kotlin)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    runtimeOnly("org.postgresql:postgresql")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## Проекции: интерфейсные/DTO-конструкторы в Spring Data, закрытые vs открытые (SpEL) и их стоимость

Проекция в контексте Spring Data — это способ сказать: «мне не нужна целиком сущность, дай только часть полей и оформь её в удобный для меня тип». В отличие от возврата полной сущности `Customer`, где JPA тащит всё, что описано в маппинге, проекция позволяет сузить форму результата до ровно тех столбцов, которые нужны UI или соседнему сервису. Это сразу даёт выигрыш по трём направлениям: меньше данных по сети, меньше работы на стороне JDBC-драйвера и меньше нагрузки на ORM (нет трекинга изменений для DTO).

На практике проекции особенно важны для списков. Если у вас страница «список клиентов» и «карточка клиента», то для списка достаточно 2–3 полей и пары агрегатов, а вот для карточки нужен полноценный граф. Попытка везде возвращать сущности почти всегда приводит либо к лишнему трафику и N+1, либо к сложным графам `join fetch`. Проекции позволяют честно разделить «view для списка» и «view для карточки» и оптимизировать каждый сценарий отдельно.

Spring Data поддерживает два основных вида проекций: интерфейсные и DTO-конструкторные. Интерфейсные проекции — это интерфейс с геттерами, имена которых совпадают с алиасами колонок в запросе или с полями сущности. Spring Data создаёт runtime-прокси, который маппит значения из результата SQL в методы интерфейса. Это дешево по разработке, достаточно гибко и хорошо работает даже для нативных запросов, если правильно проставлены алиасы.

Пример: возьмём сущность `Customer` с полями `id`, `firstName`, `lastName` и `status`. Для списков нам нужен только id и «полное имя». Интерфейсная проекция в Java выглядит так: геттеры определяют форму данных, а Spring сам подложит реализацию.

```java
package com.example.jpa.projection;

public interface CustomerNameView {

    Long getId();

    String getFirstName();

    String getLastName();
}
```

```kotlin
package com.example.jpa.projection

interface CustomerNameView {
    fun getId(): Long
    fun getFirstName(): String
    fun getLastName(): String
}
```

Репозиторий может возвращать такую проекцию вместо сущности. В простейшем случае, если колонки совпадают с именами свойств, можно обойтись без явного JPQL — Spring Data сам построит запрос. Но чаще всё же лучше писать `@Query` явно, особенно если требуется сортировка или фильтры.

```java
package com.example.jpa.projection;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface CustomerRepository extends JpaRepository<Customer, Long> {

    @Query("""
           select c.id as id,
                  c.firstName as firstName,
                  c.lastName as lastName
           from Customer c
           where c.status = :status
           """)
    List<CustomerNameView> findByStatus(String status);
}
```

```kotlin
package com.example.jpa.projection

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface CustomerRepository : JpaRepository<Customer, Long> {

    @Query(
        """
        select c.id as id,
               c.firstName as firstName,
               c.lastName as lastName
        from Customer c
        where c.status = :status
        """
    )
    fun findByStatus(status: String): List<CustomerNameView>
}
```

DTO-конструкторные проекции делают то же самое, но вместо интерфейса используют конкретный класс (record/data class) и конструктор. В JPQL это выглядит как `select new com.example.CustomerDto(...)`, а JPA принимает на себя задачу вызвать нужный конструктор и передать туда значения. Плюс такого подхода — типобезопасность: если вы переименовали поле или поменяли сигнатуру конструктора, код перестанет компилироваться, а не молча начнёт возвращать нули.

```java
package com.example.jpa.projection;

public record CustomerNameDto(Long id, String fullName) {
}
```

```java
package com.example.jpa.projection;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface CustomerDtoRepository extends JpaRepository<Customer, Long> {

    @Query("""
           select new com.example.jpa.projection.CustomerNameDto(
               c.id,
               concat(c.firstName, ' ', c.lastName)
           )
           from Customer c
           where c.status = :status
           """)
    List<CustomerNameDto> findDtosByStatus(String status);
}
```

```kotlin
package com.example.jpa.projection

data class CustomerNameDto(
    val id: Long,
    val fullName: String
)
```

```kotlin
package com.example.jpa.projection

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface CustomerDtoRepository : JpaRepository<Customer, Long> {

    @Query(
        """
        select new com.example.jpa.projection.CustomerNameDto(
            c.id,
            concat(c.firstName, ' ', c.lastName)
        )
        from Customer c
        where c.status = :status
        """
    )
    fun findDtosByStatus(status: String): List<CustomerNameDto>
}
```

Переходим к понятию «закрытые» и «открытые» проекции в Spring Data. Закрытые проекции — это когда все поля берутся напрямую из результата запроса. Это как раз наши интерфейсные и DTO-проекции выше: значения приходят напрямую из SQL, без дополнительной логики в SpEL. Открытые проекции позволяют добавлять вычисляемые свойства через SpEL-выражения в интерфейсе: Spring может подставить, например, `@Value("#{target.firstName + ' ' + target.lastName}")`. Это очень гибко, но дороже.

```java
package com.example.jpa.projection;

import org.springframework.beans.factory.annotation.Value;

public interface CustomerOpenView {

    Long getId();

    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}
```

```kotlin
package com.example.jpa.projection

import org.springframework.beans.factory.annotation.Value

interface CustomerOpenView {
    fun getId(): Long

    @get:Value("#{target.firstName + ' ' + target.lastName}")
    val fullName: String
}
```

Стоимость открытых проекций в том, что SpEL выражение вычисляется для каждой строки, и часто для него используется уже загруженная сущность-«target», а не результат «плоского» запроса. Если выражение обращается к ленивым ассоциациям, вы легко получите N+1: каждое чтение computed-поля триггерит дополнительный SELECT. Поэтому открытые проекции годятся для небольших выборок, но категорически не подходят для больших списков и тяжёлых отчётов.

Практическое правило такое: для списков используйте закрытые интерфейсные или DTO-проекции, в которые складывайте только реально нужные поля и агрегаты; для карточек, где результат один, допускается чуть больше свободы, но всё равно лучше отдавать DTO, а не сущность. Открытые проекции через SpEL — это скорее инструмент для быстрого прототипирования или для простых кейсов, но не фундамент производительности.

Наконец, важно понимать, что проекции — это часть контракта между слоем данных и внешним миром. Если вы возвращаете из контроллера DTO или интерфейсную проекцию, JSON будет ровно таким, как вы ожидаете: пробрасывание сущностей не утянет за собой лишние поля и ассоциации. Это хорошо влияет не только на производительность, но и на безопасность: вы не отдаёте наружу поля, о которых UI даже не знает.

---

## Нативные SQL: маппинг на DTO/интерфейсы, SqlResultSetMapping (где оправдано)

Иногда JPQL и Criteria-API банально не хватает. Как только появляются window-функции, CTE, специфичные для Postgres операторы по JSONB или сложные `ON CONFLICT`, вы неизбежно смотрите в сторону нативного SQL. Это нормально: Hibernate не заменяет вам SQL, а дополняет. Главное — осознанно решать, где нативный запрос оправдан, а где вам просто лень писать JPQL.

Spring Data умеет маппить результат нативного SQL в интерфейсные проекции. Секрет здесь простой: названия колонок в SELECT должны совпадать с геттерами интерфейса (либо через алиасы). Для простых отчётов этого достаточно: вы пишете `@Query(value = "...", nativeQuery = true)` и возвращаете список проекций.

```java
package com.example.jpa.nativeq;

public interface CustomerRow {

    Long getId();

    String getFirstName();

    String getLastName();

    String getStatus();
}
```

```java
package com.example.jpa.nativeq;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface NativeCustomerRepository extends JpaRepository<Customer, Long> {

    @Query(
        value = """
                select id,
                       first_name as firstName,
                       last_name  as lastName,
                       status
                from customers
                where status = :status
                order by id
                """,
        nativeQuery = true
    )
    List<CustomerRow> findNativeByStatus(String status);
}
```

```kotlin
package com.example.jpa.nativeq

interface CustomerRow {
    fun getId(): Long
    fun getFirstName(): String
    fun getLastName(): String
    fun getStatus(): String
}
```

```kotlin
package com.example.jpa.nativeq

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query

interface NativeCustomerRepository : JpaRepository<Customer, Long> {

    @Query(
        value = """
            select id,
                   first_name as firstName,
                   last_name  as lastName,
                   status
            from customers
            where status = :status
            order by id
            """,
        nativeQuery = true
    )
    fun findNativeByStatus(status: String): List<CustomerRow>
}
```

В этом примере Spring Data берёт ResultSet, смотрит на имена колонок (`id`, `firstName`, `lastName`, `status`) и маппит их на методы интерфейса. Такой подход отлично работает для плоских DTO и отчётов. Преимущество — вы используете все возможности SQL, а всё равно получаете удобный типизированный результат.

Когда же нужен `SqlResultSetMapping`? В двух случаях: когда вы хотите маппинг на обычный класс (конструктор), а не интерфейс, и когда вам нужно использовать NamedNativeQuery с более тонким контролем над маппингом. `@SqlResultSetMapping` позволяет описать, какие колонки в результате соответствуют каким параметрам конструктора или полям класса, и затем использовать это описание в `createNamedQuery`.

```java
package com.example.jpa.nativeq;

public record CustomerRevenueDto(Long customerId, String name, double totalRevenue) {
}
```

```java
package com.example.jpa.nativeq;

import jakarta.persistence.*;

@Entity
@Table(name = "customers")
@NamedNativeQuery(
        name = "Customer.revenueReport",
        query = """
                select c.id          as customer_id,
                       c.first_name  as name,
                       coalesce(sum(o.total_amount), 0) as total_revenue
                from customers c
                left join orders o on o.customer_id = c.id
                group by c.id, c.first_name
                """,
        resultSetMapping = "CustomerRevenueMapping"
)
@SqlResultSetMapping(
        name = "CustomerRevenueMapping",
        classes = @ConstructorResult(
                targetClass = CustomerRevenueDto.class,
                columns = {
                        @ColumnResult(name = "customer_id", type = Long.class),
                        @ColumnResult(name = "name", type = String.class),
                        @ColumnResult(name = "total_revenue", type = Double.class)
                }
        )
)
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String firstName;

    private String lastName;

    private String status;

    // getters/setters
}
```

```kotlin
package com.example.jpa.nativeq

data class CustomerRevenueDto(
    val customerId: Long,
    val name: String,
    val totalRevenue: Double
)
```

```kotlin
package com.example.jpa.nativeq

import jakarta.persistence.*

@Entity
@Table(name = "customers")
@NamedNativeQuery(
    name = "Customer.revenueReport",
    query = """
        select c.id          as customer_id,
               c.first_name  as name,
               coalesce(sum(o.total_amount), 0) as total_revenue
        from customers c
        left join orders o on o.customer_id = c.id
        group by c.id, c.first_name
        """,
    resultSetMapping = "CustomerRevenueMapping"
)
@SqlResultSetMapping(
    name = "CustomerRevenueMapping",
    classes = [
        ConstructorResult(
            targetClass = CustomerRevenueDto::class,
            columns = [
                ColumnResult(name = "customer_id", type = Long::class),
                ColumnResult(name = "name", type = String::class),
                ColumnResult(name = "total_revenue", type = Double::class)
            ]
        )
    ]
)
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var firstName: String = "",

    var lastName: String = "",

    var status: String = ""
)
```

Вызывать такой запрос удобнее всего из сервиса через `EntityManager`. Spring Data здесь не особо помогает, потому что ему важнее сущности и JPQL. Но сервисный слой вполне может использовать `createNamedQuery` и получить список DTO.

```java
package com.example.jpa.nativeq;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class RevenueReportService {

    @PersistenceContext
    private EntityManager em;

    @Transactional(readOnly = true)
    public List<CustomerRevenueDto> getRevenueReport() {
        return em.createNamedQuery("Customer.revenueReport", CustomerRevenueDto.class)
                 .getResultList();
    }
}
```

```kotlin
package com.example.jpa.nativeq

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class RevenueReportService(

    @PersistenceContext
    private val em: EntityManager
) {

    @Transactional(readOnly = true)
    fun getRevenueReport(): List<CustomerRevenueDto> =
        em.createNamedQuery("Customer.revenueReport", CustomerRevenueDto::class.java)
            .resultList
}
```

В чём преимущества и ограничения такого подхода? Плюсы очевидны: вы получаете всю мощь SQL, включая специфичные возможности Postgres (JSONB, window-функции, CTE), и контролируете каждый байт, который идёт по JDBC. Минусы — теряете переносимость (запрос завязан на конкретную СУБД), а маппинг становится более хрупким: любое изменение схемы может сломать `@SqlResultSetMapping`, и это ещё нужно поймать тестами.

По производительности нативные запросы часто выигрывают у JPQL, если вы умеете писать хороший SQL. Но если вы просто переписали простую выборку в native «ради спортивного интереса», то шансы велики, что профит будет нулевым, а код — сложнее и менее читаемым. Поэтому разумная стратегия — использовать `nativeQuery` и `SqlResultSetMapping` для действительно сложных отчётов и агрегатов, а всё, что возможно, держать на уровне JPQL/проекций.

Если вы замечаете, что весь слой данных превращается в зоопарк нативных запросов, это сигнал, что вам, возможно, нужен отдельный инструмент — например, jOOQ для отчётной части, оставив JPA для доменных сущностей. Но это уже тема отдельной подтемы в этом же разделе.

---

## Частичные обновления: @Modifying JPQL/SQL, возвращаемые счётчики, осторожно с кешами и L1-контекстом

Полноценное JPA-обновление выглядит так: вы загружаете сущность, меняете поле, транзакция коммитится, Hibernate на основе dirty checking генерирует нужный `UPDATE`. Это удобно и безопасно: контекст персистентности знает, что произошло, L1-кеш консистентен, и всё работает. Но иногда это дорого: например, когда вы хотите массово поменять статус тысяч записей, или когда вам нужно изменить одно поле без загрузки тяжёлого графа связей. Здесь на сцену выходят частичные обновления через `@Modifying`.

Spring Data JPA позволяет объявлять на репозитории методы, которые выполняют `UPDATE`/`DELETE` напрямую в базе, минуя загрузку сущностей. Для этого используется аннотация `@Modifying` поверх `@Query`. Такой метод возвращает количество затронутых строк и не возвращает сущности. С точки зрения JDBC это просто выполнение DML, а Hibernate лишь проксирует запрос к драйверу.

```java
package com.example.jpa.update;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;

public interface OrderRepository extends JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
           update Order o
           set o.status = :status
           where o.id = :id
           """)
    int updateStatusById(Long id, OrderStatus status);
}
```

```kotlin
package com.example.jpa.update

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query

interface OrderRepository : JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        """
        update Order o
        set o.status = :status
        where o.id = :id
        """
    )
    fun updateStatusById(id: Long, status: OrderStatus): Int
}
```

Ключевые моменты: `@Modifying` говорит Spring Data, что это не select, а update/delete; `flushAutomatically = true` заставляет сначала сбросить в базу все накопленные изменения в persistence context (чтобы не потерять их), а `clearAutomatically = true` — очистить контекст после выполнения запроса, чтобы там не оставались сущности со старым состоянием. Возвращаемое значение — количество строк, на которые повлиял update.

Важно помнить, что такие bulk-операции выполняются в рамках текущей транзакции, поэтому сервисный метод, который их вызывает, должен быть помечен `@Transactional`. В противном случае Spring Data откроет короткую транзакцию только на время выполнения update, но вы потеряете возможность комбинировать эту операцию с другими действиями в одной транзакции.

```java
package com.example.jpa.update;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void cancelOrder(Long id) {
        int updated = repository.updateStatusById(id, OrderStatus.CANCELED);
        if (updated == 0) {
            throw new IllegalArgumentException("Order not found: " + id);
        }
    }
}
```

```kotlin
package com.example.jpa.update

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderService(
    private val repository: OrderRepository
) {

    @Transactional
    fun cancelOrder(id: Long) {
        val updated = repository.updateStatusById(id, OrderStatus.CANCELED)
        if (updated == 0) {
            throw IllegalArgumentException("Order not found: $id")
        }
    }
}
```

Главный подводный камень частичных обновлений — контекст персистентности (L1-кеш). Bulk `UPDATE` в JPQL или native SQL **обходит** L1-контекст и изменяет данные напрямую в базе. Если в той же сессии у вас уже загружены сущности `Order` с этим id, их состояние в памяти останется старым. Любое обращение к этим сущностям после bulk-операции будет использовать устаревшие данные, пока вы явно не вызовете `refresh()` или не очистите контекст.

Параметр `clearAutomatically = true` как раз решает эту проблему: после выполнения update Spring Data попросит EntityManager сделать `clear()`, и все сущности будут отвязаны от контекста. Это безопасно с точки зрения консистентности, но может быть дорогим, если в транзакции у вас накоплено много сущностей. Поэтому иногда рациональнее явно ограничивать область применения bulk-операций и не совмещать их с «обычной» JPA-работой в одной транзакции.

Ещё один вариант — использовать native SQL для частичных обновлений. С точки зрения JPA разницы мало: это всё равно bulk DML, который обходит L1-кеш. Но вы получаете доступ к специфичным возможностям СУБД, например к `now()`/`current_timestamp`, `jsonb_set` и другим функциям Postgres.

```java
package com.example.jpa.update;

import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;

public interface OrderRepository extends JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        value = """
                update orders
                set status = :status,
                    updated_at = now()
                where id = :id
                """,
        nativeQuery = true
    )
    int updateStatusNative(Long id, String status);
}
```

```kotlin
package com.example.jpa.update

import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query

interface OrderRepository : JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        value = """
            update orders
            set status = :status,
                updated_at = now()
            where id = :id
            """,
        nativeQuery = true
    )
    fun updateStatusNative(id: Long, status: String): Int
}
```

Отдельный слой проблем связан с кешами второго уровня (L2) и кешем запросов. Bulk-операции их тоже обходят. Если у вас включен L2-кеш Hibernate, после `update` напрямую в БД кешированные сущности останутся с прежним состоянием, и при следующем чтении вы получите устаревшие данные. Поэтому либо нужно отключать L2 для сущностей, над которыми вы делаете bulk-операции, либо после таких операций явно инвалидировать соответствующие регионы кеша.

Частичные обновления хорошо подходят для сценариев «массовой смены статуса» или «ночной» обработки, но для транзакций, важных с точки зрения бизнес-логики, я бы сначала подумал: нельзя ли обойтись обычной JPA-операцией с загрузкой сущности. Частичный update дешевле по JDBC, но дороже по сложности: нужно помнить про кеши, контекст, L2, кеш запросов и тестировать сценарии конкуренции. В продакшне это вполне рабочий инструмент, но использовать его стоит аккуратно и целенаправленно, а не «на всякий случай».

# 3. Вставки/обновления батчами и генерация идентификаторов

В этой подтеме разберём связку JPA/Hibernate ↔ JDBC: как сильно на реальную производительность влияют стратегии генерации идентификаторов, как правильно «поджечь» JDBC-батчинг, и почему массовые `bulk update/delete` — это не «чуть-чуть быстрее обычного `save()`», а принципиально другой механизм, который обходит контекст персистентности и кеши. Всё это критично важно, когда вы выходите за рамки «иногда сохранить одну сущность» и начинаете действительно грузить базу данными.

Исходный стек для примеров тот же: Spring Boot 3.x, Spring Data JPA, Hibernate, Postgres. Конфигурации Gradle (Groovy/Kotlin) — стандартные для JPA; мы их дальше будем дополнять только при необходимости.

---

## Влияние стратегий @GeneratedValue на батчи (SEQUENCE vs IDENTITY; оптимизаторы pooled/pooled-lo)

Первое, о чём обычно не думают, когда начинают «ускорять» вставки — это способ генерации первичных ключей. На уровне JPA всё выглядит просто: `@GeneratedValue(strategy = …)`, и как будто никакой магии. На самом деле выбор между `IDENTITY` и `SEQUENCE` напрямую влияет на то, сможет ли Hibernate вообще использовать JDBC-батчинг для INSERT. Если вы на Postgres бездумно ставите `GenerationType.IDENTITY` «как в MySQL», вы почти гарантированно убиваете батчи и получаете по одному INSERT на каждую сущность.

Стратегия `IDENTITY` (в Postgres — `serial`/`identity column`) означает, что идентификатор генерируется самой БД при выполнении INSERT. Hibernate, чтобы узнать сгенерированный id, должен выполнить INSERT **сразу** и дождаться результата. Он не может отложить Insert и потом собрать несколько записей в один пакет, потому что ему нужно значение id уже сейчас для дальнейшего использования (в связях, в графе объектов и т.д.). В результате при `GenerationType.IDENTITY` Hibernate вынужден отправлять каждую вставку отдельным запросом, а значит JDBC-батчинг для INSERT фактически не работает.

Стратегия `SEQUENCE` использует отдельный объект `SEQUENCE` в БД для генерации идентификаторов. С точки зрения JDBC это два шага: сначала Hibernate делает `select nextval('seq_name')` (иногда пачкой), получает набор id, а уже потом выполняет INSERT с уже заполненным значением pk. Ключевое преимущество здесь в том, что получение id можно отделить от самого INSERT и сгруппировать вставки в батчи. Hibernate умеет вызывать `nextval` не по одному, а блоками, и затем использовать эти id для нескольких сущностей подряд.

Оптимизаторы `pooled` и `pooled-lo` как раз отвечают за то, как Hibernate взаимодействует с sequence. Без оптимизатора при каждой вставке он делает отдельный `nextval`, что при больших объёмах даёт заметную нагрузку на sequence. С `pooled` Hibernate берёт диапазон id (например, 50 штук) одним запросом, а потом раздаёт их сущностям из памяти. Как только диапазон кончается — берёт следующий. `pooled-lo` делает похожую вещь, но рассчитывает диапазоны иначе (hi/lo), уменьшая количество обращений к sequence ещё сильнее и лучше распределяя id между несколькими нодами.

В Postgres типовая рекомендация для высоконагруженных вставок через Hibernate — использовать `SEQUENCE` с оптимизатором `pooled-lo`. Начиная с Hibernate 5+, `SequenceStyleGenerator` уже умеет сам подбирать оптимизатор, но вы можете его явно настроить. Сущность в Java с явным sequence и pooled-lo может выглядеть так: мы объявляем `@SequenceGenerator`, указываем имя sequence и передаём параметры оптимизатора.

```java
package com.example.jpa.batch;

import jakarta.persistence.*;
import org.hibernate.annotations.GenericGenerator;
import org.hibernate.annotations.Parameter;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(generator = "order_seq")
    @GenericGenerator(
            name = "order_seq",
            strategy = "org.hibernate.id.enhanced.SequenceStyleGenerator",
            parameters = {
                    @Parameter(name = "sequence_name", value = "order_seq"),
                    @Parameter(name = "optimizer", value = "pooled-lo"),
                    @Parameter(name = "increment_size", value = "50")
            }
    )
    private Long id;

    private String description;

    // другие поля, геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.batch

import jakarta.persistence.*
import org.hibernate.annotations.GenericGenerator
import org.hibernate.annotations.Parameter

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue(generator = "order_seq")
    @GenericGenerator(
        name = "order_seq",
        strategy = "org.hibernate.id.enhanced.SequenceStyleGenerator",
        parameters = [
            Parameter(name = "sequence_name", value = "order_seq"),
            Parameter(name = "optimizer", value = "pooled-lo"),
            Parameter(name = "increment_size", value = "50")
        ]
    )
    var id: Long? = null,

    var description: String = ""
)
```

На стороне БД при этом должна существовать sequence `order_seq`. Для Postgres типичный SQL:

```sql
create sequence if not exists order_seq increment by 50;
```

Даже если вы не конфигурируете оптимизатор вручную, уже переход с `IDENTITY` на `SEQUENCE` даёт Hibernate возможность использовать JDBC-батчинг для INSERT. С точки зрения приложения это выглядит как обычный `saveAll`, но по проволоке уезжает не 100 отдельных INSERT’ов, а 5–10 батчей, в зависимости от настроек `hibernate.jdbc.batch_size`.

Отдельная ловушка связана с `GenerationType.AUTO`. На разных СУБД Hibernate маппит AUTO по-разному: на H2 это может быть sequence, на Postgres — тоже, а на другой базе — identity. На dev-стенде вы счастливо живёте с sequence и батчами, а на проде после смены диалекта внезапно получаете identity и «мистическое» проседание производительности вставки. Поэтому в серьёзном проекте лучше всегда явно выбирать стратегию генерации и не полагаться на AUTO «как повезёт».

И ещё одно наблюдение: если вы используете UUID как primary key и генерируете его в приложении (например, `@GeneratedValue` с `UUIDGenerator` или просто руками), вы снимаете зависимость от БД для генерации id, и Hibernate может батчить вставки без лишних обращений к sequence. Но за это вы платите более тяжёлым pk (по размеру и по index cost). В Postgres, например, `uuid` индексируется хуже, чем `bigint`. Поэтому здесь тоже нужно смотреть на workload и выбирать сознательно.

Последний штрих — обратная сторона оптимизаторов: sequence начнёт «скакать» крупными блоками id, и после рестартов приложения могут появляться большие «дыры» в значениях. С точки зрения бизнеса это почти всегда нормально, но иногда встречаются требования «id должны быть подряд без дырок» — такие требования в современных high-concurrency системах придётся признать нереалистичными.

---

## Тюнинг батчей: hibernate.jdbc.batch_size, order_inserts/updates, драйверные оптимизации (для PG — reWriteBatchedInserts)

Когда вы разобрались с генерацией id, следующий логичный шаг — включить и настроить сам JDBC-батчинг. В Hibernate ключевой параметр — `hibernate.jdbc.batch_size`. Он говорит: «если я вижу несколько однотипных SQL (INSERT/UPDATE) подряд, собирай их в пакет указанного размера и отправляй в JDBC как batch». На уровне драйвера это превращается в вызов `PreparedStatement.addBatch()` несколько раз, а затем `executeBatch()`. Это резко уменьшает количество round-trip’ов к БД и количество парсинга SQL.

Параметр задаётся в `application.yml` через Spring Boot. Дополнительно часто включают `order_inserts` и `order_updates`, чтобы Hibernate мог переупорядочить операции так, чтобы максимизировать размер батча (сгруппировать однотипные операции по таблицам). Для Postgres, чтобы батчи дали максимум, почти всегда стоит включать свойство драйвера `reWriteBatchedInserts`.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
  datasource:
    url: jdbc:postgresql://localhost:5432/demo
    username: demo
    password: demo
    hikari:
      data-source-properties:
        reWriteBatchedInserts: true
```

В этой конфигурации `batch_size=50` означает, что Hibernate будет стараться аккумулировать до 50 однотипных SQL перед тем, как вызвать `executeBatch()`. Значение 20–50 часто оказывается хорошим компромиссом: слишком маленькое не даёт выигрыша, слишком большое может привести к длинным транзакциям и большим пачкам, которые сложнее откатить и которые тяжелее ложатся на БД.

Свойства `order_inserts=true` и `order_updates=true` разрешают Hibernate переупорядочивать INSERT/UPDATE внутри одного flush так, чтобы операции по одной таблице и с одинаковым набором колонок были рядом. Это помогает в сценариях, когда вы сохраняете граф связей, и ORM в естественном порядке ходит по объектам в разной последовательности. Ordering позволяет «собрать» пачку вставок в таблицу `orders`, потом пачку `order_items` и т.д., что значительно увеличивает размер batch.

`reWriteBatchedInserts=true` — это уже настройка на уровне драйвера Postgres. Без неё драйвер просто отправляет на сервер несколько отдельных INSERT как один batched statement; сервер выполняет их по очереди. С этой настройкой драйвер переписывает батч в **один** SQL вида `insert into ... values (...), (...), (...);`. Для Postgres это часто даёт драматический выигрыш, особенно если у вас много индексов и триггеров. Но важно помнить, что с этим свойством чуть сложнее дебажить SQL, так как логируется уже переписанный statement.

На уровне кода всё выглядит просто. В сервисе вы либо используете `saveAll`, либо вручную вызываете `entityManager.persist` в цикле. Если включены батчи, Hibernate сам соберёт однотипные INSERT/UPDATE в пачки. В примере ниже `saveAll` будет использовать batching, при условии что стратегия id и настройки позволяют.

```java
package com.example.jpa.batch;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class OrderBatchService {

    private final OrderRepository repository;

    public OrderBatchService(OrderRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void saveOrders(List<Order> orders) {
        repository.saveAll(orders);
    }
}
```

```kotlin
package com.example.jpa.batch

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderBatchService(
    private val repository: OrderRepository
) {

    @Transactional
    fun saveOrders(orders: List<Order>) {
        repository.saveAll(orders)
    }
}
```

Для ещё более тонкого контроля можно работать через `EntityManager`: например, каждые N сущностей вызывать `flush()` и `clear()`, чтобы не накапливать слишком много объектов в контексте персистентности. Это особенно актуально, если вы грузите десятки тысяч строк за один запуск.

```java
package com.example.jpa.batch;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class OrderBatchInsertService {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void insertInBatches(List<Order> orders) {
        int batchSize = 50;
        int i = 0;
        for (Order order : orders) {
            em.persist(order);
            i++;
            if (i % batchSize == 0) {
                em.flush();
                em.clear();
            }
        }
        em.flush();
        em.clear();
    }
}
```

```kotlin
package com.example.jpa.batch

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderBatchInsertService(

    @PersistenceContext
    private val em: EntityManager
) {

    @Transactional
    fun insertInBatches(orders: List<Order>) {
        val batchSize = 50
        var i = 0
        for (order in orders) {
            em.persist(order)
            i++
            if (i % batchSize == 0) {
                em.flush()
                em.clear()
            }
        }
        em.flush()
        em.clear()
    }
}
```

Настройки батчинга касаются не только INSERT, но и UPDATE. Если вы в цикле меняете поле у множества уже загруженных сущностей и не вызываете `flush()` досрочно, Hibernate на момент commit увидит пачку однотипных UPDATE и тоже попробует их забатчить. Однако здесь стоит помнить про `hibernate.jdbc.batch_versioned_data`: по умолчанию Hibernate может избегать батчинга для версионируемых сущностей (`@Version`), чтобы корректно отслеживать оптимистические блокировки. Если вы хотите батчить и их, эту настройку можно включить, но тогда нужно внимательно следить за логикой обработки конфликтов версий.

Проверить, работает ли батчинг, проще всего через SQL-логи или статистику Hibernate. При включённом `hibernate.generate_statistics=true` и батч-сценарии вы увидите меньшее количество запросов, чем количество `persist`/`save`. Это хороший sanity-check: если вы включили batch_size, но статистика показывает 1000 INSERT’ов при 1000 сущностей — значит, что-то мешает батчам (чаще всего это IDENTITY или неоднородность SQL).

Стоит также иметь в виду, что слишком агрессивные батчи при долгих транзакциях могут дать обратный эффект: большие пачки блокируют много строк/страниц, увеличивают время удержания блокировок и усложняют откаты. Поэтому «поставить 1000 и забыть» — плохая идея. Более разумный подход — измерять: начать с 20–50, посмотреть на p95/p99 latency, на поведение БД под нагрузкой и уже от этого плясать.

---

## Массовые операции: bulk update/delete через JPQL — обход 1-го уровня, необходимость ручной синхронизации

Батчинг решает вопрос «как быстрее выполнить много похожих операций», но каждая операция всё равно описывается на уровне сущности (вставка/обновление конкретного объекта). Иногда же нужен принципиально другой подход: «обновить статус всех заказов старше N дней» или «удалить все лог-записи старше года». Грузить такие объёмы как сущности и крутить по ним циклы — и дорого, и бессмысленно. Для этого в JPA существует механизм bulk-операций: массовые `UPDATE`/`DELETE` в JPQL и native SQL.

Bulk-операция в JPQL — это запрос вида `update Entity e set e.field = :value where ...` или `delete from Entity e where ...`. Hibernate переводит его в один SQL-запрос на уровне БД. Он **не** загружает сущности в контекст персистентности, не выполняет lifecycle-callback’и (`@PreUpdate`, `@PreRemove` и т.д.) и не следит за `@Version`. Это мощный, но довольно «грубый» инструмент: вы напрямую управляете строками в таблице, обходя большинство механизмов ORM.

В Spring Data такие операции объявляются через `@Modifying` поверх `@Query`. В отличие от обычных select-методов, они возвращают количество изменённых строк, а не сущности. Важно также позаботиться о синхронизации контекста: bulk-операция не знает ничего про уже загруженные сущности, поэтому либо нужно очистить контекст, либо гарантировать, что в нём нет объектов, затронутых запросом.

```java
package com.example.jpa.bulk;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;

import java.time.LocalDateTime;

public interface OrderBulkRepository extends JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
           update Order o
           set o.status = com.example.jpa.bulk.OrderStatus.EXPIRED
           where o.status = com.example.jpa.bulk.OrderStatus.NEW
             and o.createdAt < :threshold
           """)
    int markExpired(LocalDateTime threshold);

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
           delete from Order o
           where o.status = com.example.jpa.bulk.OrderStatus.CANCELED
             and o.createdAt < :threshold
           """)
    int deleteOldCanceled(LocalDateTime threshold);
}
```

```kotlin
package com.example.jpa.bulk

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query
import java.time.LocalDateTime

interface OrderBulkRepository : JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        """
        update Order o
        set o.status = com.example.jpa.bulk.OrderStatus.EXPIRED
        where o.status = com.example.jpa.bulk.OrderStatus.NEW
          and o.createdAt < :threshold
        """
    )
    fun markExpired(threshold: LocalDateTime): Int

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        """
        delete from Order o
        where o.status = com.example.jpa.bulk.OrderStatus.CANCELED
          and o.createdAt < :threshold
        """
    )
    fun deleteOldCanceled(threshold: LocalDateTime): Int
}
```

Сервисный слой при этом остаётся довольно простым. Bulk-операции должны выполняться внутри транзакции, как и обычные изменения. Возвращаемые счётчики удобно использовать для логирования и контроля: можно, например, алертить, если число затронутых строк неожиданно сильно выросло или упало.

```java
package com.example.jpa.bulk;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
public class OrderMaintenanceService {

    private final OrderBulkRepository repository;

    public OrderMaintenanceService(OrderBulkRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void cleanup(LocalDateTime threshold) {
        int expired = repository.markExpired(threshold);
        int deleted = repository.deleteOldCanceled(threshold);
        System.out.printf("Expired: %d, deleted: %d%n", expired, deleted);
    }
}
```

```kotlin
package com.example.jpa.bulk

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Service
class OrderMaintenanceService(
    private val repository: OrderBulkRepository
) {

    @Transactional
    fun cleanup(threshold: LocalDateTime) {
        val expired = repository.markExpired(threshold)
        val deleted = repository.deleteOldCanceled(threshold)
        println("Expired: $expired, deleted: $deleted")
    }
}
```

Главная особенность bulk-операций — они **обходят** первый уровень кеша (persistence context). Если до вызова `markExpired()` в той же транзакции вы уже загрузили какие-то `Order`’ы, их поля `status` в памяти не изменятся магическим образом. Именно поэтому в примере на репозитории стоит `clearAutomatically = true`: после выполнения запроса Spring попросит EntityManager сделать `clear()`, чтобы в нём не осталось устаревших сущностей. Альтернатива — явно вызывать `entityManager.clear()` в сервисе, но аннотация удобнее и меньше шансов забыть.

С кешем второго уровня ситуация похожая: bulk-операции **не** обновляют L2-кеш и кеш запросов. Если вы активно используете кеш второго уровня для сущностей, к которым применяете bulk-update, ваши чтения могут долго пользоваться устаревшими данными. Решение — либо не включать L2 для этих сущностей, либо явно инвалидировать соответствующие регионы кеша после bulk-операции через API Hibernate. В противном случае кеш превращается в источник неконсистентности.

Bulk-операции также обходят оптимистическую блокировку (`@Version`). При обычном обновлении Hibernate сравнивает версию сущности в БД и версию в памяти; при расхождении кидается `OptimisticLockException`. Bulk `UPDATE` не делает таких проверок: он просто меняет строки в таблице, и concurrent update может быть спокойно перезаписан. Если вам нужно сохранить семантику оптимистического lock’а, bulk-операции не подходят — лучше обновлять по одной сущности, пусть и с использованием JDBC-батчинга, но через обычный `save`.

Ещё один аспект — блокировки и нагрузка. Большой bulk `UPDATE` по миллионам строк может надолго блокировать таблицу или её значительную часть, вызвать эскалацию блокировок или повесить конкурирующие транзакции. В таких сценариях часто лучше делить операцию на «порции», либо работать через технику «обслуживание по страницам»: выбираете пачку id по критерию, обновляете только их, коммитите, повторяете. Это чуть сложнее реализовать, но сильно уменьшает риск «заморозить» прод.

Нативный SQL для bulk-операций по сути даёт те же свойства, что и JPQL, но позволяет использовать специфичные фичи СУБД. Для Postgres, например, вы можете использовать `now()`, `interval`, функции по JSONB и т.д. В Spring Data достаточно поставить `nativeQuery = true`. Однако с точки зрения консистентности и кешей ситуация такая же, как у JPQL: контекст персистентности нужно синхронизировать вручную.

```java
package com.example.jpa.bulk;

import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;

public interface OrderBulkRepository extends JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        value = """
                update orders
                set status = 'EXPIRED'
                where status = 'NEW'
                  and created_at < now() - interval '30 days'
                """,
        nativeQuery = true
    )
    int expireOlderThan30Days();
}
```

```kotlin
package com.example.jpa.bulk

import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query

interface OrderBulkRepository : JpaRepository<Order, Long> {

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query(
        value = """
            update orders
            set status = 'EXPIRED'
            where status = 'NEW'
              and created_at < now() - interval '30 days'
            """,
        nativeQuery = true
    )
    fun expireOlderThan30Days(): Int
}
```

В итоге подход к массовым операциям обычно такой: если вам важны lifecycle-события, `@Version` и аккуратная обработка конкуренции — делайте обновления через обычные сущности и используйте JDBC-батчинг для ускорения. Если у вас чисто техническая операция (архивация, чистка логов, массовая смена статуса старых «мертвых» записей), и вы готовы пожертвовать life-cycle’ом и оптимистическим lock’ом — bulk `update/delete` через JPQL/native дадут лучший результат. Важно только не смешивать bulk-операции и работу с сущностями в одном и том же `EntityManager` без явной очистки контекста: это прямой путь к «призрачным» багам, когда в памяти одно, в БД другое.

# 4. Транзакции, изоляция и блокировки

## Границы и изоляция: поведение `readOnly=true` как хинт, влияние `FlushMode` (COMMIT/MANUAL)

Транзакция в Spring Data JPA — это не только `@Transactional` «чтобы всё откатилось при ошибке». Это ещё и граница для работы Hibernate с JDBC: в её пределах живёт `EntityManager`, копится контекст персистентности (L1-кэш), происходит `flush` изменений и действуют выбранные уровни изоляции. От того, как вы расставите границы транзакций, будут зависеть не только корректность данных, но и производительность: сколько раз Hibernate будет ходить в базу, насколько долго будут держаться блокировки и сколько памяти съест контекст.

По умолчанию в Spring Boot все публичные методы сервисов с `@Transactional` работают с тем уровнем изоляции, который задан в DataSource/БД (для Postgres — обычно `READ COMMITTED`). Сам `@Transactional` без параметров не меняет isolation, propagation или timeout, он лишь говорит: «здесь начинается и заканчивается транзакция». Важно понимать, что в рамках одной транзакции Hibernate хранит все загруженные сущности в persistence context, и повторные чтения той же сущности идут из кэша, а не в базу. Это иногда создаёт ощущение «repeatable read», даже если уровень изоляции ниже.

Атрибут `readOnly = true` в `@Transactional` — это **хинт**, а не железная гарантия. На уровне Spring он говорит, что транзакция не будет изменять данные, и часть инфраструктуры может это использовать (например, некоторые DataSource могут включать read-only режим для соединения). В связке с Hibernate Spring по умолчанию переводит Session в режим `FlushMode.MANUAL` или `FlushMode.COMMIT` в зависимости от версии, то есть Hibernate перестаёт автоматически сбрасывать изменения в базу при выполнении запросов и будет флашить только при явном `flush()` или при коммите.

Важно не переоценивать `readOnly=true`. Если внутри метода вы сами меняете сущности (`entity.setX(...)`), Hibernate по-прежнему будет трекать изменения в контексте. В большинстве конфигураций он не сделает `UPDATE` в БД (из-за флеш-мода), но L1-кэш всё равно будет расти. Поэтому «read-only метод» не должен модифицировать сущности даже «по случайности». Если внутри вы логически меняете состояние, делайте это в отдельной транзакции без readOnly, иначе получите странное поведение и возможные расхождения между тем, что в памяти, и тем, что в базе.

`FlushMode` определяет, когда Hibernate сбрасывает накопленные изменения в БД. `AUTO` (по умолчанию) означает, что flush произойдёт перед каждым запросом, который может зависеть от текущих изменений, и при commit. Это безопасно, но может приводить к лишним `UPDATE`/`INSERT` «в середине» метода. `COMMIT` означает, что Hibernate старается отложить `flush` до момента commit транзакции, за исключением особо хитрых кейсов. `MANUAL` полностью перекладывает ответственность за `flush` на разработчика: пока вы сами не скажете, Hibernate не полезет в БД.

Для read-heavy методов удобно использовать комбинацию `@Transactional(readOnly = true)` и `FlushMode.MANUAL`, чтобы Hibernate вообще не рассматривал изменения и не тратил время на dirty checking перед выполнением JPQL/Criteria. Это даёт заметный выигрыш, если у вас крупные страницы с большим количеством сущностей. Но нужно осознавать, что внутри такого метода любые изменения сущностей не будут отправлены в БД: это не «магически безопасное обновление», это просто отключённый flush.

В Spring Boot можно задать flush-mode глобально через свойства Hibernate, но гораздо полезнее управлять им на уровне конкретных операций. Например, для некоторых long-running read-only репортов вы можете явно переключать flushMode на MANUAL через `EntityManager`. Такой подход особенно полезен там, где в рамках одной транзакции вы делаете много чтений и не хотите, чтобы Hibernate каждый раз проверял, надо ли флашить.

```java
package com.example.tx;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.Session;
import org.hibernate.FlushMode;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ReportService {

    @PersistenceContext
    private EntityManager em;

    @Transactional(readOnly = true)
    public void generateBigReport() {
        Session session = em.unwrap(Session.class);
        session.setFlushMode(FlushMode.MANUAL);

        // длинная цепочка чтений, без модификаций
        // ...
    }
}
```

```kotlin
package com.example.tx

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.hibernate.FlushMode
import org.hibernate.Session
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ReportService(

    @PersistenceContext
    private val em: EntityManager
) {

    @Transactional(readOnly = true)
    fun generateBigReport() {
        val session = em.unwrap(Session::class.java)
        session.flushMode = FlushMode.MANUAL

        // длинная цепочка чтений, без модификаций
        // ...
    }
}
```

Отдельный аспект — уровень изоляции транзакций. Его можно задавать на `@Transactional(isolation = Isolation.REPEATABLE_READ)` и он пойдёт в драйвер/БД через DataSource. Для Postgres `REPEATABLE READ` означает моментальный снимок (snapshot) на момент начала транзакции, и все запросы внутри будут читать один и тот же снимок, независимо от параллельных коммитов. Но Hibernate при этом всё равно работает со своим L1-кэшем, и в большинстве случаев вы увидите одни и те же объекты даже при более слабом isolation — просто потому, что они уже находятся в persistence context.

Наконец, важно увязать границы транзакций с жизненным циклом веб-запроса. В типичном Spring Boot приложении на MVC/WebFlux транзакция живёт в сервисном слое, а не в контроллере. Это значит, что в контроллер лучше не протаскивать ленивые сущности и не полагаться на открытые сессии (`OpenSessionInView`), иначе вы рискуете получать запросы в БД уже после выхода из транзакции. Строгое правило «всё чтение и изменение — внутри сервисных методов с @Transactional» помогает держать поведение предсказуемым и хорошо контролируемым.

Чтобы всё это работало, нужны зависимости на Spring Data JPA и драйвер БД. Конфигурация Gradle стандартная: одного раза на проект достаточно, не обязательно дублировать под каждую подтему, но для полноты картины напомню типичный вид.

```groovy
// build.gradle (Groovy)
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    runtimeOnly 'org.postgresql:postgresql'
}
```

```kotlin
// build.gradle.kts (Kotlin)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    runtimeOnly("org.postgresql:postgresql")
}
```

Итого: границы транзакций — это не только «где откатывать». Это ещё и место, где вы решаете, как часто Hibernate будет флашить изменения, как он будет кэшировать сущности и какой уровень изоляции будет использоваться. `readOnly=true` помогает оптимизировать read-heavy сценарии, но не освобождает от дисциплины: не менять сущности там, где вы обещали только читать, и не складывать в одну транзакцию всё подряд.

---

## Оптимистическая блокировка `@Version`: обработка конфликтов, повтор операций

Оптимистическая блокировка — это способ сказать: «я верю, что конкуренция редкая, но если два потока всё-таки полезут править одну и ту же запись, мы это заметим и не затрём изменения друг друга». В JPA это реализуется через поле с аннотацией `@Version`: при каждом `UPDATE` Hibernate увеличивает его значение и добавляет в `WHERE` версии старое значение. Если строка уже кем-то изменена и версия в БД другая, `UPDATE` затронет 0 строк, и Hibernate бросит `OptimisticLockException`.

Оптимистическая блокировка хорошо ложится на модель веб-приложений: пользователь открывает форму, задерживается, что-то меняет, и отправляет. В это время другой пользователь мог успеть изменить те же данные. Без блокировки вы просто затрёте чужое изменение, а с `@Version` поймаете конфликт и сможете решить, что делать: показать ошибку, отобразить diff, предложить перезагрузить форму или попробовать повторно применить действие.

Типичный пример — сущность `Account` с балансом и номером версии. Версию удобно хранить в `int` или `long`, но можно использовать и `Instant/OffsetDateTime`. Hibernate сам будет увеличивать версию при каждом `UPDATE`, вам не нужно трогать поле вручную. Главное — не пытаться модифицировать версию своими руками: это ломает контракт ORM.

```java
package com.example.tx;

import jakarta.persistence.*;

@Entity
@Table(name = "accounts")
public class Account {

    @Id
    @GeneratedValue
    private Long id;

    private String owner;

    private long balance;

    @Version
    private long version;

    // getters/setters
}
```

```kotlin
package com.example.tx

import jakarta.persistence.*

@Entity
@Table(name = "accounts")
class Account(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var owner: String = "",

    var balance: Long = 0,

    @Version
    var version: Long = 0
)
```

Сервис, который меняет баланс, в коде выглядит тривиально: загрузили сущность, изменили поле, транзакция закоммитилась — Hibernate сделал `UPDATE`. Но под капотом он добавит в SQL условие по версии: `where id = ? and version = ?`. Если параллельно кто-то уже успел обновить запись, версия в БД изменится, и `UPDATE` вернёт 0 затронутых строк; Hibernate это увидит и бросит `OptimisticLockException`.

```java
package com.example.tx;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AccountService {

    private final AccountRepository repository;

    public AccountService(AccountRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void deposit(Long id, long amount) {
        Account account = repository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Account not found"));
        account.setBalance(account.getBalance() + amount);
        // при commit Hibernate попытается сделать UPDATE с проверкой версии
    }
}
```

```kotlin
package com.example.tx

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class AccountService(
    private val repository: AccountRepository
) {

    @Transactional
    fun deposit(id: Long, amount: Long) {
        val account = repository.findById(id)
            .orElseThrow { IllegalArgumentException("Account not found") }

        account.balance += amount
        // при commit Hibernate сделает UPDATE с проверкой версии
    }
}
```

Обработка `OptimisticLockException` — это всегда бизнес-решение. Можно просто пробрасывать её наверх и возвращать 409/409-like ошибку в API с сообщением «запись изменилась, попробуйте ещё раз». А можно реализовать ретраи: ещё раз прочитать актуальное состояние, применить действие и попытаться закоммитить. Для идемпотентных операций (например, установить статус в определённое значение) ретраи зачастую безопасны; для операций типа «увеличить баланс» нужно аккуратно думать, хотите ли вы применять их повторно.

Простейший пример ретрая — обёртка с циклом. Снаружи вы вызываете метод, который внутри транзакции пытается сделать действие, а снаружи ловите `OptimisticLockException` и повторяете попытку несколько раз. Важно ограничить количество ретраев и логировать ситуации, когда конфликт долго не разрешается.

```java
package com.example.tx;

import jakarta.persistence.OptimisticLockException;
import org.springframework.stereotype.Service;

@Service
public class AccountFacade {

    private final AccountService service;

    public AccountFacade(AccountService service) {
        this.service = service;
    }

    public void safeDeposit(Long id, long amount) {
        int attempts = 0;
        while (true) {
            try {
                service.deposit(id, amount);
                return;
            } catch (OptimisticLockException ex) {
                attempts++;
                if (attempts >= 3) {
                    throw ex;
                }
            }
        }
    }
}
```

```kotlin
package com.example.tx

import jakarta.persistence.OptimisticLockException
import org.springframework.stereotype.Service

@Service
class AccountFacade(
    private val service: AccountService
) {

    fun safeDeposit(id: Long, amount: Long) {
        var attempts = 0
        while (true) {
            try {
                service.deposit(id, amount)
                return
            } catch (ex: OptimisticLockException) {
                attempts++
                if (attempts >= 3) {
                    throw ex
                }
            }
        }
    }
}
```

Есть несколько типичных ошибок при использовании `@Version`. Первая — отключать версию в bulk-обновлениях, думая, что «ничего страшного»: при этом вы разрешаете тихие перезаписи конкурирующих изменений и теряете ключевое преимущество оптимистического lock’а. Вторая — использовать версию, но игнорировать `OptimisticLockException`: ловить её и ничего не делать, кроме логирования. В таком случае приложение продолжает работать, но теряет данные при конфликте, а вы узнаёте об этом только по логам.

Ещё один важный момент — выбор типа поля версии. `int`/`long` работают быстро, но в распределённых системах с большим количеством обновлений версия будет быстро расти. Обычно это не проблема, но если вы задумались об этом — можно использовать `Instant` и хранить timestamp последнего изменения. Hibernate при этом сравнивает значения на равенство, а не «больше/меньше», так что конфликт будет пойман, если в БД записано другое время.

В связке с REST оптимистическая блокировка хорошо сочетается с ETag/If-Match: вы можете выставлять версию в заголовок ETag и требовать, чтобы клиент при обновлении присылал If-Match с этой версией. Тогда бэкенд сможет сопоставить версию в БД с версией у клиента и при расхождении вернуть 412 Precondition Failed. Это уже уровень API-дизайна, но важно понимать, что JPA-версия — хороший фундамент для таких протоколов.

---

## Пессимистические блокировки: `LockModeType.PESSIMISTIC_READ/WRITE`, таймауты/эскалации, риск дедлоков

Пессимистическая блокировка идёт от обратного: вместо «верим, что конфликта не будет, но проверим», мы говорим «мы сразу возьмём блокировку на строку и не дадим другим её менять, пока не закончим». Это удобно там, где конфликт — норма (например, в горячей сущности с высокочастотными обновлениями) или где последствия конфликта слишком серьёзны, чтобы полагаться на ретраи. Но за это вы платите риском дедлоков и снижением параллелизма.

В JPA пессимистические блокировки задаются через `LockModeType` при чтении: `PESSIMISTIC_READ` и `PESSIMISTIC_WRITE`. В Postgres они маппятся на `SELECT ... FOR SHARE` и `SELECT ... FOR UPDATE` соответственно. Первый режим блокирует запись от изменения, но позволяет другим читать; второй блокирует и чтение с таким же уровнем блокировки другими транзакциями, фактически гарантируя эксклюзивный доступ.

В Spring Data JPA пессимистический lock удобно навесить через `@Lock` на метод репозитория. Например, если вам нужен «select for update» по id, вы объявляете метод с `@Lock(LockModeType.PESSIMISTIC_WRITE)` и используете его в сервисе. В результате Hibernate добавит нужный `FOR UPDATE` и повесит блокировку на выбранную строку до конца транзакции.

```java
package com.example.tx;

import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;

import java.util.Optional;

public interface AccountLockingRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Account> findWithLockById(Long id);
}
```

```kotlin
package com.example.tx

import jakarta.persistence.LockModeType
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Lock
import java.util.Optional

interface AccountLockingRepository : JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    fun findWithLockById(id: Long): Optional<Account>
}
```

Сервисный метод с такой блокировкой будет выглядеть как обычное обновление, но под капотом транзакция будет держать row lock на строку `accounts.id = ?` до самого commit. Это гарантирует, что параллельные транзакции, пытающиеся взять такой же `PESSIMISTIC_WRITE`, будут ждать или упадут по таймауту.

```java
package com.example.tx;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LockedAccountService {

    private final AccountLockingRepository repository;

    public LockedAccountService(AccountLockingRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, long amount) {
        Account from = repository.findWithLockById(fromId)
                .orElseThrow(() -> new IllegalArgumentException("From not found"));
        Account to = repository.findWithLockById(toId)
                .orElseThrow(() -> new IllegalArgumentException("To not found"));

        from.setBalance(from.getBalance() - amount);
        to.setBalance(to.getBalance() + amount);
    }
}
```

```kotlin
package com.example.tx

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class LockedAccountService(
    private val repository: AccountLockingRepository
) {

    @Transactional
    fun transfer(fromId: Long, toId: Long, amount: Long) {
        val from = repository.findWithLockById(fromId)
            .orElseThrow { IllegalArgumentException("From not found") }

        val to = repository.findWithLockById(toId)
            .orElseThrow { IllegalArgumentException("To not found") }

        from.balance -= amount
        to.balance += amount
    }
}
```

Таймауты для пессимистических блокировок задаются через хинты JPA: `javax.persistence.lock.timeout`. В Hibernate это можно прокинуть через `@QueryHints` или через `EntityManager` при вызове `find`/`lock`. Если таймаут истёк, Hibernate бросит `LockTimeoutException`. Это важный элемент дизайна: бесконечное ожидание блокировки обычно хуже, чем контролируемый отказ через таймаут и fallback.

```java
package com.example.tx;

import jakarta.persistence.EntityManager;
import jakarta.persistence.LockModeType;
import jakarta.persistence.PersistenceContext;
import jakarta.persistence.QueryHint;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

@Repository
public class AccountLockDao {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public Account findWithTimeout(Long id) {
        return em.find(
                Account.class,
                id,
                LockModeType.PESSIMISTIC_WRITE,
                java.util.Map.of("jakarta.persistence.lock.timeout", 5000)
        );
    }
}
```

```kotlin
package com.example.tx

import jakarta.persistence.EntityManager
import jakarta.persistence.LockModeType
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional

@Repository
class AccountLockDao(

    @PersistenceContext
    private val em: EntityManager
) {

    @Transactional
    fun findWithTimeout(id: Long): Account =
        em.find(
            Account::class.java,
            id,
            LockModeType.PESSIMISTIC_WRITE,
            mapOf("jakarta.persistence.lock.timeout" to 5000)
        )
}
```

Главная опасность пессимистических блокировок — дедлоки. Если два потока берут блокировки в разном порядке (первый сначала `from`, потом `to`, второй наоборот), вы легко получите взаимную блокировку на уровне БД. Чтобы этого избежать, нужно жёстко зафиксировать порядок захвата ресурсов (например, всегда блокировать аккаунты по возрастанию id), минимизировать длительность транзакций и не держать блокировки дольше, чем необходимо.

Пессимистическая блокировка также чувствительна к уровню изоляции. На `READ COMMITTED` в Postgres `FOR UPDATE` блокирует только конкретные строки, и остальные, не попавшие в выборку, обрабатываются без блокировок. На более высоких уровнях (REPEATABLE READ, SERIALIZABLE) snapshot-семантика накладывается сверху, и поведение может быть сложнее. Важно помнить: JPA не скрывает от вас особенностей СУБД — она лишь маппит `LockModeType` на соответствующие SQL-конструкции.

Уместность пессимистических блокировок зависит от сценария. Для финансовых переводов с сильной консистентностью они могут быть оправданы, особенно если вы удерживаете блокировку только на одном узком объекте (баланс аккаунта). Для большинства обычных CRUD-операций оптимистический lock с ретраями даёт лучший баланс между параллелизмом и безопасностью. В любом случае нельзя массово «включить PESSIMISTIC_WRITE на всё» — это верный путь к деградации производительности и дедлокам, которые тяжело воспроизводить.

Наконец, не забывайте, что пессимистический lock в JPA работает только в пределах транзакции. Если вы забыли `@Transactional`, JPA сделает короткую транзакцию на SELECT, возьмёт и тут же отпустит блокировку. В коде всё отработает без ошибок, но никакой защиты от конкуренции вы не получите. Это один из тех кейсов, где наличие unit-теста мало помогает: нужно интеграционное тестирование с несколькими потоками, чтобы убедиться, что блокировки реально работают как задумано.

---

## Консистентность чтения: repeatable-read vs snapshot семантика СУБД, где важны «снимки»

Изоляция транзакций — это ответ на вопрос «что происходит, если параллельно кто-то пишет в те же таблицы, из которых я читаю». На уровне СУБД есть классические уровни: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE. В реальных БД, особенно в MVCC-системах вроде Postgres, они реализованы через snapshot-семантику: каждая транзакция видит некоторую «версию базы» и работает с ней до конца. JPA поверх этого добавляет свой уровень: L1-кэш, который делает вид, что всё ещё более repeatable, чем на самом деле.

В Spring `@Transactional` позволяет указать уровень изоляции через атрибут `isolation`. Для Postgres чаще всего используется `READ COMMITTED` как дефолт, а для особо критичных сценариев — `REPEATABLE READ` или `SERIALIZABLE`. Надо понимать, что смена isolation — это не «переключение флага в Hibernate», а изменение настроек транзакции в БД: драйвер отправляет соответствующую команду, и дальше всё зависит от движка.

```java
package com.example.tx;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class IsolationService {

    private final AccountRepository repository;

    public IsolationService(AccountRepository repository) {
        this.repository = repository;
    }

    @Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
    public Account loadTwice(Long id) {
        Account first = repository.findById(id)
                .orElseThrow();
        Account second = repository.findById(id)
                .orElseThrow();
        return second;
    }
}
```

```kotlin
package com.example.tx

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Isolation
import org.springframework.transaction.annotation.Transactional

@Service
class IsolationService(
    private val repository: AccountRepository
) {

    @Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
    fun loadTwice(id: Long): Account {
        val first = repository.findById(id).orElseThrow()
        val second = repository.findById(id).orElseThrow()
        return second
    }
}
```

Даже если вы оставите изоляцию `READ_COMMITTED`, повторный вызов `findById` в рамках одной транзакции вернёт тот же объект из L1-кэша Hibernate, а не свежие данные из БД. Это важно: JPA добавляет поверх изоляции БД свой слой кэширования. В результате вы практически всегда получаете «repeatable read» на уровне сущностей, пока не сделаете `clear()`/`refresh()`. Если вам по бизнесу важно перечитать состояние строки с учётом параллельных изменений, нужно либо явно вызывать `refresh`, либо разбивать операцию на несколько транзакций.

```java
package com.example.tx;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class RefreshService {

    @PersistenceContext
    private EntityManager em;

    private final AccountRepository repository;

    public RefreshService(AccountRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public long readAndRefresh(Long id) {
        Account account = repository.findById(id)
                .orElseThrow();
        long firstBalance = account.getBalance();

        // ... кто-то параллельно мог изменить баланс ...

        em.refresh(account);
        long secondBalance = account.getBalance();

        return secondBalance - firstBalance;
    }
}
```

```kotlin
package com.example.tx

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class RefreshService(

    @PersistenceContext
    private val em: EntityManager,
    private val repository: AccountRepository
) {

    @Transactional
    fun readAndRefresh(id: Long): Long {
        val account = repository.findById(id)
            .orElseThrow()
        val firstBalance = account.balance

        // ... параллельно баланс могли изменить ...

        em.refresh(account)
        val secondBalance = account.balance
        return secondBalance - firstBalance
    }
}
```

Postgres реализует `REPEATABLE READ` как snapshot isolation: в начале транзакции фиксируется снимок видимых коммитов, и все запросы читают именно его, не видя более поздних коммитов. Это защищает от non-repeatable read и большинства аномалий, но phantoms всё ещё возможны в специфических сценариях. На практике для большей части бизнес-логики `READ COMMITTED` достаточно, а `REPEATABLE READ` используют для критичных отчётных операций, где важно, чтобы все выборки были согласованы между собой.

С точки зрения JPA часто более важна не изоляция чтения, а то, как вы формулируете запросы. Если вы на уровне приложения хотите «снимок» состояния на момент начала операции, проще сделать один `SELECT` с нужными join’ами и агрегатами, чем играть с уровнями изоляции. Уровень `SERIALIZABLE` может казаться привлекательным, но в реальных нагрузках он даёт много конфликтов транзакций и откатов, особенно в Postgres, где он реализован через проверку зависимостей, а не через глобальные блокировки.

Ещё одна важная точка — разделение чтения и записи по транзакциям. Если вы в одном методе и читаете, и пишете, и хотите, чтобы чтения видели только коммиты до начала операции, имеет смысл делать «читающую часть» в `readOnly` транзакции с нужным isolation, а «пишущую» — в отдельной. Spring позволяет это через разные сервисные методы с разными аннотациями и propagation. Это сложнее, чем «обернуть всё в одну транзакцию», но даёт более предсказуемое поведение и изоляцию.

Важно также понимать, что для некоторых отчётных задач snapshot нужна не на уровне одной транзакции, а на уровне «логического времени» системы. В таких случаях проще делать материализованные представления, отчётные таблицы или использовать `AS OF`/`txid_snapshot`-подходы на стороне СУБД, чем пытаться выставить правильный isolation для огромной транзакции. JPA здесь всё равно будет только тонкой обёрткой над SQL.

И последнее: изоляция — это всегда компромисс между целостностью и производительностью. Чем выше isolation, тем больше риск блокировок, конфликтов и откатов. JPA и Spring не снимают с вас ответственности выбрать правильный уровень и правильно спроектировать операции. Они лишь дают инструменты: `@Transactional` с `isolation`, L1-кэш с `refresh`, возможность явно управлять границами транзакций и блокировками. Остальное — архитектурное решение на уровне системы.

# 5. Управление Persistence Context

`Persistence Context` (контекст персистентности) — это сердце работы Hibernate/JPA. Это и 1-й уровень кэша (L1), и трекер изменений, и идентификационная карта всех загруженных/созданных сущностей в рамках текущей сессии/транзакции. Пока сущность «управляемая» (managed), Hibernate следит за её полями, сравнивает старое и новое состояние и при `flush()` генерирует нужные SQL. Как только контекст разрастается, вы начинаете платить памятью, временем на dirty checking и риском тащить в него «полмира», а потом героически бороться с утечками. Эта подтема как раз про то, как не дать контексту сойти с ума: когда использовать `clear()`/`detach()`, как оптимизировать read-only сценарии и как взглянуть на L1-кэш, одновременно как на защиту от лишних SELECT и как на источник «устаревших» данных.

---

## Когда уместно `clear()/detach()`, «длинные» сессии и утечки памяти

Первое, что нужно зафиксировать: в типичном Spring Boot приложении одна транзакция = один `EntityManager` = один persistence context. Вы делаете `@Transactional` на сервисе, внутри него загружаете и модифицируете сущности, при коммите Hibernate делает `flush()` и отправляет SQL, а потом контекст уничтожается. Такой «короткоживущий» подход («session-per-request») почти всегда безопасен: контекст не успевает раздуться до гигантских размеров, и риск утечек памяти минимален.

Проблемы начинаются, когда вы отходите от модели «короткая транзакция на обычный HTTP-запрос» к длинным сценариям: импорт большого файла, массовая миграция, пересчёт отчётов, ночная обработка. Там внутри одной транзакции вы можете пройтись по десяткам/сотням тысяч строк, для каждой создать/загрузить сущность, потрогать поля — и всё это окажется в одном persistence context. Если вы ничего не делаете с этим контекстом, он растёт как снежный ком, увеличивая затраты на dirty checking, потребление heap и время `flush()`.

Например, вы читаете CSV на 200 000 строк и для каждой создаёте сущность `Order`. Без специальных мер Hibernate будет держать в памяти все эти 200 000 объектов до конца транзакции. Каждый новый `persist()` будет добавлять ещё одну сущность в L1-кэш, каждая ассоциация — ещё несколько. В какой-то момент вы начнёте видеть рост heap, GC-паузы, а то и `OutOfMemoryError`. Это и есть классический пример «длинной сессии», где контекст персистентности живёт слишком долго и хранит слишком много.

Чтобы этого избежать, для batch-обработки используют паттерн «батчи внутри транзакции»: вы прогоняете данные порциями, для каждой порции делаете `flush()` и `clear()`. `flush()` отправляет накопленные изменения в БД, а `clear()` выбрасывает все сущности из контекста, делая его пустым. После этого следующая порция снова начинает с чистого листа, и контекст никогда не вырастает больше, чем размер текущей порции. Цена — вы теряете возможность работать с уже загруженными сущностями после `clear()` — они становятся detached.

Иногда вам не нужно очищать весь контекст, а нужно просто «отвязать» конкретную сущность. Для этого есть `detach(entity)`: вы говорите Hibernate, что этот конкретный объект больше не managed. Изменения в нём не будут учитываться при `flush`, и он перестанет участвовать в dirty checking. Это удобно, когда вы, например, прогружаете большой список, делаете сериализацию в JSON и больше к этим объектам внутри транзакции не возвращаетесь. В отличие от `clear()`, `detach()` не трогает остальные сущности в контексте.

Важно понимать, что после `clear()`/`detach()` все ссылки в вашем коде на эти сущности продолжают указывать на объекты в памяти, но эти объекты больше никак не связаны с контекстом персистентности. Попытка работать с ленивой коллекцией (lazy association) на таком объекте приведёт к `LazyInitializationException`, потому что для подгрузки нужна сессия/контекст, а объект уже detached. Поэтому очищать контекст нужно осознанно: делать все необходимые операции с сущностями до `clear()`, а не после.

Типичный пример batch-паттерна в Java выглядит так: мы читаем коллекцию DTO, маппим в сущности, и каждые N записей делаем `flush()` и `clear()`. Важно, что транзакция при этом одна, так что либо коммитятся все батчи, либо всё откатывается. Но persistence context внутри транзакции переиспользуется, и мы намеренно его чистим.

```java
package com.example.jpa.context;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class OrderImportService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public void importOrders(List<OrderDto> dtos) {
        int batchSize = 50;
        int count = 0;

        for (OrderDto dto : dtos) {
            Order order = new Order();
            order.setExternalId(dto.externalId());
            order.setAmount(dto.amount());
            order.setStatus(OrderStatus.NEW);

            entityManager.persist(order);

            count++;
            if (count % batchSize == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }

        entityManager.flush();
        entityManager.clear();
    }
}
```

```kotlin
package com.example.jpa.context

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderImportService(

    @PersistenceContext
    private val entityManager: EntityManager
) {

    @Transactional
    fun importOrders(dtos: List<OrderDto>) {
        val batchSize = 50
        var count = 0

        for (dto in dtos) {
            val order = Order().apply {
                externalId = dto.externalId
                amount = dto.amount
                status = OrderStatus.NEW
            }

            entityManager.persist(order)
            count++

            if (count % batchSize == 0) {
                entityManager.flush()
                entityManager.clear()
            }
        }

        entityManager.flush()
        entityManager.clear()
    }
}
```

Такой код позволяет держать размер persistence context примерно в рамках 50 сущностей, даже если всего мы импортируем десятки тысяч. Это сильно снижает нагрузку на память и ускоряет `flush()`, потому что Hibernate не приходится держать и анализировать гигантский граф объектов. При этом транзакция всё ещё одна, и вы сохраняете атомарность операции: либо импорт целиком, либо ничего.

При работе с отношениями нужно учитывать, что `clear()` выбрасывает из контекста **всё**, включая связанные сущности. Если вы, например, создаёте `Order` и связанные `OrderItem`, а потом сразу делаете `clear()`, то в оперативной памяти у вас останутся объекты с заполненными коллекциями, но с точки зрения Hibernate они detached. Любое дальнейшее изменение этих объектов не попадёт в БД. Поэтому обычно batched-паттерн строят так, чтобы после `flush()/clear()` вы больше не трогали эти сущности до конца транзакции (а лучше вообще не держали на них ссылок).

Ещё один сценарий, где `detach()` бывает полезен — когда вы загружаете сущность только для чтения, но не хотите, чтобы случайные изменения в коде привели к SQL-обновлению. Например, вы читаете сущность, передаёте её в слой, где могут быть «грязные» мапперы/маппинг на DTO, и хотите гарантировать, что никакие изменения не уйдут в БД. Можно сразу после `find()` вызвать `entityManager.detach(entity)` и тем самым превратить сущность в обычный POJO, не связанный с контекстом. Это не заменяет нормальный дизайн слоёв, но иногда помогает в легаси.

Стоит упомянуть «extended persistence context» — концепцию JPA, где контекст живёт не один HTTP-запрос, а целую user-session/«conversation». В Spring этот режим почти не используется (чаще — в Jakarta EE с stateful EJB), потому что он сильно усложняет жизнь: ещё проще получить «длинную сессию», утечки, устаревшие данные и всё то, с чем мы боремся. В Spring Boot стандарт — «transaction-scoped persistence context», и это то, на что нужно ориентироваться по умолчанию.

Итог по этой части: `clear()` и `detach()` — это не «магическая оптимизация», а хирургический инструмент. `clear()` — для batch-сценариев и борьбы с раздутыми контекстами, `detach()` — для точечного «отключения» сущностей от Hibernate. В обычных CRUD-сервисах в 99 % случаев они не нужны: достаточно коротких транзакций и здравого смысла. Но как только вы лезете в массовую обработку, эти методы становятся критически важными, чтобы не утонуть в памяти и грязных сущностях.

---

## Read-only оптимизации: `Query.setReadOnly(true)`, «immutable»-сущности, `StatelessSession` (узкие случаи)

Большая часть продакшн-нагрузки — это чтение: списки, карточки, отчёты, поисковые страницы. Каждое чтение через JPA по умолчанию превращается в managed-сущность, попадает в persistence context и участвует в dirty checking. Для обычных запросов это нормально, но если у вас большие отчёты, сложные графы и вы точно знаете, что менять ничего не будете, есть смысл помочь Hibernate и сказать: «это read-only». Это уменьшит накладные расходы на трекинг изменений и освободит часть памяти.

Первый простой инструмент — `@Transactional(readOnly = true)`, о котором мы уже говорили в предыдущей подтеме. Но на уровне конкретного запроса можно пойти дальше и сказать Hibernate, что результат этого JPQL/SQL вообще не надо отслеживать, даже если внутри кода кто-то попытается изменить поля. Для этого используют Hibernate-специфичный флаг readOnly на Query: либо через `org.hibernate.query.Query#setReadOnly(true)`, либо через `setHint("org.hibernate.readOnly", true)`.

В Spring Data JPA мы можем получить доступ к низкоуровневому API через `EntityManager#unwrap(Session.class)` и затем — к Hibernate Query. Пример: у нас есть тяжёлый отчёт `ReportRow`, который мы хотим читать большим листом без dirty checking. Мы пишем сервис, который формирует JPQL, разворачивает запрос в Hibernate Query и помечает его `setReadOnly(true)`. Тогда Hibernate поставит сущности в специальный режим: они будут загружены, но не будут включены в dirty checking и flush.

```java
package com.example.jpa.readonly;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.Session;
import org.hibernate.query.Query;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class ReportReadOnlyService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<ReportRow> loadReport() {
        Session session = entityManager.unwrap(Session.class);

        Query<ReportRow> query = session.createQuery(
                """
                select new com.example.jpa.readonly.ReportRow(
                    o.id,
                    o.customerName,
                    o.amount
                )
                from Order o
                where o.status = :status
                """,
                ReportRow.class
        );

        query.setParameter("status", OrderStatus.FINISHED);
        query.setReadOnly(true);

        return query.list();
    }
}
```

```kotlin
package com.example.jpa.readonly

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.hibernate.Session
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ReportReadOnlyService(

    @PersistenceContext
    private val entityManager: EntityManager
) {

    @Transactional(readOnly = true)
    fun loadReport(): List<ReportRow> {
        val session = entityManager.unwrap(Session::class.java)

        val query = session.createQuery(
            """
            select new com.example.jpa.readonly.ReportRow(
                o.id,
                o.customerName,
                o.amount
            )
            from Order o
            where o.status = :status
            """.trimIndent(),
            ReportRow::class.java
        )

        query.setParameter("status", OrderStatus.FINISHED)
        query.isReadOnly = true

        return query.list()
    }
}
```

Обратите внимание, что в примере мы сразу выбираем DTO `ReportRow`, а не сущность. В таком режиме Hibernate вообще не трекает объекты как сущности: это обычные Java-объекты, на которые никакой flush не влияет. Это фактически идеальный read-only сценарий: вы минимизируете накладные расходы ORM, не рискуете случайно что-то записать в БД и явно отделяете «модель чтения» от «модели записи».

Следующий уровень — «immutable»-сущности. Если у вас есть таблицы-справочники, которые меняются крайне редко или только админским скриптом (например, типы операций, статусы, конфигурационные записи), вы можете пометить соответствующие сущности как неизменяемые через `@Immutable`. Это Hibernate-аннотация, которая говорит: не пытайся генерировать `UPDATE`/`DELETE` для этой сущности, считай её read-only. Hibernate при этом не будет включать такие сущности в dirty checking, что снижает накладные расходы.

```java
package com.example.jpa.readonly;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import org.hibernate.annotations.Immutable;

@Entity
@Table(name = "operation_type")
@Immutable
public class OperationType {

    @Id
    private Integer id;

    private String code;

    private String description;

    // getters (сеттеров может и не быть)
}
```

```kotlin
package com.example.jpa.readonly

import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.hibernate.annotations.Immutable

@Entity
@Table(name = "operation_type")
@Immutable
class OperationType(

    @Id
    var id: Int? = null,

    var code: String? = null,

    var description: String? = null
)
```

Использование `@Immutable` особенно полезно, когда такие сущности часто попадают в графы: вы подгружаете `OperationType` для каждой операции, но сами типы не меняете. Без `@Immutable` Hibernate добавляет их в dirty checking и учитывает при flush, хотя реальных изменений там нет. С `@Immutable` он может пропустить этот шаг и сэкономить время.

Самый радикальный инструмент для read-only — `StatelessSession`. Это специальный режим Hibernate, в котором вообще нет persistence context: никакого L1-кэша, никакого dirty checking, никакого каскадного flush. Каждая операция (insert/update/delete/select) напрямую превращается в SQL, а результаты — в detached-объекты. Это похоже на `JdbcTemplate`, но с меппингом через ORM. Такой режим полезен для очень больших batch-чтений/записей, где вы хотите минимального overhead со стороны ORM.

В Spring доступ к `StatelessSession` можно получить через `SessionFactory`, который можно достать из обычного `Session`. Код получается чуть более низкоуровневым: вы сами отвечаете за открытие/закрытие `StatelessSession` и за то, чтобы не смешивать его с обычным `EntityManager` внутри одной транзакции.

```java
package com.example.jpa.readonly;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.StatelessSession;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class StatelessReportService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<Order> loadAllOrdersStateless() {
        Session session = entityManager.unwrap(Session.class);
        SessionFactory sessionFactory = session.getSessionFactory();

        try (StatelessSession stateless = sessionFactory.openStatelessSession()) {
            return stateless
                    .createQuery("select o from Order o", Order.class)
                    .list();
        }
    }
}
```

```kotlin
package com.example.jpa.readonly

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.hibernate.Session
import org.hibernate.StatelessSession
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class StatelessReportService(

    @PersistenceContext
    private val entityManager: EntityManager
) {

    @Transactional(readOnly = true)
    fun loadAllOrdersStateless(): List<Order> {
        val session = entityManager.unwrap(Session::class.java)
        val sessionFactory = session.sessionFactory

        StatelessSession(sessionFactory.openStatelessSession()).use { stateless ->
            @Suppress("UNCHECKED_CAST")
            return stateless.createQuery(
                "select o from Order o",
                Order::class.java
            ).list()
        }
    }
}
```

У `StatelessSession` есть серьёзные ограничения: не работает каскадирование, не обрабатываются события (`@PrePersist` и т.п.), нет L1-кэша. Вы должны чётко понимать, что получаете и чего лишаетесь. Но для сценариев типа «прочитать 10 миллионов строк, выгрузить в файл» или «сыграть в большой ETL» это может быть отличным вариантом, когда обычный `EntityManager` уже душит вас памятью и overhead’ом.

В итоге read-only оптимизации сводятся к нескольким уровням: на верхнем — просто `@Transactional(readOnly = true)` для чтений; глубже — DTO-проекции и `setReadOnly(true)` на уровне запросов; ещё глубже — `@Immutable` для действительно неизменяемых сущностей и `StatelessSession` для тяжёлых batch-сценариев. Чем ниже вы опускаетесь, тем больше ручной ответственности берёте, но тем меньше платите за сервис Hibernate при чтении.

---

## Кэш 1-го уровня как защита от двойных SELECT, но источник «устаревших» данных — критерии и баланс

Persistence context = L1-кэш по сути представляет из себя identity map: внутри `EntityManager` есть мапа `EntityKey → entity instance`. Каждый раз, когда вы делаете `find()`/`getReference()`/загружаете сущность через JPQL, Hibernate сначала смотрит в этот кэш, а только потом идёт в БД. Это даёт два ключевых свойства: защита от повторных SELECT по одному и тому же id в рамках транзакции и гарантия, что в пределах контекста для каждой записи в БД будет максимум один Java-объект.

С точки зрения производительности это отлично: если в одном сервисном методе вы дважды вызываете `findById(42L)`, в БД уйдёт только один SELECT, а второй вызов вернёт тот же объект из L1-кэша. Это особенно заметно, когда вы используете ассоциации: загрузили заказ, прошлись по списку позиций, у каждой позиции взяли ссылку на товар — Hibernate под капотом сделает JOIN или несколько SELECT’ов, но по каждому id товара будет только по одному объекту в контексте.

Ещё один плюс L1-кэша — согласованность внутри транзакции. Если вы загрузили сущность, поменяли в ней поле и потом снова её прочитали (даже через JPQL), вы увидите уже изменённое значение, даже если flush ещё не происходил. Для бизнес-логики это естественное поведение: «я в рамках операции работаю с целостным объектом, и любые мои изменения тут же видны». Hibernate за счёт identity map и dirty checking это обеспечивает.

Простейший пример: сервис, который дважды читает одну и ту же сущность в рамках транзакции. При включённом SQL-логе вы увидите только один SELECT, хотя вызовы репозитория два. Это и есть эффект L1-кэша. Именно поэтому многие «подозрительные» N+1-сценарии в тестах не так страшны, как выглядят на уровне кода: пока вы не выходите за рамки одного контекста, Hibernate не будет дёргать базу лишний раз по одному и тому же id.

```java
package com.example.jpa.l1cache;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AccountReadService {

    private final AccountRepository repository;

    public AccountReadService(AccountRepository repository) {
        this.repository = repository;
    }

    @Transactional(readOnly = true)
    public void readTwice(Long id) {
        Account first = repository.findById(id)
                .orElseThrow();

        Account second = repository.findById(id)
                .orElseThrow();

        System.out.println("Same instance: " + (first == second));
    }
}
```

```kotlin
package com.example.jpa.l1cache

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class AccountReadService(
    private val repository: AccountRepository
) {

    @Transactional(readOnly = true)
    fun readTwice(id: Long) {
        val first = repository.findById(id).orElseThrow()
        val second = repository.findById(id).orElseThrow()

        println("Same instance: ${first === second}")
    }
}
```

При этом важно понимать, что L1-кэш живёт только в рамках текущего `EntityManager`/транзакции. Как только транзакция завершилась, контекст уничтожается, и при следующем запросе всё начинается с нуля. Это не распределённый кеш, не L2-кэш, а чисто внутритранзакционный механизм. Он не уменьшает количество запросов между разными веб-запросами, он лишь оптимизирует работу внутри одной операции.

Обратная сторона медали: L1-кэш может стать источником «устаревших» данных, если вы в одной транзакции комбинируете JPA-обновления, bulk-DML или прямые SQL-операции по той же таблице. Например, вы загрузили сущность, потом где-то в этом же `EntityManager` выполнили `nativeQuery` с `UPDATE` по той же строке, а потом снова обращаетесь к сущности — в L1-кэше по-прежнему лежит старая версия. Hibernate не знает, что вы руками обошли его и что-то поменяли в БД.

Чтобы явно сказать Hibernate «перечитай эту сущность из БД», есть метод `EntityManager.refresh(entity)`. Он берёт primary key объекта, делает SELECT в БД и обновляет поля объекта в соответствии с текущим состоянием таблицы. После `refresh()` L1-кэш и БД снова синхронизированы по этой сущности. Это единственный правильный способ «насильно» подтянуть изменения, сделанные вне обычного JPA-потока.

```java
package com.example.jpa.l1cache;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AccountRefreshService {

    private final AccountRepository repository;

    @PersistenceContext
    private EntityManager entityManager;

    public AccountRefreshService(AccountRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void demoRefresh(Long id) {
        Account account = repository.findById(id)
                .orElseThrow();

        long balanceBefore = account.getBalance();

        // ... где-то здесь могли произойти изменения в БД другим процессом ...

        entityManager.refresh(account);

        long balanceAfter = account.getBalance();

        System.out.printf("Before: %d, after refresh: %d%n",
                balanceBefore, balanceAfter);
    }
}
```

```kotlin
package com.example.jpa.l1cache

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class AccountRefreshService(

    private val repository: AccountRepository,

    @PersistenceContext
    private val entityManager: EntityManager
) {

    @Transactional
    fun demoRefresh(id: Long) {
        val account = repository.findById(id).orElseThrow()

        val balanceBefore = account.balance

        // ... в это время кто-то другой мог изменить баланс в БД ...

        entityManager.refresh(account)

        val balanceAfter = account.balance

        println("Before: $balanceBefore, after refresh: $balanceAfter")
    }
}
```

Тот же эффект возникает и после bulk-операций (`update/delete` через JPQL или native), о которых говорили в предыдущей подтеме. Bulk-DML обходит L1-кэш и меняет строки напрямую в БД. L1-кэш при этом остаётся со старыми значениями, и любые чтения сущностей из него не увидят изменений. Поэтому после bulk-операций всегда либо делают `clear()` контекста, либо аккуратно `refresh()` нужные сущности. Иначе вы получаете тихую неконсистентность между памятью и базой.

Также нужно помнить, что L1-кэш — это не просто map, а map + «грязные» сущности, за которыми Hibernate следит. Чем больше сущностей вы загружаете и оставляете managed, тем больше работы для dirty checking при каждом `flush()`/запросе (в `FlushMode.AUTO`). Поэтому хорошая практика — не тянуть в одну транзакцию слишком много несвязанных сущностей и не использовать «универсальные» сервисы, которые за один вызов загружают половину базы «на всякий случай».

Баланс здесь такой: для обычных CRUD-операций вы доверяете L1-кэшу и не вмешиваетесь — он даёт вам и меньше SELECT, и локальную согласованность. Как только вы начинаете делать bulk-обновления, прямой JDBC или несколько независимых действий в одной транзакции — вы явно управляете контекстом: `clear()` там, где нужно, `refresh()` там, где важно получить свежие данные. Важно не пытаться использовать L1-кэш как «мини-L2», то есть не рассчитывать, что он «прикроет» вас между разными запросами или потоками — для этого он не предназначен.

В завершение: L1-кэш — это одновременно ваш друг и потенциальный враг. Он защищает от лишних SELECT и даёт естественную модель работы с объектами внутри транзакции. Но как только вы начинаете обходить JPA через native SQL, bulk-операции или слишком длинные транзакции, он может подложить мину из устаревших данных или избыточного потребления памяти. Ключ к здравому использованию — чёткие границы транзакций, понимание того, что происходит в контексте, и осознанное применение `clear()`/`refresh()` там, где вы нарушаете «обычный поток» ORM.

# 6. Сложные маппинги и модель

## Наследование: SINGLE_TABLE vs JOINED vs TABLE_PER_CLASS — компромиссы производительности/нормализации

В объектной модели наследование — абсолютно естественная вещь: у вас есть базовый класс `Payment`, от него наследуются `CardPayment`, `BankTransfer`, `QrPayment` и т.д. В реляционной же модели таблиц наследования нет, и JPA/Hibernate должны как-то «развернуть» иерархию классов в набор таблиц и связей. Выбор стратегии наследования напрямую влияет на структуру схемы, количество join’ов, объём данных и производительность запросов. Если отнестись к этому как к случайному выбору, очень легко получить либо раздутую таблицу, либо сложные запросы, либо очень тяжёлую поддержку в будущем.

Стратегия `SINGLE_TABLE` хранит всю иерархию в одной таблице, с дискриминаторным столбцом и набором nullable-колонок под поля всех подклассов. Это самый быстрый по чтению вариант: один `select` по одной таблице, никакого join’инга, всё прямолинейно. Но за скорость вы платите денормализацией: в таблице много колонок, из которых большая часть для конкретного типа просто `null`, и по мере роста иерархии таблица пухнет. Для одних доменов это нормально, для других — превращается в проблему с управляемостью схемы.

Плюсы `SINGLE_TABLE` очевидны: простые запросы, минимум join’ов, хорошая производительность на типичных CRUD-операциях, особенно когда запросы идут по pk или по общим полям базового класса. Минусы — ограниченная эволюция (добавление новых типов ведёт к модификации одной и той же таблицы), необходимость жить с кучей nullable-полей, сложность для DBA, которые привыкли видеть более нормализованную схему. Плюс сюда добавляются тонкости с ограничениями: например, уникальность на уровне конкретного подкласса приходится обеспечивать сложными частичными индексами или триггерами.

Стратегия `JOINED` раскладывает наследование на нормализованные таблицы: общие поля — в базовой таблице, специфичные для подкласса — в отдельной таблице с тем же `id` и `foreign key` на базовую. При чтении конкретного подкласса Hibernate делает join по этим таблицам, получая полную картину. Это хорошая середина между нормализацией и читабельностью: общие данные живут в одном месте, специфичные — в своих таблицах, схема читается, а антипаттерн с десятком nullable-колонок в одной таблице исчезает.

С `JOINED` вы выигрываете в нормализации и структурной чистоте схемы, но платите join’ами. Если у вас длинная иерархия, запрос к самому нижнему подклассу превращается в join по нескольким таблицам. Для OLTP-нагрузок это зачастую приемлемо, но для очень горячих таблиц и тяжёлых запросов нужно внимательно смотреть на планы выполнения. Отдельный нюанс: вставка нового экземпляра подкласса — это уже минимум два INSERT’а (в базовую и в таблицу подкласса), что при агрессивных батчах и массовых вставках тоже имеет значение.

Стратегия `TABLE_PER_CLASS` создаёт отдельную таблицу под каждый конкретный класс, включая базовый. Для каждого подкласса есть своя таблица с полным набором полей, и базовый класс не представлен одной общей таблицей. При запросе по базовому типу Hibernate вынужден делать `UNION` по всем таблицам подклассов. Это может быть удобно, когда вы почти всегда работаете с подклассами и почти никогда — с базовым типом, но в большинстве бизнес-домена этот вариант оказывается самым тяжёлым по запросам и редко подходит под требовательные к производительности системы.

Выбор стратегии должен быть осознанным. Если у вас небольшая иерархия и много запросов по «общим» полям, `SINGLE_TABLE` часто выигрывает и по простоте, и по скорости. Если важна нормализация, чёткая схема и количество типов растёт, `JOINED` обычно предпочтительнее. `TABLE_PER_CLASS` чаще всего оправдан либо для очень специфичных аналитических задач, либо для случаев, когда базовый тип почти не используется и каждый подкласс живёт в своём микромире. В микросервисной архитектуре с узкими bounded context’ами вы обычно можете спроектировать модель так, чтобы не упираться в TABLE_PER_CLASS.

Важно помнить и про практические ограничения ORM. У `SINGLE_TABLE` есть дискриминаторный столбец, который должен быть корректно заполнен и проиндексирован, иначе поиск по типу будет медленным. У `JOINED` есть риск неоптимальных join-планов, если не следить за индексами и не анализировать реальные запросы. У `TABLE_PER_CLASS` запросы по базовому типу могут вообще стать неприемлемыми на больших объёмах данных из-за `UNION` по множеству таблиц. Каждый из подходов требует своих DBA-практик, а не живёт «по умолчанию».

Наконец, в продакшене очень часто оказывается, что наследование как таковое можно заменить композицией или просто отдельными сущностями с общими полями в `@MappedSuperclass`. Это менее «объектно чисто», но даёт более предсказуемое поведение и структуру БД. Если иерархия поведения между типами совпадает не всегда, а данные различаются радикально, часто честнее сделать несколько отдельных агрегатов, чем пытаться загнать всё в одну JPA-иерархию и потом разбираться с дискриминаторами и каскадами.

В качестве иллюстрации рассмотрим классический пример с платёжными средствами. Мы сделаем базовый класс `BillingDetails` и два подкласса — `CreditCard` и `BankAccount` — и промапим их через `SINGLE_TABLE`. Это хорошая демонстрация, как работает дискриминаторный столбец и как в коде выглядит стратегия наследования. Код ниже показывает сами сущности и репозиторий для работы с ними.

```java
package com.example.jpa.inheritance;

import jakarta.persistence.*;

@Entity
@Table(name = "billing_details")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "billing_type")
public abstract class BillingDetails {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @Column(name = "owner", nullable = false)
    private String owner;

    // геттеры/сеттеры

    public Long getId() {
        return id;
    }

    public String getOwner() {
        return owner;
    }

    public void setOwner(String owner) {
        this.owner = owner;
    }
}

@Entity
@DiscriminatorValue("CARD")
class CreditCard extends BillingDetails {

    @Column(name = "card_number", nullable = false)
    private String cardNumber;

    @Column(name = "exp_month", nullable = false)
    private int expMonth;

    @Column(name = "exp_year", nullable = false)
    private int expYear;

    // геттеры/сеттеры
}

@Entity
@DiscriminatorValue("BANK")
class BankAccount extends BillingDetails {

    @Column(name = "account_number", nullable = false)
    private String accountNumber;

    @Column(name = "bank_name", nullable = false)
    private String bankName;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.inheritance

import jakarta.persistence.*

@Entity
@Table(name = "billing_details")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "billing_type")
abstract class BillingDetails(

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    var id: Long? = null,

    @Column(name = "owner", nullable = false)
    var owner: String = ""
)

@Entity
@DiscriminatorValue("CARD")
class CreditCard(

    owner: String = "",

    @Column(name = "card_number", nullable = false)
    var cardNumber: String = "",

    @Column(name = "exp_month", nullable = false)
    var expMonth: Int = 0,

    @Column(name = "exp_year", nullable = false)
    var expYear: Int = 0
) : BillingDetails(owner = owner)

@Entity
@DiscriminatorValue("BANK")
class BankAccount(

    owner: String = "",

    @Column(name = "account_number", nullable = false)
    var accountNumber: String = "",

    @Column(name = "bank_name", nullable = false)
    var bankName: String = ""
) : BillingDetails(owner = owner)
```

Этот пример подчёркивает, что выбор стратегии наследования — это не просто аннотация на базовом классе. За ней стоит конкретная структура таблицы, набор ограничений и профиль запросов. Если вы заранее понимаете, как реальные запросы будут работать по этой иерархии, вы сможете выбрать подходящую стратегию и не столкнуться с неожиданным деградом через год эксплуатации.

---

## Связи: избегаем «грузных» `@ManyToMany` — предпочтительна явная сущность связи (join-entity)

В объектном мире связь многие-ко-многим кажется естественной: у студента может быть много курсов, у курса — много студентов. В JPA это легко маппится через `@ManyToMany`, и первое время всё выглядит прекрасно. Но по мере развития системы оказывается, что к связи хочется добавить дополнительные атрибуты (например, дату записи на курс, статус, оценку), повлиять на каскады, оптимизировать запросы — и простой `@ManyToMany` начинает ограничивать. Поэтому в продакшн-коде гораздо чаще используют явную сущность связи (join-entity), а `@ManyToMany` оставляют для очень простых и редко меняющихся кейсов.

Классический `@ManyToMany` в JPA — это две сущности, каждая из которых ссылается на другую через коллекцию, а Hibernate под капотом создаёт третью таблицу для join’а. Таблица обычно содержит два столбца с внешними ключами, и на нее не маппится отдельный класс. Это удобно, когда связь — чистая и без дополнительных данных, и когда вы уверены, что это не изменится. Как только бизнес приходит и говорит «давайте добавим дату начала, приоритет, ограничение по ролям», такая схема начинает ломаться.

Проблема «грузных» `@ManyToMany` не только в том, что туда невозможно добавить поля. Важен контроль над направлением и каскадами. При двусторонней связи легко получить неожиданные каскадные обновления и удаление строк в join-таблице, особенно если вы используете `cascade = CascadeType.ALL` не очень осознанно. Hibernate пытается синхронизировать обе стороны связи, и при неправильном управлении коллекциями может генерировать лишние DELETE/INSERT, что резко ухудшает производительность.

Подход с явной сущностью связи решает почти все эти проблемы. Вместо `ManyToMany` вы моделируете два `ManyToOne` на отдельную сущность, например `Enrollment`, которая связывает `Student` и `Course` и несёт дополнительные атрибуты (дата регистрации, статус, источник). Эта сущность становится полноправным участником домена: по ней можно строить репозитории, индексы, накладывать уникальные ограничения и правила валидации. Для ORM это просто обычная сущность без магии.

Отдельный плюс join-entity — предсказуемость SQL. JPA с `@ManyToMany` генерирует очистку join-таблицы при изменении коллекций весьма агрессивно, иногда предпочитая «удалить все связи и вставить заново», если не может построить diff. Явная связь же ведёт себя как обычная сущность: вы явно создаёте новые записи, явно удаляете ненужные, можете использовать `@BatchSize` или `JOIN FETCH`, а также точечно оптимизировать запросы.

Композиция через join-entity даёт и более чистую модель с точки зрения домена. Появляется возможность отразить смысл: «запись на курс», «участие в проекте», «подписка» — это обычно отдельное понятие в предметной области, а не просто «строчка в join-таблице». Как только вы начинаете думать о связи как о сущности, появляются поля и правила, которые очень сложно выразить через голый `@ManyToMany` без перехода к более низкоуровневому маппингу.

Посмотрим на код. Сначала пример «как делать не стоит» — простой `@ManyToMany` между `Student` и `Course`. Он работает, но очень быстро упрётся в ограничения.

```java
package com.example.jpa.relations;

import jakarta.persistence.*;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "students")
public class Student {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(
            name = "student_course",
            joinColumns = @JoinColumn(name = "student_id"),
            inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    // геттеры/сеттеры
}

@Entity
@Table(name = "courses")
class Course {

    @Id
    @GeneratedValue
    private Long id;

    private String title;

    // обратная сторона можно добавить, но опустим для краткости
}
```

```kotlin
package com.example.jpa.relations

import jakarta.persistence.*
import java.util.*

@Entity
@Table(name = "students")
class Student(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = ""
) {

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = [JoinColumn(name = "student_id")],
        inverseJoinColumns = [JoinColumn(name = "course_id")]
    )
    var courses: MutableSet<Course> = HashSet()
}

@Entity
@Table(name = "courses")
class Course(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var title: String = ""
)
```

Теперь — вариант с явной сущностью `Enrollment`. Здесь мы отказываемся от `@ManyToMany` и строим модель через `@ManyToOne` с обеих сторон. `Enrollment` получает дополнительные поля, в БД под неё заводится отдельная таблица с собственным pk и fk на `students` и `courses`. В будущем мы можем добавлять туда атрибуты сколько угодно, не ломая модель.

```java
package com.example.jpa.relations;

import jakarta.persistence.*;

import java.time.LocalDate;

@Entity
@Table(
        name = "enrollments",
        uniqueConstraints = @UniqueConstraint(
                name = "uk_enrollment_student_course",
                columnNames = {"student_id", "course_id"}
        )
)
public class Enrollment {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    private Course course;

    @Column(name = "enrolled_at", nullable = false)
    private LocalDate enrolledAt;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private EnrollmentStatus status;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.relations

import jakarta.persistence.*
import java.time.LocalDate

@Entity
@Table(
    name = "enrollments",
    uniqueConstraints = [
        UniqueConstraint(
            name = "uk_enrollment_student_course",
            columnNames = ["student_id", "course_id"]
        )
    ]
)
class Enrollment(

    @Id
    @GeneratedValue
    var id: Long? = null,

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    var student: Student? = null,

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "course_id")
    var course: Course? = null,

    @Column(name = "enrolled_at", nullable = false)
    var enrolledAt: LocalDate = LocalDate.now(),

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    var status: EnrollmentStatus = EnrollmentStatus.ACTIVE
)
```

Такой подход даёт полный контроль: вы можете навесить уникальный индекс на пару `(student_id, course_id)`, следить за статусом записи, хранить историю, статистику и т.д. Модель остаётся прозрачной для JPA и SQL, проблем с «грузным» `@ManyToMany` и невозможностью расширяться у вас не возникает. В реальных системах join-entity — почти всегда лучший выбор.

---

## Встроенные значения: `@Embeddable` для value-объектов; `@MappedSuperclass` для общих полей, не для связей

Часть данных в домене естественно живёт как «значения», а не как «сущности». Адрес, диапазон дат, денежная величина, координаты — у этих объектов нет самостоятельной идентичности в БД, их жизнь зависит от их владельца, и сравнивать их логично по значениям полей, а не по id. В JPA таким объектам хорошо подходит `@Embeddable`: это класс, чьи поля «встраиваются» в таблицу владельца, без отдельной таблицы и fk, но при этом остаются отдельным типом в коде с собственными инвариантами и логикой.

`@Embeddable` говорит Hibernate, что поля этого класса нужно разложить по колонкам таблицы сущности, где он используется. Владелец помещает такое значение через `@Embedded` или просто по имени поля. Вы получаете сильную типизацию в коде и избежание копипасты полей по сущностям, при этом схема БД остаётся простой: вместо `address` как отдельной таблицы вы храните `address_street`, `address_city` и т.п. рядом с основными колонками.

Преимущество `@Embeddable` ещё и в том, что вы можете сконцентрировать доменную логику значения в одном классе: валидировать поля в конструкторе, реализовать `equals/hashCode` по всем компонентам, добавить методы вроде `isInCountry(String)` или `format()`. Вместо того чтобы тянуть разбросанные поля по сущностям, вы работаете с полноценным value object, что делает модель богаче и код чище.

Есть, однако, несколько подводных камней. Во-первых, изменения внутри `@Embeddable` отслеживаются Hibernate как изменения полей сущности. Это значит, что при любой модификации value-object’а будет генерироваться `UPDATE` по сущности. Во-вторых, embeddable не имеет собственного жизненного цикла и не может быть связью к другим сущностям: в `@Embeddable` не стоит определять `@ManyToOne` и подобное, это уже сильный запах плохого дизайна. Value object должен быть действительно «значением», а не полусущностью.

`@MappedSuperclass` решает другую задачу — он позволяет вынести общие поля и маппинг в базовый класс, от которого наследуются «настоящие» сущности. Такой класс не является сущностью сам по себе, по нему нельзя сделать запрос, но его поля будут промаплены в таблицы подклассов. Типичный случай — аудитные поля (`createdAt`, `updatedAt`, `createdBy`), технические флаги, общие enum’ы и т.п. Это удобно, когда таких полей много и они повторяются в десятках сущностей.

Важно понимать разницу: `@Embeddable` — это «встроенный» объект-значение, который живёт как поле сущности; `@MappedSuperclass` — это про наследование иерархии сущностей. Использовать mapped superclass для связей (`@ManyToOne` к общему родителю) почти всегда плохая идея, потому что вы начнёте тянуть в базовый класс куски конкретных агрегатов. В результате весь домен запутается: подклассы станут неотделимы друг от друга, а схема будет странно переплетена fk через классы, которые даже не являются сущностями.

Хорошая практика — держать `@MappedSuperclass` максимально «техническим», без бизнес-полей и связей. Audit, soft delete, оптимистическая версия, технический идентификатор — нормальные кандидаты. Любые бизнес-смыслы и особенно навигационные связи лучше оставить на уровне собственных сущностей, даже ценой некоторого дублирования.

Посмотрим на пример с адресом и audit-базовым классом. Мы создадим `Address` как `@Embeddable` и будем использовать его в сущности `Customer`. Параллельно определим `Auditable` как `@MappedSuperclass` с полями `createdAt`/`updatedAt`, от которого будет наследоваться сущность `Order`.

```java
package com.example.jpa.embedded;

import jakarta.persistence.*;
import java.time.Instant;

@Embeddable
public class Address {

    @Column(name = "address_street", nullable = false)
    private String street;

    @Column(name = "address_city", nullable = false)
    private String city;

    @Column(name = "address_zip", nullable = false)
    private String zip;

    protected Address() {
    }

    public Address(String street, String city, String zip) {
        this.street = street;
        this.city = city;
        this.zip = zip;
    }

    // геттеры, equals/hashCode и доменные методы
}

@Entity
@Table(name = "customers")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Embedded
    private Address address;

    // геттеры/сеттеры
}

@MappedSuperclass
public abstract class Auditable {

    @Column(name = "created_at", nullable = false, updatable = false)
    protected Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    protected Instant updatedAt;

    @PrePersist
    protected void onCreate() {
        Instant now = Instant.now();
        createdAt = now;
        updatedAt = now;
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = Instant.now();
    }
}

@Entity
@Table(name = "orders")
public class Order extends Auditable {

    @Id
    @GeneratedValue
    private Long id;

    private Long amount;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.embedded

import jakarta.persistence.*
import java.time.Instant

@Embeddable
class Address(

    @Column(name = "address_street", nullable = false)
    var street: String = "",

    @Column(name = "address_city", nullable = false)
    var city: String = "",

    @Column(name = "address_zip", nullable = false)
    var zip: String = ""
)

@Entity
@Table(name = "customers")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = "",

    @Embedded
    var address: Address = Address()
)

@MappedSuperclass
abstract class Auditable {

    @Column(name = "created_at", nullable = false, updatable = false)
    protected var createdAt: Instant? = null

    @Column(name = "updated_at", nullable = false)
    protected var updatedAt: Instant? = null

    @PrePersist
    protected fun onCreate() {
        val now = Instant.now()
        createdAt = now
        updatedAt = now
    }

    @PreUpdate
    protected fun onUpdate() {
        updatedAt = Instant.now()
    }
}

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var amount: Long = 0
) : Auditable()
```

Этот пример показывает, как грамотно разделять обязанности. Адрес реализован как embeddable и не претендует на самостоятельную жизнь. Audit реализован как mapped superclass и переиспользуется во многих сущностях. Мы сознательно не добавляем в базовый класс никаких `@ManyToOne` и других связей, чтобы не превратить его в «бога домена». Такое разделение даёт чистую схему и предсказуемое поведение ORM.

---

## События/слушатели: `@PrePersist/@PreUpdate` и entity-listener’ы — минимум логики, без I/O

Жизненный цикл сущности в Hibernate проходит через ряд стадий, и на этих стадиях можно повесить слушателей: `@PrePersist`, `@PostPersist`, `@PreUpdate`, `@PostUpdate`, `@PreRemove`, `@PostRemove`, `@PostLoad`. Эти callbacks позволяют автоматически проставлять поля (audit, дефолты), выполнять простые проверки и трансформации перед сохранением, а также реагировать на загрузку. Это мощный механизм, но его очень легко превратить в источник скрытой бизнес-логики и хаотичных побочных эффектов.

Основное правило здорового использования entity listeners — держать их максимально лёгкими и локальными. Они должны заниматься тем, что тесно связано с самой сущностью и не требует внешних ресурсов: установить timestamp, нормализовать строку, проставить дефолтный статус, вычислить derived-поле. Всё, что связано с I/O (HTTP, Kafka, файловая система), тяжёлыми вычислениями или обращением к другим агрегатам — не место для lifecycle-колбеков. Такие вещи лучше выносить в сервисный слой, где их проще контролировать, тестировать и оборачивать в retry/circuit breaker.

Слушатели можно определять прямо в сущности через методы, помеченные `@PrePersist` и т.п., а можно вынести в отдельный класс и подключать через `@EntityListeners`. Второй подход удобен, когда вы хотите переиспользовать одну и ту же логику между несколькими сущностями, например аудит. Отдельный listener-класс остаётся обычным POJO с методами, принимающими сущность как аргумент, и его проще тестировать отдельно, чем разбросанный по нескольким entity код.

Важно помнить, что lifecycle-колбеки выполняются внутри того же `EntityManager` и в рамках той же транзакции, что и операция сохранения. Если внутри listener’а вы бросите непроверенное исключение, Hibernate откатит транзакцию. Поэтому код слушателей должен быть максимально детерминированным и предсказуемым. Любые случайные NPE или ошибки в валидации превращаются в непонятные падения на уровне JPA, что усложняет отладку.

Ещё один момент — слушатели вызываются и при merge detached-сущностей. Если вы активно используете `merge`, стоит учитывать, что `@PreUpdate` будет дергаться и в этих сценариях. Если в listener’ах есть какая-то логика, которая не рассчитана на merge, вы можете получить неожиданные эффекты. Это ещё один аргумент делать listener’ы максимально простыми и не зависящими от контекста вызова.

Хорошая практика — использовать listeners только для «технических» задач: аудит, технические флаги, инварианты самой сущности. Бизнес-правила и кросс-агрегатное взаимодействие лучше держать в доменных сервисах. Слушатель должен корректно отрабатывать даже при массовых операциях, транзакционных ретраях и параллелизме — то есть не иметь скрытых зависимостей от внешнего состояния или глобальных синглтонов.

Посмотрим на пример с аудитом через отдельный listener-класс. Мы создадим `AuditableEntity` как базовый класс с полями `createdAt`/`updatedAt` и подключим к нему `AuditEntityListener`, который проставляет timestamps перед сохранением и обновлением. Логика проста, не зависит от внешних систем и безопасна для выполнения внутри транзакции.

```java
package com.example.jpa.listener;

import jakarta.persistence.*;
import java.time.Instant;

@MappedSuperclass
@EntityListeners(AuditEntityListener.class)
public abstract class AuditableEntity {

    @Column(name = "created_at", nullable = false, updatable = false)
    protected Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    protected Instant updatedAt;

    public Instant getCreatedAt() {
        return createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }
}

public class AuditEntityListener {

    @PrePersist
    public void prePersist(AuditableEntity entity) {
        Instant now = Instant.now();
        entity.createdAt = now;
        entity.updatedAt = now;
    }

    @PreUpdate
    public void preUpdate(AuditableEntity entity) {
        entity.updatedAt = Instant.now();
    }
}

@Entity
@Table(name = "customers")
class Customer extends AuditableEntity {

    @Id
    @GeneratedValue
    private Long id;

    private String name;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.listener

import jakarta.persistence.*
import java.time.Instant

@MappedSuperclass
@EntityListeners(AuditEntityListener::class)
abstract class AuditableEntity {

    @Column(name = "created_at", nullable = false, updatable = false)
    protected var createdAt: Instant? = null

    @Column(name = "updated_at", nullable = false)
    protected var updatedAt: Instant? = null

    fun getCreatedAt(): Instant? = createdAt
    fun getUpdatedAt(): Instant? = updatedAt
}

class AuditEntityListener {

    @PrePersist
    fun prePersist(entity: AuditableEntity) {
        val now = Instant.now()
        entity.createdAt = now
        entity.updatedAt = now
    }

    @PreUpdate
    fun preUpdate(entity: AuditableEntity) {
        entity.updatedAt = Instant.now()
    }
}

@Entity
@Table(name = "customers")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var name: String = ""
) : AuditableEntity()
```

В этом примере listener выполняет ровно одну задачу: проставляет время создания и обновления. Он не ходит в другие таблицы, не дергает внешние сервисы, не пишет в логи всего подряд. Такой код спокойно переживает массовые вставки, транзакционные ретраи и параллельные запросы. Если позже вы захотите добавить, например, запись пользователя-создателя, можно передавать эту информацию через `Auditing`-инфраструктуру Spring Data, а не пытаться внутри listener’а тянуть `SecurityContextHolder` или что-то подобное напрямую.

Итог по listeners простой: это удобный инструмент для автоматизации технических аспектов жизненного цикла сущности, но очень плохое место для бизнес-логики и I/O. Если следовать правилу «минимум, без внешних эффектов», listener’ы станут союзником, а не источником «магических» багов, которые невозможно воспроизвести в тестах.

# 7. Кеширование 2-го уровня и кеш запросов

## Когда имеет смысл L2: справочники/редко меняемые сущности; регионы, режимы (read-only/non-strict/transactional)

Второй уровень кеша (L2-кеш Hibernate) — это кеш поверх БД, общий для всех `EntityManager`/Session внутри одного процесса (а при кластерном провайдере — и между процессами). В отличие от контекста персистентности (L1-кеш), который живёт ровно столько, сколько живёт транзакция, L2-кеш хранит данные дольше и переживает множество транзакций и HTTP-запросов. Идея простая: если вы многократно читаете одни и те же сущности и меняете их редко, то выгоднее держать их в памяти и доставать оттуда, чем каждый раз ходить в Postgres. Но за это вы платите дополнительной памятью, сложностью инвалидации и риском столкнуться с «почти устаревшими» данными.

L2-кеш имеет смысл включать прежде всего для справочников и «почти неизменяемых» сущностей. Это всевозможные типы операций, статусы, региональные справочники, иногда — профили тарифов или продуктовые каталоги, которые меняются в основном ночными джобами. Типичный паттерн: у вас есть таблица на десятки тысяч строк, вы по ней много раз читаете «по id» или небольшие выборки, а обновляется она раз в сутки. Классическое L2-кеширование даёт высокий hit ratio, сильно разгружает БД и почти не создаёт проблем с консистентностью.

По умолчанию Hibernate разделяет кеш на регионы — логические «секции», каждая со своей политикой. Для сущностей можно задать concurrency strategy: `READ_ONLY`, `NONSTRICT_READ_WRITE`, `READ_WRITE`, `TRANSACTIONAL` (последняя обычно используется с JTA и провайдерами, поддерживающими XA). В контексте Spring Boot и Postgres чаще всего используются `READ_ONLY` и изредка `NONSTRICT_READ_WRITE`. `READ_ONLY` означает, что Hibernate предполагает полное отсутствие изменений через ORM: сущность один раз загрузили из БД и дальше только читают. В этом режиме кеш проще, быстрее и надёжнее. `NONSTRICT_READ_WRITE` допускает редкие изменения, но без строгих гарантий «прочитал — увидел точно последнюю версию».

Режим `READ_ONLY` идеально подходит для справочников. Ключевое требование — никаких `UPDATE` этой сущности через JPA. Если нужно сменить значение, вы делаете это через админские инструменты или Migra/SQL, а приложение воспринимает это как «редкое событие», после которого кеш можно сбросить целиком или дождаться истечения TTL, если провайдер его поддерживает. Взамен вы получаете очень быстрые чтения: практически все запросы по первичному ключу будут без похода в БД, и Hibernate даже не будет тратить время на сложную синхронизацию кеша.

`NONSTRICT_READ_WRITE` — компромисс: сущность можно иногда изменять через ORM, но провайдер кеша не даёт жёстких гарантий, что сразу после коммита все ноды увидят актуальное значение. Обычно инвалидация делается через «грубые» приёмы: пометить запись как потенциально грязную, сбросить регион целиком или с определённой задержкой. В системах, где допустимо прочитать чуть устаревшие данные (например, в UI-справочнике), это нормальный вариант. Но для критичных к консистентности сущностей (балансы, лимиты, статусы заявок) такой режим опасен.

Важно понимать, что L2-кеш не бесплатен. Каждая сущность в кеше занимает память, а при большом объёме кеша вы получаете более тяжёлый GC: больше объектов в old generation, более длинные паузы при сборке и риск «вылета» в stop-the-world на неподходящий момент. Поэтому кешировать «всё подряд» — плохая идея. Нужно явно выбирать те классы, где отношение «число чтений» к «числу изменений» велико, а размер агрегата относительно небольшой. Если сущность часто меняется, да ещё и имеет большой граф связей, L2-кеш будет только мешать: его постоянно будет инвалидировать, а память забивать.

На уровне конфигурации в Spring Boot включение L2-кеша делается через свойства Hibernate и через аннотации на сущностях. Типичный путь: вы включаете `hibernate.cache.use_second_level_cache=true`, выбираете провайдер (Ehcache, Hazelcast, Infinispan и т.д.) и помечаете нужные сущности `@org.hibernate.annotations.Cache`. Spring-стартера для кеша (например, `spring-boot-starter-cache`) этого не делает автоматически: L2 — именно фича Hibernate, а не Spring Cache API.

Ещё один аспект — кеширование коллекций. Hibernate позволяет кешировать не только отдельные сущности, но и коллекционные ассоциации (`@OneToMany`, `@ManyToMany`). Для этого есть отдельная стратегия concurrency для коллекций. На практике кеш коллекций полезен, когда у вас относительно небольшой и стабильный список элементов, часто читаемый целиком. Но кешировать большие и горячие коллекции опасно: повышение потребления памяти, сложная инвалидация при изменениях и риск попадания в тяжёлые GC-паузы.

Хорошая практика — начинать с очень узкого листа кандидатов на L2-кеш: один–два справочника, возможно, ещё одну важную read-only сущность. Включаете кеш для них, настраиваете регион, следите за метриками (hit/miss, размер, влияние на GC). Если всё хорошо — можно расширять. Если видите резкий рост old-gen и падение latency при остановленных GC, значит, кеш либо слишком велик, либо выбран неправильный набор сущностей. L2-кеш — это не «включил и забыл», а управляемый ресурс.

Простейший пример настройки L2 для справочника на Ehcache в Spring Boot выглядит так: в `application.yml` вы включаете параметры Hibernate для L2 и используете Ehcache как провайдер, а сущность помечаете как `@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)`. Ниже — минимальный пример конфигурации и сущности для Java и Kotlin.

```java
// application.yml (фрагмент)
//
// spring:
//   jpa:
//     properties:
//       hibernate.cache.use_second_level_cache: true
//       hibernate.cache.use_query_cache: false
//       hibernate.cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
//   cache:
//     jcache:
//       config: classpath:ehcache.xml
```

```java
package com.example.jpa.cache;

import jakarta.persistence.*;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Table(name = "operation_type")
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "operationType")
public class OperationType {

    @Id
    private Integer id;

    @Column(nullable = false, unique = true)
    private String code;

    @Column(nullable = false)
    private String description;

    // геттеры/сеттеры
}
```

```kotlin
// application.yml (фрагмент)
//
// spring:
//   jpa:
//     properties:
//       hibernate.cache.use_second_level_cache: true
//       hibernate.cache.use_query_cache: false
//       hibernate.cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
//   cache:
//     jcache:
//       config: classpath:ehcache.xml
```

```kotlin
package com.example.jpa.cache

import jakarta.persistence.*
import org.hibernate.annotations.Cache
import org.hibernate.annotations.CacheConcurrencyStrategy

@Entity
@Table(name = "operation_type")
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "operationType")
class OperationType(

    @Id
    var id: Int? = null,

    @Column(nullable = false, unique = true)
    var code: String = "",

    @Column(nullable = false)
    var description: String = ""
)
```

Этот пример не претендует на полноту боевой конфигурации, но хорошо иллюстрирует идею: L2-кеш включается точечно, под конкретные сущности, и для них выбирается консервативный режим `READ_ONLY`. Всё остальное по умолчанию продолжает ходить в БД напрямую, опираясь только на L1-кеш.

---

## Кеш запросов: риск «несогласованности», ключи инвалидации, осторожно с частыми изменениями

Кеш запросов Hibernate (query cache) — это отдельный слой, который кеширует не сами сущности, а результаты конкретных запросов: список идентификаторов и иногда агрегаты. В отличие от L2-кеша, привязанного к id сущности, query cache работает по ключу «HQL/JPQL + параметры + настройки пагинации», и при повторном выполнении того же запроса может вернуть результат из памяти, не обращаясь ни к БД, ни к L2-кешу. Это соблазнительный инструмент для популярных отчётов и списков, но с ним нужно обращаться гораздо осторожнее, чем с кешем сущностей.

Важно понимать базовую архитектуру: кеш запросов не хранит копии сущностей. Он хранит, грубо говоря, «снимок» результата в виде набора primary keys и метаданных. Когда вы повторно выполняете кешированный запрос, Hibernate берет кэшированный список id и для каждого id обращается к L2-кешу (если тот включён) или в БД. В идеальном сценарии все сущности уже есть в L2 и запрос к БД вообще не делается. В менее идеальном — часть сущностей придётся перечитать из БД, но вы всё равно сэкономите на повторном выполнении сложного SQL с join’ами.

Основная проблема кеша запросов — инвалидация. Если вы изменили одну из сущностей, участвующую в запросе, Hibernate должен понять, какие именно записи в кеш запросов теперь устарели. Для простых случаев (one-to-one mapping «таблица–сущность») он знает, в каких регионах cached queries участвуют эти сущности, и может пометить их как dirty. Но на практике часто возникает ситуация, когда запрос включает join’ы, фильтрацию по не кешированным полям, агрегаты, подзапросы — и корректная инвалидация становится нетривиальной. В результате либо кеш инвалидируется слишком грубо (сброс целых регионов), либо риск несогласованности результатов возрастает.

Кеш запросов особенно опасен при частых изменениях данных. Если вы пытаетесь кешировать, например, список активных заказов, которые меняются каждую секунду, вы получите постоянные инвалидации, churn в кеше и минимальную пользу. Более того, есть риск, что какой-то запрос на чтение увидит устаревший результат на несколько миллисекунд/секунд, пока инвалидация не успела пробежаться по всем узлам. Для финансовых и критичных к консистентности сценариев это неприемлемо. Query cache здесь скорее враг, чем друг.

Хорошее правило — кешировать запросы только поверх сущностей, которые уже кешируются во втором уровне и изменяются редко. Тогда query cache работает как дополнительная оптимизация: он избавляет от повторных тяжёлых SELECT’ов, опираясь на уже готовый L2-кеш. Если же L2-кеш выключен, а query cache включён, пользы будет заметно меньше: вы всё равно будете ходить в БД за каждой сущностью по id, просто избежите исполнения «большого» SQL. Иногда это выгодно (например, при тяжёлых join’ах), но гарантировать эффект сложно.

Ключи кеша запросов строятся по полному тексту HQL/JPQL, параметрам, их значениям и ряду дополнительных характеристик (лимит, offset). Это значит, что два почти одинаковых запроса, отличающихся лишь лишним пробелом, или с разными названиями параметров будут кешироваться отдельно. Поэтому крайне важно, чтобы код, который вы хотите кешировать, был стабильным: константная строка JPQL, одинаковый набор параметров, предсказуемый порядок. Динамически собираемые запросы, конструкторы Criteria без доп.слоя, где вы нормализуете ключ, — плохие кандидаты.

Практически в Hibernate query cache включается глобально (`hibernate.cache.use_query_cache=true`), а дальше пометкой конкретных запросов как кешируемых через hint `org.hibernate.cacheable`. Spring Data JPA позволяет использовать `@QueryHints` на уровне метода репозитория, а при явной работе с `EntityManager` можно вызвать `query.setHint("org.hibernate.cacheable", true)`. Важно не пытаться включать кеш «на всё подряд», а явно выбирать 1–2 действительно тяжёлых запроса с хорошими характеристиками (много чтений, мало изменений).

Нужно помнить и про размер результатов. Кешировать «весь каталог из 200 тысяч строк» — почти всегда плохая идея: вы забьёте память и получите тяжёлые GC, а польза окажется сомнительной, потому что такие запросы редко исполняются полностью одинаково (фильтры, пагинация). Query cache особенно хорошо работает для небольших, но часто повторяющихся выборок: «справочник активных тарифов» для конкретного региона или «последние N записей по конкретному ключу». Там размер набора мал, а вероятность повторного запроса высока.

Есть ещё риск «нестабильных» запросов: если вы включили кеш для JPQL, который логически должен всегда возвращать одно и то же, но на самом деле завязан на данные вне наблюдаемой Hibernate модели (например, на view, обновляемый внешним процессом), вы можете легко прочитать устаревший результат и долго не понимать почему. Hibernate не умеет отслеживать изменения во view или в таблицах, на которые вы не повесили сущности; для него это просто «ещё один SELECT». В итоге query cache нужно использовать только там, где вся зависимая от него схема контролируется через ORM.

Пример: у нас есть справочник активных тарифов, который меняется раз в день, а читается на каждом запросе к тарифному сервису. Мы включаем L2-кеш для сущности `Tariff` и кеш запросов для JPQL, который выбирает только активные тарифы. В репозитории Spring Data JPA это можно сделать через `@QueryHints`. Ниже — пример для Java и Kotlin.

```java
package com.example.jpa.querycache;

import jakarta.persistence.QueryHint;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface TariffRepository extends JpaRepository<Tariff, Long> {

    @Query("select t from Tariff t where t.active = true")
    @org.springframework.data.jpa.repository.QueryHints({
            @QueryHint(name = "org.hibernate.cacheable", value = "true"),
            @QueryHint(name = "org.hibernate.cacheRegion", value = "activeTariffs")
    })
    List<Tariff> findAllActive();
}
```

```kotlin
package com.example.jpa.querycache

import jakarta.persistence.QueryHint
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.jpa.repository.QueryHints

interface TariffRepository : JpaRepository<Tariff, Long> {

    @Query("select t from Tariff t where t.active = true")
    @QueryHints(
        value = [
            QueryHint(name = "org.hibernate.cacheable", value = "true"),
            QueryHint(name = "org.hibernate.cacheRegion", value = "activeTariffs")
        ]
    )
    fun findAllActive(): List<Tariff>
}
```

Для полноты картины вы, конечно, должны ещё включить query cache в конфигурации Hibernate и определить регион `activeTariffs` в провайдере кеша. Но главная мысль — кеширование запросов — это точечный инструмент, а не «галочка в настройках». Им пользуются там, где чётко понимают, как и когда будут меняться данные, какие запросы повторяются, и где допустим небольшой риск устаревания.

---

## Распределённость: репликация/инвалидация между инстансами, влияние на память и GC

Когда у вас один инстанс приложения, L2-кеш — это просто in-process структура данных: Hibernate кладёт туда сущности, извлекает их и при изменениях инвалидирует локальные записи. Как только вы переходите к нескольким инстансам (2+ поды в Kubernetes, несколько JVM за балансировщиком), возникает вопрос: что происходит с кешом между ними. Либо L2-кеш становится локальным для каждого инстанса, и вы миритесь с тем, что разные ноды могут видеть слегка разные данные, либо вы подключаете распределённый кеш-провайдер, который обеспечивает репликацию или инвалидацию по сети.

Локальный L2-кеш на каждом инстансе — самый простой вариант. Каждый JVM-процесс хранит свои данные, которые не разделяются с соседями. При изменении сущности Hibernate инвалидирует кеш только на текущем инстансе, а остальные узнают о новых значениях только после следующего чтения из БД. Это даёт вам максимальную скорость (нет сетевого хопа за кешем), но при этом снижает ценность кеша для «горячих» сущностей: вероятность того, что следующий запрос на другой ноде попадёт именно в тот же L2-кеш, заметно ниже. Зато нет сложностей с кластерной конфигурацией и распределённой согласованностью.

Распределённый L2-кеш строится поверх кластерных провайдеров: Hazelcast, Infinispan, Redis (через интеграционные модули), иногда — Ehcache в clustered-режиме. В этом случае каждый инстанс подключается к общей кеш-сети, где регионы кеша хранятся либо в репликативной, либо в партиционированной форме. При изменении сущности провайдер рассылает события инвалидации или реплицирует новые значения на другие ноды. Логика Hibernate остаётся той же, но за кулисами работает распределённый data grid.

Плюс распределённого подхода в том, что любой инстанс может получить кеш-хит по сущности, которую до этого кто-то читал на соседней ноде. При высокой доле чтения и большом количестве инстансов это даёт заметный выигрыш: вместо «теплого» кеша на одной машине вы получаете «теплый» кеш на весь кластер. Но минусы очевидны: добавляется сетевой hop, увеличивается сложность конфигурации, а согласованность кеша становится вопросом дизайна провайдера (синхронная/асинхронная репликация, eventual consistency, и т.д.).

С точки зрения памяти и GC распределённый кеш зачастую даже сложнее, чем локальный. Если вы используете полностью репликативный режим (каждый узел хранит копию всех данных), вы фактически умножаете потребление памяти на число инстансов: каждая JVM держит полный шатл кеша. Это бьёт по old-gen, приводит к тяжёлым GC-паузам и требует тщательного тюнинга размеров кеша и параметров GC (G1/ZGC). Партиционированный режим снимает часть нагрузки (каждый узел хранит только часть данных), но усложняет маршрутизацию данных внутри провайдера.

Ещё один аспект — стратегия обновления/инвалидации. В `READ_WRITE` и `TRANSACTIONAL` режимах L2-кеша нужен корректный кластерный протокол, который гарантирует, что две ноды не одновременно запишут в кеш две разные версии сущности и не разойдутся. Это довольно сложная задача, которую провайдеры решают через блокировки, versioning и транзакционные алгоритмы. В реальных системах это значит, что каждый `UPDATE` через ORM будет не только идти в БД, но и в кеше сопровождаться дополнительными синхронными операциями по сети — цена за согласованность.

Если вы используете проще режимы (`READ_ONLY`, `NONSTRICT_READ_WRITE`), провайдеру можно позволить более лёгкие схемы: асинхронные уведомления об инвалидации, периодический refresh, грубую очистку регионов. Но тогда вы сознательно принимаете eventual consistency: между коммитом на одной ноде и обновлением кеша на другой может пройти время, в течение которого запрос прочитает устаревшие данные из L2. Это нормальная цена для справочников и ряда UI-сценариев, но критична для финансовых и консистентных областей.

Важный практический момент — комбинация L1 и L2. При распределённом провайдере каждый инстанс имеет локальный L1-кеш (persistence context) и «удалённый» L2-кеш. При запросе Hibernate сначала смотрит в L1, потом — в L2, и только потом идёт в БД. Фактически L2 становится шареным «old-generation-кешем» для всей системы, а L1 — маленьким, быстрым локальным буфером. Если данных слишком много, и L2 сильно нагружен, GC будет страдать на каждом инстансе. Поэтому разумная стратегия — держать L2 относительно небольшим и использовать TTL/size-политику, а всё горячее — или читать напрямую из БД, или кешировать в специализированных решениях (например, Redis как отдельный слой, не жестко привязанный к JPA).

С точки зрения Spring Boot интеграция с кластерными провайдерами может быть реализована через Spring Cache + JCache (JSR-107) поверх Hazelcast/Infinispan, которые в свою очередь используются Hibernate как RegionFactory. Тогда вы настраиваете кластер на уровне кеш-провайдера, а Hibernate просто работает с регионами. Пример конфигурации для Hazelcast: создаёте `hazelcast.yaml` с настройками кластера, подключаете `hazelcast-spring` и указываете фабрику регионов `HazelcastCacheRegionFactory` в Hibernate.

Ниже приведён упрощённый пример конфигурации Hazelcast для L2-кеша и соответствующей настройки Hibernate в Spring Boot. Это не честный production-ready кластер, но демонстрирует базовый wiring. Java и Kotlin-код отличаются только синтаксисом конфигурационного класса.

```java
// build.gradle (фрагмент Groovy)
// implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
// implementation 'com.hazelcast:hazelcast'
// implementation 'com.hazelcast:hazelcast-spring'
```

```kotlin
// build.gradle.kts (фрагмент)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.hazelcast:hazelcast")
    implementation("com.hazelcast:hazelcast-spring")
}
```

```yaml
# application.yml (фрагмент)
spring:
  jpa:
    properties:
      hibernate.cache.use_second_level_cache: true
      hibernate.cache.use_query_cache: false
      hibernate.cache.region.factory_class: org.hibernate.cache.hazelcast.HazelcastCacheRegionFactory
      hibernate.javax.cache.missing_cache_strategy: create
  cache:
    hazelcast:
      config: classpath:hazelcast.yaml
```

```java
package com.example.jpa.clusteredcache;

import com.hazelcast.config.Config;
import com.hazelcast.config.JoinConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HazelcastConfig {

    @Bean
    public Config hazelcastConfig() {
        Config config = new Config("jpa-cache-cluster");

        // упрощённая сетевой конфиг, для прода нужен нормальный discovery
        JoinConfig join = config.getNetworkConfig().getJoin();
        join.getMulticastConfig().setEnabled(true);
        join.getTcpIpConfig().setEnabled(false);

        config.getMapConfig("operationType")
                .setTimeToLiveSeconds(3600)
                .setBackupCount(1);

        return config;
    }
}
```

```kotlin
package com.example.jpa.clusteredcache

import com.hazelcast.config.Config
import com.hazelcast.config.JoinConfig
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class HazelcastConfig {

    @Bean
    fun hazelcastConfig(): Config {
        val config = Config("jpa-cache-cluster")

        val join: JoinConfig = config.networkConfig.join
        join.multicastConfig.isEnabled = true
        join.tcpIpConfig.isEnabled = false

        config.getMapConfig("operationType")
            .timeToLiveSeconds = 3600
            .backupCount = 1

        return config
    }
}
```

Здесь мы заводим кластер Hazelcast с простейшей multicast-конфигурацией и описываем карту `operationType`, которую Hibernate будет использовать как регион кеша. В боевых условиях вы дополняете это discovery через Kubernetes API, задаёте более аккуратные TTL и политику резервирования, следите за метриками Hazelcast и GC-профилем на каждом инстансе. Ключевая мысль — распределённый L2-кеш сам по себе сложная распределённая система, и её нужно проектировать и эксплуатировать не менее внимательно, чем сам Postgres/микросервисы.

В итоге для продакшн-проекта здравый подход такой: по умолчанию вы живёте без L2-кеша, полагаясь на грамотные индексы и архитектуру запросов. Для нескольких узких сценариев вы включаете L2 (чаще всего локальный, иногда — кластерный) с `READ_ONLY` для справочников. Query cache включаете ещё более дозированно, только под реально тяжёлые и стабильные запросы. Вся конфигурация кешей сопровождается метриками, алертами и периодическим review. Тогда кеширование станет инструментом, а не миной замедленного действия.

# 8. Чтение больших объёмов и стриминг

Когда данных становится много — десятки и сотни тысяч строк, — наивный подход «считать всё в `List` и потом обрабатывать» превращается в проблему. Вы забиваете память, тратите время на аллокации, создаёте давление на GC и рискуете увидеть `OutOfMemoryError` в самый неподходящий момент. Причём проблема одинаково актуальна и для JPA, и для «чистого» JDBC: любая библиотека, которая пытается материализовать весь результат сразу, упирается в тот же предел.

Решение — перейти от модели «загрузить всё» к модели «стримить и обрабатывать по мере чтения». Вместо того чтобы держать в памяти весь набор, вы читаете по одной строке или по небольшим пачкам, сразу обрабатываете их (записываете в файл, отправляете в другой сервис, агрегируете) и отпускаете. Так нагрузка на память остаётся практически константной, а время жизни объектов становится коротким и дружелюбным к GC.

В JPA/hibernate для этого есть несколько механизмов: потоковый API Spring Data (`Stream<T>`), scrollable/cursor-результаты через Hibernate и прямой JDBC с управлением `fetchSize` и курсорами. Важно понимать, что каждый из этих подходов опирается на тот факт, что JDBC-драйвер и СУБД умеют отдавать результат порциями, а не одним гигантским блоком. Если драйвер всё равно буферизует весь результат у себя, никакой стриминг на уровне ORM уже не спасёт.

Ещё один важный аспект — связка стриминга с транзакциями и `PersistenceContext`. Поскольку большинство серверных курсоров живёт в рамках транзакции, вы не можете «протащить» стрим наружу из `@Transactional`-метода и спокойно итерироваться где-то в другом слое. К моменту, когда вы начнёте перебор, транзакция и соединение уже будут закрыты. Поэтому правильный дизайн — выполнять и получение, и обработку стрима внутри одной транзакции, аккуратно управляя временем её жизни.

И, наконец, стриминг почти всегда сочетается с read-only сценариями: вы просто читаете много данных, но не планируете их изменять. Это означает, что вы можете дополнительно использовать `@Transactional(readOnly = true)`, хинты `setReadOnly(true)` и настройку `fetchSize`, чтобы минимизировать нагрузку и на ORM, и на БД. Запись и стриминг обычно лучше разделять: массовые обновления делаются небольшими батчами, а большие чтения — отдельными потоками/курсорами.

---

## Потоки из репозиториев (`Stream<T>`), требования к `@Transactional` и закрытию

Spring Data JPA умеет возвращать не только `List<T>`, но и `Stream<T>` из методов репозитория. По сути это «ленивый» результат: пока вы не итерируетесь по стриму, JDBC-драйвер не вытягивает все строки. При переборе по стриму драйвер читает строки одна за другой, а Hibernate по мере надобности превращает их в сущности. В идеальном случае это позволяет обрабатывать большие выборки с постоянным потреблением памяти, а не собирать весь набор в `ArrayList`.

Критический момент: стрим, возвращаемый Spring Data JPA, завязан на открытый `EntityManager` и JDBC-соединение. Пока вы итерируетесь внутри активной транзакции, всё хорошо. Как только транзакция завершена и `EntityManager` закрыт, курсор на стороне драйвера закрывается, и дальнейшая попытка читать из стрима приведёт к ошибкам или просто к преждевременному завершению. Поэтому нельзя просто «вернуть Stream из сервисного метода» наружу и ожидать, что его будут безопасно читать в контроллере после завершения транзакции.

Ещё одна особенность: `Stream<T>` от Spring Data реализует `AutoCloseable`, и его нужно закрывать, чтобы освободить JDBC-ресурсы. Если проигнорировать это требование и не закрывать стрим явно или через try-with-resources, вы получаете утечки курсоров и соединений, особенно при многократных тяжелых чтениях. В простых тестах это не видно, а в продакшне вы упрётесь в пул соединений, которые будут «висеть» с открытыми курсорами.

Правильный паттерн работы со стримом из репозитория выглядит так: сервисный метод помечен `@Transactional(readOnly = true)`, внутри него вы вызываете метод репозитория, который возвращает `Stream<T>`, тут же оборачиваете его в try-with-resources (в Kotlin — `use {}`), и внутри этого блока полностью обрабатываете данные. Как только блок заканчивается, стрим закрывается, курсор освобождается, транзакция завершается. Никакого выноса стрима наружу не происходит.

Важно помнить, что даже при стриминге persistence context продолжает расти: каждое прочитанное через JPA сущностное состояние попадает в L1-кэш. Если вы просто итерируетесь по `Stream<Entity>` и ничего не делаете с контекстом, через десятки тысяч строк вы снова начинаете кушать много памяти. Для сценариев с большим количеством строк лучше либо ограничиться DTO-стримом (через `select new ...`), либо периодически очищать контекст (например, detach’ить сущности по мере обработки).

Кроме того, стримы из репозиториев нельзя «параллелить» стандартными методами `stream.parallel()` или `parallelStream()`. Комбинация Hibernate + JDBC + параллельные стримы приведёт к тому, что вы будете пытаться читать из одного курсора сразу несколькими потоками, что ни ORM, ни драйверы не гарантируют как безопасное. Потоковый API Spring Data предполагает последовательную обработку в том потоке, где открыт `EntityManager`.

При тестировании такого кода полезно включить SQL-логирование и убедиться, что запрос к БД выполняется один раз и чтение действительно идёт по мере итерирования. В профилировщике памяти вы должны видеть, что число одновременно живущих сущностей невелико (если вы их не сохраняете в коллекции), а GC не страдает от резких всплесков. Если вы вдруг наблюдаете рост heap и тормоза, значит, где-то в коде вы всё-таки накапливаете данные, вместо того чтобы сразу их обрабатывать и «отпускать».

Практически для стриминга часто делают отдельный сервисный метод, который либо записывает результаты в файл, либо гонит их в потоковую обработку (Kafka, HTTP-стрим и т.д.). Такой метод становится точкой, где чётко контролируется и транзакция, и время жизни стрима. Это лучше, чем завязывать стримы на обычные CRUD-методы, которые разработчики будут потом использовать как «ещё одну версию findAll».

Ниже пример, показывающий потоковый метод репозитория и сервис, который корректно обрабатывает стрим в рамках транзакции. В примере мы читаем всех пользователей, помеченных как активные, и считаем общее количество записей с определённым признаком. Обратите внимание на `try (Stream<User> stream = ...)` в Java и `use {}` в Kotlin.

```java
package com.example.jpa.streaming;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.stream.Stream;

@Entity
@Table(name = "app_user")
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String email;

    private boolean active;

    private boolean flagged;

    // getters/setters
}

interface UserRepository extends JpaRepository<User, Long> {

    @Query("select u from User u where u.active = true")
    Stream<User> streamAllActive();
}
```

```java
package com.example.jpa.streaming;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Stream;

@Service
public class UserStreamingService {

    private final UserRepository userRepository;

    public UserStreamingService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional(readOnly = true)
    public long countFlaggedActiveUsers() {
        AtomicLong counter = new AtomicLong();

        try (Stream<User> stream = userRepository.streamAllActive()) {
            stream.forEach(user -> {
                if (user.isFlagged()) {
                    counter.incrementAndGet();
                }
            });
        }

        return counter.get();
    }
}
```

```kotlin
package com.example.jpa.streaming

import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import java.util.stream.Stream

@Entity
@Table(name = "app_user")
class User(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var email: String = "",

    var active: Boolean = true,

    var flagged: Boolean = false
)

interface UserRepository : JpaRepository<User, Long> {

    @Query("select u from User u where u.active = true")
    fun streamAllActive(): Stream<User>
}
```

```kotlin
package com.example.jpa.streaming

import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.util.concurrent.atomic.AtomicLong

@Service
class UserStreamingService(
    private val userRepository: UserRepository
) {

    @Transactional(readOnly = true)
    fun countFlaggedActiveUsers(): Long {
        val counter = AtomicLong()

        userRepository.streamAllActive().use { stream ->
            stream.forEach { user ->
                if (user.flagged) {
                    counter.incrementAndGet()
                }
            }
        }

        return counter.get()
    }
}
```

В этом шаблоне собраны ключевые принципы: стрим создаётся и обрабатывается внутри транзакции, закрывается автоматически, и ничто не пытается использовать его за пределами этого метода. Такой подход позволяет безопасно и предсказуемо работать с большими выборками через Spring Data JPA.

---

## Скролл/курсор: forward-only, `fetchSize`/`setFetchSize`, минимизация памяти

Помимо `Stream<T>`, Hibernate и JDBC предоставляют более низкоуровневые механизмы для курсорного чтения: scrollable-результаты и управление `fetchSize`. Идея в том, чтобы явно попросить драйвер отдавать данные не одним большим блоком, а небольшими порциями, подстраиваясь под размер батча обработки. Это особенно актуально для JDBC-драйверов, умеющих серверные курсоры (PostgreSQL, Oracle и др.), где `fetchSize` превращается в реальное управление сетевыми round-trip’ами.

По умолчанию многие драйверы игнорируют `fetchSize` или реализуют его как «подсказку» для клиентского буфера. Например, старый PostgreSQL в режиме `autoCommit=true` сначала считывал весь результат в память. Но при `autoCommit=false` и `fetchSize > 0` он переходил в режим использования курсора и начинал реально подтягивать данные партиями. Поэтому первый шаг к эффективному стримингу через курсоры — убедиться, что у вас есть открытая транзакция и драйвер понимает `fetchSize` как сигнал к включению курсорного режима.

Hibernate позволяет управлять `fetchSize` и режимом курсора через API `Query` и `ScrollableResults`. Вы можете вызвать `query.setFetchSize(1000)` и получить либо обычный `List`, который будет загружаться батчами, либо `ScrollableResults`, который позволяет двигаться по результатам вперёд и, в некоторых драйверах, назад. В реальных задачах чаще всего достаточно простого forward-only курсора: читаем строки по очереди, без перемотки, и сразу обрабатываем.

Главная цель `fetchSize` — баланс между числом round-trip’ов к базе и объёмом данных в памяти. Слишком маленький `fetchSize` (например, 1) приведёт к огромному числу сетевых запросов и заметному оверхеду на latency. Слишком большой (например, 10000) снова даст всплеск памяти и GC, особенно если каждая строка — это сложный граф сущностей. На практике для OLTP-систем часто выбирают размер в диапазоне 100–1000 строк, а дальше уже крутят под конкретный workload и характеристики сети/СУБД.

Наряду с `fetchSize` важно также выставлять forward-only режим курсора. В JDBC это делается через константы `ResultSet.TYPE_FORWARD_ONLY` и `ResultSet.CONCUR_READ_ONLY` при создании `PreparedStatement`. Hibernate под капотом чаще всего и так создаёт forward-only курсоры, но если вы используете чистый JDBC, имеет смысл задать это явно. Это позволяет драйверу оптимизировать использование памяти и не держать весь набор строк в клиентском буфере.

С точки зрения памяти стриминг через курсор спасает только в том случае, если вы не накапливаете сущности/DTO в коллекциях. Любая попытка «сначала собрать всё, потом обработать» моментально нивелирует пользу `fetchSize`. Поэтому классический паттерн работы с курсором — цикл `while (rs.next()) { processRow(); }`, где `processRow` либо пишет данные в файл, либо сразу отправляет в другой сервис, либо агрегирует в небольшие структуры, а не в гигантские списки.

Hibernate-API `ScrollableResults` — ещё одна реализация курсорной модели. Вы получаете объект, который позволяет двигаться по результатам построчно через `scroll.next()` и доставать текущую сущность. Это удобно, если вы хотите использовать JPA-мэппинг, но при этом управлять чисткой `PersistenceContext`: после каждого шага можно делать `session.evict(entity)` или периодически звать `clear()`, чтобы контекст не разрастался. Такой подход часто используют в batch-джобах, где нужно пройтись по всей таблице и по каждой строке выполнить некоторую логику.

Однако `ScrollableResults` — это уже чистый Hibernate API, а не JPA, и требует явного `unwrap(Session.class)` и более осторожной работы. Кроме того, на Web-уровне он вряд ли пригодится; его естественная среда — фоновые джобы, Spring Batch, миграции и прочие отчётные сценарии. В обычных сервисных методах при необходимости стриминга лучше использовать `Stream<T>` или чистый JDBC через `JdbcTemplate`, чтобы не смешивать слишком много абстракций в одном месте.

Нельзя забывать и о том, что курсорные запросы держат транзакцию и соединение в открытом состоянии на всём протяжении обработки. Если вы будете стримить десятки миллионов строк и при этом держать одну длинную транзакцию, вы рискуете заблокировать ресурсы БД, мешать другим операциям и упираться в таймауты. В таких случаях лучше разбивать работу на несколько итераций, каждая из которых обрабатывает разумный диапазон ключей или страниц, и открывать/закрывать транзакцию на каждую порцию.

В итоге `fetchSize` и курсоры — это инструмент для аккуратного балансирования между latency, пропускной способностью и потреблением памяти. Они хорошо сочетаются с read-only транзакциями, где вы просто читаете много данных, но требуют осторожного обращения с транзакциями, пулом соединений и `PersistenceContext`. «Включить и забыть» тут не получится — придётся измерять, настраивать и иногда отказываться от курсорного стриминга в пользу более простых batch-подходов.

Ниже пример, который показывает использование `fetchSize` и forward-only курсора через Hibernate `Session` и напрямую через `JdbcTemplate`. В обоих случаях мы читаем таблицу `orders` большими объёмами и по мере чтения сразу же агрегируем данные, не накапливая их целиком в памяти.

```java
package com.example.jpa.cursor;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.Session;
import org.hibernate.query.Query;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.sql.ResultSet;
import java.sql.SQLException;

@Service
public class OrderCursorService {

    @PersistenceContext
    private EntityManager entityManager;

    private final JdbcTemplate jdbcTemplate;

    public OrderCursorService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Transactional(readOnly = true)
    public long sumAllOrderAmountsJpa() {
        Session session = entityManager.unwrap(Session.class);

        Query<Order> query = session.createQuery(
                "select o from Order o",
                Order.class
        );

        query.setFetchSize(500);

        long sum = 0L;

        try (var stream = query.stream()) {
            for (Order order : (Iterable<Order>) stream::iterator) {
                sum += order.getAmount();
                session.detach(order);
            }
        }

        return sum;
    }

    @Transactional(readOnly = true)
    public long sumAllOrderAmountsJdbc() {
        return jdbcTemplate.query(
                con -> {
                    var ps = con.prepareStatement(
                            "select amount from orders",
                            ResultSet.TYPE_FORWARD_ONLY,
                            ResultSet.CONCUR_READ_ONLY
                    );
                    ps.setFetchSize(500);
                    return ps;
                },
                (ResultSet rs) -> {
                    long sum = 0L;
                    while (rs.next()) {
                        sum += rs.getLong("amount");
                    }
                    return sum;
                }
        );
    }
}
```

```kotlin
package com.example.jpa.cursor

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.hibernate.Session
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderCursorService(

    @PersistenceContext
    private val entityManager: EntityManager,

    private val jdbcTemplate: JdbcTemplate
) {

    @Transactional(readOnly = true)
    fun sumAllOrderAmountsJpa(): Long {
        val session = entityManager.unwrap(Session::class.java)

        val query = session.createQuery(
            "select o from Order o",
            Order::class.java
        )
        query.fetchSize = 500

        var sum = 0L

        query.stream().use { stream ->
            stream.forEach { order ->
                sum += order.amount
                session.detach(order)
            }
        }

        return sum
    }

    @Transactional(readOnly = true)
    fun sumAllOrderAmountsJdbc(): Long {
        return jdbcTemplate.query(
            { con ->
                val ps = con.prepareStatement(
                    "select amount from orders",
                    java.sql.ResultSet.TYPE_FORWARD_ONLY,
                    java.sql.ResultSet.CONCUR_READ_ONLY
                )
                ps.fetchSize = 500
                ps
            },
            { rs ->
                var sum = 0L
                while (rs.next()) {
                    sum += rs.getLong("amount")
                }
                sum
            }
        )
    }
}
```

В этих примерах JPA-вариант использует `fetchSize` и detach для предотвращения раздувания контекста, а JDBC-вариант показывает классический forward-only курсор с `fetchSize`. Оба демонстрируют идею: результат не накапливается, а обрабатывается по мере чтения.

---

## Read-only транзакции + хинты драйвера; выгрузка в файлы/каналы без материализации всего списка

Большие выборки почти всегда живут в сценарии «прочитать и куда-то выгрузить»: сформировать CSV/Excel, отдать файл во внешнюю систему, отправить каждую запись в очередь, сделать снапшот для аналитики. Общий принцип здесь простой: чем меньше промежуточных структур вы создаёте, тем лучше. Оптимальный путь — читать строку, сразу записывать её в целевой канал (файл/стрим/сокет) и забывать. Тогда и нагрузка на память, и время жизни объектов минимальны.

Read-only транзакции в этом контексте работают как дополнительный полезный сигнал. Когда вы помечаете метод `@Transactional(readOnly = true)`, Spring и Hibernate могут оптимизировать поведение: поменять `FlushMode` на `MANUAL` или `COMMIT`, передать драйверу хинт о том, что операция только чтение (для некоторых СУБД), не поднимать лишние блокировки. Для больших выгрузок это снижает риски конкуренции за ресурсы с обычными write-транзакциями и избавляет от ненужных flush.

Хинты драйвера и СУБД — ещё один уровень оптимизации. Многие СУБД и JDBC-драйверы поддерживают флаги вроде `setReadOnly(true)` на `Connection`, которые позволяют базе выбирать более агрессивные планы или оптимизировать блокировки. В Spring `@Transactional(readOnly = true)` обычно автоматически ставит read-only флаг на соединение. Дополнительно можно настроить хинты для конкретного запроса: например, в Postgres — объявить запрос как `SELECT ...` с определённым планом, или в Oracle — использовать `/*+ */`-подсказки, если это действительно нужно.

При выгрузке в файлы особенно важно помнить про потоковую модель I/O. Вместо того чтобы сначала собрать все записи в `List` и потом писать его `ObjectMapper`’ом в файл, вы можете использовать стриминг JSON/CSV: по мере чтения записи из БД сериализовывать её и писать в `BufferedWriter` или `OutputStream`. Многие JSON-библиотеки (Jackson) поддерживают `JsonGenerator`, позволяющий писать массив объектов по одному, не держа весь JSON в памяти. Потоковая модель БД прекрасно сочетается с потоковой моделью сериализации.

То же самое относится к интеграции с внешними системами. Если вам нужно отправить большой объём данных в другую систему, вместо того чтобы сначала сформировать гигантскую коллекцию и затем разом отправить, вы можете работать батчами или по одной записи: прочитал строку — отправил в очередь / HTTP — перешёл к следующей. Это уменьшает не только потребление памяти, но и латентность: первые данные начинают потребляться получателем задолго до завершения всей выгрузки.

Ещё одна важная деталь — управление размером транзакции. Если выгрузка действительно огромная и занимает десятки минут, держать одну транзакцию на всё время может быть плохой идеей: вы удерживаете ресурсы СУБД, рискуете упереться в таймауты и создаёте условия для блокировок. В таких случаях разумнее разбить выгрузку на части: например, идти по первичному ключу или времени создания батчами по 10–50 тысяч записей, открывать транзакцию на каждый батч, выгружать его и закрывать. При этом внешний файл или канал может быть общим, а логика просто «дописываться» кусками.

Утечки ресурсов при потоковой выгрузке — классическая проблема. Если вы забыли закрыть `ResultSet`, `PreparedStatement`, `Stream`, `Writer` или `OutputStream`, то при многократном запуске джобы или API вы будете терять дескрипторы и соединения до тех пор, пока не упадёте. Поэтому в коде, который занимается выгрузкой, должны быть аккуратные try-with-resources или `use {}`-блоки для всех элементов цепочки: и для БД, и для файлов, и для сетевых стримов.

С точки зрения структуры приложения полезно выносить логику «и БД, и файл» в отдельный сервис/джобу, а контроллеру оставлять только запуск и, возможно, отдачу уже готового файла. Контроллер не должен сам итерироваться по стриму из JPA — это усложняет жизненный цикл транзакций и делает код менее предсказуемым. Гораздо проще и надёжнее, когда у вас есть отдельный компонент «exporter», который явно знает, что он делает долгую read-only операцию и управляет всем ресурсным циклом.

Ниже пример, который показывает выгрузку большого набора заказов в CSV-файл с использованием Spring Data JPA `Stream<Order>` и потоковой записи через `BufferedWriter`. Мы открываем read-only транзакцию, создаём временный файл, по мере чтения записей пишем строки CSV и аккуратно закрываем и стрим, и writer. В Kotlin это делается через `use {}`, в Java — через вложенные try-with-resources.

```java
package com.example.jpa.export;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.BufferedWriter;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.stream.Stream;

@Entity
@Table(name = "orders")
class Order {

    @Id
    @GeneratedValue
    private Long id;

    private String customerEmail;

    private Long amount;

    // getters/setters
}

interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("select o from Order o order by o.id")
    Stream<Order> streamAll();
}

@Service
public class OrderExportService {

    private final OrderRepository orderRepository;

    public OrderExportService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional(readOnly = true)
    public Path exportAllToCsv() throws IOException {
        Path tempFile = Files.createTempFile("orders-", ".csv");

        try (Stream<Order> stream = orderRepository.streamAll();
             BufferedWriter writer = Files.newBufferedWriter(tempFile)) {

            writer.write("id,customer_email,amount");
            writer.newLine();

            stream.forEach(order -> {
                try {
                    writer.write(order.getId()
                            + "," + order.getCustomerEmail()
                            + "," + order.getAmount());
                    writer.newLine();
                } catch (IOException e) {
                    throw new RuntimeException("Failed to write CSV line", e);
                }
            });
        }

        return tempFile;
    }
}
```

```kotlin
package com.example.jpa.export

import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.io.BufferedWriter
import java.io.IOException
import java.nio.file.Files
import java.nio.file.Path
import java.util.stream.Stream

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var customerEmail: String = "",

    var amount: Long = 0
)

interface OrderRepository : JpaRepository<Order, Long> {

    @Query("select o from Order o order by o.id")
    fun streamAll(): Stream<Order>
}

@Service
class OrderExportService(
    private val orderRepository: OrderRepository
) {

    @Transactional(readOnly = true)
    @Throws(IOException::class)
    fun exportAllToCsv(): Path {
        val tempFile = Files.createTempFile("orders-", ".csv")

        orderRepository.streamAll().use { stream ->
            Files.newBufferedWriter(tempFile).use { writer ->
                writer.write("id,customer_email,amount")
                writer.newLine()

                stream.forEach { order ->
                    writer.write(
                        "${order.id},${order.customerEmail},${order.amount}"
                    )
                    writer.newLine()
                }
            }
        }

        return tempFile
    }
}
```

В этом примере соединение с БД, курсор, стрим и файловый writer живут ровно столько, сколько длится метод `exportAllToCsv`. Данные не накапливаются в памяти, а сразу пишутся в файл построчно. Такой шаблон легко адаптируется для выгрузки в HTTP-ответ (через `StreamingResponseBody`), Kafka, S3 и любые другие каналы, где естественен потоковый формат.

# 9. DB-специфика и нестандартные типы

## JSON/JSONB, массивы, диапазоны, `hstore`: `@Converter` vs Hibernate Types; индексы (например, GIN для JSONB)

В реальных проектах вы довольно быстро выходите за рамки «скучных» типов `varchar/int/timestamp`. В PostgreSQL появляются `jsonb`, массивы (`text[]`, `bigint[]`), диапазоны (`tsrange`, `numrange`), `hstore` и прочие удобные расширения. JPA формально про эти типы ничего не знает: её модель — абстрактная, кросс-СУБД. Поэтому вам приходится решать, как именно отображать такие поля в доменную модель, не теряя при этом ни гибкости SQL, ни производительности. Ключевых инструментов два: собственные `@Converter` и сторонняя библиотека Hibernate Types (или современные хибер-типизации через `@JdbcTypeCode`), которая добавляет поддержку Postgres-спецтипов.

Самый простой путь для `jsonb` — хранить его строкой: `@Column(columnDefinition = "jsonb") private String payload;`. Тогда приложение никак не вмешивается в структуру JSON, а всё, что умеет сделать ORM, — записать строку в колонку и прочитать её обратно. Это иногда приемлемо для «чужих» payload’ов (например, вы просто проксируете запросы), но для собственного домена такое решение неудобно: вы теряете типизацию на уровне Java/Kotlin, легко ошибаетесь в структуре JSON и вынуждены руками сериализовать/десериализовать данные в каждом месте использования.

Более аккуратный подход — использовать `@Converter` и маппить JSON-колонку на нормальный value object. Вы описываете класс вроде `AddressDetails` или `ExtraAttributes`, а в конвертере превращаете его в `String` через `ObjectMapper` и обратно. Для Hibernate всё по-прежнему выглядит как `varchar/jsonb`, но в доменной модели вы работаете с полноценным типом, можете накладывать инварианты, валидировать структуру и не размазывать JSON-парсинг по всему коду. Это хороший базовый уровень, если вам не нужно писать сложные JSON-фильтры в SQL.

Пример такой связки: отдельный `@Embeddable` или просто POJO/датакласс под содержимое JSON и `AttributeConverter`, который решает проблему сериализации. Важно, чтобы конвертер был детерминированным и не бросал checked-исключений, иначе интеграция с Hibernate будет болезненной. Чаще всего внутри используют Jackson и аккуратно оборачивают возможные ошибки в `IllegalArgumentException`.

```java
package com.example.jpa.json;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.*;

@Entity
@Table(name = "customer_profile")
public class CustomerProfile {

    @Id
    @GeneratedValue
    private Long id;

    @Column(name = "email", nullable = false, unique = true)
    private String email;

    @Convert(converter = PreferencesJsonConverter.class)
    @Column(name = "preferences", columnDefinition = "jsonb")
    private Preferences preferences;

    // геттеры/сеттеры
}

@Embeddable
class Preferences {

    private boolean marketingAllowed;

    private String theme;

    // геттеры/сеттеры
}

@Converter(autoApply = false)
class PreferencesJsonConverter implements AttributeConverter<Preferences, String> {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Preferences attribute) {
        if (attribute == null) {
            return null;
        }
        try {
            return OBJECT_MAPPER.writeValueAsString(attribute);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("Failed to serialize preferences", e);
        }
    }

    @Override
    public Preferences convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }
        try {
            return OBJECT_MAPPER.readValue(dbData, Preferences.class);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("Failed to deserialize preferences", e);
        }
    }
}
```

```kotlin
package com.example.jpa.json

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue
import jakarta.persistence.AttributeConverter
import jakarta.persistence.Column
import jakarta.persistence.Convert
import jakarta.persistence.Converter
import jakarta.persistence.Embeddable
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.Id
import jakarta.persistence.Table

@Entity
@Table(name = "customer_profile")
class CustomerProfile(

    @Id
    @GeneratedValue
    var id: Long? = null,

    @Column(nullable = false, unique = true)
    var email: String = "",

    @Convert(converter = PreferencesJsonConverter::class)
    @Column(name = "preferences", columnDefinition = "jsonb")
    var preferences: Preferences? = null
)

@Embeddable
data class Preferences(
    var marketingAllowed: Boolean = false,
    var theme: String = "light"
)

@Converter(autoApply = false)
class PreferencesJsonConverter : AttributeConverter<Preferences?, String?> {

    private val mapper = jacksonObjectMapper()

    override fun convertToDatabaseColumn(attribute: Preferences?): String? =
        attribute?.let { mapper.writeValueAsString(it) }

    override fun convertToEntityAttribute(dbData: String?): Preferences? =
        dbData?.let { mapper.readValue<Preferences>(it) }
}
```

Подход с `@Converter` удобен, но имеет важное ограничение: Hibernate не знает, что внутри строки JSON. Для него это просто `VARCHAR/JSONB`. Значит, вы не сможете использовать JSON-поля в критериях, сортировках и join’ах через JPQL. В SQL, конечно, вы можете написать `where preferences->>'theme' = 'dark'`, и Postgres с этим справится, но ORM такой запрос как типобезопасный строить не умеет. Кроме того, если вы захотите использовать GIN-индексы по JSON-полям, вам придётся писать DDL руками и помнить о синтаксисе при каждом запросе.

Когда вам нужно именно «понимающее» отношение к JSON/массивам/диапазонам, на сцену выходит либо Hibernate 6 с его `@JdbcTypeCode(SqlTypes.JSON)`, либо библиотека Hibernate Types (Vlad Mihalcea), которая добавляет готовые типы для Postgres `jsonb`, массивов, диапазонов, `hstore` и т.д. Вместо того чтобы писать свои конвертеры, вы аннотируете поле специальным `@Type`, выбираете нужный тип (`JsonType`, `StringArrayType`, `JsonBinaryType` и т.п.) и получаете поддержку сериализации «из коробки», плюс возможность использовать эти поля в нативных и частично в JPQL-запросах.

Чтобы использовать Hibernate Types, достаточно добавить зависимость и указать правильный dialect. Пример для Gradle Groovy и Kotlin DSL: добавляем библиотеку и потом аннотируем поле `@Type(JsonType.class)` (для Hibernate 6 — вариант с `@JdbcTypeCode`). Этот подход особенно удобен, когда вы активно используете Postgres-спецтипов и не хотите плодить десятки кастомных конвертеров.

```groovy
// build.gradle (Groovy DSL)
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.postgresql:postgresql'
    implementation 'com.vladmihalcea:hibernate-types-60:2.21.1'
}
```

```kotlin
// build.gradle.kts (Kotlin DSL)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.postgresql:postgresql")
    implementation("com.vladmihalcea:hibernate-types-60:2.21.1")
}
```

```java
package com.example.jpa.json;

import com.vladmihalcea.hibernate.type.json.JsonType;
import jakarta.persistence.*;
import org.hibernate.annotations.Type;

import java.util.Map;

@Entity
@Table(name = "audit_log")
public class AuditLogEntry {

    @Id
    @GeneratedValue
    private Long id;

    private String eventType;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> details;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.json

import com.vladmihalcea.hibernate.type.json.JsonType
import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.hibernate.annotations.Type

@Entity
@Table(name = "audit_log")
class AuditLogEntry(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var eventType: String = "",

    @Type(JsonType::class)
    @Column(columnDefinition = "jsonb")
    var details: MutableMap<String, Any>? = null
)
```

Аналогичным образом можно работать с массивами и диапазонами. Например, массив тегов `text[]` можно промапить на `List<String>` через Hibernate Types (`StringArrayType`) или через собственный `@Converter`, который использует `Connection.createArrayOf`. Диапазоны `tsrange`/`numrange` можно отображать на собственный `Range<T>`-класс или использовать готовые типы из библиотеки. Главное помнить, что эти типы — специфичны для Postgres: если вы когда-нибудь захотите сменить СУБД, такие поля станут источником миграционных проблем.

Индексирование JSON/массивов — обязательная часть истории. Без индексов `where details->>'userId' = ?` или `where tags @> ARRAY['vip']` будут медленно сканировать таблицу целиком. Для `jsonb` и массивов в Postgres стандартная практика — GIN-индексы. Например, вы можете сделать индекс `create index on audit_log using gin (details jsonb_path_ops);` или `create index on product using gin (tags);`. Тогда операции `@>`, `?`, `@?` и прочие JSON/array-операторы будут работать значительно быстрее, особенно на больших объёмах.

Пример DDL с GIN-индексами для jsonb и массива может выглядеть так: в одном случае вы индексируете весь JSON, в другом — только конкретное поле через выражение. Второй вариант помогает уменьшить размер индекса и чётко привязать его к конкретному бизнес-запросу. Выносить такие индексы в миграции (Liquibase/Flyway) — обязательно: их нельзя создавать «по ходу» через JPA-аннотации, и контроль версий схемы по-прежнему живёт на уровне SQL.

```sql
-- JSONB: общий GIN-индекс по всему документу
CREATE INDEX IF NOT EXISTS idx_audit_log_details_gin
    ON audit_log
    USING gin (details jsonb_path_ops);

-- JSONB: индекс по конкретному полю userId
CREATE INDEX IF NOT EXISTS idx_audit_log_user_id
    ON audit_log
    USING gin ((details ->> 'userId'));

-- Массив тегов text[]
CREATE INDEX IF NOT EXISTS idx_product_tags_gin
    ON product
    USING gin (tags);
```

Тип `hstore` логически похож на `jsonb`, но ограничен картой строка→строка. Его удобно использовать для компактных динамических атрибутов (например, «параметры товара»), где типы значений не важны и нет вложенных структур. Маппинг через Hibernate Types (`HstoreType`) позволяет отобразить колонку `hstore` на `Map<String, String>` и дальше с ней работать как с обычной картой. Но и здесь действует общий принцип: чем более активно вы используете специфичные типы БД, тем сильнее вы привязаны к ней архитектурно — и это нужно признать как осознанное решение.

В итоге практическая рекомендация такая: если JSON/массив нужен лишь для хранения протокольных данных и редко участвует в запросах — можно обойтись простым `@Converter` и строкой. Если вы активно фильтруете/индексируете такие поля, используете Postgres-операторы, — лучше взять Hibernate Types или `@JdbcTypeCode` и работать через vendor-specific типы. Всегда добавляйте индексы под реальные запросы, документируйте, что это именно Postgres-фича, и не пытайтесь спрятать эту специфику за мнимой «абстрактностью» JPA.

---

## Денежные/точные типы: `BigDecimal` + scale/rounding; контроль сериализации в JSON

Денежные значения и вообще любые «точные» числа — отдельная боль. Использовать `double` или `float` для денег категорически нельзя: двоичная плавающая точка не может точно представить десятичные дроби вроде 0.1, и со временем вы получаете накопление ошибок, «копейки из ниоткуда» и разъезды между суммами в БД и суммами в API. Правильный тип на стороне Java/Kotlin — `BigDecimal`, а на стороне Postgres — `numeric(p, s)` (или, в редких случаях, встроенный `money`, но он менее универсален).

Первый шаг — правильное DDL. Для суммы денег в типичной банковской задаче часто используют `numeric(19, 4)` или похожую пару precision/scale. Precision — это общее количество цифр, scale — количество знаков после запятой. Например, `numeric(19,4)` позволяет хранить числа до ±999 999 999 999 9.9999. Выбирая precision, вы должны понимать максимальные реальные суммы в предметной области; scale зависит от требований: для RUB/EUR часто достаточно 2, но иногда выгоднее хранить 4 знака для внутренних расчётов и в UI показывать 2.

В JPA вы должны явно указать `precision` и `scale` в `@Column`, чтобы Hibernate корректно создавал столбец (если он генерирует схему) и правильно работал с типом. Если вы этого не сделаете, драйвер может создать `numeric` без явных ограничений, и поведение при вставке/обновлении будет менее предсказуемым. Кроме того, аннотация служит документацией: любой, кто читает entity, сразу видит, с какой точностью хранится поле.

```java
package com.example.jpa.money;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "payment")
public class Payment {

    @Id
    @GeneratedValue
    private Long id;

    @Column(name = "amount", precision = 19, scale = 4, nullable = false)
    private BigDecimal amount;

    @Column(name = "currency", length = 3, nullable = false)
    private String currency;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.money

import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.math.BigDecimal

@Entity
@Table(name = "payment")
class Payment(

    @Id
    @GeneratedValue
    var id: Long? = null,

    @Column(name = "amount", precision = 19, scale = 4, nullable = false)
    var amount: BigDecimal = BigDecimal.ZERO,

    @Column(name = "currency", length = 3, nullable = false)
    var currency: String = "RUB"
)
```

Следующий момент — стратегия округления. `BigDecimal` сам по себе не навязывает scale; вы можете получить значения с разным количеством знаков после запятой. Важно выработать соглашение: все суммы внутри системы нормализуем до фиксированного scale (например, 2 или 4 знака) и выбранного `RoundingMode`. Чаще всего используют `RoundingMode.HALF_UP` или `HALF_EVEN`, но выбор зависит от требований регулятора и доменных правил. Критично, чтобы одно и то же действие в разных местах кода округлялось одинаково.

Хорошая практика — инкапсулировать округление в одном месте: либо в value object «Money», либо в сервисе, который отвечает за расчёты. В этом случае вместо того, чтобы по всему коду писать `amount.setScale(2, RoundingMode.HALF_UP)`, вы вызываете один метод, а логику легко менять централизованно. Дополнительно можно на уровне БД повесить `CHECK`-ограничения, например что число знаков после запятой не превышает заданный scale. В Postgres это делается автоматически через тип `numeric(19,4)`, но для более замысловатых случаев можно написать явный `CHECK`.

Тонкий момент — сериализация денег в JSON. Если вы используете Jackson, то по умолчанию `BigDecimal` сериализуется в числовой литерал, что удобно, но может вызывать проблемы на стороне JavaScript-клиентов: JS по факту оперирует `Number` (двойная точность), и очень большие числа/очень точные дроби будут искажаться. В ряде систем на клиент выводят суммы как строки (`"123.45"`) и уже в UI приводят их к нужному виду. В этом случае Jackson можно настроить через модули или кастомный сериализатор.

К тому же при парсинге JSON Jackson может по умолчанию превращать числа в `Double`, если вы не явно указали тип `BigDecimal` в DTO. Хорошая практика — в DTO/record’ах иметь поля типа `BigDecimal` и включать опцию `DeserializationFeature.USE_BIG_DECIMAL_FOR_FLOATS`, чтобы все числа, которые должны быть точными, приходили без промежуточного `Double`. Это уменьшает риск накопления ошибок в цепочке «клиент → API → БД».

Немаловажен и вопрос валюты. Самый простой вариант — хранить сумму и трёхбуквенный код ISO (RUB, USD, EUR). Для более строгих доменов используют специализированные библиотеки (Joda-Money и др.), которые инкапсулируют не только число, но и операции с учётом currency. В JPA это можно маппить либо как `@Embeddable` (колонки `amount` и `currency`), либо через `@Converter`, который превращает объект Money в парочку примитивов. Главное — не разрывать сумму и валюту, храня их в разных местах, иначе вы легко получите несогласованные данные.

Ещё один слой защиты — ограничения на уровне БД: отрицательные суммы там, где их быть не должно; минимальные/максимальные лимиты; запрет на «лишние» дробные части. В Postgres это делается через простые `check (amount >= 0)` или более сложные выражения. Они не заменяют бизнес-валидацию в коде, но создают дополнительный барьер, который не позволит случайному багу записать явно неправильные деньги.

Наконец, важно помнить про агрегацию и отчёты. Если вы суммируете миллионы строк с `numeric`, производительность может страдать. Иногда в таких сценариях создают отдельные агрегатные таблицы, где хранят предварительно посчитанные суммы по интервалам, а затем обновляют их батчами или через триггеры. Но даже в этом случае ядром хранения остаётся `numeric + BigDecimal`: выбор «приблизительных» типов для денег почти всегда оборачивается большими проблемами позже, чем экономией сегодня.

---

## Частичные/функциональные индексы, `UNIQUE` под soft-delete, «виртуальные» столбцы/представления (`@Immutable`, `@Subselect`)

Даже если ваша модель на уровне JPA выглядит аккуратно, настоящая мощь производительности обычно прячется в индексах. Обычные B-tree индексы по колонкам, которые вы фильтруете в `where`, — это база. Но в реальном мире вы начинаете использовать частичные индексы (только для подмножества строк), функциональные индексы (по выражениям), специальные `UNIQUE`-индексы для soft-delete и даже «виртуальные» сущности поверх представлений. Всё это JPA не генерирует сама, но прекрасно с этим работает, если вы грамотно напишете миграции.

Частичный индекс — это индекс, который покрывает только строки, удовлетворяющие условию, например `where deleted_at is null`. Это удобно, когда у вас есть soft-delete: в таблице куча «мёртвых» записей, но 99% запросов работают только с активными. Вместо одного большого индекса по всей таблице вы создаёте компактный индекс по активной части. JPA про это ничего не знает — вы по-прежнему пишете `where deletedAt is null` в JPQL — но Postgres использует нужный индекс и экономит и время, и диск.

Пример миграции для частичного индекса под активных клиентов: вы создаёте обычный столбец `deleted_at` для soft-delete, а затем определяете индекс `create index ... where deleted_at is null;`. При выборках `where deleted_at is null and email = ?` СУБД пойдёт именно по этому индексу. В Liquibase/Flyway такой DDL складывают в версионируемые скрипты; JPA-аннотации вроде `@Index` для такого кейса не подходят, потому что не умеют выражать `where`-условия.

```sql
-- soft-delete: только живые клиенты
CREATE INDEX IF NOT EXISTS idx_customer_email_active
    ON customer (email)
    WHERE deleted_at IS NULL;
```

Функциональные индексы позволяют индексировать выражения: `lower(email)`, `coalesce(phone, '')`, `substring(code, 1, 3)` и т.д. Это особенно полезно для case-insensitive поисков: вместо того чтобы каждый раз делать `where lower(email) = lower(:email)`, вы можете в БД создать индекс `create index ... on customer (lower(email));` и в запросах использовать ровно такое же выражение. Здесь важен инвариант: выражение в SQL-запросе должно совпадать с выражением в индексе, иначе оптимизатор не сможет его использовать.

Soft-delete сам по себе часто требует нестандартного `UNIQUE`. Если вы просто добавите колонку `deleted_at` и оставите старый уникальный индекс по `email`, то удаление (установка `deleted_at`) не освободит место для новой записи с тем же `email`: БД по-прежнему видит две строки с одинаковым значением в уникальном индексе, только одна из них «помечена». Правильный паттерн — создать частичный уникальный индекс только по живым записям: `create unique index ... on customer(email) where deleted_at is null;`. Тогда вы сможете создавать новые записи после soft-delete старых, не ломая уникальность среди активных.

Пример миграции для такого уникального индекса: вы удаляете старый `unique constraint`, если он был, и создаёте новый partial unique index. Затем JPA продолжает думать, что `email` просто `unique`, но на уровне схемы вы фактически говорите: «у активного клиента email уникален, у удалённых может дублироваться». Это отлично сочетается с глобальными фильтрами (`@Where` или Hibernate Filter), которые скрывают soft-deleted записи от большинства запросов.

```sql
-- уникальность e-mail только среди не удалённых
CREATE UNIQUE INDEX IF NOT EXISTS uk_customer_email_active
    ON customer(email)
    WHERE deleted_at IS NULL;
```

«Виртуальные» столбцы и представления — ещё один мощный инструмент. Иногда у вас есть сложный SQL (join нескольких таблиц, агрегаты, CASE-выражения), который вы хотите использовать в разных местах. Вместо того чтобы лепить его в каждый JPQL/SQL, можно создать представление (`CREATE VIEW`), а в Hibernate промаппить его на read-only сущность через `@Subselect`. Такая сущность не соответствует реальной таблице, но выглядит для ORM как обычный entity, который можно читать (но не писать). Это удобно для отчётных форм, витрин, аналитических таблиц.

Hibernate для этого предоставляет аннотации `@Subselect` и `@Immutable`. Первая говорит: «эта сущность читается из такого-то SQL», вторая — «её нельзя изменять через ORM». Дополнительно можно указать `@Synchronize`, чтобы Hibernate понимал, какие реальные таблицы влияют на это представление. Тогда при flush и некоторых видах кэша он сможет лучше понимать, когда данные потенциально устарели. Важно дать сущности натуральный первичный ключ (либо взять из представления, либо сгенерировать), иначе ORM не сможет корректно трекать экземпляры.

```java
package com.example.jpa.view;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import org.hibernate.annotations.Immutable;
import org.hibernate.annotations.Subselect;
import org.hibernate.annotations.Synchronize;

@Entity
@Immutable
@Subselect("""
    select
        o.id as id,
        c.email as customer_email,
        o.amount as amount,
        o.created_at as created_at
    from orders o
    join customer c on c.id = o.customer_id
    where o.status = 'COMPLETED'
    """)
@Synchronize({"orders", "customer"})
@Table(name = "order_report_view")
public class OrderReportView {

    @Id
    private Long id;

    private String customerEmail;

    private Long amount;

    private java.time.Instant createdAt;

    // только геттеры
}
```

```kotlin
package com.example.jpa.view

import jakarta.persistence.Entity
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.hibernate.annotations.Immutable
import org.hibernate.annotations.Subselect
import org.hibernate.annotations.Synchronize
import java.time.Instant

@Entity
@Immutable
@Subselect(
    """
    select
        o.id as id,
        c.email as customer_email,
        o.amount as amount,
        o.created_at as created_at
    from orders o
    join customer c on c.id = o.customer_id
    where o.status = 'COMPLETED'
    """
)
@Synchronize("orders", "customer")
@Table(name = "order_report_view")
class OrderReportView(

    @Id
    val id: Long? = null,

    val customerEmail: String? = null,

    val amount: Long? = null,

    val createdAt: Instant? = null
)
```

Такие сущности отлично подходят под read-only сценарии: отчёты, панели мониторинга, выгрузки в BI. Вы можете использовать их в Spring Data репозиториях, делать пагинацию, сортировку и даже простые фильтры, при этом всю реальную «магическую» SQL спрятать в представлении. Единственное — нужно дисциплинированно поддерживать миграции: любые изменения в схемах таблиц, от которых зависит view, должны сопровождаться изменением определения представления и, возможно, сущности.

Функциональные индексы и представления иногда комбинируют. Например, если у вас есть представление со сложным выражением `coalesce(phone, email)` для колонки «контакт», к которому часто обращаются, можно построить индекс по этому выражению в основной таблице и использовать его в view. Но это уже довольно продвинутая оптимизация — важно не увлечься и не превратить схему в лабиринт взаимозависимых представлений и индексов, который никто не может поддерживать.

Итог: JPA сама по себе ничего не знает про частичные и функциональные индексы, soft-delete-инварианты и представления. Это зона ответственности схемы БД и миграций. Но при правильном дизайне вы легко совмещаете «чистую» доменную модель на уровне ORM с «умной» схемой Postgres: обеспечиваете уникальность только для нужных строк, ускоряете самые частые запросы и строите отчётные сущности поверх представлений. Главное — держать это под контролем: все такие решения должны быть задокументированы, покрыты тестами и понятны команде, а не быть «магией, которая где-то в миграциях».


# 10. Аудит и «мягкое удаление»

Аудит и мягкое удаление — это про то, чтобы не терять данные и уметь ответить на вопрос «кто, когда и что сделал», а также «почему в таблице нет этой записи». Простое `DELETE FROM table` удобно с точки зрения кода, но в продакшене почти всегда оказывается слишком радикальным: бизнесу нужны следы операций, регуляторы требуют истории, а разработчикам нужны данные для расследований инцидентов и отладки. Поэтому вокруг Spring Data JPA и Hibernate сложился набор типовых практик: аудиторные поля, soft-delete и схемы хранения истории изменений.

Если смотреть на задачу сверху, то есть несколько уровней. На самом базовом — «кто создал / когда создал / кто изменил / когда изменил». Это решается стандартным механизмом Spring Data Auditing: аннотации `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy` и слушатель `AuditingEntityListener`. Следующий уровень — soft-delete: вместо удаления строки вы ставите флаг или дату удаления и скрываете такие записи фильтрами. И третий уровень — полноценная история изменений: версии записей или отдельные аудит-таблицы, куда для каждой операции пишется старое/новое состояние.

Важный момент: эти уровни не взаимоисключающие. Почти всегда вы комбинируете все три подхода. Сущности имеют аудиторные поля, сами записи мягко удаляются, а операции поверх критичных таблиц пишутся в отдельный лог или версионируются. В итоге архитектура persistence-слоя становится чуть сложнее, но взамен вы получаете объяснимое поведение: всегда можно понять, почему данные выглядят так, а не иначе.

Нельзя забывать и про цену вопроса. Аудит и soft-delete увеличивают размер таблиц, утяжеляют индексы и влияют на планы запросов. История изменений и отдельные аудит-таблицы удваивают или утраивают объём хранимых данных. Поэтому эти механизмы нужно проектировать сознательно: чётко понимать, что именно вам нужно хранить, сколько времени и с какой точностью. «Всё и навсегда» почти всегда приводит к проблемам с производительностью и обслуживанием.

Ниже разберём три аспекта: как реализовать базовый аудит через `AuditingEntityListener`, как корректно делать soft-delete в связке с JPA/Hibernate, и какие есть варианты построения истории изменений — от версионирования записей до отдельного аудит-лога.

---

## Auditing: `@CreatedDate/@LastModifiedDate`, `AuditingEntityListener`, пользователи/тенанты

Spring Data JPA предлагает готовый механизм для автоматического заполнения полей «кто и когда создал / изменил запись». Это Spring Data Auditing. Он работает довольно просто: вы включаете `@EnableJpaAuditing`, регистрируете бин `AuditorAware<T>`, который умеет сказать, кто сейчас «текущий пользователь», и помечаете поля сущностей аннотациями `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy`. Дальше, при `persist`/`merge`, слушатель `AuditingEntityListener` сам проставит значения.

Первое, о чём нужно подумать, — формат времени. Если вы храните даты/время, почти всегда имеет смысл использовать UTC и тип `Instant` или `OffsetDateTime`. Локальное время (`LocalDateTime`) удобно в UI, но в БД и домене лучше хранить однозначные значения без привязки к смене часового пояса и переходам на летнее время. Spring Data Auditing нормально работает с `Instant`, нужно лишь убедиться, что вы настроили конвертеры JPA для маппинга в `timestamp with time zone` или аналогичный тип.

Второй вопрос — откуда брать пользователя. В типичном Spring Security-приложении вы вытаскиваете `Authentication` из `SecurityContext` и берёте имя пользователя или его id. Но нужно помнить, что аудит должен работать не только в HTTP-запросах, но и в фоновых задачах, batch-джобах, миграциях. Поэтому интерфейс `AuditorAware` удобно делать универсальным: в веб-контексте он смотрит в `SecurityContext`, в фоновых задачах — может отдавать системного пользователя вроде `"system"` или `null`, а в тестах — фиксированное значение или заглушку.

Ещё один уровень усложнения — мультитенантность. Часто аудиторная информация включает не только пользователя, но и tenant-id: условный идентификатор клиента, на чьи данные сейчас идёт операция. Вы можете добавить отдельное поле `tenantId` в базовый класс сущностей и заполнять его в том же `AuditorAware` или в отдельном фильтре/слушателе. Важно, чтобы источник tenant-id был чётко определён (например, заголовок HTTP, параметр подключения, переменная контекста) и одинаково работал для всех типов загрузки (HTTP, очереди, фоновые задачи).

С точки зрения схемы данных удобно завести базовый класс для всех «аудируемых» сущностей, который будет содержать поля аудита. В Java это может быть `@MappedSuperclass` с аннотациями `@CreatedDate` и т.д., а в Kotlin — класс с открытыми свойствами и теми же аннотациями. Это избавляет от дублирования полей и позволяет централизованно управлять форматами и типами. Важно не забыть повесить `@EntityListeners(AuditingEntityListener.class)` на этот базовый класс, чтобы слушатель отрабатывал для всех наследников.

Нужно понимать и ограничения. Аудит отрабатывает только при операциях через JPA: если вы напрямую обновляете таблицу через чистый JDBC или в миграциях, `AuditingEntityListener` не включится. Для таких случаев логика заполнения дат/пользователя должна быть либо продублирована на уровне триггеров, либо ваши процессы должны идти через сервисный слой, который работает через ORM. Полностью закрыть все пути модификации только одним механизмом, как правило, не удаётся.

Ещё один момент — тестируемость. В тестах без безопасности `AuditorAware` должен возвращать предсказуемое значение, иначе вы получите нестабильные данные в БД. Можно сделать простой `TestAuditorAware`, где текущий пользователь задаётся через ThreadLocal в самих тестах. Либо в unit-тестах не полагаться на значения аудиторных полей, а проверять только то, что они не `null`, если это важно.

Отдельно подумайте про сериализацию в API. Аудиторные поля полезно выдавать наружу в административных и служебных API, но в обычных публичных ответах они могут быть лишними. Часто делают отдельные DTO для админских экранов, где показывают «кто создал / когда изменил», а в публичных DTO этих полей нет. Это упрощает backward compatibility: меняется только внутренняя структура, а внешние контракты остаются стабильными.

Ниже пример минимальной конфигурации Spring Data Auditing и базового класса аудируемой сущности для Java и Kotlin. В конфигурации мы включаем аудит и регистрируем `AuditorAware<String>`, который в проде забирает пользователя из `SecurityContext`, а в отсутствие аутентификации отдаёт `"system"`.

```java
package com.example.jpa.audit;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Optional;

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaAuditConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null || !auth.isAuthenticated()) {
                return Optional.of("system");
            }
            return Optional.ofNullable(auth.getName());
        };
    }
}
```

```java
package com.example.jpa.audit;

import jakarta.persistence.Column;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.annotation.CreatedBy;
import org.springframework.data.annotation.LastModifiedBy;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", length = 100, updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by", length = 100)
    private String updatedBy;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.audit

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.domain.AuditorAware
import org.springframework.data.jpa.repository.config.EnableJpaAuditing
import org.springframework.security.core.context.SecurityContextHolder
import java.util.Optional

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
class JpaAuditConfig {

    @Bean
    fun auditorProvider(): AuditorAware<String> = AuditorAware {
        val auth = SecurityContextHolder.getContext().authentication
        if (auth == null || !auth.isAuthenticated) {
            Optional.of("system")
        } else {
            Optional.ofNullable(auth.name)
        }
    }
}
```

```kotlin
package com.example.jpa.audit

import jakarta.persistence.Column
import jakarta.persistence.EntityListeners
import jakarta.persistence.MappedSuperclass
import org.springframework.data.annotation.CreatedBy
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedBy
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import java.time.Instant

@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class AuditableEntity(

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    var createdAt: Instant? = null,

    @LastModifiedDate
    @Column(name = "updated_at")
    var updatedAt: Instant? = null,

    @CreatedBy
    @Column(name = "created_by", length = 100, updatable = false)
    var createdBy: String? = null,

    @LastModifiedBy
    @Column(name = "updated_by", length = 100)
    var updatedBy: String? = null
)
```

Любая сущность, которая наследуется от `AuditableEntity`, автоматически получает заполнение полей при сохранении и обновлении. Это простая, но очень полезная основа для аудита.

---

## Soft-delete: флаг + глобальные фильтры (`@Where/@SQLDelete` или фильтры Hibernate), последствия для уникальных ключей/джойнов

Мягкое удаление (soft-delete) — это практика, при которой запись логически удаляется, но физически остаётся в таблице. Обычно это реализуется через флаг `deleted` или колонку `deleted_at` с датой удаления. Основной мотив — не терять данные окончательно: вы можете восстановить запись, проверить историю, избежать проблем с внешними ключами, когда другие сущности ссылаются на «удалённую» запись. Физическое удаление при этом может делаться отдельной джобой по старым данным.

Наивный вариант soft-delete — просто добавить колонку `deleted` и везде в запросах писать `where deleted = false`. Это работает до тех пор, пока все разработчики дисциплинированно помнят о флагe. На практике всегда найдётся запрос, который забудет про `deleted`, и в UI вы увидите «призраков», а в отчётах — странные дубли. Поэтому в мире Hibernate принято прятать логику soft-delete в слой ORM: использовать глобальные фильтры `@Where` и `@SQLDelete` или хибер-фильтры.

Аннотация `@SQLDelete` позволяет переопределить SQL, который Hibernate выполняет при удалении сущности. Вместо `delete from customer where id = ?` вы можете сказать «обнови флаг»: `update customer set deleted = true, deleted_at = now() where id = ?`. Аннотация `@Where` добавляет условие ко всем запросам по этой сущности: Hibernate автоматически подставит `where deleted = false` к любому `select`. В итоге обычные операции `repository.delete(entity)` и `findAll()` начинают работать с учётом soft-delete без дополнительного кода в репозиториях.

Надо понимать, что `@Where` — это глобальный фильтр. Он применяется и к простым запросам, и к связям. Например, если у вас `@OneToMany` коллекция и вы повесите `@Where` на дочернюю сущность, то в коллекции не будет soft-deleted элементов. Это удобно в большинстве сценариев, но иногда вам нужно, наоборот, увидеть всё, включая удалённые. Тогда придётся либо писать нативный запрос, либо завести отдельный репозиторий/метод без `@Where` (например, через view или через `Session` с отключением фильтра).

С soft-delete тесно связаны уникальные ключи. Если вы просто добавите колонку `deleted` в таблицу, где есть `unique (email)`, то удаление не освободит возможность создать новую строку с тем же `email`: старая запись всё ещё нарушает уникальность. Правильный паттерн — использовать частичный уникальный индекс только для живых строк: `unique (email) where deleted = false`. Тогда soft-delete логически убирает запись из множества, по которому проверяется уникальность, и вы спокойно создаёте новую запись с тем же ключом.

Выбор типа флага тоже важен. Фиксированное `boolean deleted` удобно для логики, но не хранит информации о времени удаления. Обычно лучше использовать `deleted_at timestamp` и считать «живыми» строки, где `deleted_at is null`. Это позволяет легко найти, что удалилось за период, построить отчёты, реализовать политику «окно восстановления» (например, можно восстанавливать только в течение 30 дней после удаления). В коде JPA это отображается как `Instant deletedAt` и условие `@Where(clause = "deleted_at is null")`.

Есть и более гибкий механизм — Hibernate filters (`@FilterDef` и `@Filter`). В отличие от `@Where`, фильтры можно включать/выключать на уровне `Session`/`EntityManager`. Это полезно, когда у вас есть сценарии «обычно скрывать удалённые записи, но в админском разделе иногда показывать всё». Однако фильтры требуют чуть более сложной интеграции: нужно помнить, где их включать, и не забывать про них в новых местах. Для базового soft-delete `@Where` + `@SQLDelete` обычно достаточно.

Не забывайте про каскады. Если вы используете `orphanRemoval = true` или `CascadeType.REMOVE` на связях, Hibernate при удалении родителя попытается удалять и дочерние сущности. При soft-delete это может привести к каскадному обновлению большого числа строк (установка флага на все дочерние записи), что иногда нормально, а иногда нет. В сложных моделях лучше явно определять, какие сущности подлежат soft-delete, а какие всё-таки удаляются физически или остаются жить.

Ниже простой пример сущности `Customer` с soft-delete через `deleted_at`, глобальный фильтр `@Where` и переопределённый `@SQLDelete`. В миграции мы добавляем колонку `deleted_at` и частичный уникальный индекс по email. Java и Kotlin варианты эквивалентны.

```sql
ALTER TABLE customer
    ADD COLUMN IF NOT EXISTS deleted_at timestamptz;

CREATE UNIQUE INDEX IF NOT EXISTS uk_customer_email_active
    ON customer (email)
    WHERE deleted_at IS NULL;
```

```java
package com.example.jpa.softdelete;

import jakarta.persistence.*;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;

import java.time.Instant;

@Entity
@Table(name = "customer")
@SQLDelete(sql = "UPDATE customer SET deleted_at = now() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    private String fullName;

    @Column(name = "deleted_at")
    private Instant deletedAt;

    // геттеры/сеттеры
}
```

```kotlin
package com.example.jpa.softdelete

import jakarta.persistence.*
import org.hibernate.annotations.SQLDelete
import org.hibernate.annotations.Where
import java.time.Instant

@Entity
@Table(name = "customer")
@SQLDelete(sql = "UPDATE customer SET deleted_at = now() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    @Column(nullable = false, unique = true)
    var email: String = "",

    var fullName: String? = null,

    @Column(name = "deleted_at")
    var deletedAt: Instant? = null
)
```

С таким подходом все обычные запросы (`findById`, `findAll`, репозиторные методы) будут работать только с живыми записями. Удаление через `delete` поставит флаг, но не тронет строку физически, и уникальный email освободится. Для периодической физической чистки можно завести отдельный SQL-скрипт или джобу, которая удаляет «старые» удалённые записи, например `where deleted_at < now() - interval '90 days'`.

---

## История изменений: версионирование записей vs отдельные audit-таблицы; компромисс сложность/польза

Аудит «кто/когда создал/изменил» и soft-delete решают только часть задач. Часто бизнесу нужно знать, как именно менялись данные во времени: какую ставку кредита установили вначале, как она менялась, кто правил адрес клиента, какие статусы проходила заявка. Для этого нужен полноценный журнал изменений. В JPA/Hibernate есть два базовых подхода: версионирование записей в самой таблице (multi-version records) и отдельные аудит-таблицы, куда пишутся «снимки» сущности при каждом изменении.

Версионирование записей в основной таблице — это когда вместо одной строки на сущность вы храните несколько, по одной на каждую версию. Часто это реализуется через поля `valid_from` / `valid_to` или через номер версии и флаг «активный». При чтении «текущего» состояния вы фильтруете по `valid_to is null` или по максимальному `version`. Такой подход популярен в финансовых системах, где важно видеть полную историю параметров договора или тарифного плана. Преимущество — вся история в одной таблице, удобно делать аналитические запросы. Недостаток — «текущая» выборка должна всегда помнить о критерии активной версии, а таблица растёт быстрее.

Отдельные аудит-таблицы — другой вариант. Вы создаёте таблицу `customer_audit` или используете Hibernate Envers (`_AUD`-таблицы), где на каждую операцию (INSERT, UPDATE, DELETE/soft-delete) сохраняется отдельная запись со старым/новым состоянием. Основная таблица остаётся компактной и содержит только актуальное состояние, а аудит-таблица становится «историческим журналом». Это удобно, когда оперативные запросы не должны страдать от увеличения объёма, а история нужна главным образом для расследований, compliance и отчётности.

Готовое решение для аудит-таблиц в мире Hibernate — Envers. Вы ставите зависимость, аннотируете сущность `@Audited`, и Hibernate начинает автоматически писать изменения в отдельную таблицу `<entity>_AUD` с колонками `REV`, `REVTYPE`, полями сущности и метаданными. Это сильно экономит время: не нужно придумывать свой формат аудит-логов, писать триггеры, руками логировать каждое изменение. Но цена — зависимость от Envers, дополнительная сложность схемы и необходимость учитывать аудит-таблицы при миграциях и изменении модели.

Если вы не хотите тянуть Envers, можно сделать ручной аудит через отдельную сущность `AuditLog` и слушатели событий JPA (`@PrePersist`, `@PreUpdate`, `@PreRemove`) или через доменный сервис. При каждом изменении важной сущности вы пишете запись в `audit_log` с типом события, идентификатором сущности, пользователем и сериализованным состоянием (часто JSON). Это более гибко: вы сами решаете, что и как писать, но и больше кода и ответственности ложится на вас.

Выбор между версионированием в основной таблице и отдельными аудит-таблицами зависит от профиля нагрузки. Если основная работа — аналитика и отчёты по истории, версия в основной таблице (SCD2-подобная схема) удобнее: все данные в одном месте, можно использовать оконные функции, агрегаты, легко строить временные срезы. Но тогда «обычные» запросы должны быть аккуратными и всегда фильтровать по активной версии. Если же основной сценарий — оперативное чтение актуальных данных, а истории нужны эпизодически, отдельные аудит-таблицы предпочтительнее: производительность everyday-запросов не страдает.

Ещё один компромисс — хранить только агрегированную историю. Например, вместо полной копии сущности на каждое изменение вы храните только ключевые поля или только изменения по определённым атрибутам. Это сокращает объём аудит-данных, но и уменьшает их ценность. Такой подход оправдан, когда полная история слишком тяжёлая, а задач «восстановить полный снимок на дату» нет.

С точки зрения архитектуры важно отнести аудит в отдельный слой. Сущности не должны быть завалены логикой «куда и что писать», а сервисы — вручную дергать `auditService.log(...)` на каждую операцию. Лучше сделать централизованный механизм: или Envers, или один общий `AuditService`, который подписан на доменные события, или JPA-слушатели. Тогда изменение схемы аудита (например, добавление новых полей) не потребует массовых правок в бизнес-коде.

Ниже пример использования Hibernate Envers: мы аннотируем сущность `Account` как `@Audited`, а затем читаем историю через Envers API. Java и Kotlin варианты показывают базовую идею. В реальном проекте Envers будет создавать таблицу `account_AUD` с версиями записи, а вы сможете строить отчёты и расследования, используя эту таблицу.

```groovy
// build.gradle (Groovy DSL) – добавляем Envers
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.hibernate.orm:hibernate-envers'
}
```

```kotlin
// build.gradle.kts (Kotlin DSL)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.hibernate.orm:hibernate-envers")
}
```

```java
package com.example.jpa.envers;

import jakarta.persistence.*;
import org.hibernate.envers.Audited;

import java.math.BigDecimal;

@Entity
@Table(name = "account")
@Audited
public class Account {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, unique = true)
    private String number;

    @Column(nullable = false, precision = 19, scale = 4)
    private BigDecimal balance;

    // геттеры/сеттеры
}
```

```java
package com.example.jpa.envers;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class AccountHistoryService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<Account> getAccountHistory(Long accountId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        List<Number> revisions = reader.getRevisions(Account.class, accountId);

        return revisions.stream()
                .map(rev -> reader.find(Account.class, accountId, rev))
                .toList();
    }
}
```

```kotlin
package com.example.jpa.envers

import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.Id
import jakarta.persistence.Table
import org.hibernate.envers.Audited
import java.math.BigDecimal

@Entity
@Table(name = "account")
@Audited
class Account(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var number: String = "",

    var balance: BigDecimal = BigDecimal.ZERO
)
```

```kotlin
package com.example.jpa.envers

import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.hibernate.envers.AuditReaderFactory
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class AccountHistoryService(

    @PersistenceContext
    private val entityManager: EntityManager
) {

    @Transactional(readOnly = true)
    fun getAccountHistory(accountId: Long): List<Account> {
        val reader = AuditReaderFactory.get(entityManager)
        val revisions = reader.getRevisions(Account::class.java, accountId)
        return revisions.map { rev -> reader.find(Account::class.java, accountId, rev) }
    }
}
```

Если вы по каким-то причинам не хотите зависеть от Envers, можно сделать упрощённый аудит-лог: отдельная сущность `AuditLogEntry` с полями `entityType`, `entityId`, `action`, `payloadJson`, `createdAt`, `createdBy`, а писать в неё данные через `@PreUpdate/@PrePersist/@PreRemove` или через сервис. При этом стоит использовать `jsonb` для хранения сериализованного состояния и индексы по `entityType/entityId`, чтобы запросы по истории не сканировали всю таблицу. Такой подход проще контролировать, но больше ручной работы.

Главный вывод — история изменений всегда стоит ресурсов, и чем более подробную и долгоживущую историю вы хотите, тем дороже она обходится. Важно заранее определить границы: какие сущности действительно требуют версионирования, сколько времени вы обязаны хранить истории по закону/контрактам, какие СЛА по скорости запросов к истории приемлемы. Тогда вы сможете выбрать подходящий компромисс между «видеть всё» и «жить с нормальной производительностью и платой за хранение».

# 11. Спецификации, Criteria и QueryDSL

## `Specification<T>`: динамические фильтры, композиция условий, пагинация+сортировка

Спецификации в Spring Data JPA появляются там, где обычные метод-неймы репозиториев перестают масштабироваться. Пока у вас два-три поля фильтрации, `findByStatusAndCreatedAtBetween` ещё терпим. Но как только фильтр превращается в формочку с десятком параметров, размножение методов становится неконтролируемым, а поддерживать их — боль. `Specification<T>` решает именно эту проблему: позволяет собирать запросы из простых условий в рантайме, а не кодировать все комбинации в сигнатуры методов.

Технически `Specification<T>` — это функциональный интерфейс с методом `toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb)`. Внутри него вы работаете с Criteria API (тем же, на котором основан Hibernate) и строите `Predicate`, описывающий часть `where`. Такой предикат можно комбинировать с другими через `cb.and`/`cb.or` или через удобные методы `Specification.and()` / `Specification.or()`. В итоге весь динамический запрос становится композицией маленьких «атомарных» спецификаций, каждая из которых отвечает за одно логическое условие.

Практический шаблон выглядит так: вы определяете DTO фильтра (например, `CustomerFilter`), где все поля — опциональные. Затем создаёте набор статических методов `Specification<Customer> hasEmail(...)`, `registeredAfter(...)`, `withStatus(...)` и т.д. В сервисе вы берёте входной фильтр, проверяете, какие поля заполнены, и собираете итоговую спецификацию через цепочку `Specification.where(...).and(...).or(...)`. Пустые условия просто пропускаете. Такой подход хорошо читается и легко расширяется: добавился новый фильтр — добавили одну спецификацию и одно условие в сборку.

Сильная сторона спецификаций — композиция. Одно и то же условие можно использовать в разных сервисах и репозиториях, при этом логика фильтрации живёт в одном месте. Например, спецификация «клиенты этого тенанта» может использоваться в десятке сценариев и не копироваться. Кроме того, спецификации легко тестировать отдельно: вы можете навесить их на in-memory БД или Testcontainers и проверить, что задаются именно те условия, которые вы ожидаете.

Пагинация и сортировка в мире спецификаций устраиваются естественно: Spring Data JPA предоставляет метод `findAll(Specification<T> spec, Pageable pageable)`, который соединяет ваш `where` с `order by`, `limit` и `offset` из `Pageable`. Всё, что вам нужно — создать `PageRequest` с нужным `Sort`. Здесь важный нюанс: если внутри спецификации вы делаете `join fetch` коллекций, то пагинация через `Pageable` с коллекциями может выстрелить себе в ногу (дубли строк, некорректные лимиты). Для списков с `fetch`-коллекциями лучше выбирать два запроса: сначала ID’шники, потом — отдельный select с `in (...)` и `join fetch`.

Спецификации умеют фильтровать не только по полям самой сущности, но и по связям. Через `root.join("orders")` вы можете добавлять условия на связанные сущности (`order.status = ...`). Аналогично можно строить условия через `root.get("embedded").get("field")` для встраиваемых типов. Здесь легко поймать N+1, если в UI вы потом будете лениво тянуть коллекции; поэтому при сложных запросах полезно заранее платить за нужные join’ы, а не надеяться на ленивую загрузку.

На стороне репозитория всё просто: вы расширяете интерфейс `JpaRepository<T, ID>` интерфейсом `JpaSpecificationExecutor<T>`. Это добавляет вам методы вроде `findAll(Specification<T>)`, `findAll(Specification<T>, Pageable)`, `count(Specification<T>)`. Основная логика остаётся в спецификациях, а репозиторий можно считать тонкой обёрткой над `EntityManager`, умеющей понимать эти объекты.

Когда выбирать спецификации, а когда нет? Если фильтр статический и прост — обычный метод-нейм или JPQL будет проще и читабельнее. Если фильтр сложный, но спецификаций стало столько, что их поддержка превратилась в лабиринт, — имеет смысл посмотреть в сторону QueryDSL или отдельных SQL-репозиториев: там выразительность выше, а IDE лучше помогает. Спецификации хороши как middle-ground: динамичность плюс более-менее компактный код.

Есть и подводные камни. Легко сделать «монструозную» спецификацию с кучей ветвлений и условий внутри, которую потом никто не сможет читать. Хорошая практика — держать каждую спецификацию маленькой и чистой, без внешних зависимостей, и собирать их в сервисе. Также важно не забывать про `distinct(true)` в CriteriaQuery, если вы делаете join’ы и ожидаете уникальные сущности.

Тестирование спецификаций в продакшн-коде обязательно. Вы делаете несколько комбинаций фильтра, вызываете `findAll(spec)` и смотрите, что в результате корректный набор записей. Параллельно можно включать SQL-логирование и проверять, что получившийся запрос соответствует ожиданиям и использует нужные индексы. Это поможет поймать ситуации, когда спецификации неожиданно генерируют не тот `where` или `join`.

Ниже пример: сущность `Customer`, DTO фильтра и набор спецификаций, плюс сервис, который собирает их и использует с пагинацией. Java и Kotlin варианты делают одно и то же — динамический поиск клиентов по email, статусу и диапазону дат регистрации.

```java
package com.example.jpa.spec;

import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.data.jpa.repository.*;

import java.time.Instant;
import java.util.Objects;

@Entity
@Table(name = "customer")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String email;

    @Enumerated(EnumType.STRING)
    private Status status;

    @Column(name = "registered_at", nullable = false)
    private Instant registeredAt;

    public enum Status {
        ACTIVE, INACTIVE, BLOCKED
    }

    // getters/setters
}

class CustomerFilter {

    private String emailLike;
    private Customer.Status status;
    private Instant registeredFrom;
    private Instant registeredTo;

    // getters/setters
}

interface CustomerRepository extends JpaRepository<Customer, Long>,
        JpaSpecificationExecutor<Customer> {
}

final class CustomerSpecifications {

    private CustomerSpecifications() {
    }

    public static Specification<Customer> emailContains(String emailPart) {
        return (root, query, cb) -> {
            if (emailPart == null || emailPart.isBlank()) {
                return cb.conjunction();
            }
            return cb.like(cb.lower(root.get("email")), "%" + emailPart.toLowerCase() + "%");
        };
    }

    public static Specification<Customer> hasStatus(Customer.Status status) {
        return (root, query, cb) -> {
            if (status == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("status"), status);
        };
    }

    public static Specification<Customer> registeredBetween(Instant from, Instant to) {
        return (root, query, cb) -> {
            if (from == null && to == null) {
                return cb.conjunction();
            }
            if (from != null && to != null) {
                return cb.between(root.get("registeredAt"), from, to);
            }
            if (from != null) {
                return cb.greaterThanOrEqualTo(root.get("registeredAt"), from);
            }
            return cb.lessThanOrEqualTo(root.get("registeredAt"), to);
        };
    }

    public static Specification<Customer> fromFilter(CustomerFilter filter) {
        Objects.requireNonNull(filter, "filter must not be null");
        return Specification
                .where(emailContains(filter.getEmailLike()))
                .and(hasStatus(filter.getStatus()))
                .and(registeredBetween(filter.getRegisteredFrom(), filter.getRegisteredTo()));
    }
}

@Service
class CustomerSearchService {

    private final CustomerRepository customerRepository;

    CustomerSearchService(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    public Page<Customer> search(CustomerFilter filter, int page, int size) {
        Specification<Customer> spec = CustomerSpecifications.fromFilter(filter);
        Pageable pageable = PageRequest.of(page, size, Sort.by("registeredAt").descending());
        return customerRepository.findAll(spec, pageable);
    }
}
```

```kotlin
package com.example.jpa.spec

import jakarta.persistence.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.data.jpa.domain.Specification
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.JpaSpecificationExecutor
import org.springframework.stereotype.Service
import java.time.Instant

@Entity
@Table(name = "customer")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var email: String = "",

    @Enumerated(EnumType.STRING)
    var status: Status = Status.ACTIVE,

    @Column(name = "registered_at", nullable = false)
    var registeredAt: Instant = Instant.now()
) {
    enum class Status {
        ACTIVE, INACTIVE, BLOCKED
    }
}

data class CustomerFilter(
    val emailLike: String? = null,
    val status: Customer.Status? = null,
    val registeredFrom: Instant? = null,
    val registeredTo: Instant? = null
)

interface CustomerRepository : JpaRepository<Customer, Long>,
    JpaSpecificationExecutor<Customer>

object CustomerSpecifications {

    fun emailContains(emailPart: String?): Specification<Customer> =
        Specification { root, _, cb ->
            if (emailPart.isNullOrBlank()) {
                cb.conjunction()
            } else {
                cb.like(
                    cb.lower(root.get("email")),
                    "%${emailPart.lowercase()}%"
                )
            }
        }

    fun hasStatus(status: Customer.Status?): Specification<Customer> =
        Specification { root, _, cb ->
            if (status == null) cb.conjunction()
            else cb.equal(root.get<Customer.Status>("status"), status)
        }

    fun registeredBetween(from: Instant?, to: Instant?): Specification<Customer> =
        Specification { root, _, cb ->
            when {
                from == null && to == null -> cb.conjunction()
                from != null && to != null ->
                    cb.between(root.get("registeredAt"), from, to)

                from != null ->
                    cb.greaterThanOrEqualTo(root.get("registeredAt"), from)

                else ->
                    cb.lessThanOrEqualTo(root.get("registeredAt"), to)
            }
        }

    fun fromFilter(filter: CustomerFilter): Specification<Customer> =
        Specification.where(emailContains(filter.emailLike))
            ?.and(hasStatus(filter.status))
            ?.and(registeredBetween(filter.registeredFrom, filter.registeredTo))
            ?: Specification.where(null)
}

@Service
class CustomerSearchService(
    private val customerRepository: CustomerRepository
) {

    fun search(filter: CustomerFilter, page: Int, size: Int): Page<Customer> {
        val spec = CustomerSpecifications.fromFilter(filter)
        val pageable = PageRequest.of(page, size, Sort.by("registeredAt").descending())
        return customerRepository.findAll(spec, pageable)
    }
}
```

---

## Criteria API: типобезопасность vs шум кода — где помогает

JPA Criteria API — это базовый строитель запросов, на котором, по сути, живут и `Specification<T>`, и часть Hibernate-магии. Его цель — дать вам возможность конструировать запросы динамически, но при этом с типобезопасностью: вместо строковых имён полей вы работаете с типизированными путями и методами `cb.equal`, `cb.greaterThan`, `cb.and`, `cb.or` и т.п. Цена за это — очень многословный код: простое условие превращается в несколько строк с `Root`, `CriteriaQuery`, `Predicate`. Поэтому Criteria редко используют прямо в сервисах — чаще его прячут в спецификации или infrastructural-код.

Базовая схема работы с Criteria такова: вы берёте `CriteriaBuilder` из `EntityManager`, создаёте `CriteriaQuery<T>`, объявляете корень `Root<T> root = query.from(T.class)` и затем собираете список предикатов. После этого говорите `query.select(root).where(cb.and(predicates...))`, создаёте `TypedQuery<T>` через `entityManager.createQuery(query)` и уже на нём настраиваете пагинацию и выполняете запрос. Всё довольно механистично, но даёт полный контроль: вы можете делать join’ы, подзапросы, группировки и т.д.

Типобезопасность Criteria заключается в том, что вы оперируете типами: `root.get("amount")` возвращает `Path<BigDecimal>`, и IDE/компилятор сразу скажут, если вы попытаетесь сравнить его с `String`. Но имена полей всё равно указываются строкой, и переименование поля в entity без корректного рефакторинга приведёт к runtime-ошибке. Для полной типобезопасности нужны метамодели (`Static Metamodel`) или QueryDSL; Criteria-метамодель в JPA есть, но на практике ею пользуются немногие, как раз из-за сложности и объёма кода.

Criteria API особенно полезна, когда вам нужно строить сложные динамические запросы на уровне инфраструктуры: например, в кастомном репозитории, который умеет читать фильтры в виде DTO и превращать их в запрос, или в generic-слое, который работает с разными сущностями одинаково. Там, где заранее неизвестно, какие именно поля будут участвовать в фильтрации или сортировке, Criteria позволяет программно собрать нужные условия, не пиша руками JPQL.

Пример типичного сценария: поиск заказов по нескольким опциональным параметрам — email клиента, диапазон дат, минимальная сумма и список статусов. В Criteria это реализуется как список `Predicate`, который вы постепенно наполняете в зависимости от заполненности фильтра. Для связей (`Order` → `Customer`) вы создаёте `Join` и добавляете условия уже на него. Код получается длиннее, чем JPQL, но легко расширяется, если фильтр вырос.

С join’ами Criteria работает напрямую: `Join<Order, Customer> customerJoin = root.join("customer", JoinType.INNER)`. Вы можете строить сложные цепочки join’ов, добавлять условия на них, использовать `fetch` для подгрузки коллекций. Это очень мощный инструмент, особенно для отчётных запросов и агрегатов, но требует дисциплины: легко написать запрос, который будет красиво выглядеть в Java, но плохо работать в SQL (лишние join’ы, отсутствие индексов, фильтры «не там»).

Пагинация и сортировка в Criteria делаются вручную: `query.orderBy(cb.desc(root.get("createdAt")))`, а затем на `TypedQuery` — `setFirstResult(page * size)` и `setMaxResults(size)`. В связке со Spring Boot вы можете обернуть это в сервисный метод, принимающий `Pageable`, и маппить `Sort.Order` на `Order` Criteria: для каждого `Sort.Order` добавляете в `orderBy` либо `cb.asc(path)`, либо `cb.desc(path)`. Здесь нужно аккуратно относиться к имёнам полей, чтобы не допустить SQL-инъекцию: лучше жёстко ограничивать список колонок, по которым разрешена сортировка.

Где Criteria реально побеждает строковый JPQL — это там, где запрос сильно динамический: например, гибкая фильтрация по набору настроек пользователя, где количество условий зависит от конфигурации, или generic-поисковые репозитории, которые позволяют выбирать поля фильтрации на лету. Писать такой код на JPQL практически невозможно без адского конкатенирования строк и ручного контроля параметров.

Но есть и обратная сторона. Код на Criteria API тяжело читать и ревьюить, особенно если нет чёткого стиля. В команде лучше принять правило: Criteria используется только в инфраструктурном слое (спеки, кастомные репозитории), а на уровне бизнес-логики мы оперируем понятными методами сервисов и репозиториев. Тогда большинство разработчиков не будут вынуждены регулярно погружаться в дебри `CriteriaBuilder`, а сосредоточатся на доменных задачах.

Отладка Criteria-запросов сводится к включению SQL-логов и анализу сгенерированного запроса. Hibernate сам генерирует JPQL → SQL, и иногда результат отличается от того, что вы ожидали. Поэтому при первых внедрениях Criteria обязательно проверяйте SQL глазами, смотрите планы выполнения (EXPLAIN ANALYZE) и убеждайтесь, что используются нужные индексы, нет лишних join’ов и сортировок по неиндексированным колонкам.

Ниже пример: кастомный репозиторий для сущности `Order`, где мы вручную используем Criteria API для динамического поиска по фильтру `OrderFilter`. Java-вариант показывает работу с `CriteriaBuilder` и встроенной пагинацией, Kotlin-вариант — то же самое в более компактном виде.

```java
package com.example.jpa.criteria;

import jakarta.persistence.*;
import jakarta.persistence.criteria.*;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    private String number;

    @Column(nullable = false)
    private BigDecimal amount;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    // getters/setters
}

class OrderFilter {

    private String customerEmail;
    private Instant createdFrom;
    private Instant createdTo;
    private BigDecimal minAmount;

    // getters/setters
}

@Repository
@Transactional(readOnly = true)
public class OrderCriteriaRepository {

    @PersistenceContext
    private EntityManager em;

    public Page<Order> findByFilter(OrderFilter filter, Pageable pageable) {
        CriteriaBuilder cb = em.getCriteriaBuilder();

        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> root = cq.from(Order.class);
        Join<Order, Customer> customerJoin = root.join("customer", JoinType.INNER);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getCustomerEmail() != null && !filter.getCustomerEmail().isBlank()) {
            predicates.add(
                    cb.like(
                            cb.lower(customerJoin.get("email")),
                            "%" + filter.getCustomerEmail().toLowerCase() + "%"
                    )
            );
        }

        if (filter.getCreatedFrom() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("createdAt"), filter.getCreatedFrom()));
        }

        if (filter.getCreatedTo() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("createdAt"), filter.getCreatedTo()));
        }

        if (filter.getMinAmount() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("amount"), filter.getMinAmount()));
        }

        cq.select(root).where(cb.and(predicates.toArray(Predicate[]::new)));

        if (pageable.getSort().isSorted()) {
            List<Order> orders = new ArrayList<>();
            pageable.getSort().forEach(order -> {
                Path<?> path = root.get(order.getProperty());
                orders.add(order.isAscending() ? cb.asc(path) : cb.desc(path));
            });
            cq.orderBy(orders);
        }

        TypedQuery<Order> query = em.createQuery(cq);
        query.setFirstResult((int) pageable.getOffset());
        query.setMaxResults(pageable.getPageSize());

        List<Order> content = query.getResultList();

        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<Order> countRoot = countQuery.from(Order.class);
        Join<Order, Customer> countCustomerJoin = countRoot.join("customer", JoinType.INNER);

        List<Predicate> countPredicates = new ArrayList<>();

        if (filter.getCustomerEmail() != null && !filter.getCustomerEmail().isBlank()) {
            countPredicates.add(
                    cb.like(
                            cb.lower(countCustomerJoin.get("email")),
                            "%" + filter.getCustomerEmail().toLowerCase() + "%"
                    )
            );
        }
        if (filter.getCreatedFrom() != null) {
            countPredicates.add(cb.greaterThanOrEqualTo(countRoot.get("createdAt"), filter.getCreatedFrom()));
        }
        if (filter.getCreatedTo() != null) {
            countPredicates.add(cb.lessThanOrEqualTo(countRoot.get("createdAt"), filter.getCreatedTo()));
        }
        if (filter.getMinAmount() != null) {
            countPredicates.add(cb.greaterThanOrEqualTo(countRoot.get("amount"), filter.getMinAmount()));
        }

        countQuery.select(cb.countDistinct(countRoot))
                .where(cb.and(countPredicates.toArray(Predicate[]::new)));

        Long total = em.createQuery(countQuery).getSingleResult();

        return new PageImpl<>(content, pageable, total);
    }
}
```

```kotlin
package com.example.jpa.criteria

import jakarta.persistence.*
import jakarta.persistence.criteria.JoinType
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageImpl
import org.springframework.data.domain.Pageable
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.time.Instant

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var number: String = "",

    @Column(nullable = false)
    var amount: BigDecimal = BigDecimal.ZERO,

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now(),

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    var customer: Customer? = null
)

data class OrderFilter(
    val customerEmail: String? = null,
    val createdFrom: Instant? = null,
    val createdTo: Instant? = null,
    val minAmount: BigDecimal? = null
)

@Repository
@Transactional(readOnly = true)
class OrderCriteriaRepository(

    @PersistenceContext
    private val em: EntityManager
) {

    fun findByFilter(filter: OrderFilter, pageable: Pageable): Page<Order> {
        val cb = em.criteriaBuilder

        val cq = cb.createQuery(Order::class.java)
        val root = cq.from(Order::class.java)
        val customerJoin = root.join<Order, Customer>("customer", JoinType.INNER)

        val predicates = mutableListOf<jakarta.persistence.criteria.Predicate>()

        filter.customerEmail
            ?.takeIf { it.isNotBlank() }
            ?.let {
                predicates += cb.like(
                    cb.lower(customerJoin.get("email")),
                    "%${it.lowercase()}%"
                )
            }

        filter.createdFrom?.let {
            predicates += cb.greaterThanOrEqualTo(root.get("createdAt"), it)
        }

        filter.createdTo?.let {
            predicates += cb.lessThanOrEqualTo(root.get("createdAt"), it)
        }

        filter.minAmount?.let {
            predicates += cb.greaterThanOrEqualTo(root.get("amount"), it)
        }

        cq.select(root).where(*predicates.toTypedArray())

        if (pageable.sort.isSorted) {
            val orders = pageable.sort.map { order ->
                val path = root.get<Any>(order.property)
                if (order.isAscending) cb.asc(path) else cb.desc(path)
            }
            cq.orderBy(orders)
        }

        val query = em.createQuery(cq)
        query.firstResult = pageable.offset.toInt()
        query.maxResults = pageable.pageSize
        val content = query.resultList

        val countCq = cb.createQuery(Long::class.java)
        val countRoot = countCq.from(Order::class.java)
        val countCustomerJoin = countRoot.join<Order, Customer>("customer", JoinType.INNER)

        val countPredicates = mutableListOf<jakarta.persistence.criteria.Predicate>()

        filter.customerEmail
            ?.takeIf { it.isNotBlank() }
            ?.let {
                countPredicates += cb.like(
                    cb.lower(countCustomerJoin.get("email")),
                    "%${it.lowercase()}%"
                )
            }

        filter.createdFrom?.let {
            countPredicates += cb.greaterThanOrEqualTo(countRoot.get("createdAt"), it)
        }

        filter.createdTo?.let {
            countPredicates += cb.lessThanOrEqualTo(countRoot.get("createdAt"), it)
        }

        filter.minAmount?.let {
            countPredicates += cb.greaterThanOrEqualTo(countRoot.get("amount"), it)
        }

        countCq.select(cb.countDistinct(countRoot))
            .where(*countPredicates.toTypedArray())

        val total = em.createQuery(countCq).singleResult

        return PageImpl(content, pageable, total)
    }
}
```

---

## QueryDSL: предикаты, join-ы, подзапросы, «тяжёлые» фильтры — когда лучше, чем строковый JPQL

QueryDSL решает ту же задачу, что и Criteria API, но делает это в куда более приятной форме. Вместо того чтобы писать многословный код с `CriteriaBuilder`, вы работаете с сгенерированными классами `QEntity` и fluent-DSL: `QOrder.order.amount.gt(...)`, `order.customer.email.eq(...)`, `query.select(order).from(order).where(...)`. В результате запросы остаются типобезопасными, но читаются почти как SQL/JPQL. Цена — необходимость генерировать `Q`-классы на этапе сборки и тянуть дополнительную зависимость.

Чтобы использовать QueryDSL с JPA, вам нужны две зависимости: сама библиотека `querydsl-jpa` и APT-модуль `querydsl-apt` с профилем `jpa` для генерации метамодели. В Gradle Groovy-проекте вы добавляете `annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jpa'`, в Kotlin-проекте — `kapt("com.querydsl:querydsl-apt:5.0.0:jpa")`. После сборки у вас появляются классы `QOrder`, `QCustomer` и т.д. в `build/generated`. Они содержат типизированные поля (`StringPath`, `NumberPath`, `DateTimePath`), через которые вы строите выражения.

Основные строительные блоки QueryDSL — это `JPAQueryFactory` и предикаты. `JPAQueryFactory` оборачивает `EntityManager` и даёт удобный API `select`, `from`, `join`, `where`, `orderBy`, `offset`, `limit`. Предикаты — это выражения вроде `qOrder.amount.gt(BigDecimal.TEN)`, которые можно комбинировать через `.and` / `.or`. Всё типобезопасно: вы не сможете сравнить поле `amount` с `String`, компилятор не пропустит. Отдельный плюс — автокомплит IDE по полям сущностей: вы не пишете имена строками, а выбираете их из подсказки.

С Spring Data JPA QueryDSL интегрируется через `QuerydslPredicateExecutor` или через кастомные репозитории с внедрением `JPAQueryFactory`. Первый вариант («предикаты как аргументы методов репозитория») удобен, если вы хотите экспонировать QueryDSL наверх: сервисы строят `Predicate` и отдают его в `findAll(predicate, pageable)`. Второй — когда вы хотите скрыть QueryDSL внутри репозитория и наружу отдавать только высокоуровневые методы `findOrdersByFilter(...)`.

QueryDSL особенно выигрывает там, где запросы сложные: join нескольких таблиц, подзапросы, агрегации, фильтрация по вычисляемым выражениям. В строковом JPQL такие запросы быстро превращаются в нечитаемую простыню, Criteria — в лес вызовов `cb.something`, а QueryDSL позволяет аккуратно разбить их на части и даже вынести повторяющиеся фрагменты в методы, возвращающие `BooleanExpression`. Это делает код не только короче, но и сильно понятнее для ревью.

Предикаты QueryDSL легко комбинировать. Вы можете начать с `BooleanExpression expr = Expressions.asBoolean(true).isTrue();` и затем наращивать: `expr = expr.and(qOrder.createdAt.goe(from))`, `expr = expr.and(qOrder.status.in(statuses))` и т.д. В Kotlin это ещё приятнее благодаря extension-функциям и `null`-безопасности: можно собирать `BooleanBuilder` или писать маленькие функции `fun hasStatus(status: Status?) = status?.let { qOrder.status.eq(it) }`.

Пагинация и сортировка в QueryDSL делаются довольно естественно: `query.offset(pageable.offset).limit(pageable.pageSize).orderBy(...)`. Есть утилиты вроде `Querydsl` и `QuerydslRepositorySupport`, которые умеют маппить `Pageable` на `orderBy`, но во многих случаях проще написать маппинг вручную или ограничиться несколькими фиксированными полями сортировки. Главное — не забыть о `orderBy` при сложных запросах, иначе у вас будут плавающие результаты на разных страницах.

По сравнению с Criteria API QueryDSL почти всегда выигрывает по читабельности. Код короче, вложенность меньше, и IDE лучше помогает. Поэтому если в проекте есть серьёзные запросы, которые сложно выразить через Spring Data или JPQL, QueryDSL часто становится стандартным инструментом. Недостаток — дополнительный шаг генерации и зависимость от конкретной библиотеки: если вы когда-нибудь решите уйти с QueryDSL, придётся переписать немало кода.

Подводные камни у QueryDSL те же, что и у любого ORM-кода: N+1 на ленивых связях, неоптимальные join’ы, отсутствие нужных индексов. QueryDSL не делает запрос «хорошим» автоматически, он лишь делает его легче читаемым и типобезопасным. Поэтому всё равно нужно смотреть SQL, анализировать планы, следить за размером выборок. Отдельно стоит упомянуть, что старые методы вроде `fetchResults()` в последних версиях помечены deprecated, и лучше делать `fetch()` плюс отдельный count-запрос для пагинации.

Ещё одна тонкость — генерация `Q`-классов при сложных схемах. Если вы злоупотребляете `@ManyToMany`, «комбинаторными» связями и наследованием, `Q`-модель может получиться громоздкой и шумной. Но это скорее индикатор проблем модели, чем QueryDSL. В здоровой доменной модели `Q`-классы выглядят вполне разумно, и работа с ними не вызывает боли.

Архитектурно разумно ограничить использование QueryDSL инфраструктурным слоем. Обычно это кастомные репозитории и, максимум, сервисы уровня «поиска» или «отчётности». Вы не хотите, чтобы весь бизнес-слой оперировал `Q`-классами — это сильно привязывает его к конкретному ORM/библиотеке. Лучше наружу отдавать уже понятные методы `searchOrders(filter)` и DTO, а QueryDSL оставлять в реализации.

Ниже пример конфигурации Gradle для QueryDSL и простого репозитория, который использует `JPAQueryFactory` для поиска заказов по фильтру. В коде показаны Java и Kotlin варианты; логика одна: динамический фильтр по email клиента, диапазону дат и минимальной сумме.

```groovy
// build.gradle (Groovy DSL) – зависимости QueryDSL
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'

    implementation 'com.querydsl:querydsl-jpa:5.0.0'
    annotationProcessor 'jakarta.persistence:jakarta.persistence-api:3.1.0'
    annotationProcessor 'com.querydsl:querydsl-apt:5.0.0:jpa'
}
```

```kotlin
// build.gradle.kts (Kotlin DSL)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")

    implementation("com.querydsl:querydsl-jpa:5.0.0")
    kapt("jakarta.persistence:jakarta.persistence-api:3.1.0")
    kapt("com.querydsl:querydsl-apt:5.0.0:jpa")
}
```

```java
package com.example.jpa.querydsl;

import com.querydsl.core.types.dsl.BooleanExpression;
import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.*;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    private String number;

    private BigDecimal amount;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    // getters/setters
}

class OrderFilter {

    private String customerEmail;
    private Instant createdFrom;
    private Instant createdTo;
    private BigDecimal minAmount;

    // getters/setters
}

@Repository
@Transactional(readOnly = true)
public class OrderQuerydslRepository {

    private final JPAQueryFactory queryFactory;

    public OrderQuerydslRepository(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }

    public Page<Order> findByFilter(OrderFilter filter, Pageable pageable) {
        QOrder qOrder = QOrder.order;
        QCustomer qCustomer = QCustomer.customer;

        BooleanExpression predicate = buildPredicate(filter, qOrder, qCustomer);

        var baseQuery = queryFactory
                .selectFrom(qOrder)
                .join(qOrder.customer, qCustomer).fetchJoin()
                .where(predicate);

        if (pageable.getSort().isSorted()) {
            for (Sort.Order order : pageable.getSort()) {
                if ("createdAt".equals(order.getProperty())) {
                    baseQuery.orderBy(
                            order.isAscending() ? qOrder.createdAt.asc() : qOrder.createdAt.desc()
                    );
                }
            }
        }

        long total = baseQuery.clone().fetch().size();

        var content = baseQuery
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();

        return new PageImpl<>(content, pageable, total);
    }

    private BooleanExpression buildPredicate(OrderFilter filter,
                                             QOrder qOrder,
                                             QCustomer qCustomer) {
        BooleanExpression predicate = qOrder.isNotNull();

        if (filter.getCustomerEmail() != null && !filter.getCustomerEmail().isBlank()) {
            predicate = predicate.and(
                    qCustomer.email.containsIgnoreCase(filter.getCustomerEmail())
            );
        }

        if (filter.getCreatedFrom() != null) {
            predicate = predicate.and(qOrder.createdAt.goe(filter.getCreatedFrom()));
        }

        if (filter.getCreatedTo() != null) {
            predicate = predicate.and(qOrder.createdAt.loe(filter.getCreatedTo()));
        }

        if (filter.getMinAmount() != null) {
            predicate = predicate.and(qOrder.amount.goe(filter.getMinAmount()));
        }

        return predicate;
    }
}
```

```kotlin
package com.example.jpa.querydsl

import com.querydsl.core.types.dsl.BooleanExpression
import com.querydsl.jpa.impl.JPAQueryFactory
import jakarta.persistence.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.PageImpl
import org.springframework.data.domain.Pageable
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.time.Instant

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var number: String = "",

    var amount: BigDecimal = BigDecimal.ZERO,

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now(),

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    var customer: Customer? = null
)

data class OrderFilter(
    val customerEmail: String? = null,
    val createdFrom: Instant? = null,
    val createdTo: Instant? = null,
    val minAmount: BigDecimal? = null
)

@Repository
@Transactional(readOnly = true)
class OrderQuerydslRepository(
    em: EntityManager
) {

    private val queryFactory = JPAQueryFactory(em)

    fun findByFilter(filter: OrderFilter, pageable: Pageable): Page<Order> {
        val qOrder = QOrder.order
        val qCustomer = QCustomer.customer

        val predicate = buildPredicate(filter, qOrder, qCustomer)

        val baseQuery = queryFactory
            .selectFrom(qOrder)
            .join(qOrder.customer, qCustomer).fetchJoin()
            .where(predicate)

        if (pageable.sort.isSorted) {
            pageable.sort.forEach { order ->
                if (order.property == "createdAt") {
                    baseQuery.orderBy(
                        if (order.isAscending) qOrder.createdAt.asc() else qOrder.createdAt.desc()
                    )
                }
            }
        }

        val all = baseQuery.clone().fetch()
        val total = all.size.toLong()

        val content = baseQuery
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetch()

        return PageImpl(content, pageable, total)
    }

    private fun buildPredicate(
        filter: OrderFilter,
        qOrder: QOrder,
        qCustomer: QCustomer
    ): BooleanExpression {
        var predicate: BooleanExpression = qOrder.isNotNull

        filter.customerEmail
            ?.takeIf { it.isNotBlank() }
            ?.let {
                predicate = predicate.and(
                    qCustomer.email.containsIgnoreCase(it)
                )
            }

        filter.createdFrom?.let {
            predicate = predicate.and(qOrder.createdAt.goe(it))
        }

        filter.createdTo?.let {
            predicate = predicate.and(qOrder.createdAt.loe(it))
        }

        filter.minAmount?.let {
            predicate = predicate.and(qOrder.amount.goe(it))
        }

        return predicate
    }
}
```

В итоге картина такая: для умеренно сложных динамических фильтров удобно начинать со `Specification<T>`, под капотом которых живёт Criteria API. Когда запросы становятся тяжёлыми, много join’ов и подзапросов, — QueryDSL даёт более читаемый и типобезопасный DSL поверх того же JPA. Во всех трёх случаях важно помнить, что в конце всё упирается в SQL и индексы; инструменты лишь помогают вам написать этот SQL аккуратнее и безопаснее.

# 12. JDBC и jOOQ: когда уходить ниже

## Где JPA неэффективна: отчёты, агрегаты/окна, сложные CTE, апсерты/`ON CONFLICT`, bulk-операции

Когда мы говорим «Spring Data JPA», почти всегда имеем в виду CRUD-операции и относительно простые выборки по сущностям: найти по id, вернуть страницу по фильтру, сохранить/обновить. Для этого JPA и проектировалась: скрыть от разработчика большую часть рутины ORM, дать объектно-ориентированную модель поверх таблиц и избавить от постоянного написания `SELECT ... FROM ...`. Но у такого подхода есть естественная граница. Как только вы выходите в отчёты, тяжёлую аналитику, сложные агрегации и vendor-specific возможности СУБД, JPA начинает работать против вас.

Первая большая зона, где JPA неудобна, — отчётные запросы и агрегаты. Например, вам нужно посчитать сумму и количество заказов по дням с разбиением по статусу, выдать топ-N клиентов по обороту, построить гистограмму по корзине. С точки зрения БД это `GROUP BY`, `HAVING`, агрегатные функции и иногда оконные функции. С точки зрения JPA — необходимость либо писать сложный JPQL с конструкторными проекциями, либо сразу падать в native SQL. Чем сложнее отчёт, тем меньше смысла держаться за «объектность»: вам всё равно нужны SQL-концепции, и ORM только мешает.

Вторая зона — оконные функции и CTE (`WITH`-выражения). Современные СУБД (PostgreSQL, Oracle, SQL Server) умеют мощные вещи вроде `row_number() over (...)`, `lag/lead`, рекурсивные CTE и т.п. JPQL этого не знает, Criteria API — тоже; остаётся только `nativeQuery`. Да, Hibernate позволяет встраивать native-запросы в репозитории, но это уже не «JPA как абстракция над SQL», а «SQL с тонкой обёрткой Spring Data». Если ваш отчёт или алгоритм естественно формулируется в терминах оконных функций и CTE, честнее признать, что вы работаете на уровне SQL/JDBC и строить архитектуру вокруг этого.

Отдельная боль JPA — апсерты (UPSERT). Типичный кейс: «вставить запись, а если ключ уже существует — обновить». В PostgreSQL это `INSERT ... ON CONFLICT (key) DO UPDATE`, в других СУБД — свои варианты `MERGE`. JPA не стандартизирует апсерты: у вас либо два запроса (сначала `select`, потом `insert/update`), либо `@Query(nativeQuery = true)` со строкой SQL. При большом трафике и высокой конкуренции второй вариант предпочителен, но это уже явный выход за рамки «чистой JPA». Там, где апсерты — нормальная часть бизнес-логики (идемпотентные обработчики, интеграции), имеет смысл сразу спускаться на уровень JDBC или jOOQ.

Третья важная область — массовые операции (bulk update/delete). JPA умеет `update ... where` и `delete ... where` в JPQL, но при этом такие операции обходят 1-й уровень кэша, не вызывают entity-listeners и легко раскоординируются с состоянием persistence context. Если вы активно используете кэш сущностей, версионирование, доменные события на `@PreUpdate/@PostUpdate`, массовые апдейты через JPQL становятся источником сюрпризов. Часто проще и честнее сделать отдельный «репозиторий на JDBC», который выполняет чистые SQL-операции и никак не связан с JPA-контекстом.

Ещё одна причина уходить ниже — сложные ETL/миграции и сервисы отчётности. Бывают случаи, когда сервису нужно перелопатить миллионы строк, сделать тяжёлую агрегацию, сбросить результаты в промежуточную таблицу или внешний storage. Тащить такие объёмы через JPA-сущности, дергая `entityManager.persist()` в цикле, — худшее из решений: вы нагружаете ORM, раздуваете контекст, получаете лишние проверки dirty-tracking и тратите память. Для таких задач намного здоровее использовать `JdbcTemplate` или jOOQ с тонким контролем над размером батчей, курсорами и транзакционностью.

К специфике СУБД JPA тоже относится слабо. Примеры — JSONB в PostgreSQL, полнотекстовый поиск (`tsvector`/`tsquery`), массивы, диапазоны, специфичные типы индексов. Да, существуют расширения (`hibernate-types` и подобные), но как только запрос выходит за рамки «вытащить поле JSON», а нужно сделать приличный поиск по структуре, короткая и точная SQL-конструкция будет проще, чем попытки натянуть всё это на Criteria или JPQL. В ситуациях, когда архитектурный центр — именно возможности базы, а не объектная модель, JPA должна отойти на второй план.

Нужно помнить и про производительность. Нередко JPA генерирует SQL не так, как вы бы написали руками: лишние join’ы, подзапросы, выборка полей, которые не нужны, невозможность подсказать planner’у нужный индекс. В простых CRUD-сценариях это приемлемая цена за удобство, но в тяжёлых отчётах или при больших объёмах данных каждый лишний join превращается в лишние десятки миллисекунд и мегабайты памяти. Там, где у вас есть жёсткие SLA по тяжёлым запросам, естественный путь — взять управление SQL на себя.

Архитектурный вывод: JPA отлично подходит как «рабочая лошадь» для большинства бизнес-операций, где важно работать с доменной моделью, а не с таблицами. Но у каждого сервиса есть куски функциональности, которые по своей природе ближе к «SQL-аналитике», чем к «доменно-ориентированному CRUD». Именно для этих кусков и имеет смысл использовать JDBC или jOOQ: вы точно понимаете, какой SQL выполняется, контролируете план, используете возможности конкретной СУБД и не тратите ресурсы на ORM там, где она не даёт ценности.

Обычно архитектурный паттерн выглядит так: на уровне сервиса есть обычные Spring Data JPA-репозитории для работающей части системы (заказы, клиенты, статусы), а рядом — отчётный/технический слой на JDBC/jOOQ для отчётов, batch-задач, сложных агрегатов. Граница между ними проходит по принципу: «если запрос легко читается как JPQL по сущностям — оставляем в JPA, если он естественно формулируется как SQL по таблицам — делаем JDBC/jOOQ».

Простой пример — отчёт по обороту клиентов с использованием оконной функции `row_number` и `sum` по окну. JPQL такого не умеет, а native-запрос в JPA — по сути уже прямой SQL. В таких местах намного логичнее сделать отдельный репозиторий на `JdbcTemplate` и явно сказать: «это отчётный SQL-слой, он сознательно работает на уровне таблиц».

Ниже небольшой пример для отчётного запроса: Java-репозиторий, который выполняет native SQL с оконной функцией через `JdbcTemplate`, и эквивалент на Kotlin. Здесь основная демонстрация — что, как только запрос стал нетривиальным, мы честно переключаемся на прямой SQL, а не пытаемся выжать это из JPA.

```java
package com.example.jpa.report;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.LocalDate;
import java.util.List;

@Repository
public class CustomerTurnoverReportRepository {

    private final JdbcTemplate jdbcTemplate;

    public CustomerTurnoverReportRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<CustomerTurnoverRow> findTopCustomersByTurnover(LocalDate from, LocalDate to, int limit) {
        String sql = """
            SELECT customer_id,
                   SUM(amount)              AS total_amount,
                   COUNT(*)                 AS orders_count,
                   ROW_NUMBER() OVER (ORDER BY SUM(amount) DESC) AS rn
            FROM orders
            WHERE created_at >= ? AND created_at < ?
            GROUP BY customer_id
            ORDER BY total_amount DESC
            LIMIT ?
            """;

        return jdbcTemplate.query(
                sql,
                (rs, rowNum) -> mapRow(rs),
                from.atStartOfDay(),
                to.plusDays(1).atStartOfDay(),
                limit
        );
    }

    private CustomerTurnoverRow mapRow(ResultSet rs) throws SQLException {
        return new CustomerTurnoverRow(
                rs.getLong("customer_id"),
                rs.getBigDecimal("total_amount"),
                rs.getLong("orders_count"),
                rs.getInt("rn")
        );
    }

    public record CustomerTurnoverRow(
            Long customerId,
            BigDecimal totalAmount,
            Long ordersCount,
            Integer rank
    ) {
    }
}
```

```kotlin
package com.example.jpa.report

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.RowMapper
import org.springframework.stereotype.Repository
import java.math.BigDecimal
import java.sql.ResultSet
import java.time.LocalDate

@Repository
class CustomerTurnoverReportRepository(
    private val jdbcTemplate: JdbcTemplate
) {

    fun findTopCustomersByTurnover(
        from: LocalDate,
        to: LocalDate,
        limit: Int
    ): List<CustomerTurnoverRow> {
        val sql = """
            SELECT customer_id,
                   SUM(amount)              AS total_amount,
                   COUNT(*)                 AS orders_count,
                   ROW_NUMBER() OVER (ORDER BY SUM(amount) DESC) AS rn
            FROM orders
            WHERE created_at >= ? AND created_at < ?
            GROUP BY customer_id
            ORDER BY total_amount DESC
            LIMIT ?
        """.trimIndent()

        return jdbcTemplate.query(sql, rowMapper,
            from.atStartOfDay(),
            to.plusDays(1).atStartOfDay(),
            limit
        )
    }

    private val rowMapper = RowMapper { rs: ResultSet, _: Int ->
        CustomerTurnoverRow(
            customerId = rs.getLong("customer_id"),
            totalAmount = rs.getBigDecimal("total_amount") ?: BigDecimal.ZERO,
            ordersCount = rs.getLong("orders_count"),
            rank = rs.getInt("rn")
        )
    }
}

data class CustomerTurnoverRow(
    val customerId: Long,
    val totalAmount: BigDecimal,
    val ordersCount: Long,
    val rank: Int
)
```

Этот пример показывает суть: как только запрос начинает жить своей SQL-жизнью, не надо мучить JPA — проще честно перейти на JDBC/jOOQ и держать этот слой отдельно, но рядом с доменной моделью.

---

## `JdbcTemplate/NamedParameterJdbcTemplate`: `RowMapper`, батчи `batchUpdate`, `SimpleJdbcInsert`, таймауты и `fetchSize`

`JdbcTemplate` — базовый инструмент Spring для работы с JDBC. Он решает типичные боли «голого» JDBC: управление ресурсами, обработку исключений, шаблон `try-with-resources`, перебор `ResultSet`. Вместо того чтобы писать десяток строк для открытия соединения и закрытия всего подряд, вы пишете одну строку `jdbcTemplate.query(...)` и передаёте лямбду, которая превращает строки результата в ваши объекты. `NamedParameterJdbcTemplate` добавляет к этому поддержку именованных параметров вместо позиционных `?`, что сильно улучшает читаемость сложных запросов.

Ключевая абстракция на чтении — `RowMapper<T>`. Это интерфейс, который превращает одну строку `ResultSet` в доменный объект. Вы пишете «как маппить одну строку», а `JdbcTemplate` делает всё остальное: бежит по `ResultSet`, вызывает `RowMapper` для каждой строки, собирает список. Это намного прозрачнее и дешевле, чем создавать временные сущности JPA, особенно если вам нужны только несколько полей из таблицы. Важно не злоупотреблять `BeanPropertyRowMapper` на проде: лучше писать явный маппинг, чтобы контролировать типы и избежать сюрпризов.

На записи важную роль играют батчи. `JdbcTemplate.batchUpdate` позволяет отправить много однотипных операций (вставок/обновлений) одним сетом, а не выполнять их по одной. Для PostgreSQL и многих других СУБД это даёт существенный выигрыш по времени и нагрузке на сеть/сервер. Главное — правильно выбирать размер батча: слишком маленький не даёт эффекта, слишком большой может забить буфер и создать долгие транзакции. Типичный размер — 100–1000 строк в батче, но зависит от профиля нагрузки и СУБД.

`NamedParameterJdbcTemplate` упрощает работу с запросами, где много параметров и не хочется считать позиции `?`. Вместо `WHERE status = ? AND created_at >= ? AND created_at < ?` вы пишете `:status`, `:from`, `:to` и передаёте `Map<String, Object>` или `SqlParameterSource`. Это особенно удобно в DAO-слое, где запросы со временем эволюционируют, добавляются новые фильтры и легко ошибиться в номерах параметров.

`SimpleJdbcInsert` — ещё один полезный инструмент для «тупых» вставок, когда вам не хочется писать `INSERT` руками. Вы настраиваете таблицу и список колонок, а затем передаёте `Map<String, Object>` с данными; `SimpleJdbcInsert` сам соберёт SQL и выполнит его, при необходимости вернув сгенерированный ключ. Хорошо подходит для технологических таблиц, логов, вспомогательных сущностей, где не нужна JPA-модель и важна только запись данных.

Отдельная тема — тайм-ауты и `fetchSize`. С точки зрения надёжности важно, чтобы каждый SQL-запрос имел разумный верхний предел времени выполнения. Его можно задавать на уровне DataSource/драйвера, через `@Transactional(timeout = ...)` или через `JdbcTemplate` (например, через настройку `setQueryTimeout` на `PreparedStatement`). Для долгих запросов также важно управлять `fetchSize`: это подсказка драйверу, сколько строк за раз забирать с сервера. В PostgreSQL при правильной настройке это приводит к использованию серверных курсоров и потоковой выдаче результата.

В контексте Spring Boot большинство бинов `JdbcTemplate` и `NamedParameterJdbcTemplate` создаются автоматически, если у вас есть `DataSource`. Вам редко нужно настраивать их вручную, кроме случаев, когда вы хотите специфичные настройки (например, логирование, настройка `fetchSize`, специальные тайм-ауты). Но хороший тон — хотя бы понимать, как они устроены и что где можно подкрутить.

Частая практика — иметь в приложении JPA-репозитории для основной работы и один-два `JdbcTemplate`-репозитория для тяжёлых вещей: batch-записей, отчётов, сервисных таблиц. Так вы не тащите ORM туда, где она не нужна, но и не отказываетесь от удобств Spring Data там, где они дают максимум выгоды.

Ниже пример: конфигурация бинов `JdbcTemplate` и `NamedParameterJdbcTemplate` (если хотите их явного контроля) и репозиторий, который делает batch-вставку и выборку с именованными параметрами. Java и Kotlin варианты эквивалентны.

```java
package com.example.jdbc.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.*;

import javax.sql.DataSource;

@Configuration
public class JdbcConfig {

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        JdbcTemplate template = new JdbcTemplate(dataSource);
        template.setFetchSize(500);
        return template;
    }

    @Bean
    public NamedParameterJdbcTemplate namedParameterJdbcTemplate(DataSource dataSource) {
        return new NamedParameterJdbcTemplate(dataSource);
    }
}
```

```java
package com.example.jdbc.repo;

import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.core.namedparam.*;
import org.springframework.stereotype.Repository;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.Instant;
import java.util.List;
import java.util.Map;

@Repository
public class AuditEventJdbcRepository {

    private final NamedParameterJdbcTemplate jdbc;

    public AuditEventJdbcRepository(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public void saveBatch(List<AuditEvent> events) {
        String sql = """
            INSERT INTO audit_event(event_type, payload, created_at)
            VALUES (:eventType, :payload, :createdAt)
            """;

        SqlParameterSource[] batchParams = events.stream()
                .map(event -> new MapSqlParameterSource()
                        .addValue("eventType", event.eventType())
                        .addValue("payload", event.payload())
                        .addValue("createdAt", event.createdAt())
                )
                .toArray(SqlParameterSource[]::new);

        jdbc.batchUpdate(sql, batchParams);
    }

    public List<AuditEvent> findByTypeSince(String eventType, Instant since) {
        String sql = """
            SELECT id, event_type, payload, created_at
            FROM audit_event
            WHERE event_type = :type AND created_at >= :since
            ORDER BY created_at DESC
            """;

        MapSqlParameterSource params = new MapSqlParameterSource()
                .addValue("type", eventType)
                .addValue("since", since);

        return jdbc.query(sql, params, auditEventRowMapper);
    }

    private final RowMapper<AuditEvent> auditEventRowMapper = new RowMapper<>() {
        @Override
        public AuditEvent mapRow(ResultSet rs, int rowNum) throws SQLException {
            return new AuditEvent(
                    rs.getLong("id"),
                    rs.getString("event_type"),
                    rs.getString("payload"),
                    rs.getTimestamp("created_at").toInstant()
            );
        }
    };

    public record AuditEvent(
            Long id,
            String eventType,
            String payload,
            Instant createdAt
    ) {
    }
}
```

```kotlin
package com.example.jdbc.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
import javax.sql.DataSource

@Configuration
class JdbcConfig {

    @Bean
    fun jdbcTemplate(dataSource: DataSource): JdbcTemplate =
        JdbcTemplate(dataSource).apply {
            fetchSize = 500
        }

    @Bean
    fun namedParameterJdbcTemplate(dataSource: DataSource): NamedParameterJdbcTemplate =
        NamedParameterJdbcTemplate(dataSource)
}
```

```kotlin
package com.example.jdbc.repo

import org.springframework.jdbc.core.RowMapper
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate
import org.springframework.stereotype.Repository
import java.sql.ResultSet
import java.time.Instant

@Repository
class AuditEventJdbcRepository(
    private val jdbc: NamedParameterJdbcTemplate
) {

    fun saveBatch(events: List<AuditEvent>) {
        val sql = """
            INSERT INTO audit_event(event_type, payload, created_at)
            VALUES (:eventType, :payload, :createdAt)
        """.trimIndent()

        val batchParams = events.map {
            MapSqlParameterSource()
                .addValue("eventType", it.eventType)
                .addValue("payload", it.payload)
                .addValue("createdAt", it.createdAt)
        }.toTypedArray()

        jdbc.batchUpdate(sql, batchParams)
    }

    fun findByTypeSince(eventType: String, since: Instant): List<AuditEvent> {
        val sql = """
            SELECT id, event_type, payload, created_at
            FROM audit_event
            WHERE event_type = :type AND created_at >= :since
            ORDER BY created_at DESC
        """.trimIndent()

        val params = MapSqlParameterSource()
            .addValue("type", eventType)
            .addValue("since", since)

        return jdbc.query(sql, params, rowMapper)
    }

    private val rowMapper = RowMapper { rs: ResultSet, _: Int ->
        AuditEvent(
            id = rs.getLong("id"),
            eventType = rs.getString("event_type"),
            payload = rs.getString("payload"),
            createdAt = rs.getTimestamp("created_at").toInstant()
        )
    }

    data class AuditEvent(
        val id: Long?,
        val eventType: String,
        val payload: String,
        val createdAt: Instant
    )
}
```

Такой слой на JDBC прекрасно дополняет JPA там, где нужна высокая производительность, контроль над SQL и работа с «сырыми» таблицами.

---

## Потоковая обработка: курсоры/стримы, контроль памяти, работа с LOB

Когда речь идёт о больших объёмах данных, главное — не пытаться за один запрос вытащить всё в память. JPA по умолчанию загружает результат запроса в коллекцию и кладёт сущности в persistence context, что при сотнях тысяч строк может просто убить приложение. С JDBC ситуация похожая: если вы делаете `queryForList`, драйвер подтянет весь результат. Для по-настоящему больших выборок нужна потоковая обработка: курсоры, `fetchSize`, стримы, поэлементные обработчики.

В JDBC потоковая обработка строится вокруг `ResultSet`. Драйвер может выдавать строки по мере чтения, а не весь набор сразу, если вы настроили `fetchSize` и, для PostgreSQL, включили использование серверных курсоров. В Spring-мире это оборачивается в методы `JdbcTemplate.query`, где вы вместо `RowMapper` можете использовать `RowCallbackHandler` или `ResultSetExtractor` и обрабатывать строки по мере поступления, не копя их в списке. Важно только не выходить за пределы транзакции или жизненного цикла соединения, иначе курсор закроется.

Простой паттерн для потоковой обработки — `RowCallbackHandler`, который вызывается для каждой строки. Вместо того чтобы собирать всё в `List<T>`, вы, например, пишете данные в файл, отправляете в очередь, агрегируете статистику. Так вы контролируете память: в любой момент в heap находится только текущий объект и небольшие структуры агрегации, а не весь набор данных.

Работа с LOB (BLOB/CLOB) особенно критична. Если вы делаете `getBytes()` или `getString()` для гигантской колонки, вы загружаете весь объект в память. Для больших файлов или документов корректнее читать LOB потоком: `getBinaryStream()`/`getCharacterStream()` и передавать этот поток дальше (в файловое хранилище, HTTP-ответ, на диск). Spring предлагает абстракцию `LobHandler`, но в простых случаях можно работать напрямую с `InputStream` и не держать целый файл в памяти.

`fetchSize` — важный инструмент. Для PostgreSQL, если вы используете `setFetchSize(n)` на `Statement`, драйвер будет открывать курсор и забирать по `n` строк за один заход. Это позволяет балансировать между количеством round-trip’ов к серверу и размером памяти, занимаемым буфером. Типичные значения — 100–1000, но подбирать нужно по профилю нагрузки. Важно помнить, что некоторые драйверы игнорируют `fetchSize` при auto-commit и что при большом `fetchSize` транзакция может висеть дольше, блокируя строки.

С точки зрения транзакционности потоковая обработка почти всегда живёт внутри одной транзакции: пока вы читаете курсор, транзакция держится открытой. Это может быть проблемой, если чтение занимает минуты: блокировки и удержание ресурсов на БД. В таких случаях лучше разбивать задачу на «страницы» или использовать keyset-пагинацию: читать по условно «батчам» ID и обрабатывать их независимо, вместо одной огромной транзакции.

В Spring Data JPA есть возможность возвращать `Stream<T>` из репозитория, но под капотом это всё равно ResultSet и курсор. Нужно аккуратно закрывать стрим (try-with-resources или `@Transactional` + явное закрытие), иначе соединение останется открытым. На JDBC это под вашим контролем, поэтому многие предпочитают работать с потоками напрямую и избегать JPA для таких операций.

Для операций экспорта/архивирования часто строят pipeline: читаете данные потоково из БД, конвертируете в нужный формат, сразу же стримите в S3/MinIO/HTTP-ответ. Вы вообще не создаёте промежуточных списков и файлов на диске, вся обработка идёт чанками. Именно JDBC даёт такую гибкость; через JPA это либо сложнее, либо дорого по ресурсам.

Ниже пример: Java-репозиторий, который потоково читает пользователей пачками и пишет их в обработчик, используя `RowCallbackHandler` и `fetchSize`, и аналог на Kotlin. Также пример чтения BLOB через `getBinaryStream` и запись в `OutputStream`.

```java
package com.example.jdbc.stream;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowCallbackHandler;
import org.springframework.stereotype.Repository;

import java.io.OutputStream;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.function.Consumer;

@Repository
public class UserStreamingRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserStreamingRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        this.jdbcTemplate.setFetchSize(500);
    }

    public void streamActiveUsers(Consumer<UserRow> consumer) {
        String sql = """
            SELECT id, email, full_name
            FROM app_user
            WHERE active = true
            """;

        jdbcTemplate.query(sql, new RowCallbackHandler() {
            @Override
            public void processRow(ResultSet rs) throws SQLException {
                UserRow row = new UserRow(
                        rs.getLong("id"),
                        rs.getString("email"),
                        rs.getString("full_name")
                );
                consumer.accept(row);
            }
        });
    }

    public void streamUserAvatar(long userId, OutputStream out) {
        String sql = """
            SELECT avatar
            FROM app_user_profile
            WHERE user_id = ?
            """;

        jdbcTemplate.query(sql, rs -> {
            if (rs.next()) {
                try (var in = rs.getBinaryStream("avatar")) {
                    if (in != null) {
                        in.transferTo(out);
                    }
                }
            }
        }, userId);
    }

    public record UserRow(
            Long id,
            String email,
            String fullName
    ) {
    }
}
```

```kotlin
package com.example.jdbc.stream

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.RowCallbackHandler
import org.springframework.stereotype.Repository
import java.io.OutputStream
import java.sql.ResultSet
import java.util.function.Consumer

@Repository
class UserStreamingRepository(
    private val jdbcTemplate: JdbcTemplate
) {

    init {
        jdbcTemplate.fetchSize = 500
    }

    fun streamActiveUsers(consumer: Consumer<UserRow>) {
        val sql = """
            SELECT id, email, full_name
            FROM app_user
            WHERE active = true
        """.trimIndent()

        jdbcTemplate.query(sql, RowCallbackHandler { rs: ResultSet ->
            val row = UserRow(
                id = rs.getLong("id"),
                email = rs.getString("email"),
                fullName = rs.getString("full_name")
            )
            consumer.accept(row)
        })
    }

    fun streamUserAvatar(userId: Long, out: OutputStream) {
        val sql = """
            SELECT avatar
            FROM app_user_profile
            WHERE user_id = ?
        """.trimIndent()

        jdbcTemplate.query(sql, { rs ->
            if (rs.next()) {
                rs.getBinaryStream("avatar")?.use { input ->
                    input.transferTo(out)
                }
            }
        }, userId)
    }

    data class UserRow(
        val id: Long,
        val email: String,
        val fullName: String
    )
}
```

Такой подход позволяет аккуратно работать с большими объёмами и тяжёлыми LOB, не распухая по памяти и не ломая JPA-контекст.

---

## jOOQ как «типобезопасный SQL»: генерация DSL из схемы, тонкая оптимизация под конкретную СУБД; совместное использование с JPA в одном сервисе

jOOQ — это библиотека, которая делает SQL «первым гражданином» в вашем Java/Kotlin-коде, но при этом даёт типобезопасный DSL вместо строк. В отличие от JPA, jOOQ не пытается скрыть таблицы за сущностями; наоборот, он поднимает таблицы и колонки в виде сгенерированных классов `Tables.MY_TABLE`, `MY_TABLE.ID`, `MY_TABLE.NAME`, а дальше вы строите запросы через fluent-API `dsl.select(...).from(...).where(...)`. Главная идея: «пишите SQL, но с помощью типов, автодополнения и compile-time-проверок».

Основой jOOQ является генерация DSL из схемы БД. На этапе сборки вы подключаете плагин, который подключается к базе (или читает DDL), вытаскивает метаданные и генерирует Java-классы: по каждой таблице, по каждому типу. Для PostgreSQL это означает, что вы получите типобезопасный доступ к JSONB, массивам, enum’ам, диапазонам; для других СУБД — к их специфичным типам. Дальше в коде вы пишете `DSL.using(configuration)` и оперируете этими классами, вместо того чтобы вручную писать имена таблиц/колонок строками.

С точки зрения Spring Boot jOOQ обычно интегрируется через `DSLContext`, который вы инжектите в свои сервисы/репозитории. За транзакции отвечает всё тот же Spring: если вы оборачиваете вызовы jOOQ в `@Transactional`, запросы выполняются в рамках тех же транзакций, что и JPA, при условии, что они разделяют `DataSource`. Это позволяет иметь один сервис, внутри которого сосуществуют и JPA-репозитории, и jOOQ-репозитории, работающие с одной и той же БД.

Большое преимущество jOOQ — поддержка vendor-specific функций и диалектов. Для PostgreSQL вы можете использовать `onConflict()`, `jsonb_extract_path_text`, расширенные операторы, оконные функции, CTE — всё это есть в DSL и будет сгенерировано в корректный SQL-диалект. То, что в JPA превращается в `@Query(nativeQuery = true)` со строкой, в jOOQ становится типобезопасным выражением, которое IDE подсвечивает и проверяет. Это особенно ценно для сложных отчётов и оптимизаций под конкретную СУБД.

jOOQ хорошо ложится на архитектуру «JPA для домена, jOOQ для отчётов/тяжёлых запросов». Типичный паттерн: доменная модель живёт на сущностях и Spring Data JPA, вокруг неё строится бизнес-логика. Рядом есть слой report-/sql-репозиториев на jOOQ, которые делают всё, что требует сложного SQL: отчётность, batch-операции, maintenance-скрипты. Оба слоя делят один `DataSource` и, при необходимости, одну транзакцию, но логически они разделены — это уменьшает смешение концепций.

Есть нюанс по лицензированию и версиям jOOQ, но в техническом плане главное — настроить генерацию. В Gradle-проекте вы добавляете зависимости на `jooq` и на плагин генерации, настраиваете подключение к БД и путь для сгенерированного кода. После этого IDE увидит классы `com.example.jooq.tables.Orders`, `Orders.ORDERS` и т.п., и вы сможете пользоваться полнотой DSL.

Ниже пример: добавляем jOOQ в Spring Boot-проект через Gradle, конфигурируем `DSLContext` и пишем репозиторий, который делает UPSERT (`ON CONFLICT`) и сложный select с агрегатами. Java и Kotlin варианты используют один и тот же подход.

```groovy
// build.gradle (Groovy DSL) — зависимости jOOQ
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-jooq'
    runtimeOnly 'org.postgresql:postgresql'
}
```

```kotlin
// build.gradle.kts (Kotlin DSL)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-jooq")
    runtimeOnly("org.postgresql:postgresql")
}
```

```java
package com.example.jooq.repo;

import com.example.jooq.generated.tables.Orders;
import com.example.jooq.generated.tables.records.OrdersRecord;
import org.jooq.DSLContext;
import org.jooq.impl.DSL;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.List;

@Repository
public class OrderJooqRepository {

    private final DSLContext dsl;
    private static final Orders ORDERS = Orders.ORDERS;

    public OrderJooqRepository(DSLContext dsl) {
        this.dsl = dsl;
    }

    @Transactional
    public void upsertOrder(Long id, BigDecimal amount, String status) {
        dsl.insertInto(ORDERS)
                .set(ORDERS.ID, id)
                .set(ORDERS.AMOUNT, amount)
                .set(ORDERS.STATUS, status)
                .set(ORDERS.UPDATED_AT, OffsetDateTime.now())
                .onConflict(ORDERS.ID)
                .doUpdate()
                .set(ORDERS.AMOUNT, DSL.excluded(ORDERS.AMOUNT))
                .set(ORDERS.STATUS, DSL.excluded(ORDERS.STATUS))
                .set(ORDERS.UPDATED_AT, DSL.excluded(ORDERS.UPDATED_AT))
                .execute();
    }

    @Transactional(readOnly = true)
    public List<CustomerTurnoverRow> findCustomerTurnover() {
        return dsl.select(
                    ORDERS.CUSTOMER_ID,
                    DSL.sum(ORDERS.AMOUNT).as("total_amount"),
                    DSL.count().as("orders_count")
                )
                .from(ORDERS)
                .groupBy(ORDERS.CUSTOMER_ID)
                .orderBy(DSL.sum(ORDERS.AMOUNT).desc())
                .fetch(record -> new CustomerTurnoverRow(
                        record.get(ORDERS.CUSTOMER_ID),
                        record.get("total_amount", BigDecimal.class),
                        record.get("orders_count", Long.class)
                ));
    }

    public record CustomerTurnoverRow(
            Long customerId,
            BigDecimal totalAmount,
            Long ordersCount
    ) {
    }
}
```

```kotlin
package com.example.jooq.repo

import com.example.jooq.generated.tables.Orders
import org.jooq.DSLContext
import org.jooq.impl.DSL
import org.springframework.stereotype.Repository
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.time.OffsetDateTime

@Repository
class OrderJooqRepository(
    private val dsl: DSLContext
) {

    private val ORDERS = Orders.ORDERS

    @Transactional
    fun upsertOrder(id: Long, amount: BigDecimal, status: String) {
        dsl.insertInto(ORDERS)
            .set(ORDERS.ID, id)
            .set(ORDERS.AMOUNT, amount)
            .set(ORDERS.STATUS, status)
            .set(ORDERS.UPDATED_AT, OffsetDateTime.now())
            .onConflict(ORDERS.ID)
            .doUpdate()
            .set(ORDERS.AMOUNT, DSL.excluded(ORDERS.AMOUNT))
            .set(ORDERS.STATUS, DSL.excluded(ORDERS.STATUS))
            .set(ORDERS.UPDATED_AT, DSL.excluded(ORDERS.UPDATED_AT))
            .execute()
    }

    @Transactional(readOnly = true)
    fun findCustomerTurnover(): List<CustomerTurnoverRow> {
        return dsl.select(
            ORDERS.CUSTOMER_ID,
            DSL.sum(ORDERS.AMOUNT).`as`("total_amount"),
            DSL.count().`as`("orders_count")
        )
            .from(ORDERS)
            .groupBy(ORDERS.CUSTOMER_ID)
            .orderBy(DSL.sum(ORDERS.AMOUNT).desc())
            .fetch { record ->
                CustomerTurnoverRow(
                    customerId = record.get(ORDERS.CUSTOMER_ID),
                    totalAmount = record.get("total_amount", BigDecimal::class.java),
                    ordersCount = record.get("orders_count", Long::class.java)
                )
            }
    }

    data class CustomerTurnoverRow(
        val customerId: Long?,
        val totalAmount: BigDecimal?,
        val ordersCount: Long?
    )
}
```

Совместное использование JPA и jOOQ требует дисциплины: не смешивать сущности и jOOQ-record’ы в одном уровне, понимать, где у вас доменная модель, а где — SQL-модель. Но при правильной организации вы получаете лучшее из двух миров: удобный CRUD и навигацию по графу сущностей через JPA и при этом полный контроль над сложным SQL, апсертами, CTE и отчётами через jOOQ.

# 13. Репозитории: расширение и кастомизация

## Свой базовый интерфейс репозитория: общие методы, хук на `EntityManager`

Во всех сколько-нибудь крупных проектах очень быстро появляются одни и те же паттерны в репозиториях: «найти по id и бросить своё исключение, если нет», «пометить как удалённый вместо физического удаления», «сделать постраничный поиск с одинаковой сортировкой и фильтрами по тенанту». Если эти подходы размазывать по десяткам интерфейсов `JpaRepository`, код начинает дублироваться, а правки превращаются в квест «поправь одно и то же в 15 местах». Собственный базовый интерфейс репозитория как раз и нужен, чтобы собрать эти общие вещи в одном месте и навязать единые подходы всей команде.

В Spring Data JPA базовый репозиторий реализуется двумя составляющими: интерфейсом, который расширяет `JpaRepository<T, ID>` (и, при необходимости, ещё что-то вроде `JpaSpecificationExecutor<T>`), и базовой реализацией, которая наследуется от `SimpleJpaRepository<T, ID>` и получает доступ к `EntityManager`. В интерфейсе вы объявляете новые «общие» методы (`findRequired`, `softDelete`, `existsOrThrow` и т.п.), а в реализации — задаёте их поведение единообразно для всех сущностей. Дальше все ваши конкретные репозитории (`CustomerRepository`, `OrderRepository`) расширяют именно этот базовый интерфейс.

Доступ к `EntityManager` в базовой реализации даёт вам возможность делать вещи, которых нет в стандартном `JpaRepository`: писать общий `findRequired` с правильным логированием, работать напрямую с `Criteria` или `Query` для каких-то узких кейсов, реализовывать soft-delete на уровне базового класса (`em.remove` заменяется на обновление флага). При этом важно помнить, что базовый репозиторий должен оставаться максимально универсальным: никакой доменной логики, только инфраструктура и паттерны доступа к данным.

Типичный набор методов в таком базовом репозитории — это вариации на тему `findByIdOrThrow`, «безопасное» удаление, общие хелперы для пагинации и, иногда, методы поддержки многоарендности (например, автоматический фильтр по `tenantId` при поиске). Главное — не скатиться в то, что базовый репозиторий начинает знать о полях конкретной сущности. Как только внутри реализации появляются обращения к `SomeEntity.class` или конкретным колонкам, база превращается в свалку. Если нужен общий функционал, опирайтесь либо на интерфейсы-маркерные (`HasTenant`, `SoftDeletable`), либо на generics-ограничения с абстрактным суперклассом.

С точки зрения транзакций базовый репозиторий остаётся обычным Spring-компонентом: его методы могут быть помечены `@Transactional` или полагаться на аннотации сервисного слоя. Обычно хорошая практика — в базовом репозитории делать только `readOnly = true` для общих чтений и оставлять управление записью (`@Transactional`) на сервисы. Исключение — очень низкоуровневые операции, которые явно должны быть атомарными и не зависят от внешних сервисов (например, массовые обновления одной таблицы).

Подключение базовой реализации к Spring Data JPA делается через параметр `repositoryBaseClass` в `@EnableJpaRepositories`. Вы указываете свой класс, и фреймворк будет генерировать прокси для ваших репозиториев, используя его в качестве базового. Важно, чтобы интерфейсы репозиториев расширяли ваш базовый интерфейс, иначе всё вернётся к дефолтной `SimpleJpaRepository`. Именно поэтому стоит сразу задать единый паттерн: «у нас все репозитории наследуются от `BaseRepository<T, ID>`».

С практической точки зрения базовый репозиторий — это место, где вы можете централизовано навязать подходы к логированию и обработке ошибок. Например, `findRequiredById` может логировать запрос в debug с указанием класса сущности и ключа, а при отсутствии — кидать доменное `NotFoundException`, которое потом централизованно маппится в 404. Это лучше, чем каждый раз вручную делать `orElseThrow` с разными сообщениями и типами исключений.

Ещё одно удобство — возможность добавлять туда диагностические хелперы: измерение времени выполнения запроса, дополнительные проверки, детальную ошибку при нарушении уникальности. Да, часть этого решается на уровне AOP/`@RepositoryAdvice`, но иногда проще иметь маленький набор методов в репозитории, которые точно ведут себя одинаково, чем пытаться оборачивать всё подряд.

В существующем проекте базовый репозиторий можно вводить постепенно. Сначала вы создаёте интерфейс и реализацию, включаете их через `@EnableJpaRepositories`, затем переводите один-два репозитория, проверяете, что всё работает, и только потом массово меняете наследование. При этом не обязательно сразу переносить весь старый хелпер-код вовнутрь: можно начать с пары самых важных методов (например, `findRequired`), а остальное допиливать по мере необходимости.

Важно, чтобы команда договорилась, что доменная логика в базовый репозиторий не попадает. Любые правила предметной области — в сервисах. Базовый репозиторий — это инфраструктурный слой, задача которого облегчить работу с JPA и навязать общие практики. Если туда начать добавлять «магические» методы вроде `deactivateCustomerWithAllRelations`, через пару лет код станет нечитаемым, а архитектура — хрупкой.

Ниже пример: базовый интерфейс и реализация в Java и Kotlin, плюс конфигурация `@EnableJpaRepositories` и простой `CustomerRepository`, который наследует общий функционал.

```java
package com.example.jpa.base;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.support.SimpleJpaRepository;
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.stereotype.Repository;

import java.io.Serializable;
import java.util.Optional;

@NoRepositoryBean
public interface BaseRepository<T, ID extends Serializable> extends JpaRepository<T, ID> {

    T findRequired(ID id);

    Optional<T> findOptional(ID id);
}
```

```java
package com.example.jpa.base;

import jakarta.persistence.EntityManager;
import org.springframework.data.jpa.repository.support.SimpleJpaRepository;

import java.io.Serializable;
import java.util.Optional;

public class BaseRepositoryImpl<T, ID extends Serializable>
        extends SimpleJpaRepository<T, ID>
        implements BaseRepository<T, ID> {

    private final EntityManager entityManager;
    private final Class<T> domainClass;

    public BaseRepositoryImpl(Class<T> domainClass, EntityManager entityManager) {
        super(domainClass, entityManager);
        this.entityManager = entityManager;
        this.domainClass = domainClass;
    }

    @Override
    public T findRequired(ID id) {
        return findById(id)
                .orElseThrow(() -> new EntityNotFoundException(
                        domainClass.getSimpleName(), String.valueOf(id)
                ));
    }

    @Override
    public Optional<T> findOptional(ID id) {
        return findById(id);
    }
}
```

```java
package com.example.jpa.base;

public class EntityNotFoundException extends RuntimeException {

    public EntityNotFoundException(String entityName, String id) {
        super("Entity '%s' with id '%s' not found".formatted(entityName, id));
    }
}
```

```java
package com.example.jpa.config;

import com.example.jpa.base.BaseRepositoryImpl;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EnableJpaRepositories(
        basePackages = "com.example.jpa.repository",
        repositoryBaseClass = BaseRepositoryImpl.class
)
public class JpaConfig {
}
```

```java
package com.example.jpa.customer;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "customer")
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private String email;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt = Instant.now();

    // getters/setters
}
```

```java
package com.example.jpa.customer;

import com.example.jpa.base.BaseRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CustomerRepository extends BaseRepository<Customer, Long> {
}
```

```kotlin
package com.example.jpa.base

import jakarta.persistence.EntityManager
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.support.SimpleJpaRepository
import org.springframework.data.repository.NoRepositoryBean
import java.io.Serializable
import java.util.Optional

@NoRepositoryBean
interface BaseRepository<T, ID : Serializable> : JpaRepository<T, ID> {

    fun findRequired(id: ID): T

    fun findOptional(id: ID): Optional<T>
}

class BaseRepositoryImpl<T, ID>(
    private val domainClass: Class<T>,
    private val em: EntityManager
) : SimpleJpaRepository<T, ID>(domainClass, em),
    BaseRepository<T, ID> where ID : Serializable {

    override fun findRequired(id: ID): T =
        findById(id).orElseThrow {
            EntityNotFoundException(domainClass.simpleName, id.toString())
        }

    override fun findOptional(id: ID): Optional<T> =
        findById(id)
}

class EntityNotFoundException(
    entityName: String,
    id: String
) : RuntimeException("Entity '$entityName' with id '$id' not found")
```

```kotlin
package com.example.jpa.config

import com.example.jpa.base.BaseRepositoryImpl
import org.springframework.context.annotation.Configuration
import org.springframework.data.jpa.repository.config.EnableJpaRepositories

@Configuration
@EnableJpaRepositories(
    basePackages = ["com.example.jpa.repository"],
    repositoryBaseClass = BaseRepositoryImpl::class
)
class JpaConfig
```

```kotlin
package com.example.jpa.customer

import jakarta.persistence.*
import java.time.Instant

@Entity
@Table(name = "customer")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var email: String = "",

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now()
)
```

```kotlin
package com.example.jpa.customer

import com.example.jpa.base.BaseRepository
import org.springframework.stereotype.Repository

@Repository
interface CustomerRepository : BaseRepository<Customer, Long>
```

---

## Кастомные имплементации для отдельных репозиториев (surgical-SQL, спец-кеши)

Помимо базового репозитория, который расширяет функциональность всех репозиториев сразу, Spring Data JPA позволяет настраивать кастомные реализации для конкретных репозиториев. Это как «точечная хирургия»: у вас есть 1–2 метода, которые требуют особого подхода — сложный SQL, кэширование, комбинированный доступ JPA+JDBC, работа с jOOQ — и вы не хотите тащить всю эту специфику в базовый класс. Тогда вы создаёте отдельный интерфейс `XxxRepositoryCustom` и его реализацию `XxxRepositoryImpl`, которые Spring «подмешивает» к основному репозиторию.

Сценарии, где кастомная имплементация особенно полезна, довольно типичны. Например, вам нужно сделать один очень тяжёлый отчётный запрос по заказам с денормализованными данными, и явно виден SQL, который вы хотите написать руками. Остальные методы `OrderRepository` спокойно живут на стандартном `JpaRepository`. Логично не ломать модель ради одного кейса и не городить отдельный репозиторий «для отчётов», а расширить существующий репозиторий кастомной реализацией с нужным SQL.

Другой пример — специальное кэширование результата. Допустим, вы хотите кэшировать только один-два тяжёлых метода репозитория, оборачивая данные в локальный кэш или Redis, но не хотите включать `@Cacheable` на уровне сервиса (чтобы не смешивать там доменную логику и инфраструктуру). Тогда в кастомной реализации вы можете явно использовать `CacheManager`, управлять ключами и TTL, не влияя на остальные методы репозитория. Главное — помнить, что репозиторий остаётся частью слоя доступа к данным, и кеш здесь — инфраструктурная оптимизация, а не логика предметной области.

Технически паттерн выглядит так: вы объявляете интерфейс `UserRepositoryCustom` с методами, которых нет в стандартном `JpaRepository`, например, `findForExport(...)`. Затем создаёте класс `UserRepositoryImpl`, который реализует этот интерфейс, помечаете его `@Repository` (необязательно) и инжектите `EntityManager`, `JdbcTemplate` или `DSLContext`. Интерфейс основного репозитория `UserRepository` расширяет и `JpaRepository`, и `UserRepositoryCustom`. Spring Data по соглашению (`Impl` в названии) автоматически найдёт реализацию и сгенерирует прокси, который объединит стандартные методы и кастомные.

Кастомные реализации — хорошее место для «surgical SQL»: точечных native-запросов, которые явно заточены под конкретную СУБД и таблицы. Вы можете написать аккуратный SQL с оконными функциями, CTE, `ON CONFLICT`, использовать `JdbcTemplate` или jOOQ, а снаружи оставить чистый репозиторийный интерфейс. Это даёт хорошую изоляцию: сервисы ничего не знают о том, SQL там или JPQL, и вам проще менять реализацию, если интеграционные требования изменятся.

Важно не злоупотреблять кастомными репозиториями. Если каждый второй репозиторий имеет по кастомной имплементации с бизнес-логикой, это признак того, что слой сервисов слишком тонкий, а репозитории превратились в «супер-сервисы». Правильнее использовать кастомные реализации строго для задач доступа к данным: нетривиальные запросы, оптимизации производительности, интеграции с кэшем. Всё, что требует оркестрации нескольких репозиториев или внешних сервисов, должно жить в сервисном слое.

Транзакции для кастомных методов работают так же, как и для обычных: вы можете пометить интерфейс репозитория `@Transactional(readOnly = true)` и переопределить конкретные методы с `@Transactional` для записи. В кастомной реализации аннотации можно ставить прямо на методы, если вам нужно изменить режим по сравнению с дефолтным. При этом следует помнить, что методы стандартного `JpaRepository` и кастомные методы реализуются разными классами, но прокси Spring Data объединяет их транзакционную семантику.

Хорошая практика — держать сигнатуру кастомных методов «узкой»: принимать чёткий фильтр (DTO) или конкретные параметры, возвращать либо доменные сущности, либо простые DTO/проекции. Старайтесь не протаскивать внутрь методы сервисного слоя или другие репозитории, чтобы не перепутать уровни. Если кастомная реализация начинает вызывать сторонние сервисы, это тревожный сигнал: вероятно, её часть должна быть вынесена наверх.

С точки зрения тестирования кастомные репозитории можно проверять интеграционными тестами с Testcontainers (реальная СУБД и миграции), чтобы убедиться, что SQL корректен и использует нужные индексы. При этом вы тестируете только слой данных, без поднятия всего приложения. Для кэширования можно дополнительно проверять, что повторные вызовы не бьют по базе, а при инвалидации — SQL действительно выполняется заново.

Ниже пример: кастомный репозиторий для пользователей, где один из методов реализован через native SQL и `JdbcTemplate`. Снаружи `UserRepository` остаётся обычным Spring Data-интерфейсом, но получает дополнительный метод `findForExport`. Java и Kotlin варианты эквивалентны.

```java
package com.example.jpa.user;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "app_user")
public class User {

    @Id
    @GeneratedValue
    private Long id;

    private String email;

    private String fullName;

    @Column(name = "active", nullable = false)
    private boolean active = true;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt = Instant.now();

    // getters/setters
}
```

```java
package com.example.jpa.user;

import java.time.Instant;
import java.util.List;

public interface UserRepositoryCustom {

    List<UserForExport> findForExport(Instant createdFrom, boolean onlyActive);

    record UserForExport(
            Long id,
            String email,
            String fullName,
            Instant createdAt
    ) {
    }
}
```

```java
package com.example.jpa.user;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.Instant;
import java.util.List;

public class UserRepositoryImpl implements UserRepositoryCustom {

    private final JdbcTemplate jdbcTemplate;

    public UserRepositoryImpl(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public List<UserForExport> findForExport(Instant createdFrom, boolean onlyActive) {
        String sql = """
            SELECT id, email, full_name, created_at
            FROM app_user
            WHERE created_at >= ?
              AND (:active = FALSE OR active = TRUE)
            ORDER BY created_at ASC
            """;

        return jdbcTemplate.query(
                sql,
                (rs, rowNum) -> mapRow(rs),
                createdFrom,
                onlyActive
        );
    }

    private UserForExport mapRow(ResultSet rs) throws SQLException {
        return new UserForExport(
                rs.getLong("id"),
                rs.getString("email"),
                rs.getString("full_name"),
                rs.getTimestamp("created_at").toInstant()
        );
    }
}
```

```java
package com.example.jpa.user;

import com.example.jpa.base.BaseRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends BaseRepository<User, Long>, UserRepositoryCustom {
}
```

```kotlin
package com.example.jpa.user

import jakarta.persistence.*
import java.time.Instant

@Entity
@Table(name = "app_user")
class User(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var email: String = "",

    @Column(name = "full_name")
    var fullName: String = "",

    @Column(nullable = false)
    var active: Boolean = true,

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now()
)
```

```kotlin
package com.example.jpa.user

import java.time.Instant

interface UserRepositoryCustom {

    fun findForExport(createdFrom: Instant, onlyActive: Boolean): List<UserForExport>

    data class UserForExport(
        val id: Long,
        val email: String,
        val fullName: String,
        val createdAt: Instant
    )
}
```

```kotlin
package com.example.jpa.user

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.jdbc.core.RowMapper
import java.sql.ResultSet
import java.time.Instant

class UserRepositoryImpl(
    private val jdbcTemplate: JdbcTemplate
) : UserRepositoryCustom {

    override fun findForExport(createdFrom: Instant, onlyActive: Boolean): List<UserRepositoryCustom.UserForExport> {
        val sql = """
            SELECT id, email, full_name, created_at
            FROM app_user
            WHERE created_at >= ?
              AND (? = FALSE OR active = TRUE)
            ORDER BY created_at ASC
        """.trimIndent()

        return jdbcTemplate.query(sql, rowMapper, createdFrom, onlyActive)
    }

    private val rowMapper = RowMapper { rs: ResultSet, _: Int ->
        UserRepositoryCustom.UserForExport(
            id = rs.getLong("id"),
            email = rs.getString("email"),
            fullName = rs.getString("full_name"),
            createdAt = rs.getTimestamp("created_at").toInstant()
        )
    }
}
```

```kotlin
package com.example.jpa.user

import com.example.jpa.base.BaseRepository
import org.springframework.stereotype.Repository

@Repository
interface UserRepository :
    BaseRepository<User, Long>,
    UserRepositoryCustom
```

Такой подход позволяет не засорять базовый репозиторий и в то же время даёт точечные расширения там, где это действительно нужно.

---

## «Граница ответственности» репозитория: тонкие методы vs «толстые» сервисы — баланс читаемости и производительности

Граница ответственности между репозиторием и сервисом — один из ключевых архитектурных моментов. Репозиторий — это абстракция над хранилищем данных: он знает, как читать и записывать сущности, но не знает, зачем это делается и какие бизнес-правила за этим стоят. Сервис — это слой, где реализуются сценарии предметной области: последовательность вызовов репозиториев, валидация, взаимодействие с внешними системами. Если эту границу размыть, код быстро превращается в кашу: часть бизнес-логики уехала в репозитории, часть осталась в сервисах, а часть размазалась по контроллерам.

В идеальной картинке «тонкий» репозиторий — это набор относительно простых методов доступа к данным: `findById`, `save`, несколько осмысленных `findBy...`, возможно, пара сложных выборок, но без сценариев и оркестраций. Вся логика «если клиент заблокирован — не создавать заказ», «после обновления статуса — отправить уведомление и логировать аудиторное событие» должна жить в сервисе. Это упрощает тестирование (репозитории тестируются на уровне БД, сервисы — на уровне сценариев), делает архитектуру прозрачной и упрощает рефакторинг.

Однако реальность подкидывает компромиссы. Бывают случаи, когда один метод репозитория должен сделать довольно сложную выборку или использует специфичный для СУБД SQL. С точки зрения производительности правильнее сделать один сложный запрос, чем три простых. И вот тут легко начать «переезжать» бизнес-логику в репозиторий: «ну давайте сразу отфильтруем только активных клиентов, у которых есть незакрытые заказы и которые подходят под какие-то условия». Если эти условия — чисто технические и не меняются от сценария к сценарию, это нормально. Если же в разных бизнес-операциях условия разные, такую фильтрацию лучше всё-таки оставить сервису.

Один из признаков того, что репозиторий стал слишком «толстым» — появление методов, имена которых описывают целые бизнес-сценарии: `approveOrderAndSendNotification`, `createCustomerWithDefaultContractAndLimit`. Это не про доступ к данным, это уже чистый домен и оркестрация. Такие методы логично переносить в сервисы, оставляя в репозитории только операции «прочитать сущность», «сохранить», «найти по фильтру». В противном случае вы получаете слой, который одновременно и ходит в БД, и знает бизнес, и шлёт события в Kafka — а значит, его очень сложно менять без риска поломать полсистемы.

С другой стороны, слишком «тонкий» репозиторий с безумным количеством методов вида `findBy...` под каждый сценарий тоже не радует. Когда в одном интерфейсе десятки методов с практически одинаковыми структурами фильтрации, а сервисы используют каждый свой набор, читаемость падает. Лучше в таких случаях сделать пару более универсальных методов (например, `searchByFilter(FilterDto filter)`) с динамическими спецификациями или QueryDSL, чем гнать на каждый сценарий свой метод.

Баланс в итоге сводится к простому правилу: репозиторий отвечает за «как» достать данные, но не за «зачем». Он может скрывать сложность SQL, агрегаций, индексов, но не должен принимать решения о бизнес-смыслах этих запросов. Если метод репозитория удобно описать как «получи заказы с такими-то параметрами», это нормально. Если он описывается как «подготовь заказ к списанию и валидации лимита», это уже сервис.

Ещё один важный аспект — взаимодействие с несколькими репозиториями. Как только операция требует работы с несколькими агрегатами (клиент + заказы + счета), её место автоматически — в сервисном слое. Репозиторий работает в границах одной aggregate root (иногда чуть больше, но не оркестрирует несколько независимых подсистем). Если вы видите внутри репозитория вызов другого репозитория, почти наверняка это архитектурный запах.

Производительность иногда подталкивает к тому, чтобы «приспустить» границу: например, сделать один метод репозитория, который сразу возвращает DTO с данными из нескольких таблиц для API-эндпоинта. Это нормально, если вы чётко отделяете такой «read-model» от доменной модели: метод возвращает специальный DTO, не мутирует сущности и не включает бизнес-решения. Тогда сервис остаётся владельцем сценария, а репозиторий — владельцем доступа к данным, пусть и чуть более сложного.

С организационной точки зрения полезно отражать границу ответственности в пакетовке и именовании. Репозитории лежат в своём слое (`.repository`), сервисы — в своём (`.service`), контроллеры — отдельно. На уровне статического анализа можно использовать ArchUnit или подобные инструменты, чтобы не допускать, например, вызовов контроллеров из репозиториев или зависимостей сервисов от конкретных `EntityManager`. Это дисциплинирует и не даёт архитектуре «уползти» за пару месяцев активной разработки.

Ниже пример: сначала «плохой» вариант, где репозиторий берёт на себя бизнес-ответственность, а затем более здоровый вариант с тонким репозиторием и сервисом, который делает оркестрацию. Java и Kotlin варианты показывают одну и ту же идею.

```java
package com.example.jpa.order;

import com.example.jpa.customer.Customer;
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Entity
@Table(name = "customer")
class Customer {

    @Id
    @GeneratedValue
    private Long id;

    private boolean active;

    private String email;

    // getters/setters
}

@Entity
@Table(name = "orders")
class Order {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;

    private String status;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt = Instant.now();

    // getters/setters
}

@Repository
interface OrderRepositoryBad extends JpaRepository<Order, Long> {

    @Transactional
    default Order approveOrderAndNotify(Order order, NotificationService notificationService) {
        if (!order.getCustomer().isActive()) {
            throw new IllegalStateException("Customer is not active");
        }
        order.setStatus("APPROVED");
        Order saved = save(order);
        notificationService.sendOrderApproved(saved);
        return saved;
    }
}

interface NotificationService {
    void sendOrderApproved(Order order);
}
```

```java
package com.example.jpa.order;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByCustomerIdAndStatus(Long customerId, String status);
}
```

```java
package com.example.jpa.order;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    public OrderService(
            OrderRepository orderRepository,
            NotificationService notificationService
    ) {
        this.orderRepository = orderRepository;
        this.notificationService = notificationService;
    }

    @Transactional
    public Order approveOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("Order not found"));

        if (!order.getCustomer().isActive()) {
            throw new IllegalStateException("Customer is not active");
        }

        order.setStatus("APPROVED");
        Order saved = orderRepository.save(order);
        notificationService.sendOrderApproved(saved);
        return saved;
    }
}
```

```kotlin
package com.example.jpa.order

import jakarta.persistence.*
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.Instant

@Entity
@Table(name = "customer")
class Customer(

    @Id
    @GeneratedValue
    var id: Long? = null,

    var active: Boolean = true,

    var email: String = ""
)

@Entity
@Table(name = "orders")
class Order(

    @Id
    @GeneratedValue
    var id: Long? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    var customer: Customer? = null,

    var status: String = "NEW",

    @Column(name = "created_at", nullable = false)
    var createdAt: Instant = Instant.now()
)

interface NotificationService {
    fun sendOrderApproved(order: Order)
}

@Repository
interface OrderRepository : JpaRepository<Order, Long> {

    fun findByCustomerIdAndStatus(customerId: Long, status: String): List<Order>
}

@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val notificationService: NotificationService
) {

    @Transactional
    fun approveOrder(orderId: Long): Order {
        val order = orderRepository.findById(orderId)
            .orElseThrow { IllegalArgumentException("Order not found") }

        val customer = order.customer
            ?: throw IllegalStateException("Order has no customer")

        if (!customer.active) {
            throw IllegalStateException("Customer is not active")
        }

        order.status = "APPROVED"
        val saved = orderRepository.save(order)
        notificationService.sendOrderApproved(saved)
        return saved
    }
}
```

В «плохом» варианте репозиторий знает о `NotificationService`, реализует бизнес-правила и занимается оркестрацией — это нарушает границы и усложняет сопровождение. В «хорошем» варианте репозиторий остаётся тонким, отвечает за доступ к данным, а сервис целиком берёт на себя сценарий «одобрить заказ и отправить уведомление». Такой баланс даёт и читабельность, и предсказуемую производительность: сложные запросы — в репозиториях, бизнес — в сервисах.


# 14. Тестирование продвинутого persistence

Тесты persistence-слоя — это не просто «проверить, что `save` что-то сохраняет». На проде вас интересуют совсем другие вещи: что миграции накатываются, что запросы не бьют в N+1, что уникальные и внешние ключи реально защищают данные, что при дедлоках и таймаутах код ведёт себя предсказуемо. Всё это невозможно проверить на моках или в in-memory H2 с «примерно похожей» схемой — нужен почти настоящий стенд, только в масштабе тестов.

Хорошая стратегия тестирования persistence-слоя опирается на три столпа. Во-первых, окружение: реальная СУБД в Testcontainers, миграции, чёткая схема. Во-вторых, сценарии: отдельные тесты на производительность запросов (N+1, пагинация, графы загрузки), на инварианты БД (уникальные, `check`, FK), на поведение под нагрузкой и блокировками. В-третьих, метрики и диагностика: сбор статистики Hibernate, логов SQL и понятные ассерты, чтобы тесты ловили регрессии, а не случайность.

---

## Testcontainers с реальной СУБД, прогон миграций перед тестами, сидирование данных

Первый принцип — тестируем на той же СУБД, на которой живём в проде. Если в проде PostgreSQL, а в тестах H2 «в режиме совместимости», вы сами себе стреляете в ногу: другой планировщик запросов, другие типы, другие ограничения, другая работа индексов. Testcontainers как раз решает проблему: вы поднимаете реальный Postgres в Docker-контейнере рядом с тестами и гоняете на нём тот же DDL/DML, что и на бою (через Flyway/Liquibase).

Spring Boot хорошо дружит с Testcontainers: вы поднимаете `PostgreSQLContainer` как статическое поле, а через `@DynamicPropertySource` прокидываете в контекст URL, логин, пароль. Дальше всё работает как обычно: `spring-boot-starter-data-jpa` поднимает `DataSource`, `EntityManagerFactory`, а Flyway/Liquibase при старте гонит миграции. В тестах вы получаете схему, созданную теми же скриптами, что и в CI/prod.

Миграции должны применяться перед любым тестом, который работает с БД. В типичном Spring Boot-проекте это делается автоматически: при старте контекста Boot запускает мигратор (Flyway/Liquibase) и падает, если что-то пошло не так. Важно не «облегчать» схему для тестов: если вы уберёте часть ограничений «чтобы проще было тестировать», тесты перестанут ловить реальные проблемы. Лучше сложнее сидировать данные, чем жить с вырезанными check/unique/FK.

Сидирование данных можно делать по-разному. Самый простой путь — использовать Spring-репозитории в `@BeforeEach`/`@BeforeAll`: создаёте сущности обычным Java/Kotlin-кодом, сохраняете их через JPA, тесты работают с ними. Плюс: типобезопасность, меньше дублирования схемы, часто проще рефакторить. Минус: вы зависите от JPA-модели для подготовленных данных, иногда трудно смоделировать «грязные» кейсы, которые через ORM не создашь.

Альтернатива — чистый SQL через `JdbcTemplate` или `@Sql`-скрипты. Это даёт полный контроль: вы можете создать строки с нарушением инвариантов, проверить поведение триггеров, подготовить «грязные» данные для миграций. Плата — сложнее поддерживать синхронизацию схемы и тестовых SQL: меняется колонка — нужно не забыть поправить скрипты. Для критичных сценариев (миграции, audit, tricky-констрейнты) это оправдано.

Ещё одна важная тема — очистка БД между тестами. Вариант «прогонять миграции заново перед каждым тестом» слишком медленный. Обычно делают так: один контейнер на весь test-suite, миграции — один раз при старте, а дальше либо откаты транзакций вокруг каждого теста (`@Transactional` над тестовым классом), либо TRUNCATE/DELETE нужных таблиц в `@AfterEach`. Первый вариант проще для JPA, второй полезен, когда вы используете полусторонние вещи (bulk SQL, jOOQ).

Testcontainers по умолчанию поднимает новый контейнер на каждый запуск тестового класса. Чтобы ускорить тесты, контейнер делают `static` и используют reusable режим. Это позволяет делить один Postgres на весь набор тестов, но важно следить за изоляцией данных — именно поэтому откат транзакций и/или очищающие SQL обязательны. На CI часто используют «один контейнер на весь job» и параллельный запуск тестов на одном экземпляре БД.

Безопасность: никогда не подключайтесь к «настоящему» тестовому или dev-кластерам из unit/integration тестов. Все параметры подключения должны приходить от Testcontainers. Хорошая практика — явно задавать имя БД, пользователя и префиксы контейнера, чтобы даже при ошибке вы не попали в живую базу. Любые «временные» конфиги с подключением к shared-test-БД быстро превращаются в источник флаки-тестов и случайной порчи данных.

Наконец, не забывайте про профили. Для тестов удобно иметь профиль `test`, в котором включены нужные настройки: подробный лог SQL, включённые статистики Hibernate, tuned-down таймауты, отдельные имена схем. Testcontainers даёт вам БД, миграции создают схему, а профиль test — делает это окружение удобным для диагностики.

Ниже пример конфигурации зависимостей и простого интеграционного теста с Testcontainers и Flyway. Java и Kotlin варианты эквивалентны.

```groovy
// build.gradle (Groovy DSL)
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:postgresql'
}
```

```kotlin
// build.gradle.kts (Kotlin DSL)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")
}
```

```java
package com.example.jpa.test;

import com.example.jpa.customer.Customer;
import com.example.jpa.customer.CustomerRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestInstance;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import jakarta.annotation.Resource;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class CustomerRepositoryTest {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("testdb")
                    .withUsername("test")
                    .withPassword("test");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "none");
        registry.add("spring.flyway.enabled", () -> "true");
    }

    @Resource
    private CustomerRepository customerRepository;

    @Test
    void shouldApplyMigrationsAndPersistCustomer() {
        Customer customer = new Customer();
        customer.setEmail("test@example.com");

        Customer saved = customerRepository.save(customer);

        assertThat(saved.getId()).isNotNull();
        var found = customerRepository.findById(saved.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getEmail()).isEqualTo("test@example.com");
    }
}
```

```kotlin
package com.example.jpa.test

import com.example.jpa.customer.Customer
import com.example.jpa.customer.CustomerRepository
import jakarta.annotation.Resource
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.TestInstance
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class CustomerRepositoryTest {

    companion object {
        @Container
        @JvmStatic
        val postgres: PostgreSQLContainer<*> =
            PostgreSQLContainer("postgres:16-alpine")
                .withDatabaseName("testdb")
                .withUsername("test")
                .withPassword("test")

        @JvmStatic
        @DynamicPropertySource
        fun configureDatasource(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.jpa.hibernate.ddl-auto") { "none" }
            registry.add("spring.flyway.enabled") { "true" }
        }
    }

    @Resource
    lateinit var customerRepository: CustomerRepository

    @Test
    fun `should apply migrations and persist customer`() {
        val customer = Customer().apply {
            email = "test@example.com"
        }

        val saved = customerRepository.save(customer)

        assertThat(saved.id).isNotNull()
        val found = customerRepository.findById(saved.id!!)
        assertThat(found).isPresent
        assertThat(found.get().email).isEqualTo("test@example.com")
    }
}
```

---

## Проверки на N+1/пагинацию/графы загрузки; сбор статистики Hibernate в тестах

Проблема N+1 — классика: вы делаете один запрос за списком сущностей, а потом JPA лениво догружает коллекции/ссылки ещё N запросами. На небольших объёмах это незаметно, на проде — превращается в десятки/сотни лишних round-trip’ов и падение p95/p99. Это чисто persistence-проблема, поэтому логично ловить её именно тестами уровня репозиториев/сервисов, а не ждать профилировщика на проде.

Самый грубый способ диагностики — включить логирование SQL и глазами посмотреть, сколько запросов выполняется для одного сценария. Для разовой отладки работает, но в виде автоматического теста — слабо: сложно считать, легко «проморгать» регресс. Более надёжный подход — использовать статистику Hibernate (`SessionFactory.getStatistics()`), которая считает количество запросов, хитов кэша, загрузок сущностей и т.п., и на неё уже вешать ассерты.

Чтобы статистика Hibernate была доступна, нужно включить её в конфигурации тестового профиля: `spring.jpa.properties.hibernate.generate_statistics=true`. В тесте вы можете через `EntityManagerFactory` достать `SessionFactory`, обнулить статистику перед сценарием, выполнить код и проверить счётчики `getPrepareStatementCount()`, `getEntityLoadCount()`, `getCollectionFetchCount()` и т.д. Главное — не превращать тесты в хрупкие проверки «строго 3 запроса», лучше проверять разумный верхний предел.

Отдельная тема — тесты на пагинацию. Тут важно две вещи: корректность данных (страницы без дырок и дублей, стабильный порядок) и отсутствие лишних запросов. Часто при сложных связях и `JOIN FETCH` с коллекциями пагинация начинает вести себя странно: Hibernate делает подзапрос, а затем второй запрос на fetch, из-за чего появляются дубли и дополнительные запросы. В тестах логично проверять, что `Page<T>` возвращает правильный `totalElements`, а SQL остается в пределах разумного.

Графы загрузки (`@EntityGraph`) и `JOIN FETCH` тоже стоит покрывать тестами. Идея простая: вы вызываете метод репозитория, который должен загрузить сущность с заранее определённым набором ассоциаций, а затем в тесте обращаетесь к этим ассоциациям вне транзакции. Если всё работает без `LazyInitializationException` и при этом количество запросов не раздуто, значит граф настроен разумно. Такие тесты хорошо документируют ожидания к модели загрузки.

Для сложных сценариев иногда полезно проверять не только количество запросов, но и сами SQL — например, что используется нужный `JOIN`, нужный индекс и т.п. Но это уже сильно привязывает тесты к конкретной версии Hibernate/SQL-диалекта, поэтому стоит быть аккуратным: такие проверки оправданы только для критичных кусочков, где любое изменение плана — риск SLA.

Сбор статистики особенно ценен как regression-guard. Допустим, вы оптимизировали репозиторий и снизили количество запросов с 50 до 3 для конкретного сценария. Добавив тест, который проверяет `<= 3`, вы защищаете оптимизацию от случайного отката: любой разработчик, который «улучшит читаемость» и сломает граф загрузки, получит красный тест и повод задуматься.

Ниже пример `@DataJpaTest`, который включает статистику Hibernate и проверяет количество SQL-запросов для метода репозитория. Java и Kotlin версии аналогичны.

```java
package com.example.jpa.stats;

import com.example.jpa.customer.Customer;
import com.example.jpa.customer.CustomerRepository;
import jakarta.persistence.EntityManagerFactory;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.ActiveProfiles;

import jakarta.annotation.Resource;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@ActiveProfiles("test")
class CustomerRepositoryStatisticsTest {

    @Resource
    private CustomerRepository customerRepository;

    @Resource
    private EntityManagerFactory entityManagerFactory;

    private Statistics statistics;

    @BeforeEach
    void setUp() {
        SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
        statistics = sessionFactory.getStatistics();
        statistics.clear();
    }

    @Test
    void shouldLoadCustomersWithExpectedQueryCount() {
        for (int i = 0; i < 5; i++) {
            Customer customer = new Customer();
            customer.setEmail("user" + i + "@example.com");
            customerRepository.save(customer);
        }
        statistics.clear();

        var all = customerRepository.findAll();

        assertThat(all).hasSize(5);
        long queryCount = statistics.getPrepareStatementCount();
        assertThat(queryCount).isLessThanOrEqualTo(2);
    }
}
```

```kotlin
package com.example.jpa.stats

import com.example.jpa.customer.Customer
import com.example.jpa.customer.CustomerRepository
import jakarta.annotation.Resource
import jakarta.persistence.EntityManagerFactory
import org.assertj.core.api.Assertions.assertThat
import org.hibernate.SessionFactory
import org.hibernate.stat.Statistics
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.test.context.ActiveProfiles

@DataJpaTest
@ActiveProfiles("test")
class CustomerRepositoryStatisticsTest {

    @Resource
    lateinit var customerRepository: CustomerRepository

    @Resource
    lateinit var entityManagerFactory: EntityManagerFactory

    private lateinit var statistics: Statistics

    @BeforeEach
    fun setUp() {
        val sessionFactory: SessionFactory =
            entityManagerFactory.unwrap(SessionFactory::class.java)
        statistics = sessionFactory.statistics
        statistics.clear()
    }

    @Test
    fun `should load customers with expected query count`() {
        repeat(5) { i ->
            val customer = Customer().apply {
                email = "user$i@example.com"
            }
            customerRepository.save(customer)
        }
        statistics.clear()

        val all = customerRepository.findAll()

        assertThat(all).hasSize(5)
        val queryCount = statistics.prepareStatementCount
        assertThat(queryCount).isLessThanOrEqualTo(2)
    }
}
```

---

## Инварианты БД: уникальные/`check`/FK — негативные тесты на нарушения

Валидация на уровне Java-кода (`@NotNull`, `@Size`, `@Pattern`) защищает только на входе. В любой нетривиальной системе всегда остаётся путь обойти бизнес-слой: миграция, batch-скрипт, баг в одном сервисе, который пишет в таблицу, которую читает другой. Единственная настоящая защита от некорректных данных — ограничения на уровне БД: `NOT NULL`, `UNIQUE`, `CHECK`, внешние ключи. Поэтому их нужно не только описать в DDL, но и реально тестировать.

Негативные тесты на инварианты БД несложны по идее: вы пытаетесь сохранить некорректную сущность, ожидаете `DataIntegrityViolationException` (или более конкретный тип, зависящий от драйвера/диалекта) и проверяете, что транзакция откатывается. Важно явно вызывать `flush()` или использовать `saveAndFlush`, чтобы Hibernate действительно отправил SQL в БД и получил ошибку — иначе ограничение может сработать уже после завершения теста.

Уникальные ограничения (`UNIQUE`) — первый кандидат на такие тесты. Например, `email` пользователя должен быть уникален. Тест выглядит так: создаём первого пользователя, сохраняем; создаём второго с тем же email, пытаемся сохранить и флашим; ожидаем, что выброшено исключение. Такой тест гарантирует, что constraint действительно присутствует в миграциях, и никто не «удалил его случайно» при модификации схемы.

`CHECK`-ограничения отвечают за правдоподобие данных: сумма должна быть неотрицательной, дата начала не позже даты окончания, статус входит в список значений и т.п. Тест здесь аналогичный: создаём сущность с нарушением ограничения, пробуем сохранить и ловим ошибку. Особенно полезно это для сложных check’ов (например, по нескольким колонкам), где легко ошибиться в логике в DDL.

Внешние ключи (FK) защищают ссылки между таблицами. Тесты на них важны по двум причинам. Во-первых, чтобы убедиться, что вы действительно не можете создать «висящие» записи (например, payment без существующего заказа). Во-вторых, чтобы проверить каскадное поведение: `ON DELETE CASCADE/SET NULL/RESTRICT`. Типичный негативный тест — попытка сохранить дочернюю сущность с несуществующим `parent_id` и ожидание ошибки. Позитивный — удаление родителя и проверка того, что дети либо удалились, либо обновились по правилам.

При тестировании инвариантов важно опираться на миграции, а не на автогенерацию схемы через `hibernate.hbm2ddl.auto`. В идеале тестовый стенд должен поднимать схему ровно так же, как это делает Flyway/Liquibase на проде. Тогда тесты выступают ещё и в роли «интеграционных» для миграций: если constraint исчез, тест упадёт, и это будет видно в CI.

Ещё один момент — уровень, на котором вы ловите исключения. `DataIntegrityViolationException` — это обёртка Spring над `SQLException`. В большинстве случаев этого достаточно: вы просто проверяете, что нарушение инварианта не проходит. Если вам нужно более тонко различать типы ошибок (например, уникальность vs FK), можно смотреть на `SQLException` внутри и анализировать `SQLState`/код ошибки. Делать это стоит только в особых случаях, чтобы не привязывать тесты слишком сильно к конкретной СУБД.

Наконец, помните, что негативные тесты на инварианты должны существовать там, где от нарушения инварианта действительно больно бизнесу. Не надо тестировать каждый `NOT NULL` ради самих тестов, но уникальность бизнес-ключа, корректность суммы, ссылки на критичные сущности — вполне заслуживают отдельного сценария.

Пример интеграционного теста, который проверяет уникальность email и FK-ограничение. Java и Kotlin версии — зеркальные.

```java
package com.example.jpa.constraints;

import com.example.jpa.customer.Customer;
import com.example.jpa.customer.CustomerRepository;
import com.example.jpa.order.Order;
import com.example.jpa.order.OrderRepository;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.test.context.ActiveProfiles;

import jakarta.annotation.Resource;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

@DataJpaTest
@ActiveProfiles("test")
class ConstraintsTest {

    @Resource
    private CustomerRepository customerRepository;

    @Resource
    private OrderRepository orderRepository;

    @PersistenceContext
    private EntityManager entityManager;

    @Test
    void uniqueEmailShouldBeEnforced() {
        Customer first = new Customer();
        first.setEmail("unique@example.com");
        customerRepository.saveAndFlush(first);

        Customer second = new Customer();
        second.setEmail("unique@example.com");

        assertThatThrownBy(() -> {
            customerRepository.saveAndFlush(second);
        }).isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    void foreignKeyShouldPreventDanglingOrders() {
        Order order = new Order();
        order.setCustomerId(999_999L); // поле в Order, не @ManyToOne, а FK колонка

        assertThatThrownBy(() -> {
            orderRepository.saveAndFlush(order);
        }).isInstanceOf(DataIntegrityViolationException.class);
    }
}
```

```kotlin
package com.example.jpa.constraints

import com.example.jpa.customer.Customer
import com.example.jpa.customer.CustomerRepository
import com.example.jpa.order.Order
import com.example.jpa.order.OrderRepository
import jakarta.persistence.EntityManager
import jakarta.persistence.PersistenceContext
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.test.context.ActiveProfiles
import jakarta.annotation.Resource

@DataJpaTest
@ActiveProfiles("test")
class ConstraintsTest {

    @Resource
    lateinit var customerRepository: CustomerRepository

    @Resource
    lateinit var orderRepository: OrderRepository

    @PersistenceContext
    lateinit var entityManager: EntityManager

    @Test
    fun `unique email should be enforced`() {
        val first = Customer().apply {
            email = "unique@example.com"
        }
        customerRepository.saveAndFlush(first)

        val second = Customer().apply {
            email = "unique@example.com"
        }

        assertThatThrownBy {
            customerRepository.saveAndFlush(second)
        }.isInstanceOf(DataIntegrityViolationException::class.java)
    }

    @Test
    fun `foreign key should prevent dangling orders`() {
        val order = Order().apply {
            customerId = 999_999L  // FK колонка без существующего родителя
        }

        assertThatThrownBy {
            orderRepository.saveAndFlush(order)
        }.isInstanceOf(DataIntegrityViolationException::class.java)
    }
}
```

---

## Контроль таймаутов/блокировок: сценарии дедлоков, пессимистические lock-тесты, «грязные» чтения

Многие самые неприятные баги persistence-слоя — не про «не сохранилось», а про «подвисло» или «упало через 30 секунд с дедлоком». Постфактум такие вещи диагностируются тяжело: нужно копать pg_stat_activity, логи СУБД, смотреть графики. Гораздо приятнее иметь пару целевых тестов, которые воспроизводят типичные сценарии блокировок и таймаутов, и гарантируют, что код реагирует на них предсказуемо.

Пессимистические блокировки (`SELECT ... FOR UPDATE`, `LockModeType.PESSIMISTIC_WRITE/READ`) — первый кандидат на такие тесты. Идея в том, чтобы смоделировать две конкурентные транзакции, которые пытаются взять один и тот же ресурс. Первая успешно берёт блокировку и «засыпает», вторая — либо ждёт до тайм-аута и падает с исключением, либо сразу получает отказ, в зависимости от настроек. Тест должен убедиться, что второе поведение действительно срабатывает за разумное время, а не превращается в бесконечное ожидание.

Реализуется это обычно через запуск двух операций в разных потоках, каждая — в своей `@Transactional(propagation = REQUIRES_NEW)` обёртке. Внутри первой операции вы вызываете репозиторий с пессимистической блокировкой (`@Lock(PESSIMISTIC_WRITE)`), потом искусственно ждёте (например, `Thread.sleep`), имитируя долгую обработку. Вторая операция пытается сделать то же самое, но с низким тайм-аутом `lock_timeout` на уровне БД или с ограничением времени на стороне приложения. В тесте вы проверяете, что вторая операция падает с `PessimisticLockingFailureException` или подобным.

Тайм-ауты запросов — ещё одна важная тема. Если вы ошиблись с индексацией или запрос внезапно стал тяжёлым, приложение не должно «висеть» минутами. Тайм-аут можно задавать на разных уровнях: JDBC-драйвер (`setQueryTimeout`), Spring-транзакция (`@Transactional(timeout = ...)`), настройки пула, коннектора (например, в WebClient). Тесты можно строить вокруг искусственно тяжёлого запроса (например, `SELECT pg_sleep(2)` в PostgreSQL) и низкого тайм-аута — вы проверяете, что через N миллисекунд прилетает exception, а не блокировка thread pool’а на минуту.

Dirty read и аномалии изоляции сложнее тестировать, и сильно зависят от базы. В PostgreSQL по умолчанию `READ COMMITTED`, и «грязных» чтений в строгом смысле нет, но возможны non-repeatable reads и phantom reads на более низких уровнях. Для этих вещей редко пишут общие тесты, но конкретный баг, связанный с изоляцией (например, «при одновременном изменении баланса и чтении отчёта видны промежуточные значения»), имеет смысл зафиксировать как regression-test: два потока, разные изоляции, проверки того, что видит читающая транзакция.

При построении подобных тестов ключевой риск — флаки. Если вы завязываетесь на «сначала поток А, потом через 10 мс поток Б», рано или поздно на CI всё пойдёт не так, как ожидалось. Для надёжности используйте синхронизацию: `CountDownLatch`, `CyclicBarrier`, чёткие точки, где обе транзакции стартуют или ждут. Важнее воспроизвести порядок операций, чем конкретные интервалы времени.

Конечно, такие тесты — тяжёлая артиллерия, их не должно быть десятки. Обычно хватает 2–5 хорошо продуманных сценариев, которые отражают реальные риски: дедлок при обновлении двух таблиц, пессимистическая блокировка на «очереди задач», тайм-аут при долгом отчёте. Остальное — зона нагрузочного тестирования и профилировки, а не unit/integration тестов.

Ниже упрощённый пример, который демонстрирует тест на пессимистическую блокировку: первый поток захватывает строку, второй — получает исключение. Код иллюстративный, в реальном проекте сценарий нужно адаптировать под вашу модель. Java и Kotlin версии используют одинаковый подход.

```java
package com.example.jpa.lock;

import com.example.jpa.order.Order;
import com.example.jpa.order.OrderLockingRepository;
import jakarta.annotation.Resource;
import org.junit.jupiter.api.Test;
import org.springframework.dao.PessimisticLockingFailureException;
import org.springframework.stereotype.Service;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringJUnitConfig(LockTestConfig.class)
@DirtiesContext
class PessimisticLockTest {

    @Resource
    private LockService lockService;

    @Test
    void secondTransactionShouldFailOnPessimisticLock() throws Exception {
        Long orderId = lockService.createOrder();

        ExecutorService executor = Executors.newFixedThreadPool(2);

        CountDownLatch latch = new CountDownLatch(1);

        Future<Void> first = executor.submit(() -> {
            lockService.lockAndHold(orderId, latch);
            return null;
        });

        latch.await();

        Future<Void> second = executor.submit(() -> {
            lockService.tryLockWithTimeout(orderId);
            return null;
        });

        assertThatThrownBy(second::get)
                .hasCauseInstanceOf(PessimisticLockingFailureException.class);

        first.cancel(true);
        executor.shutdownNow();
    }
}

@Service
class LockService {

    private final OrderLockingRepository orderRepository;

    LockService(OrderLockingRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public Long createOrder() {
        Order order = new Order();
        order.setStatus("NEW");
        Order saved = orderRepository.save(order);
        return saved.getId();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void lockAndHold(Long orderId, CountDownLatch latch) {
        Order order = orderRepository.findForUpdate(orderId);
        latch.countDown();
        try {
            Thread.sleep(2_000);
        } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 1)
    public void tryLockWithTimeout(Long orderId) {
        orderRepository.findForUpdate(orderId);
    }
}
```

```java
package com.example.jpa.order;

import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.stereotype.Repository;

@Repository
public interface OrderLockingRepository extends JpaRepository<Order, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Order findForUpdate(Long id);
}
```

```kotlin
package com.example.jpa.lock

import com.example.jpa.order.Order
import com.example.jpa.order.OrderLockingRepository
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test
import org.springframework.dao.PessimisticLockingFailureException
import org.springframework.stereotype.Service
import org.springframework.test.annotation.DirtiesContext
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig
import org.springframework.transaction.annotation.Propagation
import org.springframework.transaction.annotation.Transactional
import java.util.concurrent.CountDownLatch
import java.util.concurrent.Executors
import java.util.concurrent.Future
import jakarta.annotation.Resource

@SpringJUnitConfig(LockTestConfig::class)
@DirtiesContext
class PessimisticLockTest {

    @Resource
    lateinit var lockService: LockService

    @Test
    fun `second transaction should fail on pessimistic lock`() {
        val orderId = lockService.createOrder()
        val executor = Executors.newFixedThreadPool(2)
        val latch = CountDownLatch(1)

        val first: Future<Void?> = executor.submit<Void?> {
            lockService.lockAndHold(orderId, latch)
            null
        }

        latch.await()

        val second: Future<Void?> = executor.submit<Void?> {
            lockService.tryLockWithTimeout(orderId)
            null
        }

        assertThatThrownBy { second.get() }
            .hasCauseInstanceOf(PessimisticLockingFailureException::class.java)

        first.cancel(true)
        executor.shutdownNow()
    }
}

@Service
class LockService(
    private val orderRepository: OrderLockingRepository
) {

    @Transactional
    fun createOrder(): Long {
        val order = Order().apply {
            status = "NEW"
        }
        val saved = orderRepository.save(order)
        return saved.id!!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun lockAndHold(orderId: Long, latch: CountDownLatch) {
        val order = orderRepository.findForUpdate(orderId)
        latch.countDown()
        try {
            Thread.sleep(2_000)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 1)
    fun tryLockWithTimeout(orderId: Long) {
        orderRepository.findForUpdate(orderId)
    }
}
```

Такие тесты не нужно плодить в огромном количестве, но 2–3 хорошо продуманных сценария по блокировкам и тайм-аутам дают сильную страховку от регрессий на уровне concurrency и поведения БД под нагрузкой.




