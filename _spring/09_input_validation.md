---
layout: page
title: "Валидация входных данных"
permalink: /spring/validation
---

# 0. Введение

## Что это

Валидация входных данных — это совокупность правил и механизмов, гарантирующих, что всё, что приходит в систему (по HTTP, из очереди, из CLI, из файла), соответствует ожидаемому **формату** и **смыслу** до того, как данные попадут в бизнес-логику и хранилища. Под «входом» понимаются и параметры запроса, и заголовки, и тело, и даже метаданные среды (IP, user-agent).

Валидация не равна очистке (sanitization) и не равна нормализации. Санитизация — это удаление или экранирование опасных фрагментов (например, HTML/JS), нормализация — приведение к канонической форме (обрезка пробелов, перевод в верхний регистр, ASCII-folding). Валидация отвечает на вопрос «данные корректны и допустимы?», а не «как бы их исправить».

Валидация всегда опирается на **контракт**: документация API, OpenAPI-спецификация, договорённости с источником данных или схема (Avro/Proto/JSON Schema). Чем строже и формальнее контракт, тем меньше неопределённости и «серых зон» в коде.

С точки зрения архитектуры валидация — «ворота» на границе доверия (trust boundary). Всё, что пересекает границу из внешнего мира в систему, считается недоверенным и должно быть проверено: формат → семантика → разрешение на действие. Чем раньше отклонить плохой ввод, тем меньше «загрязнение» внутренних слоёв.

Валидация — это не только про REST. Она нужна в обработчиках сообщений (Kafka/JMS), планировщиках задач (cron, batch), админских интерфейсах, импортёрах CSV/Excel, CLI-утилитах. Везде, где вы «проглатываете» чужие данные, должен быть слой проверки.

Важно помнить о **двух уровнях**: технический (синтаксис: типы, длины, шаблоны) и предметный (семантика: допустимые диапазоны, статусы, бизнес-инварианты). Первый слой удобнее выражать аннотациями Jakarta Validation, второй — доменными валидаторами/сервисами.

Для долговечности систем полезно отделять транспортные DTO, на которых висит синтаксическая валидация, от доменных моделей, которые гарантируют инварианты конструкторами/фабриками. Это снижает вероятность «просачивания» мусора внутрь предметной области.

Наконец, валидация должна быть наблюдаемой: количество отклонённых запросов, причины, поля — это продуктовая метрика качества, а не только «технический шум». По ней находят UX-проблемы и атаки.

*Gradle (базовые зависимости):*
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

*Java — минимальная валидация DTO на периметре:*

```java
package com.example.validation.intro;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {
    @PostMapping
    public User create(@RequestBody @jakarta.validation.Valid User in) { return in; }
    public record User(@NotBlank String name, @Email String email, @Positive int age) {}
}
```

*Kotlin — аналог:*

```kotlin
package com.example.validation.intro

import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserController {
    @PostMapping
    fun create(@RequestBody @Valid in: User): User = in
}

data class User(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Positive val age: Int
)
```

---

## Зачем это

Главная причина — **надёжность и безопасность**. Некорректные данные приводят к сбоям, нарушению инвариантов, утечкам и эксплойтам. Ранняя валидация на периметре снижает вероятность того, что внутренние компоненты столкнутся с «ядовитым» вводом и упадут в неожиданных местах.

Валидация экономит деньги. Каждый следующий слой обработки стоит дороже: чем позже вы отфильтруете плохие данные, тем больше ресурсов потратите на бесполезную работу (CPU, IO, транзакции, ретраи). Поэтому правило простое: **раньше отклонил — дешевле обошлось**.

Валидация улучшает опыт пользователей и интеграторов. Машиночитаемые ошибки с точными полями/кодами позволяют быстро исправлять запросы и совершенствовать клиенты. А стандартизованный формат ошибок делает интеграции предсказуемыми.

С точки зрения эксплуатационной стабильности валидация — инструмент шумоподавления. Она снижает поток «бесполезных» запросов до бизнес-логики и баз данных, помогает защититься от «шумных» клиентов, багов фронта, неаккуратных интеграций.

Валидация важна для **эволюции контрактов**. Вы можете принимать новые поля, но игнорировать их до поддержки; или наоборот — начать требовать поле с определённой даты. Чёткие правила валидации и коды ошибок позволяют управлять миграциями.

Соблюдение законов и стандартов (PCI DSS, GDPR, отраслевые регламенты) часто прямо подразумевает проверку и ограничение входящих данных, особенно PII. Валидация — часть комплаенса.

Наконец, валидация — инструмент наблюдаемости доменных проблем: когда всплески ошибок указывают на неверные предположения в продукте (например, люди массово вводят даты рождения в будущем).

*Java — демонстрация возврата понятной ошибки (RFC 7807) при невалидном вводе:*

```java
package com.example.validation.why;

import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
class ValidationErrors {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onInvalid(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(e -> java.util.Map.of("field", e.getField(), "code", e.getCode()))
                .toList());
        return pd;
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.validation.why

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ValidationErrors {
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun onInvalid(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("errors", ex.bindingResult.fieldErrors.map { mapOf("field" to it.field, "code" to it.code) })
        }
}
```

---

## Где используется

Первый очевидный слой — HTTP-периметр: контроллеры MVC/WebFlux. Здесь удобно применять Jakarta Validation на DTO/параметрах и сразу возвращать 400/422 с детальной структурой. Это «пожарный фильтр», который не пропускает мусор внутрь.

Второй слой — обработчики сообщений: Kafka/JMS/NATS. События приходят извне, и их формат/смысл тоже нужно проверять, прежде чем запускать бизнес-процессы. В идеале — ещё до десериализации на основании схемы (Avro/Proto с registry).

Третий слой — batch/CLI/импорт файлов. Даже если источник «доверенный», ошибки формата/данных неизбежны. Валидация тут должна быть быстрой, с отчётами об ошибках и возможностью частичной загрузки (строки с ошибками — в отдельный файл/таблицу).

Четвёртый слой — админские панели и внутренние API. «Мы сами себе клиенты» не освобождает от требований: человеческий фактор создаёт ошибки не реже, чем внешние интеграции. Регрессы фронта/скриптов — частый источник мусора.

Пятый слой — интеграционные шины и API-шлюзы. Часть валидации (размеры, медиатипы, схемы) можно вынести на edge (Envoy/Kong/NGINX), чтобы снимать нагрузку с приложений и унифицировать базовые проверки.

Шестой слой — внутри домена: фабрики/агрегаты/команды. Здесь валидация фиксирует инварианты предметной области, которые не зависят от транспорта (например, «сумма операции не может быть отрицательной», «дата окончания позже даты начала»).

Седьмой слой — база данных: ограничения NOT NULL, UNIQUE, CHECK, FK — это «последняя линия обороны». Даже если баг миновал приложение, БД не даст сохранить плохое состояние.

*Gradle (Kafka для примера):*
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.kafka:spring-kafka'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
}
```

*Java — валидация события в Kafka-listener’е:*

```java
package com.example.validation.where;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class UserEventsConsumer {

    @KafkaListener(topics = "user.created", groupId = "val-demo")
    public void onMessage(@Valid UserCreated evt) {
        // обработка только валидных событий
    }

    public record UserCreated(@NotBlank String id, @NotBlank String email) {}
}
```

*Kotlin — аналог:*

```kotlin
package com.example.validation.where

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class UserEventsConsumer {

    @KafkaListener(topics = ["user.created"], groupId = "val-demo")
    fun onMessage(@Valid evt: UserCreated) { /* ... */ }
}

data class UserCreated(
    @field:NotBlank val id: String,
    @field:NotBlank val email: String
)
```

---

## Какие проблемы/задачи решает

Валидация предотвращает **расползание неконсистентности**. Если пропустить «чуть-чуть кривые» данные, они быстро разойдутся по кэша́м, очередям, индексу поиска и оттуда начнут ронять другие компоненты. Простое «не принять» дешевле, чем «расхлёбывать».

Она повышает **детерминизм контрактов**. Клиенты получают точные коды/поля ошибок, могут автоматически решать — исправлять, повторять, деградировать функцию. Этот детерминизм — основа для ретраев, circuit-breaker’ов и пользовательского UX.

Валидация усиливает **безопасность**: ограничение размеров, структуры и символов защищает от DoS-паттернов (gigantic JSON, regex-бомбы), а строгая типизация — от инъекций. Это особенно важно в публичных API.

Валидация помогает **миграциям**. В переходные периоды правила могут быть мягче (варнинги/депрекейшн-коды), затем — строже. Отдельные коды ошибок позволяют отслеживать готовность клиентов и поэтапно «закручивать гайки».

Внутри домена валидация — способ явно кодировать **инварианты**: вместо «на словах» держать правила в конструкторе/фабрике/методах изменения. Ошибка инварианта — это бизнес-ошибка (422/409), а не 500.

Системно валидация — это ещё и **наблюдаемость**: метрики по причинам отказов, топ-поля, распределение по клиентам. Эти данные кормят продуктовые решения и приоритезацию.

И наконец, валидация снижает **стоимость поддержки**. Чем понятнее и стабильнее причины отказов, тем меньше тикетов уровня «что у вас за 500?». Поддержка сразу видит поле, код и, часто, ссылку на документацию.

*Java — доменный инвариант + защита БД:*

```java
package com.example.validation.problems;

import jakarta.validation.constraints.Positive;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/payments")
class PaymentsController {
    @PostMapping
    public Payment create(@RequestBody Payment in) {
        if (in.amount() <= 0) throw new IllegalArgumentException("amount must be > 0");
        // save -> может кинуть DataIntegrityViolationException при уникальном ключе
        return in;
    }
    record Payment(@Positive long amount, String idempotencyKey) {}
}
```

*Kotlin — аналог + SQL-constraint пример:*

```kotlin
package com.example.validation.problems

import jakarta.validation.constraints.Positive
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/payments")
class PaymentsController {
    @PostMapping
    fun create(@RequestBody inDto: Payment): Payment {
        require(inDto.amount > 0) { "amount must be > 0" }
        return inDto
    }
}

data class Payment(@field:Positive val amount: Long, val idempotencyKey: String)
```

*SQL (PostgreSQL) — последняя линия:*

```sql
ALTER TABLE payments
  ADD CONSTRAINT payments_amount_chk CHECK (amount > 0),
  ADD CONSTRAINT payments_idem_uk UNIQUE (idempotency_key);
```

---

# 1. Принципы и границы доверия

## Где валидировать: на периметре (контроллеры) + в домене (инварианты) + в БД (constraint’ы)

Слои валидации не взаимозаменяемы, они **дополняют** друг друга. Периметр фильтрует синтаксический мусор и очевидные нарушения контракта до входа в бизнес-код. Домен гарантирует инварианты, которые транспорт не знает. База не даёт сохранить плохое состояние, даже если баг прорвался.

На периметре удобны аннотации Bean Validation, лимиты размеров/глубины, проверка медиатипов/заголовков. Здесь же легко реализовать нормализацию (trim/каноникализация), если это оговорено контрактом. Результат — чистые DTO, готовые для бизнес-слоя.

В домене валидация не про «строки и длины», а про смысл: правила переходов состояний, монетарные ограничения, согласованность полей. Это место для фабрик/методов, которые **не создадут/не изменят** агрегат в невалидное состояние.

В БД мы закрепляем инварианты и уникальности: NOT NULL/UNIQUE/CHECK/FK. Они работают независимо от приложения и защищают от гонок/ошибок многопоточности/несинхронных сервисов. Это необходимая страховка.

Важно распределять проверку так, чтобы **не дублировать** сложные правила во многих местах. Старайтесь: транспорт — простая синтаксика; домен — вся семантика; БД — минимальные жёсткие инварианты и ссылочная целостность.

Порядок тоже важен: сначала периметр, затем домен, затем хранилище. Нарушение порядка приводит к «здоровым» 500-кам и тяжёлому разбору инцидентов, когда ошибка домена маскируется ошибкой сериализации или БД.

Тестируйте каждый слой отдельно и вместе: юнит-тестами домен, slice-тестами контроллеров — периметр, интеграцией — БД-ограничения. Это даёт ясную картину, где «дырка».

*Java — трёхслойный пример:*

```java
package com.example.validation.layers;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
class OrdersController {

    @PostMapping
    public ResponseEntity<?> create(@RequestBody @Valid CreateOrder req) {
        Order o = Order.create(req.customerId(), req.amount()); // доменный инвариант внутри
        try {
            // repository.save(o) -> может бросить DataIntegrityViolationException
            return ResponseEntity.status(HttpStatus.CREATED).body(o);
        } catch (DataIntegrityViolationException e) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
            pd.setTitle("Duplicate or FK violation");
            return ResponseEntity.status(HttpStatus.CONFLICT).body(pd);
        }
    }

    public record CreateOrder(@NotBlank String customerId, @Positive long amount) {}
    static class Order {
        final String customerId; final long amount;
        private Order(String c, long a) { this.customerId = c; this.amount = a; }
        static Order create(String c, long a) {
            if (a <= 0) throw new IllegalArgumentException("amount>0");
            return new Order(c, a);
        }
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.validation.layers

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @PostMapping
    fun create(@RequestBody @Valid req: CreateOrder): ResponseEntity<Any> {
        val order = Order.create(req.customerId, req.amount)
        return try {
            // repository.save(order)
            ResponseEntity.status(HttpStatus.CREATED).body(order)
        } catch (_: DataIntegrityViolationException) {
            ResponseEntity.status(HttpStatus.CONFLICT).body(
                ProblemDetail.forStatus(HttpStatus.CONFLICT).apply { title = "Duplicate or FK violation" }
            )
        }
    }
}

data class CreateOrder(@field:NotBlank val customerId: String, @field:Positive val amount: Long)
class Order private constructor(val customerId: String, val amount: Long) {
    companion object { fun create(c: String, a: Long) = if (a <= 0) throw IllegalArgumentException("amount>0") else Order(c, a) }
}
```

*SQL — слой БД:*

```sql
ALTER TABLE orders
  ADD CONSTRAINT orders_amount_chk CHECK (amount > 0),
  ADD CONSTRAINT orders_customer_fk FOREIGN KEY (customer_id) REFERENCES customers(id);
```

---

## Trust boundary: всё извне считается недостоверным; валидация до бизнес-логики

Граница доверия — линия, где ваши гарантии заканчиваются. Всё, что **снаружи** (клиенты, другие сервисы, админские скрипты), потенциально враждебно или ошибочно. Эту границу проводят на самом входе: диспетчер HTTP/фильтр, consumer сообщений, обработчик файлов. За границей — только проверенные данные.

Правило «недоверять вводу» означает: не использовать вход напрямую для опасных операций (SQL, файловая система, шаблоны), не полагаться на «добросовестность» клиента в длинах/типах, всегда проверять MIME/медиатипы и границы размеров. Даже «внутренний» клиент должен считаться недоверенным.

Валидация до бизнес-логики — это не только аннотации. Это и **проверка аутентичности** (подписи HMAC/JWT), и дедуп по Idempotency-Key, и ограничение «скоростей» (rate limit). Всё это — про недоверие к источнику и попытку исчерпывающе проверить то, что пришло.

Сценарии повторов требуют особого внимания: если клиент ретраит POST, важно не выполнить действие повторно. Это тоже часть валидации на границе — проверка и хранение идемпотентных ключей.

Носите минимально необходимый контекст через границу: вырезайте лишние поля, приводите типы, нормализуйте. Чем меньше «сырых» данных проникнет внутрь, тем проще гарантийные обязательства.

Граница доверия — хорошее место для метрик и логирования. Здесь вы видите, что и от кого приходит, и можете быстро выявлять злоупотребления или ошибки интегратора. На этапах внедрения это часто главная точка обратной связи.

И, наконец, оформляйте границу как **повторяемый шаблон**: фильтры/компоненты, подключаемые к каждому входу (HTTP, Kafka). Тогда новые команды не забудут про безопасность и валидацию.

*Java — фильтр валидации Idempotency-Key на периметре:*

```java
package com.example.validation.trust;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class IdempotencyFilter extends GenericFilter {
    private final Set<String> usedKeys = ConcurrentHashMap.newKeySet();

    @Override public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        if ("POST".equalsIgnoreCase(req.getMethod())) {
            String key = req.getHeader("Idempotency-Key");
            if (key == null || key.isBlank()) { send((HttpServletResponse) response, "Missing Idempotency-Key"); return; }
            if (!usedKeys.add(key)) { send((HttpServletResponse) response, "Duplicate Idempotency-Key"); return; }
        }
        chain.doFilter(request, response);
    }

    private void send(HttpServletResponse res, String detail) throws IOException {
        res.setStatus(HttpStatus.CONFLICT.value());
        res.setContentType("application/problem+json");
        String body = "{\"title\":\"Idempotency error\",\"status\":409,\"detail\":\""+detail+"\"}";
        res.getWriter().write(body);
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.validation.trust

import jakarta.servlet.*
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Component
import java.io.IOException
import java.util.concurrent.ConcurrentHashMap

@Component
class IdempotencyFilter : GenericFilter() {
    private val usedKeys = ConcurrentHashMap.newKeySet<String>()

    @Throws(IOException::class, ServletException::class)
    override fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
        val req = request as HttpServletRequest
        if (req.method.equals("POST", ignoreCase = true)) {
            val key = req.getHeader("Idempotency-Key")
            if (key.isNullOrBlank()) return send(response as HttpServletResponse, "Missing Idempotency-Key")
            if (!usedKeys.add(key)) return send(response as HttpServletResponse, "Duplicate Idempotency-Key")
        }
        chain.doFilter(request, response)
    }

    private fun send(res: HttpServletResponse, detail: String) {
        res.status = HttpStatus.CONFLICT.value()
        res.contentType = "application/problem+json"
        res.writer.write("""{"title":"Idempotency error","status":409,"detail":"$detail"}""")
    }
}
```

---

## Слои ответственности: синтаксис/формат → семантика/правила домена → существование ресурсов/прав доступа

Разделение ответственности — способ удержать код простым и проверяемым. **Синтаксический слой** отвечает за типы, длины, паттерны, обязательность полей. Он ничего не знает о бизнес-смысле. Его удобно выражать аннотациями на DTO и простыми фильтрами.

**Семантический слой** — это доменная логика: допустимость комбинаций, диапазоны в зависимости от контекста, переходы состояний, инварианты агрегатов. Здесь живут фабрики/сервисы/валидаторы, которые принимают уже «чистые» DTO и проверяют бизнес-правила.

**Слой существования/прав** отвечает на вопросы «а это вообще есть?» и «а вам можно?». Это проверки на наличие связей в БД/кэше, валидность ссылок на справочники, право пользователя выполнять действие над ресурсом (ACL, ownership, роли).

Такое деление упрощает отладку и тестирование: вы пишете отдельные юниты на синтаксис (DTO + аннотации), отдельные — на семантику (сервисы), отдельные — на проверки существования (репозитории/ACL). И интеграционные тесты склеивают всё вместе.

При таком подходе проще эволюционировать: изменения в справочнике/ACL не затрагивают синтаксические правила, а добавление новой бизнес-ветки не ломает транспорт. Это уменьшает когнитивную нагрузку и скорость регрессов.

Ещё плюс — **прозрачные ошибки**. Синтаксические нарушения → 400/422 с полями; семантические → 422/409 с кодами доменных ошибок; отсутствие/запрет → 404/403. Клиенту проще принимать решения, а продукту — мерить качество.

Наконец, это дружит с DDD: транспортные DTO не «протекают» внутрь, а доменные объекты гарантируют инварианты. База лишь закрепляет критично важное и обеспечивает целостность между агрегатами.

*Java — пример трёх слоёв в одном случае:*

```java
package com.example.validation.layers2;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/transfers")
class TransferController {
    private final TransferService service;
    TransferController(TransferService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<?> create(@RequestBody @Valid NewTransfer req) {
        service.checkSemantics(req); // семантика
        service.checkAccess(req.userId(), req.fromAccountId()); // права/владение
        return ResponseEntity.ok(service.perform(req));
    }

    public record NewTransfer(@NotBlank String userId,
                              @NotBlank String fromAccountId,
                              @NotBlank String toAccountId,
                              @Positive long amount) {}
}

@org.springframework.stereotype.Service
class TransferService {
    void checkSemantics(TransferController.NewTransfer r) {
        if (r.fromAccountId().equals(r.toAccountId())) throw new IllegalArgumentException("same account");
    }
    void checkAccess(String userId, String from) {
        // repo.checkOwner(userId, from) -> 403/404
    }
    Object perform(TransferController.NewTransfer r) { return java.util.Map.of("status", "ok"); }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.validation.layers2

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/transfers")
class TransferController(private val service: TransferService) {

    @PostMapping
    fun create(@RequestBody @Valid req: NewTransfer): ResponseEntity<Any> {
        service.checkSemantics(req)
        service.checkAccess(req.userId, req.fromAccountId)
        return ResponseEntity.ok(service.perform(req))
    }
}

data class NewTransfer(
    @field:NotBlank val userId: String,
    @field:NotBlank val fromAccountId: String,
    @field:NotBlank val toAccountId: String,
    @field:Positive val amount: Long
)

@org.springframework.stereotype.Service
class TransferService {
    fun checkSemantics(r: NewTransfer) {
        require(r.fromAccountId != r.toAccountId) { "same account" }
    }
    fun checkAccess(userId: String, from: String) { /* repo/acl check */ }
    fun perform(r: NewTransfer): Any = mapOf("status" to "ok")
}
```

# 2. Таксономия проверок

## Синтаксические (формат email, длины, regex) и семантические (дата «с-до», валюты, статусы)

Синтаксические проверки отвечают на вопрос «правильно ли это *записано*»: тип, диапазон числа, длина строки, соответствие регулярному выражению, корректная дата. Их удобно выражать средствами Jakarta Bean Validation: `@NotBlank`, `@Email`, `@Size`, `@Pattern`, `@Positive`, `@PastOrPresent` и т. п. Эти проверки быстры и дешёвы — они не требуют БД, сети и бизнес-контекста и могут выполняться прямо в контроллере.

Семантические проверки отвечают на вопрос «это *допустимо* для нашей предметной области»: дата «from» раньше «to», валюта поддерживается в магазине, статус заказа позволяет провести возврат, сумма не превышает лимит клиента. Такие проверки живут в домене и часто требуют внешних справочников/репозиториев или хотя бы доступа к конфигурации.

Разграничение важно для корректного статуса ответа: синтаксические ошибки чаще приводят к `400 Bad Request` или `422 Unprocessable Entity` (если JSON валиден, но значения нарушают правила), тогда как семантические — к `422`/`409 Conflict`/`403 Forbidden` в зависимости от природы нарушения. Это помогает клиентам принимать автоматические решения: исправить форму, повторить позже, запросить права.

Практический совет: держите синтаксис на транспортных DTO, а семантику — в сервисах/агрегатах. Так вы избежите размазывания бизнес-логики по слою представления. При этом, если семантическое правило *простое* и не тянет зависимостей (например, «from < to»), его можно выразить class-level валидатором (см. следующий пункт).

Семантика часто вовлекает «подвижные» правила: лимиты и списки валют меняются со временем. Поэтому не кодируйте их в аннотациях и регулярках — читайте из конфигурации/справочников, держите фичи-флаги и версионируйте контракт. Это снижает количество деплоев «ради одной цифры».

Чётко отделяйте нормализацию от синтаксиса: `trim` и Unicode-каноникализация — это не валидация, а подготовка входа к проверкам. Делайте её до применения синтаксических аннотаций, иначе «лишний пробел» зачем-то превратится в ошибку и ухудшит UX.

Для ошибок используйте машиночитаемые коды. Например, `EMAIL_FORMAT_INVALID` (синтаксис) и `CURRENCY_NOT_SUPPORTED` (семантика). Один тип проблем не должен «маскировать» другой, иначе фронт будет путаться в сценариях.

Наконец, согласуйте и протестируйте границы: максимальные длины, диапазоны чисел, список поддерживаемых кодов ISO. Ошибки именно в таких «мелочах» чаще всего всплывают в проде.

*Gradle (минимум для примеров)*
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — синтаксис в DTO + семантика в сервисе**

```java
package com.example.taxonomy.syntaxsem;

import jakarta.validation.Valid;
import jakarta.validation.ValidationException;
import jakarta.validation.constraints.*;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.Set;

@RestController
@RequestMapping("/api/payments")
public class PaymentController {
    private final PaymentService service;
    public PaymentController(PaymentService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<?> create(@RequestBody @Valid PaymentRequest req) {
        service.checkSemantics(req); // семантика домена
        return ResponseEntity.ok().body(req);
    }
}

@Service
class PaymentService {
    private static final Set<String> SUPPORTED = Set.of("USD", "EUR", "RUB");
    public void checkSemantics(PaymentRequest r) {
        if (!SUPPORTED.contains(r.currency())) {
            throw new ValidationException("Unsupported currency: " + r.currency());
        }
        if (r.fromDate() != null && r.toDate() != null && !r.fromDate().isBefore(r.toDate())) {
            throw new ValidationException("fromDate must be before toDate");
        }
    }
}

record PaymentRequest(
        @NotBlank @Email String email,
        @Positive long amount,
        @NotBlank String currency,
        LocalDate fromDate,
        LocalDate toDate
) {}
```

**Kotlin — аналог**

```kotlin
package com.example.taxonomy.syntaxsem

import jakarta.validation.Valid
import jakarta.validation.ValidationException
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.http.ResponseEntity
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.*
import java.time.LocalDate

@RestController
@RequestMapping("/api/payments")
class PaymentController(private val service: PaymentService) {

