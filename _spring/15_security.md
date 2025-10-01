---
layout: page
title: "Spring Security: безопасность приложения"
permalink: /spring/security
---

# 0. Основы

## Что такое Spring Security

**Spring Security** — это фреймворк безопасности для приложений на Spring, который реализует механизмы **аутентификации** (кто вы) и **авторизации** (что вам можно). Он работает как набор **фильтров** (Servlet Filter API) и бинов, перехватывающих входящие HTTP-запросы и решения о доступе. Библиотека поддерживает Basic/Form Login, JWT, OAuth2/OIDC, SAML, LDAP, mTLS и др., а также декларативные аннотации на методах.

Ключевая идея: вынести безопасность в **инфраструктуру** (фильтры и конфигурацию) и описывать правила декларативно. Это снимает «раскисание» проверок по коду и создаёт один «точный» слой контроля доступа. Плюс готовые реализации шифрования, паролей, «соли», кэширования пользователей и т. п.

Важный термин — **SecurityFilterChain**: цепочка фильтров, через которую проходят все (или часть) запросов. Её создаёт ваш `@Bean`, используя **DSL `HttpSecurity`** — декларативный билдер, где вы пишете «что разрешено и как аутентифицируемся». Ещё один термин — **AuthenticationManager**: компонент, который «проверяет» учётные данные и возвращает **Authentication** (успешного пользователя) или бросает исключение.

Минимальный пример (полностью закрытое API, только Basic auth):

```java
@Configuration
class SecurityConfig {

  @Bean
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable()); // для чистого API (stateless) обычно выключают
    return http.build();
  }
}
```

## Задачи и зачем применяется

Первая задача — **идентификация** клиента: по паролю, токену, сертификату и т. д. В терминах Spring Security это «получить `Authentication`». Вторая — **принятие решения о доступе**: есть ли у пользователя нужные **`GrantedAuthority`** (привилегии/роли) для целевого ресурса/метода. Третья — **безопасные заголовки** (HSTS, CSP, X-Content-Type-Options), защита от CSRF там, где есть сессионные формы, и корректная реакция на ошибки (401/403).

На уровне процессов Spring Security помогает выстроить **RBAC** (role-based access control) и, при необходимости, **ABAC** (attribute-based: решения зависят от атрибутов пользователя/объекта — отдел, владелец, tenant). Добавьте сюда **аудит** (кто, когда, к чему обращался) и **наблюдаемость** (корреляция запросов) — безопасность становится управляемой.

Практическая польза — в повторяемости и проверяемости. Файлы конфигураций (Java DSL) проходят код-ревью, правила меняются централизованно, а **Spring Security Test** позволяет писать читаемые интеграционные тесты: `with(jwt().authorities(...))` или `@WithMockUser(roles="ADMIN")`.

Наконец, фреймворк — это ещё и **интеграции**: ресурс-сервер с JWT (валидирует подпись по JWKs), OAuth2-клиент (логин через внешнего провайдера), Authorization Server (если вы делаете свой IdP), LDAP/AD и mTLS на периметре.

## Основные элементы и понятия в Spring Security

**`Authentication`** — «удостоверение» текущего пользователя: имя (`getName()`), набор привилегий (`getAuthorities()`), флаг `isAuthenticated()`. До логина в контексте лежит **`AnonymousAuthenticationToken`** (гость). После успешной аутентификации внедряется конкретная реализация (например, `UsernamePasswordAuthenticationToken`).

**`Principal`** — «кто» аутентифицирован (часто то же самое, что `Authentication#getPrincipal()`: `UserDetails`, Map с claims, т. п.). **`GrantedAuthority`** — атомарная привилегия/роль, строка вроде `ROLE_ADMIN` или `SCOPE_orders.read`. Сами строки вы определяете политикой безопасности.

**`SecurityContext` / `SecurityContextHolder`** — хранилище `Authentication` на время обработки запроса (ThreadLocal по умолчанию). Его наполняет фильтр аутентификации и очищает фильтр завершения запроса. Получить текущего пользователя можно через `SecurityContextHolder.getContext().getAuthentication()` или инжектировав `Authentication` в аргументы контроллера.

Пример использования в контроллере (MVC):

```java
@RestController
class MeController {

  @GetMapping("/me")
  public Map<String, Object> me(Authentication auth) {
    return Map.of("name", auth.getName(),
                  "authorities", auth.getAuthorities().toString());
  }
}
```

---

# 1. Архитектура и базовые понятия

## SecurityFilterChain, DelegatingFilterProxy, порядок фильтров

**`DelegatingFilterProxy`** — Servlet-фильтр, который делегирует вызовы Spring-бину `FilterChainProxy`. Это «мост» между контейнером сервлетов (Tomcat/Jetty) и Spring Security. `FilterChainProxy` уже управляет набором **`SecurityFilterChain`** — каждой цепочкой сопоставлен `RequestMatcher` (какие запросы она обрабатывает) и список фильтров (аутентификация, авторизация, заголовки и т. п.).

Порядок фильтров критичен. Например, **`SecurityContextHolderFilter`** должен выполниться до авторизации, чтобы контекст пользователя был доступен; **`UsernamePasswordAuthenticationFilter`** обрабатывает логин формы; **`BearerTokenAuthenticationFilter`** извлекает JWT из заголовка `Authorization`. Spring Security поставляет правильно отсортированный пайплайн, а вы можете вставлять свои фильтры через `addFilterBefore/After`.

Обычно вам не нужно вручную регистрировать `DelegatingFilterProxy` — Spring Boot делает это автоматически. Вы управляете **цепочками** через `@Bean SecurityFilterChain` и DSL `HttpSecurity`. Если необходимо обслуживать разные сегменты по разным правилам (например, `/public/**` и `/api/**`), объявляете несколько `SecurityFilterChain` с разными matchers и **приоритетом** (`@Order` или `.securityMatcher` с «более специфичный первым»).

Пример вставки кастомного фильтра до Basic Auth:

```java
@Bean
SecurityFilterChain api(HttpSecurity http, RequestLoggingFilter logging) throws Exception {
  http.securityMatcher("/api/**")
      .addFilterBefore(logging, org.springframework.security.web.authentication.www.BasicAuthenticationFilter.class)
      .httpBasic(Customizer.withDefaults())
      .authorizeHttpRequests(a -> a.anyRequest().authenticated())
      .csrf(csrf -> csrf.disable());
  return http.build();
}

@Component
class RequestLoggingFilter extends OncePerRequestFilter {
  @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    // логируем нужные поля, но ничего не ломаем
    chain.doFilter(req, res);
  }
}
```

## Authentication, Principal, GrantedAuthority, SecurityContext/Holder

`Authentication` создаётся **AuthenticationProvider’ом** — компонентом, который умеет «проверять» определённый тип учётных данных (пароль, токен, сертификат). После успеха в `Authentication` попадают `principal` и набор `GrantedAuthority` (например, роли). Эти «авторитеты» используйте для решений «пускать/не пускать».

`SecurityContextHolder` хранит текущий `SecurityContext`. По умолчанию стратегия — **`MODE_THREADLOCAL`** (на каждый поток), но доступна **`MODE_INHERITABLETHREADLOCAL`** (наследовать в дочерние потоки). В реактивных приложениях (WebFlux) используется **Reactor Context**; там нельзя читать через `SecurityContextHolder` напрямую, зато есть `ReactiveSecurityContextHolder.getContext()`.

В коде можно инжектировать `Authentication` прямо в метод контроллера, сервис или `@MessageMapping` (WebSocket). Также доступны аннотации `@AuthenticationPrincipal` (дают удобную проекцию `principal` в целевой тип) и `@CurrentSecurityContext` (инжект контекста).

Пример: инжекция principal и проверка прав «вручную»:

```java
@GetMapping("/admin/report")
public String report(@AuthenticationPrincipal UserDetails user, Authentication auth) {
  boolean isAdmin = auth.getAuthorities().stream()
      .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
  if (!isAdmin) throw new AccessDeniedException("Admins only");
  return "secret";
}
```

## Разграничение аутентификации и авторизации, точки входа и обработчики ошибок

**Аутентификация** отвечает за «кто вы» (пароль, токен, сертификат). Её результат — `Authentication`. **Авторизация** — «что вам можно»; решение принимается на основе `GrantedAuthority` и правил, которые вы описали в `HttpSecurity` или метод-аннотациях (`@PreAuthorize`, см. ниже).

Когда неаутентифицированный пользователь идёт к защищённому ресурсу, срабатывает **`AuthenticationEntryPoint`** — «точка входа» в аутентификацию. Для HTML-приложений это редирект на страницу логина, для API — отправка `401 Unauthorized`. Когда пользователь аутентифицирован, но прав не хватает, вызывается **`AccessDeniedHandler`** — отвечает `403 Forbidden`.

