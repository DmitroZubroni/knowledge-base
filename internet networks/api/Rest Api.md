# REST API

> **Теги:** #api #rest #конспект  

> [!abstract] Связи
> [[main]] | [[main API]]

---

## 🔹 Что такое REST

> [!note] Определение
> **REST** (Representational State Transfer) — архитектурный стиль Роя Филдинга (2000), не протокол и не фреймворк.

| | REST | SOAP | GraphQL | gRPC |
|---|------|------|---------|------|
| Формат | JSON/XML | XML | JSON | Protobuf |
| Контракт | OpenAPI (опционально) | WSDL | Schema | .proto |
| Transport | HTTP | HTTP/SMTP | HTTP | HTTP/2 |
| Когда | Public API, CRUD | Enterprise legacy | Flexible queries | Internal MS, streaming |

---

## 🔹 6 ограничений REST (constraints)

| Constraint | Значение | Нарушение |
|------------|----------|-----------|
| **Client-Server** | Разделение UI и данных | Fat client с бизнес-логикой в UI |
| **Stateless** | Сервер не хранит client state между запросами | Session state на сервере для API |
| **Cacheable** | Ответы помечаются cacheable или нет | Игнор Cache-Control |
| **Uniform Interface** | Единообразные URI, методы, representations | RPC `/getUser` |
| **Layered System** | Клиент не знает конечный сервер | — |
| **Code on Demand** | Сервер может отдавать исполняемый код | Редко (optional) |

---

## 🔹 Uniform Interface — 4 sub-constraints

1. **Resource identification** — URI идентифицирует ресурс: `/users/42`
2. **Manipulation through representations** — JSON/XML — представление ресурса
3. **Self-descriptive messages** — `Content-Type`, `Allow`, status codes понятны
4. **HATEOAS** — hypermedia links в ответе для навигации

```json
{
  "id": 42,
  "name": "Alice",
  "_links": {
    "self": { "href": "/users/42" },
    "orders": { "href": "/users/42/orders" }
  }
}
```

---

## 🔹 Richardson Maturity Model

```
Level 0: POST /rpc  { "action": "getUser", "id": 1 }     ← один endpoint
Level 1: GET /users/1  GET /orders/5                    ← ресурсы
Level 2: GET + POST + PUT + DELETE + 201/404/409        ← HTTP verbs + codes
Level 3: + HATEOAS links in response                    ← hypermedia
```

---

## 🔹 HTTP методы и семантика

| Метод | Safe | Idempotent | Body request | Body response |
|-------|:----:|:----------:|:------------:|:-------------:|
| GET | ✅ | ✅ | ❌ | ✅ |
| HEAD | ✅ | ✅ | ❌ | ❌ |
| POST | ❌ | ❌ | ✅ | ✅ |
| PUT | ❌ | ✅ | ✅ | ✅ |
| PATCH | ❌ | ❌* | ✅ | ✅ |
| DELETE | ❌ | ✅ | ❌** | ✅/204 |
| OPTIONS | ✅ | ✅ | ❌ | ✅ |

> [!warning] DELETE с телом / GET с телом
> Нестандартно, прокси/CDN могут отбросить — **избегать**.

---

## 🔹 HTTP Status Codes

| Код | Когда |
|-----|-------|
| **200** OK | Успешный GET/PUT/PATCH |
| **201** Created | POST создал ресурс (+ `Location`) |
| **204** No Content | DELETE успешен, нет тела |
| **301** Moved Permanently | URL изменился навсегда |
| **304** Not Modified | Conditional GET, кэш валиден |
| **400** Bad Request | Синтаксис/валидация |
| **401** Unauthorized | Нет/неверная аутентификация |
| **403** Forbidden | Аутентифицирован, но нет прав |
| **404** Not Found | Ресурс не существует |
| **409** Conflict | Конфликт версий/состояния |
| **422** Unprocessable Entity | Семантика неверна (часто validation) |
| **500** Internal Server Error | Баг сервера |
| **502** Bad Gateway | Upstream недоступен |
| **503** Service Unavailable | Overload / maintenance |
| **504** Gateway Timeout | Upstream timeout |

> [!warning] 401 vs 403
> **401** — «кто ты?» (login/token). **403** — «знаю, но нельзя».

> [!warning] 400 vs 422
> **400** — malformed JSON. **422** — JSON OK, но бизнес-валидация не прошла.

---

## 🔹 Дизайн URI

### ✅ Существительные, иерархия

```
GET    /api/v1/users
GET    /api/v1/users/42
GET    /api/v1/users/42/orders
POST   /api/v1/users/42/orders
```

### ❌ Глаголы в URI

```
GET /api/v1/getUser/42
POST /api/v1/createOrder
```

**Фильтрация и пагинация:**

```
GET /api/v1/orders?status=PAID&page=0&size=20&sort=createdAt,desc
```

---

## 🔹 Форматы запроса/ответа

```http
Content-Type: application/json
Accept: application/json
```

**Ошибки — RFC 7807 ProblemDetail:**

```json
{
  "type": "about:blank",
  "title": "Order not found",
  "status": 404,
  "detail": "Order 99 does not exist"
}
```

---

## 🔹 Пагинация

**Offset-based:**

```json
{
  "data": [...],
  "meta": { "page": 0, "size": 20, "total": 150, "hasNext": true }
}
```

> [!warning] Offset problem
> Вставка строк во время листания → дубликаты/пропуски.

**Cursor-based:**

```
GET /orders?after=eyJpZCI6MTIzfQ&size=20
```

> [!tip] Cursor
> Для real-time лент, чатов, больших таблиц.

---

## 🔹 Версионирование API

| Способ | Пример | Плюсы | Минусы |
|--------|--------|-------|--------|
| URI path | `/api/v1/users` | Явно, просто | Дублирование routes |
| Header | `Accept: application/vnd.api+json;version=2` | Чистые URI | Сложнее тестировать |
| Query | `?version=2` | Быстро | Загрязняет URL |

---

## 🔹 Идемпотентность и безопасность

**Idempotency-Key** для POST:

```http
POST /api/v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

Повтор с тем же ключом → тот же результат, без двойного списания.

---

## 🔹 Итог

1. REST — constraints, не «всё через GET».
2. Level 2 maturity — правильные HTTP методы и коды.
3. URI — существительные; ошибки — ProblemDetail.
4. 401 ≠ 403; 400 ≠ 422.
5. Пагинация: offset для админки, cursor для high-throughput.

```
Шпаргалка REST:
─────────────────────────────────────────
GET /resources/{id}              → read (safe)
POST /resources                  → create (201 + Location)
PUT /resources/{id}              → replace (idempotent)
PATCH /resources/{id}            → partial update
DELETE /resources/{id}           → 204
401 auth / 403 forbidden / 404 not found
?page&size&sort                  → pagination
Idempotency-Key                  → safe POST retry
```
