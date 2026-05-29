# MySQL Transactions — ACID, MVCC, Блокировки

> **Теги:** #mysql #database #transactions #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[MySQL]]

---

## 🔹 ACID в MySQL / InnoDB

| Свойство | Описание | Реализация в InnoDB |
|----------|----------|---------------------|
| **Atomicity** | Все операции или ни одна | Undo Log + ROLLBACK |
| **Consistency** | БД остаётся в корректном состоянии | Constraints, FK, Triggers |
| **Isolation** | Параллельные транзакции не мешают друг другу | MVCC + Lock система |
| **Durability** | Зафиксированные данные не теряются | Redo Log + `innodb_flush_log_at_trx_commit` |

---

## 🔹 Управление транзакцией

```sql
-- явное управление
START TRANSACTION;               -- или BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;

-- откат
START TRANSACTION;
DELETE FROM orders WHERE user_id = 5;
ROLLBACK;                        -- всё отменено

-- SAVEPOINT
START TRANSACTION;
  INSERT INTO orders (user_id, amount) VALUES (1, 100);
  SAVEPOINT sp_order;

  INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 10);
  -- что-то пошло не так только здесь
  ROLLBACK TO SAVEPOINT sp_order;

  -- продолжаем с другой логикой
COMMIT;

RELEASE SAVEPOINT sp_order;     -- освободить savepoint (без отката)
```

### Autocommit

```sql
-- по умолчанию каждая команда = отдельная транзакция
SHOW VARIABLES LIKE 'autocommit';    -- ON

-- отключить autocommit для сессии
SET autocommit = 0;
-- теперь нужно явно COMMIT или ROLLBACK

-- включить обратно
SET autocommit = 1;
```

> [!warning] DDL и транзакции
> В MySQL `DDL` команды (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) вызывают **неявный COMMIT** — их нельзя откатить. В отличие от PostgreSQL, где DDL транзакционен.

---

## 🔹 MVCC в InnoDB

> [!note] MVCC — Multi-Version Concurrency Control
> Механизм параллельного доступа: читатели не блокируют писателей. Каждая транзакция видит снимок данных на момент своего начала.

### Как реализовано

```
Каждая строка InnoDB содержит скрытые поля:
  DB_TRX_ID  — ID транзакции, создавшей/изменившей строку
  DB_ROLL_PTR — указатель на предыдущую версию в Undo Log
  DB_ROW_ID  — внутренний row ID (если нет PK)

Undo Log — цепочка предыдущих версий строки:

  Текущая версия (TRX_ID=100): name='Alice2'
      │
      └─► Undo: name='Alice'  (TRX_ID=90)
              │
              └─► Undo: name='Alice_old'  (TRX_ID=80)

Транзакция с ID=85 читает строку:
  TRX_ID=100 → создана после 85 → не видна
  TRX_ID=90  → создана после 85 → не видна
  TRX_ID=80  → создана до 85   → видна → 'Alice_old'
```

### Read View

```
При старте транзакции (READ COMMITTED) или первого SELECT (REPEATABLE READ)
создаётся Read View:
  - список активных транзакций на этот момент
  - min_trx_id — минимальный ID активной транзакции
  - max_trx_id — следующий ID (ещё не существует)

Строка видна если:
  DB_TRX_ID < min_trx_id          → зафиксирована до нашей транзакции → видна
  DB_TRX_ID >= max_trx_id         → создана после нас → не видна
  DB_TRX_ID в списке активных     → не зафиксирована → не видна
  иначе                           → видна
```

---

## 🔹 Аномалии параллельного доступа

### Dirty Read — грязное чтение
```
Tx A: BEGIN → UPDATE users SET age=99 WHERE id=1  (не зафиксировано)
Tx B: SELECT age FROM users WHERE id=1 → 99  ← читает незафиксированное!
Tx A: ROLLBACK
Tx B: работает с несуществующими данными
```

### Non-Repeatable Read — неповторяемое чтение
```
Tx A: BEGIN → SELECT age FROM users WHERE id=1 → 25
Tx B: UPDATE users SET age=30 WHERE id=1 → COMMIT
Tx A: SELECT age FROM users WHERE id=1 → 30  ← другой результат!
```

### Phantom Read — фантомное чтение
```
Tx A: BEGIN → SELECT COUNT(*) FROM orders WHERE user_id=1 → 5
Tx B: INSERT INTO orders (user_id,...) VALUES (1,...) → COMMIT
Tx A: SELECT COUNT(*) FROM orders WHERE user_id=1 → 6  ← новая строка!
```

