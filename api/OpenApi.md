# OpenAPI

> **Теги:** #api #openapi #конспект  

> [!abstract] Связи
> [[main]] | [[main API]]

---

## 🔹 OpenAPI Specification

> [!note] OAS 3.x
> **OpenAPI Specification** — машиночитаемое описание REST API: paths, schemas, security.

```yaml
openapi: 3.0.3
info:
  title: Order API
  version: 1.0.0
servers:
  - url: https://api.example.com
paths:
  /orders/{id}:
    get:
      tags: [Orders]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
components:
  schemas:
    Order:
      type: object
      properties:
        id: { type: integer }
        status: { type: string, enum: [CREATED, PAID] }
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - bearerAuth: []
```

---

## 🔹 Swagger UI vs OpenAPI

| | OpenAPI | Swagger |
|---|---------|---------|
| Что | Спецификация (YAML/JSON) | Экосистема инструментов |
| UI | — | Swagger UI — визуализация spec |
| Editor | — | Swagger Editor |
| Codegen | openapi-generator | swagger-codegen (legacy) |

> [!note] История
> Swagger 2.0 → переименован в OpenAPI 3.0. «Swagger» часто = UI, «OpenAPI» = spec.

---

## 🔹 springdoc-openapi в Spring Boot

```gradle
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
```

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

- `/v3/api-docs` — JSON spec
- `/swagger-ui.html` — интерактивная документация

---

## 🔹 Аннотации документирования

```java
@Tag(name = "Orders", description = "Order management")
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Operation(summary = "Get order by id")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Found"),
        @ApiResponse(responseCode = "404", description = "Not found")
    })
    @GetMapping("/{id}")
    public OrderDto get(
        @Parameter(description = "Order id") @PathVariable Long id
    ) { }

    @Hidden  // не показывать в UI
    @GetMapping("/internal/debug")
    public void debug() { }
}
```

```java
@Schema(description = "Order request")
public record OrderRequest(
    @Schema(example = "1") Long productId,
    @Schema(minimum = "1", maximum = "100") int quantity
) {}
```

---

## 🔹 Компоненты

**$ref — переиспользование:**

```yaml
components:
  schemas:
    Error:
      type: object
      properties:
        message: { type: string }
  responses:
    NotFound:
      description: Not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

**SecuritySchemes:**

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            read: Read access
```

---

## 🔹 Contract-first vs Code-first

| | Code-first | Contract-first |
|---|------------|----------------|
| Источник | Аннотации в коде | openapi.yaml в Git |
| Генерация | springdoc → spec | openapi-generator → server/client |
| Когда | Быстрый старт, внутренние API | Публичный API, несколько команд |

> [!tip] Практика
> Внутренний сервис — code-first (springdoc).  
> API для партнёров — contract-first, spec review до кода.

---

## 🔹 Итог

1. OpenAPI — стандарт описания REST.
2. springdoc — auto `/v3/api-docs` + Swagger UI.
3. `@Operation`, `@Schema`, `@ApiResponse` — документирование.
4. `$ref` + `components` — DRY в spec.
5. Contract-first для внешних API.

```
Шпаргалка OpenAPI:
─────────────────────────────────────────
openapi: 3.0.3
springdoc-openapi-starter-webmvc-ui
/v3/api-docs + /swagger-ui.html
@Operation / @Schema / @ApiResponse
components.schemas + $ref
bearerAuth securityScheme
Code-first (springdoc) vs Contract-first (yaml)
```
