---
layout: page
title: "Основы Spring: Inversion of Control & Dependency Injection"
permalink: /spring/ioc_di
---
# 0. Введение

## Что это

Инверсия управления (IoC) — это идея, при которой объект не управляет своим жизненным циклом и зависимостями сам, а отдаёт это на откуп внешнему «оркестратору». В Java-мире таким оркестратором выступает контейнер внедрения зависимостей (DI-container), например Spring IoC Container. Он создаёт объекты (бины), соединяет их между собой и управляет временем жизни.

Внедрение зависимостей (Dependency Injection, DI) — это конкретная техника реализации IoC: зависимости «вкалываются» извне через конструктор, сеттер или метод, вместо того чтобы порождаться изнутри через `new`. Так код становится слабосвязанным, гибко настраиваемым и «подменяемым» в тестах.

Spring Framework строит граф объектов по правилам, описанным в конфигурации (аннотации, Java-конфигурация или XML). Когда приложению нужен экземпляр сервиса, контейнер уже знает, какую реализацию отдать и как её сконструировать. Это развязывает руки для модульного проектирования и рефакторинга.

В контексте бизнес-разработки IoC/DI — это не про «магические аннотации», а про дисциплину архитектуры. Мы разделяем «что» (контракт) и «как» (реализация), отдаём соединение слоёв инфраструктуре и получаем предсказуемость.

Для JVM-стека (Java/Kotlin, Spring Boot) DI — стандарт де-факто. Вокруг него построены механизмы профилей, конфигурационных свойств, автоконфигурации «стартеров», тестовых срезов. Без IoC современный Spring-проект трудно представить.

В повседневной работе вы чаще взаимодействуете не с «контейнером» напрямую, а с привычными аннотациями `@Component`, `@Service`, `@Repository` и конструкторной инъекцией. Но полезно помнить: под капотом всегда граф и его жизненный цикл.

**Код (Java): минимальная конструкторная инъекция**

```java
package demo.ioc.intro;

import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;

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

**Код (Kotlin): то же**

```kotlin
package demo.ioc.intro

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

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

---

## Зачем это

Главная мотивация — управляемая связность. Когда зависимости приходят извне, мы свободно подменяем их на фейковые или мок-объекты, тестируем куски системы изолированно и эволюционируем дизайн без каскадной переплавки.

Вторая причина — конфигурируемость. Один и тот же код может работать с разными реализациями: реальный платежный шлюз в проде, фейковый — на стенде, «медленный» — в нагрузочных тестах. Контейнер подставит нужный бин по профилю или квалификатору.

Третья — повторное использование. Контракт остаётся стабильным, реализации множатся под конкретные сценарии. Мы избегаем «снежного кома» условий внутри класса и распределяем поведение по стратегиям.

Четвёртая — жизненный цикл. DI-контейнер отвечает за создание, кэширование singleton-ов, очистку ресурсов при останове. Это особенно важно для тяжёлых клиентов (пулы соединений, HTTP-клиенты, драйверы).

Пятая — безопасность и наблюдаемость «по умолчанию». Автоконфигурации и бин-постпроцессоры могут автоматически навешивать обёртки: метрики, логирование, валидацию, ретраи. Вы подключаете поведение композиционно, не трогая бизнес-код.

Шестая — прозрачность намерений. Конструктор класса становится «контрактом зависимостей»: достаточно посмотреть, что требуется на вход, чтобы понять роль компонента и его границы.

**Код (Java): две реализации под разные профили**

```java
package demo.ioc.why;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

interface PaymentGateway { boolean pay(long cents); }

@Component @Profile("prod")
class RealGateway implements PaymentGateway {
  @Override public boolean pay(long cents) { /* call real PSP */ return true; }
}

@Component @Profile("dev")
class FakeGateway implements PaymentGateway {
  @Override public boolean pay(long cents) { return cents < 10_00; }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.ioc.why

import org.springframework.context.annotation.Profile
import org.springframework.stereotype.Component

interface PaymentGateway { fun pay(cents: Long): Boolean }

@Component @Profile("prod")
class RealGateway : PaymentGateway {
    override fun pay(cents: Long) = true
}

@Component @Profile("dev")
class FakeGateway : PaymentGateway {
    override fun pay(cents: Long) = cents < 1_000
}
```

---

## Где используется

DI применим везде, где есть слои и интеграции: веб-контроллеры, доменные сервисы, репозитории, клиенты внешних систем, обработчики сообщений. В микросервисах это «кислород», позволяющий держать код чистым и заменяемым.

В интеграциях с Kafka/DB/HTTP DI делает клиентов управляемыми, единообразно настраиваемыми через свойства. Конструкторная инъекция заставляет явно «признаться» в зависимостях: подключение, сериализаторы, ретраи — всё видно.

В модульных монолитах DI ведёт границы модулей: «порты и адаптеры». Порт — интерфейс домена, адаптер — реализация для конкретной инфраструктуры (REST/DB/FS). Это позволяет менять инфраструктуру без ломки домена.

В тестах DI ускоряет цикл feedback: вы подменяете адаптеры на заглушки и проверяете чистую бизнес-логику. Слой инфраструктуры тестируется отдельно контрактными/интеграционными тестами.

В библиотеках и «стартерах» DI используется для автоконфигурации: поставщик даёт бины и условия их создания, потребитель подключает зависимость — и «получает» готовую интеграцию.

В enterprise-контуре DI облегчает кросс-срезы: логирование, аудит, безопасность, обсервабилити добавляются «снаружи» аспектами/обёртками, а не пришиваетcя к каждому сервису вручную.

**Код (Java): порт и адаптер**

```java
package demo.ioc.where;

import org.springframework.stereotype.Component;

public interface SmsPort { void send(String phone, String text); }

@Component
class TwilioAdapter implements SmsPort {
  @Override public void send(String phone, String text) { /* call Twilio */ }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.ioc.where

import org.springframework.stereotype.Component

interface SmsPort { fun send(phone: String, text: String) }

@Component
class TwilioAdapter : SmsPort {
    override fun send(phone: String, text: String) { /* call Twilio */ }
}
```

---

## Какие проблемы/задачи решает

Первая проблема — жёсткие зависимости через `new`. Такой код невозможно подменить в тестах и трудно конфигурировать. DI устраняет это, отдавая создание контейнеру.

Вторая — каскадная связность. Когда сервис сам создаёт «полдерева» объектов, любое изменение тянет за собой серию правок. DI расплющивает граф: каждый класс описывает только своё локальное окружение.

Третья — «магические» синглтоны и глобальное состояние. С DI по умолчанию вы инъектите зависимости как поля, а не обращаетесь к глобальным утилитам. Это снижает неявные эффекты и гонки.

Четвёртая — сложность инициализации. Контейнер может выполнять валидацию и пост-обработку, гарантировать порядок, life-cycle, hooks на старт/останов — то, что руками часто делается неряшливо.

Пятая — профили/фичефлаги. DI позволяет переключать реализацию по окружению и условиям, не «зашивая» это в код условными блоками. Конфигурация остаётся внешней.

Шестая — расширяемость. Появилась новая стратегия? Просто добавьте бин и дайте контейнеру выбрать по ключу/квалификатору. Базовый код не трогаем.

**Код (Java): избавляемся от `new`**

```java
package demo.ioc.solve;

import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;

interface Mailer { void send(String to, String body); }

@Component
class SmtpMailer implements Mailer {
  @Override public void send(String to, String body) { /* SMTP send */ }
}

@Service
class ReportService {
  private final Mailer mailer;
  public ReportService(Mailer mailer) { this.mailer = mailer; }
  public void sendReport(String to) { mailer.send(to, "Report"); }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.ioc.solve

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface Mailer { fun send(to: String, body: String) }

@Component
class SmtpMailer : Mailer {
    override fun send(to: String, body: String) { /* SMTP send */ }
}

@Service
class ReportService(private val mailer: Mailer) {
    fun sendReport(to: String) = mailer.send(to, "Report")
}
```

---

# 1. Проблема, которую решают IoC/DI

## Жёсткие зависимости через `new`, трудная подмена и тестирование, каскадная связность.

Когда класс сам создаёт свои зависимости через `new`, он «замыкает» на себе конкретику. Такой код плохо тестируется: чтобы проверить бизнес-логику, нужно поднимать реальные подключения или переписывать код под тест. Это ломает TDD и делает рефакторинг болезненным.

Жёсткие зависимости повышают связность: изменение одной реализации ведёт к каскадным правкам в местах, где она создаётся. В больших проектах такой снежный ком быстро превращается в «хрупкую систему».

Скрывшийся внутри `new` конструктор часто тащит конфигурацию: логины, таймауты, флаги. Эти параметры оказываются «зашитыми» и разносятся копипастой, что неуправляемо и небезопасно.

Кроме тестов страдает расширяемость. Чтобы ввести новую стратегию, приходится влезать в существующие классы и плодить `if/else`. Это противоречит принципу открытости/закрытости (OCP) и быстро гниёт.

DI решает проблему тем, что заставляет смотреть на зависимости как на «внешний контракт». Код перестаёт порождать мир вокруг себя и становится компонуемым кирпичом.

Наконец, убирается скрытая инициализация: контейнер явно строит граф, валидирует, поднимает и закрывает ресурсы. Жизненный цикл предсказуем и прозрачен.

**Код (Java): до/после**

```java
// Плохо
class BadService {
  private final SmtpMailer mailer = new SmtpMailer();
  void run() { mailer.send("a@b", "hi"); }
}

// Хорошо
class GoodService {
  private final Mailer mailer;
  GoodService(Mailer mailer) { this.mailer = mailer; }
  void run() { mailer.send("a@b", "hi"); }
}
```

**Код (Kotlin): до/после**

```kotlin
// Плохо
class BadService {
    private val mailer = SmtpMailer()
    fun run() = mailer.send("a@b", "hi")
}

// Хорошо
class GoodService(private val mailer: Mailer) {
    fun run() = mailer.send("a@b", "hi")
}
```

---

## Инверсия управления: объект не создаёт зависимости сам — их предоставляет контейнер.

Инверсия управления меняет ответ на вопрос «кто кем управляет». Раньше объект управлял своими зависимостями; теперь — контейнер управляет объектом. Это сдвиг ответственности, а не просто «ещё одна аннотация».

В Spring за создание и связывание отвечает `ApplicationContext`. Вы объявляете, что класс — компонент, а его зависимости — параметры конструктора. Контейнер найдёт подходящие бины и передаст их при создании.

Такой подход делает код декларативным: «мне нужен `Mailer`», — без уточнения «какой именно». Конкретика решается конфигурацией: профили, условия, квалификаторы.

Инверсия управления распространяется на жизнь объекта: хуки жизненного цикла (`@PostConstruct`, `@PreDestroy`), проксирование, транзакции, аспекты. Код перестаёт управлять тем, что не относится к доменной логике.

Важно, что IoC работает одинаково в Java и Kotlin: различается лишь синтаксис конструктора. Суть — то же самое: зависимости приходят «снаружи».

Именно IoC открывает путь к автоконфигурации: сторонняя библиотека может «подложить» нужные бины при наличии классов на classpath и свойств. Вы подключаете стартер — и интеграция «заводится» без ручного `new`.

**Код (Java): IoC в действии**

```java
package demo.ioc.control;

import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;

interface Clock { long now(); }

@Component
class SystemClock implements Clock { public long now() { return System.currentTimeMillis(); } }

@Service
class TimeService {
  private final Clock clock;
  public TimeService(Clock clock) { this.clock = clock; }
  public boolean isEvenSecond() { return (clock.now()/1000) % 2 == 0; }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.ioc.control

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface Clock { fun now(): Long }

@Component
class SystemClock : Clock { override fun now() = System.currentTimeMillis() }

@Service
class TimeService(private val clock: Clock) {
    fun isEvenSecond() = (clock.now() / 1000) % 2L == 0L
}
```

---

## Выигрыш: тестируемость, модульность, возможность конфигурировать «снаружи».

DI делает unit-тесты банальными: подставляете фейк реализации интерфейса — и проверяете чистую логику. Никаких сетевых вызовов, БД и «магии» — только контракт и поведение.

Модульность улучшается: границы «порт/адаптер» становятся физическими интерфейсами, реализациям проще жить в отдельных модулях и зависеть лишь от контракта.

Конфигурация выходит наружу: таймауты, URL, ключи, флаги фич — свойства, а не «магические константы». Один бинарь — много окружений, профили готовы «из коробки».

Композиция становится дешёвой: добавили новую стратегию — подключили как бин. Контейнер выберет нужную по ключу/квалификатору или инъектит коллекцию стратегий целиком.

Наблюдаемость встраивается естественно: прокси/аспекты навешиваются на бины, добавляя метрики/trace без правки бизнес-кода. Это «системная» польза IoC.

