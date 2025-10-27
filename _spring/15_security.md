---
layout: page
title: "Spring Security: безопасность приложения"
permalink: /spring/security
---

# 0. Введение

## Что это

Spring Security — это фреймворк для обеспечения безопасности приложений на платформе Spring. Он закрывает два базовых аспекта: аутентификацию (подтверждение личности) и авторизацию (проверка прав доступа), а также включает защитные механизмы уровня HTTP (заголовки, CSRF, CORS). В экосистеме Spring Boot он поставляется как стартер `spring-boot-starter-security` и по умолчанию включает безопасную конфигурацию «всё запрещено, кроме явно разрешённого».

С практической точки зрения Spring Security — это набор фильтров сервлет-уровня и инфраструктурных компонентов (провайдеры аутентификации, менеджеры решений, конвертеры, контекст безопасности), которые оформляют единообразный жизненный цикл проверки запроса. Он перехватывает HTTP-запросы до контроллера, определяет, кто пришёл, что он может, и какие дополнительные политики следует применить (например, принудительная установка безопасных заголовков).

Важно понимать, что Spring Security — это не «магическая стена», которая решит все проблемы безопасности. Он решает класс задач на уровне приложения: доступ, сессии, токены, заголовки, интеграция с провайдерами идентификации. Но он не заменяет WAF, не исправляет уязвимости в бизнес-логике, не защищает от SQL-инъекций, если вы вручную собираете SQL, и не «шифрует всё сам по себе».

Для новичка полезно начать с минимальной конфигурации: включить HTTP Basic, закрыть все эндпоинты кроме парочки публичных, и завести `UserDetailsService` с in-memory пользователями. Затем постепенно добавлять методную безопасность, CSRF для веб-форм, CORS для SPA, и уже позже переходить к JWT и OAuth2/OIDC.

Важная особенность Spring Security — сильная интеграция с остальной экосистемой Spring: аннотации на методах (`@PreAuthorize`), AOP-перехваты, реактивный стек (WebFlux) со своими аналогами конфигурации, и совместимость с Spring MVC, Spring Data, Spring Cloud Gateway.

Ещё одна ключевая черта — расширяемость. Почти каждый кусочек конвейера можно заменить или расширить: собственный `AuthenticationProvider` (например, HMAC или API-ключи), свой `AuthorizationManager` (решения на основе атрибутов доменной модели), дополнительные фильтры (`OncePerRequestFilter`) для аудита и rate limit интеграции.

С точки зрения жизненного цикла разработки Spring Security помогает внедрить «secure by default» на ранней стадии. Как только вы ставите стартер, доступ по умолчанию закрыт, что заставляет вас осознанно открывать только то, что действительно нужно. Это уменьшает поверхность атаки и дисциплинирует подход к API-контрактам.

В продакшн-эксплуатации Spring Security — это центральная точка, где сходятся настройки головных рисков: как мы валидируем JWT, как ротуем ключи, где лежат секреты, как логируются попытки входа, как ограничиваем доступ к техэндпоинтам (Actuator, Swagger), какие заголовки применяем и как настраиваем CORS.

Наконец, Spring Security — отличный «каркас обучения» безопасности. Работая с ним, разработчик быстро усваивает базовые принципы (минимальные права, defense in depth, безопасные дефолты) и учится грамотно проектировать границы доверия в микросервисной среде.

Ниже — минимальный рабочий пример, который показывает самую суть: закрываем всё, открываем `/public/**`, включаем HTTP Basic, создаём двух пользователей in-memory и настраиваем кодировщик паролей. Такой скелет позволяет руками «пощупать» поведение 401/403.

**Gradle (Groovy)**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}

java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}

