# PG Transactions — ACID, MVCC, Блокировки

> **Теги:** #postgresql #database #transactions #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[PostgreSQL]]

---

## 🔹 ACID

> [!note] ACID
> Набор свойств, гарантирующих надёжность транзакций даже при сбоях и параллельном доступе.

| Свойство | Описание | Как реализовано в PG |
|----------|----------|----------------------|
| **Atomicity** | Все операции — или ни одна | ROLLBACK + WAL |
| **Consistency** | БД переходит из одного корректного состояния в другое | Constraints, triggers |
| **Isolation** | Параллельные транзакции не мешают друг другу | MVCC + блокировки |
| **Durability** | Зафиксированные данные не теряются при сбое | WAL + fsync |

---

## 🔹 Управление транзакцией

```sql
BEGIN;                    -- начать транзакцию (или BEGIN TRANSACTION)
-- ... SQL операции ...
COMMIT;                   -- зафиксировать изменения
-- или
ROLLBACK;                 -- откатить все изменения

-- SAVEPOINT — точка отката внутри транзакции
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  SAVEPOINT after_order;

  INSERT INTO order_items (...) VALUES (...);
  -- что-то пошло не так только здесь
  ROLLBACK TO SAVEPOINT after_order;  -- откат только до точки

  -- продолжаем с другой логикой
COMMIT;

-- RELEASE SAVEPOINT — освободить (не откатывает)
RELEASE SAVEPOINT after_order;
```

```sql
-- автокоммит по умолчанию в psql — каждая команда = своя транзакция
-- явный BEGIN нужен чтобы объединить несколько команд

-- пример: перевод денег (атомарно)
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
  -- если любая команда упала — ROLLBACK автоматически при ошибке
COMMIT;
```

---

## 🔹 MVCC — Multi-Version Concurrency Control

> [!note] MVCC
> Механизм параллельного доступа без блокировок на чтение. Каждая транзакция видит **снимок** (snapshot) данных на момент своего начала. Читатели не блокируют писателей, писатели не блокируют читателей.

### Как работает

```
Таблица users (упрощённо):
┌────┬────────┬────────┬──────────┬──────────┐
│ id │  name  │ xmin   │ xmax     │ (видна?) │
├────┼────────┼────────┼──────────┼──────────┤
│  1 │ Alice  │ txid=5 │ txid=10  │ нет — удалена tx10 │
│  1 │ Alice2 │ txid=10│ infinity │ да — актуальная     │
│  2 │ Bob    │ txid=3 │ infinity │ да                  │
└────┴────────┴────────┴──────────┴──────────┘

xmin — транзакция, создавшая версию строки
xmax — транзакция, удалившая/обновившая строку (0 = живая)
```

```
Транзакция A (txid=15) начала в момент T1:
  видит все строки с xmin <= 15 и xmax = 0 (или xmax > 15)
  НЕ видит изменения транзакций начатых после T1

Транзакция B (txid=20) делает UPDATE:
  помечает старую версию xmax=20
  создаёт новую версию xmin=20

Транзакция A по-прежнему видит старую версию
→ нет блокировки чтения
```

### Dead Tuples и VACUUM

```
Проблема MVCC:
  Старые версии строк (dead tuples) остаются на диске
  Место не освобождается автоматически
  Таблица раздувается (bloat)

Решение: VACUUM
  VACUUM           — очищает dead tuples, место возвращается в таблицу
  VACUUM FULL      — полная перестройка таблицы, возвращает место ОС (блокирует!)
  VACUUM ANALYZE   — очистка + обновление статистики планировщика
  AUTOVACUUM       — фоновый процесс, запускает VACUUM автоматически
```

```sql
-- ручной VACUUM
VACUUM users;
VACUUM ANALYZE users;
VACUUM FULL users;          -- осторожно: блокирует таблицу

-- посмотреть bloat и статистику VACUUM
SELECT
    relname,
    n_dead_tup,
    n_live_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE relname = 'users';
```

> [!warning] VACUUM FULL
> `VACUUM FULL` полностью перестраивает таблицу и блокирует её на всё время. Используй только в окно обслуживания. Для продакшн предпочти `pg_repack` — выполняет то же самое без блокировки.

---

## 🔹 Аномалии параллельного доступа

> [!note] Аномалии
> Проблемы, возникающие при параллельном выполнении транзакций без должной изоляции.

