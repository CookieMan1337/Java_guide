---
layout: page
title: "Основы Spring: Inversion of Control & Dependency Injection"
permalink: /spring/ioc_di
---

## Inversion of Control & Dependency Injection 

Инверсия управления (IoC) в контексте Spring — это принцип, при котором объекты не создают и не связывают свои зависимости напрямую. Вместо этого создание и «склейка» объектов передаются **контейнеру** Spring (ApplicationContext). Разработчик описывает, какие компоненты нужны и как их найти, а контейнер берёт на себя конструирование и жизненный цикл. Практический эффект: бизнес-класс не знает, откуда берутся его зависимости, и благодаря этому остаётся простым и тестируемым.

Внедрение зависимостей (DI) — это конкретный способ реализовать IoC: зависимости «вкалываются» в объект контейнером. DI отвечает на вопрос «как поместить нужный объект в другой объект»: через конструктор, сеттер или поле. IoC — идея «кто управляет процессом», DI — техника «как именно передать зависимость». В Spring почти всё, что вы делаете, — это объявляете зависимости, а контейнер их удовлетворяет.

Бин (Bean) в Spring — это управляемый контейнером объект: созданный, сконфигурированный и помещённый в контекст. Бин может быть компонентом приложения (`@Service`, `@Repository`, `@Controller`), конфигурационным объектом (`@Bean`), инфраструктурным сервисом (DataSource, EntityManagerFactory) и т. д. Контейнер знает его имя, тип, область видимости (scope), жизненный цикл.

ApplicationContext — это «сердце» Spring: хранит бины, находит их, связывает зависимости, публикует события, отдаёт ресурсы и читает окружение/настройки. Когда вы запускаете Spring Boot приложение, он поднимает контекст, выполняет авто-конфигурации, сканирует компоненты, создаёт бины и «втыкает» их друг в друга.


Начнём с основной диаграммы ролей. У вас есть три типа слоёв: контроллер, сервис и репозиторий. Вся связка держится на DI — зависимости приходят «сверху» из контейнера.

Простой пример: конструкторная инъекция (предпочтительный подход).

```java
// Сервисный контракт
public interface GreetingService {
    String greet(String name);
}

// Реализация по умолчанию
@Service
public class DefaultGreetingService implements GreetingService {
    @Override
    public String greet(String name) {
        return "Hello, " + name;
    }
}

// Веб-контроллер
@RestController
@RequestMapping("/api")
public class GreetingController {
    private final GreetingService service; // зависимость

    // Конструкторная инъекция — зависимость обязательна и проверяется на старте
    public GreetingController(GreetingService service) {
        this.service = service;
    }

    @GetMapping("/greet")
    public String greet(@RequestParam String name) {
        return service.greet(name);
    }
}
```

Как контейнер находит бины? Spring Boot по умолчанию сканирует пакеты, начиная с пакета главного класса с `@SpringBootApplication`. Любой класс, помеченный `@Component`, `@Service`, `@Repository`, `@Controller`, попадает в контекст. Плюс вы можете объявлять бины через Java-конфиг:

```java
@Configuration
public class AppConfig {
    @Bean
    public Clock clock() {
        return Clock.systemUTC();
    }
}
```

Стереотипы `@Component`, `@Service`, `@Repository`, `@Controller` — это один и тот же базовый механизм (`@Component`) с разной семантикой слоя. Это повышает читаемость и помогает инфраструктуре: например, `@Repository` включает перевод SQL-исключений в `DataAccessException`, а `@Controller` — связывает методы с HTTP-маршрутами.

Способы внедрения зависимостей: через конструктор, сеттер или поле. Поля удобны, но минус — неотслеживаемость обязательности зависимости и сложности тестирования. Сеттеры могут быть уместны для опциональных зависимостей. Конструкторная инъекция делает зависимость обязательной, облегчает тестирование (можно подать мок) и предотвращает появление полупроинициализированных объектов — поэтому считается предпочтительной.

Разрешение неоднозначностей при автосвязывании по типу работает так: если бинов одного типа несколько, Spring сперва ищет `@Primary`, затем учитывает `@Qualifier`, затем может использовать имя параметра. Если всё равно неоднозначно — упадёт с `NoUniqueBeanDefinitionException`. Пример использования `@Primary` и `@Qualifier`:

```java
public interface PaymentClient { String charge(int amount); }

@Service @Primary
class FastPaymentClient implements PaymentClient {
    public String charge(int amount) { return "fast:" + amount; }
}

@Service
@Qualifier("secure")
class SecurePaymentClient implements PaymentClient {
    public String charge(int amount) { return "secure:" + amount; }
}

@Service
class CheckoutService {
    private final PaymentClient defaultClient;
    private final PaymentClient secureClient;

    CheckoutService(PaymentClient defaultClient,
                    @Qualifier("secure") PaymentClient secureClient) {
        this.defaultClient = defaultClient;     // FastPaymentClient из-за @Primary
        this.secureClient = secureClient;       // SecurePaymentClient по @Qualifier
    }
}
```

