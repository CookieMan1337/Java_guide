---
layout: page
title: "Миграции: Flyway/Liquibase"
permalink: /spring/migrations_flyway_liquibase
---

# Что это такое и зачем

Миграции БД — это управляемая эволюция схемы и данных через версионные скрипты, хранимые в Git и запускаемые автоматически при старте приложения или отдельной CI/CD-задачей. Идея проста: **не менять базу руками**, а описывать изменения декларативно и воспроизводимо, чтобы любой разработчик и любой стенд могли прийти к одинаковому состоянию.

В экосистеме Java наибольшую популярность получили два инструмента: **Flyway** (акцент на простые SQL-скрипты + минимализм) и **Liquibase** (богатый XML/YAML/JSON формат changelog’ов, `preConditions`, метки/контексты, rollback). Оба умеют работать и с «чистыми» SQL, и с пометками/метаданными для контроля выполнения.

Главная польза миграций — **детерминированность и аудит**. Вы видите в Git, кто и когда добавил таблицу, индекс, триггер или сделал backfill; код ревью покрывает и схему, и данные. В проде это уменьшает человеческий фактор и упрощает откат — вы не «восстанавливаете из памяти», а двигаетесь по зафиксированным шагам.

Минимальный порог входа: добавить зависимость и положить файлы в правильную папку. Ниже — рабочие примеры для обоих инструментов (Gradle Kotlin DSL):

```kotlin
dependencies {
    // ВАРИАНТ 1: Flyway
    implementation("org.flywaydb:flyway-core")
    runtimeOnly("org.postgresql:postgresql")

    // ВАРИАНТ 2: Liquibase (используйте один инструмент за раз)
    // implementation("org.liquibase:liquibase-core")
    // runtimeOnly("org.postgresql:postgresql")
}
```

---

# Общий принцип работы

И Flyway, и Liquibase содержат **метаданные** в служебных таблицах (Flyway: `flyway_schema_history`; Liquibase: `DATABASECHANGELOG` и `DATABASECHANGELOGLOCK`). Там сохраняются версии/идентификаторы изменений, чеки суммы и статус выполнения. Благодаря этому инструмент знает, что уже применено, а что — ещё нет.

Flyway ориентируется на **имя файла** и префиксы: `V1__init.sql`, `V2_1__add_index.sql` (версионные) и `R__update_view.sql` (repeatable). Liquibase читает **changelog** (обычно YAML/XML/JSON) и внутри него — набор `changeset` с уникальной парой `id`+`author`. Порядок задаётся либо версией (Flyway), либо порядком подключения файлов (`include`, Liquibase).

При запуске инструмент сравнивает «набор желаемых изменений» с тем, что записано в метаданных. Новые версии выполняются строго по порядку, при этом считается и **checksum** (защита от тихого редактирования уже применённого скрипта). Если чексумма «не сходится» — выполнение останавливается, чтобы вы явно решили конфликт.

Базовое правило: **не редактируйте** уже применённые миграции — добавляйте новые. История — это история. Исключения бывают (например, опечатка в комменте), но любой «repair» — осознанная операция, чаще — только на локе/тесте, а в проде — с письменным решением команды.

Примеры файлов:

```text
# Flyway (resources/db/migration)
V1__init.sql
V2__add_author_table.sql
R__refresh_views.sql

# Liquibase (resources/db/changelog)
db.changelog-master.yaml
db/changelog/001-init.yaml
db/changelog/002-add-author.yaml
```

---

# Интеграция со Spring Boot

Spring Boot умеет автозапуск миграций при старте приложения. Вы добавляете зависимость, кладёте файлы — и всё «заводится». Важно: **не включайте сразу два инструмента**, выберите один. Переключать удобно профилями.

Пример Flyway-конфигурации:

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: false
    clean-disabled: true
```

Пример Liquibase:

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    contexts: default
    labels: "!experimental"
    drop-first: false
```

Для профилей удобно делить: на `dev` включить Flyway, на `test` — также Flyway + Testcontainers, на `prod` — тот же инструмент, но с `clean-disabled`/`drop-first=false`. Если в проекте исторически сосуществуют оба, **строго** отключайте второй через `enabled:false`, чтобы не дублировать изменения.

