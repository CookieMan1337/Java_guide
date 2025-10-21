---
layout: page
title: "Форматы данных и обработка ошибок"
permalink: /spring/data_format_exception_handling
---

# 0. Введение

## Что это

Сериализация — это преобразование объекта (DTO, доменной модели, события) в последовательность байт или текст (JSON, XML, Protobuf), а десериализация — обратный процесс. В вебе этим занимаются **HttpMessageConverters** (в Spring MVC/WebFlux), а в «ручном» режиме — конкретные библиотеки (чаще всего **Jackson** для JSON). От того, **как** и **во что** вы сериализуете, зависят совместимость, скорость, размер полезной нагрузки и устойчивость к ошибкам.

В контексте Spring Boot по умолчанию используется JSON через Jackson. Это означает, что любой объект, который возвращает ваш контроллер, будет автоматически превращён в JSON с учётом настроек `ObjectMapper`. Аналогично входящие JSON-запросы вытягиваются в Java/Kotlin-классы, если сигнатура метода содержит `@RequestBody`.

Формат данных — это ещё и «контракт»: набор полей, их типы, правила именования, требования к датам/числам, политика null. Для текстовых форматов контракт описывают документацией + JSON Schema; для бинарных (Avro, Protobuf) — отдельной схемой, из которой часто генерируют код. Контракт должен эволюционировать **безопасно**, иначе клиенты ломаются при каждом релизе.

Сериализация встречается не только в HTTP. Это и Kafka/IBM MQ сообщения, и файлы на S3/MinIO, и JSONB-колонки в Postgres, и конфиги (YAML/Properties), и обмен в gRPC (Protobuf). Везде важны одни и те же вопросы: стабильность схемы, точность чисел (особенно денег), формат дат, локализация и производительность.

Важна осознанность в выборе: JSON — идеален для публичных REST и отладки, Protobuf/Avro — для высоконагруженных внутренних RPC/стриминга, Parquet — для аналитики. В реальном проекте часто сосуществуют **несколько** форматов, и это нормально: выбирайте под задачу.

Ещё одна грань — безопасность: сериализатор — часть «границы доверия». Он должен корректно обрабатывать неожиданные поля, запрещать десериализацию опасных типов, не «проглатывать» ошибки молча и не открывать гадостей вроде polymorphic deserialization без whitelisting.

Наконец, сериализация — заметная статья расходов CPU/GC. Плохие DTO (избыточные поля, глубокая вложенность), неоптимальные настройки Jackson или «тяжёлые» форматы там, где не нужны, — всё это конвертируется в миллисекунды задержек и лишние мегабайты трафика.

**Gradle (общая база для примеров JSON/Jackson).**
*Groovy DSL:*

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.5'
}
group = 'com.example'
version = '1.0.0'
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
repositories { mavenCentral() }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
    implementation 'com.fasterxml.jackson.module:jackson-module-kotlin'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

*Kotlin DSL:*

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.5"
}
group = "com.example"
version = "1.0.0"
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
repositories { mavenCentral() }
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.core:jackson-databind")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Мини-пример сериализации.**
*Java (REST + Jackson):*

```java
package com.example.intro;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.Instant;

@RestController
public class IntroController {
    @GetMapping("/api/ping")
    public Pong pong() { return new Pong("ok", Instant.now()); }

    public record Pong(String status, Instant ts) {}

    public static void main(String[] args) throws Exception {
        ObjectMapper om = new ObjectMapper().registerModule(new JavaTimeModule());
        byte[] bytes = om.writeValueAsBytes(new Pong("ok", Instant.EPOCH));
        System.out.println(new String(bytes));
    }
}
```

*Kotlin (REST + Jackson):*

```kotlin
package com.example.intro

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.registerKotlinModule
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.time.Instant

@RestController
class IntroController {
    @GetMapping("/api/ping")
    fun pong() = Pong("ok", Instant.now())
}

data class Pong(val status: String, val ts: Instant)

fun main() {
    val om = jacksonObjectMapper().registerKotlinModule().registerModule(JavaTimeModule())
    println(om.writeValueAsString(Pong("ok", Instant.EPOCH)))
}
```

---

## Зачем это

Главная причина — **совместимость**: ваши сервисы и клиенты договорились, как обмениваться данными. Хороший формат и стабильный контракт снимают бесконечные инциденты «у нас `amount` стал строкой». Схема (явная или неявная) делает договорённость проверяемой, а не «на словах».

Вторая — **производительность и стоимость**. Сотни миллионов сериализаций в день — это процессорное время, память, сетевые байты. Выбор формата и DTO напрямую влияет на латентность API и счёт за инфраструктуру. Внутри периметра часто выгодно перейти на бинарники (Protobuf/Avro).

Третья — **корректность**. Денежные поля нужно хранить с точностью (BigDecimal), даты — в ISO-8601/Instant, списки — предсказуемо. Неправильный выбор типа в JSON приводит к потерям центов или «сдвигам по часовым поясам».

Четвёртая — **безопасность и отказоустойчивость**. Десериализаторы — входная точка. Они должны валидировать вход, не падать на странных payload, возвращать предсказуемые ошибки клиенту и логировать контекст.

Пятая — **эволюция**. Контракты меняются: добавляются поля, меняются значения enum. Формат и политика версионирования (semver/`v2` в URL/контент-негациация) позволяют переходить без «большого шторма».

Шестая — **наблюдаемость**. Если формат прозрачен (JSON), его легче логировать и инспектировать. Если бинарный — добавьте инструменты для дешифровки (админ-эндпоинт «decode by schema»), чтобы отладка не превратилась в «чёрный ящик».

Седьмая — **регуляторика и интеграции**. XML (ISO 20022 и пр.) может быть необходим из-за внешних требований. Умение работать с несколькими форматами — не роскошь, а требование продакшна.

**Мини-бенч сериализации (размер/время).**
*Java:*

```java
package com.example.why;

import com.fasterxml.jackson.databind.ObjectMapper;

public class SizeBench {
    public record Customer(long id, String name, String email) {}
    public static void main(String[] args) throws Exception {
        ObjectMapper om = new ObjectMapper();
        Customer c = new Customer(1, "Alice", "alice@example.com");
        long t0 = System.nanoTime();
        byte[] bytes = om.writeValueAsBytes(c);
        long ms = (System.nanoTime() - t0) / 1_000_000;
        System.out.println("len=" + bytes.length + " timeMs=" + ms);
    }
}
```

*Kotlin:*

```kotlin
package com.example.why

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper

data class Customer(val id: Long, val name: String, val email: String)

fun main() {
    val om = jacksonObjectMapper()
    val t0 = System.nanoTime()
    val bytes = om.writeValueAsBytes(Customer(1, "Alice", "alice@example.com"))
    val ms = (System.nanoTime() - t0) / 1_000_000
    println("len=${bytes.size} timeMs=$ms")
}
```

---

## Где используется

В HTTP-API — это тело запроса/ответа. Spring MVC сам применяет `MappingJackson2HttpMessageConverter` для JSON, а вы управляете поведением через `ObjectMapper` и аннотации. Важно помнить про **Accept/Content-Type**: клиенты должны явно просить нужный формат.

В Kafka — это сериализаторы/десериализаторы (Serde). Для «просто JSON» используют String/ByteArray, для строгой схемы — Avro/Protobuf + Schema Registry. Это влияет на размер топика и скорость консьюмеров/продюсеров.

В БД — это JSON/JSONB поля (Postgres), которые позволяют хранить полуструктурированные данные. Но сериализация здесь — не повод тащить домены как есть: думайте о миграциях и индексах (GIN, jsonb_path_ops).

В файловых форматах — CSV для выгрузок/импорта, Parquet для аналитики, Protobuf/Avro как контейнер обмена между сервисами. Это разные «миры» с разной стоимостью операций и tooling’ом.

В конфигурации — YAML/Properties. Это тоже сериализация, только конфигурационного пространства. `@ConfigurationProperties` + Bean Validation помогают безопасно декодировать конфиги в типизированные классы.

В логировании — JSON-логи (структурированные), чтобы APM (ELK/OpenSearch) мог индексировать поля без парсинга строк. Это повышает наблюдаемость ошибок сериализации.

В интеграциях с внешними системами — XML остаётся реалией (банковские стандарты, B2B-шлюзы). Поддержка нескольких форматов — полезный навык.

**Kafka: JSON-продюсер (минимум).**
*Java:*

```java
package com.example.where;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class EventProducer {
    private final KafkaTemplate<String, String> kafka;
    public EventProducer(KafkaTemplate<String, String> kafka) { this.kafka = kafka; }
    public void sendJson(String topic, String json) { kafka.send(topic, json); }
}
```

*Kotlin:*

```kotlin
package com.example.where

import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Service

@Service
class EventProducer(private val kafka: KafkaTemplate<String, String>) {
    fun sendJson(topic: String, json: String) = kafka.send(topic, json)
}
```

---

## Какие виды бывают

Текстовые (JSON, XML, YAML), бинарные (Protobuf, Avro, Smile/CBOR, MsgPack), колоночные (Parquet, ORC) и табличные (CSV/TSV). Текстовые хороши читаемостью и простотой отладки, бинарные — компактностью и скоростью, колоночные — аналитическими селективными чтениями, табличные — совместимостью с Excel.

JSON — стандарт для веба: читаем, гибок, но двусмыслен с типами чисел. XML — строг, многословен, с богатой экосистемой схем и подписей. YAML — король конфигов, но не лучший кандидат для API. Protobuf/Avro — строгие, компактные, требуют схем и генерации.

В Spring формат выбирается по `Content-Type`/`Accept` и набору зарегистрированных `HttpMessageConverter`. Можно одновременно поддерживать JSON и XML, а выбор доверить контент-негациации — это гибко и удобно для клиентов.

Для межсервисной коммуникации можно иметь разные форматы на разных слоях: внешние REST — JSON, внутренние RPC — gRPC/Protobuf, события — Avro/Schema Registry, аналитика — Parquet в Data Lake. Это нормальный «поликультурный» стек.

Выбор — это компромисс. Например, JSON удобен, но объёмен. Protobuf быстрее, но сложнее в отладке и требует генерации. CSV прост, но слаботипизирован. Parquet великолепен для больших аналитических таблиц, но бесполезен в онлайн-запросах.

Критерии выбора: **кто потребитель**, **какой объём/латентность**, **как эволюционирует контракт**, **какие инструменты есть у команды** и **какие внешние требования**.

**Контент-негациация (дефолт JSON, но можем отдать XML).**
*Java:*

```java
package com.example.types;

import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.web.servlet.config.annotation.ContentNegotiationConfigurer;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer c) {
        c.defaultContentType(MediaType.APPLICATION_JSON);
        c.mediaType("json", MediaType.APPLICATION_JSON);
        c.mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

*Kotlin:*

```kotlin
package com.example.types

import org.springframework.context.annotation.Configuration
import org.springframework.http.MediaType
import org.springframework.web.servlet.config.annotation.ContentNegotiationConfigurer
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer

@Configuration
class WebConfig : WebMvcConfigurer {
    override fun configureContentNegotiation(c: ContentNegotiationConfigurer) {
        c.defaultContentType(MediaType.APPLICATION_JSON)
        c.mediaType("json", MediaType.APPLICATION_JSON)
        c.mediaType("xml", MediaType.APPLICATION_XML)
    }
}
```

---

## Какие задачи и проблемы решает

Правильный выбор формата снижает **стоимость интеграций**: меньше багов от несовместимых изменений, меньше времени на «разбор полётов» в проде. Наличие схем и правил эволюции позволяет автоматизировать проверки в CI (compatibility checks).

Он решает **производственные боли**: лишние мегабайты JSON замедляют мобильный клиент, неправильно сериализованные Instant «едут» по TZ, double ломает деньги. Точное описание правил и проверка на границе — страховка.

Формат помогает **версионировать API**: добавляйте поля безопасно, избегайте переименований без переходного периода, используйте `ProblemDetails` для ошибок (а не «строковые тексты»), поддерживайте несколько медиа-типов при необходимости.

Формат и сериализатор — важная часть **безопасности**: отключите опасный полиморфизм в Jackson, ограничьте классы, проверяйте входные данные, не раскрывайте лишнюю внутреннюю структуру.

В **наблюдаемости** формат определяет, как легко диагностировать сбои: JSON-логи облегчают жизнь, а бинарные требуют утилит. Добавляйте декодирующие эндпоинты/CLI, если без них тяжело.

Для **кэширования** (ETag/Last-Modified) важно, чтобы сериализация была детерминированной (одни и те же поля — один и тот же байтовый поток), иначе валидаторы кэша не сработают. Это ещё одна причина контролировать порядок полей и правила сериализации.

И наконец, формат — это **граница ответственности** между командами. Согласованный контракт снижает коммуникационные издержки и позволяет независимо релизить части системы.

**ETag руками (демо).**
*Java:*

```java
package com.example.problems;

import org.springframework.http.ResponseEntity;
import org.springframework.util.DigestUtils;
import org.springframework.web.bind.annotation.*;

import java.nio.charset.StandardCharsets;

@RestController
@RequestMapping("/api/docs")
public class EtagController {
    @GetMapping("/{id}")
    public ResponseEntity<String> get(@PathVariable String id, @RequestHeader(value="If-None-Match", required=false) String inm) {
        String body = "{\"id\":\"" + id + "\",\"content\":\"hello\"}";
        String etag = "\"" + DigestUtils.md5DigestAsHex(body.getBytes(StandardCharsets.UTF_8)) + "\"";
        if (etag.equals(inm)) return ResponseEntity.status(304).eTag(etag).build();
        return ResponseEntity.ok().eTag(etag).body(body);
    }
}
```

*Kotlin:*

```kotlin
package com.example.problems

import org.springframework.http.ResponseEntity
import org.springframework.util.DigestUtils
import org.springframework.web.bind.annotation.*
import java.nio.charset.StandardCharsets

@RestController
@RequestMapping("/api/docs")
class EtagController {
    @GetMapping("/{id}")
    fun get(@PathVariable id: String, @RequestHeader("If-None-Match", required = false) inm: String?): ResponseEntity<String> {
        val body = """{"id":"$id","content":"hello"}"""
        val etag = "\"${DigestUtils.md5DigestAsHex(body.toByteArray(StandardCharsets.UTF_8))}\""
        return if (etag == inm) ResponseEntity.status(304).eTag(etag).build()
        else ResponseEntity.ok().eTag(etag).body(body)
    }
}
```

---

# 1. Форматы данных: выбор и когда использовать

## JSON: самый распространённый формат, чтение/запись с использованием Jackson

JSON — дефолт для веб-API благодаря читаемости, широкому tooling’у и лёгкости прототипирования. В Spring Boot JSON поддерживается «из коробки»: достаточно вернуть объект из контроллера — и он превратится в JSON. Поддержка дат (JSR-310), `BigDecimal`, стратегий именования, включения/исключения полей — всё настраивается в `ObjectMapper`.

Для длительных контрактов важно определить политику эволюции: **добавление** поля — ок (клиенты его игнорируют), **удаление** — только после депрекейта и смены версии, **переименование** — сперва дублируйте новое поле, затем удаляйте старое. Так вы избегаете «скрытых» breaking changes.

С JSON легко отлаживать интеграции (curl/Postman), писать контракт-тесты (MockMvc/WebTestClient) и документировать API (Springdoc/OpenAPI). Это снижает входной порог для новых участников команды и внешних партнёров.

Минусы JSON — размер и двусмысленность чисел. Денежные суммы сериализуйте как `BigDecimal` (и по возможности как строки на фронте, чтобы избежать потерь), даты — в ISO-8601 (не timestamp, если есть люди/часовые пояса).

Для избирательной выдачи полей используйте `@JsonView`/DTO-проекции: не лепите «толстые» ответы. Это экономит байты и ускоряет клиентов, особенно мобильных.

Имейте стандарт ошибок (RFC 7807/ProblemDetails) — об этом дальше. «Сырые» исключения в JSON без структуры — антипаттерн, их трудно обрабатывать автоматически.

В завершение — замерьте сериализацию/десериализацию на реальных payload. Иногда «легкая» оптимизация (ренейминг полей, отказ от вложенных структур) даёт хороший прирост.

*Java (настройка Jackson + контроллер):*

```java
package com.example.json;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.time.Instant;

@RestController
@RequestMapping("/api/orders")
class OrderController {
    @GetMapping("/{id}")
    public OrderDto get(@PathVariable long id) {
        return new OrderDto(id, "PAID", new BigDecimal("99.90"), Instant.now());
    }

    public record OrderDto(long id, String status, BigDecimal amount, Instant createdAt) {}
}

@Configuration
class JacksonCfg {
    @Bean ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .setSerializationInclusion(JsonInclude.Include.NON_NULL)
                .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }
}
```

*Kotlin (аналог):*

```kotlin
package com.example.json

import com.fasterxml.jackson.annotation.JsonInclude
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.PropertyNamingStrategies
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.fasterxml.jackson.module.kotlin.registerKotlinModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.bind.annotation.*
import java.math.BigDecimal
import java.time.Instant

@RestController
@RequestMapping("/api/orders")
class OrderController {
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long) = OrderDto(id, "PAID", BigDecimal("99.90"), Instant.now())
    data class OrderDto(val id: Long, val status: String, val amount: BigDecimal, val createdAt: Instant)
}

@Configuration
class JacksonCfg {
    @Bean
    fun objectMapper() = ObjectMapper()
        .registerKotlinModule()
        .registerModule(JavaTimeModule())
        .setSerializationInclusion(JsonInclude.Include.NON_NULL)
        .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
}
```

---

## XML: когда имеет смысл, если API поддерживает старые системы или стандарт

XML актуален там, где требуется строгая схема (XSD), цифровые подписи (XML-DSig), стандарты индустрии (например, ISO 20022), либо когда партнёр «говорит» только на XML. Это не «устаревшая экзотика», а живой формат в B2B-интеграциях. Минусы — многословность и размер; плюсы — зрелая экосистема и валидируемость.

Spring Boot позволяет отдать XML так же просто, как JSON. Достаточно добавить модуль `jackson-dataformat-xml` и указать `produces = application/xml`. DTO можно пометить `@JacksonXmlRootElement`/`@JacksonXmlProperty`, чтобы контролировать корневой элемент и имена тегов.

Приятная особенность — можно поддерживать **оба** формата одновременно: по умолчанию JSON, по `Accept: application/xml` — XML. Это снижает трение при миграции партнёров.

Валидация по XSD — сильная сторона XML. Для критичных протоколов это полезно: вы не принимаете «мусор», а клиенты получают чёткие сообщения о несоответствии схеме. Интегрируйте XSD-валидацию в слой адаптеров.

Подписи и шифрование (WS-Security) также исторически опираются на XML-стек. Если есть требование «криптографии на уровне сообщения», возможно, без XML не обойтись.

Наконец, XML удобен для сложных документо-ориентированных структур (вложенные коллекции с атрибутами, неймспейсы). Но для обычного REST JSON обычно проще и дешевле.

**Зависимость для XML:**
*Groovy DSL:* `implementation 'com.fasterxml.jackson.dataformat:jackson-dataformat-xml'`
*Kotlin DSL:* `implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-xml")`

*Java (контроллер, который отдаёт XML):*

```java
package com.example.xml;

import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlProperty;
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlRootElement;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

@JacksonXmlRootElement(localName = "customer")
class CustomerXml {
    @JacksonXmlProperty(localName = "id") public long id;
    @JacksonXmlProperty(localName = "name") public String name;
    public CustomerXml() {}
    public CustomerXml(long id, String name) { this.id = id; this.name = name; }
}

@RestController
@RequestMapping("/api/xml")
class XmlController {
    @GetMapping(value = "/customer", produces = MediaType.APPLICATION_XML_VALUE)
    public CustomerXml get() { return new CustomerXml(1, "Alice"); }
}
```

*Kotlin (аналог):*

```kotlin
package com.example.xml

import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlProperty
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlRootElement
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@JacksonXmlRootElement(localName = "customer")
data class CustomerXml(
    @JacksonXmlProperty(localName = "id") val id: Long,
    @JacksonXmlProperty(localName = "name") val name: String
)

@RestController
@RequestMapping("/api/xml")
class XmlController {
    @GetMapping("/customer", produces = [MediaType.APPLICATION_XML_VALUE])
    fun get(): CustomerXml = CustomerXml(1, "Alice")
}
```

---

## YAML: конфигурация, но не для передачи данных, сравнение с JSON

YAML — лучший друг конфигурации: многострочные значения, якоря/алиасы, читаемость. В Spring Boot он используется в `application.yml`, а биндинг в типы делает `@ConfigurationProperties`. Однако в качестве **тела REST-запросов/ответов** YAML не рекомендован: неоднозначности пробелов/табов, нетрудно ошибиться, tooling слабее, чем у JSON.

Правильная практика: **конфиг — YAML, данные — JSON**. Так вы получаете и удобство настройки, и надёжность обмена. Если всё же надо принимать YAML (редко, но бывает) — ограничьте область и валидируйте строго.

С `@ConfigurationProperties` удобно описывать структуру конфигов и валидировать её через Jakarta Validation. Это переносит часть «валидации конфигов» из runtime в стартап приложения и CI (Spring Boot может падать при некорректных конфигурациях — и это хорошо).

Наблюдаемость конфигов повышают actuator-эндпоинты, но не светите чувствительные свойства. Для секретов — externalized config (Vault, K8s secrets), а не хранение прямо в YAML в репозитории.

При сравнении с JSON: YAML короче за счёт отсутствия кавычек/фигурных скобок, но хуже переносим в качестве протокола. JSON, наоборот, многословнее, но однозначнее и лучше поддерживается браузерами/мобильными SDK.

