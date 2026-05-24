# PG Performance — Оптимизация и EXPLAIN

> [!abstract] Связи
> [[00_PostgreSQL]] | [[PG_Indexes]] | [[PG_Transactions]] | [[PG_Architecture]]

---

## 🔹 EXPLAIN — план запроса

> [!note] EXPLAIN
> Показывает план выполнения запроса — как планировщик собирается его выполнить и с какой оценкой стоимости.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@mail.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@mail.com';
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM users WHERE email = 'test@mail.com';
```

| Опция | Описание |
|-------|----------|
| `EXPLAIN` | только план и оценка стоимости |
| `EXPLAIN ANALYZE` | выполняет запрос, показывает реальное время |
| `BUFFERS` | показывает hit/miss кэша shared_buffers |
| `FORMAT JSON` | вывод в JSON (удобно для pgMustard, explain.dalibo.com) |

> [!warning] EXPLAIN ANALYZE выполняет запрос
> Для `DELETE`/`UPDATE` — оберни в транзакцию и откати: `BEGIN; EXPLAIN ANALYZE DELETE ...; ROLLBACK;`

---

## 🔹 Чтение плана запроса

```
Seq Scan on users  (cost=0.00..458.00 rows=10000 width=64)
                    │            │        │          │
                    │            │        │          └── средний размер строки (байт)
                    │            │        └── оценка кол-ва строк
                    │            └── total cost (до последней строки)
                    └── startup cost (до первой строки)

EXPLAIN ANALYZE добавляет:
  (actual time=0.043..1.234 rows=9800 loops=1)
                │       │    │           └── сколько раз выполнялся узел
                │       │    └── реальное кол-во строк
                │       └── реальное total time (мс)
                └── реальное startup time (мс)
```

### Ключевые узлы плана

| Узел | Описание | Когда |
|------|----------|-------|
| `Seq Scan` | полное сканирование таблицы | нет индекса или он нецелесообразен |
| `Index Scan` | сканирование по индексу + fetch из таблицы | есть индекс, высокая селективность |
| `Index Only Scan` | только индекс, таблица не нужна | покрывающий индекс |
| `Bitmap Heap Scan` | несколько условий, bitmap → таблица | несколько индексов или средняя селективность |
| `Nested Loop` | для каждой строки внешней — поиск во внутренней | маленькие таблицы, индекс на внутренней |
| `Hash Join` | строит хэш одной таблицы, проверяет другую | большие таблицы без индекса |
| `Merge Join` | объединяет отсортированные наборы | обе таблицы отсортированы |
| `Sort` | сортировка | `ORDER BY`, `MERGE JOIN` |
| `Hash` | построение хэш-таблицы | `HASH JOIN`, `GROUP BY` |
| `Aggregate` | агрегация | `COUNT`, `SUM`, `AVG` |
| `Limit` | ограничение строк | `LIMIT` |

### Пример чтения плана

```
Hash Join  (cost=458.00..1124.50 rows=5000 width=128)
           (actual time=8.123..24.567 rows=4800 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (cost=0.00..312.00 rows=12000 width=64)
                             (actual time=0.012..5.432 rows=12000 loops=1)
  ->  Hash  (cost=208.00..208.00 rows=8000 width=64)
             (actual time=3.456..3.456 rows=8000 loops=1)
        Buckets: 8192  Batches: 1  Memory Usage: 640kB
        ->  Seq Scan on users u  (cost=0.00..208.00 rows=8000 width=64)
                                  (actual time=0.010..2.345 rows=8000 loops=1)

Planning Time: 0.234 ms
Execution Time: 25.123 ms
```

---

## 🔹 Красные флаги в плане

```
❌ Seq Scan на большой таблице     → нужен индекс
❌ rows estimate далеко от actual  → устаревшая статистика → ANALYZE
❌ Sort (external)                 → work_mem мал, идёт сортировка на диск
❌ Hash Batches > 1                → work_mem мал, хэш не влез в память
❌ Nested Loop с большими таблицами → нет индекса или неверный план
❌ очень высокий cost              → посмотри узел с наибольшим вкладом
```

### Проверка разрыва оценки и реальности

```sql
-- если rows estimate сильно отличается от actual rows:
ANALYZE users;                  -- обновить статистику одной таблицы
ANALYZE;                        -- обновить всё

-- увеличить точность статистики для колонки
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;  -- default 100
ANALYZE orders;
```

---

## 🔹 Настройка планировщика

```ini
# postgresql.conf

# подсказка планировщику: сколько RAM OS отдаёт под page cache
effective_cache_size = 4GB       # обычно 50-75% RAM

# стоимость случайного чтения с диска (снижай для SSD)
random_page_cost = 1.1           # default 4.0, для SSD → 1.1-1.5

# стоимость последовательного чтения
seq_page_cost = 1.0              # default, обычно не трогают

# статистика для параллельных запросов
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000.0
max_parallel_workers_per_gather = 2
```

> [!tip] random_page_cost для SSD
> На SSD разница между последовательным и случайным чтением минимальна. Снижение `random_page_cost` до 1.1 заставит планировщик чаще выбирать Index Scan вместо Seq Scan.

---

## 🔹 Медленные запросы — поиск

### pg_stat_statements

```sql
-- включить расширение (один раз)
CREATE EXTENSION pg_stat_statements;

-- добавить в postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'

-- топ-10 самых медленных запросов
SELECT
    query,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2)  AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- сбросить статистику
