# MySQL Advanced — Репликация, Партиционирование, Процедуры

> **Теги:** #mysql #database #advanced #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[MySQL]]

---

## 🔹 Репликация

> [!note] Репликация MySQL
> Асинхронная синхронизация данных между серверами через **Binary Log (binlog)**. Мастер пишет изменения в binlog, реплика читает и применяет их.

### Архитектура

```
Master (Source)                     Replica
    │                                   │
    ├── записывает в Binlog ────────►   ├── IO Thread: читает binlog
    │   (binlog.000001, 000002...)      │   → пишет в Relay Log
    │                                   ├── SQL Thread: применяет
    │                                   │   события из Relay Log
    │◄──────── async replication ──────►│   к данным реплики
    │                                   │
    │  read + write                     │  read only (обычно)
```

### Настройка Master

```ini
# my.cnf на MASTER
[mysqld]
server-id           = 1               # уникальный ID сервера
log_bin             = /var/log/mysql/binlog
binlog_format       = ROW             # ROW / STATEMENT / MIXED
binlog_row_image    = FULL            # FULL / MINIMAL / NOBLOB
expire_logs_days    = 7               # хранить binlog 7 дней
max_binlog_size     = 100M
```

```sql
-- создать пользователя для репликации
CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'StrongPass123!';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;

-- посмотреть текущую позицию binlog
SHOW MASTER STATUS\G
-- File: binlog.000003, Position: 154
```

### Настройка Replica

```ini
# my.cnf на REPLICA
[mysqld]
server-id           = 2
relay_log           = /var/log/mysql/relay-bin
read_only           = ON              # запретить запись на реплику
super_read_only     = ON              # запретить даже SUPER пользователям
log_slave_updates   = ON              # реплика тоже пишет в binlog (для каскада)
```

```sql
-- настроить репликацию (MySQL 5.7)
CHANGE MASTER TO
    MASTER_HOST     = '192.168.1.10',
    MASTER_USER     = 'replicator',
    MASTER_PASSWORD = 'StrongPass123!',
    MASTER_LOG_FILE = 'binlog.000003',
    MASTER_LOG_POS  = 154;

-- MySQL 8.0+ (новый синтаксис)
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST     = '192.168.1.10',
    SOURCE_USER     = 'replicator',
    SOURCE_PASSWORD = 'StrongPass123!',
    SOURCE_LOG_FILE = 'binlog.000003',
    SOURCE_LOG_POS  = 154;

START REPLICA;                        -- START SLAVE в 5.7
SHOW REPLICA STATUS\G                 -- SHOW SLAVE STATUS в 5.7

-- ключевые поля в SHOW REPLICA STATUS:
-- Replica_IO_Running:  Yes    ← IO Thread работает
-- Replica_SQL_Running: Yes    ← SQL Thread работает
-- Seconds_Behind_Source: 0   ← отставание в секундах (0 = без лага)
-- Last_Error:                ← ошибки (пусто = всё хорошо)
```

### GTID репликация (Global Transaction ID)

> [!note] GTID
> Global Transaction Identifier — уникальный ID каждой транзакции в формате `server_uuid:transaction_id`. Упрощает failover и мониторинг.

```ini
# my.cnf (на обоих серверах)
gtid_mode           = ON
enforce_gtid_consistency = ON
log_slave_updates   = ON
```

```sql
-- при GTID не нужно указывать файл и позицию
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST     = '192.168.1.10',
    SOURCE_USER     = 'replicator',
    SOURCE_PASSWORD = 'StrongPass123!',
    SOURCE_AUTO_POSITION = 1;          -- автоматически по GTID

-- посмотреть выполненные транзакции
SELECT @@GLOBAL.gtid_executed;
```

### Форматы Binlog

| Формат | Что записывает | Плюсы | Минусы |
|--------|---------------|-------|--------|
| `STATEMENT` | SQL команды | компактный | недетерминированные функции (NOW(), RAND()) |
| `ROW` | изменения строк | точный, безопасный | больше объём |
| `MIXED` | STATEMENT + ROW | баланс | сложнее анализировать |