Если вам нужно «схемировать» YAML — рассмотрите JSON Schema поверх YAML (да, так можно), либо жёсткий биндинг через `@ConfigurationProperties` и валидаторы — в приложении это проще и надёжнее.

*Java (биндинг и валидация YAML в тип):*

```java
package com.example.yaml;

import jakarta.validation.constraints.NotBlank;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.validation.annotation.Validated;

@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProps {
    @NotBlank
    private String title;
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
}
```

*Kotlin:*

```kotlin
package com.example.yaml

import jakarta.validation.constraints.NotBlank
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Configuration
import org.springframework.validation.annotation.Validated

@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
class AppProps {
    @NotBlank
    lateinit var title: String
}
```

*`application.yml`:*

```yaml
app:
  title: "Demo Service"
```

---

## Protobuf: эффективность для больших объёмов, бинарный формат, использование с gRPC

**Protocol Buffers** — компактный бинарный формат со строгой схемой (`.proto`). Он даёт малый размер, быструю (де)сериализацию и чёткие правила эволюции (добавление полей обычно безопасно). Типичный сценарий — gRPC-вызовы между микросервисами и события в Kafka.

Основная цена — генерация кода и «нечитаемость» без инструментов. Это решается плагинами Gradle и вспомогательными утилитами/админ-эндпоинтами для расшифровки сообщений по схеме.

При миграции с JSON «наружу JSON, внутри Protobuf» — распространённая стратегия: удобство для внешних клиентов и эффективность внутри периметра. Следите за согласованностью моделей (source of truth — proto-схема).

Валидация полезной нагрузки перекладывается на уровень схемы и бизнес-логики. Для сложных инвариантов используйте валидаторы на сервере; Protobuf гарантирует лишь типы/наличие полей.

С gRPC внимательно отнеситесь к таймаутам, ретраям, бюджету на сериализацию/десериализацию и метрикам. Бинарный формат «не бесплатный» — но почти всегда дешевле JSON под нагрузкой.

И помните: для «админского» дебага держите инструмент, умеющий преобразовать бинарь в JSON по схеме (в том числе прямо в сервисе).

**Gradle: подключение protobuf-плагина.**
*Groovy DSL:*

```groovy
plugins { id 'com.google.protobuf' version '0.9.4' }
dependencies {
    implementation 'io.grpc:grpc-netty-shaded:1.66.0'
    implementation 'io.grpc:grpc-protobuf:1.66.0'
    implementation 'io.grpc:grpc-stub:1.66.0'
    implementation 'com.google.protobuf:protobuf-java:4.28.3'
}
protobuf {
  protoc { artifact = "com.google.protobuf:protoc:4.28.3" }
}
```

*Kotlin DSL:*

```kotlin
plugins { id("com.google.protobuf") version "0.9.4" }
dependencies {
    implementation("io.grpc:grpc-netty-shaded:1.66.0")
    implementation("io.grpc:grpc-protobuf:1.66.0")
    implementation("io.grpc:grpc-stub:1.66.0")
    implementation("com.google.protobuf:protobuf-java:4.28.3")
}
protobuf {
  protoc { artifact = "com.google.protobuf:protoc:4.28.3" }
}
```

*`customer.proto`:*

```proto
syntax = "proto3";
package com.example.proto;

message Customer {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

*Java (использование сгенерированного класса):*

```java
package com.example.protobuf;

import com.example.proto.CustomerOuterClass.Customer;

public class ProtoDemo {
    public static void main(String[] args) throws Exception {
        byte[] bytes = Customer.newBuilder().setId(1).setName("Alice").setEmail("alice@example.com").build().toByteArray();
        Customer parsed = Customer.parseFrom(bytes);
        System.out.println(parsed.getName());
    }
}
```

*Kotlin:*

```kotlin
package com.example.protobuf

import com.example.proto.CustomerOuterClass.Customer

fun main() {
    val bytes = Customer.newBuilder().setId(1).setName("Alice").setEmail("alice@example.com").build().toByteArray()
    val parsed = Customer.parseFrom(bytes)
    println(parsed.name)
}
```

---

## CSV, Avro, Parquet: другие специализированные форматы для данных, которые не подходят под JSON/XML

CSV — удобный минимум для обмена табличными данными: простой, совместимый с Excel/Google Sheets, лёгок для ручной проверки. Минусы — слабая типизация, проблемы экранирования разделителей и локалей для чисел/дат. Хорош для импорта/экспорта и «ручных» операций.

Avro — бинарный формат со схемой и отличной интеграцией в Kafka (через Schema Registry). Он даёт небольшие сообщения и строгую эволюцию контрактов. По удобству отладки — между JSON и Protobuf (есть avro-tools для конвертации в JSON).

Parquet — колоночный формат для аналитики (Spark/Trino/Presto). Идеален для больших объёмов, где читаются «несколько колонок из миллиарда строк». Для онлайна Parquet не используется, но для витрин данных — must-have.

Выбор между Avro и Protobuf часто вкусовой/экосистемный. Если у вас Confluent-мир и Kafka-центристский подход — Avro частый выбор; если gRPC-RPC — Protobuf естественнее. Оба дают схему и компактность.

Для CSV держите строгий шаблон: заголовки, кодировка UTF-8, разделители, формат дат. И обязательно валидируйте содержимое до записи в БД — «грязные» CSV встречаются часто.

Для Parquet подумайте о совместимости с аналитической платформой, партиционировании путей (S3 `.../dt=YYYY-MM-DD/`) и размере файлов (не слишком мелко, не слишком крупно для оптимального сканирования).

*Java (минимальный CSV-writer):*

```java
package com.example.csv;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.List;

public class CsvDemo {
    public static void main(String[] args) throws Exception {
        List<String[]> rows = List.of(new String[]{"id","name"}, new String[]{"1","Alice"}, new String[]{"2","Bob"});
        try (PrintWriter w = new PrintWriter(new OutputStreamWriter(new FileOutputStream("out.csv"), StandardCharsets.UTF_8))) {
            for (String[] r : rows) w.println(String.join(",", r));
        }
    }
}
```

*Kotlin:*

```kotlin
package com.example.csv

import java.io.File
import java.nio.charset.StandardCharsets

fun main() {
    val rows = listOf(arrayOf("id","name"), arrayOf("1","Alice"), arrayOf("2","Bob"))
    File("out.csv").printWriter(StandardCharsets.UTF_8).use { w ->
        rows.forEach { w.println(it.joinToString(",")) }
    }
}
```
# 2. Сериализация и десериализация данных в Spring

## Jackson как дефолт для JSON: настройка `ObjectMapper`, работа с датами/числами, валидация

Jackson — стандартный JSON-механизм в Spring Boot: он автоматом подключается через `MappingJackson2HttpMessageConverter`. Это значит, что любую модель, которую вы возвращаете из контроллера, Spring превратит в JSON без вашей явной сериализации. Управляет поведением объект `ObjectMapper`, который вы можете донастроить бином.

Ключевая тема — даты и время. Начиная с Java 8 мы используем типы JSR-310 (`Instant`, `LocalDate`, `OffsetDateTime`). Чтобы Jackson писал их в ISO-8601, а не в epoch timestamps, регистрируем `JavaTimeModule` и отключаем `WRITE_DATES_AS_TIMESTAMPS`. Это повышает переносимость и предсказуемость при интеграциях в разных часовых поясах.

Числа, особенно деньги, требуют точности. Для денежных сумм выбирайте `BigDecimal` и сохраняйте количество знаков. В ряде фронтовых SDK числа могут превращаться во float/double с потерей точности; заранее согласуйте формат (часто — строки для сумм) и будьте последовательны на всех маршрутах.

Правила включения полей тоже важны. `JsonInclude.Include.NON_NULL` избавит от «шума» в ответах и уменьшит полезную нагрузку. Для крупных DTO можно настроить sn ake_case через `PropertyNamingStrategies.SNAKE_CASE`, если этого требую т клиенты, но предпочтительнее единый стиль для всей системы.

Справляться с неизвестными полями во входящих запросах удобнее через явные аннотации. `@JsonIgnoreProperties(ignoreUnknown = true)` на DTO даст вам мягкую эволюцию, когда мобильный клиент прислал больше, чем вы ожидаете, — и вы сможете игнорировать лишнее до релиза серверной поддержки.

Валидацию входящих данных не стоит смешивать с десериализацией. Пусть Jackson просто материализует объект, а проверку сделает Jakarta Bean Validation (`@Valid`, `@NotBlank`, `@DecimalMin`). Так вы разделяете ответственность: парсинг и валидация, и можете централизованно оформить ошибки.

Наконец, помните про idempotency и кэширование: детерминированная сериализация полезна для ETag. Стабильный порядок полей и постоянные правила форматирования повысят cache hit-rate у CDN/прокси и снизят трафик.

*Java (кастомизация `ObjectMapper` и простой REST):*

```java
package com.example.s2.jackson;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.time.Instant;

@RestController
@RequestMapping("/api/payments")
class PaymentController {
    @PostMapping
    public Payment create(@RequestBody Payment in) { return in.withCreatedAt(Instant.now()); }
}

record Payment(
        @NotBlank String id,
        @DecimalMin("0.00") BigDecimal amount,
        Instant createdAt
) {
    Payment withCreatedAt(Instant ts) { return new Payment(id, amount, ts); }
}

@Configuration
class JacksonConfig {
    @Bean ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .setSerializationInclusion(JsonInclude.Include.NON_NULL)
                .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }
}
```

*Kotlin (аналог):*

```kotlin
package com.example.s2.jackson

import com.fasterxml.jackson.annotation.JsonInclude
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.PropertyNamingStrategies
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.fasterxml.jackson.module.kotlin.registerKotlinModule
import jakarta.validation.constraints.DecimalMin
import jakarta.validation.constraints.NotBlank
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.bind.annotation.*
import java.math.BigDecimal
import java.time.Instant

@RestController
@RequestMapping("/api/payments")
class PaymentController {
    @PostMapping
    fun create(@RequestBody inDto: Payment) = inDto.copy(createdAt = Instant.now())
}

data class Payment(
    @field:NotBlank val id: String,
    @field:DecimalMin("0.00") val amount: BigDecimal,
    val createdAt: Instant? = null
)

@Configuration
class JacksonConfig {
    @Bean
    fun objectMapper() = ObjectMapper()
        .registerKotlinModule()
        .registerModule(JavaTimeModule())
        .setSerializationInclusion(JsonInclude.Include.NON_NULL)
        .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE)
}
```

*Gradle Groovy:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
    implementation 'com.fasterxml.jackson.module:jackson-module-kotlin'
}
```

*Gradle Kotlin:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("com.fasterxml.jackson.datatype:jackson-datatype-jsr310")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}
```

---

## `@JsonView`, `@JsonSerialize`, `@JsonDeserialize` для кастомизации сериализации/десериализации

`@JsonView` решает задачу «разные представления одного ресурса». Часто нужно отдать публичному клиенту безопасный набор полей, а внутреннему — расширенный. С `@JsonView` вы помечаете поля принадлежностью к view и выбираете нужное представление на уровне контроллера.

Кастомные сериализаторы и десериализаторы дают тонкий контроль. Например, можно маскировать PII (email, телефон), сериализовать значения с шифрованием или применять нетривиальное преобразование типа (строка↔сложный объект). Расширение делается через `StdSerializer`/`StdDeserializer`.

Нужно помнить, что кастомные сериализаторы — горячая точка производительности. Их стоит писать максимально без лишних аллокаций, использовать уже открытые `JsonGenerator` и избегать промежуточных объектов. Часто выгоднее подготовить данные на уровне DTO.

`@JsonDeserialize` удобно применять, если внешний мир присылает «странные» форматы, а менять контракт нельзя. Вы пишете адаптер, который принимает разные варианты (например, `id` как число или строку) и преобразует к вашему типу без падения.

Дополнительно стоит рассмотреть Mix-in классы Jackson. Они позволяют навесить аннотации на классы, которые вы не контролируете. Это полезно в интеграциях с внешними моделями, где правки исходников нежелательны.

Не злоупотребляйте `@JsonView` для бизнес-логики. Он решает именно «витринный» срез полей. Для различающихся сценариев лучше явные DTO, иначе вы рискуете усложнить сопровождение и тестирование.

Тестируйте кастомную сериализацию отдельно: юнит-тесты на сериализатор/десериализатор окупятся при смене версий Jackson и при эволюции контрактов. Это особенно важно, если вы маскируете данные в логах.

*Java (`@JsonView` + кастомные (де)сериализаторы):*

```java
package com.example.s2.views;

import com.fasterxml.jackson.annotation.JsonView;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;

class Views { interface Public {} interface Internal extends Public {} }

class EmailMaskSerializer extends JsonSerializer<String> {
    @Override public void serialize(String v, JsonGenerator g, SerializerProvider p) throws IOException {
        int at = v.indexOf('@'); g.writeString(at > 1 ? v.charAt(0) + "***" + v.substring(at) : "***");
    }
}

class FlexibleIdDeserializer extends JsonDeserializer<Long> {
    @Override public Long deserialize(JsonParser p, DeserializationContext c) throws IOException {
        var n = p.getText();
        return Long.parseLong(n);
    }
}

class UserDto {
    @JsonView(Views.Public.class) public Long id;
    @JsonView(Views.Public.class) public String name;
    @JsonView(Views.Internal.class) @JsonSerialize(using = EmailMaskSerializer.class) public String email;
    @JsonDeserialize(using = FlexibleIdDeserializer.class) public Long externalId;
    public UserDto(Long id, String name, String email, Long externalId) { this.id=id; this.name=name; this.email=email; this.externalId=externalId; }
}

@RestController
@RequestMapping("/api/users")
class UserController {
    @GetMapping("/{id}") @JsonView(Views.Public.class)
    public UserDto get(@PathVariable long id) { return new UserDto(id, "Alice", "alice@example.com", 42L); }
}
```

*Kotlin (аналог):*

```kotlin
package com.example.s2.views

import com.fasterxml.jackson.annotation.JsonView
import com.fasterxml.jackson.core.JsonGenerator
import com.fasterxml.jackson.core.JsonParser
import com.fasterxml.jackson.databind.DeserializationContext
import com.fasterxml.jackson.databind.JsonDeserializer
import com.fasterxml.jackson.databind.JsonSerializer
import com.fasterxml.jackson.databind.SerializerProvider
import com.fasterxml.jackson.databind.annotation.JsonDeserialize
import com.fasterxml.jackson.databind.annotation.JsonSerialize
import org.springframework.web.bind.annotation.*

object Views { interface Public; interface Internal : Public }

class EmailMaskSerializer : JsonSerializer<String>() {
    override fun serialize(value: String, gen: JsonGenerator, serializers: SerializerProvider) {
        val at = value.indexOf('@'); gen.writeString(if (at > 1) "${value[0]}***${value.substring(at)}" else "***")
    }
}

class FlexibleIdDeserializer : JsonDeserializer<Long>() {
    override fun deserialize(p: JsonParser, ctxt: DeserializationContext): Long = p.text.toLong()
}

data class UserDto(
    @JsonView(Views.Public::class) val id: Long,
    @JsonView(Views.Public::class) val name: String,
    @JsonView(Views.Internal::class) @JsonSerialize(using = EmailMaskSerializer::class) val email: String,
    @JsonDeserialize(using = FlexibleIdDeserializer::class) val externalId: Long
)

@RestController
@RequestMapping("/api/users")
class UserController {
    @GetMapping("/{id}")
    @JsonView(Views.Public::class)
    fun get(@PathVariable id: Long) = UserDto(id, "Alice", "alice@example.com", 42L)
}
```

---

## Kotlin-специфичные аспекты: `data`-классы, `@JsonInclude`, `@JvmField`, проблемы с nullability

Kotlin data-классы отлично сочетаются с Jackson, если подключён модуль `jackson-module-kotlin`. Он понимает конструкторы с дефолтными значениями и нуллабельность параметров, что делает модели компактными и безопасными. Это позволяет не писать сеттеры/геттеры, сохраняя иммутабельность.

Nullability — ключевая особенность Kotlin. Поля типа `String?` трактуются как опциональные, и Jackson корректно десериализует отсутствие поля в `null`. Но если вы укажете `String` без `?`, а поле отсутствует, будет ошибка десериализации. Это помогает ловить неполные данные рано.

Дефолтные значения конструктора делают поля опциональными без явной нуллабельности. Например, `val note: String = ""` позволяет принимать отсутствие поля, но тут легко скрыть ошибку контракта. Лучше явно использовать `?` и валидацию, чтобы не терять семантику «пусто vs отсутствует».

Аннотация `@JsonInclude(JsonInclude.Include.NON_NULL)` в Kotlin обычно ставится на класс для исключения `null`-полей из ответа. Это снижает размер JSON и упрощает клиента. Для более тонкого контроля можно настроить включение только ненулевых и небустых коллекций.

`@JvmField` полезна, когда вы делаете `object` с константами и хотите экспонировать поле как настоящую Java-константу (без геттеров). На сериализацию это не влияет, но облегчает Interop с Java-кодом и аннотациями.

С `sealed`-иерархиями и полиморфизмом в Kotlin будьте осторожны. Полиморфная десериализация Jackson требует явной конфигурации `@JsonTypeInfo`/`@JsonSubTypes` и часто — явного разрешения в `ObjectMapper`. Во многих случаях проще не тусовать полиморфизм в публичном контракте.

Наконец, присмотритесь к `@field:`-аннотациям для Jakarta Validation в Kotlin. Чтобы валидация работала, аннотацию нужно ставить на поле, а не на параметр конструктора: `@field:NotBlank val name: String`. Это частая ошибка начинающих.

*Java (эквивалент Kotlin-модели с null):*

```java
package com.example.s2.ktinterop;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record Customer(Long id, String name, String note) { }
```

*Kotlin (модель, nullability и дефолты):*

```kotlin
package com.example.s2.ktinterop

import com.fasterxml.jackson.annotation.JsonInclude
import jakarta.validation.constraints.NotBlank

@JsonInclude(JsonInclude.Include.NON_NULL)
data class Customer(
    val id: Long?,
    @field:NotBlank val name: String,
    val note: String? = null
)
```

*Контроллер Kotlin (валидация и JSON):*

```kotlin
package com.example.s2.ktinterop

import jakarta.validation.Valid
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/customers")
class CustomerController {
    @PostMapping
    fun create(@Valid @RequestBody c: Customer) = c.copy(id = 1L)
}
```

---

## Конфигурация Jackson: обработка исключений, кастомизация путей, использование мапперов для сложных типов

Ошибки десериализации — нормальная часть жизни API. Важно превращать их в понятные клиенту ответы. Через `@ControllerAdvice` можно перехватить `HttpMessageNotReadableException`, `MismatchedInputException` и вернуть согласованный формат ошибки (например, RFC 7807). Это повышает DX и ускоряет отладку.

Кастомизация путей — это про контроль имён/форматов на уровне классов. Вы можете задать глобальную стратегию именования, а где нужно — переопределить `@JsonProperty` для отдельного поля. Такой подход позволяет выдерживать внешний контракт при рефакторингах внутренних имён.

Для сложных типов, которые Jackson не знает как сериализовать (например, value-objects, Money, DomainId), удобно регистрировать модуль с сериализаторами/десериализаторами. Это изолирует конверсию в одном месте и снимает дублирование аннотаций по проекту.

Исключения сериализации тоже стоит перехватывать. Например, если вы случайно попали в цикл ссылок или словили `InvalidDefinitionException`, клиент должен получить структурированную ошибку, а не «500 и стек трейс». Централизованный обработчик упрощает сопровождение.

Разные профили окружений могут потребовать разную конфигурацию `ObjectMapper`. На проде полезно включать строгий режим (fail-fast), а на dev — более мягкий, чтобы ускорить разработку. В Spring это удобно делается через профили и отдельные конфигурационные бины.

Наблюдаемость крайне важна: логируйте корень проблемы и «path» до поля, где упало чтение. `MismatchedInputException.getPath()` содержит цепочку, ведущую к проблемному свойству. Эти данные экономят часы расследований.

Не забывайте о безопасности: запрещайте polymorphic typing по умолчанию, ограничивайте типы при `enableDefaultTyping` (лучше вообще не включать), и не принимайте на вход абстрактные типы без белых списков — это источник гадостей типа gadget-chain десериализации.

*Java (`@ControllerAdvice` + модуль):*

```java
package com.example.s2.errors;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.module.SimpleModule;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
class JsonErrorHandler {
    @ExceptionHandler(JsonProcessingException.class)
    public ResponseEntity<?> onJson(JsonProcessingException ex) {
        var body = new Problem("Invalid JSON", ex.getOriginalMessage());
        return ResponseEntity.badRequest().body(body);
    }
    record Problem(String title, String detail) {}
}

@Configuration
class MapperModuleConfig {
    @Bean ObjectMapper objectMapper() {
        SimpleModule module = new SimpleModule("valueObjects");
        // module.addSerializer(Money.class, new MoneySerializer());
        return new ObjectMapper().registerModule(module);
    }
}
```

*Kotlin (аналог):*

```kotlin
package com.example.s2.errors

import com.fasterxml.jackson.core.JsonProcessingException
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.module.SimpleModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class JsonErrorHandler {
    data class Problem(val title: String, val detail: String?)
    @ExceptionHandler(JsonProcessingException::class)
    fun onJson(ex: JsonProcessingException): ResponseEntity<Problem> =
        ResponseEntity.badRequest().body(Problem("Invalid JSON", ex.originalMessage))
}

