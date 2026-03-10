---
layout: page
title: "Бины и конфигурация: как «живёт» приложение"
permalink: /spring/beans
---

<!-- TOC -->
* [0. Введение](#0-введение)
  * [Что это](#что-это)
  * [Зачем это](#зачем-это)
  * [Где используется](#где-используется)
  * [Какие проблемы/задачи решает](#какие-проблемызадачи-решает)
* [1. Что такое бин и откуда он берётся](#1-что-такое-бин-и-откуда-он-берётся)
  * [BeanDefinition: метаданные о типе, скоупе, зависимостях, способе создания](#beandefinition-метаданные-о-типе-скоупе-зависимостях-способе-создания)
  * [Источники бинов: компонент-сканирование (`@Component`-семейство), Java-конфигурация (`@Configuration` + `@Bean`), импорт (`@Import`, `@ImportSelector`), фабрики (`FactoryBean`)](#источники-бинов-компонент-сканирование-component-семейство-java-конфигурация-configuration--bean-импорт-import-importselector-фабрики-factorybean)
  * [Именование и алиасы: каноническое имя, коллизии, правила чтения «дерева» бинов](#именование-и-алиасы-каноническое-имя-коллизии-правила-чтения-дерева-бинов)
* [2. Жизненный цикл бина (от регистрации до уничтожения)](#2-жизненный-цикл-бина-от-регистрации-до-уничтожения)
  * [Фазы старта контекста: регистрация дефиниций → пост-процессоры → создание singleton-бинов → вызовы колбэков → `ApplicationReady`](#фазы-старта-контекста-регистрация-дефиниций--пост-процессоры--создание-singleton-бинов--вызовы-колбэков--applicationready)
  * [Инициализация бина: внедрение зависимостей → `Aware`-интерфейсы → `@PostConstruct`/`afterPropertiesSet`/init-method → пост-процессоры (`BeanPostProcessor`)](#инициализация-бина-внедрение-зависимостей--aware-интерфейсы--postconstructafterpropertiessetinit-method--пост-процессоры-beanpostprocessor)
  * [Завершение: `@PreDestroy`/`DisposableBean`/destroy-method, порядок остановки](#завершение-predestroydisposablebeandestroy-method-порядок-остановки)
* [3. Конфигурация через `@Configuration` и `@Bean`](#3-конфигурация-через-configuration-и-bean)
  * [Полноценные конфигурационные классы vs «простые» компоненты; `proxyBeanMethods=true/false` и межбиновые вызовы](#полноценные-конфигурационные-классы-vs-простые-компоненты-proxybeanmethodstruefalse-и-межбиновые-вызовы)
  * [Семантика `@Bean`: скоуп, `initMethod/destroyMethod`, видимость и зависимость](#семантика-bean-скоуп-initmethoddestroymethod-видимость-и-зависимость)
  * [Method injection (`@Lookup`), вызовы `@Bean`-методов, почему нельзя подменять их прямыми вызовами как «фабрику»](#method-injection-lookup-вызовы-bean-методов-почему-нельзя-подменять-их-прямыми-вызовами-как-фабрику)
* [0. Введение](#0-введение-1)
  * [Что это](#что-это-1)
  * [Зачем это](#зачем-это-1)
  * [Где используется](#где-используется-1)
  * [Какие проблемы/задачи решает](#какие-проблемызадачи-решает-1)
* [1. Что такое бин и откуда он берётся](#1-что-такое-бин-и-откуда-он-берётся-1)
  * [BeanDefinition: метаданные о типе, скоупе, зависимостях, способе создания](#beandefinition-метаданные-о-типе-скоупе-зависимостях-способе-создания-1)
  * [Источники бинов: компонент-сканирование (`@Component`-семейство), Java-конфигурация (`@Configuration` + `@Bean`), импорт (`@Import`, `@ImportSelector`), фабрики (`FactoryBean`)](#источники-бинов-компонент-сканирование-component-семейство-java-конфигурация-configuration--bean-импорт-import-importselector-фабрики-factorybean-1)
  * [Именование и алиасы: каноническое имя, коллизии, правила чтения «дерева» бинов](#именование-и-алиасы-каноническое-имя-коллизии-правила-чтения-дерева-бинов-1)
* [2. Жизненный цикл бина (от регистрации до уничтожения)](#2-жизненный-цикл-бина-от-регистрации-до-уничтожения-1)
  * [Фазы старта контекста: регистрация дефиниций → пост-процессоры → создание singleton-бинов → вызовы колбэков → `ApplicationReady`](#фазы-старта-контекста-регистрация-дефиниций--пост-процессоры--создание-singleton-бинов--вызовы-колбэков--applicationready-1)
  * [Инициализация бина: внедрение зависимостей → `Aware`-интерфейсы → `@PostConstruct`/`afterPropertiesSet`/init-method → пост-процессоры (`BeanPostProcessor`)](#инициализация-бина-внедрение-зависимостей--aware-интерфейсы--postconstructafterpropertiessetinit-method--пост-процессоры-beanpostprocessor-1)
  * [Завершение: `@PreDestroy`/`DisposableBean`/destroy-method, порядок остановки](#завершение-predestroydisposablebeandestroy-method-порядок-остановки-1)
* [3. Конфигурация через `@Configuration` и `@Bean`](#3-конфигурация-через-configuration-и-bean-1)
  * [Полноценные конфигурационные классы vs «простые» компоненты; `proxyBeanMethods=true/false` и межбиновые вызовы](#полноценные-конфигурационные-классы-vs-простые-компоненты-proxybeanmethodstruefalse-и-межбиновые-вызовы-1)
  * [Семантика `@Bean`: скоуп, `initMethod/destroyMethod`, видимость и зависимость](#семантика-bean-скоуп-initmethoddestroymethod-видимость-и-зависимость-1)
  * [Method injection (`@Lookup`), вызовы `@Bean`-методов, почему нельзя подменять их прямыми вызовами как «фабрику»](#method-injection-lookup-вызовы-bean-методов-почему-нельзя-подменять-их-прямыми-вызовами-как-фабрику-1)
* [8. События и раннеры приложения](#8-события-и-раннеры-приложения)
  * [События контекста: `ContextRefreshed`, `ApplicationStarted`, `ApplicationReady`, `ContextClosed`; обработка через `@EventListener`](#события-контекста-contextrefreshed-applicationstarted-applicationready-contextclosed-обработка-через-eventlistener)
  * [`CommandLineRunner`/`ApplicationRunner`: инициализация данных, прогрев кэшей/соединений, порядок выполнения](#commandlinerunnerapplicationrunner-инициализация-данных-прогрев-кэшейсоединений-порядок-выполнения)
  * [`SmartLifecycle`: автозапуск, фазы и «корректная» остановка](#smartlifecycle-автозапуск-фазы-и-корректная-остановка)
* [9. Автоконфигурация Spring Boot (обзор)](#9-автоконфигурация-spring-boot-обзор)
  * [Как «подцепляются» бины из стартеров: условия (`@Conditional*`), порядок, отключение/override](#как-подцепляются-бины-из-стартеров-условия-conditional-порядок-отключениеoverride)
  * [Границы ответственности: когда писать свою автоконфигурацию, как не «засорять» пользовательский граф](#границы-ответственности-когда-писать-свою-автоконфигурацию-как-не-засорять-пользовательский-граф)
  * [Взгляд вперёд: AOT/Native-hint’ы и влияние на регистрацию бинов](#взгляд-вперёд-aotnative-hintы-и-влияние-на-регистрацию-бинов)
* [10. Практика и анти-паттерны конфигурирования](#10-практика-и-анти-паттерны-конфигурирования)
  * [Чёткое разделение: «проволока» (конфиг) vs бизнес-код; минимум логики в конфигурации](#чёткое-разделение-проволока-конфиг-vs-бизнес-код-минимум-логики-в-конфигурации)
  * [Не использовать Service Locator (`ApplicationContext#getBean`) в доменном коде; не смешивать `new` и DI](#не-использовать-service-locator-applicationcontextgetbean-в-доменном-коде-не-смешивать-new-и-di)
  * [Прокси и self-invocation: почему кэш/транзакции/аспекты не срабатывают при внутренних вызовах](#прокси-и-self-invocation-почему-кэштранзакцииаспекты-не-срабатывают-при-внутренних-вызовах)
  * [Диагностика: условные логи старта, отчёты контекста, визуализация графа зависимостей для ревью](#диагностика-условные-логи-старта-отчёты-контекста-визуализация-графа-зависимостей-для-ревью)
* [Вопросы](#вопросы-)
  * [Ответы](#ответы)
* [Теоретические материалы](#теоретические-материалы)
* [Задачи](#задачи)
  * [1. Собрать модуль уведомлений с разделением бизнес-кода и конфигурации](#1-собрать-модуль-уведомлений-с-разделением-бизнес-кода-и-конфигурации)
    * [Решение](#решение)
  * [2. Реализовать словарь статусов с корректным жизненным циклом и поздним прогревом](#2-реализовать-словарь-статусов-с-корректным-жизненным-циклом-и-поздним-прогревом)
    * [Решение](#решение-1)
  * [3. Построить маршрутизатор обработчиков через `Map<String, RequestHandler>`](#3-построить-маршрутизатор-обработчиков-через-mapstring-requesthandler)
    * [Решение](#решение-2)
  * [4. Переписать конфигурацию так, чтобы она была корректной при `proxyBeanMethods=false`](#4-переписать-конфигурацию-так-чтобы-она-была-корректной-при-proxybeanmethodsfalse)
    * [Решение](#решение-3)
  * [5. Спроектировать мини-автоконфигурацию для аудита, которая не ломает пользовательский контекст](#5-спроектировать-мини-автоконфигурацию-для-аудита-которая-не-ломает-пользовательский-контекст)
    * [Решение](#решение-4)
  * [6. Исправить пакетную обработку так, чтобы она использовала `prototype`, `@Lookup` и корректный управляемый lifecycle](#6-исправить-пакетную-обработку-так-чтобы-она-использовала-prototype-lookup-и-корректный-управляемый-lifecycle)
    * [Решение](#решение-5)
<!-- TOC -->

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


 
# Вопросы 

1. Что такое бин в Spring и чем он отличается от обычного Java-объекта?
2. Что такое `BeanDefinition` и зачем разработчику понимать этот уровень абстракции?
3. Какими способами бин может попасть в Spring-контекст?
4. Когда использовать `@Component`-семейство, а когда `@Configuration` + `@Bean`?
5. Как в Spring устроены имена бинов, алиасы и почему это важно?
6. Что происходит во время старта Spring-контекста до момента, когда приложение считается готовым?
7. В каком порядке инициализируется конкретный бин внутри контейнера?
8. Чем отличаются `@PostConstruct`, `InitializingBean` и `initMethod`?
9. Что происходит с бинами при завершении приложения?
10. Чем отличается полноценный `@Configuration`-класс от обычного `@Component` с методами `@Bean`?
11. Что означает `proxyBeanMethods=true/false` и почему это влияет на поведение приложения?
12. Как работает `@Bean` и какие задачи он решает лучше, чем компонент-сканирование?
13. Что такое scope бина и почему с `prototype` нужно работать осторожно?
14. Что такое `FactoryBean` и в каких случаях он действительно полезен?
15. Для чего нужен `@Lookup` и как он помогает получать новые экземпляры бинов?
16. Почему прямой вызов `@Bean`-метода не равен получению бина из контейнера?
17. Как связаны события приложения, `ApplicationRunner`, `CommandLineRunner` и жизненный цикл бинов?
18. Что такое `SmartLifecycle` и когда он лучше обычных startup/shutdown-хуков?
19. Как работает автоконфигурация Spring Boot и почему она должна быть “вежливой”?
20. Какие анти-паттерны чаще всего встречаются при работе с бинами и конфигурацией в Spring?
 
## Ответы

**1. Что такое бин в Spring и чем он отличается от обычного Java-объекта?**

Бин — это объект, жизненным циклом которого управляет Spring-контейнер. Контейнер знает, как его создать, какие зависимости в него внедрить, в каком scope держать, когда вызвать инициализацию и когда корректно уничтожить. Сам по себе класс ещё не делает объект бином: критично именно то, что экземпляр зарегистрирован и обслуживается контейнером.

Обычный Java-объект, созданный через `new`, может быть экземпляром того же самого класса, но Spring о нём ничего не знает. Это означает, что на него не распространяются контейнерные механизмы: автосвязывание зависимостей, `@PostConstruct`, `@PreDestroy`, AOP-прокси, транзакции, кеширование, метрики и другие расширения. Снаружи такие объекты могут выглядеть одинаково, но в рантайме их поведение часто принципиально различается.

Практический вывод очень важен для новичков: в Spring-проекте вопрос “кто создал объект?” почти так же важен, как и вопрос “какой это класс?”. Если объект должен участвовать в инфраструктуре фреймворка, его следует создавать через контейнер. Если же разработчик смешивает контейнерное управление и ручное создание объектов, приложение быстро становится непредсказуемым и хуже тестируется.

**2. Что такое `BeanDefinition` и зачем разработчику понимать этот уровень абстракции?**

`BeanDefinition` — это внутренняя мета-информация о бине, которую Spring хранит в контейнере. В ней описывается класс или способ создания объекта, scope, имя, зависимости, методы инициализации и уничтожения, а также ряд дополнительных параметров контейнерного поведения. Фактически это не сам объект, а его декларативное описание.

Большую часть времени прикладной разработчик напрямую с `BeanDefinition` не работает. Но понимание этого слоя помогает разобраться, почему Spring сначала “знает о бине”, а уже потом создаёт его. Именно поэтому контейнер способен менять поведение до фактической инициализации: добавлять прокси, применять пост-процессоры, регистрировать дополнительные бины, настраивать lifecycle-колбэки и реагировать на условия автоконфигурации.

Это знание особенно полезно при диагностике сложных проблем. Когда у приложения конфликт имён, неожиданно выбирается не тот бин, появляется лишний прокси или библиотека регистрирует свои компоненты, причина часто находится именно на уровне дефиниций, а не готовых экземпляров. Новичку не нужно сразу писать собственные `BeanDefinitionRegistryPostProcessor`, но понимать, что Spring работает сначала с метаданными, а потом с объектами, крайне полезно.

**3. Какими способами бин может попасть в Spring-контекст?**

Самый распространённый путь — компонент-сканирование. Разработчик помечает класс аннотацией `@Component`, `@Service`, `@Repository`, `@Controller` или другой стереотипной аннотацией, после чего Spring находит этот класс при сканировании пакетов и регистрирует его как бин. Такой подход удобен для обычного прикладного кода: сервисов, адаптеров, обработчиков, контроллеров.

Второй распространённый путь — Java-конфигурация через `@Configuration` и методы `@Bean`. Этот вариант лучше подходит для инфраструктурных объектов, особенно если они создаются через builder, требуют внешних параметров или принадлежат сторонней библиотеке. Например, так часто регистрируют HTTP-клиенты, пула соединений, сериализаторы, SDK-клиенты и фабрики.

Кроме того, бины могут приходить через `@Import`, `@ImportSelector`, автоконфигурации Spring Boot и через специальные механизмы вроде `FactoryBean`. Важно понимать, что бин — это не обязательно “класс с аннотацией”, а любой объект, который Spring корректно зарегистрировал в контексте. Это помогает не сужать мышление только до компонент-сканирования и понимать архитектуру стартеров и библиотек.

**4. Когда использовать `@Component`-семейство, а когда `@Configuration` + `@Bean`?**

`@Component`-семейство обычно используют для бизнесовых и прикладных классов. Это сервисы, доменные обработчики, репозитории, контроллеры, listeners и другие компоненты, которые представляют отдельные роли в приложении. Такой код проще читать, потому что назначение класса видно сразу, а его жизненный цикл полностью контролируется Spring без дополнительной ручной фабрики.

`@Configuration` + `@Bean` лучше применять там, где нужно явно описать процесс сборки объекта. Это особенно полезно для сторонних библиотек, сложных клиентов и технических компонентов, у которых есть builder, таймауты, сертификаты, пул потоков, кастомные сериализаторы или иные параметры. В таком случае конфигурационный класс становится местом, где сосредоточена “проволока”, а не бизнес-логика.

Хорошее практическое правило звучит так: прикладные роли — через компонент-сканирование, инфраструктурная сборка — через `@Bean`. Тогда архитектура остаётся прозрачной. Если всё подряд регистрировать через `@Bean`, конфигурационный слой разрастается и начинает скрывать бизнес-структуру. Если же всё пытаться засунуть в `@Component`, настройка сторонних клиентов и ресурсов становится неочевидной и плохо управляемой.

**5. Как в Spring устроены имена бинов, алиасы и почему это важно?**

У каждого бина в контейнере есть имя. Для компонентов, найденных через сканирование, по умолчанию это обычно имя класса в `camelCase`, а для `@Bean` — имя метода. Кроме основного имени, бин может иметь алиасы, то есть дополнительные имена, по которым его тоже можно получить из контекста.

На маленьких проектах имена бинов часто кажутся второстепенной деталью, потому что большинство инъекций происходит по типу. Но как только в приложении появляется несколько реализаций одного интерфейса, коллекции `Map<String, T>`, условная регистрация, стартеры или интеграция с чужими конфигурациями, имена начинают играть большую роль. Они становятся ключами выбора, диагностики и маршрутизации.

Отсюда вытекает важная дисциплина: имена должны быть осмысленными, а не случайными. Если в проекте есть несколько реализаций `NotificationSender`, названия вроде `emailSender` и `smsSender` намного полезнее, чем абстрактные `senderOne` и `senderTwo`. Осмысленные имена облегчают чтение логов, работу с `@Qualifier`, построение карт стратегий и разбор конфликтов в крупных приложениях.

**6. Что происходит во время старта Spring-контекста до момента, когда приложение считается готовым?**

Старт приложения начинается с регистрации дефиниций бинов. Spring читает конфигурационные классы, сканирует компоненты, подключает импортируемые модули и формирует представление о будущем графе зависимостей. На этом этапе многие объекты ещё не созданы, но контейнер уже знает, какие компоненты в принципе существуют и как они должны быть собраны.

Далее вступают в работу различные пост-процессоры фабрики и инфраструктурные расширения. Они могут модифицировать дефиниции, добавлять новые бины, подключать автоконфигурацию или менять способ создания существующих компонентов. После этого контейнер начинает создавать singleton-бины, внедрять в них зависимости, вызывать lifecycle-колбэки и применять `BeanPostProcessor`, который часто и добавляет прокси для AOP, транзакций, кеширования и других механизмов.

Когда основной граф singleton-бинов уже поднят, Spring Boot публикует более поздние стадии старта приложения, включая события и вызовы раннеров. Именно здесь приложение постепенно переходит из состояния “контекст собран” в состояние “готово к работе”. Для разработчика это означает, что не всё, что работает после конструктора, уже обязательно находится в финальной рабочей фазе; часть логики лучше переносить в `ApplicationReadyEvent`, раннеры или управляемые lifecycle-компоненты.

**7. В каком порядке инициализируется конкретный бин внутри контейнера?**

Сначала Spring создаёт экземпляр бина и внедряет в него зависимости. Это значит, что конструктор уже отработал, а необходимые ссылки на другие бины уже подготовлены или будут переданы следующими этапами связывания. На этом шаге объект физически создан, но его инициализация ещё не завершена.

После этого могут вызываться `Aware`-интерфейсы, если класс их реализует. Так бин может узнать своё имя, получить ссылку на `BeanFactory`, `ApplicationContext` и другую инфраструктурную информацию. Затем запускаются lifecycle-механизмы вроде `@PostConstruct`, `afterPropertiesSet()` и заданного `initMethod`. Параллельно вокруг бина работают `BeanPostProcessor`, которые способны подменить объект, обернуть его прокси или добавить дополнительное поведение.

Важно понимать не просто список этапов, а их практический смысл. Если разработчик кладёт тяжёлую инициализацию слишком рано, он замедляет старт приложения. Если он рассчитывает на работу прокси внутри неподходящей фазы, может получить неожиданное поведение. Поэтому хороший код разделяет техническую инициализацию, поздний прогрев, вход в рабочий режим и корректное завершение, а не складывает всё в одно место.

**8. Чем отличаются `@PostConstruct`, `InitializingBean` и `initMethod`?**

Все три механизма нужны для запуска кода после того, как зависимости уже внедрены. Они решают одну и ту же задачу, но делают это разными способами. `@PostConstruct` — это аннотационный и наиболее нейтральный вариант, `InitializingBean` — интерфейс Spring, а `initMethod` — способ описать инициализацию на уровне конфигурации.

`@PostConstruct` обычно удобнее в прикладном коде, потому что он не заставляет класс явно зависеть от Spring-интерфейсов. `InitializingBean` сильнее привязывает класс к фреймворку, поэтому его чаще считают менее предпочтительным для обычной доменной логики. `initMethod`, наоборот, особенно полезен тогда, когда сам класс менять нельзя: например, если объект приходит из внешней библиотеки, а инициализацию всё равно нужно встроить в lifecycle контейнера.

На практике команде стоит придерживаться единообразия. Чаще всего для собственных классов выбирают `@PostConstruct`, а `initMethod` используют в `@Bean`-конфигурациях для сторонних типов. Главное — не смешивать все подходы бессистемно. Инициализация должна быть читаемой: по коду должно быть понятно, где и почему объект начинает “жить”.

**9. Что происходит с бинами при завершении приложения?**

Когда приложение завершает работу, Spring начинает фазу остановки контекста. На этом этапе контейнер публикует соответствующие события и запускает механизмы уничтожения бинов. Для singleton-компонентов это может означать вызов `@PreDestroy`, `DisposableBean.destroy()` и `destroyMethod`, если он был настроен в `@Bean`.

Практическая цель этой фазы — не “красиво закончить”, а корректно освободить ресурсы. Нужно закрыть соединения, остановить планировщики, завершить фоновые задачи, выгрузить буферы, остановить listeners и аккуратно прекратить работу с внешними системами. Если этого не сделать, приложение может терять данные, рвать транзакции и нестабильно вести себя при rolling update.

Новичкам важно запомнить, что lifecycle Spring не заканчивается на старте. Корректная остановка — это такая же часть архитектуры, как и корректное создание графа зависимостей. Если класс открывает ресурс на старте, у него должна быть продуманная стратегия его освобождения. Иначе система будет работать “как будто нормально” до первого реального продового сценария с рестартами и graceful shutdown.

**10. Чем отличается полноценный `@Configuration`-класс от обычного `@Component` с методами `@Bean`?**

Оба варианта могут регистрировать бины, но семантически они не равны. `@Configuration` — это специальный тип компонента, который предназначен именно для описания bean-definition’ов. Spring ожидает, что такой класс является конфигурацией, и при необходимости проксирует его так, чтобы межбиновые вызовы внутри этого класса работали корректно с точки зрения контейнера.

Если методы `@Bean` находятся в обычном `@Component`, они тоже регистрируют бины, но класс уже не рассматривается как полноценный конфигурационный источник в том же смысле. Межбиновые вызовы не получают специального поведения, и разработчик легко может начать думать, что вызывает бин из контекста, хотя на деле выполняет обычный Java-метод и создаёт новый объект.

Именно поэтому для явной инфраструктурной конфигурации лучше использовать `@Configuration`. Это не просто “ещё одна аннотация”, а способ зафиксировать намерение. Когда код читают другие разработчики, они сразу понимают: этот класс отвечает за сборку инфраструктуры, а не за бизнес-логику. Такая ясность сильно снижает риск скрытых ошибок и неверных предположений о поведении приложения.

**11. Что означает `proxyBeanMethods=true/false` и почему это влияет на поведение приложения?**

Параметр `proxyBeanMethods` управляет тем, будет ли Spring проксировать конфигурационный класс для корректной обработки вызовов его `@Bean`-методов. Если используется `true`, вызовы между такими методами идут через контейнер и возвращают управляемые экземпляры, а не новые объекты, созданные прямым вызовом. Это особенно важно для singleton-бинов, где ожидается единый экземпляр.

Если же указано `false`, Spring не будет подменять вызовы `@Bean`-методов внутри конфигурации. Внешне код может выглядеть так же, но логика меняется: вызов `this.someBean()` фактически становится обычным методом Java и может создать новый объект в обход контейнера. Именно поэтому конфигурации с `proxyBeanMethods=false` пишут так, чтобы зависимости выражались через параметры `@Bean`-методов, а не через взаимные вызовы методов.

Это не просто микропараметр производительности, а часть семантики конфигурации. Да, отключение прокси может ускорить старт и упростить конфиг, но только если код уже написан безопасным способом. Ошибка здесь коварна тем, что приложение может не упасть сразу: оно просто начнёт жить с несколькими экземплярами того, что разработчик считал одним бином. Поэтому решение о `proxyBeanMethods=false` должно быть осознанным, а не механическим.

**12. Как работает `@Bean` и какие задачи он решает лучше, чем компонент-сканирование?**

Аннотация `@Bean` сообщает Spring, что возвращаемый объект метода нужно зарегистрировать в контексте как бин. Имя по умолчанию берётся из имени метода, а зависимости можно передать через параметры этого метода. Такой подход превращает метод в явную фабрику, встроенную в lifecycle контейнера.

Главная сила `@Bean` в том, что он хорошо подходит для настройки объектов, которые неудобно создавать простым компонент-сканированием. Это сторонние классы, builder-based клиенты, ресурсы с параметрами, технические фабрики, пулы соединений, сериализаторы и прочая инфраструктура. Разработчик может централизованно описать все нюансы сборки: таймауты, декораторы, init/destroy-методы, внешние свойства и условия включения.

Компонент-сканирование проще и естественнее для прикладных ролей, а `@Bean` — для управляемой технической сборки. Если начать регистрировать через `@Bean` всё подряд, конфиг-классы быстро разрастутся и станут трудно читаемыми. Но если использовать `@Bean` по назначению, он делает инфраструктуру прозрачной и контролируемой, а проект — значительно более поддерживаемым.

**13. Что такое scope бина и почему с `prototype` нужно работать осторожно?**

Scope определяет жизненный цикл и область существования бина. По умолчанию в Spring используется `singleton`: контейнер создаёт один экземпляр на контекст и затем переиспользует его во всех точках инъекции. Для большинства сервисов, клиентов и адаптеров этого более чем достаточно, потому что именно такие компоненты обычно проектируются как stateless.

Другие scope нужны, когда модель жизни объекта отличается. Например, в веб-приложениях бывают request- и session-scoped бины, а `prototype` означает: каждый запрос к контейнеру за этим бином создаёт новый экземпляр. Это удобно для короткоживущих рабочих объектов, команд, буферов, временных обработчиков и других сущностей, которые действительно не должны разделяться.

Опасность `prototype` в том, что многие ожидают от него такого же уровня lifecycle-управления, как у singleton. Но контейнер лишь создаёт новый экземпляр и отдаёт его, а дальше ответственность за использование и освобождение ресурсов уже сильно больше лежит на разработчике. Если бездумно заменить singleton на prototype, можно получить утечки, неожиданные состояния и путаницу с тем, кто вообще должен владеть жизненным циклом объекта.

**14. Что такое `FactoryBean` и в каких случаях он действительно полезен?**

`FactoryBean` — это специальный контракт Spring, при котором сам бин является не конечным сервисом, а фабрикой для другого объекта. Когда контейнер получает такой бин, по его имени он обычно возвращает не фабрику, а продукт, который она создаёт. При необходимости сам объект-фабрику можно получить отдельно по имени с префиксом `&`.

Этот механизм полезен тогда, когда создание объекта само по себе нетривиально. Например, нужно выбрать реализацию динамически, построить интеграционный клиент через сложную цепочку шагов, собрать proxy-объект или спрятать за фабрикой сложный жизненный цикл стороннего SDK. В таких случаях `FactoryBean` позволяет оформить создание аккуратно и встроить его в модель контейнера.

Однако это продвинутый инструмент, а не повседневный способ регистрировать обычные сервисы. Если `FactoryBean` начинают использовать там, где достаточно обычного `@Bean`, код только усложняется. Его стоит рассматривать как инфраструктурный механизм для библиотек и нетривиальной сборки, а не как универсальную альтернативу привычной конфигурации.

**15. Для чего нужен `@Lookup` и как он помогает получать новые экземпляры бинов?**

`@Lookup` решает типичную проблему: singleton-бину иногда нужен не один и тот же объект, а новый экземпляр зависимости на каждую операцию. Если просто внедрить `prototype` в конструктор singleton-компонента, он будет создан один раз в момент сборки singleton, а не заново на каждый вызов. В таких сценариях и нужен lookup-метод.

Разработчик помечает метод аннотацией `@Lookup`, а Spring во время создания бина подменяет его реализацию так, чтобы при вызове происходило обращение к контейнеру за свежим экземпляром нужного типа. Это позволяет писать код в объектно-ориентированном стиле, не делая явных вызовов `ApplicationContext.getBean(...)` внутри бизнес-логики.

Но `@Lookup` — это точечный инструмент, а не решение на все случаи жизни. Если он начинает массово появляться в проекте, это сигнал, что модель скоупов или фабрик спроектирована неудачно. В хорошей архитектуре `@Lookup` используют там, где действительно нужен короткоживущий рабочий объект, а не как способ “по-быстрому получить что-то из контейнера”.

**16. Почему прямой вызов `@Bean`-метода не равен получению бина из контейнера?**

Синтаксически `@Bean`-метод — это обычный Java-метод, и в этом заключается источник частой ошибки. Разработчики видят аннотацию и начинают воспринимать вызов `someConfig.myBean()` как контейнерную операцию, хотя на уровне языка это просто метод. Контейнерное поведение появляется только при соблюдении условий конфигурационного проксирования.

Если конфигурационный класс проксируется и вызов проходит через соответствующий механизм Spring, контейнер может вернуть уже зарегистрированный бин. Но если прокси не участвует — например, используется `proxyBeanMethods=false` или метод вызывается не так, как ожидает Spring, — код создаёт новый объект напрямую. Внешне это выглядит почти одинаково, а фактически полностью меняет поведение приложения.

Отсюда ключевое правило: зависимости между `@Bean`-методами лучше выражать через параметры методов, а не через прямые вызовы внутри класса. Тогда конфигурация остаётся корректной независимо от режима проксирования, проще читается и не создаёт скрытых дубликатов объектов. Это один из тех практических приёмов, который предотвращает много трудноуловимых дефектов.

**17. Как связаны события приложения, `ApplicationRunner`, `CommandLineRunner` и жизненный цикл бинов?**

События приложения и раннеры — это не альтернативы DI, а механизмы позднего расширения жизненного цикла. Они позволяют выполнить код уже после того, как контейнер собрал основной граф зависимостей. Это особенно важно, когда часть инициализации не должна находиться в конструкторе или `@PostConstruct`, потому что относится скорее к “входу в рабочий режим”, чем к созданию самого объекта.

`CommandLineRunner` и `ApplicationRunner` подходят для запуска кода после старта приложения. Часто их используют для прогрева, проверки окружения, инициализации тестовых данных, запуска CLI-сценариев и иных задач, которые должны быть выполнены после сборки контекста. Разница между ними в основном в удобстве работы с аргументами запуска.

События вроде `ApplicationReadyEvent` дают ещё более точную точку расширения. Это полезно, если нужно запускать прогрев кэшей, фоновые процедуры или поздние проверки только после того, как всё приложение действительно готово. Такой подход помогает не перегружать ранние фазы жизненного цикла и делать старт более предсказуемым как для разработчика, так и для платформы развертывания.

**18. Что такое `SmartLifecycle` и когда он лучше обычных startup/shutdown-хуков?**

`SmartLifecycle` — это интерфейс для компонентов, у которых есть управляемый старт и управляемая остановка. В отличие от простых колбэков и раннеров, он даёт более точный контроль над фазами запуска и завершения, поддерживает автозапуск и позволяет контейнеру понимать, запущен ли компонент прямо сейчас. Это особенно важно для фоновых потребителей, listeners и сервисов с собственным циклом активности.

Обычные `@PostConstruct`, `@PreDestroy` и даже `ApplicationReadyEvent` хороши для простых случаев, но они слабее в сценариях, где есть сложный порядок старта и остановки нескольких фоновых компонентов. Например, потребитель сообщений нужно запускать только после прогрева зависимостей и останавливать раньше, чем будут закрыты внешние клиенты и базы данных. Именно здесь фазы `SmartLifecycle` дают полезную управляемость.

То есть `SmartLifecycle` нужен не для каждого сервиса, а для компонентов с настоящим operational lifecycle. Если класс просто предоставляет методы и не имеет собственной активной жизни, обычного lifecycle Spring достаточно. Но если компонент стартует воркеры, слушает очередь, управляет длительными задачами или должен участвовать в graceful shutdown по фазам, `SmartLifecycle` становится очень уместным инструментом.

**19. Как работает автоконфигурация Spring Boot и почему она должна быть “вежливой”?**

Автоконфигурация Spring Boot — это механизм, который автоматически поднимает бины на основе условий. Spring Boot анализирует classpath, свойства, уже зарегистрированные бины и другие сигналы, после чего подключает подходящую инфраструктуру. Так приложение получает DataSource, MVC-конфиг, сериализаторы, actuator-бины и множество других компонентов без ручной сборки каждого из них.

Ключевой принцип хорошей автоконфигурации — она должна помогать, а не навязываться. Поэтому корректные стартеры используют `@ConditionalOnClass`, `@ConditionalOnProperty`, `@ConditionalOnBean`, `@ConditionalOnMissingBean` и другие условия. Они не должны безусловно создавать свои бины там, где приложение уже определило собственные. В противном случае автоконфигурация начинает ломать пользовательский граф и делает поведение системы неочевидным.

Для прикладного разработчика это означает две вещи. Во-первых, нужно понимать, что многие бины в приложении появляются не “сами собой”, а из автоконфигурации. Во-вторых, при проектировании собственных стартеров следует соблюдать ту же вежливость: библиотека должна давать разумные значения по умолчанию, но уступать управление приложению. Именно так строится здоровая экосистема Spring Boot.

**20. Какие анти-паттерны чаще всего встречаются при работе с бинами и конфигурацией в Spring?**

Один из самых распространённых анти-паттернов — смешивание бизнес-логики и конфигурации. Когда в `@Configuration` начинают жить предметные правила, а в сервисах появляется ручное создание инфраструктурных клиентов через `new`, слои быстро путаются. В итоге проект труднее тестировать, сложнее читать и почти невозможно поддерживать единообразно.

Второй класс проблем связан с обходом контейнера. Сюда относятся Service Locator через `ApplicationContext.getBean(...)` в бизнес-коде, прямые вызовы `@Bean`-методов как обычной фабрики, попытки чинить архитектуру через хаотичный `@Lookup`, а также неправильное понимание прокси и self-invocation. Эти практики работают “как будто бы иногда”, но создают скрытые и плохо диагностируемые дефекты.

Третий большой анти-паттерн — игнорирование жизненного цикла. Тяжёлые операции в `@PostConstruct`, отсутствие `@PreDestroy`, неуправляемые фоновые процессы, бессистемное использование `prototype`, неясные имена бинов и неочевидные автоконфигурации делают приложение хрупким. В Spring хорошие практики вокруг бинов — это не формальность, а основа эксплуатационной устойчивости проекта.

# Теоретические материалы

1. Spring Framework: [Bean Overview](https://docs.spring.io/spring-framework/reference/core/beans/definition.html)
2. Spring Framework: [Classpath Scanning and Managed Components](https://docs.spring.io/spring-framework/reference/core/beans/classpath-scanning.html)
3. Spring Framework: [Basic Concepts: `@Bean` and `@Configuration`](https://docs.spring.io/spring-framework/reference/core/beans/java/basic-concepts.html)
4. Spring Framework: [Customizing the Nature of a Bean](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)
5. Spring Framework: [Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
6. Spring Framework: [Method Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-method-injection.html)
7. Spring Framework: [Container Extension Points](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
8. Spring Boot: [SpringApplication: `ApplicationRunner` и `CommandLineRunner`](https://docs.spring.io/spring-boot/reference/features/spring-application.html)
9. Spring Boot: [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
10. Spring Boot: [Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)
11. Spring Boot: [Externalized Configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
12. Spring Boot: [Actuator Endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)
13. Baeldung: [Injecting Prototype Beans into a Singleton Instance in Spring](https://www.baeldung.com/spring-inject-prototype-bean-into-singleton)
14. Baeldung: [Circular Dependencies in Spring](https://www.baeldung.com/circular-dependencies-in-spring)
15. Habr: [Жизненный цикл бина в Spring](https://habr.com/ru/articles/893614/)
16. Habr: [Spring-потрошитель: жизненный цикл Spring Framework](https://habr.com/ru/articles/720794/)
17. Habr: [Spring: Жизненный цикл бинов, методы init() и destroy()](https://habr.com/ru/articles/658273/)
18. Habr: [@Transactional в Spring под капотом](https://habr.com/ru/articles/532000/)

# Задачи

## 1. Собрать модуль уведомлений с разделением бизнес-кода и конфигурации

В проекте есть небольшой модуль уведомлений, который уже умеет отправлять сообщения по нескольким каналам. Сейчас весь код написан вперемешку: часть инфраструктуры создаётся вручную, биновые зависимости неявны, а выбор канала трудно расширять. Из-за этого модуль неудобно тестировать и почти невозможно аккуратно развивать, когда появляется ещё один тип доставки или дополнительные настройки клиента.

Нужно привести код к форме, где бизнес-логика живёт в сервисах, а техническая сборка объектов — в конфигурации. При этом ученику придётся не просто заменить пару строк, а спроектировать нормальный граф бинов: один бин по умолчанию, один бин по явному qualifier, плюс вынесенные инфраструктурные зависимости через `@Bean`.

* Стартовый код

```java
package demo.task1;

public class NotificationHttpClient {
    private final String baseUrl;
    private final int timeoutMillis;

    public NotificationHttpClient(String baseUrl, int timeoutMillis) {
        this.baseUrl = baseUrl;
        this.timeoutMillis = timeoutMillis;
    }

    public String post(String payload) {
        return "POST " + baseUrl + " timeout=" + timeoutMillis + " payload=" + payload;
    }
}

public class MessageTemplateEngine {
    public String render(String template, String user, String text) {
        return template
            .replace("{user}", user)
            .replace("{text}", text);
    }
}

public interface NotificationSender {
    String send(String user, String text);
}

public class EmailNotificationSender implements NotificationSender {
    private final NotificationHttpClient client;
    private final MessageTemplateEngine templateEngine;

    public EmailNotificationSender(NotificationHttpClient client, MessageTemplateEngine templateEngine) {
        this.client = client;
        this.templateEngine = templateEngine;
    }

    @Override
    public String send(String user, String text) {
        return client.post(templateEngine.render("[EMAIL] to={user}; text={text}", user, text));
    }
}

public class SmsNotificationSender implements NotificationSender {
    private final NotificationHttpClient client;

    public SmsNotificationSender(NotificationHttpClient client) {
        this.client = client;
    }

    @Override
    public String send(String user, String text) {
        return client.post("[SMS] to=" + user + "; text=" + text);
    }
}

public class NotificationService {
    private final NotificationSender sender;

    public NotificationService(NotificationSender sender) {
        this.sender = sender;
    }

    public String notifyUser(String user, String text) {
        return sender.send(user, text);
    }
}
```

* Что требуется

  * Зарегистрировать `NotificationHttpClient` и `MessageTemplateEngine` через `@Configuration` + `@Bean`.
  * Сделать `EmailNotificationSender` и `SmsNotificationSender` Spring-бинами.
  * Назначить email-канал реализацией по умолчанию через `@Primary`.
  * Зарегистрировать `NotificationService` как Spring-сервис через компонент-сканирование.
  * Добавить отдельный сервис `UrgentNotificationService`, который всегда использует SMS через `@Qualifier`.
  * Не создавать зависимости вручную через `new` внутри бизнес-кода.

### Решение

```java
package demo.task1;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

public class NotificationHttpClient {
    private final String baseUrl;
    private final int timeoutMillis;

    public NotificationHttpClient(String baseUrl, int timeoutMillis) {
        this.baseUrl = baseUrl;
        this.timeoutMillis = timeoutMillis;
    }

    public String post(String payload) {
        return "POST " + baseUrl + " timeout=" + timeoutMillis + " payload=" + payload;
    }
}

public class MessageTemplateEngine {
    public String render(String template, String user, String text) {
        return template
            .replace("{user}", user)
            .replace("{text}", text);
    }
}

public interface NotificationSender {
    String send(String user, String text);
}

@Component("emailSender")
@Primary
class EmailNotificationSender implements NotificationSender {
    private final NotificationHttpClient client;
    private final MessageTemplateEngine templateEngine;

    public EmailNotificationSender(NotificationHttpClient client, MessageTemplateEngine templateEngine) {
        this.client = client;
        this.templateEngine = templateEngine;
    }

    @Override
    public String send(String user, String text) {
        return client.post(templateEngine.render("[EMAIL] to={user}; text={text}", user, text));
    }
}

@Component("smsSender")
class SmsNotificationSender implements NotificationSender {
    private final NotificationHttpClient client;

    public SmsNotificationSender(NotificationHttpClient client) {
        this.client = client;
    }

    @Override
    public String send(String user, String text) {
        return client.post("[SMS] to=" + user + "; text=" + text);
    }
}

@Service
public class NotificationService {
    private final NotificationSender sender;

    public NotificationService(NotificationSender sender) {
        this.sender = sender;
    }

    public String notifyUser(String user, String text) {
        return sender.send(user, text);
    }
}

@Service
class UrgentNotificationService {
    private final NotificationSender smsSender;

    public UrgentNotificationService(@Qualifier("smsSender") NotificationSender smsSender) {
        this.smsSender = smsSender;
    }

    public String sendUrgent(String user, String text) {
        return smsSender.send(user, "[URGENT] " + text);
    }
}

@Configuration
class NotificationConfig {

    @Bean
    NotificationHttpClient notificationHttpClient() {
        return new NotificationHttpClient("https://notify.internal/api", 3000);
    }

    @Bean
    MessageTemplateEngine messageTemplateEngine() {
        return new MessageTemplateEngine();
    }
}
```

## 2. Реализовать словарь статусов с корректным жизненным циклом и поздним прогревом

В приложении есть словарь, который получает статусы из внешнего справочного сервиса и кэширует их в памяти. Сейчас старт клиента, загрузка данных и остановка ресурса нигде явно не оформлены, поэтому непонятно, когда соединение открывается, кто инициирует прогрев и что произойдёт при завершении приложения. Такой код может случайно начать работать “по обстоятельствам”, а не по контракту.

Нужно оформить решение как набор полноценных Spring-бинов с корректным lifecycle. Важно разделить лёгкую инициализацию ресурса и более тяжёлый прогрев данных: первое должно происходить во время жизни бина, а второе — уже после готовности приложения, когда основной контекст поднят.

* Стартовый код

```java
package demo.task2;

import java.util.HashMap;
import java.util.Map;

public class ReferenceClient {
    private boolean opened;

    public void open() {
        opened = true;
    }

    public Map<String, String> fetchStatuses() {
        if (!opened) {
            throw new IllegalStateException("Client is not opened");
        }
        Map<String, String> result = new HashMap<>();
        result.put("A", "ACTIVE");
        result.put("B", "BLOCKED");
        result.put("C", "CLOSED");
        return result;
    }

    public void close() {
        opened = false;
    }
}

public class StatusDictionary {
    private final ReferenceClient client;
    private final Map<String, String> cache = new HashMap<>();

    public StatusDictionary(ReferenceClient client) {
        this.client = client;
    }

    public String getStatus(String code) {
        return cache.get(code);
    }
}
```

* Что требуется

  * Зарегистрировать `ReferenceClient` и `StatusDictionary` как Spring-бины.
  * Открывать клиент в фазе инициализации.
  * На `ApplicationReadyEvent` загружать и кэшировать статусы.
  * На остановке приложения корректно закрывать ресурс.
  * Добавить прикладной сервис, который использует словарь и возвращает человекочитаемый статус.
  * Не смешивать бизнес-логику со служебным lifecycle-кодом.

### Решение

```java
package demo.task2;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

public class ReferenceClient {
    private boolean opened;

    public void open() {
        opened = true;
    }

    public Map<String, String> fetchStatuses() {
        if (!opened) {
            throw new IllegalStateException("Client is not opened");
        }
        Map<String, String> result = new HashMap<>();
        result.put("A", "ACTIVE");
        result.put("B", "BLOCKED");
        result.put("C", "CLOSED");
        return result;
    }

    public void close() {
        opened = false;
    }
}

public class StatusDictionary {
    private final ReferenceClient client;
    private final Map<String, String> cache = new HashMap<>();

    public StatusDictionary(ReferenceClient client) {
        this.client = client;
    }

    @PostConstruct
    public void init() {
        client.open();
    }

    @EventListener(ApplicationReadyEvent.class)
    public void warmUp() {
        cache.clear();
        cache.putAll(client.fetchStatuses());
    }

    public String getStatus(String code) {
        return cache.getOrDefault(code, "UNKNOWN");
    }

    @PreDestroy
    public void shutdown() {
        client.close();
    }
}

@Service
class CustomerStatusService {
    private final StatusDictionary dictionary;

    public CustomerStatusService(StatusDictionary dictionary) {
        this.dictionary = dictionary;
    }

    public String resolveStatusLabel(String code) {
        return switch (dictionary.getStatus(code)) {
            case "ACTIVE" -> "Клиент активен";
            case "BLOCKED" -> "Клиент заблокирован";
            case "CLOSED" -> "Клиент закрыт";
            default -> "Статус неизвестен";
        };
    }
}

@Configuration
class ReferenceConfig {

    @Bean
    ReferenceClient referenceClient() {
        return new ReferenceClient();
    }

    @Bean
    StatusDictionary statusDictionary(ReferenceClient referenceClient) {
        return new StatusDictionary(referenceClient);
    }
}
```

## 3. Построить маршрутизатор обработчиков через `Map<String, RequestHandler>`

Есть заготовка движка, который должен обрабатывать несколько типов бизнес-операций. Сейчас в системе предусмотрен только один обработчик, а логика выбора реализации вообще не оформлена. В ближайшем будущем появятся новые сценарии, поэтому код нужно перестроить так, чтобы добавление новой операции не требовало правок в центральном сервисе.

Нужно реализовать расширяемую схему на основе набора бинов-обработчиков. Ученик должен не просто дописать `if/else`, а спроектировать настоящий registry-подход: несколько Spring-бинов с осмысленными именами, автоматическая сборка их в карту и понятная ошибка при неизвестной операции.

* Стартовый код

```java
package demo.task3;

public interface RequestHandler {
    String handle(String payload);
}

public class CreateRequestHandler implements RequestHandler {
    @Override
    public String handle(String payload) {
        return "created:" + payload;
    }
}

public class UpdateRequestHandler implements RequestHandler {
    @Override
    public String handle(String payload) {
        return "updated:" + payload;
    }
}

public class RequestProcessingService {
    public String process(String operation, String payload) {
        throw new UnsupportedOperationException("implement me");
    }
}
```

* Что требуется

  * Сделать `CreateRequestHandler` и `UpdateRequestHandler` Spring-бинами с осмысленными именами.
  * Добавить ещё один обработчик `delete`.
  * Внедрить в сервис `Map<String, RequestHandler>`.
  * Реализовать роутинг по ключу операции.
  * Для неизвестной операции выбрасывать понятную ошибку с перечислением доступных вариантов.
  * Не использовать ручную регистрацию обработчиков через `new HashMap<>()` в бизнес-коде.

### Решение

```java
package demo.task3;

import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.Set;

public interface RequestHandler {
    String handle(String payload);
}

@Component("create")
class CreateRequestHandler implements RequestHandler {
    @Override
    public String handle(String payload) {
        return "created:" + payload;
    }
}

@Component("update")
class UpdateRequestHandler implements RequestHandler {
    @Override
    public String handle(String payload) {
        return "updated:" + payload;
    }
}

@Component("delete")
class DeleteRequestHandler implements RequestHandler {
    @Override
    public String handle(String payload) {
        return "deleted:" + payload;
    }
}

@Service
public class RequestProcessingService {
    private final Map<String, RequestHandler> handlers;

    public RequestProcessingService(Map<String, RequestHandler> handlers) {
        this.handlers = handlers;
    }

    public String process(String operation, String payload) {
        RequestHandler handler = handlers.get(operation);
        if (handler == null) {
            Set<String> supported = handlers.keySet();
            throw new IllegalArgumentException(
                "Unsupported operation '" + operation + "'. Supported operations: " + supported
            );
        }
        return handler.handle(payload);
    }
}
```

## 4. Переписать конфигурацию так, чтобы она была корректной при `proxyBeanMethods=false`

В проекте есть конфигурационный класс, который создаёт несколько связанных интеграционных клиентов. Сейчас зависимости описаны через прямые вызовы `@Bean`-методов друг из друга, и код случайно выглядит рабочим только пока никто не думает о реальной семантике конфигурации. При переводе в режим `proxyBeanMethods=false` такая схема становится опасной и может создавать новые объекты вместо контейнерных.

Нужно переписать конфигурацию на безопасный стиль: все зависимости должны быть выражены через параметры `@Bean`-методов. Дополнительно стоит выделить отдельный фасадный сервис, который будет работать сразу с несколькими интеграционными клиентами, чтобы итоговый граф выглядел как нормальная инфраструктурная сборка, а не набор случайных factory-вызовов.

* Стартовый код

```java
package demo.task4;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

public class AuthClient {
    private final String token;

    public AuthClient(String token) {
        this.token = token;
    }

    public String token() {
        return token;
    }
}

public class OrdersClient {
    private final AuthClient authClient;

    public OrdersClient(AuthClient authClient) {
        this.authClient = authClient;
    }

    public String call() {
        return "orders by " + authClient.token();
    }
}

public class BillingClient {
    private final AuthClient authClient;

    public BillingClient(AuthClient authClient) {
        this.authClient = authClient;
    }

    public String call() {
        return "billing by " + authClient.token();
    }
}

@Configuration(proxyBeanMethods = false)
public class IntegrationConfig {

    @Bean
    AuthClient authClient() {
        return new AuthClient("secret-token");
    }

    @Bean
    OrdersClient ordersClient() {
        return new OrdersClient(authClient());
    }

    @Bean
    BillingClient billingClient() {
        return new BillingClient(authClient());
    }
}
```

* Что требуется

  * Оставить `proxyBeanMethods=false`.
  * Убрать прямые вызовы `@Bean`-методов друг из друга.
  * Выразить зависимости через параметры методов.
  * Добавить сервис `IntegrationFacade`, который использует оба клиента.
  * Сохранить единый `AuthClient` для обоих клиентов.
  * Сделать итоговый код читаемым и безопасным для дальнейшей поддержки.

### Решение

```java
package demo.task4;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;

public class AuthClient {
    private final String token;

    public AuthClient(String token) {
        this.token = token;
    }

    public String token() {
        return token;
    }
}

public class OrdersClient {
    private final AuthClient authClient;

    public OrdersClient(AuthClient authClient) {
        this.authClient = authClient;
    }

    public String call() {
        return "orders by " + authClient.token();
    }
}

public class BillingClient {
    private final AuthClient authClient;

    public BillingClient(AuthClient authClient) {
        this.authClient = authClient;
    }

    public String call() {
        return "billing by " + authClient.token();
    }
}

@Service
class IntegrationFacade {
    private final OrdersClient ordersClient;
    private final BillingClient billingClient;

    public IntegrationFacade(OrdersClient ordersClient, BillingClient billingClient) {
        this.ordersClient = ordersClient;
        this.billingClient = billingClient;
    }

    public String healthcheck() {
        return ordersClient.call() + " | " + billingClient.call();
    }
}

@Configuration(proxyBeanMethods = false)
public class IntegrationConfig {

    @Bean
    AuthClient authClient() {
        return new AuthClient("secret-token");
    }

    @Bean
    OrdersClient ordersClient(AuthClient authClient) {
        return new OrdersClient(authClient);
    }

    @Bean
    BillingClient billingClient(AuthClient authClient) {
        return new BillingClient(authClient);
    }
}
```

## 5. Спроектировать мини-автоконфигурацию для аудита, которая не ломает пользовательский контекст

Нужно смоделировать небольшой стартер аудита для нескольких приложений. Если библиотека подключена в проект, она должна поднимать аудит только тогда, когда это явно разрешено настройкой. При этом библиотека не должна навязывать свою реализацию, если приложение уже объявило собственный бин для записи аудит-событий.

Это уже задача не про обычные сервисы, а про корректное поведение библиотеки внутри чужого контекста. Здесь важно показать, что автоконфигурация должна быть условной, переопределяемой и максимально “вежливой”, иначе она быстро превратится в источник конфликтов и неожиданных побочных эффектов.

* Стартовый код

```java
package demo.task5;

public interface AuditSink {
    void emit(String event);
}

public class ConsoleAuditSink implements AuditSink {
    @Override
    public void emit(String event) {
        System.out.println("[AUDIT] " + event);
    }
}

public class AuditService {
    private final AuditSink sink;

    public AuditService(AuditSink sink) {
        this.sink = sink;
    }

    public void audit(String event) {
        sink.emit(event);
    }
}
```

* Что требуется

  * Сделать автоконфигурационный класс для аудита.
  * Создавать `AuditSink` только при `audit.enabled=true`.
  * Использовать `@ConditionalOnMissingBean`, чтобы пользователь мог подменить стандартную реализацию.
  * Создавать `AuditService` только при наличии `AuditSink`.
  * Добавить пользовательский конфиг с собственной реализацией `AuditSink`.
  * Показать быстрый тест автоконфигурации через `ApplicationContextRunner`.

### Решение

```java
package demo.task5;

import org.junit.jupiter.api.Test;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.AutoConfigurations;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.test.context.runner.ApplicationContextRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import static org.assertj.core.api.Assertions.assertThat;

public interface AuditSink {
    void emit(String event);
}

public class ConsoleAuditSink implements AuditSink {
    @Override
    public void emit(String event) {
        System.out.println("[AUDIT] " + event);
    }
}

public class AuditService {
    private final AuditSink sink;

    public AuditService(AuditSink sink) {
        this.sink = sink;
    }

    public void audit(String event) {
        sink.emit(event);
    }
}

@AutoConfiguration
@ConditionalOnProperty(prefix = "audit", name = "enabled", havingValue = "true")
class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    AuditSink auditSink() {
        return new ConsoleAuditSink();
    }

    @Bean
    @ConditionalOnBean(AuditSink.class)
    @ConditionalOnMissingBean
    AuditService auditService(AuditSink sink) {
        return new AuditService(sink);
    }
}

@Configuration
class UserAuditConfig {

    @Bean
    AuditSink customAuditSink() {
        return event -> System.out.println("[CUSTOM-AUDIT] " + event);
    }
}

class AuditAutoConfigurationTest {

    private final ApplicationContextRunner runner = new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(AuditAutoConfiguration.class));

    @Test
    void shouldNotCreateBeansWhenDisabled() {
        runner.run(context -> {
            assertThat(context).doesNotHaveBean(AuditSink.class);
            assertThat(context).doesNotHaveBean(AuditService.class);
        });
    }

    @Test
    void shouldCreateDefaultBeansWhenEnabled() {
        runner.withPropertyValues("audit.enabled=true")
            .run(context -> {
                assertThat(context).hasSingleBean(AuditSink.class);
                assertThat(context).hasSingleBean(AuditService.class);
                assertThat(context.getBean(AuditSink.class)).isInstanceOf(ConsoleAuditSink.class);
            });
    }

    @Test
    void shouldRespectUserOverride() {
        runner.withPropertyValues("audit.enabled=true")
            .withUserConfiguration(UserAuditConfig.class)
            .run(context -> {
                assertThat(context).hasSingleBean(AuditSink.class);
                assertThat(context).hasSingleBean(AuditService.class);
                assertThat(context.getBean(AuditSink.class)).isNotInstanceOf(ConsoleAuditSink.class);
            });
    }
}
```

## 6. Исправить пакетную обработку так, чтобы она использовала `prototype`, `@Lookup` и корректный управляемый lifecycle

Есть сервис пакетной обработки строк, который для каждой записи должен создавать отдельный рабочий объект со своим временным состоянием. Сейчас этот объект создаётся через `new`, поэтому он вообще не участвует в контейнерном управлении. Дополнительно нужно запустить сам обработчик как управляемый фоновой компонент: он должен уметь стартовать и останавливаться по правилам контейнера, а не “жить сам по себе”.

Нужно совместить несколько идей из темы: выделить одноразовый рабочий объект в `prototype`-бин, научить основной сервис получать новый экземпляр через `@Lookup`, а фонового координатора оформить через `SmartLifecycle`. 

* Стартовый код

```java
package demo.task6;

import java.util.ArrayList;
import java.util.List;

public class RowTask {
    private final List<String> steps = new ArrayList<>();

    public void addStep(String step) {
        steps.add(step);
    }

    public List<String> steps() {
        return steps;
    }
}

public class BatchImportService {

    public List<List<String>> importRows(List<String> rows) {
        List<List<String>> result = new ArrayList<>();
        for (String row : rows) {
            RowTask task = new RowTask();
            task.addStep("start:" + row);
            task.addStep("validate:" + row);
            task.addStep("save:" + row);
            result.add(task.steps());
        }
        return result;
    }
}
```

* Что требуется

  * Сделать `RowTask` prototype-бином.
  * Сделать сервис импорта Spring-бином.
  * Получать новый `RowTask` для каждой строки через `@Lookup`.
  * Вынести обработку одной строки в отдельный метод.
  * Добавить фоновый координатор на `SmartLifecycle`, который умеет стартовать, обрабатывать входной батч и останавливаться.
  * Не использовать `new` для создания рабочих объектов внутри бизнес-логики.

### Решение

```java
package demo.task6;

import org.springframework.beans.factory.annotation.Lookup;
import org.springframework.context.SmartLifecycle;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Component
@Scope("prototype")
class RowTask {
    private final List<String> steps = new ArrayList<>();

    public void addStep(String step) {
        steps.add(step);
    }

    public List<String> steps() {
        return List.copyOf(steps);
    }
}

@Service
abstract class BatchImportService {

    public List<List<String>> importRows(List<String> rows) {
        List<List<String>> result = new ArrayList<>();
        for (String row : rows) {
            result.add(processSingleRow(row));
        }
        return result;
    }

    protected List<String> processSingleRow(String row) {
        RowTask task = newRowTask();
        task.addStep("start:" + row);
        task.addStep("validate:" + row);
        task.addStep("save:" + row);
        return task.steps();
    }

    @Lookup
    protected abstract RowTask newRowTask();
}

@Component
class BatchImportLifecycle implements SmartLifecycle {
    private final BatchImportService batchImportService;
    private volatile boolean running;

    public BatchImportLifecycle(BatchImportService batchImportService) {
        this.batchImportService = batchImportService;
    }

    @Override
    public void start() {
        running = true;
        List<String> batch = List.of("row-1", "row-2", "row-3");
        List<List<String>> result = batchImportService.importRows(batch);
        System.out.println("Imported batch: " + result);
    }

    @Override
    public void stop(Runnable callback) {
        try {
            System.out.println("Stopping batch import lifecycle");
            running = false;
        } finally {
            callback.run();
        }
    }

    @Override
    public void stop() {
        stop(() -> {});
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    @Override
    public boolean isAutoStartup() {
        return true;
    }

    @Override
    public int getPhase() {
        return 100;
    }
}
```


