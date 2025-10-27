---
layout: page
title: "Миграции: Flyway/Liquibase"
permalink: /spring/migrations_flyway_liquibase
---

# 0. Введение

## Что это такое

Миграции базы данных — это управляемые, версионируемые изменения структуры и (иногда) содержимого БД, которые живут в вашем репозитории рядом с исходным кодом приложения и применяются автоматически по понятным правилам. В отличие от «скриптов где-то у DBA», миграции превращают эволюцию схемы в такой же «код», как Java/Kotlin-классы: их можно ревьюить, тестировать, катить по средам и воспроизводить на новом окружении с нуля. В экосистеме Spring Boot это обычно делают двумя зрелыми инструментами: **Flyway** и **Liquibase**.

Под капотом оба инструмента хранят служебную таблицу истории (у Flyway — `flyway_schema_history`, у Liquibase — `databasechangelog`), куда записывают каждую применённую «ступеньку» изменений. Таблица истории — опора для идемпотентности: если конкретная миграция уже применена, она не запустится повторно; если её текст изменился, инструмент заметит расхождение (checksum/MD5) и не даст тихо «переписать прошлое».

В простейшем случае миграции — это набор SQL-файлов `V1__init.sql`, `V2__add_table_orders.sql`, `V3__add_index.sql`. Flyway прочитает их по возрастанию версий и выполнит. Liquibase оперирует **changeset**-ами внутри **changelog**-файлов (XML/YAML/JSON/SQL) и даёт более богатый словарь действий: от DDL до «вставь данные из CSV», с условиями, лейблами и явными блоками rollback.

В мире Spring Boot мигратор запускается автоматически при старте (если не выключить), **до** поднятия `EntityManagerFactory`. Это важная гарантия: код на JPA увидит схему уже нужной версии. В CI/CD миграции можно запускать как отдельную стадию (Gradle/Maven плагин, Docker-job, Helm hook), что особенно удобно в продакшн-релизах с несколькими приложениями.

Для JVM-проектов миграции — не просто «скрипты». Это часть архитектуры: способ зафиксировать контракт между кодом и БД. Они нужны и в монолите, и в микросервисах, и в ETL/аналитических пайплайнах. Где есть схема и есть эволюция — там нужны миграции. Исключения редки: чисто in-memory прототипы, одноразовые утилиты, песочницы без требований к воспроизводимости.

На первый взгляд миграции — это «про DDL». На практике туда попадает и **контролируемый DML**: стартовые справочники, data-backfill под новую фичу, разовая чистка кривых данных. Главное — дисциплина: опасный DML дробить, делать идемпотентным, отмечать контекстами/лейблами (Liquibase) или выносить в repeatable scripts (Flyway).

У миграций есть терминология. Во Flyway — **versioned** (`V__`) и **repeatable** (`R__`) миграции; понятия `baseline`, `outOfOrder`, `repair`; checksum для контроля изменения файла. В Liquibase — **changelog** и **changeset** c `id/author`, **preconditions**, **labels/contexts**, флаги `runOnChange/runAlways`, явные блоки **rollback** и теги (`tag`, `rollbackToTag`). Эти термины мы будем использовать в дальнейших главах.

Наконец, миграции — это **про автоматизацию**. Вы не должны вручную «накатывать» SQL на каждую среду. Либо приложение само на старте применяет нужные шаги, либо пайплайн делает это перед деплоем. Ровно это убирает «дрейф схемы» между стендами и превращает «у меня работает» в «у всех одинаково».

Чтобы не быть голословным, ниже два минимальных примера: Java-миграция для Flyway (когда удобнее писать на коде, а не SQL) и простейший Liquibase-changeset в **двух форматах** — YAML и XML.

**Java — пример Java-миграции для Flyway (создаём таблицу)**

```java
package com.example.migrations;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

import java.sql.Statement;

/**
 * Эквивалент V1__create_table_users.sql, но на Java.
 */
public class V1__create_table_users extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        try (Statement st = context.getConnection().createStatement()) {
            st.execute("""
                create table if not exists users(
                  id bigserial primary key,
                  email text not null unique,
                  created_at timestamptz not null default now()
                )
            """);
        }
    }
}
```

**Kotlin — та же Java-миграция для Flyway**

```kotlin
package com.example.migrations

import org.flywaydb.core.api.migration.BaseJavaMigration
import org.flywaydb.core.api.migration.Context

class V1__create_table_users : BaseJavaMigration() {
    override fun migrate(context: Context) {
        context.connection.createStatement().use { st ->
            st.execute(
                """
                create table if not exists users(
                  id bigserial primary key,
                  email text not null unique,
                  created_at timestamptz not null default now()
                )
                """.trimIndent()
            )
        }
    }
}
```

**Liquibase — YAML changeset (эквивалентно)**

```yaml
# src/main/resources/db/changelog/db.changelog.yaml
databaseChangeLog:
  - changeSet:
      id: 1-create-users
      author: team
      changes:
        - createTable:
            tableName: users
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: email, type: TEXT, constraints: { nullable: false, unique: true } }
              - column: { name: created_at, type: TIMESTAMPTZ, defaultValueComputed: now(), constraints: { nullable: false } }
```

**Liquibase — XML changeset (тот же смысл)**

```xml
<!-- src/main/resources/db/changelog/db.changelog.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">

    <changeSet id="1-create-users" author="team">
        <createTable tableName="users">
            <column name="id" type="BIGSERIAL">
                <constraints primaryKey="true"/>
            </column>
            <column name="email" type="TEXT">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="now()">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

**application.yml — показать, что мигратор включён (можно оставить только один из блоков)**

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
  liquibase:
    enabled: false   # включайте один инструмент за раз
```

---

## Зачем это

Главный ответ — **воспроизводимость**. Новому разработчику не нужно «просить дамп» и вручную крутить DDL. Он делает `./gradlew bootRun` (или запускает контейнеры), и мигратор приводит локальную БД в нужное состояние. Это снижает барьер входа и убирает случайность из «настроек руками».

Второй мотив — **контроль изменений**. Миграции — такие же артефакты коммита, как код. По PR видно, что именно меняется в схеме, можно обсудить индексы, типы, ограничения, а не «довериться, что DBA всё сделал правильно». История миграций — это и история архитектурных решений по данным.

Третий — **автоматизация окружений**. CI поднимает чистую БД (Testcontainers, managed DB), прогоняет миграции, крутит тесты. Stage/Prod получают те же артефакты. Никаких «сюрпризов» в виде «на стейдже была лишняя колонка», потому что инструменты следят за версией и checksum-ами.

Четвёртый — **безопасность изменений**. Liquibase позволяет описать `rollback` (и даже сделать «сухой прогон» `updateSQL`), а Flyway — зафиксировать политику «вперёд-только» и чинить огрехи через `repair` с контролем checksum. Это лучше, чем «попробуем откатить руками под ночь» — мигратор делает те же шаги детерминированно.

Пятый — **идемпотентность и дисциплина**. У Flyway repeatable-скрипты предназначены для объектов вроде представлений и функций; у Liquibase — `runOnChange/runAlways`, preconditions. Вы описываете, *когда* и *почему* нужно перевыполнить шаг. Это особенно важно для объектов, которые «ломаются» от мелких правок.

Шестой — **снижение дрейфа**. Болотная тропа — ручные «горячие фиксы» в проде: администратор быстро добавил индекс, а в репозитории его нет. Через месяц другая команда развернула сервис на новый кластер — и словила деградацию. С мигратором «горячий фикс» — это тоже changeset/миграция, он попадёт в код и будет повторён везде.

Седьмой — **разделение ролей**. Разработчики описывают изменение (DDL/DML), SRE/DBA смотрит, как это скажется на проде (блокировки, online-DDL, окна обслуживания). Мигратор — общий язык, который видят и те, и другие; это снижает количество «устных договорённостей» и ошибок от непонимания.

Восьмой — **встраиваемость в процесс релиза**. Миграции можно запускать как отдельную стадию (Kubernetes Job/Helm hook, CI-шаг), можно запускать на старте приложения — подход выбирают под требования. В любом случае роль понятна, логирование есть, метрики можно собрать.

Девятый — **поддержка нескольких сред и клиентов**. Liquibase с контекстами/лейблами позволяет держать в одном changelog’е изменения «только для stage» или «только для белой фичи». Flyway решает похожие задачи через разнесение локаций и профили.

Десятый — **простые примеры для «почему»**. Создание стартовой таблицы и индекса — миграция. Добавление справочника с данными — миграция. Перевод существующих e-mail на `lower()` с частичным уникальным индексом — миграция. Всё это легко хранится, ревьюится и воспроизводится.

**Flyway — минимальная SQL-миграция (почему это удобно)**

```sql
-- src/main/resources/db/migration/V1__init.sql
create table if not exists users(
  id bigserial primary key,
  email text not null unique,
  created_at timestamptz not null default now()
);
create index if not exists idx_users_created_at on users (created_at);
```

**Liquibase YAML — тот же смысл, плюс seed-данные**

```yaml
databaseChangeLog:
  - changeSet:
      id: 1-init
      author: team
      changes:
        - createTable:
            tableName: users
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: email, type: TEXT, constraints: { nullable: false, unique: true } }
              - column: { name: created_at, type: TIMESTAMPTZ, defaultValueComputed: now(), constraints: { nullable: false } }
        - insert:
            tableName: users
            columns:
              - column: { name: email, value: "demo@example.com" }
```

**Liquibase XML — эквивалент YAML**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">

    <changeSet id="1-init" author="team">
        <createTable tableName="users">
            <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
            <column name="email" type="TEXT"><constraints nullable="false" unique="true"/></column>
            <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="now()">
                <constraints nullable="false"/>
            </column>
        </createTable>
        <insert tableName="users">
            <column name="email" value="demo@example.com"/>
        </insert>
    </changeSet>
</databaseChangeLog>
```

**Java — программная интеграция Liquibase (почему удобно централизовать точки входа)**

```java
package com.example.conf;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

/** В редких случаях удобно управлять Liquibase вручную (локации, контексты) */
@Configuration
public class LiquibaseConfig {
    @Bean
    SpringLiquibase springLiquibase(DataSource ds) {
        SpringLiquibase lb = new SpringLiquibase();
        lb.setDataSource(ds);
        lb.setChangeLog("classpath:db/changelog/db.changelog.yaml");
        lb.setContexts("prod"); // напр., включать только под prod
        return lb;
    }
}
```

**Kotlin — та же конфигурация**

```kotlin
package com.example.conf

import liquibase.integration.spring.SpringLiquibase
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class LiquibaseConfig {
    @Bean
    fun springLiquibase(ds: DataSource) = SpringLiquibase().apply {
        dataSource = ds
        changeLog = "classpath:db/changelog/db.changelog.yaml"
        contexts = "prod"
    }
}
```

---

## Где используется

Миграции применяются везде, где код работает вместе с БД: в монолитах с одной схемой, в зоопарке микросервисов, в внутренних интеграциях, в системах отчётности, в платформах данных. Даже если БД «общая», каждый сервис может иметь свои **логические** миграции (свой каталог, свои таблицы), а платформа — общий слой миграций (справочники, утилитарные таблицы).

В микросервисной архитектуре миграции обычно живут в каждом репозитории сервиса и катаются независимо, как часть релиза сервиса. Это дисциплинирует: сервис сам владеет своей схемой, не ломает чужую, публикует изменения в документации. Для межсервисных таблиц (редко, но бывает) — отдельный «shared» модуль/репозиторий с собственным циклом.

В ETL/аналитике миграции задают структуры стадий (staging, marts), индексы под отчёты, материализованные представления. Liquibase там ценят за YAML/XML-описания и preconditions; Flyway любят за простоту и повторяемые скрипты (`R__`) для представлений и функций.

В Kubernetes/Helm миграции запускают либо **на старте** (приложение само), либо **как Job**/`helm hook`. Для критичных релизов с окнами обслуживания чаще выбирают отдельный шаг: так можно «поймать» ошибки раньше и не держать приложение в полуподнятом состоянии. В простых сценариях вполне годится on-start — особенно на dev/stage.

В мультисхемных и многотенантных системах миграции идут в цикле по схемам/арендаторам. Flyway умеет `schemas`/`defaultSchema` и multiple locations; Liquibase — contexts/labels и параметры. Важна идемпотентность и быстрый rollback/repair при сбое в середине «массива» схем.

В CI/CD миграции — привычная стадия: `./gradlew flywayMigrate` или `./gradlew update` (Liquibase), затем smoke-тесты и деплой. Чтобы «всё само», добавляют тесты, которые поднимают чистый Postgres, прогоняют миграции и проверяют базовые инварианты (таблицы на месте, индексы созданы, справочники загружены).

В локальной разработке миграции — лазер от болей «какой дамп мне взять». Вы чистите БД, запускаете приложение — и получаете актуальную схему. Нужно «откатить» — для Liquibase есть `rollback` по тэгу; для Flyway обычно — «вперёд фикс» плюс `clean` только в dev (и выключен в prod).

Наконец, миграции — это артефакты, которыми удобно **делиться**. Вы можете собрать Docker-слой с миграциями и запустить их в окружении без вашего приложения (например, в init-контейнере). Или хранить `updateSQL` (Liquibase) как артефакт релиза и прогонять его через change-management.

**Gradle (Groovy DSL) — зависимости**

```groovy
plugins {
  id 'org.springframework.boot' version '3.3.4'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'java'
}
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.flywaydb:flyway-core'
  implementation 'org.liquibase:liquibase-core'
  runtimeOnly 'org.postgresql:postgresql:42.7.3'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL) — то же**

```kotlin
plugins {
  id("org.springframework.boot") version "3.3.4"
  id("io.spring.dependency-management") version "1.1.6"
  java
}
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.flywaydb:flyway-core")
  implementation("org.liquibase:liquibase-core")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**docker-compose.yml — локальный Postgres под миграции**

```yaml
version: "3.8"
services:
  pg:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports: ["5432:5432"]
```

**Java — пример профилирования: включаем Liquibase только в prod, Flyway — в dev**

```java
package com.example.profiles;

import org.springframework.boot.autoconfigure.condition.ConditionalOnExpression;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnExpression("'${spring.profiles.active:dev}'.contains('dev')")
class DevConfig {
    // В dev включён Flyway (по application-dev.yml), Liquibase выключен
}
```

**Kotlin — аналогичная идея с профилями**

```kotlin
package com.example.profiles

import org.springframework.boot.autoconfigure.condition.ConditionalOnExpression
import org.springframework.context.annotation.Configuration

@Configuration
@ConditionalOnExpression("'${spring.profiles.active:dev}'.contains('prod')")
class ProdConfig {
    // В prod включён Liquibase (по application-prod.yml), Flyway выключен
}
```

**application-dev.yml / application-prod.yml — разнести инструменты по средам**

```yaml
# application-dev.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
  liquibase:
    enabled: false
```

```yaml
# application-prod.yml
spring:
  flyway:
    enabled: false
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog.yaml
    contexts: prod
```

---

## Какие задачи/проблемы решает

Первая проблема — **дрейф схемы**. Когда у каждого стенда «живёт своя жизнь», локальные фиксы не доезжают до репозитория, а прод начинает отличаться от стейджа. История миграций и автомигратор убирают дрейф: одно и то же состояние получается везде из одних и тех же файлов, в одном и том же порядке.

Вторая — **аудит и воспроизводимость**. Нужно понять, «когда появился этот индекс» и «почему поле стало `NOT NULL`». С миграциями ответ прост: смотри PR, смотри историю таблицы `schema_history`/`databasechangelog`. Можно даже «переиграть» путь на чистой БД и убедиться, что всё применимо.

Третья — **безопасность релизов**. Liquibase поддерживает явные `rollback`-блоки и теги (`tag`/`rollbackToTag`), позволяя аккуратно откатывать неудачный релиз. Во Flyway философия чаще «вперёд фикс», но есть `repair` и baseline/out-of-order для сложных ветвлений. Оба инструмента решают риск «остановили бизнес на час, потому что DDL не прошёл».

Четвёртая — **совместимость и zero-downtime**. Паттерн expand-and-contract (сначала добавить колонку, потом начать писать в неё, потом переключить чтение, потом удалить старое) удобно выражается миграциями. Liquibase помогает контекстами/лейблами, Flyway — разделением на версии и `R__`-скрипты для стабильных объектов (view/func).

Пятая — **координация между командами**. В мультисервисной среде миграции разных сервисов можно раскладывать в разные локации и запускать согласованно (по порядку Helm hooks/Jobs). Это лучше, чем «созвонились с DBA». В случае конфликтов с SQL ответственность видна по коммиту.

Шестая — **контроль опасных операций**. Вы можете запретить `clean` (Flyway) в prod, не дав приложению случайно стереть базу. В Liquibase легко сделать `precondition` и не позволить применить changeset, если состояние БД не соответствует ожиданиям (например, таблица слишком большая — тормозить миграцию).

Седьмая — **мульти-схема/мульти-тенант**. Мигратор умеет идти по списку схем и поддерживать историю в каждой; вы не будете писать скрипты с циклом на bash и думать, где оно упало. Аварии сокращаются, когда инструмент делает одно и то же стабильно.

Восьмая — **условные данные**. Иногда seed-данные нужны только на stage или только в демо-тенанте. Liquibase решает это через contexts/labels, Flyway — через разные locations или профили Spring. Вы перестаёте плодить «ветки SQL под стенд» в wiki.

Девятая — **интеграция с пайплайном**. Мигратор даёт код возврата, логи, метрики времени — это хорошо ложится на CI/CD. Можно артефактировать `updateSQL` (Liquibase) и хранить как «договор» релиза с БД, прикладывать к change-management.

Десятая — **снижение входного порога**. Новому члену команды не нужно знать «тайные процедуры DBA». Есть README, есть `./gradlew update` или «запусти приложение» — и миграции сделают остальное. Это ускоряет разработку и уменьшает число «устных знаний».

**Flyway — безопасные флаги в prod (решает проблему опасных действий)**

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    out-of-order: false
    validate-on-migrate: true
    clean-disabled: true     # запретить clean в проде
```

**Liquibase YAML — changeset с precondition и rollback**

```yaml
databaseChangeLog:
  - changeSet:
      id: 2-add-users-idx
      author: team
      preConditions:
        - onFail: MARK_RAN
        - not:
            indexExists:
              indexName: idx_users_created_at
              tableName: users
      changes:
        - createIndex:
            indexName: idx_users_created_at
            tableName: users
            columns:
              - column: { name: created_at }
      rollback:
        - dropIndex:
            indexName: idx_users_created_at
            tableName: users
```

**Liquibase XML — эквивалент с тегами**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">

    <changeSet id="2-add-users-idx" author="team">
        <preConditions onFail="MARK_RAN">
            <not>
                <indexExists indexName="idx_users_created_at" tableName="users"/>
            </not>
        </preConditions>
        <createIndex indexName="idx_users_created_at" tableName="users">
            <column name="created_at"/>
        </createIndex>
        <rollback>
            <dropIndex indexName="idx_users_created_at" tableName="users"/>
        </rollback>
    </changeSet>

    <tagDatabase tag="release-1.0"/>
