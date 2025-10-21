---
layout: page
title: "Наблюдаемость (расширено): метрики и трейсинг"
permalink: /spring/observability
---

# 0. Введение

## Что это

Наблюдаемость — способность системы отвечать на вопросы «почему сейчас так?» без модификации кода и долгих раскопок. Технически это стек из **метрик** (агрегированные числовые ряды), **трейсов** (детальная хронология одного запроса), и **логов** (текстовые записи контекста). В современном Java-мире мы опираемся на Micrometer + Spring Boot Actuator для метрик и на OpenTelemetry (OTel) для трейсинга и логов-корреляции.

Ключевая особенность наблюдаемости — **причинно-следственная связка** между уровнями. Метрика «вырос p99» сама по себе бесполезна, если вы не можете провалиться к конкретному трейсy с проблемным спаном, а затем к логам того спана. Поэтому мы строим интегрированный pipeline: метрики с **exemplar**-ссылками на traceId, трейсинг, который проставляет **MDC** (Mapped Diagnostic Context) для логов.

Важно отличать наблюдаемость от мониторинга. Мониторинг — «следить за известными симптомами» (жив ли сервис, хватает ли CPU), наблюдаемость — «уметь ответить на новые вопросы», когда симптомов ещё не формализовали. Для этого мы собираем **богатые семантические атрибуты** (service.name, endpoint, метод, статус, tenant и др.) и ограничиваем их кардинальность.

Наконец, наблюдаемость — часть инженерной культуры. Она должна быть **по умолчанию** в каждом сервисе: Actuator подключён, базовые метрики активны, трассировка включена, а в `application.yml` заданы единые соглашения по именам, тегам и экспорту в вашу платформу (Prometheus/Tempo/Grafana, Datadog, New Relic и т.д.).

## Зачем это

Главная цель — **сократить MTTR** (время до восстановления) и **избежать повторов инцидентов**. Когда SLO нарушен, вы не «перезагружаете вслепую», а быстро находите точку деградации: «медленный SQL», «шторм ретраев у внешнего API», «истощение пула». Это снижает стоимость простоя и стресс на дежурствах.

Вторая цель — **осознанная оптимизация**. На глаз «всё нормально» редко совпадает с данными. Метрики позволяют увидеть, что узкое место — не CPU, а, например, `queue depth` вашего `ThreadPoolTaskExecutor` или `slow_call_rate` в Resilience4j. Туда и направляем усилия.

Третья — **прозрачность продукта**. Бизнес-метрики (конверсия, скорость выдачи контента, успешность платежей) привязываются к техметрикам (p95 эндпоинтов, доля 5xx) и инфраструктуре (спайки GC, диск). Это позволяет принимать продуктовые решения на основе фактов.

Четвёртая — **комплаенс и аудит**. Многие отрасли требуют журналирования операций, трассировки и ретенции данных. Правильная наблюдаемость закрывает эти требования «по умолчанию», не превращая каждую проверку в пожар.

## Где используется

В микросервисной архитектуре наблюдаемость — «кровеносная система». Каждый сервис экспортирует метрики и трейсинг, а платформа (Prometheus/Tempo/Grafana/Alertmanager) собирает и визуализирует их. Это справедливо и для монолита: принципиально те же инструменты, просто меньше компонент.

В интеграциях (HTTP-клиенты, Kafka, JDBC/Redis) наблюдаемость помогает увидеть «узкое звено»: где именно теряется время и как часто происходят ошибки. Micrometer предоставляет готовые биндинги ко многим клиентам, а OTel Java Agent — автоподключаемые интструменты, снимающие спаны без вашего кода.

В DevOps-практиках наблюдаемость используется для **кайзена**: вы измеряете эффект оптимизаций, релизов, конфигурационных изменений. Без измерений всё превращается в «кажется стало лучше».

В SecOps наблюдаемость дополняет безопасность: видно аномалии (всплески 401/403, странные паттерны запросов), лёгко связать их с релизами/конфигами. Это уменьшает время реакции на инциденты безопасности.

## Какие проблемы решает

Проблема №1 — **слепые зоны**: «падает, но не знаем где». Метрики + трейсинг закрывают это: вместо «везде провал» вы видите конкретный эндпоинт/брокер/БД.

Проблема №2 — **неповторяемые баги**. Трейсы сохраняют «киноплёнку» запроса: последовательность вызовов, тайминги, статусы — шанс воспроизвести в деве не обязателен, если есть точный «отпечаток».

Проблема №3 — **лавины ретраев и деградация**. Метрики устойчивости (p99, slow_call_rate, состояние circuit breaker, длина очередей) помогают среагировать раньше «чёрного экрана», включить деградации/kill-switch.

Проблема №4 — **стоимость инцидентов**. Хорошо настроенные алёрты по SLO (MWMBR — multi-window multi-burn-rate) дают сигнал вовремя и с правильным приоритетом, не будя людей из-за шумов.

---

# 1. Цели и таксономия наблюдаемости

## SLI/SLO/SLA

**SLI** (Service Level Indicator) — измеряемый показатель качества: доля успешных запросов, p95 задержки, доля 5xx. **SLO** (Service Level Objective) — целевое значение SLI за окно: «успешность ≥ 99.9% за 30 дней», «p95 ≤ 200 мс». **SLA** — юридическое обещание клиенту (часто с компенсациями).

Чтобы SLO не были «бумажными», привяжите их к техметрикам. Например, SLI «успешность» — это `rate(http_server_requests_seconds_count{status!~"5.."} ) / rate(http_server_requests_seconds_count)`. SLO — порог 99.9% за 30 дней; в алёртах используется «скорость прожигания» ошибки (burn rate).

Важно договориться о **границах**: что считать «успешным», как маппить коды (например, 499/408), учитывать ли ретраи клиента. Чёткая спецификация SLI снимает половину споров на постмортемах.

Пример формализации:

* **Availability SLI**: `1 - (5xx_count / all_count)`
* **Latency SLI**: `http_server_requests_seconds{status=~"2.."} p95 ≤ 200ms`
* **Error Budget**: `1 - SLO` (сколько «можно» сжечь ошибок за окно).

## RED/Golden Signals и USE

Модель **RED** (Rate, Errors, Duration) — упрощённый набор для сервисов: сколько запросов, сколько ошибок, какова длительность. Она легла в основу «золотых сигналов» SRE Google: **Traffic, Errors, Latency, Saturation**. Traffic ~ Rate, Errors — доля отказов, Latency — pXX, Saturation — «насыщение ресурса» (CPU, очередь, память).

Модель **USE** (Utilization, Saturation, Errors) больше про ресурсы: загрузка CPU/диска/сети, насыщение (насколько близки к пределу), ошибки (дропы/переполнения). Это удобно для нижних уровней (JVM, пулы, БД).

На практике комбинируем: на дашборде сервиса — RED/Golden Signals; на дашбордах инфраструктуры — USE. Так видно и бизнес-уровень («падают ответы»), и физику («очередь выросла»).

Ключ — **последовательность**. Все сервисы оформляют одни и те же панели/метрики с одинаковыми именами/лейблами. Это даёт эффект «встал к чужому сервису — всё знакомо».

## Стоимость и риск

Метрики и трейсы стоят денег (CPU, сеть, хранение). Определите **минимальный обязательный набор**: http.server/client, JVM/GC, пулы/очереди, база/кеш, устойчивость (Resilience4j), бизнес-RPS/успех. Всё остальное — «опционально и по делу».

Опасен **разгон кардинальности** (динамические лейблы со множеством значений: raw-URL, userId, sessionId). Один злосчастный лейбл может взорвать Prometheus/биллы SaaS. Введите «чёрный список» и ревью изменений в метриках.