### Lost Update — потерянное обновление
```
Tx A: SELECT balance WHERE id=1 → 1000
Tx B: SELECT balance WHERE id=1 → 1000
Tx A: UPDATE SET balance = 1000 - 100 → 900; COMMIT
Tx B: UPDATE SET balance = 1000 + 500 → 1500; COMMIT  ← перезаписал A!
Итог: 1500 вместо правильного 1400
```

---

## 🔹 Уровни изоляции

```sql
-- установить уровень для текущей сессии
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- установить для следующей транзакции
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- глобально (требует рестарт или FLUSH)
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- посмотреть текущий
SELECT @@transaction_isolation;
SHOW VARIABLES LIKE 'transaction_isolation';
```

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------|-----------|---------------------|--------------|
| `READ UNCOMMITTED` | возможен | возможен | возможен |
| `READ COMMITTED` | ❌ | возможен | возможен |
| `REPEATABLE READ` *(default)* | ❌ | ❌ | ❌ в InnoDB* |
| `SERIALIZABLE` | ❌ | ❌ | ❌ |

> [!note] REPEATABLE READ в MySQL
> MySQL по умолчанию `REPEATABLE READ` (PostgreSQL — `READ COMMITTED`). InnoDB устраняет Phantom Read через **gap locks** — не через MVCC, как PostgreSQL. Gap locks блокируют диапазоны, что может приводить к дедлокам.

### READ COMMITTED

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
-- каждый SELECT видит новый снимок (актуальные зафиксированные данные)
SELECT * FROM orders WHERE status = 'pending';   -- видит снимок на момент SELECT
-- ... другая транзакция делает INSERT ...
SELECT * FROM orders WHERE status = 'pending';   -- может видеть новые строки
COMMIT;
```

### REPEATABLE READ (default)

```sql
BEGIN;
-- снимок фиксируется на первом SELECT в транзакции
SELECT COUNT(*) FROM orders WHERE user_id = 1;  -- → 5, снимок зафиксирован
-- ... другая транзакция делает INSERT user_id=1 и COMMIT ...
SELECT COUNT(*) FROM orders WHERE user_id = 1;  -- → 5, тот же снимок!
COMMIT;
```

### SERIALIZABLE

```sql
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- все SELECT превращаются в SELECT ... LOCK IN SHARE MODE
-- полная изоляция, но много блокировок → низкая конкурентность
BEGIN;
SELECT * FROM accounts WHERE id = 1;  -- ставит SHARED LOCK на строку
-- другая транзакция не может UPDATE эту строку до COMMIT
COMMIT;
```

---

## 🔹 Блокировки InnoDB

### Row-level блокировки

```sql
-- Shared Lock (S) — разделяемая блокировка чтения
SELECT * FROM orders WHERE id = 1 LOCK IN SHARE MODE;
-- другие транзакции могут читать (S), но не изменять (X)

-- Exclusive Lock (X) — эксклюзивная блокировка записи
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- другие транзакции не могут ни читать с блокировкой, ни изменять

-- SKIP LOCKED (MySQL 8.0) — пропустить заблокированные строки
SELECT * FROM jobs WHERE status = 'pending'
ORDER BY created_at LIMIT 1
FOR UPDATE SKIP LOCKED;   -- паттерн очереди задач

-- NOWAIT (MySQL 8.0) — не ждать блокировку
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
-- сразу ошибка: ERROR 3572: Statement aborted because lock(s) could not be acquired
```

### Типы блокировок InnoDB

| Блокировка | Описание | Когда |
|-----------|----------|-------|
| **Record Lock** | блокировка конкретной строки | `WHERE id = 5 FOR UPDATE` |
| **Gap Lock** | блокировка диапазона между строками | `WHERE id BETWEEN 5 AND 10` в RR |
| **Next-Key Lock** | Record Lock + Gap Lock слева | по умолчанию в RR |
| **Insert Intention Lock** | намерение вставить в gap | перед INSERT |
| **Table Lock** | вся таблица | DDL, `LOCK TABLES` |

> [!note] Gap Lock и REPEATABLE READ
> Gap Lock — уникальная особенность InnoDB. Блокирует промежутки между строками, чтобы предотвратить Phantom Read. Активен в `REPEATABLE READ`. В `READ COMMITTED` Gap Lock отключён — меньше дедлоков, но возможен Phantom Read.

```sql
-- пример Gap Lock
-- Таблица: id = 1, 5, 10
BEGIN;
SELECT * FROM users WHERE id BETWEEN 3 AND 7 FOR UPDATE;
-- Gap Lock заблокирует диапазон (1, 10):
-- другая транзакция не может INSERT id=2,3,4,5,6,7,8,9
COMMIT;
```

### LOCK TABLES (таблица целиком)

```sql
-- явная блокировка таблицы (обходит MVCC!)
LOCK TABLES users READ;           -- другие могут читать, нельзя писать
LOCK TABLES users WRITE;          -- эксклюзивный доступ

