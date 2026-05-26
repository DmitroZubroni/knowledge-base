# Spring — Core (IoC, DI, Container)

> **Теги:** #spring #framework #ioc #di #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_Spring_Index]]

---

## 🔹 Что такое Spring Framework

> [!note] Определение
> **Spring Framework** — экосистема для enterprise Java: инверсия управления, аспектно-ориентированное программирование, абстракции над JDBC, ORM, веб, security, messaging.

**Зачем нужен:** убирает boilerplate (создание объектов, связывание зависимостей, транзакции, конфигурация), даёт единый стиль и тестируемость.

```
┌─────────────────────────────────────────────────────────┐
│                  Spring Framework                        │
├─────────────┬─────────────┬──────────────┬──────────────┤
│    Core     │     Web     │     Data     │   Security   │
│  IoC / AOP  │  MVC / REST │  JDBC / ORM  │  Auth / ACL  │
├─────────────┴─────────────┴──────────────┴──────────────┤
│  Test · Messaging · Integration · Batch · ...           │
└─────────────────────────────────────────────────────────┘
         ▲
         │ Spring Boot — opinionated auto-config поверх Core
```

---

## 🔹 IoC (Inversion of Control)

> [!note] Определение
> **IoC** — принцип: объект не создаёт свои зависимости сам, а получает их извне. Контейнер Spring **инвертирует** ответственность за создание и wiring.

### Ручное создание vs IoC

```java
// ❌ Без IoC — жёсткая связь, сложно тестировать
public class OrderService {
    private final OrderRepository repo = new JdbcOrderRepository();
}

// ✅ С IoC — зависимость приходит извне
public class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }
}
```

> [!tip] IoC ≠ паттерн
> IoC — **концепция**. DI — один из способов реализации IoC. Service Locator — другой (в Spring почти не используется).

---

## 🔹 DI (Dependency Injection)

> [!note] Определение
> **DI** — контейнер внедряет зависимости в bean после создания (или через конструктор).

| Способ | Как | Когда |
|--------|-----|-------|
| **Constructor** | Параметры конструктора + `@Autowired` (опционально с Boot) | ✅ Предпочтительно: immutable, обязательные deps, легко тестировать |
| **Setter** | Методы `setXxx()` + `@Autowired` | Опциональные зависимости, legacy-код |
| **Field** | `@Autowired` на поле | ❌ Избегать в production-коде |

### ❌ Field injection

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;  // нельзя сделать final, скрыта зависимость
}
```

### ✅ Constructor injection

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

> [!tip] Lombok
> `@RequiredArgsConstructor` + `private final` поля — эквивалент constructor injection без boilerplate.

---

## 🔹 ApplicationContext

> [!note] Определение
> **BeanFactory** — базовый контейнер: создание bean, DI.  
> **ApplicationContext** — расширение: events, i18n, resource loading, AOP, автоматический post-processing.

| Возможность | BeanFactory | ApplicationContext |
|-------------|:-----------:|:------------------:|
| Создание / DI bean | ✅ | ✅ |
| BeanPostProcessor | ✅ | ✅ (авто-регистрация) |
| ApplicationEvent | ❌ | ✅ |
| MessageSource (i18n) | ❌ | ✅ |
| ResourceLoader | ❌ | ✅ |
| Eager init singletons | ❌ (lazy по умолчанию) | ✅ при старте |

**Типы контекстов:**

| Контекст | Назначение |
|----------|------------|
| `AnnotationConfigApplicationContext` | JavaConfig (`@Configuration`) |
| `ClassPathXmlApplicationContext` | XML (legacy) |
| `AnnotationConfigWebApplicationContext` | Web-приложение без Boot |
| Spring Boot | `SpringApplication` создаёт контекст автоматически |

---

## 🔹 Bean Lifecycle

```
instantiate (конструктор)
    ↓
populate properties (DI)
    ↓
BeanNameAware.setBeanName()
    ↓
BeanFactoryAware.setBeanFactory()
    ↓
ApplicationContextAware.setApplicationContext()
    ↓
@PostConstruct (или InitializingBean.afterPropertiesSet())
    ↓
custom init-method (@Bean(initMethod = "..."))
    ↓
┌──────────────────────────────────────┐
│         Bean готов к работе          │
└──────────────────────────────────────┘
    ↓ (shutdown контекста)
@PreDestroy (или DisposableBean.destroy())
    ↓
custom destroy-method
```

```java
@Component
public class CacheWarmer {

    @PostConstruct
    public void warmUp() {
        // вызывается после полного DI
    }

    @PreDestroy
    public void shutdown() {
        // вызывается перед уничтожением bean
    }
}
```

> [!warning] `@PostConstruct` vs `afterPropertiesSet`
> Оба после injection. `InitializingBean` привязывает код к Spring API — предпочтительнее JSR-250 `@PostConstruct` или `initMethod` в `@Bean`.

---

## 🔹 Scopes

| Scope | Описание | Когда |
|-------|----------|-------|
| **singleton** (default) | Один экземпляр на контейнер | Сервисы, репозитории, stateless beans |
| **prototype** | Новый экземпляр при каждом `getBean()` | Stateful, тяжёлые объекты на запрос |
| **request** | Один bean на HTTP-запрос (web) | Данные запроса |
| **session** | Один bean на HTTP-сессию | Корзина, пользователь сессии |
| **application** | Один на ServletContext | Глобальные web-данные |

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContextHolder { }
```

