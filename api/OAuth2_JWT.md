# OAuth2 / JWT

> [!abstract] Связи
> [[main API]] | [[Spring_Security]] | [[Rest Api]]

---

## 🔹 Зачем OAuth2 / JWT

> [!note] Проблема сессий
> Cookie-сессии плохо масштабируются для API: sticky sessions, CSRF, shared session store.

**JWT + OAuth2 дают:**
- **Stateless** API — сервер проверяет подпись токена
- **Federated identity** — один IdP (Keycloak) для N микросервисов
- **Delegated authorization** — клиент не видит пароль пользователя

---

## 🔹 OAuth2 — роли и flow

| Роль | Описание |
|------|----------|
| **Resource Owner** | Пользователь |
| **Client** | Приложение (SPA, mobile, backend) |
| **Authorization Server** | Выдаёт tokens (Keycloak, Auth0) |
| **Resource Server** | API, защищённый access token |

### Authorization Code Flow (основной для web)

```
User → Client: "Login"
Client → Auth Server: redirect /authorize?client_id&redirect_uri&scope&state
User → Auth Server: login + consent
Auth Server → Client: redirect ?code=...
Client → Auth Server: POST /token (code + client_secret)
Auth Server → Client: access_token + refresh_token
Client → Resource Server: Authorization: Bearer <access_token>
```

> [!tip] PKCE
> Для SPA/mobile: `code_challenge` + `code_verifier` — защита от перехвата authorization code.

### Client Credentials Flow (M2M)

```
Client → Auth Server: POST /token (client_id + client_secret, grant_type=client_credentials)
Auth Server → Client: access_token
Client → Resource Server: API call
```

> [!warning] Implicit Flow
> **Устарел** — token в URL fragment, небезопасно. Использовать Authorization Code + PKCE.

---

## 🔹 OpenID Connect (OIDC)

> [!note] OIDC
> Расширение OAuth2 для **аутентификации**: `id_token` (JWT) + `userinfo` endpoint.

| Token | Назначение |
|-------|------------|
| **access_token** | Доступ к API (scopes) |
| **id_token** | Идентичность пользователя (claims) |
| **refresh_token** | Обновление access без повторного login |

**Claims в id_token:** `sub`, `email`, `name`, `iat`, `exp`, `iss`, `aud`.

---

## 🔹 JWT (JSON Web Token)

```
header.payload.signature
   │       │         └── HMAC/RSA подпись header+payload
   │       └── Base64url JSON (claims)
   └── alg, typ
```

**Header:**
```json
{ "alg": "RS256", "typ": "JWT" }
```

**Payload (пример):**
```json
{
  "sub": "user-123",
  "iss": "https://auth.example.com",
  "aud": "my-api",
  "exp": 1710000000,
  "iat": 1709990000,
  "roles": ["USER", "ADMIN"]
}
```

| Алгоритм | Ключ | Когда |
|----------|------|-------|
| **HS256** | Shared secret | Monolith, internal services |
| **RS256** | Private sign / Public verify (JWKS) | Microservices, IdP |

> [!warning] Payload не шифруется
> Только Base64url — **не класть** пароли, PII без необходимости.

---

## 🔹 Access Token vs Refresh Token

| | Access | Refresh |
|---|--------|---------|
| TTL | Короткий (5–60 мин) | Длинный (дни/недели) |
| Использование | Каждый API запрос | Только `/token` endpoint |
| Хранение клиента | Memory / httpOnly cookie | httpOnly cookie, secure |

### ❌ localStorage для access token
XSS украдёт token.

### ✅ httpOnly + Secure cookie или memory (SPA с коротким TTL)

**Refresh token rotation:** при refresh выдаётся новый refresh, старый инвалидируется.

**Revocation:** stateless JWT нельзя отозвать мгновенно без blacklist/TTL — планировать короткий `exp`.

---

## 🔹 JWT уязвимости

| Атака | Защита |
|-------|--------|
| `alg: none` | Whitelist алгоритмов на сервере |
| Weak secret brute force | RS256 + длинный secret / JWKS |
| Sensitive data in payload | Минимум claims |
| Expired token accepted | Проверять `exp`, clock skew |
| Token theft | HTTPS, httpOnly, short TTL |

---

## 🔹 Spring Resource Server

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myrealm
```

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    return http
        .oauth2ResourceServer(oauth -> oauth
            .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .build();
}

@Bean
JwtAuthenticationConverter jwtAuthConverter() {
    var converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        List<String> roles = jwt.getClaimAsStringList("roles");
        return roles.stream()
            .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
            .toList();
    });
    return converter;
}
```

---

## 🔹 Keycloak / Auth0 — краткий обзор

> [!note] IdP (Identity Provider)
> Готовый Authorization Server: users, realms, clients, scopes, social login.

| | Hosted (Auth0) | Self-hosted (Keycloak) |
|---|----------------|------------------------|
| Ops | Меньше | Свой кластер |
| Control | Ограничен | Полный |
| Cost | По MAU | Инфраструктура |

> [!warning] Свой auth server
> Писать с нуля — **почти никогда**. Использовать Keycloak, Auth0, Cognito.

---

## 🔹 Итог

1. OAuth2 — делегирование доступа; OIDC — идентичность.
2. JWT = header.payload.signature; RS256 для распределённых систем.
3. Authorization Code + PKCE — стандарт для web/SPA.
4. Короткий access + refresh rotation; не хранить в localStorage.
5. Spring: `oauth2ResourceServer().jwt()` + `issuer-uri`.

```
Шпаргалка OAuth2/JWT:
─────────────────────────────────────────
Authorization Code + PKCE  → web/SPA
Client Credentials           → M2M
access_token                 → API Bearer
id_token (OIDC)              → identity claims
HS256                        → shared secret
RS256 + JWKS                 → microservices
issuer-uri                   → Spring RS config
❌ alg:none, localStorage     → security risks
```