java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — минимальная конфигурация безопасности**

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain appSecurity(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable()); // для чистого API; для форм включайте!
        return http.build();
    }

    @Bean
    UserDetailsService users(PasswordEncoder encoder) {
        UserDetails user = User.withUsername("user")
                .password(encoder.encode("user123"))
                .roles("USER")
                .build();
        UserDetails admin = User.withUsername("admin")
                .password(encoder.encode("admin123"))
                .roles("ADMIN")
                .build();
        return new InMemoryUserDetailsManager(user, admin);
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**Kotlin — минимальная конфигурация безопасности**

```kotlin
package com.example.demo

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.Customizer
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.userdetails.User
import org.springframework.security.core.userdetails.UserDetailsService
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.provisioning.InMemoryUserDetailsManager
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecurityConfig {

    @Bean
    fun appSecurity(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests { auth ->
                auth.requestMatchers("/public/**", "/actuator/health").permitAll()
                    .anyRequest().authenticated()
            }
            .httpBasic(Customizer.withDefaults())
            .csrf { it.disable() } // включайте для веб-форм!
        return http.build()
    }

    @Bean
    fun users(encoder: PasswordEncoder): UserDetailsService {
        val user = User.withUsername("user").password(encoder.encode("user123")).roles("USER").build()
        val admin = User.withUsername("admin").password(encoder.encode("admin123")).roles("ADMIN").build()
        return InMemoryUserDetailsManager(user, admin)
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
}
```

---

## Зачем это

Причина №1 — риски. Даже если ваш сервис «внутренний», он живёт в сложной инфраструктуре: шлюзы, балансировщики, облачные сети, сторонние интеграции. Любая ошибка маршрутизации, неправильный firewall-rule или XSS в административной SPA могут привести к компрометации. Spring Security сокращает поверхность атаки: всё закрыто, пока вы явно не разрешите доступ.

Причина №2 — соответствие требованиям (compliance). Банковская/финансовая сфера, здравоохранение, госуслуги — везде присутствуют регламенты (например, PCI DSS, 152-ФЗ/Госкритерии), которые требуют строгого контроля доступа, аудит логинов/логинов-неудач, политики паролей, управления сессиями и т.д. Spring Security предоставляет готовую инфраструктуру для выполнения этих требований на уровне приложения.

Причина №3 — управляемость. Когда безопасность разбросана «по коду» (if-ы в контроллерах, кастомные фильтры без единой модели), команда быстро теряет контроль. Spring Security концентрирует правила в одном месте: в `SecurityFilterChain` и методных аннотациях. Это делает поведение предсказуемым и упрощает ревью.

Причина №4 — масштабирование. Сегодня у вас monolith MVC с формами входа, завтра — набор REST-микросервисов и мобильный клиент. Spring Security эволюционирует вместе с архитектурой: stateful-режимы (сессии, formLogin) и stateless (Bearer/JWT), интеграция с OAuth2/OIDC, реактивная безопасность для WebFlux.

Причина №5 — стандарты. Зачем изобретать собственную схему подписи токенов, хранение сессий, проверки ролей, заголовки безопасности? Spring Security следует принятым стандартам (RFC6750 для Bearer Tokens, JOSE/JWT, CORS/CSRF best practices) и экономит месяцы разработки и тестирования.

Причина №6 — тестируемость. С `spring-security-test` вы можете в unit/integration тестах быстро симулировать пользователей, роли, JWT, и детерминированно проверять ответы 401/403, CSRF и заголовки. Это переводит безопасность из «ручной проверки в браузере» в повторяемые автотесты.

Причина №7 — наблюдаемость и аудит. Через события Spring и интеграцию с Micrometer можно собирать метрики и логи входов, неуспешных попыток, всплесков 4xx, что критично для раннего обнаружения атак и инцидентов.

Причина №8 — DevProd. «Secure by default» — не лозунг. Это уменьшает число «сюрпризов в проде», когда вдруг оказывается, что Swagger открыт всему миру или к `/actuator` можно зайти анониму. Наличие безопасных дефолтов вынуждает разработчика подумать, что именно он открывает.

Причина №9 — удобство для фронтенда и клиентов. Чёткие политики CORS/CSRF, единообразные ответы 401 с `WWW-Authenticate: Bearer`/Basic делают интеграцию предсказуемой. Это снижает трение между командами и уменьшает «мистические» баги в браузерах.

Причина №10 — гибкость под доменные модели. Роли (RBAC) легко дополнить доменными правами (ABAC) через `PermissionEvaluator` или кастомные `AuthorizationManager` — это позволяет выражать сложные правила, например, доступ только к ресурсам своего `tenantId`.

Ниже — короткий пример, иллюстрирующий «зачем»: мы включаем методную безопасность и защищаем административный метод ролью `ADMIN`. Такой подход переносит критическую политику ближе к бизнес-логике, чтобы её нельзя было «обойти» сменой URL.

**Gradle (Groovy) — добавим модуль тестирования безопасности (пригодится позже)**

```groovy
dependencies {
    testImplementation 'org.springframework.security:spring-security-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    testImplementation("org.springframework.security:spring-security-test")
}
```

**Java — методная безопасность**

```java
package com.example.demo;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public String issueRefund(String orderId) {
        // доменная логика возврата
        return "Refund issued for " + orderId;
    }
}
```

**Kotlin — методная безопасность**

```kotlin
package com.example.demo

import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.stereotype.Service

@Service
class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    fun issueRefund(orderId: String): String {
        // доменная логика возврата
        return "Refund issued for $orderId"
    }
}
```

> Не забудьте включить методную безопасность в конфигурации: `@EnableMethodSecurity` в одном из `@Configuration`-классов.

---

## Где используется

Spring Security используется везде, где есть Spring: от небольших CRUD-сервисов до крупных банковских платформ. Он одинаково уместен в монолитах (Spring MVC + Thymeleaf), в наборах REST-микросервисов, в реактивных приложениях (WebFlux), в шлюзах (Spring Cloud Gateway) и даже в интеграции с сторонними IdP (Keycloak, Auth0, Azure AD).

В сценариях MVC он обеспечивает stateful-опыт: `formLogin`, remember-me, session management, CSRF для форм, безопасность шаблонов. Это классический корпоративный стек, где пользователи сидят за браузером и работают со страницами.

В REST-микросервисах чаще используется stateless-режим: Bearer/JWT. Здесь Spring Security выступает как «Resource Server», валидируя подпись и метаданные токена, преобразуя клеймы в `GrantedAuthority` и применяя политики на URL/метод-уровне.

В API-шлюзах (например, Spring Cloud Gateway) Spring Security используется для предварительной проверки токенов, нормализации ошибок (`401/403`) и прокидывания валидированного `Authorization` дальше по цепочке (token relay). Это снижает дублирование логики в бэкендах.

В межсервисной коммуникации актуален mTLS (взаимная TLS-аутентификация) — Spring Security интегрируется с контейнером сервлета/реактивным стеком для извлечения клиентских сертификатов и принятия решений доступа по DN/атрибутам.

Для внутренних инструментов (Actuator, Swagger UI) Spring Security выступает защитным периметром: закрыть по умолчанию, открыть по IP-allowlist или роли, отключить/включить в нужных профилях.

В мобильных/SPA-сценариях он управляет CORS, устанавливает правильные security-headers (например, CSP для защиты от XSS), и обеспечивает корректную работу CSRF при cookie-базированных схемах.

В контексте данных (особенно многоарендность) Spring Security может стать «верхним уровнем» над политиками БД (Row-Level Security), чтобы объединить проверки на уровне запроса и на уровне записи. Это повышает защищённость при сложных правилах владения данными.

В средах с повышенными требованиями к наблюдаемости Spring Security «размечает» события безопасности: успешные и неуспешные логины, попытки доступа, отказ в доступе, что помогает строить алерты и отчёты.

Ниже — пример конфигурации, показывающий типичный микс: публичные `/public/**`, защищённые REST-эндпоинты и отдельный доступ к `actuator` только для роли `ADMIN`. Это отражает реальные «места использования» в одном приложении.

**Java — типичное размещение правил**

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity
public class WhereSecurityConfig {

    @Bean
    SecurityFilterChain http(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/public/**").permitAll()
              .requestMatchers("/actuator/**").hasRole("ADMIN")
              .anyRequest().authenticated()
          )
          .httpBasic(b -> {})
          .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

**Kotlin — типичное размещение правил**

```kotlin
package com.example.demo

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
@EnableMethodSecurity
class WhereSecurityConfig {

    @Bean
    fun http(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests { auth ->
                auth.requestMatchers("/public/**").permitAll()
                    .requestMatchers("/actuator/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }
}
```

---

## Какие виды бывают

Во-первых, различают **stateful** и **stateless** режимы. Stateful — когда у нас есть серверные сессии (например, `formLogin`, cookie `JSESSIONID`), CSRF для форм, remember-me. Stateless — когда мы не храним состояние на сервере: клиент присылает Bearer/JWT в каждом запросе, сервер проверяет подпись и клеймы, а контекст не «продолжается» между запросами.

Во-вторых, по способу идентификации: Basic (для техсервисов/быстрых прототипов), Form Login (браузерная аутентификация), OAuth2 Client (логин через внешнего провайдера), OAuth2 Resource Server (проверка JWT), Mutual TLS (клиентские сертификаты), API Keys/HMAC (кастомные варианты, часто для интеграций сервис-к-сервису).

В-третьих, по стеку: сервлетный (Spring MVC) и реактивный (WebFlux). Концепции те же, но API отличаются: `HttpSecurity` vs `ServerHttpSecurity`, разные типы фильтров и конвертеров.

Также выделяют уровни авторизации: URL-уровень (`authorizeHttpRequests`), методный уровень (`@PreAuthorize`/`@Secured`), и кастомные решения (например, `AuthorizationManager` на основе атрибутов доменной модели).

Наконец, различают способы хранения учётных данных: in-memory (для демо), JDBC/LDAP (корпоративные каталоги), внешние IdP (Keycloak/Auth0/Azure AD) для SSO и централизованной аутентификации.

Выбор зависит от клиентского окружения, требований аудита и производительности. Например, для публичного REST-API — JWT (stateless); для кабинета оператора — formLogin (stateful) с 2FA; для межсервисного трафика — mTLS или signed requests.

Ниже — пример, показывающий «два мира» в одном приложении: MVC-часть с formLogin (stateful) и API-часть `/api/**` со stateless JWT. Мы строим **две** цепочки безопасности (`SecurityFilterChain`) с разными `securityMatcher`.

**application.yml — для JWT через issuer (Keycloak/другой IdP)**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/demo
```

**Java — две цепочки: formLogin и JWT**

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class MultiChainConfig {

    @Bean
    @Order(1)
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth -> oauth.jwt())
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(
                org.springframework.security.config.http.SessionCreationPolicy.STATELESS));
        return http.build();
    }

    @Bean
    @Order(2)
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(fl -> fl.loginPage("/login").permitAll())
            .logout(lo -> lo.logoutUrl("/logout"))
            .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"));
        return http.build();
    }
}
```

**Kotlin — две цепочки: formLogin и JWT**

```kotlin
package com.example.demo

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain

@Configuration
class MultiChainConfig {

    @Bean
    @Order(1)
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2ResourceServer { it.jwt() }
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
        return http.build()
    }

    @Bean
    @Order(2)
    fun web(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers("/login", "/public/**").permitAll()
                    .anyRequest().authenticated()
            }
            .formLogin { it.loginPage("/login").permitAll() }
            .logout { it.logoutUrl("/logout") }
            .csrf { it.ignoringRequestMatchers("/api/**") }
        return http.build()
    }
}
```

---

## Какие задачи и проблемы решает

Первая и главная задача — **аутентификация**: кто пришёл. Spring Security поддерживает широкий спектр способов: пароли, внешние IdP, сертификаты, токены. Он предоставляет единый `Authentication`-объект и хранит его в `SecurityContext`.

Вторая — **авторизация**: можно ли этому субъекту делать X над объектом Y. На практике это выражается в правилах URL-уровня, аннотациях на методах и доменных политиках. Spring Security даёт богатую SpEL-экспрессию (`@PreAuthorize`) и хуки для сложных проверок.

Третья — **безопасность HTTP-взаимодействия**: CSRF-защита для форм, CORS для браузерных клиентов, заголовки безопасности (HSTS, X-Content-Type-Options, CSP, X-Frame-Options), которые снижают риски XSS/кликаджеккинга/смешанного контента.

Четвёртая — **управление сессиями** и выходом: ограничения на одновременные сессии, политику фиксации сессии (создавать новую после логина), remember-me, безопасный logout (очистка cookie, аннулирование сессии, редирект).

Пятая — **интеграция с экосистемой**: шлюзы, реактивный стек, наблюдаемость, события. Это позволяет строить слой безопасности, который не «торчит» на каждой вертикали кода, а работает как инфраструктура.

Шестая — **минимизация человеческих ошибок** через «secure by default». По умолчанию всё закрыто, заголовки включены, чувствительные эндпоинты не торчат. Это спасает от банальных, но частых инцидентов.

Седьмая — **тестируемость**. Через `spring-security-test` вы можете программно проверять, что без CSRF POST возвращает 403, что без роли доступ запрещён, что CORS конфиг выдаёт правильные preflight-ответы.

Восьмая — **работа с токенами**: валидация подписи, аудитории, сроков; единообразные ошибки 401 с `WWW-Authenticate`. Это упрощает интеграцию с внешними клиентами и мобильными приложениями.

Девятая — **гранулярные доменные решения**: многоарендность, владение ресурсами, временные окна доступа — через `PermissionEvaluator` и `AuthorizationManager`.

Десятая — **снижение уязвимостей** класса XSS/CSRF/CORS-misconfig. Без правильных заголовков/политик даже хорошая авторизация может быть обойдена через браузерные векторы.

Ниже — пример, где мы включаем штатные заголовки безопасности, аккуратно настраиваем CORS, и демонстрируем CSRF-поведение: для API отключаем, для форм оставляем. Этот шаблон часто ложится в основу «боевой» конфигурации.

**Java — заголовки, CORS и CSRF по зонам**

```java
package com.example.demo;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
public class PoliciesConfig {

    @Bean
    SecurityFilterChain policyChain(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(auth -> auth
              .requestMatchers(HttpMethod.GET, "/public/**").permitAll()
              .anyRequest().authenticated()
          )
          .headers(headers -> headers
              .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
              .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).preload(true))
              .frameOptions(frame -> frame.sameOrigin())
              .xssProtection(xss -> xss.block(true))
          )
          .cors(Customizer.withDefaults())
          .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**")); // формы защищены, API — нет
        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration cfg = new CorsConfiguration();
        cfg.setAllowedOrigins(List.of("https://app.example.com"));
        cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
        cfg.setAllowedHeaders(List.of("Authorization","Content-Type"));
        cfg.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", cfg);
        return source;
    }
}
```

**Kotlin — заголовки, CORS и CSRF по зонам**

```kotlin
package com.example.demo

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.security.config.Customizer
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain
import org.springframework.web.cors.CorsConfiguration
import org.springframework.web.cors.CorsConfigurationSource
import org.springframework.web.cors.UrlBasedCorsConfigurationSource

@Configuration
class PoliciesConfig {

    @Bean
    fun policyChain(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/public/**").permitAll()
                    .anyRequest().authenticated()
            }
            .headers {
                it.contentSecurityPolicy { csp -> csp.policyDirectives("default-src 'self'") }
                    .httpStrictTransportSecurity { hsts -> hsts.includeSubDomains(true).preload(true) }
                    .frameOptions { fo -> fo.sameOrigin() }
                    .xssProtection { x -> x.block(true) }
            }
            .cors(Customizer.withDefaults())
            .csrf { it.ignoringRequestMatchers("/api/**") }
        return http.build()
    }

    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val cfg = CorsConfiguration().apply {
            allowedOrigins = listOf("https://app.example.com")
            allowedMethods = listOf("GET","POST","PUT","DELETE","OPTIONS")
            allowedHeaders = listOf("Authorization","Content-Type")
            allowCredentials = true
        }
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/**", cfg)
        }
    }
}
```

**application.yml — подсказка для прод-жёсткости**

```yaml
server:
  forward-headers-strategy: framework

management:
  endpoints:
    web:
      exposure:
        include: "health,info"
  endpoint:
    health:
      probes:
        enabled: true
spring:
  mvc:
    problemdetails:
      enabled: true
```

> На практике дополняйте это ограничением доступа к Swagger/Actuator в проде, IP-allowlist’ом на уровне прокси и включайте строгие CSP, если у вас есть собственный фронтенд.


# 1. Основы: цели и модель угроз

## Что такое аутентификация и авторизация; CIA-триада и принципы «минимальных прав»

Аутентификация — это подтверждение личности субъекта: «кто пришёл?». В веб-приложениях это может быть пароль, токен, сертификат, внешний провайдер (IdP). Результат аутентификации — объект `Authentication` с идентификатором и наборами прав.

Авторизация — следующий шаг: «что этому субъекту можно?». Здесь решается, имеет ли пользователь право читать, изменять или удалять конкретные ресурсы. В Spring Security авторизация применяется как на уровне URL, так и на уровне методов с помощью аннотаций.

CIA-триада формулирует три цели безопасности: конфиденциальность (данные видят только те, кому можно), целостность (данные нельзя незаметно изменить), доступность (сервис доступен и работает). В приложении эти цели достигаются в совокупности — один механизм не закрывает все три.

Принцип минимальных прав (Least Privilege) требует выдавать только те разрешения, которые действительно нужны. Это снижает ущерб при компрометации и ограничивает «побочные эффекты» ошибок кода. В Spring Security это выражается в точечной настройке правил и узких ролях/authorities.

Ещё один важный принцип — разделение обязанностей (Separation of Duties). Операции, которые могут привести к риску, не должны быть доступны одному субъекту без дополнительных проверок или подтверждений. Это часто реализуется комбинацией ролей и доменных правил.

Подтверждение личности не гарантирует «добрых намерений». Даже аутентифицированный пользователь может попытаться выйти за рамки своей компетенции. Поэтому авторизация должна учитывать контекст: «чей это объект?», «в каком состоянии процесс?», «какой это клиент?».

Слишком широкие роли вроде `ADMIN` ведут к «расползанию привилегий». Гораздо безопаснее декомпозировать права на атомарные authorities (`INVOICE_READ`, `INVOICE_REFUND`) и собирать из них роли. Это упрощает аудит и постепенно переводит систему к ABAC.

Модель угроз всегда конкретна: в банкинге мы защищаем деньги и персональные данные, в медицине — ПДн и клиническую информацию. Одна и та же роль «оператор» в разных доменах означает разные риски, поэтому политики доступа должны рождаться из предметной области.

Аутентификация и авторизация не живут в вакууме: они опираются на транспортную безопасность (TLS), защиту секретов, верное конфигурирование прокси, аудит, логирование и реагирование на инциденты. Отсутствие любого слоя делает всю конструкцию хрупкой.

Практический вывод: сначала определите, какие операции и данные критичны, опишите роли/права, внедрите безопасные дефолты и только потом «открывайте» нужные потоки. Это избавит от «исторически сложившихся» дыр, которые трудно закрывать задним числом.

**Java — пример минимальных прав через authorities (методная авторизация)**

```java
package com.example.security.basics;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class InvoiceService {

    @PreAuthorize("hasAuthority('INVOICE_READ')")
    public String getInvoice(String id) {
        // только чтение счетов
        return "invoice-" + id;
    }

    @PreAuthorize("hasAuthority('INVOICE_REFUND')")
    public String refund(String id) {
        // возврат — отдельное право, не часть INVOICE_READ
        return "refund-" + id;
    }
}
```

**Kotlin — тот же пример**

```kotlin
package com.example.security.basics

import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.stereotype.Service

@Service
class InvoiceService {

    @PreAuthorize("hasAuthority('INVOICE_READ')")
    fun getInvoice(id: String): String = "invoice-$id"

    @PreAuthorize("hasAuthority('INVOICE_REFUND')")
    fun refund(id: String): String = "refund-$id"
}
```

> Аннотации работают при включённой методной безопасности: `@EnableMethodSecurity` в конфигурации. Такой дизайн вынуждает явно разделять права и соответствует принципу минимальных прав.

---

## Типовые угрозы для веб/микросервисов: XSS, CSRF, session fixation, token theft, SSRF

XSS (межсайтовый скриптинг) позволяет атакующему выполнить произвольный JS в браузере жертвы. Источник — вывод «грязных» данных без экранирования или слабая CSP. В серверном коде важно никогда не возвращать пользовательский ввод как HTML без безопасной обработки.

CSRF — подмена намерений: браузер жертвы делает запрос на ваш сайт с действительными cookie, но без ведома пользователя. Защита — CSRF-токен для stateful-форм или отказ от cookie-аутентификации в API (стратегия stateless + Bearer/JWT).

Session fixation — атака, когда злоумышленник «навязывает» жертве свою session id. Правильное лечение — регенерация идентификатора сессии после логина и строгая политика cookie (HttpOnly, Secure, SameSite).

Кража токена (token theft) относится к Bearer-токенам и cookie. Основные меры: TLS везде, минимальный срок жизни токена, ротация, привязка к устройству/контексту, недопущение хранения токенов в локальном хранилище бездумно, защита от XSS.

SSRF — когда ваш сервер делает исходящий HTTP-запрос по адресу, который указал пользователь, и тем самым становится «прокси» к внутренней сети. Меры: allow-list хостов, запрет приватных/loopback сетей, отдельные резолверы/прокси, тайм-ауты.

В микросервисах SSRF особенно опасен — часто внутри сети есть метаданные облака, внутренние админки и брокеры. Любая «переадресация» пользовательского ввода в HTTP-клиент без проверки — потенциальная уязвимость.

Кроме перечисленных, встречаются уязвимости конфигурации: открытый Swagger, чрезмерно широкие CORS, забытые тестовые эндпоинты. Это не «эксплойты», но эксплуатация тривиальна и последствия реальны.

Spring Security помогает «подсобрать» защиту: включает HSTS/X-Content-Type-Options, даёт инструменты CSRF, конфигурацию CORS, методную авторизацию. Но он не валидирует ваши URL, не экранирует HTML и не делает allow-list за вас.

Поэтому стратегия — defense in depth: правильные заголовки, корректный режим аутентификации, строгие политики, плюс явные проверки в местах, где данные пересекают границы доверия (например, перед HTTP-клиентом).

Наконец, угрозы эволюционируют: появление новых браузерных политик (SameSite), новые типы атак на цепочку поставок, эксплуатация уязвимостей библиотек. Регулярные обновления зависимостей — часть вашей модели защиты, а не «когда-нибудь потом».

**Java — утилита против SSRF: allow-list доменов перед RestTemplate**

```java
package com.example.security.threats;

import org.springframework.stereotype.Component;

import java.net.URI;
import java.net.URISyntaxException;
import java.util.Set;

@Component
public class SafeHttp {

    private static final Set<String> ALLOWED_HOSTS = Set.of("api.example.com", "maps.example.org");

    public URI safeUri(String raw) throws URISyntaxException {
        URI uri = new URI(raw);
        String host = uri.getHost();
        if (host == null || !ALLOWED_HOSTS.contains(host)) {
            throw new IllegalArgumentException("Disallowed host: " + raw);
        }
        return uri;
    }
}
```

**Kotlin — тот же allow-list**

```kotlin
package com.example.security.threats

import org.springframework.stereotype.Component
import java.net.URI

@Component
class SafeHttp {

    private val allowedHosts = setOf("api.example.com", "maps.example.org")

    fun safeUri(raw: String): URI {
        val uri = URI(raw)
        val host = uri.host ?: throw IllegalArgumentException("No host")
        require(allowedHosts.contains(host)) { "Disallowed host: $raw" }
        return uri
    }
}
```

> Это не «часть» Spring Security, но именно такие проверки закрывают SSRF. В связке с CORS/CSRF/заголовками вы получаете слой защиты с разумной глубиной.

---

## Где Spring Security в архитектуре Spring Boot; что он «гарантирует», а что — нет

В сервлетном стеке Spring Security реализован как фильтровая цепочка (`SecurityFilterChain`), которая срабатывает до попадания запроса в контроллер. Это первая точка принятия решения: пустить запрос дальше или отклонить.

Spring Security гарантирует, что аноним не пройдёт в защищённые зоны, а аутентифицированный пользователь получит только разрешённое. Он также выставляет заголовки безопасности и управляет сессиями/CSRF в соответствии с конфигурацией.

Но фреймворк не знает вашей бизнес-логики. Он не определит автоматически владельца ресурса, не проверит лимиты, не прочитает из БД «что считается допустимым». Эти проверки — ваша ответственность и часть доменной модели.

Spring Security также не «лечит» небезопасные конструкции: SQL-инъекции, небезопасная сериализация, хрупкий парсинг. Используйте безопасные API (JPA, параметризованные запросы), валидируйте ввод и избегайте небезопасных десериализаторов.

Частая ошибка — положиться только на URL-правила. Если эндпоинт даёт доступ к чужим данным по ID, а методная/доменная проверка отсутствует, уязвимость останется. Нужна проверка «владения» (ownership) на уровне сервиса/репозитория.

Для сложных политик доступ может зависеть от состояния процесса, временных окон, атрибутов объекта. Это реализуется через `@PreAuthorize` с проверкой репозитория, `PermissionEvaluator`, кастомные `AuthorizationManager`.

В реактивном стеке принципы те же, но классы и точки расширения другие (`ServerHttpSecurity`, реактивные конвертеры). Важно не смешивать модели — не блокировать реактивный поток и не «вытягивать» блокирующие зависимости без адаптеров.

Ещё одна зона ответственности вне Spring Security — защита секретов: ключей подписи JWT, паролей к БД, OAuth2-секретов. Их хранят в Vault/KMS/Secret Manager, а приложение получает по защищённым каналам.

Наконец, Spring Security не заменяет WAF/Reverse Proxy/IDS. Эти компоненты дополняют друг друга: прокси выполняет rate limit и IP-фильтрацию, WAF — сигнатуры типовых атак, а приложение — бизнес-правила и контекстные решения.

**Java — пример доменной проверки: доступ только владельцу ресурса**

```java
package com.example.security.arch;

import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    @PostAuthorize("returnObject != null && returnObject.ownerId == authentication.name")
    public OrderDto findById(String id) {
        // эмуляция БД
        return new OrderDto(id, "alice");
    }

    public record OrderDto(String id, String ownerId) {}
}
```

**Kotlin — тот же паттерн**

```kotlin
package com.example.security.arch

import org.springframework.security.access.prepost.PostAuthorize
import org.springframework.stereotype.Service

@Service
class OrderService {

    @PostAuthorize("returnObject != null && returnObject.ownerId == authentication.name")
    fun findById(id: String): OrderDto = OrderDto(id, ownerId = "alice")
}

data class OrderDto(val id: String, val ownerId: String)
```

> `@PostAuthorize` проверяет результат и отсекает чужие данные даже при верном URL-правиле. В реальном проекте добавьте репозитории и проверку на уровне запросов к БД.

---

## Роли участников: пользователь, клиент, ресурс-сервер, IdP/Authorization Server

Пользователь — это человек или сервис, который пытается выполнить действие: зайти в кабинет, получить данные, инициировать операцию. У него есть идентификатор и набор атрибутов (email, tenant, department).

Клиент — приложение, через которое пользователь действует: браузер, мобильное приложение, CLI, интеграционный сервис. Клиент может иметь собственные учётные данные (client credentials) и разные доверенные свойства.

Ресурс-сервер — ваш Spring Boot сервис, владелец защищённых ресурсов (REST-эндпоинты, данные). Он проверяет токены, применяет политики и возвращает ответы. В мире OAuth2 это приложение с ролью Resource Server.

Authorization Server/IdP — отдельный сервис, который аутентифицирует пользователя и выпускает токены (Keycloak/Auth0/Azure AD). Он знает учётные данные, MFA, пароли и атрибуты пользователя, но не решает бизнес-политики доступа к вашим данным.

При stateful-вебе IdP может быть встроен в приложение (formLogin, JDBC), но чаще выделяется в отдельный сервис по соображениям безопасности и централизованного управления политиками (SSO, парольная политика, 2FA).

Коммуникация: клиент направляет пользователя в IdP, тот после логина возвращает код/токен клиенту, а клиент передаёт ресурс-серверу Bearer-токен. Ресурс-сервер проверяет подпись/валидность токена и решает, что можно отдать.

Важно различать «кто» и «что»: пользователь — субъект действий, а клиент — средство. В некоторых сценариях клиент действует сам (Client Credentials) без пользователя: это сервер-к-серверу интеграции.

Атрибуты, приходящие от IdP (claims), должны быть отображены в роли/authorities вашего сервиса. Здесь нужен явный маппинг: не все `roles` из IdP должны автоматически становиться `ROLE_*` в приложении.

Ошибочно полагаться на имена ролей из IdP как на «истину в последней инстанции». Лучше держать слой соответствия (claim → authority) под контролем владельца сервиса и логировать несоответствия.

При многоарендности (multi-tenant) в claims имеет смысл передавать `tenantId`, чтобы ресурс-сервер мог применять политики видимости данных. Это удобно маппить в `Authentication` и использовать в репозиториях.

**Java — маппинг claims → authorities для Resource Server**

```java
package com.example.security.roles;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;

@Configuration
public class JwtMappingConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http, Converter<Jwt, ? extends AbstractAuthenticationToken> converter) throws Exception {
        http
          .authorizeHttpRequests(a -> a.anyRequest().authenticated())
          .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(converter)));
        return http.build();
    }

    @Bean
    Converter<Jwt, ? extends AbstractAuthenticationToken> jwtAuthConverter() {
        return jwt -> {
            JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter();
            scopes.setAuthorityPrefix("SCOPE_"); // для scope
            Collection<GrantedAuthority> authorities = scopes.convert(jwt);
            // Дополнительно: маппим custom claim "roles" → ROLE_*
            var roles = (Collection<String>) jwt.getClaimAsStringList("roles");
            if (roles != null) {
                roles.forEach(r -> authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
            }
            return new JwtAuthenticationToken(jwt, authorities, jwt.getClaimAsString("sub"));
        };
    }
}
```

**Kotlin — тот же конвертер**

```kotlin
package com.example.security.roles

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.convert.converter.Converter
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.server.resource.authentication.AbstractAuthenticationToken
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter
import org.springframework.security.web.SecurityFilterChain

@Configuration
class JwtMappingConfig {

    @Bean
    fun api(http: HttpSecurity, converter: Converter<Jwt, out AbstractAuthenticationToken>): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2ResourceServer { it.jwt { j -> j.jwtAuthenticationConverter(converter) } }
        return http.build()
    }

    @Bean
    fun jwtAuthConverter(): Converter<Jwt, out AbstractAuthenticationToken> = Converter { jwt ->
        val scopes = JwtGrantedAuthoritiesConverter().apply { setAuthorityPrefix("SCOPE_") }
        val authorities = scopes.convert(jwt)!!.toMutableSet<GrantedAuthority>()
        jwt.getClaimAsStringList("roles")?.forEach { r -> authorities.add(SimpleGrantedAuthority("ROLE_$r")) }
        JwtAuthenticationToken(jwt, authorities, jwt.getClaimAsString("sub"))
    }
}
```

> Так вы контролируете, какие claims превращаются в права приложения, и не «прокидываете» лишнее.

---

## Базовые режимы: stateful (сессии, формы) vs stateless (Bearer/JWT, API)

Stateful-режим — сервер хранит состояние сессии, клиент — только идентификатор (cookie). Это удобно для классических веб-форм, когда есть UI, CSRF-защита и remember-me. Минус — масштабирование: sticky-sessions или внешние хранилища сессий.

Stateless-режим — никаких серверных сессий, каждый запрос несёт доказательство аутентификации (Bearer/JWT). Это удобно для API и мобильных клиентов, легче масштабируется и кэшируется на прокси. Минус — сложнее отзыв (revoke) токена до его истечения.

В stateful-вебе аутентификация чаще проходит через форму логина и `UserDetailsService`/JDBC/LDAP. Здесь важны политики паролей, MFA, ограничение одновременных сессий, регенерация id после логина.

В stateless-API аутентификация делегируется IdP (Authorization Server), который выдаёт подписанные JWT. Ресурс-сервер валидирует подпись/аудитории/срок и сопоставляет claims с правами. CSRF здесь обычно выключен.

Иногда пути смешиваются: в одном приложении страницы (formLogin) и API (JWT). Тогда применяют несколько `SecurityFilterChain` с разными `securityMatcher`, чтобы не перепутать политики.

Важно понимать, что stateful и stateless — это не про «уровень безопасности», а про модель взаимодействия и клиента. Выбор зависит от того, кто с вами разговаривает — браузер или интеграция.

Производительность: stateless убирает обращения к хранилищу сессий, но требует криптографии и обработки JWT. На практике это дешёвее диска/сети и лучше масштабируется горизонтально.

Отзыв токенов в stateless делается через короткий срок жизни и refresh-пары, чёрные списки (ревокеры), или через introspection (но это уже state-ish). Важно планировать ротацию и управление ключами.

Наконец, окружение: для stateful важна атрибутика cookie (Secure/HttpOnly/SameSite), для stateless — строгие заголовки, CORS, и грамотная обработка 401/403 с `WWW-Authenticate` для согласованной интеграции.

**Java — две конфигурации: формы (stateful) и JWT (stateless)**

```java
package com.example.security.modes;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class ModesConfig {

    @Bean
    @Order(1)
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(c -> c.disable());
        return http.build();
    }

    @Bean
    @Order(2)
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a.requestMatchers("/login", "/public/**").permitAll().anyRequest().authenticated())
            .formLogin(f -> f.loginPage("/login").permitAll())
            .logout(l -> l.logoutUrl("/logout"))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED));
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.modes

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain

@Configuration
class ModesConfig {

    @Bean
    @Order(1)
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2ResourceServer { it.jwt() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean
    @Order(2)
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/login", "/public/**").permitAll()
                    .anyRequest().authenticated()
            }
            .formLogin { it.loginPage("/login").permitAll() }
            .logout { it.logoutUrl("/logout") }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED) }
        return http.build()
    }
}
```

> Такой расклад убирает путаницу: браузерные сценарии живут в одной «зоне», API — в другой, с корректными политиками для каждого мира.

---

## Политики «secure by default»: закрыто всё, открываем точечно

Безопасные дефолты означают, что доступ запрещён, пока вы его явно не разрешите. В Spring Security это естественное состояние после подключения стартера: любой запрос потребует аутентификацию.

Открывать нужно ровно то, что должно быть публичным: статические ресурсы, `/actuator/health`, публичные страницы/эндпоинты. Всё остальное — только после аутентификации и с корректными ролями.

Правило «более конкретное выше» важно для порядка в `authorizeHttpRequests`. Сперва специфичные пути/методы, потом общие, и в самом конце — заглушка на всё остальное (`anyRequest().authenticated()`).

Ещё один «безопасный по умолчанию» аспект — заголовки. Даже если у вас чистый API, включённые HSTS/X-Content-Type-Options/X-Frame-Options не мешают, а экономят вам инциденты из-за тривиальных векторов.

CORS должен быть строгим: конкретные origins, методы, заголовки. Широкое `*` уместно только для публичного API без cookie/кредов. В противном случае это дырка, которую легко эксплуатировать.

CSRF в браузере включайте всегда для formLogin и cookie-аутентификации. Отключайте только для API-зон и только при stateless-модели. Смешение политик приводит к «случайно работающему» и хрупкому поведению.

Статические ресурсы исключайте из безопасности правильно: через `PathRequest.toStaticResources()`/`requestMatchers` в `SecurityFilterChain`, а не через `web.ignoring()` (устаревший путь), чтобы не отключать полезные заголовки.

Для техэндпоинтов (Swagger/Actuator) — белые списки IP на прокси + авторизация в приложении. В проде Swagger UI часто отключают, оставляя только JSON-спеку за авторизацией.

Метрики и логи доступа — тоже часть secure-по-умолчанию: вы не просто «запретили», вы ещё и видите попытки доступа, распределение 4xx по зонам и можете на них реагировать.

И наконец, документация: зафиксируйте, что открыто и почему. Это упрощает ревью изменений и предотвращает «случайные» открытия ради «быстрее показать демо».

**Java — шаблон «закрыто всё, открываем точечно»**

```java
package com.example.security.defaults;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.autoconfigure.security.servlet.PathRequest;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecureDefaultsConfig {

    @Bean
    SecurityFilterChain http(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(a -> a
              .requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
              .requestMatchers("/actuator/health", "/public/**").permitAll()
              .anyRequest().authenticated()
          )
          .headers(h -> h.httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true)))
          .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"));
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.defaults

import org.springframework.boot.autoconfigure.security.servlet.PathRequest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecureDefaultsConfig {

    @Bean
    fun http(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
                    .requestMatchers("/actuator/health", "/public/**").permitAll()
                    .anyRequest().authenticated()
            }
            .headers { h -> h.httpStrictTransportSecurity { it.includeSubDomains(true) } }
            .csrf { it.ignoringRequestMatchers("/api/**") }
        return http.build()
    }
}
```

> Такой каркас удобно разворачивать в каждом сервисе: дефолты безопасны, а «открытия» — редкие и осознанные.

# 2. Архитектура Spring Security

## SecurityFilterChain: порядок фильтров, DelegatingFilterProxy, OncePerRequestFilter

Spring Security в сервлет-приложении реализует безопасность как цепочку фильтров (Servlet Filter Chain), которая стоит перед контроллерами и перехватывает каждый HTTP-запрос. Эта цепочка создаётся как один или несколько бинов `SecurityFilterChain`, и на уровне контейнера сервлетов проксируется через `DelegatingFilterProxy`, который перенаправляет входящий трафик в управляемые Spring’ом фильтры. Благодаря этому любая настройка безопасности остаётся декларативной и управляется контекстом, а не конфигами контейнера.

Порядок фильтров строго определён: например, сначала работают фильтры для контекста безопасности и исключений, затем — для аутентификации (Basic, Form, Bearer), позже — авторизация и заголовки. Нарушение порядка может «сломать» логику обработки, поэтому любая кастомизация должна учитывать, куда именно вы вставляете собственный фильтр. В большинстве случаев вы используете удобные методы `addFilterBefore/After` и целитесь в хорошо известные “якоря” — `UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter` и т.д.

`DelegatingFilterProxy` — мост между миром сервлетов и миром Spring DI. Он сам по себе «тонкий»: его задача — достать из контекста Spring фактический фильтр (или цепочку) и отдать выполнение. Это снимает необходимость регистрировать каждый security-фильтр на уровне контейнера и упрощает тестирование.

Кастомные фильтры обычно реализуют `OncePerRequestFilter` — абстракцию Spring Security, гарантирующую один вызов на запрос (в отличие от “сырого” `Filter`, который может быть задействован несколько раз на внутренние диспетчеризации). Это особенно важно, если фильтр записывает в `SecurityContext` или считает метрики; дубли привели бы к некорректному состоянию.

Частая архитектурная задача — разделить «зоны» приложения: например, `/api/**` (JWT, stateless) и остальное (formLogin, stateful). Для этого регистрируют несколько `SecurityFilterChain` c разными `securityMatcher`, и каждый из них формирует свою подцепочку фильтров, что позволяет жить двум «мирам» без конфликтов.

Ещё один практический момент — исключение статических ресурсов. Их разрешают через `PathRequest`/`requestMatchers` в конфигурации, а не через глобальное «игнорирование» на уровне контейнера, чтобы не отключать полезные security-заголовки и не ронять наблюдаемость.

При отладке очень полезно логировать список активных фильтров и их порядок. В продакшене — наоборот, избегать подробных логов безопасности, чтобы не раскрыть детали конвейера или чувствительные параметры. Балансируйте информативность и конфиденциальность.

Если нужен аудит/трассировка, добавляйте лёгкие фильтры-наблюдатели (`OncePerRequestFilter`) раньше авторизации, чтобы пометить запрос trace-id и потом сопоставлять события доступа с бизнес-логами. Но такие фильтры не должны ломать тело запроса и нарушать идемпотентность.

Важно помнить, что фильтры — «горячая» часть пути запроса, поэтому любые блокировки, сетевые вызовы и тяжёлые операции в кастомных фильтрах увеличивают латентность. Старайтесь оставлять бизнес-решения на уровне сервисов и методной авторизации.

Наконец, фильтровую архитектуру удобно тестировать через `MockMvc`/`WebTestClient`, проверяя 401/403, CORS, CSRF и заголовки. Это закрепляет контракт безопасности на уровне CI и предотвращает регрессии при обновлениях Spring Boot.

**Gradle (Groovy)** — зависимости те же, что и для базовой безопасности:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — кастомный фильтр и вставка до BearerTokenAuthenticationFilter**

```java
package com.example.arch.filters;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationFilter;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

class TraceFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String traceId = req.getHeader("X-Trace-Id");
        if (traceId != null) {
            req.setAttribute("traceId", traceId);
        }
        chain.doFilter(req, res); // важно не прерывать цепочку
    }
}

@Configuration
class FilterConfig {
    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt())
            .addFilterBefore(new TraceFilter(), BearerTokenAuthenticationFilter.class);
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch.filters

import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationFilter
import org.springframework.security.web.SecurityFilterChain
import org.springframework.web.filter.OncePerRequestFilter
import java.io.IOException

class TraceFilter : OncePerRequestFilter() {
    @Throws(ServletException::class, IOException::class)
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        req.getHeader("X-Trace-Id")?.let { req.setAttribute("traceId", it) }
        chain.doFilter(req, res)
    }
}

@Configuration
class FilterConfig {
    @Bean
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2ResourceServer { it.jwt() }
            .addFilterBefore(TraceFilter(), BearerTokenAuthenticationFilter::class.java)
        return http.build()
    }
}
```

---

## Authentication → SecurityContext → Authorization: жизненный цикл запроса

Жизненный цикл в Spring Security начинается с входа запроса в цепочку фильтров. На ранних стадиях работает инфраструктура контекста (создание/извлечение `SecurityContext`) и исключений, затем — фильтры аутентификации, которые пытаются распознать представленные клиентом учётные данные: заголовок `Authorization` (Basic/Bearer), форма логина, mTLS и т.п. Если аутентификация успешна, получается объект `Authentication` с principal и authorities.

После успешной аутентификации `Authentication` помещается в `SecurityContext`, который хранится в `SecurityContextHolder`. По умолчанию используется стратегия `ThreadLocal`, а значит все последующие слои обработки в том же потоке могут получить доступ к текущему пользователю. На завершении запроса контекст очищается, чтобы избежать утечек между запросами.

При ошибке аутентификации управление перехватывает `ExceptionTranslationFilter`, который преобразует исключение в корректный HTTP-ответ: `401 Unauthorized` с правильным `WWW-Authenticate` для анонимов, или `403 Forbidden` для аутентифицированных, но недостаточно привилегированных пользователей. Это обеспечивает стандартизированные ответы и предсказуемость для клиентов.

Авторизация запускается после того, как установлен контекст и выполнено сопоставление запроса с правилами. На уровне URL работает механизм `AuthorizationManager`, а на уровне метода — AOP-перехватчик методной безопасности. Оба читают `Authentication` из контекста и принимают решение «можно/нельзя». Решение может кэшироваться в пределах запроса, но полагаться на это не стоит — лучше формулировать правила коротко и детерминированно.

В случае реактивного стека (WebFlux) `SecurityContext` переносится не через `ThreadLocal`, а через `Reactor Context`, и доступ к нему идёт через реактивные API. Это влияет на способ получения текущего пользователя и требует избегать блокирующих вызовов в конвейере.

Иногда нужно «подменить» контекст — например, запустить системную операцию с техническим пользователем. Делайте это локально и ограниченно через `SecurityContextHolder` в try-with-resources и обязательно возвращайте исходное состояние, чтобы не загрязнить поток.

Контроллеры и сервисы не должны напрямую заниматься аутентификацией. Их задача — бизнес-логика. Аутентификация — ответственность фильтров и `AuthenticationManager`. В сервисах вы только читаете текущего пользователя, чтобы принять доменное решение.

Понимание жизненного цикла важно для отладки: если контекст «пустой», значит что-то случилось до авторизации — токен не распознан, фильтр не сработал, цепочка не применена из-за `securityMatcher`. Логируйте по уровням, начиная с фильтров аутентификации.

Очистка контекста на завершении запроса — критичная деталь. Утечки `Authentication` между запросами приводят к катастрофическим последствиям, особенно в пулах потоков. Используйте стандартные механизмы Spring Security и не храните `Authentication` в статических полях.

Наконец, учитывайте многопоточность: если вы порождаете дочерние потоки, `ThreadLocal` по умолчанию не наследуется. Для редких сценариев можно включить `MODE_INHERITABLETHREADLOCAL`, но лучше переосмыслить дизайн и не тянуть `Authentication` в фоновые задачи.

**Gradle (Groovy/Kotlin)** — без доп. зависимостей:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-security' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-security") }
```

**Java — доступ к текущему пользователю и стратегия контекста**

```java
package com.example.arch.lifecycle;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/me")
public class MeController {

    @GetMapping
    public String whoAmI() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth == null ? "anonymous" : auth.getName();
    }

    static {
        // Опционально: сделать контекст наследуемым дочерними потоками
        // SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch.lifecycle

import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/me")
class MeController {
    @GetMapping
    fun whoAmI(): String =
        SecurityContextHolder.getContext().authentication?.name ?: "anonymous"
}
```

---

## UserDetails, GrantedAuthority, AuthenticationProvider, AuthenticationManager

`UserDetails` — контракт представления пользователя для внутренних целей безопасности: имя, пароль (если есть), флаги «активен/заблокирован», и коллекция `GrantedAuthority` (прав). Это «то, что знает система» о субъекте, отдельно от доменной модели пользователя в БД.

`GrantedAuthority` — атомарное право (например, `ROLE_ADMIN` или `INVOICE_READ`). Именно они участвуют в проверках `hasRole/hasAuthority`. Роли — это соглашение, обычно с префиксом `ROLE_`. На практике полезно хранить набор «тонких» authorities и собирать из них роли.

`AuthenticationProvider` — стратегий может быть несколько: один отвечает за пароли, другой — за OTP, третий — за API-ключи. Провайдер получает `Authentication`-запрос (например, логин/пароль) и возвращает аутентифицированный `Authentication` (с `UserDetails` и authorities), либо бросает исключение.

`AuthenticationManager` — оркестратор, который опрашивает список провайдеров и принимает первое успешное решение. В Boot 3.x типовой менеджер строится автоматически из зарегистрированных `AuthenticationProvider`.

В JDBC/LDAP-сценариях чаще определяют `UserDetailsService` (загрузка по username) и `PasswordEncoder`, а провайдером становится стандартный `DaoAuthenticationProvider`. В кастомных сценариях вы пишете свой `AuthenticationProvider` (например, заголовок `X-API-KEY`).

Достоинство архитектуры — расширяемость без модификации ядра. Вы можете добавить 2FA, HMAC, WebAuthn, не меняя остальную конфигурацию, просто зарегистрировав ещё один провайдер и соответствующий фильтр, который создаёт нужный вид `Authentication`.

Важно разделять доменную модель и модель безопасности. Не обязательно ваш `UserEntity` реализует `UserDetails`; вы можете маппить доменную сущность в `UserDetails` адаптером. Это сохраняет гибкость и минимизирует протекание деталей безопасности в бизнес-код.

Пароли должны храниться только в виде хэшей с солью (BCrypt/Argon2). В тестах допустимы `{noop}`/in-memory, но в проде — только стойкие энкодеры и политика ротации/сложности. Помните, что `PasswordEncoder` — часть безопасности, которую нельзя «упростить ради удобства».

Лёгкая проверка своих прав из кода (`SecurityContextHolder.getContext().getAuthentication().getAuthorities()`) допустима, но лучше использовать аннотации и конфиг — так правила остаются декларативными и покрываются тестами.

Наконец, не смешивайте аутентификацию и авторизацию в одном провайдере. Провайдер отвечает за «кто это», а «что можно» — уже на уровне правил URL/методов/доменных проверок.

**Gradle (Groovy/Kotlin)** — стандартные зависимости безопасности:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-security' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-security") }
```

**Java — кастомный AuthenticationProvider для API-ключа**

```java
package com.example.arch.auth;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.core.*;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.web.authentication.preauth.AbstractPreAuthenticatedProcessingFilter;
import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

import java.util.List;

class ApiKeyAuthentication extends AbstractAuthenticationToken {
    private final String apiKey;
    ApiKeyAuthentication(String apiKey) {
        super(null);
        this.apiKey = apiKey;
        setAuthenticated(false);
    }
    @Override public Object getCredentials() { return apiKey; }
    @Override public Object getPrincipal() { return "api-client"; }
}

class ApiKeyProvider implements AuthenticationProvider {
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String key = (String) authentication.getCredentials();
        if (!"top-secret-key".equals(key)) throw new BadCredentialsException("Invalid API key");
        List<GrantedAuthority> auth = List.of(new SimpleGrantedAuthority("ROLE_INTEGRATION"));
        ApiKeyAuthentication result = new ApiKeyAuthentication(key);
        result.setAuthenticated(true);
        result.setDetails(auth);
        return new UsernamePasswordAuthenticationToken("api-client", key, auth);
    }
    @Override
    public boolean supports(Class<?> authentication) {
        return ApiKeyAuthentication.class.isAssignableFrom(authentication);
    }
}

class ApiKeyFilter extends AbstractPreAuthenticatedProcessingFilter {
    @Override protected Object getPreAuthenticatedPrincipal(HttpServletRequest request) { return null; }
    @Override protected Object getPreAuthenticatedCredentials(HttpServletRequest request) {
        return request.getHeader("X-API-KEY");
    }
}

@Configuration
class ApiKeySecurity {

    @Bean
    AuthenticationProvider apiKeyProvider() { return new ApiKeyProvider(); }

    @Bean
    SecurityFilterChain api(HttpSecurity http, AuthenticationProvider provider) throws Exception {
        ApiKeyFilter filter = new ApiKeyFilter();
        filter.setAuthenticationManager(new ProviderManager(provider));
        http.securityMatcher("/integrations/**")
            .addFilter(filter)
            .authorizeHttpRequests(a -> a.anyRequest().hasRole("INTEGRATION"))
            .csrf(c -> c.disable());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch.auth

import jakarta.servlet.http.HttpServletRequest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.authentication.*
import org.springframework.security.core.Authentication
import org.springframework.security.core.AuthenticationException
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.preauth.AbstractPreAuthenticatedProcessingFilter
import org.springframework.security.config.annotation.web.builders.HttpSecurity

class ApiKeyAuthentication(private val key: String) : AbstractAuthenticationToken(null) {
    init { isAuthenticated = false }
    override fun getCredentials(): Any = key
    override fun getPrincipal(): Any = "api-client"
}

class ApiKeyProvider : AuthenticationProvider {
    override fun authenticate(authentication: Authentication): Authentication {
        val key = authentication.credentials as String
        if (key != "top-secret-key") throw BadCredentialsException("Invalid API key")
        val auth: List<GrantedAuthority> = listOf(SimpleGrantedAuthority("ROLE_INTEGRATION"))
        return UsernamePasswordAuthenticationToken("api-client", key, auth)
    }
    override fun supports(authentication: Class<*>): Boolean =
        ApiKeyAuthentication::class.java.isAssignableFrom(authentication)
}

class ApiKeyFilter : AbstractPreAuthenticatedProcessingFilter() {
    override fun getPreAuthenticatedPrincipal(request: HttpServletRequest): Any? = null
    override fun getPreAuthenticatedCredentials(request: HttpServletRequest): Any? = request.getHeader("X-API-KEY")
}

@Configuration
class ApiKeySecurity {

    @Bean fun apiKeyProvider(): AuthenticationProvider = ApiKeyProvider()

    @Bean
    fun api(http: HttpSecurity, provider: AuthenticationProvider): SecurityFilterChain {
        val filter = ApiKeyFilter().apply { authenticationManager = ProviderManager(provider) }
        http.securityMatcher("/integrations/**")
            .addFilter(filter)
            .authorizeHttpRequests { it.anyRequest().hasRole("INTEGRATION") }
            .csrf { it.disable() }
        return http.build()
    }
}
```

---

## AuthorizationManager/AccessDecision: как принимается решение «можно/нельзя»

В современных версиях Spring Security рекомендовано использовать `AuthorizationManager` вместо устаревшего `AccessDecisionManager`. Идея проста: когда известно *что* запрашивается (контекст запроса/метода) и *кто* (Authentication), менеджер возвращает `AuthorizationDecision` — разрешить или запретить.

`AuthorizationManager` для HTTP-уровня получает `RequestAuthorizationContext` с доступом к `HttpServletRequest`. Это позволяет реализовывать тонкие политики: проверять HTTP-метод, путь, заголовки, атрибуты, multi-tenant контекст и т.п. Решение должно быть быстрым и идемпотентным.

Порядок правил по-прежнему важен: сначала конкретные правила с `.access(...)` для узких путей, затем — более общие, и в конце — заглушка. Это уменьшает нагрузку, так как менеджер вызывается только на совпавших правилах.

Если политика сложна и зависит от доменной модели, часть проверки переносится на методный уровень (SpEL/`PermissionEvaluator`) или в БД (Row-Level Security). HTTP-уровень оставляют для coarse-grained правил: «в эту зону — только роль X или tenant Y».

`AccessDecisionManager` с голосующими `Voter` всё ещё поддерживается, но API `AuthorizationManager` проще и лучше ложится на новый DSL. Если у вас наследие на `Voter`, миграция возможна, но не обязательна.

В случае отказа `ExceptionTranslationFilter` вернёт `403 Forbidden` для аутентифицированных субъектов. Сообщения об ошибках должны быть «сухими» — не раскрывайте детали политики, чтобы не помогать злоумышленнику строить карту доступа.

Логи решений полезны для расследований. Но в проде логируйте только факт отказа с минимальным контекстом (путь, метод, пользователь, tenant), чтобы не утекали чувствительные данные. Для трассировки можно включать DEBUG на стейджинге.

Тестируемость — сильная сторона подхода. Вы можете подставить разные `Authentication` (mock JWT/пользователи) и прогнать набор URL/методов, проверяя, что менеджер даёт ожидаемые решения. Это превращает политику в «исполняемую документацию».

Для gateway-паттерна аналогичная идея помогает централизовать coarse-grained контроль: часть логики доступа живёт в gateway, остальное — в target сервисах. Но не перегружайте шлюз доменными проверками — только широкие границы и нормализация.

**Gradle** — те же зависимости:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-security' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-security") }
```

**Java — кастомный AuthorizationManager: tenant из JWT должен совпасть с заголовком**

```java
package com.example.arch.authz;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authorization.*;
import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.web.access.intercept.RequestAuthorizationContext;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class TenantAuthzConfig {

    @Bean
    AuthorizationManager<RequestAuthorizationContext> tenantMatches() {
        return (Supplier<Authentication> auth, RequestAuthorizationContext ctx) -> {
            Authentication a = auth.get();
            if (!(a instanceof JwtAuthenticationToken token)) return new AuthorizationDecision(false);
            String tenantClaim = token.getToken().getClaimAsString("tenant");
            String tenantHeader = ctx.getRequest().getHeader("X-Tenant");
            return new AuthorizationDecision(tenantClaim != null && tenantClaim.equals(tenantHeader));
        };
    }

    @Bean
    SecurityFilterChain api(HttpSecurity http, AuthorizationManager<RequestAuthorizationContext> tenantMatches) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/tenant/**").access(tenantMatches)
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(o -> o.jwt());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch.authz

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.authorization.AuthorizationDecision
import org.springframework.security.authorization.AuthorizationManager
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.Authentication
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.access.intercept.RequestAuthorizationContext
import java.util.function.Supplier

@Configuration
class TenantAuthzConfig {

    @Bean
    fun tenantMatches(): AuthorizationManager<RequestAuthorizationContext> =
        AuthorizationManager { auth: Supplier<Authentication>, ctx: RequestAuthorizationContext ->
            val a = auth.get()
            val decision = if (a is JwtAuthenticationToken) {
                val claim = a.token.getClaimAsString("tenant")
                val header = ctx.request.getHeader("X-Tenant")
                claim != null && claim == header
            } else false
            AuthorizationDecision(decision)
        }

    @Bean
    fun api(http: HttpSecurity, tenantMatches: AuthorizationManager<RequestAuthorizationContext>): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests {
                it.requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .requestMatchers("/api/tenant/**").access(tenantMatches)
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { it.jwt() }
        return http.build()
    }
}
```

---

## Аннотации уровня метода: @EnableMethodSecurity, @Pre/PostAuthorize, @Secured

Методная безопасность — второй слой авторизации, работающий через AOP-перехватчики. Аннотации позволяют выразить правило как часть контракта метода: кто может вызвать и с какими условиями. Это особенно полезно для доменных проверок (владелец объекта, состояние процесса), которые не выразить правилом URL.

`@EnableMethodSecurity` включает поддержку аннотаций и формирует прокси вокруг бинов-кандидатов. Без этой аннотации методные правила не активируются. В современных версиях включение достаточно добавить в любую `@Configuration`.

`@PreAuthorize` проверяет выражение до вызова метода. Внутри доступен `authentication`, аргументы метода (`#id`, `#dto`), переменные `#principal`, `#root`. Выражение — SpEL, что даёт огромную мощь, но требует дисциплины и тестов.

`@PostAuthorize` проверяет уже после выполнения: например, «вернуть объект можно только владельцу». Это удобно, когда проверка зависит от результата бизнес-логики. Но не злоупотребляйте — лучше заранее отсекать лишнее.

`@Secured` — простой вариант для фиксированных ролей. Он не поддерживает SpEL и контекст. Подходит для грубых правил: «этот метод только для ADMIN». Для гибких проверок используйте `@PreAuthorize`.

Для произвольной логики — `PermissionEvaluator`. Он внедряется в SpEL как `hasPermission(#obj, 'ACTION')` и может обращаться к базе или сервисам для принятия решения. Это мост между доменной моделью и авторизацией.

В реактивном мире существуют аналоги (`@PreAuthorize` также доступна), но внутри нужна реактивная реализация получения контекста/репозиториев. Смешивать блокирующие вызовы в SpEL — плохая идея; выносите проверки в сервис и делайте их реактивными.

Методная безопасность дополняет, а не заменяет URL-уровень. URL-уровень закрывает периметр, методный — выражает доменную политику. Такой «двойной замок» сложнее обойти и лучше документирует намерения.

Покрывайте методные правила тестами с `@WithMockUser`/mock JWT. Ошибки в SpEL всплывают только на выполнении, и без тестов их легко пропустить. При рефакторинге имён полей/методов SpEL не компилируется заранее, поэтому регрессии возможны.

**Gradle** — базовые зависимости:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-security' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-security") }
```

**Java — @EnableMethodSecurity, @PreAuthorize и PermissionEvaluator**

```java
package com.example.arch.methods;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler;
import org.springframework.stereotype.Service;

import java.io.Serializable;
import java.util.UUID;

@Configuration
@EnableMethodSecurity
class MethodSecurityConfig {
    @Bean
    MethodSecurityExpressionHandler mseh(PermissionEvaluator pe) {
        DefaultMethodSecurityExpressionHandler h = new DefaultMethodSecurityExpressionHandler();
        h.setPermissionEvaluator(pe);
        return h;
    }

    @Bean
    PermissionEvaluator permissionEvaluator() {
        return new PermissionEvaluator() {
            @Override
            public boolean hasPermission(org.springframework.security.core.Authentication auth, Object target, Object perm) {
                // Пример: разрешаем только владельцу ресурса
                if (target instanceof Document doc) {
                    return doc.ownerId().equals(auth.getName()) && "READ".equals(perm);
                }
                return false;
            }
            @Override
            public boolean hasPermission(org.springframework.security.core.Authentication auth, Serializable targetId, String targetType, Object perm) {
                return false;
            }
        };
    }
}

record Document(UUID id, String ownerId, String content) {}

@Service
class DocumentService {

    @PreAuthorize("hasPermission(#doc, 'READ')")
    public String read(Document doc) {
        return doc.content();
    }

    @PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.name")
    public void delete(UUID docId, String ownerId) {
        // удаление
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch.methods

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.access.PermissionEvaluator
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.stereotype.Service
import java.io.Serializable
import java.util.UUID

@Configuration
@EnableMethodSecurity
class MethodSecurityConfig {

    @Bean
    fun mseh(pe: PermissionEvaluator): MethodSecurityExpressionHandler =
        DefaultMethodSecurityExpressionHandler().apply { permissionEvaluator = pe }

    @Bean
    fun permissionEvaluator(): PermissionEvaluator = object : PermissionEvaluator {
        override fun hasPermission(auth: org.springframework.security.core.Authentication, target: Any?, perm: Any?): Boolean {
            return (target as? Document)?.let { it.ownerId == auth.name && perm == "READ" } ?: false
        }
        override fun hasPermission(auth: org.springframework.security.core.Authentication, targetId: Serializable?, targetType: String?, perm: Any?): Boolean = false
    }
}

data class Document(val id: UUID, val ownerId: String, val content: String)

@Service
class DocumentService {

    @PreAuthorize("hasPermission(#doc, 'READ')")
    fun read(doc: Document): String = doc.content

    @PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.name")
    fun delete(docId: UUID, ownerId: String) { /* ... */ }
}
```

---

## Конфиг как код: HttpSecurity DSL, SecurityMatcher, правило «наиболее конкретное — выше»

Современный подход к настройке — «конфигурация как код». Вместо XML и глобальных адаптеров мы определяем один или несколько бинов `SecurityFilterChain` и описываем политику через `HttpSecurity` DSL. Это делает конфиг типобезопасным, видимым в IDE и легко тестируемым.

`securityMatcher` позволяет привязать цепочку к подмножеству запросов. Например, `/api/**` — одна политика (JWT, stateless), а всё остальное — другая (formLogin). Это лучше, чем пытаться в одном дереве правил учесть и API, и веб-формы, и статические ресурсы.

Правило «наиболее конкретное — выше» означает, что в пределах одного `authorizeHttpRequests` сперва перечисляют точные совпадения (путь+метод), потом — более широкие, и в конце — общий случай `anyRequest()`. Так вы избегаете неожиданностей, когда общее правило «перекрывает» частный случай.

Порядок самих `SecurityFilterChain` контролируют через `@Order`. Сначала — цепочки для узких «зон», затем — «дефолтная» цепочка. Это ещё один уровень «более конкретное выше», но уже между цепочками.

DSL поддерживает читаемость: легко видно, какие зоны публичны, где применяются заголовки безопасности, где отключён CSRF. Такой код проще ревьюить и поддерживать, чем «раскиданные» фильтры и хелперы.

Для больших проектов полезно выносить «общие» части в конфигураторы/методы: например, функцию, добавляющую CSP/HSTS/headers, или функцию, настраивающую CORS. Это сохраняет единообразие между сервисами.

Тестируемость DSL — важный плюс. С `MockMvc`/`WebTestClient` можно проверить, что GET `/public/ping` даёт 200 без аутентификации, а POST `/api/items` без токена — 401, и это всё фиксируется в автотестах. Конфигурация перестаёт быть «магией».

Помните, что «конкретность» относится и к HTTP-методу: два правила для `/orders/**` с `GET` и `POST` должны стоять выше общих правил для `/orders/**` без метода. Так вы управляетесь с тонкими различиями доступа по операциям.

Наконец, держите конфиг компактным: если политику трудно прочитать, разбейте на несколько цепочек «по зонам». Это лучше, чем одна труднообслуживаемая простыня.

**Gradle** — стандартные зависимости:

```groovy
dependencies { implementation 'org.springframework.boot:spring-boot-starter-security' }
```

```kotlin
dependencies { implementation("org.springframework.boot:spring-boot-starter-security") }
```

**Java — две цепочки и порядок правил (конкретное выше)**

```java
package com.example.arch.dsl;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class DslSecurityConfig {

    @Bean
    @Order(1)
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .requestMatchers(HttpMethod.POST, "/api/orders/**").hasAuthority("ORDER_CREATE")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(o -> o.jwt())
            .csrf(c -> c.disable());
        return http.build();
    }

    @Bean
    @Order(2)
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/", "/login", "/assets/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(f -> f.loginPage("/login").permitAll());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.arch.dsl

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class DslSecurityConfig {

    @Bean
    @Order(1)
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/orders/**").hasAuthority("ORDER_CREATE")
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { it.jwt() }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean
    @Order(2)
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/", "/login", "/assets/**").permitAll()
                    .anyRequest().authenticated()
            }
            .formLogin { it.loginPage("/login").permitAll() }
        return http.build()
    }
}
```

# 3. Базовая настройка для Spring Boot 3.x/6.x

## Определяем @Bean SecurityFilterChain; отключаем `WebSecurityConfigurerAdapter`

Переход на Spring Security 6 (Spring Boot 3.x) означает отказ от старого `WebSecurityConfigurerAdapter`. Теперь конфигурация безопасности описывается как «конфиг как код» через один или несколько бинов `SecurityFilterChain`. Такой подход проще читается, лучше поддаётся тестированию и даёт явный контроль порядка и зон применения.

Главная идея — собрать политику доступа в декларативный DSL `HttpSecurity`: сперва выбираем скоуп запросов (`securityMatcher`), затем формулируем правила авторизации (`authorizeHttpRequests`), включаем конкретный механизм аутентификации (Basic/Form/JWT), настраиваем CSRF/CORS/Headers. В конце вызываем `http.build()` и отдаём `SecurityFilterChain`.

Отказ от адаптера также «развязывает руки» для использования нескольких независимых цепочек фильтров. Мы можем создавать «зоны безопасности» (например, `/api/**` со stateless JWT и «всё остальное» с formLogin) без взаимного влияния и с явным приоритетом через `@Order`.

Фреймворк подхватывает все бины `SecurityFilterChain` из контекста и сам регистрирует необходимые фильтры в контейнере сервлетов через `DelegatingFilterProxy`. Благодаря этому мы не пишем «низкоуровневых» регистраций фильтров и остаёмся в мире Spring DI.

Важная привычка — держать конфигурацию компактной. Если одно правило не читается с первого раза, вынесите его в отдельную «зону» или в «хелпер»-метод (например, общее включение заголовков безопасности и CORS). Так снижается риск скрытых конфликтов и упрощается ревью.

Всё лишнее нужно удалять. Если в проекте остались старые классы, расширяющие `WebSecurityConfigurerAdapter`, смело их убирайте: даже если проект компилируется, логика не выполнится на современном Boot и создаст ложное ощущение защищённости.

Тестирование конфигурации стало проще: достаточно поднять контекст и проверить ответы на ключевые эндпоинты (200/401/403), не прибегая к «магическим» внутренностям адаптера. Правила — явные, поэтому их легко проверять через `MockMvc`/`WebTestClient`.

Старайтесь не смешивать в одной цепочке взаимоисключающие политики (formLogin + JWT на одних и тех же путях). От этого появляются странные побочные эффекты вроде «иногда 401, иногда 302 на /login». Разделяйте зоны — «конкретное выше, общее ниже».

С точки зрения миграции, почти всё, что раньше было в `configure(HttpSecurity)`, переезжает в лямбды DSL. Отличие в том, что теперь конфиг — это возвращаемый бином объект, а не переопределённый метод жизненного цикла.

И наконец, относитесь к `SecurityFilterChain` как к публичному контракту сервиса. Любое изменение правил должно сопровождаться тестом и заметкой в changelog: это такой же API, как и REST-эндпоинты.

**Gradle (Groovy) — базовые зависимости безопасности**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

**Gradle (Kotlin)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Java — минимальный каркас `SecurityFilterChain`**

```java
package com.example.security.basic;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain app(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .httpBasic(b -> {})   // временно, для простоты
            .csrf(c -> c.disable());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.basic

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecurityConfig {

    @Bean
    fun app(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }
}
```

---

## Минимум безопасности: httpBasic/formLogin, запрет всего (`anyRequest().authenticated()`)

Минимальная безопасная конфигурация для API — выключить анонимный доступ и включить самый простой способ представления учётных данных. На этапе скелета часто используют HTTP Basic: он прозрачно работает с curl/Postman и не требует HTML-страниц.

HTTP Basic годится только для внутренних/техзадач и прототипов. Логин/пароль передаются в заголовке в base64 и должны защищаться TLS. В проде для публичных клиентов лучше переходить на OAuth2/OIDC или на форму входа в MVC.

Для браузерных приложений с серверным рендерингом удобно `formLogin()`. Он добавит эндпоинты `/login` и `/logout` и возьмёт на себя стандартные ответы/редиректы. В паре с ним включайте CSRF — это критично для cookie-сессий.

Запрет всего остального — необходимый дефолт. Правило `anyRequest().authenticated()` в конце блока авторизации чётко фиксирует, что только явно разрешённые зоны будут публичными. Это спасает от случайного «прозрачного» эндпоинта.

Важно правильно обрабатывать 401/403. Анонимному клиенту защищённого API должен приходить 401 с корректным `WWW-Authenticate` (Basic/Bearer), чтобы клиент понимал схему. Аутентифицированному, но без прав — 403.

Даже при минимуме включайте базовые security headers. Они почти «бесплатны», но сильно закрывают от банальных векторов (например, `X-Content-Type-Options: nosniff`).

CSRF включайте только там, где есть cookie-сессии и браузеры. Для чистого REST лучше отключить CSRF (или исключить `/api/**`), чтобы не подхватывать ложные 403 на небраузерных клиентах.

Не забудьте закодировать пароли через `PasswordEncoder`, иначе Spring Security отвергнет сырой пароль. В тестовых стендах допускается `{noop}`, но лучше сразу привыкайте к BCrypt/Argon2.

Для повторяемости добавьте health-эндпоинт в permitAll, чтобы оркестраторы/балансировщики могли проверять сервис без аутентификации. Остальные actuator-метрики — под роль/защиту, особенно в проде.

Минимализм — не про бедность, а про базовую гигиену. Даже эта «скелетная» конфигурация уже существенно лучше «ничего», но она лишь старт к более зрелым схемам (JWT, OIDC, многоарендность и т.п.).

**Java — минимальная API-конфигурация (Basic + deny-by-default)**

```java
package com.example.security.min;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class MinimalSecurity {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(b -> {})
            .csrf(c -> c.disable());
        return http.build();
    }

    @Bean
    InMemoryUserDetailsManager users(PasswordEncoder encoder) {
        UserDetails user = User.withUsername("user").password(encoder.encode("user123")).roles("USER").build();
        return new InMemoryUserDetailsManager(user);
    }

    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.min

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.userdetails.User
import org.springframework.security.core.userdetails.UserDetails
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.provisioning.InMemoryUserDetailsManager
import org.springframework.security.web.SecurityFilterChain

@Configuration
class MinimalSecurity {

    @Bean
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                    .anyRequest().authenticated()
            }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean
    fun users(encoder: PasswordEncoder): InMemoryUserDetailsManager {
        val user: UserDetails = User.withUsername("user").password(encoder.encode("user123")).roles("USER").build()
        return InMemoryUserDetailsManager(user)
    }

    @Bean fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
}
```

---

## Публичные эндпоинты (health/docs) и точечные матчинги по путям/HTTP-методам

Выделение публичных точек нужно для доступности и эксплуатации. Чаще всего это `GET /actuator/health` и, при необходимости, статические ассеты или публичные страницы. Всё остальное — закрыто. Это упрощает жизнь мониторингу и балансировщикам.

Spring Security 6 продвигает явные `requestMatchers`, в том числе с указанием HTTP-метода. Это важно: одно и то же «место» ресурса может требовать разных прав для `GET` и для `POST`. Использование методов делает политику чёткой и предсказуемой.

Для Swagger/OpenAPI придерживайтесь принципа «публичная спека — только на стейджинге». В проде чаще всего либо полностью закрывают UI, либо ограничивают доступ по ролям и/или по IP на уровне прокси. Сам JSON может быть доступен за аутентификацией.

При настройке public-зон избегайте широких паттернов типа `/**` «на время демо». Эти «временные» разрешения часто остаются навсегда и становятся источником инцидентов. Открывайте конкретные пути и только нужными методами.

Если вы включаете CORS, делайте это адресно. Публичный эндпоинт не означает «любой origin»: для браузера это вопрос доверия. Оставляйте конкретные списки доменов, а не `*`, особенно если планы — включать cookie/креды.

Отдельно посмотрите на `HEAD` и `OPTIONS`. Для preflight-запросов CORS можно явно разрешить `OPTIONS` на нужных путях — так вы избавитесь от «ложных» 403 при междоменных запросах.

Старайтесь не нарушать правило «самое конкретное — выше». В `authorizeHttpRequests` сперва перечисляйте public-маршруты, затем — частные, в конце — заглушку. Тогда случайное совпадение общим правилом не «затрет» исключение.

Для статических ресурсов есть специальные матчинги (см. ниже): они должны быть public, но «правильно» исключёнными. Так вы сохраните security headers на ответах и избежите лиших фильтров.

Не забывайте о Problem Details (`spring.mvc.problemdetails.enabled=true` в Boot 3): единообразные ошибки улучшают DX. Возвращайте стандартизированные 401/403, а не кастомные HTML-страницы из фильтров.

И наконец, фиксируйте публичные точки в документации сервиса. Это экономит время SRE/DevOps и снижает риск «размывания периметра» при будущих правках.

**Java — точечные правила по путям и методам**

```java
package com.example.security.publics;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class PublicEndpointsSecurity {

    @Bean
    SecurityFilterChain http(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/v3/api-docs/**").hasRole("ADMIN")
                .requestMatchers("/swagger-ui/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(b -> {})
            .csrf(c -> c.ignoringRequestMatchers("/v3/api-docs/**")); // если UI дергает POST
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.publics

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class PublicEndpointsSecurity {

    @Bean
    fun http(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                    .requestMatchers(HttpMethod.GET, "/v3/api-docs/**").hasRole("ADMIN")
                    .requestMatchers("/swagger-ui/**").hasRole("ADMIN")
                    .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                    .anyRequest().authenticated()
            }
            .httpBasic { }
            .csrf { it.ignoringRequestMatchers("/v3/api-docs/**") }
        return http.build()
    }
}
```

---

## PasswordEncoder (BCrypt/Argon2) и хранение паролей; in-memory vs JDBC/LDAP

Хранить пароли в открытом виде запрещено. В Spring Security пароль всегда должен быть закодирован `PasswordEncoder`-ом. Дефакто-стандарт — `BCryptPasswordEncoder`; также доступен `Argon2PasswordEncoder`, требовательный к памяти и устойчивый к параллелизму атак.

`BCrypt` даёт баланс между безопасностью и производительностью. Параметр «strength» влияет на стоимость проверки: повышая его, вы замедляете перебор, но увеличиваете нагрузку на легитимные логины. Начните со значения по умолчанию и измеряйте.

В тестах можно использовать `{noop}` для демо, но не переносите это в прод. Лучше создать «боевой» `PasswordEncoder` и в тестах шифровать пароли так же, чтобы на проде конфигурация не отличалась.

In-memory пользователи удобны для прототипов, smoke-тестов и примеров. Но в реальном приложении данные должны приходить из JDBC/LDAP/внешнего IdP. Для JDBC Spring Security предлагает `JdbcUserDetailsManager` и ожидает определённые схемы таблиц.

В JDBC-модели пароли хранятся в таблице `users`, а права — в `authorities` (или `group_*`). Вы можете адаптировать имена/схему, но проще начать с стандартной и лишь затем тонко настроить.

LDAP целесообразен там, где уже существует корпоративный каталог и политики паролей. Аутентификацию берёт на себя LDAP, а в приложении вы сопоставляете группы с ролями. Это снимает с вас заботы о хранении паролей и их ротации.

Если у вас SSO/IdP (Keycloak/Auth0/Azure AD), приложение не «знает» паролей вовсе — оно просто валидирует токены. Но для внутренних админок или сервисов всё ещё может понадобиться JDBC/LDAP.

Не храните соль отдельно и «ручками» — `BCrypt`/`Argon2` это уже делают правильно. Не используйте устаревшие схемы типа `MD5`, `SHA-1`, «двойной MD5» и т.п.: это создаёт ложное ощущение защиты.

Проводите миграции паролей: при успешном логине с устаревшим алгоритмом — пересчитывайте на новый. Spring Security позволяет строить «делегирующий» энкодер, который понимает префиксы `{bcrypt}`, `{argon2}` и т.п.

**Gradle (Groovy) — добавим JDBC/H2 для примера**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'com.h2database:h2'
}
```

**Gradle (Kotlin)**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-jdbc")
    runtimeOnly("com.h2database:h2")
}
```

**SQL — минимальная схема пользователей и прав**

```sql
create table if not exists users (
  username varchar(50) primary key,
  password varchar(100) not null,
  enabled boolean not null
);

create table if not exists authorities (
  username varchar(50) not null,
  authority varchar(50) not null,
  constraint fk_auth_users foreign key(username) references users(username)
);
create unique index if not exists ix_auth_username on authorities(username, authority);
```

**application.yml — H2 для демонстрации**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
    username: sa
    password:
  sql:
    init:
      mode: always
  h2:
    console:
      enabled: true
```

**Java — `JdbcUserDetailsManager` + BCrypt**

```java
package com.example.security.jdbc;

import javax.sql.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.provisioning.JdbcUserDetailsManager;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class JdbcUsersConfig {

    @Bean
    JdbcUserDetailsManager jdbcUsers(DataSource ds, PasswordEncoder enc) {
        JdbcUserDetailsManager mgr = new JdbcUserDetailsManager(ds);
        if (!mgr.userExists("alice")) {
            UserDetails u = User.withUsername("alice").password(enc.encode("p@ssw0rd")).roles("USER").build();
            mgr.createUser(u);
        }
        return mgr;
    }

    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.jdbc

import javax.sql.DataSource
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.core.userdetails.User
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.provisioning.JdbcUserDetailsManager

@Configuration
class JdbcUsersConfig {

    @Bean
    fun jdbcUsers(ds: DataSource, enc: PasswordEncoder): JdbcUserDetailsManager {
        val mgr = JdbcUserDetailsManager(ds)
        if (!mgr.userExists("alice")) {
            val u = User.withUsername("alice").password(enc.encode("p@ssw0rd")).roles("USER").build()
            mgr.createUser(u)
        }
        return mgr
    }

    @Bean fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
}
```

---

## Статические ресурсы: как исключать из фильтрации корректно

Статика должна раздаваться без аутентификации, но при этом ответы должны оставаться «закалёнными» заголовками безопасности. Поэтому её не «выкидывают» из Spring Security полностью, а разрешают через `requestMatchers(PathRequest.toStaticResources().atCommonLocations())`.

Исторический способ через `WebSecurityCustomizer#ignoring()` считался допустимым, но теперь его избегают: так вы теряете навешивание security headers и часть наблюдаемости на этих ответах. Правильный путь — разрешить запросы в рамках `SecurityFilterChain`.

`PathRequest.toStaticResources().atCommonLocations()` покрывает типичные каталоги (`/static`, `/public`, `/resources`, `/META-INF/resources`) и корректно работает с маппингом `ResourceHttpRequestHandler`. Это удобно и уменьшает шанс пропустить путь.

Если у вас кастомные пути для ассетов (например, `/assets/**`), добавьте их явным правилом `permitAll`. Но не используйте «широкие» шаблоны: лучше перечислить известные директории, чем разрешать всё «рядом».

Помните про кэширование ассетов. Правильно настроенный `Cache-Control` разгружает сервер и уменьшает влияние Spring Security на латентность, ведь ответы могут обслуживаться прямо CDN/прокси.

Если вы отдаёте HTML-страницы, полезно включать CSP. Даже для статики это снижает риск XSS через инъекцию скриптов в кэш/CDN. Заголовок можно настроить в блоке `headers()` глобально.

HSTS включайте для продакшна на домен целиком. Тогда и статические ресурсы всегда по HTTPS, и браузеры не будут «даже пробовать» HTTP-версии.

Следите за тем, чтобы пути статики не совпадали с API-путями. Конфликт может привести к неожиданным разрешениям. Явные префиксы (`/api/**`, `/assets/**`) — хорошая привычка.

Тестируйте отдать-статический-файл без аутентификации и убедитесь, что заголовки безопасности на месте. Это легко покрывается парой integration-тестов и защищает от «случайно сломали статику».

**Java — разрешаем статику корректно**

```java
package com.example.security.staticres;

import org.springframework.boot.autoconfigure.security.servlet.PathRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class StaticSecurity {

    @Bean
    SecurityFilterChain http(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
                .requestMatchers("/assets/**").permitAll()
                .anyRequest().authenticated()
            )
            .headers(h -> h.contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'")))
            .httpBasic(b -> {})
            .csrf(c -> c.disable());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.staticres

import org.springframework.boot.autoconfigure.security.servlet.PathRequest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class StaticSecurity {

    @Bean
    fun http(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
                    .requestMatchers("/assets/**").permitAll()
                    .anyRequest().authenticated()
            }
            .headers { h -> h.contentSecurityPolicy { csp -> csp.policyDirectives("default-src 'self'") } }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }
}
```

---

## Конфиги по профилям: dev упрощаем, prod ужесточаем

Spring-профили позволяют держать разные политики для dev/stage/prod без ветвления кода. Это критично для безопасности: на dev можно упростить вход и расширить логи, а на prod — ужесточить заголовки, закрыть лишние эндпоинты и усилить CORS.

На dev иногда включают HTTP Basic и открывают Swagger UI, чтобы ускорить разработку. Но эти послабления должны жить только под профилем `dev` и никогда не попадать в prod. Чёткое разделение — через `@Profile("dev")` на конфиг-классах.

В prod-профиле стоит закрыть Swagger UI, оставить только JSON-спеку и то — за авторизацией, а ещё лучше — не публиковать вовсе. Для Actuator оставьте `health`/`info`, остальное — под роль/сетевые ACL на прокси.

CORS: на dev допустимо `http://localhost:*`, но на prod нужен явный список доменов фронтенда. Не ставьте `*` вместе с `allowCredentials=true` — браузер это отклонит, а некоторые клиенты будут вести себя непредсказуемо.

Заголовки: в prod включайте HSTS с `max-age` и `includeSubDomains`, CSP без `unsafe-inline`, `X-Frame-Options: DENY` (или `SAMEORIGIN`, если нужен iframe). На dev CSP можно смягчить для удобства, но зафиксировать это в профиле.

Логи и ошибки: подробные ошибки безопасности (stacktrace) включайте только на dev. В prod используйте лаконичные Problem Details без раскрытия внутренностей. Любые детали политики — только в DEBUG и на стейджинге.

Секреты и ключи — только из Secret Manager/Vault. В dev можно использовать «файлы окружения», но это не переносимая практика. В проде хранение секретов в `application-prod.yml` — нарушение, которого следует избегать.

Проверяйте профили автотестами. Набор smoke-тестов может стартовать контекст с `spring.profiles.active=prod` и валидировать, что Swagger закрыт, CORS строгий, а `health` доступен анонимно. Это ловит «забытые» dev-конфиги ещё до релиза.

Для multi-chain конфигураций помните, что `@Order` важен внутри каждого профиля. Не полагайтесь на «случайный» порядок бинов — фиксируйте его явно аннотацией или структурой конфигов.

**application.yml — базовое разделение профилей**

```yaml
spring:
  profiles:
    group:
      dev: "dev"
      prod: "prod"

---
spring:
  config:
    activate:
      on-profile: dev

app:
  cors:
    allowed-origins: "http://localhost:3000"

management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics"

---
spring:
  config:
    activate:
      on-profile: prod

app:
  cors:
    allowed-origins: "https://app.example.com"

management:
  endpoints:
    web:
      exposure:
        include: "health,info"
```

**Java — два конфига по профилям**

```java
package com.example.security.profiles;

import org.springframework.context.annotation.*;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@Profile("dev")
class DevSecurity {

    @Bean
    SecurityFilterChain dev(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(b -> {})
            .csrf(c -> c.disable());
        return http.build();
    }
}

@Configuration
@Profile("prod")
class ProdSecurity {

    @Bean
    SecurityFilterChain prod(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .headers(h -> h
                .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).preload(true))
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
            )
            .csrf(c -> c.ignoringRequestMatchers("/api/**"));
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.profiles

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
@Profile("dev")
class DevSecurity {

    @Bean
    fun dev(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                    .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                    .anyRequest().authenticated()
            }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }
}

@Configuration
@Profile("prod")
class ProdSecurity {

    @Bean
    fun prod(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/actuator/health").permitAll()
                    .anyRequest().authenticated()
            }
            .headers {
                it.httpStrictTransportSecurity { h -> h.includeSubDomains(true).preload(true) }
                    .contentSecurityPolicy { csp -> csp.policyDirectives("default-src 'self'") }
            }
            .csrf { it.ignoringRequestMatchers("/api/**") }
        return http.build()
    }
}
```

# 4. Авторизация: роли, политики и доменные права

> Для примеров в этой подтеме предполагаем Spring Boot 3.3.x/Spring Security 6.x, JDK 17. Ниже один общий блок зависимостей для всех примеров в подтеме.

**Gradle (Groovy)**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## Роли vs authorities: префиксы ROLE_, маппинг прав, иерархия ролей

Роли и authorities — близкие, но разные понятия в Spring Security. `GrantedAuthority` — атомарное право вроде `INVOICE_READ` или «роль» с префиксом `ROLE_…`. Конвенция такова: «роль — это authority с префиксом `ROLE_`», а методы `hasRole("ADMIN")` и `hasAuthority("ROLE_ADMIN")` эквивалентны. Эта конвенция помогает отличать «грубые» роли от «тонких» прав и делает конфигурации менее неоднозначными.

Практически выгодно строить роли из набора тонких authorities. Например, `ROLE_ACCOUNTANT` может включать `INVOICE_READ`, `INVOICE_REFUND_LIMITED`, `REPORT_VIEW`. Тогда мы меняем состав роли без переписывания правил на URL/методах. Такое декомпозированное хранение прав облегчает аудит: легко понять, «из чего собрана» роль.

Маппинг прав часто происходит на границе аутентификации. Если это JWT, то клеймы (claims) `scope`/`roles` следует преобразовать в `GrantedAuthority` по вашей конвенции: например, добавлять префиксы `SCOPE_` для scope и `ROLE_` для ролей. Контроль маппинга — критичен, поскольку IdP может содержать роли чужих систем, которые не должны автоматически становиться правами в вашем приложении.

Иерархия ролей — удобный инструмент для упрощения правил. Если `ROLE_ADMIN` «содержит» `ROLE_USER`, нет смысла дублировать правила — можно задать иерархию, чтобы при проверке прав `ROLE_ADMIN` автоматически считался обладателем `ROLE_USER`. В современных версиях Spring Security это реализуется через `RoleHierarchy`/`RoleHierarchyAuthoritiesMapper` или через расширение маппинга authorities на этапе аутентификации.

Однако злоупотреблять иерархиями не стоит. Большие иерархии сложно тестировать и поддерживать; проще держать их плоскими и явными. Хорошее правило: иерархия не должна маскировать чрезмерно широкие привилегии, иначе аудит утратит прозрачность.

С точки зрения тестируемости удобно иметь «дерево» ролей в одном месте: конфигурационный бин или класс-хелпер, который переводит «крупную» роль в набор authorities. Тогда тесты легко подменяют этот бин и прогоняют матрицу решений доступа.

При работе с базой пользователей (JDBC/LDAP) храните в БД именно authorities, а уже в приложении собирайте роли. Это упрощает миграции: добавление нового атомарного права не требует перестраивать схему хранения.

В методной безопасности помните, что `hasRole` добавляет префикс `ROLE_` автоматически. Если вы уже храните `ROLE_ADMIN`, то выражение `hasAuthority('ROLE_ADMIN')` безопаснее и однозначнее, особенно когда в проекте работают несколько команд.

При многоарендности полезно различать «глобальные» и «тенант-специфичные» authorities, например `TENANT:123:INVOICE_READ`. Их можно разворачивать в обычные `GrantedAuthority` или проверять через кастомный `AuthorizationManager`/`PermissionEvaluator`, чтобы не плодить тысячи «физических» ролей.

Наконец, держите документацию на роли и права так же строго, как на REST-контракты. Как только политика скрыта в коде «между строк», вы получите дрейф поведения и конфликты при ревью.

**Java — маппинг claims → authorities + иерархия**

```java
package com.example.authz.roles;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.hierarchicalroles.RoleHierarchy;
import org.springframework.security.access.hierarchicalroles.RoleHierarchyImpl;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.mapping.GrantedAuthoritiesMapper;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.SecurityFilterChain;

import java.util.*;
import java.util.stream.Collectors;

@Configuration
public class RoleMappingConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http, JwtAuthenticationConverter converter) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(converter)));
        return http.build();
    }

    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter(GrantedAuthoritiesMapper mapper) {
        JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter();
        scopes.setAuthorityPrefix("SCOPE_");
        return new JwtAuthenticationConverter() {
            { setJwtGrantedAuthoritiesConverter(jwt -> {
                Set<String> roles = new HashSet<>(Optional.ofNullable(jwt.getClaimAsStringList("roles")).orElse(List.of()));
                Set<GrantedAuthority> authorities = new HashSet<>(Objects.requireNonNull(scopes.convert(jwt)));
                authorities.addAll(roles.stream()
                        .map(r -> "ROLE_" + r)
                        .map(org.springframework.security.core.authority.SimpleGrantedAuthority::new)
                        .collect(Collectors.toSet()));
                return mapper.mapAuthorities(authorities);
            }); }
        };
    }

    @Bean
    GrantedAuthoritiesMapper authoritiesMapper(RoleHierarchy roleHierarchy) {
        return authorities -> roleHierarchy.getReachableGrantedAuthorities(authorities);
    }

    @Bean
    RoleHierarchy roleHierarchy() {
        RoleHierarchyImpl h = new RoleHierarchyImpl();
        h.setHierarchy("ROLE_ADMIN > ROLE_USER\nROLE_USER > SCOPE_read");
        return h;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.authz.roles

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.access.hierarchicalroles.RoleHierarchy
import org.springframework.security.access.hierarchicalroles.RoleHierarchyImpl
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.authority.mapping.GrantedAuthoritiesMapper
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter
import org.springframework.security.web.SecurityFilterChain

@Configuration
class RoleMappingConfig {

    @Bean
    fun api(http: HttpSecurity, converter: JwtAuthenticationConverter): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { it.jwt { j -> j.jwtAuthenticationConverter(converter) } }
        return http.build()
    }

    @Bean
    fun jwtAuthenticationConverter(mapper: GrantedAuthoritiesMapper): JwtAuthenticationConverter {
        val scopes = JwtGrantedAuthoritiesConverter().apply { setAuthorityPrefix("SCOPE_") }
        return object : JwtAuthenticationConverter() {
            init {
                setJwtGrantedAuthoritiesConverter { jwt: Jwt ->
                    val roles = jwt.getClaimAsStringList("roles")?.toSet() ?: emptySet()
                    val fromScopes = scopes.convert(jwt)!!.toMutableSet()
                    val fromRoles = roles.map { SimpleGrantedAuthority("ROLE_$it") }.toSet()
                    mapper.mapAuthorities(fromScopes + fromRoles)
                }
            }
        }
    }

    @Bean
    fun authoritiesMapper(roleHierarchy: RoleHierarchy): GrantedAuthoritiesMapper =
        GrantedAuthoritiesMapper { auths: MutableCollection<out GrantedAuthority> ->
            roleHierarchy.getReachableGrantedAuthorities(auths)
        }

    @Bean
    fun roleHierarchy(): RoleHierarchy = RoleHierarchyImpl().apply {
        setHierarchy("ROLE_ADMIN > ROLE_USER\nROLE_USER > SCOPE_read")
    }
}
```

---

## Политики на уровне URL: `authorizeHttpRequests()` — паттерны и порядок правил

Политики на уровне URL — первый, «грубый» слой авторизации. Он отвечает за разделение зон приложения: публичные ресурсы, административные зоны, API с разными профилями доступа. Грамотно построенные правила позволяют отсеивать большинство несанкционированных обращений ещё до попадания в бизнес-слой.

Ключевой принцип — порядок правил имеет значение. Сперва ставьте наиболее конкретные соответствия пути+метода (например, `POST /api/orders/**`), затем менее конкретные (например, `/api/**`), и в самом конце — заглушку `anyRequest().authenticated()` или `denyAll()`. Такой порядок предотвращает «перекрытие» частных правил общими.

Используйте селекторы по HTTP-методам. Часто `GET /orders/{id}` должен быть доступен чтению более широкому кругу, чем `POST /orders`. Явный метод в правиле делает поведение прозрачным и уменьшает число ролей.

Правила должны быть стабильны во времени. Паттерны, завязанные на состояние («временно откроем весь `/v1/**` для демо») — путь к инцидентам. Если требуется «временно», оформите это отдельным профилем (`dev`) и не допускайте «протекания» в `prod`.

Смешивать разные миры в одной цепочке сложно и опасно. Для `/api/**` сделайте отдельный `SecurityFilterChain` с JWT и stateless, для «веба» — отдельный с formLogin. Зоны не должны конфликтовать из-за CSRF, редиректов на `/login` и т.п.

Не полагайтесь на «внутренние» сети как на политику. Даже если сервис за VPN, политику URL нужно прописывать так, будто он публичный. Сеть — отдельный слой защиты, а не замена авторизации.

Отдельно держите техэндпоинты: Actuator, Swagger UI, админ-панель. Для них, как правило, нужны более строгие правила и отдельный мониторинг попыток доступа. Не смешивайте их с «пользовательским» API.

В микросервисах хорошо работает конвенция префиксов (`/api/admin/**`, `/api/public/**`, `/internal/**`). Она облегчает чтение правил и уменьшает риск ошибочного открытия ненужного пути.

Ошибки должны быть единообразными. Для анонимов — 401 с корректным `WWW-Authenticate` (Basic/Bearer), для недостаточных прав — 403. Избегайте кастомных HTML в API — клиенты должны получать машинно-читаемые ошибки.

Пишите интеграционные тесты на политику URL. Табличные тесты (путь×метод×пользователь→статус) превращают политику в «исполняемую документацию» и предотвращают регрессии при рефакторинге.

**Java — порядок правил и методы**

```java
package com.example.authz.urls;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class UrlPolicyConfig {

    @Bean
    @Order(1)
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .requestMatchers(HttpMethod.POST, "/api/orders/**").hasAuthority("ORDER_CREATE")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(o -> o.jwt())
            .csrf(c -> c.disable());
        return http.build();
    }

    @Bean
    @Order(2)
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/", "/assets/**", "/login").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(f -> f.loginPage("/login").permitAll());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.authz.urls

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class UrlPolicyConfig {

    @Bean
    @Order(1)
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/orders/**").hasAuthority("ORDER_CREATE")
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { it.jwt() }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean
    @Order(2)
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/", "/assets/**", "/login").permitAll()
                    .anyRequest().authenticated()
            }
            .formLogin { it.loginPage("/login").permitAll() }
        return http.build()
    }
}
```

---

## Методная безопасность: SpEL в @PreAuthorize, #principal, #root, PermissionEvaluator

Методная безопасность дополняет URL-уровень, позволяя выразить доменные проверки непосредственно рядом с бизнес-методом. Это «тонкий» слой, который отвечает на вопрос «можно ли именно этому пользователю выполнить именно этот бизнес-действие для этой сущности».

Аннотация `@PreAuthorize` запускает проверку *до* выполнения метода. Внутри выражения доступен текущий `Authentication` (идентификатор — `authentication.name`), параметры метода (`#id`, `#dto`), короткие алиасы `#principal` и `#root` (мета-информация о вызове). Это позволяет писать условия вроде «владелец или админ».

`@PostAuthorize` проверяет *после* выполнения и часто используется, когда правило зависит от результата — например, «возвращаемый документ виден только владельцу». Этот механизм удобен, но не должен подменять полноценные проверки на чтение/поиск списков.

SpEL — мощный инструмент, требующий дисциплины. Старайтесь избегать сложных выражений и выносите логику в `PermissionEvaluator` или сервис, который вызываете из SpEL. Так вы получите переиспользуемую и тестируемую логику вместо «строчного» кода.

`PermissionEvaluator` — мост к доменной модели. Он позволяет написать `hasPermission(#doc, 'READ')` и внутри обратиться к базе/кэшу, проверить тенанта, состояние процесса и пр. `MethodSecurityExpressionHandler` подключает ваш evaluator к SpEL.

Методная безопасность не отменяет проверок на репозиториях. Если у вас поиск списков, ограничивайте выборку на уровне запроса (predicate по ownerId/tenantId). Иначе вы рискуете «отфильтровывать» уже найденные чужие данные на приложении — неэффективно и небезопасно.

Важно помнить про асинхронность и прокси. Аннотации работают при вызове через Spring-прокси; если вы вызываете метод из того же класса «напрямую», аннотация не сработает. Вынесите методы в отдельный бин или используйте self-injection.

Тестируйте SpEL-выражения. Ошибки в именах методов/полей всплывают только на рантайме; интеграционные тесты с `@WithMockUser` и негативные кейсы 403 помогают ловить регрессии при рефакторинге.

При JWT-аутентификации полезно прокидывать доменные claims (например, `tenant`, `department`) и использовать их в методных выражениях. Это делает правила явными и уменьшает зависимость от внешних сервисов.

Наконец, избегайте дублирования правил: если URL-уровень уже отрезает «чужой» сегмент API, не копируйте те же условия в SpEL. Пусть методный слой решает контекстные, доменные нюансы, а не повторяет периметр.

**Java — `@PreAuthorize` с `PermissionEvaluator`**

```java
package com.example.authz.methods;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler;
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.stereotype.Service;

import java.io.Serializable;
import java.util.UUID;

@Configuration
@EnableMethodSecurity
class MethodSecConfig {

    @Bean
    MethodSecurityExpressionHandler methodSecurityExpressionHandler(PermissionEvaluator pe) {
        DefaultMethodSecurityExpressionHandler h = new DefaultMethodSecurityExpressionHandler();
        h.setPermissionEvaluator(pe);
        return h;
    }

    @Bean
    PermissionEvaluator permissionEvaluator() {
        return new PermissionEvaluator() {
            @Override
            public boolean hasPermission(org.springframework.security.core.Authentication auth, Object target, Object perm) {
                if (target instanceof Doc d && "READ".equals(perm)) {
                    return d.ownerId().equals(auth.getName());
                }
                return false;
            }
            @Override
            public boolean hasPermission(org.springframework.security.core.Authentication auth, Serializable id, String type, Object perm) {
                return false;
            }
        };
    }
}

record Doc(UUID id, String ownerId, String content) {}

@Service
class DocService {

    @PreAuthorize("hasRole('ADMIN') or hasPermission(#doc, 'READ')")
    public String read(Doc doc) { return doc.content(); }

    @PreAuthorize("#ownerId == authentication.name")
    public void delete(UUID docId, String ownerId) { /* ... */ }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.authz.methods

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.access.PermissionEvaluator
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.stereotype.Service
import java.io.Serializable
import java.util.UUID

