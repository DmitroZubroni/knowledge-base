# SQL DDL — Data Definition Language

> [!abstract] Связи
> [[00_SQL]] | [[SQL_DML]] | [[SQL_Joins]]

DDL — команды для определения и изменения **структуры** базы данных: таблицы, схемы, индексы, ограничения.

---

## 🔹 CREATE TABLE

```sql
CREATE TABLE users (
    id          SERIAL          PRIMARY KEY,
    username    VARCHAR(50)     NOT NULL UNIQUE,
    email       VARCHAR(100)    NOT NULL UNIQUE,
    age         INTEGER         CHECK (age >= 0),
    role        VARCHAR(20)     DEFAULT 'user',
    created_at  TIMESTAMP       DEFAULT CURRENT_TIMESTAMP
);
```

---

## 🔹 Типы данных

### Числовые

| Тип | Описание | Диапазон |
|-----|----------|----------|
| `SMALLINT` | целое малое | -32 768 .. 32 767 |
| `INTEGER` / `INT` | целое | -2.1B .. 2.1B |
| `BIGINT` | целое большое | -9.2E18 .. 9.2E18 |
| `SERIAL` | auto-increment INT | 1 .. 2.1B |
| `BIGSERIAL` | auto-increment BIGINT | 1 .. 9.2E18 |
| `DECIMAL(p,s)` / `NUMERIC` | точное дробное | до 131072 цифр |
| `REAL` | float 4 байта | ~6 знаков |
| `DOUBLE PRECISION` | float 8 байт | ~15 знаков |

### Строковые

| Тип | Описание |
|-----|----------|
| `CHAR(n)` | фиксированная длина, дополняется пробелами |
| `VARCHAR(n)` | переменная длина, макс n символов |
| `TEXT` | неограниченная строка |

### Дата и время

| Тип | Описание |
|-----|----------|
| `DATE` | только дата `2024-01-15` |
| `TIME` | только время `14:30:00` |
| `TIMESTAMP` | дата + время без TZ |
| `TIMESTAMPTZ` | дата + время с TZ (PostgreSQL) |
| `INTERVAL` | промежуток времени |

### Прочие

| Тип | Описание |
|-----|----------|
| `BOOLEAN` | `TRUE` / `FALSE` |
| `UUID` | уникальный идентификатор |
| `JSON` / `JSONB` | JSON-документ |
| `BYTEA` | бинарные данные |

---

## 🔹 Constraints (ограничения)

> [!note] Constraint
> Ограничение — правило целостности данных, которое БД проверяет автоматически при каждом INSERT/UPDATE.

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,                          -- уникальный + NOT NULL
    user_id     INTEGER NOT NULL,                            -- обязательное поле
    amount      DECIMAL(10,2) CHECK (amount > 0),            -- проверка значения
    status      VARCHAR(20) DEFAULT 'pending',               -- значение по умолчанию
    code        VARCHAR(50) UNIQUE,                          -- уникальность
    CONSTRAINT fk_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE               -- внешний ключ
);
```

| Constraint | Описание |
|------------|----------|
| `PRIMARY KEY` | уникальный идентификатор строки, NOT NULL + UNIQUE |
| `NOT NULL` | поле не может быть пустым |
| `UNIQUE` | значение уникально в колонке |
| `CHECK (expr)` | произвольное условие |
| `DEFAULT value` | значение по умолчанию |
| `FOREIGN KEY` | ссылка на строку в другой таблице |

### ON DELETE / ON UPDATE стратегии

| Стратегия | Поведение |
|-----------|-----------|
| `CASCADE` | удалить/обновить дочерние записи вместе с родителем |
| `SET NULL` | установить NULL в дочерней записи |
| `SET DEFAULT` | установить DEFAULT в дочерней записи |
| `RESTRICT` | запретить удаление/изменение при наличии дочерних |
| `NO ACTION` | как RESTRICT, но проверяется в конце транзакции |

> [!tip] Рекомендация
> `ON DELETE CASCADE` — для зависимых сущностей (order_items → orders).
> `ON DELETE RESTRICT` — для критичных ссылок (users → roles).

---

## 🔹 ALTER TABLE

```sql
-- добавить колонку
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- удалить колонку
ALTER TABLE users DROP COLUMN phone;

