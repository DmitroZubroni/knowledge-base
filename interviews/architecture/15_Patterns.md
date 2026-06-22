> **Теги:** #interviews #architecture #patterns #gof #конспект
> [!abstract] Связи
> [[main]] | [[Interviews]] | [[Patterns_Index]]

# Design Patterns — Вопросы на собесе

---

## 🔹 Как отвечать на вопросы о паттернах

На собесе не нужно пересказывать определение из книги. Нужно показать три вещи:
1. **Проблему** которую решает паттерн
2. **Пример** из реального кода или Spring
3. **Когда НЕ использовать** (трейдоффы)

---

## 🔹 Порождающие — быстрая шпаргалка

### Singleton

```java
// ✅ Enum — лучший способ
public enum AppConfig {
    INSTANCE;
    public String getDbUrl() { return "jdbc:..."; }
}

// ✅ Static Holder — lazy без synchronized
public class Cache {
    private Cache() {}
    private static class Holder { static final Cache INSTANCE = new Cache(); }
    public static Cache getInstance() { return Holder.INSTANCE; }
}

// ✅ Double-Checked Locking — volatile ОБЯЗАТЕЛЕН
private static volatile Singleton instance;
public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) instance = new Singleton();
        }
    }
    return instance;
}
```

**В Spring:** `@Component` — singleton по умолчанию. Spring управляет жизненным циклом. В тестах подменяется через `@MockBean`.

**Проблемы ручного Singleton:** сложно тестировать (нельзя подменить), глобальное состояние = скрытые зависимости.

---

### Factory Method

**Проблема:** код знает конкретный класс при создании — жёсткая зависимость.

```java
// Фабрика скрывает логику создания
public class NotificationFactory {
    public static Notification create(String type, String dest) {
        return switch (type) {
            case "email" -> new EmailNotification(dest);
            case "sms"   -> new SmsNotification(dest);
            default      -> throw new IllegalArgumentException(type);
        };
    }
}

Notification n = NotificationFactory.create("email", "user@mail.com");
// Клиент не знает какой конкретный класс создаётся
```

**В Spring:** `@Bean` методы в `@Configuration` — это Factory Methods.

---

### Abstract Factory

**Проблема:** нужно создавать **семейства** взаимосвязанных объектов.

```java
// Вся инфраструктура переключается одной переменной
@Configuration @Profile("dev")
public class DevFactory {
    @Bean public DataSource dataSource() { return new H2DataSource(); }
    @Bean public MailSender mailSender() { return new FakeMailSender(); }
}

@Configuration @Profile("prod")
public class ProdFactory {
    @Bean public DataSource dataSource() { return new PostgresDataSource(); }
    @Bean public MailSender mailSender() { return new SmtpMailSender(); }
}
```

**Отличие от Factory Method:** FM создаёт один продукт, AF — семейство связанных.

---

### Builder

**Проблема:** объект с 10+ параметрами — нечитаемый конструктор.

```java
// ❌ Telescoping constructor
new User("John", "Doe", 25, "john@mail.com", null, null, true, false);

// ✅ Builder
User user = new User.Builder("John", "john@mail.com")
    .age(25)
    .active(true)
    .build(); // валидация здесь

// ✅ Lombok @Builder в реальном коде
@Builder
public class CreateOrderRequest {
    private final Long customerId;
    private final List<Long> productIds;
    @Builder.Default
    private final Currency currency = Currency.USD;
}
```

**Даёт:** immutable объект, читаемое создание, валидация в `build()`.
**В Spring API:** `ResponseEntity.builder()`, `UriComponentsBuilder`.

---

### Prototype

**Проблема:** создание объекта дорого, нужно много похожих.

```java
// @Scope("prototype") — новый экземпляр при каждом getBean()
@Component @Scope("prototype")
public class ReportGenerator { ... }
```

---

## 🔹 Структурные — быстрая шпаргалка

### Proxy ← самый важный, в Spring везде