Хранение — планируйте ретеншн и даунсемплинг: «детально за 7–14 дней», «агрегировано за месяцы/кварталы». Для тяжёлых вычислений используйте recording rules, а не adhoc-запросы с `histogram_quantile()` по миллионам сэмплов.

Для трейсинга включайте **sampling**. Head-sampling (на входе) 1–10% обычно достаточно; Tail-sampling — умнее (оставляет интересные/ошибочные), но сложнее в эксплуатации.

---

# 2. Дизайн метрик: типы, единицы, кардинальность, гистограммы

## Типы: Counter/Gauge/Timer/DistributionSummary/Histogram

**Counter** — только растёт (например, число запросов, ошибок). Идемпотентность: инкрементируйте один раз на факт, а не «на каждом ретрае» бездумно — иначе SLI «задвоится». **Gauge** — текущая величина (размер очереди, активные потоки). Его нужно правильно «обнулять» и не плодить многомерных gauge с высоким шумом.

**Timer** — счётчик + суммарное время; Micrometer даёт percentiles или histogram по нему. **DistributionSummary** — распределение размеров (байты/элементы). **Histogram** — идеален для latency/размеров: фиксированные корзины (Prometheus) или нативные гистограммы (Micrometer/OTel) с динамическими бинами.

Выбирайте тип по задаче: длительности — Timer/Histogram; частоты — Counter; состояние — Gauge. Не превращайте любой замер в Counter «на всё подряд» — сложно интерпретировать.

Пример кода:

```java
@Component
class BillingMetrics {
  private final Counter charges;
  private final Timer chargeTimer;
  BillingMetrics(MeterRegistry r) {
    this.charges = Counter.builder("billing.charge.count").description("charges").register(r);
    this.chargeTimer = Timer.builder("billing.charge.latency").publishPercentileHistogram().sla(Duration.ofMillis(100), Duration.ofMillis(200)).register(r);
  }
  public <T> T measure(Supplier<T> action){
    return chargeTimer.record(action::get);
  }
  public void inc(){ charges.increment(); }
}
```

## Единицы и имена

Единицы — часть смысла. Секунды/миллисекунды — в имени метрики или теге `unit="seconds"`. В Micrometer и Prometheus принято `*_seconds`, `*_bytes`, `*_total` (для счётчиков). Не смешивайте разные единицы в одном потоке — сложно читать/алертить.

Нейминг: `{domain}.{object}.{action}.{metric}` (пример: `http.server.requests` уже дан Actuator’ом; свои: `pricing.rule.apply.latency`). Стандартизируйте теги: `method`, `uri` (нормализованный путь), `status`, `outcome`, `exception`, `service`.

Документируйте метрики: описание (`description`) в коде/конфиге, ссылку на runbook. Это экономит часы в инцидентах.

Пример именования бизнес-метрики:

```java
Counter.builder("checkout.orders.total")
  .description("Placed orders count")
  .tag("channel","web")
  .register(registry);
```

## Кардинальность: контроль label’ов

Главный источник боли — **высокая кардинальность** лейблов. Нельзя класть `userId`, `sessionId`, «сырой» `uri` (со всеми ID), `errorMessage`. Такие теги порождают миллионы time-series.

Решение: нормализуйте `uri` (шаблоны `/orders/{id}`), округляйте значения (`amount_bucket="0-10/10-100/..."`), используйте `tenant` только если число тенантов ограничено, вынесите «шумные» лейблы в **логи**/трейсы, а не в метрики.

В Prometheus включите алёрт на рост `tsdb_head_series` и держите **дашборд кардинальности**: top-N метрик/лейблов по количеству серий. В код-ревью добавляйте пункт «оценка кардинальности метрик».

В Micrometer можно ограничить «разнообразие» через `MeterFilter.deny(id -> ...)` и `MeterFilter.commonTags(...)`, чтобы привести к общим тегам (service, env, region).

```java
@Bean MeterFilter denyUserId() {
  return MeterFilter.deny(id -> id.getTag("userId") != null);
}
```

## Временные распределения: buckets/percentiles

Latency — распределение с «тяжёлым хвостом». Среднее (avg) мало что говорит; нужны **квантили** (p50/p90/p95/p99). В Prometheus их оценивают через **гистограммы**: вы собираете счётчики по корзинам (`le="0.1"`, `0.2`, `0.5`, …), потом `histogram_quantile(0.95, sum(rate(metric_bucket[5m])) by (le))`.

Micrometer умеет `percentileHistogram(true)` и/или «целевые» percentiles. Для жёстких SLO используйте **SLA-корзины** (`sla(50ms,100ms,200ms,…)`) — это даст точную оценку SLI «доля ниже 200мс» без квантилей.

Нативные гистограммы (OTel/Micrometer) улучшили точность/стоимость, но поддержка зависит от бекенда. В Prometheus классические фикс-бакеты до сих пор основа. Выбирайте, что поддерживается вашей платформой.

Пример конфигурации:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      sla:
        http.server.requests: 50ms,100ms,200ms,500ms
```

## Exemplars: метрика→трейс

**Exemplar** — «прикреплённый к метке» пример значения с `traceId/spanId`. На графике p99 вы кликаете точку и попадаете в конкретный трейс, соответствующий этому «плохому» сэмплу. Это ускоряет RCA в разы.

Чтобы exemplars работали, надо: (1) включить трейсинг (OTel), (2) использовать Micrometer/OTel-registry, который умеет записывать `traceId` в метрики, (3) бекенд, поддерживающий exemplars (Prometheus/Grafana). В Spring Boot 3 Micrometer Observation + мост в OTel закрывают это «без боли».

Следите за приватностью: `traceId` сам по себе нейтрален, но не кладите в метрики PII—теги. Для спусков «метрика→лог» используйте `traceId` в MDC и связку в Grafana/лог-платформе.

Мини-пример `@Observed` (метрики + трейс + exemplar):

```java
@Observed(name="orders.create", contextualName="orders:create")
public Order createOrder(CreateOrder cmd) {
  // метрика Timer + спан с traceId; exemplar приляжет к Timer sample
  return service.create(cmd);
}
```

---

# 3. Инструментирование в Spring Boot (Micrometer/Actuator)

## Встроенные метрики: HTTP/JVM/DB/Exec/Kafka/Redis

Spring Boot Actuator с Micrometer автоматически экспонирует множество метрик: `http.server.requests` (эндпоинты, коды), `http.client.requests` (WebClient/RestTemplate), `jvm.*` (память, GC), `executor.*` (пулы), `tomcat/netty`, `datasource` (Hikari), Kafka/Redis (если стартеры подключены).

Это «бесплатные сигналы», которые уже покрывают RED/USE. Их достаточно, чтобы построить базовый дашборд: RPS, ошибки, p95, загрузка пула потоков и длительность SQL. Главное — не выключайте их «ради экономии»; лучше урезать ретеншн.

Для JDBC полезно включить **query time** через `datasource.metrics.enabled=true` (по умолчанию Hikari биндинг даёт время «займа» подключения, активных/ожидающих). Для WebClient — подключить `ExchangeFilterFunction` с Observation (в Boot 3 это приходит «из коробки»).

Мини-конфиг:

```yaml
management:
  endpoints.web.exposure.include: health,info,prometheus
  metrics.tags:
    application: order-service
```

## Observation API: единый слой + мост в OTel

**Micrometer Observation** — унифицированный API поверх метрик и трейсинга. Аннотация `@Observed` или ручной `Observation` создаёт **Timer**/**Span** одновременно, вешает теги/атрибуты, и автоматически пропихивает `traceId` в MDC.

Это закрывает типовую боль «дублирования»: раньше разработчик «забывал» добавить метрику/спан в одном из мест. Теперь одно место — Observation — и вы покрыты сразу обоими слоями.

В Boot 3+ Observation интегрирован в Web/HTTP, JDBC, R2DBC, Kafka, Redis, Resilience4j. То есть большинство «сквозных» операций уже обёрнуты. Вручную помечаем **бизнес-операции**: оформление заказа, расчёт цены, валидация промо — там где важен контекст предметной области.

Пример ручного Observation:

```java
@Autowired ObservationRegistry registry;

public Price quote(Request req) {
  var obs = Observation.start("pricing.quote", registry)
    .lowCardinalityKeyValue("sku", req.sku())
    .contextualName("pricing:quote");
  try (var scope = obs.openScope()) {
    return engine.calculate(req);
  } catch (RuntimeException e) {
    obs.error(e); throw e;
  } finally {
    obs.stop();
  }
}
```

## Кастомные метрики: Timer/LongTaskTimer/DistributionSummary

Обязательные системные метрики хороши, но вам потребуются **бизнес-метрики**: «создано заказов», «время сборки корзины», «размер ответа». Тут пригодятся `Timer`, `LongTaskTimer` (для долгих задач, например, nightly-репорт), `DistributionSummary` (байты/штуки).

Важно: **ключи/теги** должны быть стабильными и низкокардинальными (канал=web/app, регион, tenant ограниченного множества). Не складывайте туда `userId`. Лучше добавьте `traceId` в логи/трейс и свяжите по нему.

Ещё одна практика — «комбинаторы» метрик: один сервисный слой оборачивает запросы в `Timer` и дописывает теги: `endpoint`, `mode` (cache/origin), `result` (hit/miss/error). Это превращает графики из «средней температуры» в диагностические.

Пример:

```java
Timer timer = Timer.builder("catalog.fetch")
  .tag("mode", "origin")
  .publishPercentileHistogram()
  .register(registry);

Catalog cat = timer.record(() -> client.fetch());
```

## Конфигурация дистрибуций: percentiles/histogram/SLA/exemplars

В Spring Boot можно **глобально** или точечно включить percentiles и гистограммы. Для http.server.requests имеет смысл `percentileHistogram=true` и настроить SLA-корзины (например, 50/100/200/500 мс). Это даст точные SLI «доля <200мс».

Exemplars включаются автоматически при наличии трейсинга (Micrometer + OTel bridge). В Grafana вы увидите точки на линии p95 — кликните и провалитесь в Tempo/Jaeger на соответствующий трейс.

Следите за **стоимостью**: гистограммы дороже счётчиков. Включайте их только на ключевых метриках (HTTP, внешние клиенты, бизнес-операции). Для шумных внутренних метрик хватит `Timer` без дистрибуций.

Пример YAML:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        http.client.requests: true
        executor.active: false
      sla:
        http.server.requests: 50ms,100ms,200ms,500ms
```

## Экспорт: Prometheus scrape vs OTLP Metrics

Два популярных пути экспорта метрик: **Prometheus scrape** и **OTLP Metrics** через OTel-Registry/Collector. Scrape — Prometheus сам ходит на `/actuator/prometheus`, парсит текст, хранит/алёртит. Просто, прозрачно, дешёво.

OTLP-метрики — метрики отправляются (push) в OTel Collector и далее в хранилище (Prometheus Remote-Write, Mimir/Cortex/Datadog). Этот путь удобен, когда у вас уже есть центральный Collector для трейсинга/логов, и хочется единый агент и политику ретраев/батчинга.

В Spring Boot одновременно держать **оба** экспортёра не нужно. Выберите основной для вашей платформы. Если уже есть Prometheus — придерживайтесь scrape-модели, а OTel используйте для трейсинга. Если платформа «вендорская» — вероятно, предпочтение за OTLP.

Конфигурация Prometheus-эндпоинта:

```yaml
management:
  endpoint.prometheus.enabled: true
  endpoints.web.exposure.include: prometheus,health,info
```

Конфигурация OTLP-метрик через Micrometer (примерно):

```yaml
management:
  metrics.export.otlp:
    enabled: true
    url: http://otel-collector:4318/v1/metrics
    resource-attributes: service.name=order-service,service.namespace=shop
```

# 4. Прометеевский стек и правила эксплуатации

## Scrape-модель: интервалы, timeouts, relabeling, Service/PodMonitor (Operator)

Prometheus работает по **pull**-модели: сам ходит на таргеты и собирает метрики по HTTP. Два ключевых параметра — `scrape_interval` (как часто забирать) и `scrape_timeout` (максимальное время на ответ). Практичный дефолт: 15s/10s для приложений и 30s/25s для «тяжёлых» таргетов (брокеры, базы). Важно, чтобы `scrape_timeout < scrape_interval`, иначе вы можете «наложить» сборы друг на друга и создать лавинообразную нагрузку на приложение.

**Relabeling** — мощный фильтр на этапе обнаружения и сбора: им можно переименовывать лейблы, отбрасывать шумные таргеты, нормализовать имена. Например, в Kubernetes часто выкидывают «эпемерные» лейблы (`pod_template_hash`), чтобы они не попадали в метрики. Это снижает кардинальность и упрощает дашборды.

В Kubernetes лучший способ подключения — **Prometheus Operator** (или kube-prometheus-stack). Вместо ручного добавления таргетов вы описываете `ServiceMonitor`/`PodMonitor`: это Custom Resource Definitions (CRD), которые «объясняют» Prometheus, какие сервисы и на каких портах/путях скрапить. `ServiceMonitor` смотрит на **Service**, `PodMonitor` — на **Pod**. Для Spring Boot надо просто экспонировать `/actuator/prometheus`.

Мини-пример конфигов:

```yaml
# values.yaml для Prometheus (фрагмент)
global:
  scrape_interval: 15s
  scrape_timeout: 10s

# Service + ServiceMonitor для Spring Boot
apiVersion: v1
kind: Service
metadata:
  name: order-svc
  labels: { app: order }
spec:
  selector: { app: order }
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-sm
spec:
  selector:
    matchLabels: { app: order }
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
      scrapeTimeout: 10s
      scheme: http
```

И пример relabel, отбрасывающий лишние лейблы:

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_pod_template_hash]
    action: labeldrop
