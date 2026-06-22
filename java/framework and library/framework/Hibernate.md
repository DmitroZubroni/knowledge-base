# Spring — Hibernate (ORM, Mapping, Session)

> **Теги:** #spring #framework #hibernate #orm #middle #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Spring_Index]]

---

## 🔹 Что такое ORM и Hibernate

> [!note] Определение
> **ORM** (Object-Relational Mapping) — отображение объектов Java на таблицы реляционной БД. **Hibernate** — популярная ORM-реализация; **JPA** — стандартный API поверх Hibernate.

| | JDBC | JPA | Hibernate |
|---|------|-----|-----------|
| Уровень | SQL вручную | Спецификация API | Реализация + расширения |
| Маппинг | RowMapper / вручную | `@Entity`, аннотации | То же + HQL, критерии |
| Кэш | ❌ | 1st level (провайдер) | 1st + 2nd level cache |
| Портативность | SQL диалект вручную | Выше | Hibernate-specific features |

**Зачем:** меньше SQL-boilerplate, управление связями, lazy loading, кэш, versioning.

---

## 🔹 JPA vs Hibernate

```
JPA (javax/jakarta.persistence)  ← спецификация
        │
        ├── Hibernate (основная реализация)
        ├── EclipseLink
        └── OpenJPA
```

> [!note] В Spring Boot
> `spring-boot-starter-data-jpa` → Hibernate как default JPA provider.

---

## 🔹 Entity

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String email;

    @Column(name = "created_at")
    private Instant createdAt;
}
```

| `@GeneratedValue` strategy | Поведение |
|---------------------------|-----------|
| `IDENTITY` | DB auto-increment (PostgreSQL SERIAL) |
| `SEQUENCE` | Отдельная sequence (рекомендуется для batch insert) |
| `TABLE` | Таблица-счётчик (редко) |
| `AUTO` | Провайдер выбирает (часто SEQUENCE) |

---

## 🔹 Связи между сущностями

```java
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;  // owning side (@JoinColumn)

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

@Entity
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;  // inverse: mappedBy на Order.items
}
```

| Связь | Owning side | Inverse |
|-------|-------------|---------|
| `@OneToOne` | Сторона с `@JoinColumn` | `mappedBy` |
| `@OneToMany` / `@ManyToOne` | `@ManyToOne` | `@OneToMany(mappedBy=...)` |
| `@ManyToMany` | Выбирают одну сторону с `@JoinTable` | `mappedBy` |

> [!warning] Owning side
> Изменения FK сохраняются только с **owning side**. Менять только `order.setItems()` без `item.setOrder()` — FK может не обновиться.

---

## 🔹 FetchType — LAZY vs EAGER

| Связь | Default |
|-------|---------|
| `@ManyToOne`, `@OneToOne` | **EAGER** |
| `@OneToMany`, `@ManyToMany` | **LAZY** |

> [!tip] Практика
> Везде явно ставить **LAZY** для associations; загружать JOIN FETCH / EntityGraph когда нужно.

### N+1 проблема

```java
// ❌ N+1: 1 запрос orders + N запросов items для каждого order
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class)
    .getResultList();
orders.forEach(o -> o.getItems().size());  // lazy load каждый раз
```

### ✅ Решения

```java
// JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findWithItems(@Param("status") OrderStatus status);

// @EntityGraph
@EntityGraph(attributePaths = {"items"})
List<Order> findByStatus(OrderStatus status);

// batch fetching (application.yml)
spring.jpa.properties.hibernate.default_batch_fetch_size: 16
```

---

## 🔹 CascadeType

| Type | Действие |
|------|----------|
| `PERSIST` | cascade persist |
| `MERGE` | cascade merge |
| `REMOVE` | cascade remove |
| `REFRESH` | cascade refresh |
| `DETACH` | cascade detach |
| `ALL` | все выше |

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

> [!warning] Cascade на `@ManyToOne`
> Осторожно: `cascade = REMOVE` на parent может удалить shared entities.

---

## 🔹 Inheritance Mapping

| Стратегия | Таблицы | Плюсы | Минусы |
|-----------|---------|-------|--------|
| `SINGLE_TABLE` | Одна | Быстрые запросы | Nullable колонки, широкая таблица |
| `JOINED` | Таблица на класс + JOIN | Нормализовано | Много JOIN |
| `TABLE_PER_CLASS` | Таблица на конкретный класс | Простые запросы к subclass | Полиморфные запросы тяжёлые |

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment { }

@Entity
public class CardPayment extends Payment { }
```

