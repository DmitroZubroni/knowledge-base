# 00 — SQL Hub

> **Теги:** #sql #hub #database  

> [!abstract] Навигация
> [[main]] | [[main Data Base]] | [[main SQL DB]]

## 🔹 Структура раздела

| Файл | Содержание |
|------|------------|
| [[SQL_DDL]] | CREATE, ALTER, DROP, Constraints, индексы |
| [[SQL_DML]] | SELECT, INSERT, UPDATE, DELETE, подзапросы, CASE |
| [[SQL_Joins]] | INNER, LEFT, RIGHT, FULL, CROSS, SELF JOIN |
| [[SQL_Advanced]] | CTE, оконные функции, транзакции, индексы, нормализация |

---

## 🔹 Порядок изучения

[[SQL_DDL]] ↓
[[SQL_DML]] ↓
[[SQL_Joins]] ↓
[[SQL_Advanced]]

---

## 🔹 Что такое SQL

```
SQL (Structured Query Language) — декларативный язык запросов
для работы с реляционными базами данных.

Стандарт: SQL-92, SQL:1999, SQL:2003, SQL:2016
Реализации: PostgreSQL, MySQL, Oracle, MS SQL Server, SQLite
```

> [!note] Реляционная модель
> Данные хранятся в **таблицах** (relations). Каждая таблица — набор строк (rows) и столбцов (columns). Связи между таблицами — через **Foreign Key**.

---

## 🔹 Категории команд

| Категория | Расшифровка | Команды |
|-----------|-------------|---------|
| **DDL** | Data Definition Language | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** | Data Manipulation Language | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| **DCL** | Data Control Language | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |
