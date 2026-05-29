# Spring — Test (Unit, Integration, MockMvc)

> **Теги:** #spring #framework #testing #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Spring_Index]]

---

## 🔹 Пирамида тестирования

```
                    ┌─────────┐
                    │   E2E   │  мало, медленно, хрупко
                   ┌┴─────────┴┐
                   │ Integration │  контекст Spring, БД, HTTP
                  ┌┴─────────────┴┐
                  │     Unit      │  много, быстро, изолированно
                  └───────────────┘
```

| Уровень | Что тестирует | Скорость | Стоимость поддержки |
|---------|---------------|----------|---------------------|
| Unit | Класс / метод | мс | низкая |
| Integration | Несколько слоёв + Spring | секунды | средняя |
| E2E | Полный flow + реальная инфра | минуты | высокая |

> [!tip] Подробности Mockito/JUnit
> См. [[JUnit_Mockito]] — здесь фокус на Spring-специфике.

---

## 🔹 Unit-тесты без Spring

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepository;
    @InjectMocks OrderService orderService;

    @Test
    void createsOrder() {
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        Order result = orderService.create(new OrderRequest(1L, 2));

        assertThat(result.getQuantity()).isEqualTo(2);
        verify(orderRepository).save(any(Order.class));
    }

    @Test
    void spyExample() {
        List<String> list = new ArrayList<>();
        List<String> spy = spy(list);
        spy.add("a");
        verify(spy).add("a");
    }
}
```

> [!note] Без `@SpringBootTest`
> Unit-тесты не поднимают контекст — быстрые, стабильные.

---

## 🔹 @SpringBootTest

```java
@SpringBootTest
class OrderServiceIntegrationTest {
    @Autowired OrderService orderService;

    @Test
    void fullFlow() { }
}

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class ApiIntegrationTest {
    @Autowired TestRestTemplate restTemplate;
}
```

| `webEnvironment` | Поведение |
|------------------|-----------|
| `MOCK` (default) | MockMvc, без реального порта |
| `RANDOM_PORT` | Embedded server на случайном порту |
| `DEFINED_PORT` | `server.port` из properties |
| `NONE` | Без web-слоя |

> [!warning] Тяжёлые тесты
> Полный контекст — секунды на класс. Не покрывать `@SpringBootTest` всё подряд — использовать slice tests.

---

## 🔹 Sliced Tests

| Аннотация | Загружает | Мокает |
|-----------|-----------|--------|
| `@WebMvcTest` | MVC, Jackson, `@Controller` | `@Service`, `@Repository` |
| `@DataJpaTest` | JPA, Hibernate, `@Entity` | `@Service`, `@Controller` |
| `@JsonTest` | Jackson, `@JsonComponent` | остальное |
| `@RestClientTest` | RestTemplate/WebClient config | сервер |

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mvc;
    @MockBean OrderService orderService;

    @Test
    void returnsOrder() throws Exception {
        when(orderService.find(1L)).thenReturn(new OrderDto(1L, "PAID"));

        mvc.perform(get("/api/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("PAID"));
    }
}
```

---

## 🔹 MockMvc

```java
@Autowired MockMvc mvc;

mvc.perform(post("/api/orders")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""
            {"productId":1,"quantity":2}
            """))
    .andExpect(status().isCreated())
    .andExpect(jsonPath("$.id").exists())
    .andExpect(header().exists("Location"))
    .andDo(print());
```

```java
@SpringBootTest
@AutoConfigureMockMvc
class FullStackMvcTest {
    @Autowired MockMvc mvc;
}
```

**JSONPath:** `jsonPath("$.items[0].name").value("Widget")`

---

## 🔹 @MockBean / @SpyBean

| | Mockito `@Mock` | Spring `@MockBean` |
|---|----------------|-------------------|
| Контекст | Нет | Заменяет bean в Spring context |
| Где | Unit-тесты | `@WebMvcTest`, `@SpringBootTest` |

```java
@MockBean
PaymentGateway paymentGateway;  // внедрится вместо реального bean

@SpyBean
NotificationService notificationService;  // частичный mock реального bean
```

> [!warning] `@MockBean` медленнее
> Пересоздаёт bean definition — на больших suite замедляет старт контекста.

---

## 🔹 Testcontainers

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired OrderRepository orderRepository;
}
```

**Boot 3.1+ `@ServiceConnection`:**

```java
@SpringBootTest
@Testcontainers
class AppIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    // auto-config datasource без @DynamicPropertySource
}
```

> [!tip] Testcontainers
> Реальная БД в Docker — поведение ближе к production, чем H2.

---

## 🔹 Работа с БД в тестах

```java
@DataJpaTest
@Transactional  // rollback после каждого теста
class UserRepositoryTest {
    @Autowired UserRepository repo;
}

@SpringBootTest
@Sql("/testdata/orders.sql")
class OrderServiceSqlTest { }
```

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| **H2 in-memory** | Быстро, без Docker | Диалект ≠ PostgreSQL |
| **@Transactional rollback** | Чистое состояние | Не ловит commit-bugs |
| **Testcontainers** | Реальный PostgreSQL | Docker, медленнее |

### ❌ / ✅

### ❌ Только H2 для PostgreSQL-specific SQL

### ✅ Testcontainers + Flyway migrations как на prod

---

## 🔹 WireMock

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class ExternalApiTest {

    @Autowired TestRestTemplate restTemplate;

    @Test
    void callsExternal() {
        stubFor(get(urlEqualTo("/rates"))
            .willReturn(aResponse()
                .withHeader("Content-Type", "application/json")
                .withBody("{\"usd\": 1.0}")));

        Rate rate = restTemplate.getForObject("http://localhost:{port}/rates", Rate.class, wireMockPort());
    }
}
```

> [!note] Когда
> Мок внешних HTTP-сервисов без поднятия реального API.

---

## 🔹 Тестирование Security

```java
@WebMvcTest(AdminController.class)
@Import(SecurityConfig.class)
class AdminControllerTest {

    @Autowired MockMvc mvc;

    @Test
    @WithMockUser(roles = "USER")
    void forbiddenForUser() throws Exception {
        mvc.perform(get("/api/admin/stats"))
            .andExpect(status().isForbidden());
    }

    @Test
    void withJwt() throws Exception {
        mvc.perform(get("/api/orders")
                .with(jwt().authorities(new SimpleGrantedAuthority("ROLE_USER"))))
            .andExpect(status().isOk());
    }
}
```

---

## 🔹 Итог

1. Больше unit (Mockito), меньше тяжёлых `@SpringBootTest`.
2. Slice tests — точечная загрузка контекста.
3. MockMvc — тест REST без поднятия браузера.
4. `@MockBean` — замена bean в Spring context.
5. Testcontainers — реалистичная БД.
6. `@Transactional` на тесте — rollback, но не замена интеграционных сценариев с commit.
7. `@WithMockUser` / `jwt()` — Security в MockMvc.

```
Шпаргалка Test:
─────────────────────────────────────────
@ExtendWith(MockitoExtension.class)  → unit
@WebMvcTest + MockMvc + @MockBean    → controller
@DataJpaTest                         → repository
@SpringBootTest(webEnvironment=...)  → integration
@Testcontainers + PostgreSQLContainer
@ServiceConnection (Boot 3.1+)
@Transactional                       → rollback (осторожно)
@WithMockUser / jwt()                → security
@Sql("/data.sql")                    → fixtures
```
