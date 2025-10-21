---
layout: page
title: "Асихронность и планировщики"
permalink: /spring/async_scheduler
---

# 0. Введение

## Что такое асинхронность

Асинхронность — это способ выполнения операций, при котором вызывающая сторона не блокируется в ожидании результата и может продолжать работу, пока операция выполняется «в фоне». В Java это выражается через **будущие значения** (например, `CompletableFuture`), **коллбеки** или **реактивные стримы** (интерфейсы `Publisher/Subscriber` из Reactive Streams). Идея в том, чтобы эффективнее использовать время: не простаивать во время I/O (сетевые/дисковые операции), а выполнять другие задачи.

Важно отличать асинхронность от «параллельности»: асинхронный вызов может выполняться в том же потоке (event-loop) и просто не блокировать его, передав управление; параллельность же означает выполнение на нескольких потоках/ядрах одновременно. В высоконагруженных веб-сервисах асинхронность чаще всего используется, чтобы скрывать задержки I/O и повышать **throughput** без линейного роста числа потоков.

Асинхронность влияет на **программистскую модель**: код становится событийным/композиционным. Вместо «сделал вызов → подождал → вернулся», вы пишете «сделал вызов → подписался на завершение → продолжил». Это требует дисциплины: продуманной обработки ошибок, таймаутов, отмены и управления ресурсами (пулы потоков, очереди, лимиты).

В экосистеме Spring есть как **блокирующий стек** (Spring MVC + `RestTemplate`/JDBC), где асинхронность достигается пулом потоков и `@Async`, так и **неблокирующий стек** (Spring WebFlux + `WebClient`/R2DBC), где асинхронность и backpressure встроены в саму модель (Reactor). Оба подхода жизнеспособны — выбираем под задачу.

```java
// Простейший асинхронный вызов на чистом JDK: CompletableFuture
CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> httpGet("https://api"), myIoPool);
f.thenApply(resp -> parse(resp))
 .exceptionally(ex -> fallback())
 .orTimeout(800, TimeUnit.MILLISECONDS);
```

## Что такое планировщик

Планировщик — это компонент, который **запускает задачи по расписанию** (cron/каждые N секунд/с задержкой) или управляет их **распределением по потокам** (executor/scheduler). В Java роль планировщиков выполняют `ExecutorService`/`ScheduledExecutorService`, в реактивном стеке — `Schedulers` (Reactor), а в Spring — `TaskScheduler` и аннотации `@Scheduled`.

Планировщики бывают **встроенные** (в том же процессе, например Spring Scheduling) и **внешние** (Kubernetes CronJob, Airflow, Temporal). Встроенные проще, но их сложнее координировать в кластере; внешние дают персистентность задач, мониторинг и масштабирование, но сложнее в эксплуатации и требуют сетевой оркестрации.

Ключевая идея планировщика — **детерминированный запуск** с учётом зон времени, пропусков (misfire), приоритетов и ограничений ресурсов. В системах с несколькими экземплярами приложения нужен механизм **single-runner** (кто-то один выполняет джобу), обычно через распределённые локи (ShedLock, Redis, БД).

В проде планировщик — это ещё и **наблюдаемость**: метрики (время выполнения, частота, ошибки), алёрты на пропуски, аудит. Без этого планировщики часто превращаются в «чёрный ящик», где задачи «вроде бы крутятся».

```java
// Простое планирование на JDK: каждые 10 секунд
var scheduler = Executors.newSingleThreadScheduledExecutor();
scheduler.scheduleAtFixedRate(() -> runJob(), 0, 10, TimeUnit.SECONDS);
```

## Зачем они все нужны

Асинхронность нужна, чтобы **повысить пропускную способность** и **снизить латентность** при I/O-нагрузке: пока ждём внешний сервис/БД/диск, поток не простаивает. Планировщики нужны, чтобы **автоматизировать регулярную работу**: обработка отчётов, ретраи неуспешных операций, очистка данных, синхронизации.

В сервис-ориентированных архитектурах асинхронность позволяет **развязать компоненты**: запрос не ждёт долгий процесс, а ставит задачу в очередь, ответ возвращается позже (event-driven). Планировщики запускают **пакетные/периодические** задачи вне критического пути пользовательских запросов.

Асинхронность также помогает **соблюдать SLA**: можно назначить **дедлайны** для подопераций, отменять их при истечении бюджета времени и деградировать ответ частично, вместо «всё или ничего». Планировщики фиксируют **окно запуска** (например, ночной бэкап), когда нагрузка на систему минимальна.

Наконец, это инструмент **устойчивости**: ретраи с экспоненциальной паузой, таймауты, ограничители параллелизма, распределение по очередям/приоритетам — всё это реализуется через асинхронные механизмы и планировщики.

## Где используются

В веб-API — для исходящих HTTP-вызовов (оплата, профили, рекомендации), файловых операций (S3/MinIO), обращений к внешним БД. В интеграциях — для **параллельного фанаута** (позвать сразу 2–3 сервиса и объединить результат). В обработке данных — для **batch-джоб** (агрегации, экспорты), которые лучше запускать по расписанию.

В аналитике/IoT — для **стриминга**: потребление сообщений, обработка событий, публикация результатов. В админ-частях — для «длинных» операций: перерасчёт индексов, массовые миграции. В мобильных/SPA-бекендах — для SSE/WebSocket (реактивный стриминг).

Планировщики повсеместны: периодическое **очищение кэшей**, **повторная отправка** зависших платежей, **реплики** данных, **ежедневные отчёты**. В микросервисах: хранение и исполнение оркестраций (Temporal/Zeebe) или запуск сервисных джоб в Kubernetes CronJob.

И ещё — в **DevOps**: асинхронные health-checks, перекат конфигураций без простоя, «мягкие» остановки (graceful shutdown) с ожиданием фоновых задач.

## Какие задачи решают

Асинхронность решает задачи **скрытия ожиданий** (I/O), **параллелизма** (CPU-батчи), **декомпозиции** (разбили на подзадачи и собрали), **устойчивости** (ретраи/таймауты). Планировщики — **регулярность** (cron), **отложенный старт**, **синглтоны в кластере**, **управление окном** (ночные окна, исключения по календарю).

Вместе они позволяют строить **реактивные и надёжные** системы, где пользовательские запросы быстрые и не зависят от медленных подсистем, а тяжёлые процессы идут по расписанию и контролируются метриками/алертами. Это снижает **стоимость владения**: меньше инцидентов из-за блокировок, предсказуемые окна, понятная эксплуатация.

Они также помогают соблюдать **договорённости** с внешними системами: лимиты RPS, расписания выгрузок, окна доступности. Вместо «стрелять себе в ногу» мы аккуратно дозируем нагрузку, откладываем работу, повторяем корректно и компенсируем ошибки.

И наконец, это про **качество UX**: быстро отвечать пользователю, даже если часть данных придёт позже; показывать прогресс/статус фоновых операций; избегать «зависших» экранов и таймаутов.

---

# 1. Зачем асинхронность и где она уместна

## Цели: throughput при I/O-нагрузке, латентность, развязка компонентов

Главная цель — **рост пропускной способности** без линейного роста потоков. В блокирующей модели каждый I/O-вызов «замораживает» поток. Асинхронность позволяет «вклинить» другие задачи в это окно ожидания, тем самым обслужить больше запросов на том же железе.

Вторая цель — **снижение латентности**. Мы можем параллелить независимые вызовы (фанаут), накладывать дедлайны, отдавать частичный ответ. Пользователь чаще видит результат быстрее, а «хвост» вычислений доезжает позже (например, прогрессивная загрузка).

Третья цель — **развязка**. Вместо того чтобы держать HTTP-запрос открытым на десятки секунд, мы ставим работу в очередь, а клиент получает `202 Accepted` + ссылку на статус. Это упрощает SLA и снимает давление на фронт.

Четвёртая — **устойчивость**: через ретраи/таймауты мы смягчаем флуктуации зависимостей, а через планировщики — переносим тяжёлую работу в «тихие» окна.

```java
// Параллельный фанаут с дедлайном на JDK 21 (Structured Concurrency)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
  var f1 = scope.fork(() -> usersClient.get(id));      // I/O
  var f2 = scope.fork(() -> ordersClient.list(id));    // I/O
  scope.joinUntil(Instant.now().plusMillis(800));      // общий дедлайн
  var user = f1.resultNow();
  var orders = f2.resultNow();
  return assemble(user, orders);
}
```

## Concurrency vs Parallelism: конкурентность и параллелизм

**Конкурентность** — это способность системы продвигать несколько задач, «переключаясь» между ними, скрывая ожидания (чаще про I/O). **Параллелизм** — одновременное выполнение на нескольких ядрах (CPU-bound). Асинхронность — про конкурентность; параллелизм требует ресурсов (ядра) и масштабируется до предела CPU.

В веб-API большая часть времени — в сети/БД. Там выигрывает конкурентность: неблокирующий I/O, event-loop, ограниченные пулы. В аналитических батчах — параллелизм: много CPU-тяжёлых вычислений → `ForkJoinPool`, `parallelStream()`, map-reduce, Spark.

Важно не путать инструменты: запускать CPU-батч в «I/O-пуле» — плохая идея (будет оккупация потоков), как и ожидать чуда от event-loop при тяжёлом CPU — он «задушится». Разделяйте пулы и следите за метриками.

