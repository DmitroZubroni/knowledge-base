> **Теги:** #interviews #spring #jpa #hibernate #n+1 #конспект
> [!abstract] Связи
> [[main]] | [[Interviews]] | [[Spring_Data_JPA]] | [[Hibernate]]

# JPA & Hibernate — Вопросы на собесе

---

## 🔹 Жизненный цикл Entity

```
new → persist() → Managed (отслеживается) → detach() → Detached
                       ↓                          ↑
                   flush/commit              merge()
                       ↓
                   DB (сохранено)
                       ↓ remove()
                   Removed → commit() → удалено из БД
```

| Состояние | Отслеживается Hibernate | В БД |
|-----------|------------------------|------|
| **Transient** | ❌ | ❌ |
| **Managed** | ✅ (dirty checking) | Возможно |
| **Detached** | ❌ | ✅ |
| **Removed** | ✅ | ✅ (до commit) |

**Dirty Checking** — Hibernate автоматически отслеживает изменения managed сущностей и генерирует UPDATE при flush. Не нужно явно вызывать `save()` для обновления managed entity.

```java
@Transactional
public void updateName(Long id, String name) {
    User user = userRepository.findById(id).orElseThrow(); // Managed
    user.setName(name); // dirty checking → UPDATE при flush/commit
    // save() НЕ НУЖЕН если entity уже managed
}
```

---

## 🔹 N+1 Problem — самый частый вопрос

### Что это такое

```java
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    private List<Book> books; // FetchType.LAZY по умолчанию
}

// Проблема:
List<Author> authors = authorRepository.findAll(); // 1 запрос
for (Author a : authors) {
    System.out.println(a.getBooks().size()); // N запросов — по одному на каждого автора!
}
// Итого: 1 + N запросов вместо 1
```

**Почему возникает:** LAZY загрузка — `books` не загружается при загрузке автора. При первом обращении к `books` Hibernate делает отдельный SELECT для каждого автора.

**Как обнаружить:**
```yaml
# Включить логирование SQL
spring.jpa.show-sql: true
logging.level.org.hibernate.SQL: DEBUG
logging.level.org.hibernate.type.descriptor.sql: TRACE  # значения параметров

# Или Hibernate Statistics
spring.jpa.properties.hibernate.generate_statistics: true
```

Смотришь в логах: если видишь одинаковые SELECT запросы N раз — N+1.

### Решение 1: JOIN FETCH в JPQL

```java
// Загружаем авторов сразу с книгами — один запрос
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// С условием
@Query("SELECT a FROM Author a JOIN FETCH a.books b WHERE a.id = :id")
Optional<Author> findByIdWithBooks(@Param("id") Long id);

// LEFT JOIN FETCH — включает авторов без книг
@Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
List<Author> findAllWithBooksOrEmpty();
// DISTINCT нужен: без него автор с 3 книгами = 3 записи в результате
```

> [!warning] JOIN FETCH и пагинация несовместимы
> ```java
> // ❌ Hibernate предупреждение: HHH90003004
> @Query("SELECT a FROM Author a JOIN FETCH a.books")
> Page<Author> findAll(Pageable pageable); // пагинация в памяти (не в БД)!
> ```
> При JOIN FETCH с коллекцией Hibernate **не может** применить LIMIT в SQL — он загружает ВСЕ записи в память и пагинирует там. На больших таблицах — OutOfMemoryError.

### Решение 2: @EntityGraph

```java
// Определение EntityGraph на Entity
@Entity
@NamedEntityGraph(
    name = "author-with-books",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author { ... }

// Использование в репозитории
@EntityGraph(attributePaths = {"books"})               // inline (предпочтительно)
List<Author> findAll();

@EntityGraph("author-with-books")                      // через @NamedEntityGraph
Optional<Author> findById(Long id);

// EntityGraph с вложенными ассоциациями
@EntityGraph(attributePaths = {"books", "books.publisher"})
List<Author> findAllWithBooksAndPublisher();
```

> [!tip] JOIN FETCH vs @EntityGraph
> `@EntityGraph` — более декларативный способ, удобен когда нужно переиспользовать на разных методах репозитория. `JOIN FETCH` — явный, гибкий, лучше для сложных запросов с условиями.

### Решение 3: Hibernate batch_size (BatchFetch)

```yaml
# application.yml — глобальная настройка
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

```java
// Или на конкретной ассоциации
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    @BatchSize(size = 100)  // загрузить книги батчами по 100
    private List<Book> books;
}
```

**Как работает:** вместо N отдельных SELECT, Hibernate группирует их в батч:
```sql
-- Без batch_size: N запросов
SELECT * FROM books WHERE author_id = 1
SELECT * FROM books WHERE author_id = 2
...