И, наконец, снижается входной порог для новых разработчиков: «что нужно классу» видно в конструкторе. Меньше скрытых глобалей — меньше сюрпризов.

**Код (Java): лёгкий юнит-тест с подменой зависимости**

```java
package demo.ioc.win;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class TimeServiceTest {
  @Test void evenSecond() {
    Clock fake = () -> 2_000L;
    TimeService s = new TimeService(fake);
    assertTrue(s.isEvenSecond());
  }
}
```

**Код (Kotlin): то же (JUnit 5)**

```kotlin
package demo.ioc.win

import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test

class TimeServiceTest {
    @Test fun evenSecond() {
        val fake = Clock { 2_000L }
        val s = TimeService(fake)
        assertTrue(s.isEvenSecond())
    }
}
```

---

# 2. Модель зависимостей в приложении

## Контракты (интерфейсы) vs реализации: ослабляем связность по типам и по времени.

Контракт — это обещание поведения («отправь SMS», «получи курс валют»), но не способ его достижения. Реализация — конкретный способ. Разрыв между ними снижает связность: потребителю всё равно, как это сделано.

Связность по времени — это возможность менять реализацию без перекомпиляции потребителя. Мы поставляем новый модуль с реализацией, а потребитель — лишь повторно стартует с новыми бин-дефинициями.

Интерфейсы держат границы пакета/модуля. Доменные сервисы зависят от портов, а адаптеры — от внешних библиотек/клиентов. Таким образом, внешний мир «течёт вверх» только через интерфейсы.

В Kotlin интерфейсы и data-классы позволяют дополнительно фиксировать контракт на уровне nullability и sealed-иерархий. Это снимает часть рисков «неправильной» реализации.

Важно помнить: интерфейс не должен «протекать» инфраструктурой. Не кладите туда `ResponseEntity`, `ResultSet` или `HttpStatus`. Контракт — чистый и предметный.

Если реализаций несколько, то контейнер поможет с выбором: `@Primary`, `@Qualifier`, инъекция коллекций. Контракт остаётся прежним, вариативность — в конфигурации.

**Код (Java): контракт и две реализации**

```java
package demo.ioc.contract;

public interface ExchangeRatePort {
  double rate(String from, String to);
}
```

```java
package demo.ioc.contract;

import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component @Profile("fast")
class CachedExchangeRate implements ExchangeRatePort {
  @Override public double rate(String from, String to) { return 90.0; }
}

@Component @Profile("accurate")
class HttpExchangeRate implements ExchangeRatePort {
  @Override public double rate(String from, String to) { /* call HTTP */ return 91.2; }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.ioc.contract

interface ExchangeRatePort { fun rate(from: String, to: String): Double }

@org.springframework.stereotype.Component
@org.springframework.context.annotation.Profile("fast")
class CachedExchangeRate : ExchangeRatePort {
    override fun rate(from: String, to: String) = 90.0
}

@org.springframework.stereotype.Component
@org.springframework.context.annotation.Profile("accurate")
class HttpExchangeRate : ExchangeRatePort {
    override fun rate(from: String, to: String) = 91.2
}
```

---

## Границы «домена — инфраструктуры»: где DI особенно критичен (репозитории, клиенты, сервисы, стратегии).

На границе домена и инфраструктуры особенно легко «протечь». DI удерживает линию: домен говорит через порты, инфраструктура адаптирует внешние детали под этот язык.

Репозитории — классический пример: домену нужен интерфейс сохранения/поиска, а конкретика JPA/SQL/драйвера — в адаптере. DI склеит их в одном месте, но миры останутся разделёнными.

Клиенты внешних систем тоже должны быть адаптерами. Доменные сервисы знают «отослать уведомление», а не «сформировать HTTP-запрос c OAuth2 и заголовками». Это снимает хрупкость.

Стратегии — ещё один плацдарм DI. Валидация/расчёт/тарификация часто имеют несколько правил. Реализации лежат рядом, выбираются по ключу и инъектятся коллекцией.

На уровне сервисов DI сохраняет чистоту: вместо вызова `new` и ручного связывания мы просто просим нужные порты. В тестах подставляем фейки и проверяем домен в изоляции.

Такое разделение упрощает «горизонтальные» задачи — логирование, метрики, ретраи — их можно навесить на адаптеры и не трогать домен.

**Код (Java): порт репозитория и адаптер на JPA)**

```java
package demo.ioc.boundary;

public interface UserRepoPort {
  void save(User u);
  User find(String id);
}
```

```java
package demo.ioc.boundary;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Component;

interface UserJpa extends JpaRepository<UserEntity, String> {}

@Component
class UserRepoAdapter implements UserRepoPort {
  private final UserJpa jpa;
  UserRepoAdapter(UserJpa jpa) { this.jpa = jpa; }
  @Override public void save(User u) { jpa.save(UserEntity.from(u)); }
  @Override public User find(String id) { return jpa.findById(id).map(UserEntity::toDomain).orElseThrow(); }
}
```

**Код (Kotlin): то же)**

```kotlin
package demo.ioc.boundary

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Component

interface UserRepoPort {
    fun save(u: User)
    fun find(id: String): User
}

interface UserJpa : JpaRepository<UserEntity, String>

@Component
class UserRepoAdapter(private val jpa: UserJpa) : UserRepoPort {
    override fun save(u: User) { jpa.save(UserEntity.from(u)) }
    override fun find(id: String) = jpa.findById(id).map { it.toDomain() }.orElseThrow()
}
```

---

# 3. Виды внедрения зависимостей

## Через конструктор (предпочтительно): обязательные, иммутабельные зависимости, простые тесты.

Конструкторная инъекция делает зависимости обязательными и видимыми. Объект нельзя создать «полуготовым», а значит — меньше состояний и меньше скрытых дефектов.

Поля можно объявить `final` (в Kotlin — `val`), что предотвращает подмену ссылок после инициализации. Это повышает потокобезопасность и упрощает reasoning.

Тесты облегчаются до предела: создаёте класс, передаёте фейк — и вперёд. Никаких контейнеров/контекстов для unit-тестов не нужно.

Конструктор — «документация» зависимостей. Чем он толще, тем громче сигнал: класс делает слишком много, пора распиливать. Так DI помогает с рефакторингом.

Spring по умолчанию использует единственный публичный конструктор для автосвязывания. Аннотация `@Autowired` на конструкторе не нужна, если он один.

Если зависимость необязательна, подумайте, действительно ли она такая. Часто это запах дизайна: лучше распилить на две роли, чем держать `Optional` в конструкторе.

**Код (Java): конструкторная инъекция)**

```java
package demo.di.ctor;

import org.springframework.stereotype.Service;

@Service
public class BillingService {
  private final PaymentGateway gateway;
  public BillingService(PaymentGateway gateway) { this.gateway = gateway; }
  public boolean charge(long cents) { return gateway.pay(cents); }
}

interface PaymentGateway { boolean pay(long cents); }
```

**Код (Kotlin): то же)**

```kotlin
package demo.di.ctor

import org.springframework.stereotype.Service

interface PaymentGateway { fun pay(cents: Long): Boolean }

@Service
class BillingService(private val gateway: PaymentGateway) {
    fun charge(cents: Long) = gateway.pay(cents)
}
```

---

## Через сеттер/метод: опциональные зависимости, поздняя инициализация, риск неполной конфигурации.

Сеттерная инъекция допускает «позднее» присоединение зависимостей. Иногда это оправдано: опциональные расширения, циклы (которые лучше не допускать), интеграции, включаемые по флагам.

Однако сеттеры возвращают нас к изменяемости: поле можно заменить в рантайме, а объект может жить «полуинициализированным», если сеттер не был вызван. Это риск, если не контролировать порядок.

Сеттер удобно комбинировать с `@Lazy` или `ObjectProvider` для тяжёлых зависимостей. Мы не тратим ресурсы, пока не понадобилось.

Если всё-таки используете сеттеры, добавьте защиту: валидацию состояния при старте, условия, проверку «задвинуто/не задвинуто». Лучше — пересобрать дизайн под конструктор.

В инфраструктурном коде (например, бин-постпроцессоры) сеттеры встречаются чаще — там жизненный цикл специфичен. Но бизнес-код — конструктор.

Смешивать «часть через конструктор, часть через сеттер» стоит только при веских причинах; иначе вы теряете ясность.

**Код (Java): сеттер-инъекция + @Lazy)**

```java
package demo.di.setter;

import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Service;

@Service
public class AuditService {
  private @Lazy Auditor auditor;
  public void setAuditor(@Lazy Auditor auditor) { this.auditor = auditor; }
  public void audit(String event) { if (auditor != null) auditor.write(event); }
}

interface Auditor { void write(String event); }
```

**Код (Kotlin): то же)**

```kotlin
package demo.di.setter

import org.springframework.context.annotation.Lazy
import org.springframework.stereotype.Service

interface Auditor { fun write(event: String) }

@Service
class AuditService {
    @Lazy
    lateinit var auditor: Auditor
    fun audit(event: String) { if (this::auditor.isInitialized) auditor.write(event) }
}
```

---

## Полевая инъекция: коротко почему не использовать в проде (тестируемость, финальность, прокси).

Полевая инъекция (`@Autowired` на поле) кажется лаконичной, но скрывает зависимости и усложняет тестирование. Вы не можете создать объект обычным конструктором — нужно поднимать контейнер или лазить рефлексией.

Поля не являются `final`, значит ссылку можно заменить, а объект оказывается более хрупким для конкурентного доступа. Это противоречит идее иммутабельных зависимостей.

Проксирование и AOP с полевой инъекцией иногда дают сюрпризы в порядке инициализации. Конструктор предсказуемее: сначала зависимости, потом — объект.

В Kotlin полевая инъекция часто тянет за собой `lateinit var`, а это добавляет новый класс ошибок — «не инициализировано до первого доступа».

Полевая инъекция нарушает «самодокументацию» через конструктор. Чтобы понять, чем живёт класс, нужно идти по полям, а не посмотреть одну сигнатуру.

Правило простое: **не используйте** полевую инъекцию в проде. Оставьте её для крайне специфических инфраструктурных кейсов или не используйте вовсе.

**Код (Java): антипример и исправление)**

```java
// Антипример
class Bad {
  @org.springframework.beans.factory.annotation.Autowired Mailer m;
}

// Исправление
class Good {
  private final Mailer m;
  public Good(Mailer m) { this.m = m; }
}
```

**Код (Kotlin): антипример и исправление)**

```kotlin
// Антипример
class Bad {
    @org.springframework.beans.factory.annotation.Autowired
    lateinit var m: Mailer
}

// Исправление
class Good(private val m: Mailer)
```

---

## Kotlin-нюансы: `constructor`-инъекция, `lateinit` и риски, nullability как контракт.

В Kotlin конструкторная инъекция особенно приятна: свойства задаются через первичный конструктор с `val`, что фиксирует иммутабельность зависимостей.

`lateinit` для зависимостей — красный флаг: вы откладываете инициализацию и открываете двери NPE, если перепутали порядок. Лучше просить зависимость в конструкторе или использовать `ObjectProvider`.

Nullability — мощный инструмент. Интерфейс может возвращать `T?`, а зависимость может быть `Foo?` только если это действительно опционально. Так контракт «зашивается» в типы.

Из-за финальности классов/методов в Kotlin тестовые мок-фреймворки требуют `allOpen`/`mockito-inline` или использовать свободно стоящие фейки. DI не мешает: чаще всего хватает фейков.

Data-классы и sealed-иерархии хорошо дружат с портами и стратегиями: чётко выраженные варианты, явное покрытие `when` без `else`.

В остальном DI-паттерны те же: конструктор, интерфейсы на границах, адаптеры снаружи.

**Код (Kotlin): конструктор + nullability как контракт)**

```kotlin
package demo.di.kotlin

import org.springframework.stereotype.Service

interface Notifier { fun send(message: String): Boolean }

@Service
class OrderService(private val notifier: Notifier?) {
    fun place(): Boolean {
        // опциональная зависимость
        return notifier?.send("placed") ?: true
    }
}
```

**Код (Java): аналог с Optional)**

```java
package demo.di.kotlin;

import java.util.Optional;
import org.springframework.stereotype.Service;

interface Notifier { boolean send(String message); }

@Service
public class OrderService {
  private final Optional<Notifier> notifier;
  public OrderService(Optional<Notifier> notifier) { this.notifier = notifier; }
  public boolean place() { return notifier.map(n -> n.send("placed")).orElse(true); }
}
```

---

# 4. Каркас DI в Spring (в общих чертах)

## Контейнер управляет графом объектов и их зависимостями.

Spring IoC Container — это фабрика + реестр бинов. На старте он сканирует класспат, регистрирует определения бинов, решает зависимости и создаёт экземпляры по «правилам» (scope, lifecycle, проксирование).

