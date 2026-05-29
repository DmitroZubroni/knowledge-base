# SQL Database

> **Теги:** #interviews #database #sql #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 ACID

**ACID** — свойства транзакций в БД.

### A - Atomicity (Атомарность)

Транзакция выполняется полностью или не выполняется вообще.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- или ROLLBACK если ошибка
```

### C - Consistency (Согласованность)

Транзакция переводит БД из одного согласованного состояния в другое.

```sql
-- После транзакции все ограничения соблюдены
-- Баланс не может быть отрицательным
```

### I - Isolation (Изоляция)

Транзакции не мешают друг другу.

### D - Durability (Долговечность)

После COMMIT изменения сохраняются даже при сбое.

---

## 🔹 Уровни изоляции

### Уровни (SQL стандарт)

| Уровень | Dirty Reads | Non-repeatable Reads | Phantom Reads |
|---------|-------------|----------------------|---------------|
| **READ UNCOMMITTED** | ✅ Возможны | ✅ Возможны | ✅ Возможны |
| **READ COMMITTED** | ❌ Нет | ✅ Возможны | ✅ Возможны |
| **REPEATABLE READ** | ❌ Нет | ❌ Нет | ✅ Возможны |
| **SERIALIZABLE** | ❌ Нет | ❌ Нет | ❌ Нет |

### Феномены

#### Dirty Read (Грязное чтение)

Чтение незафиксированных данных другой транзакции.

```sql
-- Транзакция 1
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;  -- не COMMIT

-- Транзакция 2
SELECT balance FROM accounts WHERE id = 1;  -- читает 100 (грязное чтение)
```

#### Non-repeatable Read (Неповторяемое чтение)

При повторном чтении получаем разные данные.

```sql
-- Транзакция 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Транзакция 2
UPDATE accounts SET balance = 200 WHERE id = 1;
COMMIT;

-- Транзакция 1
SELECT balance FROM accounts WHERE id = 1;  -- 200 (неповторяемое чтение)
```

#### Phantom Read (Фантомное чтение)

При повторном чтении получаем разное количество строк.

```sql
-- Транзакция 1
BEGIN;
SELECT * FROM accounts WHERE balance > 100;  -- 5 строк

-- Транзакция 2
INSERT INTO accounts VALUES (6, 150);
COMMIT;

-- Транзакция 1
SELECT * FROM accounts WHERE balance > 100;  -- 6 строк (фантомное чтение)
```

### PostgreSQL уровни

| Уровень | Dirty Reads | Non-repeatable Reads | Phantom Reads |
|---------|-------------|----------------------|---------------|
| **READ COMMITTED** (default) | ❌ Нет | ✅ Возможны | ✅ Возможны |
| **REPEATABLE READ** | ❌ Нет | ❌ Нет | ❌ Нет (MVCC) |
| **SERIALIZABLE** | ❌ Нет | ❌ Нет | ❌ Нет |

> [!note] PostgreSQL REPEATABLE READ предотвращает фантомные чтения через MVCC

---

## 🔹 Блокировки

### Типы блокировок

| Тип | Описание |
|-----|----------|
| **Shared Lock (S)** | Блокировка на чтение, несколько S-блокировок совместимы |
| **Exclusive Lock (X)** | Блокировка на запись, несовместима ни с чем |

### Совместимость блокировок

| | S | X |
|---|---|---|
| **S** | ✅ | ❌ |
| **X** | ❌ | ❌ |

### Пример

```sql
-- Shared Lock (SELECT FOR SHARE)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- S-блокировка

-- Другая транзакция может читать
SELECT * FROM accounts WHERE id = 1;  -- OK

-- Но не может писать
UPDATE accounts SET balance = 0 WHERE id = 1;  -- BLOCKED

-- Exclusive Lock (SELECT FOR UPDATE)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- X-блокировка

-- Другая транзакция не может ни читать, ни писать
SELECT * FROM accounts WHERE id = 1;  -- BLOCKED
```

### Deadlock (Взаимная блокировка)

```sql
-- Транзакция 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- BLOCKED

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- BLOCKED