SELECT pg_stat_statements_reset();
```

### auto_explain

```ini
# postgresql.conf — логировать план медленных запросов автоматически
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000    # логировать запросы > 1 секунды
auto_explain.log_analyze = true
auto_explain.log_buffers = true
```

### log_min_duration_statement

```ini
# логировать медленные запросы в pg_log
log_min_duration_statement = 500        # ms, -1 = отключить
```

---

## 🔹 Типичные проблемы и решения

### N+1 запрос

```sql
-- ❌ N+1: получить 100 пользователей + 100 запросов за заказами
SELECT * FROM users LIMIT 100;
-- потом для каждого:
SELECT * FROM orders WHERE user_id = ?;

-- ✅ Один JOIN:
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LIMIT 100;
```

### Медленный COUNT(*)

```sql
-- ❌ COUNT(*) на большой таблице = Seq Scan
SELECT COUNT(*) FROM orders;

-- ✅ Приблизительный count из статистики (мгновенно)
SELECT reltuples::BIGINT AS approx_count
FROM pg_class WHERE relname = 'orders';

-- ✅ Для точного count по условию — частичный индекс
CREATE INDEX idx_pending_orders ON orders(id) WHERE status = 'pending';
SELECT COUNT(*) FROM orders WHERE status = 'pending';  -- Index Only Scan
```

### Медленная пагинация с OFFSET

```sql
-- ❌ OFFSET 100000 — читает и пропускает 100000 строк
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 100000;

-- ✅ Keyset pagination — по последнему ID
SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT 10;
```

### Неэффективный LIKE

```sql
-- ❌ суффиксный LIKE — не использует B-tree
WHERE username LIKE '%alex%'

-- ✅ pg_trgm — триграммный индекс для произвольного LIKE
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_username_trgm ON users USING GIN (username gin_trgm_ops);
-- теперь работает: LIKE '%alex%', ILIKE '%ALEX%'
```

### Медленный ORDER BY + LIMIT

```sql
-- ❌ сортировка всей таблицы
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- ✅ индекс для сортировки
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
-- → Index Scan Backward, без Sort узла
```

---

## 🔹 VACUUM и ANALYZE

```sql
-- ручной VACUUM (убирает dead tuples)
VACUUM orders;

-- VACUUM + обновление статистики планировщика
VACUUM ANALYZE orders;

-- полная перестройка (блокирует таблицу!)
VACUUM FULL orders;

-- только статистика
ANALYZE orders;

-- посмотреть состояние autovacuum
SELECT
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Настройка autovacuum

```ini
# postgresql.conf
autovacuum = on                            # включён по умолчанию
autovacuum_vacuum_threshold = 50           # минимум dead tuples для запуска
autovacuum_vacuum_scale_factor = 0.2       # + 20% от размера таблицы
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1

# для высоконагруженных таблиц — переопределить на уровне таблицы
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01, -- срабатывать при 1% dead tuples
    autovacuum_analyze_scale_factor = 0.01
);
```

---

## 🔹 Мониторинг производительности

```sql
-- активные запросы и их длительность
SELECT
    pid,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- долгие запросы (> 5 минут)
SELECT pid, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > INTERVAL '5 minutes';

-- размеры таблиц
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- hit rate кэша (должен быть > 99%)
SELECT
    sum(blks_hit) * 100.0 / NULLIF(sum(blks_hit) + sum(blks_read), 0) AS cache_hit_ratio
FROM pg_stat_database;

-- статистика использования индексов
SELECT
    relname AS table,
    indexrelname AS index,
    idx_scan AS scans,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

---

## 🔹 Connection Pooling

> [!note] Проблема соединений
> Каждое соединение к PostgreSQL = отдельный OS процесс (~5-10MB RAM). При 500+ соединениях это дорого. Пулер повторно использует соединения.

```
Приложение → PgBouncer → PostgreSQL

PgBouncer режимы:
  session     — одно PG соединение на сессию клиента (как без пулера)
  transaction — одно PG соединение на транзакцию (рекомендуется)
  statement   — одно PG соединение на запрос (ограничения)
```

```ini
# pgbouncer.ini (упрощённо)
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

---

## 🔹 Итог

```
EXPLAIN:
  EXPLAIN            — план без выполнения
  EXPLAIN ANALYZE    — план + реальное время
  EXPLAIN (ANALYZE, BUFFERS) — + кэш

Красные флаги:
  Seq Scan на большой таблице → индекс
  estimate ≠ actual → ANALYZE
  Sort (external) → увеличь work_mem
  Hash Batches > 1 → увеличь work_mem

Медленные запросы:
  pg_stat_statements — топ по total_time
  log_min_duration_statement — логировать
  auto_explain — план в лог автоматически

Типичные проблемы:
  N+1      → JOIN
  OFFSET   → keyset pagination (WHERE id > ?)
  LIKE %x% → pg_trgm + GIN индекс
  COUNT(*) → pg_class.reltuples (приблизительно)

VACUUM:
  autovacuum — всегда включён
  VACUUM ANALYZE — после массовых изменений
  VACUUM FULL — только в окно обслуживания

Мониторинг:
  pg_stat_activity    — активные запросы
  pg_stat_user_tables — bloat, vacuum статус
  pg_stat_user_indexes — использование индексов
  cache hit ratio > 99% — через pg_stat_database

Соединения:
  PgBouncer transaction mode — рекомендуется
  default_pool_size = 20-50 реальных соединений
```