**Проблема:** нужно добавить поведение (логирование, кэш, транзакция) не меняя исходный класс.

```java
// Паттерн: тот же интерфейс, делегирование + добавленная логика
public class LoggingUserService implements UserService {
    private final UserService delegate;
    public User findById(Long id) {
        log.info("Finding user {}", id);
        User user = delegate.findById(id); // делегируем
        log.info("Found: {}", user);
        return user;
    }
}
```

**В Spring — весь AOP это Proxy:**
```java
@Transactional  // Spring создаёт Proxy вокруг метода
public void placeOrder(Order order) { ... }

@Cacheable("users")  // Proxy: проверить кэш → вызвать → сохранить в кэш
public User findById(Long id) { ... }
```

**Два типа Proxy в Spring:**
- **JDK Dynamic Proxy** — через интерфейс. Класс должен реализовывать интерфейс.
- **CGLIB** — подкласс через байткод. Работает без интерфейса, не работает с `final`.

> [!warning] Self-invocation — главный подводный камень Spring Proxy
> ```java
> @Service
> public class OrderService {
>     @Transactional
>     public void placeOrder() {
>         this.validate(); // ❌ вызов через this — минует Proxy, @Transactional не работает!
>     }
>     @Transactional(propagation = REQUIRES_NEW)
>     public void validate() { ... }
> }
> ```

---

### Decorator

**Отличие от Proxy:** Proxy контролирует доступ, Decorator добавляет функциональность. Декораторы стекируются по выбору клиента.

```java
// Java I/O — классический пример
InputStream raw  = new FileInputStream("file.txt");      // базовый
InputStream buf  = new BufferedInputStream(raw);         // + буферизация
DataInputStream data = new DataInputStream(buf);         // + типизированное чтение

// Collections.unmodifiable* — Decorator для защиты
List<String> immutable = Collections.unmodifiableList(mutable);
```

---

### Adapter

**Проблема:** несовместимые интерфейсы — нужно соединить без изменения кода.

```java
// Legacy API → наш интерфейс
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentSystem legacy;

    @Override
    public PaymentResult charge(String customerId, BigDecimal amount) {
        boolean ok = legacy.processPayment(Long.parseLong(customerId),
                                           amount.doubleValue() * 100);
        return ok ? PaymentResult.success() : PaymentResult.failure(legacy.getLastError());
    }
}
```

**В Spring MVC:** `HandlerAdapter` — адаптирует `@Controller`, `HttpRequestHandler`, `Servlet` к единому интерфейсу `DispatcherServlet`.

---

### Facade

**Проблема:** сложная подсистема из 7 классов — клиент вынужден всё знать.

```java
// @Service как Facade — один метод вместо 7 вызовов в контроллере
@Service @RequiredArgsConstructor
public class CheckoutFacade {
    private final UserRepo userRepo;
    private final ProductRepo productRepo;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    public OrderConfirmation checkout(CheckoutRequest request) {
        User user = userRepo.findById(request.getUserId());
        Product product = productRepo.findById(request.getProductId());
        inventoryService.checkStock(product, request.getQuantity());
        PaymentResult payment = paymentService.charge(user, product.getPrice());
        Order order = orderRepo.save(Order.of(user, product, payment));
        notificationService.sendConfirmation(user, order);
        return OrderConfirmation.from(order);
    }
}
// Контроллер зависит только от одного класса — CheckoutFacade
```

---

### Composite

**Проблема:** нужно работать с деревьями объектов единообразно — и с листом, и с группой.

```java
// Один интерфейс для файла и папки
interface FileSystemItem { long getSize(); }
class File implements FileSystemItem { public long getSize() { return size; } }
class Directory implements FileSystemItem {
    private List<FileSystemItem> children = new ArrayList<>();
    public long getSize() {
        return children.stream().mapToLong(FileSystemItem::getSize).sum(); // рекурсия
    }
}
```

---

## 🔹 Поведенческие — быстрая шпаргалка

### Strategy ← частый вопрос

