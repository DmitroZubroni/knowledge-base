# MySQL Architecture — Архитектура MySQL

> **Теги:** #mysql #database #architecture #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[MySQL]]

---

## 🔹 Общая архитектура

```
┌─────────────────────────────────────────────────────────┐
│                        CLIENT                           │
│         (mysql CLI / JDBC / MySQL Connector)            │
└────────────────────────┬────────────────────────────────┘
                         │ TCP (порт 3306) / Unix socket
┌────────────────────────▼────────────────────────────────┐
│                  CONNECTION LAYER                       │
│         Аутентификация, SSL, Thread Pool                │
│         Один поток (thread) на каждое соединение        │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                    SQL LAYER                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │  Parser  │  │ Optimizer│  │  Cache   │  │ Query  │  │
│  │ (разбор) │  │  (план)  │  │ (8.0 ✝) │  │Executor│  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│              STORAGE ENGINE API (pluggable)             │
├────────────┬───────────┬──────────────┬─────────────────┤
│   InnoDB   │  MyISAM   │    Memory    │      CSV ...    │
│  (default) │ (legacy)  │  (tmp tables)│                 │
└────────────┴───────────┴──────────────┴─────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                    DISK STORAGE                         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ .ibd файлы   │  │  Binary Log  │  │  Redo / Undo  │  │
│  │ (tablespace) │  │  (binlog)    │  │     Log       │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

> [!note] Ключевое отличие от PostgreSQL
> MySQL разделяет SQL-слой и движок хранения. Оптимизатор не знает о внутренностях движка — общается через абстрактный API. Это даёт гибкость, но ограничивает возможности оптимизатора.

---

## 🔹 Storage Engines

| Engine | Транзакции | FK | Row Locks | Full-text | Применение |
|--------|-----------|-----|-----------|-----------|------------|
| **InnoDB** | ✅ | ✅ | ✅ | ✅ (5.6+) | стандарт, OLTP |
| **MyISAM** | ❌ | ❌ | ❌ (table lock) | ✅ | legacy, read-heavy |
| **Memory** | ❌ | ❌ | ✅ | ❌ | временные таблицы, кэш |
| **Archive** | ❌ | ❌ | ❌ | ❌ | компрессия, логи |
| **CSV** | ❌ | ❌ | ❌ | ❌ | обмен данными |
| **Blackhole** | ❌ | ❌ | ❌ | ❌ | репликация без хранения |
| **NDB (Cluster)** | ✅ | ✅ | ✅ | ❌ | MySQL Cluster |

```sql
-- посмотреть доступные движки
SHOW ENGINES;

-- создать таблицу с указанием движка
CREATE TABLE logs (id INT, msg TEXT) ENGINE = Archive;

-- посмотреть движок таблицы
SELECT ENGINE FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp' AND TABLE_NAME = 'users';

-- сменить движок
ALTER TABLE users ENGINE = InnoDB;
```

> [!tip] Всегда используй InnoDB
> MyISAM — устаревший движок без транзакций и FK. С MySQL 5.5 InnoDB стал движком по умолчанию. Нет причин использовать MyISAM в новых проектах.

---

## 🔹 InnoDB — внутренняя архитектура

```
┌─────────────────────────────────────────────────────────┐
│                   InnoDB MEMORY                         │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Buffer Pool                            │   │
│  │  (кэш страниц данных и индексов)                 │   │
│  │  innodb_buffer_pool_size = 128MB (default)       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  Log Buffer     │  │ Change     │  │  Adaptive   │  │
│  │  (redo log buf) │  │ Buffer     │  │  Hash Index │  │
│  └─────────────────┘  └────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   InnoDB DISK                           │
│                                                         │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │  Tablespace  │  │ Redo Log │  │    Undo Log       │  │
│  │  (.ibd файл) │  │(ib_logfile│  │  (для MVCC и     │  │
│  │  данные +    │  │  0,1...) │  │   откатов)        │  │
│  │  индексы     │  └──────────┘  └───────────────────┘  │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### Buffer Pool

> [!note] Buffer Pool
> Главный кэш InnoDB — хранит страницы таблиц и индексов в памяти. Аналог `shared_buffers` в PostgreSQL. Самый важный параметр производительности.