@Configuration
@EnableMethodSecurity
class MethodSecConfig {

    @Bean
    fun methodSecurityExpressionHandler(pe: PermissionEvaluator): MethodSecurityExpressionHandler =
        DefaultMethodSecurityExpressionHandler().apply { permissionEvaluator = pe }

    @Bean
    fun permissionEvaluator(): PermissionEvaluator = object : PermissionEvaluator {
        override fun hasPermission(auth: org.springframework.security.core.Authentication, target: Any?, perm: Any?): Boolean {
            val d = target as? Doc ?: return false
            return perm == "READ" && d.ownerId == auth.name
        }
        override fun hasPermission(auth: org.springframework.security.core.Authentication, targetId: Serializable?, type: String?, perm: Any?): Boolean = false
    }
}

data class Doc(val id: UUID, val ownerId: String, val content: String)

@Service
class DocService {

    @PreAuthorize("hasRole('ADMIN') or hasPermission(#doc, 'READ')")
    fun read(doc: Doc): String = doc.content

    @PreAuthorize("#ownerId == authentication.name")
    fun delete(docId: UUID, ownerId: String) { /* ... */ }
}
```

---

## Матричная модель RBAC/ABAC: атрибуты доменных объектов, многоарендность (tenant)

RBAC (Role-Based Access Control) даёт грубую матрицу «кто»×«что», тогда как ABAC (Attribute-Based Access Control) привносит контекст: атрибуты субъекта, ресурса, окружения и действия. В реальной жизни их комбинируют: роль определяет базовые возможности, а атрибуты сужают их в конкретных ситуациях.

В многоарендных системах (multi-tenant) ключевым атрибутом является `tenantId`. Пользователь с ролью `ROLE_MANAGER` одного арендатора не должен видеть данные другого, даже если у них одинаковая «роль». Это典ичный случай, где RBAC сам по себе недостаточен.

Атрибуты субъекта удобно передавать в JWT (например, `tenant`, `department`, `scope`). Их нужно валидировать и приводить к вашему формату authorities/claims. Эти атрибуты затем используйте в проверках URL/методов и в запросах к БД.

Атрибуты ресурса определяются в вашей доменной модели: владельцы, статусы, принадлежность арендаторам. Важно, чтобы эти атрибуты присутствовали в запросах к БД, а не только в объекте уже после загрузки. Иначе фильтрация «на приложении» оставит окно для утечек.

Простой приём: вводим «сквозной» `TenantContext`, который устанавливается на аутентификации (из JWT) и используется в репозиториях для добавления предикатов. Это можно сделать как вручную, так и с помощью фильтров/аспектов.

Для доменных проверок на методах используйте `PermissionEvaluator`/`AuthorizationManager`, где вы сравниваете `tenant` из токена с `tenantId` у ресурса. Это правило короткое, понятное и хорошо тестируется.

Отдельная тема — делегирование доступа: например, временная передача права «читать счета» другому пользователю того же арендатора. Это удобно моделировать как отдельное authority с ограниченным временем жизни, проверяемым в AuthorizationManager.

ABAC не должен превращаться в «произвольные скрипты». Договоритесь о наборе атрибутов и местах их применения. Избыток гибкости усложняет отладку и может ввести расхождения между сервисами.

С точки зрения производительности помните, что ABAC-проверки часто бьют в базу. Кэшируйте неизменчивые атрибуты (справочники, оргструктуру), но не кэшируйте решения доступа без строгого контроля инвалидирования.

При интеграции через API-шлюз полезно прокидывать нормализованные заголовки (например, `X-Tenant`) вниз по цепочке, чтобы сервисы-ресурсы имели единый источник правды. Но всё равно проверяйте соответствие между заголовком и claims из токена.

Наконец, документируйте матрицу: какие роли, какие атрибуты, как они комбинируются. Это поможет и разработчикам, и аудиторам, и вам самим через полгода.

**Java — ABAC: AuthorizationManager сверяет tenant в JWT и в ресурсе**

```java
package com.example.authz.abac;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authorization.AuthorizationDecision;
import org.springframework.security.authorization.AuthorizationManager;
import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.access.intercept.RequestAuthorizationContext;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;

import java.util.function.Supplier;

@Configuration
public class AbacConfig {

    @Bean
    AuthorizationManager<RequestAuthorizationContext> tenantMatchesHeader() {
        return (Supplier<Authentication> auth, RequestAuthorizationContext ctx) -> {
            Authentication a = auth.get();
            if (a instanceof JwtAuthenticationToken t) {
                String tenantClaim = t.getToken().getClaimAsString("tenant");
                String tenantHeader = ctx.getRequest().getHeader("X-Tenant");
                return new AuthorizationDecision(tenantClaim != null && tenantClaim.equals(tenantHeader));
            }
            return new AuthorizationDecision(false);
        };
    }

