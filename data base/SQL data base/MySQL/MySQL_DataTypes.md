# MySQL DataTypes — Типы данных MySQL

> **Теги:** #mysql #database #datatypes #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[MySQL]]

---

## 🔹 Числовые типы

| Тип | Размер | Диапазон (signed) | Диапазон (unsigned) |
|-----|--------|------------------|---------------------|
| `TINYINT` | 1 байт | -128 .. 127 | 0 .. 255 |
| `SMALLINT` | 2 байта | -32 768 .. 32 767 | 0 .. 65 535 |
| `MEDIUMINT` | 3 байта | -8.4M .. 8.4M | 0 .. 16.7M |
| `INT` / `INTEGER` | 4 байта | -2.1B .. 2.1B | 0 .. 4.3B |
| `BIGINT` | 8 байт | ±9.2×10¹⁸ | 0 .. 1.8×10¹⁹ |
| `DECIMAL(p,s)` | переменный | точный | точный |
| `FLOAT` | 4 байта | ~7 знаков | неточный |
| `DOUBLE` | 8 байт | ~15 знаков | неточный |
| `BIT(n)` | 1-8 байт | 1-64 бита | — |

```sql
-- UNSIGNED — только положительные числа
age     TINYINT UNSIGNED           -- 0..255
counter BIGINT UNSIGNED            -- 0..18.4E18

-- ZEROFILL — дополнять нулями слева (устарело в 8.0)
code    INT(6) ZEROFILL            -- 000042

-- AUTO_INCREMENT
id      INT NOT NULL AUTO_INCREMENT PRIMARY KEY
id      BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
```

> [!warning] FLOAT / DOUBLE для денег
> `FLOAT` и `DOUBLE` — неточные (IEEE 754). `0.1 + 0.2 ≠ 0.3`. Для финансов используй `DECIMAL(p,s)` или храни в копейках как `BIGINT`.

```sql
-- деньги: правильно
amount  DECIMAL(12, 2)             -- до 9 999 999 999.99
amount  BIGINT                     -- в копейках: 9999 = 99.99 руб

-- деньги: неправильно
amount  DOUBLE                     -- потеря точности
amount  FLOAT                      -- ещё хуже
```

> [!note] AUTO_INCREMENT vs GENERATED
> В MySQL нет `GENERATED AS IDENTITY` (PostgreSQL). Используй `AUTO_INCREMENT`. В MySQL 8.0 значение AUTO_INCREMENT сохраняется в redo log — не сбрасывается при рестарте (в отличие от старых версий).

---

## 🔹 Строковые типы

| Тип | Макс. размер | Хранение | Применение |
|-----|-------------|---------|------------|
| `CHAR(n)` | 255 символов | фиксированное, дополняется пробелами | коды, хэши |
| `VARCHAR(n)` | 65 535 байт | переменное + 1-2 байта длины | имена, email |
| `TINYTEXT` | 255 байт | вне строки | очень короткий текст |
| `TEXT` | 65 535 байт | вне строки | описания |
| `MEDIUMTEXT` | 16 MB | вне строки | статьи, HTML |
| `LONGTEXT` | 4 GB | вне строки | большие тексты |
| `BINARY(n)` | 255 байт | фиксированное | бинарные данные |
| `VARBINARY(n)` | 65 535 байт | переменное | бинарные данные |
| `TINYBLOB` | 255 байт | вне строки | маленькие файлы |
| `BLOB` | 65 535 байт | вне строки | файлы |
| `MEDIUMBLOB` | 16 MB | вне строки | изображения |
| `LONGBLOB` | 4 GB | вне строки | большие файлы |

```sql
-- VARCHAR vs TEXT
name        VARCHAR(255)            -- хранится в строке, индексируется напрямую
description TEXT                    -- хранится отдельно, нужен prefix-индекс

-- CHAR vs VARCHAR
status      CHAR(10)                -- 'active    ' (дополнено пробелами)
status      VARCHAR(10)             -- 'active' (только нужные байты)
```

> [!tip] VARCHAR vs TEXT в MySQL
> В отличие от PostgreSQL, в MySQL `TEXT` и `VARCHAR` хранятся **по-разному**. `VARCHAR` хранится в строке таблицы (быстрее), `TEXT` — отдельно (медленнее для коротких значений). Для полей до 65535 байт предпочитай `VARCHAR`.

### Кодировка и сортировка

```sql
-- задать кодировку на уровне таблицы
CREATE TABLE users (
    name VARCHAR(100)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- задать на уровне колонки
name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

-- популярные collation
utf8mb4_unicode_ci   -- регистронезависимое сравнение (CI = case insensitive)
utf8mb4_bin          -- побайтовое сравнение, регистрозависимое
utf8mb4_general_ci   -- быстрее unicode_ci, но менее точное

-- проверить кодировку таблицы
SHOW CREATE TABLE users\G
```

---

## 🔹 Дата и время