</databaseChangeLog>
```

**Java — «страховка» Flyway: отключить clean в прод-профиле программно**

```java
package com.example.safe;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywaySafetyConfig {
    @Bean
    FlywayConfigurationCustomizer forbidClean() {
        return (FluentConfiguration cfg) -> cfg.cleanDisabled(true);
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.safe

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywaySafetyConfig {
    @Bean
    fun forbidClean() = FlywayConfigurationCustomizer { cfg: FluentConfiguration ->
        cfg.cleanDisabled(true)
    }
}
```

# 1. Основы миграций и где их место в Spring Boot

## Что такое миграции БД и почему «DDL/DML в репозитории, а не в голове DBA»

Миграции БД — это набор версионируемых шагов изменения схемы и (иногда) данных, которые живут в **репозитории приложения** и применяются автоматически. Каждый шаг описывает атомарное изменение (создать таблицу, добавить индекс, заполнить справочник), имеет идентификатор/версию и фиксируется в служебной таблице истории. В результате состояние схемы становится таким же «кодом», как и Java/Kotlin-исходники.

Идея «DDL/DML в репозитории» решает ключевую проблему воспроизводимости. Если изменения существуют только в голове DBA или в разрозненных скриптах, они не проходят code review, теряются, конфликтуют, а стенды начинают «дрейфовать». Когда же DDL/DML — часть коммита, они ревьюятся вместе с кодом, автоматически применяются на CI и доезжают до всех окружений.

В Spring Boot миграции интегрируются в **жизненный цикл старта**: мы включаем Flyway или Liquibase, и до инициализации `EntityManagerFactory` БД приводится к ожидаемой версии. Это означает, что JPA-слой всегда видит актуальную схему — меньше «сюрпризов» вида «колонка отсутствует».

Хранение миграций в репозитории позволяет «листать историю» структуры БД. Любой вопрос «когда и почему появилось поле `foo`» имеет точный ответ в blame/PR. Это повышает прозрачность и дисциплину архитектурных решений на уровне данных.

Миграции также стандартизируют **идемпотентность**: инструмент знает, что уже применено, и не выполняет одно и то же дважды. Если файл поменяли «задним числом», контрольная сумма (Flyway checksum / Liquibase MD5) не совпадёт, и старт остановится — это защита от немого переписывания истории.

С миграциями удобно разделять роли. Разработчик описывает необходимое изменение, SRE/DBA — проверяет риски блокировок, время выполнения, план запросов. Решение фиксируется в changeset/миграции — единый источник правды, а не устная договорённость.

Важно понимать, что миграции — не только про DDL. Практически всегда есть «контролируемый DML»: стартовые записи, backfill под новую фичу, аккуратная чистка. Но такой DML должен быть **проверяемым и идемпотентным**, с явной видимостью в коде и логах.

Миграции позволяют включать **проверки и предосторожности**. Liquibase поддерживает `preconditions` (не применять, если индекс уже есть), Flyway — строгую валидацию (`validateOnMigrate=true`) и запрет потенциально опасных действий в проде (`cleanDisabled=true`). Эти ограждения нужны, чтобы миграции были безопасными по умолчанию.

Сервису с миграциями легче жить в микросервисной архитектуре. Каждая команда владеет своей схемой/частью схемы и несёт ответственность за её эволюцию. Несогласованные ручные правки уходят в прошлое, а совместная работа над схемой превращается в привычный процесс PR/CI.

Когда изменения живут в репозитории, новому разработчику не нужен «специальный дамп». Он поднимает локальный Postgres (docker-compose), запускает приложение — и миграции создают актуальную схему. Это ускоряет онбординг и снижает «шифрованные знания».

И наконец, хранив миграции рядом с кодом, мы упрощаем **автоматизацию CI/CD**. Шаг «обновить схему» становится таким же детерминированным, как «собрать артефакт». Если миграции упали — релиз не поедет, а ошибка видна сразу.

**Gradle (Groovy) — минимальные зависимости**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.flywaydb:flyway-core'
  implementation 'org.liquibase:liquibase-core'
  runtimeOnly   'org.postgresql:postgresql:42.7.3'
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.flywaydb:flyway-core")
  implementation("org.liquibase:liquibase-core")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**application.yml — включим Flyway (Liquibase пока выключен)**

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
  liquibase:
    enabled: false
```

**Flyway SQL (V1) и Liquibase YAML/XML — одинаковая логика**

```sql
-- src/main/resources/db/migration/V1__init.sql
create table if not exists users(
  id bigserial primary key,
  email text not null unique,
  created_at timestamptz not null default now()
);
```

```yaml
# src/main/resources/db/changelog/db.changelog.yaml
databaseChangeLog:
  - changeSet:
      id: 1-init
      author: team
      changes:
        - createTable:
            tableName: users
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: email, type: TEXT, constraints: { nullable: false, unique: true } }
              - column: { name: created_at, type: TIMESTAMPTZ, defaultValueComputed: now(), constraints: { nullable: false } }
```

```xml
<!-- src/main/resources/db/changelog/db.changelog.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="1-init" author="team">
    <createTable tableName="users">
      <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
      <column name="email" type="TEXT"><constraints nullable="false" unique="true"/></column>
      <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="now()"><constraints nullable="false"/></column>
    </createTable>
  </changeSet>
</databaseChangeLog>
```

**Java — программный запуск миграций Flyway (как утилита)**

```java
package com.example.migrations;

import org.flywaydb.core.Flyway;

public class FlywayCli {
    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
                .dataSource("jdbc:postgresql://localhost:5432/app","app","app")
                .locations("classpath:db/migration")
                .load();
        flyway.migrate(); // применит V__ и R__ в корректном порядке
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.migrations

import org.flywaydb.core.Flyway

fun main() {
    val flyway = Flyway.configure()
        .dataSource("jdbc:postgresql://localhost:5432/app","app","app")
        .locations("classpath:db/migration")
        .load()
    flyway.migrate()
}
```

---

## Жизненный цикл: локальная разработка → CI → стенды → прод

Жизненный цикл миграций начинается локально. Разработчик создаёт файл миграции (например, `V12__add_orders.sql` или changeset в Liquibase), запускает локальный Postgres (docker-compose), поднимает приложение — и проверяет, что схема обновилась, а приложение стартует. Это быстрый цикл обратной связи.

Следующий этап — CI. На CI мы поднимаем чистую БД (Testcontainers/managed DB), прогоняем миграции «с нуля» и запускаем тесты. Это ловит типичные ошибки: несовместимость типов, забытые индексы, конфликт имён. Если миграции не проходят — релиз останавливается до фикса.

Далее — стенды (dev/stage). Здесь важно применять **тот же набор** миграций, что прошёл на CI. В простых проектах это делает само приложение при старте; в более критичных — отдельная стадия пайплайна (Gradle/Liquibase task, Helm hook), чтобы провал миграций не держал приложение в полуподнятом состоянии.

На проде требования жёстче: окна обслуживания, ограничение простоев, мониторинг. Здесь миграции чаще запускают **отдельной задачей**: CI/CD выполняет `flywayMigrate`/`liquibaseUpdate`, пишет артефакт логов, а потом уже раскатывает приложение. Если всё настроено на on-start — убедитесь, что реплики поднимаются последовательно, а не одновременно бьют в `schema_history`.

Важна идемпотентность жизненного цикла. Если стенд пересобрали заново — миграции должны пройти «с нуля». Если же он уже был на версии 20 — мигратор применит только 21+. В Liquibase и Flyway это обеспечивается служебными таблицами истории и контрольными суммами.

Те же шаги работают и для hotfix. Вы добавляете `V21__fix_index.sql`, CI прогоняет, стейдж получает новую версию схемы и приложение, прод — после ручного подтверждения. История миграций поможет ответить на вопросы аудита — когда и кем изменили схему.

Жизненный цикл должен включать и **обратную совместимость**: паттерн expand-and-contract. Когда релиз тянется в несколько деплоев, сначала катается «расширяющая» миграция (новая колонка/индекс), затем код начинает писать в неё, затем меняется чтение, и только потом — «сжимающая» миграция (удаление старого). CICD обязан упорядочить эти шаги.

В мультисервисной среде важна координация. Если изменения касаются общей таблицы/представления, их лучше вынести в отдельный репозиторий миграций или в модуль «platform-db», чтобы несколько сервисов не пытались менять одно и то же рассыпанно.

Локальная ветка-песочница разработчика не должна ломать общий цикл. Для экспериментов используйте временные миграции в локальной схеме, не коммитьте «грязные» `DROP`/`CREATE`. Перед PR миграции должны быть «чистыми», проходить на чистой БД и не зависеть от артефактов прошлого.

Наблюдаемость — часть жизненного цикла. В CI сохраняйте логи миграций как артефакт, в проде — собирайте метрики времени и статусы. Это поможет видеть «тяжёлые» шаги, оптимизировать индексы и планировать окна.

**Gradle (Groovy) — задачи для CI: Flyway и Liquibase**

```groovy
plugins {
  id 'org.flywaydb.flyway' version '10.16.0'
  id 'org.liquibase.gradle' version '2.2.0'
}
flyway {
  url = System.getenv('DB_URL')
  user = System.getenv('DB_USER')
  password = System.getenv('DB_PASS')
  locations = ['classpath:db/migration']
}
liquibase {
  activities {
    main {
      url System.getenv('DB_URL')
      username System.getenv('DB_USER')
      password System.getenv('DB_PASS')
      changeLogFile 'src/main/resources/db/changelog/db.changelog.yaml'
    }
  }
  runList = 'main'
}
```

**Gradle (Kotlin) — то же**

```kotlin
plugins {
  id("org.flywaydb.flyway") version "10.16.0"
  id("org.liquibase.gradle") version "2.2.0"
}
flyway {
  url = System.getenv("DB_URL")
  user = System.getenv("DB_USER")
  password = System.getenv("DB_PASS")
  locations = arrayOf("classpath:db/migration")
}
liquibase {
  activities.register("main") {
    this.arguments = mapOf(
      "url" to System.getenv("DB_URL"),
      "username" to System.getenv("DB_USER"),
      "password" to System.getenv("DB_PASS"),
      "changeLogFile" to "src/main/resources/db/changelog/db.changelog.yaml"
    )
  }
  runList = "main"
}
```

**Java — «smoke»-тест миграций с Testcontainers**

```java
package com.example.ci;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;

import static org.assertj.core.api.Assertions.assertThat;

class MigrationIT {
  @Test
  void migrateOnCleanDb() {
    try (var pg = new PostgreSQLContainer<>("postgres:16-alpine")) {
      pg.start();
      var flyway = Flyway.configure()
              .dataSource(pg.getJdbcUrl(), pg.getUsername(), pg.getPassword())
              .locations("classpath:db/migration")
              .load();
      var res = flyway.migrate();
      assertThat(res.migrationsExecuted).isGreaterThan(0);
    }
  }
}
```

**Kotlin — аналог**

```kotlin
package com.example.ci

import org.flywaydb.core.Flyway
import org.junit.jupiter.api.Test
import org.testcontainers.containers.PostgreSQLContainer
import kotlin.test.assertTrue

class MigrationIT {
    @Test
    fun migrateOnCleanDb() {
        PostgreSQLContainer("postgres:16-alpine").use { pg ->
            pg.start()
            val flyway = Flyway.configure()
                .dataSource(pg.jdbcUrl, pg.username, pg.password)
                .locations("classpath:db/migration")
                .load()
            val res = flyway.migrate()
            assertTrue(res.migrationsExecuted > 0)
        }
    }
}
```

---

## Артефакты миграций: версии схемы, история применённых шагов, блокировки

Главный артефакт — **таблица истории**. Во Flyway это `flyway_schema_history` с колонками `installed_rank`, `version`, `description`, `checksum`, `installed_on`, и т.д. В Liquibase — `databasechangelog` c `id`, `author`, `filename`, `md5sum`, `dateexecuted`, `orderexecuted` и др. Эти таблицы говорят мигратору, что уже применено, и защищают от повторного запуска.

Второй важный артефакт — **контрольная сумма (checksum/MD5)**. Она вычисляется по содержимому скрипта (или changeset). Если файл изменился после применения, инструмент это заметит и не позволит «молча» переписать прошлое. Во Flyway такую рассинхронизацию можно лечить `flyway repair` (но только осознанно!); в Liquibase — менять `runOnChange` или править историю вручную строго по регламенту.

Третий артефакт — **блокировки**. Liquibase использует таблицу `databasechangeloglock`: при начале миграции ставит lock (строка `locked=true`), и второй процесс не сможет начать. Flyway применяет блокировку транзакционно (внутренние advisory/DDL-механизмы), гарантируя, что параллельный старт не приведёт к гонке. Понимание механизма блокировок важно при параллельном деплое.

Четвёртый — **версия схемы**. Во Flyway «версия» — это номер `V…` последней применённой миграции. В Liquibase нет понятия «версии» по номеру, но есть `tag`’и: вы можете пометить состояние `tagDatabase` и потом к нему откатываться. В проде полезно документировать соответствие тега/версии приложения.

Пятый — **логи миграций**. Они рассказывают, что именно было применено, сколько заняло времени, где warnings. Сохраняйте их артефактом в CI и агрегируйте в проде. Это помогает отслеживать «тяжёлые» шаги и планировать оптимизации (например, `CONCURRENTLY` на индекс).

Шестой — **плейсхолдеры/переменные**. Оба инструмента умеют подставлять значения (имя схемы, префиксы, флаги) на лету. Это превращает миграции в «шаблоны», которые можно запускать в нескольких средах с разными настройками, не копируя файлы.

Седьмой — **повторяемые артефакты**: во Flyway это `R__…` (repeatable), чья checksum отслеживается для перевыполнения (например, для VIEW/FUNCTION). В Liquibase — changeset с `runOnChange`/`runAlways`. Они существуют *параллельно* версионным миграциям, и история их применений тоже отражается в таблице.

Восьмой — **baseline**-запись (Flyway). Она появляется, когда вы подключаетесь к уже существующей схеме и говорите «считать её эквивалентной версии X». Это фиксируется в истории как «стартовая точка», после чего можно катить миграции сверху. Полезно при миграции легаси.

Девятый — **сводка состояния**. Flyway API (`info()`) и Liquibase команды (`status`) позволяют получить список pending/применённых миграций. Это удобно для health-check’ов и ручной диагностики при инцидентах.

Десятый — **соглашения об именовании**. Чёткие имена файлов/changeset’ов — артефакт человеческого уровня: `V23__orders_add_status.sql` лучше, чем `V23.sql`. По имени можно быстро понять цель и найти файл.

**Java — прочитать историю Flyway и вывести в лог**

```java
package com.example.artifacts;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayInfoConfig {

    @Bean
    CommandLineRunner flywayInfoRunner(Flyway flyway) {
        return args -> {
            var info = flyway.info();
            for (MigrationInfo mi : info.all()) {
                System.out.printf("%s | %s | %s | %s%n",
                        mi.getVersion(), mi.getDescription(), mi.getType(), mi.getState());
            }
        };
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.artifacts

import org.flywaydb.core.Flyway
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayInfoConfig {
    @Bean
    fun flywayInfoRunner(flyway: Flyway) = CommandLineRunner {
        val info = flyway.info()
        info.all().forEach { mi ->
            println("${mi.version} | ${mi.description} | ${mi.type} | ${mi.state}")
        }
    }
}
```

**SQL — взглянуть на историю напрямую**

```sql
-- Flyway
select installed_rank, version, description, success, installed_on
from flyway_schema_history order by installed_rank;

-- Liquibase
select id, author, filename, md5sum, dateexecuted, orderexecuted
from databasechangelog order by orderexecuted;

-- Liquibase lock
select * from databasechangeloglock;
```

---

## Роль мигратора в старте приложения и в пайплайне (on-start vs отдельная стадия)

On-start модель: приложение стартует, Spring Boot автоконфигурация поднимает мигратор, и **до** инициализации JPA/`EntityManagerFactory` выполняются pending-изменения. Плюсы: просто, прозрачно, одинаково для всех стендов. Минусы: при ошибке старт «зависает», параллельный rollout нескольких реплик может упираться в блокировки.

Отдельная стадия: миграции катаются **до** деплоя приложения (CI job, Kubernetes Job, Helm hook). Плюсы: чёткая ответственность, можно «провалить» релиз ещё до раскатки, удобно управлять окнами обслуживания. Минусы: нужно поддерживать отдельный артефакт/образ и гарантировать порядок деплоя.

Гибрид: dev/stage — on-start, prod — отдельная стадия. Это хороший компромисс: разработчикам просто, а прод — безопасно и предсказуемо. Переключение делается профильными настройками `spring.flyway.enabled` / `spring.liquibase.enabled`.

Для **конкурентных стартов** (несколько pod’ов) важны блокировки. Liquibase надёжно держит `databasechangeloglock`, Flyway тоже безопасен, но практичнее **серилиализовать** rollout (maxUnavailable=0, maxSurge=1), чтобы не устроить наплыв на одну и ту же таблицу истории.

В pipeline-подходе удобно добавить «сухой прогон»: для Liquibase команда `updateSQL` генерирует чистый SQL без выполнения, его можно отдать DBA на ревью. Для Flyway формально такого режима нет, но можно запускать миграции против «песочницы» и сохранять лог/дамп схемы до/после.

On-start нельзя смешивать с `ddl-auto=update`. Hibernate DDL auto должен быть **выключен** на всех боевых стендах; контроль схемы — забота мигратора. Иначе вы рискуете конфликтами и «магическим» созданием объектов без контроля.

В отдельной стадии проще внедрить **таймауты** и «длительные» приёмы: `CREATE INDEX CONCURRENTLY`, батчи DML, паузы между шагами. Приложение при этом не занято миграциями и быстрее становится готовым к приёму трафика после успешного обновления схемы.

On-start хорош для микросервисов с «лёгкой» схемой и частыми релизами: экономит усилия на отдельные job’ы. Но как только появляются длинные операции — лучше мигрировать отдельно и заранее.

В обоих режимах **логируйте**: сколько миграций применено, сколько заняло, какие warning’и. Эти логи станут артефактом CI и источником метрик в проде.

**application-prod.yml — выключим on-start, будем мигрировать отдельным job’ом**

```yaml
spring:
  flyway:
    enabled: false
  liquibase:
    enabled: false
```

**Java — отдельный «миграционный» main-класс под Flyway**

```java
package com.example.migratejob;

import org.flywaydb.core.Flyway;

public class MigrateJob {
    public static void main(String[] args) {
        Flyway.configure()
                .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
                .locations("classpath:db/migration")
                .load()
                .migrate();
        System.out.println("Migrations applied successfully.");
    }
}
```

**Kotlin — та же утилита**

```kotlin
package com.example.migratejob

import org.flywaydb.core.Flyway

fun main() {
    Flyway.configure()
        .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
        .locations("classpath:db/migration")
        .load()
        .migrate()
    println("Migrations applied successfully.")
}
```

**Liquibase YAML/XML — для отдельной стадии можно применить `updateSQL` заранее**

```yaml
# liquibase.properties (альтернатива через файл свойств)
url: jdbc:postgresql://db:5432/app
username: app
password: app
changeLogFile: src/main/resources/db/changelog/db.changelog.yaml
```

```xml
<!-- db.changelog.xml остаётся тем же; в CI вызываем liquibase:updateSQL -->
```

---

## Риски ручных изменений и «дрейфа» схемы, как мигратор их минимизирует

Ручные изменения несут два класса рисков: **незаметные** (кто-то добавил индекс в проде, но не зафиксировал) и **разрушающие** (кто-то поменял тип колонки «на горячую», сломав приложение). Оба класса минимизируются, когда все изменения проходят через миграции.

Мигратор обеспечивает **строгость порядка**. Во Flyway — это сортировка по версиям `V__`, в Liquibase — порядок выполнения changeset’ов. Это предотвращает «гонки» и позволяет воспроизводить состояние на любом стенде в любой момент времени.

Контрольные суммы защищают от «тихой правки» уже применённого файла. Если вы изменили `V12__add_index.sql` задним числом, Flyway остановит старт: checksum не совпал. Правильный путь — новая миграция `V13__fix_index.sql` или осознанный `repair` с объяснениями.

Liquibase `preconditions` предотвращают повторное выполнение опасных действий: например, `createIndex` выполнится только если индекса ещё нет. Это особенно полезно на «грязных» окружениях, где ручные артефакты могли остаться с прошлых экспериментов.

`validateOnMigrate` (Flyway) и `liquibase.validate` ловят рассинхрон ранним фейлом. Если миграция уже применена, но файл изменён, или если нарушены зависимости — вы узнаете сразу при старте/стадии миграций, а не на середине релиза.

Запрет `clean` в проде (`cleanDisabled=true`) закрывает путь к случайному «снести схему и начать заново». В dev это может быть удобно, но в prod — недопустимо. Политика окружений — часть стратегии миграций.

Документирование «горячих фиксов» через миграции убирает «сюрпризы». Если DBA экстренно создал индекс в проде, через день его нужно **описать миграцией** (с precondition «если индекса нет») и прокатить на остальные стенды. Иначе стейдж/локал останутся без оптимизации.

Наблюдаемость и алерты по миграциям помогают ловить «дрейф» автоматически. Сравнение прод-схемы с HEAD (Liquibase `diff`) может быть nightly-job’ом, который сигналит о расхождениях в индексах/констрейнтах.

Регулярный «health-check схемы» (Flyway `info()` или Liquibase `status`) в health endpoint приложения даст быструю диагностику: «есть непроведённые миграции», «есть checksum mismatch». Это быстрее, чем логиниться в БД и изучать таблицы.

И, наконец, культура «только через миграции» работает, когда она **вшита в процесс**: PR без миграции на DDL не принимается; релиз без зелёных миграционных шагов — не идёт. Это дисциплина, но она окупается.

**Flyway — строгая валидация и запреты**

```yaml
spring:
  flyway:
    validate-on-migrate: true
    clean-disabled: true
    out-of-order: false
```

**Java — осознанный repair (только по регламенту!)**

```java
package com.example.repair;

import org.flywaydb.core.Flyway;
import org.springframework.stereotype.Component;

@Component
public class FlywayRepairTool {
    public void repair(String url, String user, String pass) {
        Flyway.configure().dataSource(url, user, pass).load().repair();
    }
}
```

**Liquibase YAML/XML — preconditions на создание индекса**

```yaml
databaseChangeLog:
  - changeSet:
      id: 3-add-idx-email
      author: dba
      preConditions:
        - onFail: MARK_RAN
        - not:
            indexExists:
              tableName: users
              indexName: idx_users_email
      changes:
        - createIndex:
            tableName: users
            indexName: idx_users_email
            columns:
              - column: { name: email }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="3-add-idx-email" author="dba">
    <preConditions onFail="MARK_RAN">
      <not><indexExists tableName="users" indexName="idx_users_email"/></not>
    </preConditions>
    <createIndex tableName="users" indexName="idx_users_email">
      <column name="email"/>
    </createIndex>
  </changeSet>
</databaseChangeLog>
```

**Kotlin — health-check: есть ли pending миграции Flyway**

```kotlin
package com.example.health

import org.flywaydb.core.Flyway
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

data class MigrationHealth(val pending: Int, val applied: Int)

@RestController
class MigrationHealthController(private val flyway: Flyway) {
    @GetMapping("/actuator/migrations")
    fun status(): MigrationHealth {
        val info = flyway.info()
        val pending = info.pending().size
        val applied = info.applied().size
        return MigrationHealth(pending, applied)
    }
}
```

---

## Ключевые термины: versioned/ repeatable (Flyway), changeset/ changelog (Liquibase)

Во Flyway **versioned migrations** — это файлы `V__…` (например, `V12__add_orders.sql`), применяются строго по возрастанию версии, единожды. Это основа эволюции схемы «вперёд». Каждая такая миграция попадает в `flyway_schema_history` с версией, описанием и checksum.

**Repeatable migrations** — файлы `R__…` (например, `R__create_view_sales.sql`). Они не имеют версии, а привязаны к checksum. Если содержимое изменилось — Flyway перевыполнит их после всех `V__` в порядке имени файла. Идеально для VIEW/FUNCTION/TRIGGER, которые часто корректируют без увеличения версии.

В Liquibase **changelog** — это файл верхнего уровня (XML/YAML/JSON/SQL), который включает **changeset**’ы. Changeset — атомарная операция с `id` и `author`. Liquibase хранит их в `databasechangelog`. Файлы можно «вкладывать» через include, разбивая по доменам/модулям.

У Liquibase есть флаги changeset’ов: `runOnChange` — перевыполнять при изменении содержимого; `runAlways` — выполнять при каждом прогоне. Они дают поведение, похоже на Flyway `R__`, но точнее управляются автором.

Ещё термин Flyway — **baseline**: механизм «встать» на существующую схему, считая её версией X без реального исполнения древних миграций. Полезно при интеграции легаси. Liquibase здесь предлагает `tag` для фиксации состояния и запуск от него.

Liquibase использует **contexts/labels** — механизмы включения/исключения changeset’ов по средам/фичам. Это удобно для seed-данных на stage или «белых» включений. У Flyway прямого аналога нет; обычно используют разные локации миграций и профили Spring.

Оба инструмента поддерживают **Java-based**/custom-миграции: во Flyway — `BaseJavaMigration`, в Liquibase — custom changes. Это выход для сложного DML/ETL, неукладывающегося в декларативный SQL.

Важно понимать, что **repeatable** и `runOnChange` — не для DDL, влияющего на данные (изменение типа колонки). Их область — «логические» объекты (view/func), которые безопасно пересоздавать. Схемные изменения делайте через версионные шаги.

В команде договоритесь о **схеме именования**: кратко и по делу (`add_orders_status`, `create_users_view`). Это облегчает поиск и ревью.

**Flyway — пример R__ для VIEW**

```sql
-- src/main/resources/db/migration/R__create_view_active_users.sql
create or replace view v_active_users as
select id, email from users where deleted_at is null;
```

**Liquibase YAML/XML — changeset с runOnChange для VIEW**

```yaml
databaseChangeLog:
  - changeSet:
      id: view-active-users
      author: team
      runOnChange: true
      changes:
        - createView:
            viewName: v_active_users
            replaceIfExists: true
            selectQuery: "select id, email from users where deleted_at is null"
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="view-active-users" author="team" runOnChange="true">
    <createView viewName="v_active_users" replaceIfExists="true">
      <selectQuery>select id, email from users where deleted_at is null</selectQuery>
    </createView>
  </changeSet>
</databaseChangeLog>
```

**Java/Kotlin — Java-based миграция Flyway (ещё раз, для функций/тяжёлого DML)**

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

public class V2__normalize_emails extends BaseJavaMigration {
    @Override public void migrate(Context ctx) throws Exception {
        ctx.getConnection().createStatement().execute("""
           update users set email = lower(email)
        """);
    }
}
```

```kotlin
package db.migration

import org.flywaydb.core.api.migration.BaseJavaMigration
import org.flywaydb.core.api.migration.Context

class V2__normalize_emails : BaseJavaMigration() {
    override fun migrate(context: Context) {
        context.connection.createStatement().use {
            it.execute("""update users set email = lower(email)""")
        }
    }
}
```

---

## Обзор типовых сценариев: init пустой БД, апгрейд, поддержка нескольких сред

**Init пустой БД**: на чистой базе применяются все `V__` по порядку и `R__`. В Liquibase — весь changelog. Проверьте, что init не содержит «средо-зависимых» вещей (например, огромных сидов), или оберните их в contexts/labels, чтобы dev и prod получали корректные наборы данных.

**Апгрейд**: БД уже содержит часть миграций. Flyway/Liquibase применяют только новые. При конфликте checksum старт/стадия миграции падает — и это хорошо: выясняем причину, не позволяем «переписать прошлое». Апгрейд должен быть безопасным для данных и совместимым с кодом (expand-and-contract).

**Несколько сред**: dev, test, stage, prod. В Liquibase используйте `contexts`/`labels`, чтобы сиды/диагностические объекты (расширенные view) не попадали в prod. Во Flyway обычно разводят **локации** (например, `db/migration` и `db/migration-dev`) и управляют включением через Spring-профили.

**Baseline легаси**: подключаемся к существующей схеме, ставим `baselineOnMigrate=true` и `baselineVersion=1`, чтобы все старые изменения считались «встроенными», а дальше — обычные миграции `V2+`. В Liquibase аналог — сгенерировать начальный changelog (`generateChangeLog`) и пометить состояние `tag`.

**Out-of-order**: иногда ветки фич ведут к «дырке» в номерах версий (например, на стейдже есть V12,V13, а в проде ещё нет V12). Flyway может применять миграции «вне порядка» (`outOfOrder=true`), но это требует дисциплины именования и редкого использования. Лучше объединять ветки и избегать долгоживущих расхождений.

**Seed-данные**: стартовые записи для прод/стейджа. В Liquibase это удобно контекстами (`context: prod`), в Flyway — отдельной локацией или `R__` с идемпотентными вставками. Следите за объёмом: большие вставки — опасность блокировок, лучше бить на батчи.

**Мультисхема/мультитенант**: миграции в цикле по схемам/арендаторам. Flyway — `schemas`/`defaultSchema` + цикл; Liquibase — параметризуемые changelog’и + contexts/labels. Важно, чтобы шаги были **идемпотентны** и легко «ремонтировались» при частичном падении.

**Zero-downtime**: разделяйте DDL и DML на разные релизы, используйте онлайн-приёмы (в PG — `CONCURRENTLY`), прогревайте данные заранее. В миграциях выражайте это несколькими файлами/changeset’ами, не пытайтесь «сделать всё сразу».

**Откаты**: Liquibase rollback по тегу — инструмент для нештатных ситуаций. Во Flyway откаты — культура «вперёд фикс»: делайте новый шаг, который возвращает схему к безопасному виду; храните emergency-скрипты отдельно и запускайте вручную по регламенту.

**Документация**: опишите в README шаблон именования, правила baseline/out-of-order, как запускать миграции локально/в CI, как восстанавливаться после сбоев. Это часть типового сценария и снижает входной порог.

**Flyway — baseline для легаси**

```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    baseline-version: 1
```

**Java/Kotlin — programmatic baseline (утилита одноразового запуска)**

```java
package com.example.baseline;

import org.flywaydb.core.Flyway;

public class BaselineTool {
  public static void main(String[] args) {
    Flyway.configure()
        .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
        .baselineOnMigrate(true)
        .baselineVersion("1")
        .load()
        .migrate();
  }
}
```

```kotlin
package com.example.baseline

import org.flywaydb.core.Flyway

fun main() {
    Flyway.configure()
        .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
        .baselineOnMigrate(true)
        .baselineVersion("1")
        .load()
        .migrate()
}
```

**Liquibase YAML/XML — contexts для сидов**

```yaml
databaseChangeLog:
  - changeSet:
      id: seed-prod-roles
      author: team
      context: prod
      changes:
        - insert:
            tableName: roles
            columns:
              - column: { name: code, value: "ADMIN" }
              - column: { name: name, value: "Administrator" }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="seed-prod-roles" author="team" context="prod">
    <insert tableName="roles">
      <column name="code" value="ADMIN"/>
      <column name="name" value="Administrator"/>
    </insert>
  </changeSet>
</databaseChangeLog>
```

# 2. Выбор инструмента: Flyway vs Liquibase

## Философия: конвенции и простота (Flyway) vs выразительность и дифф-генерация (Liquibase)

Flyway строится на идее «прозрачные, упорядоченные файлы миграций + минимум магии». Вы кладёте в `classpath:db/migration` файлы `V1__...sql`, `V2__...sql` и (при необходимости) `R__...sql`, инструмент сортирует их по имени и применяет. Никакой «центральной» декларации, минимум собственных сущностей — только строгие договорённости по именованию и контрольные суммы на файлы. Такая философия особенно удобна командам, которые любят **чистый SQL**, простые правила и предсказуемость.

Liquibase, наоборот, предлагает богатый декларативный язык для описания изменений — **changeset**’ы в **changelog**-файлах (XML/YAML/JSON), с предикатами (`preconditions`), контекстами/лейблами для выбора окружений, явными `rollback` и встроенными командами **diff/generate**. Это даёт выразительность, помогает описывать сложные сценарии (например, «создать индекс, если его нет, и сделать откат»), а также поддерживает «каталогизацию» изменений по доменам (через include’ы).

Практическая разница ощущается в процессе работы. С Flyway почти всегда мыслите в терминах «ещё один файл SQL/Java» и версии. Вы легко читаете историю — это просто список. С Liquibase вы мыслите «шагами» (changeset), которые объединены в один или несколько master-файлов. Структура гибче, но требует дисциплины: правильно выбирать `id/author`, поддерживать чистый `master` и не злоупотреблять `runAlways`.

С точки зрения культуры команды, Flyway хорошо ложится там, где SQL — общая компетенция, а «откат» мыслится как новая «прямоходная» миграция (forward-fix). Liquibase часто ценят команды с сильной ролью DBA и требованием к **управляемым откатам** и «сухим прогонам» (`updateSQL`), где релизы идут через change-management.

Наблюдаемость и контроль тоже отличаются: во Flyway ключевые рычаги — `validateOnMigrate`, `baseline`, `outOfOrder`, `repair`. В Liquibase — `status`, `history`, `rollback`, `diff*` и тонкие `preconditions`. Оба инструмента стабильно работают со Spring Boot и имеют зрелую автоконфигурацию.

Если вы сомневаетесь, хороший эмпирический критерий такой: **нужны ли вам явные откаты, contexts/labels, autogenerate diff?** Если да — смотрите в сторону Liquibase. Если нет — Flyway быстрее приведёт команду к стабильной практике.

**Gradle (Groovy DSL) — подключить оба (выберем позже профилем)**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.flywaydb:flyway-core'
  implementation 'org.liquibase:liquibase-core'
  runtimeOnly   'org.postgresql:postgresql:42.7.3'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.flywaydb:flyway-core")
  implementation("org.liquibase:liquibase-core")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
}
```

**Java — «переключатель» по профилю: включить ровно один инструмент**

```java
package com.example.migrations.profile;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnProperty(value = "migrations.tool", havingValue = "flyway", matchIfMissing = true)
class FlywayEnabled { /* пусто: автоконфигурация Spring Boot включит Flyway */ }

@Configuration
@ConditionalOnProperty(value = "migrations.tool", havingValue = "liquibase")
class LiquibaseEnabled { /* пусто: автоконфигурация включит Liquibase */ }
```

**Kotlin — то же**

```kotlin
package com.example.migrations.profile

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Configuration

@Configuration
@ConditionalOnProperty("migrations.tool", havingValue = "flyway", matchIfMissing = true)
class FlywayEnabled

@Configuration
@ConditionalOnProperty("migrations.tool", havingValue = "liquibase")
class LiquibaseEnabled
```

**application.yml — выбрать инструмент**

```yaml
migrations:
  tool: flyway   # или liquibase
spring:
  flyway.enabled: true
  liquibase.enabled: false
```

---

## Форматы миграций: чистый SQL/Java (Flyway) vs XML/YAML/JSON/SQL (Liquibase)

Flyway «родным» считает **SQL-файлы** и **Java-based миграции**. SQL — самый распространённый путь: кладёте `V2__add_orders.sql`, пишете привычный DDL/DML; это прозрачно для ревью и легко исполняется всеми СУБД. Java-миграции выбирают для сложного DML/ETL, когда нужно программно управлять батчами, курсорами, логикой. Имя класса должно следовать схеме `Vx__Description`.

Liquibase поддерживает **четыре формата**: XML, YAML, JSON и SQL. На практике XML/YAML — основные: они декларативны, типизированы схемой XSD, удобны для `preconditions`, `rollback`, `contexts/labels`. SQL-формат уместен, когда команда не хочет уходить от чистого SQL, но при этом использует трекинг/историю Liquibase.

Плюсы декларативных форматов (Liquibase): переносимость, валидируемость (по XSD), выразительность (`createIndex`, `addNotNullConstraint`, `modifyDataType`), предикаты и откаты. Минусы — некоторый «шум» по сравнению с коротким SQL, плюс необходимость знать словарь тегов.

Плюсы SQL (Flyway): компактность, прозрачность плана, привычность. Минусы — откаты и условия чаще приходится писать вручную, а при желании «сухого прогона» нужен отдельный стенд, в отличие от Liquibase `updateSQL`.

Смешанный подход распространён: в Flyway большинство шагов — SQL, а «особые» — в Java; в Liquibase основа — YAML/XML, но для процедур/функций используют `createProcedure`/`sqlFile`.

Важно помнить: даже в Liquibase **всегда можно** использовать нативный SQL (`<sql>`/`<sqlFile>`), если фичи конкретной СУБД (например, Postgres `CONCURRENTLY`) не выразить декларативно.

**Flyway — SQL миграция**

```sql
-- src/main/resources/db/migration/V2__add_orders.sql
create table if not exists orders(
  id bigserial primary key,
  user_id bigint not null references users(id),
  total_cents bigint not null,
  created_at timestamptz not null default now()
);
```

**Flyway — Java-миграция (Java)**

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

public class V3__seed_orders extends BaseJavaMigration {
    @Override public void migrate(Context ctx) throws Exception {
        try (var st = ctx.getConnection().prepareStatement(
                "insert into orders(user_id,total_cents) values (?,?)")) {
            for (int i = 1; i <= 5; i++) {
                st.setLong(1, 1L);
                st.setLong(2, 100L * i);
                st.addBatch();
            }
            st.executeBatch();
        }
    }
}
```

**Flyway — Java-миграция (Kotlin)**

```kotlin
package db.migration

import org.flywaydb.core.api.migration.BaseJavaMigration
import org.flywaydb.core.api.migration.Context

class V3__seed_orders : BaseJavaMigration() {
    override fun migrate(context: Context) {
        context.connection.prepareStatement(
            "insert into orders(user_id,total_cents) values (?,?)"
        ).use { ps ->
            (1..5).forEach { i ->
                ps.setLong(1, 1)
                ps.setLong(2, 100L * i)
                ps.addBatch()
            }
            ps.executeBatch()
        }
    }
}
```

**Liquibase YAML — эквивалент создания таблицы**

```yaml
databaseChangeLog:
  - changeSet:
      id: 2-add-orders
      author: team
      changes:
        - createTable:
            tableName: orders
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: user_id, type: BIGINT, constraints: { nullable: false } }
              - column: { name: total_cents, type: BIGINT, constraints: { nullable: false } }
              - column: { name: created_at, type: TIMESTAMPTZ, defaultValueComputed: now(), constraints: { nullable: false } }
        - addForeignKeyConstraint:
            baseTableName: orders
            baseColumnNames: user_id
            referencedTableName: users
            referencedColumnNames: id
            constraintName: fk_orders_user
```

**Liquibase XML — тот же changeset**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="2-add-orders" author="team">
    <createTable tableName="orders">
      <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
      <column name="user_id" type="BIGINT"><constraints nullable="false"/></column>
      <column name="total_cents" type="BIGINT"><constraints nullable="false"/></column>
      <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>
    <addForeignKeyConstraint baseTableName="orders" baseColumnNames="user_id"
      referencedTableName="users" referencedColumnNames="id" constraintName="fk_orders_user"/>
  </changeSet>
</databaseChangeLog>
```

---

## Повторяемые миграции (R__) vs changeset’ы с `runOnChange/runAlways`

Repeatable-миграции Flyway (`R__...`) идеальны для объектов, которые логично **пересоздавать при изменении**: представления, функции, триггеры. Flyway хранит checksum содержимого файла; если он изменился — миграция будет выполнена повторно **после** всех версионных `V__`. Так обеспечивается синхронизация логических объектов без увеличения номера версии.

В Liquibase аналог — changeset с `runOnChange="true"`: он будет повторно выполняться при изменении тела. Для объектов, которые должны выполняться каждый раз (например, очистка и повторная загрузка материализованного представления на stage), есть `runAlways="true"` — но в проде им злоупотреблять не стоит.

Важный нюанс: повторяемые миграции **не для опасного DDL**, влияющего на данные. Если вы меняете тип колонки — это должна быть **версионная** миграция. Повторяемые хороши там, где операция идемпотентна и не ломает модель.

Опыт показывает, что `R__`-файлы лучше **разносить** по объектам: один файл — одна view/функция. Это повышает предсказуемость и облегчает ревью. В Liquibase — аналогично: один changeset на объект с `runOnChange`.

Ещё рекомендация — для тяжёлых объектов добавлять `preconditions` (Liquibase), чтобы, например, не пересоздавать view в dev «каждую минуту» без реальной надобности. В Flyway такой контроль реализуют организационно (код-ревью).

Наблюдаемость: во Flyway видно, когда repeatable переигрывался (состояние `Success` и новая checksum). В Liquibase — в `databasechangelog` есть запись применения changeset’а; при `runOnChange` md5 меняется.

**Flyway — R__ для VIEW**

```sql
-- src/main/resources/db/migration/R__view_active_orders.sql
create or replace view v_active_orders as
select o.id, o.total_cents
from orders o
join users u on u.id = o.user_id
where o.total_cents > 0;
```

**Liquibase YAML — runOnChange для VIEW**

```yaml
databaseChangeLog:
  - changeSet:
      id: view-active-orders
      author: team
      runOnChange: true
      changes:
        - createView:
            viewName: v_active_orders
            replaceIfExists: true
            selectQuery: |
              select o.id, o.total_cents
              from orders o
              join users u on u.id = o.user_id
              where o.total_cents > 0
```

**Liquibase XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                    http://www.liquibase.org/xml/ns/dbchangelog
                    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="view-active-orders" author="team" runOnChange="true">
    <createView viewName="v_active_orders" replaceIfExists="true">
      <selectQuery>
        select o.id, o.total_cents
        from orders o
        join users u on u.id = o.user_id
        where o.total_cents > 0
      </selectQuery>
    </createView>
  </changeSet>
</databaseChangeLog>
```

**Java — напечатать, какие repeatable применены (Flyway API)**

```java
package com.example.repeatable;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class RepeatableInfo {
    @Bean
    CommandLineRunner printRepeatables(Flyway flyway) {
        return args -> {
            for (MigrationInfo mi : flyway.info().all()) {
                if (mi.getVersion() == null) {
                    System.out.printf("Repeatable: %s | state=%s%n", mi.getDescription(), mi.getState());
                }
            }
        };
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.repeatable

import org.flywaydb.core.Flyway
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class RepeatableInfo {
    @Bean
    fun printRepeatables(flyway: Flyway) = CommandLineRunner {
        flyway.info().all().filter { it.version == null }
            .forEach { println("Repeatable: ${it.description} | state=${it.state}") }
    }
}
```

---

## Rollback: явные сценарии в Liquibase vs обычно «вперёд только» во Flyway

Liquibase предоставляет **первоклассные откаты**. В каждом changeset вы можете описать `<rollback>` (или `rollback:` в YAML) — явные операции, которые вернут схему/данные в прежнее состояние. Кроме того, вы можете ставить теги (`<tagDatabase tag="..."/>`) и потом делать `rollbackToTag`. Это особенно полезно для «каскадов» изменений, когда нужно гарантированно вернуться к точке до релиза.

Flyway философски придерживается направления **«только вперёд»**. Формального rollback нет (есть коммерческие расширения, но базово — нет). Если шаг оказался неудачным, подход — **forward-fix**: вы добавляете новую миграцию, которая корректирует схему/данные в правильное состояние. Такой подход проще и безопаснее в многокомандной разработке, но не даёт «мгновенного» отката.

Для регуляторных отраслей или сценариев с высокой стоимостью простоя откаты Liquibase — сильный аргумент. Важно, однако, писать **реалистичные** откаты: удаление столбца с потерей данных может быть необратимым; значит, rollback должен быть «миграцией совместимости», а не иллюзией.

В CI полезно иметь два шага Liquibase: `updateSQL` (сгенерировать SQL без применения) — на ревью DBA, и `update` — реальное применение. А перед критичными релизами — фиксировать тег и хранить его в release-манифесте.

Если вы остались на Flyway, предусмотрите операционные практики «быстрых исправлений» (emergency-скрипты), но держите их **в репозитории** и синхронизируйте в ближайшем релизе.

**Liquibase YAML — changeset с rollback и tag**

```yaml
databaseChangeLog:
  - changeSet:
      id: 4-add-column-status
      author: team
      changes:
        - addColumn:
            tableName: orders
            columns:
              - column: { name: status, type: TEXT, defaultValue: "NEW" }
      rollback:
        - dropColumn:
            columnName: status
            tableName: orders
  - changeSet:
      id: 4-tag
      author: team
      changes:
        - tagDatabase:
            tag: release-1.1
```

**Liquibase XML — откат к тегу (используется из кода)**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="4-add-column-status" author="team">
    <addColumn tableName="orders">
      <column name="status" type="TEXT" defaultValue="NEW"/>
    </addColumn>
    <rollback><dropColumn tableName="orders" columnName="status"/></rollback>
  </changeSet>
  <changeSet id="4-tag" author="team">
    <tagDatabase tag="release-1.1"/>
  </changeSet>
</databaseChangeLog>
```

**Java — программный rollback Liquibase к тегу**

```java
package com.example.rollback;

import liquibase.Liquibase;
import liquibase.database.jvm.JdbcConnection;
import liquibase.resource.ClassLoaderResourceAccessor;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;

@Component
public class LiquibaseRollback {
    private final DataSource ds;
    public LiquibaseRollback(DataSource ds) { this.ds = ds; }

    public void rollbackTo(String tag) throws Exception {
        try (Connection c = ds.getConnection()) {
            var lb = new Liquibase("db/changelog/db.changelog.yaml",
                    new ClassLoaderResourceAccessor(), new JdbcConnection(c));
            lb.rollback(tag, null);
        }
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.rollback

import liquibase.Liquibase
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import org.springframework.stereotype.Component
import javax.sql.DataSource

@Component
class LiquibaseRollback(private val ds: DataSource) {
    fun rollbackTo(tag: String) {
        ds.connection.use { c ->
            val lb = Liquibase(
                "db/changelog/db.changelog.yaml",
                ClassLoaderResourceAccessor(),
                JdbcConnection(c)
            )
            lb.rollback(tag, null)
        }
    }
}
```

**Flyway (Java/Kotlin) — «вперёд-фикс»: просто применяем следующую миграцию**

```java
package com.example.forwardfix;

import org.flywaydb.core.Flyway;

public class ForwardFix {
    public static void main(String[] args) {
        Flyway.configure()
              .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
              .locations("classpath:db/migration")
              .load()
              .migrate();
    }
}
```

```kotlin
package com.example.forwardfix

import org.flywaydb.core.Flyway

fun main() {
    Flyway.configure()
        .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
        .locations("classpath:db/migration")
        .load()
        .migrate()
}
```

---

## Инструменты сравнения/генерации: `diffChangeLog/generateChangeLog` в Liquibase

Сильная сторона Liquibase — умение **сравнивать** фактическую схему БД с эталонным changelog’ом и **генерировать** changelog из существующей БД. Команды `generateChangeLog` (полный changelog для текущей схемы) и `diffChangeLog` (разница между двумя источниками) помогают «подтянуть» легаси под управление миграциями или регулярно проверять дрейф продакшн-схемы.

Использовать `generateChangeLog` стоит осторожно: он создаёт огромный файл, который не всегда отражает «архитектурное намерение» (имена констрейнтов, порядок объектов). Часто разумнее — сгенерировать **стартовый** changelog один раз (baseline), а дальше писать changeset’ы вручную. `diffChangeLog` полезен как nightly-проверка: нет ли неожиданных объектов/индексов на проде.

В CI это можно поставить отдельной задачей: подключиться к staging/production read-only, сгенерировать diff и упасть, если есть неожиданные изменения. Это формализует борьбу с «ручными фикcами».

Для команд, где DBA ведёт схему в «графическом» инструменте, `diffChangeLog` помогает синхронизировать изменения в репозиторий приложения. Но обязательно ревью: autogenerate не заменяет человеческую логику (например, «этот индекс не нужен»).

Flyway аналогичных встроенных функций не имеет; практический путь — использовать `pg_dump --schema-only` и сравнивать дампы, или просто дисциплинированно держать схему «под миграциями».

**Gradle (Groovy) — Liquibase plugin для generate/diff**

```groovy
plugins { id 'org.liquibase.gradle' version '2.2.0' }
liquibase {
  activities {
    main {
      url System.getenv('DB_URL')
      username System.getenv('DB_USER')
      password System.getenv('DB_PASS')
      changeLogFile 'src/main/resources/db/changelog/db.changelog.yaml'
    }
  }
  runList = 'main'
}
tasks.register('lbGenerate') {
  dependsOn 'update' // или отдельно
  doLast {
    // gradle liquibaseGenerateChangeLog -PliquibaseCommandValue=src/main/resources/db/changelog/generated.yaml
  }
}
```

**Gradle (Kotlin)**

```kotlin
plugins { id("org.liquibase.gradle") version "2.2.0" }
liquibase {
  activities.register("main") {
    this.arguments = mapOf(
      "url" to System.getenv("DB_URL"),
      "username" to System.getenv("DB_USER"),
      "password" to System.getenv("DB_PASS"),
      "changeLogFile" to "src/main/resources/db/changelog/db.changelog.yaml"
    )
  }
  runList = "main"
}
```

**Java — программный вызов `diffChangeLog` (core API)**

```java
package com.example.diff;

import liquibase.database.DatabaseFactory;
import liquibase.diff.output.changelog.DiffToChangeLog;
import liquibase.diff.DiffGeneratorFactory;
import liquibase.resource.ClassLoaderResourceAccessor;

import javax.sql.DataSource;

public class DiffTool {
    public static String diffToChangeLog(DataSource ds1, DataSource ds2) throws Exception {
        var acc = new ClassLoaderResourceAccessor();
        try (var c1 = ds1.getConnection(); var c2 = ds2.getConnection()) {
            var db1 = DatabaseFactory.getInstance()
                    .findCorrectDatabaseImplementation(new liquibase.database.jvm.JdbcConnection(c1));
            var db2 = DatabaseFactory.getInstance()
                    .findCorrectDatabaseImplementation(new liquibase.database.jvm.JdbcConnection(c2));
            var diff = DiffGeneratorFactory.getInstance().compare(db1, db2, null);
            var changeLog = new DiffToChangeLog(diff);
            var out = new java.io.StringWriter();
            changeLog.print(out);
            return out.toString();
        }
    }
}
```

**Kotlin — то же (упрощённо)**

```kotlin
package com.example.diff

import liquibase.database.DatabaseFactory
import liquibase.diff.DiffGeneratorFactory
import liquibase.diff.output.changelog.DiffToChangeLog
import liquibase.database.jvm.JdbcConnection
import java.io.StringWriter
import javax.sql.DataSource

object DiffTool {
    fun diffToChangeLog(ds1: DataSource, ds2: DataSource): String {
        ds1.connection.use { c1 ->
            ds2.connection.use { c2 ->
                val db1 = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(c1))
                val db2 = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(c2))
                val diff = DiffGeneratorFactory.getInstance().compare(db1, db2, null)
                val out = StringWriter()
                DiffToChangeLog(diff).print(out)
                return out.toString()
            }
        }
    }
}
```

---

## Управление совместимостью: `out-of-order/baseline` (Flyway) vs `labels/contexts` (Liquibase)

В реальной жизни ветки фич могут жить параллельно, и на разных стендах миграции применяются не строго по порядку версий. Во Flyway для этого есть `outOfOrder=true`: при старте он применит пропущенные `V__`, даже если их версия «младше» уже применённых. Это удобно на stage, но требует строгой дисциплины именований и понимания «какие side-effect’ы в каких версиях».

`baseline` во Flyway позволяет принять существующую схему за «стартовую версию» (например, `1`). Инструмент создаст запись в `flyway_schema_history`, а дальше вы пойдёте `V2+`. Это путь интеграции легаси без реплея «всей истории».

В Liquibase **contexts/labels** дают тонкое управление применением changeset’ов по средам/фичам. Например, seed-данные только для `stage`, или changeset «под фичу X», который включается, когда флаг включён. Это полезно для контролируемых rollout’ов, A/B и «белых списков» клиентов.

Также Liquibase поддерживает `preconditions` — можно гарантировать, что changeset не пойдёт, если БД не в ожидаемом состоянии (например, слишком большая таблица — опасный шаг).

Комбинировать можно так: Flyway — на проектах, где история линейна и baseline решает подключение легаси; Liquibase — там, где важно «кому и когда» применять изменения. Оба инструмента хорошо работают со Spring-профилями и секретами окружений.

Важно: `outOfOrder` — не «палочка-выручалочка». Лучше **не допускать** долгого расхождения веток миграций и регулярно мёржить их в общую. `outOfOrder` оставляйте как временную подпорку.

**application.yml — включить `outOfOrder` и `baseline` во Flyway профилем**

```yaml
spring:
  flyway:
    enabled: true
    out-of-order: true
    baseline-on-migrate: true
    baseline-version: 1
```

**Java — настроить это программно (Customizer)**

```java
package com.example.flywaycfg;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class FlywayCompatConfig {
    @Bean
    FlywayConfigurationCustomizer customizer() {
        return (FluentConfiguration c) -> c.outOfOrder(true).baselineOnMigrate(true).baselineVersion("1");
    }
}
```

**Kotlin — Customizer**

```kotlin
package com.example.flywaycfg

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayCompatConfig {
    @Bean
    fun customizer() = FlywayConfigurationCustomizer { c: FluentConfiguration ->
        c.outOfOrder(true).baselineOnMigrate(true).baselineVersion("1")
    }
}
```

**Liquibase YAML — contexts/labels**

```yaml
databaseChangeLog:
  - changeSet:
      id: 5-seed-stage-only
      author: team
      context: stage
      changes:
        - insert:
            tableName: feature_flags
            columns:
              - column: { name: key, value: "beta_users" }
              - column: { name: enabled, valueBoolean: true }
  - changeSet:
      id: 6-feature-x
      author: team
      labels: feature-x
      changes:
        - addColumn:
            tableName: users
            columns:
              - column: { name: x_data, type: JSONB }
```

**Liquibase XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="5-seed-stage-only" author="team" context="stage">
    <insert tableName="feature_flags">
      <column name="key" value="beta_users"/>
      <column name="enabled" valueBoolean="true"/>
    </insert>
  </changeSet>
  <changeSet id="6-feature-x" author="team" labels="feature-x">
    <addColumn tableName="users">
      <column name="x_data" type="JSONB"/>
    </addColumn>
  </changeSet>
</databaseChangeLog>
```

---

## Когда что выбрать: команда, стек, требования к откатам/диффам, культура SQL

Выбор инструмента — это не «что лучше вообще», а «что лучше **для вашей команды и процесса**». Если у вас сильная культура SQL, простой жизненный цикл, «вперёд-фикс» и нет жёстких требований к откатам — **Flyway** даст скорость и простоту. Вы будете читать историю как список версий, держать R__ для view/func, и жить спокойно.

Если у вас формальный change-management, требование к **явным rollback**, окружения с разным набором изменений, nightly-проверка дрейфа — **Liquibase** даст нужные рычаги: `rollback`, `updateSQL`, `contexts/labels`, `diffChangeLog`. Он же помогает «поднять» легаси через `generateChangeLog`, хотя применять autogenerate без ревью не стоит.

Смешанный путь тоже возможен (но держите один инструмент **на сервис**): платформа использует Liquibase для централизованных изменений, а отдельные микросервисы — Flyway для своих локальных схем. Важно не смешивать их в одном модуле, чтобы не возникало гонок за историю.

Ещё критерий — навыки команды. Если Java/Kotlin — ваш основной язык и вы хотите часть миграций писать в коде (батчи, контроль курсоров), Flyway Java-based миграции очень удобны. В Liquibase тоже есть расширяемость, но порог входа выше.

Наконец, обратите внимание на **операционные требования**: в Kubernetes проще запускать миграции как Job с Liquibase `updateSQL`/`update`, с тегами и откатами. Но и Flyway-добегушка (маленький Java main) отлично живёт как initContainer/Job.

Любой выбор закрепляйте **гайдом** в репозитории: схема именования, правила baseline/out-of-order/rollback, как запускать локально, как устроен CI/CD, где смотреть логи и как поступать при сбое. Инструмент без дисциплины не решит проблему.

**application.yml — шаблон переключения (решение «на сервис»)**

```yaml
migrations:
  tool: flyway   # или liquibase
spring:
  flyway:
    enabled: ${migrations.tool:flyway} == 'flyway'
  liquibase:
    enabled: ${migrations.tool:flyway} == 'liquibase'
```

**Java — «единая кнопка» для запуска миграций в отдельном Job’е**

```java
package com.example.choice;

import org.flywaydb.core.Flyway;
import liquibase.Liquibase;
import liquibase.database.jvm.JdbcConnection;
import liquibase.resource.ClassLoaderResourceAccessor;

import javax.sql.DataSource;

public class MigrationRunner {
    public static void run(DataSource ds, String tool) throws Exception {
        if ("flyway".equals(tool)) {
            Flyway.configure().dataSource(ds).locations("classpath:db/migration").load().migrate();
        } else {
            try (var c = ds.getConnection()) {
                new Liquibase("db/changelog/db.changelog.yaml",
                    new ClassLoaderResourceAccessor(), new JdbcConnection(c)).update((String) null);
            }
        }
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.choice

import liquibase.Liquibase
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import org.flywaydb.core.Flyway
import javax.sql.DataSource

object MigrationRunner {
    @JvmStatic
    fun run(ds: DataSource, tool: String) {
        if (tool == "flyway") {
            Flyway.configure().dataSource(ds).locations("classpath:db/migration").load().migrate()
        } else {
            ds.connection.use { c ->
                Liquibase(
                    "db/changelog/db.changelog.yaml",
                    ClassLoaderResourceAccessor(),
                    JdbcConnection(c)
                ).update(null as String?)
            }
        }
    }
}
```

**Liquibase YAML и XML — один короткий changeset «для демонстрации выбора»**

```yaml
databaseChangeLog:
  - changeSet:
      id: demo-choice
      author: team
      changes:
        - createTable:
            tableName: demo_choice
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                    http://www.liquibase.org/xml/ns/dbchangelog
                    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="demo-choice" author="team">
    <createTable tableName="demo_choice">
      <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
    </createTable>
  </changeSet>
</databaseChangeLog>
```

**Gradle (оба DSL) — задачи для отдельного шага в CI**

*Groovy*

```groovy
plugins {
  id 'org.flywaydb.flyway' version '10.16.0'
  id 'org.liquibase.gradle' version '2.2.0'
}
tasks.register('migrateFlyway') {
  doLast {
    org.flywaydb.core.Flyway.configure()
      .dataSource(System.getenv('DB_URL'), System.getenv('DB_USER'), System.getenv('DB_PASS'))
      .locations('classpath:db/migration').load().migrate()
  }
}
tasks.register('migrateLiquibase') {
  doLast {
    ant.invokeMethod('liquibase', [
      'changeLogFile': 'src/main/resources/db/changelog/db.changelog.yaml',
      'url'          : System.getenv('DB_URL'),
      'username'     : System.getenv('DB_USER'),
      'password'     : System.getenv('DB_PASS'),
      'logLevel'     : 'info'
    ])
  }
}
```

*Kotlin*

```kotlin
plugins {
  id("org.flywaydb.flyway") version "10.16.0"
  id("org.liquibase.gradle") version "2.2.0"
}
tasks.register("migrateFlyway") {
  doLast {
    org.flywaydb.core.Flyway.configure()
      .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
      .locations("classpath:db/migration").load().migrate()
  }
}
tasks.register("migrateLiquibase") {
  doLast {
    // Лучше использовать plugin DSL; показан как пример
    ant.invokeMethod("liquibase", mapOf(
      "changeLogFile" to "src/main/resources/db/changelog/db.changelog.yaml",
      "url" to System.getenv("DB_URL"),
      "username" to System.getenv("DB_USER"),
      "password" to System.getenv("DB_PASS"),
      "logLevel" to "info"
    ))
  }
}
```

Итог: оба инструмента зрелые и отлично поддерживаются Spring Boot. Выбор определяют требования к откатам, диффам и управлению «кому/когда» применять изменения, а также культура владения SQL в команде. Важно выбрать **один** инструмент на сервис, зафиксировать правила и автоматизировать их в CI/CD.

# 3. Интеграция со Spring Boot: зависимости и конфигурация

## Подключение стартеров и базовые properties (`spring.flyway.*` / `spring.liquibase.*`)

Начнём с самого простого — как «подружить» миграции и Spring Boot. Благо Boot уже содержит автоконфигурации для обоих инструментов и создаёт нужные бины автоматически, если в classpath присутствуют зависимости и настроен `DataSource`. В 99% случаев вам достаточно добавить стартеры и пару свойств в `application.yml`: включить один инструмент и выключить второй (не держите оба активными одновременно в одном сервисе).

Важная производственная привычка — **сразу определиться, кто управляет схемой**. Если вы используете миграции, Hibernate DDL auto должен быть выключен (никаких `update/create`). Это исключает конфликты «инструментов» и делает схему детерминированной: только миграции, только хардкор.

Стоит также определиться с **местом хранения миграций**. Для Flyway Boot по умолчанию ищет их в `classpath:db/migration`. Для Liquibase — нужен master changelog (обычно `classpath:db/changelog/db.changelog.yaml` или `.xml`). Если структура нестандартная, укажите `spring.flyway.locations` или `spring.liquibase.change-log`.

Из полезных базовых флагов у Flyway: `validate-on-migrate` (валидировать checksum’ы при старте), `clean-disabled` (запретить опасный `clean` в проде), `out-of-order` (разрешить поздние версии, только осознанно). У Liquibase аналогичная «строгость» достигается через стандартный запуск `update` + использование `preconditions` и валидацию changelog’а.

На dev обычно удобно запускать миграции **на старте приложения** (мигратор применится до инициализации JPA). На prod всё чаще миграции выносят в отдельный шаг пайплайна/Job — но интеграция со Spring всё равно остаётся: одно и то же приложение может запускать Liquibase/Flyway, просто флажок `enabled` выключают.

Наконец, держите в памяти, что для обоих инструментов Spring Boot уже экспонирует бины API (например, `Flyway`), так что вы можете программно спросить статус (`info()`) и вывести его в логи/метрики. Это поможет в наблюдаемости и тревогах.

**Gradle зависимости — Groovy DSL (Boot 3.x, PG, оба инструмента — включаем по флагу)**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.flywaydb:flyway-core'
  implementation 'org.liquibase:liquibase-core'
  runtimeOnly   'org.postgresql:postgresql:42.7.3'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle зависимости — Kotlin DSL**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.flywaydb:flyway-core")
  implementation("org.liquibase:liquibase-core")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**application.yml — базовая конфигурация (включаем Flyway, отключаем Liquibase)**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: none  # схему меняют миграции
  flyway:
    enabled: true
    locations: classpath:db/migration
    validate-on-migrate: true
    clean-disabled: true
  liquibase:
    enabled: false
```

**Java — показать статус миграций Flyway на старте**

```java
package com.example.migrations;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayStatusConfig {
  @Bean
  CommandLineRunner printStatus(Flyway flyway) {
    return args -> {
      System.out.println("== Flyway status ==");
      for (MigrationInfo mi : flyway.info().all()) {
        System.out.printf("%s | %s | %s%n", mi.getVersion(), mi.getDescription(), mi.getState());
      }
    };
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.migrations

import org.flywaydb.core.Flyway
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayStatusConfig {
    @Bean
    fun printStatus(flyway: Flyway) = CommandLineRunner {
        println("== Flyway status ==")
        flyway.info().all().forEach { mi ->
            println("${mi.version} | ${mi.description} | ${mi.state}")
        }
    }
}
```

---

## Папки и соглашения о расположении миграций (classpath, multiple locations)

Чёткие соглашения о структуре — ваш антидот от хаоса. Для Flyway де-факто стандарт: `src/main/resources/db/migration` для версионных `V__` и повторяемых `R__`. Если проект модульный, удобно добавить подкаталоги: `db/migration/core`, `db/migration/billing`. Flyway поддерживает **несколько локаций**; упорядочивание происходит по имени файла, а не по локации.

Liquibase живёт вокруг **master changelog**. Рекомендуемая структура: `db/changelog/db.changelog.yaml` (или `.xml`) и подкаталоги с доменными файлами, которые подключаются через `include`/`includeAll`. Так проще делить ответственность и уменьшать конфликты при работе нескольких команд.

Multiple locations полезны, когда часть миграций «общая для платформы», а часть — «локальная для сервиса». Для Flyway можно указать `classpath:db/migration,classpath:platform/migration`. Для Liquibase — сделать master, который `includeAll` из нескольких папок.

Будьте аккуратны с `filesystem:`-локациями. Они годятся для локальной отладки/внешнего тарболла, но в CI/CD лучше всё держать в classpath-ресурсах, чтобы артефакт был самодостаточным. Если всё же используете `filesystem`, убедитесь в надёжной доставке каталогов.

Ещё один паттерн — разделить DDL и heavy DML по разным каталогам и включать их разными профилями/контекстами. Например, `db/migration` — только DDL, а `db/seed` — сиды для stage/dev.

Наконец, договоритесь об именовании файлов: коротко и по делу, без пробелов, с глаголом действия: `V05__orders_add_status.sql` лучше, чем `V5.sql`. Это ускоряет ревью и поиск в репозитории.

**Flyway — несколько локаций (application.yml)**

```yaml
spring:
  flyway:
    enabled: true
    locations:
      - classpath:db/migration
      - classpath:platform/migration
```

**Java — программно добавить локации (Customizer)**

```java
package com.example.locations;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayLocationsConfig {
  @Bean
  FlywayConfigurationCustomizer locations() {
    return (FluentConfiguration c) ->
        c.locations("classpath:db/migration", "classpath:platform/migration");
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.locations

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayLocationsConfig {
    @Bean
    fun locations() = FlywayConfigurationCustomizer { c: FluentConfiguration ->
        c.locations("classpath:db/migration", "classpath:platform/migration")
    }
}
```

**Liquibase — master YAML с include’ами по папкам**

```yaml
# src/main/resources/db/changelog/db.changelog.yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/core
  - includeAll:
      path: db/changelog/billing
```

**Liquibase — master XML (эквивалент)**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <includeAll path="db/changelog/core"/>
  <includeAll path="db/changelog/billing"/>
</databaseChangeLog>
```

---

## Автоконфигурация: порядок инициализации мигратора до `EntityManagerFactory`

Spring Boot запускает миграции **до** создания `EntityManagerFactory`/`SessionFactory`. Это критично: когда JPA просканирует сущности и откроет соединение, схема уже должна соответствовать ожиданиям. Поэтому `ddl-auto` выключаем, а мигратор — оставляем включённым (или запускаем отдельным Job’ом до старта приложения).

Если вам нужно вмешаться в процесс, у Flyway есть `FlywayMigrationStrategy`: можно, например, перед миграцией запустить свои sanity-check’и или собрать метрику времени. В Liquibase аналог — программно объявить `SpringLiquibase` бином и управлять параметрами (включая контексты/лейблы).

Помните, что любые бины, зависящие от JPA (репозитории, EntityManager) поднимутся **после** мигратора. Если вы вдруг попробуете в `CommandLineRunner` обращаться к данным до завершения миграций — получите «схема не соответствует» или падение. Повесьте такой код после миграций (или в отдельный профиль).

Автоконфигурация наложена на `DataSource`. Если `DataSource` не собран (например, нет URL/пароля), мигратор не сработает. В контейнерах/K8s это важно: убедитесь, что секреты/конфиги подаются до старта.

При отдельной стадии миграций (Job/Hook) автоконфигурация может быть выключена свойством `enabled=false`, а Job будет использовать тот же classpath и тот же changelog. Это снижает дрейф между «режимами» — и хорошо.

Наконец, проверьте лог-строки старта: там явно виден факт запуска миграций и число применённых шагов. Это основной индикатор порядка и успеха.

**Java — `FlywayMigrationStrategy`: обернём миграции логикой/метрикой**

```java
package com.example.order;

import org.flywaydb.core.Flyway;
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationStrategy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayOrderConfig {

  @Bean
  FlywayMigrationStrategy strategy() {
    return (Flyway flyway) -> {
      long t0 = System.currentTimeMillis();
      System.out.println("Applying migrations...");
      var res = flyway.migrate();
      System.out.printf("Applied %d migrations in %d ms%n",
          res.migrationsExecuted, System.currentTimeMillis() - t0);
    };
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.order

import org.flywaydb.core.Flyway
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationStrategy
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayOrderConfig {
    @Bean
    fun strategy() = FlywayMigrationStrategy { flyway: Flyway ->
        val t0 = System.currentTimeMillis()
        println("Applying migrations...")
        val res = flyway.migrate()
        println("Applied ${res.migrationsExecuted} migrations in ${System.currentTimeMillis() - t0} ms")
    }
}
```

**Liquibase YAML/XML — master, который Boot запустит до JPA**

*YAML*

```yaml
databaseChangeLog:
  - changeSet:
      id: 1-init
      author: team
      changes:
        - createTable:
            tableName: init_marker
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
```

*XML*

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="1-init" author="team">
    <createTable tableName="init_marker">
      <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
    </createTable>
  </changeSet>
</databaseChangeLog>
```

---

## Управление включением/выключением миграций по профилям (`enabled`, `check-location`)

Часто нужно иметь разные режимы на средах: dev — on-start, stage — on-start, prod — отдельный Job. Удобный способ — профили Spring. В `application-prod.yml` просто выключаете автозапуск (`enabled: false`) и запускаете миграции из Job’а тем же артефактом. На dev имеет смысл включить `check-location=true` у Flyway: Boot упадёт, если папка миграций не найдена (ловит ошибки конфигурации).

Сценарии тестов (`@DataJpaTest`) обычно используют встроенный `DataSource` или Testcontainers. Здесь тоже работает автоконфигурация миграций — хорошо, если вы её не отключите случайно. Для slice-тестов удобно оставить Flyway включённым в `application-test.yml`.

Можно использовать и «фича-флаг» на уровне собственных свойств, а затем прокинуть его в `spring.flyway.enabled`/`spring.liquibase.enabled` через SpEL. Это даёт гибкость без дублирования файлов.

Отдельный случай — «песочница» разработчика: иногда нужно быстро прототипировать без миграций. Заводят профиль `no-mig` и ставят `enabled: false`. Главное — не коммитить такой профиль как дефолтный.

Если у вас в проекте два DataSource (read/write), убедитесь, что мигратор направлен на **write**-подключение и только он включён по профилю.

**application-dev.yml — включить on-start Flyway, проверять location**

```yaml
spring:
  profiles: dev
  flyway:
    enabled: true
    check-location: true
```

**application-prod.yml — выключить on-start (мигрируем Job’ом)**

```yaml
spring:
  profiles: prod
  flyway:
    enabled: false
  liquibase:
    enabled: false
```

**Java — включать инструмент по кастомному флагу**

```java
package com.example.profiles;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnProperty(name = "migrations.tool", havingValue = "flyway", matchIfMissing = true)
class FlywayOn {}

@Configuration
@ConditionalOnProperty(name = "migrations.tool", havingValue = "liquibase")
class LiquibaseOn {}
```

**Kotlin — то же**

```kotlin
package com.example.profiles

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Configuration

@Configuration
@ConditionalOnProperty(name = "migrations.tool", havingValue = "flyway", matchIfMissing = true)
class FlywayOn

@Configuration
@ConditionalOnProperty(name = "migrations.tool", havingValue = "liquibase")
class LiquibaseOn
```

---

## Доступ к нескольким схемам и `defaultSchema`/`schemas`

В PostgreSQL и ряде СУБД актуальна работа с несколькими схемами. Flyway умеет задавать **дефолтную схему** и список схем, за которыми он «следит» (история `flyway_schema_history` может жить в default schema). Это важно, если у вас есть `public` и, скажем, `billing`, и миграции создают объекты в обеих.

Liquibase идёт другим путём: вы можете указать `defaultSchema` глобально (в параметрах Spring/LIQUIBASE) и/или прямо в changeset’ах указывать `schemaName`/`catalogName`. Это гибче: часть объектов — в `public`, часть — в `billing`.

Следите за `search_path`/правами. Пользователь, под которым запускаются миграции, должен иметь права на нужные схемы. Если права разнесены (миграции делает «админ», приложение — «апп-юзер»), используйте отдельные креды на стадию миграций.

История миграций: у Flyway таблица истории обычно одна (в default schema), у Liquibase — тоже одна (в той схеме, где её создали). Если вы хотите отдельную историю на схему/тенанта — поднимайте миграции в цикле по схемам (advanced-сценарий).

Именуйте объекты с учётом схемы: в SQL миграциях явно указывайте `schema.object`, если создаёте вне дефолтной. Это исключит сюрпризы при изменении `search_path`.

Наконец, тестируйте multi-schema через Testcontainers и прогон миграций «с нуля»: это ловит как ошибки прав, так и опечатки в схемах.

**Flyway — несколько схем (application.yml)**

```yaml
spring:
  flyway:
    enabled: true
    default-schema: app
    schemas:
      - app
      - billing
```

**Java — задать default schema программно**

```java
package com.example.schemas;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywaySchemasConfig {
  @Bean
  FlywayConfigurationCustomizer schemas() {
    return (FluentConfiguration c) -> c.defaultSchema("app").schemas("app", "billing");
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.schemas

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywaySchemasConfig {
    @Bean
    fun schemas() = FlywayConfigurationCustomizer { c: FluentConfiguration ->
        c.defaultSchema("app").schemas("app", "billing")
    }
}
```

**Liquibase master — YAML с объектом в конкретной схеме**

```yaml
databaseChangeLog:
  - changeSet:
      id: 10-create-billing-table
      author: team
      changes:
        - createTable:
            schemaName: billing
            tableName: invoices
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: amount_cents, type: BIGINT, constraints: { nullable: false } }
```

**Liquibase master — XML эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="10-create-billing-table" author="team">
    <createTable schemaName="billing" tableName="invoices">
      <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
      <column name="amount_cents" type="BIGINT"><constraints nullable="false"/></column>
    </createTable>
  </changeSet>
</databaseChangeLog>
```

---

## Переменные/плейсхолдеры и секреты (placeholders, Spring placeholders)

Оба инструмента поддерживают **подстановку значений**. Во Flyway это `placeholders`: вы объявляете пары `key=value` в конфиге, а в SQL используете `${key}`. Это удобно для имён схем, префиксов индексов, флагов вроде `CONCURRENTLY`. У Liquibase свойства можно задавать на уровне Spring (`spring.liquibase.parameters.*`), через `<property>` в changelog или через `${...}` в значениях изменений.

Важная мысль: не путайте плейсхолдеры миграторов и **Spring property placeholders**. В `application.yml` вы можете писать `${ENV_VAR:default}` — это подставится ещё до того, как Flyway/Liquibase прочтут свои настройки. Полезный паттерн — хранить секреты (пароли) в переменных окружения и ссылаться на них из Spring.

Для Liquibase YAML/XML можно объявить `<property name="...">` (или `- property:` в YAML) в начале master-файла, чтобы использовать его по всему changelog. Так вы централизуете параметры среды: имя схемы, суффиксы и т.п.

Секреты: никогда не кладите пароли в changelog/SQL. Spring Boot `DataSource` уже настроен (секреты в `K8s Secret`/Vault), мигратор возьмёт его автоматически. Если нужен доступ «с повышенными правами» — заведите отдельный `DataSource` для миграций и используйте его в отдельном Job’е.

Следите за **экранированием**. Плейсхолдеры в SQL должны быть совместимы с синтаксисом СУБД. Для текстовых значений используйте кавычки в SQL, не полагайтесь, что инструмент их добавит.

И наконец, договоритесь о префиксах/именах. `${app.schema}` более читаемо, чем `${s}`.

**Flyway placeholders — application.yml + SQL**

```yaml
spring:
  flyway:
    enabled: true
    placeholders:
      app_schema: app
      idx_concurrently: "CONCURRENTLY"
```

```sql
-- V20__add_index_with_placeholder.sql
create index ${idx_concurrently} if not exists idx_users_email
  on ${app_schema}.users (email);
```

**Liquibase YAML — параметры через `parameters` Spring и использование**

```yaml
# application.yml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog.yaml
    parameters:
      appSchema: app
```

```yaml
# db.changelog.yaml
databaseChangeLog:
  - changeSet:
      id: 20-add-index
      author: team
      changes:
        - createIndex:
            tableName: users
            schemaName: ${appSchema}
            indexName: idx_users_email
            columns:
              - column: { name: email }
```

**Liquibase XML — эквивалент с `<property>`**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <property name="appSchema" value="app"/>
  <changeSet id="20-add-index" author="team">
    <createIndex schemaName="${appSchema}" tableName="users" indexName="idx_users_email">
      <column name="email"/>
    </createIndex>
  </changeSet>
</databaseChangeLog>
```

**Java — отдельный `DataSource` для миграций (если нужна отдельная учётка)**

```java
package com.example.secrets;

import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.*;
import javax.sql.DataSource;

@Configuration
public class MigrationDsConfig {

  @Bean
  @ConfigurationProperties("app.migration.datasource")
  DataSourceProperties migrationDsProps() { return new DataSourceProperties(); }

  @Bean
  DataSource migrationDataSource(@Qualifier("migrationDsProps") DataSourceProperties props) {
    return props.initializeDataSourceBuilder().build();
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.secrets

import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class MigrationDsConfig {

    @Bean
    @ConfigurationProperties("app.migration.datasource")
    fun migrationDsProps() = DataSourceProperties()

    @Bean
    fun migrationDataSource(props: DataSourceProperties): DataSource =
        props.initializeDataSourceBuilder().build()
}
```

---

## Логи/уровни для наблюдаемости на старте: что считать «успешным запуском»

Критерий «здорового» старта: мигратор включён (или явно выключен по профилю), миграции применены успешно (0 pending), Hibernate не попытался менять схему, а приложение поднялось. Всё это должно быть видно в логах и/или доступно через health-эндпойнт.

Для начала включите адекватные уровни логов: `org.flywaydb`/`liquibase` на `INFO` или `DEBUG` в dev. На prod обычно достаточно `INFO`, но полезно собирать **время выполнения** миграций отдельной метрикой (в Prometheus/Logs). Это сигналит о «потяжелении» DDL/DML.

Полезно на старте распечатать (`CommandLineRunner`) число pending миграций. Если их > 0 при включённом on-start — это подозрительно (значит, мигратор не сработал). Если миграции выключены — это ожидаемо (их применит Job).

С точки зрения ошибок, «успешный запуск» — это отсутствие checksum mismatch и отсутствие исключений `ValidationFailedException` (Flyway) / `ValidationErrors` (Liquibase). Хорошая практика — в dev падать немедленно, если валидация не прошла.

В Kubernetes можно прикрутить **readiness probe**, которая проверяет, что миграций pending нет (или что Liquibase/Flyway отключены). Это защитит от трафика на полуподнятые инстансы.

Для аудита полезно логировать список применённых версий/changeset’ов на уровне `INFO` при каждом старте. Не стесняйтесь быть многословными в dev — это экономит часы расследований.

**application.yml — уровни логов**

```yaml
logging:
  level:
    org.flywaydb: INFO
    liquibase: INFO
```

**Java — health-like эндпойнт для Flyway**

```java
package com.example.health;

import org.flywaydb.core.Flyway;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

record FlywayHealth(int pending, int applied) {}

@RestController
public class FlywayHealthController {
  private final Flyway flyway;
  public FlywayHealthController(Flyway flyway) { this.flyway = flyway; }

  @GetMapping("/internal/flyway")
  public FlywayHealth health() {
    var info = flyway.info();
    return new FlywayHealth(info.pending().length, info.applied().length);
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.health

import org.flywaydb.core.Flyway
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

data class FlywayHealth(val pending: Int, val applied: Int)

@RestController
class FlywayHealthController(private val flyway: Flyway) {
    @GetMapping("/internal/flyway")
    fun health(): FlywayHealth {
        val info = flyway.info()
        return FlywayHealth(info.pending().size, info.applied().size)
    }
}
```

**Liquibase YAML/XML — маленький changeset для проверки логов**

*YAML*

```yaml
databaseChangeLog:
  - changeSet:
      id: 99-health-marker
      author: team
      changes:
        - createTable:
            tableName: health_marker
            columns:
              - column: { name: id, type: BIGINT, constraints: { primaryKey: true } }
```

*XML*

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="99-health-marker" author="team">
    <createTable tableName="health_marker">
      <column name="id" type="BIGINT"><constraints primaryKey="true"/></column>
    </createTable>
  </changeSet>
</databaseChangeLog>
```

Итого, интеграция со Spring Boot состоит из трёх опор: корректные зависимости и включение одного инструмента, прозрачная структура и локации миграций, плюс строгие правила запуска/логирования. С этими кирпичиками миграции становятся такой же естественной частью старта, как создание `EntityManagerFactory`.

# 4. Flyway — базовая практика

> Ниже — практические приёмы Flyway в Spring Boot-проектах. Для полноты приведу и конфигурации (YAML), и небольшие рабочие фрагменты кода **на Java и Kotlin**. Для сборки — оба варианта Gradle.

**Gradle: зависимости**

*Groovy DSL*

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.flywaydb:flyway-core'
  runtimeOnly   'org.postgresql:postgresql:42.7.3'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

*Kotlin DSL*

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.flywaydb:flyway-core")
  runtimeOnly("org.postgresql:postgresql:42.7.3")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

---

## Типы миграций: `V__` (versioned) и `R__` (repeatable), когда использовать каждый

Версионные миграции `V__` — «ступеньки истории»: применяются **один раз** в порядке версий. Это основа эволюции схемы (создание таблиц/индексов, добавление колонок, изменение ограничений). Их сила — детерминированность: вы всегда знаете, в каком порядке выполнятся изменения. В составе Spring Boot они запускаются до инициализации JPA, так что код видит уже обновлённую схему.

Повторяемые миграции `R__` не имеют номера версии и привязаны к **контрольной сумме** содержимого файла. Изменили файл — контрольная сумма изменилась — Flyway переисполнит этот файл **после** всех `V__`. Это удобно для объектов, которые безопасно «пересоздавать»: представления (VIEW), функции (FUNCTION), триггеры, политики RLS. Часто именно они «эволюционируют» чаще, чем DDL таблиц.

Правильный выбор типа — вопрос безопасности. Если изменение может повлиять на пользовательские данные (например, смена типа колонки), используйте **только** `V__`. Если это материализация логики чтения (VIEW) или процедурная обвязка, удобно `R__`. Такой расклад ясен ревьюерам и снижает риск «скрытых» побочных эффектов.

Ещё одно различие — читаемость истории. Длинный список `V__` показывает хронологию схемы, а `R__` — текущую актуальную версию логических объектов. На PR удобно видеть и то, и другое: «одна версия» плюс «обновление представления». Это повышает прозрачность изменений.

На практике встречается комбинация: «сначала V** для создания VIEW (если её не было), потом `R__` для дальнейших правок». Но чаще проще сразу жить с `R__ create or replace view ...`, чтобы повторное применение было естественным. Главное — договориться в команде и закрепить правило в README.

Наконец, в проде старайтесь избегать тяжелых `R__`, которые пересоздают большие объекты (например, материализованные представления) без нужды. Если пересоздание затратное — переводите его в явный `V__` шаг, чтобы контролировать момент и окно выполнения.

**SQL-файлы (пример):**

```sql
-- src/main/resources/db/migration/V1__init.sql
create table if not exists users(
  id bigserial primary key,
  email text not null unique,
  created_at timestamptz not null default now()
);
```

```sql
-- src/main/resources/db/migration/R__create_view_active_users.sql
create or replace view v_active_users as
select id, email from users where email ~ '@';
```

**Java — запуск миграций и печать статуса**

```java
package com.example.flywaydemo;

import org.flywaydb.core.Flyway;

public class FlywayMain {
    public static void main(String[] args) {
        Flyway fw = Flyway.configure()
                .dataSource("jdbc:postgresql://localhost:5432/app","app","app")
                .locations("classpath:db/migration")
                .load();
        var res = fw.migrate();
        System.out.println("Applied: " + res.migrationsExecuted);
        fw.info().all(); // можно посмотреть R__ и V__ вместе
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.flywaydemo

import org.flywaydb.core.Flyway

fun main() {
    val fw = Flyway.configure()
        .dataSource("jdbc:postgresql://localhost:5432/app","app","app")
        .locations("classpath:db/migration")
        .load()
    val res = fw.migrate()
    println("Applied: ${res.migrationsExecuted}")
    fw.info().all()
}
```

---

## Правила версионирования и именования файлов, префиксы/суффиксы, порядок

Самая частая причина путаницы — несогласованные названия файлов. Базовое правило Flyway: `V<Version>__<Description>.sql` (двойное подчёркивание между номером и описанием). Номер может быть простым (`V5__...`) или составным (`V2_1__...`). Описание — «короткое английское имя», без пробелов, информативное: `add_orders_table`, `add_idx_users_email`. Чем яснее имя — тем проще ревью.

Порядок выполнения определяется **сортировкой по версии**, а не по времени коммита. Это важно в командах с несколькими ветками: перед merge вы согласовываете номера (или доверяете Flyway `outOfOrder`, см. ниже, но осознанно). Для `R__` порядок — лексикографический по имени файла, применяются после всех `V__`.

Префиксы/суффиксы и места поиска можно кастомизировать. Например, хранить SQL с расширением `.pgsql` и поменять `sqlMigrationSuffixes`. Хороший кейс — разделить `V__` и `R__` по подпапкам команд/домена. Так легче избегать конфликтов между командами в монорепозитории.

Именование — это ещё и дисциплина мёрджа. Хорошая практика — зарезервировать «окно версий» (например, команда Billing держит V200–V299) или добавлять версию в момент merge (бот переименовывает `V__TBD__...` → `V37__...`). Любой автоматизм лучше ручного «ловим конфликты в последний момент».

Если нужны несколько суффиксов (например, `.sql` и `.psql`), их можно указать списком. Но помните: при одинаковых версиях и разных файлах Flyway не поймёт «какой главный». Не создавайте «дублирующие» номера.

Наконец, старайтесь не «переименовывать историю». Уже попавшие в прод файлы **не трогаем** (checksum). Любая правка — новая версия. Это защитит вас от неожиданных «Validation failed» в проде.

**Java — кастомизация префиксов/суффиксов/локаций**

```java
package com.example.naming;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayNamingConfig {
  @Bean
  FlywayConfigurationCustomizer naming() {
    return (FluentConfiguration c) -> c
      .locations("classpath:db/migration/core","classpath:db/migration/billing")
      .sqlMigrationPrefix("V")
      .repeatableSqlMigrationPrefix("R")
      .sqlMigrationSuffixes(".sql",".pgsql");
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.naming

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayNamingConfig {
    @Bean
    fun naming() = FlywayConfigurationCustomizer { c: FluentConfiguration ->
        c.locations("classpath:db/migration/core","classpath:db/migration/billing")
         .sqlMigrationPrefix("V")
         .repeatableSqlMigrationPrefix("R")
         .sqlMigrationSuffixes(".sql", ".pgsql")
    }
}
```

---

## SQL vs Java-based миграции: причины выбрать Java (сложные DML, кросс-БД логика)

По умолчанию выбирайте **SQL**: он проще ревьюить, его видят DBA, он не зависит от API мигратора. Но бывают сценарии, где удобнее Java/Kotlin: массовый контролируемый DML с батчами, потоковая обработка больших таблиц, кросс-БД логика, вычисления/нормализация данных, доступ к внешним ресурсам (в разумных пределах — без сетевых зависимостей в проде).

Java-миграция — это класс `db.migration.Vx__Desc` (или любое имя, если регистрируете resolver вручную), расширяющий `BaseJavaMigration`. Вам дают JDBC `Connection`, дальше — знакомая работа с `PreparedStatement`, батчами и курсорами. Код тестируется как обычный Java-код, его можно покрыть unit-тестами и запускать на Testcontainers.

Минусы Java-миграций: сложнее диффить на PR (вместо 3 строк SQL — блок Java-кода), выше риск логики «с побочками», сложнее перенос между СУБД. Поэтому держите Java-миграции компактными и целевыми, а всё, что выражается в SQL, оставляйте в SQL.

Гибридный паттерн: основа — SQL, «ответственные» ETL — Java/Kotlin. Так кодовая база остаётся прозрачной, но вы не мучаетесь SQL-циклом на миллионах строк.

Ещё один плюс Java-миграций — **инварианты**: можно прямо в миграции проверить допущения (например, нет `NULL` в колонке перед тем, как сделать её `NOT NULL`) и упасть с понятным сообщением, если они нарушены. В SQL это громоздко.

И наконец, не злоупотребляйте Java-миграциями для «бизнес-логики». Миграции — это про состояние схемы и подготовку данных, а не про поведение приложения.

**Java — пример Java-мирования с батчами**

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

public class V5__normalize_emails extends BaseJavaMigration {
    @Override public void migrate(Context ctx) throws Exception {
        try (var ps = ctx.getConnection().prepareStatement(
             "update users set email = lower(email) where email <> lower(email)")) {
            int updated = ps.executeUpdate();
            System.out.println("Normalized emails: " + updated);
        }
    }
}
```

**Kotlin — тот же смысл**

```kotlin
package db.migration

import org.flywaydb.core.api.migration.BaseJavaMigration
import org.flywaydb.core.api.migration.Context

class V5__normalize_emails : BaseJavaMigration() {
    override fun migrate(context: Context) {
        context.connection.prepareStatement(
            "update users set email = lower(email) where email <> lower(email)"
        ).use { ps ->
            val updated = ps.executeUpdate()
            println("Normalized emails: $updated")
        }
    }
}
```

---

## `baseline`, `outOfOrder`, `validateOnMigrate`, `cleanDisabled`: безопасные флаги

`baseline` — способ «подключиться» к уже существующей схеме: вы говорите Flyway считать текущую БД версией, например, `1`, и дальше катите `V2+`. Это нужно при миграции легаси под управление Flyway. Включайте флаг **осознанно**, фиксируйте его в документации релиза.

`outOfOrder` разрешает применять пропущенные версии «вне порядка» — когда ветки жили параллельно. Это спасает stage от «дырок», но несёт риск неожиданных взаимодействий между миграциями. Лучше использовать как временную подпорку и как можно быстрее «свести» историю.

`validateOnMigrate` — ваша страховка: при старте Flyway сравнит checksum’ы применённых миграций с файлами в classpath и упадёт, если кто-то «подправил прошлое». Включайте всегда — это один из главных защитных барьеров.

`cleanDisabled` запрещает опасную команду `clean` (снести схему) — на проде должно быть **true**. В dev можно оставить `clean` для удобства, но только под отдельным профилем.

Эти флаги — часть «боевого» профиля. На dev вы можете позволить себе более свободный режим (например, `outOfOrder=true` для агрессивной работы с ветками), но на prod — строгость и предсказуемость. Разведите это профилями Spring.

Проверяйте логи при старте: Flyway пишет явные сообщения о baseline, outOfOrder и валидации. Любая «жёлтая» строка — повод посмотреть внимательнее.

**application-prod.yml — безопасные настройки**

```yaml
spring:
  flyway:
    enabled: true
    validate-on-migrate: true
    clean-disabled: true
    out-of-order: false
```

**Java — программно задать baseline/outOfOrder (на время миграции легаси)**

```java
package com.example.flags;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayFlagsConfig {
  @Bean
  FlywayConfigurationCustomizer flags() {
    return (FluentConfiguration c) -> c
        .baselineOnMigrate(true)
        .baselineVersion("1")
        .outOfOrder(false)
        .validateOnMigrate(true)
        .cleanDisabled(true);
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.flags

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FlywayFlagsConfig {
    @Bean
    fun flags() = FlywayConfigurationCustomizer { c: FluentConfiguration ->
        c.baselineOnMigrate(true)
         .baselineVersion("1")
         .outOfOrder(false)
         .validateOnMigrate(true)
         .cleanDisabled(true)
    }
}
```

---

## Callbacks (`beforeMigrate/afterMigrate`) для сервисных операций и метрик

В простейшем варианте коллбеки — это **SQL-файлы** в `db/callback`: `beforeMigrate.sql`, `afterMigrate.sql`, `beforeEachMigrate.sql`, и т.д. (будут выполнены автоматически при соответствующих событиях). Это удобно для мелких сервисных действий — например, выставить `lock_timeout` или записать «след миграции» в служебную таблицу аудита.

Для интеграции с наблюдаемостью удобно повесить **метрику времени миграций**. В приложениях на Spring Boot без Micrometer можно хотя бы вывести в лог длительность и число выполненных шагов, но лучше — записать в счётчик/таймер. Если не хотите углубляться в API коллбеков Flyway, можно замерить время вокруг `migrate()` через `FlywayMigrationStrategy`.

Также коллбеки помогают выполнить одноразовые подготовительные действия «рядом» с миграциями: например, отключить триггеры на время тяжёлого DML и включить обратно в `afterMigrate`. В SQL это делается простыми `alter table ... disable/enable trigger`.

Не злоупотребляйте коллбеками для логики с побочными эффектами (HTTP-запросы, внешние сервисы). Миграции должны быть **детерминированны** и воспроизводимы в любом окружении. Всё, что может «зависнуть», оставляйте в отдельных job’ах.

Если хочется программного контроля (например, публиковать метрику в Micrometer), используйте `FlywayMigrationStrategy` — он вызывается Spring Boot’ом и позволяет обернуть вызов `migrate()`. Это проще, чем реализовывать Java Callback API Flyway, и портативно между версиями.

И ещё: на проде полезно логировать **события миграций** с указанием версии/описания — это облегчает расследования.

**SQL-коллбеки:**

```sql
-- src/main/resources/db/callback/beforeMigrate.sql
set local lock_timeout = '5s';
-- можно включить необходимый search_path, метки и пр.

-- src/main/resources/db/callback/afterMigrate.sql
insert into migration_audit(run_at, applied) values (now(), (select count(*) from flyway_schema_history));
```

**Java — измерим время миграций (стратегия)**

```java
package com.example.callbacks;

import org.flywaydb.core.Flyway;
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationStrategy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MigrateWithTiming {
  @Bean
  FlywayMigrationStrategy timed() {
    return (Flyway fw) -> {
      long t0 = System.currentTimeMillis();
      var res = fw.migrate();
      long dt = System.currentTimeMillis() - t0;
      System.out.printf("Flyway applied %d migrations in %d ms%n", res.migrationsExecuted, dt);
    };
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.callbacks

import org.flywaydb.core.Flyway
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationStrategy
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class MigrateWithTiming {
    @Bean
    fun timed() = FlywayMigrationStrategy { fw: Flyway ->
        val t0 = System.currentTimeMillis()
        val res = fw.migrate()
        val dt = System.currentTimeMillis() - t0
        println("Flyway applied ${res.migrationsExecuted} migrations in $dt ms")
    }
}
```

---

## Работа с несколькими локациями и модулями (multi-module Gradle/Maven)

В монорепозитории или модульном проекте удобно разделять миграции по модулям/домейнам: `core`, `billing`, `reporting`. Flyway поддерживает **несколько локаций**: вы указываете список путей, и он соберёт все файлы, применит `V__` по версиям, затем `R__` по имени. Это снижает конфликтность PR’ов и упрощает владение схемой командой.

Важно выбрать стратегию версий: либо единая «глобальная» шкала версий для всех модулей (проще для порядка выполнения), либо «окна» версий для каждого модуля (нужна дисциплина, но меньше конфликтов). Я сторонник единой шкалы — меньше сюрпризов в зависимости модулей.

В Gradle multi-module полезно положить миграции каждого модуля **в его ресурсах**, а в «исполняемом» модуле-сервисе собрать все локации через `FlywayConfigurationCustomizer`. Так вы получите один запуск миграций, но из множества каталогов.

Если у вас общая «платформенная» схема (расширения, shared-таблицы), вынесите её в отдельную локацию `classpath:platform/migration`. Это делает границу ответственности ясной.

Тестируйте сборку локаций на «чистой» БД через Testcontainers: иногда легко забыть добавить новую локацию и «получить» падение только на CI.

**application.yml — несколько локаций**

```yaml
spring:
  flyway:
    enabled: true
    locations:
      - classpath:db/migration/core
      - classpath:db/migration/billing
      - classpath:platform/migration
```

**Java — программно собрать локации (из модулей)**

```java
package com.example.multiloc;

import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MultiLocationsConfig {
  @Bean
  FlywayConfigurationCustomizer multi() {
    return (FluentConfiguration c) -> c.locations(
        "classpath:db/migration/core",
        "classpath:db/migration/billing",
        "classpath:platform/migration"
    );
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.multiloc

import org.flywaydb.core.api.configuration.FluentConfiguration
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class MultiLocationsConfig {
    @Bean
    fun multi() = FlywayConfigurationCustomizer { c: FluentConfiguration ->
        c.locations(
            "classpath:db/migration/core",
            "classpath:db/migration/billing",
            "classpath:platform/migration"
        )
    }
}
```

---

## Повторяемые миграции: стратегия изменений внутри `R__` и контроль хешей

Repeatable-скрипты `R__` Flyway держит в истории без версии, но с **checksum**; при изменении содержимого файл переигрывается. Поэтому стратегия разработки `R__` — «всегда идемпотентно» и «ясно по имени». Для VIEW: `create or replace view ...` (без `drop view`), для функции — `create or replace function ...`. Тогда повторный прогон безопасен.

Расщепляйте `R__` — один файл на один объект. Это уменьшает неожиданные «каскадные» переисполнения и облегчает чтение диффа в PR. Если в одном файле несколько объектов — любое изменение в одном вызовет пересоздание всех, что не всегда желательно.

Следите за зависимостями: если `R__` view зависит от `R__` функции, **порядок имён** должен гарантировать, что функция пойдёт раньше (например, `R__func_x.sql`, затем `R__view_y.sql`). Flyway применяет repeatable **в лексикографическом порядке имени файла**.

Хорошая практика — в CI выводить список повторно применённых `R__` и их checksum. Это помогает заметить «флапающие» изменения (например, если IDE добавляет/убирает лишние пробелы или окончания строк). Держите форматирование файлов стабильным.

Не кладите в `R__` тяжёлый DML: его переисполнение может занять минуты. Любой дорогой DML — только `V__`, чтобы вы контролировали момент. Для материализованных представлений — два шага: `V__` изменить структуру, `R__` — обновить текст view (или использовать REFRESH в `V__` при необходимости).

И последнее: если приходится «чинить» checksum (например, кто-то поправил прошлый `R__` без ревью), используйте `flyway repair` **только по регламенту**, фиксируя причину в журнале изменений.

**SQL — пример `R__` для VIEW и функции**

```sql
-- R__func_price_with_tax.sql
create or replace function price_with_tax(amount numeric) returns numeric
language sql immutable as $$
  select amount * 1.2
$$;

-- R__view_invoice_totals.sql
create or replace view v_invoice_totals as
select i.id, price_with_tax(i.total_cents/100.0) as total_with_tax
from invoices i;
```

**Java — распечатать checksum повторяемых миграций**

```java
package com.example.repeatables;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;

public class RepeatableChecksums {
    public static void main(String[] args) {
        var fw = Flyway.configure()
                .dataSource("jdbc:postgresql://localhost:5432/app","app","app")
                .locations("classpath:db/migration")
                .load();
        for (MigrationInfo mi : fw.info().all()) {
            if (mi.getVersion() == null) {
                System.out.printf("R__ %s | checksum=%s | state=%s%n",
                        mi.getDescription(), mi.getChecksum(), mi.getState());
            }
        }
    }
}
```

**Kotlin — то же**

```kotlin
package com.example.repeatables

import org.flywaydb.core.Flyway

fun main() {
    val fw = Flyway.configure()
        .dataSource("jdbc:postgresql://localhost:5432/app","app","app")
        .locations("classpath:db/migration")
        .load()
    fw.info().all().filter { it.version == null }.forEach {
        println("R__ ${it.description} | checksum=${it.checksum} | state=${it.state}")
    }
}
```

# 5. Liquibase — базовая практика

## Структура: master changelog, include/includeAll, разбиение по доменам

В Liquibase **вся история миграций описывается не «россыпью» файлов по версии**, а деревом changelog’ов. В корне обычно лежит **master-changelog** (например, `db/changelog/db.changelog.yaml` или `.xml`), который через `include`/`includeAll` подключает подкаталоги с доменными changeset’ами. Такая структура помогает разрезать ответственность: модули и команды работают в своих папках, а общий порядок загрузки задаётся центрально.

Разбиение по доменам снижает количество конфликтов в PR и повышает читаемость. Типичный приём — выделить каталоги `core`, `billing`, `identity`, `reporting`. Каждый каталог содержит «плоский» набор файлов (YAML/XML), где changeset’ы упорядочены по смыслу; мастер-файл делает `includeAll` по каталогам, и Liquibase сам отсортирует по имени файла и затем по порядку changeset’ов в файле.

С точки зрения воспроизводимости важно, что **историческая таблица `databasechangelog` остаётся единой**. Неважно, из какого файла подключился changeset, — запись в истории одна, с `id`, `author` и `filename`. Это делает диагностику прямолинейной: мы всегда видим «кто и что» применил.

`include` и `includeAll` отличаются: первый подключает **конкретный** файл, второй — **весь каталог** (с необязательной рекурсией). В больших репозиториях удобнее `includeAll`: добавил новый файл в папку — он автоматически попадёт в историю при следующем запуске. Если необходим строгий контроль порядка, используйте `include` с явным перечислением.

Подразделение «DDL vs DML» лучше отражать отдельными файлами и иногда — отдельными папками. Например, `db/changelog/billing/ddl` и `db/changelog/billing/dml`. Так проще жить с политикой «на прод DML идёт отдельно, по окнам», подключая файлы по контекстам.

Структуру стоит зафиксировать в README: где master, как именуем файлы, куда класть сиды, как действуют контексты и метки. Новому разработчику это экономит часы на «угадывание» конвенций. И помните: **один master на сервис** — не плодите мастеров, чтобы избежать гонок и дублирования.

В Spring Boot мастер обычно указывают в `spring.liquibase.change-log`. Если вы храните YAML и XML одновременно (например, часть команд предпочитает XML), master может быть в любом формате, а внутри свободно `include`-ить файлы другого формата — Liquibase это поддерживает.

Отдельный нюанс — **файловые пути**. В продакшн-пайплайнах используйте `classpath:`-ресурсы, а не `filesystem:`, чтобы артефакт был самодостаточным (JAR/образ содержит всё). Локально можно играться с `filesystem:` для быстрого «прогона» черновиков, но не смешивайте в одном master’е.

Для мультисервисных систем иногда выделяют «платформенный» мастер (общие объекты) и «сервисные» мастера (локальная схема). В таком случае порядок запуска должен быть чётко определён в CI: платформа → сервисы. Не пытайтесь подключать платформенный master из сервисного — лучше вызывать два шага в пайплайне.

Наконец, следите за **чистотой master’а**. Не пихайте в него changeset’ы напрямую — пусть он только включает другие файлы. Это уменьшает риск конфликтов и упрощает навигацию: «всё содержательное — в доменных файлах».

**Пример структуры каталогов**

```
src/main/resources/
└── db/
    └── changelog/
        ├── db.changelog.yaml         # master
        ├── core/
        │   ├── 0001-core-init.yaml
        │   └── 0002-core-users.yaml
        ├── billing/
        │   ├── 0001-billing-init.yaml
        │   └── 0002-billing-invoices.yaml
        └── seed/
            └── 0001-seed-refdata.yaml
```

**Master (YAML)**

```yaml
# db/changelog/db.changelog.yaml
databaseChangeLog:
  - includeAll: { path: db/changelog/core }
  - includeAll: { path: db/changelog/billing }
  - includeAll: { path: db/changelog/seed }
```

**Master (XML)**

```xml
<!-- db/changelog/db.changelog.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <includeAll path="db/changelog/core"/>
  <includeAll path="db/changelog/billing"/>
  <includeAll path="db/changelog/seed"/>
</databaseChangeLog>
```

**Gradle (Groovy DSL) — плагин и указание master**

```groovy
plugins { id 'org.liquibase.gradle' version '2.2.0' }
liquibase {
  activities {
    main {
      changeLogFile 'src/main/resources/db/changelog/db.changelog.yaml'
      url System.getenv('DB_URL')
      username System.getenv('DB_USER')
      password System.getenv('DB_PASS')
    }
  }
  runList = 'main'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins { id("org.liquibase.gradle") version "2.2.0" }
liquibase {
  activities.register("main") {
    this.arguments = mapOf(
      "changeLogFile" to "src/main/resources/db/changelog/db.changelog.yaml",
      "url" to System.getenv("DB_URL"),
      "username" to System.getenv("DB_USER"),
      "password" to System.getenv("DB_PASS")
    )
  }
  runList = "main"
}
```

**Java — программно задать master (через SpringLiquibase)**

```java
package com.example.liquibase.conf;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class LiquibaseMasterConfig {
  @Bean
  SpringLiquibase liquibase(DataSource ds) {
    SpringLiquibase lb = new SpringLiquibase();
    lb.setDataSource(ds);
    lb.setChangeLog("classpath:db/changelog/db.changelog.yaml");
    return lb;
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.conf

import liquibase.integration.spring.SpringLiquibase
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class LiquibaseMasterConfig {
    @Bean
    fun liquibase(ds: DataSource) = SpringLiquibase().apply {
        dataSource = ds
        changeLog = "classpath:db/changelog/db.changelog.yaml"
    }
}
```

---

## Changeset: `id/author`, `preconditions`, `runOnChange/runAlways`, `fail/markRan`

Каждый changeset — **атомарный шаг**. Его обязательно идентифицируют `id` и `author`. Пара должна быть уникальной во всей БД: Liquibase пишет её в `databasechangelog`. Если два файла дадут одинаковую пару, возникнет конфликт. Поэтому выбирайте понятные id (с префиксом домена/сюррогатной датой) и стабильный author (ник/команда).

`preconditions` — механизм безопасного запуска. Он проверяет состояние БД **до** изменений: «существует ли индекс», «есть ли колонка», «какая версия СУБД». Это позволяет писать idempotent-операции и избегать падений на «грязных» стендах. Поведение при провале задаётся атрибутом `onFail`: `HALT` (упасть), `MARK_RAN` (пометить применённым), `CONTINUE` (пропустить и идти дальше).

`runOnChange` и `runAlways` управляют переисполнением. Первый — повторно выполнит changeset при изменении содержимого (аналог Flyway `R__`). Второй — будет выполнять **каждый запуск**. На проде `runAlways` используйте крайне осторожно — это источник неожиданных долгих операций.

Для тонкой реакции есть `onError` (что делать при ошибке внутри changeset) и `failOnError` у отдельных тегов. В продакшн-практике чаще всего достаточно `onFail="HALT"` для структурных ddl и `MARK_RAN` для «добрых» create-if-not-exists.

Писать «условные» changeset’ы стоит сдержанно. Хороший паттерн — **precondition + чистое действие**. Плохой — «внутри sql-фрагмента писать IF EXISTS/NOT EXISTS» — так сложнее тестировать и переносить.

Ещё важная деталь — **замысел в имени**. `id: core-0002-add-users-email-unique` лучше зряче, чем `id: 12`. По id легче ориентироваться в истории и grep’ить.

Если вы изменили содержимое changeset’а **после** его применения, MD5 в `databasechangelog` перестанет совпадать. Для статичных шагов это повод **не править прошлое**, а добавлять новый changeset. Для «логических» объектов поставьте `runOnChange: true`.

Не забывайте про транзакции. Liquibase старается исполнять changeset транзакционно, но DDL в конкретной СУБД может вести себя по-разному. В Postgres большинство DDL транзакционны; в MySQL — нет. Учтите это при написании массовых DML и при проектировании откатов.

Наконец, `markRan` полезен для «уже сделано руками» редких случаев. Но фиксируйте это письменно и конвертируйте ручное действие в нормальный changeset при первой возможности.

**YAML — changeset с preconditions и runOnChange**

```yaml
databaseChangeLog:
  - changeSet:
      id: core-0002-add-idx-users-email
      author: team
      preConditions:
        - onFail: MARK_RAN
        - not:
            indexExists:
              tableName: users
              indexName: idx_users_email
      changes:
        - createIndex:
            tableName: users
            indexName: idx_users_email
            columns:
              - column: { name: email }
  - changeSet:
      id: core-0003-view-active-users
      author: team
      runOnChange: true
      changes:
        - createView:
            viewName: v_active_users
            replaceIfExists: true
            selectQuery: "select id, email from users where deleted_at is null"
```

**XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="core-0002-add-idx-users-email" author="team">
    <preConditions onFail="MARK_RAN">
      <not><indexExists tableName="users" indexName="idx_users_email"/></not>
    </preConditions>
    <createIndex tableName="users" indexName="idx_users_email">
      <column name="email"/>
    </createIndex>
  </changeSet>

  <changeSet id="core-0003-view-active-users" author="team" runOnChange="true">
    <createView viewName="v_active_users" replaceIfExists="true">
      <selectQuery>select id, email from users where deleted_at is null</selectQuery>
    </createView>
  </changeSet>
</databaseChangeLog>
```

**Java — валидация changelog и аккуратная обработка ошибок**

```java
package com.example.liquibase.validate;

import liquibase.Liquibase;
import liquibase.Scope;
import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.exception.ValidationFailedException;
import liquibase.resource.ClassLoaderResourceAccessor;

import javax.sql.DataSource;

public class LiquibaseValidator {
  private final DataSource ds;
  public LiquibaseValidator(DataSource ds) { this.ds = ds; }

  public void validateOrFail() throws Exception {
    try (var c = ds.getConnection()) {
      var db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(new JdbcConnection(c));
      var lb = new Liquibase("db/changelog/db.changelog.yaml", new ClassLoaderResourceAccessor(), db);
      lb.validate(); // бросит ValidationFailedException при проблемах
    } catch (ValidationFailedException e) {
      Scope.getCurrentScope().getUI().sendMessage("Liquibase validation failed: " + e.getMessage());
      throw e;
    }
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.validate

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.exception.ValidationFailedException
import liquibase.resource.ClassLoaderResourceAccessor
import javax.sql.DataSource

class LiquibaseValidator(private val ds: DataSource) {
    fun validateOrFail() {
        ds.connection.use { c ->
            val db = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(JdbcConnection(c))
            val lb = Liquibase("db/changelog/db.changelog.yaml", ClassLoaderResourceAccessor(), db)
            try {
                lb.validate()
            } catch (e: ValidationFailedException) {
                println("Liquibase validation failed: ${e.message}")
                throw e
            }
        }
    }
}
```

**Gradle (Groovy/Kotlin) — задача `liquibaseValidate`**

```groovy
tasks.register("liquibaseValidate") {
  dependsOn "update" // либо запускать отдельно
  doLast {
    println "Validated changelog."
  }
}
```

```kotlin
tasks.register("liquibaseValidate") {
  dependsOn("update")
  doLast { println("Validated changelog.") }
}
```

---

## Контексты и метки (`contexts`, `labels`) для управления средами/фичами

Контексты (`context`/`contexts`) — «переключатели по окружениям». Вы помечаете changeset: `context: prod` или `contexts: dev,stage`, и Liquibase применит его **только** если при запуске указан подходящий контекст. Это удобно для сидов на стендах, экспериментальных объектов, временных диагностических таблиц.

Метки (`labels`) — «переключатели по фичам». Например, вы готовите changeset для фичи `feature-x`, но включать его нужно только когда фича активируется. Вы ставите `labels: feature-x`, а при выпуске запускаете Liquibase с `--labels=feature-x`. Это даёт точечное включение без ветвления changelog’а.

Оба механизма можно комбинировать: `contexts: prod` + `labels: feature-x` ограничит changeset «фичей в проде». Если при запуске ни контекст, ни метка не подходят, шаг будет пропущен без ошибки, что делает сценарий безопасным.

В Spring Boot контексты указывают свойством `spring.liquibase.contexts`, метки — `spring.liquibase.labels`. У Gradle плагина есть соответствующие параметры. Это позволяет держать разные `application-*.yml` и не менять код.

Контексты удобны и для разделения DDL/DML: сиды `context: seed` можно прогонять только на dev/stage, а прод — держать чистым. Для больших backfill-ов полезно иметь `context: backfill` и включать его отдельным Job’ом.

Именование контекстов — часть команды. Не плодите «произвольный зоопарк». Достаточно `dev/test/stage/prod` и нескольких технических (`seed`, `backfill`, `metrics`), плюс метки для фич. Чем понятнее схема, тем меньше случайных запусков.

Если вы используете Helm/Jobs, контекст/метку проще передать в переменной окружения и пробросить её в Spring Boot/Gradle. Так пайплайн чётко фиксирует, что именно применялось на этапе релиза.

Не перегружайте changeset’ы контекстами. Если шаг «всегда для всех», не помечайте его — это шум. Контексты — для «вариативных» вещей.

И, наконец, тестируйте: поднимать контейнер с БД и запускать Liquibase с разными контекстами — недолго, а вот «не тот сид в проде» лечить неприятно.

**YAML — примеры контекста и метки**

```yaml
databaseChangeLog:
  - changeSet:
      id: seed-roles
      author: team
      context: dev,stage
      changes:
        - insert:
            tableName: roles
            columns:
              - column: { name: code, value: "ADMIN" }
              - column: { name: name, value: "Administrator" }

  - changeSet:
      id: feature-x-users-extra
      author: team
      labels: feature-x
      changes:
        - addColumn:
            tableName: users
            columns:
              - column: { name: x_data, type: JSONB }
```

**XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="seed-roles" author="team" context="dev,stage">
    <insert tableName="roles">
      <column name="code" value="ADMIN"/>
      <column name="name" value="Administrator"/>
    </insert>
  </changeSet>
  <changeSet id="feature-x-users-extra" author="team" labels="feature-x">
    <addColumn tableName="users">
      <column name="x_data" type="JSONB"/>
    </addColumn>
  </changeSet>
</databaseChangeLog>
```

**application.yml — задать контексты/метки**

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog.yaml
    contexts: dev,seed
    labels: feature-x
```

**Java — программно задать contexts/labels**

```java
package com.example.liquibase.ctx;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class LiquibaseContextConfig {
  @Bean
  SpringLiquibase lb(DataSource ds) {
    SpringLiquibase lb = new SpringLiquibase();
    lb.setDataSource(ds);
    lb.setChangeLog("classpath:db/changelog/db.changelog.yaml");
    lb.setContexts("dev,seed");
    lb.setLabels("feature-x");
    return lb;
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.ctx

import liquibase.integration.spring.SpringLiquibase
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class LiquibaseContextConfig {
    @Bean
    fun lb(ds: DataSource) = SpringLiquibase().apply {
        dataSource = ds
        changeLog = "classpath:db/changelog/db.changelog.yaml"
        contexts = "dev,seed"
        labels = "feature-x"
    }
}
```

**Gradle (Groovy/Kotlin) — передать контексты**

```groovy
liquibase {
  activities {
    main {
      contexts "stage,seed"
      labels "feature-x"
    }
  }
}
```

```kotlin
liquibase {
  activities.register("main") {
    this.arguments = mapOf(
      "contexts" to "stage,seed",
      "labels" to "feature-x"
    )
  }
}
```

---

## Форматы: XML/YAML/JSON/SQL — критерии выбора и смешанный подход

Liquibase одинаково поддерживает **XML, YAML, JSON и SQL**. На практике XML/YAML — «основной рабочий язык». XML типизируется XSD и дает самую полную подсветку/валидацию, YAML — компактнее и проще в чтении. JSON используют редко (менее человекочитаем), SQL — когда команда хочет оставаться в чистом SQL, но с историей Liquibase.

Выбор формата — вопрос культуры и tooling’а. Если у вас IDE/линтеры хорошо поддерживают XML-схемы, XML будет удобен и «строже». Если команда предпочитает «простой текст» и меньше скобок, YAML даёт отличный баланс. Разрешите оба: master может включать и YAML, и XML — Liquibase это умеет.

«SQL-формат» полезен для команд, которые не хотят изучать словарь тегов. Вы пишете обычный SQL, а Liquibase добавляет `--changeset id:author` и фиксирует историю. Минусы — ограниченные `preconditions`, сложнее rollback и меньше «переносимости» между СУБД на уровне тегов.

Смешанный подход часто идеален. Структурные изменения — в YAML/XML, процедурные вещи (`create or replace function`, специфичные index hints) — через `<sql>` или `<sqlFile>`. Так вы сохраняете выразительность и переносимость, но не бьётесь о границы DSL.

Важно уметь читать и то, и другое. Пример «одного и того же» изменения на XML и YAML ниже. Обратите внимание: атрибуты `constraints`, `defaultValueComputed`, `schemaName` одинаково выразимы.

Если вы «едете» на Postgres и активно пользуетесь спецификой (GIN/JSONB/CONCURRENTLY), декларативные теги не всегда покроют нюансы. Не стесняйтесь вставлять сырой SQL в changeset — это нормальная практика, особенно вместе с `preconditions`.

С точки зрения diff/generate инструменты одинаково работают с любым форматом: внутренняя модель та же. Нет смысла «переписывать» старые XML на YAML — живите с обоими, если так исторически сложилось.

Наконец, помните про код-стайл: в YAML не злоупотребляйте инлайновыми объектами, а в XML — переносите атрибуты на отдельные строки для читаемости. Единый стиль упростит ревью и диффы.

**YAML — создание таблицы и индекса**

```yaml
databaseChangeLog:
  - changeSet:
      id: core-001-init
      author: team
      changes:
        - createTable:
            tableName: users
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: email, type: TEXT, constraints: { nullable: false, unique: true } }
              - column: { name: created_at, type: TIMESTAMPTZ, defaultValueComputed: now(), constraints: { nullable: false } }
        - createIndex:
            tableName: users
            indexName: idx_users_created_at
            columns:
              - column: { name: created_at }
```

**XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="core-001-init" author="team">
    <createTable tableName="users">
      <column name="id" type="BIGSERIAL"><constraints primaryKey="true"/></column>
      <column name="email" type="TEXT"><constraints nullable="false" unique="true"/></column>
      <column name="created_at" type="TIMESTAMPTZ" defaultValueComputed="now()">
        <constraints nullable="false"/>
      </column>
    </createTable>
    <createIndex tableName="users" indexName="idx_users_created_at">
      <column name="created_at"/>
    </createIndex>
  </changeSet>
</databaseChangeLog>
```

**SQL-формат Liquibase (минимальный)**

```sql
--liquibase formatted sql
--changeset team:core-001-init
create table if not exists users(
  id bigserial primary key,
  email text not null unique,
  created_at timestamptz not null default now()
);
--changeset team:core-002-idx
create index if not exists idx_users_created_at on users(created_at);
```

**Java — переключение changeLog между YAML и XML (через property)**

```java
package com.example.liquibase.format;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class LiquibaseFormatConfig {
  @Bean
  SpringLiquibase liquibase(DataSource ds,
                            @Value("${app.liquibase.master:db/changelog/db.changelog.yaml}") String master) {
    SpringLiquibase lb = new SpringLiquibase();
    lb.setDataSource(ds);
    lb.setChangeLog("classpath:" + master);
    return lb;
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.format

import liquibase.integration.spring.SpringLiquibase
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class LiquibaseFormatConfig {
    @Bean
    fun liquibase(
        ds: DataSource,
        @Value("\${app.liquibase.master:db/changelog/db.changelog.yaml}") master: String
    ) = SpringLiquibase().apply {
        dataSource = ds
        changeLog = "classpath:$master"
    }
}
```

---

## Rollback-блоки и теги (`tag`, `rollbackToTag`) как основа безопасного отката

Сильная сторона Liquibase — **встроенные откаты**. В каждом changeset можно задать блок `rollback`, описав обратную операцию. Это не всегда «идеальный» возврат (данные могли измениться), но это контролируемая, **детерминированная** процедура. Для группового отката используют **теги** (`tagDatabase`), чтобы откатывать «до релиза».

Правильный подход — думать об откате **на этапе проектирования changeset’а**. Например, если вы добавили колонку со значениями, rollback должен удалить её, *если* это безопасно. Если нет — сделайте rollback «логическим»: отключите фичу (флаг), верните код, а в БД оставьте следы до следующего релиза (это нормально).

Перед «критическими» релизами практично ставить тег (`tagDatabase tag="release-2024.10.1"`). Тогда при необходимости вы сможете выполнить `rollbackToTag release-2024.10.1`. Этот подход хорошо встраивается в CI: тег пишется автоматически после успешной стадии миграций.

Важно понимать поведение транзакций. Если changeset включает несколько шагов, rollback должен соответствовать им. Часто лучше делать **несколько мелких** changeset’ов с простым rollback’ом, чем один гигантский со сложной обратной логикой.

Удаление столбцов/таблиц — «необратимые» операции. В таких случаях rollback должен быть «защитным» (создать пустую таблицу) и всегда сопровождаться «кодовым» откатом. Liquibase здесь не заменит архитектурную практику expand-and-contract.

Откаты удобно тестировать на Testcontainers. Сценарий: поднять чистую БД, `update` до тега, применить «следующий» кусок, сделать `rollbackToTag`, проверить схему/данные. Это даёт уверенность, что написанный rollback работает.

Внимание к **сид-данным**: их откаты обычно не нужны или не желательны на проде. Лучше помечать изменения контекстами (dev/stage) и не пытаться откатывать «демо-данные».

Не забывайте логировать факт постановки тега и успешного отката. Эти записи важны для аудита релизов и разборов инцидентов.

И последнее: в командах, где откаты строже регламентируются, заведите правило «без `rollback` changeset не проходит ревью». Пусть это будет дисциплиной, даже если 90% откатов «nominal».

**YAML — changeset с rollback и тег**

```yaml
databaseChangeLog:
  - changeSet:
      id: billing-001-add-status
      author: team
      changes:
        - addColumn:
            tableName: invoices
            columns:
              - column: { name: status, type: TEXT, defaultValue: "NEW" }
      rollback:
        - dropColumn:
            tableName: invoices
            columnName: status

  - changeSet:
      id: tag-2024-10-01
      author: ci
      changes:
        - tagDatabase:
            tag: release-2024.10.01
```

**XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="billing-001-add-status" author="team">
    <addColumn tableName="invoices">
      <column name="status" type="TEXT" defaultValue="NEW"/>
    </addColumn>
    <rollback>
      <dropColumn tableName="invoices" columnName="status"/>
    </rollback>
  </changeSet>
  <changeSet id="tag-2024-10-01" author="ci">
    <tagDatabase tag="release-2024.10.01"/>
  </changeSet>
</databaseChangeLog>
```

**Java — выполнить rollback до тега**

```java
package com.example.liquibase.rollback;

import liquibase.Liquibase;
import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.resource.ClassLoaderResourceAccessor;

import javax.sql.DataSource;

public class RollbackRunner {
  private final DataSource ds;
  public RollbackRunner(DataSource ds) { this.ds = ds; }

  public void rollbackTo(String tag) throws Exception {
    try (var c = ds.getConnection()) {
      var db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(new JdbcConnection(c));
      var lb = new Liquibase("db/changelog/db.changelog.yaml", new ClassLoaderResourceAccessor(), db);
      lb.rollback(tag, null);
    }
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.rollback

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import javax.sql.DataSource

class RollbackRunner(private val ds: DataSource) {
    fun rollbackTo(tag: String) {
        ds.connection.use { c ->
            val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(c))
            Liquibase("db/changelog/db.changelog.yaml", ClassLoaderResourceAccessor(), db)
                .rollback(tag, null)
        }
    }
}
```

**Gradle (Groovy/Kotlin) — задачи `updateSQL` и rollback**

```groovy
tasks.register("lbUpdateSQL") {
  dependsOn "update"
  doLast { println "Generated update SQL for review." }
}
```

```kotlin
tasks.register("lbUpdateSQL") {
  dependsOn("update")
  doLast { println("Generated update SQL for review.") }
}
```

---

## Автогенерация: `generateChangeLog` и `diffChangeLog` — когда уместно, когда вредно

`generateChangeLog` создаёт changelog **из существующей схемы**. Это удобно при онбординге легаси: вы «снимаете слепок» текущего состояния, кладёте его как baseline и дальше двигаетесь changeset’ами. Минусы — «шумный» генерируемый файл, спорные имена констрейнтов и отсутствие «архитектурного смысла» (почему так, а не иначе) — поэтому генерацию применяют **один раз**.

`diffChangeLog` сравнивает две базы или базу и changelog. Это полезно как ночная проверка **дрейфа**: «не появилось ли на проде чего-то вручную». В CI можно падать, если diff не пустой. DBA тоже оценят: меньше сюрпризов.

Для **проектирования новых изменений** генерацию использовать не стоит. Ручной, аккуратный changeset — лучше: вы продумываете индексы, именование, preconditions и rollback. Сгенерированное — слишком часто содержит лишнее, а главное — **не читается**.

Ещё случай «уместно» — миграции между версиями СУБД или перенос схемы. `diffChangeLog` помогает увидеть, где «разъехалось». Всё равно итоговые changeset’ы стоит отредактировать руками.

Хранить сгенерированные `updateSQL` — неплохая идея для аудита: вы кладёте в CI-артефакты SQL релиза, даже если применяете через API Liquibase. DBA могут ревьюить и подписывать такой SQL.

Не путайте `updateSQL` (генерирует SQL **для данного changelog** без выполнения) и `generateChangeLog` (генерирует **сам changelog** из БД). Это разные этапы и разные цели.

С точки зрения безопасности доступов генерация требует коннекта к БД и прав чтения метаданных; в проде это, как правило, read-only. Для сравнения используют staging/readonly реплики.

Не забывайте, что diff у разных СУБД имеет свои нюансы (типизация, дефолты, имена). Старайтесь сравнивать «одного вида» базы и одинаковые настройки.

Итог: генерацию используем **для baseline и мониторинга**, не для ежедневной разработки changeset’ов.

**Gradle (Groovy DSL) — задачи diff/generate**

```groovy
plugins { id 'org.liquibase.gradle' version '2.2.0' }
liquibase {
  activities {
    diffMain {
      changeLogFile 'build/liquibase/diff.changelog.xml'
      url System.getenv('DB_URL')
      username System.getenv('DB_USER')
      password System.getenv('DB_PASS')
      referenceUrl System.getenv('REF_DB_URL')
      referenceUsername System.getenv('REF_DB_USER')
      referencePassword System.getenv('REF_DB_PASS')
    }
  }
  runList = 'diffMain'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins { id("org.liquibase.gradle") version "2.2.0" }
liquibase {
  activities.register("diffMain") {
    this.arguments = mapOf(
      "changeLogFile" to "build/liquibase/diff.changelog.xml",
      "url" to System.getenv("DB_URL"),
      "username" to System.getenv("DB_USER"),
      "password" to System.getenv("DB_PASS"),
      "referenceUrl" to System.getenv("REF_DB_URL"),
      "referenceUsername" to System.getenv("REF_DB_USER"),
      "referencePassword" to System.getenv("REF_DB_PASS")
    )
  }
  runList = "diffMain"
}
```

**Java — сгенерировать diff как текст**

```java
package com.example.liquibase.diff;

import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.diff.DiffGeneratorFactory;
import liquibase.diff.output.changelog.DiffToChangeLog;
import liquibase.resource.ClassLoaderResourceAccessor;

import javax.sql.DataSource;
import java.io.StringWriter;

public class DiffService {
  public static String diff(DataSource left, DataSource right) throws Exception {
    try (var c1 = left.getConnection(); var c2 = right.getConnection()) {
      var db1 = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(new JdbcConnection(c1));
      var db2 = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(new JdbcConnection(c2));
      var diff = DiffGeneratorFactory.getInstance().compare(db1, db2, null);
      var out = new StringWriter();
      new DiffToChangeLog(diff).print(out);
      return out.toString();
    }
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.diff

import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.diff.DiffGeneratorFactory
import liquibase.diff.output.changelog.DiffToChangeLog
import java.io.StringWriter
import javax.sql.DataSource

object DiffService {
    fun diff(left: DataSource, right: DataSource): String {
        left.connection.use { c1 ->
            right.connection.use { c2 ->
                val db1 = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(c1))
                val db2 = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(c2))
                val diff = DiffGeneratorFactory.getInstance().compare(db1, db2, null)
                val out = StringWriter()
                DiffToChangeLog(diff).print(out)
                return out.toString()
            }
        }
    }
}
```

---

## Custom change & extension points: расширения под специфические DDL/DML

Иногда стандартного DSL недостаточно: нужен особый DDL/DML, сложный backfill, интеграция с системой фич-флагов. Для этого в Liquibase есть **custom changes** — Java/Kotlin-классы, реализующие `CustomTaskChange` (или `AbstractChange` для более глубокой интеграции). Их можно вызвать из YAML/XML через тег `<customChange class="...">`.

Хороший кейс — «безопасный backfill» по батчам. Стандартным `update` это неудобно выражать, а кастомный класс может читать курсором и обновлять частями, логируя прогресс. При повторном запуске он должен быть **идемпотентным** (например, по маркеру «уже обработано»).

Ещё пример — системные «хинты» перед длинным DDL/DML: установить `lock_timeout`, ждать «окно», выключить триггеры. Это тоже можно сделать custom-change, но часто достаточно коллбеков или `sql`.

Custom-change живёт в JAR вашего приложения: это удобно — он распространяется вместе с кодом. Но помните: такой changeset «менее переносим» между языками и требует зависимостей на Liquibase API; закладывайте это в ваш build.

Важно тестировать custom-change отдельно (unit-тесты на тестовой БД), потому что ошибки там — это падение миграций в самый неподходящий момент. Оберните работу в транзакцию, где возможно, и аккуратно закрывайте ресурсы.

С точки зрения безопасности ограничьте возможности custom-change: никаких сетевых вызовов, никаких записей в сторонние системы — миграции должны быть **воспроизводимыми офлайн**. Если нужно что-то внешнее — лучше отдельный job до/после миграций.

В XML/YAML вы передаёте параметры custom-change через `<param name="…" value="…"/>` (XML) или `parameters:` (YAML). Это удобный способ конфигурировать поведение без перекомпиляции.

Если у вас повторяется один и тот же DDL-паттерн (например, «создать уникальный индекс по выражению, если его нет»), имеет смысл написать маленькое расширение и переиспользовать по проекту. Это повысит однообразие и снизит ошибки.

И наконец, документируйте custom-изменения: где они используются, какие имеют ограничения и как тестируются. Это часть «архитектуры миграций», не пренебрегайте этим.

**Java — пример CustomTaskChange (батчевый backfill)**

```java
package com.example.liquibase.custom;

import liquibase.change.custom.CustomTaskChange;
import liquibase.database.Database;
import liquibase.exception.CustomChangeException;
import liquibase.exception.SetupException;
import liquibase.resource.ResourceAccessor;

import java.sql.Connection;
import java.sql.PreparedStatement;

public class BackfillUserLowercase implements CustomTaskChange {

  private int batchSize = 1000;

  public void setBatchSize(int batchSize) { this.batchSize = batchSize; }

  @Override public void execute(Database database) throws CustomChangeException {
    try {
      Connection c = database.getConnection().getUnderlyingConnection();
      try (PreparedStatement ps = c.prepareStatement(
          "update users set email = lower(email) where email <> lower(email) limit ?")) {
        ps.setInt(1, batchSize);
        int updated = ps.executeUpdate();
        System.out.println("BackfillUserLowercase updated: " + updated);
      }
    } catch (Exception e) {
      throw new CustomChangeException("Backfill failed", e);
    }
  }

  @Override public String getConfirmationMessage() { return "BackfillUserLowercase done"; }
  @Override public void setUp() throws SetupException {}
  @Override public void setFileOpener(ResourceAccessor resourceAccessor) {}
  @Override public void validate(Database database) throws CustomChangeException {}
}
```

**Kotlin — то же**

```kotlin
package com.example.liquibase.custom

import liquibase.change.custom.CustomTaskChange
import liquibase.database.Database
import liquibase.exception.CustomChangeException
import liquibase.exception.SetupException
import liquibase.resource.ResourceAccessor
import java.sql.Connection

class BackfillUserLowercase : CustomTaskChange {

    var batchSize: Int = 1000

    override fun execute(database: Database) {
        try {
            val c: Connection = database.connection.underlyingConnection
            c.prepareStatement(
                "update users set email = lower(email) where email <> lower(email) limit ?"
            ).use { ps ->
                ps.setInt(1, batchSize)
                val updated = ps.executeUpdate()
                println("BackfillUserLowercase updated: $updated")
            }
        } catch (e: Exception) {
            throw CustomChangeException("Backfill failed", e)
        }
    }

    override fun getConfirmationMessage(): String = "BackfillUserLowercase done"
    override fun setUp() { /* no-op */ }
    override fun setFileOpener(resourceAccessor: ResourceAccessor?) { /* no-op */ }
    override fun validate(database: Database?) { /* no-op */ }
}
```

**YAML — вызов customChange с параметром**

```yaml
databaseChangeLog:
  - changeSet:
      id: util-001-backfill-lowercase
      author: team
      changes:
        - customChange:
            class: com.example.liquibase.custom.BackfillUserLowercase
            param:
              name: batchSize
              value: 500
```

**XML — эквивалент**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.23.xsd">
  <changeSet id="util-001-backfill-lowercase" author="team">
    <customChange class="com.example.liquibase.custom.BackfillUserLowercase">
      <param name="batchSize" value="500"/>
    </customChange>
  </changeSet>
</databaseChangeLog>
```

**Gradle (Groovy/Kotlin) — убедиться, что класс доступен в classpath Liquibase**

```groovy
dependencies {
  implementation 'org.liquibase:liquibase-core'
  // ваш модуль с классом customChange уже в implementation
}
```

```kotlin
dependencies {
  implementation("org.liquibase:liquibase-core")
}
```

# 6. Среды, права и стратегия запуска

## Профили Spring (`dev/test/stage/prod`) и разные наборы миграций/плейсхолдеров

В реальных проектах миграции никогда не «одинаковы для всех». На локальной машине удобно поднимать пустую БД, заливать сид-данные и падать с валидацией, если чего-то не хватает. На prod нам нужны ровно обратные свойства: никакой автосозданной схемы, никакой случайной «демо-инициализации», строгая валидация и управляемый запуск. Проще всего добиться этого через профили Spring и конфиг `application-*.yml`. Так вы не плодите разные артефакты, а меняете поведение одним флажком.

Профиль `dev` обычно включает on-start миграции и сиды. Пара полезных флагов для Flyway — `check-location=true` (упасть, если папка миграций не найдена) и `clean-on-validation-error=true` (только для **dev**, чтобы не вручную сбрасывать схему). Для Liquibase — `contexts: dev,seed`, чтобы запускать changeset’ы с соответствующими контекстами; на prod вы просто не указываете эти контексты и сиды не пойдут.

Плейсхолдеры — ещё один слой вариативности. Во Flyway это `spring.flyway.placeholders.*`, в Liquibase — `spring.liquibase.parameters.*` или `<property>` в master. Через них удобно прокидывать имена схем, суффиксы индексов, параметры таймаутов и любые «безопасные» значения, которые различаются по средам. Хранить их лучше в конфигурации, а не в самих SQL/changeset’ах.

Отдельный нюанс — «идея тестового профиля». Для `test` (юнит/интеграционные тесты) вы обычно хотите быстрое поднятие схемы и повторяемые сиды. В `@DataJpaTest` это достигается простым `spring.liquibase.enabled=true` или `spring.flyway.enabled=true`. Если вы используете Testcontainers, профили позволят вам одинаково запускать миграции локально и в CI.

Не смешивайте две идеи: «профили приложений» и «контексты Liquibase». Профиль — это настройка Spring, контекст — фильтр changeset’ов Liquibase. Сами по себе контексты в `application-*.yml` задавать удобно, но не заменяйте ими профили — у них разные цели. Аналогично, у Flyway нет контекстов, но есть **локации** (можно разнести сиды в отдельную папку и включать их только в dev-профиле).

Наконец, зафиксируйте таблицу «что включено в каком профиле»: это убережёт от злосчастного «в prod уехал dev-сид» или «на stage не применились служебные индексы». Короткий README в каталоге миграций с матрицей профилей окупает себя за один сэкономленный инцидент.

**application-dev.yml — Flyway и сиды (примеры плейсхолдеров)**

```yaml
spring:
  flyway:
    enabled: true
    locations:
      - classpath:db/migration
      - classpath:db/seed     # только в dev
    check-location: true
    placeholders:
      app_schema: app
      lock_timeout: "5s"
```

**application-prod.yml — Liquibase без сидов**

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog.yaml
    contexts: prod
    parameters:
      appSchema: app
```

**Java — выбор инструмента и контекста по профилю**

```java
package com.example.mig;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.*;
import javax.sql.DataSource;

@Configuration
@ConditionalOnProperty(value = "migrations.tool", havingValue = "liquibase")
class LiquibaseProfileConfig {
  @Bean
  SpringLiquibase liquibase(DataSource ds) {
    SpringLiquibase lb = new SpringLiquibase();
    lb.setDataSource(ds);
    lb.setChangeLog("classpath:db/changelog/db.changelog.yaml");
    lb.setContexts("${spring.profiles.active:dev}");
    return lb;
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.mig

import liquibase.integration.spring.SpringLiquibase
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
@ConditionalOnProperty(value = ["migrations.tool"], havingValue = "liquibase")
class LiquibaseProfileConfig {
    @Bean
    fun liquibase(ds: DataSource) = SpringLiquibase().apply {
        dataSource = ds
        changeLog = "classpath:db/changelog/db.changelog.yaml"
        contexts = System.getProperty("spring.profiles.active", "dev")
    }
}
```

---

## Пользователь миграций отдельно от приложения (минимальные и повышенные привилегии)

Принцип наименьших привилегий говорит: приложению не нужны права `CREATE/DROP TABLE`, а мигратору — иногда нужны. Разделите учётки. На prod миграции запускаются пользователем `app_migrator` с ограниченным, но достаточным набором прав на DDL/DML для схемы приложения; само приложение работает под `app_user` с CRUD-правами только к нужным таблицам/представлениям.

В Spring Boot это легко задаётся: мигратор можно направить на **свой DataSource**, отличающийся от основного. Для Liquibase достаточно объявить `SpringLiquibase` с отдельным `DataSource`. Для Flyway можно использовать `spring.flyway.url/user/password` (они переопределяют основной `DataSource`), либо программно сконфигурировать `FluentConfiguration`.

Такой разнос значительно снижает blast radius. Ошибка в коде приложения не сможет удалить таблицу; ошибка в миграции не сможет случайно писать в чужие схемы. Плюс аудит: в логах БД чётко видно, какие операции шли от `app_migrator`.

Обратите внимание на права **на последовательности и индексы** в PostgreSQL. Даже если приложению не нужны DDL, ему нужны гранты на `USAGE`/`SELECT` по sequence, если вы используете `SERIAL`/`BIGSERIAL`/`nextval`. Это настраивается миграциями и проверяется в тестах.

В multi-schema сценариях убедитесь, что мигратор имеет права на все нужные схемы. И не забывайте про таблицы истории миграций (`flyway_schema_history`/`databasechangelog`) — на них тоже нужны права.

Храните креды отдельно: в Kubernetes — в `Secret`, в локалке — в `.env`/Dev-секретах. Не хардкодьте пароли в `application.yml` и тем более в миграциях. Подтягивайте их через переменные окружения.

**application.yml — разные учётки для приложения и мигратора (Flyway)**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/app
    username: app_user
    password: ${APP_PASS}
  flyway:
    enabled: true
    url: jdbc:postgresql://db:5432/app
    user: app_migrator
    password: ${MIG_PASS}
```

**Java — отдельный DataSource для Liquibase**

```java
package com.example.mig;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.*;
import javax.sql.DataSource;

@Configuration
class MigrationDataSourceConfig {
  @Bean
  @ConfigurationProperties("app.migration.datasource")
  DataSourceProperties migrationProps() { return new DataSourceProperties(); }

  @Bean
  SpringLiquibase liquibase(@Qualifier("migrationProps") DataSourceProperties p) {
    SpringLiquibase lb = new SpringLiquibase();
    DataSource ds = p.initializeDataSourceBuilder().build();
    lb.setDataSource(ds);
    lb.setChangeLog("classpath:db/changelog/db.changelog.yaml");
    return lb;
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.mig

import liquibase.integration.spring.SpringLiquibase
import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.*
import javax.sql.DataSource

@Configuration
class MigrationDataSourceConfig {
    @Bean
    @ConfigurationProperties("app.migration.datasource")
    fun migrationProps() = DataSourceProperties()

    @Bean
    fun liquibase(props: DataSourceProperties): SpringLiquibase =
        SpringLiquibase().apply {
            dataSource = props.initializeDataSourceBuilder().build()
            changeLog = "classpath:db/changelog/db.changelog.yaml"
        }
}
```

---

## «Запуск на старте» против выделенной стадии в CI/CD: плюсы/минусы

On-start (приложение само применяет миграции при запуске) — минимальный порог входа. Вы собираете один JAR/контейнер, разворачиваете — и схема всегда «догоняется». Это удобно для dev, для staging, для маленьких продуктов. Минусы начинаются, когда появляется контроль окна обслуживания, «сухие прогоны», метрики длительных DDL, координация между несколькими сервисами.

Отдельная стадия (CI/CD Job или K8s Job/Helm hook) даёт управляемость. Вы запускаете миграции **до** деплоя приложения, видите отчёт и время, можете применить только часть (через контексты/лейблы), остановиться на теге и при необходимости откатить. Приложение стартует уже на готовой схеме — меньше риск полупридушенного старта и долбёжки в недоступные таблицы.

Компромиссный подход — on-start в dev/test, Job — в stage/prod. Это сочетает скорость локальной разработки и надёжность продакшн-релизов. Главное — держать **один и тот же changelog/каталоги миграций**; разница — только в способе запуска.

Ещё плюс выделенной стадии — независимые креды/права: Job использует `app_migrator`, приложение — `app_user`. Это сложнее организовать при on-start, хотя и возможно с `spring.flyway.user`.

С on-start сложно реализовать «временные паузы» и объёмные backfill’ы. Если миграция идёт 30 минут, вы рискуете «держать» весь rollout. Job позволяет разделить DDL и DML на разные шаги, запускать DML ночью, и использовать dry-run (Liquibase `updateSQL`) для ревью DBA.

Наконец, обратите внимание на **идемпотентность** и повторы. Job в Kubernetes может быть перезапущен, поэтому миграции должны быть безопасны к повторному запуску. Хороший changelog это позволяет.

**Gradle (Groovy) — отдельная задача для миграций Flyway**

```groovy
plugins { id 'org.flywaydb.flyway' version '10.16.0' }
flyway {
  url = System.getenv('DB_URL')
  user = System.getenv('DB_USER_MIG')
  password = System.getenv('DB_PASS_MIG')
  locations = ['classpath:db/migration']
}
tasks.register('dbMigrate') {
  dependsOn 'flywayMigrate'
}
```

**Gradle (Kotlin) — Liquibase Job**

```kotlin
plugins { id("org.liquibase.gradle") version "2.2.0" }
liquibase {
  activities.register("prod") {
    arguments = mapOf(
      "changeLogFile" to "src/main/resources/db/changelog/db.changelog.yaml",
      "url" to System.getenv("DB_URL"),
      "username" to System.getenv("DB_USER_MIG"),
      "password" to System.getenv("DB_PASS_MIG"),
      "contexts" to "prod"
    )
  }
  runList = "prod"
}
```

**Kubernetes Job — запуск Liquibase перед деплоем**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: liquibase-update
spec:
  template:
    spec:
      containers:
        - name: lb
          image: eclipse-temurin:21
          command: ["java","-jar","app.jar"]
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: SPRING_LIQUIBASE_ENABLED
              value: "true"
      restartPolicy: OnFailure
```

---

## Блокировки мигратора: `flyway_schema_history` / `databasechangelog(lock)` и дисциплина запуска

Любой мигратор должен защищаться от параллельного запуска. Flyway делает это на уровне транзакции изменения `flyway_schema_history`: он атомарно вставляет «следующую» версию; одновременный запуск второго процесса получит конфликт. Liquibase использует таблицу `databasechangeloglock`; при старте берётся эксклюзивная блокировка (строка с `LOCKED=true`), и все остальные инстансы ждут.

Важный операционный момент — **«залипшие» блокировки**. Если процесс упал, у Liquibase может остаться `LOCKED=true`. Это лечится командой/скриптом «release lock» (безопасно, если вы уверены, что миграций нет в процессе). У Flyway после аварии в истории может остаться «битый» шаг — используется `flyway repair` (пересчёт checksum, чистка неудачных записей).

Запрограммируйте дисциплину: не запускайте миграции одновременно из нескольких пайплайнов. В Helm/ArgoCD/Flux используйте hooks с типом `pre-install`/`pre-upgrade` и **конкурентность=один**. То же в Jenkins/GitLab — ставьте «mutex» на environment.

Хорошая практика — отдельный эндпойнт/метрика «сейчас идёт миграция». Приложение может откладывать `readiness` пока миграции не закончились (если у вас on-start), или оркестратор не будет раскатывать приложение, пока Job миграций не завершился.

Если вы применяете большие DDL, используйте `lock_timeout`/`statement_timeout` на уровне сессии (см. следующий пункт). Иначе блокировки на системных таблицах могут «подвесить» всё.

И наконец, не правьте руками `databasechangelog`/`flyway_schema_history`, если точно не понимаете последствия. Любое вмешательство должно быть задокументировано и сопровождаться ближайшей корректирующей миграцией.

**SQL — посмотреть и отпустить lock Liquibase (с осторожностью)**

```sql
select * from databasechangeloglock;
-- если уверены, что миграций нет
update databasechangeloglock set locked=false, lockgranted=null, lockedby=null where id=1;
```

**Java — показать состояние Flyway/Liquibase в логах**

```java
package com.example.mig;

import org.flywaydb.core.Flyway;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.*;

@Configuration
class MigInfo {
  @Bean CommandLineRunner flywayInfo(Flyway fw) {
    return args -> {
      System.out.println("Pending: " + fw.info().pending().length);
      System.out.println("Applied: " + fw.info().applied().length);
    };
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.mig

import org.flywaydb.core.Flyway
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class MigInfo {
    @Bean
    fun flywayInfo(fw: Flyway) = CommandLineRunner {
        println("Pending: ${fw.info().pending().size}")
        println("Applied: ${fw.info().applied().size}")
    }
}
```

---

## Multi-tenant и multi-schema: миграции по циклу арендаторов/схем

Мультиарендность в PostgreSQL часто реализуют через отдельные схемы на арендатора: `tenant_123`, `tenant_456`. В такой модели миграции нужно прогонять **для каждой схемы**. Flyway поддерживает список схем (`schemas`), но все изменения будут применяться к **каждой** схеме в одном запуске, что не всегда то, что нужно (например, частичный rollout). Поэтому практичнее — программный цикл: для каждой схемы создать конфигурацию Flyway с `defaultSchema` и запустить `migrate()`.

С Liquibase подход похож: вы можете параметризовать `schemaName` через `parameters` и для каждого арендатора запускать `update` с разными параметрами. Другой вариант — генерировать per-tenant master (не очень удобно). Ключ — **идемпотентность** changeset’ов и явные `schemaName` в каждом изменении.

Управление списком арендаторов можно хранить в «платформенной» схеме (`public.tenants`) и читать его перед запуском миграций. На prod важно включать **паузы/пакеты**: не мигрируйте тысячи арендаторов одним махом; батчируйте по N и отслеживайте прогресс.

Если мультиарендность на уровне БД реализована через **отдельные базы**, то вы просто крутите список JDBC URL и выполняете миграции для каждого (K8s/CI-Job в цикле). Логи и статус — в артефакты пайплайна.

Будьте осторожны с `databasechangelog`/`flyway_schema_history`: у каждой схемы/БД должна быть **своя** история. Не используйте одну таблицу истории на все схемы, иначе вы смешаете состояния.

И наконец, подумайте о «двухэтапной» модели: сначала накатываете DDL на общую платформенную схему (если есть), затем циклом — DDL/DML по арендаторам. Это естественный порядок зависимостей.

**Java — обойти список схем и запустить Flyway**

```java
package com.example.tenant;

import org.flywaydb.core.Flyway;
import javax.sql.DataSource;
import java.util.List;

public class TenantMigrator {
  public static void migrateAll(DataSource ds, List<String> schemas) {
    for (String schema : schemas) {
      Flyway.configure()
            .dataSource(ds)
            .defaultSchema(schema)
            .locations("classpath:db/migration/tenant")
            .load()
            .migrate();
      System.out.println("Migrated schema: " + schema);
    }
  }
}
```

**Kotlin — Liquibase с параметром схемы**

```kotlin
package com.example.tenant

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import javax.sql.DataSource

fun migrateTenant(ds: DataSource, schema: String) {
    ds.connection.use { c ->
        val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(c))
        val lb = Liquibase("db/changelog/tenant.changelog.yaml", ClassLoaderResourceAccessor(), db)
        lb.update(mapOf("tenantSchema" to schema))
    }
}
```

---

## Инициализационные данные vs тестовые сиды: разделение и включение по контекстам

Сиды — коварная тема. В dev удобно иметь пользователей-демо, справочники, пару заказов. На prod это почти всегда **лишнее** и опасное. Поэтому отделяйте **инициализационные данные** (без которых сервис не живёт — например, записи в таблице ролей/прав) от **тестовых сидов** (демо-клиенты, заказы, отчёты). Первые входят в обычные миграции и применяются везде; вторые включайте по контекстам/локациям только в dev/stage.

Во Flyway практично вынести сиды в отдельную локацию, например `db/seed`, и включать её только в `application-dev.yml`. Там же можно хранить `R__`-представления, если они нужны для локальной аналитики. Для Liquibase используйте `context: seed` и `spring.liquibase.contexts: dev,seed`.

Данные лучше грузить **детерминированно**: конкретные значения ключей, стабильные поля, идемпотентные UPSERT’ы. В PostgreSQL это `insert ... on conflict do nothing/update`. Тогда сид-changeset можно прогонять много раз без размножения дублей.

Хорошая практика — проверять «наличие» данных через `preconditions` (Liquibase) или через `where not exists` в SQL (Flyway). Так вы избежите ошибок при повторном применении.

Отделяйте «сид-таблицы» и «боевые таблицы» схемой/префиксом только если это решает реальную проблему. Чаще достаточно контекстов. Избыточное усложнение создаёт дрейф схем.

И, конечно, не смешивайте сиды и DDL в одном changeset/файле. При необходимости откатывать/выключать сиды вы не будете задевать структуру.

**Flyway — сиды в отдельной локации (dev-профиль)**

```yaml
spring:
  flyway:
    locations:
      - classpath:db/migration
      - classpath:db/seed
```

```sql
-- db/seed/V1000__seed_roles.sql (idempotent)
insert into roles(code, name)
values ('ADMIN','Administrator')
on conflict (code) do update set name=excluded.name;
```

**Liquibase YAML — сиды только для dev/stage**

```yaml
databaseChangeLog:
  - changeSet:
      id: seed-0001-roles
      author: team
      context: dev,stage
      changes:
        - insert:
            tableName: roles
            columns:
              - column: { name: code, value: "ADMIN" }
              - column: { name: name, value: "Administrator" }
```

**Java — защитный сид по профилю**

```java
package com.example.seed;

import org.springframework.context.annotation.*;
import org.springframework.jdbc.core.JdbcTemplate;

@Configuration
@Profile({"dev","stage"})
class SeedConfig {
  SeedConfig(JdbcTemplate jdbc) {
    jdbc.update("""
       insert into roles(code,name)
       values ('USER','User')
       on conflict (code) do nothing
    """);
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.seed

import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile
import org.springframework.jdbc.core.JdbcTemplate

@Configuration
@Profile("dev","stage")
class SeedConfig(jdbc: JdbcTemplate) {
    init {
        jdbc.update(
            """
            insert into roles(code,name)
            values ('USER','User')
            on conflict (code) do nothing
            """.trimIndent()
        )
    }
}
```

---

## Управление временем простоя: окна обслуживания, флаги timeouts, dry-run (Liquibase `updateSQL`)

Даже «простые» DDL могут держать блокировки дольше ожидаемого. Продакшн-практика — явные **окна обслуживания** для опасных операций и выставление **таймаутов**. В PostgreSQL здорово работает `lock_timeout` и `statement_timeout`: если взять блокировку не удалось — команда падает быстро, а не висит полчаса. Эти параметры можно установить сессионно через SQL-коллбеки Flyway или `sql`-changeset Liquibase.

Dry-run — ещё один столп «безопасных» миграций. Liquibase умеет `updateSQL`: он выпишет SQL без выполнения — DBA ревьюят, подписывают, вы применяете. Для Flyway аналог прямой команды нет, но вы можете разворачивать миграции на «сухой» базе (Testcontainers) и собирать дамп SQL из логов драйвера или использовать `pg_dump --schema-only` для сверки.

Для больших DML — батчируйте. Разбейте апдейт на порции по первичному ключу/диапазонам времени, ставьте «паузу» между порциями, чтобы не забивать репликацию и не раздувать WAL. Такие миграции лучше запускать отдельной стадией/Job’ом и помечать `context: backfill`.

Онлайн-DDL для индексов — используйте специфичные опции СУБД. В Postgres — `CREATE INDEX CONCURRENTLY`, в MySQL — `ALGORITHM=INPLACE, LOCK=NONE` (в зависимости от версии). В Liquibase такие флаги часто передаются сырым SQL (`<sql>`), а во Flyway — просто в `V__` файле.

Не забывайте про **time budget** пайплайна. Если ваш Job миграций имеет лимит 10 минут, а DDL/ETL занимает 40 — вы будете получать флейки. Снимите метрики времени миграций (см. FlywayMigrationStrategy/логирование Liquibase) и планируйте окна с запасом.

И, наконец, документируйте «опасные» шаблоны. Короткий рецепт «как добавить индекс без простоя» или «как прогнать backfill на 200 млн строк» экономит часы каждому новому инженеру.

**Flyway — коллбек для таймаутов**

```sql
-- src/main/resources/db/callback/beforeMigrate.sql
set local lock_timeout = '5s';
set local statement_timeout = '10min';
```

**Liquibase YAML — задать таймауты перед шагом**

```yaml
databaseChangeLog:
  - changeSet:
      id: safe-idx
      author: team
      changes:
        - sql: "set local lock_timeout='5s'; set local statement_timeout='10min';"
        - sql: "create index concurrently if not exists idx_users_email on users(email);"
```

**Gradle (Groovy) — сгенерировать updateSQL для ревью**

```groovy
tasks.register('lbUpdateSQL') {
  doLast {
    ant.invokeMethod('liquibase', [
      'changeLogFile': 'src/main/resources/db/changelog/db.changelog.yaml',
      'url': System.getenv('DB_URL'),
      'username': System.getenv('DB_USER_MIG'),
      'password': System.getenv('DB_PASS_MIG'),
      'logLevel': 'info',
      'promptOnNonLocalDatabase': 'false',
      'command': 'updateSQL'
    ])
  }
}
```

**Gradle (Kotlin) — то же**

```kotlin
tasks.register("lbUpdateSQL") {
    doLast {
        ant.invokeMethod("liquibase", mapOf(
            "changeLogFile" to "src/main/resources/db/changelog/db.changelog.yaml",
            "url" to System.getenv("DB_URL"),
            "username" to System.getenv("DB_USER_MIG"),
            "password" to System.getenv("DB_PASS_MIG"),
            "logLevel" to "info",
            "promptOnNonLocalDatabase" to "false",
            "command" to "updateSQL"
        ))
    }
}
```

**Java — установить таймауты на уровне подключения (fallback)**

```java
package com.example.timeout;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class SessionTimeouts {
  public SessionTimeouts(JdbcTemplate jdbc) {
    jdbc.execute("set lock_timeout='5s'; set statement_timeout='10min';");
  }
}
```

**Kotlin — то же**

```kotlin
package com.example.timeout

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Component

@Component
class SessionTimeouts(jdbc: JdbcTemplate) {
    init {
        jdbc.execute("set lock_timeout='5s'; set statement_timeout='10min';")
    }
}
```

---

Суммируя, стратегия продакшн-миграций складывается из дисциплины профилей, разнесения прав, осознанного выбора способа запуска, аккуратной работы с блокировками, поддержки мультиарендности, чистого разделения сидов и строгого управления временем. Эти практики не только снижают риск простоя, но и делают релизы предсказуемыми для всей команды — от разработчиков до DBA и SRE.


# 7. Тестирование и качество миграций

## Локальная проверка: Testcontainers + миграции на чистой БД «с нуля»

В продакшн-командах критично иметь быстрый, повторяемый и изолированный способ проверить, что ваши миграции применяются «с нуля» к пустой базе и дают корректную схему. **Testcontainers** решает это изящно: каждый тест поднимает «свежее» окружение (например, PostgreSQL 15) в Docker-контейнере, вы исключаете влияние локальных установок, и результат становится детерминированным. Такая проверка — первая линия обороны против «дрейфа» схемы и накопленных ручных фиксов.

При локальной проверке нужно мыслить не только «DDL прошёл», а «приложение сможет работать поверх этой схемы». Поэтому, помимо запуска мигратора (Flyway/Liquibase), мы обязательно выполняем минимальные smoke-квери: проверить наличие ключевых таблиц/индексов/констрейнтов, выполнить вставку/чтение, убедиться в валидности дефолтов и not null-ограничений. Эти шаги ловят ошибки, которые «проскальзывают», если только смотреть на лог «Successfully applied N migrations».

Изоляция от `spring.jpa.hibernate.ddl-auto` обязательна: в тестах миграций схема управляется **только** мигратором. Hibernate-автогенерацию схемы выключайте (или вовсе не подключайте spring-контекст) в этих проверках. Это исключает ситуацию «работает, потому что Hibernate допилил за вас».

Организуйте артефакты: храните миграции в `classpath:db/migration` (Flyway) или `classpath:db/changelog` (Liquibase). Для монорепо/мульти-модулей используйте несколько `locations`/`includeAll` и согласованные префиксы. Это позволит запускать один и тот же тест над конкретной подсхемой или доменом.

Есть два способа запускать миграции в тесте: через Spring Boot (поднимая контекст и давая авто-конфигурации сделать своё дело) или **программно** через API Flyway/Liquibase. Второй способ быстрее, легче дебажить и не требует прогрева целого приложения. Поэтому для smoke-тестов предпочитаем программный путь, а интеграционные тесты Spring оставляем для сценариев «приложение+ORM».

Следите за версией Postgres, соответствующей продакшн-кластерам. Если в проде уже 14/15/16, локально тестируйте именно её. Баги и особенности DDL (например, поведение `CONCURRENTLY`, `ALTER TYPE ... USING`) версиозависимы, и «пройдёт локально на 13» — плохой индикатор для продакшна на 16.

Плейсхолдеры и секреты. Для локальных тестов удобно использовать чистый JDBC URL контейнера и прямые креды. Плейсхолдеры в миграциях (Flyway placeholders / Spring placeholders) подставляйте через API конфигурации. Это позволяет в тесте менять схемы/имена индексов/параметры без редактирования SQL.

Повторяемые миграции (Flyway `R__`) и Liquibase-changeset’ы с `runOnChange` также должны входить в smoke-прогон. Важно проверить, что хеши/контрольные суммы правильно пересчитываются и повторяемки не «пляшут» между прогонами.

Для скорости используйте re-use контейнеров Testcontainers только в «разогреве» локально; в CI лучше поднимать «чисто» каждый раз. Перезапуск контейнера гарантирует, что ваша миграция не зависит от предшествующего состояния.

Наконец, не забывайте про логику «после миграции»: если вы используете callback’и (инициализационные данные, заполнение справочников), smoke-тест должен подтвердить, что эти данные есть и пригодны к чтению. Это снимает риск «схема есть, но приложенияm нечего читать».

**Gradle зависимости (общие для тестов миграций)**
*Groovy DSL:*

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.flywaydb:flyway-core"
    implementation "org.liquibase:liquibase-core"
    testImplementation "org.junit.jupiter:junit-jupiter:5.10.3"
    testImplementation "org.testcontainers:junit-jupiter:1.20.3"
    testImplementation "org.testcontainers:postgresql:1.20.3"
    testImplementation "org.springframework:spring-jdbc" // для ScriptUtils при желании
}
test {
    useJUnitPlatform()
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.flywaydb:flyway-core")
    implementation("org.liquibase:liquibase-core")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.3")
    testImplementation("org.testcontainers:junit-jupiter:1.20.3")
    testImplementation("org.testcontainers:postgresql:1.20.3")
    testImplementation("org.springframework:spring-jdbc")
}
tasks.test {
    useJUnitPlatform()
}
```

**Java (Flyway, smoke-тест с нуля)**

```java
package demo.migration;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Assertions;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

@Testcontainers
class FlywaySmokeTest {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:15-alpine");

    @Test
    void migrateAndCheck() throws Exception {
        Flyway flyway = Flyway.configure()
                .dataSource(POSTGRES.getJdbcUrl(), POSTGRES.getUsername(), POSTGRES.getPassword())
                .locations("classpath:db/migration")
                .baselineOnMigrate(false)
                .cleanDisabled(false) // в тесте можно чистить
                .load();

        flyway.clean(); // начать с нуля
        flyway.migrate();

        try (Connection c = DriverManager.getConnection(
                POSTGRES.getJdbcUrl(), POSTGRES.getUsername(), POSTGRES.getPassword());
             Statement st = c.createStatement()) {
            ResultSet rs = st.executeQuery("select count(*) from information_schema.tables where table_name='person'");
            rs.next();
            int cnt = rs.getInt(1);
            Assertions.assertTrue(cnt >= 1, "Таблица person должна существовать");
        }
    }
}
```

**Kotlin (Liquibase, smoke-тест с нуля)**

```kotlin
package demo.migration

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.sql.DriverManager

@Testcontainers
class LiquibaseSmokeTest {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:15-alpine")
    }

    @Test
    fun migrateAndCheck() {
        DriverManager.getConnection(pg.jdbcUrl, pg.username, pg.password).use { conn ->
            val db = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
            val accessor = ClassLoaderResourceAccessor()
            val lb = Liquibase("db/changelog/master.yaml", accessor, db)

            // Чистка в тесте (не делайте это в проде)
            conn.createStatement().use { it.execute("drop schema if exists public cascade; create schema public;") }

            lb.update() // применяем master changelog

            conn.createStatement().use {
                val rs = it.executeQuery("select count(*) from information_schema.tables where table_name='person'")
                rs.next()
                assertTrue(rs.getInt(1) >= 1)
            }
        }
    }
}
```

**Пример Liquibase changelog (YAML)**

```yaml
databaseChangeLog:
  - changeSet:
      id: 001-init
      author: demo
      changes:
        - createTable:
            tableName: person
            columns:
              - column:
                  name: id
                  type: BIGINT
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: name
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
```

**Тот же changelog (XML)**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="
     http://www.liquibase.org/xml/ns/dbchangelog
     http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
    <changeSet id="001-init" author="demo">
        <createTable tableName="person">
            <column name="id" type="BIGINT">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="name" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

---

## «Семантические» тесты: миграция старых дампов до HEAD и валидация схемы/данных

Smoke-тесты проверяют структуру «с нуля», но реальный мир сложнее: у вас есть прод-данные, старые версии схемы и исторические артефакты. **Семантические тесты** берут снимок базы (дамп) старой версии, поднимают его в контейнере, прогоняют миграции до HEAD и затем **валидируют смысл**: констрейнты, заполненность новых колонок, корректность преобразований DML. Так вы убеждаетесь, что не только «применилось», но и «данные остались консистентными».

Где взять дампы? Подготовьте в репозитории несколько «эталонных» фикстур: минимальный (микро-датасет), средний (тысячи строк), и «угловые» случаи (например, старые enum-значения, null-аномалии). Дамп можно хранить в сжатом виде и разворачивать перед тестом, исполняя SQL.

После применения миграций важно проверять не только «существование колонок», но и **инварианты**: `NOT NULL` действительно не нарушается; значения по умолчанию проставились; сложные преобразования (`ALTER TYPE`, `split column`, нормализация данных) дали ожидаемый результат. Здесь пригодятся «контрольные запросы» и подготовленные asserts.

Скриптовый раннер. Можно использовать `ScriptUtils` из `spring-jdbc` или простой «разделитель по `;`», если дампы не содержат хитрых конструкций. Для больших дампов лучше грузить `pg_restore` отдельно в контейнер Postgres, но для unit-уровня хватит SQL.

Семантическая проверка особенно полезна, если вы используете Liquibase-rollback/теги. Вы можете прогнать *несколько* промежуточных состояний: `tag v1 -> update to v2 -> validate`, затем `v2 -> v3`, ловя регрессии в серединных шагах, а не только на «HEAD».

Не забывайте про нестандартные объекты (последовательности, индексы, матвью, функции). Счётчики `nextval` должны остаться валидными, индексы — актуальными, а функции — компилироваться. Для этого включайте в валидацию метаданные `pg_catalog`.

Проверка семантики — это ещё и **документация контракта**: тесты читаемо объясняют, что должно случиться с данными при апгрейде. В ревью к миграциям полезно ссылаться на эти тесты как на подтверждение инвариантов.

Прогоните эти тесты в CI на nightly-ветке. Полный дамп может быть тяжёлым для PR-проверки, но ночной прогон отловит дрейф, если кто-то добавил миграции, ломающее преобразование или изменил порядок файлов.

Наконец, не смешивайте «семантические» тесты с unit-тестами доменной логики — это отдельный набор «миграционных контрактов», запускаемый целенаправленно. Отделение снижает шум и ускоряет обычные сборки.

**Java (импорт SQL-дампа + Flyway миграция + проверки)**

```java
package demo.migration;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Assertions;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.sql.*;

@Testcontainers
class SemanticDumpToHeadTest {

    @Container
    static final PostgreSQLContainer<?> PG = new PostgreSQLContainer<>("postgres:15-alpine");

    @Test
    void migrateOldDumpAndValidateData() throws Exception {
        try (Connection c = DriverManager.getConnection(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword());
             Statement st = c.createStatement()) {

            // Заливаем старый дамп (упрощённый loader, подходит для небольших SQL)
            String sql = Files.readString(Paths.get("src/test/resources/dumps/v1_small.sql"), StandardCharsets.UTF_8);
            for (String s : sql.split(";")) {
                String trimmed = s.trim();
                if (!trimmed.isEmpty()) st.execute(trimmed);
            }
        }

        Flyway flyway = Flyway.configure()
                .dataSource(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword())
                .locations("classpath:db/migration")
                .load();
        flyway.migrate();

        try (Connection c = DriverManager.getConnection(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword());
             Statement st = c.createStatement()) {

            // Пример семантической проверки: новая колонка not null и заполнена дефолтом
            ResultSet rs = st.executeQuery(
                    "select count(*) from person where new_flag is null");
            rs.next();
            Assertions.assertEquals(0, rs.getInt(1), "new_flag должен быть заполнен после миграции");
        }
    }
}
```

**Kotlin (импорт SQL-дампа + Liquibase update + проверки)**

```kotlin
package demo.migration

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.nio.file.Files
import java.nio.file.Paths
import java.sql.DriverManager

@Testcontainers
class SemanticDumpToHeadLiquibaseTest {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:15-alpine")
    }

    @Test
    fun migrateOldDumpAndValidateData() {
        DriverManager.getConnection(pg.jdbcUrl, pg.username, pg.password).use { conn ->
            conn.createStatement().use { st ->
                val sql = Files.readString(Paths.get("src/test/resources/dumps/v1_small.sql"))
                sql.split(";").map { it.trim() }.filter { it.isNotEmpty() }.forEach { st.execute(it) }
            }

            val db = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
            val lb = Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db)
            lb.update()

            conn.createStatement().use { st ->
                val rs = st.executeQuery("select count(*) from person where new_flag is null")
                rs.next()
                assertEquals(0, rs.getInt(1))
            }
        }
    }
}
```

---

## Проверка обратной совместимости (expand/contract): старый код на новой схеме и наоборот

«Expand/contract» — базовый подход к zero-downtime. Сначала **расширяем** схему так, чтобы старая версия приложения могла продолжать работать (добавляем новую колонку с дефолтом, создаём совместимые представления, сохраняем старые колонки). Затем включаем новую логику записи/чтения, и лишь позже **сжимаем** («contract») схему, удаляя устаревшее. Проверки должны доказывать: старый код живёт на новой схеме, новый код — на старой (в пределах допустимого).

Что значит «старый код на новой схеме» на уровне теста? Мы имитируем поведение старой версии: выполняем её характерные запросы (например, инсерты без новых колонок), убеждаемся, что дефолты и триггеры обеспечивают корректность данных. Параллельно проверяем, что новые индексы/колонки не ломают существующие планы выполнения.

«Новый код на старой схеме» тестируется реже (в идеале, мы не деплоим новый код до expand-шага), но бывает нужно для горячих фиксов. Тогда проверки подтверждают, что новый код умеет работать в режиме «фича выключена», не требуя наличия новых колонок (feature flag/условная логика ORM).

Ещё аспект совместимости — **виды**. Удобно добавлять `compat`-представления, которые маскируют изменения схемы для старого кода. Тест должен убедиться, что старые запросы к виду возвращают тот же набор колонок и семантику значений, что и до миграции. Это защищает от регрессий при рефакторинге таблиц.

В Liquibase проверка совместимости легко комбинируется с тегами: вы можете проставить `tag` на шаге expand, запускать приложение «старой версии» на этом теге, затем делать `update` до следующего тега для «нового кода». Тесты по тегам документируют этапы развёртки.

Важно помнить про транзакционность DDL: некоторые изменения в Postgres не транзакционны (`CREATE INDEX CONCURRENTLY`), и тесты должны учитывать реальное поведение (автокоммит, невозможность отката). Это влияет на дизайн откатов и на сценарии временной совместимости.

Проверки «expand-before-contract» стоит сделать обязательными в CI для всех PR, где меняется схема. Шаблон теста облегчает жизнь: разработчик добавляет маленький кусок для своей сущности, а инфраструктура проверки остаётся общей.

С точки зрения эксплуатации такие проверки — не только «качество миграций», но и **снижение риска релиза**: если что-то пойдёт не так, у вас есть уверенность, что старая версия сможет пережить частично применённые шаги, пока вы исправляете проблему.

**Java (эмуляция старого кода: insert без новой колонки, дефолт заполняется)**

```java
package demo.migration;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Assertions;

import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.*;

@Testcontainers
class ExpandContractCompatibilityTest {

    @Container
    static final PostgreSQLContainer<?> PG = new PostgreSQLContainer<>("postgres:15-alpine");

    @Test
    void oldInsertOnNewSchema() throws Exception {
        Flyway flyway = Flyway.configure()
                .dataSource(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword())
                .locations("classpath:db/migration")
                .load();
        flyway.migrate(); // в миграциях добавлена новая колонка new_flag boolean not null default false

        try (Connection c = DriverManager.getConnection(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword());
             Statement st = c.createStatement()) {
            st.executeUpdate("insert into person(id, name) values (1, 'Alice')"); // старый код не знает new_flag
            ResultSet rs = st.executeQuery("select new_flag from person where id=1");
            rs.next();
            Assertions.assertFalse(rs.getBoolean(1), "Дефолт для new_flag должен быть false");
        }
    }
}
```

**Kotlin (совместимость через view для старых запросов)**

```kotlin
package demo.migration

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.sql.DriverManager

@Testcontainers
class ViewCompatibilityTest {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:15-alpine")
    }

    @Test
    fun viewProvidesOldContract() {
        DriverManager.getConnection(pg.jdbcUrl, pg.username, pg.password).use { conn ->
            val db = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
            val lb = Liquibase("db/changelog/expand.yaml", ClassLoaderResourceAccessor(), db)
            lb.update()

            conn.createStatement().use { st ->
                st.executeUpdate("insert into person(id, name) values (1,'Bob')")
                val rs = st.executeQuery("select id, name from person_compat_view where id=1")
                rs.next()
                assertEquals("Bob", rs.getString("name"))
            }
        }
    }
}
```

**Liquibase (expand-шаг, YAML)**

```yaml
databaseChangeLog:
  - changeSet:
      id: 100-expand
      author: demo
      changes:
        - addColumn:
            tableName: person
            columns:
              - column:
                  name: new_flag
                  type: boolean
                  defaultValueBoolean: false
        - addNotNullConstraint:
            tableName: person
            columnName: new_flag
            columnDataType: boolean
        - createView:
            viewName: person_compat_view
            selectQuery: "select id, name from person"
```

**Liquibase (expand-шаг, XML)**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
    <changeSet id="100-expand" author="demo">
        <addColumn tableName="person">
            <column name="new_flag" type="boolean" defaultValueBoolean="false"/>
        </addColumn>
        <addNotNullConstraint tableName="person" columnName="new_flag" columnDataType="boolean"/>
        <createView viewName="person_compat_view">
            <selectQuery>select id, name from person</selectQuery>
        </createView>
    </changeSet>
</databaseChangeLog>
```

---

## Контроль дрейфа: `validate` (Flyway), `diff` (Liquibase) против продакшн-схемы

Даже при дисциплине «все изменения — через миграции» со временем появляется дрейф: кто-то поменял индекс в проде руками, пропустили файл, конфликт версий. Инструменты помогают ловить это автоматически. У Flyway есть `validate`: он проверяет соответствие истории и хешей миграций факту в `flyway_schema_history`. Liquibase предоставляет `validate`, `diff`, `diffChangeLog`: можно сравнить целевую БД со «снимком» или описанием, получая детальный отчёт.

`validate` стоит запускать в каждом старте приложения (или как минимум в CI), чтобы гарантировать, что набор файлов на диске совпадает с тем, что уже применено в БД. Любая правка содержимого уже применённого файла должна быть запрещена; единственный безопасный путь — **новая миграция**.

`diff` в Liquibase полезен для аудита продакшн-схемы: сравнить прод со Stage или с эталонным snapshot’ом. По результату вы либо создаёте changeLog, чтобы «привести к эталону», либо фиксируете, что расхождение ожидаемо (и документируете его). Этот шаг хорошо автоматизировать nightly-таской с артефактом отчёта.

Важно понимать ограничения: сравнение схем не всегда «понимает» функциональную эквивалентность (например, порядок колонок не важен, а diff его покажет). Фильтруйте шум и настраивайте правила игнора (схемы, временные объекты, служебные индексы).

Для Flyway полезно хранить и проверять **checksum**: если файл перезаписали, `validate` упадёт. В редких случаях (например, изменили комментарий в SQL) допустим `flyway repair` — но это должно быть осознанным решением, задокументированным в MR.

Периодический diff прод против кода — не только про «найти ошибку», но и про прозрачность: вы создаёте репорт, который читает релиз-менеджер/DBA, видит, что структура соответствует ожиданиям. Это снижает риск «тихих» ручных фиксов, которые копятся и потом «взрываются».

Важная практика — **фиксировать версию схемы** (например, тег/версию релиза) и при diff ссылаться на неё. Тогда отчёт «прод против v1.12.3» становится частью артефактов релиза и его можно поднять пост-фактум.

Не забывайте про многосхемные базы: запускайте сравнение по всем схемам, которые затрагивает приложение. Иначе легко пропустить дрейф в «второстепенной» схеме.

**Java (Flyway validate как тест качества)**

```java
package demo.migration;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.exception.FlywayValidateException;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Assertions;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class FlywayValidateTest {

    @Container
    static final PostgreSQLContainer<?> PG = new PostgreSQLContainer<>("postgres:15-alpine");

    @Test
    void validateChecksumsAndState() {
        Flyway flyway = Flyway.configure()
                .dataSource(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword())
                .locations("classpath:db/migration")
                .load();

        flyway.migrate();
        try {
            flyway.validate();
        } catch (FlywayValidateException e) {
            Assertions.fail("Дрейф миграций: " + e.getMessage());
        }
    }
}
```

**Kotlin (Liquibase validate + diff в файл)**

```kotlin
package demo.migration

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.diff.DiffGeneratorFactory
import liquibase.diff.compare.CompareControl
import liquibase.diff.output.changelog.DiffToChangeLog
import liquibase.resource.ClassLoaderResourceAccessor
import org.junit.jupiter.api.Test
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.io.File
import java.sql.DriverManager

@Testcontainers
class LiquibaseValidateDiffTest {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:15-alpine")
    }

    @Test
    fun validateAndDiff() {
        DriverManager.getConnection(pg.jdbcUrl, pg.username, pg.password).use { conn ->
            val db = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
            val lb = Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db)

            lb.update()
            val errors = lb.validate()
            if (errors.hasErrors()) error("Liquibase validate errors: $errors")

            // Дифф против пустой базы как пример (в реале сравните с snapshot/эталоном)
            val reference = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn)) // здесь мог бы быть другой источник
            val diffResult = DiffGeneratorFactory.getInstance()
                .compare(reference, db, CompareControl.STANDARD)
            val out = File("build/liquibase-diff.xml")
            out.parentFile.mkdirs()
            DiffToChangeLog(diffResult).print(out.outputStream())
        }
    }
}
```

---

## Статические проверки: линтеры SQL, запрет `DROP` вне контекста, forbid long locks

Часть проблем миграций легче всего ловить до запуска БД — простыми статическими проверками. Примеры: **запрет `DROP TABLE`** вне особого контекста (labels/contexts), запрет `ALTER TABLE ... DROP COLUMN` без подтверждённого contract-шага, запрет `CREATE INDEX` без `CONCURRENTLY` для больших таблиц. Такие правила тривиально реализуются как файлопроход с регулярками.

Для команд, где много SQL, имеет смысл добавить специализированные линтеры. Но даже самодельный «сторожевой» тест даёт отличную отдачу: разработчик мгновенно видит, что миграция нарушает политики продакшн-устойчивости, и исправляет раньше ревью.

Статические проверки особенно полезны для внешних контрибьюторов или новичков в проекте. Вы формализуете «устные правила» (например, **никаких `clean`/`drop`** в проде), и CI блокирует небезопасные изменения.

Список правил зависит от базы и SLA. В Postgres для нагруженных таблиц проверяйте, что индексы создаются `CONCURRENTLY`, что `NOT VALID` используется для констрейнтов, которые валидируются позже, что нет `VACUUM FULL`/`REINDEX` без соответствующих окон.

Полезно проверять и **размер миграций**: длинные файлы с сотнями DML-операций обычно признак «слишком много за один релиз». Разбейте на партии и добавьте комменты-маркеры прогресса.

Статические проверки не заменяют интеграционные тесты, но сокращают цикл обратной связи. Лучшее место — отдельная Gradle-таска, запускаемая в CI до Testcontainers-тестов. Ошибки — быстрые и понятные.

Регулярки аккуратно настраивайте: избегайте ложных срабатываний в комментариях. Наивная проверка `contains("drop table")` будет шуметь. Фильтруйте комментарии и строки, учитывайте регистр.

Результаты линта сохраняйте как артефакт (например, `build/migration-lint-report.txt`). Это позволит ревьюеру быстро увидеть, какие файлы прошли/провалились, и почему.

**Java (простейший «сторож» для SQL-миграций)**

```java
package demo.migration;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Assertions;

import java.io.IOException;
import java.nio.file.*;
import java.util.List;

class SqlLinterTest {

    @Test
    void forbidDangerousCommands() throws IOException {
        Path root = Paths.get("src/main/resources/db/migration");
        StringBuilder report = new StringBuilder();
        Files.walk(root)
                .filter(p -> p.toString().endsWith(".sql"))
                .forEach(p -> {
                    try {
                        String content = Files.readString(p);
                        String noComments = content.replaceAll("(?s)/\\*.*?\\*/", "")
                                                   .replaceAll("(?m)--.*$", "");
                        List<String> rules = List.of(
                                "(?i)\\bdrop\\s+table\\b",
                                "(?i)\\balter\\s+table\\b.*\\bdrop\\s+column\\b",
                                "(?i)\\bcreate\\s+index\\b(?!.*\\bconcurrently\\b).*\\bon\\b.*\\b(big_)?table\\b"
                        );
                        for (String r : rules) {
                            if (noComments.matches("(?s).*" + r + ".*")) {
                                report.append("Forbidden pattern ").append(r)
                                        .append(" in ").append(p).append("\n");
                            }
                        }
                    } catch (IOException e) {
                        report.append("Failed to read ").append(p).append(": ").append(e.getMessage()).append("\n");
                    }
                });
        if (report.length() > 0) {
            Assertions.fail("SQL lint failed:\n" + report);
        }
    }
}
```

**Kotlin (тот же лент)**

```kotlin
package demo.migration

import org.junit.jupiter.api.Assertions.fail
import org.junit.jupiter.api.Test
import java.nio.file.Files
import java.nio.file.Paths

class SqlLinterKtTest {

    @Test
    fun forbidDangerousCommands() {
        val root = Paths.get("src/main/resources/db/migration")
        val report = StringBuilder()
        Files.walk(root).filter { it.toString().endsWith(".sql") }.forEach { p ->
            val content = Files.readString(p)
            val noComments = content
                .replace(Regex("(?s)/\\*.*?\\*/"), "")
                .replace(Regex("(?m)--.*$"), "")
            val rules = listOf(
                Regex("(?i)\\bdrop\\s+table\\b"),
                Regex("(?i)\\balter\\s+table\\b.*\\bdrop\\s+column\\b"),
                Regex("(?i)\\bcreate\\s+index\\b(?!.*\\bconcurrently\\b).*\\bon\\b.*\\b(big_)?table\\b")
            )
            rules.forEach { r ->
                if (r.containsMatchIn(noComments)) {
                    report.append("Forbidden pattern ${r.pattern} in $p\n")
                }
            }
        }
        if (report.isNotEmpty()) fail("SQL lint failed:\n$report")
    }
}
```

**Gradle таска (Groovy DSL)**

```groovy
tasks.register("lintMigrations") {
    group = "verification"
    description = "Static checks for SQL migrations"
    dependsOn test // или запустите отдельно, если хотите
}
```

**Gradle таска (Kotlin DSL)**

```kotlin
tasks.register("lintMigrations") {
    group = "verification"
    description = "Static checks for SQL migrations"
    dependsOn("test")
}
```

---

## Производительность: замер времени на больших таблицах в CI на синтетических данных

Даже корректные миграции могут быть **слишком медленными**. В реальности это значит простой, таймауты, реплика-лаг. Поэтому добавляем перформанс-тесты миграций на синтетических данных. Идея проста: в контейнерную базу загружаем N миллионов (или «приближённых») строк, запускаем «тяжёлую» миграцию (например, индексирование, пересчёт колонки), меряем время и проверяем, что оно в заданном бюджете.

Синтетические данные должны быть «похожи» на реальные: разреженность, распределение, размеры строк. Минимум — количество строк и ширина. В идеале — ещё и кардинальность значений для индексируемых колонок.

Меряем не только «общее время», но и **побочные эффекты**: блокировки, рост размера индексов, влияние на планы запросов. Простейший подход — считать время и выполнять `EXPLAIN ANALYZE` для ключевых запросов до/после миграции.

`CREATE INDEX CONCURRENTLY` — обязательный выбор для больших таблиц в Postgres, но он имеет собственные ограничения (не в транзакции, может быть длиннее по времени). Тест должен это учитывать: отключите автокоммит только там, где позволено, и запускайте индекс отдельно.

Порог времени — не догма. Для PR-веток можно ставить «мягкие» лимиты (логировать, но не падать), а для `main` — «жёсткие», чтобы не деградировать от релиза к релизу. Храните историю (метрики времени) как артефакт, чтобы видеть тренды.

Для больших объёмов данные лучше грузить пакетами (batch insert) — так тест занимает минуты, а не часы. Можно использовать временные таблицы, потом `INSERT INTO ... SELECT` в основную.

Не забывайте про `maintenance_work_mem` и прочие настройки Postgres, влияющие на скорость индексации. В Testcontainers вы можете задавать параметры командной строки при старте контейнера (`withCommand`), приближая условия к продакшну.

Иногда выгодно **разнести** DDL и DML: в одном релизе добавили колонку и индексы, в другом — наполнили её данными батчами. Перформанс-тест должен подсветить, когда «за один раз» становится слишком дорого.

Наконец, включите в отчёт размер таблицы/индексов и оценку IO. Это поможет аргументированно обсуждать с DBA изменения бюджета.

**Java (замер времени индексации, упрощённый пример)**

```java
package demo.migration;

import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Assertions;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.sql.*;
import java.time.Duration;

@Testcontainers
class MigrationPerformanceTest {

    @Container
    static final PostgreSQLContainer<?> PG = new PostgreSQLContainer<>("postgres:15-alpine");

    @Test
    void indexCreationWithinBudget() throws Exception {
        Flyway flyway = Flyway.configure()
                .dataSource(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword())
                .locations("classpath:db/migration")
                .load();
        flyway.migrate();

        try (Connection c = DriverManager.getConnection(PG.getJdbcUrl(), PG.getUsername(), PG.getPassword());
             Statement st = c.createStatement()) {

            st.execute("create table if not exists events(id bigserial primary key, ts timestamp not null, payload text)");
            c.setAutoCommit(false);
            try (PreparedStatement ps = c.prepareStatement("insert into events(ts, payload) values (now(), ?)")) {
                for (int i = 0; i < 100_000; i++) {
                    ps.setString(1, "x".repeat(50));
                    ps.addBatch();
                    if (i % 1000 == 0) ps.executeBatch();
                }
                ps.executeBatch();
            }
            c.commit();
            c.setAutoCommit(true);

            long start = System.nanoTime();
            st.execute("create index concurrently if not exists idx_events_ts on events(ts)");
            long millis = Duration.ofNanos(System.nanoTime() - start).toMillis();

            System.out.println("Index build ms: " + millis);
            Assertions.assertTrue(millis < 60_000, "Индекс строится слишком долго");
        }
    }
}
```

**Kotlin (тот же сценарий)**

```kotlin
package demo.migration

import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import java.sql.DriverManager
import java.time.Duration

@Testcontainers
class MigrationPerformanceKtTest {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:15-alpine")
    }

    @Test
    fun indexCreationWithinBudget() {
        DriverManager.getConnection(pg.jdbcUrl, pg.username, pg.password).use { conn ->
            conn.createStatement().use { st ->
                st.execute("create table if not exists events(id bigserial primary key, ts timestamp not null, payload text)")
            }
            conn.autoCommit = false
            conn.prepareStatement("insert into events(ts, payload) values (now(), ?)").use { ps ->
                for (i in 0 until 100_000) {
                    ps.setString(1, "x".repeat(50))
                    ps.addBatch()
                    if (i % 1000 == 0) ps.executeBatch()
                }
                ps.executeBatch()
            }
            conn.commit()
            conn.autoCommit = true

            val start = System.nanoTime()
            conn.createStatement().use { it.execute("create index concurrently if not exists idx_events_ts on events(ts)") }
            val millis = Duration.ofNanos(System.nanoTime() - start).toMillis()
            println("Index build ms: $millis")
            assertTrue(millis < 60_000, "Индекс строится слишком долго")
        }
    }
}
```

---

## Репортинг: артефакт логов миграций в CI, хранение истории для аудита

Качество — это ещё и прозрачность. Полезно сохранять **отчёт о миграциях** как артефакт CI: какие версии применены, сколько заняли, были ли повторяемые шаги, какие параметры окружения. Такой файл прикладывается к релизу и формирует «бумажный след» для аудита.

Flyway и Liquibase позволяют получить «информацию о состоянии». В тесте/джобе после миграции вы можете опросить состояние API и записать в файл: текущая версия, список pending, длительности шагов (если логируете), путь к каждому файлу. Эти данные пригодятся при разборе инцидентов: «что именно было применено?».

Хорошая практика — **логировать каждый changeSet/файл** с начала и конца, фиксируя длительность. В Liquibase для этого можно использовать листенеры/логгеры; во Flyway — просто измерять время вокруг `migrate()` и, при необходимости, разбивать миграции на более мелкие шаги или использовать callbacks.

Отдельно сохраняйте **дамп схемы после миграций** (например, `pg_dump --schema-only`). В CI можно положить этот файл рядом с отчётом. Сравнение дампов между релизами — быстрый способ понять, что действительно изменилось.

Артефакты — часть культуры «infrastructure as code»: если релиз зафиксировал `migration-report.txt` и `schema.sql`, восстановить картину через месяц проще, чем искать в логах. Это экономит часы расследований.

Путь репорта задайте стабильный (`build/reports/migration-report.txt`), чтобы CI легко подхватывал его. Не забывайте чистить секреты (URL без паролей), иначе артефакт нельзя будет хранить.

При многосервисном деплое собирайте отчёты по каждому сервису с указанием версии. Это поможет координировать мультисервисные изменения схемы.

Наконец, добавьте «человеческое» summary для релиз-нот: кратко, какие таблицы/колонки затронуты, есть ли обратимые шаги, ожидались ли долгие операции. Это повышает качество коммуникаций с продуктом и поддержкой.

**Java (формирование простого отчёта Flyway)**

```java
package demo.migration;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Arrays;

class MigrationReportTest {

    @Test
    void writeFlywayReport() throws IOException {
        // В реальном CI параметры берём из env/секретов
        String url = System.getProperty("db.url", "jdbc:postgresql://localhost:5432/postgres");
        String user = System.getProperty("db.user", "postgres");
        String pass = System.getProperty("db.pass", "postgres");

        Flyway flyway = Flyway.configure()
                .dataSource(url, user, pass)
                .locations("classpath:db/migration")
                .load();
        flyway.migrate();

        var info = flyway.info();
        StringBuilder sb = new StringBuilder();
        sb.append("Current: ").append(info.current() != null ? info.current().getVersion() : "none").append("\n");
        sb.append("Applied migrations:\n");
        for (MigrationInfo mi : info.applied()) {
            sb.append("- ").append(mi.getVersion()).append(" ")
              .append(mi.getDescription())
              .append(" [").append(mi.getType()).append("]\n");
        }
        Path out = Path.of("build/reports/migration-report.txt");
        Files.createDirectories(out.getParent());
        Files.writeString(out, sb.toString());
    }
}
```

**Kotlin (Liquibase: summary после update)**

```kotlin
package demo.migration

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import org.junit.jupiter.api.Test
import java.nio.file.Files
import java.nio.file.Path
import java.sql.DriverManager

class LiquibaseReportTest {

    @Test
    fun writeLiquibaseReport() {
        val url = System.getProperty("db.url", "jdbc:postgresql://localhost:5432/postgres")
        val user = System.getProperty("db.user", "postgres")
        val pass = System.getProperty("db.pass", "postgres")

        DriverManager.getConnection(url, user, pass).use { conn ->
            val db = DatabaseFactory.getInstance()
                .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
            val lb = Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db)
            lb.update()

            val report = buildString {
                appendLine("Liquibase Status:")
                val status = lb.reportStatus(true, null, System.out)
                appendLine("See console for detailed status: $status")
            }

            val out = Path.of("build/reports/migration-report.txt")
            Files.createDirectories(out.parent)
            Files.writeString(out, report)
        }
    }
}
```

**Gradle (Groovy DSL): публикация артефакта**

```groovy
tasks.register("archiveMigrationReport", Zip) {
    group = "distribution"
    from("build/reports") { include "migration-report.txt" }
    archiveFileName = "migration-report.zip"
}
artifacts {
    archives tasks.named("archiveMigrationReport")
}
```

**Gradle (Kotlin DSL): публикация артефакта**

```kotlin
tasks.register<Zip>("archiveMigrationReport") {
    group = "distribution"
    from("build/reports") { include("migration-report.txt") }
    archiveFileName.set("migration-report.zip")
}
artifacts {
    add("archives", tasks.named("archiveMigrationReport"))
}
```

**application.yml (пример логирования на старте)**

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    validate-on-migrate: true
  liquibase:
    enabled: false # включайте по профилю, если нужно
logging:
  level:
    org.flywaydb: info
    liquibase: info
```

# 8. Траблшутинг и операции сопровождения

> Ниже — операционные практики и готовые сниппеты для быстрого устранения проблем в проде и на стендах. Для каждого пункта: 10+ абзацев разъяснений и 2 аналогичных примера кода (Java/Kotlin). Где уместно — добавлены Liquibase YAML+XML изменения.

---

## Залипшие блокировки: как безопасно снимать `databasechangeloglock` / чинить `flyway repair`

Проблема «залипшей» блокировки у Liquibase проявляется, когда `update` завершился нештатно (kill -9, OOM, разрыв сети), а строка в `databasechangeloglock` осталась с `locked = true`. Следующий запуск видит «блок заняти», миграции не стартуют, а приложение может падать на старте. В Flyway чаще «залипания» нет (используется row-level lock и транзакции), но после аварии может понадобиться «ремонт» метаданных.

Первый шаг — убедиться, что мигратор действительно *не* работает прямо сейчас. Проверяем CI-джобы, pod’ы, логи и активные подключения к БД. Ошибка многих команд — «снять замок» одновременно с живым мигратором, что создаёт гонку и состояние гоняния разных версий.

Liquibase предоставляет штатный способ: `releaseLocks` (или `forceReleaseLock` через API). Это предпочтительнее прямого SQL, потому что учитывает нюансы разных СУБД и корректно обновляет служебные поля. Ручное `update databasechangeloglock set locked=false` — только в последнюю очередь и строго в «окно».

Для Flyway ключевой инструмент — `repair()`. Он не «снимает» блоки (их там обычно нет), а чинит `flyway_schema_history`: помечает «failed» миграции как исправленные, пересчитывает checksums для уже применённых файлов, убирает «future» записи. Это нужно, если во время падения успели записаться метаданные о начатой миграции.

После разблокировки — валидация. У Flyway: `info()` и `validate()`, у Liquibase: `status`, `validate`. Цель — убедиться, что история консистентна, нет «висячих» changeSet’ов, а следующие запуски пройдут без ручных вмешательств.

Причины «вечных» блоков часто в конфигурации: разные `defaultSchema`/`search_path` по средам и мигратор смотрит «не туда». Иногда `databasechangeloglock` просто в другой схеме или не создан (первый запуск прервался до инициализации). Проверьте схему и параметры подключения.

Избегайте параллельных запусков миграций. Вынесите миграции в отдельную, сериализованную CI-стадию (mutex/lock в пайплайне). Это в разы сокращает шанс получить залипания и конфликтные истории.

Отдельный плейбук для on-call’а обязателен: как проверить активность, как безопасно освободить, как валидировать, как эскалировать. Документированный сценарий уменьшает стресс ночью и ускоряет MTTR.

Если блоки стали частыми — это симптом архитектурной проблемы: приложение стартует «всегда с миграциями», при этом крутится автоскейл, и pod’ы соревнуются за схему. Лечение — отдельная job миграций (K8s Job/Helm hook) перед раскаткой приложения.

После аварии полезно собрать и приложить к инциденту артефакты: лог миграций, `schema dump`, краткий отчёт о состоянии метатаблиц. Это позволит ретроспективно понять первопричину.

Если в процессе разблокировки вы сделали что-то «ручное», фиксируйте это в репозитории (OPERATIONS.md/CHANGELOG) — команда должна знать, что изменяли служебные таблицы и почему.

**Java — безопасное снятие lock’ов Liquibase через API (аналогичный код будет в Kotlin ниже):**

```java
package ops.lock;

import liquibase.Scope;
import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.lockservice.LockService;
import liquibase.lockservice.LockServiceFactory;

import java.sql.Connection;
import java.sql.DriverManager;

public class LiquibaseReleaseLocks {
    public static void main(String[] args) throws Exception {
        String url  = System.getProperty("db.url",  "jdbc:postgresql://localhost:5432/postgres");
        String user = System.getProperty("db.user", "postgres");
        String pass = System.getProperty("db.pass", "postgres");

        try (Connection c = DriverManager.getConnection(url, user, pass)) {
            var db = DatabaseFactory.getInstance()
                    .findCorrectDatabaseImplementation(new JdbcConnection(c));
            LockService lockService = LockServiceFactory.getInstance().getLockService(db);
            Scope.child(Scope.Attr.database.name(), db, () -> lockService.forceReleaseLock());
            System.out.println("Liquibase locks released (if any).");
        }
    }
}
```

**Kotlin — тот же сценарий API Liquibase (аналог Java):**

```kotlin
package ops.lock

import liquibase.Scope
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.lockservice.LockServiceFactory
import java.sql.DriverManager

fun main() {
    val url  = System.getProperty("db.url",  "jdbc:postgresql://localhost:5432/postgres")
    val user = System.getProperty("db.user", "postgres")
    val pass = System.getProperty("db.pass", "postgres")

    DriverManager.getConnection(url, user, pass).use { conn ->
        val db = DatabaseFactory.getInstance()
            .findCorrectDatabaseImplementation(JdbcConnection(conn))
        val lockService = LockServiceFactory.getInstance().getLockService(db)
        Scope.child(Scope.Attr.database.name, db) { lockService.forceReleaseLock() }
        println("Liquibase locks released (if any).")
    }
}
```

---

## Несовпадение checksum/MD5: когда допустим `repair`/`clearCheckSums`, а когда — категорически нельзя

Checksum фиксирует точный контент применённых миграций. Любое редактирование файла после применения — даже переносы строк — изменяет контрольную сумму. Это защитный механизм: он предотвращает «тихую» подмену истории.

Если изменение *несемантическое* (комментарий, форматирование, пустые строки), для Flyway допустим `repair()`, который синхронизирует checksums в `flyway_schema_history` с новым содержимым. Для Liquibase аналог — `clearCheckSums()`: после него при следующем `update` контрольные суммы пересчитаются по фактическим файлам.

Категорический запрет: нельзя «чинить чек-суммы», если менялась семантика DDL/DML — добавили/удалили колонку, изменили тип, поправили данные. В этом случае нужно создавать *новую* миграцию, которая доведёт схему и данные до нужного вида.

Другая причина рассинхронизации — разные плейсхолдеры/параметры окружения. Если в CI/проде подставляются иные значения, а файл использует placeholder-replacement, содержимое исполняемого SQL отличается. Выровняйте параметры (`spring.flyway.placeholders.*`, Liquibase `parameters` и т.д.) и сделайте их стабильными.

В Liquibase `runOnChange=true` сознательно допускает повторный прогон changeSet’а при изменении содержимого. Это полезно для объектов вроде VIEW/FUNCTION. Но применять `runOnChange` на «опасных» DDL нельзя: повторное выполнение должно быть идемпотентным.

Рассинхрон возможен из-за смены пути файла. Liquibase хранит `logicalFilePath`. Если переносите файлы — задавайте `logicalFilePath` явно, чтобы история оставалась сопоставимой между средами.

Поддерживайте политику: «править уже применённые файлы запрещено». Исключения — только до попадания миграции на stage/prod и с обязательным `repair/clearCheckSums` на *всех* стендах, где она побывала. Документируйте исключения в MR.

Хорошая практика — форматировать SQL единообразно (editorconfig/formatter), чтобы не провоцировать фальш-положительные изменения checksum из-за автоформатирования IDE.

При спорных случаях проще и безопаснее создать новую `V__`/changeSet с нужным изменением, чем пытаться «подкрутить» историю. Цена новой миграции мала, а риски «починки» истории велики.

Добавьте тест `flyway.validate()`/`liquibase.validate()` в CI — любое расхождение будет видно сразу в PR, а не в деплое.

**Java — Flyway: безопасный repair только для несемантических правок:**

```java
package ops.checksum;

import org.flywaydb.core.Flyway;

public class RepairChecksums {
    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
                .dataSource("jdbc:postgresql://localhost:5432/postgres", "postgres", "postgres")
                .locations("classpath:db/migration")
                .load();
        flyway.repair();
        System.out.println("Checksums repaired (non-semantic edits only).");
    }
}
```

**Kotlin — Liquibase: пересчёт checksum через `clearCheckSums()` (аналог Java-подхода):**

```kotlin
package ops.checksum

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import java.sql.DriverManager

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        val db = DatabaseFactory.getInstance()
            .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
        val lb = Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db)
        lb.clearCheckSums()
        println("Liquibase checksums cleared; next update() will recalc.")
    }
}
```

---

## Out-of-order: когда включать, как документировать «поздние» версии

Out-of-order у Flyway позволяет применить «опоздавшие» версии (например, `V11_3` после уже применённой `V12`). Это спасает при параллельных ветках. Но каждое такое включение — риск пересечения с уже накатанными изменениями, поэтому режим должен быть осознанным исключением, а не дефолтом.

Минимизируйте необходимость out-of-order организационно: используйте датированные версии (`V2025_10_23_001`) и быстрый merge. Тогда «вошедший позже» файл получает более поздний номер и применяется в естественном порядке.

Если out-of-order всё же включён, ограничьте допустимые действия в «опоздавших» файлах: только добавления/идемпотентные изменения, никаких destructive-операций. Это формализуется как правило ревью.

Документируйте каждую «позднюю» миграцию: зачем она опоздала, какие зависимости учитывает, какие таблицы трогает. В CHANGELOG/OPERATIONS.md фиксируйте «временной контекст».

В Liquibase порядок определяется master-changelog’ом (include-порядок). Аналог out-of-order — добавить include «позднего» файла ниже, чем уже применённые, с `preconditions` на состояние. Это явный и контролируемый способ.

Проверяйте на стейдже сценарий «прод уже впереди, догружаете опоздавший файл». На уровне smoke-тестов моделируйте это через два шага применения включений.

Убедитесь, что репликация, логический декодинг, CDC и прочие интеграции не зависят от жёсткой последовательности DDL — некоторые коннекторы чувствительны к перестановкам.

Если у вас моносхема и много команд — заведите «окно конвергенции»: период, когда все команды обязаны смержить миграции и выровнять порядок до релиза.

Запрещайте Out-of-order на проде через конфигурацию по умолчанию. Разрешайте только точечно (env override) и только после согласования.

Держите тест, который имитирует отставшую ветку и проверяет, что «опоздавшие» миграции не ломают уже применённое.

**Java — Flyway с включённым out-of-order (аналог ниже в Kotlin):**

```java
package ops.ooo;

import org.flywaydb.core.Flyway;

public class FlywayOutOfOrder {
    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
                .dataSource("jdbc:postgresql://localhost:5432/postgres","postgres","postgres")
                .locations("classpath:db/migration")
                .outOfOrder(true)
                .load();
        flyway.migrate();
    }
}
```

**Kotlin — тот же сценарий (аналог Java):**

```kotlin
package ops.ooo

import org.flywaydb.core.Flyway

fun main() {
    val flyway = Flyway.configure()
        .dataSource("jdbc:postgresql://localhost:5432/postgres","postgres","postgres")
        .locations("classpath:db/migration")
        .outOfOrder(true)
        .load()
    flyway.migrate()
}
```

**Liquibase — «поздний» include c preconditions (YAML и XML):**

```yaml
databaseChangeLog:
  - include: { file: db/changelog/001-init.yaml }
  - include: { file: db/changelog/010-feature-a.yaml }
  # поздний фикс
  - include:
      file: db/changelog/009-late-fix.yaml
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
    <include file="db/changelog/001-init.xml"/>
    <include file="db/changelog/010-feature-a.xml"/>
    <!-- поздний фикс -->
    <include file="db/changelog/009-late-fix.xml"/>
</databaseChangeLog>
```

---

## «Чистка» БД (`clean`) — почему запрещать по умолчанию и где всё же применять

Команды `clean` (Flyway) и `dropAll` (Liquibase) уничтожают объекты схемы. В проде это должно быть запрещено технически (конфигом) и организационно (политикой). Любой промах с переменными окружения — и вы стираете данные.

Разрешённые случаи — изолированные среды: локальная разработка, ephemeral-окружения для PR, короткоживущие тестовые стенды. Там «чистка» ускоряет цикл «сломал—почистил—накатил заново».

Никогда не смешивайте пользователей: учётка, которая имеет право `clean/dropAll`, не должна использоваться приложением или CI-пайплайном продакшн-деплоя. Разделяйте секреты и права.

Включайте «страховку» в код утилит: проверяйте `APP_ENV == DEV`/имя схемы/имя базы, прежде чем вызывать «чистку». Ошибка окружения тогда остановит скрипт.

В Spring Boot держите `spring.flyway.clean-disabled=true` во всех профилях, кроме *специально предназначенных* для разработки/демо. Для Liquibase — вообще не автоматизируйте `dropAll`.

После чистки всегда прогоняйте миграции «с нуля» и `validate`. Это ловит дрейф между файлами и ожидаемым состоянием, прежде чем вы отдадите стенд QA/автотестам.

Не позволяйте «чистить» через HTTP-админки приложения. Любые подобные действия — только через отдельные CLI-утилиты вручную и в DEV.

Если схема «засорена» устаревшими объектами, сначала делайте «контрактные» миграции по удалению, а «чистку» — как последний шаг на «одноразовых» стендах.

В мульти-схемных базах чистка должна быть адресной: чистите только целевые схемы, а не всё `public`. Неправильный `search_path` может привести к неприятностям.

На проде вместо «чистки» — откаты Liquibase (rollback/tag) или «forward-fix» во Flyway: новая миграция, исправляющая состояние.

**Java — «железный» запрет clean (аналог в Kotlin):**

```java
package ops.clean;

import org.flywaydb.core.Flyway;

public class ForbidClean {
    public static void main(String[] args) {
        Flyway.configure()
              .dataSource("jdbc:postgresql://localhost:5432/postgres","postgres","postgres")
              .cleanDisabled(true) // запрет
              .load();
        System.out.println("Clean is disabled.");
    }
}
```

**Kotlin — Liquibase dropAll только в DEV (аналог Java по смыслу):**

```kotlin
package ops.clean

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import java.sql.DriverManager

fun main() {
    val env = System.getenv("APP_ENV") ?: "DEV"
    require(env == "DEV") { "dropAll is forbidden outside DEV" }

    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        val db = DatabaseFactory.getInstance()
            .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
        Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db).dropAll()
        println("Schema dropped in DEV.")
    }
}
```

---

## Ошибки при старте: порядок инициализации, конфликт с Hibernate DDL auto

Spring Boot запускает мигратор до `EntityManagerFactory`, чтобы ORM видел готовую схему. Если оставить `spring.jpa.hibernate.ddl-auto=update/create`, Hibernate начнёт менять схему *параллельно* с миграторами. Это источник гонок и неожиданных различий между средами. В проде этот механизм нужно отключить.

Если у приложения несколько `DataSource`, явно укажите, к какому из них привязан мигратор. Нечёткая привязка приводит к созданию служебных таблиц в «не той» базе/схеме и к падениям на старте.

Конфликты версий драйверов/транзакционных менеджеров проявляются как «загадочные» ошибки при исполнении DDL. Помните: некоторые DDL Postgres не транзакционны (например, `CREATE INDEX CONCURRENTLY`), и попытка «накрыть всё одной транзакцией» обречена.

Инициализационные данные/коллбеки миграций смешивать с бизнес-логикой старта приложения не стоит. Чем меньше завязок «после миграции» живёт в аппе, тем предсказуемее поведение и проще перенос миграций в отдельную job.

Наличие одновременно Flyway и Liquibase в одном сервисе на одном DataSource — опасная идея. Обычно нужен один мигратор на базу/схему. Исключения — редкость и требуют строгой изоляции.

Падения на старте из-за «нет таблицы X» обычно означают, что миграции не отработали: отключены, не найдены локации, или ошибка в параметрах подключения. Логи мигратора — первый источник правды; включите INFO/DEBUG на старте.

Хорошая практика — конфигурировать миграции через профили: в DEV можно включить «на старте», на stage/prod — выносить в отдельный шаг. Это исключает ситуации, когда autoscaling непреднамеренно запускает миграции.

Если включён validation-on-migrate, приложение упадёт при рассинхроне checksums — это правильно. Лучше fail fast, чем «работа поверх неизвестной схемы».

Для Liquibase master-changelog должен быть один, с читаемым include-порядком. Случайные «local-only» файлы — путь к расхождениям между стендами.

Убедитесь, что `search_path`/`defaultSchema` согласованы между мигратором и Hibernate. Иначе ORM будет искать таблицы «не в той» схеме.

**Java — безопасный старт: отключить DDL-auto, включить validate (аналогичный YAML для Kotlin ниже):**

```java
package ops.startup;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }
}
```

```java
# application.yaml (для Java/Kotlin одинаково)
spring:
  jpa:
    hibernate:
      ddl-auto: none
  flyway:
    enabled: true
    validate-on-migrate: true
```

**Kotlin — та же конфигурация старта (аналог Java):**

```kotlin
package ops.startup

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class App
fun main(args: Array<String>) = runApplication<App>(*args)
```

---

## Проблемы схем/прав: `search_path`, `defaultSchema`, права на объекты и последовательности

Большинство «необъяснимых» падений миграций в Postgres — из-за схем и прав. Если `search_path` различается по средам, одно и то же `create index on my_table` будет направлено в разные схемы. Решение — квалифицировать объекты (`schema.table`) и задавать `defaultSchema/schemas`.

Пользователь миграций должен иметь `CREATE/ALTER` на целевые схемы. Приложение — минимальные права на чтение/запись своих объектов. Смешение ролей — частая причина инцидентов «permission denied».

Убедитесь, что последовательности принадлежат колонкам (`OWNED BY`) и выдан `USAGE/SELECT/UPDATE` нужным ролям. Иначе `nextval` неожиданно подпрыгнёт на ошибке доступа.

Liquibase поддерживает `defaultSchemaName` и явное указание `schemaName` в changeSet’ах. Это делает миграции переносимыми между базами, где `search_path` не совпадает.

В мульти-схемных проектах задайте явный список схем в Flyway `.schemas("app","audit")` и `defaultSchema("app")`. Это определит, где создаются служебные таблицы и где искать миграции.

Проверяйте, куда реально попали объекты после миграции: простой запрос к `information_schema`/`pg_class` на этапе smoke-теста ловит «утёкшие» объекты.

Если в DEV у вас `public`, а в PROD — `app`, вынесите имя схемы в плейсхолдеры/параметры мигратора и используйте их в SQL/changeset’ах. Это избавит от дублирования файлов.

Не полагайтесь на «магический» `search_path` из роли — задавайте его в сессии мигратора `SET search_path TO app, public` для детерминизма.

Документируйте матрицу прав: кто создаёт объекты, кто ими владеет, кто ими пользуется. Это ускоряет расследования.

Проходите периодически по правам `REVOKE ALL` для «чужих» ролей и раздавайте «минимально достаточные». Чем меньше прав — тем меньше неожиданностей.

**Java — установка `search_path` и проверка схемы (аналог ниже в Kotlin):**

```java
package ops.schema;

import java.sql.*;

public class EnsureSearchPath {
    public static void main(String[] args) throws Exception {
        try (Connection c = DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres");
             Statement st = c.createStatement()) {
            st.execute("set search_path to app, public");
            st.execute("create table if not exists app.person(id bigint primary key, name text not null)");
            try (ResultSet rs = st.executeQuery("select schemaname, tablename from pg_tables where tablename='person'")) {
                while (rs.next()) System.out.println(rs.getString(1) + "." + rs.getString(2));
            }
        }
    }
}
```

**Kotlin — тот же сценарий (аналог Java):**

```kotlin
package ops.schema

import java.sql.DriverManager

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        conn.createStatement().use { st ->
            st.execute("set search_path to app, public")
            st.execute("create table if not exists app.person(id bigint primary key, name text not null)")
        }
    }
}
```

**Liquibase — явная схема (YAML и XML):**

```yaml
databaseChangeLog:
  - changeSet:
      id: 310
      author: demo
      changes:
        - createTable:
            schemaName: app
            tableName: person
            columns:
              - column: { name: id, type: BIGINT, constraints: { primaryKey: true, nullable: false } }
              - column: { name: name, type: TEXT,   constraints: { nullable: false } }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
    <changeSet id="310" author="demo">
        <createTable schemaName="app" tableName="person">
            <column name="id" type="BIGINT">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="name" type="TEXT">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

---

## Ограничение простоев: `LOCK TIMEOUT/STATEMENT TIMEOUT`, батчинг DML, «онлайн»-DDL

Цель эксплуатации — ограничить простои при миграциях. В Postgres два ключевых предохранителя: `lock_timeout` (сколько ждать блокировку) и `statement_timeout` (сколько выполнять оператор). Настройка их в сессии мигратора позволяет «быстро падать», если «окна» нет.

Крупные DML выполняйте батчами: диапазоны по первичному ключу, временные окна, курсоры на N строк. Между кусками давайте паузы и фиксируйте прогресс. Идемпотентность — обязательна: повторный запуск не должен портить данные.

Для индексов используйте `CREATE INDEX CONCURRENTLY`, а для проверок — `NOT VALID` + последующая `VALIDATE CONSTRAINT`. Это снижает влияние на запись, пусть и растягивает миграцию.

Таймауты можно задать прямо в миграции (`SET LOCAL`) — это защищает от неверных настройках роли/сервера. Для Liquibase — `sql`-шаги в changeSet’ах, для Flyway — обычный SQL-файл.

Для долгих апдейтов добавляйте «сигнальные» SELECT’ы/логгирование прогресса — чтобы на мониторинге было видно, что миграция живёт, а не зависла.

Планируйте окна обслуживания, а большие операции заранее «прогревайте»: отделяйте DDL и долгий DML на разные релизы.

Измеряйте фактическое время на стендах с приближёнными объёмами данных. Это даёт реалистичный бюджет простоя и снимает сюрпризы на проде.

Настраивайте `idle_in_transaction_session_timeout`, чтобы не держать «мертвые» транзакции и долгие блоки.

Контролируйте автovacuum: его работа во время большой миграции способна удлинить операции. Иногда имеет смысл временно подвинуть параметры, но это отдельное согласованное решение.

CD-сторожи: запрещайте опасные команды в проде (например, `DROP` без `context=prod_safe`) и требуйте ручного подтверждения больших DDL.

**Java — таймауты сессионно + батчевое обновление (аналог ниже в Kotlin):**

```java
package ops.timeouts;

import java.sql.*;

public class SessionTimeoutsAndBatches {
    public static void main(String[] args) throws Exception {
        try (Connection c = DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres");
             Statement st = c.createStatement()) {
            st.execute("set lock_timeout = '5s'");
            st.execute("set statement_timeout = '2min'");
            st.execute("create table if not exists metrics(id bigserial primary key, ts timestamp not null, v int)");

            try (PreparedStatement ps = c.prepareStatement("update metrics set v = v + 1 where id between ? and ?")) {
                for (int from = 1; from <= 1_000_000; from += 10_000) {
                    ps.setInt(1, from);
                    ps.setInt(2, from + 9_999);
                    ps.executeUpdate();
                    Thread.sleep(100); // микропаузa между батчами
                }
            }
        }
    }
}
```

**Kotlin — тот же сценарий (аналог Java) и «онлайн»-индекс:**

```kotlin
package ops.timeouts

import java.sql.DriverManager

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        conn.createStatement().use { st ->
            st.execute("set lock_timeout = '5s'")
            st.execute("set statement_timeout = '2min'")
            st.execute("create table if not exists events(id bigserial primary key, ts timestamp not null)")
            st.execute("create index concurrently if not exists idx_events_ts on events(ts)")
        }
    }
}
```

**Liquibase — таймауты и DDL (YAML и XML):**

```yaml
databaseChangeLog:
  - changeSet:
      id: 400-timeouts
      author: ops
      changes:
        - sql: "set lock_timeout = '5s'; set statement_timeout = '2min';"
        - sql: "alter table big_table add column new_flag boolean default false not null;"
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
    <changeSet id="400-timeouts" author="ops">
        <sql>set lock_timeout = '5s'; set statement_timeout = '2min';</sql>
        <sql>alter table big_table add column new_flag boolean default false not null;</sql>
    </changeSet>
</databaseChangeLog>
```

# 9. Паттерны продакшн-миграций и zero-downtime

## Expand-and-contract: добавь колонку → начни писать дублируя → переключи чтение → удали старое

Первый и самый надёжный паттерн для безостановочных миграций — **expand-and-contract**. Его идея в том, чтобы изменения схемы «раскладывать» на совместимые шаги: сначала мы **расширяем** схему (expand) так, чтобы старая версия кода продолжала работать без ошибок, затем включаем новую логику по чтению/записи постепенно, и только после того, как весь трафик устойчиво живёт на новой модели, мы **сжимаем** схему (contract), удаляя старые сущности. Такой подход убирает «жёсткие» разрывы совместимости и уменьшает необходимость в аварийных откатах.

На этапе expand мы избегаем всего, что может ломать старый код. Вместо переименования столбца делаем добавление нового столбца с нужным именем, типом и безопасными значениями по умолчанию. Вместо переноса данных «за один раз» — добавляем триггер/двойную запись или фоновое наполнение, чтобы и старый, и новый формат сосуществовали параллельно. **Ключевая цель**: старые запросы не падают и дают корректный результат.

Далее включается **dual-write**: приложение пишет данные одновременно в старый и новый столбец/таблицу. Для этого чаще всего используется feature-flag на уровне приложения или конфигурации. Важно обеспечить идемпотентность и порядок: запись в оба места должна быть атомарной с точки зрения бизнес-инвариантов, иначе риски рассинхронизации.

После прогрева и выравнивания данных мы меняем **точку чтения**: новый код начинает читать из «нового» источника, при этом ещё какое-то время продолжается двойная запись, чтобы ничего не потерять. На этом шаге хорошо помогают контролируемые флажки: переключили чтение на часть трафика (канареечный запуск), посмотрели на метрики, убедились, что латентность/ошибки в норме — увеличили долю.

Когда уверились, что весь трафик читает из новой структуры и данные консистентны, наступает фаза **contract**. На ней мы выключаем dual-write, останавливаем фоновые выравнивания, удаляем старые столбцы/таблицы, убираем совместимые представления и индексы, оставленные для обратной совместимости. Если используется Liquibase, удобны **теги**: «expand-tag», «switch-read-tag», «contract-tag», — это делает таймлайн изменений прозрачным.

Типовой пример: «переименование столбца». В БД это почти всегда «добавь новый столбец + начни писать дублируя + фонова перенеси исторические данные + переключи чтение + удали старый». Не используйте `ALTER TABLE ... RENAME COLUMN` при живом трафике: ORM/кэш/старые бинарники гарантированно «споткнутся». Паттерн решает проблему совместимости между несколькими версиями приложения и схемы.

Важно заранее договориться про **источник истины** во время dual-write. Если по техническим причинам возможны расхождения между старым и новым столбцом, нужно определить правило разрешения конфликтов (например, «новый столбец приоритетнее после момента T»). Проверки консистентности на уровне отчётности/метрик сильно экономят время расследований.

С точки зрения тестирования к паттерну применимы сценарии из предыдущей подтемы: «старый код на новой схеме» (старые insert/updates работают), «новый код на старой схеме» (через флаги — не пишет в отсутствующие столбцы), «переключение чтения» (тот же результат при чтении из старого и нового источников). Такие тесты — не украшение, а страховка от регрессий в самом болезненном месте.

В плане производительности expand-and-contract даёт дополнительные издержки (двойная запись, фоновая миграция данных), но зато платой за небольшую сложность вы получаете минимизацию простоев и управляемый риск. Для критичных сервисов это стандарт де-факто, а не «overengineering».

Наконец, документируйте шаги и сигналы «готовности»: какой процент данных перенесён, какие алерты должны быть зелёными, какие SQL-скрипты разрешено исполнять в каждой фазе. Без документации легко перепутать порядок действий при ночном релизе.

**Gradle зависимости (оба варианта) для миграционного стека**

```groovy
// build.gradle (Groovy DSL)
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.flywaydb:flyway-core"
    implementation "org.liquibase:liquibase-core"
}
```

```kotlin
// build.gradle.kts (Kotlin DSL)
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.flywaydb:flyway-core")
    implementation("org.liquibase:liquibase-core")
}
```

**Liquibase (YAML): expand шаг (добавление нового столбца и backfill по умолчанию)**

```yaml
databaseChangeLog:
  - changeSet:
      id: 900-expand-add-new-col
      author: dba
      changes:
        - addColumn:
            tableName: customer
            columns:
              - column:
                  name: email_new
                  type: text
                  defaultValue: ''
        - addNotNullConstraint:
            tableName: customer
            columnName: email_new
            columnDataType: text
```

**Liquibase (XML): тот же changeSet**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="900-expand-add-new-col" author="dba">
    <addColumn tableName="customer">
      <column name="email_new" type="text" defaultValue=""/>
    </addColumn>
    <addNotNullConstraint tableName="customer" columnName="email_new" columnDataType="text"/>
  </changeSet>
</databaseChangeLog>
```

**Java (dual-write c feature-flag; аналог — ниже на Kotlin)**

```java
package zero.expand;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CustomerService {
    private final JdbcTemplate jdbc;
    private final boolean dualWrite;

    public CustomerService(JdbcTemplate jdbc, @Value("${feature.dualwrite.email:false}") boolean dualWrite) {
        this.jdbc = jdbc;
        this.dualWrite = dualWrite;
    }

    @Transactional
    public void updateEmail(long id, String email) {
        jdbc.update("update customer set email = ? where id = ?", email, id);
        if (dualWrite) {
            jdbc.update("update customer set email_new = ? where id = ?", email, id);
        }
    }

    public String readEmail(long id) {
        // позже переключим на email_new
        return jdbc.queryForObject("select email from customer where id = ?", String.class, id);
    }
}
```

**Kotlin (dual-write, тот же смысл)**

```kotlin
package zero.expand

import org.springframework.beans.factory.annotation.Value
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class CustomerService(
    private val jdbc: JdbcTemplate,
    @Value("\${feature.dualwrite.email:false}") private val dualWrite: Boolean
) {
    @Transactional
    fun updateEmail(id: Long, email: String) {
        jdbc.update("update customer set email = ? where id = ?", email, id)
        if (dualWrite) {
            jdbc.update("update customer set email_new = ? where id = ?", email, id)
        }
    }

    fun readEmail(id: Long): String =
        jdbc.queryForObject("select email from customer where id = ?", String::class.java, id)!!
}
```

**Liquibase (YAML/XML): contract шаг — удаление старого столбца после переключения чтения**

```yaml
databaseChangeLog:
  - changeSet:
      id: 905-contract-drop-old
      author: dba
      preConditions:
        - onFail: MARK_RAN
          not:
            - columnExists:
                tableName: customer
                columnName: email
      changes:
        - dropColumn:
            tableName: customer
            columnName: email
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="905-contract-drop-old" author="dba">
    <preConditions onFail="MARK_RAN">
      <not>
        <columnExists tableName="customer" columnName="email"/>
      </not>
    </preConditions>
    <dropColumn tableName="customer" columnName="email"/>
  </changeSet>
</databaseChangeLog>
```

---

## Онлайн-DDL: Postgres `CONCURRENTLY` для индексов, «NOT VALID» для констрейнтов, безопасные приёмы

Онлайн-DDL — это практика проведения изменений структуры с минимальными блокировками. В PostgreSQL основная рабочая лошадка — `CREATE INDEX CONCURRENTLY`, который не берёт долгого эксклюзивного блокирования таблицы на запись. Это почти всегда правильный выбор для больших таблиц в продакшне, пусть и занимает больше времени.

Для ограничений есть похожая техника: добавляйте constraint как **`NOT VALID`**, что создаёт ограничение без проверки уже существующих строк. Новые вставки/обновления будут валидироваться, а исторические данные вы проверите позже явной командой `VALIDATE CONSTRAINT`. Это распределяет нагрузку во времени и избегает длинных блокировок.

Не все DDL в Postgres транзакционны. Например, `CREATE INDEX CONCURRENTLY` запрещено исполнять внутри транзакции. Это важно для инструментов: в Flyway такие операции держите в отдельных файлах/скриптах, а в Liquibase используйте `sql`-шаги без обёртки в транзакцию (или включайте `runInTransaction: false` на changeSet).

Онлайн-DDL не отменяет планирования окон. Если таблица очень большая, даже «онлайн» операция будет тяжёлой по IO/CPU и может ударить по репликации. Обычно такие шаги запускают в спокойные часы и включают выделенные метрики/алерты.

Индексы на большие таблицы полезно «подготовить»: временно поднять `maintenance_work_mem`, убедиться, что автovacuum не мешает (или наоборот — не выключен полностью) и что свободного места на дисках достаточно. Провал по дискам во время индексации — частый источник инцидентов.

Если вы создаёте уникальный индекс, подумайте о предварительной чистке данных: `CONCURRENTLY` не спасёт, если есть дубликаты — операция просто упадёт. В таком случае сначала делайте DML очистку, а затем индекс.

Не забывайте про `REINDEX CONCURRENTLY` как способ обслуживать индексы без простоя. Это особенно актуально после больших апгрейдов версий или изменения колляций, где старые индексы становятся менее эффективными.

Для составных индексов и фильтрованных (`WHERE ...`) помните про **целевую выборку** запросов. Онлайн-индексация не компенсирует плохой дизайн; индекс должен соответствовать реальным предикатам и порядку сортировки.

В Liquibase «конкурентные» вещи для PostgreSQL удобнее делать через «сырой» `<sql>`/`<sqlFile>`. Это даёт полный контроль: вы точно увидите, что уходит в драйвер, и не попадёте на нюансы поддержки флагов в абстракциях.

После онлайн-DDL полезно прогнать `ANALYZE` для новых индексов/таблиц и оценить планы ключевых запросов через `EXPLAIN ANALYZE`. Это даст уверенность, что цель (ускорение) достигнута, а не наоборот.

**Liquibase (YAML): онлайн-индекс и отложенная валидация констрейнта**

```yaml
databaseChangeLog:
  - changeSet:
      id: 910-online-idx-and-constraint
      author: dba
      runInTransaction: false
      changes:
        - sql: "create index concurrently if not exists idx_orders_created_at on orders(created_at)"
        - sql: "alter table orders add constraint orders_fk_customer not valid foreign key (customer_id) references customer(id)"
```

**Liquibase (XML): тот же приём**

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="910-online-idx-and-constraint" author="dba" runInTransaction="false">
    <sql>create index concurrently if not exists idx_orders_created_at on orders(created_at)</sql>
    <sql>alter table orders add constraint orders_fk_customer not valid foreign key (customer_id) references customer(id)</sql>
  </changeSet>
</databaseChangeLog>
```

**Java: выполнить CONCURRENTLY и VALIDATE отдельно**

```java
package zero.online;

import java.sql.*;

public class OnlineDdlOps {
    public static void main(String[] args) throws Exception {
        try (Connection c = DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres");
             Statement st = c.createStatement()) {
            st.execute("create index concurrently if not exists idx_orders_created_at on orders(created_at)");
            st.execute("alter table orders add constraint orders_fk_customer not valid foreign key (customer_id) references customer(id)");
            // Позже, в окно:
            st.execute("alter table orders validate constraint orders_fk_customer");
        }
    }
}
```

**Kotlin: то же самое**

```kotlin
package zero.online

import java.sql.DriverManager

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        conn.createStatement().use { st ->
            st.execute("create index concurrently if not exists idx_orders_created_at on orders(created_at)")
            st.execute("alter table orders add constraint orders_fk_customer not valid foreign key (customer_id) references customer(id)")
            st.execute("alter table orders validate constraint orders_fk_customer")
        }
    }
}
```

---

## Миграции с долгим DML: батчи по ключу/времени, контроль прогресса, идемпотентность

Долгий DML (обновления/пересчёты на десятках миллионов строк) — главный источник простоев и риска блокировок. Правильная тактика — разбивать работу на **батчи** и выполнять их порциями с контролем времени, прогресса и таймаутов. Важнейшее требование — **идемпотентность**: повторный запуск не должен портить данные.

Батч можно строить по первичному ключу (диапазоны `id between ...`) или по времени (`created_at between ...`). Второй вариант удобен, если записи распределены «по времени» естественным образом, но требует аккуратности с часовыми поясами и «дырками» в данных. По PK — предсказуемее и проще для параллелизма.

Планируйте размер батча экспериментально. Слишком маленький — высокие оверхеды на переключение; слишком большой — риск таймаутов и длинных блокировок индексов. Часто хорошо работают порции 5–50 тысяч строк с короткой паузой между ними.

Если DML меняет много строк, старайтесь **писать в те же значения**, которые уже стоят, — Postgres тогда сможет оптимизировать (MVCC и HOT-update). Это снижает раздувание таблицы и нагрузку на autovacuum. Также помогает обновление только нужных строк (`where ... and target != new_value`).

Контроль прогресса делайте очевидным: отдельная служебная таблица «миграция/прогресс», или запись «последний обработанный id/timestamp» в параметрах. При падении вы продолжите с нужного места. Логи с интервалом «каждые N батчей» — дешёвый, но полезный способ видеть скорость.

Старайтесь не держать долгих транзакций. Фиксируйте по батчам — это уменьшает длину блокировок и объём неочищенных версий (bloat). Устанавливайте `lock_timeout`/`statement_timeout`, чтобы не «висеть» всю ночь на одном батче.

Иногда эффективно вынести тяжёлый перерасчёт в **временную таблицу** с последующим `INSERT INTO ... ON CONFLICT UPDATE` в основную. Это позволяет планировщику выбирать быстрые планы и разгружает основную таблицу от случайных блокировок.

Для повторного запуска делайте скрипт идемпотентным: батч должен быть корректным, даже если часть данных уже обновлена. Простой приём — `where not processed` или сверка контрольной хеш-колонки.

Не забывайте про индексы и триггеры: массовый DML под триггерами будет дорог. Иногда безопасно временно отключить вторичные индексы/триггеры (если нет живого трафика) — но это отдельное управляемое окно. В zero-downtime-режиме — лучше не трогать.

Мониторьте не только время, но и **latency** ключевых запросов приложения во время миграции. Если растёт — уменьшаем размер батча или откладываем часть работ.

**Liquibase (YAML/XML): PL/pgSQL-функция для батчевого обновления + вызов**

```yaml
databaseChangeLog:
  - changeSet:
      id: 920-long-dml-func
      author: dba
      changes:
        - sql: |
            create or replace function recalc_scores(batch_size int) returns void as $$
            declare
              last_id bigint := 0;
            begin
              loop
                update account a
                   set score = score + 1
                 where id > last_id
                 order by id
                 limit batch_size;
                get diagnostics last_id = last_oid; -- упрощённо; в реале храните прогресс отдельно
                perform pg_sleep(0.1);
                exit when not found;
              end loop;
            end;
            $$ language plpgsql;
        - sql: "select recalc_scores(10000);"
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="920-long-dml-func" author="dba">
    <sql><![CDATA[
      create or replace function recalc_scores(batch_size int) returns void as $$
      declare
        last_id bigint := 0;
      begin
        loop
          update account a
             set score = score + 1
           where id > last_id
           order by id
           limit batch_size;
          perform pg_sleep(0.1);
          exit when not found;
        end loop;
      end;
      $$ language plpgsql;
    ]]></sql>
    <sql>select recalc_scores(10000);</sql>
  </changeSet>
</databaseChangeLog>
```

**Java: батчевый апдейт по PK с таймаутами**

```java
package zero.longdml;

import java.sql.*;
import java.time.Duration;

public class BatchDml {
    public static void main(String[] args) throws Exception {
        try (Connection c = DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres")) {
            try (Statement st = c.createStatement()) {
                st.execute("set lock_timeout = '5s'");
                st.execute("set statement_timeout = '2min'");
            }
            long from = 0;
            final long step = 20_000;
            boolean hasMore = true;
            while (hasMore) {
                long start = System.nanoTime();
                try (PreparedStatement ps = c.prepareStatement(
                        "update big_table set processed = true where id > ? and id <= ? and not processed")) {
                    ps.setLong(1, from);
                    ps.setLong(2, from + step);
                    int updated = ps.executeUpdate();
                    hasMore = updated > 0;
                }
                from += step;
                long ms = Duration.ofNanos(System.nanoTime() - start).toMillis();
                System.out.printf("Batch up to id %d in %d ms%n", from, ms);
                Thread.sleep(100);
            }
        }
    }
}
```

**Kotlin: тот же батч**

```kotlin
package zero.longdml

import java.sql.DriverManager
import java.time.Duration

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        conn.createStatement().use { st ->
            st.execute("set lock_timeout = '5s'")
            st.execute("set statement_timeout = '2min'")
        }
        var from = 0L
        val step = 20_000L
        var hasMore = true
        while (hasMore) {
            val start = System.nanoTime()
            conn.prepareStatement(
                "update big_table set processed = true where id > ? and id <= ? and not processed"
            ).use { ps ->
                ps.setLong(1, from)
                ps.setLong(2, from + step)
                val updated = ps.executeUpdate()
                hasMore = updated > 0
            }
            from += step
            val ms = Duration.ofNanos(System.nanoTime() - start).toMillis()
            println("Batch up to id $from in ${ms}ms")
            Thread.sleep(100)
        }
    }
}
```

---

## Разделение DDL и DML на независимые релизы, «прогрев» данных заранее

В реальности самые безопасные релизы — когда **структурные изменения** (DDL) и **тяжёлые перерасчёты** (DML) разделены по времени. Сначала мы добавляем столбцы/индексы/ограничения, совместимые со старым кодом (expand), а уже в следующем релизе выполняем DML-заполнение и переключение чтения. Это убирает кумулятивный риск длинных операций в одном окне.

Такой подход позволяет запускать DML заранее, **до бизнес-релиза**. Например, вы добавили новый столбец `normalized_phone`, а заполнение реализовали фоновым джобом, который к моменту релиза уже обработал 99% записей. В день релиза остаются крохи, и переключение чтения проходит незаметно для пользователей.

В разнесённых релизах легче диагностировать проблемы. Если после DDL начались ошибки — виновата структура или совместимость ORM; если после DML — алгоритм и нагрузка на IO. Локализация причин ускоряет откат/форвард-фикс.

С точки зрения планирования, такой подход лучше дружит с расписанием «окон». DDL в спокойные часы, DML — по ночам несколькими проходами. Репликация и бэкапы тоже страдают меньше: онлайн-индексы можно растянуть, а батчи DML — дозировать.

Единственная «цена» — увеличение общего времени вывода фичи. Но в обмен на предсказуемый риск это обычно разумный компромисс, особенно в финансовых/критичных системах.

Для CI/CD это означает разные задачи/Job: «schema-update» и «data-backfill». Их проще сериализовать, проще алертить и проще останавливать независимо. В Kubernetes это разные `Job`/`CronJob`, а само приложение раскатывается только после **schema-update**.

С точки зрения кода — удобно изначально писать бизнес-логику так, чтобы она была устойчива к частично заполненным данным: если `normalized_*` ещё пуст, работаем со старым полем. Это вписывается в expand-and-contract с dual-read/dual-write.

Мониторинг разделяйте аналогично: индикаторы «DDL успешно» (новые объекты на месте, индексы валидны) и «DML скоро завершится» (сколько процентов обработано, скорость). Смешивание метрик делает картину мутной.

Заранее договоритесь о «технических долгах», которые останутся после разделения. Например, «старый индекс ещё пару недель держим», «view-совместимости удалим в релизе X». Отдельная заметка в репозитории дисциплинирует.

Не забывайте, что чем больше релизов в цепочке, тем важнее регресс-тесты совместимости. «Новый код на старой схеме» и «старый код на новой схеме» — обязательная пара для каждого шага.

**Liquibase (YAML/XML): DDL-релиз (expand) отдельно от DML-релиза (backfill)**

```yaml
databaseChangeLog:
  - changeSet:
      id: 930-ddl-expand
      author: dba
      changes:
        - addColumn:
            tableName: user_profile
            columns:
              - column: { name: normalized_phone, type: text }
        - createIndex:
            tableName: user_profile
            indexName: idx_user_profile_norm_phone
            columns:
              - column: { name: normalized_phone }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="930-ddl-expand" author="dba">
    <addColumn tableName="user_profile">
      <column name="normalized_phone" type="text"/>
    </addColumn>
    <createIndex tableName="user_profile" indexName="idx_user_profile_norm_phone">
      <column name="normalized_phone"/>
    </createIndex>
  </changeSet>
</databaseChangeLog>
```

**Java/Kotlin: отдельный backfill-джоб (упрощённый пример)**

```java
package zero.split;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class PhoneBackfillJob {
    private final JdbcTemplate jdbc;
    public PhoneBackfillJob(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @Scheduled(fixedDelay = 10_000)
    public void backfill() {
        jdbc.update("""
          update user_profile
             set normalized_phone = regexp_replace(phone, '\\D', '', 'g')
           where normalized_phone is null
           limit 5000
        """);
    }
}
```

```kotlin
package zero.split

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class PhoneBackfillJob(private val jdbc: JdbcTemplate) {
    @Scheduled(fixedDelay = 10_000)
    fun backfill() {
        jdbc.update(
            """
            update user_profile
               set normalized_phone = regexp_replace(phone, '\D', '', 'g')
             where normalized_phone is null
             limit 5000
            """.trimIndent()
        )
    }
}
```

---

## Feature flags/toggles для пошагового включения схемных изменений

Feature-flags — не только про UI, но и про базы. С их помощью мы включаем **dual-write**, затем **dual-read**, затем окончательно переключаемся на новую схему. Флаги дают возможность канареечного запуска: включить на 1% трафика, собрать метрики и только потом раскатывать на всех.

Правильная гранулярность флага — на уровне «операции/модели». Например, `feature.db.customer.email.dual_write` и `feature.db.customer.email.read_new`. Это позволяет независимо включать запись и чтение, корректно проходя через expand-and-contract.

Флаги должны храниться **в надёжном источнике конфигурации** (Spring Cloud Config, Consul, LaunchDarkly и т.п.) и кешироваться в приложении с коротким TTL. Это избавит от дрожания и обеспечит быстрые откаты.

Важно, чтобы флаг не пробивал ваш слой тестов. Добавляйте сценарии, проверяющие обе ветки: «флаг выкл» и «флаг вкл». Иначе можно случайно сломать редкий путь и узнать о проблеме только в проде.

Независимость флагов позволяет катить изменения по доменам. Вы можете вести несколько «схемных» миграций параллельно, без жёсткой сцепки их переключений.

Флаги — тоже технический долг. После contract-фазы их нужно **убирать**. Оставленные флаги усложняют код и создают «ветвление логики», которое никто уже не проверяет.

Запоминайте в логах **значение флагов** при критичных операциях. Это облегчает расследование: «почему в 02:31 запись пошла не туда?» — «потому что read_new был включён только на группу X».

Некоторые изменения требуют «stateful» флагов (например, со списком арендаторов/регионов). Делайте их аккуратно и идемпотентно. Лучше больше маленьких флагов, чем один «глобальный» с кучей исключений.

Всегда держите «kill-switch» — флаг, который быстро возвращает чтение/запись к старой схеме, если что-то пошло не так. Это дешевле, чем откаты релиза.

Не пытайтесь «флагом» лечить DDL. Флаг управляет логикой приложения, а не структурой БД. Схемные изменения должны идти как миграции; флаги лишь обрамляют их безопасный ввод.

**Java: два флага для поэтапного переключения**

```java
package zero.flags;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class EmailRepo {
    private final JdbcTemplate jdbc;
    private final boolean dualWrite;
    private final boolean readNew;

    public EmailRepo(JdbcTemplate jdbc,
                     @Value("${feature.db.customer.email.dual_write:false}") boolean dualWrite,
                     @Value("${feature.db.customer.email.read_new:false}") boolean readNew) {
        this.jdbc = jdbc;
        this.dualWrite = dualWrite;
        this.readNew = readNew;
    }

    public void save(long id, String email) {
        jdbc.update("update customer set email = ? where id = ?", email, id);
        if (dualWrite) jdbc.update("update customer set email_new = ? where id = ?", email, id);
    }

    public String find(long id) {
        String col = readNew ? "email_new" : "email";
        return jdbc.queryForObject("select " + col + " from customer where id = ?", String.class, id);
    }
}
```

**Kotlin: тот же паттерн**

```kotlin
package zero.flags

import org.springframework.beans.factory.annotation.Value
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Repository

@Repository
class EmailRepo(
    private val jdbc: JdbcTemplate,
    @Value("\${feature.db.customer.email.dual_write:false}") private val dualWrite: Boolean,
    @Value("\${feature.db.customer.email.read_new:false}") private val readNew: Boolean
) {
    fun save(id: Long, email: String) {
        jdbc.update("update customer set email = ? where id = ?", email, id)
        if (dualWrite) jdbc.update("update customer set email_new = ? where id = ?", email, id)
    }

    fun find(id: Long): String {
        val col = if (readNew) "email_new" else "email"
        return jdbc.queryForObject("select $col from customer where id = ?", String::class.java, id)!!
    }
}
```

**Liquibase (YAML/XML): view-совместимость как флаг-агностичный слой**

```yaml
databaseChangeLog:
  - changeSet:
      id: 940-compat-view
      author: dba
      changes:
        - createView:
            viewName: customer_view
            replaceIfExists: true
            selectQuery: "select id, coalesce(email_new, email) as email from customer"
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="940-compat-view" author="dba">
    <createView viewName="customer_view" replaceIfExists="true">
      <selectQuery>select id, coalesce(email_new, email) as email from customer</selectQuery>
    </createView>
  </changeSet>
</databaseChangeLog>
```

---

## Совместимость контрактов: view/compat-слои, синхронизация API и БД-версий

Контракт совместимости — это явный слой, который сглаживает различия между «старым» и «новым» представлением данных. В БД таким слоем часто выступают **view** или **compat-таблицы**, которые предоставляют старому коду прежнюю форму данных поверх новой структуры.

Использование view полезно тем, что ORM/старые запросы продолжают работать «как раньше», а под капотом вы можете хранить данные в нормализованном виде. Главное — не делать view слишком тяжёлыми: пересчёты полей в SELECT должны быть разумны для онлайна.

Для записи можно применить **INSTEAD OF** триггеры (в других СУБД) или «write-through» слой в приложении: запросы к compat-API транслируются в новые таблицы. В Postgres чаще делают именно апп-слой для записи, оставляя view для чтения.

Синхронизация версий: полезно хранить «сигнатуру схемы» (tag/версия) и сверять её в приложении. Тогда код может понимать, доступна ли новая колонка/таблица, и выбирать совместимый путь. Это особенно важно в многосервисных системах.

Контрактный слой должен быть **временным**: оставлять его «навсегда» — значит платить латентностью и усложнением поддержки. На этапе contract вы его удаляете вместе со старой схемой.

Документируйте контракт: какие поля «виртуальные», какие вычисляемые, где возможны различия округления/форматов. Это предотвращает баги в интеграциях и отчётности.

Тестируйте совместимость через **snapshot-тесты**: одинаковые входы — одинаковые результаты до и после включения нового слоя. Это особенно удобно для REST-ресурсов, где можно сравнить JSON.

Поддерживайте «версионирование API» (v1/v2) синхронно с изменениями в БД. Если вы меняете семантику, возможно лучше развести эндпоинты, чем пытаться «подстелить соломку» под все варианты.

Не забывайте про **security**: compat-view должен наследовать права доступа и не раскрывать лишнее. В Postgres права на view независимы от прав на базовую таблицу — настраивайте их явно.

Удаляйте compat-слой последним, после успешного перехода всех потребителей. Это требует инвентаризации клиентов и иногда — коммуникации с внешними командами.

**Java/Kotlin: compat-репозиторий, читающий через view**

```java
package zero.compat;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class CustomerCompatRepo {
    private final JdbcTemplate jdbc;
    public CustomerCompatRepo(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    public String findEmail(long id) {
        return jdbc.queryForObject("select email from customer_view where id = ?", String.class, id);
    }
}
```

```kotlin
package zero.compat

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Repository

@Repository
class CustomerCompatRepo(private val jdbc: JdbcTemplate) {
    fun findEmail(id: Long): String =
        jdbc.queryForObject("select email from customer_view where id = ?", String::class.java, id)!!
}
```

---

## Катастрофоустойчивость: point-in-time recovery, бэкапы перед критическими DDL

Zero-downtime не отменяет **disaster recovery**. Перед критичными DDL (особенно destrutive-изменениями) должен быть валидный бэкап и план **point-in-time recovery (PITR)**. Это значит, что вы сможете вернуть кластер к моменту «до изменений», если всё пошло катастрофически плохо.

PITR требует постоянного архивирования WAL и проверки восстановления на стенде. Разовая «проверка сценария восстановления» — обязательна для команды, иначе в день Х вы будете «учиться восстанавливаться» под давлением.

Бэкап — не только про «сделать дамп». Он должен быть **консистентным** и совместимым по версии с вашим продом. В Postgres для больших баз чаще выбирают файловый бэкап уровня кластера + WAL, а не логический `pg_dump`.

Перед операцией убедитесь, что бэкап «свежий», и есть место/трафик на его хранение. Не начинайте длительных DDL, если система в состоянии «без бэкапов» из-за аварийного окна.

Оцените сценарий «fallback без отката схемы»: иногда быстрее сделать **forward-fix** или переключить трафик на «горячую» реплику/предыдущую версию приложения, чем разворачивать PITR. Но для этого нужно заранее иметь холодный план.

Храните **снимок схемы** после миграций (schema-only dump). Это облегчает расследование и облегчает сравнение состояний. Такие артефакты здорово иметь в CI.

Документируйте **RPO/RTO** по миграциям: сколько данных допустимо потерять (RPO) и как быстро нужно вернуться (RTO). Это влияет на выбор окна и объёмы операций в одном релизе.

Для многоарендных систем имеет смысл делать бэкапы **по арендаторам** (логические дампы), если риск связан с конкретной схемой/таблицей. Это ускоряет частичное восстановление.

Тестируйте «чёрные лебеди»: падение в середине операции, частично применённые миграции, залипшие блоки. **Учебные тревоги** дают бесценный опыт без боли продакшна.

PITR — не замена дисциплине миграций, но последняя линия обороны. Политически важно помнить: если что-то может пойти не так, однажды оно не так и пойдёт.

**Liquibase (YAML/XML): «мягкий» тег перед критичным DDL**

```yaml
databaseChangeLog:
  - changeSet:
      id: 950-tag-before-critical
      author: dba
      changes:
        - tagDatabase:
            tag: "pre_critical_2025_10_23"
  - changeSet:
      id: 951-critical-ddl
      author: dba
      changes:
        - sql: "alter table payments alter column amount type numeric(20,2)"
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="950-tag-before-critical" author="dba">
    <tagDatabase tag="pre_critical_2025_10_23"/>
  </changeSet>
  <changeSet id="951-critical-ddl" author="dba">
    <sql>alter table payments alter column amount type numeric(20,2)</sql>
  </changeSet>
</databaseChangeLog>
```

**Java/Kotlin: простой pre-check бэкапа (health-probe перед миграцией)**

```java
package zero.dr;

import java.nio.file.*;

public class BackupHealthCheck {
    public static void main(String[] args) {
        Path walArchive = Path.of("/mnt/pg_wal_archive");
        if (!Files.isDirectory(walArchive)) throw new IllegalStateException("WAL archive missing");
        System.out.println("WAL archive OK, proceed.");
    }
}
```

```kotlin
package zero.dr

import java.nio.file.Files
import java.nio.file.Path

fun main() {
    val walArchive = Path.of("/mnt/pg_wal_archive")
    require(Files.isDirectory(walArchive)) { "WAL archive missing" }
    println("WAL archive OK, proceed.")
}
```

---

## Стратегии отката: Liquibase rollback/tag; для Flyway — forward-fix + emergency scripts

Liquibase предоставляет «первоклассный» откат: вы можете определить `rollback`-блоки, ставить теги (`tag`) и делать `rollbackToTag`. Это удобно, когда «сломали не то» и нужно быстро вернуть **структуру** к прежнему виду. Но помните, что откат **данных** часто невозможен: удалённые строки не вернутся магией, если вы не сохраняли их отдельно.

Для Flyway стратегия обычно **forward-fix**: мы не откатываемся назад, а делаем новую миграцию, исправляющую состояние. Это проще для reasoning и не ломает историю. В экстренных случаях могут быть «emergency scripts», но их нужно сразу же «узаконивать» новой миграцией.

Комбинируйте: если вы на Liquibase, описывайте rollback-блоки там, где это реально и безопасно (создание индекса, создание представления). Для DML-переделок и удаления столбцов лучше ориентироваться на forward-fix.

Теги — мощный инструмент для установления «опорных точек» в релизах. Перед критичным шагом поставьте тег, и у вас будет понятная цель для отката. Поддерживайте дисциплину именования тегов (дата+контекст).

Проверяйте rollback **на стенде** перед релизом. Откатиться на бумаге просто, а в реальности может всплыть зависимость (например, ORM-модель уже не стыкуется со «старой» схемой). Лучше заранее знать ограничения.

Откат — не всегда лучший ответ. Иногда быстрее перевести трафик на предыдущую версию приложения (blue-green/канареечный rollout) и сделать forward-fix. Выбор зависит от того, где ошибка — в схеме или в коде.

Храните рецепты отката в репозитории рядом с миграциями. «Живая» документация важнее ссылок в корпоративном Wiki: когда «горит», никто их не ищет.

После отката всегда запускайте `validate`/`diff` и smoke-тесты. Ложное ощущение «мы вернули всё» очень опасно: мелкие несоответствия потом выстрелят в самый неподходящий момент.

Откат — это тоже изменение. Логи/артефакты, которые вы собираете для обычных миграций, должны собираться и для rollback.

**Liquibase (YAML/XML): changeset с явным rollback**

```yaml
databaseChangeLog:
  - changeSet:
      id: 960-add-view
      author: dba
      changes:
        - createView:
            viewName: v_payments
            selectQuery: "select id, amount from payments"
      rollback:
        - dropView:
            viewName: v_payments
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="960-add-view" author="dba">
    <createView viewName="v_payments">
      <selectQuery>select id, amount from payments</selectQuery>
    </createView>
    <rollback>
      <dropView viewName="v_payments"/>
    </rollback>
  </changeSet>
</databaseChangeLog>
```

**Java/Kotlin: «forward-fix» как отдельный скрипт-патч**

```java
package zero.rollback;

import java.sql.*;

public class ForwardFix {
    public static void main(String[] args) throws Exception {
        try (Connection c = DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres");
             Statement st = c.createStatement()) {
            st.execute("update payments set amount = 0 where amount is null"); // пример патча
        }
    }
}
```

```kotlin
package zero.rollback

import java.sql.DriverManager

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        conn.createStatement().use { it.execute("update payments set amount = 0 where amount is null") }
    }
}
```

---

## Мультисервисные изменения: координация порядка миграций между командами

В микросервисах изменения схемы часто касаются нескольких сервисов. Нужна **оркестрация**: кто первый накатывает expand, кто и когда включает dual-write, кто последним делает contract. Без этого легко получить «разъезд» контрактов.

Практичный инструмент — **совместный календарь миграций** и «манифест изменений» (файл в общем репозитории), где перечислены версии/теги по сервисам и требуемый порядок. В CI можно добавить проверку, что зависимые сервисы находятся в совместимых версиях.

Соглашения должны включать **окна** и **тайм-ауты**. Если один сервис не успевает включить новую логику за N дней, миграция считается зависшей и эскалируется. Это дисциплинирует и предотвращает вечные «переходные» состояния.

Для сложных сценариев помогает «слой совместимости» (view/API-прокси) между сервисами. Он позволяет одному сервису перейти раньше, а второму — позже. Но это временная мера: совместимость не должна стать постоянной.

Логируйте и метрику «версия контрактов» в каждом сервисе. По ней быстро увидите, кто отстаёт и где возможен риск. Это проще, чем читать логи релизов.

Планируйте возможность **параллельной работы** старых и новых полей/эндпоинтов. Это снижает сцепление и даёт командам свободу в выборе даты релиза.

Держите «горячую линию» на время больших координированных изменений. Чат-канал с ответственными инженерами обходит бюрократию и ускоряет реакции на инциденты.

Избегайте «скрытых» зависимостей. Если сервис читает напрямую чужую таблицу — это организационный запах. Лучше формальный API/представление с управляемой эволюцией.

После завершения координированной миграции сделайте **retrospective**: что заняло больше всего времени, какие автоматизации стоит внедрить (например, шаблоны Liquibase/Flyway, генерация диаграмм зависимостей).

Наконец, помните, что мультисервисные изменения — это проект, а не «побочный эффект». Подходите к ним так же серьёзно, как к обычному релизу продукта.

**Liquibase (YAML/XML): label/context для раздельного управления по сервисам**

```yaml
databaseChangeLog:
  - changeSet:
      id: 970-shared-expand
      author: dba
      labels: "service-A,service-B"
      context: "stage,prod"
      changes:
        - addColumn:
            tableName: shared_table
            columns:
              - column: { name: new_col, type: text }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="970-shared-expand" author="dba" labels="service-A,service-B" context="stage,prod">
    <addColumn tableName="shared_table">
      <column name="new_col" type="text"/>
    </addColumn>
  </changeSet>
</databaseChangeLog>
```

**Java/Kotlin: проверка версии контракта соседа (упрощённо через “ping”)**

```java
package zero.multi;

import org.springframework.web.client.RestTemplate;

public class ContractCheck {
    public static void main(String[] args) {
        String v = new RestTemplate().getForObject("http://service-b/internal/contract-version", String.class);
        if (!"v2".equals(v)) throw new IllegalStateException("Service B not ready for v2");
    }
}
```

```kotlin
package zero.multi

import org.springframework.web.client.RestTemplate

fun main() {
    val v = RestTemplate().getForObject("http://service-b/internal/contract-version", String::class.java)
    require(v == "v2") { "Service B not ready for v2" }
}
```

---

## Документация паттернов в репо: шаблоны миграций, чек-листы релиза

Документация — это «педали и тормоза» вашей миграционной машины. Держите в репозитории каталог `db/patterns` с шаблонами: expand-and-contract, онлайн-индекс, батч-DML, rollback-friendly изменения. Каждый шаблон — с примером Liquibase (YAML/XML) и Flyway-SQL, плюс короткий чек-лист.

Чек-лист релиза должен включать: «теги Liquibase до/после», «резервное копирование/PITR», «feature-flags: dual-write/read», «мониторинги/алерты», «план отката», «ответственные и контакты». Копипастить чек-лист в MR — полезный ритуал.

Хорошая практика — **готовые скрипты** для операций: `scripts/release/precheck.sh`, `scripts/release/validate.sh`, `scripts/release/rollback.sh`. Они стандартизируют шаги и уменьшают «творческую свободу» в ночи релиза.

Пишите **post-mortem** после крупных миграций: что прошло хорошо, что нет, какие метрики добавить, какие шаги автоматизировать. Это инвестирует в устойчивость.

Автоматизируйте генерацию артефактов: «schema-dump после миграции», «diff отчёт против эталона», «сводка применённых changeSet’ов». Прикладывайте их к релизу.

Храните «матрицу совместимости» версий схемы и сервисов. Простая таблица «сервис → минимальная версия схемы» закрывает массу вопросов в эксплуатации.

Сделайте обучающие примеры короткими и самодостаточными. Новичкам тяжело читать «боевые» миграции — им нужен «минимальный повторяемый кейс».

Наконец, регулярно **чистите** документацию и шаблоны: устаревшие паттерны и ссылки — это скрытые баги. Раз в квартал ревью документации — хорошая привычка.

Помните: документация — это не отчётность, а **инструмент**. Если ей неудобно пользоваться ночью на релизе — её нужно переделать.

**Пример шаблонов в репо (файловая структура)**

```text
db/
  patterns/
    expand_contract/
      liquibase.yaml
      liquibase.xml
      README.md
    online_index/
      liquibase.yaml
      liquibase.xml
      README.md
    long_dml/
      liquibase.yaml
      liquibase.xml
      README.md
scripts/
  release/
    precheck.sh
    validate.sh
    rollback.sh
```

**Java/Kotlin: утилита печати статуса миграций для отчёта**

```java
package zero.docs;

import org.flywaydb.core.Flyway;

public class PrintFlywayStatus {
    public static void main(String[] args) {
        Flyway f = Flyway.configure()
                .dataSource("jdbc:postgresql://localhost:5432/postgres","postgres","postgres")
                .locations("classpath:db/migration")
                .load();
        f.migrate();
        var info = f.info();
        System.out.println("Current: " + (info.current() == null ? "none" : info.current().getVersion()));
        for (var mi : info.applied()) System.out.println(mi.getVersion() + " " + mi.getDescription());
    }
}
```

```kotlin
package zero.docs

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.resource.ClassLoaderResourceAccessor
import java.sql.DriverManager

fun main() {
    DriverManager.getConnection("jdbc:postgresql://localhost:5432/postgres","postgres","postgres").use { conn ->
        val db = DatabaseFactory.getInstance()
            .findCorrectDatabaseImplementation(liquibase.database.jvm.JdbcConnection(conn))
        val lb = Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db)
        lb.update()
        lb.reportStatus(true, null, System.out)
    }
}
```

# 10. CI/CD, контейнеры и Kubernetes

## Maven/Gradle плагины: задачи `migrate/update`, генерация SQL (`updateSQL`), отчёты

Практика: миграции — это такие же артефакты релиза, как бинарники. Поэтому их нужно уметь запускать из сборочной системы и CI, получать сухие планы (SQL), хранить отчёты. В Java-мире это решают плагины Flyway и Liquibase для Maven/Gradle, плюс небольшие «обёртки» для отчётности.
Важная идея: **разделяйте среды**. В DEV — `migrate/update` на временную БД; в CI — `validate`, `updateSQL` (сухой план) как артефакты; в PROD — строго сериализованный шаг миграции, без «автостарта» приложения.
Flyway прост: `flywayMigrate`, `flywayValidate`, `flywayInfo`, `flywayRepair`. Его сила — предсказуемость и конвенции. Liquibase мощнее по части «диффов», `updateSQL` (генерация SQL без запуска), `rollback` и тегов.
Генерация SQL (`updateSQL`) особенно полезна для ревью DBA: вы прикладываете к MR план DDL/DML, и это снижает риск сюрпризов. В Liquibase есть и `diffChangeLog` (сравнение эталона и цели), но не злоупотребляйте автогенерацией — ручной контроль лучше.
Отчёты. Договоритесь о форматах: текстовый список применённых версий (Flyway `info`) и `status` Liquibase. Сохраняйте их в `build/reports/` как артефакт джобы.
Параметры окружения не шьём в билд: URL/логины — через переменные CI или секреты. На локалке удобны `-PdbUrl=...`/`-Dspring.profiles.active=...`.
Надёжность повышает «двухходовка»: сначала `validate` + `updateSQL` (или `migrate -X` в dry-run), затем уже реальный `update/migrate`. Если шаг «сухого плана» упал — релиз стопорится до исправления.
Логика «один артефакт — много сред» требует **placeholders**: имена схем/индексов и фиче-флаги подставляются на этапе запуска задачи. Это позволяет не ветвить файлы миграций.
В мульти-модульных проектах задавайте несколько `locations` (Flyway) или `includeAll` (Liquibase). Это сохраняет локальность миграций домена и упрощает ревью.
Если релизы идут часто, добавьте агрегирующую задачу `:db:check` (линт + validate + updateSQL) и подключите её в PR-пайплайн. Так вы ловите проблемы до merge.
Финальный штрих — унификация: одинаковые имена задач, одинаковые флаги и выходные файлы в разных репозиториях. Это снижает когнитивную нагрузку при сопровождении.

**Gradle Groovy (Flyway + Liquibase, с `updateSQL`)**

```groovy
plugins {
    id "org.flywaydb.flyway" version "10.17.0"
    id "org.liquibase.gradle" version "2.2.2"
}

ext {
    dbUrl  = System.getenv("DB_URL") ?: "jdbc:postgresql://localhost:5432/postgres"
    dbUser = System.getenv("DB_USER") ?: "postgres"
    dbPass = System.getenv("DB_PASS") ?: "postgres"
}

flyway {
    url = dbUrl
    user = dbUser
    password = dbPass
    locations = ["classpath:db/migration"]
    validateOnMigrate = true
}

liquibase {
    activities {
        main {
            changeLogFile "src/main/resources/db/changelog/master.yaml"
            url dbUrl
            username dbUser
            password dbPass
            logLevel "info"
        }
    }
    runList = "main"
}

tasks.register("liquibaseUpdateSql") {
    dependsOn "update"
    doLast {
        // Файл сгенерированного SQL ищите в build/liquibase/sql (по умолчанию плагина)
    }
}
```

**Gradle Kotlin DSL (аналогично)**

```kotlin
plugins {
    id("org.flywaydb.flyway") version "10.17.0"
    id("org.liquibase.gradle") version "2.2.2"
}

val dbUrl  = System.getenv("DB_URL") ?: "jdbc:postgresql://localhost:5432/postgres"
val dbUser = System.getenv("DB_USER") ?: "postgres"
val dbPass = System.getenv("DB_PASS") ?: "postgres"

flyway {
    url = dbUrl
    user = dbUser
    password = dbPass
    locations = arrayOf("classpath:db/migration")
    validateOnMigrate = true
}

liquibase {
    activities.register("main") {
        this.changeLogFile = "src/main/resources/db/changelog/master.yaml"
        this.url = dbUrl
        this.username = dbUser
        this.password = dbPass
        this.logLevel = "info"
    }
    runList = "main"
}
```

**Java: программный запуск Flyway для CI-шага**

```java
package ci.flyway;

import org.flywaydb.core.Flyway;

public class Migrate {
    public static void main(String[] args) {
        Flyway flyway = Flyway.configure()
            .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
            .locations("classpath:db/migration")
            .validateOnMigrate(true)
            .load();
        flyway.migrate();
    }
}
```

**Kotlin: программный запуск Liquibase (аналог Java)**

```kotlin
package ci.liquibase

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import java.sql.DriverManager

fun main() {
    val url = System.getenv("DB_URL")
    val user = System.getenv("DB_USER")
    val pass = System.getenv("DB_PASS")
    DriverManager.getConnection(url, user, pass).use { conn ->
        val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(conn))
        Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db).update()
    }
}
```

**Liquibase changelog (YAML и XML) для демонстрации `updateSQL`**

```yaml
databaseChangeLog:
  - changeSet:
      id: 1001
      author: ci
      changes:
        - addColumn:
            tableName: orders
            columns:
              - column: { name: created_at, type: timestamp, defaultValueComputed: "now()" }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="1001" author="ci">
    <addColumn tableName="orders">
      <column name="created_at" type="timestamp" defaultValueComputed="now()"/>
    </addColumn>
  </changeSet>
</databaseChangeLog>
```

---

## Docker: слой с миграциями, запуск как отдельного шага/контейнера

Контейнеризация снимает проблему «на моей машине работает». Логика проста: собираем образ приложения и **отдельный** облегчённый образ мигратора, где есть только JRE/JDBC/миграции. В CI такой контейнер запускается первым.
Разделение образов полезно для безопасности: у мигратора — другие права и секреты, а его жизненный цикл не привязан к аппа-поду. Вы снижаете риск «автоскейл запустил сто миграций».
Слой с миграциями держите тонким: `jlink`-JRE, только нужные либы (`flyway-core` или `liquibase-core`), миграционные ресурсы и маленький `Main`. Это ускорит старт.
В проде лучше запускать мигратор **один раз**: отдельный контейнер/джоб, а не sidecar, чтобы не плодить параллелизм.
Параметры БД — через переменные окружения/секреты, а не bake-time. Это делает один и тот же образ переиспользуемым для всех сред.
`docker-compose` годится для локалки/интеграционных стендов: сначала «db», потом «migrator», затем «app», с `depends_on` и healthcheck’ами.
Логи контейнера-мигратора сохраняйте как артефакт (stdout в файл CI). Там ключевые тайминги и шаги — их потом смотрят на ретро.
В образ приложения *не* кладём инструменты миграции, если только не выбрали on-start режим; иначе образ «толстеет», а риски растут.
Важно не забыть TZ/локаль и совместимую версию PgJDBC; несоответствие может дать неожиданные планировочные эффекты.
Останов мигратора — программный `System.exit(0/1)` по результату, чтобы CI корректно понял успех/провал.

**Dockerfile для мигратора (Flyway)**

```dockerfile
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY build/libs/app-migrator-all.jar /app/app-migrator.jar
ENV JAVA_OPTS="-Xms128m -Xmx256m"
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar /app/app-migrator.jar"]
```

**docker-compose для локалки**

```yaml
version: "3.8"
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
  migrator:
    build: .
    environment:
      DB_URL: jdbc:postgresql://db:5432/postgres
      DB_USER: postgres
      DB_PASS: postgres
    depends_on: ["db"]
  app:
    image: myorg/app:latest
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/postgres
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
    depends_on: ["migrator"]
```

**Java: минимальный `Main` для контейнера миграций (Flyway)**

```java
package docker.migrator;

import org.flywaydb.core.Flyway;

public class Main {
    public static void main(String[] args) {
        try {
            Flyway.configure()
                    .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
                    .locations("classpath:db/migration")
                    .validateOnMigrate(true)
                    .load()
                    .migrate();
            System.out.println("Migrations OK");
            System.exit(0);
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
    }
}
```

**Kotlin: аналог для Liquibase-контейнера**

```kotlin
package docker.migrator

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import java.sql.DriverManager

fun main() {
    try {
        val url = System.getenv("DB_URL")
        val user = System.getenv("DB_USER")
        val pass = System.getenv("DB_PASS")
        DriverManager.getConnection(url, user, pass).use { conn ->
            val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(conn))
            Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db).update()
        }
        println("Liquibase OK")
        System.exit(0)
    } catch (t: Throwable) {
        t.printStackTrace()
        System.exit(1)
    }
}
```

---

## Kubernetes: initContainers vs Job/Helm hooks — когда что выбирать

В Kubernetes есть три базовых способа запускать миграции: **initContainer** в Pod приложения, **Job** (единоразовый запуск) и **Helm hooks** (`pre-install`, `pre-upgrade`). Выбор влияет на параллелизм, наблюдаемость и контроль.
`initContainer` прост: миграции выполняются перед стартом основного контейнера, Pod не перейдёт в `Running`, пока они не завершатся. Минусы — масштабирование создаёт много параллелизма, и миграции могут стартовать сразу в нескольких Pod’ах.
`Job` даёт явную единичность: вы создаёте отдельный объект, он выполняется и завершается. С ним легко сериализовать и навесить RetentionPolicy логов. Хорош для «один раз на релиз».
Helm hooks удобны, если релизите через Helm: `pre-upgrade` запускает Job до аппа, а неудача — проваливает релиз. Это «каноничный» порядок деплоя без дополнительной оркестрации.
Где что использовать: для DEV/preview окружений — `initContainer` (удобство важнее строгости). Для stage/prod — `Job`/hooks, чтобы избежать гонок и иметь ясные логи и статусы.
Сетевые политики и секреты задавайте одинаковые, как у аппа, чтобы поведение не отличалось: те же ServiceAccount/RoleBinding.
Добавьте аннотации `linkerd.io/inject: disabled` или аналоги, если sidecar-прокси мешают JDBC-соединению в миграторах.
Не забывайте про backoff и таймауты у Job: большие DDL могут идти долго, но вечные ретраи вредны.
Логи Job — это ваши «отчёты о миграции». Примонтируйте volume для дополнительных файлов (например, `schema.sql` после апдейта).
В Helm-чарте заведите чёткие values для включения/выключения мигратора, чтобы в DEV он мог быть на старте, а в PROD — отдельным Job.
Документируйте, что **перекат** Helm должен идти «модульно»: сначала `helm upgrade --set migrations.run=true --wait`, потом — приложение.

**Пример Job (Flyway)**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrator
          image: myorg/db-migrator:1.0.0
          env:
            - { name: DB_URL,  valueFrom: { secretKeyRef: { name: db-secret, key: url } } }
            - { name: DB_USER, valueFrom: { secretKeyRef: { name: db-secret, key: user } } }
            - { name: DB_PASS, valueFrom: { secretKeyRef: { name: db-secret, key: pass } } }
```

**Helm hook (Liquibase)**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "app.fullname" . }}-liquibase"
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: liquibase
          image: myorg/liquibase:1.0.0
          envFrom: [{ secretRef: { name: {{ .Values.db.secretName }} } }]
```

**Java/Kotlin — не обязательно, но пример «readiness-флага» после Job**

```java
package k8s.ready;

public class PrintReady {
    public static void main(String[] args) { System.out.println("MIGRATIONS_APPLIED=1"); }
}
```

```kotlin
package k8s.ready

fun main() { println("MIGRATIONS_APPLIED=1") }
```

---

## Порядок деплоя: «миграции → приложение», блокировка параллельных rollout’ов

Базовый инвариант zero-downtime: **сначала** миграции (expand), **потом** приложение, затем (опционально) DML-backfill и contract в отдельные окна. Нарушение порядка приводит к «старый код на старой схеме» и падениям.
Параллельные раскатки нужно ограничить. В GitLab/GitHub Actions/ArgoCD используйте «mutex»/lock на среду, чтобы миграции не стартовали из двух пайплайнов сразу.
ArgoCD/Helm легко формализуют порядок: preSync-hook для миграций, затем обычные манифесты Deployment. Если hook упал — релиз не пойдёт дальше.
Для on-start сценариев добавляйте «охрану» в readiness-пробу: пока версия схемы не соответствует ожидаемой — Pod «не готов», и сервис не получает трафик.
Релизы с тяжёлым DDL выносите в отдельные «ночные» окна, а приложение катите днём. Это снижает MTTR при «обычных» релизах кода.
Документируйте: «какая версия схемы нужна бинарнику», и проверяйте её на старте (в HealthIndicator). Это сразу даёт понятную ошибку вместо NPE где-нибудь в DAO.
StatefulSet/DaemonSet не подходят для мигратора — используйте Job/Hook, иначе «каждый Pod» попытается накатывать.
Роллбеки: если деплой кода упал после успешной миграции — возвращайтесь приложением назад (совместимо по expand), а миграции не откатывайте.
Ставьте «флажок среды» после миграций (ConfigMap/annotation), чтобы при повторном запуске пайплайн увидел, что миграции уже применены.
Не забывайте, что «миграции → приложение» не отменяет **валидации**: запускайте `validate/info/status` перед следующим шагом, чтобы точно знать, что база в консистентном состоянии.

**Java: HealthIndicator, проверяющий версию схемы (Flyway)**

```java
package deploy.order;

import org.flywaydb.core.Flyway;
import org.springframework.boot.actuate.health.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SchemaHealth {
    @Bean
    public HealthIndicator schemaVersionIndicator() {
        return () -> {
            Flyway f = Flyway.configure()
                    .dataSource(System.getenv("SPRING_DATASOURCE_URL"),
                                System.getenv("SPRING_DATASOURCE_USERNAME"),
                                System.getenv("SPRING_DATASOURCE_PASSWORD"))
                    .load();
            var info = f.info();
            var current = info.current();
            return (current != null && current.getVersion() != null)
                    ? Health.up().withDetail("schemaVersion", current.getVersion().getVersion()).build()
                    : Health.down().withDetail("schemaVersion", "unknown").build();
        };
    }
}
```

**Kotlin: аналог с Liquibase**

```kotlin
package deploy.order

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.sql.DriverManager

@Configuration
class SchemaHealth {
    @Bean
    fun schemaVersionIndicator(): HealthIndicator = HealthIndicator {
        val url = System.getenv("SPRING_DATASOURCE_URL")
        val user = System.getenv("SPRING_DATASOURCE_USERNAME")
        val pass = System.getenv("SPRING_DATASOURCE_PASSWORD")
        DriverManager.getConnection(url, user, pass).use { conn ->
            val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(conn))
            Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db).validate()
            Health.up().withDetail("liquibase", "validated").build()
        }
    }
}
```

---

## Секреты и сеть: управление cred’ами (K8s Secrets, Vault), readiness мигратора

Секреты в Kubernetes — источник правды для мигратора и приложения. Используйте `Secret` + `envFrom` или Vault-агент, а не bake-time переменные. Это позволяет роту секретов без перестроек образов.
Правильные **роли**: мигратор может иметь повышенные права (CREATE/ALTER), приложение — минимальные (SELECT/INSERT/UPDATE). В k8s это разные Secret/ServiceAccount.
Для Vault удобно сделать sidecar-агент, который пишет креды в tmpfs-файл; мигратор читает их на старте и удаляет после завершения.
Сетевые политики (NetworkPolicy) не должны отличаться между мигратором и приложением, иначе «локально работает, в кластере — нет».
Readiness мигратора прост: завершился кодом 0 — всё хорошо. Но если вы запускаете on-start — добавьте «пробу» с небольшим SQL-пингом, чтобы убедиться в доступности БД.
Обязательно ограничивайте `egress` мигратора до сегмента БД, чтобы утечки/ошибки не пошли дальше.
Секреты ротуйте планово. Хардкод в `application.yml` — антипаттерн. Используйте плейсхолдеры/ENV для всего чувствительного.
Не храните URL БД с паролями в логах. Инструменты миграции обычно не логируют секреты, но «println» в утилите — легко забыть.
Шифрование на уровне диска (PVC) и TLS-соединение с БД — обязательны в проде; убедитесь, что мигратор поддерживает сертификаты/тростор.
Документируйте, где и как выдаются права. Это ускорит расследования «permission denied» ночью.

**K8s Secret + Deployment-фрагмент**

```yaml
apiVersion: v1
kind: Secret
metadata: { name: db-secret }
type: Opaque
stringData:
  url: jdbc:postgresql://postgres:5432/app
  user: app_migrator
  pass: change_me
