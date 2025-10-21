---
layout: page
title: "Устойчивость: таймауты, ретраи, circuit breaker"
permalink: /spring/stability_resilience
---

# 0. Введение

## Что это всё такое

Устойчивость (resilience) — это набор технических приёмов, которые позволяют системе оставаться работоспособной при частичных отказах и деградациях зависимостей. В веб-сервисах это прежде всего защита от сетевых задержек и сбоев внешних API/БД, чтобы ваши SLA не разваливались из-за «чужих» проблем. Базовые кирпичики: **таймауты** (ограничение времени операции), **ретраи** (повтор временной ошибки), **circuit breaker** (размыкатель цепи при массовых сбоях), **bulkhead** (ограничение параллелизма/очередей), **rate limiter** (ограничитель скорости) и **fallback** (деградация функциональности).

Важно разделять «функциональную корректность» и «техническую устойчивость». Первое — чтобы бизнес-результат верен; второе — чтобы даже при сбоях вы отвечали предсказуемо и быстро (пусть и упрощённо). Устойчивость не делает систему «магически правильной», она делает её **предсказуемой под стрессом**.

В современном Spring-стеке стандартом стала библиотека **Resilience4j**: набор лёгких декораторов для таймаутов, ретраев, брейкеров, bulkhead и rate limiting, с Micrometer-метриками и Spring Boot-автоконфигом. Она не завязана на потоковой модели (работает и в MVC, и в WebFlux) и хорошо конфигурируется из `application.yml`.

Мини-пример композиции политик вокруг HTTP-вызова:

```java
var retry = Retry.ofDefaults("ext");
var cb    = CircuitBreaker.ofDefaults("ext");
var tl    = TimeLimiter.of(Duration.ofMillis(800));

Supplier<String> call = () -> webClient.get().uri("/ext").retrieve().bodyToMono(String.class).block();
var guarded = Decorators.ofSupplier(call)
  .withTimeLimiter(tl, Executors.newSingleThreadScheduledExecutor())
  .withRetry(retry)
  .withCircuitBreaker(cb)
  .decorate();

String body = Try.ofSupplier(guarded).get(); // io.vavr.Try
```

## Зачем

Зачем вообще эта сложность? Во-первых, чтобы **сдержать каскадные отказы**. Один зависший интеграционный вызов без таймаута способен подвесить поток, забить пул соединений, остановить приём новых запросов — и вы «ложитесь» вместе с зависимостью. Таймауты и брейкеры отрезают «больное место», сохраняя остальную систему в строю.

Во-вторых, чтобы **стабилизировать задержки**. Даже редкие «длинные хвосты» (p99) без таймаутов и ретраев могут убить SLA. Устойчивость как раз про то, чтобы «узкие места» ограничивать по времени и заменять частичным ответом или кэшем.

В-третьих, чтобы **снизить стоимость инцидентов**. Если политик хватает, чтобы вернуть корректный код ошибки/частичный результат быстро, нагрузка на поддержку и MTTR падают. Это не отменяет исправление корня, но делает инциденты менее разрушительными.

И, наконец, устойчивость — это **контракт с клиентом**. Он должен знать, чего ждать: «если зависимость недоступна, получишь 503 за ≤800 мс, а UI предложит ретрай позже». Это лучше, чем «крутилка бесконечности».

## Где используется

Практически везде, где есть сеть: внешние платёжные/почтовые/гейтовые API, межсервисные HTTP-вызовы, JDBC/R2DBC к базе, Redis/кеш, очереди сообщений. Даже внутри одного дата-центра встречаются задержки, GC-паузы, DNS-глюки — устойчивость нужна не только «на границе».

В MVC-приложениях вы используете блокирующие клиенты (RestTemplate/Feign/OkHttp/JDBC) — значит, жизненно важны **таймауты сокетов и пулов**, а также **ограничение параллелизма** (bulkhead). В WebFlux — не блокируете event-loop, работаете через WebClient и **reactive**-варианты декораторов.

На уровне фронта и гейтвеев — свои политики: **rate limiting** по пользователю/токену, **stale-while-revalidate** на CDN, быстрые отказоустойчивые страницы. Ровно те же идеи, только ближе к пользователю.

В data-плоскости — **queryTimeout** в JDBC/драйверах, дедлайны на агрегации, ретраи только на идемпотентных чтениях. Даже внутри «простой» БД-транзакции может быть внешний вызов — его обязательно оборачивать.

## Какие проблемы решает

Перечислим типовые боли: (1) «полумёртвые» зависимости, отвечающие по 10–30 секунд — лечится таймаутами, (2) «шторма» ретраев — лечатся backoff+джиттером и брейкером, (3) истощение пулов соединений — лечится лимитами, bulkhead и быстрым fail-fast, (4) «залипшие» потоки — лечатся отменой и правильной очисткой ресурсов.

Другая проблема — **обратная связь**: ваши ретраи/брейкеры могут сами усугублять чужую деградацию. Поэтому устойчивость подразумевает «вежливость»: ограничивайте скорость (rate limit), уважайте `Retry-After`, используйте экспоненциальный backoff с джиттером.

Ещё одна — **несогласованность** при повторах. Без идемпотентности ретраи приводят к дублям/двойным списаниям. Правильный дизайн (Idempotency-Key, дедуп в БД) — часть устойчивости.

И наконец — **наблюдаемость**. Без метрик/трейсинга устойчивость превращается в «набат»: нужно видеть slow-call rate, состояние брейкеров, глубину очередей и rejected.

---

# 1. Базовые принципы устойчивости

## Failure modes: сетевые задержки, «полумёртвые» зависимости, истощение пулов, GC-паузы

Сбои бывают разными. Самые частые — **сетевые задержки и потери пакетов**: TCP-ретрансмиссии, перегруженные каналы, «шум» от соседей. Это проявляется как «иногда очень долго» (длинные хвосты), а не как ровный таймаут.

Есть режим **half-alive** («полумёртвая» зависимость): сокеты устанавливаются, но ответы приходят слишком поздно или с 5xx. Такие деградации особенно коварны — они медленно «засасывают» ваши пулы потоков и соединений.

**Истощение пулов** — результат «зависших» операций. Потоки, занятые ожиданием, не обслуживают новые запросы, очередь растёт, CPU простаивает, пользователи получают 503. Это лечится «крышкой» на время операции и вместимостью очередей.

Наконец, **GC-паузы** и stop-the-world события в JVM. Даже в современном G1/ZGC возможны кратковременные задержки: они срывают SLA и «зажёвывают» таймауты. Поэтому наблюдайте GC-метрики и держите таймауты с запасом.

## Fail-fast и деградация вместо «подвисаний»

Принцип **fail-fast**: лучше быстро вернуть ошибку/частичный ответ, чем «держать» пользователя в ожидании. Это сохраняет ресурсы и защищает систему от лавин. Ошибка, возвращённая за 200–800 мс, часто «лучше», чем «висение» 15 секунд.

**Деградация** (graceful degradation) — заранее продуманные «ступени вниз»: вернуть кэш/часть данных, отключить необязательные виджеты, показать статическую плашку «повторите позже». Это архитектурная, а не «хакерская» история: проектируйте, что можно временно упростить.

Fail-fast не про «сдаться сразу». Он про **ограничение** времени и попыток в заданный бюджет. Например, 1 ретрай с backoff и всё. А дальше — fallback и телеметрия.

В Spring удобно описывать fallback либо через `Try.recover`, либо через аннотации Resilience4j с `fallbackMethod`. Главное — не «глушить» исключение, а маппить его в понятный пользователю ответ (HTTP 503 + ProblemDetail).

```java
@CircuitBreaker(name = "ext", fallbackMethod = "fallback")
public Product fetchProduct(String id) { ... }

public Product fallback(String id, Throwable ex) {
  return Product.partial(id); // частичный ответ
}
```

## Deadline propagation: общий тайм-бюджет и протаскивание вниз

**Дедлайн** — абсолютный срок «когда нужно закончить всё». Он важнее, чем «сложение отдельных таймаутов». Вы задаёте бюджет на весь запрос (скажем, 800 мс), и **каждый вызов внутри** получает «остаток» (например, 400 мс на сервис A, 250 мс на сервис B).

Технически это делается через **контекст**: заголовок `X-Deadline-Ms`/`X-Request-Timeout` и ThreadLocal/Reactor Context с Instant дедлайна. В каждом клиенте таймаут выставляется как `Duration.between(now, deadline)`, но с нижней «планкой» (минимум 50–100 мс).

Прелесть дедлайнов — они «схлопывают» многослойные таймауты в **общий бюджет**. В противном случае легко получить «перекрывающиеся» ожидания (например, три по 1 секунде) и вылет за SLO.

В Spring MVC можно сделать `HandlerInterceptor`, который вычисляет дедлайн из внешнего SLA; в WebFlux — `WebFilter` с `Context`. На клиентах (WebClient/Feign/JDBC) — выставлять оставшееся время.