Области видимости (scopes) отвечают за жизненный цикл бина. По умолчанию — `singleton`: один экземпляр на весь контекст. Это быстро и экономно по памяти на инфраструктурные вещи. `prototype` создаёт новый экземпляр при каждом запросе к контексту — уместен для «одноразовых» объектов с состоянием. В веб-приложениях есть специальные области: `request` (на HTTP-запрос), `session` (на HTTP-сессию), `application` (на ServletContext), `websocket` (на WebSocket-сессию). Пример:

```java
@Service
@Scope("request")
class RequestScopedContext {
    private final String id = UUID.randomUUID().toString();
    public String id() { return id; }
}
```

Важно: если вы внедряете `request`/`prototype` в `singleton`, получите «ловушку»: контейнер подставит один экземпляр при создании синглтона, и он будет «застывшим». Для корректной семантики нужен ленивый доступ к фабрике экземпляров:

```java
@Service
class UsesPrototype {
    private final ObjectProvider<RequestScopedContext> provider; // или Provider<T>

    UsesPrototype(ObjectProvider<RequestScopedContext> provider) {
        this.provider = provider;
    }

    public String compute() {
        return provider.getObject().id(); // каждый вызов — актуальный экземпляр
    }
}
```

Жизненный цикл бинов поддерживает хуки `@PostConstruct` (инициализация) и `@PreDestroy` (освобождение ресурсов). Они вызываются контейнером после создания/перед уничтожением:

```java
@Service
class ResourcefulService implements AutoCloseable {
    private CloseableHttpClient client;

    @PostConstruct
    void init() {
        client = HttpClients.createDefault();
    }

    @PreDestroy
    void shutdown() throws Exception {
        client.close();
    }

    @Override public void close() throws Exception { shutdown(); }
}
```

BeanPostProcessor vs BeanFactoryPostProcessor: первый работает **над экземплярами бинов** — позволяет оборачивать/модифицировать уже созданные объекты, например, добавлять прокси, логировать, внедрять перехватчики. Второй работает **над метаданными фабрики** до создания бинов — вы можете менять `BeanDefinition`, добавлять/удалять/править определения. Пример простого `BeanPostProcessor`:

```java
@Component
class TimingPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof GreetingService) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    long t0 = System.nanoTime();
                    try { return method.invoke(bean, args); }
                    finally { System.out.println(beanName + "." + method.getName() + " took " + (System.nanoTime()-t0)); }
                }
            );
        }
        return bean;
    }
}
```

Внедрение примитивов/строк/настроек: через `@Value` и через типобезопасные `@ConfigurationProperties`. Первый путь быстрый, второй — предпочтителен для групп настроек.

```yaml
app:
  timeout-ms: 2000
  base-url: https://example.org
```

```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppProps {
    private int timeoutMs;
    private URI baseUrl;
    // getters/setters
}

@Service
class Client {
    Client(AppProps props) { /* используем props.getTimeoutMs(), props.getBaseUrl() */ }
}
```

Управление порядком инициализации: жёстких гарантий порядка нет — контейнер создаёт бины по зависимостям. Если нужно явно указать, что A должен быть раньше B — используйте `@DependsOn("aBean")` на B. Для выполнения действий после старта контекста — `ApplicationRunner`/`CommandLineRunner`. `@Order` влияет на порядок внутри коллекций при автосвязывании, а не на момент создания.

```java
@Component
@Order(10)
class RunnerA implements ApplicationRunner { public void run(ApplicationArguments args){ /*...*/ } }

@Component
@Order(20)
class RunnerB implements ApplicationRunner { public void run(ApplicationArguments args){ /*...*/ } }
```

Расширяемость контейнера («контейнер расширяем»): кроме пост-процессоров, есть `BeanDefinitionRegistryPostProcessor` (можно регистрировать бины программно), `ApplicationContextInitializer` (подготовка контекста до refresh), `ApplicationListener` (реакция на события контекста), `FactoryBean` (кастомная фабрика бинов). Реальные кейсы: автоматическая регистрация клиентов, оборачивание репозиториев метриками/трассировкой, внедрение кросс-срезов (логирование, безопасности) без изменения бизнес-кода.

Можно ли смешивать Java-конфиг `@Bean` и авто-сканирование? Да, это нормальная практика. Рекомендуемый подход: свой код — через стереотипы и сканирование; сторонние объекты/фабрики/клиенты библиотек — через `@Bean` в `@Configuration`. Это повышает прозрачность и избавляет от избыточных «магических» сканов.

