# JPA & Hibernate

> **Теги:** #interviews #jpa #hibernate #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 JPA vs Hibernate

### JPA (Java Persistence API)

**JPA** — спецификация для ORM (Object-Relational Mapping) в Java.

- Спецификация (не реализация)
- Определяет API и аннотации
- Часть Jakarta EE (ранее Java EE)

### Hibernate

**Hibernate** — реализация JPA.

- ORM фреймворк
- Реализует JPA спецификацию
- Добавляет свои возможности (HQL, Criteria API)

### Схема

```
JPA (спецификация)
    ↓
Hibernate (реализация)
    ↓
JDBC (драйвер БД)
```

---

## 🔹 Entity и аннотации

### @Entity

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false, unique = true)
    private String username;

    @Column(name = "email")
    private String email;

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at")
    private Date createdAt;
}
```

### Основные аннотации

| Аннотация | Назначение |
|-----------|------------|
| **@Entity** | Обозначает класс как сущность |
| **@Table** | Имя таблицы |
| **@Id** | Первичный ключ |
| **@GeneratedValue** | Стратегия генерации ID |
| **@Column** | Настройка колонки |
| **@Transient** | Поле не сохраняется в БД |

---

## 🔹 ID стратегии

### GenerationType

| Стратегия | Описание | Когда использовать |
|-----------|----------|-------------------|
| **IDENTITY** | Автоинкремент БД | PostgreSQL, MySQL |
| **SEQUENCE** | Sequence БД | Oracle, PostgreSQL |
| **TABLE** | Отдельная таблица для ID | Портабельность |
| **AUTO** | Hibernate выбирает | По умолчанию |

### Примеры

```java
// IDENTITY (auto-increment)
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// SEQUENCE
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_sequence", allocationSize = 1)
private Long id;

// TABLE (portable)
@Id
@GeneratedValue(strategy = GenerationType.TABLE)
private Long id;
```

### IDENTITY vs SEQUENCE

| Характеристика | IDENTITY | SEQUENCE |
|----------------|----------|----------|
| Производительность | Медленнее (INSERT для получения ID) | Быстрее (предварительное выделение) |
| Batch insert | Не поддерживается | Поддерживается |
| Портабельность | Зависит от БД | Зависит от БД |

---

## 🔹 N+1 Problem

**N+1 Problem** — проблема когда выполняется N+1 SQL запросов вместо одного.

### Пример проблемы

```java
@Entity
public class User {
    @Id
    private Long id;
    
    @OneToMany(fetch = FetchType.LAZY)  // LAZY по умолчанию
    private List<Order> orders;
}

// Запрос
List<User> users = userRepository.findAll();
for (User user : users) {
    user.getOrders().size();  // N дополнительных запросов!
}
```

**SQL:**
```sql
-- 1 запрос для пользователей
SELECT * FROM users;

-- N запросов для заказов каждого пользователя
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
-- ...
```

### Решения

#### 1. EntityGraph (рекомендуется)

```java
@Entity
@NamedEntityGraph(
    name = "User.withOrders",
    attributeNodes = @NamedAttributeNode("orders")
)
public class User {
    // ...
}

// Использование
EntityGraph graph = em.getEntityGraph("User.withOrders");
List<User> users = em.createQuery("SELECT u FROM User u", User.class)
                     .setHint("javax.persistence.fetchgraph", graph)
                     .getResultList();
```

#### 2. JOIN FETCH

```java
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

#### 3. @EntityGraph аннотация

```java
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();
```

---

## 🔹 FetchType (LAZY vs EAGER)

### LAZY (рекомендуется)

```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
```

- Загружается только когда обращаемся
- По умолчанию для @OneToMany, @ManyToMany
- Экономит память

### EAGER

```java
@OneToMany(fetch = FetchType.EAGER)
private List<Order> orders;
```

- Загружается сразу вместе с сущностью
- По умолчанию для @OneToOne, @ManyToOne
- Может привести к N+1

### Сравнение

| FetchType | Когда загружается | По умолчанию |
|-----------|-------------------|--------------|
| **LAZY** | При первом обращении | @OneToMany, @ManyToMany |
| **EAGER** | Сразу при загрузке сущности | @OneToOne, @ManyToOne |

> [!warning] LAZY и LazyInitializationException
```java
@Transactional
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}

// Без @Transactional
User user = userService.getUser(1L);
user.getOrders().size();  // LazyInitializationException!
```

Решение: использовать @Transactional или загрузить данные в транзакции.

---

## 🔹 EntityGraph

**EntityGraph** — определение графа сущностей для загрузки.

### @NamedEntityGraph