Оба обработчика настраиваются в `exceptionHandling`. В современных API удобно отдавать ошибки в формате **RFC-7807** (Problem Details). Ниже — пример кастомизации под JSON-ошибки для REST:

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
  http.exceptionHandling(ex -> ex
      .authenticationEntryPoint((req, res, e) -> {
        res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        res.setContentType("application/problem+json");
        res.getWriter().write("""
          {"type":"about:blank","title":"Unauthorized","status":401}
        """);
      })
      .accessDeniedHandler((req, res, e) -> {
        res.setStatus(HttpServletResponse.SC_FORBIDDEN);
        res.setContentType("application/problem+json");
        res.getWriter().write("""
          {"type":"about:blank","title":"Forbidden","status":403}
        """);
      })
  );
  // ...
  return http.build();
}
```

## Поток запроса: от входа в фильтры до принятия AccessDecision

Поток таков: запрос попадает в **`DelegatingFilterProxy`** → прокидывается в `FilterChainProxy` → выбирается подходящая **SecurityFilterChain** (по `RequestMatcher`). Ранние фильтры восстанавливают контекст (`SecurityContextHolderFilter`), затем фильтры аутентификации пытаются извлечь креденшлы (Basic, Bearer, Form) и вызвать `AuthenticationManager`. При успехе в контекст ложится `Authentication`.

Далее работает **авторизация**: `AuthorizationFilter` / `FilterSecurityInterceptor` проверяет правило (`authorizeHttpRequests`) и вызывает **AccessDecisionManager**. Последний собирает вердикты от набора **Voter’ов** (голосующих), например `RoleVoter` (по ролям), `AuthenticatedVoter` (требование «быть аутентифицированным»). Решение зависит от стратегии (affirmative/consensus/unanimous).

Если правило на уровне URL пропускает, то дальше может включиться **метод-безопасность** — аннотации `@PreAuthorize/@PostAuthorize` (см. раздел 4 ниже). Для них используется `MethodSecurityInterceptor` и собственные Voter’ы/SpEL-выражения.

Если где-то по пути аутентификация/авторизация проваливается, управление передаётся `AuthenticationEntryPoint` (401) или `AccessDeniedHandler` (403). Успех → контроллер/хэндлер, откуда вы отдаёте ответ.

---

# 2. Современная конфигурация в Spring Boot 3.x / Security 6.x

## Отказ от WebSecurityConfigurerAdapter: bean SecurityFilterChain + HttpSecurity DSL

Начиная с Security 5.7+ и Spring Boot 3, класс **`WebSecurityConfigurerAdapter`** удалён. Вместо него вы объявляете **`@Bean SecurityFilterChain`** и настраиваете всё через **DSL `HttpSecurity`**. Это делает конфигурации явными и модульными (несколько цепочек, чёткие `@Bean`).

`HttpSecurity` — билдер, у которого вы включаете нужные фичи: `.httpBasic()`, `.formLogin()`, `.oauth2ResourceServer().jwt()`, `.authorizeHttpRequests()`, `.csrf()`, `.cors()` и т. п. Порядок вызовов **не** задаёт порядок фильтров — он формируется самим фреймворком корректно; вы лишь включаете/выключаете функциональные блоки.

Если нужно несколько разных правил для разных участков приложения (например, `/actuator/**`, `/assets/**`, `/api/**`), просто объявляйте **несколько** методов `@Bean SecurityFilterChain` и задавайте **matcher** (через `.securityMatcher` или `requestMatcher`). Чем специфичнее matcher — тем раньше его надо проверять (используйте `@Order`).

Минимальная заготовка:

```java
@Configuration
@EnableMethodSecurity // включает @PreAuthorize/@PostAuthorize/и т.д.
class SecurityConfig {

  @Bean
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable());
    return http.build();
  }
}
```

## Конфигурация статуса сессии, CORS/CSRF, HSTS, headers, exceptionHandling

**Сессии**: для REST обычно **stateless** — `sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))`. Так `SecurityContext` не сохраняется в HttpSession, нет риска фиксации сессии, и горизонтальное масштабирование проще. Для UI/форм — `IF_REQUIRED`/`ALWAYS` (по ситуации).

**CSRF** (межсайтовая подделка запросов) нужен, когда у вас **cookie-сессии и браузерный UI**. Для чистого REST на токенах — обычно **выключают** (`csrf.disable()`), а защиту берут на себя bearer-токены и CORS. **CORS** (Cross-Origin Resource Sharing) включают через `cors()` + `CorsConfigurationSource` бин — обязательно описать `allowedOrigins`, методы, заголовки.

**HSTS** (HTTP Strict Transport Security) — обязательный заголовок на проде при HTTPS: заставляет браузер всегда использовать HTTPS. Настраивается через `headers(h -> h.httpStrictTransportSecurity(Customizer.withDefaults()))`. Туда же — `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`.

Пример «базово-безопасной» конфигурации для API:

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
  http.securityMatcher("/api/**")
      .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .cors(Customizer.withDefaults())
      .csrf(csrf -> csrf.disable())
      .headers(h -> h
          .httpStrictTransportSecurity(Customizer.withDefaults())
          .contentTypeOptions(Customizer.withDefaults())
          .frameOptions(f -> f.sameOrigin()))
      .exceptionHandling(ex -> ex
          .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
          .accessDeniedHandler(new AccessDeniedHandlerImpl()))
      .authorizeHttpRequests(a -> a.anyRequest().authenticated())
      .httpBasic(Customizer.withDefaults());
  return http.build();
}
```

## Разделение конфигураций на несколько цепочек (ant/request matcher)

Когда правила различаются для разных зон (статические ресурсы, actuator, публичные страницы, API), объявляйте **несколько цепочек**. В Security 6 вместо `antMatchers` в DSL авторизации используют `requestMatchers` (а для **выбора цепочки** — `securityMatcher`). Важно: «более специфичные» цепочки должны проверяться раньше — используйте `@Order`.

Пример: статике разрешаем всё, actuator защищаем Basic’ом, API — JWT-ресурс-сервер:

```java
@Configuration
class ChainsConfig {

  @Bean @Order(1)
  SecurityFilterChain staticChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/assets/**", "/favicon.ico")
        .authorizeHttpRequests(a -> a.anyRequest().permitAll());
    return http.build();
  }

  @Bean @Order(2)
  SecurityFilterChain actuatorChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/actuator/**")
        .authorizeHttpRequests(a -> a.requestMatchers("/actuator/health").permitAll()
                                     .anyRequest().hasRole("OPS"))
        .httpBasic(Customizer.withDefaults());
    return http.build();
  }

  @Bean @Order(3)
  SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .csrf(csrf -> csrf.disable())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    return http.build();
  }
}
```

## Профили и модульность: dev vs prod, переиспользуемые фрагменты

Часто нужно смягчить правила в **dev** (логин попроще, выключить HSTS, открыть Swagger) и ужесточить в **prod**. Вынесите конфигурации в разные классы с `@Profile("dev")` / `@Profile("prod")`. Общие части (например, заголовки или CORS) — в отдельные `@Bean`, которые импортируются из обоих.

Ещё приём — собрать «фрагменты» конфигурации как методы-хелперы или отдельные конфигурационные классы: `CommonSecurity`, `ApiSecurity`, `UiSecurity`. Так легче поддерживать несколько приложений с одинаковыми политиками.

Пример разделения профилей:

```java
@Configuration
@Profile("dev")
class DevSecurity {
  @Bean SecurityFilterChain devChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(a -> a.requestMatchers("/swagger-ui/**","/v3/api-docs/**").permitAll()
                                     .anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable());
    return http.build();
  }
}

@Configuration
@Profile("prod")
class ProdSecurity {
  @Bean SecurityFilterChain prodChain(HttpSecurity http) throws Exception {
    http.headers(h -> h.httpStrictTransportSecurity(Customizer.withDefaults()))
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .csrf(csrf -> csrf.disable())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    return http.build();
  }
}
```

---

# 3. Аутентификация: источники и провайдеры

## DaoAuthenticationProvider, PasswordEncoder (BCrypt/Argon2), UserDetailsService

**`UserDetailsService`** — интерфейс, который по username возвращает `UserDetails` (пользователь, пароли, роли). **`DaoAuthenticationProvider`** — провайдер аутентификации, который использует `UserDetailsService` + `PasswordEncoder` для проверки пароля. **`PasswordEncoder`** — контракт хеширования паролей (например, `BCryptPasswordEncoder`, `Argon2PasswordEncoder`).

BCrypt — дефолтный и безопасный вариант для большинства случаев (адаптивная сложность `strength`). Argon2 — современный победитель Password Hashing Competition, хорошо борется с GPU-атаками (имеет параметры памяти/итераций). Главное — **никогда не хранить пароли в открытом виде** и не «сравнивать вручную».

Пример конфигурации DAO-провайдера:

```java
@Configuration
class AuthConfig {

  @Bean
  PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // cost factor
  }

  @Bean
  UserDetailsService users(PasswordEncoder enc) {
    var admin = User.withUsername("admin")
        .password(enc.encode("s3cr3t"))
        .roles("ADMIN")
        .build();
    var user = User.withUsername("user")
        .password(enc.encode("password"))
        .roles("USER")
        .build();
    return new InMemoryUserDetailsManager(admin, user);
  }

  // AuthenticationManager строится автоматически из провайдеров
}
```

В проде чаще `UserDetailsService` обращается к БД/LDAP. Не забывайте «соль» (в BCrypt встроена) и ротацию паролей. Для миграций паролей можно использовать **DelegatingPasswordEncoder** — он поддерживает несколько схем (`{bcrypt}`, `{argon2}`) и позволит «перехешировать» постепенно.

## Form Login/HTTP Basic/Bearer, remember-me, ограничение попыток логина

**Form Login** — классический HTML-логин: `http.formLogin()`. Подходит для серверных UI или если SPA делает редирект на вашу форму. **HTTP Basic** — простая схема с заголовком `Authorization: Basic ...`; удобно для админок/actuator, но только поверх HTTPS. **Bearer** — заголовок с токеном (обычно JWT), включается через `oauth2ResourceServer().jwt()`.

**Remember-me** — механика «долгой» сессии через cookie с подписью. Включается `http.rememberMe()`. Безопаснее использовать **PersistentTokenRepository** (таблица токенов) и ротировать их при каждом логине. Помните про **SameSite**/`Secure`/`HttpOnly` флаги для cookie.

Ограничение попыток логина (anti-bruteforce) можно реализовать фильтром/сервисом, который считает попытки по IP/username и временно блокирует после N неудач. Библиотеки вроде Bucket4j удобны, но и простая карта с TTL подойдёт для старта. Важна метрика и алертинг.

```java
@Bean
SecurityFilterChain ui(HttpSecurity http) throws Exception {
  http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
      .formLogin(form -> form.loginPage("/login").permitAll())
      .rememberMe(rm -> rm.tokenValiditySeconds(1209600)) // 14 дней
      .csrf(Customizer.withDefaults());
  return http.build();
}
```

## LDAP/AD, SSO через корпоративных провайдеров, клиентские сертификаты (mTLS)

**LDAP/Active Directory** — каталоги пользователей. Spring Security умеет аутентифицировать через LDAP: либо bind-аутентификация, либо сравнение пароля с атрибутом. Для AD есть `ActiveDirectoryLdapAuthenticationProvider` (вводите домен, URL DC).

Пример простой LDAP-настройки (с Boot-стартером `spring-boot-starter-ldap`):

```java
@Bean
AuthenticationManager ldapAuth(HttpSecurity http) throws Exception {
  http
    .authenticationProvider(
      new LdapAuthenticationProvider(
        new BindAuthenticator(contextSource()), // bind user DN + password
        new DefaultLdapAuthoritiesPopulator(contextSource(), "ou=groups")));
  return http.getSharedObject(AuthenticationManager.class);
}

@Bean
LdapContextSource contextSource() {
  var cs = new LdapContextSource();
  cs.setUrl("ldaps://ldap.example.com:636");
  cs.setBase("dc=example,dc=com");
  cs.setUserDn("cn=service,ou=system,dc=example,dc=com");
  cs.setPassword("secret");
  return cs;
}
```

**SSO (Single Sign-On)** сегодня чаще делают по **OAuth2/OIDC** через корпоративных провайдеров (Keycloak, Okta, Azure AD). В этом случае ваше приложение — **OAuth2 Client**: редиректит на логин провайдера, принимает `code`, обменивает на токены и ведёт сессию. Для API — наоборот: вы ресурс-сервер, валидируете `Bearer`-токены.