@Configuration
class MapperModuleConfig {
    @Bean
    fun objectMapper(): ObjectMapper {
        val module = SimpleModule("valueObjects")
        // module.addSerializer(Money::class.java, MoneySerializer())
        return ObjectMapper().registerModule(module)
    }
}
```

---

## Десериализация коллекций, вложенных объектов и типов данных в JSON (`Map`, `List<T>`)

Из-за стирания типов в Java Jackson не знает, во что разворачивать обобщения, если вы не подскажете. Для `List<T>` и `Map<K,V>` используйте `TypeReference`. Это сохранит информацию о типе в рантайме и позволит корректно десериализовать вложенные структуры.

В Kotlin `jackson-module-kotlin` добавляет reified-экстеншены `readValue<T>()`, и жизнь проще. Однако на пересечении с Java-кодом или при сложных обобщениях всё равно иногда удобнее объявить явный `TypeReference`.

С вложенными объектами сочетайте типовые ссылки и валидацию. Глубокие графы лучше уплощать в DTO для конкретных сценариев, избегая сериализации «толстых» доменных агрегатов. Так вы экономите байты и упрощаете эволюцию.

При работе с `Map<String, Any>` осторожно: вы теряете типовую безопасность и получаете «сырой» JSON-словарь. Это допустимо в точках расширения, но старайтесь быстро возвращаться к типам — так проще тестировать и поддерживать.

Если коллекции очень большие, подумайте о стриминговом чтении (`JsonParser`) или paging-паттерне. Не всегда нужно держать весь массив в памяти. Для гигантских входных данных выгодно читать объект за объектом и обрабатывать инкрементально.

С датами и числами в коллекциях применяются те же правила: JSR-310 и `BigDecimal`. Если у вас есть `List<BigDecimal>`, убедитесь, что клиент не отправляет строки, или подготовьте кастомный десериализатор, чтобы унифицировать поведение.

В тестах проверяйте и правильность типов, и корректность путей. Ошибки типа «список объектов стал списком мап» всплывают не сразу, поэтому контракт-тесты на JSON-структуры — must-have.

*Java (`TypeReference` для коллекций):*

```java
package com.example.s2.collections;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.List;
import java.util.Map;

public class ReadCollections {
    public static void main(String[] args) throws Exception {
        var json = """
            [{"id":1,"name":"A","attrs":{"k":"v"}},{"id":2,"name":"B","attrs":{"k":"w"}}]
        """;
        var om = new ObjectMapper();
        record Item(long id, String name, Map<String,String> attrs) {}
        List<Item> items = om.readValue(json, new TypeReference<List<Item>>() {});
        System.out.println(items.get(0).attrs().get("k"));
    }
}
```

*Kotlin (рефайд-чтение):*

```kotlin
package com.example.s2.collections

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue

data class Item(val id: Long, val name: String, val attrs: Map<String, String>)

fun main() {
    val json = """[{"id":1,"name":"A","attrs":{"k":"v"}}]"""
    val om = jacksonObjectMapper()
    val items: List<Item> = om.readValue(json)
    println(items.first().attrs["k"])
}
```

---

# 3. Проблемы с сериализацией и производительность

## Проблемы с большими объектами и глубокой вложенностью: рекурсия, циклические зависимости

Сложные доменные модели часто образуют циклы: заказ содержит покупателя, а покупатель — список заказов. Наивная сериализация такого графа приводит к бесконечной рекурсии и падению. Поэтому в публичных контракт ах следует отдавать **DTO**, а не сущности JPA.

Jackson предоставляет несколько способов разорвать цикл. Самые простые — пары `@JsonManagedReference`/`@JsonBackReference` или ссылочная модель `@JsonIdentityInfo`, где повторные узлы заменяются id. Оба подхода спасают от бесконечных обходов.

Разумнее проектировать ответы без циклов. Вместо «покупатель с заказами, в которых снова покупатель» отдавайте «покупатель с минимальной карточкой заказа», а у заказа — «id покупателя», но не весь объект. Это снижает нагрузку и убирает двусмысленности.

Глубокая вложенность бьёт и по производительности, и по UX. Большие JSON-деревья тяжело парсить на фронте, они создают задержки. Лучше разбивать ответ на несколько ресурсов или вводить «расширяемые» проекции по параметрам запроса.

Если вы вынуждены отдавать вложенные структуры, настройте лимиты глубины на уровне построителя DTO. Не полагайтесь на «случай, что никто не пришлёт граф из 20 уровней»; в проде такие ситуации происходят из-за данных.

Тесты на глубокую вложенность и циклы обязательны. Создайте фабрики, собирающие нетривиальные графы, и проверяйте, что сериализация не падает и не выдаёт мегабайтные JSON-ы.

Не сериализуйте ленивые коллекции JPA напрямую. `LAZY`-отношения при внезапном доступе в контроллере вызывают N+1 и неожиданную нагрузку. Делайте выборки, формируйте DTO в сервисе и возвращайте ровно то, что нужно.

*Java (цикл и разрыв ссылок):*

```java
package com.example.s3.cycles;

import com.fasterxml.jackson.annotation.JsonBackReference;
import com.fasterxml.jackson.annotation.JsonManagedReference;
import java.util.List;

class Customer {
    public long id;
    @JsonManagedReference public List<Order> orders;
}

class Order {
    public long id;
    @JsonBackReference public Customer customer;
}
```

*Kotlin (аналог):*

```kotlin
package com.example.s3.cycles

import com.fasterxml.jackson.annotation.JsonBackReference
import com.fasterxml.jackson.annotation.JsonManagedReference

data class Customer(
    val id: Long,
    @JsonManagedReference val orders: List<Order>
)

data class Order(
    val id: Long,
    @JsonBackReference val customer: Customer
)
```

---

## Настройки Jackson для ограничения глубины, лимитов объектов и избегания StackOverflow

В самом Jackson нет «магической» настройки «maxDepth», но вы можете построить защитный слой. Один вариант — собственные сериализаторы для «опасных» типов, отслеживающие глубину через `SerializerProvider.getAttribute`/`setAttribute` или `ThreadLocal`. При превышении лимита — выбрасывать контролируемое исключение.

`@JsonIdentityInfo` меняет модель сериализации на ссылочную и устраняет бесконечные циклы. Это полезно для графов, где повторные узлы могут встречаться часто. Клиент при этом должен понимать ссылочную модель — это компромисс между читаемостью и безопасностью.

Второй уровень защиты — запрет сериализации «тяжёлых» полей по умолчанию, с явным включением через параметр запроса. Например, `?expand=items` на сервере раскрывает коллекции, а без него возвращается агрегат без вложенностей. Такой подход снижает риск случайных перегрузок.

Можно использовать микс-ин для изоляции «опасных» аннотаций. Вы не трогаете доменную модель, а навешиваете ограничения сериализации изолированно. Это удобно, когда модель используется в разных контекстах с разными требованиями.

Не пренебрегайте лимитами на размер ответа в reverse-proxy (nginx, envoy) и контролем времени выполнения. Если сериализация «увела» вас в долгий GC из-за гигантского дерева, таймаут на уровне шлюза защитит фронт и мобильных пользователей.

Логи должны содержать идентификатор запроса и путь до поля, на котором «оборвалась» сериализация. Это важно, если вы ввели depth-лимиты: клиенту и разработчику нужна точная причина, почему он получил усечённый ответ или ошибку.

Pixel-perf оптимизации Jackson (например, отключение WRITE_DATES_AS_TIMESTAMPS, включение/выключение `FAIL_ON_EMPTY_BEANS`) не решают проблему глубины. Фундаментальный способ — архитектурно ограничивать графы и применять DTO-маппинг.

*Java (пример depth-ограничителя для конкретного типа):*

```java
package com.example.s3.depth;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.ser.std.StdSerializer;

class DepthLimitedNodeSerializer extends StdSerializer<Node> {
    private static final Object DEPTH_KEY = new Object();
    private final int maxDepth;
    DepthLimitedNodeSerializer(int maxDepth) { super(Node.class); this.maxDepth = maxDepth; }

    @Override
    public void serialize(Node value, JsonGenerator gen, SerializerProvider sp) throws java.io.IOException {
        Integer depth = (Integer) sp.getAttribute(DEPTH_KEY);
        depth = depth == null ? 0 : depth;
        if (depth > maxDepth) { gen.writeString("..."); return; }
        sp.setAttribute(DEPTH_KEY, depth + 1);
        gen.writeStartObject();
        gen.writeNumberField("id", value.id);
        gen.writeFieldName("children");
        gen.writeStartArray();
        for (Node n : value.children) {
            new DepthLimitedNodeSerializer(maxDepth).serialize(n, gen, sp);
        }
        gen.writeEndArray();
        gen.writeEndObject();
        sp.setAttribute(DEPTH_KEY, depth);
    }
}

class Node { public long id; public java.util.List<Node> children; }
```

*Kotlin (аналог):*

```kotlin
package com.example.s3.depth

import com.fasterxml.jackson.core.JsonGenerator
import com.fasterxml.jackson.databind.SerializerProvider
import com.fasterxml.jackson.databind.ser.std.StdSerializer

data class Node(val id: Long, val children: List<Node>)

class DepthLimitedNodeSerializer(private val maxDepth: Int) : StdSerializer<Node>(Node::class.java) {
    companion object { val DEPTH_KEY = Any() }
    override fun serialize(value: Node, gen: JsonGenerator, sp: SerializerProvider) {
        val current = (sp.getAttribute(DEPTH_KEY) as? Int) ?: 0
        if (current > maxDepth) { gen.writeString("..."); return }
        sp.setAttribute(DEPTH_KEY, current + 1)
        gen.writeStartObject()
        gen.writeNumberField("id", value.id)
        gen.writeFieldName("children")
        gen.writeStartArray()
        value.children.forEach { DepthLimitedNodeSerializer(maxDepth).serialize(it, gen, sp) }
        gen.writeEndArray()
        gen.writeEndObject()
        sp.setAttribute(DEPTH_KEY, current)
    }
}
```

---

## Влияние сериализации на нагрузку: измерение времени, оптимизация объектов, уменьшение количества полей

Производительность — это цифры. Измеряйте размер JSON, время сериализации/десериализации и аллокации. Простые микробенчи на «реальных» DTO дадут интуицию, куда уходят миллисекунды. Затем оптимизируйте: уплощайте модели, избавляйтесь от лишних полей и вложенных коллекций.

Сокращение полей — самый дешёвый выигрыш. Часто фронту не нужны все детали сущности. Возьмите за правило делать «тонкие» DTO под сценарий, а не тащить доменную модель «как есть». Это улучшает и латентность, и стабильность клиентов при эволюции.

Следите за аллокациями в сериализаторах. Кастомные сериализаторы должны по возможности писать напрямую в `JsonGenerator`, не формируя промежуточные строки. Там, где нужна конверсия формата, пробуйте переиспользование буферов.

Etags, кэширование на уровне CDN и сервер-сайд кеш (например, Caffeine) снимают нагрузку с сериализации. Для часто запрашиваемых неизменяемых ресурсов кэш может давать двузначный кратный прирост пропускной способности.

Если вы используете WebFlux, помните о back-pressure. Слишком тяжёлые ответы могут занимать EventLoop и ухудшать латентность для соседних запросов. Stream/NDJSON или пагинация помогут сбалансировать поток.

Для бинарных тел (файлы, изображения) отдавайте `byte[]`/`Resource` и не пропускайте это через JSON. Это очевидно, но встречается часто, когда в JSON инлайнится base64-контент — так вы раздуваете ответы на 33% и тратите CPU впустую.

Не забывайте профилировать сервер под реальной нагрузкой APM-инструментами. Профиль покажет долю Jackson в процентах CPU. Это снимает спор «JSON точно не узкое место?» и направляет усилия туда, где ROI выше.

*Java (микробенч сериализации):*

```java
package com.example.s3.bench;

import com.fasterxml.jackson.databind.ObjectMapper;

public class Bench {
    public record Payload(long id, String name, String blob) {}
    public static void main(String[] args) throws Exception {
        var om = new ObjectMapper();
        var dto = new Payload(1, "Alice", "x".repeat(10000));
        long t = 0; for (int i=0;i<100;i++) t += om.writeValueAsBytes(dto).length;
        System.out.println("avgSize=" + (t/100));
    }
}
```

*Kotlin (аналог + уплощение DTO):*

```kotlin
package com.example.s3.bench

import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper

data class Full(val id: Long, val name: String, val details: String, val notes: List<String>)
data class Thin(val id: Long, val name: String) // тонкий DTO

fun main() {
    val om = jacksonObjectMapper()
    val full = Full(1, "Alice", "x".repeat(10000), List(100) { "n$it" })
    val thin = Thin(1, "Alice")
    println("full=${om.writeValueAsBytes(full).size} thin=${om.writeValueAsBytes(thin).size}")
}
```

---

## Бинарные форматы (например, Protobuf) как альтернатива для сокращения объёмов передачи

Когда JSON становится узким местом, на внутренних интерфейсах разумно перейти на бинарные форматы. Protobuf даёт компактность и скорость, а строгая схема снижает риск «тихих» несовместимостей. Минус — сложнее отладка и необходимость генерации кода.

Частая стратегия: наружу — JSON, внутрь — gRPC/Protobuf. Это позволяет сохранить DX для внешних команд и получить эффективность в микросервисной шине. Важно держать единый источник истины для модели (чаще — `.proto`), а REST-DTO генерировать/собирать из неё.

Разница в размере может быть кратной. На небольших сообщениях выигрыш особенно заметен в сумме: меньше трафика, меньше аллокаций при парсинге, больше Throughput. На больших сообщениях выигрыш есть, но нужно учитывать компрессию HTTP.

С эволюцией схемы в Protobuf проще: новые optional-поля «не ломают» старых клиентов. Но следите за нумерацией тегов, не переиспользуйте их, документируйте зарезервированные значения. Это дисциплина, которая окупается при долгоживущих интеграциях.

Для отладки добавьте в сервис вспомогательный эндпоинт: «скормить» бинарь и получить JSON по схеме. Это уменьшит трение в разработке и поможет SRE-команде при расследованиях инцидентов.

Если вы используете Kafka, альтернатива Protobuf — Avro с Schema Registry. Он лучше интегрирован в Kafka-экосистему, а tooling для «снятия дампа» и конвертации в JSON зрелый. Выбор часто определяется инфраструктурой.

Смешанные потоки возможны. Не бойтесь иметь часть событий в JSON (админ-журнал), а критичные «тяжёлые» — в Avro/Protobuf. Главное — дисциплина схем и мониторинг совместимости при релизах.

*Gradle Groovy (protobuf-плагин):*

```groovy
plugins { id 'com.google.protobuf' version '0.9.4' }
dependencies {
    implementation 'io.grpc:grpc-netty-shaded:1.66.0'
    implementation 'io.grpc:grpc-protobuf:1.66.0'
    implementation 'io.grpc:grpc-stub:1.66.0'
    implementation 'com.google.protobuf:protobuf-java:4.28.3'
}
protobuf { protoc { artifact = "com.google.protobuf:protoc:4.28.3" } }
```

*Gradle Kotlin:*

```kotlin
plugins { id("com.google.protobuf") version "0.9.4" }
dependencies {
    implementation("io.grpc:grpc-netty-shaded:1.66.0")
    implementation("io.grpc:grpc-protobuf:1.66.0")
    implementation("io.grpc:grpc-stub:1.66.0")
    implementation("com.google.protobuf:protobuf-java:4.28.3")
}
protobuf { protoc { artifact = "com.google.protobuf:protoc:4.28.3" } }
```

*`customer.proto`:*

```proto
syntax = "proto3";
package com.example.proto;
message Customer { int64 id = 1; string name = 2; string email = 3; }
```

*Java (сравнение размеров):*

```java
package com.example.s3.pb;

import com.example.proto.CustomerOuterClass.Customer;
import com.fasterxml.jackson.databind.ObjectMapper;

public class Compare {
    public static void main(String[] args) throws Exception {
        var j = new ObjectMapper();
        record Dto(long id, String name, String email) {}
        byte[] json = j.writeValueAsBytes(new Dto(1,"Alice","alice@example.com"));
        byte[] pb = Customer.newBuilder().setId(1).setName("Alice").setEmail("alice@example.com").build().toByteArray();
        System.out.println("json=" + json.length + " pb=" + pb.length);
    }
}
```

*Kotlin (аналог):*

```kotlin
package com.example.s3.pb

import com.example.proto.CustomerOuterClass.Customer
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper

data class Dto(val id: Long, val name: String, val email: String)

fun main() {
    val om = jacksonObjectMapper()
    val json = om.writeValueAsBytes(Dto(1, "Alice", "alice@example.com"))
    val pb = Customer.newBuilder().setId(1).setName("Alice").setEmail("alice@example.com").build().toByteArray()
    println("json=${json.size} pb=${pb.size}")
}
```

# 4. Обработка ошибок в REST API: концепция

## Ошибки как ресурсы: код состояния HTTP (4xx/5xx), описание причины и контекста ошибки

Ошибки в REST — это такие же ресурсы, как и успешные ответы: у них есть структура, смысл и контракт. Клиент не должен «угадывать», что пошло не так по обрывкам текста; он должен получать машинно-обрабатываемую сущность, в которой указаны причина, контекст и следствия. На практике это означает: единый формат тела ответа на ошибку, согласованные коды HTTP и стабильно заполняемые поля.

Границу между классами ошибок удобно проводить по семействам кодов: **4xx** означает, что проблема на стороне клиента (неверные данные, несуществующий ресурс, отсутствие прав), а **5xx** — что сервер не смог выполнить корректный запрос из-за своих внутренних проблем (падение БД, таймаут внешнего сервиса, баг). Это разделение важно для ретраев: 4xx обычно бесполезно повторять, 5xx — можно с разумной политикой повторов.

Каждая ошибка должна быть **самодостаточной**: заголовок (чтобы быстро понять тип), деталь (что именно не так), код состояния, указание «где это произошло» (URI ресурса или идентификатор операции), и, по возможности, ссылки на документацию. Это снимает нагрузку с поддержки и ускоряет реакцию клиента.

Контекст критичен: одно и то же «400 Bad Request» может означать десяток разных причин. Если у вас есть поле, где можно уточнить, какое именно поле в JSON не прошло валидацию, — отдавайте его. Если есть внутренний код причины (например, `ERR_AGREEMENT_CLOSED`), — включайте его, но **не раскрывайте чувствительные детали** (SQL, стек-трейсы).

Не пытайтесь «зашить» ошибки в 200 OK + «status":"ERROR"». Такой ответ ломает семантику HTTP: кэширование, прокси, политика ретраев, APM-агенты — все ожидают, что 2xx означает успех. Вы теряете совместимость и предсказуемость поведения инфраструктуры.

Важно, чтобы ошибки были **детерминированы** и согласованы между командами. Это достигается либо формальным стандартом (RFC 7807), либо внутренним гайдлайном, где оговорены поля, коды и сообщения. В CI можно завести проверку, что все исключения транслируются в корректные ответы.

И наконец, не забывайте о локализации и аудитах. Пользовательские сообщения могут требовать перевода, а внутренняя диагностика — нет. Разведите «сообщение для клиента» и «техническую деталь» по разным полям, чтобы не смешивать уровни.

**Java — примитивная ошибка как ресурс (ResponseEntity + ProblemDetail):**

```java
package com.example.errors.concept;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/concept")
public class ConceptController {
    @GetMapping("/orders/{id}")
    public ResponseEntity<?> getOrder(@PathVariable String id) {
        if ("0".equals(id)) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
            pd.setTitle("Order not found");
            pd.setDetail("Order with id=0 does not exist");
            pd.setInstance(java.net.URI.create("/api/concept/orders/0"));
            return ResponseEntity.status(HttpStatus.NOT_FOUND).body(pd);
        }
        return ResponseEntity.ok(new OrderDto(id));
    }
    public record OrderDto(String id) {}
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.concept

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/concept")
class ConceptController {
    @GetMapping("/orders/{id}")
    fun getOrder(@PathVariable id: String): ResponseEntity<Any> =
        if (id == "0") {
            val pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
                title = "Order not found"
                detail = "Order with id=0 does not exist"
                instance = java.net.URI.create("/api/concept/orders/0")
            }
            ResponseEntity.status(HttpStatus.NOT_FOUND).body(pd)
        } else {
            ResponseEntity.ok(OrderDto(id))
        }
}
data class OrderDto(val id: String)
```

---

## Формат ошибок: стандартные структуры (RFC 7807, ProblemDetails), семантика `title`, `status`, `detail`, `instance`

Стандарт **RFC 7807** («Problem Details for HTTP APIs») описывает минимальный, но достаточный каркас ответа на ошибку. Он не навязывает конкретных кодов, а лишь формализует поля: `type` (URI с описанием типа проблемы), `title` (краткое человекочитаемое название), `status` (HTTP-код), `detail` (подробности), `instance` (URI проблемного ресурса/запроса). Такой каркас легко расширять дополнительными полями.

Spring 6 / Spring Boot 3 добавили класс **`ProblemDetail`**, который прямо мапится на RFC 7807. Это существенно упрощает жизнь: вы можете создавать и заполнять объект и возвращать его в `ResponseEntity`, а Spring конвертирует его в корректный JSON. При этом можно добавлять произвольные свойства через `setProperty`.

Поле `type` не обязательно вести на реальный URL — достаточно стабильного URI, но **настоящая** страница с описанием типа ошибки повышает DX. Это может быть ваша документация, где для каждого кода указаны причины, советы и примеры решения.

Поле `title` — краткая константа («Validation failed», «Out of credit»), но не краткий пересказ `detail`. Оно помогает быстро классифицировать инциденты. Поле `detail` — конкретика по этой инстанции проблемы: какое поле не прошло валидацию, какой лимит превышен, какой ресурс отсутствует.

Поле `instance` удобно использовать для связывания логов и ошибок. Если сервер генерирует `requestId`/`traceId`, вы можете кодировать его в `instance` или добавлять отдельным свойством. Это значительно ускоряет трассировку.

Расширения — важная часть RFC 7807: добавьте `errors` для списка полей валидации, `code` для внутреннего короткого кода, `timestamp` для читаемой метки времени. Главное — держите расширения стабильными и документированными.

Единый формат с `ProblemDetail` упрощает работу фронтенда: он может унифицированно рендерить ошибки, подсвечивать поля и показывать ссылки на справку. Это делает поведение приложения предсказуемым и уменьшает «зоопарк» форматов.

**Java — формирование RFC 7807 с расширениями:**

```java
package com.example.errors.problem;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.time.Instant;