Что произойдёт при ошибке резолвинга зависимостей? Если бин отсутствует — `NoSuchBeanDefinitionException` на этапе поднятия контекста. Если кандидатов несколько — `NoUniqueBeanDefinitionException`. Эти ошибки «хорошие»: вы получаете фидбек на старте, а не в рантайме, поэтому конструкторная инъекция полезнее — она заставляет контейнер проверить всё сразу.

Ленивая инициализация `@Lazy` откладывает создание бина до первого использования. Это помогает ускорить старт и разорвать циклические зависимости, но будьте осторожны: ошибка инициализации переедет в рантайм первого вызова. Безопасно применять `@Lazy` к тяжёлым клиентам/провайдерам, которые не нужны всем путям выполнения.

Циклические зависимости опасны: A зависит от B, B — от A. При конструкторной инъекции это сразу выявляется (контейнер не сможет построить граф), при полевой/сеттерной — может сработать благодаря ранним ссылкам, но легко получить NPE и трудно диагностируемые баги. Правильное решение — разорвать цикл через выделение третьего компонента, событийную модель, командный интерфейс или внедрение фабрики/поставщика для одной из сторон.

Когда нужен `BeanPostProcessor`? Когда требуется массово и единообразно модифицировать/оборачивать бины определённого типа: добавить трассировку, метрики, валидацию, автопроксирование аспектов. Если нужна правка именно **определений** бинов (менять свойства до создания) — используйте `BeanFactoryPostProcessor`/`BeanDefinitionRegistryPostProcessor`.

И наконец, вопрос «зачем выбирать между стереотипами и `@Bean`, если можно оба?». Реальный ответ — не выбирать «или-или», а применять по месту. Стереотипы дают минимальный бойлерплейт для вашего кода, `@Bean` — точный контроль для внешних/инфраструктурных объектов. В одном проекте обычно сосуществуют оба подхода.

---

## Потенциальные ошибки новичков

Ошибка 1: внедрение `prototype`/`request` бина напрямую в `singleton`. Результат — один экземпляр, «замороженный» на момент создания синглтона. Решение: `ObjectProvider<T>` / `Provider<T>` / `@Lookup`-метод или scoped-proxy для веб-скоупов. Это обеспечивает получение актуального экземпляра при каждом обращении.

Ошибка 2: злоупотребление полевой инъекцией. Она скрывает обязательность зависимостей, усложняет тестирование и способствует циклам. Конструкторная инъекция решает эти проблемы: зависимости видны, обязательность проверяется на старте, моки легко подставить в тестах.

Ошибка 3: неоднозначности при автосвязывании. Несколько реализаций одного интерфейса без `@Primary`/`@Qualifier` приводят к `NoUniqueBeanDefinitionException`. Решение: помечайте одну реализацию как `@Primary` и явно выбирайте остальные через `@Qualifier`.

Ошибка 4: попытки управлять порядком «создания» бинов через `@Order`. Аннотация влияет на порядок **коллекций** при инъекции и на порядок раннеров/аспектов, но не навязывает контейнеру общий граф инициализации. Если нужен строгий порядок — используйте зависимости (`A` зависит от `B`) или `@DependsOn`.

---

## Итог

IoC и DI — фундамент Spring. IoC отвечает «кто управляет жизненным циклом и связями» (контейнер), DI — «как зависимости попадают внутрь» (конструктор/сеттер/поле). Такой подход делает код слабосвязанным, тестируемым и предсказуемым: классы описывают контракты, контейнер реализует сборку.

Бин — это единица управления контейнера. Он имеет тип, имя, область видимости и жизненный цикл. Контейнер находит бины через сканирование стереотипов и Java-конфигурации, связывает их по типу/квалификаторам, управляет инициализацией и уничтожением. Веб-скоупы дают правильную семантику для объектов, живущих на запрос/сессию, а `singleton` остаётся оптимальным по умолчанию.

Разработчику важно понимать механизм разрешения кандидатов (`@Primary`, `@Qualifier`), уметь внедрять настройки (`@Value`, `@ConfigurationProperties`), корректно работать с жизненным циклом (`@PostConstruct`, `@PreDestroy`), разруливать порядок инициализации (`@DependsOn`, Runner’ы) и, при необходимости, расширять контейнер (пост-процессоры). Эти знания предотвращают типовые ловушки, особенно вокруг прототипов, ленивых бинов и циклов.

Практически: предпочитайте конструкторную инъекцию, используйте `@Component`/`@Service`/`@Repository`/`@Controller` для своего кода и `@Bean` для сторонних объектов; не смешивайте скоупы без фабрик/провайдеров; помечайте «дефолтную» реализацию `@Primary` и названные — `@Qualifier`; применяйте `@ConfigurationProperties` для конфигурации; добавляйте расширения контейнера, только когда они действительно упрощают код. С таким каркасом вы уверенно пройдёте к следующим темам — транзакциям, веб-слою, миграциям БД и наблюдаемости.
