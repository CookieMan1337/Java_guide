---
layout: page
title: "Логирование и наблюдаемость (базовый уровень)"
permalink: /spring/logging
---

# 0. Введение

## Что это такое

Логирование и наблюдаемость — это совокупность практик и инструментов, позволяющих видеть, **что именно делает** ваш сервис в рантайме, какие события происходят, каковы задержки и где «болит». Под «логами» понимаются текстовые (чаще — **структурированные JSON**) сообщения о важных шагах выполнения. Под «наблюдаемостью» (observability) — более широкий зонт: логи, метрики, трейсинг, health-пробы и служебная информация о версии и окружении.

В экосистеме Spring Boot 3.x у вас из коробки есть дружелюбные дефолты: Logback как движок логов, Micrometer как единый API метрик, Actuator для health/metrics/info-эндпоинтов и Micrometer Tracing (с OpenTelemetry) для распределённой трассировки. Всё это собирается через авто-конфигурации, а вы настраиваете поведение простыми свойствами.

Ключевая мысль: наблюдаемость — это **не роскошь**, а средство управления риском. Если у сервиса нет логов и метрик, в инцидент вы идёте «вслепую». Если же базовый контур настроен, любая деградация будет видна: ошибки в логах, всплеск латентности на графике, failing health — вы реагируете быстрее и точнее.

Современные практики требуют **структурных** логов (JSON) вместо свободного текста. Такой формат хорошо парсится агентами (fluent-bit/filebeat), переживает изменения шаблонов и одинаково читается человеком и машиной. Структура — это темплейт стабильных полей: timestamp, level, service, env, traceId и т.д.

Наблюдаемость — ещё и **контракт команды**. Вы не просто «что-то где-то пишете», а согласуете уровни логов, состав полей, маскирование секретов, политику хранения, набор метрик и минимальные дашборды. Это снимает споры и упрощает поддержку.

Наконец, важно помнить о **стоимости**: логи и метрики занимают дисковое/сетевое место и CPU. Задача базового уровня — найти баланс: не «лить океан», но и не экономить до немоты. В следующих разделах мы зафиксируем рабочие дефолты.

**Код (Java): «минимальный» лог в приложении Boot**
*(показывает подключение SLF4J/Logback и базовое сообщение)*

```java
package demo.intro;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;

@SpringBootApplication
public class IntroApp {
  private static final Logger log = LoggerFactory.getLogger(IntroApp.class);

  public static void main(String[] args) {
    SpringApplication.run(IntroApp.class, args);
    log.info("Service started");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.intro

import org.slf4j.LoggerFactory
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class IntroApp

fun main(args: Array<String>) {
    val log = LoggerFactory.getLogger("demo.intro.IntroApp")
    runApplication<IntroApp>(*args)
    log.info("Service started")
}
```

Gradle зависимости (Groovy/Kotlin) — логирование уже есть в любом стартере:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter' // включает logback + slf4j
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter")
}
```

---

## Зачем это

Главная причина — **диагностика инцидентов**. Когда приходит алерт, вы открываете дашборд метрик, видите провал по p95 латентности, переходите к логам с фильтром по traceId/корреляции и находите конкретную ошибку с контекстом. Без этого цепочка была бы обратной и долгой: «угадай, где упало».

Вторая причина — **подтверждение бизнес-событий**. Не всё — ошибки. Оплата прошла, счёт выставлен, документ подписан — это нормальные события, которые стоит логировать структурно: id операции, сумма, статус. По таким логам строят аудит и отчёты.

Третья — **SLO и ретроспективы**. Метрики (RPS/коды/латентность) показывают, выполняете ли вы целевые уровни обслуживания. Логи дают «почему» (root cause). Трейсинг — «где именно в пути» (какой сервис/зависимость тормозит).

Четвёртая — **поддержка разработки**. На dev-стендах DEBUG-логи экономят время: быстро понять, какое условие сработало, какой SQL ушёл, чем ответил внешний сервис. Главное — по умолчанию держать DEBUG выключенным в проде.

Пятая — **безопасность и аудит**. У вас должен быть след критичных действий (смена прав, доступ к чувствительным данным), но без утечек ПДн/секретов. Маскирование и whitelist-логирования — часть дисциплины.

И наконец, **регуляторика и договоры с платформой**. Часто требования к логам/метрикам формализованы: «экспорт в Prometheus», «traceId обязателен», «хранение 90 дней». Наблюдаемость — это и про соответствие.

**Код (Java): бизнес-лог «оплата проведена»**

```java
package demo.why;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Map;

public class PaymentService {
  private static final Logger log = LoggerFactory.getLogger(PaymentService.class);

  public void capture(String paymentId, long amountMinor) {
    // ... доменная логика
    log.info("payment.captured id={} amountMinor={} currency={}", paymentId, amountMinor, "RUB");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.why

import org.slf4j.LoggerFactory

class PaymentService {
    private val log = LoggerFactory.getLogger(javaClass)
    fun capture(paymentId: String, amountMinor: Long) {
        // ... доменная логика
        log.info("payment.captured id={} amountMinor={} currency={}", paymentId, amountMinor, "RUB")
    }
}
```

---

## Где используется

Логирование и метрики нужны **везде**, где есть код, который может падать, тормозить или взаимодействовать с внешним миром. В HTTP-сервисах это контроллеры/фильтры, клиенты к БД/очередям/внешним API. В batch/CLI — шаги задач, импорт/экспорт файлов, длительные вычисления.

В микросервисной архитектуре наблюдаемость — это **сквозной** аспект. У каждого сервиса должны быть одинаковые теги (service.name, env, version), согласованный формат, общий traceId. Тогда путь запроса легко отследить по всем hop’ам.

Внутри JVM-мира встречаются и «фоновики»: планировщики (`@Scheduled`), обрабочики Kafka, внутренние очереди. Для них особенно важно отделять **технические** логи (шины, ретраи) от **бизнес-логов** (события домена), чтобы поддержка быстро находила нужное.

В облаках и Kubernetes логирование — часть стандартного пайплайна: писать в stdout/stderr, а платформа заберёт через агент и отвезёт в централизованное хранилище (ELK/Cloud Logging). Там же настраиваются дашборды и алерты.

Даже локально (на ноутбуке) структурные логи и простой Prometheus — отличный способ отлавливать регрессы производительности. Бонус: одинаковая конфигурация на всех средах упрощает «повтор багов».

И, наконец, для асинхронной интеграции (Kafka/AMQP) **корреляция** — ключ к поиску. Передавайте свой `correlationId` (или используйте traceId), тащите его в MDC, логируйте в каждом сообщении — и вы перестанете «искать иголку».

**Код (Java): простой фильтр, логирующий входящие запросы**

```java
package demo.where;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger; import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
public class AccessLogFilter implements Filter {
  private static final Logger log = LoggerFactory.getLogger(AccessLogFilter.class);

  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    long t0 = System.nanoTime();
    try { chain.doFilter(req, res); }
    finally { log.info("http.access method={} path={} tookMs={}", r.getMethod(), r.getRequestURI(), (System.nanoTime()-t0)/1_000_000); }
  }
}
```

**Код (Kotlin): аналог (WebFilter для WebFlux)**

```kotlin
package demo.where

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
class AccessLogFilterReactive : WebFilter {
    private val log = LoggerFactory.getLogger(javaClass)
    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        val t0 = System.nanoTime()
        return chain.filter(exchange).doFinally {
            val took = (System.nanoTime() - t0) / 1_000_000
            log.info("http.access method={} path={} tookMs={}", exchange.request.method, exchange.request.path.value(), took)
        }
    }
}
```

---

## Какие задачи/проблемы решает

Первая задача — **быстрый поиск причин**. Со связкой «метрики → трейсинг → логи» вы находите **конкретный** падающий шаг и сразу видите входные параметры/исключение. Без логов вы видите только симптомы (рост p95), а причина остаётся в тени.

Вторая — **контроль шума**. Политика уровней и разделение access-логов от бизнес-логов не дают «утонуть» в потоке. INFO для нормальной жизни, WARN для временных проблем, ERROR — для реальных сбоев. DEBUG в проде выключен, но его можно адресно включить.

Третья — **аудит и расследование**. Структурные логи позволяют писать запросы вроде: «все операции списания > 10k за период с ошибкой 4xx» и быстро отвечать на вопросы бизнеса/безопасности — без «парсинга строк».

Четвёртая — **снижение MTTR** (mean time to recovery). Наблюдаемость — это про **скорость обратной связи**. Чем быстрее вы увидели и поняли проблему, тем меньше даунтайм и ущерб.

Пятая — **предсказуемость релизов**. С включёнными health/metrics можно делать канареечные раскатки, dark-launch и прогрев. Любая деградация будет зафиксирована и остановит промоушен версии.

Шестая — **соблюдение требований**. Маскирование секретов, ограничение размеров сообщений, шифрование/хранение — всё это часть зрелой эксплуатации и ваша защита от утечек.

**Конфигурация Logback JSON (logstash-encoder)**

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp/>
        <pattern>
          <pattern>
            {
              "level":"%level",
              "logger":"%logger",
              "thread":"%thread",
              "message":"%message",
              "service":"${spring.application.name:-app}",
              "env":"${APP_ENV:-local}"
            }
          </pattern>
        </pattern>
        <mdc/> <!-- возьмём traceId/correlationId -->
        <stackTrace/>
      </providers>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

Gradle зависимости (Groovy/Kotlin) — JSON-логирование:

```groovy
dependencies {
  implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
}
```

```kotlin
dependencies {
  implementation("net.logstash.logback:logstash-logback-encoder:7.4")
}
```

---

# 1. Зачем логирование и чем оно отличается от метрик/трейсинга

## Цели: диагностика инцидентов, аудит действий, подтверждение бизнес-событий, подтверждение SLO

**Диагностика.** Когда что-то сломалось, вы хотите быстро найти конкретный стек/параметры. Логи — это «пиктограммы» событий: кто вызвал, с какими аргументами, что сломалось. В идеале каждое ERROR/WARN сопровождается понятным человеком сообщением и ключевыми полями (id, статус).

**Аудит.** Отдельный класс логов — **аудит действий**: смена пароля, назначение ролей, доступ к ресурсам. Тут важно не количество, а полнота и целостность: кто, когда, к чему обратился и в каком окружении. Такие логи идут в отдельный индекс/поток.

**Подтверждение бизнес-событий.** Не каждый INFO — шум. Факт «заказ оплачен» или «счёт отправлен» — это полезная телеметрия для downstream-процессинга и разборов кейсов. Логируйте это отдельным «событийным» ключом (event), чтобы легко фильтровать.

**Подтверждение SLO.** Метрики — формальный ответ «выполняем/нет». Но иногда нужны пояснения: почему выросла латентность? Логи дают качественный контекст (ретраи, таймауты), трейсинг — где именно просело (клиент БД или внешний API).

**Обратная связь разработке.** На dev QA-команда может включить DEBUG для конкретного пакета и получить детальные логи — без перекомпиляции. Это ускоряет воспроизведение багов и снижает нагрузку на разработчиков.

**Коммуникация.** Логи — артефакт, который прикладывают к инцидентам, тикетам и RCA. Хороший лог — это мини-документ: «что произошло, где, с какими параметрами, к чему привело». Пишите сообщения «для людей».

**Код (Java): «аудит» и «бизнес-событие» как разные каналы**

```java
package demo.goals;

import org.slf4j.Logger; import org.slf4j.LoggerFactory;

public class AuditAndBiz {
  private static final Logger audit = LoggerFactory.getLogger("audit");
  private static final Logger log = LoggerFactory.getLogger(AuditAndBiz.class);

