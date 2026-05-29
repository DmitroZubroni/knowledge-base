# SQL DML — Data Manipulation Language

> **Теги:** #sql #database #dml #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[SQL]]

DML — команды для работы с **данными**: выборка, вставка, изменение, удаление.

---

## 🔹 SELECT — базовая выборка

```sql
SELECT column1, column2        -- что выбираем (* = все)
FROM table_name                -- откуда
WHERE condition                -- фильтр строк
GROUP BY column                -- группировка
HAVING condition               -- фильтр групп
ORDER BY column ASC|DESC       -- сортировка
LIMIT n                        -- ограничение количества
OFFSET n;                      -- пропустить n строк
```

> [!note] Порядок выполнения SELECT (логический)
> ```
> FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT/OFFSET
> ```
> Важно: `WHERE` выполняется **до** `SELECT`, поэтому алиасы из SELECT нельзя использовать в WHERE.

### Примеры

```sql
-- все записи
SELECT * FROM users;

-- конкретные колонки с алиасом
SELECT id, username AS name, email FROM users;

-- уникальные значения
SELECT DISTINCT role FROM users;

-- вычисляемые колонки
SELECT
    id,
    first_name || ' ' || last_name AS full_name,
    age * 12 AS age_months
FROM users;
```

---

## 🔹 WHERE — фильтрация

```sql
-- сравнение
WHERE age > 18
WHERE age >= 18 AND age <= 65
WHERE status = 'active'
WHERE status != 'deleted'

-- диапазон
WHERE age BETWEEN 18 AND 65       -- включительно

-- список значений
WHERE role IN ('admin', 'manager')
WHERE role NOT IN ('guest')

-- поиск по шаблону
WHERE email LIKE '%@gmail.com'    -- % = любое количество символов
WHERE username LIKE 'a_ex'        -- _ = ровно один символ
WHERE email ILIKE '%@gmail%'      -- ILIKE = регистронезависимо (PostgreSQL)

-- NULL
WHERE phone IS NULL
WHERE phone IS NOT NULL

-- логические операторы
WHERE age > 18 AND role = 'admin'
WHERE age < 18 OR role = 'guest'
WHERE NOT (status = 'deleted')
```

> [!warning] NULL и сравнения
> `WHERE phone = NULL` — **не работает**. Всегда используй `IS NULL` / `IS NOT NULL`.
> NULL не равен ничему, включая другой NULL.

---

## 🔹 ORDER BY и LIMIT

```sql
-- сортировка по одной колонке
SELECT * FROM users ORDER BY created_at DESC;

-- сортировка по нескольким
SELECT * FROM users ORDER BY role ASC, username ASC;

-- NULL в сортировке
ORDER BY age ASC NULLS LAST     -- NULL в конце
ORDER BY age ASC NULLS FIRST    -- NULL в начале

-- пагинация
SELECT * FROM users
ORDER BY id
LIMIT 10 OFFSET 20;             -- страница 3 (по 10 записей)
```

> [!tip] Пагинация через OFFSET
> При больших таблицах `OFFSET` работает медленно — БД всё равно читает все пропущенные строки. Для высоконагруженных систем используй пагинацию по курсору (keyset pagination): `WHERE id > :last_id LIMIT 10`.

---

## 🔹 GROUP BY и агрегатные функции

> [!note] Агрегатные функции
> Вычисляют одно значение для группы строк. Используются вместе с `GROUP BY` или для всей таблицы.

```sql
-- без GROUP BY — агрегат по всей таблице
SELECT COUNT(*) FROM users;
SELECT AVG(age) FROM users;
SELECT MAX(amount) FROM orders;
SELECT MIN(amount) FROM orders;
SELECT SUM(amount) FROM orders;

-- с GROUP BY
SELECT
    role,
    COUNT(*)        AS total,
    AVG(age)        AS avg_age,
    MAX(created_at) AS last_registered
FROM users
GROUP BY role;

-- фильтрация групп через HAVING
SELECT role, COUNT(*) AS cnt
FROM users
GROUP BY role
HAVING COUNT(*) > 10;    -- только роли с более чем 10 пользователями
```

