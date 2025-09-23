---
layout: page
title: "Spring Beans"
permalink: /spring/beans
---

## Что такое бин и кто управляет его жизненным циклом

Бин в Spring — это объект, который создаётся, настраивается и управляется контейнером Spring (ApplicationContext). В отличие от «ручного» `new`, создание бина идёт по декларации (аннотации, Java-конфиг, авто-конфигурации), контейнер отслеживает зависимости, область видимости (scope) и жизненный цикл (инициализация/уничтожение). В момент поднятия контекста Spring строит граф зависимостей и гарантирует, что каждый бин получит валидные зависимости или приложение не стартует.

Жизненный цикл включает стадии: обнаружение определения (definition), создание экземпляра (instantiate), внедрение свойств/зависимостей (populate), вызов `BeanPostProcessor`’ов (before/after initialization), пользовательские хуки (`@PostConstruct`, `initMethod`), работа в рантайме, затем корректное завершение (`@PreDestroy`, `destroyMethod`). Контейнер обеспечивает упорядоченность и последовательность этих шагов.

Важно понимать: «живёт» не класс, а **экземпляр**, строгий владелец которого — контекст. Потому даже если вы вручную создадите объект, Spring о нём не узнает, не внедрит зависимости и не вызовет хуки жизненного цикла — такой объект будет «вне экосистемы» контейнера.

Отсюда следствие для архитектуры: бизнес-классы не должны знать, **кто** их создаёт. Они декларируют зависимости, а контейнер решает, **когда** и **в какой конфигурации** создавать экземпляры. Это упрощает тестирование (моки/стабы), переиспользование и замену реализаций.

---

## Регистрация бинов: стереотипы vs @Bean, базовое сканирование, имена, @Primary/@Qualifier

Два основных декларативных способа регистрации бинов: стереотипы (`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController` и т. п.) и методы-конфигураторы `@Bean` в классах `@Configuration`. Стереотипы удобны для вашего «прикладного» кода: достаточно повесить аннотацию — и класс попадёт в контекст через компонент-скан. `@Bean` — точечный фабричный способ, особенно хорош для сторонних классов (из библиотек), когда вы не можете/не хотите их помечать аннотациями.

В Spring Boot базовый пакет для сканирования определяется **пакетом главного класса** с `@SpringBootApplication`: всё внутри этого пакета и его подпакетов автоматически сканируется. Если структура другая — задайте явно: `@SpringBootApplication(scanBasePackages = "com.acme.app")` или используйте `@ComponentScan` на нужных пакетах.

Имя бина по умолчанию — это либо **имя метода** для `@Bean`, либо **lowerCamelCase от имени класса** для стереотипов (например, `class OrderService` → бин `orderService`). Имя можно задать явно: `@Component("billingOrderService")` или `@Bean(name = "primaryDataSource")`. Явные имена уместны, когда вы хотите стабильные идентификаторы в логике, `@Qualifier` или в условиях (`@ConditionalOnBean(name="...")`).

Разрешение неоднозначностей: `@Primary` помечает «дефолтного» кандидата среди нескольких реализаций одного интерфейса, а `@Qualifier` позволяет выбрать конкретный бин по имени/квалификатору. Типичный приём: одну реализацию пометить `@Primary`, остальные — именованными `@Qualifier`.

```java
public interface TaxPolicy { BigDecimal apply(BigDecimal net); }

@Component("basicTax")
class BasicTaxPolicy implements TaxPolicy {
    public BigDecimal apply(BigDecimal net) { return net.multiply(new BigDecimal("0.13")); }
}

@Component("reducedTax")
@Primary
class ReducedTaxPolicy implements TaxPolicy {
    public BigDecimal apply(BigDecimal net) { return net.multiply(new BigDecimal("0.06")); }
}

@Service
class InvoiceService {
    private final TaxPolicy defaultPolicy;
    private final TaxPolicy basicPolicy;

    InvoiceService(TaxPolicy defaultPolicy, @Qualifier("basicTax") TaxPolicy basicPolicy) {
        this.defaultPolicy = defaultPolicy; // ReducedTaxPolicy из-за @Primary
        this.basicPolicy = basicPolicy;     // Явный выбор по @Qualifier
    }
}
```

---

## @Configuration, проксирование методов @Bean, proxyBeanMethods=false, static @Bean, «тонкость прямого вызова»

