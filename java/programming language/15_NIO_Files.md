# 15 — NIO.2 / Files API

> **Теги:** #java #programming #nio #files #io #конспект

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_JAVA]]

---

## 🔹 Path + Files

```java
Path p = Path.of("logs", "app.log");
Files.createDirectories(p.getParent());
Files.writeString(p, "started
", StandardOpenOption.CREATE, StandardOpenOption.APPEND);
```

---

## 🔹 Обход дерева

```java
try (var s = Files.walk(Path.of("src"))) {
  s.filter(Files::isRegularFile).forEach(System.out::println);
}
```

---

## 🔹 Итог

```
Шпаргалка NIO.2:
─────────────────────────────────────────
Path.of
Files.readString/writeString
Files.copy/move/deleteIfExists
Files.walk/find
prefer NIO.2 over legacy File
```