    @PostMapping
    fun create(@RequestBody @Valid req: PaymentRequest): ResponseEntity<Any> {
        service.checkSemantics(req)
        return ResponseEntity.ok(req)
    }
}

@Service
class PaymentService {
    private val supported = setOf("USD", "EUR", "RUB")
    fun checkSemantics(r: PaymentRequest) {
        require(supported.contains(r.currency)) { "Unsupported currency: ${r.currency}" }
        if (r.fromDate != null && r.toDate != null && !r.fromDate.isBefore(r.toDate)) {
            throw ValidationException("fromDate must be before toDate")
        }
    }
}

data class PaymentRequest(
    @field:NotBlank @field:Email val email: String,
    @field:Positive val amount: Long,
    @field:NotBlank val currency: String,
    val fromDate: LocalDate?,
    val toDate: LocalDate?
)
```

---

## Межполевые (кросс-field) и агрегатные (на уровне объекта/коллекции)

Межполевые проверки касаются отношения между значениями нескольких полей одного объекта. Типичный пример — «from < to», «если `type=CARD`, то `pan` обязателен», «если `country=US`, то `state` обязателен». Их удобнее описывать class-level валидаторами, потому что одно поле без контекста не отвечает на вопрос «валидно ли».

Агрегатные проверки смотрят шире: на коллекции и составные структуры. Например, элементы списка уникальны, общая сумма строк документа не превышает лимит, все элементы принадлежат одному владельцу. Такие ограничения сложно выразить одиночными аннотациями — нужен валидатор, которому видна коллекция целиком.

Class-level валидаторы пишут через свою аннотацию и `ConstraintValidator<Annotation, Type>`. Внутри — чистая логика: если правило нарушено, возвращаем `false` и добавляем violation с понятным сообщением. При желании можно сформировать несколько ошибок с привязкой к полям.

Для коллекций пригодятся type-use аннотации (`List<@NotBlank String>`) — ими мы задаём ограничения для каждого элемента, а агрегатные проверки дополняем отдельным валидатором для всего списка (`@Distinct`). Такой дуэт покрывает и «микро», и «макро»-правила.

Проверки уникальности элементов нужно делать с учётом регистра и нормализации: «ABC» и «abc» — это одно и то же? Ответ зависит от домена. Фиксируйте политику и приводите значения к канону перед сравнением (см. раздел о нормализации).

Проверки больших коллекций должны быть эффективными: `HashSet` вместо вложенных циклов, лимиты на количество нарушений, чтобы не собирать тысячи сообщений в одном ответе. Это влияет на производительность и UX.

Класс-валидатор — хорошее место для «склейки» статуса: если агрегатная проверка провалилась, обычно это `422`. Но не кодируйте HTTP внутри валидатора — он должен оставаться транспорт-агностичным; трансляцию в HTTP выполняйте в обработчике ошибок.

Тестируйте такие валидаторы юнитами отдельно от веб-слоя: это чистые функции, и их удобно покрывать параметризованными кейсами. Так вы обеспечите стабильность контрактов независимо от транспортного стека.

*Gradle*
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-validation' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-validation") }
```

**Java — class-level @ValidInterval + @Distinct для списка**

```java
package com.example.taxonomy.cross;

import jakarta.validation.*;
import jakarta.validation.constraints.NotBlank;
import java.lang.annotation.*;
import java.time.LocalDate;
import java.util.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = IntervalValidator.class)
@interface ValidInterval {
    String message() default "from must be before to";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
class IntervalValidator implements ConstraintValidator<ValidInterval, Period> {
    @Override public boolean isValid(Period p, ConstraintValidatorContext c) {
        return p == null || (p.from() != null && p.to() != null && p.from().isBefore(p.to()));
    }
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DistinctValidator.class)
@interface Distinct {
    String message() default "elements must be unique";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
class DistinctValidator implements ConstraintValidator<Distinct, List<String>> {
    @Override public boolean isValid(List<String> value, ConstraintValidatorContext context) {
        if (value == null) return true;
        Set<String> s = new HashSet<>(value);
        return s.size() == value.size();
    }
}

@ValidInterval
record Period(LocalDate from, LocalDate to) {}

record Report(
        @Distinct List<@NotBlank String> tags,
        @Valid Period period
) {}
```

**Kotlin — аналог**

```kotlin
package com.example.taxonomy.cross

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import jakarta.validation.constraints.NotBlank
import java.lang.annotation.ElementType
import java.lang.annotation.Retention
import java.lang.annotation.RetentionPolicy
import java.time.LocalDate

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [IntervalValidator::class])
annotation class ValidInterval(
    val message: String = "from must be before to",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)
class IntervalValidator : ConstraintValidator<ValidInterval, Period> {
    override fun isValid(value: Period?, context: ConstraintValidatorContext): Boolean =
        value == null || (value.from != null && value.to != null && value.from.isBefore(value.to))
}

@Target(AnnotationTarget.FIELD)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [DistinctValidator::class])
annotation class Distinct(
    val message: String = "elements must be unique",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)
class DistinctValidator : ConstraintValidator<Distinct, List<String>?> {
    override fun isValid(value: List<String>?, context: ConstraintValidatorContext): Boolean =
        value?.toSet()?.size == value?.size
}

data class Period(val from: LocalDate?, val to: LocalDate?)
data class Report(
    @field:Distinct val tags: List<@NotBlank String>,
    @field:jakarta.validation.Valid val period: Period
)
```

---

## Онлайн/офлайн: мгновенные проверки и отложенные (например, уникальность по БД)

Не все проверки можно выполнить мгновенно на периметре. Уникальность email, существование внешней сущности, кредитный лимит — требуют обращения к БД или внешнему сервису. Мы называем их «онлайн»-проверками: они зависят от *состояния* системы и влекут сетевые задержки.

Онлайн-проверки переезжают в сервисный слой и должны иметь таймауты и деградацию. Если зависимость «лежит», нельзя вешать запросы навечно. Возвращайте чёткую ошибку (`503/504` для внешних зависимостей) или отложите операцию в очередь для повторной обработки.

Отложенные («офлайн») проверки — стратегия, когда не критично «упереться» в правило до завершения запроса. Например, импорт миллионов строк: часть с нарушением уникальности лучше отложить в DLQ и продолжить обработку. Пользователю позже вернётся отчёт. Такой подход держит SLA и уменьшает латентность.

Уникальность — особый кейс. Проверка `existsByEmail` *перед* сохранением подвержена гонкам; настоящая гарантия — уникальный индекс в БД. Приложение может предварительно подсказать пользователю, что email занят, но окончательное слово — за `UNIQUE`-constraintом и обработкой `DataIntegrityViolationException`.

Для онлайн-проверок держите кэш справочников, чтобы снизить латентность и нагрузку на БД. Но кэшировать нужно с TTL и инвалидацией: stale-данные — источник ложных отказов. В критичных сценариях — только БД.

Возвращайте корректные коды: конфликт уникальности — `409 Conflict`, «ресурс не найден» — `404`, «внешний сервис недоступен» — `503`. Это машинные сигналы, по которым фронтенд и оркестраторы строят поведение.

Не забывайте о трассировке: онлайн-проверки — точки риска. Пропустите через них `traceId`, метрики латентности и долю отказов. Это позволит быстро диагностировать деградации.

В тестах проверяйте и «счастливые» пути, и гонки: два параллельных запроса на одинаковый email. Убедитесь, что один падает `409`, а не оба «успешно» пишут разные записи.

*Gradle (добавим JPA + PostgreSQL для примера)*
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
}
```

**Java — уникальность email: пред-чек + защита constraint’ом**

```java
package com.example.taxonomy.online;

import jakarta.persistence.*;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.http.*;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.*;

import java.util.Optional;

@Entity @Table(name="users", uniqueConstraints=@UniqueConstraint(columnNames="email"))
class UserEntity {
    @Id @GeneratedValue(strategy=GenerationType.IDENTITY) Long id;
    @Column(nullable=false, unique=true) String email;
    @Column(nullable=false) String name;
    protected UserEntity() {}
    UserEntity(String email, String name){ this.email=email; this.name=name; }
}

interface UserRepo extends JpaRepository<UserEntity, Long> {
    boolean existsByEmail(String email);
    Optional<UserEntity> findByEmail(String email);
}

record CreateUser(@NotBlank @Email String email, @NotBlank String name) {}

@Service
class UserService {
    private final UserRepo repo;
    UserService(UserRepo repo){ this.repo = repo; }

    public UserEntity create(@Valid CreateUser dto) {
        if (repo.existsByEmail(dto.email())) {
            throw new IllegalStateException("email already used"); // будет мапиться в 409
        }
        try { return repo.save(new UserEntity(dto.email(), dto.name())); }
        catch (DataIntegrityViolationException ex) { // гонка: подстраховываемся
            throw new IllegalStateException("email already used");
        }
    }
}

@RestController
@RequestMapping("/api/users")
class UsersController {
    private final UserService service;
    UsersController(UserService s){ this.service = s; }

    @PostMapping
    public ResponseEntity<?> create(@RequestBody @Valid CreateUser dto) {
        try {
            var saved = service.create(dto);
            return ResponseEntity.status(HttpStatus.CREATED).body(saved.id);
        } catch (IllegalStateException ex) {
            var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
            pd.setTitle("Email not unique");
            pd.setDetail(ex.getMessage());
            return ResponseEntity.status(HttpStatus.CONFLICT).body(pd);
        }
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.taxonomy.online

import jakarta.persistence.*
import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import org.springframework.dao.DataIntegrityViolationException
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.*

@Entity
@Table(name = "users", uniqueConstraints = [UniqueConstraint(columnNames = ["email"])])
class UserEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,
    @Column(nullable = false, unique = true)
    var email: String = "",
    @Column(nullable = false)
    var name: String = ""
)

interface UserRepo : JpaRepository<UserEntity, Long> {
    fun existsByEmail(email: String): Boolean
    fun findByEmail(email: String): java.util.Optional<UserEntity>
}

data class CreateUser(@field:NotBlank @field:Email val email: String, @field:NotBlank val name: String)

@Service
class UserService(private val repo: UserRepo) {
    fun create(@Valid dto: CreateUser): UserEntity =
        if (repo.existsByEmail(dto.email)) throw IllegalStateException("email already used")
        else try { repo.save(UserEntity(email = dto.email, name = dto.name)) }
             catch (e: DataIntegrityViolationException) { throw IllegalStateException("email already used") }
}

@RestController
@RequestMapping("/api/users")
class UsersController(private val service: UserService) {

    @PostMapping
    fun create(@RequestBody @Valid dto: CreateUser): ResponseEntity<Any> =
        try {
            val id = service.create(dto).id
            ResponseEntity.status(HttpStatus.CREATED).body(id)
        } catch (e: IllegalStateException) {
            ResponseEntity.status(HttpStatus.CONFLICT).body(
                ProblemDetail.forStatus(HttpStatus.CONFLICT).apply {
                    title = "Email not unique"; detail = e.message
                }
            )
        }
}
```

---

## Нормализация vs валидация: trim/ASCII-folding/каноникализация до проверки правил

Нормализация — это подготовка данных к валидации и дальнейшей обработке. Самый простой пример — `trim`: пользователи часто копируют значения с пробелами. Другой пример — Unicode-каноникализация (NFC/NFKC), когда визуально одинаковые символы представлены разными кодпоинтами. «Сырые» строки без нормализации легко проваливают `@Pattern`/`@Email` по формальным причинам.

Нормализация — это не «исправление ошибок пользователя» втихую. Любая модификация входа должна быть прозрачной и безопасной: удалять невидимые символы окей, но менять «I» на «l» — нет. Если вы применяете агрессивные правила (ASCII-folding), делайте это только для полей, где потеря диакритики допустима (например, поиск), а не для юридических имён.

Где делать нормализацию? Либо до биндинга (фильтры/конвертеры), либо на уровне Jackson-десериализации, либо в `@InitBinder` через `PropertyEditor`. Важно: нормализацию запускают **до** синтаксической валидации, иначе вы получите лишние отказы. При этом в ответе клиенту можно показывать уже нормализованное значение, чтобы он понимал, что именно принято системой.

Глобальная нормализация строк через Jackson ускоряет разработку: все `String`-поля проходят один и тот же путь. Но это мощный инструмент: он может повлиять на пароли, подписи и бинарные данные. Поэтому обычно лучше нормализовать только входные DTO, а не *всё подряд*.

Отдельный кейс — каноникализация email/телефона. Для email безопасно триммить и приводить домен к нижнему регистру, но локальная часть до `@` теоретически регистрозависима. Для телефонов используйте библиотеку, понимающую страны (например, `libphonenumber`) и храните канон (E.164), а отображайте — форматированно.

Обязательно логируйте и метрику «сколько раз нормализация что-то изменила». Если цифра ненулевая, у продукта будет шанс поправить UX (подсказки, маски ввода).

С точки зрения безопасности нормализация помогает против одних атак (скрытые символы в путях/заголовках), но не заменяет валидации и экранирования. Не пытайтесь «очистить всё» — держите слои защиты.

Наконец, согласуйте нормализацию кросс-сервисно. Если один сервис триммит email, а другой — нет, уникальность и сопоставление пользователей развалятся. «Единый канон» — это тоже контракт.

*Gradle*
*Groovy DSL*

```groovy
dependencies { implementation 'com.fasterxml.jackson.core:jackson-databind' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("com.fasterxml.jackson.core:jackson-databind") }
```

**Java — глобальный трим/каноникализация строк через Jackson Module**

```java
package com.example.taxonomy.normalize;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.deser.std.StdScalarDeserializer;
import com.fasterxml.jackson.databind.module.SimpleModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.IOException;
import java.text.Normalizer;

@Configuration
public class JacksonNormalizationConfig {

    public static class TrimmingStringDeserializer extends StdScalarDeserializer<String> {
        public TrimmingStringDeserializer() { super(String.class); }
        @Override
        public String deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
            String s = p.getValueAsString();
            if (s == null) return null;
            String normalized = Normalizer.normalize(s, Normalizer.Form.NFKC).trim();
            return normalized;
        }
    }

    @Bean
    public com.fasterxml.jackson.databind.Module trimmingModule() {
        SimpleModule m = new SimpleModule();
        m.addDeserializer(String.class, new TrimmingStringDeserializer());
        return m;
    }
}
```

**Kotlin — аналог + выборочная нормализация полей через кастомный десериализатор**

```kotlin
package com.example.taxonomy.normalize

import com.fasterxml.jackson.core.JsonParser
import com.fasterxml.jackson.databind.DeserializationContext
import com.fasterxml.jackson.databind.Module
import com.fasterxml.jackson.databind.deser.std.StdScalarDeserializer
import com.fasterxml.jackson.databind.module.SimpleModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.text.Normalizer

@Configuration
class JacksonNormalizationConfig {

    class TrimmingStringDeserializer : StdScalarDeserializer<String>(String::class.java) {
        override fun deserialize(p: JsonParser, ctxt: DeserializationContext): String? {
            val s = p.valueAsString ?: return null
            return Normalizer.normalize(s, Normalizer.Form.NFKC).trim()
        }
    }

    @Bean
    fun trimmingModule(): Module = SimpleModule().apply {
        addDeserializer(String::class.java, TrimmingStringDeserializer())
    }
}
```

---

# 3. Bean Validation в Spring (MVC/Service)

## Jakarta Validation 3.x: `@Valid`/`@Validated` в контроллерах и на сервисных методах

`@Valid` включает проверку DTO при биндинге входного тела, параметров и вложенных объектов. В MVC при нарушениях Spring бросит `MethodArgumentNotValidException`, а в WebFlux — `WebExchangeBindException`. Это самый простой способ отсеять некорректные данные ещё до вызова бизнес-кода.

`@Validated` — аннотация Spring для метод-уровневой валидации. Её ставят на класс сервиса или контроллер, чтобы активировать проверки параметров и возвращаемых значений методов (например, `@Min` на аргументе метода сервиса). В Boot 3 метод-валидация включается авто-конфигурацией при наличии `spring-boot-starter-validation` — отдельный `MethodValidationPostProcessor` добавлять не нужно.

Хорошая практика — использовать `@Valid` на входных DTO в контроллерах, а `@Validated` — на сервисах/компонентах, где проходят доменно-ориентированные проверки сигнатур. Это разделяет транспорт и домен и делает код читаемым.

Метод-валидация полезна для публичных API ваших бинов, которые вызываются из разных мест. Например, сервис оплаты может требовать положительную сумму и не пустой идентификатор заказа независимо от того, пришёл ли вызов по REST или из консольного скрипта.

Возвращаемые значения тоже можно валидировать: `@NotNull` на результате метода. Если метод вернул `null`, валидация бросит исключение. Это помогает зафиксировать контракт на уровне кода и раньше ловить ошибки.

Не забывайте про группы (см. ниже) — они работают и с метод-валидацией. Это позволяет применять разные наборы правил для разных use-case (create/update) без дублирования DTO.

Важный нюанс — видимость прокси. Метод-валидация работает через AOP-прокси, поэтому самовызовы внутри бина не активируют её. Вынесите валидируемые методы в отдельный бин или вызывайте через интерфейс/прокси.

В тестах метод-валидации используйте полноценно поднятый контекст (`@SpringBootTest`) или явно создавайте валидатор через `LocalValidatorFactoryBean`. Для контроллеров — `MockMvc`/`WebTestClient`.

*Gradle*
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-validation' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-validation") }
```

**Java — `@Valid` в контроллере и `@Validated` в сервисе**

```java
package com.example.beanvalidation.basic;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/invoices")
class InvoiceController {
    private final InvoiceService service;
    InvoiceController(InvoiceService service) { this.service = service; }

    @PostMapping
    public Invoice create(@RequestBody @Valid Invoice dto) {
        return service.create(dto.customerId(), dto.amount());
    }
}

@Validated
@Service
class InvoiceService {
    public Invoice create(@NotBlank String customerId, @Positive long amount) {
        return new Invoice(customerId, amount);
    }
}

record Invoice(@NotBlank String customerId, @Positive long amount) {}
```

**Kotlin — аналог**

```kotlin
package com.example.beanvalidation.basic

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.stereotype.Service
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/invoices")
class InvoiceController(private val service: InvoiceService) {
    @PostMapping
    fun create(@RequestBody @Valid dto: Invoice): Invoice =
        service.create(dto.customerId, dto.amount)
}

@Validated
@Service
class InvoiceService {
    fun create(@NotBlank customerId: String, @Positive amount: Long): Invoice =
        Invoice(customerId, amount)
}

data class Invoice(
    @field:NotBlank val customerId: String,
    @field:Positive val amount: Long
)
```

---

## `BindingResult` vs исключения: когда возвращать 400/422, когда поднимать исключение

При `@Valid` по умолчанию Spring бросает исключение, которое вы перехватываете глобальным `@ControllerAdvice` и преобразуете в `ProblemDetail`. Это удобный централизованный путь, особенно когда у вас единый формат ошибок и общей логикой нужно обогащать ответы (traceId, коды, ссылки).

Иногда нужен *тонкий контроль* над телом ответа в конкретном методе: тогда используйте `BindingResult` сразу после валидируемого аргумента. Если есть ошибки, вы собираете их в структуру и возвращаете `422` без генерации исключения. Это немного снижает «шум» логов и даёт гибкость, но требует дисциплины, чтобы не расходиться с глобальной политикой.

Статусы: сломанный JSON/тип параметра — `400 Bad Request`; валидный JSON с нарушением правил — `422 Unprocessable Entity`. Если ошибка бизнес-конфликта (например, уникальность) — `409 Conflict`. Не путайте уровни и не возвращайте 500 за ожидаемые нарушения.

Опасность подхода с `BindingResult` — дублирование кода сборки ошибок. Лекарство — общий помощник/фабрика или всё-таки централизованный обработчик. Локальный `BindingResult` оставляйте для редких кейсов, где действительно нужна особая форма ответа.

Исключения удобны тем, что их легко классифицировать и мапить на статусы: `ValidationException` → 422, `ResourceNotFoundException` → 404. Имена классов читаемы, а хендлеры компактны.

В реактивных контроллерах `BindingResult` уже не работает в таком стиле; там удобнее оставить исключения и глобальные хендлеры (`@RestControllerAdvice`), которые перехватывают `WebExchangeBindException`.

В любом случае тестируйте оба пути: и локальный `BindingResult`, и обработку исключений в `@ControllerAdvice`, чтобы гарантировать единообразие.

*Gradle*
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

**Java — два варианта: BindingResult и исключения**

```java
package com.example.beanvalidation.binding;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/customers")
class CustomersController {

    @PostMapping("/local")
    public ResponseEntity<?> local(@RequestBody @Valid Customer c, BindingResult br) {
        if (br.hasErrors()) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setProperty("errors", br.getFieldErrors().stream()
                    .map(e -> Map.of("field", e.getField(), "message", e.getDefaultMessage()))
                    .toList());
            return ResponseEntity.unprocessableEntity().body(pd);
        }
        return ResponseEntity.ok(c);
    }

    @PostMapping("/global")
    public Customer global(@RequestBody @Valid Customer c) { return c; }
}

@RestControllerAdvice
class GlobalAdvice {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail invalid(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(e -> Map.of("field", e.getField(), "message", e.getDefaultMessage())).toList());
        return pd;
    }
}
record Customer(@NotBlank String name) {}
```

**Kotlin — аналог**

```kotlin
package com.example.beanvalidation.binding

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.validation.BindingResult
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/customers")
class CustomersController {

    @PostMapping("/local")
    fun local(@RequestBody @Valid c: Customer, br: BindingResult): ResponseEntity<Any> =
        if (br.hasErrors()) {
            val pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"
                setProperty("errors", br.fieldErrors.map { mapOf("field" to it.field, "message" to it.defaultMessage) })
            }
            ResponseEntity.unprocessableEntity().body(pd)
        } else ResponseEntity.ok(c)

    @PostMapping("/global")
    fun global(@RequestBody @Valid c: Customer): Customer = c
}

@RestControllerAdvice
class GlobalAdvice {
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun invalid(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("errors", ex.bindingResult.fieldErrors.map { mapOf("field" to it.field, "message" to it.defaultMessage) })
        }
}

data class Customer(@field:NotBlank val name: String)
```

---

## Вложенные DTO и коллекции: каскадная валидация, обязательные/опциональные поля

Каскадная валидация включает проверки вложенных объектов/элементов коллекций. Чтобы она сработала, пометьте поле `@Valid` (Java) или `@field:Valid` (Kotlin). Без этого вложенные DTO останутся «невидимыми» для валидатора, и ошибки в них пройдут внутрь.

Элементы коллекций валидируются двумя способами: аннотируйте саму коллекцию для агрегатных правил (например, `@Size(min=1)`), а тип элементов — для синтаксики (`List<@Valid Item>`). В Java поддерживаются type-use аннотации, в Kotlin — аналогично, но синтаксис отличается.

Опциональные поля описывайте явно через тип и аннотации. Если поле необязательно, делайте его nullable и не ставьте `@NotNull/@NotBlank`. Если оно обязательно, используйте не-nullable (Kotlin) + `@NotBlank` и дефолтные значения только там, где это оправдано. Смешение nullable-типов и строгих аннотаций часто приводит к «молчаливому» пропуску валидации.

Для вложенных списков хорошая практика — ограничивать их размер (`@Size(max=100)`), иначе можно получить «слонопотамский» JSON, который уронит сервис. Это относится к безопасности и производительности.

Валидация адресов/контактов (композитных объектов) должна быть самостоятельной: вынесите адрес в отдельный DTO с набором правил. Тогда вы сможете переиспользовать его в разных API, а изменения делаете в одном месте.

Старайтесь не использовать `Optional` в полях DTO (Java) — Jackson не любит его без доп-модулей, а семантика опциональности лучше выражается nullable-типами у Kotlin и «обычным» `null` у Java.

Для читаемости ошибок указывайте «путь» до поля (например, `items[2].sku`). Это бесплатно даёт BindingResult — достаточно правильно собрать массив ошибок в ответе.

*Gradle*
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-validation' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-validation") }
```

**Java — каскадная валидация вложенных DTO и коллекций**

```java
package com.example.beanvalidation.nested;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @PostMapping
    public Order create(@RequestBody @Valid Order order) { return order; }
}

record Order(
        @NotBlank String id,
        @Size(min = 1, max = 100) List<@Valid Item> items,
        @Valid Address shippingAddress
) {}

record Item(@NotBlank String sku, @Positive int qty) {}

record Address(@NotBlank String city, @NotBlank String street, @Size(min = 5, max = 10) String zip) {}
```

**Kotlin — аналог**

```kotlin
package com.example.beanvalidation.nested

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import jakarta.validation.constraints.Size
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @PostMapping
    fun create(@RequestBody @Valid order: Order): Order = order
}

