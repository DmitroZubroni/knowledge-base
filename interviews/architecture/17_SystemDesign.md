> **Теги:** #interviews #architecture #system-design #конспект
> [!abstract] Связи
> [[main]] | [[Interviews]]

# System Design — Вопросы на собесе

---

## 🔹 Шаблон ответа — как вести себя на интервью

System Design интервью длится 45-60 минут. Интервьюер оценивает не только знание технологий, но и **структуру мышления**. Следуй этому шаблону:

```
1. Clarify Requirements     ~5 мин  — уточнить что строим
2. Estimates                ~5 мин  — посчитать нагрузку
3. High-Level Design        ~10 мин — общая архитектура
4. Deep Dive                ~20 мин — детали ключевых компонентов
5. Bottlenecks & Tradeoffs  ~10 мин — узкие места и компромиссы
```

---

### Шаг 1: Clarify Requirements — что именно строим

Никогда не начинай рисовать архитектуру сразу. Сначала задай вопросы:

```
Функциональные требования (что система делает):
  - Какие основные функции? (create, read, delete?)
  - Кто пользователи? (end users, internal?)
  - Нужна ли аналитика / история?
  - Нужен ли real-time?

Нефункциональные требования (как работает):
  - Сколько пользователей? DAU/MAU?
  - Какой ожидаемый RPS?
  - Допустимая задержка (latency)?
  - Доступность (99.9%? 99.99%?)
  - Сколько хранить данные?
  - Нужна ли консистентность или достаточно eventual?
```

> [!tip] Зачем это нужно
> Вопросы показывают что ты думаешь системно. Одна и та же задача ("сделай чат") при 1000 пользователей и при 100 миллионах решается принципиально по-разному.

---

### Шаг 2: Estimates — оценка нагрузки

Это показывает что ты умеешь работать с числами. Не нужна точность — нужен правильный порядок.

**Полезные константы:**
```
1 день    = 86 400 сек ≈ 100K сек
1 месяц   = ~2.5M сек
1M DAU    → при равномерной нагрузке ≈ 10 RPS

Размеры:
  1 tweet/пост    ≈ 1 KB
  1 фото          ≈ 200 KB (thumbnail) / 2 MB (full)
  1 видео минута  ≈ 10 MB (720p)
  1 UUID          = 16 bytes
  1 URL           ≈ 200 bytes
```

**Шаблон расчёта:**
```
Пример: URL Shortener
  - 100M новых URL в день
  - Write RPS:  100M / 86400 ≈ 1 200 RPS
  - Read/Write ratio = 10:1 → Read RPS ≈ 12 000 RPS
  
  Хранение:
  - 1 запись ≈ 500 bytes (URL + метаданные)
  - 100M × 500 bytes = 50 GB/день
  - За 5 лет: 50 GB × 365 × 5 ≈ 90 TB
  
  Вывод:
  - Чтение: 12K RPS → нужен кэш
  - Хранение: 90 TB → нужна СУБД с шардированием или NoSQL
```

---

### Шаг 3: High-Level Design — общая архитектура

Нарисуй схему из 5-7 компонентов:

```
Client
  ↓
CDN (статика, кэш на уровне PoP)
  ↓
Load Balancer (L7, nginx/ALB)
  ↓
API Servers (stateless, горизонтально масштабируемы)
  ↓
Cache Layer (Redis)       Message Queue (Kafka)
  ↓                              ↓
Primary DB ←──── Read ────→ Worker Services
  ↓
Replica(s)
```

**Стандартные компоненты и когда добавлять:**

| Компонент | Когда добавить |
|-----------|---------------|
| CDN | Статика, изображения, видео, глобальные пользователи |
| Load Balancer | Больше одного API сервера |
| Cache (Redis) | Read-heavy, latency < 10ms, повторяющиеся запросы |
| Message Queue (Kafka) | Асинхронная обработка, decoupling, spike в нагрузке |
| Read Replicas | Read RPS > Write RPS |
| Sharding | Данных > 1-2 TB или Write RPS слишком высокий |
| Object Storage (S3) | Файлы, изображения, видео |

---

### Шаг 4: Deep Dive — детали

Интервьюер обычно сам укажет куда копать: "расскажи подробнее как генерируется ID" или "как работает кэш". Но имей готовые ответы по:
- Ключевой алгоритм (генерация ID, хэширование)
- Схема данных (таблицы, индексы)
- API endpoints
- Как решаешь конкретную сложность (N+1, race condition)

---

### Шаг 5: Bottlenecks & Tradeoffs

Самостоятельно озвучь проблемы до того как спросит интервьюер:

```
"Узкое место здесь — база данных при росте записей.
 Решение — шардирование по user_id.
 Компромисс — cross-shard запросы станут сложнее."

"Я выбрал eventual consistency вместо strong consistency
 потому что для ленты новостей пользователь готов
 подождать секунду пока данные распространятся."
```

---

## 🔹 Числа которые нужно знать

```
Latency (порядок):
  L1 кэш CPU      ~0.5 ns
  L2 кэш CPU      ~7 ns
  RAM              ~100 ns
  SSD random read  ~100 μs
  HDD random read  ~10 ms
  Redis GET        ~0.1 ms
  БД запрос (local)~1-5 ms
  HTTP round-trip  ~100-300 ms (в пределах ДЦ ~1ms)

Пропускная способность:
  SSD              ~500 MB/s
  Сеть (1Gbps)     ~100 MB/s
  Сеть (10Gbps)    ~1 GB/s

Availability:
  99%   → 3.65 дней downtime/год
  99.9% → 8.7 часов/год
  99.99% → 52 минуты/год (four nines)
  99.999% → 5 минут/год (five nines)
```

---

## 🔹 CAP теорема

В распределённой системе при сетевом разделении (Partition) нельзя одновременно иметь:
- **C**onsistency — все узлы видят одинаковые данные
- **A**vailability — каждый запрос получает ответ

**P (Partition Tolerance) обязательна** — сеть ненадёжна, разделение неизбежно. Выбор всегда между C и A при P.

| Выбор | Поведение при разделении | Примеры |
|-------|--------------------------|---------|
| **CP** | Отклонить запрос → нет устаревших данных | ZooKeeper, HBase, MongoDB (default) |
| **AP** | Ответить устаревшими данными → eventual consistency | Cassandra, DynamoDB, CouchDB |

**Практический выбор:**
- Финансы, инвентарь, бронирование → **CP** (нельзя продать то чего нет)
- Лента новостей, счётчик лайков, профиль → **AP** (пользователь подождёт секунду)

**PACELC** — расширение CAP: даже без разделения выбираешь между Latency и Consistency:
```
При Partition: A или C
Else (нормальная работа): L (latency) или C (consistency)
```

---

## 🔹 Масштабирование БД

```
Нагрузка растёт → стратегия:

Шаг 1: Вертикальное масштабирование (scale up)
  → Больше RAM, CPU, SSD. Просто, но есть предел.

Шаг 2: Read Replicas
  → Мастер — только запись, реплики — только чтение.
  → Работает если Read >> Write.
  → Replication lag — реплики могут отставать.

Шаг 3: Кэш перед БД (Redis)
  → 80% запросов обычно можно кэшировать.
  → Снимает нагрузку с БД радикально.

Шаг 4: Шардирование (Horizontal Partitioning)
  → Разбить данные на несколько БД-серверов.
  → Стратегии: Hash sharding, Range sharding, Directory sharding.
  → Проблемы: cross-shard JOIN, resharding при изменении числа шардов.

Шаг 5: CQRS
  → Разделить модели чтения и записи.
  → Read model оптимизирован для запросов (денормализован).
  → Write model — нормализован, консистентен.
```

---

## 🔹 Разбор задачи: Design URL Shortener

**Функциональные требования:**
- `POST /shorten` → вернуть короткий URL (например `bit.ly/abc123`)
- `GET /:code` → redirect на оригинальный URL
- Аналитика кликов (опционально)

**Нефункциональные:**
- 100M URL создаётся в день
- Read:Write = 10:1
- Latency редиректа < 10ms
- High Availability (99.99%)

**Оценка:**
```
Write RPS: 100M / 86400 ≈ 1 200
Read RPS:  12 000
Хранение:  100M × 500B ≈ 50GB/день → за 5 лет ~90TB
```

**Ключевой вопрос: как генерировать короткий код?**

```
Вариант 1: MD5/SHA256 от long URL → взять первые 7 символов
  Минус: коллизии (разные URL → одинаковый код)

Вариант 2: Base62 от auto-increment ID
  62^7 = ~3.5 трлн уникальных кодов
  ID 1 → "0000001", ID 2 → "0000002" ... ID 125 → "0000023"
  Минус: предсказуемость (можно угадать следующий URL)

Вариант 3: Base62 от UUID / random
  Нет коллизий, непредсказуемо. Рекомендуется.
```

**High-Level архитектура:**
```
Client
  ↓
Load Balancer
  ↓
API Server (stateless, N инстансов)
  ├── POST /shorten → генерация кода → запись в DB → кэш
  └── GET /:code   → проверить Redis → если нет: читать DB → 301/302 redirect

Redis (кэш code → long_url)
  TTL: 24 часа для популярных, 1 час для остальных

Database (PostgreSQL или Cassandra)
  urls: id, short_code, long_url, user_id, created_at, expires_at
  clicks: id, short_code, timestamp, user_agent, country
```

