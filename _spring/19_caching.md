---
layout: page
title: "Кэширование"
permalink: /spring/caching
---

# 0. Введение

## Что такое кэширование

Кэширование — это техника хранения **ранее вычислённых** или **полученных** данных в более быстром и/или близком хранилище, чтобы последующие обращения обслуживать быстрее и дешевле. Источник истинных данных называют **origin** (БД, внешний HTTP-сервис, файловое хранилище), а кэш — прослойкой между приложением и origin. Смысл прост: данные, которые «часто читают, но редко меняют», выгодно держать под рукой.

Кэш может быть **локальным** (в памяти процесса) или **распределённым** (отдельный кластер Redis/Hazelcast). Локальный кэш даёт минимальную задержку, но виден только одному инстансу и на него влияет GC/перезапуски. Распределённый кэш переживает рестарт сервиса, делится данными между репликами, но требует сети и несёт сетевые задержки. Часто оба комбинируют (L1/L2).

Важно помнить: кэш — это **не источник истины**. В нём могут лежать устаревшие значения, а логика инвалидации (очистки) добавляет сложность. Поэтому кэш не «всегда полезен», а только там, где его **стоимость** (сложность, память, инвалидация) оправдывается **выигрышем** в SLA/стоимости.

Кэш — это ещё и **политики**: когда удалять (TTL/TTI), как обновлять (refresh ahead), как защищаться от «налёта» запросов (singleflight), что делать при отказе кластера (degrade to origin). Разумно оформлять это в архитектурные правила и тестировать как обычную бизнес-логику.

Минимальный пример кэширования с Spring Cache и локальным Caffeine:

```java
// build.gradle.kts: implementation("org.springframework.boot:spring-boot-starter-cache")
//                   implementation("com.github.ben-manes.caffeine:caffeine")
@Configuration
@EnableCaching
class CacheConfig {
  @Bean
  public CacheManager cacheManager() {
    var caffeine = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(java.time.Duration.ofMinutes(10));
    return new org.springframework.cache.caffeine.CaffeineCacheManager("users","products"){{
      setCaffeine(caffeine);
    }};
  }
}

@Service
class UserService {
  private final UserRepository repo;
  UserService(UserRepository repo){ this.repo = repo; }

  @Cacheable(cacheNames = "users", key = "#id")
  public UserDto getUser(long id) {
    // дорогой поход в origin (БД/HTTP)
    return repo.findUserDtoById(id);
  }
}
```

## Зачем он нужен

Главная причина — **снижение латентности** и **нагрузки** на origin. Если мы можем вернуть результат из памяти за микросекунды/миллисекунды, вместо десятков/сотен мс похода в БД/HTTP, пользователю будет быстрее, а инфраструктуре — легче. Под пиками трафика кэш «сглаживает» нагрузку и помогает удерживать **SLA**.

Вторая причина — **стоимость**. Обращения к внешним API/облачным БД стоят денег. Сохранив «горячие» ответы, мы экономим. Особенно это заметно в BFF-слоях, которые агрегируют данные из нескольких медленных источников: кэш здесь уменьшает число «дорогих» обращений.

Третья — **стабильность**. Даже если origin «дрожит» (например, периодически отвечает с повышенной латентностью), кэш позволяет выдержать приемлемое качество ответа, а запросы к origin выполнять дозированно (refresh-ahead, rate limit). Это снижает вероятность каскадных отказов.

Четвёртая — **декомпозиция**. Иногда выгодно материализовать «сложные» вычисления (агрегации, отчёты) и отдавать готовые срезы; кэш превращает тяжёлые операции в дешёвые чтения, а обновление переносится на фон.

Пример: обёртка к сложной агрегации, которую кэшируем на 5 минут:

```java
@Cacheable(cacheNames = "report", key = "#range.from + ':' + #range.to")
public ReportDto aggregate(Period range) {
  return expensiveAggregation(range); // секунды CPU/несколько запросов к БД
}
```

## Где используется

Кэш повсеместен в **читаемых** сценариях: каталоги товаров, профили, справочники, конфигурации, разрешения. На уровне API — кэшируем **GET** (200/301/404) и используем edge/CDN для статических/полустатических ресурсов. В БД слое — кэшируем **lookup**-ы (по id/ключу), результаты сложных JOIN/агрегаций, если они меняются редко.

В микросервисах распределённый кэш помогает **делиться** объектами между репликами (например, токены сессий, лимиты, feature-флаги). Локальный (L1) — ускоряет повторные чтения на инстансе/потоке (например, под запросом). Пара «L1 + Redis» — классическая для Spring.

На фронтираx (API-шлюзы, CDN) кэшируют ответы, чтобы уменьшить hops до origin. Для приватных API пользовательский уровень кэширования важен меньше (нужна авторизация и vary-политики), но слой edge всё равно снимает часть нагрузки (stale-while-revalidate/stale-if-error).

Наконец, кэш полезен в **вычислительных** задачах: цена/рейтинг, подготовленные выборки, предварительно вычисленные результаты ML-моделей. Главное — корректно формализовать, **когда** значение считается устаревшим.

## Как задачи/проблемы решает

Кэш решает четыре класса проблем: **быстродействие** (меньше времени ответа), **нагрузка** (меньше запросов к origin), **стабильность** (меньше провалов по SLA под пиком/дрожью), **стоимость** (меньше дорогих операций). Но взамен приносит **инвалидацию** и риск **устаревших** данных — это надо осознанно принимать и проектировать.

Кэш помогает бороться с **штормами промахов** (cache miss stampede), когда множество запросов одновременно ломятся за одним и тем же ключом. Используются **singleflight** (одно загрузочное выполнение на ключ) и **jitter** для TTL, чтобы разгладить синхронное истечение. В Spring это `@Cacheable(sync = true)`/пер-ключевые локи.

Также кэш — отличный инструмент **дефенсивного дизайна**: при отказе Redis можно деградировать на origin, а при слишком больших payload — отказаться от кэширования специально (фильтры «кэшируем/не кэшируем»). Мы сознательно ограничиваем объёмы и живучесть, чтобы не навредить.

Шаблон edge-кеширования HTTP-ответов:

```java
@GetMapping("/public/catalog/{id}")
public ResponseEntity<CatalogItem> item(@PathVariable long id) {
  var dto = service.getItem(id); // под капотом Cacheable
  return ResponseEntity.ok()
      .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)).staleIfError(Duration.ofMinutes(10)))
      .eTag("\"%s\"".formatted(dto.version()))
      .body(dto);
}
```

---

# 1. Зачем кеш и где он окупается

## Цели: снижение латентности/нагрузки, стабилизация SLA, экономия

Первая цель — **ускорение**. Если медленный путь (БД/HTTP) заменяется обращением к памяти/Redis, сокращается p50/p95, пользователи замечают реактивность приложения. Особенно заметно при «толстых» агрегатах, которые строятся из нескольких источников.

Вторая — **снятие нагрузки** с origin. Базы и внешние API — дорогие ресурсы. Кэш переводит часть трафика в «дешёвые» обращения. Это помогает и при **пиковых** нагрузках: вы избегаете очередей в пулах соединений и contention в БД.

Третья — **стабилизация SLA**. «Гладкие» хвосты задержек за счёт выдачи ответов из кэша и ограничение запросов к зависимостям (refresh с rate limit) дают лучшую предсказуемость. Даже если origin в этот момент болит, пользователи меньше страдают.

Четвёртая — **экономика**. Вы уменьшаете «покупку» RPS у внешних поставщиков (billable API), меньше апгрейдите БД, сокращаете compute. В пересчёте на деньги кеш часто окупается быстро, особенно в read-heavy системах.

Пример простого эксперимента A/B: включаем кэш на частичном трафике и сравниваем метрики:

```java
@Cacheable(cacheNames = "prices", key = "#sku", condition = "#root.target.flags.ab('cache')") 
public Price price(String sku) { return pricingClient.get(sku); }
// где flags.ab(name) — ваша фича-система A/B
```

## Типы нагрузок: read-heavy, write-heavy, mixed; горячие/тёплые/холодные ключи

Кэш особенно эффективен в **read-heavy** сценариях: много чтений на одно изменение. «Горячие» ключи читают часто и их выгодно держать рядом как можно дольше. «Тёплые» — периодически. «Холодные» — редкие, для них кэш «не прогреется» и будет только усложнять.

В **write-heavy** системах (частые изменения) кэш окупается хуже из-за высокой стоимости инвалидации. Но и здесь есть островки эффективности: справочники, метаданные, большие неизменяемые части. В **mixed**-нагрузке кэш лучше применять точечно, покрывая именно «горячие» запросы.

Понимание «теплоты» ключей приходит из метрик: top-N, частота попаданий/промахов, размеры payload. Слепое кэширование «всего подряд» почти всегда заканчивается сложностью без выгод.

Полезно классифицировать ключи и держать **разные TTL**: горячим — больше, холодным — меньше или вовсе не кэшировать.

Снятие профиля горячих ключей (Micrometer + Redis INFO/slowlog) и промахов:

```java
// пример счетчика промахов
@Component
class CacheMissCounter implements CacheErrorHandler {
  private final Counter misses = Metrics.counter("cache.miss", "name", "users");
  @Override public void handleCacheMissError(RuntimeException e, Cache cache, Object key) { misses.increment(); }
  // остальные методы no-op
}
```

## Классы кешей: локальный (in-proc), распределённый (Redis/Hazelcast), edge/CDN; иерархии L1/L2

**Локальный** (in-proc) кэш — самый быстрый: нет сети, только память JVM. Идеален для «подзапросных» повторных чтений, констант и мелких справочников. Минусы: не разделяется между инстансами, не переживает рестарт, может усиливать несогласованность.

**Распределённый** (Redis/Hazelcast) — общий для всех реплик и живёт вне процесса. Позволяет делиться «прогревом», управлять объёмом, получать pub/sub для инвалидаций. Минусы: сеть и отдельная инфраструктура.

**Edge/CDN** — фронтовое кэширование HTTP-ответов (CloudFront/Fastly/Nginx). Прекрасно для публичных GET, статики, медиа. Требует аккуратных заголовков (`Cache-Control`, `ETag`, `Vary`) и понимания приватности.