| Функция | Описание |
|---------|----------|
| `COUNT(*)` | количество строк |
| `COUNT(col)` | количество NOT NULL значений |
| `COUNT(DISTINCT col)` | количество уникальных значений |
| `SUM(col)` | сумма |
| `AVG(col)` | среднее |
| `MAX(col)` | максимум |
| `MIN(col)` | минимум |
| `STRING_AGG(col, sep)` | конкатенация строк через разделитель |
| `ARRAY_AGG(col)` | собрать в массив (PostgreSQL) |

> [!warning] WHERE vs HAVING
> - `WHERE` — фильтрует строки **до** группировки
> - `HAVING` — фильтрует группы **после** группировки
> ```sql
> -- правильно: отфильтровать active ДО группировки
> SELECT role, COUNT(*) FROM users WHERE status = 'active' GROUP BY role;
> -- HAVING нужен только для условий на агрегаты
> HAVING COUNT(*) > 10
> ```

---

## 🔹 INSERT

```sql
-- вставка одной строки
INSERT INTO users (username, email, age)
VALUES ('john', 'john@example.com', 25);

-- вставка нескольких строк
INSERT INTO users (username, email, age)
VALUES
    ('alice', 'alice@example.com', 30),
    ('bob',   'bob@example.com',   22);

-- вставка из SELECT
INSERT INTO archived_users (username, email)
SELECT username, email FROM users WHERE status = 'deleted';

-- вернуть вставленные данные (RETURNING — PostgreSQL)
INSERT INTO users (username, email)
VALUES ('kate', 'kate@example.com')
RETURNING id, created_at;
```

### INSERT ON CONFLICT (Upsert)

```sql
-- игнорировать конфликт
INSERT INTO users (username, email)
VALUES ('john', 'john@example.com')
ON CONFLICT (username) DO NOTHING;

-- обновить при конфликте
INSERT INTO users (username, email, login_count)
VALUES ('john', 'john@example.com', 1)
ON CONFLICT (username) DO UPDATE
    SET email       = EXCLUDED.email,
        login_count = users.login_count + 1;
```

> [!note] EXCLUDED
> `EXCLUDED` — псевдотаблица с данными, которые пытались вставить, но вызвали конфликт.

---

## 🔹 UPDATE

```sql
-- обновить все строки
UPDATE users SET role = 'user';

-- обновить по условию
UPDATE users
SET role = 'admin', updated_at = NOW()
WHERE id = 1;

-- обновить несколько колонок
UPDATE orders
SET
    status     = 'shipped',
    shipped_at = CURRENT_TIMESTAMP
WHERE id = 42 AND status = 'paid';

-- UPDATE с JOIN (PostgreSQL)
UPDATE orders o
SET status = 'vip_order'
FROM users u
WHERE o.user_id = u.id AND u.role = 'vip';

-- вернуть изменённые данные
UPDATE users SET role = 'admin' WHERE id = 1
RETURNING id, username, role;
```

> [!warning] UPDATE без WHERE
> `UPDATE users SET role = 'guest'` — изменит **все** строки. Всегда проверяй WHERE перед выполнением.

---

## 🔹 DELETE

```sql
-- удалить по условию
DELETE FROM users WHERE status = 'deleted';

-- удалить все строки (медленнее TRUNCATE)
DELETE FROM users;

-- DELETE с подзапросом
DELETE FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE status = 'banned'
);

-- DELETE с JOIN (PostgreSQL)
DELETE FROM orders o
USING users u
WHERE o.user_id = u.id AND u.status = 'banned';

-- вернуть удалённые данные
DELETE FROM users WHERE id = 5
RETURNING *;
```

---

## 🔹 Подзапросы (Subqueries)

