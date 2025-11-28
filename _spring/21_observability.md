---
layout: page
title: "Наблюдаемость (расширено): метрики и трейсинг"
permalink: /spring/observability
---

<!-- TOC -->
* [0. Введение](#0-введение)
  * [Что это](#что-это)
  * [Зачем это](#зачем-это)
  * [Где используется](#где-используется)
  * [Какие проблемы решает](#какие-проблемы-решает)
* [1. Цели и таксономия наблюдаемости](#1-цели-и-таксономия-наблюдаемости)
  * [SLI/SLO/SLA](#slislosla)
  * [RED/Golden Signals и USE](#redgolden-signals-и-use)
  * [Стоимость и риск](#стоимость-и-риск)
* [2. Дизайн метрик: типы, единицы, кардинальность, гистограммы](#2-дизайн-метрик-типы-единицы-кардинальность-гистограммы)
  * [Типы: Counter/Gauge/Timer/DistributionSummary/Histogram](#типы-countergaugetimerdistributionsummaryhistogram)
  * [Единицы и имена](#единицы-и-имена)
  * [Кардинальность: контроль label’ов](#кардинальность-контроль-labelов)
  * [Временные распределения: buckets/percentiles](#временные-распределения-bucketspercentiles)
  * [Exemplars: метрика→трейс](#exemplars-метрикатрейс)
* [3. Инструментирование в Spring Boot (Micrometer/Actuator)](#3-инструментирование-в-spring-boot-micrometeractuator)
  * [Встроенные метрики: HTTP/JVM/DB/Exec/Kafka/Redis](#встроенные-метрики-httpjvmdbexeckafkaredis)
  * [Observation API: единый слой + мост в OTel](#observation-api-единый-слой--мост-в-otel)
  * [Кастомные метрики: Timer/LongTaskTimer/DistributionSummary](#кастомные-метрики-timerlongtasktimerdistributionsummary)
  * [Конфигурация дистрибуций: percentiles/histogram/SLA/exemplars](#конфигурация-дистрибуций-percentileshistogramslaexemplars)
  * [Экспорт: Prometheus scrape vs OTLP Metrics](#экспорт-prometheus-scrape-vs-otlp-metrics)
* [4. Прометеевский стек и правила эксплуатации](#4-прометеевский-стек-и-правила-эксплуатации)
  * [Scrape-модель: интервалы, timeouts, relabeling, Service/PodMonitor (Operator)](#scrape-модель-интервалы-timeouts-relabeling-servicepodmonitor-operator)
  * [Правила: recording rules и MWMBR-алёрты по SLO](#правила-recording-rules-и-mwmbr-алёрты-по-slo)
  * [Масштабирование хранения: Thanos/Mimir/Cortex, remote_write, ретеншн и downsampling](#масштабирование-хранения-thanosmimircortex-remote_write-ретеншн-и-downsampling)
  * [Кардинальность под контролем: top-N, explorer, лимиты/квоты](#кардинальность-под-контролем-top-n-explorer-лимитыквоты)
* [5. Grafana: дашборды, алёрты, практики](#5-grafana-дашборды-алёрты-практики)
  * [Дашборды «по роли»: SRE/инциденты, разработчик сервиса, бизнес-метрики](#дашборды-по-роли-sreинциденты-разработчик-сервиса-бизнес-метрики)
  * [Алёртинг: SLO-правила, тишина vs paging, dead-man switch](#алёртинг-slo-правила-тишина-vs-paging-dead-man-switch)
  * [Drill-down: метрика→трейс (exemplars), метрика→логи (traceId в MDC)](#drill-down-метрикатрейс-exemplars-метрикалоги-traceid-в-mdc)
  * [Отчётность: артефакты пост-мортема, автоссылки на runbooks](#отчётность-артефакты-пост-мортема-автоссылки-на-runbooks)
* [6. JVM/пулы/GC: «железная» часть метрик](#6-jvmпулыgc-железная-часть-метрик)
  * [GC (G1/ZGC): паузы, частота, reclaimed bytes; heap/metaspace, allocation rate](#gc-g1zgc-паузы-частота-reclaimed-bytes-heapmetaspace-allocation-rate)
  * [Пулы потоков/очереди: size/active/queue depth/rejected; CallerRunsPolicy](#пулы-потоковочереди-sizeactivequeue-depthrejected-callerrunspolicy)
  * [JFR/JMX как источники сигналов, интеграция с Micrometer/OTel](#jfrjmx-как-источники-сигналов-интеграция-с-micrometerotel)
  * [Нагрузочные бюджеты: читать saturations и планировать CPU/memory/IO](#нагрузочные-бюджеты-читать-saturations-и-планировать-cpumemoryio)
* [7. Трейсинг: модель, семантика и контекст](#7-трейсинг-модель-семантика-и-контекст)
  * [Спаны: серверный/клиентский, атрибуты (семантика OTel), events/status](#спаны-серверныйклиентский-атрибуты-семантика-otel-eventsstatus)
  * [Пропагация: W3C traceparent/tracestate, baggage (минимум)](#пропагация-w3c-traceparenttracestate-baggage-минимум)
  * [Ошибки и статус: где ставить error.flag, как не раздувать спаны логами](#ошибки-и-статус-где-ставить-errorflag-как-не-раздувать-спаны-логами)
  * [Инструментирование Spring: javaagent OTel и вручную через Observation/Tracer](#инструментирование-spring-javaagent-otel-и-вручную-через-observationtracer)
* [8. Пайплайн трейсинга и sampling](#8-пайплайн-трейсинга-и-sampling)
  * [Архитектура: агент/SDK → OTel Collector (receivers/processors/exporters) → хранилище (Tempo/Jaeger/X-Ray)](#архитектура-агентsdk--otel-collector-receiversprocessorsexporters--хранилище-tempojaegerx-ray)
  * [Head vs Tail-sampling: динамика по атрибутам (ошибки/долгие/VIP-клиенты/эндпойнты), бюджет %](#head-vs-tail-sampling-динамика-по-атрибутам-ошибкидолгиеvip-клиентыэндпойнты-бюджет-)
  * [Лимиты: размеры спана, количество атрибутов/событий/линков; батч-экспорт и бэкпрешр](#лимиты-размеры-спана-количество-атрибутовсобытийлинков-батч-экспорт-и-бэкпрешр)
  * [Безопасность: фильтрация/редакция PII, маппинг ресурсных атрибутов (service.name, env, region, pod)](#безопасность-фильтрацияредакция-pii-маппинг-ресурсных-атрибутов-servicename-env-region-pod)
* [9. Связь метрик ↔ трейсев ↔ логов](#9-связь-метрик--трейсев--логов)
  * [Exemplars в Prometheus/Grafana: прокладка «из p99 на конкретный trace»](#exemplars-в-prometheusgrafana-прокладка-из-p99-на-конкретный-trace)
  * [Span metrics: агрегация трасс в метрики (RPS/latency/error_rate), где это уместно](#span-metrics-агрегация-трасс-в-метрики-rpslatencyerror_rate-где-это-уместно)
  * [Корреляция логов: traceId/spanId в MDC, структурированные логи и sampling для шумных сервисов](#корреляция-логов-traceidspanid-в-mdc-структурированные-логи-и-sampling-для-шумных-сервисов)
  * [Диагностика инцидентов: сценарии «медленный эндпоинт», «шторм ретраев», «утечка кардинальности»](#диагностика-инцидентов-сценарии-медленный-эндпоинт-шторм-ретраев-утечка-кардинальности)
* [10. Kubernetes и governance](#10-kubernetes-и-governance)
  * [Prometheus Operator: ServiceMonitor/PodMonitor/Probe, scrape на sidecar/agent, метки/аннотации](#prometheus-operator-servicemonitorpodmonitorprobe-scrape-на-sidecaragent-меткианнотации)
  * [Multi-cluster/мультирегион: федерация/Thanos, tenant-изоляция и права](#multi-clusterмультирегион-федерацияthanos-tenant-изоляция-и-права)
  * [Политики: стандарты нейминга, «черный список» лейблов, ревью добавления метрик/тегов, лимиты ретеншна](#политики-стандарты-нейминга-черный-список-лейблов-ревью-добавления-метриктегов-лимиты-ретеншна)
  * [Runbooks и испытания: fault-injection (latency/packet-loss), нагрузка на хранилища, проверки MWMBR-алёртов в staging](#runbooks-и-испытания-fault-injection-latencypacket-loss-нагрузка-на-хранилища-проверки-mwmbr-алёртов-в-staging)
  * [Экономика: сэмплинг трейсинга, downsampling метрик, агрегации/recording rules вместо тяжёлых adhoc](#экономика-сэмплинг-трейсинга-downsampling-метрик-агрегацииrecording-rules-вместо-тяжёлых-adhoc)
<!-- TOC -->

# 0. Введение

## Что это

Наблюдаемость в контексте backend-систем — это способность по внешним сигналам понять, **что происходит внутри системы прямо сейчас** и **почему** она ведёт себя так, а не иначе. В отличие от «поставили пару графиков на CPU и живём», наблюдаемость предполагает, что у тебя есть достаточный набор метрик, логов и трейсов, чтобы ответить на вопросы, которые ты ещё не формулировал в момент написания кода. То есть не только «упало/не упало», а «почему упало именно здесь, именно для этой группы пользователей, именно в таких условиях».

Часто наблюдаемость противопоставляют мониторингу. Мониторинг — это статичный набор проверок и алёртов вроде «если CPU > 90% 5 минут — позвони в PagerDuty». Наблюдаемость шире: она включает мониторинг, но добавляет возможность **разведки** — когда у тебя есть необычный симптом (например, редкое увеличение p99 в одном регионе), ты должен по имеющимся данным отследить путь запроса, увидеть, какие зависимости тормозят, какие фичи активированы, где именно произошёл срыв SLA. Хорошая наблюдаемость делает это возможным без добавления кода прямо во время инцидента.

Традиционно выделяют три «столпа» наблюдаемости: **метрики, логи и трейсинг**. Метрики — агрегированные численные ряды (latency, error rate, количество запросов). Логи — детальные записи событий, часто структурированные. Трейсинг — цепочки спанов, связывающие путь одного конкретного запроса через множество сервисов. В расширенном понимании сюда добавляют ещё профилирование (JFR, flame graphs), дампы и даже бизнес-метрики (количество заказов, конверсии), но сердцем технической наблюдаемости остаются именно эти три.

В мире Spring Boot базовым слоем наблюдаемости является **Micrometer + Actuator**. Actuator даёт стандартные эндпоинты (`/actuator/health`, `/actuator/metrics`, `/actuator/prometheus`), а Micrometer предоставляет абстракцию поверх разных систем метрик (Prometheus, Graphite, Datadog и т.д.). Для трейсинга сейчас де-факто стандартом становится OpenTelemetry, который в Spring 3 интегрируется через Micrometer Tracing и Observation API — единый слой, через который ты можешь одновременно выдавать и метрики, и трейсы.

Важно понимать, что наблюдаемость — это не только про «подключить библиотеку», а про **дизайн системы**. Если твои сервисы не передают корректно traceId, если статусы и коды ошибок хаотичны, если бизнес-события не логируются в нужных местах, никакой самый модный стек не спасёт. Наблюдаемость — это свойство дизайна: как называются операции, какие идентификаторы используются, какие атрибуты попадают в метрики и спаны, как вы договорились именовать бизнес-метрики.

Ещё один момент: наблюдаемость — это **поток**. Есть слой инструментирования (где в коде рождаются метрики и спаны), слой сбора (агенты, экспортеры, Prometheus-scrape), слой хранения (TSDB, трейс-хранилища, лог-кластер) и слой визуализации/алёртов (Grafana, alertmanager, on-call). Ошибка на любом уровне ломает всю картину: если экспортер не отдаёт метрики, то красивая конфигурация алёртов бесполезна; если в коде отсутствует traceId, то Grafana Tempo покажет тебе красивые цепочки «без имени».

Для Java-микросервисов наблюдаемость неизбежно включает **JVM-специфику**: GC-паузы, размеры heap/metaspace, количество потоков, состояние пулов, размер очередей. Без этого ты не поймёшь, почему latency выросла: это проблема сети, базы, внешнего API или просто GC встал на 1.5 секунды, потому что кто-то привёз гигантский объект в память.

Отдельный слой — наблюдаемость **инфраструктуры**: Kubernetes-кластер, ноды, ingress-контроллеры, сетевые политики. Сервис может быть идеальным с точки зрения внутренних метрик, но requests всё равно падают, потому что kube-proxy чудит или ingress перегружен. В нормальной setup’е сигналы от приложений и от инфраструктуры живут в одном информационном поле (Prometheus/Grafana + трейсинг на уровне mesh/ingress).

Наблюдаемость становится особенно критичной в **реактивных и асинхронных** приложениях (WebFlux, Kafka, @Async). Там стек вызовов «расползается» по пулу потоков, и обычный stacktrace мало помогает. Без корректного переноса контекста в Reactor, без traceId в MDC и правильной маркировки спанов ты просто не поймёшь, какое событие привело к какому побочному эффекту.

В этом курсе мы говорим именно про «расширенную» наблюдаемость: не только «подключить Actuator», но и **спроектировать метрики**, выбрать правильные единицы, контролировать кардинальность, связывать метрики с трейсам через exemplars, строить дашборды под разные роли и превращать всё это в реальные SLO/алёрты. Плюс — покрыть JVM и пулы потоков, чтобы бэкэнд был наблюдаем не хуже, чем бизнес-логика.

Наконец, нужно держать в голове, что наблюдаемость — это **командная практика**, а не только забота DevOps/SRE. Разработчик, который пишет контроллер или Kafka-consumer, должен понимать, какие метрики и спаны нужны для его части системы, а SRE — как использовать эти сигналы для алёртов и расследований. Хорошая наблюдаемость получается только тогда, когда обе стороны работают вместе и говорят на одном языке.

Чтобы не быть совсем абстрактным, покажу минимальный скелет Spring Boot-приложения с Actuator и метриками, который станет базой для всех дальнейших примеров.

**Gradle: зависимости для базовой наблюдаемости (Actuator + Prometheus)**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — самый простой контроллер + кастомный Counter**

```java
package com.example.observability.intro;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IntroController {

    private final Counter helloCounter;

    public IntroController(MeterRegistry meterRegistry) {
        this.helloCounter = Counter.builder("demo_hello_requests_total")
                .description("Количество запросов к /hello")
                .tag("endpoint", "/hello")
                .register(meterRegistry);
    }

    @GetMapping("/hello")
    public String hello() {
        helloCounter.increment();
        return "Hello, observability!";
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.intro

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class IntroController(
    meterRegistry: MeterRegistry
) {

    private val helloCounter: Counter = Counter.builder("demo_hello_requests_total")
        .description("Количество запросов к /hello")
        .tag("endpoint", "/hello")
        .register(meterRegistry)

    @GetMapping("/hello")
    fun hello(): String {
        helloCounter.increment()
        return "Hello, observability!"
    }
}
```

Этот код демонстрирует базовую идею: при каждом запросе к `/hello` мы увеличиваем счётчик, а Prometheus может забирать его через `/actuator/prometheus`. Дальше на этом фундаменте будем строить гораздо более богатую схему метрик и трейсов.

---

## Зачем это

Главный ответ на вопрос «зачем нужна наблюдаемость» — чтобы **сократить время от первой аномалии до понимания причины и исправления**. В реальной жизни проблемы редко выглядят как «всё упало для всех сразу». Гораздо чаще это тонкие эффекты: в одном регионе периодически растёт latency, у определённой группы клиентов растёт error rate, время от клика до подтверждения заказа скачет без видимой причины. Без наблюдаемости такие вещи неделями висят в виде «странной жалобы в чате поддержки».

Наблюдаемость превращает систему из «чёрного ящика» в «просвечиваемый» объект. Когда приходит инцидент — алёрт или тикет — хорошая наблюдаемость даёт тебе возможность **формулировать и проверять гипотезы**: «Тормозим ли мы на БД?», «Подозрительно много retry у внешнего сервиса?», «GC стал чаще?», «Лимиты пула достигнуты?». Вместо «я не знаю, где копать» ты по цепочке от SLO-алёрта идёшь к конкретному трейсингу запроса, к конкретной зависимости и к конкретному куску кода.

Вторая мотивация — **управление рисками изменений**. Без наблюдаемости любой релиз — это «надеемся, что всё ОК». При нормальном стеке у тебя есть baseline-графики (до релиза) и возможность сравнить их с метриками после выката: p95/p99 latency, error rate, нагрузка на БД, количество таймаутов. Это позволяет осознанно использовать canary-релизы, progressive delivery и kill-switch’и: вы откатываете не по субъективному «кажется, стало хуже», а по конкретным SLO.

Третья причина — **оптимизация ресурсов и стоимости**. Метрики CPU/memory/GC, количество запросов, hit ratio кэша и latency внешних API позволяют принимать решения, основанные на фактах: действительно ли нужно увеличивать реплики, оправдан ли переход на более дорогой managed-сервис, выгодно ли добавить кэш перед медленной БД. Без этого легко либо переплатить за ресурсы, либо выжать железо «до свиста», а потом месяц ловить странные провалы по latency.

Наблюдаемость также критична для **диагностики проблем производительности**, которые проявляются только под нагрузкой. Локально всё выглядит отлично, а на продакшне часть запросов внезапно упирается в JPA N+1, синхронные вызовы между сервисами или неожиданные блокировки. С правильным набором метрик и трейсов ты видишь, где именно расходуется время: в приложении, в БД, в сети, в внешнем API или в кэше.

Отдельный слой мотивации — **коммуникация с бизнесом и заказчиками**. Когда у тебя есть SLO по доступности и времени ответа, ты можешь честно ответить на вопросы «как часто у нас падает сервис», «как быстро мы восстанавливаемся», «сколько пользователей реально страдает». Это совсем другой уровень разговора, чем «ну, вроде редко падает» или «мы посмотрели логи — вроде всё нормально».

И, наконец, наблюдаемость помогает **делать разработку безопаснее** для команды. Когда разработчик знает, что его сервис хорошо инструментирован, ему легче внедрять изменения: он понимает, что любые побочные эффекты будут видны по метрикам и алёртам, а не всплывут через месяц как странный баг. Это снижает страх перед рефакторингом и позволяет двигаться быстрее без превращения продакшна в минное поле.

Чтобы показать практический смысл даже на уровне «зачем», приведу простой пример: мы оборачиваем бизнес-метод в таймер Micrometer, чтобы всегда видеть, сколько реально занимает операция. Это тот же «зачем»: мы хотим иметь объективные цифры, а не впечатления.

**Gradle: Micrometer Timer в Spring Boot**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'io.micrometer:micrometer-core'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("io.micrometer:micrometer-core")
}
```

**Java — измеряем время обработки бизнес-операции**

```java
package com.example.observability.why;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class OrderProcessingService {

    private final Timer processingTimer;

    public OrderProcessingService(MeterRegistry meterRegistry) {
        this.processingTimer = Timer.builder("order_processing_duration")
                .description("Время обработки заказа")
                .publishPercentileHistogram()
                .maximumExpectedValue(Duration.ofSeconds(10))
                .register(meterRegistry);
    }

    public void processOrder(String orderId) {
        processingTimer.record(() -> {
            // здесь могла бы быть реальная логика
            simulateWork();
        });
    }

    private void simulateWork() {
        try {
            Thread.sleep(200L); // имитация I/O или бизнес-логики
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.why

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class OrderProcessingService(
    meterRegistry: MeterRegistry
) {

    private val processingTimer: Timer = Timer.builder("order_processing_duration")
        .description("Время обработки заказа")
        .publishPercentileHistogram()
        .maximumExpectedValue(Duration.ofSeconds(10))
        .register(meterRegistry)

    fun processOrder(orderId: String) {
        processingTimer.record {
            simulateWork()
        }
    }

    private fun simulateWork() {
        try {
            Thread.sleep(200L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

Это всё ещё простейший пример, но он уже показывает «зачем»: по этой метрике можно строить SLO по времени обработки заказа и измерять эффект любых изменений кода или инфраструктуры.

---

## Где используется

Наблюдаемость нужна везде, где от системы ждут **предсказуемого поведения под нагрузкой** и **быстрого восстановления при инцидентах**. В первую очередь это микросервисные и распределённые системы: десятки сервисов, очереди, брокеры, внешние API, разные команды. Здесь цепочка одного запроса может пройти через 5–10 hop’ов, и без метрик и трейсинга ты просто не узнаешь, где именно «тонкое место».

В монолите наблюдаемость ничуть не менее важна: да, всё в одном процессе, но проблем не меньше — зависания БД, GC, блокировки потоков, конкуренция за пулы. Разница лишь в том, что трейсинг будет в основном внутри одного процесса, а не через сеть. Тем не менее те же SLO, метрики latency и error rate, джоб-метрики для batch-задач и GC-паузы остаются настолько же критичными.

В системах, ориентированных на **обработку событий** (Kafka, JMS, другие брокеры), наблюдаемость нужна для понимания того, как ведут себя consumer’ы: что с лагом по топикам, не перегружены ли партиции, нет ли «яловых» consumer group’ов, где один инстанс перегружен, а другие простаивают. Здесь важны метрики типа `records_lag`, `records_consumed_rate`, latency обработки сообщений и количество ошибок/дед-леттеров.

В системах, где много **фоновых задач и планировщиков** (@Scheduled, Spring Batch, внешние воркеры), наблюдаемость помогает контролировать время выполнения задач, частоту ошибок, объем обрабатываемых данных. Без этого легко получить сценарий «ночная джоба иногда не успевает» или «batch-процесс забивает БД до утра», и никто не поймёт, что происходит, пока бизнес не пожалуется.

Особенно критична наблюдаемость для **клиентских API**, через которые ходят мобильные и веб-приложения. Здесь важно не только «не падать», но и держать приемлемую latency и доступность для пользователя. Метрики по HTTP-эндпоинтам, распределение latency по статус-кодам, трейсинг с клиентскими атрибутами (тип клиента, версия приложения, регион) позволяют точно понимать, какие сегменты пользователей страдают во время инцидента.

В high-risk доменах — финтех, health-care, биллинг — наблюдаемость становится частью **комплаенса и отчётности**. История изменений, аудиторские логи, трейсы критичных операций вроде списания и возврата, метрики отказов по платёжным провайдерам — всё это потом попадает в отчёты для регуляторов, партнёров и внутренних аудиторов.

Внутри команды разработчиков наблюдаемость используется ещё и как **инструмент обратной связи**. Когда ты выкатываешь новую версию фичи, можно отслеживать не только технические метрики, но и бизнес-метрики: как изменилась конверсия, вырос ли средний чек, появилась ли аномальная нагрузка на соседние сервисы. Это помогает делать A/B-тесты и принимать решения об отключении/включении фичей на основе данных.

Наконец, наблюдаемость используется в **обучении и онбординге**. Новому разработчику гораздо проще «почувствовать» реальную систему, когда он может открыть Grafana и посмотреть реальные графики своего сервиса, увидеть трейсинг запроса от API-гейта до базы, посмотреть лог-записи с traceId. Это гораздо эффективнее, чем читать «архитектурный документ», написанный год назад.

Поскольку мы говорим про Spring Boot, естественное место применения наблюдаемости — это сами Spring Boot-сервисы, поднятые в Docker/Kubernetes. В них Actuator даёт экспозицию метрик, Micrometer интегрирует всё это с Prometheus/OTel, а дальше уже Grafana и трейс-бекенд (Jaeger, Tempo, OTEL collector) позволяют использовать эти сигналы по полной.

Чтобы показать, «где» наблюдаемость появляется в обычном контроллере, приведу упрощённый пример: HTTP-эндпоинт, который обращается к внешнему сервису и отдаёт результат. Здесь удобно навесить и метрику, и трейсинг — и использовать это на дашбордах.

**Gradle: web + actuator + tracing (минимальный стек)**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
}
```

**Java — контроллер, который сразу попадает в HTTP-метрики и трейсы**

```java
package com.example.observability.where;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProfileController {

    private final Timer profileTimer;

    public ProfileController(MeterRegistry meterRegistry) {
        this.profileTimer = Timer.builder("profile_fetch_duration")
                .description("Время получения профиля пользователя")
                .tag("service", "profile")
                .publishPercentileHistogram()
                .register(meterRegistry);
    }

    @GetMapping("/api/profile/{id}")
    public String getProfile(@PathVariable String id) {
        return profileTimer.record(() -> {
            // Здесь мог бы быть вызов к базе или внешнему API
            simulateExternalCall();
            return "profile:" + id;
        });
    }

    private void simulateExternalCall() {
        try {
            Thread.sleep(100L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.where

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RestController

@RestController
class ProfileController(
    meterRegistry: MeterRegistry
) {

    private val profileTimer: Timer = Timer.builder("profile_fetch_duration")
        .description("Время получения профиля пользователя")
        .tag("service", "profile")
        .publishPercentileHistogram()
        .register(meterRegistry)

    @GetMapping("/api/profile/{id}")
    fun getProfile(@PathVariable id: String): String {
        return profileTimer.record<String> {
            simulateExternalCall()
            "profile:$id"
        }
    }

    private fun simulateExternalCall() {
        try {
            Thread.sleep(100L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

Этот код показывает, как наблюдаемость «встроена» прямо в бизнес-эндпоинт: каждый вызов попадает в стандартные HTTP-метрики Spring Boot, в кастомный таймер `profile_fetch_duration` и в трейсинг (если он подключён). Дальше эти сигналы используются и SRE, и разработчиками, и иногда даже бизнесом (через агрегированные отчёты).

---

## Какие проблемы решает

Без наблюдаемости продакшн превращается в черный ящик, и большинство проблем решаются через «угадай по логам». Основная проблема, которую наблюдаемость снимает, — это **непредсказуемость и долгие расследования**. Время от инцидента до понимания «что вообще произошло» может уходить в часы или дни, особенно если в системе участвует несколько команд и внешних провайдеров. Метрики и трейсы позволяют резко сократить это время: вместо вручную собираемых логов ты за пару минут видишь, где именно в цепочке возникла аномалия.

Вторая большая проблема — **отсутствие объективных критериев качества**. Если у тебя нет SLI/SLO и соответствующих метрик, ты не можешь честно сказать, работает ли система хорошо. Пара случайных жалоб в поддержку может скрывать серьёзную деградацию в определённом сегменте клиентов, а может быть просто шумом. Наблюдаемость даёт количественные показатели: «99% запросов заканчиваются за X мс», «error rate за последние 30 минут ниже Y%», «все nightly batch’и завершились в срок».

Третья проблема — **невидимый технический долг и «скрытые» bottleneck’и**. Когда система растёт, в ней накапливаются медленные запросы к БД, неоптимальные алгоритмы, неоправданно «толстые» DTO, зависимость от медленных внешних API. Без метрик latency и профилей нагрузки эти вещи проявляются только в пиковые моменты (Чёрная Пятница, массовые рассылки), и часто уже слишком поздно. С хорошими дашбордами ты видишь такие проблемы задолго до того, как они превращаются в инцидент.

Наблюдаемость помогает решать и проблему **неочевидных кросс-эффектов между сервисами**. Например, включение тяжёлой отчётной функции в одном сервисе может незаметно убить latency другого, потому что оба бьются в одну и ту же базу или Redis-кластер. По трейсам и метрикам хорошо видно, как меняется нагрузка на общие зависимости, и можно вовремя поставить дополнительные лимиты или вынести хранилище.

Ещё один класс проблем — **«фликерующие» ошибки**: редкие, трудно воспроизводимые баги, которые проявляются только в определённых комбинациях данных или под специфической нагрузкой. Без трейсов с атрибутами (userId, тип клиента, feature-флаги) и без достаточной детализации логов такие вещи почти нереально поймать. Наблюдаемость даёт возможность выделять и исследовать именно такие «хвостовые» кейсы.

С точки зрения эксплуатаций наблюдаемость решает проблему **«инциденты без контекста»**. Алёрт «5xx выросли» сам по себе мало полезен. Алёрт «5xx на /api/checkout для клиентов региона EU выросли до 10% за последние 5 минут, breaker paymentsApi в OPEN» — уже почти готовый диагноз. Для этого метрики и трейсы должны быть достаточно богатыми, а алёрты — правильно построенными.

Также наблюдаемость позволяет решать проблему **неоптимальных или опасных изменений конфигурации**. Без метрик и snapshot-тестов по конфигу легко случайно увеличить таймаут внешней зависимости, выключить breaker или задрать retry до безумных значений. С нормальной наблюдаемостью такие изменения либо не проходят CI-гейты, либо быстро обнаруживаются по деградации SLO и rollback’у.

Наконец, наблюдаемость решает проблему **отсутствия коллективной памяти**. Пост-мортемы, построенные на данных (графики, трейсы, лог-фрагменты), превращают «однажды было что-то странное» в документированную историю: почему случился инцидент, какие сигналы сработали, какие нет, что было исправлено. Это позволяет не наступать на одни и те же грабли и строить процессы SRE, основанные на реальных данных.

Чтобы связать всё это с кодом, покажу пример использования Observation API для обёртки бизнес-операции: здесь мы одновременно получаем и метрику, и трейс, и контекст, который потом поможет разбирать проблемы.

**Gradle: Observation + Micrometer Tracing**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-observation'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-observation")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
}
```

**Java — Observation вокруг бизнес-операции**

```java
package com.example.observability.problems;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.stereotype.Service;

@Service
public class CheckoutService {

    private final ObservationRegistry observationRegistry;

    public CheckoutService(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    public void checkout(String userId, long amount) {
        Observation.createNotStarted("checkout.operation", observationRegistry)
                .lowCardinalityKeyValue("user_segment", segmentFor(userId))
                .lowCardinalityKeyValue("operation", "checkout")
                .observe(() -> {
                    // здесь бизнес-логика оформления заказа
                    simulateRiskyCall();
                });
    }

    private String segmentFor(String userId) {
        return userId.startsWith("vip-") ? "vip" : "regular";
    }

    private void simulateRiskyCall() {
        try {
            Thread.sleep(150L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример Observation**

```kotlin
package com.example.observability.problems

import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import org.springframework.stereotype.Service

@Service
class CheckoutService(
    private val observationRegistry: ObservationRegistry
) {

    fun checkout(userId: String, amount: Long) {
        Observation.createNotStarted("checkout.operation", observationRegistry)
            .lowCardinalityKeyValue("user_segment", segmentFor(userId))
            .lowCardinalityKeyValue("operation", "checkout")
            .observe {
                simulateRiskyCall()
            }
    }

    private fun segmentFor(userId: String): String =
        if (userId.startsWith("vip-")) "vip" else "regular"

    private fun simulateRiskyCall() {
        try {
            Thread.sleep(150L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

Такая обёртка решает сразу несколько проблем: даёт метрики по latency/успехам операции `checkout`, добавляет теги для разбивки по сегментам, генерирует трейсы для последующего drill-down и делает возможным полноценный пост-мортем по данным, если вдруг что-то пойдёт не так.

# 1. Цели и таксономия наблюдаемости

## SLI/SLO/SLA

В основе современной наблюдаемости лежит связка трёх аббревиатур: **SLI, SLO и SLA**. Это не «модные слова из SRE-книги», а очень приземлённые инструменты: они превращают абстрактное «система вроде работает» в конкретные цифры и договорённости. **SLI (Service Level Indicator)** — это измеряемый показатель качества сервиса: доля успешных запросов, p95 латентности, свежесть данных и т.д. **SLO (Service Level Objective)** — целевой диапазон для SLI («99.9% запросов к /api/pay за 30 дней должны заканчиваться быстрее 500 мс»). **SLA (Service Level Agreement)** — юридический или контрактный документ, который ссылается на SLO и описывает последствия нарушения (скидки, штрафы, кредиты).

Важно понимать иерархию: сначала вы выбираете, **что именно важно для пользователя** (SLI), затем ставите для этого адекватную цель (SLO), и лишь потом, если нужно, упаковываете это в SLA для внешних клиентов. Если перепутать порядок и начать с «нам нужен SLA 99.99% потому что так делают большие компании», получится либо заведомо невыполнимая цель, либо бессмысленная цифра, которая никак не связана с реальным опытом пользователей.

SLI должны быть **измеримыми и привязанными к пользовательскому опыту**. Пример: для API платежей разумный SLI — это процент запросов, которые завершились «успешно и вовремя» (например, статус 2xx и время ответа менее 800 мс). Количество pod’ов в Kubernetes — уже не SLI, это внутренняя техническая метрика. Если вы не можете выразить через SLI: «пользователю было хорошо или плохо», значит, это не тот показатель, который нужно выносить на уровень SLO.

SLO — это не «красивые числа», а **компромисс между бизнесом, разработкой и эксплуатацией**. Чем выше SLO (например, 99.99% вместо 99.5%), тем дороже инфраструктура, жёстче процессы релизов, более строгие требования к тестированию и устойчивая архитектура. Ошибка начинающих команд — сразу ставить себе «пять девяток», не имея ни отказоустойчивой архитектуры, ни опыта эксплуатации. Гораздо честнее начать с более скромного SLO, отмерить фактическое поведение системы, а потом постепенно его улучшать.

Внутри одной системы может быть несколько уровней SLO: **на уровне сервиса** («наш сервис отвечает с p95 < 300 мс»), **на уровне эндпоинтов** («/api/pay должен быть быстрее /api/report»), **на уровне бизнес-сценариев** («оформление заказа от клика до подтверждения»). Для разработчика Spring Boot-сервиса практичнее всего начинать с SLO на уровне HTTP API и затем добавлять бизнес-сценарные SLO, когда команда привыкнет к базовым метрикам.

Практически SLI реализуются через **метрики**. Например, Spring Boot Actuator автоматически даёт `http_server_requests_seconds_*`, и вы можете построить SLI вида: `SLI = 1 - (кол-во 5xx / общее кол-во запросов)` за окно в 30 дней. Другой SLI — доля запросов с латентностью < 500 мс. И для первого, и для второго вы сможете задавать SLO: «не меньше 99.5% успешных» или «не меньше 99% быстрых». Наблюдаемость здесь — не самоцель, а способ поддерживать SLO в допустимом диапазоне и видеть, когда вы начали «проедать error budget».

**Error budget** — это «запас на ошибки», который вы готовы принять в рамках SLO. Если SLO по доступности — 99.9% за месяц, это значит, что у вас есть примерно 43 минуты «разрешённого даунтайма» (или эквивалент по ошибкам и медленным ответам). Наблюдаемость позволяет не просто видеть, сколько бюджета вы уже потратили, но и принимать решения: «мы уже сожгли 70% бюджета — давайте притормозим с экспериментами и релизами».

Тесная связка SLI/SLO с наблюдаемостью проявляется и в **процессах**. SLO не живут отдельно от графиков и алёртов: на них должны быть построены дашборды, и по ним должны завязываться алёрты уровня on-call. Типичный шаблон: два алёрта — «предупреждение» (error rate > 1% некоторое время) и «paging» (error rate > 5% или SLO нарушается с высокой скоростью). Без наблюдаемости SLO превращаются в презентацию, которую вспоминают только на ретроспективах.

Есть несколько типичных ошибок при работе с этой троицей. Первая — **слишком много SLO**: когда у каждого эндпоинта по 5–7 целей, команда перестаёт на них смотреть. Лучше иметь 2–3 ключевых SLO на сервис и 1–2 на бизнес-сценарий. Вторая — **путаница SLO и SLA**: на уровне команды вы можете иметь достаточно жёсткие внутренние SLO и при этом более мягкие SLA для клиентов, чтобы иметь запас. Третья — **непоследовательность во времени**: менять SLO нужно осознанно, а не каждый спринт в ответ на единичные инциденты.

В итоге SLI/SLO/SLA задают **рамку, зачем вообще всё это измерять**. Без них наблюдаемость легко скатывается в «доставку графиков ради графиков». Когда же есть понятные SLO и error budget, становится ясно, какие метрики важны, какие алёрты нужны и какие изменения в коде стоит приоритизировать.

Чтобы связать это с реальным кодом, покажем простой пример: мы добавляем HTTP-метрики и кастомный счётчик ошибок, который затем используем как SLI «доля успешных запросов».

**Gradle: базовые зависимости для HTTP-метрик и Prometheus**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — контроллер с явным счётчиком ошибок как SLI**

```java
package com.example.observability.sli;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/payments")
public class PaymentsController {

    private final Counter successCounter;
    private final Counter errorCounter;

    public PaymentsController(MeterRegistry meterRegistry) {
        this.successCounter = Counter.builder("payments_requests_total")
                .description("Успешные запросы к платежному API")
                .tag("status", "success")
                .register(meterRegistry);

        this.errorCounter = Counter.builder("payments_requests_total")
                .description("Ошибка при обработке платежного запроса")
                .tag("status", "error")
                .register(meterRegistry);
    }

    @PostMapping
    public ResponseEntity<String> charge(@RequestParam("userId") String userId,
                                         @RequestParam("amount") long amount) {
        try {
            // здесь могла бы быть реальная логика платежа
            if (amount <= 0) {
                throw new IllegalArgumentException("Amount must be positive");
            }
            successCounter.increment();
            return ResponseEntity.ok("OK");
        } catch (Exception ex) {
            errorCounter.increment();
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                    .body("ERROR: " + ex.getMessage());
        }
    }
}
```

**Kotlin — тот же контроллер**

```kotlin
package com.example.observability.sli

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/payments")
class PaymentsController(
    meterRegistry: MeterRegistry
) {

    private val successCounter: Counter = Counter.builder("payments_requests_total")
        .description("Успешные запросы к платежному API")
        .tag("status", "success")
        .register(meterRegistry)

    private val errorCounter: Counter = Counter.builder("payments_requests_total")
        .description("Ошибка при обработке платежного запроса")
        .tag("status", "error")
        .register(meterRegistry)

    @PostMapping
    fun charge(
        @RequestParam("userId") userId: String,
        @RequestParam("amount") amount: Long
    ): ResponseEntity<String> {
        return try {
            if (amount <= 0) {
                throw IllegalArgumentException("Amount must be positive")
            }
            successCounter.increment()
            ResponseEntity.ok("OK")
        } catch (ex: Exception) {
            errorCounter.increment()
            ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body("ERROR: ${ex.message}")
        }
    }
}
```

В Prometheus по этим метрикам легко сформировать SLI: `success_rate = success / (success + error)` за последний период. На него уже можно вешать SLO и алёрты.

---

## RED/Golden Signals и USE

Когда SLI/SLO дают рамку «что считать», возникает вопрос «а какие именно показатели для сервисов и ресурсов выбирать». Тут помогают наборы «best practices» — **RED, Golden Signals и USE**. Они задают минимальный набор метрик, без которых говорить о наблюдаемости сервисов и инфраструктуры сложно.

Модель **RED (Rate, Errors, Duration)** фокусируется на сервисах и API. **Rate** — скорость обработки запросов (RPS), **Errors** — доля неуспешных запросов (ошибки или нежелательные ответы), **Duration** — время обработки запросов, как правило p95/p99. Прелесть RED в том, что он простой, универсальный и хорошо ложится на HTTP, gRPC, message consumption и т.д. Для любого эндпоинта или RPC-вызова ты можешь построить три графика и уже по ним понять, жив он или страдает.

**Golden Signals** из Google SRE расширяют RED до четырёх сигналов: **latency, traffic, errors, saturation**. Первые три практически совпадают с RED, а **saturation** добавляет измерение «насколько мы близки к пределу». Для HTTP-сервиса saturation — это загрузка CPU, глубина очередей, использование пула потоков, количество открытых соединений. Для Kafka-consumer’а — лаг по партициям и загрузка consumer’ов. Идея в том, что много проблем видно в saturations ещё до того, как они превращаются в заметный рост latency и errors.

Модель **USE (Utilization, Saturation, Errors)** применяют к **ресурсам**: CPU, память, диски, сеть, пулы потоков, кластеры. **Utilization** — доля использования (например, CPU 70%, heap 60%), **Saturation** — величина очередей и «давление» (run queue длины, глубина очередей в пулах), **Errors** — явные ошибки (дропы пакетов, ошибки диска, rejected tasks). USE полезен, когда ты ищешь корень проблемы: сервис видит рост latency, а ты выясняешь, что на ноде, где он крутится, диск в saturation, а CPU забит GC.

На практике RED/Golden Signals и USE работают в связке. Сначала ты видишь по RED/Golden Signals, что у сервиса выросла latency и error rate на определённом эндпоинте. Потом по USE смотришь, что происходит с его ресурсами: нет ли переполненных пулов потоков, не забит ли GC, не выросла ли очередь задач. Если ресурсы выглядят нормально, идёшь дальше вниз по зависимостям: БД, кэш, внешние API.

В Spring Boot половину RED/Golden Signals можно получить «из коробки» благодаря Actuator и Micrometer. Метрики вида `http_server_requests_seconds_*` дают latency и rate по HTTP, `tomcat_threads_*` (или jetty/netty-аналог) — saturation по пулу HTTP-потоков, `jvm_*` — USE по памяти и GC. Задача команды — не «изобрести» сигналы, а правильно их выбрать, назвать и вывести на дашборды.

С точки зрения дизайна метрик важно не только иметь нужные сигналы, но и **не взорвать кардинальность**. Желание тегировать `http_server_requests_seconds` по `userId` приводит к десяткам тысяч time series и падению Prometheus. RED/Golden Signals предполагают аккуратные теги: `status`, `outcome`, `uri_pattern`, `method`. Для бизнес-аналитики используются другие инструменты; на уровне технических метрик идентификаторы пользователей стоит либо хэшировать, либо избегать вообще.

Для USE-модели в Spring-приложении полезно отдельно выставить метрики по **пулами потоков и очередям**: `ThreadPoolTaskExecutor`, очереди задач, пул соединений к БД. Да, у HikariCP уже есть свои метрики, но своё приложение часто имеет и внутренние executors, про которые Prometheus сам ничего не знает. Небольшой кусок кода, регистрирующий gauge по глубине очереди и количеству активных потоков, даёт мощный сигнал saturation.

На уровне дашбордов под один сервис обычно делают **RED/Golden Signals-доску** (latency, error rate, RPS, saturation) и **USE-доску** по JVM и пулам. Такой набор покрывает подавляющее большинство инцидентов: сразу видно, случилось ли что-то с самим сервисом, с его ресурсами или с зависимостями. Остальные метрики добавляются точечно под конкретные домены или проблемы.

Чтобы показать, как на практике собрать часть USE-сигналов, возьмём `ThreadPoolTaskExecutor`, который используется для @Async-методов, и зарегистрируем по нему gauges: размер пула, активные потоки, глубина очереди. Это классический пример ресурсов, на которые редко смотрят, пока они не начинают отстреливать `RejectedExecutionException`.

**Gradle: зависимости для Micrometer и @Async**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework:spring-context'
    implementation 'io.micrometer:micrometer-core'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework:spring-context")
    implementation("io.micrometer:micrometer-core")
}
```

**Java — регистрация USE-метрик для пула потоков**

```java
package com.example.observability.reduse;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.BlockingQueue;

@Configuration
public class AsyncExecutorConfig {

    @Bean
    public ThreadPoolTaskExecutor businessExecutor(MeterRegistry meterRegistry) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setThreadNamePrefix("business-");
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.initialize();

        BlockingQueue<Runnable> queue = executor.getThreadPoolExecutor().getQueue();

        meterRegistry.gauge("executor_business_pool_size", executor,
                exec -> exec.getThreadPoolExecutor().getPoolSize());
        meterRegistry.gauge("executor_business_active_threads", executor,
                exec -> exec.getThreadPoolExecutor().getActiveCount());
        meterRegistry.gauge("executor_business_queue_size", queue, BlockingQueue::size);

        return executor;
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.reduse

import io.micrometer.core.instrument.MeterRegistry
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import java.util.concurrent.BlockingQueue

@Configuration
class AsyncExecutorConfig {

    @Bean
    fun businessExecutor(meterRegistry: MeterRegistry): ThreadPoolTaskExecutor {
        val executor = ThreadPoolTaskExecutor().apply {
            threadNamePrefix = "business-"
            corePoolSize = 10
            maxPoolSize = 20
            setQueueCapacity(100)
            initialize()
        }

        val queue: BlockingQueue<Runnable> = executor.threadPoolExecutor.queue

        meterRegistry.gauge("executor_business_pool_size", executor) {
            it.threadPoolExecutor.poolSize.toDouble()
        }
        meterRegistry.gauge("executor_business_active_threads", executor) {
            it.threadPoolExecutor.activeCount.toDouble()
        }
        meterRegistry.gauge("executor_business_queue_size", queue) {
            it.size.toDouble()
        }

        return executor
    }
}
```

Такие метрики — это прямое воплощение USE-модели: utilization (pool_size), saturation (queue_size, active_threads), errors (rejected — можно регистрировать от `RejectedExecutionHandler`). В связке с RED/Golden Signals это даёт почти полную картину того, как сервис переживает нагрузку.

---

## Стоимость и риск

Наблюдаемость, как и любая инженерная практика, **стоит денег** — буквально и фигурально. Метрики и трейсы занимают место в хранилищах, агенты и экспортеры потребляют CPU и память, разработчики тратят время на инструментирование и поддержку дашбордов. Любое решение «давайте всё померим» должно быть привязано к **риску**, который вы хотите снизить: потеря данных, длительный даунтайм, невыполнение SLA, невозможность расследовать инцидент.

Самая очевидная статья затрат — **хранение и обработка данных наблюдаемости**. В Prometheus каждая time series — это память и диск; если бездумно добавлять label’ы вроде `userId` и `orderId`, кардинальность растёт экспоненциально, и в какой-то момент TSDB начинает просто умирать. Trace-бекенды (Jaeger, Tempo, OTEL Collector + backend) также дороги: хранить 100% трейсов для high-load-сервиса почти всегда невозможно, приходится семплировать. Логи — это отдельная боль: пара неудачных циклов debug-логов на высоком трафике легко выливаются в десятки гигабайт в день.

Есть и **непрямые** затраты. Каждый дополнительный график, алёрт или метрика требует ментального пространства: если у вас 200 графиков для одного сервиса, разработчик или SRE просто перестаёт ими пользоваться. Слишком много алёртов превращают on-call в «шумовую атаку»: люди перестают реагировать, а нужные сигналы тонут в потоке «minor warning». Поэтому любые новые метрики и алёрты нужно пропускать через фильтр «какой риск они помогают контролировать».

Риск, который наблюдаемость удерживает, — это не только «упадёт/не упадёт». Есть риск **незаметной деградации** (SLO медленно ухудшается, а вы не видите), риск **невозможности расследования** (инцидент был, но данных недостаточно, чтобы понять причину), риск **ошибочных решений** (вы оптимизируете не то, что реально тормозит). На стороне бизнеса — риск потерянной выручки и доверия пользователей, когда инциденты происходят часто и непредсказуемо.

По этой причине уровень наблюдаемости должен **соответствовать критичности сервиса**. Для вспомогательного сервиса, обслуживающего внутренний репортинг, можно обойтись базовыми метриками RED и несколькими алёртами. Для платёжного шлюза или авторизации наблюдаемость должна быть гораздо глубже: SLO, подробные трейсы критичных операций, отдельные дашборды, более долгий retention, fine-grained алёрты. Подход «один размер на всех» здесь не работает.

Отдельный вид риска — **само наблюдение может сломать систему**. Плохое решение по метрикам (высокая кардинальность), жадный экспортер, JMX-коллектор, который под нагрузкой нагружает саму JVM — всё это реальная практика. Поэтому важно соблюдать разумные пределы: не делать heavy-операции в экспортерах, не лезть каждые 5 секунд в тяжёлый JMX без нужды, не включать 100% sampling для трейсинга на критичных сервисах без реальной необходимости.

Наблюдаемость влияет и на **скорость разработки**. С одной стороны, инструментирование требует времени: продумать метрики, прописать Observation, собрать дашборды. С другой — без неё вы тратите гораздо больше времени на расследование и «слепые» попытки оптимизации. В долгую устойчивые сервисы выигрывают именно за счёт того, что инженер час потратил на метрики и treсing, а не неделю на отлов редкого бага в проде.

Нельзя забывать и про **стоимость ложных алёртов и выгорание on-call**. Если наблюдаемость настроена так, что SRE просыпается каждую ночь из-за случайных всплесков p95, это прямой путь к снижению качества работы и росту рисков. Алёрты должны быть жёстко привязаны к SLO и бизнес-рискам, с понятными runbook’ами и приоритизацией: где достаточно уведомления в Slack, а где нужен немедленный paging.

Хорошая практика — относиться к наблюдаемости как к **продукту**: оценивать её «ROI». Если у вас есть метрики, которые никто не использует ни в алёртах, ни в расследованиях, возможно, их стоит удалить или не хранить долго. Если какой-то дашборд каждый инцидент спасает кучу времени, может быть, имеет смысл расширить его и инвестировать в него больше усилий. Наблюдаемость должна эволюционировать вместе с системой, а не обрастать мусором.

На уровне кода «стоимость» и «риск» проявляются даже в выборе того, какие теги вешать на Observation. `lowCardinalityKeyValue` (Micrometer) хороши для метрик, `highCardinalityKeyValue` — опасны для Prometheus, но полезны для трейсинга. Грамотное разделение — когда вы используете `lowCardinality` для классов операций и статусов, а high-cardinality атрибуты (например, отдельный orderId) уносятся в трейсы и логи.

Ниже покажу небольшой пример, где мы явно разделяем low- и high-cardinality атрибуты: это иллюстрация того, как на уровне кода управлять стоимостью наблюдаемости, не теряя полезной информации для расследований.

**Gradle: зависимости для Observation API**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-observation'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-observation")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
}
```

**Java — управление кардинальностью через Observation**

```java
package com.example.observability.cost;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.stereotype.Service;

@Service
public class InvoiceService {

    private final ObservationRegistry observationRegistry;

    public InvoiceService(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    public void generateInvoice(String customerSegment, String invoiceId) {
        Observation.createNotStarted("invoice.generate", observationRegistry)
                .lowCardinalityKeyValue("segment", customerSegment)
                .lowCardinalityKeyValue("operation", "generate")
                // высокий разброс значений: только для трейсов, а не для агрегированных метрик
                .highCardinalityKeyValue("invoice.id", invoiceId)
                .observe(this::doGenerate);
    }

    private void doGenerate() {
        try {
            Thread.sleep(120L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.cost

import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import org.springframework.stereotype.Service

@Service
class InvoiceService(
    private val observationRegistry: ObservationRegistry
) {

    fun generateInvoice(customerSegment: String, invoiceId: String) {
        Observation.createNotStarted("invoice.generate", observationRegistry)
            .lowCardinalityKeyValue("segment", customerSegment)
            .lowCardinalityKeyValue("operation", "generate")
            // высокий разброс значений: только для трейсов, а не для агрегированных метрик
            .highCardinalityKeyValue("invoice.id", invoiceId)
            .observe {
                doGenerate()
            }
    }

    private fun doGenerate() {
        try {
            Thread.sleep(120L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

Здесь сегмент клиента (`segment`) попадает в метрики и используется в SLI/SLO по сегментам, а конкретный `invoiceId` — только в трейсы и логи, где его можно использовать для детального расследования, не раздувая кардинальность метрик. Это простой пример того, как **осознанно управлять стоимостью наблюдаемости**, снижая риск перегрузить инфраструктуру мониторинга и не утонуть в данных.

# 2. Дизайн метрик: типы, единицы, кардинальность, гистограммы

## Типы: Counter/Gauge/Timer/DistributionSummary/Histogram

Первое, с чего начинается нормальный дизайн метрик, — это правильный выбор **типа**. Micrometer (и назад — Prometheus, OpenTelemetry) сознательно ограничивают набор примитивов: счётчики, gauge’и, таймеры и распределения. Идея в том, что ты не придумываешь произвольные «метрики любого вида», а выбираешь подходящий инструмент под задачу, чтобы мониторинговая БД могла эффективно агрегировать и интерпретировать данные. Ошибка здесь приводит к кривым графикам: медленное растущее значение в gauge вместо счётчика, измерение latency в gauge вместо таймера и т.п.

**Counter** — это монотонно растущее значение: его можно только увеличивать (и сбрасывать при рестарте процесса). Семантика простая: «сколько раз что-то произошло с момента старта». Классические примеры: количество запросов, количество ошибок, количество обработанных сообщений. В Prometheus это `counter`, в Micrometer — `Counter`. Важно держать в голове, что «уменьшить» counter нельзя; если ты делаешь `set(value)`, то это уже не counter и лучше использовать другой тип.

Counter удобен тем, что операции типа **скорости** (RPS, количество ошибок в секунду) строятся на его основе: TSDB берёт разность значений на двух точках и делит на время. Поэтому в коде мы просто вызываем `increment()`, а любую аналитику по скоростям строит уже система мониторинга. Счётчики также хорошо участвуют в SLI: доля ошибок — это отношение двух counters, посчитанное на стороне Prometheus/Grafana.

**Gauge** — это значение, которое может свободно увеличиваться и уменьшаться во времени. Он показывает «состояние сейчас»: размер очереди, количество активных потоков, размер heap’а, длина лагов. В отличие от counter, gauge мы не «инкрементим», а считываем из какого-то объекта или функции. Основная опасность gauge’ев — это неправильный жизненный цикл объекта: если отдать в Micrometer ссылку на объект, который потом будет собираться GC, можно получить странные значения или вообще `NaN`. Поэтому лучше регистрировать gauge над чем-то долговечным: пул потоков, очередь, кэш.

**Timer** — это обёртка для измерения длительности операций. Технически он сочетает в себе counter (количество событий) и summary/histogram по времени. Вызывая `timer.record(Runnable)` или `timer.record(Supplier)`, мы получаем среднее время, распределения и возможность строить p95/p99. Для latency HTTP, работы с БД, выполнения бизнес-операций Timer — базовый инструмент. В микросервисах большинство SLO по времени строится именно на основе таймеров.

**DistributionSummary** очень похож на Timer, но работает не с временем, а с произвольными **размерами**: количество байт в ответе, сумма денег в транзакции, количество элементов в коллекции. Мы записываем не `duration`, а `long`/`double`, и затем можем считать средние, суммы, percentiles. Это удобно для бизнес-метрик: типичный кейс — распределение суммы заказов, размеров payload’ов, количества изменённых записей в batch-операции.

Термин **Histogram** в Micrometer и Prometheus описывает способ представления распределения: значения раскладываются по «корзинам» (buckets). В Micrometer Histogram включается у Timer и DistributionSummary через настройки: `publishPercentileHistogram()` или `serviceLevelObjectives(...)`. В хранилище это превращается в несколько time series: `*_bucket{le="0.1"}`, `*_bucket{le="0.5"}` и т.д. Это даёт возможность строить p95/p99 и SLO-проверки **на стороне Prometheus**, а не считать percentiles в приложении.

Ошибки в выборе типа достаточно типичны. Например, люди заводят gauge `total_requests` и сами прибавляют к нему 1 — в результате из-за рестартов и разных инстансов получить нормальную скорость почти невозможно. Или используют timer только как «стопвотч» и не включают гистограмму — тогда получаются только средние, а хвосты (p95/p99) остаются невидимыми. Или пишут business-метрики в gauge, хотя по смыслу это распределение (например, размер корзины).

Тип метрики важно подбирать исходя из **вопроса, на который ты хочешь отвечать**. «Сколько всего было» — counter, «сколько сейчас» — gauge, «сколько длилось» — timer, «какой размер/количество» — DistributionSummary. Histogram — не отдельный тип в коде, а способ хранить и использовать Timer/DistributionSummary. Хорошая практика — перед добавлением метрики сформулировать вопрос вроде «как будет выглядеть график в Grafana» и уже под него выбрать подходящий тип.

В Spring Boot с Micrometer всё это работает через `MeterRegistry`. Ты либо используешь готовые метрики (Actuator уже считает HTTP, JVM, JDBC и т.п.), либо регистрируешь свои вручную через `Counter.builder`, `Gauge.builder`, `Timer.builder`, `DistributionSummary.builder`. Главное — не плодить метрики «на всякий случай»: лучше иметь 10 хорошо продуманных, чем 50 случайных.

Покажем пример сервиса, в котором используются все пять типов: счётчик запросов, gauge размеров очереди, таймер обработки, DistributionSummary по сумме заказа и таймер с включённой гистограммой. Это хороший базовый шаблон для проектирования метрик под бизнес-операции.

**Gradle: зависимости для работы с Micrometer в Spring Boot**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-core'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-core")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — использование Counter/Gauge/Timer/DistributionSummary/Histogram**

```java
package com.example.observability.metrics.types;

import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;

@Service
public class OrderMetricsService {

    private final Counter ordersCounter;
    private final Counter failedOrdersCounter;
    private final Queue<String> inMemoryQueue = new ConcurrentLinkedQueue<>();
    private final Timer processingTimer;
    private final DistributionSummary amountSummary;

    public OrderMetricsService(MeterRegistry meterRegistry) {
        this.ordersCounter = Counter.builder("orders_total")
                .description("Количество обработанных заказов")
                .tag("result", "success")
                .register(meterRegistry);

        this.failedOrdersCounter = Counter.builder("orders_total")
                .description("Количество неуспешно обработанных заказов")
                .tag("result", "failed")
                .register(meterRegistry);

        Gauge.builder("orders_queue_size", inMemoryQueue, Queue::size)
                .description("Размер очереди заказов в памяти")
                .register(meterRegistry);

        this.processingTimer = Timer.builder("order_processing_duration")
                .description("Время обработки заказа")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(100),
                        Duration.ofMillis(300),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1)
                )
                .maximumExpectedValue(Duration.ofSeconds(5))
                .register(meterRegistry);

        this.amountSummary = DistributionSummary.builder("order_amount_distribution")
                .description("Распределение сумм заказов")
                .baseUnit("currency_unit")
                .publishPercentileHistogram()
                .serviceLevelObjectives(500.0, 1000.0, 5000.0)
                .register(meterRegistry);
    }

    public void enqueueOrder(String orderId) {
        inMemoryQueue.add(orderId);
    }

    public void processOrder(String orderId, double amount) {
        processingTimer.record(() -> {
            try {
                simulateWork();
                amountSummary.record(amount);
                ordersCounter.increment();
            } catch (Exception ex) {
                failedOrdersCounter.increment();
            } finally {
                inMemoryQueue.remove(orderId);
            }
        });
    }

    private void simulateWork() {
        try {
            Thread.sleep(150L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — аналогичный пример**

```kotlin
package com.example.observability.metrics.types

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.DistributionSummary
import io.micrometer.core.instrument.Gauge
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service
import java.time.Duration
import java.util.Queue
import java.util.concurrent.ConcurrentLinkedQueue

@Service
class OrderMetricsService(
    meterRegistry: MeterRegistry
) {

    private val ordersCounter: Counter = Counter.builder("orders_total")
        .description("Количество обработанных заказов")
        .tag("result", "success")
        .register(meterRegistry)

    private val failedOrdersCounter: Counter = Counter.builder("orders_total")
        .description("Количество неуспешно обработанных заказов")
        .tag("result", "failed")
        .register(meterRegistry)

    private val inMemoryQueue: Queue<String> = ConcurrentLinkedQueue()

    private val processingTimer: Timer = Timer.builder("order_processing_duration")
        .description("Время обработки заказа")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(100),
            Duration.ofMillis(300),
            Duration.ofMillis(500),
            Duration.ofSeconds(1)
        )
        .maximumExpectedValue(Duration.ofSeconds(5))
        .register(meterRegistry)

    private val amountSummary: DistributionSummary =
        DistributionSummary.builder("order_amount_distribution")
            .description("Распределение сумм заказов")
            .baseUnit("currency_unit")
            .publishPercentileHistogram()
            .serviceLevelObjectives(500.0, 1000.0, 5000.0)
            .register(meterRegistry)

    fun enqueueOrder(orderId: String) {
        inMemoryQueue.add(orderId)
    }

    fun processOrder(orderId: String, amount: Double) {
        processingTimer.record {
            try {
                simulateWork()
                amountSummary.record(amount)
                ordersCounter.increment()
            } catch (ex: Exception) {
                failedOrdersCounter.increment()
            } finally {
                inMemoryQueue.remove(orderId)
            }
        }
    }

    private fun simulateWork() {
        try {
            Thread.sleep(150L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

Этот пример показывает, как разные типы метрик логично распределяются по смыслу: счётчики для успешных/ошибочных заказов, gauge для очереди, timer для latency, DistributionSummary с гистограммой для сумм.

---

## Единицы и имена

После выбора типа встаёт вопрос, который часто недооценивают: **как назвать метрику и в каких единицах её измерять**. Неправильные имена и единицы приводят к путанице на дашбордах: кто-то думает, что это миллисекунды, кто-то — что секунды; кто-то ожидает байты, а получает мегабайты. В итоге SLO на 500 мс сравнивают с метрикой в секундах и удивляются, почему всё «ломается». Поэтому имя, описание и единица — это часть публичного API твоего сервиса.

В Micrometer у каждой метрики есть **name, description и base unit**. Name — это машинно-читаемый идентификатор (`order_processing_duration`), description — человекочитаемое объяснение («Время обработки заказа»), base unit — физическая единица, с которой правильно работать (seconds, bytes, requests, currency_unit и т.п.). Хорошая практика — всегда заполнять description и unit, особенно для бизнес-метрик; иначе через полгода ты сам забудешь, что именно измерял.

Для таймеров и latency в Java-мире де-факто стандарт — **секунды**. Даже если в коде ты работаешь с миллисекундами, на уровне экспозиции в Prometheus метрика будет в секундах (`_seconds`). Actuator так и делает: `http_server_requests_seconds`. Для разработчика это означает: если ты создаёшь свой Timer, то либо используешь секунды, либо явным образом указываешь base unit и не забываешь про него в дашбордах.

Для размеров данных стандарт — **байты**. Любые значения по памяти, размеру payload’ов, размерам файлов — в байтах; мегабайты и гигабайты — уже производные, которые лучше считать на стороне Grafana. Это упрощает агрегации и снижает шанс, что разные команды будут использовать разные нормы (одни — мегабайты, другие — байты). То же самое касается количества запросов: unit «requests» или «events» позволяет потом легко складывать метрики разных сервисов.

Имена метрик — это отдельная договорённость. В Prometheus традиционно используются **snake_case**, префиксы по доменам и типам (`http_server_requests_seconds`, `jvm_memory_used_bytes`). Для своих метрик стоит придерживаться похожих конвенций: начинать с домена (`order_`, `payment_`, `kafka_consumer_`), использовать понятные глаголы (`count`, `duration`, `size`), избегать слишком общих названий вроде `metric1`. Это кажется очевидным, но в реальных проектах полно `custom_timer` и `my_counter`, которые никто не понимает.

Название метрики и набор тегов должны образовывать **стабильный контракт**. Если ты сегодня называешь latency `order_processing_duration`, а завтра переименуешь его в `orders_processing_ms`, все существующие дашборды и алёрты сломаются. Поэтому любые изменения имен и единиц нужно планировать как миграции: завести новую метрику, задеприкейтить старую, постепенно переехать на новую.

Для бизнес-метрик единицы могут быть нетривиальными: «рубли», «штуки», «сессии». Micrometer не ограничивает тебя, но стоит использовать разумные названия unit (`rub`, `items`, `sessions`). Главное — чтобы это было **последовательно** внутри организации: если один сервис использует `rub`, другой — `RUR`, третий — `RUB`, жизнь SRE сильно осложняется.

Отдельная тема — **агрегированность имен**. Иногда полезно закладывать в имя то, что могло бы быть tag’ом, если ты хочешь избежать высоких кардинальностей. Например, `kafka_consumer_lag_records` и `kafka_consumer_lag_seconds` вместо одной метрики с тегом `unit`. Это компромисс между гибкостью и простотой: чем больше тегов и вариантов, тем выше кардинальность; иногда проще иметь две метрики с разными именами.

И ещё один момент: **consistent suffixes**. Для таймеров — `_seconds`, для счётчиков — `_total` или `_count`, для sizes — `_bytes`. Это помогает даже тем, кто не знает контекста, по одному имени понять, что он видит на графике. Spring Boot сам следует этим соглашениям, и если ты будешь делать то же самое, твои метрики органично впишутся в существующий набор.

Для закрепления покажем пример сервиса, который аккуратно выбирает имена и единицы: таймер в секундах, DistributionSummary в «рублях», counter с суффиксом `_total`, gauge в байтах. Это простые детали, но они сильно влияют на читаемость дашбордов.

**Java — пример аккуратного выбора имён и единиц**

```java
package com.example.observability.metrics.naming;

import io.micrometer.core.instrument.DistributionSummary;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class PaymentMetricsService {

    private final Timer paymentDuration;
    private final DistributionSummary paymentAmount;

    public PaymentMetricsService(MeterRegistry meterRegistry) {
        this.paymentDuration = Timer.builder("payment_request_duration_seconds")
                .description("Время обработки платежного запроса")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(200),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1)
                )
                .register(meterRegistry);

        this.paymentAmount = DistributionSummary.builder("payment_amount_rub")
                .description("Сумма платежа в рублях")
                .baseUnit("rub")
                .publishPercentileHistogram()
                .register(meterRegistry);
    }

    public void pay(String userId, double amountRub) {
        paymentDuration.record(() -> {
            simulateRemoteCall();
            paymentAmount.record(amountRub);
        });
    }

    private void simulateRemoteCall() {
        try {
            Thread.sleep(120L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.metrics.naming

import io.micrometer.core.instrument.DistributionSummary
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class PaymentMetricsService(
    meterRegistry: MeterRegistry
) {

    private val paymentDuration: Timer = Timer.builder("payment_request_duration_seconds")
        .description("Время обработки платежного запроса")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(200),
            Duration.ofMillis(500),
            Duration.ofSeconds(1)
        )
        .register(meterRegistry)

    private val paymentAmount: DistributionSummary =
        DistributionSummary.builder("payment_amount_rub")
            .description("Сумма платежа в рублях")
            .baseUnit("rub")
            .publishPercentileHistogram()
            .register(meterRegistry)

    fun pay(userId: String, amountRub: Double) {
        paymentDuration.record {
            simulateRemoteCall()
            paymentAmount.record(amountRub)
        }
    }

    private fun simulateRemoteCall() {
        try {
            Thread.sleep(120L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

В этом примере из имён сразу ясно: мы измеряем duration в секундах, суммы — в рублях, причём в виде распределения (а не только sums/avg). Такой стиль именования делает дашборды самодокументируемыми.

---

## Кардинальность: контроль label’ов

Кардинальность — это количество **уникальных комбинаций label’ов** (тегов) для метрики. Это скрытая «мина» почти в каждом проекте: метрика сама по себе выглядит невинно, но кто-то решает добавить в теги userId или orderId, и внезапно число time series вырастает на четыре порядка, Prometheus начинает кушать гигабайты памяти, а запросы к нему тормозят. Поэтому контроль кардинальности — один из ключевых аспектов дизайна метрик.

Каждый label — это дополнительное измерение. Если у тебя есть label `status` с 5 значениями и `region` с 3 значениями, то в худшем случае это 15 time series. Добавь `method` с 4 значениями — уже 60. Теперь добавь `userId` (миллионы значений) — и ты только что превратил небольшую метрику в миллионы рядов. Monitoring-система перестаёт работать как мониторинг и превращается в storage для телеметрии, который уже никто не может эффективно читать.

Правило номер один: **никаких high-cardinality идентификаторов в тегах** метрик. Никаких `userId`, `sessionId`, `requestId`, `orderId`, `email`, `phone`, `cardNumber`. Всё это место в логах и трейcах, но не в labels. Иногда кажется, что «ну это же удобно — посмотреть latency по конкретному пользователю», но это бизнес-задача для аналитики, а не техническая метрика, которая собирается каждую секунду со всех инстансов.

Правило номер два: **контролируй количество различных значений даже для low-cardinality тегов**. Например, тег `endpoint` должен содержать паттерны (`/api/orders/{id}`), а не конкретные пути (`/api/orders/123`). Для этого Spring уже даёт `uri` как шаблон, а не raw path. То же самое для `exception`: если начать писать в тег `exception` полный FQCN всех ошибок, легко получить сотни вариантов. Иногда достаточно пометить «business_error» vs «technical_error» или держать отдельную метрику по типам ошибок.

Полезная практика — думать о tag’ах как о **размерностях аналитики**, которые реально будут использоваться. Если никто никогда не смотрит на разбивку по `locale` или `platform`, не надо тащить эти теги в метрики и платить за них памятью. Лучше начать с 2–3 базовых тегов: `status`, `outcome`, `endpoint`, `instance`, а всё остальное добавлять только после того, как стало понятно, что это действительно нужно.

Особое внимание — к метрикам, которые разметены динамическими значениями: кэши по ключам, лейблы хранилищ, tenant-id в мультиарендных системах. Здесь либо нужно **жёстко ограничивать список значений** (например, заранее известный список tenant’ов), либо не тащить это в метрики вообще, а использовать для этого бизнес-логирование и аналитические системы.

В Micrometer есть разграничение на **lowCardinalityKeyValue** и **highCardinalityKeyValue** в Observation API. Low-cardinality попадает в метрики, high-cardinality — только в трейсы. Это встроенный механизм защиты: всё, что может породить огромную кардинальность, должно идти через high-cardinality, иначе Prometheus просто сломается. Важно, чтобы разработчики дисциплинированно его использовали, а не пихали всё в low-cardinality ради «красоты».

Ещё один аспект — **временная кардинальность**. Иногда значение label’а меняется от релиза к релизу: новая версия `app_version`, новый region, новые типы событий. Если не чистить старые значения, метрика обрастает «хвостом» из неиспользуемых комбинаций. В кластерах с auto-scaling’ом это может приводить к очень большому количеству time series и росту нагрузки на Prometheus. Поэтому стоит периодически пересматривать метрики на предмет устаревших тегов и их значений.

Хорошая практическая стратегия — вводить для новых метрик **ограничения и проверки**: Prometheus имеет лимиты на число series на target, а в коде можно сделать примитивную защиту, не логируя и не тегируя подозрительно большие значения (например, если длина строки > N или не совпадает с whitelisted-шаблоном). Это не серебряная пуля, но помогает избежать самых грубых ошибок.

Покажем пример: сервис, который обрабатывает сообщения, и мы хотим иметь метрики по статусу и типу сообщения, но не хотим случайно добавить в теги messageId. Для этого используем только ограниченный набор тегов, а идентификатор сообщения отправляем в high-cardinality контекст трейса и логов.

**Java — контроль кардинальности в метриках обработки сообщений**

```java
package com.example.observability.metrics.cardinality;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.stereotype.Service;

@Service
public class MessageProcessor {

    private final MeterRegistry meterRegistry;
    private final ObservationRegistry observationRegistry;

    public MessageProcessor(MeterRegistry meterRegistry,
                            ObservationRegistry observationRegistry) {
        this.meterRegistry = meterRegistry;
        this.observationRegistry = observationRegistry;
    }

    public void process(String messageId, String type) {
        Observation.createNotStarted("message.process", observationRegistry)
                .lowCardinalityKeyValue("type", normalizeType(type))
                .lowCardinalityKeyValue("component", "message-processor")
                .highCardinalityKeyValue("message.id", messageId)
                .observe(() -> {
                    try {
                        doWork(type);
                        successCounter(type).increment();
                    } catch (Exception ex) {
                        errorCounter(type).increment();
                    }
                });
    }

    private void doWork(String type) {
        try {
            Thread.sleep(50L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }

    private String normalizeType(String type) {
        return switch (type) {
            case "ORDER_CREATED", "ORDER_UPDATED" -> "order";
            case "PAYMENT_CREATED", "PAYMENT_FAILED" -> "payment";
            default -> "other";
        };
    }

    private Counter successCounter(String type) {
        return Counter.builder("messages_processed_total")
                .description("Количество успешно обработанных сообщений")
                .tag("type", normalizeType(type))
                .tag("status", "success")
                .register(meterRegistry);
    }

    private Counter errorCounter(String type) {
        return Counter.builder("messages_processed_total")
                .description("Количество сообщений с ошибкой")
                .tag("type", normalizeType(type))
                .tag("status", "error")
                .register(meterRegistry);
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.metrics.cardinality

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import org.springframework.stereotype.Service

@Service
class MessageProcessor(
    private val meterRegistry: MeterRegistry,
    private val observationRegistry: ObservationRegistry
) {

    fun process(messageId: String, type: String) {
        Observation.createNotStarted("message.process", observationRegistry)
            .lowCardinalityKeyValue("type", normalizeType(type))
            .lowCardinalityKeyValue("component", "message-processor")
            .highCardinalityKeyValue("message.id", messageId)
            .observe {
                try {
                    doWork(type)
                    successCounter(type).increment()
                } catch (ex: Exception) {
                    errorCounter(type).increment()
                }
            }
    }

    private fun doWork(type: String) {
        try {
            Thread.sleep(50L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }

    private fun normalizeType(type: String): String =
        when (type) {
            "ORDER_CREATED", "ORDER_UPDATED" -> "order"
            "PAYMENT_CREATED", "PAYMENT_FAILED" -> "payment"
            else -> "other"
        }

    private fun successCounter(type: String): Counter =
        Counter.builder("messages_processed_total")
            .description("Количество успешно обработанных сообщений")
            .tag("type", normalizeType(type))
            .tag("status", "success")
            .register(meterRegistry)

    private fun errorCounter(type: String): Counter =
        Counter.builder("messages_processed_total")
            .description("Количество сообщений с ошибкой")
            .tag("type", normalizeType(type))
            .tag("status", "error")
            .register(meterRegistry)
}
```

Здесь `message.id` остаётся только в high-cardinality контексте (для трейсов и логов), а в метриках остаются лишь нормализованные типы и статусы, ограниченные небольшим набором значений. Это классический приём контроля кардинальности.

---

## Временные распределения: buckets/percentiles

Latency и любые другие времена — это **распределения**, а не одно число. Среднее значение почти всегда врет: можно иметь 50% запросов по 1 мс, 49% по 10 мс и 1% по 5 секунд, и среднее будет выглядеть «красиво», хотя пользователи будут периодически получать дикие лаги. Поэтому для времени нам нужны либо **percentiles** (p90/p95/p99), либо **гистограммы** с заранее определёнными buckets. На стороне Prometheus это реализуется через `histogram` или `summary`, в Micrometer Timer умеет оба подхода.

Гистограмма разбивает ось времени на «корзины»: «до 10 мс», «до 50 мс», «до 100 мс», «до 500 мс», «до 1 секунды» и т.д. Каждый запрос падает в одну из корзин, и Prometheus хранит кумулятивные счётчики по каждой. Это удобно для **SLO-проверок**: если SLO говорит «95% запросов должны быть быстрее 300 мс», ты можешь смотреть на bucket `le="0.3"` и сравнивать его значение с общим счётчиком. Timer в Micrometer позволяет задать buckets через `serviceLevelObjectives`.

Percentiles можно считать по-разному. В одном варианте (client-side) Micrometer сам считает p95/p99 и отдаёт их как gauge’и; это удобно, но не даёт гибкости в Prometheus и может быть неточно на больших объёмах. В другом варианте (server-side) Prometheus строит percentiles из гистограммы. В большинстве продакшн-сценариев предпочтителен именно второй путь: гистограммы с разумными buckets и вычисление percentiles в запросах Grafana.

Дизайн buckets — это ремесло. Если сделать их слишком широкими, потеряешь детализацию: bucket `le="1.0"` не отличает 100 мс от 900 мс. Если слишком узкими и многочисленными — увеличишь количество time series и нагрузку на Prometheus. Ориентироваться нужно на **ваши реальные SLO и диапазоны**: если SLA 500 мс, имеет смысл иметь buckets до 100, 200, 300, 400, 500, 1000 мс, а далее, возможно, грубее. В Micrometer это можно оформить как `serviceLevelObjectives(Duration.ofMillis(100), ...)`.

Важно помнить, что Timer без включённой гистограммы даёт только count/totalTime/avg. Это не даёт информации о хвостах. Поэтому для любых метрик latency, которые будут использоваться в SLO-алёртах, гистограмму надо включать явно. При этом не обязательно включать percentiles сразу; можно ограничиться bucket’ами и считать percentiles в Prometheus/Grafana по необходимости.

У DistributionSummary история похожая: он тоже умеет гистограммы и SLO-бакеты. Если тебе важно понимать, насколько «жирные» заказы или payload’ы, можно задать SLO для сумм и смотреть, какой процент операций укладывается в «нормальные» диапазоны. Это уже больше про бизнес-аналитику, но инфраструктура та же.

Надо учитывать, что включённые гистограммы увеличивают количество time series. Каждый bucket — отдельный ряд. Если у тебя 10 bucket’ов, 5 тегов и 10 инстансов, получишь довольно внушительный набор. Поэтому гистограммы стоит включать **точечно** для тех метрик, которые реально используются в SLO и алёртах. Для второстепенных операций хватит count/avg, чтобы не плодить лишние ряды.

Покажем пример: Timer с гистограммой и SLO-бакетами для HTTP-запросов к внешнему API. Мы зададим бакеты в миллисекундах и включим `publishPercentileHistogram`, чтобы Prometheus мог строить percentiles. Это типичный шаблон для любого критичного внешнего вызова.

**Java — Timer с гистограммой и SLO-бакетами**

```java
package com.example.observability.metrics.histogram;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class ExternalApiClient {

    private final Timer apiTimer;

    public ExternalApiClient(MeterRegistry meterRegistry) {
        this.apiTimer = Timer.builder("external_api_request_duration_seconds")
                .description("Время ответа внешнего API")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(50),
                        Duration.ofMillis(100),
                        Duration.ofMillis(200),
                        Duration.ofMillis(300),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1),
                        Duration.ofSeconds(2)
                )
                .maximumExpectedValue(Duration.ofSeconds(5))
                .register(meterRegistry);
    }

    public String call(String request) {
        return apiTimer.record(() -> {
            simulateRemoteCall();
            return "ok:" + request;
        });
    }

    private void simulateRemoteCall() {
        try {
            Thread.sleep(120L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.metrics.histogram

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service
import java.time.Duration

@Service
class ExternalApiClient(
    meterRegistry: MeterRegistry
) {

    private val apiTimer: Timer = Timer.builder("external_api_request_duration_seconds")
        .description("Время ответа внешнего API")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(50),
            Duration.ofMillis(100),
            Duration.ofMillis(200),
            Duration.ofMillis(300),
            Duration.ofMillis(500),
            Duration.ofSeconds(1),
            Duration.ofSeconds(2)
        )
        .maximumExpectedValue(Duration.ofSeconds(5))
        .register(meterRegistry)

    fun call(request: String): String {
        return apiTimer.record<String> {
            simulateRemoteCall()
            "ok:$request"
        }
    }

    private fun simulateRemoteCall() {
        try {
            Thread.sleep(120L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

В результате в `/actuator/prometheus` появятся ряды вида `external_api_request_duration_seconds_bucket{le="0.1",...}`, по которым можно построить и percentiles, и SLO-алёрты — например, «если более 5% запросов выходят за 300 мс, поднимай алёрт».

---

## Exemplars: метрика→трейс

Exemplar — это небольшая, но мощная фича, которая связывает **конкретный трейc** с **конкретным bucket’ом метрики**. Идея проста: когда Timer записывает очередное значение в гистограмму, он может вместе с ним сохранить идентификатор спана/трейса (например, `traceId`). В Prometheus это хранится как специальный label, а Grafana умеет показывать на графике точки, кликая по которым можно перейти прямо в трейс, который «представляет» этот bucket.

Зачем это нужно? Метрики дают агрегированную картину: «p95 вырос», «в bucket > 500 мс стало больше запросов». Но они не говорят, **что именно случилось** с конкретным запросом. Трейсы наоборот: по одному запросу можно видеть весь путь, но выбрать интересный запрос среди тысяч сложно. Exemplars решают эту проблему: они позволяют из точки на графике latency перейти к одному репрезентативному трейс-экземпляру, который попал в этот bucket.

В Micrometer и OpenTelemetry поддержка exemplars реализована через **Observation API и tracing-бридж**. Когда у тебя включён OTel javaagent или Micrometer Tracing, observation знает о текущем span’е и может прикрепить к измерению Timer’а ссылку на `traceId`. Экспортер для Prometheus/OTLP сохраняет это в соответствующее поле, а дальше уже дело UI — показать красивую точку и линк «View trace».

С точки зрения дизайна метрик это означает, что для **критичных latency-метрик** (HTTP-эндпоинты, внешние API, БД) полезно включать не только гистограмму, но и exemplars. Тогда при инциденте SRE/разработчик может не только увидеть рост p99, но и быстро открыть трейс конкретного «хвостового» запроса и понять, где он застрял: в БД, во внешнем API, в GC, в синхронном вызове другого сервиса.

При этом важно понимать, что exemplars не увеличивают кардинальность метрик в классическом смысле: они не делают отдельные series, а просто добавляют дополнительные метаданные к существующим bucket’ам. Но всё равно хранение трейсов и работа UI стоит ресурсов, поэтому обычно не включают exemplars на все метрики подряд, а ограничиваются ключевыми таймерами.

На стороне Spring Boot/Java включение exemplars обычно сводится к двум шагам: настроить Micrometer Tracing + OTel (или другой tracer) и убедиться, что **интересующие операции обёрнуты в Observation/Timer**, который знает о текущем span’е. Оставшаяся магия по привязке `traceId` к гистограмме делается библиотеками.

Покажем пример: контроллер, обёрнутый в Observation, с Timer’ом, для которого включены гистограммы и exemplars (подразумевается, что экспортер и backend уже поддерживают exemplars). Код сам по себе не отличается от обычного использования Observation, но его метрики будут богаче.

**Java — использование Observation + Timer для exemplars**

```java
package com.example.observability.metrics.exemplars;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.time.Duration;

@RestController
public class ExemplarController {

    private final ObservationRegistry observationRegistry;
    private final Timer timer;

    public ExemplarController(ObservationRegistry observationRegistry,
                              MeterRegistry meterRegistry) {
        this.observationRegistry = observationRegistry;
        this.timer = Timer.builder("profile_lookup_duration_seconds")
                .description("Время поиска профиля пользователя")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(50),
                        Duration.ofMillis(100),
                        Duration.ofMillis(250),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1)
                )
                .register(meterRegistry);
    }

    @GetMapping("/api/exemplar/profile/{id}")
    public String getProfile(@PathVariable("id") String id) {
        return Observation.createNotStarted("profile.lookup", observationRegistry)
                .lowCardinalityKeyValue("endpoint", "/api/exemplar/profile/{id}")
                .lowCardinalityKeyValue("result", "success")
                .observe(() -> timer.record(() -> {
                    simulateLookup();
                    return "profile:" + id;
                }));
    }

    private void simulateLookup() {
        try {
            Thread.sleep(80L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — аналогичный пример**

```kotlin
package com.example.observability.metrics.exemplars

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RestController
import java.time.Duration

@RestController
class ExemplarController(
    private val observationRegistry: ObservationRegistry,
    meterRegistry: MeterRegistry
) {

    private val timer: Timer = Timer.builder("profile_lookup_duration_seconds")
        .description("Время поиска профиля пользователя")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(50),
            Duration.ofMillis(100),
            Duration.ofMillis(250),
            Duration.ofMillis(500),
            Duration.ofSeconds(1)
        )
        .register(meterRegistry)

    @GetMapping("/api/exemplar/profile/{id}")
    fun getProfile(@PathVariable("id") id: String): String {
        return Observation.createNotStarted("profile.lookup", observationRegistry)
            .lowCardinalityKeyValue("endpoint", "/api/exemplar/profile/{id}")
            .lowCardinalityKeyValue("result", "success")
            .observe<String> {
                timer.record<String> {
                    simulateLookup()
                    "profile:$id"
                }
            }
    }

    private fun simulateLookup() {
        try {
            Thread.sleep(80L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

При наличии настроенного OpenTelemetry и backend’а для трейсов такие измерения будут автоматически снабжаться exemplars: на графике `profile_lookup_duration_seconds_bucket` ты увидишь точки, кликая по которым можно провалиться в конкретный трейс этого запроса. Это резко ускоряет анализ проблем в хвостах латентности.

# 3. Инструментирование в Spring Boot (Micrometer/Actuator)

## Встроенные метрики: HTTP/JVM/DB/Exec/Kafka/Redis

Spring Boot с Actuator и Micrometer из коробки даёт довольно богатый набор встроенных метрик, и это та база, на которую почти всегда стоит опираться, прежде чем городить свои счётчики и таймеры. Ключевая идея: ты подключаешь пару зависимостей, включаешь нужные эндпоинты, и у тебя уже есть HTTP-метрики, метрики JVM, пулов потоков, JDBC, Redis, Kafka и т.д. Это «скелет» наблюдаемости — без него дальше двигаться в сторону SLO и продвинутого трейсинга нет смысла.

HTTP-метрики в Spring Boot строятся поверх фильтра, который оборачивает каждый запрос и пишет таймер `http_server_requests_seconds`. Для каждого URI-паттерна, метода и статуса метрика считает количество запросов, время обработки и ошибки. Это сразу даёт тебе RED/Golden Signals по HTTP-API: rate, errors, duration. При этом Spring нормализует путь до шаблона (`/api/orders/{id}`), чтобы не взорвать кардинальность. Эти метрики автоматически появляются в `/actuator/metrics` и, при наличии Prometheus-регистра, в `/actuator/prometheus`.

JVM-метрики включают в себя состояние heap и non-heap памяти, metaspace, количество потоков, GC-паузы и объём освобождённой памяти, размер и использование различных пулов. Они появляются с префиксом `jvm_...` и позволяют применять USE-модель: utilization, saturation, errors для ресурсов виртуальной машины. Это та часть наблюдаемости, которая помогает понять, почему растут latency или падает throughput: возможно, дело не в БД, а в том, что GC зашёл в stop-the-world.

Метрики БД в Spring Boot чаще всего приходят через интеграцию с пулом соединений HikariCP и библиотекой Micrometer. При использовании `spring-boot-starter-data-jpa` и дефолтного Hikari у тебя автоматически появляются метрики `hikaricp_connections_*`, которые показывают количество активных, свободных и ожидающих соединений, а также скорость выдачи. Это критичный сигнал для понимания, упирается ли сервис в базу физически (нет соединений) или логически (запросы в самой БД тормозят).

Метрики исполнительных пулов (Exec) покрывают thread pool’ы, которые Spring и ты сам создаёшь для `@Async`, `@Scheduled` и других задач. Для `TaskExecutor` можно включить автоинструментирование, но часто имеет смысл регистрировать свои gauge’и, чтобы видеть размеры очередей, количество активных и максимальных потоков. Это напрямую связано с устойчивостью: если пул saturate’ится, ты увидишь рост времени обработки задач и rejected-ошибки, и сможешь вовремя отреагировать.

Kafka-метрики в экосистеме Spring приходят либо напрямую из Kafka-клиента, либо через интеграцию `spring-kafka` и Micrometer. Они дают lag по партициям, скорость потребления/производства, состояние consumer group’ов. Это позволяет наблюдать event-driven часть системы: видишь, не отстают ли consumer’ы, не упирается ли топик в лимиты брокера, не происходит ли «волна» ретраев из-за проблем на стороне обработчиков. Для микросервисов с Kafka это не менее важно, чем HTTP-метрики.

Redis-метрики добавляются через `spring-boot-starter-data-redis` и Micrometer, давая информацию по количеству подключений, latency команд, загруженности, ошибкам. В ситуациях, когда Redis используется как кэш или брокер, это жизненно важно: проблемы с Redis очень быстро превращаются в массовые деградации, но по HTTP-метрикам не всегда сразу понятно, в чём именно дело. Наличие `redis.commands.duration`, `redis.connections.active` и подобных метрик сильно ускоряет диагностику.

Чтобы все эти метрики реально появились, нужно включить Actuator и соответствующие зависимости. Для базового набора достаточно подключить Actuator и Prometheus-регистратор, открыть эндпоинты и, при необходимости, добавить интеграцию с Kafka/Redis. Ниже пример минимальной конфигурации зависимостей и `application.yml`, который включает ключевые метрики и экспонирует их наружу.

**Gradle: зависимости для встроенных метрик**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'

    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.postgresql:postgresql:42.7.3'

    implementation 'org.springframework.kafka:spring-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")

    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.postgresql:postgresql:42.7.3")

    implementation("org.springframework.kafka:spring-kafka")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
}
```

**`application.yml` — включение Actuator и прометеевского эндпоинта**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when_authorized
  metrics:
    tags:
      application: demo-observability
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/demo
    username: demo
    password: demo
  kafka:
    bootstrap-servers: localhost:9092
  data:
    redis:
      host: localhost
      port: 6379
```

**Java — простой контроллер, использующий встроенные HTTP и JVM/DB метрики**

```java
package com.example.observability.builtin;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class BuiltinMetricsController {

    @GetMapping("/api/builtin/ping")
    public String ping() {
        // HTTP-метрики, JVM, пул БД и прочее будут собраны автоматически
        return "pong";
    }
}
```

**Kotlin — аналогичный контроллер**

```kotlin
package com.example.observability.builtin

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class BuiltinMetricsController {

    @GetMapping("/api/builtin/ping")
    fun ping(): String {
        // HTTP-метрики, JVM, пул БД и прочее будут собраны автоматически
        return "pong"
    }
}
```

Этот минимум уже даёт тебе достаточно богатый набор сигналов. В дальнейшем к нему добавляются кастомные метрики и Observation API, но фундамент в виде встроенных HTTP/JVM/DB/Exec/Kafka/Redis-метрик должен присутствовать почти в каждом Spring Boot-сервисе.

---

## Observation API: единый слой + мост в OTel

Observation API в Micrometer — это попытка дать единый программный слой для наблюдаемости, поверх которого уже можно навешивать и метрики, и трейсинг, и логи. Вместо того чтобы в одном месте писать `Timer.record`, в другом — вызывать методы tracer’а, а в третьем — логировать вручную, Observation предоставляет объект `Observation`, который живёт вокруг операции, собирает ключи-значения и «отдаёт» их всем зарегистрированным обработчикам.

Ключевая идея Observation: у тебя есть **ObservationRegistry**, в котором зарегистрированы обработчики (observation handlers) для метрик, трейсинга, логирования и т.п. Когда ты создаёшь Observation через `Observation.createNotStarted(name, observationRegistry)` и вызываешь `observe`, Micrometer создаёт span/метрику, прикрепляет теги и запускает обработчики. Если подключён Micrometer Tracing с OpenTelemetry, этот же Observation будет создавать и завершать OTel-span.

Это даёт одно важное преимущество: ты пишешь наблюдаемость **один раз** и не завязан на конкретный трейсинг-бэкэнд. Сегодня у тебя Prometheus + Jaeger, завтра — Prometheus + Tempo + OTEL collector; Observation-слой при этом не меняется. Всё, что меняется, — это конфигурация бриджей между Observation и конкретными экспортерами.

В Spring Boot 3 Observation API интегрирован очень глубоко. HTTP, JDBC, Kafka, WebClient — всё это уже оборачивается в Observations, и метрики/спаны рождаются автоматически. Для своего кода ты можешь использовать те же паттерны: либо помечать методы аннотацией `@Observed`, либо явно создавать Observation в коде и добавлять low/high-cardinality key values. Это особенно удобно, когда тебе нужно связать бизнес-метрики и трейсинг: `operation=checkout`, `user_segment=vip` и т.п.

Мост в OpenTelemetry реализуется через зависимость `micrometer-tracing-bridge-otel` и конфигурацию трейсера. При наличии javaagent’а OTel или ручной конфигурации exporter’ов все Observation’ы, которые создаёт Micrometer, превращаются в полноценные OTel-spans с traceId/spanId, статусами и атрибутами. Это позволяет в Grafana или другом UI видеть цепочки вызовов, начиная с HTTP и заканчивая БД/внешними API, без дополнительного кода.

Важное свойство Observation — умение работать с кардинальностью ключей. Low-cardinality key values идут в метрики (как labels), high-cardinality — только в спаны. Это встроенный защитный механизм от взрыва кардинальности, о котором мы уже говорили. Правильное использование Observation помогает чётко разделить «что идёт в metrics», а что — в «diagnostics».

Аннотация `@Observed` — это удобный способ быстро обернуть public-метод в Observation. Она позволяет задать имя Observation, дополнительные теги и категорию. Spring сам поднимет аспект/прокси, который вокруг вызова создаст Observation, заполнит стандартные ключи и вызовет handler’ы. Это хороший вариант для бизнес-сервисов и use case-методов, где ты хочешь видеть время выполнения и базовую статистику.

Для более тонкого контроля создают Observation вручную. Это нужно, когда тебе нужно обернуть только часть метода, добавить специфические ключи или контролировать жизненный цикл Observation (например, оставить его открытым на время асинхронной операции и закрыть позже). В этом случае ты работаешь с `Observation.start()` и `stop()`, а не с `observe()`.

Ниже — пример, как подключить Observation + OTel-бридж и как обернуть сервис в `@Observed`. Это показывает, как единый слой Observation превращается и в метрику, и в span, не заставляя тебя думать о деталях экспорта.

**Gradle: Observation + Micrometer Tracing + OTel**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    implementation 'io.micrometer:micrometer-observation'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.40.0'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    implementation("io.micrometer:micrometer-observation")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.40.0")
}
```

**Java — сервис с `@Observed` и ручным Observation**

```java
package com.example.observability.observation;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.annotation.Observed;
import org.springframework.stereotype.Service;

@Service
public class CheckoutObservedService {

    private final ObservationRegistry observationRegistry;

    public CheckoutObservedService(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    @Observed(name = "checkout.operation", contextualName = "checkout-service")
    public void checkout(String userId, long amount) {
        // этот метод автоматически обёрнут в Observation с name=checkout.operation
        simulateWork();
        recordAdditionalObservation(userId, amount);
    }

    private void recordAdditionalObservation(String userId, long amount) {
        Observation.createNotStarted("checkout.amount.observation", observationRegistry)
                .lowCardinalityKeyValue("segment", segmentFor(userId))
                .highCardinalityKeyValue("user.id", userId)
                .lowCardinalityKeyValue("operation", "checkout")
                .observe(() -> simulateWork());
    }

    private String segmentFor(String userId) {
        return userId.startsWith("vip-") ? "vip" : "regular";
    }

    private void simulateWork() {
        try {
            Thread.sleep(80L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.observation

import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import io.micrometer.observation.annotation.Observed
import org.springframework.stereotype.Service

@Service
class CheckoutObservedService(
    private val observationRegistry: ObservationRegistry
) {

    @Observed(name = "checkout.operation", contextualName = "checkout-service")
    fun checkout(userId: String, amount: Long) {
        // этот метод автоматически обёрнут в Observation с name=checkout.operation
        simulateWork()
        recordAdditionalObservation(userId, amount)
    }

    private fun recordAdditionalObservation(userId: String, amount: Long) {
        Observation.createNotStarted("checkout.amount.observation", observationRegistry)
            .lowCardinalityKeyValue("segment", segmentFor(userId))
            .highCardinalityKeyValue("user.id", userId)
            .lowCardinalityKeyValue("operation", "checkout")
            .observe {
                simulateWork()
            }
    }

    private fun segmentFor(userId: String): String =
        if (userId.startsWith("vip-")) "vip" else "regular"

    private fun simulateWork() {
        try {
            Thread.sleep(80L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

С таким подходом ты не задумываешься о том, как именно span попадёт в Jaeger или Tempo: за это отвечает конфигурация OTel-экспортера. а Observation-слой даёт тебе единый API для всей наблюдаемости.

---

## Кастомные метрики: Timer/LongTaskTimer/DistributionSummary

Встроенных метрик для HTTP/JVM/DB часто хватает, чтобы понять «здоровье» сервиса, но для реальной работы почти всегда нужны **кастомные метрики**. Они описывают именно бизнес-операции и специфические технические процессы: время оформления заказа, количество вручную перезапущенных задач, распределение размера batch’ей в ETL-процессах, длительность фонового пересчёта отчётов. Здесь в игру входят `Timer`, `LongTaskTimer` и `DistributionSummary`.

Обычный `Timer` мы уже обсуждали: он измеряет длительность операций и предоставляет count, totalTime, mean, а при включённой гистограмме — SLO-бакеты и percentiles. Его логично использовать для относительно коротких операций (до нескольких секунд), которые происходят часто: HTTP, вызовы внешних API, быстрые бизнес-методы. Именно на этих операциях чаще всего строятся SLO и RED/Golden Signals.

`LongTaskTimer` нужен тогда, когда операция может длиться «долго» и необязательно синхронно: пересчёт отчёта, генерация PDF для большого массива данных, batch-процесс, который крутится минуты и часы. В отличие от обычного Timer, LongTaskTimer позволяет **начать** измерение, затем периодически экспортировать «сколько уже прошло» и **остановить** измерение позже. Метрика показывает количество активных задач и время, прошедшее с начала каждой.

`DistributionSummary` используется для измерения **размеров и количеств**, а не времени. Для бизнес-метрик это часто незаменимо: сумма заказа, количество позиций в корзине, размер выгрузки в CSV, количество строк в batch’е. Распределение этих величин важно не только для бизнес-аналитики, но и для технических решений: если ты видишь, что большинство batch’ей в районе 10–20 тысяч записей, можно оценить, как это влияет на нагрузку на БД.

Хорошая практика — держать кастомные метрики в отдельном «metrics-сервисе» или компоненте, чтобы их можно было переиспользовать и тестировать отдельно. Вместо того чтобы разбрасывать по коду `Timer.builder(...)`, один раз создаёшь таймеры в конструкторе или конфиге и прокидываешь их туда, где нужно. Это помогает не плодить метрики с разными именами и тегами для одной и той же операции.

При проектировании кастомных метрик важно задавать им **смысловые имена и теги**, как мы уже обсуждали. Для таймера «оформление заказа» логично назвать метрику `order_checkout_duration_seconds` и добавить теги `channel` (web/mobile), `result` (success/fail) и, возможно, `currency`, если это критично. Не стоит вешать теги вроде `userId` или `orderId` — это прямой путь к взрыву кардинальности.

Показательный пример — фоновые джобы. Для них можно использовать связку `LongTaskTimer` + обычный `Timer`. LongTaskTimer покажет, сколько задач сейчас активны и как долго они уже идут, а Timer — распределение длительности завершённых задач. Это удобно и для наблюдения (видно, что какая-то джоба «залипла»), и для SLO по времени выполнения.

Рассмотрим пример сервиса, который обновляет рекомендации для пользователя. Для коротких обновлений используем `Timer`, а для «тяжёлых» пересчётов — `LongTaskTimer`. Плюс добавим DistributionSummary, чтобы видеть распределение количества рекомендаций после пересчёта.

**Java — кастомные Timer/LongTaskTimer/DistributionSummary**

```java
package com.example.observability.custom;

import io.micrometer.core.instrument.DistributionSummary;
import io.micrometer.core.instrument.LongTaskTimer;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Random;

@Service
public class RecommendationService {

    private final Timer quickUpdateTimer;
    private final LongTaskTimer fullRebuildTimer;
    private final DistributionSummary recommendationsCountSummary;
    private final Random random = new Random();

    public RecommendationService(MeterRegistry meterRegistry) {
        this.quickUpdateTimer = Timer.builder("recommendation_quick_update_duration_seconds")
                .description("Время быстрого обновления рекомендаций для пользователя")
                .publishPercentileHistogram()
                .maximumExpectedValue(Duration.ofSeconds(2))
                .register(meterRegistry);

        this.fullRebuildTimer = LongTaskTimer.builder("recommendation_full_rebuild_duration_seconds")
                .description("Длительность полного пересчёта рекомендаций")
                .register(meterRegistry);

        this.recommendationsCountSummary = DistributionSummary.builder("recommendations_count")
                .description("Количество рекомендаций после пересчёта")
                .baseUnit("items")
                .publishPercentileHistogram()
                .serviceLevelObjectives(10.0, 50.0, 100.0)
                .register(meterRegistry);
    }

    public void quickUpdate(String userId) {
        quickUpdateTimer.record(this::simulateQuickUpdate);
    }

    public void fullRebuild() {
        LongTaskTimer.Sample sample = fullRebuildTimer.start();
        try {
            int count = simulateFullRebuild();
            recommendationsCountSummary.record(count);
        } finally {
            sample.stop();
        }
    }

    private void simulateQuickUpdate() {
        try {
            Thread.sleep(100L + random.nextInt(150));
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }

    private int simulateFullRebuild() {
        try {
            Thread.sleep(1_000L + random.nextInt(2_000));
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
        return 10 + random.nextInt(100);
    }
}
```

**Kotlin — аналогичный пример**

```kotlin
package com.example.observability.custom

import io.micrometer.core.instrument.DistributionSummary
import io.micrometer.core.instrument.LongTaskTimer
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service
import java.time.Duration
import kotlin.random.Random

@Service
class RecommendationService(
    meterRegistry: MeterRegistry
) {

    private val quickUpdateTimer: Timer = Timer.builder("recommendation_quick_update_duration_seconds")
        .description("Время быстрого обновления рекомендаций для пользователя")
        .publishPercentileHistogram()
        .maximumExpectedValue(Duration.ofSeconds(2))
        .register(meterRegistry)

    private val fullRebuildTimer: LongTaskTimer = LongTaskTimer.builder("recommendation_full_rebuild_duration_seconds")
        .description("Длительность полного пересчёта рекомендаций")
        .register(meterRegistry)

    private val recommendationsCountSummary: DistributionSummary =
        DistributionSummary.builder("recommendations_count")
            .description("Количество рекомендаций после пересчёта")
            .baseUnit("items")
            .publishPercentileHistogram()
            .serviceLevelObjectives(10.0, 50.0, 100.0)
            .register(meterRegistry)

    fun quickUpdate(userId: String) {
        quickUpdateTimer.record {
            simulateQuickUpdate()
        }
    }

    fun fullRebuild() {
        val sample = fullRebuildTimer.start()
        try {
            val count = simulateFullRebuild()
            recommendationsCountSummary.record(count.toDouble())
        } finally {
            sample.stop()
        }
    }

    private fun simulateQuickUpdate() {
        try {
            Thread.sleep(100L + Random.nextInt(150))
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }

    private fun simulateFullRebuild(): Int {
        try {
            Thread.sleep(1_000L + Random.nextInt(2_000))
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
        return 10 + Random.nextInt(100)
    }
}
```

Такой подход даёт тебе полностью наблюдаемый бизнес-процесс: видно, как часто запускается быстрый апдейт, как долго идёт полный пересчёт, сколько рекомендаций результат даёт, и всё это можно привязать к SLO, алёртам и трейсам.

---

## Конфигурация дистрибуций: percentiles/histogram/SLA/exemplars

Micrometer позволяет конфигурировать то, **как именно** хранятся и публикуются распределения для Timer и DistributionSummary. Это важный уровень тонкой настройки: можно включить percentiles, гистограммы, задать SLA-бакеты и exemplars, и всё это как на уровне отдельных метрик, так и глобально через `application.yml`. Если этого не делать, ты получишь только средние значения без хвостов, а это почти всегда недостаточно для SLO.

На уровне кода мы уже видели методы `publishPercentileHistogram()`, `serviceLevelObjectives(...)`, `maximumExpectedValue(...)`. Они управляют тем, какие buckets будут созданы для гистограммы, до каких значений мы вообще считаем метрику и какие SLA-бакеты стоит отслеживать. Это локальная конфигурация: ты настраиваешь конкретный таймер или summary.

На уровне Spring Boot есть глобальный механизм через `management.metrics.distribution`. Он позволяет задать параметры по маске имён метрик. Например, можно сказать: для всех метрик, начинающихся с `http.server.requests`, включить гистограммы по latency и percentiles p95/p99; для метрик `external_api_...` — задать SLA-бакеты и включить exemplars. Это особенно удобно, когда ты не хочешь лазить в каждую точку кода и добавлять настройку вручную.

Конфигурация percentiles через `percentiles` позволяет указать, какие percentiles считать на стороне Micrometer (client-side). Это даёт gauge-метрики типа `metric_name{quantile="0.95"}`, которые удобно рисовать в Grafana, но не очень хорошо подходят для SLO-алёртов в Prometheus, потому что на них сложнее писать алёрты и они менее стабильны на больших объёмах. Поэтому чаще предпочтительнее гистограммы, а percentiles считать на стороне Prometheus.

Секция `sla` в конфигурации distribution позволяет задать SLA-бакеты: значения, которые имеют смысл как границы для SLO. Например, если SLO по HTTP — 95% запросов быстрее 300 мс, можно задать `sla: 0.1, 0.3, 0.5, 1.0` (в секундах), и Prometheus будет иметь отдельные bucket’ы именно по этим порогам. Тогда алёрт «слишком много запросов > 0.3» элементарно пишется через ratio bucket’ов.

Exemplars включаются автоматически при наличии трейсинга и правильной интеграции, но зависят от того, как настроены гистограммы. Без гистограммы просто нечего снабжать exemplar’ами. Поэтому, если ты хочешь из графика latency прыгать в трейсы, нужно и включить гистограммы, и настроить трейсинг (Micrometer Tracing + OTEL).

На глобальном уровне Spring Boot даёт возможность задавать конфигурацию через YAML. Например, можно включить гистограммы и SLA-бакеты для метрик HTTP и внешних API сразу для всех сервисов, просто добавив одну секцию в shared-конфиг. Это удобно, когда в организации много сервисов, и хочется соблюсти единый стандарт.

Покажем пример `application.yml`, который включает percentiles и гистограммы для HTTP и кастомных external API метрик, и Java/Kotlin-конфиг, который добавляет дополнительный `MeterFilter` для тонкой настройки. Это даёт представление, как управлять дистрибуциями на обоих уровнях.

**`application.yml` — глобальная конфигурация дистрибуций**

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        external_api_request_duration_seconds: true
      sla:
        http.server.requests: 0.1, 0.3, 0.5, 1.0
        external_api_request_duration_seconds: 0.05, 0.1, 0.2, 0.5
      percentiles:
        http.server.requests: 0.95, 0.99
      minimum-expected-value:
        http.server.requests: 0.001
      maximum-expected-value:
        http.server.requests: 5.0
```

**Java — кастомный MeterFilter для настройки гистограммы**

```java
package com.example.observability.distribution;

import io.micrometer.core.instrument.config.MeterFilter;
import io.micrometer.core.instrument.distribution.DistributionStatisticConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class MetricsDistributionConfig {

    @Bean
    public MeterFilter externalApiHistogramFilter() {
        return MeterFilter.matchName("external_api_request_duration_seconds",
                (id, config) -> DistributionStatisticConfig.builder()
                        .percentilesHistogram(true)
                        .serviceLevelObjectives(
                                Duration.ofMillis(50).toNanos(),
                                Duration.ofMillis(100).toNanos(),
                                Duration.ofMillis(200).toNanos(),
                                Duration.ofMillis(500).toNanos()
                        )
                        .minimumExpectedValue(Duration.ofMillis(1).toNanos())
                        .maximumExpectedValue(Duration.ofSeconds(5).toNanos())
                        .build()
                        .merge(config));
    }
}
```

**Kotlin — тот же MeterFilter**

```kotlin
package com.example.observability.distribution

import io.micrometer.core.instrument.config.MeterFilter
import io.micrometer.core.instrument.distribution.DistributionStatisticConfig
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class MetricsDistributionConfig {

    @Bean
    fun externalApiHistogramFilter(): MeterFilter {
        return MeterFilter.matchName(
            "external_api_request_duration_seconds"
        ) { _, config ->
            DistributionStatisticConfig.builder()
                .percentilesHistogram(true)
                .serviceLevelObjectives(
                    Duration.ofMillis(50).toNanos(),
                    Duration.ofMillis(100).toNanos(),
                    Duration.ofMillis(200).toNanos(),
                    Duration.ofMillis(500).toNanos()
                )
                .minimumExpectedValue(Duration.ofMillis(1).toNanos())
                .maximumExpectedValue(Duration.ofSeconds(5).toNanos())
                .build()
                .merge(config)
        }
    }
}
```

Такой подход позволяет централизованно управлять тем, какие метрики имеют гистограммы и SLA-бакеты, а какие — нет, не меняя бизнес-код. Это критично для контроля стоимости и качества наблюдаемости.

---

## Экспорт: Prometheus scrape vs OTLP Metrics

Когда метрики уже рождаются в приложении, встаёт вопрос: **куда и как их отправлять**. В мире Spring Boot и Micrometer два основных варианта — Prometheus-style scrape и экспорт по протоколу OTLP Metrics (OpenTelemetry Protocol) в OTEL collector или напрямую в SaaS. Оба подхода живут рядом, и нередко используются одновременно: Prometheus для оперативного мониторинга, OTLP — для централизованной телеметрии и SaaS-платформ.

Prometheus-модель — это **pull/scrape**. Приложение поднимает HTTP-эндпоинт `/actuator/prometheus`, где отдаёт все метрики в текстовом формате. Prometheus сервер периодически скрейпит этот эндпоинт, собирает данные и хранит их в своей TSDB. Преимущество этого подхода — простота и прозрачность: можно легко открыть `/actuator/prometheus` в браузере и увидеть, какие метрики реально экспонируются, а также использовать стандартные инструменты Prometheus и Grafana.

OTLP Metrics — это **push-модель**. Приложение (через Micrometer или OTel SDK) отправляет метрики в OTEL collector по gRPC/HTTP, а collector уже форвардит их дальше: в Prometheus-compatible storage, в SaaS (Datadog, New Relic, Grafana Cloud), в другие системы. Преимущество — гибкость и возможность иметь единый агент/collector, который агрегирует метрики, логирует, трейсит и применяет политики (batching, downsampling, фильтрация).

В Spring Boot для Prometheus достаточно подключить `micrometer-registry-prometheus`, включить Actuator и добавить `management.endpoints.web.exposure.include=prometheus`. Для OTLP Metrics нужен `micrometer-registry-otlp` (или использование OTel SDK напрямую) и конфигурация endpoint’а collector’а. В первом случае Prometheus сам ходит за метриками, во втором — приложение активно пушит их наружу.

С точки зрения эксплуатаций Prometheus-скрейп хорошо подходит для Kubernetes: можно автоматически обнаруживать pod’ы через ServiceMonitor/PodMonitor, навешивать relabeling, контролировать частоту scrape. OTLP больше ориентирован на сценарии, где у тебя уже есть central OTEL collector, к которому стекаются трейсы, логи и метрики. Тогда приложение один раз настраивается на collector, а дальше уже SRE управляют маршрутизацией и хранением.

Важно понимать, что выбор между Prometheus и OTLP не взаимоисключающий. Micrometer поддерживает несколько регистров одновременно: ты можешь подключить PrometheusRegistry и OtlpMeterRegistry, и метрики будут идти и в Prometheus, и в OTEL collector. Это удобно при миграции: можно постепенно переносить дашборды и алёрты, не ломая существующий мониторинг.

Ещё один аспект — безопасность. Prometheus-эндпоинт `/actuator/prometheus` — это HTTP, и его нужно правильно ограничивать: не отдавать наружу в интернет, защищать аутентификацией/сетевыми политиками. OTLP же обычно идёт в приватный collector, но и там нужно заботиться о TLS и авторизации. В любом случае, открытый всем желающим эндпоинт с полным набором метрик — это не лучшая идея.

Ниже — пример Gradle-конфигурации для одновременной поддержки Prometheus и OTLP, `application.yml` с включённым `/actuator/prometheus` и Java/Kotlin-конфиг для регистрации OtlpMeterRegistry. Это типичная схема для сервисов, которые живут в экосистеме Prometheus/Grafana, но при этом отправляют телеметрию в централизованный OTEL стек.

**Gradle: Prometheus + OTLP Metrics**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-registry-otlp'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-registry-otlp")
}
```

**`application.yml` — включение Prometheus и настройка OTLP**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    tags:
      application: demo-observability
    export:
      prometheus:
        enabled: true
      otlp:
        enabled: true
        url: http://otel-collector:4318/v1/metrics
        step: 30s
```

**Java — конфиг Micrometer с Prometheus и OTLP**

```java
package com.example.observability.export;

import io.micrometer.core.instrument.Clock;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.otlp.metrics.OtlpMeterRegistry;
import io.micrometer.otlp.metrics.OtlpMeterRegistryConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsExportConfig {

    @Bean
    public OtlpMeterRegistry otlpMeterRegistry(Clock clock) {
        OtlpMeterRegistryConfig config = new OtlpMeterRegistryConfig() {
            @Override
            public String get(String key) {
                return null; // используем значения из application.yml
            }

            @Override
            public String url() {
                return "http://otel-collector:4318/v1/metrics";
            }
        };
        return OtlpMeterRegistry.builder(config)
                .clock(clock)
                .build();
    }
}
```

**Kotlin — тот же конфиг**

```kotlin
package com.example.observability.export

import io.micrometer.core.instrument.Clock
import io.micrometer.otlp.metrics.OtlpMeterRegistry
import io.micrometer.otlp.metrics.OtlpMeterRegistryConfig
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class MetricsExportConfig {

    @Bean
    fun otlpMeterRegistry(clock: Clock): OtlpMeterRegistry {
        val config = object : OtlpMeterRegistryConfig {
            override fun get(key: String): String? = null // использовать значения по умолчанию

            override fun url(): String = "http://otel-collector:4318/v1/metrics"
        }
        return OtlpMeterRegistry.builder(config)
            .clock(clock)
            .build()
    }
}
```

С такой конфигурацией твой Spring Boot-сервис становится полноценным «гражданином» мира наблюдаемости: его метрики доступны и по `/actuator/prometheus` для Prometheus, и пушатся в OTEL collector для дальнейшей маршрутизации и аналитики. Дальше остаётся только грамотно использовать эти данные в дашбордах и алёртах.

# 4. Прометеевский стек и правила эксплуатации

## Scrape-модель: интервалы, timeouts, relabeling, Service/PodMonitor (Operator)

Prometheus живёт в **pull-модели**: он сам ходит к таргетам и опрашивает их по HTTP. Для Spring Boot-сервиса это, как правило, эндпоинт `/actuator/prometheus`, который отдаёт все текущие метрики в текстовом формате. Важно понимать, что приложение не «пушит» данные в Prometheus, а только отдаёт срез на момент запроса; логику частоты и таймаутов полностью контролирует сам Prometheus. Это влияет на всё остальное: на размер хранилища, на разрешение графиков, на нагрузку на сервисы.

Интервал scrape’а — это фактически **разрешение твоих метрик во времени**. Если ставишь 5 секунд, графики будут гладкими и подробными, но количество точек вырастет в 6 раз по сравнению с 30 секундами. Для типичных прод-сервисов хорошая отправная точка — 15–30 секунд; меньше имеет смысл делать только для действительно высоконагруженных и критичных систем, где важно видеть пики и всплески почти в реальном времени. Внутри одного кластера можно использовать разные интервалы для разных job’ов: важные API — чаще, batch-сервисы — реже.

Таймаут scrape’а задаёт **сколько Prometheus готов ждать ответ** от таргета. Если таймаут равен интервалу (например, оба по 30 секунд), то «залипший» endpoint может заблокировать воркер и вызывать каскадные задержки. Хорошая практика — таймаут в 1/2 или 2/3 от интервала; например, интервал 30с и таймаут 10–15с. Если сервис не успевает ответить за этот срок, это тоже сигнал: значит, либо он перегружен, либо сам эндпоинт `/actuator/prometheus` делает что-то слишком тяжёлое.

Механизм **relabeling** в Prometheus — это «ножницы» для меток таргетов. Перед тем как таргет попадёт в `target_labels`, можно переименовать, удалить или построить новые лейблы на основании существующих. Это нужно, чтобы нормализовать `job`, `instance`, окружение, имена namespace’ов. Например, можно убрать из `instance` порт, а environment вынести в отдельный label. Грамотное relabeling помогает избежать хаоса в именах и облегчает работу с дэшбордами.

Отдельно есть **metric_relabel_configs** — те же «ножницы», но применяемые уже к отдельным метрикам. С их помощью можно дропать ненужные метрики, ограничивать кардинальность по label’ам, переименовывать или агрегировать значения. Это очень мощный инструмент, но его легко использовать неправильно: если сделать сложные regex-операции на каждую метрику, можно серьёзно нагрузить сам Prometheus. Поэтому metric relabel лучше использовать точечно: для отбрасывания «мусорных» метрик или агрессивных label’ов.

В Kubernetes редко прописывают scrape-target’ы вручную; вместо этого используют **Prometheus Operator** и CRD-ресурсы `ServiceMonitor` и `PodMonitor`. `ServiceMonitor` «подписывается» на сервисы с определёнными лейблами и автоматически генерирует для них scrape-конфиги: если у тебя есть `Service` с `app=payments`, достаточно создать `ServiceMonitor`, который смотрит на этот label. Это убирает необходимость держать в голове список endpoint’ов: всё работает через label-селекторы.

`PodMonitor` похож на `ServiceMonitor`, но работает прямо с Pod’ами, обходя `Service`. Он полезен для sidecar-контейнеров или демонов, которые не прикрыты Kubernetes-сервисами. Однако у него есть подводный камень: если вместе использовать ServiceMonitor и PodMonitor, можно случайно начать опрашивать один и тот же процесс дважды, что искажает метрики и увеличивает нагрузку. Поэтому важно чётко определиться, какую модель ты используешь для какой группы таргетов.

Особое внимание — к **безопасности scrape-эндпоинта**. `/actuator/prometheus` нельзя просто так выставлять в интернет: он содержит внутренние имена, метки, иногда — потенциально чувствительные данные (например, имена tenant’ов или внутренних топиков). В Kubernetes это обычно решается тем, что Prometheus сидит в том же кластере и ходит к pod’ам по внутренней сети, а на ингресс `/actuator/**` вообще не выводят. Если по каким-то причинам надо дать доступ извне, его следует закрывать mTLS, basic auth и сетевыми политиками.

Наконец, важно не забыть, что `scrape_interval` и `scrape_timeout` — это параметры **per job**, а не «глобальный тумблер». Сервис с тяжёлым `/actuator/prometheus` можно опрашивать реже, а лёгкое API-приложение — чаще. В совокупности эти настройки определяют, сколько данных попадёт в TSDB, какую нагрузку создаст сам Prometheus и насколько сырой или сглаженной будет картина на дэшбордах.

Чтобы связать всё это с Spring Boot, покажу минимальную конфигурацию приложения, которое отдаёт `/actuator/prometheus`, и пример YAML для `Service` + `ServiceMonitor`. Код на Java/Kotlin будет обычным REST-контроллером; главное здесь — правильная конфигурация Actuator и endpoint’а.

**Gradle: зависимости для Prometheus-экспозиции**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**`application.yml` — включаем `/actuator/prometheus`**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: when_authorized
  metrics:
    tags:
      application: demo-observability
```

**Java — простой контроллер, таргет для scrape**

```java
package com.example.prometheus.scrape;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ScrapeDemoController {

    @GetMapping("/api/scrape-demo/ping")
    public String ping() {
        // HTTP-метрики и JVM-метрики попадут в /actuator/prometheus автоматически
        return "pong";
    }
}
```

**Kotlin — тот же контроллер**

```kotlin
package com.example.prometheus.scrape

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class ScrapeDemoController {

    @GetMapping("/api/scrape-demo/ping")
    fun ping(): String {
        // HTTP-метрики и JVM-метрики попадут в /actuator/prometheus автоматически
        return "pong"
    }
}
```

**Kubernetes `Service` и `ServiceMonitor` (фрагмент, иллюстрация)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-observability
  labels:
    app: demo-observability
spec:
  selector:
    app: demo-observability
  ports:
    - name: http
      port: 8080
      targetPort: 8080

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: demo-observability
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: demo-observability
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s
      scrapeTimeout: 10s
```

Эта связка демонстрирует классический сценарий: Spring Boot отдаёт метрики на `/actuator/prometheus`, сервис помечен лейблом `app=demo-observability`, а `ServiceMonitor` по этому лейблу автоматически подхватывает таргет и задаёт интервал и таймаут scrape’а.

---

## Правила: recording rules и MWMBR-алёрты по SLO

Recording rules в Prometheus — это способ **предварительно посчитать дорогие или часто используемые выражения** и сохранить их как отдельные метрики. Вместо того чтобы каждый раз писать сложную PromQL-формулу в дэшборде или алёрте, ты один раз описываешь правило, которое по расписанию считает это выражение и складывает результат в новую метрику. Это снижает нагрузку на Prometheus, упрощает дэшборды и делает логику SLO более явной.

Типичный пример — SLI по HTTP-успеху: `success_rate = (успешные запросы) / (все запросы)`. Можно каждый раз писать длинную формулу с фильтрами по `status` и `outcome`, но удобнее завести запись `record: http_requests:success_rate:ratio`, которая раз в N секунд считает нужное отношение. Тогда алёрты и графики уже работают с этим pre-computed рядом, а не с сырыми counters.

В SLO-подходе recording rules часто используют для вычисления **error rate и хорошей доли запросов** в скользящих окнах: 1 минута, 5 минут, 30 минут, 1 час. Это помогает сгладить шум и оперировать уже агрегированными величинами. Например, правило может считать `http_requests:errors:rate5m` и `http_requests:total:rate5m`, а следующее правило — `1 - errors/total` как долю успешных за последние 5 минут. Всё это сохранится как отдельные метрики, а SRE будут работать с ними, не вспоминая исходную формулу.

Многооконные алёрты **Multi-Window, Multi-Burn-Rate (MWMBR)** — это приём, предложенный в SRE-подходе Google, чтобы алёрты по SLO были и чувствительными, и не слишком шумными. Идея в том, что ты смотришь на расход error budget в нескольких окнах и с несколькими коэффициентами «скорости сжигания». Если за последние 5 минут сервис сжигает бюджет в 14 раз быстрее, чем допускает SLO, это повод для быстрого, но агрессивного алёрта; если за последние 1–3 часа расход в 2–3 раза выше нормы, это более мягкий, но всё равно значимый сигнал.

Например, при SLO 99.9% у тебя есть 0.1% error budget. Если в 5-минутном окне error rate = 1.4% (в 14 раз выше допустимого), это «жёсткое» нарушение и повод будить людей. Для 1-часового окна можно использовать более мягкие пороги, чтобы не реагировать на короткие всплески. Суть MWMBR в том, что ты **не смотришь только на текущее «ошибка > X%»**, а учитываешь и длительность нарушения, и скорость расхода бюджета.

На практике MWMBR реализуют через пару recording rules и пару алёрт-правил. Сначала считаются error rate и good rate по нескольким окнам, затем считается burn rate (отношение текущего error rate к допустимому), и уже по burn rate заводятся алёрты для короткого и длинного окон. В результате получается два алёрта: один «срочный» (маленькое окно, большой burn rate), второй «плановый» (большое окно, меньший burn rate). Это сильно снижает количество ложных срабатываний.

Важно, что MWMBR-алёрты завязаны не на «в абсолюте error rate > N», а на **SLO и error budget**. То есть одна и та же архитектура алёртов может применяться к сервисам с SLO 99.0%, 99.9% и 99.99%, но порог burn rate будет одинаковым, а допустимый error rate рассчитывается из SLO. В терминах PromQL это часто выглядит как `error_rate / (1 - SLO) > burn_rate_threshold`.

Recording rules и алёрт-правила должны **именоваться и организовываться консистентно**. Обычный шаблон: для SLI заводишь имена вида `service:httpreq:success_rate:ratio`, для SLO-проверок — `slo:service:availability`, для burn rate — `slo:service:error_budget:burn_rate`. Это может казаться бюрократией, но без этого через полгода никто не будет понимать, какая метрика чем является и как её правильно интерпретировать.

Связь с кодом Spring Boot прямая: recording rules и алёрты зависят от того, **как ты называешь и тегируешь метрики в приложении**. Если завтра ты решишь переименовать `application`-tag или URI-шаблон, можешь легко поломать SLO-алёрты. Поэтому дизайн метрик и дизайн rules должен быть согласован: либо есть общий стандарт имён/tags, либо любые изменения в метриках сопровождаются миграцией правил.

Хорошая практика — держать rules рядом с кодом: в репозитории сервиса или в отдельном монорепо с инфраструктурой, но с понятным versioning. Тогда изменение метрик и изменение правил проходят через pull request, и можно обсудить, не ломает ли это существующие дэшборды и алёрты. «Живые» SLO-алёрты требуют такого же аккуратного отношения, как и код.

Чтобы привязать это к Java/Kotlin, покажу пример: контроллер, который выставляет SLI-метрику для checkout-операции, и по этой метрике можно писать recording rules. Здесь мы создаём счётчики успехов/ошибок и таймер длительности, а в Prometheus затем можно сделать recording rule на success rate и burn rate.

**Gradle: зависимости для Micrometer и Actuator**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — SLI для checkout, основа для recording rules**

```java
package com.example.prometheus.slo;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.Duration;

@RestController
@RequestMapping("/api/checkout")
public class CheckoutController {

    private final Counter successCounter;
    private final Counter errorCounter;
    private final Timer latencyTimer;

    public CheckoutController(MeterRegistry registry) {
        this.successCounter = Counter.builder("checkout_requests_total")
                .description("Запросы checkout, успешные")
                .tag("result", "success")
                .register(registry);

        this.errorCounter = Counter.builder("checkout_requests_total")
                .description("Запросы checkout, ошибки")
                .tag("result", "error")
                .register(registry);

        this.latencyTimer = Timer.builder("checkout_request_duration_seconds")
                .description("Время обработки checkout")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(200),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1)
                )
                .register(registry);
    }

    @PostMapping
    public ResponseEntity<String> checkout(@RequestParam("userId") String userId,
                                           @RequestParam("amount") long amount) {
        return latencyTimer.record(() -> {
            try {
                simulateExternalCalls();
                successCounter.increment();
                return ResponseEntity.ok("OK");
            } catch (Exception ex) {
                errorCounter.increment();
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                        .body("ERROR");
            }
        });
    }

    private void simulateExternalCalls() {
        try {
            Thread.sleep(150L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же SLI для checkout**

```kotlin
package com.example.prometheus.slo

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.time.Duration

@RestController
@RequestMapping("/api/checkout")
class CheckoutController(
    registry: MeterRegistry
) {

    private val successCounter: Counter = Counter.builder("checkout_requests_total")
        .description("Запросы checkout, успешные")
        .tag("result", "success")
        .register(registry)

    private val errorCounter: Counter = Counter.builder("checkout_requests_total")
        .description("Запросы checkout, ошибки")
        .tag("result", "error")
        .register(registry)

    private val latencyTimer: Timer = Timer.builder("checkout_request_duration_seconds")
        .description("Время обработки checkout")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(200),
            Duration.ofMillis(500),
            Duration.ofSeconds(1)
        )
        .register(registry)

    @PostMapping
    fun checkout(
        @RequestParam("userId") userId: String,
        @RequestParam("amount") amount: Long
    ): ResponseEntity<String> {
        return latencyTimer.record<ResponseEntity<String>> {
            try {
                simulateExternalCalls()
                successCounter.increment()
                ResponseEntity.ok("OK")
            } catch (ex: Exception) {
                errorCounter.increment()
                ResponseEntity
                    .status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("ERROR")
            }
        }
    }

    private fun simulateExternalCalls() {
        try {
            Thread.sleep(150L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

На этой метрике легко построить recording rule для error rate и затем MWMBR-алёрты, не трогая приложенческий код.

---

## Масштабирование хранения: Thanos/Mimir/Cortex, remote_write, ретеншн и downsampling

Один Prometheus-сервер — это **single-node TSDB**, у которого есть пределы: по CPU, памяти, диску. Для небольшого кластера и пары сотен таргетов этого достаточно, но по мере роста числа сервисов, метрик и кардинальности приходится думать о масштабировании. Здесь в игру вступают remote_write и системы вроде Thanos, Mimir, Cortex, которые умеют делать из многих Prometheus’ов единый логический источник метрик с долгим хранением.

`remote_write` в Prometheus — это механизм отправки данных из локальной TSDB в удалённое хранилище. Локальный Prometheus продолжает скрейпить цели и хранит у себя данные на ограниченный период (например, 7–14 дней), а параллельно пушит time series в remote backend. Это может быть Thanos Receive, Mimir, Cortex или SaaS. Преимущество — локальный Prometheus остаётся кэшем и «быстрым» источником, а исторические данные уходят в более дешёвое и масштабируемое хранилище.

Системы **Thanos/Mimir/Cortex** решают несколько задач: делают Prometheus-данные **горизонтально масштабируемыми**, обеспечивают **HA-режим** для Prometheus-инстансов и дают **долгосрочное хранение** (месяцы и годы). Они подтягивают данные из локальных Prometheus’ов или remote_write, складывают их в объектное хранилище (S3/GCS и т.п.) и поднимают слой query-агрегации. Для приложения это прозрачно: Grafana ходит не в отдельный Prometheus, а в Thanos/Mimir-endpoint, который под капотом объединяет несколько источников.

Ретеншн (retention) — это **как долго хранить данные** и с каким разрешением. Локальный Prometheus обычно хранит «сырые» данные (каждый scrape) 7–30 дней. В долгосрочном хранилище сырые данные могут быть слишком дорогими: хранить метрику с шагом 15 секунд за полгода — весьма накладно. Поэтому используют **downsampling**: агрегируют значения в более грубые интервалы (1 минута, 5 минут) и для старых периодов хранят только агрегаты. Thanos, например, умеет автоматически downsample’ить блоки по мере старения.

Это приводит к важному компромиссу: чем дольше ты хранишь высокодетализированные метрики, тем дороже система. Для расследования свежих инцидентов обычно достаточно 7–14 дней с детализацией 15–30 секунд; для SLA-отчётов за квартал вполне хватит minute-level или даже 5-minute-level. Поэтому стратегии хранения часто выглядят так: «30 дней сырых данных, 6 месяцев данных с шагом 1 минута, 2 года данных с шагом 5 минут».

С точки зрения приложений и Dev-команд масштабирование хранилища влияет не напрямую, но опосредованно. Во-первых, **шаг экспорта** (`management.metrics.export.*.step`) определяет «минимальное» разрешение метрик, которое имеет смысл хранить: если приложение публикует данные раз в 1 минуту, нет смысла скрейпить его каждые 5 секунд. Во-вторых, количество и кардинальность метрик прямо влияют на объём хранилища; если разработчики заваливают систему тысячами высококардинальных series, инфраструктура будет страдать и требовать дорогих решений.

В dev/test окружениях можно позволить себе более агрессивный ретеншн (3–7 дней, маленький объём), и там чаще всего не нужен Thanos. В prod-кластер, особенно multi-region, почти всегда имеет смысл ставить либо HA-Prometheus с общим remote storage, либо полноценно выстраивать Thanos/Mimir. Важно при этом не забыть **мониторить сам мониторинг**: сколько sample’ов в секунду, какая загрузка TSDB, нет ли backpressure на remote_write.

`remote_write` сам по себе может стать узким местом: если удалённое хранилище недоступно или медлит, Prometheus может начать буферизировать данные в памяти и диске, а затем дропать sample’ы. С точки зрения сервисов это будет означать «дыры» на графиках. Поэтому remote-backend должен быть не менее надёжным, чем сам Prometheus, а на его состояние нужно заводить отдельные алёрты.

Ещё один аспект масштабирования — **мультикластерность**. В большой инфраструктуре обычно несколько Kubernetes-кластеров, и каждый имеет свой Prometheus. Thanos/Mimir решают это, объединяя их данные и позволяя строить дэшборды и SLO на уровне всей системы. Но это требует дисциплины в именовании метрик и label’ов, чтобы потом можно было агрегировать без конфликтов между кластерами.

На уровне Spring Boot то, что ты реально контролируешь, — это **частота экспорта** (для push-регистров), размер и кардинальность метрик. Например, если используешь OTLP-экспорт в collector, можно настроить `step` и таймауты, чтобы не заспамить collector миллионами точек. Также можно использовать `MeterFilter`, чтобы не экспортировать явно ненужные метрики и тем самым экономить место.

Приведу пример: конфигурация Spring Boot, которая задаёт шаг экспорта для Prometheus и OTLP, а также простой конфиг-класс, который программно настраивает step для OTLP-registry. Это примитивный, но показательный кусок про то, как приложение может «подстроиться» под стратегию хранения.

**Gradle: зависимости для Prometheus и OTLP**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-registry-otlp'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-registry-otlp")
}
```

**`application.yml` — шаг экспорта и базовый ретеншн на уровне приложения**

```yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
        step: 15s
      otlp:
        enabled: true
        step: 30s
        url: http://otel-collector:4318/v1/metrics
    tags:
      application: demo-storage-scaling
```

**Java — кастомизация OTLP-registry (пример)**

```java
package com.example.prometheus.storage;

import io.micrometer.core.instrument.Clock;
import io.micrometer.otlp.metrics.OtlpMeterRegistry;
import io.micrometer.otlp.metrics.OtlpMeterRegistryConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class OtlpMetricsConfig {

    @Bean
    public OtlpMeterRegistry otlpMeterRegistry(Clock clock) {
        OtlpMeterRegistryConfig config = new OtlpMeterRegistryConfig() {
            @Override
            public String get(String key) {
                return null; // значения по умолчанию или из application.yml
            }

            @Override
            public String url() {
                return "http://otel-collector:4318/v1/metrics";
            }

            @Override
            public Duration step() {
                return Duration.ofSeconds(30);
            }
        };

        return OtlpMeterRegistry.builder(config)
                .clock(clock)
                .build();
    }
}
```

**Kotlin — тот же конфиг OTLP-registry**

```kotlin
package com.example.prometheus.storage

import io.micrometer.core.instrument.Clock
import io.micrometer.otlp.metrics.OtlpMeterRegistry
import io.micrometer.otlp.metrics.OtlpMeterRegistryConfig
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class OtlpMetricsConfig {

    @Bean
    fun otlpMeterRegistry(clock: Clock): OtlpMeterRegistry {
        val config = object : OtlpMeterRegistryConfig {
            override fun get(key: String): String? = null

            override fun url(): String = "http://otel-collector:4318/v1/metrics"

            override fun step(): Duration = Duration.ofSeconds(30)
        }

        return OtlpMeterRegistry.builder(config)
            .clock(clock)
            .build()
    }
}
```

Такая конфигурация не заменяет Thanos/Mimir/Cortex, но показывает, что приложение может управлять частотой экспорта и тем самым влиять на объём и разрешение данных в хранилище.

---

## Кардинальность под контролем: top-N, explorer, лимиты/квоты

Тема кардинальности уже появлялась, но на уровне Prometheus-стека она приобретает **операционный** характер. Если разработчики не следят за тем, какие label’ы они используют, и если SRE не контролируют рост числа time series, в какой-то момент Prometheus начинает «топиться» в объёмах, а графики и запросы становятся медленными. Поэтому управление кардинальностью — одна из ключевых практик эксплуатации прометеевского стека.

Первое, что нужно уметь делать, — это **диагностировать, кто именно раздувает кардинальность**. В Prometheus есть функции и UI для этого: можно запросить `count by (__name__)({__name__=~".+"})`, можно использовать built-in tools или внешние утилиты, которые показывают топ-метрики по количеству series. В Grafana часто используют «explorer»-дашборды, которые показывают top-N метрик по number of series, а также label’ы, внутри которых больше всего значений.

Следующий шаг — посмотреть не только на имя метрики, но и на **конкретные label-комбинации**. Например, метрика `http_server_requests_seconds` в норме имеет разумную кардинальность, но если кто-то начинает тегировать её по `userId`, количество series взмывает вверх. Аналогично с бизнес-метриками: `orders_total{orderId="..."}` — прямой путь к миллионам series. Такие вещи нужно уметь находить запросами и инструментами кардинальности, а потом возвращаться к команде, которая их внедрила.

На уровне Prometheus есть возможность применять **лимиты и квоты**. Можно ограничить количество time series per scrape, ограничить общий объём метрик, выставить лимиты на HTTP-ответ. Это не удобные инструменты в бою (когда лимит срабатывает, часть метрик начинает дропаться), но они спасают систему от полного падения. Более мягкий вариант — использовать `metric_relabel_configs` для дропа особо вредных label’ов или целых метрик, которые не нужны в проде.

С точки зрения разработки лучше всего **предотвращать проблему на источнике**. Мы уже обсуждали, что в Micrometer можно использовать Observation с разделением low/high cardinality, и что не стоит добавлять в labels идентификаторы пользователей и заказов. Дополнительно есть инструмент `MeterFilter`, который позволяет в приложении отфильтровать метрики по имени или по тегам. Это хорошее место, чтобы отрезать какую-то часть «шумных» метрик до того, как они попадут в Prometheus.

Ещё одна практика — **ограничение возможных значений label’ов**. Например, для `tenantId` допустим небольшой список (10–20 арендаторов); если их становится сотни, нужно пересматривать архитектуру. Для `endpoint` разумно использовать нормализованные URI-шаблоны, а не raw path. Это частично решается самим Spring Boot, но кастомные метрики нужно проектировать с тем же подходом: минимум label’ов, разумный набор значений.

Top-N-дашборды и explorers помогают не только при расследовании проблем, но и как инструмент **code review для наблюдаемости**. Когда команда выкатывает новую версию сервиса, можно посмотреть, как изменилась кардинальность его метрик: появились ли новые series, не стало ли их вдвое больше. Если да — это повод вернуться к коду и пересмотреть метрики до того, как это вырастет в проблему на проде.

Важно понимать, что **кардинальность — это не только метрики приложений**, но и метрики самого Kubernetes/Node Exporter’ов/Sidecar’ов. Иногда именно системные метрики раздувают TSDB: каждый pod имеет новые label’ы, каждый новый deployment добавляет новые series. Поэтому лимиты и анализ кардинальности нужно применять к всему стеку, а не только к бизнес-метрикам.

Теперь свяжем это с кодом Spring Boot. Покажу `MeterFilter`, который дропает метрики по имени и по label’у, если они нарушают наши правила, и демонстрационный сервис, в котором мы специально не даём «опасным» label’ам попасть в метрики. Это не серебряная пуля, но хороший слой «защиты от дурака» на стороне приложения.

**Gradle: зависимости для Micrometer и Actuator (как раньше)**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-core'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-core")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — MeterFilter для контроля кардинальности**

```java
package com.example.prometheus.cardinality;

import io.micrometer.core.instrument.Meter;
import io.micrometer.core.instrument.config.MeterFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CardinalityMeterFilterConfig {

    @Bean
    public MeterFilter denyHighRiskMeters() {
        return new MeterFilter() {
            @Override
            public Meter.Id map(Meter.Id id) {
                // Пример: режем метрики, в имени которых есть "user_id"
                if (id.getName().contains("user_id")) {
                    return null;
                }
                // Пример: отбрасываем метрики, где есть тег orderId (слишком высокая кардинальность)
                boolean hasOrderIdTag = id.getTags().stream()
                        .anyMatch(t -> t.getKey().equals("orderId"));
                if (hasOrderIdTag) {
                    return null;
                }
                return id;
            }
        };
    }
}
```

**Kotlin — тот же MeterFilter**

```kotlin
package com.example.prometheus.cardinality

import io.micrometer.core.instrument.Meter
import io.micrometer.core.instrument.config.MeterFilter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class CardinalityMeterFilterConfig {

    @Bean
    fun denyHighRiskMeters(): MeterFilter {
        return object : MeterFilter {
            override fun map(id: Meter.Id): Meter.Id? {
                if (id.name.contains("user_id")) {
                    return null
                }
                val hasOrderIdTag = id.tags.any { it.key == "orderId" }
                if (hasOrderIdTag) {
                    return null
                }
                return id
            }
        }
    }
}
```

**Java — сервис, который осознанно избегает high-cardinality тегов**

```java
package com.example.prometheus.cardinality;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Service;

@Service
public class SafeMetricsOrderService {

    private final MeterRegistry meterRegistry;

    public SafeMetricsOrderService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void completeOrder(String orderId, String channel) {
        // Вместо orderId используем нормализованный канал и общий счётчик
        Counter counter = Counter.builder("orders_completed_total")
                .description("Завершённые заказы по каналам")
                .tag("channel", normalizeChannel(channel))
                .register(meterRegistry);

        counter.increment();
        // orderId отправляем в логи или трейсы, но не в метрики
    }

    private String normalizeChannel(String channel) {
        return switch (channel.toLowerCase()) {
            case "web", "browser" -> "web";
            case "ios", "android", "mobile" -> "mobile";
            default -> "other";
        };
    }
}
```

**Kotlin — тот же сервис**

```kotlin
package com.example.prometheus.cardinality

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.stereotype.Service

@Service
class SafeMetricsOrderService(
    private val meterRegistry: MeterRegistry
) {

    fun completeOrder(orderId: String, channel: String) {
        val counter: Counter = Counter.builder("orders_completed_total")
            .description("Завершённые заказы по каналам")
            .tag("channel", normalizeChannel(channel))
            .register(meterRegistry)

        counter.increment()
        // orderId отправляем в логи/трейсы, но не в метрики
    }

    private fun normalizeChannel(channel: String): String =
        when (channel.lowercase()) {
            "web", "browser" -> "web"
            "ios", "android", "mobile" -> "mobile"
            else -> "other"
        }
}
```

Такой подход — комбинация технических ограничений (MeterFilter, лимиты в Prometheus) и дисциплины при проектировании метрик — позволяет держать кардинальность под контролем и не превращать мониторинг в ещё одну нестабильную систему.

# 5. Grafana: дашборды, алёрты, практики

## Дашборды «по роли»: SRE/инциденты, разработчик сервиса, бизнес-метрики

Ролевые дашборды — это попытка честно признать, что один и тот же набор метрик нужен разным людям в разном виде. SRE в момент инцидента смотрит на одни графики, разработчик сервиса в рабочий день — на другие, а продукт/бизнесу важны вообще третьи показатели. Если этого не учитывать и делать «один большой дашборд на всех», он быстро превращается в кладбище панелей, которое никто не читает и которым невозможно пользоваться под давлением инцидента.

Инцидентный дашборд для SRE обычно отвечает на вопрос: «Сервис жив или нет, и если нет — где именно болит?». На первом экране — SLO/SLA по ключевым endpoint’ам (успешные запросы, латентность p95/p99), error rate, состояние breaker’ов, нагрузка по CPU/памяти/GC, насыщенность пулов (thread pools, connection pools). В норме SRE вообще не обязан знать внутреннюю бизнес-логику сервиса: ему нужны сигналы уровня «HTTP 5xx растут», «latency скачет», «коннекты к базе выбиты». Чем меньше ему нужно кликать при инциденте — тем лучше.

Дашборд разработчика сервиса, наоборот, может быть глубже и «грязнее». Здесь уже могут появляться бизнес-метрики низкого уровня: count по типам ошибок, распределение исключений, размер batch’ей, retry rate, очередь задач, лаг по Kafka, статистика кэша. Этот дашборд нужен не чтобы «потушить пожар», а чтобы понять, как сервис ведёт себя в разных сценариях, где есть узкие места, где можно улучшить производительность или устойчивость. Здесь допустимы более детализированные панели, но всё равно стоит держать структуру: от общих сигналов — к конкретным.

Бизнес-дашборд — отдельная история. Он опирается на те же технические метрики, но смотрит на них через призму домена: количество оформленных заказов, конверсия, средний чек, доля ошибок по бизнес-коду, время прохождения ключевого бизнес-процесса (например, «от нажатия на кнопку до получения подтверждения по SMS»). Этот дашборд обычно не живёт в одиночестве: он часто комбинируется с сырыми метриками из Data Warehouse, но технические SLI/SLO всё равно остаются основой для оценки здоровья продукта.

Хорошая практика — **не смешивать ролевые сценарии на одном экране**. Лучше иметь три небольших, но понятных дашборда, чем один монстр с 40 панелями. Для SRE — дашборд «Service X — On-call», для разработчика — «Service X — Deep Dive», для бизнеса — «Service X — Product KPIs». При этом панели могут переиспользоваться через библиотечные панели Grafana: один и тот же график latency можно включить в несколько дашбордов с разными окружениями/фильтрами.

Структура дашборда по ролям тоже различается. Для инцидентного — первый ряд панелей отвечает на вопрос «зашёл ли алёрт случайно»: есть ли всплеск error rate, есть ли деградация latency, живы ли зависимости (БД, кэш, внешние API). Ниже — более детальные панели по конкретным зависимостям. Для разработчика полезно начинать с общей картины нагрузки и ошибок, а дальше проваливаться в отдельные компоненты: очередь задач, thread pools, операции БД, внешние вызовы.

Ещё один важный аспект — **мульти-окружения**. Часто хочется одним дашбордом покрыть dev, test, stage, prod. Это соблазнительно, но рискованно: всегда найдётся тот, кто на алёрте будет смотреть на stage вместо prod. Лучше использовать переменные (variables) для выбора `environment`, но при этом сохранять отдельные «преднастроенные» версии дашборда для прод-окружения, где переменная уже жёстко установлена. Так SRE по ссылке из алёрта сразу попадает на нужный контекст.

Сами метки и теги в метриках сильнее всего определяют, насколько удобно строить ролевые дашборды. Если у тебя в каждой метрике есть `application`, `environment`, `region`, `instance`, то SRE может быстро агрегировать данные по нужному уровню: сервис, зона, отдельный pod. Для бизнеса важны другие теги: `channel`, `segment`, `plan` и т.д. Это ещё один аргумент в пользу аккуратного дизайна метрик: задуматься, какие срезы нужны техническим ролям, а какие — продуктовым.

Наконец, ролевой подход к дашбордам хорошо помогает в онбординге. Новому разработчику проще объяснить: «Вот дашборд разработчика, вот где смотреть латентность, вот где кэш, вот Kafka»; SRE — показать: «Вот дашборд для on-call, первая строка — твой лучший друг», продукту — «Вот сколько денег/заказов/конверсии». Если же в Grafana chaos и 50 дашбордов без единого стандарта, любой инцидент превращается в поиск «той самой панели, которую показывали два месяца назад».

Для того чтобы такие дашборды вообще было из чего собирать, приложение должно отдавать хорошо структурированные метрики. Ниже простой пример Java/Kotlin-кода: сервис генерирует несколько бизнес- и техметрик, которые можно использовать на разных ролевых дашбордах — число заказов, число ошибок, latency по ключевой операции.

**Java — метрики для разных ролевых дашбордов**

```java
package com.example.grafana.roles;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Random;

@Service
public class OrderMetricsService {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTimer;
    private final Random random = new Random();

    public OrderMetricsService(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders_created_total")
                .description("Количество успешно созданных заказов (бизнес-метрика)")
                .tag("channel", "web")
                .register(registry);

        this.ordersFailed = Counter.builder("orders_failed_total")
                .description("Количество неуспешных попыток создания заказа (для SRE/разработчиков)")
                .tag("channel", "web")
                .register(registry);

        this.orderProcessingTimer = Timer.builder("order_processing_duration_seconds")
                .description("Время обработки заказа от приёма до подтверждения")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(200),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1)
                )
                .register(registry);
    }

    public void processOrder(String userId, long amount) {
        orderProcessingTimer.record(() -> {
            try {
                // Имитация обработки
                Thread.sleep(100L + random.nextInt(200));
                if (random.nextDouble() < 0.05) {
                    ordersFailed.increment();
                    throw new IllegalStateException("Payment gateway error");
                }
                ordersCreated.increment();
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        });
    }
}
```

**Kotlin — те же метрики**

```kotlin
package com.example.grafana.roles

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.stereotype.Service
import java.time.Duration
import kotlin.random.Random

@Service
class OrderMetricsService(
    registry: MeterRegistry
) {

    private val ordersCreated: Counter = Counter.builder("orders_created_total")
        .description("Количество успешно созданных заказов (бизнес-метрика)")
        .tag("channel", "web")
        .register(registry)

    private val ordersFailed: Counter = Counter.builder("orders_failed_total")
        .description("Количество неуспешных попыток создания заказа (для SRE/разработчиков)")
        .tag("channel", "web")
        .register(registry)

    private val orderProcessingTimer: Timer = Timer.builder("order_processing_duration_seconds")
        .description("Время обработки заказа от приёма до подтверждения")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(200),
            Duration.ofMillis(500),
            Duration.ofSeconds(1)
        )
        .register(registry)

    private val random = Random.Default

    fun processOrder(userId: String, amount: Long) {
        orderProcessingTimer.record {
            try {
                Thread.sleep(100L + random.nextInt(200))
                if (random.nextDouble() < 0.05) {
                    ordersFailed.increment()
                    throw IllegalStateException("Payment gateway error")
                }
                ordersCreated.increment()
            } catch (ex: InterruptedException) {
                Thread.currentThread().interrupt()
            }
        }
    }
}
```

Эти метрики затем легко превращаются в панели: бизнес смотрит на `orders_created_total` с группировкой по `channel`, SRE — на отношение `orders_failed_total / (orders_created_total + orders_failed_total)` и латентность `order_processing_duration_seconds`.

---

## Алёртинг: SLO-правила, тишина vs paging, dead-man switch

Алёрты — это не просто «ещё одни графики с красными линиями», а механика, которая определяет, когда и кого будить. Основная идея: страницы (paging) должны приходить только тогда, когда нарушен пользовательский опыт или вот-вот будет нарушен; всё остальное либо идёт в тикеты (non-paging alert), либо остаётся на уровне дэшбордов и логов. Грубая ошибка — пытаться повесить алёрт на каждую метрику и каждую ошибку: тогда on-call просто перестаёт на них реагировать.

SLO-алёртинг начинается с формализации: «Какой процент запросов должен быть успешным и за какое время?». Допустим, SLO: 99.9% запросов корзины должны завершаться успешным ответом быстрее 500 мс за 30-дневное окно. Из этого SLO выводится error budget (0.1% ошибок) и допустимый error rate. Алёрты строятся на **расходе error budget’а**: если в течение последних 10 минут ошибка растёт так, что за день ты выжжешь весь бюджет — это повод для срочного paging; если за последние 3 часа расход удвоился, но остаётся терпимым — повод для тикета и плановой реакции.

Тишина vs paging — это про правильную грануляцию сигналов. Большинство алёртов должно быть **«тихими»**: они попадают в Slack/issue tracker и служат индикаторами деградации, но не будят людей ночью. Paging-алёрты должны быть привязаны к SLO и бизнес-критичным путям: недоступность главного API, массовые 5xx, падение throughput’а на платёжном сервисе. Если всё подряд бьёт по PagerDuty, on-call через неделю перестанет замечать даже важные сигналы.

Dead-man switch — это техника, которая гарантирует, что алёрт сработает, если **мониторинг перестал работать**. Это какой-нибудь периодический heartbeat-метрик или алёрт вроде «за последние 5 минут не пришло ни одного sample от этого job’а». Пока Prometheus работает и видит данные — алёрт молчит. Как только что-то ломается (сам Prometheus, сеть, exporter) — алёрт срабатывает и сигнализирует, что мониторинг уже не надёжный. Это особенно важно для инфраструктурных компонент.

Графана (особенно в новом unified alerting) часто выступает как единая точка для алёртов, даже если данные приходят из разных источников. Здесь важно **синхронизировать названия и описания алёртов** с SLO и runbook’ами. На панелях, где выводится SLO-график, удобно сразу показывать текущий статус SLO (выполняется/нет) и текущий burn rate; алёрт должен ссылаться на этот же график и на конкретный runbook.

Ещё один часто упускаемый момент — **тестирование алёртов**. Alert rule нужно проверять так же, как и код: через staging, через time-shift в PromQL, через искусственные нагрузки. Простой способ — поднять временно значение error rate выше порога и убедиться, что алёрт приходит, что в нём корректные лейблы/ссылки и он понятен on-call-у. Без этого легко получить алёрты, которые либо никогда не стреляют, либо стреляют не по делу.

На уровне приложения ключевая задача — **давать хорошие метрики для SLO**: чёткие success/error-счётчики и таймеры по критичным операциям. Дальше SRE уже на стороне Prometheus/Grafana пишет правила. Ниже — пример Java/Kotlin-кода, который формирует метрики для SLO по endpoint’у checkout: успехи, ошибки, latency. Это «сырьё» для SLO-алёртов и MWMBR-подхода.

**Java — метрики для SLO-алёртов по checkout**

```java
package com.example.grafana.alerts;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.Duration;

@RestController
@RequestMapping("/api/alerts/checkout")
public class CheckoutSloController {

    private final Counter successCounter;
    private final Counter errorCounter;
    private final Timer latencyTimer;

    public CheckoutSloController(MeterRegistry registry) {
        this.successCounter = Counter.builder("checkout_requests_total")
                .description("Запросы checkout, успешные")
                .tag("result", "success")
                .register(registry);

        this.errorCounter = Counter.builder("checkout_requests_total")
                .description("Запросы checkout, ошибки")
                .tag("result", "error")
                .register(registry);

        this.latencyTimer = Timer.builder("checkout_request_duration_seconds")
                .description("Время обработки checkout-запроса")
                .publishPercentileHistogram()
                .serviceLevelObjectives(
                        Duration.ofMillis(100),
                        Duration.ofMillis(300),
                        Duration.ofMillis(500),
                        Duration.ofSeconds(1)
                )
                .register(registry);
    }

    @PostMapping
    public ResponseEntity<String> checkout(@RequestParam("userId") String userId,
                                           @RequestParam("amount") long amount) {
        return latencyTimer.record(() -> {
            try {
                simulateExternalDependency();
                successCounter.increment();
                return ResponseEntity.ok("OK");
            } catch (Exception ex) {
                errorCounter.increment();
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                        .body("ERROR");
            }
        });
    }

    private void simulateExternalDependency() {
        try {
            Thread.sleep(150L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — те же метрики SLO**

```kotlin
package com.example.grafana.alerts

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.time.Duration

@RestController
@RequestMapping("/api/alerts/checkout")
class CheckoutSloController(
    registry: MeterRegistry
) {

    private val successCounter: Counter = Counter.builder("checkout_requests_total")
        .description("Запросы checkout, успешные")
        .tag("result", "success")
        .register(registry)

    private val errorCounter: Counter = Counter.builder("checkout_requests_total")
        .description("Запросы checkout, ошибки")
        .tag("result", "error")
        .register(registry)

    private val latencyTimer: Timer = Timer.builder("checkout_request_duration_seconds")
        .description("Время обработки checkout-запроса")
        .publishPercentileHistogram()
        .serviceLevelObjectives(
            Duration.ofMillis(100),
            Duration.ofMillis(300),
            Duration.ofMillis(500),
            Duration.ofSeconds(1)
        )
        .register(registry)

    @PostMapping
    fun checkout(
        @RequestParam("userId") userId: String,
        @RequestParam("amount") amount: Long
    ): ResponseEntity<String> {
        return latencyTimer.record<ResponseEntity<String>> {
            try {
                simulateExternalDependency()
                successCounter.increment()
                ResponseEntity.ok("OK")
            } catch (ex: Exception) {
                errorCounter.increment()
                ResponseEntity
                    .status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("ERROR")
            }
        }
    }

    private fun simulateExternalDependency() {
        try {
            Thread.sleep(150L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

По этим метрикам в Prometheus/Grafana легко построить SLO-графики и MWMBR-алёрты: error rate, good rate, burn rate, статусы SLO.

---

## Drill-down: метрика→трейс (exemplars), метрика→логи (traceId в MDC)

Drill-down — это сценарий «от симптома к причине»: ты видишь, что на графике latency выросла на 200 мс, кликаешь, попадаешь в конкретный trace с аномалией, а оттуда — в логи нужного запроса. Чтобы это работало, нужно связать **метрики, трейсы и логи**: метрика должна знать traceId, trace должен знать spanId и контекст, а лог — содержать traceId/spanId так, чтобы их можно было искать.

Exemplars — механизм Prometheus/OTel, который позволяет «прикрепить» к bucket’у гистограммы ссылку на конкретный trace. Когда в приложении включён OTel-трейсинг и Micrometer умеет отдавать exemplars, каждый sample latency может содержать небольшие метаданные с `trace_id`. Grafana, видя эти exemplars, показывает точки на графике, по клику на которые можно перейти в trace-viewer. Это очень мощный инструмент: ты сразу попадаешь именно в тот запрос, который был «подозрительным» для данного измерения.

Для того чтобы exemplars появлялись, нужно три вещи: гистограммы в метриках, включённый tracing (Micrometer Tracing + OTel bridge) и корректная интеграция с Prometheus/OTEL collector, который не выкидывает exemplars при экспорте. На уровне кода ты обычно только включаешь `publishPercentileHistogram()` и подключаешь Micrometer Tracing; всё остальное делает инфраструктура. Observation API автоматически связывает метрики и span’ы.

Связка метрика→логи делается по traceId. Если ты кладёшь traceId в MDC (Mapped Diagnostic Context), логгер (Logback/Log4j2) может выводить его в каждую строку. Тогда из trace-viewer (Jaeger/Tempo) можно взять traceId, вставить в поиск по логам (Loki/ELK) и увидеть все строки логов именно этого запроса. Важно, что traceId должен быть **одинаковым** для всех hop’ов цепочки: это обеспечивается пропагацией контекста через HTTP-заголовки (`traceparent`/`tracestate`) и через Kafka/очереди.

Observation API и Micrometer Tracing сильно упрощают жизнь. В Spring Boot 3 достаточно подключить соответствующие зависимости и (опционально) javaagent OTel. Тогда HTTP-запросы, WebClient, JDBC, Kafka автоматически получают span’ы, а traceId попадает в MDC. Всё, что нужно от тебя — настроить формат логов так, чтобы traceId выводился, и не забывать прокидывать контекст через свои асинхронные вызовы (TaskExecutor/Project Reactor).

Drill-down-сценарий в идеале выглядит так: алёрт в Grafana указывает на панель latency; на панели ты видишь всплеск и точки exemplars; кликаешь на точку — попадаешь в trace; в trace видишь, что основное время ушло на запрос в конкретный внешней API; в trace есть spanId; по traceId идёшь в логи и читаешь контекст — параметры, бизнес-идентификаторы, stack trace. После этого у тебя есть гипотеза о причине и конкретные артефакты для пост-мортема.

Если traceId не попал в логи, drill-down ломается. Типичный баг — использование асинхронных пулах потоков/реактивщины без переноса контекста: traceId теряется, span’ы обрываются, в MDC ничего нет. Для TaskExecutor это лечится `TaskDecorator`’ом, для Reactor — `Context` и `Reactor Context Propagation`. Важно явно проверять, что в логах действительно присутствует traceId/spanId для нужных операций.

Ещё один момент — **sampling**. Если ты сильно сэмплируешь трейсы (например, пишешь только 1% запросов), то exemplars и drill-down будут работать только для части запросов. Это нормально, но нужно понимать: если в пике трафика ты видишь рост latency, конкретный проблемный trace может не попасть в сохранённые. Поэтому sampling нужно балансировать: либо динамическое, либо повышенное для ошибочных запросов.

Наконец, интеграция метрика→трейс→лог должна быть не просто технической игрушкой, а частью регулярного процесса расследования инцидентов. On-call должен знать: «Алёрт — панель — trace — логи — гипотеза», и всё это должно быть документировано в runbook’е. Без этого человек под давлением инцидента всё равно будет «тыкаться» в разные дашборды и лог-вьюеры, теряя время.

Ниже пример Java/Kotlin-кода: простой контроллер с `@Observed`, логгирование с traceId в MDC, плюс конфигурация логгера через Logback-паттерн. Это основа для того, чтобы Grafana могла делать drill-down: метрики будут иметь exemplars, трейсы — traceId/spanId, логи — traceId.

**Gradle: зависимости для Micrometer Tracing + OTel (общее для пунктов 3/4/5/7)**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    implementation 'io.micrometer:micrometer-observation'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.40.0'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    implementation("io.micrometer:micrometer-observation")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.40.0")
}
```

**Java — контроллер с `@Observed` и логированием traceId**

```java
package com.example.grafana.drilldown;

import io.micrometer.observation.annotation.Observed;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DrillDownController {

    private static final Logger log = LoggerFactory.getLogger(DrillDownController.class);

    @GetMapping("/api/drilldown/demo")
    @Observed(name = "drilldown.demo", contextualName = "drilldown-demo")
    public String demo() {
        // traceId/spanId уже в MDC (если Micrometer Tracing + OTel настроены)
        String traceId = MDC.get("trace_id");
        log.info("Handling drilldown demo request, traceId={}", traceId);
        simulateWork();
        log.info("Finished drilldown demo request, traceId={}", traceId);
        return "ok";
    }

    private void simulateWork() {
        try {
            Thread.sleep(120L);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Kotlin — тот же контроллер**

```kotlin
package com.example.grafana.drilldown

import io.micrometer.observation.annotation.Observed
import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class DrillDownController {

    private val log = LoggerFactory.getLogger(DrillDownController::class.java)

    @GetMapping("/api/drilldown/demo")
    @Observed(name = "drilldown.demo", contextualName = "drilldown-demo")
    fun demo(): String {
        val traceId = MDC.get("trace_id")
        log.info("Handling drilldown demo request, traceId={}", traceId)
        simulateWork()
        log.info("Finished drilldown demo request, traceId={}", traceId)
        return "ok"
    }

    private fun simulateWork() {
        try {
            Thread.sleep(120L)
        } catch (ex: InterruptedException) {
            Thread.currentThread().interrupt()
        }
    }
}
```

**Фрагмент `logback-spring.xml` — вывод traceId/spanId в логах**

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} -
                traceId=%X{trace_id} spanId=%X{span_id} - %msg%n
            </pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

С такими настройками в Grafana можно кликнуть на exemplar, попасть в trace, взять traceId и по нему найти логи нужного запроса.

---

## Отчётность: артефакты пост-мортема, автоссылки на runbooks

Наблюдаемость — это не только «в момент инцидента», но и **после**: нужно уметь собирать артефакты, писать пост-мортемы, строить отчёты по SLO и показывать бизнесу и менеджменту, как сервис живёт в разрезе недель и месяцев. Здесь Grafana выступает как «визуальный отчётный инструмент»: дашборды, аннотации, снапшоты, PDF-экспорт, ссылки на runbook’и и пост-мортем-документы.

Пост-мортем — это структурированный документ про инцидент: что случилось, когда, какие были симптомы, как отреагировали, сколько времени заняло восстановление, что сделали, чтобы такого больше не было. Без хороших метрик и трейсинга пост-мортем превращается в набор субъективных воспоминаний. С нормальной наблюдаемостью ты можешь показывать скриншоты графиков error rate и latency, выдержки из логов, ссылаться на конкретные trace’ы и SLO-графики. В Grafana удобно использовать аннотации: помечать на графике момент начала инцидента, время срабатывания алёрта и время полного восстановления.

Отчётность по SLO — это регулярные (например, еженедельные) отчёты о том, какие SLO выполнялись, какие нет, сколько было «выпито» error budget’а, сколько инцидентов случилось и какой был их вклад. Эти отчёты можно строить прямо в Grafana через дашборды с агрегациями, а можно использовать Grafana API или Prometheus API, чтобы строить offline-отчёты (PDF/Confluence/почта). Главное — опираться на те же метрики, которые используются в алёртах и операционной работе.

Автоссылки на runbook’и — важный UX-элемент. У каждого алёрта и каждой ключевой панели должен быть **привязан** runbook: документ, который говорит «Если эта панель красная — сделай раз, два, три». В Grafana это делается через annotations/links: панель может иметь ссылку на Confluence/Notion/репозиторий с runbook’ом. Алёрт-сообщение (в Slack/Teams/PagerDuty) должно содержать ту же ссылку, чтобы on-call не искал нужный документ в спешке.

Артефакты инцидентов — это не только графики, но и ссылки на конкретные trace’ы и лог-запросы. Хорошая практика — в пост-мортем-документ вставлять permalinks на дашборды с зафиксированным временным окном, ссылку на trace (если система трейсинга это поддерживает), поиск по логам. Тогда человек, который читает пост-мортем через месяц, может нажать на ссылку и увидеть картину «как оно было» без ручного подбора временных интервалов.

Для автоматизации отчётности можно использовать API Grafana/Prometheus/OTEL collector и скрипты, которые периодически (по cron’у или через CI) собирают ключевые метрики и формируют текстовые/табличные отчёты. В Java/Kotlin можно опросить Prometheus HTTP API, но ещё проще — иногда читать значения метрик прямо из `MeterRegistry` и логировать агрегированные показатели (например, суточный максимум latency, средний error rate). Это не заменяет SLO-отчёты, но даёт дополнительный уровень «самоконтроля» самого сервиса.

Ещё одна полезная техника — **авто-аннотации из приложения**. Сервис может при старте релиза или при активации фичи отправлять событие в Grafana/Prometheus (через Alertmanager или прямой API), которое превращается в аннотацию на графике. Тогда в пост-мортеме легко показать, что проблема началась сразу после deployment’а такой-то версии, и не нужно гадать «когда именно мы выкатывались». Это можно делать либо через отдельный job’ик, либо прямо из Spring Boot при старте.

Runbook’и сами по себе должны быть версионируемыми и доступными: храниться в Git/Confluence, иметь owner’а, отражать актуальную архитектуру. В Grafana лучше ссылаться не на raw-URL (чтобы при переезде wiki не ломать все алёрты), а на стабильные ссылки (например, через короткие redirect’ы или стабильные page-id’ы). Обновление runbook’а должно быть частью процесса изменения сервиса: если ты меняешь алёрты/метрики, надо обновить и документацию.

Сделать «минимально полезную» отчётность можно даже без тяжёлых интеграций: достаточно собирать ключевые метрики и раз в день/неделю логировать агрегированные значения. Ниже пример Java/Kotlin-кода: планировщик раз в час берёт из `MeterRegistry` значения некоторых метрик и пишет их в лог в компактном виде. Эти логи можно потом собрать, проанализировать или превратить в отчёт. В бою чаще используют внешние системы, но принцип «сервис сам умеет рассказать, как он живёт» полезен.

**Java — простой отчёт по метрикам через `MeterRegistry` и `@Scheduled`**

```java
package com.example.grafana.reporting;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.search.Search;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class MetricsReportingJob {

    private static final Logger log = LoggerFactory.getLogger(MetricsReportingJob.class);

    private final MeterRegistry meterRegistry;

    public MetricsReportingJob(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Scheduled(cron = "0 0 * * * *") // каждый час
    public void reportKeyMetrics() {
        double http5xxRate = Search
                .in(meterRegistry)
                .name("http_server_requests_seconds_count")
                .tag("outcome", "SERVER_ERROR")
                .meters()
                .stream()
                .mapToDouble(m -> m.measure().stream()
                        .findFirst()
                        .map(v -> v.getValue())
                        .orElse(0.0))
                .sum();

        double ordersCreated = Search
                .in(meterRegistry)
                .name("orders_created_total")
                .meters()
                .stream()
                .mapToDouble(m -> m.measure().stream()
                        .findFirst()
                        .map(v -> v.getValue())
                        .orElse(0.0))
                .sum();

        log.info("Hourly metrics report: http5xxTotal={}, ordersCreatedTotal={}",
                http5xxRate, ordersCreated);
    }
}
```

**Kotlin — та же задача отчётности**

```kotlin
package com.example.grafana.reporting

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.search.Search
import org.slf4j.LoggerFactory
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class MetricsReportingJob(
    private val meterRegistry: MeterRegistry
) {

    private val log = LoggerFactory.getLogger(MetricsReportingJob::class.java)

    @Scheduled(cron = "0 0 * * * *") // каждый час
    fun reportKeyMetrics() {
        val http5xxTotal = Search
            .in(meterRegistry)
            .name("http_server_requests_seconds_count")
            .tag("outcome", "SERVER_ERROR")
            .meters()
            .sumOf { meter ->
                meter.measure()
                    .firstOrNull()
                    ?.value ?: 0.0
            }

        val ordersCreatedTotal = Search
            .in(meterRegistry)
            .name("orders_created_total")
            .meters()
            .sumOf { meter ->
                meter.measure()
                    .firstOrNull()
                    ?.value ?: 0.0
            }

        log.info(
            "Hourly metrics report: http5xxTotal={}, ordersCreatedTotal={}",
            http5xxTotal,
            ordersCreatedTotal
        )
    }
}
```

Этот код сам по себе не делает красивых PDF, но показывает важный принцип: сервис может периодически «снимать срез» своих ключевых метрик и сохранять его в виде артефактов (логи, файлы, события). В связке с Grafana, Prometheus и runbook’ами это даёт полный цикл: сигнал → диагностика → пост-мортем → отчётность.

# 6. JVM/пулы/GC: «железная» часть метрик

## GC (G1/ZGC): паузы, частота, reclaimed bytes; heap/metaspace, allocation rate

Garbage Collector — один из главных источников как проблем, так и сигналов в JVM-сервисах. В Java 17 по умолчанию используется G1, ZGC доступен как low-latency варианта. Для наблюдаемости важно не только «включить GC-метрики», но и понимать, что именно ты смотришь: паузы (сколько и как часто стопорит мир), частоту циклов, объём освобождённой памяти и общую картину по heap/metaspace. Это позволяет отличать нормальную работу от утечек, фрагментации и ситуаций, когда приложение просто живёт «на грани» по памяти.

В Micrometer GC-метрики обычно приходят из биндеров JVM: `JvmGcMetrics`, `JvmMemoryMetrics` и т.п., а в Spring Boot они включены по умолчанию через Actuator. Ключевые метрики: `jvm_gc_pause_seconds` (гистограмма пауз), `jvm_gc_pause_seconds_count/sum` (сколько пауз и суммарное время), `jvm_gc_memory_allocated_bytes_total` (allocation rate) и `jvm_memory_used_bytes` с разбивкой по memory pool’ам. Уже этого достаточно, чтобы построить графики: сколько времени мы теряем на GC и как быстро растёт heap.

Частота и длительность пауз — разные вещи, и их нужно смотреть вместе. Если GC бегает часто, но паузы короткие, это не всегда проблема; если паузы редкие, но по 2–3 секунды, пользователи это почувствуют. Для G1 характерны относительно короткие, но периодические stop-the-world-паузы; для ZGC целятся в миллисекунды. Поэтому типичные SLO по GC-паузам: p95 < 100–200 мс, p99 < 500 мс, доля времени в паузах < 1–2%. Всё, что выше, — повод разбираться в heap-размере, аллокациях и профиле нагрузки.

Reclaimed bytes (освобождённая память) и allocation rate помогают поймать сценарии «мы просто безумно много создаём объектов». Если `jvm_gc_memory_allocated_bytes_total` растёт со скоростью сотни мегабайт/сек и при этом `jvm_memory_used_bytes` периодически подбирается к верхней границе heap, GC будет работать в тяжёлой нагрузке, даже если паузы пока в рамках. Это хороший повод посмотреть на профилировщик (JFR/async-profiler) и оптимизировать горячие участки кода или поменять структуры данных.

Heap и metaspace в метриках позволяют понять, куда именно утекает память. Heap делится на young/old (у G1 это регионы; но в метриках всё равно видно «eden/survivor/old»), metaspace — область для классов. Если heap стабильно растёт волнами и не возвращается к «базовой линии», это часто признак утечки. Если же растёт metaspace, можно подозревать динамическую генерацию классов (Proxy, ByteBuddy, Hibernate, MapStruct) с утечкой класслоадеров или частые redeploy’и без перезапуска JVM.

Важно помнить, что выбор GC-алгоритма (G1 vs ZGC) меняет форму метрик, но не сами принципы. G1 — компромисс между throughput и паузами, работает через региональный heap и инкрементальные сборки. ZGC — нацелен на минимальные паузы, работать комфортно ему нужно больше памяти. В обоих случаях ты смотришь на одни и те же показатели: pauserate, allocation rate, memory used. Но для ZGC больше внимания уделяется «хвостам» p99, а для G1 — общей доле времени, проведённой в GC.

С точки зрения алёртов по GC-метрикам разумно начинать с простых порогов: доля времени в GC за 5 минут > 10%, p99 `jvm_gc_pause_seconds` > 0.5–1 секунды, рост heap без возвращения к baseline’у в течение N часов. При этом тонкая шкала («сколько именно регионов освобождено») чаще нужна при глубокой диагностике, чем в ежедневном мониторинге. На первом уровне важно видеть тренды и нарушение SLO по latency, а GC уже рассматривать как одну из возможных причин.

Другая практическая мелочь — размер heap. Слишком маленький heap приводит к частым GC, слишком большой — к долгим паузам и медленной фрагментации. Метрики `jvm_memory_max_bytes` и `jvm_memory_used_bytes` помогают увидеть, насколько heap «заполнен», и скоррелировать это с GC-активностью. В Kubernetes это можно связать с requests/limits по памяти: если heap почти равен limit’у, любая вспышка аллокаций будет приводить к OOMKill.

JFR и GC-логирование дополняют метрики: для «разборов полётов» полезно иметь и time-series, и подробный профиль конкретного периода. Наблюдаемость через Prometheus/Grafana отвечает на вопрос «когда и как много GC», а JFR/GC-логи — «что конкретно делал GC в этот момент». Но даже до профилирования GC-метрики из Micrometer дают сильный сигнал, что проблема именно в сборке мусора, а не, например, в сетевых задержках.

На уровне кода Spring Boot почти всё для GC-метрик уже сделано из коробки. Но иногда нужно «подсветить» GC-метрики отдельно или добавить свои агрегаты. Ниже — пример конфигурации, которая подключает стандартные JVM-биндеры и добавляет простой отчёт по GC-пауза-таймеру раз в минуту. Это не заменяет Prometheus, но показывает, как работать с `MeterRegistry` и GC-метриками.

**Gradle: Micrometer JVM-биндеры**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-core'
    implementation 'io.micrometer:micrometer-jvm'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-core")
    implementation("io.micrometer:micrometer-jvm")
}
```

**Java — регистрация JVM-метрик и пример чтения GC-паузы**

```java
package com.example.jvm.gc;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ClassLoaderMetrics;
import io.micrometer.core.instrument.binder.jvm.JvmGcMetrics;
import io.micrometer.core.instrument.binder.jvm.JvmMemoryMetrics;
import io.micrometer.core.instrument.binder.jvm.JvmThreadMetrics;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class GcMetricsConfig {

    private static final Logger log = LoggerFactory.getLogger(GcMetricsConfig.class);

    private final MeterRegistry meterRegistry;

    public GcMetricsConfig(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        new ClassLoaderMetrics().bindTo(meterRegistry);
        new JvmMemoryMetrics().bindTo(meterRegistry);
        new JvmGcMetrics().bindTo(meterRegistry);
        new JvmThreadMetrics().bindTo(meterRegistry);
    }

    @Scheduled(fixedDelayString = "60000")
    public void logGcPauseSummary() {
        meterRegistry.find("jvm_gc_pause_seconds_sum")
                .meters()
                .forEach(meter -> {
                    double value = meter.measure()
                            .stream()
                            .findFirst()
                            .map(m -> m.getValue())
                            .orElse(0.0);
                    log.info("GC pause sum metric={} value={}", meter.getId(), value);
                });
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.jvm.gc

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.binder.jvm.ClassLoaderMetrics
import io.micrometer.core.instrument.binder.jvm.JvmGcMetrics
import io.micrometer.core.instrument.binder.jvm.JvmMemoryMetrics
import io.micrometer.core.instrument.binder.jvm.JvmThreadMetrics
import org.slf4j.LoggerFactory
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class GcMetricsConfig(
    private val meterRegistry: MeterRegistry
) {

    private val log = LoggerFactory.getLogger(GcMetricsConfig::class.java)

    init {
        ClassLoaderMetrics().bindTo(meterRegistry)
        JvmMemoryMetrics().bindTo(meterRegistry)
        JvmGcMetrics().bindTo(meterRegistry)
        JvmThreadMetrics().bindTo(meterRegistry)
    }

    @Scheduled(fixedDelayString = "60000")
    fun logGcPauseSummary() {
        meterRegistry.find("jvm_gc_pause_seconds_sum")
            .meters()
            .forEach { meter ->
                val value = meter.measure()
                    .firstOrNull()
                    ?.value ?: 0.0
                log.info("GC pause sum metric={} value={}", meter.id, value)
            }
    }
}
```

В реальном проде ты не будешь логировать это руками, но такой пример показывает, как включить JVM-биндеры и при необходимости анализировать GC-метрики прямо из кода.

---

## Пулы потоков/очереди: size/active/queue depth/rejected; CallerRunsPolicy

Thread pool — ещё один фундаментальный ресурс, который легко «убить» и который обязан быть наблюдаемым. В Spring Boot потоки используются для `@Async`, `@Scheduled`, Web-серверов, JDBC-пулов, Kafka-consumer’ов и т.д. Для каждого важного пула нужно понимать, сколько потоков активно, сколько задач в очереди, есть ли rejected-задачи и какова политика отказа (например, `CallerRunsPolicy`). Без этих метрик любая проблема с пулами выглядит как «просто всё тормозит».

Основные сигналы по пулу: текущий размер (`poolSize`), активные потоки (`activeCount`), длина очереди (`queueSize`), количество отказанных задач (`rejectedCount`). Когда очередь близка к capacity, пул saturate’ится: новые запросы либо ждут, либо получают отказ. Графики по этим метрикам часто напрямую коррелируют с latency по API и с error rate: если HTTP-пул saturated, ты увидишь рост 503/504, даже если все нижележащие системы работают нормально.

Micrometer имеет биндер `ExecutorServiceMetrics`, который умеет оборачивать `ExecutorService`/`ThreadPoolExecutor` и давать метрики `executor_pool_size`, `executor_active_threads`, `executor_queued_tasks`, `executor_completed_tasks`, `executor_rejected`. Spring Boot также умеет автоматически инстрментировать некоторые executors, но в продакшене надёжнее явно мониторить те пулы, которые критичны для SLA, чтобы не зависеть от магии автоконфигурации.

Политика отказов `RejectedExecutionHandler` определяет, что произойдёт, когда пул и очередь забиты. `CallerRunsPolicy` — один из безопасных вариантов: задача выполняется в вызывающем потоке, что замедляет «продюсеров» и даёт системе шанс «рассосаться». Но при этом latency может резко вырасти, а если вызвать `@Async` из HTTP-потока, запрос будет выполняться синхронно. Другие политики, вроде `AbortPolicy`, кидают исключение и должны сопровождаться корректной обработкой.

Отдельный вопрос — разделение пулов по классам операций. Нехорошо, когда критичные HTTP-запросы и фоновые «тяжёлые» задачи делят один и тот же пул: backoffice-очередь может выжрать все потоки и повесить фронтовые запросы. Хорошая практика — отдельные executors: один для API, один для отчётов, один для долгих джоб. На каждую такую сущность стоит повесить свои метрики и алёрты.

Spring даёт удобный способ настроить `ThreadPoolTaskExecutor` и сделать его наблюдаемым. Мы можем создать бин, настроить poolSize/queueCapacity, обернуть его в `ExecutorServiceMetrics` и использовать как `@Async`-executor. Тогда Micrometer автоматически будет публиковать метрики по этому пулу, и их можно выводить в Grafana.

Ниже пример Java/Kotlin-конфигурации пула для `@Async` с мониторингом через Micrometer. Мы используем `ExecutorServiceMetrics.monitor`, задаём имя `async-executor` и пару тегов. После этого в Prometheus появятся метрики `executor_pool_size{name="async-executor"}` и т.п., и можно строить графики очереди и rejected-задач.

**Java — конфиг ThreadPoolTaskExecutor + Micrometer**

```java
package com.example.jvm.pool;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;

@Configuration
@EnableAsync
public class AsyncExecutorConfig {

    @Bean("asyncExecutor")
    public TaskExecutor asyncExecutor(MeterRegistry meterRegistry) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();

        ThreadPoolExecutor threadPoolExecutor = executor.getThreadPoolExecutor();

        ExecutorService monitored = ExecutorServiceMetrics.monitor(
                meterRegistry,
                threadPoolExecutor,
                "async-executor",
                "executor", "async"
        );

        return new DelegatingExecutorServiceTaskExecutor((ExecutorService) monitored, executor);
    }
}
```

Дополнительный адаптер, чтобы вернуть `TaskExecutor`, оборачивающий `ExecutorService`:

```java
package com.example.jvm.pool;

import org.springframework.core.task.TaskExecutor;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.ExecutorService;

public class DelegatingExecutorServiceTaskExecutor implements TaskExecutor {

    private final ExecutorService executorService;
    private final ThreadPoolTaskExecutor delegate;

    public DelegatingExecutorServiceTaskExecutor(ExecutorService executorService,
                                                 ThreadPoolTaskExecutor delegate) {
        this.executorService = executorService;
        this.delegate = delegate;
    }

    @Override
    public void execute(Runnable task) {
        executorService.submit(task);
    }

    public ThreadPoolTaskExecutor getDelegate() {
        return delegate;
    }
}
```

**Kotlin — тот же конфиг**

```kotlin
package com.example.jvm.pool

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.task.TaskExecutor
import org.springframework.scheduling.annotation.EnableAsync
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import java.util.concurrent.ExecutorService
import java.util.concurrent.ThreadPoolExecutor

@Configuration
@EnableAsync
class AsyncExecutorConfig {

    @Bean("asyncExecutor")
    fun asyncExecutor(meterRegistry: MeterRegistry): TaskExecutor {
        val executor = ThreadPoolTaskExecutor().apply {
            corePoolSize = 8
            maxPoolSize = 16
            setQueueCapacity(200)
            setThreadNamePrefix("async-")
            setRejectedExecutionHandler(ThreadPoolExecutor.CallerRunsPolicy())
            initialize()
        }

        val monitored: ExecutorService = ExecutorServiceMetrics.monitor(
            meterRegistry,
            executor.threadPoolExecutor,
            "async-executor",
            "executor", "async"
        )

        return DelegatingExecutorServiceTaskExecutor(monitored, executor)
    }
}
```

```kotlin
package com.example.jvm.pool

import org.springframework.core.task.TaskExecutor
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
import java.util.concurrent.ExecutorService

class DelegatingExecutorServiceTaskExecutor(
    private val executorService: ExecutorService,
    val delegate: ThreadPoolTaskExecutor
) : TaskExecutor {

    override fun execute(task: Runnable) {
        executorService.submit(task)
    }
}
```

С такой конфигурацией пул потоков становится полноценным объектом наблюдаемости: видно размер, очередь, rejected-задачи и влияние `CallerRunsPolicy` на нагрузку.

---

## JFR/JMX как источники сигналов, интеграция с Micrometer/OTel

JFR (Java Flight Recorder) и JMX — это стандартные механизмы JVM, которые дают глубокие сигналы о работе приложения: профили CPU/памяти, события GC, блокировки, IO-операции, исключения. Они старше и «ниже уровня», чем Micrometer/Prometheus, но прекрасно их дополняют. Задача наблюдаемости — связать эти источники: использовать JFR/JMX как сырьё и экспортировать нужные показатели в Prometheus/OTel.

JFR — это встроенный в JVM профилировщик, который пишет двоичные события (events). Есть готовый `micrometer-registry-jfr`, который умеет читать поток JFR-событий и преобразовывать их в метрики Micrometer. Это позволяет, например, получать метрики по allocation rate, горячим методам, ошибкам и даже пользовательским событиям, не включая тяжёлый профилировщик на проде. JFR эффективен и малозатратен, поэтому его можно держать включённым постоянно с разумной конфигурацией.

JMX — это стандартная модель экспорта MBean’ов. Многие компоненты Java-мира (Kafka, Tomcat, HikariCP, Hazelcast, Redis-клиенты) публикуют свои метрики через MBean’ы. Micrometer имеет `JmxMeterRegistry`, но чаще применяется обратное: вместо того чтобы отдавать метрики в JMX, мы читаем JMX-атрибуты и по ним строим Micrometer-метрики. Для этого можно писать свои бриджи или использовать готовые binders.

Интеграция JFR с Micrometer выглядит так: создаётся `JfrMeterRegistry`, который слушает JFR-события и создаёт метрики. Затем этот реестр либо напрямую экспонируется в Prometheus/OTLP, либо добавляется в composite registry Spring Boot. В результате GC-события, allocation rate и прочие JFR-записи становятся обычными time-series, доступными в Grafana. Это даёт более глубокие сигналы, чем чисто Micrometer-JVM-биндеры, но требует чуть больше настройки.

Интеграция JMX с Micrometer чуть более ручная. Можно создавать gauge’и, которые при чтении ходят в JMX за значением. Например, можно по MBean’у Kafka Consumer’а построить метрики lag’а, количества байт, ошибок. Но здесь важно не переусердствовать: JMX-запросы сами по себе могут быть тяжёлыми, и дергать их на каждом scrape’е нужно с умом. Часто лучше, если библиотека сама публикует метрики в Micrometer, а JMX оставлять как резерв.

В контексте OTel трейсинга JFR и JMX не генерируют спаны напрямую, но могут быть использованы для корреляции: JFR-события GC можно сопоставить с всплеском latency по span’ам; JMX-метрика queue depth — с лагом в Kafka-span’ах. Некоторые решения (например, JFR-to-OTLP экспортеры) умеют превращать JFR-события в OTel-маркировки, но это уже отдельная тема.

Хорошая практика — использовать JFR как «микроскоп» при сложных проблемах производительности и GC, а JMX — как «библиотеку», где лежат low-level-показатели специфичных компонентов. Поверх этого укрепляется слой Micrometer/Prometheus/OTel с метриками уровня сервисов и бизнесов, а трейсинг даёт энд-ту-энд-картину запросов. Всё вместе формирует полноценную картину observability.

Ниже пример Java/Kotlin-конфигурации JFR-registry в Spring Boot. Мы подключаем `micrometer-registry-jfr`, создаём `JfrMeterRegistry` как бин и добавляем его в систему. После этого JFR-метрики становятся доступны в `/actuator/prometheus` или отправляются в OTLP вместе с остальными.

**Gradle: JFR и JMX-регистры Micrometer**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-registry-jfr'
    implementation 'io.micrometer:micrometer-registry-jmx'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-registry-jfr")
    implementation("io.micrometer:micrometer-registry-jmx")
}
```

**Java — конфигурация JFR и JMX MeterRegistry**

```java
package com.example.jvm.jfrjmx;

import io.micrometer.core.instrument.Clock;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.composite.CompositeMeterRegistry;
import io.micrometer.jfr.JfrConfig;
import io.micrometer.jfr.JfrMeterRegistry;
import io.micrometer.jmx.JmxConfig;
import io.micrometer.jmx.JmxMeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JfrJmxRegistryConfig {

    @Bean
    public JfrMeterRegistry jfrMeterRegistry(Clock clock) {
        JfrConfig config = key -> null;
        return new JfrMeterRegistry(config, clock);
    }

    @Bean
    public JmxMeterRegistry jmxMeterRegistry(Clock clock) {
        JmxConfig config = key -> null;
        return new JmxMeterRegistry(config, clock);
    }

    @Bean
    public MeterRegistry compositeMeterRegistry(JfrMeterRegistry jfr,
                                                JmxMeterRegistry jmx) {
        CompositeMeterRegistry composite = new CompositeMeterRegistry();
        composite.add(jfr);
        composite.add(jmx);
        return composite;
    }
}
```

**Kotlin — аналогичная конфигурация**

```kotlin
package com.example.jvm.jfrjmx

import io.micrometer.core.instrument.Clock
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.composite.CompositeMeterRegistry
import io.micrometer.jfr.JfrConfig
import io.micrometer.jfr.JfrMeterRegistry
import io.micrometer.jmx.JmxConfig
import io.micrometer.jmx.JmxMeterRegistry
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class JfrJmxRegistryConfig {

    @Bean
    fun jfrMeterRegistry(clock: Clock): JfrMeterRegistry {
        val config = JfrConfig { null }
        return JfrMeterRegistry(config, clock)
    }

    @Bean
    fun jmxMeterRegistry(clock: Clock): JmxMeterRegistry {
        val config = JmxConfig { null }
        return JmxMeterRegistry(config, clock)
    }

    @Bean
    fun compositeMeterRegistry(
        jfr: JfrMeterRegistry,
        jmx: JmxMeterRegistry
    ): MeterRegistry {
        return CompositeMeterRegistry().apply {
            add(jfr)
            add(jmx)
        }
    }
}
```

В реальном сервисе CompositeMeterRegistry дополнительно будет включать Prometheus/OTLP-регистры, а JFR/JMX станут источниками низкоуровневых метрик, автоматически попадающих в общую телеметрию.

---

## Нагрузочные бюджеты: читать saturations и планировать CPU/memory/IO

Нагрузочный бюджет — это практическое воплощение идеи «сколько ресурсов у нас есть и сколько мы готовы отдать этому сервису». На уровне наблюдаемости он выражается в метриках utilization/saturation/error по CPU, памяти, IO, пулам потоков и соединений. Наша задача — не просто смотреть на «CPU 70%» в вакууме, а связывать эти показатели с SLO, нагрузкой и архитектурными решениями.

CPU-utilization (`system_cpu_usage`, `process_cpu_usage`) говорит, насколько приложение загружает ядра. Если сервис живёт на 70–80% CPU при пиковых нагрузках, у него мало запаса для всплесков и «чёрных лебедей». GC и thread pools — лишь часть картины: при высоком CPU latency растёт даже при идеальных настройках GC. Нагрузочный бюджет по CPU обычно задаётся как «целевой диапазон»: например, 50–60% в среднем, до 80% в пике, при этом SLO по latency должен выдерживаться.

Память/heap — другой важный ресурс. Метрики `jvm_memory_used_bytes` и `jvm_memory_max_bytes` в связке с GC-метриками показывают, насколько мы близки к OOM и сколько «запаса» у heap. Если heap заполнен на 90–95%, GC начнёт работать агрессивнее, паузы вырастут, а любое дополнительное выделение может привести к убийству pod’а в Kubernetes. Нагрузочный бюджет по памяти — это выбор heap-размера, который даёт баланс между плотностью размещения pod’ов и устойчивостью.

IO-ресурсы — менее очевидны, но не менее важны: количество открытых файлов (FD), сеть (bandwidth, RTT), дисковая подсистема (latency, throughput). В Java-микросервисах чаще всего «узким местом» оказывается сеть и БД, а не CPU. Метрики пула соединений к БД (HikariCP), кэшам (Redis), внешним API (HTTP-клиенты) дают представление о saturations: сколько соединений активны, сколько ждут, сколько отказов из-за лимитов.

Thread pools и connection pools тесно связаны с нагрузочным бюджетом. Если пул слишком маленький, он создаёт искусственное «бутылочное горлышко» и повышает latency; если слишком большой — создаёт лишнее давление на CPU и БД. Через метрики `executor_active_threads`, `executor_queued_tasks`, `hikaricp_connections_active` можно видеть, как сервис использует свои ресурсы и где он упирается. Именно на этой информации строится решение «увеличить пул», «поднять реплики», «шардировать БД».

Планирование ресурсов в Kubernetes (requests/limits) опирается на наблюдаемость. Без метрик CPU/memory/GC/пулов ты либо ставишь «с запасом на всякий случай» и переплачиваешь, либо даёшь слишком мало и постоянно ловишь throttling и OOMKill. Нагрузочный бюджет должен быть измеримым: «при X RPS сервис потребляет Y CPU и Z памяти, мы держим запас 30%». Тогда изменения в коде можно контролировать по изменению профиля метрик, а не угадывать.

Нагрузочные бюджеты также помогают в capacity planning: сколько ещё сервисов можно посадить на этот кластер, сколько реплик нужно, чтобы выдержать чёрную пятницу, где нужна горизонтальная, а где вертикальная масштабируемость. Если по GC/heap видно, что сервис комфортно живёт на 512МБ heap при 2 vCPU, а latency и error rate в норме, то масштабирование в первую очередь будет по числу реплик, а не по «поднимем heap до 4ГБ и посадим одну жирную ноду».

Практический способ использовать наблюдаемость для бюджетов — периодические «срезы» и отчёты. Например, раз в час/день снимать агрегаты по CPU, heap, GC, пулам, записывать их в логи и использовать как базу для отчётов. В проде это чаще делается на уровне Prometheus/Grafana, но и сам сервис может «понимать», что он живёт близко к лимитам, и сигнализировать через свои метрики.

Ниже пример простой Java/Kotlin-задачи, которая раз в минуту читает CPU/heap-метрики из `MeterRegistry` и логирует их. В бою лучше полагаться на Prometheus для агрегаций, но такой код иллюстрирует, как можно использовать наблюдаемость внутри сервиса для самоконтроля или, например, для адаптивного тюнинга пулов.

**Java — чтение CPU/heap и логирование нагрузочного среза**

```java
package com.example.jvm.capacity;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.search.Search;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class CapacitySnapshotJob {

    private static final Logger log = LoggerFactory.getLogger(CapacitySnapshotJob.class);

    private final MeterRegistry meterRegistry;

    public CapacitySnapshotJob(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Scheduled(fixedDelayString = "60000")
    public void logCapacitySnapshot() {
        double processCpu = getGauge("process.cpu.usage");
        double systemCpu = getGauge("system.cpu.usage");
        double heapUsed = getGauge("jvm.memory.used", "area", "heap");
        double heapMax = getGauge("jvm.memory.max", "area", "heap");

        log.info("Capacity snapshot: processCpu={}, systemCpu={}, heapUsed={} MB, heapMax={} MB",
                processCpu,
                systemCpu,
                heapUsed / (1024 * 1024),
                heapMax / (1024 * 1024));
    }

    private double getGauge(String name) {
        return Search.in(meterRegistry)
                .name(name)
                .gauges()
                .stream()
                .findFirst()
                .map(g -> g.value())
                .orElse(0.0);
    }

    private double getGauge(String name, String tagKey, String tagValue) {
        return Search.in(meterRegistry)
                .name(name)
                .tag(tagKey, tagValue)
                .gauges()
                .stream()
                .findFirst()
                .map(g -> g.value())
                .orElse(0.0);
    }
}
```

**Kotlin — тот же snapshot-job**

```kotlin
package com.example.jvm.capacity

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.search.Search
import org.slf4j.LoggerFactory
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class CapacitySnapshotJob(
    private val meterRegistry: MeterRegistry
) {

    private val log = LoggerFactory.getLogger(CapacitySnapshotJob::class.java)

    @Scheduled(fixedDelayString = "60000")
    fun logCapacitySnapshot() {
        val processCpu = getGauge("process.cpu.usage")
        val systemCpu = getGauge("system.cpu.usage")
        val heapUsed = getGauge("jvm.memory.used", "area", "heap")
        val heapMax = getGauge("jvm.memory.max", "area", "heap")

        log.info(
            "Capacity snapshot: processCpu={}, systemCpu={}, heapUsed={} MB, heapMax={} MB",
            processCpu,
            systemCpu,
            heapUsed / (1024 * 1024),
            heapMax / (1024 * 1024)
        )
    }

    private fun getGauge(name: String): Double =
        Search.in(meterRegistry)
            .name(name)
            .gauges()
            .firstOrNull()
            ?.value ?: 0.0

    private fun getGauge(name: String, tagKey: String, tagValue: String): Double =
        Search.in(meterRegistry)
            .name(name)
            .tag(tagKey, tagValue)
            .gauges()
            .firstOrNull()
            ?.value ?: 0.0
}
```

В реальной системе такие срезы будут визуализированы в Grafana и станут основой для решений по capacity planning: сколько CPU/памяти/IO мы выделяем сервису, где лимиты, как меняется нагрузка со временем и какие ресурсы станут узким местом при росте трафика.

# 7. Трейсинг: модель, семантика и контекст

## Спаны: серверный/клиентский, атрибуты (семантика OTel), events/status

Спан — это минимальная «единица работы» в распределённом трейсинге. Один HTTP-запрос, один вызов внешнего сервиса, один шаг бизнес-процесса — всё это кандидаты на отдельные спаны. Набор связанных спанов образует трейс (trace) — дерево, показывающее путь запроса через систему. В терминах OpenTelemetry есть разные виды спанов (SpanKind): SERVER (обработка входящего запроса), CLIENT (исходящий вызов), INTERNAL (внутренняя операция), PRODUCER/CONSUMER (сообщения). Понимать, какой именно вид спана вы создаёте, важно для корректной визуализации и агрегаций.

Серверный спан обычно создаётся автоматически HTTP-фреймворком или агентом: входящий запрос в Spring MVC/WebFlux, gRPC-вызов, Kafka-сообщение. Такой спан становится корнем (или дочерним к уже существующему, если пришёл trace-контекст) и несёт базовые атрибуты: путь, метод, статус, размер тела, сетевые параметры. В нормальной жизни разработчик редко создаёт серверный спан руками — это делает автоинструментация (OTel javaagent, Spring Observations), а ваша задача — не ломать этот контекст и правильно создавать дочерние спаны.

Клиентский спан отражает исходящий вызов: HTTP-запрос через WebClient/RestTemplate, обращение к Redis, запрос в БД, публикация события в брокер. Здесь важно, чтобы клиентский спан был дочерним к текущему серверному — тогда трейс останется сквозным. Типичная ошибка — вызывать внешние API без активного контекста (например, в отдельном потоке без переноса контекста), тогда клиентские спаны станут отдельными корнями, и трейс «разорвётся». Поэтому важно, чтобы код, делающий вызов, выполнялся внутри `Span.current()`/`Observation`.

Атрибуты (attributes) — это пара ключ-значение, которая описывает спан. Семантика OpenTelemetry предлагает стандартные ключи (`http.method`, `http.route`, `db.system`, `messaging.system`, `exception.type` и т.д.), чтобы разные библиотеки и визуализаторы понимали друг друга. К этому набору вы добавляете свои доменные атрибуты: `app.order.id`, `app.tenant.id`, `app.feature`. Главное — не превращать атрибуты в свалку PII и не класть туда high-cardinality значения типа `email`, `fullName` и т.п. Для идентификации есть логи, а в атрибутах важнее агрегируемые признаки.

Атрибуты живут на всём протяжении спана и хорошо подходят для того, чтобы фильтровать и группировать трейсы. Например, можно быстро посмотреть все запросы с `app.tenant.id=123`, или все спаны, где `app.payment.method=card`. Это удобно для расследования инцидентов и анализа поведения конкретных сегментов пользователей. Но за это платишь кардинальностью: чем больше разных значений, тем тяжелее хранилищу трейсинга. Поэтому доменные атрибуты нужно проектировать так же аккуратно, как лейблы метрик.

Events (события) внутри спана — это отметки по времени, привязанные к конкретным точкам исполнения. Это не логи, а структурированные «маячки»: `validation.started`, `validation.completed`, `db.query.started`, `db.query.slow`. Каждое событие имеет timestamp и опциональные атрибуты. Их удобно использовать, чтобы пометить ключевые фазы внутри одной операции и в UI трейсинга видеть, где конкретно произошёл скачок времени, не создавая новых спанов и не захламляя древо.

Status у спана отвечает не за HTTP-код, а за логическое завершение операции: `UNSET` (по умолчанию), `OK` или `ERROR`. В OpenTelemetry статус задаётся один раз (реально — последний вызов «побеждает»), и нормальная практика — ставить `ERROR` только тогда, когда действительно была ошибка, нарушающая контракт операции. HTTP 4xx не всегда ошибка со стороны сервиса (часто это «клиент ошибся»), и не обязаны автоматически помечаться как `ERROR`. Важно договориться в команде, что именно считается ошибочным исходом.

Спаны организованы в дерево: корневой спан (обычно HTTP SERVER или messaging CONSUMER) и дочерние спаны, описывающие отдельные шаги. Глубина дерева не должна быть избыточной: если на каждый `if` делать отдельный спан — трейс станет нечитаемым. Общая идея: один спан на логический шаг, плюс события внутри него. Для тяжёлых операций с несколькими подэтапами имеет смысл отдельные спаны; для мелких — достаточно events.

Знание устройства спанов помогает читать трейсы как «таймлайн запроса»: где он начал обрабатываться, какие внешние вызовы делались, где были ретраи, какие исключения произошли. Серверный спан показывает общую длительность и статус, клиентские спаны — вклад зависимостей, внутренние спаны — распределение работы по бизнес-шагам. Если при расследовании инцидента вы видите, что серверный спан нормальный, а клиентский HTTP-спан вешается на 5 секунд — ищите проблему во внешней системе или сети, а не в вашем коде.

Кроме атрибутов и событий спаны служат связующим звеном с метриками и логами. Через exemplars метрика latency может указывать на конкретный spаn, а через traceId из span’а вы найдёте логи в Loki/ELK. Поэтому важно, чтобы основной обработчик запроса всегда был под span’ом, а логи писались с traceId. Если разработчик создаёт собственные внутренние спаны, нужно использовать тот же Tracer/Observation, который использует инфраструктура, иначе вы получите две «реальности» в трейсах.

Практическое правило: минимум 3 вида спанов — входящий запрос (SERVER), исходящий вызов (CLIENT) и крупные бизнес-операции (INTERNAL). Остальное — по необходимости. Всегда используйте семантику OTel для технических атрибутов и аккуратно добавляйте доменные, избегая лишней кардинальности. События включайте для важных шагов внутри спана, а статус и ошибки выставляйте в одном месте, ближе к границе слоя, который «владеет» операцией.

Ниже — пример ручного создания внутреннего спана с атрибутами, событиями и статусом на Java и Kotlin с использованием OpenTelemetry API. В реальном Spring Boot большую часть спанов создаст автоинструментация (javaagent), а такие конструкции пригодятся для бизнес-шагов внутри вашего сервиса.

**Java — внутренний спан с атрибутами и events**

```java
package com.example.tracing.spans;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import org.springframework.stereotype.Service;

@Service
public class OrderTracingService {

    private final Tracer tracer;

    public OrderTracingService() {
        this.tracer = GlobalOpenTelemetry
                .getTracer("com.example.tracing.spans.OrderTracingService");
    }

    public void processOrder(String orderId, String userId) {
        Span span = tracer.spanBuilder("order.process")
                .setSpanKind(SpanKind.INTERNAL)
                .setAttribute("app.order.id", orderId)
                .setAttribute("app.user.id", userId)
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            span.addEvent("validation.started");
            validateOrder(orderId);
            span.addEvent("validation.completed");

            span.addEvent("payment.started");
            callPaymentGateway(orderId);
            span.addEvent("payment.completed");

            span.setStatus(StatusCode.OK);
        } catch (Exception ex) {
            span.recordException(ex);
            span.setStatus(StatusCode.ERROR, "Order processing failed");
            throw ex;
        } finally {
            span.end();
        }
    }

    private void validateOrder(String orderId) {
        // Здесь могла бы быть настоящая валидация
    }

    private void callPaymentGateway(String orderId) {
        // Здесь мог бы быть HTTP-вызов внешнего платёжного сервиса
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.tracing.spans

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.trace.Span
import io.opentelemetry.api.trace.SpanKind
import io.opentelemetry.api.trace.StatusCode
import io.opentelemetry.api.trace.Tracer
import io.opentelemetry.context.Scope
import org.springframework.stereotype.Service

@Service
class OrderTracingService {

    private val tracer: Tracer = GlobalOpenTelemetry
        .getTracer("com.example.tracing.spans.OrderTracingService")

    fun processOrder(orderId: String, userId: String) {
        val span: Span = tracer.spanBuilder("order.process")
            .setSpanKind(SpanKind.INTERNAL)
            .setAttribute("app.order.id", orderId)
            .setAttribute("app.user.id", userId)
            .startSpan()

        try {
            val scope: Scope = span.makeCurrent()
            scope.use {
                span.addEvent("validation.started")
                validateOrder(orderId)
                span.addEvent("validation.completed")

                span.addEvent("payment.started")
                callPaymentGateway(orderId)
                span.addEvent("payment.completed")

                span.setStatus(StatusCode.OK)
            }
        } catch (ex: Exception) {
            span.recordException(ex)
            span.setStatus(StatusCode.ERROR, "Order processing failed")
            throw ex
        } finally {
            span.end()
        }
    }

    private fun validateOrder(orderId: String) {
        // Здесь могла бы быть настоящая валидация
    }

    private fun callPaymentGateway(orderId: String) {
        // Здесь мог бы быть HTTP-вызов внешнего платёжного сервиса
    }
}
```

---

## Пропагация: W3C traceparent/tracestate, baggage (минимум)

Пропагация — это механизм, который позволяет trace-контексту пройти через десятки сервисов, не потерявшись. Технологически это «просто» про набор HTTP-заголовков или метаданных в протоколе, но логически — про непрерывный запрос, зашедший в систему и прошедший через все её слои. Если контекст порвался — трейс распадается на отдельные куски, и end-to-end картинка теряется. Поэтому важно понимать базовые стандарты и не ломать их своими фильтрами/проксями.

Стандарт W3C Trace Context описывает два основных заголовка: `traceparent` и `tracestate`. `traceparent` содержит версию, traceId, spanId и набор флагов (например, sampled). Это минимальный набор, который позволяет однозначно идентифицировать трейс и текущий спан. `tracestate` — расширяемый ключ-значение список для вендор-специфичных данных: какая система создала трейс, какие флаги она выставила, какие внутренние идентификаторы используются. Большинство приложений напрямую `tracestate` не трогают, им достаточно корректно прокидывать его дальше.

Когда Spring Boot с OTel-агентом получает входящий HTTP-запрос, он читает `traceparent`/`tracestate`, создаёт SpanContext и серверный спан, привязанный к этому контексту. При исходящем вызове (WebClient/RestTemplate/Feign) тот же контекст сериализуется обратно в заголовки и уходит дальше. Благодаря этому цепочка «браузер → API-шлюз → BFF → микросервисы → базы/кэши» остаётся одним traceId. Важно не потерять эти заголовки на проксях и балансировщиках (Nginx/Istio и т.п.).

Baggage — это отдельный набор key-value пар, который путешествует вместе с trace-контекстом, но логически относится не к спану, а к «контексту запроса». Типичный пример — `tenant.id`, `locale`, `experiment.group`. Это именно «минимальный контекст», который нужен многим сервисам по ходу обработки запроса. Он кодируется в заголовки (`baggage` по W3C или `baggage`/`correlation-context` в старых спецификациях) и доступен из любого места, где есть текущий контекст.

Основной риск baggage — превратить его в «тележку с PII и лишними данными». Каждый ключ в baggage повторяется на каждом хопе, увеличивает размер заголовков и нагрузку на сеть/балансировщики. Если туда положить `user.email` или JSON, легко получить и проблемы с безопасностью, и технические ограничения (длина заголовков, лимиты в проксях). Поэтому правило простое: baggage — только для пары-тройки критичных доменных признаков, которые действительно нужны почти всем участникам цепочки, и никак не для персональных данных.

В Spring-мире пропагацией занимаются либо автоинструментированные библиотеки (OTel javaagent, Spring Cloud Sleuth 3/Brave в прошлом), либо сам OpenTelemetry SDK. Большинство HTTP-клиентов и серверов уже умеют работать с W3C-форматом. Но случаи ручной работы с контекстом всё равно встречаются: кастомные протоколы, нестандартные заголовки, пропагция через сообщения в брокерах. Здесь вступают в игру propagator’ы (`TextMapPropagator`), которые сериализуют/десериализуют контекст.

При ручной работе важно всегда пользоваться тем же propagator’ом, который использует SDK, обычно это `W3CTraceContextPropagator`. Он знает, какие заголовки читать/писать, как кодировать флаги, как не сломать `tracestate`. Прямое «самописное» управление заголовком `traceparent` без знания формата — верный путь к поломке совместимости с другими языками/библиотеками, особенно если система полиглотная.

Контекст OTel — это `Context.current()`, внутри которого лежат и Span, и Baggage. Когда вы делаете `span.makeCurrent()`, текущий `Context` обновляется: теперь `Span.current()` и `Baggage.current()` работают корректно. Например, вы можете добавить в baggage `tenant.id`, и он автоматически поедет дальше с каждым исходящим вызовом. Либо вы можете извлечь baggage, чтобы проставить tenant в логах или в доменной логике.

Ручная пропагация особенно важна там, где по умолчанию нет интеграции: нестандартные HTTP-клиенты, бинарные протоколы, файловые очереди. Здесь вы сами вызываете `propagator.inject` при отправке и `propagator.extract` при получении. Это позволяет сохранять сквозной traceId даже в старых/нестандартных интеграциях, а в трейс-вьюере видеть общую цепочку. Важно аккуратно выбрать carrier (объект, в который записываются заголовки/метаданные) и сеттер/геттер для него.

Ниже пример, где мы вручную создаём клиентский спан и с помощью W3C propagator’а инжектим trace-контекст и baggage в HTTP-заголовки перед вызовом внешнего сервиса. В реальном приложении за вас это сделает WebClient + OTel javaagent, но пример показывает, ЧТО именно происходит под капотом.

**Java — ручная пропагация traceparent/baggage в RestTemplate**

```java
package com.example.tracing.propagation;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.baggage.Baggage;
import io.opentelemetry.api.baggage.BaggageBuilder;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ExternalClientService {

    private final RestTemplate restTemplate;
    private final Tracer tracer;
    private final TextMapPropagator propagator;

    public ExternalClientService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
        this.tracer = GlobalOpenTelemetry
                .getTracer("com.example.tracing.propagation.ExternalClientService");
        this.propagator = W3CTraceContextPropagator.getInstance();
    }

    public String callExternalApi(String url, String tenantId) {
        BaggageBuilder baggageBuilder = Baggage.current().toBuilder();
        Baggage baggage = baggageBuilder
                .put("tenant.id", tenantId)
                .build();

        Span span = tracer.spanBuilder("external.call")
                .setSpanKind(SpanKind.CLIENT)
                .setAttribute("app.tenant.id", tenantId)
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            Context contextWithBaggage = baggage.storeInContext(Context.current());

            HttpHeaders headers = new HttpHeaders();
            TextMapSetter<HttpHeaders> setter = (carrier, key, value) -> carrier.set(key, value);

            propagator.inject(contextWithBaggage, headers, setter);

            HttpEntity<String> entity = new HttpEntity<>("{}", headers);
            return restTemplate
                    .exchange(url, HttpMethod.POST, entity, String.class)
                    .getBody();
        } finally {
            span.end();
        }
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.tracing.propagation

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.baggage.Baggage
import io.opentelemetry.api.trace.Span
import io.opentelemetry.api.trace.SpanKind
import io.opentelemetry.api.trace.Tracer
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator
import io.opentelemetry.context.Context
import io.opentelemetry.context.Scope
import io.opentelemetry.context.propagation.TextMapPropagator
import io.opentelemetry.context.propagation.TextMapSetter
import org.springframework.http.HttpEntity
import org.springframework.http.HttpHeaders
import org.springframework.http.HttpMethod
import org.springframework.stereotype.Service
import org.springframework.web.client.RestTemplate

@Service
class ExternalClientService(
    private val restTemplate: RestTemplate
) {

    private val tracer: Tracer = GlobalOpenTelemetry
        .getTracer("com.example.tracing.propagation.ExternalClientService")

    private val propagator: TextMapPropagator =
        W3CTraceContextPropagator.getInstance()

    fun callExternalApi(url: String, tenantId: String): String? {
        val baggage: Baggage = Baggage.current()
            .toBuilder()
            .put("tenant.id", tenantId)
            .build()

        val span: Span = tracer.spanBuilder("external.call")
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("app.tenant.id", tenantId)
            .startSpan()

        return try {
            val scope: Scope = span.makeCurrent()
            scope.use {
                val contextWithBaggage: Context = baggage.storeInContext(Context.current())

                val headers = HttpHeaders()
                val setter = TextMapSetter<HttpHeaders> { carrier, key, value ->
                    carrier.set(key, value)
                }

                propagator.inject(contextWithBaggage, headers, setter)

                val entity = HttpEntity("{}", headers)
                restTemplate
                    .exchange(url, HttpMethod.POST, entity, String::class.java)
                    .body
            }
        } finally {
            span.end()
        }
    }
}
```

---

## Ошибки и статус: где ставить error.flag, как не раздувать спаны логами

Работа с ошибками в трейсинге — это баланс между полезностью и шумом. Если помечать `ERROR` и `exception` на каждом чихе, трейс-хранилище быстро превратится в помойку, а визуализатор будет показывать всё красным. Если не помечать вовсе — при инциденте придётся вручную выискивать, где именно случилась проблема. Поэтому нужны правила: где устанавливать статус, сколько раз записывать исключение, как не дублировать одно и то же по всей цепочке спанов.

В OpenTelemetry статус спана — это скорее «результат операции», чем «наличие исключения в коде». Представьте API, который валидирует входные данные и при ошибке возвращает 400 без выброса Exception на уровне сервера: с точки зрения трейсинга это нормальное завершение операции, статус можно оставить `OK` или `UNSET`, потому что контракт API соблюдён. Наоборот, если из-за внутренней ошибки или таймаута сервис возвращает 500 — это логическая ошибка, и спан должен получить `StatusCode.ERROR`.

`Span.recordException` — мощный инструмент, но он должен вызываться не во всех местах подряд. Идея в том, что exception записывается в том слое, который «владеет» операцией и принимает решение о её результате. Например, если у вас есть слой DAO, сервис и контроллер, то exception при ошибке в БД лучше записать на спане уровня сервиса или контроллера, а не в каждом DAO-методе. Иначе вы получите три события об одном и том же исключении в разных спанах.

Нужно чётко разделять ошибки уровня инфраструктуры (таймауты, соединения, отказ зависимостей) и ошибки бизнес-логики (валидация, бизнес-ограничения). В трейсинг обычно попадают обе, но интерпретация разная. Инфраструктурные ошибки чаще приводят к нарушению SLO и требуют реакций SRE; бизнес-ошибки важнее продукту и аналитикам. Поэтому в атрибутах и статусе имеет смысл явно отмечать тип ошибки: `error.type=INFRASTRUCTURE/BUSINESS`, `error.category=TIMEOUT/VALIDATION`.

Нельзя превращать спан в лог-файл. Спан должен содержать краткую, структурированную информацию: пару атрибутов, одно-два события, один `recordException` с message и stacktrace. Большие текстовые тела, debug-данные, дампы объектов — это задача логов. Если каждое событие в спане — это копия строки из лога, вы получите огромные трейсы и медленную визуализацию. Лучше записать ключевые факты в атрибуты (код ошибки, тип, ID операции), а детали оставить в логах, связанных через traceId.

Отдельный вопрос — как работать с ретраями. Часто внешний вызов обёрнут в retry-механику, и один логический шаг даёт несколько клиентских спанов: первый вызов, второй, третий. Если каждый из них помечать `ERROR`, в трейс-вьюере будет много красного, хотя на уровне бизнеса всё закончилось ОК. Здесь помогают атрибуты: на клиентских спанах можно ставить `retry.attempt=1/2/3`, `error.severity=transient`, а итоговый серверный спан оставить `OK`, если ретрай прошёл успешно.

Важный аспект — кто принимает финальное решение об error-статусе. Обычно это граница API: контроллер или handler. Внутренний сервис может поймать исключение, завернуть в доменный тип и вернуть «частичный успех»; контроллер решит, какой HTTP-код отдать и как отметить спан. Если внутренняя функция пометит спан как `ERROR`, а верхний уровень — как `OK`, будут противоречия. Поэтому лучше выработать правило: статус корневого спана выставляет верхний уровень, а внутренние спаны — только в случае локальных фатальных ошибок.

Инструментально всё сводится к тому, что при перехвате исключения вы делаете две вещи: `span.recordException(ex)` и `span.setStatus(StatusCode.ERROR, "…")`. Некоторые фреймворки делают это автоматически (например, OTel-инструментация для Spring Web), но в своём бизнес-коде вы часто будете делать это вручную или через `@Observed` и обработчики. Главное — не повторять запись одного и того же exception в десяти местах.

Ниже пример сервиса, где бизнес-операция оборачивается в спан: при успешном завершении выставляется `OK`, при ошибке — `ERROR` и записывается exception. Логирование при этом остаётся компактным и опирается на traceId для детализации в лог-системе.

**Java — обработка ошибок и статус спана**

```java
package com.example.tracing.errors;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class PaymentErrorHandlingService {

    private static final Logger log = LoggerFactory.getLogger(PaymentErrorHandlingService.class);

    private final Tracer tracer;

    public PaymentErrorHandlingService() {
        this.tracer = GlobalOpenTelemetry
                .getTracer("com.example.tracing.errors.PaymentErrorHandlingService");
    }

    public void processPayment(String paymentId) {
        Span span = tracer.spanBuilder("payment.process")
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("app.payment.id", paymentId);
            doProcess(paymentId);
            span.setStatus(StatusCode.OK);
        } catch (BusinessException ex) {
            span.setAttribute("error.category", "BUSINESS");
            span.recordException(ex);
            span.setStatus(StatusCode.ERROR, ex.getMessage());
            log.warn("Business error while processing payment {}: {}", paymentId, ex.getMessage());
            throw ex;
        } catch (Exception ex) {
            span.setAttribute("error.category", "INFRASTRUCTURE");
            span.recordException(ex);
            span.setStatus(StatusCode.ERROR, "Unexpected infrastructure error");
            log.error("Infrastructure error while processing payment {}", paymentId, ex);
            throw ex;
        } finally {
            span.end();
        }
    }

    private void doProcess(String paymentId) {
        // Здесь может быть вызов БД, внешних сервисов и др. логика
    }

    public static class BusinessException extends RuntimeException {
        public BusinessException(String message) {
            super(message);
        }
    }
}
```

**Kotlin — тот же паттерн**

```kotlin
package com.example.tracing.errors

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.trace.Span
import io.opentelemetry.api.trace.StatusCode
import io.opentelemetry.api.trace.Tracer
import io.opentelemetry.context.Scope
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service

@Service
class PaymentErrorHandlingService {

    private val log = LoggerFactory.getLogger(PaymentErrorHandlingService::class.java)

    private val tracer: Tracer = GlobalOpenTelemetry
        .getTracer("com.example.tracing.errors.PaymentErrorHandlingService")

    fun processPayment(paymentId: String) {
        val span: Span = tracer.spanBuilder("payment.process").startSpan()

        try {
            val scope: Scope = span.makeCurrent()
            scope.use {
                span.setAttribute("app.payment.id", paymentId)
                doProcess(paymentId)
                span.setStatus(StatusCode.OK)
            }
        } catch (ex: BusinessException) {
            span.setAttribute("error.category", "BUSINESS")
            span.recordException(ex)
            span.setStatus(StatusCode.ERROR, ex.message ?: "Business error")
            log.warn(
                "Business error while processing payment {}: {}",
                paymentId,
                ex.message
            )
            throw ex
        } catch (ex: Exception) {
            span.setAttribute("error.category", "INFRASTRUCTURE")
            span.recordException(ex)
            span.setStatus(StatusCode.ERROR, "Unexpected infrastructure error")
            log.error("Infrastructure error while processing payment {}", paymentId, ex)
            throw ex
        } finally {
            span.end()
        }
    }

    private fun doProcess(paymentId: String) {
        // Здесь может быть вызов БД, внешних сервисов и др. логика
    }

    class BusinessException(message: String) : RuntimeException(message)
}
```

---

## Инструментирование Spring: javaagent OTel и вручную через Observation/Tracer

В Spring Boot есть два основных пути включить трейсинг: автоинструментация через OpenTelemetry javaagent и «ручное» инструментирование через Micrometer Observation/Tracer. На практике почти всегда используется комбинация: javaagent даёт покрытие для стандартных технологий (HTTP, WebClient, JDBC, Kafka и т.п.), а Observation вы используете для бизнес-методов и специфичных участков кода. Важно понимать, кто за что отвечает, чтобы не дублировать работу и не получать двойные спаны.

Javaagent — это `-javaagent:opentelemetry-javaagent.jar`, который вы добавляете в команду запуска JVM (Docker/K8s). Агент подключается к байткоду на лету и оборачивает популярные фреймворки: Spring MVC/WebFlux, RestTemplate, WebClient, JDBC-драйверы, Kafka-клиенты, gRPC и т.д. Он сам создаёт серверные и клиентские спаны, выставляет стандартные атрибуты, инжектит и экстрактит `traceparent`/`tracestate`. Для приложения это полностью прозрачно: код не меняется, но трейсинг появляется.

Micrometer Observation в Spring Boot 3+ — это программная API, которая позволяет описывать операции, к которым нужно прикрутить метрики и трейсы. Когда Observation-слой сконфигурирован с Micrometer Tracing + OTel, каждая `Observation` превращается во span. Аннотация `@Observed` даёт способ быстро обернуть метод сервиса: создаётся span с заданным именем, автоматически добавляются таймеры, traceId попадает в MDC. Это удобный способ «подсветить» бизнес-шифты внутри уже инструментированного по HTTP приложения.

Если javaagent уже создаёт HTTP-спаны и спаны WebClient, вам не нужно руками оборачивать контроллеры в `Span`/`Tracer` — вы только рискуете получить двойные спаны и странную структуру. Вместо этого используйте `@Observed` или `Observation` в местах, где агент ничего не знает о семантике: шаги бизнес-процесса, обращение к доменным компонентам, вычислительные куски, которые важно видеть в трейсах. Это даёт красивое дерево: верхний HTTP-спан → бизнес-спаны → клиентские спаны.

Если по каким-то причинам javaagent использовать нельзя (ограничения инфраструктуры, политика безопасности), у вас остаётся путь ручного конфигурирования OTel SDK и Spring-интеграции. Тогда вы создаёте `SdkTracerProvider`, настраиваете OTLP-экспорт, регистрируете `OpenTelemetry`, а затем используете либо низкоуровневый `io.opentelemetry.api.trace.Tracer`, либо Spring-обёртку `io.micrometer.tracing.Tracer` через Micrometer Tracing. Это более многословно, но даёт полный контроль.

Observation API удобно тем, что работает сразу на два фронта: метрики и трейсинг. Вы описываете «наблюдаемую операцию», а дальше регистры метрик и трейсинга сами решают, какие таймеры/спаны создать. В `@Observed` можно задать `lowCardinalityKeyValues` и `highCardinalityKeyValues`; первые идут в лейблы метрик и в атрибуты спана, вторые — обычно только в спан, чтобы не рвать кардинальность. Это делает Observation хорошим инструментом для «центрированной» вокруг бизнес-операций observability.

Вручную можно делать и более тонкие вещи: `Observation.createNotStarted("inventory.update", registry).lowCardinalityKeyValue("warehouse", warehouse).observe(() -> { ... })`. Внутри `observe` уже будет сделан span, таймер и связка с текущим контекстом (HTTP-спаном, например). Это особенно полезно в сложном доменном коде, где хотите видеть отдельные шаги и измерять их время, но не хочется трогать низкоуровневый OTel API.

Конфигурационно для Spring Boot 3 достаточно добавить зависимости `micrometer-tracing-bridge-otel` и `opentelemetry-exporter-otlp`, выключить устаревший Sleuth и включить OTLP-экспорт. Дальше либо javaagent, либо ручной OTel SDK будут выдавать трейс-данные, а Micrometer Observation — связывать их с кодом. В Kubernetes-окружении обычно есть OTEL Collector как посредник, который принимает OTLP, делает батчинг, фильтрацию и отправляет дальше в Tempo/Jaeger/Zipkin.

Ниже пример сервиса, где один метод аннотирован `@Observed`, а другой использует `Observation` вручную. Они будут создавать спаны, совместимые с автоспанами от агента, и попадут в общую картину трейсинга. Код — на Java и Kotlin.

**Java — использование `@Observed` и ручной `Observation` в Spring Boot 3**

```java
package com.example.tracing.spring;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.annotation.Observed;
import org.springframework.stereotype.Service;

@Service
public class SpringTracingService {

    private final ObservationRegistry observationRegistry;

    public SpringTracingService(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    @Observed(
            name = "payment.confirm",
            contextualName = "payment-confirm",
            lowCardinalityKeyValues = {"operation", "confirm_payment"}
    )
    public void confirmPayment(String paymentId) {
        // Здесь уже есть span/trace, созданный на основе Observation
        // Можно просто писать бизнес-логику
        checkStatus(paymentId);
    }

    public void updateInventory(String itemId, int delta) {
        Observation.createNotStarted("inventory.update", observationRegistry)
                .lowCardinalityKeyValue("item.id", itemId)
                .observe(() -> {
                    // Внутри этого блока создаётся span и таймер
                    doUpdateInventory(itemId, delta);
                });
    }

    private void checkStatus(String paymentId) {
        // Проверка статуса платежа
    }

    private void doUpdateInventory(String itemId, int delta) {
        // Обновление остатков на складе
    }
}
```

**Kotlin — тот же пример с Observation**

```kotlin
package com.example.tracing.spring

import io.micrometer.observation.Observation
import io.micrometer.observation.ObservationRegistry
import io.micrometer.observation.annotation.Observed
import org.springframework.stereotype.Service

@Service
class SpringTracingService(
    private val observationRegistry: ObservationRegistry
) {

    @Observed(
        name = "payment.confirm",
        contextualName = "payment-confirm",
        lowCardinalityKeyValues = ["operation", "confirm_payment"]
    )
    fun confirmPayment(paymentId: String) {
        // Здесь уже есть span/trace, созданный на основе Observation
        checkStatus(paymentId)
    }

    fun updateInventory(itemId: String, delta: Int) {
        Observation.createNotStarted("inventory.update", observationRegistry)
            .lowCardinalityKeyValue("item.id", itemId)
            .observe {
                doUpdateInventory(itemId, delta)
            }
    }

    private fun checkStatus(paymentId: String) {
        // Проверка статуса платежа
    }

    private fun doUpdateInventory(itemId: String, delta: Int) {
        // Обновление остатков на складе
    }
}
```

# 8. Пайплайн трейсинга и sampling

## Архитектура: агент/SDK → OTel Collector (receivers/processors/exporters) → хранилище (Tempo/Jaeger/X-Ray)

Пайплайн трейсинга в продакшене почти никогда не выглядит как «приложение → Jaeger». Между приложением и хранилищем почти всегда стоит OpenTelemetry Collector или его аналоги, а само приложение либо использует встроенный SDK, либо автоинструментируется через javaagent. Логика простая: приложение должно как можно меньше думать о транспорте и форматах, а collector занимается принятием, обработкой, фильтрацией и экспортом данных дальше.

На самом нижнем уровне у тебя есть **агент или SDK**. Агент (javaagent) подключается к JVM без изменений кода, перехватывает фреймворки (Spring, JDBC, Kafka, HTTP-клиенты) и создаёт спаны автоматически. SDK — это ситуация, когда ты сам прописываешь зависимости `opentelemetry-sdk`, настраиваешь `SdkTracerProvider`, экспортеры и сам инициируешь спаны. На практике для типичного Spring Boot микросервиса агент удобнее: быстро даёт покрытие, а руками приходится только слегка доинструментировать бизнес-код.

Дальше — транспорт. Агент/SDK обычно экспортирует трейсы по протоколу **OTLP** (gRPC/HTTP) на адрес OTel Collector’а. Это может быть sidecar в том же pod’е, daemonset на ноде или центральный «gateway-collector» в кластере. Такой подход снимает нагрузку с приложений: им не нужно уметь говорить с Jaeger/Tempo/X-Ray напрямую, они просто отдают OTLP и забывают, а collector уже конвертит, буферизует, ретраит и шлёт в нужные backend’ы.

Внутри OTel Collector архитектура однотипная: **receivers** принимают данные (otlp, jaeger, zipkin, kafka), **processors** их обрабатывают (batch, attributes, filter, tail_sampling, memory_limiter), **exporters** отправляют дальше (otlp, loki, tempo, jaeger, zipkin, awsxray). Это конвейер: входящий поток проходит через цепочку шагов, где на каждом можно что-то «подкрутить» — добавить/удалить атрибуты, ограничить размер, применить sampling, сделать батчинг.

Хранилище трейсинга — третья ступень. Это может быть Tempo, Jaeger, Zipkin, AWS X-Ray, New Relic, Datadog, что угодно. С точки зрения архитектуры лучше относиться к ним как к pluggable backend’ам: collector знает, куда и как слать, но приложение об этом не думает. Если ты завтра решишь переехать с Jaeger на Tempo, достаточно сменить exporter/endpoint в collector’е, не трогая сервисы.

Collector хорошо подходит для **мульти-языковой** среды. Java, Go, Node.js, Python — все могут отправлять OTLP в один и тот же collector, а он уже централизованно обрабатывает всё. Это важный аргумент против подхода «каждый сервис сам говорит с Jaeger по-своему». В микросервисной архитектуре без collector’а быстро наступает зоопарк конфигураций и нестабильность.

В Kubernetes более-менее стандартные топологии две. Первая — **agent mode**: на каждой ноде стоит collector-daemonset, на localhost:4317/4318 слушает OTLP, а приложения на этой ноде отправляют данные именно туда. Collector на ноде делает локальный батчинг и отправляет трейсы дальше в центральный gateway или прямо в хранилище. Вторая — **gateway mode**: есть один или несколько collector’ов в роли ingress для телеметрии, куда ходят все приложения. Часто их комбинируют: node-агенты + центральный gateway.

Зачем вообще нужно всё это усложнение? Во-первых, **декуплинг**: приложение не падает и не тормозит от того, что Jaeger/Tempo недоступен — collector может буферизовать, сбрасывать, деградировать, а приложение продолжать работу. Во-вторых, **политики**: весь sampling, фильтрация PII, агрегация, маршрутизация делают в одном месте. В-третьих, **экономика**: collector может делать downsampling, батчинг и сжатие так, чтобы не разориться на объёмах трейсинга.

Важно понимать, что у тебя в этом пайплайне две точки контроля: на стороне приложения/агента и на стороне collector’а. На стороне приложения ты решаешь, что вообще считать span’ом и какие атрибуты в него класть. На стороне collector’а ты решаешь, какие трейсы сохранять, какие урезать, как их маршрутизировать и какие лимиты применить. Это два уровня, и их нельзя путать: попытка утащить всю логику sampling’а в код приведёт к жёсткой связанности, а попытка всё делать только в collector’е может создать лишнюю нагрузку на сеть.

Ещё один инженерный момент — **батчинг и очередь**. Ни приложения, ни collector не должны отправлять каждый спан как отдельный HTTP/gRPC-запрос. SDK и collector используют `BatchSpanProcessor`: спаны складываются в очередь, периодически пачками отправляются экспортеру. Это снижает накладные расходы и даёт место для backpressure. Но одновременно вводит риск: при очень высоком трафике эта очередь может заполняться, и спаны начнут отбрасываться — это тоже нужно контролировать.

На уровне кода Spring Boot ты обычно не создаёшь collector — он живёт отдельно. Твоя задача — сконфигурировать OpenTelemetry SDK так, чтобы он отправлял трейсы в collector. Ниже простой пример конфигурации Java/Kotlin, где мы вручную настраиваем `SdkTracerProvider` с OTLP exporter’ом (gRPC) на адрес collector’а. В реальном проде чаще используется javaagent, но пример хорошо показывает нижний уровень пайплайна.

**Gradle зависимости для OpenTelemetry SDK и OTLP exporter**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'io.opentelemetry:opentelemetry-api:1.40.0'
    implementation 'io.opentelemetry:opentelemetry-sdk:1.40.0'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.40.0'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("io.opentelemetry:opentelemetry-api:1.40.0")
    implementation("io.opentelemetry:opentelemetry-sdk:1.40.0")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.40.0")
}
```

**Java — конфигурация SDK → OTLP → Collector**

```java
package com.example.tracing.pipeline;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenTelemetryPipelineConfig {

    @PostConstruct
    public void initSdk() {
        SpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .build();

        Resource resource = Resource.getDefault()
                .toBuilder()
                .put(ResourceAttributes.SERVICE_NAME, "orders-service")
                .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "prod")
                .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setResource(resource)
                .addSpanProcessor(
                        BatchSpanProcessor.builder(exporter)
                                .setScheduleDelay(java.time.Duration.ofMillis(500))
                                .setMaxExportBatchSize(512)
                                .setMaxQueueSize(4096)
                                .build()
                )
                .build();

        OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .buildAndRegisterGlobal();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

**Kotlin — тот же пайплайн SDK → OTLP → Collector**

```kotlin
package com.example.tracing.pipeline

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.resources.Resource
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class OpenTelemetryPipelineConfig {

    @PostConstruct
    fun initSdk() {
        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()

        val resource = Resource.getDefault()
            .toBuilder()
            .put(ResourceAttributes.SERVICE_NAME, "orders-service")
            .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "prod")
            .build()

        val tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(
                BatchSpanProcessor.builder(exporter)
                    .setScheduleDelay(Duration.ofMillis(500))
                    .setMaxExportBatchSize(512)
                    .setMaxQueueSize(4096)
                    .build()
            )
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

С таким конфигом приложение отправляет трейсы в collector, а дальше ты уже играешься с receivers/processors/exporters на стороне OTel Collector, не трогая код.

---

## Head vs Tail-sampling: динамика по атрибутам (ошибки/долгие/VIP-клиенты/эндпойнты), бюджет %

Sampling — это единственный способ сделать трейсинг экономически устойчивым в больших системах. Писать каждый запрос, каждую ошибку и каждое событие означает очень быстро убить и хранилище, и сеть, и бюджет. Поэтому приходится выбирать: какие трейсы мы сохраняем, какие выкидываем, и с какой долей. Здесь есть две базовых стратегии: **head-based sampling** и **tail-based sampling**.

Head-based sampling происходит **на входе**, в момент создания первого спана. SDK/агент принимает решение «этот трейс будем писать» или «этот трейс пропускаем» ещё до того, как увидит результат операции. Это дешёво и просто: решение принимает код на стороне приложения, не требует сложной инфраструктуры, хорошо масштабируется. Но есть фундаментальный минус: в момент выбора мы не знаем, будет ли запрос успешным, быстрым или упадёт с ошибкой, а значит, можем случайно выкинуть как раз те трейсы, которые нужны для расследования инцидентов.

Tail-based sampling работает **на выходе**, когда collector видит уже целый трейс: все спаны, статусы, латентность, атрибуты. Он может принять решение: «этот трейс с ошибкой HTTP 500, его обязательно сохраняем», «это очень длинный трейс p99, тоже сохраняем», «это VIP-клиент tenant=gold, держим все запросы», а обычные успешные p50 можно сэмплировать с низкой долей. Так достигается адаптивность: мы экономим объём, но не теряем важные проблемные случаи.

Практически это выглядит так: на стороне приложения используется простой **head-sampler** с фиксированным процентом — например, 10% всех запросов. Это резко снижает нагрузку на сеть и collector. Дальше в collector включается **tail-sampling-процессор**, который из пришедших трейсингов выбирает те, что соответствуют условиям: ошибки, долгие, VIP-арендаторы, важные эндпоинты. Остальные либо выкидываются, либо сэмплируются ещё сильнее. Таким образом, общий бюджет sampling’а можно держать, например, на уровне 1–5%, при этом для ошибок и VIP он становится ≈100%.

Tail-sampling требует **буферизации трейсинга**: collector должен подержать спаны в памяти/кеше, пока не увидит конец трейса и не посчитает агрегаты (статус, суммарную длительность). Это увеличивает потребление памяти и усложняет конфигурацию. Поэтому tail-sampling обычно делают не на «edge-collector’ах», а на центральном gateway-collector’е, который имеет достаточные ресурсы и может эффективно обрабатывать большие объёмы.

С точки зрения правил sampling’а ты обычно задаёшь **набор политик и процентный бюджет**. Пример: «все трейсы с error=true — 100%», «все трейсы с duration>1s — 50%», «все трейсы для endpoint=/api/payment/** — 30%», «остальные — 1%». Collector будет применять правила по очереди, пока не будет достигнут лимит по общему проценту/скорости. Это позволяет сосредоточить внимание на проблемных зонах, не захламляя хранилище бесконечным потоком нормальных запросов.

Head-sampling в коде настраивается выбором `Sampler`: `AlwaysOn`, `AlwaysOff`, `TraceIdRatioBased`, `ParentBased` и их комбинации. Типичный вариант — `ParentBased(TraceIdRatioBased)`: если родительский трейс уже был выбран (например, фронт/шлюз поставил sampled=1), то дочерние сервисы тоже сохраняют свои трейсы; если родителя нет, то используется ratio-based. Это сохраняет целостность сквозных запросов: один раз принял решение — дальше все сервисы его уважают.

Tail-sampling в collector’е реализуется как `tail_sampling`-processor с набором правил по атрибутам и наборам span’ов. Настройки делаются в YAML collector’а и не трогают application-код. Это правильно: логика «какие трейсы нам важны» — инфраструктурная политика, её не стоит хардкодить в сервисах.

Нужно понимать, что sampling — это компромисс: ты **осознанно теряешь данные** ради ресурса. Поэтому для критичных бизнес-потоков (например, платёжный сервис) часто делают sampling «0%» только на нормальные запросы, но «100%» на все ошибки и странные случаи. Это даёт хороший баланс: при инциденте у тебя всегда будут полные трейсы, а в штатном режиме — достаточно наблюдений для анализа паттернов.

Ниже пример Java/Kotlin-конфига, в котором мы вручную настраиваем head-based sampling на уровне SDK: parent-based с ratio 0.1 (10%). Это иллюстрация того, как код «участвует» в sampling-пайплайне. Tail-sampling для collector’а будет уже YAML’ом, его можно показать отдельно.

**Java — настройка head-based sampling в SDK**

```java
package com.example.tracing.sampling;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.Sampler;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HeadSamplingConfig {

    @PostConstruct
    public void initSampling() {
        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .build();

        Resource resource = Resource.getDefault()
                .toBuilder()
                .put(ResourceAttributes.SERVICE_NAME, "checkout-service")
                .build();

        Sampler sampler = Sampler.parentBased(
                Sampler.traceIdRatioBased(0.1) // 10% новых трасс
        );

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setResource(resource)
                .setSampler(sampler)
                .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
                .build();

        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

**Kotlin — то же самое head-sampling**

```kotlin
package com.example.tracing.sampling

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.resources.Resource
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.Sampler
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration

@Configuration
class HeadSamplingConfig {

    @PostConstruct
    fun initSampling() {
        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()

        val resource = Resource.getDefault()
            .toBuilder()
            .put(ResourceAttributes.SERVICE_NAME, "checkout-service")
            .build()

        val sampler: Sampler = Sampler.parentBased(
            Sampler.traceIdRatioBased(0.1) // 10% новых трасс
        )

        val tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .setSampler(sampler)
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

На стороне collector’а ты потом добавишь tail-sampling в YAML, написав правила типа «sample by status/error, duration, tenant», но приложение останется неизменным.

---

## Лимиты: размеры спана, количество атрибутов/событий/линков; батч-экспорт и бэкпрешр

Трейсинг — это поток данных, и если его никак не ограничивать, он легко превратится в DoS на саму систему наблюдаемости. Каждый спан — это набор атрибутов, событий, линков, stacktrace’ов и т.д. У каждого есть размер, а у pipeline’а — пропускная способность. Поэтому у OpenTelemetry SDK и Collector’а есть несколько слоёв лимитов: на атрибуты, на события, на длину строк, на размер очередей, на размер батча и время экспорта.

На уровне спана есть понятие **attribute limits**: максимальное количество атрибутов, длина ключей/значений, количество событий и линков. Если ты начнёшь пихать в атрибут каждое поле DTO или огромные JSON’ы, SDK будет либо урезать, либо выкидывать лишнее. Это защита от случайных (или сознательных) злоупотреблений. Часть лимитов настраивается в SDK-конфиге, часть — в Collector’е (через `attributes`/`transform` processors).

Дальше идёт **BatchSpanProcessor**, который контролирует очередь и размер батчей. Основные параметры: `maxQueueSize` (сколько спанов можно накопить перед экспортом), `maxExportBatchSize` (сколько спанов отправляется за раз), `scheduleDelay` (как часто экспорт происходит таймером) и `exportTimeout`. Если приложению генерирует спаны быстрее, чем exporter успевает отправить, очередь заполняется, и SDK начинает выкидывать самые старые/new спаны, в зависимости от реализации. Это и есть built-in backpressure: лучше потерять часть трейсинга, чем положить сервис.

Collector добавляет ещё один уровень: **batch processor** и **memory_limiter**. Batch-процессор собирает входящие спаны в батчи, обеспечивает эффективный экспорт и может добавлять небольшой «джиттер» по времени. Memory_limiter следит за тем, чтобы сам collector не вышел за пределы памяти: при превышении лимитов новые данные могут временно дропаться, а экспортеры — замедляться. Это очередной компромисс: сохраняем живой мониторинг ценой частичной потери данных.

Лимиты нужны не только чтобы не «убить» инфраструктуру, но и чтобы защитить ядро приложения. Если exporter завис или хранилище недоступно, SDK не должен бесконечно накапливать спаны в памяти. Отсюда важный вывод: **трейсинг не должен влиять на функциональность**. Любая ошибка экспорта должна логироваться и вести к отбрасыванию данных, а не к падению сервиса. Кроме того, задержки в exporter’е не должны блокировать бизнес-логики — именно поэтому используется асинхронный batch-процессор и очередь.

Отдельная тема — лимиты по **количеству событий и exception’ов**. `Span.recordException` может включать stacktrace, который занимает ощутимый объём. Если каждый этап pipeline’а будет логировать одно и то же исключение, общий размер трейса вырастет в разы. Поэтому есть смысл либо ограничивать количество `recordException` на один трейс, либо на уровне collector’а фильтровать повторяющиеся исключения по типу/сообщению.

В промышленных системах нередко включают ещё и **rate limiting** на уровне Collector’а или перед ним. Например, «не принимать больше N спанов в секунду от одного сервиса». Это выглядит жёстко, но защищает от runaway-сценариев, когда один плохо настроенный сервис начинает генерировать триллионы спанов из-за цикла или рекурсивной ретраировки. В идеале такие сценарии ловятся на деве, но в реальности всегда может уползти что-то в прод.

Практический приём — согласовать лимиты SDK и Collector’а. Если SDK умеет держать 4k спанов в очереди и отправлять по 512 спанов в батче раз в 200–500 мс, то collector должен быть настроен так, чтобы успевать проглатывать такой поток с небольшим запасом. Если collector тормозит, это сразу видно по метрикам экспорта (ошибки, задержки) и по count отброшенных спанов. Поэтому в observability-дашбордах должны быть метрики **самого pipeline’а трейсинга**, а не только бизнес-метрики.

Ниже пример Java/Kotlin-конфига, где мы явно задаём лимиты для `BatchSpanProcessor`: размер очереди, размер батча и таймаут экспорта. Это иллюстрирует, как на уровне SDK ты управляешь backpressure и памятью, не давая трейсингу «выпереть» сам сервис.

**Java — настройка BatchSpanProcessor и лимитов очереди**

```java
package com.example.tracing.limits;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class BatchLimitsConfig {

    @PostConstruct
    public void initBatchProcessor() {
        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .build();

        BatchSpanProcessor spanProcessor = BatchSpanProcessor.builder(exporter)
                .setMaxQueueSize(2000)                 // не больше 2000 спанов в очереди
                .setMaxExportBatchSize(256)            // экспортировать по 256 спанов
                .setScheduleDelay(Duration.ofMillis(500)) // не реже, чем раз в 500 мс
                .setExporterTimeout(Duration.ofSeconds(5)) // не ждать экспортер дольше 5 секунд
                .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .addSpanProcessor(spanProcessor)
                .build();

        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

**Kotlin — те же лимиты BatchSpanProcessor**

```kotlin
package com.example.tracing.limits

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class BatchLimitsConfig {

    @PostConstruct
    fun initBatchProcessor() {
        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()

        val spanProcessor = BatchSpanProcessor.builder(exporter)
            .setMaxQueueSize(2000)                     // не больше 2000 спанов в очереди
            .setMaxExportBatchSize(256)                // экспорт по 256 спанов
            .setScheduleDelay(Duration.ofMillis(500))  // таймер экспорта
            .setExporterTimeout(Duration.ofSeconds(5)) // таймаут экспорта
            .build()

        val tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(spanProcessor)
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

С такими настройками трейсинг перестаёт быть «безграничным»: если collector не успевает, SDK начнёт отбрасывать спаны, но не будет ломать приложение.

---

## Безопасность: фильтрация/редакция PII, маппинг ресурсных атрибутов (service.name, env, region, pod)

Трейсы — это практически «рентген» приложения: параметры запросов, идентификаторы пользователей, названия эндпоинтов, доменные атрибуты. Если не думать о безопасности, туда легко утечёт PII (персональные данные), платёжные данные, токены, внутренние секреты. Поэтому security в трейсинге — не «дополнительная опция», а обязательная часть дизайна пайплайна: что мы собираем, что фильтруем, что маскируем, где храним, кто имеет доступ.

Первая линия обороны — **на уровне кода и SDK**. Здесь ты решаешь, какие атрибуты вообще добавлять к спанам. Простое правило: **не класть в атрибуты то, что нельзя безопасно сохранить и показать диагностам**. PII (ФИО, email, телефон), токены, номера карт, содержимое запросов — всё это должно быть либо маскировано, либо полностью исключено. Для таких вещей лучше использовать логи с отдельной политикой retention’а и доступов, а в атрибуты класть только технические и агрегируемые идентификаторы (internal ID, tenant, тип операции).

Вторая линия — **фильтрация/редакция в Collector’е**. Здесь в ход идут `attributes`/`filter` processors: можно удалить или замаскировать конкретные атрибуты (`http.request.header.authorization`, `user.email`), отфильтровать целые трейсы по признакам (например, не отправлять тэсы из dev-окружения в продовый бэкенд), обрезать строки до заданной длины. Это удобно тем, что правится централизованно и может подстраиваться под compliance-требования без перекомпиляции сервисов.

Отдельная тема — **resource attributes**: `service.name`, `service.namespace`, `service.instance.id`, `deployment.environment`, `cloud.region`, `k8s.pod.name` и т.д. Это метаданные о том, откуда пришёл спан. Их нужно заполнять максимально аккуратно и стандартизировать. Если каждый сервис сам называет себя «order-service», «orderservice», «order-svc», трейсинг быстро превращается в хаос. Хорошая практика — жёсткий стандарт: `service.name` = `company-domain-serviceName`, `deployment.environment` = `dev|test|stage|prod`, `service.namespace` = продукт/домейн.

Resource-атрибуты также участвуют в security: по ним строятся **права доступа и tenant-изоляция** в бэкенде трейсинга. Например, SRE-платформы видят все `service.namespace=*`, а команды продукта — только своё пространство. Поэтому нужно следить, чтобы туда не утекали PII, а только технические идентификаторы окружения и сервиса. В Collector’е часто добавляют/переписывают resource-атрибуты через `resource`-processor (например, подтягивают `env` из k8s-annotations).

Важный момент — **маскирование чувствительных данных**: вместо того, чтобы вообще не иметь информации, лучше иметь обезличенную. Например, вместо `user.email` можно писать `user.id_hash` или `user.segment`. В трейсинге часто достаточно знать, что это один и тот же пользователь в серии трейсов, а не знать его реальные данные. Хэширование или псевдонимизация ID — хороший компромисс между диагностикой и приватностью.

Третья линия — **управление доступом к хранилищу трейсинга**. Tempo/Jaeger/ELK/Loki должны жить за SSO и RBAC, а не быть открытыми по HTTP без аутентификации. Тут уже речь не про код, а про DevOps/безопасников: кто может смотреть трейсы прод-сервисов, есть ли разделение PROD/NON-PROD, ведётся ли аудит доступа. С точки зрения разработчика важно понимать, что всё, что ты кладёшь в трейс, теоретически увидит любой, у кого есть доступ к трейс-вьюеру.

На кодовом уровне удобно реализовать **SpanProcessor-фильтр**, который перед экспортом чистит спаны: удаляет запрещённые атрибуты, маскирует шаблонно некоторые значения, обрезает длину строк. Это дополняет фильтрацию на уровне Collector’а и защищает на случай, если часть сервисов отправляет трейсы напрямую в бэкенд, минуя общий collector. В Java/Kotlin это делается через `SpanProcessor`/`SpanExporter`-обёртки.

Наконец, хорошая практика — иметь **чек-лист по безопасности трейсинга**: «мы не логируем auth header, не кладём токены в атрибуты, маскируем user-id, resource-атрибуты стандартизованы, доступ по SSO, retention у трейс-хранилища ограничен (например, 7–14 дней)». Без этого трейсинг легко превращается в новую зону утечек, которую никто не мониторит с точки зрения compliance.

Ниже пример Java/Kotlin SpanProcessor’а, который на лету проходит по атрибутам и удаляет/маскирует набор ключей: Authorization, cookies, некоторые user.* поля. В реальном проекте лучше часть этой логики вынести в Collector, но этот пример показывает, как сделать фильтрацию прямо в приложении.

**Java — SpanProcessor для редактирования чувствительных атрибутов**

```java
package com.example.tracing.security;

import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.AttributesBuilder;
import io.opentelemetry.sdk.trace.ReadWriteSpan;
import io.opentelemetry.sdk.trace.ReadableSpan;
import io.opentelemetry.sdk.trace.SpanProcessor;
import org.jetbrains.annotations.NotNull;

import java.util.Set;

public class RedactingSpanProcessor implements SpanProcessor {

    private static final Set<String> SENSITIVE_KEYS = Set.of(
            "http.request.header.authorization",
            "http.request.header.cookie",
            "user.email",
            "user.phone"
    );

    @Override
    public void onStart(@NotNull ReadWriteSpan span) {
        AttributesBuilder builder = span.toBuilder().getAttributes().toBuilder();

        span.toBuilder().getAttributes().forEach((key, value) -> {
            if (SENSITIVE_KEYS.contains(key.getKey())) {
                builder.put((AttributeKey<String>) key, "[REDACTED]");
            }
        });

        span.setAllAttributes(builder.build());
    }

    @Override
    public void onEnd(@NotNull ReadableSpan span) {
        // Ничего не делаем при завершении
    }

    @Override
    public boolean isStartRequired() {
        return true;
    }

    @Override
    public boolean isEndRequired() {
        return false;
    }
}
```

И регистрация этого processor’а в конфиге SDK:

```java
package com.example.tracing.security;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SecurityTracingConfig {

    @PostConstruct
    public void initSecureTracing() {
        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .addSpanProcessor(new RedactingSpanProcessor())
                .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
                .build();

        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

**Kotlin — тот же RedactingSpanProcessor**

```kotlin
package com.example.tracing.security

import io.opentelemetry.api.common.AttributeKey
import io.opentelemetry.api.common.AttributesBuilder
import io.opentelemetry.sdk.trace.ReadWriteSpan
import io.opentelemetry.sdk.trace.ReadableSpan
import io.opentelemetry.sdk.trace.SpanProcessor

class RedactingSpanProcessor : SpanProcessor {

    private val sensitiveKeys = setOf(
        "http.request.header.authorization",
        "http.request.header.cookie",
        "user.email",
        "user.phone"
    )

    override fun onStart(span: ReadWriteSpan) {
        val builder: AttributesBuilder = span.toBuilder().attributes.toBuilder()

        span.toBuilder().attributes.forEach { key, _ ->
            if (sensitiveKeys.contains(key.key)) {
                @Suppress("UNCHECKED_CAST")
                val stringKey = key as AttributeKey<String>
                builder.put(stringKey, "[REDACTED]")
            }
        }

        span.setAllAttributes(builder.build())
    }

    override fun onEnd(span: ReadableSpan) {
        // Ничего не делаем при завершении
    }

    override fun isStartRequired(): Boolean = true

    override fun isEndRequired(): Boolean = false
}
```

И регистрация в конфиге:

```kotlin
package com.example.tracing.security

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration

@Configuration
class SecurityTracingConfig {

    @PostConstruct
    fun initSecureTracing() {
        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()

        val tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(RedactingSpanProcessor())
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

Такой подход не отменяет фильтрацию в Collector’е, но добавляет дополнительный «safety net» на уровне сервисов.

# 9. Связь метрик ↔ трейсев ↔ логов

## Exemplars в Prometheus/Grafana: прокладка «из p99 на конкретный trace»

Когда у тебя уже есть метрики, трейсы и логи, главный вопрос не «как всё это собирать», а **как быстро пройти путь от симптома к причине**. Типичный симптом — график latency: p95/p99 какого-то эндпоинта внезапно вырос. Сама по себе метрика говорит лишь «стало хуже», но не показывает, **какой именно запрос** был медленным. Для этого и существуют *exemplars* — отдельные, помеченные измерения внутри метрики, которые содержат ссылку на конкретный traceId, позволящую кликнуть из пика p99 сразу в трейс.

Exemplar в Prometheus — это по сути «точка» внутри гистограммы/таймера, у которой кроме значения и времени есть дополнительные лейблы, в простейшем случае `trace_id` (и иногда `span_id`). На уровне хранения это просто ещё одна часть сэмпла, но на уровне Grafana — возможность: наводишь на столбец гистограммы с p99, видишь иконку, кликаешь — и Grafana открывает трейс в Tempo/Jaeger по этому traceId. Переход становится однокликовым, а не «открыть отдельный трейс UI и гадать, какой трейс искать».

В мире Spring Boot и Micrometer exemplars не нужно «рисовать руками» на каждой метрике — они появляются как побочный эффект связки **Observation → Tracing → Metrics**. Когда ты записываешь таймер через `Observation` (или `@Observed`/`@Timed`), и при этом включён tracing (Micrometer Tracing + OTel), Micrometer умеет выбирать отдельные измерения и снабжать их ссылкой на текущий span. В Prometheus это превращается в exemplar, а в Tempo/Grafana — в кликабельную стрелку от метрики к трассе.

Важный момент в дизайне: **не все метрики нужны с exemplars**. На практике ты хочешь их на таймерах, которые описывают внешние и пользовательские запросы: HTTP server, WebClient, вызовы БД, очередей. Таймеры внутренней логики, счётчики «сколько раз зашёл в if», гейджи для debugging — всё это в exemplars не нуждается. Если размечать всё подряд, ты увеличишь нагрузку на цепочку метрик и усложнишь UI, но не выиграешь в скорости диагностики.

Экономика exemplars гораздо мягче, чем у трейсов: ты не пишешь *все* запросы с ссылками, а только выборочные сэмплы. Обычно Micrometer выбирает exemplars для тех измерений, у которых есть активный span и которые прошли sampling. Даже при head-sampling 10% и небольшом количестве exemplars этого достаточно, чтобы «нащупать» характерный трейс для проблемного пика. Тебе не нужно иметь 100% соответствие, нужно иметь достаточно примеров.

Есть и подводные камни. Если у тебя много слоёв агрегации (прокси, gateway, sidecar), exemplar может «потеряться», если span контекст теряется по пути или метрика пишет значения вне этого контекста. Поэтому критично, чтобы **таймеры HTTP-запросов писались внутри активного span**, а не в отдельном техническом потоке без контекста. Иначе exemplar будет без traceId, и переход не сработает. Точно так же, если ты самостоятельно пишешь метрику без участия Observation/Tracing, expect exemplars не будет.

Практически в Spring Boot 3.x всё сводится к тому, чтобы: включить Actuator + Prometheus, подключить Micrometer Tracing + OTel, экспонировать `/actuator/prometheus` и настроить Grafana/Tempo. После этого HTTP-метрики `http_server_requests_seconds` (или их эквивалент) уже будут иметь exemplars для части сэмплов. Нужно лишь подключить Tempo как источник трейсинга в Grafana и настроить «метрика → трейс» в панели. Код приложения при этом остаётся относительно простым.

В повседневных расследованиях exemplars экономят реально много времени. Вместо подхода «видим, что p99 вырос, смотрим на логи, пытаемся угадать, какие запросы были медленными» ты просто кликаешь по exemplar’у, видишь трейс, в нём — дерево спанов, видишь, что именно тормозит, а дальше уже по spanId уходишь в логи. Особенно это ценно, когда нагрузка большая, и логов очень много: подобрать подходящий traceId «на глаз» становится невозможно.

С точки зрения практики, хорошая стратегия — **сначала настроить базовый трейсинг и метрики**, затем добавить exemplars, и только потом усложнять конфигурацию sampling’а. Если попытаться сразу сделать tail-sampling, сложные правила и множественные backend’ы, легко запутаться. Начни с простого: head-sampling 10%, exemplars на HTTP и паре ключевых таймеров, Tempo + Grafana, а затем по мере взросления системы усложняй pipeline.

В конечном итоге exemplars — это не «магия», а просто аккуратная связка traceId со значениями метрик. Если ты держишь трейсинг и метрики в одной экосистеме (Micrometer + OTel + Prometheus/Tempo), настройка становится главным образом инфраструктурной задачей. На уровне кода тебе достаточно использовать `Observation`/`@Observed`/`@Timed`, а весь «мост» от p99 до конкретного trace будет реализован инфраструктурой.

Ниже — упрощённый пример Java/Kotlin-кода, показывающий, как в Spring Boot 3 использовать `@Observed` для бизнес-метода и тем самым дать Micrometer достаточно информации, чтобы построить и спан, и таймер с exemplars. При наличии настроенного Prometheus и трейсинга Grafana сможет кликнуть из метрики в конкретную трассу.

**Gradle зависимости (минимум для метрик + трейсинга + Prometheus)**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-tracing-bridge-otel'
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

**Java — бизнес-метод под `@Observed` (таймер + спан → exemplars)**

```java
package com.example.observability.exemplars;

import io.micrometer.observation.annotation.Observed;
import org.springframework.stereotype.Service;

@Service
public class CheckoutService {

    @Observed(
            name = "checkout.process",
            contextualName = "checkout-process",
            lowCardinalityKeyValues = {"operation", "checkout"}
    )
    public void processCheckout(String orderId) {
        // Здесь будет создан спан и таймер,
        // Micrometer свяжет их, и в Prometheus появятся exemplars.
        doInventoryCheck(orderId);
        doPayment(orderId);
    }

    private void doInventoryCheck(String orderId) {
        // Проверка наличия
    }

    private void doPayment(String orderId) {
        // Оплата
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.exemplars

import io.micrometer.observation.annotation.Observed
import org.springframework.stereotype.Service

@Service
class CheckoutService {

    @Observed(
        name = "checkout.process",
        contextualName = "checkout-process",
        lowCardinalityKeyValues = ["operation", "checkout"]
    )
    fun processCheckout(orderId: String) {
        // Будет создан span и таймер; при включенном трейсинге
        // Prometheus получит exemplars с traceId.
        doInventoryCheck(orderId)
        doPayment(orderId)
    }

    private fun doInventoryCheck(orderId: String) {
        // Проверка наличия
    }

    private fun doPayment(orderId: String) {
        // Оплата
    }
}
```

---

## Span metrics: агрегация трасс в метрики (RPS/latency/error_rate), где это уместно

Метрики и трейсы можно собирать независимо: метрики — через Micrometer, трейсинг — через OTel. Но иногда полезно **получать метрики «из» трейсинга**. Это называется *span metrics*: когда мы берём поток спанов, агрегируем их по определённым правилам и получаем знакомые числа: RPS, latency, error rate, distribution по статусам. Преимущество — тебе не нужно отдельно инструментировать каждый HTTP handler таймером, если трейсинг и так уже всё видит.

Span metrics особенно уместны там, где трейсинг уже настроен и стандартизирован: например, во входящем HTTP-трафике, gRPC, запросах к БД и брокерам. OpenTelemetry Collector умеет автоматически превращать часть спанов в метрики (через соответствующий processor), но можно делать это и внутри приложения, если хочется гимнастики или нужна специфичная логика, завязанная на бизнес-атрибуты. Однако общий принцип одинаков: **каждый завершённый спан — это ещё одно наблюдение для метрики**.

В отличие от прямого Micrometer-инструментирования, span metrics имеют одну важную особенность: они зависят от sampling’а. Если ты пишешь только 10% трейсов, и на их основе считаешь RPS, то либо ты соглашаешься на «относительные» значения, либо должен масштабировать результат (умножать на 10), либо держать sampling для конкретного набора спанов = 100%. Поэтому span metrics чаще используют для **относительной** диагностики (динамика, распределения, доля ошибок), а не для «абсолютного» биллинга запросов.

Там, где sampling невелик или отсутствует, span metrics могут частично заменить ручные таймеры. Например, collector может взять все спаны с семантикой `http.server`, сгруппировать их по атрибутам `http.route`, `http.method`, `http.status_code` и построить метрики latency/error rate автоматически. В таком случае приложение вообще не знает о метриках HTTP — за него всё делает трейсинг и инфраструктура. Это упрощает код, но переносит ответственность за конфиг на платформенную команду.

В Spring-приложении иногда хочется сделать что-то подобное локально: например, считать метрику «сколько бизнес-операций `order.process` завершились с ошибкой» не из таймеров, а из спанов. Для этого можно написать **SpanProcessor**, который при завершении спана смотрит на его имя/атрибуты/статус и увеличивает нужные счётчики/таймеры Micrometer. Это гибкий, но довольно низкоуровневый подход; использовать его стоит аккуратно, чтобы не удвоить метрики, которые уже считает Observations.

Важный момент — **кардинальность** при агрегировании span metrics. Если ты будешь тегировать метрику по `span.name`, `http.route`, `user.id`, `request.id`, `correlation.id`, то получишь миллионы time-series. Поэтому при маппинге спана в метрику нужно очень аккуратно выбирать набор лейблов: только стабильные и агрегируемые: route, метод, статус, tenant (если их немного), тип операции. Точно не стоит тащить туда requestId, e-mail и аналогичные поля.

Практически span metrics стоит использовать там, где: а) у тебя уже есть хороший, единый трейсинг; б) шаблон спанов стабилен; в) нужно быстро получать SLI-метрики по всем сервисам без правки кода. Для специфичных бизнес-метрик и для случаев с неочевидной семантикой всё равно проще и надёжнее использовать явные Micrometer-инструменты (`@Timed`, `Timer`, `Counter`) в коде.

Ниже — пример Java/Kotlin `SpanProcessor`, который на стороне приложения смотрит на завершённые спаны и обновляет Micrometer-метрики: считает количество успешных/ошибочных бизнес-операций по имени спана и измеряет длительность. В реальности ты, вероятно, сделаешь это на уровне OTel Collector’а, но пример показывает принцип.

**Java — SpanProcessor, агрегирующий спаны в Micrometer-метрики**

```java
package com.example.observability.spanmetrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.trace.ReadableSpan;
import io.opentelemetry.sdk.trace.SpanProcessor;
import org.jetbrains.annotations.NotNull;

import java.time.Duration;

public class SpanMetricsProcessor implements SpanProcessor {

    private final MeterRegistry meterRegistry;

    public SpanMetricsProcessor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void onStart(@NotNull io.opentelemetry.sdk.trace.ReadWriteSpan span) {
        // Ничего не делаем на старте
    }

    @Override
    public void onEnd(@NotNull ReadableSpan span) {
        String spanName = span.getName();
        boolean error = span.getStatus().getStatusCode().isError();

        Timer timer = Timer.builder("span.duration")
                .tag("span.name", spanName)
                .tag("status", error ? "error" : "ok")
                .register(meterRegistry);

        long durationNanos = span.getLatencyNanos();
        timer.record(Duration.ofNanos(durationNanos));

        meterRegistry.counter(
                        "span.count",
                        "span.name", spanName,
                        "status", error ? "error" : "ok"
                )
                .increment();
    }

    @Override
    public boolean isStartRequired() {
        return false;
    }

    @Override
    public boolean isEndRequired() {
        return true;
    }
}
```

Регистрация процессора в конфиге SDK (упрощённо):

```java
package com.example.observability.spanmetrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SpanMetricsConfig {

    private final MeterRegistry meterRegistry;

    public SpanMetricsConfig(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @PostConstruct
    public void initSpanMetrics() {
        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .build();

        SpanMetricsProcessor spanMetricsProcessor = new SpanMetricsProcessor(meterRegistry);

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .addSpanProcessor(spanMetricsProcessor)
                .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
                .build();

        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

**Kotlin — аналогичный SpanProcessor для Micrometer-метрик**

```kotlin
package com.example.observability.spanmetrics

import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Timer
import io.opentelemetry.sdk.trace.ReadWriteSpan
import io.opentelemetry.sdk.trace.ReadableSpan
import io.opentelemetry.sdk.trace.SpanProcessor
import java.time.Duration

class SpanMetricsProcessor(
    private val meterRegistry: MeterRegistry
) : SpanProcessor {

    override fun onStart(span: ReadWriteSpan) {
        // Ничего не делаем на старте
    }

    override fun onEnd(span: ReadableSpan) {
        val spanName = span.name
        val error = span.status.statusCode.isError

        val timer: Timer = Timer.builder("span.duration")
            .tag("span.name", spanName)
            .tag("status", if (error) "error" else "ok")
            .register(meterRegistry)

        val durationNanos = span.latencyNanos
        timer.record(Duration.ofNanos(durationNanos))

        meterRegistry.counter(
            "span.count",
            "span.name", spanName,
            "status", if (error) "error" else "ok"
        ).increment()
    }

    override fun isStartRequired(): Boolean = false

    override fun isEndRequired(): Boolean = true
}
```

И регистрация:

```kotlin
package com.example.observability.spanmetrics

import io.micrometer.core.instrument.MeterRegistry
import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration

@Configuration
class SpanMetricsConfig(
    private val meterRegistry: MeterRegistry
) {

    @PostConstruct
    fun initSpanMetrics() {
        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()

        val spanMetricsProcessor = SpanMetricsProcessor(meterRegistry)

        val tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(spanMetricsProcessor)
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

---

## Корреляция логов: traceId/spanId в MDC, структурированные логи и sampling для шумных сервисов

Если метрики говорят «где болит», а трейсы показывают «как прошёл запрос», то логи отвечают на вопрос «что именно происходило в нужный момент». Чтобы не искать вручную нужные строки в тоннах логов, нужен общий ключ, связывающий их с трейсами. В observability этот ключ — **traceId и spanId**. Если они есть и в спанах, и в логах, ты всегда можешь взять traceId из трейс-вьюера, найти все логи по нему и получить контекст инцидента.

Технически связка строится через **MDC (Mapped Diagnostic Context)** в SLF4J/Logback. MDC — это локальный для потока map `ключ → значение`, который логгер автоматически подставляет в шаблон. Если при обработке запроса в MDC лежат `traceId` и `spanId`, ты можешь добавить в pattern `%X{traceId}` и `%X{spanId}` или включить их в JSON-поле. Тогда любая строка лога, написанная в контексте этого запроса, будет иметь те же значения traceId/spanId, что и спаны.

В Spring Boot 3 с Micrometer Tracing и OTel это в основном делается автоматически: текущий span из трейсинга транслируется в MDC, и дальше всё зависит от твоего лог-паттерна. Но иногда приходится работать руками: в асинхронном коде без правильного переноса контекста MDC может оказаться пустым, или тебе нужно проставить дополнительные поля, вроде `tenantId` или `requestId`. Тогда ты обращаешься к текущему span (через OTel API или Micrometer Tracer) и сам кладёшь IDs в MDC.

Структурированные логи (JSON) усиливают эффект. Вместо того чтобы парсить строку руками, ты получаешь JSON c полями `traceId`, `spanId`, `level`, `message`, `logger`, `tenantId`, `operation`. Любая нормальная система логов умеет по этим полям фильтровать, группировать и строить графики. Тогда workflow становится простым: смотришь трейс, копируешь traceId, вставляешь в лог UI — и получаешь все связанные строки. Или наоборот: нашёл интересный лог, взял traceId, открыл трейс в Tempo.

Отдельный вопрос — **лог-сэмплинг**. Если сервис шумный (много debug/trace для внутренней логики), логировать всё подряд нельзя — просто взорвёшь объём. Тогда обычно делают: а) `INFO` минимален; б) `DEBUG` включают только по фиче-флагу или для части запросов; в) используют rate-limited logging (ограничение сообщений в единицу времени); г) лог-сэмплинг по traceId (например, один из N traceId логируем подробно, остальные — только ошибки). В любом из этих вариантов критично, чтобы ошибки (`WARN`/`ERROR`) всегда логировались с traceId.

Картина усложняется в асинхронном и реактивном коде. MDC по умолчанию привязан к потоку, а не к «логическому запросу». В `@Async`/`CompletableFuture`/Reactor без специальных декораторов контекст легко теряется: в worker-пуле нет traceId/spanId, и логи становятся «анонимными». Поэтому в Spring мире используют `TaskDecorator`/`DelegatingSecurityContextAsyncTaskExecutor` для переноса контекста, а в реактивном мире — `Context` и интеграцию Micrometer Tracing с Reactor. Это нужно, чтобы трейсинг и логи оставались связанными по всей цепочке вызовов.

Практически полезно договориться о **единых именах полей** в логах: `traceId`, `spanId`, `tenantId`, `requestId`. Тогда и разработчики, и SRE, и BI знают, по чему искать, и могут строить кросс-источниковые запросы: «найти все логи с этой ошибкой и этим tenantId, сгруппировать по traceId». Если каждый сервис использует свои имена (`corrId`, `x_trace_id`, `rid`), общая аналитика становится мучением.

Еще один аспект — **privacy**: traceId/spanId сами по себе не PII, но то, к чему они привязаны, может быть чувствительным. Поэтому важно не перепутать: traceId — это технический идентификатор, его можно смело использовать как «главный ключ» в логах, а вот payload’ы запросов, PII и токены нужно или выкидывать, или маскировать. Корреляция по traceId позволяет держать логи достаточно скудными по контенту, но богатыми по связи с трейсами.

Ниже — пример Java/Kotlin-сервиса, который вручную берет текущий span, извлекает traceId/spanId и помещает их в MDC перед логированием. В реальном Spring Boot 3 это часто делает за тебя инфраструктура, но пример показывает, как эта связь выглядит на уровне кода и что ты можешь сделать в кастомных местах (например, в низкоуровневом клиенте или в отдельном worker’е).

**Java — ручная корреляция логов с traceId/spanId через MDC**

```java
package com.example.observability.logs;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;

@Service
public class LoggingCorrelationService {

    private static final Logger log = LoggerFactory.getLogger(LoggingCorrelationService.class);

    public void handleBusinessOperation(String operationId) {
        Span currentSpan = Span.current();
        SpanContext context = currentSpan.getSpanContext();

        if (context.isValid()) {
            MDC.put("traceId", context.getTraceId());
            MDC.put("spanId", context.getSpanId());
        }

        try {
            log.info("Handling business operation {}", operationId);
            // Бизнес-логика
        } finally {
            MDC.remove("traceId");
            MDC.remove("spanId");
        }
    }
}
```

**Kotlin — тот же пример с MDC и traceId/spanId**

```kotlin
package com.example.observability.logs

import io.opentelemetry.api.trace.Span
import org.slf4j.LoggerFactory
import org.slf4j.MDC
import org.springframework.stereotype.Service

@Service
class LoggingCorrelationService {

    private val log = LoggerFactory.getLogger(LoggingCorrelationService::class.java)

    fun handleBusinessOperation(operationId: String) {
        val span = Span.current()
        val context = span.spanContext

        if (context.isValid) {
            MDC.put("traceId", context.traceId)
            MDC.put("spanId", context.spanId)
        }

        try {
            log.info("Handling business operation {}", operationId)
            // Бизнес-логика
        } finally {
            MDC.remove("traceId")
            MDC.remove("spanId")
        }
    }
}
```

С таким подходом любая строка лога внутри `handleBusinessOperation` будет содержать traceId/spanId, и ты сможешь легко связать её с нужным трейсом в Tempo/Jaeger.

---

## Диагностика инцидентов: сценарии «медленный эндпоинт», «шторм ретраев», «утечка кардинальности»

Когда observability уже настроена, вся ценность проявляется в момент инцидента. В спокойное время графики красивые, дашборды зелёные, трейсы бегут сами по себе. Настоящая проверка — это «ночной» или «праздничный» инцидент, когда нужно быстро понять, что происходит и где копать. В таких ситуациях связка метрик, трейсинга и логов даёт возможность двигаться по понятному сценарию, а не вслепую. Рассмотрим три типичных кейса: медленный эндпоинт, шторм ретраев и утечка кардинальности.

Сценарий «медленный эндпоинт» начинается с метрик. Ты или алёрт видишь, что p95/p99 `http_server_requests` для конкретного маршрута выросли, например, с 200 мс до 2–3 секунд. Сначала проверяется общая нагрузка, ошибки, состояние GC и пулов — чтобы исключить инфраструктурные проблемы вроде saturations CPU/heap или исчерпания соединений. Если всё выглядит нормально, следующий шаг — экземплары: кликаешь по пику p99, попадаешь в конкретный трейс того самого медленного запроса.

В трейс-вьюере ты видишь корневой HTTP SERVER span, а внутри него — дочерние спаны: WebClient-вызовы, обращения к БД, кэшам, очередям. Один из этих спанов явно «торчит» по длительности: например, внешнее API отвечает 1.8 секунды вместо 80 мс. Теперь понятно, что сам сервис здоров, а тормозит конкретная зависимость. Отсюда два пути: временно деградировать (например, fallback к кэшу) и параллельно разбираться, что происходит с зависимостью. Логи по traceId дополнительно покажут, какой именно запрос ушёл вовне и какая ошибка или задержка была.

Сценарий «шторм ретраев» обычно проявляется в метриках как рост RPS, ошибок и нагрузок на внешние сервисы. Ты видишь, что error rate вырос, RPS также вырос, а зависимость от «payment-service» или «orders-api» стала ещё более загруженной. Из трейсинга видно, что каждый запрос к твоему сервису порождает несколько клиентских спанов к внешнему API: `payment.call`, `payment.call` снова, третий раз и т.д. По атрибуту `retry.attempt` или по структуре трейсинга понимаешь, что включился механизм ретраев, и он усиливает нагрузку.

Логи дополняют: видно повторы одной и той же ошибки, например `connect timeout` или `503` от внешнего сервиса, и попытки Resilience4j или кастомного retry-политики обратиться ещё раз. Здесь связка метрики ↔ трейсы ↔ логи помогает отличить «шторм из-за клиентов» от «шторм из-за наших ретраев». Часто оказывается, что ретраи настроены слишком агрессивно: много попыток, короткий backoff, нет верхнего лимита времени. Из метрик ты видишь нагрузку и пиковые значения, из трейсинга — структуру ретраев, из логов — детали по ошибкам и HTTP-кодам.

Сценарий «утечка кардинальности» проявляется в метриках как взрыв количества time-series. Prometheus начинает ругаться на лимиты, дашборды тормозят, алёрты говорят о росте числа лейблов. Сначала смотришь на метрики `prometheus_tsdb_head_series` или аналогичные показатели от remote_write/Thanos и понимаешь, что séries стало на порядок больше. Дальше ищешь, какая метрика больше всего «раздулась»: либо через Prometheus-консоль, либо через internal-метрики самого Prometheus.

Когда метрика найдена, оказывается, что она тегирована по чему-то слишком детальному: `user.id`, `request.id`, `session.id`, `product.code`. Трейсинг здесь помогает тем, что ты можешь посмотреть на span attributes и увидеть, что именно из доменной части внёс high-cardinality. Например, кто-то решил добавить `user.id` в `lowCardinalityKeyValues` `@Observed`, и этот ключ ушёл и в спаны, и в метрики. Логи подтверждают это: в них видно, что логгер для этой операции пишет много разных значений для этого же ключа.

Диагностика и исправление этого кейса строится как мини-инцидент: сначала масштаб проблемы через метрики (какие метрики и сколько серий), потом поиск источника в коде (какие `@Observed`/таймеры/SpanProcessor добавляют лейблы), затем изменение кода (`user.id` → high-cardinality only в span, либо вообще убрать из метрик) и, при необходимости, чистка старых series через рестарт Prometheus/коррекцию retention. Связка с трейсингом и логами позволяет однозначно доказать, что именно это поле «рвануло» кардинальность, а не какие-то системные теги.

В реальной жизни инциденты редко бывают «чистыми» — часто всё перемешано: медленный эндпоинт одновременно вызывает шторм ретраев и повышенную кардинальность. Здесь помогает наличие **runbook’ов**: пошаговых инструкций «что смотреть в первую очередь», «какие графики открыть», «куда кликнуть из метрики в трейс», «какими запросами искать логи по traceId». Чем лучше связаны между собой слои метрик, трейсов и логов, тем проще формализовать такой runbook.

Ниже — пример упрощённого контроллера и сервиса в Spring Boot, где HTTP-эндпоинт обёрнут `@Timed` и `@Observed`, а внутри используется WebClient с ретраями. Такой код генерит и метрики latency/RPS, и трейсы с дочерними спанами, и логи с traceId. На его основе легко воспроизвести описанные сценарии в тестовом стенде.

**Java — эндпоинт с метриками, трейсингом и WebClient (для сценариев диагностики)**

```java
package com.example.observability.incidents;

import io.github.resilience4j.retry.annotation.Retry;
import io.micrometer.core.annotation.Timed;
import io.micrometer.observation.annotation.Observed;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

@RestController
class DiagnosticController {

    private final DiagnosticService diagnosticService;

    DiagnosticController(DiagnosticService diagnosticService) {
        this.diagnosticService = diagnosticService;
    }

    @GetMapping("/api/diagnostic")
    public Mono<String> diagnosticEndpoint() {
        return diagnosticService.callExternal();
    }
}

@Service
class DiagnosticService {

    private static final Logger log = LoggerFactory.getLogger(DiagnosticService.class);

    private final WebClient webClient;

    DiagnosticService(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://external-service").build();
    }

    @Timed(value = "diagnostic.endpoint", histogram = true)
    @Observed(
            name = "diagnostic.call",
            contextualName = "diagnostic-call",
            lowCardinalityKeyValues = {"operation", "diagnostic"}
    )
    @Retry(name = "diagnosticExternal")
    public Mono<String> callExternal() {
        log.info("Calling external service");
        return webClient.get()
                .uri("/slow-endpoint")
                .retrieve()
                .bodyToMono(String.class);
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.observability.incidents

import io.github.resilience4j.retry.annotation.Retry
import io.micrometer.core.annotation.Timed
import io.micrometer.observation.annotation.Observed
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.web.reactive.function.client.WebClient
import reactor.core.publisher.Mono

@RestController
class DiagnosticController(
    private val diagnosticService: DiagnosticService
) {

    @GetMapping("/api/diagnostic")
    fun diagnosticEndpoint(): Mono<String> =
        diagnosticService.callExternal()
}

@Service
class DiagnosticService(
    builder: WebClient.Builder
) {

    private val log = LoggerFactory.getLogger(DiagnosticService::class.java)

    private val webClient: WebClient = builder
        .baseUrl("http://external-service")
        .build()

    @Timed(value = "diagnostic.endpoint", histogram = true)
    @Observed(
        name = "diagnostic.call",
        contextualName = "diagnostic-call",
        lowCardinalityKeyValues = ["operation", "diagnostic"]
    )
    @Retry(name = "diagnosticExternal")
    fun callExternal(): Mono<String> {
        log.info("Calling external service")
        return webClient.get()
            .uri("/slow-endpoint")
            .retrieve()
            .bodyToMono(String::class.java)
    }
}
```

С таким кодом в проде или на стенде ты получаешь полный набор сигналов: метрики по latency и ошибкам, трейсы с ретраями и внешними вызовами, логи с traceId. А дальше — дело техники и дисциплины: прописать runbook’и и регулярно тренироваться в решении подобных сценариев.

# 10. Kubernetes и governance

## Prometheus Operator: ServiceMonitor/PodMonitor/Probe, scrape на sidecar/agent, метки/аннотации

В Kubernetes практически никогда не настраивают Prometheus «ручной» конфигурацией таргетов. Вместо этого используют Prometheus Operator: это контроллер, который управляет экземплярами Prometheus, Alertmanager и добавляет набор CRD — `ServiceMonitor`, `PodMonitor`, `Probe` и другие. Смысл в том, что конфигурация scrape-целей становится декларативной, частью Kubernetes-API, а не отдельного YAML где-то «сбоку». Для Spring Boot разработчика это важно: чтобы метрички сервиса подхватили, достаточно корректно оформить `Service` и `ServiceMonitor`.

`ServiceMonitor` — наиболее типичный ресурс. Он говорит Prometheus: «собирать метрики со всех Pod’ов, которые стоят за сервисами с такими-то лейблами, с такого-то порта и пути». Оператор постоянно следит за появлением/исчезновением сервисов и подов, и динамически обновляет конфиг Prometheus. Таким образом клистер может масштабироваться, pods могут пересоздаваться, а Prometheus не нужно перезагружать или редактировать конфиг руками.

`PodMonitor` нужен, когда сервис по каким-то причинам не подходит: например, у тебя sidecar с отдельным портом метрик, а `Service` проброшен только на основной контейнер; или у тебя DaemonSet без сервисов; или нужно скрейпить несколько портов с разных Pod’ов. `PodMonitor` выбирает Pods по лейблам напрямую и позволяет описать несколько target’ов/портов на Pod. Важно, что `ServiceMonitor` и `PodMonitor` не взаимоисключающие: ты можешь использовать и то, и то, для разных типов workload’ов.

`Probe` в контексте Prometheus Operator — это не Kubernetes liveness/readiness-probe, а отдельный CRD, позволяющий проверять внешние эндпоинты, которые не живут в кластере: например, SaaS-API или внешний HTTP-сервис. Prometheus создаёт target на такой Probe и регулярно опрашивает его. Это удобно, если часть твоих зависимостей — внешние системы, и ты хочешь иметь их доступность в той же системе мониторинга, что и внутренние сервисы. Но для Spring Boot-приложения чаще всего достаточно ServiceMonitor/PodMonitor.

Scrape «на sidecar/agent» означает, что не сам Prometheus ходит в приложение, а в контейнер-агент, который сидит рядом и, например, агрегирует метрики или трейсинг. Частый паттерн: в Pod’е Spring Boot сервис + OTel Collector sidecar, Prometheus скрейпит метрики из sidecar, а тот уже локально агрегирует/трассирует. Это даёт дополнительный уровень гибкости (можно конвертить формат, добавлять labels, фильтровать), но усложняет схему. В маленьком кластере sidecar часто избыточен, и Prometheus ходит напрямую в `/actuator/prometheus`.

Аннотации типа `prometheus.io/scrape: "true"` — более старый и простой способ подсказать stateless Prometheus’у, что этот Pod/Service нужно скрейпить. В мире Prometheus Operator это постепенно уходит на второй план: operator ожидает декларативный `ServiceMonitor`/`PodMonitor` и работает не на основе аннотаций, а на основе CRD. Однако аннотации всё ещё можно использовать как сигнал для генерации `ServiceMonitor`’ов или как временное решение при миграции. Governance-составляющая здесь в том, чтобы договориться: в организации используется один способ (через CRD), а аннотации — только как вспомогательные.

Kubernetes liveness/readiness/startup-пробы напрямую не связаны с прометеевскими Probe, но по факту они критичны для наблюдаемости. Если readiness неправильно настроен и Pod начинает принимать трафик до того, как `/actuator/prometheus` доступен, ты получаешь «дырки» в метриках и странные артефакты на графиках. Поэтому хорошая практика — readiness проверяет либо health-check из Actuator, либо отдельный лёгкий endpoint, который готов чуть раньше метрик, а Prometheus для метрик использует тот же порт/app, что и readiness.

Лейблы и аннотации в Kubernetes — ключ к управляемому мониторингу. Стандартный набор: `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`, плюс `team`, `owner`, `environment`. `ServiceMonitor`/`PodMonitor` обычно фильтруют по этим лейблам, а Prometheus добавляет их как labels к метрикам. Governance здесь — не позволить каждому сервису придумывать свои «app=xxx1, role=yyy2» и получить кашу. Нужен централизованный стандарт, по которому все сервисы лейблятся одинаково.

Ниже — пример сервис + ServiceMonitor для Spring Boot приложения с Actuator. Важные моменты: отдельный порт для management (или тот же, но другой path), корректные лейблы, и селекторы в ServiceMonitor, совпадающие с ними. Prometheus Operator увидит ServiceMonitor, найдёт соответствующие Service’ы и начнёт скрейпить `/actuator/prometheus`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  labels:
    app.kubernetes.io/name: orders-service
    app.kubernetes.io/part-of: ecommerce
spec:
  selector:
    app.kubernetes.io/name: orders-service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: metrics
      port: 8081
      targetPort: 8081
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: orders-service-monitor
  labels:
    team: payments
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: orders-service
  endpoints:
    - port: metrics
      path: /actuator/prometheus
      interval: 15s
      scrapeTimeout: 10s
```

В самом приложении нужно включить Actuator и Prometheus registry, а также при желании вынести management-порт отдельно. В Spring Boot это решается парой настроек в `application.yml` плюс зависимости Gradle. Далее весь pipeline — Kubernetes Service + ServiceMonitor.

**Gradle зависимости для Spring Boot Actuator + Prometheus**

*Groovy DSL:*

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

*Kotlin DSL:*

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
}
```

**Java — базовое приложение с Actuator и отдельным management-портом**

```java
package com.example.k8s.prometheus;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class OrdersServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrdersServiceApplication.class, args);
    }
}

@RestController
class HealthController {

    @GetMapping("/api/orders/ping")
    public String ping() {
        return "OK";
    }
}
```

```yaml
# application.yml
server:
  port: 8080

management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: "health,info,prometheus"
  endpoint:
    health:
      probes:
        enabled: true
```

**Kotlin — тот же сервис**

```kotlin
package com.example.k8s.prometheus

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class OrdersServiceApplication

fun main(args: Array<String>) {
    runApplication<OrdersServiceApplication>(*args)
}

@RestController
class HealthController {

    @GetMapping("/api/orders/ping")
    fun ping(): String = "OK"
}
```

Этот минимум даёт тебе полноценную интеграцию: Kubernetes следит за готовностью/живостью контейнера через Actuator, Prometheus Operator — за метриками через ServiceMonitor, а ты получаешь единообразный подход для всех сервисов.

---

## Multi-cluster/мультирегион: федерация/Thanos, tenant-изоляция и права

Когда у организации один кластер — архитектура наблюдаемости относительно проста: один Prometheus, один Tempo/Jaeger, один Grafana. Как только у тебя появляется несколько кластеров (регионов, облаков, on-prem + cloud), сразу возникают вопросы: где хранить метрики и трейсы, как строить дашборды «поверх» всех кластеров, как изолировать tenants и команды, чтобы не превратить мониторинг в монолитный зоопарк.

Классический подход к метрикам — Prometheus-федерация: один «верхнеуровневый» Prometheus периодически опрашивает (`federate`) другие Prometheus-инстансы и собирает агрегированные метрики. Это работает, но плохо масштабируется и требует аккуратного дизайна, какие именно метрики федератить. Современный более устойчивый вариант — Thanos/Mimir/Cortex: отдельный слой поверх Prometheus, который делает данные глобальными и долговременными, а локальные Prometheus’ы в кластерах превращает в «шарды» хранилища.

Thanos (как один из примеров) добавляет sidecar к каждому Prometheus или использует API, чтобы забирать блоки данных и складывать их в объектное хранилище (S3/GCS и т.п.). Query-frontends и querier’ы поверх этого видят метрики из всех кластеров как единое логическое пространство, дополненное labels `cluster`, `region` и т.п. Для разработчика это означает: ты можешь открыть один Grafana и увидеть метрики всех кластеров, фильтруя по `cluster="prod-eu"` или `cluster="stage-us"`.

Играть важную роль начинает **tenant-изоляция**. В небольших командах «тенант» — это просто команда/продукт; в больших — целые бизнес-юниты, иногда с отдельными юрлицами. В observability это означает, что разные команды не должны видеть метрики и трейсы друг друга, или хотя бы иметь ограниченный доступ к чувствительным частям. Отсюда вытекает: каждый кластер и сервис должен иметь чёткие resource-лейблы `service.name`, `service.namespace`, `k8s.cluster.name`, `tenant`. Эти лейблы используются и в Prometheus (labels), и в Tempo/Jaeger (resource attributes), и в Grafana (folder/datasource-permissions).

В multi-cluster-трейсинге похожие подходы: ты либо поднимаешь один глобальный backend (например, Tempo) с multi-tenant моделью, либо несколько региональных, а Grafana умеет ходить в разные источники. Resource attributes `service.name`, `deployment.environment`, `cloud.region`, `k8s.cluster.name` становятся обязательными: без них невозможно фильтровать трейсы по кластеру или региону. Обычно OTel Collector на стороне кластера добавляет эти атрибуты ко всем входящим спанам.

Горизонтальное разделение прав делается сочетанием Grafana RBAC и label-based фильтрации: каждому data source можно задать query-constraints (например, автоматически добавлять `tenant="foo"` ко всем запросам этой роли), а сами dashboards складывать по папкам с разными правами доступа. Таким образом, команды видят только свои сервисы, а платформа — всё. To做到 это нужно, чтобы везде использовалась одна и та же схема лейблов, а не «как получится».

Networking и безопасность в multi-region-обзорности тоже нетривиальны. Часто Prometheus/Tempo/Collector живут в отдельном «наблюдаемом» кластере или сети, к которому остальные кластеры подключаются по mTLS. В k8s значения для `OTEL_EXPORTER_OTLP_ENDPOINT`, `PROMETHEUS_REMOTE_WRITE_URL` и сертификатов приходят через `ConfigMap/Secret`, а сервисы не имеют прямого доступа к хранилищам — всю работу делает Collector/Agent. Это снижает риск ошибочной утечки ключей и делает управление проще.

С технической точки зрения важный шаг — добавить `externalLabels`/resource attributes, чтобы в прометеевских метриках и спанах всегда было понятно, *из какого кластера* и *из какого региона* пришёл сигнал. В Prometheus это делается на уровне конфигурации, в OTel — на уровне Resource/Collector. Если этого не сделать, все кластеры будут выглядеть одинаково, и поиск проблем «где именно упало» сильно усложнится.

Ниже — пример фрагмента конфигурации Prometheus (через Helm values или напрямую), где задаются `externalLabels` для кластера/региона. Они будут добавлены ко всем метрикам, исходящим от этого Prometheus, и далее попадут в Thanos/федерацию.

```yaml
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: "prod-eu"
      region: "eu-central-1"
      environment: "prod"
```

С точки зрения трейсинга аналогичная роль у resource attributes. Если по каким-то причинам ты хочешь задать их на уровне SDK (а не только Collector’а), можно собрать Resource с нужными атрибутами, используя значения из переменных среды, которые Kubernetes прокидывает в Pod. Ниже — пример Java/Kotlin конфигурации OTel SDK, который добавляет `k8s.cluster.name` и `cloud.region` в Resource.

**Java — добавление cluster/region в Resource OpenTelemetry SDK**

```java
package com.example.k8s.multicluster;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MultiClusterTracingConfig {

    @PostConstruct
    public void initTracing() {
        String clusterName = System.getenv().getOrDefault("K8S_CLUSTER_NAME", "unknown-cluster");
        String region = System.getenv().getOrDefault("CLOUD_REGION", "unknown-region");

        Resource resource = Resource.getDefault()
                .toBuilder()
                .put(ResourceAttributes.SERVICE_NAME, "orders-service")
                .put(ResourceAttributes.SERVICE_NAMESPACE, "ecommerce")
                .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "prod")
                .put(ResourceAttributes.K8S_CLUSTER_NAME, clusterName)
                .put(ResourceAttributes.CLOUD_REGION, region)
                .build();

        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setResource(resource)
                .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
                .build();

        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

**Kotlin — тот же подход к Resource**

```kotlin
package com.example.k8s.multicluster

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.resources.Resource
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Configuration

@Configuration
class MultiClusterTracingConfig {

    @PostConstruct
    fun initTracing() {
        val clusterName = System.getenv("K8S_CLUSTER_NAME") ?: "unknown-cluster"
        val region = System.getenv("CLOUD_REGION") ?: "unknown-region"

        val resource = Resource.getDefault()
            .toBuilder()
            .put(ResourceAttributes.SERVICE_NAME, "orders-service")
            .put(ResourceAttributes.SERVICE_NAMESPACE, "ecommerce")
            .put(ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "prod")
            .put(ResourceAttributes.K8S_CLUSTER_NAME, clusterName)
            .put(ResourceAttributes.CLOUD_REGION, region)
            .build()

        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .build()

        val tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

Такой Resource потом используется Thanos/Tempo/Grafana для фильтрации и мультирегионных дашбордов: ты видишь, какие трейсы/метрики относятся к какому кластеру и региону, и можешь строить отчёты и алёрты на нужном уровне.

---

## Политики: стандарты нейминга, «черный список» лейблов, ревью добавления метрик/тегов, лимиты ретеншна

Наблюдаемость без governance очень быстро превращается в хаос: у разных команд разные соглашения по именам метрик, лейблам, retention, alert’ам. Через год такую систему практически невозможно поддерживать: сложно понять, за что отвечает конкретная метрика, какой у неё владелец, какие SLO к ней привязаны. Поэтому на уровне организации нужны **политики**: минимальный набор правил и процессов, которые делают observability управляемой.

Первая группа политик — стандарты нейминга. Для метрик обычно задают формат: `<domain>_<object>_<operation>_<unit>`, где unit — явный суффикс (`seconds`, `bytes`, `count`, `ratio`). Например: `http_server_requests_seconds`, `db_query_duration_seconds`, `kafka_consumer_lag_records`. Для бизнес-метрик может быть свой префикс (`business_orders_created_total`). Важно, чтобы в имени метрики сразу было понятно, *что* измеряется и *в чём* измеряется. Для трейсов и спанов — похожие принципы: `http.server`, `payment.process`, `order.reserve`.

Вторая группа — «чёрный список» лейблов. Это список ключей, которые **запрещено** использовать как labels метрик и low-cardinality атрибуты спанов: `userId`, `email`, `sessionId`, `requestId`, любые значения с потенциально огромной кардинальностью или PII. Вместо этого используются агрегирующие поля: `tenant`, `user_segment`, `plan`, `region`. Документированная политика говорит: «Такой-то список лейблов нельзя добавлять без отдельного security-review; PII в лейблы запрещён; high-card поля только в high-cardinality данных (traces/logs), но не в метрики».

Третья группа — ревью добавления метрик и тегов. Идея простая: добавление новой метрики в прод — это **изменение API observability**, которое должно проходить через code-review с участием SRE/платформенной команды. Reviewer смотрит на название, тип, набор лейблов, оценку кардинальности, предположительную частоту обновления и retention. Если метрика потенциально опасна (слишком много лейблов, высокий rate, PII), её либо перепроектируют, либо отправляют в логи/трейсы вместо метрик.

Retention — четвёртая часть политики. Обычно прометеевские сырые метрики хранятся недолго (например, 7–30 дней), а агрегированные (через recording rules) — дольше (3–12 месяцев). Для разных окружений retention различается: dev/stage обычно имеют очень короткий срок (1–3 дня), prod — дольше, но всё равно ограничен. Эти значения не должны быть случайными: они завязаны на cost-модели и SLO уровней; данные, которые никто не использует через N дней, не нужно хранить дольше.

Важно также иметь договорённость по **ресурсным меткам**: все сервисы в Kubernetes используют одни и те же k8s-лейблы (`app.kubernetes.io/*`), все метрики включают `service`, `env`, `cluster`, все трейсы используют одинаковые resource attributes. Это сильно упрощает написание recording rules и алёртов: ты можешь взять общий шаблон SLO для всех HTTP-сервисов, потому что знаешь, что `service` и `env` всегда существуют и имеют стандартизированные значения.

Технически enforce-политик можно частично автоматизировать. Например, в Java/Kotlin можно сделать обёртку над Micrometer API, которая проверяет имена метрик и tags на соответствие регуляркам, запрещённым ключам и ограничению на длину. Это не идеальный enforcement (есть обходы), но он поймает многие ошибки ещё на стадии разработки. Остальная часть — процесс и культура: обсуждать метрики в дизайне, добавлять их в дизайн-документы и runbook’и, держать список «официальных» SLI.

Ниже — упрощённый пример Java/Kotlin-утилиты, которая создаёт счётчик с проверкой имени и лейблов. В боевом варианте здесь будет гораздо больше правил, но идея понятна: при попытке использовать «плохие» лейблы или имя метрики метод бросит исключение, и разработчик узнает об этом ещё в dev-цикле.

**Java — утилита для создания безопасных метрик с простейшей валидацией**

```java
package com.example.observability.policy;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;

import java.util.Map;
import java.util.Set;
import java.util.regex.Pattern;

public class SafeMetrics {

    private static final Pattern NAME_PATTERN =
            Pattern.compile("^[a-zA-Z_:][a-zA-Z0-9_:]*$");

    private static final Set<String> FORBIDDEN_LABELS = Set.of(
            "userId", "email", "sessionId", "requestId"
    );

    private SafeMetrics() {
    }

    public static Counter safeCounter(MeterRegistry registry,
                                      String name,
                                      Map<String, String> tags) {
        if (!NAME_PATTERN.matcher(name).matches()) {
            throw new IllegalArgumentException("Invalid metric name: " + name);
        }

        for (String key : tags.keySet()) {
            if (FORBIDDEN_LABELS.contains(key)) {
                throw new IllegalArgumentException("Forbidden label: " + key);
            }
        }

        Counter.Builder builder = Counter.builder(name);
        tags.forEach(builder::tag);
        return builder.register(registry);
    }
}
```

**Kotlin — аналогичный helper для метрик**

```kotlin
package com.example.observability.policy

import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry

object SafeMetrics {

    private val namePattern = Regex("^[a-zA-Z_:][a-zA-Z0-9_:]*$")

    private val forbiddenLabels = setOf(
        "userId", "email", "sessionId", "requestId"
    )

    fun safeCounter(
        registry: MeterRegistry,
        name: String,
        tags: Map<String, String>
    ): Counter {
        require(namePattern.matches(name)) {
            "Invalid metric name: $name"
        }

        tags.keys.forEach { key ->
            require(!forbiddenLabels.contains(key)) {
                "Forbidden label: $key"
            }
        }

        val builder = Counter.builder(name)
        tags.forEach { (k, v) -> builder.tag(k, v) }
        return builder.register(registry)
    }
}
```

Дополнительно в k8s обычно прописывают retention для Prometheus через аргументы командной строки или Helm values. Например, `--storage.tsdb.retention.time=15d` для короткой истории в staging и `--storage.tsdb.retention.time=30d` для prod. Это часть политик, и её стоит оформлять в виде стандартизированного Helm-шаблона, а не решать каждый раз с нуля.

---

## Runbooks и испытания: fault-injection (latency/packet-loss), нагрузка на хранилища, проверки MWMBR-алёртов в staging

Наблюдаемость мало просто «включить» — её нужно регулярно проверять и тренировать. Здесь на сцену выходят runbook’и и испытания. Runbook — это документированный сценарий действий при определённой проблеме: «если вырос p95 latency на таком-то эндпоинте — делаем шаги 1–2–3». Испытания — это Controlled Chaos: мы осознанно создаём аномалии в staging/канаре, чтобы убедиться, что метрики/трейсы/алёрты срабатывают так, как задумано, а команда умеет ими пользоваться.

Runbook’и фиксируют **инженерное знание** о системе: какие дашборды открыть, как перейти от метрики к трейсам, какие лог-запросы выполнить, как интерпретировать значения. Для каждого SLO (например, успешность 99.5% запросов к `/api/checkout`) желательно иметь отдельный runbook. В Kubernetes-контексте туда можно включить шаги по проверке состояния Pod’ов, Deployment’ов, сетевых политик и readiness/liveness-probe. Это связывает мир observability и мир кластерной инфраструктуры.

Fault injection — это способ убедиться, что observability действительно отражает реальные проблемы. Например, ты запускаешь Toxiproxy или Chaos Mesh, чтобы добавить +500 мс латентности к вызовам внешнего платежного сервиса, и смотришь, как реагируют метрики latency, error rate, ретраи и circuit breaker. Если SLO/алёрты настроены корректно, через некоторое время появится алёрт на ухудшение p95/p99, а трейсы покажут длинные client HTTP spans. Если алёрта нет — значит, SLO или порог настроены неправильно.

Похожим образом тестируется packet loss: Toxiproxy или сетевые политики могут обрывать часть запросов, создавая `connect timeout`/`read timeout`. На графиках ты должен увидеть рост ошибок и ретраев, а в трейсах — multiple client spans с ошибками. Это хороший способ проверить, что твои resilience-паттерны (ретраи, circuit breaker) и наблюдаемость работают согласованно, а не только на красивой схеме в документации.

Нагрузка на хранилища наблюдаемости — отдельный вид испытаний. Мониторинг и трейсинг сами по себе — сервисы, и они тоже могут быть узким местом. Полезно время от времени запускать нагрузочные тесты, которые генерируют много метрик/трейсов, смотря, как ведут себя Prometheus/Tempo/Collector: не заполняется ли диск, не растёт ли latency запросов к Grafana, не дропаются ли спаны из-за переполнения очередей. Это лучше сделать в staging/отдельном «обсервабельном» окружении, чем поймать подобную ситуацию в проде.

MWMBR-алёрты (Multi-Window, Multi-Burn-Rate) — стандартный подход для SLO: они используют несколько временных окон (короткое и длинное) и несколько уровней burn-rate (насколько быстро тратим «ошибочный бюджет»). Их нужно не только настроить, но и проверить. Исследования показывают, что большинство алёртов либо «орёт постоянно», либо «молчит даже при серьёзных проблемах». Испытания с fault injection в staging позволяют прогнать несколько сценариев (быстрый сильный инцидент, медленный деградирующий) и посмотреть, где именно срабатывают MWMBR-алёрты.

В коде сервиса иногда удобно иметь *chaos endpoint* для побитых/закрытых окружений: endpoint, который иногда специально замедляет ответ или выбрасывает ошибку. В prod он должен быть закрыт (feature-flag, auth, вообще выключен), но в staging/канаре его можно использовать для генерации контролируемых аномалий. С точки зрения observability это источник «тестовых инцидентов», а для команды — тренировочный полигон.

Ниже — пример Spring Boot-сервиса с простейшим chaos endpoint, который по флагу замедляет ответ и иногда выбрасывает исключения. Такой код можно использовать в staging, чтобы проверить, как срабатывают SLO/алёрты и как команда проходит путь: Grafana → Tempo → логи.

**Java — chaos endpoint для проверки observability**

```java
package com.example.observability.chaos;

import io.micrometer.core.annotation.Timed;
import io.micrometer.observation.annotation.Observed;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.Duration;
import java.util.Random;

@RestController
public class ChaosController {

    private static final Logger log = LoggerFactory.getLogger(ChaosController.class);

    @Value("${chaos.enabled:false}")
    private boolean chaosEnabled;

    private final Random random = new Random();

    @GetMapping("/api/chaos")
    @Timed(value = "chaos.request", histogram = true)
    @Observed(
            name = "chaos.request",
            contextualName = "chaos-request",
            lowCardinalityKeyValues = {"operation", "chaos"}
    )
    public String chaos() throws InterruptedException {
        if (!chaosEnabled) {
            return "Chaos disabled";
        }

        int delay = 100 + random.nextInt(2000);
        boolean fail = random.nextDouble() < 0.2;

        log.info("Chaos endpoint: delay={}ms, fail={}", delay, fail);
        Thread.sleep(delay);

        if (fail) {
            throw new RuntimeException("Chaos failure");
        }

        return "Chaos OK after " + Duration.ofMillis(delay);
    }
}
```

**Kotlin — тот же chaos endpoint**

```kotlin
package com.example.observability.chaos

import io.micrometer.core.annotation.Timed
import io.micrometer.observation.annotation.Observed
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Value
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import java.time.Duration
import kotlin.random.Random

@RestController
class ChaosController(

    @Value("\${chaos.enabled:false}")
    private val chaosEnabled: Boolean
) {

    private val log = LoggerFactory.getLogger(ChaosController::class.java)

    @GetMapping("/api/chaos")
    @Timed(value = "chaos.request", histogram = true)
    @Observed(
        name = "chaos.request",
        contextualName = "chaos-request",
        lowCardinalityKeyValues = ["operation", "chaos"]
    )
    @Throws(InterruptedException::class)
    fun chaos(): String {
        if (!chaosEnabled) {
            return "Chaos disabled"
        }

        val delay = 100 + Random.nextInt(2000)
        val fail = Random.nextDouble() < 0.2

        log.info("Chaos endpoint: delay={}ms, fail={}", delay, fail)
        Thread.sleep(delay.toLong())

        if (fail) {
            throw RuntimeException("Chaos failure")
        }

        return "Chaos OK after ${Duration.ofMillis(delay.toLong())}"
    }
}
```

Такой endpoint в сочетании с Toxiproxy/Chaos Mesh позволяет относительно безопасно тренировать observability-пайплайн: ты знаешь, когда и как именно создал проблему, и можешь проверить, как система наблюдаемости на это реагирует.

---

## Экономика: сэмплинг трейсинга, downsampling метрик, агрегации/recording rules вместо тяжёлых adhoc

Наблюдаемость всегда стоит денег: CPU/память/диск/сетевой трафик/лицензии. В Kubernetes, где сервисов и Pod’ов много, можно легко прийти к ситуации, когда «мониторинг стоит дороже, чем прикладная нагрузка». Поэтому, помимо технического дизайна, нужна **экономическая модель** observability: какие сигналы собираем с какой частотой, где делаем sampling, что агрегируем, сколько храним и кто платит.

Первый рычаг — sampling трейсинга. Писать все трейсы в проде обычно нецелесообразно: объём данных растёт линейно с RPS, а хранение и поиск стоят дорого. Head-sampling на уровне SDK/агента (например, 5–10%) плюс tail-sampling в Collector’е для ошибок и важных эндпоинтов дают хороший баланс: для «нормальных» запросов храним только часть трасс, но для ошибок и деградаций стремимся к 100%. В dev/stage можно sampling выключать (AlwaysOn), потому что трафик небольшой и данные нужны для отладки.

Второй рычаг — downsampling метрик. Сырые time-series с шагом 15–30 секунд за долгое время занимают много места. Простой приём: хранить raw-метрики недолго (7–15 дней), а дальше — использовать заранее рассчитанные recording rules, которые агрегируют данные по минутам/часам. В Prometheus это делается через дополнительный инстанс/Thanos или записывая агрегаты в отдельные серии. Grafana для долгих периодов обращается уже не к raw, а к downsampled-рядом.

Третий рычаг — дизайн дашбордов и запросов. Ad-hoc запросы с `rate()`/`histogram_quantile()` по сотням тысяч series — это не только медленно, но и дорого. Поэтому для часто используемых SLI/SLO нужно писать отдельные recording rules, которые заранее считают нужные агрегаты. Тогда дашборды и алёрты используют лёгкие запросы к уже аггрегированным рядам, а не каждый раз делают тяжёлые расчёты на лету.

Четвёртый рычаг — rate limiting и budgets на уровне observability. Можно задать per-namespace/per-service лимиты: сколько метрик, логов и трейсов сервис имеет право генерировать. Это может быть жёстко (обрезать данные) или мягко (алёрты команде: «вы вышли за бюджет, оптимизируйте метрики»). В Kubernetes это можно реализовать через лимиты в Collector’е, ретеншн в Prometheus и квоты на лог storage. Governance-команда при этом следит, чтобы бюджеты были реалистичными и согласованы с критичностью сервисов.

Пятый аспект — разные уровни для разных сред. Dev и ephemeral-окружения обычно получают минимальный набор сигналов и короткий ретеншн: трейсы без sampling (но мало трафика), метрики с простым retention (1–3 дня), логи с ограничением размера. Prod окружения получают более сложную конфигурацию: SLO-метрики, MWMBR-алёрты, трейсы с sampling, более длинный ретеншн и регулярные отчёты. Важно не переносить прод-политику 1:1 на dev: это бессмысленно и дорого.

Практически это сводится к тому, что конфигурация sampling/retention/recording rules становится частью Helm-chart’ов и ConfigMap’ов: для каждого окружения — свой values.yaml. В коде тоже иногда нужно учитывать экономику: например, брать sampling-коэффициент из конфигурации, а не хардкодить `0.1`. Это позволяет менять политику без перекомпиляции (например, временно поднять sampling на инцидент, а потом опустить).

Ниже — пример конфигурации Prometheus recording rules для SLO по HTTP 5xx, а также пример Java/Kotlin-конфига OpenTelemetry Sampler, который берёт sampling-коэффициент из `application.yml` (или переменной среды). Это показывает, как экономика и код связаны: политика задаётся в конфиге, а SDK подстраивается под неё.

```yaml
# prometheus recording rules (пример для SLO)
groups:
  - name: http_slo_rules
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum by (job, handler, status) (
          rate(http_server_requests_seconds_count[5m])
        )
      - record: job:http_requests_errors:rate5m
        expr: sum by (job, handler) (
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
        )
      - record: job:http_requests_error_ratio:rate5m
        expr: job:http_requests_errors:rate5m
          /
          job:http_requests_total:rate5m
```

**Java — Sampler с коэффициентом из конфигурации Spring Boot**

```java
package com.example.observability.economy;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.Sampler;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SamplingEconomyConfig {

    @Value("${observability.tracing.sampling-ratio:0.1}")
    private double samplingRatio;

    @Value("${observability.tracing.otlp-endpoint:http://otel-collector:4317}")
    private String otlpEndpoint;

    @PostConstruct
    public void initTracing() {
        Sampler sampler = Sampler.parentBased(
                Sampler.traceIdRatioBased(samplingRatio)
        );

        OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint(otlpEndpoint)
                .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setSampler(sampler)
                .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
                .build();

        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();

        GlobalOpenTelemetry.set(openTelemetry);
    }
}
```

```yaml
# application.yml
observability:
  tracing:
    sampling-ratio: 0.05
    otlp-endpoint: http://otel-collector:4317
```

**Kotlin — тот же конфиг с параметризованным sampling**

```kotlin
package com.example.observability.economy

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.sdk.OpenTelemetrySdk
import io.opentelemetry.sdk.trace.SdkTracerProvider
import io.opentelemetry.sdk.trace.Sampler
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter
import jakarta.annotation.PostConstruct
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Configuration

@Configuration
class SamplingEconomyConfig(

    @Value("\${observability.tracing.sampling-ratio:0.1}")
    private val samplingRatio: Double,

    @Value("\${observability.tracing.otlp-endpoint:http://otel-collector:4317}")
    private val otlpEndpoint: String
) {

    @PostConstruct
    fun initTracing() {
        val sampler: Sampler = Sampler.parentBased(
            Sampler.traceIdRatioBased(samplingRatio)
        )

        val exporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint(otlpEndpoint)
            .build()

        val tracerProvider = SdkTracerProvider.builder()
            .setSampler(sampler)
            .addSpanProcessor(BatchSpanProcessor.builder(exporter).build())
            .build()

        val openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build()

        GlobalOpenTelemetry.set(openTelemetry)
    }
}
```

Такая привязка к конфигу позволяет менять sampling для разных окружений через Helm values, а не через код: для prod — 5%, для canary — 20%, для dev — 100%. В сочетании с downsampling метрик и recording rules это формирует управляемую, предсказуемую и экономически оправданную систему наблюдаемости поверх Kubernetes.


