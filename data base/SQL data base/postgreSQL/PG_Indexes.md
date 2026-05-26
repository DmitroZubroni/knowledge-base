# PG Indexes — Индексы PostgreSQL

> **Теги:** #postgresql #database #indexes #конспект  

> [!abstract] Связи
> [[main]] | [[main Data Base]] | [[main SQL DB]] | [[00_PostgreSQL]]

---

## 🔹 Что такое индекс

> [!note] Index
> Вспомогательная структура данных, хранящая отсортированные указатели на строки таблицы. Ускоряет поиск за счёт дополнительного места на диске и накладных расходов на запись.

```
Без индекса (Seq Scan):
  SELECT * FROM users WHERE email = 'a@b.com'
  → читаем все 1 000 000 строк → O(n)

С индексом (Index Scan):
  → B-tree: спускаемся по дереву → O(log n)
  → ~20 уровней дерева = ~20 чтений вместо 1 000 000
```

```
Индекс ускоряет:   SELECT, WHERE, JOIN, ORDER BY, GROUP BY
Индекс замедляет:  INSERT, UPDATE, DELETE (нужно обновить индекс)
```

---

## 🔹 Типы индексов

### Обзор

| Тип | Структура | Операторы | Применение |
|-----|-----------|-----------|------------|
| **B-tree** | сбалансированное дерево | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'abc%'` | по умолчанию, универсальный |
| **Hash** | хэш-таблица | только `=` | точное равенство |
| **GIN** | инвертированный индекс | `@>`, `?`, `&&`, `@@` | массивы, JSONB, full-text |
| **GiST** | обобщённое дерево поиска | геометрия, диапазоны | PostGIS, RANGE, full-text |
| **BRIN** | блочный диапазонный | `=`, `<`, `>` | огромные таблицы с физическим порядком |
| **SP-GiST** | разбиение пространства | геометрия, IP, текст | несбалансированные структуры |

---

## 🔹 B-tree индекс

```sql
-- создание (по умолчанию B-tree)
CREATE INDEX idx_users_email ON users(email);

-- убедиться что используется
EXPLAIN SELECT * FROM users WHERE email = 'test@mail.com';
-- → Index Scan using idx_users_email
```

### Когда работает

```sql
WHERE email = 'test@mail.com'          -- равенство
WHERE age > 18                         -- сравнение
WHERE age BETWEEN 18 AND 65            -- диапазон
WHERE username LIKE 'alex%'            -- префикс (не %alex)
ORDER BY created_at DESC LIMIT 10      -- сортировка
```

### Когда НЕ работает

```sql
WHERE LOWER(email) = 'test@mail.com'   -- функция над колонкой → функциональный индекс
WHERE email LIKE '%mail.com'           -- суффикс → полнотекстовый или pg_trgm
WHERE status = 'active'                -- если 90% строк active — seq scan выгоднее
WHERE id::TEXT = '123'                 -- неявное приведение типа
```

---

## 🔹 Hash индекс

```sql
CREATE INDEX idx_orders_status_hash ON orders USING HASH (status);
```

> [!note] Hash индекс
> Работает **только** для `=`. Меньше размер чем B-tree для этого случая. С PostgreSQL 10 поддерживается WAL (надёжен при сбоях).

> [!tip] B-tree vs Hash
> На практике B-tree почти всегда выигрывает или равен Hash — и при этом поддерживает больше операторов. Hash используют в специфических случаях с высокой кардинальностью и только операцией `=`.

---

## 🔹 GIN индекс

> [!note] GIN — Generalized Inverted Index
> Инвертированный индекс: хранит маппинг "значение → список строк с этим значением". Идеален для составных типов (массивы, JSONB, tsvector).

```sql
-- для массивов
CREATE INDEX idx_products_tags ON products USING GIN (tags);

-- для JSONB
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- оптимизированный для @> (только containment)
CREATE INDEX idx_events_payload2 ON events USING GIN (payload jsonb_path_ops);