```sql
-- в WHERE
SELECT * FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE role = 'vip'
);

-- скалярный подзапрос (возвращает одно значение)
SELECT username,
       (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) AS order_count
FROM users;

-- в FROM (derived table)
SELECT avg_data.role, avg_data.avg_age
FROM (
    SELECT role, AVG(age) AS avg_age
    FROM users
    GROUP BY role
) AS avg_data
WHERE avg_data.avg_age > 25;

-- EXISTS
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- NOT EXISTS
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

| Тип | Возвращает | Использование |
|-----|-----------|---------------|
| Скалярный | 1 значение | `SELECT`, `WHERE col = (...)` |
| Строковый | 1 строку | `WHERE (col1, col2) = (...)` |
| Табличный | таблицу | `FROM (...)`, `IN (...)` |
| `EXISTS` | boolean | `WHERE EXISTS (...)` |

---

## 🔹 CASE WHEN

```sql
-- простой CASE
SELECT
    username,
    CASE role
        WHEN 'admin'   THEN 'Администратор'
        WHEN 'manager' THEN 'Менеджер'
        ELSE 'Пользователь'
    END AS role_name
FROM users;

-- поисковый CASE
SELECT
    id,
    amount,
    CASE
        WHEN amount > 10000 THEN 'high'
        WHEN amount > 1000  THEN 'medium'
        ELSE 'low'
    END AS amount_category
FROM orders;

-- CASE в WHERE, ORDER BY
SELECT * FROM orders
ORDER BY
    CASE status
        WHEN 'urgent' THEN 1
        WHEN 'normal' THEN 2
        ELSE 3
    END;
```

---

## 🔹 Полезные функции

### Строковые

```sql
LENGTH(str)                        -- длина строки
LOWER(str) / UPPER(str)            -- регистр
TRIM(str)                          -- убрать пробелы по краям
LTRIM / RTRIM                      -- только слева / справа
SUBSTRING(str FROM 1 FOR 3)        -- подстрока
REPLACE(str, 'old', 'new')         -- замена
CONCAT(a, b) / a || b              -- конкатенация
SPLIT_PART(str, delim, n)          -- часть строки по разделителю
POSITION('x' IN str)               -- позиция подстроки
```

### Числовые

```sql
ROUND(3.14159, 2)                  -- округление → 3.14
CEIL(3.1) / FLOOR(3.9)             -- вверх / вниз
ABS(-5)                            -- модуль
MOD(10, 3)                         -- остаток от деления → 1
POWER(2, 10)                       -- возведение в степень
```

### Дата и время

```sql
NOW()                              -- текущая дата + время
CURRENT_DATE                       -- текущая дата
CURRENT_TIME                       -- текущее время
EXTRACT(YEAR FROM NOW())           -- извлечь часть даты
DATE_TRUNC('month', NOW())         -- усечь до месяца
AGE(NOW(), birth_date)             -- разница дат
NOW() + INTERVAL '7 days'         -- арифметика дат
TO_CHAR(NOW(), 'DD.MM.YYYY')      -- форматирование
```

### NULL-функции

```sql
COALESCE(a, b, c)                  -- первое NOT NULL значение
NULLIF(a, b)                       -- NULL если a = b, иначе a
```

---

## 🔹 Итог

```
SELECT выполняется в порядке:
  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

Фильтрация:
  WHERE    — до группировки (строки)
  HAVING   — после группировки (группы)

Агрегаты: COUNT, SUM, AVG, MAX, MIN

Вставка:    INSERT INTO ... VALUES / SELECT
Upsert:     ON CONFLICT DO UPDATE / DO NOTHING
Изменение:  UPDATE ... SET ... WHERE
Удаление:   DELETE FROM ... WHERE

RETURNING — вернуть данные после INSERT/UPDATE/DELETE (PostgreSQL)

Подзапросы: IN, EXISTS, scalar, derived table
CASE WHEN   — условная логика в SELECT / ORDER BY

NULL:
  IS NULL / IS NOT NULL
  COALESCE(val, default)
```