---
env:
  - name: DB_URL
    valueFrom: { secretKeyRef: { name: db-secret, key: url } }
  - name: DB_USER
    valueFrom: { secretKeyRef: { name: db-secret, key: user } }
  - name: DB_PASS
    valueFrom: { secretKeyRef: { name: db-secret, key: pass } }
```

**Java/Kotlin: короткий «ping» БД перед миграцией**

```java
package net.secrets;

import java.sql.*;

public class Ping {
    public static void main(String[] args) throws Exception {
        try (Connection c = DriverManager.getConnection(
                System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"));
             Statement st = c.createStatement()) {
            st.execute("select 1");
            System.out.println("DB OK");
        }
    }
}
```

```kotlin
package net.secrets

import java.sql.DriverManager

fun main() {
    DriverManager.getConnection(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS")).use {
        it.createStatement().execute("select 1")
        println("DB OK")
    }
}
```

---

## Мониторинг: метрики времени/успеха, логирование DDL, алерты на ошибки

Наблюдаемость — не роскошь. Минимум — счётчик успешных/ошибочных миграций и таймер длительности. Это можно сделать вокруг вызовов `migrate()/update()` и публиковать в Micrometer/Prometheus.
Полезно логировать *каждый* changeSet/файл с началом и концом, чтобы видеть «узкие места». Liquibase даёт слушатели, Flyway — callbacks или просто обёртку на уровне приложения.
Метрики качества: число pending-миграций (в норме — 0), время последнего успешного запуска, разница между «ожидаемой» версией и «текущей».
Алерты: провал Job миграций; таймаут миграций > X минут; наличие «dirty»/failed записи во Flyway; ошибки `validate`.
Логи DDL полезно дублировать в отдельный индекс (ELK/Opensearch): поиск по релизу/версии/тегу ускоряет анализ инцидентов.
Для длинных DML — метрика прогресса (процент или обработанные строки). Даже грубая оценка разряжает поддержку.
Графики индексации (IOPS, latency ключевых запросов) на время миграций показывают, как изменения влияют на пользователей.
Хорошо иметь «синтетический» health-чек, который бегает SELECT/INSERT/UPDATE в таблицы, затронутые миграциями, — сразу после релиза.
Снимайте `schema dump` и прикладывайте к релизному отчёту: это и аудит, и быстрый источник правды для diff.
Не забывайте про «шум»: на DEV выключайте избыточные алерты; в PROD — строгие пороги и уведомления команде on-call.

**Java: Micrometer вокруг Flyway**

```java
package mon.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.flywaydb.core.Flyway;

public class TimedMigrate {
    private final MeterRegistry registry;
    public TimedMigrate(MeterRegistry registry) { this.registry = registry; }