```java
// утилита
public static Duration remainingDeadline() {
  Instant dl = DeadlineContext.get(); // ThreadLocal/Context
  return Duration.between(Instant.now(), dl).clampedTo(Duration.ofMillis(50), Duration.ofSeconds(5));
}
```

## Идемпотентность как фундамент для ретраев

Ретрай безопасен только если операция **идемпотентна**: повтор приводит к тому же эффекту. GET/PUT/DELETE обычно ок; POST — нет, если создаёт новый ресурс. Решение — **Idempotency-Key** в заголовке и дедупликация на сервере.

Держите таблицу **processed_ops(op_key PK, processed_at)** и попытка «вставить» ключ второй раз приводит к `DuplicateKeyException` — вы возвращаете тот же ответ (или говорите, что уже обработано). Это дешёвый и надёжный способ «не задвоить».

В очередях (Kafka/JMS) применяйте «Inbox»: список обработанных сообщений (по ключу/offset) с TTL. Это заменяет «exactly-once», которое в общем случае недостижимо.

Думайте про идемпотентность **в домене**: например, «применить статус X, если версия Y актуальна»; «обновить цену по версии Z». Это лучше, чем «повторить ещё раз и надеяться».

```sql
create table processed_ops(
  op_key text primary key,
  processed_at timestamptz default now()
);
```

---

# 2. Таймауты: таксономия и бюджет

## Клиентские таймауты: HTTP/БД/кеш

В HTTP есть как минимум **connect**, **read** и **write** таймауты. Connect — время на установку TCP/TLS; read — ожидание байтов ответа; write — отправка запроса (важно для больших тел). Все три должны быть настроены; дефолты библиотек часто слишком велики.

В JDBC — `setQueryTimeout`/`Statement.setQueryTimeout` (секунды) и таймауты пула (borrow, maxLifetime). В R2DBC — таймауты на уровне драйвера/клиента. Для Redis (Lettuce) — `commandTimeout` и таймауты коннекта/пула.

В Spring WebClient таймаут «логически» задают через `.timeout(Duration)` (реактивная отмена), но сетевые таймауты тоже важны: их нужно выставить на уровне `HttpClient` (Reactor Netty).

Пример базовых настроек:

```java
HttpClient hc = HttpClient.create()
  .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 300)
  .responseTimeout(Duration.ofMillis(700));

WebClient wc = WebClient.builder()
  .clientConnector(new ReactorClientHttpConnector(hc))
  .build();
```

## Абсолютные дедлайны vs per-hop; «внутренние < внешний SLO»

Правило: внутренние таймауты должны быть **строже**, чем внешний SLO. Если SLA ответа — 1000 мс, то суммарный бюджет всех внутренних шагов — 800–900 мс, чтобы осталось на логирование/сериализацию.

Абсолютный дедлайн помогает избежать «накопления» задержек: если после двух шагов осталось 200 мс, то третий шаг получает не «свои 500», а **оставшиеся 200**. Это уменьшает число «перерасходов» времени.

Per-hop таймауты полезны как «плановые»: например, для платежной системы — «внешний банк не должен занимать >400 мс». Но и они должны подчиняться дедлайну.

Делайте **матрицу**: класс операции × зависимость → «верхний потолок» времени. Это превращается в конфиг Resilience4j/клиента.

## Отмена: cooperative cancellation, cleanup

В JVM отмена **кооперативная**. В блокирующем коде — `Thread.interrupt()` и периодические проверки `Thread.interrupted()`. В реактивном — `dispose()`/`timeout()` завершает подписку и закрывает TCP.

Важно **освобождать ресурсы**: закрывать `Response`, `InputStream`, JDBC-`Statement`. Не оставляйте «висячие» сокеты — они выжрут файл-дескрипторы.

В `CompletableFuture` есть `orTimeout/completeOnTimeout/cancel`. Но отмена работает, только если внутри код уважает прерывания/исключения таймаута.

Где возможно — замыкайте ресурсы в `try-with-resources` и не ловите/глушите `InterruptedException`.

```java
var fut = CompletableFuture.supplyAsync(this::slowCall);
scheduler.schedule(() -> fut.cancel(true), 800, MILLISECONDS);
```

## Тюнинг под классы операций

Не бывает «универсального» таймаута. Для **read-only** вызовов — короткие (200–800 мс), для «тяжёлых» мутаций — длиннее, но ограниченные (например, ≤2–3 сек). Для внутренних очередей — свой бюджет.

Учитывайте **хвосты**: если p95=120 мс, p99 может быть 500 мс. Таймаут 200 мс даст ложные отказы; 800 мс — может скрыть реальные беды. Нужен компромисс, обычно p99×1.2.

Не забывайте про **сетевые страны**/регионы: межрегиональные вызовы — отдельные профили таймаутов или вовсе запрет.

Разделяйте клиенты по **use-case**: не один «общий» WebClient, а несколько бинов с разными таймаутами/пулами.

```yaml
resilience4j.timelimiter:
  instances:
    ext-read:  timeout-duration: 800ms
    ext-write: timeout-duration: 2s
```

---

# 3. Ретраи: когда можно, когда нельзя

## Классификация ошибок: transient vs permanent

Ретрай уместен для **временных** сбоев: 408/429/5xx, таймауты, «connection reset». Для **постоянных** (4xx: 400/401/403/404/422) — нельзя: повтор ничего не изменит и только усилит нагрузку.

Классифицируйте **на уровне клиента**: Feign/WebClient/RestTemplate — перехватывайте исключения и решайте, ретраить ли. В Resilience4j это `RetryConfig#retryExceptions/ignoreExceptions`.

Помните про «ложноположительные»: 404 иногда «временно» (кеш/индекс не догнался), но обычно это бизнес-ошибка. Не ретраим неочевидные кейсы без явного подтверждения.

И наконец, учитывайте **баланс**: даже допустимые ретраи без backoff/лимитов приведут к шторму.

```java
RetryConfig cfg = RetryConfig.custom()
  .maxAttempts(2)
  .waitDuration(Duration.ofMillis(200))
  .retryExceptions(TimeoutException.class, IOException.class)
  .ignoreExceptions(BadRequest.class, Unauthorized.class)
  .build();
```

## Backoff + jitter; лимит попыток и общий бюджет

Экспоненциальный backoff — увеличиваем паузу между попытками: 100 → 200 → 400 мс. **Джиттер** (рандом ±30–50%) рассинхронизирует клиентов, чтобы все не стучали одновременно. Это главный антидот от ретрай-штормов.

Лимит попыток обычно **1–2**, редко 3. Больше — почти всегда вред. И все попытки должны поместиться в **общий дедлайн**: если осталось 250 мс, нечестно ретраить ещё два раза по 400.

Смотрите на **тип операции**: GET может ретраиться агрессивнее, чем POST. Но POST можно ретраить **только** если у вас есть Idempotency-Key/дедуп.

Записывайте «попытку» в атрибуты спана (tracing): это сильно помогает в RCA.

```java
@EnableRetry
@Service
class Client {
  @Retryable(
    include = {SocketTimeoutException.class, ConnectException.class},
    maxAttempts = 2,
    backoff = @Backoff(delay = 200, multiplier = 2.0, random = true))
  String call(){ ... }

  @Recover String recover(Throwable ex){ return "fallback"; }
}
```

## Идемпотентность: Idempotency-Key, дедуп, защита от «двойных списаний»

Для безопасных повторов мутаторов передавайте `Idempotency-Key` и храните результат под этим ключом. Повторный запрос с тем же ключом возвращает тот же ответ, не создавая побочных эффектов.

На сервере — таблица `processed_ops` или «Outbox/Inbox». Ключ — детерминированный: сочетание бизнес-идентификаторов (например, `paymentId + stepVersion`). Не используйте «рандом в каждом ретрае».

В платежах удостоверьтесь, что **провайдер** тоже поддерживает идемпотентность (Stripe-подобные API). Иначе риск «двойных списаний» на стороне поставщика останется.

При получении дубля — **быстро** отвечайте «уже обработано», не повторяя все подсистемы (экономите ресурсы).

```kotlin
fun tryProcess(key: String, action: () -> Unit): Boolean =
  try { jdbc.update("insert into processed_ops(op_key) values (?)", key); action(); true }
  catch (e: DuplicateKeyException) { false }
```

## Hedged/parallel requests; уважение Retry-After

**Hedged** (или «дублирующий») запрос — отправка **двух** параллельных запросов в разные зоны/узлы, чтобы уменьшить хвосты. Вы берёте первый успешный ответ и отменяете второй. Полезно, когда хвосты обусловлены «редкими, но большими» задержками.

Это нужно применять аккуратно: хеджируетесь только на **чтениях**, не на мутациях, и с оглядкой на нагрузку (rate limit).

Уважайте `Retry-After`: если сервер ответил 429/503 с этим заголовком, **не игнорируйте**. Настройте Retry так, чтобы учитывать подсказку сервера (Resilience4j позволяет динамически рассчитать паузу).