data class Order(
    @field:NotBlank val id: String,
    @field:Size(min = 1, max = 100) val items: List<@Valid Item>,
    @field:Valid val shippingAddress: Address
)

data class Item(@field:NotBlank val sku: String, @field:Positive val qty: Int)
data class Address(
    @field:NotBlank val city: String,
    @field:NotBlank val street: String,
    @field:Size(min = 5, max = 10) val zip: String
)
```

---

## Kotlin-нюансы: nullability как контракт, конструкторная инициализация, default values

Главное отличие Kotlin — типовая система с null-безопасностью. Если поле объявлено как `String`, валидатору уже не нужно проверять «не null» — сериализация упадёт раньше. Но *пустая строка* (`""`) всё ещё возможна, поэтому `@NotBlank` сохраняет смысл. Если поле — `String?`, то `@NotBlank` **не сработает**, пока значение `null`. Это частая ловушка.

Чтобы аннотация применялась к полю, используйте таргет `@field:` (`@field:NotBlank`). Без него вы повесите constraint на аксессор/параметр конструктора, и валидатор может пропустить правило. Это одно из самых частых «почему не валидируется в Kotlin?».

Default values в data-классах улучшают DX: Jackson-модуль для Kotlin заполняет отсутствующие поля значениями по умолчанию. Но это может «маскировать» пропущенные обязательные поля. Для критичных полей не давайте дефолта и ставьте строгие аннотации.

`lateinit var` в DTO опасен: поле останется неинициализированным, но не-nullable, что приведёт к NPE позже. Предпочитайте `val` с явной обязательностью или nullable-типы, если поле действительно опционально.

Если в проекте включена строгая нотация имен (`SNAKE_CASE` у Jackson), убедитесь, что имена полей DTO совпадают с контрактом или настройте `@JsonProperty`. Иначе BindingResult вернёт «java-имена», а фронт будет ждать `snake_case`, — синхронизируйте формат.

Для method-валидации в Kotlin аннотации ставьте на параметры без `@field:` (там не нужно), но не забывайте `@Validated` на классе. Возвращаемые значения валидируйте через `@NotNull`/`@Positive` прямо на сигнатуре.

И наконец, всегда подключайте `jackson-module-kotlin`, иначе десериализация в Kotlin-data-классы будет странной (игнорируются дефолты/nullable) и валидация получит не то, что вы ожидаете.

*Gradle (Kotlin-модуль Jackson)*
*Groovy DSL*

```groovy
dependencies { implementation 'com.fasterxml.jackson.module:jackson-module-kotlin' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("com.fasterxml.jackson.module:jackson-module-kotlin") }
```

**Java — эквивалент (для сравнения поведения)**

```java
package com.example.beanvalidation.kotlinpitfalls;

import jakarta.validation.constraints.NotBlank;

public record Profile(@NotBlank String email) {}
```

**Kotlin — правильные таргеты аннотаций, nullable и дефолты**

```kotlin
package com.example.beanvalidation.kotlinpitfalls

import com.fasterxml.jackson.annotation.JsonProperty
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/kprofiles")
class ProfilesController {
    @PostMapping
    fun create(@RequestBody profile: Profile): Profile = profile
}

@Validated
data class Profile(
    @field:NotBlank @field:Email
    @JsonProperty("email") val email: String,            // обязателен, без дефолта
    @field:Positive
    val age: Int = 18,                                   // дефолт допустим, если бизнес-правило позволяет
    val nickname: String? = null                         // опциональное поле
)
```
# 4. Аннотации и свои констрейнты

## Базовые: `@NotNull`/`@NotBlank`/`@Size`/`@Pattern`/`@Email`/`@Positive`/`@PastOrPresent` и т.д.

Валидационные аннотации — первый эшелон защиты от «грязного» ввода. Они дешёвые по вычислению и не требуют внешних зависимостей, поэтому идеально подходят для применения на транспортных DTO (параметры запроса, тело JSON). Важно понимать различия схожих аннотаций: `@NotNull` запрещает `null`, но пустую строку пропустит; `@NotEmpty` запрещает `null` и пустую коллекцию/строку; `@NotBlank` дополнительно «подрежет» пробелы и запретит строку из одних пробелов. Для строк, вводимых человеком, надёжнее `@NotBlank`.

`@Size` контролирует длину строк и размер коллекций. Частая ошибка — полагаться на `@Size(max=…)` только в приложении: если поле хранится в БД с меньшей длиной столбца, при расхождении ограничений вы поймаете исключение на уровне JDBC. Держите договорённости в одном месте: либо подгоняйте `@Column(length=…)` к `@Size`, либо генерируйте схему из модели.

`@Pattern` полезен для строго формализованных форматов (например, артикулы, «чистый» номер телефона, код партнёра). Регулярки должны быть простыми и ограничены по времени; излишне сложные паттерны открывают дорогу regex-DoS. Для email лучше использовать `@Email` из валидатора, а не придумывать регэкспы — стандартная проверка сбалансирована по строгости и перформансу.

Числовые ограничения (`@Positive`, `@PositiveOrZero`, `@Negative`, `@Min`, `@Max`) привязывайте к типам, которые реально отражают вашу модель. Так, для денег лучше использовать `BigDecimal` и дополнить `@Digits(integer=…, fraction=…)`, чем целые и «копейки». И не забывайте, что `@Positive` на wrapper-типе (`Long`) не эквивалентен «обязательности»: для «null запрещён» нужна ещё и `@NotNull`.

Временные аннотации (`@Past`, `@PastOrPresent`, `@Future`, `@FutureOrPresent`) проверяют отношение к «сейчас» на стороне сервера. Согласуйте таймзону и формат даты: если вы принимаете OffsetDateTime/ZonedDateTime, сервер без проблем проверит относительность; если LocalDateTime — валидация идёт в системной зоне. Для бизнес-правил типа «дата окончания после начала» базовые аннотации недостаточны — понадобится class-level валидатор.

Отдельный кейс — UUID. В стандарте Bean Validation нет универсальной `@UUID`, поэтому часто применяют `@Pattern` с простым шаблоном или пишут небольшой кастомный валидатор, который пытается распарсить `java.util.UUID`. У паттерна плюс в скорости; у кастомного — в корректности (проверяет и версию/вариант).

На уровне контроллеров базовые аннотации дают «мгновенный» отказ с 400/422 и понятной ошибкой. Не дублируйте те же правила в доменном слое, если там уже есть сильные инварианты — разделяйте ответственностии: синтаксис/формат на DTO, смысл — в домене.

И, наконец, не забывайте про тесты на типичные пограничные случаи: «пустая строка», «строка из пробелов», «макс. длина + 1», «некорректный символ». Такие тесты дешевы, но на проде экономят часы.

*Gradle (для примеров ниже)*
*Groovy DSL*:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

*Kotlin DSL*:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

*Java — базовые аннотации в DTO и контроллере:*

```java
package com.example.valid.basic;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;

@RestController
@RequestMapping("/api/customers")
public class CustomerController {

    @PostMapping
    public Customer create(@RequestBody @Valid Customer dto) { return dto; }

    public record Customer(
            @NotBlank String name,
            @Email String email,
            @Pattern(regexp = "^[0-9]{10,15}$") String phone,
            @Positive long balance,
            @PastOrPresent LocalDate registeredAt
    ) {}
}
```

*Kotlin — аналогичные ограничения:*

```kotlin
package com.example.valid.basic

import jakarta.validation.Valid
import jakarta.validation.constraints.*
import org.springframework.web.bind.annotation.*
import java.time.LocalDate

@RestController
@RequestMapping("/api/customers")
class CustomerController {
    @PostMapping
    fun create(@RequestBody @Valid dto: Customer): Customer = dto
}

data class Customer(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Pattern(regexp = "^[0-9]{10,15}$") val phone: String,
    @field:Positive val balance: Long,
    @field:PastOrPresent val registeredAt: LocalDate
)
```

---

## Кастомный constraint: `@interface` + `ConstraintValidator`; повторное использование и тестируемость

Даже богатого набора базовых аннотаций может не хватать. Правило «сильный пароль», проверка бизнес-идентификатора или «нулевой ЛПУ недопустим» — уникальны для вашего домена. Для таких случаев мы создаём **кастомный constraint**: собственную аннотацию с мета-аннотацией `@Constraint` и валидатор, реализующий `ConstraintValidator<A, T>`.

Сильная сторона кастомных ограничений — **повторное использование**. Написав `@StrongPassword`, вы используете его на любом DTO/команде, а правило не размазывается по сервисам. Это повышает читаемость: по коду понятно, «почему не проходит пароль», без нужды читать регулярки.

Кастомный валидатор легко тестируется в юнитах, потому что это обычный класс с детерминированной логикой. Вы можете прогнать десятки паролей и зафиксировать поведение на годы вперёд. А если правило меняется, меняется ровно одно место.

Интересная деталь Spring Boot: интеграция с Hibernate Validator позволяет **инжектить бины** в валидатор. Это значит, что вы можете проверять сложные правила (например, «не входит в список утечек»), обращаясь к сервисам/кэшу. Главное — помнить о перформансе и выставлять таймауты.

Сообщения в кастомных ограничениях поддерживают placeholders (`{min}`, `{max}`, `{validatedValue}`), а также коды сообщений из bundles. Это позволяет локализовать и сделать тексты дружелюбными.

Кастомное ограничение может быть **составным**: вы можете разместить на поле и `@StrongPassword`, и `@Size`, и `@Pattern`, но нередко лучше собрать всё в одном валидаторе и вернуть одно человеческое сообщение вместо «стены» нарушений.

Ещё одна практика — **метод-уровневые** валидаторы (class-level): если правило касается нескольких полей, пишите аннотацию для класса, а не поля (см. следующую подтему). Полезно добавить параметр `target()` в аннотацию для гибкой настройки.

Важно не перегружать правила «интеллектом»: валидация — про вход, а не про сложные вычисления. Держите границы ответственности, особенно если валидатор делает сетевые вызовы.

*Java — `@StrongPassword`:*

```java
package com.example.valid.custom;

import jakarta.validation.*;
import java.lang.annotation.*;
import java.util.regex.Pattern;

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {
    String message() default "Password must be 8+ chars, with upper, lower, digit and special";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    int min() default 8;
}

class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    private int min;
    private static final Pattern UPPER = Pattern.compile("[A-Z]");
    private static final Pattern LOWER = Pattern.compile("[a-z]");
    private static final Pattern DIGIT = Pattern.compile("[0-9]");
    private static final Pattern SPEC  = Pattern.compile("[^A-Za-z0-9]");

    @Override public void initialize(StrongPassword ann) { this.min = ann.min(); }

    @Override public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true; // @NotNull решает "обязательность"
        return value.length() >= min &&
               UPPER.matcher(value).find() &&
               LOWER.matcher(value).find() &&
               DIGIT.matcher(value).find() &&
               SPEC.matcher(value).find();
    }
}
```

*Kotlin — `@StrongPassword` (с параметром `min`):*

```kotlin
package com.example.valid.custom

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import java.util.regex.Pattern
import kotlin.reflect.KClass

@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [StrongPasswordValidator::class])
annotation class StrongPassword(
    val message: String = "Password must be 8+ chars, with upper, lower, digit and special",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val min: Int = 8
)

class StrongPasswordValidator : ConstraintValidator<StrongPassword, String> {
    private var min: Int = 8
    private val upper = Pattern.compile("[A-Z]")
    private val lower = Pattern.compile("[a-z]")
    private val digit = Pattern.compile("[0-9]")
    private val spec  = Pattern.compile("[^A-Za-z0-9]")

    override fun initialize(annotation: StrongPassword) { min = annotation.min }
    override fun isValid(value: String?, context: ConstraintValidatorContext): Boolean =
        value == null || (value.length >= min &&
                upper.matcher(value).find() &&
                lower.matcher(value).find() &&
                digit.matcher(value).find() &&
                spec.matcher(value).find())
}
```

*Контроллер-демо (Java/Kotlin) — применение:*

```java
@RestController
@RequestMapping("/api/auth")
class AuthController {
    @PostMapping("/signup")
    public Map<String,String> signup(@RequestParam @StrongPassword String password) {
        return Map.of("ok","true");
    }
}
```

```kotlin
@RestController
@RequestMapping("/api/auth")
class AuthController {
    @PostMapping("/signup")
    fun signup(@RequestParam @StrongPassword password: String) = mapOf("ok" to "true")
}
```

---

## Сообщения и локализация: codes, message bundles, placeholders; понятные тексты для клиента

Понятные тексты ошибок — часть DX и UX. Jakarta Validation позволяет вынести сообщения в bundles (`messages.properties`), что даёт локализацию и консистентность. В аннотации вместо текста указывайте ключ `{user.email.invalid}`, а в ресурсах храните перевод для каждой локали.

Placeholders упрощают создание информативных сообщений без «ручной склейки»: в стандартных аннотациях доступны `{min}`, `{max}`, `{regexp}`, `{validatedValue}`. Например, `@Size(min=2, max=10, message="{name.size}")` при подстановке превратится в «Имя должно быть от 2 до 10 символов», если так записано в `messages.properties`.

Spring Boot использует `MessageSource` для резолва. В большинстве случаев достаточно просто добавить `messages.properties` на classpath; но если вы хотите явной конфигурации и кодировок, определите `ResourceBundleMessageSource` и включите `Accept-Language`-резолвер, чтобы клиент мог управлять языком через заголовки.

Следите за тональностью: сообщения в DTO — технические (для разработчиков клиента), а пользовательские тексты лучше строить на фронте из машиночитаемых кодов. Это позволит менять wording без релиза бэкенда и держать единый стиль.

Прозрачность: в ответе `ProblemDetail` не обязательно отдавать весь текст валидации; достаточно кода (`Email`) и понятного краткого `detail`. Но при отладке (стейджинг/приватные API) полнотекстовые сообщения ускоряют внедрение — можно включать флагом.

Не храните в сообщениях PII (например, «email [user@example.com](mailto:user@example.com) недопустим») — это может попасть в логи и аналитики. Лучше «Поле email недопустимо» и возвращать проблемное значение только в безопасных средах.

Держите единый словарь ключей сообщений. Это дисциплинирует команды и упрощает поиск/замены. Полезно ввести префиксы: `user.*`, `order.*`, `common.*`.

Покрывайте i18n-тестами хотя бы ключевые правила: при смене локали тексты должны меняться, а ключи — резолвиться. Ложно-отсутствующий ключ в рантайме превращается в «{…}» в ответе — неприятно для клиентов.

*Java — конфигурация `MessageSource` и пример DTO с локализацией:*

```java
package com.example.valid.i18n;

import jakarta.validation.constraints.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/i18n")
class I18nController {
    @PostMapping
    public Profile create(@RequestBody Profile p) { return p; }
}

record Profile(
        @NotBlank(message = "{profile.name.notBlank}") String name,
        @Email(message = "{profile.email.invalid}") String email
) {}

@org.springframework.context.annotation.Configuration
class I18nConfig {
    @Bean
    ResourceBundleMessageSource messageSource() {
        var ms = new ResourceBundleMessageSource();
        ms.setBasenames("messages");
        ms.setDefaultEncoding("UTF-8");
        return ms;
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.valid.i18n

import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import org.springframework.context.annotation.Bean
import org.springframework.context.support.ResourceBundleMessageSource
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/i18n")
class I18nController {
    @PostMapping
    fun create(@RequestBody p: Profile): Profile = p
}

data class Profile(
    @field:NotBlank(message = "{profile.name.notBlank}") val name: String,
    @field:Email(message = "{profile.email.invalid}") val email: String
)

@org.springframework.context.annotation.Configuration
class I18nConfig {
    @Bean
    fun messageSource() = ResourceBundleMessageSource().apply {
        setBasenames("messages")
        setDefaultEncoding("UTF-8")
    }
}
```

*`src/main/resources/messages.properties`:*

```
profile.name.notBlank=Имя обязательно для заполнения
profile.email.invalid=Некорректный email
```

---

## Группы и сценарии: группы по use-case (create/update), fail-fast режим и его влияние

Группы валидации позволяют применять **разные наборы правил** для разных сценариев. Классический пример: при создании сущности поле обязательно (`@NotBlank(groups=Create.class)`), а при обновлении — опционально, но если пришло, должно соответствовать формату (`@Email(groups=Update.class)`). Вы объявляете маркерные интерфейсы `Create`/`Update` и активируете их через `@Validated(Create.class)` на методе/классе.

Это избавляет от дублирования DTO «под каждый случай» и чётко фиксирует контракты. При этом следите за читаемостью: если групп слишком много, проще разнести на разные DTO, особенно если payloadы реально различаются.

Fail-fast — режим, при котором валидатор останавливается на **первом нарушении**. Он экономит CPU и сокращает латентность на «шумных» запросах, но ухудшает UX, потому что клиент получает ошибки по одной. Для публичных API удобнее **собирать все нарушения**; для внутренних высоконагруженных сервисов fail-fast уместен.

В Boot можно включить fail-fast программно, настроив `LocalValidatorFactoryBean` и передав свойство `hibernate.validator.fail_fast=true`. Делать это стоит осознанно и точечно: например, для batch-валидаторов, а не для периметра HTTP.

Группы работают и в method-валидации: можно требовать разные параметры для разных методов сервиса. Это полезно для команд с различными правами/контекстами.

Учитывайте взаимодействие групп с вложенными объектами: каскадная валидация также учитывает группы. Если в дочернем DTO стоят группы `Create`, то при `@Validated(Create.class)` они активируются, а при `Update` — нет.

Тестируйте сценарии отдельно: «create» с отсутствующим обязательным полем должен падать, «update» с тем же отсутствием — проходить, но падать при неправильном формате, если поле присутствует. Это фиксирует границы контракта.

Наконец, группы — не панацея. Если payloadы действительно разные, короткие DTO «CreateX/UpdateX» зачастую проще и надёжнее, чем усложнение одного класса десятком групп.

*Java — группы и fail-fast конфигурация:*

```java
package com.example.valid.groups;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.context.annotation.Bean;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

interface Create {}
interface Update {}

@RestController
@RequestMapping("/api/profiles")
class ProfilesController {

    @PostMapping
    public Profile create(@RequestBody @Validated(Create.class) Profile p) { return p; }

    @PatchMapping("/{id}")
    public Profile update(@RequestBody @Validated(Update.class) Profile p) { return p; }
}

record Profile(
        @NotBlank(groups = Create.class, message="{profile.name.required}") String name,
        @Email(groups = {Create.class, Update.class}) String email,
        @Positive(groups = Create.class) Integer age // при update не обязателен
) {}

@org.springframework.context.annotation.Configuration
class ValidationCfg {
    @Bean
    LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean v = new LocalValidatorFactoryBean();
        v.getValidationPropertyMap().put("hibernate.validator.fail_fast", "false"); // или true — по задаче
        return v;
    }
}
```

*Kotlin — группы и конфигурация валидатора:*

```kotlin
package com.example.valid.groups

import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.context.annotation.Bean
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*

interface Create
interface Update

@RestController
@RequestMapping("/api/profiles")
class ProfilesController {
    @PostMapping
    fun create(@RequestBody @Validated(Create::class) p: Profile): Profile = p

