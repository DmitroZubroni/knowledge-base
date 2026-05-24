# Spring — Security (Auth, Filters, JWT, OAuth2)

> [!abstract] Связи
> [[00_Spring_Index]] | [[Spring_Web_REST]] | [[OAuth2_JWT]]

---

## 🔹 Архитектура Security

```
HTTP Request
    ↓
SecurityFilterChain (цепочка фильтров)
    ├── SecurityContextPersistenceFilter
    ├── UsernamePasswordAuthenticationFilter  (form login)
    ├── BearerTokenAuthenticationFilter         (JWT / OAuth2 RS)
    ├── AuthorizationFilter
    └── ExceptionTranslationFilter
    ↓
DispatcherServlet → Controller
```

**Ключевые компоненты:**

| Компонент | Роль |
|-----------|------|
| `SecurityFilterChain` | Конфигурация фильтров и правил |
| `AuthenticationManager` | Делегирует `AuthenticationProvider` |
| `AuthenticationProvider` | Проверяет credentials → `Authentication` |
| `SecurityContextHolder` | Thread-local контекст текущего пользователя |

---

## 🔹 SecurityFilterChain

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // REST — часто отключён
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

> [!note] Spring Security 6+
> DSL через lambda (`authorizeHttpRequests`) — без `authorizeRequests()` (deprecated).

---

## 🔹 Authentication

```java
// после успешной аутентификации
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
```

```
login request
    → AuthenticationManager.authenticate(token)
        → AuthenticationProvider (UserDetailsService + PasswordEncoder)
            → Authentication (authenticated=true)
                → SecurityContextHolder
```

---

## 🔹 UserDetails и UserDetailsService

```java
public interface UserDetails {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}

@Service
@RequiredArgsConstructor
public class DbUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        var user = userRepository.findByEmail(username)
            .orElseThrow(() -> new UsernameNotFoundException(username));
        return User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())
            .roles(user.getRole().name())
            .build();
    }
}
```

---

## 🔹 Password Encoding

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // strength 10-12 typical
}

// при регистрации
String hash = passwordEncoder.encode(rawPassword);
// при login — AuthenticationProvider сравнивает через matches()
```

> [!warning] {noop}
> `{noop}password` — **только для dev/tests**. В production всегда BCrypt / Argon2 / PBKDF2.

| Encoder | Использование |
|---------|---------------|
| `BCryptPasswordEncoder` | Default choice |
| `Argon2PasswordEncoder` | Современный, memory-hard |
| `DelegatingPasswordEncoder` | Миграция старых хешей (`{bcrypt}`, `{argon2}`) |

---

## 🔹 HTTP Basic и Form Login

```java
http.httpBasic(Customizer.withDefaults());  // Authorization: Basic ...

http.formLogin(form -> form
    .loginPage("/login")
    .defaultSuccessUrl("/home")
);  // session-based web apps
```

| Механизм | Когда |
|----------|-------|
| **Basic** | Простые internal tools, не для браузера (credentials в каждом запросе) |
| **Form login** | SSR приложения с сессией |
| **JWT / OAuth2** | REST API, SPA, микросервисы |

---

## 🔹 JWT Authentication

```
1. POST /api/auth/login  { username, password }
        ↓
2. AuthenticationManager → valid
        ↓
3. JwtService.generateToken(user)  → access token (+ refresh optional)
        ↓
4. Client: Authorization: Bearer <token>
        ↓
5. JwtAuthenticationFilter (OncePerRequestFilter)
        → parse → validate signature & expiry → set SecurityContext
        ↓
6. Controller
```

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String header = req.getHeader(HttpHeaders.AUTHORIZATION);
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(req, res);
            return;
        }
        String jwt = header.substring(7);
        String username = jwtService.extractUsername(jwt);
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails user = userDetailsService.loadUserByUsername(username);
            if (jwtService.isValid(jwt, user)) {
                var auth = new UsernamePasswordAuthenticationToken(
                    user, null, user.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(req, res);
    }
}
```

**Claims:** `sub`, `exp`, `iat`, custom (`roles`, `tenantId`). Signing: HMAC (secret) или RSA (JWKS).

---

## 🔹 OAuth2 / OpenID Connect

| Роль | Назначение |
|------|------------|
| **Authorization Server** | Выдаёт tokens (Keycloak, Auth0, Okta) |
| **Resource Server** | API, валидирует access token |
| **Client** | Приложение, инициирует OAuth flow |

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/myrealm
          # или jwk-set-uri