Полезно комбинировать hedged с «быстрым путём» (например, локальный кэш или компактный эндпоинт «только заголовки») — иногда «меньше данных» = «лучше SLA».

```java
Mono<String> a = wc.get().uri("/dc-a/data").retrieve().bodyToMono(String.class);
Mono<String> b = wc.get().uri("/dc-b/data").retrieve().bodyToMono(String.class);
Mono<String> hedged = Mono.firstWithSignal(a, b).timeout(Duration.ofMillis(800));
```

отлично — продолжим «Устойчивость: таймауты, ретраи, circuit breaker» и подробно развернём **подтемы 4–7**. В каждом подпункте — минимум четыре абзаца и рабочие фрагменты кода (Java 17+, Spring Boot 3.x, Resilience4j).

---

# 4. Circuit Breaker: разрыв каскадов

## Модель состояний, окна и пороги

Circuit Breaker (CB) «размыкает цепь», когда доля ошибок или медленных вызовов превышает порог. **Closed** — всё нормально, считаем метрики; **Open** — «сразу ошибка» без попыток ходить в зависимость; **Half-Open** — пробное окно: пускаем ограниченное число запросов и смотрим, улучшилось ли. Решение об открытии/закрытии принимается по **скользящему окну** (по количеству вызовов или по времени) и порогам: `failureRateThreshold` (например, 50%) и/или `slowCallRateThreshold` (например, 70% медленных вызовов длиннее `slowCallDurationThreshold`). Также важны `minimumNumberOfCalls` (минимум наблюдений, чтобы судить) и `permittedNumberOfCallsInHalfOpenState` (сколько проб пускать).

Правильно настроенный CB защищает от «полумёртвых» зависимостей: когда среднее ещё ок, но хвосты огромные. Порог «медленных» вызовов позволяет открывать CB даже без ошибок, если SLA систематически не выдерживается. **Время в Open** (`waitDurationInOpenState`) должно быть достаточно длинным, чтобы зависимость успела восстановиться, но не чрезмерным (иначе долго живём без функциональности).

Важно понимать, что CB — диаграмма состояний **на конкретной зависимости/эндпоинте**. Слишком крупная «гранулярность» (один CB на все HTTP-клиенты) приведёт к «чёрно-белому» миру: деградация одной интеграции вырубит все. Слишком мелкая (каждый URL-параметр) — потеря статистики и лишний оверхед.

При проектировании учитывайте **долгие перехваты**: если таймаута нет или он слишком велик, CB не спасёт — вы просто не наберёте ошибок/медленных вызовов. Всегда ставьте **таймаут** (TimeLimiter) и уже вокруг — CB.

```yaml
resilience4j.circuitbreaker:
  instances:
    ext-read:
      slidingWindowType: COUNT_BASED
      slidingWindowSize: 50
      minimumNumberOfCalls: 20
      failureRateThreshold: 50
      slowCallRateThreshold: 70
      slowCallDurationThreshold: 500ms
      waitDurationInOpenState: 10s
      permittedNumberOfCallsInHalfOpenState: 5
      automaticTransitionFromOpenToHalfOpenEnabled: true
```

## Half-Open: политики проб и автоматический/ручной reset

Состояние **Half-Open** — момент истины. Консервативная политика — **single-probe**: пропускаем по одному запросу; более быстрая — **limited probes** (например, 5 параллельно), чтобы быстрее собрать статистику. В Resilience4j это `permittedNumberOfCallsInHalfOpenState`. Если все (или доля) успешны и быстры, возвращаемся в Closed; иначе снова Open.

Автоматический переход из Open в Half-Open включается флагом `automaticTransitionFromOpenToHalfOpenEnabled`: после `waitDurationInOpenState` брейкер сам начинает пускать пробы. Это снимает необходимость «тыкать» зависимость вручную. Но имейте в виду — всплески нагрузки в момент Half-Open нежелательны; ограничьте параллельность (bulkhead/rate limiter), чтобы не обрушить восстанавливающийся сервис.

Иногда нужен **ручной reset** (админ-кнопка) — например, вы поправили конфиг DNS и уверены, что зависимость ок. Через `CircuitBreakerRegistry` можно получить инстанс и вызвать `.reset()`. Это же место, где можно **подписаться на события**: CLOSED→OPEN, OPEN→HALF_OPEN и т.д., логировать и строить алёрты.

Удобно собирать события в метрики/логи: «сколько раз открылся CB за час», «среднее время в Open». Это помогает отличать случайные всплески от системной деградации и не реагировать «по ощущениям».

```java
@Bean
ApplicationRunner cbEvents(CircuitBreakerRegistry registry) {
  return args -> registry.circuitBreaker("ext-read").getEventPublisher()
      .onStateTransition(e -> log.warn("CB ext-read: {} -> {}", e.getStateTransition().getFromState(), e.getStateTransition().getToState()))
      .onError(e -> log.warn("CB ext-read error {}", e.getThrowable().toString()));
}
```

## Границы применения: per-endpoint / per-dependency

Гранулярность CB — архитектурное решение. Часто разумно иметь отдельные инстансы **по классу операций** и **по зависимостям**: `payments.read`, `payments.charge`, `catalog.search`, `catalog.details`. У разных операций разная «цена» ошибки и разные таймауты/пороговые значения. Например, `search` можно быстро деградировать (показать «популярное»), а `charge` — критичен и требует другой политики (меньше ретраев, длиннее таймаут, без hedged).

Для мульти-DC/мульти-региона полезно делить CB **по зоне** (A/B): если деградировала конкретная зона, не «портите репутацию» всего провайдера. В Feign/WebClient это достигается через разные бины клиентов и разные имена CB.

Не смешивайте в одном брейкере блокирующие и реактивные клиенты: у них разные профили латентности и разные failure modes. Разделение улучшит «сигнал-шум» и упростит тюнинг.

Если интеграция «токсичная» (под пиками часто деградирует), изолируйте её не только CB, но и **исполнителями** (bulkhead/отдельный пул потоков или семафор) и **rate limiter**. Так вы не дадите ей «утянуть» ресурсы у остального сервиса.

## Взаимодействие с ретраями/таймаутами

CB работает вместе с другими политиками. Мы придерживаемся порядка: **timeout → retry → circuit → bulkhead → rate limiter** (обоснование — в разделе 6). Тогда **таймаут** гарантирует разумную длительность попытки, **ретрай** даёт ограниченный шанс на временную ошибку, а **CB** принимает решение по агрегированной картине отказов/медленных вызовов, не «маскируемой» бесконечными ожиданиями.

Важно исключить «обратную связь»: если разместить ретрай **после** CB, то открытый брейкер будет мгновенно «скармливать» ретраю `CallNotPermittedException`, и вы получите локальный шторм пустых повторов. Размещая **ретрай до CB**, вы позволяете сделать 1–2 попытки и уже результат (успех/ошибка/slow) учитывает CB.

Таймаут должен **не превышать** `slowCallDurationThreshold`. Иначе вызов будет считаться «медленным» только после таймаута, и CB станет слепым к «почти медленным» кейсам. Обычно `slowCallDurationThreshold` = ~80% вашего TimeLimiter.

В тестах проверяйте: (1) CB открывается при нужной доле ошибок, (2) Half-Open допускает заданное число проб, (3) ретраи не запускаются при Open (fail-fast), (4) метрики slow-call корректно растут при искусственном замедлении.

---

# 5. Bulkhead/изоляция и ограничение нагрузки

## Пулы и семафоры per-класс операций

Bulkhead («шпангоут») — это **ограничение одновременности** для определённой операции. Для блокирующего кода используйте **ThreadPoolBulkhead**: отдельный пул потоков + очередь. Для реактивного/неблокирующего — **SemaphoreBulkhead**: фиксированное количество параллельных разрешений (permits).

Смысл прост: даже если зависимость зависла, у вас «зависнет» только N потоков/операций, а остальным хватит ресурсов. Важно задать **bounded очередь**. Без ограничений вы просто перенесёте проблему «подвисаний» из пула в бесконечную очередь (отложенные таймауты, рост памяти).

Под каждую «токсичную» интеграцию заведите **свой bulkhead**. Не делайте один общий «IO-пул»: он станет точкой конкуренции всех клиентов, и деградация одного будет «голодать» других.

Полезна политика **CallerRuns** на уровне бизнес-очередей: когда очередь на bulkhead переполнена, небольшой процент запросов выполняйте «здесь и сейчас» (если это микрозадача) или сразу возвращайте 429/503 (load shedding).

```yaml
resilience4j.bulkhead:
  instances:
    payments-bh:
      maxConcurrentCalls: 32
      maxWaitDuration: 50ms  # сколько ждём permit
  thread-pool:
    instances:
      http-blocking-bh:
        coreThreadPoolSize: 32
        maxThreadPoolSize: 64
        queueCapacity: 1000
```

## Rate limiter и load shedding