    @PatchMapping("/{id}")
    fun update(@RequestBody @Validated(Update::class) p: Profile): Profile = p
}

data class Profile(
    @field:NotBlank(groups = [Create::class], message = "{profile.name.required}") val name: String?,
    @field:Email(groups = [Create::class, Update::class]) val email: String?,
    @field:Positive(groups = [Create::class]) val age: Int?
)

@org.springframework.context.annotation.Configuration
class ValidationCfg {
    @Bean
    fun validator(): LocalValidatorFactoryBean =
        LocalValidatorFactoryBean().apply {
            validationPropertyMap["hibernate.validator.fail_fast"] = "false"
        }
}
```

---

# 5. Кросс-полевые/агрегатные правила

## Class-level валидаторы: согласованность пар («from < to», «type ↔ поля»)

Есть целый класс правил, которые невозможно выразить на уровне отдельного поля. Простейший пример — интервал дат: `from` должен быть строго раньше `to`. Другие примеры — «если `type=CARD`, поле `pan` обязательно», «для `deliveryType=PICKUP` поле `address` должно быть пустым». Для таких случаев создают **class-level** аннотации, валидатор которых видит весь объект.

Class-level валидатор снижает дублирование: вместо копирования проверок по контроллерам и сервисам — один проверяющий компонент. Такой подход повышает читаемость DTO: по аннотации на классе видно, что у него существуют межполевые инварианты.

Ещё плюс — правильное сопоставление ошибки с полем. Внутри валидатора вы можете вызвать `context.buildConstraintViolationWithTemplate(...).addPropertyNode("to").addConstraintViolation()` и тем самым прикрепить нарушение к конкретному полю, пусть проверка и class-level. Это существенно улучшает UX на фронте (подсветка конкретного поля).

Class-level валидаторы не должны тянуть внешние зависимости (БД/сети). Их ответственность — логические связи **внутри** объекта. Всё, что требует обращений к репозиториям/кэшам, — в сервисные проверки (см. ниже).

Правильно документируйте поведение: если интервал допускает равенство границ, об этом нужно явно сказать в названии аннотации (`@ValidInterval(strict = false)`). Иначе клиенты будут гадать, почему «from == to» валидно/не валидно.

Покрывайте юнит-тестами: создать объект и прогнать валидатор просто, а ошибки выявляются рано. Полезно составлять таблицы граничных значений (null/null, null/val, val/null, val<val, val=val, val>val).

Не забывайте про nullable-поля: решите, что считать валидным при `null`. Частая стратегия — «если поле отсутствует, проверка интервала не срабатывает» — это даёт возможность частичных обновлений.

И наконец, class-level-ограничения должны быть атомарны и узконаправленны. Не сваливайте всю «бизнес-валидацию» в один огромный валидатор — его потом невозможно поддерживать.

*Java — `@ValidInterval` и «тип ↔ поля»:*

```java
package com.example.cross.classlevel;

import jakarta.validation.*;
import java.lang.annotation.*;
import java.time.LocalDate;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = IntervalValidator.class)
public @interface ValidInterval {
    String message() default "from must be before to";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    boolean strict() default true;
}

class IntervalValidator implements ConstraintValidator<ValidInterval, PeriodDto> {
    private boolean strict;
    @Override public void initialize(ValidInterval ann){ this.strict = ann.strict(); }
    @Override public boolean isValid(PeriodDto p, ConstraintValidatorContext ctx) {
        if (p == null || p.from == null || p.to == null) return true;
        boolean ok = strict ? p.from.isBefore(p.to) : !p.from.isAfter(p.to);
        if (!ok) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("to must be after from")
               .addPropertyNode("to").addConstraintViolation();
        }
        return ok;
    }
}

enum PaymentType { CARD, CASH }

@ValidInterval
@TypeFieldsCoherent
class PeriodDto {
    public LocalDate from;
    public LocalDate to;
    public PaymentType type;
    public String pan;      // номер карты для CARD
    public String comment;  // произвольное поле
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = TypeFieldsValidator.class)
@interface TypeFieldsCoherent {
    String message() default "type-field mismatch";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
class TypeFieldsValidator implements ConstraintValidator<TypeFieldsCoherent, PeriodDto> {
    @Override public boolean isValid(PeriodDto v, ConstraintValidatorContext ctx) {
        if (v == null) return true;
        if (v.type == PaymentType.CARD && (v.pan == null || v.pan.isBlank())) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("pan required for CARD")
               .addPropertyNode("pan").addConstraintViolation();
            return false;
        }
        if (v.type == PaymentType.CASH && v.pan != null) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("pan must be null for CASH")
               .addPropertyNode("pan").addConstraintViolation();
            return false;
        }
        return true;
    }
}
```

*Kotlin — аналоги:*

```kotlin
package com.example.cross.classlevel

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import java.time.LocalDate
import kotlin.reflect.KClass

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [IntervalValidator::class])
annotation class ValidInterval(
    val message: String = "from must be before to",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = [],
    val strict: Boolean = true
)

class IntervalValidator : ConstraintValidator<ValidInterval, PeriodDto> {
    private var strict: Boolean = true
    override fun initialize(annotation: ValidInterval) { strict = annotation.strict }
    override fun isValid(p: PeriodDto?, ctx: ConstraintValidatorContext): Boolean {
        if (p?.from == null || p.to == null) return true
        val ok = if (strict) p.from.isBefore(p.to) else !p.from.isAfter(p.to)
        if (!ok) {
            ctx.disableDefaultConstraintViolation()
            ctx.buildConstraintViolationWithTemplate("to must be after from")
                .addPropertyNode("to").addConstraintViolation()
        }
        return ok
    }
}

enum class PaymentType { CARD, CASH }

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [TypeFieldsValidator::class])
annotation class TypeFieldsCoherent(
    val message: String = "type-field mismatch",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class TypeFieldsValidator : ConstraintValidator<TypeFieldsCoherent, PeriodDto> {
    override fun isValid(v: PeriodDto?, ctx: ConstraintValidatorContext): Boolean {
        v ?: return true
        if (v.type == PaymentType.CARD && v.pan.isNullOrBlank()) {
            ctx.disableDefaultConstraintViolation()
            ctx.buildConstraintViolationWithTemplate("pan required for CARD")
                .addPropertyNode("pan").addConstraintViolation()
            return false
        }
        if (v.type == PaymentType.CASH && v.pan != null) {
            ctx.disableDefaultConstraintViolation()
            ctx.buildConstraintViolationWithTemplate("pan must be null for CASH")
                .addPropertyNode("pan").addConstraintViolation()
            return false
        }
        return true
    }
}

data class PeriodDto(
    var from: LocalDate? = null,
    var to: LocalDate? = null,
    var type: PaymentType? = null,
    var pan: String? = null,
    var comment: String? = null
)
```

---

## Вычисляемые ограничения: сумма элементов, уникальность в коллекции

Иногда ограничение касается **агрегата**: «сумма строк не должна превышать кредитный лимит», «в корзине товары уникальны», «все суммы — кратны шагу». Это не про синтаксис отдельных полей, а про вычисление по нескольким элементам. Удобно оформить такие правила как class-level constraints с параметрами.

Суммы нужно считать аккуратно: для денег используйте `BigDecimal`, избегая ошибок округления и переполнений. Проверяйте валидность каждого элемента (`qty > 0`, `price >= 0`) прежде чем суммировать — иначе логика может «съесть» баги из элементов и ответ получится нечестным.

Уникальность в коллекции — частая задача. Для строк и идентификаторов хватит `Set` и сравнения по канонической форме (например, в нижнем регистре, без пробелов, если таков ваш доменный канон). Но обязательно зафиксируйте канон, чтобы два сервиса не трактовали «равенство» по-разному.

При больших коллекциях ограничивайте верхний предел (`@Size(max = …)`) и лимит количества сообщений об ошибках (например, не больше 20) — иначе ответ получится слишком громоздким, а сервер потратит много CPU на формирование детальной диагностики.

Возвращайте понятные нарушения: «повторы на индексах 2, 5», «сумма X больше лимита Y». Это ускоряет исправление на клиенте, особенно для фронтовых форм со списками.

Если правило связано с конфигурацией (кредитный лимит из профиля клиента), оно уже не «чистое» агрегатное — переносите в сервис и не делайте сетевых запросов из валидатора. Валидатор должен быть быстрым и детерминированным.

Проверяйте агрегатные ограничения в юнит-тестах на таблицах случаев: пустой список, одна позиция, несколько, превышение на копейку, повтор элементов. Это быстро и надёжно.

И наконец, отделяйте «жёсткие» ограничения от «подсказок». Например, допустимо возвращать `422`, если превышен лимит, и `200` с рекомендацией при «нестандартных, но допустимых» комбинациях — но это уже про бизнес-правила.

*Java — `@TotalAmountMax` и `@UniqueSkus`:*

```java
package com.example.cross.aggregate;

import jakarta.validation.*;
import jakarta.validation.constraints.*;
import java.lang.annotation.*;
import java.math.BigDecimal;
import java.util.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = TotalAmountMaxValidator.class)
public @interface TotalAmountMax {
    String message() default "Total exceeds limit";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String limitProperty() default "limit";
}
class TotalAmountMaxValidator implements ConstraintValidator<TotalAmountMax, Order> {
    private String limitProp;
    @Override public void initialize(TotalAmountMax ann){ this.limitProp = ann.limitProperty(); }
    @Override public boolean isValid(Order o, ConstraintValidatorContext ctx) {
        if (o == null || o.items == null) return true;
        BigDecimal total = o.items.stream()
                .map(i -> i.price.multiply(BigDecimal.valueOf(i.qty)))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        if (total.compareTo(o.limit) > 0) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("total="+total+" > limit="+o.limit)
               .addPropertyNode("items").addConstraintViolation();
            return false;
        }
        return true;
    }
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueSkusValidator.class)
@interface UniqueSkus { String message() default "Duplicate SKUs"; Class<?>[] groups() default {}; Class<? extends Payload>[] payload() default {}; }
class UniqueSkusValidator implements ConstraintValidator<UniqueSkus, List<Order.Item>> {
    @Override public boolean isValid(List<Order.Item> items, ConstraintValidatorContext ctx) {
        if (items == null) return true;
        Set<String> seen = new HashSet<>();
        for (Order.Item i : items) if (!seen.add(i.sku.toLowerCase())) return false;
        return true;
    }
}

@TotalAmountMax
class Order {
    @UniqueSkus
    public List<Item> items = new ArrayList<>();
    @NotNull public BigDecimal limit = BigDecimal.valueOf(100);
    public static class Item {
        @NotBlank public String sku; @Positive public int qty; @PositiveOrZero public BigDecimal price;
        public Item(String sku, int qty, BigDecimal price){ this.sku=sku; this.qty=qty; this.price=price; }
    }
}
```

*Kotlin — аналоги:*

```kotlin
package com.example.cross.aggregate

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import jakarta.validation.constraints.PositiveOrZero
import java.math.BigDecimal
import kotlin.reflect.KClass

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [TotalAmountMaxValidator::class])
annotation class TotalAmountMax(
    val message: String = "Total exceeds limit",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class TotalAmountMaxValidator : ConstraintValidator<TotalAmountMax, Order> {
    override fun isValid(o: Order?, ctx: ConstraintValidatorContext): Boolean {
        o ?: return true
        val total = o.items.sumOf { it.price.multiply(BigDecimal(it.qty)) }
        return if (total > o.limit) {
            ctx.disableDefaultConstraintViolation()
            ctx.buildConstraintViolationWithTemplate("total=$total > limit=${o.limit}")
                .addPropertyNode("items").addConstraintViolation()
            false
        } else true
    }
}

@Target(AnnotationTarget.FIELD)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [UniqueSkusValidator::class])
annotation class UniqueSkus(
    val message: String = "Duplicate SKUs",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class UniqueSkusValidator : ConstraintValidator<UniqueSkus, List<Order.Item>?> {
    override fun isValid(items: List<Order.Item>?, context: ConstraintValidatorContext): Boolean {
        items ?: return true
        val seen = HashSet<String>()
        for (i in items) if (!seen.add(i.sku.lowercase())) return false
        return true
    }
}

data class Order(
    @field:UniqueSkus val items: List<Item> = emptyList(),
    val limit: BigDecimal = BigDecimal(100)
) {
    data class Item(
        @field:NotBlank val sku: String,
        @field:Positive val qty: Int,
        @field:PositiveOrZero val price: BigDecimal
    )
}
```

---

## Ограничение состояний: конечные автоматы домена и допустимые переходы

Многие доменные объекты живут по правилам конечного автомата: заказ «CREATED → PAID → SHIPPED → COMPLETED/RETURNED», тикет «OPEN → IN_PROGRESS → RESOLVED → CLOSED». Запрет недопустимых переходов — **валидация состояния**, которая должна сработать раньше, чем вы измените агрегат и зафиксируете событие.

Такое правило удобно описывать кастомным ограничением `@ValidTransition`, валидатор которого знает допустимые пары (`from`, `to`). Источником может быть перечисление или даже внешний конфиг, если процессы настраиваемые — но для валидации лучше держать **локальную копию** графа, чтобы не зависеть от сети.

Важно различать «транспортный» переход (то, что прислал клиент) и «фактическое состояние» в БД. Для команд идёт проверка пары из DTO; а в сервисе вы дополнительно проверяете актуальность: «а не ушёл ли объект в другое состояние с тех пор?» — иначе поймаете гонку и «прыжок» через состояния.

Сами правила автомата — **семантика**, не тяните их в аннотации на транспортных DTO, если они нестабильны. Но если «скелет» стабильный, class-level валидатор удобно документирует инвариант и предотвращает очевидные ошибки уже на периметре.

Возвраты в прошлые состояния, «промотка» через несколько состояний, переходы в терминальные стейты — всё это должно быть явно описано. Хорошая практика — выделить терминальные состояния и запретить любые переходы из них.

Не забывайте про сообщения: «Transition from SHIPPED to CREATED is not allowed» — гораздо полезнее, чем «Invalid state». Клиент сможет показать пользователю контекст и не плодить тикеты.

Тесты строятся как таблица переходов (matrix): все пары `from → to` и ожидание «ok/запрет». Это позволяет не забыть редкие ветки и наглядно видеть изменения при эволюции автомата.

И, конечно, на уровне БД поддерживайте инварианты (например, через CHECK-ограничения или триггеры), если это критично. Но лучше, когда приложение не пытается записать дурной переход вообще.

*Java — `@ValidTransition` для пары статусов:*

```java
package com.example.cross.fsm;

import jakarta.validation.*;
import java.lang.annotation.*;
import java.util.EnumSet;
import java.util.Set;

enum Status { CREATED, PAID, SHIPPED, COMPLETED, RETURNED }

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = TransitionValidator.class)
@interface ValidTransition {
    String message() default "Invalid status transition";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

class TransitionValidator implements ConstraintValidator<ValidTransition, TransitionDto> {
    private static final Set<Pair> ALLOWED = Set.of(
            new Pair(Status.CREATED, Status.PAID),
            new Pair(Status.PAID, Status.SHIPPED),
            new Pair(Status.SHIPPED, Status.COMPLETED),
            new Pair(Status.SHIPPED, Status.RETURNED)
    );
    @Override public boolean isValid(TransitionDto t, ConstraintValidatorContext ctx) {
        if (t == null || t.from == null || t.to == null) return true;
        boolean ok = ALLOWED.contains(new Pair(t.from, t.to));
        if (!ok) {
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("Transition from "+t.from+" to "+t.to+" is not allowed")
               .addPropertyNode("to").addConstraintViolation();
        }
        return ok;
    }
    private record Pair(Status from, Status to) {}
}

@ValidTransition
class TransitionDto {
    public Status from;
    public Status to;
}
```

*Kotlin — аналог:*

```kotlin
package com.example.cross.fsm

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import kotlin.reflect.KClass

enum class Status { CREATED, PAID, SHIPPED, COMPLETED, RETURNED }

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [TransitionValidator::class])
annotation class ValidTransition(
    val message: String = "Invalid status transition",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class TransitionValidator : ConstraintValidator<ValidTransition, TransitionDto> {
    private val allowed = setOf(
        Pair(Status.CREATED, Status.PAID),
        Pair(Status.PAID, Status.SHIPPED),
        Pair(Status.SHIPPED, Status.COMPLETED),
        Pair(Status.SHIPPED, Status.RETURNED)
    )
    override fun isValid(t: TransitionDto?, ctx: ConstraintValidatorContext): Boolean {
        t ?: return true
        val ok = allowed.contains(Pair(t.from, t.to))
        if (!ok) {
            ctx.disableDefaultConstraintViolation()
            ctx.buildConstraintViolationWithTemplate("Transition from ${t.from} to ${t.to} is not allowed")
                .addPropertyNode("to").addConstraintViolation()
        }
        return ok
    }
}

data class TransitionDto(var from: Status? = null, var to: Status? = null)
```

---

## Проверки существования внешних сущностей: справочники/БД/кэши с таймаутами и деградацией

Часть правил требует знания **внешнего состояния**: «customerId существует», «товар активен», «страна поддерживается по справочнику». Такие проверки — уже не «чистая» Bean Validation, потому что зависят от БД/кэшей/HTTP. Тем не менее их удобно оформлять как кастомные constraints с инжекцией зависимостей, чтобы переиспользовать и держать код рядом с DTO.

Инжекция в валидатор работает благодаря `LocalValidatorFactoryBean`, который в Boot подключает Spring-aware фабрику валидаторов. Значит, в `ConstraintValidator` можно `@Autowired` репозитории/сервисы. Но держите в голове **перформанс**: валидация должна быть быстрой, а значит проверки наличия — с индексами, кэшами и таймаутами.

Проверка **уникальности** и **существования** — разные вещи. «Существование» — разрешение на ссылку (`FK-подобная` проверка), «уникальность» — запрет дубликатов. Для уникальности окончательная защита — уникальный индекс в БД; валидатор лишь «подсказывает» пользователю до сохранения.

Если внешний сервис недоступен, важно **не вешать** запрос. Либо возвращайте `503/504`, либо деградируйте (например, временно разрешайте «мягкие» проверки, если это безопасно). Валидация — не место для бесконечных ретраев; максимум — небольшой backoff и отказ.

Будьте осторожны с массовыми операциями (импорт CSV): наивная построчная проверка «существует ли» превратится в N запросов. Используйте батч-операции (IN-клаузы), кэш на время обработки файла и агрегатные запросы.

Храните в кэше **положительные** результаты коротко, отрицательные — ещё короче (или не кэшируйте, чтобы не закрепить временную «несуществуемость»). TTL и инвалидация — обязательны, иначе поймаете «фантомные» ошибки.

В ответе отдавайте только безопасные детали: «customerId not found», без попытки раскрыть внутренние атрибуты объекта. Параллельно логируйте `traceId` и факт отказа внешнего справочника для расследований.

И, конечно, покрывайте такие валидаторы интеграционными тестами с Testcontainers/WireMock, чтобы поймать дедлоки, таймауты и поведение на сетевых сбоях.

*Java — `@ExistingCustomerId` с инжекцией репозитория:*

```java
package com.example.cross.exist;

import jakarta.validation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.*;
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ExistingCustomerIdValidator.class)
public @interface ExistingCustomerId {
    String message() default "Customer does not exist";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
class ExistingCustomerIdValidator implements ConstraintValidator<ExistingCustomerId, String> {
    private final CustomerRepository repo;
    @Autowired ExistingCustomerIdValidator(CustomerRepository repo) { this.repo = repo; }
    @Override public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isBlank()) return true;
        return repo.existsById(value);
    }
}

interface CustomerRepository { boolean existsById(String id); /* реализуйте через JPA/Jdbc */ }

@RestController
@RequestMapping("/api/invoices")
class InvoiceController {
    @PostMapping
    public Invoice create(@ExistingCustomerId @RequestParam String customerId) {
        return new Invoice(customerId);
    }
    public record Invoice(String customerId) {}
}
```

*Kotlin — аналог:*

```kotlin
package com.example.cross.exist

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.stereotype.Component
import org.springframework.web.bind.annotation.*
import kotlin.reflect.KClass