### Dirty Read — грязное чтение
```
Tx A:  BEGIN → UPDATE users SET age=99 WHERE id=1 → (не зафиксировано)
Tx B:  SELECT age FROM users WHERE id=1 → видит 99 (незафиксированное!)
Tx A:  ROLLBACK → возраст откатывается
Tx B:  работает с неверными данными
```

### Non-Repeatable Read — неповторяемое чтение
```
Tx A:  BEGIN → SELECT age FROM users WHERE id=1 → 25
Tx B:  UPDATE users SET age=30 WHERE id=1 → COMMIT
Tx A:  SELECT age FROM users WHERE id=1 → 30  ← другой результат!
```

### Phantom Read — фантомное чтение
```
Tx A:  BEGIN → SELECT COUNT(*) FROM orders WHERE user_id=1 → 5
Tx B:  INSERT INTO orders (user_id,...) VALUES (1,...) → COMMIT
Tx A:  SELECT COUNT(*) FROM orders WHERE user_id=1 → 6  ← новая строка!
```

### Lost Update — потерянное обновление
```
Tx A:  SELECT balance FROM accounts WHERE id=1 → 1000
Tx B:  SELECT balance FROM accounts WHERE id=1 → 1000
Tx A:  UPDATE accounts SET balance = 1000 - 100 → 900
Tx B:  UPDATE accounts SET balance = 1000 + 200 → 1200  ← перезаписал A!
Итог:  1200 вместо правильного 1100
```

---

## 🔹 Уровни изоляции

```sql
BEGIN TRANSACTION ISOLATION LEVEL <уровень>;
-- или
SET TRANSACTION ISOLATION LEVEL <уровень>;
```

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update |
|---------|-----------|---------------------|--------------|-------------|
| `READ UNCOMMITTED` | возможен | возможен | возможен | возможен |
| `READ COMMITTED` *(default)* | ❌ | возможен | возможен | возможен |
| `REPEATABLE READ` | ❌ | ❌ | ❌ в PG* | ❌ в PG* |
| `SERIALIZABLE` | ❌ | ❌ | ❌ | ❌ |

> [!note] PostgreSQL и MVCC
> PostgreSQL не реализует `READ UNCOMMITTED` — даже при указании этого уровня ведёт себя как `READ COMMITTED`. `REPEATABLE READ` в PostgreSQL через MVCC защищает и от Phantom Read (в отличие от стандарта SQL).

### READ COMMITTED (по умолчанию)
```sql
BEGIN;
-- каждый SELECT видит снимок на момент ЭТОГО SELECT (не BEGIN)
SELECT * FROM orders WHERE status = 'pending';
-- если между двумя SELECT другая tx изменила данные — увидим новое
SELECT * FROM orders WHERE status = 'pending';
COMMIT;
```

### REPEATABLE READ
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- все SELECT видят снимок на момент ПЕРВОГО запроса в транзакции
-- подходит для: отчёты, выгрузки — нужна консистентность среза данных
SELECT SUM(amount) FROM orders;
SELECT COUNT(*) FROM orders;  -- та же картина, даже если другие tx меняли данные
COMMIT;
```

### SERIALIZABLE
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- SSI (Serializable Snapshot Isolation) — полная изоляция
-- транзакции выполняются как будто строго последовательно
-- при конфликте одна из транзакций получает ошибку serialization failure
-- нужно повторить транзакцию при ошибке 40001
COMMIT;
```

```python
# обработка serialization failure в коде
for attempt in range(3):
    try:
        with conn.transaction(isolation_level='SERIALIZABLE'):
            # ... бизнес-логика ...
        break
    except SerializationFailure:
        if attempt == 2:
            raise
        continue
```

---

## 🔹 Блокировки

### Табличные блокировки (Table-Level Locks)

```sql
-- явная блокировка таблицы
LOCK TABLE orders IN SHARE MODE;
LOCK TABLE orders IN EXCLUSIVE MODE;
LOCK TABLE orders IN ACCESS EXCLUSIVE MODE;  -- самая строгая
```

| Режим | Конфликтует с | Когда используется |
|-------|-------------|-------------------|
| `ACCESS SHARE` | ACCESS EXCLUSIVE | SELECT |
| `ROW SHARE` | EXCLUSIVE, ACCESS EXCLUSIVE | SELECT FOR UPDATE/SHARE |
| `ROW EXCLUSIVE` | SHARE, EXCLUSIVE, ACCESS EXCLUSIVE | INSERT, UPDATE, DELETE |
| `SHARE` | ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE INDEX |
| `EXCLUSIVE` | всё кроме ACCESS SHARE | редко, явно |
| `ACCESS EXCLUSIVE` | всё | ALTER TABLE, DROP, VACUUM FULL |

