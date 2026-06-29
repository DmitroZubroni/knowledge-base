# 14 — JPMS (Java Platform Module System)

> **Теги:** #java #programming #jpms #modules #конспект

> [!abstract] Связи
> [[main]] | [[main Java]]

---

## 🔹 module-info.java

```java
module com.example.app {
  requires java.sql;
  exports com.example.api;
  opens com.example.entity to org.hibernate.orm.core;
}
```

---

## 🔹 Когда нужен JPMS

- большие библиотеки с жёсткими API границами
- jlink runtime
- строгая инкапсуляция

Spring Boot чаще работает без module-path (classpath).

---

## 🔹 Итог

```
Шпаргалка JPMS:
─────────────────────────────────────────
requires / exports / opens
module-info.java
opens needed for reflection (Spring/Hibernate)
classpath mode common in Boot
```