Граф — это ориентированный ациклический граф (в идеале) зависимостей. Контейнер пытается создать бины в порядке, удовлетворяющем конструкторные зависимости, и применяет пост-процессоры (включая автопрокси).

Именно контейнер решает «кто именно» будет внедрён: по типу, по квалификатору, по имени, по условиям (`@Conditional`, профили). Код об этом не знает и знать не должен.

Лайф-циклы (`singleton`, `prototype`, web-scopes) управляются контейнером. Бизнес-код получает уже готовый экземпляр и не заботится о пересоздании/кешировании.

Контейнер также умеет «подмешивать» инфраструктуру: транзакции, безопасность, кеширование. Это делается через прокси/аспекты, которые контейнер создаёт поверх реальных бинов.

При остановке контейнер вызывает `@PreDestroy` и освобождает ресурсы. Это критично для пула соединений, файловых дескрипторов, клиентов.

**Код (Java): жизненный цикл)**

```java
package demo.di.container;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
class LifecycleBean {
  @PostConstruct void init() { System.out.println("init"); }
  @PreDestroy void destroy() { System.out.println("destroy"); }
}
```

**Код (Kotlin): то же)**

```kotlin
package demo.di.container

import jakarta.annotation.PostConstruct
import jakarta.annotation.PreDestroy
import org.springframework.stereotype.Component

@Component
class LifecycleBean {
    @PostConstruct fun init() = println("init")
    @PreDestroy fun destroy() = println("destroy")
}
```

---

## Виды компонентов: сервисы/адаптеры/порты — договоримся о роли каждого.

Порт — интерфейс домена, через который он «говорит» наружу. Адаптер — реализация порта, использующая конкретную технологию. Сервис — доменная координация, композиция портов, без инфраструктурных деталей.

Такой словарь упрощает навигацию: по названию пакета/класса понятно, где домен, где интеграция, где композиция. Это снижает ментальную нагрузку в больших кодовых базах.

В Spring это выражается аннотациями: `@Service` для сервисов, `@Component`/`@Repository` для адаптеров (в зависимости от роли), интерфейсы портов лежат отдельно и не аннотируются.

Сервис не должен создавать адаптеры и знать о конкретных клиентах. Он просит порты в конструкторе и оперирует предметными моделями.

Адаптеры зависят «вниз» от клиентов/драйверов, но реализуют «вверх» порт. Таким образом, внешний мир изолирован доменным контрактом.

Чёткие роли помогают внедрять кросс-срезы: кеширование — на адаптере репозитория, ретраи — на HTTP-клиенте, аудит — в сервисе (как бизнес-события).

**Код (Java): порт/адаптер/сервис)**

```java
package demo.di.roles;

public interface EmailPort { void send(String to, String body); }
```

```java
package demo.di.roles;

import org.springframework.stereotype.Component;
@Component
class SmtpAdapter implements EmailPort {
  @Override public void send(String to, String body) { /* smtp */ }
}
```

```java
package demo.di.roles;

import org.springframework.stereotype.Service;

@Service
class UserService {
  private final EmailPort email;
  UserService(EmailPort email) { this.email = email; }
  public void register(String to) { /* domain */ email.send(to, "welcome"); }
}
```

**Код (Kotlin): то же)**

```kotlin
package demo.di.roles

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface EmailPort { fun send(to: String, body: String) }

@Component
class SmtpAdapter : EmailPort {
    override fun send(to: String, body: String) { /* smtp */ }
}

@Service
class UserService(private val email: EmailPort) {
    fun register(to: String) { email.send(to, "welcome") }
}
```

---

## Где будут жить определения бинов — обсудим в следующей теме; здесь фиксируем лишь идею.

Определения бинов могут появляться из сканирования компонентов (`@ComponentScan`), из Java-конфигурации (`@Configuration` + `@Bean`) и из автоконфигураций стартеров. На уровне архитектуры важно не «как», а «что» попадает в контейнер.

Практический принцип: в доменном модуле — только порты и сервисы; адаптеры живут в инфраструктурных модулях. Это упрощает переиспользование и делает зависимости направленными.

Границы модулей подчёркиваются пакетами и зависимостями сборки. Порты не должны зависеть от фреймворка (в идеале — чистый Java/Kotlin), адаптеры — могут.

В тестах определения бинов можно подменять тестовыми конфигурациями. Это ещё одна грань DI: конфигурация меняется по контексту исполнения.

Способ «как положить бин в контейнер» мы детально разберём в отдельной главе: аннотации, `@Bean`, условные бины, профили, порядок. Сейчас важно понять идею графа и ролей.

Именно эта договорённость команды (что считать портом, где хранить адаптеры) даёт долгосрочную отдачу от DI и снижает «энтропию» проекта.

**Код (Java): пример @Bean-конфигурации)**

```java
package demo.di.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
class AppConfig {
  @Bean Mailer mailer() { return new SmtpMailer(); }
}

interface Mailer { void send(); }
class SmtpMailer implements Mailer { public void send() {} }
```

**Код (Kotlin): то же)**

```kotlin
package demo.di.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class AppConfig {
    @Bean fun mailer(): Mailer = SmtpMailer()
}

interface Mailer { fun send() }
class SmtpMailer : Mailer { override fun send() {} }
```
# 5. Выбор реализации и «множественность» бинов

## Несколько кандидатов одного типа: `@Qualifier`, имя бина, `@Primary` — правила разрешения

Когда в контейнере присутствует несколько бинов одного типа, Spring должен понять, какой именно внедрять. По умолчанию он ищет по типу; если кандидатов несколько, включается разрешение по **имени параметра** конструктора и по **имени бина**. Если и это не помогает — возникает ошибка «NoUniqueBeanDefinitionException». Правильная реакция разработчика — не «выключить автопровайд» и не перейти на ручные обращение к контексту, а **сделать выбор явным**.

Первый способ — пометить «дефолтную» реализацию `@Primary`. Такой бин будет побеждать в столкновении кандидатов по типу. Это удобно, когда в 80% случаев подходит одна реализация, а остальные — экзотика по профилям или квалификаторам. При этом `@Primary` не мешает использовать `@Qualifier`: явный квалификатор всегда сильнее.

Второй способ — `@Qualifier`. Он вводит **ключ** (строковый ярлык), которым помечают и **поставщика** (бин), и **потребителя** (параметр конструктора). В продакшне это самый надёжный приём для выбора реализации, особенно когда вы внедряете сразу несколько похожих адаптеров (несколько HTTP-клиентов, несколько валидаторов).

Третий механизм — **имя бина**. Если параметр конструктора называется так же, как и имя одного из кандидатов, Spring предпочтёт его. Это работает «само», но не рекомендуется как единственный контракт: переименование параметра или класса легко сломает внедрение. Лучше быть явным с `@Qualifier`.

Четвёртый — **профили** и **условия**. Если реализация предназначена строго для определённого окружения, её проще скрыть/показать через `@Profile("prod")` или `@ConditionalOnProperty(...)`. Тогда в рантайме не будет «нескольких кандидатов», и контейнер справится по типу.

Главное правило: **чем больше реализаций — тем более явным должен быть выбор**. Выберите одну стратегию командой: `@Primary` для дефолта + `@Qualifier` для конкретики. Это упростит чтение кода и снимет бо́льшую часть «магии».

**Код (Java): `@Primary` и `@Qualifier`**

```java
package demo.di.multi;

import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

interface TaxCalculator { long computeCents(long amountCents); }

@Component("flat")
@Primary
class FlatTax implements TaxCalculator {
  @Override public long computeCents(long amountCents) { return Math.round(amountCents * 0.1); }
}

@Component("progressive")
class ProgressiveTax implements TaxCalculator {
  @Override public long computeCents(long amountCents) { return amountCents > 10_000 ? 2_000 : 500; }
}

@Service
class Billing {
  private final TaxCalculator defaultTax;
  private final TaxCalculator progressive;
  public Billing(TaxCalculator defaultTax,
                 @org.springframework.beans.factory.annotation.Qualifier("progressive") TaxCalculator progressive) {
    this.defaultTax = defaultTax; // FlatTax из-за @Primary
    this.progressive = progressive; // выбран явно
  }
  public long taxDefault(long amount) { return defaultTax.computeCents(amount); }
  public long taxProgressive(long amount) { return progressive.computeCents(amount); }
}
```

**Код (Kotlin): `@Primary` и `@Qualifier`**

```kotlin
package demo.di.multi

import org.springframework.context.annotation.Primary
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface TaxCalculator { fun computeCents(amountCents: Long): Long }

@Component("flat")
@Primary
class FlatTax : TaxCalculator {
    override fun computeCents(amountCents: Long) = Math.round(amountCents * 0.1).toLong()
}

@Component("progressive")
class ProgressiveTax : TaxCalculator {
    override fun computeCents(amountCents: Long) = if (amountCents > 10_000) 2_000 else 500
}

@Service
class Billing(
    private val defaultTax: TaxCalculator, // FlatTax по @Primary
    @Qualifier("progressive") private val progressive: TaxCalculator
) {
    fun taxDefault(amount: Long) = defaultTax.computeCents(amount)
    fun taxProgressive(amount: Long) = progressive.computeCents(amount)
}
```

---

## Инъекция коллекций/карт: `List<T>`, `Map<String,T>`, упорядочивание через `@Order`

Вместо «выбери единственный бин» часто удобнее получить **все** реализации контракта и решить, как их применить. В Spring можно инъектить `List<T>` — контейнер отдаст упорядоченный список всех бинов типа `T`. Порядок по умолчанию не гарантируется, но его можно задать через `@Order` на классах или `Ordered`-интерфейс.

Инъекция `Map<String, T>` даёт ещё больше контроля: ключ — **имя бина**, значение — сам бин. Это удобно для реализации паттерна Strategy по ключу (см. следующий пункт), а также для программной инициализации маршрутизаторов.

Упорядочивание через `@Order` важно, когда есть **цепочки** обработчиков: например, фильтры валидации, пост-обработчики событий, пайплайны миграций. Явное число в `@Order(…)` фиксирует позицию, и этот контракт виден прямо на классе.

Если порядок вычисляется динамически (например, по конфигурации), можно инъектить `List<T>` без упорядочивания и сортировать самостоятельно на старте: по приоритетам, весам, расписанию. Это хорошо работает вместе с `@ConfigurationProperties`.

Не злоупотребляйте «всеми реализациями»: если у вас 12 «стратегий», но применяется всегда одна по ключу — честнее инъектить `Map<String,Strategy>` и выбирать нужную, а не перебирать все двенадцать в рантайме.

И наконец, помните, что коллекции — это **снимок на момент построения контекста**. Динамически «подложить» стратегию «после старта» без перезапуска контекста нельзя (если только вы не пишете собственный загрузчик плагинов).

**Код (Java): `List` и `Map` c `@Order`**

```java
package demo.di.collections;

import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

interface Step { String run(String in); }

@Component @Order(1)
class TrimStep implements Step { public String run(String in) { return in.trim(); } }

@Component @Order(2)
class LowerStep implements Step { public String run(String in) { return in.toLowerCase(); } }

@Component("reverse")
@Order(3)
class ReverseStep implements Step { public String run(String in) { return new StringBuilder(in).reverse().toString(); } }

@Service
class Pipeline {
  private final java.util.List<Step> steps;
  private final java.util.Map<String, Step> byName;
  public Pipeline(java.util.List<Step> steps, java.util.Map<String, Step> byName) {
    this.steps = steps; this.byName = byName;
  }
  public String processAll(String s) {
    String out = s;
    for (Step st : steps) out = st.run(out);
    return out;
  }
  public String processNamed(String s, String name) {
    return byName.get(name).run(s);
  }
}
```

**Код (Kotlin): `List` и `Map` c `@Order`**

```kotlin
package demo.di.collections

import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface Step { fun run(input: String): String }

@Component @Order(1)
class TrimStep : Step { override fun run(input: String) = input.trim() }

@Component @Order(2)
class LowerStep : Step { override fun run(input: String) = input.lowercase() }

@Component("reverse") @Order(3)
class ReverseStep : Step { override fun run(input: String) = input.reversed() }

@Service
class Pipeline(
    private val steps: List<Step>,
    private val byName: Map<String, Step>
) {
    fun processAll(s: String) = steps.fold(s) { acc, st -> st.run(acc) }
    fun processNamed(s: String, name: String) = byName.getValue(name).run(s)
}
```

---

## Паттерн Strategy через DI: «подсунуть» правильную реализацию по ключу

Strategy — классика: у вас есть **семейство алгоритмов** с единым интерфейсом, и вы выбираете нужный в зависимости от контекста (ключа). DI делает это естественно: инъектируем `Map<String, Strategy>`, где ключ — **имя бина** или **квалификатор**, а значение — реализация.

