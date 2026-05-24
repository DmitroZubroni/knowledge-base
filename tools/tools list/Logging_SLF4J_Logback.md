# SLF4J / Logback

> [!abstract] Связи
> [[main Tools]] | [[Spring_Boot]]

---

## 🔹 Архитектура логирования Java

```
Application code
    ↓
SLF4J API (Logger interface)     ← фасад
    ↓
Logback / Log4j2 / JUL           ← реализация
    ↓
Console / File / ELK
```

> [!note] Почему SLF4J
> Код зависит только от API — можно сменить Logback на Log4j2 без изменения `log.info()`.

**Spring Boot default:** SLF4J + Logback.

---

## 🔹 Уровни логирования

| Уровень | Когда |
|---------|-------|
| **TRACE** | Детальная отладка (редко) |
| **DEBUG** | Dev: SQL, flow |
| **INFO** | Бизнес-события: старт, заказ создан |
| **WARN** | Деградация: retry, fallback |
| **ERROR** | Ошибки, требующие внимания |

> [!warning] Не логировать
> Пароли, tokens, полные PAN, персональные данные без маскирования.

---

## 🔹 SLF4J — API

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

// Lombok
@Slf4j
@Service
public class OrderService { }
```

### ❌ Конкатенация

```java
log.debug("User data: " + user.toString());  // toString() всегда вызывается
```

### ✅ Параметризованные сообщения

```java
log.debug("User {} created order {}", userId, orderId);
log.debug("Expensive: {}", () -> heavyComputation());  // lazy (если поддерживается)
```

---

## 🔹 Logback конфигурация

`logback-spring.xml` — с поддержкой Spring profiles.

```xml
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>50MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d %level [%X{requestId}] %logger - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.example" level="DEBUG"/>
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

---

## 🔹 Spring Boot Logging

```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
  file:
    name: logs/application.log
  pattern:
    console: "%d %-5level [%X{requestId}] %logger - %msg%n"
```

**Structured JSON (Boot 3.4+):**

```yaml
logging:
  structured:
    format:
      console: ecs  # или logstash
```

---

## 🔹 MDC на практике

> [!note] MDC
> **Mapped Diagnostic Context** — ThreadLocal map: `requestId`, `userId` в каждой строке лога.

```java
@Component
public class RequestIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String requestId = Optional.ofNullable(req.getHeader("X-Request-Id"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("requestId", requestId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();  // обязательно — thread pool reuse
        }
    }
}
```

Pattern: `[%X{requestId}]`

---

## 🔹 Типичные ошибки

### ❌ Логировать и бросать

```java
catch (Exception e) {
    log.error("Failed", e);
    throw e;  // GlobalHandler тоже залогирует — дубль
}
```

### ✅ Один уровень ответственности

```java
catch (Exception e) {
    throw new OrderProcessingException("Failed to process", e);  // log в handler
}
```

### ❌ Проглотить

```java
catch (Exception e) {
    log.error("oops");  // без stack trace, без rethrow
}
```

---

## 🔹 Итог

1. SLF4J API + Logback implementation в Boot.
2. Параметризованные сообщения — не конкатенация.
3. MDC + Filter — correlation id в логах.
4. `logback-spring.xml` для file rotation.
5. Не логировать secrets; `MDC.clear()` в finally.

```
Шпаргалка Logging:
─────────────────────────────────────────
@Slf4j / LoggerFactory.getLogger
log.info("x={}", x)                → не concat
TRACE < DEBUG < INFO < WARN < ERROR
logging.level.com.example=DEBUG
MDC.put("requestId", id)           → correlation
MDC.clear() in finally
logback-spring.xml                 → appenders, rotation
❌ passwords, tokens in logs
```