    @Bean
    SecurityFilterChain api(HttpSecurity http, AuthorizationManager<RequestAuthorizationContext> tenantMatchesHeader) throws Exception {
        http.securityMatcher("/api/tenant/**")
            .authorizeHttpRequests(a -> a
                .requestMatchers("/api/tenant/**").access(tenantMatchesHeader)
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(o -> o.jwt());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.authz.abac

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.authorization.AuthorizationDecision
import org.springframework.security.authorization.AuthorizationManager
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.Authentication
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.access.intercept.RequestAuthorizationContext
import java.util.function.Supplier

@Configuration
class AbacConfig {

    @Bean
    fun tenantMatchesHeader(): AuthorizationManager<RequestAuthorizationContext> =
        AuthorizationManager { auth: Supplier<Authentication>, ctx: RequestAuthorizationContext ->
            val a = auth.get()
            val ok = (a as? JwtAuthenticationToken)?.let {
                val claim = it.token.getClaimAsString("tenant")
                val header = ctx.request.getHeader("X-Tenant")
                claim != null && claim == header
            } ?: false
            AuthorizationDecision(ok)
        }

    @Bean
    fun api(http: HttpSecurity, tenantMatchesHeader: AuthorizationManager<RequestAuthorizationContext>): SecurityFilterChain {
        http.securityMatcher("/api/tenant/**")
            .authorizeHttpRequests {
                it.requestMatchers("/api/tenant/**").access(tenantMatchesHeader)
                    .anyRequest().authenticated()
            }
            .oauth2ResourceServer { it.jwt() }
        return http.build()
    }
}
```

---

## Разделение ответственности: «правила в коде» vs «правила в БД/политиках»

Разумная архитектура не кладёт весь доступ «в код» и не пытается переложить всё на БД. Нужен баланс. Грубые правила периметра — в коде (URL/методы, роли), тонкие доменные и массовые фильтры по данным — ближе к данным (предикаты в запросах, представления, Row-Level Security).

Row-Level Security (RLS) в PostgreSQL — мощный инструмент, позволяющий выражать «кто видит какие строки» на уровне СУБД. Включив RLS на таблице и определив политику по `tenant_id`, вы гарантируете, что ни один запрос (даже «забывчивый») не вытащит чужие строки. Это особенно ценно, если у вас несколько сервисов и хочется единообразия.

Однако RLS не заменяет приложенческие правила. Сервис всё равно решает «когда» и «почему» можно выполнить операцию, проверяет состояния процессов, бизнес-лимиты. БД просто гарантирует отсутствие утечек из-за ошибки в WHERE.

Политики в БД требуют дисциплины с подключениями: переменная контекста `current_tenant` должна устанавливаться *перед* запросом. Это можно делать на уровне пула/фильтра, в транзакционном интерсепторе или прямо в `DataSource`-декораторе.

В коде выражайте правила, которые завязаны на действия, а не только на данные. Например, «возврат возможен только в течение 30 дней» — это методная проверка. Она не относится к строкам БД напрямую.

Смешивать RLS и методную безопасность можно и нужно: метод ограничивает действие, RLS — видимость строк. Такое «двойное запирание» уменьшает вероятность обхода через ошибку на одном уровне.

Документируйте, где живёт какое правило. Таблица истин должна показывать: периметр, метод, доменная проверка, фильтры БД. Иначе при расследованиях инцидентов вы потратите дни на поиск «того самого» куска логики.

Тестируйте RLS на уровне интеграционных тестов. Поднимайте Postgres (Testcontainers), устанавливайте контекст арендатора и проверяйте, что «чужие» строки не выдаются даже при прямом доступе к репозиторию.

Избегайте «жёсткого кодирования» tenantId в запросах. Вытаскивайте tenant из контекста (JWT/заголовок), устанавливайте в сессию БД и используйте политики. Это делает код чище и уменьшает риск забыть фильтр.

Наконец, не забывайте про производительность: политики RLS — это дополнительные условия; индексы по `tenant_id` и по часто используемым атрибутам обязательны, иначе потеряете в скорости.

**SQL — включение RLS и политика по tenant**

```sql
-- Пример для PostgreSQL
alter table invoice enable row level security;

create policy invoice_tenant_policy
on invoice
using (tenant_id = current_setting('app.tenant_id', true));
```

**Java — установка tenant в сессии БД + репозиторий**

```java
package com.example.authz.rls;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Repository;

@Component
public class TenantSession {

    private final JdbcTemplate jdbc;

    public TenantSession(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    public void setTenant(String tenant) {
        jdbc.execute("select set_config('app.tenant_id', '" + tenant + "', true)");
    }
}

@Repository
class InvoiceRepository {
    private final JdbcTemplate jdbc;

    InvoiceRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    public int countVisible() {
        return jdbc.queryForObject("select count(*) from invoice", Integer.class);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.authz.rls

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.stereotype.Component
import org.springframework.stereotype.Repository

@Component
class TenantSession(private val jdbc: JdbcTemplate) {
    fun setTenant(tenant: String) {
        jdbc.execute("select set_config('app.tenant_id', '$tenant', true)")
    }
}

@Repository
class InvoiceRepository(private val jdbc: JdbcTemplate) {
    fun countVisible(): Int = jdbc.queryForObject("select count(*) from invoice", Int::class.java)!!
}
```

> В реальном коде экранируйте значения, используйте `PreparedStatement`, а установку контекста оборачивайте в аспект/фильтр до выполнения запросов.

---

## Аудит отказов/успехов: ApplicationEvent’ы, AuditEvents, логирование

Аудит в безопасности — это не «приятное дополнение», а обязательная часть. Он отвечает на вопросы «кто пытался зайти», «кто получил отказ», «кто выполнил чувствительную операцию». Без аудита вы не отличите баг от атаки и не докажете соблюдение регламентов.

Spring Security публикует события аутентификации: `AuthenticationSuccessEvent`, `AbstractAuthenticationFailureEvent` и др. На авторизации на уровне HTTP/методов публикуются события из семейства `AuthorizationEvent`. Подписавшись на них, вы можете логировать факты, метаданные запроса и проводить корреляцию с бизнес-логами.

Spring Boot предоставляет `AuditEventRepository`, куда можно писать «аудит-события» приложения (`type`, `principal`, `data`). Это удобно для унификации и последующей интеграции с SIEM/лог-хранилищами. Не путайте с «детальными логами» — аудит должен быть лаконичным и структурированным.

Следите за тем, чтобы аудит не раскрывал чувствительные данные: не логируйте токены, пароли, полные объекты. Достаточно идентификаторов, типов операций, статусов и ключевых атрибутов (tenant, ip).

Единообразие кодов ошибок важно и для аудита. Унификация 401/403, наличие `WWW-Authenticate` и Problem Details делает анализ проще: можно строить метрики «уровня отказов» по зонам и видеть всплески.

В микросервисах полезно расширять аудиторским полем trace-id/span-id. Тогда события безопасности «сцепляются» с бизнес-трассировкой и быстрее расследуются инциденты. Встраивайте это в MDC и добавляйте в поля аудита.

Логи — не хранилище навсегда. Продумайте ретеншн и ротацию, чтобы важные данные не исчезали раньше времени, но и не забивали диски. Чувствительные зоны (логины, админ-запросы) можно хранить дольше.

Порог алертов имеет значение. Слишком низкий — шум, слишком высокий — пропущенные атаки. Начните со «всплесков» 401/403, частоты ошибок логина по пользователю/IP, и постепенно уточняйте.

Тестируйте аудит. Моки `ApplicationEventPublisher`/интеграционные тесты должны проверять, что событие публикуется при успехе/отказе и содержит нужные поля. Это так же важно, как и тесты на 401/403.

Наконец, разделяйте аудит доступа и аудит изменений данных. Первое — зона Spring Security, второе — зона бизнес-событий (domain events). В связке они дают полную картину.

**Java — слушатели событий аутентификации/авторизации и аудит**

```java
package com.example.authz.audit;

import org.springframework.boot.actuate.audit.AuditEvent;
import org.springframework.boot.actuate.audit.AuditEventRepository;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.event.AbstractAuthenticationFailureEvent;
import org.springframework.security.authentication.event.AuthenticationSuccessEvent;
import org.springframework.security.authorization.event.AuthorizationDeniedEvent;
import org.springframework.security.authorization.event.AuthorizationGrantedEvent;

import java.util.Map;

@Configuration
public class SecurityAuditListeners {

    private final AuditEventRepository audit;

    public SecurityAuditListeners(AuditEventRepository audit) {
        this.audit = audit;
    }

    @Configuration
    static class AuthSuccessListener implements ApplicationListener<AuthenticationSuccessEvent> {
        private final AuditEventRepository audit;
        AuthSuccessListener(AuditEventRepository audit) { this.audit = audit; }
        @Override public void onApplicationEvent(AuthenticationSuccessEvent event) {
            audit.add(new AuditEvent(event.getAuthentication().getName(), "AUTH_SUCCESS", Map.of()));
        }
    }

    @Configuration
    static class AuthFailureListener implements ApplicationListener<AbstractAuthenticationFailureEvent> {
        private final AuditEventRepository audit;
        AuthFailureListener(AuditEventRepository audit) { this.audit = audit; }
        @Override public void onApplicationEvent(AbstractAuthenticationFailureEvent event) {
            audit.add(new AuditEvent(event.getAuthentication().getName(), "AUTH_FAILURE",
                    Map.of("error", event.getException().getClass().getSimpleName())));
        }
    }

    @Configuration
    static class AuthzListeners implements ApplicationListener<org.springframework.context.ApplicationEvent> {
        private final AuditEventRepository audit;
        AuthzListeners(AuditEventRepository audit) { this.audit = audit; }
        @Override public void onApplicationEvent(org.springframework.context.ApplicationEvent e) {
            if (e instanceof AuthorizationGrantedEvent<?> g) {
                audit.add(new AuditEvent(g.getAuthentication().get().getName(), "AUTHZ_GRANTED",
                        Map.of("details", g.getAuthorizationDecision().toString())));
            } else if (e instanceof AuthorizationDeniedEvent<?> d) {
                audit.add(new AuditEvent(d.getAuthentication().get().getName(), "AUTHZ_DENIED",
                        Map.of("details", d.getAuthorizationDecision().toString())));
            }
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.authz.audit

import org.springframework.boot.actuate.audit.AuditEvent
import org.springframework.boot.actuate.audit.AuditEventRepository
import org.springframework.context.ApplicationEvent
import org.springframework.context.ApplicationListener
import org.springframework.context.annotation.Configuration
import org.springframework.security.authentication.event.AbstractAuthenticationFailureEvent
import org.springframework.security.authentication.event.AuthenticationSuccessEvent
import org.springframework.security.authorization.event.AuthorizationDeniedEvent
import org.springframework.security.authorization.event.AuthorizationGrantedEvent

@Configuration
class SecurityAuditListeners(private val audit: AuditEventRepository) {

    @Configuration
    class AuthSuccessListener(private val audit: AuditEventRepository) : ApplicationListener<AuthenticationSuccessEvent> {
        override fun onApplicationEvent(event: AuthenticationSuccessEvent) {
            audit.add(AuditEvent(event.authentication.name, "AUTH_SUCCESS", mapOf()))
        }
    }

    @Configuration
    class AuthFailureListener(private val audit: AuditEventRepository) : ApplicationListener<AbstractAuthenticationFailureEvent> {
        override fun onApplicationEvent(event: AbstractAuthenticationFailureEvent) {
            audit.add(AuditEvent(event.authentication.name, "AUTH_FAILURE",
                mapOf("error" to event.exception.javaClass.simpleName)))
        }
    }

    @Configuration
    class AuthzListeners(private val audit: AuditEventRepository) : ApplicationListener<ApplicationEvent> {
        override fun onApplicationEvent(e: ApplicationEvent) {
            when (e) {
                is AuthorizationGrantedEvent<*> -> audit.add(
                    AuditEvent(e.authentication.get().name, "AUTHZ_GRANTED", mapOf("details" to e.authorizationDecision.toString()))
                )
                is AuthorizationDeniedEvent<*> -> audit.add(
                    AuditEvent(e.authentication.get().name, "AUTHZ_DENIED", mapOf("details" to e.authorizationDecision.toString()))
                )
            }
        }
    }
}
```

> Чтобы `AuditEventRepository` появился, добавьте стартер `spring-boot-starter-actuator`. Не забудьте настроить ротацию логов и уровни детализации.


Вы правы — в прошлой подтеме я недобрал абзацев в некоторых пунктах. Исправляю: ниже — та же **подтема 5** с расширенными объяснениями, чтобы **каждый пункт содержал ≥ 10 содержательных абзацев** (код, как и прежде, присутствует и **не учитывается** в подсчёте).

# 5. Сеансы, CSRF, CORS и заголовки

## Сессии: режимы `ALWAYS/IF_REQUIRED/NEVER/STATELESS`, фиксация и политика cookie

Первое, что нужно усвоить про HTTP-сессии в Spring Security: это всего лишь транспорт для хранения аутентификационного контекста между запросами. Они удобны для браузерных приложений, но избыточны для REST-API. Правильный выбор `SessionCreationPolicy` напрямую влияет на масштабируемость и устойчивость.

Режим `ALWAYS` исторически использовался в некоторых MVC-приложениях, но сегодня он почти не нужен: вы создаёте сессии даже там, где они не понадобятся. Это неоправданный оверхед по памяти/GC и нагрузке на распределённое хранилище сессий, если вы его используете.

`IF_REQUIRED` — оптимальный дефолт для formLogin: сессия появится только после успешного входа. До логина анонимные GET-запросы идут без сессии, а после — браузер будет таскать `JSESSIONID`, и сервер восстановит `SecurityContext` из хранилища.

`NEVER` используется, когда внешние компоненты уже создают сессию (например, SSO-прокси), а ваше приложение должно лишь «приклеиться» к ней. Вы их не создаёте, но уважаете, если они есть. Это достаточно редкий режим.

`STATELESS` — стандарт для микросервисов: сервер не хранит состояние авторизации, а читает его из каждого запроса (обычно Bearer/JWT). Это резко упрощает горизонтальное масштабирование: нет липких сессий, нет внешнего кэша сессий, меньше межузловой синхронизации.

Критически важно включить защиту от фиксации сессии (session fixation). Spring Security по умолчанию мигрирует идентификатор сессии после логина — это означает, что навязанный злоумышленником `JSESSIONID` перестанет работать сразу после аутентификации. Не отключайте `migrateSession()` без веской причины.

Куки сессии — вторая линия обороны. Выставляйте `Secure` (только HTTPS), `HttpOnly` (недоступно JS) и корректный `SameSite`. Для классического кабинета хватит `Lax`, для кросс-доменных SSO-потоков нужен `None` **только** вместе с HTTPS. Неправильный `SameSite` — частая причина «таинственных» разлогинов.

Планируйте конкуренцию сессий. Для чувствительных систем полезно ограничить количество одновременных сессий на пользователя до 1–2. Это снижает риск «забытых» сессий на общедоступных ПК и затрудняет злоумышленнику скрытно пользоваться украденной cookie.

Если вы используете Spring Session и Redis, думайте о TTL и эвикации. Слишком длинная жизнь сессий — повышенный риск компрометации; слишком короткая — ухудшение UX. Балансируйте и автоматически вычищайте «осиротевшие» записи.

Наконец, помните, что в гибридных приложениях удобно развести «мир сессий» и «мир API». Делайте отдельную `SecurityFilterChain` для `/api/**` в режиме `STATELESS` и не смешивайте её с formLogin-частью — так вы избежите странных конфликтов CSRF, редиректов на `/login` и утечек cookie в API.

**Gradle (Groovy)**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

**Java — политика сессий, миграция, конкуренция и cookie**

```java
package com.example.security.sessions;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SessionSecurityConfig {

    @Bean
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(a -> a.requestMatchers("/login","/public/**").permitAll().anyRequest().authenticated())
          .formLogin(f -> f.loginPage("/login").permitAll())
          .sessionManagement(sm -> sm
              .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
              .sessionFixation(sf -> sf.migrateSession())
              .sessionConcurrency(sc -> sc.maximumSessions(1).maxSessionsPreventsLogin(true))
          );
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.sessions

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SessionSecurityConfig {

    @Bean
    fun web(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests { it.requestMatchers("/login","/public/**").permitAll().anyRequest().authenticated() }
            .formLogin { it.loginPage("/login").permitAll() }
            .sessionManagement {
                it.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                    .sessionFixation { sf -> sf.migrateSession() }
                    .sessionConcurrency { sc -> sc.maximumSessions(1).maxSessionsPreventsLogin(true) }
            }
        return http.build()
    }
}
```

**application.yml — атрибуты cookie сессии**

```yaml
server:
  servlet:
    session:
      cookie:
        secure: true         # только HTTPS
        http-only: true
        same-site: lax       # для SSO может потребоваться none (+ HTTPS)
```


Дополнительно подумайте о телеметрии сессий: подписка на события `SessionCreatedEvent/DestroyedEvent` помогает видеть аномалии (например, всплеск новых сессий с одного IP). Это полезная метрика для детектирования брут-форса.

Отдельная тонкость — перенос контекста в асинхронные задачи. `SecurityContext` хранится в `ThreadLocal`, и при offload в общий пул он теряется. Не привязывайтесь к нему в задачах бэкграунда; лучше явно передавайте нужные атрибуты (userId/tenant).

Частая ошибка — держать большой объём данных в сессии (слепки больших объектов, настройки UI). Это увеличивает сетевой/дисковый трафик в кластере и замедляет каждый запрос. Сессия — не кэш; держите там минимум.

Регулярно проверяйте политику «принудительного выхода» при смене пароля: после reset все активные сессии должны инвалидироваться. В Spring Session это делается удалением записей по ключу пользователя в бекенде (Redis/JDBC).

---

## CSRF: когда нужен (формы/браузер), когда отключается (чистый API)

CSRF актуален, когда браузер автоматически прикладывает учётные данные (cookie) к запросу, а злоумышленник может побудить жертву выполнить нежелательный POST/PUT/DELETE. Без дополнительного токена сервер не отличит «добровольное» действие от подставленного.

В чистом REST-API на Bearer/JWT CSRF не нужен: браузер не добавляет заголовок `Authorization` сам по себе, и «пассивная» атака не сработает. Поэтому для `/api/**` следует отключать проверку токена CSRF, чтобы не получать ложные 403 в Postman/интеграциях.

В MVC-приложениях держите CSRF включённым: Spring Security автоматически проверяет токен на «опасных» методах. Для классических HTML-форм токен помещается в скрытое поле; для SPA — в заголовок.

Для SPA удобен `CookieCsrfTokenRepository`: токен в cookie (не HttpOnly), JS копирует его в заголовок `X-CSRF-TOKEN`. Это согласуется с `fetch`/Axios и позволяет работать без серверного рендеринга. Не забывайте включить `credentials: 'include'`, если вы используете cookie-сессии.

CSRF — не панацея от XSS. Если в приложении есть XSS, злоумышленник украдёт CSRF-токен и выполнит запросы от имени жертвы. Поэтому пара CSRF+CSP обязательна: первая защищает от «случайного» отправления формы, вторая — от исполнения вредного JS.

Убедитесь, что `/logout` защищён. По умолчанию Spring Security требует CSRF для `POST /logout`. Это правильно: иначе внешняя страница могла бы разлогинить пользователя «невинной» картинкой/формой.

Осторожно с прокси: некоторые балансировщики/веб-серверы переписывают/удаляют заголовки. Проверьте, что `X-CSRF-TOKEN` не теряется на пути и что `SameSite`/CORS не блокируют доставку cookie с токеном в SPA.

В сценариях SSO учитывайте редиректы: после успешного OIDC-логина вы можете вернуть пользователя на страницу с формой. Если форма ожидает CSRF-токен, позаботьтесь о его генерации/обновлении, иначе получите редкие 403 «иногда».

Тестируйте негативные кейсы: без токена запрос должен стабильно возвращать 403, с токеном — 2xx. Это ловит случайные отключения CSRF и регрессии при рефакторинге конфигурации.

И наконец, не путайте отключение CSRF «для удобства» с осознанным решением. Если у вас formLogin и cookie-сессии — CSRF нужен почти всегда. И наоборот: если это чистый JSON-API с Bearer — он не нужен.


**Java — включённый CSRF + выдача токена в cookie для SPA**

```java
package com.example.security.csrf;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class CsrfConfig {

    @Bean
    SecurityFilterChain http(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(a -> a
              .requestMatchers("/api/**").authenticated()
              .anyRequest().permitAll()
          )
          .csrf(csrf -> csrf
              .ignoringRequestMatchers("/api/**") // stateless API
              .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
          );
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.csrf

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.csrf.CookieCsrfTokenRepository

@Configuration
class CsrfConfig {

    @Bean
    fun http(http: HttpSecurity): SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers("/api/**").authenticated()
                    .anyRequest().permitAll()
            }
            .csrf {
                it.ignoringRequestMatchers("/api/**")
                    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            }
        return http.build()
    }
}
```

---

## CORS: allowed origins/headers/methods, preflight, различие с CSRF

CORS регулирует, может ли JS с одного источника (Origin) обращаться к другому. Это политика браузера, а не сервера. Сервер лишь сообщает свои намерения заголовками `Access-Control-Allow-*`, а браузер принимает решение.

CORS и CSRF решают разные задачи: первый — про кросс-доменный JS-доступ, второй — про подмену намерений при cookie-сессиях. Можно иметь строгий CSRF и одновременно «адресный» CORS для доверенных фронтов — это нормальная практика.

Preflight-запрос (`OPTIONS`) возникает, когда «настоящий» запрос не является «простым» (нестандартный заголовок/метод). Сервер должен корректно ответить разрешёнными методами/заголовками/источниками, иначе браузер даже не отправит основной запрос.

Никогда не ставьте `allowedOrigins="*"` вместе с `allowCredentials=true`: браузер такой ответ отвергнет. Если вам нужны cookie/креды, перечисляйте конкретные домены фронтенда. Это жёстко, но безопасно и предсказуемо.

Думайте о кэше: ответы на CORS зависят от Origin, поэтому выставляйте `Vary: Origin`. Иначе CDN/прокси могут кэшировать ответ для одного Origin и отдать его другому, ломая семантику и безопасность.

Экспорт заголовков (`Access-Control-Expose-Headers`) нужен, если клиент должен читать ответные заголовки (`Location`, `X-Request-Id`). По умолчанию браузер «не показывает» большинство заголовков JS-коду, пока вы их явно не перечислите.

`Access-Control-Max-Age` снижает нагрузку на сервер, кэшируя preflight-решение в браузере. Это улучшает UX для SPA, активно делающих PATCH/PUT/DELETE. Но не ставьте слишком большой период — вы усложните оперативные изменения CORS-политики.

CORS не относится к сервер-к-серверу. Postman «всё работает» не потому, что сервер настроен, а потому что Postman — не браузер. Проверяйте CORS в браузере (или через E2E-тесты), иначе получите ложные ощущения «всё ок».

Разные зоны могут требовать разные CORS-правила. Удобно регистрировать `CorsConfigurationSource` с привязкой к путям (`/api/**`, `/auth/**`). Это позволяет не открывать лишнее и точно обслуживать только нужные маршруты.

Не забывайте, что WebSocket — отдельная история: классический CORS к нему не применяется, но браузер всё равно ограничивает подключение по Origin на уровне протокола. Для ws/wss настройте проверку Origin на бэкенде отдельно.

**Java — адресный CORS + preflight**

```java
package com.example.security.cors;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.*;

import java.util.List;

@Configuration
public class CorsSecurityConfig {

    @Bean
    SecurityFilterChain corsChain(HttpSecurity http) throws Exception {
        http.cors(c -> {}).csrf(cs -> cs.disable()); // для JSON API
        http.authorizeHttpRequests(a -> a.anyRequest().permitAll());
        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration cfg = new CorsConfiguration();
        cfg.setAllowedOrigins(List.of("https://app.example.com"));
        cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
        cfg.setAllowedHeaders(List.of("Authorization","Content-Type"));
        cfg.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", cfg);
        return source;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.cors

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain
import org.springframework.web.cors.CorsConfiguration
import org.springframework.web.cors.CorsConfigurationSource
import org.springframework.web.cors.UrlBasedCorsConfigurationSource

@Configuration
class CorsSecurityConfig {

    @Bean
    fun corsChain(http: HttpSecurity): SecurityFilterChain {
        http.cors { }.csrf { it.disable() }
        http.authorizeHttpRequests { it.anyRequest().permitAll() }
        return http.build()
    }

    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource =
        UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/api/**", CorsConfiguration().apply {
                allowedOrigins = listOf("https://app.example.com")
                allowedMethods = listOf("GET","POST","PUT","DELETE","OPTIONS")
                allowedHeaders = listOf("Authorization","Content-Type")
                allowCredentials = true
            })
        }
}
```

---

## Security headers: HSTS, X-Content-Type-Options, X-Frame-Options, CSP — минимум по умолчанию

Заголовки безопасности — дешёвый и эффективный слой защиты на периметре браузера. Они не требуют изменения бизнес-логики, но закрывают целые классы векторов и помогают браузеру «поступать правильно».

HSTS переводит домен на «только HTTPS»: после первого визита браузер больше не попробует HTTP-вариант. Для прод-доменов включайте `includeSubDomains` и по готовности — `preload` (добавление в список браузеров), но делайте это после полной миграции на HTTPS во всех субдоменах.

`X-Content-Type-Options: nosniff` запрещает «угадывать» MIME-тип. Это предотвращает исполнение скриптов, если кто-то отдаёт их под видом изображений/текста. Это один из самых безопасных и «бесплатных» заголовков.

`X-Frame-Options: DENY|SAMEORIGIN` защищает от кликджекинга. Если у вас есть легитимные iframe (например, админка внутри портала), используйте `SAMEORIGIN`, иначе — `DENY`. Для гибких сценариев можно перейти на директивы `frame-ancestors` в CSP.

CSP — главный инструмент против XSS. Начинайте с `default-src 'self'; object-src 'none'` и постепенно разрешайте необходимые источники (`img-src`, `font-src`, `connect-src`). Избегайте `unsafe-inline`; если нужно исполнять встроенные скрипты, используйте nonce/хэши.

Полезные дополнительные политики: `Referrer-Policy: no-referrer` (или `strict-origin-when-cross-origin`), `Permissions-Policy` (ограничение доступа к камере/гео и т.п.), `Cross-Origin-Opener-Policy`/`Cross-Origin-Embedder-Policy` для изоляции контекстов. Они уменьшают зону неожиданностей в современных браузерах.

Для API на том же домене, что и UI, заголовки должны быть единообразны. Даже если JSON сам по себе не «исполняется», XFO/CSP/HSTS на едином домене предотвращают смешанные сценарии атак.

Следите за сторонними скриптами (аналитика, виджеты). Широкая CSP ради «удобной интеграции» — самый быстрый путь вернуть XSS. Лучше добавить конкретные источники и использовать Subresource Integrity (SRI) там, где возможно.

Тестируйте заголовки. Chrome DevTools/Firefox покажут нарушения CSP, а в CI можно проверять наличие ключевых заголовков (RestAssured/HTTP-тесты). Это дисциплинирует и предотвращает «забыли включить HSTS на новом сервисе».

Помните, что слишком строгая CSP может сломать легитимные сценарии (например, загрузку шрифтов с внешнего CDN). Идите итеративно: сначала `Report-Only`, собирайте отчёты, затем включайте «боевой» режим.


**Java — включение CSP/HSTS/XFO/Nosniff**

```java
package com.example.security.headers;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class HeadersConfig {

    @Bean
    SecurityFilterChain headersChain(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(a -> a.anyRequest().permitAll())
          .headers(h -> h
              .httpStrictTransportSecurity(hsts -> hsts.includeSubDomains(true).preload(true))
              .xssProtection(x -> x.block(true))
              .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'; object-src 'none'"))
              .frameOptions(f -> f.sameOrigin())
              .contentTypeOptions(c -> {}) // включает X-Content-Type-Options: nosniff
          );
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.headers

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class HeadersConfig {

    @Bean
    fun headersChain(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().permitAll() }
            .headers {
                it.httpStrictTransportSecurity { h -> h.includeSubDomains(true).preload(true) }
                    .xssProtection { x -> x.block(true) }
                    .contentSecurityPolicy { csp -> csp.policyDirectives("default-src 'self'; object-src 'none'") }
                    .frameOptions { f -> f.sameOrigin() }
                    .contentTypeOptions { } // nosniff
            }
        return http.build()
    }
}
```

---

## Logout/remember-me: риски и безопасные настройки

Logout — это не просто редирект. Он должен очистить аутентификационный контекст, инвалидировать серверную сессию и удалить все чувствительные cookie. В противном случае старое окно браузера может «воскресить» сессию.

Remember-me увеличивает удобство, но повышает риск компрометации. Если он включён, используйте «persistent» вариант с серверным хранилищем (JDBC) и длинным случайным секретом. Это позволяет отозвать токен, если cookie утекла, и контролировать срок жизни.

Никогда не воспринимайте remember-me как замену MFA. Он облегчает повторный вход, но не защищает чувствительные операции. Для подтверждения переводов/изменения настроек запросите повторную аутентификацию или второй фактор.

В stateless-API «выход» — это, по сути, забывание refresh-токена на стороне клиента и отзыв (ревокация) на стороне Authorization Server. Ресурс-сервер обычно не «убивает» токены — он просто проверяет подпись и сроки, и по истечении токен перестаёт работать.

Защитите `/logout` от CSRF: либо оставляйте `POST /logout` (по умолчанию), либо, если политика требует `GET /logout`, добавьте подтверждение и защиту от открытых редиректов (валидируйте `redirect`).

Соблюдайте хорошую гигиену cookie: удаляйте и `JSESSIONID`, и `remember-me`, выставляйте им `Path`/`Domain` корректно. Иначе может остаться «вторая» cookie, которую браузер выберет «не ту».

Для persistent remember-me настройте регулярную чистку просроченных записей в БД. С ростом трафика таблица токенов будет пухнуть, а поиск по ней — замедляться. Индексы по (series, username) обязательны.

Подумайте о «девайсных» remember-me: храните «имя устройства»/браузера, чтобы пользователь мог посмотреть список активных «доверенных» устройств и отозвать отдельные. Это улучшает безопасность и UX.

E2E-тесты должны проверять, что после logout защищённые страницы больше не доступны, и что cookie действительно удалены. Такой тест ловит регрессии при правках фильтров и обновлениях Spring.

Если у вас OIDC SSO, поддержите Single Logout (front-channel/back-channel) со стороны IdP. Это убирает «висячие» сессии в нескольких приложениях одновременно, что важно для корпоративных сред.


**Java — безопасный logout + persistent remember-me (демо)**

```java
package com.example.security.logout;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.rememberme.InMemoryTokenRepositoryImpl;

@Configuration
public class LogoutRememberMeConfig {

    @Bean
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(a -> a.anyRequest().authenticated())
          .formLogin(f -> f.permitAll())
          .rememberMe(r -> r
              .key("super-long-random-server-side-secret-key")
              .tokenValiditySeconds(60 * 60 * 24 * 7) // 7 дней
              .tokenRepository(new InMemoryTokenRepositoryImpl()) // для прода используйте JDBC
          )
          .logout(l -> l
              .logoutUrl("/logout")
              .deleteCookies("JSESSIONID", "remember-me")
              .invalidateHttpSession(true)
              .clearAuthentication(true)
              .logoutSuccessUrl("/login?logout")
          );
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.logout

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.rememberme.InMemoryTokenRepositoryImpl

@Configuration
class LogoutRememberMeConfig {

    @Bean
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .formLogin { it.permitAll() }
            .rememberMe {
                it.key("super-long-random-server-side-secret-key")
                    .tokenValiditySeconds(60 * 60 * 24 * 7)
                    .tokenRepository(InMemoryTokenRepositoryImpl())
            }
            .logout {
                it.logoutUrl("/logout")
                    .deleteCookies("JSESSIONID", "remember-me")
                    .invalidateHttpSession(true)
                    .clearAuthentication(true)
                    .logoutSuccessUrl("/login?logout")
            }
        return http.build()
    }
}
```

---

## Ограничение частоты (rate limit) на уровне прокси/фильтров как доп. слой

Rate limit — это «пояс безопасности» до логики аутентификации/авторизации. Он гасит брут-форс и бурсты еще до того, как запросы начнут грузить JVM/БД. Идеальное место — фронтовой прокси/шлюз: там виден истинный исходный IP и есть дешёвая, быстрая реализация бакетов.

Логин — первый кандидат на жёсткие лимиты. Ограничивайте попытки по комбинации IP+username, вводите капчу/замедление после нескольких неудач. Это существенно повышает стоимость атаки и разгружает каталог/БД.

API тоже нуждается в лимитах: «общая» квота на клиента, более строгие лимиты на тяжёлые операции (отчёты, экспорты), а также защита «админских» путей. Если используете JWT, учитывайте `sub` или `client_id` как ключ.

В приложении можно держать лёгкий фильтр-предохранитель. Он не заменит прокси, но предотвратит локальные «заливки» (например, неправильный скрипт тестов). Пишите его неблокирующим, без сетевых вызовов и с понятной семантикой.

Возвращайте 429 (Too Many Requests) и при возможности `Retry-After`. Это улучшает поведение добросовестных клиентов и позволяет правильно настроить ретраи. Логируйте ключевые атрибуты (ip, path, client) и поднимайте алерты на всплески.

Для распределённых лимитов возьмите Redis/Envoy/Nginx-модуль: локальная память не видит весь кластер, и злоумышленник может «раскидать» нагрузку по узлам. Координация лимитов вне приложения — более надёжная стратегия.

Не забывайте исключать служебные пути (health, readiness) из лимитов — иначе оркестратор/балансировщик может решить, что сервис «плохой», из-за искусственно созданных 429 на пробах.

Комбинируйте лимиты с «заморозкой» учётной записи при аномалиях: странные паттерны (много неудачных попыток с разных IP) — повод временно блокировать вход или потребовать MFA. Это уже не только rate limit, но и поведенческая защита.

Покрывайте лимиты тестами: проверьте, что 51-й запрос за 10 секунд получает 429, а через 10 секунд окно обнуляется. Это защитит от случайных регрессий при рефакторингах фильтров и конфигураций.

Подумайте о метриках: экспортируйте количество 429, распределение по путям, верхние IP/клиенты. Это позволит замечать реальные атаки раньше, чем они выльются в инцидент производительности.


**Java — простой in-memory rate-limit фильтр (по IP)**

```java
package com.example.security.ratelimit;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.*;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

class SimpleRateLimiter {
    private static final long WINDOW_MS = 10_000; // 10 сек
    private static final int LIMIT = 50;          // 50 запросов на окно
    private record Bucket(long windowStart, int count) {}
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    synchronized boolean allow(String key) {
        long now = Instant.now().toEpochMilli();
        Bucket b = buckets.getOrDefault(key, new Bucket(now, 0));
        if (now - b.windowStart() > WINDOW_MS) {
            buckets.put(key, new Bucket(now, 1));
            return true;
        }
        if (b.count() + 1 > LIMIT) return false;
        buckets.put(key, new Bucket(b.windowStart(), b.count() + 1));
        return true;
    }
}

class RateLimitFilter extends OncePerRequestFilter {
    private final SimpleRateLimiter limiter = new SimpleRateLimiter();

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String key = req.getRemoteAddr();
        if (!limiter.allow(key)) {
            res.setStatus(429);
            res.setHeader("Retry-After", "10");
            res.getWriter().write("Too Many Requests");
            return;
        }
        chain.doFilter(req, res);
    }
}

@Configuration
class RateLimitConfig {

    @Bean
    SecurityFilterChain limited(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a.anyRequest().permitAll())
            .addFilterBefore(new RateLimitFilter(), UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.security.ratelimit

import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
import org.springframework.web.filter.OncePerRequestFilter
import java.io.IOException
import java.time.Instant
import java.util.concurrent.ConcurrentHashMap

private class SimpleRateLimiter {
    private val windowMs = 10_000L
    private val limit = 50
    private data class Bucket(val windowStart: Long, val count: Int)
    private val buckets = ConcurrentHashMap<String, Bucket>()

    @Synchronized
    fun allow(key: String): Boolean {
        val now = Instant.now().toEpochMilli()
        val b = buckets.getOrDefault(key, Bucket(now, 0))
        return if (now - b.windowStart > windowMs) {
            buckets[key] = Bucket(now, 1); true
        } else if (b.count + 1 > limit) {
            false
        } else {
            buckets[key] = Bucket(b.windowStart, b.count + 1); true
        }
    }
}

private class RateLimitFilter : OncePerRequestFilter() {
    private val limiter = SimpleRateLimiter()

    @Throws(ServletException::class, IOException::class)
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        val key = req.remoteAddr
        if (!limiter.allow(key)) {
            res.status = 429
            res.setHeader("Retry-After", "10")
            res.writer.write("Too Many Requests")
            return
        }
        chain.doFilter(req, res)
    }
}

@Configuration
class RateLimitConfig {
    @Bean
    fun limited(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().permitAll() }
            .addFilterBefore(RateLimitFilter(), UsernamePasswordAuthenticationFilter::class.java)
        return http.build()
    }
}
```

# 6. Stateless API и JWT

Ниже — цельная глава про построение stateless-API на Spring Security 6.x (Boot 3.3.x) с JWT. Мы будем исходить из модели «ресурс-сервер» (Resource Server), где сервис валидирует готовые access-токены, а аутентификацию/выдачу токенов делает внешний Authorization Server (IdP).

**Gradle (Groovy) — общие зависимости для всей подтемы**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2024.0.3"
    }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'   // для refresh в клиенте и token relay
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'      // для примера с Gateway
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2024.0.3")
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
    implementation("org.springframework.cloud:spring-cloud-starter-gateway")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## Spring Resource Server: `oauth2ResourceServer().jwt()`, JwtDecoder/keys (JWKs)

Stateless-подход означает, что сервер не хранит сессии: каждый запрос несёт самодостаточное доказательство — access-токен (обычно JWT) в `Authorization: Bearer ...`. Это облегчает горизонтальное масштабирование и упрощает отказоустойчивость.

Ресурс-сервер не «логинит» пользователей, а только **валидирует** токен: проверяет подпись, срок действия, аудиторию/издателя и, при необходимости, дополнительные утверждения. Выдачу токена делает IdP/Authorization Server.

В Spring Boot 3.x это настраивается одной строкой в DSL: `http.oauth2ResourceServer().jwt()`. Под капотом будет нужен `JwtDecoder` — компонент, который умеет по открытому ключу проверить подпись и распарсить claims.

Открытые ключи чаще всего публикуются IdP через JWK-сет (`/.well-known/jwks.json`). `NimbusJwtDecoder` умеет качать и кешировать JWK-набор по `jwk-set-uri`, выбирать ключ по `kid` и валидировать алгоритм подписи.

Важно обеспечить **TLS и доверие** к JWK-эндпоинту. Если злоумышленник подменит JWKS, вы начнёте доверять чужим подписям. В проде используйте только HTTPS, пиннинг CA/сертификата прокси — плюс.

Ключи ротуются. IdP держит несколько ключей одновременно: старый и новый. Наличие `kid` в JWT позволяет выбрать корректный ключ из набора. Ваш ресурс-сервер должен корректно переживать ротацию без рестартов.

Альтернатива удалённому JWKS — локальная конфигурация публичного ключа (PEM). Это быстрее и независимее, но требует процесса доставки ключей и их ротации по DevOps-процедурам (например, через Vault/Config Server).

Поведение при недоступном JWKS должно быть «fail-closed»: если ключи не загрузились, **не** пускать трафик, а не принимать токены вслепую. Так вы избежите «окна» из-за сбоя сети.

В тестах удобно создавать `NimbusJwtDecoder.withPublicKey(...)` на лету и подписывать токены тестовым приватным ключом. Для юнитов — мокайте `JwtDecoder` и проверяйте downstream-логику.

Дополнительно ограничьте допустимые алгоритмы (например, только `RS256`/`ES256`) и запретите `alg=none`. Это настраивается в `NimbusJwtDecoder` и через валидаторы токена.

Наконец, не забывайте режим `STATELESS`: `sessionCreationPolicy(STATELESS)` и отключённый CSRF для API. Это избавит вас от лишних побочных эффектов, характерных для сессий.

**application.yml — указание JWK Set URI**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://idp.example.com/.well-known/jwks.json
```

**Java — минимальная конфигурация Resource Server с JWT**

```java
package com.example.jwt.rs;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class ResourceServerConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(cs -> cs.disable())
            .oauth2ResourceServer(o -> o.jwt());
        return http.build();
    }

    // Вариант с фиксированным JWK Set URI (можно опустить при использовании application.yml)
    @Bean
    JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.rs

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.oauth2.jwt.JwtDecoder
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder
import org.springframework.security.web.SecurityFilterChain

@Configuration
class ResourceServerConfig {

    @Bean
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests { it.anyRequest().authenticated() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .csrf { it.disable() }
            .oauth2ResourceServer { it.jwt() }
        return http.build()
    }

    @Bean
    fun jwtDecoder(): JwtDecoder =
        NimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build()
}
```

---

## Маппинг claims → authorities (scope/roles), префиксы и конвенции

JWT — это контейнер утверждений (claims). Имена и смысл claims зависят от IdP, но есть де-факто стандарты: `sub`, `iss`, `exp`, `scope` и в OIDC — `email`, `name`. Для авторизации нас интересуют «права»: `scope` и/или `roles`.

Spring Security преобразует claims в `GrantedAuthority`, после чего все механизмы (`hasRole`, `hasAuthority`) начинают работать как обычно. Ключ — **конвенции**: что считать «ролью», что — «скоупом», какие префиксы применять.

По умолчанию `JwtGrantedAuthoritiesConverter` превращает `scope`/`scp` в authorities с префиксом `SCOPE_`. Это полезно для «узких» прав уровня API: `SCOPE_orders.read`, `SCOPE_orders.write`.

Роли в IdP часто лежат в кастомном claim’е (`roles`, `realm_access.roles`, `resource_access.<client>.roles`). Их нужно явно взять и превратить в authorities с префиксом `ROLE_`, чтобы `hasRole("ADMIN")` работало консистентно.

Смешивать роли и скоупы в одну кучу — плохая идея. Держите префиксы (`ROLE_` и `SCOPE_`) раздельно и проверяйте их осознанно: роли — для coarse-grained доступа, скоупы — для «что именно в API разрешено».

Имена claims лучше нормализовать на своей стороне: не тянуть «всё подряд» из IdP, а выбирать подмножество и маппить в понятные вашему сервису authorities. Это снижает связанность и упрощает тестирование.

Мульти-клиентские IdP (Keycloak, Auth0) помещают роли по клиентам. Выберите соглашение: либо вытаскивать роли только для вашего `client_id`, либо добавлять префикс клиента в authority, чтобы избежать коллизий.

Кириллица/регистр в ролях — источник боли. Нормализуйте к верхнему регистру латиницей на стороне сервиса и держите единый стиль. Так вы избежите неожиданностей в проверках.

Если у вас есть доменные атрибуты (tenant, department), **не** превращайте их в роли. Лучше хранить их в `Authentication` как атрибуты (или в `JwtAuthenticationToken#getToken()`) и проверять в `AuthorizationManager`/`PermissionEvaluator`.

Логируйте результат маппинга на DEBUG в стейджинге: это помогает ловить рассинхронизацию между IdP и приложением, когда в токен пришла новая роль/скоуп.

Наконец, покрывайте конвертеры тестами: сделайте таблицу «входные claims → ожидаемые authorities». Это стабилизирует поведение при миграциях IdP.

**Java — конвертер claims → authorities c ROLE_/SCOPE_**

```java
package com.example.jwt.claims;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.*;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.*;
import java.util.*;
import java.util.stream.Collectors;

@Configuration
public class ClaimsMappingConfig {

    @Bean
    Converter<Jwt, ? extends AbstractAuthenticationToken> jwtAuthConverter() {
        var scopes = new JwtGrantedAuthoritiesConverter();
        scopes.setAuthorityPrefix("SCOPE_");

        return (Jwt jwt) -> {
            Set<GrantedAuthority> auths = new HashSet<>(Objects.requireNonNull(scopes.convert(jwt)));
            List<String> roles = Optional.ofNullable(jwt.getClaimAsStringList("roles")).orElse(List.of());
            auths.addAll(roles.stream()
                    .map(r -> "ROLE_" + r.toUpperCase(Locale.ROOT))
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toSet()));
            return new JwtAuthenticationToken(jwt, auths, jwt.getClaimAsString("sub"));
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.claims

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.convert.converter.Converter
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.server.resource.authentication.AbstractAuthenticationToken
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter
import java.util.Locale

@Configuration
class ClaimsMappingConfig {

    @Bean
    fun jwtAuthConverter(): Converter<Jwt, out AbstractAuthenticationToken> {
        val scopes = JwtGrantedAuthoritiesConverter().apply { setAuthorityPrefix("SCOPE_") }
        return Converter { jwt ->
            val auths = scopes.convert(jwt)!!.toMutableSet<GrantedAuthority>()
            jwt.getClaimAsStringList("roles")?.forEach { r ->
                auths.add(SimpleGrantedAuthority("ROLE_${r.uppercase(Locale.ROOT)}"))
            }
            JwtAuthenticationToken(jwt, auths, jwt.getClaimAsString("sub"))
        }
    }
}
```

---

## Валидация токенов: подпись, аудитории, срок, clock skew

Первая проверка — криптографическая подпись. Если подпись неверна, разговор закончен. Выбирайте устойчивые алгоритмы (`RS256`, `ES256`) и запрещайте неожиданные (`none`, неподходящие `HS*`).

Вторая — «кто выдал» (`iss`). Ресурс-сервер должен доверять ровно одному/нескольким известным издателям. Это отсекает токены соседних окружений или других IdP.

Третья — аудитория (`aud`). Токен должен быть предназначен именно вашему API (или набору). Это защищает от «переназначения» токена клиента А в клиент Б.

Четвёртая — время: `exp` (не просрочен), опционально `nbf` (не использовать до) и `iat` (для диагностики). В распределённых системах всегда есть дрейф часов, поэтому используйте «поблажку» (`clockSkew`), например 30–60 секунд.

Добавочные проверки — соответствие доменных claims (tenant, azp/client_id), требования к присутствию определённых скоупов/ролей. Их лучше оформлять отдельными валидаторами и включать в цепочку.

Не валидируйте «на глаз» JSON: всегда доверяйте только результату `JwtDecoder`, который уже проверил подпись и стандартные поля. Ручной парсинг без проверки подписи — типовая уязвимость.

При ошибках валидации возвращайте 401 с корректным `WWW-Authenticate: Bearer error="invalid_token", error_description="..."`. Это улучшает DX и соответствует RFC 6750.

Кэш JWK надо обновлять, но не слишком часто. Nimbus decoder делает это сам, подтягивая ключи по `kid` при встрече «незнакомого» ключа. Не отключайте этот механизм.

Если у вас сразу несколько issuers (multi-tenant IdP), используйте `JwtIssuerAuthenticationManagerResolver` — он подберёт нужный `JwtDecoder` по `iss`. Каждый decoder может иметь свою политику аудиторий и skew.

Тестируйте крайние случаи: истекший токен, неправильная аудитория, «будущий» токен, неверная подпись. Табличные тесты на негативы — обязательны.

**Java — валидаторы iss/aud + clock skew**

```java
package com.example.jwt.validate;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.core.*;
import org.springframework.security.oauth2.jwt.*;
import java.time.Duration;
import java.util.Set;

@Configuration
public class JwtValidationConfig {

    @Bean
    JwtDecoder jwtDecoderWithValidators() {
        NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build();

        OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer("https://idp.example.com/");
        OAuth2TokenValidator<Jwt> audience = jwt -> {
            Set<String> aud = Set.copyOf(jwt.getAudience());
            return aud.contains("orders-api")
                    ? OAuth2TokenValidatorResult.success()
                    : OAuth2TokenValidatorResult.failure(new OAuth2Error("invalid_token", "Wrong audience", null));
        };
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(withIssuer, audience, new JwtTimestampValidator(Duration.ofSeconds(60))));
        return decoder;
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.validate

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.oauth2.core.OAuth2Error
import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult
import org.springframework.security.oauth2.jwt.*
import java.time.Duration

@Configuration
class JwtValidationConfig {

    @Bean
    fun jwtDecoderWithValidators(): JwtDecoder {
        val decoder = NimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build()
        val issuer = JwtValidators.createDefaultWithIssuer("https://idp.example.com/")
        val audience = OAuth2TokenValidator<Jwt> { jwt ->
            if (jwt.audience.contains("orders-api")) OAuth2TokenValidatorResult.success()
            else OAuth2TokenValidatorResult.failure(OAuth2Error("invalid_token", "Wrong audience", null))
        }
        decoder.setJwtValidator(DelegatingOAuth2TokenValidator(issuer, audience, JwtTimestampValidator(Duration.ofSeconds(60))))
        return decoder
    }
}
```

---

## Refresh/rotation: где место refresh-токена и почему не в Resource Server

Refresh-токен — это **про клиента**. Его задача — получить **новый access-токен** у Authorization Server без повторного ввода пароля пользователем. Ресурс-сервер **не должен** принимать refresh-токены и «менять» их на доступ: это бы превратило его во второй Authorization Server.

Клиент хранит refresh-токен (аккуратно!) и при истечении access-токена делает `POST /oauth2/token` с `grant_type=refresh_token`. Ответ — новый access-токен (и, возможно, новый refresh). Ресурс-сервер об этом не знает и не должен.

Сроки жизни: access — короткий (минуты), refresh — длиннее (дни/часы) и может ротироваться (rotating refresh tokens). Если refresh украдут, злоумышленник сможет выпускать новые access-токены — поэтому хранение и выдача refresh’ей — зона повышенной ответственности клиента/IdP.

Ротация ключей подписи (JWKS) — другая тема. IdP может начать подписывать новые токены новым ключом, но некоторое время принимать и старые. Ресурс-сервер должен уметь валидировать оба (через JWK-набор).

Мобильные/SPA-клиенты не должны хранить refresh-токены в небезопасном месте. В браузере безопаснее использовать Authorization Code + PKCE и «серверный» обмен токена прокси-бэкендом, чтобы refresh жил на бэке, а не в JS.

Если вам нужен «тихий» продление сессии в браузере — используйте iframe/hidden fetch к IdP или backend-endpoint, который обновит access и вернёт фронту. Но не превращайте ресурс-сервер в этот endpoint.

Активация MFA часто «ломает» автоматический refresh: IdP может потребовать интеракцию. Готовьте UX и обработку ошибок, чтобы корректно отправлять пользователя на повторный login-поток.

Ротация refresh-токенов (rotation) уменьшает окно для повторного использования украденного токена: при каждом обмене старый становится недействительным. Клиенты обязаны использовать только «последний выданный».

Логи клиента должны бережно относиться к секретам: нельзя логировать refresh/access токены. Маскируйте или вовсе отключайте логи на уровне HTTP-клиента для чувствительных урлов.

Наконец, продумайте эвакуацию: как вы отзывать access/refresh токены при инциденте. На стороне Resource Server это может быть «чёрный список JTI» для access и конфигурация IdP — для refresh.

**Java — клиентский обмен refresh → access через WebClient**

```java
package com.example.jwt.refresh;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;

@Component
public class TokenClient {

    private final WebClient webClient = WebClient.builder().baseUrl("https://idp.example.com").build();

    public TokenResponse refresh(String clientId, String clientSecret, String refreshToken) {
        return webClient.post()
            .uri("/oauth2/token")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .bodyValue("grant_type=refresh_token&client_id=" + clientId + "&client_secret=" + clientSecret + "&refresh_token=" + refreshToken)
            .retrieve()
            .bodyToMono(TokenResponse.class)
            .block();
    }

    public record TokenResponse(String access_token, String refresh_token, long expires_in, String token_type) {}
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.refresh

import org.springframework.http.MediaType
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.WebClient

@Component
class TokenClient {

    private val webClient = WebClient.builder().baseUrl("https://idp.example.com").build()

    data class TokenResponse(
        val access_token: String,
        val refresh_token: String?,
        val expires_in: Long,
        val token_type: String
    )

    fun refresh(clientId: String, clientSecret: String, refreshToken: String): TokenResponse =
        webClient.post()
            .uri("/oauth2/token")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .bodyValue("grant_type=refresh_token&client_id=$clientId&client_secret=$clientSecret&refresh_token=$refreshToken")
            .retrieve()
            .bodyToMono(TokenResponse::class.java)
            .block()!!
}
```

---

## Ошибки и ответы: Bearer challenge, унификация 401/403, Problem Details

В stateless-API важно возвращать **правильные коды** и **заголовки**. Для анонимного запроса к защищённому ресурсу — `401 Unauthorized` с `WWW-Authenticate: Bearer`. Для аутентифицированного, но без прав — `403 Forbidden`.

RFC 6750 описывает формат «Bearer challenge»: можно добавить `realm`, `scope`, а для ошибок — `error`, `error_description`. Это помогает клиентам понять, что делать (получить токен, запросить нужный scope и т.п.).

Spring Security уже умеет это: `BearerTokenAuthenticationEntryPoint` для 401 и `BearerTokenAccessDeniedHandler` для 403. Их можно слегка настроить и дополнить возвратом Problem Details (RFC 7807) в теле.

Единый формат ошибок облегчает интеграции. В Boot 3 включите Problem Details глобально и верните лаконичное тело (`type`, `title`, `status`, `detail`, `instance`). Но не переусердствуйте с деталями: не раскрывайте политику и внутренние идентификаторы.

Журналы — это тоже часть контракта. Логируйте отказ по минимуму: путь, метод, subject/tenant, причина. Не печатайте токен/claims целиком. Чувствительные сведения маскируйте.

Для CORS убедитесь, что и на 401/403 вы отдаёте нужные `Access-Control-Allow-*`. Иначе браузер скроет тело/заголовки, и клиент «не увидит» причину отказа.

Ещё один нюанс — различать «нет токена» и «невалидный токен». Первая ситуация — обычный 401 без `error`, вторая — с `error="invalid_token"`. Это улучшает поведение SDK/клиентов.

При тестировании проверяйте: без токена — 401 с `WWW-Authenticate`, с «битым» токеном — 401 с `invalid_token`, без нужной роли — 403. Таблица таких проверок должна жить в CI.

Кастомные entry point/handler не должны «ломать» стандартные заголовки. Если вы отдаёте Problem Details, всё равно оставляйте `WWW-Authenticate` на 401.

Наконец, при частых 401 из-за истечения — проверьте клиент: корректно ли он обновляет токены, не «теряет» их при ретраях, не кэширует ли случайно старые значения.

**Java — настраиваемые обработчики 401/403 с Problem Details**

```java
package com.example.jwt.errors;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.ProblemDetail;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.server.resource.web.access.BearerTokenAccessDeniedHandler;
import org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationEntryPoint;
import org.springframework.security.web.SecurityFilterChain;

import java.io.IOException;

@Configuration
public class ErrorHandlingConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        var entryPoint = new BearerTokenAuthenticationEntryPoint();
        entryPoint.setRealmName("orders-api");

        var denied = new BearerTokenAccessDeniedHandler();

        http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> {}))
            .exceptionHandling(e -> e
                .authenticationEntryPoint((req, res, ex) -> {
                    entryPoint.commence(req, res, ex);
                    writeProblem(res, 401, "Unauthorized", "Bearer token required or invalid");
                })
                .accessDeniedHandler((req, res, ex) -> {
                    denied.handle(req, res, ex);
                    writeProblem(res, 403, "Forbidden", "Insufficient permissions");
                })
            );
        return http.build();
    }

    private void writeProblem(HttpServletResponse res, int status, String title, String detail) throws IOException {
        res.setContentType("application/problem+json");
        res.setStatus(status);
        var pd = ProblemDetail.forStatusAndDetail(status, detail);
        pd.setTitle(title);
        res.getWriter().write("{\"title\":\"" + title + "\",\"status\":" + status + ",\"detail\":\"" + detail + "\"}");
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.errors

import jakarta.servlet.http.HttpServletResponse
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.ProblemDetail
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.oauth2.server.resource.web.access.BearerTokenAccessDeniedHandler
import org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationEntryPoint
import org.springframework.security.web.SecurityFilterChain

@Configuration
class ErrorHandlingConfig {

    @Bean
    fun api(http: HttpSecurity): SecurityFilterChain {
        val entry = BearerTokenAuthenticationEntryPoint().apply { setRealmName("orders-api") }
        val denied = BearerTokenAccessDeniedHandler()

        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2ResourceServer { it.jwt { } }
            .exceptionHandling {
                it.authenticationEntryPoint { req, res, ex ->
                    entry.commence(req, res, ex)
                    writeProblem(res, 401, "Unauthorized", "Bearer token required or invalid")
                }.accessDeniedHandler { req, res, ex ->
                    denied.handle(req, res, ex)
                    writeProblem(res, 403, "Forbidden", "Insufficient permissions")
                }
            }
        return http.build()
    }

    private fun writeProblem(res: HttpServletResponse, status: Int, title: String, detail: String) {
        res.contentType = "application/problem+json"
        res.status = status
        val body = """{"title":"$title","status":$status,"detail":"$detail"}"""
        res.writer.write(body)
    }
}
```

---

## Token relay через Gateway, защита от повторного воспроизведения (jti)

Шлюз (API Gateway) часто выступает фронтом микросервисов. «Token relay» — шаблон, когда шлюз принимает пользовательский токен и **прокидывает** его дальше к бэкенд-сервисам, чтобы они выполняли свою локальную авторизацию.

Важно: relay не означает «доверяем шлюзу». Каждый целевой сервис остаётся ресурс-сервером и **сам** валидирует подпись/срок/аудиторию. Шлюз — удобство маршрутизации и кросс-срезы (rate limit, CORS), но не источник истины.

В Spring Cloud Gateway есть готовый фильтр `TokenRelay`, который достаёт access-токен из текущего аутентифицированного клиента (OAuth2 Client) и добавляет `Authorization: Bearer` в исходящий запрос. Это удобно для BFF-шаблонов.

Если шлюз не «логинит» (нет OAuth2 Client), можно просто **сохранить** исходный заголовок `Authorization` и пробросить его как есть. Но всё равно **не снимайте проверку** в целевом сервисе.

Повторное воспроизведение (replay) — риск для JWT: если злоумышленник украл access-токен, он может повторно отправлять его в окна жизни токена. Криптография подписи не спасает от «повторов».

Для защиты используйте `jti` (ID токена) и **реестр использованных идентификаторов** с TTL до `exp`. Ресурс-сервер при первом использовании `jti` запоминает его (Redis/Caffeine), а последующие обращения с тем же `jti` отвергает.

Балансируйте: jti-проверка — это stateful-вставка в stateless-архитектуру. Делайте её только на **критичных методах** (денежные операции), а не на каждом GET. На ровном месте можно перегрузить Redis.

Дополнительно для чувствительных действий используйте идемпотентные ключи на уровне бизнес-операций (Idempotency-Key) и antiforgery-механики. Они комплиментарны jti-защите.

На стороне шлюза комбинируйте token relay с rate limiting и аудитом. Вы видите общий периметр, там удобнее обнаруживать аномалии и отсекать мусор до JVM.

Не превращайте шлюз в «центральный авторизатор» с доменными правилами — это усложняет и дублирует логику. Пусть он делает периметр, а сервисы — контекстную авторизацию.

При истечении токена шлюз не должен сам «обновлять» его за счёт refresh, если это пользовательский токен браузера; это задача клиента/BFF. Иначе вы незаметно создадите stateful-зависимость и хранение секретов в шлюзе.

Наконец, тестируйте цепочку end-to-end: клиент → шлюз → сервис с валидным/просроченным токеном, с повтором того же `jti`. Автотесты поймают регрессии при обновлениях Spring Cloud.

**Java — Spring Cloud Gateway с TokenRelay (маршрутизация)**

```java
package com.example.jwt.gateway;

import org.springframework.cloud.gateway.route.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayConfig {

    @Bean
    RouteLocator routes(RouteLocatorBuilder b) {
        return b.routes()
            .route("orders-api", r -> r.path("/api/orders/**")
                .filters(f -> f.tokenRelay()) // требует spring-boot-starter-oauth2-client
                .uri("http://orders-service:8080"))
            .build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.gateway

import org.springframework.cloud.gateway.route.RouteLocator
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class GatewayConfig {

    @Bean
    fun routes(b: RouteLocatorBuilder): RouteLocator =
        b.routes()
            .route("orders-api") { r ->
                r.path("/api/orders/**")
                    .filters { f -> f.tokenRelay() }
                    .uri("http://orders-service:8080")
            }
            .build()
}
```

**Java — фильтр защиты от replay на основе `jti` (демо, in-memory)**

```java
package com.example.jwt.replay;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.*;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class JtiReplayFilter extends OncePerRequestFilter {

    private final Map<String, Long> seen = new ConcurrentHashMap<>();

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        Object auth = req.getUserPrincipal();
        if (auth instanceof JwtAuthenticationToken token) {
            String jti = token.getToken().getId(); // claim: jti
            Long exp = token.getToken().getExpiresAt() != null ? token.getToken().getExpiresAt().toEpochMilli() : null;
            if (jti != null && exp != null) {
                Long now = Instant.now().toEpochMilli();
                seen.entrySet().removeIf(e -> e.getValue() < now); // простая очистка
                Long prev = seen.putIfAbsent(jti, exp);
                if (prev != null) {
                    res.setStatus(401);
                    res.getWriter().write("invalid_token: replay detected");
                    return;
                }
            }
        }
        chain.doFilter(req, res);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.replay

import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import java.io.IOException
import java.time.Instant
import java.util.concurrent.ConcurrentHashMap

@Component
class JtiReplayFilter : OncePerRequestFilter() {

    private val seen = ConcurrentHashMap<String, Long>()

    @Throws(ServletException::class, IOException::class)
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        val auth = req.userPrincipal
        if (auth is JwtAuthenticationToken) {
            val jti = auth.token.id
            val exp = auth.token.expiresAt?.toEpochMilli()
            if (jti != null && exp != null) {
                val now = Instant.now().toEpochMilli()
                seen.entries.removeIf { it.value < now }
                val prev = seen.putIfAbsent(jti, exp)
                if (prev != null) {
                    res.status = 401
                    res.writer.write("invalid_token: replay detected")
                    return
                }
            }
        }
        chain.doFilter(req, res)
    }
}
```

> В проде храните `jti` в Redis c TTL до `exp` и добавляйте проверку **только** на чувствительных POST/PUT. Это компромисс между безопасностью и стоимостью stateful-проверок.

# 7. OAuth2/OIDC и внешние провайдеры

**Gradle (Groovy)**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf' // для простого /login
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## OAuth2 Client: авторизация через сторонние IdP (Keycloak, Auth0, Azure AD)

OAuth2 Client в Spring Security — это клиентская сторона потоков авторизации. В MVC-приложении чаще всего используется `oauth2Login()`: фреймворк перенаправляет пользователя в внешний IdP (Identity Provider), получает авторизационный код, обменивает его на токены и создаёт `SecurityContext` с `OAuth2User`/`OidcUser`. Это заменяет собственный логин/пароль и экономит вам реализацию аутентификации и хранение паролей.

Практически каждый вендор (Keycloak, Auth0, Azure AD, Okta и т.п.) предоставляет «кнопку входа» по одному и тому же стандарту Authorization Code (для OIDC — «сверху» к нему добавляется id_token). Разница между провайдерами главным образом в адресах эндпоинтов и структуре claims, а также в том, как выдаются роли/группы.

В Spring Boot конфигурация клиента декларативна: в `application.yml` описываем `spring.security.oauth2.client.registration.*` (регистрация клиента: client-id/secret, scopes) и `provider.*` (эндпоинты `authorization-uri`, `token-uri`, `jwk-set-uri`, `userinfo-uri`). Для OIDC многие поля подтягиваются с discovery документа (`/.well-known/openid-configuration`).

При успешном входе Spring создаёт `OAuth2AuthenticationToken` (или `OidcUser` для OIDC), который доступен в контроллерах через `@AuthenticationPrincipal`. Там же можно прочитать атрибуты профиля — email, name, groups — и сопоставить их внутриролевой модели приложения.

Если вы рендерите собственную страницу логина, достаточно на ней разместить ссылки на `/oauth2/authorization/{registrationId}`. `{registrationId}` — это имя регистрации клиента (например, `keycloak`, `azure`, `auth0`), указанное в конфигурации. Spring сам сформирует правильный redirect на провайдера с нужным `state`/`nonce`.

Безопасность этого потока основана на редиректах и cookie сессии. Для MVC обязательно держите включённый CSRF и строгие флаги cookie (`Secure`, `HttpOnly`, `SameSite=Lax/None`). Для SPA лучше использовать BFF-подход (backend-for-frontend), чтобы не хранить чувствительные токены в браузере.

Маппинг ролей/атрибутов — чувствительный момент. Разные IdP по-разному кладут группы и роли (например, `realm_access.roles` в Keycloak и `roles`/`groups` в Azure AD). Рекомендуется централизовать конверсию в один бин `OidcUserService`/`OAuth2UserService`, который превратит claims в `GrantedAuthority` вашего приложения.

Не переносите «как есть» внешние роли в вашу систему прав. Лучше ввести собственные роли/authorities и написать явный маппинг «внешнее → внутреннее». Это снижает связанность и упрощает аудит, особенно когда у вас несколько провайдеров.

Если провайдеры разные для разных сред (dev/stage/prod), разделяйте конфигурацию по профилям Spring и держите отдельные clients/realms/tenants. Это упрощает отладку и не даёт тестовым секретам попасть в прод.

Отдельно подумайте об обработке ошибок: отказ пользователя, истёкшее состояние, неверная конфигурация redirect URI. Все эти сценарии должны приводить к понятным страницам/ошибкам, а не к белым screens of death. В Spring это настраивается через `failureUrl`/`AuthenticationFailureHandler`.

И наконец, не храните `client-secret` в `application.yml` в открытом виде. В проде секреты — только из Vault/KMS/переменных окружения. В репозитории — максимум шаблон значений.

**application.yml — регистрации для Keycloak и Azure AD (пример)**

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: app-frontend
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            scope: openid,profile,email
            provider: keycloak
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
          azure:
            client-id: ${AZURE_CLIENT_ID}
            client-secret: ${AZURE_CLIENT_SECRET}
            scope: openid,profile,email
            provider: azure
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          keycloak:
            issuer-uri: https://keycloak.example.com/realms/demo
          azure:
            issuer-uri: https://login.microsoftonline.com/${AZURE_TENANT_ID}/v2.0
```

**Java — минимальная конфигурация `oauth2Login()` + получение профиля**

```java
package com.example.oidc.login;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

import java.util.Map;

@Configuration
public class OAuth2LoginSecurity {

    @Bean
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/", "/login", "/css/**").permitAll()
                .anyRequest().authenticated())
            .oauth2Login(o -> o.loginPage("/login"))
            .logout(l -> l.logoutSuccessUrl("/"));
        return http.build();
    }
}

@Controller
class MeController {

    @GetMapping("/me")
    public Map<String, Object> me(@AuthenticationPrincipal OidcUser user) {
        return Map.of(
            "name", user.getFullName(),
            "email", user.getEmail(),
            "claims", user.getClaims()
        );
    }

    @GetMapping("/login")
    public String loginPage() { return "login"; } // thymeleaf: ссылки на /oauth2/authorization/keycloak и /oauth2/authorization/azure
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.oidc.login

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.security.oauth2.core.oidc.user.OidcUser
import org.springframework.security.web.SecurityFilterChain
import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.GetMapping

@Configuration
class OAuth2LoginSecurity {

    @Bean
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/", "/login", "/css/**").permitAll()
                    .anyRequest().authenticated()
            }
            .oauth2Login { it.loginPage("/login") }
            .logout { it.logoutSuccessUrl("/") }
        return http.build()
    }
}

@Controller
class MeController {

    @GetMapping("/me")
    fun me(@AuthenticationPrincipal user: OidcUser) =
        mapOf("name" to user.fullName, "email" to user.email, "claims" to user.claims)

    @GetMapping("/login")
    fun loginPage(): String = "login"
}
```

---

## Потоки: Authorization Code (PKCE), Client Credentials, Device Code — когда какие

Authorization Code + PKCE — стандарт для интерактивных пользователей (браузер/мобилка). В MVC Spring всё делает за вас через `oauth2Login()`. В SPA/мобилках PKCE защищает от утечек секретов: клиент доказывает, что именно он начал поток, предъявляя `code_verifier` при обмене кода на токен.

Client Credentials — это машина-к-машине («машинный» клиент без пользователя). Он получает access-токен, представляя **себя** (client_id/secret). Такой токен не несёт пользователя, только «клиента», и годится для фоновых задач, интеграций, вызовов «внутренних» API. На уровне прав стоит отделять такие токены от «пользовательских» (другие `audience/scope`).

Device Code — для устройств без удобного ввода (TV, CLI). Пользователь авторизуется на втором устройстве, а «немой» клиент получает токен, периодически опрашивая токен-эндпоинт. Это тоже про «пользовательский» контекст, но без браузера прямо на устройстве.

Когда выбирать: для веб-кабинетов и SSO между системами — Authorization Code/OIDC. Для бэкендов/интеграций — Client Credentials. Для CLI/TV — Device Code. Важно не смешивать: токен, полученный по CC, не должен использоваться как «пользовательский» в UI, иначе вы потеряете аудит и рискуете широкими привилегиями.

В Spring «автоматический» клиент — это `OAuth2AuthorizedClientManager` с нужными провайдерами. Он инкапсулирует хранение/обновление токенов и интегрируется с `WebClient` через `ServletOAuth2AuthorizedClientExchangeFilterFunction`.

Для PKCE в MVC ничего отдельно делать не требуется — Spring Security сам сформирует `code_challenge`/`code_verifier` и завершит обмен. В SPA-архитектуре PKCE реализуют на стороне фронта или BFF-бэкенда (рекомендуется BFF, чтобы не отдавать refresh в браузер).

Device Code в Spring можно реализовать вручную через `WebClient`: запросить `device_code`/`user_code`, показать пользователю URL подтверждения, затем по таймеру поллить `token` до успеха или истечения. Для CLI-утилит это удобно и не требует браузера.

Контролируйте scopes. Для CC обычно достаточно «узких» скоупов (например, `service.read`/`service.write`). Для AC/OIDC следите, чтобы фронт не запрашивал лишних разрешений, и не принимайте «широкие» токены в сервисах без необходимости.

Квоты и rate limits различаются: CC-поток может создавать «бурсты» запросов к токен-эндпоинту. Старайтесь кэшировать токены до истечения и не плодить лишних клиентов. Для Device Code уважайте интервалы опроса (RFC).

Наконец, все три потока должны быть документированы в ваших интеграционных гайдлайнах: «какой поток где», «какие scopes», «какие аудитории». Это снижает риск неправильного применения и инцидентов.

**Java — Client Credentials через `OAuth2AuthorizedClientManager` и `WebClient`**

```java
package com.example.oauth2.cc;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.client.*;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.oauth2.client.web.*;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class ClientCredentialsConfig {

    @Bean
    OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository repo,
            OAuth2AuthorizedClientService service) {

        var provider = OAuth2AuthorizedClientProviderBuilder.builder()
                .clientCredentials()
                .build();

        var manager = new DefaultOAuth2AuthorizedClientManager(repo, service);
        manager.setAuthorizedClientProvider(provider);
        return manager;
    }

    @Bean
    WebClient apiClient(OAuth2AuthorizedClientManager manager) {
        var oauth2 = new ServletOAuth2AuthorizedClientExchangeFilterFunction(manager);
        oauth2.setDefaultClientRegistrationId("orders-cc"); // registration.orders-cc
        return WebClient.builder().apply(oauth2.oauth2Configuration()).build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.oauth2.cc

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.oauth2.client.*
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository
import org.springframework.security.oauth2.client.web.ServletOAuth2AuthorizedClientExchangeFilterFunction
import org.springframework.web.reactive.function.client.WebClient

@Configuration
class ClientCredentialsConfig {

    @Bean
    fun authorizedClientManager(
        repo: ClientRegistrationRepository,
        service: OAuth2AuthorizedClientService
    ): OAuth2AuthorizedClientManager {
        val provider = OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()
            .build()
        return DefaultOAuth2AuthorizedClientManager(repo, service).apply {
            authorizedClientProvider = provider
        }
    }

    @Bean
    fun apiClient(manager: OAuth2AuthorizedClientManager): WebClient {
        val oauth2 = ServletOAuth2AuthorizedClientExchangeFilterFunction(manager)
        oauth2.setDefaultClientRegistrationId("orders-cc")
        return WebClient.builder().apply(oauth2.oauth2Configuration()).build()
    }
}
```

**Java — Device Code (ручная реализация через WebClient)**

```java
package com.example.oauth2.device;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Map;

@Component
public class DeviceCodeClient {

    private final WebClient idp = WebClient.builder().baseUrl("https://idp.example.com").build();

    public Map<String, Object> start(String clientId, String scope) {
        return idp.post().uri("/oauth2/device_authorization")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .bodyValue("client_id=" + clientId + "&scope=" + scope)
            .retrieve().bodyToMono(Map.class).block();
    }

    public Map<String, Object> pollToken(String clientId, String deviceCode, int intervalSec) {
        while (true) {
            var resp = idp.post().uri("/oauth2/token")
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .bodyValue("grant_type=urn:ietf:params:oauth:grant-type:device_code&client_id=" + clientId + "&device_code=" + deviceCode)
                .retrieve().bodyToMono(Map.class).onErrorResume(e -> Mono.empty()).block();
            if (resp != null && resp.get("access_token") != null) return resp;
            try { Thread.sleep(Duration.ofSeconds(intervalSec).toMillis()); } catch (InterruptedException ignored) {}
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.oauth2.device

import org.springframework.http.MediaType
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.WebClient
import java.time.Duration

@Component
class DeviceCodeClient {

    private val idp = WebClient.builder().baseUrl("https://idp.example.com").build()

    fun start(clientId: String, scope: String): Map<String, Any> =
        idp.post().uri("/oauth2/device_authorization")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .bodyValue("client_id=$clientId&scope=$scope")
            .retrieve().bodyToMono(Map::class.java).block() as Map<String, Any>

    fun pollToken(clientId: String, deviceCode: String, intervalSec: Int): Map<String, Any> {
        while (true) {
            val resp = idp.post().uri("/oauth2/token")
                .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .bodyValue("grant_type=urn:ietf:params:oauth:grant-type:device_code&client_id=$clientId&device_code=$deviceCode")
                .retrieve().bodyToMono(Map::class.java).onErrorResume { _ -> reactor.core.publisher.Mono.empty() }.block()
            if (resp != null && resp["access_token"] != null) return resp as Map<String, Any>
            Thread.sleep(Duration.ofSeconds(intervalSec.toLong()).toMillis())
        }
    }
}
```

---

## OIDC надстройка: id_token, userinfo, claims и сессии в браузере

OpenID Connect — это идентификационный слой поверх OAuth2. Ключевое отличие: клиент получает **id_token** (JWT) с информацией о пользователе, подписанную IdP. Этот токен предназначен для клиента (вашего приложения), а не для ресурс-серверов. Он помогает установить локальную сессию и заполнить профиль пользователя.

Помимо id_token, OIDC определяет `userinfo` — эндпоинт, с которого можно получить дополнительные атрибуты профиля по access-токену. Весомая часть интеграций использует оба: id_token для первичной идентификации, userinfo — для обогащения (name, locale, picture и т.д.).

В Spring при `oauth2Login()` вы получаете `OidcUser`, уже собранного `OidcUserService`: он объединяет claims из id_token и userinfo (если сконфигурирован провайдер). В контроллере вы видите «сшитую» картину и можете работать с ней как с единой моделью.

Хорошая практика — **нормализовать** атрибуты. Разные IdP имеют разные поля (`preferred_username`/`upn`/`email`). Создайте адаптер, который из `OidcUser` собирает ваш «профиль» с едиными полями (username, email, displayName) и сохраняет его в доменную модель, не протаскивая IdP-специфику повсюду.

Роли/группы в OIDC — не стандарт, а расширение. В Keycloak часто используют `realm_access.roles` или `resource_access.<client>.roles`. В Azure AD — `roles`/`groups` при соответствующей настройке приложения. Эти поля нужно превращать в `GrantedAuthority` с префиксами (`ROLE_...`) по вашей конвенции.

При первой интеграции часто хочется «скопировать всё, что пришло в id_token». Так делать не стоит. Храните в своей базе **минимум** (user id, email, displayName, audit поля) и обновляйте их при каждом входе. Полный набор claims легко меняется на стороне IdP и приведёт к миграциям схемы без необходимости.

Сессии в MVC — cookie-базирующиеся. После успешного OIDC-входа Spring создаёт сессию и наполняет `SecurityContext`. Обязательно держите CSRF включённым и настроенные security headers: XSS + доступ к cookie могут украсть сессию. CSP и HttpOnly — must-have.

Если у вас SPA, не прокидывайте id_token/access-токен напрямую в JS, если можно избежать. BFF-шаблон позволяет хранить токены на бэкенде и проксировать API-вызовы, тем самым снижая риск утечки.

Обратите внимание на clock skew: поля `auth_time`, `iat`, `exp` должны проверяться с некоторым допуском времени. В противном случае пользователи с «неправильными» часами на устройстве периодически будут «вылетать».

Добавляйте тесты, проверяющие присутствие критичных claims (email/verified и т.п.). Если провайдер внезапно исключит их из id_token (меняли политику), вы поймаете регрессию раньше релиза.

**Java — нормализация ролей из OIDC claims через кастомный `OidcUserService`**

```java
package com.example.oidc.mapping;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.client.oidc.userinfo.*;
import org.springframework.security.oauth2.core.oidc.user.*;
import org.springframework.security.web.SecurityFilterChain;

import java.util.*;
import java.util.stream.Collectors;

@Configuration
public class OidcMappingConfig {

    @Bean
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2Login(o -> o.userInfoEndpoint(u -> u.oidcUserService(oidcUserService())));
        return http.build();
    }

    @Bean
    public OidcUserService oidcUserService() {
        var delegate = new OidcUserService();
        return new OidcUserService() {
            @Override
            public OidcUser loadUser(OidcUserRequest userRequest) {
                OidcUser user = delegate.loadUser(userRequest);
                Set<String> roles = new HashSet<>();
                Map<String, Object> claims = user.getClaims();

                Map<String, Object> realmAccess = (Map<String, Object>) claims.get("realm_access");
                if (realmAccess != null) {
                    roles.addAll((Collection<String>) realmAccess.getOrDefault("roles", List.of()));
                }
                List<String> azureRoles = (List<String>) claims.getOrDefault("roles", List.of());
                roles.addAll(azureRoles);

                Collection<GrantedAuthority> auth = roles.stream()
                        .map(r -> "ROLE_" + r.toUpperCase(Locale.ROOT))
                        .map(SimpleGrantedAuthority::new)
                        .collect(Collectors.toSet());

                return new DefaultOidcUser(auth, user.getIdToken(), user.getUserInfo(), "preferred_username");
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.oidc.mapping

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.client.oidc.userinfo.OidcUserRequest
import org.springframework.security.oauth2.client.oidc.userinfo.OidcUserService
import org.springframework.security.oauth2.core.oidc.user.DefaultOidcUser
import org.springframework.security.oauth2.core.oidc.user.OidcUser
import org.springframework.security.web.SecurityFilterChain
import java.util.Locale

@Configuration
class OidcMappingConfig {

    @Bean
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2Login { o -> o.userInfoEndpoint { u -> u.oidcUserService(oidcUserService()) } }
        return http.build()
    }

    @Bean
    fun oidcUserService(): OidcUserService {
        val delegate = OidcUserService()
        return object : OidcUserService() {
            override fun loadUser(userRequest: OidcUserRequest): OidcUser {
                val user = delegate.loadUser(userRequest)
                val claims = user.claims

                val roles = mutableSetOf<String>()
                val realmAccess = claims["realm_access"] as? Map<*, *>
                val kcRoles = (realmAccess?.get("roles") as? Collection<*>)?.filterIsInstance<String>() ?: emptyList()
                roles.addAll(kcRoles)
                val azureRoles = (claims["roles"] as? Collection<*>)?.filterIsInstance<String>() ?: emptyList()
                roles.addAll(azureRoles)

                val auth = roles.map { SimpleGrantedAuthority("ROLE_${it.uppercase(Locale.ROOT)}") }.toSet()
                return DefaultOidcUser(auth, user.idToken, user.userInfo, "preferred_username")
            }
        }
    }
}
```

---

## Single Sign-On/Logout, state/nonce, защита редиректов

SSO (Single Sign-On) позволяет пользователю войти один раз в IdP и быть аутентифицированным в нескольких приложениях. В браузере это обычно cookie-сессия у IdP и локальные сессии у приложений, которые «вырастают» после успешного потока Authorization Code.

Параметр `state` защищает от CSRF-атак на сам процесс авторизации: приложение кладёт непредсказуемое значение, а IdP возвращает его обратно. Spring Security делает это автоматически — хранит в сессии и проверяет соответствие. Никогда не отключайте это поведение.

В OIDC добавляется `nonce` — защита от повторного воспроизведения id_token. Клиент посылает `nonce` в auth-запросе, а затем проверяет, что этот же `nonce` вернулся в id_token. Это предотвращает подмену «старым» id_token с другой сессии.

Logout бывает локальный (разрыв сессии в приложении) и глобальный (Single Logout). Для OIDC глобальный выход реализуется через `end_session_endpoint`. В Spring есть `OidcClientInitiatedLogoutSuccessHandler`, который посылает пользователя на IdP с `id_token_hint` и `post_logout_redirect_uri`.

Важно защитить финальный редирект после логина/логаута. Никогда не принимайте произвольный `redirect` из запроса без валидации — это открытые редиректы. Используйте белый список доменов/путей, либо полностью игнорируйте внешние указатели и ведите на фиксированную страницу.

С точки зрения UX для SSO-набора систем полезно унифицировать страницу выбора IdP/логина и иметь общий «портал», который запускает поток Auth Code для нужного приложения. Тогда state/nonce контролируются централизованно, а приложения получают ровно то, что им нужно.

При ошибках в обмене кодов возвращайте понятные экраны, а в логах сохраняйте корреляцию (trace-id, state) для расследований. Не логируйте коды/токены.

Проверяйте «глубокие» ссылки: пользователь должен возвращаться туда, откуда начинал, но только если URL в белом списке. Для этого применяйте собственный `AuthenticationSuccessHandler` с валидацией, а не бесконтрольный `?continue=`.

Глобальный logout в корпоративных средах часто требует «хукнуть» несколько IdP/приложений. Проверьте поддерживаемые режимы у вашего IdP (front-channel vs back-channel) и протестируйте «параллельные» выходы, чтобы не оставались «висячие» сессии.

Тщательно тестируйте state/nonce — именно их ошибки чаще всего приводят к загадочным 401/400 в проде при скачках clock skew или при некорректных настройках прокси.

**Java — OIDC Single Logout и валидация «куда редиректить»**

```java
package com.example.oidc.logout;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.client.oidc.web.logout.OidcClientInitiatedLogoutSuccessHandler;
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;

import java.net.URI;
import java.util.Set;

@Configuration
public class OidcLogoutConfig {

    private static final Set<String> ALLOWED_REDIRECTS = Set.of("/", "/dashboard");

    @Bean
    SecurityFilterChain web(HttpSecurity http, ClientRegistrationRepository repos) throws Exception {
        var logoutHandler = new OidcClientInitiatedLogoutSuccessHandler(repos);
        logoutHandler.setPostLogoutRedirectUri("{baseUrl}/");

        AuthenticationSuccessHandler success = (request, response, authentication) -> {
            String target = request.getParameter("continue");
            if (target == null || !ALLOWED_REDIRECTS.contains(URI.create(target).getPath())) {
                new SavedRequestAwareAuthenticationSuccessHandler().onAuthenticationSuccess(request, response, authentication);
            } else {
                response.sendRedirect(target);
            }
        };

        http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2Login(o -> o.successHandler(success))
            .logout(l -> l.logoutSuccessHandler(logoutHandler));
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.oidc.logout

import jakarta.servlet.http.HttpServletRequest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.oauth2.client.oidc.web.logout.OidcClientInitiatedLogoutSuccessHandler
import org.springframework.security.oauth2.client.registration.ClientRegistrationRepository
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.AuthenticationSuccessHandler
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler
import java.net.URI

@Configuration
class OidcLogoutConfig {

    private val allowedRedirects = setOf("/", "/dashboard")

    @Bean
    fun web(http: HttpSecurity, repos: ClientRegistrationRepository): SecurityFilterChain {
        val logoutHandler = OidcClientInitiatedLogoutSuccessHandler(repos).apply {
            setPostLogoutRedirectUri("{baseUrl}/")
        }

        val success = AuthenticationSuccessHandler { request, response, auth ->
            val target = request.getParameter("continue")
            if (target == null || !allowedRedirects.contains(URI.create(target).path)) {
                SavedRequestAwareAuthenticationSuccessHandler().onAuthenticationSuccess(request, response, auth)
            } else {
                response.sendRedirect(target)
            }
        }

        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2Login { it.successHandler(success) }
            .logout { it.logoutSuccessHandler(logoutHandler) }
        return http.build()
    }
}
```

---

## Мульти-IdP и настройка маппинга атрибутов в роли системы

Многие приложения должны принимать вход из нескольких источников — корпоративный Azure AD, партнёрский Keycloak, социальные провайдеры. В Spring это тривиально: вы описываете несколько `registration.*` и `provider.*`, а на странице логина показываете набор ссылок на `/oauth2/authorization/{registrationId}`.

Однако «мульти» — это не только вход. Разные IdP возвращают разные claims и «семантики» ролей. Вам нужен единый слой маппинга в «внутренние» роли/права. Самый удобный способ — кастомный `OAuth2UserService`/`OidcUserService`, который по `registrationId` решает, из каких claims извлекать роли, нормализует их и возвращает `GrantedAuthority`.

Если часть интеграций — не OIDC, а «голый» OAuth2 (без id_token), вы получите `DefaultOAuth2User` вместо `OidcUser`. Поддержите оба случая: делайте общий интерфейс для извлечения атрибутов и ветвление на основе типа пользователя.

Не увлекайтесь «зашиванием» маппинга в код. Хорошая практика — хранить таблицу соответствий (provider → externalRole → internalAuthority) в БД/конфиге, а в коде держать кэш и аккуратный слой трансформации. Это позволит менять политику без релиза.

Белый список провайдеров на страницу логина — ещё один контроль: если временно отключаете IdP, просто скрывайте ссылку/отклоняйте попытки входа по нему. В коде это можно реализовать через `AuthenticationFailureHandler` с понятным сообщением.

Пользовательские имена конфликтуют между IdP. Используйте стабильный «внутренний» ключ (например, `iss|sub` или `provider:subject`) и храните соответствие в своей БД. Не пытайтесь «угадать» уникальность по email — он может меняться, и не всегда уникален.

Для аналитики полезно добавлять поле «источник аутентификации» в профиль пользователя и в аудит. Это поможет оценивать трафик по провайдерам и отслеживать проблемы в конкретных интеграциях.

Отдельно продумайте fallback-сценарии: если IdP недоступен, что видит пользователь? У ссылки должна быть дружественная ошибка, а приложение — не «падать» на пустом месте.

И наконец, имейте «песочницу» для каждого IdP: отдельный тестовый tenant/realm, тестовые клиенты/пользователи. Это позволит обновлять настройки IdP без риска для прод-пользователей.

**application.yml — два провайдера в одном приложении**

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: app-frontend
            client-secret: ${KEYCLOAK_SECRET}
            scope: openid,profile,email
            provider: keycloak
          azure:
            client-id: ${AZURE_CLIENT_ID}
            client-secret: ${AZURE_CLIENT_SECRET}
            scope: openid,profile,email
            provider: azure
        provider:
          keycloak:
            issuer-uri: https://keycloak.example.com/realms/demo
          azure:
            issuer-uri: https://login.microsoftonline.com/${AZURE_TENANT_ID}/v2.0
```

**Java — общий `OidcUserService` с разным извлечением ролей по `registrationId`**

```java
package com.example.multiidp;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.client.oidc.userinfo.OidcUserRequest;
import org.springframework.security.oauth2.client.oidc.userinfo.OidcUserService;
import org.springframework.security.oauth2.core.oidc.user.DefaultOidcUser;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.web.SecurityFilterChain;

import java.util.*;
import java.util.stream.Collectors;

@Configuration
public class MultiIdpConfig {

    @Bean
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2Login(o -> o.userInfoEndpoint(u -> u.oidcUserService(oidcUserService())));
        return http.build();
    }

    @Bean
    OidcUserService oidcUserService() {
        var delegate = new OidcUserService();
        return new OidcUserService() {
            @Override
            public OidcUser loadUser(OidcUserRequest req) {
                OidcUser base = delegate.loadUser(req);
                String idp = req.getClientRegistration().getRegistrationId();
                Set<String> roles = new HashSet<>();
                Map<String, Object> c = base.getClaims();

                if ("keycloak".equals(idp)) {
                    Map<String, Object> ra = (Map<String, Object>) c.get("realm_access");
                    if (ra != null) roles.addAll((Collection<String>) ra.getOrDefault("roles", List.of()));
                } else if ("azure".equals(idp)) {
                    roles.addAll((Collection<String>) c.getOrDefault("roles", List.of()));
                }

                var auth = roles.stream().map(r -> "ROLE_"+r.toUpperCase(Locale.ROOT))
                        .map(SimpleGrantedAuthority::new).collect(Collectors.toSet());
                return new DefaultOidcUser(auth, base.getIdToken(), base.getUserInfo(), "preferred_username");
            }
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.multiidp

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.client.oidc.userinfo.OidcUserRequest
import org.springframework.security.oauth2.client.oidc.userinfo.OidcUserService
import org.springframework.security.oauth2.core.oidc.user.DefaultOidcUser
import org.springframework.security.oauth2.core.oidc.user.OidcUser
import org.springframework.security.web.SecurityFilterChain
import java.util.Locale

@Configuration
class MultiIdpConfig {

    @Bean
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests { it.anyRequest().authenticated() }
            .oauth2Login { o -> o.userInfoEndpoint { u -> u.oidcUserService(oidcUserService()) } }
        return http.build()
    }

    @Bean
    fun oidcUserService(): OidcUserService {
        val delegate = OidcUserService()
        return object : OidcUserService() {
            override fun loadUser(req: OidcUserRequest): OidcUser {
                val base = delegate.loadUser(req)
                val idp = req.clientRegistration.registrationId
                val claims = base.claims
                val roles = mutableSetOf<String>()

                when (idp) {
                    "keycloak" -> {
                        val ra = claims["realm_access"] as? Map<*, *>
                        roles += (ra?.get("roles") as? Collection<*>)?.filterIsInstance<String>() ?: emptyList()
                    }
                    "azure" -> {
                        roles += (claims["roles"] as? Collection<*>)?.filterIsInstance<String>() ?: emptyList()
                    }
                }

                val auth = roles.map { SimpleGrantedAuthority("ROLE_${it.uppercase(Locale.ROOT)}") }.toSet()
                return DefaultOidcUser(auth, base.idToken, base.userInfo, "preferred_username")
            }
        }
    }
}
```

---

## Тестирование входа через OAuth2: mock OAuth2Login, Stub IdP

Тесты безопасности должны покрывать «счастливые» и «негативные» сценарии входа. Библиотека `spring-security-test` даёт удобные постпроцессоры `oauth2Login()` и `oidcLogin()` для `MockMvc`: вы можете «подложить» аутентифицированного пользователя без реального редиректа и обмена кодов.

С `oauth2Login()` вы моделируете «чистый OAuth2» (без id_token). Это полезно, если ваш провайдер не OIDC. С `oidcLogin()` добавляется `OidcIdToken` и, при необходимости, `OidcUserInfo`. Вы контролируете claims и тем самым тестируете свой маппинг ролей.

Пишите тесты на URL-политику: защищённый путь без аутентификации → 302 на `/login` (или 401 для API), с аутентификацией → 200, без нужной роли → 403. Это фиксирует контракт и предотвращает регрессии при изменении DSL.

Для интеграционных тестов IdP хорошо подходит Testcontainers Keycloak или локальный Stub. С Keycloak можно импортировать realm и клиентов, запускать его в CI и проходить полный поток Authorization Code. Это медленнее, но даёт уверенность в «боевой» конфигурации.

Если поднимаете Stub IdP сами (для unit-уровня), достаточно замокать discovery и токен-эндпоинт. Но обычно проще использовать `spring-security-test` — он быстро и надёжно закрывает большую часть сценариев.

Не забывайте тестировать logout: после обращения к `/logout` сессия должна исчезать, а защищённые страницы — отдавать 302/401. Для OIDC-SLO мокните вызов `end_session_endpoint` (или проверьте, что формируется корректный URL).

Для BFF-схем важно протестировать, что backend проксирует запросы с пользовательским контекстом и правильно обрабатывает истечение токена (например, 302 на логин при 401 от ресурс-сервера). Это можно сделать на уровне MockMvc, подменив ответ downstream сервиса.

Тесты не должны раскрывать секреты. Никогда не кладите реальные client-secret в фикстуры. Для Keycloak в Testcontainers используйте случайные секреты и динамически сгенерированные порты.

Отдельный набор тестов — на маппинг claims: «входящие» роли/группы → «внутренние» authorities. Табличные тесты тут очень удобны: меняете конфиг маппинга — тесты покажут, что поменялось.

**Java — MockMvc с `oidcLogin()` и проверка доступа**

```java
package com.example.oidc.tests;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.oauth2.core.oidc.OidcIdToken;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;
import org.springframework.test.web.servlet.MockMvc;

import java.time.Instant;
import java.util.Map;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import org.springframework.beans.factory.annotation.Autowired;

@SpringBootTest
@AutoConfigureMockMvc
class OidcLoginTests {

    @Autowired
    MockMvc mvc;

    @Test
    void securedWithRole() throws Exception {
        var id = new OidcIdToken("token", Instant.now(), Instant.now().plusSeconds(60),
                Map.of("sub", "123", "preferred_username", "alice", "roles", java.util.List.of("ADMIN")));
        mvc.perform(get("/admin").with(SecurityMockMvcRequestPostProcessors.oidcLogin().idToken(id)))
            .andExpect(status().isOk());
    }

    @Test
    void forbiddenWithoutRole() throws Exception {
        var id = new OidcIdToken("token", Instant.now(), Instant.now().plusSeconds(60),
                Map.of("sub", "123", "preferred_username", "bob"));
        mvc.perform(get("/admin").with(SecurityMockMvcRequestPostProcessors.oidcLogin().idToken(id)))
            .andExpect(status().isForbidden());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.oidc.tests

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.security.oauth2.core.oidc.OidcIdToken
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.oidcLogin
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.time.Instant

@SpringBootTest
@AutoConfigureMockMvc
class OidcLoginTests(
    @Autowired val mvc: MockMvc
) {

    @Test
    fun securedWithRole() {
        val id = OidcIdToken("token", Instant.now(), Instant.now().plusSeconds(60),
            mapOf("sub" to "123", "preferred_username" to "alice", "roles" to listOf("ADMIN")))
        mvc.perform(get("/admin").with(oidcLogin().idToken(id)))
            .andExpect(status().isOk)
    }

    @Test
    fun forbiddenWithoutRole() {
        val id = OidcIdToken("token", Instant.now(), Instant.now().plusSeconds(60),
            mapOf("sub" to "123", "preferred_username" to "bob"))
        mvc.perform(get("/admin").with(oidcLogin().idToken(id)))
            .andExpect(status().isForbidden)
    }
}
```

**docker-compose.yml — быстрый Keycloak для e2e (демо)**

```yaml
version: "3.8"
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0
    command: ["start-dev", "--http-port=8081", "--import-realm"]
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8081:8081"
    volumes:
      - ./keycloak/realm-export:/opt/keycloak/data/import
```

> Импортируйте заранее экспортированный `realm.json` с клиентом `app-frontend` и разрешённым redirect URI `http://localhost:8080/login/oauth2/code/keycloak`.


# 8. Продвинутые сценарии и кастомизация

**Зависимости для всей подтемы (добавьте один раз и при необходимости расширяйте под конкретный пункт):**

**Gradle (Groovy)**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.4'
    id 'io.spring.dependency-management' version '1.1.6'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-core'
    implementation 'org.springframework.boot:spring-boot-starter-webflux' // для реактивного пункта
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

**Gradle (Kotlin DSL)**

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.4"
    id("io.spring.dependency-management") version "1.1.6"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(17)) } }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-core")
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}
```

---

## Свой AuthenticationProvider (логика входа: API keys, HMAC, OTP, WebAuthn/FIDO2)

Пользовательский `AuthenticationProvider` нужен, когда стандартных провайдеров недостаточно: например, требуется вход по API-ключу в заголовке, верификация HMAC-подписи запроса, одноразовый пароль (TOTP), либо аутентификация по клиентскому сертификату/ключу безопасности (WebAuthn/FIDO2). Суть расширения проста: вы определяете собственный тип `Authentication` (токен запроса) и пишете провайдер, который умеет его валидировать и возвращать «аутентифицированный» токен с полномочиями.

Самый практичный случай для микросервисов — API-ключ. Он прост для интеграций, не требует браузера и часто привязан к техническому аккаунту. Ключ безопаснее хранить как случайный идентификатор, а секрет — хэшированным (например, BCrypt) с возможностью ротации и отзывов. Провайдер должен доставать запись ключа из хранилища, сверять состояние/срок, и формировать `Authentication` с нужными `GrantedAuthority`.

Для сценариев «подписи запроса» (HMAC) вместо одного заголовка у клиента есть `keyId` и вычисленная подпись тела/псевдозаголовков. На сервере вы ищете ключ по `keyId`, заново считаете подпись и сравниваете с предоставленной. Обязательно учитывайте `timestamp`/`nonce`, чтобы предотвратить повтор воспроизведения (replay).

OTP (например, TOTP по RFC 6238) часто используют как второй фактор. В Spring Security его удобно реализовать как «вторую ступень» аутентификации: сначала базовая проверка (пароль/сертификат), затем дополнительный `AuthenticationProvider`, который подтверждает одноразовый код, и только после этого вы даёте «полный» набор authorities.

WebAuthn/FIDO2 — современный безпарольный вариант, где приватный ключ хранится в безопасном ключе (TPM/USB/NFC), а приложение проверяет подпись вызовом к веб-аутентификатору. В серверной части вы храните публичный ключ и счётчик, проверяете подписи/оригин/челлендж. Это существенно усложняет интеграцию, но даёт высокую стойкость к фишингу и краже паролей.

Архитектурно важно разделить «извлечение кредов из запроса» и «проверку». Извлечение удобно делать во `OncePerRequestFilter`: он достаёт заголовки и кладёт предварительный `Authentication` в `SecurityContext` через `AuthenticationManager`. Валидация — задача `AuthenticationProvider`, а не фильтра.

Безопасность в деталях: для API-ключей добавляйте ограничения по IP/рефереру, TTL, квоты. Логируйте использование ключей (кто, когда, какие пути), чтобы видеть утечки и аномалии. Для HMAC следите за канонизацией строк (пробелы, порядок заголовков, нормализация путей).

Относитесь к собственным провайдерам как к публичному контракту: покрывайте негативными тестами (неверный ключ, устаревшая подпись, неверный nonce), метриками (сколько успехов/ошибок), и держите административные операции (выпуск/отзыв) в журналах аудита.

Не смешивайте «технические» API-ключи и «пользовательскую» авторизацию: технические ключи часто описывают «клиента» (сервис/интеграцию), а не пользователя. Выдавайте им отдельные authorities и не позволяйте выполнять пользовательские операции без явного делегирования.

Если нужен «ключ от имени пользователя» (personal access token), описывайте это как отдельный тип токена с явной привязкой к пользователю и сроком жизни. Это упростит аудит и отзыв.

**Java — простой API-key AuthenticationProvider и фильтр**

```java
package com.example.customauth.apikey;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.*;
import org.springframework.security.core.*;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.*;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

class ApiKeyAuthenticationToken extends AbstractAuthenticationToken {
    private final String apiKey;
    private final Object principal;
    ApiKeyAuthenticationToken(String apiKey) { super(null); this.apiKey = apiKey; this.principal = "anonymous"; setAuthenticated(false); }
    ApiKeyAuthenticationToken(Object principal, List<GrantedAuthority> auths) { super(auths); this.apiKey = null; this.principal = principal; setAuthenticated(true); }
    @Override public Object getCredentials() { return apiKey; }
    @Override public Object getPrincipal() { return principal; }
}

class ApiKeyAuthenticationProvider implements AuthenticationProvider {
    @Override public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String key = (String) authentication.getCredentials();
        // Демонстрация: в проде ищите ключ в БД и проверяйте статус/TTL
        if ("demo-api-key-123".equals(key)) {
            return new ApiKeyAuthenticationToken("integration-demo", List.of(new SimpleGrantedAuthority("ROLE_INTEGRATION")));
        }
        throw new BadCredentialsException("Invalid API key");
    }
    @Override public boolean supports(Class<?> authentication) { return ApiKeyAuthenticationToken.class.isAssignableFrom(authentication); }
}

class ApiKeyFilter extends OncePerRequestFilter {
    private final AuthenticationManager authManager;
    ApiKeyFilter(AuthenticationManager authManager) { this.authManager = authManager; }
    @Override protected void doFilterInternal(@NonNull HttpServletRequest req, @NonNull HttpServletResponse res, @NonNull FilterChain chain)
            throws ServletException, IOException {
        String key = req.getHeader("X-API-KEY");
        if (key != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            Authentication pre = new ApiKeyAuthenticationToken(key);
            Authentication auth = authManager.authenticate(pre);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}

@Configuration
class ApiKeySecurityConfig {
    @Bean AuthenticationProvider apiKeyProvider() { return new ApiKeyAuthenticationProvider(); }
    @Bean SecurityFilterChain api(HttpSecurity http, AuthenticationConfiguration cfg) throws Exception {
        http.securityMatcher("/integration/**")
            .authorizeHttpRequests(a -> a.anyRequest().hasRole("INTEGRATION"))
            .csrf(cs -> cs.disable())
            .addFilterBefore(new ApiKeyFilter(cfg.getAuthenticationManager()), UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.customauth.apikey

import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.lang.NonNull
import org.springframework.security.authentication.*
import org.springframework.security.core.Authentication
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
import org.springframework.web.filter.OncePerRequestFilter
import java.io.IOException

class ApiKeyAuthenticationToken : AbstractAuthenticationToken {
    private val apiKey: String?
    private val principalObj: Any
    constructor(apiKey: String) : super(null) { this.apiKey = apiKey; this.principalObj = "anonymous"; isAuthenticated = false }
    constructor(principal: Any, auths: List<GrantedAuthority>) : super(auths) { this.apiKey = null; this.principalObj = principal; isAuthenticated = true }
    override fun getCredentials(): Any? = apiKey
    override fun getPrincipal(): Any = principalObj
}

class ApiKeyAuthenticationProvider : AuthenticationProvider {
    override fun authenticate(authentication: Authentication): Authentication {
        val key = authentication.credentials as String
        if (key == "demo-api-key-123") {
            return ApiKeyAuthenticationToken("integration-demo", listOf(SimpleGrantedAuthority("ROLE_INTEGRATION")))
        }
        throw BadCredentialsException("Invalid API key")
    }
    override fun supports(authentication: Class<*>): Boolean = ApiKeyAuthenticationToken::class.java.isAssignableFrom(authentication)
}

class ApiKeyFilter(private val authManager: AuthenticationManager) : OncePerRequestFilter() {
    @Throws(ServletException::class, IOException::class)
    override fun doFilterInternal(@NonNull req: HttpServletRequest, @NonNull res: HttpServletResponse, @NonNull chain: FilterChain) {
        val key = req.getHeader("X-API-KEY")
        if (key != null && SecurityContextHolder.getContext().authentication == null) {
            val pre = ApiKeyAuthenticationToken(key)
            val auth = authManager.authenticate(pre)
            SecurityContextHolder.getContext().authentication = auth
        }
        chain.doFilter(req, res)
    }
}

@Configuration
class ApiKeySecurityConfig {
    @Bean fun apiKeyProvider(): AuthenticationProvider = ApiKeyAuthenticationProvider()
    @Bean
    fun api(http: org.springframework.security.config.annotation.web.builders.HttpSecurity, cfg: AuthenticationConfiguration): SecurityFilterChain {
        http.securityMatcher("/integration/**")
            .authorizeHttpRequests { it.anyRequest().hasRole("INTEGRRATION") } // опечатку исправим: INTEGRATION
        http.authorizeHttpRequests { it.anyRequest().hasRole("INTEGRATION") }
            .csrf { it.disable() }
            .addFilterBefore(ApiKeyFilter(cfg.authenticationManager), UsernamePasswordAuthenticationFilter::class.java)
        return http.build()
    }
}
```

---

## Несколько SecurityFilterChain (multi-HttpSecurity) для разных участков приложения

В реальных системах часто требуется различная политика для разных «зон»: публичный сайт с формой логина, API-зона с JWT и актюаторы/внутренние точки с техническими ключами. Разделение на несколько `SecurityFilterChain` позволяет не смешивать правила и механизмы, избежать конфликтов (например, редиректы на `/login` внутри API), и упростить ментальную модель.

За приоритет отвечает `@Order`. Цепочка с меньшим `@Order` обрабатывает запрос первой, и если `securityMatcher` совпал — дальнейшие цепочки не участвуют. Поэтому правило «наиболее конкретное — выше» здесь критично: цепь `/api/**` должна иметь более высокий приоритет, чем «общая» web-цепь.

Стейт-менеджмент следует держать независимым: для API — `STATELESS`, выключенный CSRF; для MVC — `IF_REQUIRED` и включённый CSRF; для actuator — чаще всего Basic/токены + строгие роли/сетевые ACL. Так вы минимизируете неожиданные взаимодействия.

Каждая цепь может иметь собственные фильтры. Например, для `/integration/**` вы добавляете API-key-фильтр, а для `/api/**` — `BearerTokenAuthenticationFilter`. Это ещё один аргумент в пользу разделения: фильтры проще сопровождать, когда они применяются только к своим зонам.

Конфликт «кто отвечает 401/302» устраняется автоматически. В web-цепи — `formLogin()` и перенаправления, в API — `BearerTokenAuthenticationEntryPoint` и 401. Клиент получает ожидаемую семантику, DX улучшается.

Не забывайте про статические ресурсы и CORS: их логично обслуживать в web-цепи и явно разрешать. Для API — отдельная CORS-конфигурация «по путям», если фронт общается кросс-доменом.

Тестируйте цепи изолированно. В интеграционных тестах вызывайте конкретные пути и проверяйте статус/заголовки. Ошибки приоритета `@Order` видны сразу: «в API прилетает 302» — верный симптом, что web-цепь стоит выше.

Документируйте зонирование и распечатку правил на ревью. Это «исполняемая документация безопасности», которая экономит часы при расследовании инцидентов и ревью изменений.

Если зон много, лучше вынести каждую цепь в отдельный конфиг-класс с собственным `@Order` и читаемым именем бина. Большие «монолитные» классы с несколькими цепями хуже читаются и реже тестируются.

Следите за отсутствием «дырок» между цепями: последний `anyRequest()` каждой цепи должен быть «закрывающим» — либо `authenticated()`, либо `denyAll()`. Незаполненные «хвосты» порождают случайные открытия.

**Java — три независимые цепочки для /api, /web и /actuator**

```java
package com.example.multichain;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class MultiHttpSecurityConfig {

    @Bean @Order(1)
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a
                .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .anyRequest().authenticated())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(cs -> cs.disable())
            .oauth2ResourceServer(o -> o.jwt());
        return http.build();
    }

    @Bean @Order(2)
    SecurityFilterChain actuator(HttpSecurity http) throws Exception {
        http.securityMatcher("/actuator/**")
            .authorizeHttpRequests(a -> a.requestMatchers("/actuator/health","/actuator/info").permitAll()
                                         .anyRequest().hasRole("OPS"))
            .httpBasic(b -> {})
            .csrf(cs -> cs.disable());
        return http.build();
    }

    @Bean @Order(3)
    SecurityFilterChain web(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/", "/assets/**", "/login").permitAll()
                .anyRequest().authenticated())
            .formLogin(f -> f.loginPage("/login").permitAll());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.multichain

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.http.HttpMethod
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain

@Configuration
class MultiHttpSecurityConfig {

    @Bean @Order(1)
    fun api(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/api/**")
            .authorizeHttpRequests {
                it.requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                    .anyRequest().authenticated()
            }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .csrf { it.disable() }
            .oauth2ResourceServer { it.jwt() }
        return http.build()
    }

    @Bean @Order(2)
    fun actuator(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/actuator/**")
            .authorizeHttpRequests {
                it.requestMatchers("/actuator/health", "/actuator/info").permitAll()
                    .anyRequest().hasRole("OPS")
            }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean @Order(3)
    fun web(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/", "/assets/**", "/login").permitAll()
                    .anyRequest().authenticated()
            }
            .formLogin { it.loginPage("/login").permitAll() }
        return http.build()
    }
}
```

---

## mTLS/Mutual TLS для межсервисного трафика; пиннинг сертификатов

Mutual TLS (mTLS) добавляет взаимную аутентификацию: клиент предъявляет сертификат, сервер проверяет его цепочку доверия и, опционально, атрибуты (CN/SAN). Это удобно для внутризонных вызовов между сервисами и обеспечивает «кто именно передо мной» на уровне канала, до любой логики приложения.

На стороне сервера в Spring Boot mTLS включается конфигурацией SSL: сервер использует свой `keystore` (сертификат/ключ) и доверяет клиентам по `truststore`. Параметр `client-auth: need` заставляет предъявлять клиентский сертификат и завершает рукопожатие, если он не валиден.

После TLS-рукопожатия в приложении становится доступен цепочка `X509Certificate` запроса. Spring Security умеет «доставать» субъект из сертификата через `http.x509()` и строить `X509AuthenticationToken`. Дальше — обычная авторизация по ролям/правам, но уже на уровне маппинга DN/SAN к вашим пользователям/сервисам.

Сертификат-пиннинг — более жёсткая проверка: вы не просто доверяете CA/chain, а ещё и сверяете отпечаток/публичный ключ конкретного сертификата (или ключа). Это снижает риск подмены при компрометации CA/ошибках в цепочке доверия, но требует аккуратной ротации ключей.

Для исходящих запросов (к вашим провайдерам) пиннинг делается в HTTP-клиенте (Reactor Netty/Apache HttpClient): вы пишете `TrustManager`, который сравнивает SPKI-хэш/серийник с белым списком. Важно: **не** отключать валидацию вообще — вы должны сначала пройти стандартную проверку цепочки, затем дополнять её пиннингом.

mTLS не заменяет авторизацию прикладного уровня. Даже зная, какой сервис к вам пришёл, вы должны проверять, **что** он может делать. Хорошая схема — выдать сервисам «технические роли» и проверять их так же, как пользовательские, но с другим набором прав/лимитов.

Сроки жизни и ротация ключей — ключевой операционный аспект. Старайтесь держать короткие сроки (месяцы), автоматизировать выпуск/обновление (ACME/внутренний CA), и тестировать «двойной выпуск» (новый+старый) для плавной замены без даунтайма.

Следите за производительностью: mTLS добавляет стоимость рукопожатия. Для высоконагруженных каналов держите keep-alive, пулы соединений и кэш сессий TLS. На Kubernetes подумайте об оффлоуде TLS на sidecar/ingress, если это соответствует политике.

Аудитируйте DN/SAN подписчика в логах: это ценно при расследовании инцидентов. Но не пишите полный сертификат в лог — достаточно subject/serial.

Для zero-trust-подхода полезно комбинировать mTLS на сетевом уровне и JWT на прикладном (внутри тела запроса). Тогда компрометация одного уровня не открывает систему полностью.

Наконец, тесты: поднимайте Testcontainers с nginx+сертификатами или используйте WireMock c TLS, чтобы прогонять e2e с реальными ключами. Покройте позитивы и негативы (неправильный truststore, неверный клиентский сертификат).

**application.yml — включение mTLS на сервере**

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:tls/server-keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    trust-store: classpath:tls/server-truststore.p12
    trust-store-password: changeit
    trust-store-type: PKCS12
    client-auth: need
```

**Java — маппинг X.509 → пользователь + авторизация**

```java
package com.example.mtls;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class MtlsSecurityConfig {

    @Bean
    SecurityFilterChain mtls(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a
                .requestMatchers("/internal/**").hasRole("INTERNAL")
                .anyRequest().denyAll())
            .x509(x -> x.subjectPrincipalRegex("CN=(.*?)(?:,|$)"))
            .userDetailsService(username -> org.springframework.security.core.userdetails.User
                .withUsername(username)
                .password("{noop}x509") // не используется
                .roles("INTERNAL")
                .build());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.mtls

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class MtlsSecurityConfig {

    @Bean
    fun mtls(http: HttpSecurity): SecurityFilterChain {
        http.authorizeHttpRequests {
                it.requestMatchers("/internal/**").hasRole("INTERNAL")
                    .anyRequest().denyAll()
            }
            .x509 { x -> x.subjectPrincipalRegex("CN=(.*?)(?:,|$)") }
            .userDetailsService { username ->
                org.springframework.security.core.userdetails.User.withUsername(username)
                    .password("{noop}x509").roles("INTERNAL").build()
            }
        return http.build()
    }
}
```

---

## Политики доступа на уровне данных: Row-Level Security (БД) + PermissionEvaluator

Правила на уровне URL/методов — это «внешний периметр». Но утечки часто происходят через «забытый фильтр в запросе». Row-Level Security (RLS) в СУБД добавляет второй замок: даже если код ошибся, БД не вернёт чужие строки. Это особенно важно для многоарендных систем.

В PostgreSQL RLS включается на таблице и задаётся политика, которая использует контекст сессии (например, `current_setting('app.tenant_id')`). Перед запросом приложение устанавливает нужный `tenant_id` в сессии — и любые запросы автоматически ограничиваются.

Однако RLS — не серебряная пуля. Вам всё ещё нужно выражать доменные правила («кто может изменять в статусе X», «владелец или админ»). Здесь помогает `PermissionEvaluator`/`AuthorizationManager`, который сопоставляет текущего пользователя, атрибуты сущности и действие.

Хорошая практика — держать «тонкие» проверки ближе к данным: предикаты в SQL/ORM на уровне репозиториев (например, `where tenant_id = :tenant`). RLS — страховка, а не единственный механизм. Комбинация уменьшает вероятность обхода.

Контекст арендатора удобно передавать из JWT (claim `tenant`) в `SecurityContext`, затем в фильтре/интерсепторе устанавливать в сессию БД. Это делает политику декларативной и независимой от каждого запроса.

С `PermissionEvaluator` выражайте проверки, которые касаются конкретной сущности/операции. Например, «можно ли пользователю P редактировать инвойс I». Внутри evaluator читайте краткие данные (owner/tenant/status) и принимайте решение. Это место для кэширования и оптимизаций.

Аудитируйте отказы/успехи таких решений: это помогает понимать, как «живёт» политика и где её обходят. Не логируйте чувствительные поля, ограничьтесь идентификаторами и статусами.

Тестируйте RLS интеграционно: поднимайте Postgres (Testcontainers), устанавливайте `app.tenant_id` и проверяйте, что «чужие» строки не видны даже при прямом `select *`. Тестируйте evaluator’ы юнитами: подставляйте данные и проверяйте true/false для матрицы кейсов.

Следите за индексами: политика по `tenant_id` должна сопровождаться индексом, иначе полный скан таблицы съест производительность. Это особенно актуально при выборках «по списку арендаторов».

Документируйте зону ответственности: «что закрывает БД», «что проверяет приложение». Это экономит часы на расследования и облегчает ревью изменений.

И наконец, не превращайте SpEL-выражения в «мини-язык правил». Большие выражения вынесите в сервис/`PermissionEvaluator`, оставив в аннотациях только вызов: читабельнее и тестируемее.

**SQL — включение RLS и политика по tenant**

```sql
alter table invoice enable row level security;

create policy invoice_tenant_policy
on invoice
using (tenant_id = current_setting('app.tenant_id', true));
```

**Java — установка tenant в сессии + PermissionEvaluator**

```java
package com.example.rls;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;

import java.io.Serializable;
import java.util.Map;
import java.util.UUID;

@Component
class TenantSession {
    private final JdbcTemplate jdbc;
    TenantSession(JdbcTemplate jdbc) { this.jdbc = jdbc; }
    public void setTenant(String tenant) { jdbc.execute("select set_config('app.tenant_id', '" + tenant + "', true)"); }
}

record Invoice(UUID id, String tenant, String owner) {}

@Component
class InvoicePermissionEvaluator implements PermissionEvaluator {
    @Override public boolean hasPermission(org.springframework.security.core.Authentication auth, Object target, Object perm) {
        if (target instanceof Invoice inv && "EDIT".equals(perm)) {
            return inv.tenant().equals(auth.getAuthorities().stream().filter(a->a.getAuthority().startsWith("TENANT_")).findFirst().map(a->a.getAuthority().substring(7)).orElse("")) &&
                   inv.owner().equals(auth.getName());
        }
        return false;
    }
    @Override public boolean hasPermission(org.springframework.security.core.Authentication auth, Serializable id, String type, Object perm) { return false; }
}

@Service
class InvoiceService {
    private final JdbcTemplate jdbc;
    InvoiceService(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    @PreAuthorize("hasPermission(#inv, 'EDIT')")
    public void update(Invoice inv) {
        jdbc.update("update invoice set ... where id = ?", inv.id());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.rls

import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.security.access.PermissionEvaluator
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.stereotype.Component
import org.springframework.stereotype.Service
import java.io.Serializable
import java.util.UUID

@Component
class TenantSession(private val jdbc: JdbcTemplate) {
    fun setTenant(tenant: String) {
        jdbc.execute("select set_config('app.tenant_id', '$tenant', true)")
    }
}

data class Invoice(val id: UUID, val tenant: String, val owner: String)

@Component
class InvoicePermissionEvaluator : PermissionEvaluator {
    override fun hasPermission(auth: org.springframework.security.core.Authentication, target: Any?, perm: Any?): Boolean {
        val inv = target as? Invoice ?: return false
        if (perm != "EDIT") return false
        val userTenant = auth.authorities.firstOrNull { it.authority.startsWith("TENANT_") }?.authority?.removePrefix("TENANT_") ?: ""
        return inv.tenant == userTenant && inv.owner == auth.name
    }
    override fun hasPermission(auth: org.springframework.security.core.Authentication, targetId: Serializable?, type: String?, perm: Any?): Boolean = false
}

@Service
class InvoiceService(private val jdbc: JdbcTemplate) {
    @PreAuthorize("hasPermission(#inv, 'EDIT')")
    fun update(inv: Invoice) {
        jdbc.update("update invoice set ... where id = ?", inv.id)
    }
}
```

---

## Реактивная безопасность (WebFlux): ServerHttpSecurity, ReactiveJwtAuthenticationConverter

В реактивных приложениях (Spring WebFlux) безопасность строится вокруг `ServerHttpSecurity` и `SecurityWebFilterChain`. Поток обработки — неблокирующий, а значит, блокирующие операции (БД/LDAP) в провайдерах — антипаттерн. Валидация JWT и авторизация должны быть асинхронными/реактивными.

Для ресурс-сервера в WebFlux используется тот же стартер `spring-boot-starter-oauth2-resource-server`, но конфигурация отличается: вместо `HttpSecurity` — `ServerHttpSecurity`, вместо servlet-фильтров — `WebFilter`. Включение JWT делается через `oauth2ResourceServer().jwt()`.

Маппинг claims → authorities в реактивном мире решают через `ReactiveJwtAuthenticationConverter`, который возвращает `Mono<Collection<GrantedAuthority>>`. Часто нужно объединить `scope` и кастомные `roles` в один набор authorities с префиксами `SCOPE_`/`ROLE_`.

Порядок правил и принципы «наиболее конкретное — выше» сохраняются, но API — декларативный DSL для реактивной среды. Ошибки 401/403 отдаются через реактивные entry point/handler’ы, совместимые с WebFlux.

CSRF по умолчанию выключен для WebFlux-API. Для браузерных приложений на WebFlux MVC-подобной логики меньше — чаще делают BFF или чистый API. Поэтому сочетание «JWT+stateless» здесь особенно естественно.

Реактивные контроллеры должны оперировать контекстом безопасности через `ReactiveSecurityContextHolder`. Однако в большинстве случаев достаточно аннотаций `@PreAuthorize` — Spring Security сам подтянет `Authentication` из реактивного контекста.

Помните про «блокировки»: любые вызовы JDBC, файловой системы и др. блокирующие операции должны выполняться вне реактивного потока (например, через `Schedulers.boundedElastic()`), иначе вы теряете смысл WebFlux.

Тестирование — через `WebTestClient`, а не `MockMvc`. Библиотека `spring-security-test` даёт постпроцессоры для реактивных сценариев, включая подстановку JWT/пользователя.

Наблюдаемость в WebFlux — через Micrometer и Reactor-hooks. Важно включить пропагацию MDC/trace-id в реактивном контексте, иначе логи/метрики потеряют корреляцию. Boot 3 делает это «из коробки», но свои фильтры/обработчики стоит писать с оглядкой на контекст.

Наконец, строго разделяйте зависимые от контекста расчёты (например, tenant из токена) и бизнес-логику: слой безопасности должен оставаться тонким, быстрым и предсказуемым.

**Java — WebFlux Resource Server + ReactiveJwtAuthenticationConverter**

```java
package com.example.webflux.sec;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.convert.converter.Converter;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.server.SecurityWebFilterChain;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.Collection;
import java.util.List;
import java.util.Locale;
import java.util.stream.Stream;

@Configuration
public class ReactiveJwtConfig {

    @Bean
    SecurityWebFilterChain api(ServerHttpSecurity http) {
        http.securityMatcher(new org.springframework.web.server.handler.DefaultWebFilterChainSelector.ServerWebExchangeMatcher(e -> e.getRequest().getPath().toString().startsWith("/r/")))
            .authorizeExchange(a -> a.pathMatchers("/r/public/**").permitAll()
                                     .anyExchange().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtAuthConverter())))
            .csrf(cs -> cs.disable());
        return http.build();
    }

    @Bean
    Converter<Jwt, Mono<AbstractAuthenticationToken>> jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter();
        scopes.setAuthorityPrefix("SCOPE_");
        return (Jwt jwt) -> {
            Collection<GrantedAuthority> scopeAuth = scopes.convert(jwt);
            List<String> roles = jwt.getClaimAsStringList("roles");
            Flux<GrantedAuthority> roleAuth = roles == null ? Flux.empty()
                    : Flux.fromStream(roles.stream().map(r -> new SimpleGrantedAuthority("ROLE_" + r.toUpperCase(Locale.ROOT))));
            return Flux.fromIterable(scopeAuth).concatWith(roleAuth)
                    .collectList()
                    .map(auths -> new JwtAuthenticationToken(jwt, auths, jwt.getSubject()));
        };
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.webflux.sec

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.convert.converter.Converter
import org.springframework.security.config.web.server.ServerHttpSecurity
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.server.resource.authentication.AbstractAuthenticationToken
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter
import org.springframework.security.web.server.SecurityWebFilterChain
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import java.util.Locale

@Configuration
class ReactiveJwtConfig {

    @Bean
    fun api(http: ServerHttpSecurity): SecurityWebFilterChain {
        http.authorizeExchange {
                it.pathMatchers("/r/public/**").permitAll()
                    .anyExchange().authenticated()
            }
            .oauth2ResourceServer { it.jwt { j -> j.jwtAuthenticationConverter(jwtAuthConverter()) } }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean
    fun jwtAuthConverter(): Converter<Jwt, Mono<out AbstractAuthenticationToken>> {
        val scopes = JwtGrantedAuthoritiesConverter().apply { setAuthorityPrefix("SCOPE_") }
        return Converter { jwt ->
            val scopeAuth = scopes.convert(jwt)!!
            val roles = jwt.getClaimAsStringList("roles") ?: emptyList()
            val roleAuth = Flux.fromIterable(roles).map { SimpleGrantedAuthority("ROLE_${it.uppercase(Locale.ROOT)}") }
            Flux.fromIterable(scopeAuth).concatWith(roleAuth)
                .collectList()
                .map { auths -> JwtAuthenticationToken(jwt, auths, jwt.subject) }
        }
    }
}
```

---

## Интеграция с аудитом и наблюдаемостью: Micrometer, events, trace-id в решениях

Без наблюдаемости безопасность — «чёрный ящик». Нужны метрики (сколько 401/403, какие причины отказов, какие пути/клиенты), логи с корреляцией (trace-id), и аудит-события (кто вошёл/не вошёл, кто получил доступ/отказ). Всё это должно быть системно и единообразно.

Spring Boot Actuator + Micrometer дают каркас: реестр метрик (`MeterRegistry`) и экспозицию в Prometheus/OTel. На стороне безопасности удобно инкрементировать счётчики в обработчиках 401/403 и подписчиках на события `AuthenticationSuccessEvent`, `AbstractAuthenticationFailureEvent`, `AuthorizationGranted/DeniedEvent`.

Для единообразного учёта авторизаций можно обернуть ваш `AuthorizationManager` в декоратор, который пишет метрики/логи и добавляет теги (путь, метод, решение, роль). Это концентрирует телеметрию в одном месте и не расползается по конфигам.

Trace-id/spans — база для расследований. Включите распределённый трейсинг (OTel/Zipkin) и добавляйте traceId в логи через MDC (в Boot 3 это делается автоматически). В аудит-событиях храните traceId/tenant/subject — этого достаточно, чтобы быстро найти сессию в логах.

Ошибки безопасности возвращайте в формате Problem Details (RFC 7807) и добавляйте в тело поле «код причины» (не чувствительное). Клиентам проще автоматизировать ретраи и показывать осмысленные сообщения.

Собирайте метрики и по «успехам»: процент успешных логинов, среднее время между логином и первой ошибкой 403, распределение по ролям. Это помогает выявлять странные учётки и атаки со «смешанными» правами.

Не забывайте о бюджете метрик: не плодите сериальные карточки на основе userId. Агрегируйте по «безопасным» тегам (путь/роль/исходная подсеть), чтобы не утонуть в числе временных рядов.

Аудит храните дольше, чем обычные логи, но следите за регуляторикой (PII). Маскируйте email/телефон, не логируйте токены/ключи. Для чувствительных операций (смена пароля, выдача ключа) пишите отдельные события с минимальным набором полей.

Выведите SLO/алерты: всплеск 401, рост доли 403 на конкретном пути, резкий рост отказов логина для одной подсети. Это позволит увидеть атаку раньше, чем пользователи поймут, что «что-то не так».

Пишите тесты на метрики: в юнитах можно создать `SimpleMeterRegistry`, вызвать обработчики и проверить значения счётчиков. Это убережёт от «тихого» отваливания телеметрии при рефакторинге.

**Java — метрики для 401/403 и событий аутентификации**

```java
package com.example.obs;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.boot.actuate.audit.AuditEvent;
import org.springframework.boot.actuate.audit.AuditEventRepository;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.event.*;
import org.springframework.security.authorization.event.AuthorizationDeniedEvent;
import org.springframework.security.authorization.event.AuthorizationGrantedEvent;

@Configuration
public class SecurityObservability {

    public SecurityObservability(MeterRegistry registry, AuditEventRepository audit) {
        registry.counter("security.authz.decisions", "result", "granted"); // инициализация
    }

    @Configuration
    static class AuthEvents implements ApplicationListener<AbstractAuthenticationEvent> {
        private final MeterRegistry meter;
        private final AuditEventRepository audit;
        AuthEvents(MeterRegistry meter, AuditEventRepository audit) { this.meter = meter; this.audit = audit; }

        @Override
        public void onApplicationEvent(AbstractAuthenticationEvent e) {
            if (e instanceof AuthenticationSuccessEvent s) {
                meter.counter("security.auth.success").increment();
                audit.add(new AuditEvent(s.getAuthentication().getName(), "AUTH_SUCCESS"));
            } else if (e instanceof AbstractAuthenticationFailureEvent f) {
                meter.counter("security.auth.failure", "reason", f.getException().getClass().getSimpleName()).increment();
                audit.add(new AuditEvent(f.getAuthentication().getName(), "AUTH_FAILURE"));
            }
        }
    }

    @Configuration
    static class AuthzEvents implements ApplicationListener<org.springframework.context.ApplicationEvent> {
        private final MeterRegistry meter;
        private final AuditEventRepository audit;
        AuthzEvents(MeterRegistry meter, AuditEventRepository audit) { this.meter = meter; this.audit = audit; }

        @Override
        public void onApplicationEvent(org.springframework.context.ApplicationEvent e) {
            if (e instanceof AuthorizationGrantedEvent<?> g) {
                meter.counter("security.authz.decisions", "result", "granted").increment();
                audit.add(new AuditEvent(g.getAuthentication().get().getName(), "AUTHZ_GRANTED"));
            } else if (e instanceof AuthorizationDeniedEvent<?> d) {
                meter.counter("security.authz.decisions", "result", "denied").increment();
                audit.add(new AuditEvent(d.getAuthentication().get().getName(), "AUTHZ_DENIED"));
            }
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.obs

import io.micrometer.core.instrument.MeterRegistry
import org.springframework.boot.actuate.audit.AuditEvent
import org.springframework.boot.actuate.audit.AuditEventRepository
import org.springframework.context.ApplicationEvent
import org.springframework.context.ApplicationListener
import org.springframework.context.annotation.Configuration
import org.springframework.security.authentication.event.AbstractAuthenticationEvent
import org.springframework.security.authentication.event.AbstractAuthenticationFailureEvent
import org.springframework.security.authentication.event.AuthenticationSuccessEvent
import org.springframework.security.authorization.event.AuthorizationDeniedEvent
import org.springframework.security.authorization.event.AuthorizationGrantedEvent

@Configuration
class SecurityObservability(registry: MeterRegistry, audit: AuditEventRepository) {
    init { registry.counter("security.authz.decisions", "result", "granted") }
}

@Configuration
class AuthEvents(private val meter: MeterRegistry, private val audit: AuditEventRepository) :
    ApplicationListener<AbstractAuthenticationEvent> {
    override fun onApplicationEvent(e: AbstractAuthenticationEvent) {
        when (e) {
            is AuthenticationSuccessEvent -> {
                meter.counter("security.auth.success").increment()
                audit.add(AuditEvent(e.authentication.name, "AUTH_SUCCESS"))
            }
            is AbstractAuthenticationFailureEvent -> {
                meter.counter("security.auth.failure", "reason", e.exception.javaClass.simpleName).increment()
                audit.add(AuditEvent(e.authentication.name, "AUTH_FAILURE"))
            }
        }
    }
}

@Configuration
class AuthzEvents(private val meter: MeterRegistry, private val audit: AuditEventRepository) :
    ApplicationListener<ApplicationEvent> {
    override fun onApplicationEvent(e: ApplicationEvent) {
        when (e) {
            is AuthorizationGrantedEvent<*> -> {
                meter.counter("security.authz.decisions", "result", "granted").increment()
                audit.add(AuditEvent(e.authentication.get().name, "AUTHZ_GRANTED"))
            }
            is AuthorizationDeniedEvent<*> -> {
                meter.counter("security.authz.decisions", "result", "denied").increment()
                audit.add(AuditEvent(e.authentication.get().name, "AUTHZ_DENIED"))
            }
        }
    }
}
```

**application.yml — экспонируем метрики и трассировку**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,prometheus"
  metrics:
    tags:
      application: orders-api
  tracing:
    enabled: true
```

# 9. Тестирование безопасности

**Зависимости для всей подтемы (вынесены один раз, используйте по мере необходимости).**

**Gradle (Groovy)**

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testImplementation 'io.rest-assured:rest-assured:5.5.0'
    testImplementation 'org.testcontainers:junit-jupiter:1.20.2'
    testImplementation 'org.testcontainers:postgresql:1.20.2'
    testImplementation 'dasniko:testcontainers-keycloak:3.3.1'
    testImplementation 'com.nimbusds:nimbus-jose-jwt:9.40' // генерация/парсинг JWT в тестах
}
test {
    useJUnitPlatform()
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("io.rest-assured:rest-assured:5.5.0")
    testImplementation("org.testcontainers:junit-jupiter:1.20.2")
    testImplementation("org.testcontainers:postgresql:1.20.2")
    testImplementation("dasniko:testcontainers-keycloak:3.3.1")
    testImplementation("com.nimbusds:nimbus-jose-jwt:9.40")
}
tasks.test { useJUnitPlatform() }
```

---

## Spring Security Test: @WithMockUser, @WithAnonymousUser, тестирование URL-правил

Первое, что нужно освоить для быстрых и надёжных юнит-/интеграционных тестов безопасности, — библиотека `spring-security-test`. Она подключает «постпроцессоры» к MockMvc и удобные аннотации, позволяющие подменять аутентификацию без настоящего логина, редиректов и токенов. Это резко ускоряет цикл TDD: вы пишете правило в `HttpSecurity`, сразу закрепляете его тестом и получаете быстрый фидбэк.

Аннотация `@WithMockUser` имитирует аутентифицированного пользователя с заданным именем и ролями. Важно помнить, что роли в аннотации указываются **без** префикса `ROLE_` — его Spring добавит сам. Это помогает проверить, что `hasRole("ADMIN")` действительно работает, а `hasAuthority("ROLE_ADMIN")` эквивалентен в вашем коде.

Аннотация `@WithAnonymousUser` явно фиксирует поведение для анонима: это полезно, чтобы убедиться, что публичные пути (`/health`, `/docs`) доступны, а остальные возвращают 401/302. Частая ошибка: правила авторизации написаны, но из-за «дырки» в порядке матчеров аноним случайно получает доступ.

Пакет `SecurityMockMvcRequestPostProcessors` даёт тонкую настройку контекста: можно подложить пользователя с конкретными authorities, добавить CSRF-токен, выставить заголовки. Это удобно для негативных кейсов: «без CSRF — 403», «без роли — 403», «с ролью — 200».

Проверяйте не только статус, но и **маршрутизацию**: в web-цепочке при доступе без логина нужно ожидать 302 на `/login`, а в API-цепочке — 401 без редиректа. Такой тест ловит перепутанные `SecurityFilterChain` и неправильный `@Order`.

Разделяйте тесты по зонам: `/api/**`, `/web/**`, `/actuator/**`. Это снижает когнитивную нагрузку — вы точно видите, какой набор правил проверяется, и быстрее находите регрессии. Хорошая практика — один класс тестов на одну цепочку безопасности.

Чтобы тесты были стабильны, инициализируйте контекст безопасности явно в каждом методе, а не полагайтесь на «остатки» от предыдущего теста. `spring-security-test` сам чистит контекст между тестами, но явные аннотации делают намерения прозрачными.

Проверяйте порядок правил: создайте два теста на похожие пути (`/admin` и `/admin/public`), чтобы обеспечить покрытие по принципу «наиболее конкретное — выше». Это частый источник багов при рефакторинге DSL.

Для многоарендных приложений добавляйте в mock-пользователя tenant-атрибуты (через `SecurityContextHolder` или кастомный `Authentication`). Это позволит валидировать SpEL-предикаты в `@PreAuthorize` и PermissionEvaluator.

И наконец, держите рядом «таблицу истинности»: набор тестов, где для каждого ключевого URL проверяются 3 сценария — аноним, аутентифицированный без роли, аутентифицированный с ролью. Такой минимум часто спасает от незаметных дырок.

**Java — MockMvc + @WithMockUser/@WithAnonymousUser**

```java
package com.example.sec.test;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithAnonymousUser;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class UrlRulesTests {

    @Autowired MockMvc mvc;

    @Test
    @WithAnonymousUser
    void adminShouldRedirectToLoginForAnonymous() throws Exception {
        mvc.perform(get("/admin"))
           .andExpect(status().is3xxRedirection())
           .andExpect(header().string("Location", "/login"));
    }

    @Test
    @WithMockUser(username = "alice", roles = {"USER"})
    void adminShouldBeForbiddenForUserRole() throws Exception {
        mvc.perform(get("/admin")).andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "bob", roles = {"ADMIN"})
    void adminShouldBeOkForAdmin() throws Exception {
        mvc.perform(get("/admin")).andExpect(status().isOk());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.sec.test

import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.security.test.context.support.WithAnonymousUser
import org.springframework.security.test.context.support.WithMockUser
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.header
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status

@SpringBootTest
@AutoConfigureMockMvc
class UrlRulesTests(
    @Autowired val mvc: MockMvc
) {

    @Test
    @WithAnonymousUser
    fun adminShouldRedirectToLoginForAnonymous() {
        mvc.perform(get("/admin"))
            .andExpect(status().is3xxRedirection)
            .andExpect(header().string("Location", "/login"))
    }

    @Test
    @WithMockUser(username = "alice", roles = ["USER"])
    fun adminShouldBeForbiddenForUserRole() {
        mvc.perform(get("/admin")).andExpect(status().isForbidden)
    }

    @Test
    @WithMockUser(username = "bob", roles = ["ADMIN"])
    fun adminShouldBeOkForAdmin() {
        mvc.perform(get("/admin")).andExpect(status().isOk)
    }
}
```

---

## MockMvc/WebTestClient/RestAssured: проверка 401/403, CSRF, CORS, headers

Второй слой уверенности — проверка протокольных деталей. С MockMvc удобно тестировать форму логина, редиректы и CSRF. RestAssured хорошо подходит для «проводных» проверок заголовков и CORS: вы видите ровно то, что увидит HTTP-клиент. WebTestClient полезен, если вы используете WebFlux или хотите тестировать поднятое приложение на `RANDOM_PORT`.

Для CSRF-ветки создайте два теста на одну и ту же форму/эндпоинт: без токена должен быть 403, с токеном — 2xx. В `spring-security-test` есть `.with(csrf())`, который добавляет корректный токен в запрос. Такой тест предотвращает «случайное» отключение CSRF при рефакторинге цепочки.

CORS следует тестировать реальными preflight-запросами `OPTIONS` и обычными запросами с заголовками `Origin` и `Access-Control-Request-Method`. Ответ должен содержать корректные `Access-Control-Allow-*`, а для «запрещённого» источника — не содержать их вовсе. Браузер именно так решает судьбу кросс-доменного вызова.

Заголовки безопасности (HSTS, X-Content-Type-Options, CSP, X-Frame-Options) проверяйте как часть smoke-тестов. Для API на отдельном домене важно, чтобы даже 401/403 включали нужные заголовки — иначе браузер может скрыть тело и не отдать его JS.

RestAssured позволяет писать читаемые проверки: `given().when().get("/api").then().statusCode(401)`. Добавляйте `.header(...)` для проверки HSTS/CSP, но не уходите в «пиксель-перфект» — директивы CSP могут меняться, фиксируйте базовый минимум.

WebTestClient уместен даже в servlet-приложении: можно биндингом к `RANDOM_PORT` отправлять реальные HTTP и ловить ошибки прокси/фильтров. Это полезно для CORS/headers, где MockMvc (в отличие от браузера) не всегда проявляет тайные углы.

Отдельно проверьте, что «не JSON» методы защищены корректно: например, `GET /logout` должен быть недоступен или требовать CSRF. Такие проверки ловят регрессии после обновления Spring Boot.

Быстрый лайфхак: заведите утиль-методы для RestAssured, которые автоматически добавляют `Origin`, нужные заголовки и валидируют «коробку» безопасности. Это уменьшает копипасту и дисциплинирует команду.

Следите за независимостью тестов: не «подготавливайте» их порядком запуска. Каждый тест должен сам выставлять контекст (аннотации, CSRF) и не зависеть от предыдущего.

И в конце — всегда фиксируйте негативные кейсы для ключевых путей: без токена, с «битым» токеном, без роли. Это основной контракт безопасности вашего API.

**Java — CSRF + CORS + headers (MockMvc/RestAssured)**

```java
package com.example.sec.http;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;
import io.restassured.RestAssured;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class HttpContractsTests {

    @Autowired MockMvc mvc;

    @Test
    void csrfRequired() throws Exception {
        mvc.perform(post("/form/submit")).andExpect(status().isForbidden());
        mvc.perform(post("/form/submit").with(csrf())).andExpect(status().is2xxSuccessful());
    }

    @Test
    void corsPreflightAllowed() {
        RestAssured.given()
            .header("Origin", "https://app.example.com")
            .header("Access-Control-Request-Method", "POST")
            .when().options("/api/orders")
            .then().statusCode(200)
            .header("Access-Control-Allow-Origin", "https://app.example.com")
            .header("Access-Control-Allow-Methods", org.hamcrest.Matchers.containsString("POST"));
    }

    @Test
    void securityHeadersPresent() {
        RestAssured.given()
            .when().get("/any")
            .then().statusCode(org.hamcrest.Matchers.anyOf(org.hamcrest.Matchers.is(200), org.hamcrest.Matchers.is(302), org.hamcrest.Matchers.is(401)))
            .header("X-Content-Type-Options", "nosniff")
            .header("X-Frame-Options", org.hamcrest.Matchers.anyOf(org.hamcrest.Matchers.is("DENY"), org.hamcrest.Matchers.is("SAMEORIGIN")));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.sec.http

import io.restassured.RestAssured
import org.hamcrest.Matchers.anyOf
import org.hamcrest.Matchers.containsString
import org.hamcrest.Matchers.is
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class HttpContractsTests(@Autowired val mvc: MockMvc) {

    @Test
    fun csrfRequired() {
        mvc.perform(post("/form/submit")).andExpect(status().isForbidden)
        mvc.perform(post("/form/submit").with(csrf())).andExpect(status().is2xxSuccessful)
    }

    @Test
    fun corsPreflightAllowed() {
        RestAssured.given()
            .header("Origin", "https://app.example.com")
            .header("Access-Control-Request-Method", "POST")
            .`when`().options("/api/orders")
            .then().statusCode(200)
            .header("Access-Control-Allow-Origin", "https://app.example.com")
            .header("Access-Control-Allow-Methods", containsString("POST"))
    }

    @Test
    fun securityHeadersPresent() {
        RestAssured.given()
            .`when`().get("/any")
            .then().statusCode(anyOf(is(200), is(302), is(401)))
            .header("X-Content-Type-Options", "nosniff")
            .header("X-Frame-Options", anyOf(is("DENY"), is("SAMEORIGIN")))
    }
}
```

---

## Мок OIDC/JWT: Nimbus test keys, JwtDecoder bean override, тестовые JWKs

Работать с внешним IdP в каждом тесте дорого и ненадёжно. Чаще всего вам нужен контролируемый токен с нужными claims. Подходов два: подменить `JwtDecoder` в тестовом контексте или поднять «фейковый» JWKS (локальный HTTP-сервер, отдающий JWK Set). Оба варианта позволяют выпускать токены тестовым ключом и проверять маппинг/валидацию.

Подмена бина — самый простой путь. В `@TestConfiguration` создайте RSA-пару через Nimbus JOSE, опубликуйте `JwtDecoder` и `JwtEncoder` (опционально), чтобы выпускать и проверять токены в одном процессе. Так вы тестируете весь путь resource-server без сети и нестабильностей.

Если хотите ближе к «бою», поднимайте tiny HTTP-сервер (например, через `MockWebServer` или `WireMock`), который отдаёт `/.well-known/jwks.json`. Тогда `NimbusJwtDecoder` будет вести себя как в проде, вытягивая ключи по `kid`. Это полезно для сценариев ротации ключей.

Токены удобно собирать через `JWTClaimsSet.Builder()` из Nimbus: добавляете `iss`, `aud`, `sub`, `scope`, `roles`, ставите `exp/iat`, подписываете `SignedJWT` и вкладываете строку в заголовок Authorization. Такой токен ассемблируется в нескольких строках.

Следите за соответствием ваших валидаторов: если в коде стоит `createDefaultWithIssuer("https://idp.example.com")` — тестовый `iss` должен совпадать. Иначе получите 401 и ложное чувство «всё сломано». В тестовой конфигурации упростите валидаторы, либо выставьте такой же `iss`.

Для multi-issuer используйте `JwtIssuerAuthenticationManagerResolver` и отдавайте разные токены в разных тестах. Это пригодится, если у вас multi-tenant.

Подменяйте `Clock`/`Instant` для проверки `exp/nbf/iat`. Это позволяет писать deterministic-тесты на «просроченный» и «будущий» токены без сна `Thread.sleep`.

Добавляйте негативные тесты: «подпись не та», «aud не совпадает», «нет нужного scope». Эти тесты ценные: они фиксируют вашу модель угроз, а не только «зелёные» сценарии.

Не логируйте токен целиком в тест-логах — даже тестовые секреты лучше не разбрасывать в CI-артефактах. Если очень нужно — маскируйте середину.

И наконец, придерживайтесь конвенций префиксов (`ROLE_`/`SCOPE_`) в генерации токена. Тогда ваши проверочные выражения в контроллерах/конфиге будут работать идентично продовым.

**Java — TestConfiguration: генерация RSA, override JwtDecoder, выпуск тестового JWT**

```java
package com.example.jwt.tests;

import com.nimbusds.jose.*;
import com.nimbusds.jose.jwk.*;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.*;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;

import java.time.Instant;
import java.util.Date;
import java.util.List;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class JwtMockTests {

    @Autowired MockMvc mvc;
    @Autowired JwtDecoder decoder; // используется ресурс-сервером
    @Autowired TestJwtFactory jwtFactory;

    @TestConfiguration
    static class TestJwtConfig {
        @Bean
        public RSAKey rsaJwk() throws Exception {
            return new RSAKeyGenerator(2048).keyID("test-kid").generate();
        }
        @Bean
        public JwtDecoder jwtDecoder(RSAKey jwk) {
            return NimbusJwtDecoder.withPublicKey(jwk.toRSAPublicKey()).build();
        }
        @Bean
        public TestJwtFactory testJwtFactory(RSAKey jwk) { return new TestJwtFactory(jwk); }
    }

    static class TestJwtFactory {
        private final RSAKey jwk;
        TestJwtFactory(RSAKey jwk) { this.jwk = jwk; }

        String issue(List<String> scopes, List<String> roles) throws Exception {
            var now = Instant.now();
            var claims = new JWTClaimsSet.Builder()
                .issuer("https://idp.test")
                .audience("orders-api")
                .subject("alice")
                .claim("scope", String.join(" ", scopes))
                .claim("roles", roles)
                .issueTime(Date.from(now))
                .expirationTime(Date.from(now.plusSeconds(600)))
                .build();
            var header = new JWSHeader.Builder(JWSAlgorithm.RS256).keyID(jwk.getKeyID()).build();
            var jws = new SignedJWT(header, claims);
            jws.sign(new RSASSASigner(jwk.toPrivateKey()));
            return jws.serialize();
        }
    }

    @Test
    void apiRequiresToken() throws Exception {
        mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized());
    }

    @Test
    void apiAcceptsValidToken() throws Exception {
        String token = jwtFactory.issue(List.of("orders.read"), List.of("USER"));
        mvc.perform(get("/api/orders").header("Authorization", "Bearer " + token))
           .andExpect(status().isOk());
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.jwt.tests

import com.nimbusds.jose.JWSAlgorithm
import com.nimbusds.jose.JWSHeader
import com.nimbusds.jose.crypto.RSASSASigner
import com.nimbusds.jose.jwk.RSAKey
import com.nimbusds.jose.jwk.gen.RSAKeyGenerator
import com.nimbusds.jwt.JWTClaimsSet
import com.nimbusds.jwt.SignedJWT
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.context.TestConfiguration
import org.springframework.context.annotation.Bean
import org.springframework.security.oauth2.jwt.JwtDecoder
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.time.Instant
import java.util.Date

@SpringBootTest
@AutoConfigureMockMvc
class JwtMockTests(
    @Autowired val mvc: MockMvc,
    @Autowired val jwtFactory: TestJwtFactory
) {

    @TestConfiguration
    class TestJwtConfig {
        @Bean fun rsaJwk(): RSAKey = RSAKeyGenerator(2048).keyID("test-kid").generate()
        @Bean fun jwtDecoder(jwk: RSAKey): JwtDecoder = NimbusJwtDecoder.withPublicKey(jwk.toRSAPublicKey()).build()
        @Bean fun testJwtFactory(jwk: RSAKey) = TestJwtFactory(jwk)
    }

    class TestJwtFactory(private val jwk: RSAKey) {
        fun issue(scopes: List<String>, roles: List<String>): String {
            val now = Instant.now()
            val claims = JWTClaimsSet.Builder()
                .issuer("https://idp.test")
                .audience("orders-api")
                .subject("alice")
                .claim("scope", scopes.joinToString(" "))
                .claim("roles", roles)
                .issueTime(Date.from(now))
                .expirationTime(Date.from(now.plusSeconds(600)))
                .build()
            val header = JWSHeader.Builder(JWSAlgorithm.RS256).keyID(jwk.keyID).build()
            val jws = SignedJWT(header, claims)
            jws.sign(RSASSASigner(jwk.toPrivateKey()))
            return jws.serialize()
        }
    }

    @Test
    fun apiRequiresToken() {
        mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized)
    }

    @Test
    fun apiAcceptsValidToken() {
        val token = jwtFactory.issue(listOf("orders.read"), listOf("USER"))
        mvc.perform(get("/api/orders").header("Authorization", "Bearer $token"))
            .andExpect(status().isOk)
    }
}
```

---

## Контрактные тесты для защищённых API: OpenAPI + negative cases

Контрактные тесты проверяют не реализацию, а **соглашение** между сервисом и потребителем. В безопасности это особенно важно: мы фиксируем, что заданные эндпоинты требуют аутентификацию, описаны нужные коды ошибок (401/403) и возвращают унифицированный формат Problem Details. Такой подход уменьшает количество «на глазок» интерпретаций.

Если вы публикуете OpenAPI (springdoc), добавьте security-схему (bearer/JWT) и отметьте, какие пути защищены. Дальше можно написать тест, который загрузит сгенерированный JSON и проверит, что «критичные» пути действительно имеют раздел `security`. Так вы поймаете случай, когда разработчик добавил эндпоинт и забыл пометить его защищённым.

Второй пласт — негативные кейсы. Для каждого защищённого пути должен существовать тест «без токена → 401», «без нужной роли → 403». Это автоматизируется через параметризованные тесты на основе OpenAPI: вытягиваем все пути с тегом `secured` и прогоняем их в цикле.

Контракт ошибки важен не меньше. Если в проекте принят RFC 7807, убедитесь, что тело ошибок содержит `type/title/status/detail`, и что при 401 отдан заголовок `WWW-Authenticate: Bearer`. Клиентам и гейтвеям это критично.

Генерация тестов «из OpenAPI» дисциплинирует документацию: если документация несовместима с реализацией — тест падает. Это хороший двигатель качества, особенно в больших командах и при внешних интеграциях.

Не путайте контракт с e2e: контрактные тесты могут работать на MockMvc (без сети) и быть очень быстрыми. Они не проверяют IdP, CORS и прокси — только договор между контроллером и клиентом.

Для внутренних систем можно добавить кастомные расширения в OpenAPI (x-requires-scope) и валидировать их: «эндпоинт должен требовать `orders.read`». Так вы формализуете политику прав на уровне артефакта.

При версионировании API держите контрактные тесты на каждую ветку схемы (v1/v2), чтобы миграции прав не сломали старых клиентов. Это особенно важно при «усилении» требований (добавили роль к операции).

Старайтесь не «мокать» безопасность в контрактных тестах — иначе потеряете смысл. Лучше подложите «битый» токен или вообще его не кладите, чтобы увидеть реальные 401/403.

И, наконец, не забывайте о документации для клиентов: в OpenAPI опишите, где получать токен, какие скоупы нужны, какие типовые ошибки возможны. Это часть DX и снижает число вопросов в интеграциях.

**Java — верификация OpenAPI: security на путях + негатив через RestAssured**

```java
package com.example.contract;

import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.core.io.ClassPathResource;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.InputStream;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OpenApiContractTests {

    @Test
    void openapiHasSecurityOnProtectedPaths() throws Exception {
        try (InputStream is = new ClassPathResource("static/openapi.json").getInputStream()) {
            JsonNode root = new ObjectMapper().readTree(is);
            JsonNode paths = root.get("paths");
            paths.fields().forEachRemaining(e -> {
                String path = e.getKey();
                if (path.startsWith("/api/orders")) {
                    JsonNode get = e.getValue().get("get");
                    assertThat(get.get("security")).isNotNull();
                }
            });
        }
    }

    @Test
    void negativeUnauthorizedByContract() {
        RestAssured.given().when().get("/api/orders").then().statusCode(401);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.contract

import com.fasterxml.jackson.databind.ObjectMapper
import io.restassured.RestAssured
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.core.io.ClassPathResource

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OpenApiContractTests {

    @Test
    fun openapiHasSecurityOnProtectedPaths() {
        ClassPathResource("static/openapi.json").inputStream.use { isr ->
            val root = ObjectMapper().readTree(isr)
            val paths = root["paths"]
            val iter = paths.fields()
            while (iter.hasNext()) {
                val (path, node) = iter.next()
                if (path.startsWith("/api/orders")) {
                    val get = node["get"]
                    assertThat(get["security"]).isNotNull
                }
            }
        }
    }

    @Test
    fun negativeUnauthorizedByContract() {
        RestAssured.given().`when`().get("/api/orders").then().statusCode(401)
    }
}
```

---

## Интеграция с Testcontainers Keycloak/OPA для реалистичных e2e

Ни один мок не заменит «живой» поток авторизации. Для e2e полезно поднимать IdP в контейнере. Модуль `dasniko/testcontainers-keycloak` стартует Keycloak с импортом realm и клиентов, что даёт вам «настоящие» id_token/access-токены и проверку URL-правил, маппинга ролей и SSO-редиректов. Это медленнее, но необходимо хотя бы для «золотого пути».

Поднимите Keycloak с `start-dev` и импортом realm, где настроены клиенты (`public/confidential`), роли и маппинг «группы → roles». В тесте получите токен через admin API или «публичный» password flow (для e2e это допустимо) и используйте его против своего ресурс-сервера. Так вы валидируете и подписи, и `iss/aud`, и свои валидаторы.

Open Policy Agent (OPA) — отличный инструмент для вынесения политик авторизации. В тестах его тоже можно поднять контейнером и подложить простую Rego-политику. Сервис будет запрашивать у OPA «разрешить/запретить» и принимать решение. Такой e2e показывает, что интеграция работает и при отказах (OPA недоступен) поведение соответствует «fail-closed».

В e2e важно управлять временем: добавляйте `await().untilAsserted(...)` вокруг зависимых шагов, Keycloak может не сразу выдать JWKS. Это предотвратит «пляшущие» падения в CI. Также полезно прогревать JWKS запросом к вашему API с валидным токеном.

Думайте о производительности CI: e2e помечайте отдельным тэгом/профилем и гоняйте их реже, чем быстрые юниты. Но не убирайте полностью — «золотой путь» с реальным токеном должен проходить на каждом релизе.

Отдельно тестируйте ротацию ключей: перезапустите Keycloak с новым realm-экспортом, чтобы JWKS обновился, и убедитесь, что ресурс-сервер продолжает принимать токены с `kid` нового ключа.

Для OPA зафиксируйте контракт «что передаём в input»: user, roles, path, method, tenant. На тестах прогоняйте позитивы/негативы для разных комбинаций, чтобы политика не позволила лишнего.

Не забывайте очищать ресурсы: Testcontainers делает это сам, но при падениях тестов полезно иметь отладочные логи Keycloak/OPA (включите `withLogConsumer`). Они помогут понять, почему «вчера работало, сегодня — нет».

И наконец, держите минимальный realm для тестов: чем меньше «магии», тем стабильнее CI. Всегда можно добавить второй набор e2e с более богатым сценарием на nightly.

**Java — e2e с Keycloak и OPA (минимальный скелет)**

```java
package com.example.e2e;

import dasniko.testcontainers.keycloak.KeycloakContainer;
import org.junit.jupiter.api.*;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.MountableFile;
import io.restassured.RestAssured;

import static org.hamcrest.Matchers.is;

@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class E2eSecurityTests {

    static KeycloakContainer keycloak = new KeycloakContainer("quay.io/keycloak/keycloak:25.0")
            .withRealmImportFile("keycloak/realm-export.json")
            .withReuse(false);
    static GenericContainer<?> opa = new GenericContainer<>("openpolicyagent/opa:latest")
            .withExposedPorts(8181)
            .withCopyFileToContainer(MountableFile.forClasspathResource("opa/policy.rego"), "/policy.rego")
            .withCommand("run", "--server", "/policy.rego");

    @BeforeAll
    static void start() { keycloak.start(); opa.start(); }

    @AfterAll
    static void stop() { opa.stop(); keycloak.stop(); }

    @Test @Order(1)
    void shouldRejectWithoutToken() {
        RestAssured.given().when().get("http://localhost:8080/api/orders").then().statusCode(401);
    }

    @Test @Order(2)
    void shouldAcceptValidTokenFromKeycloak() {
        // Упрощённо: получаем password-грантом (для e2e это допустимый трюк)
        var token = RestAssured.given()
            .formParam("grant_type", "password")
            .formParam("client_id", "app-client")
            .formParam("username", "alice")
            .formParam("password", "alice")
            .post(keycloak.getAuthServerUrl() + "/realms/demo/protocol/openid-connect/token")
            .then().statusCode(200).extract().path("access_token");
        RestAssured.given().header("Authorization", "Bearer " + token)
            .when().get("http://localhost:8080/api/orders")
            .then().statusCode(200);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.e2e

import dasniko.testcontainers.keycloak.KeycloakContainer
import io.restassured.RestAssured
import org.hamcrest.Matchers.isA
import org.junit.jupiter.api.*
import org.testcontainers.containers.GenericContainer
import org.testcontainers.utility.MountableFile

@TestMethodOrder(MethodOrderer.OrderAnnotation::class)
class E2eSecurityTests {

    companion object {
        private val keycloak = KeycloakContainer("quay.io/keycloak/keycloak:25.0")
            .withRealmImportFile("keycloak/realm-export.json")
            .withReuse(false)

        private val opa = GenericContainer("openpolicyagent/opa:latest")
            .withExposedPorts(8181)
            .withCopyFileToContainer(MountableFile.forClasspathResource("opa/policy.rego"), "/policy.rego")
            .withCommand("run", "--server", "/policy.rego")

        @BeforeAll @JvmStatic fun start() { keycloak.start(); opa.start() }
        @AfterAll @JvmStatic fun stop() { opa.stop(); keycloak.stop() }
    }

    @Test @Order(1)
    fun shouldRejectWithoutToken() {
        RestAssured.given().`when`().get("http://localhost:8080/api/orders").then().statusCode(401)
    }

    @Test @Order(2)
    fun shouldAcceptValidTokenFromKeycloak() {
        val token = RestAssured.given()
            .formParam("grant_type", "password")
            .formParam("client_id", "app-client")
            .formParam("username", "alice")
            .formParam("password", "alice")
            .post("${keycloak.authServerUrl}/realms/demo/protocol/openid-connect/token")
            .then().statusCode(200).extract().path<String>("access_token")
        RestAssured.given().header("Authorization", "Bearer $token")
            .`when`().get("http://localhost:8080/api/orders")
            .then().statusCode(200)
    }
}
```

**resources/opa/policy.rego (пример политики):**

```rego
package http.authz

default allow = false

allow {
  input.method == "GET"
  startswith(input.path, "/api/orders")
  some r
  r := input.roles[_]
  r == "ROLE_USER"
}
```

---

## Линт/скан: проверка заголовков, секретов, dependency audit (OWASP, Snyk, osv)

Тесты — это не всё. Необходимо автоматизировать «санитарную» проверку: наличие security-заголовков, отсутствие секретов в репозитории, уязвимости в зависимостях. Эти проверки быстро находят классы проблем, которые трудно поймать обычными юнитами.

Для заголовков заведите простой smoke-тест (на RestAssured/MockMvc), который дергает несколько URL и проверяет HSTS, X-Content-Type-Options, X-Frame-Options, базовую CSP. Такой тест мало чувствителен к бизнес-логике и падает только при реальной конфигурационной проблеме.

Проверку секретов имеет смысл встроить в CI с использованием детекторов (GitHub Secret Scanning, TruffleHog, gitleaks). На локальном уровне можно хранить pre-commit hook, который отсекает очевидные ключи (`-----BEGIN PRIVATE KEY-----`, `AKIA...` и т.д.). Для Java — не кладите секреты в `application.yml`, используйте переменные окружения/сикрет-менеджеры.

Dependency audit — отдельная линия обороны. Инструменты: OWASP Dependency Check (плагин Gradle), Snyk CLI/Gradle plugin, `gradle-versions-plugin` + консультация в OSV. Регулярный отчёт в CI с fail-on-high — хорошая привычка. Главное — не глушите алерты, а разруливайте: обновление, исключение, временный workaround.

В «линте безопасности» полезно проверять и CORS: на тестовом стенде запросы с неожиданными `Origin` не должны проходить preflight. Этот тест можно запустить автономно без контекста базы.

Для репозиториев артефактов включайте политики «только whitelisted источники» и проверяйте подписи/суммы. Supply-chain сегодня уязвим не меньше, чем ваш код.

Документируйте исключения. Если вы отключили какой-то заголовок или разрешили специфический `Origin`, это должно быть видно в PR/плейбуке. Иначе через месяц никто не вспомнит «почему так».

Регулярно запускайте dependency audit не только в PR, но и по расписанию (раз в день/неделю). Уязвимости появляются без ваших изменений — вы должны их увидеть вовремя.

Проверяйте на CI, что сборка падает, если пропал обязательный заголовок. Это простое, но действенное правило: конфигурацию нельзя «потерять» незаметно.

И наконец, объедините все «линты» в один отчёт пайплайна, чтобы разработчик видел сводку «прошло/провалилось» и не бегал по разным джобам.

**Gradle (Groovy) — OWASP Dependency Check**

```groovy
plugins {
    id 'org.owasp.dependencycheck' version '10.0.4'
}
dependencyCheck {
    failBuildOnCVSS = 7.0
    suppressionFile = 'config/dependency-check-suppressions.xml'
}
```

**Gradle (Kotlin DSL) — OWASP Dependency Check**

```kotlin
plugins {
    id("org.owasp.dependencycheck") version "10.0.4"
}
dependencyCheck {
    failBuildOnCVSS = 7.0.toBigDecimal()
    suppressionFile = "config/dependency-check-suppressions.xml"
}
```

**Java — smoke-тест заголовков безопасности**

```java
package com.example.lint;

import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

class SecurityHeadersLintTest {

    @Test
    void headersPresent() {
        RestAssured.given()
            .when().get("http://localhost:8080/")
            .then().statusCode(org.hamcrest.Matchers.any(Integer.class))
            .header("X-Content-Type-Options", "nosniff")
            .header("X-Frame-Options", org.hamcrest.Matchers.anyOf(org.hamcrest.Matchers.is("DENY"), org.hamcrest.Matchers.is("SAMEORIGIN")));
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.lint

import io.restassured.RestAssured
import org.hamcrest.Matchers.any
import org.hamcrest.Matchers.anyOf
import org.hamcrest.Matchers.isA
import org.junit.jupiter.api.Test

class SecurityHeadersLintTest {
    @Test
    fun headersPresent() {
        RestAssured.given()
            .`when`().get("http://localhost:8080/")
            .then().statusCode(any(Int::class.java))
            .header("X-Content-Type-Options", "nosniff")
            .header("X-Frame-Options", anyOf(isA(String::class.java)))
    }
}
```

# 10. Эксплуатация и hardening в проде

## Секреты: Vault/KMS, rotation, ограничение доступа к приватным ключам/JWKs

В проде секреты — это не «значения в `application.yml`», а управляемые артефакты с аудитом доступа, сроком жизни и процедурами замены. Минимальный стандарт — секрет-менеджер (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) или KMS (Key Management Service) для шифрования на хранении и в канале. Важно сразу отделить «где храним» от «как доставляем в приложение»: первая задача — операционная, вторая — инженерная.

Доставку секретов лучше реализовывать как «pull при старте и периодический refresh», а не bake-time: так вы сможете отозвать секрет без пересборки образа. В Spring это решается Spring Cloud Vault/Config или SDK провайдера. Для JVM-процессов особенно полезна функция динамических секретов (DB credentials с TTL): сработал ротационный цикл — приложение обновило пул соединений на лету.

Никогда не передавайте секреты через логи, исключения и метрики. Опыт показал, что единственная «всплывшая» строка в логах может стоить инцидента. Включайте маскирование (`logging.pattern.mask`) и проверяйте, что ваши фильтры не печатают тело запроса с токенами. Дополнительная защита — ограничение доступа к логам (RBAC в лог-хранилище).

К приватным ключам подписи (если ваше приложение что-то подписывает: вебхуки, JWT на стороне AS/BFF) доступ должен быть минимальным. В идеале — ключи находятся вне процесса в HSM/KMS; приложение получает «операцию подписи» как сервис. Это устраняет класс уязвимостей «экфильтрация приватника» через чтение ФС/дампы памяти.

Если прямой KMS/HSM недоступен, храните ключ в защищённом секрет-менеджере и грузите его в память процесса как `char[]`/`byte[]`, не как строку и не как файл на диске. Файлы неизбежно оставляют следы в контейнере/слоях образа; даже с `tmpfs` ошибки конфигурации случаются. В Kubernetes — используйте `secret` с tmpfs volume и `fsGroup`/`readOnly`.

Процедуры ротации должны быть отработаны заранее: как вы меняете пароль БД/клиентский секрет OAuth2/приватный ключ, какие сервисы перезапускаете, где обновляете `kid`. Идеально иметь «красную кнопку» — скрипт, который выполняет последовательность шагов без ручных правок.

Права доступа к секретам в секрет-менеджере должны соответствовать принципу минимальных прав: сервис «Orders» видит только «orders/*», SRE — только прод-окружение, разработчик — только dev. Аудит доступа включён по умолчанию; регулярные отчёты помогают засечь аномалии.

При локальной разработке используйте «заглушки»: `.env`/`docker-compose` с пустяковыми значениями, заведомо отличающимися от продовых. Это снижает риск «случайно залить продовый секрет в Git» и дисциплинирует конфигурацию.

Симметричные ключи для HMAC храните и версионируйте как и любые другие секреты: `v1`, `v2` и периодический переход. Симметричные секреты особенно опасны — их нельзя «поделиться частично», как в RSA/EC, поэтому контроль распространения и аудит критичны.

Наконец, продумайте «запрет административного доступа»: единственный путь до секретов — секрет-менеджер. Сборочные артефакты, Docker-образы, Helm values не должны содержать секреты вообще. Любая «временная» вставка секретов в values — это технический долг и потенциальный инцидент.

**Gradle (Groovy) — Spring Cloud Vault**

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-vault-config:4.1.3'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-vault-config:4.1.3")
}
```

**application.yml — подключение к Vault (пример)**

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: kubernetes
      kubernetes:
        role: orders-app
      fail-fast: true
      generic:
        enabled: true
        backend: secret
        default-context: app/orders
```

**Java — загрузка приватного ключа для подписи из секрета (PEM в памяти)**

```java
package com.example.hardening.secrets;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.Base64Utils;

import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;

@Configuration
public class SigningKeyConfig {

    // значение приходит из Vault -> Spring Config: строка BASE64(PKCS8)
    @Bean
    public PrivateKey signingKey(@Value("${signing.pkcs8.base64}") String base64) throws Exception {
        byte[] der = Base64Utils.decodeFromString(base64);
        var spec = new PKCS8EncodedKeySpec(der);
        return KeyFactory.getInstance("RSA").generatePrivate(spec);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.hardening.secrets

import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.security.KeyFactory
import java.security.PrivateKey
import java.security.spec.PKCS8EncodedKeySpec
import java.util.Base64

@Configuration
class SigningKeyConfig {

    @Bean
    fun signingKey(@Value("\${signing.pkcs8.base64}") base64: String): PrivateKey {
        val der = Base64.getDecoder().decode(base64)
        val spec = PKCS8EncodedKeySpec(der)
        return KeyFactory.getInstance("RSA").generatePrivate(spec)
    }
}
```

---

## Политики ключей и ротация: kid, dual-signing, план аварийного перевыпуска

Ключи подписи/шифрования — расходный материал, а не «раз и навсегда». Политика ротации отвечает на вопросы: как часто меняем, как публикуем новые публичные ключи, как обеспечиваем совместимость, как действуем при компрометации. В мире JWT стандартный инструмент — поле заголовка `kid`, позволяющее получать из JWKS правильный ключ.

Dual-signing — приём, когда одновременно существуют «старый» и «новый» ключи: вы публикуете в JWKS оба открытых ключа, ресурс-сервера валидируют подпись любым из них, а сам сервер подписи начинает выпускать токены с новым `kid`. Через оговорённое окно старый ключ удаляется. Это позволяет выполнить безболезненную замену.

Ротация — это не только про подпись JWT. Любые ключи (mTLS, HMAC вебхуков, клиентские секреты OAuth2) должны иметь `notAfter`/TTL и документированный процесс перевыпуска. Важно заранее смоделировать «медленные» клиенты, которые не обновились, — для них понадобится грация и коммуникация.

В кластере приложений переключение на новый ключ должно быть идемпотентным и «атомарным». Выгодно хранить «активный kid» в централизованной конфигурации (Vault/Config Server). Тогда релиз «без даунтайма»: вы заранее раскатали новый ключ, затем переключили `activeKid`, все инстансы сразу подписывают новым.

Если компрометация, план действий агрессивнее: немедленно исключить ключ из JWKS, перевыпустить креды, инвалидировать активные токены (черный список по `jti`/реестр сеансов), уведомить интеграции. Такие runbook-и должны быть написаны до инцидента и проверены на стенде.

Учитывайте кэширование JWKS на ресурс-серверах и в прокси. Если TTL большой, то клиенты могут «долго не увидеть» новый ключ. Nimbus/Okta клиенты по `kid` обычно обновляют автоматически, но это нужно проверить e2e.

Алгоритмы тоже ротируются: переход с `RS256` на `ES256` или обратно — это проект. Dual-publish для разных алгоритмов возможен (два `kid` с разными `alg`), но убедитесь, что потребители поддерживают оба.

Логи и метрики ротации — отдельная тема: заведите счётчик «каким kid подписано N% токенов», чтобы видеть прогресс миграции. Если через сутки всё ещё есть 30% старых — значит, где-то сидит кэш или «вечный токен».

Документируйте timeline ротации: T0 — публикуем новый ключ, T0+1h — начинаем подписывать новым, T0+24h — удаляем старый из JWKS. Для критичных систем сократите окно, но убедитесь в совместимости.

И наконец, разделяйте роли: кто может генерировать ключ, кто может активировать `kid`, кто может удалять старые ключи. Это уменьшает риск ошибочных действий и повышает следуемость процессу.

**Java — два ключа в JWKS и выбор активного по свойству**

```java
package com.example.hardening.keys;

import com.nimbusds.jose.jwk.*;
import com.nimbusds.jose.jwk.gen.RSAKeyGenerator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DualKeyConfig {

    @Bean
    public JWKSet jwkSet() throws Exception {
        RSAKey k1 = new RSAKeyGenerator(2048).keyID("k-2025-09").generate();
        RSAKey k2 = new RSAKeyGenerator(2048).keyID("k-2025-10").generate();
        return new JWKSet(k1.toPublicJWK(), k2.toPublicJWK());
    }

    @Bean
    public RSAKey activeKey(@Value("${signing.active-kid}") String activeKid) throws Exception {
        // В реале — грузим из Vault; здесь генерим для примера
        return new RSAKeyGenerator(2048).keyID(activeKid).generate();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.hardening.keys

import com.nimbusds.jose.jwk.JWKSet
import com.nimbusds.jose.jwk.RSAKey
import com.nimbusds.jose.jwk.gen.RSAKeyGenerator
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class DualKeyConfig {

    @Bean
    fun jwkSet(): JWKSet {
        val k1: RSAKey = RSAKeyGenerator(2048).keyID("k-2025-09").generate()
        val k2: RSAKey = RSAKeyGenerator(2048).keyID("k-2025-10").generate()
        return JWKSet(k1.toPublicJWK(), k2.toPublicJWK())
    }

    @Bean
    fun activeKey(@Value("\${signing.active-kid}") activeKid: String): RSAKey =
        RSAKeyGenerator(2048).keyID(activeKid).generate()
}
```

**application.yml — переключение активного ключа**

```yaml
signing:
  active-kid: k-2025-10
```

---

## Логи/метрики: аномалии входа, 4xx/5xx по безопасности, алерты, rate spikes

Эксплуатация безопасности без наблюдаемости невозможна. В проде нужно видеть «дыхание» системы: распределение 2xx/4xx/5xx, причины 401/403, всплески 429, динамику логинов/отказов, процент JWT-ошибок (`invalid_token`, `expired_token`). Эти сигналы — первые индикаторы атаки или деградации IdP.

Начинайте с общего: включите access-логи на фронтовом прокси/ingress с тегами `user`, `subnet`, `path`, `status`, `duration`, `bytes`. Для приложений — Micrometer + Prometheus/OTel: счётчики `security.auth.success/failure`, `security.authz.decisions{result=...}`, таймеры «время до первой 403».

Не засоряйте логи PII. Отображайте стабильно хэшированный идентификатор субъекта (`hash(sub)`) и tenant, но не email/телефон. Важно, чтобы по логам можно было коррелировать события без раскрытия персональных данных. Маскирование включайте на уровне лог-фреймворка.

Алертинг: простые правила «больше N 401 за M минут» дают шум. Добавляйте агрегаты: «рост 401 на 300% по сравнению с медианой часа», «всплеск 5xx на `/oauth2/`», «аномальный процент `invalid_token` от одного IP». Такие алерты устойчивее и отражают сдвиги.

Свяжите события приложения и IdP: при падении токен-эндпоинта вы увидите лавину 401/403. В метриках держите отдельный счётчик ошибок интеграции с IdP (timeouts, 5xx), чтобы различать «мы не принимаем токены» и «клиенты не могут получить токены».

Сырые 5xx по безопасности — тревожны: ошибка в валидаторе JWT, падение конвертера ролей, NPE в `AuthorizationManager`. Такие ошибки должны сразу вызывать пейджер — это дыра, через которую можно либо обойти проверку, либо положить сервис.

Полезно писать «решение авторизации» в DEBUG на стейджинге: кто зашёл на какой путь, какой набор authorities, почему отказ (какое правило). В проде эту детализацию выключайте или редактируйте (сэмплинг), иначе лог-объём взлетит.

Для rate spikes держите счётчики на шлюзе: «сколько запросов в минуту по пути/клиенту». Быстрый «режим защиты» — временный глобальный лимит или «снижение» от одного клиента. Эти действия должны быть процедурно оформлены: кто включает, что меняется, как выводим из режима.

Поддерживайте «диагностические» эндпоинты в закрытой зоне: `/actuator/metrics`, `/actuator/health/*` с правами `OPS`. Это помогает SRE понимать ситуацию без SSH на хосты. Но эти эндпоинты нельзя выставлять наружу — только через VPN/бастион.

И последнее — тренируйте инцидент-респонс: симулируйте рост 401, дроп JWKS, отвал KMS. Команда должна за минуты понять, что происходит, и выполнить плейбук. Метрики и логи — ваши глаза; если они молчат или перегружены мусором, у вас нет шансов.

**Java — фильтр для учёта 4xx/5xx и обогащения MDC**

```java
package com.example.hardening.obs;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import io.micrometer.core.instrument.MeterRegistry;

@Component
public class SecurityObsFilter extends OncePerRequestFilter {

    private final MeterRegistry registry;

    public SecurityObsFilter(MeterRegistry registry) { this.registry = registry; }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, java.io.IOException {
        try {
            MDC.put("path", req.getRequestURI());
            MDC.put("method", req.getMethod());
            chain.doFilter(req, res);
        } finally {
            int sc = res.getStatus();
            if (sc >= 400 && sc < 600) {
                registry.counter("http.responses", "status", Integer.toString(sc), "path", req.getRequestURI()).increment();
            }
            MDC.clear();
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.hardening.obs

import io.micrometer.core.instrument.MeterRegistry
import jakarta.servlet.FilterChain
import jakarta.servlet.ServletException
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.MDC
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import java.io.IOException

@Component
class SecurityObsFilter(private val registry: MeterRegistry) : OncePerRequestFilter() {

    @Throws(ServletException::class, IOException::class)
    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        try {
            MDC.put("path", req.requestURI)
            MDC.put("method", req.method)
            chain.doFilter(req, res)
        } finally {
            val sc = res.status
            if (sc in 400..599) {
                registry.counter("http.responses", "status", sc.toString(), "path", req.requestURI).increment()
            }
            MDC.clear()
        }
    }
}
```

---

## Безопасная публикация Swagger/Actuator: IP-allowlist, auth, выключение в проде

Swagger/OpenAPI — отличный DX-инструмент, но в проде он становится картой ваших эндпоинтов для злоумышленника. Базовое правило: не публиковать UI наружу, а JSON — только за аутентификацией и/или по allowlist IP. Внутренним пользователям — VPN/бастион и доступ через портал разработчика.

Actuator — инструмент SRE, а не пользователей. В проде часто достаточно `health` и `info` наружу, а все остальные — за аутентификацией на отдельном management-порте, привязанном к приватной сети/namespace. Плюс — ограничение по IP и роль `OPS`.

Профили помогают держать разные политики по окружениям: в `dev` Swagger-UI разрешён, в `stage` — только для VPN, в `prod` — выключен UI, JSON доступен по токену и только для списка адресов. Такой подход легко документируется и масштабируется.

Не забывайте про CORS: Swagger-UI — это браузерный SPA, он может упереться в CORS при обращении к API. На внутренних стендах разрешайте `Origin` портала документации, а не `*`. Это предотвратит утечки через сторонние страницы.

Открытые actuator-метрики/эндпоинты — источник-разведка. `env`, `configprops`, `threaddump` не должны быть вообще доступны без строгой аутентификации. Если нужен временный доступ — поднимите отдельный тул-сервис в приватной сети.

С точки зрения кодовой базы удобнее задать отдельную `SecurityFilterChain` для `/v3/api-docs/**`, `/swagger-ui/**` и `/actuator/**`. Тогда вы точно контролируете «кто видит что» и не смешиваете правила с основным API.

Для IP-allowlist используйте кастомный `AuthorizationManager`, читающий заголовок `X-Forwarded-For` и сравнивающий с белым списком. Важно учитывать, что true IP может скрываться за балансировщиком — доставайте «правильный» адрес только из доверенного заголовка.

При публикации API JSON добавьте банальные, но важные HTTP-заголовки: `Cache-Control: no-store`, чтобы прокси/браузеры не кешировали разметку. Это не панацея, но снижает риск случайной утечки.

Регулярно прогоняйте сканеры на внешнем периметре: /actuator «не должен открыться случайно» после обновления Boot. Линт-тест на CI, проверяющий 404/401/403 для запрещённых путей, предотвратит неприятные сюрпризы.

И наконец, обучайте команду: Swagger — для локалки/стендов, в проде — только JSON и по правилам. Единый плейбук снижает вероятность «давайте включим на часок…» в пятницу вечером.

**Gradle (Groovy) — springdoc-openapi**

```groovy
dependencies {
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencies {
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
}
```

**Java — цепочки безопасности и IP-allowlist для Swagger/Actuator**

```java
package com.example.hardening.docs;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.context.annotation.*;
import org.springframework.core.annotation.Order;
import org.springframework.security.authorization.*;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Set;

@Configuration
public class DocsSecurityConfig {

    private static final Set<String> ALLOW = Set.of("10.0.0.0/8", "192.168.0.0/16", "127.0.0.1/32");

    private AuthorizationManager<HttpServletRequest> ipAllowlist() {
        return (authentication, request) -> {
            String ip = request.getRemoteAddr(); // в проде — парсить X-Forwarded-For от доверенного прокси
            boolean ok = ip.startsWith("10.") || ip.startsWith("192.168.") || "127.0.0.1".equals(ip);
            return new AuthorizationDecision(ok);
        };
    }

    @Bean @Order(1)
    SecurityFilterChain actuator(HttpSecurity http) throws Exception {
        http.securityMatcher("/actuator/**")
            .authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .anyRequest().access(ipAllowlist()))
            .httpBasic(b -> {})
            .csrf(cs -> cs.disable());
        return http.build();
    }

    @Bean @Order(2)
    SecurityFilterChain swagger(HttpSecurity http) throws Exception {
        http.securityMatcher("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html")
            .authorizeHttpRequests(a -> a.anyRequest().access(ipAllowlist()))
            .csrf(cs -> cs.disable());
        return http.build();
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.hardening.docs

import jakarta.servlet.http.HttpServletRequest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.annotation.Order
import org.springframework.security.authorization.AuthorizationDecision
import org.springframework.security.authorization.AuthorizationManager
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.web.SecurityFilterChain

@Configuration
class DocsSecurityConfig {

    private fun ipAllowlist(): AuthorizationManager<HttpServletRequest> =
        AuthorizationManager { _, request ->
            val ip = request.remoteAddr // в проде — доверенный X-Forwarded-For
            val ok = ip.startsWith("10.") || ip.startsWith("192.168.") || ip == "127.0.0.1"
            AuthorizationDecision(ok)
        }

    @Bean @Order(1)
    fun actuator(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/actuator/**")
            .authorizeHttpRequests {
                it.requestMatchers("/actuator/health", "/actuator/info").permitAll()
                    .anyRequest().access(ipAllowlist())
            }
            .httpBasic { }
            .csrf { it.disable() }
        return http.build()
    }

    @Bean @Order(2)
    fun swagger(http: HttpSecurity): SecurityFilterChain {
        http.securityMatcher("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html")
            .authorizeHttpRequests { it.anyRequest().access(ipAllowlist()) }
            .csrf { it.disable() }
        return http.build()
    }
}
```

**application.yml — отключить UI в проде**

```yaml
spring:
  profiles: active: prod

---
spring:
  config:
    activate:
      on-profile: prod
springdoc:
  swagger-ui:
    enabled: false
```

---

## Supply-chain: dependency pinning, репозитории артефактов, SBOM

Современные инциденты часто приходят «через зависимость». Управление цепочкой поставки — это дисциплина: закрепить версии (pinning), собирать из доверенных источников, генерировать SBOM (список компонентов) и автоматизировать аудит уязвимостей.

Dependency pinning в Gradle решается lock-файлами/версионными каталогами: вы фиксируете точные версии и обновляете их осознанно. Это гарантирует, что билд на CI сегодня и через неделю тянет одинаковые артефакты. Для мультипроектов это практически обязательно.

Репозитории артефактов (Nexus/Artifactory) — ваш буфер: проксируйте Maven Central, но кэшируйте и подписывайте скачанное. Это защищает от «supply-chain outage» и даёт контроль над зависимостями. На уровне Gradle отключайте HTTP-репозитории, разрешайте только HTTPS и свой прокси.

SBOM (CycloneDX/SPDX) — артефакт, который вы публикуете вместе с релизом. Он нужен SRE/безопасности, чтобы быстро ответить «а у нас есть `foo:1.2.3`?» в день, когда выходит CVE. Генерация — плагином, доставка — как артефакт сборки.

Подписанные релизы и проверка контрольных сумм зависят от экосистемы, но для бинарей/нативных либ это must-have. Если вы что-то скачиваете на этапе запуска (например, модели/правила) — проверяйте SHA-256/подпись перед использованием.

CI должен иметь отдельную роль и доступ к «read-only» секретам. Нельзя собирать артефакты с дев-машин и пушить в прод-репозиторий. Artefact provenance (SLSA) и «двухчленная схема» публикации — хороший стандарт.

Автоматические сканеры (OWASP DC, Snyk, osv-scanner, Trivy) запускайте и на PR, и по расписанию. Важно договариваться «что делать с красными флагами»: обновить/запинать/заменить. Молчаливые отчёты никого не спасают.

Сторонние плагины Gradle — тоже supply chain. Минимизируйте их список и проверяйте популярность/историю. Используйте версии из lock-файла. «Свежий плагин от неизвестного автора» — риск.

Документируйте процесс обновления зависимостей: кто аппрувит, какие тесты обязательны, какие стенды. Это снимает риск «обновили в пятницу — прод встал».

И, наконец, воспроизводимость — не только версии, но и параметры компиляции/окружения. Фиксируйте toolchain (Java 17), включайте `reproducible` флаги, публикуйте checksums артефактов.

**Gradle (Groovy) — dependency locking + CycloneDX SBOM**

```groovy
dependencyLocking {
    lockAllConfigurations()
}
tasks.register('writeLocks') {
    doLast { println "Run: ./gradlew dependencies --write-locks" }
}
plugins {
    id 'org.cyclonedx.bom' version '1.10.0'
}
cyclonedxBom {
    includeConfigs = ['runtimeClasspath']
    projectType = 'application'
}
```

**Gradle (Kotlin DSL)**

```kotlin
dependencyLocking { lockAllConfigurations() }
tasks.register("writeLocks") { doLast { println("Run: ./gradlew dependencies --write-locks") } }
plugins { id("org.cyclonedx.bom") version "1.10.0" }
cyclonedxBom {
    includeConfigs.set(listOf("runtimeClasspath"))
    projectType.set("application")
}
```

**Java — проверка SHA-256 скачанного файла перед использованием**

```java
package com.example.hardening.supply;

import java.io.InputStream;
import java.net.URL;
import java.nio.file.*;
import java.security.MessageDigest;
import java.util.HexFormat;

public class ChecksumDownload {

    public static void downloadAndVerify(String url, String expectedSha256, Path target) throws Exception {
        try (InputStream in = new URL(url).openStream()) {
            Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
        }
        byte[] bytes = Files.readAllBytes(target);
        byte[] hash = MessageDigest.getInstance("SHA-256").digest(bytes);
        String got = HexFormat.of().formatHex(hash);
        if (!got.equalsIgnoreCase(expectedSha256)) {
            Files.deleteIfExists(target);
            throw new IllegalStateException("Checksum mismatch");
        }
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.hardening.supply

import java.net.URL
import java.nio.file.Files
import java.nio.file.Path
import java.nio.file.StandardCopyOption
import java.security.MessageDigest

object ChecksumDownload {
    @JvmStatic
    fun downloadAndVerify(url: String, expectedSha256: String, target: Path) {
        URL(url).openStream().use { input ->
            Files.copy(input, target, StandardCopyOption.REPLACE_EXISTING)
        }
        val bytes = Files.readAllBytes(target)
        val got = MessageDigest.getInstance("SHA-256").digest(bytes).joinToString("") { "%02x".format(it) }
        if (!got.equals(expectedSha256, ignoreCase = true)) {
            Files.deleteIfExists(target)
            error("Checksum mismatch")
        }
    }
}
```

---

## Чек-листы релиза: что проверить перед выкладкой (headers, CORS, endpoints, roles)

Чек-лист — самый дешёвый и надёжный инструмент hardening’а. Перед релизом вы хотите увериться, что конфигурация безопасности соответствует политике: заголовки включены, CORS не «на всё», закрытые эндпоинты действительно закрыты, роли не «размякли», секреты не протекли в логи.

Пункт «заголовки»: проверить HSTS (включен, `includeSubDomains`, срок), X-Content-Type-Options, X-Frame-Options (или `frame-ancestors` в CSP), базовую CSP без `unsafe-inline` (или с `nonce`). Это можно автоматизировать smoke-тестом по нескольким путям (200/401/403).

Пункт «CORS»: разрешены конкретные origins, нет `*` вместе с `credentials`, preflight отвечает корректно, `Vary: Origin` на месте. Для фронта — прописан точный список методов и заголовков, которые реально используются.

Пункт «эндпоинты»: /actuator исключительно то, что разрешено; /swagger-ui выключен в проде; /v3/api-docs за auth или allowlist. Никаких отладочных/внутренних контроллеров (health-probes на месте). Хорошая практика — автогенерить список маппингов и сравнивать с «золотым» эталоном.

Пункт «роли/правила»: ключевые операции защищены нужными `ROLE_`/`SCOPE_`, SpEL выражения работают как задумано, на «админские» пути есть явные правила deny/allow. Тесты «таблица истинности» зелёные.

Пункт «секреты»: переменные окружения на стенде/в CI корректные, нет секретов в образе/values, токены доступа в логи не печатаются. Включены политики ротации и аудит в секрет-менеджере.

Пункт «наблюдаемость»: метрики и алерты сконфигурированы, дешборды обновлены под новый релиз, журнал аудита пишет ключевые события (вход/выход/отказ). На стенде «день релиза» алерты не должны молчать.

Пункт «JWT/JWKS»: ресурс-сервер видит JWKS, kid актуальный, clock skew разумный, аудитория совпадает. Если есть ротация — прогоните «новый kid» на стейдже, убедитесь, что старые токены принимаются.

Пункт «gateway»: CORS/headers/ratelimit на периметре в соответствии с политикой, токены корректно прокидываются, 401/403 унифицированы, `WWW-Authenticate: Bearer` присутствует.

Пункт «инцидент-процедуры»: плейбуки актуальны, «красные кнопки» работают (откат ключа, выключение Swagger, глобальный rate limit), ответственная смена назначена, коммуникационный план на случай деградации готов.

И вишенка — автоматизируйте этот чек-лист. Пусть приложение само при старте выполнит ряд самопроверок и напишет в логи «зелёные/красные» пункты. Это не заменяет тесты, но ловит конфигурационные промахи ещё до приёма трафика.

**Java — самопроверка при старте (headers/CORS/endpoints)**

```java
package com.example.hardening.checks;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Component
public class SecuritySelfCheck implements CommandLineRunner {
    private static final Logger log = LoggerFactory.getLogger(SecuritySelfCheck.class);
    private final Environment env;

    public SecuritySelfCheck(Environment env) { this.env = env; }

    @Override
    public void run(String... args) {
        String[] profiles = env.getActiveProfiles();
        log.info("Profiles: {}", Arrays.toString(profiles));

        boolean swaggerUiEnabled = env.getProperty("springdoc.swagger-ui.enabled", Boolean.class, true);
        if (Arrays.asList(profiles).contains("prod") && swaggerUiEnabled) {
            log.error("SWAGGER UI ENABLED IN PROD — should be disabled");
        } else {
            log.info("Swagger-UI policy OK");
        }

        String allowedOrigins = env.getProperty("app.cors.allowed-origins", "");
        if (allowedOrigins.contains("*") && env.getProperty("app.cors.allow-credentials", Boolean.class, true)) {
            log.error("CORS misconfig: '*' with credentials=true");
        } else {
            log.info("CORS policy OK");
        }

        String hsts = env.getProperty("server.hsts.enabled", "true");
        log.info("HSTS enabled: {}", hsts);
    }
}
```

**Kotlin — эквивалент**

```kotlin
package com.example.hardening.checks

import org.slf4j.LoggerFactory
import org.springframework.boot.CommandLineRunner
import org.springframework.core.env.Environment
import org.springframework.stereotype.Component

@Component
class SecuritySelfCheck(private val env: Environment) : CommandLineRunner {

    private val log = LoggerFactory.getLogger(SecuritySelfCheck::class.java)

    override fun run(vararg args: String?) {
        val profiles = env.activeProfiles.toList()
        log.info("Profiles: {}", profiles)

        val swaggerUiEnabled = env.getProperty("springdoc.swagger-ui.enabled", Boolean::class.java, true)!!
        if (profiles.contains("prod") && swaggerUiEnabled) {
            log.error("SWAGGER UI ENABLED IN PROD — should be disabled")
        } else {
            log.info("Swagger-UI policy OK")
        }

        val allowedOrigins = env.getProperty("app.cors.allowed-origins", "")
        val allowCreds = env.getProperty("app.cors.allow-credentials", Boolean::class.java, true)!!
        if (allowedOrigins.contains("*") && allowCreds) {
            log.error("CORS misconfig: '*' with credentials=true")
        } else {
            log.info("CORS policy OK")
        }

        val hsts = env.getProperty("server.hsts.enabled", "true")
        log.info("HSTS enabled: {}", hsts)
    }
}
```

**application.yml — пример параметров для самопроверки**

```yaml
app:
  cors:
    allowed-origins: "https://app.example.com"
    allow-credentials: true
server:
  hsts:
    enabled: true
```