Наконец, следите за **Amdahl’s law**: ускорение ограничено последовательной частью. Иногда лучше оптимизировать алгоритм/кэш, чем «крестить» его потоками.

## Типы задач: I/O-шоты, CPU-батчи, долгоживущие/периодические, оркестрации

**Короткие I/O-шоты** — запросы к БД/HTTP: идеально для асинхронности, timeouts, ограничителей параллелизма. **CPU-батчи** — расчёты/кодеки: лучше выделенный `ForkJoinPool`/fixed pool и чёткий лимит.

**Долгоживущие** — генерация отчёта 2–10 минут: переносим в планировщик/очередь, возвращаем 202 и статусный эндпоинт. **Периодические** — cron-задачи. **Оркестрации** — последовательность шагов с компенсациями (сага); лучше выносить во внешние движки (Temporal/Zeebe) при сложной логике и длительности.

Классифицируйте задачи и выбирайте подходящий инструмент. Это экономит ресурсы и делает поведение предсказуемым.

```java
// CPU-bound пул под парсинг больших файлов
ExecutorService cpuPool = new ThreadPoolExecutor(
    Runtime.getRuntime().availableProcessors(), 
    Runtime.getRuntime().availableProcessors(),
    0, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    new NamedThreadFactory("cpu-parse"),
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure
);
```

## SLA/SLO: тайм-бюджеты, дедлайны, отмена, идемпотентность

SLA/SLO диктуют **тайм-бюджеты**: если p95 ответа 300 мс, внутренним шагам нельзя «есть» по секунде. Назначайте **дедлайны** подвызывам и отменяйте их по истечении — это не «плохая практика», это защита сервиса.

**Отмена (cancellation)** должна быть кооперативной: при отмене будущего/Publisher нижние уровни должны получать сигнал и прекращать работу (HTTP-клиенты, драйверы БД). Если библиотека такого не умеет — ставьте короткие таймауты и не запускайте «бесконечное».

**Идемпотентность** критична для ретраев: используйте `Idempotency-Key` для POST, делайте PUT/DELETE идемпотентными по дизайну. Так вы спокойно ретраите «временные» ошибки, не плодя дублей.

Хорошая практика — **deadline propagation**: прокиньте дедлайн в заголовки (`X-Deadline-Millis`) или контекст, чтобы нижние клиенты обрезали операции раньше, а не «тянули до упора».

---

# 2. Потоки в JVM: платформа и настройки

## ОС-потоки, ForkJoinPool, свитчи, cache locality и GC

Каждый Java-поток (до Loom) — это **ОС-поток** с собственным стеком (обычно 1–2 МБ). Много потоков → больше памяти, больше **context switch** и нагрузка на планировщик ядра. Частые свитчи ухудшают **cache locality** (данные вываливаются из кэша CPU), что бьёт по производительности.

`ForkJoinPool` — специализированный пул с **ворк-стилингом** для мелких задач (fork/join). Он эффективен для рекурсивных/разделяемых вычислений. По умолчанию `CompletableFuture.supplyAsync` использует **общий** FJ-пул — это удобно, но опасно: можно «забить» его тяжёлыми задачами.

GC тоже страдает от «штормов» мелких задач с кучей короткоживущих объектов. Наблюдайте метрики аллокаций/пауз и избегайте лишних копирований/буферизаций.

Итог: планируйте количество потоков, избегайте избыточных контекст-свитчей, разделяйте пулы под разные типы нагрузки.

## Пулы: fixed/cached/Work-Stealing; CPU-bound и I/O-bound

`fixed` — фиксированное число воркеров + очередь: стабильно и предсказуемо. `cached` — растущий пул без очереди (может «раздуться» и убить систему). `Work-Stealing` (`ForkJoinPool`) — хорош для CPU-bound мелких задач.

Правило: для **CPU-bound** ставим `N = cores` (±1), чтобы не было лишних свитчей. Для **I/O-bound** допустимо `N ≫ cores`, потому что потоки часто ждут I/O — но лимитируйте **очередь** и **время**, чтобы не копить тысячи подвешенных задач.

В проде у каждого пула должна быть **политика отказа** (`RejectedExecutionHandler`): `CallerRunsPolicy` создаёт «естественный» backpressure (заставляет вызывающего выполнять задачу), вместо того чтобы бросать всё подряд в бездонную пропасть.

```java
ExecutorService ioPool = new ThreadPoolExecutor(
    64, 64, 0, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new NamedThreadFactory("io"),
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure
);
```

## Virtual Threads (Loom): когда выгодно, какие риски

**Virtual Threads** (JDK 21) — лёгкие потоки JVM, мультиплексируемые поверх малого числа ОС-потоков. Они идеальны для **I/O-интенсивной** нагрузки: можно создать тысячи V-threads, каждый «блокирует» на I/O без реальной блокировки ОС-потока — планировщик просто паркует V-thread.

Выгода: простая **императивная** модель (писать «как обычно», без CF/реактивщины) при высокой конкурентности. Риски: **блокирующие синхронизации** (`synchronized`, мониторы, глобальные локи) и «тяжёлые» блочные библиотеки (например, драйверы, которые «крутят» CPU) всё равно будут мешать. Также следите за **ресурсами** (сокеты, файлы): V-threads не магия — открытых дескрипторов становится много.

Используйте `Executors.newVirtualThreadPerTaskExecutor()` для задач, где доминирует I/O; под CPU-bound по-прежнему лучше фиксированный пул. Мигрируя, обязательно нагрузочно протестируйте — не все библиотеки одинаково дружат с Loom.

```java
try (var vexec = Executors.newVirtualThreadPerTaskExecutor()) {
  List<Callable<String>> calls = List.of(
      () -> httpGet("https://a"), () -> httpGet("https://b"));
  vexec.invokeAll(calls); // тысячи подобных задач на V-threads
}
```

## Диагностика: thread dump, deadlock, starvation, лимиты очередей

**Thread dump** (`jcmd`, `jstack`) — первая линия при зависаниях: ищите блокировки (`BLOCKED`), ожидания I/O (`WAITING on …`), длинные стеки. **Deadlock** детектится теми же инструментами (JDK сам подсвечивает взаимные блокировки).

**Starvation** — задачи не получают CPU (очередь огромная, приоритеты кривые). Следите за метриками пула: глубина очереди, среднее/максимальное время ожидания, количество отклонённых задач. Выставляйте **лимиты очередей** и обрабатывайте отказ корректно (ошибка 503/429 и «попробуйте позже»).

В проде держите кнопки: включить **дополнительное логирование** в пулах, снять дампы, ограничить RPS на входе. Это поможет стабилизировать систему в аварии.

---

# 3. Асинхронность на чистом JDK

## `ExecutorService`/`CompletableFuture`: композиция, таймауты, ошибки, отмена

`ExecutorService` — базовый интерфейс пулов. `CompletableFuture` — высокоуровневый API для **композиции** асинхронных задач: `thenApply`, `thenCompose`, `allOf`, `anyOf`. Он позволяет строить цепочки, обрабатывать ошибки (`exceptionally`, `handle`) и задавать таймауты (`orTimeout`, `completeOnTimeout`).

Крайне важно **использовать правильный пул**: по умолчанию `supplyAsync` пойдёт в общий `ForkJoinPool.commonPool()`. В проде лучше явно передавать свой `Executor` (I/O-пул, CPU-пул), чтобы избежать взаимного влияния между подсистемами.

Ошибки обрабатывайте **локально** и централизованно: в местах, где ошибка «ожидаема», — `exceptionally`, в конце — общий «наблюдатель» для логирования. **Отмена** (`cancel(true)`) кооперативна: код задачи должен периодически проверять `Thread.currentThread().isInterrupted()` и корректно завершаться.

```java
CompletableFuture<User> u = CompletableFuture.supplyAsync(() -> usersClient.get(id), ioPool);
CompletableFuture<List<Order>> o = CompletableFuture.supplyAsync(() -> ordersClient.list(id), ioPool);

UserWithOrders res = u.thenCombine(o, (user, orders) -> new UserWithOrders(user, orders))
    .orTimeout(800, TimeUnit.MILLISECONDS)
    .exceptionally(ex -> fallbackUserWithOrders(id));
```

## Structured Concurrency (JDK 21): родительский скоуп, совместное завершение, дедлайны

**Structured Concurrency** вводит идею «родитель-порождает-детей-и-ждёт-их-завершения». Это делает параллельные сценарии **компактными** и **безопасными**: все подзадачи управляются в одном месте, отменяются при ошибках, дедлайны общие.

Есть разные политики: `ShutdownOnFailure` (если любая задача упала — отменить остальные) и `ShutdownOnSuccess` (достаточно первой успешной). Встроенные методы `join()`, `joinUntil(deadline)` упрощают дедлайны; `resultNow()` читается только после `join`.