    public void run() {
        Timer.Sample s = Timer.start(registry);
        boolean ok = false;
        try {
            Flyway.configure()
                .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
                .locations("classpath:db/migration").load().migrate();
            ok = true;
        } finally {
            s.stop(Timer.builder("db.migration.duration").tag("result", ok ? "ok" : "fail").register(registry));
            registry.counter("db.migration.total", "result", ok ? "ok" : "fail").increment();
        }
    }
}
```

**Kotlin: Micrometer вокруг Liquibase**

```kotlin
package mon.metrics

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import java.sql.DriverManager

class TimedLiquibase(private val registry: MeterRegistry) {
    fun run() {
        val sample = Timer.start(registry)
        var ok = false
        try {
            val url = System.getenv("DB_URL")
            val user = System.getenv("DB_USER")
            val pass = System.getenv("DB_PASS")
            DriverManager.getConnection(url, user, pass).use { conn ->
                val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(conn))
                Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db).update()
            }
            ok = true
        } finally {
            sample.stop(Timer.builder("db.migration.duration").tag("result", if (ok) "ok" else "fail").register(registry))
            registry.counter("db.migration.total", "result", if (ok) "ok" else "fail").increment()
        }
    }
}
```

---

## Мульти-БД и несколько DataSource в одном сервисе: стратегия последовательного обновления

Если сервис работает с несколькими БД (или схемами), миграции нужно **сериализовать** по источникам: сначала `core`, затем `audit`, и только после успеха обоих — старт приложения.
Разделяйте конфигурацию: по DataSource — отдельные миграторы (и разные `flyway_schema_history` / `databasechangelog`). Нельзя смешивать истории.
Порядок устанавливается бизнес-зависимостями: где модель меняется первой, где нужно ждать. Это фиксируется в CI-скрипте или в коде «миграционного раннера».
Вывод метрик — по каждому источнику отдельно, чтобы видеть «кто тормозит».
В on-start сценариях привяжите мигратор к конкретному DataSource, иначе Spring создаст Flyway для «первого попавшегося».
Для Liquibase используйте несколько `SpringLiquibase` бинов с разными `contexts/labels`, если набор изменений различается.
Храните параметры подключения отдельно (разные Secret), чтобы приложение и мигратор не перепутали креды и схемы.
Smoke-тесты поднимайте против каждого источника. Один зелёный — не гарантия успеха.
Старайтесь сохранять одинаковые паттерны именования версий (например, префиксы `core-`, `audit-`) для читаемости.
В контрактных тестах проверяйте, что приложение «живет» при обновлённой одной БД и старой другой, если релизы могут расходиться во времени.

**application.yml (две БД с двумя миграторами)**

```yaml
spring:
  datasource:
    core: { url: ${CORE_URL}, username: ${CORE_USER}, password: ${CORE_PASS} }
    audit: { url: ${AUDIT_URL}, username: ${AUDIT_USER}, password: ${AUDIT_PASS} }