**Проблема:** огромный `if-else` по типу — каждый новый тип требует изменения существующего кода.

```java
// ❌ Без Strategy
public void process(String type) {
    if (type.equals("card")) { /* ... */ }
    else if (type.equals("paypal")) { /* ... */ }
    else if (type.equals("crypto")) { /* добавили новый — меняем метод */ }
}

// ✅ Со Strategy — каждый тип в отдельном @Component
public interface PaymentStrategy {
    PaymentResult pay(BigDecimal amount);
    String getType();
}

@Service
public class PaymentService {
    private final Map<String, PaymentStrategy> strategies;

    // Spring инжектирует ВСЕ реализации PaymentStrategy
    public PaymentService(List<PaymentStrategy> list) {
        this.strategies = list.stream()
            .collect(Collectors.toMap(PaymentStrategy::getType, s -> s));
    }

    public PaymentResult process(String type, BigDecimal amount) {
        return strategies.get(type).pay(amount); // никакого if-else!
    }
}
// Добавить новый тип = создать новый @Component. Существующий код не меняется. OCP ✅
```

**Пример из JDK:** `Comparator` — это стратегия сортировки.

---

### Observer ← частый вопрос

**Проблема:** одно событие должно инициировать несколько независимых реакций.

```java
// ❌ Без Observer — OrderService знает всех получателей
public void placeOrder(Order order) {
    orderRepo.save(order);
    emailService.sendConfirmation(order);    // жёсткая зависимость
    inventoryService.reserve(order);         // добавить нового получателя = менять метод
    analyticsService.track(order);
}

// ✅ С Observer через Spring Events
@Service
public class OrderService {
    private final ApplicationEventPublisher events;

    public Order placeOrder(OrderRequest request) {
        Order order = orderRepo.save(Order.from(request));
        events.publishEvent(new OrderCreatedEvent(order)); // публикуем, не знаем кто слушает
        return order;
    }
}

// Подписчики — независимые @Component
@Component
public class EmailListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendConfirmation(event.order());
    }
}

@Component
public class InventoryListener {
    @EventListener @Async
    public void onOrderCreated(OrderCreatedEvent event) {
        inventoryService.reserve(event.order());
    }
}
// Добавить нового подписчика = создать новый @Component. OrderService не меняется. OCP ✅
```

---

### Command

**Проблема:** нужно инкапсулировать действие как объект — для очереди, undo/redo, логирования.

```java
public interface Command { void execute(); void undo(); }

public class InsertTextCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final String text;

    public void execute() { editor.insertText(position, text); }
    public void undo()    { editor.deleteText(position, text.length()); }
}

// CommandHistory — стек с undo
public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();
    public void execute(Command cmd) { cmd.execute(); history.push(cmd); }
    public void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```

**В Spring:** `Runnable`/`Callable` в `ExecutorService` — это Command. `@Async` методы — Command выполняется асинхронно.

---

### Template Method

**Проблема:** алгоритм с фиксированной структурой, но шаги варьируются.

```java
// Абстрактный класс фиксирует алгоритм, шаги — переопределяются
public abstract class DataExporter {
    public final void export(List<?> data, String dest) { // final — не переопределить
        List<String> formatted = formatData(data);    // abstract — переопределить
        saveToDestination(formatted, dest);            // abstract — переопределить
        onExportComplete(dest);                        // hook — опционально
    }
    protected abstract List<String> formatData(List<?> data);
    protected abstract void saveToDestination(List<String> data, String dest);
    protected void onExportComplete(String dest) { } // default: ничего
}
```

**В Spring:** `JdbcTemplate` — фиксированный алгоритм (соединение → запрос → закрытие), вариативная часть — твой SQL и `RowMapper`. Все `Abstract*` классы в Spring используют этот паттерн.

---

### Chain of Responsibility

**Проблема:** запрос должен пройти через цепочку обработчиков, каждый решает сам — обрабатывать или передать дальше.

```java
// Spring Security — цепочка фильтров
http
    .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
// Request → CorsFilter → JwtAuthFilter → AuthorizationFilter → Controller
```

