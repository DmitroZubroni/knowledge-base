# System Design

> **Теги:** #interviews #architecture #system-design #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Структура ответа на System Design интервью

### Шаг 1. Clarify Requirements (Уточнение требований)

**Вопросы:**
- Какой функционал нужен?
- Сколько пользователей (DAU/MAU)?
- Какой трафик (RPS)?
- Какие ограничения (latency, availability)?

**Пример:**
```
Q: Design a URL shortener
A: Questions:
- Сколько URL в день? (1M)
- Сколько запросов в секунду? (1000 RPS)
- Нужно ли аналитика? (да)
- Какой max length URL? (2048)
```

### Шаг 2. High-Level Design (Высокоуровневый дизайн)

**Компоненты:**
- Client (Web/Mobile)
- Load Balancer
- API Server
- Database
- Cache (Redis)
- Message Queue (если нужно)

**Пример:**
```
Client → Load Balancer → API Server → Cache → Database
```

### Шаг 3. Data Model (Модель данных)

**Таблицы:**
```sql
CREATE TABLE urls (
    id SERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Шаг 4. Detailed Design (Детальный дизайн)

**API endpoints:**
- `POST /shorten` — создать короткий URL
- `GET /{short_code}` — перенаправление

**Алгоритмы:**
- Генерация short code (Base62)
- Кэширование популярных URL

### Шаг 5. Bottlenecks & Scalability (Узкие места и масштабирование)

**Проблемы:**
- База данных не справляется → шардирование
- Кэш miss →预热 (warm-up)
- Single point of failure → репликация

### Шаг 6. Trade-offs (Компромиссы)

| Решение | Плюсы | Минусы |
|----------|-------|--------|
| SQL DB | ACID, сложные запросы | Сложнее масштабирование |
| NoSQL DB | Масштабирование | Нет ACID |

---

## 🔹 CAP Theorem

**CAP** — теорема о распределённых системах: можно выбрать только 2 из 3.

### C - Consistency (Согласованность)

Все узлы видят одни и те же данные одновременно.

```
Node A: x = 1
Node B: x = 1
После записи x = 2 на Node A:
Node A: x = 2
Node B: x = 2 (сразу)
```

### A - Availability (Доступность)

Каждый запрос получает ответ (успех или ошибка).

```
Даже если Node A упал, Node B отвечает
```

### P - Partition Tolerance (Устойчивость к разделению)

Система продолжает работать при потере связи между узлами.

```
Node A ⟷ Node B (связь потеряна)
Система продолжает работать
```

### CAP Trade-offs

| Система | C | A | P | Пример |
|---------|---|---|---|--------|
| **CP** | ✅ | ❌ | ✅ | MongoDB (default) |
| **AP** | ❌ | ✅ | ✅ | Cassandra, DynamoDB |
| **CA** | ✅ | ✅ | ❌ | RDBMS (single node) |

> [!note] В распределённых системах P обязательно (сеть ненадёжна)

### Примеры

#### CP (Consistency + Partition Tolerance)

```java
// MongoDB (write concern: majority)
// При разделении сети — ошибки записи
```

**Когда использовать:** Финансовые операции, где важна согласованность.

#### AP (Availability + Partition Tolerance)

```java
// Cassandra (eventual consistency)
// При разделении сети — продолжаем работать, данные расходятся
```

**Когда использовать:** Социальные сети, где важна доступность.

---

## 🔹 Масштабирование

### Вертикальное (Vertical Scaling)

Увеличение ресурсов одного сервера.

```
CPU: 4 cores → 8 cores
RAM: 8GB → 16GB
```

**Плюсы:**
- Простота
- Нет изменений в коде

**Минусы:**
- Ограничение физическими ресурсами
- Single point of failure
- Дорого

**Когда использовать:** Небольшие приложения, быстрый старт.

### Горизонтальное (Horizontal Scaling)

Добавление новых серверов.

```
Server 1 → Server 1 + Server 2 + Server 3
```

**Плюсы:**
- Теоретически неограниченно
- Отказоустойчивость
- Дешевле

**Минусы:**
- Сложность (load balancing, state management)
- Требует stateless

**Когда использовать:** Высокая нагрузка, микросервисы.

### Load Balancing

**Load Balancer** — распределение трафика между серверами.

#### Алгоритмы

| Алгоритм | Описание |
|----------|----------|
| **Round Robin** | По очереди |
| **Least Connections** | Наименьшее число соединений |
| **IP Hash** | По IP клиента (sticky session) |
| **Weighted** | С весами |

#### Примеры

```nginx
upstream backend {
    server server1:8080;
    server server2:8080;
    server server3:8080;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

---

## 🔹 Caching (Кэширование)

### Уровни кэширования

```
Client Cache (Browser)
    ↓
CDN Cache (Cloudflare, CloudFront)
    ↓
Load Balancer Cache
    ↓
Application Cache (Redis, Memcached)
    ↓
Database
```

### Стратегии кэширования

#### Cache-Aside (Lazy Loading)

```java
String value = cache.get(key);
if (value == null) {
    value = database.get(key);
    cache.put(key, value);
}
return value;
```

#### Write-Through

```java
database.set(key, value);
cache.set(key, value);  // синхронно
```

#### Write-Behind (Write-Back)

```java
cache.set(key, value);
// асинхронная запись в БД
```

### Cache Invalidation

**Проблема:** устаревшие данные.

**Решения:**
- TTL (Time To Live)
- Explicit invalidation
- Cache stampede prevention (single flight)

---

## 🔹 Database Scaling

### Read Replicas

```
Master (write) → Replica 1 (read)
                 → Replica 2 (read)
                 → Replica 3 (read)
```

**Плюсы:** Чтение масштабируется
**Минусы:** Запись не масштабируется

### Sharding

Разделение данных по shards.

```
Shard 1: user_id % 3 = 0
Shard 2: user_id % 3 = 1
Shard 3: user_id % 3 = 2
```

**Плюсы:** И чтение, и запись масштабируются
**Минусы:** Сложность, cross-shard queries

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **System Design структура:** Requirements → High-Level → Data Model → Detailed → Bottlenecks → Trade-offs
> - **CAP:** C (согласованность), A (доступность), P (устойчивость к разделению) — выбери 2
> - **CP:** MongoDB, **AP:** Cassandra, **CA:** RDBMS (single node)
> - **Vertical Scaling:** больше ресурсов на одном сервере
> - **Horizontal Scaling:** больше серверов, load balancing
> - **Caching:** Cache-Aside, Write-Through, Write-Behind
> - **Database Scaling:** Read Replicas (чтение), Sharding (чтение + запись)

```
System Design структура:
1. Clarify Requirements
2. High-Level Design
3. Data Model
4. Detailed Design
5. Bottlenecks & Scalability
6. Trade-offs

CAP:
C + P → MongoDB (согласованность важнее доступности)
A + P → Cassandra (доступность важнее согласованности)
C + A → RDBMS single node (без разделения сети)

Масштабирование:
Vertical → scale up (CPU, RAM)
Horizontal → scale out (больше серверов)

Caching:
Cache-Aside → lazy loading
Write-Through → синхронная запись
Write-Behind → асинхронная запись

Database Scaling:
Read Replicas → чтение масштабируется
Sharding → чтение + запись масштабируются
```