Для ручного запуска под CI используйте Gradle-таски: `./gradlew flywayMigrate`, `./gradlew flywayValidate`, `./gradlew liquibaseUpdate`, `./gradlew liquibaseValidate`. Это снимает зависимость от старта Spring и даёт явный шаг «миграции прошли — можно деплоить».

---

# Версии, порядок и контроль изменений

В Flyway порядок — это сортировка по **версии** в имени файла. Принятое соглашение: `V<major>_<minor>__<desc>.sql` (подчёркивания интерпретируются как точки, описание — через двойное подчёркивание). Для «горячих фиксов» иногда добавляют микроверсии: `V2_3_1__fix_index.sql`.

В Liquibase порядок вы задаёте явно через `db.changelog-master.yaml`, подключая под-changelog’и по очереди. У каждого `changeset` есть `id` и `author` — пара должна быть уникальной. Если вы поменяете сам `changeset`, его checksum изменится, и Liquibase попросит вас либо откатить/переиграть, либо сделать `clearCheckSums`/`changelogSync` (осторожно).

Оба инструмента используют checksum для защиты от тихого редактирования. В Flyway есть командa **repair** (исправляет checksum в таблице истории, если вы *осознанно* изменили уже применённый SQL). Но это инструмент **исключений**, а не повседневная практика. В Liquibase аналогично можно «синхронизировать» состояние с реальностью, но нормальный путь — **новый changeset**.

Примеры:

```sql
-- Flyway: V3__add_index.sql
create index if not exists idx_book_title on books using gin (to_tsvector('simple', title));
```

```yaml
# Liquibase: 003-add-index.yaml
databaseChangeLog:
  - changeSet:
      id: 003-add-index
      author: dev
      changes:
        - createIndex:
            indexName: idx_book_title
            tableName: books
            columns:
              - column:
                  name: title
      preConditions:
        - not:
            indexExists:
              tableName: books
              indexName: idx_book_title
```

---

# Repeatable/идемпотентные операции

У Flyway повторяемые миграции — файлы с префиксом `R__`. Они **переисполняются**, когда меняется их содержимое (checksum). Это удобно для представлений, функций и прав, которые логично поддерживать «актуальными». Хорошая практика — держать DDL-представлений в `R__*`.

```sql
-- Flyway: R__refresh_views.sql
create or replace view v_books AS
select b.id, b.title
from books b;
```

В Liquibase есть флаги `runOnChange: true` (перезапускать при изменении) и `runAlways: true` (выполнять всегда). Для идемпотентности используйте `preConditions` (`not tableExists`, `not indexExists`) или «IF NOT EXISTS» в SQL. Это снижает риск падений на стендах с «дрейфом» схемы.

Идемпотентность важна и для DDL, и для DML. Если вы делаете `insert` справочников, добавляйте `on conflict do nothing` (Postgres) или проверку существования. Для Liquibase есть `loadData`/`loadUpdateData` — второй режим умеет upsert по заданным ключам.

Пример идемпотентного SQL:

```sql
-- Flyway: V4__seed_dictionary.sql
insert into region(code, name)
values ('77','Москва')
on conflict (code) do update set name = excluded.name;
```

---

# Запуск на существующей БД: baseline и валидация

Частый кейс: база уже существует, а вы только внедряете инструмент миграций. В Flyway для этого есть **baseline** — отметка «считать начальное состояние применённым». Вы задаёте версию, с которой начать учитывать миграции, и инструмент создаёт запись в `flyway_schema_history`.

Настройка Flyway:

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 100  # ваша "нулевая" версия прод-схемы
```

В Liquibase аналог — **changelogSync**: вы говорите «отметить все changeset’ы как выполненные, не выполняя их». В Gradle/Maven есть соответствующие таски/голы. После sync вы продолжаете жить «от текущего состояния».

Валидация — обязательный шаг в CI. Для Flyway — `flywayValidate`, для Liquibase — `liquibaseValidate`. Это ловит несоответствия checksum, пропуски версий, проблемы с доступностью файлов. Не пускайте деплой без успешной валидации: так вы не внезапно «сломаете» прод миграцией из соседней ветки.

---

# Out-of-order, блокировки и конкуренция

С параллельными ветками возникает ситуация «поздняя версия накатана раньше». Flyway по умолчанию запрещает **out-of-order**, но может его включить: он применит пропущенные «старые» версии позже, если их ещё нет в истории. Это удобно при Git-флоу с частыми релизными ветками, но требует дисциплины в конфликтующих DDL.

```yaml
spring:
  flyway:
    out-of-order: true   # включать осознанно!
