---
layout: page
title: "Бины и конфигурация: как «живёт» приложение"
permalink: /spring/beans
---
# 0. Введение

## Что это

Бин (Bean) — это управляемый контейнером Spring объект с известной идентичностью, скоупом и жизненным циклом. Вокруг него существует «паспорт» — `BeanDefinition`, где описано, как бин создаётся, какие у него зависимости, какой у него скоуп и когда он должен быть уничтожен. Когда мы говорим «внедрение зависимостей», на практике это означает: контейнер создаёт бины, связывает их в граф и подставляет в конструкторы других бинов.

Важный нюанс: бин — это не просто «любой объект». Он «родился» от контейнера (через сканирование компонентов или конфигурацию), поэтому на него распространяются инфраструктурные механизмы Spring: пост-процессоры, аспекты, транзакции, кеширование, безопасность, метрики. Если вы создадите тот же класс через `new`, это будет уже не бин, и многие «магии» не сработают.

Spring Boot упрощает жизнь тем, что автоконфигурирует контейнер: соберёт стандартные бины (веб-слой, Jackson, валидатор, DataSource и т.д.) из стартеров и включит их по условиям. Но принцип остаётся тем же: это объект, управляемый контекстом, который «знает» про его создание и смерть.

Бины связаны зависимостями. Чаще всего мы используем конструкторную инъекцию: класс объявляет, что ему нужно, и контейнер подставляет подходящие бины по типу (или по квалификатору). Это делает код слабосвязанным и заменяемым: реализацию можно переключать профилями/свойствами.

Граф бинов — это сердцебиение приложения. По нему видно архитектуру: где домен, где адаптеры, где инфраструктура. Хороший проект читается по конструкторам: заглянул в сервис — сразу понятно, что он требует на вход и какими портами пользуется.

И, наконец, «бин» — это единица расширения. Пост-процессоры могут оборачивать его прокси, добавлять кросс-срезы (логирование, метрики), менять поведение без правки бизнес-кода. Это фундаментальная причина, почему стоит «играть по правилам» контейнера.

**Код (Java): минимальная программа с бином**

```java
package app.intro;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@SpringBootApplication
public class IntroApp {
  public static void main(String[] args) { SpringApplication.run(IntroApp.class, args); }
}

interface Greeter { String hello(String name); }

@Component
class SimpleGreeter implements Greeter {
  @Override public String hello(String name) { return "Hello, " + name; }
}

@Service
class WelcomeService {
  private final Greeter greeter;
  public WelcomeService(Greeter greeter) { this.greeter = greeter; }
  public String welcome(String name) { return greeter.hello(name); }
}
```

**Код (Kotlin): тот же пример**

```kotlin
package app.intro

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

@SpringBootApplication
class IntroApp
fun main(args: Array<String>) = runApplication<IntroApp>(*args)

interface Greeter { fun hello(name: String): String }

@Component
class SimpleGreeter : Greeter {
    override fun hello(name: String) = "Hello, $name"
}

@Service
class WelcomeService(private val greeter: Greeter) {
    fun welcome(name: String) = greeter.hello(name)
}
```

Gradle (Groovy):

```groovy
plugins { id 'org.springframework.boot' version '3.3.4'; id 'io.spring.dependency-management' version '1.1.6'; id 'java' }
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

Gradle (Kotlin):

```kotlin
plugins { id("org.springframework.boot") version "3.3.4"; id("io.spring.dependency-management") version "1.1.6"; kotlin("jvm") version "1.9.25"; kotlin("plugin.spring") version "1.9.25" }
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

## Зачем это

Контейнер бинов снимает с разработчика рутину создания и связывания объектов. Вы декларируете роли классов, а инфраструктура берёт на себя порядок инициализации, внедрение зависимостей, проксирование и жизненный цикл. Это ускоряет разработку и уменьшает количество ошибок «порядка старта».

Бины повышают тестируемость. Конструкторная инъекция означает, что вы легко подменяете зависимости фейками/моками и тестируете бизнес-логику без реальных интеграций. Для интеграционных тестов контейнер поднимет «настоящие» бины.

Гибкость конфигурации — ещё одна причина. Один и тот же класс может стать разным бином в разных профилях: в `dev` — фейковый клиент, в `prod` — реальный с TLS, трейсингом и пулом. Переключение — через свойства, а не через `if`-ы в коде.

На бины удобно навешивать кросс-срезы: транзакции, кеширование, ретраи, метрики. Это делается «снаружи» через прокси/аспекты, а не «вшивается» в бизнес-методы. Код остаётся чистым, а поведение — расширяемым.

Жизненный цикл под контролем. Контейнер вызовет `@PostConstruct` для инициализации, `@PreDestroy` для освобождения ресурсов, гарантирует «мягкую» остановку по сигналу. Вы меньше думаете о служебных деталях и сосредотачиваетесь на домене.

Наконец, бины — это прозрачность. По именам и типам видно, какие части системы активны, а по отчётам Actuator можно понять, что именно поднято и как сконфигурировано.

**Код (Java): профили — зачем и как)**

```java
package app.why;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

interface Mailer { void send(String to, String body); }

@Component @Profile("dev")
class ConsoleMailer implements Mailer {
  public void send(String to, String body) { System.out.println("DEV mail: " + to + " -> " + body); }
}

@Component @Profile("prod")
class SmtpMailer implements Mailer {
  public void send(String to, String body) { /* SMTP call */ }
}
```

**Код (Kotlin): профили — аналог**

```kotlin
package app.why

import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

interface Mailer { fun send(to: String, body: String) }

@Component @Profile("dev")
class ConsoleMailer : Mailer {
    override fun send(to: String, body: String) = println("DEV mail: $to -> $body")
}

@Component @Profile("prod")
class SmtpMailer : Mailer {
    override fun send(to: String, body: String) { /* SMTP */ }
}
```

---

## Где используется

Бины — везде, где есть слои приложения: контроллеры (`@RestController`), сервисы (`@Service`), репозитории (`@Repository`), клиенты внешних систем (`@Component` + конфигурация), конфиг-классы (`@Configuration`), слушатели (`@EventListener`). Это единая модель для веба, задач и реактивного стека.

В интеграциях бины представляют «порты и адаптеры». Порт — интерфейс домена, адаптер — конкретная реализация (например, HTTP-клиент). Контейнер «подставит» нужный адаптер в сервис. Так домен изолирован от технологий и легче тестируется.

Во входном слое бины формируют маршрутизацию: фильтры, интерсепторы, контроллеры — все они бины, и на них легко навешиваются кросс-срезы (метрики, трейсинг). Это делает поведение централизованно управляемым.

В хранилищах бины управляют ресурсами: пул соединений — бин, `EntityManagerFactory` — бин, репозитории — бины. Контейнер закрывает их корректно на остановке, а в рантайме умеет инспектировать состояние.

В задачах и «воркерах» бины играют ту же роль: объявляете обработчик, читаете из Kafka, пишете в БД — всё это управляемые объекты с понятной жизнью. То же касается batch-джобов и планировщиков.

И, наконец, в тестах: спринговые «срезы» поднимают ровно те бины, что нужны для сценария (`@WebMvcTest`, `@DataJpaTest`), экономя время и упрощая диагностику.

**Код (Java): контроллер + сервис + репозиторий как бины)**

```java
package app.where;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.web.bind.annotation.*;
import org.springframework.stereotype.*;

@Entity class User { @Id public String id; public String name; }

interface UserRepo extends JpaRepository<User, String> {}

@Service
class UserService {
  private final UserRepo repo;
  UserService(UserRepo repo) { this.repo = repo; }
  public User create(String id, String name) { var u = new User(); u.id=id; u.name=name; return repo.save(u); }
}

@RestController
@RequestMapping("/users")
class UserController {
  private final UserService service;
  UserController(UserService service) { this.service = service; }
  @PostMapping public User create(@RequestParam String id, @RequestParam String name) { return service.create(id, name); }
}
```

**Код (Kotlin): аналогичное расположение бинов)**

```kotlin
package app.where

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.*
import jakarta.persistence.Entity
import jakarta.persistence.Id

@Entity class User(@Id var id: String = "", var name: String = "")

interface UserRepo : JpaRepository<User, String>

@Service
class UserService(private val repo: UserRepo) {
    fun create(id: String, name: String) = repo.save(User(id, name))
}

@RestController
@RequestMapping("/users")
class UserController(private val service: UserService) {
    @PostMapping fun create(@RequestParam id: String, @RequestParam name: String) = service.create(id, name)
}
```

---

## Какие проблемы/задачи решает

Контейнер решает проблему «кто создаёт кого и когда». Без него порядок старта и зависимости между объектами часто оказываются неявными. С контейнером у нас есть регистрация дефиниций, проверка конфликтов, предсказуемое создание инициализированных синглтонов.

Вторая задача — ресурсная дисциплина. Пулы соединений, планировщики, клиентские SDK должны корректно стартовать и завершаться. Контейнер вызывает `@PostConstruct`/`@PreDestroy`, закрывает ресурсы, останавливает задачи. Мы не теряем дескрипторы и не «роняем» JVM при остановке.

Третья — сквозные политики. Транзакции, кеширование, ретраи, безопасность включаются через AOP/прокси на уровне бинов. Не нужно дублировать одно и то же во всех классах. Проще управлять и проверять.

Четвёртая — конфигурируемость. Все параметры («проволока») — во внешних настройках; один образ — много окружений. Автоконфигурации подключают нужные бины при наличии классов и свойств.

Пятая — наблюдаемость. Через пост-процессоры и автоконфигурации легко включить метрики/трейсинг на все бины нужного типа, не трогая бизнес-код. Это экономит недели инженерного времени.

Шестая — безопасность изменений. Конфликты бинов, циклы зависимостей и ошибки wiring’а ловятся при старте. Чем раньше ошибка, тем дешевле её исправить.

**Код (Java): инициализация/завершение ресурса)**

```java
package app.problems;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class ConnectionHolder {
  @PostConstruct void init() { /* open connections / warm-up */ }
  @PreDestroy void shutdown() { /* close connections */ }
}
```

**Код (Kotlin): то же)**

```kotlin
package app.problems

import jakarta.annotation.PostConstruct
import jakarta.annotation.PreDestroy
import org.springframework.stereotype.Component

@Component
class ConnectionHolder {
    @PostConstruct fun init() { /* open connections / warm-up */ }
    @PreDestroy fun shutdown() { /* close connections */ }
}
```

---

# 1. Что такое бин и откуда он берётся

## BeanDefinition: метаданные о типе, скоупе, зависимостях, способе создания

Внутри контейнера каждый бин представлен `BeanDefinition` — структурой с метаданными: какой класс создавать, какие конструкторные аргументы подставить, какой скоуп (singleton/prototype/веб-скоуп), какие методы инициализации/уничтожения вызвать. Эти метаданные появляются из аннотаций, из `@Bean`-методов или из ручной регистрации.

`BeanDefinition` — это причина, по которой контейнер может «переигрывать» создание: менять скоуп, оборачивать прокси, добавлять зависимости, настраивать `autowire mode`. Благодаря этому пост-процессоры способны видоизменять граф до фактического создания объектов.

С практической точки зрения разработчик редко строит `BeanDefinition` руками, но полезно знать, что он существует, и уметь его читать. Это помогает при диагностике: почему бин прототип? почему у него такое имя? почему на него «нацепилась» транзакция?

Доступ к реестру дефиниций возможен через `BeanDefinitionRegistry` и `BeanFactory`. В пост-процессорах можно инспектировать и даже программно регистрировать новые бины. Так пишутся «автоконфиги» и плагины.

Важно понимать, что `BeanDefinition` — декларативная «паспортная» запись. Пока бин не создан, вы можете менять дефиницию. После создания синглтона менять его «на лету» неправильно; для этого существуют прокси/декораторы.

Наконец, `BeanDefinition` — не про бизнес-код, а про «проволоку». В рамках команды достаточно, чтобы 1–2 инженера глубоко понимали эти механизмы для сложных кейсов; остальным полезно знать «куда смотреть».

**Код (Java): вывод базовой информации о дефинициях)**

