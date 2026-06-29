# 12 — Паттерны проектирования (Design Patterns)

> **Теги:** #java #programming #patterns #gof #middle #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]]

---

## 🔹 Что такое паттерны проектирования

> [!note] Определение
> **Паттерн проектирования** — проверенное решение типовой задачи проектирования ПО. Не готовый код, а шаблон взаимодействия классов.

**Классификация GoF:**

| Группа | Назначение |
|--------|------------|
| **Creational** | Создание объектов |
| **Structural** | Композиция классов и объектов |
| **Behavioral** | Распределение обязанностей и алгоритмов |

```
Паттерн          | Группа      | Spring
─────────────────┼─────────────┼──────────────────────────
Singleton        | Creational  | bean scope singleton
Factory Method   | Creational  | @Bean, BeanFactory
Strategy         | Behavioral  | @Primary, PaymentGateway
Observer         | Behavioral  | ApplicationEvent
Proxy            | Structural  | AOP @Transactional
Decorator        | Structural  | Security filters
Chain of Resp.   | Behavioral  | SecurityFilterChain
Template Method  | Behavioral  | JdbcTemplate
Facade           | Structural  | @Service layer
```

---

## 🔹 Creational (Порождающие)

### Singleton

```java
// ❌ ручной singleton — проблемы с тестами и сериализацией
public class Config {
    private static Config instance = new Config();
    public static Config getInstance() { return instance; }
}
```

> [!tip] В Spring
> `@Component` + scope `singleton` (default) — один bean на контейнер. Потокобезопасность — контейнер создаёт bean один раз.

### Factory Method

```java
public interface NotificationFactory {
    Notification create(String type);
}

@Component
public class EmailNotificationFactory implements NotificationFactory {
  public Notification create(String type) {
    return new EmailNotification();
  }
}
```

> [!note] Spring
> `BeanFactory.getBean()` — фабрика bean. `@Bean` методы в `@Configuration` — factory method pattern.

### Abstract Factory

```java
// семейство связанных объектов для окружения
public interface DataSourceFactory {
    DataSource createDataSource();
    TransactionManager createTxManager();
}
```

> [!tip] Пример
> Разные `@Profile` конфигурации: `dev` → H2, `prod` → PostgreSQL.

### Builder

```java
Order order = Order.builder()
    .productId(1L)
    .quantity(2)
    .build();

ResponseEntity.created(uri).body(dto);  // ResponseEntity builder
```

### Prototype

```java
@Component
@Scope("prototype")
public class ReportJob { }  // новый экземпляр при каждом getBean()
```

---

## 🔹 Structural (Структурные)

### Adapter

```java
// legacy API → наш интерфейс
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentClient client;
    public void charge(Money amount) { client.pay(amount.cents()); }
}
```

> [!note] Spring MVC
> `HandlerAdapter` — адаптирует разные типы handler (`@Controller`, `HttpRequestHandler`) к единому вызову.

### Decorator

```java
// Servlet Filter оборачивает request/response
public class LoggingFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        log.info("Before");
        chain.doFilter(req, res);
        log.info("After");
    }
}
```

> [!tip] Security
> Цепочка фильтров — декораторы над `HttpServletRequest`.

### Proxy

```java
@Service
public class OrderService {
    @Transactional  // Spring создаёт proxy, открывает TX вокруг метода
    public void placeOrder() { }
}
```

> [!note] AOP Proxy
> **JDK dynamic proxy** — интерфейсы. **CGLIB** — subclass для классов без интерфейса.

### Facade

```java
@Service
public class CheckoutFacade {
    // скрывает OrderService + PaymentService + InventoryService
    public Receipt checkout(Cart cart) { ... }
}
```

### Composite

```java
// дерево: Department содержит Employee и подотделы
public interface OrgNode {
    int getHeadcount();
}
```

### Bridge

> [!note] JDBC
> `DataSource` — абстракция; `Driver` — реализация для PostgreSQL/MySQL. Приложение не зависит от конкретного драйвера.

---

## 🔹 Behavioral (Поведенческие)

### Strategy

