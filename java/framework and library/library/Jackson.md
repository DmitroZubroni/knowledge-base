# Jackson (JSON)

> **Теги:** #java #library #json #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Library]]

---

## 🔹 Что такое Jackson

> [!note] Определение
> **Jackson** — библиотека JSON ↔ Java. В Spring MVC — default `HttpMessageConverter`.

| Модуль | Назначение |
|--------|------------|
| `jackson-core` | Streaming parser/generator |
| `jackson-databind` | `ObjectMapper`, POJO mapping |
| `jackson-annotations` | `@JsonProperty`, ... |

```java
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user);
User user = mapper.readValue(json, User.class);
```

---

## 🔹 Сериализация / Десериализация

```java
// generic collections
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() {});

// stream
mapper.writeValue(response.getOutputStream(), order);
```

> [!warning] ObjectMapper thread-safe
> **Один singleton** на приложение — Spring Boot создаёт автоматически.

### ❌ new ObjectMapper() в каждом методе

---

## 🔹 Аннотации

```java
public class User {
    @JsonProperty("user_id")
    private Long id;

    @JsonIgnore
    private String internalNote;

    @JsonAlias({"mail", "e-mail"})
    private String email;

    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String phone;

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss", timezone = "UTC")
    private Instant createdAt;
}
```

```java
@JsonSerialize(using = MoneySerializer.class)
@JsonDeserialize(using = MoneyDeserializer.class)
private Money balance;

@JsonCreator
public User(@JsonProperty("id") Long id, @JsonProperty("email") String email) {
    this.id = id;
    this.email = email;
}
```

---

## 🔹 Naming Strategies

```java
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
// userId → user_id в JSON
```

```yaml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
```

| Strategy | JSON example |
|----------|--------------|
| `LOWER_CAMEL_CASE` | `userId` (default) |
| `SNAKE_CASE` | `user_id` |
| `UPPER_CAMEL_CASE` | `UserId` |

---

## 🔹 Date/Time

```java
// JavaTimeModule — ISO-8601 для java.time
mapper.registerModule(new JavaTimeModule());
mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
```

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
```

### ❌ Без JavaTimeModule
`Instant` → число milliseconds.

### ✅ С модулем
`"2024-01-15T10:30:00Z"`

---

## 🔹 ObjectMapper конфигурация

```java
@Bean
ObjectMapper objectMapper() {
    return JsonMapper.builder()
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
        .enable(SerializationFeature.INDENT_OUTPUT)
        .addModule(new JavaTimeModule())
        .build();
}
```

| Feature | Эффект |
|---------|--------|
| `FAIL_ON_UNKNOWN_PROPERTIES=false` | Игнор лишних полей JSON |
| `INDENT_OUTPUT` | Pretty print (dev) |

---

## 🔹 JsonNode — динамический JSON

```java
JsonNode root = mapper.readTree(json);
String name = root.path("user").path("name").asText();
if (root.get("optional").isNull()) { }
```

> [!tip] Когда
> Структура неизвестна или частично динамична (webhook payloads).

---

## 🔹 @JsonView

```java
public class Views {
    public interface Public {}
    public interface Internal extends Public {}
}

public class User {
    @JsonView(Views.Public.class)
    private String name;
    @JsonView(Views.Internal.class)
    private String ssn;
}

// Controller
@JsonView(Views.Public.class)
@GetMapping("/users/{id}")
public User get(@PathVariable Long id) { }
```

---

## 🔹 Производительность

- Переиспользовать `ObjectMapper` bean
- `@JsonIgnore` на тяжёлые lazy-поля JPA (или DTO)
- DTO вместо entity в API — меньше графа сериализации

---

## 🔹 Итог

1. Jackson — default JSON в Spring Boot.
2. Аннотации контролируют имена, ignore, format.
3. `JavaTimeModule` + `write-dates-as-timestamps=false` для ISO dates.
4. `JsonNode` — dynamic JSON.
5. Один `ObjectMapper` bean, не создавать в методах.

```
Шпаргалка Jackson:
─────────────────────────────────────────
@JsonProperty / @JsonIgnore / @JsonAlias
@JsonInclude(NON_NULL)
@JsonFormat(pattern, timezone)
JavaTimeModule + WRITE_DATES_AS_TIMESTAMPS=false
FAIL_ON_UNKNOWN_PROPERTIES=false
TypeReference<List<T>>              → generics
@JsonView                           → response views
ObjectMapper bean (singleton)
```