Ключ может приходить из запроса (`type=paypal`), из конфигурации клиента или быть рассчитанным в процессе. Это избавляет от скучного `switch`/`if`-каскада и держит код **открытым к расширениям**: новая стратегия — новый бин, основной код не меняется.

Для читаемости и безопасности лучше договориться об именовании ключей: например, `@Component("paypal")`, `@Component("stripe")`. Тогда карта будет именно такой: `{ "paypal" -> PayPalStrategy, "stripe" -> StripeStrategy }`. Альтернатива — собственная аннотация-квалификатор с `@Qualifier("pay:stripe")`.

Если реализация зависит от окружения, совместите Strategy с `@Profile` или `@ConditionalOnProperty`: карта будет содержать только доступные стратегии, и избыточных веток не будет.

Сложные маршрутизации удобно выделять в отдельный **Router/Registry**: он держит карту, валидирует ключи, логирует выбор, обрабатывает «дефолт», если ключ неизвестен. Сервису остаётся «попросить» у роутера нужную реализацию.

Обратите внимание на **трассировку**: при выборе стратегии полезно логировать `strategyKey` и имя бина. Это помогает диагностикам, когда поведение зависит от данных.

**Код (Java): Strategy с картой реализаций**

```java
package demo.di.strategy;

import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

interface PaymentStrategy { String charge(long cents); }

@Component("paypal")
class PaypalStrategy implements PaymentStrategy { public String charge(long c) { return "paypal:" + c; } }

@Component("stripe")
class StripeStrategy implements PaymentStrategy { public String charge(long c) { return "stripe:" + c; } }

@Service
class StrategyRouter {
  private final java.util.Map<String, PaymentStrategy> strategies;
  public StrategyRouter(java.util.Map<String, PaymentStrategy> strategies) { this.strategies = strategies; }
  public String pay(String key, long cents) {
    PaymentStrategy s = strategies.getOrDefault(key, strategies.get("stripe"));
    return s.charge(cents);
  }
}
```

**Код (Kotlin): Strategy с картой реализаций**

```kotlin
package demo.di.strategy

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface PaymentStrategy { fun charge(cents: Long): String }

@Component("paypal")
class PaypalStrategy : PaymentStrategy { override fun charge(cents: Long) = "paypal:$cents" }

@Component("stripe")
class StripeStrategy : PaymentStrategy { override fun charge(cents: Long) = "stripe:$cents" }

@Service
class StrategyRouter(private val strategies: Map<String, PaymentStrategy>) {
    fun pay(key: String, cents: Long): String =
        strategies.getOrElse(key) { strategies.getValue("stripe") }.charge(cents)
}
```

---

# 6. Работа с опциональными и ленивыми зависимостями

## `Optional<T>`/nullable как сигнал «может отсутствовать»

Иногда зависимость действительно **может отсутствовать**: например, аудит включается только в prod, а в dev — нет. В Java такой контракт удобно выразить через `Optional<T>` в конструкторе. Контейнер либо положит бин внутрь, либо передаст `Optional.empty()`, и это явный сигнал потребителю.

В Kotlin ту же идею отражает **nullable-тип**. Когда параметр конструктора объявлен как `Foo?`, контейнер попытается найти бин `Foo`, а при отсутствии — передаст `null`. Сигнатура конструктора становится документацией: «зависимость не обязательна».

Важно не превращать `Optional`/`?` в «инфекцию» по коду. Опциональность имеет смысл на **границе**, где вы принимаете решение «использовать/пропустить». Внутрь доменной логики лучше передавать уже «точно существующие» зависимости, либо декорировать их «no-op» реализацией.

Частый паттерн — «no-op adapter»: реализуйте интерфейс так, чтобы он ничего не делал, и регистрируйте его, когда реальной зависимости нет. Тогда конструктор остаётся чистым (`Foo` вместо `Optional<Foo>`), а код — однообразным.

Отдельная тема — тесты. Там вы сами решаете, давать ли реализацию. Если тест проверяет поведение при «аудите выключен» — просто не регистрируйте бин/передайте `null`.

Опциональность не равна «ленивости». Даже если бин есть, он может быть тяжёлым; для этого у Spring есть `ObjectProvider` и `@Lazy` (следующий пункт).

**Код (Java): `Optional` в конструкторе и no-op реализация**

```java
package demo.di.optional;

import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

interface AuditSink { void emit(String event); }

@Component
class NoopAudit implements AuditSink { public void emit(String event) { /* no-op */ } }

@Service
class Checkout {
  private final AuditSink audit;
  public Checkout(java.util.Optional<AuditSink> auditOpt) {
    this.audit = auditOpt.orElseGet(NoopAudit::new);
  }
  public void pay() { audit.emit("pay"); }
}
```

**Код (Kotlin): nullable зависимость**

```kotlin
package demo.di.optional

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface AuditSink { fun emit(event: String) }

@Component
class NoopAudit : AuditSink { override fun emit(event: String) { /* no-op */ } }

@Service
class Checkout(private val audit: AuditSink?) {
    fun pay() { (audit ?: NoopAudit()).emit("pay") }
}
```

---

## Поставщики и ленивость: `ObjectProvider<T>`/`Provider<T>` — отложенное разрешение, безопасные циклы

`ObjectProvider<T>` — это «ленивый» поставщик бина из контейнера. В отличие от `Optional<T>`, он **не создаёт** зависимость на старте: вы можете позвать `getIfAvailable()` в нужный момент, а до этого — контейнер даже не будет строить целевой бин. Это полезно для тяжёлых клиентов (HTTP, кэши), запускаемых «по требованию».

Кроме ленивости, `ObjectProvider` помогает **разорвать цикл** зависимостей. Если конструкторы двух классов требуют друг друга, получим цикл. Через провайдера можно попросить зависимость **после** создания объекта, например в методе, где она действительно нужна.

Java/Jakarta `javax.inject.Provider<T>` — более общий интерфейс, поддерживаемый Spring; он чуть проще по API (только `get()`), без удобств `getIfAvailable`. В большинстве случаев `ObjectProvider` комфортнее.

Поставщики удобно использовать для «опциональной регистрации» обработчиков. Можно запросить `provider.stream().forEach(...)` — он отдаст все подходящие бины, если они есть, и ноль — если нет. Это элегантно для расширяемых точек.

Важно помнить о **семантике scope**. Если целевой бин — singleton, все вызовы `get()` вернут один и тот же экземпляр. Если prototype — каждый вызов породит новый. Это мощный инструмент, но используйте его осознанно.

И наконец, **не прячьте** через поставщика базовый архитектурный запах. Если вам нужны провайдеры для обычных сервисных зависимостей, скорее всего стоит переосмыслить дизайн (упростить граф, ввести порт/фасад).

**Код (Java): `ObjectProvider` и «ленивый» клиент**

```java
package demo.di.provider;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;

interface ExpensiveClient { String call(); }

@Service
class UseClient {
  private final ObjectProvider<ExpensiveClient> provider;
  public UseClient(ObjectProvider<ExpensiveClient> provider) { this.provider = provider; }
  public String doWork(boolean need) {
    if (!need) return "skip";
    ExpensiveClient c = provider.getIfAvailable(() -> () -> "fallback");
    return c.call();
  }
}
```

**Код (Kotlin): `ObjectProvider`**

```kotlin
package demo.di.provider

import org.springframework.beans.factory.ObjectProvider
import org.springframework.stereotype.Service

interface ExpensiveClient { fun call(): String }

@Service
class UseClient(private val provider: ObjectProvider<ExpensiveClient>) {
    fun doWork(need: Boolean): String =
        if (!need) "skip" else provider.getIfAvailable { ExpensiveClient { "fallback" } }.call()
}
```

---

## `@Lazy`: когда оправдано (тяжёлые/редко используемые зависимости), когда лучше пересобрать дизайн

Аннотация `@Lazy` заставляет Spring проксировать зависимость и отложить её фактическое создание до **первого обращения**. Это снижает время старта и потребление памяти, когда зависимость действительно используется редко (например, «админ-клиент»).

`@Lazy` работает на уровне **зависимости** (параметра конструктора/поля) и на уровне **бина** (на классе/методе `@Bean`). В первом случае вы получаете прокси вместо реального объекта, во втором — сам бин создаётся «по требованию», когда кто-то попросит его из контейнера.

Есть ограничения. Для классов без интерфейсов Spring использует CGLIB-прокси. В Kotlin по умолчанию классы `final`; чтобы класс проксировался, он должен быть `open` или внедряться как интерфейс. На практике лучший совет — **зависеть от интерфейсов**.

`@Lazy` — это **инструмент оптимизации**, а не средство исправления циклов. Если вы ставите `@Lazy` только чтобы «обмануть» цикл в конструкторах, лучше поправить архитектуру (ввести порт, разнести ответственность).

Ленивая зависимость усложняет диагностику: ошибки инициализации могут проявиться не на старте, а при первом же обращении. Компенсация — хорошая прогревочная фаза и тесты, которые «трогают» ленивые бины заранее.

Наконец, `@Lazy` не отменяет «дороговизну» самого вызова. Если клиент тяжёлый по CPU/IO, лучше вынести его в отдельный компонент и управлять жизненным циклом явно, а не создавать каждый раз.

**Код (Java): `@Lazy` на зависимости**

```java
package demo.di.lazy;

import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Service;

interface AdminClient { String cmd(String x); }

@Service
class Audit {
  private final AdminClient admin;
  public Audit(@Lazy AdminClient admin) { this.admin = admin; }
  public String rotate() { return admin.cmd("rotate-keys"); }
}
```

**Код (Kotlin): `@Lazy` на зависимости**

```kotlin
package demo.di.lazy

import org.springframework.context.annotation.Lazy
import org.springframework.stereotype.Service

interface AdminClient { fun cmd(x: String): String }

@Service
class Audit(@Lazy private val admin: AdminClient) {
    fun rotate() = admin.cmd("rotate-keys")
}
```

---

# 7. Противопоказания и анти-паттерны

## Service Locator (ручной `ApplicationContext.getBean(...)`) — скрытая связность, тесты ломаются

Service Locator — соблазн «взять контейнер под мышку» и просить из него бины по мере надобности. В Spring это часто делается через `ApplicationContextAware` и вызовы `getBean(...)`. В итоге зависимость **прячется**: по конструктору не видно, что класс тянет ещё три сервиса, а тестам приходится поднимать весь контекст.

Такой код нарушает инверсию управления: объект снова сам решает, что ему нужно, и когда это создать. Вместо «зависимости в конструкторе» появляется «произвольный поиск в реестре». Это усложняет reasoning, ломает модульность и мешает рефакторингу.

Тестируемость страдает: для unit-теста вам уже не хватает фейков — приходится мокать сам контекст или поднимать Spring, что превращает unit в интеграционный тест. Скорость обратной связи падает.

Service Locator ослабляет контракт: компилятор больше не поможет. Ошибся именем бина — получите ошибку **в рантайме**. С конструкторной инъекцией такие ошибки ловятся на старте контекста, а часто — ещё на этапе компиляции.

Бывает, что локатор — единственный выход в инфраструктурном слое (например, в стороннем callback, куда DI не достаёт). Но даже там стоит завернуть доступ в **узкий адаптер** и инъектить уже его, а не контекст.

Итог: **не используйте Service Locator в бизнес-коде**. Вынесите зависимости в конструктор, а локатор оставьте крайним и редким исключением инфраструктуры.

**Код (Java): антипример и исправление)**

```java
// Плохо: Service Locator
class BadMailer {
  private final org.springframework.context.ApplicationContext ctx;
  BadMailer(org.springframework.context.ApplicationContext ctx) { this.ctx = ctx; }
  void send(String to) {
    var smtp = ctx.getBean(SmtpClient.class);
    smtp.push(to);
  }
}

// Хорошо: конструкторная инъекция
class GoodMailer {
  private final SmtpClient smtp;
  GoodMailer(SmtpClient smtp) { this.smtp = smtp; }
  void send(String to) { smtp.push(to); }
}

interface SmtpClient { void push(String to); }
```

**Код (Kotlin): антипример и исправление)**

```kotlin
// Плохо
class BadMailer(private val ctx: org.springframework.context.ApplicationContext) {
    fun send(to: String) { ctx.getBean(SmtpClient::class.java).push(to) }
}

// Хорошо
class GoodMailer(private val smtp: SmtpClient) {
    fun send(to: String) = smtp.push(to)
}

interface SmtpClient { fun push(to: String) }
```

---

## `new` внутри бизнес-кода для зависимостей — потеря управляемости, дубли настроек

Создавая зависимости напрямую через `new`, вы теряете преимущества DI: общую конфигурацию, жизненный цикл, сквозные аспекты (метрики, ретраи), тестируемость. Особенно болезненно это в клиентах: таймауты, пулы, TLS должны настраиваться централизованно, а не разъезжаться по коду.

