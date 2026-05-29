# 00 — PostgreSQL Hub

> **Теги:** #postgresql #hub #database  

> [!abstract] Навигация
> [[main]] | [[main SQL DB]] | [[main SQL DB]]

## 🔹 Структура раздела

| Файл | Содержание |
|------|------------|
| [[PG_Architecture]] | Архитектура СУБД, процессы, память, WAL |
| [[PG_DataTypes]] | Типы данных: JSONB, UUID, Array, ENUM, Range |
| [[PG_Indexes]] | B-tree, Hash, GIN, GiST, BRIN, partial, covering |
| [[PG_Transactions]] | ACID, MVCC, уровни изоляции, блокировки |
| [[PG_Performance]] | EXPLAIN ANALYZE, VACUUM, статистика, оптимизация |
| [[PG_Advanced]] | Партиционирование, репликация, расширения |

---

## 🔹 Порядок изучения

[[PG_Architecture]] ↓
[[PG_DataTypes]] ↓
[[PG_Indexes]] ↓
[[PG_Transactions]] ↓
[[PG_Performance]] ↓
[[PG_Advanced]]

---

## 🔹 Что такое PostgreSQL

```
PostgreSQL — объектно-реляционная СУБД с открытым исходным кодом.
Версия 1.0 — 1995. Текущая стабильная — PostgreSQL 16+.

Ключевые особенности:
  - Полное соответствие ACID
  - MVCC — многоверсионный контроль параллелизма
  - Расширяемость: custom types, functions, extensions
  - Богатые типы: JSONB, Array, UUID, Range, геометрия
  - Партиционирование, физическая и логическая репликация
  - Расширения: PostGIS, pg_vector, TimescaleDB и др.
```

---

## 🔹 Быстрый старт

```sql
-- создать БД и подключиться
CREATE DATABASE myapp;
\c myapp

-- базовые мета-команды psql
\dt          -- список таблиц
\d users     -- описание таблицы
\di          -- список индексов
\dn          -- список схем
\q           -- выйти
```

```bash
# подключение
psql -h localhost -p 5432 -U postgres -d myapp

# дамп / восстановление
pg_dump -U postgres myapp > myapp.sql
psql  -U postgres myapp < myapp.sql
```