| Тип | Размер | Диапазон | Точность |
|-----|--------|----------|---------|
| `DATE` | 3 байта | 1000-01-01 .. 9999-12-31 | день |
| `TIME` | 3 байта | -838:59:59 .. 838:59:59 | секунда |
| `DATETIME` | 5 байт | 1000-01-01 .. 9999-12-31 | микросекунда (6) |
| `TIMESTAMP` | 4 байта | 1970-01-01 .. 2038-01-19 | микросекунда (6) |
| `YEAR` | 1 байт | 1901 .. 2155 | год |

```sql
-- DATETIME vs TIMESTAMP
created_at DATETIME(6)              -- хранит как есть, нет TZ, до 9999 года
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

-- TIMESTAMP автоматически конвертирует в UTC при записи
-- и обратно в TZ сессии при чтении
-- Проблема 2038: TIMESTAMP переполнится 19 января 2038!
-- Для новых проектов предпочти DATETIME

-- дробные секунды (MySQL 5.6.4+)
created_at DATETIME(3)              -- миллисекунды
created_at DATETIME(6)              -- микросекунды
```

> [!warning] Проблема 2038 года
> `TIMESTAMP` хранит 4-байтный Unix timestamp — переполнится 19 января 2038. Для новых проектов используй `DATETIME` или `BIGINT` (Unix timestamp в миллисекундах).

```sql
-- полезные функции дат
NOW()                               -- текущая дата + время
CURDATE() / CURRENT_DATE            -- только дата
CURTIME() / CURRENT_TIME            -- только время
UNIX_TIMESTAMP()                    -- Unix timestamp
FROM_UNIXTIME(ts)                   -- из Unix timestamp
DATE_FORMAT(NOW(), '%d.%m.%Y')     -- форматирование
STR_TO_DATE('15.01.2024', '%d.%m.%Y') -- парсинг строки
DATEDIFF(date1, date2)              -- разница в днях
DATE_ADD(NOW(), INTERVAL 7 DAY)     -- прибавить интервал
DATE_SUB(NOW(), INTERVAL 1 MONTH)   -- вычесть интервал
EXTRACT(YEAR FROM NOW())            -- извлечь часть
YEAR(NOW()) / MONTH(NOW()) / DAY(NOW()) -- то же самое
TIMESTAMPDIFF(MINUTE, start, end)   -- разница в минутах
```

---

## 🔹 ENUM и SET

```sql
-- ENUM — одно значение из списка
status  ENUM('pending', 'paid', 'shipped', 'cancelled') DEFAULT 'pending'

-- SET — несколько значений из списка (битовая маска)
permissions SET('read', 'write', 'delete', 'admin')

-- использование
INSERT INTO users (status) VALUES ('active');
INSERT INTO articles (permissions) VALUES ('read,write');

SELECT * FROM users WHERE status = 'active';
SELECT * FROM articles WHERE FIND_IN_SET('write', permissions);
```

| | ENUM | SET |
|--|------|-----|
| Значений | 1 | несколько (до 64) |
| Хранение | 1-2 байта | 1-8 байт |
| Поиск | = оператор | FIND_IN_SET() |

> [!warning] ENUM — осторожно
> Добавление нового значения в ENUM требует `ALTER TABLE` — на больших таблицах это медленно (в MySQL 5.x блокирует таблицу). В MySQL 8.0 добавление в конец списка — быстрая операция. Для гибкости рассмотри `VARCHAR` + CHECK constraint или справочную таблицу.

---

## 🔹 JSON (MySQL 5.7+)

```sql
-- создание
CREATE TABLE events (
    id      INT AUTO_INCREMENT PRIMARY KEY,
    payload JSON NOT NULL
);

INSERT INTO events (payload) VALUES
('{"user_id": 1, "action": "login", "tags": ["web", "mobile"]}');

-- доступ к данным
SELECT payload -> '$.user_id'           FROM events;  -- как JSON: 1
SELECT payload ->> '$.action'           FROM events;  -- как строка: login
SELECT payload -> '$.tags[0]'           FROM events;  -- первый элемент массива
SELECT JSON_EXTRACT(payload, '$.action') FROM events; -- явный вызов

-- проверки
SELECT * FROM events WHERE JSON_CONTAINS(payload, '1', '$.user_id');
SELECT * FROM events WHERE payload ->> '$.action' = 'login';

-- изменение
UPDATE events SET payload = JSON_SET(payload, '$.processed', true) WHERE id = 1;
UPDATE events SET payload = JSON_INSERT(payload, '$.new_field', 'value');
UPDATE events SET payload = JSON_REPLACE(payload, '$.action', 'logout');
UPDATE events SET payload = JSON_REMOVE(payload, '$.temp');

-- агрегация
SELECT JSON_ARRAYAGG(username) FROM users;
SELECT JSON_OBJECTAGG(id, username) FROM users;

-- создание JSON
SELECT JSON_OBJECT('id', id, 'name', username) FROM users;
SELECT JSON_ARRAY(1, 2, 3);
```