Часто «локальный `new`» означает копипаст настроек: сегодня выставили `connectTimeout=500ms` в одном месте, забыли в другом — и поведение нестабильно. С контейнером клиент настраивается **один раз**, а внедряется везде одинаковый.

Ещё одна потеря — наблюдаемость. Автоконфигурации и бин-постпроцессоры умеют автоматически вешать метрики/трейсинг на создаваемые бины. При `new` этого не происходит, и вы теряете видимость.

В тестах прямой `new` мешает подменам. Вместо фейка вы получаете реальный HTTP-вызов, и приходится «закручивать» PowerMock/ByteBuddy или прокладывать лишние уровни абстракций.

Лучшее решение — вынести создание зависимости в конфигурацию (`@Bean`) и инъектить интерфейс в бизнес-код. Внутри конфигурации можно использовать все преимущества Spring (профили, свойства, условность).

В редких случаях локальный `new` оправдан для **легковесных** объектов без внешних эффектов (например, форматтеров). Но для клиентов/репозиториев — это красный флаг.

**Код (Java): антипример и исправление с `@Bean`)**

```java
// Плохо
class BadClientUser {
  String call() {
    var http = java.net.http.HttpClient.newHttpClient();
    // ...
    return "ok";
  }
}

// Хорошо
@org.springframework.context.annotation.Configuration
class HttpConfig {
  @org.springframework.context.annotation.Bean
  java.net.http.HttpClient httpClient() {
    return java.net.http.HttpClient.newBuilder()
      .connectTimeout(java.time.Duration.ofMillis(800)).build();
  }
}

class GoodClientUser {
  private final java.net.http.HttpClient http;
  GoodClientUser(java.net.http.HttpClient http) { this.http = http; }
  String call() { return "ok"; }
}
```

**Код (Kotlin): антипример и исправление)**

```kotlin
// Плохо
class BadClientUser {
    fun call(): String {
        val http = java.net.http.HttpClient.newHttpClient()
        return "ok"
    }
}

// Хорошо
@org.springframework.context.annotation.Configuration
class HttpConfig {
    @org.springframework.context.annotation.Bean
    fun httpClient(): java.net.http.HttpClient =
        java.net.http.HttpClient.newBuilder()
            .connectTimeout(java.time.Duration.ofMillis(800)).build()
}

class GoodClientUser(private val http: java.net.http.HttpClient) {
    fun call(): String = "ok"
}
```

---

## Статические «хелперы» с состоянием — проблемная совместимость с DI и тестами

Статические одиночки со внутренним состоянием — **антипод** DI. Их нельзя переопределить в тестах без тяжёлых инструментов, они создают скрытую глобальную точку синхронизации и ломают потокобезопасность.

Частая боль — «статический кэш» внутри `Util`. В многопоточной среде это грозит гонками, а при раскатке нескольких инстансов сервиса — рассинхронизацией «кэшей каждого инстанса». DI предлагает явный компонент-кэш с управляемым временем жизни.

Другой нюанс — конфигурация. Статическим хелперам сложно передать настройки и секреты «по правилам». В итоге значения шьются в код или читаются из системных пропертей как попало.

DI решает проблему, превращая статический хелпер в **бин** с интерфейсом. Его можно настроить, завернуть декораторами (метрики/логирование), подменить на фейк в тестах.

В Kotlin особенно легко злоупотребить `object`-синглтонами. Они отличные для **без-состояния** утилит (pure functions), но как только появляется состояние — лучше мигрировать на бин.

Резюме: не храните состояние в статике. Пусть оно живёт в компоненте, который контейнер создаёт и за который отвечает.

**Код (Java): антипример и исправление)**

```java
// Плохо
class GlobalCache {
  private static final java.util.Map<String,String> map = new java.util.concurrent.ConcurrentHashMap<>();
  static void put(String k, String v) { map.put(k,v); }
  static String get(String k) { return map.get(k); }
}

// Хорошо
interface Cache { void put(String k, String v); String get(String k); }

@org.springframework.stereotype.Component
class LocalCache implements Cache {
  private final java.util.Map<String,String> map = new java.util.concurrent.ConcurrentHashMap<>();
  public void put(String k, String v) { map.put(k,v); }
  public String get(String k) { return map.get(k); }
}
```

**Код (Kotlin): антипример и исправление)**

```kotlin
// Плохо
object GlobalCache {
    private val map = java.util.concurrent.ConcurrentHashMap<String,String>()
    fun put(k: String, v: String) { map[k] = v }
    fun get(k: String) = map[k]
}

// Хорошо
interface Cache { fun put(k: String, v: String); fun get(k: String): String? }

@org.springframework.stereotype.Component
class LocalCache : Cache {
    private val map = java.util.concurrent.ConcurrentHashMap<String,String>()
    override fun put(k: String, v: String) { map[k] = v }
    override fun get(k: String) = map[k]
}
```

---

## Тонкие места: циклические зависимости в конструкторах, «божественные» сервисы

Циклические зависимости в конструкторах (`A` требует `B`, а `B` требует `A`) — частая ловушка. Spring падает на старте с ошибкой цикла. В 99% случаев это сигнал к **рефакторингу**: выделите порт/фасад и разорвите цикл.

Один из приёмов — вынести «общий» функционал в третий компонент `C`, от которого зависят оба. Другой — заменить одну из зависимостей на **событие/коллбек**, чтобы зависимость была направленной: `A` публикует событие, `B` подписывается.

Бывает искушение «починить» цикл `@Lazy`. Так можно запустить приложение, но логика останется «взаимозавязана», и вы получите хрупкость и сложность reasoning. Лучше переписать.

«Божественные» сервисы (god objects) — ещё один запах. Когда у класса 12 зависимостей и он «делает всё», даже DI не спасёт: тесты громоздки, ответственность размыта. Сигнал к распилу на **агрегаторы** более узкой ответственности.

Правильный дизайн снижает энтропию: короткие конструкторы, зависимости по абстракциям, простая направленность графа «снаружи внутрь». Любое локальное усложнение (новая стратегия, новая интеграция) должно добавляться без изменения базовых классов.

Поддерживайте дисциплину ревью: циклы и распухшие сервисы должны «звенеть» в PR. Это дешевле, чем чинить в проде.

**Код (Java): цикл и разрыв через фасад)**

```java
// Плохо: A -> B, B -> A
class A { A(B b) {} }
class B { B(A a) {} }

// Хорошо: A -> Facade, B -> Facade
interface Facade { void op(); }

@org.springframework.stereotype.Component
class FacadeImpl implements Facade { public void op() {} }

class A2 { A2(Facade f) {} }
class B2 { B2(Facade f) {} }
```

**Код (Kotlin): цикл и разрыв)**

```kotlin
// Плохо
class A(b: B)
class B(a: A)

// Хорошо
interface Facade { fun op() }

@org.springframework.stereotype.Component
class FacadeImpl : Facade { override fun op() {} }

class A2(private val f: Facade)
class B2(private val f: Facade)
```

---

# 8. Контракты и композиция

## «Зависимости по абстракциям»: интерфейсы на границах модулей и слоёв

Золотое правило DI: зависимости направлены **к абстракциям**, а не к деталям. Домены зависят от портов (интерфейсов), а детали — от доменов (реализуя эти интерфейсы). Это и есть «Dependency Inversion» из SOLID.

Такой подход уменьшает дрейф инфраструктуры. Замена HTTP-клиента, смена брокера, перенос из PostgreSQL в другой источник — всё это затрагивает адаптеры, но не переплавляет домен.

Границы «порт/адаптер» должны быть **видимы в коде**: отдельные пакеты/модули, минимальные зависимости. Хорошая проверка — можно ли собрать доменный модуль без Spring и внешних библиотек.

Контракты должны говорить языком домена. Никаких `ResponseEntity`, `HttpHeaders`, `ResultSet` на границе. Примитивы, value-объекты, доменные типы — так тестировать и развивать проще.

Для версионирования контрактов лучше предпочесть **расширение** поверх **изменения**: новый порт V2, новая стратегия — а старые остаются, пока не будет миграции потребителей.

Командные договорённости критичны: что считаем портом, где лежат адаптеры, как их называть. Это сохраняет структуру годами.

**Код (Java): порт/адаптер/домен)**

```java
package demo.di.contracts;

public interface SmsPort { void send(String phone, String text); }
```

```java
package demo.di.contracts;

import org.springframework.stereotype.Component;
@Component
class AwsSmsAdapter implements SmsPort {
  @Override public void send(String phone, String text) { /* AWS SNS */ }
}
```

```java
package demo.di.contracts;

import org.springframework.stereotype.Service;
@Service
class RegistrationService {
  private final SmsPort sms;
  public RegistrationService(SmsPort sms) { this.sms = sms; }
  public void register(String phone) { /* domain */ sms.send(phone, "code"); }
}
```

**Код (Kotlin): порт/адаптер/домен)**

```kotlin
package demo.di.contracts

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface SmsPort { fun send(phone: String, text: String) }

@Component
class AwsSmsAdapter : SmsPort { override fun send(phone: String, text: String) { /* AWS SNS */ } }

@Service
class RegistrationService(private val sms: SmsPort) {
    fun register(phone: String) { sms.send(phone, "code") }
}
```

---

## Компоновка через конструкторные графы: явная передача всего нужного, отказ от скрытых глобалей

Конструктор — «портал» для зависимостей. Когда всё нужное приходит через него, композиция становится **явной**: агрегатор-сервис точно демонстрирует, что ему требуется для работы. Это снижает когнитивную нагрузку и облегчает рефакторинг.

Агрегаторы должны быть **тонкими**: координировать, а не реализовывать бизнес-логику. Логику держим в специализированных доменных сервисах/стратегиях; агрегатор просто вызывает их в нужном порядке.

Отказ от скрытых глобалей означает, что вы не лезете за почтой/кэшем/временем через статические хелперы. Всё это — зависимости. Даже «чтение времени» лучше вынести в интерфейс `Clock`, чтобы тесты были детерминированными.

Композиция легко проверяется unit-тестами: подставили фейки зависимостей — проверили сценарий. Ошибки wiring’а ловятся на старте контекста, а не в неожиданных местах рантайма.

Горизонтальные срезы (логирование, метрики, идемпотентность) лучше инкапсулировать в адаптерах/декораторах, а не прошивать в агрегатор. DI позволяет удобно оборачивать бины.

И, наконец, композиция должна **меняться редко**. Если агрегатор регулярно пухнет, это сигнал, что вы в него кладёте «не ту» ответственность — пора дробить.

**Код (Java): агрегатор, собирающий несколько портов)**

```java
package demo.di.compose;

import org.springframework.stereotype.Service;

interface PricingPort { long price(String sku); }
interface DiscountPort { long discount(String sku); }
interface StockPort { boolean available(String sku); }

@Service
class CheckoutService {
  private final PricingPort pricing;
  private final DiscountPort discount;
  private final StockPort stock;
  public CheckoutService(PricingPort p, DiscountPort d, StockPort s) {
    this.pricing = p; this.discount = d; this.stock = s;
  }
  public long total(String sku) {
    if (!stock.available(sku)) throw new IllegalStateException("no stock");
    return pricing.price(sku) - discount.discount(sku);
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package demo.di.compose

import org.springframework.stereotype.Service

interface PricingPort { fun price(sku: String): Long }
interface DiscountPort { fun discount(sku: String): Long }
interface StockPort { fun available(sku: String): Boolean }

@Service
class CheckoutService(
    private val pricing: PricingPort,
    private val discount: DiscountPort,
    private val stock: StockPort
) {
    fun total(sku: String): Long {
        require(stock.available(sku)) { "no stock" }
        return pricing.price(sku) - discount.discount(sku)
    }
}
```

---

## Версионирование контрактов: расширение интерфейсов без лома

Контракты живут дольше реализаций. Менять их «в лоб» — значит ломать потребителей. Лучше добавлять **новые** методы с дефолтной реализацией (в Java 8+), либо вводить **новый порт V2**, а старый поддерживать до миграции.

Подход «V2 расширяет V1» хорош, когда новая функциональность — надстройка над старой. Тогда старые потребители продолжают работать, а новые — получают дополнительный метод. Реализация может имплементировать оба интерфейса.

Когда изменение меняет семантику (например, инициирует другой протокол), лучше объявить **совершенно новый порт**. Это честнее и безопаснее, чем «перегрузить» старый метод несовместимым смыслом.

На уровне DI вы можете держать параллельно обе реализации и выбирать по профилям/квалификаторам. В переходный период сервисы будут мигрировать, а вы — собирать метрики использования каждой версии.

Документируйте сроки и план миграции: контракт — это API для внутренних команд. Договорённость снижает хаос и залипания на «вечной поддержке V1».