**Схема данных:**
```sql
CREATE TABLE urls (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    expires_at  TIMESTAMPTZ
);
CREATE INDEX idx_short_code ON urls(short_code);

-- Аналитика — отдельная таблица или Kafka → ClickHouse
CREATE TABLE clicks (
    id          BIGSERIAL,
    short_code  VARCHAR(10),
    clicked_at  TIMESTAMPTZ,
    country     VARCHAR(2),
    user_agent  TEXT
);
```

**Bottlenecks:**
```
Проблема: 12K RPS чтений → БД не справится
Решение:  Redis кэш, TTL = 24ч. 99% запросов из кэша.

Проблема: 90TB за 5 лет → один сервер не вместит
Решение:  Шардирование по short_code (hash sharding)
          или: Cassandra — хорошо масштабируется горизонтально

Проблема: одна точка отказа
Решение:  Multi-region deployment, Redis Cluster, DB replicas

Проблема: 301 vs 302 redirect?
  301 Permanent → браузер кэширует, меньше нагрузка, нет аналитики кликов
  302 Temporary → каждый клик через нас, есть аналитика. ВЫБИРАЕМ 302.
```

---

## 🔹 Разбор задачи: Design Rate Limiter

**Зачем нужен:**
- Защита от DDoS и abuse
- Fair use между клиентами
- Защита backend от перегрузки

**Функциональные требования:**
- Ограничить RPS на уровне API Gateway или в приложении
- Гибкие правила: по IP, по user_id, по endpoint
- Возвращать 429 Too Many Requests при превышении

**Алгоритмы:**

### Token Bucket (самый популярный)

```
Ведро ёмкостью N токенов. Каждую секунду добавляется M токенов.
Запрос потребляет 1 токен. Нет токенов → 429.

Плюсы: выдерживает burst (всплески), прост в реализации
Минусы: burst может пропустить слишком много запросов
```

```java
// Реализация через Redis
public boolean allowRequest(String userId) {
    String key = "rate_limit:" + userId;
    String script = """
        local tokens = tonumber(redis.call('GET', KEYS[1])) or tonumber(ARGV[1])
        if tokens < 1 then
            return 0
        end
        redis.call('SET', KEYS[1], tokens - 1)
        redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
        return 1
        """;
    // ARGV[1] = max_tokens, ARGV[2] = window_seconds
    Long result = redis.execute(script, List.of(key), "100", "60");
    return Long.valueOf(1L).equals(result);
}
```

### Sliding Window Counter (точнее)

```
Разбить время на маленькие интервалы (bucket'ы по 1 сек).
Хранить счётчик в каждом bucket.
Суммировать bucket'ы за последний период (60 сек → 60 bucket'ов).

Плюсы: плавный лимит, нет edge case на границе окна
Минусы: больше памяти
```

```java
// Через Redis ZSet (Sorted Set)
public boolean allowRequest(String userId) {
    String key = "rate:sliding:" + userId;
    long now = System.currentTimeMillis();
    long windowStart = now - 60_000;
    int limit = 100;

    String script = """
        redis.call('ZREMRANGEBYSCORE', KEYS[1], '-inf', ARGV[1])
        local count = redis.call('ZCARD', KEYS[1])
        if tonumber(count) < tonumber(ARGV[3]) then
            redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2])
            redis.call('EXPIRE', KEYS[1], 61)
            return 1
        end
        return 0
        """;

    Long allowed = redis.execute(script, List.of(key),
        String.valueOf(windowStart),
        String.valueOf(now),
        String.valueOf(limit));
    return Long.valueOf(1L).equals(allowed);
}
```

### Fixed Window vs Sliding Window:

```
Fixed Window (простой):
  00:00 - 01:00: счётчик = 0
  00:59: запрос 100 → OK (счётчик = 100, лимит достигнут)
  01:00: счётчик сбрасывается в 0
  01:01: запрос 100 → OK
  → 200 запросов за 2 секунды! Это edge case проблема.

Sliding Window:
  В любую секунду: смотрим последние 60 сек.
  Нет edge case на границе окна.
```

**High-Level архитектура:**
```
Client Request
  ↓
API Gateway / Load Balancer
  ↓
Rate Limiter Middleware
  ├── Получить rule: по IP, user_id, endpoint
  ├── Проверить Redis → счётчик превышен?
  │     YES → 429 Too Many Requests, заголовки:
  │             X-RateLimit-Limit: 100
  │             X-RateLimit-Remaining: 0
  │             Retry-After: 30
  │     NO  → пропустить, увеличить счётчик
  ↓
Backend Service
```

