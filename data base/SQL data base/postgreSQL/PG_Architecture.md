# PG Architecture — Архитектура PostgreSQL

> **Теги:** #postgresql #database #architecture #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[PostgreSQL]]

---

## 🔹 Общая архитектура

```
┌─────────────────────────────────────────────────────────┐
│                        CLIENT                           │
│              (psql / JDBC / libpq / app)                │
└────────────────────────┬────────────────────────────────┘
                         │ TCP (порт 5432) / Unix socket
┌────────────────────────▼────────────────────────────────┐
│                   POSTMASTER                            │
│          (главный процесс, слушает соединения)          │
│               fork() для каждого клиента                │
└──────┬──────────────────────────────────────┬───────────┘
       │                                      │
┌──────▼──────┐                    ┌──────────▼──────────┐
│  Backend    │  × N (per client)  │  Background Workers │
│  Process    │                    │  ─────────────────  │
│             │                    │  checkpointer       │
│  parser     │                    │  bgwriter           │
│  rewriter   │                    │  walwriter          │
│  planner    │                    │  autovacuum         │
│  executor   │                    │  stats collector    │
└──────┬──────┘                    │  wal sender/receiver│
       │                           └─────────────────────┘
┌──────▼──────────────────────────────────────────────────┐
│                   SHARED MEMORY                         │
│  ┌─────────────────┐   ┌──────────┐   ┌─────────────┐  │
│  │  Shared Buffers │   │ WAL Buf  │   │  Lock Table │  │
│  │  (кэш страниц)  │   │          │   │             │  │
│  └─────────────────┘   └──────────┘   └─────────────┘  │
└──────────────────────────────┬──────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────┐
│                    DISK STORAGE                         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Data Files  │  │   WAL Files  │  │  System       │  │
│  │  (base/)     │  │  (pg_wal/)   │  │  Catalog      │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 🔹 Процессы PostgreSQL

### Postmaster
Главный процесс — слушает входящие соединения и создаёт `fork()` backend-процесс для каждого клиента.

```bash
# посмотреть процессы
ps aux | grep postgres
```

### Backend процесс (на каждого клиента)
```
Жизненный цикл запроса в backend:
  1. Parser     — разобрать SQL в дерево разбора (parse tree)
  2. Rewriter   — применить правила (rules), раскрыть VIEW
  3. Planner    — выбрать оптимальный план выполнения
  4. Executor   — выполнить план, вернуть результат
```

### Фоновые процессы

| Процесс | Роль |
|---------|------|
| **checkpointer** | периодически сбрасывает грязные страницы на диск (checkpoint) |
| **bgwriter** | фоново записывает грязные страницы из shared buffers на диск |
| **walwriter** | сбрасывает WAL-буфер на диск |
| **autovacuum launcher** | запускает autovacuum workers для очистки мёртвых версий строк |
| **stats collector** | собирает статистику использования (pg_stat_*) |
| **wal sender** | отправляет WAL реплике (при репликации) |
| **wal receiver** | получает WAL от мастера (на реплике) |

---

## 🔹 Память

```
┌─────────────────────────────────────────────────────┐
│              SHARED MEMORY (общая для всех)         │
│                                                     │
│  shared_buffers        — кэш страниц таблиц/индексов│
│  wal_buffers           — буфер WAL перед записью    │
│  lock space            — таблица блокировок         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│           PER-PROCESS MEMORY (на каждый backend)    │
│                                                     │
│  work_mem              — сортировка, хэш-таблицы    │
│  maintenance_work_mem  — VACUUM, CREATE INDEX       │
│  temp_buffers          — временные таблицы          │
└─────────────────────────────────────────────────────┘
```

### Ключевые параметры памяти

```ini
# postgresql.conf

shared_buffers = 256MB          # кэш страниц. Рекомендация: 25% RAM
work_mem = 4MB                  # на каждую операцию сортировки/хэша
                                # Если 100 соединений × 5 операций = 500 × work_mem
maintenance_work_mem = 64MB     # для VACUUM, REINDEX, CREATE INDEX
effective_cache_size = 4GB      # подсказка планировщику: сколько OS кэша доступно
```

> [!tip] Настройка shared_buffers
> Стандартная рекомендация — 25% RAM. Но PostgreSQL также использует OS page cache, поэтому aggressive увеличение shared_buffers сверх 40% может не дать эффекта.

> [!warning] work_mem — осторожно
> `work_mem` задаётся **на одну операцию сортировки**, а не на соединение. Один сложный запрос может использовать несколько work_mem буферов. Формула: `work_mem × max_connections × 3-5 < RAM`.

---

## 🔹 WAL — Write-Ahead Log

> [!note] WAL (Write-Ahead Log)
> Журнал упреждающей записи. Все изменения сначала пишутся в WAL, потом применяются к страницам данных. Гарантирует **Durability** и **Atomicity** из ACID.

```
Без WAL (опасно):
  1. Изменить страницу в памяти
  2. В любой момент — сбой питания → данные потеряны

