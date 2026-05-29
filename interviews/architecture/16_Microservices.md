# Microservices

> **Теги:** #interviews #architecture #microservices #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Монолит vs Микросервисы

### Монолит

**Монолит** — единое приложение, все модули в одном коде.

```
┌─────────────────────────────┐
│         Монолит             │
├─────────────────────────────┤
│  UI  │  Business  │  Data   │
├─────────────────────────────┤
│  Orders  │  Users  │  Auth   │
├─────────────────────────────┤
│        Shared Database       │
└─────────────────────────────┘
```

**Преимущества:**
- Простота разработки и деплоя
- Легкое тестирование
- Нет сетевых задержек
- Единая транзакция

**Недостатки:**
- Сложность масштабирования (масштабируется всё)
- Зависимость технологий (один язык/фреймворк)
- Медленный CI/CD (пересборка всего)
- Single point of failure

### Микросервисы

**Микросервисы** — набор небольших независимых сервисов.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Orders  │  │  Users   │  │   Auth   │
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
      └─────────────┴─────────────┘
                    │
           ┌────────┴────────┐
           │  API Gateway    │
           └─────────────────┘
```

**Преимущества:**
- Независимое масштабирование
- Разные технологии для разных сервисов
- Быстрый CI/CD (пересборка только изменившегося)
- Изоляция сбоев

**Недостатки:**
- Сложность распределённых систем
- Сетевые задержки
- Распределённые транзакции
- Сложность отладки и мониторинга

### Сравнение

| Характеристика | Монолит | Микросервисы |
|----------------|---------|--------------|
| Масштабирование | Всё вместе | По отдельности |
| Технологии | Одни | Разные |
| CI/CD | Медленный | Быстрый |
| Сложность | Низкая | Высокая |
| Транзакции | Локальные | Распределённые |

---

## 🔹 API Gateway

**API Gateway** — единая точка входа для всех микросервисов.

### Функции

- Роутинг запросов к нужному сервису
- Аутентификация и авторизация
- Rate limiting
- Load balancing
- Logging и мониторинг
- Response aggregation

### Варианты реализации

#### 1. Spring Cloud Gateway

```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}

@Configuration
public class GatewayConfig {
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("users", r -> r
                .path("/api/users/**")
                .uri("lb://users-service"))
            .route("orders", r -> r
                .path("/api/orders/**")
                .uri("lb://orders-service"))
            .build();
    }
}
```

#### 2. Netflix Zuul (устарел)

```java
@SpringBootApplication
@EnableZuulProxy
public class GatewayApplication {
    // Zuul 1.x устарел, рекомендуется Spring Cloud Gateway
}
```

#### 3. Kong

```yaml
# Kong configuration
services:
  - name: users-service
    url: http://users-service:8080

routes:
  - name: users-route
    service: users-service
    paths:
      - /api/users
```

#### 4. Nginx

```nginx
upstream users_service {
    server users-service:8080;
}

upstream orders_service {
    server orders-service:8080;
}

server {
    listen 80;
    
    location /api/users {
        proxy_pass http://users_service;
    }
    
    location /api/orders {
        proxy_pass http://orders_service;
    }
}
```

### Сравнение

| Gateway | Преимущества | Недостатки |
|---------|--------------|------------|
| **Spring Cloud Gateway** | Reactive, Java-based | Требует JVM |
| **Kong** | Высокая производительность, плагины | Сложная настройка |
| **Nginx** | Высокая производительность | Ограниченная динамичность |

---

## 🔹 Saga Pattern

**Saga** — паттерн для управления распределёнными транзакциями.

### Проблема распределённых транзакций

```java
// Сервис 1
@Transactional
public void createOrder() {
    orderRepository.save(order);
    // вызов сервиса 2
    paymentService.processPayment();
}

// Сервис 2
@Transactional
public void processPayment() {
    paymentRepository.save(payment);
    // вызов сервиса 3
    inventoryService.reserveItems();
}
```

Проблема: если `inventoryService` упал — откатить всё сложно.

### Saga Pattern решения

#### 1. Choreography (Хореография)

Каждый сервис публикует события, другие подписываются.

```
Order Service → OrderCreated event
    ↓