```java
package beans.defs;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;

@Component
class DefPrinter implements BeanFactoryPostProcessor {
  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) throws BeansException {
    for (String name : bf.getBeanDefinitionNames()) {
      var bd = bf.getBeanDefinition(name);
      System.out.println(name + " -> " + bd.getBeanClassName() + ", scope=" + bd.getScope());
    }
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package beans.defs

import org.springframework.beans.BeansException
import org.springframework.beans.factory.config.BeanFactoryPostProcessor
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory
import org.springframework.stereotype.Component

@Component
class DefPrinter : BeanFactoryPostProcessor {
    @Throws(BeansException::class)
    override fun postProcessBeanFactory(bf: ConfigurableListableBeanFactory) {
        bf.beanDefinitionNames.forEach { name ->
            val bd = bf.getBeanDefinition(name)
            println("$name -> ${bd.beanClassName}, scope=${bd.scope}")
        }
    }
}
```

---

## Источники бинов: компонент-сканирование (`@Component`-семейство), Java-конфигурация (`@Configuration` + `@Bean`), импорт (`@Import`, `@ImportSelector`), фабрики (`FactoryBean`)

Самый частый источник — компонент-сканирование. Вы помечаете класс `@Component`/`@Service`/`@Repository`/`@Controller`, Spring сканирует пакеты и регистрирует дефиниции. Это удобно для «обычных» сервисов/адаптеров.

Java-конфигурация — второй способ. `@Configuration`-класс объявляет методы-фабрики `@Bean`, каждый из которых регистрирует бин. Это идеально для инфраструктуры: пула соединений, HTTP-клиентов, кэшей. Так код настройки лежит рядом, его проще читать и тестировать.

`@Import` подключает дополнительные конфигурации/классы. Через `@ImportSelector` можно программно вернуть список классов к импорту по условиям (как делают автоконфигурации Spring Boot). Это мощная точка расширения, позволяющая строить «стартеры».

`FactoryBean<T>` — особый контракт: сам класс — не «тот» бин, а фабрика, которая производит бин типа `T`. Это полезно, когда создание сложное: нужно прочитать секреты, собрать клиент по билдеру, выбрать реализацию в рантайме. Контейнер узнаёт, что отдавать из контекста: `T` по имени, или `FactoryBean` по имени с префиксом `&`.

Импорт и фабрики позволяют держать код переиспользуемым. Библиотека может предоставить «набор» бинов и подключать их через условия/свойства. Приложение — решает, включать ли их.

Важно не смешивать подходы хаотично. Обычная практика: бизнес-код — через сканирование, инфраструктура/клиенты — через `@Bean`, подключаемые модули — через `@Import`.

**Код (Java): все источники сразу — примеры)**

```java
package beans.sources;

import org.springframework.context.annotation.*;
import org.springframework.stereotype.Component;

// 1) Компонент-сканирование
@Component
class PingService { public String ping() { return "pong"; } }

// 2) Java-конфигурация
@Configuration
class HttpConfig {
  @Bean java.net.http.HttpClient httpClient() { return java.net.http.HttpClient.newHttpClient(); }
}

// 3) Импорт конфигурации
@Configuration @Import(HttpConfig.class)
class RootConfig {}

// 4) FactoryBean
class Client { final String who; Client(String who) { this.who = who; } }
class ClientFactory implements FactoryBean<Client> {
  @Override public Client getObject() { return new Client("factory"); }
  @Override public Class<?> getObjectType() { return Client.class; }
}
@Configuration
class ClientConfig {
  @Bean ClientFactory client() { return new ClientFactory(); }
}
```

**Код (Kotlin): те же идеи)**

```kotlin
package beans.sources

import org.springframework.context.annotation.*
import org.springframework.stereotype.Component
import java.net.http.HttpClient

@Component
class PingService { fun ping() = "pong" }

@Configuration
class HttpConfig {
    @Bean fun httpClient(): HttpClient = HttpClient.newHttpClient()
}

@Configuration
@Import(HttpConfig::class)
class RootConfig

class Client(val who: String)
class ClientFactory : FactoryBean<Client> {
    override fun getObject(): Client = Client("factory")
    override fun getObjectType(): Class<*> = Client::class.java
}
@Configuration
class ClientConfig {
    @Bean fun client(): ClientFactory = ClientFactory()
}
```

---

## Именование и алиасы: каноническое имя, коллизии, правила чтения «дерева» бинов

Каждый бин имеет **каноническое имя** — строковый идентификатор в контейнере. Для компонентов по умолчанию это «camelCase-имя класса» (например, `simpleGreeter`). Для `@Bean` имя — это имя метода. Можно задавать своё имя явным образом.

Алиасы — дополнительные имена, по которым бин можно получить. Они удобны, когда один объект должен быть доступен под несколькими ролями/ключами. В `@Bean` можно указать массив имён. На уровне реестра возможна программная регистрация алиасов.

Коллизии имён приводят к перезаписи или ошибкам — в зависимости от настроек. Хорошая практика — единый стиль именования и отсутствие «магических» совпадений. Для множества реализаций используйте осмысленные имена и `@Qualifier`.

«Дерево» бинов — это то, как они зависимы. Смотреть на имена полезно для диагностики: по логам видно, какой конкретно бин выбрался. А в картах стратегий (`Map<String,Strategy>`) ключом именно служит имя бина — это удобно.

Стоит помнить про `FactoryBean`: по имени «client» вы получите `Client`, а по имени «&client» — саму фабрику. Это часто удивляет, когда пытаются залогировать «тип бина» и получают фабрику вместо продукта.

Стандартизируйте имена: `emailNotifier`, `smsNotifier`, `kafkaConsumerOrders`, `pricingHttpClient`. Так у вас будет единый, читаемый словарь.

**Код (Java): имена и алиасы)**

```java
package beans.names;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

@Component("emailNotifier")
class EmailNotifier {}

@Configuration
class NamesConfig {
  @Bean(name = {"mainClient","aliasClient"})
  String clientName() { return "X"; }
}
```

**Код (Kotlin): имена и алиасы)**

```kotlin
package beans.names

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.stereotype.Component

@Component("emailNotifier")
class EmailNotifier

@Configuration
class NamesConfig {
    @Bean(name = ["mainClient","aliasClient"])
    fun clientName(): String = "X"
}
```

---

# 2. Жизненный цикл бина (от регистрации до уничтожения)

## Фазы старта контекста: регистрация дефиниций → пост-процессоры → создание singleton-бинов → вызовы колбэков → `ApplicationReady`

Старт начинается с **регистрации дефиниций**: сканирование, чтение `@Configuration` и `@Bean`, подключение импортов. На этом этапе граф ещё «плоский», это просто записи в реестре: какие бины потенциально есть.

Затем запускаются **пост-процессоры фабрики** (`BeanFactoryPostProcessor`/`BeanDefinitionRegistryPostProcessor`). Они могут модифицировать дефиниции: менять скоуп, добавлять свойства, регистрировать новые бины. Это последняя точка, где можно «перекроить» план.

Дальше контейнер переходит к **созданию singleton-бинов** (eager by default). Он вычисляет зависимости, строит объекты, внедряет поля/сеттеры/конструкторные аргументы, применяет `BeanPostProcessor` до/после инициализации. Прототипы по умолчанию не создаются на старте.

После создания всех синглтонов вызываются «глобальные» хуки вроде `SmartInitializingSingleton`: это шанс выполнить код «после того, как все синглтоны готовы». Полезно для проверок целостности и проволоки.

Spring Boot затем публикует события `ApplicationStartedEvent` и `ApplicationReadyEvent`. Последнее означает, что приложение готово принимать трафик. Это удобная точка для прогрева кэшей, показа баннера готовности и старта фоновых процессов (осторожно).

Важно не перегружать старт: долгие задачи лучше вынести в отдельные джобы/инициаторы или выполнять асинхронно. Иначе вы получите большие `startup time` и сложную диагностику.

**Код (Java): слушатель событий старта)**

```java
package lifecycle.phases;

import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
class ReadyListener {
  @EventListener(ApplicationReadyEvent.class)
  public void onReady() { System.out.println("Application is READY"); }
}
```

**Код (Kotlin): слушатель `ApplicationReady`)**

```kotlin
package lifecycle.phases

import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class ReadyListener {
    @EventListener(ApplicationReadyEvent::class)
    fun onReady() = println("Application is READY")
}
```

---

## Инициализация бина: внедрение зависимостей → `Aware`-интерфейсы → `@PostConstruct`/`afterPropertiesSet`/init-method → пост-процессоры (`BeanPostProcessor`)

Когда контейнер создаёт экземпляр, он сначала **внедряет зависимости** (конструктор/сеттеры/поля). На этом этапе бин уже «собран», но ещё не инициализирован логически.

Затем вызываются **`Aware`-интерфейсы** (если реализованы): `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`. Это способ получить ссылку на инфраструктуру. В бизнес-коде их лучше избегать, чтобы не скатываться в Service Locator.

Далее — **инициализация**. Если есть `@PostConstruct`, он будет вызван. Если бин реализует `InitializingBean`, вызовется `afterPropertiesSet()`. Если у `@Bean` указан `initMethod`, он тоже вызовется. Обычно используют `@PostConstruct` как самый нейтральный способ.

Параллельно работают **пост-процессоры бинов** (`BeanPostProcessor`): метод `postProcessBeforeInitialization` может заменить бин до инициализации, а `postProcessAfterInitialization` — обернуть прокси/декоратор. Именно здесь включаются транзакции, кеширование, AOP.

Инициализация должна быть быстрой и безопасной. Не запускайте тяжёлые задачи в `@PostConstruct`, не блокируйте поток старта. Для прогрева используйте `ApplicationReadyEvent`/отдельные механизмы.

Помните, что порядок: зависимости → Aware → init → postProcessors(после init). Это помогает диагностировать «почему логика не сработала»: иногда дело в том, что вы сделали self-invocation до появления прокси.

**Код (Java): все стадии инициализации)**

```java
package lifecycle.init;

import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.stereotype.Component;

@Component
class InitDemo implements BeanNameAware, InitializingBean {
  private String name;
  @Override public void setBeanName(String name) { this.name = name; }

  @PostConstruct
  void post() { System.out.println("PostConstruct for " + name); }

  @Override
  public void afterPropertiesSet() { System.out.println("afterPropertiesSet for " + name); }
}
```

**Код (Kotlin): то же)**

```kotlin
package lifecycle.init

import jakarta.annotation.PostConstruct
import org.springframework.beans.factory.BeanNameAware
import org.springframework.beans.factory.InitializingBean
import org.springframework.stereotype.Component

@Component
class InitDemo : BeanNameAware, InitializingBean {
    private var name: String? = null
    override fun setBeanName(name: String) { this.name = name }

    @PostConstruct fun post() = println("PostConstruct for $name")
    override fun afterPropertiesSet() = println("afterPropertiesSet for $name")
}
```

---

## Завершение: `@PreDestroy`/`DisposableBean`/destroy-method, порядок остановки

При остановке приложения контейнер корректно завершает жизненный цикл бинов. Сначала публикуется `ContextClosedEvent`, затем для каждого бина вызываются колбэки разрушения: `@PreDestroy`, `DisposableBean.destroy()`, `destroyMethod` из `@Bean`.

На этапе завершения важно освободить ресурсы: закрыть соединения, остановить планировщики, слить буферы. Это помогает избежать утечек и «подвешенных» потоков, мешающих JVM корректно выйти.

Порядок остановки можно контролировать через `SmartLifecycle` и фазы. Компоненты с меньшим номером фаз останавливаются позже/раньше (в зависимости от логики), что позволяет аккуратно выключать потребителей прежде, чем отключать брокера/ресурсы.

Если вы используете «короткоживущие» скоупы (request/session), их объекты уничтожаются вместе со сессией/запросом. Но большинство инфраструктурных ресурсов — синглтоны, поэтому `@PreDestroy` — ваш друг.

Важно, чтобы уничтожение было **идемпотентно**. Колбэки могут вызываться повторно при некоторых сценариях (например, перезапуске контекста в тестах). Защитите операции от дублей.

В микросервисах корректная остановка особенно заметна: брокеры/балансировщики меньше «ругаются» на разорванные соединения, нагрузочные пики при rolling-update снижаются.

**Код (Java): завершение ресурсов)**

