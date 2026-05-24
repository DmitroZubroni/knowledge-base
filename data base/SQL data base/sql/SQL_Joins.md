# SQL Joins — объединение таблиц

> [!abstract] Связи
> [[00_SQL]] | [[SQL_DML]] | [[SQL_Advanced]]

JOIN — механизм объединения строк из двух и более таблиц по условию связи.

---

## 🔹 Типы JOIN — визуально

```
Таблица A (users):          Таблица B (orders):
┌────┬────────┐             ┌────┬─────────┬────────┐
│ id │  name  │             │ id │ user_id │ amount │
├────┼────────┤             ├────┼─────────┼────────┤
│  1 │ Alice  │             │  1 │    1    │  500   │
│  2 │ Bob    │             │  2 │    1    │  300   │
│  3 │ Carol  │             │  3 │    2    │  800   │
│  4 │ Dave   │             │  4 │    5    │  100   │ ← нет такого user
└────┴────────┘             └────┴─────────┴────────┘

INNER JOIN  → только совпадения: Alice, Bob (id 1,2)
LEFT JOIN   → все из A + совпадения из B: Alice, Bob, Carol, Dave
RIGHT JOIN  → совпадения из A + все из B: Alice, Bob + строка с user_id=5
FULL JOIN   → все из обеих: все строки, NULL где нет пары
CROSS JOIN  → каждая строка A × каждая строка B (4×4 = 16 строк)
```

---

## 🔹 INNER JOIN

> [!note] INNER JOIN
> Возвращает только те строки, для которых есть совпадение **в обеих** таблицах.

```sql
SELECT
    u.id,
    u.username,
    o.id    AS order_id,
    o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- INNER можно опустить — JOIN по умолчанию INNER
SELECT u.username, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;
```

### ❌ Проблема

```sql
-- пользователи без заказов не попадут в результат
SELECT u.username, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.username;
-- Dave (0 заказов) пропадёт
```

### ✅ Решение

```sql
-- используй LEFT JOIN чтобы сохранить всех пользователей
SELECT u.username, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.username;
```

---

## 🔹 LEFT JOIN (LEFT OUTER JOIN)

> [!note] LEFT JOIN
> Возвращает **все строки из левой** таблицы. Для строк без совпадения в правой таблице — NULL.

```sql
-- все пользователи и их заказы (если есть)
SELECT
    u.username,
    o.id     AS order_id,
    o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- пользователи БЕЗ заказов
SELECT u.username
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;             -- фильтр строк где правая часть NULL
```

> [!tip] LEFT JOIN + WHERE IS NULL
> Это мощный паттерн для поиска "сирот" — записей без связанных данных.

---

## 🔹 RIGHT JOIN (RIGHT OUTER JOIN)

> [!note] RIGHT JOIN
> Возвращает **все строки из правой** таблицы. На практике используется редко — обычно меняют порядок таблиц и применяют LEFT JOIN.

```sql
-- эквивалентно: orders LEFT JOIN users
SELECT u.username, o.amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- заказы без пользователя (orphan records)
SELECT o.id, o.amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id
WHERE u.id IS NULL;
```

---

## 🔹 FULL OUTER JOIN

> [!note] FULL JOIN
> Возвращает все строки из **обеих** таблиц. NULL там, где нет совпадения с другой стороны.

```sql
SELECT
    u.username,
    o.id AS order_id,
    o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- пользователи без заказов + заказы без пользователей
SELECT
    u.username,
    o.id AS order_id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id
WHERE u.id IS NULL OR o.id IS NULL;
```

---

## 🔹 CROSS JOIN

> [!note] CROSS JOIN
> Декартово произведение — каждая строка левой таблицы соединяется с каждой строкой правой. Без условия ON.

```sql
-- все возможные комбинации размера и цвета
SELECT s.size, c.color
FROM sizes s
CROSS JOIN colors c;

-- генерация тестовых данных, матрицы расписаний
SELECT d.day, h.hour
FROM days d CROSS JOIN hours h;
```

