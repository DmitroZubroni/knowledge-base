# Spring — Boot (Auto-configuration, Starters)

> **Теги:** #spring #framework #boot #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_Spring_Index]]

---

## 🔹 Что такое Spring Boot

> [!note] Определение
> **Spring Boot** — надстройка над Spring Framework: auto-configuration, embedded server, starters, production-ready features (Actuator).

| Spring Framework | Spring Boot |
|------------------|-------------|
| Нужна ручная конфигурация | Convention over configuration |
| WAR на внешний Tomcat | Embedded server, executable JAR |
| Сборка зависимостей вручную | Starters |
| Много XML/JavaConfig | `application.yml` + условные auto-config |

**Что решает:** быстрый старт, единообразная структура, меньше boilerplate, встроенные метрики и health.

---

## 🔹 Структура проекта

```
src/
├── main/
│   ├── java/
│   │   └── com/example/app/
│   │       ├── Application.java      ← @SpringBootApplication
│   │       ├── config/
│   │       ├── controller/
│   │       ├── service/
│   │       └── repository/
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       └── static/ templates/
└── test/
    └── java/ ...
```

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 🔹 @SpringBootApplication

```java
@SpringBootApplication
// эквивалент:
// @SpringBootConfiguration  → @Configuration
// @EnableAutoConfiguration  → подключение auto-config
// @ComponentScan            → scan пакета класса и ниже
public class Application { }
```

> [!tip] Scan base
> `@SpringBootApplication(scanBasePackages = "com.example")` — если main-класс не в корневом пакете.

---

## 🔹 Auto-configuration

**Механизм (Boot 3.x):**
```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    → список классов *AutoConfiguration
    → каждый класс с @Conditional* аннотациями
    → bean создаётся только если условие выполнено
```

| Аннотация | Условие |
|-----------|---------|
| `@ConditionalOnClass` | Класс в classpath |
| `@ConditionalOnMissingBean` | Bean ещё не определён |
| `@ConditionalOnProperty` | Property = значение |
| `@ConditionalOnWebApplication` | Web-приложение |
| `@ConditionalOnExpression` | SpEL true |
| `@ConditionalOnResource` | Файл существует |
| `@ConditionalOnJava` | Версия Java |
| `@ConditionalOnSingleCandidate` | Ровно один кандидат bean |

```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.url")
public class DataSourceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties props) { ... }
}
```

> [!tip] Отладка
> `debug=true` или `--debug` — в логе **CONDITIONS EVALUATION REPORT**: какие auto-config сработали / не сработали.

---

## 🔹 Starters

> [!note] Определение
> **Starter** — Maven/Gradle dependency, подтягивающая набор согласованных библиотек + auto-configuration.

| Starter | Подтягивает |
|---------|-------------|
| `spring-boot-starter-web` | MVC, Jackson, Tomcat, REST |
| `spring-boot-starter-data-jpa` | JPA, Hibernate, JDBC, HikariCP |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-boot-starter-validation` | Bean Validation (Hibernate Validator) |
| `spring-boot-starter-actuator` | Health, metrics endpoints |
| `spring-boot-starter-oauth2-resource-server` | JWT validation |
| `spring-boot-starter-webflux` | WebFlux, Netty |

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

---

## 🔹 application.properties / yaml

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

**Приоритет источников** (выше = приоритетнее, упрощённо):

1. Command line args
2. `SPRING_APPLICATION_JSON`
3. Java System properties
4. OS environment variables
5. `application-{profile}.properties` (profile-specific)
6. `application.properties` / `application.yml`
7. `@PropertySource` на `@Configuration`
8. Default properties

> [!tip] 15 уровней в документации
> Полный список: [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)

---

## 🔹 @ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public record MailProperties(
    @NotBlank String host,
    @Min(1) int port,
    Duration timeout
) {}

@Configuration
@EnableConfigurationProperties(MailProperties.class)
public class MailConfig {
    @Bean
    public JavaMailSender mailSender(MailProperties props) { ... }
}
```

