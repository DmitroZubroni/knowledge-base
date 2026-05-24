# 00 — MySQL Hub

## 🔹 Структура раздела

| Файл | Содержание |
|------|------------|
| [[MySQL_Architecture]] | Архитектура, Storage Engines, память, конфигурация |
| [[MySQL_DataTypes]] | Типы данных, отличия от стандарта SQL |
| [[MySQL_Indexes]] | B-tree, Full-text, Spatial, составные, покрывающие |
| [[MySQL_Transactions]] | ACID, InnoDB, уровни изоляции, блокировки |
| [[MySQL_Performance]] | EXPLAIN, slow query log, оптимизация запросов |
| [[MySQL_Advanced]] | Репликация, партиционирование, хранимые процедуры |

---

## 🔹 Порядок изучения

[[MySQL_Architecture]] ↓
[[MySQL_DataTypes]] ↓
[[MySQL_Indexes]] ↓
[[MySQL_Transactions]] ↓
[[MySQL_Performance]] ↓
[[MySQL_Advanced]]

---

## 🔹 Что такое MySQL

```
MySQL — реляционная СУБД с открытым исходным кодом.
Версия 1.0 — 1995. Владелец — Oracle Corporation (с 2010).
Текущая стабильная версия — MySQL 8.0 / 8.4 LTS.

Ключевые особенности:
  - Pluggable Storage Engines (InnoDB, MyISAM, Memory и др.)
  - InnoDB — основной движок: ACID, MVCC, FK, row-level locks
  - Широко используется в web (WordPress, Drupal, GitLab)
  - Binlog-based репликация
  - JSON поддержка с версии 5.7
  - Window functions, CTE — с версии 8.0
```

---

## 🔹 Быстрый старт

```sql
-- создать БД
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE myapp;

-- базовые команды
SHOW DATABASES;
SHOW TABLES;
DESCRIBE users;          -- или DESC users
SHOW CREATE TABLE users;
SHOW INDEX FROM users;
```

```bash
# подключение
mysql -h localhost -P 3306 -u root -p myapp

# дамп / восстановление
mysqldump -u root -p myapp > myapp.sql
mysql -u root -p myapp < myapp.sql

# дамп с параметрами
mysqldump -u root -p --single-transaction --routines --triggers myapp > myapp.sql
```

---

## 🔹 MySQL vs PostgreSQL — кратко

| | MySQL | PostgreSQL |
|--|-------|------------|
| Движок | Pluggable (InnoDB default) | Единый |
| MVCC | InnoDB | Нативный |
| Arrays | ❌ | ✅ |
| JSONB | ❌ (только JSON) | ✅ |
| RETURNING | ❌ | ✅ |
| Window functions | 8.0+ | Давно |
| CTE | 8.0+ | Давно |
| Full-text | InnoDB/MyISAM | tsvector + GIN |
| Репликация | Binlog | WAL Streaming |
| Популярность | Web, LAMP стек | Enterprise, геоданные |
