# Spring — Data JPA (Repository, Query, Transactions)

> **Теги:** #spring #framework #jpa #middle #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Spring_Index]]

---

## 🔹 Архитектура Spring Data

```
Repository (marker)
    └── CrudRepository<T, ID>
            └── PagingAndSortingRepository<T, ID>
                    └── JpaRepository<T, ID>
```

| Интерфейс | Ключевые методы |
|-----------|-----------------|
| `CrudRepository` | `save`, `findById`, `findAll`, `deleteById`, `count` |
| `PagingAndSortingRepository` | `findAll(Pageable)`, `findAll(Sort)` |
| `JpaRepository` | `flush`, `saveAndFlush`, `deleteInBatch`, `getReferenceById` |

```java
public interface UserRepository extends JpaRepository<User, Long> { }
```

> [!note] Spring создаёт реализацию
> Proxy в runtime — писать `implements` не нужно.

---

## 🔹 Derived Query Methods

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByStatus(OrderStatus status);

    Optional<Order> findByIdAndUserId(Long id, Long userId);

    boolean existsByUserEmail(String email);

    long countByStatus(OrderStatus status);

    void deleteByStatus(OrderStatus status);

    List<Order> findByCreatedAtBetween(Instant from, Instant to);

    List<Order> findByItemsProductName(String productName);  // вложенное свойство
}
```

| Keyword | Пример | SQL-подобно |
|---------|--------|-------------|
| `And` / `Or` | `findByStatusAndUserId` | AND / OR |
| `Between` | `findByPriceBetween` | BETWEEN |
| `LessThan` / `GreaterThan` | `findByAgeLessThan` | < / > |
| `Like` / `Containing` | `findByNameContaining` | LIKE |
| `In` | `findByStatusIn` | IN (...) |
| `IsNull` / `IsNotNull` | `findByDeletedAtIsNull` | IS NULL |
| `OrderBy` | `findByStatusOrderByCreatedAtDesc` | ORDER BY |
| `Top` / `First` | `findTop10By...` | LIMIT |

> [!warning] Сложные запросы
> Длинные имена методов нечитаемы — переходи на `@Query` или Specifications.

---

## 🔹 @Query

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findWithItems(@Param("id") Long id);

@Modifying
@Transactional
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
int deactivateInactive(@Param("date") Instant date);

@Query(value = """
    SELECT * FROM orders o
    WHERE o.total > :minTotal
    """, nativeQuery = true)
List<Order> findHighValue(@Param("minTotal") BigDecimal minTotal);
```

> [!warning] `@Modifying`
> Требует `@Transactional` на методе или классе; после update — `clearAutomatically` / `flushAutomatically` при необходимости.

---

## 🔹 Paging и Sorting

```java
Page<Order> page = orderRepository.findByStatus(
    OrderStatus.PAID,
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
);

page.getContent();
page.getTotalElements();
page.getTotalPages();
page.hasNext();
```

| Тип | Содержимое | Когда |
|-----|------------|-------|
| `Page<T>` | Данные + total count | UI с пагинацией, нужен total |
| `Slice<T>` | Данные + hasNext | Infinite scroll, count дорогой |

```java
Slice<Order> slice = orderRepository.findByStatus(status, Pageable.ofSize(20));
```

---

## 🔹 @Transactional

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;

    @Transactional(readOnly = true)
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }

    @Transactional(rollbackFor = Exception.class)
    public Order placeOrder(OrderRequest req) {
        // несколько операций в одной транзакции
    }
}
```

### Propagation

| Значение | Поведение |
|----------|-----------|
| `REQUIRED` (default) | Присоединиться к текущей или создать новую |
| `REQUIRES_NEW` | Всегда новая транзакция (suspend текущую) |
| `NESTED` | Savepoint внутри текущей |
| `MANDATORY` | Должна быть активная, иначе exception |
| `SUPPORTS` | Транзакция если есть, иначе без |
| `NOT_SUPPORTED` | Suspend транзакцию |
| `NEVER` | Exception если транзакция есть |
| `SUPPORTS` / `NOT_SUPPORTED` | Для read-only внешних вызовов |

### Isolation

| Уровень | Dirty read | Non-repeatable | Phantom |
|---------|:----------:|:--------------:|:-------:|
| `READ_UNCOMMITTED` | ✅ | ✅ | ✅ |
| `READ_COMMITTED` | ❌ | ✅ | ✅ |
| `REPEATABLE_READ` | ❌ | ❌ | ✅ |
| `SERIALIZABLE` | ❌ | ❌ | ❌ |

### ❌ Self-invocation

```java
@Service
public class OrderService {
    public void create() {
        this.saveInternal();  // @Transactional НЕ сработает — вызов без proxy
    }

    @Transactional
    public void saveInternal() { }
}
```

### ✅ Решение

Вынести в отдельный `@Service` или вызвать через `self` proxy / `TransactionTemplate`.

> [!note] Где ставить `@Transactional`
> На **service layer**, не на repository (кроме `@Modifying` queries).

---

## 🔹 Projections

```java
// interface projection
public interface UserSummary {
    Long getId();
    String getEmail();
}