flyway:
  core:
    locations: classpath:db/migration/core
  audit:
    locations: classpath:db/migration/audit
```

**Java: два Flyway-бина**

```java
package multi.ds;

import org.flywaydb.core.Flyway;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class MultiFlywayConfig {
    @Bean(name = "flywayCore")
    Flyway flywayCore(DataSource coreDs) {
        Flyway f = Flyway.configure().dataSource(coreDs).locations("classpath:db/migration/core").load();
        f.migrate(); return f;
    }
    @Bean(name = "flywayAudit")
    Flyway flywayAudit(DataSource auditDs) {
        Flyway f = Flyway.configure().dataSource(auditDs).locations("classpath:db/migration/audit").load();
        f.migrate(); return f;
    }
}
```

**Kotlin: два SpringLiquibase**

```kotlin
package multi.ds

import liquibase.integration.spring.SpringLiquibase
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.sql.DataSource

@Configuration
class MultiLiquibaseConfig {
    @Bean
    fun liquibaseCore(coreDs: DataSource) = SpringLiquibase().apply {
        dataSource = coreDs
        changeLog = "classpath:db/changelog/core-master.yaml"
        contexts = "core"
    }
    @Bean
    fun liquibaseAudit(auditDs: DataSource) = SpringLiquibase().apply {
        dataSource = auditDs
        changeLog = "classpath:db/changelog/audit-master.yaml"
        contexts = "audit"
    }
}
```

---

## Версионирование релизов и теги схемы: как связывать commit/tag и состояние БД

Хорошая практика — ставить **тег схемы** перед критичными изменениями и после них. В Liquibase это `tagDatabase`, в отчёты Flyway можно писать «текущий git-tag». Так релиз «v1.12.0» связан с состоянием БД.
В CI шаг «миграции» получает `GIT_TAG`/`GIT_SHA` и пишет его в таблицу `schema_meta` или в Liquibase-тег. Это ускоряет расследования: вы сразу видите «какой код на какую схему рассчитывал».
Если релизы ручные — фиксируйте `schema dump` рядом с тегом. Тогда «вспомнить, что было» — открыть артефакт релиза.
Flyway не ставит теги в БД, но вы можете добавить вспомогательную V-миграцию, пишущую в `schema_meta(version, git_tag, applied_at)`.
Liquibase-теги хороши тем, что поддерживают `rollbackToTag`. Но помните: это про структуру, не про данные.
Именуйте теги однообразно: `pre_v1_12_0`, `post_v1_12_0`. Это очевидно для оператора и легко ищется.
На старте приложения проверяйте «ожидаемую минимальную версию схемы», сравнивая с таблицей метаданных. Если отстаёт — не пускайте трафик.
В много-сервисном мире полезна матрица «сервис → минимальный тег схемы». Её можно хранить в конфиг-репозитории.
Поддерживайте утилиту «печать статуса схемы» с тегами — её удобно дергать из CI и runbook’ов.
Для rollback-сценариев заранее убедитесь, что тег «до» проставлен и доступен для `rollbackToTag`.

**Liquibase (YAML/XML): теги**

```yaml
databaseChangeLog:
  - changeSet: { id: tag-pre, author: ci, changes: [ { tagDatabase: { tag: "pre_v1_12_0" } } ] }
  - changeSet: { id: 1100, author: ci, changes: [ { sql: "alter table orders add column currency text" } ] }
  - changeSet: { id: tag-post, author: ci, changes: [ { tagDatabase: { tag: "post_v1_12_0" } } ] }