Структурная конкуррентность помогает избежать «потерянных» задач и утечек. Код становится ближе к последовательному по читабельности, но при этом эффективно использует конкурентность.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
  var a = scope.fork(() -> callA());
  var b = scope.fork(() -> callB());
  scope.joinUntil(Instant.now().plusMillis(700));
  return combine(a.resultNow(), b.resultNow());
} catch (TimeoutException e) {
  return partial(); // деградация
}
```

## Flow API (Reactive Streams): `Publisher/Subscriber`, backpressure

С Java 9 появился **Flow API** — минимальный стандарт Reactive Streams: `Publisher`, `Subscriber`, `Subscription`, `Processor`. Он решает задачу **backpressure** — потребитель запрашивает ровно столько элементов, сколько готов обработать (`request(n)`), а производитель не «затапливает» его.

Сам по себе Flow — низкоуровневый API; обычно используют библиотеки поверх (Project Reactor, RxJava), которые дают сотни операторов трансформации. Но знать основы полезно: это протокол, на котором строятся реактивные клиенты/серверы и неблокирующие драйверы.

Flow полезен там, где нужно **потоковое** потребление: обработка событий, SSE, реактивные репозитории. Для «просто асинхронного HTTP» чаще достаточно `CompletableFuture` или WebClient.

```java
class IntPublisher implements Flow.Publisher<Integer> {
  @Override
  public void subscribe(Flow.Subscriber<? super Integer> s) {
    s.onSubscribe(new Flow.Subscription() {
      int i = 0; boolean cancelled = false;
      @Override public void request(long n) {
        for (long k = 0; k < n && !cancelled && i < 100; k++) s.onNext(i++);
        if (i >= 100) s.onComplete();
      }
      @Override public void cancel() { cancelled = true; }
    });
  }
}
```

## Безопасность: `ThreadFactory` с именами, MDC-пропагация контекста

Для диагностики важно **именовать потоки**: свой `ThreadFactory` поможет быстро понять в дампе, какой пул «завис». Ещё важнее — **прокидывать контекст логирования** (`MDC`) в асинхронные задачи, чтобы `traceId/requestId` не терялся между потоками.

Паттерн: обёртывать `Runnable/Callable` в «контекстные» врапперы, которые копируют MDC из текущего потока в пул и очищают после выполнения. В Spring это проще — Micrometer Tracing/Logback MDC bridge делают большую часть работы автоматически; но для чистого JDK полезна ручная обёртка.

Также следите за **SecurityContext/Locale** (в Spring) — при `@Async` их нужно явно пропагировать (Spring делает это с `DelegatingSecurityContextExecutor`/`TaskDecorator`).

```java
class MdcAwareThreadFactory implements ThreadFactory {
  private final String prefix; private final AtomicInteger idx = new AtomicInteger();
  MdcAwareThreadFactory(String prefix) { this.prefix = prefix; }
  @Override public Thread newThread(Runnable r) {
    var t = new Thread(() -> {
      var ctx = MDC.getCopyOfContextMap();
      try { if (ctx != null) MDC.setContextMap(ctx); r.run(); }
      finally { MDC.clear(); }
    });
    t.setName(prefix + "-" + idx.incrementAndGet());
    t.setDaemon(true);
    return t;
  }
}

// Использование
ExecutorService exec = new ThreadPoolExecutor(
    16, 16, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<>(500),
    new MdcAwareThreadFactory("io"), new ThreadPoolExecutor.CallerRunsPolicy());
```

отлично — продолжаю «Асинхронность и планировщики» и подробно раскрываю **подтемы 4–7**. Для каждого подпункта — 4+ абзацев и рабочие фрагменты кода (Java 17+, Spring Boot 3.x).

---

# 4. Spring Boot: `@Async` и управление пулами

## `@Async` на методах сервисов: свой `TaskExecutor` per-use-case, изоляция блокирующего I/O

Аннотация `@Async` позволяет выполнить метод **в другом пуле потоков** и сразу вернуть управление вызывающему. Это удобно для «побочных» действий (логирование, нотификации, выгрузки), а также для параллелизации независимых вызовов. Важно: `@Async` работает через **Spring-прокси**, значит, аннотируем **публичные методы** бинов и не вызываем их напрямую из того же класса (иначе прокси не сработает).
Разделяйте **пулы по типам нагрузки**: отдельный — для I/O (больше потоков, очередь), отдельный — для CPU (равенство с числом ядер), это исключит взаимное влияние и «голодание» задач. И обязательно давайте понятные имена потокам — это сильно помогает в диагностиках и thread-dump.

Ещё один нюанс: `@Async` — **блокирующая модель** (потоки ОС). Если приложение на WebFlux, не используйте `@Async` в горячем пути вместо Reactor — лучше оффлоадить блокирующие вызовы на `boundedElastic`. В MVC же это нормальная практика, особенно когда нужно не держать HTTP-запрос дольше необходимого (например, fire-and-forget).

Результаты `@Async` могут быть `void`, `CompletableFuture<T>` или `ListenableFuture<T>`. Если возвращаем `void`, продумайте **обработку исключений**: по умолчанию они уйдут в лог-обработчик `AsyncUncaughtExceptionHandler`. Для версий с `Future` — оборачиваем цепочками `handle/exceptionally`.

Пример: два пула и два асинхронных метода под разные задачи.

```java
@Configuration
@EnableAsync
class AsyncConfig {

  @Bean("ioExecutor")
  public Executor ioExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(32);
    ex.setMaxPoolSize(64);
    ex.setQueueCapacity(1000);
    ex.setThreadNamePrefix("io-");
    ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    ex.initialize();
    return ex;
  }

  @Bean("cpuExecutor")
  public Executor cpuExecutor() {
    int n = Runtime.getRuntime().availableProcessors();
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(n);
    ex.setMaxPoolSize(n);
    ex.setQueueCapacity(100);
    ex.setThreadNamePrefix("cpu-");
    ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    ex.initialize();
    return ex;
  }
}

@Service
class ReportService {

  @Async("ioExecutor")
  public CompletableFuture<byte[]> exportCsv(long id) {
    // блокирующие I/O операции — читаем БД, пишем в файл/облако
    byte[] bytes = doBlockingExport(id);
    return CompletableFuture.completedFuture(bytes);
  }

  @Async("cpuExecutor")
  public CompletableFuture<Result> heavyCompute(Input in) {
    return CompletableFuture.supplyAsync(() -> crunch(in)); // CPU-тяжёлое
  }
}
```

## Конфигурация: core/max/queueCapacity, политика отказа, метрики пула

Правильно подберите параметры пула. `corePoolSize` — минимальное число активных потоков, `maxPoolSize` — максимум, `queueCapacity` — глубина очереди. Для I/O нагруженного пула разумно держать **большую очередь** (но конечную), чтобы не терять задания при всплесках. Для CPU пула — очередь маленькая, чтобы быстрее сигнализировать о перегрузе.

Политика отказа (`RejectedExecutionHandler`) определяет, что делать при «полной» очереди. `CallerRunsPolicy` — хороший дефолт для бэкап-давления: вызывающий поток сам выполнит задачу, замедлив подачу новых задач. Другие варианты — `AbortPolicy` (бросить исключение) или `DiscardPolicy` (потерять задачу; почти никогда не то, что нужно).

Наблюдаемость пула — критична. В Spring Boot метрики можно получить «из коробки» через `TaskExecutionMetrics` (регистрируется автоматически для `ThreadPoolTaskExecutor`): глубина очереди, активные потоки, завершённые задачи. Включайте Actuator + Prometheus/Grafana, стройте алерты по **долгим очередям** и **росту отказов**.

Не забудьте про **логирование «долгих» задач**: TaskDecorator/Proxy вокруг `Runnable`/`Callable`, который меряет время выполнения и логирует задачи, превысившие SLO. Это помогает быстро увидеть «паразитов» в пуле.

```yaml
management:
  endpoints.web.exposure.include: health,metrics,prometheus
  metrics:
    enable:
      executor: true
```

## Исключения и контекст: `@Async` + транзакции, Security/Locale/MDC, обработка ошибок

`@Async` **разрывает транзакционные границы**: если вы вызываете асинхронный метод из транзакционного, изменения ещё могут не зафиксироваться. Поэтому «запустить после коммита» лучше через события `TransactionSynchronization` или отложенные задачи. Если нужно именно в транзакции — значит, это не `@Async`-кейс.

Контексты (Security, Locale, MDC) в новых потоках **не наследуются автоматически**. Используйте `TaskDecorator` или готовые обёртки Spring Security (`DelegatingSecurityContextAsyncTaskExecutor`), чтобы прокидывать `Authentication`. Для MDC сделайте свой `TaskDecorator`, копирующий карту.

Для методов с `void` непойманные исключения идут в `AsyncUncaughtExceptionHandler`. Зарегистрируйте свой обработчик и пишите туда понятные события (с аргументами метода). Для `Future`-вариантов подключайте `.exceptionally/handle` и превращайте ошибки в доменные.

```java
@Configuration
class AsyncErrorHandling implements AsyncConfigurer {
  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return (ex, method, params) -> log.error("Async error in {}({})", method.getName(), Arrays.toString(params), ex);
  }

  @Bean
  public TaskDecorator mdcDecorator() {
    return runnable -> {
      var contextMap = org.slf4j.MDC.getCopyOfContextMap();
      return () -> {
        try { if (contextMap != null) org.slf4j.MDC.setContextMap(contextMap); runnable.run(); }
        finally { org.slf4j.MDC.clear(); }
      };
    };
  }

  @Bean("ioExecutor")
  public Executor ioExecutor(TaskDecorator decorator) { /* как выше, но ex.setTaskDecorator(decorator); */ return null; }
}
```

## Ограничение конкуренции: семафоры и bulkhead поверх `@Async`

Даже хороший пул можно «убить», если отпустить конкуренцию бесконтрольно. Используйте **bulkhead** (шпангоут): ограничивайте **одновременные** выполнения для опасных участков (вызов внешнего API, тяжёлый экспорт). Дешёвый способ — `Semaphore`; более богато — Resilience4j Bulkhead.

Bulkhead — это не только «лимит потоков», это **очередь ожидания** + **таймаут** ожидания места. Лучше **сразу отказывать** (например, 429/503), чем зависать на минуты. А для периодических задач с высоким риском — ставьте отдельный пул с низким лимитом.

Пример с семафором:

```java
@Service
class SafeGateway {
  private final Semaphore slots = new Semaphore(10); // не больше 10 одновременных
  private final ExternalClient client;

