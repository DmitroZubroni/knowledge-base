# Flyway (Schema Migrations)

> [!abstract] Связи
> [[main Tools]] | [[Spring_Data_JPA]] | [[Hibernate]] | [[00_PostgreSQL]]

---

## 🔹 Зачем миграции схемы

> [!warning] ddl-auto: update в production
> Hibernate `update` может добавить колонку, но **не удалит** лишнее, не откатит, не версионирует. Разные окружения расходятся.

**Schema as Code:**
- SQL в Git — как исходники приложения
- Воспроизводимый deploy: dev = staging = prod (по схеме)
- Code review миграций

---

## 🔹 Flyway vs Liquibase

| | Flyway | Liquibase |
|---|--------|-----------|
| Формат | SQL-first (`.sql`) | XML/YAML/JSON + SQL |
| Простота | ✅ Минимум абстракций | Больше возможностей |
| Откат | Undo (Teams) / compensating | rollback changesets |
| Когда Flyway | Spring Boot default, SQL-команда | Сложные conditional changes |

---

## 🔹 Структура миграций

```
src/main/resources/db/migration/
    V1__create_users.sql
    V1_1__add_index_on_email.sql
    V2__create_orders.sql
    R__refresh_report_view.sql      ← repeatable
```

| Префикс | Поведение |
|---------|-----------|
| `V{version}__{desc}.sql` | Версионная, один раз |
| `R__{desc}.sql` | Repeatable — переприменяется при изменении checksum |
| `U{version}__` | Undo (Flyway Teams) |

> [!warning] Нельзя менять применённую миграцию
> Flyway хранит checksum — изменение `V1` после deploy сломает migrate.

---

## 🔹 Версионирование

```
V1__init.sql
V2__add_orders.sql
V2_1__add_index.sql   ← допустимо между V2 и V3
V3__add_payments.sql
```

**Семантика:** версии сравниваются лексикографически с учётом разделителей.

---

## 🔹 Конфигурация в Spring Boot

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true   # БД уже с таблицами — пометить baseline
    schemas: public
    clean-disabled: true        # ❌ flyway clean на prod
```

```gradle
implementation 'org.flywaydb:flyway-core'
// + flyway-database-postgresql для PG 15+
```

> [!tip] Порядок старта
> Flyway migrate **до** Hibernate `validate` — схема актуальна перед JPA.

---

## 🔹 Таблица flyway_schema_history

| Колонка | Смысл |
|---------|-------|
| `installed_rank` | Порядок применения |
| `version` | V1, V2, ... |
| `description` | из имени файла |
| `script` | имя файла |
| `checksum` | хеш содержимого |
| `success` | применена ли |
| `installed_on` | timestamp |

---

## 🔹 Типичные ситуации

**Добавить колонку:**

```sql
-- V3__add_phone_to_users.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- nullable OK для существующих строк
```

**NOT NULL на существующей таблице (PostgreSQL):**

```sql
-- V4a — добавить nullable
ALTER TABLE users ADD COLUMN tier VARCHAR(20);
UPDATE users SET tier = 'FREE' WHERE tier IS NULL;
-- V4b — constraint
ALTER TABLE users ALTER COLUMN tier SET NOT NULL;
```

**Переименовать колонку (без downtime):**

```
V5__add_email_new.sql     → ADD email_new
V6__copy_data.sql         → UPDATE ... SET email_new = email
V7__drop_email_old.sql    → DROP email (после deploy кода)
```

> [!warning] ALTER COLUMN NOT NULL без DEFAULT на большой таблице PG
> Полная блокировка таблицы — делать в несколько шагов.

**Rollback:** compensating migration `V8__revert_v7.sql`, не править V7.

---

## 🔹 Flyway в тестах

```yaml
# application-test.yml
spring:
  flyway:
    clean-disabled: false  # только тесты!
  jpa:
    hibernate:
      ddl-auto: validate
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@Testcontainers
class OrderRepositoryIT {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    // Flyway применяется автоматически при старте контекста
}
```

---

## 🔹 Итог

1. Миграции — единственный способ менять prod-схему.
2. `V{version}__{desc}.sql` — неизменяемые после apply.
3. Flyway + `ddl-auto: validate` в Spring Boot.
4. Большие ALTER на PG — поэтапно, без блокирующего NOT NULL сразу.
5. Testcontainers + Flyway = prod-like тесты.

```
Шпаргалка Flyway:
─────────────────────────────────────────
db/migration/V1__desc.sql        → versioned
R__view.sql                      → repeatable
❌ edit applied migration         → checksum fail
baseline-on-migrate              → existing DB
flyway_schema_history            → audit trail
ddl-auto: validate + Flyway      → prod pattern
compensating V{n+1}__revert      → rollback strategy
```