```yaml
app:
  mail:
    host: smtp.example.com
    port: 587
    timeout: 30s
```

> [!tip] `@ConfigurationProperties` vs `@Value`
> Для **группы** связанных свойств — `@ConfigurationProperties` (type-safe, validation). Для одиночных — `@Value`.

---

## 🔹 Profiles в Boot

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://db:5432/app
```

```java
@SpringBootTest
@ActiveProfiles("test")
class OrderServiceTest { }
```

---

## 🔹 Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env
  endpoint:
    health:
      show-details: when_authorized
```

| Endpoint | Назначение |
|----------|------------|
| `/actuator/health` | Статус приложения, БД, disk |
| `/actuator/info` | Build info (git, version) |
| `/actuator/metrics` | Micrometer metrics |
| `/actuator/env` | Properties (осторожно!) |

```java
@Component
public class PaymentHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        return paymentGateway.isUp()
            ? Health.up().build()
            : Health.down().withDetail("reason", "timeout").build();
    }
}
```

> [!warning] Безопасность
> В production ограничить exposure Actuator, защитить Spring Security, не открывать `env` публично.

---

## 🔹 Embedded Server

| Сервер | По умолчанию | Особенности |
|--------|--------------|-------------|
| **Tomcat** | `starter-web` | Servlet, зрелый |
| **Jetty** | опционально | Легче, embedded |
| **Undertow** | опционально | XNIO, высокая производительность |

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: secret
```

```gradle
implementation('org.springframework.boot:spring-boot-starter-web') {
    exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
}
implementation 'org.springframework.boot:spring-boot-starter-undertow'
```

---

## 🔹 Banner, ExitCode, ApplicationRunner

```yaml
spring:
  main:
    banner-mode: console  # off | log | console
```

```java
@Component
public class StartupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // после полного старта контекста
    }
}

@Component
public class CommandLineStartup implements CommandLineRunner {
    @Override
    public void run(String... args) { }
}
```

**Порядок:** `ApplicationRunner` / `CommandLineRunner` → после контекста, до приёма трафика.

```java
SpringApplication.exit(context, () -> 0);  // graceful shutdown + exit code
```

---

## 🔹 Graceful Shutdown

> [!note] Определение
> **Graceful Shutdown** — корректное завершение: сначала перестаём принимать новые запросы,
> ждём завершения текущих, потом гасим контекст.

```yaml
server:
  shutdown: graceful  # immediate (default) | graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # ждём до 30с
```

```
SIGTERM
    ↓
SmartLifecycle.stop() — перестаём принимать запросы
    ↓
Ждём завершения in-flight запросов (timeout)
    ↓
@PreDestroy / DisposableBean.destroy()
    ↓
JVM exit
```

> [!warning] Kubernetes
> В k8s graceful shutdown критичен: `terminationGracePeriodSeconds` должен быть
> больше `timeout-per-shutdown-phase` + время на `preStop` hook.

> [!tip] Проверка
> `kill -SIGTERM <pid>` или `docker stop` — правильно завершает.
> `kill -SIGKILL` — принудительно, данные могут потеряться.

---

## 🔹 Итог

1. Boot = auto-config + starters + embedded server.
2. `@SpringBootApplication` = Configuration + ComponentScan + EnableAutoConfiguration.
3. Auto-config условный — `@ConditionalOn*`.
4. Properties: yaml + profiles + env vars; приоритет у command line.
5. `@ConfigurationProperties` — type-safe конфигурация.
6. Actuator — health/metrics для production.
7. `ApplicationRunner` — логика после старта.

```
Шпаргалка Boot:
─────────────────────────────────────────
@SpringBootApplication           → entry point
spring-boot-starter-*            → зависимости
application-{profile}.yml        → профили
@ConditionalOnMissingBean        → не перезаписывать custom bean
@ConfigurationProperties          → группа props
management.endpoints.web.exposure  → Actuator
debug=true                       → отчёт auto-config
ApplicationRunner                → post-start hook
server.shutdown: graceful       → ждём завершения запросов
```