**Где хранить состояние:**
```
In-memory (локально): ультра-быстро, но каждый инстанс считает отдельно
  → при 3 инстансах лимит фактически 300, а не 100

Redis (централизованно): один счётчик на все инстансы
  → небольшая latency (~1ms), но точно

Redis Cluster: для высокой доступности
  → repликация rate limit данных
```

**Race Condition:**
```
Проблема: два запроса одновременно читают счётчик = 99,
оба разрешают (99 < 100), оба инкрементируют → счётчик = 101

Решение: Lua скрипт (атомарное check + increment)
         или Redis INCR (атомарный) + проверка
```

---

## 🔹 Разбор задачи: Design Notification System

**Функциональные требования:**
- Push уведомления (iOS/Android)
- Email уведомления
- SMS уведомления
- 10M уведомлений в день

**Оценка:**
```
10M / 86400 ≈ 115 уведомлений/сек в среднем
Пиковая нагрузка × 5 → 575/сек
```

**Архитектура:**
```
Services (OrderService, PaymentService...)
  ↓
Notification Service API
  ↓
Kafka Topics:
  ├── topic: push-notifications
  ├── topic: email-notifications
  └── topic: sms-notifications
  ↓
Workers (Consumer Groups):
  ├── Push Worker → APNs (Apple) / FCM (Google)
  ├── Email Worker → SendGrid / SES
  └── SMS Worker → Twilio

MongoDB:
  - notification_log (history)
  - user_preferences (opt-in/out, device tokens)
```

**Ключевые решения:**
```
Почему Kafka (а не прямые вызовы):
  - Сервисы не ждут доставки уведомления
  - Retry при сбое внешнего API (APNs/FCM)
  - Пиковые нагрузки сглаживаются очередью

Retry стратегия:
  - Exponential backoff: 1s, 2s, 4s, 8s...
  - Max 3 retry → Dead Letter Queue
  - DLT обрабатывается отдельным consumer

Дедупликация:
  - notification_id → idempotency key
  - Проверять перед отправкой: не отправляли ли уже?
  - Redis SET NX для quick check
```

---

## 🔹 Типичные паттерны и когда применять

```
Паттерн              Когда                              Пример
───────────────────────────────────────────────────────────────
Read Replicas        Read >> Write                      Каталог товаров
Sharding             Данных > 1TB, Write bottleneck     User data в соц. сети
Cache (Redis)        Повторяющиеся запросы, < 10ms      Session, profile, feed
CDN                  Статика, глобальные пользователи   Images, CSS, JS
Message Queue        Async, decoupling, spike           Email, notification
CQRS                 Read/Write модели сильно отличаются BI + OLTP
Event Sourcing       Аудит, replay, история изменений   Финансы, заказы
Saga                 Distributed transactions            Checkout flow
Circuit Breaker      Внешние API могут падать           Payment gateway
Consistent Hashing   Шардирование с минимальным rehash  Cache nodes, shards
```

---

## 🔹 Шпаргалка

```
Шаблон ответа (45 мин):
  1. Clarify ~5 мин  — функц. требования, DAU, RPS, latency, availability
  2. Estimates ~5 мин — Write RPS, Read RPS, хранение за 5 лет
  3. High-Level ~10 мин — схема из 5-7 компонентов
  4. Deep Dive ~20 мин — алгоритм, схема данных, API
  5. Bottlenecks ~10 мин — узкие места, трейдоффы, озвучь сам

Числа:
  1M DAU → ~10 RPS среднее
  1 день = 86 400 сек
  99.99% = 52 мин downtime/год

CAP: при Partition → C (отказать) или A (устаревшие данные)
  CP: ZooKeeper, MongoDB. AP: Cassandra, DynamoDB.

URL Shortener:
  Генерация: Base62(UUID) → 7 символов = 62^7 ≈ 3.5 трлн
  301 (permanent) = нет аналитики, 302 (temporary) = есть аналитика
  Кэш редиректов в Redis с TTL

Rate Limiter:
  Token Bucket — выдерживает burst
  Sliding Window (ZSet) — точнее, нет edge case на границе
  Атомарность: Lua скрипт или Redis INCR
  X-RateLimit-Limit / X-RateLimit-Remaining / Retry-After

Масштабирование БД:
  1. Vertical (scale up)
  2. Read Replicas
  3. Cache (Redis)
  4. Sharding (hash by key)
  5. CQRS

Трейдоффы которые нужно знать:
  SQL vs NoSQL: ACID vs масштабирование
  Strong vs Eventual Consistency: точность vs latency
  Push vs Pull: сервер инициирует vs клиент опрашивает
  Sync vs Async: latency vs resilience
  Monolith vs Microservices: простота vs независимое масштабирование
```