```java
package lifecycle.destroy;

import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class ResourceHolder {
  @PreDestroy
  void shutdown() {
    // close pools, stop schedulers
    System.out.println("Resources closed");
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package lifecycle.destroy

import jakarta.annotation.PreDestroy
import org.springframework.stereotype.Component

@Component
class ResourceHolder {
    @PreDestroy
    fun shutdown() {
        println("Resources closed")
    }
}
```

---

# 3. Конфигурация через `@Configuration` и `@Bean`

## Полноценные конфигурационные классы vs «простые» компоненты; `proxyBeanMethods=true/false` и межбиновые вызовы

`@Configuration` — это особый компонент. По умолчанию (`proxyBeanMethods=true`) Spring проксирует его классом CGLIB, чтобы межбиновые вызовы (внутри этого конфиг-класса) проходили через контейнер и возвращали **singleton** из контекста, а не создавали новый объект прямым вызовом метода.

Если поставить `proxyBeanMethods=false` (как делает Spring Boot в большинстве автоконфигураций), вы отключите межбиновые прокси. Это ускоряет старт и убирает оверхед, но означает: вызов `this.someBean()` внутри конфигурации создаст **новый объект**, а не возьмёт его из контейнера. Поэтому такие конфиги пишут так, чтобы **не вызывать** свои `@Bean`-методы друг из друга.

«Простой» `@Component` с методами `@Bean` — допустим, но это уже не «полноценный конфиг»: межбиновые вызовы там никогда не будут проксироваться. Для инфраструктурной настройки лучше использовать `@Configuration` и чётко придерживаться семантики.

Когда выбирать что? Если в конфиге есть зависимости между `@Bean`-методами и вы хотите гарантировать один и тот же инстанс — оставляйте `proxyBeanMethods=true`. Если конфиг прост и вы не вызываете `@Bean`-методы друг из друга, смело ставьте `false` для скорости.

Командная договорённость важна: не вызывать `@Bean`-методы напрямую как «фабрику». Если нужна фабрика — вынесите её в отдельный класс или используйте `ObjectProvider`.

И помните: читаемость важнее микрооптимизаций. Сначала правильная семантика, затем — отключение прокси, когда это безопасно и понятно.

**Код (Java): отличие proxyBeanMethods)**

```java
package config.proxy;

import org.springframework.context.annotation.*;

@Configuration(proxyBeanMethods = true)
class ConfigWithProxy {
  @Bean String a() { return "A"; }
  @Bean String b() { return a(); } // вернёт singleton "A" через прокси
}

@Configuration(proxyBeanMethods = false)
class ConfigNoProxy {
  @Bean String x() { return "X"; }
  @Bean String y() { return x(); } // создаст НОВЫЙ объект, не из контейнера
}
```

**Код (Kotlin): аналог)**

```kotlin
package config.proxy

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration(proxyBeanMethods = true)
class ConfigWithProxy {
    @Bean fun a(): String = "A"
    @Bean fun b(): String = a() // вернёт singleton через прокси
}

@Configuration(proxyBeanMethods = false)
class ConfigNoProxy {
    @Bean fun x(): String = "X"
    @Bean fun y(): String = x() // создаст новый объект
}
```

---

## Семантика `@Bean`: скоуп, `initMethod/destroyMethod`, видимость и зависимость

Аннотация `@Bean` регистрирует бин с именем метода и классом возвращаемого типа. По умолчанию — `singleton`. Можно задать `@Scope("prototype")` для прототипов или веб-скоупы (в веб-приложениях). Это влияет на то, когда создаётся объект и кто его уничтожает.

`initMethod` и `destroyMethod` позволяют привязать методы жизненного цикла, если вы не используете аннотации `@PostConstruct`/`@PreDestroy` (например, библиотечный класс, который нельзя аннотировать). Контейнер вызовет их после внедрения зависимостей и перед уничтожением.

Видимость бина определяется «контекстом»: внутри одного `ApplicationContext` имя уникально. Если конфигураций несколько, понимаем, в каком контексте бин зарегистрирован. Это важно для тестов и иерархии контекстов.

Зависимости `@Bean` выражаются параметрами метода. Если метод принимает аргументы, контейнер найдёт подходящие бины и подставит их. Это читаемо и упрощает wiring инфраструктуры.

С `@Bean` удобно конфигурировать сторонние клиенты: вы получаете централизованную точку настройки (таймауты, ретраи, TLS), и один клиент переиспользуют многие сервисы. Это лучше, чем множество `new` с разъехавшимися опциями.

Наконец, помните про имена: `@Bean(name="httpClient")` делает ключ явным. Это полезно, если у вас несколько HTTP-клиентов под разные цели.

**Код (Java): `@Bean` со скоупом и init/destroy)**

```java
package config.bean;

import org.springframework.context.annotation.*;
import java.time.Clock;

@Configuration
class Beans {
  @Bean(initMethod = "start", destroyMethod = "stop")
  @Scope("singleton")
  Worker worker(Clock clock) { return new Worker(clock); }

  @Bean Clock clock() { return Clock.systemUTC(); }
}

class Worker {
  private final Clock clock;
  Worker(Clock clock) { this.clock = clock; }
  public void start() { /* init */ }
  public void stop() { /* destroy */ }
}
```

**Код (Kotlin): то же)**

```kotlin
package config.bean

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Scope
import java.time.Clock

@Configuration
class Beans {
    @Bean(initMethod = "start", destroyMethod = "stop")
    @Scope("singleton")
    fun worker(clock: Clock) = Worker(clock)

    @Bean fun clock(): Clock = Clock.systemUTC()
}

class Worker(private val clock: Clock) {
    fun start() { /* init */ }
    fun stop() { /* destroy */ }
}
```

---

## Method injection (`@Lookup`), вызовы `@Bean`-методов, почему нельзя подменять их прямыми вызовами как «фабрику»

Иногда синглтон-бин должен на каждый запрос получать **новый экземпляр** прототипа. Делать `new` в коде — нарушать DI. Для этого существует **method injection**: пометьте абстрактный/виртуальный метод `@Lookup`, и контейнер подменит его реализацию, возвращая свежий бин из контекста.

`@Lookup` устраняет привязку к контейнеру в бизнес-коде: вы вызываете метод как обычный, а фактически получаете `getBean` под капотом. Это удобно для «одноразовых» объектов: команд, билдеров, временных буферов.

Важно не путать `@Lookup` с вызовом `@Bean`-метода напрямую. Вызов `someConfig.someBean()` — **не** «получить бин из контейнера», особенно если `proxyBeanMethods=false`. Это обычный метод Java, который может вернуть новый объект и обойти инфраструктуру Spring.

Правильный способ получить другой бин — через инъекцию в конструкторе, через параметры `@Bean`-метода или через `ObjectProvider`. `@Lookup` — частный приём для прототипов, когда инъекция не подходит.

Понимание разницы экономит часы отладки: многие ошибки «почему у меня два разных экземпляра?» происходят из-за прямых вызовов `@Bean`-методов и отключённых прокси конфигурации.

И ещё: злоупотреблять `@Lookup` не стоит. Если он «нужен везде», это сигнал о дизайне: возможно, стоит выделить фабрику или пересмотреть скоупы.

**Код (Java): `@Lookup` для прототипа)**

```java
package config.lookup;

import org.springframework.beans.factory.annotation.Lookup;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component @Scope("prototype")
class Task { String id() { return java.util.UUID.randomUUID().toString(); } }

@Component
abstract class TaskRunner {
  public String run() { return newTask().id(); }

  @Lookup
  protected abstract Task newTask(); // Spring сгенерирует реализацию: getBean(Task.class)
}
```

**Код (Kotlin): `@Lookup` для прототипа)**

```kotlin
package config.lookup

import org.springframework.beans.factory.annotation.Lookup
import org.springframework.context.annotation.Scope
import org.springframework.stereotype.Component
import java.util.UUID

@Component
@Scope("prototype")
class Task { fun id(): String = UUID.randomUUID().toString() }

@Component
abstract class TaskRunner {
    fun run(): String = newTask().id()

    @Lookup
    protected abstract fun newTask(): Task
}
```

# 0. Введение

## Что это

Бин (Bean) — это управляемый контейнером Spring объект с известной идентичностью, скоупом и жизненным циклом. Вокруг него существует «паспорт» — `BeanDefinition`, где описано, как бин создаётся, какие у него зависимости, какой у него скоуп и когда он должен быть уничтожен. Когда мы говорим «внедрение зависимостей», на практике это означает: контейнер создаёт бины, связывает их в граф и подставляет в конструкторы других бинов.

Важный нюанс: бин — это не просто «любой объект». Он «родился» от контейнера (через сканирование компонентов или конфигурацию), поэтому на него распространяются инфраструктурные механизмы Spring: пост-процессоры, аспекты, транзакции, кеширование, безопасность, метрики. Если вы создадите тот же класс через `new`, это будет уже не бин, и многие «магии» не сработают.

Spring Boot упрощает жизнь тем, что автоконфигурирует контейнер: соберёт стандартные бины (веб-слой, Jackson, валидатор, DataSource и т.д.) из стартеров и включит их по условиям. Но принцип остаётся тем же: это объект, управляемый контекстом, который «знает» про его создание и смерть.

Бины связаны зависимостями. Чаще всего мы используем конструкторную инъекцию: класс объявляет, что ему нужно, и контейнер подставляет подходящие бины по типу (или по квалификатору). Это делает код слабосвязанным и заменяемым: реализацию можно переключать профилями/свойствами.

Граф бинов — это сердцебиение приложения. По нему видно архитектуру: где домен, где адаптеры, где инфраструктура. Хороший проект читается по конструкторам: заглянул в сервис — сразу понятно, что он требует на вход и какими портами пользуется.

И, наконец, «бин» — это единица расширения. Пост-процессоры могут оборачивать его прокси, добавлять кросс-срезы (логирование, метрики), менять поведение без правки бизнес-кода. Это фундаментальная причина, почему стоит «играть по правилам» контейнера.

**Код (Java): минимальная программа с бином**

```java
package app.intro;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@SpringBootApplication
public class IntroApp {
  public static void main(String[] args) { SpringApplication.run(IntroApp.class, args); }
}

interface Greeter { String hello(String name); }

@Component
class SimpleGreeter implements Greeter {
  @Override public String hello(String name) { return "Hello, " + name; }
}

@Service
class WelcomeService {
  private final Greeter greeter;
  public WelcomeService(Greeter greeter) { this.greeter = greeter; }
  public String welcome(String name) { return greeter.hello(name); }
}
```

**Код (Kotlin): тот же пример**

```kotlin
package app.intro

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

@SpringBootApplication
class IntroApp
fun main(args: Array<String>) = runApplication<IntroApp>(*args)

interface Greeter { fun hello(name: String): String }

@Component
class SimpleGreeter : Greeter {
    override fun hello(name: String) = "Hello, $name"
}

@Service
class WelcomeService(private val greeter: Greeter) {
    fun welcome(name: String) = greeter.hello(name)
}
```

Gradle (Groovy):

```groovy
plugins { id 'org.springframework.boot' version '3.3.4'; id 'io.spring.dependency-management' version '1.1.6'; id 'java' }
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

Gradle (Kotlin):

```kotlin
plugins { id("org.springframework.boot") version "3.3.4"; id("io.spring.dependency-management") version "1.1.6"; kotlin("jvm") version "1.9.25"; kotlin("plugin.spring") version "1.9.25" }
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

## Зачем это

Контейнер бинов снимает с разработчика рутину создания и связывания объектов. Вы декларируете роли классов, а инфраструктура берёт на себя порядок инициализации, внедрение зависимостей, проксирование и жизненный цикл. Это ускоряет разработку и уменьшает количество ошибок «порядка старта».

Бины повышают тестируемость. Конструкторная инъекция означает, что вы легко подменяете зависимости фейками/моками и тестируете бизнес-логику без реальных интеграций. Для интеграционных тестов контейнер поднимет «настоящие» бины.

Гибкость конфигурации — ещё одна причина. Один и тот же класс может стать разным бином в разных профилях: в `dev` — фейковый клиент, в `prod` — реальный с TLS, трейсингом и пулом. Переключение — через свойства, а не через `if`-ы в коде.

