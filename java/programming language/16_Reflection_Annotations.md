# 16 — Reflection и Annotations

> **Теги:** #java #programming #reflection #annotations #spring #конспект

> [!abstract] Связи
> [[main]] | [[main Java]]

---

## 🔹 Reflection API

```java
Class<?> c = Class.forName("com.example.UserService");
var m = c.getMethod("findById", Long.class);
Object result = m.invoke(c.getDeclaredConstructor().newInstance(), 1L);
```

---

## 🔹 Annotations

- `@Retention(RUNTIME)` — видна через reflection
- `@Target` — где можно ставить

---

## 🔹 Spring под капотом

- classpath scan `@Component`
- DI resolution `@Autowired`
- request mapping `@RequestMapping`
- AOP proxies `@Transactional`

---

## 🔹 Итог

```
Шпаргалка Reflection:
─────────────────────────────────────────
Class/Method/Field API
invoke / getAnnotation
RUNTIME retention
Spring uses reflection + proxies
```