```java
public interface PaymentStrategy { void pay(BigDecimal amount); }

@Service
@Primary
public class CardPayment implements PaymentStrategy { }

@Service
public class PayPalPayment implements PaymentStrategy { }

@Service
@RequiredArgsConstructor
public class CheckoutService {
    private final PaymentStrategy strategy;  // или Map + @Qualifier
}
```

### Template Method

```java
// JdbcTemplate — фиксирует алгоритм, hook — твой RowMapper
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users",
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("email"))
);
```

### Observer

```java
publisher.publishEvent(new OrderCreatedEvent(this, orderId));

@EventListener
public void onOrderCreated(OrderCreatedEvent e) { }
```

### Chain of Responsibility

```
Request → Filter1 → Filter2 → ... → DispatcherServlet → Interceptor → Controller
```

### Command

```java
@Async
public void sendEmailAsync(EmailCommand cmd) { }

// Runnable / Callable — инкапсуляция действия
```

### State

```java
public enum OrderStatus { CREATED, PAID, SHIPPED, CANCELLED }
// переходы: CREATED → PAID только после payment
```

### Iterator

```java
for (User u : users) { }           // Iterable
users.stream().filter(...).toList(); // Stream — internal iterator
```

### Visitor

> [!note] Пример
> `ExpressionVisitor` в парсерах, AST-обход. В бизнес-коде редко; чаще — открытые иерархии через pattern matching (Java 21+).

---

## 🔹 Антипаттерны (знать на Middle)

| Антипаттерн | Проблема | Решение |
|-------------|----------|---------|
| **God Object** | Один класс делает всё | SRP, разбить на сервисы |
| **Anemic Domain Model** | Entity только getters/setters, логика в Service | Rich domain model где уместно |
| **Magic Numbers** | `if (status == 3)` | Enum, константы |
| **Service Locator** | `context.getBean()` везде | Constructor DI |
| **Singleton Overuse** | Глобальное mutable state | DI, stateless services |

### ❌ Service Locator

```java
UserService user = ApplicationContextHolder.getContext().getBean(UserService.class);
```

### ✅ Dependency Injection

```java
@RequiredArgsConstructor
public class OrderController {
    private final UserService userService;
}
```

---

## 🔹 Spring и паттерны — сводная таблица

| Паттерн | Где в Spring | Пример |
|---------|--------------|--------|
| Singleton | Bean scope | `@Service` default |
| Factory Method | `@Bean` | `DataSource` config |
| Strategy | `@Primary` / `@Qualifier` | `PaymentGateway` |
| Template Method | `JdbcTemplate`, `RestTemplate` | `query(sql, mapper)` |
| Observer | Events | `@EventListener` |
| Proxy | AOP | `@Transactional`, `@Cacheable` |
| Decorator | Filters | `OncePerRequestFilter` |
| Chain of Responsibility | Security | `SecurityFilterChain` |
| Facade | Service layer | `CheckoutService` |
| Adapter | MVC | `HandlerAdapter` |

---

## 🔹 Итог

1. GoF: Creational / Structural / Behavioral.
2. Spring реализует многие паттерны «из коробки» — не изобретать вручную.
3. Strategy + DI — выбор реализации без `if-else`.
4. Proxy/AOP — cross-cutting concerns без дублирования.
5. Избегать God Object, Service Locator, Anemic Model без причины.

```
Шпаргалка Patterns:
─────────────────────────────────────────
Singleton        → Spring bean (default scope)
Factory          → @Bean, BeanFactory
Strategy         → interface + @Primary/@Qualifier
Observer         → ApplicationEvent
Proxy            → @Transactional (AOP)
Decorator        → Servlet Filter chain
Chain of Resp.   → SecurityFilterChain
Template Method  → JdbcTemplate
Facade           → @Service orchestration
❌ Service Locator → ✅ Constructor DI
```

---

## 🔗 Связанные конспекты

> [!abstract] Смотри также
> - [[15_Patterns]] (интервью) — паттерны GoF на собеседовании
> - [[10_SOLID]] — принципы, которые лежат в основе паттернов
> - [[Spring_Core]] — Spring реализует Singleton, Factory, Proxy, Template Method
> - [[16_Microservices]] (интервью) — архитектурные паттерны микросервисов