Rate limiter ограничивает **скорость** вызовов к зависимости: «не больше N за окно T». Это повышает «вежливость» клиента и спасает при резкой вспышке трафика. В Resilience4j — `limitRefreshPeriod`, `limitForPeriod`, `timeoutDuration` (сколько ждать токен). Если токенов нет — быстро отказать (429), не подвешивая поток.

**Load shedding** — сознательный сброс части нагрузки, когда вы близки к исчерпанию ресурсов. Реализуется как простой фильтр: если глубина очереди/доля CPU/GC-паузы превышают порог, немедленно отвечаем 429/503 для «не-критичных» запросов. Это хуже, чем полноценный ответ, но лучше тотального коллапса.

Важная деталь — **приоритеты**: лимитируйте фоновые и «тяжёлые» операции раньше пользовательских критичных путей. Держите отдельные лимитеры и bulkhead’ы для фона (репорты, экспорты, ресинк).

В метриках и логах фиксируйте **причину** отклонения (no-permits, rate-exceeded, queue-full) — это ускоряет RCA и настройку порогов.

```yaml
resilience4j.ratelimiter:
  instances:
    ext-rl:
      limitRefreshPeriod: 1s
      limitForPeriod: 100
      timeoutDuration: 0 # не ждём токен
```

## Адаптивная конкуренция (AIMD/gradient)

Статические лимиты — это начало. В реальности «оптимальная» параллельность меняется со временем: утренние пики, GC, соседняя нагрузка. Адаптивные алгоритмы (AIMD — «аддитивно увеличиваем, мультипликативно уменьшаем», или градиентные как Netflix Concurrency Limits) динамически подстраивают допустимую конкуренцию по наблюдаемой задержке.

Идея: пока p50/p95 низкие — **потихоньку** увеличивать permits; как только замечаем рост латентности/ошибок — **резко** уменьшать. Это позволяет «ползти» к границе пропускной способности и отскакивать при первых признаках перегруза.

В проде можно начать с **простого AIMD** поверх `Semaphore`: раз в секунду, если p95 < целевого — `permits++`; если p95 > порога — `permits = max(1, permits/2)`. Обязательно добавьте «порог минимальных измерений» и «тормоз» на изменение, чтобы не дрожать.

Такой подход особенно полезен для «дорогих» внешних API: вы не «выжимаете газ в пол», а едете по «круиз-контролю» под наблюдением метрик.

```java
class AdaptivePermits {
  private final AtomicInteger permits = new AtomicInteger(16);
  void onTick(Duration p95) {
    if (p95.compareTo(Duration.ofMillis(200)) < 0) permits.incrementAndGet();
    else permits.updateAndGet(p -> Math.max(1, p / 2));
  }
  int current() { return permits.get(); }
}
```

## Приоритеты: критичные пути vs фон

Установите **политику приоритетов**. Критичные бизнес-пути (создание заказа, авторизация) должны иметь «запас» ресурсов: выделенные bulkhead’ы, меньшие timeouts (fail-fast), отдельные лимитеры. Фоновые задачи (индексация, экспорты) — наоборот: «бюджет остатка», паузы и рестарт позже.

В Spring используйте **разные executors**: `@Async("criticalExecutor")` и `@Async("backgroundExecutor")`, где у первого — небольшой пул, короткая очередь и строгий таймаут, у второго — большой пул и политика CallerRuns/отложенные ретраи.

В HTTP-слое можно маркировать запросы тегами/ролью и применять разные цепочки политик. В WebFlux — через разные маршруты/фильтры, в MVC — через HandlerInterceptor.

Такая стратификация не даёт «второстепенным» задачам вытеснять пользовательский трафик — классическая причина «дневных» инцидентов.

---

# 6. Композиция политик и fallback-стратегии

## Порядок декораторов (почему именно так)

Рекомендуемый порядок: **TimeLimiter (timeout) → Retry → CircuitBreaker → Bulkhead → RateLimiter**. Сначала **ограничиваем время** одной попытки, чтобы не плодить «долгие хвосты». Затем даём шанс на **один короткий повтор** (только для транзиентов) внутри **общего дедлайна**. Дальше **CB** принимает решение по картине отказов/slow-calls, а **Bulkhead** и **RateLimiter** — это «физические» ограничители, которые применяются на самом внешнем уровне и защищают процесс от перегруза.

Если поменять порядок, возникают побочные эффекты. Например, если Retry «снаружи» CB — несколько попыток одной пользовательской операции будут «молотить» CB и искажать статистику. Если CB «снаружи» TimeLimiter — он не увидит slow-calls (таймаут не срабатывает внутри). Поэтому держимся предложенной последовательности.

В реактивном стеке порядок задаётся композицией операторов/декораторов над `Publisher`. В аннотационном стиле (Spring AOP) порядок аспектов контролируется `@Order`/`Ordered` — при сложных кейсах лучше использовать **programmatic** `Decorators`.

Тестируйте композицию отдельно: симулируйте медленные ответы и 5xx, проверяйте, что именно TimeLimiter «вскрывает» задержки, Retry делает ровно N попыток, CB переходит в Open при заданных порогах, а Bulkhead/Ratelimiter отсекают излишки.

```java
Supplier<String> call = () -> client.get();
var decorated = Decorators.ofSupplier(call)
  .withTimeLimiter(TimeLimiter.of(Duration.ofMillis(800)), ScheduledExecutorServiceHolder.ses())
  .withRetry(retry)
  .withCircuitBreaker(cb)
  .withBulkhead(SemaphoreBulkhead.ofDefaults("bh"))
  .withRateLimiter(RateLimiter.ofDefaults("rl"))
  .decorate();
```

## Fallback: кэш/статический ответ/частичная функциональность

Fallback не должен быть «случайным». Это заранее продуманный **альтернативный путь**: отдать кеш/снапшот (даже устаревший в пределах бюджета), вернуть «урезанную» модель (без дорогостоящих полей), показать stub («не удалось загрузить рекомендации»).

В Spring + Resilience4j можно задать `fallbackMethod` у аннотаций или обернуть вызов в `Try.recover`. Функция fallback должна быть **быстрой** и **надёжной** (никаких внешних вызовов с непредсказуемыми таймаутами). Источники: локальный кэш (Caffeine), L2 (Redis) со «stale if error», файл конфигурации, заранее подготовленный список.

Следите за **семантикой ответов**: если вернули частичный ответ, пометьте это флагом/заголовком. На клиенте/фронте можно отрисовать «упрощённый» вид. Документируйте такие режимы — это часть UX.

И наконец: **логируйте** факт fallback’а с причиной (`TimeoutException`, `CallNotPermittedException`, `RateLimitExceeded`). Это важно для пост-анализов и для алёртов (внезапный рост доли fallback — симптом деградации).

```java
@CircuitBreaker(name = "catalog", fallbackMethod = "fallbackCatalog")
@TimeLimiter(name = "catalog-tl")
public CompletableFuture<Catalog> catalog() {
  return CompletableFuture.supplyAsync(() -> client.load());
}
private CompletableFuture<Catalog> fallbackCatalog(Throwable ex) {
  return CompletableFuture.completedFuture(cache.getOrDefault("catalog", Catalog.partial()));
}
```

## Anti-patterns (чего не делать)

**Бесконечные ретраи** и ретраи без backoff/jitter — путь к «шторму» и DDoS чужого API. Ограничивайте попытки и соблюдайте общий дедлайн.

**Ретраить после Open CB** — бессмысленно: `CallNotPermittedException` надо быстро превращать в fallback/ошибку, а не «гонять» повтор.

**Одинаковые таймауты на всех уровнях**: если connect=read=write=5s, вы получите 15-секундное «молчание». Подбирайте таймауты с учётом профиля операции.

**Bulkhead без очереди/таймаута** или бесконечная очередь — замаскированный «крах медленно». Ограничивайте очередь и ставьте маленький `maxWaitDuration`.

## Стандартизация политик по классам операций

Лучше один раз договориться и закрепить в коде/конфигурации «**политики по классам операций**»:

* `read-fast`: TL=300–800ms, Retry=1, CB пороги выше к slow-calls, BH small.
* `read-heavy`: TL=1–2s, Retry=1, CB strict к slow-calls, BH medium.
* `write-critical`: TL=2–3s, Retry=0–1 (только транзиенты), без hedged, CB мягче к ошибкам, но ниже к slow-calls.
* `batch`: TL длиннее, Retry возможно, но жёсткие лимитеры и отдельные executors.

Зашейте это в `application.yml` и в небольшой «фабрике» декораторов/клиентов. Тогда добавление новой интеграции — выбор профиля и минимум кастомщины.

```yaml
resilience4j:
  timelimiter.instances:
    read-fast: timeout-duration: 800ms
    write-critical: timeout-duration: 2s
  retry.instances:
    read-fast:  max-attempts: 2; wait-duration: 150ms; enable-exponential-backoff: true
    write-critical: max-attempts: 1
  circuitbreaker.instances:
    read-fast:  failureRateThreshold: 50; slowCallRateThreshold: 70; slowCallDurationThreshold: 500ms
    write-critical: failureRateThreshold: 30; slowCallRateThreshold: 50; slowCallDurationThreshold: 1200ms
```