@RestController
@RequestMapping("/api/problems")
public class ProblemController {
    @GetMapping("/forbidden")
    public ProblemDetail forbidden() {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
        pd.setType(URI.create("https://docs.example.com/problems/forbidden"));
        pd.setTitle("Access denied");
        pd.setDetail("You lack the required role to access this resource.");
        pd.setInstance(URI.create("/api/problems/forbidden"));
        pd.setProperty("code", "ERR_FORBIDDEN");
        pd.setProperty("timestamp", Instant.now().toString());
        return pd;
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.problem

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*
import java.net.URI
import java.time.Instant

@RestController
@RequestMapping("/api/problems")
class ProblemController {
    @GetMapping("/forbidden")
    fun forbidden(): ProblemDetail = ProblemDetail.forStatus(HttpStatus.FORBIDDEN).apply {
        type = URI.create("https://docs.example.com/problems/forbidden")
        title = "Access denied"
        detail = "You lack the required role to access this resource."
        instance = URI.create("/api/problems/forbidden")
        setProperty("code", "ERR_FORBIDDEN")
        setProperty("timestamp", Instant.now().toString())
    }
}
```

---

## Не использовать 500 для ошибок клиента (например, 404 для несуществующих ресурсов, 400 для неверных данных)

Смешивание семантики — источник боли в продакшне. Когда клиентский промах (неверный JSON, отсутствующий ресурс, нарушение предикатов бизнес-правил) маскируется под 500, ломаются ретраи, мониторинг и алёрты. Операторы видят «красные» графики по 5xx и начинают искать поломку в инфраструктуре, хотя реальная проблема — в данных запроса.

Используйте точные коды: **400** для синтаксических проблем запроса и общего «плохого запроса», **404** для отсутствующего ресурса, **409** для конфликтов версий/состояний, **401/403** для аутентификации/авторизации, **422** («Unprocessable Content») для семантических ошибок в корректно сформированном запросе. Это помогает клиентам принимать верные решения.

Чёткая типизация ошибок влияет на UX. Если фронтенд знает, что пришёл 409, он может предложить пользователю обновить страницу или перезагрузить версию документа. Если пришёл 422 — подсветить поля и объяснить, что именно нужно поправить. Универсальный «500» этого не даёт.

С 5xx экономно: **500** — когда действительно «сломалось внутри», **502/503/504** — транзитные проблемы внешних зависимостей. Именно эти ответы должны включать механизмы ретраев со стороны клиента/оркестратора запросов и попадать в SLO/алёрты SRE.

Внутри сервиса старайтесь выкидывать разные типы исключений для разных классов ошибок. Это облегчает маппинг в коды и формат ответов. Например, `ResourceNotFoundException` → 404, `ValidationException` → 422, `AccessDeniedException` → 403, а вот необработанное `RuntimeException` → 500.

Ещё один нюанс — идемпотентность. Для идемпотентных методов (GET/PUT/DELETE) правильный код помогает кэшу/прокси не делать лишних повторов. Например, 404 для повторного DELETE уже удалённого ресурса может быть допустимым поведением, если контракт так определён.

И наконец, тестируйте коды статусов в автотестах. Контракт-тесты с MockMvc/WebTestClient на «ошибочные пути» — обязательная часть набора: это страховка от случайных регрессий при рефакторинге.

**Java — демонстрация разных кодов для разных ситуаций:**

```java
package com.example.errors.codes;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/codes")
public class CodesController {
    @GetMapping("/not-found")
    public ProblemDetail notFound() {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("Resource missing");
        pd.setDetail("Entity with given id is not present.");
        return pd;
    }

    @PostMapping("/validation")
    public ProblemDetail validation() {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setDetail("Amount must be positive.");
        return pd;
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.codes

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/codes")
class CodesController {
    @GetMapping("/not-found")
    fun notFound(): ProblemDetail = ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
        title = "Resource missing"
        detail = "Entity with given id is not present."
    }

    @PostMapping("/validation")
    fun validation(): ProblemDetail = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
        title = "Validation failed"
        detail = "Amount must be positive."
    }
}
```

---

## Перехват и нормализация ошибок через `@ControllerAdvice`, централизованное маппирование ошибок

Централизованный перехват ошибок — основа «чистого» API. Аннотация **`@ControllerAdvice`** позволяет объявить глобальный обработчик исключений для всех контроллеров. Это избавляет от дублирования и гарантирует, что каждое исключение будет трансформировано в корректный формат, а не превратится в HTML-страницу по умолчанию.

Обычно создают набор `@ExceptionHandler`-методов, каждый из которых принимает свой тип исключения и возвращает `ProblemDetail`/`ResponseEntity<ProblemDetail>`. Так вы настраиваете таблицу соответствий «исключение → HTTP-код/поля». Этот слой — единственная точка правды для ошибок.

Обработчик — хорошее место для обогащения контекстом: добавляйте `traceId`, `requestId`, ссылки на документацию, метку времени. Но не логируйте чувствительные поля запроса. Старайтесь, чтобы публичные детали были безопасными, а приватные — уходили в логи на уровень `ERROR`/`WARN`.

Нормализация ошибок подразумевает и работу с валидацией. Исключения типа `MethodArgumentNotValidException` и `BindException` стоит преобразовывать в 400/422 с массивом полей и сообщений. Это облегчает фронту подсветку ошибок.

Не забывайте о fallback-обработчике на `Exception` — он должен возвращать 500 с минимально нужной информацией и уникальным идентификатором, по которому команда SRE найдёт стек-трейс в логах. Это защищает пользователей от «сырого» исключения и снижает риски утечки.

Тестируйте обработчик отдельно: поднимайте контекст с MockMvc/WebTestClient и проверяйте и код, и тело. Это регресс-страховка: если кто-то выкинул «левое» исключение, вы быстро увидите, что ответ перестал соответствовать договорённости.

Наконец, разделяйте HTTP-обработку ошибок и бизнес-логику. Контроллеры и сервисы кидают исключения, а `@ControllerAdvice` превращает их в HTTP. Это упрощает сопровождение и позволяет переиспользовать бизнес-ошибки из других интерфейсов (например, из gRPC).

**Java — глобальный обработчик с ProblemDetail:**

```java
package com.example.errors.advice;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalErrorHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail onNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("Resource not found");
        pd.setDetail(ex.getMessage());
        pd.setProperty("timestamp", Instant.now().toString());
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setDetail("One or more fields are invalid.");
        var errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .collect(Collectors.toList());
        pd.setProperty("errors", errors);
        return pd;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail fallback(Exception ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal error");
        pd.setDetail("Unexpected error. Contact support with traceId.");
        return pd;
    }
}

class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String msg) { super(msg); }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.advice

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.time.Instant

@RestControllerAdvice
class GlobalErrorHandler {

    @ExceptionHandler(ResourceNotFoundException::class)
    fun onNotFound(ex: ResourceNotFoundException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
            title = "Resource not found"
            detail = ex.message
            setProperty("timestamp", Instant.now().toString())
        }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun onValidation(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            detail = "One or more fields are invalid."
            val errors = ex.bindingResult.fieldErrors.map { "${it.field}: ${it.defaultMessage}" }
            setProperty("errors", errors)
        }

    @ExceptionHandler(Exception::class)
    fun fallback(@Suppress("UNUSED_PARAMETER") ex: Exception): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR).apply {
            title = "Internal error"
            detail = "Unexpected error. Contact support with traceId."
        }
}

class ResourceNotFoundException(message: String) : RuntimeException(message)
```

---

# 5. Обработка ошибок в Spring: исключения и их обработка

## `@ExceptionHandler`: обработка ошибок на уровне контроллеров, возвращение стандартных ответов

Аннотация **`@ExceptionHandler`** может применяться прямо в контроллере — это локальный перехватчик исключений, актуальный только для данного класса. Такой подход полезен, когда у конкретного контроллера есть специфическая логика обработки, отличная от глобальной политики. Например, вы хотите скрыть детали ошибок интеграции для внешнего API, но оставить их в админском.

Локальные хендлеры не отменяют глобальные; Spring сначала ищет метод-перехватчик в текущем контроллере, затем — в `@ControllerAdvice`. Это позволяет гибко переопределять политику для отдельных маршрутов, не ломая общий контракт.

Возвращаемое значение `@ExceptionHandler` может быть `ProblemDetail`, `ResponseEntity<ProblemDetail>` или любым сериализуемым типом. Предпочтительнее — `ProblemDetail` ради унификации. Вы также можете выставлять заголовки (например, `Retry-After` для 429/503) и кэш-политику.

Полезно явно логировать исключения внутри хендлера, но следите за уровнем логирования и чувствительностью данных. Часто достаточно записать тип, код, основные детали и `traceId`, не включая тело запроса.

Хендлеры хорошо тестируются с MockMvc/WebTestClient. Вы можете вызвать endpoint, провоцирующий исключение, и убедиться, что возвращается ожидаемый статус и тело. Это повышает уверенность при рефакторингах.

Не злоупотребляйте локальными хендлерами. Если логика одинакова для всех контроллеров, уносите её в `@ControllerAdvice`. Дублирование обработчиков ведёт к рассинхронизации контрактов и ошибкам сопровождения.

И наконец, учитывайте реактивный стек (WebFlux): там `@ExceptionHandler` тоже работает, но исключения могут рождаться асинхронно в другом месте. Убедитесь, что вы перехватываете их на нужной стадии цепочки (в противном случае добавьте глобальный фильтр/обработчик).

**Java — локальный `@ExceptionHandler` в контроллере:**

```java
package com.example.errors.local;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/local")
public class LocalController {

    @GetMapping("/boom")
    public String boom() { throw new PartnerGatewayException("Gateway timeout"); }

    @ExceptionHandler(PartnerGatewayException.class)
    public ProblemDetail onGateway(PartnerGatewayException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_GATEWAY);
        pd.setTitle("Upstream error");
        pd.setDetail(ex.getMessage());
        return pd;
    }
}

class PartnerGatewayException extends RuntimeException {
    public PartnerGatewayException(String msg) { super(msg); }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.local

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/local")
class LocalController {
    @GetMapping("/boom")
    fun boom(): String = throw PartnerGatewayException("Gateway timeout")

    @ExceptionHandler(PartnerGatewayException::class)
    fun onGateway(ex: PartnerGatewayException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_GATEWAY).apply {
            title = "Upstream error"
            detail = ex.message
        }
}

class PartnerGatewayException(message: String) : RuntimeException(message)
```

---

## Глобальный обработчик через `@ControllerAdvice`: централизованное управление ошибками для всех контроллеров

`@ControllerAdvice` — инструмент системного масштаба: он позволяет собрать всю политику ошибок в одном месте и выдавать консистентные ответы. В больших системах обычно есть один-два таких класса: один для публичного REST, другой — для внутренних админских/технических эндпоинтов.

Глобальный обработчик — это также «прослойка безопасности»: тут вы гарантированно вычищаете стек-трейсы из ответа, маскируете PII, нормализуете сообщения и добавляете машиночитаемые поля (`code`, `errors`). При этом вы можете различать окружения (dev/prod) и включать дополнительную диагностику только в dev.

Важно не превращать обработчик в «свалку». Структурируйте методы по смыслу: «ресурс не найден», «валидация», «аутентификация/авторизация», «внешние зависимости», «прочее». Это облегчает сопровождение и делает код читаемым.

Глобальный хендлер — место для записи метрик по ошибкам: счётчики по типам ошибок/кодам, гистограммы латентности, доля 4xx/5xx. Эти показатели помогают отслеживать SLO и находить деградации. Интеграция с Micrometer делается в сервисном слое, но метки можно проставлять здесь.

Старайтесь избегать «проглатывания» исключений. Если вы отдаёте 500, убедитесь, что стек-трейс записан в логи с нужным уровнем, иначе расследование инцидента превратится в пытку. Хорошая практика — добавлять в ответ `errorId` и логировать его вместе с исключением.

Покройте обработчик контракт-тестами. Любое изменение — новый PR с тестами: статус, `title`, `detail`, формат массива ошибок, дополнительные поля. Это застрахует фронт и мобильные клиенты от сюрпризов.

И наконец, поддерживайте обратную совместимость. Если вы меняете структуру ошибок, делайте это как полноценную версию (например, новый `type` или новый медиатип `application/problem+json;v=2`), как вы поступаете с обычными API.

**Java — пример упорядоченного `@ControllerAdvice`:**

```java
package com.example.errors.global;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class ApiErrorHandler {

    @ExceptionHandler(AccessDeniedException.class)
    public ProblemDetail onAccessDenied(AccessDeniedException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
        pd.setTitle("Access denied");
        pd.setDetail("You do not have permission to perform this action.");
        return pd;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onInvalid(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setDetail("Invalid request payload.");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors()
                .stream().map(e -> e.getField() + ": " + e.getDefaultMessage()).toList());
        return pd;
    }

    @ExceptionHandler(Throwable.class)
    public ProblemDetail onAny(Throwable ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal error");
        pd.setDetail("Unexpected server error.");
        return pd;
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.global

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.security.access.AccessDeniedException
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ApiErrorHandler {

    @ExceptionHandler(AccessDeniedException::class)
    fun onAccessDenied(@Suppress("UNUSED_PARAMETER") ex: AccessDeniedException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.FORBIDDEN).apply {
            title = "Access denied"
            detail = "You do not have permission to perform this action."
        }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun onInvalid(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            detail = "Invalid request payload."
            setProperty("errors", ex.bindingResult.fieldErrors.map { "${it.field}: ${it.defaultMessage}" })
        }

    @ExceptionHandler(Throwable::class)
    fun onAny(@Suppress("UNUSED_PARAMETER") ex: Throwable): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR).apply {
            title = "Internal error"
            detail = "Unexpected server error."
        }
}
```

---

## Разделение ошибок: специфичные ошибки для бизнес-логики (`UserNotFoundException`), общие ошибки (например, неверный формат запроса)

Смешивание бизнес-ошибок и инфраструктурных — типичная причина хаоса. Бизнес-ошибки описывают правила предметной области: «пользователь заблокирован», «лимит превышен», «счёт закрыт». Инфраструктурные — про транспорт и парсинг: «неверный JSON», «таймаут». Разделив их, вы получаете ясную схему маппинга в коды и прогнозируемые реакции клиента.

Создайте иерархию исключений для домена: базовая `BusinessException` с полем `code` и наследники `UserNotFoundException`, `AccountClosedException`, `LimitExceededException`. В обработчике ошибок превращайте их в 404/409/422 с соответствующими `title` и дополнительными полями. Это лучше, чем «кидать RuntimeException с текстом».

Общие ошибки вроде «неверный формат запроса» и «нечитаемый JSON» пусть маппятся на 400/422 с понятным `detail`. Не подменяйте ими бизнес-ошибки: иначе клиенты не смогут различать «пользователь не существует» и «поле name не строка».

Внутренний короткий код причины (`code`) полезен для автоматизации: фронт может по нему выбирать текст локализованного сообщения, система аналитики — считать частоты. Держите словарь кодов в документации, привязывайте их к RFC 7807 `type`.

Разделение помогает и тестированию. Вы пишете отдельные наборы тестов: контракт-тесты для инфраструктурных ошибок (плохой JSON), и BDD-сценарии для бизнес-правил (запрещённая операция). Это ускоряет локализацию регрессий.

В логах бизнес-ошибки обычно пишутся на уровне `INFO/WARN` (ожидаемые сценарии), а инфраструктурные — `ERROR` (неожиданное поведение). Такое разведение предотвращает «шум» алёртов при нормальной работе системы.

И наконец, не забывайте про идемпотентность и конкарренси. Бизнес-конфликты версий (`OptimisticLockingFailureException`) — это 409, а не 500. Это важная подсказка для клиентов, что проблему можно решить повтором с обновлённой версией.

**Java — бизнес-исключения и маппинг:**

```java
package com.example.errors.domain;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDto get(@PathVariable long id) {
        if (id == 42) throw new UserNotFoundException("User 42 not found", "USR_NOT_FOUND");
        return new UserDto(id, "Alice");
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail onUserNotFound(UserNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("User not found");
        pd.setDetail(ex.getMessage());
        pd.setProperty("code", ex.code());
        return pd;
    }

    public record UserDto(long id, String name) {}
}

class UserNotFoundException extends RuntimeException {
    private final String code;
    public UserNotFoundException(String message, String code) { super(message); this.code = code; }
    public String code() { return code; }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.domain

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserController {
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long): UserDto =
        if (id == 42L) throw UserNotFoundException("User 42 not found", "USR_NOT_FOUND")
        else UserDto(id, "Alice")

    @ExceptionHandler(UserNotFoundException::class)
    fun onUserNotFound(ex: UserNotFoundException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
            title = "User not found"
            detail = ex.message
            setProperty("code", ex.code)
        }
}

data class UserDto(val id: Long, val name: String)

class UserNotFoundException(message: String, val code: String) : RuntimeException(message)
```

---

## Валидация: ошибки валидации через `BindingResult`, подробные сообщения об ошибках

Валидация входящих данных — первый рубеж качества. Аннотации Jakarta Validation (`@NotBlank`, `@Email`, `@Positive`) на DTO и `@Valid` на параметрах контроллера автоматически включают проверку и генерируют исключения при нарушениях. В MVC это чаще `MethodArgumentNotValidException`, в котором есть `BindingResult` с деталями по полям.

Иногда удобнее **не** бросать исключение, а обработать `BindingResult` в самом методе: это позволяет вернуть 422 с детализированным списком ошибок без поднятия исключения. Однако такой стиль требует дисциплины и может дублировать глобальный обработчик — выбирайте один подход для проекта.

Важно отдавать клиенту **структурированные** ошибки: список полей, к каждому — код и сообщение. Это позволяет фронту подсвечивать поля и строить локализованные сообщения. В `BindingResult` есть `FieldError`, откуда можно взять имя поля, код и текст.

Сообщения валидации лучше хранить в `messages.properties` и ссылаться на них через `{code}`. Так вы получите локализацию «из коробки» и стандартизацию текстов. Не шифруйте в коде «сырые» сообщения; это усложнит перевод и поддержку.

Старайтесь разводить валидацию на два слоя: синтаксическая (структура JSON, типы) и бизнес-валидация (правила домена). Первую обеспечивает Jakarta Validation и `@Valid`, вторую — сервисные валидаторы или бизнес-исключения. Это делает ответы предсказуемыми и улучшает читаемость кода.

При сложной валидации удобно заводить `@Validated` с группами и кастомными аннотациями-constraint’ами. Например, `@ValidInterval` на паре дат или `@ValidAmountCurrency` на сочетании суммы и валюты. Это повышает выразительность и повторное использование.

И наконец, не надо возвращать 500, если валидация не прошла. Это ожидаемый сценарий работы пользователя; ответ должен быть 400 или 422 с подробностями по полям. Это улучшает UX и снижает нагрузку на поддержку.

**Java — валидация через `BindingResult` без исключения:**

```java
package com.example.errors.validation;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;
import org.springframework.validation.BindingResult;

import java.util.List;

@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    @PostMapping
    public Object create(@RequestBody @Valid PaymentDto dto, BindingResult br) {
        if (br.hasErrors()) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setDetail("One or more fields are invalid.");
            List<String> errors = br.getFieldErrors().stream()
                    .map(e -> e.getField() + ": " + e.getDefaultMessage()).toList();
            pd.setProperty("errors", errors);
            return pd;
        }
        return dto;
    }

    public record PaymentDto(@NotBlank String id, @Positive long amount) {}
}
```

**Kotlin — аналог:**

```kotlin
package com.example.errors.validation

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.validation.BindingResult
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/payments")
class PaymentController {

    @PostMapping
    fun create(@RequestBody @Valid dto: PaymentDto, br: BindingResult): Any =
        if (br.hasErrors()) {
            ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"
                detail = "One or more fields are invalid."
                setProperty("errors", br.fieldErrors.map { "${it.field}: ${it.defaultMessage}" })
            }
        } else dto
}

data class PaymentDto(
    @field:NotBlank val id: String,
    @field:Positive val amount: Long
)
```
# 6. Стандарты ошибок: ProblemDetails и его реализация

## RFC 7807: структура ошибок, которую можно применять к REST API

RFC 7807 задаёт минимальный, но достаточный каркас для представления ошибки в HTTP-API. Идея проста: ошибка — это тоже ресурс, и её нужно сериализовать в стабильный объект с полями, понятными как человеку, так и машине. Спецификация предлагает пять базовых полей: `type`, `title`, `status`, `detail`, `instance`. Такой каркас избавляет команды от «зоопарка» форматов и делает обработку ошибок на клиентах тривиальной.

Поле `type` — это URI, указывающий на “тип проблемы”. Оно может вести на страницу документации, где описаны причины, последствия и шаги по исправлению. Даже если вы не публикуете публичную документацию, используйте стабильные URI: клиенты смогут опираться на них для ветвления логики (например, на фронтенде).

Поле `title` — краткое человекочитаемое название класса проблемы. Оно не подменяет `detail`, а помогает быстро классифицировать инцидент («Validation failed», «Access denied»). Старайтесь держать `title` константным для данного `type`, чтобы в логах и алертах не появлялись случайные вариации.

Поле `status` дублирует HTTP-код ошибки, чтобы тело было самодостаточным. Это полезно в системах, где тело ошибки попадает в очереди/хранилища без заголовков HTTP. Непротиворечивость между статусом в заголовке и в теле — обязательное требование для предсказуемости клиентов.

Поле `detail` — конкретика именно этой инстанции проблемы. Здесь уместно описывать, какие поля запроса неверны, какой лимит превышен, какую проверку не прошёл пользователь. Важно не путать его с “пользовательским” сообщением: `detail` — техническая деталь для разработчиков клиента; тексты для людей лучше выводить на стороне клиента, опираясь на коды.

Поле `instance` даёт ссылку на “экземпляр” проблемы. Как минимум это может быть URI запроса, на котором она возникла. Часто в него закладывают `traceId` или уникальный `errorId`, чтобы быстро находить соответствующие логи в APM/ELK. Это ускоряет расследования и снижает MTTR.

RFC 7807 допускает расширения: любые дополнительные свойства, не конфликтующие с базовыми. Обычно добавляют `code` (внутренний короткий код проблемы), `errors` (подробности по полям), `timestamp` (ISO-8601), `correlationId` и др. Главное — стандартизировать и задокументировать их, чтобы расширения не “плыли” от команды к команде.

*Java — минимальный endpoint, возвращающий RFC 7807:*

```java
package com.example.std.rfc;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.time.Instant;

