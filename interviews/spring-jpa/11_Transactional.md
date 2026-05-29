# @Transactional

> **Теги:** #interviews #spring #transactional #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 @Transactional под капотом

**@Transactional** — аннотация для управления транзакциями в Spring.

### Как работает

```
1. Spring создаёт прокси вокруг bean с @Transactional
2. При вызове метода прокси перехватывает вызов
3. Прокси начинает транзакцию (TransactionManager)
4. Вызывается оригинальный метод
5. Если успешно → commit
6. Если исключение → rollback
```

### Пример

```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        repository.save(user);
    }
}
```

### Требования

- **@EnableTransactionManagement** — включение управления транзакциями (автоматически в Spring Boot)
- **PlatformTransactionManager** — bean для управления транзакциями (автоматически создаётся с JPA)
- **DataSource** — источник данных

---

## 🔹 Propagation (распространение транзакции)

**Propagation** — поведение транзакции при вызове из другой транзакции.

### Типы propagation

| Propagation | Описание |
|-------------|----------|
| **REQUIRED** (default) | Использует существующую транзакцию, если есть. Если нет — создаёт новую |
| **REQUIRES_NEW** | Всегда создаёт новую транзакцию. Приостанавливает существующую |
| **SUPPORTS** | Использует существующую, если есть. Если нет — выполняется без транзакции |
| **NOT_SUPPORTED** | Всегда выполняется без транзакции. Приостанавливает существующую |
| **MANDATORY** | Требует существующую транзакцию. Если нет — исключение |
| **NEVER** | Не должен выполняться в транзакции. Если есть — исключение |
| **NESTED** | Вложенная транзакция (savepoint) |

### Примеры

```java
@Service
public class OrderService {
    @Transactional(propagation = Propagation.REQUIRED)  // default
    public void createOrder(Order order) {
        // использует существующую транзакцию или создаёт новую
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrder(Order order) {
        // всегда новая транзакция
    }
}
```

### REQUIRED vs REQUIRES_NEW

```java
@Transactional
public void methodA() {
    methodB();  // methodB в той же транзакции
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
    // новая транзакция, methodA приостанавливается
}
```

---

## 🔹 Self-Invocation ловушка

**Self-Invocation** — вызов метода из того же класса, @Transactional не работает!

### Пример проблемы

```java
@Service
public class UserService {
    @Transactional
    public void methodA() {
        methodB();  // @Transactional на methodB не сработает!
    }

    @Transactional
    public void methodB() {
        // транзакция не открывается
    }
}
```

### Почему не работает

Spring создаёт прокси вокруг bean. При вызове `methodA()` через прокси транзакция открывается. Но внутри `methodA()` вызов `methodB()` идёт напрямую на `this`, минуя прокси.

### Решения

#### 1. Внедрить себя через @Lazy

```java
@Service
public class UserService {
    @Autowired
    @Lazy
    private UserService self;

    @Transactional
    public void methodA() {
        self.methodB();  // вызов через прокси → @Transactional сработает
    }

    @Transactional
    public void methodB() {
        // ...
    }
}
```

#### 2. Вынести в отдельный bean

```java
@Service
public class UserService {
    @Autowired
    private TransactionService transactionService;

    @Transactional
    public void methodA() {
        transactionService.methodB();  // вызов через другой bean
    }
}

@Service
public class TransactionService {
    @Transactional
    public void methodB() {
        // ...
    }
}
```

---

## 🔹 Rollback (откат транзакции)

**Rollback** — когда транзакция откатывается.

### По умолчанию

Транзакция откатывается при **RuntimeException** и **Error**.

```java
@Transactional
public void createUser(User user) {
    repository.save(user);
    throw new RuntimeException("Error");  // rollback
}
```

**НЕ откатывается** при checked exceptions:

```java
@Transactional
public void createUser(User user) throws IOException {
    repository.save(user);
    throw new IOException("Error");  // NO rollback!
}
```

### rollbackFor / noRollbackFor

```java
@Transactional(rollbackFor = IOException.class)
public void createUser(User user) throws IOException {
    repository.save(user);
    throw new IOException("Error");  // rollback
}

@Transactional(noRollbackFor = IllegalArgumentException.class)
public void createUser(User user) {
    repository.save(user);
    throw new IllegalArgumentException("Error");  // NO rollback
}
```

### rollbackForClassName

```java
@Transactional(rollbackForClassName = "java.io.IOException")
public void createUser(User user) throws IOException {
    repository.save(user);
    throw new IOException("Error");  // rollback
}
```

---

## 🔹 Isolation (уровень изоляции)

**Isolation** — уровень изоляции транзакции от других транзакций.

### Уровни

| Isolation | Описание |
|-----------|----------|
| **DEFAULT** (default) | Использует уровень изоляции БД по умолчанию |
| **READ_UNCOMMITTED** | Чтение незафиксированных данных (dirty reads) |
| **READ_COMMITTED** | Чтение только зафиксированных данных |
| **REPEATABLE_READ** | Гарантирует, что при повторном чтении те же данные |
| **SERIALIZABLE** | Полная изоляция, транзакции выполняются последовательно |

### Пример

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void createUser(User user) {
    repository.save(user);
}
```

### Проблемы изоляции

| Проблема | READ_UNCOMMITTED | READ_COMMITTED | REPEATABLE_READ | SERIALIZABLE |
|----------|------------------|----------------|----------------|--------------|
| Dirty Reads | ✅ Возможны | ❌ Нет | ❌ Нет | ❌ Нет |
| Non-repeatable Reads | ✅ Возможны | ✅ Возможны | ❌ Нет | ❌ Нет |
| Phantom Reads | ✅ Возможны | ✅ Возможны | ✅ Возможны | ❌ Нет |

---

## 🔹 Timeout (таймаут транзакции)

**Timeout** — время выполнения транзакции в секундах.

```java
@Transactional(timeout = 5)  // 5 секунд
public void longRunningOperation() {
    // если занимает больше 5 секунд → rollback
}
```

### Default timeout

По умолчанию зависит от TransactionManager (обычно нет таймаута).

---

## 🔹 readOnly

**readOnly** — транзакция только для чтения.

```java
@Transactional(readOnly = true)
public User getUser(Long id) {
    return repository.findById(id);
}
```

### Преимущества

- Hibernate не проверяет dirty checking
- Оптимизация JDBC драйвера
- Лучшая производительность

> [!warning] readOnly только для чтения
```java
@Transactional(readOnly = true)
public void updateUser(User user) {
    repository.save(user);  // может не сработать или быть медленнее
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **@Transactional** — прокси перехватывает вызовы, управляет транзакцией
> - **Propagation:** REQUIRED (default), REQUIRES_NEW, SUPPORTS
> - **Self-Invocation** — не работает, решение: @Lazy или отдельный bean
> - **Rollback:** RuntimeException/Error по умолчанию, rollbackFor для checked
> - **Isolation:** DEFAULT (БД), READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
> - **readOnly** — оптимизация для чтения

```
@Transactional под капотом:
прокси → TransactionManager → commit/rollback

Propagation:
REQUIRED → использует существующую или создаёт новую
REQUIRES_NEW → всегда новая

Self-Invocation:
this.methodB() → не работает
self.methodB() (@Lazy) → работает

Rollback:
RuntimeException/Error → rollback по умолчанию
rollbackFor = IOException.class → rollback для checked

Isolation:
DEFAULT → уровень БД
READ_COMMITTED → без dirty reads
SERIALIZABLE → полная изоляция
```