> [!tip] Рекомендуется ROW
> ROW формат безопаснее — нет проблем с недетерминированными функциями, хранимыми процедурами, триггерами. Используй `binlog_row_image = MINIMAL` для уменьшения объёма.

### Мониторинг репликации

```sql
-- состояние реплики
SHOW REPLICA STATUS\G

-- лаг репликации
SELECT TIMESTAMPDIFF(SECOND,
    MIN(last_queued_transaction_start_queue_timestamp),
    NOW()
) AS replication_lag_sec
FROM performance_schema.replication_applier_status_by_worker;

-- бинлог файлы на мастере
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'binlog.000003' LIMIT 20;
```

---

## 🔹 Партиционирование

> [!note] Партиционирование
> Разбиение таблицы на логические части (партиции) с раздельным физическим хранением. Ускоряет запросы по ключу партиционирования и упрощает управление данными.

### RANGE партиционирование

```sql
-- по диапазону числовых значений
CREATE TABLE orders (
    id         INT NOT NULL,
    user_id    INT,
    amount     DECIMAL(10,2),
    created_at DATE NOT NULL
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- по диапазону дат (RANGE COLUMNS)
CREATE TABLE logs (
    id         BIGINT NOT NULL AUTO_INCREMENT,
    message    TEXT,
    created_at DATE NOT NULL,
    PRIMARY KEY (id, created_at)             -- PK должен включать ключ партиции
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p2023_q1 VALUES LESS THAN ('2023-04-01'),
    PARTITION p2023_q2 VALUES LESS THAN ('2023-07-01'),
    PARTITION p2023_q3 VALUES LESS THAN ('2023-10-01'),
    PARTITION p2023_q4 VALUES LESS THAN ('2024-01-01'),
    PARTITION p2024_q1 VALUES LESS THAN ('2024-04-01')
);
```

### LIST партиционирование

```sql
-- по списку значений
CREATE TABLE users (
    id     INT NOT NULL,
    region VARCHAR(10) NOT NULL,
    name   VARCHAR(100)
)
PARTITION BY LIST COLUMNS (region) (
    PARTITION p_eu VALUES IN ('DE', 'FR', 'IT', 'ES', 'PL'),
    PARTITION p_us VALUES IN ('US', 'CA', 'MX'),
    PARTITION p_asia VALUES IN ('CN', 'JP', 'KR', 'IN'),
    PARTITION p_other VALUES IN ('OTHER')
);
```

### HASH партиционирование

```sql
-- равномерное распределение по хэшу
CREATE TABLE events (
    id      INT NOT NULL,
    payload JSON
)
PARTITION BY HASH(id)
PARTITIONS 8;                          -- 8 партиций

-- LINEAR HASH — быстрее ADD/DROP PARTITION, менее равномерно
PARTITION BY LINEAR HASH(id)
PARTITIONS 8;
```

### KEY партиционирование

```sql
-- как HASH, но MySQL сам вычисляет хэш-функцию
CREATE TABLE sessions (
    session_id CHAR(32) NOT NULL,
    user_id    INT,
    data       JSON
)
PARTITION BY KEY(session_id)
PARTITIONS 16;
```

### Управление партициями

```sql
-- посмотреть партиции
SELECT partition_name, partition_method, partition_expression,
       table_rows, data_length / 1024 / 1024 AS data_mb
FROM information_schema.PARTITIONS
WHERE table_schema = 'myapp' AND table_name = 'orders';

-- добавить партицию (для RANGE/LIST)
ALTER TABLE orders ADD PARTITION (
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- удалить партицию (мгновенно! вместе с данными)
ALTER TABLE orders DROP PARTITION p2022;

-- очистить партицию (удалить данные, сохранить структуру)
ALTER TABLE orders TRUNCATE PARTITION p2022;

-- реорганизовать партицию (разбить или объединить)
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- принудительно использовать партицию в запросе
SELECT * FROM orders PARTITION (p2024) WHERE user_id = 5;
```

> [!tip] Partition Pruning
> MySQL автоматически пропускает нерелевантные партиции. Убедись что WHERE использует ключ партиционирования:
> ```sql
> EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024;
> -- partitions: p2024 ← только одна партиция!
> ```