### Индекс на JSON поле (через виртуальную колонку)

```sql
-- MySQL не поддерживает прямой индекс на JSON
-- Решение: виртуальная колонка + индекс

ALTER TABLE events
    ADD COLUMN action VARCHAR(50) GENERATED ALWAYS AS (payload ->> '$.action') VIRTUAL;

CREATE INDEX idx_events_action ON events(action);

-- теперь работает с индексом:
SELECT * FROM events WHERE action = 'login';
```

> [!note] JSON vs JSONB
> MySQL хранит JSON как текст с валидацией — нет бинарного формата как в PostgreSQL JSONB. Прямой GIN-индекс на JSON-поле невозможен — нужны виртуальные колонки.

---

## 🔹 Виртуальные и хранимые колонки

```sql
-- VIRTUAL — вычисляется при чтении, не хранится на диске
-- STORED  — вычисляется при INSERT/UPDATE, хранится на диске

CREATE TABLE users (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    first_name  VARCHAR(50),
    last_name   VARCHAR(50),

    -- виртуальная (вычисляется при SELECT)
    full_name   VARCHAR(101) GENERATED ALWAYS AS
                    (CONCAT(first_name, ' ', last_name)) VIRTUAL,

    -- хранимая (вычисляется при записи, можно индексировать)
    email_lower VARCHAR(100) GENERATED ALWAYS AS
                    (LOWER(email)) STORED,

    email       VARCHAR(100)
);

-- индекс на хранимую колонку
CREATE INDEX idx_users_email_lower ON users(email_lower);

-- теперь работает без учёта регистра:
SELECT * FROM users WHERE email_lower = 'test@mail.com';
```

---

## 🔹 Прочие типы

```sql
-- BOOLEAN (алиас для TINYINT(1))
is_active   BOOLEAN DEFAULT TRUE     -- хранится как 0/1
is_active   TINYINT(1) DEFAULT 1     -- то же самое

-- TRUE/FALSE или 1/0
INSERT INTO users (is_active) VALUES (TRUE);
SELECT * FROM users WHERE is_active = TRUE;
SELECT * FROM users WHERE is_active;   -- короткая форма

-- UUID (нет встроенного типа)
id          CHAR(36)                  -- хранить как строку: '550e8400-...'
id          BINARY(16)                -- компактнее: UUID_TO_BIN(uuid)

INSERT INTO users (id) VALUES (UUID());          -- генерация
INSERT INTO users (id) VALUES (UUID_TO_BIN(UUID())); -- бинарный формат

SELECT BIN_TO_UUID(id) FROM users;   -- обратное преобразование

-- GEOMETRY (пространственные данные)
location    POINT
area        POLYGON
path        LINESTRING

INSERT INTO places (location) VALUES (ST_GeomFromText('POINT(37.6 55.7)'));
SELECT ST_Distance(location, ST_GeomFromText('POINT(37.6 55.7)')) FROM places;
```

---

## 🔹 Выбор типа — шпаргалка

```
ID записи:           INT / BIGINT AUTO_INCREMENT
                     или CHAR(36) для UUID
Имя, описание:       VARCHAR(n)
Длинный текст:       TEXT / MEDIUMTEXT
Деньги:              DECIMAL(12,2)  или BIGINT (копейки)
Дата + время:        DATETIME(6)  — предпочтительнее TIMESTAMP (нет 2038)
Только дата:         DATE
Флаг:                TINYINT(1) / BOOLEAN
Статус (небольшой):  ENUM(...)
Бинарные данные:     BLOB / VARBINARY
JSON:                JSON + виртуальная колонка для индекса
Сетевой адрес:       VARCHAR(45)  или INT UNSIGNED (IPv4)
Кодировка:           utf8mb4  — всегда
```

---

## 🔹 Итог

```
Числа:
  INT / BIGINT — целые числа
  DECIMAL(p,s) — точные дробные (деньги)
  FLOAT/DOUBLE — неточные, не для денег
  UNSIGNED     — только положительные

Строки:
  VARCHAR(n)   — до 65535 байт, хранится в строке
  TEXT         — отдельное хранение, нужен prefix-индекс
  CHAR(n)      — фиксированная длина, быстрее для коротких
  utf8mb4      — единственно верная кодировка

Дата и время:
  DATETIME(6)  — предпочтительнее, нет 2038 проблемы
  TIMESTAMP    — автоконвертация UTC, до 2038 года
  DATE / TIME  — отдельно дата или время

Специальные:
  ENUM         — фиксированный список (ALTER TABLE для добавления)
  JSON         — хранится как текст, индекс через виртуальную колонку
  BOOLEAN      — алиас TINYINT(1)
  UUID()       — генерация, хранить в CHAR(36) или BINARY(16)

Виртуальные колонки:
  VIRTUAL  — вычисляется при чтении
  STORED   — хранится, можно индексировать
```