-- для full-text search
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('russian', content));
```

```sql
-- операторы использующие GIN
WHERE tags @> ARRAY['mobile']              -- массив содержит
WHERE tags && ARRAY['mobile', 'tablet']    -- пересечение
WHERE 'mobile' = ANY(tags)                 -- содержит элемент
WHERE payload @> '{"status": "active"}'   -- JSONB containment
WHERE payload ? 'user_id'                  -- наличие ключа
```

---

## 🔹 GiST индекс

> [!note] GiST — Generalized Search Tree
> Расширяемая структура для сложных типов: геометрия, диапазоны, полнотекстовый поиск.

```sql
-- для диапазонов (EXCLUDE constraint)
CREATE INDEX idx_bookings_period ON bookings USING GIST (period);

-- для геометрии (PostGIS)
CREATE INDEX idx_places_location ON places USING GIST (location);

-- полнотекстовый поиск
CREATE INDEX idx_articles_fts ON articles USING GIST (to_tsvector('russian', content));
```

### GIN vs GiST для full-text

| | GIN | GiST |
|--|-----|------|
| Скорость поиска | быстрее | медленнее |
| Скорость обновления | медленнее | быстрее |
| Размер | больше | меньше |
| Выбор | статичные данные | часто обновляемые |

---

## 🔹 BRIN индекс

> [!note] BRIN — Block Range Index
> Хранит минимум и максимум значений для каждого блока страниц. Очень маленький, но работает только если данные физически упорядочены.

```sql
-- для временных рядов: новые данные добавляются в конец
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at);

-- размер pages_per_range (блоков на диапазон)
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at) WITH (pages_per_range = 128);
```

```
Пример: таблица logs с 100M строк, created_at монотонно растёт
  B-tree индекс: ~2GB
  BRIN индекс:   ~256KB
  Поиск: WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
  BRIN прочитает только нужные блоки диапазона
```

> [!tip] BRIN применим когда
> - Таблица очень большая (сотни GB и более)
> - Данные вставляются в порядке (логи, события, временные ряды)
> - Запросы по диапазонам дат/времени

---

## 🔹 Составной индекс

```sql
-- индекс по двум колонкам
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### Правило левого префикса

```sql
-- индекс (user_id, status) РАБОТАЕТ для:
WHERE user_id = 5                              -- ✅ левый префикс
WHERE user_id = 5 AND status = 'paid'          -- ✅ оба
WHERE user_id = 5 AND status = 'paid' AND ...  -- ✅ расширение

-- индекс (user_id, status) НЕ РАБОТАЕТ для:
WHERE status = 'paid'                          -- ❌ без user_id
WHERE status = 'paid' AND amount > 100         -- ❌
```

> [!tip] Порядок колонок в составном индексе
> 1. Сначала колонки с равенством (`=`)
> 2. Потом колонки с диапазоном (`<`, `>`, `BETWEEN`)
> ```sql
> -- запрос: WHERE user_id = 5 AND created_at > '2024-01-01'
> CREATE INDEX ON orders(user_id, created_at);    -- правильно
> CREATE INDEX ON orders(created_at, user_id);    -- хуже
> ```

---

## 🔹 Частичный индекс (Partial)

> [!note] Partial Index
> Индекс только по части строк — тех, которые удовлетворяют условию WHERE.

```sql
-- индексировать только активных пользователей
CREATE INDEX idx_active_users ON users(email)
    WHERE status = 'active';

-- индексировать только незакрытые заказы
CREATE INDEX idx_open_orders ON orders(user_id, created_at)
    WHERE status NOT IN ('delivered', 'cancelled');

-- уникальность только среди не-удалённых
CREATE UNIQUE INDEX idx_users_email_active ON users(email)
    WHERE deleted_at IS NULL;
```

```
Плюсы:
  - Меньше размер (нет ненужных строк)
  - Быстрее INSERT/UPDATE для строк вне условия
  - Точно попадает в планировщик для нужных запросов

Условие:
  WHERE в запросе должно соответствовать WHERE в индексе
```

---

## 🔹 Функциональный индекс