**mTLS (Mutual TLS)** — взаимная аутентификация по клиентскому сертификату. Настраивается на уровне контейнера (Tomcat/Ingress): сервер требует сертификат клиента, Spring получает `X509Certificate` из запроса и маппит его на пользователя (`x509()` в DSL). Подходит для машинных B2B-интеграций и межсервисной аутентификации.

```java
@Bean
SecurityFilterChain mtls(HttpSecurity http) throws Exception {
  http.x509(Customizer.withDefaults())
      .authorizeHttpRequests(a -> a.anyRequest().hasRole("PARTNER"));
  return http.build();
}
```

## Кастомные AuthenticationProvider и собственные схемы токенов

Иногда нужен свой способ аутентификации — например, **API-ключ** в заголовке `X-Api-Key` или «подписанный» заголовок. В этом случае пишете свой **`AuthenticationProvider`** и (часто) фильтр, который извлекает креденшлы из запроса и кладёт `Authentication` в контекст.

Пример: провайдер для API-ключа и фильтр до авторизации:

```java
class ApiKeyAuthentication extends AbstractAuthenticationToken {
  private final String key;
  ApiKeyAuthentication(String key) { super(List.of()); this.key = key; setAuthenticated(false); }
  ApiKeyAuthentication(String key, Collection<? extends GrantedAuthority> auths) {
    super(auths); this.key = key; setAuthenticated(true);
  }
  @Override public Object getCredentials() { return key; }
  @Override public Object getPrincipal() { return "api-key-user"; }
}

@Component
class ApiKeyAuthenticationProvider implements AuthenticationProvider {
  @Override
  public Authentication authenticate(Authentication authentication) {
    String key = (String) authentication.getCredentials();
    if ("secret-key-123".equals(key)) {
      return new ApiKeyAuthentication(key, List.of(new SimpleGrantedAuthority("ROLE_API")));
    }
    throw new BadCredentialsException("Invalid API key");
  }
  @Override public boolean supports(Class<?> aClass) { return ApiKeyAuthentication.class.isAssignableFrom(aClass); }
}

@Component
class ApiKeyFilter extends OncePerRequestFilter {
  private final AuthenticationManager authManager;
  ApiKeyFilter(AuthenticationManager authManager) { this.authManager = authManager; }
  @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    String key = req.getHeader("X-Api-Key");
    if (key != null) {
      try {
        var auth = authManager.authenticate(new ApiKeyAuthentication(key));
        SecurityContextHolder.getContext().setAuthentication(auth);
      } catch (AuthenticationException ex) {
        res.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid API key");
        return;
      }
    }
    chain.doFilter(req, res);
  }
}

@Configuration
class ApiKeySecurity {

  @Bean
  SecurityFilterChain api(HttpSecurity http, ApiKeyFilter f) throws Exception {
    http.securityMatcher("/api/**")
        .addFilterBefore(f, UsernamePasswordAuthenticationFilter.class)
        .authorizeHttpRequests(a -> a.anyRequest().hasRole("API"))
        .csrf(csrf -> csrf.disable())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    return http.build();
  }
}
```

Такой подход удобен, когда у вас нет внешнего IdP, а клиентам проще выдавать ключи. Но помните про **ротацию ключей**, «ограничение источников» (IP allow-list) и rate limit.

---

Отлично — продолжаю «главу 15», подробно раскрывая **подтемы 4–6**. Для каждого подпункта — 4+ абзацев, с работающими примерами кода и пояснениями терминов/аннотаций. Стек: **Java 17+, Spring Boot 3.x, Spring Security 6.x**.

---

# 4. Авторизация: правила доступа и выражения

## Аннотации: `@PreAuthorize`/`@PostAuthorize`/`@Secured`/`@RolesAllowed`, включение метод-безопасности

В Spring Security можно проверять доступ не только на уровне URL (фильтры), но и **на уровне методов** (сервисов/контроллеров). Это включает аннотации: `@PreAuthorize` — проверяет выражение **до** вызова метода; `@PostAuthorize` — **после** (когда нужен доступ к возвращаемому объекту); `@Secured` — старая аннотация, которая перечисляет роли (например, `"ROLE_ADMIN"`); `@RolesAllowed` — стандарт JSR-250 с похожей семантикой. Чтобы они заработали, включите **метод-безопасность** аннотацией `@EnableMethodSecurity` (замена старой `@EnableGlobalMethodSecurity`), где можно также активировать pre/post-проверки.

Отличие `@PreAuthorize` в том, что она умеет **SpEL** (Spring Expression Language): можно смотреть аргументы метода (`#id`, `#dto.ownerId`), текущего пользователя (`authentication`, `principal`), а также вызывать бины (например, `@security.eval(#id)`). Это делает проверку точной и предметной: например, «пользователь может читать только свои ресурсы, а админ — любые». `@PostAuthorize` удобна, когда нужна фильтрация или проверка уже сформированного результата (например, запретить возврат чужого объекта).

`@Secured` и `@RolesAllowed` проще, но менее гибки: они не знают SpEL и работают только с ролями/авторитетами. Их плюс — читабельность в простых случаях и совместимость с Java EE-аннотациями. В проектах часто смешивают: `@PreAuthorize` для сложных проверок и `@RolesAllowed` для «простых» административных операций.

Минимальная настройка и пример:

```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
class MethodSecurityConfig {}

@Service
class AccountService {
  @PreAuthorize("hasRole('ADMIN') or #id == authentication.name")
  public AccountDto get(String id) { ... }

  @RolesAllowed("ADMIN")
  public void delete(String id) { ... }

  @PostAuthorize("returnObject.ownerId == authentication.name or hasAuthority('SCOPE_accounts.read')")
  public AccountDto find(String id) { ... }
}
```

## SpEL-выражения: `hasRole`/`hasAuthority`/`hasPermission`, собственный `PermissionEvaluator`

**SpEL** (Spring Expression Language) — язык выражений, который Spring Security использует для декларативных правил. Частые функции: `hasRole('ADMIN')` — проверяет наличие **роли** (под капотом добавляет префикс `ROLE_`), `hasAuthority('SCOPE_orders.read')` — проверяет **авторитет** (точное совпадение строки), `hasAnyAuthority(...)`, `isAuthenticated()`, `isAnonymous()`. В контексте выражения доступны объекты: `authentication` (текущий `Authentication`), `principal` (его `principal`), `#argName` (параметры метода).

Для предметных разрешений (например, «пользователь имеет право `EDIT` для документа X») используется `hasPermission(target, permission)`. Это опирается на интерфейс **`PermissionEvaluator`** — вы реализуете логику, как по идентификатору объекта и пользователю решить, разрешено ли действие. Так строят **ABAC/ACL**-модели (attribute/owner-based, списки контроля доступа).

Важно помнить различие между **ролью** и **авторитетом**. Роль — соглашение (обычно с префиксом `ROLE_`), авторитет — любая строка-привилегия. В JWT-мира часто используют **`SCOPE_*`** как авторитеты (например, `SCOPE_users.read`). Тогда `hasAuthority('SCOPE_users.read')` — естественный выбор, а вот `hasRole` — нет, потому что он будет добавлять `ROLE_`.

Пример своего `PermissionEvaluator` и его использование:

```java
@Component
class DocumentPermissionEvaluator implements PermissionEvaluator {
  @Override
  public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
    if (!(targetDomainObject instanceof Document doc)) return false;
    if ("READ".equals(permission)) {
      return doc.isPublic() || doc.ownerId().equals(auth.getName()) || has(auth, "ROLE_ADMIN");
    }
    if ("EDIT".equals(permission)) {
      return doc.ownerId().equals(auth.getName()) || has(auth, "ROLE_EDITOR");
    }
    return false;
  }
  @Override
  public boolean hasPermission(Authentication auth, Serializable targetId, String targetType, Object permission) {
    // Загрузка доменного объекта по targetId/targetType и делегирование в метод выше
    return false;
  }
  private boolean has(Authentication a, String authority) {
    return a.getAuthorities().stream().anyMatch(x -> x.getAuthority().equals(authority));
  }
}

@Service
class DocumentService {
  @PreAuthorize("hasPermission(#doc, 'EDIT')")
  public void update(Document doc) { ... }
}
```

## Политики: RBAC vs ABAC, доверительные границы и multi-tenant

**RBAC** (Role-Based Access Control) — простая модель: пользователю назначают роли, роли дают доступ к операциям. Прозрачно и дёшево в обслуживании, но грубовато: «админ» часто «умеет всё». **ABAC** (Attribute-Based) принимает решения на основе атрибутов **пользователя**, **ресурса** и **контекста**: отдел, владелец, метки, время, **tenant**. ABAC гибче (можно сказать «сотрудник читает документы своего отдела в рабочее время»), но требует дисциплины (где брать атрибуты, как их валидировать) и тщательного тестирования.

**Доверительная граница** — место, где вы перестаёте доверять входящим данным/заголовкам и полагаетесь только на проверенные источники (подписанный JWT, mTLS, заголовки от шлюза). Например, если «tenantId» приходит в заголовке от клиента, доверять ему нельзя; лучше — в claim токена (подписан IdP), а веб-шлюз продублирует его в заголовок для удобства логирования.

В **multi-tenant** системах (много арендаторов) важно «пронести» `tenantId` повсюду: из токена (claim), в `Authentication`, в SpEL. Частая практика — добавить `tenant` как **authorities prefix** (например, `TENANT_acme`) или прокладывать его отдельным компонентом, доступным в SpEL (`@tenantContext.current()`).

Пример правила «пользователь может работать только в пределах своего арендатора»:

```java
@Service
class TenantGuard {
  boolean sameTenant(String tenant) {
    var auth = (JwtAuthenticationToken) SecurityContextHolder.getContext().getAuthentication();
    var claimTenant = auth.getToken().getClaimAsString("tenant");
    return tenant.equals(claimTenant) || hasOps(auth);
  }
  private boolean hasOps(Authentication a) {
    return a.getAuthorities().stream().anyMatch(ga -> ga.getAuthority().equals("ROLE_OPS"));
  }
}

@Service
class InvoiceService {
  @PreAuthorize("@tenantGuard.sameTenant(#tenantId)")
  public InvoiceDto get(String tenantId, String id) { ... }
}
```

## AccessDecisionManager/Voter, иерархия ролей, принцип наименьших привилегий

**`AccessDecisionManager`** — компонент, который принимает итоговое решение по доступу, агрегируя голоса набора **Voter’ов** (реализации `AccessDecisionVoter`). Стратегии: **affirmative** (достаточно одного «за»), **consensus** (большинство), **unanimous** (только если никто не против). По умолчанию Spring Security использует современный **AuthorizationManager**, но понятие voter’ов остаётся релевантным (их можно подключать).

**`RoleHierarchy`** — иерархия ролей. Пример: `ROLE_ADMIN > ROLE_MANAGER > ROLE_USER` — это означает, что админ «унаследует» привилегии менеджера и пользователя. Регистрируется бин `RoleHierarchyImpl`, а затем через `RoleHierarchyAuthoritiesMapper` подмешивается в `Authentication`. Иерархии удобны, но используйте их умеренно — избыточная иерархия затрудняет анализ прав.

**Принцип наименьших привилегий** (Least Privilege) — выдавайте **минимум** прав, достаточный для задачи. Это означает granular-скоупы (вместо «*») и раздельные роли для административных и пользовательских функций. Для сервис-аккаунтов — отдельные роли с ограниченными скоупами, не наследуемые от пользовательских.

Пример иерархии и подключения:

```java
@Bean
RoleHierarchy roleHierarchy() {
  var rh = new RoleHierarchyImpl();
  rh.setHierarchy("""
      ROLE_ADMIN > ROLE_MANAGER
      ROLE_MANAGER > ROLE_USER
  """);
  return rh;
}

@Bean
GrantedAuthoritiesMapper authoritiesMapper(RoleHierarchy rh) {
  return new RoleHierarchyAuthoritiesMapper(rh);
}
```

---

# 5. JWT и ресурс-серверы

## `oauth2ResourceServer().jwt()`: валидация, JWKs, ротация ключей, clock-skew

**JWT** (JSON Web Token) — самоподписанный токен: внутри claims (кто пользователь, какие scope/roles), подпись **JWS** (обычно RS256/ES256). Ресурс-сервер проверяет подпись и стандартные поля: `exp` (срок действия), `iat` (время выпуска), `nbf` (не раньше), `iss` (issuer — кто выпустил токен), `aud` (аудитория — кому предназначен).

В Spring Security включение ресурс-сервера делается так: `http.oauth2ResourceServer(o -> o.jwt())`. Источник ключей — **JWK Set** (JSON Web Key Set): эндпоинт `/.well-known/jwks.json` у вашего IdP. Указать можно через `spring.security.oauth2.resourceserver.jwt.issuer-uri` (Boot сам найдёт JWKs по OIDC-метаданным) или через `jwk-set-uri`.

Важно учесть **ротацию ключей** (IdP меняет ключи): `NimbusJwtDecoder` кэширует JWKs и переопрашивает при необходимости. Ещё тонкость — **clock skew** (рассинхрон часов): чтобы не отклонять токен, который «только что» выдан, настройте допустимое смещение времени.

Пример конфигурации:

```java
@Configuration
class ResourceServerConfig {

  @Bean
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .csrf(csrf -> csrf.disable())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    return http.build();
  }

  @Bean
  JwtDecoder jwtDecoder() {
    var decoder = NimbusJwtDecoder.withJwkSetUri("https://idp.example.com/.well-known/jwks.json").build();
    // добавим проверку issuer и допустимое смещение времени
    OAuth2TokenValidator<Jwt> withIssuer = JwtValidators.createDefaultWithIssuer("https://idp.example.com/");
    OAuth2TokenValidator<Jwt> withClockSkew = (token) -> {
      var now = Instant.now();
      if (token.getExpiresAt() != null && token.getExpiresAt().isBefore(now.minusSeconds(30)))
        return OAuth2TokenValidatorResult.failure(new OAuth2Error("token_expired"));
      return OAuth2TokenValidatorResult.success();
    };
    decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(withIssuer, withClockSkew));
    return decoder;
  }
}
```

## Стейтлес-подход, отключение CSRF для API, Cache-Control

Для REST-API на Bearer-токенах лучшая практика — **stateless**: не хранить `SecurityContext`/сессию между запросами. Это уменьшает поверхность атак и упрощает масштабирование. В DSL это `sessionCreationPolicy(SessionCreationPolicy.STATELESS)`. Параллельно **выключают CSRF** (`csrf.disable()`), потому что CSRF актуален для cookie-сессий в браузере, а не для токенов в заголовке Authorization.

Для ответов API полезно настраивать **кеш-заголовки** (особенно на защищённых эндпоинтах): `Cache-Control: no-store, no-cache` для чувствительных данных; либо, наоборот, `max-age` для публичных (`/public/**`). Это можно задать через `headers().cacheControl(...)` или на уровне контроллеров/респондеров.

Не забывайте про другие заголовки безопасности: **HSTS**, **X-Content-Type-Options**, **X-Frame-Options** (или `frameOptions().sameOrigin()`), **CSP** (Content Security Policy) — особенно важно для страниц UI, но и API может отдавать файлы/HTML.

```java
http
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
  .csrf(csrf -> csrf.disable())
  .headers(h -> h.cacheControl(c -> {}) // включает no-cache/no-store по умолчанию
                 .httpStrictTransportSecurity(Customizer.withDefaults())
                 .contentTypeOptions(Customizer.withDefaults()));
```

## Маппинг claims → authorities, scope-driven доступ, audience/issuer

Когда вы используете JWT, в нём часто есть claim `scope`/`scp` (строка или массив), `roles`, `groups`. Чтобы «превратить» их в **`GrantedAuthority`**, настраивают **`JwtAuthenticationConverter`**. Стандартный `JwtGrantedAuthoritiesConverter` берёт `scope` и добавляет префикс `SCOPE_`. Если роли лежат в нестандартном claim’е, можно указать свой путь и префикс.

**Scope-driven** доступ — это когда вы пишете `hasAuthority('SCOPE_orders.read')`. Он удобен для API-разрешений. Если у вас и роли, и scope, можно объединять: админская роль даёт «широкие» права, скоупы — «узкие» на конкретные операции.

**`aud`** (audience) — кому предназначен токен. Ресурс-сервер должен проверять, что он — часть аудитории (иначе можно «проносить» токены, выпущенные для другого сервиса). **`iss`** (issuer) — кто выпустил токен; его нужно валидировать, чтобы не принять «случайный» JWT из другой экосистемы.

```java
@Bean
JwtAuthenticationConverter jwtAuthConverter() {
  var scopes = new JwtGrantedAuthoritiesConverter();
  scopes.setAuthorityPrefix("SCOPE_");
  scopes.setAuthoritiesClaimName("scp"); // или "scope"

  return new JwtAuthenticationConverter() {
    @Override
    protected Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
      var base = scopes.convert(jwt);
      var roles = ((Collection<String>) jwt.getClaim("roles"))
          .stream().map(r -> new SimpleGrantedAuthority("ROLE_" + r)).toList();
      return Stream.concat(base.stream(), roles.stream()).toList();
    }
  };
}

@Bean
SecurityFilterChain api(HttpSecurity http, JwtAuthenticationConverter conv) throws Exception {
  http.oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(conv)));
  // ...
  return http.build();
}
```

## Интроспекция vs самоподписанные JWT: компромиссы безопасности/производительности

Альтернатива самоподписанным JWT — **непрозрачные (opaque) токены** с **интроспекцией**: ресурс-сервер на каждый запрос ходит к IdP на `/introspect` и спрашивает «этот токен действителен, какие у него scope?». Плюсы: можно **немедленно отозвать** токен, меньше риска неверной валидации ключей на ресурс-сервере. Минусы: сетевой звонок на каждый запрос (надо кэшировать), зависимость от доступности IdP.

В Spring Security конфиг выглядит так: `http.oauth2ResourceServer().opaqueToken(ot -> ot.introspectionUri(...).introspectionClientCredentials(...))`. Стоит добавить **кэш** результатов интроспекции (например, Caffeine на 1–5 минут, уважая `exp`), и по возможности использовать **mTLS**/приватную сеть до IdP, потому что интроспекция — чувствительная операция.

Выбор между JWT и интроспекцией зависит от требований: если важна **офлайн-валидация** и максимальная производительность — JWT; если требуются **мгновенные отзыв/блокировка** и централизованный контроль — интроспекция (или «короткоживущие» JWT + токен-чекер на шлюзе).

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
  http.oauth2ResourceServer(o -> o.opaqueToken(ot -> ot
    .introspectionUri("https://idp.example.com/oauth2/introspect")
    .introspectionClientCredentials("rs-client", "rs-secret")));
  return http.build();
}
```

---

# 6. OAuth2/OIDC клиенты и авторизационный сервер

## Authorization Code (PKCE), client credentials, device code — где применяются

**Authorization Code** — основной **интерактивный** поток для пользовательских логинов: приложение редиректит в IdP, пользователь проходит аутентификацию, приложение получает **authorization code** и обменивает его на токены. **PKCE** (Proof Key for Code Exchange) — защита от перехвата кода: клиент генерирует `code_verifier`/`code_challenge`. В Spring Boot это настраивается через `spring-boot-starter-oauth2-client` и `oauth2Login()`.

**Client Credentials** — **машинный** поток: приложение (клиент) получает токен по своей паре `client_id`/`client_secret` без участия пользователя. Подходит для межсервисных вызовов и job’ов. В Spring используют `OAuth2AuthorizedClientManager` для автоматической выдачи/ротации токенов.

**Device Code** — для устройств без браузера/клавиатуры (TV, IoT): устройство показывает код, пользователь подтверждает на телефоне/ПК. В Spring Security есть поддержка как клиента (через `oauth2-client`) у поставщиков, которые реализуют этот флоу (Azure AD/Okta и т. п.).

Конфигурация клиента (YAML) и включение:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          my-oidc:
            provider: myidp
            client-id: ui-client
            client-secret: ${UI_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: openid, profile, email
          svc-client:
            provider: myidp
            client-id: svc-client
            client-secret: ${SVC_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: orders.read
        provider:
          myidp:
            issuer-uri: https://idp.example.com/
```

```java
@Configuration
class ClientSecurity {
  @Bean
  SecurityFilterChain ui(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(a -> a
          .requestMatchers("/", "/public/**").permitAll()
          .anyRequest().authenticated())
        .oauth2Login(Customizer.withDefaults()); // включаем OIDC login
    return http.build();
  }
}
```

## Spring Authorization Server vs внешние провайдеры (Keycloak/Okta/Azure AD)

**Spring Authorization Server (SAS)** — проект команды Spring, реализующий **Authorization Server**: выдачу токенов, JWKs, управление клиентами, поддержку OIDC. Плюс: полная интеграция со Spring, возможность «тонко» кастомизировать флоу, собственные гранты. Минус: вы становитесь владельцем IdP (надо обеспечить безопасность, масштабирование, HA, UI управления, аудит).

**Keycloak/Okta/Azure AD** — готовые провайдеры. Плюс: зрелые функции (MFA, админ-консоль, политики), поддержка пользователей/групп, кластера, отчёты, согласия, социальные логины. Минус: меньше контроля за деталями (хотя Keycloak тоже высоко кастомизируем), внешняя зависимость, стоимость (SaaS-решения).

