# MySQL Indexes — Индексы MySQL

> **Теги:** #mysql #database #indexes #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[MySQL]]

---

## 🔹 Что такое индекс

> [!note] Index
> Вспомогательная структура данных, ускоряющая поиск строк за счёт хранения отсортированных указателей. Ускоряет SELECT, замедляет INSERT/UPDATE/DELETE.

```
Без индекса (Full Table Scan):
  SELECT * FROM users WHERE email = 'a@b.com'
  → читаем все 1 000 000 строк → O(n)

С индексом (B-tree):
  → спускаемся по дереву → O(log n)
  → ~20 шагов вместо 1 000 000 чтений
```

---

## 🔹 Типы индексов в MySQL

| Тип | Движок | Операторы | Применение |
|-----|--------|-----------|------------|
| **B-tree** | InnoDB, MyISAM | `=`, `<`, `>`, `BETWEEN`, `LIKE 'x%'` | универсальный |
| **Hash** | Memory | только `=` | временные таблицы |
| **Full-text** | InnoDB, MyISAM | `MATCH ... AGAINST` | полнотекстовый поиск |
| **Spatial (R-tree)** | InnoDB, MyISAM | `ST_*` функции | геометрия |

> [!note] InnoDB и Hash
> InnoDB не поддерживает Hash индексы на диске — только в памяти (Adaptive Hash Index, управляется автоматически). Явно создать Hash индекс в InnoDB нельзя.

---

## 🔹 B-tree индекс

### Создание

```sql
-- простой индекс
CREATE INDEX idx_users_email ON users(email);

-- уникальный
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- составной
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- убрать индекс
DROP INDEX idx_users_email ON users;
ALTER TABLE users DROP INDEX idx_users_email;

-- посмотреть индексы
SHOW INDEX FROM users;
SHOW CREATE TABLE users\G
```

### Когда B-tree работает

```sql
WHERE email = 'test@mail.com'          -- равенство
WHERE age > 18                         -- сравнение
WHERE age BETWEEN 18 AND 65            -- диапазон
WHERE username LIKE 'alex%'            -- префикс (не %alex!)
ORDER BY created_at DESC LIMIT 10      -- сортировка + лимит
GROUP BY department_id                 -- группировка
JOIN orders ON users.id = orders.user_id -- условие JOIN
```

### Когда B-tree НЕ работает

```sql
-- ❌ функция над индексированной колонкой
WHERE YEAR(created_at) = 2024
WHERE LOWER(email) = 'test@mail.com'

-- ❌ суффиксный LIKE
WHERE username LIKE '%alex'
WHERE username LIKE '%alex%'

-- ❌ неравенство на первой колонке составного индекса
-- INDEX ON (user_id, status)
WHERE status = 'paid'                  -- без user_id

-- ❌ OR между разными колонками (без отдельных индексов)
WHERE email = 'a@b.com' OR username = 'alex'

-- ❌ неявное приведение типов
WHERE id = '123'                       -- id INT, '123' строка → конвертация

-- ❌ IS NULL (в InnoDB поддерживается, но часто игнорируется планировщиком)
```

---

## 🔹 Clustered Index (кластерный индекс)

> [!note] Clustered Index
> В InnoDB таблица физически организована в виде B-tree по первичному ключу. Данные хранятся прямо в листовых узлах индекса — это и есть кластерный индекс. Он всегда один на таблицу.

```
Clustered Index (PRIMARY KEY):
  Листовые узлы B-tree = сами данные строк
  Физический порядок строк = порядок PK

Secondary Index (все остальные):
  Листовые узлы = значение индексной колонки + значение PK
  Для получения данных → дополнительный lookup по кластерному индексу
```

```sql
-- InnoDB выбирает кластерный индекс по правилам:
-- 1. Явный PRIMARY KEY
-- 2. Первый UNIQUE NOT NULL индекс
-- 3. Внутренний скрытый 6-байтный row ID (если нет PK и UNIQUE NOT NULL)

-- ✅ всегда явно задавай PRIMARY KEY
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    PRIMARY KEY (id)
);
```

> [!tip] UUID как Primary Key
> UUID v4 случаен — вставки в случайные позиции B-tree вызывают фрагментацию и page splits. Для UUID предпочти:
> - UUID v7 (монотонный, MySQL 8.0+)
> - `BINARY(16)` + `UUID_TO_BIN(uuid, 1)` (swap flag для монотонности)
> - Оставь `BIGINT AUTO_INCREMENT` как PK, UUID — как отдельное поле с UNIQUE

