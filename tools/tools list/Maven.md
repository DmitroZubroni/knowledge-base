# Maven

> **Теги:** #tools #maven #build #конспект

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Maven vs Gradle

| | Maven | Gradle |
|---|-------|--------|
| Конфиг | `pom.xml` | `build.gradle.kts` |
| Модель | Convention over configuration | Гибче, программируемый DSL |
| Производительность | Стабильно, но медленнее | Daemon + cache быстрее |

---

## 🔹 Базовый `pom.xml`

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
  </parent>

  <groupId>com.example</groupId>
  <artifactId>demo</artifactId>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
</project>
```

---

## 🔹 Lifecycle

`validate → compile → test → package → verify → install → deploy`

```bash
mvn clean package
mvn test
mvn install
```

---

## 🔹 Итог

```
Шпаргалка Maven:
─────────────────────────────────────────
pom.xml
mvn clean package
dependency scope: compile/test/provided
spring-boot-starter-parent
~/.m2/repository
```