Аннотация `@Configuration` делает класс **конфигурацией**: его методы с `@Bean` регистрируют бины. При `proxyBeanMethods = true` (классический режим) Spring создаёт CGLIB-прокси вокруг конфигурации, чтобы перехватывать **внутренние вызовы** методов `@Bean`. Это обеспечивает семантику «один и тот же singleton на каждый вызов», даже если вы вызываете `otherBean()` вручную внутри другого метода.

Когда уместно `proxyBeanMethods = false`? Когда ваши `@Bean`-методы **не вызывают друг друга напрямую**, а все зависимости приходят параметрами методов. Такой «lite mode» ускоряет старт и снижает накладные расходы на прокси. В авто-конфигурациях Spring Boot это дефолтный стиль; в прикладном коде — применяйте осознанно.

Почему прямой вызов опасен при `proxyBeanMethods = false`? Потому что при «лайтовом» режиме **внутренние вызовы не перехватываются**, и вы можете неосознанно создать **новый экземпляр** зависимого бина, обходя контейнер и его кэш синглтонов. Правильно — передавать зависимости параметрами метода `@Bean`, чтобы контейнер их резолвил.

```java
@Configuration(proxyBeanMethods = true) // «полный» режим
class DataConfig {

    @Bean
    DataSource dataSource() { /* создаём pooled DataSource */ }

    @Bean
    JdbcTemplate jdbcTemplate() {
        // прямой вызов безопасен, будет перехвачен и вернёт singleton из контекста
        return new JdbcTemplate(dataSource());
    }
}

@Configuration(proxyBeanMethods = false) // «лайт» режим
class WebConfig {

    @Bean
    RestClient restClient() { return RestClient.create(); }

    @Bean
    MyApi myApi(RestClient client) { // передаём зависимость параметром
        return new MyApi(client);
    }
}
```

Отдельный нюанс — `static @Bean`: статические бины инициализируются **раньше** экземпляра конфигурации. Это нужно, если бин сам по себе — пост-процессор фабрики (например, `PropertySourcesPlaceholderConfigurer`, `BeanFactoryPostProcessor`) и должен существовать до создания остальных бинов. Такой бин укажите как `public static` метод, чтобы гарантировать раннее подключение.

---

## Коллекции реализаций, @DependsOn, initMethod/destroyMethod, BPP vs BFPP

Spring умеет внедрять **все реализации** интерфейса сразу — как `List<T>` (в порядке `@Order`) и как `Map<String, T>` (где ключ — имя бина). Это база для паттернов «плагины»/«стратегии». Плюс: не нужно знать конкретные классы — достаточно контрактов.

```java
public interface DiscountPolicy { BigDecimal apply(BigDecimal price); }

@Service
class PriceCalculator {
    private final List<DiscountPolicy> policies;
    private final Map<String, DiscountPolicy> policiesByName;

    PriceCalculator(List<DiscountPolicy> policies, Map<String, DiscountPolicy> policiesByName) {
        this.policies = policies;
        this.policiesByName = policiesByName;
    }

    public BigDecimal applyAll(BigDecimal price) {
        for (DiscountPolicy p : policies) price = p.apply(price);
        return price;
    }
}
```

`@DependsOn` обозначает **жёсткую зависимость по порядку инициализации**. Это не DI-ссылка, а «контракт старта»: сначала создайте X, потом — Y. Полезно для «подготовителей» ресурсов (прогрев кэша, миграция схемы не в БД, прогрев локальных словарей), где сама зависимость не нужна, но порядок важен.

`initMethod`/`destroyMethod` у `@Bean` дают удобный способ связать методы жизненного цикла без аннотаций в третьесторонних классах. Это особенно уместно для сторонних клиентов/серверов (Netty, embedded-broker, длительные соединения).

```java
@Bean(initMethod = "start", destroyMethod = "stop")
public NettyServer nettyServer(ServerConfig cfg) {
    return new NettyServer(cfg.port());
}
```

`BeanPostProcessor` (BPP) работает с **экземплярами бинов**: может обернуть, заменить, добавить прокси, внедрить сквозную функциональность (логирование, метрики, безопасность). `BeanFactoryPostProcessor` (BFPP) работает с **определениями** (BeanDefinition) до создания экземпляров: позволяет править свойства, добавлять/убирать бины, влиять на скоуп и автосвязывание. Разница в фазе и уровне: BPP — «над объектами», BFPP — «над метаданными».