```sql
-- поиск без учёта регистра
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- теперь работает:
WHERE LOWER(email) = 'test@mail.com'

-- индекс на вычисляемое выражение
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM created_at));
-- WHERE EXTRACT(YEAR FROM created_at) = 2024

-- индекс на поле JSONB
CREATE INDEX idx_events_action ON events((payload ->> 'action'));
-- WHERE payload ->> 'action' = 'login'
```

---

## 🔹 Покрывающий индекс (INCLUDE)

> [!note] Covering Index
> Индекс содержит не только ключевые колонки для поиска, но и дополнительные колонки. Позволяет выполнить `Index Only Scan` — без обращения к таблице.

```sql
-- без INCLUDE: Index Scan + обращение к таблице за amount и status
CREATE INDEX idx_orders_user ON orders(user_id);

-- с INCLUDE: Index Only Scan — всё из индекса
CREATE INDEX idx_orders_user_cover ON orders(user_id) INCLUDE (amount, status);

-- теперь запрос читает только индекс:
SELECT user_id, amount, status FROM orders WHERE user_id = 5;
-- → Index Only Scan
```

> [!tip] INCLUDE vs составной индекс
> Колонки в `INCLUDE` не используются для поиска и сортировки — только для "покрытия". Это позволяет не раздувать дерево поиска лишними данными.

---

## 🔹 Уникальный индекс

```sql
-- уникальный индекс = UNIQUE constraint + индекс
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- составной уникальный
CREATE UNIQUE INDEX idx_user_role ON user_roles(user_id, role_id);

-- частичный уникальный (только среди активных)
CREATE UNIQUE INDEX idx_users_email_active ON users(email)
    WHERE deleted_at IS NULL;
```

---

## 🔹 Конкурентное создание (CONCURRENTLY)

```sql
-- обычное создание — блокирует таблицу на запись
CREATE INDEX idx_users_email ON users(email);

-- конкурентное — не блокирует, но дольше
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- конкурентное удаление
DROP INDEX CONCURRENTLY idx_users_email;
```

> [!warning] CONCURRENTLY в транзакции
> `CREATE INDEX CONCURRENTLY` нельзя выполнить внутри транзакции. При сбое остаётся "invalid" индекс — его нужно удалить и пересоздать.

---

## 🔹 Обслуживание индексов

```sql
-- посмотреть все индексы таблицы
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';

-- размер индексов
SELECT
    indexrelname AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;

-- использование индексов (нет ли неиспользуемых?)
SELECT
    relname AS table,
    indexrelname AS index,
    idx_scan AS scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- пересобрать индекс (убрать bloat)
REINDEX INDEX idx_users_email;
REINDEX TABLE users;
REINDEX CONCURRENTLY INDEX idx_users_email;  -- без блокировки
```

> [!tip] Неиспользуемые индексы
> Регулярно проверяй `pg_stat_user_indexes.idx_scan = 0`. Индексы без использования только замедляют запись — удаляй их.

---

## 🔹 Итог

```
Типы:
  B-tree   — универсальный (=, <, >, BETWEEN, LIKE 'x%')
  Hash     — только = (редко нужен)
  GIN      — массивы, JSONB, full-text (@>, ?, &&)
  GiST     — геометрия, диапазоны
  BRIN     — огромные таблицы с монотонными данными

Виды:
  Простой           — CREATE INDEX ON t(col)
  Составной         — ON t(col1, col2) — правило левого префикса
  Частичный         — ON t(col) WHERE condition
  Функциональный    — ON t(LOWER(col))
  Покрывающий       — ON t(col) INCLUDE (col2, col3)
  Уникальный        — CREATE UNIQUE INDEX

Создание без блокировки:
  CREATE INDEX CONCURRENTLY

Мониторинг:
  pg_indexes             — список индексов
  pg_stat_user_indexes   — статистика использования
  REINDEX                — пересборка

Правило:
  Индексируй Foreign Keys, WHERE-колонки с высокой кардинальностью
  Удаляй индексы с idx_scan = 0
```