Комбинация **L1/L2** — частая практика: локальный Caffeine как near-cache над Redis. Чтение — сначала L1, затем L2, потом origin; инвалидации — через pub/sub, чтобы сбросить L1 у всех инстансов.

Конфигурация L1+L2 в Spring (упрощённо):

```java
@Configuration
@EnableCaching
class CacheCfg {

  @Bean CacheManager redisManager(RedisConnectionFactory f) {
    var serializer = new org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer();
    var config = org.springframework.data.redis.cache.RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .serializeValuesWith(org.springframework.data.redis.serializer.RedisSerializationContext.SerializationPair.fromSerializer(serializer))
        .prefixCacheNameWith("app:");
    return org.springframework.data.redis.cache.RedisCacheManager.builder(f).cacheDefaults(config).build();
  }

  @Bean CacheManager caffeineManager() {
    var c = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
        .maximumSize(20_000).expireAfterWrite(Duration.ofMinutes(2));
    return new org.springframework.cache.caffeine.CaffeineCacheManager(){{
      setCaffeine(c);
    }};
  }

  // CompositeCacheManager: сначала Caffeine (L1), потом Redis (L2)
  @Bean CacheManager composite(CacheManager caffeine, CacheManager redis) {
    var cm = new org.springframework.cache.support.CompositeCacheManager(caffeine, redis);
    cm.setFallbackToNoOpCache(false);
    return cm;
  }
}
```

## Ограничения и риски: устаревшие данные, poisoning, сложность инвалидации

Главный риск — **устаревшие данные**. Если бизнес не готов к «slightly stale», то каждое несоответствие будет инцидентом. Важно договориться о **бюджете устаревания** (сколько секунд/минут допустимо), и только потом выбирать TTL/политики.

**Cache poisoning** — загрязнение кэша неверными/вредоносными данными (ошибка в origin, инъекция через параметры). Рецепт — валидация перед кэшированием, лимиты размеров/TTL, отказ от кэширования небезопасных ответов (например, если не прошли схемную валидацию).

Сложность **инвалидации** — основная цена кэша. Чем больше «связей» между ключами, тем тяжелее сбрасывать корректно (тэги/иерархии/суррогатные ключи помогают). Нельзя «костылить» руками — должны быть процессы/скрипты/эндпоинты инвалидаций и тесты.

Ещё риск — **шторм промахов** (stampede): синхронное истечение ключей вызывает «налёт» на origin. Это лечится `sync=true` (singleflight), `jitter` для TTL, refresh-ahead и пер-ключевыми локами.

Пример: отказ кэшировать слишком большие payload (эвристика):

```java
@Cacheable(cacheNames = "profiles", key = "#id", unless = "#result == null || #result.estimatedSize() > 100_000")
public Profile get(long id) { return repo.fetchProfile(id); }
```

---

# 2. Модели согласованности и паттерны

## Cache-Aside (загрузку/инвалидацию контролирует заказчик)

**Cache-Aside** — дефолт для бизнес-сервисов. Приложение сначала ищет ключ в кэше; при промахе — грузит из origin, кладёт в кэш и возвращает. При изменении данных приложение **само** сбрасывает/обновляет кэш. Плюс — простота и контроль, минус — легко забыть инвалидацию.

В Spring Cache это делается через `@Cacheable` для чтений и `@CacheEvict/@CachePut` для записей. Важно понимать порядок: сначала коммит изменений в origin, затем инвалидация/обновление кэша — чтобы не раздать устаревшее. Используйте транзакционные события «после коммита».

Также полезно включать `sync=true` в `@Cacheable`, чтобы при промахе **один** поток выполнял загрузку, а остальные ждали результат — защита от stampede. Для очень горячих ключей можно добавить refresh-ahead.

Пример Cache-Aside с событиями «после коммита»:

```java
@Service
class ProductService {
  private final ProductRepository repo;
  private final ApplicationEventPublisher events;
  ProductService(ProductRepository repo, ApplicationEventPublisher events){ this.repo=repo; this.events=events; }

  @Cacheable(cacheNames = "product", key = "#id", sync = true)
  public ProductDto get(long id) { return repo.findDto(id); }

  @Transactional
  public void update(long id, UpdateProduct cmd) {
    repo.update(id, cmd);
    events.publishEvent(new ProductChanged(id)); // AFTER_COMMIT listener → evict
  }
}

@Component
class ProductCacheInvalidation {
  @Autowired CacheManager cm;

  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void on(ProductChanged e) {
    cm.getCache("product").evict(e.id());
  }
}
```

## Read-Through/Write-Through/Write-Behind

**Read-Through**: приложение обращается к кэшу как к «хранилищу», а тот сам идёт в origin при промахе. Удобно, когда кэш-слой стандартизован (например, Hazelcast MapStore). **Write-Through**: запись идёт через кэш в origin синхронно. **Write-Behind**: запись буферизуется и позже сбрасывается пачками — это ускоряет запись, но усложняет гарантии.

В Spring Cache абстракции «чистого» read/write-through нет из коробки, но часто достаточно `@Cacheable` + `@CachePut`: `@CachePut` принудительно обновляет кеш **результатом** метода (например, после изменения). Для write-behind потребуется кастомный слой (очередь/батчи) и учёт сбоев/повторов.

Выбирать эти паттерны стоит, когда действительно нужен **единый слой доступа**: например, когда кэш расположен «рядом» с данными и логически является их фасадом. Для обычного микросервиса с БД в 90% случаев хватает Cache-Aside.

Пример обновления (write-through-подобный) через `@CachePut`:

```java
@CachePut(cacheNames = "product", key = "#id")
@Transactional
public ProductDto change(long id, UpdateProduct cmd) {
  repo.update(id, cmd);
  return repo.findDto(id); // вернём свежее значение — попадёт в кэш
}
```

## Refresh-Ahead/Background Refresh, Soft/Hard TTL, jitter

**Refresh-Ahead** — фоновые обновления ключа до истечения TTL. Пользователи получают быстрый ответ из кэша, а «параллельно» ключ обновляется. Хорошо работает с **Soft TTL**: до soft-порога значение «свежее», после — «условно свежее» (можно отдавать, но запускаем refresh), и есть **Hard TTL**, после которого ключ считается непригодным.

Чтобы избежать «синхронной смерти ключей», на TTL добавляют **jitter** (случайный разброс ±x%), чтобы тысячи ключей не истекали в одну секунду. Caffeine поддерживает `refreshAfterWrite`, Redis — требует фоновой логики (например, @Scheduled).

В Spring Cache можно реализовать refresh-ahead через планировщик, который дергает `@CachePut` для «горячих» ключей, или использовать Caffeine с `refreshAfterWrite` и `CacheLoader`.

Пример Caffeine с `refreshAfterWrite`:

```java
@Bean
public CacheManager caffeineWithRefresh() {
  var loader = new com.github.benmanes.caffeine.cache.CacheLoader<Long, ProductDto>() {
    @Override public ProductDto load(Long id) { return repo().findDto(id); }
    @Override public ProductDto reload(Long id, ProductDto oldValue) { return repo().findDto(id); }
  };
  var cache = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
      .maximumSize(20_000)
      .expireAfterWrite(Duration.ofMinutes(30))
      .refreshAfterWrite(Duration.ofMinutes(5)) // refresh-ahead
      .build(loader);
  return new org.springframework.cache.caffeine.CaffeineCacheManager(){{
    setCaffeine(cache.policy().eviction().map(e -> com.github.benmanes.caffeine.cache.Caffeine.newBuilder()).orElseThrow()); // трюк не обязателен
  }};
}
```

## Singleflight/Coalescing: защита от stampede

**Singleflight** (a.k.a. coalescing) — стратегия, когда множество конкурентных запросов к **одному** ключу не исполняют N одинаковых загрузок, а «подписываются» на **одну**. В Spring `@Cacheable(sync = true)` делает именно это: один поток выполняет `load`, остальные ждут результат и кладут в кэш.

Для L2 (Redis) можно сделать пер-ключевую блокировку (Redisson `RLock`) перед загрузкой, чтобы координировать даже между инстансами. Это снижает шторм, когда горячий ключ истёк одновременно у всех.

Важно не забывать про **таймауты**: если загрузка зависнет, блоки тоже зависнут. Не держите глобальные логи на доли секунд, будьте готовы к деградации (вернуть «старое» с пометкой, если refresh не успел).

Пример защиты sync=true:

```java
@Cacheable(cacheNames = "fx", key = "#pair", sync = true)
public Rate getRate(String pair) { return fxClient.get(pair); }
```

---

# 3. Проектирование ключей и полезной нагрузки

## Схема ключей: namespace:версия:тенант:локаль:фильтры:идентификатор

Ключ должен быть **уникальным и стабильным**. Рекомендуемая форма:
`<ns>:<v>:<tenant>:<locale>:<filters>:<id>` — где namespace отличает систему/сервис, версия — контракт (чтобы не ломать кэш при изменениях схем), tenant/locale сегментируют мультиарендность и локализацию, filters — важные query-параметры, id — первичный идентификатор.

Жёстко определяйте **что входит** в ключ. Если ответ зависит от `Accept-Language`, `X-Role`, флагов AB-тестов — эти параметры должны участвовать в ключе или ответ не должен кэшироваться. Иначе получите «перепутанные» ответы.

Версионирование обязательно: при изменении DTO/логики поднимайте `v` (например, `v2`) и очищайте старый регион. Это дешёвый способ не ловить «призраков» после релиза.

Для Spring Cache можно написать **KeyGenerator**, чтобы централизовать правило построения:

```java
@Configuration
class Keys {
  @Bean
  public KeyGenerator namespacedKey() {
    return (target, method, params) -> {
      var tenant = TenantContext.get();
      var locale = LocaleContextHolder.getLocale().toLanguageTag();
      return "app:v2:%s:%s:%s:%s".formatted(tenant, method.getName(), Arrays.deepToString(params), locale);
    };
  }
}

@Cacheable(cacheNames = "product", keyGenerator = "namespacedKey")
public ProductDto get(long id, Filter f) { ... }
```

