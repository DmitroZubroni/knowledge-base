# SQL Advanced — CTE, Window Functions, Transactions, Indexes

> [!abstract] Связи
> [[00_SQL]] | [[SQL_DML]] | [[SQL_Joins]] | [[PG_Performance]] | [[PG_Transactions]]

Продвинутые возможности SQL: CTE, оконные функции, транзакции, индексы, нормализация.

---

## 🔹 CTE — Common Table Expressions

> [!note] CTE (WITH)
> Именованный временный результат запроса, который существует только в рамках одного SQL-выражения. Делает сложные запросы читаемыми.

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT * FROM cte_name;
```

### Пример: без CTE vs с CTE

### ❌ Трудно читать

```sql
SELECT u.username, stats.total
FROM users u
JOIN (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    WHERE status = 'paid'
    GROUP BY user_id
) stats ON u.id = stats.user_id
WHERE stats.total > 1000;
```

### ✅ С CTE — чисто и понятно

```sql
WITH paid_orders AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    WHERE status = 'paid'
    GROUP BY user_id
),
big_spenders AS (
    SELECT user_id, total
    FROM paid_orders
    WHERE total > 1000
)
SELECT u.username, bs.total
FROM users u
JOIN big_spenders bs ON u.id = bs.user_id;
```

### Несколько CTE

```sql
WITH
    active_users AS (
        SELECT id FROM users WHERE status = 'active'
    ),
    recent_orders AS (
        SELECT user_id, COUNT(*) AS cnt
        FROM orders
        WHERE created_at > NOW() - INTERVAL '30 days'
        GROUP BY user_id
    )
