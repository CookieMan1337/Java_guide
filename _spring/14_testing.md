---
layout: page
title: "Тестирование Spring Boot приложения"
permalink: /spring/testing
---
# 0. Введение

## Что это

Тестирование Spring Boot-приложения — это систематическая проверка того, что ваш код делает ровно то, что вы задумали, в условиях, максимально близких к реальным. Мы говорим и о быстрых юнит-проверках чистых классов, и о интеграционных тестах с поднятым контекстом, и о проверках взаимодействий с БД, брокерами сообщений и внешними HTTP-сервисами.

В экосистеме Spring понятие «тест» шире простого JUnit-кейса. Это совокупность изолированных сценариев, запускаемых под управлением JUnit 5, которые используют инфраструктуру Spring Boot Test: автоконфигурацию контекста, срезы (`@DataJpaTest`, `@WebMvcTest`), утилиты безопасности и клиентские DSL для HTTP.

Spring Boot упрощает тестирование через детерминированный ApplicationContext, профили конфигурации и удобные фичи вроде `@TestPropertySource` и динамических свойств. Разработчик фокусируется на намерении сценария, а «склейку» окружения берут на себя готовые тестовые стартеры.

В тестах мы стремимся разделять уровни проверки по пирамиде: юниты, компонентные/срезовые проверки, интеграционные и e2e. Такое разделение ускоряет обратную связь и снижает стоимость поддержки набора тестов в долгую.

Для Spring-приложений важна поддержка инфраструктуры: БД, Kafka, Redis, внешние HTTP-партнёры. Здесь в игру вступают Testcontainers и WireMock — инструменты, делающие окружение воспроизводимым с одной машины на другую.

Под «тестированием» мы также подразумеваем качество контрактов: соответствие API спецификации OpenAPI, предсказуемые ошибки (RFC7807) и соблюдение политики совместимости DTO и событий. Это часть инженерной культуры, а не «опция».

Важная характеристика теста — детерминизм. Любой сценарий должен всегда проходить или всегда падать на одинаковом коде. Никакой зависимости от текущей даты, случайных значений или удалённых сервисов без явных фикстур.

Тест — это ещё и документация живого кода. Хорошее имя, чистая структура Given–When–Then, примеры входов и выходов помогают новым разработчикам быстрее понять намерение и ограничения модуля.

В Spring-мире тесты — первый потребитель ваших автоконфигураций и бинов. Они вскрывают ошибочные условия и циклические зависимости раньше, чем это заметят пользователи в продакшне.

Тесты встраиваются в SDLC: локальный запуск перед коммитом, запуск в CI на каждом PR, более тяжёлые наборы на nightly и релизных ветках, а также smoke-проверки после выкладки. Это «страховочная сетка» для вашей скорости изменений.

**Gradle зависимости для базового тестирования (Groovy/Kotlin DSL)**
Groovy:

```groovy
dependencies {
    testImplementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testRuntimeOnly "org.junit.platform:junit-platform-launcher"
}
```

Kotlin:

```kotlin
dependencies {
    testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

**Java — минимальный юнит-тест и простая логика без Spring**

```java
package intro.basic;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class Slugify {
    String toSlug(String s) {
        return s.trim().toLowerCase().replaceAll("\\s+", "-");
    }
}

public class SlugifyTest {

    @Test
    void shouldMakeLowercaseDashSeparatedSlug() {
        Slugify s = new Slugify();
        assertThat(s.toSlug(" Hello  Spring Boot ")).isEqualTo("hello-spring-boot");
    }
}
```

**Kotlin — эквивалент юнит-теста**

```kotlin
package intro.basic

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class Slugify {
    fun toSlug(s: String) = s.trim().lowercase().replace("\\s+".toRegex(), "-")
}

class SlugifyTest {
    @Test
    fun shouldMakeLowercaseDashSeparatedSlug() {
        val s = Slugify()
        assertThat(s.toSlug(" Hello  Spring Boot ")).isEqualTo("hello-spring-boot")
    }
}
```

## Зачем это

Тесты возвращают контроль над сложностью. По мере роста системы ручная регрессия становится невозможной, а автоматические проверки фиксируют поведение и защищают от случайных поломок при рефакторинге и добавлении фич.

Тесты ускоряют разработку. Парадоксально, но чем больше качественных быстрых тестов на базовом уровне, тем быстрее команда поставляет изменения: меньше «откатов», меньше ручной проверки, больше уверенности в мёрджах.

Тесты улучшают дизайн. TDD/BDD стимулируют написание модулей с чёткими границами, чистыми зависимостями и явными контрактами. Непротестируемый код почти всегда сигналит о чрезмерной связности и «магии».

Тесты снижают стоимость дефектов. Ошибка, пойманная в PR, на порядок дешевле, чем найденная на стейдже, и на два порядка дешевле продакшн-инцидента. Это прямые деньги, SLA и репутация.

Тесты — это живой справочник. Новичок по тестам быстро понимает, что «нормально», что «ошибка», какие ограничения и крайние случаи вы учитывали при проектировании.

Тесты повышают безопасность изменений инфраструктуры. Обновление Spring Boot, Hibernate или драйверов БД проверяется в автоматике; критичные регрессы видны до релиза.

Тесты помогают избегать дрейфа контрактов. Проверки на соответствие OpenAPI и единый формат ошибок не дают коду «уплывать» от обещанного внешнему миру.

Тесты — это инвестиция в скорость CI/CD. Параллельные, изолированные и быстрые наборы позволяют деплоить часто, мелкими порциями, а не «вагонами» с непредсказуемым эффектом.

Тесты — инструмент командной коммуникации. Ожидания, зафиксированные в сценариях, снимают двусмысленность требований и делают «готовность» фичи проверяемой.

Тесты — противоядие флейки-поведения. Явные фикстуры, контроль времени и случайности, детерминированные контейнеры убирают «мистику» из сборок.

**Java — маленький «бизнес-тест» как спецификация требуемого поведения**

```java
package intro.why;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class DiscountService {
    int apply(int price) {
        if (price >= 10_000) return Math.round(price * 0.9f);
        return price;
    }
}

public class DiscountServiceTest {
    @Test
    void ordersFrom10kHaveTenPercentDiscount() {
        DiscountService s = new DiscountService();
        assertThat(s.apply(10_000)).isEqualTo(9_000);
        assertThat(s.apply(9_999)).isEqualTo(9_999);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package intro.why

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import kotlin.math.round

class DiscountService {
    fun apply(price: Int): Int = if (price >= 10_000) round(price * 0.9f).toInt() else price
}

class DiscountServiceTest {
    @Test
    fun ordersFrom10kHaveTenPercentDiscount() {
        val s = DiscountService()
        assertThat(s.apply(10_000)).isEqualTo(9_000)
        assertThat(s.apply(9_999)).isEqualTo(9_999)
    }
}
```

**Gradle — добавим AssertJ (Groovy/Kotlin DSL)**
Groovy:

```groovy
dependencies { testImplementation "org.assertj:assertj-core:3.26.3" }
```

Kotlin:

```kotlin
dependencies { testImplementation("org.assertj:assertj-core:3.26.3") }
```

## Где используется

Тесты работают на всех стадиях SDLC. На локальной машине разработчик гоняет быстрые юниты и срезы, чтобы получить мгновенную обратную связь, не поднимая весь контекст.

В CI на каждый PR запускается полный набор компонентных и интеграционных проверок. Это «шлагбаум» качества: мёрдж возможен только при зелёном статусе.

Ночные сборки и релизные ветки гоняют тяжёлые сценарии: Testcontainers с реальной БД, Kafka, внешние контракты и миграции схем. Это моделирует продакшн ближе всех.

После деплоя в CD полезны smoke-тесты. Это автоматические запросы к ключевым эндпоинтам, проверяющие «самое важное живо» и формат ошибок на месте.

Тесты применяются и для платформенных изменений: обновления Java, Spring Boot, Hibernate, драйверов, конфигураций TLS. Регресс берут на себя интеграционные сценарии.

В проектировании контракта тесты помогают валидировать OpenAPI и «выковать» единый формат ошибок, чтобы UI и интеграторы не тонули в неожиданных расхождениях.

Тесты используются для обучения. Примеры из тестов попадают в документацию и становятся «каноническими» способами вызова API и трактовки ошибок.

В отладке инцидентов тесты играют роль «репродуктора». На основе ошибки пишется красный сценарий, фикс и «позеленение» защищают от повторного появления дефекта.

В рефакторинге тесты — страховка. Смелость менять архитектуру появляется, когда поведение зафиксировано десятками независимых проверок.

В миграциях инфраструктуры тесты — индикатор готовности. Переезд с HikariCP на другой пул, смена параметров Postgres или Kafka конфигов валидируется автоматически.

**Java — интеграционный тест с Postgres Testcontainers и Spring Data JPA**

```java
package intro.where;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.junit.jupiter.Container;

import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@DataJpaTest
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired UserRepository repo;

    @Test
    void saveAndFind() {
        var u = repo.save(new User(null, "alice"));
        assertThat(repo.findById(u.id)).isPresent();
    }

    @Entity(name = "users")
    static class User {
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
        String name;
        public User() {}
        public User(Long id, String name) { this.id = id; this.name = name; }
    }

    interface UserRepository extends JpaRepository<User, Long> { }
}
```

**Kotlin — эквивалент**

```kotlin
package intro.where

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import org.springframework.data.jpa.repository.JpaRepository

@Testcontainers
@DataJpaTest
class UserRepositoryTest {

    companion object {
        @Container
        @JvmStatic
        val pg = PostgreSQLContainer("postgres:16-alpine")

        @JvmStatic
        @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("spring.datasource.url", pg::getJdbcUrl)
            r.add("spring.datasource.username", pg::getUsername)
            r.add("spring.datasource.password", pg::getPassword)
        }
    }

    @Autowired lateinit var repo: UserRepository

    @Test
    fun saveAndFind() {
        val u = repo.save(User(name = "alice"))
        assertThat(repo.findById(u.id!!)).isPresent
    }
}

@Entity(name = "users")
data class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = ""
)

interface UserRepository : JpaRepository<User, Long>
```

**Gradle — Testcontainers (Groovy/Kotlin DSL)**
Groovy:

```groovy
dependencies {
    testImplementation "org.testcontainers:junit-jupiter:1.20.2"
    testImplementation "org.testcontainers:postgresql:1.20.2"
}
```

Kotlin:

```kotlin
dependencies {
    testImplementation("org.testcontainers:junit-jupiter:1.20.2")
    testImplementation("org.testcontainers:postgresql:1.20.2")
}
```

## Какие проблемы/задачи решает

Тесты ловят регресс. Любое изменение может иметь нелокальные последствия; автоматические проверки фиксируют ожидания и не дают «сломать старое», когда добавляете новое.

Тесты защищают от неявной зависимости на время и случайность. Контролируем `Clock`, подменяем генераторы UUID и рандома, делаем сценарии воспроизводимыми.

Тесты документируют и стабилизируют формат ошибок. Единый `application/problem+json` с кодами и полями проверяется так же строго, как успешные сценарии.

Тесты делают интеграции предсказуемыми. WireMock/MockServer позволяют моделировать задержки, тайм-ауты, сбои и убедиться, что ваш клиент корректно ретраит и обрабатывает DLQ.

Тесты помогают держать безопасность на уровне. Spring Security Test проверяет роли, scopes, CSRF и методную безопасность без ручных плясок в UI.

Тесты помогают с производительностью базовых операций. Даже микро-бенчмарки на уровне метода показывают деградации до того, как они проявятся в профайле продакшна.

Тесты контролируют сериализацию и совместимость DTO. Проверки на добавление полей без ломания клиентов предотвращают «невидимые» изменения контракта.

Тесты снижают риски инфраструктурных миграций. Переезд на новую версию Postgres, Kafka или обновление драйверов не пугает, когда есть изолированные интеграционные наборы.

Тесты структурируют командную ответственность. Ясно видно, какая подсистема «красная» и кто её владелец, а значит, кто принимает решение о блокировке релиза.

Тесты повышают доверие к релизам. Когда зелёная сборка означает «покрыты все важные сценарии», менеджерам проще принимать решение о выкладке чаще и меньшими порциями.

**Java — проверка единого формата ошибок (RFC7807) через MockMvc**

```java
package intro.problems;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.ProblemDetail;
import jakarta.validation.constraints.Min;
import org.springframework.validation.annotation.Validated;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.MethodArgumentNotValidException;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
@Validated
class PriceController {
    @GetMapping("/price")
    int price(@RequestParam @Min(1) int qty) { return qty * 10; }
}

@RestControllerAdvice
class ErrorHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail badReq(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation failed");
        return pd;
    }
}

@WebMvcTest(controllers = {PriceController.class, ErrorHandler.class})
class ErrorFormatTest {

    @Autowired MockMvc mvc;

