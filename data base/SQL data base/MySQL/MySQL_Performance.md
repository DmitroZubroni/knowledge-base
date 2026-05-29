# MySQL Performance — Оптимизация и EXPLAIN

> **Теги:** #mysql #database #performance #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[MySQL]]

---

## 🔹 EXPLAIN — план запроса

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@mail.com';
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'test@mail.com';

-- MySQL 8.0: реальное выполнение с метриками
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@mail.com';
```

> [!warning] EXPLAIN ANALYZE выполняет запрос
> Для `DELETE`/`UPDATE` — оберни в транзакцию: `START TRANSACTION; EXPLAIN ANALYZE DELETE ...; ROLLBACK;`

---

## 🔹 Чтение вывода EXPLAIN

```sql
EXPLAIN SELECT u.username, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active' AND o.amount > 100;
```

```
+----+-------------+-------+--------+---------------+------------------+---------+------------------+------+-------------+
| id | select_type | table | type   | possible_keys | key              | key_len | ref              | rows | Extra       |
+----+-------------+-------+--------+---------------+------------------+---------+------------------+------+-------------+
|  1 | SIMPLE      | u     | ref    | idx_status    | idx_status       | 43      | const            |  500 | Using where |
|  1 | SIMPLE      | o     | ref    | idx_user_id   | idx_user_id      | 4       | myapp.u.id       |    3 | Using where |
+----+-------------+-------+--------+---------------+------------------+---------+------------------+------+-------------+
```

### Колонки EXPLAIN

| Колонка | Описание |
|---------|----------|
| `id` | порядок выполнения (больше = раньше) |
| `select_type` | тип SELECT (SIMPLE, PRIMARY, SUBQUERY, UNION...) |
| `table` | таблица |
| `type` | **тип доступа** — самое важное поле |
| `possible_keys` | кандидаты на индекс |
| `key` | выбранный индекс (NULL = нет индекса) |
| `key_len` | длина использованного префикса индекса |
| `ref` | с чем сравнивается индекс |
| `rows` | оценка количества строк для чтения |
| `filtered` | % строк после WHERE фильтра |
| `Extra` | дополнительная информация |

---

## 🔹 Тип доступа (type) — от лучшего к худшему

```
system   → таблица с одной строкой
const    → поиск по PRIMARY KEY или UNIQUE с константой (WHERE id = 5)
eq_ref   → JOIN по PRIMARY KEY / UNIQUE (одна строка на каждую из внешней)
ref      → поиск по неуникальному индексу (WHERE status = 'active')
range    → диапазон по индексу (WHERE age BETWEEN 18 AND 65)
index    → полный scan индекса (медленнее range, быстрее ALL)
ALL      → полный scan таблицы ← ПЛОХО для больших таблиц
```

> [!tip] Целевые типы
> `const`, `eq_ref` — отлично. `ref`, `range` — хорошо. `index` — приемлемо. `ALL` на большой таблице — нужен индекс.

### Поле Extra — важные значения

| Extra | Значение |
|-------|----------|
| `Using index` | ✅ покрывающий индекс — таблица не читается |
| `Using where` | фильтрация после чтения из движка |
| `Using index condition` | ✅ Index Condition Pushdown (ICP) |
| `Using filesort` | ⚠️ сортировка в памяти/на диске — нет индекса для ORDER BY |
| `Using temporary` | ⚠️ временная таблица — сложный GROUP BY или ORDER BY |
| `Using join buffer` | ⚠️ JOIN без индекса на внутренней таблице |
| `Impossible WHERE` | условие всегда ложно — 0 строк |

---

## 🔹 Красные флаги в EXPLAIN

```sql
-- ❌ type = ALL на большой таблице → нет подходящего индекса
type: ALL, rows: 1000000

-- ❌ key = NULL → индекс не используется
possible_keys: idx_email, key: NULL

-- ❌ Using filesort → ORDER BY не покрыт индексом
Extra: Using filesort

-- ❌ Using temporary → GROUP BY требует временной таблицы
Extra: Using temporary; Using filesort

-- ❌ rows сильно завышен → устаревшая статистика
ANALYZE TABLE users;     -- обновить статистику

-- ❌ key_len меньше ожидаемого → составной индекс используется частично
-- INDEX(user_id, status), key_len = 4 → используется только user_id (4 байта)
-- key_len = 4 + 43 = 47 → используются обе колонки
```

---

## 🔹 Поиск медленных запросов

### Slow Query Log

```ini
# my.cnf
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1                   # секунды
log_queries_not_using_indexes = ON    # запросы без индекса
log_slow_admin_statements = ON        # ALTER, OPTIMIZE и т.д.
min_examined_row_limit = 1000         # только если прочитано > 1000 строк
```

```sql
-- включить/выключить без рестарта
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;     -- 500ms