-- Deadlock! БД откатит одну транзакцию
```

---

## 🔹 JOIN vs UNION

### JOIN

Объединение строк из разных таблиц по условию.

```sql
SELECT users.name, orders.product
FROM users
JOIN orders ON users.id = orders.user_id;
```

### UNION

Объединение результатов двух запросов (удаление дубликатов).

```sql
SELECT name FROM users
UNION
SELECT name FROM admins;
```

### UNION ALL

Объединение результатов без удаления дубликатов.

```sql
SELECT name FROM users
UNION ALL
SELECT name FROM admins;
```

### Сравнение

| Операция | Назначение | Дубликаты |
|----------|------------|-----------|
| **JOIN** | Объединение по условию | Сохраняются |
| **UNION** | Объединение результатов | Удаляются |
| **UNION ALL** | Объединение результатов | Сохраняются |

---

## 🔹 Foreign Key (FK)

**Foreign Key** — внешний ключ, ссылка на другую таблицу.

### Создание FK

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    product VARCHAR(100),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### ON DELETE / ON UPDATE

| Действие | Описание |
|-----------|----------|
| **CASCADE** | Каскадное удаление/обновление |
| **RESTRICT** (default) | Запрещает удаление/обновление |
| **SET NULL** | Устанавливает NULL |
| **SET DEFAULT** | Устанавливает значение по умолчанию |
| **NO ACTION** | Аналогично RESTRICT |

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### Пример CASCADE

```sql
DELETE FROM users WHERE id = 1;  -- удалятся все связанные orders
```

---

## 🔹 Constraints (Ограничения)

### Типы ограничений

| Ограничение | Описание |
|-------------|----------|
| **PRIMARY KEY** | Уникальный идентификатор |
| **UNIQUE** | Уникальное значение |
| **NOT NULL** | Обязательное поле |
| **CHECK** | Проверка условия |
| **FOREIGN KEY** | Ссылка на другую таблицу |

### Примеры

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) NOT NULL,
    age INTEGER CHECK (age >= 18),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Добавление ограничений

```sql
-- UNIQUE
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

-- CHECK
ALTER TABLE users ADD CONSTRAINT check_age CHECK (age >= 18);

-- NOT NULL
ALTER TABLE users ALTER COLUMN username SET NOT NULL;
```

---

## 🔹 Индексы

**Index** — структура данных для ускорения поиска.

### Создание индекса

```sql
CREATE INDEX idx_users_username ON users(username);
```

### Типы индексов

| Тип | Описание |
|-----|----------|
| **B-Tree** (default) | Для равенства и диапазонов |
| **Hash** | Только для равенства |
| **GIN** | Для массивов и JSON |
| **GiST** | Для геометрических данных |

### Композитный индекс

```sql
CREATE INDEX idx_users_name_age ON users(name, age);
```

> [!tip] Порядок колонок важен
```sql
-- Индекс (name, age)
WHERE name = 'John' AND age = 30  -- использует индекс
WHERE age = 30                     -- НЕ использует индекс
```

### Когда индекс НЕ используется

- Функция на колонке: `WHERE LOWER(name) = 'john'`
- LIKE с префиксом: `WHERE name LIKE '%john'`
- OR: `WHERE name = 'John' OR age = 30`

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **ACID:** Atomicity, Consistency, Isolation, Durability
> - **Уровни изоляции:** READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE
> - **Феномены:** Dirty Reads, Non-repeatable Reads, Phantom Reads
> - **Блокировки:** Shared (S), Exclusive (X), Deadlock
> - **JOIN** — объединение по условию, **UNION** — объединение результатов
> - **FK:** CASCADE, RESTRICT, SET NULL
> - **Constraints:** PRIMARY KEY, UNIQUE, NOT NULL, CHECK
> - **Индексы:** B-Tree, Hash, GIN, композитные

```
ACID:
A - Atomicity (всё или ничего)
C - Consistency (согласованность)
I - Isolation (изоляция)
D - Durability (долговечность)

Уровни изоляции:
READ UNCOMMITTED → грязное чтение
READ COMMITTED → без грязного чтения
REPEATABLE READ → без неповторяемого чтения
SERIALIZABLE → полная изоляция

Блокировки:
S (Shared) → чтение, совместимы
X (Exclusive) → запись, несовместимы

JOIN vs UNION:
JOIN → по условию
UNION → объединение результатов (без дубликатов)
UNION ALL → с дубликатами

FK:
ON DELETE CASCADE → каскадное удаление
ON DELETE RESTRICT → запрещает

Constraints:
PRIMARY KEY → уникальный идентификатор
UNIQUE → уникальное значение
NOT NULL → обязательно
CHECK → проверка условия

Индексы:
B-Tree → равенство + диапазоны
Hash → только равенство
Композитный → порядок колонок важен
```