```

```java
http.oauth2ResourceServer(oauth -> oauth.jwt(jwt -> jwt
    .jwtAuthenticationConverter(jwtAuthConverter())
));
```

> [!tip] Не писать свой JWT с нуля
> Для production — проверенный IdP + `spring-boot-starter-oauth2-resource-server`.

---

## 🔹 Authorization

```java
// HTTP
.requestMatchers("/api/admin/**").hasRole("ADMIN")      // ROLE_ADMIN
.requestMatchers("/api/reports/**").hasAuthority("report:read")

// Method security
@EnableMethodSecurity
public class MethodSecurityConfig { }

@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserDto getUser(@PathVariable Long userId) { }

@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) { }

@Secured("ROLE_MANAGER")
public void approve() { }
```

| Выражение | Смысл |
|-----------|-------|
| `permitAll()` | Без аутентификации |
| `authenticated()` | Любой залогиненный |
| `hasRole('X')` | ROLE_X |
| `hasAuthority('x')` | Точное имя authority |

---

## 🔹 CSRF

> [!note] CSRF
> Cross-Site Request Forgery — браузер отправляет cookie сессии на чужой сайт.

| Сценарий | CSRF |
|----------|------|
| Session + cookie auth | ✅ Включён |
| Stateless REST + JWT header | ❌ Обычно отключён |
| SPA + cookie | Double-submit cookie / SameSite |

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

---

## 🔹 CORS в Security

```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}

// в filterChain:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

---

## 🔹 Session Management

```java
http.sessionManagement(sm -> sm
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // JWT API
);

// или для web
.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
.maximumSessions(1)
```

| Policy | Поведение |
|--------|-----------|
| `STATELESS` | Без HttpSession (JWT) |
| `IF_REQUIRED` | Сессия при form login |
| `NEVER` | Не создавать, использовать если есть |

---

## 🔹 Exception Handling в Security

| Исключение | HTTP | Обработчик |
|-----------|------|-----------|
| Нет аутентификации | 401 | `AuthenticationEntryPoint` |
| Нет прав | 403 | `AccessDeniedHandler` |

```java
@Component
public class CustomAuthEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res,
                         AuthenticationException ex) throws IOException {
        res.setContentType(MediaType.APPLICATION_JSON_VALUE);
        res.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        res.getWriter().write("{\"error\": \"Unauthorized\"}");
    }
}

@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest req, HttpServletResponse res,
                       AccessDeniedException ex) throws IOException {
        res.setContentType(MediaType.APPLICATION_JSON_VALUE);
        res.setStatus(HttpServletResponse.SC_FORBIDDEN);
        res.getWriter().write("{\"error\": \"Forbidden\"}");
    }
}
```

```java
// Подключение в SecurityFilterChain
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint(customAuthEntryPoint)
    .accessDeniedHandler(customAccessDeniedHandler)
);
```

> [!note] По умолчанию
> Без кастомных обработчиков Spring Security вернёт HTML-страницу ошибки.
> Для REST API — всегда переопределять на JSON-ответ.

---

## 🔹 Security тесты

```java
@WebMvcTest(UserController.class)
class UserControllerSecurityTest {

    @Autowired MockMvc mvc;

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminEndpoint() throws Exception {
        mvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk());
    }

    @Test
    void unauthorized() throws Exception {
        mvc.perform(get("/api/admin/users"))
            .andExpect(status().isUnauthorized());
    }
}
```

| Аннотация | Назначение |
|-----------|------------|
| `@WithMockUser` | Fake user в SecurityContext |
| `@WithUserDetails` | Загрузка через UserDetailsService |
| `mvc.perform(...).with(user(...))` | Per-request user |

Подробнее — [[Spring_Test]].

---

## 🔹 Итог

1. Security — цепочка фильтров до DispatcherServlet.
2. `SecurityFilterChain` — declarative config (Boot 3 style).
3. `UserDetailsService` + `PasswordEncoder` — классическая аутентификация.
4. JWT — stateless; фильтр + validate claims.
5. OAuth2 Resource Server — JWT от IdP через `issuer-uri`.
6. `@PreAuthorize` — method-level authorization.
7. CSRF off для pure REST JWT; CORS настроить явно.

```
Шпаргалка Security:
─────────────────────────────────────────
@EnableWebSecurity + SecurityFilterChain
UserDetailsService + BCryptPasswordEncoder
SessionCreationPolicy.STATELESS  → JWT API
oauth2ResourceServer().jwt()     → OAuth2 RS
hasRole / hasAuthority / @PreAuthorize
OncePerRequestFilter             → JWT filter
@WithMockUser                    → тесты
csrf().disable()                 → только REST + token
AuthenticationEntryPoint         → кастомный 401 JSON
AccessDeniedHandler              → кастомный 403 JSON
```