> [!warning] Осторожно с размером
> 1000 строк × 1000 строк = 1 000 000 строк. Всегда контролируй объём.

---

## 🔹 SELF JOIN

> [!note] SELF JOIN
> Таблица соединяется сама с собой. Необходимы алиасы для различения копий.

```sql
-- иерархия: сотрудник и его менеджер (из той же таблицы employees)
SELECT
    e.name      AS employee,
    m.name      AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- найти пользователей из одного города
SELECT
    a.username AS user1,
    b.username AS user2,
    a.city
FROM users a
JOIN users b ON a.city = b.city AND a.id < b.id;  -- a.id < b.id убирает дубли
```

---

## 🔹 JOIN нескольких таблиц

```sql
SELECT
    u.username,
    o.id        AS order_id,
    p.name      AS product_name,
    oi.quantity,
    oi.price
FROM users u
JOIN orders o           ON u.id = o.user_id
JOIN order_items oi     ON o.id = oi.order_id
JOIN products p         ON oi.product_id = p.id
WHERE u.status = 'active'
ORDER BY o.created_at DESC;
```

> [!tip] Порядок JOIN
> БД сама выбирает оптимальный план выполнения. Но читабельность важна: пиши JOIN в логическом порядке зависимостей.

---

## 🔹 Условия JOIN — ON vs USING vs NATURAL

```sql
-- ON — явное условие (рекомендуется)
JOIN orders o ON u.id = o.user_id

-- USING — когда колонки называются одинаково в обеих таблицах
JOIN orders USING (user_id)         -- если в обеих таблицах есть user_id

-- NATURAL JOIN — автоматически по всем одинаковым колонкам (не рекомендуется)
NATURAL JOIN orders                 -- опасно: неявное поведение
```

> [!warning] NATURAL JOIN
> Избегай в продакшн-коде — при добавлении новой колонки с одинаковым именем поведение запроса изменится незаметно.

---

## 🔹 Сравнение JOIN

| JOIN | Строки из LEFT | Строки из RIGHT | NULL |
|------|---------------|----------------|------|
| `INNER JOIN` | только совпадения | только совпадения | нет |
| `LEFT JOIN` | все | только совпадения | справа |
| `RIGHT JOIN` | только совпадения | все | слева |
| `FULL JOIN` | все | все | с обеих сторон |
| `CROSS JOIN` | все × все | все × все | нет |

---

## 🔹 Производительность JOIN

> [!tip] Индексы для JOIN
> Всегда создавай индекс на колонке Foreign Key — она используется в условии JOIN каждый раз.

```sql
-- если часто делаешь:
JOIN orders o ON u.id = o.user_id

-- убедись что есть индекс:
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

> [!note] Алгоритмы JOIN (PostgreSQL выбирает автоматически)
> ```
> Nested Loop  — маленькие таблицы, есть индекс
> Hash Join    — большие таблицы, нет индекса
> Merge Join   — обе таблицы отсортированы
> ```
> Подробнее — [[PG_Performance]]

---

## 🔹 Итог

```
INNER JOIN  — только совпадения в обеих таблицах
LEFT JOIN   — все из левой + совпадения из правой (NULL справа)
RIGHT JOIN  — совпадения из левой + все из правой (NULL слева)
FULL JOIN   — все из обеих таблиц (NULL с обеих сторон)
CROSS JOIN  — декартово произведение (нет условия)
SELF JOIN   — таблица с собой (алиасы обязательны)

Паттерны:
  LEFT JOIN WHERE right.id IS NULL    → записи без пары
  FULL JOIN WHERE ... IS NULL         → "сироты" с обеих сторон

Производительность:
  CREATE INDEX на Foreign Key колонках
  EXPLAIN ANALYZE для анализа плана
```