@RestController
@RequestMapping("/api/rfc7807")
public class RfcController {
    @GetMapping("/forbidden")
    public ResponseEntity<ProblemDetail> forbidden() {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
        pd.setType(URI.create("https://docs.example.com/problems/forbidden"));
        pd.setTitle("Access denied");
        pd.setDetail("You lack the required role to access this resource.");
        pd.setInstance(URI.create("/api/rfc7807/forbidden"));
        pd.setProperty("code", "ERR_FORBIDDEN");
        pd.setProperty("timestamp", Instant.now().toString());
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(pd);
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.std.rfc

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.net.URI
import java.time.Instant

@RestController
@RequestMapping("/api/rfc7807")
class RfcController {
    @GetMapping("/forbidden")
    fun forbidden(): ResponseEntity<ProblemDetail> {
        val pd = ProblemDetail.forStatus(HttpStatus.FORBIDDEN).apply {
            type = URI.create("https://docs.example.com/problems/forbidden")
            title = "Access denied"
            detail = "You lack the required role to access this resource."
            instance = URI.create("/api/rfc7807/forbidden")
            setProperty("code", "ERR_FORBIDDEN")
            setProperty("timestamp", Instant.now().toString())
        }
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(pd)
    }
}
```

---

## Реализация через `ProblemDetail`: создание и кастомизация классов ошибок, расширение стандартных полей для бизнес-логики

Начиная с Spring Framework 6/Spring Boot 3 в стандартную библиотеку включён класс `org.springframework.http.ProblemDetail`. Это прямая материализация RFC 7807, поэтому он сразу сериализуется в `application/problem+json`. Благодаря ему не нужно поддерживать собственные обёртки — вы получаете готовый тип с `setTitle`, `setDetail`, `setInstance` и `setProperty` для расширений.

Типичный приём — фабрика ошибок или утилита-билдер, которая добавляет повторяющиеся поля: `timestamp`, `traceId/correlationId` из MDC, ссылку `type` на документацию. Это избавляет от копипасты и гарантирует единообразие ответов независимо от места возникновения исключения.

Для бизнес-ошибок удобно определять иерархию исключений, где у каждого наследника есть собственный “внутренний код” и рекомендуемый HTTP-статус. Глобальный обработчик (`@ControllerAdvice`) на основе типа исключения вызывает вашу фабрику и заполняет `ProblemDetail`. Такую схему легко расширять: новые типы ошибок добавляются через новые исключения.

`ProblemDetail` поддерживает произвольные свойства через `setProperty`. Этим удобно пользоваться для детализированной валидации (`errors: [{field, code, message}]`), гиперссылок на справку (`seeAlso`), информации для автоматического восстановления (например, `retryAfterMs`). Клиенты получают стабильный JSON, пригодный для автоматических реакций.

Если у вас несколько аудиторий (внешние клиенты и внутренние пользователи), держите один формат, но различайте “уровень доверия” через количество деталей. На проде публичному клиенту не нужно видеть внутренние идентификаторы исключений; логируйте их в серверных логах, а в `ProblemDetail` кладите безопасные `code` и `detail`.

С точки зрения сериализации `ProblemDetail` — обычный POJO. Его легко логировать “как есть” и писать в аудит. А если вы используете WebFlux, он столь же прозрачно превращается в JSON-ответ с нужным статусом: адаптеры Spring сделают это автоматически.

Ещё одна небольшая, но полезная деталь: `ProblemDetail` можно возвращать и из `@ExceptionHandler`, и из метода контроллера. Это позволяет не смешивать “штатные” и “ошибочные” ответы: контракт явно говорит клиенту, что ему пришёл объект-проблема.

*Java — фабрика и глобальный обработчик на `ProblemDetail`:*

```java
package com.example.std.problem;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.time.Instant;

class Problems {
    static ProblemDetail of(HttpStatus status, String type, String title, String detail, String code, String instance) {
        ProblemDetail pd = ProblemDetail.forStatus(status);
        pd.setType(URI.create(type));
        pd.setTitle(title);
        pd.setDetail(detail);
        pd.setInstance(URI.create(instance));
        pd.setProperty("code", code);
        pd.setProperty("timestamp", Instant.now().toString());
        return pd;
    }
}

@RestControllerAdvice
class GlobalAdvice {
    @ExceptionHandler(UserBannedException.class)
    ProblemDetail onUserBanned(UserBannedException ex) {
        return Problems.of(HttpStatus.FORBIDDEN,
                "https://docs.example.com/problems/user-banned",
                "User banned", ex.getMessage(), "USR_BANNED", "/api/*");
    }
}

class UserBannedException extends RuntimeException {
    public UserBannedException(String msg){ super(msg); }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.std.problem

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.net.URI
import java.time.Instant

object Problems {
    fun of(status: HttpStatus, type: String, title: String, detail: String, code: String, instance: String): ProblemDetail =
        ProblemDetail.forStatus(status).apply {
            this.type = URI.create(type)
            this.title = title
            this.detail = detail
            this.instance = URI.create(instance)
            setProperty("code", code)
            setProperty("timestamp", Instant.now().toString())
        }
}

@RestControllerAdvice
class GlobalAdvice {
    @ExceptionHandler(UserBannedException::class)
    fun onUserBanned(ex: UserBannedException): ProblemDetail =
        Problems.of(HttpStatus.FORBIDDEN,
            "https://docs.example.com/problems/user-banned",
            "User banned", ex.message ?: "Banned", "USR_BANNED", "/api/*")
}

class UserBannedException(message: String) : RuntimeException(message)
```

---

## Конвертация ошибок в JSON через `@ResponseStatus` и `@ExceptionHandler`

Аннотация `@ResponseStatus` позволяет “прибить” HTTP-статус к классу исключения. Это удобно для простых случаев, когда достаточно правильного кода и не требуется тело ответа. Однако в продакшне почти всегда нужен унифицированный JSON-формат ошибки, поэтому `@ResponseStatus` лучше использовать вместе с `@ExceptionHandler`.

`@ExceptionHandler` в `@ControllerAdvice` даёт полный контроль: вы выбираете статус, строите `ProblemDetail`, добавляете расширения. Такая централизованная точка маппинга превращает любую ошибку в предсказуемый RFC 7807, а не в HTML-страницу по умолчанию или пустой ответ.

Важно понимать приоритеты: если на исключении есть `@ResponseStatus`, Spring установит статус согласно аннотации даже при наличии хендлера. Это может вызывать неожиданности. Практика показывает, что лучше не смешивать: либо вы вообще не ставите `@ResponseStatus` и всё делаете в хендлерах, либо ставите только там, где не нужен body.

Ещё одна тонкость — “пузырящиеся” исключения. Если в глубине стека вы бросили `IllegalArgumentException`, а наверху нет хендлера, клиент получит 500. Поэтому покрытие `@ExceptionHandler` для базовых классов (`RuntimeException`) с fallback-ответом — обязательный элемент защиты.

В случае валидационных ошибок, формируемых инфраструктурой Spring (`MethodArgumentNotValidException`, `BindException`), не полагайтесь на `@ResponseStatus`. Эти исключения нужно перехватывать и конвертировать в 400/422 с детализированным списком ошибок в расширениях RFC 7807.

Не забывайте об аудите: хендлер — удобное место, чтобы логировать краткое описание инцидента вместе с `traceId` и внутренним `errorId`. В ответе клиенту оставьте только безопасные поля: `type`, `title`, `status`, `detail`, `instance`, `code`.

Если у вас несколько интерфейсов (например, REST и gRPC), старайтесь держать единую семантику ошибок. Для gRPC есть собственные коды (`Status`), но вы можете дублировать смысл ваших `code`/`type` в метаданных, чтобы не терять выразительность.

*Java — `@ResponseStatus` + `@ControllerAdvice` со сборкой ProblemDetail:*

```java
package com.example.std.status;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@ResponseStatus(HttpStatus.NOT_FOUND)
class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String msg) { super(msg); }
}

@RestControllerAdvice
class ErrorsAdvice {
    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail notFound(OrderNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("Order not found");
        pd.setDetail(ex.getMessage());
        pd.setProperty("code", "ORD_NOT_FOUND");
        return pd;
    }
}

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @GetMapping("/{id}")
    public String get(@PathVariable String id) {
        throw new OrderNotFoundException("Order " + id + " does not exist");
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.std.status

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.*

@ResponseStatus(HttpStatus.NOT_FOUND)
class OrderNotFoundException(message: String) : RuntimeException(message)

@RestControllerAdvice
class ErrorsAdvice {
    @ExceptionHandler(OrderNotFoundException::class)
    fun notFound(ex: OrderNotFoundException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
            title = "Order not found"
            detail = ex.message
            setProperty("code", "ORD_NOT_FOUND")
        }
}

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @GetMapping("/{id}")
    fun get(@PathVariable id: String): String =
        throw OrderNotFoundException("Order $id does not exist")
}
```

---

# 7. Ошибки валидации: обработка с помощью JSR-303 и Spring

## Использование `@Valid` и `@Validated` для валидации входящих данных

`@Valid` включает Jakarta Bean Validation на параметре метода контроллера или на бине. Когда Spring успешно десериализует тело запроса в DTO, он запускает валидатор, и при нарушениях бросает исключение (`MethodArgumentNotValidException` для MVC). Это даёт простую и декларативную проверку входов без ручной “if-логики”.

`@Validated` — похожая аннотация из Spring, добавляющая поддержку групп и метод-уровневой валидации (включая параметры методов сервисов и возвращаемые значения). Часто её ставят на класс сервиса, а `@Valid` — на параметры контроллера. Так вы разделяете техническую валидацию транспорта и бизнес-валидацию на уровне домена.

Для вложенных объектов работает каскадная валидация: отметьте поле вложенного DTO аннотацией `@Valid`, и его ограничения тоже будут проверены. Это полезно для сложных форм, где есть повторяющиеся блоки (адреса, контакты и т. п.). Без `@Valid` вложенные поля останутся “невидимыми” для валидатора.

Группы валидации (`Default`, свои группы) позволяют применять разные наборы правил в разных сценариях. Например, при создании ресурса одни поля обязательны, при обновлении — другие. Аннотации поддерживают `groups = {Create.class}`; `@Validated(Create.class)` активирует нужный набор.

Надёжный паттерн: на транспортных DTO — синтаксическая валидация (`@NotBlank`, `@Email`, `@Positive`), на сервисах — семантическая (`@Validated` + проверка существования/состояния). Ошибки первой группы должны маппиться на 400/422, второй — чаще на 409/403/404 в зависимости от правила.

Важно помнить о нотации аннотаций в Kotlin: используйте `@field:NotBlank`, чтобы аннотация висела на поле, а не на параметре конструктора. Иначе валидатор просто не увидит ограничение. В Java о таком не нужно заботиться, аннотация и так попадает на поле/аксессор.

Тестируйте валидацию как отдельный слой. Для контроллеров используйте MockMvc/WebTestClient с примерами “плохих” JSON-ов. Для сервисов — юнит-тесты, где валидатор активен через `@Validated`. Это гарантирует, что ошибки не “утекут” в доменную логику.

*Java — `@Valid` в контроллере и `@Validated` в сервисе:*

```java
package com.example.val.basic;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/customers")
class CustomerController {
    private final CustomerService service;
    CustomerController(CustomerService service){ this.service = service; }

    @PostMapping
    CustomerDto create(@RequestBody @Valid CustomerDto dto) { return service.create(dto); }
}

@Validated
@Service
class CustomerService {
    CustomerDto create(@Valid CustomerDto dto) { return dto; }
}

record CustomerDto(
        @NotBlank String name,
        @Email String email,
        @Positive long age
) {}
```

*Kotlin — аналог:*

```kotlin
package com.example.val.basic

import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import org.springframework.stereotype.Service
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/customers")
class CustomerController(private val service: CustomerService) {
    @PostMapping
    fun create(@RequestBody @Valid dto: CustomerDto): CustomerDto = service.create(dto)
}

@Validated
@Service
class CustomerService {
    fun create(@Valid dto: CustomerDto): CustomerDto = dto
}

data class CustomerDto(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Positive val age: Long
)
```

---

## Валидация на уровне бина, методов контроллеров, полей и параметров

Поля DTO — базовый уровень: ставим аннотации на поля/аксессоры, и валидатор проверяет их при десериализации. Это покрывает “формальные” требования к данным: пустые строки, формат email, минимальные длины, диапазоны чисел. Такие проверки быстры и дешёвы.

Биновый уровень (class-level constraints) полезен для инвариантов, зависящих от нескольких полей. Например, дата окончания должна быть после даты начала, валюта должна соответствовать стране и т. п. Это делается через аннотацию-контейнер и собственный валидатор, которому на вход приходит весь объект.

Метод-уровневая валидация работает в сервисах и контроллерах: вы можете ограничивать значения параметров методов (`@Min`, `@Pattern`) и проверять возвращаемые значения (`@NotNull` на `@ReturnValue`). Активируется она `@Validated` на классе и поддерживается прокси-механизмами Spring.

Параметры контроллера вне тела запроса — тоже кандидаты на валидацию. Аннотации (`@RequestParam`, `@PathVariable`) можно снабжать ограничениями, и Spring проверит их до выполнения метода. Это удобно для пагинации, сортировки, фильтров: плохие параметры не доходят до бизнес-кода.

Каскадная валидация на коллекциях и картах позволяет проверять элементы (`List<@Email String>`). В Java это работает через type-use аннотации, в Kotlin — через соответствующие позиции с `@field:` при необходимости. Это экономит десятки ручных циклов по коллекциям.

Для агрегатов доменной модели лучше выносить валидацию в фабрики/сервисы, а не лепить Jakarta-аннотации прямо на сущности JPA. Так вы избежите сюрпризов при ленивой загрузке/прокси и сохраните слой валидации независимым от ORM.

Наконец, помните, что валидация — это не защита от атак, а средство обеспечения корректности данных. От SQL-инъекций и XSS вас спасают параметризованные запросы, экранирование и CSP; Jakarta Validation решает другую задачу — входную корректность.

*Java — class-level constraint и валидация параметров:*

```java
package com.example.val.bean;

import jakarta.validation.*;
import jakarta.validation.constraints.Min;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.lang.annotation.*;
import java.time.LocalDate;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = IntervalValidator.class)
@interface ValidInterval { String message() default "end must be after start"; Class<?>[] groups() default {}; Class<? extends Payload>[] payload() default {}; }

class IntervalValidator implements ConstraintValidator<ValidInterval, Booking> {
    public boolean isValid(Booking b, ConstraintValidatorContext c) {
        return b == null || b.start().isBefore(b.end());
    }
}

@ValidInterval
record Booking(LocalDate start, LocalDate end) {}

