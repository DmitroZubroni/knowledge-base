# HTTP & Network

> **Теги:** #interviews #tools #http #network #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 HTTP методы

### Основные методы

| Метод | Idempotent | Safe | Cacheable | Описание |
|-------|------------|------|-----------|----------|
| **GET** | ✅ | ✅ | ✅ | Получение ресурса |
| **POST** | ❌ | ❌ | ❌ | Создание ресурса |
| **PUT** | ✅ | ❌ | ❌ | Полное обновление |
| **PATCH** | ❌ | ❌ | ❌ | Частичное обновление |
| **DELETE** | ✅ | ❌ | ❌ | Удаление ресурса |

### Idempotent (Идемпотентность)

Один и тот же запрос даёт один и тот же результат.

```http
GET /users/1  → всегда один и тот же результат
PUT /users/1  → один и тот же результат при повторении
DELETE /users/1 → второй DELETE вернёт 404, но эффект тот же
```

### Safe (Безопасность)

Не изменяет состояние сервера.

```http
GET, HEAD, OPTIONS — safe
POST, PUT, DELETE — не safe
```

---

## 🔹 HTTP статус коды

### Категории

| Код | Категория | Описание |
|-----|-----------|----------|
| **1xx** | Informational | Информационные |
| **2xx** | Success | Успех |
| **3xx** | Redirection | Перенаправление |
| **4xx** | Client Error | Ошибка клиента |
| **5xx** | Server Error | Ошибка сервера |

### Часто используемые

| Код | Описание |
|-----|----------|
| **200 OK** | Успех |
| **201 Created** | Ресурс создан |
| **204 No Content** | Успех, без тела |
| **301 Moved Permanently** | Постоянное перенаправление |
| **302 Found** | Временное перенаправление |
| **400 Bad Request** | Неверный запрос |
| **401 Unauthorized** | Требуется аутентификация |
| **403 Forbidden** | Нет прав |
| **404 Not Found** | Ресурс не найден |
| **409 Conflict** | Конфликт (например, дубликат) |
| **500 Internal Server Error** | Ошибка сервера |
| **503 Service Unavailable** | Сервис недоступен |

---

## 🔹 HTTP заголовки

### Основные заголовки

| Заголовок | Описание |
|-----------|----------|
| **Content-Type** | Тип содержимого (application/json, text/html) |
| **Content-Length** | Длина тела в байтах |
| **Authorization** | Аутентификация (Bearer token, Basic) |
| **Accept** | Какие форматы принимает клиент |
| **User-Agent** | Информация о клиенте |
| **Host** | Имя хоста |
| **Cache-Control** | Управление кэшированием |
| **ETag** | Идентификатор версии ресурса |

### Пример

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer abc123
Content-Length: 45

{"name": "Alice", "email": "alice@example.com"}
```

---

## 🔹 REST vs gRPC

### REST (Representational State Transfer)

```http
GET /users/1
Content-Type: application/json

{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com"
}
```

**Преимущества:**
- Универсален (HTTP/JSON)
- Легко отлаживать (человекочитаемый)
- Поддержка браузерами

**Недостатки:**
- Текстовый формат (JSON) — больше данных
- Нет строгой типизации
- Медленнее (текст vs binary)

### gRPC (Google Remote Procedure Call)

```protobuf
// user.proto
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  int32 id = 1;
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

**Преимущества:**
- Binary формат (Protocol Buffers) — меньше данных
- Строгая типизация
- Быстрее (binary vs text)
- Поддержка streaming

**Недостатки:**
- Требует генерации кода
- Не поддерживается браузерами напрямую
- Сложнее отлаживать

### Сравнение

| Характеристика | REST | gRPC |
|----------------|------|-------|
| Формат | JSON (text) | Protobuf (binary) |
| Типизация | Слабая | Строгая |
| Производительность | Медленнее | Быстрее |
| Отладка | Легко | Сложнее |
| Streaming | Нет | Да |
| Поддержка браузеров | Да | Нет (требует gateway) |

---

## 🔹 Sync vs Async IO

### Synchronous (Blocking)

```java
// RestTemplate (synchronous)
RestTemplate restTemplate = new RestTemplate();
User user = restTemplate.getForObject("http://api.example.com/users/1", User.class);
// блокирует поток до получения ответа
```

**Проблемы:**
- Блокирует поток
- Не масштабируется при высокой нагрузке
- Требует много потоков

### Asynchronous (Non-blocking)

```java
// WebClient (asynchronous)
WebClient webClient = WebClient.create();
Mono<User> userMono = webClient.get()
    .uri("http://api.example.com/users/1")
    .retrieve()
    .bodyToMono(User.class);

userMono.subscribe(user -> {
    // обрабатываем ответ асинхронно
});
```