Рекомендация: если у вас нет жёсткой потребности владеть IdP, **берите внешний провайдер**. SAS целесообразен в случаях высоких требований к кастомизации, air-gapped сред, особых регуляторных ограничений. В любом случае внимательно отнеситесь к SLO/Key rotation/Revocation/Аудиту.

Минимальный фрагмент SAS (как ресурс для прототипов):

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-oauth2-authorization-server")

// конфиг
@Configuration(proxyBeanMethods = false)
class AuthServerConfig {
  @Bean fun authServerSecurityFilterChain(http: HttpSecurity): SecurityFilterChain {
    OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
    return http.build();
  }
  @Bean fun registeredClientRepository(): RegisteredClientRepository { /* клиенты */ }
  @Bean fun jwkSource(): JWKSource<SecurityContext> { /* ключи */ }
}
```

## Логин через соц/корп-провайдеры: `@RegisteredOAuth2AuthorizedClient`, токен-релей

После `oauth2Login()` у вас появляется **сессионный пользователь** (`OAuth2User`/`OidcUser`), а также «привязанный» **`OAuth2AuthorizedClient`** — набор токенов (access/refresh) для текущего пользователя. Аннотация `@RegisteredOAuth2AuthorizedClient("my-oidc")` позволяет инжектировать этот объект в контроллер и получить **access token**.

**Token relay** — «проброс» пользовательского токена при вызове других сервисов. Для `WebClient` есть `ServerOAuth2AuthorizedClientExchangeFilterFunction` (для MVC — аналог `ServletOAuth2AuthorizedClientExchangeFilterFunction`): он добавит `Authorization: Bearer …` автоматически из текущего авторизованного клиента.

Примеры:

```java
@RestController
class ProfileController {

  @GetMapping("/profile")
  Map<String, Object> profile(@AuthenticationPrincipal OidcUser user) {
    return Map.of("sub", user.getSubject(), "email", user.getEmail(), "claims", user.getClaims());
  }

  @GetMapping("/call-downstream")
  Mono<String> callDownstream(
      @RegisteredOAuth2AuthorizedClient("my-oidc") OAuth2AuthorizedClient client) {

    var token = client.getAccessToken().getTokenValue();
    return WebClient.create("https://api.example.com")
        .get().uri("/me")
        .headers(h -> h.setBearerAuth(token))
        .retrieve().bodyToMono(String.class);
  }
}
```

```java
@Configuration
class OAuth2WebClientConfig {
  @Bean
  WebClient oauth2WebClient(ClientRegistrationRepository repo,
                            OAuth2AuthorizedClientRepository clients) {
    var oauth2 = new ServletOAuth2AuthorizedClientExchangeFilterFunction(repo, clients);
    oauth2.setDefaultOAuth2AuthorizedClient(true); // взять «текущего» клиента
    return WebClient.builder().apply(oauth2.oauth2Configuration()).build();
  }
}
```

## Multi-tenant, кастомные claim-мапперы, согласование userinfo и локальной модели

В **multi-tenant** системах один UI может работать с несколькими IdP/issuer’ами. Для ресурс-сервера это решают **динамическим выбором декодера** (по `iss` в токене): `JwtIssuerAuthenticationManagerResolver` сам подберёт соответствующий `JwtDecoder` на основе `issuer-uri → JWKs`. Для клиента (oauth2Login) можно динамически выбирать регистрацию по домену/параметру (мульти-регистрации в `application.yml`).

**Кастомные claim-мапперы** — это преобразование claims → authorities/профиль. Для JWT — см. `JwtAuthenticationConverter` выше. Для OIDC-пользователя — кастомный `OAuth2UserService<OidcUserRequest, OidcUser>`: можно загрузить дополнительные атрибуты (через userinfo endpoint), сопоставить с локальными ролями, создать пользователя в локальной БД, если его ещё нет.

Согласование userinfo и локальной модели — типичный шаг: IdP знает «кто он», а ваша система — «что ему можно». Обычно реализуют: (1) извлечь `sub`, `email`, `groups`, (2) найти/создать локального `User`, (3) определить authorities по корпоративной матрице (RBAC/ABAC), (4) отдать `OidcUser` с нужными ролями.

Пример: динамический выбор по issuer + кастомный маппер:

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
  var resolver = JwtIssuerAuthenticationManagerResolver.fromTrustedIssuers(
      "https://idp-a.example.com/", "https://idp-b.example.com/");
  http.oauth2ResourceServer(o -> o.authenticationManagerResolver(resolver));
  return http.build();
}

@Bean
OAuth2UserService<OidcUserRequest, OidcUser> oidcUserService(RoleService roles) {
  return req -> {
    var delegate = new OidcUserService(); // стандартный загрузчик userinfo
    var user = delegate.loadUser(req);
    var tenant = user.getClaimAsString("tenant");
    var authorities = roles.mapForTenant(tenant, user.getEmail());
    return new DefaultOidcUser(authorities, user.getIdToken(), user.getUserInfo());
  };
}
```

# 7. Веб-слой: защита REST и UI

## Конфигурация CORS и CSRF (когда включать/выключать), SameSite и cookie-политики

**CORS** (Cross-Origin Resource Sharing) — механизм браузера, разрешающий фронтенду с домена A ходить к вашему бекенду на домене B. В Spring включается через `http.cors()` и бин `CorsConfigurationSource`. Нельзя оставлять «всё позволено» в проде: указывайте точные `allowedOrigins`, методы и заголовки, и включайте `allowCredentials` только когда реально нужны куки/авторизация. Для API-клиентов вне браузера CORS не играет роли (он «живёт» в браузере).

**CSRF** (Cross-Site Request Forgery) — атака на stateful-приложения, где аутентификация держится на cookie-сессии (браузер автоматически отправит cookie). Для «чистых» REST-API со **stateless** подходом (Bearer/JWT в заголовке `Authorization`) CSRF обычно **отключают**: `.csrf(csrf -> csrf.disable())`. Для UI с формами — наоборот, **включают** по умолчанию, выдавая токен через hidden-поле или header `X-CSRF-TOKEN`.

Политики cookie важны для безопасности браузера. Флаг **`HttpOnly`** защищает от чтения cookie JS-кодом, **`Secure`** обязует HTTPS, **`SameSite`** управляет межсайтовой отправкой: `Lax` (дефолт: не шлём в большинстве кросс-сценариев), `Strict` (только «своё» происхождение), `None` (разрешаем межсайт, но требуют `Secure`). Для Single-Page Apps с кросс-доменным бекендом нередко нужен `SameSite=None; Secure` и корректная CORS-настройка.

Пример продуманной CORS/CSRF/headers-конфигурации для API и UI:

```java
@Configuration
class WebSecurity {
  @Bean
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .cors(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable())
        .headers(h -> h
          .httpStrictTransportSecurity(Customizer.withDefaults())
          .contentTypeOptions(Customizer.withDefaults()))
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt());
    return http.build();
  }

  @Bean
  SecurityFilterChain ui(HttpSecurity http) throws Exception {
    http.securityMatcher("/", "/app/**", "/login/**")
        .authorizeHttpRequests(a -> a.anyRequest().permitAll())
        .formLogin(Customizer.withDefaults())       // stateful UI
        .csrf(Customizer.withDefaults())            // CSRF включён
        .cors(Customizer.withDefaults());
    return http.build();
  }

  @Bean
  CorsConfigurationSource corsSource() {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://frontend.example.com")); // не "*"
    cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
    cfg.setAllowedHeaders(List.of("Authorization","Content-Type","X-Requested-With"));
    cfg.setAllowCredentials(true);
    var src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", cfg);
    return src;
  }
}
```

## Ограничение частоты (rate limiting), защита от brute-force и перечисления пользователей

**Rate limiting** — ограничение количества запросов за интервал. Это защита от DDoS/сканирования/дорогих операций. В Spring можно повесить фильтр до аутентификации и/или использовать пер-эндпоинт/пер-пользователь лимиты. Библиотеки вроде **Bucket4j** дают токен-бакеты в памяти/Redis. Минимальный «свой» лимитер можно собрать на `ConcurrentHashMap<key, Counter>` с TTL.

**Brute-force** — перебор паролей. Ограничивайте попытки логина по `(username, ip)` и добавляйте «притормаживание» (progressive delay). Не возвращайте разные ответы на «пользователь не существует»/«неверный пароль» — это предотвращает **user enumeration** (перечисление пользователей). Возвращайте одинаковое «не удалось войти», логируйте детально только на сервере.

Для API-ключей и JWT также уместны лимиты по ключу/клиенту. Для критичных операций (подтверждение платежа) добавляйте **second-factor** (OTP, WebAuthn), а не полагайтесь только на лимитирование. Обязательно алертите всплески 401/403/429 по клиентам — это ранние индикаторы атаки.

Пример грубого rate-limit фильтра на Bucket4j:

```java
@Component
class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, io.github.bucket4j.Bucket> buckets = new ConcurrentHashMap<>();

  @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    String key = Optional.ofNullable(req.getHeader("X-Client-Id"))
                         .orElse(req.getRemoteAddr()); // упрощённый ключ
    var bucket = buckets.computeIfAbsent(key, k ->
        io.github.bucket4j.Bucket4j.builder()
           .addLimit(io.github.bucket4j.Bandwidth.simple(100, Duration.ofMinutes(1)))
           .build());
    if (bucket.tryConsume(1)) {
      chain.doFilter(req, res);
    } else {
      res.setStatus(429);
      res.setContentType("application/problem+json");
      res.getWriter().write("""{"title":"Too Many Requests","status":429}""");
    }
  }
}
```

## Обработчики ошибок: `AuthenticationEntryPoint`/`AccessDeniedHandler`, ProblemDetail

`AuthenticationEntryPoint` — «вход» в аутентификацию для **неаутентифицированного** пользователя: для HTML — редирект на `/login`, для REST — 401 с JSON-телом. `AccessDeniedHandler` обрабатывает 403 для **аутентифицированного**, но не уполномоченного пользователя. В Spring Security 6 настраивается через `http.exceptionHandling(...)`.

Формат **RFC-7807 (Problem Details)** — универсальный JSON для ошибок: поля `type`, `title`, `status`, `detail`, `instance`. Spring Boot 3 имеет класс `ProblemDetail`. Его можно использовать и в фильтрах, и в `@ControllerAdvice`, чтобы унифицировать ошибки приложения и слоя безопасности.

Важный нюанс: при 401 добавляйте `WWW-Authenticate` (Basic, Bearer) — браузеры/клиенты ожидают его. Для 403 — чёткий `title/detail` без утечки чувствительных данных (не пишите «роль admin нужна», если это помогает злоумышленнику).