## Мульти-тенант/мульти-регион/AB-тесты: сегментация

В multi-tenant приложениях **обязательно** включайте `tenantId` в ключ. Для мульти-региона — регион/шард, если данные различаются. Для AB-тестов — флаг варианта (A/B), если макет ответа разный. Это предотвращает утечку данных между сегментами.

Сегментация должна быть **доступной** из контекста: чтобы KeyGenerator мог её достать. Удобно хранить tenant/variant в `ThreadLocal`/Reactor Context и прокидывать через фильтры.

Не перегружайте ключ «всем подряд»: включайте только те параметры, от которых **реально** зависит ответ. Иначе будет взрыв кардинальности и низкая эффективность кэша.

Пример добавления tenant/variant в ключ (см. выше) и в HTTP-кеш (Vary):

```java
return ResponseEntity.ok()
    .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)))
    .varyBy("Accept-Language","X-Tenant","X-Variant")
    .body(dto);
```

## Форматы значений и сериализация: JSON/CBOR/Smile/ProtoBuf

Для L2 выбирайте сериализацию, совместимую между языками/версиями. **JSON** — универсален, но шумный и тяжёлый; **CBOR/Smile** — бинарные варианты JSON быстрее/компактнее; **ProtoBuf/Avro** — ещё компактнее, но сложнее эволюция. Избегайте **Java-сериализации** (небезопасна и хрупка).

В Spring Data Redis удобно использовать `GenericJackson2JsonRedisSerializer` и стабильные DTO (без полей «сюрпризов»). При больших payload добавляйте **компрессию** (GZIP/LZ4) — но не переусердствуйте, это CPU. Взвешивайте: где дешевле — байты в сети или CPU на компрессор.

Следите за **эволюцией** схем: добавляйте поля с дефолтами, не переименовывайте без версии, используйте строго типизированные поля для денег/дат (BigDecimal как строка с scale/ISO-8601 Instant).

RedisCacheManager с кастомной сериализацией:

```java
@Bean
RedisCacheManager redisCacheManager(RedisConnectionFactory f, ObjectMapper om) {
  var ser = new org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer(om);
  var cfg = org.springframework.data.redis.cache.RedisCacheConfiguration.defaultCacheConfig()
      .serializeValuesWith(org.springframework.data.redis.serializer.RedisSerializationContext.SerializationPair.fromSerializer(ser))
      .entryTtl(Duration.ofMinutes(15))
      .disableCachingNullValues();
  return org.springframework.data.redis.cache.RedisCacheManager.builder(f).cacheDefaults(cfg).build();
}
```

## Компрессия, лимиты размера, эвристики «кэшируем/не кэшируем»

Большие payload (сотни КБ/МБ) не всегда стоит кэшировать: они «засоряют» память, давят на сеть и GC. Введите **лимиты**: «если > X байт — не кэшировать» или кэшировать **укороченный срез** (например, только summary). Для Redis можно хранить **сжатые** байты, оборачивая сериализатор.

Полезно иметь **функцию-решатель**: кэшировать ли этот ответ? Учитывайте тип запроса, размер, важность, «горячесть» ключа. Возвращайте флаг и TTL. Это дисциплинирует — кэш становится осмысленным.

Не кэшируйте PII/секреты без шифрования. Часто проще **вообще не** кэшировать персональные ответы (или кэшировать только «публичные» части).

Пример сжатого сериализатора для Redis:

```java
class GzipSerializer implements RedisSerializer<Object> {
  private final RedisSerializer<Object> delegate = new GenericJackson2JsonRedisSerializer();
  @Override public byte[] serialize(Object o) throws SerializationException {
    var raw = delegate.serialize(o);
    try (var baos = new ByteArrayOutputStream(); var gzip = new GZIPOutputStream(baos)) {
      gzip.write(raw); gzip.finish(); return baos.toByteArray();
    } catch (IOException e) { throw new SerializationException("gzip", e); }
  }
  @Override public Object deserialize(byte[] bytes) throws SerializationException {
    try (var gis = new GZIPInputStream(new ByteArrayInputStream(bytes))) {
      return delegate.deserialize(gis.readAllBytes());
    } catch (IOException e) { throw new SerializationException("gunzip", e); }
  }
}
```

## Версионирование контрактов и «срезы» DTO; защита от несоответствия схем

Меняя DTO/логику, увеличивайте **версию** префикса ключа, чтобы старые данные не подсовывались новому коду. Для больших объектов храните **срезы** (summary), если 90% запросов не требуют полных данных — это экономит объём и ускоряет сериализацию.

Защищайтесь от ошибок декодирования: конфигурируйте `ObjectMapper` на «мягкую» эволюцию (ignore unknown), валидируйте типы/поля перед кэшированием (Bean Validation/JSON Schema), а при ошибке — **не кэшируйте** значение.

Проводите **миграции**: если изменилась структура, прогрейте кэш заново (batch-проход) и только потом включайте чтение. Либо держите code-path, который умеет работать со старой версией «на чтение».

Пример игнора неизвестных полей:

```java
@Bean ObjectMapper objectMapper() {
  return new ObjectMapper()
      .registerModule(new JavaTimeModule())
      .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
}
```

## Политика конфиденциальности: PII/секреты

Не кэшируйте персональные данные (телефоны, e-mail, адреса) **в общих кэшах**. Если очень нужно — шифруйте значения (KMS/Vault), ограничивайте TTL, используйте приватные пространства/ключи с минимальными правами и аудитом доступа.

Для публичных API применяйте edge-кэш строго на «public, max-age», а приватные — помечайте `Cache-Control: private, no-store` или держите отдельный CDN-слой с **подписанными URL/cookie** и сегрегацией контента по пользователю.

Проверяйте, не утекают ли секреты в логи/метрики кэша (ключи/значения). Маскируйте части ключей (например, не включайте e-mail в ключ без хеша) и обрезайте большие значения при логировании.

Мини-пример хеша в ключе:

```java
String key = "user:byEmail:" + DigestUtils.sha256Hex(email.toLowerCase(Locale.ROOT).trim());
```
отлично — продолжаем «Кэширование» и подробно раскрываем **подтемы 4–7** (инвалидация/обновление, локальные кэши L1, распределённые кэши L2, Spring Cache Abstraction). Для каждого подпункта — 4+ абзацев и рабочие фрагменты кода (Java/Kotlin, Spring Boot 3.x).

---

# 4. Инвалидация и обновление

## Точечная, пакетная, теговая/иерархическая инвалидация (surrogate keys)

Инвалидация — это контроль «жизненного цикла» кэшированных значений. **Точечная** (point eviction) удаляет конкретный ключ (например, `product:42`) при изменении ресурса. Она самая дешёвая и предсказуемая: вы знаете, какой ключ «грязный», сбрасываете его и всё. Проблема начинается, когда один ресурс участвует во множестве представлений (деталька, список, топ-10) — тогда точечная инвалидация перестаёт хватать.

**Пакетная** инвалидация — снятие целого набора ключей, чаще «в пределах кэша/префикса». В Redis это опасно выполнять через `SCAN`/`DEL` на проде (дорого и непредсказуемо). Правильнее заранее **учитывать членство** ключей в наборах (см. теговую инвалидацию) или хранить префикс-версии (bump версии → старые ключи считаются «устаревшими» без физического удаления).

**Теговая/иерархическая** инвалидация (surrogate keys) — подход из мира CDN: каждый кэшируемый объект привязывается к одному/нескольким «тегам» (например, `product:42`, `category:7`). Удаляя тег, вы инвалидируете все связные ключи. В Redis реализуется через наборы: `SADD tag:product:42 key1 key2`, затем `SMEMBERS` → `DEL`. Важно контролировать размер наборов и выполнять операции пайплайном/Lua.

Мини-пример surrogate keys на Redis (Kotlin + Lettuce):

```kotlin
fun cacheWithTags(redis: StringRedisTemplate, key: String, tags: List<String>, value: String, ttl: Duration) {
    redis.executePipelined { ops ->
        ops.opsForValue().set(key, value, ttl)
        tags.forEach { tag -> ops.opsForSet().add("tag:$tag", key) }
        null
    }
}

fun invalidateTag(redis: StringRedisTemplate, tag: String) {
    val keys = redis.opsForSet().members("tag:$tag").orEmpty()
    if (keys.isNotEmpty()) {
        redis.executePipelined { ops ->
            keys.forEach { k -> ops.delete(k) }
            ops.delete("tag:$tag")
            null
        }
    }
}
```

## TTL/TTI, max-age, stale-while-revalidate/stale-if-error

**TTL** (time-to-live) — жёсткое «протухание»: по истечении времени значение больше недействительно. **TTI** (time-to-idle) — протухание по неиспользованию: если не обращались N минут, ключ удаляется, но каждое обращение «освежает» время. В локальных кэшах (Caffeine) TTI часто полезнее TTL для «хвостов» ключей.

На уровне HTTP распространены **Cache-Control** директивы: `max-age` (аналог TTL на клиент/edge), `stale-while-revalidate` (разрешает отдавать устаревшее N секунд, пока в фоне обновляется) и `stale-if-error` (в случае ошибки origin допускает устаревшее ответить ещё N секунд). Эти же идеи можно встроить и в свой бизнес-кэш: держать «soft TTL» и «hard TTL» и в периоды между ними делать refresh на фоне.

Важно, чтобы TTL были **обоснованы** бизнесом: «насколько можно устаревать?». Слепое «5 минут» без SLA приводит к сюрпризам. Для «горячих» ключей TTL можно длиннее, если есть refresh-ahead; для «чувствительных» — короче и с событийной инвалидацией.

Пример HTTP-ответа с подсказками edge-кэшу:

```java
return ResponseEntity.ok()
  .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5))
    .staleWhileRevalidate(Duration.ofMinutes(2))
    .staleIfError(Duration.ofMinutes(10)))
  .eTag("\"%s\"".formatted(dto.version()))
  .body(dto);
```

## Борьба со штормом: пер-ключовые локи, lease token, ограничители частоты