@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [ExistingCustomerIdValidator::class])
annotation class ExistingCustomerId(
    val message: String = "Customer does not exist",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

@Component
class ExistingCustomerIdValidator @Autowired constructor(
    private val repo: CustomerRepository
) : ConstraintValidator<ExistingCustomerId, String> {
    override fun isValid(value: String?, context: ConstraintValidatorContext): Boolean =
        value.isNullOrBlank() || repo.existsById(value)
}

interface CustomerRepository { fun existsById(id: String): Boolean }

@RestController
@RequestMapping("/api/invoices")
class InvoiceController {
    @PostMapping
    fun create(@ExistingCustomerId @RequestParam customerId: String) = Invoice(customerId)
    data class Invoice(val customerId: String)
}
```
# 6. Входные каналы: что и как проверять

## Path/Query/Header/Cookie: типизация, обязательность, множества значений, enum-ы

Параметры пути (`@PathVariable`), строки запроса (`@RequestParam`), заголовки (`@RequestHeader`) и куки (`@CookieValue`) — самый «близкий к пользователю» слой ввода. Их сила в том, что Spring уже умеет типизировать значения: строки превращаются в числа, перечисления, даты. Но это не повод расслабляться: любое поле, пришедшее из сети, находится за границей доверия и требует явной политики обязательности и допустимых значений.

Для обязательности используйте явные контракты: «параметр обязателен» — тогда ставьте `required = true` (по умолчанию у `@RequestParam` так и есть) и валидацию `@NotBlank`/`@NotNull`. Если параметр опционален — лучше типизировать его как `Optional<T>` (Java) или `T?` (Kotlin), а не принимать пустую строку и пытаться догадаться. Чем точнее тип, тем меньше двусмысленностей.

С перечислениями (`enum`) старайтесь не принимать «сырые строки». Пусть Spring конвертирует значение к `EnumType`: тогда некорректные значения упадут с `400` ещё на этапе биндинга, а вы сохраните чистоту контроллера. Для дружелюбного DX добавьте хендлер, который превращает `MethodArgumentTypeMismatchException` в аккуратный `ProblemDetail` с перечислением разрешённых констант.

Для множественных значений (например, `?status=NEW&status=PAID`) используйте `List<T>` — Spring корректно соберёт коллекцию. Важно определиться с «правдой по умолчанию»: пустой список равно «нет фильтра» или «ничего не подходит»? Это бизнес-решение, которое стоит явно документировать и покрыть тестами.

С заголовками действует тот же принцип: минимально необходимый набор, строгие имена и валидация. «Идентификатор запроса», «идемпотентный ключ», «версия клиента» — это всё важные флаги, которые влияют на поведение. Не полагайтесь на «если пусто — придумаем сами»: клиент должен знать правила и коды ошибок.

Куки часто используют для сессий и трекинга. Даже если у вас stateless API, встречаются вспомогательные cookies. Проверяйте их присутствие только там, где это действительно требуется, и не смешивайте с авторизационными токенами — последние должны жить в `Authorization` и валидироваться отдельным слоем безопасности.

Типизация дат и времени в параметрах — частый источник сюрпризов. Если принимаете Date/Time в query, выбирайте ISO-8601 (`OffsetDateTime`, `LocalDate`) и включайте строгий формат (`@DateTimeFormat(iso = ISO.DATE)` для MVC). Любые «человеческие» форматы быстро ломают интеграции.

Наконец, не забывайте о разумных лимитах: длина параметров, количество значений в списках. Без ограничений недобросовестный клиент может закатить многокилобайтные query-string и заставить ваш сервер делать тяжёлые парсы и запросы.

*Gradle (MVC, валидация)*
**Groovy DSL**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — типизация и валидация Path/Query/Header/Cookie**

```java
package com.example.input.http;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MissingRequestCookieException;
import org.springframework.web.bind.MethodArgumentTypeMismatchException;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.util.List;
import java.util.Set;

@RestController
@RequestMapping("/api/orders")
public class OrderQueryController {

    enum Status { NEW, PAID, SHIPPED }

    @GetMapping("/{id}")
    public ResponseEntity<?> byId(@PathVariable("id") long id,
                                  @RequestHeader("X-Client-Version") @NotBlank String clientVersion,
                                  @CookieValue("session") String sessionId) {
        return ResponseEntity.ok().body(java.util.Map.of("id", id, "clientVersion", clientVersion, "session", sessionId));
    }

    @GetMapping
    public ResponseEntity<?> search(@RequestParam(defaultValue = "0") @Min(0) int page,
                                    @RequestParam(defaultValue = "20") @Max(100) int size,
                                    @RequestParam(required = false) List<Status> status,
                                    @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
                                    @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {
        if (from != null && to != null && !from.isBefore(to)) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setDetail("from must be before to");
            return ResponseEntity.unprocessableEntity().body(pd);
        }
        return ResponseEntity.ok().body(java.util.Map.of("page", page, "size", size, "status", status));
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    ProblemDetail onTypeMismatch(MethodArgumentTypeMismatchException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Bad parameter");
        if (ex.getRequiredType() != null && ex.getRequiredType().isEnum()) {
            @SuppressWarnings("rawtypes")
            Class<? extends Enum> et = (Class<? extends Enum>) ex.getRequiredType();
            pd.setDetail("Allowed values: " + Set.of(et.getEnumConstants()));
        } else {
            pd.setDetail("Parameter '" + ex.getName() + "' has wrong type");
        }
        return pd;
    }

    @ExceptionHandler(MissingRequestCookieException.class)
    ProblemDetail onMissingCookie(MissingRequestCookieException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Missing cookie");
        pd.setDetail(ex.getMessage());
        return pd;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.input.http

import jakarta.validation.constraints.Max
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.springframework.format.annotation.DateTimeFormat
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.MissingRequestCookieException
import org.springframework.web.bind.MethodArgumentTypeMismatchException
import org.springframework.web.bind.annotation.*
import java.time.LocalDate

@RestController
@RequestMapping("/api/orders")
class OrderQueryController {

    enum class Status { NEW, PAID, SHIPPED }

    @GetMapping("/{id}")
    fun byId(
        @PathVariable id: Long,
        @RequestHeader("X-Client-Version") @NotBlank clientVersion: String,
        @CookieValue("session") sessionId: String
    ): ResponseEntity<Any> =
        ResponseEntity.ok(mapOf("id" to id, "clientVersion" to clientVersion, "session" to sessionId))

    @GetMapping
    fun search(
        @RequestParam(defaultValue = "0") @Min(0) page: Int,
        @RequestParam(defaultValue = "20") @Max(100) size: Int,
        @RequestParam(required = false) status: List<Status>?,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) from: LocalDate?,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) to: LocalDate?
    ): ResponseEntity<Any> {
        if (from != null && to != null && !from.isBefore(to)) {
            val pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"; detail = "from must be before to"
            }
            return ResponseEntity.unprocessableEntity().body(pd)
        }
        return ResponseEntity.ok(mapOf("page" to page, "size" to size, "status" to status))
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException::class)
    fun onTypeMismatch(ex: MethodArgumentTypeMismatchException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            title = "Bad parameter"
            detail = if (ex.requiredType?.isEnum == true) {
                val allowed = ex.requiredType?.enumConstants?.toList()
                "Allowed values: $allowed"
            } else "Parameter '${ex.name}' has wrong type"
        }

    @ExceptionHandler(MissingRequestCookieException::class)
    fun onMissingCookie(ex: MissingRequestCookieException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            title = "Missing cookie"; detail = ex.message
        }
}
```

---

## Тело запроса: JSON-DTO, multipart (проверка MIME/размера/расширений), потоки

Тело запроса — основная «масса» данных. Для JSON действует привычная схема: DTO + Bean Validation. Важная деталь — нормализация и ограничения размеров (ниже). Не допускайте «свободного» `Map<String, Object>` без явной необходимости: строгая схема на входе делает API долговечнее.

С файлами (`multipart/form-data`) правила строже: проверяйте одновременно **MIME-типы**, **расширения** и **размеры**. Один из этих сигналов легко подменить; надёжность достигается комбинацией. MIME из заголовка — первое сито, расширение — второе, а при необходимости делайте сигнатурную проверку нескольких первых байтов (магические числа). В простых случаях достаточно MIME+расширение.

Лимиты должны стоять на двух уровнях: на сервере приложений (Tomcat/Netty) и на уровне кода (перед сохранением). В Spring MVC это `spring.servlet.multipart.max-file-size` и `max-request-size`; в коде — ручная проверка `file.getSize()` и отказ с `413 Payload Too Large` или `422`, если лимит — часть бизнес-правил.

Если компактность важна, требуйте архивы (zip) и проверяйте **внутренние** имена/размеры/тип содержимого до распаковки. Никогда не распаковывайте в произвольные директории без нормализации путей — защита от zip-slip (лучше распаковывать в temp и переносить только валидные файлы).

Стриминговые тела (NDJSON, большие JSON, бинарные потоки) требуют другого подхода: не буферизовать всё в памяти. Для MVC есть `InputStream` и стриминг в репозиторий; для WebFlux — реактивный `DataBuffer`. Но даже при стриминге делайте бюджет на максимальный размер и таймауты чтения — иначе DoS.

Контент-негациация: возвращайте `415 Unsupported Media Type` для неподдерживаемых `Content-Type`. Не пытайтесь «угадывать» JSON по расширению URL: доверяйте заголовкам и проверяйте их на входе.

Чувствительные загрузки (изображения аватаров и т. п.) требуют инициализации безопасных библиотек обработки (без выполнения скриптов в SVG, без внешних ссылок). Если не уверены — принимайте только «тупые» бинарники (JPEG/PNG) и перепаковывайте сами.

Логи и ошибки должны быть осторожны с файлами: не печатайте имена архивов/полные пути пользователя, не храните в логах байтовые первые N символов. Достаточно размера, MIME и хэша для расследования.

*Конфигурация лимитов (application.yml, MVC):*

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 5MB
      max-request-size: 6MB
```

**Java — JSON + multipart с проверкой**

```java
package com.example.input.body;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.Set;

@RestController
@RequestMapping("/api/profile")
public class ProfileController {

    private static final Set<String> ALLOWED_MIME = Set.of("image/png", "image/jpeg");
    private static final Set<String> ALLOWED_EXT = Set.of("png", "jpg", "jpeg");
    private static final long MAX_AVATAR = 1_000_000; // ~1MB

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<?> create(@RequestBody @Valid ProfileDto dto) {
        return ResponseEntity.status(HttpStatus.CREATED).body(dto);
    }

    @PostMapping(path = "/avatar", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<?> upload(@RequestParam("file") MultipartFile file) {
        String mime = file.getContentType();
        String ext = StringUtils.getFilenameExtension(file.getOriginalFilename());
        if (mime == null || !ALLOWED_MIME.contains(mime)) return bad("Unsupported MIME");
        if (ext == null || !ALLOWED_EXT.contains(ext.toLowerCase())) return bad("Unsupported extension");
        if (file.getSize() > MAX_AVATAR) return tooLarge("File too large");
        return ResponseEntity.ok().build();
    }

    private ResponseEntity<ProblemDetail> bad(String detail) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE);
        pd.setTitle("Invalid file");
        pd.setDetail(detail);
        return ResponseEntity.status(pd.getStatus()).body(pd);
    }

    private ResponseEntity<ProblemDetail> tooLarge(String detail) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.PAYLOAD_TOO_LARGE);
        pd.setTitle("Invalid file");
        pd.setDetail(detail);
        return ResponseEntity.status(pd.getStatus()).body(pd);
    }

    public record ProfileDto(@NotBlank String name, @Email String email, @Positive int age) {}
}
```

**Kotlin — аналог**

```kotlin
package com.example.input.body

import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.util.StringUtils
import org.springframework.web.bind.annotation.*
import org.springframework.web.multipart.MultipartFile

@RestController
@RequestMapping("/api/profile")
class ProfileController {

    private val allowedMime = setOf("image/png", "image/jpeg")
    private val allowedExt = setOf("png", "jpg", "jpeg")
    private val maxAvatar = 1_000_000L

    @PostMapping(consumes = [MediaType.APPLICATION_JSON_VALUE])
    fun create(@RequestBody @Valid dto: ProfileDto): ResponseEntity<Any> =
        ResponseEntity.status(HttpStatus.CREATED).body(dto)

    @PostMapping("/avatar", consumes = [MediaType.MULTIPART_FORM_DATA_VALUE])
    fun upload(@RequestParam("file") file: MultipartFile): ResponseEntity<Any> {
        val mime = file.contentType
        val ext = StringUtils.getFilenameExtension(file.originalFilename)
        if (mime == null || !allowedMime.contains(mime)) return bad("Unsupported MIME")
        if (ext == null || !allowedExt.contains(ext.lowercase())) return bad("Unsupported extension")
        if (file.size > maxAvatar) return tooLarge("File too large")
        return ResponseEntity.ok().build()
    }

    private fun bad(detail: String): ResponseEntity<ProblemDetail> =
        ResponseEntity.status(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
            .body(ProblemDetail.forStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE).apply {
                title = "Invalid file"; this.detail = detail
            })

    private fun tooLarge(detail: String): ResponseEntity<ProblemDetail> =
        ResponseEntity.status(HttpStatus.PAYLOAD_TOO_LARGE)
            .body(ProblemDetail.forStatus(HttpStatus.PAYLOAD_TOO_LARGE).apply {
                title = "Invalid file"; this.detail = detail
            })
}

data class ProfileDto(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Positive val age: Int
)
```

---

## Ограничения на размер/глубину/сложность JSON, защита от «больших» запросов

«Большие» запросы — частая причина деградации: клиент по ошибке или злоумышленник может отправить мегабайтные JSON с глубокой вложенностью и гигантскими числами. Это ударяет по памяти и CPU парсера. Защита строится слоями: сетевые лимиты (reverse-proxy), серверные лимиты (Tomcat/Netty) и **парсерные** лимиты (Jackson StreamReadConstraints).

Сетевой слой: в балансировщике/прокси ограничьте `max_body_size` и таймауты чтения. Это снимет часть нагрузки до вашего приложения. Даже если у вас нет внешнего прокси, встроенный сервер имеет параметры на максимальный размер запроса/заголовков (в Boot зависят от контейнера).

На уровне Spring MVC полезно ограничить `max-post-size` (для Tomcat — через свойство сервера) и отключить поддержку слишком больших буферов. Для WebFlux — `spring.codec.max-in-memory-size` (лимит на буферизацию при чтении тела). Но эти лимиты не защищают от «патологических» JSON с экстремальной вложенностью и длинными токенами.

Начиная с Jackson 2.15+ есть `StreamReadConstraints`: можно задать максимальную глубину вложенности, максимальную длину числовых значений, максимальную длину имени/строки. Это работает **внутри** парсера и отсекает опасные документы ещё до создания DTO. Рекомендуется выставлять ограниченные, но разумные значения (например, глубина 100, длина числа 1000 символов).

Отдельно задумайтесь о количестве элементов в коллекциях. Jackson не умеет ограничивать «число элементов» напрямую, но вы можете добавить «ранний» десериалайзер, который будет считать элементы и бросать исключение после N-го — или, что проще, принимать массивы только на уровнях, где у вас есть `@Size(max = …)`.

Генерация «безразмерных» логов сама по себе опасна. При ошибках парсера не печатайте весь текст запроса — достаточно первых N символов (или вообще хэш/размер). Иначе вы рискуете DoS уже на уровне логирования.

Документируйте и тестируйте лимиты. Клиенты должны понимать, почему «вчера проходило, а сегодня — 413». Хорошая практика — отдавать `ProblemDetail` с машиночитаемым кодом и ссылкой на документацию с лимитами.

Наконец, не подменяйте лимиты валидацией: попытка «считать вручную» все поля и проверять длины дороже, чем отрубить небезопасный документ на уровне парсера. Валидация — про содержание, лимиты — про форму и безопасность.

**Java — Jackson StreamReadConstraints + WebFlux лимит**

```java
package com.example.input.limits;

import com.fasterxml.jackson.core.StreamReadConstraints;
import com.fasterxml.jackson.core.json.JsonFactory;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.web.reactive.function.client.WebClientCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.codec.CodecConfigurer;
import org.springframework.http.codec.ClientCodecConfigurer;
import org.springframework.http.codec.support.DefaultClientCodecConfigurer;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;

@Configuration
public class JacksonLimitsConfig {

    @Bean
    ObjectMapper objectMapper() {
        StreamReadConstraints constraints = StreamReadConstraints.builder()
                .maxNestingDepth(100)
                .maxNumberLength(1000)
                .maxStringLength(1_000_000)
                .build();
        JsonFactory jf = JsonFactory.builder().streamReadConstraints(constraints).build();
        return Jackson2ObjectMapperBuilder.json().factory(jf).build();
    }
}
```

*Для WebFlux (application.yml):*

```yaml
spring:
  codec:
    max-in-memory-size: 2MB
server:
  max-http-request-header-size: 16KB
```

**Kotlin — конфигурация Jackson через builder customizer**

```kotlin
package com.example.input.limits

import com.fasterxml.jackson.core.StreamReadConstraints
import com.fasterxml.jackson.core.json.JsonFactory
import com.fasterxml.jackson.databind.ObjectMapper
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder

@Configuration
class JacksonLimitsConfig {

    @Bean
    fun objectMapper(): ObjectMapper {
        val constraints = StreamReadConstraints.builder()
            .maxNestingDepth(100)
            .maxNumberLength(1000)
            .maxStringLength(1_000_000)
            .build()
        val jf = JsonFactory.builder().streamReadConstraints(constraints).build()
        return Jackson2ObjectMapperBuilder.json().factory(jf).build()
    }
}
```

---

## Идемпотентность и безопасные повторы: Idempotency-Key валидации для мутаций

Идемпотентность — свойство операции давать один и тот же эффект при повторном выполнении. Для мутирующих HTTP-запросов (POST/PUT/DELETE) это критически важно: сети ненадёжны, ретраи неизбежны. Паттерн — заголовок `Idempotency-Key`, уникальный для семантически одной операции. Сервер должен «запомнить» обработанные ключи и при повторе вернуть тот же результат или конфликт.

Валидация ключа — часть границы доверия: он должен присутствовать для всех мутаций (по вашему контракту), соответствовать формату (UUID/случайная строка фиксированной длины) и иметь TTL в сторе. Ключ должен идентифицировать **намерение**, а не попытку: повторный POST с тем же ключом — это не новая операция.

Где хранить ключи? Минимум — в памяти (ConcurrentHashMap) с регулярной очисткой, достаточно для одного инстанса. В продакшне — Redis/БД/распределённый кэш, чтобы несколько экземпляров сервиса видели один и тот же реестр ключей. Хранить стоит «ключ → результат/статус/ресурсId», чтобы на повтор быстро отдать кешированный ответ.

Статусы ответов: если операция уже отработала — возвращайте **тот же** HTTP-статус и тело; если она ещё в процессе — можно вернуть 409/425 (Too Early) с предложением повторить позже; если ключ конфликтует по контексту (другая операция) — 409 c деталью. В любом случае не выполняйте побочные эффекты повторно.

Ключи должны иметь разумный TTL (минуты/часы в зависимости от SLA). Слишком короткий TTL обнулит защиту, слишком длинный — забьёт хранилище. Выбор — компромисс между нагрузкой и риском двойного исполнения.

Обязательно логируйте и метрики: доля повторов, конфликты, «долго висящие» ключи. Это помогает наблюдать сетевые проблемы и неправильных клиентов.

И, конечно, тестируйте гонки: параллельные запросы с одинаковым `Idempotency-Key` должны приводить к одному исполнению. Это важно при высоких нагрузках и штормах ретраев.

**Java — простой фильтр/интерсептор Idempotency-Key**

```java
package com.example.input.idem;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class IdempotencyInterceptor implements HandlerInterceptor {

    private final Map<String, Result> store = new ConcurrentHashMap<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String method = request.getMethod();
        if (!("POST".equals(method) || "PUT".equals(method) || "DELETE".equals(method))) return true;

        String key = request.getHeader("Idempotency-Key");
        if (key == null || key.isBlank()) {
            write(response, HttpStatus.CONFLICT, "Missing Idempotency-Key");
            return false;
        }
        Result existed = store.get(key);
        if (existed != null) {
            response.setStatus(existed.status);
            response.setContentType("application/json");
            try {
                response.getWriter().write(existed.body);
            } catch (Exception ignored) {}
            return false;
        }
        // помечаем ключ как «в процессе»
        store.putIfAbsent(key, new Result(HttpStatus.ACCEPTED.value(), "{\"status\":\"processing\"}"));
        return true;
    }

    public void saveResult(String key, HttpStatus status, String body) {
        store.put(key, new Result(status.value(), body));
    }

    private void write(HttpServletResponse res, HttpStatus st, String detail) {
        try {
            res.setStatus(st.value());
            res.setContentType("application/problem+json");
            res.getWriter().write("{\"title\":\"Idempotency error\",\"status\":"+st.value()+",\"detail\":\""+detail+"\"}");
        } catch (Exception ignored) {}
    }

    private record Result(int status, String body) {}
}
```

**Kotlin — аналог**

```kotlin
package com.example.input.idem

import org.springframework.http.HttpStatus
import org.springframework.stereotype.Component
import org.springframework.web.servlet.HandlerInterceptor
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import java.util.concurrent.ConcurrentHashMap

@Component
class IdempotencyInterceptor : HandlerInterceptor {

    private val store = ConcurrentHashMap<String, Result>()

    override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
        val method = request.method
        if (method != "POST" && method != "PUT" && method != "DELETE") return true

        val key = request.getHeader("Idempotency-Key")
        if (key.isNullOrBlank()) {
            write(response, HttpStatus.CONFLICT, "Missing Idempotency-Key")
            return false
        }
        val existed = store[key]
        if (existed != null) {
            response.status = existed.status
            response.contentType = "application/json"
            response.writer.write(existed.body)
            return false
        }
        store.putIfAbsent(key, Result(HttpStatus.ACCEPTED.value(), """{"status":"processing"}"""))
        return true
    }

    fun saveResult(key: String, status: HttpStatus, body: String) {
        store[key] = Result(status.value(), body)
    }

    private fun write(res: HttpServletResponse, st: HttpStatus, detail: String) {
        res.status = st.value()
        res.contentType = "application/problem+json"
        res.writer.write("""{"title":"Idempotency error","status":${st.value()},"detail":"$detail"}""")
    }

    private data class Result(val status: Int, val body: String)
}
```

---

# 7. Реактивность и асинхронность (WebFlux, @Async, messaging)

## WebFlux: привязка ошибок (WebExchangeBindException), валидация в функциональных маршрутах

В WebFlux валидация работает схоже с MVC, но детали отличаются. При ошибках биндинга/валидации Spring кидает `WebExchangeBindException`. Это исключение содержит `FieldError`/`ObjectError` с путями до полей — на его основе легко сформировать `ProblemDetail` с массивом `errors`. В `@RestController` с аннотациями (`@RequestBody @Valid`) всё прозрачно, а вот в **функциональном стиле** (Router/Handler) валидацию нужно вызвать вручную.

Функциональные маршруты (`RouterFunction`) дают тонкий контроль над пайплайном: вы читаете тело как `Mono<DTO>`, применяете проверку (`validator.validate(dto)`), и в случае нарушений — формируете реактивный ответ с `422`. Такой подход полезен, когда нужна особая форма ошибок, трассировка или дополнительная нормализация.

Важно помнить, что `Validator` — синхронный и лёгкий. В реактивной цепочке проверка должна оставаться «холостой» по IO, иначе вы получите блокировки. Никаких обращений к БД/HTTP из валидатора — только синтаксис/семантика без внешних зависимостей.

Глобальная обработка исключений в WebFlux реализуется через `@RestControllerAdvice` так же, как в MVC, но типы исключений другие. Перехватывайте `WebExchangeBindException`, `ServerWebInputException` (например, когда прислали `text/plain` вместо JSON), и возвращайте стандартизованный `ProblemDetail`.

Не забывайте про `content-type`: реактивный стек строже относится к медиатипам. Если клиент не указал `Content-Type: application/json`, десериализация может не запуститься. Это хорошо: вы возвращаете `415` и не тратите CPU на гадание формата.

Функциональные маршруты удобны для построения «слоёных» обработчиков: нормализация → валидация → домен → сериализация. Такая композиция делает код модульным и хорошо тестируемым, особенно при большом количестве endpoinтов.

Отдельно проверьте поведение при «больших» телах: для WebFlux это `spring.codec.max-in-memory-size`. Валидация не должна запускаться, если сообщение уже отрезано по лимитам. Порядок важен: сначала сетевые ограничения, затем десериализация, затем валидация.

Наконец, учитывайте back-pressure. Если вы валидируете поток элементов (NDJSON, SSE), нарушивший элемент должен закрывать поток с ошибкой — иначе клиент будет «думать», что часть элементов валидна. Контракт потоковых API следует обсуждать и документировать отдельно.

*Gradle (WebFlux)*
**Groovy DSL**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — функциональные маршруты с валидацией и глобальным хендлером**

```java
package com.example.webflux.routes;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validator;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.reactive.function.server.*;
import reactor.core.publisher.Mono;

import java.util.Map;

@Configuration
public class UserRoutes {

    @Bean
    RouterFunction<ServerResponse> routes(UserHandler h) {
        return RouterFunctions.route()
                .POST("/r/users", h::create)
                .build();
    }

    static record UserReq(@NotBlank String name, @Email String email) {}

    @Bean
    UserHandler userHandler(Validator validator) { return new UserHandler(validator); }

    static class UserHandler {
        private final Validator validator;
        UserHandler(Validator validator) { this.validator = validator; }

        public Mono<ServerResponse> create(ServerRequest req) {
            return req.bodyToMono(UserReq.class)
                    .flatMap(dto -> {
                        var violations = validator.validate(dto);
                        if (!violations.isEmpty()) return ServerResponse.status(422)
                                .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                                .bodyValue(problemFrom(violations));
                        return ServerResponse.status(201).contentType(MediaType.APPLICATION_JSON).bodyValue(dto);
                    });
        }

        private ProblemDetail problemFrom(java.util.Set<ConstraintViolation<UserReq>> v) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setProperty("errors", v.stream().map(cv -> Map.of("field", cv.getPropertyPath().toString(), "message", cv.getMessage())).toList());
            return pd;
        }
    }
}

@RestControllerAdvice
class WebFluxAdvice {

    @ExceptionHandler(org.springframework.web.bind.support.WebExchangeBindException.class)
    ProblemDetail onBind(org.springframework.web.bind.support.WebExchangeBindException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setProperty("errors", ex.getFieldErrors().stream().map(fe -> Map.of("field", fe.getField(), "message", fe.getDefaultMessage())).toList());
        return pd;
    }

    @ExceptionHandler(org.springframework.web.server.UnsupportedMediaTypeStatusException.class)
    ProblemDetail onMedia(org.springframework.web.server.UnsupportedMediaTypeStatusException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE);
        pd.setTitle("Unsupported media type");
        pd.setDetail(ex.getMessage());
        return pd;
    }
}
```

**Kotlin — функциональные маршруты и advice**

```kotlin
package com.example.webflux.routes

import jakarta.validation.ConstraintViolation
import jakarta.validation.Validator
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import org.springframework.web.reactive.function.server.*
import reactor.core.publisher.Mono

@Configuration
class UserRoutes {

    data class UserReq(@field:NotBlank val name: String, @field:Email val email: String)

    @Bean
    fun routes(h: UserHandler): RouterFunction<ServerResponse> =
        coRouter {
            POST("/r/users", h::create)
        }

    @Bean
    fun userHandler(validator: Validator) = UserHandler(validator)

    class UserHandler(private val validator: Validator) {
        fun create(req: ServerRequest): Mono<ServerResponse> =
            req.bodyToMono(UserReq::class.java)
                .flatMap { dto ->
                    val v = validator.validate(dto)
                    if (!v.isEmpty()) {
                        ServerResponse.status(422).contentType(MediaType.APPLICATION_PROBLEM_JSON).bodyValue(problemFrom(v))
                    } else ServerResponse.status(201).contentType(MediaType.APPLICATION_JSON).bodyValue(dto)
                }

        private fun problemFrom(v: Set<ConstraintViolation<UserReq>>): ProblemDetail =
            ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"
                setProperty("errors", v.map { mapOf("field" to it.propertyPath.toString(), "message" to it.message) })
            }
    }
}

@RestControllerAdvice
class WebFluxAdvice {
    @ExceptionHandler(org.springframework.web.bind.support.WebExchangeBindException::class)
    fun onBind(ex: org.springframework.web.bind.support.WebExchangeBindException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("errors", ex.fieldErrors.map { mapOf("field" to it.field, "message" to it.defaultMessage) })
        }

    @ExceptionHandler(org.springframework.web.server.UnsupportedMediaTypeStatusException::class)
    fun onMedia(ex: org.springframework.web.server.UnsupportedMediaTypeStatusException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE).apply {
            title = "Unsupported media type"; detail = ex.message
        }
}
```

---

## Reactor: недопустимость блокировок; валидация как «лёгкий» оператор в цепочке, таймауты/ретраи

Reactor предполагает неблокирующий стиль. Любая блокировка (`.block()`, JDBC, файловая система без offloading) разрушает модель back-pressure и отбирает потоки у соседей. Валидация в реактивной цепочке должна быть **лёгкой**: только CPU, без IO. Если вам нужно проверить что-то во внешней системе, вынесите это в отдельный пул (`publishOn(Schedulers.boundedElastic())`) или, лучше, в асинхронный клиент.

Валидацию удобно оформлять как оператор `flatMap(dto -> validate(dto).thenReturn(dto))`, где `validate` возвращает `Mono<Void>` и сигнализирует об ошибке через `Mono.error`. Это компонуется, легко тестируется и прозрачно в трассировке. «Мягкие» проверки можно делать синхронно через `handle`, выбрасывая исключения.

Таймауты — обязательны на **каждом** внешнем вызове: `.timeout(Duration.ofSeconds(2))`. Без них поток может занять соединение «навечно». Комбинируйте таймауты с ретраями (`retryWhen`) только для транзитных ошибок, фильтруя по типам исключений. При валидации сетевых справочников повтор часто бессмысленен — лучше вернуть `503` с `Retry-After`.

Старайтесь не «размазывать» валидацию по цепочке. Соберите небольшой «валидационный блок» в одном месте: нормализация → синтаксис → семантика → домен. Это облегчает чтение и профилирование (видно, где тратится время).

Для диагностики используйте `checkpoint("validation")`: если что-то пойдёт не так, стек будет понятнее. В dev-окружении можно включить `Hooks.onOperatorDebug()`, но в проде — только `checkpoint`, он лёгкий.

Ошибки валидации должны доходить до глобального хендлера (см. предыдущие главы) как специализированные исключения, которые мапятся в `422`. Не превращайте ошибки в «успех с флагом» — это ломает контракты клиента.

Если валидация очень тяжёлая по CPU (сложные regex, большие структуры), рассмотрите offloading на `boundedElastic` и ограничение параллелизма. Но чаще всего правильнее **упростить правила**, чем бороться за CPU.

И наконец, помните о back-pressure при потоках (`Flux`): если вы валидируете элементы, каждая ошибка должна сигнализировать `onError` и завершать поток, если контракт не предусматривает частичный успех. Документируйте поведение.

**Java — реактивная валидация как оператор**

```java
package com.example.webflux.reactor;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validator;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.util.Map;
import java.util.Set;

@RestController
@RequestMapping("/rx/users")
class RxController {
    private final Validator validator;
    private final ExternalCatalog catalog;

    RxController(Validator validator, ExternalCatalog catalog) {
        this.validator = validator;
        this.catalog = catalog;
    }

    @PostMapping
    Mono<User> create(@RequestBody Mono<User> in) {
        return in.flatMap(this::validateDto)
                .publishOn(Schedulers.boundedElastic())
                .flatMap(this::checkCatalog) // внешняя проверка
                .timeout(java.time.Duration.ofSeconds(2))
                .checkpoint("validation")
                .onErrorMap(e -> e); // мапьте в доменные исключения по необходимости
    }

    private Mono<User> validateDto(User u) {
        Set<ConstraintViolation<User>> v = validator.validate(u);
        if (!v.isEmpty()) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setProperty("errors", v.stream().map(cv -> Map.of("field", cv.getPropertyPath().toString(), "message", cv.getMessage())).toList());
            return Mono.error(new ValidationProblem(pd));
        }
        return Mono.just(u);
    }

    private Mono<User> checkCatalog(User u) {
        // имитация внешней проверки (например, «страна существует»)
        if (!catalog.existsCountry(u.country())) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Unknown country"); pd.setDetail(u.country());
            return Mono.error(new ValidationProblem(pd));
        }
        return Mono.just(u);
    }

    record User(@NotBlank String name, @NotBlank String country) {}
    interface ExternalCatalog { boolean existsCountry(String code); }
    static class ValidationProblem extends RuntimeException {
        final ProblemDetail pd; ValidationProblem(ProblemDetail pd){ this.pd=pd; }
        public ProblemDetail problem() { return pd; }
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.webflux.reactor

import jakarta.validation.ConstraintViolation
import jakarta.validation.Validator
import jakarta.validation.constraints.NotBlank
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers

@RestController
@RequestMapping("/rx/users")
class RxController(
    private val validator: Validator,
    private val catalog: ExternalCatalog
) {

    @PostMapping
    fun create(@RequestBody inMono: Mono<User>): Mono<User> =
        inMono
            .flatMap { validateDto(it) }
            .publishOn(Schedulers.boundedElastic())
            .flatMap { checkCatalog(it) }
            .timeout(java.time.Duration.ofSeconds(2))
            .checkpoint("validation")

    private fun validateDto(u: User): Mono<User> {
        val v: Set<ConstraintViolation<User>> = validator.validate(u)
        return if (v.isNotEmpty()) {
            val pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"
                setProperty("errors", v.map { mapOf("field" to it.propertyPath.toString(), "message" to it.message) })
            }
            Mono.error(ValidationProblem(pd))
        } else Mono.just(u)
    }

    private fun checkCatalog(u: User): Mono<User> =
        if (!catalog.existsCountry(u.country)) {
            val pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Unknown country"; detail = u.country
            }
            Mono.error(ValidationProblem(pd))
        } else Mono.just(u)
}

data class User(@field:NotBlank val name: String, @field:NotBlank val country: String)
fun interface ExternalCatalog { fun existsCountry(code: String): Boolean }
class ValidationProblem(val problem: ProblemDetail) : RuntimeException()
```

---

## Сообщения (Kafka/JMS): валидация событий до обработки; схемы (JSON Schema/Avro/Proto) на границе

Событийная интеграция требует той же дисциплины: всё, что приходит из шины, недоверенно. Лучшая защита — **схема** на границе: Avro/Proto с registry или JSON Schema. Тогда сообщение отфильтруется на десериализации, а ваш код увидит типизированный объект — как и в REST.

Даже с типизированной схемой сохраняются семантические правила: «статус допустим», «сумма неотрицательна», «ссылка на внешний ресурс существует». Эти проверки выполняются **до** побочных эффектов: запись в БД, вызовы внешних сервисов. Нарушения — в DLQ (см. следующий пункт) или «parking lot» с ручной обработкой.

В Spring Kafka удобно вешать Bean Validation прямо на параметр обработчика: `@KafkaListener` + `@Valid` на DTO. Валидационные исключения можно перехватывать кастомным `ErrorHandler`/`DefaultErrorHandler` и перенаправлять в DLQ. Это создаёт единый контракт ошибок и снижает «ядовитость» сообщений.

С JSON без registry полезно держать «тонкий» валидатор схемы (например, Everit) на входе консьюмера. Но даже без внешних зависимостей вы можете обойтись типизированным DTO + Bean Validation. Главное — не принимайте «сырой» `String` и не парсьте вручную.

Версионирование схем — отдельный вопрос. События — долговечные; меняйте их **эволюционно**: добавляйте новые поля как опциональные, старые — помечайте как deprecated и не ломайте читателей. Валидация должна быть совместима: не ругаться на «неизвестные» поля и терпеть отсутствующие «новые».

Нагрузка и перформанс: не пытайтесь в каждом валидаторе ходить в БД. Агрегируйте справочные проверки, кешируйте и ограничивайте время. Если зависимость недоступна — отправьте сообщение в повторную очередь с задержкой, а не блокируйте поток.

Логи, метрики и трассировка: у каждого «отбракованного» сообщения должен быть reason-code и счётчик. Это помогает видеть плохих производителей и деградации. В DLQ добавляйте заголовки с причиной, версией схемы, traceId.

Наконец, тестируйте консьюмеры интеграционно: Testcontainers для Kafka, отправка валидных и невалидных сообщений, проверка, что «плохие» гарантированно оказываются в DLQ, а «хорошие» доходят до бизнес-обработки.

*Gradle (Kafka)*
**Groovy DSL**

```groovy
dependencies {
    implementation 'org.springframework.kafka:spring-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — консьюмер с Bean Validation**

```java
package com.example.messaging.validation;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class OrdersConsumer {

    @KafkaListener(topics = "orders", groupId = "orders-val")
    public void onOrder(@Payload @Valid OrderEvent evt, ConsumerRecord<String, OrderEvent> raw) {
        // если дошли сюда — синтаксис валиден
        // дополнительные семантические проверки и обработка
    }

    public record OrderEvent(@NotBlank String id, @Positive long amount, @NotBlank String currency) {}
}
```

**Kotlin — аналог**

```kotlin
package com.example.messaging.validation

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.messaging.handler.annotation.Payload
import org.springframework.stereotype.Component

@Component
class OrdersConsumer {

    @KafkaListener(topics = ["orders"], groupId = "orders-val")
    fun onOrder(@Payload @Valid evt: OrderEvent, raw: ConsumerRecord<String, OrderEvent>) {
        // синтаксис валиден; делаем семантику/бизнес
    }
}

data class OrderEvent(
    @field:NotBlank val id: String,
    @field:Positive val amount: Long,
    @field:NotBlank val currency: String
)
```

---

## Поведение при сбоях: DLQ/parking lot для «ядовитых» сообщений, метрики отклонений

«Ядовитые» сообщения — те, которые системно не проходят валидацию (сломанный формат, недопустимая комбинация полей). Ретраи им не помогут, и их нужно **изолировать**: отправить в Dead Letter Queue (DLQ) или «parking lot» для ручной обработки. Это сохраняет пропускную способность основного потока и упрощает расследования.

В Spring Kafka есть готовая инфраструктура: `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`. Вы настраиваете количество попыток и стратегию (экспоненциальный backoff), а при исчерпании — сообщение уходит в `topic.DLT`. В DLQ полезно добавлять заголовки с причиной (`x-exception-class`, `x-exception-message`), трассировкой и оригинальным топиком/партицией/offset.

Для временных сбоев (недоступна БД/справочник) уместны повторные попытки и «delay queue». Но важно различать «временно» и «навсегда»: избыточные ретраи ядовых сообщений только засоряют лог и тормозят обработку. Хорошая эвристика — классификация исключений: `ValidationException` → сразу DLQ, `TransientException` → ретраи.

Метрики отклонений — ключ к наблюдаемости. Снимайте счётчики по **типу причины**, **топику**, **производителю**. В дашбордах быстро увидите всплески и сможете принять меры: отключить «шумного» клиента, обновить справочник, расширить лимиты.

Про «parking lot»: иногда нужно отложить сообщение для ручного разбора, не теряя его. Это отдельная очередь/топик, в который раскладываются спорные кейсы. У него другой SLA, и доступ к нему имеет узкая группа поддержки/аналитики.

Не забывайте про GDPR/PII: в DLQ и логах не должно быть чувствительных данных. Если событие содержит PII, обнуляйте/маскируйте поля при публикации в DLQ или вовсе храните ссылку на исходник в безопасном хранилище.

Тестируйте обработку сбоев: unit — на классификацию исключений, интеграция — что ядовые сообщения действительно оказываются в DLT, а «временные» — ретраятся заданное число раз. Это спасёт от неожиданностей в проде.

И наконец, держите процедуры «очистки» DLQ: периодически выгружайте, анализируйте, исправляйте первопричины. Мёртвые очереди не должны разрастаться бесконтрольно.

**Java — DLQ-конфигурация Spring Kafka (DefaultErrorHandler)**

```java
package com.example.messaging.dlq;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.util.backoff.ExponentialBackOff;

@Configuration
public class KafkaErrorHandlingConfig {

    @Bean
    DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
                template,
                (record, ex) -> new org.apache.kafka.common.TopicPartition(record.topic() + ".DLT", record.partition())
        );
        ExponentialBackOff backOff = new ExponentialBackOff(100L, 2.0);
        backOff.setMaxElapsedTime(2000L);
        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
        handler.addNotRetryableExceptions(javax.validation.ValidationException.class);
        return handler;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.messaging.dlq

import org.apache.kafka.common.TopicPartition
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer
import org.springframework.kafka.listener.DefaultErrorHandler
import org.springframework.util.backoff.ExponentialBackOff

@Configuration
class KafkaErrorHandlingConfig {

    @Bean
    fun errorHandler(template: KafkaTemplate<Any, Any>): DefaultErrorHandler {
        val recoverer = DeadLetterPublishingRecoverer(template) { record, _ ->
            TopicPartition(record.topic() + ".DLT", record.partition())
        }
        val backoff = ExponentialBackOff(100L, 2.0).apply { maxElapsedTime = 2000L }
        return DefaultErrorHandler(recoverer, backoff).apply {
            addNotRetryableExceptions(jakarta.validation.ValidationException::class.java)
        }
    }
}
```

# 8. Ошибки и контракт ответа

## Единый формат ошибок: RFC 7807 (ProblemDetail) с полями field/code для валидации

Единый формат ошибок — это фундамент договорённостей между клиентом и сервером. В мире HTTP уже есть стандарт RFC 7807, который описывает структуру «проблемы» с ключевыми полями `type`, `title`, `status`, `detail`, `instance`. Мы используем его как базу и дополняем машинными деталями валидации (`errors`: массив объектов `{field, code, message}`), чтобы фронт и интеграторы могли автоматически подсвечивать поля и строить UX.

Главная польза стандарта — предсказуемость. Клиенту не нужно «угадывать» формат, он парсит единое поле `errors` и одинаковые «шапки» проблем. Это снижает когнитивную нагрузку и уменьшает количество «if-else» в клиентах. А ещё такой ответ легко логировать и агрегировать метриками по `type`/`code`.

Важно отделять человеко-читаемый `detail` (для быстрого понимания) от машинных кодов в `errors`. `detail` может быть локализованным, а `errors[i].code` — стабильным техническим идентификатором правила, который не меняется от релиза к релизу. Клиенты опираются на код, а текст показывают из своего словаря.

Поле `instance` удобно заполнять абсолютным URL запроса или его канонической формой. Это помогает трассировать ошибки и сопоставлять их с access-логами и APM. Поле `type` можно указывать как ссылку на страницу документации (даже если страница пока-заглушка) — это повышает самообучаемость интеграторов.

Для ошибок валидации полезно добавлять `errors[].field` в форме путей типа `payload.address.zip` или `items[3].sku`. Такое имя поля извлекается из `BindingResult`/`WebExchangeBindException` и даёт фронту точный якорь для подсветки. Если ошибка class-level, можно использовать `field=null` или имена виртуальных полей (`range`).

Не забывайте про расширяемость: Spring `ProblemDetail` позволяет задавать произвольные свойства через `setProperty`. Сюда мы добавляем `errors`, `traceId`, `timestamp`, а при необходимости — ссылки на документацию `docs`.

Ошибки должны быть консистентны для всех входных каналов: HTTP MVC/WebFlux, обработчики сообщений, CLI. Даже если транспорт другой, структура «проблемы» остаётся прежней — так проще сопровождать и диагностировать инциденты.

В логике возврата статуса оставляйте минимализм: `status` в шапке и уже внутри `errors` — конкретные коды правил. Не смешивайте бизнес-коды и HTTP: первое — про домен, второе — про транспорт и семантику протокола.

**Gradle (зависимости для ProblemDetail и валидации)**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'       // MVC + ProblemDetail
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — централизованный формат RFC 7807 с errors[]**

```java
package com.example.errors.contract;

import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.time.OffsetDateTime;
import java.util.Map;

@RestControllerAdvice
public class GlobalErrorHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onInvalid(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setType(URI.create("https://docs.example.com/problems/validation-error"));
        pd.setTitle("Validation failed");
        pd.setDetail("Payload contains invalid fields");
        pd.setInstance(URI.create(ex.getParameter().getExecutable().toGenericString()));
        pd.setProperty("timestamp", OffsetDateTime.now().toString());
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> Map.of(
                        "field", fe.getField(),
                        "code", fe.getCode(),              // машинный код правила (Size, NotBlank, Pattern…)
                        "message", fe.getDefaultMessage()))
                .toList());
        return pd;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.errors.contract

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.net.URI
import java.time.OffsetDateTime