---

# 7. Spring Boot / Resilience4j: практическая интеграция

## Аннотации и конфиг через `application.yml`

Spring Boot 3.x + Resilience4j поддерживает **аннотационный** стиль: `@TimeLimiter`, `@Retry`, `@CircuitBreaker`, `@Bulkhead`, `@RateLimiter`. Каждая аннотация ссылается на **имя инстанса** из `application.yml`. Это самый быстрый путь начать и держать конфиг централизованно.

Плюс аннотаций — читаемость и простая маршрутизация политик «по методу». Минус — сложнее контролировать **порядок** при сложных комбинациях; если он критичен, переключайтесь на **programmatic** Decorators (см. выше).

Для метрик Resilience4j автоконфиг «подхватывает» Micrometer: вы получите таймеры, счётчики ошибок/slow-calls, gauge по состоянию CB, rejected bulkhead/ratelimiter. Дальше остаётся настроить алёрты (например, рост `slow_call_rate` > 30% или `state=open`).

Ниже — базовый `application.yml` с «чтениями» и «записями»:

```yaml
resilience4j:
  timelimiter.instances:
    ext-read:  timeout-duration: 800ms
    ext-write: timeout-duration: 2s
  retry.instances:
    ext-read:
      max-attempts: 2
      wait-duration: 150ms
      enable-exponential-backoff: true
      exponential-backoff-multiplier: 2.0
      retry-exceptions: [java.io.IOException, java.net.SocketTimeoutException]
      ignore-exceptions: [com.example.api.BadRequestException]
    ext-write:
      max-attempts: 1
  circuitbreaker.instances:
    ext-read:
      slidingWindowType: COUNT_BASED
      slidingWindowSize: 50
      minimumNumberOfCalls: 20
      failureRateThreshold: 50
      slowCallRateThreshold: 70
      slowCallDurationThreshold: 500ms
      waitDurationInOpenState: 10s
      permittedNumberOfCallsInHalfOpenState: 5
      automaticTransitionFromOpenToHalfOpenEnabled: true
  bulkhead.instances:
    ext-read-bh:  maxConcurrentCalls: 32; maxWaitDuration: 50ms
  ratelimiter.instances:
    ext-rl: limitRefreshPeriod: 1s; limitForPeriod: 100; timeoutDuration: 0
```

Использование на сервисе:

```java
@Service
class CatalogClient {
  @TimeLimiter(name="ext-read")
  @Retry(name="ext-read")
  @CircuitBreaker(name="ext-read")
  @Bulkhead(name="ext-read-bh", type=Bulkhead.Type.SEMAPHORE)
  @RateLimiter(name="ext-rl")
  public CompletableFuture<Catalog> getCatalog() {
    return CompletableFuture.supplyAsync(() -> http.getCatalog()); // блокирующий вызов — перенесите в отдельный executor
  }
}
```

## WebClient/Feign/RestTemplate: пер-клиентные политики и таймауты сокетов/пулов

Для **WebClient** в реактивном стеке настраивайте сетевые таймауты на уровне Reactor Netty (connect/response) и используйте аннотации/декораторы Resilience4j для TL/Retry/CB. Не блокируйте event-loop: если внутри неизбежно блокирующая операция, переносите её на `boundedElastic`.

```java
@Bean
WebClient webClient() {
  var http = HttpClient.create()
      .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 300)
      .responseTimeout(Duration.ofMillis(700));
  return WebClient.builder().clientConnector(new ReactorClientHttpConnector(http)).build();
}
```

С **Feign** задайте сокетные таймауты клиента (OkHttp/HC5) и оборачивайте **методы сервиса**, вызывающего Feign-клиент, аннотациями Resilience4j (или Spring Cloud CircuitBreaker, если он у вас принят). Так проще контролировать композицию политик.

С **RestTemplate** используйте `HttpComponentsClientHttpRequestFactory`/OkHttp3 и задавайте `connectTimeout`, `readTimeout`, `connectionRequestTimeout` (для пула). Важно: **пер-клиентные** factory/bean’ы — разные зависимости → разные таймауты/пулы.

```java
@Bean
RestTemplate restTemplate() {
  var reqFactory = new HttpComponentsClientHttpRequestFactory();
  reqFactory.setConnectTimeout(300);
  reqFactory.setReadTimeout(700);
  return new RestTemplate(reqFactory);
}
```

## Data-клиенты: JDBC/R2DBC/Redis

Для **JDBC**: `HikariCP` с разумными лимитами пула (`maximumPoolSize`, `connectionTimeout`, `maxLifetime`), и обязательно `Statement.setQueryTimeout` (секунды). На Postgres можно выставить `statement_timeout` на уровне **соединения** (session level), чтобы БД сама прерывала долгие запросы.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      connection-timeout: 500
      max-lifetime: 30s