```java
@Entity
@NamedEntityGraph(
    name = "User.withOrdersAndItems",
    attributeNodes = {
        @NamedAttributeNode("orders"),
        @NamedAttributeNode(value = "orders", subgraphs = {
            @NamedSubgraph(name = "orders.items", attributeNodes = @NamedAttributeNode("items"))
        })
    }
)
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}

@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    private List<Item> items;
}
```

### Программное создание

```java
EntityGraph<User> graph = em.createEntityGraph(User.class);
graph.addAttributeNodes("orders");
Subgraph<Order> ordersSubgraph = graph.addSubgraph("orders");
ordersSubgraph.addAttributeNodes("items");

List<User> users = em.createQuery("SELECT u FROM User u", User.class)
                     .setHint("javax.persistence.fetchgraph", graph)
                     .getResultList();
```

### fetchgraph vs loadgraph

| Тип | Поведение |
|-----|-----------|
| **fetchgraph** | Загружает только указанные атрибуты |
| **loadgraph** | Загружает указанные + EAGER атрибуты |

---

## 🔹 Оптимистичные блокировки

**Optimistic Locking** — блокировка на уровне приложения (версия).

### @Version

```java
@Entity
public class Product {
    @Id
    private Long id;
    
    private String name;
    
    @Version
    private Long version;  // версия для оптимистичной блокировки
}
```

### Как работает

```
1. Читаем продукт (version = 1)
2. Другой пользователь обновляет продукт (version = 2)
3. Пытаемся обновить (version = 1)
4. Hibernate проверяет: version в БД != version в объекте
5. OptimisticLockException
```

### Пример

```java
Product product = repository.findById(1L).orElseThrow();
// product.version = 1

// Другой поток обновил
product.setName("New Name");
repository.save(product);  // OptimisticLockException
```

---

## 🔹 Пессимистичные блокировки

**Pessimistic Locking** — блокировка на уровне БД.

### Типы

| LockModeType | Описание |
|--------------|----------|
| **PESSIMISTIC_READ** | Блокировка на чтение (SELECT FOR SHARE) |
| **PESSIMISTIC_WRITE** | Блокировка на запись (SELECT FOR UPDATE) |
| **PESSIMISTIC_FORCE_INCREMENT** | Блокировка + инкремент версии |

### Пример

```java
// PESSIMISTIC_WRITE
@EntityGraph(attributePaths = {"orders"})
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "3000"))
Optional<User> findById(Long id);
```

### Когда использовать

- **Optimistic** — когда мало конфликтов (web приложения)
- **Pessimistic** — когда много конфликтов (финансовые операции)

---

## 🔹 Cascade

**Cascade** — каскадные операции над связанными сущностями.

### CascadeType

| CascadeType | Описание |
|-------------|----------|
| **PERSIST** | Сохранение связанной сущности |
| **MERGE** | Объединение связанной сущности |
| **REMOVE** | Удаление связанной сущности |
| **REFRESH** | Обновление связанной сущности |
| **DETACH** | Отсоединение связанной сущности |
| **ALL** | Все операции |

### Пример

```java
@Entity
public class Order {
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Item> items;
}

// При сохранении Order сохранятся и Items
orderRepository.save(order);
```

### orphanRemoval

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
private List<Item> items;

// Если удалить Item из списка → удалится из БД
order.getItems().remove(item);
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **JPA** — спецификация, **Hibernate** — реализация
> - **ID стратегии:** IDENTITY (auto-increment), SEQUENCE (portable)
> - **N+1:** EntityGraph, JOIN FETCH, @EntityGraph
> - **FetchType:** LAZY (по умолчанию для коллекций), EAGER (для одиночных)
> - **EntityGraph:** fetchgraph (только указанные), loadgraph (указанные + EAGER)
> - **Optimistic Locking:** @Version, OptimisticLockException
> - **Pessimistic Locking:** PESSIMISTIC_READ, PESSIMISTIC_WRITE
> - **Cascade:** PERSIST, MERGE, REMOVE, ALL

```
JPA vs Hibernate:
JPA → спецификация
Hibernate → реализация

ID стратегии:
IDENTITY → auto-increment БД
SEQUENCE → sequence БД
TABLE → отдельная таблица (portable)

N+1 Problem:
1 запрос + N запросов
Решение: EntityGraph, JOIN FETCH

FetchType:
LAZY → при обращении (коллекции)
EAGER → сразу (одиночные)

EntityGraph:
fetchgraph → только указанные
loadgraph → указанные + EAGER

Locking:
Optimistic → @Version
Pessimistic → PESSIMISTIC_READ/WRITE

Cascade:
PERSIST, MERGE, REMOVE, ALL
orphanRemoval → удаление сирот
```