Пример единообразных обработчиков:

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
  http.exceptionHandling(ex -> ex
      .authenticationEntryPoint((req, res, e) -> {
        res.setStatus(401);
        res.setHeader("WWW-Authenticate", "Bearer");
        res.setContentType("application/problem+json");
        res.getWriter().write("""{"type":"about:blank","title":"Unauthorized","status":401}""");
      })
      .accessDeniedHandler((req, res, e) -> {
        res.setStatus(403);
        res.setContentType("application/problem+json");
        res.getWriter().write("""{"type":"about:blank","title":"Forbidden","status":403}""");
      })
  );
  // ...
  return http.build();
}
```

## Безопасные загрузки/скачивания: Content-Type/Disposition, диапазоны и валидация

Загрузка файлов — источник рисков. Проверяйте **MIME-тип** (`Content-Type`) и **расширение**, лимитируйте размер (`spring.servlet.multipart.max-file-size`) и **никогда** не сохраняйте под именем, предоставленным клиентом, без нормализации пути (path traversal). Валидируйте содержимое (magic bytes), если это критично (например, изображение/CSV).

Отдача файлов: используйте корректные **`Content-Type`** и **`Content-Disposition`**. Чтобы предотвратить XSS в браузере при открытии, отдавайте **скачивание**: `Content-Disposition: attachment; filename="..."` для несейфового контента. Для больших файлов поддерживайте **диапазоны** (HTTP Range) — отдача кусками снижает нагрузку и дружит с возобновлением скачивания.

Если файлы лежат во внешнем хранилище (S3/MinIO), предпочтительнее выдавать **pre-signed URL** с ограниченным TTL, а не проксировать через приложение. Это снимает нагрузку и уменьшает риск утечки ключей.

Пример скачивания с безопасными заголовками:

```java
@GetMapping("/api/files/{id}")
public ResponseEntity<Resource> download(@PathVariable String id) {
  var file = storage.load(id); // проверка авторизации/владения
  var body = new InputStreamResource(file.inputStream());
  return ResponseEntity.ok()
      .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + file.safeName() + "\"")
      .header(HttpHeaders.CACHE_CONTROL, "no-store")
      .contentType(MediaType.parseMediaType(file.contentType()))
      .contentLength(file.length())
      .body(body);
}
```

---

# 8. Микросервисы и периметр безопасности

## API-шлюз/ingress: централизованный login/logout, токен-релей и маппинг заголовков

**API-шлюз** (Spring Cloud Gateway, NGINX Ingress, Envoy) — точка входа в микросервисы. На нём удобно централизовать **OIDC-логин/логаут**, проверку токенов (как ресурс-сервер), **rate limiting**, CORS и унифицированные заголовки (traceId, user, tenant). Внутренним сервисам шлюз передаст проверенную идентичность через `Authorization` или доверенные заголовки (например, `X-User-Id`), но добавляйте **подпись**/mTLS, иначе заголовки легко подменить.

**Token relay** — проксирование токена пользователя дальше по цепочке, чтобы downstream-сервисы делали авторизацию сами. В Spring Cloud Gateway это делается фильтрами (`TokenRelay`), а в приложениях — через `WebClient` с `ServletOAuth2AuthorizedClientExchangeFilterFunction`.

Маппинг заголовков — единый контракт: какие заголовки всегда есть (`X-Request-Id`, `X-Tenant`), какие чувствительные (`Authorization`) и как их логировать/маскировать. Шлюз — хорошее место, чтобы внедрить **HSTS**, **CSP**, политик заголовков и централизованный 401/403/429 ответ.

Пример token relay на gateway:

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay
      routes:
        - id: orders
          uri: http://orders:8080
          predicates: [ Path=/api/orders/** ]
```

## Межсервисные вызовы: пропагация идентичности, opaque tokens introspection, mTLS

Для S2S-взаимодействий два пути: (1) **пропагация пользовательского токена** (token relay), если downstream должен знать «кто пользователь»; (2) **client credentials** (сервис-аккаунт), если запрос выполняется от имени сервиса. В обоих случаях downstream — ресурс-сервер, проверяющий токен локально (JWT) или через **интроспекцию** (opaque).

Интроспекция (`/introspect`) позволяет централизовать управление (немедленный отзыв токенов), но добавляет сетевой вызов и требует кэша. Для критичных внутренних каналов включайте **mTLS** (взаимная TLS-аутентификация): сервисы предъявляют клиентский сертификат, а ingress/peer проверяют цепочку. В Spring Boot это настраивается на уровне контейнера (Tomcat/Undertow) и `server.ssl.*`, а в Security — `.x509()`.

Пример клиента с client-credentials:

```java
@Configuration
class S2SClientConfig {
  @Bean
  OAuth2AuthorizedClientManager clientManager(ClientRegistrationRepository cr, OAuth2AuthorizedClientRepository rep) {
    var provider = OAuth2AuthorizedClientProviderBuilder.builder().clientCredentials().build();
    var m = new DefaultOAuth2AuthorizedClientManager(cr, rep);
    m.setAuthorizedClientProvider(provider);
    return m;
  }

  @Bean
  WebClient s2sWebClient(OAuth2AuthorizedClientManager mgr) {
    var oauth2 = new ServletOAuth2AuthorizedClientExchangeFilterFunction(mgr);
    oauth2.setDefaultClientRegistrationId("svc-client");
    return WebClient.builder().apply(oauth2.oauth2Configuration()).build();
  }
}
```

## Сегментация ролей/скоупов между сервисами, сервис-аккаунты для batch/job

В микросервисах роли/скоупы должны быть **минимальными и изолированными**. Не раздавайте `admin` «на весь периметр» — определяйте `orders.read`, `orders.write`, `payments.refund` и т. п. У каждого сервиса — свой набор, у каждого публичного API — свои аудитории (`aud`). Это уменьшает blast-radius при компрометации.

**Сервис-аккаунты** — клиенты OAuth2 с grant **client_credentials**. Их права — не пользовательские: не назначайте им `ROLE_USER`. Задавайте конкретные scope (например, `inventory.sync`) и срок жизни секретов (ротация). Для **batch/job** лучше выделить отдельные логины/ключи, чтобы можно было быстро отозвать их, не ломая другие интеграции.

Пример проверки узкоспециализированного скоупа:

```java
@RestController
class AdminOpsController {
  @PostMapping("/api/admin/refund/{id}")
  @PreAuthorize("hasAuthority('SCOPE_payments.refund')")
  public void refund(@PathVariable String id) { /* ... */ }
}
```

## Секьюрный конфиг: secrets/vault, ключевая ротация, минимальные права в CI/CD

Храните секреты (клиентские пароли, приватные ключи, JDBC-пароли) вне репозитория: **Vault**, AWS/GCP Secret Manager, Kubernetes Secrets. Spring имеет интеграции (Spring Cloud Vault/Config). Не логируйте секреты, используйте маскирование. В dev удобно иметь `.env.sample`, а реальные значение — в секретах CI.

Настройте **ротацию** ключей/сертификатов/JWKs (короткий срок жизни, двойной выпуск на время перехода). Пайплайны CI/CD должны иметь **минимальные привилегии** (principle of least privilege): доступ только к нужным секретам, read-only к репозиторию артефактов, запрет на прод-сервера без явного approver’а.

Пример Spring Cloud Vault (фрагмент):

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      authentication: kubernetes
      kubernetes:
        role: app-role
      generic:
        enabled: true
        backend: secret
        default-context: app/prod
```

---

# 9. Тестирование безопасности

## Spring Security Test: `withMockUser()`, `SecurityMockMvcRequestPostProcessors.jwt()`, тесты `@PreAuthorize`

Библиотека **spring-security-test** даёт аннотации/пост-процессоры для имитации аутентификации. `@WithMockUser(roles="ADMIN")` — создаёт фиктивного пользователя. Для ресурс-сервера с JWT используйте `SecurityMockMvcRequestPostProcessors.jwt()` — он добавляет в запрос **поддельный JWT** с нужными authorities/claims.

Метод-безопасность (`@PreAuthorize`) лучше тестировать **на уровне контроллеров/сервисов** отдельными тестами, вызывая методы напрямую (для сервисов) или через MockMvc (для контроллеров). Проверяйте и положительные, и отрицательные сценарии: 200/403, а также 401 для анонимных.

Пример:

```java
@WebMvcTest(controllers = AdminOpsController.class)
@Import(SecurityConfig.class) // цепочка для теста
class AdminOpsControllerTest {
  @Autowired MockMvc mvc;

  @Test
  void forbidsWithoutScope() throws Exception {
    mvc.perform(post("/api/admin/refund/123").with(jwt().authorities())) // без scope
       .andExpect(status().isForbidden());
  }