На бины удобно навешивать кросс-срезы: транзакции, кеширование, ретраи, метрики. Это делается «снаружи» через прокси/аспекты, а не «вшивается» в бизнес-методы. Код остаётся чистым, а поведение — расширяемым.

Жизненный цикл под контролем. Контейнер вызовет `@PostConstruct` для инициализации, `@PreDestroy` для освобождения ресурсов, гарантирует «мягкую» остановку по сигналу. Вы меньше думаете о служебных деталях и сосредотачиваетесь на домене.

Наконец, бины — это прозрачность. По именам и типам видно, какие части системы активны, а по отчётам Actuator можно понять, что именно поднято и как сконфигурировано.

**Код (Java): профили — зачем и как)**

```java
package app.why;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

interface Mailer { void send(String to, String body); }

@Component @Profile("dev")
class ConsoleMailer implements Mailer {
  public void send(String to, String body) { System.out.println("DEV mail: " + to + " -> " + body); }
}

@Component @Profile("prod")
class SmtpMailer implements Mailer {
  public void send(String to, String body) { /* SMTP call */ }
}
```

**Код (Kotlin): профили — аналог**

```kotlin
package app.why

import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

interface Mailer { fun send(to: String, body: String) }

@Component @Profile("dev")
class ConsoleMailer : Mailer {
    override fun send(to: String, body: String) = println("DEV mail: $to -> $body")
}

@Component @Profile("prod")
class SmtpMailer : Mailer {
    override fun send(to: String, body: String) { /* SMTP */ }
}
```

---

## Где используется

Бины — везде, где есть слои приложения: контроллеры (`@RestController`), сервисы (`@Service`), репозитории (`@Repository`), клиенты внешних систем (`@Component` + конфигурация), конфиг-классы (`@Configuration`), слушатели (`@EventListener`). Это единая модель для веба, задач и реактивного стека.

В интеграциях бины представляют «порты и адаптеры». Порт — интерфейс домена, адаптер — конкретная реализация (например, HTTP-клиент). Контейнер «подставит» нужный адаптер в сервис. Так домен изолирован от технологий и легче тестируется.

Во входном слое бины формируют маршрутизацию: фильтры, интерсепторы, контроллеры — все они бины, и на них легко навешиваются кросс-срезы (метрики, трейсинг). Это делает поведение централизованно управляемым.

В хранилищах бины управляют ресурсами: пул соединений — бин, `EntityManagerFactory` — бин, репозитории — бины. Контейнер закрывает их корректно на остановке, а в рантайме умеет инспектировать состояние.

В задачах и «воркерах» бины играют ту же роль: объявляете обработчик, читаете из Kafka, пишете в БД — всё это управляемые объекты с понятной жизнью. То же касается batch-джобов и планировщиков.

И, наконец, в тестах: спринговые «срезы» поднимают ровно те бины, что нужны для сценария (`@WebMvcTest`, `@DataJpaTest`), экономя время и упрощая диагностику.

**Код (Java): контроллер + сервис + репозиторий как бины)**

```java
package app.where;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.web.bind.annotation.*;
import org.springframework.stereotype.*;

@Entity class User { @Id public String id; public String name; }

interface UserRepo extends JpaRepository<User, String> {}

@Service
class UserService {
  private final UserRepo repo;
  UserService(UserRepo repo) { this.repo = repo; }
  public User create(String id, String name) { var u = new User(); u.id=id; u.name=name; return repo.save(u); }
}

@RestController
@RequestMapping("/users")
class UserController {
  private final UserService service;
  UserController(UserService service) { this.service = service; }
  @PostMapping public User create(@RequestParam String id, @RequestParam String name) { return service.create(id, name); }
}
```

**Код (Kotlin): аналогичное расположение бинов)**

```kotlin
package app.where

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Service
import org.springframework.web.bind.annotation.*
import jakarta.persistence.Entity
import jakarta.persistence.Id

@Entity class User(@Id var id: String = "", var name: String = "")

interface UserRepo : JpaRepository<User, String>

@Service
class UserService(private val repo: UserRepo) {
    fun create(id: String, name: String) = repo.save(User(id, name))
}

@RestController
@RequestMapping("/users")
class UserController(private val service: UserService) {
    @PostMapping fun create(@RequestParam id: String, @RequestParam name: String) = service.create(id, name)
}
```

---

## Какие проблемы/задачи решает

Контейнер решает проблему «кто создаёт кого и когда». Без него порядок старта и зависимости между объектами часто оказываются неявными. С контейнером у нас есть регистрация дефиниций, проверка конфликтов, предсказуемое создание инициализированных синглтонов.

Вторая задача — ресурсная дисциплина. Пулы соединений, планировщики, клиентские SDK должны корректно стартовать и завершаться. Контейнер вызывает `@PostConstruct`/`@PreDestroy`, закрывает ресурсы, останавливает задачи. Мы не теряем дескрипторы и не «роняем» JVM при остановке.

Третья — сквозные политики. Транзакции, кеширование, ретраи, безопасность включаются через AOP/прокси на уровне бинов. Не нужно дублировать одно и то же во всех классах. Проще управлять и проверять.

Четвёртая — конфигурируемость. Все параметры («проволока») — во внешних настройках; один образ — много окружений. Автоконфигурации подключают нужные бины при наличии классов и свойств.

Пятая — наблюдаемость. Через пост-процессоры и автоконфигурации легко включить метрики/трейсинг на все бины нужного типа, не трогая бизнес-код. Это экономит недели инженерного времени.

Шестая — безопасность изменений. Конфликты бинов, циклы зависимостей и ошибки wiring’а ловятся при старте. Чем раньше ошибка, тем дешевле её исправить.

**Код (Java): инициализация/завершение ресурса)**

```java
package app.problems;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class ConnectionHolder {
  @PostConstruct void init() { /* open connections / warm-up */ }
  @PreDestroy void shutdown() { /* close connections */ }
}
```

**Код (Kotlin): то же)**

```kotlin
package app.problems

import jakarta.annotation.PostConstruct
import jakarta.annotation.PreDestroy
import org.springframework.stereotype.Component

@Component
class ConnectionHolder {
    @PostConstruct fun init() { /* open connections / warm-up */ }
    @PreDestroy fun shutdown() { /* close connections */ }
}
```

---

# 1. Что такое бин и откуда он берётся

## BeanDefinition: метаданные о типе, скоупе, зависимостях, способе создания

Внутри контейнера каждый бин представлен `BeanDefinition` — структурой с метаданными: какой класс создавать, какие конструкторные аргументы подставить, какой скоуп (singleton/prototype/веб-скоуп), какие методы инициализации/уничтожения вызвать. Эти метаданные появляются из аннотаций, из `@Bean`-методов или из ручной регистрации.

`BeanDefinition` — это причина, по которой контейнер может «переигрывать» создание: менять скоуп, оборачивать прокси, добавлять зависимости, настраивать `autowire mode`. Благодаря этому пост-процессоры способны видоизменять граф до фактического создания объектов.

С практической точки зрения разработчик редко строит `BeanDefinition` руками, но полезно знать, что он существует, и уметь его читать. Это помогает при диагностике: почему бин прототип? почему у него такое имя? почему на него «нацепилась» транзакция?

Доступ к реестру дефиниций возможен через `BeanDefinitionRegistry` и `BeanFactory`. В пост-процессорах можно инспектировать и даже программно регистрировать новые бины. Так пишутся «автоконфиги» и плагины.

Важно понимать, что `BeanDefinition` — декларативная «паспортная» запись. Пока бин не создан, вы можете менять дефиницию. После создания синглтона менять его «на лету» неправильно; для этого существуют прокси/декораторы.

Наконец, `BeanDefinition` — не про бизнес-код, а про «проволоку». В рамках команды достаточно, чтобы 1–2 инженера глубоко понимали эти механизмы для сложных кейсов; остальным полезно знать «куда смотреть».

**Код (Java): вывод базовой информации о дефинициях)**

```java
package beans.defs;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.stereotype.Component;

@Component
class DefPrinter implements BeanFactoryPostProcessor {
  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) throws BeansException {
    for (String name : bf.getBeanDefinitionNames()) {
      var bd = bf.getBeanDefinition(name);
      System.out.println(name + " -> " + bd.getBeanClassName() + ", scope=" + bd.getScope());
    }
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package beans.defs

import org.springframework.beans.BeansException
import org.springframework.beans.factory.config.BeanFactoryPostProcessor
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory
import org.springframework.stereotype.Component

@Component
class DefPrinter : BeanFactoryPostProcessor {
    @Throws(BeansException::class)
    override fun postProcessBeanFactory(bf: ConfigurableListableBeanFactory) {
        bf.beanDefinitionNames.forEach { name ->
            val bd = bf.getBeanDefinition(name)
            println("$name -> ${bd.beanClassName}, scope=${bd.scope}")
        }
    }
}
```

---

## Источники бинов: компонент-сканирование (`@Component`-семейство), Java-конфигурация (`@Configuration` + `@Bean`), импорт (`@Import`, `@ImportSelector`), фабрики (`FactoryBean`)

Самый частый источник — компонент-сканирование. Вы помечаете класс `@Component`/`@Service`/`@Repository`/`@Controller`, Spring сканирует пакеты и регистрирует дефиниции. Это удобно для «обычных» сервисов/адаптеров.

Java-конфигурация — второй способ. `@Configuration`-класс объявляет методы-фабрики `@Bean`, каждый из которых регистрирует бин. Это идеально для инфраструктуры: пула соединений, HTTP-клиентов, кэшей. Так код настройки лежит рядом, его проще читать и тестировать.

`@Import` подключает дополнительные конфигурации/классы. Через `@ImportSelector` можно программно вернуть список классов к импорту по условиям (как делают автоконфигурации Spring Boot). Это мощная точка расширения, позволяющая строить «стартеры».

`FactoryBean<T>` — особый контракт: сам класс — не «тот» бин, а фабрика, которая производит бин типа `T`. Это полезно, когда создание сложное: нужно прочитать секреты, собрать клиент по билдеру, выбрать реализацию в рантайме. Контейнер узнаёт, что отдавать из контекста: `T` по имени, или `FactoryBean` по имени с префиксом `&`.

Импорт и фабрики позволяют держать код переиспользуемым. Библиотека может предоставить «набор» бинов и подключать их через условия/свойства. Приложение — решает, включать ли их.

Важно не смешивать подходы хаотично. Обычная практика: бизнес-код — через сканирование, инфраструктура/клиенты — через `@Bean`, подключаемые модули — через `@Import`.

**Код (Java): все источники сразу — примеры)**

```java
package beans.sources;

import org.springframework.context.annotation.*;
import org.springframework.stereotype.Component;

// 1) Компонент-сканирование
@Component
class PingService { public String ping() { return "pong"; } }

// 2) Java-конфигурация
@Configuration
class HttpConfig {
  @Bean java.net.http.HttpClient httpClient() { return java.net.http.HttpClient.newHttpClient(); }
}

// 3) Импорт конфигурации
@Configuration @Import(HttpConfig.class)
class RootConfig {}

// 4) FactoryBean
class Client { final String who; Client(String who) { this.who = who; } }
class ClientFactory implements FactoryBean<Client> {
  @Override public Client getObject() { return new Client("factory"); }
  @Override public Class<?> getObjectType() { return Client.class; }
}
@Configuration
class ClientConfig {
  @Bean ClientFactory client() { return new ClientFactory(); }
}
```

**Код (Kotlin): те же идеи)**

```kotlin
package beans.sources

import org.springframework.context.annotation.*
import org.springframework.stereotype.Component
import java.net.http.HttpClient

@Component
class PingService { fun ping() = "pong" }

@Configuration
class HttpConfig {
    @Bean fun httpClient(): HttpClient = HttpClient.newHttpClient()
}

@Configuration
@Import(HttpConfig::class)
class RootConfig

class Client(val who: String)
class ClientFactory : FactoryBean<Client> {
    override fun getObject(): Client = Client("factory")
    override fun getObjectType(): Class<*> = Client::class.java
}
@Configuration
class ClientConfig {
    @Bean fun client(): ClientFactory = ClientFactory()
}
```

---

## Именование и алиасы: каноническое имя, коллизии, правила чтения «дерева» бинов

Каждый бин имеет **каноническое имя** — строковый идентификатор в контейнере. Для компонентов по умолчанию это «camelCase-имя класса» (например, `simpleGreeter`). Для `@Bean` имя — это имя метода. Можно задавать своё имя явным образом.