> [!warning] Singleton → Prototype
> Singleton bean не получит новый prototype при каждый вызов метода — нужен `@Lookup` или `ObjectProvider<PrototypeBean>`.

---

## 🔹 Конфигурация

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| **XML** | Явно, без перекомпиляции | Verbose, нет type-safety |
| **JavaConfig** | Type-safe, рефакторинг IDE | Больше кода для простых bean |
| **Component Scan** | Минимум boilerplate | «Магия» — неочевидно что в контексте |

```java
@Configuration
@ComponentScan(basePackages = "com.example.app")
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

- `@Configuration` — класс с `@Bean`-методами (CGLIB-proxy для singleton `@Bean`)
- `@ComponentScan` — автоматическая регистрация `@Component` и стереотипов
- `@Import` — подключение других конфигураций

---

## 🔹 Аннотации стереотипов

| Аннотация | Семантика | Слой |
|----------|-----------|------|
| `@Component` | Общий Spring-компонент | Любой |
| `@Service` | Бизнес-логика | Service |
| `@Repository` | Доступ к данным + translation исключений persistence | DAO |
| `@Controller` | Web MVC, возвращает view name | Presentation |
| `@RestController` | `@Controller` + `@ResponseBody` на классе | REST API |

> [!note] Разница только семантическая
> Технически все — `@Component` с разными именами для читаемости и AOP (например, `@Repository` → `PersistenceExceptionTranslationPostProcessor`).

---

## 🔹 @Autowired, @Qualifier, @Primary

```java
public interface PaymentGateway { }

@Service
@Primary
public class StripeGateway implements PaymentGateway { }

@Service
public class PayPalGateway implements PaymentGateway { }

@Service
public class CheckoutService {
    private final PaymentGateway gateway;

    // без @Qualifier — возьмёт @Primary (Stripe)
    public CheckoutService(PaymentGateway gateway) { this.gateway = gateway; }

    // явный выбор
    public CheckoutService(@Qualifier("payPalGateway") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

| Ситуация | Решение |
|----------|---------|
| Несколько реализаций | `@Qualifier("beanName")` или `@Primary` на одной |
| Optional dependency | `Optional<T>`, `@Autowired(required = false)`, `ObjectProvider<T>` |
| Коллекция всех реализаций | `List<PaymentGateway>` — Spring внедрит все bean |

---

## 🔹 @Value и @PropertySource

```java
@Configuration
@PropertySource("classpath:app.properties")
public class MailConfig {

    @Value("${mail.host}")
    private String host;

    @Value("${mail.port:587}")  // default 587
    private int port;

    @Value("#{systemProperties['user.home']}")  // SpEL
    private String userHome;
}
```

**SpEL** (Spring Expression Language): `#{...}` — выражения над bean, properties, системными свойствами.

```properties
# app.properties
app.name=MyService
app.max-retries=3
```

---

## 🔹 Profiles

```java
@Configuration
@Profile("dev")
public class DevDataSourceConfig { }

@Configuration
@Profile("prod")
public class ProdDataSourceConfig { }
```

```yaml
# application.yml
spring:
  profiles:
    active: dev
```

```bash
java -jar app.jar --spring.profiles.active=prod
```

> [!tip] Типичное использование
> `dev` — H2, mock; `prod` — PostgreSQL, реальные credentials; `test` — изолированная конфигурация для CI.

---

## 🔹 Events

```java
// событие
public class OrderCreatedEvent extends ApplicationEvent {
    private final Long orderId;
    public OrderCreatedEvent(Object source, Long orderId) {
        super(source);
        this.orderId = orderId;
    }
    public Long getOrderId() { return orderId; }
}

// публикация
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public void createOrder(Order order) {
        // ...
        publisher.publishEvent(new OrderCreatedEvent(this, order.getId()));
    }
}

// обработка
@Component
public class OrderEventListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // отправка email, метрики...
    }

    @EventListener
    @Async  // нужен @EnableAsync
    public void onOrderCreatedAsync(OrderCreatedEvent event) { }
}
```

> [!note] Синхронно по умолчанию
> `@EventListener` выполняется в потоке публикатора, если не `@Async`.

---

## 🔹 Итог

1. **IoC** — контейнер управляет жизненным циклом и зависимостями.
2. **Constructor injection** — предпочтительный способ DI.
3. **ApplicationContext** — основной API контейнера в приложениях.
4. **Lifecycle** — `@PostConstruct` / `@PreDestroy` для init/shutdown.
5. **Scopes** — singleton по умолчанию; prototype для stateful; request/session в web.
6. **JavaConfig + Component Scan** — стандарт современных проектов.
7. **@Primary / @Qualifier** — разрешение конфликтов bean.
8. **Profiles** — разные конфигурации по окружению.
9. **Events** — слабая связь между компонентами.

```
Шпаргалка Core:
─────────────────────────────────────────
@Configuration + @Bean          → явные bean
@ComponentScan                  → авто-обнаружение
@Service / @Repository          → стереотипы
Constructor injection           → ✅ deps
@Value("${key}")                → properties
@Profile("dev")                 → окружение
@EventListener                  → обработка событий
singleton (default) | prototype → scopes
@Primary | @Qualifier           → выбор bean
```