SELECT u.id, ro.cnt
FROM active_users u
JOIN recent_orders ro ON u.id = ro.user_id;
```

### Рекурсивный CTE

> [!note] RECURSIVE CTE
> Позволяет делать рекурсивные запросы — иерархии, деревья, графы.

```sql
-- обход иерархии сотрудников
WITH RECURSIVE org_tree AS (
    -- базовый случай: топ-менеджер (без начальника)
    SELECT id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- рекурсивный шаг: подчинённые текущего уровня
    SELECT e.id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT level, name FROM org_tree ORDER BY level, name;
```

```
Вывод:
level=0  CEO
level=1  CTO, CFO
level=2  Dev Lead, QA Lead
level=3  Alice, Bob, ...
```

---

## 🔹 Оконные функции (Window Functions)

> [!note] Оконная функция
> Вычисляет значение для каждой строки **с учётом соседних строк** (окна), не схлопывая результат как GROUP BY.

```sql
функция() OVER (
    PARTITION BY col    -- разбить на группы (как GROUP BY, но строки остаются)
    ORDER BY col        -- порядок внутри окна
    ROWS/RANGE BETWEEN  -- рамка окна (опционально)
)
```

### Ранжирующие функции

```sql
SELECT
    username,
    department,
    salary,
    ROW_NUMBER()   OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()         OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK()   OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank,
    NTILE(4)       OVER (ORDER BY salary DESC)                         AS quartile
FROM employees;
```

| Функция | Описание | При одинаковых значениях |
|---------|----------|--------------------------|
| `ROW_NUMBER()` | уникальный номер строки | всегда разные |
| `RANK()` | ранг с пропусками | одинаковый ранг, следующий пропущен |
| `DENSE_RANK()` | ранг без пропусков | одинаковый ранг, следующий подряд |
| `NTILE(n)` | разбить на n равных групп | — |

```
salary: 100, 100, 80, 60
ROW_NUMBER:  1,   2,  3,  4
RANK:        1,   1,  3,  4   ← пропуск 2
DENSE_RANK:  1,   1,  2,  3   ← без пропуска
```

### Аналитические функции

```sql
SELECT
    id,
    amount,
    department,
    -- бегущая сумма
    SUM(amount)    OVER (PARTITION BY department ORDER BY created_at) AS running_total,
    -- скользящее среднее (текущая + 2 предыдущие)
    AVG(amount)    OVER (ORDER BY created_at ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg,
    -- разница с предыдущей строкой
    LAG(amount, 1) OVER (ORDER BY created_at)  AS prev_amount,
    LEAD(amount, 1) OVER (ORDER BY created_at) AS next_amount,
    -- первое и последнее значение в группе
    FIRST_VALUE(amount) OVER (PARTITION BY department ORDER BY created_at) AS first_in_dept,
    LAST_VALUE(amount)  OVER (PARTITION BY department ORDER BY created_at
                              ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_in_dept
FROM orders;
```

| Функция | Описание |
|---------|----------|
| `LAG(col, n)` | значение col из n строк **назад** |
| `LEAD(col, n)` | значение col из n строк **вперёд** |
| `FIRST_VALUE(col)` | первое значение в окне |
| `LAST_VALUE(col)` | последнее значение в окне |
| `NTH_VALUE(col, n)` | n-е значение в окне |

### Рамка окна (Frame)

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- от начала до текущей
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING           -- 2 до + текущая + 2 после
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING   -- от текущей до конца
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- по значениям (не строкам)
```

### Практический пример: TOP-N в каждой группе

```sql
-- топ-3 заказа по сумме для каждого пользователя
WITH ranked AS (
    SELECT
        user_id,
        id AS order_id,
        amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
    FROM orders
)
SELECT user_id, order_id, amount
FROM ranked
WHERE rn <= 3;
```

---

## 🔹 Транзакции

> [!note] Транзакция
> Группа SQL-операций, которые выполняются атомарно — либо все, либо ни одна.

### ACID

| Свойство | Описание |
|----------|----------|
| **Atomicity** | Все операции выполняются или ни одна |
| **Consistency** | БД остаётся в корректном состоянии |
| **Isolation** | Транзакции не мешают друг другу |
| **Durability** | Зафиксированные данные не теряются |

### Управление транзакцией

```sql
BEGIN;                          -- начать транзакцию

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

COMMIT;                         -- зафиксировать

-- или откатить при ошибке
ROLLBACK;
```

### SAVEPOINT

```sql
BEGIN;

INSERT INTO orders (user_id, amount) VALUES (1, 100);

SAVEPOINT sp1;                  -- точка сохранения

INSERT INTO order_items (...) VALUES (...);

-- если что-то пошло не так только во второй части
ROLLBACK TO SAVEPOINT sp1;      -- откат к точке, не всей транзакции

COMMIT;
```

### Уровни изоляции

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|-----------|---------------------|--------------|
| `READ UNCOMMITTED` | ✅ возможно | ✅ возможно | ✅ возможно |
| `READ COMMITTED` *(default)* | ❌ | ✅ возможно | ✅ возможно |
| `REPEATABLE READ` | ❌ | ❌ | ✅ возможно |
| `SERIALIZABLE` | ❌ | ❌ | ❌ |

```sql
-- установить уровень для транзакции
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

> [!note] Аномалии
> - **Dirty Read** — читаем незафиксированные данные другой транзакции
> - **Non-repeatable Read** — повторный SELECT возвращает другой результат
> - **Phantom Read** — повторный SELECT возвращает новые строки

> [!tip] PostgreSQL по умолчанию
> `READ COMMITTED` — в большинстве случаев достаточно. `REPEATABLE READ` — для отчётов, где важна консистентность снимка данных.

---

## 🔹 Индексы — подробно

> [!note] Index
> Вспомогательная структура, ускоряющая поиск за счёт хранения отсортированных указателей на строки.

### Типы индексов

| Тип | Применение |
|-----|-----------|
| `B-tree` | равенство, диапазон, сортировка (по умолчанию) |
| `Hash` | только равенство (`=`) |
| `GIN` | массивы, JSON, полнотекстовый поиск |
| `GiST` | геометрия, диапазоны, полнотекст |
| `BRIN` | очень большие таблицы с физически упорядоченными данными |

```sql
-- B-tree (по умолчанию)
CREATE INDEX idx_users_email ON users(email);

-- Hash
CREATE INDEX idx_orders_status_hash ON orders USING HASH (status);

-- Составной
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Частичный (partial) — только для подмножества строк
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Покрывающий (covering) — INCLUDE
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (amount, status);

-- Функциональный
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
```

### ❌ Индекс не используется когда

```sql
-- функция над индексированной колонкой
WHERE LOWER(email) = 'test@mail.com'    -- нужен функциональный индекс

-- LIKE с префиксом %
WHERE email LIKE '%gmail.com'           -- не использует B-tree

-- неявное приведение типов
WHERE id = '123'                        -- id INTEGER, сравниваем со строкой

-- слишком мало уникальных значений (низкая кардинальность)
WHERE status = 'active'                 -- если 90% строк active — индекс не нужен
```

### ✅ Индекс используется когда

```sql
WHERE email = 'test@gmail.com'
WHERE age BETWEEN 18 AND 30
WHERE user_id = 5 AND status = 'paid'   -- составной индекс (user_id, status)
ORDER BY created_at DESC LIMIT 10       -- индекс для сортировки
```

> [!warning] Порядок в составном индексе
> `INDEX ON orders(user_id, status)` работает для `WHERE user_id = 5` и `WHERE user_id = 5 AND status = 'paid'`, но **НЕ** для `WHERE status = 'paid'` (без user_id).

---

## 🔹 EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@gmail.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@gmail.com';
```

```
Seq Scan on users  (cost=0.00..25.00 rows=1 width=100)
  Filter: (email = 'test@gmail.com')

-- после создания индекса:
Index Scan using idx_users_email on users (cost=0.15..8.17 rows=1 width=100)
  Index Cond: (email = 'test@gmail.com')
```

| Узел плана | Описание |
|-----------|----------|
| `Seq Scan` | полное сканирование таблицы |
| `Index Scan` | сканирование по индексу |
| `Index Only Scan` | данные из индекса, таблица не нужна |
| `Bitmap Heap Scan` | несколько индексных условий, потом таблица |
| `Hash Join` | соединение через хэш-таблицу |
| `Nested Loop` | соединение вложенным циклом |
| `Merge Join` | соединение слиянием отсортированных |

> [!tip] Смотри на cost и actual time
> `EXPLAIN` — оценка, `EXPLAIN ANALYZE` — реальное выполнение. Разрыв между оценкой и реальностью = устаревшая статистика → запусти `ANALYZE table_name`.

---

## 🔹 Нормализация

> [!note] Нормальная форма
> Набор правил для устранения избыточности и аномалий в реляционной схеме.

### 1NF — Первая нормальная форма
- Каждая ячейка содержит атомарное значение
- Нет повторяющихся групп столбцов

### ❌ Нарушение 1NF
```
users: id | name | phones
       1  | Alice | "79001112233, 79004445566"
```

### ✅ 1NF
```
users:        id | name
user_phones:  user_id | phone
```

---

### 2NF — Вторая нормальная форма
- Соответствует 1NF
- Каждый неключевой атрибут полностью зависит от **всего** первичного ключа

### ❌ Нарушение 2NF (составной PK: order_id + product_id)
```
order_items: order_id | product_id | product_name | quantity
             product_name зависит только от product_id, не от всего ключа
```

### ✅ 2NF
```
order_items: order_id | product_id | quantity
products:    product_id | product_name
```

---

### 3NF — Третья нормальная форма
- Соответствует 2NF
- Нет транзитивных зависимостей (неключевой → неключевой)

### ❌ Нарушение 3NF
```
employees: id | name | department_id | department_name
           department_name зависит от department_id, а не от id
```

### ✅ 3NF
```
employees:   id | name | department_id
departments: department_id | department_name
```

> [!tip] На практике
> Для большинства задач достаточно 3NF. BCNF и выше — редкие случаи с несколькими перекрывающимися ключами-кандидатами.

---

## 🔹 Итог

```
CTE (WITH):
  Именованный подзапрос, один раз — используй многократно
  RECURSIVE — для иерархий и деревьев

Оконные функции:
  ROW_NUMBER / RANK / DENSE_RANK — ранжирование
  SUM / AVG OVER (...)            — агрегат без схлопывания
  LAG / LEAD                      — сдвиг по строкам
  PARTITION BY — разбить на группы
  ORDER BY     — порядок внутри окна
  ROWS BETWEEN — рамка окна

Транзакции:
  BEGIN / COMMIT / ROLLBACK
  SAVEPOINT / ROLLBACK TO SAVEPOINT
  Уровни: READ COMMITTED (default) → SERIALIZABLE

Индексы:
  B-tree (default) — диапазоны, равенство, сортировка
  Hash             — только равенство
  GIN              — JSON, массивы, full-text
  Составной        — порядок колонок важен (leftmost prefix)
  Частичный        — WHERE условие в индексе
  Функциональный   — LOWER(email)

EXPLAIN ANALYZE:
  Seq Scan   — плохо для больших таблиц
  Index Scan — хорошо
  cost=X..Y  — оценка, actual time — реальность

Нормализация:
  1NF — атомарные значения
  2NF — полная зависимость от PK
  3NF — нет транзитивных зависимостей
```