```

В коде:

```java
jdbcTemplate.query(con -> {
  var ps = con.prepareStatement(SQL);
  ps.setQueryTimeout(1); // сек
  return ps;
}, rs -> map(rs));
```

Для **R2DBC** используйте параметры драйвера/пула (`maxSize`, `maxAcquireTime`, `maxIdleTime`) и таймаут на сторону клиента (`.timeout(Duration)`) при подписке. Помните: R2DBC — реактивный, но **сервер** может быть медленным; TimeLimiter обязателен.

Для **Redis (Lettuce)** задайте `commandTimeout`, `connectTimeout`, настройки пула, и оберните кэш-вызовы **CB/Retry** как и любую зависимость. Падение Redis не должно валить бизнес — используйте `CacheErrorHandler` и fallback «на origin».

## Security/MDC/trace контекст; маппинг в ProblemDetail

Асинхронность и реактивность ломают «потоковый» контекст. Для блокирующего `@Async` используйте **TaskDecorator** (копирование MDC/Locale/SecurityContext). Для Reactor — `Context` и **Micrometer Context Propagation**/Spring Sleuth/Otel-bridge: это сохранит `traceId/spanId` через операторные границы, а логи станут читабельными.

Ошибки устойчивости нужно **единообразно** маппить в HTTP-ответы: `TimeoutException` → 504, `CallNotPermittedException` (CB Open) → 503, `BulkheadFullException` → 429/503, `RequestNotPermitted` (rate) → 429. Делайте это через `@ControllerAdvice` + `ProblemDetail` (RFC-7807), добавляя детали: `policy`, `attempt`, `retryAfter`.

```java
@RestControllerAdvice
class ResilienceErrors {
  @ExceptionHandler(TimeoutException.class)
  public ResponseEntity<ProblemDetail> timeout(TimeoutException ex){
    var pd = ProblemDetail.forStatus(HttpStatus.GATEWAY_TIMEOUT);
    pd.setTitle("Upstream timed out");
    return ResponseEntity.status(504).body(pd);
  }
  @ExceptionHandler(CallNotPermittedException.class)
  public ResponseEntity<ProblemDetail> cb(CallNotPermittedException ex){
    var pd = ProblemDetail.forStatus(HttpStatus.SERVICE_UNAVAILABLE);
    pd.setTitle("Dependency unavailable (circuit open)");
    return ResponseEntity.status(503).body(pd);
  }
}
```

В трейсинг добавляйте **атрибуты политики**: имя инстанса, `attempt`, `backoffMs`, `cb_state`, `rl_permit`. Это затем связывается с бизнес-метриками (конверсия, ошибки) и позволяет быстро увидеть, **какая** политика «стреляет» и **почему**.

отлично — завершаю тему «Устойчивость: таймауты, ретраи, circuit breaker» подробными **подтемами 8–10**. Как и раньше, каждый подпункт раскрыт минимум в 4 абзацах и снабжён практическими фрагментами кода (Java/Kotlin, Spring Boot 3.x, Resilience4j, SQL).

---

# 8. Данные и согласованность под ретраями

## Outbox/Inbox + транзакции: атомарность «write + publish», обработка дублей

Паттерн **Outbox** решает базовую проблему: как сделать так, чтобы изменение в вашей БД и публикация события (например, в Kafka) были **атомарны**. Мы записываем доменное изменение и **событие** в одну и ту же транзакцию той же БД. Публикация наружу переносится во второй этап — отдельный «ретранслятор» читает таблицу outbox и надёжно шлёт наружу. Если процесс упал между коммитом и отправкой, событие не теряется — оно лежит в таблице и будет отправлено при следующем проходе.

Outbox таблица обычно имеет `id`, `aggregate_type/id`, `event_type`, `payload` (JSON), `created_at`, `status`/`sent_at`. Важны индексы по `status` и `created_at`, чтобы быстро выбирать «неотправленные». Сам ретранслятор помечает запись как `SENT` **после** успешной отправки; если отправка упала, он повторит позже — это и есть «at-least-once».

Комплементарный паттерн **Inbox** живёт у потребителя: он хранит `message_id`/`event_id` в своей БД и отклоняет **повторную обработку** события с тем же ключом (дедупликация). Это закрывает вопрос дублей на приёме и делает идемпотентным сам потребитель.

DDL для Outbox и простой ретранслятор:

```sql
create table outbox (
  id uuid primary key,
  aggregate_type text not null,
  aggregate_id text not null,
  event_type text not null,
  payload jsonb not null,
  created_at timestamptz not null default now(),
  sent_at timestamptz,
  status text not null default 'NEW'
);
create index on outbox(status, created_at);
```

```java
@Service
class OutboxRelay(
    JdbcTemplate jdbc, KafkaTemplate<String,String> kafka
) {
  @Transactional
  public int relayBatch(int limit) {
    var list = jdbc.query("""
      select id, event_type, payload::text
      from outbox where status='NEW'
      order by created_at asc
      limit ?
    """, (rs, i) -> new Out(rs.getString(1), rs.getString(2), rs.getString(3)), limit);

    for (var e : list) {
      kafka.send("domain-events", e.id(), e.payload()).get(); // дождаться ack
      jdbc.update("update outbox set status='SENT', sent_at=now() where id=? and status='NEW'", e.id());
    }
    return list.size();
  }
  record Out(String id, String type, String payload) {}
}
```

## Сага/компенсации для цепочек действий; idempotent consumers

Когда бизнес-транзакция растянута на несколько сервисов (резервирование денег, бронирование товара, выписка накладной), **саги** дают модель согласованности: каждый шаг имеет **компенсацию** (отмена резерва, возврат товара к доступным). Это не «два шага в одной транзакции», а **оркестрация** «сделай/отмени» с гарантией завершения.

Оркестратор может быть явным (сервис действий) или неявным (хореография событий). Для устойчивости важно: (1) шаги должны быть **идемпотентны**, (2) компенсации — тоже, (3) сообщения — «at-least-once», а потребители — с Inbox-дедупом. Тогда ретраи при сбоях не приводят к «дважды списали»/«дважды вернули».

Идемпотентный потребитель хранит «последнюю обработанную версию» агрегата или «последний event_id». Если приходит дубликат — просто подтверждает и делает `ACK`, не инициируя побочные эффекты. Это особенно критично для шагов, имеющих внешние действия (платёжные шлюзы, SMS).

Пример Inbox-дедупа у потребителя:

```sql
create table inbox (
  message_id text primary key,
  received_at timestamptz default now()
);
```

```java
@Transactional
public void onMessage(Event e) {
  try {
    jdbc.update("insert into inbox(message_id) values (?)", e.id());
  } catch (DuplicateKeyException dup) {
    return; // уже обработано
  }
  // безопасная идемпотентная логика шага
  process(e);
}
```

## «Exactly-once» как маркетинг: проектирование под at-least-once + дедуп/версии

Термин **exactly-once** часто звучит в маркетинге брокеров, но в общем случае сквозного потока (HTTP → Kafka → БД → внешний API) «ровно один раз» **не достигается** без жёстких предпосылок. Реалистичная цель — **at-least-once** с дедупликацией и идемпотентностью на каждом узле. Тогда даже при ретраях/дубликатах итоговое состояние будет корректным.

Один из простых механизмов — **версионирование**: каждое действие несёт `version` / `sequence`. При применении состояния вы проверяете «текущая версия + 1?» — если нет, значит дубликат/несинхронность, и действие отклоняется. Это решается как на уровне SQL (`where version = ?`) через optimistic locking, так и на уровне «таблицы обработанных операций».

Для API — **Idempotency-Key** и сохранение результата по ключу. Для брокера — Inbox/Outbox. Для БД — **upsert** («вставь или обнови по ключу»), чтобы повтор не выбрасывал исключения, а консистентно завершался.

Пример upsert с версией:

```sql
-- обновить цену, только если пришла следующая версия
update product_price set amount=:amount, version=:newVer
where product_id=:id and version=:prevVer;
-- если 0 строк обновлено — дубликат или соревновательная запись
```

## Защита от «двойных списаний» и повторной отправки платёжных операций

В платежах идемпотентность — критична: сети могут терять ответы, пользователи жмут «повторить», интеграции флапают. Клиент **должен присылать Idempotency-Key**, а сервер — сохранять его вместе с результатом «действия» (chargeId, статус). При повторе с тем же ключом нужно вернуть **тот же** результат, не инициируя новое списание.

На уровне интеграции с провайдерами (Stripe, YooKassa, Adyen) используйте их **корреляционные ключи** и проверяйте «уже был чардж?» перед новым запросом. Если у провайдера нет нативной идемпотентности — добавляйте свою: храните внешний `provider_operation_id` и маппинг на ваши операции, чтобы повтор не порождал новый вызов.

Гарантию «не больше одного списания» дополняйте **финальной сверкой** (reconciliation) — фоновый процесс проверяет ваши записи против отчётов провайдера и в случае рассинхронизации запускает компенсации. Это закрывает крайние случаи сетевых сбоев «после» 200 OK.

Скелет сервиса списания с ключом идемпотентности:

```java
@Transactional
public ChargeResult charge(String idemKey, ChargeCmd cmd) {
  try {
    jdbc.update("insert into processed_ops(op_key) values (?)", idemKey);
  } catch (DuplicateKeyException e) {
    return loadResultByKey(idemKey); // вернуть тот же ответ
  }
  var external = paymentsGateway.charge(cmd); // внешнее списание
  saveResult(idemKey, external);
  return external;
}
```

---

# 9. Наблюдаемость, алёрты и эксплуатация

## Метрики: latency p50/p95/p99, error/slow-call rate, состояние breaker’ов, глубина очередей, rejected

Без метрик устойчивость превращается в «чёрный ящик». Нужны **распределения задержек** (p50/p95/p99), доля ошибок и **slow-call rate** (сколько вызовов превысили порог), состояние брейкеров (open/half-open), глубина очередей bulkhead и число **rejected** (нет permit/места в очереди). Всё это даёт Resilience4j + Micrometer «из коробки».

Снимайте метрики **помесячно и понедельно**, чтобы видеть тренды, и **по инцидентам** — чтобы анализировать отклонения. Отдельные time series по «классам операций» (`read-fast`, `write-critical`) и по «зависимостям» (`payments`, `catalog`) помогут увидеть, где «печёт».

Сигналы для алёртов: рост `slow_call_rate` (например, >30% 5 минут), `state=open` у брейкера, всплеск `rejected.calls` у bulkhead/ratelimiter, рост p99 сверх SLO, «зубцы» глубины очередей. Эти алёрты должны вести не к «перепиливанию» политики, а к **RCA** зависимостей (DB/сеть/внешний API).

Выведите в дашбордах «комбинированный вид»: сверху — бизнес-метрики (ошибки, конверсия/доход), ниже — технические (таймауты, CB, retry), и ниже — инфраструктурные (CPU/GC/сеть, пул соединений). Связь «бизнес → тех → инфра» позволяет быстро локализовать корень.

Код: дополнительный таймер вокруг «клиента» + gauge очереди:

```java
@Bean MeterBinder customMetrics(ThreadPoolTaskExecutor httpExecutor) {
  return registry -> {
    Gauge.builder("bulkhead.queue.depth", httpExecutor, e -> e.getThreadPoolExecutor().getQueue().size())
        .description("HTTP bulkhead queue depth").register(registry);
  };
}

@Observed(name="ext.call", contextualName="ext:getCatalog")
public Catalog call() { return client.getCatalog(); }
```

## Трейсинг: спаны с атрибутами политики, корреляция с бизнес-метриками

**Трейсинг** (OpenTelemetry) — ваш микроскоп. Каждый внешний вызов — отдельный спан с атрибутами **политики**: `retry.attempt`, `retry.backoff_ms`, `cb.state`, `timelimiter.timeout_ms`, `bulkhead.permit`. Это позволяет на одной диаграмме понимать, сколько попыток было, какой был backoff, в каком состоянии был брейкер.

Коррелируйте **traces** с бизнес-метриками. Например, спаны «оплата/charge» и показатель «успешные оплаты» на одном графике быстро покажут: «ретраи растут — успешность падает — CB открыт». Это снижает MTTR и делает отчёты по инцидентам предметными.

В Spring Boot используйте `@Observed`/`Observation` и `ObservationConvention` для добавления низкокардинальных (name, dependency) и высококардинальных (endpoint/attempt) тегов. В реактивном стеке убедитесь, что trace-контекст **пропагируется** через Reactor (Micrometer Context Propagation / OTel bridge).

Пример добивки атрибутов в спан:

```java
@Component
class RetryObservationConvention implements ObservationConvention<RetryObservationContext> {
  public boolean supportsContext(Observation.Context ctx){ return ctx instanceof RetryObservationContext; }
  public KeyValues getLowCardinalityKeyValues(RetryObservationContext ctx) {
    return KeyValues.of("dep", ctx.getName(), "attempt", String.valueOf(ctx.getRetryAttempt()));
  }
}
```

## Логирование причин отказов/таймаутов; сэмплирование «шторма»

Логи должны быть **структурированными** и **редуцированными**. В обычном режиме — выборочное логирование ошибок (sample 1–5%), в «шторм» — повышенное, но с защитой от «лог-штормов» (rate limit). Обязательно логируйте **причину** (`Timeout`, `ConnectTimeout`, `CB_OPEN`, `RATE_LIMIT_EXCEEDED`) и **контекст** (dependency, endpoint, attempt, backoff).

Используйте **MDC** для корреляции: `traceId`, `spanId`, `userId`/`tenant`, `requestId`. Это позволяет склеивать логи приложения и внешних компонентов (ingress, WAF, прокси) в одном Kibana/Tempo/Jaeger.

Полезно ввести поле «policy_outcome»: `OK`, `RETRY`, `TIMEOUT`, `CB_OPEN`, `FALLBACK`. Тогда по простому фильтру видно «долю» каждого исхода и можно строить дашборды «с чего начинать копать».

Часть логов должна идти в **аудит** (особенно платежи): кто-то инициировал компенсацию, сработал ли fallback и какой. Это помогает и в «разборе полётов», и в соответствиях требованиям (PCI DSS и т.п.).

```java
log.warn("ext call failed: dep=catalog endpoint={} cause={} attempt={} backoffMs={}",
  path, cause, attempt, backoff);