@RestControllerAdvice
class GlobalErrorHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun onInvalid(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            type = URI.create("https://docs.example.com/problems/validation-error")
            title = "Validation failed"
            detail = "Payload contains invalid fields"
            instance = URI.create(ex.parameter.executable.toGenericString())
            setProperty("timestamp", OffsetDateTime.now().toString())
            setProperty("errors", ex.bindingResult.fieldErrors.map {
                mapOf("field" to it.field, "code" to it.code, "message" to it.defaultMessage)
            })
        }
}
```

---

## Разделять 400 (некорректный формат) и 422 (семантическая ошибка), 409 (конфликт уникальности)

HTTP-статус — это «светофор» для машин. `400 Bad Request` уместен, когда сервер вообще не может прочитать или распарсить запрос: сломанный JSON, неподдерживаемый медиатип, неправильный тип параметра. Здесь бизнес-логика даже не стартовала, поэтому `ProblemDetail` описывает **формат** и `errors` обычно пустой либо содержит параметры запроса.

`422 Unprocessable Entity` — когда JSON валиден, типы сходятся, но значения нарушают правила валидации/семантики. Это основной статус для результата `Bean Validation` (DTO прошло десериализацию, но не прошло проверку) и бизнес-инвариантов. Клиент может исправить данные и повторить.

`409 Conflict` — для конфликтов состояния: нарушена уникальность (уникальный ключ уже занят), версия ресурса не совпала (optimistic lock), переход состояния недопустим. Здесь важно не смешивать с `422`: в 409 клиенту, как правило, нужно синхронизироваться с актуальным состоянием, а не «исправлять форму».

Важная грань: `404 Not Found` — отсутствие ресурса, а не «не прошёл формат идентификатора». Если `id` пришёл строкой `abc` там, где ожидается long, это `400` (тип не тот); если `id` валиден по типу, но ресурса нет — `404`. Соблюдение этого правила делает API предсказуемым.

`415 Unsupported Media Type` и `406 Not Acceptable` — тоже часть «форматного» слоя. Если клиент шлёт `text/plain`, а вы ждёте JSON, — это 415. Если он просит `Accept: application/xml`, а вы умеете только JSON, — 406. Эти статусы помогают API само-документироваться через HTTP-диалог.

Статус `500` оставляйте для неожиданных сбоев сервера. Никогда не используйте его, чтобы обозначить ошибки пользователя — это ломает SLO и наблюдаемость. Да, иногда быстрее «всё маппить на 500», но это путь к бездне.

В логике маппинга исключений полезно иметь «таблицу»: `MethodArgumentNotValidException → 422`, `HttpMessageNotReadableException → 400`, `MethodArgumentTypeMismatchException → 400`, `DataIntegrityViolationException → 409`. Так вы избежите дрейфа статусов между командами.

Наконец, тесты должны фиксировать статусы: contract/slice-тесты на контроллеры и интеграционные тесты на сценарии уникальности/конфликтов. Любое изменение статуса — это потенциальный breaking change для клиентов.

**Gradle**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    runtimeOnly 'org.postgresql:postgresql' // для примера 409
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    runtimeOnly("org.postgresql:postgresql")
}
```

**Java — маппинг статусов на исключения**

```java
package com.example.errors.statuses;

import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.*;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingRequestHeaderException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;

@RestControllerAdvice
class StatusAdvice {

    @ExceptionHandler(HttpMessageNotReadableException.class)
    ProblemDetail unreadable(HttpMessageNotReadableException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Malformed request");
        pd.setDetail(ex.getMostSpecificCause().getMessage());
        return pd;
    }

    @ExceptionHandler({MethodArgumentNotValidException.class})
    ProblemDetail validation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> java.util.Map.of("field", fe.getField(), "code", fe.getCode()))
                .toList());
        return pd;
    }

    @ExceptionHandler({MethodArgumentTypeMismatchException.class, MissingRequestHeaderException.class})
    ProblemDetail badParams(Exception ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Bad parameter");
        pd.setDetail(ex.getMessage());
        return pd;
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    ProblemDetail conflict(DataIntegrityViolationException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        pd.setTitle("Conflict");
        pd.setDetail("Unique constraint violated or state conflict");
        return pd;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.errors.statuses

import org.springframework.dao.DataIntegrityViolationException
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.converter.HttpMessageNotReadableException
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.MissingRequestHeaderException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException

@RestControllerAdvice
class StatusAdvice {

    @ExceptionHandler(HttpMessageNotReadableException::class)
    fun unreadable(ex: HttpMessageNotReadableException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            title = "Malformed request"; detail = ex.mostSpecificCause.message
        }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun validation(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("errors", ex.bindingResult.fieldErrors.map { mapOf("field" to it.field, "code" to it.code) })
        }

    @ExceptionHandler(MethodArgumentTypeMismatchException::class, MissingRequestHeaderException::class)
    fun badParams(ex: Exception): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply { title = "Bad parameter"; detail = ex.message }

    @ExceptionHandler(DataIntegrityViolationException::class)
    fun conflict(@Suppress("UNUSED_PARAMETER") ex: DataIntegrityViolationException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.CONFLICT).apply {
            title = "Conflict"; detail = "Unique constraint violated or state conflict"
        }
}
```

