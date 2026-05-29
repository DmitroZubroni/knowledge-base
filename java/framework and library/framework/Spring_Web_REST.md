# Spring — Web REST (MVC, REST, Validation)

> **Теги:** #spring #framework #mvc #rest #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Spring_Index]]

---

## 🔹 DispatcherServlet

```
HTTP Request
    ↓
DispatcherServlet
    ↓
HandlerMapping          → какой @Controller метод
    ↓
HandlerAdapter          → вызов метода, аргументы
    ↓
@Controller / @RestController method
    ↓
Return value handler    → @ResponseBody, ResponseEntity
    ↓
HttpMessageConverter    → JSON (Jackson)
    ↓
HTTP Response
```

> [!note] Front Controller
> Один `DispatcherServlet` на приложение — единая точка входа для MVC.

---

## 🔹 @RestController vs @Controller

| | `@Controller` | `@RestController` |
|---|---------------|-------------------|
| Return | View name → ViewResolver | Объект → JSON/XML (`@ResponseBody`) |
| Использование | SSR (Thymeleaf) | REST API |

```java
@RestController  // = @Controller + @ResponseBody на классе
@RequestMapping("/api/v1/users")
public class UserController { }
```

---

## 🔹 Mapping аннотации

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderDto get(@PathVariable Long id) { }

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<OrderDto> create(@RequestBody @Valid OrderRequest req) { }

    @PutMapping("/{id}")
  public OrderDto update(@PathVariable Long id, @RequestBody OrderRequest req) { }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { }

    @PatchMapping("/{id}")
    public OrderDto patch(@PathVariable Long id, @RequestBody Map<String, Object> patch) { }
}
```

| `@RequestMapping` attr | Назначение |
|------------------------|------------|
| `path` / `value` | URL pattern |
| `method` | GET, POST, ... |
| `consumes` | Content-Type запроса |
| `produces` | Content-Type ответа |
| `params` / `headers` | Условия на query/header |

---

## 🔹 Получение данных из запроса

```java
@GetMapping("/search")
public List<Product> search(
    @RequestParam String q,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(required = false) String category
) { }

@GetMapping("/{id}")
public Product get(@PathVariable("id") Long productId) { }

@PostMapping
public Order create(@RequestBody @Valid OrderRequest body) { }

@GetMapping("/me")
public User me(
    @RequestHeader("Authorization") String auth,
    @CookieValue(name = "session_id", required = false) String sessionId
) { }
```

| Аннотация | Источник |
|----------|----------|
| `@PathVariable` | URI template `/users/{id}` |
| `@RequestParam` | Query `?page=0` |
| `@RequestBody` | Body (JSON) |
| `@RequestHeader` | HTTP header |
| `@CookieValue` | Cookie |
| `@RequestPart` | Multipart part |

---

## 🔹 ResponseEntity

```java
@GetMapping("/{id}")
public ResponseEntity<OrderDto> get(@PathVariable Long id) {
    return orderService.find(id)
        .map(dto -> ResponseEntity.ok(dto))
        .orElse(ResponseEntity.notFound().build());
}

@PostMapping
public ResponseEntity<OrderDto> create(@Valid @RequestBody OrderRequest req) {
    OrderDto created = orderService.create(req);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
        .path("/{id}")
        .buildAndExpand(created.id())
        .toUri();
    return ResponseEntity.created(location).body(created);
}

// builder
return ResponseEntity
    .status(HttpStatus.CONFLICT)
    .header("X-Error-Code", "DUPLICATE")
    .body(errorBody);
```

> [!tip] Зачем ResponseEntity
> Полный контроль: status, headers, body в одном месте.

---

## 🔹 Валидация

```java
public record OrderRequest(
    @NotNull @Positive Long productId,
    @Min(1) @Max(100) int quantity,
    @Email String contactEmail,
    @Pattern(regexp = "^[A-Z]{2}$") String countryCode
) {}

@PostMapping
public OrderDto create(@Valid @RequestBody OrderRequest req) { }

@PostMapping("/group")
public void validateGroup(@Validated(Create.class) @RequestBody OrderRequest req) { }
```

```java
@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody OrderRequest req, BindingResult br) {
    if (br.hasErrors()) {
        return ResponseEntity.badRequest().body(br.getAllErrors());
    }
    // ...
}
```

| Аннотация | Проверка |
|-----------|----------|
| `@NotNull` / `@NotBlank` | Не null / не пустая строка |
| `@Size(min,max)` | Длина |
| `@Min` / `@Max` | Число |
| `@Email` | Email format |
| `@Past` / `@Future` | Дата |
| `@Valid` | Каскад на вложенные объекты |

> [!tip] `@Validated` на классе controller
> Включает validation groups на уровне параметров метода.

---

## 🔹 Exception Handling

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class OrderNotFoundException extends RuntimeException { }

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setProperty("errors", ex.getBindingResult().getFieldErrors());
        return problem;
    }
}
```