```ini
# my.cnf
innodb_buffer_pool_size = 1G       # рекомендация: 50-80% RAM
innodb_buffer_pool_instances = 4   # разбить на несколько пулов (для многопоточности)
                                   # 1 instance на каждый 1GB пула
```

```sql
-- статистика Buffer Pool
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- hit rate — должен быть > 99%
SELECT
    (1 - (
        variable_value / (
            SELECT variable_value FROM performance_schema.global_status
            WHERE variable_name = 'Innodb_buffer_pool_read_requests'
        )
    )) * 100 AS hit_rate_pct
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_buffer_pool_reads';
```

### Redo Log

> [!note] Redo Log
> Журнал повтора — записывает все изменения для восстановления после сбоя. Аналог WAL в PostgreSQL. Круговой буфер фиксированного размера.

```ini
innodb_log_file_size = 256M        # размер одного redo log файла
innodb_log_files_in_group = 2      # количество файлов (ib_logfile0, ib_logfile1)
innodb_flush_log_at_trx_commit = 1 # 1 = fsync при каждом COMMIT (надёжно)
                                   # 2 = сброс в OS cache (потеря до 1 сек)
                                   # 0 = раз в секунду (самый быстрый, небезопасно)
```

### Undo Log

> [!note] Undo Log
> Хранит предыдущие версии строк — нужен для MVCC (читатели видят старые версии) и для ROLLBACK транзакций. В MySQL 8.0 вынесен в отдельные undo tablespaces.

---

## 🔹 Жизненный цикл запроса

```
Client отправляет SQL
        │
        ▼
   [Connection Layer]
   Аутентификация, получить поток из thread pool
        │
        ▼
   [Parser]
   Лексический и синтаксический анализ
   SQL → Parse Tree
        │
        ▼
   [Preprocessor]
   Проверка прав, существования таблиц/колонок
        │
        ▼
   [Query Optimizer]
   Генерация вариантов плана
   Оценка стоимости (статистика из information_schema)
   Выбор оптимального плана
        │
        ▼
   [Query Executor]
   Вызов Storage Engine API
        │
        ▼
   [Storage Engine — InnoDB]
   Чтение/запись страниц через Buffer Pool
   Управление транзакциями, блокировками
        │
        ▼
   Результат → Client
```

> [!note] Query Cache (удалён в MySQL 8.0)
> До MySQL 8.0 существовал Query Cache — кэширование результатов SELECT. Удалён из-за проблем с производительностью при высокой конкурентности (глобальный мьютекс на запись).

---

## 🔹 Потоки и соединения

```ini
# my.cnf
max_connections = 151              # максимум соединений (default 151)
thread_cache_size = 9              # кэш потоков для переиспользования
thread_stack = 1M                  # стек каждого потока

# thread_handling = one-thread-per-connection (default)
# или thread_pool (MySQL Enterprise / Percona / MariaDB)
```

```sql
-- активные соединения
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW PROCESSLIST;                  -- текущие процессы

-- убить запрос/соединение
KILL QUERY  <id>;                  -- убить только запрос
KILL        <id>;                  -- убить соединение
```

> [!tip] Connection Pooling
> Как и в PostgreSQL, создание нового соединения к MySQL дорого. Используй пулы соединений на уровне приложения: **HikariCP** (Java), **SQLAlchemy pool** (Python), **ProxySQL** (прокси-пулер с балансировкой).

---

## 🔹 Файловая структура

```
/var/lib/mysql/                    ← datadir
├── myapp/                         ← папка БД
│   ├── users.ibd                  ← данные + индексы таблицы (file-per-table)
│   └── orders.ibd
├── ibdata1                        ← системный tablespace (undo log, data dictionary)
├── ib_logfile0                    ← redo log (файл 0)
├── ib_logfile1                    ← redo log (файл 1)
├── binlog.000001                  ← binary log (для репликации)
├── binlog.index
├── mysql.sock                     ← Unix socket
└── mysql/                         ← системная БД (привилегии, метаданные)
```

```ini
# my.cnf — важные пути
datadir         = /var/lib/mysql
socket          = /var/lib/mysql/mysql.sock
log_error       = /var/log/mysql/error.log
slow_query_log_file = /var/log/mysql/slow.log
general_log_file    = /var/log/mysql/general.log
```

---