**Преимущества:**
- Не блокирует поток
- Масштабируется лучше
- Меньше потоков

**Недостатки:**
- Сложнее код (reactive)
- Сложнее отладка

---

## 🔹 RestTemplate vs WebClient

### RestTemplate (Spring Boot < 3.0)

```java
RestTemplate restTemplate = new RestTemplate();

// GET
User user = restTemplate.getForObject("http://api.example.com/users/1", User.class);

// POST
User newUser = restTemplate.postForObject(
    "http://api.example.com/users",
    user,
    User.class
);

// PUT
restTemplate.put("http://api.example.com/users/1", user);

// DELETE
restTemplate.delete("http://api.example.com/users/1");
```

**Характеристики:**
- Synchronous (blocking)
- Простой API
- Deprecated в Spring Boot 3.0

### WebClient (Spring Boot 3.0+)

```java
WebClient webClient = WebClient.create();

// GET
User user = webClient.get()
    .uri("http://api.example.com/users/1")
    .retrieve()
    .bodyToMono(User.class)
    .block();  // или subscribe() для асинхронности

// POST
User newUser = webClient.post()
    .uri("http://api.example.com/users")
    .bodyValue(user)
    .retrieve()
    .bodyToMono(User.class)
    .block();

// PUT
webClient.put()
    .uri("http://api.example.com/users/1")
    .bodyValue(user)
    .retrieve()
    .bodyToMono(Void.class)
    .block();

// DELETE
webClient.delete()
    .uri("http://api.example.com/users/1")
    .retrieve()
    .bodyToMono(Void.class)
    .block();
```

**Характеристики:**
- Asynchronous (non-blocking)
- Reactive (Project Reactor)
- Рекомендуется в Spring Boot 3.0+

### Сравнение

| Характеристика | RestTemplate | WebClient |
|----------------|--------------|------------|
| Тип | Synchronous | Asynchronous |
| Blocking | Да | Нет |
| Reactor | Нет | Да (Mono/Flux) |
| Spring Boot 3.0+ | Deprecated | Рекомендуется |
| Сложность | Простой | Сложнее |

---

## 🔹 TCP vs UDP

### TCP (Transmission Control Protocol)

**Надёжный, ориентированный на соединение.**

```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
(соединение установлено)
```

**Характеристики:**
- Гарантирует доставку
- Гарантирует порядок
- Контроль ошибок
- Медленнее (overhead)

**Когда использовать:** HTTP, FTP, SSH, Email

### UDP (User Datagram Protocol)

**Ненадёжный, без соединения.**

```
Client → Datagram → Server
(без подтверждения)
```

**Характеристики:**
- Нет гарантии доставки
- Нет гарантии порядка
- Быстрее (нет overhead)
- Меньше данных

**Когда использовать:** DNS, Video streaming, Gaming, VoIP

### Сравнение

| Характеристика | TCP | UDP |
|----------------|-----|-----|
| Надёжность | Гарантирует | Не гарантирует |
| Порядок | Гарантирует | Не гарантирует |
| Скорость | Медленнее | Быстрее |
| Соединение | Да | Нет |
| Использование | HTTP, FTP | DNS, Streaming |

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **HTTP методы:** GET (safe, idempotent), POST (не safe), PUT (idempotent), DELETE (idempotent)
> - **Статус коды:** 2xx (успех), 4xx (ошибка клиента), 5xx (ошибка сервера)
> - **Заголовки:** Content-Type, Authorization, Accept, Cache-Control
> - **REST vs gRPC:** REST (JSON, универсальный), gRPC (Protobuf, быстрый, типизированный)
> - **Sync vs Async:** Sync (blocking, RestTemplate), Async (non-blocking, WebClient)
> - **TCP vs UDP:** TCP (надёжный, HTTP), UDP (быстрый, DNS, streaming)

```
HTTP методы:
GET → safe, idempotent, cacheable
POST → не safe, не idempotent
PUT → idempotent
DELETE → idempotent

Статус коды:
2xx → успех (200, 201)
4xx → ошибка клиента (400, 401, 403, 404)
5xx → ошибка сервера (500, 503)

REST vs gRPC:
REST → JSON, text, универсальный
gRPC → Protobuf, binary, быстрый, типизированный

RestTemplate vs WebClient:
RestTemplate → synchronous, blocking, deprecated
WebClient → asynchronous, non-blocking, reactive

TCP vs UDP:
TCP → надёжный, соединение, HTTP
UDP → ненадёжный, без соединения, DNS, streaming
```