  @Test
  void allowsWithScope() throws Exception {
    mvc.perform(post("/api/admin/refund/123")
       .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_payments.refund"))))
       .andExpect(status().isOk());
  }
}
```

## Интеграционные тесты контроллеров: MockMvc/WebTestClient, негативные 401/403

Для MVC-приложений `MockMvc` быстрый и богатый на проверки (`jsonPath`, `header`). Для реактивных — `WebTestClient`. Убедитесь, что тесты покрывают **негатив**: неаутентифицированный доступ (401), недостаточные права (403), CORS preflight (OPTIONS) и CSRF (для форм: 403 без токена).

В `@SpringBootTest(webEnvironment=RANDOM_PORT)` можно дополнительно прогнать «настоящий» HTTP через `TestRestTemplate`, чтобы увидеть поведение реального сервера (заголовки `WWW-Authenticate`, CORS, cookies).

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityFlowsIT {
  @Autowired MockMvc mvc;

  @Test void anonymousGets401() throws Exception {
    mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized());
  }

  @Test void userGets403() throws Exception {
    mvc.perform(get("/api/admin/metrics").with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_orders.read"))))
       .andExpect(status().isForbidden());
  }
}
```

## Контрактные тесты с Keycloak/Testcontainers, заглушки introspection/JWKs через WireMock

Для «настоящего» потока OIDC удобно поднимать **Keycloak** через **Testcontainers**: импортировать realm, создать клиента/пользователя и провести полный код-флоу/валидацию JWT. Это тяжёлые тесты — обычно в отдельном наборе (`integrationTest`) или nightly.

Если Keycloak поднимать слишком дорого, заглушайте **JWKs** (`/.well-known/jwks.json`) и/или **introspection** endpoint **WireMock’ом**, чтобы ресурс-сервер успешно валидировал токен. В первом случае вы контролируете набор ключей, во втором — отвечаете `{"active": true, "scope": "orders.read"}`.

Пример заглушки JWKs:

```java
@AutoConfigureWireMock(port = 0)
@SpringBootTest(properties = {
  "spring.security.oauth2.resourceserver.jwt.jwk-set-uri=http://localhost:${wiremock.server.port}/jwks"
})
class JwksStubIT {
  @Test void jwksServed() {
    stubFor(get("/jwks").willReturn(okJson("""
      {"keys":[{"kty":"RSA","kid":"test","n":"...","e":"AQAB"}]}
    """)));
    // далее подставляете JWT, подписанный "test" ключом (или отключаете подпись в тесте)
  }
}
```

## Снапшоты правил доступа, регрессии и «песочница» ролей

Полезная техника — **снапшоты доступа**: таблица «эндпоинт → роли/скоупы → ожидаемый статус». Их можно хранить в `src/test/resources/access-matrix.json` и прогонять тестом: для каждой роли «песочницы» отправлять запросы и сверять статусы. Это помогает обнаружить регресс при изменении правил `authorizeHttpRequests` или `@PreAuthorize`.

Такой «матрицей» удобно делиться с командой/безопасниками; она же служит документацией. Поддерживайте в CI «красный» при несоответствии — иначе регресс в доступах пройдёт незаметно.

---

# 10. Хардненинг, наблюдаемость и отладка

## Заголовки безопасности (HSTS, X-Content-Type-Options, CSP), логирование аутентификации/доступа

**HSTS** (HTTP Strict Transport Security) заставляет браузер использовать только HTTPS (на период `max-age`); включать обязательно в проде. **X-Content-Type-Options: nosniff** запрещает MIME-sniffing; **X-Frame-Options**/`frameOptions()` защищает от clickjacking; **CSP** (Content Security Policy) контролирует источники скриптов/стилей — мощная защита от XSS, но требует аккуратной настройки.

Логирование **аутентификации** и **доступа** помогает расследовать инциденты. Фильтры `AuthenticationLogging` можно повесить до/после auth-фильтров, а успешные/неуспешные попытки логина — через `AuthenticationEventPublisher`. Для HTTP-доступа используйте **access log** приложения/ingress с полями: метод, путь, статус, user/tenant, scope, traceId.

Пример заголовков и событий аутентификации:

```java
http.headers(h -> h
   .httpStrictTransportSecurity(Customizer.withDefaults())
   .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'")))
  .securityMatcher("/**");

@Bean
ApplicationListener<AbstractAuthenticationEvent> authEvents() {
  return evt -> log.info("Auth event: {} for {}", evt.getClass().getSimpleName(),
                         (evt.getAuthentication() != null ? evt.getAuthentication().getName() : "anon"));
}
```

## Аудит действий пользователей, трассировка (traceId/spanId) и корреляция с журналами

**Аудит** — кто что сделал и когда. В Spring можно использовать `AuditEventRepository` (Spring Boot Actuator) или собственный сервис аудита. Минимум: `principal`, действие (CRUD/бизнес-операция), объект (ID/тип), результат (успех/ошибка), `traceId`. Старайтесь **не** логировать чувствительные данные (PII/секреты) — используйте маскирование.

**Трассировка** (Micrometer Tracing/Brave/OpenTelemetry) добавляет `traceId/spanId` в логи и заголовки (`X-B3-TraceId`/`traceparent`). Обязательно прокидывайте их через шлюз и логируйте на каждом сервисе: это связывает события аутентификации, авторизации и бизнес-операции в единую цепочку и ускоряет расследование.

MDC (Mapped Diagnostic Context) добавляет в логи поля вроде user/tenant. На входе запроса положите их в MDC, на выходе — очистите. Для реактивного стека используйте «reactor context → MDC bridge».

```java
@Component
class CorrelationFilter extends OncePerRequestFilter {
  @Override protected void doFilterInternal(HttpServletRequest r, HttpServletResponse s, FilterChain c)
      throws IOException, ServletException {
    try (var ignored = MDC.putCloseable("user", Optional.ofNullable(r.getUserPrincipal()).map(Principal::getName).orElse("anon"))) {
      c.doFilter(r, s);
    }
  }
}
```

## Общие уязвимости: открытые эндпоинты-«дыры», переизбыточные скоупы, неверный order фильтров

Частая ошибка — **случайно открытые эндпоинты** (например, забыли добавить `/internal/**` в защищённую цепочку или первое правило `permitAll()` перехватывает слишком много). Лечится **матрицей доступа** (см. выше), строгими `securityMatcher` и `@Order`. Вторая проблема — **широкие скоупы** (`*`, `admin`) или смешение пользовательских и сервисных ролей, что расширяет зону поражения.

Неверный **порядок фильтров** (вставили свой фильтр после авторизации, а он должен был быть до неё) ломает логику или «вскрывает» контекст. Вставляйте через `addFilterBefore/After` относительно «якорных» фильтров (`UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`). Ещё одна ловушка — «доверие» к заголовкам из вне (например, `X-User-Id`) без верификации на периметре.

Наконец, **CORS-дырки** (`allowedOrigins=*`, `allowCredentials=true`) и неправильный CSRF (выключили для UI) встречаются регулярно. Следите за безопасными дефолтами: для API — stateless + CSRF off + CORS точечный; для UI — CSRF on + HSTS/CSP.

## Roadmap: passkeys/WebAuthn, FIDO2, Zero-Trust, policy-as-code (OPA/SOPA)

**WebAuthn/Passkeys (FIDO2)** — аутентификация без пароля (аппаратные ключи/биометрия). В экосистеме Spring это реализуют через внешние провайдеры (Keycloak/Okta с WebAuthn) или специализированные библиотеки. Для приложений это значит: меньше password-поверхности, MFA «по умолчанию», но нужна поддержка на фронтенде и в IdP.

**Zero-Trust** — не доверяй сети/периметру: проверяй идентичность и состояние устройства на каждом запросе, минимальные привилегии, микросегментация. Для Spring-микросервисов это означает системный JWT/opaque с короткими TTL, частые ротации, mTLS, явная авторизация на каждом сервисе, непрерывное аудит-логирование.

**Policy-as-Code** (OPA — Open Policy Agent, SOPA — Spring OPA интеграции) — вынесение авторизационных правил в декларативные политики (Rego/SpEL) с централизованной проверкой. Это повышает прозрачность и позволяет независимым командам менять правила без релиза кода (при аккуратной валидации/тестировании).

Планируя будущее, закладывайте: короткоживущие токены + refresh-потоки, автоматическую **ротацию ключей** (JWKs), стандартизацию scopes/roles, и «синие/красные» тесты в CI на нарушения матрицы доступа. Это упростит миграцию на новые механизмы без рефакторинга всей кодовой базы.

---

отлично — вот два минимальных «мини-репозитория» + список частых проблем. Каждый шаблон самодостаточный: скопируй файлы, выполни `./gradlew test` (Java 17+, Spring Boot 3.2.x).

---

# 1) Готовые `SecurityFilterChain` для API/UI (CORS, CSRF, headers)

**Структура**

```
security-chains-api-ui/
  build.gradle.kts
  settings.gradle.kts
  src/main/java/com/example/security/WebSecurityConfig.java
  src/main/java/com/example/web/ApiController.java
  src/main/java/com/example/web/UiController.java
  src/main/resources/application.yml
  src/test/java/com/example/security/SecurityChainsIT.java
```

**`settings.gradle.kts`**

```kotlin
rootProject.name = "security-chains-api-ui"
```

**`build.gradle.kts`**

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.2.6"
  id("io.spring.dependency-management") version "1.1.5"
}

group = "com.example"; version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_17
repositories { mavenCentral() }

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-security")

  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.springframework.security:spring-security-test")
  testImplementation("org.assertj:assertj-core:3.26.0")
}

tasks.test { useJUnitPlatform() }
```

**`src/main/resources/application.yml`**

```yaml
server:
  port: 8080

spring:
  application:
    name: security-chains-api-ui
```

**`src/main/java/com/example/security/WebSecurityConfig.java`**

```java
package com.example.security;

import java.util.List;
import org.springframework.context.annotation.*;
import org.springframework.http.HttpStatus;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.HeadersConfigurer;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.*;
import org.springframework.security.web.authentication.HttpStatusEntryPoint;
import org.springframework.web.cors.*;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class WebSecurityConfig {

  // API: stateless, CORS ON, CSRF OFF, безопасные заголовки
  @Bean @Order(1)
  SecurityFilterChain api(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .cors(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable())
        .headers(h -> h
            .httpStrictTransportSecurity(HeadersConfigurer.HstsConfig::includeSubDomains)
            .contentTypeOptions(Customizer.withDefaults())
            .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin))
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)))
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());
    return http.build();
  }

  // UI: stateful, CSRF ON, formLogin, CORS ON
  @Bean @Order(2)
  SecurityFilterChain ui(HttpSecurity http) throws Exception {
    http.securityMatcher("/", "/app/**", "/login/**")
        .authorizeHttpRequests(a -> a
            .requestMatchers("/", "/login/**").permitAll()
            .anyRequest().authenticated())
        .formLogin(Customizer.withDefaults()) // /login
        .csrf(Customizer.withDefaults())
        .cors(Customizer.withDefaults());
    return http.build();
  }

  // CORS: разрешаем фронтенд-домен
  @Bean
  CorsConfigurationSource corsConfigurationSource() {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://frontend.example.com")); // не "*"
    cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
    cfg.setAllowedHeaders(List.of("Authorization","Content-Type","X-Requested-With"));
    cfg.setAllowCredentials(true);
    var src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", cfg);
    return src;
  }
}
```

**`src/main/java/com/example/web/ApiController.java`**

```java
package com.example.web;

import java.util.Map;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class ApiController {

  @GetMapping("/public/ping")
  public Map<String, Object> publicPing() {
    return Map.of("ok", true, "zone", "public");
  }

  @GetMapping("/hello")
  public Map<String, Object> hello() {
    return Map.of("ok", true, "zone", "api");
  }
}
```

**`src/main/java/com/example/web/UiController.java`**

```java
package com.example.web;

import java.util.Map;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/app")
public class UiController {

  @GetMapping("/profile")
  @ResponseBody
  public Map<String, Object> profile() {
    return Map.of("ok", true, "zone", "ui");
  }

  // Пример POST, который требует CSRF-токен
  @PostMapping("/profile")
  @ResponseBody
  public Map<String, Object> update() {
    return Map.of("status", "updated");
  }
}
```

**`src/test/java/com/example/security/SecurityChainsIT.java`**

```java
package com.example.security;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;

@SpringBootTest
@AutoConfigureMockMvc
class SecurityChainsIT {

  @Autowired org.springframework.test.web.servlet.MockMvc mvc;