@Validated
@RestController
@RequestMapping("/api/booking")
class BookingController {
    @GetMapping
    Booking get(@RequestParam @Min(1) int nights) {
        return new Booking(LocalDate.now(), LocalDate.now().plusDays(nights));
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.val.bean

import jakarta.validation.Constraint
import jakarta.validation.ConstraintValidator
import jakarta.validation.ConstraintValidatorContext
import jakarta.validation.Payload
import jakarta.validation.constraints.Min
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*
import java.time.LocalDate
import kotlin.reflect.KClass

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [IntervalValidator::class])
annotation class ValidInterval(
    val message: String = "end must be after start",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class IntervalValidator : ConstraintValidator<ValidInterval, Booking> {
    override fun isValid(value: Booking?, context: ConstraintValidatorContext): Boolean =
        value == null || value.start.isBefore(value.end)
}

@ValidInterval
data class Booking(val start: LocalDate, val end: LocalDate)

@Validated
@RestController
@RequestMapping("/api/booking")
class BookingController {
    @GetMapping
    fun get(@RequestParam @Min(1) nights: Int) =
        Booking(LocalDate.now(), LocalDate.now().plusDays(nights.toLong()))
}
```

---

## Интеграция с Jakarta Validation и кастомизация сообщений об ошибках

Spring Boot подключает Jakarta Validation через `spring-boot-starter-validation`. По умолчанию сообщения берутся из аннотаций, но вы можете локализовать и стандартизировать тексты в `messages.properties`. Ссылки вида `{customer.name.notBlank}` в аннотациях подставляются из файла сообщений, что упрощает поддержку и перевод.

Локализация активируется через `LocaleResolver`/`LocaleContextResolver`. Для большинства приложений достаточно стандартных настроек, где локаль берётся из заголовка `Accept-Language`. Главное — чтобы у вас были файлы `messages_xx.properties` для нужных языков и единый словарь кодов сообщений.

Кастомные аннотации-ограничения также поддерживают собственные ключи сообщений. В аннотации укажите `message = "{interval.invalid}"`, а в ресурсах добавьте строку `interval.invalid=Дата окончания должна быть позже даты начала`. Это делает сообщения консистентными и легко поддерживаемыми.

Иногда нужно динамически формировать детали (например, в сообщение подставить порог). В аннотации используйте плейсхолдеры, а в валидаторе — `ConstraintValidatorContext#buildConstraintViolationWithTemplate`, чтобы подставить конкретные числа. Это даёт богатые, но при этом локализуемые тексты.

Сообщения валидации не обязаны попадать “как есть” в публичный API. Часто в `ProblemDetail` кладут структурированные коды ошибок для полей, а фронтенд уже маппит их в тексты на клиентской стороне. Это позволяет гибко менять тексты без релизов бэкенда.

Если у вас много доменных правил, полезно завести единый реестр кодов ошибок и следовать префиксам (`USR_*`, `ORD_*`, `PAY_*`). Эти коды отображайте и в логах, и в ответах. Регулярность кодов делает аналитику ошибок (метрики/дашборды) гораздо понятнее.

Наконец, помните про тесты на сообщения. Даже если фронтенд маппит коды, полезно убедиться, что ключи существуют и подставляются корректно. Для этого пишите небольшие тесты валидаторов с заданной локалью.

*Java — конфигурация i18n и локализованные сообщения:*

```java
package com.example.val.i18n;

import jakarta.validation.constraints.NotBlank;
import org.springframework.context.annotation.Bean;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/i18n")
class I18nController {
    @PostMapping
    public User create(@RequestBody @Validated User u) { return u; }
}

record User(@NotBlank(message = "{user.name.notBlank}") String name) {}

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
package com.example.val.i18n

import jakarta.validation.constraints.NotBlank
import org.springframework.context.annotation.Bean
import org.springframework.context.support.ResourceBundleMessageSource
import org.springframework.validation.annotation.Validated
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/i18n")
class I18nController {
    @PostMapping
    fun create(@RequestBody @Validated u: User): User = u
}

data class User(@field:NotBlank(message = "{user.name.notBlank}") val name: String)

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
user.name.notBlank=Имя обязательно для заполнения
```

*`application.yml` (пример локали по умолчанию):*

```yaml
spring:
  mvc:
    locale: ru_RU
    locale-resolver: accept-header
```

---

## Обработка ошибок валидации через `BindingResult` и возвращение подробных сообщений с кодами ошибок

По умолчанию при нарушении валидации Spring бросает исключение, и его удобно перехватить глобальным `@ControllerAdvice`, преобразовав в RFC 7807 с массивом полей. Но иногда вы хотите гибко управлять телом ответа прямо в методе — для этого параметр `BindingResult` ставят рядом с валидируемым аргументом. Если он содержит ошибки, возвращайте 422 и структурированные детали.

Структура массива деталей обычно включает `field`, `code`, `message`, а иногда и `rejectedValue` (с осторожностью — не светите PII). `code` берите из `FieldError.getCode()` или собственного маппинга. Клиенты по коду стабильно отображают локализованный текст и подсвечивают поля формы.

Важный момент — разграничение 400 и 422. Когда JSON не читается вовсе (сломанный синтаксис), уместно 400. Когда JSON валиден, но противоречит правилам вашего API (пустые/некорректные значения полей), уместно 422. Это разделение улучшает DX клиентов и помогает правильно строить ретраи.

Глобальный обработчик для `MethodArgumentNotValidException` — базовая необходимость. В нём вы центролизованно нормализуете структуру ошибок и форматируете поле `errors` в ProblemDetail. Убедитесь, что имена полей совпадают с контрактом (например, используйте стратегию имён JSON, если отличны от имён Java-полей).

Для параметров запроса (`@RequestParam`, `@PathVariable`) валидатор также может бросить `ConstraintViolationException`. Обработайте его отдельно: в нарушениях хранятся пути вида `create.arg0.name`, их стоит преобразовать к “плоским” именам, понятным клиенту.

Старайтесь не смешивать ручную и автоматическую валидацию. Если вы используете `BindingResult` в методе, не забудьте про единообразие структуры ответов с тем, что делает глобальный хендлер — иначе клиенты получат два разных формата ошибок в зависимости от пути.

И наконец, покрывайте сценарии автотестами: “пустое имя”, “некорректный email”, “число вне диапазона”, “сломанный JSON”. Для каждого кейса проверяйте и статус, и тело ошибки. Это защищает вас от случайных регрессий при обновлении Spring/Jackson.

*Java — обработка через `BindingResult` и глобальный хендлер:*

```java
package com.example.val.binding;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;

import java.util.Map;

@RestController
@RequestMapping("/api/val")
class ValController {

    @PostMapping("/user")
    public Object create(@RequestBody @Valid UserDto dto, BindingResult br) {
        if (br.hasErrors()) {
            ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
            pd.setTitle("Validation failed");
            pd.setDetail("Request contains invalid fields.");
            pd.setProperty("errors", br.getFieldErrors().stream()
                    .map(e -> Map.of("field", e.getField(), "code", e.getCode(), "message", e.getDefaultMessage()))
                    .toList());
            return pd;
        }
        return dto;
    }
}

record UserDto(@NotBlank String name, @Email String email, @Min(18) int age) {}

@RestControllerAdvice
class ValidationAdvice {

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    ProblemDetail onTypeMismatch(MethodArgumentTypeMismatchException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Bad request");
        pd.setDetail("Parameter has wrong type: " + ex.getName());
        return pd;
    }

    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException.class)
    ProblemDetail onBodyInvalid(org.springframework.web.bind.MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setDetail("Invalid request payload.");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(e -> Map.of("field", e.getField(), "code", e.getCode(), "message", e.getDefaultMessage()))
                .toList());
        return pd;
    }
}
```

*Kotlin — аналог:*

```kotlin
package com.example.val.binding

import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.validation.BindingResult
import org.springframework.web.bind.annotation.*
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException

@RestController
@RequestMapping("/api/val")
class ValController {

    @PostMapping("/user")
    fun create(@RequestBody @Valid dto: UserDto, br: BindingResult): Any =
        if (br.hasErrors()) {
            ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
                title = "Validation failed"
                detail = "Request contains invalid fields."
                setProperty("errors", br.fieldErrors.map {
                    mapOf("field" to it.field, "code" to it.code, "message" to it.defaultMessage)
                })
            }
        } else dto
}

data class UserDto(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Min(18) val age: Int
)

@RestControllerAdvice
class ValidationAdvice {

    @ExceptionHandler(MethodArgumentTypeMismatchException::class)
    fun onTypeMismatch(ex: MethodArgumentTypeMismatchException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            title = "Bad request"
            detail = "Parameter has wrong type: ${ex.name}"
        }

    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException::class)
    fun onBodyInvalid(ex: org.springframework.web.bind.MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            detail = "Invalid request payload."
            setProperty("errors", ex.bindingResult.fieldErrors.map {
                mapOf("field" to it.field, "code" to it.code, "message" to it.defaultMessage)
            })
        }
}
```

# 8. Асинхронные ошибки и эксепшн-хандлинг

## Обработка ошибок в асинхронных сервисах: `Mono.error`, `Flux.error`, `@Async`-методы

В асинхронном коде ошибка не «бросается» в вызывающий стек, а превращается в **сигнал об ошибке**. В реактивной модели Project Reactor это терминальные сигналы `onError` в `Mono`/`Flux`. Когда вы пишете `Mono.error(new MyException())`, вы не кидаете исключение синхронно — вы создаёте источник, который при подписке завершится ошибкой. Это важно для понимания: обработчик ошибок должен находиться **в реактивной цепочке**.

В `Flux` аналогично: `Flux.error` создаёт поток, который сразу завершится ошибкой, а `Flux.concat(a, Flux.error(ex))` завершит ошибкой после элементов из `a`. Это позволяет программировать ошибки композиционно: подмешивать их в поток, эмулировать сбои в тестах и явно указывать моменты, где нужно прервать обработку.

В мире Spring `@Async` — другой механизм асинхронности, основанный на пулах потоков и прокси. Если метод помечен `@Async`, он выполняется в «чужом» потоке. Если он возвращает `CompletableFuture`/`ListenableFuture`/`Mono`/`Flux`, то исключения попадут в соответствующие механизмы обработки; если же метод `void`, то непойманные исключения уйдут в **`AsyncUncaughtExceptionHandler`** — его нужно настроить явно.

Комбинировать Reactor и `@Async` обычно не нужно: Reactor уже асинхронный и неблокирующий. Но `@Async` полезен в MVC-приложениях или когда вы вызываете **блокирующий** код и хотите вынести его на отдельный пул. Помните, что возвращать стоит «контейнер», который умеет нести ошибку (`CompletableFuture`, `Mono`), чтобы её можно было корректно обработать на уровне контроллера.

Особое внимание — **пропагации контекста** (MDC, безопасность). При переходе на другой поток (через `@Async` или `publishOn`) контекст не переносится автоматически, поэтому логи могут потерять `traceId`. Для `@Async` решается `TaskDecorator`-ом, для Reactor — через `Context` и специальные хуки (ниже будет пример).

Проверяйте ошибки на границе слоя: сервис пусть возвращает `Mono<T>`/`CompletableFuture<T>` и не преобразует ошибки «в текст». Преобразование в формат API (ProblemDetails) выполняйте в контроллере или глобальном хендлере. Это разделит ответственность и облегчит тестирование.

В тестах используйте `StepVerifier` (Reactor) и `CompletableFuture` API, чтобы проверять **тип** и **сообщение** ошибки, а не только факт падения. Так вы зафиксируете контракт ошибок.

**Gradle (добавьте WebFlux/AOP для примеров):**
*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-aop'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-aop")
}
```

**Java — `Mono.error` и `@Async` с обработчиком:**

```java
package com.example.async.basic;

import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;
import org.springframework.context.annotation.*;
import org.springframework.scheduling.annotation.*;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

import java.lang.reflect.Method;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

@Configuration
@EnableAsync
class AsyncCfg implements AsyncConfigurer {
    @Override public Executor getAsyncExecutor() { return Executors.newFixedThreadPool(4); }
    @Override public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (Throwable ex, Method method, Object... params) ->
                System.err.println("ASYNC UNCAUGHT in " + method.getName() + ": " + ex.getMessage());
    }
}

@Service
class AsyncService {
    public Mono<String> reactiveCall(boolean fail) {
        return fail ? Mono.error(new IllegalStateException("reactive boom")) : Mono.just("ok");
    }