Финальный шаг — **удаление** старого контракта. Не держите «мертвые» интерфейсы — они мешают читабельности и вводят в заблуждение.

**Код (Java): V1/V2 и внедрение)**

```java
package demo.di.version;

public interface CurrencyPortV1 { double rate(String from, String to); }

public interface CurrencyPortV2 extends CurrencyPortV1 {
  default double inverse(String from, String to) { return 1.0 / rate(from, to); }
}
```

```java
package demo.di.version;

import org.springframework.stereotype.Component;

@Component("v2")
class HttpCurrencyAdapter implements CurrencyPortV2 {
  @Override public double rate(String from, String to) { return 90.0; }
}

@Component("v1")
class CachedCurrencyAdapter implements CurrencyPortV1 {
  @Override public double rate(String from, String to) { return 89.5; }
}
```

```java
package demo.di.version;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
class PricingService {
  private final CurrencyPortV2 currency;
  public PricingService(@Qualifier("v2") CurrencyPortV2 currency) { this.currency = currency; }
  public double priceIn(String from, String to) { return currency.inverse(from, to); }
}
```

**Код (Kotlin): V1/V2 и внедрение)**

```kotlin
package demo.di.version

import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.stereotype.Component
import org.springframework.stereotype.Service

interface CurrencyPortV1 { fun rate(from: String, to: String): Double }
interface CurrencyPortV2 : CurrencyPortV1 {
    fun inverse(from: String, to: String) = 1.0 / rate(from, to)
}

@Component("v2")
class HttpCurrencyAdapter : CurrencyPortV2 {
    override fun rate(from: String, to: String) = 90.0
}

@Component("v1")
class CachedCurrencyAdapter : CurrencyPortV1 {
    override fun rate(from: String, to: String) = 89.5
}

@Service
class PricingService(@Qualifier("v2") private val currency: CurrencyPortV2) {
    fun priceIn(from: String, to: String) = currency.inverse(from, to)
}
```

---

# 9. DI и тестируемость

## Подмена зависимостей в unit-тестах (моки/фейки): DI делает это тривиальным

Главная выгода DI — возможность **скормить** классу фейковые зависимости. Это превращает unit-тест в быстрый и детерминированный: никакой сети, БД и «плавающих» часов.

Фейки лучше моков там, где поведение легко описать вручную и важно сохранить реалистичность. Моки удобны, когда требуется проверка взаимодействий (сколько раз вызван метод, с какими аргументами). В любом случае, конструкторная инъекция делает и то, и другое простым.

Важно, чтобы тест не поднимал Spring, когда это не нужно. Конструируйте класс руками, передавайте заглушки — и вы получите миллисекундные прогоны. Spring нужен для интеграционных/срезовых тестов.

Ещё одно правило — **тонкие конструкторы**: минимум логики внутри, максимум — в методах. Тогда создание класса в тесте не запускает побочных эффектов (см. следующий пункт).

Коллекции стратегий в тестах тоже легко подменять: передайте список с одной-двумя фейковыми реализациями — и проверяйте ветки без поднятия контейнера.

Наконец, договоритесь о простых фабриках тестовых данных (builders/fixtures), чтобы тесты были читабельнее. DI в этом не мешает, а помогает: границы классов чистые.

**Код (Java): unit-тест без Spring)**

```java
package demo.di.tests;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CheckoutServiceTest {
  static class FakePricing implements PricingPort { public long price(String sku) { return 1000; } }
  static class FakeDiscount implements DiscountPort { public long discount(String sku) { return 200; } }
  static class FakeStock implements StockPort { public boolean available(String sku) { return true; } }

  @Test void total_ok() {
    var s = new CheckoutService(new FakePricing(), new FakeDiscount(), new FakeStock());
    assertEquals(800, s.total("sku1"));
  }
}
```

**Код (Kotlin): unit-тест без Spring)**

```kotlin
package demo.di.tests

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Test

class CheckoutServiceTest {
    class FakePricing : PricingPort { override fun price(sku: String) = 1000L }
    class FakeDiscount : DiscountPort { override fun discount(sku: String) = 200L }
    class FakeStock : StockPort { override fun available(sku: String) = true }

    @Test fun total_ok() {
        val s = CheckoutService(FakePricing(), FakeDiscount(), FakeStock())
        assertEquals(800, s.total("sku1"))
    }
}
```

Gradle (Groovy) для JUnit 5:

```groovy
dependencies { testImplementation 'org.springframework.boot:spring-boot-starter-test' }
test { useJUnitPlatform() }
```

Gradle (Kotlin):

```kotlin
dependencies { testImplementation("org.springframework.boot:spring-boot-starter-test") }
tasks.test { useJUnitPlatform() }
```

---

## Договор о «тонких» конструкторах: максимум зависимостей — минимум логики в конструкторе

Толстые конструкторы — источник боли: они начинают сеть вызовов, читают файлы, ходят в сеть, валидируют БД. Такой класс тяжело создать в тесте, а ошибки появляются **на фазе создания**, а не в явном вызове бизнес-метода.

Лучше считать конструктор **проводником** зависимостей: присвоить поля, сделать лёгкую валидацию инвариантов и остановиться. Всё, что может упасть/тормозить, — отложить до публичных методов или `@PostConstruct` (и то осторожно).

Это делает unit-тесты быстрыми: вы создаёте объект мгновенно, а затем пошагово вызываете методы и проверяете результаты. Ошибки воспроизводятся при вызове, а не «где-то при создании».

С точки зрения DI это тоже проще: контейнер строит граф без «неожиданных» эффектов. Порядок создания бинов становится не критичным, снижается вероятность циклов.

Если инициализация действительно тяжёлая (подгрузка справочников, прогрев кэшей), отделите её в явный **инициализатор**/сервис прогрева, который контейнер вызовет отдельно. Это прозрачнее и управляемее.

И наконец, тонкий конструктор облегчает рефакторинг: вы можете переставлять зависимости и менять реализацию без переписывания тестов, которые больше не «ловят» побочки на конструировании.

**Код (Java): толстый vs тонкий конструктор)**

```java
// Плохо
class BadService {
  BadService(HttpClient c) {
    // «работа» в конструкторе
    c.send(java.net.http.HttpRequest.newBuilder().uri(java.net.URI.create("http://x")).build(),
           java.net.http.HttpResponse.BodyHandlers.discarding());
  }
  void doWork() {}
}

// Хорошо
class GoodService {
  private final java.net.http.HttpClient c;
  GoodService(java.net.http.HttpClient c) { this.c = c; } // только присвоение
  void doWork() { /* здесь логика */ }
}
```

**Код (Kotlin): толстый vs тонкий)**

```kotlin
// Плохо
class BadService(c: java.net.http.HttpClient) {
    init {
        c.send(
            java.net.http.HttpRequest.newBuilder().uri(java.net.URI.create("http://x")).build(),
            java.net.http.HttpResponse.BodyHandlers.discarding()
        )
    }
    fun doWork() {}
}

// Хорошо
class GoodService(private val c: java.net.http.HttpClient) {
    fun doWork() { /* логика */ }
}
```

---

## Изоляция побочных эффектов: I/O — за интерфейсами, доменная логика — чистые методы

Чистые методы — детерминированны, не зависят от внешнего мира и не имеют побочных эффектов. Их легко тестировать и переиспользовать. В DI-архитектуре это означает: **весь I/O спрятан за портами**, а доменная логика работает с данными/моделями.

Пример: скидки/цены считаются чисто, а вот получение курса валюты — это порт. В юнит-тесте вы подмените этот порт на фейк и проверите расчёт. В интеграционном тесте — проверите связку с реальным адаптером.

Такой расклад облегчает и производительность: чистую логику можно профилировать и оптимизировать без внешнего шума, а I/O — кешировать/ретраить/контролировать таймаутами на уровне адаптеров.

Изоляция побочки делает код «переносимым»: бизнес-правила легко вынести в библиотеку/отдельный модуль. В больших проектах это часто становится «доменным ядром».

Наконец, это понижает риски безопасности: доступ к секретам/ключам ограничен адаптерами, а домен их не видит. Меньше мест, где секреты могут утечь в логи/метрики.

Договоритесь командой: «домен — чистый, I/O — через порты». DI сделает остальное.

**Код (Java): чистая логика + порт I/O)**

```java
package demo.di.pure;

interface RatePort { double rate(String from, String to); }

class DiscountEngine {
  private final RatePort rates;
  DiscountEngine(RatePort rates) { this.rates = rates; }
  long finalPrice(long baseCents, String cur) {
    double r = rates.rate(cur, "RUB");
    long rub = Math.round(baseCents * r);
    return Math.max(0, rub - 100);
  }
}
```

**Код (Kotlin): чистая логика + порт I/O)**

```kotlin
package demo.di.pure

interface RatePort { fun rate(from: String, to: String): Double }

class DiscountEngine(private val rates: RatePort) {
    fun finalPrice(baseCents: Long, cur: String): Long {
        val r = rates.rate(cur, "RUB")
        val rub = Math.round(baseCents * r)
        return kotlin.math.max(0, rub - 100)
    }
}
```

# 10. Иммутабельность и потокобезопасность

## Конструкторная инъекция + `final` поля → меньше состояний, проще reasoning

Иммутабельные зависимости — это простейший и самый действенный способ уменьшить пространство состояний. Когда вы передаёте зависимости через конструктор и кладёте их в `final` (в Kotlin — `val`) поля, объект перестаёт «меняться под ногами». Мы исключаем замену ссылок, устраняем гонки за инициализацию и делаем класс безопаснее при параллельном доступе. Такой подход особенно важен в контейнерах — Spring по умолчанию создаёт singleton-бины, которыми одновременно пользуются многие потоки.

Ещё одно преимущество — прозрачность. Конструктор становится фактической документацией на класс: заглянул — и сразу видно, какие сервисы нужны. Когда зависимости «расползаются» по сеттерам/полям, эту информацию приходится собирать по всему файлу и помнить порядок вызовов. Иммутабельность дисциплинирует архитектуру: длинный конструктор сигналит о «божественных» сервисах и подсказывает распил.

С точки зрения тестирования конструкторная инъекция избавляет от необходимости поднимать контекст. Вы просто создаёте объект и передаёте фейки. Если внутри конструктора нет побочных эффектов (они и не должны быть), тесты становятся мгновенными и детерминированными. Это важная часть «быстрой» инженерной обратной связи.

Наконец, иммутабельность облегчает внедрение аспектов — прокси/декораторов. Бин оборачивается ровно один раз, а его зависимостям больше не угрожает «переустановка» в рантайме. Это делает наблюдаемость, кеширование и ретраи надёжнее и предсказуемее.

Из практики: объявляйте один публичный конструктор, не используйте полевую инъекцию, не держите «резервные сеттеры». Если зависимость опциональна — лучше дайте `Optional<T>`/`T?` в конструкторе или зарегистрируйте `no-op` реализацию; не открывайте объект для частичной конфигурации.

**Код (Java): иммутабельные зависимости через конструктор**

```java
package demo.immutability;

import org.springframework.stereotype.Service;

interface Clock { long nowMillis(); }

@Service
public class ReportService {
  private final Clock clock; // final — зависимость неизменна

  public ReportService(Clock clock) {
    this.clock = clock;
  }

  public String stamp(String msg) {
    return msg + " @ " + clock.nowMillis();
  }
}
```

**Код (Kotlin): то же с `val`**

```kotlin
package demo.immutability

import org.springframework.stereotype.Service

interface Clock { fun nowMillis(): Long }

@Service
class ReportService(private val clock: Clock) {
    fun stamp(msg: String) = "$msg @ ${clock.nowMillis()}"
}
```

---

## Thread-safety на уровне сервисов: отсутствие изменяемого общего состояния — цель по умолчанию

Потокобезопасность сервисов в Spring обычно достигается не блокировками, а **отсутствием общего изменяемого состояния**. Типичный сервис — stateless: он координирует зависимости и опирается на их потоко-безопасные реализации (пулы соединений, клиенты). Если вам кажется, что нужен `synchronized`, часто это повод пересмотреть дизайн, а не ставить замок.

Где всё же нужна изменяемость — например, счётчики или скользящие окна — используйте специализированные структуры (`AtomicLong`, concurrent-коллекции) и изолируйте их в отдельных компонентах с чёткими границами ответственности. Не распыляйте «кусочки состояния» по многим бинам — так легче ошибиться и труднее тестировать.

Избегайте `ThreadLocal` для бизнес-состояния. Он имеет смысл в инфраструктуре (корреляция traceId, контекст безопасности), но как хранение предметных данных может привести к утечкам и трудноотлавливаемым багам при ресайзинге пулов. Лучше явно передавать значения в параметры методов.

