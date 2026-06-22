# Spring — Core (IoC, DI, Container)

> **Теги:** #spring #framework #ioc #di #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Spring_Index]]

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

---

## 🔹 Жизненный цикл бина — подробно

### BeanPostProcessor — перехват каждого бина

`BeanPostProcessor` позволяет вмешаться в инициализацию **каждого** бина в контейнере. Именно через него работают `@Autowired`, `@PostConstruct`, AOP-прокси.

```java
public interface BeanPostProcessor {
    // вызывается ДО @PostConstruct / afterPropertiesSet
    Object postProcessBeforeInitialization(Object bean, String beanName);

    // вызывается ПОСЛЕ @PostConstruct / afterPropertiesSet
    Object postProcessAfterInitialization(Object bean, String beanName);
}
```

**Полный порядок жизненного цикла с BeanPostProcessor:**

```
1.  Создание экземпляра (конструктор)
2.  Populate properties (DI: @Autowired поля, setter injection)
3.  BeanNameAware.setBeanName()
4.  BeanFactoryAware.setBeanFactory()
5.  ApplicationContextAware.setApplicationContext()
6.  BeanPostProcessor.postProcessBeforeInitialization()  ← @PostConstruct обрабатывается здесь
7.  InitializingBean.afterPropertiesSet()
8.  @Bean(initMethod = "...")
9.  BeanPostProcessor.postProcessAfterInitialization()   ← AOP Proxy создаётся здесь!
    ↓
    Bean готов к использованию
    ↓
10. @PreDestroy (при shutdown контекста)
11. DisposableBean.destroy()
12. @Bean(destroyMethod = "...")
```

> [!note] AOP Proxy создаётся в postProcessAfterInitialization
> `AnnotationAwareAspectJAutoProxyCreator` — это `BeanPostProcessor`. Он обёртывает бин в CGLIB или JDK proxy на шаге 9. Именно поэтому `@Transactional` / `@Cacheable` работают: к моменту когда бин попадает в контейнер — он уже является прокси.

```java
// Свой BeanPostProcessor — пример логирования создания бинов
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        log.debug("Before init: {}", beanName);
        return bean; // обязательно вернуть bean (можно вернуть другой объект!)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        log.debug("After init: {} class={}", beanName, bean.getClass().getSimpleName());
        return bean;
    }
}
```

> [!warning] BeanPostProcessor и circular dependencies
> BPP регистрируются и создаются раньше обычных бинов. Если BPP зависит от обычного бина — возможна проблема с circular dependency. Spring предупредит в логах: "Bean '...' is not eligible for getting processed by all BeanPostProcessors".

---

### BeanFactoryPostProcessor — изменение метаданных бинов

Работает **до создания** бинов — изменяет их определения (BeanDefinition).

```java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
}
```

**Встроенные реализации:**
- `PropertySourcesPlaceholderConfigurer` — подставляет `${property}` из файлов properties
- `ConfigurationClassPostProcessor` — обрабатывает `@Configuration`, `@Bean`, `@ComponentScan`

```java
// Свой BeanFactoryPostProcessor — заменить scope всех бинов на prototype
@Component
public class PrototypeScopePostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        for (String name : factory.getBeanDefinitionNames()) {
            BeanDefinition bd = factory.getBeanDefinition(name);
            bd.setScope(BeanDefinition.SCOPE_PROTOTYPE); // изменить scope
        }
    }
}
```

---

### Как работает @Configuration с CGLIB

Когда `@Bean` методы вызываются несколько раз — в обычном классе каждый вызов создаст новый объект. Spring решает это через CGLIB.

```java
@Configuration
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // вызываем serviceB() внутри
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

Spring создаёт **CGLIB подкласс** `AppConfig`. В нём `serviceB()` перехватывается: при повторном вызове возвращается уже созданный singleton из контейнера, а не новый объект.

```
Вызов serviceA() → serviceB() внутри → CGLIB перехватил
                                       → getBean("serviceB") из контейнера
                                       → тот же экземпляр
```

> [!note] @Configuration vs @Component для @Bean
> ```java
> @Configuration  // CGLIB proxy — межбиновые вызовы возвращают singleton
> class Config {
>     @Bean ServiceA a() { return new ServiceA(b()); } // b() → singleton из контейнера
>     @Bean ServiceB b() { return new ServiceB(); }
> }
>
> @Component      // НЕТ CGLIB proxy — обычный класс
> class Config {
>     @Bean ServiceA a() { return new ServiceA(b()); } // b() → НОВЫЙ объект каждый раз!
>     @Bean ServiceB b() { return new ServiceB(); }
> }
> ```
> Используй `@Configuration` когда `@Bean` методы вызывают другие `@Bean` методы.

---

### Порядок создания бинов — @DependsOn и @Lazy

```java
// @DependsOn — явно указать что бин должен создаться после другого
@Component
@DependsOn("databaseInitializer")
public class UserRepository { }

// @Lazy — создать бин при первом обращении, а не при старте контекста
@Service
@Lazy
public class HeavyService { }

// @Lazy на точке инжекции — инжектировать proxy, создать при первом вызове
@Service
public class OrderService {
    @Lazy
    @Autowired
    private HeavyService heavyService; // создастся при первом вызове метода
}
```

---

### ObjectProvider — ленивое и опциональное получение бинов

```java
@Service
public class ReportService {

    // Не бросит исключение если бин не найден
    private final ObjectProvider<PdfRenderer> pdfRenderer;

    public ReportService(ObjectProvider<PdfRenderer> pdfRenderer) {
        this.pdfRenderer = pdfRenderer;
    }

    public byte[] render(Report report) {
        return pdfRenderer.getIfAvailable(() -> new DefaultPdfRenderer())
                          .render(report);
    }
}
```

`ObjectProvider<T>` — ленивый, не бросает исключений при отсутствии бина, поддерживает несколько реализаций:

```java
pdfRenderer.getIfAvailable()          // null если нет
pdfRenderer.getIfAvailable(defaultFn) // default supplier если нет
pdfRenderer.ifAvailable(consumer)     // выполнить если есть
pdfRenderer.stream()                  // все реализации как Stream
```