```

## Chaos-практики: Toxiproxy/latency/packet-loss, fault-injection; runbooks и «kill-switch»

**Chaos engineering** — управляемые эксперименты: «а что будет, если медленная сеть/потеря пакетов/пиковые таймауты?». В dev/stage можно использовать **Toxiproxy** (TC) для инъекции задержек/потерь/обрывов между приложением и зависимостью. Это помогает проверить, что политики действительно работают, а не просто «настроены».

**Fault injection** — включение «искусственных» ошибок в тестовых средах (например, через WireMock/Toxiproxy/HAProxy). Сценарии: 20% 5xx, перцентильная задержка >1s, полный обрыв на 60 секунд. Тогда вы видите, как ведут себя ретраи/брейкеры/лимитеры и что происходит с SLA.

Нужны **runbooks**: пошаговые инструкции для on-call — «если CB открыт >5 минут, включить kill-switch у интеграции X (feature-flag), перевести UI на fallback». Kill-switch — это флаг конфигурации, который временно выключает функциональность/интеграцию, чтобы «спасти ядро».

Пример Testcontainers + Toxiproxy:

```java
@Container static ToxiproxyContainer toxiproxy = new ToxiproxyContainer("shopify/toxiproxy:2.7.0");
@Container static GenericContainer<?> ext = new GenericContainer<>("kennethreitz/httpbin").withExposedPorts(80);

@Test
void latency_injection() throws IOException {
  Proxy proxy = toxiproxy.createProxy(ext, 80);
  proxy.toxics().latency("slow", ToxicDirection.DOWNSTREAM, 600); // 600ms
  // WebClient goes via proxy.getContainerIpAddress():proxy.getProxyPort()
  // проверяем, что TL/CB сработали
}
```

---

# 10. Тестирование и CI/CD-гейты

## Интеграционные тесты: WireMock/MockWebServer, Retry-After; Toxiproxy для сети

Для HTTP хорошо подходят **WireMock**/`MockWebServer` (OkHttp). Вы моделируете 5xx/429/timeout и проверяете, что **ровно N** ретраев произошло, что `Retry-After` уважён, что **общая длительность** укладывается в дедлайн. Для сетевых «флапов» используйте **Toxiproxy**: инжектируйте задержки, потери, обрывы.

Тест должен проверять не только «получили ответ/ошибку», но и **события** Resilience4j (например, что CB открылся), а также метрики (рост `failed.calls`, `slow.calls`). Это превращает устойчивость из «элегантной конфигурации» в **проверяемое свойство** системы.

Стаб 429 c `Retry-After`:

```java
WireMock.stubFor(get(urlEqualTo("/ext"))
  .inScenario("rate")
  .whenScenarioStateIs(STARTED)
  .willReturn(aResponse().withStatus(429).withHeader("Retry-After","1"))
  .willSetStateTo("ok"));

WireMock.stubFor(get(urlEqualTo("/ext"))
  .inScenario("rate").whenScenarioStateIs("ok")
  .willReturn(aResponse().withStatus(200).withBody("OK")));
```

Проверка `WebClient` + Retry уважает подсказку и укладывается в бюджет:

```java
StepVerifier.withVirtualTime(() -> service.callExt())
  .thenAwait(Duration.ofSeconds(2))
  .expectNext("OK")
  .verifyComplete();
```

## Нагрузочные прогоны: шторм ретраев, деградация зависимостей, лимиты пулов

Нагрузочные тесты должны моделировать **шторм ретраев**: возьмите профиль с малым лимитом внешнего API (например, 100 RPS), выставьте клиенту 2 попытки и прогоните 1000 RPS: увидите, как быстро вы упираетесь в лимитер/CB и что делает ваш сервис (держится или коллапсирует). Это лучше делать на **stage** с Toxiproxy/HAProxy, имитирующим лимиты и задержки.

Проверьте **истощение пулов**: слишком длинные таймауты/отсутствие TL приводят к росту очередей в Hikari/HTTP-пуле. Меряйте `active/idle`, `wait`, `borrow` тайминги. В отчётах нагрузочного прогона ищите «лесенку» p99 и «пилу» throughput — это симптомы неправильной композиции политик.

Добавьте сценарии «медленная деградация» (latency grows до 2–3 сек) — это проверит `slow-call rate` и реакцию CB на «не ошибки, но уже плохо». И отдельный сценарий «полный обрыв 60 сек» — должен включиться CB и система должна **быстро** возвращать 503/фоллбек, не подвисая.

Если нет Gatling/k6, можно собрать лёгкий harness на JMH/Java Executors: 100–1000 параллельных задач, замер распределений и экспорт в Prometheus для графиков.

## Контрактные проверки политик: снапшоты конфигов

Политики устойчивости — такой же контракт, как и OpenAPI. Держите **снапшоты** `application.yml` (секцию Resilience4j) в тестах и проверяйте, что «по умолчанию» у чтений и мутаций **разные** таймауты/ретраи/пороги. Это предотвращает случайный регресс (кто-то «подровнял» все таймауты под одно).

Маппинг в конфигурационный бин и проверка AssertJ:

```java
@ConfigurationProperties(prefix="resilience4j.timelimiter.instances")
record TLProps(Map<String, Instance> instances) {
  record Instance(Duration timeoutDuration) {}
}

@SpringBootTest
class ResilienceConfigTest {
  @Autowired TLProps tl;
  @Test void reads_vs_writes() {
    assertThat(tl.instances().get("ext-read").timeoutDuration()).isLessThan(Duration.ofSeconds(1));
    assertThat(tl.instances().get("ext-write").timeoutDuration()).isGreaterThan(Duration.ofSeconds(1));
  }
}
```

Подобным образом проверьте `retry.maxAttempts`, `cb.slowCallDurationThreshold` и `bulkhead.maxConcurrentCalls` для разных профилей. Это дешёвая страховка, чтобы не «снести» плавность системы.

## Progressive delivery: rollout, динамическая перенастройка, алёрты на p95/p99

Для безопасного внедрения политик используйте **progressive delivery**: включайте новые значения только для части трафика (по tenant/фиче/проценту). Для Resilience4j возможно **динамическое** обновление конфигов через Spring Cloud Config / Consul / собственный админ-API, а сам `CircuitBreakerRegistry`/`RetryRegistry` позволяет программно пересоздавать/перенастраивать инстансы.

Держите **алёрты на регресс p95/p99** и долю fallback: при rollout сначала на 5% → наблюдение → 25% → 100%. Если пошёл регресс — автоматический откат (feature-flag). Важно, чтобы «крутилки» политик были **в коде/конфиге**, а не в «скрытой» админке без версионирования.

Мини-пример динамического переключения порогов CB:

```java
@RestController
class ResilienceAdmin(
    CircuitBreakerRegistry registry
) {
  @PostMapping("/admin/cb/{name}/slow-threshold")
  void setSlow(@PathVariable String name, @RequestParam Duration d) {
    var cb = registry.circuitBreaker(name);
    var cfg = cb.getCircuitBreakerConfig().toBuilder()
        .slowCallDurationThreshold(d)
        .build();
    registry.replace(name, cfg); // пересоздаёт инстанс с новым конфигом
  }
}
```

Добавьте «kill-switch» фичу: пер-клиентный **отключатель** интеграции — если зависимость падает, отключаем обращения (сразу 503/фоллбек), а не «ждём милости погоды». Это лучший друг on-call ночью.


# Вопросы

Ниже — 30 собес-вопросов по теме «Устойчивость: таймауты, ретраи, circuit breaker» с чёткими ответами и короткими примерами (Java 17+, Spring Boot 3.x, Resilience4j).

---

1. **Что такое «устойчивость» (resilience) в микросервисах?**
   Это набор техник, которые делают систему предсказуемой под частичными отказами: таймауты, ретраи, circuit breaker, bulkhead, rate limiter, fallback. Цель — сдержать каскадные отказы, стабилизировать задержки (p95/p99) и сохранить SLA.

2. **Какие типовые failure modes надо учитывать?**
   Сетевые задержки/потери, «полумёртвые» зависимости (долго отвечают), истощение пулов (HTTP/JDBC), GC-паузы/stop-the-world, DNS-флапы. Каждый тип требует своей защиты: таймауты, ограничение параллелизма, быстрый отказ.

3. **Почему fail-fast лучше «подвисаний»?**
   Быстрый предсказуемый отказ освобождает ресурсы (потоки/коннекты), защищает пулы и даёт клиенту шанс сделать повтор позже/по другой стратегии. «Долгие» хвосты размазывают проблему и взрывают p99.

4. **Что такое deadline propagation и зачем он нужен?**
   Единый дедлайн на запрос (например, 800 мс) протаскивается вниз по стеку. Каждый внутренний вызов получает «остаток» времени, а не фиксированный таймаут. Это предотвращает сумму перехлёстывающих таймаутов > внешнего SLO.

5. **Где задавать HTTP-таймауты и какие именно?**
   Connect/read/write на уровне клиента (OkHttp/HC5/Reactor Netty) + логический `TimeLimiter`. Пример WebClient:

```java
HttpClient hc = HttpClient.create()
 .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 300)
 .responseTimeout(Duration.ofMillis(700));