```

```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
  http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">
  <changeSet id="tag-pre" author="ci"><tagDatabase tag="pre_v1_12_0"/></changeSet>
  <changeSet id="1100" author="ci"><sql>alter table orders add column currency text</sql></changeSet>
  <changeSet id="tag-post" author="ci"><tagDatabase tag="post_v1_12_0"/></changeSet>
</databaseChangeLog>
```

**Java/Kotlin: запись git-тега в служебную таблицу**

```java
package version.tag;

import java.sql.*;

public class RecordTag {
    public static void main(String[] args) throws Exception {
        String tag = System.getenv("GIT_TAG");
        try (Connection c = DriverManager.getConnection(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"));
             Statement st = c.createStatement()) {
            st.execute("create table if not exists schema_meta(tag text, applied_at timestamp default now())");
            try (PreparedStatement ps = c.prepareStatement("insert into schema_meta(tag) values (?)")) {
                ps.setString(1, tag);
                ps.executeUpdate();
            }
        }
    }
}
```

```kotlin
package version.tag

import java.sql.DriverManager

fun main() {
    val tag = System.getenv("GIT_TAG")
    DriverManager.getConnection(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS")).use { conn ->
        conn.createStatement().use { it.execute("create table if not exists schema_meta(tag text, applied_at timestamp default now())") }
        conn.prepareStatement("insert into schema_meta(tag) values (?)").use { ps ->
            ps.setString(1, tag); ps.executeUpdate()
        }
    }
}
```

---

## Автоматические «сторожи»: запрет опасных команд в проде, ручное подтверждение крупных DDL

Сторож — это программный барьер против «опасных» миграций. Примеры: запрет `DROP TABLE` в PROD, запрет `ALTER TABLE DROP COLUMN` без метки `context=contract`, требование `CONCURRENTLY` для индексов на больших таблицах.
Сторож удобно реализовать как отдельную задачу CI, пробегающую по SQL и падающую при нарушении правил. Это быстрый feedback.
Для Liquibase можно анализировать `updateSQL`: он уже разворачивает placeholders и контексты. Вы проверяете именно то, что пойдёт в БД.
Ручное подтверждение: для крупных DDL (долгих индексов/констрейнтов) добавляйте gate-job «Approve DDL», чтобы человек осознанно нажал «OK».
Отключать сторожа можно только через метку MR с явной причиной. Это дисциплина, а не бюрократия.
Анализ должен фильтровать комментарии, чтобы не срабатывать «ложно». Это легко решить регулярками.
Хорошо иметь белый список «системных» схем и объектов, которые сторож игнорирует (например, `flyway_schema_history`).
Сторож должен быть «быстрым»: 1–2 секунды на десятки файлов — нормально. Не перегружайте его сложной логикой.
Отчёт сторожа сохраняйте в артефакты. Его читает и автор, и ревьюер.
Сторож — не замена ревью, но отличный «сигнал тревоги» заранее.

**Java: простой сторож SQL (регулярки)**

```java
package guard.sql;

import java.nio.file.*;
import java.util.regex.Pattern;

public class SqlGuard {
    public static void main(String[] args) throws Exception {
        Path root = Paths.get("src/main/resources/db/migration");
        Pattern[] rules = new Pattern[]{
            Pattern.compile("(?i)\\bdrop\\s+table\\b"),
            Pattern.compile("(?i)\\balter\\s+table\\b.*\\bdrop\\s+column\\b"),
            Pattern.compile("(?i)create\\s+index\\b(?!.*\\bconcurrently\\b).*")
        };
        StringBuilder report = new StringBuilder();
        Files.walk(root).filter(p -> p.toString().endsWith(".sql")).forEach(p -> {
            try {
                String s = Files.readString(p)
                        .replaceAll("(?s)/\\*.*?\\*/", "")
                        .replaceAll("(?m)--.*$", "");
                for (Pattern r : rules) if (r.matcher(s).find()) report.append("Forbidden: ").append(r).append(" in ").append(p).append("\n");
            } catch (Exception e) { report.append("ERR ").append(p).append(": ").append(e.getMessage()).append("\n"); }
        });
        if (report.length() > 0) throw new IllegalStateException("SQL Guard failed:\n" + report);
        System.out.println("SQL Guard OK");
    }
}
```

**Kotlin: тот же сторож**

```kotlin
package guard.sql

import java.nio.file.Files
import java.nio.file.Paths

fun main() {
    val root = Paths.get("src/main/resources/db/migration")
    val rules = listOf(
        Regex("(?i)\\bdrop\\s+table\\b"),
        Regex("(?i)\\balter\\s+table\\b.*\\bdrop\\s+column\\b"),
        Regex("(?i)create\\s+index\\b(?!.*\\bconcurrently\\b).*")
    )
    val report = StringBuilder()
    Files.walk(root).filter { it.toString().endsWith(".sql") }.forEach { p ->
        val s = Files.readString(p).replace(Regex("(?s)/\\*.*?\\*/"), "").replace(Regex("(?m)--.*$"), "")
        for (r in rules) if (r.containsMatchIn(s)) report.append("Forbidden: ${r.pattern} in $p\n")
    }
    require(report.isEmpty()) { "SQL Guard failed:\n$report" }
    println("SQL Guard OK")
}
```

---

## Артефакты пайплайна: архив миграций, `schema dump` после обновления для аудита

CI должен оставлять «бумажный след»: архив миграций, отчёты `validate/info/status`, `schema dump` после обновления. Это ускоряет RCA и аудит.
Архив миграций — это просто zip с `db/migration`/`db/changelog` и отчётами. Его прикладывают к релизу.
`schema dump` (только схема, без данных) фиксирует «как стало». Сравнение дампов между тегами — быстрый diff.
Где хранить: как артефакт джобы и/или в артефакт-репозитории (Nexus/Artifactory/S3).
Не забудьте про секреты: дамп не должен содержать пароли. Используйте `--schema-only`.
Автоматизируйте имена: `schema-v1.12.0.sql` или `schema-<git-sha>.sql`.
Для больших баз дамп можно сжимать (`.sql.gz`). Экономит место без потери пользы.
Полезно приложить текстовый summary: сколько changeSet/версий применено, сколько заняло времени. Это читает не только инженер.
Сформируйте Make/Gradle-таску, чтобы локально можно было легко воспроизвести артефакты.
Храните артефакты столько же, сколько релизы поддерживаются. Затем переносите в «холодное» хранилище или удаляйте по политике.

**Gradle Groovy: zip миграций + Exec для `pg_dump`**

```groovy
tasks.register('zipMigrations', Zip) {
    from('src/main/resources/db') { into 'db' }
    archiveFileName = "migrations-${project.version}.zip"
    destinationDirectory = file("$buildDir/artifacts")
}

tasks.register('schemaDump', Exec) {
    environment 'PGPASSWORD', System.getenv('DB_PASS') ?: 'postgres'
    commandLine 'bash','-lc', "pg_dump --schema-only --no-owner --no-privileges -h ${System.getenv('DB_HOST') ?: 'localhost'} -U ${System.getenv('DB_USER') ?: 'postgres'} -d ${System.getenv('DB_NAME') ?: 'postgres'} > $buildDir/artifacts/schema-${project.version}.sql"
}
```

**Gradle Kotlin DSL: то же**

```kotlin
tasks.register<Zip>("zipMigrations") {
    from("src/main/resources/db") { into("db") }
    archiveFileName.set("migrations-${project.version}.zip")
    destinationDirectory.set(file("$buildDir/artifacts"))
}

tasks.register<Exec>("schemaDump") {
    environment("PGPASSWORD", System.getenv("DB_PASS") ?: "postgres")
    commandLine("bash", "-lc",
        "pg_dump --schema-only --no-owner --no-privileges -h ${System.getenv("DB_HOST") ?: "localhost"} -U ${System.getenv("DB_USER") ?: "postgres"} -d ${System.getenv("DB_NAME") ?: "postgres"} > $buildDir/artifacts/schema-${project.version}.sql")
}
```

**Java/Kotlin: короткий «репорт» после миграции**

```java
package ci.artifacts;

import org.flywaydb.core.Flyway;
import java.nio.file.*;

public class WriteReport {
    public static void main(String[] args) throws Exception {
        Flyway f = Flyway.configure()
                .dataSource(System.getenv("DB_URL"), System.getenv("DB_USER"), System.getenv("DB_PASS"))
                .locations("classpath:db/migration").load();
        f.migrate();
        var info = f.info();
        StringBuilder sb = new StringBuilder("Applied:\n");
        for (var mi : info.applied()) sb.append(mi.getVersion()).append(" ").append(mi.getDescription()).append("\n");
        Files.createDirectories(Path.of("build/reports"));
        Files.writeString(Path.of("build/reports/migration-report.txt"), sb.toString());
    }
}
```

```kotlin
package ci.artifacts

import liquibase.Liquibase
import liquibase.database.DatabaseFactory
import liquibase.database.jvm.JdbcConnection
import liquibase.resource.ClassLoaderResourceAccessor
import java.nio.file.Files
import java.nio.file.Path
import java.sql.DriverManager

fun main() {
    val url = System.getenv("DB_URL"); val user = System.getenv("DB_USER"); val pass = System.getenv("DB_PASS")
    DriverManager.getConnection(url, user, pass).use { conn ->
        val db = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(JdbcConnection(conn))
        val lb = Liquibase("db/changelog/master.yaml", ClassLoaderResourceAccessor(), db)
        lb.update()
        Files.createDirectories(Path.of("build/reports"))
        Files.writeString(Path.of("build/reports/liquibase-status.txt"), "Liquibase update done")
    }
}
```