Алиасы — дополнительные имена, по которым бин можно получить. Они удобны, когда один объект должен быть доступен под несколькими ролями/ключами. В `@Bean` можно указать массив имён. На уровне реестра возможна программная регистрация алиасов.

Коллизии имён приводят к перезаписи или ошибкам — в зависимости от настроек. Хорошая практика — единый стиль именования и отсутствие «магических» совпадений. Для множества реализаций используйте осмысленные имена и `@Qualifier`.

«Дерево» бинов — это то, как они зависимы. Смотреть на имена полезно для диагностики: по логам видно, какой конкретно бин выбрался. А в картах стратегий (`Map<String,Strategy>`) ключом именно служит имя бина — это удобно.

Стоит помнить про `FactoryBean`: по имени «client» вы получите `Client`, а по имени «&client» — саму фабрику. Это часто удивляет, когда пытаются залогировать «тип бина» и получают фабрику вместо продукта.

Стандартизируйте имена: `emailNotifier`, `smsNotifier`, `kafkaConsumerOrders`, `pricingHttpClient`. Так у вас будет единый, читаемый словарь.

**Код (Java): имена и алиасы)**

```java
package beans.names;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

@Component("emailNotifier")
class EmailNotifier {}

@Configuration
class NamesConfig {
  @Bean(name = {"mainClient","aliasClient"})
  String clientName() { return "X"; }
}
```

**Код (Kotlin): имена и алиасы)**

```kotlin
package beans.names

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.stereotype.Component

@Component("emailNotifier")
class EmailNotifier

@Configuration
class NamesConfig {
    @Bean(name = ["mainClient","aliasClient"])
    fun clientName(): String = "X"
}
```

---

# 2. Жизненный цикл бина (от регистрации до уничтожения)

## Фазы старта контекста: регистрация дефиниций → пост-процессоры → создание singleton-бинов → вызовы колбэков → `ApplicationReady`

Старт начинается с **регистрации дефиниций**: сканирование, чтение `@Configuration` и `@Bean`, подключение импортов. На этом этапе граф ещё «плоский», это просто записи в реестре: какие бины потенциально есть.

Затем запускаются **пост-процессоры фабрики** (`BeanFactoryPostProcessor`/`BeanDefinitionRegistryPostProcessor`). Они могут модифицировать дефиниции: менять скоуп, добавлять свойства, регистрировать новые бины. Это последняя точка, где можно «перекроить» план.

Дальше контейнер переходит к **созданию singleton-бинов** (eager by default). Он вычисляет зависимости, строит объекты, внедряет поля/сеттеры/конструкторные аргументы, применяет `BeanPostProcessor` до/после инициализации. Прототипы по умолчанию не создаются на старте.

После создания всех синглтонов вызываются «глобальные» хуки вроде `SmartInitializingSingleton`: это шанс выполнить код «после того, как все синглтоны готовы». Полезно для проверок целостности и проволоки.

Spring Boot затем публикует события `ApplicationStartedEvent` и `ApplicationReadyEvent`. Последнее означает, что приложение готово принимать трафик. Это удобная точка для прогрева кэшей, показа баннера готовности и старта фоновых процессов (осторожно).

Важно не перегружать старт: долгие задачи лучше вынести в отдельные джобы/инициаторы или выполнять асинхронно. Иначе вы получите большие `startup time` и сложную диагностику.

**Код (Java): слушатель событий старта)**

```java
package lifecycle.phases;

import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
class ReadyListener {
  @EventListener(ApplicationReadyEvent.class)
  public void onReady() { System.out.println("Application is READY"); }
}
```

**Код (Kotlin): слушатель `ApplicationReady`)**

```kotlin
package lifecycle.phases

import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component

@Component
class ReadyListener {
    @EventListener(ApplicationReadyEvent::class)
    fun onReady() = println("Application is READY")
}
```

---

## Инициализация бина: внедрение зависимостей → `Aware`-интерфейсы → `@PostConstruct`/`afterPropertiesSet`/init-method → пост-процессоры (`BeanPostProcessor`)

Когда контейнер создаёт экземпляр, он сначала **внедряет зависимости** (конструктор/сеттеры/поля). На этом этапе бин уже «собран», но ещё не инициализирован логически.

Затем вызываются **`Aware`-интерфейсы** (если реализованы): `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`. Это способ получить ссылку на инфраструктуру. В бизнес-коде их лучше избегать, чтобы не скатываться в Service Locator.

Далее — **инициализация**. Если есть `@PostConstruct`, он будет вызван. Если бин реализует `InitializingBean`, вызовется `afterPropertiesSet()`. Если у `@Bean` указан `initMethod`, он тоже вызовется. Обычно используют `@PostConstruct` как самый нейтральный способ.

Параллельно работают **пост-процессоры бинов** (`BeanPostProcessor`): метод `postProcessBeforeInitialization` может заменить бин до инициализации, а `postProcessAfterInitialization` — обернуть прокси/декоратор. Именно здесь включаются транзакции, кеширование, AOP.

Инициализация должна быть быстрой и безопасной. Не запускайте тяжёлые задачи в `@PostConstruct`, не блокируйте поток старта. Для прогрева используйте `ApplicationReadyEvent`/отдельные механизмы.

Помните, что порядок: зависимости → Aware → init → postProcessors(после init). Это помогает диагностировать «почему логика не сработала»: иногда дело в том, что вы сделали self-invocation до появления прокси.

**Код (Java): все стадии инициализации)**

```java
package lifecycle.init;

import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.stereotype.Component;

@Component
class InitDemo implements BeanNameAware, InitializingBean {
  private String name;
  @Override public void setBeanName(String name) { this.name = name; }

  @PostConstruct
  void post() { System.out.println("PostConstruct for " + name); }

  @Override
  public void afterPropertiesSet() { System.out.println("afterPropertiesSet for " + name); }
}
```

**Код (Kotlin): то же)**

```kotlin
package lifecycle.init

import jakarta.annotation.PostConstruct
import org.springframework.beans.factory.BeanNameAware
import org.springframework.beans.factory.InitializingBean
import org.springframework.stereotype.Component

@Component
class InitDemo : BeanNameAware, InitializingBean {
    private var name: String? = null
    override fun setBeanName(name: String) { this.name = name }

    @PostConstruct fun post() = println("PostConstruct for $name")
    override fun afterPropertiesSet() = println("afterPropertiesSet for $name")
}
```

---

## Завершение: `@PreDestroy`/`DisposableBean`/destroy-method, порядок остановки

При остановке приложения контейнер корректно завершает жизненный цикл бинов. Сначала публикуется `ContextClosedEvent`, затем для каждого бина вызываются колбэки разрушения: `@PreDestroy`, `DisposableBean.destroy()`, `destroyMethod` из `@Bean`.

На этапе завершения важно освободить ресурсы: закрыть соединения, остановить планировщики, слить буферы. Это помогает избежать утечек и «подвешенных» потоков, мешающих JVM корректно выйти.

Порядок остановки можно контролировать через `SmartLifecycle` и фазы. Компоненты с меньшим номером фаз останавливаются позже/раньше (в зависимости от логики), что позволяет аккуратно выключать потребителей прежде, чем отключать брокера/ресурсы.

Если вы используете «короткоживущие» скоупы (request/session), их объекты уничтожаются вместе со сессией/запросом. Но большинство инфраструктурных ресурсов — синглтоны, поэтому `@PreDestroy` — ваш друг.

Важно, чтобы уничтожение было **идемпотентно**. Колбэки могут вызываться повторно при некоторых сценариях (например, перезапуске контекста в тестах). Защитите операции от дублей.

В микросервисах корректная остановка особенно заметна: брокеры/балансировщики меньше «ругаются» на разорванные соединения, нагрузочные пики при rolling-update снижаются.

**Код (Java): завершение ресурсов)**

```java
package lifecycle.destroy;

import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class ResourceHolder {
  @PreDestroy
  void shutdown() {
    // close pools, stop schedulers
    System.out.println("Resources closed");
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package lifecycle.destroy

import jakarta.annotation.PreDestroy
import org.springframework.stereotype.Component

@Component
class ResourceHolder {
    @PreDestroy
    fun shutdown() {
        println("Resources closed")
    }
}
```

---

# 3. Конфигурация через `@Configuration` и `@Bean`

## Полноценные конфигурационные классы vs «простые» компоненты; `proxyBeanMethods=true/false` и межбиновые вызовы

`@Configuration` — это особый компонент. По умолчанию (`proxyBeanMethods=true`) Spring проксирует его классом CGLIB, чтобы межбиновые вызовы (внутри этого конфиг-класса) проходили через контейнер и возвращали **singleton** из контекста, а не создавали новый объект прямым вызовом метода.

Если поставить `proxyBeanMethods=false` (как делает Spring Boot в большинстве автоконфигураций), вы отключите межбиновые прокси. Это ускоряет старт и убирает оверхед, но означает: вызов `this.someBean()` внутри конфигурации создаст **новый объект**, а не возьмёт его из контейнера. Поэтому такие конфиги пишут так, чтобы **не вызывать** свои `@Bean`-методы друг из друга.

«Простой» `@Component` с методами `@Bean` — допустим, но это уже не «полноценный конфиг»: межбиновые вызовы там никогда не будут проксироваться. Для инфраструктурной настройки лучше использовать `@Configuration` и чётко придерживаться семантики.

Когда выбирать что? Если в конфиге есть зависимости между `@Bean`-методами и вы хотите гарантировать один и тот же инстанс — оставляйте `proxyBeanMethods=true`. Если конфиг прост и вы не вызываете `@Bean`-методы друг из друга, смело ставьте `false` для скорости.

Командная договорённость важна: не вызывать `@Bean`-методы напрямую как «фабрику». Если нужна фабрика — вынесите её в отдельный класс или используйте `ObjectProvider`.

И помните: читаемость важнее микрооптимизаций. Сначала правильная семантика, затем — отключение прокси, когда это безопасно и понятно.

**Код (Java): отличие proxyBeanMethods)**

```java
package config.proxy;

import org.springframework.context.annotation.*;

@Configuration(proxyBeanMethods = true)
class ConfigWithProxy {
  @Bean String a() { return "A"; }
  @Bean String b() { return a(); } // вернёт singleton "A" через прокси
}

@Configuration(proxyBeanMethods = false)
class ConfigNoProxy {
  @Bean String x() { return "X"; }
  @Bean String y() { return x(); } // создаст НОВЫЙ объект, не из контейнера
}
```

**Код (Kotlin): аналог)**

```kotlin
package config.proxy

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration(proxyBeanMethods = true)
class ConfigWithProxy {
    @Bean fun a(): String = "A"
    @Bean fun b(): String = a() // вернёт singleton через прокси
}

@Configuration(proxyBeanMethods = false)
class ConfigNoProxy {
    @Bean fun x(): String = "X"
    @Bean fun y(): String = x() // создаст новый объект
}
```

---

## Семантика `@Bean`: скоуп, `initMethod/destroyMethod`, видимость и зависимость

Аннотация `@Bean` регистрирует бин с именем метода и классом возвращаемого типа. По умолчанию — `singleton`. Можно задать `@Scope("prototype")` для прототипов или веб-скоупы (в веб-приложениях). Это влияет на то, когда создаётся объект и кто его уничтожает.

`initMethod` и `destroyMethod` позволяют привязать методы жизненного цикла, если вы не используете аннотации `@PostConstruct`/`@PreDestroy` (например, библиотечный класс, который нельзя аннотировать). Контейнер вызовет их после внедрения зависимостей и перед уничтожением.

Видимость бина определяется «контекстом»: внутри одного `ApplicationContext` имя уникально. Если конфигураций несколько, понимаем, в каком контексте бин зарегистрирован. Это важно для тестов и иерархии контекстов.

Зависимости `@Bean` выражаются параметрами метода. Если метод принимает аргументы, контейнер найдёт подходящие бины и подставит их. Это читаемо и упрощает wiring инфраструктуры.

С `@Bean` удобно конфигурировать сторонние клиенты: вы получаете централизованную точку настройки (таймауты, ретраи, TLS), и один клиент переиспользуют многие сервисы. Это лучше, чем множество `new` с разъехавшимися опциями.