«Шторм промахов» (cache stampede) возникает, когда многие потоки одновременно видят истечение и стартуют одинаковые загрузки в origin. Первый барьер — **singleflight** (`@Cacheable(sync = true)`), но это касается только одного инстанса. В распределённой среде используйте **пер-ключевые локи** (Redisson `RLock`) или **lease token**: краткоживущая «аренда» на загрузку.

**Lease**-подход: поток, который первым получил «аренду» на ключ, загружает и обновляет кэш; остальные либо ждут, либо получают stale значение, либо быстро возвращают 503/partial. Токен хранится в Redis с коротким TTL (например, 2–5 сек). Это дешевле, чем держать долгие блокировки, и хорошо переживает падения воркера.

Не забывайте **rate limiting** на стороне origin: даже при хорошем singleflight на горячих ключах refresh может «долбить» внешний сервис — ограничивайте параллельность/частоту фона.

Пример lease на Redis (Lua для атомарности):

```java
// acquire lease if not exists
String script = """
if (redis.call('SETNX', KEYS[1], ARGV[1]) == 1) then
  redis.call('PEXPIRE', KEYS[1], ARGV[2])
  return 1
else return 0 end
""";
Boolean acquired = stringRedisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class),
    List.of("lease:product:42"), UUID.randomUUID().toString(), String.valueOf(3000));
```

## Детект «ядовитых» ключей, карантин, ограничение объёма промахов; pub/sub

«Ядовитые» ключи — те, чья загрузка **почти всегда падает**/выполняется слишком долго/генерирует огромные payload. Их нужно **обнаруживать**: вести счётчики промахов, длительность загрузок, количество ошибок подряд. При достижении порогов — отправлять ключ в **карантин** (не кэшировать/меньше TTL/возвращать дефолт).

Ограничивайте **объём промахов** на окно времени (per-key rate limit), чтобы один проблемный ключ не положил origin. Если ключ «сломан», лучше быстро ответить «нет данных сейчас» (или выдать stale), чем запускать сотни тяжёлых загрузок.

В кластере не забудьте **распространять инвалидации** (pub/sub): при изменении ресурса инстанс-писатель публикует событие, подписчики сбрасывают свои L1. В Redis это `RedisMessageListenerContainer`; в Kafka — отдельная тема инвалидаций.

Мини-пример pub/sub для L1:

```kotlin
@Configuration
class InvalidationPubSub(
    private val cacheManager: CacheManager,
    connectionFactory: RedisConnectionFactory
) {
    @Bean
    fun container(): RedisMessageListenerContainer =
        RedisMessageListenerContainer().apply {
            setConnectionFactory(connectionFactory)
            addMessageListener(MessageListener { _, message ->
                val gameId = UUID.fromString(String(message.body))
                cacheManager.getCache("activeSlides")?.evict(gameId)
            }, ChannelTopic("invalidate:activeSlides"))
        }
}

// где-то в коде при изменении
stringRedisTemplate.convertAndSend("invalidate:activeSlides", gameId.toString())
```

---

# 5. Локальные кэши (L1)

## Caffeine: TinyLFU, размер vs вес, expireAfterWrite/Access, refreshAfterWrite

**Caffeine** — быстрый локальный кэш для JVM со state-of-the-art политиками. По умолчанию использует **TinyLFU** (частотный фильтр) + **W-TinyLFU** (windowed) для балансировки между recency и frequency: «горячие» ключи удерживаются лучше, «единичные» не засоряют кэш.

Вы управляете **размером** как количеством элементов (`maximumSize`) или **весом** (`maximumWeight` + `weigher`), что полезно при неравномерных payload’ах. Для «устаревания» есть `expireAfterWrite` (TTL) и `expireAfterAccess` (TTI). Для фонового обновления — `refreshAfterWrite`, который вызывает `CacheLoader.reload`.

Комбинируйте: для «быстрых» справочников достаточно `expireAfterWrite`, для реактивных данных удобно `refreshAfterWrite` с дешёвой перезагрузкой, чтобы пользователи редко видели промахи.

Пример Caffeine с различными политиками (Java):

```java
@Bean
Cache<String, ProductDto> productCache() {
  return Caffeine.newBuilder()
      .recordStats()
      .maximumWeight(100 * 1024 * 1024) // 100 MB
      .weigher((String k, ProductDto v) -> v.estimatedSize())
      .expireAfterAccess(Duration.ofMinutes(10))
      .refreshAfterWrite(Duration.ofMinutes(2))
      .build(key -> repo().findDto(Long.parseLong(key)));
}
```

## Weak/Soft refs, преднагрев (warm-up)

Caffeine поддерживает **weakKeys/weakValues/softValues**, но использовать это нужно осторожно. Weak/soft-ссылки отдают управление GC: под давлением памяти значения «исчезают». Это может быть полезно для «второстепенных» кэшей, но непредсказуемо для SLA. В большинстве production-кейсов лучше **жёсткие** ссылки и чёткие лимиты (size/weight).

**Преднагрев** (warm-up) ускоряет «холодный старт». Сервис при запуске может прочитать «горячие» ключи (списки категорий, конфиг) и положить их в L1, чтобы первые запросы не упали в origin. Делайте это **асинхронно**, чтобы не удлинять старт, и с лимитами RPS к origin.

Полезно измерять «уровень прогрева»: метрика `cache.warm.percent` — доля ключей, которые доступны в L1 сразу после старта. Это помогает оценить эффект warm-up’а.

Пример преднагрева:

```kotlin
@Component
class WarmUpRunner(
    private val svc: ProductService,
    @Qualifier("productCache") private val cache: com.github.benmanes.caffeine.cache.Cache<String, ProductDto>
) : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        val hotIds = svc.topProductIds(limit = 100)
        hotIds.forEach { id -> cache.put(id.toString(), svc.getProductDto(id)) }
    }
}
```

## Near-cache над Redis: чтение из L1, запись/invalid через L2

Типичный паттерн — **near-cache**: чтения сначала попадают в L1 (Caffeine), при промахе — в L2 (Redis), затем в origin. Инвалидация всегда идёт через L2 (pub/sub), чтобы синхронизировать все инстансы. Это даёт скорость локального кэша и согласованность распределённого.

В Spring можно построить `CompositeCacheManager`: сначала Caffeine, потом Redis. Но у такого подхода есть нюанс: `@Cacheable` обращается к **первому** `Cache` c данным именем. Чтобы near-cache работал как задумано, либо используйте библиотеку (Redisson local cache), либо сделайте **обёртку Cache**, которая читает-из-L1/пишет-в-оба и слушает pub/sub.

Если хотите быстро — используйте **Redisson** с локальными кэшами (`RLocalCachedMap`): он делает near-cache под капотом и размечает инвалидации через топики.

Пример Redisson local cache:

```java
RLocalCachedMap<String, ProductDto> map = redisson.getLocalCachedMap("product",
    LocalCachedMapOptions.<String, ProductDto>defaults()
      .syncStrategy(LocalCachedMapOptions.SyncStrategy.UPDATE)
      .reconnectionStrategy(LocalCachedMapOptions.ReconnectionStrategy.CLEAR)
      .timeToLive(600_000)
      .maxIdle(120_000)
      .cacheSize(20_000));
```

## JMH-замеры и профилирование попаданий/промахов

Не верьте «на глаз» — **профилируйте**. Снимайте `cache.stats()` (hit/miss/eviction), строьте дистрибуции задержек (Micrometer Timers вокруг «loaders»), измеряйте «время в очереди» при stampede. Для микробенчмарков используйте **JMH**: сравните чтения из L1 и походы в Redis/БД, посмотрите, где «утекает» CPU.

Удобный приём — инжектировать «наблюдаемый» `CacheLoader`: он считает время `load`/`reload` и отдаёт метрики с тегами per-key-class (например, `product`, `category`). На основе метрик легко найти «ядовитые» ключи и устранить узкие места.

Шаблон JMH (Kotlin/Gradle) — skeleton:

```java
@State(Scope.Benchmark)
public class CaffeineBench {
  Cache<Integer, byte[]> cache;
  @Setup
  public void setup() {
    cache = Caffeine.newBuilder().maximumSize(100_000).build(k -> new byte[1024]);
  }

  @Benchmark public byte[] hit() { return cache.get(42); }
  @Benchmark public byte[] miss() { return cache.get(ThreadLocalRandom.current().nextInt()); }
}
```

---

# 6. Распределённые кэши (L2)

## Redis: кластеры/реплика, Sentinel, RDB/AOF, пулы и таймауты

**Redis** — самый популярный L2. Для продакшена выбирайте **кластер** (шардирование) или мастера с **репликами** и Sentinel (фейловер). У каждого режима свой компромисс по доступности/сложности. В managed-облаках (AWS ElastiCache, GCP Memorystore) часть проблем снимается провайдером.

Снимите настройки **подключений** и **таймаутов**. В Lettuce/Netty важно задать `commandTimeout`, `shutdownTimeout`, `pool` (maxTotal, maxIdle, minIdle), `soKeepalive`, и разумный `connectTimeout`. Под пиками именно пул/таймауты спасают от лавинообразных зависаний.

Снимите решение по **персистентности**: RDB (снимки) дешевле по CPU, но потеря последних секунд возможна; AOF (журнал операций) надёжнее, но дороже по диску/CPU. В кэше допустим **без персистентности** (если L2 — только кеш); для «квазихранилища» — настройте AOF.

Конфиг Lettuce в Spring (Java):

```java
@Bean
public LettuceConnectionFactory redisConnectionFactory() {
  var cfg = new RedisStandaloneConfiguration("redis", 6379);
  var clientCfg = LettuceClientConfiguration.builder()
      .commandTimeout(Duration.ofMillis(500))
      .shutdownTimeout(Duration.ofMillis(200))
      .clientOptions(ClientOptions.builder()
          .autoReconnect(true)
          .build())
      .poolConfig(GenericObjectPoolConfig.builder()
          .maxTotal(64).maxIdle(32).minIdle(8).build())
      .build();
  return new LettuceConnectionFactory(cfg, clientCfg);
}
```