-- анализ slow log
-- mysqldumpslow — встроенный инструмент
-- pt-query-digest — Percona Toolkit (рекомендуется)
```

```bash
# анализ через mysqldumpslow
mysqldumpslow -t 10 -s t /var/log/mysql/slow.log   # топ-10 по суммарному времени
mysqldumpslow -t 10 -s c /var/log/mysql/slow.log   # топ-10 по количеству вызовов

# pt-query-digest (Percona Toolkit)
pt-query-digest /var/log/mysql/slow.log | head -100
```

### Performance Schema

```sql
-- включить (по умолчанию ON в MySQL 5.7+)
SHOW VARIABLES LIKE 'performance_schema';

-- топ запросов по суммарному времени
SELECT
    DIGEST_TEXT                              AS query,
    COUNT_STAR                               AS exec_count,
    ROUND(SUM_TIMER_WAIT / 1e12, 3)         AS total_sec,
    ROUND(AVG_TIMER_WAIT / 1e12, 3)         AS avg_sec,
    ROUND(MAX_TIMER_WAIT / 1e12, 3)         AS max_sec,
    SUM_ROWS_EXAMINED                        AS rows_examined,
    SUM_ROWS_SENT                            AS rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- сбросить статистику
TRUNCATE performance_schema.events_statements_summary_by_digest;

-- активные запросы дольше 5 секунд
SELECT id, user, host, db, time, state, info
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep' AND TIME > 5
ORDER BY TIME DESC;
```

---

## 🔹 Типичные проблемы и решения

### N+1 запрос

```sql
-- ❌ N+1: 1 запрос за пользователями + N запросов за заказами
SELECT * FROM users WHERE status = 'active' LIMIT 100;
-- потом для каждого user:
SELECT * FROM orders WHERE user_id = ?;

-- ✅ JOIN за один запрос
SELECT u.id, u.username, o.id AS order_id, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
LIMIT 100;
```

### Медленный COUNT(*)

```sql
-- ❌ MyISAM хранит count — мгновенно, но InnoDB — Seq Scan
SELECT COUNT(*) FROM orders;                  -- Full Scan в InnoDB

-- ✅ приблизительный count из статистики
SELECT table_rows FROM information_schema.TABLES
WHERE table_schema = 'myapp' AND table_name = 'orders';

-- ✅ покрывающий индекс для COUNT по условию
CREATE INDEX idx_orders_status ON orders(status);
SELECT COUNT(*) FROM orders WHERE status = 'pending';   -- Using index
```

### Медленная пагинация с OFFSET

```sql
-- ❌ OFFSET 100000 — читает и пропускает 100000 строк
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 100000;

-- ✅ keyset pagination
SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT 10;

-- ✅ если нужен OFFSET — используй покрывающий индекс
SELECT o.* FROM orders o
JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 10 OFFSET 100000
) AS ids ON o.id = ids.id;
-- сначала быстро выбираем ID через индекс, потом join с таблицей
```

### Функция над индексированной колонкой

```sql
-- ❌ индекс не используется
WHERE YEAR(created_at) = 2024
WHERE DATE(created_at) = '2024-01-15'
WHERE LOWER(email) = 'test@mail.com'

-- ✅ переписать без функции
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16'

-- ✅ или виртуальная колонка + индекс
ALTER TABLE users ADD COLUMN email_lower VARCHAR(100)
    GENERATED ALWAYS AS (LOWER(email)) STORED;
CREATE INDEX idx_users_email_lower ON users(email_lower);
WHERE email_lower = 'test@mail.com';
```

### SELECT *

```sql
-- ❌ SELECT * — тянет все колонки (нет покрывающего индекса)
SELECT * FROM users WHERE status = 'active';

-- ✅ только нужные колонки — возможен covering index
SELECT id, username, email FROM users WHERE status = 'active';
```

### OR между разными колонками

```sql
-- ❌ OR — оптимизатор часто выбирает Full Scan
SELECT * FROM users WHERE email = 'a@b.com' OR username = 'alice';

-- ✅ UNION — каждая часть использует свой индекс
SELECT * FROM users WHERE email = 'a@b.com'
UNION
SELECT * FROM users WHERE username = 'alice';
```

---

## 🔹 Оптимизация таблиц

```sql
-- обновить статистику планировщика
ANALYZE TABLE users;
ANALYZE TABLE users, orders;       -- несколько таблиц