Наконец, помните про имена: `@Bean(name="httpClient")` делает ключ явным. Это полезно, если у вас несколько HTTP-клиентов под разные цели.

**Код (Java): `@Bean` со скоупом и init/destroy)**

```java
package config.bean;

import org.springframework.context.annotation.*;
import java.time.Clock;

@Configuration
class Beans {
  @Bean(initMethod = "start", destroyMethod = "stop")
  @Scope("singleton")
  Worker worker(Clock clock) { return new Worker(clock); }

  @Bean Clock clock() { return Clock.systemUTC(); }
}

class Worker {
  private final Clock clock;
  Worker(Clock clock) { this.clock = clock; }
  public void start() { /* init */ }
  public void stop() { /* destroy */ }
}
```

**Код (Kotlin): то же)**

```kotlin
package config.bean

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Scope
import java.time.Clock

@Configuration
class Beans {
    @Bean(initMethod = "start", destroyMethod = "stop")
    @Scope("singleton")
    fun worker(clock: Clock) = Worker(clock)

    @Bean fun clock(): Clock = Clock.systemUTC()
}

class Worker(private val clock: Clock) {
    fun start() { /* init */ }
    fun stop() { /* destroy */ }
}
```

---

## Method injection (`@Lookup`), вызовы `@Bean`-методов, почему нельзя подменять их прямыми вызовами как «фабрику»

Иногда синглтон-бин должен на каждый запрос получать **новый экземпляр** прототипа. Делать `new` в коде — нарушать DI. Для этого существует **method injection**: пометьте абстрактный/виртуальный метод `@Lookup`, и контейнер подменит его реализацию, возвращая свежий бин из контекста.

`@Lookup` устраняет привязку к контейнеру в бизнес-коде: вы вызываете метод как обычный, а фактически получаете `getBean` под капотом. Это удобно для «одноразовых» объектов: команд, билдеров, временных буферов.

Важно не путать `@Lookup` с вызовом `@Bean`-метода напрямую. Вызов `someConfig.someBean()` — **не** «получить бин из контейнера», особенно если `proxyBeanMethods=false`. Это обычный метод Java, который может вернуть новый объект и обойти инфраструктуру Spring.

Правильный способ получить другой бин — через инъекцию в конструкторе, через параметры `@Bean`-метода или через `ObjectProvider`. `@Lookup` — частный приём для прототипов, когда инъекция не подходит.

Понимание разницы экономит часы отладки: многие ошибки «почему у меня два разных экземпляра?» происходят из-за прямых вызовов `@Bean`-методов и отключённых прокси конфигурации.

И ещё: злоупотреблять `@Lookup` не стоит. Если он «нужен везде», это сигнал о дизайне: возможно, стоит выделить фабрику или пересмотреть скоупы.

**Код (Java): `@Lookup` для прототипа)**

```java
package config.lookup;

import org.springframework.beans.factory.annotation.Lookup;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component @Scope("prototype")
class Task { String id() { return java.util.UUID.randomUUID().toString(); } }

@Component
abstract class TaskRunner {
  public String run() { return newTask().id(); }

  @Lookup
  protected abstract Task newTask(); // Spring сгенерирует реализацию: getBean(Task.class)
}
```

**Код (Kotlin): `@Lookup` для прототипа)**

```kotlin
package config.lookup

import org.springframework.beans.factory.annotation.Lookup
import org.springframework.context.annotation.Scope
import org.springframework.stereotype.Component
import java.util.UUID

@Component
@Scope("prototype")
class Task { fun id(): String = UUID.randomUUID().toString() }

@Component
abstract class TaskRunner {
    fun run(): String = newTask().id()

    @Lookup
    protected abstract fun newTask(): Task
}
```

# 8. События и раннеры приложения

## События контекста: `ContextRefreshed`, `ApplicationStarted`, `ApplicationReady`, `ContextClosed`; обработка через `@EventListener`

Событийная модель Spring даёт безопасные «точки» в жизненном цикле приложения. Во время старта и остановки контекст публикует ряд событий: ранние (`ApplicationStarting`/`EnvironmentPrepared`), средние (`ApplicationStarted`), поздние (`ApplicationReady`), а при останове — `ContextClosed`. Каждое событие означает, что контейнер прошёл определённую фазу и обеспечил необходимые инварианты (например, «все singletons созданы»). Это удобно, чтобы **не прятать инициализацию в конструкторах**, а выполнять её там, где это ожидаемо и контролируемо.

На практике чаще всего используют два события: `ApplicationStartedEvent` и `ApplicationReadyEvent`. Первое гарантирует, что контекст поднят, но веб-сервер может ещё принимать трафик не полностью. Второе — что приложение «готово»; в Boot оно публикуется после полного старта веб-слоя. В продакшне тяжёлые прогревы (кэши, JIT-нагрев, подключение к сторонним системам) разумно запускать именно на `ApplicationReady`, чтобы readiness-проба сигнализировала «готов» только после цикла прогрева.

`ContextClosedEvent` — симметричная точка для остановки фоновых тасков, аккуратного завершения очередей, фиксации метрик. Это особенно важно в Kubernetes: ваши обработчики должны **успеть** отработать в периоде graceful-shutdown, пока Pod «сливает» трафик. Привязка к событию, а не к «финализаторам», делает это поведение предсказуемым.

Обрабатывать события удобно через `@EventListener`. Это декларативный способ подписки, без прямой зависимости от `ApplicationEventPublisher`. Метод-слушатель может быть синхронным (по умолчанию) или асинхронным, если включён `@EnableAsync` и метод помечен `@Async`. Важно помнить, что синхронные обработчики входят во время старта, — не блокируйте их долгими задачами.

События хороши своей «локальностью»: код инициализации лежит рядом с ответственным классом. Но не стоит превращать их в «скрытый канал» бизнес-общения; ограничения должны оставаться инфраструктурными. Для доменных событий используйте собственные публикации (`ApplicationEventPublisher`) или брокер сообщений.

Диагностируйте порядок и продолжительность обработчиков: логируйте начало/конец, измеряйте длительность. Если `ApplicationReady` занимает секунды из-за прогревов, это должно быть видно в логах старта и SLO.

**Код (Java): обработчики ключевых событий**

```java
package app.events;

import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.boot.context.event.ApplicationStartedEvent;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class LifecycleListeners {

  @EventListener(ApplicationStartedEvent.class)
  public void onStarted() {
    System.out.println("ApplicationStarted: context refreshed, starting web server...");
  }

  @EventListener(ApplicationReadyEvent.class)
  @Async // при включённом @EnableAsync выполнится вне основного потока
  public void onReady() {
    System.out.println("ApplicationReady: warm caches / pre-connect");
    // прогрев кэшей, «сухой» вызов внешних API, JIT-warmup
  }

  @EventListener(ContextClosedEvent.class)
  public void onClosed() {
    System.out.println("ContextClosed: flush buffers / stop schedulers");
  }
}
```

**Код (Kotlin): то же, с асинхронным прогревом**

```kotlin
package app.events

import org.springframework.boot.context.event.ApplicationReadyEvent
import org.springframework.boot.context.event.ApplicationStartedEvent
import org.springframework.context.event.ContextClosedEvent
import org.springframework.context.event.EventListener
import org.springframework.scheduling.annotation.Async
import org.springframework.stereotype.Component

@Component
class LifecycleListeners {

    @EventListener(ApplicationStartedEvent::class)
    fun onStarted() {
        println("ApplicationStarted: context refreshed, starting web server...")
    }

    @EventListener(ApplicationReadyEvent::class)
    @Async
    fun onReady() {
        println("ApplicationReady: warm caches / pre-connect")
    }

    @EventListener(ContextClosedEvent::class)
    fun onClosed() {
        println("ContextClosed: flush buffers / stop schedulers")
    }
}
```

Gradle (Groovy/Kotlin) — включаем `spring-boot-starter` и, для `@Async`, веб или отдельный `spring-context` + `spring-boot-starter`:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter")
}
```

`application.yml` (пример управления async-выполнением):

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 4
        max-size: 16
```

---

## `CommandLineRunner`/`ApplicationRunner`: инициализация данных, прогрев кэшей/соединений, порядок выполнения

Раннеры — простейший способ выполнить код **после старта** (после создания всех синглтонов). Класс реализует `CommandLineRunner` или `ApplicationRunner` и будет вызван на старте. Второй отличается более удобным доступом к аргументам (обёртка `ApplicationArguments` с парсингом опций). Это идеальное место для семян данных, очистки «грязных» состояний dev-окружения, локальной миграции артефактов.

Важно помнить про **порядок**. Если раннеров несколько, используйте `@Order` или интерфейс `Ordered`. Это делает инициализацию детерминированной: сначала «инфраструктура», затем «бизнес». Избегайте долговременных блокировок: раннер — синхронный, он удерживает фазу старта, и readiness-проба задержится.

Раннеры удобны в batch/CLI-приложениях, где сервис выполняет одну задачу и завершается. Вы можете собрать модуль, запускаемый в Kubernetes Job или GitLab Runner. В таком подходе раннер — фактический «main» предметного сценария; он читает параметры, собирает зависимости и запускает пайплайн.

В сервисах с веб-трафиком раннер подходит именно для **прогрева**. Например, выполнить первые «пустые» запросы к внешнему API, чтобы JIT/каналы/сертификаты прогрелись. Либо — загодя построить кэш часто используемых справочников, чтобы первые пользователи не платили эту цену.

Если вам нужен контроль/повторяемость, оберните логику в явный сервис «Warmup» и параметризуйте его; раннер лишь дернёт методы. Это облегчит тестирование и переиспользование инициализации в ручном режиме (например, админский эндпойнт «перегреть кэши»).

Добавляйте **идемпотентность**: повторный запуск раннера не должен ломать систему. Например, добавляйте «upsert» вместо insert, не удаляйте данные без явной команды, логируйте, что именно сделано.

**Код (Java): раннеры и порядок**

```java
package app.runners;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(10)
class InfraWarmupRunner implements CommandLineRunner {
  @Override public void run(String... args) {
    System.out.println("Warmup: http clients / caches");
  }
}

@Component
@Order(20)
class DataSeedRunner implements ApplicationRunner {
  @Override public void run(ApplicationArguments args) {
    boolean seed = args.containsOption("seed");
    if (seed) {
      System.out.println("Seeding reference data...");
    }
  }
}
```

**Код (Kotlin): раннеры c `@Order`**

```kotlin
package app.runners

import org.springframework.boot.ApplicationArguments
import org.springframework.boot.ApplicationRunner
import org.springframework.boot.CommandLineRunner
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component

@Component
@Order(10)
class InfraWarmupRunner : CommandLineRunner {
    override fun run(vararg args: String) {
        println("Warmup: http clients / caches")
    }
}

@Component
@Order(20)
class DataSeedRunner : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        if (args.containsOption("seed")) {
            println("Seeding reference data...")
        }
    }
}
```

Запуск c опцией:

```
java -jar app.jar --seed
```

---

## `SmartLifecycle`: автозапуск, фазы и «корректная» остановка

Интерфейс `SmartLifecycle` даёт точный контроль над **автозапуском и остановкой** компонентов. В отличие от раннеров, `SmartLifecycle` интегрирован с фазами контейнера: можно указать порядок запуска/остановки через `getPhase()`; бины с меньшей фазой стартуют раньше и останавливаются позже (или наоборот, в зависимости от реализации), что удобно для **управляемой деградации**.

Типичный сценарий — потребители сообщений/планировщики. Нам нужно начать читать из Kafka **после** того, как прогрелись кэши и поднялись хранилища, и надо остановить чтение **до** того, как закроется пул соединений или выключится сетевой стек. `SmartLifecycle` решает эту хореографию. Ещё одна польза — явное `isAutoStartup()`: можно отключить автозапуск и стартовать компонент вручную (например, после health-проверок).

Методы `start()`/`stop()` должны быть быстрыми и идемпотентными. Если нужно дождаться корректного завершения фоновых задач, используйте перегрузку `stop(Runnable callback)` — вызовите callback после фактической остановки, чтобы контейнер знал, когда продолжать. Это критично для аккуратной остановки в Kubernetes, где у вас ограниченное окно (grace period).