---

## Понятные коды ошибок для клиентов (machine-readable), без утечки PII

Машиночитаемые коды — это контракт, на который клиенты могут опираться. В отличие от текстов, коды стабильны и не зависят от локали. Они помогают фронту выбирать правильный сценарий: подсветить конкретное поле, показать конкретную подсказку, включить/выключить кнопку. Хороший код — короткий, в SCREAMING_SNAKE_CASE и привязан к правилу, а не к тексту: `EMAIL_FORMAT_INVALID`, `FROM_AFTER_TO`, `CURRENCY_NOT_SUPPORTED`.

Храните коды в едином перечислении или словаре и переиспользуйте по всему проекту. Это исключит дубли и даст быструю навигацию по использованию. Чем меньше «произвольных строк» по коду, тем легче жить поддержке.

Коды нужны не только в `errors[]`, но и на уровне `ProblemDetail` в свойстве `code` — это «мастерхук» для клиентов, когда нет привязки к полям (например, при `409 CONFLICT` или `503 SERVICE UNAVAILABLE`). Тогда фронт может отобразить общий баннер и предложить ретрай.

Никогда не кладите в сообщения/коды персональные данные: email, телефон, полное имя. И в логах — тоже. Если очень нужно отдать обратно значение, замаскируйте его (`user****@domain.tld`). В `errors[].message` держите нейтральные тексты.

Полезно ввести «пространства имён» кодов: `VALIDATION.*`, `AUTH.*`, `ACCESS.*`, `CONFLICT.*`. Это даёт фильтры в метриках и логах, а также облегчает поиск первопричин.

Версионируйте коды осторожно: удаление/переименование — breaking change для клиентов. Лучше добавлять новые и объявлять старые deprecated в документации. Если всё же меняете — делайте миграционную таблицу.

Документируйте коды на внутреннем портале: пример запроса/ответа, бизнес-смысл, где может возникнуть, как лечить. Эта страница — спасение для новых команд интеграции.

И, наконец, накройте коды тестами: контрактные тесты, которые проверяют, что на конкретный вход API возвращает ожидаемый `errors[].code`. Такие тесты — страховка от случайных регрессов при рефакторинге.

**Gradle**
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

**Java — единый enum кодов и формирование ProblemDetail**

```java
package com.example.errors.codes;

import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

enum ApiErrorCode {
    VALIDATION_EMAIL_FORMAT_INVALID,
    VALIDATION_RANGE_FROM_AFTER_TO,
    CONFLICT_EMAIL_NOT_UNIQUE
}

@RestController
@RequestMapping("/api/codes")
class CodesController {

    @GetMapping("/demo")
    public ResponseEntity<?> demo() {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setProperty("code", ApiErrorCode.VALIDATION_EMAIL_FORMAT_INVALID.name());
        pd.setProperty("errors", java.util.List.of(
                Map.of("field", "email", "code", ApiErrorCode.VALIDATION_EMAIL_FORMAT_INVALID.name(), "message", "Invalid email")
        ));
        return ResponseEntity.unprocessableEntity().body(pd);
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.errors.codes

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

enum class ApiErrorCode {
    VALIDATION_EMAIL_FORMAT_INVALID,
    VALIDATION_RANGE_FROM_AFTER_TO,
    CONFLICT_EMAIL_NOT_UNIQUE
}

@RestController
@RequestMapping("/api/codes")
class CodesController {
    @GetMapping("/demo")
    fun demo(): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("code", ApiErrorCode.VALIDATION_EMAIL_FORMAT_INVALID.name)
            setProperty("errors", listOf(mapOf(
                "field" to "email",
                "code" to ApiErrorCode.VALIDATION_EMAIL_FORMAT_INVALID.name,
                "message" to "Invalid email"
            )))
        }
}
```

---

## Локализация сообщений, корреляция с traceId/requestId, пригодность для фронта и логики ретраев

Локализация — это про `message`, а не про `code`. Клиент присылает `Accept-Language`, сервер выбирает локаль и заполняет `detail`/`errors[].message` из `MessageSource`. При этом `errors[].code` остаётся прежним — клиенты сравнивают коды, а не тексты. Такой подход позволяет менять wording без релиза бэкенда.

`traceId`/`requestId` — обязательные поля для корреляции. Идеально, если они проходят через весь путь: от edge-прокси до БД. В Spring их удобно хранить в MDC или формировать через наблюдаемость (Micrometer Tracing). В ответах `ProblemDetail` кладём `traceId` отдельным свойством — так поддержка быстро найдёт событие в логах и трассах.

Локализация должна быть быстрой и безопасной. Не допускайте `MessageSource`-ошибок в проде: включите fallback на базовую локаль и мониторьте «неразрешённые ключи». Ошибочный ключ превращается в `{profile.name.notBlank}` в ответе — это портит UX.

Для ретраев клиентам нужны «машинные сигналы»: статус 503/504, заголовок `Retry-After`, коды `TRANSIENT_*`. Не используйте `422` для «попробуйте позже» — это вводит в заблуждение. Валидация — исправьте и повторите, отказ инфраструктуры — подождите и повторите.

Отдавайте `instance` в виде полного URL запроса — фронту это удобно и для диалога с поддержкой, и для «copy to clipboard». Но не добавляйте туда PII: если путь содержит чувствительные идентификаторы, лучше хранить их в `errors` как `field=value(masked)`.

При локализации избегайте «слияния» разных сообщений через строковую конкатенацию на сервере. Лучше отдавать коды и параметры (`{min}`, `{max}`, `{field}`) и позволить фронту собирать текст. Это гибче и снимает проблему разных родов/чисел языков.

Пригодность для фронта — это ещё и стабильность `errors[].field`: придерживайтесь соглашений имён (camelCase/snake_case) и не меняйте их без необходимости. Укажите в документации соответствие между JSON и полями DTO.

Наконец, добавьте в ответы безопасный `supportId` (тот же `traceId` или хэш от него). Это «человеческий» идентификатор инцидента, который пользователь может назвать в тикете, не раскрывая внутренних деталей.

**Gradle**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — локализация + traceId в ProblemDetail**

```java
package com.example.errors.i18ntrace;

import org.slf4j.MDC;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.Locale;
import java.util.Map;

@RestControllerAdvice
public class I18nTraceAdvice {
    private final MessageSource ms;
    public I18nTraceAdvice(MessageSource ms) { this.ms = ms; }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail invalid(MethodArgumentNotValidException ex) {
        Locale loc = LocaleContextHolder.getLocale();
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle(ms.getMessage("validation.title", null, "Validation failed", loc));
        pd.setDetail(ms.getMessage("validation.detail", null, "Payload contains invalid fields", loc));
        pd.setProperty("traceId", MDC.get("traceId"));
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> Map.of("field", fe.getField(),
                              "code", fe.getCode(),
                              "message", ms.getMessage(fe, loc)))
            .toList());
        return pd;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.errors.i18ntrace

import org.slf4j.MDC
import org.springframework.context.MessageSource
import org.springframework.context.i18n.LocaleContextHolder
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class I18nTraceAdvice(private val ms: MessageSource) {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun invalid(ex: MethodArgumentNotValidException): ProblemDetail {
        val loc = LocaleContextHolder.getLocale()
        return ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = ms.getMessage("validation.title", null, "Validation failed", loc)
            detail = ms.getMessage("validation.detail", null, "Payload contains invalid fields", loc)
            setProperty("traceId", MDC.get("traceId"))
            setProperty("errors", ex.bindingResult.fieldErrors.map {
                mapOf("field" to it.field, "code" to it.code, "message" to ms.getMessage(it, loc))
            })
        }
    }
}
```

---

# 9. Производительность, безопасность и эксплуатация

## Fail-fast vs сбор всех ошибок: баланс UX/нагрузки; лимиты на количество нарушений

Fail-fast заставляет валидатор останавливаться на первом нарушении. Это снижает CPU/латентность и уменьшает объём ответа — полезно для нагруженных внутренних API. Но UX страдает: пользователь исправляет по одному полю за раз. Сбор всех ошибок улучшает UX и снижает количество циклов клиента, но стоит дороже по вычислениям и размеру ответов.

Подход выбирается осознанно. Для публичных форм и интеграций лучше возвращать сразу все нарушения, чтобы клиент мог массово исправить данные. Для высокочастотных внутренних сервисов, где вход стабилен, fail-fast экономит ресурсы и упрощает трассировку.

Hibernate Validator поддерживает fail-fast флагом `hibernate.validator.fail_fast`. В Spring Boot его можно задать через `LocalValidatorFactoryBean`. Но будьте осторожны: это глобальная настройка. Если нужно поведение «на сценарий», создайте два бина валидатора или управляйте логикой вручную (обрубая сбор нарушений после N штук).

Лимит на количество нарушений — компромисс между UX и безопасностью. На «злонамеренном» запросе с тысячей ошибок вы не хотите формировать «роман» в ответе. Отсекайте список, например, первыми 50 нарушениями, добавляя флаг `truncated=true`. Это также сокращает время сериализации ответа.

Профилируйте: включение «сбор всех нарушений» на больших структурах может заметно увеличить время CPU. Микробенчмарки на локали и нагрузочное тестирование на стенде подскажут, насколько чувствителен ваш сервис к изменению политики.

Учитывайте логирование: при fail-fast логов меньше, а при большом списке нарушений лог-файлы могут разрастаться. Никогда не логируйте весь payload с PII; фиксируйте только диагностические коды, поле и укороченные значения.

Сценарии batch/импорта — отдельный случай. Там иногда полезно собрать **все** ошибки записи и приложить CSV-отчёт, но это делается офлайн: не в рамках одного HTTP-ответа, а в результирующем файле/почте.

И, наконец, зафиксируйте политику в документации проекта: «на HTTP-периметре собираем все ошибки, но не более 50; на внутренних шинах — fail-fast». Это снимет споры между командами и обеспечит стабильность.

**Gradle**
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-validation' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-validation") }
```

**Java — конфигурация fail-fast и усечение списка ошибок**

```java
package com.example.perf.failfast;

import org.springframework.context.annotation.Bean;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import org.springframework.http.*;

@RestControllerAdvice
class TruncatingAdvice {

    private static final int MAX_ERRORS = 50;

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onInvalid(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
                .limit(MAX_ERRORS)
                .map(fe -> java.util.Map.of("field", fe.getField(), "code", fe.getCode()))
                .toList();
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setProperty("errors", errors);
        pd.setProperty("truncated", ex.getBindingResult().getErrorCount() > MAX_ERRORS);
        return pd;
    }
}

@org.springframework.context.annotation.Configuration
class ValidationConfig {
    @Bean
    LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean v = new LocalValidatorFactoryBean();
        v.getValidationPropertyMap().put("hibernate.validator.fail_fast", "false"); // переключите на true для fail-fast
        return v;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.perf.failfast

import org.springframework.context.annotation.Bean
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class TruncatingAdvice {
    private val maxErrors = 50

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun onInvalid(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            val list = ex.bindingResult.fieldErrors.take(maxErrors)
                .map { mapOf("field" to it.field, "code" to it.code) }
            setProperty("errors", list)
            setProperty("truncated", ex.bindingResult.errorCount > maxErrors)
        }
}

@org.springframework.context.annotation.Configuration
class ValidationConfig {
    @Bean
    fun validator() = LocalValidatorFactoryBean().apply {
        validationPropertyMap["hibernate.validator.fail_fast"] = "false"
    }
}
```

---

## Дедуп и rate-limit на «шумные» ошибки; защита от regex-DoS, чёрные списки символов

Когда клиент «шумит» (много неверных запросов), серверу нужно защищать себя и остальной трафик. Простая техника — rate-limit на **ошибочные** ответы: если с одного ключа (IP, API-ключ, клиентский id) прилетает слишком много 4xx за короткое время, начинаем отвечать 429 или разрывать соединение. Это сдерживает ботов и баги.

Дедуп — это склейка одинаковых ошибок на уровне логов/метрик. Вместо тысячи записей «EMAIL_FORMAT_INVALID» за 10 секунд вы агрегируете их счётчиком, а в логи пишете кратко. Это не «валидация», но эксплуатационная гигиена, которая снижает шум и цену хранения.

Regex-DoS — класс атак, когда «плохая» регулярка на большом вводе зацикливает движок (catastrophic backtracking). Лечение: короткие и предсказуемые паттерны, предварительная отсечка по длине, **предкомпиляция** `Pattern`, использование «жадных без возврата» квантификаторов (`++`, `*+` в Java) и запрет «универсальных» конструкций. При возможности — альтернативный движок (re2j), но чаще достаточно дисциплины.

Чёрные списки символов полезны для некоторых полей (имя файла, логин): отсекаем управляющие, невидимые, RTL-override и опасные суррогаты. Но помните, что whitelist-подход обычно надёжнее: чётко разрешённый алфавит с ограничениями длины.

Технически проверку «чёрного списка» лучше делать **до** дорогостоящих операций: в фильтре или десериализации. Возвращайте 422 с кодом `FORBIDDEN_CHARACTERS` и подсказкой «разрешены латиница/цифры/символы …».

Не забывайте о полях, где символы критичны: пароли, подписи, бинарные ключи. К ним чёрные списки неприменимы — только длина и базовая валидация формата.

Для rate-limit практичен «token bucket» в памяти/Redis. Ошибок много? Сбрасываем ведро быстрее. В проде часто внедряют это на edge (API Gateway), но прикладной уровень тоже может подсобить локальной защитой.

И ещё: валидация должна быстро «проваливать» заведомо плохие входы. Проверка длины и символов — первая линия. Только потом — regex и сложные правила. Это экономит CPU.

**Gradle**
*Groovy DSL*

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-web' }
```

*Kotlin DSL*

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-web") }
```

**Java — безопасный Pattern + простой rate-limit по ошибкам**

```java
package com.example.perf.safety;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.regex.Pattern;

@Component
public class SafetyFilter extends GenericFilter {

    private static final Pattern SAFE_EMAIL = Pattern.compile("^[A-Za-z0-9._%+-]{1,64}@[A-Za-z0-9.-]{1,253}\\.[A-Za-z]{2,63}$");
    private static final int WINDOW_SEC = 10;
    private static final int MAX_4XX = 50;

    private final Map<String, Window> errors = new ConcurrentHashMap<>();

    @Override
    public void doFilter(ServletRequest r, ServletResponse s, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) r;
        HttpServletResponse res = (HttpServletResponse) s;

        // Пример чёрного списка для заголовка
        String email = req.getHeader("X-User-Email");
        if (email != null && (email.length() > 320 || !SAFE_EMAIL.matcher(email).matches())) {
            res.setStatus(HttpStatus.UNPROCESSABLE_ENTITY.value());
            res.setContentType("application/problem+json");
            res.getWriter().write("{\"title\":\"Validation failed\",\"status\":422,\"detail\":\"Invalid email\"}");
            register4xx(req);
            return;
        }

        chain.doFilter(r, s);
        if (res.getStatus() >= 400 && res.getStatus() < 500) register4xx(req);
        if (tooNoisy(req)) {
            res.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
        }
    }

    private void register4xx(HttpServletRequest req) {
        String key = req.getRemoteAddr();
        errors.compute(key, (k, w) -> (w == null || w.isExpired()) ? new Window() : w.bump());
    }

    private boolean tooNoisy(HttpServletRequest req) {
        Window w = errors.get(req.getRemoteAddr());
        return w != null && !w.isExpired() && w.count > MAX_4XX;
    }

    private static class Window {
        final long started = Instant.now().getEpochSecond();
        int count = 1;
        boolean isExpired() { return Instant.now().getEpochSecond() - started > WINDOW_SEC; }
        Window bump() { this.count++; return this; }
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.perf.safety

import jakarta.servlet.*
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Component
import java.time.Instant
import java.util.concurrent.ConcurrentHashMap

@Component
class SafetyFilter : GenericFilter() {

    private val safeEmail = Regex("^[A-Za-z0-9._%+-]{1,64}@[A-Za-z0-9.-]{1,253}\\.[A-Za-z]{2,63}$")
    private val windowSec = 10
    private val max4xx = 50
    private val errors = ConcurrentHashMap<String, Window>()

    override fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
        val req = request as HttpServletRequest
        val res = response as HttpServletResponse

        val email = req.getHeader("X-User-Email")
        if (email != null && (email.length > 320 || !safeEmail.matches(email))) {
            res.status = HttpStatus.UNPROCESSABLE_ENTITY.value()
            res.contentType = "application/problem+json"
            res.writer.write("""{"title":"Validation failed","status":422,"detail":"Invalid email"}""")
            register4xx(req); return
        }

        chain.doFilter(request, response)
        if (res.status in 400..499) register4xx(req)
        if (tooNoisy(req)) res.status = HttpStatus.TOO_MANY_REQUESTS.value()
    }

    private fun register4xx(req: HttpServletRequest) {
        val key = req.remoteAddr
        errors.compute(key) { _, w -> if (w == null || w.isExpired()) Window() else w.bump() }
    }

    private fun tooNoisy(req: HttpServletRequest): Boolean {
        val w = errors[req.remoteAddr] ?: return false
        return !w.isExpired() && w.count > max4xx
    }

    private data class Window(val started: Long = Instant.now().epochSecond, var count: Int = 1) {
        fun isExpired() = Instant.now().epochSecond - started > 10
        fun bump(): Window { count++; return this }
    }
}
```

---

## БД-constraint’ы как последняя линия: уникальные индексы, NOT NULL, check-ограничения

Даже при отличной валидации в приложении данные попадают в БД через разные каналы: другие сервисы, миграции, консольные скрипты. Единственная надёжная защита от логической порчи — ограничения на уровне БД. Уникальные индексы предотвращают дубликаты, `NOT NULL` не пустит «дыры», `CHECK` фиксирует критичные инварианты, FK держат ссылочную целостность.

Приложение должно уважать эти границы и правильно переводить ошибки БД в HTTP. `DataIntegrityViolationException` → `409 Conflict` для уникальности, `400/422` для `CHECK`, `404/409` для FK (зависит от сценария). В ответе — машинный код (`CONFLICT_EMAIL_NOT_UNIQUE`) и дружелюбный `detail`. Так клиенты понимают, что произошло, и не штурмуют саппорт.

Производительность: уникальные индексы стоят места и времени на запись. Но попытка «сэкономить» удалением индекса дороже — вы потеряете целостность. Дизайн индексов — часть доменного моделирования, их стоит ревьюить вместе с кодом.

Надёжность: проверка «existsBy…» до сохранения подвержена гонкам. Это подсказка для UX, но не гарантия. Единственная гарантия уникальности — индекс. В коде ловим исключение и мапим на 409. Такое дублирование — норма.

Миграции: используйте Liquibase/Flyway, чтобы держать ограничения в VCS. Это делает схему воспроизводимой и избавляет от «дрейфа» между стендами. Пишите изменения идемпотентно и тестируйте накатывание/откат.

Трассировка: текст ошибок БД не должен утекать клиенту (там могут быть имена колонок/таблиц). Скрывайте детали и отдавайте дружелюбный код/текст. В логи пишите первопричину и SQLSTATE.

И не забывайте про CHECK. Часто люди игнорируют его, а это мощный инструмент: «amount > 0», «status in (…)». Он работает быстро и дешёв, а польза огромная.

**SQL (PostgreSQL) — ограничения**

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  age INT NOT NULL CHECK (age >= 0)
);

CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  amount NUMERIC(18,2) NOT NULL CHECK (amount > 0)
);
```

**Java — маппинг нарушений на 409/422**

```java
package com.example.db.guard;

import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
class DbConstraintAdvice {

    @ExceptionHandler(DataIntegrityViolationException.class)
    ProblemDetail onDiv(DataIntegrityViolationException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        pd.setTitle("Conflict");
        pd.setDetail("Unique or FK constraint violated");
        pd.setProperty("code", "CONFLICT_DB_CONSTRAINT");
        return pd;
    }
}
```

**Kotlin — аналог**

```kotlin
package com.example.db.guard

import org.springframework.dao.DataIntegrityViolationException
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class DbConstraintAdvice {
    @ExceptionHandler(DataIntegrityViolationException::class)
    fun onDiv(@Suppress("UNUSED_PARAMETER") ex: DataIntegrityViolationException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.CONFLICT).apply {
            title = "Conflict"
            detail = "Unique or FK constraint violated"
            setProperty("code", "CONFLICT_DB_CONSTRAINT")
        }
}
```

---

## Частичные обновления (PATCH): валидация «только присланных» полей, мердж с текущим состоянием

PATCH-операции отличаются от PUT/POST: клиент присылает **только изменяемые поля**. Валидация здесь должна учитывать «не прислано» как «оставить как есть». Простая техника — отдельный DTO с nullable-полями и без `@NotBlank/@NotNull` (обязательность не нужна), а затем: (1) подтягиваем текущее состояние, (2) применяем изменения, (3) валидируем **готовую** модель целиком.

Альтернатива — JSON Merge Patch / JSON Patch. Для Merge Patch сервер получает document с частичными значениями, накладывает его на текущий JSON сущности и валидирует результат. Это снимает вопросы об «обязательности» на уровне DTO и делает поведение явно определённым стандартом.

Валидация «только присланных» полей безопасна, но требует дисциплины: правила «межполевые» всё равно должны проверяться на итоговой модели, иначе можно создать неконсистентное состояние (например, поменять `from`, не проверив `from<to`).

Ошибки PATCH’a часто завязаны на состояние (версии, блокировки). Поддерживайте ETag/If-Match и мапьте `412 Precondition Failed` при конфликте версий. Это отдельная от валидации история, но она рядом и влияет на UX.

В ответах на PATCH удобно отдавать `ProblemDetail` с `errors[].field` на «патчевые» пути (JSON Pointer), чтобы фронт понимал, какое именно обновление не принято. Для Merge Patch поле может быть `/email`, для JSON Patch — index операции.

С точки зрения безопасности не позволяйте менять «запрещённые» поля (имя владельца, статус независимо от автомата). Это тоже проверяется на шаге мерджа: игнорируйте или возвращайте `403/422`.

Тестируйте PATCH отдельно: «только email», «только адрес», «несогласованные поля», «пустой патч». И регрессно — что PUT (полная замена) по-прежнему валидируется как «всё обязательное».

Если у вас Kotlin, удобно использовать копирование `copy(...)` и именованные параметры: применяем патч точечно и валидируем итоговый `data class`.

**Gradle**
*Groovy DSL*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
}
```

*Kotlin DSL*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("com.fasterxml.jackson.core:jackson-databind")
}
```

**Java — PATCH через JSON Merge Patch и последующая валидация**

```java
package com.example.patch.merge;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import jakarta.validation.Valid;
import jakarta.validation.Validator;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UsersPatchController {
    private final ObjectMapper om;
    private final Validator validator;

    public UsersPatchController(ObjectMapper om, Validator validator) {
        this.om = om; this.validator = validator;
    }

    @PatchMapping("/{id}")
    public ResponseEntity<?> patch(@PathVariable long id, @RequestBody ObjectNode mergePatch) {
        // 1) загрузить текущее состояние (заглушка)
        User current = new User("Alice", "alice@example.com");
        // 2) наложить merge patch
        ObjectNode currentJson = om.valueToTree(current);
        currentJson.setAll(mergePatch);
        User updated = om.convertValue(currentJson, User.class);
        // 3) валидировать итоговую модель
        var violations = validator.validate(updated);
        if (!violations.isEmpty()) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setProperty("errors", violations.stream().map(v ->
                    java.util.Map.of("field", v.getPropertyPath().toString(), "message", v.getMessage()))
                .toList());
            return ResponseEntity.unprocessableEntity().body(pd);
        }
        return ResponseEntity.ok(updated);
    }

    public record User(@NotBlank String name, @Email String email) {}
}
```