Параметры методов и локальные переменные всегда потокобезопасны — пользуйтесь этим преимуществом. Делайте методы идемпотентными, по возможности — чистыми, чтобы ретраи/повторная доставка сообщений не приводили к дублям и расщеплению состояния.

И ещё: `@Transactional` не делает код потокобезопасным. Он управляет атомарностью и изоляцией на уровне БД, но не синхронизирует доступ к памяти Java-процесса. Если у вас есть кэш/буферы в памяти — продумайте их жизненный цикл и конкурентный доступ отдельно.

**Код (Java): stateless-сервис + атомарный счётчик**

```java
package demo.threads;

import java.util.concurrent.atomic.AtomicLong;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

@Component
class RequestCounter {
  private final AtomicLong total = new AtomicLong();
  public long incAndGet() { return total.incrementAndGet(); }
  public long get() { return total.get(); }
}

@Service
public class StatelessService {
  private final RequestCounter counter;
  public StatelessService(RequestCounter counter) { this.counter = counter; }

  public String handle(String input) {
    long n = counter.incAndGet();
    return "req#" + n + ":" + input.trim();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.threads

import org.springframework.stereotype.Component
import org.springframework.stereotype.Service
import java.util.concurrent.atomic.AtomicLong

@Component
class RequestCounter {
    private val total = AtomicLong()
    fun incAndGet(): Long = total.incrementAndGet()
    fun get(): Long = total.get()
}

@Service
class StatelessService(private val counter: RequestCounter) {
    fun handle(input: String) = "req#${counter.incAndGet()}:${input.trim()}"
}
```

---

## Кэш/пулы/буферы — отдельные компоненты с чёткими границами

Кэш — это состояние. А состояние в многопоточной системе должно жить в **одном месте** и быть управляемым компонентом. Не прячьте `ConcurrentHashMap` внутри «проходного» сервиса. Выделите `Cache` как бин, определите политику (TTL/максимум элементов), экспонируйте метрики. Так проще контролировать память и прогнозировать поведение.

Пулы (соединений БД, HTTP) — то же самое. Их конфигурация должна быть централизованной: один бин `DataSource`, один бин `HttpClient`/`WebClient`, один набор таймаутов. Тогда все потребители будут работать одинаково, а наблюдаемость «прилипнет» ко всем вызовам сразу.

Буферы I/O (например, `ByteBuffer` для обработки файлов) тоже лучше оформлять как явные границы — через арену/пул, а не «каждый создаёт сколько хочет». Это помогает держать под контролем off-heap и не попадать в OOM при всплесках нагрузки.

DI отлично поддерживает такой подход: кэш/пул/буфер становятся **самостоятельными бинами**, а сервисы получают их через конструктор. Появился новый сценарий — создайте ещё один бин с иным профилем или параметрами, а не меняйте десятки мест в коде.

И не забывайте про выключатели. Если кэш не критичен, держите «no-op» реализацию и переключайтесь конфигурацией. Это полезно для тестов и «чистых» прогонов в проде при отладке.

**Код (Java): кэш на Caffeine + Hikari DataSource**

```java
package demo.resources;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class InfraConfig {
  @Bean
  Cache<String, String> localCache() {
    return Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .recordStats()
        .build();
  }

  @Bean
  HikariDataSource dataSource() {
    HikariDataSource ds = new HikariDataSource();
    ds.setJdbcUrl("jdbc:postgresql://db:5432/app");
    ds.setUsername("app");
    ds.setPassword("app");
    ds.setMaximumPoolSize(10);
    return ds;
  }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.resources

import com.github.benmanes.caffeine.cache.Cache
import com.github.benmanes.caffeine.cache.Caffeine
import com.zaxxer.hikari.HikariDataSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.time.Duration

@Configuration
class InfraConfig {
    @Bean
    fun localCache(): Cache<String, String> =
        Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
            .build()

    @Bean
    fun dataSource() = HikariDataSource().apply {
        jdbcUrl = "jdbc:postgresql://db:5432/app"
        username = "app"; password = "app"
        maximumPoolSize = 10
    }
}
```

Gradle зависимости (Groovy/Kotlin):

```groovy
implementation 'com.github.ben-manes.caffeine:caffeine:3.1.8'
runtimeOnly 'org.postgresql:postgresql:42.7.4'
```

```kotlin
implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
runtimeOnly("org.postgresql:postgresql:42.7.4")
```

---

# 11. Эволюция зависимостей и рефакторинг

## «Сигналы к распилу»: слишком много полей в конструкторе, распухшие сервисы, циклы

DI делает запахи заметнее. Если у класса 9–12 зависимостей — это почти наверняка «божественный» сервис. Такой конструктор трудно читать, тесты становятся объёмными, а любое изменение ломает десятки сценариев. Это сигнал вынести подсистемы в отдельные компоненты с узкой ответственностью.

Второй сигнал — сильная связность: сервис знает слишком много о внутренностях зависимостей, лезет в «тонкие места» адаптеров и принимает решения за них. Это нарушает границы слоёв и приводит к каскадным правкам. В ответ — выделяйте фасады/порты и делегируйте.

Третий — циклы зависимостей. Если `A` зависит от `B`, а `B` от `A`, контейнер разбудит вас ранним падением. Это не баг Spring, а подсказка: связь разнонаправленная и дизайн требует пересмотра (вынести общий кусок, ввести события).

Четвёртый — ветвистые `if/else` по типам стратегий. Когда код вручную выбирает «какую реализацию позвать», пора переносить выбор в DI: карта `Map<String, Strategy>` + понятные ключи. Основной сервис остаётся открытым к расширениям.

Пятый — сложная инициализация внутри конструктора. Если объект «живо» работает прямо при создании (чтение файлов, HTTP-вызовы), он плохо тестируется и ломает порядок старта. Вынесите это в `@PostConstruct` (осторожно) или отдельный «инициализатор»/job.

**Код (Java): распухший сервис → разбор на роли**

```java
// До: много зависимостей и логики
class MegaService {
  MegaService(A a, B b, C c, D d, E e, F f, G g) { /* ... */ }
}

// После: фасад + узкие сервисы
interface Pricing { long price(String sku); }
interface Availability { boolean has(String sku); }
interface Discounts { long discount(String sku); }

class CheckoutFacade {
  private final Pricing pricing; private final Availability avail; private final Discounts disc;
  CheckoutFacade(Pricing p, Availability a, Discounts d) { this.pricing = p; this.avail = a; this.disc = d; }
  long total(String sku) { if (!avail.has(sku)) throw new IllegalStateException(); return pricing.price(sku)-disc.discount(sku); }
}
```

**Код (Kotlin): то же**

```kotlin
// До
class MegaService(a: A, b: B, c: C, d: D, e: E, f: F, g: G)

// После
interface Pricing { fun price(sku: String): Long }
interface Availability { fun has(sku: String): Boolean }
interface Discounts { fun discount(sku: String): Long }

class CheckoutFacade(
    private val pricing: Pricing,
    private val avail: Availability,
    private val disc: Discounts
) {
    fun total(sku: String): Long {
        require(avail.has(sku)) { "no stock" }
        return pricing.price(sku) - disc.discount(sku)
    }
}
```

---

## Вынос сложных агрегатов в отдельные компоненты/фабрики

Когда создание объекта становится сложным (много параметров, подготовка зависимостей, валидация), не тащите конструктор-монстра в бизнес-класс. Вынесите сборку в фабрику/билдер — отдельный бин с явной ответственностью «собрать X». Это разгрузит основной сервис и снимет вопросы порядка инициализации.

Фабрика хороша и для управления кэшем/жизненным циклом агрегатов. Например, вы создаёте клиент «по арендаторам» (tenant) и хотите переиспользовать его между запросами. Класс-фабрика держит карту `tenantId → client`, а сервисы просто просят «дай клиент для t». Так состояние локализовано и контролируемо.

Ещё одна польза — тестируемость. Фабрику можно подменить на фейк, который возвращает «пресет» объектов, а основная логика останется неизменной. Это удобно в сценариях, где создание сложное, а поведение — простое.

Если агрегат зависит от множества опций — отдайте их через объект настроек (`Config`), а не 12 позиционных аргументов. Тогда фабрика сможет валидировать целиком и логировать конфигурацию.

Иногда вместо фабрики уместен «компоновщик» (`Assembler`), который собирает граф из простых деталей — стратегия против «гигантского конструктора». И то, и другое — друзья DI.

**Код (Java): фабрика клиентов по арендатору**

```java
package demo.factory;

import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;
import org.springframework.stereotype.Component;

interface TenantClient { String who(); }

@Component
public class TenantClientFactory {
  private final Map<String, TenantClient> cache = new ConcurrentHashMap<>();

  public TenantClient forTenant(String tenant) {
    return cache.computeIfAbsent(tenant, t -> () -> "client:" + t);
  }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.factory

import org.springframework.stereotype.Component
import java.util.concurrent.ConcurrentHashMap

interface TenantClient { fun who(): String }

@Component
class TenantClientFactory {
    private val cache = ConcurrentHashMap<String, TenantClient>()
    fun forTenant(tenant: String): TenantClient =
        cache.computeIfAbsent(tenant) { TenantClient { "client:$it" } }
}
```

---

## Миграция «ручного» кода к DI по шагам: интерфейс → внедрение → выкинуть `new`

Переходить с «ручного `new`» на DI лучше итеративно. Шаг 1 — выделите интерфейс перед конкретной реализацией. Сервис начинает зависеть от абстракции, а не детали. Шаг 2 — зарегистрируйте существующую реализацию как бин (через `@Component` или `@Bean`). Шаг 3 — замените места `new` на конструкторную инъекцию. Шаг 4 — удалите остатки фабрик/локаторов, если они больше не нужны.

Параллельно вынесите настройки в свойства (`application.yml`) и передавайте их через конфигурационные бины. Это уберёт «магические константы» из кода и позволит переключать поведение профилями. Для переходного периода пригодится профиль «legacy», где старая ветка ещё доступна.

Проверяйте тестами. Юнит-тесты сервисов переедут на фейки, интеграционные — будут поднимать контекст и проверять, что wiring собран верно. Это хороший момент прикрутить «срезы» (`@DataJpaTest`, `@WebMvcTest`) и ускорить прогон.

Не бойтесь временно жить с двумя реализациями. DI как раз рассчитан на множественность бинов: `@Primary` назначит «дефолт», `@Qualifier` даст доступ к «старой» ветке там, где нужно сравнить поведение.

И в конце — удалите старьё. Чем дольше «две системы», тем выше операционные риски и путаница в коде. Когда новая реализация покрыла сценарии — выносите `legacy` в архив.

**Код (Java): эволюция `new` → DI**

```java
// Шаг 0: было
class BadReport {
  String load() {
    var http = java.net.http.HttpClient.newHttpClient();
    return "ok"; // зовём клиента напрямую
  }
}

// Шаг 1-3: стало
interface Http { String get(String url); }

@org.springframework.stereotype.Component
class JdkHttp implements Http {
  public String get(String url) { return "ok"; }
}

class GoodReport {
  private final Http http;
  GoodReport(Http http) { this.http = http; }
  String load() { return http.get("http://example"); }
}
```

**Код (Kotlin): то же**

```kotlin
// Было
class BadReport {
    fun load(): String {
        val http = java.net.http.HttpClient.newHttpClient()
        return "ok"
    }
}

// Стало
interface Http { fun get(url: String): String }

@org.springframework.stereotype.Component
class JdkHttp : Http { override fun get(url: String) = "ok" }

class GoodReport(private val http: Http) {
    fun load() = http.get("http://example")
}
```

---

# 12. Конфигурация и сборка графа (preview без деталей)

## Где определяются компоненты (сканирование/конфигурационные методы) и как они находят друг друга

В Spring есть два основных способа положить объекты в контейнер. Первый — **сканирование компонентов**: помечаем классы аннотациями `@Component`/`@Service`/`@Repository`, включаем `@ComponentScan`, и контейнер сам найдёт их на classpath. Этот способ хорош для доменных сервисов и простых адаптеров.

Второй — **Java-конфигурация**: создаём `@Configuration` класс и объявляем бины фабричными методами `@Bean`. Это удобно для сторонних клиентов, когда реализация и её настройки живут рядом. Плюс легко читать: «в одном месте собраны все инфраструктурные зависимости».

На практике проект использует оба. Бизнес-слой обычно сканируется, а инфраструктура создаётся явными методами, где мы видим, откуда берутся таймауты, как устроен пул, какие декораторы навешаны. Всё это прекрасно сочетается с property-конфигурацией.

«Как они находят друг друга?» — по типам и (при необходимости) квалификаторам. Контейнер анализирует конструктор, выбирает кандидатов и строит граф. Если кандидатов несколько — помогают `@Primary`, `@Qualifier`, профили и условные аннотации.