## 🔹 Ключевые параметры конфигурации

```ini
# my.cnf / my.ini
[mysqld]

# ── Память ──────────────────────────────────────────────
innodb_buffer_pool_size     = 4G        # 50-80% RAM — самый важный параметр
innodb_log_file_size        = 512M      # больше = меньше checkpoint pressure
innodb_log_buffer_size      = 64M       # буфер redo log в памяти
key_buffer_size             = 32M       # кэш индексов MyISAM (если используется)
tmp_table_size              = 64M       # размер временных таблиц в памяти
max_heap_table_size         = 64M       # максимум для MEMORY таблиц
sort_buffer_size            = 4M        # буфер сортировки на соединение
join_buffer_size            = 4M        # буфер JOIN на соединение
read_buffer_size            = 2M        # буфер чтения для Seq Scan

# ── Соединения ───────────────────────────────────────────
max_connections             = 200
thread_cache_size           = 50
wait_timeout                = 28800     # таймаут idle соединения (сек)
interactive_timeout         = 28800

# ── InnoDB ──────────────────────────────────────────────
innodb_flush_log_at_trx_commit = 1      # 1 = надёжно, 2 = быстрее
innodb_flush_method         = O_DIRECT  # минуя OS cache (для Linux)
innodb_file_per_table       = ON        # каждая таблица = свой .ibd файл
innodb_io_capacity          = 200       # для HDD; для SSD → 2000+
innodb_read_io_threads      = 4
innodb_write_io_threads     = 4

# ── Кодировка ────────────────────────────────────────────
character_set_server        = utf8mb4   # ВСЕГДА utf8mb4, не utf8
collation_server            = utf8mb4_unicode_ci

# ── Логи ─────────────────────────────────────────────────
slow_query_log              = ON
slow_query_log_file         = /var/log/mysql/slow.log
long_query_time             = 1         # запросы дольше 1 секунды
log_queries_not_using_indexes = ON      # логировать запросы без индекса

general_log                 = OFF       # логировать ВСЕ запросы (только для отладки!)
```

> [!warning] utf8 vs utf8mb4
> `utf8` в MySQL — это 3-байтная кодировка, которая не поддерживает emoji и ряд Unicode символов. **Всегда используй `utf8mb4`** — полноценный UTF-8 (4 байта).

---

## 🔹 Системные таблицы и метаданные

```sql
-- информация о таблицах
SELECT table_name, engine, table_rows,
       ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE table_schema = 'myapp'
ORDER BY size_mb DESC;

-- информация о колонках
SELECT column_name, data_type, is_nullable, column_default, column_key
FROM information_schema.COLUMNS
WHERE table_schema = 'myapp' AND table_name = 'users';

-- информация об индексах
SELECT index_name, column_name, non_unique, index_type
FROM information_schema.STATISTICS
WHERE table_schema = 'myapp' AND table_name = 'users';

-- статус InnoDB
SHOW ENGINE INNODB STATUS\G

-- переменные сервера
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW GLOBAL VARIABLES LIKE '%timeout%';

-- статус сервера
SHOW GLOBAL STATUS LIKE 'Innodb_row%';
SHOW GLOBAL STATUS LIKE 'Com_%';       -- количество операций каждого типа
```

---

## 🔹 Итог

```
Архитектура:
  Connection Layer → SQL Layer → Storage Engine API → Disk

SQL Layer:
  Parser → Preprocessor → Optimizer → Executor

Storage Engines:
  InnoDB  — default, ACID, FK, row locks, MVCC → всегда используй
  MyISAM  — legacy, нет транзакций → избегай
  Memory  — временные таблицы

InnoDB память:
  Buffer Pool         — кэш страниц (50-80% RAM)
  Redo Log            — журнал для восстановления
  Undo Log            — MVCC и ROLLBACK

Ключевые параметры:
  innodb_buffer_pool_size  — главный параметр памяти
  innodb_flush_log_at_trx_commit = 1  — надёжность
  innodb_file_per_table = ON  — изоляция файлов
  character_set_server = utf8mb4  — кодировка

Мониторинг:
  SHOW PROCESSLIST           — активные запросы
  SHOW ENGINE INNODB STATUS  — состояние InnoDB
  information_schema.TABLES  — метаданные таблиц
```