---

## 🔹 Составной индекс

```sql
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at);
```

### Правило левого префикса

```sql
-- индекс (user_id, status, created_at) РАБОТАЕТ для:
WHERE user_id = 5                                       -- ✅ левый префикс
WHERE user_id = 5 AND status = 'paid'                   -- ✅
WHERE user_id = 5 AND status = 'paid' AND created_at > '2024-01-01' -- ✅

-- индекс НЕ РАБОТАЕТ для:
WHERE status = 'paid'                                   -- ❌ без user_id
WHERE created_at > '2024-01-01'                         -- ❌ без user_id и status
WHERE user_id = 5 AND created_at > '2024-01-01'         -- ⚠️ частично (только user_id)
```

### Порядок колонок

```sql
-- запрос: WHERE user_id = 5 AND status = 'paid' AND amount > 100
-- правило: сначала равенство (=), потом диапазон (>, <, BETWEEN)

-- ✅ правильный порядок
CREATE INDEX idx ON orders(user_id, status, amount);

-- ⚠️ неоптимальный — amount с диапазоном в середине
CREATE INDEX idx ON orders(user_id, amount, status);
-- индекс будет использоваться только по user_id + amount, status игнорируется
```

---

## 🔹 Покрывающий индекс (Covering Index)

> [!note] Covering Index
> Индекс, который содержит все колонки, нужные для запроса — данные берутся из индекса без обращения к таблице. В EXPLAIN виден как `Using index`.

```sql
-- запрос: только user_id, status, amount
SELECT user_id, status, amount FROM orders WHERE user_id = 5;

-- покрывающий индекс — все нужные колонки в индексе
CREATE INDEX idx_orders_cover ON orders(user_id, status, amount);

-- EXPLAIN покажет: Using index (без обращения к таблице)
EXPLAIN SELECT user_id, status, amount FROM orders WHERE user_id = 5;
```

```
Without covering index:
  Index Scan → получить PK → lookup в clustered index → вернуть строку

With covering index:
  Index Scan → вернуть данные прямо из индекса (быстрее)
```

---

## 🔹 Prefix Index

> [!note] Prefix Index
> Индекс по первым N символам строки. Нужен для `TEXT` и `BLOB` колонок (нельзя индексировать целиком), а также для уменьшения размера индекса для длинных `VARCHAR`.

```sql
-- индекс по первым 20 символам
CREATE INDEX idx_users_email ON users(email(20));

-- для TEXT обязателен
CREATE INDEX idx_articles_content ON articles(content(100));

-- уникальный prefix-индекс
CREATE UNIQUE INDEX idx_url ON pages(url(255));
```

> [!warning] Ограничение prefix-индекса
> Prefix-индекс не может быть покрывающим и не поддерживает `ORDER BY` по этой колонке. Определи оптимальную длину через анализ кардинальности:

```sql
-- найти оптимальную длину prefix
SELECT
    COUNT(DISTINCT LEFT(email, 5))  AS prefix_5,
    COUNT(DISTINCT LEFT(email, 10)) AS prefix_10,
    COUNT(DISTINCT LEFT(email, 20)) AS prefix_20,
    COUNT(DISTINCT email)           AS full
FROM users;
-- когда prefix_N ≈ full — этой длины достаточно
```

---

## 🔹 Full-text индекс

> [!note] Full-text Index
> Специальный инвертированный индекс для полнотекстового поиска. Поддерживается в InnoDB (5.6+) и MyISAM.

```sql
-- создать full-text индекс
CREATE FULLTEXT INDEX idx_articles_fts ON articles(title, content);

-- или при создании таблицы
CREATE TABLE articles (
    id      INT AUTO_INCREMENT PRIMARY KEY,
    title   VARCHAR(255),
    content TEXT,
    FULLTEXT INDEX (title, content)
) ENGINE = InnoDB;

-- поиск (Natural Language Mode — по умолчанию)
SELECT *, MATCH(title, content) AGAINST ('быстрая лиса') AS relevance
FROM articles
WHERE MATCH(title, content) AGAINST ('быстрая лиса')
ORDER BY relevance DESC;

-- Boolean Mode — операторы поиска
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST (
    '+mysql -oracle "быстрый поиск"'
    IN BOOLEAN MODE
);
-- + обязательное слово
-- - исключить слово
-- "фраза" точная фраза
-- * префикс: mysql*

-- Query Expansion Mode — расширение запроса синонимами
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST ('mysql' WITH QUERY EXPANSION);
```