  public void grantRole(String userId, String role) {
    audit.info("audit.role.granted userId={} role={}", userId, role);
  }
  public void orderPaid(String orderId, long amount) {
    log.info("event.order.paid id={} amountMinor={} currency={}", orderId, amount, "RUB");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.goals

import org.slf4j.LoggerFactory

class AuditAndBiz {
    private val audit = LoggerFactory.getLogger("audit")
    private val log = LoggerFactory.getLogger(javaClass)

    fun grantRole(userId: String, role: String) {
        audit.info("audit.role.granted userId={} role={}", userId, role)
    }
    fun orderPaid(orderId: String, amount: Long) {
        log.info("event.order.paid id={} amountMinor={} currency={}", orderId, amount, "RUB")
    }
}
```

---

## Логи vs метрики vs трейсинг: логи — детали события; метрики — агрегаты и тренды; трейсинг — путь запроса по сервисам

**Логи** — дискретные записи о событиях; богаты контекстом, но их много, они «тяжёлые» и пост-фактум. Хорошо подходят для расследований и аудита. Их читают **после** того, как метрика вспыхнула.

**Метрики** — агрегаты (счётчики, таймеры, гистограммы). Они лёгкие и дешёвые, идеально подходят для алёртинга и графиков. Плохие для «почему?», отличные для «сколько и когда». В Spring — это Micrometer: единый API с разными реестрами (Prometheus, OTLP и т.п.).

**Трейсинг** — распределённые «следы» запроса: дерево спанов с таймингами и ошибками. Это мост между метриками и логами: видите длинный спан внешнего клиента — идёте в логи этого места. В Boot 3 — Micrometer Tracing + OTEL экспортер.

Связка тройки: **метрики** триггерят внимание, **трейсинг** показывает «где», **логи** объясняют «почему». Каждая часть дополняет другую; нельзя заменить всё логами или метриками.

Важно помнить о цене. Метрики почти бесплатны, трейсинг дороже (сэмплинг!), логи — самые тяжёлые. Балансируйте: метрики собирайте всегда; трейсинг включайте с разумным сэмплингом; логи — структурно и без болтовни.

И ещё: все три должны быть **коррелируемы**. В логах — traceId/spanId, в метриках — теги service/env/version, в трейсинге — те же атрибуты. Это делает «прыжки» между системами мгновенными.

**Код (Java): Micrometer + таймер HTTP**

```java
package demo.compare;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

@Service
public class CalcService {
  private final Timer timer;

  public CalcService(MeterRegistry registry) {
    this.timer = Timer.builder("calc.process").description("processing time").register(registry);
  }

  public int process(int x) {
    return timer.record(() -> x * 2);
  }
}
```

**Код (Kotlin): извлечение traceId для логов**

```kotlin
package demo.compare

import io.micrometer.tracing.Tracer
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component

@Component
class TraceAwareLogger(private val tracer: Tracer) {
    private val log = LoggerFactory.getLogger(javaClass)
    fun logWithTrace(msg: String) {
        val traceId = tracer.currentSpan()?.context()?.traceId() ?: "NA"
        log.info("msg={} traceId={}", msg, traceId)
    }
}
```

Gradle (Groovy/Kotlin) — наблюдаемость:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
  implementation 'io.micrometer:micrometer-tracing-bridge-otel'
  runtimeOnly 'io.opentelemetry:opentelemetry-exporter-otlp'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  runtimeOnly("io.micrometer:micrometer-registry-prometheus")
  implementation("io.micrometer:micrometer-tracing-bridge-otel")
  runtimeOnly("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

---

## Минимальный набор для любого сервиса: структурные логи + базовые метрики HTTP + health

Стартовый чек-лист: 1) включить структурные логи JSON; 2) включить Actuator `/health` и базовые метрики (HTTP сервер/клиент, JVM, БД); 3) пометить логи и метрики тегами service/env/version; 4) записывать traceId в каждый лог.

Базовые HTTP-метрики приходят «бесплатно» с `spring-boot-starter-actuator`. Они включают счётчики по кодам и таймеры по путям. Для Prometheus достаточно добавить реестр и открыть эндпоинт `/actuator/prometheus`.

Health — это «сигнал готовности» и «живости». Liveness говорит «процесс ок»; Readiness — «готов принимать трафик». В микросервисах это часть стратегии деплоя (readiness управляет балансировкой).

Структурные логи — это не только формат, но и **набор полей**: timestamp, level, logger, thread, message, service, env, version, traceId/spanId. Без traceId у вас «разорвана» цепочка между логами и трейсингом.

Сразу отделяйте access-логи (входящие HTTP) в свой аппендер/инструмент. Они более «шумные» и часто потребляют большую часть объёма. Бизнес-логи должны оставаться читаемыми.

Наконец, включите `/actuator/info` и передавайте туда git sha/версию — это позволит по логу сразу понять, «какой двоичный код» работает на инстансе.

**application.yml — минимальный контур**

```yaml
spring:
  application:
    name: payments
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      probes:
        enabled: true
```

**Код (Java/Kotlin): простой контроллер для демонстрации метрик и логов**

```java
package demo.min;

import org.slf4j.*; import org.springframework.web.bind.annotation.*;

@RestController
public class HelloController {
  private static final Logger log = LoggerFactory.getLogger(HelloController.class);

  @GetMapping("/hello")
  public String hello(@RequestParam String name) {
    log.info("hello.request name={}", name);
    return "Hello " + name;
  }
}
```

```kotlin
package demo.min

import org.slf4j.LoggerFactory
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController

@RestController
class HelloController {
    private val log = LoggerFactory.getLogger(javaClass)
    @GetMapping("/hello")
    fun hello(@RequestParam name: String) : String {
        log.info("hello.request name={}", name)
        return "Hello $name"
    }
}
```

---

# 2. Уровни логирования и политика уровней

## TRACE/DEBUG/INFO/WARN/ERROR: чёткие критерии применения; в проде DEBUG выключен по умолчанию

**TRACE** — сверхдетальные внутренности (побайтные дампы, ветвления алгоритмов). Обычно выключен везде. Включать — только локально и точечно.

**DEBUG** — подробности для разработчика: параметры вызовов, результаты промежуточных шагов, выбор веток. Полезен на dev/stage, **в проде по умолчанию выключен**.

**INFO** — значимые «нормальные» события: старт/останов, бизнес-события («заказ оплачен»), конфигурация на старте. INFO — ваш основной рабочий уровень.

**WARN** — временные сбои, деградации, ретраи, отказоустойчивые фэйлы (например, таймаут к кэшу, но запрос обслужён). WARN — маркёр «обратить внимание».

**ERROR** — реальный сбой, при котором операция не выполнена, данные потеряны или нарушен контракт. ERROR должен будить алёртами, поэтому экономно и осмысленно.

Смысл уровней — **сигналы**. Ошибка пользователя (400 Bad Request) — это не ERROR, а INFO/WARN с контекстом. А вот 500/таймаут/исключение парсинга — ERROR с stacktrace.

**Код (Java): разные уровни**

```java
package demo.levels;

import org.slf4j.*;

public class LevelDemo {
  private static final Logger log = LoggerFactory.getLogger(LevelDemo.class);

  public void process(String input) {
    log.debug("processing start input={}", input);
    if (input == null) { log.warn("empty input"); return; }
    try {
      // ...
      log.info("processed ok id={}", 42);
    } catch (RuntimeException ex) {
      log.error("processing failed input={}", input, ex);
    }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.levels

import org.slf4j.LoggerFactory

class LevelDemo {
    private val log = LoggerFactory.getLogger(javaClass)
    fun process(input: String?) {
        log.debug("processing start input={}", input)
        if (input == null) { log.warn("empty input"); return }
        try {
            log.info("processed ok id={}", 42)
        } catch (ex: RuntimeException) {
            log.error("processing failed input={}", input, ex)
        }
    }
}
```

---

## Политика повышения уровня: ошибка клиента ≠ ERROR, временные сбои — WARN с контекстом

HTTP 4xx — не вина сервера. Логируйте их как INFO (или WARN, если подозрительная активность). ERROR для 4xx приведёт к «красному» продакшн-каналу без реальных аварий. Включайте в сообщение идентификаторы запроса/пользователя.

Временные сбои (таймаут внешнего API, падение кэша) — **WARN** с ретраями и таймингами. Если у вас есть fallback и пользователь не пострадал, это не ERROR. Но важно иметь **метрики** таких WARN, чтобы видеть тренды и не пропустить системную деградацию.

ERROR — когда вы реально не выполнили контракт: не записали в БД, не доставили событие, транзакция откатилась. Тут нужен stacktrace, бизнес-контекст и traceId. Такие логи должны попадать в алёртинг.

Нельзя «поднимать уровень» для привлечения внимания. Если нужен сигнал — делайте метрику/алёрт. Уровень лога — это семантика, а не система оповещения.

Хорошая практика — зафиксировать в команде **таблицу**: «какие ситуации — какой уровень». Это убирает вариативность между разработчиками и уменьшает шум.

И помните про **объём**: INFO-«болтовня» в горячих участках (цикл, маппинг каждого элемента) сожрёт бюджет логов и замаскирует WARN/ERROR. Для таких мест — DEBUG.

**Код (Java): маппинг HTTP статусов → уровни**

```java
package demo.policy;

import org.slf4j.*; import org.springframework.http.*;
import org.springframework.web.server.ResponseStatusException;

public class HttpPolicy {
  private static final Logger log = LoggerFactory.getLogger(HttpPolicy.class);

  public void handleError(ResponseStatusException ex, String rid) {
    HttpStatus s = ex.getStatusCode();
    String msg = ex.getReason();
    if (s.is4xxClientError()) {
      log.info("client.error status={} reason={} requestId={}", s.value(), msg, rid);
    } else {
      log.error("server.error status={} reason={} requestId={}", s.value(), msg, rid, ex);
    }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.policy

import org.slf4j.LoggerFactory
import org.springframework.http.HttpStatus
import org.springframework.web.server.ResponseStatusException

class HttpPolicy {
    private val log = LoggerFactory.getLogger(javaClass)
    fun handleError(ex: ResponseStatusException, rid: String) {
        val s: HttpStatus = ex.statusCode
        val msg = ex.reason
        if (s.is4xxClientError) {
            log.info("client.error status={} reason={} requestId={}", s.value(), msg, rid)
        } else {
            log.error("server.error status={} reason={} requestId={}", s.value(), msg, rid, ex)
        }
    }
}
```

---

## Переключение уровней на лету (в том числе через Actuator) и контроль шумности

Иногда нужна деталь на проде «здесь и сейчас». Включать DEBUG глобально — плохая идея. В Spring Boot есть `/actuator/loggers`: можно поднять уровень **для конкретного логгера/пакета** на время расследования, затем вернуть назад.

Это безопаснее, чем менять конфиги и перезапускать сервис. Платформа (K8s/облако) может даже автоматизировать «включить DEBUG на 10 минут». В любом случае фиксируйте такие операции в runbook — чтобы их не забывали отключать.

Контроль шумности начинается с **паттернов аппендеров**. Для JSON-логов храните только важные поля; для dev-консоли можно оставить человекочитаемый формат. Access-логи — в отдельный поток.

Не забывайте про AsyncAppender в Logback: он разгружает горячие пути, но требует буфера и понимания «что будет, если переполнится». Для продового сервиса это часто оправдано.

Отдельная тема — **rate limit** логов в «болтливых» местах. Иногда лучше логировать каждую N-ю запись или агрегировать счётчики, чем заливать гигабайты повторяющихся WARN.

И, конечно, **маскирование**. Добавьте фильтры, которые вычищают токены/пароли, прежде чем сообщение попадёт в аппендер. Это удобно сделать на уровне Logback или через собственный `TurboFilter`.

**application.yml — экспозиция loggers**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: loggers
```

**Код (Java): программная смена уровня через LoggingSystem**

```java
package demo.runtime;

import org.springframework.boot.logging.LogLevel;
import org.springframework.boot.logging.LoggingSystem;
import org.springframework.stereotype.Service;

@Service
public class LogTuner {
  private final LoggingSystem loggingSystem;
  public LogTuner(LoggingSystem loggingSystem) { this.loggingSystem = loggingSystem; }
  public void setDebug(String loggerName) {
    loggingSystem.setLogLevel(loggerName, LogLevel.DEBUG);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.runtime

import org.springframework.boot.logging.LogLevel
import org.springframework.boot.logging.LoggingSystem
import org.springframework.stereotype.Service

@Service
class LogTuner(private val loggingSystem: LoggingSystem) {
    fun setDebug(loggerName: String) {
        loggingSystem.setLogLevel(loggerName, LogLevel.DEBUG)
    }
}
```

---

# 3. Структурные логи (JSON) и состав полей

## Обязательные поля: timestamp, level, logger, thread, message, service.name, version, env

Структурный лог — это **словарь полей** с чёткой схемой. Минимальный набор: `@timestamp`, `level`, `logger`, `thread`, `message`. К нему добавляем **инвариантные теги**: `service.name` (из `spring.application.name`), `version` (из `/actuator/info` или переменной окружения), `env` (dev/stage/prod), `region`/`zone` (если есть).

Почему это важно? Любая аналитика/алёртинг/поиск строится по этим ключам. Если названия «плавают», запросы становятся хрупкими. Стандартизируйте схему в команде/организации и проверьте, что парсер агента (fluent-bit/filebeat) ожидает именно её.

Поле `message` — для «человеческого» резюме. Не пытайтесь запихнуть всю структуру туда; бизнес-атрибуты — отдельными полями (через MDC или готовый JSON-энкодер). Это экономит когнитивное усилие на чтение.

Добавьте `traceId`/`spanId` — без них теряется связность между логами и трейсингом. Если трейсинг пока не включён — генерируйте свой correlationId, но в будущем лучше мигрировать на официальный `traceparent`.

Версию (`version`) и git sha удобно подставлять в логгер через переменные среды — тогда в каждом сообщении видно, какой именно бинарник пишет.

И, наконец, придерживайтесь стабильности. Если меняете схему — делайте это аккуратно и версионируйте (например, поле `schemaVersion`), чтобы парсеры и дашборды не ломались.

**Logback JSON c обязательными полями**

```xml
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp/>
        <pattern><pattern>{"level":"%level","logger":"%logger","thread":"%thread","service":"${spring.application.name:-app}","version":"${APP_VERSION:-dev}","env":"${APP_ENV:-local}","message":"%message"}</pattern></pattern>
        <mdc/>
        <stackTrace/>
      </providers>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="JSON"/></root>
</configuration>
```

**Код (Java/Kotlin): лог с бизнес-полями как частью message (дополнительно к MDC)**

```java
Logger log = org.slf4j.LoggerFactory.getLogger("biz");
log.info("order.created id={} amountMinor={} currency={}", "ORD-1", 5000, "RUB");
```

```kotlin
val log = org.slf4j.LoggerFactory.getLogger("biz")
log.info("order.created id={} amountMinor={} currency={}", "ORD-1", 5000, "RUB")
```

---

## Корреляция: traceId/spanId, внешние correlationId (если есть), requestId

Для **сквозного** анализа вам нужен идентификатор запроса, проходящий через все сервисы. В современном мире — это `traceId`/`spanId` из OpenTelemetry. Spring Boot с Micrometer Tracing автоматически прокидывает их через HTTP-заголовки (`traceparent`) и кладёт в MDC.

Если у вас есть свой внешний `X-Correlation-Id`, не конфликтуйте с трейсингом: сохраняйте оба. В логах держите `traceId` как основной, а `correlationId` — как дополнительный атрибут (часто полезно для фронтов/партнёров, которые не знают про OTEL).

MDC (Mapped Diagnostic Context) — класс SLF4J, который «цепляет» набор ключей к текущему потоку/реактивному контексту. Любой логгер добавит эти ключи в запись. В MVC их удобно устанавливать в `OncePerRequestFilter`, в WebFlux — в `WebFilter`.

Важно не забывать **очистку** MDC (в фильтрах `finally`), чтобы ключи не «протекали» между запросами — особенно в thread-pool.

В реактивном стеке используйте `Context` и готовые фильтры из Micrometer Tracing — там прокидывание уже реализовано. В императивном — проставьте `traceId`/`spanId` вручную, если Micrometer ещё не используется.

И, наконец, возвращайте `traceId` в ответ клиенту (заголовком). Тогда при тикете «у меня 500» поддержка быстро найдёт этот запрос.

**Код (Java): фильтр для correlationId + MDC**

```java
package demo.corr;

import jakarta.servlet.*; import jakarta.servlet.http.*; import java.io.IOException;
import org.slf4j.MDC; import org.springframework.stereotype.Component;
import java.util.UUID;

@Component
public class CorrelationFilter implements Filter {
  public static final String HDR = "X-Correlation-Id";
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    HttpServletResponse w = (HttpServletResponse) res;
    String cid = r.getHeader(HDR);
    if (cid == null || cid.isBlank()) cid = UUID.randomUUID().toString();
    MDC.put("correlationId", cid);
    w.setHeader(HDR, cid);
    try { chain.doFilter(req, res); }
    finally { MDC.remove("correlationId"); }
  }
}
```

**Код (Kotlin): получение traceId из Tracer и запись в лог**

```kotlin
package demo.corr

import io.micrometer.tracing.Tracer
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component

@Component
class TraceLogger(private val tracer: Tracer) {
    private val log = LoggerFactory.getLogger(javaClass)
    fun log(msg: String) {
        val traceId = tracer.currentSpan()?.context()?.traceId() ?: "NA"
        log.info("msg={} traceId={}", msg, traceId)
    }
}
```

---

## Контекст: ключевые business-атрибуты (id операции, сумма, статус), но без ПДн и секретов

Хороший лог — это **контекст + действие**. Контекст — это ключевые идентификаторы: `orderId`, `userId` (псевдонимизированный), сумма в minor-единицах, статус, способ оплаты, регион. Действие — «что произошло»: `order.paid`, `refund.failed`, `kafka.retry`.

Нельзя логировать ПДн и секреты: полные номера карт, CVV, пароли, токены доступа. Если бизнесу нужна отладка, используйте маскирование (последние 4 цифры), хеширование или агрегаты (сумма, тип метода), а не сырые значения.

Контекст удобнее прокидывать через **MDC**. Тогда любые логи в глубине стека автоматически получат нужные поля — без передачи десятка аргументов по всем слоям. Главное — ставить и удалять аккуратно.

Не перегружайте лог «всем подряд». Вынесите 3–7 ключевых атрибутов — остальное можно найти через трейсинг/метрики или отдельные запросы к БД. Читаемость важнее.

Старайтесь использовать **стабильные** названия полей (`orderId`, `amountMinor`, `status`). Тогда KQL/LogQL запросы не будут ломаться после рефакторинга.

И ещё — логируйте **смысл**, а не «мы пошли туда-то». Сообщение «refund failed» без причины мало полезно; «refund failed — gateway timeout, attempt=2, backoffMs=200» — то, что нужно.

**Код (Java): установка бизнес-контекста в MDC**

```java
package demo.ctx;

import org.slf4j.Logger; import org.slf4j.LoggerFactory; import org.slf4j.MDC;

public class BizLogger {
  private static final Logger log = LoggerFactory.getLogger(BizLogger.class);

  public void refund(String orderId, long amountMinor) {
    MDC.put("orderId", orderId);
    MDC.put("amountMinor", String.valueOf(amountMinor));
    try {
      log.info("refund.requested");
      // ...
      log.info("refund.completed");
    } catch (Exception ex) {
      log.error("refund.failed", ex);
    } finally {
      MDC.remove("orderId"); MDC.remove("amountMinor");
    }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.ctx

import org.slf4j.LoggerFactory
import org.slf4j.MDC

class BizLogger {
    private val log = LoggerFactory.getLogger(javaClass)
    fun refund(orderId: String, amountMinor: Long) {
        MDC.put("orderId", orderId)
        MDC.put("amountMinor", amountMinor.toString())
        try {
            log.info("refund.requested")
            log.info("refund.completed")
        } catch (ex: Exception) {
            log.error("refund.failed", ex)
        } finally {
            MDC.remove("orderId"); MDC.remove("amountMinor")
        }
    }
}
```

---

## Форматирование исключений: stacktrace в отдельном поле, сокращение «шума» общих фреймов

Стек-трейсы полезны, но «шумные». В JSON-логах правильно выводить их **в отдельном поле** (`stack_trace`) — так парсеры не будут ломать документ. Logstash-encoder делает это из коробки (`<stackTrace/>` провайдер).

Сокращайте «общие» фреймы (JDK/Spring инфраструктура), оставляя суть. У Logback есть `ShortenedThrowableConverter`, который укорачивает пакеты по маскам. Это экономит место и делает логи читабельнее.

Не превращайте исключения в «строки». `log.error(".. {}", ex.toString())` теряет стек. Используйте перегрузку `log.error(msg, ex)` — так форматтер положит стек в корректное поле.

Повторяющиеся ERROR по одному и тому же traceId иногда стоит **дедуплицировать** на уровне алёртинга или агентов (но не «заглушать» в коде). Иначе один шумный участок «забивает» канал.

Для «ожидаемых» исключений (например, `TimeoutException` при ретраях) используйте WARN, и логируйте **первое** и **последнее** с полезным контекстом, а не каждую попытку.

Всегда добавляйте **ключи** к ошибке: `orderId`, `attempt`, `endpoint`. Это сокращает путь от лога к фиксу.

**Logback: короткий stacktrace**

```xml
<configuration>
  <conversionRule conversionWord="shortThrowable" converterClass="net.logstash.logback.stacktrace.ShortenedThrowableConverter"/>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp/><mdc/>
        <stackTrace class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
          <rootCauseFirst>true</rootCauseFirst>
          <maxDepthPerThrowable>30</maxDepthPerThrowable>
          <shortenedClassNameLength>20</shortenedClassNameLength>
        </stackTrace>
      </providers>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="JSON"/></root>
</configuration>
```

**Код (Java/Kotlin): корректное логирование исключения**

```java
try {
  // ...
} catch (Exception ex) {
  org.slf4j.LoggerFactory.getLogger("err").error("refund.failed orderId={} attempt={}", "ORD-1", 2, ex);
}
```

```kotlin
try {
    // ...
} catch (ex: Exception) {
    org.slf4j.LoggerFactory.getLogger("err").error("refund.failed orderId={} attempt={}", "ORD-1", 2, ex)
}
```

# 4. Конфигурация логгера в Spring Boot

## Logback как дефолт: `logback-spring.xml`/`application.yml` для уровней и appenders

Spring Boot по умолчанию использует Logback и маршрутизирует все вызовы `org.slf4j.Logger` в Logback. Это даёт быстрый старт без ручной настройки: консольный аппендер, читаемый паттерн, уровни из `application.yml`. Но в проде почти всегда нужен свой `logback-spring.xml`, потому что нам важны JSON-формат, отдельные каналы и асинхронность.

Файл с именем **`logback-spring.xml`** загружается раньше контекста, понимает Spring-плейсхолдеры `${…}` и профили `<springProfile>`. Это удобнее, чем «сырой» `logback.xml`, где нет доступа к свойствам Spring. Рекомендация: всегда использовать именно `logback-spring.xml`.

Уровни логирования можно задать как в `logback-spring.xml`, так и в `application.yml` через секцию `logging.level`. Второй способ — для грубой регулировки (например, включить DEBUG для пакета) без правки XML. Первый — для топологии аппендеров/энкодеров.

Для локальной разработки обычно достаточно консольного аппендера; на стендах полезен **двойной** вывод: в консоль (для сбора агентом) и в rolling file (на случай сбоев агента). При этом запись в файл должна быть отключаемой флагом окружения.

Важно: любые сессионные параметры (service.name, env, version) лучше подставлять из свойств Spring. Это обеспечит единообразие across сервисов и облегчит поиск в централизованном хранилище.

Договоритесь о **минимальной схеме**: `timestamp`, `level`, `logger`, `thread`, `message`, `service`, `env`, `version`. Всё остальное — через MDC или отдельные поля JSON-энкодера. Это позволит с первого дня строить дашборды и фильтры.

**Пример `logback-spring.xml` (консоль + уровни из YAML):**

```xml
<configuration>
  <property name="SERVICE" value="${spring.application.name:-app}"/>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{ISO8601} %-5level [%thread] %logger - %msg %n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
  </root>
</configuration>
```

**`application.yml` — уровни:**

```yaml
spring:
  application:
    name: payments
logging:
  level:
    root: INFO
    com.example.demo: DEBUG
```

**Код (Java): простой лог с уровнем**

```java
package demo.logback;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;

@SpringBootApplication
public class LogbackApp {
  private static final Logger log = LoggerFactory.getLogger(LogbackApp.class);
  public static void main(String[] args) {
    SpringApplication.run(LogbackApp.class, args);
    log.info("Service={} started", "payments");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.logback

import org.slf4j.LoggerFactory
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class LogbackApp

fun main(args: Array<String>) {
    val log = LoggerFactory.getLogger("demo.logback.LogbackApp")
    runApplication<LogbackApp>(*args)
    log.info("Service={} started", "payments")
}
```

Gradle зависимости (Groovy/Kotlin) — Logback уже в стартере:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

## Console + RollingFile (dev/стенд) и AsyncAppender для неблокирующей записи

Консольный вывод — лучший источник правды в контейнере: агент забирает stdout/stderr и отправляет в централизованное хранилище. Но на стендах иногда полезен **запасной** RollingFileAppender: если агент «упал», у вас останется локальный лог за последние N дней.

RollingFileAppender управляет ротацией по размеру/времени и лимитом истории. Это снимает вопрос ручного логрота, но не отменяет правило писать в stdout. В Kubernetes файлы внутри контейнера могут пропасть при рестарте — поэтому хранение логов должно быть вне контейнера (агент/volume).

Блокирующая запись в аппендер может задержать горячие пути. Чтобы разгрузить поток запроса, используем **AsyncAppender**: он буферизует события и пишет их фоном. Нужно выставить ёмкость буфера и понять поведение при переполнении (обычно «бросить» низкоуровневые записи).

Не стоит делать все аппендеры асинхронными: достаточно один AsyncAppender, внутри которого ссылки на «реальные» аппендеры (console/rolling). Так проще управлять потоками и диагностировать потерю событий.

Важное правило: и в dev, и в prod структура логов должна совпадать, меняется лишь транспорт/асинхронность. Это снижает «сюрпризы» при релизе и делает репродукции надёжными.

Проверяйте работу буфера под нагрузкой — простой нагрузочный тест, который пишет много INFO/DEBUG, покажет, хватает ли размеров очереди и не теряются ли события.

**`logback-spring.xml` — Console + Rolling + Async:**

```xml
<configuration>
  <property name="LOG_PATH" value="${LOG_PATH:-${java.io.tmpdir}/app-logs}"/>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder><pattern>%d %-5level %logger - %msg%n</pattern></encoder>
  </appender>
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_PATH}/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${LOG_PATH}/app-%d{yyyy-MM-dd}.gz</fileNamePattern>
      <maxHistory>7</maxHistory>
      <totalSizeCap>1GB</totalSizeCap>
    </rollingPolicy>
    <encoder><pattern>%d %-5level %logger - %msg%n</pattern></encoder>
  </appender>
  <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>8192</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <includeCallerData>false</includeCallerData>
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
  </appender>
  <root level="INFO"><appender-ref ref="ASYNC"/></root>
</configuration>
```

**Код (Java): нагрузочный логгер**

```java
package demo.async;

import org.slf4j.*;
import org.springframework.stereotype.Component;

@Component
public class BurstLogger {
  private static final Logger log = LoggerFactory.getLogger(BurstLogger.class);
  public void burst(int n) {
    for (int i = 0; i < n; i++) log.info("burst message {}", i);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.async

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component

@Component
class BurstLogger {
    private val log = LoggerFactory.getLogger(javaClass)
    fun burst(n: Int) {
        repeat(n) { log.info("burst message {}", it) }
    }
}
```

---

## Паттерны/энкодеры для JSON (логгер, метки, поля MDC)

Человекочитаемые паттерны хороши в консоли, но для прод-стека нужен **строго структурированный JSON**. Самый удобный способ — `logstash-logback-encoder`. Он собирает JSON из «провайдеров»: `timestamp`, `mdc`, `pattern`, `stackTrace` и т.д.

Мы формируем «каркас» полей (service, env, version) через `<pattern>`, а переменные контекста (traceId, correlationId, business-id) попадают через `<mdc/>`. Это даёт гибкость: одно и то же сообщение правильно парсится агентами и читается человеком.

Encoder легко расширяется. Можно добавить провайдер `context` с произвольной парой «ключ–значение» или кастомные поля (`<pattern>{"appInstance":"${HOSTNAME:-unknown}"}</pattern>`). Главное — не перегружать лог: 10–15 полей обычно достаточно.

Stacktrace добавляем отдельным полем — это упрощает фильтрацию. Используйте `ShortenedThrowableConverter`, чтобы обрезать «шумные» фреймы.

Стабильность схемы важна для дашбордов. Любое изменение меняет запросы/индикаторы. Если вы меняете ключи — сделайте миграцию в парсерах и сохраните обратную совместимость на время.

Не забывайте про локаль/часовой пояс. Всегда используйте ISO8601/UTC — так легче склеивать события между сервисами и регионами.

**Gradle зависимости для JSON (Groovy/Kotlin):**

```groovy
dependencies { implementation 'net.logstash.logback:logstash-logback-encoder:7.4' }
```

```kotlin
dependencies { implementation("net.logstash.logback:logstash-logback-encoder:7.4") }
```

**`logback-spring.xml` — JSON-энкодер:**

```xml
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp/>
        <pattern><pattern>{"service":"${spring.application.name:-app}","env":"${APP_ENV:-local}","version":"${APP_VERSION:-dev}","message":"%message","logger":"%logger","thread":"%thread","level":"%level"}</pattern></pattern>
        <mdc/>
        <stackTrace/>
      </providers>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="JSON"/></root>
</configuration>
```

**Код (Java): заполнение MDC полей**

```java
package demo.json;

import org.slf4j.Logger; import org.slf4j.LoggerFactory; import org.slf4j.MDC;
import org.springframework.stereotype.Service;

@Service
public class JsonLogService {
  private static final Logger log = LoggerFactory.getLogger(JsonLogService.class);
  public void logOrder(String orderId) {
    MDC.put("orderId", orderId);
    try { log.info("order.created"); }
    finally { MDC.clear(); }
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.json

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.stereotype.Service

@Service
class JsonLogService {
    private val log = LoggerFactory.getLogger(javaClass)
    fun logOrder(orderId: String) {
        MDC.put("orderId", orderId)
        try { log.info("order.created") } finally { MDC.clear() }
    }
}
```

---

## Разделение технических логов и access-логов (входящие HTTP-запросы) в разные аппендеры

Access-логи — потоковые и шумные. Смешивать их с бизнес-логами неудобно: тонут сигналы, растут счета за хранение. Лучший подход — **отдельный логгер/аппендер** под имя `access`, с простым JSON-шаблоном и коротким retention.

В MVC можно сделать `OncePerRequestFilter`, в WebFlux — `WebFilter`, которые пишут в логгер `access`. В Logback настраиваем отдельный аппендер `ACCESS_JSON` и категорию `logger name="access"`.

Access-лог должен содержать минимум: метод, путь, статус, время обработки, traceId, client-ip. Не пишите тела запросов/ответов — риск утечек и взрыв объёма.

Раздельные файлы помогают операторам: быстро посмотреть RPS/коды «сырыми» глазами, не копаясь в доменных сообщениях. А парсерам — настраивать разные пайплайны и retention.

При необходимости можно включать access-лог только на стендах или по флагу свойства. На проде он часто собирается Nginx/Ingress’ом — дублирование не нужно.

Держите имена полей согласованными с бизнес-логами: `service`, `env`, `traceId`. Это упрощает корреляцию и объединённые дашборды.

**`logback-spring.xml` — отдельный логгер `access`:**

```xml
<configuration>
  <appender name="ACCESS_JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp/>
        <pattern><pattern>{"category":"access","service":"${spring.application.name:-app}","message":"%message","level":"%level"}</pattern></pattern>
        <mdc/>
      </providers>
    </encoder>
  </appender>
  <logger name="access" level="INFO" additivity="false">
    <appender-ref ref="ACCESS_JSON"/>
  </logger>
  <root level="INFO">
    <appender-ref ref="ACCESS_JSON"/> <!-- опционально -->
  </root>
</configuration>
```

**Код (Java): фильтр access-лога**

```java
package demo.access;

import jakarta.servlet.*; import jakarta.servlet.http.*; import java.io.IOException;
import org.slf4j.Logger; import org.slf4j.LoggerFactory;
import org.slf4j.MDC; import org.springframework.stereotype.Component;

@Component
public class AccessLogFilter implements Filter {
  private static final Logger access = LoggerFactory.getLogger("access");
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req; long t0 = System.nanoTime();
    try { chain.doFilter(req, res); }
    finally {
      long tookMs = (System.nanoTime()-t0)/1_000_000;
      String traceId = MDC.get("traceId");
      access.info("method={} path={} tookMs={} traceId={}", r.getMethod(), r.getRequestURI(), tookMs, traceId);
    }
  }
}
```

**Код (Kotlin): WebFlux-фильтр**

```kotlin
package demo.access

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
class AccessWebFilter : WebFilter {
    private val access = LoggerFactory.getLogger("access")
    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        val start = System.nanoTime()
        return chain.filter(exchange).doFinally {
            val took = (System.nanoTime() - start) / 1_000_000
            access.info("method={} path={} tookMs={} traceId={}",
                exchange.request.method, exchange.request.path.value(), took, MDC.get("traceId"))
        }
    }
}
```

---

# 5. Безопасность логов и маскирование

## Маска секретов: токены, пароли, номера карт; регулярки/фильтры до записи

Логи часто уходят из вашего периметра в SaaS-хранилища. Любая утечка секретов в лог — это инцидент. Минимум защиты — **маскирование** чувствительных полей: токены, пароли, номера карт, ключи API.

Маскирование делается до записи: фильтр «перехватывает» событие, заменяет совпадения по регуляркам на `***` и только потом отдаёт в энкодер. В Logback это реализуемо через `TurboFilter` или через кастомный `CompositeJsonEncoder` провайдер.

Регулярки должны быть узкими, чтобы не «пожирать» полезный контент. Например, для карт — PAN с Luhn-проверкой, для токенов — чёткие префиксы (`Bearer`, `Basic`, `x-token=`).

Проверяйте маскирование в тестах: берите реальный «грязный» лог и убеждайтесь, что все чувствительные участки скрыты. Это важнее «идеального» покрытия regexp’ами.

Важно: **не логируйте** тела запросов/ответов без whitelisting (см. следующий пункт). Ни один фильтр не справится с мегабайтами JSON от внешнего партнёра в режиме DEBUG.

Документируйте политику: список запрещённых полей, ответственность за нарушения, процедуры ревизии. Чем понятнее правила, тем меньше «сюрпризов» в инцидентах.

**Код (Java): TurboFilter для маскирования**

```java
package demo.mask;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.classic.turbo.TurboFilter;
import ch.qos.logback.core.spi.FilterReply;
import java.util.regex.Pattern;

public class MaskingTurboFilter extends TurboFilter {
  private final Pattern token = Pattern.compile("(Bearer|Basic)\\s+[A-Za-z0-9._-]+");
  private final Pattern card = Pattern.compile("\\b(\\d{4}[- ]?){3}\\d{4}\\b");
  @Override
  public FilterReply decide(org.slf4j.Marker marker, ch.qos.logback.classic.Logger logger,
                            ch.qos.logback.classic.Level level, String format, Object[] params, Throwable t) {
    if (format != null) {
      String masked = token.matcher(format).replaceAll("$1 ***");
      masked = card.matcher(masked).replaceAll("****-****-****-****");
      if (!masked.equals(format)) {
        if (params == null) logger.callAppenders(new LogEventProxy(level, masked));
      }
    }
    return FilterReply.NEUTRAL;
  }
  static class LogEventProxy extends ch.qos.logback.classic.spi.LoggingEvent {
    LogEventProxy(ch.qos.logback.classic.Level level, String msg) {
      setLevel(level); setMessage(msg);
    }
  }
}
```

**`logback-spring.xml` — подключение фильтра:**

```xml
<configuration>
  <turboFilter class="demo.mask.MaskingTurboFilter"/>
  <!-- остальная конфигурация -->
</configuration>
```

**Код (Kotlin): маскирование строки утилитой перед логом (доп. защита)**

```kotlin
package demo.mask

import org.slf4j.LoggerFactory

object Mask {
    private val token = Regex("(Bearer|Basic)\\s+[A-Za-z0-9._-]+")
    private val card = Regex("\\b(\\d{4}[- ]?){3}\\d{4}\\b")
    fun apply(s: String) = s.replace(token, "$1 ***").replace(card, "****-****-****-****")
}

class MaskedLogger {
    private val log = LoggerFactory.getLogger(javaClass)
    fun safeInfo(msg: String) = log.info(Mask.apply(msg))
}
```

---

## Запрет логирования тел запросов/ответов по умолчанию; whitelist полей для бизнес-событий

Тела HTTP-запросов и ответов часто содержат ПДн и секреты. Логировать их целиком — плохая практика. По умолчанию **запрещаем** логирование тел и включаем только белые списки полей, которые согласованы с безопасностью.

Если есть сильная потребность в body-логах (например, в интеграции с партнёром), лучше организовать **сниффер** на стенде или специальный флаг с ограничениями: лимит размера, маскирование и TTL хранения.

Для бизнес-событий готовьте явный DTO с whitelisted полями и логируйте именно этот объект. Это убережёт от случайного «залёта» секретов из доменной модели или внешнего DTO.

Опасная зона — ретраи/дедупликации. Не повторяйте логи с телами много раз; логируйте только «первый» и «последний» кадр, между ними — метрики и статусы.

Контракты с партнёрами часто требуют аудита. Ведите **отдельный** канал «событий аудита» с белым списком полей и долговременным хранением. Это не те же логи, что «технические».

Наконец, помните, что Logback-фильтры — защита второго уровня. Первый уровень — **культура**: разработчики не пишут тела в логи, а готовят «безопасные» summary.

**Код (Java): логируем белый список полей**

```java
package demo.whitelist;

import org.slf4j.*;
import org.springframework.stereotype.Service;

@Service
public class AuditLogger {
  private static final Logger audit = LoggerFactory.getLogger("audit");
  public record PaymentEvent(String orderId, long amountMinor, String currency, String status) {}
  public void paymentLogged(String orderId, long amountMinor, String currency) {
    audit.info("payment.event {}", new PaymentEvent(orderId, amountMinor, currency, "CAPTURED"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.whitelist

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class AuditLogger {
    private val audit = LoggerFactory.getLogger("audit")
    data class PaymentEvent(val orderId: String, val amountMinor: Long, val currency: String, val status: String)
    fun paymentLogged(orderId: String, amountMinor: Long, currency: String) {
        audit.info("payment.event {}", PaymentEvent(orderId, amountMinor, currency, "CAPTURED"))
    }
}
```

---

## Ограничение размера сообщений и частоты (rate limit) для «болтливых» точек

Даже без тел можно «утонуть» в миллионах INFO. Выручает **rate limiting**: пропускаем каждое N-е сообщение или не чаще X раз в секунду. В Logback это можно сделать обёрткой над логгером или с помощью `EvaluatorFilter`.

Простой подход — добавить «тихий» логгер, который считает события по ключу (например, по типу сообщения) и пишет только первые/каждые N. Это уменьшает нагрузку на CPU/IO и бюджет в хранилище.

Ещё вариант — агрегировать счётчики и писать итог раз в минуту: «validation.warn count=123 for last 60s». Это помогает увидеть тренд, не перегружая канал.

Не применяйте rate limit к ERROR: они должны доходить всегда. Для WARN используйте аккуратно — можно «заглушить» проблему. Делайте исключения по категориям.

Заранее договоритесь о «шумных» точках: валидация входных данных, poll-циклы, batch-итерации. Для них — только DEBUG и/или rate limit.

Тестируйте rate limit под средовой нагрузкой, чтобы не потерять важные сигналы. И обязательно ведите метрику «сколько отброшено», чтобы не работать «вслепую».

**Код (Java): простая rate-limit обёртка**

```java
package demo.rate;

import org.slf4j.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class RateLimitedLogger {
  private final Logger log;
  private final ConcurrentHashMap<String, AtomicLong> counters = new ConcurrentHashMap<>();
  private final long n;
  public RateLimitedLogger(Class<?> type, long n) { this.log = LoggerFactory.getLogger(type); this.n = n; }
  public void infoEvery(String key, String pattern, Object... args) {
    long c = counters.computeIfAbsent(key, k -> new AtomicLong()).incrementAndGet();
    if (c % n == 1L) log.info(pattern, args);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.rate

import org.slf4j.LoggerFactory
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.atomic.AtomicLong

class RateLimitedLogger(clazz: Class<*>, private val n: Long) {
    private val log = LoggerFactory.getLogger(clazz)
    private val counters = ConcurrentHashMap<String, AtomicLong>()
    fun infoEvery(key: String, pattern: String, vararg args: Any?) {
        val c = counters.computeIfAbsent(key) { AtomicLong() }.incrementAndGet()
        if (c % n == 1L) log.info(pattern, *args)
    }
}
```

---

## Политика хранения: срок, доступы, шифрование на диске/транзите

Хранение логов — зона ответственности платформы и безопасности. Ваша задача — **сигнализировать требования**: срок хранения (например, 7/30/90 дней), необходимость шифрования «на диске» (at-rest) и «в полёте» (in-transit), ограничения доступов и аудит чтения.

На уровне приложения мы задаём только то, что нам доступно: **ротация** (через RollingPolicy), **ограничение размера** (`totalSizeCap`), отсутствие PII/секретов. Всё остальное — настройки стека логирования (Fluent Bit/Filebeat → Kafka/ELK/Cloud Logging).

Если всё же используете локальные файлы на стендах, выставляйте `maxHistory` и `totalSizeCap`, чтобы не занять диск. И обеспечьте защиту доступа к каталогу логов (права/umask).

Шифрование на диске обычно реализует платформа (шифрованные диски/тома). В транзите используется TLS от агента до хранилища. Важно договориться с DevSecOps, что логи из прод-окружения идут только по шифрованным каналам.

В документации сервиса укажите, где искать логи, какие индексы/теги, какие права нужны и какова ретенция. Это уберёт «потери времени» в ночных инцидентах.

Контролируйте стоимость: логи — деньги. Уменьшайте болтовню, не пишите payload’ы, используйте агрегацию и сжатие (`.gz` в ротации).

**`logback-spring.xml` — политика ротации:**

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>${LOG_PATH:-/var/log/app}/app.json</file>
  <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    <fileNamePattern>${LOG_PATH:-/var/log/app}/app-%d{yyyy-MM-dd}.json.gz</fileNamePattern>
    <maxHistory>14</maxHistory>
    <totalSizeCap>2GB</totalSizeCap>
    <cleanHistoryOnStart>true</cleanHistoryOnStart>
  </rollingPolicy>
  <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
    <providers><timestamp/><mdc/><pattern><pattern>{"message":"%message"}</pattern></pattern></providers>
  </encoder>
</appender>
```

**Код (Java/Kotlin): запись «сигнального» сообщения старта/версии**

```java
package demo.retention;

import org.slf4j.*; import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener; import org.springframework.stereotype.Component;

@Component
public class StartupBanner {
  private static final Logger log = LoggerFactory.getLogger(StartupBanner.class);
  @EventListener(ApplicationReadyEvent.class)
  public void ready() { log.info("service.ready version={} env={}", System.getenv("APP_VERSION"), System.getenv("APP_ENV")); }
}
```

```kotlin
package demo.retention

import org.slf4j.LoggerFactory
import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class StartupBanner {
    private val log = LoggerFactory.getLogger(javaClass)
    @EventListener(ApplicationReadyEvent::class)
    fun ready() {
        log.info("service.ready version={} env={}", System.getenv("APP_VERSION"), System.getenv("APP_ENV"))
    }
}
```

---

# 6. Корреляция через MDC и базовая интеграция с трейсингом

## MDC (Mapped Diagnostic Context): прокидываем traceId/correlationId во все логи потока запроса

MDC — это карта «ключ → значение», привязанная к текущему потоку. Любой логгер добавит значения MDC в запись, если энкодер поддерживает `<mdc/>`. Это удобный способ не передавать `orderId`/`traceId` через десяток методов.

В императивном стекe (Servlet) MDC живёт в thread-local, поэтому **обязательно очищайте** его в фильтре после обработки запроса. Иначе данные «протекут» на следующий запрос, если потоки переиспользуются.

В реактивном стекe (WebFlux) MDC сам по себе не работает, потому что нет постоянного потока. Здесь используйте интеграцию Micrometer Tracing + Reactor Context, или библиотеки, которые «переносят» MDC в реактивный контекст. Проще — логировать traceId через `Tracer` и не полагаться на MDC.

Добавляйте в MDC не только `traceId`, но и ваш `correlationId`, если он используется на границе (например, приходит от фронта). Это позволит поддержке искать конкретные кейсы по внешнему идентификатору.

Держите список полей MDC короче: `traceId`, `spanId`, `correlationId`, `orderId`/`userId` по ситуации. Большие MDC раздувают каждую запись и бьют по I/O.

Проверьте, что JSON-энкодер выводит MDC **как объект** (`"mdc":{"traceId":"...","orderId":"..."}`) или встраивает его ключи в корень — фиксируйте формат для агентского парсера.

**Код (Java): Servlet-фильтр, управляющий MDC**

```java
package demo.mdc;

import jakarta.servlet.*; import jakarta.servlet.http.*; import java.io.IOException;
import org.slf4j.MDC; import org.springframework.stereotype.Component;
import java.util.UUID;

@Component
public class MdcFilter implements Filter {
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletRequest r = (HttpServletRequest) req;
    String cid = r.getHeader("X-Correlation-Id");
    if (cid == null || cid.isBlank()) cid = UUID.randomUUID().toString();
    MDC.put("correlationId", cid);
    try { chain.doFilter(req, res); }
    finally { MDC.clear(); }
  }
}
```

**Код (Kotlin): WebFlux — логируем traceId через Tracer**

```kotlin
package demo.mdc

import io.micrometer.tracing.Tracer
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component

@Component
class TraceAwareService(private val tracer: Tracer) {
    private val log = LoggerFactory.getLogger(javaClass)
    fun work() {
        val traceId = tracer.currentSpan()?.context()?.traceId() ?: "NA"
        log.info("do.work traceId={}", traceId)
    }
}
```

---

## Пропагация заголовков межсервисно (например, `traceparent`, `x-correlation-id`)

Корреляция не заканчивается на одном сервисе — заголовки нужно **передавать дальше**. Для OpenTelemetry это `traceparent` (и опционально `tracestate`). Плюс ваш `X-Correlation-Id`, если он участвует в пользовательских сценариях.

В Spring Boot 3 с Micrometer Tracing мост в RestClient/RestTemplate/WebClient уже есть: он добавит `traceparent` автоматически. Если нужен ещё и `X-Correlation-Id`, добавьте перехватчик/фильтр.

Пропагируйте только «белые» заголовки. Никогда не переносите `Authorization`/куки между доменами. Корреляция — это про трассировку, а не про аутентификацию.

Для Kafka/AMQP перенесите идентификатор в заголовки сообщений (`headers["traceparent"]`). Micrometer Tracing поддерживает и это, но следует убедиться, что ваш клиент/библиотека интегрированы.

Проверяйте, что внешний партнёр возвращает ваш `X-Correlation-Id` в ответ. Это облегчит обсуждение инцидентов — обе стороны видят одинаковый id.

Наконец, логируйте «дырки» в цепочке: если к вам пришёл запрос без `traceparent`, создайте новый спан и пометьте событие как «untraced_incoming=true» — это поможет анализировать «серые» зоны.

**Код (Java): RestClient с пропагацией correlation-id**

```java
package demo.propagate;

import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.web.client.RestClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.slf4j.MDC;

@Configuration
public class RestClientConfig {
  @Bean RestClient restClient() {
    ClientHttpRequestInterceptor corr = (req, body, exec) -> {
      String cid = MDC.get("correlationId");
      if (cid != null) req.getHeaders().add("X-Correlation-Id", cid);
      return exec.execute(req, body);
    };
    return RestClient.builder().requestInterceptor(corr).build();
  }
}
```

**Код (Kotlin): WebClient фильтр**

```kotlin
package demo.propagate

import org.slf4j.MDC
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.client.ExchangeFilterFunction
import org.springframework.web.reactive.function.client.WebClient

@Configuration
class WebClientConfig {
    @Bean
    fun webClient(): WebClient {
        val corr = ExchangeFilterFunction.ofRequestProcessor { req ->
            val cid = MDC.get("correlationId")
            val mutated = if (cid != null) req.attribute("X-Correlation-Id", cid) else req
            java.util.concurrent.CompletableFuture.completedFuture(mutated)
        }
        return WebClient.builder().filter(corr).build()
    }
}
```

---

## Как минимум: записывать traceId в каждый лог и в ответ клиенту — этого хватает для базовой трассировки

Даже без полноценного трейсинга «золотой минимум» — **traceId в каждом логе** и **возврат его клиенту**. Тогда любой инцидент можно связать от запроса до бэка и обратно.

Сервер — вставляет `traceId` в MDC (из трейсинга или генерирует), пишет его в логи и **возвращает заголовком** в ответе (`X-Trace-Id`/`traceparent`). Клиент — логирует `traceId` ответа и прикладывает его к тикету.

Внутри приложения это несколько строк в фильтре и контроллере. На фронте — middleware, который подхватывает заголовок и сохраняет id.

Согласуйте имя заголовка с фронтом/партнёрами. Если используете OTEL — безопаснее отправлять `traceparent`. Для людей можно дублировать «короткий» `X-Trace-Id`.

Не забывайте о CORS и «скрываемых» заголовках: чтобы браузер отдал `X-Trace-Id` клиентскому JS, он должен быть в списке `Access-Control-Expose-Headers`.

Наконец, добавьте в `/actuator/httptrace` аналог или отдельный эндпоинт «по traceId», если это облегчает поддержку. Но чаще достаточно фильтра по полю `traceId` в лог-хранилище.

**Код (Java): выдаём traceId в ответ**

```java
package demo.mintrace;

import jakarta.servlet.*; import jakarta.servlet.http.*; import java.io.IOException;
import org.slf4j.MDC; import org.springframework.stereotype.Component;

@Component
public class TraceHeaderFilter implements Filter {
  @Override public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
    HttpServletResponse w = (HttpServletResponse) res;
    String traceId = MDC.get("traceId");
    if (traceId != null) w.setHeader("X-Trace-Id", traceId);
    chain.doFilter(req, res);
  }
}
```

**Код (Kotlin): контроллер, возвращающий traceId**

```kotlin
package demo.mintrace

import io.micrometer.tracing.Tracer
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class TraceController(private val tracer: Tracer) {
    @GetMapping("/trace-id")
    fun trace(): Map<String, String> =
        mapOf("traceId" to (tracer.currentSpan()?.context()?.traceId() ?: "NA"))
}
```

---

# 7. Минимальные метрики для начала (Micrometer + Actuator)

## HTTP: количество запросов, коды, латентность (p50/p95)

Spring Boot Actuator автоматически включает HTTP-метрики сервера: количество запросов, коды ответов, латентность. Они экспортируются Micrometer’ом в выбранный реестр (например, Prometheus). Это базис для любого сервиса: вы видите RPS, долю 5xx, p95/p99 по маршрутам.

Включите `/actuator/metrics` и, для Prometheus, `/actuator/prometheus`. Этого достаточно, чтобы добавить сервис на общий дашборд. Правильно проставленные теги (`service`, `env`, `version`) дадут сравнимость между инстансами.

Метрики клиента (RestClient/WebClient) тоже включаются автоматически (через наблюдаемость Spring). Они показывают задержки и ошибки во внешних вызовах — отличный индикатор проблем «вне нас».

Гистограммы нужны, чтобы видеть **перцентили** (p50/p95/p99). Их включают явным свойством `management.metrics.distribution.percentiles-histogram.http.server.requests=true` и указывают интересующие перцентили.

Не забывайте про imprecision: p95 — приблизительное значение. Для точного анализа используйте трейсинг и логи. Но для алёртинга и SLO этого достаточно.

Сразу договоритесь о **SLO**: например, «HTTP p95 < 300мс, 5xx < 0.5%». Тогда алёрты будут не «по ощущениям», а по целям.

**`application.yml` — включаем метрики и перцентили:**

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  metrics:
    tags:
      service: ${spring.application.name}
      env: ${APP_ENV:local}
    distribution:
      percentiles:
        http.server.requests: 0.5,0.9,0.95,0.99
      percentiles-histogram:
        http.server.requests: true
```

**Код (Java): ручной таймер для бизнес-операции**

```java
package demo.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

@Service
public class PriceCalc {
  private final Timer timer;
  public PriceCalc(MeterRegistry reg) {
    this.timer = Timer.builder("biz.price.calc").description("price calculation time").register(reg);
  }
  public long calc(long base) { return timer.record(() -> base * 2); }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.metrics

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service

@Service
class PriceCalc(registry: MeterRegistry) {
    private val timer: Timer = Timer.builder("biz.price.calc").description("price calculation time").register(registry)
    fun calc(base: Long): Long = timer.record<Long> { base * 2 }
}
```

---

## Технические: пул потоков, пул соединений к БД/HTTP-клиенту, очередь задач, ошибки

Помимо HTTP-метрик, полезно наблюдать **технические** показатели: загрузку пулов потоков, размер очередей, состояние пулов соединений к БД/HTTP, количество ошибок. Многие из них уже интегрированы через Micrometer binders (JVM, Tomcat, HikariCP).

Для HikariCP метрики включаются автоматически (если Actuator на месте). Вы увидите `hikaricp.connections.*`: активные, ожидающие, максимальные. Это помогает ловить «утечки» или слишком маленький пул.

Для ваших `ExecutorService` стоит зарегистрировать **гейджи**: текущий размер пула, активные потоки, длина очереди. Тогда вы замечаете «захлёбывание» потоков до того, как пользователи почувствуют задержки.

Ошибки — это отдельные счётчики: увеличивайте `Counter` на каждой «бизнес-ошибке» или техническом сбое. Это позволяет сопоставить с WARN/ERROR в логах и строить алёрты без парсинга логов.

Для HTTP-клиента создавайте таймеры/счётчики по именованным endpoint’ам. Унифицированный тег `client`/`endpoint` облегчит агрегацию.

Документируйте набор метрик: список имён, теги, смысл. Так операторы быстрее понимают, какую метрику смотреть при инциденте.

**Код (Java): гейджи для Executor и счётчик ошибок**

```java
package demo.tech;

import io.micrometer.core.instrument.*;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;

import java.util.concurrent.*;

@Configuration
public class ExecMetricsConfig {
  @Bean ExecutorService appExecutor(MeterRegistry reg) {
    ThreadPoolExecutor ex = (ThreadPoolExecutor) Executors.newFixedThreadPool(16);
    Gauge.builder("executor.pool.size", ex, ThreadPoolExecutor::getPoolSize).register(reg);
    Gauge.builder("executor.active", ex, ThreadPoolExecutor::getActiveCount).register(reg);
    Gauge.builder("executor.queue.size", ex, e -> e.getQueue().size()).register(reg);
    return ex;
  }

  @Bean Counter businessErrors(MeterRegistry reg) {
    return Counter.builder("biz.errors").description("business errors").register(reg);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.tech

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.Gauge
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.Executors
import java.util.concurrent.ThreadPoolExecutor

@Configuration
class ExecMetricsConfig {
    @Bean
    fun appExecutor(reg: MeterRegistry): java.util.concurrent.ExecutorService {
        val ex = Executors.newFixedThreadPool(16) as ThreadPoolExecutor
        Gauge.builder("executor.pool.size", ex, ThreadPoolExecutor::getPoolSize).register(reg)
        Gauge.builder("executor.active", ex, ThreadPoolExecutor::getActiveCount).register(reg)
        Gauge.builder("executor.queue.size", ex) { it.queue.size }.register(reg)
        return ex
    }

    @Bean
    fun businessErrors(reg: MeterRegistry): Counter =
        Counter.builder("biz.errors").description("business errors").register(reg)
}
```

---

## Экспорт: включить `/actuator/metrics` и Prometheus-эндпоинт; предварительно — локальный дашборд

Micrometer — абстракция над реестрами. Для Prometheus добавьте зависимость `micrometer-registry-prometheus` и откройте `/actuator/prometheus`. Готово: Prometheus сможет скрапить метрики, а Grafana — рисовать графики.

Для OTLP (OpenTelemetry Collector) добавьте экспортер OTLP и укажите эндпоинт коллектора. Это удобно, если в компании принят «единый коллектор» и вы хотите отправлять метрики и трейс напрямую туда.

На локальной машине можно обойтись без Prometheus: просто сходить по `/actuator/metrics` и посмотреть значения по имени. Но для **перцентили** нужeн реестр с гистограммами — то есть Prometheus.

Настройте **теги** по умолчанию: `service`, `env`, `version`, `instance`. Это делает метрики сравнимыми между инстансами и окружениями. В Boot 3 их удобно задать в `management.metrics.tags`.

Не забывайте о безопасности: эндпоинт `/actuator/prometheus` должен быть доступен только из сети мониторинга. В публичный интернет его не выпускаем.

Добавьте «минимальный» дашборд: RPS, p95, 5xx, ошибки бизнес-логики (counter), состояние пулов (Hikari, executor). Это покрывает 80% инцидентов.

**Gradle (Groovy/Kotlin) — зависимость Prometheus:**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  runtimeOnly 'io.micrometer:micrometer-registry-prometheus'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  runtimeOnly("io.micrometer:micrometer-registry-prometheus")
}
```

**`application.yml` — теги и экспозиция:**

```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  metrics:
    tags:
      service: ${spring.application.name}
      env: ${APP_ENV:local}
      version: ${APP_VERSION:dev}
```

**Код (Java/Kotlin): проверочный запрос к реестру**

```java
package demo.export;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class DumpMetrics implements CommandLineRunner {
  private final MeterRegistry reg;
  public DumpMetrics(MeterRegistry reg){ this.reg = reg; }
  @Override public void run(String... args){ reg.getMeters().forEach(m -> System.out.println(m.getId())); }
}
```

```kotlin
package demo.export

import io.micrometer.core.instrument.MeterRegistry
import org.springframework.boot.CommandLineRunner
import org.springframework.stereotype.Component

@Component
class DumpMetrics(private val reg: MeterRegistry) : CommandLineRunner {
    override fun run(vararg args: String?) { reg.meters.forEach { println(it.id) } }
}
```

---

## Перекрёстная проверка: метрика ошибок должна совпадать с частотой ERROR/WARN в логах

Логи и метрики — два взгляда на одно и то же. Чтобы не «обманывать» себя, делайте **перекрёстную проверку**: сравнивайте график `biz.errors`/`http.server.requests{status=5xx}` с количеством ERROR/WARN в логах за тот же период. Они не обязаны совпадать до единицы, но тренды должны быть одинаковыми.

Если в логах всплеск WARN, а метрика «ровная», значит, вы не инкрементируете счётчик в нужных местах или у вас «размазан» уровень WARN. Аналогично, если `5xx` высок, а ERROR почти нет — вы логируете ошибки как INFO. Это повод исправить политику уровней.

Такой «санити-чек» полезно автоматизировать в алёртинге: триггер, когда наблюдается отклонение трендов. Это защитит от «тихих поломок» логирования (фильтр отрезал часть сообщений, агент завис, уровень «слетел»).

Интеграция простая: при обработке исключения **инкрементируйте** `Counter` и логируйте ERROR. Для бизнес-ошибок (валидация) — инкрементируйте отдельный `Counter` и пишите INFO/WARN.

Не перегружайте систему: не превращайте каждую WARN в отдельную метрику. Лучше иметь один/несколько счётчиков по категориям ошибок с тегами (`type=timeout|validation|external`). Это гибче и дешевле.

В RCA прикладывайте обе картинки: «метрики ошибок» и «частота ERROR в логах». Это дисциплинирует команду и делает выводы очевидными.

**Код (Java): счётчик + ERROR-лог в одном месте**

```java
package demo.cross;

import io.micrometer.core.instrument.Counter;
import org.slf4j.*; import org.springframework.stereotype.Service;

@Service
public class ErrorRecorder {
  private static final Logger log = LoggerFactory.getLogger(ErrorRecorder.class);
  private final Counter serverErrors;
  public ErrorRecorder(io.micrometer.core.instrument.MeterRegistry reg) {
    this.serverErrors = Counter.builder("server.errors").tag("type","exception").register(reg);
  }
  public void record(Exception ex, String ctx) {
    serverErrors.increment();
    log.error("server.error context={}", ctx, ex);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.cross

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class ErrorRecorder(registry: MeterRegistry) {
    private val log = LoggerFactory.getLogger(javaClass)
    private val serverErrors: Counter = Counter.builder("server.errors").tag("type","exception").register(registry)
    fun record(ex: Exception, ctx: String) {
        serverErrors.increment()
        log.error("server.error context={}", ctx, ex)
    }
}
```

# 8. Health/Info и технические проверки

## `/actuator/health` (liveness/readiness) и `/actuator/info` (версия, git sha) — включить всегда

Для эксплуатации сервисов важны два базовых эндпоинта: `/actuator/health` и `/actuator/info`. Первый отвечает за «жизненность» (liveness) и «готовность» (readiness) приложения, второй — сообщает версию и служебные сведения (git commit, дата сборки). Эти точки нужны не только людям: оркестраторы (kubelet/ingress) и балансировщики используют их для снятия и подачи трафика, а CD-пайплайны — для проверки успешного старта релиза. Поэтому они должны быть включены на всех средах, даже в dev.

В Spring Boot 3 включение максимально простое: `spring-boot-starter-actuator` плюс настройка экспозиции эндпоинтов. Отдельно обратите внимание на опцию `management.endpoint.health.probes.enabled=true` — она автоматически добавляет «зонтичные» компоненты `liveness` и `readiness`, что удобно для интеграции с Kubernetes. На практике вы будете запрашивать `/actuator/health/liveness` для проверки «процесс жив», и `/actuator/health/readiness` — «готов принимать трафик».

`/actuator/info` часто недооценивают, но в инциденте он экономит минуты: открываете страницу — видите точную версию, git sha, время сборки, а иногда и build number. Стайлгайд: не кладите в `info` секреты и переменные окружения без отбора — только безопасные метки релиза. Хорошо, если ваши CI-скрипты автоматически подставляют значения.

С точки зрения безопасности, решите: будут ли эти эндпоинты доступны без аутентификации. В большинстве компаний `/health` для read-only GET остаётся открытым внутри кластера/сети мониторинга, но не наружу; `/info` — по той же логике. Если вы публикуете сервис в интернет, спрячьте эти точки за ингресс-правилами или ограничьте их видимостью.

Помните, что `health` — не «проверка всего на свете». Liveness должен быть максимально быстрым и устойчивым (не ходить во внешние системы). Readiness может быть чуть тяжелее: например, убедиться в наличии соединений с базой, очередью, кэшем. При этом важно ставить таймауты и не блокировать поток надолго.

Наконец, договоритесь о том, что CI/CD использует именно readiness для принятия решения о «готов к трафику», а не просто «контейнер запустился». Это снижает вероятность «зелёного деплоя», который сразу начнёт отдавать 500-ки из-за неподключившейся БД.

**`application.yml` — включаем health/info и пробы**

```yaml
spring:
  application:
    name: payments
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      probes:
        enabled: true
info:
  app:
    version: ${APP_VERSION:dev}
    commit: ${GIT_COMMIT:unknown}
```

**Код (Java): простой InfoContributor**

```java
package demo.healthinfo;

import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

@Component
public class BuildInfoContributor implements InfoContributor {
  @Override public void contribute(Info.Builder builder) {
    builder.withDetail("build", java.util.Map.of(
        "java", System.getProperty("java.version"),
        "startedBy", System.getProperty("user.name")
    ));
  }
}
```

**Код (Kotlin): аналог InfoContributor**

```kotlin
package demo.healthinfo

import org.springframework.boot.actuate.info.Info
import org.springframework.boot.actuate.info.InfoContributor
import org.springframework.stereotype.Component

@Component
class BuildInfoContributor : InfoContributor {
    override fun contribute(builder: Info.Builder) {
        builder.withDetail(
            "build", mapOf(
                "java" to System.getProperty("java.version"),
                "startedBy" to System.getProperty("user.name")
            )
        )
    }
}
```

Gradle (Groovy/Kotlin) — зависимости:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-actuator' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-actuator") }
```

---

## Примитивный self-check: проверка ключевых зависимостей (БД/кеш/очередь) с таймаутами

Readiness-проверка должна отражать способность сервиса **обслуживать запросы**. Это не «пинг всего мира», а минимальный набор: доступна ли БД (или пул соединений), подключён ли брокер сообщений, доступен ли кэш. В Spring Boot эти проверки можно доверить готовым `HealthIndicator` из стартеров (Hikari, Redis, Kafka), либо написать свои для критичных интеграций.

Главное правило self-check’ов — **быстрота и таймауты**. Никогда не делайте сетевой вызов без ограничений: readiness должен отвечать стабильно и быстро, иначе оркестратор начнёт дёргать контейнер. Для JDBC достаточно «SELECT 1» через пул, а не реальный бизнес-запрос. Для очередей — проверка метаданных соединения, а не чтение/запись сообщения.

Если зависимость временно недоступна, вы хотите видеть `OUT_OF_SERVICE` на readiness, чтобы трафик сняли с инстанса. При этом liveness остаётся `UP`, иначе платформы начнут рестартовать контейнеры, усугубляя проблему. Так вы достигаете «дренирования» без перезапуска.

Расширяйте `HealthIndicator` аккуратно: повторные вызовы должны кэшироваться на короткое время (например, 1–5 секунд), чтобы не устроить DoS собственной БД. В проде readiness вызывается часто балансировщиком и kube-prober’ом.

Пишите self-check для «своих» клиентов, если автоконфигов нет: например, вы оборачиваете внешний REST-API — можно дернуть лёгкий эндпоинт с таймаутом 100–200 мс. Если он недоступен, это не обязательно `DOWN`, но полезно отразить в детале `degraded=true`, чтобы операторы видели деградацию.

Логику «считаться готовым при частичной деградации» реализуйте через групповые health-группы: можно объявить `/actuator/health/readiness` как `anyMatch(UP, WARN)` и поместить «не критичные» интеграции отдельно.

**Код (Java): кастомный HealthIndicator с таймаутом**

```java
package demo.healthinfo;

import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Component;

import java.net.HttpURLConnection;
import java.net.URL;
import java.time.Duration;

@Component
public class PartnerApiHealth implements HealthIndicator {
  @Override public Health health() {
    try {
      URL url = new URL("https://status.partner.example/ping");
      HttpURLConnection c = (HttpURLConnection) url.openConnection();
      c.setConnectTimeout((int) Duration.ofMillis(200).toMillis());
      c.setReadTimeout((int) Duration.ofMillis(200).toMillis());
      c.connect();
      int code = c.getResponseCode();
      return (code == 200) ? Health.up().withDetail("partner", "ok").build()
          : Health.status("DEGRADED").withDetail("code", code).build();
    } catch (Exception e) {
      return Health.down(e).withDetail("partner", "unreachable").build();
    }
  }
}
```

**Код (Kotlin): аналог HealthIndicator**

```kotlin
package demo.healthinfo

import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.stereotype.Component
import java.net.HttpURLConnection
import java.net.URL
import java.time.Duration

@Component
class PartnerApiHealth : HealthIndicator {
    override fun health(): Health = try {
        val c = (URL("https://status.partner.example/ping").openConnection() as HttpURLConnection).apply {
            connectTimeout = Duration.ofMillis(200).toMillis().toInt()
            readTimeout = Duration.ofMillis(200).toMillis().toInt()
            connect()
        }
        if (c.responseCode == 200) Health.up().withDetail("partner", "ok").build()
        else Health.status("DEGRADED").withDetail("code", c.responseCode).build()
    } catch (e: Exception) {
        Health.down(e).withDetail("partner", "unreachable").build()
    }
}
```

**Группы и кэширование health (YAML)**

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: readinessState, db, redis, kafka, partnerApiHealth
      cache:
        time-to-live: 2s
```

---

## Стандартные теги: service.name, env, region — единообразно для дашбордов и логов

Смысл наблюдаемости — сравнимость данных между сервисами и инстансами. Для этого вводят **стандартные теги**: `service` (название приложения), `env` (окружение: dev/stage/prod), `region`/`zone` (география/зона доступности), `version` (git sha/семвер). Эти метки должны одинаково присутствовать в логах, метриках и health-деталях.

В Spring Boot метрики тегируются через `management.metrics.tags.*`. Для логов такие же значения добавляются в шаблон JSON-энкодера (Logback) и/или прокидываются в MDC. Для info/health — в `InfoContributor` и детали health. Главное — держать **одно и то же имя** тегов везде: это упрощает построение универсальных дашбордов.

Источник правды для тегов — окружение. Не хардкодьте «prod» в коде; передавайте `APP_ENV`, `APP_REGION`, `APP_VERSION` при деплое. Тогда перенос сервиса между кластерами не потребует релиза, а метки всегда актуальны.

Если используете несколько компонент в одном сервисе (http, воркер, консюмер), добавьте тег `component=api|worker|consumer`. Это позволяет видеть отдельные профили нагрузок и не мешать их на одном графике.

Не перегружайте набор тегов: 4–6 инвариантных меток достаточно. Слишком большое количество тегов в метриках может взорвать количество временных рядов (high cardinality) и поднимать стоимость мониторинга.

Согласуйте регистр и формат: `service` — в kebab-case (`billing-api`), `env` — маленькими (`prod`), `region` — как в вашей платформе (`eu-central-1`). Переименование задним числом ломает запросы и дашборды.

**`application.yml` — единые теги для метрик**

```yaml
management:
  metrics:
    tags:
      service: ${spring.application.name}
      env: ${APP_ENV:local}
      region: ${APP_REGION:dev-1}
      version: ${APP_VERSION:dev}
```

**Код (Java): добавляем те же теги в MDC на старте**

```java
package demo.tags;

import org.slf4j.MDC;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class TagBootstrap {
  @EventListener(ApplicationReadyEvent.class)
  public void putTags() {
    MDC.put("service", System.getProperty("spring.application.name", "app"));
    MDC.put("env", System.getenv().getOrDefault("APP_ENV", "local"));
    MDC.put("region", System.getenv().getOrDefault("APP_REGION", "dev-1"));
    MDC.put("version", System.getenv().getOrDefault("APP_VERSION", "dev"));
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.tags

import org.slf4j.MDC
import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class TagBootstrap {
    @EventListener(ApplicationReadyEvent::class)
    fun putTags() {
        MDC.put("service", System.getProperty("spring.application.name", "app"))
        MDC.put("env", System.getenv().getOrDefault("APP_ENV", "local"))
        MDC.put("region", System.getenv().getOrDefault("APP_REGION", "dev-1"))
        MDC.put("version", System.getenv().getOrDefault("APP_VERSION", "dev"))
    }
}
```

---

# 9. Логи в контейнерах и прод-окружении

## Писать в stdout/stderr, ротацию делегировать платформе (Docker/лог-агенту)

В контейнерах лучший канал логирования — **stdout/stderr**. Docker и оркестраторы читают их нативно, а агенты (fluent-bit/filebeat) подбирают и отправляют дальше. Это избавляет приложение от забот о файловой системе, ротации и прав доступа. Любые локальные файлы — лишь аварийная подстраховка на стендах.

Ротация — работа платформы. Docker умеет ограничивать размер/историю логов драйвера; в Kubernetes ротацию делает node-level logrotate. Приложению не нужно «саморотироваться». Если вы всё же включаете RollingFile на стенде, держите маленькую историю и общий объём, чтобы не забить диск.

Асинхронный аппендер в приложении полезен и в контейнере: запись в stdout — это всё равно вывод в pipe; небольшая очередь снижает задержки в горячих путях. Но помните про потерю событий при переполнении — включайте метрику отбрасываний и алёрт.

Для форматирования в проде предпочтителен **JSON** — агенты легко парсят его без хрупких grok-паттернов. Следите, чтобы в JSON попадали инвариантные теги (service/env/version/region/traceId). На dev можно оставить человекочитаемый паттерн.

Убедитесь, что контейнер запускается с правильной локалью/таймзоной или, лучше, логируйте в UTC. Это упростит корреляцию событий между сервисами и кластерами.

Если у вас несколько процессов в контейнере (редко, но бывает), следите, чтобы они тоже писали в stdout/stderr. Логи sidecar-агентов можно отделять по имени контейнера/стрима в хранилище.

**Dockerfile — ничего лишнего, пишем в stdout**

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY build/libs/app.jar app.jar
# никакой лог-директории, всё в stdout/stderr
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

**Код (Java): убедимся, что логи идут в stdout**

```java
package demo.container;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class StdoutPing {
  private static final Logger log = LoggerFactory.getLogger(StdoutPing.class);
  @EventListener(ApplicationReadyEvent.class)
  public void started() { log.info("stdout.alive true"); }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.container

import org.slf4j.LoggerFactory
import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class StdoutPing {
    private val log = LoggerFactory.getLogger(javaClass)
    @EventListener(ApplicationReadyEvent::class)
    fun started() { log.info("stdout.alive true") }
}
```

---

## Передача логов в централизованное хранилище (агенты типа filebeat/fluent-bit) с парсингом JSON

В проде логи должны стекаться в централизованную систему (ELK, Loki, Cloud Logging). На узлах кластера работают агенты (fluent-bit/filebeat), которые читают stdout контейнеров, парсят и отправляют дальше. Ваша часть — **стабильный JSON** и метки для маршрутизации (service/env/region).

Парсинг JSON в агентах почти всегда «из коробки»: достаточно указать, что вход — JSON, и выбрать поля времени/уровня. Так вы избегаете хрупких регулярных выражений и проблем при изменении паттерна. Если JSON сложный, согласуйте схему с платформой заранее.

Транспорт должен быть надёжным: агенты буферизуют локально, шлют по TLS, используют backpressure при перегрузке. Приложение при этом ничего не знает о хранилище и не тормозит при проблемах сети. Это правильная изоляция обязанностей.

В хранилище логи попадают в индексы/стримы по ключам. Договоритесь о слайсинге: «один индекс на сервис в сутки» или «общий индекс по окружению с тегом service». Выбор влияет на стоимость и скорость поиска.

Не забудьте про **retention** и ILM (lifecycle): «горячие» сутки — быстрое хранилище, «тёплые» 7–30 дней — подешевле, дальше — архив. Это снижает счёт от мониторинга и избегает «бесконечного» роста.

Для отладки интеграции добавьте в сервис endpoint «/observability/ping-log», который пишет тестовую JSON-строку. Смотрим, дошла ли она в нужный индекс и с нужными тегами.

**Код (Java): маркер «я в проде, JSON читается»**

```java
package demo.ship;

import org.slf4j.Logger; import org.slf4j.LoggerFactory;
import org.slf4j.MDC; import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/observability")
public class ShipLogController {
  private static final Logger log = LoggerFactory.getLogger("ship");
  @GetMapping("/ping-log")
  public String ping() {
    MDC.put("probe", "true");
    log.info("log.probe ok");
    MDC.clear();
    return "ok";
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.ship

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController @RequestMapping("/observability")
class ShipLogController {
    private val log = LoggerFactory.getLogger("ship")
    @GetMapping("/ping-log")
    fun ping(): String {
        MDC.put("probe", "true")
        log.info("log.probe ok")
        MDC.clear()
        return "ok"
    }
}
```

---

## Единые теги для поиска: приложение, версия, окружение, компонент

При централизованном поиске важно быстро сузить выборку: «покажи ERROR для `payments` `prod` версии `1.4.2` компонента `worker` за последние 15 минут». Это достигается **едиными тегами**, приходящими в лог вместе с каждым сообщением. Мы уже наметили: `service`, `env`, `version`, `region`, плюс `component`.

Полезно дублировать эти теги и в метриках. Тогда можно делать «джойн в голове»: граф `5xx` и тут же — фильтр логов по той же комбинации тегов. Операторы не должны вспоминать «как называется поле в логах и как — в метриках»; имена должны совпадать.

Ещё один тег — `instance` (например, `pod_name`). Его обычно добавляет агент, но неплохо проставить и в самом приложении как `HOSTNAME`. Это помогает видеть «плохой под», который отличается по уровню ошибок.

Для потоков обработки (Kafka, scheduled) стоит добавить `component=consumer|scheduler` и, возможно, `stream` (имя топика). Тогда поиск «ошибок консюмера X» — секундное дело.

Наконец, договоритесь о **жёстком наборе**: какие теги обязательны, какие опциональны. Любая команда, подключающаяся к централизованным логам, должна следовать тем же правилам — иначе получится «вавилон».

**`logback-spring.xml` — JSON с едиными тегами**

```xml
<configuration>
  <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
      <providers>
        <timestamp/>
        <pattern>
          <pattern>{"service":"${spring.application.name}","env":"${APP_ENV:-local}","version":"${APP_VERSION:-dev}","region":"${APP_REGION:-dev-1}","component":"${APP_COMPONENT:-api}","message":"%message","logger":"%logger","level":"%level"}</pattern>
        </pattern>
        <mdc/><stackTrace/>
      </providers>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="JSON"/></root>
</configuration>
```

**Код (Java/Kotlin): лог с явным component**

```java
org.slf4j.MDC.put("component","worker");
org.slf4j.LoggerFactory.getLogger("biz").info("worker.started");
org.slf4j.MDC.remove("component");
```

```kotlin
org.slf4j.MDC.put("component", "worker")
org.slf4j.LoggerFactory.getLogger("biz").info("worker.started")
org.slf4j.MDC.remove("component")
```

---

## Базовые дашборды: RPS/коды/латентность + топ-ошибки и их шаблоны сообщений

Даже без сложной телеметрии полезно иметь «стартовый» дашборд: графики RPS, распределение кодов (2xx/4xx/5xx), p50/p95/пики, и панель «топ-ошибок» из логов (частые шаблоны сообщений за период). Это покрывает 80% инцидентов: вы сразу видите «когда началось», «сколько влияет» и «какая ошибка доминирует».

Где брать: RPS/коды/латентность приходят из Micrometer HTTP-метрик. «Топ-ошибки» — агрегация по полю `message`/`error.type` в логах (важна стабильность текста сообщения!). В идеале вы добавляете отдельное поле `errorCode` в лог ошибок и блюдете его стабильность — так дашборд вообще не зависит от формулировки сообщения.

Полезно добавить панель «внешние зависимости»: p95 по внешним HTTP-клиентам, число таймаутов, ошибки Kafka/БД. Эти графики напрямую указывают на «узкое место» и позволяют быстро решить, кто on-call: ваша команда или платформа/партнёр.

Ещё одна панель — «баланс по инстансам»: сравнение 5xx/latency между подами. Если один под выбивается — проблема локальная (например, горячий GC). Если все — общая деградация (внешняя зависимость).

Желательно иметь кнопку (или ссылку) «перейти к логам с теми же фильтрами». В Grafana/ELK это делается ссылкой, в которую подставляются те же теги и время. Это сокращает цикл «увидел на графике → открыл конкретные логи».

И не забывайте про учётную запись для разработчиков и on-call: у всех, кто отвечает за инциденты, должен быть доступ к этим дашбордам. Нет доступа — нет наблюдаемости.

**Код (Java): счётчик бизнес-ошибок с `errorCode`**

```java
package demo.dash;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class ErrorStats {
  private final org.slf4j.Logger log = LoggerFactory.getLogger(ErrorStats.class);
  private final MeterRegistry reg;
  public ErrorStats(MeterRegistry reg) { this.reg = reg; }

  public void recordValidationError(String code) {
    Counter.builder("biz.error").tag("code", code).register(reg).increment();
    log.warn("validation.failed code={}", code);
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.dash

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class ErrorStats(private val reg: MeterRegistry) {
    private val log = LoggerFactory.getLogger(javaClass)
    fun recordValidationError(code: String) {
        Counter.builder("biz.error").tag("code", code).register(reg).increment()
        log.warn("validation.failed code={}", code)
    }
}
```

---

# 10. Базовый чек-лист качества логирования

## Есть структурный формат, traceId в каждом сообщении, маскирование секретов — да/нет

Первый пункт аудита — **структурные логи**. Если логи текстовые, вы зависите от шаблонов и парсеров; если JSON — парсинг прост и стабилен. Проверяйте: есть ли поля `service`, `env`, `version`, `traceId`, и выводится ли `stack_trace` отдельным полем. Это база для поиска и алёртов.

Второй — **корреляция**. В каждом сообщении должен быть `traceId`/`spanId`, чтобы быстро переходить от метрик/трейсов к логам. Если трейсинг не подключен, генерируйте `X-Correlation-Id` на входе и записывайте его в MDC и ответ клиенту.

Третий — **маскирование секретов**. Любые токены/пароли/карты должны скрываться до записи. Нужна не только библиотека фильтрации, но и культура: запрет на логирование тел запросов/ответов без белого списка. Проверяйте это код-ревью и тестами.

Четвёртый — **одинаковость схемы**. Поля должны называться одинаково во всех сервисах. Если в одном сервисе `app`, в другом `service`, а в третьем `svc`, дашборды и запросы расползаются. Этот пункт — организационный: договоритесь и закрепите.

Пятый — **UTC-время** и ISO-формат. Смешение таймзон делает расследования болью, особенно в мульти-регионе. Если время приходит из агента — убедитесь, что он нормализует таймстемп.

Шестой — **проверка на стендах**. Имитируйте тестовый инцидент: сгенерируйте ошибку, найдите её по traceId, убедитесь, что стек виден и нет секретов. Это быстрая практика, показывающая реальные «дыры» до продакшна.

**Код (Java): smoke-тест наличия traceId в логах (WebTestClient)**

```java
package demo.checklist;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

import org.springframework.beans.factory.annotation.Autowired;

@SpringBootTest
@AutoConfigureWebTestClient
class TraceIdSmokeTest {
  @Autowired WebTestClient web;

  @Test void returnsTraceIdHeader() {
    web.get().uri("/hello?name=A").exchange()
      .expectStatus().is2xxSuccessful()
      .expectHeader().exists("X-Trace-Id");
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.checklist

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.reactive.server.WebTestClient

@SpringBootTest
@AutoConfigureWebTestClient
class TraceIdSmokeTest {
    @Autowired lateinit var web: WebTestClient

    @Test
    fun returnsTraceIdHeader() {
        web.get().uri("/hello?name=A").exchange()
            .expectStatus().is2xxSuccessful
            .expectHeader().exists("X-Trace-Id")
    }
}
```

Gradle (Groovy/Kotlin) — тест:

```groovy
dependencies { testImplementation 'org.springframework.boot:spring-boot-starter-test' }
```

```kotlin
dependencies { testImplementation("org.springframework.boot:spring-boot-starter-test") }
```

---

## Уровни настроены: DEBUG выключен в проде, ERROR только по реальным сбоям — да/нет

Политика уровней — второй столп. Проверяем: `root` в проде — минимум `INFO`, а DEBUG включается **точечно** и **временно** через `/actuator/loggers`. Ошибки клиента (4xx) не порождают ERROR — иначе алёрты будут ложными. Временные сбои (ретраи, timeouts с fallback) — WARN.

Хорошо иметь «реестр правил» по уровням: таблицу типовых ситуаций и целевой уровень для них. Это выравнивает команду и облегчает код-ревью. Добавьте в CI линтер, который ищет `log.error` рядом с 4xx — мелочь, но дисциплинирует.

Тестируйте поведение уровней. Простейшая проверка — через Actuator: поднимите уровень логгера на DEBUG, выполните запрос, верните обратно. Так убедитесь, что «в проде можно адресно включить детали» без релиза.

ERROR должен сопровождаться stacktrace и бизнес-контекстом (id операции). WARN — кратким резюме и ключевыми полями для поиска. INFO — «жизненный цикл» и бизнес-события.

Не давайте developer-friendly DEBUG «просачиваться» на горячие пути в проде. Пару таких строк в цикле — и вы получите гигабайты логов и нагрузку на CPU/IO.

И не забывайте про «шумные библиотеки»: задайте уровни для `org.apache.http`, `org.hibernate.SQL` и т.п. Отдельно на dev можно включать их DEBUG, но прод — только INFO/WARN.

**`application.yml` — дефолтные уровни и доступ к /loggers**

```yaml
logging:
  level:
    root: INFO
    org.hibernate.SQL: WARN
management:
  endpoints.web.exposure.include: loggers
```

**Код (Java): программная смена уровня (пример)**

```java
package demo.checklist;

import org.springframework.boot.logging.LoggingSystem;
import org.springframework.boot.logging.LogLevel;
import org.springframework.stereotype.Service;

@Service
public class DebugSwitch {
  private final LoggingSystem sys;
  public DebugSwitch(LoggingSystem sys){ this.sys = sys; }
  public void enableDebug(String logger) { sys.setLogLevel(logger, LogLevel.DEBUG); }
  public void disable(String logger) { sys.setLogLevel(logger, null); } // back to parent
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.checklist

import org.springframework.boot.logging.LogLevel
import org.springframework.boot.logging.LoggingSystem
import org.springframework.stereotype.Service

@Service
class DebugSwitch(private val sys: LoggingSystem) {
    fun enableDebug(logger: String) = sys.setLogLevel(logger, LogLevel.DEBUG)
    fun disable(logger: String) = sys.setLogLevel(logger, null)
}
```

---

## Access-логи отделены, health/metrics включены, метрики согласованы с логами — да/нет

Третий пункт чек-листа — **структура контура**. Access-логи отделены от бизнес-логов (свои логгеры/индексы), `/actuator/health` и `/actuator/prometheus` включены, и метрики по ошибкам/латентности коррелируют с логами. Это не «красиво», это уменьшает MTTR.

Access-логи в отдельном канале позволяют быстро оценить уровни кодов, RPS и «hot paths». Бизнес-логи остаются читаемыми и менее «болтливыми». Если у вас есть ingress-access-логи, вы можете отключить собственные — но убедитесь, что сохраняется traceId.

Метрики должны совпадать по трендам с логами: всплеск 5xx → всплеск ERROR/WARN. Если это не так — ищите разъезжающиеся политики уровней, потерю логов агентом, неправильную категоризацию метрик.

Health с readiness обязателен для деплоя без простоя. Если он отсутствует — оркестратор будет слепо слать трафик в «полузапущенный» под, а вы увидите «поезд-призрак» 500-ок на старте.

Согласованность имён и тегов между логами и метриками — критична для «сквозных» дашбордов. Повторим: одинаковые `service/env/version/region`.

Пропишите в документации: где смотреть access-логи, где бизнес-логи, где метрики и какие панели в Grafana «главные». Новый инженер должен за 5 минут найти нужную информацию.

**Код (Java): WebFilter для access-логов (короткий JSON)**

```java
package demo.accesscheck;

import org.slf4j.LoggerFactory; import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.server.*;
import reactor.core.publisher.Mono;

@Component
public class AccessFilter implements WebFilter {
  private static final org.slf4j.Logger access = LoggerFactory.getLogger("access");
  @Override public Mono<Void> filter(ServerWebExchange ex, WebFilterChain chain) {
    long t0 = System.nanoTime();
    return chain.filter(ex).doFinally(sig -> {
      long took = (System.nanoTime()-t0)/1_000_000;
      access.info("method={} path={} tookMs={} traceId={}",
        ex.getRequest().getMethod(), ex.getRequest().getPath().value(), took, MDC.get("traceId"));
    });
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.accesscheck

import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
class AccessFilter : WebFilter {
    private val access = LoggerFactory.getLogger("access")
    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        val t0 = System.nanoTime()
        return chain.filter(exchange).doFinally {
            val took = (System.nanoTime() - t0) / 1_000_000
            access.info(
                "method={} path={} tookMs={} traceId={}",
                exchange.request.method, exchange.request.path.value(), took, MDC.get("traceId")
            )
        }
    }
}
```

**`application.yml` — включаем метрики/health**

```yaml
management:
  endpoints.web.exposure.include: health,metrics,prometheus
  endpoint.health.probes.enabled: true
```

---

## Документация: что логируем, что нельзя логировать, где смотреть, как включить DEBUG на узком участке — да/нет

Последний пункт — **процедуры**. Технически можно сделать всё правильно и «провалиться» из-за отсутствия документации: где искать логи и дашборды, какие есть поля и их семантика, как временно включить DEBUG для пакета «X» и через сколько выключить.

Вики-страница сервиса должна описывать: схему логов (обязательные поля), набор метрик с именами и тегами, список health-проверок и их смысл, а также ссылки на Grafana/Kibana панели. К этому добавьте «runbook инцидента»: шаги проверки, типовые запросы в логах, как эскалировать.

Запреты и маскирование: перечислите поля, которые запрещено писать (PII/секреты), и механизмы маскирования. Пропишите ответственность и процесс ревизии — это важно для безопасности и аудита.

Процедура включения DEBUG должна быть безопасной: кто имеет право, как именно включать (`/actuator/loggers` или через SRE-панель), на какой срок, как зафиксировать факт. Добавьте напоминание о «возврате уровней».

Для новых разработчиков полезно иметь «песочницу» — sample-приложение, которое показывает все практики: JSON-лог, MDC, метрики, health, трейсинг. Это ускоряет onboarding.

И не забудьте про автоматическую проверку: линтер/тест, который «пингует» `/health` и ищет в логах нужные поля — дешёвая страховка, что базовая гигиена не сломалась.

**Код (Java): эндпоинт «observability help»**

```java
package demo.docs;

import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/observability")
public class HelpController {
  @GetMapping("/help")
  public Map<String, Object> help() {
    return Map.of(
      "logs", Map.of("format","json","fields","service,env,version,traceId"),
      "endpoints", Map.of("health","/actuator/health","metrics","/actuator/prometheus","loggers","/actuator/loggers"),
      "howToEnableDebug","POST /actuator/loggers/{loggerName} {\"configuredLevel\":\"DEBUG\"}"
    );
  }
}
```

**Код (Kotlin): аналог**

```kotlin
package demo.docs

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/observability")
class HelpController {
    @GetMapping("/help")
    fun help(): Map<String, Any> = mapOf(
        "logs" to mapOf("format" to "json", "fields" to "service,env,version,traceId"),
        "endpoints" to mapOf("health" to "/actuator/health", "metrics" to "/actuator/prometheus", "loggers" to "/actuator/loggers"),
        "howToEnableDebug" to "POST /actuator/loggers/{loggerName} {\"configuredLevel\":\"DEBUG\"}"
    )
}
```

**Gradle (Groovy/Kotlin) — Actuator и web для примеров**

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```