### Строчные блокировки (Row-Level Locks)

```sql
-- SELECT FOR UPDATE — заблокировать строки для обновления
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- другие транзакции не смогут UPDATE/DELETE эту строку
UPDATE orders SET status = 'processing' WHERE id = 1;
COMMIT;

-- SELECT FOR SHARE — заблокировать для чтения (другие могут тоже FOR SHARE)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- SKIP LOCKED — пропустить заблокированные строки (очереди задач)
SELECT * FROM jobs WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- NOWAIT — не ждать блокировку, вернуть ошибку сразу
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
```

> [!tip] SKIP LOCKED для очередей
> Классический паттерн для job queue: несколько воркеров параллельно берут задачи без конкуренции за одну строку.

### Дедлок (Deadlock)

```
Tx A:  LOCK row 1 → ждёт row 2 (занята Tx B)
Tx B:  LOCK row 2 → ждёт row 1 (занята Tx A)
→ взаимная блокировка = deadlock

PostgreSQL обнаруживает дедлок и откатывает одну из транзакций:
ERROR: deadlock detected
DETAIL: Process 123 waits for ShareLock on transaction 456
```

```sql
-- предотвращение дедлоков: всегда блокировать в одном порядке
-- ❌ опасно: разный порядок в разных транзакциях
Tx A: UPDATE accounts SET ... WHERE id = 1; UPDATE ... WHERE id = 2;
Tx B: UPDATE accounts SET ... WHERE id = 2; UPDATE ... WHERE id = 1;

-- ✅ безопасно: одинаковый порядок
Tx A: UPDATE accounts SET ... WHERE id = 1; UPDATE ... WHERE id = 2;
Tx B: UPDATE accounts SET ... WHERE id = 1; UPDATE ... WHERE id = 2;
```

### Мониторинг блокировок

```sql
-- текущие блокировки
SELECT
    pid,
    locktype,
    relation::regclass AS table,
    mode,
    granted
FROM pg_locks
WHERE NOT granted;

-- кто кого блокирует
SELECT
    blocking.pid     AS blocking_pid,
    blocking.query   AS blocking_query,
    blocked.pid      AS blocked_pid,
    blocked.query    AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- завершить зависший процесс
SELECT pg_terminate_backend(pid);
SELECT pg_cancel_backend(pid);   -- мягче: отменяет запрос, не сессию
```

---

## 🔹 Advisory Locks

> [!note] Advisory Lock
> Пользовательские блокировки — PostgreSQL не привязывает их к конкретным строкам/таблицам. Используются для координации между приложениями.

```sql
-- сессионные (держатся до явного unlock или закрытия соединения)
SELECT pg_advisory_lock(12345);           -- эксклюзивная
SELECT pg_advisory_lock_shared(12345);    -- разделяемая
SELECT pg_advisory_unlock(12345);

-- транзакционные (освобождаются при COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);

-- попытка без ожидания (возвращает true/false)
SELECT pg_try_advisory_lock(12345);

-- пример: distributed lock для крона
BEGIN;
  IF pg_try_advisory_xact_lock(42) THEN
    -- только один процесс выполняет это
    PERFORM run_scheduled_job();
  END IF;
COMMIT;
```

---

## 🔹 Итог

```
ACID:
  Atomicity   — WAL + ROLLBACK
  Consistency — Constraints + Triggers
  Isolation   — MVCC + Locks
  Durability  — WAL + fsync

MVCC:
  Читатели не блокируют писателей
  Каждая tx видит снимок данных
  Dead tuples → VACUUM (autovacuum)

Уровни изоляции:
  READ COMMITTED   — default, каждый SELECT = новый снимок
  REPEATABLE READ  — снимок на момент первого запроса tx
  SERIALIZABLE     — полная изоляция, возможен serialization failure

Аномалии:
  Dirty Read          — READ COMMITTED устраняет
  Non-Repeatable Read — REPEATABLE READ устраняет
  Phantom Read        — SERIALIZABLE устраняет
  Lost Update         — FOR UPDATE или REPEATABLE READ

Блокировки:
  FOR UPDATE          — строчная эксклюзивная
  FOR SHARE           — строчная разделяемая
  SKIP LOCKED         — паттерн очередей
  NOWAIT              — не ждать, бросить ошибку
  Дедлок              — всегда блокировать в одном порядке

Мониторинг:
  pg_locks            — текущие блокировки
  pg_blocking_pids()  — кто блокирует
  pg_terminate_backend() — завершить процесс
```