**Kotlin — аналог с data class и copy**

```kotlin
package com.example.patch.merge

import com.fasterxml.jackson.databind.JsonNode
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import jakarta.validation.Validator
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UsersPatchController(private val validator: Validator) {

    private val om = jacksonObjectMapper()

    @PatchMapping("/{id}")
    fun patch(@PathVariable id: Long, @RequestBody mergePatch: JsonNode): ResponseEntity<Any> {
        val current = User(name = "Alice", email = "alice@example.com")
        val currentJson = om.valueToTree<JsonNode>(current) as com.fasterxml.jackson.databind.node.ObjectNode
        currentJson.setAll<ObjectNode>(mergePatch as com.fasterxml.jackson.databind.node.ObjectNode)
        val updated = om.treeToValue(currentJson, User::class.java)
        val violations = validator.validate(updated)
        return if (violations.isNotEmpty()) {
            val pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"
                setProperty("errors", violations.map { mapOf("field" to it.propertyPath.toString(), "message" to it.message) })
            }
            ResponseEntity.unprocessableEntity().body(pd)
        } else ResponseEntity.ok(updated)
    }
}

data class User(
    @field:NotBlank val name: String,
    @field:Email val email: String
)
```

# 10. Тестирование и инструменты

## Unit-тесты валидаторов и групп; параметризованные негативные кейсы

Юнит-тесты валидаторов — дешёвый и самый надёжный способ зафиксировать поведение правил на годы. Валидация — это чистая функция «вход → набор нарушений», поэтому идеально подходит для параметризованных тестов: вы подаёте десятки граничных примеров и получаете воспроизводимый результат. Особенно важно покрыть негативные сценарии: пустые строки, пробелы, максимальные длины +1, экзотические символы, «почти валидные» форматы.

Структура таких тестов проста: инициализируете `jakarta.validation.Validator` (через `Validation.buildDefaultValidatorFactory()` или `LocalValidatorFactoryBean`), готовите DTO с аннотациями и валидируете. Для кастомных constraint’ов (например, `@StrongPassword`) тестируете как сам `ConstraintValidator`, так и его «встроенность» в DTO: первый уровень проверяет алгоритм, второй — интеграцию с фреймворком (правильные таргеты, работа с `null`, группы).

Группы — отдельная тема. Если вы применяете сценарии `Create`/`Update`, тесты должны явно активировать нужную группу и проверять, что правила «включаются» и «выключаются» корректно. Это помогает избежать хрупкости, когда, например, правило по ошибке срабатывает и на create, и на update.

Параметризованные тесты в JUnit 5 позволяют компактно выразить десятки кейсов. Вместо копипасты методов вы даёте таблицу входов и ожиданий: так легче читать и поддерживать. В Kotlin при использовании JUnit 5 сохраняется тот же подход; если хочется ещё более декларативно — можно подключить Kotest/Strikt, но базовый JUnit + AssertJ вполне достаточен.

Обратите внимание на семантику `null`. По умолчанию большинство валидаторов считают `null` валидным значением (обязательность — забота `@NotNull/@NotBlank`). Тестами зафиксируйте, что ваш кастомный валидатор не «делает» обязательность сам по себе, иначе вы получите неожиданные двойные сообщения об ошибках.

Для валидаторов, зависящих от внешних бинов (репозитории, кэши), допускается мокать зависимости и тестировать через Spring-контекст. Но лучше держать их как можно «чище»: правило — быстрое и детерминированное, без IO. В противном случае юнит-тест превратится в интеграционный и станет хрупким.

Границы значений — скучная, но критически важная часть покрытия. Если есть `@Size(max=100)`, протестируйте 99/100/101. Если `@Pattern`, протестируйте минимальные, максимальные и «патологические» строки (длинные, однородные, с нестандартными Unicode). Это защищает от регрессий при смене библиотек и JVM.

Для читаемости ошибок в тестах удобно сравнивать **коды правил** (`violation.getConstraintDescriptor().getAnnotation().annotationType().getSimpleName()` или `code` из Spring-ошибок), а не тексты. Тексты локализуются и частично зависят от bundles, а коды стабильны.

*Gradle (тесты валидаторов)*
**Groovy DSL**

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — параметризованные тесты для `@StrongPassword` и групп**

```java
package com.example.validation.tests;

import jakarta.validation.Validation;
import jakarta.validation.Validator;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.groups.Default;
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

// Группы сценариев
interface Create {}
interface Update {}

class StrongPasswordUnitTest {

    record Signup(
            @NotBlank(groups = Create.class) String name,
            @Email(groups = {Create.class, Update.class}) String email,
            @StrongPassword(min = 8) String password // обязательность задаёт @NotBlank при необходимости
    ) {}

    private static Validator validator;

    @BeforeAll
    static void init() {
        validator = Validation.buildDefaultValidatorFactory().getValidator();
    }

    @ParameterizedTest
    @ValueSource(strings = {"short", "Noupper1!", "NOLOWER1!", "NoDigit!", "NoSpec1"})
    void weak_passwords_fail(String pwd) {
        var v = validator.validate(new Signup("A", "a@b.c", pwd), Default.class, Create.class);
        assertThat(v).anyMatch(cv -> cv.getMessage().toLowerCase().contains("password"));
    }

    @Test
    void groups_create_requires_name_email() {
        var v = validator.validate(new Signup(null, "bad", "Aa1!aaaa"), Create.class);
        assertThat(v).extracting(cv -> cv.getConstraintDescriptor().getAnnotation().annotationType().getSimpleName())
                .contains("NotBlank", "Email");
    }

    @Test
    void groups_update_allows_missing_name_but_checks_email() {
        var v = validator.validate(new Signup(null, "bad", null), Update.class);
        assertThat(v).extracting(cv -> cv.getConstraintDescriptor().getAnnotation().annotationType().getSimpleName())
                .contains("Email").doesNotContain("NotBlank");
    }
}
```

**Kotlin — параметризованные тесты JUnit 5**

```kotlin
package com.example.validation.tests

import jakarta.validation.Validation
import jakarta.validation.Validator
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource

annotation class Create
annotation class Update

class StrongPasswordUnitTest {

    data class Signup(
        @field:NotBlank(groups = [Create::class]) val name: String?,
        @field:Email(groups = [Create::class, Update::class]) val email: String?,
        @field:StrongPassword(min = 8) val password: String?
    )

    companion object {
        lateinit var validator: Validator
        @JvmStatic @BeforeAll fun init() {
            validator = Validation.buildDefaultValidatorFactory().validator
        }
    }

    @ParameterizedTest
    @ValueSource(strings = ["short", "Noupper1!", "NOLOWER1!", "NoDigit!", "NoSpec1"])
    fun `weak passwords fail`(pwd: String) {
        val v = validator.validate(Signup("A", "a@b.c", pwd))
        assertThat(v.any { it.message.contains("Password", ignoreCase = true) }).isTrue()
    }

    @Test
    fun `create requires name and email`() {
        val v = validator.validate(Signup(null, "bad", "Aa1!aaaa"), Create::class.java)
        val codes = v.map { it.constraintDescriptor.annotation.annotationClass.simpleName }
        assertThat(codes).contains("NotBlank", "Email")
    }

    @Test
    fun `update allows missing name but checks email`() {
        val v = validator.validate(Signup(null, "bad", null), Update::class.java)
        val codes = v.map { it.constraintDescriptor.annotation.annotationClass.simpleName }
        assertThat(codes).contains("Email").doesNotContain("NotBlank")
    }
}
```

---

## Slice-тесты контроллеров (MockMvc/WebTestClient): 400/422, структура ProblemDetail, локализация

Slice-тесты поднимают тонкий срез веб-слоя без всей инфраструктуры: для MVC — `@WebMvcTest`, для WebFlux — `@WebFluxTest`. Это быстрые проверки «контроллер ↔ сериализация ↔ валидация ↔ advice». Мы убеждаемся, что статусы `400/422` выставляются корректно, `ProblemDetail` содержит нужные поля, а локализация подтягивается из `MessageSource`.

Такие тесты хороши для фиксации контракта на уровне HTTP: JSON-форма ошибки (`application/problem+json`), наличие массива `errors`, правильная привязка `field` со вложенными путями (`items[2].sku`). Также мы проверяем, что «сломанный JSON» превращается в `400`, а валидный, но некорректный — в `422`. Это линии обороны, которые чаще всего ломаются при рефакторингах.

С локализацией важно протестировать как минимум две локали. Для этого в тесте можно передать `Accept-Language` и проверить, что `title/detail` меняются на ожидаемые, а `errors[].code` остаются прежними. Так вы не привяжете клиентов к конкретным текстам.

MockMvc и WebTestClient позволяют проверять JSON через `jsonPath`: вы легко ассертите статус, медиатип и значения полей. Для `ProblemDetail` в Spring Boot 3 это обычный JSON-объект, поэтому проверка прозрачна.

Тесты должны покрывать и happy-path, иначе можно написать advice, который «ломает» успешные ответы (например, фильтр крутит заголовки). Но основной фокус — на ошибках: статусы, тело, локаль, коды, instance/type.

Если ваш контроллер использует `BindingResult`, убедитесь, что локальный ответ не расходится по форме с глобальным advice. Slice-тест — удобное место, чтобы поймать рассинхронизацию.

Стабильность таких тестов зависит от жёсткости матчинга. Не сравнивайте целиком JSON «строкой» — используйте точечные `jsonPath` проверки. Это снизит ложные срабатывания при добавлении новых «безопасных» полей.

*Gradle (slice-тесты)*
**Groovy DSL**

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**Java — `@WebMvcTest` + локализация и проверка 400/422**

```java
package com.example.slices.mvc;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = MvcSlice.Controller.class)
class MvcSlice {

    @Autowired MockMvc mvc;

    @RestController
    static class Controller {
        @PostMapping("/mvc/profiles")
        public Profile create(@RequestBody Profile p) { return p; }
    }

    record Profile(@NotBlank(message="{profile.name}") String name, @Email(message="{profile.email}") String email) {}

    @org.springframework.context.annotation.Configuration
    static class Cfg {
        @Bean ResourceBundleMessageSource messageSource() {
            var ms = new ResourceBundleMessageSource();
            ms.setBasenames("messages");
            ms.setDefaultEncoding("UTF-8");
            return ms;
        }
    }

    @Test
    void returns_422_with_localized_messages() throws Exception {
        String json = "{\"name\":\" \",\"email\":\"bad\"}";
        mvc.perform(post("/mvc/profiles")
                .contentType(MediaType.APPLICATION_JSON).content(json)
                .header("Accept-Language", "ru"))
           .andExpect(status().isUnprocessableEntity())
           .andExpect(content().contentType("application/problem+json"))
           .andExpect(jsonPath("$.title").exists())
           .andExpect(jsonPath("$.errors").isArray())
           .andExpect(jsonPath("$.errors[?(@.field=='name')]").exists());
    }

    @Test
    void malformed_json_is_400() throws Exception {
        String json = "{\"name\":";
        mvc.perform(post("/mvc/profiles").contentType(MediaType.APPLICATION_JSON).content(json))
           .andExpect(status().isBadRequest());
    }
}
```

**Kotlin — `@WebFluxTest` + WebTestClient**

```kotlin
package com.example.slices.webflux

import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest
import org.springframework.context.annotation.Bean
import org.springframework.context.support.ResourceBundleMessageSource
import org.springframework.http.MediaType
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.*

@WebFluxTest(controllers = [FluxSlice.Controller::class])
class FluxSlice {

    @Autowired lateinit var client: WebTestClient

    @RestController
    class Controller {
        @PostMapping("/fx/profiles")
        fun create(@RequestBody p: Profile) = p
    }

    data class Profile(
        @field:NotBlank(message = "{profile.name}") val name: String?,
        @field:Email(message = "{profile.email}") val email: String?
    )

    @org.springframework.context.annotation.Configuration
    class Cfg {
        @Bean
        fun messageSource() = ResourceBundleMessageSource().apply {
            setBasenames("messages"); setDefaultEncoding("UTF-8")
        }
    }

    @Test
    fun `422 with errors array`() {
        val json = """{"name":" ","email":"bad"}"""
        client.post().uri("/fx/profiles").contentType(MediaType.APPLICATION_JSON)
            .header("Accept-Language", "ru")
            .bodyValue(json)
            .exchange()
            .expectStatus().isEqualTo(422)
            .expectHeader().contentType("application/problem+json")
            .expectBody()
            .jsonPath("$.errors").isArray
            .jsonPath("$.errors[0].field").exists()
    }

    @Test
    fun `400 on malformed json`() {
        client.post().uri("/fx/profiles").contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"name":""")
            .exchange()
            .expectStatus().isBadRequest
    }
}
```

---

## Контракт-тесты/генеративные проверки (schemathesis/JSON Schema) и синхронизация с OpenAPI

Контракт-тесты проверяют соответствие реального API заявленному контракту (OpenAPI). Это страховка от «случайных» ломаний форматов, полей и статусов. В бэкенде удобно держать минимальную `openapi.yaml`, собранную springdoc’ом или вручную, и валидировать ответы и ошибки в тестах. Для ошибок можно выделить JSON Schema на `ProblemDetail` и проверять его для всех 4xx/5xx.

Генеративные проверки идут дальше: на основе схемы инструмент (например, Schemathesis) генерирует множество входов, включая граничные и сломанные, и прогоняет по вашему API. Это ловит неожиданные провалы парсинга/валидации. В CI такие тесты можно запускать «легким» профилем, а на nightly — в расширенном варианте.

Синхронизация с OpenAPI жизненно важна: документация — это часть контракта. Любое изменение DTO/валидации должно попадать в контракт, иначе клиенты начнут отправлять «устаревшие» поля и получать 422. Автоматизация через springdoc уменьшает ручной труд, но всё равно требует ревью: описать коды ошибок, `application/problem+json` и структуры `errors[]`.

На JVM есть несколько библиотек для проверки соответствия: openapi4j (валидация запросов/ответов относительно OpenAPI) и JSON Schema валидаторы (например, networknt). Подход практичный: в тесте вызываете контроллер (MockMvc/WebTestClient), захватываете ответ и валидируете против схемы. Для отдельных эндпоинтов с нестандартными ответами можно писать небольшие кастомные матчеры.

Особое внимание уделите тем, кто «ломает» контракт чаще всего: поля с enum, статусы (`400/422/409`), форматы дат, маскирования PII в ошибках. На эти места заведите отдельные контрактные тесты — это сэкономит часы интеграционного администрирования.

Если вы используете JSON Schema для входов (например, в Kafka событиях), те же схемы применимы для REST. Это уменьшает количество источников правды и снижает риск рассинхронизации между «вебом» и «шиной».

И наконец, договоритесь о политике: любой PR, меняющий DTO/валидацию, обязан обновить OpenAPI/Schema и тесты. CI должен «краснеть», если контракт не синхронизирован. Это дисциплинирует и повышает доверие к документации.

*Gradle (валидация JSON Schema/OpenAPI)*
**Groovy DSL**

```groovy
dependencies {
    testImplementation 'com.networknt:json-schema-validator:1.0.86'
    testImplementation 'org.openapi4j:openapi-operation-validator:1.0.10'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    testImplementation("com.networknt:json-schema-validator:1.0.86")
    testImplementation("org.openapi4j:openapi-operation-validator:1.0.10")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — проверка `ProblemDetail` против JSON Schema**

```java
package com.example.contract.schema;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.networknt.schema.*;
import org.junit.jupiter.api.Test;

import java.io.InputStream;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class ProblemSchemaTest {

    @Test
    void problem_detail_matches_schema() throws Exception {
        ObjectMapper om = new ObjectMapper();
        String problem = """
            {
              "type":"https://docs.example.com/problems/validation-error",
              "title":"Validation failed",
              "status":422,
              "detail":"Payload contains invalid fields",
              "errors":[{"field":"email","code":"VALIDATION_EMAIL_FORMAT_INVALID","message":"Invalid email"}]
            }""";
        try (InputStream schemaIs = getClass().getResourceAsStream("/schemas/problem.schema.json")) {
            JsonSchemaFactory factory = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V202012);
            JsonSchema schema = factory.getSchema(schemaIs);
            Set<ValidationMessage> errors = schema.validate(om.readTree(problem));
            assertThat(errors).isEmpty();
        }
    }
}
```

**Kotlin — валидация ответа контроллера против OpenAPI (openapi4j)**

```kotlin
package com.example.contract.openapi

import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.openapi4j.core.model.OAIContext
import org.openapi4j.operation.validator.adapters.server.servlet.OpenApiInteractionValidator
import org.openapi4j.operation.validator.model.Request
import org.openapi4j.operation.validator.model.impl.Body
import org.openapi4j.operation.validator.model.impl.DefaultRequest
import org.openapi4j.operation.validator.model.impl.DefaultResponse
import java.nio.charset.StandardCharsets
import kotlin.test.assertTrue

class OpenApiContractTest {

    companion object {
        lateinit var validator: OpenApiInteractionValidator

        @JvmStatic @BeforeAll
        fun init() {
            validator = OpenApiInteractionValidator.createFor("/openapi.yaml").build()
        }
    }

    @Test
    fun `response conforms to contract`() {
        val req: Request = DefaultRequest.Builder("POST", "/api/profiles")
            .withContentType("application/json")
            .withBody(Body.from("{\"name\":\"\",\"email\":\"bad\"}", "application/json"))
            .build()

        val res = DefaultResponse.Builder(422)
            .withContentType("application/problem+json")
            .withBody("""{"title":"Validation failed","status":422,"errors":[{"field":"email"}]}"""
                .toByteArray(StandardCharsets.UTF_8))
            .build()

        val report = validator.validate(req, res)
        assertTrue(report.messages.isEmpty(), "Contract violations: ${report.messages}")
    }
}
```

*Пример `schemas/problem.schema.json` (фрагмент):*

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["title", "status"],
  "properties": {
    "type": { "type": "string", "format": "uri" },
    "title": { "type": "string" },
    "status": { "type": "integer", "minimum": 100, "maximum": 599 },
    "detail": { "type": "string" },
    "errors": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["field"],
        "properties": {
          "field": { "type": "string" },
          "code": { "type": "string" },
          "message": { "type": "string" }
        }
      }
    }
  }
}
```

---

## Метрики и наблюдаемость: доля отклонённых запросов, топ-правил, алёрты на всплески валидационных ошибок

Валидация — часть пользовательского пути, и её надо измерять. Базовые метрики: **доля отклонённых запросов** (422/все), **распределение по кодам правил** (какие чаще всего срабатывают), **латентность валидации** и **всплески**. Эти данные помогают отделить баги клиентов от регрессий сервера и влияют на приоритезацию UX-улучшений.

Micrometer делает сбор метрик простым: вы инжектите `MeterRegistry` и инкрементируете счётчики в централизованном обработчике ошибок. Хорошая гранулярность — теги `endpoint`, `ruleCode`, `status`. Для WebFlux аналогично, только обновление счётчиков происходят в реактивном хендлере/фильтре. Важно: не добавляйте PII в теги; теги — это имя правила и статусы.

Также полезны **лог-метрики**: если у вас «тонкий» фильтр, который режет «большие запросы», инкрементируйте отдельный счётчик `validation.rejected.largePayload`. Это позволит быстро диагностировать конфигурационные ошибки прокси или фронта. Для временных проблем внешних справочников заведите `validation.degraded.externalCatalog`.

Алёрты: простейшее правило — «доля 422 за 5 минут больше X%» и «скачок конкретного кода (например, EMAIL_FORMAT_INVALID) в N раз». Это укажет на баг в фронте/мобиле или на сломанный релиз клиента. Отдельный алерт — резкий рост `409 Conflict` для уникальности — часто сигнал гонок или массового ретрая.

Графики «топ-правил» (bar chart по ruleCode) раскрывают скрытые проблемы UX: пользователи системно ошибаются в одном поле — значит, нужен автокомплит/маска, а не жёсткое правило. Эти дашборды — предмет совместной работы бэкенда и фронта.

Наблюдаемость — это и трассировка. Добавляйте `traceId` в ProblemDetail и логи; связывайте ошибки валидации с входными запросами. В Micrometer Tracing можно повесить «span» вокруг блока нормализация/валидация и замерять латентность отдельно от домена — это поможет понять, где вы тратите CPU.

Не забывайте про **rate-limit** на «шумящие» 4xx (см. предыдущую главу): количество срезанных запросов — тоже метрика. Если она растёт — ваш периметр работает как WAF, и его правила требуют пересмотра/коммуникации с клиентами.

Наконец, метрики должны жить в CI/CD как «SLO»: если доля 422 внезапно подскочила на канареечной версии — автоматом останавливаем раскатку. Это дёшево реализовать и экономит много нервов на релизах.

*Gradle (Actuator + Micrometer Prometheus)*
**Groovy DSL**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
}
```

**Kotlin DSL**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — счётчики ошибок в `@RestControllerAdvice`**

```java
package com.example.metrics.validation;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class ValidationMetricsAdvice {

    private final MeterRegistry registry;

    public ValidationMetricsAdvice(MeterRegistry registry) { this.registry = registry; }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onInvalid(MethodArgumentNotValidException ex) {
        registry.counter("validation.rejected", "status", "422").increment();
        ex.getBindingResult().getFieldErrors().forEach(fe ->
            registry.counter("validation.rule.hit", "rule", fe.getCode(), "field", fe.getField()).increment()
        );
        var pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        return pd;
    }

    @ExceptionHandler(org.springframework.http.converter.HttpMessageNotReadableException.class)
    public ProblemDetail onBad(HttpMessageNotReadableException ex) {
        registry.counter("validation.rejected", "status", "400").increment();
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Malformed request");
        pd.setDetail(ex.getMostSpecificCause().getMessage());
        return pd;
    }
}
```

**Kotlin — фильтр для метрик «большой запрос» и конфигурация Actuator**

```kotlin
package com.example.metrics.validation

import io.micrometer.core.instrument.MeterRegistry
import jakarta.servlet.*
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component

@Component
@Order(10)
class LargePayloadMetricFilter(private val registry: MeterRegistry) : Filter {
    override fun doFilter(request: ServletRequest, response: ServletResponse, chain: FilterChain) {
        val req = request as HttpServletRequest
        val res = response as HttpServletResponse
        chain.doFilter(request, response)
        if (res.status == 413) {
            registry.counter("validation.rejected.largePayload", "path", req.requestURI).increment()
        }
    }
}
```

*Конфигурация Actuator/Prometheus (`application.yml`):*

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus,metrics"
  metrics:
    tags:
      application: validation-service
```

*Алерт Prometheus (пример):*

```yaml
groups:
- name: validation.rules
  rules:
  - alert: ValidationErrorRateHigh
    expr: sum(rate(validation_rejected_total{status="422"}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) > 0.2
    for: 5m
    labels: { severity: warning }
    annotations:
      summary: "Высокая доля 422"
      description: "Доля 422 > 20% за 5 минут"
```