> [!note] RFC 7807 ProblemDetail
> Стандартный JSON для ошибок (Boot 3+): `type`, `title`, `status`, `detail`.

```
Иерархия (пример):
RuntimeException
└── BusinessException
    ├── OrderNotFoundException
    └── PaymentFailedException
```

---

## 🔹 Фильтры и Interceptors

```
Request → Filter chain (Servlet) → DispatcherServlet → Interceptor → Controller
```

| | Filter | HandlerInterceptor |
|---|--------|-------------------|
| API | Servlet | Spring MVC |
| Scope | Любой request | Только Spring-handled |
| Пример | Logging, encoding | Auth check, timing |

```java
@Component
public class RequestIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        MDC.put("requestId", UUID.randomUUID().toString());
        try { chain.doFilter(req, res); }
        finally { MDC.clear(); }
    }
}

@Component
public class TimingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("start", System.currentTimeMillis());
        return true;
    }
}
```

> [!note] Security filters
> Spring Security filters — в Servlet chain **до** DispatcherServlet. См. [[Spring_Security]].

---

## 🔹 Content Negotiation

```java
@GetMapping(value = "/report", produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
public Report getReport() { }

@PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
public void upload(@RequestBody Report report) { }
```

Клиент влияет через `Accept` / `Content-Type` header. По умолчанию REST → JSON (Jackson).

---

## 🔹 RestTemplate

```java
RestTemplate rt = new RestTemplate();
User user = rt.getForObject("https://api.example.com/users/{id}", User.class, id);
ResponseEntity<User> resp = rt.exchange(url, HttpMethod.PUT, entity, User.class);
```

> [!warning] RestTemplate
> **Maintenance mode** — блокирующий, один экземпляр на поток. Для новых проектов — **WebClient**.

---

## 🔹 WebClient

```java
@Bean
WebClient apiClient(WebClient.Builder builder) {
    return builder.baseUrl("https://api.example.com").build();
}

// синхронно (блокирует!)
User user = apiClient.get()
    .uri("/users/{id}", id)
    .retrieve()
    .bodyToMono(User.class)
    .block();

// реактивно
Mono<User> userMono = apiClient.get().uri("/users/{id}", id).retrieve().bodyToMono(User.class);
Flux<Order> orders = apiClient.get().uri("/orders").retrieve().bodyToFlux(Order.class);
```

> [!note] Mono / Flux
> **Mono** — 0..1 элемент. **Flux** — 0..N. Не блокировать reactive pipeline без необходимости.

---

## 🔹 Multipart / File Upload

```java
@PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public String upload(@RequestParam("file") MultipartFile file) {
    file.transferTo(Path.of("/tmp", file.getOriginalFilename()));
    return "ok";
}
```

```yaml
spring.servlet.multipart.max-file-size: 10MB
spring.servlet.multipart.max-request-size: 15MB
```

---

## 🔹 CORS

```java
@CrossOrigin(origins = "https://frontend.example.com", maxAge = 3600)
@GetMapping("/public")
public List<Item> list() { }

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://frontend.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true);
    }
}
```

> [!warning] CORS + Security
> Настройка в Security filter chain может переопределить MVC CORS — согласовать оба. См. [[Spring_Security]].

---

## 🔹 Итог

1. `DispatcherServlet` — центр MVC pipeline.
2. `@RestController` — JSON API; `@Controller` — views.
3. `ResponseEntity` — status + headers + body.
4. Bean Validation — `@Valid` на `@RequestBody`.
5. `@RestControllerAdvice` + `ProblemDetail` — единый формат ошибок.
6. Filter (Servlet) vs Interceptor (MVC).
7. WebClient вместо RestTemplate для новых HTTP-клиентов.

```
Шпаргалка Web REST:
─────────────────────────────────────────
@RestController + @RequestMapping
@GetMapping / @PostMapping / @PathVariable / @RequestBody
@Valid + jakarta.validation.*
ResponseEntity.status().body()
@RestControllerAdvice + @ExceptionHandler
ProblemDetail (RFC 7807)
OncePerRequestFilter
WebClient.retrieve().bodyToMono()
@CrossOrigin / WebMvcConfigurer.addCorsMappings
```
