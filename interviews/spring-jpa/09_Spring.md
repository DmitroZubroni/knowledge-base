# Spring

> **Теги:** #interviews #spring #ioc #di #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 IoC (Inversion of Control)

**IoC** — инверсия управления, принцип когда управление созданием и зависимостями передаётся контейнеру.

### Без IoC

```java
public class UserService {
    private UserRepository repository = new UserRepository();  // жёсткая связь
}
```

### С IoC

```java
public class UserService {
    private UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;  // внедрение зависимости
    }
}
```

**Преимущества:**
- Слабая связность (loose coupling)
- Легкое тестирование (можно подменить зависимости)
- Гибкость конфигурации

---

## 🔹 DI (Dependency Injection)

**DI** — внедрение зависимостей, реализация IoC.

### Способы внедрения

#### 1. Constructor Injection (рекомендуется)

```java
@Component
public class UserService {
    private final UserRepository repository;

    @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

#### 2. Field Injection (не рекомендуется)

```java
@Component
public class UserService {
    @Autowired
    private UserRepository repository;  // сложно тестировать
}
```

#### 3. Setter Injection

```java
@Component
public class UserService {
    private UserRepository repository;

    @Autowired
    public void setRepository(UserRepository repository) {
        this.repository = repository;
    }
}
```

### Сравнение

| Способ | Тестирование | Immutability | Рекомендация |
|--------|--------------|--------------|--------------|
| **Constructor** | Легко | Да (final) | ✅ Рекомендуется |
| **Field** | Сложно | Нет | ❌ Не рекомендуется |
| **Setter** | Средне | Нет | Опционально |

---

## 🔹 Bean Lifecycle

**Lifecycle** — жизненный цикл bean в Spring контейнере.

### Этапы

```
1. Instantiate (создание экземпляра)
2. Populate properties (внедрение зависимостей)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. @PostConstruct
7. InitializingBean.afterPropertiesSet()
8. Custom init-method
9. Bean готов к использованию
10. @PreDestroy
11. DisposableBean.destroy()
12. Custom destroy-method
```

### Пример

```java
@Component
public class UserService implements InitializingBean, DisposableBean {
    