> [!warning] Ограничения партиционирования
> - Максимум 8192 партиций на таблицу
> - PRIMARY KEY и UNIQUE KEY должны включать ключ партиционирования
> - Foreign Keys не поддерживаются в партиционированных таблицах
> - Полнотекстовые индексы не поддерживаются

---

## 🔹 Хранимые процедуры и функции

### Хранимая процедура

```sql
DELIMITER $$

CREATE PROCEDURE transfer_money(
    IN  from_id  INT,
    IN  to_id    INT,
    IN  amount   DECIMAL(12,2),
    OUT result   VARCHAR(50)
)
BEGIN
    DECLARE from_balance DECIMAL(12,2);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET result = 'ERROR: Transaction failed';
    END;

    START TRANSACTION;

    -- проверить баланс
    SELECT balance INTO from_balance FROM accounts WHERE id = from_id FOR UPDATE;

    IF from_balance < amount THEN
        ROLLBACK;
        SET result = 'ERROR: Insufficient funds';
    ELSE
        UPDATE accounts SET balance = balance - amount WHERE id = from_id;
        UPDATE accounts SET balance = balance + amount WHERE id = to_id;
        COMMIT;
        SET result = 'OK';
    END IF;
END$$

DELIMITER ;

-- вызов
CALL transfer_money(1, 2, 500.00, @result);
SELECT @result;
```

### Хранимая функция

```sql
DELIMITER $$

CREATE FUNCTION get_user_total(user_id_param INT)
RETURNS DECIMAL(12,2)
READS SQL DATA
DETERMINISTIC
BEGIN
    DECLARE total DECIMAL(12,2) DEFAULT 0;

    SELECT COALESCE(SUM(amount), 0) INTO total
    FROM orders
    WHERE user_id = user_id_param AND status = 'paid';

    RETURN total;
END$$

DELIMITER ;

-- использование в запросах
SELECT username, get_user_total(id) AS total FROM users;
```

### Управление процедурами

```sql
SHOW PROCEDURE STATUS WHERE Db = 'myapp';
SHOW CREATE PROCEDURE transfer_money\G
DROP PROCEDURE IF EXISTS transfer_money;

SHOW FUNCTION STATUS WHERE Db = 'myapp';
DROP FUNCTION IF EXISTS get_user_total;
```

---

## 🔹 Триггеры

```sql
DELIMITER $$

-- триггер: автоматически обновлять updated_at
CREATE TRIGGER trg_users_before_update
    BEFORE UPDATE ON users
    FOR EACH ROW
BEGIN
    SET NEW.updated_at = NOW();
END$$

-- триггер: аудит изменений
CREATE TABLE audit_log (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(50),
    operation  ENUM('INSERT', 'UPDATE', 'DELETE'),
    old_data   JSON,
    new_data   JSON,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by VARCHAR(100) DEFAULT (USER())
);

CREATE TRIGGER trg_orders_after_update
    AFTER UPDATE ON orders
    FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data)
    VALUES (
        'orders',
        'UPDATE',
        JSON_OBJECT('id', OLD.id, 'status', OLD.status, 'amount', OLD.amount),
        JSON_OBJECT('id', NEW.id, 'status', NEW.status, 'amount', NEW.amount)
    );
END$$

DELIMITER ;

-- посмотреть триггеры
SHOW TRIGGERS FROM myapp\G
DROP TRIGGER IF EXISTS trg_users_before_update;
```

> [!warning] Осторожно с триггерами
> Триггеры выполняются неявно, их сложно отлаживать. Они могут замедлить запись при большой нагрузке. Для аудита рассмотри альтернативы: CDC (Change Data Capture) через binlog.

---

## 🔹 Представления (Views)

```sql
-- создать представление
CREATE OR REPLACE VIEW v_active_users AS
SELECT id, username, email, created_at
FROM users
WHERE status = 'active' AND deleted_at IS NULL;

-- использование
SELECT * FROM v_active_users WHERE username LIKE 'al%';

-- updatable view (можно INSERT/UPDATE/DELETE если нет GROUP BY, DISTINCT и т.д.)
UPDATE v_active_users SET email = 'new@mail.com' WHERE id = 1;

-- посмотреть представления
SHOW FULL TABLES WHERE Table_type = 'VIEW';
SHOW CREATE VIEW v_active_users\G

DROP VIEW IF EXISTS v_active_users;
```