  SafeGateway(ExternalClient client) { this.client = client; }

  @Async("ioExecutor")
  public CompletableFuture<String> call(String id) {
    boolean acquired = slots.tryAcquire();
    if (!acquired) return CompletableFuture.failedFuture(new RejectedExecutionException("busy"));
    return CompletableFuture.supplyAsync(() -> client.get(id))
        .whenComplete((r, t) -> slots.release());
  }
}
```

---

# 5. Планировщики и периодические задачи

## Spring Scheduling: `@Scheduled` (fixedRate/fixedDelay/initialDelay), TimeZone, дрейф

Spring Scheduling позволяет пометить методы `@Scheduled` и выполнять их **периодически**. `fixedRate` считает интервал **от начала** предыдущего запуска, `fixedDelay` — **от конца**. `initialDelay` задаёт стартовую задержку — полезно, чтобы не «стрелять» сразу при старте приложения. Для cron-выражений используйте `@Scheduled(cron="…", zone="…")` и задавайте явную `TimeZone`, иначе возможны сюрпризы на DST.

По умолчанию `@Scheduled` выполняется на **одном** потоке — если задача «длинная», следующие запуски будут ожидать. Вы можете задать свой `TaskScheduler` (пул планировщика) и включить **параллельное** выполнение (осторожно с идемпотентностью). Если задача не должна **перекрываться**, реализуйте **мьютекс** (лок) или проверку «уже выполняется» через флаги/блокировки.

Дрейф расписания — проблема при `fixedRate` и (особенно) при «длинных» задачах. Если важно запускать **не чаще**, выбирайте `fixedDelay`. Для cron-задач учитывайте «misfire» (пропущенный запуск) — `@Scheduled` не хранит историю, и пропущенные триггеры **не догоняются**.

Наблюдаемость: логируйте **начало/конец** задач, время исполнения, исключения. Выносите параметры (периоды) в конфиг и валидируйте их на старте.

```java
@Configuration
@EnableScheduling
class SchedulingCfg {

  @Scheduled(fixedDelayString = "${jobs.cleanup.delay-ms:60000}", initialDelay = 5000)
  public void cleanup() { /* удаляем старые записи, идемпотентно */ }

  @Scheduled(cron = "0 5 3 * * MON-FRI", zone = "Europe/Moscow")
  public void weekdayReport() { /* отчёт по будням */ }
}
```

## `TaskScheduler` vs Quartz: misfire, календари, персистентность

`TaskScheduler` — лёгкий планировщик внутри приложения. Он не хранит задания персистентно: перезапуск приложения «забывает» запланированное. Для сложных сценариев (много триггеров, приоритеты, **misfire-политики**, календари праздников, **персистентность** в БД) используйте **Quartz**.

Quartz позволяет настроить, что делать при пропущенном запуске (misfire): **выполнить сразу**, **пропустить**, **перенести**. Есть **календари** (исключения дней/часов), **job-data map** (параметры), **паузы** и **приоритеты**. Для кластера Quartz использует **JDBC Store**, чтобы только один инстанс исполнял конкретный триггер.

Минусы Quartz — сложнее конфиг, повышенные накладные расходы. Если требования простые — оставайтесь на `@Scheduled`. Если нужна **гарантированная персистентность** «запланированного на будущее» — Quartz или внешний оркестратор.

```java
@Configuration
class QuartzCfg {

  @Bean
  public JobDetail sampleJobDetail() {
    return JobBuilder.newJob(SampleJob.class)
        .withIdentity("sample")
        .usingJobData("tenant", "acme")
        .storeDurably()
        .build();
  }

  @Bean
  public Trigger sampleTrigger(JobDetail sampleJobDetail) {
    return TriggerBuilder.newTrigger()
        .forJob(sampleJobDetail)
        .withIdentity("sampleTrigger")
        .withSchedule(CronScheduleBuilder.cronSchedule("0 0/5 * * * ?")
            .withMisfireHandlingInstructionFireAndProceed())
        .build();
  }
}

public class SampleJob implements Job {
  @Override public void execute(JobExecutionContext ctx) {
    var tenant = ctx.getMergedJobDataMap().getString("tenant");
    // выполнить для tenant
  }
}
```

## Single-runner в кластере: ShedLock/распределённые локи, идемпотентность

В кластере несколько инстансов приложения одновременно вызовут `@Scheduled`-метод. Чтобы «бежал один», используйте **ShedLock**: аннотация `@SchedulerLock` и хранилище (таблица БД/Redis). Блокировка делает запуск **эксклюзивным**. Важно правильно подобрать `lockAtMostFor` (защита на случай падения экземпляра) и `lockAtLeastFor` (минимальный интервал).

Сам планировщик **не гарантирует идемпотентность** — это ваша ответственность. Внутри задачи проверьте «а делали ли уже» (по метке/версии), ведите «журнал запусков» в БД и не падайте на «повторном» выполнении. Иначе после рестарта вы легко получите повторный эффект.

ShedLock прост и надёжен для «один-бежит» сценариев. Для сложных воркфлоу лучше внешний движок.

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT10M")
class ShedLockCfg { /* DataSource + таблица shedlock */ }

@Service
class Jobs {

  @Scheduled(cron = "0 */1 * * * *")
  @SchedulerLock(name = "cleanupJob", lockAtLeastFor = "PT30S")
  public void cleanup() { /* идемпотентная очистка */ }
}
```

## Внешние планировщики: K8s CronJob, Airflow/Argo/Temporal

Когда задачи **тяжёлые/долгие**, есть зависимые шаги/ретраи/компенсации или нужна прозрачная персистентность и наблюдаемость — выносите планирование наружу.
**Kubernetes CronJob** — запуск контейнера по расписанию: просто и надёжно. Состояние — вне вашего приложения, масштабирование — Kubernetes-овское.
**Airflow/Argo** — DAG-оркестрации, зависимости, ретраи, визуализация.
**Temporal/Zeebe** — «долгоживущие» воркфлоу с состоянием, таймерами, компенсациями и надёжным хранением прогресса.

Минус внешних оркестраторов — дополнительная инфраструктура и компетенции. Но плюсы часто перевешивают: **устойчивость**, **наблюдаемость**, **операбельность**. Для критичных процессов (платежи, синхронизации) это стандарт де-факто.

```yaml
# k8s CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "5 3 * * *" # каждый день в 03:05
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: app
              image: your/app:latest
              args: ["--run-job=dailyReport"]
          restartPolicy: OnFailure
```

---

# 6. Асинхронность через сообщения и очереди

## Kafka/JMS/RabbitMQ: at-least-once, дедупликация, ключ идемпотентности

Очереди и стримы — естественный инструмент асинхронности. По умолчанию гарантия — **at-least-once**: сообщение может прийти **повторно**, поэтому **обработчик должен быть идемпотентным**. Сохраняйте «ключ идемпотентности» (messageId/aggregateId+version) в БД и отбрасывайте дубликаты.

В Kafka порядок **только внутри партиции**, поэтому выбирайте ключ маршрутизации так, чтобы сообщения агрегата попадали в одну партицию. В Rabbit для очередей можно обеспечить порядок для одного потребителя.

Для Spring Kafka отключайте авто-коммиты, подтверждайте вручную **после** успешной обработки. Это позволит безопасно ретраить при падениях и не «терять» сообщения.

```java
@EnableKafka
@Configuration
class KafkaCfg {
  @Bean
  ConcurrentKafkaListenerContainerFactory<String, String> factory(ConsumerFactory<String, String> cf) {
    var f = new ConcurrentKafkaListenerContainerFactory<String, String>();
    f.setConsumerFactory(cf);
    f.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    return f;
  }
}

@Service
class Consumer {

  @KafkaListener(topics = "orders", groupId = "orders-svc")
  public void onMessage(ConsumerRecord<String, String> rec, Acknowledgment ack) {
    if (isDuplicate(rec)) { ack.acknowledge(); return; }
    process(rec.value());
    ack.acknowledge();
  }
}
```

## Транзакционные границы: Outbox/Inbox, «write + publish» атомарно

Проблема «записал в БД, но не отправил в очередь» решается паттерном **Outbox**: в транзакции с доменной записью пишем событие в outbox-таблицу, а **отдельный конвейер** (Debezium/сервис-читатель) публикует его в Kafka. Так достигается атомарность на уровне одной БД-транзакции.

Обратный паттерн — **Inbox**: входящее сообщение перед обработкой регистрируют в таблице (с состоянием «новое»), после успешной обработки — помечают «готово». Повторная доставка просто увидит «уже обработано». Это даёт сильную идемпотентность, но требует дисциплины.