## Структуры данных: string/hash/set/zset, bitmap/bloom

Правильный выбор структур Redis экономит память и ускоряет операции. **String** — базовый тип для сериализованных объектов. **Hash** — хранение «полей» объекта отдельными слотами (частичные обновления/чтения), экономит память на мелких значениях. **Set/Sorted Set** — для связей (теги, топы). **Bitmap/Bloom** — экономичные фильтры/множества для дедупликации/проверок «видели ли id».

Если вы часто обновляете **часть** объекта (одно поле) — выгодно разбить на Hash. Но не переусердствуйте: сложнее управление TTL и версионирование. Для surrogate keys `Set` — идеален; для топов (рейтинги) — `ZSET`.

Пример Hash:

```java
// HSET product:42 name "Phone" price "699.00" ver "v2"
redisTemplate.opsForHash().put("product:42", "name", dto.getName());
redisTemplate.opsForHash().put("product:42", "price", dto.getPrice().toPlainString());
redisTemplate.expire("product:42", Duration.ofMinutes(10));
```

## Атомарность операций (Lua/EVAL), Redisson-локи/семафоры/мапы

Иногда нужно выполнить несколько операций **атомарно**: проверить версию, обновить значение, инкрементить счётчик — всё одним шагом. В Redis это делается через **Lua** (`EVAL`). Скрипт выполняется как одна команда, что гарантирует консистентность без внешних блокировок.

**Redisson** добавляет распределённые примитивы: `RLock` (лок), `RSemaphore` (семафор), `RMap` (распределённые карты) с автопролонгацией. Аккуратно с долгими блокировками; предпочитайте **lease + короткий TTL**, где возможно.

Пример Lua CAS:

```java
String lua = """
local key = KEYS[1]
local expected = ARGV[1]
local next = ARGV[2]
local cur = redis.call('GET', key)
if cur == expected then
  redis.call('SET', key, next, 'EX', ARGV[3])
  return 1
else
  return 0
end
""";
Boolean ok = stringRedisTemplate.execute(new DefaultRedisScript<>(lua, Boolean.class),
  List.of("product:42:ver"), "v2", "v3", "600");
```

## Hazelcast/Coherence/Infinispan; стратегии деградации

Не всегда Redis — ответ. **Hazelcast/Infinispan/Coherence** предлагают **in-memory data grid**: распределённые структуры, **near-cache** из коробки, вычисления «на узлах» (entry processor), SQL/MapReduce. Это полезно при высокой взаимосвязи данных и требовании к **распределённым вычислениям** прямо «на кластере кэша».

Стратегии деградации обязательны: если L2 недоступен — **не падать** бизнес-логикой. Делайте «read-through на origin» и временно отключайте запись в кэш (feature-flag). Ставьте **circuit breaker** вокруг кэша (да, кэш — тоже зависимость) и сдерживайте штормы.

Метрика «доля ответов из кэша» (hit ratio) — не самоцель. Важнее **p95/p99** и количество **обращений к origin** при отказе L2 — чтобы понять, выдержит ли он без кэша.

---

# 7. Spring Cache Abstraction (Boot)

## Аннотации: `@Cacheable/@CachePut/@CacheEvict/@Caching`; `cacheManager/cacheResolver/keyGenerator`

Spring Cache даёт декларативный API поверх разных провайдеров (Caffeine/Redis/Hazelcast).

* `@Cacheable` — чтение с кэшированием; при промахе вычисляет и кладёт.
* `@CachePut` — всегда исполняет метод и **обновляет** кэш свежим значением.
* `@CacheEvict` — удаляет ключ/весь кэш.
* `@Caching` — композиция нескольких аннотаций (например, put + evict из списков).

Решатель кэша — либо `cacheManager` (имя бина менеджера), либо `cacheResolver` (вычисляет набор кэшей по контексту), ключ — через `key` (SpEL) или `keyGenerator`. Для сложных ключей лучше общий `KeyGenerator`, чтобы не дублировать шаблон.

Быстрая иллюстрация:

```kotlin
@Service
class ProductService(
    private val repo: ProductRepository
) {

    @Cacheable(cacheNames = ["product"], key = "#id", sync = true, unless = "#result == null")
    fun get(id: Long): ProductDto? = repo.findDto(id)

    @Caching(
        put = [CachePut(cacheNames = ["product"], key = "#id")],
        evict = [CacheEvict(cacheNames = ["productList"], allEntries = true)]
    )
    @Transactional
    fun update(id: Long, cmd: UpdateProduct): ProductDto {
        repo.update(id, cmd)
        return repo.findDto(id)!!
    }

    @CacheEvict(cacheNames = ["product"], key = "#id")
    fun delete(id: Long) { repo.delete(id) }
}
```

## `CaffeineCacheManager/RedisCacheManager`: пер-кешовые TTL, префикс, сериализация

В Boot 3 стоит настраивать `CacheManager` программно для гибкости: **пер-кешовые TTL**, префиксы, сериализаторы. Для Redis — `RedisCacheManager` с `RedisCacheConfiguration` на каждый кэш; для Caffeine — `CaffeineCacheManager` с разными билдами при необходимости.

Пример конфигурации с разными TTL (Java):

```java
@Configuration
@EnableCaching
public class CacheConfig {

  @Bean
  public RedisCacheManager redisCacheManager(RedisConnectionFactory f, ObjectMapper om) {
    var base = RedisCacheConfiguration.defaultCacheConfig()
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer(om)))
        .prefixCacheNameWith("app:");

    Map<String, RedisCacheConfiguration> perCache = new HashMap<>();
    perCache.put("product", base.entryTtl(Duration.ofMinutes(15)));
    perCache.put("productList", base.entryTtl(Duration.ofMinutes(2)));

    return RedisCacheManager.builder(f)
        .cacheDefaults(base.entryTtl(Duration.ofMinutes(5)))
        .withInitialCacheConfigurations(perCache)
        .transactionAware() // применять после commit
        .build();
  }

  @Bean
  public CaffeineCacheManager caffeine() {
    var cm = new CaffeineCacheManager("fx", "dict");
    cm.setCaffeine(Caffeine.newBuilder()
        .maximumSize(20_000)
        .expireAfterWrite(Duration.ofMinutes(10)));
    return cm;
  }

  @Bean
  public CacheManager composite(CaffeineCacheManager l1, RedisCacheManager l2) {
    var cm = new CompositeCacheManager(l1, l2);
    cm.setFallbackToNoOpCache(false);
    return cm;
  }
}
```

## SpEL: `condition/unless`, `sync=true` (singleflight); ошибки кэша

SpEL делает кэш «умнее». `condition` — **когда кэшировать** (до вызова), `unless` — **когда не класть** (после вызова, доступен `#result`). Например, `unless = "#result == null || #result.size() > 1000"` — не кэшировать слишком большие ответы. `sync = true` включает **singleflight** внутри инстанса.

Важно: **ошибки кэша** не должны валить бизнес-логику. Настройте `CacheErrorHandler`, чтобы логировать и «проглатывать» исключения из кэша. Для Redis-сбоев это особенно критично: пусть запрос пойдёт в origin и вернёт ответ, вместо 500.

Пример:

```java
@Bean
public CacheErrorHandler tolerantHandler() {
  return new CacheErrorHandler() {
    public void handleCacheGetError(RuntimeException e, Cache cache, Object key) { log.warn("cache get error", e); }
    public void handleCachePutError(RuntimeException e, Cache cache, Object key, Object val) { log.warn("cache put error", e); }
    public void handleCacheEvictError(RuntimeException e, Cache cache, Object key) { log.warn("cache evict error", e); }
    public void handleCacheClearError(RuntimeException e, Cache cache) { log.warn("cache clear error", e); }
  };
}
```

## Подводные камни прокси: self-invocation, final-классы (Kotlin), транзакции и момент кэширования

Spring Cache работает через **AOP-прокси**. Это означает:

* **Self-invocation**: вызов аннотированного метода из того же класса **обходит** прокси и кэш не сработает. Решения: выносить методы в отдельный бин, инжектить `self`-прокси, либо использовать AspectJ (реже).
* **Kotlin final**: по умолчанию классы/методы `final` → JDK-прокси не сможет перехватить. Используйте плагин `kotlin-spring` (делает классы/методы `open`) или `proxyTargetClass=true` (CGLIB).
* **Транзакции**: момент кэширования важен. Если кэшируем результат записи, делайте это **после коммита** (см. предыдущий наш разбор). Либо `RedisCacheManager.transactionAware()`, либо `@TransactionalEventListener(AFTER_COMMIT)`.
* **Видимость бинов**: аннотация должна стоять на `public` методе бина, управляемого Spring. Внутренние приватные/helper методы кэшем не покрываются.

Kotlin-gradle плагин:

```kotlin
plugins {
    kotlin("plugin.spring") version "1.9.24" // делает классы/методы open для прокси
}
```

## Метрики/трейсинг: Micrometer (hit/miss/evictions/size), @Observed

Spring Boot автоматически биндит метрики кэшей (Micrometer) для Caffeine/RedisCacheManager. Вы получите `cache.gets`, `cache.puts`, `cache.evictions`, `cache.size`, теги: `cache`, `name`. Используйте их для алёртов (шторм промахов, рост evictions, size → max).

Для трассировки окружайте «загрузчики» `@Observed`/`Observation`, чтобы видеть спаны «cache.load» и их p95. Это помогает найти «длинные» загрузки и отложить их в фон/выровнять TTL.

Пример:

```java
@Service
class FxService {
  @Observed(name = "fx.load", contextualName = "fx:load")
  @Cacheable(cacheNames = "fx", key = "#pair", sync = true, unless = "#result == null")
  public Rate rate(String pair) {
    return fxClient.fetch(pair); // наблюдаемый спан + кэш
  }
}
```

отлично — завершаю «Кэширование» подробным разбором **подтем 8–10**. Как и раньше, каждый подпункт раскрываю объёмно (4+ абзацев) и прикладываю рабочие фрагменты конфигураций/кода (Spring Boot 3.x, Java/Kotlin, Nginx/CDN).

---