    @Async
    public CompletableFuture<String> asyncCall(boolean fail) {
        if (fail) throw new IllegalArgumentException("async boom"); // попадёт в AsyncUncaughtExceptionHandler, если никто не слушает
        return CompletableFuture.completedFuture("ok");
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.async.basic

import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.annotation.Async
import org.springframework.scheduling.annotation.AsyncConfigurer
import org.springframework.scheduling.annotation.EnableAsync
import org.springframework.stereotype.Service
import reactor.core.publisher.Mono
import java.lang.reflect.Method
import java.util.concurrent.CompletableFuture
import java.util.concurrent.Executor
import java.util.concurrent.Executors

@Configuration
@EnableAsync
class AsyncCfg : AsyncConfigurer {
    override fun getAsyncExecutor(): Executor = Executors.newFixedThreadPool(4)
    override fun getAsyncUncaughtExceptionHandler(): AsyncUncaughtExceptionHandler =
        AsyncUncaughtExceptionHandler { ex: Throwable, m: Method, _: Array<out Any> ->
            System.err.println("ASYNC UNCAUGHT in ${m.name}: ${ex.message}")
        }
}

@Service
class AsyncService {
    fun reactiveCall(fail: Boolean): Mono<String> =
        if (fail) Mono.error(IllegalStateException("reactive boom")) else Mono.just("ok")

    @Async
    fun asyncCall(fail: Boolean): CompletableFuture<String> =
        if (fail) throw IllegalArgumentException("async boom")
        else CompletableFuture.completedFuture("ok")
}
```

---

## Реактивные исключения и управление потоком ошибок в WebFlux

В реактивных цепочках ошибки — такие же «первоклассные» события, как и данные. Управлять ими нужно операторами: `onErrorResume` (замена на альтернативный publisher), `onErrorReturn` (быстрое значение по умолчанию), `onErrorMap` (переклассификация), `doOnError` (побочный эффект), `retryWhen` (повторы). Это позволяет выразить **политику** обработки там, где она принадлежит — рядом с бизнес-кодом.

`onErrorResume` полезен для graceful-degradation: если зависимость временно недоступна, можно вернуть кэш/заглушку. Важно не «проглатывать» все ошибки подряд — ограничивайте по типам. `onErrorResume(MyDependencyException.class, e -> fallback())` оставляет остальное наверх.

`onErrorMap` — инструмент нормализации исключений: вы превращаете любые внутренние ошибки в одно из «доменных» исключений, которые затем глобальный хендлер мапит в ProblemDetails. Это делает поведение предсказуемым и устраняет «рандомные» классы исключений наверху.

`doOnError` не меняет поток, а лишь добавляет сайд-эффект — чаще логирование. Помните, что `doOnError` сработает только когда ошибка действительно пройдёт до этого оператора, и не сработает, если вы её перехватили ранним `onErrorResume`.

В контроллерах WebFlux вы возвращаете `Mono<T>`/`Flux<T>`. Исключения, «вышедшие» из цепочки, перехватит ваш `@ControllerAdvice`. Следите за тем, чтобы **не оборачивать исключения в данные** (например, `Mono.just(ProblemDetail)` на успешный статус) — возвращайте их как ошибки, пусть Spring сам сформирует ответ со статусом.

Про диагностирование: используйте `checkpoint("label")` — он делает стек трассировки понятнее, добавляя «маяки» в stacktrace. В продакшне это экономит время расследований «где именно в лямбде случилось NPE».

Отдавая потоки на boundary (HTTP-ответ), помните о back-pressure. Если вы в `Flux` обрабатываете «бесконечный» источник, убедитесь, что ошибки (и завершение) корректно сигналятся клиенту, иначе соединение может «зависнуть» с подвешенной подпиской.

**Java — типовые операторы обработки в контроллере WebFlux:**

```java
package com.example.async.webflux;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;

import java.time.Duration;

@RestController
@RequestMapping("/api/reactive")
class ReactiveController {
    private final ExternalClient client;
    ReactiveController(ExternalClient client) { this.client = client; }

    @GetMapping("/profile/{id}")
    Mono<Profile> profile(@PathVariable long id) {
        return client.fetchProfile(id)
                .checkpoint("fetchProfile")
                .retryWhen(Retry.backoff(2, Duration.ofMillis(100))
                        .filter(ex -> ex instanceof TemporaryFailure))
                .onErrorMap(e -> e instanceof NotFound ? new ResourceNotFound() : e)
                .onErrorResume(ResourceNotFound.class, e -> Mono.error(e)) // передадим в глобальный хендлер
                .doOnError(e -> System.err.println("reactive error: " + e.getMessage()));
    }
}

record Profile(long id, String name) {}
class TemporaryFailure extends RuntimeException {}
class NotFound extends RuntimeException {}
class ResourceNotFound extends RuntimeException {}
interface ExternalClient { Mono<Profile> fetchProfile(long id); }
```

**Kotlin — аналог:**

```kotlin
package com.example.async.webflux

import org.springframework.web.bind.annotation.*
import reactor.core.publisher.Mono
import reactor.util.retry.Retry
import java.time.Duration

@RestController
@RequestMapping("/api/reactive")
class ReactiveController(private val client: ExternalClient) {

    @GetMapping("/profile/{id}")
    fun profile(@PathVariable id: Long): Mono<Profile> =
        client.fetchProfile(id)
            .checkpoint("fetchProfile")
            .retryWhen(Retry.backoff(2, Duration.ofMillis(100)).filter { it is TemporaryFailure })
            .onErrorMap { if (it is NotFound) ResourceNotFound() else it }
            .onErrorResume(ResourceNotFound::class.java) { Mono.error(it) }
            .doOnError { System.err.println("reactive error: ${it.message}") }
}

data class Profile(val id: Long, val name: String)
class TemporaryFailure : RuntimeException()
class NotFound : RuntimeException()
class ResourceNotFound : RuntimeException()
fun interface ExternalClient { fun fetchProfile(id: Long): Mono<Profile> }
```

---

## Проблемы с асинхронным выводом: трассировка контекста ошибок, возможность потери данных

Асинхронность усложняет **трассировку**: стек «разрывается», исключение возникает в другом потоке/операторе, а MDC-контекст (например, `traceId`) теряется. Из-за этого логи без корреляции превращаются в пазл. Первая защита — везде, где входите в ваш код снаружи (HTTP, consumer), создавайте/получайте `traceId` и сохраняйте его в контексте.

В MVC это делается `OncePerRequestFilter`, который кладёт `traceId` в `MDC`. Проблема в том, что при `@Async` таски выполняются в другом потоке без MDC. Решение — `TaskDecorator`, который при сабмите задачи копирует текущий MDC в «дочерний» поток, и обратно очищает его после выполнения. Это малоизвестная, но очень полезная деталь.

В WebFlux MDC по умолчанию не работает, потому что Reactor сам управляет потоками. Здесь используют Reactor **Context** как «носитель» метаданных, а затем при каждом сигнале копируют нужные ключи из `Context` в MDC (и обратно удаляют). Так достигается связность логов без жёсткой привязки к потокам.

Ещё одна ловушка — «тихая потеря» данных при ошибках: если вы пишете в ответ кусками (server-sent events, streaming) и случается ошибка, часть уже отправленных данных не откатить. Клиенту нужно явно сообщить о прерывании, а на сервере — закрыть ресурсы. Старайтесь по возможности отправлять атомарные структуры либо версионируйте поток так, чтобы клиент мог восстановиться.

Обрабатывайте таймауты. Если асинхронная операция зависла, фронт увидит обрыв соединения без полезной диагностики, а у вас в логах не будет «островков» для расследования. Добавляйте `timeout`/`take` и фиксируйте в логах сам факт таймаута с контекстом `traceId` и именем операции.

Не смешивайте «сервисные» исключения и «транспортные». Пусть `TimeoutException`/`CancellationException` обрабатываются ближе к boundary, где вы решаете — вернуть 504/408 или выполнить fallback. Бизнес-исключения («нет прав», «ресурс закрыт») должны идти своим маршрутом, описанным в хендлере.

Наконец, учтите, что «файр-энд-форгет» (`@Async void`) — опасный режим: вызывающий код не видит ошибки. Либо возвращайте `CompletableFuture` и обрабатывайте ошибки через `handle/exceptionally`, либо настройте `AsyncUncaughtExceptionHandler` и покрывайте такие методы алертингом/метриками.

**Java — перенос MDC в `@Async` и `traceId` в MVC:**

```java
package com.example.async.context;

import org.slf4j.MDC;
import org.springframework.context.annotation.*;
import org.springframework.core.task.TaskDecorator;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.web.filter.OncePerRequestFilter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.http.*;
import java.io.IOException;
import java.util.Map;
import java.util.UUID;

@Configuration
@EnableAsync
class AsyncMdcConfig {

    @Bean
    public ThreadPoolTaskExecutor mdcExecutor() {
        ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(4);
        ex.setTaskDecorator(mdcTaskDecorator());
        ex.initialize();
        return ex;
    }

    @Bean
    public TaskDecorator mdcTaskDecorator() {
        return runnable -> {
            Map<String, String> contextMap = MDC.getCopyOfContextMap();
            return () -> {
                Map<String, String> old = MDC.getCopyOfContextMap();
                if (contextMap != null) MDC.setContextMap(contextMap);
                try { runnable.run(); }
                finally { if (old != null) MDC.setContextMap(old); else MDC.clear(); }
            };
        };
    }

    @Bean
    public OncePerRequestFilter requestIdFilter() {
        return new OncePerRequestFilter() {
            @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
                    throws ServletException, IOException {
                String traceId = req.getHeader("X-Request-Id");
                if (traceId == null || traceId.isBlank()) traceId = UUID.randomUUID().toString();
                MDC.put("traceId", traceId);
                try { res.setHeader("X-Request-Id", traceId); chain.doFilter(req, res); }
                finally { MDC.remove("traceId"); }
            }
        };
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.async.context

import org.slf4j.MDC
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.task.TaskDecorator
import org.springframework.scheduling.annotation.EnableAsync
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import org.springframework.web.filter.OncePerRequestFilter
import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import java.io.IOException
import java.util.UUID

@Configuration
@EnableAsync
class AsyncMdcConfig {

    @Bean
    fun mdcExecutor() = ThreadPoolTaskExecutor().apply {
        corePoolSize = 4
        setTaskDecorator(mdcTaskDecorator())
        initialize()
    }

    @Bean
    fun mdcTaskDecorator(): TaskDecorator = TaskDecorator { runnable ->
        val contextMap = MDC.getCopyOfContextMap()
        Runnable {
            val old = MDC.getCopyOfContextMap()
            if (contextMap != null) MDC.setContextMap(contextMap)
            try { runnable.run() } finally { if (old != null) MDC.setContextMap(old) else MDC.clear() }
        }
    }

    @Bean
    fun requestIdFilter() = object : OncePerRequestFilter() {
        @Throws(ServletException::class, IOException::class)
        override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
            var traceId = req.getHeader("X-Request-Id")
            if (traceId.isNullOrBlank()) traceId = UUID.randomUUID().toString()
            MDC.put("traceId", traceId)
            try { res.setHeader("X-Request-Id", traceId); chain.doFilter(req, res) }
            finally { MDC.remove("traceId") }
        }
    }
}
```

---

## Шаблоны обработки: повторные попытки, отказ по времени, fallback-стратегии

Повторы (retry) — полезный, но опасный инструмент. Применяйте **селективно**: ретрайте только транзитные ошибки (сетевые сбои, 5xx от внешнего сервиса), ограничивайте количество попыток и используйте **экспоненциальную задержку** с джиттером, чтобы не усугублять перегрузку зависимостей. Для Reactor есть `retryWhen(Retry.backoff(...))`.

Таймауты — обязательны на всех внешних вызовах. Без них «подвисания» будут съедать потоки/коннекты, а пользователи — получать обрывы без объяснений. В Reactor — оператор `timeout`, в HTTP-клиенте (WebClient) — таймауты на коннект/чтение. На уровне API это обычно 504/408 и чёткое `ProblemDetail`.

Fallback — «план Б», если зависимость недоступна: кэш, деградированная функциональность, более древние данные. В Reactor — `onErrorResume`/`switchIfEmpty`. Никогда не возвращайте «тихий успех» вместо ошибки, если это нарушает инварианты домена — лучше честно сообщить о деградации.

Идемпотентность и дедупликация: при повторах есть риск «дублирования эффекта». Для POST используйте идемпотентные ключи (Idempotency-Key), а для команд — версионирование/условные обновления. Иначе ретраи могут повредить данные.

Circuit Breaker — рациональное дополнение к retry/timeout. Он «коротит» быстрыми отказами, когда зависимость заведомо «падает», и постепенно «пробует» её снова. В реактивном стеке удобно использовать Resilience4j (`resilience4j-reactor`), но даже без него вы можете построить простой «ручной» предохранитель.

Наблюдаемость критична: метрики по ретраям, таймаутам и фоллбэкам показывают, где система реально страдает. Снимайте гистограммы латентности, счётчики ошибок по типам, и используйте `checkpoint`/`doOnError` для точной диагностики.

Разводите ответственность: где-то ретраи уместнее на уровне клиента (WebClient), где-то — в бизнес-сервисе, который лучше знает семантику ошибки. Убедитесь, что они не дублируются (двойной retry) — это классическая «бомба» под зависимость.

**Java — retry/timeout/fallback в Reactor:**

```java
package com.example.async.patterns;

import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;

import java.time.Duration;

@Service
class PatternsService {
    Mono<String> callWithPolicies(RemoteApi api) {
        return api.get()
                .timeout(Duration.ofSeconds(2))
                .retryWhen(Retry.backoff(3, Duration.ofMillis(200))
                        .filter(ex -> ex instanceof TransientError))
                .onErrorResume(TimeoutException.class, e -> Mono.just("stale-cache"))
                .onErrorResume(TransientError.class, e -> Mono.just("fallback"));
    }
    interface RemoteApi { Mono<String> get(); }
    static class TransientError extends RuntimeException {}
    static class TimeoutException extends RuntimeException {}
}
```

**Kotlin — аналог:**

```kotlin
package com.example.async.patterns

import org.springframework.stereotype.Service
import reactor.core.publisher.Mono
import reactor.util.retry.Retry
import java.time.Duration

@Service
class PatternsService {
    fun callWithPolicies(api: RemoteApi): Mono<String> =
        api.get()
            .timeout(Duration.ofSeconds(2))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(200)).filter { it is TransientError })
            .onErrorResume(TimeoutException::class.java) { Mono.just("stale-cache") }
            .onErrorResume(TransientError::class.java) { Mono.just("fallback") }
}

fun interface RemoteApi { fun get(): Mono<String> }
class TransientError : RuntimeException()
class TimeoutException : RuntimeException()
```

---

# 9. Логирование ошибок и трассировка

## Логирование ошибок с контекстом: `log.error("Ошибка с данными клиента: {}", customerId, e)`

Хорошие логи — это **структурированная история** о том, что произошло. Ошибки должны логироваться с контекстом: кто (пользователь/сервис), что (операция/ресурс), когда (timestamp/latency), где (endpoint), и почему (исключение/сообщение). Используйте параметризованные сообщения SLF4J, а не конкатенацию строк: это и быстрее, и безопаснее.

Логируйте исключения там, где вы **принимаете решение**, что с ними делать (обычно — в глобальном обработчике или на границе слоя). Не логируйте одно и то же `Throwable` несколько раз на пути вверх — получите «лог-шум» и ложные алёрты. Внутри цепочки используйте `debug/trace` и `doOnError`, но без стека, если его поднимут наверху.

Добавляйте ключевые идентификаторы: `traceId`, `requestId`, `customerId`, `orderId`. Это даёт поисковую основу для расследований. Передавайте их через MDC или Reactor Context, чтобы каждый лог-запись автоматически включала эти поля.

Чувствительные данные (PII) маскируйте. Никогда не логируйте пароли, токены, номера карт и т. п. Если важно увидеть фрагмент, маскируйте часть значения (первые/последние символы). Фильтры для `logback` способны удалять/подменять запрещённые шаблоны.

Стандартизируйте уровни: ожидаемые бизнес-исключения — `WARN` (или даже `INFO`), неожиданные ошибки — `ERROR`. Согласуйте это с алёртингом: не надо «кричать» на **все** 4xx-ошибки пользователей, иначе вы не увидите настоящих проблем.

Структурируйте логи: JSON-формат (через Logback encoder) упрощает анализ в ELK/OpenSearch. Если остаётесь на текстовом формате, настраивайте шаблон с MDC-полями и выделяйте ошибку в конце, чтобы парсеры могли её извлечь.

И наконец, договоритесь о **политике ретенции** и PII-комплаенсе: как долго храним логи, какие поля удаляем, какие окружения «обрезаны». Это снизит риски и упростит ответы на вопросы аудита.

**Java — пример логирования с контекстом и исключением:**

```java
package com.example.logging.basic;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogDemo {
    private static final Logger log = LoggerFactory.getLogger(LogDemo.class);

    public static void main(String[] args) {
        String customerId = "c-123";
        try {
            throw new IllegalStateException("boom");
        } catch (Exception e) {
            log.error("Ошибка при обработке клиента {} в операции {}", customerId, "create-order", e);
        }
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.logging.basic

import org.slf4j.LoggerFactory

fun main() {
    val log = LoggerFactory.getLogger("LogDemo")
    val customerId = "c-123"
    try {
        throw IllegalStateException("boom")
    } catch (e: Exception) {
        log.error("Ошибка при обработке клиента {} в операции {}", customerId, "create-order", e)
    }
}
```

*Пример `logback-spring.xml` (добавляет traceId из MDC в шаблон):*

```xml
<configuration>
  <property name="pattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} traceId=%X{traceId} - %msg%n%ex"/>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder><pattern>${pattern}</pattern></encoder>
  </appender>
  <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

---

## Важность сохранения контекста: traceId, requestId, временные метки для отслеживания цепочки ошибок

Без корреляции логи превращаются в набор несвязанных строк. `traceId`/`requestId` позволяют связать все записи одной операции: от приёма запроса до ответа, включая вызовы зависимостей. Это облегчает как ручное расследование, так и автоматическую трассировку (Micrometer Tracing/Zipkin/Tempo).

В MVC используйте фильтр `OncePerRequestFilter`: он проверяет заголовок `X-Request-Id`, создаёт его при отсутствии, кладёт в MDC и возвращает клиенту. Так клиенты могут присылать свой `requestId`, а вы — продолжать его путь через все сервисы. Стабильные имена заголовков и поля в логах — важная часть договора с внешними системами.

В WebFlux роль фильтра выполняет `WebFilter`. Дополнительно придётся проложить мост в Reactor Context, чтобы `traceId` «жил» не только в MDC, но и в реактивной цепочке. Иначе при переключении потоков вы его потеряете. В конце запроса `traceId` нужно очистить, чтобы не протекал в следующий запрос на том же потоке.

Добавляйте временные метки и длительности. Даже если у вас есть метрики, «сколько заняла операция» в логе рядом с ошибкой иногда быстрее ориентирует в проблеме. Стандартизируйте формат (`ms`, целые значения) и порядок полей.

Клиентам полезно возвращать `X-Request-Id` в ответе. Тогда, видя ошибку в UI, они смогут передать этот id в поддержку, и вы быстро найдёте нужные логи. Это особенно ценно при массовых системах с миллионами записей.

Согласуйте контракт по переносимым контекстам между сервисами (B3, W3C TraceContext, «собственные» заголовки). Если вы используете Micrometer Tracing/Brave, многие вещи сделают библиотеки; но даже без них базовый `X-Request-Id`+MDC — уже огромный шаг вперёд.

Не храните PII в контексте. В MDC следует класть только безопасные короткие идентификаторы (uuid, короткие коды). Всё остальное — потенциальный риск утечки через логи.

**Java — фильтр и WebFilter с ответным заголовком:**

```java
package com.example.logging.context;

import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.*;
import reactor.core.publisher.Mono;

import java.io.IOException;
import java.util.UUID;

@Component
class RequestIdFilter extends OncePerRequestFilter {
    @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String id = req.getHeader("X-Request-Id");
        if (id == null || id.isBlank()) id = UUID.randomUUID().toString();
        MDC.put("traceId", id);
        try { res.setHeader("X-Request-Id", id); chain.doFilter(req, res); }
        finally { MDC.remove("traceId"); }
    }
}

@Component
class ReactiveRequestIdFilter implements WebFilter {
    @Override public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String id = exchange.getRequest().getHeaders().getFirst("X-Request-Id");
        if (id == null || id.isBlank()) id = UUID.randomUUID().toString();
        exchange.getResponse().getHeaders().set("X-Request-Id", id);
        return chain.filter(exchange).contextWrite(ctx -> ctx.put("traceId", id));
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.logging.context

import org.slf4j.MDC
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import reactor.core.publisher.Mono
import java.io.IOException
import java.util.UUID

@Component
class RequestIdFilter : OncePerRequestFilter() {
    @Throws(ServletException::class, IOException::class)
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        var id = req.getHeader("X-Request-Id")
        if (id.isNullOrBlank()) id = UUID.randomUUID().toString()
        MDC.put("traceId", id)
        try { res.setHeader("X-Request-Id", id); chain.doFilter(req, res) }
        finally { MDC.remove("traceId") }
    }
}

@Component
class ReactiveRequestIdFilter : WebFilter {
    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        var id = exchange.request.headers.getFirst("X-Request-Id")
        if (id.isNullOrBlank()) id = UUID.randomUUID().toString()
        exchange.response.headers["X-Request-Id"] = id
        return chain.filter(exchange).contextWrite { it.put("traceId", id) }
    }
}
```

---

## Ошибки и исключения в логах: подробности исключения, стеки вызова

Записывать стек-трейс нужно **ровно один раз** — там, где вы «завернули» исключение в HTTP-ответ или в ответ очереди. Это создаёт единое место диагностики и не засоряет логи дубликатами. Все промежуточные уровни могут дополнить контекст (id, попытки ретрая), но не обязаны «сыпать» ERROR со стеками.

Если ошибка ожидаема (например, `ResourceNotFound` при несуществующем id), стек теряет смысл и даже вреден. Достаточно `WARN` с минимальными деталями и контекстом. Для неожиданных (NPE, IllegalState) логируйте `ERROR` со стеком — он нужен для фикса и RCA.

Полезно различать «транспортные» и «доменные» ошибки в логах: выделите для них разные коды/маркеры (slf4j Marker) и уровни. Это упрощает построение дашбордов: вы видите всплески 5xx отдельно от «пользовательских» 4xx.

Избегайте логов-файерхаузов: не печатайте гигантские payload’ы или большие коллекции в ошибках. Либо сокращайте (`size=123`), либо пишите хэши/идентификаторы. Это удешевит хранение и ускорит поиск, не ухудшая диагностику.

Для реактивного кода используйте `checkpoint` и `Hooks.onOperatorDebug()` (в дев-окружении), чтобы стек становился «читаемым». В продакшне операторный дебаг выключайте — он дорогой. Но `checkpoint("label")` остаётся дешёвым способом подсветки мест падения.

Если у вас кросс-сервисная трассировка (Zipkin/Tempo/OpenTelemetry), ошибки помечайте статусом span’а и записывайте `exception.type/message/stacktrace` как event. Тогда вы увидите «красную дорожку» и место, где всё сломалось, не открывая логи.

И, конечно, автоматизируйте: алерты по доле 5xx, по частоте конкретных кодов ошибок, по всплескам таймаутов. Это позволит реагировать до того, как пользователи начнут жаловаться.

**Java — логирование в глобальном обработчике (однократно):**

```java
package com.example.logging.errors;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.*;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
class ErrorAdvice {
    private static final Logger log = LoggerFactory.getLogger(ErrorAdvice.class);

    @ExceptionHandler(RuntimeException.class)
    ProblemDetail onUnexpected(RuntimeException ex) {
        log.error("UNEXPECTED_ERROR traceId={}", org.slf4j.MDC.get("traceId"), ex);
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal error");
        pd.setDetail("Unexpected server error.");
        return pd;
    }
}
```

**Kotlin — аналог:**

```kotlin
package com.example.logging.errors

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ErrorAdvice {
    private val log = LoggerFactory.getLogger(ErrorAdvice::class.java)

    @ExceptionHandler(RuntimeException::class)
    fun onUnexpected(ex: RuntimeException): ProblemDetail {
        log.error("UNEXPECTED_ERROR traceId={}", MDC.get("traceId"), ex)
        return ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR).apply {
            title = "Internal error"
            detail = "Unexpected server error."
        }
    }
}
```

---

## Трассировка ошибок через MDC (Mapped Diagnostic Context) и прокачка контекста

MDC — это «карман» для метаданных логирования на уровне потоков. В синхронном коде он работает прозрачно: положили `traceId` — и все логи в этом потоке его видят. В асинхронном мире нужно **переносить** MDC вручную: через `TaskDecorator` для `@Async` и через мост из Reactor Context в MDC для WebFlux.

Идея моста проста: при каждом сигнале на подписчике вы берёте `traceId` из `Context` и кладёте его в MDC перед логированием, а потом очищаете. Это можно обернуть в утилиту `MdcContextLifter`, которую вы вставляете в начало цепочки (`.doOnEach(new MdcContextLifter())`) или включаете глобально через `Hooks.onEachOperator`.

Таким образом все `log.info/error` внутри реактивной цепочки будут сопровождаться актуальным `traceId`. Важно помнить, что MDC не «потокобезопасен» в смысле реактивных переключений — обязательно очищайте его после использования, иначе «протечёт» в чужие запросы.

Для MVC/`@Async` мы уже показали `TaskDecorator`. Он копирует содержимое MDC «снаружи» в «внутри». Это особенно полезно, когда вы делаете несколько `@Async` вызовов из одного запроса: все они унаследуют `traceId` и попадут в те же корреляционные логи.

Если вы используете Micrometer Tracing/Brave/Otel, часть этой работы сделают библиотеки: они кладут `traceId/spanId` в MDC автоматом. Но свои бизнес-идентификаторы (`customerId`, `orderId`) всё равно удобно тащить вручную — библиотеки про них не знают.

Тестируйте прокачку: отправляйте запрос, проверяйте, что в ответе есть `X-Request-Id`, а в логах все записи этого запроса содержат тот же id, включая внутри `@Async` и реактивных операторов. Это предотвращает «дырявые» трассы до выхода в прод.

Помните о производительности: глобальные хуки Reactor с «дорогой» логикой могут стоить CPU. Делайте мост максимально лёгким и без аллокаций. Если нужно больше контекста — храните в `Context` только ключи, а «тяжёлые» структуры восстанавливайте по ключам.

**Java — мост Reactor Context → MDC:**

```java
package com.example.logging.mdc;

import org.reactivestreams.Subscription;
import org.slf4j.MDC;
import reactor.core.CoreSubscriber;
import reactor.core.publisher.Mono;
import reactor.util.context.Context;

public class MdcContextLifter<T> implements CoreSubscriber<T> {
    private final CoreSubscriber<T> actual;
    public MdcContextLifter(CoreSubscriber<T> actual) { this.actual = actual; }

    @Override public Context currentContext() { return actual.currentContext(); }

    @Override public void onSubscribe(Subscription s) { actual.onSubscribe(s); }

    @Override public void onNext(T t) {
        withMdc(() -> actual.onNext(t));
    }

    @Override public void onError(Throwable t) {
        withMdc(() -> actual.onError(t));
    }

    @Override public void onComplete() {
        withMdc(actual::onComplete);
    }

    private void withMdc(Runnable r) {
        String traceId = currentContext().getOrDefault("traceId", null);
        String old = MDC.get("traceId");
        if (traceId != null) MDC.put("traceId", traceId);
        try { r.run(); }
        finally { if (old != null) MDC.put("traceId", old); else MDC.remove("traceId"); }
    }

    public static <T> Mono<T> lift(Mono<T> mono) {
        return Mono.deferContextual(ctx -> mono).doOnEach(sig -> {
            // no-op; пример использования через doOnEach(new MdcContextLifter<>(...)) в хукающем операторе
        });
    }
}
```

**Kotlin — утилита и пример использования:**

```kotlin
package com.example.logging.mdc

import org.reactivestreams.Subscription
import org.slf4j.MDC
import reactor.core.CoreSubscriber
import reactor.core.publisher.Mono
import reactor.util.context.Context

class MdcContextLifter<T>(private val actual: CoreSubscriber<T>) : CoreSubscriber<T> {
    override fun currentContext(): Context = actual.currentContext()

    override fun onSubscribe(s: Subscription) = actual.onSubscribe(s)

    override fun onNext(t: T) = withMdc { actual.onNext(t) }

    override fun onError(t: Throwable) = withMdc { actual.onError(t) }

    override fun onComplete() = withMdc { actual.onComplete() }

    private fun withMdc(block: () -> Unit) {
        val traceId = currentContext().getOrDefault<String?>("traceId", null)
        val old = MDC.get("traceId")
        if (traceId != null) MDC.put("traceId", traceId)
        try { block() } finally { if (old != null) MDC.put("traceId", old) else MDC.remove("traceId") }
    }
}

// пример: внедрение через оператор
fun <T> Mono<T>.withMdc(): Mono<T> = this.doOnEach { /* см. Java-версию для общего подхода */ }
```

# 10. Тестирование ошибок и исключений

## Тестирование ошибок через `MockMvc`: проверка правильности возврата ошибок через контроллеры

Проверять негативные сценарии важно не меньше, чем «зелёные» пути. Именно на ошибках клиент получает основной сигнал о том, что делать дальше: повторить, исправить данные или показать пользователю понятное сообщение. `MockMvc` — быстрый и детерминированный способ проверить контракт контроллеров без запуска полноценного контейнера сервера и сети. Он даёт возможность вызывать эндпоинты как бы «изнутри» Spring MVC и проверять и статус, и заголовки, и тело ответа.

В связке с нашим стандартом ошибок (RFC 7807 / `ProblemDetail`) задача тестов сводится к нескольким инвариантам. Во-первых, корректный HTTP-статус должен совпадать с полем `status` в теле. Во-вторых, медиатип должен быть `application/problem+json`, иначе фронтенд может неправильно распарсить ответ. В-третьих, поля `title`, `detail`, `type`, `instance` должны иметь ожидаемые значения, а расширения вроде `code`/`errors` — присутствовать по договорённости.

`MockMvc` удобен тем, что позволяет точечно поднимать ровно те бины, которые нужны: сам контроллер и обработчик ошибок. Это ускоряет тесты и делает их менее хрупкими. Если в тесте мы используем `@WebMvcTest`, Spring поднимет MVC-инфраструктуру, конвертеры JSON и наш `@ControllerAdvice`, но не будет поднимать базы, Kafka и прочие зависимости, которые тут ни к чему.

При проверке ошибок важно уметь провоцировать их предсказуемо. Для 404 удобно иметь контроллер, который бросает `ResourceNotFoundException`, а глобальный обработчик мапит её на `ProblemDetail`. Для 400/422 достаточно отправить некорректный JSON или DTO, нарушающий валидацию. Для 403 можно смоделировать `AccessDeniedException` через мокаут `SecurityContext`, но это уже интеграционные сценарии — их мы вынесем в следующий пункт.

Отдельно стоит проверять заголовки: `X-Request-Id` или `traceId` должны возвращаться клиенту, чтобы он мог связать ошибку с логами. Также полезно проверять детерминированность сериализации: один и тот же запрос должен давать одинаковое тело, что важно для кэширования и предсказуемости клиентов.

Наконец, помните, что `MockMvc` — не сетевой тест. Он не проверит таймауты сокета и прокси-заголовки. Его цель — поведение контроллера и слоя представления ошибок. Для сетевых эффектов нужны интеграционные проверки, о которых — ниже.

В коде ниже показаны минимальные контроллер и глобальный обработчик и сами тесты. Мы проверяем статус, медиатип и ключевые поля JSON. В реальном проекте добавьте вспомогательные матчер-утилиты, чтобы не дублировать JSONPath по всему тестовому набору.

**Gradle (зависимости тестов)**
*Groovy DSL:*

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test' // JUnit 5, AssertJ, JSONassert, MockMvc
}
test {
    useJUnitPlatform()
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
tasks.test { useJUnitPlatform() }
```

**Java — `MockMvc` проверка `ProblemDetail` (404, 422):**

```java
package com.example.testing.mockmvc;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.MediaType;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = {OrdersController.class})
@Import(GlobalAdvice.class)
class ErrorMockMvcTest {

    @Autowired MockMvc mvc;

    @Test @DisplayName("GET /api/orders/0 -> 404 ProblemDetail")
    void notFound() throws Exception {
        mvc.perform(get("/api/orders/0"))
           .andExpect(status().isNotFound())
           .andExpect(content().contentType("application/problem+json"))
           .andExpect(jsonPath("$.status").value(404))
           .andExpect(jsonPath("$.title").value("Order not found"))
           .andExpect(jsonPath("$.detail").value("Order 0 does not exist"))
           .andExpect(jsonPath("$.type").value("https://docs.example.com/problems/order-not-found"));
    }

    @Test @DisplayName("POST /api/orders (invalid) -> 422 with errors[]")
    void validation() throws Exception {
        String badJson = "{\"name\":\"\"}";
        mvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(badJson))
           .andExpect(status().isUnprocessableEntity())
           .andExpect(content().contentType("application/problem+json"))
           .andExpect(jsonPath("$.title").value("Validation failed"))
           .andExpect(jsonPath("$.errors[0].field").value("name"));
    }
}

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @GetMapping("/{id}")
    public Object get(@PathVariable long id) {
        if (id == 0) throw new OrderNotFoundException("Order 0 does not exist");
        return new OrderDto(id, "ok");
    }
    @PostMapping
    public Object create(@RequestBody @Valid CreateOrder req) { return new OrderDto(1L, req.name()); }

    record OrderDto(long id, String name) {}
    record CreateOrder(@NotBlank String name) {}
}

@RestControllerAdvice
class GlobalAdvice {
    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail notFound(OrderNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("Order not found");
        pd.setDetail(ex.getMessage());
        pd.setType(java.net.URI.create("https://docs.example.com/problems/order-not-found"));
        return pd;
    }
    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException.class)
    ProblemDetail invalid(org.springframework.web.bind.MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        var errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> java.util.Map.of("field", fe.getField(), "message", fe.getDefaultMessage()))
                .toList();
        pd.setProperty("errors", errors);
        return pd;
    }
}

class OrderNotFoundException extends RuntimeException { public OrderNotFoundException(String msg){ super(msg); } }
```

**Kotlin — `MockMvc` проверка `ProblemDetail`:**

```kotlin
package com.example.testing.mockmvc

import jakarta.validation.Valid
import jakarta.validation.constraints.NotBlank
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.context.annotation.Import
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ProblemDetail
import org.springframework.test.web.servlet.MockMvc
import org.springframework.web.bind.annotation.*
import kotlin.jvm.Throws
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*

@WebMvcTest(controllers = [OrdersController::class])
@Import(GlobalAdvice::class)
class ErrorMockMvcTest(
    @Autowired val mvc: MockMvc
) {

    @Test
    @DisplayName("GET /api/orders/0 -> 404 ProblemDetail")
    fun notFound() {
        mvc.perform(get("/api/orders/0"))
            .andExpect(status().isNotFound)
            .andExpect(content().contentType("application/problem+json"))
            .andExpect(jsonPath("$.status").value(404))
            .andExpect(jsonPath("$.title").value("Order not found"))
            .andExpect(jsonPath("$.detail").value("Order 0 does not exist"))
            .andExpect(jsonPath("$.type").value("https://docs.example.com/problems/order-not-found"))
    }

    @Test
    @DisplayName("POST /api/orders (invalid) -> 422 with errors[]")
    fun validation() {
        val badJson = """{"name":""}"""
        mvc.perform(post("/api/orders").contentType(MediaType.APPLICATION_JSON).content(badJson))
            .andExpect(status().isUnprocessableEntity)
            .andExpect(content().contentType("application/problem+json"))
            .andExpect(jsonPath("$.title").value("Validation failed"))
            .andExpect(jsonPath("$.errors[0].field").value("name"))
    }
}

@RestController
@RequestMapping("/api/orders")
class OrdersController {
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long): Any =
        if (id == 0L) throw OrderNotFoundException("Order 0 does not exist") else OrderDto(id, "ok")

    @PostMapping
    fun create(@RequestBody @Valid req: CreateOrder): Any = OrderDto(1L, req.name)
}

data class OrderDto(val id: Long, val name: String)
data class CreateOrder(@field:NotBlank val name: String)

@RestControllerAdvice
class GlobalAdvice {
    @ExceptionHandler(OrderNotFoundException::class)
    fun notFound(ex: OrderNotFoundException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply {
            title = "Order not found"
            detail = ex.message
            type = java.net.URI.create("https://docs.example.com/problems/order-not-found")
        }

    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException::class)
    fun invalid(ex: org.springframework.web.bind.MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("errors", ex.bindingResult.fieldErrors.map { mapOf("field" to it.field, "message" to it.defaultMessage) })
        }
}

