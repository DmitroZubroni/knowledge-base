# Swagger

> **Теги:** #api #swagger #конспект  

> [!abstract] Связи
> [[main]] | [[main API]]

---

## 🔹 Swagger vs OpenAPI

> [!note] История
> **Swagger** (2011) — первый инструмент описания REST API. С 2015 спецификация передана **OpenAPI Initiative** и переименована в **OpenAPI Specification (OAS)**.

| Термин сегодня | Что это |
|----------------|---------|
| **OpenAPI** | Спецификация (YAML/JSON) — формат документа |
| **Swagger UI** | UI для просмотра и тестирования spec |
| **Swagger Editor** | Редактор spec в браузере |
| **Swagger Codegen** | Генерация клиента/сервера из spec (legacy, есть openapi-generator) |

> [!tip] В разговоре
> «Swagger-документация» часто = OpenAPI spec + Swagger UI. Технически spec — **OpenAPI 3.x**. Подробнее — [[OpenApi]].

---

## 🔹 Swagger 2.0 vs OpenAPI 3.x

| | Swagger 2.0 | OpenAPI 3.x |
|---|-------------|-------------|
| `host` + `basePath` | Отдельные поля | `servers[]` |
| Body parameters | `in: body` | `requestBody` |
| Reusable models | `definitions` | `components/schemas` |
| Callbacks / Links | ❌ | ✅ |
| Поддержка | Legacy | Актуальный стандарт |

```yaml
# Swagger 2.0 (legacy)
swagger: "2.0"
host: api.example.com
basePath: /v1
paths:
  /users:
    get:
      responses:
        200:
          description: OK
```

```yaml
# OpenAPI 3.x (использовать это)
openapi: 3.0.3
servers:
  - url: https://api.example.com/v1
paths:
  /users:
    get:
      responses:
        '200':
          description: OK
```

---

## 🔹 Swagger Editor

- Онлайн: [editor.swagger.io](https://editor.swagger.io)
- Live preview + валидация spec
- Экспорт YAML/JSON

> [!tip] Contract-first
> Пишешь spec в Editor → генерируешь код через openapi-generator → реализуешь интерфейсы.

---

## 🔹 Swagger Codegen vs openapi-generator

| | swagger-codegen | openapi-generator |
|---|-----------------|-------------------|
| Статус | Maintenance | Активная разработка |
| Генерация | Java, TS, Python, ... | Больше языков и шаблонов |

```bash
# openapi-generator (рекомендуется)
openapi-generator generate -i openapi.yaml -g spring -o ./generated
```

---

## 🔹 Spring Boot — что использовать сегодня

> [!note] Не springfox
> **springfox** (Swagger 2) — deprecated. Используй **springdoc-openapi** → см. [[OpenApi]].

```
springdoc-openapi → генерирует OpenAPI 3 spec
Swagger UI        → /swagger-ui.html (визуализация)
```

---

## 🔹 Итог

1. **OpenAPI** — spec; **Swagger** — экосистема инструментов вокруг spec.
2. Новые проекты — только **OAS 3.x**, не Swagger 2.0.
3. Документация в Spring — **springdoc**, не springfox.
4. Генерация кода — **openapi-generator**.

```
Шпаргалка Swagger:
─────────────────────────────────────────
OpenAPI 3.x        → spec (YAML/JSON)
Swagger UI         → визуализация + Try it out
Swagger Editor     → редактирование spec
springdoc-openapi  → Spring Boot (не springfox)
openapi-generator  → codegen из spec
Подробности        → [[OpenApi]]
```