# 8. HTTP/Edge кеширование

## Cache-Control, ETag/If-None-Match, Last-Modified/If-Modified-Since; Vary

HTTP-кеширование строится на **семантике протокола**: сервер сообщает клиенту/прокси, можно ли и как долго хранить ответ. Базовые директивы заголовка `Cache-Control` — `max-age` (сколько секунд ресурс годен), `public/private` (можно ли хранить в shared-кэшах), `no-store/no-cache` (запрет или обязательная перезаборка), `must-revalidate` (строгое соблюдение срока годности). Для статического контента разумно выставлять большие `max-age` и использовать **контентные версии** в URL (fingerprint), для динамического — короткие TTL или условия повторной валидации.

**Условная валидация** экономит трафик и латентность: клиент присылает `If-None-Match` с предыдущим **ETag** (хэш/версия ресурса) или `If-Modified-Since` с датой. Если сервер видит, что ресурс не изменился, он отвечает **304 Not Modified** без тела — дешёво и быстро. Практика: для коллекций используйте ETag, посчитанный от версии набора/максимального `updatedAt`, а для отдельных сущностей — от `version`/`updatedAt` конкретного объекта.

Заголовок **`Vary`** указывает, **от каких заголовков** зависит представление ресурса (например, `Accept-Encoding`, `Accept-Language`, `Authorization`, кастомные `X-Tenant`/`X-Variant`). Если забыть `Vary: Accept-Language`, CDN может отдать английскую версию русскоязычному клиенту — классический источник «кэш-поизнга». Поэтому список `Vary` — часть контракта API/фронта и должен поддерживаться дисциплинированно.

В Spring MVC легко собрать правильный ответ: `ResponseEntity` с `cacheControl`, `eTag`, `lastModified`. А для условных запросов используйте `WebRequest.checkNotModified(etag,lastModified)` — Spring сам вернёт 304, если всё совпало. Пример:

```java
@GetMapping("/api/products/{id}")
public ResponseEntity<ProductDto> product(@PathVariable long id, WebRequest req) {
  var dto = service.find(id); // с версией/updatedAt
  String etag = "\"" + dto.version() + "\"";
  long lastModified = dto.updatedAt().toInstant().toEpochMilli();
  if (req.checkNotModified(etag, lastModified)) return null; // 304
  return ResponseEntity.ok()
      .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)).mustRevalidate())
      .eTag(etag)
      .lastModified(lastModified)
      .body(dto);
}
```

## stale-while-revalidate/stale-if-error на CDN/Reverse-proxy; инвалидация по тегам/путям

Директивы **`stale-while-revalidate`** и **`stale-if-error`** (расширения RFC 5861/Cache-Control Extensions) позволяют отдавать **устаревший** (stale) ответ некоторое время. В первом случае прокси отдаёт старый ответ **сразу**, а обновление делает **в фоне**; во втором — при ошибке origin (5xx/timeout) можно «спасти» пользователей, отдав старое. Эти механики поддерживают современные CDN (CloudFront/Fastly) и многие обратные прокси (Varnish, часть Nginx-сборок), и они драматически улучшают SLA при качелях на backend.

Инвалидация на edge бывает **по путям** (purge `/images/*`) и **по тегам** (surrogate keys). Теги удобнее: один объект может участвовать в десятках представлений (страница категории, карточка товара, список рекомендаций). Присвоив всем этим ответам тег `product:42`, вы сможете одним PURGE удалить их из edge-кэша. Для этого в ответ добавляют `Surrogate-Key: product:42 category:7`, а в CDN настраивают маппинг purge-команд.

Если CDN/прокси не поддерживает surrogate keys, остаётся **версионирование URL** (fingerprint) и принудительный `PURGE`/`BAN` по маскам путей. Это менее точечно и может стоить дороже (временное «холодное» состояние кэша на всё дерево путей), но тоже рабочий подход для редких обновлений.

Spring может помогать формировать эти заголовки: добавьте кастомный `Filter`/`HandlerInterceptor`, который по DTO/модели собирает список тегов и проставляет `Surrogate-Key`. При изменении ресурса публикуйте событие во внутренний «инвалидатор», который дергает CDN API:

```java
@Component
class SurrogateKeyInterceptor implements HandlerInterceptor {
  @Override public void postHandle(HttpServletRequest req, HttpServletResponse res, Object handler, ModelAndView mav) {
    var keys = (List<String>)req.getAttribute("surrogateKeys");
    if (keys != null && !keys.isEmpty()) res.addHeader("Surrogate-Key", String.join(" ", keys));
  }
}
```

## Безопасность: приватные ответы vs shared cache, signed URLs/cookies

**Shared caches** (CDN, корпоративные прокси) не должны хранить приватные ответы. Помечайте такие ответы `Cache-Control: private, no-store` или вообще не кэшируйте. Любой ответ, зависящий от `Authorization`/персональных атрибутов, — кандидат на «только клиентский кэш» (браузер). Забудете — получите утечку персональных данных через «общий» слой.

Для ограниченного публичного контента (например, платный файл/видео) используйте **подписанные URL/куки**: CDN проверяет подпись/время жизни и отдаёт контент, а сам объект можно держать глубоко закэшированным (большой TTL). Это разгружает origin, а безопасность обеспечивается криптографией. Следите за синхронизацией времени и уязвимостями переиспользования токенов.

Не логируйте чувствительные заголовки (`Authorization`, `Cookie`) в access-логах edge. В `Vary` старайтесь избегать `Authorization`, иначе кэш превращается в «почти пустой» (вариантов столько, сколько токенов). Лучше отделять публичные и приватные эндпоинты.

И, наконец, не забывайте про **белые списки методов**: кэшируют только **безопасные** и **идемпотентные** ответы (GET/HEAD, реже OPTIONS). POST/PUT/DELETE по дефолту не кэшируются; попытки «хакнуть» кэширование небезопасных ответов часто заканчиваются уязвимостями.

## API-слой: идемпотентные GET, кэшируемые 200/301/404; GraphQL — отдельные стратегии

На уровне REST-API хорошая практика — сделать **GET** действительно идемпотентным и кэшируемым. Кэшируемые коды — **200/203/206/301/308/404/410** (в зависимости от контекста), и на них стоит выставлять корректные директивы. Ошибки 5xx/429 не кэшируйте, чтобы не «замораживать» аварии; для этого есть `stale-if-error` на edge.

Для **GraphQL** стратегия отличается: один URL, разные **операции** (query/mutation) и **переменные**. Популярный подход — **APQ** (Automatic Persisted Queries): клиент посылает **хэш запроса**, сервер — реальный запрос при промахе. На CDN можно кэшировать по ключу `opName + hash(variables)` и контролировать TTL на уровне «типа» операции. Переносить `Vary` по заголовкам (например, `X-Client-Features`) — нормальная практика, но следите за кардинальностью ключа.

В WebFlux/MVC удобно инкапсулировать правила кэширования в **аспект/фильтр**, привязав его к метаданным операции (аннотация рядом с контроллером). Тогда разработчики не будут вручную «подбирать» директивы, а контракт останется единым.

---

# 9. ORM/БД и вычислительные кеши

## Hibernate 2nd Level/Query Cache: регионы, режимы (read-only/transactional/non-strict), когда полезно

**Второй уровень кэша Hibernate (L2)** хранит сущности/коллекции между сессиями и разделяется всеми EntityManager’ами. Он **не обязателен** и включается там, где много **повторных чтений одних и тех же сущностей** и низкая изменчивость. В качестве провайдеров — Ehcache 3, Infinispan, Hazelcast; Redis тоже встречается (через адаптеры), но аккуратнее.

Режимы кэширования: **READ_ONLY** — для неизменяемых сущностей (справочники) — самый быстрый и безопасный; **NONSTRICT_READ_WRITE** — редкие обновления, допускается «слегка устаревшее»; **READ_WRITE** — жестче, с «мягкими» блокировками; **TRANSACTIONAL** — синхрон с JTA. Для большинства микросервисов хватает READ_ONLY/ NONSTRICT_READ_WRITE на справочниках и тяжёлых read-моделях.

**Query cache** — кэширует **результаты HQL/Criteria** (списки ID/значений) и завязан на regions + «timestamps region». Полезен для небольших, часто повторяемых запросов с одинаковыми параметрами. Но есть ловушки: он **не спасает от N+1**, он кэширует **результат запроса**, а не каждую строку (хотя строки могут подтянуться из L2). При высокой кардинальности параметров query cache бесполезен.

Аннотирование сущностей и включение кэша:

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY, region = "dict.products")
class Product { /* ... */ }

@QueryHints({
  @QueryHint(name = org.hibernate.annotations.QueryHints.CACHEABLE, value = "true"),
  @QueryHint(name = org.hibernate.annotations.QueryHints.CACHE_REGION, value = "q.products.byCategory")
})
@Query("select p from Product p where p.category.id = :catId")
List<Product> findByCategory(@Param("catId") long catId);
```

`application.yml` (Ehcache/Infinispan) и Hibernate:

```yaml
spring:
  jpa:
    properties:
      hibernate.cache.use_second_level_cache: true
      hibernate.cache.use_query_cache: true
      hibernate.cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
    hibernate:
      ddl-auto: validate
```

## Кеши справочников/идемпотентных lookups, материализованные представления/снапшоты

Справочники (валюты, страны, статусы) — идеальные кандидаты для READ_ONLY L2 и/или прикладного кэша (Caffeine/Redis) с длинным TTL. Идемпотентные lookups по «натуральному ключу» (поиск по коду/slug) — тоже отличная цель: вы разгружаете БД от «мелочи».

**Материализованные представления** (MV) — способ «предрассчитать» тяжёлые запросы и отдавать быстрый SELECT. В Postgres — `CREATE MATERIALIZED VIEW … WITH NO DATA; REFRESH MATERIALIZED VIEW CONCURRENTLY …`. Обновление MVs можно запускать по расписанию (cron) или по событиям (CDC → job). Для API MV можно дополнительно кэшировать (Redis) — получится «двойной» буст.

**Снапшоты** — сериализованные срезы агрегатов (например, профиль пользователя + рекомендации) в быструю KV (Redis). Снапшоты пересчитывают фоном или при событиях изменения, а API делает O(1) GET. Это особенно полезно, когда фронту нужна «страница» из десятка источников — всё собрано заранее.

Пример MV в Postgres:

```sql
create materialized view mv_top_products as
select p.id, p.name, sum(o.qty) as sold
from orders o join products p on p.id=o.product_id
where o.created_at > now() - interval '7 days'
group by p.id, p.name;