---

## 🔹 Session и EntityManager

**Состояния entity:**

```
transient ──persist()──► persistent ──detach()/close──► detached
                              │
                         remove()
                              ▼
                           removed
```

```java
@Transactional
public void example(EntityManager em) {
  User u = new User();           // transient
  em.persist(u);                 // persistent (в persistence context)

  em.detach(u);                  // detached
  u = em.merge(u);               // reattach copy

  em.remove(u);                  // removed (при flush/commit)
}
```

> [!note] В Spring
> `@Transactional` на сервисе открывает EntityManager на границе транзакции; persistence context = один на транзакцию (обычно).

---

## 🔹 First & Second Level Cache

| Уровень | Scope | По умолчанию |
|---------|-------|--------------|
| **1st level** | Persistence context (сессия) | Всегда включён |
| **2nd level** | SessionFactory, shared | Выключен |

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { }
```

```yaml
spring.jpa.properties.hibernate.cache.use_second_level_cache: true
spring.jpa.properties.hibernate.cache.region.factory_class: jcache
```

> [!warning] 2nd level cache
> Только для **редко меняющихся** read-heavy сущностей; инвалидация при write сложнее.

---

## 🔹 HQL / JPQL

```java
// JPQL — работает с entity, не таблицами
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// Native SQL
@Query(value = "SELECT * FROM users WHERE status = ?1", nativeQuery = true)
List<User> findByStatusNative(String status);
```

| | SQL | JPQL/HQL |
|---|-----|----------|
| Объекты | Таблицы, колонки | Entity, поля |
| Портативность | Диалект БД | JPA provider переводит |

```java
@Entity
@NamedQuery(name = "User.findActive",
    query = "SELECT u FROM User u WHERE u.active = true")
public class User { }
```

---

## 🔹 Criteria API

```java
public List<User> findByFilters(String email, Boolean active) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> cq = cb.createQuery(User.class);
    Root<User> root = cq.from(User.class);

    List<Predicate> predicates = new ArrayList<>();
    if (email != null) predicates.add(cb.like(root.get("email"), "%" + email + "%"));
    if (active != null) predicates.add(cb.equal(root.get("active"), active));

    cq.where(predicates.toArray(new Predicate[0]));
    return em.createQuery(cq).getResultList();
}
```

> [!tip] Когда использовать
> Динамические фильтры без конкатенации строк. В Spring Data — чаще **Specifications** (см. [[Spring_Data_JPA]]).

---

## 🔹 @Embeddable / @EmbeddedId

```java
@Embeddable
public record Address(String street, String city, String zip) {}

@Entity
public class Customer {
    @Embedded
    private Address address;
}

@Embeddable
public class OrderId implements Serializable {
    private Long customerId;
    private Long orderSeq;
}