С WAL (надёжно):
  1. Записать изменение в WAL на диск (fsync)
  2. Изменить страницу в памяти (shared buffers)
  3. Фоново (bgwriter/checkpoint) сбросить страницу на диск
  При сбое: воспроизвести WAL → база консистентна
```

### Checkpoint
```
Checkpoint — момент, когда все грязные страницы из памяти
сброшены на диск. После checkpoint можно удалить старые WAL.

Параметры:
  checkpoint_timeout = 5min     # максимальное время между checkpoint
  max_wal_size = 1GB            # максимальный размер WAL до checkpoint
```

### WAL и репликация

```
Master                          Replica
  │                               │
  ├─ пишет WAL ──────────────────►├─ wal receiver получает WAL
  │                               ├─ применяет к своей копии БД
  │◄─────── streaming replication ─┤
```

---

## 🔹 Хранение данных на диске

### Структура директорий

```
$PGDATA/
├── base/              ← данные баз (каждая БД — своя папка по OID)
│   ├── 1/             ← template1
│   ├── 16384/         ← ваша БД
│   │   ├── 12345      ← файл таблицы (heap)
│   │   ├── 12345_vm   ← visibility map
│   │   └── 12345_fsm  ← free space map
├── pg_wal/            ← WAL файлы (сегменты по 16MB)
├── pg_stat/           ← файлы статистики
├── postgresql.conf    ← конфигурация
├── pg_hba.conf        ← аутентификация клиентов
└── postmaster.pid     ← PID мастер-процесса
```

### Страницы (Pages)

> [!note] Page
> Базовая единица хранения и I/O — **8 КБ** по умолчанию. Таблицы и индексы — набор страниц.

```
Структура страницы (8KB):
┌─────────────────────────────────┐
│ Page Header (24 bytes)          │
├─────────────────────────────────┤
│ Item Pointers (array of offsets)│
├─────────────────────────────────┤
│                                 │
│         FREE SPACE              │
│                                 │
├─────────────────────────────────┤
│ Tuple (row) N                   │
│ ...                             │
│ Tuple 2                         │
│ Tuple 1  (HeapTupleHeader+data) │
└─────────────────────────────────┘
```

---

## 🔹 System Catalog

> [!note] System Catalog
> Набор системных таблиц, хранящих метаданные обо всех объектах БД.

```sql
-- все таблицы в БД
SELECT tablename FROM pg_tables WHERE schemaname = 'public';

-- все колонки таблицы
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'users';

-- все индексы
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';

-- активные соединения
SELECT pid, usename, application_name, state, query
FROM pg_stat_activity;

-- размер таблицы
SELECT pg_size_pretty(pg_total_relation_size('users'));

-- размер БД
SELECT pg_size_pretty(pg_database_size('myapp'));

-- статистика по таблице (нужна для планировщика)
SELECT reltuples, relpages FROM pg_class WHERE relname = 'users';
```

---

## 🔹 Жизненный цикл запроса

```
Client отправляет SQL
        │
        ▼
   [Parser]
   SQL → Parse Tree
   Проверка синтаксиса
        │
        ▼
   [Rewriter]
   Раскрытие VIEW
   Применение RULES
        │
        ▼
   [Planner / Optimizer]
   Собирает статистику (pg_statistic)
   Генерирует возможные планы
   Оценивает cost каждого
   Выбирает минимальный cost
        │
        ▼
   [Executor]
   Выполняет план
   Читает страницы через shared_buffers
   Применяет фильтры, JOIN, сортировку
        │
        ▼
   Результат → Client
```

---

## 🔹 Конфигурация подключений

```ini
# postgresql.conf
max_connections = 100           # максимум одновременных соединений
listen_addresses = '*'          # слушать все интерфейсы
port = 5432
```

```
# pg_hba.conf — правила аутентификации
# TYPE  DATABASE  USER  ADDRESS       METHOD
local   all       all                 peer
host    all       all   127.0.0.1/32  md5
host    myapp     app   10.0.0.0/8    scram-sha-256
```

> [!tip] Connection Pooling
> При большом числе соединений используй пулер: **PgBouncer** (transaction/session pooling) или **Pgpool-II**. Сырые соединения к PostgreSQL дорогие — каждое fork() процесс.

---

## 🔹 Итог

```
Архитектура:
  Postmaster → fork() Backend (per client)
  Фоновые: checkpointer, bgwriter, walwriter, autovacuum

Память:
  shared_buffers  — общий кэш страниц (25% RAM)
  work_mem        — на операцию сортировки/хэша
  wal_buffers     — буфер WAL

WAL:
  Сначала запись в журнал, потом в данные
  Checkpoint — сброс грязных страниц на диск
  Основа для репликации и восстановления

Хранение:
  Страница = 8KB = базовая единица I/O
  base/ — данные, pg_wal/ — журнал

Запрос: Parser → Rewriter → Planner → Executor

Конфиг: postgresql.conf, pg_hba.conf
Мониторинг: pg_stat_activity, pg_class, pg_indexes
```