`SmartLifecycle` хорошо дружит с событиями. На `ApplicationReady` вы можете выполнить лёгкий прогрев и лишь затем позволить автозапуску читателей. Либо — наоборот — запретить авто-старт (возвращать `false`), а в обработчике `ApplicationReady` стартовать вручную.

Следите за **фазами**. Распределите их по слоям: источники входящего трафика (слушатели) — поздние фазы; клиенты внешних систем — ранние; инфраструктура — самая ранняя. Документируйте схему, чтобы новые разработчики не ломали порядок.

И наконец, покрывайте остановку тестами. Создавайте контекст, закрывайте его и проверяйте, что ваши компоненты корректно отработали `stop()`. Это дёшево и экономит пот на проде.

**Код (Java): `SmartLifecycle` c фазами**

```java
package app.lifecycle;

import org.springframework.context.SmartLifecycle;
import org.springframework.stereotype.Component;

@Component
public class KafkaConsumerLifecycle implements SmartLifecycle {
  private volatile boolean running;

  @Override public boolean isAutoStartup() { return true; }
  @Override public void start() { running = true; System.out.println("Kafka consumer started"); }

  @Override public void stop(Runnable callback) {
    try {
      // drain / commit / close
      System.out.println("Kafka consumer stopping gracefully");
    } finally {
      running = false;
      callback.run();
    }
  }

  @Override public void stop() { stop(() -> {}); }

  @Override public boolean isRunning() { return running; }

  @Override public int getPhase() { return 100; } // стартует позже инфраструктуры
}
```

**Код (Kotlin): `SmartLifecycle`**

```kotlin
package app.lifecycle

import org.springframework.context.SmartLifecycle
import org.springframework.stereotype.Component

@Component
class KafkaConsumerLifecycle : SmartLifecycle {
    @Volatile private var running = false

    override fun isAutoStartup(): Boolean = true
    override fun start() { running = true; println("Kafka consumer started") }

    override fun stop(callback: Runnable) {
        try {
            println("Kafka consumer stopping gracefully")
        } finally {
            running = false
            callback.run()
        }
    }

    override fun stop() = stop(Runnable { })
    override fun isRunning(): Boolean = running
    override fun getPhase(): Int = 100
}
```

---

# 9. Автоконфигурация Spring Boot (обзор)

## Как «подцепляются» бины из стартеров: условия (`@Conditional*`), порядок, отключение/override

Магия стартеров в том, что они приносят **автоконфигурации** — классы с `@AutoConfiguration` (или `@Configuration` в старых версиях), которые регистрируются автоматически при наличии маркеров: класса на classpath (`@ConditionalOnClass`), пропертей (`@ConditionalOnProperty`), уже существующих бинов (`@ConditionalOnBean`) и т.д. Внутри такие конфиги создают бины через `@Bean`, при этом вежливо проверяют `@ConditionalOnMissingBean`, чтобы не «переписать» пользовательские.

Порядок подключения управляется зависимостями (`@AutoConfigureAfter/@AutoConfigureBefore`) и природой условий. Например, JPA-автоконфиг ждёт `DataSource`, web завязан на `DispatcherServlet` и т.д. За счёт условий Boot способен собрать «минимальный» граф ровно под ваш classpath.

Отключить автоконфигурацию можно глобально (`spring.autoconfigure.exclude`) или локально аннотацией `@SpringBootApplication(exclude = {...})`. Переопределить бин — достаточно объявить у себя такой же бин (тип/название), а автоконфиг должен был поставить `@ConditionalOnMissingBean` и уступить место. Это ключевая договорённость: библиотека — по умолчанию, приложение — источник истины.

Стартеры часто приносят и `@ConfigurationProperties`, связывая настройки с POJO и валидируя их. Это формирует стабильный контракт: вы читаете `application.yml`, а автоконфиг провалидает значения и создаст клиент. Пользователь может легко «подменить» любую деталь, объявив собственный бин.

Чтобы понять, **почему** бин (или автоконфиг) включился/не включился, включайте «условные логи»: `debug: true` в `application.yml`. Boot выведет «Condition evaluation report», где видно, какие условия сработали. Это главный инструмент диагностики автоконфигов.

Не забывайте про actuator-эндпоинты `beans/conditions` (в Boot 3 они доступны при включении). Они дают обзор реального контекста и условий — must-have на ревью сложных интеграций.

**Код (Java): простейшая автоконфигурация с условиями**

```java
package starter.demo;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

public interface HelloClient { String hello(String name); }

@AutoConfiguration
@ConditionalOnClass(name = "org.springframework.web.client.RestClient")
@ConditionalOnProperty(prefix = "hello.client", name = "enabled", havingValue = "true", matchIfMissing = true)
public class HelloClientAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean
  public HelloClient helloClient() {
    return name -> "Hello, " + name;
  }
}

@Configuration
class UserConfig {
  // пользователь при желании может переопределить бин HelloClient тут
}
```

**Код (Kotlin): аналог автоконфигурации**

```kotlin
package starter.demo

import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean

interface HelloClient { fun hello(name: String): String }

@AutoConfiguration
@ConditionalOnClass(name = ["org.springframework.web.client.RestClient"])
@ConditionalOnProperty(prefix = "hello.client", name = ["enabled"], havingValue = "true", matchIfMissing = true)
class HelloClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    fun helloClient(): HelloClient = HelloClient { name -> "Hello, $name" }
}
```

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
starter.demo.HelloClientAutoConfiguration
```

`application.yml` — выключить автоконфиг:

```yaml
hello:
  client:
    enabled: false
spring:
  autoconfigure:
    exclude:
      - starter.demo.HelloClientAutoConfiguration
```

Gradle (Groovy/Kotlin) — типичная зависимость на стартер:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter") }
```

---

## Границы ответственности: когда писать свою автоконфигурацию, как не «засорять» пользовательский граф

Автоконфигурация — инфраструктурный инструмент. Её задача — собрать «проволоку» вокруг библиотечных возможностей: создать клиентов, повесить декораторы (метрики, трейсинг), связать свойства. Не стоит пихать туда **бизнес-логику** или «выбор по доменным правилам». Там, где вариативность несёт предметный смысл, оставьте решение приложению.

Пишите автоконфиги, когда у вас переиспользуемый модуль: общий HTTP-клиент к внутреннему API; «пакет» интеграции с брокером; стандартная политика ретраев/логирования/трассировки. Тогда приложениям достаточно подключить стартер и поставить параметры в `application.yml`, не читая тонны инструкций.

Не «засоряйте» граф. Каждая автоконфигурация должна быть **условной** и «вежливой»: `@ConditionalOnClass` (нет класса — нет бинов), `@ConditionalOnProperty` (флаг «включить»), `@ConditionalOnMissingBean` (пользователь сильнее). Выносите «тяжёлые» бины за отдельные свойства (`enabled`) и выключайте их по умолчанию, если они не критичны.

Избегайте «скрытых» побочек. Автоконфиг не должен неожиданно запускать фоновые треды, менять глобальное состояние, подменять стандартные бины без веской причины. Если всё же нужно — подробно задокументируйте и добавьте явный флаг.

Организуйте структуру стартеров: `*-autoconfigure` (собственно конфиги) и `*-starter` (тонкая обёртка зависимостей, которая тянет autoconfigure и необходимые либы). Так пользователи подключают только стартер, а IDE им подсказывает доступные проперти через `spring-boot-configuration-processor`.

Тестируйте автоконфиги «контекстными» тестами: `ApplicationContextRunner` (из `spring-boot-test`) даёт мини-контекст, в котором вы включаете/выключаете свойства и проверяете появление бинов. Это быстрая и надёжная регрессия.

**Код (Java): «вежливая» автоконфигурация + тест через `ApplicationContextRunner`**

```java
package starter.boundary;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.context.annotation.Bean;

public interface AuditSink { void emit(String s); }

@AutoConfiguration
@ConditionalOnProperty(prefix="audit", name="enabled", havingValue = "true")
public class AuditAutoConfig {

  @Bean
  @ConditionalOnMissingBean
  AuditSink auditSink() { return s -> System.out.println("AUDIT " + s); }
}

// тест
class AuditAutoConfigTest {
  private final ApplicationContextRunner runner = new ApplicationContextRunner()
      .withConfiguration(org.springframework.boot.autoconfigure.AutoConfigurations.of(AuditAutoConfig.class));

  @org.junit.jupiter.api.Test
  void disabledByDefault() {
    runner.run(ctx -> org.assertj.core.api.Assertions.assertThat(ctx).doesNotHaveBean(AuditSink.class));
  }

  @org.junit.jupiter.api.Test
  void enabledWhenPropertySet() {
    runner.withPropertyValues("audit.enabled=true").run(ctx ->
        org.assertj.core.api.Assertions.assertThat(ctx).hasSingleBean(AuditSink.class));
  }
}
```

**Код (Kotlin): тестирование автоконфига**

```kotlin
package starter.boundary

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.autoconfigure.AutoConfiguration
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.boot.test.context.runner.ApplicationContextRunner
import org.springframework.context.annotation.Bean

interface AuditSink { fun emit(s: String) }

@AutoConfiguration
@ConditionalOnProperty(prefix = "audit", name = ["enabled"], havingValue = "true")
class AuditAutoConfig {
    @Bean
    @ConditionalOnMissingBean
    fun auditSink(): AuditSink = AuditSink { println("AUDIT $it") }
}

class AuditAutoConfigTest {
    private val runner = ApplicationContextRunner()
        .withConfiguration(org.springframework.boot.autoconfigure.AutoConfigurations.of(AuditAutoConfig::class.java))

    @Test fun disabledByDefault() {
        runner.run { ctx -> assertThat(ctx).doesNotHaveBean(AuditSink::class.java) }
    }
    @Test fun enabledWhenPropertySet() {
        runner.withPropertyValues("audit.enabled=true").run { ctx ->
            assertThat(ctx).hasSingleBean(AuditSink::class.java)
        }
    }
}
```

Gradle (Groovy/Kotlin) — тестовая зависимость:

```groovy
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

---

## Взгляд вперёд: AOT/Native-hint’ы и влияние на регистрацию бинов

С появлением Spring AOT и GraalVM Native образа, механика регистрации бинов и рефлексии стала строже. Во время **AOT-фазы** Spring генерирует «подсказки» (runtime hints) для рефлексии, ресурсов, прокси, чтобы нативный образ знал, что сохранять. Если вы пишете автоконфиги или динамически проксируете классы, вам может понадобиться явная регистрация хинтов.

Для рефлексии используйте `RuntimeHintsRegistrar` и аннотацию `@ImportRuntimeHints`. В регистраторе укажите классы/поля/конструкторы, которые понадобятся рефлексии (например, JSON-биндинг, маппинги драйверов). Для ресурсов — `hints.resources().registerPattern(...)`. Для прокси — `hints.proxies().registerJdkProxy(...)` или `registerClassProxy(...)`.

AOT влияет на «скрытые» практики: прямые вызовы `@Bean`-методов, неявная загрузка ресурсов, динамическая генерация классов. Всё это следует сделать явным или снабдить подсказками. Поэтому в библиотеках избегайте магии, которая тяжело описывается хинтами, и тестируйте стартеры в режиме `nativeTest`.

Условные аннотации по-прежнему работают, но учтите, что в native все «мертвые ветки» действительно исчезают из бинарника. Это плюс для старта/памяти, но если вы рассчитывали на динамическую загрузку класса — так не выйдет. Принимайте решение о включении на конфигурации/класспате во время сборки/генерации.

Документируйте требования к AOT: какие проперти обязаны быть, какие клиенты требуют hints, какие ресурсы кладутся в образ. Это сбережёт время интеграторам.

**Код (Java): регистрация AOT-хинтов**

```java
package starter.aot;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.context.annotation.ImportRuntimeHints;

@ImportRuntimeHints(MyHints.class)
public class NativeFriendlyConfig {}

class MyHints implements RuntimeHintsRegistrar {
  @Override
  public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
    hints.reflection().registerType(starter.demo.HelloClient.class);
    hints.resources().registerPattern("certs/*.pem");
    hints.proxies().registerJdkProxy(starter.demo.HelloClient.class);
  }
}
```

**Код (Kotlin): AOT-хинты**

```kotlin
package starter.aot

