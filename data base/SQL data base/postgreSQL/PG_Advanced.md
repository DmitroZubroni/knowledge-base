# PG Advanced — Партиционирование, Репликация, Расширения

> **Теги:** #postgresql #database #advanced #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[PostgreSQL]]

---

## 🔹 Партиционирование (Partitioning)

> [!note] Партиционирование
> Разбиение одной большой таблицы на физически раздельные части (партиции) с сохранением единого логического интерфейса. Улучшает производительность запросов и упрощает управление данными.

### Типы партиционирования

```sql
-- RANGE — по диапазону значений (чаще всего по дате)
CREATE TABLE orders (
    id          BIGINT,
    user_id     INTEGER,
    amount      NUMERIC(12,2),
    created_at  TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- LIST — по списку значений
CREATE TABLE users (
    id      BIGINT,
    region  TEXT NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE users_eu  PARTITION OF users FOR VALUES IN ('DE', 'FR', 'IT', 'ES');
CREATE TABLE users_us  PARTITION OF users FOR VALUES IN ('US', 'CA');
CREATE TABLE users_other PARTITION OF users DEFAULT;

-- HASH — равномерное распределение по хэшу
CREATE TABLE events (
    id      BIGINT,
    payload JSONB
) PARTITION BY HASH (id);

CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE events_2 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE events_3 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Partition Pruning

```sql
-- PostgreSQL автоматически пропускает нерелевантные партиции
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- → читает только orders_2024, не трогает orders_2023

-- убедиться что pruning работает
SET enable_partition_pruning = on;  -- включён по умолчанию
```

### Индексы и партиции

```sql
-- индекс создаётся на каждой партиции
CREATE INDEX ON orders (user_id);        -- создаст индекс на каждой партиции
CREATE INDEX ON orders (created_at);     -- аналогично

-- уникальный индекс должен включать ключ партиционирования
CREATE UNIQUE INDEX ON orders (id, created_at);  -- ✅
CREATE UNIQUE INDEX ON orders (id);              -- ❌ ошибка
```

### Управление партициями

```sql
-- добавить новую партицию
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- отсоединить партицию (превращается в обычную таблицу)
ALTER TABLE orders DETACH PARTITION orders_2023;

-- удалить старую партицию (мгновенно, без VACUUM)
DROP TABLE orders_2023;

-- посмотреть партиции
SELECT
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_bound
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child  ON pg_inherits.inhrelid  = child.oid
WHERE parent.relname = 'orders';
```

> [!tip] Автоматическое создание партиций
> Используй `pg_partman` — расширение для автоматического управления партициями по времени (создание будущих, архивирование/удаление старых).

---

## 🔹 Репликация

> [!note] Репликация
> Синхронизация данных между несколькими серверами PostgreSQL для высокой доступности и масштабирования чтения.

### Физическая репликация (Streaming Replication)

```
Master (Primary)                    Replica (Standby)
    │                                     │
    ├── записывает WAL ────────────────►  ├── wal receiver
    │                                     ├── применяет WAL
    │◄────── streaming replication ──────►│
    │                                     │
    │   read/write                        │   read only
```

```ini
# postgresql.conf на PRIMARY
wal_level = replica                  # или logical
max_wal_senders = 3                  # количество реплик
wal_keep_size = 1GB                  # держать WAL для реплик

# pg_hba.conf на PRIMARY — разрешить репликацию
host  replication  replicator  replica_ip/32  scram-sha-256
```

```bash
# создать базовую копию на реплике
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -P -R
# -R создаёт standby.signal и postgresql.auto.conf с primary_conninfo
```

```ini
# postgresql.conf на REPLICA
primary_conninfo = 'host=primary_host port=5432 user=replicator password=...'
hot_standby = on                     # разрешить чтение с реплики
```

### Синхронная vs асинхронная

```ini
# PRIMARY: синхронная репликация — транзакция не фиксируется
# пока реплика не подтвердит получение WAL
synchronous_standby_names = 'replica1'
synchronous_commit = on              # default