```

Liquibase сам определяет порядок через changelog, но на уровне конкурентного запуска использует **DATABASECHANGELOGLOCK**. Таблица-лок защищает от одновременного применения миграций несколькими инстансами. Если процесс умер и «забыл снять замок», используйте `liquibase releaseLocks` (или аналогичный метод API).

В Flyway конкуренция решается транзакциями вокруг `flyway_schema_history`: при одновременном старте миграции «побеждает» один процесс, второй получит конфликт/повтор. Обычно достаточно запускать миграции в **одном** сервисе/джобе (pre-deploy step), а не надеяться на «кто успел — тот и применил».

Золотое правило: миграции — часть **релизного** процесса. Вы либо выносите их в отдельную джобу перед деплоем приложения, либо назначаете один «лидер-под» (init-container), который мигрирует схему, пока остальные ждут.

---

# Безопасность и «чистка»

Никогда не включайте «чистку» схемы на проде. В Flyway: `clean-disabled: true` (по умолчанию в Boot) и **не используйте** `flyway.clean` вне локалки. В Liquibase: `drop-first: false` и внимательность с командами `dropAll`. Любая «чистка» — только на ephemeral-стендах.

Разграничьте роли: отдельный пользователь для **миграций** (DDL/DML расширенные права) и отдельный — для **приложения** (минимально необходимые права). Это уменьшает blast-radius и исключает неожиданную DDL-активность от приложения. Пример выдачи прав:

```sql
-- создаём роли
create role app_migrator login password '***';
create role app_runtime login password '***';

-- права
grant connect on database app to app_migrator, app_runtime;
grant usage on schema public to app_migrator, app_runtime;
grant create on schema public to app_migrator;            -- только миграциям
grant select, insert, update, delete on all tables in schema public to app_runtime;
```

Следите за секретами: не храните пароли к БД в Git, используйте Vault/K8s Secrets. В CI маршрутизируйте права так, чтобы «джоба миграций» имела DDL, а «джоба приложения» — только DML. И обязательно включайте backup-стратегию на проде — миграции уменьшают риски, но не отменяют их.

Наконец, включайте аудит миграций: логи запуска, кто применил, на какой версии, сколько времени заняло. В Flyway/Liquibase это видно из метатаблиц, но не лишним будет отдельный лог в CI.

---

# Zero-downtime (expand/contract)

Безостановочная эволюция схемы требует стратегии **expand/contract**: сначала расширяем схему, чтобы новый код работал и со старой, и с новой, затем, когда все сервисы обновлены, сужаем (удаляем старое). Это двушаговый (иногда трёхшаговый) процесс.

Пример: переименовать колонку `full_name` → `display_name` без даунтайма. Шаг 1 (**expand**): добавляем новую колонку, делаем триггер/двойную запись или бэкофилл.

```sql
-- V10__expand_display_name.sql
alter table users add column display_name text;
update users set display_name = full_name where display_name is null;
create or replace function users_sync_display_name() returns trigger as $$
begin
  new.display_name := new.full_name;
  return new;
end; $$ language plpgsql;

drop trigger if exists trg_users_name_sync on users;
create trigger trg_users_name_sync before insert or update on users
for each row execute function users_sync_display_name();
```

Шаг 2: деплоим приложение, которое читает `display_name` и **пишет обе** колонки (или хотя бы новую). Убеждаемся, что все инстансы на новом коде и данные консистентны.

Шаг 3 (**contract**): удаляем старую колонку и синхронизирующие объекты.

```sql
-- V11__contract_drop_full_name.sql
drop trigger if exists trg_users_name_sync on users;
drop function if exists users_sync_display_name();
alter table users drop column full_name;
```

Важно: в zero-downtime запретите блокирующие операции в «горячий» период (`ALTER TABLE ... TYPE` без `USING` может блокировать), используйте **пошаговые** миграции и, при необходимости, онлайн-индексацию/конкурентные индексы (`create index concurrently` в Postgres).

---

# Тестирование миграций

Лучший способ — **интеграционные тесты** с реальным Postgres через Testcontainers. Поднимаем контейнер, запускаем миграции, проверяем схему/данные. Это ловит проблемы совместимости SQL/диалекта, которых не видно на H2.

Пример с Flyway:

```java
@Testcontainers
class FlywayMigrationsIT {

