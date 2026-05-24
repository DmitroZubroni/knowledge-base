# JUnit 5 / Mockito

> [!abstract] Связи
> [[main Tools]] | [[Spring_Test]]

---

## 🔹 JUnit 5 — архитектура

```
JUnit Platform
    ├── Jupiter (JUnit 5 API)     ← @Test, assertions
    └── Vintage (JUnit 4 bridge)  ← legacy @Test org.junit
```

```java
@BeforeAll  static void initAll() { }   // один раз на класс
@BeforeEach void init() { }              // перед каждым тестом
@AfterEach  void tearDown() { }
@AfterAll   static void tearDownAll() { }
```

```java
@TestMethodOrder(OrderAnnotation.class)
@DisplayName("Order service tests")
@Tag("unit")
class OrderServiceTest { }
```

---

## 🔹 Assertions (AssertJ vs JUnit native)

```java
// JUnit
assertEquals(2, result.getQuantity());
assertTrue(result.isActive());
assertThrows(OrderNotFoundException.class, () -> service.find(99L));

assertAll(
    () -> assertEquals("PAID", order.getStatus()),
    () -> assertNotNull(order.getId())
);
```

```java
// AssertJ — читаемые цепочки
assertThat(order.getStatus()).isEqualTo("PAID");
assertThat(users).hasSize(3).extracting(User::getEmail).contains("a@b.com");
assertThatThrownBy(() -> service.find(99L))
    .isInstanceOf(OrderNotFoundException.class)
    .hasMessageContaining("99");
```

### ❌ JUnit без контекста

```java
assertTrue(list.contains(x) && list.size() == 3);  // неясно что сломалось
```

### ✅ AssertJ

```java
assertThat(list).containsExactly(x, y, z);
```

---

## 🔹 @ParameterizedTest

```java
@ParameterizedTest
@ValueSource(strings = { "ACTIVE", "PAID", "SHIPPED" })
void validStatuses(String status) {
    assertThat(OrderStatus.isValid(status)).isTrue();
}

@ParameterizedTest
@CsvSource({ "1, 10, 10", "2, 5, 10", "0, 100, 0" })
void multiply(int a, int b, int expected) {
    assertThat(a * b).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("orderProvider")
void processOrders(Order order) { }

static Stream<Order> orderProvider() {
    return Stream.of(new Order(1L), new Order(2L));
}

@ParameterizedTest
@EnumSource(OrderStatus.class)
void allStatuses(OrderStatus status) { }
```

---

## 🔹 Exception Testing

### ✅ assertThrows

```java
@Test
void notFound() {
    var ex = assertThrows(OrderNotFoundException.class,
        () -> service.find(999L));
    assertThat(ex.getMessage()).contains("999");
}
```

### ❌ try-catch в тесте

```java
try {
    service.find(999L);
    fail("Expected exception");
} catch (OrderNotFoundException e) { }  // старый стиль
```

---

## 🔹 Mockito — основы

| | `@Mock` | `@Spy` | `@InjectMocks` |
|---|---------|--------|----------------|
| Объект | Полностью fake | Обёртка над real | Тестируемый класс |
| Вызовы | Только stub | Real + можно stub | Реальная логика |
| Когда | Изоляция unit | Partial mock | SUT с @Mock deps |

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repository;
    @InjectMocks OrderService service;

    @Test
    void creates() {
        when(repository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        Order result = service.create(new OrderRequest(1L, 2));

        assertThat(result.getQuantity()).isEqualTo(2);
        verify(repository).save(any(Order.class));
    }
}
```

```java
when(repo.find(1L)).thenThrow(new RuntimeException("db down"));
when(repo.find(2L)).thenAnswer(inv -> new Order(2L));

// spy — doReturn для избежания реального вызова
doReturn(mockUser).when(spy).loadUser("admin");
```

---

## 🔹 Mockito — verify

```java
verify(repository).save(argThat(o -> o.getQuantity() == 2));
verify(repository, times(1)).findById(1L);
verify(repository, never()).deleteAll();
verify(repository, atLeastOnce()).flush();

verifyNoMoreInteractions(repository);  // осторожно — хрупко при рефакторинге
```

**ArgumentCaptor:**

```java
ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);
verify(repository).save(captor.capture());
assertThat(captor.getValue().getStatus()).isEqualTo(PAID);
```

---

## 🔹 ArgumentMatchers

```java
when(repo.findByEmail(anyString())).thenReturn(Optional.of(user));
when(repo.findById(eq(1L))).thenReturn(Optional.of(order));

verify(repo).save(argThat(o -> o.getTotal().signum() > 0));
```

### ❌ Смешивать matchers и реальные значения

```java
when(repo.find(1L, anyString())).thenReturn(...);  // OK — все matchers или все literal
when(repo.find(1L, "x")).thenReturn(...);           // OK
// when(repo.find(any(), "x")) — только any() для обоих если один matcher
```

---

## 🔹 Mockito — advanced

```java
@Captor ArgumentCaptor<Order> orderCaptor;

@Test
void inOrder() {
    InOrder inOrder = inOrder(repo, notificationService);
    service.placeOrder(req);
    inOrder.verify(repo).save(any());
    inOrder.verify(notificationService).send(any());
}

@Test
void staticMock() {
    try (MockedStatic<Instant> mocked = mockStatic(Instant.class)) {
        mocked.when(Instant::now).thenReturn(fixedInstant);
        // ...
    }
}
```

---

## 🔹 Test Fixtures и паттерны

```java
// Object Mother
public class OrderMother {
    public static Order paidOrder() {
        return Order.builder().status(PAID).quantity(1).build();
    }
}

// ❌ god @BeforeEach
@BeforeEach void setup() {
    // 50 строк создания всего подряд
}
```

### ✅ Минимальный setup + фабрики

```java
@Test
void test() {
    Order order = OrderMother.paidOrder();
    when(repo.findById(1L)).thenReturn(Optional.of(order));
}
```

---

## 🔹 Итог

1. JUnit 5 Jupiter — стандарт; Vintage для legacy.
2. AssertJ — предпочтительнее для читаемости.
3. `@ParameterizedTest` — табличные кейсы.
4. `@Mock` + `@InjectMocks` — unit без Spring.
5. `verify` + `ArgumentCaptor` — поведение, не только результат.
6. Object Mother — переиспользуемые тестовые данные.

```
Шпаргалка JUnit/Mockito:
─────────────────────────────────────────
@ExtendWith(MockitoExtension.class)
@Mock / @InjectMocks / @Spy
when(x).thenReturn(y) / thenThrow
verify(mock, times(n)).method()
ArgumentCaptor.forClass(T)
assertThrows(Ex.class, () -> ...)
@ParameterizedTest + @CsvSource
assertThat(x).isEqualTo(y)       → AssertJ
❌ try-catch / mixed matchers
```