-- дефрагментация таблицы (аналог VACUUM FULL в PostgreSQL)
OPTIMIZE TABLE users;              -- пересоздаёт таблицу, блокирует!

-- для больших таблиц без блокировки:
-- pt-online-schema-change (Percona Toolkit)
-- gh-ost (GitHub)

-- проверить фрагментацию
SELECT
    table_name,
    ROUND(data_length / 1024 / 1024, 2)  AS data_mb,
    ROUND(data_free  / 1024 / 1024, 2)  AS free_mb,
    ROUND(data_free * 100.0 / (data_length + data_free), 1) AS frag_pct
FROM information_schema.TABLES
WHERE table_schema = 'myapp'
  AND data_free > 0
ORDER BY data_free DESC;
```

---

## 🔹 Кэширование запросов (Query Cache — удалён в 8.0)

> [!note] Query Cache удалён
> Query Cache был удалён в MySQL 8.0 из-за проблем с производительностью при высокой конкурентности (глобальный мьютекс). Для кэширования используй **Redis** или **Memcached** на уровне приложения.

---

## 🔹 Мониторинг производительности

```sql
-- размеры таблиц и индексов
SELECT
    table_name,
    ROUND((data_length)                    / 1024 / 1024, 2) AS data_mb,
    ROUND((index_length)                   / 1024 / 1024, 2) AS index_mb,
    ROUND((data_length + index_length)     / 1024 / 1024, 2) AS total_mb,
    table_rows
FROM information_schema.TABLES
WHERE table_schema = 'myapp'
ORDER BY total_mb DESC;

-- статус Buffer Pool
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
-- Innodb_buffer_pool_reads      — чтений с диска
-- Innodb_buffer_pool_read_requests — всего запросов
-- hit rate = 1 - (reads / read_requests) > 99%

-- статус InnoDB — полная информация
SHOW ENGINE INNODB STATUS\G

-- глобальные счётчики
SHOW GLOBAL STATUS LIKE 'Com_select';      -- количество SELECT
SHOW GLOBAL STATUS LIKE 'Com_insert';      -- количество INSERT
SHOW GLOBAL STATUS LIKE 'Slow_queries';    -- медленных запросов всего
SHOW GLOBAL STATUS LIKE 'Created_tmp%';    -- временные таблицы

-- текущие соединения и запросы
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;                     -- полный текст запроса
```

---

## 🔹 Ключевые настройки производительности

```ini
# my.cnf — производительность

# ── Память ──────────────────────────────────────────────
innodb_buffer_pool_size     = 4G    # 50-80% RAM
innodb_log_file_size        = 512M  # больше = реже checkpoint, но дольше восстановление
sort_buffer_size            = 4M    # на соединение — для ORDER BY
join_buffer_size            = 4M    # на JOIN без индекса
read_rnd_buffer_size        = 4M    # для MyISAM random read
tmp_table_size              = 64M   # в памяти для GROUP BY / tmp tables
max_heap_table_size         = 64M   # лимит для MEMORY таблиц

# ── InnoDB I/O ───────────────────────────────────────────
innodb_io_capacity          = 2000  # для SSD (200 для HDD)
innodb_io_capacity_max      = 4000
innodb_read_io_threads      = 4
innodb_write_io_threads     = 4
innodb_flush_method         = O_DIRECT    # для Linux SSD

# ── Оптимизатор ──────────────────────────────────────────
optimizer_switch = 'index_merge=on,mrr=on,batched_key_access=on'
```

---

## 🔹 Итог

```
EXPLAIN поля:
  type:  const > eq_ref > ref > range > index > ALL
  key:   NULL → нет индекса → проблема
  Extra: Using filesort / Using temporary → нужен индекс
         Using index → покрывающий индекс → отлично

Поиск медленных запросов:
  slow_query_log + long_query_time
  performance_schema.events_statements_summary_by_digest
  pt-query-digest (Percona Toolkit)

Типичные проблемы:
  N+1          → JOIN
  OFFSET       → keyset pagination (WHERE id > ?)
  Функция(col) → виртуальная колонка + индекс
  OR           → UNION
  COUNT(*)     → информация из information_schema (приблизительно)
  SELECT *     → только нужные колонки + covering index

Оптимизация таблиц:
  ANALYZE TABLE   — обновить статистику
  OPTIMIZE TABLE  — дефрагментация (блокирует!)
  pt-osc / gh-ost — без блокировки для больших таблиц

Мониторинг:
  SHOW ENGINE INNODB STATUS     — состояние InnoDB
  SHOW PROCESSLIST              — активные запросы
  information_schema.TABLES     — размеры
  Buffer Pool hit rate > 99%
```