---

## 🔹 События (Events / Scheduler)

```sql
-- включить планировщик событий
SET GLOBAL event_scheduler = ON;

-- создать повторяющееся событие
CREATE EVENT evt_cleanup_old_logs
    ON SCHEDULE EVERY 1 DAY
    STARTS CURRENT_TIMESTAMP
    DO
    DELETE FROM logs WHERE created_at < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- разовое событие
CREATE EVENT evt_send_report
    ON SCHEDULE AT '2024-12-31 23:59:00'
    DO
    CALL generate_year_report();

-- посмотреть события
SHOW EVENTS FROM myapp\G

-- включить/выключить событие
ALTER EVENT evt_cleanup_old_logs DISABLE;
ALTER EVENT evt_cleanup_old_logs ENABLE;

DROP EVENT IF EXISTS evt_cleanup_old_logs;
```

---

## 🔹 Full-text Search — расширенно

```sql
-- настройка минимальной длины слова
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';  -- default 3

-- изменить (требует перестройки индекса)
SET GLOBAL innodb_ft_min_token_size = 2;
OPTIMIZE TABLE articles;               -- пересобрать full-text индекс

-- стоп-слова (слова, которые не индексируются)
SELECT * FROM information_schema.INNODB_FT_DEFAULT_STOPWORD;

-- отключить стоп-слова
SET GLOBAL innodb_ft_enable_stopword = OFF;

-- кастомные стоп-слова
CREATE TABLE my_stopwords (value VARCHAR(30));
INSERT INTO my_stopwords VALUES ('и'), ('в'), ('на'), ('с');
SET GLOBAL innodb_ft_server_stopword_table = 'myapp/my_stopwords';
```

---

## 🔹 Полезные инструменты

```bash
# Percona Toolkit — набор утилит для MySQL
pt-query-digest /var/log/mysql/slow.log     # анализ slow query log
pt-online-schema-change --alter "ADD INDEX idx_email(email)" D=myapp,t=users
pt-table-checksum                            # проверка консистентности реплики
pt-table-sync                                # синхронизация реплики с мастером

# gh-ost (GitHub) — онлайн ALTER TABLE без блокировки
gh-ost --user=root --password=pass --host=localhost \
       --database=myapp --table=users \
       --alter="ADD INDEX idx_email(email)" \
       --execute

# mydumper / myloader — быстрый параллельный дамп
mydumper -u root -p pass -B myapp -o /backup/myapp
myloader -u root -p pass -B myapp -d /backup/myapp

# mysqlcheck — проверка и восстановление таблиц
mysqlcheck -u root -p --all-databases --auto-repair
```

---

## 🔹 Итог

```
Репликация:
  Binlog-based (асинхронная по умолчанию)
  IO Thread → Relay Log → SQL Thread
  Форматы: ROW (рекомендуется), STATEMENT, MIXED
  GTID — упрощает failover и мониторинг
  SHOW REPLICA STATUS → Seconds_Behind_Source

Партиционирование:
  RANGE  — по диапазону (даты, числа)
  LIST   — по списку значений
  HASH   — равномерное хэш-распределение
  KEY    — как HASH, MySQL вычисляет функцию
  PK должен включать ключ партиции
  DROP PARTITION — мгновенное удаление данных
  FK не поддерживаются

Хранимые процедуры:
  CALL procedure_name(args)
  IN / OUT / INOUT параметры
  DECLARE переменные, handlers
  DDL внутри → неявный COMMIT

Триггеры:
  BEFORE / AFTER + INSERT / UPDATE / DELETE
  OLD.col / NEW.col — старые и новые значения
  Используй осторожно — скрытая логика

Events (Scheduler):
  SET GLOBAL event_scheduler = ON
  ON SCHEDULE EVERY N DAY/HOUR/MINUTE
  Замена cron для задач на уровне БД

Инструменты:
  pt-query-digest      — анализ slow log
  pt-online-schema-change / gh-ost — ALTER без блокировки
  mydumper / myloader  — быстрый дамп
```
