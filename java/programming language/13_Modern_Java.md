# 13 — Modern Java (17–21+)

> **Теги:** #java #programming #modern-java #records #sealed #patterns #конспект

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_JAVA]]

---

## 🔹 Text blocks

```java
String json = """
  {"id": 1, "name": "Alice"}
""";
```

---

## 🔹 records

```java
public record UserDto(Long id, String email) {}
```

---

## 🔹 sealed + pattern matching

```java
sealed interface Result permits Ok, Err {}
record Ok(String v) implements Result {}
record Err(String e) implements Result {}
```

---

## 🔹 switch expressions

```java
String x = switch(day) {
  case SATURDAY, SUNDAY -> "weekend";
  default -> "work";
};
```

---

## 🔹 Итог

```
Шпаргалка Modern Java:
─────────────────────────────────────────
text blocks
record for DTO
sealed hierarchies
pattern matching
switch expressions
```