@Entity
public class Order {
    @EmbeddedId
    private OrderId id;
}
```

---

## 🔹 Optimistic / Pessimistic Locking

```java
@Entity
public class Account {
    @Version
    private Long version;  // optimistic lock
}
```

| LockModeType | Смысл |
|--------------|-------|
| `OPTIMISTIC` | Проверка `@Version` при flush |
| `PESSIMISTIC_READ` | SELECT ... FOR SHARE |
| `PESSIMISTIC_WRITE` | SELECT ... FOR UPDATE |

```java
Account acc = em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE);
```

> [!note] OptimisticLockException
> При конфликте версий — retry или сообщение пользователю.

---

## 🔹 DDL auto

```yaml
spring.jpa.hibernate.ddl-auto: validate  # none | validate | update | create | create-drop
```

| Значение | Поведение | Где |
|----------|-----------|-----|
| `none` | Ничего | Production |
| `validate` | Проверка схемы | Production CI |
| `update` | Добавление колонок | Dev only |
| `create` | Пересоздание | Тесты |
| `create-drop` | Создать и удалить | Интеграционные тесты |

> [!warning] Production
> **Никогда** `update` / `create` на prod. Схема — через [[Flyway]] / Liquibase.

---

## 🔹 Итог

1. JPA — API, Hibernate — реализация в Boot.
2. Owning side определяет FK; `mappedBy` — inverse.
3. LAZY + JOIN FETCH / EntityGraph — против N+1.
4. Cascade осознанно; `orphanRemoval` для composition.
5. Entity states: transient → persistent → detached → removed.
6. JPQL для entity-запросов; native — когда нужен диалект SQL.
7. `@Version` — optimistic locking.
8. DDL auto только dev/test; prod — миграции.

```
Шпаргалка Hibernate:
─────────────────────────────────────────
@Entity @Table @Id @GeneratedValue
@ManyToOne(LAZY) + @JoinColumn     → owning
@OneToMany(mappedBy=...)           → inverse
JOIN FETCH / @EntityGraph          → N+1 fix
CascadeType.ALL + orphanRemoval    → composition
@Version                           → optimistic lock
ddl-auto: validate (prod)          → + Flyway
persist / merge / remove / detach  → lifecycle
```

---

## 🔹 Кэш первого уровня (L1) — Session Cache

L1 кэш — это **Persistence Context**. Всегда включён, область видимости — одна сессия (транзакция).

```java
@Transactional
public void example() {
    User u1 = userRepo.findById(1L).orElseThrow(); // SELECT выполняется
    User u2 = userRepo.findById(1L).orElseThrow(); // SELECT НЕ выполняется — из L1!
    System.out.println(u1 == u2); // true — тот же объект

    u1.setName("Alice"); // dirty checking — изменение отслеживается
    // при commit: автоматически UPDATE
}
// После завершения @Transactional — L1 кэш очищается
```

**Что хранит L1:** снимок (snapshot) каждой loaded entity для dirty checking.

**Ловушка:** в batch-обработке (1 транзакция, 10 000 записей) L1 кэш растёт бесконечно → OOM.

```java
@Transactional
public void batchProcess(List<Long> ids) {
    for (int i = 0; i < ids.size(); i++) {
        process(entityRepo.findById(ids.get(i)).orElseThrow());
        if (i % 100 == 0) {
            entityManager.flush();  // сбросить изменения в БД
            entityManager.clear();  // очистить L1 кэш
        }
    }
}
```

---

## 🔹 Кэш второго уровня (L2) — Shared Cache

L2 кэш живёт на уровне `SessionFactory` — **разделяется между всеми сессиями**. Подходит для редко меняющихся справочных данных.

### Настройка (EhCache / Caffeine через JCache)

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            missing_cache_strategy: create  # создать регион если нет конфигурации
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

### Стратегии конкурентного доступа

```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country {
    @Id private Long id;
    private String name;
    private String code;
}
```

| Стратегия | Поведение | Когда |
|-----------|-----------|-------|
| `READ_ONLY` | Нет блокировок. Исключение при попытке изменить. | Иммутабельные справочники (страны, категории) |
| `NONSTRICT_READ_WRITE` | Нет транзакционных гарантий. Короткое окно несогласованности. | Редко меняемые, допустима небольшая задержка |
| `READ_WRITE` | Soft-лок при обновлении. Согласованность. | Часто читается, иногда обновляется |
| `TRANSACTIONAL` | Полная транзакционная согласованность (JTA). | Транзакционные кэши (JBoss Cache) |

> [!tip] Рекомендация
> `READ_ONLY` — для справочников (страны, валюты, типы).
> `READ_WRITE` — для сущностей которые иногда обновляются (профиль пользователя, настройки).

### Инвалидация L2 кэша

Hibernate автоматически инвалидирует L2 при `save()` / `delete()` через JPA. Но:
- При **native SQL** запросах Hibernate не знает что изменилось → кэш не инвалидируется!
- Решение: `@QueryHint` с `CacheMode.IGNORE` для native, или ручная очистка

```java
// Ручная инвалидация региона
@Autowired
private EntityManagerFactory emf;