create unique index on mv_top_products(id);

-- обновление
refresh materialized view concurrently mv_top_products;
```

## Кеширование результатов дорогих вычислений (агрегации, прайсинг), дедуп запросов

Дорогие **агрегации** (price rules, ML score, отчёты) разумно кэшировать на уровне **функции**: `Cacheable` по `(операция, параметры)`. Важно проектировать **ключ** так, чтобы он отражал сущностные параметры (версия формулы/правила). Изменили алгоритм — bump версии в ключе.

Дедупликация запросов (singleflight) и коалесинг — must-have для вычислительных кешей: один поток рассчитывает, остальные ждут или получают предыдущий результат, если актуален. В распределённой среде используйте Redis-локи/lease-токены на ключ «операции».

Ещё одна практика — **частичная кэшируемость**: кэшируется **часть** результата (например, список ID) и **дешёвое** достраивание (hydrate) выполняется всегда. Это уменьшает объём кэша и снижает риск несогласованности (ID-список меняется реже, чем поле каждого объекта).

Пример «функционального» кэша:

```java
@Cacheable(cacheNames = "pricing", key = "T(java.util.Objects).hash(#sku, #currency, #userTier) + ':v3'")
public BigDecimal price(String sku, Currency currency, String userTier) {
  return pricingEngine.calculate(sku, currency, userTier);
}
```

## Ловушки: несогласованность, N+1 под query cache, область жизни

Главная ловушка ORM-кэшей — **ложное чувство безопасности**. L2/Query Cache не заменяют **правильные запросы**. Если есть N+1, кэш может «замести следы» на небольших сэмплах, но под нагрузкой вы всё равно упрётесь в сеть/базу (или в промахи кэша). Всегда профилируйте SQL и смотрите план выполнения.

Вторая ловушка — **несогласованность**. Если приложение A обновляет БД, а приложение B читает из L2/Query Cache и **не инвалидирует** соответствующие регионы, вы отдадите устаревшие данные. Решение: событийная инвалидация (CDC → invalidate), строгие регионы READ_ONLY только для **по-настоящему** неизменяемых вещей.

Третья — **область жизни**. Кэш на слой ORM — одна из иерархий (L1/L2/application/edge). Плохо, когда слой L2 «конкурирует» с Redis/сервисным кэшем и они расходятся по политике TTL/инвалидаций. Выберите «исторически главный» кэш и подчините ему остальные (например, Redis как источник правды кэш-уровня).

Четвёртая — **кардинальность** query cache. Шаблон «поиск по разным фильтрам» часто приводит к миллионам уникальных ключей — такой кэш бесполезен и только съест память. Кэшируйте либо **детальные** сущности (по ID), либо **стабильные** и немногочисленные выборки.

---

# 10. Эксплуатация, тестирование и безопасность

## Тесты: изоляция L1/L2, Embedded/TC Redis, проверка TTL/refresh, штормы/отказы

Юнит-тесты кэшируемых сервисов должны **изолировать** кэш от внешнего мира: используйте `Caffeine`/`ConcurrentMapCache` для L1 и **Testcontainers Redis** для L2 — это ближе к реальности, чем embedded-эмуляторы. Для интеграций включайте реальный `RedisCacheManager` и проверяйте **TTL** через команду `TTL`/`PTTL`.

Сценарии: **refresh-ahead** («старый» ответ отдан, фоновое обновление произошло), **sync=true** (одна загрузка на ключ под конкурентными запросами), **шторм промахов** (100 параллельных GET одновременно при истёкшем ключе), **отказ L2** (симулируйте `connectTimeout`/`readTimeout`: кэш не должен валить бизнес). Для последнего пригодится **Toxiproxy** или Lettuce `timeout` + WireMock, чтобы показать деградацию.

Проверка TTL на Redis в тесте:

```java
@Testcontainers
class CacheIT {
  @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7.2").withExposedPorts(6379);
  @DynamicPropertySource static void props(DynamicPropertyRegistry r) {
    r.add("spring.data.redis.host", () -> redis.getHost());
    r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
  }

  @Autowired StringRedisTemplate redisTpl;
  @Autowired ProductService svc;

  @Test void ttl_is_set() {
    svc.get(42L);
    String key = "app::product::42"; // зависит от конфигурации префикса
    Long ttl = redisTpl.getRequiredConnectionFactory().getConnection().keyCommands().ttl(key.getBytes());
    assertThat(ttl).isBetween(1L, 900L); // например, <=15 минут
  }
}
```

## Наблюдаемость: per-key top-N, промахи/штормы, «холодный старт», warm-up/priming

Метрики уровня «кэш-в-целом» (hits/misses/evictions) полезны, но для инцидентов нужен **срез по ключам**. Введите счётчики **per-key-class** (product/category/fx/pricing) и ведите top-N ключей по промахам/длительности загрузки (можно в агрегирующем ZSET в Redis, обновляемым редким фоном, либо в Prometheus с кардинальностью до 100–200).

Следите за **cold start**: после релиза/перезапуска доля промахов растёт — warm-up/приминг должны её гасить. Отдельная метрика — **stampede detector**: всплеск конкурентных промахов на один ключ (N>threshold за окно). Сигнализируйте и включайте «карантин» для ключа.

В логах используйте **корреляцию**: ключ/регион/elapsed/load-source (L1/L2/origin), чтобы быстро вычленять, где проблема. Добавьте «маркеры качества» в бизнес-метрики: % ответов из кэша на endpoint, p95 с и без кэша.

Пример простого **обёртывания** загрузчика метриками:

```java
class MeteredLoader<K,V> implements Function<K,V> {
  private final Function<K,V> delegate;
  private final Timer timer;
  private final Counter errors;
  MeteredLoader(String name, Function<K,V> delegate, MeterRegistry r) {
    this.delegate = delegate;
    this.timer = r.timer("cache.load", "name", name);
    this.errors = r.counter("cache.load.errors", "name", name);
  }
  public V apply(K key) {
    return timer.record(() -> {
      try { return delegate.apply(key); }
      catch (RuntimeException e) { errors.increment(); throw e; }
    });
  }
}
```

## Катастрофы: потеря Redis, split-brain, медленная сеть — планы B/C

Кэш — зависимость. Он имеет право «упасть», «медленно отвечать», «расщепиться» (split-brain на кластере/HA). План **B**: при ошибках кэша **не падать**, а идти в origin (CacheErrorHandler «глотает» исключения). План **C**: снижать нагрузку на origin — сокращать TTL в ответах, выключать refresh, включать частичный ответ/заглушки. Убедитесь, что бюджеты origin выдержат временный рост RPS (нагрузочные тесты без кэша).

Split-brain (две «мастера», рассинхронизация) приводит к **расходящимся** кешам. Если используете менеджер с автоматическим failover (Sentinel/Cluster), убедитесь в **идемпотентности** инвалидаций и в коротких TTL на «чувствительных» регионах, чтобы система «сошлась». Хорошая идея — выключать «запись в кэш» на время аварии (feature-flag) и работать в read-through режиме.

Медленная сеть — скрытый убийца SLA. Ограничьте **таймауты** клиента Redis (commandTimeout), используйте **bulk**/pipeline для массовых операций и держите **локальный fallback** (L1) хоть с коротким TTL, чтобы смягчить пик.

Сделайте **плейбуки**: как очистить регионы, как массово bump-нуть версию префикса, как включить/выключить флаги деградации, какие дашборды смотреть и в каком порядке.

## Хардненинг: защита от cache-poisoning, валидация входящих, лимиты ключей/значений

**Cache poisoning** случается, когда в кэш попадает неправильное/вредоносное значение (ошибка origin, инъекция параметров). Лечится **валидацией** результата перед кэшированием: схемная проверка (JSON Schema/Jakarta Validation), лимиты размера, checksum/версия. Если не прошли проверку — **не кэшировать** и логировать.

Лимиты: допустимая **длина ключа**, **размер значения**, **время загрузки** (если loader > SLO — карантин). Это спасает от «ядовитых» ключей и перегруза сети/памяти. На уровне Redis включайте `maxmemory-policy` (лучше `allkeys-lru`/`volatile-lru`) и контролируйте eviction-метрики.

Следите за **Vary**/ключами: не включайте PII в ключ в явном виде (e-mail → хэш), не допускайте бесконтрольного роста кардинальности (персональные feature-flags в ключ — путь к взрыву). В логах ключи маскируйте.

Пример простой схемной валидации перед кэшированием:

```java
@Cacheable(cacheNames="userProfile", key="#id", unless="#result == null || !#root.target.valid(#result)")
public Profile getProfile(long id) { return service.fetch(id); }