WebClient wc = WebClient.builder()
 .clientConnector(new ReactorClientHttpConnector(hc)).build();
```

6. **Чем отличаются absolute deadline и per-hop таймауты?**
   Per-hop — «плановые» лимиты для каждого шага; absolute — общий жёсткий срок завершения. Внутренние таймауты всегда должны быть **короче** внешнего SLO и подчиняться дедлайну.

7. **Как корректно «отменять» работу в JVM?**
   Кооперативно: в блокирующем коде уважать `InterruptedException` и закрывать ресурсы; в реактивном — `timeout()/dispose()`. В `CompletableFuture` — `orTimeout/cancel(true)`; в WebClient — оператор `.timeout()`.

8. **Когда ретраи допустимы, а когда нет?**
   Только при transient-ошибках (timeouts, 5xx, 429). Не ретраим 4xx (бизнес-ошибки). Для POST — лишь при обеспеченной идемпотентности (Idempotency-Key, дедуп на сервере).

9. **Почему backoff нужен с jitter?**
   Экспоненциальный backoff снижает нагрузку; jitter рассинхронизирует клиентов, предотвращая «ударный» ретрай-шторм. Пример Resilience4j:

```java
RetryConfig cfg = RetryConfig.custom()
 .maxAttempts(2).waitDuration(150ms)
 .enableExponentialBackoff().exponentialBackoffMultiplier(2)
 .build();
```

10. **Что такое hedged (дублирующие) запросы?**
    Параллельный запуск нескольких чтений (в разные зоны) и взятие первого успешного. Применять только для идемпотентных GET и с ограничением скорости, чтобы не удвоить нагрузку.

11. **Как обеспечить идемпотентность POST?**
    Клиент шлёт `Idempotency-Key`. Сервер хранит `processed_ops(op_key PK, result)` и при повторе возвращает тот же результат. Это защищает от дублей/двойных списаний.

```sql
create table processed_ops(op_key text primary key, result jsonb);
```

12. **Модель состояний Circuit Breaker и ключевые пороги?**
    Closed → собираем статистику; Open → мгновенный отказ; Half-Open → пробные запросы. Пороги: `failureRateThreshold`, `slowCallRateThreshold`, `slowCallDurationThreshold`, `minimumNumberOfCalls`, `waitDurationInOpenState`.

13. **Когда CB должен открываться даже без ошибок?**
    При высокой доле «медленных» вызовов (slow-call rate). Это сигнал системной деградации SLA, даже если коды ответов 200.

14. **Что происходит в Half-Open и как его настроить?**
    Допускается ограниченное число проб (`permittedNumberOfCallsInHalfOpenState`). Если успешны (и быстры) — CB закрывается, иначе снова open. Часто включают `automaticTransitionFromOpenToHalfOpenEnabled=true`.

15. **Какой порядок политик считается правильным и почему?**
    `TimeLimiter → Retry → CircuitBreaker → Bulkhead → RateLimiter`. Сначала ограничиваем попытку по времени, затем даём шанс на один повтор, потом CB принимает агрегированное решение, а «физические» ограничители защищают процесс от перегруза.

16. **Чем Bulkhead отличается от Rate Limiter?**
    Bulkhead ограничивает **одновременность** (потоки/permits/очередь), Rate Limiter — **скорость** (токены за окно). Часто используются вместе: «не больше N параллельно» и «не больше M в секунду».

17. **Когда выбрать ThreadPoolBulkhead, а когда SemaphoreBulkhead?**
    Для блокирующих вызовов и изоляции пулом потоков — ThreadPoolBulkhead; для неблокирующих (WebFlux, быстрые операции) — SemaphoreBulkhead (permits). Всегда используйте **ограниченную** очередь.

18. **Как реализовать load shedding?**
    При достижении порогов (очередь, CPU, GC-паузы) быстро отвечать 429/503 для некритичных путей, не накапливая работу в очереди. Это сохраняет ядро сервиса живым.

19. **Что такое адаптивная конкуренция (AIMD/gradient)?**
    Динамическая подстройка permits по наблюдаемой латентности: если p95 ниже целевого — слегка увеличиваем, при росте — резко сокращаем (AIMD). Помогает «ехать у границы» без ручного тюнинга.

20. **Как описать fallback правильно?**
    Как заранее спроектированную деградацию: кеш/снапшот, частичный ответ, stub. Он должен быть быстрым и надёжным, с явной маркировкой в ответе (флаг/заголовок). В Resilience4j — `fallbackMethod` или `Try.recover`.

21. **Какие HTTP-статусы маппить на типовые отказы устойчивости?**
    `TimeoutException` → 504, `CallNotPermittedException` (CB open) → 503, `RequestNotPermitted` (rate) → 429, `BulkheadFullException` → 429/503. Используйте `@ControllerAdvice` + `ProblemDetail`.

22. **Как интегрировать Resilience4j в Spring Boot «из коробки»?**
    Через аннотации `@TimeLimiter/@Retry/@CircuitBreaker/@Bulkhead/@RateLimiter` с конфигом в `application.yml`. Метрики автоматически уедут в Micrometer.

23. **Как настроить таймауты и пулы для Feign/RestTemplate?**
    Feign: клиент OkHttp/HC5 с connect/read, лимиты пула. RestTemplate: `HttpComponentsClientHttpRequestFactory` с `setConnectTimeout`/`setReadTimeout` и пулом. Делать **пер-клиентные** бины под разные зависимости.

24. **Какие JDBC-параметры критичны для устойчивости?**
    `HikariCP.maximumPoolSize/connectionTimeout/maxLifetime` и `Statement.setQueryTimeout`/`statement_timeout` (Postgres). Без query timeout долгие SQL повиснут и забьют пул.

25. **Чем R2DBC отличается в контексте устойчивости?**
    Он неблокирующий, но таймауты всё равно нужны (`.timeout(Duration)` + параметры пула/драйвера). Не блокируйте event-loop; любые блокировки — на `boundedElastic`.

26. **Зачем Outbox/Inbox под ретраями?**
    Outbox делает «write + publish» атомарным (одна БД-транзакция + ретранслятор). Inbox у потребителя даёт дедуп дубликатов (at-least-once становится безопасным). Это фундамент согласованности при ретраях.

27. **Как защищаться от «двойных списаний» в платежах?**
    Idempotency-Key на входе, сохранение результата по ключу, проверка «у провайдера уже был чардж?» перед повтором, reconcile-процессы. Любые повторы — возвращают прежний результат.

28. **Какие метрики/сигналы включить в алёрты?**
    `slow_call_rate`, p95/p99, `state=open` у CB, `rejected.calls` у bulkhead/ratelimiter, глубина очередей, рост таймаутов и rate ошибок. Держите дашборды «бизнес → тех → инфра».

29. **Как тестировать устойчивость?**
    Интеграционные тесты WireMock/MockWebServer (429/5xx/timeout, `Retry-After`), сетевые фейлы через Toxiproxy, проверка событий CB/Retry/metrics. Нагрузочные прогоны со штормом ретраев и деградацией зависимостей.

30. **Распространённые анти-паттерны?**
    Бесконечные ретраи без backoff, одинаковые таймауты на всех слоях, Retry «после» CB, бесконечные очереди у bulkhead, отсутствие дедлайна, микс блокирующего кода в реактивном event-loop, отсутствие идемпотентности у POST.