# асинхронная (возможна потеря данных при сбое мастера)
synchronous_commit = off
```

| Режим | Потеря данных при сбое | Производительность |
|-------|----------------------|-------------------|
| Синхронная | невозможна | ниже (ждём реплику) |
| Асинхронная | возможна (lag секунды) | выше |

### Failover и Switchover

```sql
-- проверить lag реплики (на PRIMARY)
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    (sent_lsn - replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;

-- продвинуть реплику до PRIMARY (на REPLICA)
SELECT pg_promote();
-- или через pg_ctl:
-- pg_ctl promote -D /var/lib/postgresql/data
```

> [!tip] Инструменты failover
> Для автоматического failover используй **Patroni** (рекомендуется) или **Repmgr**. Patroni использует etcd/consul/ZooKeeper для распределённого консенсуса при выборе нового мастера.

### Логическая репликация

```sql
-- на PRIMARY: создать публикацию
CREATE PUBLICATION my_pub FOR TABLE users, orders;
-- или все таблицы
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- на REPLICA: создать подписку
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=myapp user=replicator password=...'
    PUBLICATION my_pub;

-- посмотреть статус
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_publication_tables;
```

> [!note] Физическая vs Логическая репликация
> - **Физическая** — побайтовая копия, реплика read-only, та же версия PG
> - **Логическая** — на уровне строк, реплика может быть read-write, разные версии PG, можно реплицировать отдельные таблицы

---

## 🔹 Full-Text Search

```sql
-- tsvector — индексируемое представление документа
-- tsquery  — поисковый запрос

-- создание tsvector
SELECT to_tsvector('russian', 'Быстрая коричневая лиса прыгает');
-- → 'быстр':1 'коричнев':2 'лис':3 'прыгает':4

-- создание tsquery
SELECT to_tsquery('russian', 'лис & прыгает');
SELECT plainto_tsquery('russian', 'быстрая лиса');   -- без операторов
SELECT websearch_to_tsquery('russian', '"быстрая лиса"');  -- фраза

-- поиск
SELECT * FROM articles
WHERE to_tsvector('russian', content) @@ to_tsquery('russian', 'лис & прыгает');

-- с индексом:
ALTER TABLE articles ADD COLUMN content_tsv TSVECTOR
    GENERATED ALWAYS AS (to_tsvector('russian', content)) STORED;

CREATE INDEX idx_articles_fts ON articles USING GIN (content_tsv);

SELECT * FROM articles
WHERE content_tsv @@ plainto_tsquery('russian', 'быстрая лиса');

-- ранжирование результатов
SELECT
    title,
    ts_rank(content_tsv, query) AS rank
FROM articles, plainto_tsquery('russian', 'лиса') query
WHERE content_tsv @@ query
ORDER BY rank DESC;

-- подсветка результатов
SELECT ts_headline('russian', content, plainto_tsquery('russian', 'лиса'),
    'MaxWords=50, MinWords=20, StartSel=<b>, StopSel=</b>')
FROM articles;
```

---

## 🔹 Генерируемые колонки (Generated Columns)

```sql
-- STORED — вычисляется при INSERT/UPDATE, хранится на диске
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    first_name TEXT,
    last_name  TEXT,
    full_name  TEXT GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED,
    content    TEXT,
    content_tsv TSVECTOR GENERATED ALWAYS AS (to_tsvector('russian', content)) STORED
);

-- нельзя писать в generated column напрямую
INSERT INTO users (first_name, last_name) VALUES ('Иван', 'Петров');
SELECT full_name FROM users;  -- 'Иван Петров'
```

---

## 🔹 Расширения (Extensions)

```sql
-- посмотреть доступные расширения
SELECT name, default_version, comment FROM pg_available_extensions;

-- установить расширение
CREATE EXTENSION IF NOT EXISTS <name>;

-- удалить
DROP EXTENSION <name>;
```

### Популярные расширения

| Расширение | Назначение |
|-----------|------------|
| `pg_stat_statements` | статистика запросов |
| `uuid-ossp` | генерация UUID |
| `pgcrypto` | криптографические функции |
| `pg_trgm` | триграммный поиск (LIKE '%...%') |
| `hstore` | key-value в одной колонке |
| `PostGIS` | геопространственные данные |
| `pg_partman` | автоуправление партициями |
| `TimescaleDB` | временные ряды |
| `pg_vector` | векторные эмбеддинги (AI/ML) |
| `pg_cron` | планировщик задач внутри PG |
| `pgaudit` | аудит SQL-операций |

```sql
-- pg_trgm — поиск по подстроке
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_trgm ON users USING GIN (username gin_trgm_ops);
SELECT * FROM users WHERE username ILIKE '%alex%';

-- pgcrypto — хэширование паролей
CREATE EXTENSION pgcrypto;
INSERT INTO users (password) VALUES (crypt('mypassword', gen_salt('bf')));
SELECT * FROM users WHERE password = crypt('mypassword', password);