**Паттерн:** каждый `Filter` — обработчик. Передаёт управление через `chain.doFilter()` или останавливает (401/403).

---

### State

**Проблема:** поведение объекта зависит от его состояния, переходы — сложный switch.

```java
// State через enum с методами — чисто и компактно
public enum OrderStatus {
    CREATED {
        public OrderStatus pay()    { return PAID; }
        public OrderStatus cancel() { return CANCELLED; }
        public OrderStatus ship()   { throw new IllegalStateException("Pay first"); }
    },
    PAID {
        public OrderStatus pay()    { throw new IllegalStateException("Already paid"); }
        public OrderStatus ship()   { return SHIPPED; }
        public OrderStatus cancel() { return CANCELLED; }
    },
    SHIPPED {
        public OrderStatus pay()    { throw new IllegalStateException(); }
        public OrderStatus ship()   { throw new IllegalStateException(); }
        public OrderStatus cancel() { throw new IllegalStateException("Already shipped"); }
    },
    CANCELLED {
        public OrderStatus pay()    { throw new IllegalStateException(); }
        public OrderStatus ship()   { throw new IllegalStateException(); }
        public OrderStatus cancel() { throw new IllegalStateException(); }
    };

    public abstract OrderStatus pay();
    public abstract OrderStatus ship();
    public abstract OrderStatus cancel();
}

// Использование
OrderStatus status = OrderStatus.CREATED;
status = status.pay();   // → PAID
status = status.ship();  // → SHIPPED
```

---

## 🔹 Антипаттерны — знать обязательно

```java
// ❌ God Object — класс знает и делает всё
@Service
public class OrderGodService {
    // 50 методов: создание, оплата, доставка, аналитика, email...
    // Нарушает SRP. Решение: разбить по доменам.
}

// ❌ Anemic Domain Model — объекты только с геттерами, логика в Service
@Entity
public class Order {
    private OrderStatus status;
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus s) { this.status = s; } // нет логики!
}
@Service
public class OrderService {
    public void pay(Order order) {
        if (order.getStatus() == CREATED) order.setStatus(PAID); // логика в сервисе!
    }
}
// ✅ Rich Domain Model — логика внутри объекта
@Entity
public class Order {
    public void pay() {
        if (status != CREATED) throw new IllegalStateException();
        this.status = PAID;
        registerEvent(new OrderPaidEvent(this));
    }
}

// ❌ Service Locator — скрытые зависимости
public class OrderService {
    public void process() {
        UserService users = AppContext.getBean(UserService.class); // скрытая зависимость!
    }
}
// ✅ Dependency Injection — явные зависимости
@Service @RequiredArgsConstructor
public class OrderService {
    private final UserService userService; // явно видно что нужно
}
```

---

## 🔹 Паттерны в Spring — итоговая карта

| Паттерн | Где в Spring |
|---------|-------------|
| Singleton | `@Component` (scope=singleton по умолчанию) |
| Factory Method | `@Bean` методы в `@Configuration` |
| Abstract Factory | `@Configuration` + `@Profile` |
| Builder | `ResponseEntity.builder()`, `UriComponentsBuilder` |
| Prototype | `@Scope("prototype")` |
| **Proxy** | AOP: `@Transactional`, `@Cacheable`, `@Secured` |
| Decorator | `OncePerRequestFilter`, `Collections.unmodifiable*` |
| Adapter | `HandlerAdapter`, `MessageConverter` |
| **Facade** | `@Service` оркестраторы |
| Composite | `SecurityFilterChain`, компонентное дерево |
| **Strategy** | `Map<String, Strategy>` injection, `@Primary`/`@Qualifier` |
| **Observer** | `ApplicationEventPublisher` + `@EventListener` |
| Command | `@Async`, `Runnable`/`Callable` в `ExecutorService` |
| Template Method | `JdbcTemplate`, `RestTemplate`, `Abstract*` классы |
| Chain of Resp. | `SecurityFilterChain`, `HandlerInterceptor` |
| State | `OrderStatus` enum с абстрактными методами |