```

## Правила: recording rules и MWMBR-алёрты по SLO

**Recording rules** позволяют предварительно агрегировать тяжёлые запросы в новые тайм-ряды и не считать их «на лету» на каждом графике и алёрте. Типичный пример — квантили по гистограммам (`histogram_quantile`), rate по тысячам серий и свёртки в «SLI готовые для алёртов». Это разгружаетTSDB и делает алёрты стабильными.

Для SLO используем **multi-window multi-burn-rate (MWMBR)**: правила «сгорания бюджета» ошибки одновременно на коротких и длинных окнах (например, 5m/30m/2h/6h). Если на коротком окне всё плохо — это «пейдж», на длинном — «тикет/наблюдение». Так вы ловите и резкие всплески, и затяжные деградации.

Пример набора правил: сначала SLI, потом алёрты MWMBR. Предположим, SLI успешности — доля 2xx/3xx (исключая 5xx) по `http_server_requests_seconds_count`.

```yaml
# recording rules
groups:
- name: slo:records
  interval: 30s
  rules:
  - record: job:http_request_total:rate5m
    expr: sum by (job)(rate(http_server_requests_seconds_count[5m]))
  - record: job:http_success_total:rate5m
    expr: sum by (job)(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
  - record: job:sli_availability:ratio5m
    expr: job:http_success_total:rate5m / job:http_request_total:rate5m
```

И сами алёрты по SLO 99.9%:

```yaml
groups:
- name: slo:alerts
  rules:
  # 5m/1h
  - alert: SLOErrorBudgetBurn
    expr: (1 - job:sli_availability:ratio5m) > (1 - 0.999) * 14.4
    for: 5m
    labels: { severity: page }
    annotations:
      summary: "SLO burn fast (5m window)"
  # 30m/6h
  - alert: SLOErrorBudgetBurnSlow
    expr: (1 - avg_over_time(job:sli_availability:ratio5m[30m])) > (1 - 0.999) * 6
    for: 30m
    labels: { severity: ticket }
    annotations:
      summary: "SLO burn slow (30m window)"
```

Здесь множители (14.4, 6) — коэффициенты «скорости сгорания» (см. методику Google SRE). Настраивайте под свой бюджет и окно SLO.

## Масштабирование хранения: Thanos/Mimir/Cortex, remote_write, ретеншн и downsampling

Обычный Prometheus — **single-node** и локальное хранение. Для долговременной истории, федерации кластеров и высокой доступности используется **Thanos** (или Mimir/Cortex). Thanos добавляет слой блок-хранилища (S3/GCS), компакторы, кэш и **кросс-кластерный** query. При этом локальные Prometheus остаются «источниками правды» и работают независимо.

Если у вас SaaS-бэкенд метрик (Datadog/Mimir), рассмотрите **`remote_write`**: локальный Prometheus пушит метрики в удалённое хранилище. Это удобно для централизованной аналитики и длинной ретенции. Но **не полагайтесь только на remote_write**: для алёртов лучше держать локальные prom-инстансы, чтобы алёртинг не зависел от сети до SaaS.

Ретеншн — баланс стоимости и пользы. Часто: «деталь» 7–14 дней, «сжатие» (downsampling) на месяцы. Thanos Compactor умеет агрегировать ряды с крупным шагом, облегчая дальние запросы. Планируйте S3-бакеты, политики lifecycle (glacier/expiration), чтобы не взорвать счёт.

Примеры конфигов:

```yaml
# prometheus.yml
remote_write:
  - url: https://mimir.example/api/v1/push
    queue_config: { capacity: 5000, max_shards: 20, max_samples_per_send: 10000 }
```

```yaml
# thanos-compactor flags (k8s args)
args:
  - --objstore.config-file=/etc/thanos/objstore.yml
  - --retention.resolution-raw=30d
  - --retention.resolution-5m=180d
  - --retention.resolution-1h=2y
```

## Кардинальность под контролем: top-N, explorer, лимиты/квоты

В Prometheus каждая **уникальная комбинация лейблов** — отдельный тайм-ряд. «Взорвать» хранилище легко одной метрикой с лейблом `userId`. Введите процесс: ревью метрик, «чёрный список» лейблов, дашборд «cardinality explorer» (top-N метрик по числу серий), алёрты на рост `tsdb_head_series` и `prometheus_tsdb_wal_fsync_duration_seconds`.

На входе (scrape) режьте **ненужные лейблы** через `metric_relabel_configs`: можно удалить `labeldrop` или «свести» `uri` к шаблону (если ваша библиотека кладёт сырые пути). На стороне приложения используйте Micrometer `MeterFilter` для запрета «опасных» тегов.

Примеры:

```yaml
scrape_configs:
- job_name: apps
  metric_relabel_configs:
    - source_labels: [uri]
      regex: "/orders/.*"
      target_label: uri
      replacement: "/orders/{id}"
```

```java
@Bean MeterFilter guard() {
  return MeterFilter.deny(id -> id.getTag("userId") != null || id.getTag("sessionId") != null);
}
```

И держите **квоты**: лимит на общее число серий и на прирост. Это организационная мера — чтобы одна команда не «уронила» платформу всем.

---

# 5. Grafana: дашборды, алёрты, практики

## Дашборды «по роли»: SRE/инциденты, разработчик сервиса, бизнес-метрики

Грамотная графана — не один «супердашборд на всё». Делим по ролям: **SRE-панель инцидента** (RED/Golden Signals, карта зависимостей, состояние брейкеров, очереди, GC), **панель разработчика сервиса** (детальная разбивка по эндпоинтам/SQL/пулами), **бизнес-панель** (конверсия, скорость страниц, % успешных операций).

Соблюдайте **шаблонность**: одинаковые названия панелей, единицы (`s`, `ms`, `bytes`, `rps`), легенды (`method`, `uri`, `status`). Шаблон переменных (namespace, service, env) ускорит переключения между сервисами.

Старайтесь чтобы первая панель была «сводной»: 8–12 виджетов (RPS, p95, error rate, saturation по пулу, CB open, JVM heap, GC pause p95, DB borrow wait). Это даёт быстрый ответ на вопрос «где болит». Далее — вкладки или ряды с деталями.

Расположение — слева направо «бизнес → тех → инфра». На постмортемах такая структура экономит минуты (иногда — часы) анализа.

## Алёртинг: SLO-правила, тишина vs paging, dead-man switch

Правила алёртов делайте **по SLO**. Используйте MWMBR: короткие окна для пэйджинга («всё горит»), длинные — для тикетов («пошла деградация»). Избегайте «порогов без контекста» (например, «error rate > 5%»): в ночную низкую нагрузку это шумно, а днём — поздно. Алёрты должны быть «самодокументируемыми» — ссылаться на дашборды и runbook.

Режим **тишины** ночью для «низкоприоритетных» сервисов — нормальная практика: отправляйте тикеты вместо пэйджинга. Критические SLO — наоборот, всегда paging. Важно договориться на уровне on-call политики.

**Dead-man switch** — «пульс» от системы мониторинга; если он перестаёт приходить (Prometheus/Alertmanager упал), срабатывает критический алёрт на независимый канал. Это страховка от «мы всё выключили и думаем, что всё хорошо».

В Grafana Alerting (unified) теперь можно описывать правила как **rule groups** с несколькими выражениями и параметрами «no data / error». Документируйте поведение «no data» для каждой метрики (часто — `NoData = Alerting` для критических метрик).

## Drill-down: метрика→трейс (exemplars), метрика→логи (traceId в MDC)

Чтобы «кликнуть p99 → увидеть трейс», включите **exemplars**: Grafana при наведении на линию квантиля покажет точки-примеры; клик — и вы в Tempo/Jaeger на нужном спане. Это требует правильной связки Micrometer ↔ OTel и включенного exemplar-storage в Prometheus.

Связка «метрика→логи» делается через `traceId`/`spanId` в MDC и **Derived fields** в Grafana Loki: вы добавляете правило распознавания `trace_id=(\w+)` и настраиваете ссылку «в Tempo по этому trace_id». Тогда из таблицы логов вы прыгаете в трейс или наоборот — видите логи конкретного спана.

На панелях добавляйте **dashboard links**/**data links** к runbook, чтобы SRE не искал в wiki: кнопку «How to mitigate» сразу рядом с графиком CB open. Это снижает MTTR.

Пример Loki запроса по traceId:

```logql
{app="order"} |~ "trace_id=(?P<trace>[a-f0-9-]{16,32})"
```

## Отчётность: артефакты пост-мортема, автоссылки на runbooks

Хороший пост-мортем содержит **скриншоты** ключевых графиков, ссылки на трейсы, логи и git-коммиты релизов. В Grafana можно экспортировать панели/графики в PNG/PDF автоматически (Reporting) или через ссылку `render`.

Храните артефакты инцидентов в одном месте (ticket/Confluence), а в панели — добавляйте кнопки «Open incident template»/«Open last incident», чтобы ускорить документирование. Это дисциплинирует команду и делает ретро продуктивнее.

Runbooks должны быть живыми: меняются вместе с системой. Добавляйте в аннотации алёртов ссылки на конкретные разделы runbook («Circuit open: check dependency health, enable kill-switch feature flag»). Не отправляйте людей «в общий раздел», это теряет минуты.

Для повторяемости RCA полезно иметь «shared dashboard» для инцидентов: шаблон + переменные (`service`, `env`, `time range`). Это сокращает «сбор рабочей панели» во время пожара.

---

# 6. JVM/пулы/GC: «железная» часть метрик

## GC (G1/ZGC): паузы, частота, reclaimed bytes; heap/metaspace, allocation rate

Garbage Collector напрямую влияет на задержки. Для G1/ZGC интересны: **паузы (pause time)** — p50/p95/p99, частота циклов, **reclaimed bytes** (сколько памяти освобождалось), **allocation rate** (скорость аллокаций). В Micrometer/Actuator уже есть `jvm.gc.pause`, `jvm.memory.used`, `jvm.memory.max` — их достаточно для первых дашбордов.

Смотрите не только на абсолютные значения пауз, но и на **корреляцию**: рост p99 HTTP совпадает с пиками GC? Тогда оптимизируйте аллокации (пулы буферов, избегать боксинга, «тёплые» коллекции), настройте размеры регионов/heap для G1 или используйте ZGC при экстремальных требованиях к латентности.

Metaspace/Direct Buffer — часто забываемые зоны. Лейте метрики `jvm.buffer.*` и `jvm.memory.used{area="nonheap",id="Metaspace"}`. Утекающий Metaspace (динамическая генерация классов, heavy proxy) способен грохнуть процесс из-за OOM.

Для долгих «batch» задач добавьте `LongTaskTimer`. Он покажет «сколько одновременно» долгих операций и их длительность — легко поймать кейс «batch заполняет heap → GC начинает «пилить».

## Пулы потоков/очереди: size/active/queue depth/rejected; CallerRunsPolicy

Пулы потоков — «насосы» приложения. Обязательно снимайте метрики: `pool size`, `active`, `queue depth`, `completed`, `rejected`. Для `ThreadPoolTaskExecutor` доступны биндинги Micrometer; а если у вас свой `Executor`, сделайте `MeterBinder`.

Сигналы для алёртов: рост `queue depth`, появление `rejected` (переполнено), время ожидания коннекта в пулах HTTP/JDBC (borrow). Это часто первопричина лавин ретраев и провалов SLO.

`CallerRunsPolicy` — полезная стратегия деградации: когда очередь полна, выполняем задачу в текущем потоке. Это «ест» ресурсы вызывающего, но предотвращает потерю задач. Нужен контроль, чтобы не уронить веб-потоки — отделяйте пул «критикала»/фон.

Пример биндинга:

```java
@Bean
MeterBinder execMetrics(ThreadPoolTaskExecutor exec) {
  return registry -> {
    Gauge.builder("executor.queue.depth", exec, e -> e.getThreadPoolExecutor().getQueue().size())
        .tag("name", exec.getThreadNamePrefix()).register(registry);
    Gauge.builder("executor.pool.size", exec, e -> e.getPoolSize()).register(registry);
  };
}
```

## JFR/JMX как источники сигналов, интеграция с Micrometer/OTel

**Java Flight Recorder (JFR)** — профильщик с низким оверхедом. Его можно держать постоянно включённым (continuous) и выгружать события (GC, аллокации, lock contentions). OTel Java Agent умеет забирать часть JFR событий в трейс/метрики; есть и экспортеры JFR→Prometheus.

Запуск JFR в проде:

```bash
JAVA_TOOL_OPTIONS='-XX:StartFlightRecording=filename=/var/log/app.jfr,settings=profile,maxage=1d'
```

**JMX** — универсальный канал. Micrometer может публиковать метрики в JMX (для локальной диагностики/старых стеков). В обратную сторону — многие JVM показатели уже приходят в Micrometer через Actuator, так что самостоятельный JMX-poll часто не нужен.

Следите за оверхедом: не включайте «тяжёлые» JFR события постоянно. Для инцидентов — временно повышайте профиль. И заранее подготовьте инструменты выгрузки/просмотра (JDK Mission Control, jfr-parser).

## Нагрузочные бюджеты: читать saturations и планировать CPU/memory/IO

Модель USE (Utilization/Saturation/Errors) помогает планировать **мощности**. Utilization близка к 100% — ещё не беда, если Saturation (глубина очередей, задержки borrow) низкая. Но при росте Saturation и p95 задержек пора **скейлиться**: горизонтально (реплики) или выделять пулы.

Сведите «бюджеты» в одну панель: CPU%/load, GC pause, heap usage, queue depth, rejected, borrow wait, iowait. Добавьте аннотации «релизы» / «конфиг изменения» — это часто объясняет резкие сдвиги.

Оценивайте «стоимость» улучшений: уменьшили p95 → где «вышла» нагрузка? Может, увеличилось число ретраев или выросла конкуренция в другом пуле. Наблюдаемость должна помогать видеть **перетоки**.

Для планирования используйте **перцентильные** метрики, а не средние. И не забывайте про «тёплый» запуск: cold start спустя деплой — отдельный профиль.

---

# 7. Трейсинг: модель, семантика и контекст

## Спаны: серверный/клиентский, атрибуты (семантика OTel), events/status

Трейс — это дерево **спанов**: серверные (приём запроса), клиентские (исходящие вызовы), внутренние (бизнес-операции). Каждый спан несёт тайминги, **атрибуты** и **статус**. OpenTelemetry описывает **семантические конвенции**: `http.method`, `http.route`, `http.status_code`, `db.system`, `net.peer.name` и т.д. Старайтесь следовать им — это делает дашборды переносимыми.

**Events** — лёгкие «точки» внутри спана: «retried attempt=2 delay=150ms», «cache miss». Это лучше, чем плодить микроспаны на каждую мелочь. **Status** — OK/ERROR; ERROR ставьте только при реальной ошибке бизнес-операции, не маркируйте ERROR «каждый 404» (если для вас это норма).

Для бизнес-контекста добавляйте **низкокардинальные** атрибуты: `order.channel=web`, `tenant=eu1`. Не кладите PII или высокую кардинальность (userId) в атрибуты — оставляйте это логам.

Пример ручного спана с атрибутами и event:

```java
@Autowired Tracer tracer;

public Product getProduct(String id) {
  Span span = tracer.nextSpan().name("product:get").start();
  try (var s = span.makeCurrent()) {
    span.setAttribute("product.id", id);
    Product p = client.fetch(id);
    span.addEvent("cache_miss", Attributes.of(stringKey("key"), id));
    return p;
  } catch (Exception e) {
    span.recordException(e); span.setStatus(StatusCode.ERROR, "fetch failed"); throw e;
  } finally {
    span.end();
  }
}
```

## Пропагация: W3C traceparent/tracestate, baggage (минимум)

Контекст трейсинга переходит между сервисами через заголовки **W3C**: `traceparent`/`tracestate`. Большинство гейтвеев/прокси сейчас их поддерживают; OTel агент — тоже, «из коробки». Важно не «переписывать» и не «удалять» эти заголовки на границе.

**Baggage** — произвольные пары ключ/значение, которые путешествуют с трейсом. Используйте очень экономно: только для действительно необходимых «сквозных» атрибутов (например, `tenant`, `locale`), и только безопасные значения (без PII/секретов). Любой лишний baggage — это байты в каждом запросе.

В Spring Boot OTel агент автоматически включает W3C пропагацию. Если делаете ручную — используйте `Context`/`Baggage` из OTel API.

Пример чтения baggage:

```java
String tenant = io.opentelemetry.api.baggage.Baggage.current().getEntryValue("tenant");
```

## Ошибки и статус: где ставить error.flag, как не раздувать спаны логами

Правило: **один спан — одна ответственность**. Ошибка фиксируется там, где она произошла. Если клиентский HTTP вызов вернул 500 — пометьте **клиентский** спан `Status=ERROR` и добавьте `http.status_code=500`; серверный спан вашего эндпоинта может остаться OK (если вы корректно обработали ошибку и вернули 503 с fallback).

Не помещайте «больше логов в спан» — это увеличит размер и стоимость. Используйте **events** для ключевых моментов и оставляйте подробности логам (с тем же `traceId/spanId` в MDC). Включите **batch exporter** и лимиты на размер спана/атрибутов.

Исключения записывайте через `span.recordException(e)` — агент добавит тип/сообщение/stacktrace (зачастую усечённый). Статус ставьте явно: `span.setStatus(StatusCode.ERROR, "reason")`. Это критично для фильтров «покажи мне только ошибочные спаны».

## Инструментирование Spring: javaagent OTel и вручную через Observation/Tracer

Самый быстрый путь — **OTel Java Agent**: добавляете JVM-флаг и получаете автоспаны для HTTP сервер/клиент, JDBC, Redis, Kafka, gRPC и т.д. В Spring Boot 3 Micrometer Observation уже настраивает мост между метриками и трейсом.

Запуск агента:

```bash
java -javaagent:/otel/opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.traces.exporter=otlp \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
  -Dotel.resource.attributes=deployment.environment=prod,service.version=1.12.3 \
  -jar app.jar
```

Если нужно **обогатить** трейс доменными атрибутами — используйте `@Observed`/`Observation` (создаст span+timer) или `Tracer` (как выше). Это дополняет агент, не конфликтует. Для реактивности проверьте, что включена **context propagation** (Micrometer Context Propagation/OTel bridge).

Конфиг Spring Boot для экспорта в Collector:

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # head-sampling 10%
  otlp:
    tracing:
      endpoint: http://otel-collector:4318
```

отлично — продолжаем «Наблюдаемость (расширено): метрики и трейсинг» и подробно раскрываем **подтемы 4–7**. В каждом подпункте — минимум четыре абзаца, плюс практичные конфиги и фрагменты кода (Spring Boot 3.x, Micrometer/Actuator, OpenTelemetry, Prometheus/Grafana, Kubernetes).

---

# 4. Прометеевский стек и правила эксплуатации

## Scrape-модель: интервалы, timeouts, relabeling, Service/PodMonitor (Operator)

Prometheus работает по **pull**-модели: сам ходит на таргеты и собирает метрики по HTTP. Два ключевых параметра — `scrape_interval` (как часто забирать) и `scrape_timeout` (максимальное время на ответ). Практичный дефолт: 15s/10s для приложений и 30s/25s для «тяжёлых» таргетов (брокеры, базы). Важно, чтобы `scrape_timeout < scrape_interval`, иначе вы можете «наложить» сборы друг на друга и создать лавинообразную нагрузку на приложение.

**Relabeling** — мощный фильтр на этапе обнаружения и сбора: им можно переименовывать лейблы, отбрасывать шумные таргеты, нормализовать имена. Например, в Kubernetes часто выкидывают «эпемерные» лейблы (`pod_template_hash`), чтобы они не попадали в метрики. Это снижает кардинальность и упрощает дашборды.

В Kubernetes лучший способ подключения — **Prometheus Operator** (или kube-prometheus-stack). Вместо ручного добавления таргетов вы описываете `ServiceMonitor`/`PodMonitor`: это Custom Resource Definitions (CRD), которые «объясняют» Prometheus, какие сервисы и на каких портах/путях скрапить. `ServiceMonitor` смотрит на **Service**, `PodMonitor` — на **Pod**. Для Spring Boot надо просто экспонировать `/actuator/prometheus`.

Мини-пример конфигов:

```yaml
# values.yaml для Prometheus (фрагмент)
global:
  scrape_interval: 15s
  scrape_timeout: 10s

# Service + ServiceMonitor для Spring Boot
apiVersion: v1
kind: Service
metadata:
  name: order-svc
  labels: { app: order }
spec:
  selector: { app: order }
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-sm
spec:
  selector:
    matchLabels: { app: order }
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
      scrapeTimeout: 10s
      scheme: http
```

И пример relabel, отбрасывающий лишние лейблы:

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_pod_template_hash]
    action: labeldrop