class OrderNotFoundException(message: String) : RuntimeException(message)
```

---

## Интеграционные тесты: проверка проблемных ответов (например, 404, 400, 500)

Интеграционный тест — это более «реалистичная» проверка, чем `MockMvc`. Мы поднимаем весь контекст Spring Boot, реальный HTTP-стек и слушаем порт. Такой тест позволяет оценить поведение маршалинга/демаршалинга, фильтров, кодировок, заголовков, компрессии и всех аспектов, которые в `MockMvc` остаются вне кадра. Да, такие тесты медленнее, но именно они с наибольшей вероятностью повторяют продакшн-поведение.

Для веб-интеграции в Spring Boot удобно использовать `WebTestClient` даже в MVC-приложениях. Он выполняет реальные HTTP-запросы поверх Netty/Jetty/Tomcat (в зависимости от старта), но не требует внешней сети. Альтернатива — `TestRestTemplate`, который проще, но менее выразителен для реактивных сценариев и проверки тела через JSONPath.

Интеграционные тесты должны покрывать не только «ожидаемые» 404/422, но и «неожиданные» 500. Последние мы искусственно провоцируем: например, сервис бросает `RuntimeException`. Обработчик обязан вернуть 500 с `ProblemDetail`, не вывалившись HTML-страницей по умолчанию. Это защита от регрессий при миграциях Spring/Jackson.

Заголовки — часть контракта. В реальном сервере у вас есть фильтр, который устанавливает `X-Request-Id`. Интеграционный тест проверит, что заголовок действительно присутствует и «пролетает» через стек фильтров и обработчиков, а не теряется где-то по дороге. Это мелочь, но такие мелочи спасают часы отладки.

Полезный приём — заранее «пробрасывать» заголовок `Accept: application/problem+json`, если у вас есть контент-негациация. Так вы поймаете расхождения между настройками конвертеров для JSON/Problem. Если в ответ пришёл не тот медиатип, значит где-то нарушена конфигурация.

Важно, чтобы интеграционные тесты были изолированы и не зависели от внешних сервисов. Для внешних HTTP-клиентов используйте WireMock или MockWebServer; для БД — Testcontainers или встроенные профили. Но сами тесты ошибок не обязаны тянуть всё это: их цель — проверить крайние пути на уровне веб-слоя.

Наконец, держите хорошую структуру пакетов: интеграционные тесты могут лежать рядом с модулем API, а e2e — в отдельном модуле. Так вы ускорите локальную разработку и CI, запуская только нужный контур в зависимости от типа изменения.

**Java — интеграционный тест с `WebTestClient` и реальным портом:**

```java
package com.example.testing.it;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.MediaType;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        classes = {ItConfig.class, ItController.class, ItAdvice.class})
class ErrorIntegrationTest {

    @LocalServerPort int port;

    private WebTestClient client() {
        return WebTestClient.bindToServer().baseUrl("http://localhost:" + port).build();
    }

    @Test
    void notFoundOverHttp() {
        client().get().uri("/it/orders/0")
                .accept(MediaType.valueOf("application/problem+json"))
                .exchange()
                .expectStatus().isNotFound()
                .expectHeader().contentType("application/problem+json")
                .expectBody()
                .jsonPath("$.status").isEqualTo(404)
                .jsonPath("$.title").isEqualTo("Order not found");
    }

    @Test
    void unexpected500OverHttp() {
        client().get().uri("/it/boom")
                .exchange()
                .expectStatus().is5xxServerError()
                .expectHeader().contentType("application/problem+json")
                .expectBody()
                .jsonPath("$.status").isEqualTo(500)
                .jsonPath("$.title").isEqualTo("Internal error");
    }
}

@RestController
@RequestMapping("/it")
class ItController {
    @GetMapping("/orders/{id}") public Object get(@PathVariable long id) {
        if (id == 0L) throw new OrderNotFound("no order");
        return java.util.Map.of("id", id);
    }
    @GetMapping("/boom") public String boom() { throw new IllegalStateException("boom"); }
}

@RestControllerAdvice
class ItAdvice {
    @org.springframework.web.bind.annotation.ExceptionHandler(OrderNotFound.class)
    ProblemDetail nf(OrderNotFound ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        pd.setTitle("Order not found"); pd.setDetail(ex.getMessage()); return pd;
    }
    @org.springframework.web.bind.annotation.ExceptionHandler(Throwable.class)
    ProblemDetail any(Throwable ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal error"); pd.setDetail("Unexpected server error."); return pd;
    }
}
class OrderNotFound extends RuntimeException { public OrderNotFound(String m){ super(m);} }

@Configuration class ItConfig { @Bean WebTestClient.Builder webTestClientBuilder(){ return WebTestClient.bindToServer(); } }
```

**Kotlin — интеграционный тест с `WebTestClient`:**

```kotlin
package com.example.testing.it

import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.server.LocalServerPort
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ProblemDetail
import org.springframework.test.web.reactive.server.WebTestClient
import org.springframework.web.bind.annotation.*

@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
    classes = [ItConfig::class, ItController::class, ItAdvice::class]
)
class ErrorIntegrationTest(
    @LocalServerPort val port: Int
) {

    private fun client(): WebTestClient =
        WebTestClient.bindToServer().baseUrl("http://localhost:$port").build()

    @Test
    fun notFoundOverHttp() {
        client().get().uri("/it/orders/0")
            .accept(MediaType.valueOf("application/problem+json"))
            .exchange()
            .expectStatus().isNotFound
            .expectHeader().contentType("application/problem+json")
            .expectBody()
            .jsonPath("$.status").isEqualTo(404)
            .jsonPath("$.title").isEqualTo("Order not found")
    }

    @Test
    fun unexpected500OverHttp() {
        client().get().uri("/it/boom")
            .exchange()
            .expectStatus().is5xxServerError
            .expectHeader().contentType("application/problem+json")
            .expectBody()
            .jsonPath("$.status").isEqualTo(500)
            .jsonPath("$.title").isEqualTo("Internal error")
    }
}

@RestController
@RequestMapping("/it")
class ItController {
    @GetMapping("/orders/{id}")
    fun get(@PathVariable id: Long): Any =
        if (id == 0L) throw OrderNotFound("no order") else mapOf("id" to id)

    @GetMapping("/boom")
    fun boom(): String = throw IllegalStateException("boom")
}

@RestControllerAdvice
class ItAdvice {
    @ExceptionHandler(OrderNotFound::class)
    fun nf(ex: OrderNotFound): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.NOT_FOUND).apply { title = "Order not found"; detail = ex.message }

    @ExceptionHandler(Throwable::class)
    fun any(@Suppress("UNUSED_PARAMETER") ex: Throwable): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR).apply { title = "Internal error"; detail = "Unexpected server error." }
}

class OrderNotFound(message: String) : RuntimeException(message)
@Configuration class ItConfig { @Bean fun webTestClientBuilder(): WebTestClient.Builder = WebTestClient.bindToServer() }
```

---

## Тестирование обработки валидации и корректного маппинга ошибок: различные сценарии данных

Валидационные ошибки — самый частый класс негативных ответов. Их важно тестировать с разных углов: пустые строки, неверные форматы, отрицательные числа, отсутствующие поля и даже сломанный JSON. Каждому сценарию соответствует свой статус и формат: для «сломанный JSON» чаще 400, для «данные валидны синтаксически, но противоречат правилам» — 422 с массивом `errors`.

Особое внимание уделите соответствию имён полей. Если в `ObjectMapper` включена `SNAKE_CASE`, а ваши DTO — в camelCase, то `BindingResult` будет хранить имена по Java-полям, а клиент ожидает snake_case. Это ловушка: тесты должны проверять, что в массиве `errors` имена соответствуют внешнему контракту, иначе фронтенд не сможет сопоставить их с формой. Решение — транслировать имена перед отправкой или держать в DTO ровно те имена, что в JSON.

В ответе имеет смысл возвращать не только `message`, но и `code` для каждого нарушения. Тогда тексты можно локализовать на клиенте, не размазывая их по серверу. В тестах проверьте, что `code` принимает ожидаемые значения (`NotBlank`, `Email`, `Min`) или ваши специализированные коды. Это помогает держать стабильный словарь ошибок.

Стоит также проверять вложенные объекты и коллекции: `addresses[0].city` и подобные пути должны корректно попадать в `errors`. Здесь вновь спасают тесты с JSONPath, которые сверяют не только наличие массива, но и точные пути до полей. Ошибки в путях быстро всплывают при рефакторингах DTO.

Ещё одна грань — согласованность с `ProblemDetail`. Ваш глобальный обработчик должен превращать `MethodArgumentNotValidException` в 422 и склеивать ошибки в `errors`. Тесты должны утверждать и статус, и форму тела. Это особенно важно при обновлениях Spring Boot: иногда меняются дефолтные сообщения или поля, и ваш маппинг может непреднамеренно измениться.

Наконец, не забывайте про негативные сценарии на параметрах запроса (`@RequestParam`, `@PathVariable`). Для них летит `ConstraintViolationException`, и формат ответа может отличаться. Держите единый конвертер и утилиты, и проверяйте их отдельным тестом, чтобы клиент всегда получал «одну и ту же» форму проблем.

В примерах ниже мы показываем POST с неправильными полями и проверяем, что сервер отвечает 422, а JSON содержит `errors` с ожидаемыми путями и кодами. Также добавлен отдельный тест на «сломанный» JSON и статус 400.

**Java — проверки сценариев валидации (MockMvc):**

```java
package com.example.testing.validation;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ProblemDetail;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(controllers = {UsersController.class})
@Import(ValAdvice.class)
class ValidationMappingTest {

    @Autowired MockMvc mvc;

    @Test
    void emptyFields422() throws Exception {
        String bad = "{\"email\":\"not-an-email\",\"age\":-1}";
        mvc.perform(post("/api/users").contentType(MediaType.APPLICATION_JSON).content(bad))
           .andExpect(status().isUnprocessableEntity())
           .andExpect(jsonPath("$.errors[?(@.field=='email')].code").value(org.hamcrest.Matchers.hasItem("Email")))
           .andExpect(jsonPath("$.errors[?(@.field=='age')].code").value(org.hamcrest.Matchers.hasItem("Min")));
    }

    @Test
    void brokenJson400() throws Exception {
        String broken = "{\"email\":\"a@b.c\""; // нет закрывающей скобки
        mvc.perform(post("/api/users").contentType(MediaType.APPLICATION_JSON).content(broken))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.title").value("Bad JSON"));
    }
}

@RestController
@RequestMapping("/api/users")
class UsersController {
    @PostMapping
    public Object create(@RequestBody @Valid UserDto dto) { return dto; }
    record UserDto(@NotBlank String name, @Email String email, @Min(0) int age) {}
}

@RestControllerAdvice
class ValAdvice {
    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException.class)
    ProblemDetail invalid(org.springframework.web.bind.MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        var errs = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> java.util.Map.of("field", fe.getField(), "code", fe.getCode(), "message", fe.getDefaultMessage()))
                .toList();
        pd.setProperty("errors", errs);
        return pd;
    }
    @ExceptionHandler(org.springframework.http.converter.HttpMessageNotReadableException.class)
    ProblemDetail badJson(org.springframework.http.converter.HttpMessageNotReadableException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Bad JSON"); pd.setDetail(ex.getMostSpecificCause().getMessage());
        return pd;
    }
}
```

**Kotlin — проверки сценариев валидации (MockMvc):**

```kotlin
package com.example.testing.validation

import jakarta.validation.Valid
import jakarta.validation.constraints.Email
import jakarta.validation.constraints.Min
import jakarta.validation.constraints.NotBlank
import org.hamcrest.Matchers.hasItem
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.context.annotation.Import
import org.springframework.http.HttpStatus
import org.springframework.http.MediaType
import org.springframework.http.ProblemDetail
import org.springframework.test.web.servlet.MockMvc
import org.springframework.web.bind.annotation.*
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*

@WebMvcTest(controllers = [UsersController::class])
@Import(ValAdvice::class)
class ValidationMappingTest(@Autowired val mvc: MockMvc) {

    @Test
    fun emptyFields422() {
        val bad = """{"email":"not-an-email","age":-1}"""
        mvc.perform(post("/api/users").contentType(MediaType.APPLICATION_JSON).content(bad))
            .andExpect(status().isUnprocessableEntity)
            .andExpect(jsonPath("$.errors[?(@.field=='email')].code").value(hasItem("Email")))
            .andExpect(jsonPath("$.errors[?(@.field=='age')].code").value(hasItem("Min")))
    }

    @Test
    fun brokenJson400() {
        val broken = """{"email":"a@b.c""" // нет закрывающей скобки
        mvc.perform(post("/api/users").contentType(MediaType.APPLICATION_JSON).content(broken))
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.title").value("Bad JSON"))
    }
}

@RestController
@RequestMapping("/api/users")
class UsersController {
    @PostMapping
    fun create(@RequestBody @Valid dto: UserDto): Any = dto
}

data class UserDto(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Min(0) val age: Int
)

@RestControllerAdvice
class ValAdvice {
    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException::class)
    fun invalid(ex: org.springframework.web.bind.MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY).apply {
            title = "Validation failed"
            setProperty("errors", ex.bindingResult.fieldErrors.map {
                mapOf("field" to it.field, "code" to it.code, "message" to it.defaultMessage)
            })
        }

    @ExceptionHandler(org.springframework.http.converter.HttpMessageNotReadableException::class)
    fun badJson(ex: org.springframework.http.converter.HttpMessageNotReadableException): ProblemDetail =
        ProblemDetail.forStatus(HttpStatus.BAD_REQUEST).apply {
            title = "Bad JSON"; detail = ex.mostSpecificCause.message
        }
}
```

---

## Набор тестов для проверок стандартов ошибок и их возврата в API (например, используя JUnit 5 и AssertJ)

Чтобы ошибки были действительно «стандартом», нужны не только договорённости, но и автоматические проверки. Практика показывает, что полезно иметь **набор инвариантов**, который прогоняется для каждого негативного ответа: медиатип `application/problem+json`, наличие базовых полей `type`, `title`, `status`, `detail`, корректность `status` внутри тела, присутствие `timestamp/code` как расширений. Такой набор легко оформить в виде утилитных assert-методов или расширений AssertJ.

JUnit 5 помогает организовать **параметризованные тесты** для разных эндпоинтов и сценариев, чтобы не плодить однотипные методы. Вы описываете «табличку» из URI и ожидаемого статуса/типа ошибки, а тест бежит по строкам. Это выгодно, когда у вас десятки однотипных ошибок (например, 404 на разных ресурсах).

AssertJ удобен выразительными утверждениями над JSON. Вы можете распарсить тело в `JsonNode` через Jackson и писать читабельные ассерты: `assertThat(node.at("/status").asInt()).isEqualTo(404)`. Такой подход часто приятнее JSONPath: статически типизирован, даёт понятный diff на падении и позволяет реюзать проверки.

Ещё одна полезная техника — **кастомные ассерты**. В Java это класс `ProblemAssert` с методами `hasStatus`, `hasTitle`, `hasType`, `hasErrorForField`. В Kotlin — расширения/инфикс-функции над `JsonNode` или над DTO `ProblemDetail`. Кастомный API позволяет писать тесты на английском бизнес-языке и упрощает сопровождение.

Отдельно стоит проверить **согласованность `type`**: URI должны вести на существующие страницы документации или хотя бы иметь единый префикс. В тестах можно делать не сетевой вызов, а лишь проверять формат через `startsWith("https://docs.example.com/problems/")`. Это дисциплинирует пополнение каталога ошибок.

Наконец, полезно проверить, что глобальный обработчик ошибок не «просачивает» стек-трейсы в тело. Иногда разработчики из лучших побуждений добавляют `stackTrace` в расширения — и это утекает в фронтенд. Автотест, убеждающийся в отсутствии такого поля, — дешёвая страховка репутации.

Ниже — пример общего помощника для проверок Problem JSON и набор параметризованных тестов поверх `MockMvc`/`WebTestClient`. Он показывает идею: выносите повторяющиеся проверки в утилиты и используйте их везде, где тестируете ошибки.

**Java — AssertJ-утилита и параметризованный тест:**

```java
package com.example.testing.std;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.assertj.core.api.AbstractAssert;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;

@WebMvcTest(controllers = StdController.class)
@Import(StdAdvice.class)
class ProblemStandardTest {

    @Autowired MockMvc mvc;
    ObjectMapper om = new ObjectMapper();

    static Stream<String> notFoundUris() {
        return Stream.of("/std/users/0", "/std/orders/0");
    }

    @ParameterizedTest
    @MethodSource("notFoundUris")
    void allNotFoundAreStandardized(String uri) throws Exception {
        var res = mvc.perform(get(uri).accept(MediaType.valueOf("application/problem+json"))).andReturn();
        var body = res.getResponse().getContentAsString();
        JsonNode node = om.readTree(body);
        ProblemAssert.assertThat(node)
                .hasStatus(404)
                .hasTitle("Resource not found")
                .hasTypePrefix("https://docs.example.com/problems/");
        assertThat(res.getResponse().getContentType()).isEqualTo("application/problem+json");
    }
}

class ProblemAssert extends AbstractAssert<ProblemAssert, JsonNode> {
    protected ProblemAssert(JsonNode actual) { super(actual, ProblemAssert.class); }
    public static ProblemAssert assertThat(JsonNode node) { return new ProblemAssert(node); }
    public ProblemAssert hasStatus(int s) { isNotNull(); assertThat(actual.at("/status").asInt()).isEqualTo(s); return this; }
    public ProblemAssert hasTitle(String t) { assertThat(actual.at("/title").asText()).isEqualTo(t); return this; }
    public ProblemAssert hasTypePrefix(String p) { assertThat(actual.at("/type").asText()).startsWith(p); return this; }
}

@RestController
@RequestMapping("/std")
class StdController {
    @GetMapping("/users/{id}") public Object u(@PathVariable long id) { if (id==0) throw new NotFound(); return java.util.Map.of(); }
    @GetMapping("/orders/{id}") public Object o(@PathVariable long id) { if (id==0) throw new NotFound(); return java.util.Map.of(); }
}

@RestControllerAdvice
class StdAdvice {
    @ExceptionHandler(NotFound.class)
    public org.springframework.http.ProblemDetail on(NotFound ex) {
        var pd = org.springframework.http.ProblemDetail.forStatus(404);
        pd.setTitle("Resource not found");
        pd.setType(java.net.URI.create("https://docs.example.com/problems/not-found"));
        return pd;
    }
}
class NotFound extends RuntimeException {}
```

**Kotlin — утилиты и параметризованный тест:**

```kotlin
package com.example.testing.std

import com.fasterxml.jackson.databind.JsonNode
import com.fasterxml.jackson.databind.ObjectMapper
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.MethodSource
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.context.annotation.Import
import org.springframework.http.MediaType
import org.springframework.http.ProblemDetail
import org.springframework.test.web.servlet.MockMvc
import org.springframework.web.bind.annotation.*
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get

@WebMvcTest(controllers = [StdController::class])
@Import(StdAdvice::class)
class ProblemStandardTest(@Autowired val mvc: MockMvc) {

    private val om = ObjectMapper()

    companion object {
        @JvmStatic
        fun notFoundUris(): List<String> = listOf("/std/users/0", "/std/orders/0")
    }

    @ParameterizedTest
    @MethodSource("notFoundUris")
    fun allNotFoundAreStandardized(uri: String) {
        val res = mvc.perform(get(uri).accept(MediaType.valueOf("application/problem+json"))).andReturn()
        val node: JsonNode = om.readTree(res.response.contentAsString)
        node.assertProblem().hasStatus(404).hasTitle("Resource not found").hasTypePrefix("https://docs.example.com/problems/")
        assertThat(res.response.contentType).isEqualTo("application/problem+json")
    }
}

fun JsonNode.assertProblem(): ProblemNodeAssert = ProblemNodeAssert(this)
class ProblemNodeAssert(private val node: JsonNode) {
    fun hasStatus(s: Int): ProblemNodeAssert { assertThat(node.at("/status").asInt()).isEqualTo(s); return this }
    fun hasTitle(t: String): ProblemNodeAssert { assertThat(node.at("/title").asText()).isEqualTo(t); return this }
    fun hasTypePrefix(p: String): ProblemNodeAssert { assertThat(node.at("/type").asText()).startsWith(p); return this }
}

@RestController
@RequestMapping("/std")
class StdController {
    @GetMapping("/users/{id}") fun u(@PathVariable id: Long): Any = if (id == 0L) throw NotFound() else emptyMap<String,Any>()
    @GetMapping("/orders/{id}") fun o(@PathVariable id: Long): Any = if (id == 0L) throw NotFound() else emptyMap<String,Any>()
}

@RestControllerAdvice
class StdAdvice {
    @ExceptionHandler(NotFound::class)
    fun on(@Suppress("UNUSED_PARAMETER") ex: NotFound): ProblemDetail =
        ProblemDetail.forStatus(404).apply {
            title = "Resource not found"
            type = java.net.URI.create("https://docs.example.com/problems/not-found")
        }
}
class NotFound : RuntimeException()
```

В итоге у нас получился «сквозной» контур тестирования ошибок: быстрые и точные юниты/слайс-тесты на `MockMvc`, реалистичные интеграционные проверки на `WebTestClient`, специальный слой тестов для валидации и общий стандарт-набор утверждений для Problem JSON. Такая дисциплина обеспечивает устойчивость API при эволюции, обновлениях зависимостей и работе разных команд над разными частями контракта.