    @PostConstruct
    public void init() {
        System.out.println("PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("afterPropertiesSet");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("PreDestroy");
    }

    @Override
    public void destroy() {
        System.out.println("destroy");
    }
}
```

### @PostConstruct vs InitializingBean

| Способ | Когда использовать |
|--------|-------------------|
| **@PostConstruct** | Рекомендуется (JSR-250) |
| **InitializingBean** | Если нужна совместимость со старым кодом |

---

## 🔹 Bean Scopes

**Scope** — область видимости bean.

### Основные scopes

| Scope | Описание | Когда использовать |
|-------|----------|-------------------|
| **singleton** | Один экземпляр на контекст (default) | Stateless сервисы |
| **prototype** | Новый экземпляр при каждом запросе | Stateful объекты |
| **request** | Один экземпляр на HTTP запрос | Web приложения |
| **session** | Один экземпляр на HTTP сессию | Web приложения |

### Примеры

```java
@Component
@Scope("singleton")  // default
public class UserService {
    // один экземпляр на всё приложение
}

@Component
@Scope("prototype")
public class UserSession {
    // новый экземпляр при каждом getBean()
}
```

### Singleton vs Prototype

```java
// Singleton — один и тот же объект
UserService service1 = context.getBean(UserService.class);
UserService service2 = context.getBean(UserService.class);
service1 == service2;  // true

// Prototype — разные объекты
UserSession session1 = context.getBean(UserSession.class);
UserSession session2 = context.getBean(UserSession.class);
session1 == session2;  // false
```

> [!warning] Prototype bean в Singleton bean
```java
@Component
@Scope("singleton")
public class SingletonService {
    @Autowired
    private PrototypeBean prototype;  // будет создан один раз!
}
```

Решение: `@Lookup` или `ObjectProvider`

---

## 🔹 @Component vs @Bean

### @Component

Аннотация на классе, Spring создаёт bean автоматически.

```java
@Component
public class UserService {
    // Spring создаёт bean через reflection
}
```

### @Bean

Аннотация на методе в @Configuration классе, мы создаём bean программно.

```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService(UserRepository repository) {
        return new UserService(repository);
    }

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/db");
        return ds;
    }
}
```

### Сравнение

| Характеристика | @Component | @Bean |
|----------------|------------|-------|
| Где использовать | На наших классах | В @Configuration методах |
| Кто создаёт | Spring (reflection) | Мы (программно) |
| Когда использовать | Наши классы | Third-party классы, сложная логика |

### Стереотипные аннотации

| Аннотация | Назначение |
|----------|------------|
| **@Component** | Общий компонент |
| **@Service** | Сервисный слой (бизнес-логика) |
| **@Repository** | DAO слой (работа с БД) |
| **@Controller** | MVC контроллер (Spring MVC) |

---

## 🔹 @ComponentScan

**@ComponentScan** — указывает Spring где искать @Component аннотированные классы.

### Использование

```java
@SpringBootApplication
@ComponentScan(basePackages = "com.example")
public class Application {
    // сканирует com.example и подпакеты
}
```

### По умолчанию

```java
@SpringBootApplication
// автоматически сканирует пакет класса и подпакеты
public class Application {
}
```

### Фильтрация

```java
@ComponentScan(
    basePackages = "com.example",
    includeFilters = @ComponentScan.Filter(type = FilterType.REGEX, pattern = ".*Service.*"),
    excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Controller.class)
)
```

---

## 🔹 Proxy Pattern в Spring

**Proxy** — прокси-объект, который перехватывает вызовы методов (AOP).

### JDK Dynamic Proxy

Для интерфейсов.

```java
public interface UserService {
    User getUser(Long id);
}

@Component
public class UserServiceImpl implements UserService {
    @Override
    public User getUser(Long id) {
        return new User();
    }
}

// Spring создаёт JDK Dynamic Proxy
```

### CGLIB Proxy

Для классов (без интерфейса).

```java
@Component
public class UserService {
    public User getUser(Long id) {
        return new User();
    }
}

// Spring создаёт CGLIB Proxy
```

### Сравнение

| Тип | Требования | Когда используется |
|-----|------------|-------------------|
| **JDK Proxy** | Нужен интерфейс | Если есть интерфейс |
| **CGLIB Proxy** | Класс не final | Если нет интерфейса |

### @Transactional и Proxy

```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        // Spring создаёт прокси для управления транзакцией
        repository.save(user);
    }
}
```

> [!warning] Проблема self-invocation
```java
@Service
public class UserService {
    @Transactional
    public void methodA() {
        methodB();  // @Transactional не сработает!
    }

    @Transactional
    public void methodB() {
        // ...
    }
}
```

Решение: внедрить себя через `@Lazy` или вынести в отдельный bean.

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **IoC** — контейнер создаёт объекты
> - **DI** — Constructor Injection лучше (final, тестируемость)
> - **Lifecycle:** @PostConstruct → ready → @PreDestroy
> - **Scopes:** singleton (default), prototype, request, session
> - **@Component** — на классах, **@Bean** — в @Configuration
> - **@ComponentScan** — сканирование пакетов
> - **Proxy:** JDK (интерфейсы), CGLIB (классы)

```
IoC/DI:
контейнер создаёт и внедряет зависимости
Constructor Injection → final поля, легко тестировать

Lifecycle:
@PostConstruct → afterPropertiesSet → ready
@PreDestroy → destroy

Scopes:
singleton → один экземпляр
prototype → новый при запросе

@Component vs @Bean:
@Component → на классах (Spring создаёт)
@Bean → в @Configuration (мы создаём)

Proxy:
JDK Proxy → интерфейсы
CGLIB Proxy → классы
```