---

## 🔹 Типичные вопросы и ответы

**Q: Чем Proxy отличается от Decorator?**
A: Proxy контролирует **доступ** к объекту — клиент обычно не знает что работает с прокси (Spring AOP). Decorator **добавляет функциональность** — клиент сам стекирует декораторы (Java I/O). Proxy создаётся инфраструктурой, Decorator — самим клиентом.

**Q: Чем Strategy отличается от Template Method?**
A: Template Method использует **наследование** — алгоритм в базовом классе, шаги переопределяются. Strategy использует **композицию** — алгоритм передаётся как объект. Strategy предпочтительнее (composition over inheritance).

**Q: Почему Singleton — антипаттерн?**
A: Глобальное состояние = скрытые зависимости (не видно из сигнатуры что класс нужен), сложно тестировать (нельзя подменить mock), нарушает SRP (класс сам управляет своим жизненным циклом). В Spring: используй `@Component`, контейнер управляет singleton-ом.

**Q: Что такое self-invocation в Spring AOP?**
A: Вызов `this.method()` внутри класса минует Proxy — аннотации `@Transactional`, `@Cacheable` не сработают. Решение: вынести метод в другой бин, или инжектировать сам себя (`@Autowired` self-injection), или использовать `AopContext.currentProxy()`.

**Q: В чём разница Observer и Event-Driven?**
A: Observer — синхронный, одна JVM, субъект держит список наблюдателей. Event-Driven (Kafka, RabbitMQ) — асинхронный, разные сервисы, broker хранит события. `ApplicationEventPublisher` в Spring — Observer внутри JVM. `@Async @EventListener` — промежуточный вариант (асинхронно но в рамках одного приложения).

**Q: Когда Factory Method, когда Abstract Factory?**
A: Factory Method — создать один объект, конкретный тип определяется подклассом. Abstract Factory — создать **семейство связанных объектов** (кнопка + чекбокс + поле ввода для одной платформы). Abstract Factory обычно содержит несколько Factory Methods.

---

## 🔹 Шпаргалка

```
Порождающие (КАК создавать):
  Singleton   — один экземпляр. Enum > Holder > DCL. В Spring: @Component.
  Factory     — скрыть логику создания. В Spring: @Bean методы.
  Abstract F. — семейство объектов. В Spring: @Configuration + @Profile.
  Builder     — много параметров, часть опциональна. Lombok @Builder.
  Prototype   — дешёвое клонирование. В Spring: @Scope("prototype").

Структурные (КАК организовать):
  Proxy       — контроль доступа. В Spring: @Transactional, @Cacheable (AOP).
                ⚠️ self-invocation = proxy не срабатывает!
  Decorator   — стекируемые обёртки. Java I/O, Collections.unmodifiable*.
  Adapter     — совместить несовместимые интерфейсы. HandlerAdapter.
  Facade      — простое API над сложной подсистемой. @Service оркестраторы.
  Composite   — деревья объектов. Лист и ветвь за одним интерфейсом.

Поведенческие (КАК взаимодействовать):
  Strategy    — убрать if-else. Map<String, Strategy> + @Component в Spring.
  Observer    — одно событие → много реакций. ApplicationEventPublisher.
  Command     — действие как объект. Undo/redo, @Async, ExecutorService.
  Template M. — алгоритм в базовом, шаги в подклассах. JdbcTemplate.
  Chain of R. — цепочка обработчиков. SecurityFilterChain.
  State       — поведение по состоянию. Enum с абстрактными методами.

Антипаттерны:
  God Object       — один класс делает всё (нарушает SRP)
  Anemic Model     — объекты без логики (логика размазана по Service)
  Service Locator  — скрытые зависимости (используй DI)

Proxy vs Decorator: Proxy = контроль доступа, Decorator = добавить функциональность
Strategy vs Template Method: Strategy = композиция, Template Method = наследование
```
