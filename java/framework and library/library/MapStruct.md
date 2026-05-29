# MapStruct

> **Теги:** #java #library #mapstruct #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[Library]]

---

## 🔹 Что такое MapStruct

> [!note] Определение
> **MapStruct** — annotation processor: генерирует **compile-time** мапперы (не reflection как ModelMapper).

| | MapStruct | ModelMapper |
|---|-----------|-------------|
| Время | Compile | Runtime |
| Скорость | Как hand-written | Медленнее |
| Ошибки | Compile-time | Runtime |

```gradle
implementation 'org.mapstruct:mapstruct:1.6.0'
annotationProcessor 'org.mapstruct:mapstruct-processor:1.6.0'
```

---

## 🔹 Базовый маппинг

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(User entity);
    User toEntity(UserDto dto);
    List<UserDto> toDtoList(List<User> entities);
}
```

> [!note] Одинаковые имена полей
> Маппятся автоматически: `email` → `email`.

```java
@Mapping(source = "createdAt", target = "registeredAt")
@Mapping(target = "password", ignore = true)
UserDto toDto(User user);
```

---

## 🔹 Конфигурация маппингов

```java
@Mapping(target = "fullName",
    expression = "java(user.getFirstName() + \" \" + user.getLastName())")
@Mapping(source = "birthDate", dateFormat = "dd-MM-yyyy")
UserDto toDto(User user);

@Mapping(target = "city", source = "address.city")
OrderDto toDto(Order order);
```

**Patch / update:**

```java
@Mapping(target = "id", ignore = true)
void updateUserFromDto(UserDto dto, @MappingTarget User user);
```

> [!tip] @MappingTarget
> Обновляет существующий объект — partial update в PUT/PATCH.

---

## 🔹 Коллекции

```java
List<OrderDto> toDtoList(List<Order> orders);  // auto

@IterableMapping(qualifiedByName = "toSummary")
List<OrderSummaryDto> toSummaries(List<Order> orders);

@Named("toSummary")
OrderSummaryDto toSummary(Order order);
```

---

## 🔹 Конвертеры и кастомная логика

```java
@Mapper(componentModel = "spring", uses = { MoneyMapper.class })
public interface OrderMapper {
    OrderDto toDto(Order order);

    default String mapStatus(OrderStatus status) {
        return status.name();
    }
}

@Mapper
public interface MoneyMapper {
    @Named("toMoneyDto")
    MoneyDto toDto(Money money);
}

// qualifiedByName
@Mapping(source = "total", target = "total", qualifiedByName = "toMoneyDto")
OrderDto toDto(Order order);
```

```java
@Mapper(componentModel = "spring", uses = TaxService.class)
public interface InvoiceMapper {
    @Mapping(target = "tax", expression = "java(taxService.calculate(order))")
    InvoiceDto toDto(Order order, @Context TaxContext ctx);
}
```

---

## 🔹 MapStruct + Lombok

> [!warning] Порядок processors
> Lombok должен генерировать getters **до** MapStruct.

```gradle
annotationProcessor 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
annotationProcessor 'org.mapstruct:mapstruct-processor:1.6.0'
```

---

## 🔹 Тестирование маппера

```java
@Test
void mapsCorrectly() {
    UserMapper mapper = Mappers.getMapper(UserMapper.class);
    User user = new User(1L, "a@b.com");
    UserDto dto = mapper.toDto(user);
    assertThat(dto.getEmail()).isEqualTo("a@b.com");
}
```

> [!tip] Без Spring
> Unit-тест маппера — быстрый, без контекста.

---

## 🔹 Итог

1. MapStruct — compile-time, type-safe маппинг.
2. `@Mapper(componentModel = "spring")` — bean в контексте.
3. `@Mapping` — rename, ignore, expression.
4. `@MappingTarget` — update existing entity.
5. Lombok binding — правильный порядок annotation processors.

```
Шпаргалка MapStruct:
─────────────────────────────────────────
@Mapper(componentModel = "spring")
@Mapping(source, target) / ignore = true
@MappingTarget                     → patch update
uses = { OtherMapper.class }
@Named + qualifiedByName
Mappers.getMapper(X.class)         → unit test
lombok-mapstruct-binding           → Lombok first
```