import org.springframework.aot.hint.RuntimeHints
import org.springframework.aot.hint.RuntimeHintsRegistrar
import org.springframework.context.annotation.ImportRuntimeHints
import starter.demo.HelloClient

@ImportRuntimeHints(MyHints::class)
class NativeFriendlyConfig

class MyHints : RuntimeHintsRegistrar {
    override fun registerHints(hints: RuntimeHints, classLoader: ClassLoader?) {
        hints.reflection().registerType(HelloClient::class.java)
        hints.resources().registerPattern("certs/*.pem")
        hints.proxies().registerJdkProxy(HelloClient::class.java)
    }
}
```

Gradle (Groovy/Kotlin) — сборка native:

```groovy
plugins { id 'org.graalvm.buildtools.native' version '0.10.2' }
```

```kotlin
plugins { id("org.graalvm.buildtools.native") version "0.10.2" }
```

---

# 10. Практика и анти-паттерны конфигурирования

## Чёткое разделение: «проволока» (конфиг) vs бизнес-код; минимум логики в конфигурации

Конфигурация — это **проволока**: создание клиентов, пулов, кэшей, связывание пропертей. Бизнес-логику в конфиги тянуть нельзя. Как только в `@Configuration` появляется ветвление предметных правил, вы теряете тестируемость и смешиваете слои. Конфиг должен быть детерминированным и быстро выполняться — он не место для принятия бизнес-решений.

Держите стратегию так: бизнес-сервисы — через `@Component`/`@Service` и конструкторы, инфраструктура — через `@Bean` (HTTP-клиенты, DataSource, сериализаторы). В `@Bean` допускаются только **технические** условия: класс есть на classpath, свойство включено, бин отсутствует. Никаких «если клиент VIP — то другой пул».

Минимизируйте логику в фабричных методах. Допустимы сборка билдера, таймауты, декораторы (метрики, ретраи). Появилась сложность — вынесите билдер/фабрику в отдельный класс и покройте тестами. Тогда `@Bean` просто вызывает `factory.create(props)`.

Избегайте чтения внешних данных в конфигурации (HTTP/файлы). Конфиг выполняется при старте — такие вызовы делают старт хрупким и медленным. Лучше подгрузить значения на `ApplicationReady` и кэшировать их как обычные зависимости.

Ясно изолируйте настройки: `@ConfigurationProperties` + `@Validated` дают типобезопасную конфигурацию. Это лучше, чем растаскивать `@Value` по коду. Так вы получаете один объект настроек и точку валидации на старте.

И наконец, ревью. К конфигам отношение особенно придирчивое: они влияют на всю систему. Любая «логика» здесь — потенциальный сюрприз. На PR спрашивайте: можно ли вынести это в сервис? можно ли задокументировать поведение в prop’ах?

**Код (Java): плохой и хороший конфиг**

```java
// Плохо: бизнес-логика в конфиге
@org.springframework.context.annotation.Configuration
class BadConfig {
  @org.springframework.context.annotation.Bean
  Payments payments(UserTierService tiers) {
    if (tiers.isVip()) { /* совсем другой клиент */ }
    return cents -> true;
  }
}

// Хорошо: конфиг только собирает инфраструктуру
@org.springframework.context.annotation.Configuration
class GoodConfig {
  @org.springframework.context.annotation.Bean
  Payments payments(PaymentProps props) {
    return cents -> cents < props.limitCents();
  }
}

interface Payments { boolean pay(long cents); }

@org.springframework.boot.context.properties.ConfigurationProperties("pay")
record PaymentProps(long limitCents) {}
```

**Код (Kotlin): исправленный подход**

```kotlin
@org.springframework.context.annotation.Configuration
class GoodConfig {
    @org.springframework.context.annotation.Bean
    fun payments(props: PaymentProps): Payments = Payments { cents -> cents < props.limitCents }
}

fun interface Payments { fun pay(cents: Long): Boolean }

@org.springframework.boot.context.properties.ConfigurationProperties("pay")
data class PaymentProps(val limitCents: Long = 10_00)
```

Gradle — процессор конфигурационных свойств:

```groovy
annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

```kotlin
annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")
implementation("org.springframework.boot:spring-boot-starter-validation")
```

---

## Не использовать Service Locator (`ApplicationContext#getBean`) в доменном коде; не смешивать `new` и DI

Service Locator ломает инверсию управления: объект снова сам «ищет» зависимости, пряча связность от конструктора. Это бьёт по тестируемости (нужно поднимать контекст), по читаемости (непонятно, что нужно классу), по устойчивости (строковые имена бинов дают ошибки в рантайме). Всё это — анти-паттерн вне инфраструктурных углов.

Смешивание `new` и DI разбивает централизованную конфигурацию. Любая тонкая настройка клиентов (таймауты, TLS, ретраи) должна жить в одном месте. Если где-то «быстрый `new`», вы получите зоопарк настроек, который невозможно диагностировать. Кроме того, такие объекты не попадают под пост-процессоры: не будет метрик/трассировки.

Правильный путь — конструкторная инъекция. Если реально нужно создать «одноразовый» объект — используйте `@Lookup`/`ObjectProvider` или явную фабрику-бин. Это даёт вам и гибкость, и управляемость контейнера.

Есть узкие места, где локатор допустим (например, интеграция со сторонним API, куда DI не достаёт). Тогда окружите его **адаптером** в инфраструктурном слое: пусть один класс получает бин из контекста и отдаёт чистый интерфейс домену.

Иногда тянет к `new` в тестах — и это нормально, если вы создаёте **сами** тестируемый класс и его фейки. Но в продакшн-коде `new` для зависимостей — красный флаг.

**Код (Java): антипример и исправления**

```java
// Плохо
class BadMailer {
  private final org.springframework.context.ApplicationContext ctx;
  BadMailer(org.springframework.context.ApplicationContext ctx) { this.ctx = ctx; }
  void send(String to) { ctx.getBean(SmtpClient.class).push(to); }
}

// Хорошо
class GoodMailer {
  private final SmtpClient smtp;
  GoodMailer(SmtpClient smtp) { this.smtp = smtp; }
  void send(String to) { smtp.push(to); }
}

interface SmtpClient { void push(String to); }
```

**Код (Kotlin): то же**

```kotlin
class GoodMailer(private val smtp: SmtpClient) {
    fun send(to: String) = smtp.push(to)
}
interface SmtpClient { fun push(to: String) }
```

---

## Прокси и self-invocation: почему кэш/транзакции/аспекты не срабатывают при внутренних вызовах

Аспекты Spring (транзакции `@Transactional`, кеш `@Cacheable`, ретраи и др.) навешиваются через **прокси** вокруг бина. Когда вы вызываете аннотированный метод **извне**, вызов проходит через прокси — совет срабатывает. Но при **внутреннем вызове** метода из того же класса (self-invocation) вы обходите прокси и попадаете напрямую в реализацию — аннотация не активируется. Это частая причина «не работает @Transactional/@Cacheable».

Есть три рабочих пути. Первый — вынести аннотированный метод в **отдельный бин** и вызывать его как зависимость. Второй — инжектить **self-proxy** (например, через `ObjectProvider<ThisType>` или `AopContext.currentProxy()`) и вызывать метод через прокси. Третий — делать аспект на внешний слой (например, на сервис), а внутри вызывать приватные методы без аннотаций — тогда всё пройдёт через прокси на входе.

У self-proxy есть нюансы: класс должен быть проксируемым (интерфейс или неблокирующий CGLIB-прокси), а код становится менее очевидным. Поэтому вынос в отдельный компонент обычно чище и тестопригоднее.

Проверить, есть ли прокси, можно логированием класса (`getClass()`) или включив `spring.aop.proxy-target-class=true`/логи инфраструктуры AOP. В проде держите это поведение стабильным и документированным.

Не забывайте про финальные методы и Kotlin-классы: CGLIB не может переопределить `final` — аспекты на такие методы не навесятся. В Kotlin помогает плагин `kotlin-spring`, который делает классы/методы `open` там, где это нужно.

**Код (Java): self-invocation и исправление**

```java
package app.self;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class PriceService {
  // НЕ сработает при вызове getPrice() -> calc() внутри класса
  @Cacheable("price")
  public long calc(String sku) { return 42; }

  public long getPrice(String sku) { return calc(sku); } // обходит прокси
}
```

```java
package app.self;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
class PriceServiceFixed {
  private final ObjectProvider<PriceServiceFixed> self;
  PriceServiceFixed(ObjectProvider<PriceServiceFixed> self) { this.self = self; }

  @Cacheable("price") public long calc(String sku) { return 42; }

  public long getPrice(String sku) { return self.getObject().calc(sku); } // через прокси
}
```

**Код (Kotlin): вынос в отдельный бин**

```kotlin
package app.self

import org.springframework.cache.annotation.Cacheable
import org.springframework.stereotype.Service

@Service
class PriceCalculator {
    @Cacheable("price")
    fun calc(sku: String): Long = 42
}

@Service
class PriceService(private val calc: PriceCalculator) {
    fun getPrice(sku: String): Long = calc.calc(sku) // проходит через прокси PriceCalculator
}
```

Gradle — кеш и AOP:

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-aop'
  implementation 'org.springframework.boot:spring-boot-starter-cache'
}
```

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-aop")
  implementation("org.springframework.boot:spring-boot-starter-cache")
}
```

---

## Диагностика: условные логи старта, отчёты контекста, визуализация графа зависимостей для ревью

Первая линия обороны — включить «условные логи»: `debug: true` в `application.yml`. Boot напечатает отчёт условий (какие автоконфиги активны, почему некоторые отключены). Для сложных конфликтов бинов это бесценно: видны `@ConditionalOnClass/@OnMissingBean` и конкретные причины.

Вторая — actuator-эндпоинты `beans` и `conditions`. Они дают JSON-картины: «какие бины есть (тип, скоуп, зависимости)» и «какие условия прошли/не прошли». Подключите их в dev/stage (не в общий прод) и добавьте ссылку в README. Это ускоряет ревью и on-call диагностику.

Третья — «микро-инструменты» из кода: короткие `BeanFactoryPostProcessor`/слушатели, печатающие дефиниции и порядок фаз, когда это уместно. Они помогают локализовать проблемы старта, циклы, неожиданные скоупы. Уберите их за флаг конфигурации, чтобы не шуметь в проде.

Для автоконфигов используйте `ApplicationContextRunner` — воспроизводимые тесты, показывающие, какие бины появляются при тех или иных пропертях. На ревью проще обсуждать тесты, чем скриншоты логов.

Для визуализации графа есть удобные сторонние инструменты, но зачастую достаточно `/actuator/beans` + маленький скрипт. Главное — договориться в команде, что перед мержем «тяжёлых» конфиг-правок делается снимок графа и прикладывается к PR.

И не забывайте про уровни логирования для «org.springframework`: поднять на `INFO/DEBUG` точечные пакеты (`condition`, `beans`, `aop`) и вернуть назад после диагностики. Дисциплина логирования — лучшая страховка.

**Код (Java): включаем отчёты и печать дефиниций под флагом)**

```java
// application.yml
/*
debug: true
management:
  endpoints:
    web:
      exposure:
        include: beans,conditions,health,info
*/

package app.diagnostics;

import org.springframework.beans.factory.config.*;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("diag")
public class BeanDefsPrinter implements BeanFactoryPostProcessor {
  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
    for (String name : bf.getBeanDefinitionNames()) {
      var bd = bf.getBeanDefinition(name);
      System.out.println("[DEF] " + name + " -> " + bd.getBeanClassName() + " scope=" + bd.getScope());
    }
  }
}
```

**Код (Kotlin): то же**

```kotlin
// application.yml см. выше

package app.diagnostics

import org.springframework.beans.factory.config.BeanFactoryPostProcessor
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory
import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

@Component
@Profile("diag")
class BeanDefsPrinter : BeanFactoryPostProcessor {
    override fun postProcessBeanFactory(bf: ConfigurableListableBeanFactory) {
        bf.beanDefinitionNames.forEach { name ->
            val bd = bf.getBeanDefinition(name)
            println("[DEF] $name -> ${bd.beanClassName} scope=${bd.scope}")
        }
    }
}
```

Gradle — actuator:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-actuator' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-actuator") }
```