Kafka-транзакции (**idempotent producer**, `enable.idempotence=true`) решают только часть пути (между продюсером и брокером). Для «end-to-end» лучше доверять Outbox/Inbox + идемпотентной обработке.

```java
@Transactional
public void createOrder(Order o) {
  orderRepo.save(o);
  outboxRepo.save(new Outbox("order.created", toJson(o)));
  // коммит ⇒ и данные, и событие в БД
}
```

## Порядок и партиции: ключи маршрутизации, DLQ/parking lot; «ядовитые» сообщения

В Kafka порядок определяется ключом: выбирайте ключ так, чтобы все события агрегата шли в одну партицию (например, `orderId`). Это гарантирует правильную последовательность для конкретного агрегата, и вы избежите гонок статуса.

**Ядовитые** сообщения (всегда падают) нельзя ретраить бесконечно. Настройте **DLQ** (dead letter queue): после N попыток отправляйте сообщение в отдельную тему, а основная консьюмер-группа двигается дальше. Для анализа держите «parking lot» — ручную обработку.

В Spring Kafka используйте `DeadLetterPublishingRecoverer` + `DefaultErrorHandler` с backoff и количеством попыток.

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
  var recoverer = new DeadLetterPublishingRecoverer(template);
  var backoff = new FixedBackOff(500L, 3L);
  return new DefaultErrorHandler(recoverer, backoff);
}
```

## Долгие процессы: Saga/оркестратор vs хореография, компенсации

Саги — способ управлять **долгими многошаговыми** процессами без распределённых транзакций.
**Оркестратор** (центральный координатор) посылает команды сервисам и ждёт ответы/события; он же инициирует **компенсации** (отмена/возвраты) при ошибке шага.
**Хореография** — без центра: сервисы реагируют на события и публикуют свои — проще, но сложнее контролировать ветвления/компенсации.

Компенсации нужно проектировать **с начала**: для каждого шага — обратный шаг (отмена/рефанд/удаление/статус). В противном случае «зависшие» частичные состояния превратятся в ручной колхоз.

Если процесс сложный/долгий — смотрите в сторону Temporal/Zeebe: сохранение состояния воркфлоу, таймеры, ретраи, видимость.

```java
@Service
class OrderSaga {
  public void place(OrderCmd cmd) {
    // 1. резервируем товары
    // 2. блокируем средства
    // 3. подтверждаем заказ
    // если (2) упал ⇒ компенсировать (1)
  }
}
```

---

# 7. Реактивность и планировщики Reactor

## `Schedulers`: `parallel`/`boundedElastic`/`immediate`; правило «не блокировать event-loop»

В Reactor всё по умолчанию выполняется в **event-loop** (Netty), который нельзя блокировать — иначе весь сервер «подвиснет». Когда нужно выполнить блокирующий код (JDBC/файлы), используйте `Schedulers.boundedElastic()` — это управляемый пул для блокировки. Для CPU-тяжёлого — `Schedulers.parallel()`.

`subscribeOn` выбирает **где начнётся** выполнение цепочки, `publishOn` переключает **где продолжать** после оператора. Чётко отмечайте границы: блокирующее — на boundedElastic, тяжёлое — на parallel, всё остальное — оставляйте в event-loop.

Следите за **размером** пулов: boundedElastic динамически расширяется «по требованию» (в разумных пределах), но не делайте блокировку стилем — это выход на крайний случай. Лучше перейти на неблокирующие драйверы (R2DBC, AsynchronousFileChannel).

```java
Mono<User> findUser(long id) {
  return Mono.fromCallable(() -> jdbcTemplate.queryForObject("...", rowMapper, id))
             .subscribeOn(Schedulers.boundedElastic()); // оффлоад блокирующего JDBC
}
```

## Backpressure: `onBackpressure*`, лимиты буферов, `timeout`, `retry` с экспонентой

Backpressure — механизм, который защищает потребителя от «потопа» производителя. В Reactor источники типа `Flux.create`/`interval` могут генерировать быстрее, чем подписчик успевает. Стратегии: `onBackpressureBuffer` (буферизовать до лимита), `onBackpressureDrop` (отбрасывать новые/старые), `onBackpressureLatest` (оставлять последние).

Старайтесь **лимитировать** буферы и выставлять **timeouts** на долгие операции: `timeout(Duration)` преобразует «вечные ожидания» в понятные ошибки. Ретраи (`retryWhen(Retry.backoff(...))`) используйте с **джиттером** и ограничением числа попыток, чтобы не устроить «шторм» при аварии. Все эти политики — часть вашей **SLA**.

Для сетевых клиентов (`WebClient`) ограничивайте **размер тела в памяти** (`maxInMemorySize`) и используйте **стриминг** (например, `bodyToFlux(DataBuffer)`) для больших ответов, иначе легко словить OOM.

```java
Flux<Data> stream = source()
    .onBackpressureBuffer(1000, BufferOverflowStrategy.DROP_OLDEST)
    .timeout(Duration.ofSeconds(5))
    .retryWhen(Retry.backoff(2, Duration.ofMillis(200)).jitter(0.3));
```

## Мосты: WebFlux ↔ блокирующие репозитории (изоляция на `boundedElastic`)

Интеграция WebFlux с блокирующими зависимостями должна быть **точечной** и **изолированной**. Если у вас JPA/JDBC — оборачивайте каждый блокирующий вызов в `Mono.fromCallable(...).subscribeOn(boundedElastic)`. Не используйте `.block()` в контроллерах — это возвращает нас в блокирующий мир и убивает масштабирование.

Лучше — переходить на **реактивные драйверы** (R2DBC/Reactive Mongo/Redis), но при миграции допустим «мост». Важно измерять: сколько времени уходит в boundedElastic, нет ли «просадок» при всплесках, не растут ли очереди.

При работе с файлами — `DataBufferUtils.read` + реактивная передача в ответ, без чтения всего файла в память.

```java
@GetMapping("/api/file/{name}")
public Mono<Void> download(@PathVariable String name, ServerHttpResponse resp) {
  Path p = storage.resolve(name);
  resp.getHeaders().setContentType(MediaType.APPLICATION_OCTET_STREAM);
  return DataBufferUtils.read(p, 0, 64 * 1024).flatMap(resp::writeWith).then();
}
```

## Context propagation: Security/MDC в Reactor Context, operators-логи

В реактивном стеке контекст (Security, MDC, correlation) хранится в **Reactor Context**. Доступ к нему — через `deferContextual`/`contextWrite`. Spring Security кладёт `SecurityContext` туда; получить его можно `ReactiveSecurityContextHolder.getContext()` или принимать `Authentication` прямо в методе контроллера.

Чтобы MDC был в логах каждого оператора, используйте **`Hooks.enableAutomaticContextPropagation()`** (в новых версиях это идёт через Micrometer/OTel) или вручную копируйте атрибуты в `contextWrite`. Для корреляции полезно добавить фильтр, который кладёт `traceId/requestId` в контекст на входе запроса.

Логируйте **сигналы** Reactor (operator-level): `.log("tag")` и `doOnEach` в нужных местах дают видимость ошибок и отмен (`onError`, `onCancel`). Не злоупотребляйте — точечные «маяки» эффективнее, чем «включить всё подряд».

```java
@Bean
public WebFilter correlationFilter() {
  return (ex, chain) -> {
    String rid = Optional.ofNullable(ex.getRequest().getHeaders().getFirst("X-Request-Id"))
        .orElse(UUID.randomUUID().toString());
    return chain.filter(ex).contextWrite(ctx -> ctx.put("rid", rid));
  };
}

Mono<String> handler() {
  return Mono.deferContextual(ctx -> {
    String rid = ctx.getOrDefault("rid", "n/a");
    log.info("rid={}", rid);
    return Mono.just("ok");
  });
}
```

отлично — завершаю «Асинхронность и планировщики» подробным разбором **подтем 8–10**. Как просил, каждый подпункт — 4+ абзацев, с пояснениями терминов/аннотаций и рабочими примерами кода (Java 17+, Spring Boot 3.x).

---

# 8. Надёжность: таймауты, ретраи, ограничители

## Policy-набор: timeout → retry → circuit breaker → rate limiter → bulkhead

Первое правило устойчивости — **правильный порядок** политик. Мы сначала ограничиваем **время** операции (`timeout`), затем позволяем **повторить** «временные» сбои (`retry` с бэкоффом), дальше включаем **предохранитель** (`circuit breaker`), который размыкает цепь при серии падений, добавляем **ограничение скорости** (`rate limiter`) и, наконец, физически **лимитируем параллелизм** (`bulkhead`). Такой стек превращает хаотичные ошибки в контролируемые сценарии.

Таймаут — это «жёсткая крышка» на длительность операции. В JDK это `Future#orTimeout`, в WebClient — `.timeout(Duration)`, в Feign/OkHttp — `readTimeout/callTimeout`. Без таймаута ретраи бессмысленны — один зависший запрос может повиснуть на минуты, утащив поток.

`Retry` должен быть **селективным**: ретраим только идемпотентные операции и только временные коды (408/429/5xx, сетевые таймауты). Экспоненциальный бэкофф + **джиттер** (рандомный разброс) предотвращают «шторм» при массовых сбоях. Количество попыток ограничивайте (обычно 1–2).

`CircuitBreaker` защищает внешние системы и ваши пулы. Он переводит клиента в «half-open/open» при росте доли ошибок, и через время проверяет здоровье пробными вызовами. В паре с **bulkhead** (ограничитель параллелизма/очереди) это не даёт одному интеграционному узлу положить весь процесс.