Здесь важно помнить про **чистоту модулей**. Доменные модули не должны зависеть от Spring вообще (кроме как в приложении). Складывайте адаптеры и конфигурацию в инфраструктурные модули; домен — только интерфейсы и чистая логика. Тогда сборка графа — «внешний» слой, как и должно быть.

И наконец, автоконфигурация стартеров — это тоже DI: сторонние библиотеки кладут бины при наличии классов и пропертей. Это мощно, но требует дисциплины: понимайте, какие бины откуда берутся.

**Код (Java): сканирование + Java-конфигурация**

```java
package demo.wiring;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.context.annotation.*;

@SpringBootApplication
public class App {
  public static void main(String[] args) { SpringApplication.run(App.class, args); }
}

@Configuration
class HttpConfig {
  @Bean
  java.net.http.HttpClient http() {
    return java.net.http.HttpClient.newBuilder()
        .connectTimeout(java.time.Duration.ofMillis(800))
        .build();
  }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.wiring

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.net.http.HttpClient
import java.time.Duration

@SpringBootApplication
class App

fun main(args: Array<String>) = runApplication<App>(*args)

@Configuration
class HttpConfig {
    @Bean fun http(): HttpClient =
        HttpClient.newBuilder().connectTimeout(Duration.ofMillis(800)).build()
}
```

---

## Критерий качества: «граф читается по конструктору», зависимости минимальны и очевидны

Хорошая конфигурация — не та, где «всё магически взлетело», а та, где **любой разработчик** быстро понимает, что от чего зависит. Главный критерий — по конструктору класса можно восстановить его окружение: «этот сервис использует такие-то порты и стратегии». Если для понимания надо открывать 5 конфигов и читать 100 строк аннотаций — это тревожный знак.

Минимальность зависимостей — вторая метрика. Чем меньше сервис «просит» на вход, тем легче его тестировать, переносить и переиспользовать. Любая новая зависимость должна быть оправдана и находиться рядом по смыслу, а не просто «на всякий случай».

Третий аспект — явность места выбора. Когда есть несколько реализаций, мы видим, **где** именно решается выбор (`@Qualifier` в конструкторе, конфиг со стратегией). Если выбор «рассредоточен» (то там, то сям), код становится хрупким.

Четвёртое — низкая связность модулей. Доменные пакеты не импортируют инфраструктуру; порты не «знают» про HTTP/SQL. Это тоже читается по конструктору: типы — предметные, адаптеры спрятаны за интерфейсами.

Пятое — простая отладка. Упавший на старте контекст должен понятной ошибкой указать на конфликт бинов, а не тоннами стека. Для этого избегайте «скрытого» Service Locator и держите wiring явным.

И, наконец, воспроизводимость: поднять минимальное окружение (локально/в тесте) должно быть просто. Если конструктор читается, wiring прозрачен и зависимости минимальны — остальные шаги обычно идут гладко.

**Код (Java): «конструктор читабелен»**

```java
package demo.quality;

import org.springframework.stereotype.Service;

interface Payments { boolean pay(long cents); }
interface Risk { boolean allow(String userId, long cents); }
interface Audit { void log(String msg); }

@Service
public class Checkout {
  private final Payments payments; private final Risk risk; private final Audit audit;
  public Checkout(Payments payments, Risk risk, Audit audit) {
    this.payments = payments; this.risk = risk; this.audit = audit;
  }
  public boolean buy(String userId, long cents) {
    if (!risk.allow(userId, cents)) return false;
    boolean ok = payments.pay(cents);
    if (ok) audit.log("ok"); else audit.log("fail");
    return ok;
  }
}
```

**Код (Kotlin): то же**

```kotlin
package demo.quality

import org.springframework.stereotype.Service

interface Payments { fun pay(cents: Long): Boolean }
interface Risk { fun allow(userId: String, cents: Long): Boolean }
interface Audit { fun log(msg: String) }

@Service
class Checkout(
    private val payments: Payments,
    private val risk: Risk,
    private val audit: Audit
) {
    fun buy(userId: String, cents: Long): Boolean {
        if (!risk.allow(userId, cents)) return false
        val ok = payments.pay(cents)
        audit.log(if (ok) "ok" else "fail")
        return ok
    }
}
```

---

# 13. Практические договорённости команды

## Всегда конструкторная инъекция, один публичный конструктор

Командная договорённость снимает десятки мелких споров и делает код единообразным. Базовое правило: **всегда конструкторная инъекция**, **ровно один публичный конструктор**. Так исчезают «полуинициализированные» объекты, поломки от сеттеров, становится проще читать зависимости.

Ещё один пункт — «никакой логики в конструкторе». Конструктор только принимает зависимости и записывает в поля. Всё остальное — в методы/инициализаторы. Это ускоряет тесты и уменьшает вероятность падений на старте.

Где опциональность — используйте `Optional<T>`/`T?` или регистрируйте `no-op` бин. Не оставляйте «открытую дверь» через сеттеры без веской причины. Такой подход улучшает потокобезопасность.

Поддерживайте правило ревью: PR с полевой инъекцией/множеством конструкторов отклоняется. Автоматизируйте проверку через ArchUnit (пример ниже) — так договор превращается в исполняемое правило.

И, наконец, фиксируйте это в README/Contributing: новичкам будет проще включиться, а у опытных разработчиков меньше поводов спорить о базовых вещах.

**Код (Java): ArchUnit-правило на конструкторную инъекцию**

```java
package demo.arch;

import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition;

@AnalyzeClasses(packages = "demo")
public class DiRules {
  @ArchTest
  static final com.tngtech.archunit.lang.ArchRule fields_should_not_be_autowired =
      ArchRuleDefinition.noFields().should().beAnnotatedWith(org.springframework.beans.factory.annotation.Autowired.class);
}
```

**Код (Kotlin): ArchUnit JUnit 5 декларация**

```kotlin
package demo.arch

import com.tngtech.archunit.junit.AnalyzeClasses
import com.tngtech.archunit.junit.ArchTest
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noFields
import org.springframework.beans.factory.annotation.Autowired

@AnalyzeClasses(packages = ["demo"])
class DiRules {
    @ArchTest
    val noAutowiredOnFields = noFields().should().beAnnotatedWith(Autowired::class.java)
}
```

Gradle зависимости:

```groovy
testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'
```

```kotlin
testImplementation("com.tngtech.archunit:archunit-junit5:1.3.0")
```

---

## Интерфейсы у внешних портов и сложной доменной логики; реализации — отдельными бинами

Следующий принцип — «зависим от интерфейсов». На границах модулей/слоёв всегда объявляем **контракты**, а не передаём конкретные классы. Это снижает связанность и делает замену инфраструктуры безболезненной. Реализации же живут отдельными бинами — часто в другом модуле.

Для сложной доменной логики интерфейс помогает отдельно развивать и тестировать алгоритмы (в чистом модуле), а адаптеры держать ближе к среде исполнения (Spring, драйверы). Таким образом, у нас получается «лук»: ядро — порты/домены, внешние слои — реализации и конфигурация.

Чёткое разделение облегчает автотесты: юнит-тесты домена не тянут Spring; интеграционные проверяют связки порт-адаптер. Это сильно ускоряет прогон и повышает надёжность.

Ещё один плюс — множественность реализаций. Сегодня у вас REST-адаптер, завтра gRPC — контракт тот же, DI подложит нужную реализацию по профилю/квалификатору.

Старайтесь, чтобы интерфейсы были **языком домена**, а не транспортом. Никаких `ResponseEntity` в портах; только предметные типы. Это сделает код устойчивым к технологиям.

**Код (Java): порт/реализация в разных пакетах)**

```java
package demo.contracts;

public interface Notifier { void send(String userId, String msg); }
```

```java
package demo.adapters;

import demo.contracts.Notifier;
import org.springframework.stereotype.Component;

@Component("emailNotifier")
public class EmailNotifier implements Notifier {
  @Override public void send(String userId, String msg) { /* SMTP */ }
}
```

**Код (Kotlin): то же)**

```kotlin
package demo.contracts
interface Notifier { fun send(userId: String, msg: String) }
```

```kotlin
package demo.adapters

import demo.contracts.Notifier
import org.springframework.stereotype.Component

@Component("emailNotifier")
class EmailNotifier : Notifier {
    override fun send(userId: String, msg: String) { /* SMTP */ }
}
```

---

## Ясные имена бинов и квалификаторов, общий стиль ошибок при конфликте кандидатов

Когда реализаций много, имена бинов и квалификаторы — ваши «API-ключи» внутри кода. Договоритесь о стиле: `emailNotifier`, `smsNotifier`, `paypal`, `stripe`. Без пробелов/заглавных, с осмысленными, стабильными ключами. Это упростит инъекцию карт `Map<String,T>` и снизит шанс опечаток.

Второй пункт — единый подход к конфликтам кандидатов. Базово ставим `@Primary` для «дефолта», а в местах выбора используем явный `@Qualifier("…")`. Не оставляйте «магический выбор по имени параметра» — это хрупко к переименованиям.

Третий — понятные сообщения об ошибках. Если бин «не нашёлся» или «кандидатов несколько», бросайте исключение с явным перечислением ключей и текущей конфигурации. Это особенно важно в роутерах стратегий.

Выносите ключи в константы/enum — это удешевляет рефакторинг и защищает от расхождений. Хорошая практика — держать «реестр» ключей рядом с интерфейсом.

И помните про документацию. Список доступных стратегий (бин-имён) полезно показать на `/actuator/info` или отдельном эндпойнте для внутренних пользователей — это ускорит интеграции между командами.

**Код (Java): явные имена и проверка конфликтов)**

```java
package demo.names;

import org.springframework.stereotype.Service;
import java.util.Map;

interface Strategy { String doIt(); }

@Service
public class StrategyRouter {
  private final Map<String, Strategy> strategies;
  public StrategyRouter(Map<String, Strategy> strategies) { this.strategies = strategies; }

  public String run(String key) {
    Strategy s = strategies.get(key);
    if (s == null) throw new IllegalArgumentException("Unknown strategy '"+key+"'. Known: "+strategies.keySet());
    return s.doIt();
  }
}
```

**Код (Kotlin): то же)**

```kotlin
package demo.names

import org.springframework.stereotype.Service

interface Strategy { fun doIt(): String }

@Service
class StrategyRouter(private val strategies: Map<String, Strategy>) {
    fun run(key: String): String {
        val s = strategies[key] ?: error("Unknown strategy '$key'. Known: ${strategies.keys}")
        return s.doIt()
    }
}
```

---

## Запрет на `ApplicationContextAware` и Service Locator вне инфраструктурного слоя

В бизнес-коде сервис не должен «лазить» в контейнер за зависимостями. Это ломает инверсию управления, скрывает связность и усложняет тестирование. Договоримся: `ApplicationContextAware` и прямые вызовы `getBean(...)` разрешены только в узком инфраструктурном слое (например, если требуется интеграция с внешним callback, куда DI не дотягивается). Весь остальной код живёт на конструкторной инъекции.

Если сторонний API навязывает вам фабричный коллбек без DI (например, драйвер даёт `Supplier`), оберните точку входа адаптером. Пусть **один** инфраструктурный класс достанет бин из контекста (или примет провайдер), а дальше раздаёт его по контракту. Так вы локализуете «грязь» и не пускаете Service Locator в домен.

Ещё одна причина избегать локатора — рантайм-ошибки. `getBean("name")` — строковый контракт без поддержки компилятора. Любая опечатка станет сюрпризом в проде. Конструкторная инъекция ловит несоответствия на старте, а иногда — на этапе сборки.

Запрет уместно автоматизировать (см. ArchUnit выше) и закрепить в код-стайле. Исключения должны быть редкими и хорошо задокументированными: «почему именно здесь, почему по-другому нельзя».

Если вам часто «приходится» тянуться к контексту — это симптом дефекта архитектуры. Введите фасады, пересоберите граф зависимостей, разделите ответственность.

**Код (Java): инфраструктурный адаптер, локализующий доступ к контексту)**

```java
package demo.locator;

import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;

// Допустимо только здесь, на границе интеграции
@Component
public class CallbackAdapter {
  private final ApplicationContext ctx;
  public CallbackAdapter(ApplicationContext ctx) { this.ctx = ctx; }

  public java.util.function.Function<String,String> handler() {
    var svc = ctx.getBean("processor", Processor.class);
    return svc::process; // дальше — чистый контракт
  }
}

interface Processor { String process(String in); }
```

**Код (Kotlin): то же)**

```kotlin
package demo.locator

import org.springframework.context.ApplicationContext
import org.springframework.stereotype.Component

interface Processor { fun process(input: String): String }

@Component
class CallbackAdapter(private val ctx: ApplicationContext) {
    fun handler(): (String) -> String = ctx.getBean("processor", Processor::class.java)::process
}
```