-- С batch_size=100: ⌈N/100⌉ запросов
SELECT * FROM books WHERE author_id IN (1, 2, 3, ..., 100)
SELECT * FROM books WHERE author_id IN (101, 102, ...)
```

**Преимущество:** работает с пагинацией! Не нужно менять запросы.
**Недостаток:** не полностью устраняет N+1, только уменьшает количество запросов.

### Решение 4: Subselect Fetching

```java
@Entity
public class Author {
    @OneToMany(mappedBy = "author")
    @Fetch(FetchMode.SUBSELECT)  // загрузить всё одним подзапросом
    private List<Book> books;
}
```

```sql
-- Генерирует:
SELECT * FROM authors WHERE ...
SELECT * FROM books WHERE author_id IN (SELECT id FROM authors WHERE ...)
```

Загружает ВСЕ книги для всех авторов одним запросом. Хорошо для небольших наборов данных.

### Сравнение решений N+1

| Решение | Количество запросов | Работает с пагинацией | Когда использовать |
|---------|--------------------|-----------------------|-------------------|
| JOIN FETCH | **1** | ❌ (без пагинации) | Один объект или без пагинации |
| @EntityGraph | **1** | ❌ (без пагинации) | Гибкое переиспользование |
| batch_size | ⌈N/size⌉ | ✅ | Пагинация + LAZY коллекции |
| @Subselect | **2** | ⚠️ (частично) | Загрузить всё сразу |

---

## 🔹 FetchType — EAGER vs LAZY

```java
@ManyToOne(fetch = FetchType.LAZY)   // рекомендуется — загружать по требованию
private User user;

@OneToMany(fetch = FetchType.LAZY)   // default для коллекций — оставь LAZY
private List<Order> orders;

@ManyToOne(fetch = FetchType.EAGER)  // ❌ default для @ManyToOne, @OneToOne
private Address address;              // загружает address на каждый SELECT user!
```

**Правило:** всегда используй LAZY, загружай явно через JOIN FETCH / @EntityGraph когда нужно.

> [!warning] LazyInitializationException
> Попытка обратиться к LAZY ассоциации вне транзакции:
> ```java
> User user = userRepository.findById(1L).orElseThrow(); // транзакция завершена
> user.getOrders().size(); // ❌ LazyInitializationException — нет сессии Hibernate
> ```
> Решения:
> - Загрузить в транзакции через JOIN FETCH
> - `@Transactional` на методе сервиса
> - `spring.jpa.open-in-view=false` + явная загрузка (рекомендуется)

---

## 🔹 Стратегии наследования

```java
// 1. SINGLE_TABLE (default) — одна таблица для всей иерархии
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Payment { ... }

@Entity
@DiscriminatorValue("CARD")
public class CardPayment extends Payment { ... }

// 2. TABLE_PER_CLASS — своя таблица для каждого конкретного класса
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment { ... }

// 3. JOINED — таблица для базового + таблица для каждого наследника
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment { ... }
```

| Стратегия | Таблицы | JOIN | NULL поля | Когда |
|-----------|---------|------|-----------|-------|
| SINGLE_TABLE | 1 | Нет | Много | Простая иерархия, производительность |
| JOINED | 1+N | JOIN | Нет | Нормализация важна |
| TABLE_PER_CLASS | N | UNION | Нет | Редко, проблемы с полиморфизмом |

---

## 🔹 Оптимистичная и пессимистичная блокировка

### Оптимистичная (@Version)

```java
@Entity
public class Account {
    @Version
    private Long version; // автоматически инкрементируется при UPDATE

    private BigDecimal balance;
}

// Сценарий конфликта:
// Поток A читает account (version=1), Поток B читает account (version=1)
// Поток A сохраняет (version → 2)
// Поток B пытается сохранить → WHERE id=? AND version=1 → 0 rows → OptimisticLockException
```

```java
@Transactional
public void transfer(Long id, BigDecimal amount) {
    try {
        Account account = accountRepository.findById(id).orElseThrow();
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account); // может бросить OptimisticLockException
    } catch (OptimisticLockException e) {
        // повторить попытку или сообщить пользователю
    }
}
```

**Когда:** редкие конфликты, высокая производительность чтения.

### Пессимистичная (@Lock)

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findByIdForUpdate(@Param("id") Long id);
// Генерирует: SELECT ... FOR UPDATE — строка блокируется до конца транзакции
```

**Когда:** частые конфликты, критичные операции (финансы), короткие транзакции.

---

## 🔹 Кэши Hibernate

### Кэш первого уровня (L1) — Session Cache

```java
// Автоматически — в рамках одной сессии (транзакции)
User u1 = userRepo.findById(1L).orElseThrow(); // SELECT
User u2 = userRepo.findById(1L).orElseThrow(); // НЕТ SELECT — из L1 кэша
// u1 == u2 — тот же объект
```

L1 всегда включён, очищается при закрытии сессии.

