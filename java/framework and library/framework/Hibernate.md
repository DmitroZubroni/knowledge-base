# Spring — Hibernate (ORM, Mapping, Session)

> [!abstract] Связи
> [[00_Spring_Index]] | [[Spring_Data_JPA]] | [[JDBC_JPA]] | [[00_PostgreSQL]]

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