```java
// Resilience4j: комбинируем политики вокруг вызова
var cb = CircuitBreaker.ofDefaults("ext");
var rl = RateLimiter.ofDefaults("ext");
var tl = TimeLimiter.of(Duration.ofMillis(800));
var retry = Retry.of("ext", RetryConfig.custom()
    .maxAttempts(2).waitDuration(Duration.ofMillis(200)).build());

Supplier<String> call = () -> extClient.fetch(); // любой синхронный вызов
var decorated = Decorators.ofSupplier(call)
    .withTimeLimiter(tl, Executors.newSingleThreadScheduledExecutor())
    .withRetry(retry)
    .withCircuitBreaker(cb)
    .withRateLimiter(rl)
    .decorate();
String res = Try.ofSupplier(decorated).get(); // io.vavr.control.Try
```

## Идемпотентность: ключи (PUT/POST), дедуп в БД/кэше, ловушки «exactly-once»

Идемпотентность — свойство операции давать **один и тот же эффект при повторе**. Для PUT/DELETE это естественно: «приведи ресурс в состояние X». Для POST (создания) используйте **Idempotency-Key** — уникальный ключ операции, по которому сервер вернёт **тот же** результат при повторе, а не создаст дубликат.

На стороне потребителя делайте **дедупликацию**: храните обработанные ключи/идентификаторы в БД/Redis с TTL. Это особенно важно при at-least-once доставке (Kafka/JMS): одно и то же сообщение может прийти дважды. Таблица «inbox/outbox» — классический приём, который удерживает «вторичные» эффекты.

«Exactly-once» в распределённых системах — ловушка. В реальности мы комбинируем **idempotent-by-design** и **at-least-once + дедуп**. Важно фиксировать «идемпотентный ключ» в доменной модели: например, номер платежа + версия шага, чтобы повтор не проваливал инварианты.

```sql
-- таблица дедупликации операций
create table if not exists processed_ops(
  op_key text primary key,
  processed_at timestamptz default now()
);
```

```java
boolean tryMarkProcessed(String key) {
  try { jdbc.update("insert into processed_ops(op_key) values (?)", key); return true; }
  catch (DuplicateKeyException e) { return false; } // уже обработано
}
```

## Отмена: cooperative cancellation, deadline-propagation (HTTP/DB/клиенты)

**Отмена** — это сигнал «дальше не нужно». В Java отмена кооперативная: вы прерываете поток (`interrupt`), но задача должна **проверять** флаг и корректно завершаться. В `CompletableFuture` есть `cancel(true)`, в Reactor — `Disposable#dispose()`, в WebClient — отмена подписки (**закрывает** HTTP-канал).

Пропагандируйте **дедлайны** вниз по стеку. В WebClient — `.timeout()`, в JDBC — `setQueryTimeout`/`Statement.cancel()`, в Feign/OkHttp — `callTimeout`. Для внутренних вызовов прокиньте `X-Deadline-Millis`/`X-Request-Timeout`, чтобы соседний сервис тоже ограничивал свои шаги.

Следите за «неотменяемыми» блоками (синхронные I/O без таймаута, `synchronized`): там отмена бессильна. Оборачивайте такие участки таймаутом верхнего уровня и **не повторяйте** их бесконечно — иначе получите «шторм» блокировок.

```java
// CF: отмена
var fut = CompletableFuture.supplyAsync(() -> longCall());
scheduler.schedule(() -> fut.cancel(true), 800, MILLISECONDS);

// WebClient: отмена по таймауту
Mono<String> m = webClient.get().uri("/slow").retrieve().bodyToMono(String.class)
  .timeout(Duration.ofMillis(800)); // отменит подписку и запрос
```

## Деградация: graceful shutdown, частичные успехи, компенсирующие шаги

**Деградация** — заранее продуманные «ступени вниз», когда всё плохо: вернуть частичный ответ, включить кэш, уменьшить нагрузку, временно скрыть функцонал. Это лучше, чем «упасть с 500» и ничего не сказать пользователю. Сервис должен знать, что «не обязательно всё или ничего».

**Graceful shutdown** — мягкая остановка. В Spring Boot это: `server.shutdown=graceful` + ожидание активных запросов + «дождаться» асинхронных задач и корректно закрыть очереди/пулы. Настройте `spring.lifecycle.timeout-per-shutdown-phase`, чтобы не висеть вечность. Завершайте `ExecutorService` через `shutdown()`/`awaitTermination()`.

Компенсации — обратные действия при провале шага (возврат денег, откат статуса). Их стоит проектировать **сразу**, иначе долги «компенсаций» превращаются в ручную работу. С точки зрения пользователя это «частичный успех»: заказ создан, но доставка не назначена — будет назначена позже.

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

```java
@Bean
SmartLifecycle gracefulExecutors(List<ExecutorService> pools) {
  return new SmartLifecycle() {
    public void stop(Runnable callback) {
      pools.forEach(ExecutorService::shutdown);
      pools.forEach(p -> p.awaitTermination(15, TimeUnit.SECONDS));
      callback.run();
    }
    public boolean isRunning() { return true; }
  };
}
```

---

# 9. Наблюдаемость и эксплуатация

## Метрики: per-executor timers, queue depth, rejected tasks, бизнес-метрики

Наблюдаемость начинается с **метрик**. Для пулов потоков собирайте: активные потоки, размер очереди, длину ожидания, количество **rejected**. В Spring Boot `ThreadPoolTaskExecutor` автоматически экспонирует метрики через Micrometer, если включён Actuator; но **queue depth** иногда удобнее добавить вручную как `Gauge`.

Не ограничивайтесь техническими метриками. Делайте **бизнес-метрики**: успешные/неуспешные задачи, время генерации отчёта p95/p99, количество пропусков расписаний. Это помогает видеть не только «жив ли пул», но и «жив ли бизнес».

Стройте **дистрибуции** (percentile histogram) по ключевым операциям, чтобы образы p95/p99 попадали в алёрты. По пулу полезны **rate** rejected-задач и «время в очереди» (можно мерить TaskDecorator-ом).

Теги метрик — важны. Разделяйте по **use-case** (`executor=io`, `job=dailyReport`), по тенанту/региону, если уместно. Это даст срезы и ускорит RCA.

```java
@Component
class PoolMetrics implements MeterBinder {
  private final ThreadPoolTaskExecutor ex;
  PoolMetrics(@Qualifier("ioExecutor") ThreadPoolTaskExecutor ex) { this.ex = ex; }
  @Override public void bindTo(MeterRegistry r) {
    Gauge.builder("executor.queue.size", ex, e -> e.getThreadPoolExecutor().getQueue().size())
        .tag("name", "io").register(r);
  }
}
```

## Трейсинг: OpenTelemetry/Sleuth, связь спанов «планировщик → воркер → внешние вызовы»

**Трейсинг** связывает события от планировщика до воркера и внешних вызовов. В Spring Boot 3 используйте **Micrometer Tracing** (мост к OpenTelemetry). Аннотация `@Observed` или явный `Observation` создаёт **спан** вокруг метода; интеграции WebClient/Feign/R2DBC автоматически создают дочерние спаны.

Важно, чтобы контекст трейсинга **переезжал** в асинхронные задачи. Делайте `TaskDecorator`, который переносит `io.micrometer.context.ContextSnapshot`/MDC. В реактивном стеке включите **автопропагацию** контекста (Hook/Bridge), и `traceId/spanId` окажутся в логах и спанах.

Связка «планировщик → воркер» достигается тем, что спан «schedule» становится **родителем** спана «execution». В Quartz/`@Scheduled` можно снимать «кишки» через `@Observed(name="job.run", contextualName="dailyReport")` и логи с `traceId`.

```java
@Service
class ObservedJobs {

  @Observed(name = "job.cleanup", contextualName = "cleanup")
  @Scheduled(fixedDelay = 60000)
  public void cleanup() { /* ... */ }
}
```

```kotlin
dependencies {
  implementation("io.micrometer:micrometer-tracing-bridge-otel")
  runtimeOnly("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

## Логи: корреляция traceId/spanId, sampling, аудит длительных/пропущенных задач

Логи — «истина последней мили». Добавляйте в шаблон лога `traceId`/`spanId` и `X-Request-Id`, чтобы связать записи с трейсом. В Logback это делается через `%X{traceId}`. Для высоких нагрузок включайте **семплирование** (sampling) в трейсинге, чтобы не тонуть в объёме, но держите **ошибки** на 100%.

Ведите **аудит длительных задач**: лог «старт/финиш/длительность/результат» + ключи корреляции. Порог — из SLO (например, всё дольше 5 секунд пишем WARN). Для расписаний полезно логировать **пропуски** (misfire) и причины (старт не состоялся, лок занят).

Структурируйте логи в JSON (ключ-значение) — так их проще анализировать в Loki/ELK. И никогда не пишите в логи персональные/секретные данные: маскируйте или выкидывайте поля.

```xml
<!-- logback-spring.xml (фрагмент) -->
<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
  <providers>
    <timestamp/>
    <mdc/>
    <pattern>
      <pattern>
        {
          "level":"%level",
          "logger":"%logger",
          "message":"%message",
          "traceId":"%X{traceId}",
          "spanId":"%X{spanId}"
        }
      </pattern>
    </pattern>
  </providers>