  @Container
  static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

  @Test
  void migrate_and_verify_schema() {
    var dataSource = DataSourceBuilder.create()
        .url(pg.getJdbcUrl()).username(pg.getUsername()).password(pg.getPassword())
        .build();

    Flyway flyway = Flyway.configure()
        .dataSource(dataSource)
        .locations("classpath:db/migration")
        .load();

    flyway.migrate();

    try (var con = dataSource.getConnection();
         var rs = con.createStatement().executeQuery("select count(*) from books")) {
      assertTrue(rs.next());
    }
  }
}
```

Пример с Liquibase в Spring Boot:

```java
@SpringBootTest
@Testcontainers
class LiquibaseMigrationsIT {

  @Container
  static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
    r.add("spring.liquibase.change-log", () -> "classpath:db/changelog/db.changelog-master.yaml");
  }

  @Autowired DataSource ds;

  @Test
  void schema_is_ready() throws Exception {
    try (var c = ds.getConnection();
         var rs = c.createStatement().executeQuery("select 1 from books limit 1")) {
      // если таблица есть — всё ок
    }
  }
}
```

В CI заведите отдельные шаги: `flywayValidate/migrate` или `liquibaseValidate/updateSQL` (генерация SQL без применения). Это делает миграции «первоклассным гражданином» пайплайна, а не побочным эффектом старта приложения.

---

# Организация репозитория и соглашения

Папки и имена — половина успеха. Для Flyway используйте `src/main/resources/db/migration` и имена `V<ver>__<desc>.sql` (латиница, `snake_case` в описании). Для Liquibase — `src/main/resources/db/changelog/db.changelog-master.yaml` и подкаталоги `db/changelog/xxx-<topic>.yaml`. Один MR/PR — одна миграция (или пакет логически связанных).

Разделяйте **DDL** и **DML** по файлам и шагам. DDL влияет на блокировки/жизненный цикл, DML — на данные и время выполнения. Крупные backfill’ы делайте пакетно и в ночные окна, либо через фоновые джобы приложения. Ничего «тяжёлого» не пихайте в `@PostConstruct`.

Зафиксируйте соглашения в `CONTRIBUTING.md`: как именовать файлы, когда использовать `R__`, какие индексы обязательны для FK, как писать rollback (Liquibase), как проверять планы (`EXPLAIN ANALYZE`). Это уменьшит «зоопарк» стилей между командами.

Наконец, подключите линтеры/проверки. Для Liquibase есть формат-валидаторы, для SQL можно добавить `sqlfluff` в pre-commit или хотя бы `psql -f` проверку синтаксиса в CI. Любая автоматическая проверка экономит часы ревью.

---

## Мини-справочник: быстрые шаблоны

**Flyway, базовая инициализация:**

```sql
-- V1__init.sql
create table if not exists books (
  id bigint primary key generated by default as identity,
  title text not null
);
```

**Liquibase, базовый changeset:**

```yaml
# 001-init.yaml
databaseChangeLog:
  - changeSet:
      id: 001-init
      author: you
      changes:
        - createTable:
            tableName: books
            columns:
              - column: { name: id, type: BIGINT, constraints: { primaryKey: true, nullable: false } }
              - column: { name: title, type: TEXT, constraints: { nullable: false } }
```

**Spring Boot: переключение инструментов профилем:**

```yaml
# application.yml
spring:
  flyway.enabled: false
  liquibase.enabled: false
---
spring:
  config.activate.on-profile: flyway
  flyway.enabled: true
---
spring:
  config.activate.on-profile: liquibase
  liquibase.enabled: true
```

---

# Итог

Миграции — фундамент продуктивной и безопасной разработки. Flyway хорош, когда нужны «прямые» SQL и минимализм управления; Liquibase — когда важны декларативные changelog’и, `preConditions`, метки и контролируемые rollback’и. Оба прекрасно интегрируются со Spring Boot и CI.

Держите простые правила: **одна задача — одна миграция**, не редактируйте применённые скрипты, не включайте «чистку» в проде, разделяйте DDL и тяжёлый DML, используйте **expand/contract** для zero-downtime, валидируйте миграции в CI и тестируйте их на реальном Postgres через Testcontainers. Тогда эволюция схемы станет такой же предсказуемой и приятной, как обычный код-ревью.