-- изменить тип колонки
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;

-- переименовать колонку
ALTER TABLE users RENAME COLUMN username TO login;

-- добавить constraint
ALTER TABLE users ADD CONSTRAINT chk_age CHECK (age >= 18);

-- удалить constraint
ALTER TABLE users DROP CONSTRAINT chk_age;

-- добавить NOT NULL
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- снять NOT NULL
ALTER TABLE users ALTER COLUMN phone DROP NOT NULL;

-- переименовать таблицу
ALTER TABLE users RENAME TO accounts;
```

---

## 🔹 DROP и TRUNCATE

```sql
-- удалить таблицу со всеми данными
DROP TABLE users;

-- удалить только если существует
DROP TABLE IF EXISTS users;

-- удалить таблицу + все зависимые объекты
DROP TABLE users CASCADE;

-- очистить таблицу (быстро, без логирования каждой строки)
TRUNCATE TABLE users;

-- очистить + сбросить счётчики SERIAL
TRUNCATE TABLE users RESTART IDENTITY;

-- очистить + каскадно
TRUNCATE TABLE users CASCADE;
```

> [!warning] DROP vs TRUNCATE vs DELETE
> - `DELETE` — удаляет строки по условию, логирует каждую, можно откатить
> - `TRUNCATE` — удаляет все строки сразу, быстрее, сбрасывает sequences
> - `DROP` — удаляет саму таблицу со структурой

---

## 🔹 Схемы (SCHEMA)

> [!note] Schema
> Схема — пространство имён внутри БД. Позволяет изолировать объекты (таблицы, функции) по модулям или командам.

```sql
-- создать схему
CREATE SCHEMA billing;

-- создать таблицу в схеме
CREATE TABLE billing.invoices (
    id SERIAL PRIMARY KEY,
    amount DECIMAL(10,2)
);

-- задать путь поиска схем
SET search_path TO billing, public;

-- удалить схему
DROP SCHEMA billing CASCADE;
```

---

## 🔹 Индексы (базовые)

> [!note] Index
> Индекс — вспомогательная структура данных для ускорения поиска. Ускоряет SELECT, замедляет INSERT/UPDATE/DELETE.

```sql
-- простой индекс
CREATE INDEX idx_users_email ON users(email);

-- уникальный индекс
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- составной индекс
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- удалить индекс
DROP INDEX idx_users_email;
```

> [!tip] Когда создавать индекс
> - колонка часто используется в `WHERE`, `JOIN`, `ORDER BY`
> - колонка имеет высокую кардинальность (много уникальных значений)
> - таблица большая (от ~10k строк)

> [!warning] Не индексируй всё подряд
> Каждый индекс занимает место на диске и замедляет запись. Подробнее — [[PG_Indexes]]

---

## 🔹 Итог

```
DDL команды:
  CREATE TABLE  — создать таблицу
  ALTER TABLE   — изменить структуру
  DROP TABLE    — удалить таблицу
  TRUNCATE      — очистить таблицу

Constraints:
  PRIMARY KEY   — уникальный идентификатор
  NOT NULL      — обязательное поле
  UNIQUE        — уникальность значения
  CHECK         — произвольное условие
  FOREIGN KEY   — ссылочная целостность
  DEFAULT       — значение по умолчанию

ON DELETE:
  CASCADE       — удалить дочерние
  SET NULL      — обнулить ссылку
  RESTRICT      — запретить удаление

Индексы:
  CREATE INDEX  — обычный индекс
  CREATE UNIQUE INDEX — уникальный
  DROP INDEX    — удалить
```