</encoder>
```

## Руководства для on-call: застрявшие джобы, «шторм» ретраев, DST/тайм-зоны

Для дежурных (on-call) нужны **плейбуки**. «Застряла джоба»: проверить ShedLock («висит лок?»), посмотреть в метрики очереди/пула, снять thread-dump, остановить входящие триггеры (disabling cron/скейл-даун реплик), вручную освободить лок с TTL, перезапустить аккуратно.

«Шторм ретраев»: отключить retry на уровне конфигурации (feature-flag), поднять backoff/jitter, увеличить таймауты соседних сервисов, включить circuit-breaker, локально снизить RPS (rate limiter). Потом искать корень: деградация зависимости, DNS, TLS, лимиты пула.

DST/тайм-зоны: cron без явной `zone` может «прыгнуть». Решение — **хранить** зоны явно, критичные джобы — в UTC или в Quartz с календарями. При переводе часов ожидать «двойные/пропущенные» слоты и быть готовыми к ручному запуску.

Полезно иметь **cheat-sheet** команд: `kubectl` для CronJob, SQL для проверки/снятия ShedLock, curl к health/metrics, `jcmd Thread.print` для дампа. Чем меньше ручных шагов и «магии», тем лучше.

```sql
-- ShedLock: посмотреть/снять лок
select * from shedlock;
delete from shedlock where name='cleanupJob' and lock_until < now();
```

---

# 10. Тестирование асинхронности и планировщиков

## Детеминизм: Awaitility/Condition, паузы vs виртуальное время (Reactor StepVerifier)

Тесты асинхронности страдают флейками из-за реального времени. **Awaitility** решает это: вы «ожидаете условие» с таймаутом и polling-интервалом, а не «спите». Это делает тесты устойчивыми к «малым» дрожаниям времени.

В реактивном мире используйте **виртуальное время**. `StepVerifier.withVirtualTime` «перематывает» планировщик и позволяет тестировать `delay/retry/timeout` без реальных пауз. Это ускоряет CI и делает тесты детерминированными.

Избегайте `Thread.sleep` в тестах — это всегда антипаттерн. Лучше «подождите условие» (очередь опустела, запись появилась в БД) или сымитируйте события (заглушка возвращает ответ через N «виртуальных» секунд).

Если всё же приходится использовать реальное время (интеграции), заложите разумные таймауты и увеличьте их на CI (медленнее), а не на локали. Флейки убиваются наблюдаемостью: печатайте промежуточные состояния и счётчики.

```java
// Awaitility
AtomicBoolean done = new AtomicBoolean(false);
exec.submit(() -> { doWork(); done.set(true); });

Awaitility.await().atMost(2, TimeUnit.SECONDS).untilTrue(done);
```

```java
// Reactor virtual time
StepVerifier.withVirtualTime(() -> service.fetchWithRetry())
  .thenAwait(Duration.ofSeconds(1)) // промотали таймеры
  .expectNextMatches(r -> r.ok())
  .verifyComplete();
```

## Контуры интеграций: Testcontainers (Kafka/Redis/DB), флейки от гонок и идемпотентные фикстуры

Для интеграций поднимайте **реальные зависимости** через **Testcontainers**: Postgres для джоб с БД, Kafka для потребителей, Redis для ShedLock. Это «ближе к правде», чем in-memory имитации, и ловит классы ошибок совместимости/сетевых задержек.

Гонки убиваются **идемпотентными фикстурами**: уникальные ключи на тест/префиксы таблиц/топиков, очистка данных в `@AfterEach`, time-boxed операции. Параллельные тесты — только при изоляции окружения (namespace на контейнер/тест).

Не забудьте **ускорители** CI: re-use контейнеров (`testcontainers.reuse.enable=true` локально), предварительный pull образов, отказ от «тяжёлых» контейнеров там, где можно WireMock. Но на PR-гейтах критичные интеграции всё равно должны быть покрыты.

```java
@Testcontainers
class KafkaIT {
  @Container static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
  }
  // … тесты продюсера/консюмера с идемпотентной логикой
}
```

## Контрактные тесты расписаний: cron-шаблоны, misfire-кейсы, «после коммита»

Расписания должны тестироваться **контрактно**. Для `@Scheduled(cron=…)` можно вынести выражение в конфиг и протестировать его библиотекой `cron-utils`: перечислить ближайшие «файринги» и проверить, что они попадают в нужное окно (например, будни 03:05). Это ловит ошибки в минутах/часах и переходах DST.

Misfire-кейсы (Quartz): приостановить планировщик, «проспать» пару триггеров, затем возобновить и проверить политику (`FireAndProceed` vs `DoNothing`). Это даёт уверенность, что после простоя не произойдёт лавина запусков.

«Запуск после коммита» тестируйте через `@TransactionalEventListener(phase = AFTER_COMMIT)`: в тесте откатывайте транзакцию и убеждайтесь, что слушатель **не сработал**, затем коммитьте и ждите через Awaitility факт запуска.

```java
@Component
class AfterCommitPublisher {
  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void onCommitted(MyEvent e) { /* запуск фоновой задачи */ }
}
```

```java
@Test
void firesOnlyAfterCommit() {
  txTemplate.executeWithoutResult(st -> {
    appDomain.doSomething(); // публикует MyEvent
    // без коммита
  });
  Awaitility.await().during(500, MILLISECONDS).until(() -> !jobStarted.get()); // не запустилось

  txTemplate.executeWithoutResult(st -> appDomain.doSomethingAndCommit()); // коммит
  Awaitility.await().atMost(1, SECONDS).untilTrue(jobStarted);
}
```

## Перформанс-тесты: пулы, агрессивные ретраи, «шторм таймаутов», лимиты Docker/K8s

Перф-тесты для асинхронности важны: именно под нагрузкой всплывают «штормы ретраев», вырождение пулов и длинные очереди. Начинайте с **микротестов**: N параллельных задач против заглушки с заданной латентностью/дребезгом, метрики «время в очереди», rejected-rate, p95. Это ловит неэффективные настройки пула и порядок политик.

Дальше — **системные** тесты: дев-кластер, **агрессивные** таймауты и ретраи (временно), симуляция деградации зависимости (Toxiproxy: задержки/дропы). Смотрите на «полку» пропускной способности, реакции CB и RateLimiter, поведение graceful shutdown.

В Docker/K8s учитывайте **лимиты**: CPU/Memory лимиты влияют на планировщик JVM и GC. При недостатке CPU пул «голодает», таймауты срабатывают слишком рано — SLO ломается. Тестируйте под теми же лимитами, что и прод.

Для CPU-критичных участков можно подключить **JMH** (микробенчмарк) — измерить производительность алгоритмов без влияния сетевых эффектов. Но для асинхронных систем главная правда — в **нагрузочных интеграционных** тестах.

```java
// JUnit «шторм ретраев» против WireMock с 50% 503 и 300мс задержкой
IntStream.range(0, 200).parallel().forEach(i -> {
  try {
    service.callWithRetry("id-" + i);
  } catch (Exception e) { /* учесть ошибки */ }
});
// собрать метрики: p95, retries per call, rejected, queue size
```

# Вопросы

---

1. **В чём разница между асинхронностью и параллелизмом?**
   Асинхронность — про то, чтобы **не ждать** завершения операции (чаще I/O) и продолжать работу; параллелизм — реальное **одновременное** выполнение на разных ядрах (CPU-bound). Асинхронность повышает throughput, параллелизм ускоряет вычисление CPU-задач. Эти вещи дополняют друг друга, но не тождественны.

2. **Когда асинхронность уместна в веб-сервисах?**
   Когда доминирует **I/O-латентность**: внешние HTTP, БД, файловое хранилище. Асинхронность позволяет скрывать ожидания, делать фанаут, возвращать частичные ответы и не держать поток «висящим». Для CPU-тяжёлого — полезнее выделенный пул/обработка вне запроса.

3. **Что делает `@Async` и почему иногда «не срабатывает»?**
   `@Async` выполняет метод в **другом пуле** через Spring-прокси. Работает только на **public**-методах бина и **между** бинами (self-invocation не триггерит прокси). Возвращаемые типы: `void`, `CompletableFuture<T>`, `ListenableFuture<T>`.

```java
@EnableAsync
@Service
class Mailer {
  @Async("ioExecutor")
  public CompletableFuture<Void> send(...) { ...; return CompletableFuture.completedFuture(null); }
}
```

4. **Как выбирать параметры пула для `@Async`?**
   Для I/O: больше потоков (например, 32–64) и конечная очередь (например, 1000). Для CPU: `cores±1` и маленькая очередь. Политика отказа — `CallerRunsPolicy` для бэк-прешсура. Метрики пула (активные, очередь, reject) обязателены.

```java
var ex = new ThreadPoolTaskExecutor();
ex.setCorePoolSize(32); ex.setMaxPoolSize(64); ex.setQueueCapacity(1000);
ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

5. **Как прокинуть Security/MDC/Locale в `@Async`?**
   Через `TaskDecorator` (копирует MDC/Locale) и/или `DelegatingSecurityContextAsyncTaskExecutor` для Security. Иначе контекст потеряется в другом потоке.

```java
@Bean TaskDecorator mdc() { return r -> () -> { var ctx=MDC.getCopyOfContextMap(); MDC.setContextMap(ctx); try{r.run();} finally{MDC.clear();}};}
```