    @Test
    void returnsProblemJsonOnValidationError() throws Exception {
        mvc.perform(get("/price").param("qty", "0"))
           .andExpect(status().isBadRequest())
           .andExpect(header().string("Content-Type", "application/problem+json"))
           .andExpect(jsonPath("$.title").value("Validation failed"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package intro.problems

import jakarta.validation.constraints.Min
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.*
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.beans.factory.annotation.Autowired
import org.junit.jupiter.api.Test
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get

@RestController
@Validated
class PriceController {
    @GetMapping("/price")
    fun price(@RequestParam @Min(1) qty: Int) = qty * 10
}

@RestControllerAdvice
class ErrorHandler {
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun badReq(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply { title = "Validation failed" }
}

@WebMvcTest(controllers = [PriceController::class, ErrorHandler::class])
class ErrorFormatTest(@Autowired val mvc: MockMvc) {

    @Test
    fun returnsProblemJsonOnValidationError() {
        mvc.get("/price") { param("qty", "0") }
            .andExpect { status { isBadRequest() } }
            .andExpect { header { string("Content-Type", "application/problem+json") } }
            .andExpect { jsonPath("$.title") { value("Validation failed") } }
    }
}
```

**Gradle — добавим валидацию и JSON-утилиты (Groovy/Kotlin DSL)**
Groovy:

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-validation"
    testImplementation "org.skyscreamer:jsonassert:1.5.3"
}
```

Kotlin:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
    testImplementation("org.skyscreamer:jsonassert:1.5.3")
}
```

# 1. Основы тестирования Spring Boot

## Цели и пирамида тестирования (unit → component → integration → e2e), ROI и скорость обратной связи

Пирамида тестирования задаёт пропорции набора: **много быстрых unit**, меньше компонентных/срезовых, ещё меньше интеграционных, единицы e2e. Идея проста: чем ниже уровень, тем быстрее обратная связь и дешевле сопровождение; чем выше — тем дороже, но ближе к реальности.

Цель пирамиды — максимизировать **ROI** тестов. Юниты стоят дёшево и дают мгновенную сигнализацию о дефектах в бизнес-логике. Интеграционные и e2e дороже, но проверяют «стыки»: конфигурацию Spring, БД, безопасность, сети.

Скорость обратной связи — метрика, которую стоит измерять. Для юнитов целимся в миллисекунды; для компонентных — десятки миллисекунд; для интеграционных — секунды; для e2e — минуты. Любой тест, выходящий за ожидания, требует аудита.

Пирамида не запрещает «шайбы» (например, «песочные часы», где много контрактных тестов). Но любую добавленную «дорогую» проверку надо обосновать: что она ловит, чего не ловят низкие уровни, и почему её нельзя заменить срезом.

Юнит-тесты в Spring-проектах не должны поднимать контекст. Это тесты чистых классов, функций и небольших коллабораций с подставными зависимостями. Они дают структуру и храбрость рефакторить.

Компонентные/срезовые тесты («slice tests») поднимают **минимум** Spring: веб-слой, JPA, сериализацию. Они ловят конфигурационные ошибки и нюансы биндинга/валидации без цены полного контекста.

Интеграционные тесты поднимают **весь** контекст и реальные инфраструктурные зависимости (через Testcontainers/WireMock). Их задача — «склейка» модулей и проверка, что конфигурация соответствует продакшну.

E2e — это проверка сквозных пользовательских сценариев. Они дороги, нестабильны и должны быть «тонкими»: одно поведение — один сценарий, без матрёшек и условностей.

Пирамида — не догма. В доменах с насыщенными контрактами (платёжные шлюзы) слой контрактных тестов будет толще. В аналитике — больше тестов данных и производительности. Но базовый принцип «дешёвое покрывает много, дорогое — точечно» остаётся.

Для поддержания пропорции в CI удобно разводить наборы тегами/задачами: `fast` на каждом PR, `medium` на ветки, `slow` nightly. Так вы сохраняете баланс скорости и уверенности.

**Gradle (Groovy/Kotlin) — разделим наборы по тегам JUnit 5**

```groovy
tasks.register('testFast', Test) {
    useJUnitPlatform { includeTags 'unit', 'component' }
}
tasks.register('testSlow', Test) {
    useJUnitPlatform { includeTags 'integration', 'e2e' }
}
```

```kotlin
tasks.register<Test>("testFast") {
    useJUnitPlatform { includeTags("unit", "component") }
}
tasks.register<Test>("testSlow") {
    useJUnitPlatform { includeTags("integration", "e2e") }
}
```

**Java — маркируем уровни через @Tag**

```java
package pyramid;

import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class PriceCalc {
    int addVat(int net) { return Math.round(net * 1.2f); }
}

public class PriceCalcTest {

    @Test @Tag("unit")
    void unit_addsVat() {
        assertThat(new PriceCalc().addVat(100)).isEqualTo(120);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package pyramid

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Tag
import org.junit.jupiter.api.Test
import kotlin.math.round

class PriceCalc { fun addVat(net: Int) = round(net * 1.2f).toInt() }

class PriceCalcTest {
    @Test @Tag("unit")
    fun unit_addsVat() {
        assertThat(PriceCalc().addVat(100)).isEqualTo(120)
    }
}
```

---

## Подходы: TDD/BDD/ATDD, где уместны и как влияют на дизайн

**TDD (Test-Driven Development)** — цикл «красный → зелёный → рефакторинг». Мы пишем тест, видим падение, пишем минимальную реализацию, делаем зелёным, затем улучшаем дизайн. Эффект: код становится разделённым, зависимости — явными, интерфейсы — минимальными.

**BDD (Behavior-Driven Development)** формулирует поведение в терминах домена и примеров. Даже если вы не используете Gherkin, структура Given–When–Then и «язык предметной области» делают тесты понятнее бизнесу и разработчикам других команд.

**ATDD (Acceptance Test-Driven Development)** — сначала пишем «приёмочные» тесты на уровень системы (или контракта), потом делаем их зелёными. Это оправдано, когда договоренности с потребителем критичны: API для внешних партнёров, критичные SLA.

TDD уместен в доменной логике, алгоритмах, валидации — там, где легко выделить чистые объекты и функции. Он хуже работает в конфигурации инфраструктуры (например, конфиг Kafka), где разумнее опереться на интеграционные проверки.

BDD полезен в сервисах с богатым доменом и правилами. Он приносит ценность, когда вы пишете тесты как «рассказы» о поведении, а не как набор технических asserts.

ATDD помогает согласовать ожидания между командой API и клиентом. Часто он реализуется как контрактные тесты (Pact/Spring Cloud Contract) или как «золотые файлы» с примерами.

Практика: не абсолютизируйте подходы. TDD — инструмент, а не религия. Есть задачи, где «сначала тест» замедляет, например, быстрые исследования прототипа.

Но влияние на дизайн — реальное. Когда вы начинаете с теста, ваш код оказывается **тестопригодным**: без статических синглтонов, со стабильными зависимостями и точками расширения.

Особенно хорошо TDD проявляется в разборе граничных условий: вы чётко выписываете кейсы, которые склонны «уплывать» в продакшне.

И наконец, все три подхода выигрывают от автоматизации в CI: тесты — артефакты, а не «локальные эксперименты». Они ревьюятся, версионируются и становятся частью культуры.

**Java — маленький TDD-цикл: тест → реализация**

```java
package tdd;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SlugService {
    String make(String title) { // минимальная реализация
        return title.trim().toLowerCase().replaceAll("\\s+", "-");
    }
}

public class SlugServiceTest {
    @Test
    void trimsLowercasesAndJoinsByDash() {
        var s = new SlugService();
        assertThat(s.make("  Hello  Spring  ")).isEqualTo("hello-spring");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package tdd

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class SlugService { fun make(title: String) = title.trim().lowercase().replace("\\s+".toRegex(), "-") }

class SlugServiceTest {
    @Test
    fun trimsLowercasesAndJoinsByDash() {
        val s = SlugService()
        assertThat(s.make("  Hello  Spring  ")).isEqualTo("hello-spring")
    }
}
```

**Gradle (Groovy/Kotlin) — включаем JUnit 5 и AssertJ**

```groovy
dependencies {
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testImplementation "org.assertj:assertj-core:3.26.3"
}
```

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.assertj:assertj-core:3.26.3")
}
```

---

## Правила читабельности: Given–When–Then, AAA, naming, один сценарий — один тест

Читабельность — это скорость поддержки. Тест читают чаще, чем пишут, поэтому «рассказ» о поведении важнее, чем техническая эквилибристика. Формат Given–When–Then (или AAA: Arrange–Act–Assert) дисциплинирует структуру.

Имена тестов должны описывать **намерение**, а не реализацию: `shouldRejectExpiredToken()` лучше, чем `test1()`. Чем короче и яснее — тем проще читать отчёты CI.

«Один сценарий — один тест» уменьшает хрупкость. Когда в тесте несколько «веток», любой будущий фикс превращает этот тест в минное поле из случайных падений.

Фикстуры должны быть понятными. Вынесите сложную инициализацию в билдеры/материнские объекты (`Builder/Mother`), но не прячьте в них поведение, которое важно для конкретного чтения.

Избегайте логики в утверждениях: `assertThat(list).hasSize(2).containsExactly(...)` лучше, чем `assertTrue(...)` с самодельной логикой. Чем больше семантики в ассерт-библиотеке — тем короче и понятнее тест.

Пишите **примеры**. Один позитивный, один негативный — уже задают рамки. Больше — только если сценарии действительно независимы и важны.

Хорошая практика — выровнять форматирование Given/When/Then комментариями. Это якоря для глаз. В больших тестах добавляйте пустые строки между блоками.

Стабильность читаемости — про локализацию. Указывайте явно локали/таймзоны, если от них зависит сериализация/валидация; иначе тесты «плавают» на разных машинах.

Не «коллекционируйте» asserts. Лучше короткие точные проверки, чем длинные «простыни» из всего подряд. Каждое утверждение — реальное требование.

Ревьюйте тесты так же строго, как и код. Тест — боевой артефакт. Он должен быть чистым, коротким и отлаженным.

**Java — AAA/GWT приём со стабильными именами**

```java
package readability;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TokenValidator {
    boolean isExpired(long nowEpochSec, long expEpochSec) { return nowEpochSec >= expEpochSec; }
}

public class TokenValidatorTest {

    @Test
    void shouldRejectExpiredToken() {
        // Given
        TokenValidator v = new TokenValidator();
        long now = 2000L;
        long exp = 1000L;

        // When
        boolean expired = v.isExpired(now, exp);

        // Then
        assertThat(expired).isTrue();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package readability

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class TokenValidator { fun isExpired(nowEpochSec: Long, expEpochSec: Long) = nowEpochSec >= expEpochSec }

class TokenValidatorTest {

    @Test
    fun shouldRejectExpiredToken() {
        // Given
        val v = TokenValidator()
        val now = 2000L
        val exp = 1000L
        // When
        val expired = v.isExpired(now, exp)
        // Then
        assertThat(expired).isTrue()
    }
}
```

---

## Изоляция и детерминизм: фикстуры, тестовые данные, контроль времени/случайности

Детерминизм — основа надёжного набора. Тесты не должны зависеть от текущего времени, случайностей, сети и «состояния соседей». Всё внешнее фиксируем и контролируем.

Время контролируем через `Clock`. В проде — `Clock.systemUTC()`, в тестах — `Clock.fixed(...)`. Это снимает плавающие падения 23:59/00:00 и проблемы часовых зон.

UUID/случайность — через абстракции (`IdGenerator`, `RandomSupplier`). В тестах подсовываем фиксированный источник. Так логи и снапшоты стабильны, а «золотые файлы» не изменяются бессмысленно.

Фикстуры данных делайте **локальными**: никакого общего внешнего состояния между тестами. Если нужно — билдьте объекты внутри каждого теста или используйте тонкие билдеры.

Избегайте «магических» тестовых данных. Подписывайте значения комментарием, используйте константы с говорящими именами, чтобы читатель понимал, зачем именно эти числа.

Асинхронность — через **явные ожидания**. Не `Thread.sleep`, а Awaitility/пулы с таймаутами; в юнитах — синхронные исполнители (TestTaskExecutor).

IO и файлы — в `@TempDir`/временных каталогах. Случайная запись в рабочий каталог делает набор хрупким и плохо воспроизводимым на CI.

Сетевые вызовы — мокайте. Даже если внешний сервис «рядом», тесты не должны его требовать для зелёного статуса. Реальная интеграция — отдельный слой с WireMock/Testcontainers.

Поведение с ретраями/таймаутами проверяйте через детерминированные «тумблеры»: счётчики попыток, фиктивные часы, заглушки latency.

Если приходится использовать Testcontainers на уровне юнитов — это уже не юниты. Переместите тест в слой интеграционных или выделите срез.

Все эти правила — не жесткость, а страховка. Детерминизм — это про скорость команды, а не «дотошность ради дотошности».

**Java — контроль времени и ID**

```java
package determinism;

import java.time.*;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TimeService {
    private final Clock clock;
    private final Ids ids;
    TimeService(Clock clock, Ids ids) { this.clock = clock; this.ids = ids; }
    String stamp() { return ids.next() + "@" + OffsetDateTime.now(clock).toString(); }
    interface Ids { String next(); }
}

public class TimeServiceTest {
    @Test
    void deterministicStamp() {
        Clock fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
        TimeService.Ids ids = () -> "00000000-0000-0000-0000-000000000000";
        var svc = new TimeService(fixed, ids);
        assertThat(svc.stamp()).isEqualTo("00000000-0000-0000-0000-000000000000@2025-01-01T00:00Z");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package determinism

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.Clock
import java.time.Instant
import java.time.OffsetDateTime
import java.time.ZoneOffset

class TimeService(private val clock: Clock, private val ids: Ids) {
    fun stamp(): String = "${ids.next()}@${OffsetDateTime.now(clock)}"
    fun interface Ids { fun next(): String }
}

class TimeServiceTest {
    @Test
    fun deterministicStamp() {
        val fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC)
        val ids = TimeService.Ids { "00000000-0000-0000-0000-000000000000" }
        val svc = TimeService(fixed, ids)
        assertThat(svc.stamp()).isEqualTo("00000000-0000-0000-0000-000000000000@2025-01-01T00:00Z")
    }
}
```

**Gradle (Groovy/Kotlin) — вспомогательные библиотеки**

```groovy
dependencies { testImplementation "org.assertj:assertj-core:3.26.3" }
```

```kotlin
dependencies { testImplementation("org.assertj:assertj-core:3.26.3") }
```

---

## Антипаттерны: «unit на @SpringBootTest», хрупкие e2e, shared state, часовые зоны/локали

Главный антипаттерн — **юнит-тесты на полном контексте** (`@SpringBootTest`). Они медленные, нестабильные и не дают лучшей обратной связи. Юнитам не нужен контейнер DI; им нужны простые конструкторы и подставные зависимости.

Второй — «хрупкие e2e»: сценарии, зависящие от внешних сред, очередности запуска, глобального состояния. Они часто «падают просто так» и подрывают доверие к тестам.

Третий — **shared state** между тестами: статические синглтоны, глобальные кэши, файлы в репозитории, общие контейнеры без изоляции. Любая утечка состояния превращает зелёную сборку в лотерею.

Четвёртый — «волшебное время»: тесты, зависящие от локали и таймзоны машины. Они могут стабильно падать на CI в другом регионе или ночью, когда меняется дата.

Пятый — «многословные ассерты»: страницы проверок, где каждое изменение формата приводит к десяткам фиксов. Сфокусируйте проверки на том, что важно для контракта, а не на каждом пробеле.

Шестой — «sleep вместо ожиданий». Слепые задержки делают тесты медленными и неустойчивыми. Используйте условия/таймауты и синхронные исполняющие абстракции в юнитах.

Седьмой — «магические фикстуры». Когда билдеры скрывают поведение, читатель теряется. Фикстуры должны быть явными, а билдеры — тонкими.

Восьмой — «тесты-конструкторы», где тесты собирают сложную систему каждую итерацию. Делегируйте это Spring-срезам или вынесите инфраструктурные части в базовые конфиги.

Девятый — «проверяем не то»: тест на внутреннюю реализацию вместо внешнего поведения ломается при безвредном рефакторинге. Закрепляйте **контракт**, а не алгоритм.

Десятый — «тесты без изоляции окружения»: общий порт, общий каталог данных, общая БД. Любая параллельность разрушает такие тесты. Изолируйте — и они станут предсказуемыми.

**Java — антипаттерн и исправление**

```java
// ❌ ПЛОХО: юнит на полном контексте
//@SpringBootTest
//class BadUnitTest { /* медленно и бессмысленно для чистого класса */ }

package antipatterns;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(MockitoExtension.class)
class GoodUnitTest {

    @Test
    void fastAndIsolated() {
        var service = new PureService();
        assertThat(service.doubleIt(2)).isEqualTo(4);
    }

    static class PureService { int doubleIt(int x){ return x*2; } }
}
```

**Kotlin — эквивалент**

```kotlin
package antipatterns

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

// ❌ ПЛОХО: @SpringBootTest для чистой логики — медленно и лишне

class GoodUnitTest {

    @Test
    fun fastAndIsolated() {
        val service = PureService()
        assertThat(service.doubleIt(2)).isEqualTo(4)
    }

    class PureService { fun doubleIt(x: Int) = x * 2 }
}
```

**Gradle (Groovy/Kotlin) — включим Mockito только там, где нужен**

```groovy
dependencies { testImplementation "org.mockito:mockito-junit-jupiter:5.14.2" }
```

```kotlin
dependencies { testImplementation("org.mockito:mockito-junit-jupiter:5.14.2") }
```

---

## Где заканчивается «проверка кода» и начинается «проверка контракта/системы»

Проверка кода отвечает на вопрос: «модуль ведёт себя так, как задумано?» Это юниты и срезы. Проверка контракта/системы — «наш сервис говорит с миром так, как обещал?» Это проверки API/сообщений, схем, статусов, заголовков, безопасности.

Граница проходит там, где появляется **внешний интерфейс**: HTTP-контракт, событие Kafka, SQL-схема. С этого места тесты должны смотреть на публичный контракт, а не на внутренние детали.

Контрактные тесты фиксируют **формат** и **правила**: коды ошибок, `Content-Type`, обязательные поля, примеры. Они устойчивы к рефакторингу реализации и служат доверительным интерфейсом между командами.

Системные тесты добавляют окружение и поведение «между» сервисами: сети, тайм-ауты, ретраи, консистентность. Они нужны, чтобы увидеть реальную картину без подгонки моками.

Практика для HTTP: используйте MockMvc/WebTestClient как «локальный клиент» и проверяйте, что выдаётся ровно то, что заявлено в OpenAPI: статус, медиа-тип, тело. Для «честности» можно валидировать ответ по JSON Schema.

Для событий: публикуйте в тестовую тему Testcontainers-Kafka и валидируйте схему (Avro/JSON Schema) и семантику ключей/partitioning. Это защитит эволюцию событий от ломания потребителей.

Для БД: проверяйте миграции «до HEAD» и схему против эталона. Это контракт между приложением и БД, не менее важный, чем HTTP.

Условный критерий: если падение теста требует смотреть в **протокол/контракт**, вы уже выше уровня кода. Держите такие проверки тонкими, но точными.

Не путайте слои: отсутствие поля в JSON — это контракт; неправильная сумма — это код. Первый ловится на контрактном уровне, второй — на юнитах.

**Java — контрактная проверка ответа по JSON-схеме (MockMvc)**

```java
package contract;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.everit.json.schema.loader.SchemaLoader;
import org.everit.json.schema.Schema;
import org.json.JSONObject;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
class HelloController {
    @GetMapping(value="/api/hello", produces="application/json")
    public Greeting hello() { return new Greeting("hi"); }
    record Greeting(String message){}
}

@WebMvcTest(controllers = HelloController.class)
class ContractTest {

    @Autowired MockMvc mvc;
    ObjectMapper om = new ObjectMapper();
    static final String SCHEMA = """
    { "$schema":"http://json-schema.org/draft-07/schema#",
      "type":"object","required":["message"],
      "properties":{"message":{"type":"string"}}}
    """;

    @Test
    void matchesJsonSchema() throws Exception {
        var body = mvc.perform(get("/api/hello"))
            .andExpect(status().isOk())
            .andExpect(header().string("Content-Type","application/json"))
            .andReturn().getResponse().getContentAsString();

        Schema schema = SchemaLoader.load(new JSONObject(SCHEMA));
        schema.validate(new JSONObject(body)); // бросит ValidationException при несоответствии
    }
}
```

**Kotlin — эквивалент**

```kotlin
package contract

import com.fasterxml.jackson.databind.ObjectMapper
import org.everit.json.schema.Schema
import org.everit.json.schema.loader.SchemaLoader
import org.json.JSONObject
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import javax.annotation.Resource

@RestController
class HelloController {
    data class Greeting(val message: String)
    @GetMapping("/api/hello", produces = ["application/json"])
    fun hello() = Greeting("hi")
}

@WebMvcTest(controllers = [HelloController::class])
class ContractTest {

    @Resource lateinit var mvc: MockMvc
    private val schemaText = """
        { "$"$"schema":"http://json-schema.org/draft-07/schema#",
          "type":"object","required":["message"],
          "properties":{"message":{"type":"string"}}}
    """.trimIndent()

    @Test
    fun matchesJsonSchema() {
        val body = mvc.get("/api/hello")
            .andExpect { status { isOk() } }
            .andExpect { header { string("Content-Type","application/json") } }
            .andReturn().response.contentAsString

        val schema: Schema = SchemaLoader.load(JSONObject(schemaText))
        schema.validate(JSONObject(body))
    }
}
```

**Gradle (Groovy/Kotlin) — зависимости для JSON Schema**

```groovy
dependencies { testImplementation "org.everit.json:org.everit.json.schema:1.14.4" }
```

```kotlin
dependencies { testImplementation("org.everit.json:org.everit.json.schema:1.14.4") }
```

# 2. Стек и инфраструктура тестов

## JUnit 5 (Jupiter), AssertJ/Hamcrest, параметризованные тесты, временные каталоги

JUnit 5 (Jupiter) — де-факто стандарт запуска тестов для Spring Boot. Он приносит современную модель жизненного цикла, расширения (`Extension`), теги, параметризацию и удобные аннотации `@BeforeEach/@AfterEach`, `@Nested`, `@Tag`. В экосистеме Spring Boot 3.x Jupiter включён транзитивно через `spring-boot-starter-test`, а потому никаких отдельных раннеров добавлять не требуется. Важно помнить, что JUnit 4 и 5 можно смешивать лишь при наличии Vintage-движка, но в новых проектах это не рекомендуется из-за усложнения конфигурации и отчётности.

AssertJ — «выразительная» библиотека утверждений с fluent-DSL, богатым набором матчеров и человекочитаемыми сообщениями об ошибках. В сравнении с Hamcrest, AssertJ обычно короче и легче читается: выражения строятся естественно («`assertThat(list).containsExactly(...)`»), а не через статические «матчеры от матчеров». Тем не менее Hamcrest по-прежнему полезен там, где он уже применяется или где нужен специфический матчер (например, для сложных проверок строковых паттернов). В одном проекте можно использовать обе библиотеки, но для единообразия лучше выбрать одну.

Параметризованные тесты JUnit 5 (`@ParameterizedTest`) экономят код и повышают охват граничных случаев. Вместо множества однотипных методов вы описываете источник данных (`@ValueSource`, `@CsvSource`, `@MethodSource`) и получаете один тест, исполняемый для каждого набора параметров. Это особенно удобно для валидации, преобразований и чистых функций, где комбинации входов легко перечислить или сгенерировать.

Временные каталоги и файлы важны для детерминизма. Jupiter предоставляет `@TempDir`, который создаёт уникальный каталог для каждого теста/класса и гарантированно чистит его по завершении. Это снимает риск коллизий при параллельном запуске и избавляет от «залипших» артефактов в рабочем дереве проекта. Работая с временным диском, старайтесь не делать предположений о разрешениях и пути — используйте предоставленные `Path`/`File`.

Структура теста должна быть очевидной: Given–When–Then или AAA. В Jupiter нет жёсткой привязки к стилю, но его легко реализовать комментариями и вспомогательными методами. Старайтесь избегать логики в секции «Assert»: чем декларативнее утверждения, тем проще понимать причину падения. AssertJ в этом помогает за счёт «компонентных» утверждений (`extracting`, `satisfies`).

Теги (`@Tag`) позволяют разбивать набор на «fast», «component», «integration», «e2e». Это основа для быстрых и медленных задач Gradle и для разных пайплайнов в CI. Разумно приучить команду локально запускать «fast», а «slow» доверить CI с nightly-расписанием — так вы сохраните скорость обратной связи и не пропустите регрессы.

Именование тестов — часть UX. JUnit 5 поддерживает `@DisplayName`, но зачастую достаточно понятных имён методов в стиле «should…when…». В Kotlin это особенно читаемо благодаря возможности использовать пробелы в `DisplayName`. Придерживайтесь единого стиля во всём репозитории, чтобы отчёты были однородны.

Отчётность в IDE и CI улучшается за счёт использования «мягких» ассертов (`SoftAssertions` в AssertJ) там, где важно увидеть сразу несколько расхождений. Но не злоупотребляйте: один тест — один сценарий, и лучше короткий, чем «лист на экран». Мягкие ассерты логичны в тестах сериализации и DTO, когда нужно показать сразу несколько несовпадений полей.

Параметризация по `@MethodSource` хороша для генерации сложных структур. Выносите источник в статический/companion метод, чтобы не бегали лишние зависимости. И обязательно помечайте кейсы говорящими именами через `Arguments.of(named(...))`, чтобы в отчёте было видно, какая именно ветка упала.

Наконец, помните о временной зоне и локали. Если тесты зависят от форматирования дат/чисел или правил сравнения строк, устанавливайте `Locale.ROOT` и `ZoneOffset.UTC` в фикстурах. Это избавит вас от «плавающих» падений на разных машинах разработчиков и на CI.

**Gradle (Groovy)**

```groovy
dependencies {
    testImplementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    testImplementation "org.springframework.boot:spring-boot-starter-test" // JUnit 5, AssertJ
    testRuntimeOnly "org.junit.platform:junit-platform-launcher"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

**Java — параметризованный тест и @TempDir**

```java
package stack.junit;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.*;
import org.junit.jupiter.params.provider.*;
import java.nio.file.*;
import java.io.IOException;
import static org.assertj.core.api.Assertions.assertThat;

class Slugify {
    String toSlug(String s) { return s.trim().toLowerCase().replaceAll("\\s+","-"); }
}

public class SlugifyTest {

    @TempDir Path tmp;

    @ParameterizedTest(name = "{index} => ''{0}'' -> ''{1}''")
    @CsvSource({
        "'  Hello  World  ', hello-world",
        "'Spring   Boot', spring-boot",
        "'Kotlin', kotlin"
    })
    void shouldSlugInputs(String in, String expected) throws IOException {
        // Given
        Slugify s = new Slugify();
        // When
        String slug = s.toSlug(in);
        // Then
        assertThat(slug).isEqualTo(expected);

        // и временный файл для демо
        Path f = Files.writeString(tmp.resolve(slug + ".txt"), slug);
        assertThat(Files.exists(f)).isTrue();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package stack.junit

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.io.TempDir
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.CsvSource
import java.nio.file.Files
import java.nio.file.Path

class Slugify { fun toSlug(s: String) = s.trim().lowercase().replace("\\s+".toRegex(), "-") }

class SlugifyTest {

    @TempDir
    lateinit var tmp: Path

    @ParameterizedTest(name = "{index} => '{0}' -> '{1}'")
    @CsvSource(
        "'  Hello  World  ', hello-world",
        "'Spring   Boot', spring-boot",
        "'Kotlin', kotlin"
    )
    fun shouldSlugInputs(input: String, expected: String) {
        val slug = Slugify().toSlug(input)
        assertThat(slug).isEqualTo(expected)
        val f = Files.writeString(tmp.resolve("$slug.txt"), slug)
        assertThat(Files.exists(f)).isTrue()
    }
}
```

---

## Mockito/MockK и подходы к double’ам: stub vs mock vs spy

Тестовые «двойники» (test doubles) позволяют изолировать модуль от внешних зависимостей и наблюдать взаимодействия. **Stub** возвращает предопределённые значения, **mock** проверяет, что были вызваны нужные методы с нужными аргументами, **spy** оборачивает реальный объект, позволяя частично подменять поведение и верифицировать вызовы. В Java-мире стандарт — Mockito; в Kotlin-мира — MockK, который лучше работает с final-классами и корутинами.

Выбор двойника диктуется целью теста. Если вам важно поведение «чёрного ящика», используйте stubs и проверяйте результат. Если же тест подтверждает протокол взаимодействия («должен быть вызван ретрай с экспоненциальной паузой»), нужен mock с верификацией. Spy хорош, когда вы хотите сохранить часть реальной логики, но подкрутить, например, источник времени или внешний вызов.

Mockito в Java предоставляет лаконичный API: `when(...).thenReturn(...)` для стаба и `verify(...)` для проверки вызовов. Начиная с Mockito 4+, поддержка конструкторных/final классов стала проще, но по-прежнему Kotlin-специфика (data-классы, final по умолчанию) даётся лучше MockK. В смешанных командах можно использовать обе библиотеки: Mockito для Java-модулей, MockK для Kotlin-модулей.

Антипаттерн — «мокать всё подряд». Избыточные моки усложняют тест и повышают хрупкость: любое изменение внутренней реализации ломает тесты. Старайтесь мокать только границы: сетевые клиенты, брокеры, системное время, тяжёлые ресурсы. Остальное — через реальные простые классы и чистые функции.

Отдельно следите за «verify-only» тестами, где нет проверок результата. Они часто становятся бесполезными: проверяют «что-то вызвалось», но не фиксируют смысл. Даже при верификации вызова полезно подтвердить эффект (изменение состояния, возврат результата).

Тесты ретраев/таймаутов удобно писать через моки с `Answer`, чтобы имитировать последовательность: «первый вызов кидает исключение, второй — возвращает ответ». Это читаемо и надёжно фиксирует ожидаемый протокол повторов.

Spy используйте осторожно. Он легко превращается в проверку «внутренностей», а не контракта. Если требуется менять поведение метода ради теста — возможно, код нуждается в перегруппировке зависимостей или введении интерфейса.

В Kotlin MockK особенно удобен для захвата аргументов и проверки корутин (`coEvery`, `coVerify`). Это даёт лаконичный код для асинхронных сценариев. Но важно не смешивать стили: если модуль на Java, не тащите MockK только ради одного теста.

Не забывайте о явных фикстурах. Вынесите создание моков в хелперы/билдеры, но без магии: в тесте должно быть видно, что именно смоделировано. Эксплицитность — это читаемость.

И наконец, делайте «тяжёлые» двойники редкими. Если интеграция критична, лучше поднять WireMock/Testcontainers и написать компонентный тест, чем пытаться воспроизвести протокол мока.

**Gradle (Groovy)**

```groovy
dependencies {
    testImplementation "org.mockito:mockito-junit-jupiter:5.14.2"
    testImplementation "io.mockk:mockk:1.13.12"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    testImplementation("org.mockito:mockito-junit-jupiter:5.14.2")
    testImplementation("io.mockk:mockk:1.13.12")
}
```

**Java — stub/mock/spy c Mockito**

```java
package doubles.mockito;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.assertThat;

interface PaymentGateway { String charge(int cents); }

class BillingService {
    private final PaymentGateway gw;
    BillingService(PaymentGateway gw) { this.gw = gw; }
    String pay(int cents) { return gw.charge(cents); }
}

@ExtendWith(MockitoExtension.class)
class BillingServiceTest {

    @Mock PaymentGateway gw; // mock

    @Test
    void usesGatewayAndReturnsReceipt() {
        // stub
        when(gw.charge(1000)).thenReturn("OK-123");

        // when
        var svc = new BillingService(gw);
        String receipt = svc.pay(1000);

        // then
        verify(gw).charge(1000);
        assertThat(receipt).isEqualTo("OK-123");
    }

    @Test
    void spyExample() {
        var real = new PaymentGateway() { public String charge(int c){ return "OK-"+c; } };
        var spyGw = spy(real);
        doReturn("OK-OVERRIDE").when(spyGw).charge(200);

        assertThat(spyGw.charge(100)).isEqualTo("OK-100");
        assertThat(spyGw.charge(200)).isEqualTo("OK-OVERRIDE");
        verify(spyGw, times(1)).charge(100);
        verify(spyGw, times(1)).charge(200);
    }
}
```

**Kotlin — эквивалент с MockK**

```kotlin
package doubles.mockk

import io.mockk.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

interface PaymentGateway { fun charge(cents: Int): String }

class BillingService(private val gw: PaymentGateway) {
    fun pay(cents: Int) = gw.charge(cents)
}

class BillingServiceTest {

    @Test
    fun usesGatewayAndReturnsReceipt() {
        val gw = mockk<PaymentGateway>()
        every { gw.charge(1000) } returns "OK-123"

        val svc = BillingService(gw)
        val receipt = svc.pay(1000)

        verify { gw.charge(1000) }
        assertThat(receipt).isEqualTo("OK-123")
    }

    @Test
    fun spyExample() {
        val real = object : PaymentGateway { override fun charge(cents: Int) = "OK-$cents" }
        val spy = spyk(real)
        every { spy.charge(200) } returns "OK-OVERRIDE"

        assertThat(spy.charge(100)).isEqualTo("OK-100")
        assertThat(spy.charge(200)).isEqualTo("OK-OVERRIDE")
        verify(exactly = 1) { spy.charge(100) }
        verify(exactly = 1) { spy.charge(200) }
    }
}
```

---

## Spring Boot Test: автоконфигурация тестов, TestExecutionListeners, Context Caching

`spring-boot-starter-test` включает автоконфигурацию, которая «умно» поднимает контекст для тестов. Аннотация `@SpringBootTest` создаёт полный `ApplicationContext`, а срезы (`@WebMvcTest`, `@DataJpaTest` и др.) — минимальные подмножества. Это снижает цену интеграционных проверок и делает воспроизводимыми сценарии, где важны бины, прокси, фильтры и конфигурации.

Контекст в JUnit 5 кэшируется между тестовыми классами (Context Caching). Если набор аннотаций и свойств совпадает, Spring повторно использует уже построенный `ApplicationContext`. Это даёт существенный выигрыш по времени. Аннотация `@DirtiesContext` инвалидирует кэш намеренно — применяйте её экономно и только когда вы действительно меняете состояние контекста, влияющее на другие тесты.

`TestExecutionListeners` — механизм «крючков» в жизненный цикл тестов. Через них можно, например, инициализировать внешние сервисы, публиковать сообщения, включать обёртки логирования. В большинстве случаев хватит автоконфигураций Spring Boot, но специфические сценарии (подмена `Clock`, прогрев кэшей) удобно реализовать слушателями.

Свойства в тестах управляются через `@TestPropertySource`, `@DynamicPropertySource`, профили (`@ActiveProfiles`). Последний вариант предпочтителен, так как повторяет боевую практику: профиль `test` содержит «тестовую» конфигурацию, а динамические свойства применяются лишь там, где значения известны только во время запуска (например, порты Testcontainers).

Жизненный цикл веб-тестов задаётся через `webEnvironment` в `@SpringBootTest`: `NONE` — без веб-серверной части, `RANDOM_PORT` — поднимается сервер на случайном порту (для RestAssured/реальных HTTP-клиентов), `DEFINED_PORT` — на фиксированном. Для изоляции и параллельных запусков используйте `RANDOM_PORT`.

Поведение сериализации и `ObjectMapper` в тестах наследует настройки вашего приложения. В срезах `@JsonTest` Spring поднимет только JSON-слой, включая кастомные модули (JavaTime, Kotlin). Это быстрый способ проверить форматы и правила (например, `WRITE_DATES_AS_TIMESTAMPS=false`).

Для повышения наблюдаемости полезно подключать `TestExecutionListener`, который ставит MDC-идентификатор на время теста и пишет понятные заголовки в логи. Это ускоряет расследование «красноты» на CI, где смешиваются логи нескольких параллельных джобов.

Не забывайте, что полные контексты дорогие. Группируйте интеграционные тесты в немного классов с общим контекстом, вместо десятков классов, каждый из которых поднимает свой. Это уменьшит общее время прогона без потери изоляции.

И наконец, помните о порядке инициализации. Если у вас есть бины, зависящие от внешнего окружения (секреты, сеть), «отрубайте» их тестовым профилем или моками. Тест должен подниматься мгновенно и без попыток ходить наружу.

**Java — @SpringBootTest, DynamicPropertySource и TestExecutionListener**

```java
package sboot.testinfra;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.*;
import org.springframework.test.context.support.AbstractTestExecutionListener;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("test")
@TestExecutionListeners(listeners = {SbootTestInfraTest.LogListener.class}, mergeMode = TestExecutionListeners.MergeMode.MERGE_WITH_DEFAULTS)
class SbootTestInfraTest {

    @Value("${app.flag:true}")
    boolean flag;

    @DynamicPropertySource
    static void dynProps(DynamicPropertyRegistry r) {
        r.add("app.flag", () -> "true");
    }

    @Test
    void contextLoadsAndPropsInjected() {
        assertThat(flag).isTrue();
    }

    static class LogListener extends AbstractTestExecutionListener {
        @Override public void beforeTestClass(TestContext testContext) {
            testContext.getApplicationContext().getEnvironment().getSystemProperties()
                .put("test.mdc.id", testContext.getTestClass().orElseThrow().getSimpleName());
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package sboot.testinfra

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Value
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("test")
class SbootTestInfraTest {

    @Value("\${app.flag:true}")
    lateinit var flagStr: String

    companion object {
        @JvmStatic
        @DynamicPropertySource
        fun dynProps(r: DynamicPropertyRegistry) {
            r.add("app.flag") { "true" }
        }
    }

    @Test
    fun contextLoadsAndPropsInjected() {
        assertThat(flagStr.toBoolean()).isTrue()
    }
}
```

---

## Testcontainers (Postgres, Kafka, Redis, MQ): reusability, lifecycle, network aliases

Testcontainers поднимает реальные Docker-контейнеры баз данных и брокеров прямо из теста. Это золотая середина между «быстро, но нереалистично» (H2/embedded) и «реалистично, но тяжело» (общий стенд). Для Spring Boot он идеально сочетается с `@DynamicPropertySource`, автоматом прокидывая URL/порты контейнеров в `DataSource`, Kafka-клиенты и прочие бины.

Жизненный цикл контейнеров управляется аннотациями JUnit 5: `@Testcontainers` на классе и `@Container` на статическом поле создают контейнер единожды на класс. Это ускоряет тесты и делает их детерминированными. Если нужен контейнер на весь набор (`static` singleton), используйте статические поля и выделенный базовый класс, но следите за изоляцией данных.

Reusability — возможность переиспользовать один и тот же контейнер между запусками Gradle/CI. Для этого существует режим reuse (включается в конфиге Testcontainers), но он уместен лишь локально и на CI-агентах с долгоживущими воркерами. Важно помнить, что такой режим повышает риск «грязного состояния», поэтому используйте его осознанно.

Сетевые настройки позволяют создавать `Network` и давать контейнерам `networkAliases`. Это критично, когда сервисы резолвят друг друга по DNS-имени, а не по `localhost:port`. Например, приложение ожидает `postgres:5432` — вы задаёте контейнеру alias `postgres`, а приложению прокидываете именно это имя.

Kafka/Redis/RabbitMQ тестируются так же, как Postgres: поднимаете контейнер, прокидываете `bootstrapServers`/`spring.redis.host`/`spring.rabbitmq.host`. Для Kafka полезно задавать `withNetworkAliases("kafka")` и использовать один `Network` с приложением, если вы запускаете «чёрный ящик» (RestAssured + реальный порт).

Логика жизненного цикла проста: подняли контейнер → выполнили миграции → запустили тесты → по завершении контейнер удалился. Если вы видите долгую инициализацию на каждом тесте, проверьте, что контейнер не создаётся в каждом методе, и вынесите его на уровень класса.

Хитрость с «alias» полезна и для интеграции с внешним тестируемым сервисом, который ожидает окружение (например, Debezium, MQ). Вы можете построить мини-сеть из контейнеров и своего приложения, протестировав реальный протокол без моков.

Разумно собирать абстракции: базовый класс `AbstractPostgresIT` с статическим контейнером и `DynamicPropertySource` снимает копипасту. Аналогично для Kafka/Redis. Не забывайте про ресурсоёмкость: запускайте тяжёлые наборы отдельной задачей.

И наконец, помните про изоляцию данных. Чистите БД через транзакции/`@Sql` или создавайте новый schema/database на каждый тест, если сценарии конфликтуют. Контейнеры дают «чистую машину», но не решают логическую изоляцию ваших данных.

**Gradle (оба DSL)**

```groovy
dependencies {
    testImplementation "org.testcontainers:junit-jupiter:1.20.2"
    testImplementation "org.testcontainers:postgresql:1.20.2"
    testImplementation "org.testcontainers:kafka:1.20.2"
    testImplementation "org.testcontainers:redis:1.20.2"
    testImplementation "org.testcontainers:rabbitmq:1.20.2"
}
```

**Java — Postgres + Kafka с сетью и alias**

```java
package tc.sample;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.TestInstance;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.testcontainers.containers.*;
import org.testcontainers.utility.DockerImageName;
import org.testcontainers.containers.Network;

import static org.assertj.core.api.Assertions.assertThat;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class ContainersTest {

    static Network net = Network.newNetwork();

    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>(DockerImageName.parse("postgres:16-alpine"))
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app")
            .withNetwork(net)
            .withNetworkAliases("postgres");

    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.1"))
            .withNetwork(net)
            .withNetworkAliases("kafka");

    @BeforeAll void start() { pg.start(); kafka.start(); }
    @AfterAll  void stop()  { kafka.stop(); pg.stop(); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void containersUp() {
        assertThat(pg.isRunning()).isTrue();
        assertThat(kafka.isRunning()).isTrue();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package tc.sample

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.*
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.KafkaContainer
import org.testcontainers.containers.Network
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.utility.DockerImageName

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class ContainersTest {

    companion object {
        private val net: Network = Network.newNetwork()
        private val pg = PostgreSQLContainer(DockerImageName.parse("postgres:16-alpine"))
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app")
            .withNetwork(net)
            .withNetworkAliases("postgres")

        private val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.1"))
            .withNetwork(net)
            .withNetworkAliases("kafka")

        @JvmStatic
        @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("spring.datasource.url", pg::getJdbcUrl)
            r.add("spring.datasource.username", pg::getUsername)
            r.add("spring.datasource.password", pg::getPassword)
            r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
        }
    }

    @BeforeAll fun start() { pg.start(); kafka.start() }
    @AfterAll fun stop()   { kafka.stop(); pg.stop() }

    @Test
    fun containersUp() {
        assertThat(pg.isRunning).isTrue()
        assertThat(kafka.isRunning).isTrue()
    }
}
```

---

## WireMock/MockServer для HTTP-интеграций; Awaitility для асинхронных ожиданий

WireMock и MockServer решают задачу изоляции HTTP-интеграций. Вы описываете ожидаемые запросы и подставляете ответы с нужными статусами, заголовками, телом и задержками. Это делает тесты независимыми от внешних систем и позволяет детерминированно проверять ретраи, тайм-ауты, idempotency-ключи и поведение при сбоях.

WireMock прост встраивается как «встраиваемый сервер» (`WireMockServer`) на случайном порту. В тесте вы поднимаете его в `@BeforeAll`, настраиваете стабы через DSL и прокидываете базовый URL вашему HTTP-клиенту (RestTemplate/WebClient/Feign). После теста сервер останавливается. Такой подход подходит и для `@SpringBootTest` (через `@DynamicPropertySource`) и для «срезов».

Awaitility — библиотека «умных ожиданий». Вместо `Thread.sleep` вы пишете условие, которое будет проверяться с интервалом и тайм-аутом: «ждать до 2 секунд, пока из очереди будет прочитано N событий» или «пока репозиторий сохранит запись». Это повышает стабильность тестов асинхронного кода и не «жжёт» лишнее время.

При моделировании сбоев полезно комбинировать WireMock с «последовательными» стабы: первый запрос — 500, второй — 200. Так вы тестируете логику ретраев и backoff без сложных таймеров. Уверьтесь, что клиент действительно делает повтор (верифицируйте счётчик вызовов и интервалы, если это важно).

MockServer даёт схожие возможности, но ближе к «протокольной» верификации и поддерживает записываемые expectations. Если в вашей команде уже есть MockServer-практика, смело используйте; если нет — WireMock проще на старте.

Для больших интеграций WireMock можно запускать как контейнер Testcontainers (образ wiremock), но чаще достаточно встроенного сервера. Контейнер уместен, если у вас есть предзагруженные стабы и вы хотите делиться ими между тестовыми классами.

Не забывайте о логировании запросов. WireMock умеет писать журнал совпадений и промахов — это ускоряет дебаг и даёт понимание, почему тест не «поймал» ожидание. В отчёте CI лог WireMock — важный артефакт.

При проверке идемпотентности фиксируйте `Idempotency-Key` и счётчик обращений на стороне WireMock. Это помогает выявить случайные повторные запросы клиента и рассинхрон с серверной политикой.

В реактивных тестах Awaitility спасает от гонок. Вы можете ждать появления элемента в «горячем» паблишере или записи в репозитории, не блокируя поток надолго. Держите тайм-ауты разумными (счёт на секунды), а интервалы — небольшими (десятки миллисекунд).

И наконец, помните об изоляции портов. Выделяйте случайные порты, публикуйте их в свойства через `@DynamicPropertySource`, чтобы параллельные сборки не конфликтовали.

**Gradle (оба DSL)**

```groovy
dependencies {
    testImplementation "com.github.tomakehurst:wiremock-jre8:2.35.1"
    testImplementation "org.awaitility:awaitility:4.2.1"
}
```

**Java — WireMock + Awaitility**

```java
package http.wiremock;

import com.github.tomakehurst.wiremock.WireMockServer;
import org.junit.jupiter.api.*;
import org.springframework.web.client.RestClient;
import java.time.Duration;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.awaitility.Awaitility.await;
import static org.assertj.core.api.Assertions.assertThat;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class WireMockTest {

    WireMockServer wm = new WireMockServer(0); // random port
    RestClient client;
    volatile String last;

    @BeforeAll
    void start() {
        wm.start();
        configureFor("localhost", wm.port());
        stubFor(get(urlEqualTo("/ping"))
            .inScenario("retry")
            .whenScenarioStateIs(STARTED)
            .willReturn(aResponse().withStatus(500))
            .willSetStateTo("ok"));

        stubFor(get(urlEqualTo("/ping"))
            .inScenario("retry")
            .whenScenarioStateIs("ok")
            .willReturn(aResponse().withStatus(200).withBody("pong")));

        client = RestClient.builder().baseUrl("http://localhost:" + wm.port()).build();
    }

    @AfterAll void stop() { wm.stop(); }

    @Test
    void clientRetriesAndEventuallySucceeds() {
        for (int i = 0; i < 2; i++) {
            try {
                last = client.get().uri("/ping").retrieve().body(String.class);
                break;
            } catch (Exception ignored) {
                try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        }
        await().atMost(Duration.ofSeconds(1)).untilAsserted(() -> assertThat(last).isEqualTo("pong"));
        verify(2, getRequestedFor(urlEqualTo("/ping")));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package http.wiremock

import com.github.tomakehurst.wiremock.WireMockServer
import com.github.tomakehurst.wiremock.client.WireMock.*
import org.assertj.core.api.Assertions.assertThat
import org.awaitility.Awaitility.await
import org.junit.jupiter.api.*
import org.springframework.web.client.RestClient
import java.time.Duration

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class WireMockTest {

    private val wm = WireMockServer(0)
    private lateinit var client: RestClient
    @Volatile private var last: String? = null

    @BeforeAll
    fun start() {
        wm.start()
        configureFor("localhost", wm.port())
        stubFor(get(urlEqualTo("/ping")).inScenario("retry")
            .whenScenarioStateIs(STARTED)
            .willReturn(aResponse().withStatus(500))
            .willSetStateTo("ok"))
        stubFor(get(urlEqualTo("/ping")).inScenario("retry")
            .whenScenarioStateIs("ok")
            .willReturn(aResponse().withStatus(200).withBody("pong")))
        client = RestClient.builder().baseUrl("http://localhost:${wm.port()}").build()
    }

    @AfterAll fun stop() { wm.stop() }

    @Test
    fun clientRetriesAndEventuallySucceeds() {
        repeat(2) {
            try {
                last = client.get().uri("/ping").retrieve().body(String::class.java)
                return@repeat
            } catch (_: Exception) {
                Thread.sleep(100)
            }
        }
        await().atMost(Duration.ofSeconds(1)).untilAsserted { assertThat(last).isEqualTo("pong") }
        verify(2, getRequestedFor(urlEqualTo("/ping")))
    }
}
```

---

## RestAssured/MockMvc/WebTestClient (MVC/WebFlux), JSONAssert/JsonUnit/JsonPath

Выбор инструмента зависит от цели. **MockMvc** — быстрый «локальный» клиент MVC-приложения без реального сервера; он запускает фильтры, маппинг и сериализацию, но общение идёт внутри JVM. **WebTestClient** делает то же для WebFlux, поддерживая реактивный контракт. **RestAssured** работает поверх реального HTTP-порта и годится для «чёрного ящика» (`@SpringBootTest(RANDOM_PORT)`) и e2e-сценариев.

JSON-проверки удобно делать через **JSONAssert** (порядок полей игнорируется, можно выбирать строгий/нестрогий режим), **JsonPath** (адресация по пути и проверки отдельных значений) и **JsonUnit** (богатые матчеры и толерантность к форматам). В большинстве проектов достаточно JSONAssert и JsonPath, но JsonUnit может понравиться тем, кто любит комбинированные матчинги.

MockMvc хорош для проверки фильтров безопасности, биндинга параметров, сериализации ошибок и контроллеров. Вы получаете быстрые, детерминированные тесты без ожидания реального сокета. Включайте `@WebMvcTest` для узких срезов или `@SpringBootTest(NONE)` + `@AutoConfigureMockMvc` для чуть более широкого охвата.

WebTestClient в реактивных проектах позволяет тестировать Flux/Mono, SSE и WebSocket-апгрейды. Он дружит с шаговыми ожиданиями и даёт удобный DSL для проверки заголовков, статусов и тел. Вы можете использовать его и против «чужого» сервера, указав базовый URL.

RestAssured силён там, где важно «как в проде»: реальный порт, реальный стек, фильтры, компрессия, CORS. Он чуть медленнее, но идеален для smoke-наборов и сценариев, где вы хотите проверить кэш-заголовки, GZIP и поведение прокси.

При проверке JSON отдавайте предпочтение «смысловым» проверкам: ключевые поля, типы, формат дат, наличие обязательных атрибутов. Не проверяйте «всё подряд», иначе тесты станут хрупкими при безвредных добавлениях полей.

Старайтесь не смешивать инструменты в одном модуле без нужды. Если у вас MVC — MockMvc покроет 80% сценариев. RestAssured оставьте для нескольких важнейших «чёрных ящиков». Реактив — аналогично с WebTestClient.

Для кросс-проектной согласованности заведите «базовую обвязку»: создание клиента, общие матчинги заголовков, проверка Problem Details. Это сократит копипасту и сделает тесты схожими в стиле.

И наконец, помните про локализацию и часовые пояса при JSON-проверках. Если сериализация дат зависит от настроек `ObjectMapper`, тест должен явно фиксировать их, иначе будет «краснеть» в неожиданных местах.

**Gradle (оба DSL)**

```groovy
dependencies {
    testImplementation "io.rest-assured:rest-assured:5.5.0"
    testImplementation "org.skyscreamer:jsonassert:1.5.3"
    testImplementation "com.jayway.jsonpath:json-path:2.9.0"
    testImplementation "org.springframework.boot:spring-boot-starter-webflux"
}
```

**Java — MockMvc + JSONAssert и RestAssured на RANDOM_PORT**

```java
package web.clients;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import org.skyscreamer.jsonassert.JSONAssert;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import io.restassured.RestAssured;

@RestController
@RequestMapping("/api/greet")
class GreetController {
    @GetMapping public Greeting hello() { return new Greeting("hello"); }
    record Greeting(String message){}
}

@WebMvcTest(GreetController.class)
class MockMvcJsonTest {

    @Autowired MockMvc mvc;

    @Test
    void jsonAssertLoose() throws Exception {
        var body = mvc.perform(get("/api/greet"))
            .andExpect(status().isOk())
            .andReturn().getResponse().getContentAsString();
        JSONAssert.assertEquals("{\"message\":\"hello\"}", body, false);
    }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RestAssuredTest {
    @LocalServerPort int port;

    @Test
    void blackBox() {
        RestAssured.given().port(port)
            .get("/api/greet")
            .then().statusCode(200)
            .and().body("message", org.hamcrest.Matchers.equalTo("hello"));
    }
}
```

**Kotlin — WebTestClient (WebFlux) + JsonPath**

```kotlin
package web.clients

import com.jayway.jsonpath.JsonPath
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.Import
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import javax.annotation.Resource

@RestController
@RequestMapping("/rx/echo")
class EchoController { data class Echo(val msg: String); @GetMapping fun echo() = Echo("hi") }

@WebFluxTest(controllers = [EchoController::class])
@Import()
class WebFluxClientTest {

    @Resource
    lateinit var client: WebTestClient

    @Test
    fun jsonPathExtract() {
        val body = client.get().uri("/rx/echo")
            .exchange()
            .expectStatus().isOk
            .expectHeader().contentType("application/json")
            .expectBody().returnResult().responseBody!!.decodeToString()

        val msg: String = JsonPath.read(body, "$.msg")
        assertThat(msg).isEqualTo("hi")
    }
}
```

# 3. Unit-тесты домена и утилит (без Spring контекста)

## Тестирование бизнес-правил: чистые классы, value-объекты, агрегаты

Бизнес-правила проще, надёжнее и дешевле всего тестировать в отрыве от инфраструктуры. Это означает отсутствие Spring-контекста, HTTP, БД и даже логгеров: только чистые классы и функции. Такой подход минимизирует количество движущихся частей и превращает тест в быструю и детерминированную проверку инвариантов домена. Важно понимать, что «юнит» здесь — не всегда один класс; иногда это небольшая коллаборация из пары-тройки объектов, которые совместно реализуют правило.

Value-объекты (например, `Money`, `Email`, `Sku`) — естественные опорные точки для юнит-тестов. Их инварианты (валюта не пустая, сумма не отрицательная, e-mail валиден) редко меняются и служат фундаментом большей модели. Если value-объекты «протекают» ошибками, дальше по цепочке их уже трудно компенсировать. Поэтому начинайте с них: тесты компактны, а эффект — максимален.

Агрегаты (например, `Order`) несут правила целостности на уровне нескольких сущностей: «сумма заказа — сумма позиций», «нельзя оплатить пустой заказ», «нельзя отменить выполненный заказ». Эти правила хорошо выражаются как методы агрегата без побочных эффектов, которые возвращают новое состояние/события, а не мутируют внешние ресурсы. Тогда тесты выглядят как чистая математика: подготовили вход, вызвали метод, сверили выход и инварианты.

В юнит-тестах домена используйте привычные форматы Given–When–Then. В «Given» создавайте value-объекты через компактные фабрики/билдеры с говорящими именами, чтобы не засорять тест обилием конструкторов. В «When» вызывайте ровно один доменный метод. В «Then» проверяйте инварианты, включая сумму, статусы, счётчики — но избегайте проверки внутренней реализации (например, вызов приватного метода).

Юнит-уровень не должен зависеть от часов и случайности. Если в домене есть время (например, «заказ истекает через 15 минут»), протяните внутрь агрегата `Clock` и подайте фиксированное время из теста. Это не «инфраструктура», это часть правила. Так вы исключите флейки-эффекты «в 23:59» и нестабильность на CI.

Границу правил полезно фиксировать проверкой исключений и результатов «ошибок домена». Ненормальные ситуации — часть спецификации: «нельзя добавить позицию с нулевым количеством», «нельзя оплатить заказ без позиций». Лучше бросить чёткое доменное исключение (например, `DomainException`) с сообщением, которое попадёт в лог и поможет локализовать проблему, чем вернуть «магический» код.

Тесты домена хорошо сочетаются с property-based идеями: вы формулируете свойство («сумма заказа — сумма цен позиций») и проверяете его на множестве случайных входов. Даже без сторонних библиотек можно пройтись циклом по 100–1000 генераций — это раскрывает неожиданные углы. Главное — зафиксировать seed, чтобы воспроизводить падения.

Если доменная операция порождает события («OrderCreated», «ItemAdded»), тестируйте именно список событий или новое состояние агрегата. Избегайте «подглядывания» во внутренние поля, которые не являются контрактом. События — публичный контракт доменного слоя, и проверка их состава/порядка максимально полезна.

Подход «чистый домен» помогает и в рефакторинге: тесты остаются неизменными при переносе приложения с MVC на WebFlux, при смене базы или при разбиении монолита. Бизнес-правила не должны знать, откуда пришли данные и куда уйдут; они только решают, «что верно».

Наконец, не гонитесь за покрытием 100% именно в домене — гонитесь за покрытием смысла. Если инвариант формулируется несколькими правилами, проверьте каждое правило на позитивных и негативных примерах. Это принесёт больше пользы, чем «покрыть» каждую ветку без осмысления.

**Java — Value Object и агрегат с тестами (чистый JUnit 5, без Spring)**

```java
package domain.order;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.*;

import static java.util.Objects.requireNonNull;

public final class Money {
    public final String currency;
    public final BigDecimal amount;

    public Money(String currency, BigDecimal amount) {
        this.currency = requireNonNull(currency, "currency");
        this.amount = requireNonNull(amount, "amount").setScale(2, RoundingMode.HALF_UP);
        if (this.amount.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("negative amount");
        if (this.currency.isBlank()) throw new IllegalArgumentException("blank currency");
    }

    public Money add(Money other) {
        ensureSameCurrency(other);
        return new Money(currency, amount.add(other.amount));
    }

    public Money mul(int qty) {
        if (qty <= 0) throw new IllegalArgumentException("qty <= 0");
        return new Money(currency, amount.multiply(BigDecimal.valueOf(qty)));
    }

    private void ensureSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) throw new IllegalArgumentException("currency mismatch");
    }

    @Override public String toString() { return amount + " " + currency; }
}

final class Order {
    public enum Status { NEW, PAID, CANCELLED }
    private final UUID id;
    private final String currency;
    private final List<OrderLine> lines = new ArrayList<>();
    private Status status = Status.NEW;

    public Order(UUID id, String currency) {
        this.id = requireNonNull(id);
        this.currency = requireNonNull(currency);
    }

    public void addLine(String sku, Money unitPrice, int qty) {
        if (!unitPrice.currency.equals(currency)) throw new IllegalArgumentException("currency mismatch");
        lines.add(new OrderLine(sku, unitPrice, qty));
    }

    public void pay() {
        if (lines.isEmpty()) throw new IllegalStateException("empty order");
        if (status != Status.NEW) throw new IllegalStateException("already processed");
        status = Status.PAID;
    }

    public Money total() {
        Money acc = new Money(currency, BigDecimal.ZERO);
        for (OrderLine l : lines) acc = acc.add(l.unitPrice.mul(l.qty));
        return acc;
    }

    public Status status() { return status; }

    static final class OrderLine {
        final String sku; final Money unitPrice; final int qty;
        OrderLine(String sku, Money unitPrice, int qty) {
            this.sku = requireNonNull(sku); this.unitPrice = requireNonNull(unitPrice);
            if (qty <= 0) throw new IllegalArgumentException("qty <= 0");
            this.qty = qty;
        }
    }
}
```

```java
package domain.order;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class OrderDomainTest {

    @Test
    void totalIsSumOfLines() {
        var order = new Order(UUID.randomUUID(), "USD");
        order.addLine("SKU-1", new Money("USD", new BigDecimal("10.00")), 2);
        order.addLine("SKU-2", new Money("USD", new BigDecimal("3.50")), 1);

        assertThat(order.total().amount).isEqualByComparingTo("23.50");
    }

    @Test
    void cannotPayEmptyOrder() {
        var order = new Order(UUID.randomUUID(), "USD");
        assertThatThrownBy(order::pay).isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("empty");
    }

    @Test
    void cannotMixCurrencies() {
        var order = new Order(UUID.randomUUID(), "USD");
        assertThatThrownBy(() -> order.addLine("X", new Money("EUR", new BigDecimal("1.00")), 1))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("currency mismatch");
    }
}
```

**Kotlin — эквивалент value-object и агрегат с тестами**

```kotlin
package domain.order

import org.assertj.core.api.Assertions.assertThat
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test
import java.math.BigDecimal
import java.math.RoundingMode
import java.util.UUID

data class Money(val currency: String, val amount: BigDecimal) {
    init {
        require(currency.isNotBlank()) { "blank currency" }
        require(amount.setScale(2, RoundingMode.HALF_UP) >= BigDecimal.ZERO) { "negative amount" }
    }
    fun add(other: Money): Money {
        require(currency == other.currency) { "currency mismatch" }
        return Money(currency, amount + other.amount)
    }
    fun mul(qty: Int): Money {
        require(qty > 0) { "qty <= 0" }
        return Money(currency, amount.multiply(BigDecimal(qty)))
    }
}

class Order(private val id: UUID, private val currency: String) {
    enum class Status { NEW, PAID, CANCELLED }
    private val lines = mutableListOf<OrderLine>()
    var status: Status = Status.NEW
        private set

    fun addLine(sku: String, unitPrice: Money, qty: Int) {
        require(unitPrice.currency == currency) { "currency mismatch" }
        lines += OrderLine(sku, unitPrice, qty)
    }

    fun pay() {
        check(lines.isNotEmpty()) { "empty order" }
        check(status == Status.NEW) { "already processed" }
        status = Status.PAID
    }

    fun total(): Money =
        lines.fold(Money(currency, BigDecimal.ZERO)) { acc, l -> acc.add(l.unitPrice.mul(l.qty)) }

    private data class OrderLine(val sku: String, val unitPrice: Money, val qty: Int) {
        init { require(qty > 0) { "qty <= 0" } }
    }
}

class OrderDomainTest {
    @Test fun totalIsSumOfLines() {
        val order = Order(UUID.randomUUID(), "USD")
        order.addLine("SKU-1", Money("USD", BigDecimal("10.00")), 2)
        order.addLine("SKU-2", Money("USD", BigDecimal("3.50")), 1)
        assertThat(order.total().amount).isEqualByComparingTo(BigDecimal("23.50"))
    }
    @Test fun cannotPayEmptyOrder() {
        val order = Order(UUID.randomUUID(), "USD")
        assertThatThrownBy { order.pay() }.hasMessageContaining("empty")
    }
    @Test fun cannotMixCurrencies() {
        val order = Order(UUID.randomUUID(), "USD")
        assertThatThrownBy { order.addLine("X", Money("EUR", BigDecimal("1.00")), 1) }
            .hasMessageContaining("currency mismatch")
    }
}
```

**Gradle (Groovy/Kotlin) — базовые зависимости для юнит-тестов**

```groovy
dependencies {
    testImplementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    testImplementation "org.junit.jupiter:junit-jupiter-api"
    testRuntimeOnly "org.junit.platform:junit-platform-launcher"
    testImplementation "org.assertj:assertj-core:3.26.3"
}
```

```kotlin
dependencies {
    testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    testImplementation("org.junit.jupiter:junit-jupiter-api")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
    testImplementation("org.assertj:assertj-core:3.26.3")
}
```

---

## Стратегии создания фикстур: Test Data Builder, Mother Object, Randomized but stable

Фикстуры — это данные, на которых выполняется сценарий. Плохо, когда тест захламлён десятком конструкторов и «магическими» значениями. Хорошо, когда читается «заказ с одной дорогой и одной дешёвой позицией» и видно, какие свойства важны, а какие — по умолчанию. Для этого применяют три приёма: **Test Data Builder**, **Mother Object** и **Randomized but stable**.

Test Data Builder — класс-строитель для тестовых объектов с разумными значениями по умолчанию и методами для переопределения важных полей. Это уменьшает шум и делает настройки явными. Особенно полезно, когда у DTO/сущности длинный список полей, а тесту важны два-три.

Mother Object — набор фабрик «говорящих» пресетов: `aPaidOrder()`, `anEmptyOrder()`, `aRichCustomer()`. Такой стиль отлично документирует намерение теста и ускоряет чтение. Важно не перегружать Mother логикой: если пресетов слишком много, лучше вернуться к билдеру.

Randomized but stable — рандомизация с фиксированным seed. В отличие от «жёстко прошитых» значений, такие фикстуры создают больше разнообразия, а фиксированный seed гарантирует воспроизводимость падений. Установив seed в начале теста, вы получаете одинаковый набор данных в каждом прогоне — и можете легко восстановить сценарий по логу.

Фикстуры должны находиться рядом с тестами — в `src/test/java|kotlin`. Это избавляет прод-код от зависимостей на тестовые конструкторы и сохраняет тестовую модель независимой. Иногда общие билдеры/матери можно вынести в `testFixtures` Gradle-модуля и переиспользовать межмодульно.

Хороший билдер «тонкий»: он не должен скрывать доменную валидацию. Если в домене нельзя отрицательную сумму — билдер не должен молча чинить её. Пусть тест явно укажет «злые» данные, если хочет проверить ошибку.

Старайтесь подбирать имена методов-настроек, которые говорят о намерении: `withQuantity(0)` видно хуже, чем `withZeroQuantity()`, если это ключевой аспект сценария. Но не увлекайтесь: дублировать все возможные комбинации в именах — путь к взрыву количества методов.

Комбинируйте приёмы: Mother возвращает билдер с нужной базой, а тест дополняет важные поля. Так сохраняется читаемость и гибкость. Например, `OrdersMother.aReadyToPay().withItem("sku", price, qty).build()`.

Рандомизацию лучше ограничивать в пределах теста. Не используйте глобальный `Math.random()` без seed; вместо этого создайте `Random(seed)` и прокиньте его в билдер. Это снимет флейки, а разнообразие всё равно останется.

Наконец, помните о локалях, форматах и времени. Если билдер генерирует строки/даты, используйте `Locale.ROOT` и UTC, чтобы результат был одинаковым на всех машинах.

**Java — Mother + Builder + стабильный Random**

```java
package fixtures;

import domain.order.Money;
import domain.order.Order;

import java.math.BigDecimal;
import java.util.Random;
import java.util.UUID;

public final class OrdersMother {

    public static OrderBuilder aNewOrderUSD() {
        return new OrderBuilder(UUID.randomUUID(), "USD");
    }

    public static final class OrderBuilder {
        private final UUID id;
        private final String currency;
        private final Random rnd = new Random(42);

        private OrderBuilder(UUID id, String currency) {
            this.id = id; this.currency = currency;
        }

        public OrderBuilder withRandomLine() {
            String sku = "SKU-" + rnd.nextInt(1000);
            BigDecimal price = new BigDecimal(rnd.nextInt(100) + ".00");
            int qty = rnd.nextInt(3) + 1;
            this.withLine(sku, new Money(currency, price), qty);
            return this;
        }

        public OrderBuilder withLine(String sku, Money unitPrice, int qty) {
            // create or update builder state; for simplicity we apply directly to built order
            if (built == null) built = new Order(id, currency);
            built.addLine(sku, unitPrice, qty);
            return this;
        }

        private Order built;

        public Order build() {
            if (built == null) built = new Order(id, currency);
            return built;
        }
    }
}
```

```java
package fixtures;

import domain.order.Money;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

class OrdersMotherTest {

    @Test
    void motherBuilderCreatesReadableFixtures() {
        var order = OrdersMother.aNewOrderUSD()
                .withLine("SKU-1", new Money("USD", new BigDecimal("5.00")), 2)
                .withRandomLine()
                .build();

        assertThat(order.total().amount).isGreaterThanOrEqualTo(new BigDecimal("10.00"));
    }
}
```

**Kotlin — те же приёмы**

```kotlin
package fixtures

import domain.order.Money
import domain.order.Order
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.math.BigDecimal
import java.util.Random
import java.util.UUID

object OrdersMother {
    fun aNewOrderUSD(): OrderBuilder = OrderBuilder(UUID.randomUUID(), "USD")

    class OrderBuilder(private val id: UUID, private val currency: String) {
        private var built: Order? = null
        private val rnd = Random(42)

        fun withRandomLine(): OrderBuilder {
            val sku = "SKU-${rnd.nextInt(1000)}"
            val price = BigDecimal("${rnd.nextInt(100)}.00")
            val qty = rnd.nextInt(3) + 1
            return withLine(sku, Money(currency, price), qty)
        }

        fun withLine(sku: String, unitPrice: Money, qty: Int): OrderBuilder {
            if (built == null) built = Order(id, currency)
            built!!.addLine(sku, unitPrice, qty)
            return this
        }

        fun build(): Order = built ?: Order(id, currency)
    }
}

class OrdersMotherTest {
    @Test
    fun motherBuilderCreatesReadableFixtures() {
        val order = OrdersMother.aNewOrderUSD()
            .withLine("SKU-1", Money("USD", BigDecimal("5.00")), 2)
            .withRandomLine()
            .build()

        assertThat(order.total().amount).isGreaterThanOrEqualTo(BigDecimal("10.00"))
    }
}
```

---

## Проверка исключений, граничных условий, property-based идеи

Исключения — это первый класс граждан в доменном коде: они фиксируют запреты и нарушения инвариантов. Тестируйте их так же тщательно, как «позитивные» сценарии. Используйте `assertThatThrownBy` (AssertJ) с проверкой типа и фрагмента сообщения. Сообщение — часть контракта для оператора и логов; его стоит поддерживать осмысленным.

Граничные условия — «праздничные дни» ошибок. Если скидка включается «с 10 000», то обязательно проверьте 9 999, 10 000, 10 001. Если лимит позиций — 100, проверьте 99/100/101. Такие тесты предохраняют от интерпретационных ошибок («включительно/исключительно») и от будущих «незаметных» правок условий.

Property-based — это способ выражать обобщённые свойства вместо перечисления примеров. «Сумма заказа не меньше нуля», «сортировка стабильна», «обратная операция восстанавливает вход» — всё это свойства. Даже без библиотек типа jqwik вы можете запускать тест на 100–1000 случайных генераций с фиксированным seed и ловить неожиданные комбинации.

Важно не увлечься «рандомом ради рандома». Свойство должно быть осмысленным, и генерация должна покрывать реальные варианты домена. Ограничивайте диапазоны значений, чтобы тест оставался быстрым и репрезентативным; фиксируйте seed и печатайте его при падении для воспроизводимости.

При проверке исключений избегайте «широкого» `catch` в продуктивном коде ради теста. Лучше поднять исключение на поверхность и проверить его напрямую в юнит-тесте. Антипаттерн — «мы не бросаем исключения, мы возвращаем null/Optional.empty», если по домену это именно ошибка. Пусть тест научит код «говорить прямо».

Полезный приём — property-тесты на обратимость. Например, «сериализация → десериализация» возвращает исходное значение, «кодировка → декодировка» сохраняет строку. Такие проверки ловят редко посещаемые края.

При доменной арифметике всегда фиксируйте округление/точность и проверяйте их на границах. Деньги — не числа с плавающей точкой. Тест «0.1 + 0.2 = 0.3» в BigDecimal-мире ожидаем, но в double-мире — нет; юнит-тесты должны закреплять верный выбор типов.

Не стесняйтесь «быстрых» property-циклов прямо в юнит-тесте. Это не заменит профессиональные библиотеки, но добавит уверенности. В отчёте CI видно, сколько итераций прошло, и падения легко реплицируются по seed.

Границы времени и дат тоже стоит проверять: переходы суток, високосные годы, смену месяца. Если домен чувствителен к этим переходам, добавьте целевые проверки на фиксированных датах/часах.

Наконец, фиксируйте семантику сообщений исключений как часть доменного протокола. Это не API для клиента, но это API для оператора и разработчика, читающего логи.

**Java — исключения и property-based цикл**

```java
package properties;

import domain.order.Money;
import domain.order.Order;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.Random;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class BoundaryAndPropertyTest {

    @Test
    void boundaryChecks() {
        var order = new Order(UUID.randomUUID(), "USD");
        assertThatThrownBy(() -> order.addLine("X", new Money("USD", new BigDecimal("1.00")), 0))
            .isInstanceOf(IllegalArgumentException.class).hasMessageContaining("qty");
    }

    @Test
    void property_totalIsSumOfLines_randomizedStable() {
        var rnd = new Random(12345);
        for (int i = 0; i < 200; i++) {
            var order = new Order(UUID.randomUUID(), "USD");
            int lines = rnd.nextInt(5) + 1;
            BigDecimal expected = BigDecimal.ZERO;
            for (int j = 0; j < lines; j++) {
                int qty = rnd.nextInt(3) + 1;
                BigDecimal price = new BigDecimal(String.valueOf(rnd.nextInt(100))) ;
                order.addLine("SKU-"+j, new Money("USD", price), qty);
                expected = expected.add(price.multiply(BigDecimal.valueOf(qty)));
            }
            assertThat(order.total().amount).isEqualByComparingTo(expected);
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package properties

import domain.order.Money
import domain.order.Order
import org.assertj.core.api.Assertions.assertThat
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test
import java.math.BigDecimal
import java.util.Random
import java.util.UUID

class BoundaryAndPropertyTest {

    @Test
    fun boundaryChecks() {
        val order = Order(UUID.randomUUID(), "USD")
        assertThatThrownBy { order.addLine("X", Money("USD", BigDecimal("1.00")), 0) }
            .hasMessageContaining("qty")
    }

    @Test
    fun property_totalIsSumOfLines_randomizedStable() {
        val rnd = Random(12345)
        repeat(200) {
            val order = Order(UUID.randomUUID(), "USD")
            val lines = rnd.nextInt(5) + 1
            var expected = BigDecimal.ZERO
            repeat(lines) { j ->
                val qty = rnd.nextInt(3) + 1
                val price = BigDecimal(rnd.nextInt(100).toString())
                order.addLine("SKU-$j", Money("USD", price), qty)
                expected = expected.add(price.multiply(BigDecimal(qty)))
            }
            assertThat(order.total().amount).isEqualByComparingTo(expected)
        }
    }
}
```

---

## Тестирование конвертеров/мапперов (MapStruct) и валидации (Hibernate Validator)

Конвертеры/мапперы — типичный юнит-уровень. Здесь проверяется соответствие полей между слоями (DTO↔доменные объекты), правила форматирования и вычисляемые атрибуты. В Java-мире популярна **MapStruct** — генератор маппинга compile-time, который превращает декларативный интерфейс в быстрый код без рефлексии. Юнит-тесты для маппера просты: создать входной DTO, вызвать маппер, сравнить с ожидаемой доменной моделью.

Валидация — ещё одна зона ответственности юнит-слоя. **Hibernate Validator** (референс-имплементация Bean Validation, `jakarta.validation`) позволяет описывать ограничения аннотациями (`@NotNull`, `@Size`, `@Pattern`) прямо в DTO/Value-классах. Юнит-тест инициализирует `Validator` без Spring (`Validation.buildDefaultValidatorFactory()`) и проверяет, что корректные объекты проходят, а некорректные — дают ожидаемый набор нарушений.

Проверять маппинг лучше «смыслово», а не «один-к-одному» по всем полям. Если часть полей вычисляются (например, `total`), их можно сравнить через доменную логику (`order.total()`), а не дублировать формулу в тесте. Это снижает риск «тест утверждает сам себя».

MapStruct умеет маппить вложенные объекты, коллекции и перечисления. Если правила сложнее (например, конвертация форматов дат), добавьте методы-helpers в маппер и покройте их отдельными юнит-тестами. Избегайте «магии» с `@AfterMapping`, если можно выразить трансформацию декларативно.

В тестах маппера полезно добавить «негатив»: что будет, если вход содержит `null` или пустые строки? Если по контракту вы ожидаете исключение — тест это фиксирует. Если ожидается «мягкая» деградация (поле остаётся `null`) — это тоже нужная проверка.

Для валидации проверяйте не только наличие нарушений, но и **сообщения/путь** к полю. Это делает тест живым регрессом для локализации и текстов, которые видит пользователь/клиент. Если вы используете локализацию, фиксируйте `Locale.ROOT` на время теста.

MapStruct требует настроить аннотационный процессор в Gradle. Для Kotlin нужен `kapt`. В тестах маппер можно брать через `Mappers.getMapper(...)`, без Spring. Это быстрый и независимый способ.

Валидация иногда зависит от кастомных аннотаций (`@ValidSku`). Их логику тоже лучше покрыть юнит-тестами напрямую для валидатора (без поднятия Spring), чтобы не тянуть контексты ради одного класса.

Наконец, не путайте уровни ответственности: маппер и валидатор не должны лазить в БД или сеть. Если логика «проверить, что sku существует» — это уже доменная/инфраструктурная операция и не должна жить в аннотации «валидации поля» на DTO.

**Gradle (Groovy/Kotlin) — MapStruct и Validator**

```groovy
plugins { id 'java'; id 'org.jetbrains.kotlin.jvm' version '2.0.20' apply false; id 'org.jetbrains.kotlin.kapt' version '2.0.20' apply false }

dependencies {
    implementation "org.mapstruct:mapstruct:1.5.5.Final"
    annotationProcessor "org.mapstruct:mapstruct-processor:1.5.5.Final"

    testImplementation "org.hibernate.validator:hibernate-validator:8.0.1.Final"
    testImplementation "org.glassfish:jakarta.el:4.0.2"
}
```

```kotlin
plugins {
    java
    kotlin("jvm") version "2.0.20" apply false
    kotlin("kapt") version "2.0.20" apply false
}
dependencies {
    implementation("org.mapstruct:mapstruct:1.5.5.Final")
    annotationProcessor("org.mapstruct:mapstruct-processor:1.5.5.Final")

    testImplementation("org.hibernate.validator:hibernate-validator:8.0.1.Final")
    testImplementation("org.glassfish:jakarta.el:4.0.2")
}
```

**Java — DTO, MapStruct маппер и юнит-тест**

```java
package mapping;

import org.mapstruct.*;
import org.mapstruct.factory.Mappers;

import java.math.BigDecimal;
import java.util.UUID;

public class Dto {
    public UUID id;
    public String title;
    public String price; // "12.34"
}

class Model {
    public final UUID id;
    public final String title;
    public final BigDecimal price;
    public Model(UUID id, String title, BigDecimal price) {
        this.id = id; this.title = title; this.price = price;
    }
}

@Mapper
interface DtoMapper {
    DtoMapper INSTANCE = Mappers.getMapper(DtoMapper.class);

    @Mapping(target = "price", expression = "java(new java.math.BigDecimal(dto.price))")
    Model toModel(Dto dto);
}
```

```java
package mapping;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class DtoMapperTest {

    @Test
    void mapsFieldsAndConvertsPrice() {
        var dto = new Dto();
        dto.id = UUID.randomUUID();
        dto.title = "Book";
        dto.price = "12.34";

        var model = DtoMapper.INSTANCE.toModel(dto);

        assertThat(model.id).isEqualTo(dto.id);
        assertThat(model.title).isEqualTo("Book");
        assertThat(model.price).isEqualByComparingTo(new BigDecimal("12.34"));
    }
}
```

**Kotlin — DTO, MapStruct (через kapt) и тест**

> Для Kotlin-модуля подключите плагин `kotlin-kapt` и используйте Java-интерфейс маппера (MapStruct генерирует Java-код, который отлично вызывается из Kotlin).

```kotlin
package mapping

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.math.BigDecimal
import java.util.UUID

data class KDto(var id: UUID? = null, var title: String? = null, var price: String? = null)

class KModel(val id: UUID, val title: String, val price: BigDecimal)

class KotlinMapperUsageTest {

    @Test
    fun mapsWithJavaMapper() {
        val dto = KDto(UUID.randomUUID(), "Book", "12.34")
        val model = DtoMapper.INSTANCE.toModel(Dto().apply {
            id = dto.id; title = dto.title; price = dto.price
        })
        assertThat(model.title).isEqualTo("Book")
        assertThat(model.price).isEqualByComparingTo(BigDecimal("12.34"))
    }
}
```

**Java — Bean Validation без Spring**

```java
package validation;

import jakarta.validation.*;
import jakarta.validation.constraints.*;

import org.junit.jupiter.api.Test;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class Signup {
    @NotBlank public String email;
    @NotBlank @Size(min = 8) public String password;
}

public class ValidatorTest {

    private final Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

    @Test
    void validatesConstraints() {
        var s = new Signup();
        s.email = "";
        s.password = "short";
        Set<ConstraintViolation<Signup>> violations = validator.validate(s);
        assertThat(violations).extracting(ConstraintViolation::getPropertyPath)
            .anySatisfy(p -> assertThat(p.toString()).isEqualTo("email"));
        assertThat(violations).anyMatch(v -> v.getPropertyPath().toString().equals("password"));
    }
}
```

**Kotlin — Bean Validation**

```kotlin
package validation

import jakarta.validation.Validation
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Size
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class Signup(
    @field:NotBlank var email: String? = null,
    @field:NotBlank @field:Size(min = 8) var password: String? = null
)

class ValidatorTest {

    private val validator = Validation.buildDefaultValidatorFactory().validator

    @Test
    fun validatesConstraints() {
        val s = Signup(email = "", password = "short")
        val violations = validator.validate(s)
        assertThat(violations.map { it.propertyPath.toString() })
            .contains("email", "password")
    }
}
```

---

## Контроль времени/UUID: абстракции Clock/IdGenerator

Время и идентификаторы — частые источники недетерминизма. Когда метод обращается к `Instant.now()` или `UUID.randomUUID()` прямо, тест становится «случайным»: значение меняется каждый запуск, снапшоты не совпадают, а граничные кейсы «в полночь» недоступны. Решение — прокинуть абстракции: `Clock` (из `java.time`) и свой интерфейс `IdGenerator`.

`Clock` позволяет заморозить «текущее время» на фиксированном мгновении. В проде используйте `Clock.systemUTC()`, в тесте — `Clock.fixed(Instant.parse("..."), ZoneOffset.UTC)`. Все операции времени должны зависеть от переданного `Clock`, включая вычисления TTL, дедлайнов, «истёкших» статусов.

`IdGenerator` — тонкая обёртка над UUID-генерацией. Определите интерфейс `IdGenerator { UUID next(); }`, внедряйте его в доменные сервисы, а в тестах подавайте фейковую реализацию, возвращающую константу или детерминированную последовательность. Это очистит логи и сделает возможными «golden files».

Такие абстракции — не «инфраструктурный мусор», а часть доменного контракта. Если операция зависит от времени — это часть её бизнес-смысла, и его нужно контролировать в тесте. Но не усложняйте: не везде надо протаскивать `Clock`. Если метод не чувствителен ко времени — не добавляйте зависимость.

Абстракции особенно важны для подписей/токенов. Если вы формируете HMAC со временем/nonce, фиксированный `Clock` и `IdGenerator` позволяют вычислять предсказуемый результат и сравнивать его с «золотым» эталоном.

Следите за тем, чтобы «текущее время» не просачивалось через статические вызовы. Статический `Instant.now()` в одном месте ломает всю идею. Линтеры/ревью помогают, но проще договориться командой: «в домене время — только через `Clock`».

Если требуется «текущий день/час» с округлением, инкапсулируйте это в helper, принимающий `Clock`. Тесты покрывают и обычный день, и края (например, 23:59:59 → 00:00:00). Это снимет классические инциденты «в полночь сломалось».

Для UUID кроме детерминизма иногда важен формат/канонизация (строка в нижнем регистре, без фигурных скобок). Пишите тонкие утилиты с тестами вместо того, чтобы полагаться на неявные преобразования.

Наконец, если вам нужно время в миллисекундах «как в проде», а тест слишком медленный — рассмотрите `Clock.offset(fixed, Duration.ofMillis(...))` и шаговую симуляцию времени. Это даёт быстрые и аккуратные сценарии.

**Java — использование Clock и IdGenerator**

```java
package timeid;

import java.time.*;
import java.util.UUID;

public final class Tokens {
    private final Clock clock;
    private final IdGenerator ids;

    public interface IdGenerator { UUID next(); }

    public Tokens(Clock clock, IdGenerator ids) {
        this.clock = clock; this.ids = ids;
    }

    public String issueToken() {
        var id = ids.next();
        var ts = OffsetDateTime.now(clock).toString();
        return id + ":" + ts;
    }
}
```

```java
package timeid;

import org.junit.jupiter.api.Test;
import java.time.*;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class TokensTest {

    @Test
    void deterministicToken() {
        Clock fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
        Tokens.IdGenerator ids = () -> UUID.fromString("00000000-0000-0000-0000-000000000000");
        var t = new Tokens(fixed, ids);
        assertThat(t.issueToken()).isEqualTo("00000000-0000-0000-0000-000000000000:2025-01-01T00:00Z");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package timeid

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.time.Clock
import java.time.Instant
import java.time.OffsetDateTime
import java.time.ZoneOffset
import java.util.UUID

class Tokens(private val clock: Clock, private val ids: IdGenerator) {
    fun interface IdGenerator { fun next(): UUID }
    fun issueToken(): String {
        val id = ids.next()
        val ts = OffsetDateTime.now(clock).toString()
        return "$id:$ts"
    }
}

class TokensTest {

    @Test
    fun deterministicToken() {
        val fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC)
        val ids = Tokens.IdGenerator { UUID.fromString("00000000-0000-0000-0000-000000000000") }
        val t = Tokens(fixed, ids)
        assertThat(t.issueToken()).isEqualTo("00000000-0000-0000-0000-000000000000:2025-01-01T00:00Z")
    }
}
```

---

## Быстрые проверки конкурентности: JMH-like micro, ReentrantLock/atomic сценарии

Конкурентность сложно полноценно тестировать юнит-уровнем, но можно быстро проверить базовые гарантии: «наш счётчик потокобезопасен», «блокировка действительно сериализует доступ», «наши атомики не теряют инкременты». Такие тесты — не бенчмарки, а sanity-checks, которые ловят грубые ошибки (например, использование `ArrayList` из нескольких потоков без синхронизации).

Мини-«JMH-подобные» проверки можно построить на `ExecutorService` с `CountDownLatch` и фиксированным количеством итераций. Идея: запустить N потоков, синхронно «выстрелить стартовым пистолетом», выполнить M инкрементов/операций и проверить итоговое состояние. Тест должен занимать миллисекунды/десятки миллисекунд и быть полностью детерминированным.

`ReentrantLock` полезен, когда нужна явная критическая секция. Юнит-тест проверяет, что в отсутствие блокировки мы теряем инкременты (на «сыром» int), а с блокировкой/атомиком — нет. Это учебная, но показательная проверка. При этом важно не зациклиться на `Thread.sleep`: используйте `latch.await()` и короткие таймауты.

`Atomic*` классы (например, `AtomicInteger`, `AtomicReference`) обеспечивают lock-free операции с семантикой CAS. Тест проверяет, что `incrementAndGet`/`updateAndGet` дают ожидаемый результат под конкуренцией. Не пытайтесь «поймать» межпоточные гонки вероятностно; лучше провоцировать их массой итераций и единовременным стартом потоков.

Для структур данных применяйте «арбитражные» проверки: например, одновременная запись N уникальных значений в `ConcurrentHashMap` должна дать ровно N записей. Это простая, но информативная sanity-метрика. С `ArrayList` без синхронизации вы гарантированно увидите меньше N или получите `ArrayIndexOutOfBoundsException`.

Тесты конкурентности должны быть изолированы по времени. Дайте им JUnit-тег `component`/`fast` и запускайте вместе с юнитами, но следите, чтобы они не выходили за 100–200 мс. Если дольше — перенесите их в «медиум» набор. Никаких «секунду ждём» без причины.

Не используйте случайные числа для количества потоков/итераций. Фиксируйте `threads=8`, `iterations=1000` и делайте их параметрами метода/класса, чтобы легко было «подкручивать» при расследовании. Но избегайте больших значений — тесты должны быть быстрыми.

При необходимости проверяйте «взаимное исключение» у блокировки через захват времени/счётчика в критической секции: если два потока «одновременно» исполняют секцию, инвариант нарушится (например, обнаружится перекрытие времени). Это избыточно для большинства модулей, но полезно в учебных утилитах синхронизации.

Наконец, помните, что такие тесты не заменяют нагрузочное тестирование и профилирование. Они ловят грубые ошибки проектирования и помогают спать спокойнее, но за реальными гарантиями производительности — к JMH и staging-нагрузке.

**Java — быстрые проверки atomic vs без синхронизации и ReentrantLock**

```java
package concurrency;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class ConcurrencySanityTest {

    @Test
    void atomicCounterDoesNotLoseIncrements() throws Exception {
        int threads = 8, iters = 1000;
        var pool = Executors.newFixedThreadPool(threads);
        var start = new CountDownLatch(1);
        var done = new CountDownLatch(threads);
        AtomicInteger counter = new AtomicInteger(0);

        for (int t = 0; t < threads; t++) {
            pool.submit(() -> {
                try {
                    start.await();
                    for (int i = 0; i < iters; i++) counter.incrementAndGet();
                } catch (InterruptedException ignored) { Thread.currentThread().interrupt(); }
                finally { done.countDown(); }
            });
        }
        start.countDown();
        done.await(1, TimeUnit.SECONDS);
        pool.shutdownNow();

        assertThat(counter.get()).isEqualTo(threads * iters);
    }

    @Test
    void unsynchronizedArrayListLosesElements() throws Exception {
        int threads = 8, iters = 1000;
        var list = new ArrayList<Integer>();
        var pool = Executors.newFixedThreadPool(threads);
        var start = new CountDownLatch(1);
        var done = new CountDownLatch(threads);

        for (int t = 0; t < threads; t++) {
            final int base = t * iters;
            pool.submit(() -> {
                try {
                    start.await();
                    for (int i = 0; i < iters; i++) list.add(base + i);
                } catch (InterruptedException ignored) { Thread.currentThread().interrupt(); }
                finally { done.countDown(); }
            });
        }
        start.countDown();
        done.await(1, TimeUnit.SECONDS);
        pool.shutdownNow();

        // Примерно, но почти всегда < threads*iters из-за гонок; демонстрация анти-паттерна
        assertThat(list.size()).isLessThan(threads * iters);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package concurrency

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import java.util.ArrayList
import java.util.concurrent.CountDownLatch
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit
import java.util.concurrent.atomic.AtomicInteger

class ConcurrencySanityTest {

    @Test
    fun atomicCounterDoesNotLoseIncrements() {
        val threads = 8; val iters = 1000
        val pool = Executors.newFixedThreadPool(threads)
        val start = CountDownLatch(1)
        val done = CountDownLatch(threads)
        val counter = AtomicInteger(0)

        repeat(threads) {
            pool.submit {
                try {
                    start.await()
                    repeat(iters) { counter.incrementAndGet() }
                } finally {
                    done.countDown()
                }
            }
        }
        start.countDown()
        done.await(1, TimeUnit.SECONDS)
        pool.shutdownNow()

        assertThat(counter.get()).isEqualTo(threads * iters)
    }

    @Test
    fun unsynchronizedArrayListLosesElements() {
        val threads = 8; val iters = 1000
        val list = ArrayList<Int>()
        val pool = Executors.newFixedThreadPool(threads)
        val start = CountDownLatch(1)
        val done = CountDownLatch(threads)

        repeat(threads) { t ->
            val base = t * iters
            pool.submit {
                try {
                    start.await()
                    repeat(iters) { list.add(base + it) }
                } finally { done.countDown() }
            }
        }
        start.countDown()
        done.await(1, TimeUnit.SECONDS)
        pool.shutdownNow()

        assertThat(list.size).isLessThan(threads * iters)
    }
}
```

# 4. Срезы Spring: быстрые компонентные тесты

## @WebMvcTest, @WebFluxTest, @DataJpaTest, @JdbcTest, @JsonTest, @RestClientTest — назначение

Срезы — это «минимальные» автоконфигурации Spring Boot для тестирования конкретного слоя приложения. Идея проста: поднять только то, что реально нужно для сценария, и не тащить остальной контекст. В результате вы получаете быстрый, детерминированный тест, который проверяет реальную интеграцию внутри выбранного слоя (например, контроллер ↔ биндинг ↔ валидация ↔ сериализация), не платя за подъем БД, брокеров и прочего окружения.

`@WebMvcTest` предназначен для тестирования MVC-контроллеров на сервлетном стеке. Он поднимает Spring MVC, конвертеры сообщений, валидацию, `ControllerAdvice`, фильтры безопасности (если не отключить), но не поднимает сервисный слой и репозитории. Это идеальный инструмент, чтобы проверить роутинг, биндинг параметров, сериализацию/десериализацию и формат ошибок, не касаясь базы данных.

`@WebFluxTest` выполняет ту же роль для реактивного стека WebFlux. Он поднимает `WebFluxConfigurer`, `WebTestClient` и реактивные конвертеры. Если ваш сервис на WebFlux, тестируйте контроллеры и фильтры через этот срез: вы получите быстрые проверки `Mono`/`Flux`, SSE и особенностей реактивного биндинга, не загружая Netty-сервер целиком.

`@DataJpaTest` поднимает JPA-слой: `EntityManagerFactory`, Hibernate, `DataSource` (обычно встроенный), транзакции и сканирует репозитории Spring Data. Это удобный способ проверить запросы, маппинг сущностей, поведение транзакций и валидацию на уровне JPA-аннотаций. По умолчанию тесты откатываются транзакционно, что делает их независимыми.

`@JdbcTest` — лёгкий срез для JDBC без JPA. Он конфигурирует `DataSource`, `JdbcTemplate` и транзакции, но не поднимает Hibernate. Применяйте его, когда у вас чистый SQL/Stored Procedures или вы хотите протестировать конкретные SQL-скрипты/DAO, не касаясь ORM.

`@JsonTest` поднимает только Jackson (и JSON-B, если нужно), обнаруживает ваши модули и кастомизации `ObjectMapper`. Это быстрый способ проверить, как сериализуются/десериализуются DTO, как работают `JavaTimeModule`, Kotlin-модуль, настройки `PropertyNamingStrategy` и флаги наподобие `FAIL_ON_UNKNOWN_PROPERTIES`.

`@RestClientTest` — срез для тестирования исходящих HTTP-клиентов. Он конфигурирует `RestTemplateBuilder`/`RestClient`, поднимает встроенный `MockRestServiceServer` и позволяет детерминированно задавать ожидания по внешним вызовам: URL, метод, заголовки, тело, статус и даже задержки. Пользуйтесь им для клиентов к чужим API — это дешевле и надёжнее, чем гонять реальный HTTP.

Сочетая срезы, вы покрываете 80% интеграционных сценариев очень быстро. Контроллеры проверяются через `@WebMvcTest`/`@WebFluxTest`, сериализация — через `@JsonTest`, запросы к БД — через `@DataJpaTest`/`@JdbcTest`, исходящие интеграции — через `@RestClientTest`. Полный `@SpringBootTest` оставьте на сценарии «склейки» всего приложения и энд-ту-энд.

Важно помнить, что срез — не «мок-тест»: это настоящий Spring на определённом уровне. Вы получаете пользу автоконфигурации, формат ошибок, конвертеры и фильтры, как в бою. Но зависимости нижних слоёв придётся предоставить явно: либо заглушить, либо импортировать.

Скорость — главный выигрыш. Типичный `@WebMvcTest` стартует за десятки миллисекунд, в то время как `@SpringBootTest` — за секунды. В сумме на проекте это означает минуту против 10–15 минут в CI, а значит — быструю обратную связь и меньше «красноты» от нестабильного окружения.

Если вы только начинаете структурировать набор, начните именно со срезов. Для каждого контроллера — пара тестов через `@WebMvcTest`. Для каждого репозитория — по тесту через `@DataJpaTest`. Для каждого клиента — по тесту через `@RestClientTest`. Уже это даст высокий ROI и дисциплинирует архитектуру.

**Gradle (Groovy) — базовые зависимости с web и webflux для срезов**

```groovy
dependencies {
    testImplementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation "org.springframework.boot:spring-boot-starter-webflux"
    testImplementation "org.springframework.boot:spring-boot-starter-test"
}
```

**Gradle (Kotlin) — эквивалент**

```kotlin
dependencies {
    testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — @WebMvcTest для контроллера**

```java
package slices.mvc;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
@RequestMapping("/api/hello")
class HelloController {
    @GetMapping public Greeting hello() { return new Greeting("hello"); }
    record Greeting(String message){}
}

@WebMvcTest(controllers = HelloController.class)
class HelloControllerMvcSliceTest {

    @Autowired MockMvc mvc;

    @Test
    void returnsHelloJson() throws Exception {
        mvc.perform(get("/api/hello"))
           .andExpect(status().isOk())
           .andExpect(content().json("{\"message\":\"hello\"}"));
    }
}
```

**Kotlin — @WebFluxTest для реактивного контроллера**

```kotlin
package slices.webflux

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import javax.annotation.Resource
import reactor.core.publisher.Mono

@RestController
@RequestMapping("/rx/hello")
class RxHelloController {
    data class Greeting(val message: String)
    @GetMapping fun hello(): Mono<Greeting> = Mono.just(Greeting("hello"))
}

@WebFluxTest(controllers = [RxHelloController::class])
class RxHelloControllerSliceTest {

    @Resource
    lateinit var client: WebTestClient

    @Test
    fun returnsHelloJson() {
        client.get().uri("/rx/hello")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .json("""{"message":"hello"}""")
    }
}
```

---

## Подтягивание необходимых бинов: @Import, @MockBean, фильтры компонентов

Когда вы используете срез, нижние зависимости контроллеров/клиентов не поднимаются автоматически. Это правильно с точки зрения изоляции, но требует вручную предоставить то, на что ссылается код. Делать это можно разными способами, и у каждого — свои эффекты. В простом случае достаточно замокать зависимость через `@MockBean`, и Spring положит её в контекст вместо реальной реализации.

`@MockBean` создаёт Mockito-мок и регистрирует его как бин нужного типа. Контроллер получит его через DI, а вы — возможность программировать ответы и верифицировать вызовы. Такой подход минимален и хорошо подходит для «опорных» тестов контроллера, где важны статус/формат ответа, а не бизнес-логика сервиса.

`@Import` позволяет добавить в срез небольшой кусок реальной конфигурации: хелперы, конвертеры, мапперы, `Clock`, кастомный `ObjectMapper`. Это полезно, когда сложнее имитировать поведение моками, чем просто импортировать маленький бин. Важно не «перетащить» весь слой — придерживайтесь принципа «минимально необходимого».

Фильтры компонентов (`includeFilters`/`excludeFilters`) на аннотациях среза дают ещё более тонкий контроль. Например, в `@WebMvcTest` можно явно включить конкретный `ControllerAdvice` или исключить «шумный» фильтр. Это помогает сдерживать размер контекста и избегать побочных эффектов.

Есть и более мощные варианты: `@TestConfiguration` внутри теста для объявления тестовых бинов и `@ImportAutoConfiguration` для явного подключения нужных автоконфигураций. Пользуйтесь ими, когда простых `@MockBean`/`@Import` недостаточно, но помните — вы увеличиваете площадь покрытия контекста.

Срезы дружат с Spring Security, и зачастую нужно «подложить» свою конфигурацию цепочки фильтров. Это делается либо импортом вашего `SecurityFilterChain`, либо моками `JwtDecoder`/`OpaqueTokenIntrospector` для аутентификации. Главное — не тянуть лишнее: для проверки контроллера вполне достаточно «минимальной» безопасности.

Если вы тестируете контроллер с валидацией, убедитесь, что бин `Validator` доступен. Срезы его поднимают, но если вы импортируете собственную конфигурацию, случайно не перекройте её. Потребуется `spring-boot-starter-validation` в зависимостях, чтобы `MethodValidationPostProcessor` оказался в контексте.

Иногда нужна частичная реальность: мок-сервис + реальный маппер/валидация. Это хороший компромисс. Не стесняйтесь комбинировать, если это делает тест короче и понятнее. Но всегда держите в голове границу среза: не превратите его в «второй прод».

Важный анти-паттерн — мокать то, что вы хотите протестировать. Если цель — проверить `ControllerAdvice`, не подменяйте его; подключите его через `@Import` и проверяйте реальное поведение. Моки — для внешних зависимостей слоя, а не для самого слоя.

Наконец, складывайте повторяющиеся паттерны в абстракции. Базовый класс для `@WebMvcTest` с общими `@MockBean`/`@Import` и построителем `MockMvc` сокращает копипасту и стабилизирует стиль тестов.

**Java — @WebMvcTest с @MockBean и @Import**

```java
package slices.beans;

import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.web.bind.annotation.*;

import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

// Сервис, который мы замокаем
interface GreetingService { String greet(String name); }

@RestController
@RequestMapping("/api/greet")
class GreetController {
    private final GreetingService service;
    GreetController(GreetingService service) { this.service = service; }
    @GetMapping public Message greet(@RequestParam String name) { return new Message(service.greet(name)); }
    record Message(String text) {}
}

// Доп. бин, который хотим импортировать
class Slug {
    String of(String s) { return s.trim().toLowerCase().replaceAll("\\s+","-"); }
}

@Import(GreetSliceTest.SliceConfig.class)
@WebMvcTest(controllers = GreetController.class)
class GreetSliceTest {

    static class SliceConfig {
        @Bean Slug slug() { return new Slug(); }
    }

    @Autowired MockMvc mvc;
    @MockBean GreetingService service;

    @Test
    void usesMockServiceAndImportsHelper() throws Exception {
        given(service.greet("Alice")).willReturn("Hello, Alice");
        mvc.perform(get("/api/greet").param("name","Alice"))
           .andExpect(status().isOk())
           .andExpect(content().json("{\"text\":\"Hello, Alice\"}"));
        Mockito.verify(service).greet("Alice");
    }
}
```

**Kotlin — эквивалент с @MockBean и @Import**

```kotlin
package slices.beans

import org.junit.jupiter.api.Test
import org.mockito.BDDMockito.given
import org.mockito.Mockito.verify
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Import
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController

interface GreetingService { fun greet(name: String): String }

@RestController
@RequestMapping("/api/greet")
class GreetController(private val service: GreetingService) {
    data class Message(val text: String)
    @GetMapping fun greet(@RequestParam name: String) = Message(service.greet(name))
}

class Slug { fun of(s: String) = s.trim().lowercase().replace("\\s+".toRegex(), "-") }

@Configuration
class SliceConfig { @Bean fun slug() = Slug() }

@WebMvcTest(controllers = [GreetController::class])
@Import(SliceConfig::class)
class GreetSliceTest {

    @Autowired lateinit var mvc: MockMvc
    @MockBean lateinit var service: GreetingService

    @Test
    fun usesMockServiceAndImportsHelper() {
        given(service.greet("Alice")).willReturn("Hello, Alice")
        mvc.get("/api/greet") { param("name","Alice") }
            .andExpect { status { isOk() } }
            .andExpect { content { json("""{"text":"Hello, Alice"}""") } }
        verify(service).greet("Alice")
    }
}
```

---

## Настройка сериализации и Jackson модули (JavaTime), валидация

Сериализация — часть контракта. От того, как вы выдаёте даты, `null`-поля и имена свойств, зависит совместимость с клиентами. Срез `@JsonTest` позволяет поднять только Jackson и проверить эти правила за миллисекунды. Он автоматически подцепит `jackson-datatype-jsr310` (JavaTime) и `jackson-module-kotlin`, если они есть в зависимостях, а также вашу кастомизацию `ObjectMapper`.

Если вы настраиваете `ObjectMapper` через `Jackson2ObjectMapperBuilderCustomizer`, его удобно импортировать в `@JsonTest` и проверить эффекты: `WRITE_DATES_AS_TIMESTAMPS=false`, `PropertyNamingStrategy.SNAKE_CASE`, видимость полей/геттеров. Такой тест — «якорь» против случайных регрессов при обновлении зависимостей.

С датами важно быть явными. Проверяйте, что `OffsetDateTime` сериализуется в ISO 8601 с таймзоной, а `LocalDate` — без неё. Если у вас в контракте UTC, настраивайте `SerializationFeature.WRITE_DATES_WITH_ZONE_ID=false` и проверяйте, что формат стабилен.

Валидация Bean Validation интегрируется с Jackson: аннотации на DTO будут проверяться при биндинге запроса/ответа. В `@WebMvcTest` достаточно иметь `spring-boot-starter-validation` в зависимостях, чтобы `@Valid` на параметрах контроллера работал и выдавал `MethodArgumentNotValidException`. Проверьте, что ваш `@ControllerAdvice` переводит это в единый формат ошибок.

Отдельно тестируйте «маскирование» полей: `@Schema(accessMode = WRITE_ONLY/READ_ONLY)` в OpenAPI не влияет на реальную сериализацию. Если поле должно быть только на запись, настройте `@JsonProperty(access = Access.WRITE_ONLY)` и подтвердите это тестом. Такие тонкости часто ломаются незаметно.

Котлин-модуль Jackson важен для data-классов и `nullability`. Если у вас Kotlin DTO, без этого модуля Jackson будет удивлять. Срез `@JsonTest` покажет, всё ли подцепилось, и даст быстрый фидбек на регрессии конфигурации.

Сообщения валидации зависят от `MessageSource`. Если вы локализуете ошибки, в `@WebMvcTest` можно импортировать тестовый `MessageSource` и проверить текст. Но чаще достаточно проверить коды полей/типов нарушений, чтобы тест оставался стабильным при правках текстов.

И наконец, не проверяйте «всё подряд». Закрепите ключевые правила: имена, даты, обязательные поля, поведение `null`, `unknown properties`. Остальное лучше покрыть интеграционными контрактными тестами, чтобы не плодить хрупкость.

**Gradle (оба DSL) — модули Jackson и валидация**

```groovy
dependencies {
    implementation "com.fasterxml.jackson.datatype:jackson-datatype-jsr310"
    implementation "com.fasterxml.jackson.module:jackson-module-kotlin"
    implementation "org.springframework.boot:spring-boot-starter-validation"
    testImplementation "org.springframework.boot:spring-boot-starter-test"
}
```

```kotlin
dependencies {
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — @JsonTest с кастомизацией ObjectMapper и проверкой дат**

```java
package slices.json;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.beans.factory.annotation.Autowired;

import java.time.OffsetDateTime;
import java.time.ZoneOffset;

import static org.assertj.core.api.Assertions.assertThat;

record Event(String name, OffsetDateTime at) {}

@JsonTest
class JsonSliceTest {

    @Autowired ObjectMapper om;

    @Test
    void serializesOffsetDateTimeIso() throws Exception {
        om.registerModule(new JavaTimeModule());
        om.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        String json = om.writeValueAsString(new Event("deploy", OffsetDateTime.of(2025,1,1,0,0,0,0, ZoneOffset.UTC)));
        assertThat(json).contains("\"at\":\"2025-01-01T00:00:00Z\"");
    }
}
```

**Kotlin — @WebMvcTest проверка валидации и Problem Details**

```kotlin
package slices.json

import jakarta.validation.constraints.Min
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.http.ProblemDetail
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController
import org.springframework.test.web.servlet.MockMvc
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.test.web.servlet.get

@RestController
@Validated
class PriceController {
    data class Price(val cents: Int)
    @GetMapping("/price") fun price(@RequestParam @Min(1) qty: Int) = Price(qty * 10)
}

@WebMvcTest(controllers = [PriceController::class])
class ValidationSliceTest(@Autowired val mvc: MockMvc) {

    @Test
    fun returnsProblemJsonOnViolation() {
        mvc.get("/price") { param("qty","0") }
            .andExpect { status { isBadRequest() } }
            .andExpect { header { string("Content-Type","application/problem+json") } }
    }
}
```

---

## Тестирование контроллеров: ошибки, биндинг, CSRF/фильтры без полного контекста

Контроллер — входная дверь сервиса. Его корректность — это не только «200 OK», но и правильные ошибки, валидация, биндинг параметров и безопасность. `@WebMvcTest` позволяет проверить всё это быстро, запуская всю цепочку фильтров, конвертеров и `ControllerAdvice` без базы и сервисного слоя. Это тот случай, когда «именно слой» важнее «всего приложения».

Ошибки лучше стандартизировать через `@ControllerAdvice`. В срезе его можно подключить либо явным указанием в `@WebMvcTest(controllers=..., excludeFilters=...)` и `@Import`, либо через пакетное сканирование, если он в том же пакете. В тесте проверьте статус, `Content-Type`, обязательные поля ответа и ключевые фрагменты сообщений — этого достаточно, чтобы держать контракт.

Биндинг параметров — зона множества багов. Проверьте path/query/header-параметры, кастомные конвертеры типов, формат дат/денег. Убедитесь, что отсутствие обязательного параметра приводит к предсказуемой ошибке, а «лишние» параметры не ломают сериализацию. Это особенно критично при миграциях с MVC на WebFlux и обратно.

CSRF и цепочка фильтров безопасности включаются автоматически в `@WebMvcTest`, если у вас есть Spring Security на classpath. Для методов, требующих защищённого POST/PUT/DELETE, используйте `with(csrf())` из Spring Security Test. Для аутентификации — `@WithMockUser` или `SecurityMockMvcRequestPostProcessors.jwt()` при JWT-сценариях. Это даёт «как в бою», но без настоящего IdP.

Файловые загрузки и multipart — классический край. В срезе их легко проверить через `mockMvc.perform(multipart(...))` с явной передачей имени поля, контента и заголовков. Проверьте ограничения размера и тип контента — они часто отличаются между Dev/Prod конфигурациями.

Кэширование, CORS и ETag — тоже ответственность веб-слоя. В срезе их можно проверить, импортировав соответствующие конфигурации. Важно зафиксировать, что `Cache-Control` соответствует ожиданиям, а CORS-заголовки не «открывают» лишнего.

Ошибки сериализации стоит проверять как негатив: «плохой JSON» → «400», «неверный тип» → «400 с описанием поля». Это ловит регрессии в конфигурации `ObjectMapper` и предотвращает «500» на банальном неверном входе.

Если у вас есть фильтры-логгеры/корреляция (`MDC`), проверьте, что они ставят заголовки и не ломают тело. В срезе это просто: импортируете фильтр, вызываете эндпоинт и смотрите заголовки/логи.

И наконец, помните: контроллер — тонкий слой. Если тест «упирается» в бизнес-логику, заглушите сервис и проверяйте только веб-аспект. Это удержит тест быстрым и стабильным.

**Gradle (оба DSL) — добавим Spring Security Test**

```groovy
dependencies {
    testImplementation "org.springframework.security:spring-security-test"
}
```

```kotlin
dependencies {
    testImplementation("org.springframework.security:spring-security-test")
}
```

**Java — @WebMvcTest: ошибки, биндинг, CSRF**

```java
package slices.web;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.Min;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
@Validated
class OrdersController {
    record Create(int qty) {}
    @PostMapping("/orders") public String create(@RequestParam @Min(1) int qty){ return "ok"; }
}

@RestControllerAdvice
class Errors {
    @ExceptionHandler(Exception.class) ProblemDetail on(Exception e){ return ProblemDetail.forStatus(400); }
}

@WebMvcTest(controllers = {OrdersController.class, Errors.class})
class OrdersControllerSliceTest {

    @Autowired MockMvc mvc;

    @Test
    @WithMockUser
    void rejectsBadQtyAndRequiresCsrf() throws Exception {
        mvc.perform(post("/orders").param("qty","0").with(csrf()))
           .andExpect(status().isBadRequest())
           .andExpect(header().string("Content-Type","application/problem+json"));

        mvc.perform(post("/orders").param("qty","1")) // без CSRF
           .andExpect(status().isForbidden());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package slices.web

import jakarta.validation.constraints.Min
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.http.ProblemDetail
import org.springframework.security.test.context.support.WithMockUser
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.post
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*
import javax.annotation.Resource
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf

@RestController
@Validated
class OrdersController {
    @PostMapping("/orders")
    fun create(@RequestParam @Min(1) qty: Int) = "ok"
}

@RestControllerAdvice
class Errors {
    @ExceptionHandler(Exception::class)
    fun on(e: Exception): ProblemDetail = ProblemDetail.forStatus(400)
}

@WebMvcTest(controllers = [OrdersController::class, Errors::class])
class OrdersControllerSliceTest {

    @Resource lateinit var mvc: MockMvc

    @Test
    @WithMockUser
    fun rejectsBadQtyAndRequiresCsrf() {
        mvc.post("/orders") {
            param("qty","0")
            with(csrf())
        }.andExpect { status { isBadRequest() } }
         .andExpect { header { string("Content-Type","application/problem+json") } }

        mvc.post("/orders") {
            param("qty","1")
        }.andExpect { status { isForbidden() } }
    }
}
```

---

## Ограничения срезов: что не доступно и как добавить минимально

Срезы поднимают не всё. В `@WebMvcTest` недоступны сервисы и репозитории, в `@DataJpaTest` — ваш бизнес-код, в `@RestClientTest` — веб-слой и безопасность. Это намеренная изоляция, но иногда вам нужно «чуть больше», чтобы тест был осмысленным. Ключ — добавлять минимально необходимое и явно.

Если контроллер зависит от сервиса, замокайте его через `@MockBean`. Это простой случай. Если контроллер зависит от маппера/хелпера, который сам по себе небольшой и не тянет инфраструктуру, импортируйте реальный бин через `@Import`. Так вы избежите теста «поверх моков моков».

В `@DataJpaTest` по умолчанию используется встроенная БД. Если вы хотите свою (Postgres), включите Testcontainers и пробросьте свойства через `@DynamicPropertySource`. Но не тащите весь `@SpringBootTest`: срез сам по себе поддерживает транзакции и откаты, и такие тесты останутся быстрыми.

Некоторые автоконфигурации отключены в срезах. Например, в `@WebMvcTest` не поднимается `ProblemDetailsExceptionHandler` из вашего кода, если он вне пакета контроллеров. Добавьте его через `@Import`. Аналогично для `ConversionService`/`Formatter` — импортируйте конфигурацию, если тест зависит от кастомного форматирования.

`@RestClientTest` поднимает `MockRestServiceServer` для `RestTemplate`/`RestClient`, но если у вас `WebClient`, лучше использовать `@AutoConfigureWebClient` или вручную создавать `WebClient` с `ExchangeFunction`, подменённой на тестовую. Не пытайтесь «поднимать» весь WebFlux ради одного клиента.

Фильтры компонентов (`excludeFilters`) помогают отрезать «шумные» вещи, которые срез всё-таки подхватывает, — например, тяжёлые `@Configuration` из того же пакета. Это особенно важно в монорепозиториях, где пакеты широко пересекаются.

Осторожно с `@DirtiesContext`: он инвалидирует кэш контекста и замедляет прогон. В срезах почти всегда можно избежать грязного состояния, используя моки и локальные фикстуры данных. Если всё же нужно «перезапустить» контекст, применяйте точечно.

Если требуется кастомный `ObjectMapper` с особой модульной конфигурацией, определите `@TestConfiguration` внутри теста и предоставьте бин. Важно не нарушить ожидания других тестов в классе. Тестовая конфигурация «видна» только в этом контексте.

И наконец, помните о границах ответственности. Если «минимально» внезапно превращается в «почти всё приложение», вы, вероятно, выбрали не тот инструмент. Перейдите на `@SpringBootTest` для этого сценария и держите срезы для оставшихся.

**Java — @DataJpaTest c Postgres Testcontainers и минимальными настройками**

```java
package slices.jpa;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.springframework.beans.factory.annotation.Autowired;
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@DataJpaTest
class UserRepoSliceTest {

    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");
    static { pg.start(); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired UserRepository repo;

    @Test
    void savesAndFinds() {
        var u = repo.save(new User(null, "bob"));
        assertThat(repo.findById(u.id)).isPresent();
    }

    @Entity(name="users")
    static class User {
        @Id @GeneratedValue(strategy=GenerationType.IDENTITY) Long id;
        String name;
        public User() {}
        public User(Long id, String name){ this.id=id; this.name=name; }
    }
    interface UserRepository extends JpaRepository<User, Long> {}
}
```

**Kotlin — @RestClientTest с MockRestServiceServer**

```kotlin
package slices.restclient

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.client.RestClientTest
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.http.MediaType
import org.springframework.test.web.client.MockRestServiceServer
import org.springframework.web.client.RestClient
import org.springframework.test.web.client.match.MockRestRequestMatchers.*
import org.springframework.test.web.client.response.MockRestResponseCreators.*

class ExternalClient(private val rest: RestClient) {
    fun ping(): String = rest.get().uri("/ping").retrieve().body(String::class.java)!!
}

@RestClientTest(components = [ExternalClient::class])
class ExternalClientSliceTest {

    @Autowired lateinit var client: ExternalClient
    @Autowired lateinit var server: MockRestServiceServer

    @Test
    fun pingOk() {
        server.expect(requestTo("/ping"))
            .andExpect(method(org.springframework.http.HttpMethod.GET))
            .andRespond(withSuccess("""{"res":"pong"}""", MediaType.APPLICATION_JSON))

        val body = client.ping()
        assertThat(body).contains("pong")
    }
}
```

---

## Быстродействие: когда срез предпочтителен вместо @SpringBootTest

Скорость обратной связи — валюта разработки. Срезы выигрывают у полного контекста за счёт меньшего количества бинов, отсутствия сети и БД и более простого жизненного цикла. Типичный проект с десятками контроллеров и репозиториев тратит секунды на срезы и минуты на `@SpringBootTest`. На уровне CI это разница между «зелёным PR за 5 минут» и «ожиданием 20+ минут».

Срез предпочтителен, когда вы тестируете чистые обязанности слоя: роутинг/сериализацию/валидацию в контроллерах, маппинг сущностей и запросы в репозиториях, формат JSON в DTO, поведение исходящего клиента. Эти сценарии не требуют поднимать весь мир, и их изоляция делает тесты стабильнее.

Полный `@SpringBootTest` нужен для «склейки» слоёв: конфигурация безопасности + web + бины приложения, взаимодействие с миграциями и внешними брокерами, кастомные конвертеры, которые зависят от нескольких автоконфигураций. Такой тест дороже, но он ловит ошибки проводки и «магии» автоконфигов, которых не видно на срезе.

Практика: для каждого контроллера — 2–4 `@WebMvcTest` на happy-path и валидацию, плюс 1–2 «чёрных ящика» на `@SpringBootTest(RANDOM_PORT)` в smoke-наборе. Для каждого репозитория — 1–2 `@DataJpaTest` на жизненно важные запросы и индексы, а сложные агрегирующие сценарии — на интеграцию с Testcontainers.

Срезы заметно лучше кэшируются. Поскольку конфигурация минимальна, одинаковые наборы аннотаций легко попадают в кэш контекста JUnit/Spring, и второй-третий класс тестов стартует практически мгновенно. Полные контексты «грязнятся» чаще, и кэш помогает меньше.

Параллелизм играет на стороне срезов. Меньший контекст — меньше блокировок и меньше конкуренция за ресурсы. На CI вы можете запускать 4–8 воркеров без риска «убить» агента, в то время как полные интеграции начнут спорить за порты и память.

Экономия проявляется и в эволюции стеков. Обновление Spring Boot/Starter’ов ломает меньше тестов на срезах, потому что площадь автоконфигурации меньше. Это даёт более предсказуемые обновления и меньше «шумной» красноты.

Иногда заказчики просят «побольше интеграций». Аргументируйте: срезы не снижают качество, они сокращают время обратной связи и делают интеграции точечными. Плюс их проще поддерживать: ошибка в контракте контроллера видна тут же, а не в «чёрном ящике» через 5 минут.

Метрика полезности — медиана времени задачи `test` и доля `@SpringBootTest` в наборе. Если медиана превышает 2–3 минуты и доля полных контекстов > 20%, вы почти наверняка платите лишнее временем команды.

В завершение: стройте пирамиду. Срезы — средний слой между юнитами и интеграциями. Они дают много ценности за небольшую стоимость, и именно они превращают «медленный» набор в «быстрый».

**Gradle (Groovy) — отдельные таски для fast/slow**

```groovy
tasks.register('testFast', Test) {
    useJUnitPlatform { includeTags 'unit','component' }
    systemProperty "junit.jupiter.execution.parallel.enabled", "true"
}
tasks.register('testSlow', Test) {
    useJUnitPlatform { includeTags 'integration','e2e' }
}
```

**Gradle (Kotlin) — эквивалент**

```kotlin
tasks.register<Test>("testFast") {
    useJUnitPlatform { includeTags("unit","component") }
    systemProperty("junit.jupiter.execution.parallel.enabled", "true")
}
tasks.register<Test>("testSlow") {
    useJUnitPlatform { includeTags("integration","e2e") }
}
```

**Java — микросравнение времени среза vs полного контекста (эскиз)**

```java
package slices.speed;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.context.SpringBootTest;
import static org.assertj.core.api.Assertions.assertThat;

@WebMvcTest
class SliceBootTimeTest {
    @Test void sliceLoadsFast() { assertThat(true).isTrue(); } // сам факт загрузки среза
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class FullBootTimeTest {
    @Test void fullLoads() { assertThat(true).isTrue(); }
}
```

**Kotlin — эквивалент**

```kotlin
package slices.speed

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.boot.test.context.SpringBootTest

@WebMvcTest
class SliceBootTimeTest {
    @Test fun sliceLoadsFast() { assertThat(true).isTrue() }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class FullBootTimeTest {
    @Test fun fullLoads() { assertThat(true).isTrue() }
}
```

# 5. Интеграционные тесты с @SpringBootTest

## Режимы окружения: WebEnvironment.NONE/RANDOM_PORT/DEFINED_PORT

Интеграционные тесты в Spring Boot стартуют с `@SpringBootTest`, и первый выбор — **режим веб-окружения**. `WebEnvironment.NONE` поднимает полный `ApplicationContext` **без** веб-сервера. Это актуально для тестов бинов, транзакций, слушателей событий, security-конфигурации без HTTP, миграций и работы с БД. Такой режим дешевле по ресурсам и быстрее, чем поднимать порт.

`WebEnvironment.RANDOM_PORT` стартует реальный встроенный сервер (Tomcat/Jetty/Netty) на **случайном порту**. Это «чёрный ящик»: вы тестируете HTTP так же, как клиент в проде. Режим идеален для RestAssured, HTTP-кеширования, CORS, компрессии, фильтров, реальной цепочки security. Случайный порт устраняет конфликты в параллельных прогонах на CI.

`WebEnvironment.DEFINED_PORT` поднимает сервер на **фиксированном порту** из конфигурации (`server.port`). Так удобно тестировать взаимодействие с внешними контейнерами, которым нужно «знать» адрес сервиса (например, Testcontainers-Nginx перед приложением) или когда вы эмулируете внешние health-пробки. Минус — конкуренция за порт и меньшая параллельность.

Выбор режима влияет на изоляцию. Для «белых ящиков» (проверка конфигурации бинов/транзакций) используйте `NONE`. Для «чёрных ящиков» (HTTP-контракт, заголовки, gzip, кэш) — `RANDOM_PORT`. Для интеграции с окружением, ожидающим фиксированный адрес, — `DEFINED_PORT`, но запускайте такие тесты ограниченно и помечайте тегами.

Даже при `RANDOM_PORT` вы можете использовать `MockMvc` (через `@AutoConfigureMockMvc`), но тогда вы уже не «бьёте» реальную сеть и часть поведения (компрессия, сокеты) останется за кадром. Если нужна именно сеть — работайте RestAssured/WebClient по `localhost:{port}`.

Не забывайте про безопасность. В `RANDOM_PORT` вы запускаете полноценную цепочку фильтров, поэтому настройте пользователей/токены через Spring Security Test, иначе получите 401/403 вместо полезных проверок. В `NONE` вы тоже можете прогнать security через прямой вызов фильтров, но чаще это не требуется.

Сценарии загрузки файлов, SSE, WebSocket требуют **реального** сервера. Если вы используете WebFlux SSE — `RANDOM_PORT` или хотя бы `MOCK` со `WebTestClient` в другом слое. В интеграции выбирайте сетевой путь, чтобы исключить сюрпризы на проде.

Режимы можно комбинировать между классами: часть классов — `NONE` (быстро), небольшая пачка — `RANDOM_PORT` как smoke. Разделите тасками Gradle по тегам, чтобы PR не зависел от длинных сетевых сценариев.

Для `DEFINED_PORT` зафиксируйте порт в `application-test.yml`, а в CI заведите «одиночный» джоб. Параллелить такие тесты на одном агенте без namespace-изоляции нельзя — вы получите flaky.

И помните: поднятие реального сервера — это минуты на большом проекте. Чем реже вы это делаете, тем быстрее обратная связь. Сведите «чёрные ящики» к нескольким ключевым сценариям.

**Java — NONE и RANDOM_PORT (RestAssured)**

```java
package it.env;

import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import static org.hamcrest.Matchers.equalTo;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class ContextOnlyIT {
    @Test
    void contextLoads() {
        assertThat(true).isTrue();
    }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpBlackBoxIT {

    @LocalServerPort
    int port;

    @Test
    void helloEndpoint() {
        RestAssured.given().port(port)
                .get("/api/hello")
                .then().statusCode(200)
                .and().body("message", equalTo("hello"));
    }
}
```

**Kotlin — DEFINED_PORT и RANDOM_PORT**

```kotlin
package it.env

import io.restassured.RestAssured.given
import org.hamcrest.Matchers.equalTo
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class HttpBlackBoxIT {

    @LocalServerPort
    var port: Int = 0

    @Test
    fun helloEndpoint() {
        given().port(port)
            .get("/api/hello")
            .then().statusCode(200)
            .and().body("message", equalTo("hello"))
    }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class DefinedPortIT {
    @Test
    fun serverIsUp() {
        // порт берётся из application-test.yml -> server.port
    }
}
```

**application-test.yml (пример для DEFINED_PORT)**

```yaml
server:
  port: 18080
```

**Gradle (Groovy/Kotlin) — RestAssured**

```groovy
dependencies { testImplementation "io.rest-assured:rest-assured:5.5.0" }
```

```kotlin
dependencies { testImplementation("io.rest-assured:rest-assured:5.5.0") }
```

---

## Профили/свойства: @ActiveProfiles, @TestPropertySource, динамические свойства

Профили — основа управляемой конфигурации интеграционных тестов. Аннотация `@ActiveProfiles("test")` подбирает `application-test.yml` и все условные бины под профиль `test`. Это имитирует продакшн-настройку, но с безопасными подстановками (например, отключённые внешние интеграции, заменённые секреты).

`@TestPropertySource` добавляет или переопределяет свойства точечно **только** для этого класса/иерархии тестов. Это удобно, когда отличие от общего профиля касается пары ключей (например, таймаутов клиента или пути к каталогу). Свойства в `@TestPropertySource` имеют более высокий приоритет, чем profile-файлы.

`@DynamicPropertySource` программно публикует значения в рантайме, когда они известны **только после старта внешнего ресурса**: порты Testcontainers, временные директории, сгенерированные секреты. Это правильный способ «склеить» контейнеры и Spring без хрупких статических путей.

Порядок разрешения важен для предсказуемости. Вкратце: свойства в `@TestPropertySource` и `@DynamicPropertySource` перекрывают `application-*.yml`, а системные свойства/ENV — перекрывают всё. В тестах держитесь правила «меньше магии»: профиль задаёт основу, `@DynamicPropertySource` — данные, появляющиеся в рантайме, `@TestPropertySource` — точечные Overrides.

Не смешивайте **слишком много** источников на один класс: сложно отлаживать. Лучше вынести общие override в абстрактный базовый класс `AbstractIT` и наследовать. Так вы сохраняете единообразие и экономите на кэше контекста.

Для секретов используйте подстановки Spring (`${VAR:default}`) и публикуйте через `@DynamicPropertySource` или ENV на CI. Никогда не храните реальные секреты в `application-test.yml`. Контейнеры БД/брокеров можно заводить с тестовыми кредами «app/app».

Если у вас много модулей, имеет смысл завести **тестовый стартер** (Gradle subproject), который подключает общий `@TestConfiguration` с дефолтами и экспортирует его как зависимость `testImplementation(project(":test-support"))`. Это уменьшит дублирование.

Учитывайте, что `@TestPropertySource` **плохо** сочетается с параллельным запуском, если вы переопределяете путь к общим файлам. Генерируйте временные каталоги и прокидывайте их через `@DynamicPropertySource`.

Разграничивайте «тестовые» и «локальные» профили. `test` — для CI, детерминизм и контейнеры. `local` — для разработки, можно подключить внешнюю БД и дебажный логгер. В интеграционных тестах используйте `test`, а не `local`.

И не забывайте закрывать ресурсы. Если `@DynamicPropertySource` выдаёт порт живого контейнера, контейнер должен жить `@Container`/`@BeforeAll` и остановиться в `@AfterAll` (если не статический singleton). Утечки портов дают редкие и болезненные flaky.

**Java — профиль + TestPropertySource + DynamicPropertySource (Postgres)**

```java
package it.props;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.*;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@ActiveProfiles("test")
@TestPropertySource(properties = {
        "app.http.client.timeout=500ms",
        "logging.level.com.example=DEBUG"
})
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class PropsIT {

    static final PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    static { pg.start(); }

    @DynamicPropertySource
    static void dyn(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Test
    void contextLoads() {
        assertThat(pg.isRunning()).isTrue();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package it.props

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.springframework.test.context.TestPropertySource
import org.testcontainers.containers.PostgreSQLContainer

@ActiveProfiles("test")
@TestPropertySource(properties = [
    "app.http.client.timeout=500ms",
    "logging.level.com.example=DEBUG"
])
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class PropsIT {

    companion object {
        @JvmStatic val pg = PostgreSQLContainer("postgres:16-alpine").apply { start() }

        @JvmStatic
        @DynamicPropertySource
        fun dyn(r: DynamicPropertyRegistry) {
            r.add("spring.datasource.url", pg::getJdbcUrl)
            r.add("spring.datasource.username", pg::getUsername)
            r.add("spring.datasource.password", pg::getPassword)
        }
    }

    @Test
    fun contextLoads() {
        assertThat(pg.isRunning).isTrue()
    }
}
```

---

## Инициализация данных: @Sql, data.sql vs миграции, idempotent setup/teardown

Данные — кровь интеграционных тестов. Есть три основных способа их инициализировать: **миграции** (Flyway/Liquibase), **скрипты запуска Spring** (`schema.sql`/`data.sql`) и **точечные @Sql-скрипты** на класс/метод. Правильная стратегия — **схема через миграции**, а **тестовые данные через @Sql или фабрики**.

Миграции — единственный источник истины для схемы. Интеграционный тест должен подниматься на чистой БД с прогнанными миграциями и быть готов проверить репозитории/запросы. Нельзя «тайно» создавать таблицы в `schema.sql` для тестов, иначе вы не увидите дрейф между тестом и продом.

`data.sql` выполняется автоматически после поднятия контекста и создания схемы (если включено). Это удобно, но опасно: файл общий для всех тестов, и любые правки ломают независимость. Лучше использовать **`@Sql`** со scope на конкретный класс/метод, где вы чётко контролируете содержимое и порядок.

`@Sql` поддерживает `executionPhase` (`BEFORE_TEST_METHOD`/`AFTER_TEST_METHOD`) и может ссылаться на несколько файлов. Для **teardown** используйте `AFTER` с `TRUNCATE`/`DELETE`, но пишите **идемпотентные** скрипты: `DELETE FROM t WHERE ...;` или `TRUNCATE ... CASCADE;`. Не падайте, если данных уже нет — тесты должны быть повторяемыми.

Для вставок делайте скрипты **самодостаточными**: вставляйте все зависимые записи или используйте `ON CONFLICT DO NOTHING`/`INSERT ... WHERE NOT EXISTS`. Никаких «загадочных» внешних ключей, которые создаёт другой тест. Идеально, если каждый класс тестов изолируется в своей схеме/БД (особенно на параллельном CI).

Отличная практика — фабрики данных в коде (builders/Mothers) вместо SQL, когда нужны сложные связи. Но базовые справочники и большие вставки удобно делать SQL-файлом — это быстрее и нагляднее. Можно комбинировать: `@Sql` для справочников, билдеры — для конкретного сценария.

В Testcontainers можно инициализировать БД через `withInitScript(...)`, но помните: это на **контейнер**, а не на тест. Если вы делите контейнер между классами, сценарии должны быть полностью идемпотентными или обнуляйте схему между классами.

Важная деталь — кодировка и команды. Убедитесь, что SQL-файлы в UTF-8 и совместимы с вашей БД. В Postgres лучше избегать `SERIAL`, использовать `GENERATED ... AS IDENTITY` и явно указывать схемы.

И наконец, логируйте SQL на уровне тестов (`logging.level.org.hibernate.SQL=DEBUG`) в случае расследований, но держите по умолчанию INFO, чтобы отчёты CI не раздувались. В артефакты CI кладите применённые миграции и schema dump для аудита.

**Java — @Sql BEFORE/AFTER и репозиторий**

```java
package it.sql;

import jakarta.persistence.*;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.beans.factory.annotation.Autowired;

import static org.assertj.core.api.Assertions.assertThat;

@Entity(name = "users")
class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String name;
    public User() {}
    public User(String name) { this.name = name; }
}

interface UserRepo extends JpaRepository<User, Long> { long countByName(String name); }

@SpringBootTest
@Sql(scripts = "/sql/users_setup.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/sql/users_teardown.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class SqlDataIT {

    @Autowired UserRepo repo;

    @Test
    void seesSeedDataAndSavesNew() {
        assertThat(repo.countByName("seed")).isEqualTo(1);
        repo.save(new User("alice"));
        assertThat(repo.countByName("alice")).isEqualTo(1);
    }
}
```

**resources/sql/users_setup.sql**

```sql
-- идемпотентная вставка
INSERT INTO users(name) VALUES ('seed') ON CONFLICT DO NOTHING;
```

**resources/sql/users_teardown.sql**

```sql
TRUNCATE TABLE users RESTART IDENTITY CASCADE;
```

**Kotlin — эквивалент**

```kotlin
package it.sql

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.test.context.jdbc.Sql
import javax.annotation.Resource

@Entity(name = "users")
class User(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var name: String = ""
)
interface UserRepo : JpaRepository<User, Long> {
    fun countByName(name: String): Long
}

@SpringBootTest
@Sql(scripts = ["/sql/users_setup.sql"], executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = ["/sql/users_teardown.sql"], executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class SqlDataIT {

    @Resource lateinit var repo: UserRepo

    @Test
    fun seesSeedDataAndSavesNew() {
        assertThat(repo.countByName("seed")).isEqualTo(1)
        repo.save(User(name = "alice"))
        assertThat(repo.countByName("alice")).isEqualTo(1)
    }
}
```

---

## Транзакционность тестов: @Transactional, rollback, ограничения с Testcontainers

По умолчанию интеграционный тест **не** транзакционный. Если вы пометите метод/класс `@Transactional`, Spring стартует транзакцию **перед** тестом и сделает **rollback** после. Это удобно для изоляции: каждый тест оставляет БД в исходном состоянии без явного teardown.

У отката есть нюанс видимости. Если вы проверяете «что запись **не** осталась в БД», помните: rollback произойдёт **после** метода, и вы уже не выполните проверку. Решение — использовать отдельную транзакцию внутри теста через `TransactionTemplate`/новый `EntityManager`/репозиторий с `@Transactional(propagation = REQUIRES_NEW)` для «внешнего» чтения.

Если вам именно нужен **commit** (например, чтобы другая система/поток увидела изменения), поставьте `@Commit`/`@Rollback(false)` на тест. Делайте это точечно: такие тесты сложнее изолировать, и teardown обязателен.

С Testcontainers и `@DataJpaTest`/`@SpringBootTest` транзакции работают одинаково — это реальный Postgres. Ограничения касаются **долгих** операций и блокировок: транзакция поверх долгого DDL/аналитических запросов может держать блоки и замедлять соседние тесты. В таких сценариях не оборачивайте тест целиком в транзакцию — управляйте коммитами вручную.

Асинхронные слушатели (Kafka/JMS) работают в других потоках и **не** видят незакоммиченные изменения вашей тестовой транзакции. Если вы хотите, чтобы слушатель увидел запись, сначала коммитните изменения (`@Commit` или `TransactionTemplate.execute`), потом ожидайте событие.

При проверке уровня изоляции (`READ_COMMITTED`/`REPEATABLE_READ`) нельзя опираться на одну транзакцию. Создавайте две обособленных сессии и моделируйте гонки. Это уже ближе к специфическим интеграционным тестам на блокировки/фантомы.

Следите за автосбросом (`flush`). В JPA `save` не гарантирует немедленной записи; `flush` фиксирует SQL к БД внутри транзакции. Если вы проверяете `ConstraintViolationException`/уникальные индексы, вызовите `flush()` и ловите исключение, иначе оно поднимется на коммите, и тест может некорректно пройти.

Для чистых JDBC тестов аналогично: используйте `@Transactional` и `JdbcTemplate`. Отдельно контролируйте `autoCommit=false` и `commit/rollback` в сценариях, где важно поведение транзакции на границах.

И наконец, знайте меру. Большинство интеграционных тестов отлично живут **без** общей транзакции на метод. Изолируйте данные через teardown и используйте транзакцию только там, где это реально нужно (проверка ACID-поведения, rollback, блокировки).

**Java — rollback по умолчанию и явный commit для «внешнего» чтения**

```java
package it.tx;

import jakarta.persistence.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.Commit;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.support.TransactionTemplate;
import static org.assertj.core.api.Assertions.assertThat;

@Entity(name = "notes")
class Note {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String text;
    public Note() {}
    public Note(String text){ this.text=text; }
}

interface NoteRepo extends org.springframework.data.jpa.repository.JpaRepository<Note, Long> { long countByText(String t); }

@SpringBootTest
class TxIT {

    @Autowired NoteRepo repo;
    @Autowired PlatformTransactionManager txm;

    @Test
    @org.springframework.transaction.annotation.Transactional
    void rolledBackByDefault() {
        repo.save(new Note("temp"));
        assertThat(repo.countByText("temp")).isEqualTo(1);
        // после метода транзакция откатится, запись не останется
    }

    @Test
    void commitAndCheckFromNewTx() {
        var tt = new TransactionTemplate(txm);
        tt.execute(status -> { repo.save(new Note("committed")); return null; });
        long cnt = tt.execute(status -> repo.countByText("committed"));
        assertThat(cnt).isEqualTo(1);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package it.tx

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.transaction.PlatformTransactionManager
import org.springframework.transaction.annotation.Transactional
import org.springframework.transaction.support.TransactionTemplate
import javax.annotation.Resource

@Entity(name = "notes")
class Note(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var text: String = ""
)

interface NoteRepo : JpaRepository<Note, Long> { fun countByText(t: String): Long }

@SpringBootTest
class TxIT {

    @Resource lateinit var repo: NoteRepo
    @Resource lateinit var txm: PlatformTransactionManager

    @Test
    @Transactional
    fun rolledBackByDefault() {
        repo.save(Note(text = "temp"))
        assertThat(repo.countByText("temp")).isEqualTo(1)
        // после метода будет rollback
    }

    @Test
    fun commitAndCheckFromNewTx() {
        val tt = TransactionTemplate(txm)
        tt.execute { repo.save(Note(text = "committed")); null }
        val cnt = tt.execute { repo.countByText("committed") }
        assertThat(cnt).isEqualTo(1)
    }
}
```

---

## Context caching: ускорение, разделение на slow/fast наборы

Spring кэширует контексты между тестовыми классами, если **набор аннотаций и свойств совпадает**. Это радикально ускоряет прогон: дорогое построение контекста выполняется один раз. Ключ к скорости — **стабильные конфигурации** и отсутствие лишних инвалидаторов.

`@DirtiesContext` инвалидирует кэш. Используйте его **экономно**: только когда тест намеренно меняет состояние контекста (например, регистрирует/удаляет бины динамически) и это повлияет на следующего потребителя. Чрезмерное использование превращает кэш в тыкву и замедляет весь набор.

Группируйте тесты по классам, а не плодите сотни классов с уникальными аннотациями. Один класс — несколько методов — один контекст. Так кэш отдаёт максимум пользы. Старайтесь, чтобы все MVC-интеграции имели одинаковую «шапку» аннотаций, и все JPA-интеграции — тоже.

Разнесите «fast/slow» набора на уровне Gradle: быстрые интеграции (NONE/slices) и более тяжёлые (RANDOM_PORT/контейнеры). Это снижает среднее время PR. Nightly пусть гоняет всё, включая `@DirtiesContext`-сценарии и e2e.

Помните, что **профили и свойства** участвуют в кэш-ключе. Два класса с разными `@ActiveProfiles` получат **разные** контексты. Не плодите уникальные профили «на каждый случай» — централизуйте.

Не смешивайте разные контейнеры в одном контексте без нужды. Если одному тесту нужна Redis, а другому — Kafka, вы создадите «толстый» контекст. Разделите на два набора и кэшируйте каждый отдельно.

Хитрый ускоритель — **абстрактные базовые классы** с общей «шапкой» (`@SpringBootTest`, `@ActiveProfiles`, `@AutoConfigure...`). Конкретные классы расширяют базовый, не добавляя ничего нового, и получают кэш-хит.

Откажитесь от «динамической» регистрации бинов там, где можно обойтись конфигурацией. Любая динамика сложнее кэшируется и чаще приводит к `@DirtiesContext`. Простые бины — через `@TestConfiguration`/`@Import`.

Если вы включили параллельный запуск JUnit, следите за изоляцией: общий `DEFINED_PORT` или общая БД без namespace ломают параллельность и кэш. RANDOM_PORT и отдельные схемы/БД решают это.

И наблюдайте метрики: время построения контекста и число уникальных контекстов на сборку — важные показатели. Если их много — упростите «шапки» и объедините.

**Java — пример базового класса и точечного DirtiesContext**

```java
package it.cache;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.DirtiesContext;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
abstract class BaseIT { }

class A_IT extends BaseIT {
    @Test void a() { assertThat(true).isTrue(); }
}

@DirtiesContext // инвалидирует кэш после класса — используем только если нужно
class B_IT extends BaseIT {
    @Test void b() { assertThat(true).isTrue(); }
}
```

**Kotlin — эквивалент и Gradle-теги**

```kotlin
package it.cache

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.annotation.DirtiesContext

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
abstract class BaseIT

class A_IT : BaseIT() {
    @Test fun a() { assertThat(true).isTrue() }
}

@DirtiesContext
class B_IT : BaseIT() {
    @Test fun b() { assertThat(true).isTrue() }
}
```

**Gradle (Groovy/Kotlin) — разделение наборов**

```groovy
tasks.register('itFast', Test) {
    useJUnitPlatform { includeTags 'integration-fast' }
}
tasks.register('itSlow', Test) {
    useJUnitPlatform { includeTags 'integration-slow' }
}
```

```kotlin
tasks.register<Test>("itFast") { useJUnitPlatform { includeTags("integration-fast") } }
tasks.register<Test>("itSlow") { useJUnitPlatform { includeTags("integration-slow") } }
```

---

## Наблюдаемость: перехват логов, метрик, TestObservationRegistry

Интеграционные тесты — шанс проверить не только **функцию**, но и **сигналы**: логи, метрики, наблюдения (Observations). Это повышает уверенность в эксплуатационных аспектах: алерты сработают, дашборды не «умрут» после рефакторинга, ключевые события видны.

Перехват логов в Spring Boot удобен через `OutputCaptureExtension` (JUnit 5). Он захватывает stdout/stderr и вывод логгеров SLF4J/Logback в пределах теста. Вы можете утверждать наличие строки, префикса — или наоборот, отсутствие чувствительных данных. Не делайте проверки хрупкими: ищите **смысловые маркеры**, а не целый текст.

Метрики в Micrometer доступны через `MeterRegistry`. Если ваш код инкрементит счётчики/таймеры, в тесте можно после действия прочитать метрику (через `registry.find("name").counter().count()`) и сравнить с ожиданием. Для таймеров проверяйте количество записей, а не абсолютные наносекунды.

Наблюдения (Micrometer Observation) дают трассировку на уровне действий. Для тестов используйте `TestObservationRegistry` (либо из зависимости `micrometer-observation-test`, либо подмените `ObservationRegistry` на тестовый бин), который накапливает события. Дальше можно проверить имена/тэги завершённых Observations.

Будьте аккуратны с порядком: логи/метрики пишутся **асинхронно** в некоторых реализациях. Если ваш экспортёр буферизует данные, в тесте используйте синхронный backend или дождитесь через Awaitility. Для простого `SimpleMeterRegistry` это не требуется.

Проверки логов особенно полезны для ошибок: вы фиксируете шаблон сообщения и код причины. Это снимает риск «тихих» провалов, когда код перестал логировать важные проблемы. Но не превращайте тесты в «snapshot логов»: проверяйте только ключ.

Метрики — часть контрактов SRE. Если команда договорилась, что каждый успешный заказ увеличивает `orders.processed`, тест закрепляет это. Любой рефакторинг, забывший инкремент, будет пойман автоматикой.

Observations полезны при интеграциях: вы фиксируете, что HTTP-клиент/репозиторий оборачивается в Observation с нужными тегами (uri, статус, exception). На проде это уйдёт в трассировку; в тесте вы видите наличие и базовые поля.

Для изоляции определяйте тестовые бины `MeterRegistry`/`ObservationRegistry` в `@TestConfiguration` (Simple-реализации). Это исключит влияние глобальных экспортеров (Prometheus, OTLP) и сделает подсчёт предсказуемым.

И наконец, кладите логи интеграционных тестов в артефакты CI. При «красноте» это ускоряет анализ, а для аудита — подтверждает, что тесты проверяют эксплуатационные сигналы.

**Gradle (оба DSL) — зависимости наблюдаемости**

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-actuator"
    testImplementation "io.micrometer:micrometer-core"
    testImplementation "io.micrometer:micrometer-observation-test:1.13.1"
}
```

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    testImplementation("io.micrometer:micrometer-core")
    testImplementation("io.micrometer:micrometer-observation-test:1.13.1")
}
```

**Java — перехват логов и проверка метрики/наблюдения**

```java
package it.obs;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.observation.Observation;
import io.micrometer.observation.tck.TestObservationRegistry;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.slf4j.Logger; import org.slf4j.LoggerFactory;
import org.springframework.boot.test.system.CapturedOutput;
import org.springframework.boot.test.system.OutputCaptureExtension;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ExtendWith(OutputCaptureExtension.class)
class ObservabilityIT {

    @Configuration
    static class TestCfg {
        @Bean TestObservationRegistry testObservationRegistry() { return TestObservationRegistry.create(); }
        @Bean MeterRegistry registry() { return new io.micrometer.core.instrument.simple.SimpleMeterRegistry(); }
        @Bean DemoService demo(MeterRegistry r, TestObservationRegistry or) { return new DemoService(r, or); }
    }

    static class DemoService {
        private static final Logger log = LoggerFactory.getLogger(DemoService.class);
        private final MeterRegistry r; private final TestObservationRegistry or;
        DemoService(MeterRegistry r, TestObservationRegistry or){ this.r=r; this.or=or; }
        void doWork() {
            Observation.createNotStarted("demo.work", or).observe(() -> {
                r.counter("demo.count").increment();
                log.info("processed ok");
            });
        }
    }

    @Autowired DemoService svc;
    @Autowired MeterRegistry reg;
    @Autowired TestObservationRegistry or;

    @Test
    void logsMetricsAndObservations(CapturedOutput out) {
        svc.doWork();
        assertThat(out.getOut()).contains("processed ok");
        assertThat(reg.counter("demo.count").count()).isEqualTo(1.0);
        assertThat(or.getCompletedObservations())
                .anySatisfy(ctx -> assertThat(ctx.getContextualName()).contains("demo.work"));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package it.obs

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.simple.SimpleMeterRegistry
import io.micrometer.observation.Observation
import io.micrometer.observation.tck.TestObservationRegistry
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import org.slf4j.LoggerFactory
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.system.CapturedOutput
import org.springframework.boot.test.system.OutputCaptureExtension
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import javax.annotation.Resource

@SpringBootTest
@ExtendWith(OutputCaptureExtension::class)
class ObservabilityIT {

    @Configuration
    class TestCfg {
        @Bean fun testObservationRegistry(): TestObservationRegistry = TestObservationRegistry.create()
        @Bean fun registry(): MeterRegistry = SimpleMeterRegistry()
        @Bean fun demo(r: MeterRegistry, or: TestObservationRegistry) = DemoService(r, or)
    }

    class DemoService(private val r: MeterRegistry, private val or: TestObservationRegistry) {
        private val log = LoggerFactory.getLogger(DemoService::class.java)
        fun doWork() {
            Observation.createNotStarted("demo.work", or).observe {
                r.counter("demo.count").increment()
                log.info("processed ok")
            }
        }
    }

    @Resource lateinit var svc: DemoService
    @Resource lateinit var reg: MeterRegistry
    @Resource lateinit var or: TestObservationRegistry

    @Test
    fun logsMetricsAndObservations(out: CapturedOutput) {
        svc.doWork()
        assertThat(out.out).contains("processed ok")
        assertThat(reg.counter("demo.count").count()).isEqualTo(1.0)
        assertThat(or.completedObservations).anySatisfy { ctx ->
            assertThat(ctx.contextualName).contains("demo.work")
        }
    }
}
```

# 6. Тестирование веб-слоя и безопасности

## MockMvc/WebTestClient/RestAssured: выбор инструмента (MVC vs WebFlux vs black-box)

Первый вопрос веб-тестов — на каком уровне вы хотите «резать» приложение. Если проверяется только сервлетный слой (роутинг, биндинг, сериализация, фильтры), то `MockMvc` — быстрый и достаточный инструмент, который исполняет весь MVC-пайплайн внутри JVM без реального сокета.

Для реактивного стека аналог `MockMvc` — это `WebTestClient`. Он работает как в «встраиваемом» режиме против контроллеров WebFlux, так и как HTTP-клиент к реальному серверу. Если ваш сервис на Netty/Reactive, WebTestClient даёт естественный DSL и умеет проверять `Mono/Flux` и SSE.

Когда же цель — «чёрный ящик» через настоящий порт (компрессия, CORS, кэш, proxy-заголовки), логичнее выбрать RestAssured. Он идёт поверх реального `RANDOM_PORT` и проверяет HTTP-контракт так, как это делает внешний потребитель: с TCP-стеком, кодировками и всеми сетевыми нюансами.

Выбор инструмента определяет стоимость теста. `MockMvc`/`WebTestClient` как срезы стартуют за миллисекунды и масштабируются в CI. RestAssured поднимает сервер и стоит дороже, поэтому его применяют точечно — для smoke-наборов и сценариев, где важно «как в проде».

Даже в MVC-проектах уместно иметь оба подхода. Большинство сценариев — быстрые `@WebMvcTest` с `MockMvc`; несколько ключевых — `@SpringBootTest(RANDOM_PORT)` с RestAssured: загрузка файлов, проверка кэша, кросс-origin, gzip. Это держит баланс скорости и уверенности.

Ошибки «склейки» часто видны только на реальном порту (например, конфликт фильтров безопасности и прокси). Поэтому полезно держать один-два интеграционных «черных ящика», которые потенциально ветвятся на уровне инфраструктуры.

WebFlux-команды обычно делают тот же микс: тактические проверки контроллеров через `@WebFluxTest` и 1-2 smoke-класса через `RANDOM_PORT` и `WebTestClient.bindToServer()`/RestAssured для проверок, зависящих от сети.

`MockMvc` дружит со Spring Security Test из коробки: CSRF, аутентификация, методная безопасность. В реактивном мире аналог — конфигураторы `SecurityMockServerConfigurers` для WebTestClient. Это позволяет полноценно проверять защищённые эндпоинты без IdP.

Наконец, смотрите на «точку боли» вашей команды. Если баги чаще в сериализации и обработке ошибок — ставьте акцент на срезы. Если в заголовках/кэше/прокси — добавляйте «черные ящики». Выбор инструмента — отражение вашей архитектурной поверхности.

**Gradle (Groovy)**

```groovy
dependencies {
    testImplementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testImplementation "io.rest-assured:rest-assured:5.5.0"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.rest-assured:rest-assured:5.5.0")
}
```

**Java — @WebMvcTest с MockMvc**

```java
package web.choice.mvc;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
@RequestMapping("/api/hello")
class HelloController {
    record Greeting(String message) {}
    @GetMapping public Greeting hello() { return new Greeting("hello"); }
}

@WebMvcTest(controllers = HelloController.class)
class HelloControllerMvcTest {
    @Autowired MockMvc mvc;

    @Test
    void returns200Json() throws Exception {
        mvc.perform(get("/api/hello"))
           .andExpect(status().isOk())
           .andExpect(content().json("{\"message\":\"hello\"}"));
    }
}
```

**Kotlin — @WebFluxTest с WebTestClient**

```kotlin
package web.choice.webflux

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Mono
import javax.annotation.Resource

@RestController
@RequestMapping("/rx/hello")
class RxHelloController {
    data class Greeting(val message: String)
    @GetMapping fun hello(): Mono<Greeting> = Mono.just(Greeting("hello"))
}

@WebFluxTest(controllers = [RxHelloController::class])
class RxHelloControllerTest {
    @Resource lateinit var client: WebTestClient

    @Test
    fun returns200Json() {
        client.get().uri("/rx/hello")
            .exchange()
            .expectStatus().isOk
            .expectBody().json("""{"message":"hello"}""")
    }
}
```

**Java — RestAssured на RANDOM_PORT**

```java
package web.choice.blackbox;

import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

import static org.hamcrest.Matchers.equalTo;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class BlackBoxIT {
    @LocalServerPort int port;

    @Test
    void hello() {
        RestAssured.given().port(port)
            .get("/api/hello")
            .then().statusCode(200)
            .and().body("message", equalTo("hello"));
    }
}
```

---

## Валидация запросов/ответов: media types, схемы, локализация сообщений

Валидация начинается с медиатипов. Контроллер должен корректно реагировать на `Accept`/`Content-Type`: отдавать 406 при неподдерживаемом формате и 415 при неподдерживаемом теле. Такие проверки на уровне `MockMvc`/`WebTestClient` дешевы и предотвращают «немые» несовместимости клиентов.

Следующий слой — структура тела. Полноценная JSON Schema в юнит-наборах тяжелее, поэтому в веб-тестах часто достаточно точечных проверок JSONPath/JSONAssert: обязательные поля, типы, формат дат. Схемную валидацию оставляют контрактным или e2e-слоям.

Локализация сообщений — частый источник флейков. Если API локализует тексты ошибок, тест должен задавать `Accept-Language` и проверять ключевые маркеры, а не весь текст. Для стабильности лучше сверять коды и поля, а тексты складывать в отдельные тесты локализации.

Валидация запроса на стороне `@Valid` превращается в `MethodArgumentNotValidException`. Глобальный маппер ошибок должен переводить его в единый формат (например, Problem Details). Тесты должны закрепить ожидаемые поля: `type`, `title`, `status`, `detail`, `errors`.

При парсинге дат/денег важно фиксировать формат. На уровне `@JsonTest` вы проверяете сериализацию, а в веб-тестах — что заголовки `Content-Type` и `Accept` корректно согласованы, а попытка прислать неправильный формат приводит к 400 с нужным кодом ошибки.

Смешение медиа-типов (например, `application/json` и `application/problem+json`) легко ломается автоконфигурациями. Включите проверку `produces/consumes` в каждом важном эндпоинте: это дешево и спасает от регрессов при обновлении Spring Boot.

Если API поддерживает несколько языков, заведите тесты на «дефолт» и одну альтернативу. Ключевое — не проверять весь текст, а проверять наличие локализованного маркера (например, «должно быть ≥ 1» на нужном языке) и код поля.

Валидация ответов включает и заголовки кэширования, CORS, ETag. Их проверим ниже, но здесь важно помнить: заголовки — часть контракта. Любое незапланированное изменение приведёт к деградации UX клиента.

Для бинарных ответов проверяйте `Content-Type` и `Content-Disposition`. Даже если тело проверяется в другом месте, правильная разметка — гарантия, что клиент распознаёт файл.

И наконец, держите пример «правильный/неправильный» для каждого публичного эндпоинта. Это помогает не только инженерам, но и документации — они служат живыми «золотыми файлами» поведения.

**Java — валидация media types и локализации с MockMvc**

```java
package web.validation;

import jakarta.validation.constraints.Min;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.TestConfiguration;
import org.springframework.http.MediaType;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import org.springframework.context.MessageSource;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController @Validated
class PriceController {
    record Price(int cents) {}
    @PostMapping(value="/price", consumes=MediaType.APPLICATION_FORM_URLENCODED_VALUE,
                               produces=MediaType.APPLICATION_JSON_VALUE)
    public Price price(@RequestParam @Min(1) int qty) { return new Price(qty*10); }

    @GetMapping(value="/price", produces=MediaType.APPLICATION_JSON_VALUE)
    public Price ping() { return new Price(10); }
}

@WebMvcTest(controllers = PriceController.class)
class PriceControllerValidationTest {

    @TestConfiguration
    static class Cfg {
        @Bean MessageSource messageSource() {
            var ms = new ResourceBundleMessageSource();
            ms.setBasename("messages");
            ms.setDefaultEncoding("UTF-8");
            return ms;
        }
    }

    @Autowired MockMvc mvc;

    @Test
    void rejectsUnsupportedAccept() throws Exception {
        mvc.perform(get("/price").accept(MediaType.APPLICATION_XML))
           .andExpect(status().isNotAcceptable());
    }

    @Test
    void rejectsBadParamAndLocalizes() throws Exception {
        mvc.perform(post("/price")
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .param("qty","0")
                .header("Accept-Language","ru"))
            .andExpect(status().isBadRequest())
            .andExpect(header().string("Content-Type","application/problem+json"));
    }
}
```

**resources/messages_ru.properties (фрагмент)**

```
jakarta.validation.constraints.Min.message=должно быть не меньше {value}
```

**Kotlin — эквивалент с WebFlux и WebTestClient**

```kotlin
package web.validation

import jakarta.validation.constraints.Min
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.support.ResourceBundleMessageSource
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Mono
import javax.annotation.Resource

@RestController @Validated
class RxPriceController {
    data class Price(val cents: Int)
    @PostMapping("/rx/price", consumes=[MediaType.APPLICATION_JSON_VALUE])
    fun price(@RequestBody req: Map<String, Int>): Mono<Price> =
        Mono.just(Price((req["qty"] ?: 0) * 10))
}

@WebFluxTest(controllers = [RxPriceController::class])
class RxPriceControllerValidationTest {

    @Configuration
    class Cfg {
        @Bean fun messageSource() = ResourceBundleMessageSource().apply {
            setBasename("messages"); setDefaultEncoding("UTF-8")
        }
    }

    @Resource
    lateinit var client: WebTestClient

    @Test
    fun rejectsWrongContentType() {
        client.post().uri("/rx/price")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .bodyValue("qty=1")
            .exchange()
            .expectStatus().isUnsupportedMediaType
    }
}
```

---

## Глобальные обработчики ошибок: @ControllerAdvice, единый формат (RFC7807)

Единый формат ошибок — опора зрелых API. В Spring Boot 3 доступен `ProblemDetail` (реализация RFC7807), через который удобно описывать `type`, `title`, `status`, `detail` и дополнительные поля. Глобальный совет `@RestControllerAdvice` превращает исключения и валидационные ошибки в стандартизованный JSON.

Преимущество централизованного маппинга — слабая связность. Контроллеры не заботятся о «как сформировать ошибку», а маппер отвечает за формат, коды и локализацию. Это упрощает поддержку и исключает разнобой.

Обязательная проверка — соответствие кода статуса и поля `status`. Несогласованность легко возникает при ручной сборке ответов. Тесты должны фиксировать, что 400 = 400 и в статусе, и в `ProblemDetail`.

Валидационные ошибки удобнее представлять как массив `errors` с `field`/`message`/`code`. RFC7807 позволяет расширения через произвольные свойства. Маппер собирает нарушения Bean Validation и добавляет массив к `ProblemDetail`.

Исключения домена (например, `DomainException`) лучше маппить в 409/422, а не в 500. Правило маппинга должно быть очевидным и задокументированным, а тесты — закреплять это как контракт.

Локализация текстов делается через `MessageSource`. В тестах важно проверять не только статус, но и «тип» (`type` как URI), если вы используете его для категоризации, и ключевые поля массива ошибок. Полные тексты проверяйте точечно, чтобы не ломать тесты при улучшениях копирайта.

Для WebFlux совет работает аналогично — тот же `@RestControllerAdvice`. В реактивных контроллерах исключение так же поднимается по цепочке и маппится глобально.

Сторонние исключения (например, от HTTP-клиента) полезно ловить отдельно, чтобы скрывать внутренности и отдавать «безопасную» ошибку. Тесты должны покрывать этот «перевод» в безопасный ответ.

И наконец, держите negative-тесты на «грязный» JSON/тип поля. Это спасает от внезапных 500 при невалидном входе и проверяет, что совет не «утекает» стек-трейсами наружу.

**Java — @RestControllerAdvice → ProblemDetail и тест**

```java
package web.errors;

import jakarta.validation.ConstraintViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
import org.springframework.validation.BindException;
import org.springframework.web.bind.MethodArgumentNotValidException;

import java.net.URI;
import java.util.*;

@RestControllerAdvice
class GlobalErrors {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation failed");
        pd.setType(URI.create("https://example.com/problems/validation"));
        List<Map<String, String>> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
                .toList();
        pd.setProperty("errors", errors);
        return pd;
    }

    @ExceptionHandler({BindException.class, MethodArgumentTypeMismatchException.class})
    ProblemDetail onBind(Exception ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Bad request");
        pd.setType(URI.create("https://example.com/problems/bad-request"));
        pd.setDetail(ex.getMessage());
        return pd;
    }

    @ExceptionHandler(Exception.class)
    ProblemDetail onOther(Exception ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal error");
        pd.setType(URI.create("https://example.com/problems/internal"));
        return pd;
    }
}
```

```java
package web.errors;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
@RequestMapping("/api/err")
class ErrController {
    @GetMapping("/boom") String boom(){ throw new RuntimeException("x"); }
}

@WebMvcTest(controllers = {ErrController.class, GlobalErrors.class})
class GlobalErrorsTest {

    @Autowired MockMvc mvc;

    @Test
    void mapsRuntimeToProblem() throws Exception {
        mvc.perform(get("/api/err/boom"))
           .andExpect(status().isInternalServerError())
           .andExpect(header().string("Content-Type","application/problem+json"))
           .andExpect(jsonPath("$.title").value("Internal error"))
           .andExpect(jsonPath("$.status").value(500));
    }
}
```

**Kotlin — эквивалент для WebFlux**

```kotlin
package web.errors

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import javax.annotation.Resource

@RestController
@RequestMapping("/rx/err")
class RxErrController { @GetMapping("/boom") fun boom(): String = throw RuntimeException("x") }

@WebFluxTest(controllers = [RxErrController::class, GlobalErrors::class])
class GlobalErrorsRxTest {
    @Resource lateinit var client: WebTestClient

    @Test
    fun mapsRuntimeToProblem() {
        client.get().uri("/rx/err/boom")
            .exchange()
            .expectStatus().is5xxServerError
            .expectHeader().valueEquals("Content-Type","application/problem+json")
            .expectBody().jsonPath("$.title").isEqualTo("Internal error")
    }
}
```

---

## Spring Security Test: @WithMockUser, JWT/OAuth2, CSRF, методная безопасность

Тесты безопасности должны проверять три пласта: аутентификацию, авторизацию и защитные механизмы (CSRF, CORS, заголовки). Spring Security Test предоставляет аннотации и «постпроцессоры» запросов, которые эмулируют пользователя/токен без реального IdP.

Для формальной аутентификации достаточно `@WithMockUser` (имя, роли). Это быстро подключает пользователя к контексту Security. Для авторизации на уровне методов (`@PreAuthorize`) тесты проверяют 403 для неподходящих ролей и 200 для подходящих.

JWT-сценарии тестируются через `SecurityMockMvcRequestPostProcessors.jwt()` для MVC и `SecurityMockServerConfigurers.mockJwt()` для WebFlux. Вы можете задавать `claim`, `scope`, `authorities` и проверять свою `JwtAuthenticationConverter`.

CSRF по умолчанию включён для небезопасных методов (POST/PUT/PATCH/DELETE). В MVC-тестах используйте `with(csrf())`, чтобы запрос прошёл. Наличие 403 без CSRF — часть контракта безопасности.

Методная безопасность требует поднятия прокси, но в `@WebMvcTest`/`@SpringBootTest` это уже есть. Важно иметь тесты, где доступ разрешён/запрещён, и фиксировать статусы. Это защищает от случайного расширения доступа.

Для WebFlux всё аналогично, но конфигураторы другие: `mutateWith(mockJwt())`/`mutateWith(mockOAuth2Login())`. Структура проверок та же.

Ошибки безопасности желательно отдавать в едином формате Problem Details, чтобы клиенты получали единообразные ответы. Это можно сделать через `AuthenticationEntryPoint`/`AccessDeniedHandler` или `@ControllerAdvice`. Тесты должны покрывать эти ветви.

Не забывайте «негатив»: попытка доступа без аутентификации, с просроченным токеном (эмулируется claim), с неправильным scope. Эти проверки дешевы, но ловят реальные инциденты.

Наконец, сами тесты не должны ходить в сеть к IdP. Всё эмулируется локально, что делает набор быстрым и детерминированным.

**Gradle (оба DSL)**

```groovy
dependencies { testImplementation "org.springframework.security:spring-security-test" }
```

```kotlin
dependencies { testImplementation("org.springframework.security:spring-security-test") }
```

**Java — MVC: @WithMockUser, JWT и CSRF**

```java
package web.sec.mvc;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
@RequestMapping("/api/secure")
class SecureController {
    @GetMapping("/admin")
    @PreAuthorize("hasRole('ADMIN')")
    String admin() { return "ok"; }

    @PostMapping("/submit")
    String submit() { return "ok"; }
}

@WebMvcTest(controllers = SecureController.class)
class SecureControllerMvcTest {

    @Autowired MockMvc mvc;

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminAllowed() throws Exception {
        mvc.perform(get("/api/secure/admin")).andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void adminForbidden() throws Exception {
        mvc.perform(get("/api/secure/admin")).andExpect(status().isForbidden());
    }

    @Test
    void jwtWithScopeAllows() throws Exception {
        mvc.perform(get("/api/secure/admin")
                .with(jwt().authorities(a -> a.add(() -> "ROLE_ADMIN"))))
           .andExpect(status().isOk());
    }

    @Test
    @WithMockUser
    void csrfRequired() throws Exception {
        mvc.perform(post("/api/secure/submit")).andExpect(status().isForbidden());
        mvc.perform(post("/api/secure/submit").with(csrf())).andExpect(status().isOk());
    }
}
```

**Kotlin — WebFlux: mockJwt и методная безопасность**

```kotlin
package web.sec.webflux

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.mockJwt
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import javax.annotation.Resource

@RestController
@RequestMapping("/rx/secure")
class RxSecureController {
    @GetMapping("/admin")
    @PreAuthorize("hasRole('ADMIN')")
    fun admin() = "ok"
}

@WebFluxTest(controllers = [RxSecureController::class])
class RxSecureControllerTest {

    @Resource lateinit var client: WebTestClient

    @Test
    fun adminAllowedWithJwt() {
        client.mutateWith(mockJwt().authorities { it.add { "ROLE_ADMIN" } })
            .get().uri("/rx/secure/admin")
            .exchange()
            .expectStatus().isOk
    }

    @Test
    fun adminForbiddenWithoutRole() {
        client.mutateWith(mockJwt())
            .get().uri("/rx/secure/admin")
            .exchange()
            .expectStatus().isForbidden
    }
}
```

---

## Фильтры, interceptors, CORS и кэширование — проверка цепочки

Фильтры и перехватчики — часть поведения веб-слоя: корреляция, логирование, метрики, заголовки. Их проще всего проверять через `@WebMvcTest`/`@AutoConfigureMockMvc`, импортируя сами фильтры. Тест должен подтверждать присутствие заголовка/модификацию ответа.

CORS часто ломается «тихо»: на фронте просто не уходят запросы. Юнит-проверка `OPTIONS` preflight и нужных `Access-Control-Allow-*` заголовков предотвращает регрессы. Лучше держать тест с конкретным origin/методом/заголовками.

Кэширование на уровне HTTP фиксируется заголовками `Cache-Control`, `ETag`, `Last-Modified`. Самый простой «якорь» — проверить `Cache-Control`. Если используете `ShallowEtagHeaderFilter`, можно проверить и условный GET: второй запрос с `If-None-Match` возвращает 304.

Interceptors (HandlerInterceptor) удобно использовать для корреляции (`X-Request-Id`) и времени обработки. Тесты должны подтверждать, что заголовок добавлен и не пропадает на ошибках.

Порядок фильтров влияет на поведение. Если у вас есть security-цепочка + собственные фильтры, тестируйте, что ваш фильтр выполняется «после/до» нужного места — например, что заголовок не перезаписывается.

Для WebFlux все те же идеи, только интерфейсы другие (`WebFilter`, `HandlerFilterFunction`). WebTestClient позволяет проверять заголовки и статусы на равных с MVC.

Важно не перегружать проверками. Зафиксируйте ключ: наличие CORS для нужного origin, политика кэша на публичных ресурсах, корректный ETag на неизменном ресурсе, наличие корреляционного идентификатора.

Если кэш-контракт включает «vary» (по `Accept-Language`, `Accept-Encoding`), его тоже полезно закрепить — иначе CDN/прокси начнут отдавать «не тот» вариант.

И, наконец, не забывайте про негативы: запретный origin должен получать 403/нет CORS-заголовков; попытка небезопасного метода без разрешённых заголовков должна падать.

**Java — фильтр, CORS и Cache-Control с MockMvc**

```java
package web.chain;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletResponse;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.filter.ShallowEtagHeaderFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.web.servlet.config.annotation.*;

import java.io.IOException;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
class CacheController {
    @GetMapping("/api/cacheable")
    public String data() { return "v1"; }
}

@Configuration
class WebCfg implements WebMvcConfigurer {
    @Bean Filter requestIdFilter() {
        return new Filter() {
            @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
                ((HttpServletResponse)res).addHeader("X-Request-Id", "test-id");
                chain.doFilter(req, res);
            }
        };
    }
    @Bean ShallowEtagHeaderFilter etag() { return new ShallowEtagHeaderFilter(); }
    @Override public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**").allowedOrigins("https://example.com").allowedMethods("GET","POST");
    }
}

@WebMvcTest(controllers = CacheController.class)
@Import(WebCfg.class)
class ChainTest {

    @Autowired MockMvc mvc;

    @Test
    void addsRequestIdHeader() throws Exception {
        mvc.perform(get("/api/cacheable"))
           .andExpect(header().string("X-Request-Id","test-id"));
    }

    @Test
    void corsPreflight() throws Exception {
        mvc.perform(options("/api/cacheable")
                .header("Origin","https://example.com")
                .header("Access-Control-Request-Method","GET"))
           .andExpect(status().isOk())
           .andExpect(header().string("Access-Control-Allow-Origin","https://example.com"));
    }

    @Test
    void etagAndNotModified() throws Exception {
        var etag = mvc.perform(get("/api/cacheable"))
                .andReturn().getResponse().getHeader("ETag");
        mvc.perform(get("/api/cacheable").header("If-None-Match", etag))
           .andExpect(status().isNotModified());
    }
}
```

**Kotlin — WebFlux WebFilter и CORS**

```kotlin
package web.chain

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Import
import org.springframework.http.HttpHeaders
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.cors.reactive.CorsWebFilter
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource
import org.springframework.web.cors.CorsConfiguration
import org.springframework.web.server.WebFilter
import reactor.core.publisher.Mono
import javax.annotation.Resource

@RestController
class RxCacheController { @GetMapping("/rx/cacheable") fun data() = "v1" }

@Configuration
class RxWebCfg {
    @Bean fun reqIdFilter(): WebFilter = WebFilter { _, chain ->
        chain.filter(it).contextWrite { ctx -> ctx }.doOnEach { sig ->
            // заголовок добавим после ответа:
        }
    }
    @Bean fun cors(): CorsWebFilter {
        val cfg = CorsConfiguration().apply {
            addAllowedOrigin("https://example.com"); addAllowedMethod("*"); addAllowedHeader("*")
        }
        val source = UrlBasedCorsConfigurationSource().apply { registerCorsConfiguration("/rx/**", cfg) }
        return CorsWebFilter(source)
    }
}

@WebFluxTest(controllers = [RxCacheController::class])
@Import(RxWebCfg::class)
class RxChainTest {

    @Resource lateinit var client: WebTestClient

    @Test
    fun corsPreflight() {
        client.options().uri("/rx/cacheable")
            .header(HttpHeaders.ORIGIN, "https://example.com")
            .header("Access-Control-Request-Method","GET")
            .exchange()
            .expectStatus().isOk
            .expectHeader().valueEquals("Access-Control-Allow-Origin","https://example.com")
    }
}
```

---

## Файлы/стримы/WebSocket/SSE — нюансы тестирования

Загрузка файлов (multipart) удобно тестируется через `MockMvc` `multipart(...)`. Тест должен проверять статус, тип ответа и то, что файл реально попал в контроллер (например, по размеру/имени). Для безопасности добавьте проверку ограничений размера/типа в отдельном тесте.

Выгрузка файлов — это `Content-Disposition` и корректный `Content-Type`. Даже если тело бинарное, тест может проверить длину или хеш при необходимости. Главное — закрепить правильные заголовки, иначе браузеры/клиенты поведут себя непредсказуемо.

Стриминговые ответы в MVC (например, `StreamingResponseBody`) проверяются на наличие заголовков и, опционально, на префикс/структуру байтов. Это не нагрузочный тест, а sanity-check контракта.

SSE (Server-Sent Events) — естественная зона WebFlux. `WebTestClient` позволяет проверять «горячие» стримы и читать определённое количество событий. Тесты должны быть с таймаутом и без `Thread.sleep` — используйте встроенные ожидания.

WebSocket лучше проверять «черным ящиком» на `RANDOM_PORT`, подключаясь `ReactorNettyWebSocketClient`. Тест отправляет сообщение и ожидает ответ (echo/ack). Важно закрыть сессию и зафиксировать, что соединение действительно устанавливается.

Файлы и стримы часто попадают под ограничения безопасности (CSRF для multipart, авторизация для скачиваний). Тест должен эмулировать пользователя и CSRF при необходимости.

Большие файлы/долгие стримы не стоит проверять в интеграционном наборе — это задача нагрузки/контрактов. Здесь мы закрепляем только форматы и базовую работоспособность.

При SSE/WS полезно проверять заголовки кэширования (обычно отключены) и CORS для соответствующих endpoints, если их потребляет браузер.

Наконец, не забывайте чистить временные файлы. В контроллере используйте `@TempDir` в тесте или моковый стор, чтобы не оставлять мусор.

**Java — multipart upload и streaming download с MockMvc**

```java
package web.files;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.*;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.web.servlet.MockMvc;

import java.nio.charset.StandardCharsets;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.multipart;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RestController
class FileController {

    @PostMapping(path="/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public String upload(@RequestPart("file") org.springframework.web.multipart.MultipartFile file) {
        return "size=" + file.getSize();
    }

    @GetMapping("/download")
    public ResponseEntity<StreamingResponseBody> download() {
        var body = (StreamingResponseBody) out -> out.write("hello".getBytes(StandardCharsets.UTF_8));
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"greet.txt\"")
                .contentType(MediaType.TEXT_PLAIN)
                .body(body);
    }
}

@WebMvcTest(controllers = FileController.class)
class FileControllerTest {

    @Autowired MockMvc mvc;

    @Test
    void uploadMultipart() throws Exception {
        var file = new MockMultipartFile("file", "a.txt", "text/plain", "abc".getBytes());
        mvc.perform(multipart("/upload").file(file))
           .andExpect(status().isOk())
           .andExpect(content().string("size=3"));
    }

    @Test
    void downloadStreaming() throws Exception {
        mvc.perform(get("/download"))
           .andExpect(status().isOk())
           .andExpect(header().string(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"greet.txt\""))
           .andExpect(header().string("Content-Type","text/plain"))
           .andExpect(content().string("hello"));
    }
}
```

**Kotlin — SSE (WebFlux) с WebTestClient**

```kotlin
package web.sse

import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import reactor.core.publisher.Flux
import java.time.Duration
import javax.annotation.Resource

@RestController
class SseController {
    data class Event(val msg: String)
    @GetMapping(path = ["/sse"], produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun sse(): Flux<Event> = Flux.interval(Duration.ofMillis(50)).take(3).map { Event("e$it") }
}

@WebFluxTest(controllers = [SseController::class])
class SseControllerTest {

    @Resource lateinit var client: WebTestClient

    @Test
    fun streamsThreeEvents() {
        client.get().uri("/sse")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk
            .returnResult(String::class.java)
            .responseBody
            .take(1) // достаточно убедиться, что поток идёт
            .blockFirst()
    }
}
```

**Java — WebSocket echo на RANDOM_PORT**

```java
package web.ws;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import org.springframework.web.socket.*;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.config.annotation.*;

import java.net.URI;
import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThat;

@Configuration
@EnableWebSocket
class WsConfig implements WebSocketConfigurer {
    @Override public void registerWebSocketHandlers(WebSocketHandlerRegistry reg) {
        reg.addHandler(new EchoHandler(), "/ws/echo").setAllowedOrigins("*");
    }
    static class EchoHandler extends TextWebSocketHandler {
        @Override protected void handleTextMessage(WebSocketSession s, TextMessage m) throws Exception {
            s.sendMessage(new TextMessage("echo:" + m.getPayload()));
        }
    }
}

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, classes = WsConfig.class)
class WebSocketIT {

    @LocalServerPort int port;

    @Test
    void echo() throws Exception {
        var client = new StandardWebSocketClient();
        var latch = new CountDownLatch(1);
        var result = new ArrayBlockingQueue<String>(1);

        client.doHandshake(new WebSocketHandler() {
            @Override public void afterConnectionEstablished(WebSocketSession session) throws Exception {
                session.sendMessage(new TextMessage("ping"));
            }
            @Override public void handleMessage(WebSocketSession session, WebSocketMessage<?> message) {
                result.add(((TextMessage)message).getPayload()); latch.countDown();
            }
            @Override public void handleTransportError(WebSocketSession session, Throwable exception) { }
            @Override public void afterConnectionClosed(WebSocketSession session, CloseStatus closeStatus) { }
            @Override public boolean supportsPartialMessages() { return false; }
        }, "ws://localhost:" + port + "/ws/echo").get();

        latch.await(1, TimeUnit.SECONDS);
        assertThat(result.poll(100, TimeUnit.MILLISECONDS)).isEqualTo("echo:ping");
    }
}
```

# 7. Данные и БД: реалистичность против скорости

## Почему H2 ≠ прод и когда он уместен; лучше — Testcontainers с вашей БД.

Первое правило тестирования хранилищ — **совместимость важнее скорости**. Встраиваемые БД вроде H2 удобны: быстрый старт, нулевые внешние зависимости, лёгкая отладка. Но они не повторяют поведение продакшн-БД: типы, индексы, оптимизатор, блокировки и даже синтаксис SQL могут отличаться критично. Это означает риск «зелёных» тестов и «красного» продакшна.

У Postgres и H2 разные диалекты и типы: `jsonb`, `timestamp with time zone`, `generated as identity`, `on conflict do nothing`, `gist/gin` — всё это нативно для Postgres и эмулируется частично или не эмулируется вовсе в H2. Даже режим `MODE=PostgreSQL` не устраняет всех расхождений: функции и планы выполнения будут другими.

Поведение транзакций и блокировок различается: Postgres — MVCC со своим набором уровней изоляции и долгоживущих снапшотов; H2 проще и часто «прощает» ошибки синхронизации. Тест, который «не поймал» взаимную блокировку на H2, легко упадёт под нагрузкой на Postgres.

Индексы и их типы — ещё одна зона несовместимости. Часто приходят баги «на проде медленно», хотя тесты на H2 летали. На самом деле запросы зависят от типов индексов (B-Tree, GIN/GiST), статистики и кардинальности — того, чего у H2 просто нет в нужном виде.

Когда же H2 уместен? Для **срезов** и лёгких компонентных тестов, где вы проверяете маппинг JPA, биндинг DTO, базовую валидацию, не опираясь на специфичный SQL. Также H2 полезен в **юнитах DAO** для проверки синтаксиса «общего» SQL и структурной связанности без дорогого окружения.

Для интеграционных тестов, миграций и производительных DML — лучше **Testcontainers** с той же версией Postgres, что и в проде. Да, это медленнее, но даёт реалистичность: вы видите те же типы, ошибки блокировок, планы выполнения и поведение индексов.

Хорошая практика — держать две дорожки: быстрые `@DataJpaTest` на H2 для «отклика» (если ваш домен не зависит от специфики) и критический набор `@DataJpaTest`/`@SpringBootTest` на **Postgres Testcontainers**. Вторая дорожка покрывает JSONB/array, upsert-ы, индексы, блокировки — всё, что часто «кусает» на проде.

Контейнеры можно сделать переиспользуемыми между тестами и сборками (reuse). В CI это экономит минуты: контейнер тёплый, миграции одни и те же, тесты летят. Важно не смешивать наборы данных между классами — чистите схему.

Заметьте, что скорость контейнера сильно зависит от стратегии миграций. Если вы прогоняете тысячу DDL каждый класс — будет больно. Вынесите миграции в отдельный слой и прогоняйте их один раз на набор, а к тестам подключайте уже «готовую» схему.

И последнее: **ошибка несовместимости — это не каприз**. Если в вашем коде есть `jsonb_path_ops` или `generated always as identity`, забудьте про H2 для этих тестов. Сразу целитесь в Postgres-контейнер — вы сэкономите время команды.

**Gradle (Groovy) — базовые зависимости для этого раздела**

```groovy
dependencies {
    implementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
    implementation "org.springframework.boot:spring-boot-starter-data-jpa"
    runtimeOnly "org.postgresql:postgresql:42.7.4"
    testImplementation "org.springframework.boot:spring-boot-starter-test"
    testImplementation "org.testcontainers:junit-jupiter:1.20.3"
    testImplementation "org.testcontainers:postgresql:1.20.3"
}
```

**Gradle (Kotlin) — эквивалент**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql:42.7.4")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:junit-jupiter:1.20.3")
    testImplementation("org.testcontainers:postgresql:1.20.3")
}
```

**Java — @DataJpaTest на Postgres Testcontainers (рекомендуемый путь)**

```java
package db.realism;

import jakarta.persistence.*;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeAll;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Entity(name = "items")
class Item {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false) String name;
    @Column(columnDefinition = "jsonb") String payload; // хранит JSON-строку
    protected Item() {}
    Item(String name, String payload){ this.name = name; this.payload = payload; }
}

interface ItemRepo extends JpaRepository<Item, Long> {
    List<Item> findByName(String name);
}

@DataJpaTest
class PostgresDataJpaTest {

    static final PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16-alpine");

    @BeforeAll static void start() { pg.start(); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
        r.add("spring.jpa.hibernate.ddl-auto", () -> "create"); // для примера; в жизни миграции
    }

    @Autowired ItemRepo repo;

    @Test
    void persistsAndReadsJsonb() {
        repo.save(new Item("doc", "{\"a\":1}"));
        assertThat(repo.findByName("doc")).hasSize(1);
    }
}
```

**Kotlin — эквивалент @DataJpaTest на Postgres**

```kotlin
package db.realism

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import javax.annotation.Resource

@Entity(name = "items")
class Item(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false) var name: String = "",
    @Column(columnDefinition = "jsonb") var payload: String? = null
)

interface ItemRepo : JpaRepository<Item, Long> {
    fun findByName(name: String): List<Item>
}

@DataJpaTest
class PostgresDataJpaTest {

    companion object {
        @JvmStatic val pg = PostgreSQLContainer("postgres:16-alpine").apply { start() }

        @JvmStatic
        @DynamicPropertySource
        fun props(r: DynamicPropertyRegistry) {
            r.add("spring.datasource.url", pg::getJdbcUrl)
            r.add("spring.datasource.username", pg::getUsername)
            r.add("spring.datasource.password", pg::getPassword)
            r.add("spring.jpa.hibernate.ddl-auto") { "create" }
        }
    }

    @Resource lateinit var repo: ItemRepo

    @Test
    fun persistsAndReadsJsonb() {
        repo.save(Item(name = "doc", payload = """{"a":1}"""))
        assertThat(repo.findByName("doc")).hasSize(1)
    }
}
```

---

## Flyway/Liquibase в тестах: миграции на «чистой» БД, проверка дрейфа схемы.

Тесты должны подниматься на **той же схеме**, что и продакшн. Это означает: миграции (`Flyway` или `Liquibase`) прогоняются на «чистой» БД перед тестами репозиториев. Так вы ловите дрейф схемы, несовместимые DDL и проблемы совместимости драйверов ещё в CI, а не после релиза.

Стратегия простая: в профиле `test` отключить `ddl-auto`, подключить мигратор и указать путь к миграциям из основного артефакта. Контейнер с БД стартует, миграции отрабатывают, тесты бегут по готовой схеме. Это медленнее, чем `create-drop`, но даёт реализм и гарантию воспроизводимости.

Проверка дрейфа — обязательная часть. Для Flyway включите `validateOnMigrate=true` и падение на несовпадении checksum. Для Liquibase используйте `clearCheckSums` и контроль `DATABASECHANGELOG`, а также `diff` против эталонного дампа для тяжёлых проектов.

Миграции — это не только DDL. Часть данных — справочники и конфигурация — тоже должна входить в миграции, чтобы любое окружение получало одинаковую начальную базу. Тесты могут валидировать наличие этих данных (например, «в таблице currency есть USD/EUR»).

Важен **детерминизм**. Для тестов не используйте шаги, зависящие от внешних сервисов или нефункциональных функций (`now()` без контроля тайм-зоны, «случайные» UUID). Всё, что влияет на схему и данные из миграций, должно быть воспроизводимо.

Подход на Liquibase удобен для rollback-сценариев: в тесте прогнали `update`, выполнили проверки, затем `rollback` до tag и проверили «чистоту». Это ценно для сложных изменений, где нужен безопасный откат.

Flyway проще и быстрее, что хорошо для базовых проектов. Если нужен rollback, используйте стратегию **forward-fix**: в случае ошибки — новая миграция, возвращающая схему в рабочее состояние. Тесты фиксируют, что обе миграции играют вместе.

Старайтесь не плодить тысячи файлов в одном каталоге. Разбейте миграции на модули/домены и используйте несколько locations. В тестах это прозрачно: указанные locations прогоняются в том же порядке.

Храните артефакты миграций из CI: «лог миграций» и «schema dump» после тестового апдейта. Это ускоряет расследования и позволяет быстро сравнить схему с продом.

Наконец, соблюдайте дисциплину: любые ручные изменения в тестовой БД — табу. Только миграции. Иначе проверка дрейфа теряет смысл.

**application-test.yml — запуск миграций (Flyway)**

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: none
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
```

**Java — интеграционный тест, который убеждается, что миграции отработали**

```java
package db.migrations;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.beans.factory.annotation.Autowired;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest // контекст поднимет Flyway и применит миграции
class FlywayMigrationsIT {

    @Autowired JdbcTemplate jdbc;

    @Test
    void tablesExistAndBaselineApplied() {
        Integer cnt = jdbc.queryForObject(
            "select count(*) from information_schema.tables where table_name in ('users','orders')", Integer.class);
        assertThat(cnt).isGreaterThanOrEqualTo(2);
    }
}
```

**Kotlin — эквивалент теста для Liquibase**

```kotlin
package db.migrations

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.jdbc.core.JdbcTemplate
import javax.annotation.Resource

@SpringBootTest(properties = [
    "spring.liquibase.enabled=true",
    "spring.jpa.hibernate.ddl-auto=none",
    "spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.yaml"
])
class LiquibaseMigrationsIT {

    @Resource lateinit var jdbc: JdbcTemplate

    @Test
    fun changelogApplied() {
        val applied = jdbc.queryForObject("select count(*) from databasechangelog", Int::class.java)
        assertThat(applied ?: 0).isGreaterThan(0)
    }
}
```

**Пример миграции Flyway (V1__init.sql)**

```sql
create table if not exists users(
  id bigserial primary key,
  email text not null unique,
  created_at timestamp with time zone not null default now()
);
```

**Пример Liquibase (YAML, db/changelog/db.changelog-master.yaml)**

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: you
      changes:
        - createTable:
            tableName: orders
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: user_id, type: BIGINT, constraints: { nullable: false } }
              - column: { name: total_cents, type: BIGINT, constraints: { nullable: false } }
```

---

## Наполнение данных: Database Rider/DBUnit, фабрики данных, snapshot-тесты схемы.

Наполнение данных должно быть **быстрым, воспроизводимым и читаемым**. Три подхода, которые работают в связке: фабрики данных в коде, дата-фреймворки (Database Rider/DBUnit) и снапшоты схемы/метаданных. У каждого — своя роль и зона применения.

Фабрики (Test Data Builder/Mother) — лучший выбор для «сценарных» тестов: они читаемы, живут рядом с тестом и описывают намерение, а не «мегаскрипт» SQL. Они хорошо комбинируются с транзакционными тестами и позволяют создавать сложные графы сущностей.

Database Rider/DBUnit удобны, когда нужно залить **большой пласт детерминированных** данных: справочники, таблицы со множеством связей, стабильные «каталоги». Вы описываете state в YAML/JSON/XML и загружаете его перед тестом; по окончании база возвращается к «чистому» состоянию.

Снапшоты схемы — это не про данные, а про **структуру**. В тяжёлых БД полезно иметь тест, который сравнивает актуальную схему (из `information_schema`) с эталоном (дамп/JSON-описание). Это ловит случайные изменения миграций или ручные правки в средах.

Хорошая стратегия — комбинировать: миграции поднимают схему, Database Rider заливает базовые справочники, фабрики дополняют сценарные данные. Так вы держите баланс скорости и читаемости.

Важно не зависеть от порядка тестов. Любая заливка должна быть идемпотентной или откатываться. Database Rider умеет `cleanBefore`/`cleanAfter`, а фабрики — жить в транзакции с откатом.

Избегайте «непрозрачных» data.sql, общих для всех. Они ломают изоляцию. Лучше локальные файлы или Rider-сценарии, привязанные к конкретному классу/методу.

Не пересеребряйте: иногда дешевле написать пару `repo.save(...)` в тесте, чем заводить новый Rider-файл. Но если один и тот же массив данных нужен десяткам тестов — вынесите его в Rider/DBUnit.

Снапшоты схемы не должны быть хрупкими. Сравнивайте только важное: имена таблиц/колонок, типы, индексы, ограничения. Игнорируйте флаги, зависящие от версии БД. Для простоты можно собрать лёгкий JSON-слепок в рантайме и сравнивать с зафиксированным «золотым» файлом.

Документируйте, что является «источником истины»: миграции. Rider/DBUnit — только для данных. Любые изменения в структуре — через миграции, и снапшот должен это подтвердить.

**Gradle (оба DSL) — Database Rider**

```groovy
dependencies {
    testImplementation "com.github.database-rider:rider-spring:1.42.0"
}
```

```kotlin
dependencies {
    testImplementation("com.github.database-rider:rider-spring:1.42.0")
}
```

**Java — Database Rider: загрузка фикстур**

```java
package db.rider;

import com.github.database.rider.core.api.dataset.DataSet;
import com.github.database.rider.spring.api.DBRider;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@DBRider
class RiderIT {

    @Autowired JdbcTemplate jdbc;

    @Test
    @DataSet(value = "datasets/users.yml", cleanBefore = true, cleanAfter = true)
    void loadsUsers() {
        Integer cnt = jdbc.queryForObject("select count(*) from users where email='seed@example.com'", Integer.class);
        assertThat(cnt).isEqualTo(1);
    }
}
```

**resources/datasets/users.yml**

```yaml
users:
  - id: 1
    email: seed@example.com
    created_at: 2025-01-01T00:00:00Z
```

**Kotlin — «снапшот» схемы через information_schema**

```kotlin
package db.snapshot

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.jdbc.core.JdbcTemplate
import javax.annotation.Resource

@SpringBootTest
class SchemaSnapshotIT {

    @Resource lateinit var jdbc: JdbcTemplate

    @Test
    fun hasExpectedColumns() {
        val cols = jdbc.queryForList("""
            select table_name, column_name, data_type 
            from information_schema.columns 
            where table_name in ('users','orders')
        """.trimIndent())
        val names = cols.map { it["table_name"].toString() + "." + it["column_name"] }
        assertThat(names).contains("users.email", "orders.total_cents")
    }
}
```

---

## Тестирование репозиториев: транзакции, блокировки, индексы, JSONB/array поля.

Репозитории — тонкий слой над БД, и смысл тестов здесь — поймать **семантику** запросов и поведение под транзакциями. Проверяйте не только «найти/сохранить», но и индексы, уникальные ограничения, upsert-ы, блокировки, JSONB/array.

Транзакции важны в негативных сценариях: нарушение уникального индекса, внешнего ключа — нужно вызвать `flush()` и поймать исключение именно там, где оно родится. Иначе ошибка всплывёт на коммите контекста, и тест станет хрупким.

Блокировки — отдельная боль. Смоделируйте «первый поток держит `select for update`, второй пытается обновить» и задайте таймаут (statement/lock). Ожидаемое поведение — второй получает timeout/исключение. Такие тесты предотвращают дедлоки в проде.

Индексы проверяются не по планам (это для перф-тестов), а по **семантике**: уникальный индекс действительно запрещает дубль; частичный индекс разрешает уникальность только по `WHERE is_active`. Такие тесты короткие и полезные.

JSONB/array — сила Postgres. Убедитесь, что маппинг корректен: `@Column(columnDefinition = "jsonb")` + сериализация строки или `hibernate-types` для маппинга объектов. Запросы можно писать нативные с оператором `@>` или `?`/`?|` для массивов.

Upsert — `on conflict do update`. Проверьте, что обновление действительно происходит, и что конфликтный ключ — тот, что вы ожидаете (по уникальному индексу). Важно покрыть «мягкую» идемпотентность.

Отдельно — «медленные» запросы. Вы не измеряете время здесь, но фиксируете **корректность пагинации, сортировки и фильтров**. Это предотвращает логические ошибки, которые потом не чинятся индексом.

Следите за «timezone» при сохранении и выборке `timestamptz`. В тестах установите фиксированную зону и сверяйте ISO-строки, чтобы не ловить сюрпризы.

И наконец, не забывайте про `@Transactional(readOnly = true)` — это влияет на поведение сессии и может менять планы выполнения. Тесты могут проверять, что репозиторий «читает» в read-only, а «пишущий» метод — в обычной транзакции.

**Java — уникальный индекс, flush и JSONB-запрос**

```java
package db.repo;

import jakarta.persistence.*;
import org.hibernate.annotations.Type;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.data.jpa.repository.*;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.beans.factory.annotation.Autowired;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

@Entity(name = "docs")
@Table(name="docs", uniqueConstraints = @UniqueConstraint(name="uk_docs_key", columnNames = "doc_key"))
class Doc {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(name="doc_key", nullable=false) String key;
    @Column(columnDefinition = "jsonb") String payload;
    protected Doc() {}
    Doc(String key, String payload){ this.key=key; this.payload=payload; }
}

interface DocRepo extends JpaRepository<Doc, Long> {

    @Query(value = "select * from docs d where d.payload @> cast(:json as jsonb)", nativeQuery = true)
    List<Doc> findByJsonContains(String json);
}

@DataJpaTest
class RepoIT {

    @Autowired DocRepo repo;
    @Autowired EntityManager em;

    @Test
    @Transactional
    void uniqueIndexViolatesOnFlush() {
        repo.save(new Doc("K1", "{\"a\":1}"));
        repo.save(new Doc("K1", "{\"a\":2}"));
        assertThatThrownBy(() -> { em.flush(); })
            .isInstanceOf(DataIntegrityViolationException.class);
    }

    @Test
    void jsonbContains() {
        repo.save(new Doc("K2", "{\"tags\":[\"a\",\"b\"]}"));
        assertThat(repo.findByJsonContains("{\"tags\":[\"a\"]}")).hasSize(1);
    }
}
```

**Kotlin — массивы и upsert через native query**

```kotlin
package db.repo

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.assertj.core.api.Assertions.assertThatThrownBy
import org.junit.jupiter.api.Test
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query
import org.springframework.transaction.annotation.Transactional
import javax.annotation.Resource

@Entity(name = "articles")
@Table(name = "articles", uniqueConstraints = [UniqueConstraint(name="uk_articles_slug", columnNames=["slug"])])
class Article(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable=false) var slug: String = "",
    @Column(name="tags", columnDefinition = "text[]") var tags: String? = null
)

interface ArticleRepo : JpaRepository<Article, Long> {

    @Query(value = "select * from articles a where a.tags && :arr::text[]", nativeQuery = true)
    fun findAnyTag(arr: String): List<Article>

    @Modifying
    @Transactional
    @Query(value = """
        insert into articles(slug, tags) values (:slug, :tags::text[])
        on conflict(slug) do update set tags = excluded.tags
    """, nativeQuery = true)
    fun upsert(slug: String, tags: String)
}

@DataJpaTest
class ArticleRepoIT {

    @Resource lateinit var repo: ArticleRepo
    @Resource lateinit var em: EntityManager

    @Test
    fun uniqueAndFlush() {
        repo.save(Article(slug = "x", tags = "{a,b}"))
        repo.save(Article(slug = "x", tags = "{c}"))
        assertThatThrownBy { em.flush() }.isInstanceOf(DataIntegrityViolationException::class.java)
    }

    @Test
    fun arrayQueryAndUpsert() {
        repo.upsert("s1", "{a,b}")
        val list = repo.findAnyTag("{b}")
        assertThat(list).isNotEmpty
        repo.upsert("s1", "{c}") // обновит tags
        val updated = repo.findAnyTag("{c}")
        assertThat(updated.first().slug).isEqualTo("s1")
    }
}
```

---

## Производительные DML: батчи, тайм-ауты, долгие миграции в CI.

Интеграционные тесты должны не только «работать», но и выявлять **медленные места**. На уровне тестов мы не измеряем TPS, но ловим конфигурационные ошибки: отсутствие батчей, слишком короткие/длинные тайм-ауты, неидемпотентные долгие миграции.

Для JPA включите батчирование `hibernate.jdbc.batch_size` и отключите ordered inserts/updates при необходимости. В тесте можно выполнить 1000 вставок и проверить, что драйвер не «падает» по тайм-ауту и размер батча реально применяется (через логи JDBC или счётчик вызовов батчей — в простом варианте достаточно sanity-check).

Для чистого JDBC используйте `JdbcTemplate.batchUpdate` — он предсказуем и быстрый. Важно выставить statement timeout и lock timeout (на Postgres — `set local statement_timeout`), чтобы зависшие запросы не подвешивали весь прогон.

Долгие миграции тестируйте как **сухой прогон** (Liquibase `updateSQL`) или как частичный прогон на синтетических данных (например, 100k строк). Цель — поймать грубые ошибки и оценить порядок величин, не устраивая полноценный бенчмарк в CI.

Идемпотентность — ключ к повторяемости. Любой DML в миграциях должен корректно отрабатывать «повторный запуск»: `insert ... on conflict do nothing`, `update ... where not exists`, «мягкие» рекалькуляции. Тесты могут запускать миграцию дважды и утверждать отсутствие ошибок.

Тайм-ауты зависят от среды. В локале можно ставить меньше, в CI — больше. В тестах задавайте явные значения через свойства и ожидайте корректное падение при превышении, чтобы не ловить «вечные зависания».

Не смешивайте «объёмные» DML с DDL в одном релизе. Тесты должны отражать это: миграции DDL проходят быстро, а DML — в отдельном наборе с увеличенными тайм-аутами. Это дисциплинирует релизы и держит окно риска коротким.

Батчи в JPA требуют внимания к генерации ключей (`IDENTITY` vs `SEQUENCE`). Для `IDENTITY` батч работает хуже. В тестах можно использовать последовательности и проверить, что батчи действительно срабатывают (в логе драйвера виден `batch size` > 1).

Помните про индексы: массив апдейтов по колонке без индекса будет медленным, но тест не должен измерять скорость — он должен убедиться, что индекс **существует** перед апдейтом. Это можно зафиксировать тестом-«сторожем» схемы.

И напоследок: большие тестовые вставки на CI делайте **редко** и «за флажком» (tag). Основной прогон должен оставаться быстрым, а «тяжёлый» запускаться nightly.

**Java — батч через JPA и timeout через JdbcTemplate**

```java
package db.batch;

import jakarta.persistence.*;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

import java.util.stream.IntStream;

import static org.assertj.core.api.Assertions.assertThat;

@Entity(name="logs")
class LogEntry {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "logs_seq")
    @SequenceGenerator(name="logs_seq", sequenceName="logs_seq", allocationSize=50)
    Long id;
    String msg;
    protected LogEntry(){}
    LogEntry(String msg){ this.msg=msg; }
}

interface LogRepo extends org.springframework.data.jpa.repository.JpaRepository<LogEntry, Long> {}

@SpringBootTest
@TestPropertySource(properties = {
    "spring.jpa.properties.hibernate.jdbc.batch_size=50",
    "spring.jpa.properties.hibernate.order_inserts=true",
    "spring.jpa.properties.hibernate.generate_statistics=true"
})
class BatchIT {

    @Autowired LogRepo repo;
    @Autowired JdbcTemplate jdbc;

    @Test
    void insertsInBatchesAndRespectsTimeout() {
        IntStream.range(0, 500).forEach(i -> repo.save(new LogEntry("m"+i)));
        // локальный statement_timeout (100ms) для демонстрации синтаксиса
        jdbc.execute("set local statement_timeout = 100ms");
        Integer cnt = jdbc.queryForObject("select count(*) from logs", Integer.class);
        assertThat(cnt).isGreaterThanOrEqualTo(500);
    }
}
```

**Kotlin — batchUpdate и lock timeout**

```kotlin
package db.batch

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.jdbc.core.BatchPreparedStatementSetter
import org.springframework.jdbc.core.JdbcTemplate
import javax.annotation.Resource

@SpringBootTest
class JdbcBatchIT {

    @Resource lateinit var jdbc: JdbcTemplate

    @Test
    fun batchInsertWithTimeout() {
        jdbc.execute("set local statement_timeout = 200ms")
        jdbc.batchUpdate("insert into logs(msg) values (?)", object: BatchPreparedStatementSetter {
            override fun setValues(ps: java.sql.PreparedStatement, i: Int) {
                ps.setString(1, "k$i")
            }
            override fun getBatchSize(): Int = 300
        })
        val cnt = jdbc.queryForObject("select count(*) from logs where msg like 'k%'", Int::class.java) ?: 0
        assertThat(cnt).isEqualTo(300)
    }
}
```

---

## @DataJpaTest vs full context, аудит и временные зоны (timestamp with tz).

`@DataJpaTest` поднимает **только** слой JPA: `EntityManager`, репозитории, транзакции и embedded/настроенную БД. Это быстро и изолированно, идеально для тестов маппинга, запросов, индексов и ограничений. Полный `@SpringBootTest` — для сценариев, где участие других слоёв критично: аудит, слушатели доменных событий, безопасность, многоисточниковые транзакции.

Аудит JPA (`@CreatedDate`, `@LastModifiedDate`) требует включённого `@EnableJpaAuditing` и `AuditingEntityListener`. В `@DataJpaTest` это **не** включается автоматически — его нужно импортировать отдельной конфигурацией. В `@SpringBootTest` — обычно включён в основной конфигурации.

Временные зоны — источник сюрпризов. Для Postgres используйте `timestamp with time zone` (или храните UTC в `timestamp without time zone` осознанно) и фиксируйте зону в приложении/тестах. Проверьте, что сериализация/десериализация и сохранение/чтение дают одинаковые значения в UTC.

Сравнивать `Instant` проще всего. Если вы храните `OffsetDateTime`/`ZonedDateTime`, переводите к `Instant` в проверках — это снимает вопросы локалей и daylight saving. В тестах задайте `Clock.fixed`, если бизнес-логика зависит от «сейчас».

Разница между «срезом» и «полным» проявляется и в кэше второго уровня/статистике Hibernate. Если тест должен проверить кэш, нужен `@SpringBootTest` с соответствующей конфигурацией. `@DataJpaTest` держите для чистого SQL/ORM.

Если у вас многосхемная БД или несколько `DataSource`, `@DataJpaTest` может быть недостаточен — проще поднять полный контекст. Тоже относится к Bean Validation, зависящей от кастомного `MessageSource`.

Тесты аудита должны проверять автоустановку и автозаполнение полей, а также обновление `@LastModifiedDate` при изменении сущности. Отдельно проверьте, что тайм-зона не «дрейфует».

`timestamp with time zone` в Postgres хранит момент, а не смещение. Это удобно: вы сравниваете инстанты. Но сериализация в JSON должна быть в ISO 8601 с `Z`/смещением. Зафиксируйте это контрактом веб-тестов.

И наконец, помните «границу ценности»: `@DataJpaTest` даёт быстрый фидбек и должен покрывать 80% сценариев репозиториев. Остальные 20% — в полном контексте, где играют аудит, безопасность и интеграции.

**Java — @DataJpaTest с аудитом и UTC**

```java
package db.audit;

import jakarta.persistence.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;

import java.time.*;

import static org.assertj.core.api.Assertions.assertThat;

@Entity(name="notes_audit")
@EntityListeners(AuditingEntityListener.class)
class NoteA {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    String text;
    @CreatedDate OffsetDateTime createdAt;
    @LastModifiedDate OffsetDateTime updatedAt;
    protected NoteA(){}
    NoteA(String text){ this.text=text; }
}

interface NoteARepo extends JpaRepository<NoteA, Long> {}

@Configuration
@EnableJpaAuditing(dateTimeProviderRef = "fixedProvider")
class AuditCfg {
    @Bean org.springframework.data.auditing.DateTimeProvider fixedProvider() {
        var fixed = OffsetDateTime.of(2025,1,1,0,0,0,0, ZoneOffset.UTC);
        return () -> java.util.Optional.of(fixed);
    }
}

@DataJpaTest(properties = {
    "spring.jpa.properties.hibernate.jdbc.time_zone=UTC"
})
@Import(AuditCfg.class)
class AuditDataJpaTest {

    @Autowired NoteARepo repo;

    @Test
    void setsAuditFieldsInUtc() {
        var n = repo.save(new NoteA("x"));
        assertThat(n.createdAt.getOffset()).isEqualTo(ZoneOffset.UTC);
        assertThat(n.updatedAt).isNotNull();
    }
}
```

**Kotlin — полноконтекстный тест аудита (SpringBootTest)**

```kotlin
package db.audit

import jakarta.persistence.*
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.config.EnableJpaAuditing
import java.time.OffsetDateTime
import java.time.ZoneOffset
import javax.annotation.Resource

@Entity(name="notes_audit2")
@EntityListeners(AuditingEntityListener::class)
class NoteB(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    var text: String = "",
    @CreatedDate var createdAt: OffsetDateTime? = null,
    @LastModifiedDate var updatedAt: OffsetDateTime? = null
)

interface NoteBRepo : JpaRepository<NoteB, Long>

@Configuration
@EnableJpaAuditing
class BootAuditCfg {
    @Bean
    fun dateTimeProvider() = org.springframework.data.auditing.DateTimeProvider {
        java.util.Optional.of(OffsetDateTime.of(2025,1,1,0,0,0,0, ZoneOffset.UTC))
    }
}

@SpringBootTest(classes = [BootAuditCfg::class])
class AuditBootIT {

    @Resource lateinit var repo: NoteBRepo

    @Test
    fun auditFieldsFilled() {
        val n = repo.save(NoteB(text = "y"))
        assertThat(n.createdAt!!.offset).isEqualTo(ZoneOffset.UTC)
        assertThat(n.updatedAt).isNotNull
    }
}
```

**application-test.yml — фиксация тайм-зоны Hibernate**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
```

# 8. Интеграции и сообщения

## HTTP-клиенты: Feign/WebClient под WireMock, задержки/сбои/ретраи/idempotency

Первое: **эмулируем внешний HTTP** через WireMock, чтобы сделать тесты детерминированными. В проде сеть нестабильна, а в тесте нам нужна воспроизводимость: заданные статусы, тела, задержки и сбои. Так мы валидируем ретраи, тайм-ауты и защиту от дублей без реальных зависимостей.

Второе: **выбор клиента**. Для реактивного стека (WebFlux) используйте `WebClient` — гибкая настройка тайм-аутов и встроенные ретраи через Reactor. Для сервлетного — подойдут `RestClient/RestTemplate` или Feign (декларативно, удобно генерировать клиентов из OpenAPI).

Третье: **ретраи должны быть целенаправленными**: повторяем только при временных ошибках (5xx/сетевые исключения), не трогаем 4xx. В реактивном коде это `retryWhen` с фильтром; в Feign — `Retryer`/Resilience4j. В тесте закрепляем: «после двух 500 приходит 200, суммарно 3 вызова».

Четвёртое: **idempotency** — обязательна для небезопасных методов (POST). Клиент выставляет `Idempotency-Key`, а провайдер гарантирует отсутствие дубликатов при повторах. В тесте проверяем наличие заголовка и число обращений на WireMock.

Пятое: **тайм-ауты**. Разделяйте connect/read/write. Слишком большой — зависание потоков, слишком маленький — ложные фейлы. В тесте сценарий «ответ через 300 мс при тайм-ауте 100 мс» должен падать контролируемо и логировать причину.

Шестое: **деградация и fallback**. Если после N попыток сервис недоступен — возвращаем частичный ответ/кэш/дефолт. Тест должен подтверждать, что код не «зависает» и отдаёт предсказуемую ошибку либо degrade-значение.

Седьмое: **наблюдаемость как контракт**. Счётчик попыток, таймер длительности, теги `outcome=SUCCESS/ERROR`. В тесте сверяем инкремент таймера/счётчиков, чтобы не потерять метрики при рефакторинге.

Восьмое: **структура стаба WireMock** — сценарии (`scenarioName/state`) для последовательности ответов, матчинг по заголовкам/телу, `withFixedDelay` для латентности, `fault()` для обрыва TCP. Это покрывает реальные сбои без сложной инфраструктуры.

Девятое: **логирование запросов/ответов**. Не пишите PII/секреты, но фиксируйте URL, метод, статус, размер тела и корреляционный `X-Request-Id`. Тест может проверять присутствие корреляционного ID.

Десятое: **черные-ящики vs срезы**. Бóльшая часть — срез с WireMock; несколько «дымов» можно прогонять через `RANDOM_PORT` как чёрный ящик, если важно проверить сетевые заголовки (gzip, proxy).

**Gradle (Groovy)**

```groovy
dependencies {
  implementation "org.springframework.boot:spring-boot-starter-webflux"
  testImplementation "org.springframework.boot:spring-boot-starter-test"
  testImplementation "com.github.tomakehurst:wiremock-jre8:2.35.1"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-webflux")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("com.github.tomakehurst:wiremock-jre8:2.35.1")
}
```

**Java — WebClient + WireMock, ретраи и Idempotency**

```java
package http.client;

import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;
import reactor.util.retry.Retry;

import java.time.Duration;
import java.util.UUID;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

class ExternalClient {
  private final WebClient client;
  ExternalClient(String baseUrl, Duration timeout) {
    var http = HttpClient.create().responseTimeout(timeout);
    this.client = WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(http))
        .baseUrl(baseUrl)
        .build();
  }
  String getPing() {
    return client.get().uri("/api/ping")
        .header("Idempotency-Key", UUID.randomUUID().toString())
        .retrieve()
        .bodyToMono(String.class)
        .retryWhen(Retry.fixedDelay(2, Duration.ofMillis(50)).filter(ex -> true))
        .block(Duration.ofSeconds(2));
  }
}

public class ExternalClientTest {
  @RegisterExtension
  static WireMockExtension wm = WireMockExtension.newInstance().options(options().dynamicPort()).build();

  @Test
  void retries500Then200_and_setsIdempotencyKey() {
    wm.stubFor(get("/api/ping").inScenario("s").whenScenarioStateIs(STARTED)
        .willReturn(aResponse().withStatus(500)).willSetStateTo("next"));
    wm.stubFor(get("/api/ping").inScenario("s").whenScenarioStateIs("next")
        .willReturn(aResponse().withStatus(200).withBody("pong")));
    var c = new ExternalClient(wm.getRuntimeInfo().getHttpBaseUrl(), Duration.ofSeconds(1));
    assertThat(c.getPing()).isEqualTo("pong");
    wm.verify(2, getRequestedFor(urlEqualTo("/api/ping")).withHeader("Idempotency-Key", matching(".+")));
  }
}
```

**Kotlin — эквивалент**

```kotlin
package http.client

import com.github.tomakehurst.wiremock.client.WireMock.*
import com.github.tomakehurst.wiremock.junit5.WireMockExtension
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.RegisterExtension
import org.springframework.web.reactive.function.client.WebClient
import reactor.util.retry.Retry
import java.time.Duration
import java.util.*

class KClient(private val base: String) {
  private val client = WebClient.builder().baseUrl(base).build()
  fun ping(): String = client.get().uri("/api/ping")
    .header("Idempotency-Key", UUID.randomUUID().toString())
    .retrieve().bodyToMono(String::class.java)
    .retryWhen(Retry.fixedDelay(2, Duration.ofMillis(50))).block(Duration.ofSeconds(2))!!
}

class KClientTest {
  companion object {
    @JvmField @RegisterExtension
    val wm: WireMockExtension = WireMockExtension.newInstance().options(options().dynamicPort()).build()
  }
  @Test
  fun retriesAndHeader() {
    wm.stubFor(get("/api/ping").inScenario("s").whenScenarioStateIs(STARTED)
      .willReturn(aResponse().withStatus(500)).willSetStateTo("next"))
    wm.stubFor(get("/api/ping").inScenario("s").whenScenarioStateIs("next")
      .willReturn(aResponse().withStatus(200).withBody("pong")))
    val c = KClient(wm.runtimeInfo.httpBaseUrl)
    assertThat(c.ping()).isEqualTo("pong")
    wm.verify(2, getRequestedFor(urlEqualTo("/api/ping")).withHeader("Idempotency-Key", matching(".+")))
  }
}
```

---

## Kafka: Embedded vs Testcontainers, сериализация (Avro/JSON), DLQ/повторы

Первое: **две стратегии окружения**. `@EmbeddedKafka` — быстрый брокер «в памяти», хорошо для логики слушателей/продьюсеров. Testcontainers Kafka — реальный брокер в Docker, медленнее, но максимально совместим с продом (ACL, транзакции, idempotence).

Второе: **сериализация**. JSON проще для старта, Avro/Schema Registry — для строгих контрактов и эволюции. В тестах без Registry начните с JSON; для Avro используйте mock-registry или Testcontainers-реестр, иначе клиенты не поднимутся.

Третье: **повторы и DLQ**. В Spring Kafka используйте `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`: N ретраев — и сообщение публикуется в `<topic>.DLT`. Тест: слушатель бросает исключение → сообщение оказывается в DLQ-топике.

Четвёртое: **идемпотентность продьюсера** (`enable.idempotence=true`, `acks=all`). Это защищает от дублей при ретраях. В тесте достаточно проверить успешную отправку и отсутствие ошибок при временных сбоях.

Пятое: **конкурентность**. Масштабируйте слушателей по числу партиций. В тесте закладывайте ожидания без `sleep`: `KafkaTestUtils`/poll с тайм-аутами, чтобы избежать флейки.

Шестое: **защита от «ядовитых» сообщений**. Неверный формат не должен бесконечно перерабатываться. Конфиг `DefaultErrorHandler` + `BackOff` и публикация в DLQ — обязательный контракт. Тест закрепляет именно это.

Седьмое: **наблюдаемость**. Метрики продьюсера/консюмера (исправленные/ошибочные сообщения, лаги) и логи важны для SRE. Тест пусть хотя бы проверяет инкремент счётчиков ошибок, если у вас есть обёртка с Micrometer.

Восьмое: **заказ доставки**. Если порядок важен — одна партиция или ключевание с фиксированным ключом. В тесте проверяем последовательность через партицию и ключ.

Девятое: **транзакции Kafka** — редки, но возможны для «exactly-once» с БД. Для интеграционных тестов — тяжёлые; чаще ограничиваемся «at-least-once» + идемпотентные хендлеры.

Десятое: **конфигурация в тестах**. Не перегружайте: один брокер, небольшой тайм-аут, минимум тем. Ровно столько, чтобы зафиксировать контракт.

**Gradle (Groovy)**

```groovy
dependencies {
  implementation "org.springframework.kafka:spring-kafka"
  testImplementation "org.springframework.kafka:spring-kafka-test"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
  implementation("org.springframework.kafka:spring-kafka")
  testImplementation("org.springframework.kafka:spring-kafka-test")
}
```

**Java — EmbeddedKafka: DLQ после ошибок**

```java
package kafka.test;

import org.apache.kafka.clients.admin.NewTopic;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.*;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.support.serializer.JsonSerializer;
import org.springframework.kafka.test.EmbeddedKafkaBroker;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.util.backoff.FixedBackOff;

import java.util.Map;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders","orders.DLT"})
class KafkaDlqIT {

  record Order(long id) {}
  static final ArrayBlockingQueue<Order> DLQ = new ArrayBlockingQueue<>(1);

  @Configuration
  static class Cfg {
    @Bean NewTopic t1(){ return new NewTopic("orders",1,(short)1); }
    @Bean NewTopic t2(){ return new NewTopic("orders.DLT",1,(short)1); }
    @Bean ProducerFactory<String, Order> pf(EmbeddedKafkaBroker b) {
      return new DefaultKafkaProducerFactory<>(Map.of(
        org.apache.kafka.clients.producer.ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, b.getBrokersAsString()
      ), new org.apache.kafka.common.serialization.StringSerializer(), new JsonSerializer<>());
    }
    @Bean KafkaTemplate<String, Order> kt(ProducerFactory<String, Order> pf){ return new KafkaTemplate<>(pf); }
    @Bean ConcurrentKafkaListenerContainerFactory<String, Order> lcf(EmbeddedKafkaBroker b, KafkaTemplate<String, Order> kt) {
      var cf = new DefaultKafkaConsumerFactory<String, Order>(Map.of(
        ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, b.getBrokersAsString(),
        ConsumerConfig.GROUP_ID_CONFIG, "g"
      ), new StringDeserializer(), new org.springframework.kafka.support.serializer.JsonDeserializer<>(Order.class,false));
      var f = new ConcurrentKafkaListenerContainerFactory<String, Order>();
      f.setConsumerFactory(cf);
      f.setCommonErrorHandler(new DefaultErrorHandler(new DeadLetterPublishingRecoverer(kt), new FixedBackOff(10,1)));
      return f;
    }
  }

  @KafkaListener(topics="orders", groupId="g")
  void consume(Order o){ throw new RuntimeException("fail"); }

  @KafkaListener(topics="orders.DLT", groupId="g")
  void dlq(Order o){ DLQ.add(o); }

  @Test
  void goesToDlq(@org.springframework.beans.factory.annotation.Autowired KafkaTemplate<String, Order> kt) throws Exception {
    kt.send("orders", new Order(42));
    var got = DLQ.poll(3, TimeUnit.SECONDS);
    assertThat(got).isNotNull();
    assertThat(got.id()).isEqualTo(42);
  }
}
```

**Kotlin — минимальная отправка/чтение (в связке с EmbeddedKafka)**

```kotlin
package kafka.test

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.kafka.core.KafkaTemplate
import javax.annotation.Resource

class SimpleKafkaKotlinTest {
  @Resource lateinit var kafkaTemplate: KafkaTemplate<String, String>
  @Test
  fun sendSmoke() {
    val res = kafkaTemplate.send("orders", "ok").completable().join()
    assertThat(res.recordMetadata.topic()).isEqualTo("orders")
  }
}
```

---

## JMS/RabbitMQ/MQ: слушатели, транзакционность, подтверждения, редоставки

Первое: в **RabbitMQ** нам важны подтверждения (`ack`/`nack`), политика повторной доставки и DLQ. По умолчанию авто-подтверждение может «терять» сообщения при падении обработчика. В тестах включаем MANUAL и управляем ack/nack.

Второе: **DLX/DLQ**. Очередь настраиваем с `x-dead-letter-exchange` и `x-dead-letter-routing-key`. При ошибках/`nack requeue=false` сообщение уходит в DLQ. Тест должен это фиксировать.

Третье: **ограничение повторов**. В RabbitMQ это обычно политика очереди (TTL + max-length) или логика слушателя (nack без requeue). Проще и детерминированнее — не ре-кьюить при фатальной ошибке.

Четвёртое: **идемпотентность хендлера**. Повторы возможны всегда; обработчик не должен генерировать побочные эффекты повторно. В тесте — дважды отправить одно и то же и убедиться, что побочный эффект с ключом не дублируется.

Пятое: **наблюдаемость**. Логируем deliveryTag, routingKey, время обработки; метрики успехов/ошибок. Тест может проверять ключевые логи/счётчики.

Шестое: **prefetch** и конкурентность. Ставьте `prefetch=1` для упорядоченной нагрузки. В тесте это помогает избежать гонок и флейки.

Седьмое: **порядок и транзакции**. В RabbitMQ редко включают транзакции; подтверждения каналов достаточно. Если нужна именно транзакционность — используйте `Channel.Tx` осознанно, но в интеграционных тестах это избыточно.

Восьмое: **TTL/делая**. Сообщения должны умирать по TTL, если потребитель недоступен. В тестах можно выставить маленький TTL и убедиться, что сообщение ушло в DLQ/удалилось.

Девятое: **идентификаторы сообщений** (`messageId`, `correlationId`) — база для идемпотентности и трассировки. В тестах проверяйте, что продьюсер выставляет их.

Десятое: **простота окружения**. Для скорости можно использовать Testcontainers RabbitMQ; для совсем быстрых smoke — мок шаблонов, но для подтверждений лучше реальный брокер.

**Gradle (Groovy)**

```groovy
dependencies {
  implementation "org.springframework.boot:spring-boot-starter-amqp"
  testImplementation "org.springframework.boot:spring-boot-starter-test"
  testImplementation "org.testcontainers:rabbitmq:1.20.3"
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-amqp")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.testcontainers:rabbitmq:1.20.3")
}
```

**Java — RabbitMQ: MANUAL ack и DLQ**

```java
package mq.rabbit;

import org.junit.jupiter.api.Test;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.listener.SimpleRabbitListenerContainerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.*;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class RabbitDlqIT {

  static final ArrayBlockingQueue<String> DLQ = new ArrayBlockingQueue<>(1);

  @Configuration
  static class Cfg {
    @Bean Queue q() { return QueueBuilder.durable("q.main")
      .withArgument("x-dead-letter-exchange","")
      .withArgument("x-dead-letter-routing-key","q.main.dlq")
      .build(); }
    @Bean Queue dlq(){ return QueueBuilder.durable("q.main.dlq").build(); }
    @Bean SimpleRabbitListenerContainerFactory factory(ConnectionFactory cf) {
      var f = new SimpleRabbitListenerContainerFactory();
      f.setConnectionFactory(cf);
      f.setAcknowledgeMode(AcknowledgeMode.MANUAL);
      f.setDefaultRequeueRejected(false);
      return f;
    }
  }

  @RabbitListener(queues="q.main", containerFactory="factory")
  void consume(org.springframework.amqp.core.Message msg, com.rabbitmq.client.Channel ch) throws Exception {
    // имитируем сбой и отправку в DLQ
    ch.basicNack(msg.getMessageProperties().getDeliveryTag(), false, false);
  }

  @RabbitListener(queues="q.main.dlq")
  void dlq(String body){ DLQ.add(body); }

  @Autowired RabbitTemplate rt;

  @Test
  void goesToDlq() throws Exception {
    rt.convertAndSend("", "q.main", "payload");
    assertThat(DLQ.poll(2, TimeUnit.SECONDS)).isEqualTo("payload");
  }
}
```

**Kotlin — минимальный слушатель DLQ**

```kotlin
package mq.rabbit

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.amqp.rabbit.core.RabbitTemplate
import org.springframework.boot.test.context.SpringBootTest
import javax.annotation.Resource
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.TimeUnit

@SpringBootTest
class RabbitDlqKotlinIT {
  companion object { val DLQ = ArrayBlockingQueue<String>(1) }
  @Resource lateinit var template: RabbitTemplate
  @org.springframework.amqp.rabbit.annotation.RabbitListener(queues = ["q.main.dlq"])
  fun dlq(body: String) { DLQ.add(body) }
  @Test
  fun sendToDlq() {
    template.convertAndSend("", "q.main.dlq", "x")
    assertThat(DLQ.poll(2, TimeUnit.SECONDS)).isEqualTo("x")
  }
}
```

---

## Расписание/асинхронщина: @Async/@Scheduled, Awaitility, тестовый TaskExecutor/Clock

Первое: асинхронный код делает тесты **недетерминированными**, если ждать «наугад». Решение — Awaitility: ждём условие с тайм-аутом, а не спим фиксированное время.

Второе: `@Async` запускается на `TaskExecutor`. В тестах используйте **однопоточный** executor — меньше гонок, повторяемые результаты. Можно внедрить его как тестовый бин.

Третье: `@Scheduled` проверяем короткими интервалами и счётчиком вызовов. Не меряем производительность — только тот факт, что задача стартует и живёт.

Четвёртое: **время**. Замените `Clock` на `Clock.fixed` в тестах, чтобы «сейчас» было константным. Так уходят флейки из-за тайм-зоны/летнего времени.

Пятое: **избегайте shared state** между тестами. Очереди/счётчики — поля экземпляра тестового бина, а не статические глобальные переменные.

Шестое: **очищайте планировщик**. Если вы создаёте задачи на fixedDelay, убедитесь, что тестовый контекст закрывается и задачи прекращаются — иначе «фон» мешает соседним тестам.

Седьмое: **наблюдаемость**. Записывайте Observation/Meter вокруг задач — в тесте легко проверить инкремент, что подтверждает «живость» расписания.

Восьмое: **исключения в задачах**. По умолчанию Spring логирует, но не падает сборка. Убедитесь, что критичные ошибки не «теряются»: тест может инжектить кастомный `ErrorHandler` и проверять, что он вызван.

Девятое: **реактивные задачи** можно тестировать виртуальным временем Reactor, но если код смешанный — абстрагируйте `Clock` и не усложняйте.

Десятое: **границы пользы**. Не превращайте интеграции в нагрузочные тесты. Наша цель — контракт и жизнеспособность, а не TPS.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies { testImplementation "org.awaitility:awaitility:4.2.0" }
```

```kotlin
dependencies { testImplementation("org.awaitility:awaitility:4.2.0") }
```

**Java — @Async + @Scheduled + Awaitility + Clock**

```java
package async.sched;

import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.*;
import org.springframework.scheduling.annotation.*;
import org.springframework.stereotype.Service;

import java.time.*;
import java.util.concurrent.*;

import static java.util.concurrent.TimeUnit.SECONDS;
import static org.assertj.core.api.Assertions.assertThat;

@Configuration @EnableAsync @EnableScheduling
class AsyncCfg {
  @Bean Executor taskExecutor(){ return Executors.newSingleThreadExecutor(); }
  @Bean Clock clock(){ return Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC); }
}

@Service
class Jobs {
  private final Clock clock;
  final BlockingQueue<Instant> done = new ArrayBlockingQueue<>(4);
  Jobs(Clock clock){ this.clock = clock; }
  @Async public void runAsync(){ done.add(Instant.now(clock)); }
  @Scheduled(fixedDelay = 50) public void tick(){ done.add(Instant.now(clock)); }
}

@SpringBootTest(classes = AsyncCfg.class)
class AsyncIT {
  @org.springframework.beans.factory.annotation.Autowired Jobs jobs;
  @Test void fires() {
    jobs.runAsync();
    Awaitility.await().atMost(2, SECONDS).untilAsserted(() -> assertThat(jobs.done).isNotEmpty());
  }
}
```

**Kotlin — эквивалент**

```kotlin
package async.sched

import org.awaitility.Awaitility.await
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.annotation.EnableAsync
import org.springframework.scheduling.annotation.EnableScheduling
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.Instant
import java.time.ZoneOffset
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.Executors
import javax.annotation.Resource

@Configuration @EnableAsync @EnableScheduling
class KAsyncCfg {
  @Bean fun exec() = Executors.newSingleThreadExecutor()
  @Bean fun clock(): Clock = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC)
}
@Service
class KJobs(private val clock: Clock) {
  val done = ArrayBlockingQueue<Instant>(4)
  @org.springframework.scheduling.annotation.Async fun runAsync() { done.add(Instant.now(clock)) }
  @Scheduled(fixedDelay = 50) fun tick(){ done.add(Instant.now(clock)) }
}
@SpringBootTest(classes = [KAsyncCfg::class])
class KAsyncIT {
  @Resource lateinit var jobs: KJobs
  @Test fun fires() { jobs.runAsync(); await().until { !jobs.done.isEmpty() } }
}
```

---

## Кэш/Redis: TTL, инвалидация, консистентность с событиями

Первое: **цель тестов кэша** — предсказуемость: попадание, промах, истечение TTL, инвалидация. Мы не меряем скорость — только контракт поведения.

Второе: **Redis Testcontainers** даёт реалистичный кэш. Конфигурируем Spring Cache с TTL (например, 200 мс), проверяем: первый вызов — обращение к источнику, второй — кэш, после TTL — снова источник.

Третье: **инвалидация**. `@CacheEvict` по ключу/паттерну должен очищать запись. В тесте сразу после `evict` обращение должно снова сходить к источнику.

Четвёртое: **ключи**. Явно фиксируйте ключ (`key="#id"`) и пространство (`value="users"`). Тогда просто проверить TTL в Redis через `RedisTemplate` и не гадать, как сформирован ключ.

Пятое: **событийная консистентность**. Часто мы чистим кэш при событии «данные обновились». В тесте можно напрямую вызвать слушателя события/послать Spring-event и затем проверить промах.

Шестое: **ошибки Redis** не должны валить бизнес-логику: кэш — опционален. Тестируем graceful-деградацию: при недоступном Redis сервис всё равно отвечает, но без кэша.

Седьмое: **наблюдаемость**. Метрики попаданий/промахов полезно иметь (Micrometer для RedisCache не даёт из коробки «hit/miss», но можно обернуть CacheManager). В тесте хотя бы проверяем TTL/ключи.

Восьмое: **бэкграунд-инвалидация** (pub/sub) может быть нестабильной в тестах; для интеграций достаточно точечного `evict`.

Девятое: **размер значений**. В тестах не гоняем большие payload — достаточно маленьких строк, важна семантика.

Десятое: **границы пользы**. Кэш-интеграции — небольшая часть набора. Закрепите контракт и двигайтесь дальше.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies {
  implementation "org.springframework.boot:spring-boot-starter-data-redis"
  testImplementation "org.springframework.boot:spring-boot-starter-test"
  testImplementation "org.testcontainers:redis:1.20.3"
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-data-redis")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.testcontainers:redis:1.20.3")
}
```

**Java — Spring Cache + Redis TTL/evict**

```java
package cache.redis;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cache.annotation.*;
import org.springframework.context.annotation.*;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.StringRedisTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest @EnableCaching
class RedisCacheIT {

  @Configuration
  static class Cfg {
    @Bean LettuceConnectionFactory lcf(){ return new LettuceConnectionFactory(); }
    @Bean org.springframework.cache.CacheManager cm(LettuceConnectionFactory cf){
      var cfg = org.springframework.data.redis.cache.RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(java.time.Duration.ofMillis(200));
      return org.springframework.data.redis.cache.RedisCacheManager.builder(cf).cacheDefaults(cfg).build();
    }
    @Bean StringRedisTemplate rt(LettuceConnectionFactory cf){ return new StringRedisTemplate(cf); }
    @Bean UserService svc(){ return new UserService(); }
  }

  static class UserService {
    int calls = 0;
    @Cacheable(value="users", key="#id")
    public String user(long id){ calls++; return "U"+id; }
    @CacheEvict(value="users", key="#id") public void evict(long id){}
  }

  @org.springframework.beans.factory.annotation.Autowired UserService svc;

  @Test
  void ttlAndEvict() throws Exception {
    assertThat(svc.user(1)).isEqualTo("U1");
    assertThat(svc.user(1)).isEqualTo("U1");
    assertThat(svc.calls).isEqualTo(1);
    Thread.sleep(220);
    assertThat(svc.user(1)).isEqualTo("U1");
    assertThat(svc.calls).isEqualTo(2);
    svc.evict(1);
    assertThat(svc.user(1)).isEqualTo("U1");
    assertThat(svc.calls).isEqualTo(3);
  }
}
```

**Kotlin — эквивалент сервиса**

```kotlin
package cache.redis

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.cache.annotation.CacheEvict
import org.springframework.cache.annotation.Cacheable
import org.springframework.cache.annotation.EnableCaching
import org.springframework.stereotype.Service
import javax.annotation.Resource

@SpringBootTest @EnableCaching
class KRedisCacheIT {
  @Service
  class KUserService {
    var calls = 0
    @Cacheable(value = ["users"], key = "#id") fun user(id: Long) = "U${++calls}".replaceFirst("U","U$id")
    @CacheEvict(value = ["users"], key = "#id") fun evict(id: Long) {}
  }
  @Resource lateinit var svc: KUserService
  @Test
  fun hitsEvict() {
    assertThat(svc.user(2)).startsWith("U2")
    assertThat(svc.user(2)).startsWith("U2")
    svc.evict(2)
    assertThat(svc.user(2)).startsWith("U2")
  }
}
```

---

## Файлы/объектные хранилища (S3/MinIO): местные стораджи и контракты

Первое: реальные S3-интеграции тестируем **локально** через MinIO (совместим с S3 API). Так мы проверяем ключи, метаданные, права без внешней сети.

Второе: **конфигурация клиента**. AWS SDK v2 с `endpointOverride`, `forcePathStyle(true)` и тестовыми кредами. Это даёт идентичный способ подписи (SigV4) и поведение API.

Третье: **основные сценарии**: `put/get`, корректный `Content-Type`, `Content-Disposition` при скачивании. В тесте сравниваем байты и заголовки.

Четвёртое: **ошибки**: чтение несуществующего ключа — 404; истёкший presigned URL — тоже ошибка. Эти ветви стоит закрепить как негативные тесты.

Пятое: **идемпотентность**. Повторная загрузка с тем же ключом — перезапись или запрет, в зависимости от вашего контракта. Тест должен зафиксировать выбранное поведение.

Шестое: **префиксы и ACL**. По умолчанию приватно. Если в проде часть объектов публична — тест подтверждает корректность политик/префиксов (часто это уже e2e-уровень, но smoke можно держать здесь).

Седьмое: **большие объёмы**. В интеграциях не нужны: достаточно коротких байтовых массивов. Нагрузку перенесите в перф-набор.

Восьмое: **наблюдаемость**. Логируйте ключ, размер, длительность; метрики — число операций и ошибки. Тест может проверять хотя бы успех базовой операции.

Девятое: **шифрование/сервер-сайд**. Если используете SSE-S3/SSE-KMS — это лучше тестировать в облаке; локально ограничьтесь базовым протоколом.

Десятое: **миграции хранилища**. Тесты — страховка при смене SDK/версий: поведение контента и заголовков остаётся прежним.

**Gradle (Groovy/Kotlin)**

```groovy
dependencies {
  implementation "software.amazon.awssdk:s3:2.25.62"
  testImplementation "org.springframework.boot:spring-boot-starter-test"
}
```

```kotlin
dependencies {
  implementation("software.amazon.awssdk:s3:2.25.62")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — MinIO + AWS SDK v2: put/get**

```java
package s3.minio;

import org.junit.jupiter.api.Test;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.*;

import java.net.URI;

import static org.assertj.core.api.Assertions.assertThat;

class MinioIT {
  private S3Client s3() {
    return S3Client.builder()
      .endpointOverride(URI.create("http://localhost:9000")) // подставьте из TC/локального MinIO
      .credentialsProvider(StaticCredentialsProvider.create(AwsBasicCredentials.create("minio","minio123")))
      .region(Region.US_EAST_1)
      .forcePathStyle(true)
      .build();
  }
  @Test
  void putAndGet() {
    var s3 = s3();
    var bucket = "test-bucket";
    try { s3.createBucket(CreateBucketRequest.builder().bucket(bucket).build()); } catch (BucketAlreadyOwnedByYouException ignored) {}
    s3.putObject(PutObjectRequest.builder().bucket(bucket).key("greet.txt").contentType("text/plain").build(),
        RequestBody.fromBytes("hello".getBytes()));
    var res = s3.getObject(GetObjectRequest.builder().bucket(bucket).key("greet.txt").build());
    var bytes = res.readAllBytes();
    assertThat(new String(bytes)).isEqualTo("hello");
  }
}
```

**Kotlin — эквивалент**

```kotlin
package s3.minio

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider
import software.amazon.awssdk.core.sync.RequestBody
import software.amazon.awssdk.regions.Region
import software.amazon.awssdk.services.s3.S3Client
import software.amazon.awssdk.services.s3.model.CreateBucketRequest
import software.amazon.awssdk.services.s3.model.GetObjectRequest
import software.amazon.awssdk.services.s3.model.PutObjectRequest
import java.net.URI

class MinioKotlinIT {
  private fun s3(): S3Client = S3Client.builder()
    .endpointOverride(URI.create("http://localhost:9000"))
    .credentialsProvider(StaticCredentialsProvider.create(AwsBasicCredentials.create("minio","minio123")))
    .region(Region.US_EAST_1)
    .forcePathStyle(true)
    .build()

  @Test
  fun putGet() {
    val c = s3()
    val bucket = "test-bucket"
    runCatching { c.createBucket(CreateBucketRequest.builder().bucket(bucket).build()) }
    c.putObject(
      PutObjectRequest.builder().bucket(bucket).key("greet.txt").contentType("text/plain").build(),
      RequestBody.fromBytes("hello".toByteArray())
    )
    val body = c.getObject(GetObjectRequest.builder().bucket(bucket).key("greet.txt").build()).readAllBytes()
    assertThat(String(body)).isEqualTo("hello")
  }
}
```

# 9. Контракты, совместимость и регресс

## Consumer-driven контрактные тесты: Spring Cloud Contract/Pact

Контрактное тестирование помогает «распилить» ответственность между командами: потребитель формулирует ожидания, провайдер гарантирует их выполнение. Подход **consumer-driven** (CDC) уменьшает риск, что интеграции сломаются в момент релиза, потому что контракты валидируются заранее на обоих концах. В экосистеме Spring наиболее распространены два инструмента: Spring Cloud Contract (SCC) и Pact JVM. В обоих случаях идея одна: описать ожидаемое взаимодействие, сгенерировать стабы для потребителя и тест верификации для провайдера.

Практически CDC запускается с потребителя: пишем тест, который обращается не к реальному провайдеру, а к мок-серверу по контракту. В Pact такой тест сам генерирует файл контракта (pact file) — это JSON, куда попадают path, метод, заголовки, тело запроса/ответа и статусы. После прогона файл публикуют в репозиторий контрактов (локально — просто в артефакты сборки). Этот файл — «истина», с которой сверяется провайдер.

На стороне провайдера контракт «проигрывается» один к одному: Pact запускает провайдера в тестовом режиме и подаёт на него все взаимодействия из pact file. Если хоть одна проверка не совпала (другое поле, тип, заголовок, статус) — сборка провайдера «краснеет». Такой цикл делает невозможной публикацию несовместимых изменений «по ошибке».

У SCC похожая механика, но контракты обычно пишут вручную (Groovy/YAML) и из них генерируются и стабы, и provider-тесты. Это удобно, когда потребителей много и вы хотите держать «общие» контракты рядом с провайдером как источник истины. Pact, напротив, оптимален, когда потребителей несколько и они эволюционируют быстрее — они и формируют ожидания.

Ключевой нюанс CDC — **эволюция**. Контракт не должен «ломаться» из-за добавления необязательных полей; лучшая практика — описывать минимально необходимое и использовать матчинги «по схеме», а не «по байтам». В Pact это достигается через matchers (например, «любая строка вида UUID»), в SCC — через регулярки и предикаты.

Контракты — это не только happy-path. Обязательно покрывайте ошибки: 400/401/403/404/409/422/429/5xx. Потребитель должен уметь отличать эти ветви, а провайдер — отдавать их устойчиво. CDC-тесты закрепляют это на артефактах, а не в устной договорённости.

Поток доставки контрактов — важная организационная деталь. Простая схема — артефакты в CI: потребитель публикует pact файлы как build artifact, провайдер забирает их в своей сборке. Более зрелая — независимый «пакт-брокер» с политиками согласования версий. Но даже без брокера CDC приносит пользу, если есть дисциплина.

CDC не отменяет e2e. Он сокращает поверхность риска и ускоряет фидбек, но не проверяет сеть, DNS, TLS, прокси и real-world latency. Поэтому оставляйте небольшие «дымы» e2e, а основную совместимость закрывайте контрактами.

Важный технический штрих: фиксируйте **идемпотентность**. Если потребитель шлёт POST с `Idempotency-Key`, контракт должен требовать заголовок и описывать поведение при ретраях. Это снимет класс ошибок «дубли».

Наконец, следите за читаемостью. Контракт — такой же код: он должен быть минимальным, самодокументирующимся и рассортированным по доменам. Плохой контракт приводит к ложным тревогам и саботажу практики.

**Gradle (Groovy) — Pact JVM (consumer + provider)**

```groovy
dependencies {
  testImplementation platform("org.springframework.boot:spring-boot-dependencies:3.3.4")
  testImplementation "org.springframework.boot:spring-boot-starter-test"
  testImplementation "au.com.dius.pact.consumer:junit5:4.6.11"
  testImplementation "au.com.dius.pact.provider:junit5spring:4.6.11"
}
```

**Gradle (Kotlin) — эквивалент**

```kotlin
dependencies {
  testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.4"))
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("au.com.dius.pact.consumer:junit5:4.6.11")
  testImplementation("au.com.dius.pact.provider:junit5spring:4.6.11")
}
```

**Java (consumer) — тест генерирует pact-файл**

```java
package cdc.pact.consumer;

import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.*;
import au.com.dius.pact.core.model.RequestResponsePact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import java.util.Map;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

@ExtendWith(PactConsumerTestExt.class)
class UserConsumerPactTest {

  @Pact(consumer = "billing-service", provider = "user-service")
  RequestResponsePact pact(PactDslWithProvider p) {
    return p
      .uponReceiving("get user by id")
        .path("/api/users/42").method("GET")
        .willRespondWith().status(200)
        .headers(Map.of("Content-Type", "application/json"))
        .body("{\"id\":42,\"email\":\"u@example.com\"}")
      .toPact();
  }

  @Test
  @PactTestFor(providerName = "user-service", port = "12345")
  void consumer_uses_stub() {
    given().port(12345)
      .get("/api/users/42")
      .then().statusCode(200)
      .body("email", equalTo("u@example.com"));
  }
}
```

**Kotlin (provider) — верификация контракта против живого Spring Boot**

```kotlin
package cdc.pact.provider

import au.com.dius.pact.provider.junit5.PactVerificationContext
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.TestTemplate
import org.junit.jupiter.api.extension.ExtendWith
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProviderVerificationTest {

    @LocalServerPort var port: Int = 0

    @BeforeEach
    fun setup(context: PactVerificationContext) {
        System.setProperty("pact.provider.baseUri", "http://localhost:$port")
        context.setTarget(au.com.dius.pact.provider.junit5.http.HttpTestTarget("localhost", port, "/"))
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider::class)
    fun verify(context: PactVerificationContext) {
        context.verifyInteraction()
    }
}
```

---

## OpenAPI как контракт: валидация ответов, oas-diff/линтеры (Spectral)

OpenAPI — формальный контракт HTTP-API. Если держать спецификацию актуальной, её можно использовать для **автоматической валидации** запросов и ответов в тестах. Это снимает расхождения «документация сказала одно, сервис делает другое» и помогает потребителям генерировать типобезопасные клиенты.

Минимальный уровень — проверять, что каждый публичный эндпоинт соответствует схеме: коды статусов, медиатипы, структура JSON. Это можно делать через фильтры в RestAssured, которые сверяют ответ с OAS, или через валидаторы вроде openapi4j. Плюс — ловим расхождения при каждом PR; минус — важно поддерживать спецификацию в порядке.

Следующий слой — **линтинг**. Spectral проверяет стиль: договорённости по именованию, обязательность описаний, согласованность ошибок (RFC7807), запрет «небезопасных по умолчанию» паттернов. Такой линтер ставят в CI, чтобы спецификация не деградировала.

Для контроля регресса между версиями удобно использовать **oas-diff**: инструмент сравнивает старую и новую спеки и классифицирует изменения как ломающие или нет. Это полезно для release-нотов, но важнее — для **quality-gate** в CI: ломаем — поднимаем мажорную версию, не ломаем — минор/патч.

Спецификация — не только JSON-структуры. В ней лежат и требования безопасности (`securitySchemes`), и заголовки (`Link`, `ETag`), и контент-негациация. Тесты по OAS должны учитывать это: если контракт объявляет `application/problem+json` для ошибок, сервис обязан его отдавать.

Важно отделять **code-first** и **contract-first**. В первом вы «вынимаете» OAS из контроллеров аннотациями, во втором — пишете OAS руками, а потом генерируете заглушки/клиенты. Но независимо от подхода, **валидировать по спецификации** стоит всегда — это снимает классы ошибок «забыли обновить описание».

Когда спецификация растёт, удобно делить её на группы: публичные, внутренние, экспериментальные. В тестах это отражается через разные наборы валидаторов и разные папки спеки, чтобы случайно не «протащить» внутренний эндпоинт наружу.

Хорошая практика — публиковать OAS как артефакт сборки, а не строить её «на лету» из рантайм-инстанса. Тогда валидатор в тестах работает на **точно таком же** файле, который пойдёт клиентам/порталам.

Не забывайте про примеры (`examples`) в OAS: они служат и документацией, и «golden» образцами для тестов. Их можно подставлять в запросы/ответы и валидировать, что сервис реально их понимает.

И наконец, спецификация должна жить рядом с кодом. Любая смена схемы в PR должна одновременно менять реализацию и спецификацию — и линтер/валидатор это приложит к делу.

**Gradle (оба DSL) — валидация через swagger-request-validator + RestAssured**

```groovy
dependencies {
  testImplementation "io.rest-assured:rest-assured:5.5.0"
  testImplementation "com.atlassian.oai:swagger-request-validator-restassured:2.43.2"
}
```

```kotlin
dependencies {
  testImplementation("io.rest-assured:rest-assured:5.5.0")
  testImplementation("com.atlassian.oai:swagger-request-validator-restassured:2.43.2")
}
```

**Java — проверка ответа против OpenAPI**

```java
package oas.validate;

import com.atlassian.oai.validator.restassured.OpenApiValidationFilter;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

class OasValidationIT {

  private final OpenApiValidationFilter filter =
      new OpenApiValidationFilter("src/test/resources/openapi.yaml");

  @Test
  void response_conforms_to_oas() {
    given().filter(filter)
      .get("http://localhost:8080/api/users/42")
      .then().statusCode(200)
      .body("id", equalTo(42));
  }
}
```

**Kotlin — эквивалент**

```kotlin
package oas.validate

import com.atlassian.oai.validator.restassured.OpenApiValidationFilter
import io.restassured.RestAssured.given
import org.junit.jupiter.api.Test

class KOasValidationIT {
    private val filter = OpenApiValidationFilter("src/test/resources/openapi.yaml")

    @Test
    fun conforms() {
        given().filter(filter)
            .get("http://localhost:8080/api/users/42")
            .then().statusCode(200)
    }
}
```

---

## Бек/форвард совместимость DTO/схем; эволюция событий (Kafka) и версионность

Совместимость — это способность **нового кода читать старые данные** (backward) и **старого кода читать новые данные** (forward, чаще частичная). Для REST/JSON это достигается простыми правилами: добавляйте поля только как необязательные, никогда не меняйте смысл существующих, не удаляйте резко. Для десериализации включайте игнорирование неизвестных полей, а для обязательных — дефолты.

События в шине эволюционируют сложнее: у вас есть уже опубликованные сообщения, которые «живут» в топиках и репликах. Поэтому схема события должна быть версионирована и эволюционировать по правилам: добавление поля с default — совместимо; удаление/переименование — нет; изменение типа — почти всегда ломающе. Авро-схемы и реестры помогают формализовать это, но и в JSON можно прожить, если есть дисциплина.

В тестах совместимость проверяется **примером из прошлого**. Держите файлы «старых» JSON/Avro и прогоняйте десериализацию текущим кодом. Для обратной совместимости — генерируйте «новые» объекты и сериализуйте их, затем пытайтесь прочитать «старым» DTO (эмулируйте старую версию) и ожидайте, что важные поля доступны.

Частая ошибка — полагаться на «универсальные» мапперы, а потом ловить падения на неизвестных полях. Включите `FAIL_ON_UNKNOWN_PROPERTIES=false` и добавляйте явные `@JsonIgnoreProperties(ignoreUnknown = true)`. Это дешёвая страховка.

Для событий уместно держать **версии** в заголовке или теле (`type`, `version`). Потребитель должен уметь маршрутизировать обработку: если версия неизвестна — отправить в DLQ/парковку. В тестах создайте старую/новую версии и проверьте корректный разбор.

Forward-совместимость в REST часто означает, что старый клиент видит «лишние» поля, но может продолжать работу. Поэтому сервер должен быть аккуратен с **обязательностью**: не превращайте опциональное вчера в обязательное сегодня, иначе старые клиенты упадут.

Сложный случай — переименование поля. Для REST-контрактов лучше **дублировать** поле на время перехода (старое и новое одновременно), а затем удалить старое в следующем мажоре. В тестах держите оба JSON-варианта и проверяйте, что маппинг корректен.

В Kafka-мире можно использовать «адаптеры» — консьюмер читает старую версию, маппит в новую и публикует дальше. В тестах эмулируйте обе версии и проверьте, что адаптация сохраняет смысл и не теряет данные.

Под конец — документируйте политику эволюции: что такое «breaking», как долго живут деприкейты, каковы сроки и процесс миграции. Тесты — это механика, но правила — договорённость команд.

**Java — back/forward для JSON DTO**

```java
package compat.dto;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class DtoCompatTest {

  @JsonIgnoreProperties(ignoreUnknown = true)
  static class UserV1 {
    public long id;
    public String email;
  }

  @JsonIgnoreProperties(ignoreUnknown = true)
  static class UserV2 {
    public long id;
    public String email;
    public String fullName; // новое необязательное поле
  }

  @Test
  void backward_and_forward() throws Exception {
    var om = new ObjectMapper();
    var oldJson = """
      {"id":42,"email":"u@example.com"}
    """;
    var newJson = """
      {"id":42,"email":"u@example.com","fullName":"U X"}
    """;

    var readByV2 = om.readValue(oldJson, UserV2.class);
    assertThat(readByV2.fullName).isNull();

    var readByV1 = om.readValue(newJson, UserV1.class);
    assertThat(readByV1.email).isEqualTo("u@example.com");
  }
}
```

**Kotlin — события с версией и маршрутизация**

```kotlin
package compat.event

import com.fasterxml.jackson.annotation.JsonIgnoreProperties
import com.fasterxml.jackson.databind.ObjectMapper
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

@JsonIgnoreProperties(ignoreUnknown = true)
data class UserCreatedV1(val id: Long, val email: String)

@JsonIgnoreProperties(ignoreUnknown = true)
data class UserCreatedV2(val id: Long, val email: String, val fullName: String?)

class EventRouter(private val om: ObjectMapper = ObjectMapper()) {
    fun parse(json: String): Any = when {
        "\"version\":2" in json -> om.readValue(json, UserCreatedV2::class.java)
        else -> om.readValue(json, UserCreatedV1::class.java)
    }
}

class EventCompatTest {
    @Test
    fun routesBoth() {
        val r = EventRouter()
        val v1 = r.parse("""{"version":1,"id":1,"email":"e"}""")
        val v2 = r.parse("""{"version":2,"id":2,"email":"e","fullName":"N"}""")
        assertThat(v1).isInstanceOf(UserCreatedV1::class.java)
        assertThat(v2).isInstanceOf(UserCreatedV2::class.java)
    }
}
```

---

## Стабильные фикстуры для SDK/клиентов; тесты на заголовки/коды/идиоматику REST

SDK и клиенты опираются на **стабильные примеры**. Держите «золотые» JSON-фикстуры рядом с тестами и не меняйте их без нужды. Это позволяет в любой момент переиспользовать один и тот же пример для генерации клиентов и для проверок API.

Идиоматика REST — это не только JSON. Правильные коды и заголовки критично важны. Создание ресурса — `201 Created` и `Location` на новый URI. PATCH должен быть идемпотентным по контракту вашей системы и возвращать актуальное представление. Удаление — `204 No Content`. Эти правила надо **зацементировать** тестами, иначе они быстро разъедутся в реалиях рефакторинга.

Пагинация — ещё одна зона контрактов: `Link` с `rel="next"`/`prev"`, `X-Total-Count`, согласованность лимита. Клиентам нужно предсказуемое поведение и стабильные имена полей. Фиксируйте это «голденами» и валидаторами заголовков.

Контент-негациация — требование «поддерживайте `application/json` и `application/problem+json`». Если спецификация говорит об этом, тест обязательно проверяет и успешные, и ошибочные медиатипы. Это влияет на SDK, потому что генераторы ожидают конкретные типы.

Не пренебрегайте **идентификаторами идемпотентности** и корреляции (`Idempotency-Key`, `X-Request-Id`) — клиенты часто используют их напрямую. Фиксируйте обязательность и echo-вида поведение в тестах.

Фикстуры не должны быть «магическими». Добавляйте комментарии в JSON, если формат позволяет, или README рядом: что означает поле, откуда взялось значение. Это снижает путаницу при ревью.

Для локализации сообщений в ответах SDK-уровня используйте коды ошибок, а не тексты. Тесты должны проверять коды/типы, а тексты — только в отдельном наборе локализации, иначе они будут флейкать из-за копирайта.

Держите минимальный, но показательный набор «золотых» примеров: один happy-path и 2-3 ошибки на каждый публичный ресурс. Этого достаточно, чтобы клиенты и документация оставались в синхроне.

И наконец, не забывайте про **кэширование**. Если ресурс кэшируемый, тесты должны фиксировать `ETag`/`Last-Modified` и корректный `304 Not Modified`. Это тоже часть контракта, на которую клиенты рассчитывают.

**Java — проверка REST-идиоматики и «голденов»**

```java
package rest.goldens;

import org.junit.jupiter.api.Test;
import org.springframework.http.HttpHeaders;

import java.nio.file.Files;
import java.nio.file.Path;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

class RestIdiomsIT {

  @Test
  void create_returns_201_with_location_and_body_matches_golden() throws Exception {
    var golden = Files.readString(Path.of("src/test/resources/goldens/user_get.json"));

    var resp = given().contentType("application/json")
      .body("{\"email\":\"u@example.com\"}")
      .post("http://localhost:8080/api/users")
      .then().statusCode(201)
      .header(HttpHeaders.LOCATION, startsWith("http://localhost:8080/api/users/"))
      .extract().asString();

    // простая структурная проверка по «золотому» примеру (можно JSONAssert/JSON Unit)
    org.skyscreamer.jsonassert.JSONAssert.assertEquals(golden, resp, false);
  }
}
```

**Kotlin — проверка заголовков пагинации**

```kotlin
package rest.goldens

import io.restassured.RestAssured.given
import org.hamcrest.Matchers.containsString
import org.junit.jupiter.api.Test

class PaginationHeadersIT {
    @Test
    fun hasLinkAndTotalCount() {
        given().get("http://localhost:8080/api/users?limit=2&offset=0")
            .then().statusCode(200)
            .header("Link", containsString("""rel="next""""))
            .header("X-Total-Count", org.hamcrest.Matchers.notNullValue())
    }
}
```

---

## Регрессионные наборы: snapshot-тесты, автоген примеров, golden files

Регрессионные тесты фиксируют **внешний вид** ответа: структура, имена полей, значения по умолчанию. Подход «снапшотов» сохраняет эталонный ответ в файл и сравнивает с текущим. Любое расхождение требует осознанного обновления снапшота. Это дисциплинирует изменения контрактов.

Снапшоты особенно полезны там, где схема большая, а тест писать вручную дорого. Они не заменяют семантические проверки, но дают дешёвый «радар» на случайный дрейф. Хорошо сочетать их с OpenAPI-валидацией: OAS гарантирует типы и коды, снапшоты — фактическое содержимое.

Автоген примеров можно строить из тех же e2e/интеграционных тестов: вызвали API — сохранили тело в `src/test/resources/goldens/...`, но только после ручного ревью. Не автоматизируйте обновление «по умолчанию» — так легко пропустить ломающее изменение.

Надёжность снапшотов зависит от **детерминизма**. Убирайте поля времени/рандома или маскируйте их. Хорошая практика — нормализовать даты к фиксированному `Clock` или удалять нестабильные поля перед сравнением.

Снапшоты — это и инструмент документации. Их можно подключать к порталам/SDK как «пример реального ответа». Поэтому держите их минимальными, но валидными и соответствующими бизнесу.

По мере роста API удобно группировать «золотые» файлы по ресурсам и версиям, а не складывать в одну папку. Так проще отслеживать эволюцию и удалять устаревшие примеры.

В контексте событий (Kafka) снапшоты — это пример payload’ов разных версий. Они помогают увидеть, какие поля добавлялись, и как адаптеры справляются с ними.

Если у вас много клиентских языков, держите общий набор «золотых» и используйте его в генерации SDK/контракт-тестах клиентов. Это снижает рассинхрон между сервисом и библиотеками.

Наконец, снапшоты — часть code review. Любой PR, меняющий их, должен объяснять «почему» и ссылаться на версионную политику API.

**Gradle (оба DSL) — JSON Unit**

```groovy
dependencies { testImplementation "net.javacrumbs.json-unit:json-unit-assertj:2.38.0" }
```

```kotlin
dependencies { testImplementation("net.javacrumbs.json-unit:json-unit-assertj:2.38.0") }
```

**Java — «снапшот» сравнение с нормализацией**

```java
package regress.snapshot;

import net.javacrumbs.jsonunit.assertj.JsonAssertions;
import org.junit.jupiter.api.Test;

import java.nio.file.Files;
import java.nio.file.Path;

class SnapshotIT {
  @Test
  void matchesGolden() throws Exception {
    var actual = """
      {"id":42,"email":"u@example.com","generatedAt":"2025-01-01T00:00:00Z"}
    """;
    var golden = Files.readString(Path.of("src/test/resources/goldens/user_get.json"));

    // удалим нестабильные поля перед сравнением
    actual = actual.replaceAll("\"generatedAt\":\"[^\"]+\"", "\"generatedAt\":\"<ignored>\"");
    var normGolden = golden.replaceAll("\"generatedAt\":\"[^\"]+\"", "\"generatedAt\":\"<ignored>\"");

    JsonAssertions.assertThatJson(actual).isEqualTo(normGolden);
  }
}
```

**Kotlin — запись «золотого» при первом запуске (опционально)**

```kotlin
package regress.snapshot

import net.javacrumbs.json-unit.assertj.assertThatJson
import org.junit.jupiter.api.Test
import java.nio.file.Files
import java.nio.file.Path

class SnapshotKTest {
    @Test
    fun compareOrCreate() {
        val actual = """{"id":7,"email":"k@example.com","generatedAt":"<ignored>"}"""
        val path = Path.of("src/test/resources/goldens/user_get_k.json")
        if (!Files.exists(path)) {
            Files.createDirectories(path.parent); Files.writeString(path, actual)
        }
        val golden = Files.readString(path)
        assertThatJson(actual).isEqualTo(golden)
    }
}
```

---

## Негативные сценарии и лимиты: rate limits, circuit breakers, timeouts

Негативные сценарии — «страховочный трос» API. Если вы не проверяете лимиты, тайм-ауты и деградацию, именно там и случится инцидент. Начинаем с **rate limiting**: при превышении квоты сервис должен отдавать `429 Too Many Requests` и полезные заголовки (`Retry-After`, `X-RateLimit-Remaining`). Тесты должны закрепить не только статус, но и корректное вычисление окна и остатка.

Следующий слой — **circuit breaker**. При сериях сбоев в зависимости сервис обязан «оторваться» и быстро отказывать/отдавать fallback, чтобы не тратить ресурсы на бесполезные ретраи. В Resilience4j это проверяется через конфиг «малого окна» и искусственные сбои; тест фиксирует переходы `CLOSED → OPEN → HALF_OPEN`.

Тайм-ауты — базовая гигиена. Разделяйте `connect`, `read`, `write` и не ставьте «бесконечные». В тестах эмулируйте задержку и проверяйте, что код отказывает за разумное время и делает это **контролируемо**, с предсказуемым форматом ошибки (Problem Details).

Негативы включают и «грязные» входные данные: неверный JSON, большие тела, неподдерживаемые медиатипы. В каждом публичном эндпоинте держите хотя бы один тест на 400/415/413. Это дёшево и ловит много уязвимостей класса «падение на бине».

Для **bulkhead**-ограничений (ограничение параллелизма) достаточно сократить пул до 1–2 и послать несколько запросов; один проходит, остальные получают 503/429 или ждут — в зависимости от политики. Тест должен фиксировать именно выбранное поведение.

Не забывайте про кэш и условные GET в негативе: неправильный `If-Match` должен отдавать `412 Precondition Failed`. Это важнейший кусок «безопасной записи» в REST.

Сетевые заголовки — тоже контракт: при ошибках обязательно `Content-Type: application/problem+json`, корректные `Cache-Control` (обычно no-store), отсутствие «лишних» CORS, если это внутренний эндпоинт. Тесты должны это фиксировать.

Для внешних сервисов держите негативы отдельно: «провайдер недоступен», «провайдер отдаёт 500», «провайдер отвечает 200, но тело невалидное». Ваш код должен отличать эти случаи и вести себя предсказуемо.

Наконец, **идемпотентность ошибок**. Повторный POST с тем же `Idempotency-Key` при прошлой ошибке должен давать тот же результат или чёткий отказ, а не «случайно создать ещё раз». Это тоже закрепляется тестом.

**Gradle (оба DSL) — Resilience4j + RestAssured**

```groovy
dependencies {
  implementation "io.github.resilience4j:resilience4j-spring-boot3:2.2.0"
  testImplementation "io.rest-assured:rest-assured:5.5.0"
}
```

```kotlin
dependencies {
  implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")
  testImplementation("io.rest-assured:rest-assured:5.5.0")
}
```

**Java — circuit breaker и 429 с заголовками**

```java
package negative.cb;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@Configuration
class Cfg {
  @Bean TestController testController(){ return new TestController(); }
}

@RestController
@RequestMapping("/api/neg")
class TestController {
  volatile int fails = 0;

  @GetMapping("/limited")
  public ResponseEntity<Void> limited(@RequestHeader(value="X-Quota", defaultValue = "0") int quota) {
    if (quota <= 0) {
      return ResponseEntity.status(429)
        .header("Retry-After","1")
        .header("X-RateLimit-Remaining","0")
        .build();
    }
    return ResponseEntity.ok().build();
  }

  @GetMapping("/unstable")
  @CircuitBreaker(name = "cb1", fallbackMethod = "fallback")
  public String unstable() {
    fails++;
    throw new RuntimeException("boom");
  }
  public String fallback(Throwable t){ return "fallback"; }
}

@SpringBootTest(classes = Cfg.class,
  properties = {
    "resilience4j.circuitbreaker.instances.cb1.failure-rate-threshold=50",
    "resilience4j.circuitbreaker.instances.cb1.sliding-window-size=2",
    "resilience4j.circuitbreaker.instances.cb1.permitted-number-of-calls-in-half-open-state=1",
    "resilience4j.circuitbreaker.instances.cb1.wait-duration-in-open-state=1s"
})
class NegativeIT {

  @Test
  void rateLimitHeaders() {
    given().get("http://localhost:8080/api/neg/limited")
      .then().statusCode(429)
      .header("Retry-After", equalTo("1"))
      .header("X-RateLimit-Remaining", equalTo("0"));
  }

  @Test
  void circuitBreakerOpensAndFallsBack() {
    given().get("http://localhost:8080/api/neg/unstable").then().statusCode(200).body(equalTo("fallback"));
    given().get("http://localhost:8080/api/neg/unstable").then().statusCode(200).body(equalTo("fallback"));
  }
}
```

**Kotlin — тайм-аут WebClient и Problem Details**

```kotlin
package negative.timeout

import org.junit.jupiter.api.Test
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import reactor.core.publisher.Mono
import reactor.netty.http.server.HttpServer
import java.time.Duration

class TimeoutKTest {
    @Test
    fun readTimeoutProducesProblem() {
        val server = HttpServer.create().port(0).handle { _, res ->
            res.addHeader("Content-Type", "application/json")
            Mono.delay(Duration.ofMillis(300)).then(res.sendString(Mono.just("""{"ok":true}""")))
        }.bindNow()
        val client = WebTestClient.bindToServer().responseTimeout(Duration.ofMillis(100))
            .baseUrl("http://localhost:${server.port()}").build()

        client.get().uri("/slow")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().is5xxServerError // или ваша маппинговая ошибка/Problem
        server.disposeNow()
    }
}
```


# 10. Качество, скорость и CI/CD

## Организация набора: fast unit, medium component, slow integration/e2e; теги и профили

Правильная организация набора тестов начинается с осознанного разбиения по скорости и стоимости. Лёгкие юнит-тесты дают мгновенную обратную связь, их должно быть много и они должны запускаться при каждом изменении. Компонентные тесты со Spring-срезами и локальными двойниками дороже, но всё ещё быстры. Интеграционные и e2e запускаются реже: они проверяют стык систем, но тормозят цикл разработки.

Границу между слоями стоит фиксировать не «по ощущениям», а по критериям окружения. Юнит — без Spring-контекста и без сети, строго в памяти. Компонентный — минимальный Spring-контекст нужного слоя и изоляция внешних зависимостей. Интеграционный — реальная БД через Testcontainers, реальный HTTP клиент/сервер и полноценные миграции.

Чтобы сборка была управляемой, весь набор маркируют тегами JUnit 5. Теги превращаются в «каналы» запуска: быстрый канал для локального цикла, средний для MR/PR, медленный для nightly. При этом сами тесты не дублируются: один набор, разные фильтры.

Весомую роль играют Spring-профили. Профиль `test-fast` конфигурирует in-memory фикстуры, профиль `test-int` включает Testcontainers и реальные миграции. Это избавляет от `if` в коде и позволяет настраивать поведение через `application-*.yml`.

Структура исходников влияет на читаемость и скорость. Отдельный исходный набор `integrationTest` с собственным classpath помогает не случайно тащить тяжёлые зависимости в быстрые тесты. Туда же можно переместить ресурсные файлы миграций, «золотые» фикстуры и контейнерные конфиги.

Важна детерминированность того, что запускается в конкретном пайплайне. Задайте явные Gradle-таски, которые включают/исключают теги, и используйте их последовательно в CI. Тогда новые тесты автоматически попадают в «правильный» канал просто благодаря тегу.

Релевантность каналов поддерживается дисциплиной ревью. Любой тест, который тянет Spring-контекст без нужды, нужно возвращать в юнит-уровень. Любой тест, который лезет в сеть, должен идти в интеграции. Чёткое разделение удерживает сборку быстрой.

Отдельно подумайте о smoke-наборе. Это крошечный набор сквозных проверок, который идёт даже в коротком CI, чтобы поймать грубые ошибки, не дожидаясь nightly. Smoke не заменяет полный прогон, но снижает риск, что «всё зелёное» окажется косметикой.

В коммуникации с командой полезно зафиксировать SLA по времени выполнения для каждого канала. Например, fast ≤ 1 мин, medium ≤ 5 мин, slow ≤ 20 мин. Любой PR, который нарушает SLA, должен либо переносить тест на другой слой, либо оптимизировать конфигурацию.

И, наконец, не переусложняйте. Каналов не должно быть слишком много: три-четыре уровня достаточно для большинства проектов. Чем проще правила, тем легче их соблюдать.

**Gradle (Groovy) — каналы по тегам и отдельный sourceSet**

```groovy
plugins {
    id 'java'
}

sourceSets {
    integrationTest {
        java.srcDir file('src/integrationTest/java')
        resources.srcDir file('src/integrationTest/resources')
        compileClasspath += sourceSets.main.output + configurations.testRuntimeClasspath
        runtimeClasspath += output + compileClasspath
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

tasks.register('testFast', Test) {
    useJUnitPlatform { includeTags 'fast' }
}

tasks.register('testComponent', Test) {
    useJUnitPlatform { includeTags 'component' }
}

tasks.register('integrationTest', Test) {
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    useJUnitPlatform { includeTags 'integration' }
    shouldRunAfter test
}
```

**Gradle (Kotlin) — эквивалент**

```kotlin
plugins {
    java
}

sourceSets {
    create("integrationTest") {
        java.srcDir("src/integrationTest/java")
        resources.srcDir("src/integrationTest/resources")
        compileClasspath += sourceSets["main"].output + configurations["testRuntimeClasspath"]
        runtimeClasspath += output + compileClasspath
    }
}
configurations {
    getByName("integrationTestImplementation").extendsFrom(getByName("testImplementation"))
    getByName("integrationTestRuntimeOnly").extendsFrom(getByName("testRuntimeOnly"))
}
tasks.register<Test>("testFast") {
    useJUnitPlatform { includeTags("fast") }
}
tasks.register<Test>("testComponent") {
    useJUnitPlatform { includeTags("component") }
}
tasks.register<Test>("integrationTest") {
    testClassesDirs = sourceSets["integrationTest"].output.classesDirs
    classpath = sourceSets["integrationTest"].runtimeClasspath
    useJUnitPlatform { includeTags("integration") }
    shouldRunAfter(tasks.test)
}
```

**Java — теги и профили**

```java
package ci.channels;

import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("fast")
class PureUnitTest {
    @Test void sum() { assertThat(1 + 1).isEqualTo(2); }
}

@Tag("component")
@ActiveProfiles("test-fast")
class ComponentSliceTest {
    @Test void placeholder() { assertThat("ok").isEqualTo("ok"); }
}

@Tag("integration")
@ActiveProfiles("test-int")
class IntegrationMarkerTest {
    @Test void placeholder() { assertThat(true).isTrue(); }
}
```

**Kotlin — теги и профили**

```kotlin
package ci.channels

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Tag
import org.junit.jupiter.api.Test
import org.springframework.test.context.ActiveProfiles

@Tag("fast")
class PureUnitKTest {
    @Test fun sum() { assertThat(1 + 1).isEqualTo(2) }
}

@Tag("component")
@ActiveProfiles("test-fast")
class ComponentSliceKTest {
    @Test fun placeholder() { assertThat("ok").isEqualTo("ok") }
}

@Tag("integration")
@ActiveProfiles("test-int")
class IntegrationMarkerKTest {
    @Test fun placeholder() { assertThat(true).isTrue }
}
```

---

## Борьба с flaky: ретраи тестов, timeouts, изоляция по порту/БД, random ports

Флейки — это тесты, которые иногда падают без изменения кода. Главные причины — гонки, недетерминированность времени и конкурирующие ресурсы. Флейки подрывают доверие к проверкам и замедляют поставку, поэтому борьба с ними — приоритетный инженерный процесс.

Первый контур — тайм-ауты вместо бессрочных ожиданий. Любое ожидание в тесте должно иметь предел по времени, и падать контролируемо, если условие не наступило. Это касается HTTP-клиентов, асинхронных очередей, Awaitility и любых блокировок.

Второй контур — изоляция по портам. Никогда не хардкодьте порт сервера и брокера в тестах. Используйте `RANDOM_PORT` в `@SpringBootTest` и динамическую конфигурацию для контейнеров, чтобы несколько прогонов не конфликтовали между собой и с чужими процессами.

Третий контур — изоляция по данным. Каждый тест должен иметь уникальные ключи, имена таблиц/тем, пути к файлам. Это дешёво и снимает целый класс гонок, особенно при параллельном запуске.

Четвёртый контур — ретраи там, где окружение действительно может хлопать. Повторный запуск теста не лечит гонку в коде, но помогает пережить редкую сетевую ошибку или случайный задержавшийся контейнер. Ретраи должны быть целевыми и короткими.

Пятый контур — стабилизация времени. Любой код, зависящий от «сейчас», должен получать время через `Clock`. В тестах заменяйте его на `Clock.fixed`, а не гадайте, сколько спать. Это убирает неуловимые ±1 сек ошибки.

Шестой контур — контроль случайности. Для генераторов данных используйте фиксированный seed, а для property-based подходов фиксируйте упавшие значения. Рандом без seed — источник нестабильности.

Седьмой контур — чистое завершение. Планировщики, executors и потоковые ресурсы должны корректно закрываться по окончании теста. Утечки потоков и задач приводят к зависаниям и случайным падениям соседних тестов.

Восьмой контур — сетевые моки вместо внешней сети. WireMock/MockServer заменяют настоящие HTTP-зависимости и делают ответы предсказуемыми, снимая нестабильность DNS/прокси/сертификации.

Девятый контур — диагностика. Любой флейк должен оставлять достаточно логов и артефактов для анализа: «почему ждали», «сколько попыток», «какой порт занят». Это достигается включением debug-логов в тест-профиле и сохранением артефактов в CI.

Десятый контур — карантин. Если флейк воспроизводится редко и у вас нет быстрых фиксов, временно пометьте тест как `@Tag("flaky")` и исключите из обязательного набора, но обязательно заведите задачу на устранение. Культура важнее героизма.

**Gradle (оба DSL) — плагин retry для нестабильных окружений**

```groovy
plugins { id "org.gradle.test-retry" version "1.5.8" }
test {
    useJUnitPlatform()
    retry {
        maxRetries = 1
        maxFailures = 20
        failOnPassedAfterRetry = false
    }
}
```

```kotlin
plugins { id("org.gradle.test-retry") version "1.5.8" }
tasks.test {
    useJUnitPlatform()
    extensions.getByType(org.gradle.testretry.TestRetryTaskExtension::class).apply {
        maxRetries.set(1)
        maxFailures.set(20)
        failOnPassedAfterRetry.set(false)
    }
}
```

**Java — JUnit 5 Extension для точечных ретраев**

```java
package ci.flaky;

import org.junit.jupiter.api.extension.*;
import java.util.concurrent.atomic.AtomicInteger;

public class RetryOnFailureExtension implements TestExecutionExceptionHandler, BeforeTestExecutionCallback {
    private static final int MAX_RETRIES = 1;
    private final AtomicInteger attempts = new AtomicInteger();

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        attempts.set(0);
    }

    @Override
    public void handleTestExecutionException(ExtensionContext context, Throwable throwable) throws Throwable {
        if (attempts.getAndIncrement() < MAX_RETRIES) {
            context.getRequiredTestMethod().invoke(context.getRequiredTestInstance());
        } else {
            throw throwable;
        }
    }
}
```

```java
package ci.flaky;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(RetryOnFailureExtension.class)
class FlakyRetryTest {
    static int counter = 0;
    @Test void sometimes() {
        counter++;
        assertThat(counter % 2 == 0).isTrue();
    }
}
```

**Kotlin — случайный порт и тайм-аут**

```kotlin
package ci.flaky

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort
import java.net.HttpURLConnection
import java.net.URL
import java.time.Duration

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RandomPortKTest {
    @LocalServerPort var port: Int = 0
    @Test
    fun ping_with_timeout() {
        val url = URL("http://localhost:$port/actuator/health")
        val conn = url.openConnection() as HttpURLConnection
        conn.connectTimeout = Duration.ofSeconds(1).toMillis().toInt()
        conn.readTimeout = Duration.ofSeconds(1).toMillis().toInt()
        assertThat(conn.responseCode).isBetween(200, 503) // демонстрация тайм-аута и изоляции
    }
}
```

---

## Ускорение: параллельные тесты, reusing Testcontainers, разделение по модулям

Ускорение — это комбинация параллельного запуска, переиспользования дорогих ресурсов и архитектуры репозитория. Слепое включение параллельности без изоляции обычно делает хуже, поэтому начинать нужно с детерминизма и уникальных ресурсов.

Параллельный запуск в JUnit 5 включается на уровне платформы. Параллельно можно гонять юниты и часть компонентных тестов, которые не делят глобальные ресурсы. Интеграции с контейнерами лучше ограничивать до уровня «параллельно классы, последовательно методы» или вовсе держать последовательно.

Пул JVM-форков Gradle даёт дополнительный выигрыш на многоядре. Важно подбирать `maxParallelForks` в соответствии с доступной памятью и количеством контейнеров, чтобы не упереться в docker-daemon и планировщик.

Testcontainers можно подружить со скоростью двумя способами. Первый — общий статический контейнер на весь JVM-процесс: `@Container static final ...` и запуск в `@BeforeAll`. Второй — глобальное переиспользование контейнеров через файл `~/.testcontainers.properties` с `testcontainers.reuse.enable=true` и `withReuse(true)` в коде.

Переиспользование критично для БД: прогреть Postgres, применить миграции один раз и дальше запускать десятки классов поверх того же инстанса. Это снимает минуты в CI, особенно в больших проектах с Flyway/Liquibase.

Разделение репозитория на модули — не про эстетику, а про скорость. Модуль «домен без Spring» собирается и тестируется молниеносно; модуль «адаптеры» включает тяжёлые интеграции и гоняется реже. Межмодульные зависимости дают точные инкрементальные сборки.

Отдельно стоит вынести контракты и генерацию клиентов. Эти модули тяжелы на плагины и IO, и не должны каждый раз тормозить изменение бизнес-класса. CI может триггерить их только при изменении соответствующих путей.

Кеширование Gradle и build cache помогают повторно использовать результаты тестов на неизменённых модулях. Это безопасно только для детерминированных тестов, поэтому борьба с флейками — ещё и про экономию времени.

Не забывайте про локальную оптимизацию разработчика. Алгоритм простой: `testFast` по тегам после сохранения, `testComponent` перед коммитом, `integrationTest` перед пушем. Средний набор ловит 80% ошибок, не отрывая от контекста.

И наконец, измеряйте. Добавьте логирование времени задач в Gradle и смотрите, где «горит». Без измерения оптимизация вслепую редко попадает в цель.

**junit-platform.properties — включение параллельности**

```
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.mode.classes.default = concurrent
```

**Gradle (оба DSL) — параллельные форки и системные свойства**

```groovy
test {
    useJUnitPlatform()
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2).toInteger()
    systemProperty "junit.jupiter.execution.parallel.enabled", "true"
}
```

```kotlin
tasks.test {
    useJUnitPlatform()
    maxParallelForks = (Runtime.getRuntime().availableProcessors() / 2).coerceAtLeast(1)
    systemProperty("junit.jupiter.execution.parallel.enabled", "true")
}
```

**Java — статический контейнер с reuse**

```java
package ci.speed;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("integration")
class SharedPgIT {

    static final PostgreSQLContainer<?> PG =
        new PostgreSQLContainer<>("postgres:16-alpine").withReuse(true);

    @BeforeAll static void start() { if (!PG.isRunning()) PG.start(); }

    @Test void works() {
        assertThat(PG.getJdbcUrl()).contains("jdbc:postgresql");
    }
}
```

**Kotlin — модульный smoke-тест**

```kotlin
package ci.speed

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Tag
import org.junit.jupiter.api.Test

@Tag("fast")
class DomainOnlyKTest {
    @Test fun pure() { assertThat("domain").startsWith("dom") }
}
```

---

## Качество: JaCoCo покрытие, mutation testing (PIT), arch-тесты (ArchUnit)

Покрытие кода — не цель, а сигнал. JaCoCo даёт метрики строчного/ветвевого покрытия и подсвечивает слепые зоны. Цель — не догнать 100%, а удерживать разумные пороги и не допускать регресса в критичных областях.

Mutation testing проверяет, что тесты действительно ловят ошибки. Плагин PIT мутирует байткод (меняет сравнения, убирает возвраты) и проверяет, «убили» ли мутацию тесты. Высокая доля «живых» мутаций показывает тесты-манекены, которые ничего не проверяют.

Архитектурные тесты на ArchUnit контролируют зависимости между слоями. Это дешёвый способ не допустить «утечку» инфраструктуры в домен и сращивание контроллеров с репозиториями. Правила живут как обычные JUnit-тесты, запускаются быстро и дают понятные сообщения.

Комбинация этих техник создаёт «решётку» качества. Покрытие гарантирует минимальную видимость, PIT проверяет содержательность ассертов, ArchUnit удерживает архитектуру. В сумме меньше сюрпризов при рефакторинге и масштабировании команды.

Порог JaCoCo должен быть дифференцирован. Для домена он выше, для инфраструктуры — ниже, для авто-сгенерированного кода — исключение. Технически это делается через фильтры и различные правила для разных пакетов.

PIT дорог по времени, поэтому его включают не в каждый PR, а по расписанию и на изменённые модули. В PR достаточно «light»-набора — например, мутации только для доменного модуля или для затронутых классов.

ArchUnit полезен не только для слоёв. На нём удобно проверять запреты вроде «ни один класс из `..controller..` не должен кидать `SQLException`», «в пакете `..api..` нет аннотаций `@Entity`», «названия DTO оканчиваются на `Dto`». Такие правила сохраняют здравый смысл кода.

Метрики качества стоит проговаривать в ревью. Если покрытие упало — объяснить почему, если «живых» мутаций много — дописать тесты. Это поддерживает культуру, а не «гонку процентов».

Наконец, любые правила должны быть достижимы. Если порог завышен, команда начнёт «рисовать» тесты. Лучше рост по шагам и здравый компромисс, чем формальная зелень.

**Gradle (Groovy) — Jacoco + отчёт + правило**

```groovy
plugins { id 'jacoco' }

jacoco {
    toolVersion = "0.8.11"
}
test { useJUnitPlatform() }
jacocoTestReport {
    dependsOn test
    reports { xml.required = true; html.required = true }
}
jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = 'CLASS'
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.60
            }
        }
    }
}
check.dependsOn jacocoTestReport, jacocoTestCoverageVerification
```

**Gradle (Kotlin) — PIT mutation testing**

```kotlin
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}

pitest {
    targetClasses.set(listOf("com.example..*"))
    pitestVersion.set("1.15.8")
    junit5PluginVersion.set("1.2.1")
    threads.set(Runtime.getRuntime().availableProcessors())
    outputFormats.set(listOf("HTML", "XML"))
    timestampedReports.set(false)
}
```

**Java — ArchUnit правило слоёв**

```java
package ci.quality;

import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.library.Architectures;
import org.junit.jupiter.api.Test;

class ArchitectureTest {

    @Test
    void layers() {
        JavaClasses classes = new ClassFileImporter().importPackages("com.example");

        Architectures.layeredArchitecture()
            .layer("Controller").definedBy("..controller..")
            .layer("Service").definedBy("..service..")
            .layer("Repository").definedBy("..repository..")
            .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
            .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
            .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
            .check(classes);
    }
}
```

**Kotlin — ArchUnit дополнительное правило**

```kotlin
package ci.quality

import com.tngtech.archunit.core.importer.ClassFileImporter
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses
import org.junit.jupiter.api.Test

class NamingRulesKTest {
    @Test
    fun controllersShouldNotAccessRepositories() {
        val classes = ClassFileImporter().importPackages("com.example")
        noClasses().that().resideInAnyPackage("..controller..")
            .should().accessClassesThat().resideInAnyPackage("..repository..")
            .check(classes)
    }
}
```

---

## Отчётность: Surefire/Failsafe, Allure/ReportPortal, артефакты логов/дампов

Отчётность отвечает на два вопроса: «что сломалось» и «почему». Даже идеальные тесты бесполезны, если их результаты трудно читать. Поэтому в каждом проекте должен быть единый формат отчётов и привычное место, где их искать.

В мире Gradle стандартный отчёт — HTML/XML от `test`, но он скудный для больших e2e. Allure добавляет шаги, вложенные действия, скриншоты и артефакты. Это удобно и для автотестов API, и для UI, и для интеграционных сценариев с несколькими шагами.

ReportPortal даёт потоковую отправку результатов и анализ флейков. Он полезен в больших организациях, где много параллельных наборов и нужно централизованное наблюдение. Если команда небольшая, Allure часто достаточно.

Архивация артефактов должна быть стандартной. Логи приложения, дампы БД/схемы, WireMock-мэппинги, «золотые» ответы — всё это должно автоматом попадать в «папку отчёта» Gradle и загружаться CI как artifacts. Это экономит часы расследований.

Для диагностики падений имеет смысл увеличивать уровень логирования в профиле тестов и сохранять `debug.log`. Нагрузку на CI это почти не добавляет, а пользу приносит огромную. Особенно, когда нужно понять тайминг ретраев и сетевых ошибок.

Важно уметь «подшивать» внешние отчёты в общий портал. Если контракты гоняются в отдельном пайплайне, итоговый отчёт должен ссылаться на него. Если генерируются SDK, отчёты линтера и генератора должны быть рядом.

Сами тесты можно аннотировать шагами и аттачами. Для Allure это `Allure.step()` и вложения. Описанные шаги превращают сырые логи в историю сценария, понятную человеку на ревью.

При падении интеграционного теста с контейнером сохраняйте `docker logs` соответствующего сервиса. Это легко автоматизировать в `@AfterEach`/`@AfterAll` и значительно ускоряет RCA.

Наконец, нужно помнить про деградацию отчётности. Если отчёты не читают — они перестают быть полезны. Держите их минималистичными, быстрыми и сфокусированными на сути.

**Gradle (оба DSL) — Allure и публикация результатов**

```groovy
plugins { id "io.qameta.allure" version "2.12.0" }

allure {
    version = "2.27.0"
    useJUnit5 { version = "2.27.0" }
}

tasks.test {
    useJUnitPlatform()
    reports.html.required = true
    finalizedBy "allureReport"
}
```

```kotlin
plugins { id("io.qameta.allure") version "2.12.0" }

allure {
    version.set("2.27.0")
    useJUnit5 { version.set("2.27.0") }
}
tasks.test {
    useJUnitPlatform()
    finalizedBy("allureReport")
}
```

**Java — шаги и вложения Allure**

```java
package ci.reports;

import io.qameta.allure.Allure;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class AllureStepsTest {

    @Test
    void scenario() {
        Allure.step("Arrange test data");
        Allure.step("Act: call API", () -> {
            String body = "{\"ok\":true}";
            Allure.addAttachment("response.json", "application/json", body, "json");
            assertThat(body).contains("ok");
        });
        Allure.step("Assert: status and body");
    }
}
```

**Kotlin — копия логов контейнера в артефакты**

```kotlin
package ci.reports

import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Test
import org.testcontainers.containers.GenericContainer
import java.nio.file.Files
import java.nio.file.Path
import kotlin.test.assertTrue

class LogsAttachmentKTest {

    companion object {
        private val svc = GenericContainer("nginx:alpine").withExposedPorts(80).apply { start() }

        @JvmStatic @AfterAll
        fun tearDown() {
            val logs = svc.logs
            Files.writeString(Path.of("build/reports/tests/container-nginx.log"), logs)
            svc.stop()
        }
    }

    @Test
    fun smoke() {
        assertTrue(svc.isRunning)
    }
}
```

---

## Политики в CI: quality gates (SonarQube), pre-commit hooks, запуск контрактов и линтеров

Политики в CI формализуют «минимальные требования» к коду. Quality Gate в SonarQube агрегирует метрики покрытия, дублирования, code smells и security hotspots. Если пороги нарушены — мерж блокируется. Это снимает субъективность ревью и удерживает качество на заданном уровне.

Pre-commit-хуки сокращают цикл обратной связи до локальной машины. Форматирование, линтеры, быстрый `testFast` — всё это лучше ловить до пуша. Хуки не заменяют CI, но экономят время всей команды, потому что в репозиторий попадает меньше «мусора».

Запуск контрактов и линтеров должен входить в стандартный пайплайн. Контрактные тесты провайдера по pact-файлам потребителей и линтер OAS (Spectral) для документации API — типичные примеры. Линтеры не должны тормозить, но и не должны быть опциональными.

Важный момент — последовательность стадий. Сначала быстрые проверки, затем компонентные, затем интеграции. Чем раньше мы узнаем о проблеме, тем дешевле её правка. В больших пайплайнах стоит включать условные переходы: если быстрые проверки «красные», дальше не идём.

Политики должны быть прозрачны для разработчика. Все пороги, теги задач и команды запуска должны быть описаны в README и воспроизводимы локально одной командой Gradle. Если локально нельзя запустить то же, что в CI, процесс быстро расползается.

Секреты и доступы не должны мешать допуску к проверкам. Для SonarQube и внешних брокеров контрактов настройте защищённые переменные в CI и используйте `-D` параметры, не хардкодя токены в репозитории.

Надёжная стратегия кэширования CI ускоряет сборки. Кэш Maven/Gradle, Docker-слои базовых образов, артефакты миграций снижают время обратной связи и расширяют окно для более медленных проверок, которые раньше «не влезали».

Инцидент-drill полезен и для CI. Раз в спринт стоит симулировать падение одного из гейтов и проверять, что пайплайн правильно блокирует релиз и отправляет нотификации. Это повышает доверие к автоматизации.

Не забывайте про чистоту. Периодически пересматривайте пороги и отключайте «шумные» проверки, которые не приносят пользу. Политики должны помогать, а не быть религиозным атрибутом.

И главное — автоматизация всегда должна иметь «красную кнопку». Если поломка в пайплайне мешает критическому фиксу, должна быть понятная процедура временного обхода, оформленная в процесс.

**Gradle (оба DSL) — SonarQube и зависимости от задач качества**

```groovy
plugins { id "org.sonarqube" version "5.1.0.4882" }
sonarqube {
    properties {
        property "sonar.projectKey", "example:service"
        property "sonar.projectName", "example-service"
        property "sonar.javascript.exclusions", "**/*"
    }
}
check.dependsOn "jacocoTestReport"
```

```kotlin
plugins { id("org.sonarqube") version "5.1.0.4882" }
sonarqube {
    properties {
        property("sonar.projectKey", "example:service")
        property("sonar.projectName", "example-service")
        property("sonar.javascript.exclusions", "**/*")
    }
}
tasks.check { dependsOn(tasks.named("jacocoTestReport")) }
```

**Git pre-commit hook — быстрые проверки локально**

```bash
#!/usr/bin/env bash
set -euo pipefail
./gradlew -q spotlessApply || true
./gradlew -q testFast --console=plain
```

**Java — запуск OAS линтера и контрактов из Gradle (псевдо-таск shell)**

```java
package ci.policy;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class CiPolicyDoc {

    @Test
    void describe() {
        String policy = """
            CI stages:
            1) ./gradlew testFast jacocoTestReport
            2) spectral lint openapi.yaml
            3) ./gradlew testComponent
            4) ./gradlew integrationTest
            5) ./gradlew sonarqube
            """;
        assertThat(policy).contains("integrationTest");
    }
}
```

**Kotlin — smoke-проверка наличия нужных env для CI**

```kotlin
package ci.policy

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class EnvCheckKTest {
    @Test
    fun sonarTokenPresentInCI() {
        val ci = System.getenv("CI") != null
        if (ci) {
            assertThat(System.getenv("SONAR_TOKEN")).isNotBlank
        }
    }
}
```