> [!note] Минимальная длина слова
> По умолчанию MySQL не индексирует слова короче 4 символов (`ft_min_word_len = 4` для MyISAM, `innodb_ft_min_token_size = 3` для InnoDB). Для поиска по коротким словам измени параметр и перестрой индекс.

> [!tip] Full-text vs LIKE
> `MATCH ... AGAINST` значительно быстрее `LIKE '%слово%'` на больших таблицах, плюс поддерживает ранжирование по релевантности. Для сложного полнотекстового поиска рассмотри Elasticsearch.

---

## 🔹 Invisible и Descending индексы (MySQL 8.0)

```sql
-- Invisible Index — индекс существует, но оптимизатор его игнорирует
-- Полезно для безопасного тестирования: "что будет если удалить этот индекс?"
CREATE INDEX idx_users_age ON users(age);
ALTER TABLE users ALTER INDEX idx_users_age INVISIBLE;

-- форсировать использование invisible индекса в сессии
SET SESSION optimizer_switch = 'use_invisible_indexes=on';

-- вернуть видимость
ALTER TABLE users ALTER INDEX idx_users_age VISIBLE;

-- Descending Index (8.0) — индекс в убывающем порядке
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);

-- особенно полезен для составных индексов с разными направлениями
CREATE INDEX idx_orders_mixed ON orders(user_id ASC, created_at DESC);
-- запрос: ORDER BY user_id ASC, created_at DESC — использует индекс без сортировки
```

---

## 🔹 Мониторинг индексов

```sql
-- план выполнения
EXPLAIN SELECT * FROM orders WHERE user_id = 5 AND status = 'paid';

-- расширенный план (MySQL 8.0)
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 5;
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = 5;

-- использование индексов (через performance_schema)
SELECT
    object_schema,
    object_name,
    index_name,
    count_read,
    count_write
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'myapp'
ORDER BY count_read DESC;

-- неиспользуемые индексы
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = 'myapp'
ORDER BY object_name;

-- размер индексов
SELECT
    table_name,
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'myapp'
  AND stat_name = 'size'
ORDER BY size_mb DESC;
```

---

## 🔹 Рекомендации по индексам

```sql
-- ✅ индексировать Foreign Key колонки
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- ✅ индексировать колонки в WHERE с высокой кардинальностью
CREATE INDEX idx_users_email ON users(email);

-- ✅ покрывающий индекс для частых SELECT
CREATE INDEX idx_orders_cover ON orders(user_id, status, amount, created_at);

-- ✅ составной индекс вместо нескольких одноколоночных
-- вместо: INDEX(user_id) + INDEX(status)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- ❌ не индексировать колонки с низкой кардинальностью
-- is_active с 2 значениями (0/1) — Full Scan часто выгоднее

-- ❌ не создавать дублирующие индексы
-- INDEX(a) + INDEX(a, b) — первый лишний, второй покрывает его
```

---

## 🔹 Итог

```
Типы:
  B-tree     — универсальный (=, <, >, BETWEEN, LIKE 'x%')
  Full-text  — MATCH ... AGAINST, ранжирование
  Spatial    — геометрия ST_* функции
  Hash       — только Memory engine

Clustered Index:
  InnoDB хранит данные в порядке PK
  Secondary index → lookup по PK → данные
  Всегда задавай явный PRIMARY KEY

Составной индекс:
  Правило левого префикса — запрос должен начинать с левой колонки
  Порядок: сначала = условия, потом диапазонные

Покрывающий индекс:
  Все нужные SELECT колонки в индексе
  EXPLAIN → Using index (без Table lookup)

Prefix Index:
  Для TEXT/BLOB обязателен
  Найди минимальную длину через COUNT(DISTINCT LEFT(col, n))

MySQL 8.0:
  Invisible Index — тест "что если удалить"
  Descending Index — ORDER BY col DESC без сортировки

Мониторинг:
  EXPLAIN / EXPLAIN ANALYZE
  performance_schema.table_io_waits_summary_by_index_usage
  Удаляй индексы с count_star = 0
```