  @Test
  void corsPreflightIsOk() throws Exception {
    mvc.perform(options("/api/hello")
            .header("Origin","https://frontend.example.com")
            .header("Access-Control-Request-Method","GET"))
        .andExpect(status().isOk())
        .andExpect(header().string("Access-Control-Allow-Origin","https://frontend.example.com"));
  }

  @Test
  void apiAnonymousGets401() throws Exception {
    mvc.perform(get("/api/hello")).andExpect(status().isUnauthorized());
  }

  @Test
  @WithMockUser // Basic заходит в API-цепочку
  void apiAuthGets200AndHstsHeader() throws Exception {
    mvc.perform(get("/api/hello"))
        .andExpect(status().isOk())
        .andExpect(header().exists("Strict-Transport-Security"));
  }

  @Test
  void uiPostWithoutCsrfIs403() throws Exception {
    mvc.perform(post("/app/profile").contentType(MediaType.APPLICATION_FORM_URLENCODED))
        .andExpect(status().isForbidden());
  }

  @Test
  @WithMockUser
  void uiPostWithCsrfIs200() throws Exception {
    mvc.perform(post("/app/profile")
            .with(SecurityMockMvcRequestPostProcessors.csrf())
            .contentType(MediaType.APPLICATION_FORM_URLENCODED))
        .andExpect(status().isOk());
  }
}
```

> Запуск:
> `cd security-chains-api-ui && ./gradlew test`

---

# 2) Ресурс-сервер на JWT + интеграционные тесты `jwt()`

**Структура**

```
jwt-resource-server/
  build.gradle.kts
  settings.gradle.kts
  src/main/java/com/example/security/ResourceServerConfig.java
  src/main/java/com/example/web/OrdersController.java
  src/main/resources/application.yml
  src/test/java/com/example/security/JwtResourceServerIT.java
```

**`settings.gradle.kts`**

```kotlin
rootProject.name = "jwt-resource-server"
```

**`build.gradle.kts`**

```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.2.6"
  id("io.spring.dependency-management") version "1.1.5"
}

group = "com.example"; version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_17
repositories { mavenCentral() }

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-security")
  implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")

  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.springframework.security:spring-security-test")
  testImplementation("org.assertj:assertj-core:3.26.0")
}

tasks.test { useJUnitPlatform() }
```

**`src/main/resources/application.yml`**

```yaml
server:
  port: 8080
spring:
  application:
    name: jwt-resource-server
# Для реального продакшена укажи issuer-uri или jwk-set-uri:
# spring.security.oauth2.resourceserver.jwt.issuer-uri: https://idp.example.com/
```

**`src/main/java/com/example/security/ResourceServerConfig.java`**

```java
package com.example.security;

import java.util.*;
import java.util.stream.Stream;
import org.springframework.context.annotation.*;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity
public class ResourceServerConfig {

  @Bean
  SecurityFilterChain api(HttpSecurity http, JwtAuthenticationConverter jwtConv) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/health").permitAll()
            .requestMatchers("/api/orders/**").hasAuthority("SCOPE_orders.read")
            .anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtConv)))
        .csrf(csrf -> csrf.disable())
        .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
    return http.build();
  }

  // Преобразуем claims → authorities: scp → SCOPE_*, roles → ROLE_*
  @Bean
  JwtAuthenticationConverter jwtAuthenticationConverter() {
    var scopes = new JwtGrantedAuthoritiesConverter();
    scopes.setAuthoritiesClaimName("scp"); // или "scope"
    scopes.setAuthorityPrefix("SCOPE_");

    return new JwtAuthenticationConverter() {
      @Override
      protected Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
        var base = scopes.convert(jwt);
        var roles = Optional.ofNullable(jwt.getClaimAsStringList("roles")).orElse(List.of());
        var roleAuths = roles.stream().map(r -> new SimpleGrantedAuthority("ROLE_" + r)).toList();
        return Stream.concat(base.stream(), roleAuths.stream()).toList();
      }
    };
  }

  // В тестах мы используем .with(jwt()), поэтому реальный JwtDecoder не обязателен.
  // Для продакшена подключи NimbusJwtDecoder с issuer-uri/jwk-set-uri.
}
```

**`src/main/java/com/example/web/OrdersController.java`**

```java
package com.example.web;

import java.util.List;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrdersController {

  @GetMapping
  @PreAuthorize("hasAuthority('SCOPE_orders.read')")
  public List<String> all() {
    return List.of("ORD-1", "ORD-2");
  }

  @GetMapping("/{id}")
  @PreAuthorize("hasAuthority('SCOPE_orders.read')")
  public String one(@PathVariable String id) {
    return "ORD-" + id;
  }
}
```

**`src/test/java/com/example/security/JwtResourceServerIT.java`**

```java
package com.example.security;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;

@SpringBootTest
@AutoConfigureMockMvc
class JwtResourceServerIT {

  @Autowired org.springframework.test.web.servlet.MockMvc mvc;

  @Test
  void anonymousGets401() throws Exception {
    mvc.perform(get("/api/orders")).andExpect(status().isUnauthorized());
  }

  @Test
  void userWithoutScopeGets403() throws Exception {
    mvc.perform(get("/api/orders")
            .with(SecurityMockMvcRequestPostProcessors.jwt()
                  .authorities(new SimpleGrantedAuthority("SCOPE_profile.read"))))
        .andExpect(status().isForbidden());
  }

  @Test
  void userWithScopeGets200() throws Exception {
    mvc.perform(get("/api/orders")
            .with(SecurityMockMvcRequestPostProcessors.jwt()
                  .authorities(new SimpleGrantedAuthority("SCOPE_orders.read"))))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$[0]").value("ORD-1"));
  }

  @Test
  void roleIsMappedButNotEnoughWithoutScope() throws Exception {
    mvc.perform(get("/api/orders")
            .with(SecurityMockMvcRequestPostProcessors.jwt()
                  .jwt(jwt -> jwt.claim("roles", java.util.List.of("ADMIN")))
                  .authorities())) // без SCOPE_orders.read — недостаточно
        .andExpect(status().isForbidden());
  }
}
```

> Запуск:
> `cd jwt-resource-server && ./gradlew test`

---

# Часто встречающиеся проблемы

1. **Смешение ролей и скоупов**
   Симптом: правила с `hasRole()` не срабатывают для JWT.
   Причина: в токене нет `ROLE_*`, есть `SCOPE_*`.
   Решение: используйте `hasAuthority('SCOPE_x')` или настройте `JwtAuthenticationConverter`.

2. **Случайно открытые эндпоинты**
   Симптом: приватный эндпоинт доступен анонимно.
   Причина: слишком широкое `permitAll()` или порядок цепочек.
   Решение: проверяйте `securityMatcher`/`@Order`, ведите «матрицу доступа» в тестах.

3. **Неверная CORS-конфигурация**
   Симптом: браузер блокирует запросы, ошибки preflight.
   Причина: `allowedOrigins="*"` вместе с `allowCredentials=true` или забыли OPTIONS.
   Решение: укажи точные origin’ы, включи методы/заголовки, протестируй preflight.

4. **Выключенный CSRF для UI**
   Симптом: POST-формы проходят без токена, риск CSRF.
   Причина: глобальный `.csrf().disable()`.
   Решение: держи CSRF включённым для UI-цепочки, отключай только для статeless API.

5. **H2-console или Swagger открыты в проде**
   Симптом: доступ к `/h2-console`/`/swagger-ui` без авторизации.
   Причина: dev-настройки уехали в prod.
   Решение: профилируй (`@Profile("dev")`), явные правила доступа, скрывай в проде.

6. **Bearer-токен в логе**
   Симптом: токены попадают в логи/трейсы.
   Причина: verbose-логирование request headers.
   Решение: маскирование, фильтрация логов, запрет DEBUG на проде.

7. **Clock skew и «истёкшие» JWT**
   Симптом: внезапные 401 около времени выпуска/продления.
   Причина: рассинхрон часов.
   Решение: допустимое смещение (±30–60 сек) в валидаторах токенов, синхронизация времени.

8. **Неверный порядок фильтров**
   Симптом: кастомный фильтр не видит пользователя или ломает контекст.
   Причина: `addFilterAfter`/`Before` относительно wrong «якоря».
   Решение: вставляй относительно `UsernamePasswordAuthenticationFilter`/`BearerTokenAuthenticationFilter` и тестируй.

9. **Сессии в API (stateless забыли)**
   Симптом: JSESSIONID у REST-клиентов, липкие сессии в балансере.
   Причина: не задан `STATELESS`.
   Решение: `.sessionCreationPolicy(STATELESS)` для API-цепочки.

10. **User enumeration**
    Симптом: разные ответы для «пользователь не существует» и «неправильный пароль».
    Решение: унифицируй ответы, логируй детали только на сервере, лимить попыток.

11. **«Глобальный» админ-скоуп**
    Симптом: сервис-аккаунт с `admin` может всё в любом сервисе.
    Решение: сегментируй скоупы/аудитории per-service, минимальные привилегии.

12. **Отсутствие Negative-тестов 401/403**
    Симптом: случайно расширили доступ, тесты зелёные.
    Решение: всегда добавляй тесты на 401/403 и матрицу доступа.

13. **Конвертер не мапит claims**
    Симптом: роли из `roles` игнорируются.
    Причина: не настроен `JwtAuthenticationConverter`.
    Решение: добавь кастомный конвертер (см. шаблон #2).

14. **CSP отсутствует на UI**
    Симптом: XSS через инъекцию скриптов.
    Решение: добавь CSP заголовок, используй nonce/sha256 для разрешённых ресурсов.

15. **CSRF включён на API c токенами**
    Симптом: POST/PUT/DELETE возвращают 403 для API-клиентов.
    Решение: отключи CSRF на stateless-цепочке.

16. **Невалидный `aud`/`iss`**
    Симптом: принимаете токены от чужих клиентов/issuer’ов.
    Решение: валидируй `issuer` и `audience` (JwtValidators + custom checks).

17. **Отсутствие ротации ключей**
    Симптом: истёкшие JWKs/сертификаты ломают прод ночью.
    Решение: план ротации, «двойная публикация» ключей, мониторинг сроков.

18. **Секреты в репозитории**
    Симптом: токены/пароли в git-истории.
    Решение: Vault/Secret Manager, `.env.sample`, сканеры секретов в CI, ревокация.

19. **Одинаковые правила для dev/prod**
    Симптом: удобные dev-доступы попадают в прод.
    Решение: профили dev/prod, отдельные цепочки, фичефлаги.

20. **Нет аудита/трассировки**
    Симптом: тяжело расследовать инцидент.
    Решение: audit trail (кто/что/когда/результат), traceId/spanId в логах, корреляция на шлюзе.

