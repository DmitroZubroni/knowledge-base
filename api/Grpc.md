# gRPC

> **Теги:** #api #grpc #конспект  

> [!abstract] Связи
> [[main]] | [[main Internet Networks]]

---

## 🔹 Что такое gRPC

> [!note] Определение
> **gRPC** (Google RPC) — фреймворк RPC поверх **HTTP/2** с сериализацией **Protocol Buffers (Protobuf)**. Контракт задаётся в `.proto` файле.

| | REST (JSON) | gRPC (Protobuf) |
|---|-------------|-----------------|
| Протокол | HTTP/1.1 (часто) | HTTP/2 |
| Формат | JSON (текст) | Binary Protobuf |
| Контракт | OpenAPI (опционально) | `.proto` (обязательно) |
| Streaming | Long polling / SSE | Native bidirectional |
| Browser | ✅ Native | ❌ Нужен gRPC-Web |
| Human-readable | ✅ | ❌ (нужны tools) |
| Performance | Хорошо | Выше (меньше payload, multiplexing) |

---

## 🔹 Protocol Buffers

```protobuf
// user.proto
syntax = "proto3";

package com.example.user;

option java_package = "com.example.user.grpc";
option java_multiple_files = true;

message GetUserRequest {
  int64 id = 1;
}

message UserResponse {
  int64 id = 1;
  string email = 2;
  string name = 3;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (UserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream UserResponse);
}
```

> [!tip] Поля numbered
> Номера полей (`= 1`) — part of wire format. Не переиспользовать номера после удаления полей.

```bash
# генерация Java stubs
protoc --java_out=src/main/java user.proto
# или через protobuf-maven-plugin / gradle plugin
```

---

## 🔹 Типы вызовов (RPC modes)

| Mode | Client → Server | Пример |
|------|-----------------|--------|
| **Unary** | 1 request → 1 response | `GetUser` |
| **Server streaming** | 1 request → N responses | `ListUsers stream` |
| **Client streaming** | N requests → 1 response | Upload chunks |
| **Bidirectional streaming** | N ↔ N | Chat, real-time sync |

```
Unary:           Client ──req──► Server ──res──► Client

Server stream:   Client ──req──► Server ──res──►
                                      ──res──►
                                      ──res──►

Client stream:   Client ──req──►
                 Client ──req──► Server ──res──► Client

Bidi stream:     Client ◄──────────────────────► Server
```

---

## 🔹 gRPC vs REST — когда что

| Выбирай **gRPC** | Выбирай **REST** |
|------------------|------------------|
| Микросервисы internal | Public API, партнёры |
| High throughput, low latency | Простота интеграции |
| Streaming (real-time) | Кэширование HTTP, CDN |
| Строгий контракт proto | Браузер, mobile без proxy |
| Polyglot с codegen | OpenAPI ecosystem |

> [!warning] Public API
> Внешним клиентам чаще **REST + OpenAPI**. gRPC — между своими сервисами в private network.

---

## 🔹 Spring gRPC

```gradle
implementation("net.devh:grpc-spring-boot-starter:3.1.0.RELEASE")
implementation("io.grpc:grpc-protobuf")
implementation("io.grpc:grpc-stub")
```

```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<UserResponse> responseObserver) {
        User user = userService.find(request.getId());
        UserResponse response = UserResponse.newBuilder()
            .setId(user.getId())
            .setEmail(user.getEmail())
            .build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

```yaml
grpc:
  server:
    port: 9090
```

**gRPC client:**

```java
@GrpcClient("user-service")
private UserServiceGrpc.UserServiceBlockingStub userStub;

UserResponse user = userStub.getUser(
    GetUserRequest.newBuilder().setId(1L).build()
);
```

---

## 🔹 Status codes и ошибки

gRPC использует свои коды (не HTTP status в теле):

| Code | Аналог HTTP | Когда |
|------|-------------|-------|
| `OK` | 200 | Успех |
| `INVALID_ARGUMENT` | 400 | Невалидный запрос |
| `NOT_FOUND` | 404 | Ресурс не найден |
| `UNAUTHENTICATED` | 401 | Нет auth |
| `PERMISSION_DENIED` | 403 | Нет прав |
| `DEADLINE_EXCEEDED` | 504 | Timeout |
| `UNAVAILABLE` | 503 | Сервис недоступен |

```java
responseObserver.onError(
    Status.NOT_FOUND.withDescription("User not found").asRuntimeException()
);
```

---

## 🔹 Итог

1. gRPC = HTTP/2 + Protobuf + `.proto` contract.
2. Unary / streaming — native, без костылей.
3. Internal microservices — gRPC; public — REST.
4. Spring: `grpc-spring-boot-starter`, `@GrpcService`, `@GrpcClient`.

```
Шпаргалка gRPC:
─────────────────────────────────────────
.proto → protoc / gradle plugin
Unary / Server stream / Client stream / Bidi
HTTP/2 + binary Protobuf
@GrpcService / UserServiceImplBase
Status.NOT_FOUND, DEADLINE_EXCEEDED
Internal MS → gRPC | Public API → REST
```