-- разблокировать
UNLOCK TABLES;
```

> [!warning] LOCK TABLES
> `LOCK TABLES` блокирует таблицу полностью и неявно фиксирует текущую транзакцию. Используй только в крайних случаях (например, при создании резервной копии без инструментов типа mysqldump --single-transaction).

---

## 🔹 Дедлок (Deadlock)

```
Tx A: LOCK row 1 → ждёт row 2
Tx B: LOCK row 2 → ждёт row 1
→ взаимная блокировка = deadlock

InnoDB автоматически обнаруживает дедлок:
→ откатывает транзакцию с наименьшим объёмом изменений
→ другая транзакция получает блокировку и завершается
```

```sql
-- посмотреть последний дедлок
SHOW ENGINE INNODB STATUS\G
-- раздел LATEST DETECTED DEADLOCK

-- настройка
SHOW VARIABLES LIKE 'innodb_deadlock_detect';  -- ON по умолчанию
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout'; -- 50 сек по умолчанию
SET innodb_lock_wait_timeout = 10;              -- уменьшить таймаут ожидания
```

### Предотвращение дедлоков

```sql
-- ✅ всегда блокировать в одном порядке
-- Tx A и Tx B: всегда сначала меньший id
UPDATE accounts SET balance = balance - 100 WHERE id = MIN(id1, id2);
UPDATE accounts SET balance = balance + 100 WHERE id = MAX(id1, id2);

-- ✅ минимизировать размер транзакции
-- не держать транзакцию открытой дольше необходимого

-- ✅ READ COMMITTED уменьшает дедлоки (нет Gap Locks)
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- ✅ SELECT FOR UPDATE вместо двух запросов
-- ❌ SELECT → бизнес-логика → UPDATE (большой gap между чтением и записью)
-- ✅ SELECT FOR UPDATE → бизнес-логика → UPDATE (строка заблокирована сразу)
```

---

## 🔹 Мониторинг блокировок

```sql
-- текущие блокировки (MySQL 8.0)
SELECT * FROM performance_schema.data_locks;

-- ожидающие блокировки
SELECT * FROM performance_schema.data_lock_waits;

-- кто кого блокирует
SELECT
    r.trx_id              AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query           AS waiting_query,
    b.trx_id              AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query           AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;

-- завершить блокирующий процесс
KILL <blocking_thread_id>;

-- активные транзакции
SELECT trx_id, trx_started, trx_state, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

---

## 🔹 Итог

```
ACID:
  Atomicity   — Undo Log + ROLLBACK
  Consistency — Constraints + FK
  Isolation   — MVCC + Locks
  Durability  — Redo Log + innodb_flush_log_at_trx_commit=1

Управление транзакцией:
  START TRANSACTION / BEGIN
  COMMIT / ROLLBACK
  SAVEPOINT / ROLLBACK TO SAVEPOINT
  DDL → неявный COMMIT (нельзя откатить!)

MVCC:
  Читатели не блокируют писателей
  Undo Log хранит предыдущие версии
  Read View — снимок данных на момент транзакции

Уровни изоляции (default: REPEATABLE READ):
  READ UNCOMMITTED — грязное чтение (не использовать)
  READ COMMITTED   — каждый SELECT = новый снимок, меньше блокировок
  REPEATABLE READ  — снимок фиксируется, Gap Locks
  SERIALIZABLE     — полная изоляция, S-lock на все SELECT

Блокировки:
  LOCK IN SHARE MODE    — S-lock (читать, нельзя изменять)
  FOR UPDATE            — X-lock (эксклюзивный)
  SKIP LOCKED           — пропустить заблокированные (очереди)
  NOWAIT                — ошибка если занято
  Gap Lock              — диапазонная (REPEATABLE READ)

Дедлок:
  InnoDB обнаруживает автоматически
  Откатывает транзакцию с меньшим объёмом изменений
  Профилактика: единый порядок блокировок
  SHOW ENGINE INNODB STATUS — последний дедлок
```