-- pg_cron — расписание внутри PostgreSQL
CREATE EXTENSION pg_cron;
SELECT cron.schedule('0 3 * * *', 'VACUUM ANALYZE orders');
SELECT * FROM cron.job;
```

---

## 🔹 Функции и процедуры

```sql
-- функция (возвращает значение)
CREATE OR REPLACE FUNCTION get_user_order_count(user_id_param INTEGER)
RETURNS INTEGER
LANGUAGE sql
AS $$
    SELECT COUNT(*)::INTEGER FROM orders WHERE user_id = user_id_param;
$$;

SELECT get_user_order_count(1);

-- функция на PL/pgSQL (процедурный язык)
CREATE OR REPLACE FUNCTION transfer_money(
    from_id INTEGER,
    to_id   INTEGER,
    amount  NUMERIC
) RETURNS VOID
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Source account % not found', from_id;
    END IF;

    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Target account % not found', to_id;
    END IF;
END;
$$;

SELECT transfer_money(1, 2, 500.00);

-- процедура (нет RETURNS, можно COMMIT внутри)
CREATE OR REPLACE PROCEDURE archive_old_orders(cutoff_date DATE)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < cutoff_date;
    DELETE FROM orders WHERE created_at < cutoff_date;
    COMMIT;
END;
$$;

CALL archive_old_orders('2023-01-01');
```

---

## 🔹 Триггеры

```sql
-- функция для триггера
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

-- триггер: автоматически обновлять updated_at
CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- триггер аудита: логировать изменения
CREATE TABLE audit_log (
    id         SERIAL PRIMARY KEY,
    table_name TEXT,
    operation  TEXT,
    old_data   JSONB,
    new_data   JSONB,
    changed_at TIMESTAMPTZ DEFAULT NOW(),
    changed_by TEXT DEFAULT current_user
);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP = 'DELETE' THEN row_to_json(OLD)::JSONB ELSE NULL END,
        CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW)::JSONB ELSE NULL END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

---

## 🔹 Материализованные представления (Materialized Views)

```sql
-- создать материализованное представление
CREATE MATERIALIZED VIEW mv_user_stats AS
SELECT
    u.id,
    u.username,
    COUNT(o.id)     AS order_count,
    SUM(o.amount)   AS total_spent,
    MAX(o.created_at) AS last_order_at
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;

-- индекс на материализованное представление
CREATE INDEX ON mv_user_stats(id);
CREATE INDEX ON mv_user_stats(total_spent DESC);

-- запрос — мгновенный (данные предвычислены)
SELECT * FROM mv_user_stats WHERE total_spent > 10000;

-- обновить данные
REFRESH MATERIALIZED VIEW mv_user_stats;                  -- блокирует на чтение
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_stats;    -- без блокировки (нужен UNIQUE INDEX)

-- уникальный индекс для CONCURRENTLY
CREATE UNIQUE INDEX ON mv_user_stats(id);
```

> [!tip] Когда использовать
> Материализованные представления идеальны для тяжёлых агрегирующих запросов (отчёты, дашборды), которые запускаются часто, а данные можно обновлять с задержкой (раз в час/день).

---

## 🔹 Итог

```
Партиционирование:
  RANGE  — по диапазону (даты, числа)
  LIST   — по списку значений
  HASH   — равномерное распределение
  Partition Pruning — автоматически пропускает ненужные партиции
  pg_partman — автоматическое управление

Репликация:
  Физическая — побайтовая, read-only реплика, streaming WAL
  Логическая — на уровне строк, отдельные таблицы, разные версии PG
  Синхронная — нет потери данных, медленнее
  Асинхронная — возможна потеря (lag), быстрее
  Patroni — автоматический failover

Full-Text Search:
  tsvector — документ, to_tsvector()
  tsquery  — запрос, to_tsquery() / plainto_tsquery()
  @@ — оператор поиска
  GIN индекс на tsvector колонку
  ts_rank() — ранжирование

Расширения:
  pg_stat_statements — мониторинг запросов
  pg_trgm            — LIKE '%...%' с индексом
  pgcrypto           — хэширование, шифрование
  PostGIS            — геоданные
  pg_partman         — партиции
  pg_vector          — AI эмбеддинги

Материализованные view:
  Предвычисленные результаты тяжёлых запросов
  REFRESH MATERIALIZED VIEW CONCURRENTLY
```