List<UserSummary> findByActiveTrue();

// class-based DTO
public record UserDto(Long id, String email) {}

@Query("SELECT new com.example.UserDto(u.id, u.email) FROM User u")
List<UserDto> findAllDtos();

// closed projection с SpEL
public interface OrderView {
    Long getId();
    @Value("#{target.user.email}")
    String getUserEmail();
}
```

---

## 🔹 Specifications

```java
public interface OrderRepository extends JpaRepository<Order, Long>,
        JpaSpecificationExecutor<Order> { }

public class OrderSpecs {
    public static Specification<Order> hasStatus(OrderStatus status) {
        return (root, query, cb) -> cb.equal(root.get("status"), status);
    }

    public static Specification<Order> createdAfter(Instant from) {
        return (root, query, cb) -> cb.greaterThan(root.get("createdAt"), from);
    }
}

// использование
orderRepository.findAll(
    Specification.where(OrderSpecs.hasStatus(PAID))
        .and(OrderSpecs.createdAfter(weekAgo))
);
```

---

## 🔹 Auditing

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Article {
    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    private String createdBy;
}
```

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig { }
```

---

## 🔹 Custom Repository

```java
public interface OrderRepositoryCustom {
    List<Order> findComplex(OrderFilter filter);
}

public class OrderRepositoryImpl implements OrderRepositoryCustom {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Order> findComplex(OrderFilter filter) { ... }
}

public interface OrderRepository extends JpaRepository<Order, Long>, OrderRepositoryCustom { }
```

> [!note] Именование
> Impl-класс: `{InterfaceName}Impl` (настраивается через `repositoryImplementationPostfix`).

---

## 🔹 @Lock на репозитории

> [!note] Определение
> Spring Data позволяет ставить блокировку прямо на метод репозитория через `@Lock`,
> не требуя `EntityManager` вручную.

```java
public interface AccountRepository extends JpaRepository<Account, Long> {

    // Pessimistic Write — SELECT ... FOR UPDATE
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);

    // Pessimistic Read — SELECT ... FOR SHARE
    @Lock(LockModeType.PESSIMISTIC_READ)
    Optional<Account> findById(Long id);

    // Optimistic — проверяет @Version при flush
    @Lock(LockModeType.OPTIMISTIC)
    Optional<Account> findByNumber(String number);
}
```

| LockModeType | SQL | Когда |
|---|---|---|
| `PESSIMISTIC_WRITE` | FOR UPDATE | Конкурентное изменение одной записи |
| `PESSIMISTIC_READ` | FOR SHARE | Чтение без изменений конкурентами |
| `OPTIMISTIC` | Проверка `@Version` | Редкие конфликты, высокая производительность |

> [!warning] Deadlock
> `PESSIMISTIC_WRITE` на нескольких строках в разном порядке → deadlock.
> Всегда блокировать в одном порядке или использовать timeout:
> `@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))`

> [!tip] Связь с Hibernate
> `@Lock` в репозитории — Spring-обёртка над `EntityManager.find(LockModeType)`.
> Подробнее о стратегиях блокировки — [[Hibernate]].

---

## 🔹 Flyway / Liquibase

> [!note] Миграции схемы
> Версионирование DDL в SQL-файлах. См. [[Flyway]].

```
src/main/resources/db/migration/
    V1__create_users.sql
    V2__add_orders.sql
```

```yaml
spring.jpa.hibernate.ddl-auto: validate
spring.flyway.enabled: true
```

> [!warning] Не полагаться на `ddl-auto: update` в production — только миграции.

---

## 🔹 Итог

1. `JpaRepository` — стандартный базовый интерфейс.
2. Derived queries — для простых фильтров; `@Query` — для сложных.
3. `Page` vs `Slice` — total count vs производительность.
4. `@Transactional` на service; знать propagation и self-invocation.
5. Projections — меньше данных из БД.
6. Specifications — динамические фильтры.
7. Схема БД — Flyway, не ddl-auto на prod.

```
Шпаргалка Data JPA:
─────────────────────────────────────────
extends JpaRepository<Entity, ID>
findByXxxAndYyy                    → derived query
@Query JPQL / nativeQuery
@Modifying + @Transactional        → UPDATE/DELETE
PageRequest.of(page, size, sort)
@Transactional(readOnly=true)      → read
REQUIRES_NEW                       → отдельная TX
JpaSpecificationExecutor           → Specs
@EnableJpaAuditing                 → audit fields
Flyway + ddl-auto: validate        → prod
@Lock(PESSIMISTIC_WRITE)          → SELECT FOR UPDATE
```

---

## 🔗 Связанные конспекты

> [!abstract] Смотри также
> - [[12_JPA_Hibernate]] (интервью) — Entity, Lazy/Eager, N+1, кэши на собеседовании
> - [[11_Transactional]] (интервью) — @Transactional, уровни изоляции на собеседовании
> - [[13_SQL_DB]] (интервью) — SQL и индексы в контексте JPA-запросов
> - [[Hibernate]] — ORM под капотом Spring Data JPA
> - [[main SQL DB]] — реляционные базы данных