public void evictCache(Class<?> entityClass) {
    emf.getCache().evict(entityClass);       // конкретная сущность
    emf.getCache().evictAll();               // весь L2 кэш
    emf.getCache().evict(entityClass, id);   // один объект по ID
}

// Или через Hibernate SessionFactory
SessionFactory sf = emf.unwrap(SessionFactory.class);
sf.getCache().evictEntityData(Country.class);
```

---

## 🔹 Query Cache — кэш результатов запросов

Query Cache хранит **список идентификаторов** результата запроса. При попадании в кэш — Hibernate загружает объекты по ID из L2 (или БД если нет в L2).

> [!warning] Query Cache требует L2 Cache
> Query Cache хранит только ID → загрузка объектов по ID идёт через L2. Без L2 — каждый объект будет загружен из БД отдельным запросом (N+1 эффект!). Используй Query Cache только вместе с L2.

```yaml
spring.jpa.properties.hibernate.cache.use_query_cache: true
```

```java
// Включить кэширование конкретного запроса
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
@Query("SELECT c FROM Country c ORDER BY c.name")
List<Country> findAllCached();

// Или через EntityManager
List<Country> countries = em.createQuery("SELECT c FROM Country c", Country.class)
    .setHint("org.hibernate.cacheable", true)
    .setHint("org.hibernate.cacheRegion", "countries.all") // именованный регион
    .getResultList();
```

**Когда использовать Query Cache:**
- Одинаковый запрос с одинаковыми параметрами выполняется часто
- Данные редко меняются
- Результат большой (много объектов)

---

## 🔹 Статистика кэша — мониторинг

```yaml
spring.jpa.properties.hibernate.generate_statistics: true
logging.level.org.hibernate.stat: DEBUG
```

```java
// Программный доступ к статистике
@Autowired EntityManagerFactory emf;

Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();

// L2 кэш
System.out.printf("L2 hit ratio: %.2f%% (%d hits / %d misses)%n",
    stats.getSecondLevelCacheHitCount() * 100.0 /
        (stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount()),
    stats.getSecondLevelCacheHitCount(),
    stats.getSecondLevelCacheMissCount()
);

// Query кэш
System.out.printf("Query cache hits: %d, misses: %d%n",
    stats.getQueryCacheHitCount(),
    stats.getQueryCacheMissCount()
);

// Запросы
System.out.println("Total queries: " + stats.getQueryExecutionCount());
System.out.println("Slowest query: " + stats.getQueryExecutionMaxTime() + "ms");
System.out.println("Slow query string: " + stats.getQueryExecutionMaxTimeQueryString());
```

**На что смотреть:**
```
Hit ratio L2 > 90%      → кэш работает эффективно
Hit ratio L2 < 50%      → пересмотреть стратегию TTL или что кэшировать
Query cache miss >> hit → возможно запросы с разными параметрами, кэш неэффективен
getQueryExecutionMaxTime → длинные запросы для оптимизации
```

---

## 🔹 L2 Cache vs Redis — когда что

| | Hibernate L2 Cache | Redis |
|-|--------------------|-------|
| Область | JVM / кластер через JCache | Внешний сервис |
| Прозрачность | Автоматически для Entity | Явное кодирование |
| Инвалидация | Автоматическая при CUD | Ручная |
| Распределённость | Через JCache (Hazelcast, Infinispan) | Нативно |
| Типы данных | Только Entity | Любые |
| Когда | Entity-level кэш прозрачно | Сложная логика, любые объекты |

**Правило:** если нужно кэшировать JPA Entity прозрачно — L2. Если нужен кэш произвольных данных (DTO, результаты сложных вычислений, внешние API) — Redis + `@Cacheable`.

```java
// Комбинация: @Cacheable (Redis) для DTO + L2 (Hibernate) для Entity
@Service
public class ProductService {

    // Spring Cache (Redis) — кэш DTO для API
    @Cacheable(value = "products", key = "#id")
    public ProductDTO findById(Long id) {
        // Hibernate L2 кэш прозрачно применяется здесь при загрузке Entity
        Product product = productRepository.findById(id).orElseThrow();
        return ProductDTO.from(product);
    }
}
```