### Кэш второго уровня (L2) — Shared Cache

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... } // кэшируется между сессиями
```

```java
// Query Cache — кэш результатов запросов (отдельно от L2)
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Country> findAll();
```

| | L1 (Session) | L2 (Shared) | Query Cache |
|-|-------------|-------------|-------------|
| Область | Одна сессия | Весь приложение | Весь приложение |
| Включён по умолчанию | ✅ | ❌ | ❌ |
| Что кэшируется | Entities | Entities | Результаты запросов |
| Когда использовать | Всегда | Справочники, редко меняемые | Тяжёлые запросы |

---

## 🔹 Open Session in View Anti-Pattern

```yaml
spring.jpa.open-in-view: true  # default в Spring Boot ❌
```

**Что делает:** держит Hibernate Session открытой на время всего HTTP запроса (включая рендеринг View). Позволяет LAZY загрузку в контроллерах.

**Проблема:** DB соединение удерживается на всё время запроса. При медленном рендеринге — соединение занято зря. Скрывает N+1 проблемы.

```yaml
spring.jpa.open-in-view: false  # ✅ рекомендуется
```

При `false` — LazyInitializationException вне `@Transactional` даёт понять где именно нужна явная загрузка.

---

## 🔹 Типичные вопросы и ответы

**Q: Что такое N+1 проблема и как её решить?**
A: При LAZY загрузке коллекции Hibernate делает 1 запрос для родительских сущностей и N запросов для каждой дочерней коллекции. Решения: `JOIN FETCH` или `@EntityGraph` для единичных запросов; `batch_size` для пагинации.

**Q: Чем отличается JPQL от SQL?**
A: JPQL работает с объектами и их полями (`SELECT u FROM User u`), а не с таблицами и столбцами. Hibernate транслирует JPQL в SQL с учётом диалекта БД.

**Q: Когда save() возвращает новый объект и зачем его использовать?**
A: `save()` возвращает managed entity. Если entity transient — возвращает managed копию с установленным ID. Для detached entity — возвращает merged копию. Всегда используй возвращаемое значение: `user = userRepo.save(user)`.

**Q: В чём разница findById() и getReferenceById()?**
A: `findById()` — делает SELECT немедленно, возвращает `Optional`. `getReferenceById()` — возвращает lazy proxy без SELECT (нужен только ID для создания FK). Используй `getReferenceById()` когда нужна только ссылка для установки FK отношения.

```java
// ✅ getReferenceById — не делает SELECT, просто нужна ссылка для FK
order.setUser(userRepo.getReferenceById(userId));
orderRepo.save(order);
```

**Q: Что такое dirty checking и когда UPDATE генерируется автоматически?**
A: Hibernate сравнивает снимок entity при загрузке с текущим состоянием при flush. Если есть изменения — генерирует UPDATE. Происходит при: явном `flush()`, завершении транзакции (`commit`), перед выполнением запроса (если `FlushMode = AUTO`).

**Q: Как работает оптимистичная блокировка через @Version?**
A: Hibernate добавляет `WHERE id=? AND version=?` к UPDATE. Если другая транзакция уже изменила запись — version изменился, 0 rows обновлено → `OptimisticLockException`. Version инкрементируется автоматически.

**Q: Почему EAGER загрузка — антипаттерн?**
A: Загружает связанные данные при каждом SELECT родителя, даже если они не нужны. При нескольких EAGER ассоциациях — декартово произведение JOIN'ов → много лишних данных. Сложно контролировать производительность.

**Q: Как избежать LazyInitializationException?**
A: 1) Загружать нужные данные внутри `@Transactional` через JOIN FETCH/@EntityGraph. 2) Использовать Projections/DTO. 3) `spring.jpa.open-in-view=false` чтобы ошибка была явной. НЕ делать `open-in-view=true` как "решение".

---

## 🔹 Шпаргалка

```
N+1 проблема:
  Симптом: N SQL запросов в цикле
  Диагностика: spring.jpa.show-sql=true + hibernate.generate_statistics=true
  
  Решения:
    JOIN FETCH          → один запрос, нет пагинации
    @EntityGraph        → декларативно, нет пагинации
    @BatchSize(size=N)  → ⌈N/size⌉ запросов, работает с пагинацией ✅

FetchType:
  LAZY  = загружать по требованию (рекомендуется всегда)
  EAGER = загружать сразу (антипаттерн, порождает N+1)
  @ManyToOne/@OneToOne — EAGER по умолчанию → явно ставь LAZY

Жизненный цикл: Transient → Managed → Detached → Removed
  Managed: dirty checking → автоматический UPDATE, save() не нужен

Блокировки:
  @Version         → оптимистичная, редкие конфликты, нет блокировки БД
  @Lock(PESSIMISTIC_WRITE) → пессимистичная, SELECT FOR UPDATE

Кэши:
  L1 (Session)  — всегда, в рамках транзакции
  L2 (Shared)   — справочники, @Cache на entity
  Query Cache   — результаты запросов, @QueryHint

Антипаттерны:
  open-in-view=true  — держит соединение на весь HTTP запрос
  FetchType.EAGER    — N+1 при любом findAll()
  save() без результата для detached entity
  Обращение к LAZY коллекции вне @Transactional

getReferenceById() vs findById():
  findById()         → SELECT немедленно, Optional
  getReferenceById() → lazy proxy, нет SELECT (для FK ссылок)
```