```

## Правила: recording rules и MWMBR-алёрты по SLO

**Recording rules** позволяют предварительно агрегировать тяжёлые запросы в новые тайм-ряды и не считать их «на лету» на каждом графике и алёрте. Типичный пример — квантили по гистограммам (`histogram_quantile`), rate по тысячам серий и свёртки в «SLI готовые для алёртов». Это разгружаетTSDB и делает алёрты стабильными.

Для SLO используем **multi-window multi-burn-rate (MWMBR)**: правила «сгорания бюджета» ошибки одновременно на коротких и длинных окнах (например, 5m/30m/2h/6h). Если на коротком окне всё плохо — это «пейдж», на длинном — «тикет/наблюдение». Так вы ловите и резкие всплески, и затяжные деградации.

Пример набора правил: сначала SLI, потом алёрты MWMBR. Предположим, SLI успешности — доля 2xx/3xx (исключая 5xx) по `http_server_requests_seconds_count`.

```yaml
# recording rules
groups:
- name: slo:records
  interval: 30s
  rules:
  - record: job:http_request_total:rate5m
    expr: sum by (job)(rate(http_server_requests_seconds_count[5m]))
  - record: job:http_success_total:rate5m
    expr: sum by (job)(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
  - record: job:sli_availability:ratio5m
    expr: job:http_success_total:rate5m / job:http_request_total:rate5m
```

И сами алёрты по SLO 99.9%:

```yaml
groups:
- name: slo:alerts
  rules:
  # 5m/1h
  - alert: SLOErrorBudgetBurn
    expr: (1 - job:sli_availability:ratio5m) > (1 - 0.999) * 14.4
    for: 5m
    labels: { severity: page }
    annotations:
      summary: "SLO burn fast (5m window)"
  # 30m/6h
  - alert: SLOErrorBudgetBurnSlow
    expr: (1 - avg_over_time(job:sli_availability:ratio5m[30m])) > (1 - 0.999) * 6
    for: 30m
    labels: { severity: ticket }
    annotations:
      summary: "SLO burn slow (30m window)"
```

Здесь множители (14.4, 6) — коэффициенты «скорости сгорания» (см. методику Google SRE). Настраивайте под свой бюджет и окно SLO.

## Масштабирование хранения: Thanos/Mimir/Cortex, remote_write, ретеншн и downsampling

Обычный Prometheus — **single-node** и локальное хранение. Для долговременной истории, федерации кластеров и высокой доступности используется **Thanos** (или Mimir/Cortex). Thanos добавляет слой блок-хранилища (S3/GCS), компакторы, кэш и **кросс-кластерный** query. При этом локальные Prometheus остаются «источниками правды» и работают независимо.

Если у вас SaaS-бэкенд метрик (Datadog/Mimir), рассмотрите **`remote_write`**: локальный Prometheus пушит метрики в удалённое хранилище. Это удобно для централизованной аналитики и длинной ретенции. Но **не полагайтесь только на remote_write**: для алёртов лучше держать локальные prom-инстансы, чтобы алёртинг не зависел от сети до SaaS.

Ретеншн — баланс стоимости и пользы. Часто: «деталь» 7–14 дней, «сжатие» (downsampling) на месяцы. Thanos Compactor умеет агрегировать ряды с крупным шагом, облегчая дальние запросы. Планируйте S3-бакеты, политики lifecycle (glacier/expiration), чтобы не взорвать счёт.

Примеры конфигов:

```yaml
# prometheus.yml
remote_write:
  - url: https://mimir.example/api/v1/push
    queue_config: { capacity: 5000, max_shards: 20, max_samples_per_send: 10000 }
```

```yaml
# thanos-compactor flags (k8s args)
args:
  - --objstore.config-file=/etc/thanos/objstore.yml
  - --retention.resolution-raw=30d
  - --retention.resolution-5m=180d
  - --retention.resolution-1h=2y
```

## Кардинальность под контролем: top-N, explorer, лимиты/квоты

В Prometheus каждая **уникальная комбинация лейблов** — отдельный тайм-ряд. «Взорвать» хранилище легко одной метрикой с лейблом `userId`. Введите процесс: ревью метрик, «чёрный список» лейблов, дашборд «cardinality explorer» (top-N метрик по числу серий), алёрты на рост `tsdb_head_series` и `prometheus_tsdb_wal_fsync_duration_seconds`.

На входе (scrape) режьте **ненужные лейблы** через `metric_relabel_configs`: можно удалить `labeldrop` или «свести» `uri` к шаблону (если ваша библиотека кладёт сырые пути). На стороне приложения используйте Micrometer `MeterFilter` для запрета «опасных» тегов.

Примеры:

```yaml
scrape_configs:
- job_name: apps
  metric_relabel_configs:
    - source_labels: [uri]
      regex: "/orders/.*"
      target_label: uri
      replacement: "/orders/{id}"
```

```java
@Bean MeterFilter guard() {
  return MeterFilter.deny(id -> id.getTag("userId") != null || id.getTag("sessionId") != null);
}
```

И держите **квоты**: лимит на общее число серий и на прирост. Это организационная мера — чтобы одна команда не «уронила» платформу всем.

---

# 5. Grafana: дашборды, алёрты, практики

## Дашборды «по роли»: SRE/инциденты, разработчик сервиса, бизнес-метрики

Грамотная графана — не один «супердашборд на всё». Делим по ролям: **SRE-панель инцидента** (RED/Golden Signals, карта зависимостей, состояние брейкеров, очереди, GC), **панель разработчика сервиса** (детальная разбивка по эндпоинтам/SQL/пулами), **бизнес-панель** (конверсия, скорость страниц, % успешных операций).

Соблюдайте **шаблонность**: одинаковые названия панелей, единицы (`s`, `ms`, `bytes`, `rps`), легенды (`method`, `uri`, `status`). Шаблон переменных (namespace, service, env) ускорит переключения между сервисами.

Старайтесь чтобы первая панель была «сводной»: 8–12 виджетов (RPS, p95, error rate, saturation по пулу, CB open, JVM heap, GC pause p95, DB borrow wait). Это даёт быстрый ответ на вопрос «где болит». Далее — вкладки или ряды с деталями.

Расположение — слева направо «бизнес → тех → инфра». На постмортемах такая структура экономит минуты (иногда — часы) анализа.

## Алёртинг: SLO-правила, тишина vs paging, dead-man switch

Правила алёртов делайте **по SLO**. Используйте MWMBR: короткие окна для пэйджинга («всё горит»), длинные — для тикетов («пошла деградация»). Избегайте «порогов без контекста» (например, «error rate > 5%»): в ночную низкую нагрузку это шумно, а днём — поздно. Алёрты должны быть «самодокументируемыми» — ссылаться на дашборды и runbook.

Режим **тишины** ночью для «низкоприоритетных» сервисов — нормальная практика: отправляйте тикеты вместо пэйджинга. Критические SLO — наоборот, всегда paging. Важно договориться на уровне on-call политики.

**Dead-man switch** — «пульс» от системы мониторинга; если он перестаёт приходить (Prometheus/Alertmanager упал), срабатывает критический алёрт на независимый канал. Это страховка от «мы всё выключили и думаем, что всё хорошо».

В Grafana Alerting (unified) теперь можно описывать правила как **rule groups** с несколькими выражениями и параметрами «no data / error». Документируйте поведение «no data» для каждой метрики (часто — `NoData = Alerting` для критических метрик).

## Drill-down: метрика→трейс (exemplars), метрика→логи (traceId в MDC)

Чтобы «кликнуть p99 → увидеть трейс», включите **exemplars**: Grafana при наведении на линию квантиля покажет точки-примеры; клик — и вы в Tempo/Jaeger на нужном спане. Это требует правильной связки Micrometer ↔ OTel и включенного exemplar-storage в Prometheus.

Связка «метрика→логи» делается через `traceId`/`spanId` в MDC и **Derived fields** в Grafana Loki: вы добавляете правило распознавания `trace_id=(\w+)` и настраиваете ссылку «в Tempo по этому trace_id». Тогда из таблицы логов вы прыгаете в трейс или наоборот — видите логи конкретного спана.

На панелях добавляйте **dashboard links**/**data links** к runbook, чтобы SRE не искал в wiki: кнопку «How to mitigate» сразу рядом с графиком CB open. Это снижает MTTR.

Пример Loki запроса по traceId:

```logql
{app="order"} |~ "trace_id=(?P<trace>[a-f0-9-]{16,32})"
```

## Отчётность: артефакты пост-мортема, автоссылки на runbooks

Хороший пост-мортем содержит **скриншоты** ключевых графиков, ссылки на трейсы, логи и git-коммиты релизов. В Grafana можно экспортировать панели/графики в PNG/PDF автоматически (Reporting) или через ссылку `render`.

Храните артефакты инцидентов в одном месте (ticket/Confluence), а в панели — добавляйте кнопки «Open incident template»/«Open last incident», чтобы ускорить документирование. Это дисциплинирует команду и делает ретро продуктивнее.

Runbooks должны быть живыми: меняются вместе с системой. Добавляйте в аннотации алёртов ссылки на конкретные разделы runbook («Circuit open: check dependency health, enable kill-switch feature flag»). Не отправляйте людей «в общий раздел», это теряет минуты.

Для повторяемости RCA полезно иметь «shared dashboard» для инцидентов: шаблон + переменные (`service`, `env`, `time range`). Это сокращает «сбор рабочей панели» во время пожара.

---

# 6. JVM/пулы/GC: «железная» часть метрик

## GC (G1/ZGC): паузы, частота, reclaimed bytes; heap/metaspace, allocation rate

Garbage Collector напрямую влияет на задержки. Для G1/ZGC интересны: **паузы (pause time)** — p50/p95/p99, частота циклов, **reclaimed bytes** (сколько памяти освобождалось), **allocation rate** (скорость аллокаций). В Micrometer/Actuator уже есть `jvm.gc.pause`, `jvm.memory.used`, `jvm.memory.max` — их достаточно для первых дашбордов.

Смотрите не только на абсолютные значения пауз, но и на **корреляцию**: рост p99 HTTP совпадает с пиками GC? Тогда оптимизируйте аллокации (пулы буферов, избегать боксинга, «тёплые» коллекции), настройте размеры регионов/heap для G1 или используйте ZGC при экстремальных требованиях к латентности.

Metaspace/Direct Buffer — часто забываемые зоны. Лейте метрики `jvm.buffer.*` и `jvm.memory.used{area="nonheap",id="Metaspace"}`. Утекающий Metaspace (динамическая генерация классов, heavy proxy) способен грохнуть процесс из-за OOM.

Для долгих «batch» задач добавьте `LongTaskTimer`. Он покажет «сколько одновременно» долгих операций и их длительность — легко поймать кейс «batch заполняет heap → GC начинает «пилить».

## Пулы потоков/очереди: size/active/queue depth/rejected; CallerRunsPolicy

Пулы потоков — «насосы» приложения. Обязательно снимайте метрики: `pool size`, `active`, `queue depth`, `completed`, `rejected`. Для `ThreadPoolTaskExecutor` доступны биндинги Micrometer; а если у вас свой `Executor`, сделайте `MeterBinder`.

Сигналы для алёртов: рост `queue depth`, появление `rejected` (переполнено), время ожидания коннекта в пулах HTTP/JDBC (borrow). Это часто первопричина лавин ретраев и провалов SLO.

`CallerRunsPolicy` — полезная стратегия деградации: когда очередь полна, выполняем задачу в текущем потоке. Это «ест» ресурсы вызывающего, но предотвращает потерю задач. Нужен контроль, чтобы не уронить веб-потоки — отделяйте пул «критикала»/фон.

Пример биндинга:

```java
@Bean
MeterBinder execMetrics(ThreadPoolTaskExecutor exec) {
  return registry -> {
    Gauge.builder("executor.queue.depth", exec, e -> e.getThreadPoolExecutor().getQueue().size())
        .tag("name", exec.getThreadNamePrefix()).register(registry);
    Gauge.builder("executor.pool.size", exec, e -> e.getPoolSize()).register(registry);
  };
}
```

## JFR/JMX как источники сигналов, интеграция с Micrometer/OTel

**Java Flight Recorder (JFR)** — профильщик с низким оверхедом. Его можно держать постоянно включённым (continuous) и выгружать события (GC, аллокации, lock contentions). OTel Java Agent умеет забирать часть JFR событий в трейс/метрики; есть и экспортеры JFR→Prometheus.

Запуск JFR в проде:

```bash
JAVA_TOOL_OPTIONS='-XX:StartFlightRecording=filename=/var/log/app.jfr,settings=profile,maxage=1d'
```

**JMX** — универсальный канал. Micrometer может публиковать метрики в JMX (для локальной диагностики/старых стеков). В обратную сторону — многие JVM показатели уже приходят в Micrometer через Actuator, так что самостоятельный JMX-poll часто не нужен.

Следите за оверхедом: не включайте «тяжёлые» JFR события постоянно. Для инцидентов — временно повышайте профиль. И заранее подготовьте инструменты выгрузки/просмотра (JDK Mission Control, jfr-parser).

## Нагрузочные бюджеты: читать saturations и планировать CPU/memory/IO

Модель USE (Utilization/Saturation/Errors) помогает планировать **мощности**. Utilization близка к 100% — ещё не беда, если Saturation (глубина очередей, задержки borrow) низкая. Но при росте Saturation и p95 задержек пора **скейлиться**: горизонтально (реплики) или выделять пулы.

Сведите «бюджеты» в одну панель: CPU%/load, GC pause, heap usage, queue depth, rejected, borrow wait, iowait. Добавьте аннотации «релизы» / «конфиг изменения» — это часто объясняет резкие сдвиги.

Оценивайте «стоимость» улучшений: уменьшили p95 → где «вышла» нагрузка? Может, увеличилось число ретраев или выросла конкуренция в другом пуле. Наблюдаемость должна помогать видеть **перетоки**.

Для планирования используйте **перцентильные** метрики, а не средние. И не забывайте про «тёплый» запуск: cold start спустя деплой — отдельный профиль.

---

# 7. Трейсинг: модель, семантика и контекст

## Спаны: серверный/клиентский, атрибуты (семантика OTel), events/status

Трейс — это дерево **спанов**: серверные (приём запроса), клиентские (исходящие вызовы), внутренние (бизнес-операции). Каждый спан несёт тайминги, **атрибуты** и **статус**. OpenTelemetry описывает **семантические конвенции**: `http.method`, `http.route`, `http.status_code`, `db.system`, `net.peer.name` и т.д. Старайтесь следовать им — это делает дашборды переносимыми.

**Events** — лёгкие «точки» внутри спана: «retried attempt=2 delay=150ms», «cache miss». Это лучше, чем плодить микроспаны на каждую мелочь. **Status** — OK/ERROR; ERROR ставьте только при реальной ошибке бизнес-операции, не маркируйте ERROR «каждый 404» (если для вас это норма).

Для бизнес-контекста добавляйте **низкокардинальные** атрибуты: `order.channel=web`, `tenant=eu1`. Не кладите PII или высокую кардинальность (userId) в атрибуты — оставляйте это логам.

Пример ручного спана с атрибутами и event:

```java
@Autowired Tracer tracer;

public Product getProduct(String id) {
  Span span = tracer.nextSpan().name("product:get").start();
  try (var s = span.makeCurrent()) {
    span.setAttribute("product.id", id);
    Product p = client.fetch(id);
    span.addEvent("cache_miss", Attributes.of(stringKey("key"), id));
    return p;
  } catch (Exception e) {
    span.recordException(e); span.setStatus(StatusCode.ERROR, "fetch failed"); throw e;
  } finally {
    span.end();
  }
}
```

## Пропагация: W3C traceparent/tracestate, baggage (минимум)

Контекст трейсинга переходит между сервисами через заголовки **W3C**: `traceparent`/`tracestate`. Большинство гейтвеев/прокси сейчас их поддерживают; OTel агент — тоже, «из коробки». Важно не «переписывать» и не «удалять» эти заголовки на границе.

**Baggage** — произвольные пары ключ/значение, которые путешествуют с трейсом. Используйте очень экономно: только для действительно необходимых «сквозных» атрибутов (например, `tenant`, `locale`), и только безопасные значения (без PII/секретов). Любой лишний baggage — это байты в каждом запросе.

В Spring Boot OTel агент автоматически включает W3C пропагацию. Если делаете ручную — используйте `Context`/`Baggage` из OTel API.

Пример чтения baggage:

```java
String tenant = io.opentelemetry.api.baggage.Baggage.current().getEntryValue("tenant");
```

## Ошибки и статус: где ставить error.flag, как не раздувать спаны логами

Правило: **один спан — одна ответственность**. Ошибка фиксируется там, где она произошла. Если клиентский HTTP вызов вернул 500 — пометьте **клиентский** спан `Status=ERROR` и добавьте `http.status_code=500`; серверный спан вашего эндпоинта может остаться OK (если вы корректно обработали ошибку и вернули 503 с fallback).

Не помещайте «больше логов в спан» — это увеличит размер и стоимость. Используйте **events** для ключевых моментов и оставляйте подробности логам (с тем же `traceId/spanId` в MDC). Включите **batch exporter** и лимиты на размер спана/атрибутов.

Исключения записывайте через `span.recordException(e)` — агент добавит тип/сообщение/stacktrace (зачастую усечённый). Статус ставьте явно: `span.setStatus(StatusCode.ERROR, "reason")`. Это критично для фильтров «покажи мне только ошибочные спаны».

## Инструментирование Spring: javaagent OTel и вручную через Observation/Tracer

Самый быстрый путь — **OTel Java Agent**: добавляете JVM-флаг и получаете автоспаны для HTTP сервер/клиент, JDBC, Redis, Kafka, gRPC и т.д. В Spring Boot 3 Micrometer Observation уже настраивает мост между метриками и трейсом.

Запуск агента:

```bash
java -javaagent:/otel/opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.traces.exporter=otlp \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
  -Dotel.resource.attributes=deployment.environment=prod,service.version=1.12.3 \
  -jar app.jar
```

Если нужно **обогатить** трейс доменными атрибутами — используйте `@Observed`/`Observation` (создаст span+timer) или `Tracer` (как выше). Это дополняет агент, не конфликтует. Для реактивности проверьте, что включена **context propagation** (Micrometer Context Propagation/OTel bridge).

Конфиг Spring Boot для экспорта в Collector:

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # head-sampling 10%
  otlp:
    tracing:
      endpoint: http://otel-collector:4318
```

---

хочешь — продолжу **подтемы 8–10**: пайплайн трейсинга и sampling (Collector, head/tail-sampling, лимиты и безопасность), связь метрик↔трейсов↔логов (exemplars, span-metrics, MDC), Kubernetes & governance (Operator, мультикластер, стандарты, runbooks, экономика). Также могу приложить мини-репозиторий: Spring Boot + Actuator/Micrometer, Prometheus scrape, OTel agent+Collector (Tempo/Loki/Grafana), примеры `@Observed`/Tracer и готовые правила SLO (MWMBR).