6. **Почему `@Async` «разрывает» транзакции и как запускать «после коммита»?**
   Асинхронный метод исполняется **вне текущей транзакции** — коммита ещё может не быть. Для запуска «после коммита» используйте `@TransactionalEventListener(phase = AFTER_COMMIT)` или очередь/планировщик.

7. **`thenApply` vs `thenCompose` в `CompletableFuture`?**
   `thenApply` — трансформация результата (T→U) **без** плоского объединения; `thenCompose` — «плоская» композиция, когда коллбек сам возвращает `CompletableFuture<U>`. Для последовательных асинхронных шагов применяем `thenCompose`.

8. **Как задать таймаут/отмену для `CompletableFuture`?**
   Используйте `orTimeout(d,unit)`/`completeOnTimeout(value,d,unit)` и `cancel(true)`; отмена кооперативна — код должен уважать `InterruptedException`.

```java
cf.orTimeout(800, MILLISECONDS).exceptionally(ex -> fallback());
```

9. **Что даёт Structured Concurrency (JDK 21)?**
   Позволяет запускать «детей» в scope и **совместно управлять** ими: общие дедлайны, авто-отмена при ошибке (`ShutdownOnFailure`) или ранний успех (`ShutdownOnSuccess`). Повышает читабельность и безопасность параллельных сценариев.

```java
try(var s=new StructuredTaskScope.ShutdownOnFailure()){
  var a=s.fork(this::callA); var b=s.fork(this::callB);
  s.joinUntil(Instant.now().plusMillis(800));
  return combine(a.resultNow(), b.resultNow());
}
```

10. **Зачем Reactive Streams/Flow API и что такое backpressure?**
    Это стандарт взаимодействия `Publisher/Subscriber` с **управлением скоростью**: потребитель сам запрашивает `n` элементов, чтобы его не «затопили». Практически используется в Reactor/RxJava, в чистом JDK — низкоуровневая основа.

11. **`subscribeOn` vs `publishOn` (Reactor) и когда `boundedElastic`?**
    `subscribeOn` выбирает **где стартует** цепочка; `publishOn` — **где продолжать** после оператора. Блокирующее (JDBC/файлы) переносим на `Schedulers.boundedElastic()`, CPU-тяжёлое — `Schedulers.parallel()`. Нельзя блокировать event-loop.

12. **Почему нельзя блокировать event-loop и как «мостить» JDBC в WebFlux?**
    Блокировка Netty-потока «вешает» весь сервер. Оборачивайте блокирующие вызовы: `Mono.fromCallable(...).subscribeOn(boundedElastic)` и настраивайте короткие таймауты. Лучше переходить на R2DBC, но «мост» допустим на миграции.

13. **Какие таймауты есть у WebClient и как отменять запрос?**
    В цепочке — `.timeout(Duration)` (отмена подписки). На уровне клиента — connection/read/write timeouts через `HttpClient` (Reactor Netty). Для больших ответов — настройте `maxInMemorySize` и используйте стриминг `bodyToFlux(DataBuffer)`.

14. **Правильный порядок политик устойчивости и почему именно так?**
    `timeout` → `retry` → `circuit breaker` → `rate limiter` → `bulkhead`. Сначала ограничиваем **время**, затем даём **ограниченный шанс** повтору, дальше защищаемся **CB**, лимитируем **скорость**, и физически ограничиваем **параллелизм**.

15. **Какие ошибки стоит ретраить и почему нужна идемпотентность?**
    408/429/5xx и сетевые таймауты. Мутаторы ретраим только с **идемпотентностью** (PUT/DELETE by design, для POST — `Idempotency-Key`). Иначе повтор может создать дубли.

16. **Как работает Circuit Breaker (Resilience4j) и что такое half-open?**
    Считает долю ошибок в окне; при превышении порога «размыкает» цепь (open) и быстро отказывает. Спустя интервал входит в **half-open** и пускает ограниченное число пробных запросов: если успешны — закрывается, иначе снова open.

17. **Для чего rate limiter на клиенте и как уважать `Retry-After`?**
    Чтобы не перегружать внешний API и быть «вежливым». При 429 читайте заголовок `Retry-After` и ждите указанное время (или используйте политику с джиттером). На своей стороне ограничивайте RPS библиотеками типа Resilience4j.

18. **Чем bulkhead отличается от настроек пула?**
    Пул — общий ресурс исполнителей; **bulkhead** ограничивает **одновременность** конкретной операции/критической секции (часто семафор + очередь + таймаут ожидания). Это «шпангоут» против локальных штормов.

19. **Зачем ShedLock и что значат `lockAtMostFor/lockAtLeastFor`?**
    Для **single-runner** в кластере: чтобы плановая задача исполнялась ровно одним инстансом. `lockAtMostFor` — защита от «вечного» лока (если инстанс умер), `lockAtLeastFor` — минимальный интервал между запусками.

```java
@Scheduled(cron="0 * * * * *")
@SchedulerLock(name="cleanupJob", lockAtMostFor="PT10M", lockAtLeastFor="PT30S")
public void cleanup(){...}
```

20. **`fixedRate` vs `fixedDelay` vs `cron` (Spring Scheduling) и TimeZone/DST?**
    `fixedRate` — интервал от **старта** предыдущего, `fixedDelay` — от **окончания**. `cron` — по календарю. Явно задавайте `zone`, иначе сюрпризы на **DST**. По умолчанию `@Scheduled` — один поток: длинные задачи блокируют последующие.

21. **Когда нужен Quartz вместо `@Scheduled`?**
    Когда важны **misfire-политики**, **персистентность** расписаний, **приоритеты**, **календари** (праздники) и кластерная координация. Quartz хранит задания в БД и гарантирует исполнение одним инстансом при отказах.

22. **Как тестировать расписания и обрабатывать misfire?**
    Cron выносите в конфиг и валидируйте библиотекой (например, `cron-utils`). Для Quartz симулируйте паузу (misfire) и проверяйте политику `FireAndProceed/DoNothing`. Логируйте факт пропуска и «догонялки», если они нужны.

23. **Как обеспечить идемпотентность потребителя Kafka (at-least-once)?**
    Отключить авто-коммит, подтверждать offset **после** успешной обработки. Дедуплицировать по ключу (inbox/таблица processed_ops). Понимать, что сообщение может прийти повторно — обработчик обязан быть идемпотентным.

```java
@KafkaListener(...)
public void on(ConsumerRecord<String,String> r, Acknowledgment ack){
  if(isDuplicate(r)) { ack.acknowledge(); return; }
  process(r.value()); ack.acknowledge();
}
```

24. **Что такое Outbox/Inbox и зачем они?**
    Outbox: событие пишется **в ту же БД** в одной транзакции с доменными данными, а отдельный процесс публикует его в шину — «write + publish» становится атомарным. Inbox: входящее сообщение регистрируется и помечается обработанным — защищает от повторов.

25. **Почему «exactly-once» — миф и что делать на практике?**
    В распределённых системах E2E «ровно один раз» трудно и дорого. Практика: **idempotent-by-design** операции + **at-least-once** доставка + дедупликация (ключи, версии, TTL). Kafka транзакции решают производитель↔брокер, но не весь путь.

26. **Как сделать graceful shutdown в Spring Boot?**
    Включить `server.shutdown=graceful`, задать таймаут фазы, **остановить приём** новых задач, подождать активные, закрыть пулы/коннекты. Для `ExecutorService` — `shutdown()`→`awaitTermination()`.

```yaml
server.shutdown: graceful
spring.lifecycle.timeout-per-shutdown-phase: 20s
```

27. **Как пропагировать дедлайн вниз по стеку?**
    Передавать в заголовках (`X-Deadline-Millis`) и **реально** выставлять таймауты на клиентах: WebClient `.timeout`, JDBC `setQueryTimeout`, Feign/OkHttp `callTimeout`. Нижние уровни должны обрезать работу раньше, чем истечёт бюджет.

28. **Какие метрики нужны для наблюдаемости асинхронности?**
    Per-executor: активные потоки, размер очереди, rejected rate, среднее/макс время в очереди. Бизнес-метрики: успех/ошибка задач, p95/p99 длительности, пропуски расписаний. Для клиентов — таймеры пер-эндпоинт, распределение задержек.

29. **Как тестировать асинхронность без флейков?**
    Использовать **Awaitility** (ожидания условий вместо `sleep`) и **Reactor StepVerifier с виртуальным временем** для тестов `delay/retry/timeout`. Для интеграций — Testcontainers и разумные таймауты/пулы, изоляция данных.

```java
Awaitility.await().atMost(2, SECONDS).untilTrue(flag);
StepVerifier.withVirtualTime(() -> service.call()).thenAwait(Duration.ofSeconds(1)).expectNext(...).verifyComplete();
```

30. **Когда использовать Virtual Threads (Loom) и какие ограничения?**
    При **I/O-интенсивной** нагрузке, когда хочется сохранить простой императивный код без реактивщины. Ограничения: блокирующие синхронизации и «тяжёлые» CPU-участки всё равно мешают, ресурсы (сокеты/файлы) ограниченны. Для CPU-bound по-прежнему лучше фиксированные пулы.

```java
try(var vexec = Executors.newVirtualThreadPerTaskExecutor()){
  vexec.submit(() -> httpGet(...)); // тысячи лёгких задач
}
```