---

## Скоупы (scope), почему prototype + singleton — ловушка, @Lazy и способы решения

Скоуп определяет, **сколько живёт** экземпляр и **когда создаётся**: `singleton` (по умолчанию, 1 на контекст), `prototype` (новый на каждый запрос к контейнеру), веб-скоупы (`request`, `session`, `application`, `websocket`). В большинстве приложений 90% бинов — `singleton`: они статичны, потокобезопасны и дешёвы.

Прямая инъекция `prototype` в `singleton` — ловушка: контейнер создаст **один** экземпляр `prototype` **на момент создания** синглтона и будет использовать его всегда. Повторные вызовы методов с ожиданием «нового» экземпляра окажутся обманутыми. Это свойство DI-графа, а не баг: зависимости «замораживаются» при сборке графа.

Как правильно получать «свежие» экземпляры: использовать `ObjectProvider<T>`/`ObjectFactory<T>`/`javax.inject.Provider<T>` и вызывать `.getObject()` по мере необходимости; для веб-скоупов — прокси-скоуп (scoped proxy), чтобы прокси запрашивал актуальный объект из текущего контекста запроса/сессии; альтернативно — `@Lookup`-метод (динамический метод-фабрика, который контейнер переопределяет).

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class Task { /* состояние на единичную операцию */ }

@Service
class TaskService {
    private final ObjectProvider<Task> taskProvider;
    TaskService(ObjectProvider<Task> taskProvider) { this.taskProvider = taskProvider; }

    public Result runOnce(Input in) {
        Task t = taskProvider.getObject();
        return t.process(in);
    }
}
```

`@Lazy` откладывает создание до первого реального использования. Это ускоряет старт, уменьшает память на старте и помогает разрубить редкие циклы. Но цена — перенос ошибок инициализации в runtime-точку первого доступа, возможные гонки (если ленивый бин не потокобезопасен), «холодные» пути (первый вызов медленнее). Включайте точечно, документируйте причины и ограждайте тяжёлые инициализации.

---

## Условная регистрация, получение бина по имени, смешивание стилей, дубликаты типов, модульная конфигурация, @Configuration vs @Component, FactoryBean

Условная регистрация бинов решает задачу «создавать бин только если…». Базовый механизм — `@Conditional` с кастомным `Condition`: вы решаете на основе окружения/системных свойств/классов на classpath. В Spring Boot есть удобные сокращения: `@ConditionalOnProperty`, `@ConditionalOnMissingBean`, `@ConditionalOnClass`, `@ConditionalOnExpression` — они покрывают 90% кейсов прикладной конфигурации.

```java
@Configuration
class MetricsConfig {

    @Bean
    @ConditionalOnProperty(prefix = "metrics", name = "enabled", havingValue = "true", matchIfMissing = true)
    MeterRegistry registry() { return new SimpleMeterRegistry(); }

    @Bean
    @ConditionalOnMissingBean(MeterRegistry.class)
    MeterRegistry fallbackRegistry() { return new NoopMeterRegistry(); }
}
```

Иногда нужен доступ «по имени» в рантайме — например, выбрали стратегию строковым ключом и хотите получить бин: используйте `ApplicationContext.getBean(String)` или внедрите `Map<String, T>` и достаньте по ключу из коллекции. Прямые `getBean` лучше изолировать в одном месте, чтобы не размывать DI и не плодить сервис-локатор по всему коду.

Смешивать стереотипы и Java-конфигурацию в одном проекте — нормально и правильно: прикладные классы через стереотипы (меньше бойлерплейта), сторонние объекты/клиенты — через `@Bean`. В больших проектах это дополните **модульной конфигурацией**: выделите несколько `@Configuration` по слоям/фичам (WebConfig, DataConfig, MessagingConfig), собирайте их в «корневую» через `@Import`, а внешним модулям — публикуйте стартеры.

Дубликаты типов: если два `@Bean` возвращают один и тот же тип без `@Primary`/`@Qualifier`, автосвязывание по типу упадёт с `NoUniqueBeanDefinitionException`. Ровно для этого и существуют `@Primary` и именованные `@Qualifier`. Для коллекций это не ошибка — наоборот, желаемая ситуация.

Практическая разница `@Configuration` и `@Component`: `@Configuration` в «полном» режиме (`proxyBeanMethods = true`) включает проксирование `@Bean`-методов и обеспечивает корректную семантику singleton при прямых вызовах внутри конфигурации. `@Component` с методами `@Bean` работает в «лайтовом» режиме: методы **не проксируются**, прямые вызовы создадут новые экземпляры и сломают семантику DI. Поэтому если вы пишете фабричные методы и иногда вызываете один `@Bean` из другого — используйте именно `@Configuration` (или `proxyBeanMethods=false` + зависимости только параметрами).

`FactoryBean<T>` — фабрика, чьим «продуктом» является бин типа `T`. Внедряя по имени фабрики, вы получаете **продукт**, а чтобы получить саму фабрику — используйте имя с префиксом `&`.

```java
@Component("httpClientFactory")
class HttpClientFactoryBean implements FactoryBean<CloseableHttpClient> {
    public CloseableHttpClient getObject() { return HttpClients.createDefault(); }
    public Class<?> getObjectType() { return CloseableHttpClient.class; }
    public boolean isSingleton() { return true; }
}

