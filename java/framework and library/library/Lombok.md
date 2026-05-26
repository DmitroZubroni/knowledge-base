# Lombok

> **Теги:** #java #library #lombok #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_Library]]

---

## 🔹 Что такое Lombok

> [!note] Определение
> **Lombok** — annotation processor: генерирует bytecode (getters, constructors, builders) на этапе **компиляции**.

```gradle
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
```

> [!tip] IDE
> Плагин Lombok обязателен — иначе «cannot find symbol» в IDE при нормальной компиляции.

---

## 🔹 Основные аннотации

```java
@Getter
@Setter
@ToString(exclude = "password")
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @EqualsAndHashCode.Include
    private Long id;
    private String email;
}

@NoArgsConstructor
@AllArgsConstructor
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;  // final → в конструктор
}

@Data  // @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
public class OrderDto {
    private Long id;
    private String status;
}
```

---

## 🔹 Immutable objects

```java
@Value  // final class, all fields private final, getters only
public class Money {
    BigDecimal amount;
    String currency;
}

@Builder
@Getter
public class Order {
    private final Long id;
    @Builder.Default
    private final int quantity = 1;

    public static Order copy(Order o) {
        return o.toBuilder().build();
    }
}
```

---

## 🔹 Полезные аннотации

```java
@Slf4j
@Service
public class PaymentService {
    public void pay() {
        log.info("Processing payment");
    }
}

@Cleanup("close")
InputStream in = new FileInputStream("file.txt");  // auto-close

@SneakyThrows
void readFile() {
    Files.readString(Path.of("x"));  // checked exception без throws
}

public void setEmail(@NonNull String email) { }
```

---

## 🔹 Lombok + Spring

```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
}
// = constructor injection без boilerplate
```

> [!warning] @EqualsAndHashCode на @Entity
> Использовать **только `id`** (или не генерировать) — иначе lazy-loaded поля ломают equals/hashCode.

```java
@Entity
@Getter @Setter
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @EqualsAndHashCode.Include
    @Id
    private Long id;
}
```

---

## 🔹 Подводные камни

| Проблема | Решение |
|----------|---------|
| `@Data` на `@Entity` | ❌ Избегать — equals на все поля + lazy |
| `@Builder` + наследование | `@SuperBuilder` на parent и child |
| final fields + JPA | `@NoArgsConstructor(force=true)` для proxy |

### ❌ @Data на Entity

```java
@Data
@Entity
public class Order {
    @OneToMany List<Item> items;  // equals/hashCode → LazyInitializationException
}
```

---

## 🔹 Итог

1. Lombok — compile-time codegen, не runtime magic.
2. `@RequiredArgsConstructor` + `final` — идеально для Spring DI.
3. `@Builder` — сложные immutable объекты.
4. `@Slf4j` — logger без boilerplate.
5. На JPA Entity — осторожно с `@Data` / `@EqualsAndHashCode`.

```
Шпаргалка Lombok:
─────────────────────────────────────────
@RequiredArgsConstructor           → DI
@Getter @Setter / @Value           → immutable
@Builder @Builder.Default
@Data                             → DTO only, not Entity
@Slf4j
@EqualsAndHashCode(onlyExplicitlyIncluded=true) → Entity by id
@SuperBuilder                      → inheritance
annotationProcessor in Gradle
```