Payment Service → PaymentProcessed event
    ↓
Inventory Service → ItemsReserved event
    ↓
Order Service → OrderCompleted event

Если ошибка → Compensation events
```

**Преимущества:**
- Децентрализованный
- Гибкость

**Недостатки:**
- Сложность отладки
- Циклические зависимости

#### 2. Orchestration (Оркестрация)

Центральный координатор управляет процессом.

```
Saga Orchestrator
    ↓
1. Order Service → createOrder()
    ↓
2. Payment Service → processPayment()
    ↓
3. Inventory Service → reserveItems()
    ↓
4. Order Service → completeOrder()

Если ошибка → rollback в обратном порядке
```

**Преимущества:**
- Централизованная логика
- Легче отладка

**Недостатки:**
- Single point of failure
- Сложность координатора

### Пример реализации (Orchestration)

```java
@Service
public class OrderSaga {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    public void execute(Order order) {
        try {
            // Step 1
            orderService.createOrder(order);
            
            // Step 2
            paymentService.processPayment(order);
            
            // Step 3
            inventoryService.reserveItems(order);
            
            // Success
            orderService.completeOrder(order);
            
        } catch (Exception e) {
            // Compensation
            inventoryService.releaseItems(order);
            paymentService.refundPayment(order);
            orderService.cancelOrder(order);
        }
    }
}
```

---

## 🔹 Масштабирование

### Виды масштабирования

#### 1. Vertical Scaling (Scale Up)

Увеличение ресурсов одного сервера.

```
CPU: 4 cores → 8 cores
RAM: 8GB → 16GB
```

**Преимущества:**
- Простота
- Нет изменений в коде

**Недостатки:**
- Ограничение физическими ресурсами
- Single point of failure
- Дорого

#### 2. Horizontal Scaling (Scale Out)

Добавление новых серверов.

```
Server 1 → Server 1 + Server 2 + Server 3
```

**Преимущества:**
- Теоретически неограниченно
- Отказоустойчивость
- Дешевле

**Недостатки:**
- Сложность (load balancing, state management)
- Требует изменений в коде (stateless)

### Масштабирование микросервисов

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Orders  │  │  Users   │  │   Auth   │
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
   x3 instances   x1 instance   x2 instances
```

- **Orders** — высокая нагрузка → масштабируем
- **Users** — низкая нагрузка → 1 инстанс
- **Auth** — средняя нагрузка → 2 инстанса

### Stateless vs Stateful

**Stateless** — состояние хранится внешне (Redis, DB).

```java
// Stateless
@RestController
public class UserController {
    @Autowired
    private UserRepository repository;
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return repository.findById(id);
    }
}
```

**Stateful** — состояние хранится в памяти.

```java
// Stateful (плохо для масштабирования)
@RestController
public class UserController {
    private Map<Long, User> cache = new HashMap<>();  // состояние в памяти
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return cache.get(id);
    }
}
```

> [!tip] Для горизонтального масштабирования — stateless

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Монолит:** единое приложение, простота, но сложное масштабирование
> - **Микросервисы:** независимые сервисы, сложность, но гибкое масштабирование
> - **API Gateway:** единая точка входа, роутинг, аутентификация (Spring Cloud Gateway, Kong, Nginx)
> - **Saga:** распределённые транзакции (Choreography, Orchestration)
> - **Масштабирование:** Vertical (scale up), Horizontal (scale out)
> - **Stateless** — для горизонтального масштабирования

```
Монолит vs Микросервисы:
Монолит → простота, единая БД, медленный CI/CD
Микросервисы → сложность, независимые БД, быстрый CI/CD

API Gateway:
Spring Cloud Gateway → reactive, Java
Kong → высокая производительность
Nginx → высокая производительность, ограниченная динамичность

Saga:
Choreography → децентрализованная, события
Orchestration → централизованная, координатор

Масштабирование:
Vertical → больше ресурсов на одном сервере
Horizontal → больше серверов
Stateless → обязательно для horizontal
```