// Внедрение продукта:
@Autowired CloseableHttpClient client;
// Получение самой фабрики:
@Autowired @Qualifier("&httpClientFactory") FactoryBean<CloseableHttpClient> factory;
```

Наконец, почему **прямой вызов** `otherBean()` из класса с `@Configuration(proxyBeanMethods=false)` может нарушить семантику DI? Потому что вы обойдёте контейнер и создадите новый объект, минуя кэш синглтонов и весь пайплайн пост-процессоров. Правильный способ в lite-режиме — передавать зависимости параметрами метода `@Bean`, чтобы контейнер их резолвил сам.

---

## Потенциальные ошибки новичков

1. Использовать `@Component` вместо `@Configuration` там, где методы `@Bean` вызывают друг друга. Итог — дублирующиеся экземпляры, загадочные баги. Решение: `@Configuration(proxyBeanMethods=true)` **или** не вызывать `@Bean`-методы вручную, а передавать зависимости параметрами и использовать `proxyBeanMethods=false`.

2. Инъекция `prototype`/`request` напрямую в `singleton`. Итог — один «замороженный» экземпляр. Решение: `ObjectProvider<T>`/`Provider<T>`/`@Lookup`/scoped-proxy.

3. «Глобальный» `@Lazy` на всё подряд. Итог — отложенные ошибки в проде, холодные пути, непрогретые клиенты. Решение: точечный `@Lazy`, понятные комментарии/документация, прогрев в `ApplicationRunner` при необходимости.

4. Дублирующиеся `@Bean` одного типа без `@Primary`/`@Qualifier`. Итог — `NoUniqueBeanDefinitionException` на старте. Решение: назначить `@Primary` дефолтной реализации и использовать `@Qualifier` для альтернатив.

5. Прямое использование `ApplicationContext.getBean()` по всему коду. Итог — сервис-локатор, жёсткая связность. Решение: DI через конструктор, `Map<String,T>`/`List<T>` там, где нужен «каталог» реализаций, и только точечные обращения к контексту в инфраструктурном слое.

---

## Итог

В этой теме вы увидели, **как на самом деле живут бины**: от определения до уничтожения, чем отличаются способы регистрации (`@Component`-семейство против `@Bean`), как работает проксирование конфигураций и почему `proxyBeanMethods` — не «галочка наугад», а осознанное решение. Мы разобрали внедрение коллекций реализаций, порядок инициализации через `@DependsOn`, хуки `initMethod/destroyMethod`, различия BPP и BFPP, скоупы и их ловушки, ленивую инициализацию и условия регистрации.

Ключевые практики:
— Предпочитайте явные зависимости параметрами `@Bean`-методов;
— Используйте `@Configuration(proxyBeanMethods=true)` только если действительно вызываете `@Bean`-методы друг из друга;
— Для «свежих» экземпляров применяйте `ObjectProvider`/прокси-скоупы;
— Разруливайте неоднозначности `@Primary`/`@Qualifier`;
— Делайте конфигурацию модульной и читаемой;
— Применяйте условные аннотации из Spring Boot вместо собственных велосипедов, если они покрывают кейс.