public boolean valid(Profile p) {
  return p.getEmail() != null && p.getEmail().contains("@") && p.getName().length() <= 200;
}
```

## Governance: чек-листы «что кэшируем», бюджеты устаревания, ревью TTL/инвалидаций

Кэш — это **продуктовое** решение, не только инженерное. Введите **чек-лист**:
(1) Что кэшируем и зачем (цели SLA/стоимости).
(2) Допустимая **устарелость** (SLO stale-budget).
(3) Политики TTL/TTI/refresh и инвалидации.
(4) Ключи/версии/сегментация (tenant/locale/AB).
(5) Безопасность/PII.
(6) Наблюдаемость/алерты.
(7) Тесты (штормы, отказы).
(8) Планы B/C.

Любое новое кэширование проходит **ревью**: разработчик приносит «паспортизованный» кэш с измерениями до/после, выбранными TTL и механизмом инвалидации. На релизах — **фиксация версии префикса** и план прогрева. Раз в N месяцев — аудит регионов: что не окупается, что надо ужесточить/смягчить.

И наконец — **документация**: где живут ключи, как их инвалидировать вручную/скриптом, какие «теги» у эндпоинтов на edge, какие метрики смотреть при инциденте. Это экономит часы on-call’у и снижает MTTR.


Ниже — 30 собес-вопросов по теме «Кэширование» с чёткими ответами и короткими примерами (Java 17+, Spring Boot 3.x).

---

1. **Что такое кэш и зачем он нужен?**
   Кэш — быстрый слой хранения (память/Redis/CDN) между приложением и origin (БД/HTTP). Он снижает латентность, нагрузку на зависимости, стабилизирует SLA под пиками и часто экономит деньги.

2. **Когда кэш окупается, а когда нет?**
   Окупается при read-heavy доступе и «горячих» ключах. В write-heavy сценариях выгода мала из-за частых инвалидаций; лучше кэшировать справочники/агрегации и дорогие GET.

3. **Чем отличаются L1 и L2 кэши?**
   L1 (локальный, in-proc) — минимальная задержка, но виден одному инстансу. L2 (распределённый, Redis/Hazelcast) — общий для кластера, переживает рестарты, но есть сетевые издержки; часто комбинируют L1+L2 (near-cache).

4. **Какие основные паттерны кэширования вы знаете?**
   Cache-Aside (дефолт в микросервисах), Read-Through/Write-Through (кэш как фасад к данным), Write-Behind (буферная запись), Refresh-Ahead (фоновое обновление). Выбор зависит от владения данными и требований к согласованности.

5. **Что такое Cache-Aside и как выглядит в Spring?**
   Приложение сначала читает из кэша, при промахе — в origin, кладёт в кэш; при изменении — явно инвалидирует. В Spring: `@Cacheable` для чтений и `@CacheEvict/@CachePut` после записи.

```kotlin
@Cacheable("product", key="#id", sync=true) fun get(id: Long)=repo.findDto(id)
@CacheEvict("product", key="#id") @Transactional fun update(id:Long,c:Cmd){ repo.update(id,c) }
```

6. **В чём разница Read-Through/Write-Through и Write-Behind?**
   Read/Write-Through пропускают все операции через кэш (он сам ходит в origin), Write-Behind пишет в origin асинхронно (быстро, но сложнее гарантии). В бизнес-сервисах чаще применяют Cache-Aside.

7. **Что такое TTL/TTI и когда выбирать одно или другое?**
   TTL — протухание по времени с момента записи; TTI — протухание по неиспользованию (каждый доступ «освежает»). Для справочников чаще TTL, для «редко трогаемых хвостов» — TTI.

8. **Как избежать stampede (шторм промахов)?**
   Включить singleflight на инстансе (`@Cacheable(sync=true)`), в кластере — пер-ключевые локи (Redisson `RLock`) или lease-токены в Redis + короткий TTL, добавить TTL-jitter и/или refresh-ahead.

9. **Что такое Refresh-Ahead и Soft/Hard TTL?**
   Refresh-Ahead — фоновое обновление ключа до истечения. Soft TTL: можно отдавать устаревшее и параллельно обновлять; Hard TTL: после него значение нельзя отдавать.

10. **Как проектировать ключи кэша?**
    Шаблон: `ns:v:tenant:locale:filters:id`. Включайте только параметры, от которых реально зависит ответ, обязательно version (для эволюции схем), изолируйте tenant/variant/locale.

11. **Какие риски у кэширования и как их снижать?**
    Устаревшие данные, cache-poisoning, сложность инвалидаций и штормы. Снижаем через чёткие TTL/SLA stale-budget, валидацию перед кэшированием, событийную инвалидацию и защиту от stampede.

12. **Как правильно инвалидировать «множество» связанных ключей?**
    Тегововая (surrogate-keys) инвалидация: у каждого ответа есть теги (`product:42`, `category:7`), удаляем по тегу. В Redis реализуем через `SET` списков ключей на тег и пайплайн/ Lua-скрипт для удаления.

13. **Что такое near-cache и как его сделать?**
    Это L1 над L2: читаем из L1, при промахе — L2 → origin; инвалидации распространяем по pub/sub. Быстрый способ — Redisson `RLocalCachedMap`; либо `CompositeCacheManager` + собственная логика инвалидаций.

14. **Как настроить Redis для кэша (минимум параметров)?**
    Задайте таймауты (connect/command), пул соединений, политику памяти (`maxmemory-policy`), персистентность (для «чистого» кэша можно без неё). В Boot: Lettuce `commandTimeout`, pool, `RedisCacheManager` с TTL и сериализатором.

```java
@Bean RedisCacheManager rcm(RedisConnectionFactory f,ObjectMapper om){
 var ser=new GenericJackson2JsonRedisSerializer(om);
 var cfg=RedisCacheConfiguration.defaultCacheConfig()
   .entryTtl(Duration.ofMinutes(10))
   .serializeValuesWith(SerializationPair.fromSerializer(ser))
   .prefixCacheNameWith("app:");
 return RedisCacheManager.builder(f).cacheDefaults(cfg).transactionAware().build();
}
```

15. **Какие структуры Redis полезны для кэширования?**
    `String` для сериализованных значений; `Hash` для частичных полей; `Set/ZSet` для связей (теги/топы); `Bitmap/Bloom` для экономичных отметок/проверок «видели ли id». Выбор влияет на память и стоимость операций.

16. **Зачем Redis Lua и пример использования?**
    Для атомарности сложных операций (CAS/lease). Пример: установить lease, если его ещё нет, с TTL — один поток грузит, остальные ждут/отказываются.

```lua
-- SETNX + PEXPIRE
if(redis.call('SETNX', KEYS[1], ARGV[1])==1) then redis.call('PEXPIRE', KEYS[1], ARGV[2]); return 1 else return 0 end
```

17. **Как сериализовать значения в Redis? Почему не Java-serialization?**
    Используйте JSON/CBOR/Smile/ProtoBuf; в Spring — `GenericJackson2JsonRedisSerializer`. Java-serialization небезопасна и хрупка для эволюции схем, кросс-языковости нет.

18. **Как включить «транзакционную осведомлённость» кэша?**
    Включить `spring.cache.transaction.enabled=true` или `RedisCacheManager.transactionAware()`. Тогда `put/evict` применяются **после коммита**, что исключает «ранний» сброс при rollback.

19. **Какие аннотации Spring Cache и в чём разница?**
    `@Cacheable` — чтение/кладём при промахе; `@CachePut` — всегда кладём (обновляем); `@CacheEvict` — удаляем; `@Caching` — композиция. Часто: `get` — `@Cacheable(sync=true)`, `update` — `@CachePut` + `@CacheEvict` списка.

20. **Почему `@Cacheable` может «не сработать»?**
    Прокси-ограничения: self-invocation (вызов самого себя), `final`-классы/методы в Kotlin без `kotlin-spring`, не-public методы. Решение: вынос в отдельный бин, включить CGLIB или использовать AspectJ.

21. **Как безопасно интегрировать кэш с транзакциями записи?**
    Публиковать доменное событие и в `@TransactionalEventListener(AFTER_COMMIT)` выполнять `evict/put`, либо включить `transactionAware`. Так исключаем ситуации, когда кэш очистился, а транзакция откатилась.

22. **Как настроить условия кэширования в SpEL?**
    `condition` — «кэшировать только если…» (до вызова), `unless` — «не класть если…» (после вызова, доступен `#result`). Пример: `unless="#result==null || #result.size()>1000"` — не кэшировать слишком большие ответы.

23. **Какие заголовки участвуют в HTTP-кэшировании?**
    `Cache-Control` (max-age, public/private, stale-while-revalidate, stale-if-error), `ETag`/`If-None-Match`, `Last-Modified`/`If-Modified-Since`, `Vary`. `Vary` обязателен для параметров, влияющих на представление (язык/тенант/вариант).

24. **Чем помогает `stale-while-revalidate` и `stale-if-error` на edge?**
    Позволяют отдавать устаревшее мгновенно, пока идёт фоновая актуализация (`s-w-r`), и сохранять SLA при ошибках origin (`s-i-e`). Существенно сглаживает пики и «дрожь» бэкенда.

25. **Что такое surrogate-keys на CDN и как их использовать?**
    Это теги для ответов (`Surrogate-Key: product:42 category:7`). По одному PURGE-запросу инвалидируется пачка связанных объектов. Удобно для страниц, где один объект «участвует» во многих представлениях.

26. **Hibernate L2 и Query Cache: когда включать и какие режимы?**
    Включать для редко меняемых сущностей/часто повторяемых чтений (справочники). Режимы: `READ_ONLY` (безопасно/быстро), `NONSTRICT_READ_WRITE`, `READ_WRITE`, реже `TRANSACTIONAL`. Query Cache кэширует **результаты запросов** и полезен только при низкой кардинальности параметров.

27. **Чем опасен Query Cache и N+1?**
    Query Cache не лечит N+1 — он кэширует список ID/строк, а затем всё равно может тянуть сущности по одной. Нужны правильные выборки/фетч-графы; иначе кэш «скрывает» проблему на малых сэмплах.

28. **Как тестировать кэширование?**
    Юнит — с Caffeine/ConcurrentMapCache; интеграции — с Testcontainers Redis и `RedisCacheManager`. Проверяйте TTL (`PTTL`), `sync=true` под конкурентной нагрузкой, деградацию при таймаутах Redis (CacheErrorHandler не должен валить бизнес).

29. **Какие метрики и логи важны для кэша?**
    Hit/miss/evictions/size, p95/p99 «loader», время в очереди при stampede, top-N ключей по промахам. В логах — источник ответа (L1/L2/origin), ключ-класс, длительность; включить трейсинг вокруг loader (`@Observed`).

30. **Как обеспечить безопасность кэширования (PII/poisoning)?**
    Не кэшируйте PII/секреты в shared-кэшах; если нужно — шифруйте и ставьте короткие TTL. Валидация результата перед кэшированием, лимиты размера/времени, хешируйте чувствительные части ключей. Для приватных ответов — `Cache-Control: private, no-store`.
