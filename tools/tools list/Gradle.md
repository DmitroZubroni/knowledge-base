# Gradle

> [!abstract] Связи
> [[main Tools]] | [[Spring_Boot]]

---

## 🔹 Gradle vs Maven

| | Gradle | Maven |
|---|--------|-------|
| Синтаксис | Kotlin/Groovy DSL | XML |
| Производительность | Daemon, build cache, incremental | Стабильный, медленнее на больших |
| Гибкость | Высокая (tasks, plugins) | Convention over configuration |
| Multi-module | ✅ | ✅ |
| Spring Boot | Отлично | Отлично |

---

## 🔹 Структура проекта

```
project/
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
├── gradlew
└── gradle/wrapper/gradle-wrapper.properties
```

```kotlin
// settings.gradle.kts
rootProject.name = "order-service"
include(":core", ":api")
```

---

## 🔹 Базовый build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
}

group = "com.example"
version = "1.0.0"

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    testImplementation("org.springframework.boot:spring-boot-starter-test")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
}

tasks.test {
    useJUnitPlatform()
}
```

| Configuration | Назначение |
|---------------|------------|
| `implementation` | Compile + runtime, не транзитивно для consumers |
| `api` | Как implementation + транзитивно (java-library) |
| `compileOnly` | Только compile (Lombok) |
| `annotationProcessor` | APT (Lombok, MapStruct) |
| `testImplementation` | Только тесты |

---

## 🔹 Dependency Management

**BOM (Spring Boot):**

```kotlin
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:3.3.0"))
    implementation("org.springframework.boot:spring-boot-starter-web")  // без версии
}
```

**Version catalog (`gradle/libs.versions.toml`):**

```toml
[versions]
spring-boot = "3.3.0"

[libraries]
spring-web = { module = "org.springframework.boot:spring-boot-starter-web" }

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
```

```kotlin
dependencies {
    implementation(libs.spring.web)
}
```

---

## 🔹 Tasks

```bash
./gradlew build          # compile + test + jar
./gradlew test
./gradlew bootRun
./gradlew bootJar
./gradlew clean
./gradlew :api:build     # конкретный модуль
```

```kotlin
tasks.register("printVersion") {
    doLast {
        println(project.version)
    }
}

tasks.build {
    dependsOn("printVersion")
}
```

---

## 🔹 Multi-module проект

```kotlin
// :api/build.gradle.kts
dependencies {
    implementation(project(":core"))
}
```

```
settings.gradle.kts:
include(":core", ":api", ":service")
```

---

## 🔹 Gradle Wrapper

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.9-bin.zip
```

> [!tip] Зачем
> Все разработчики и CI используют **одну** версию Gradle.

```bash
./gradlew wrapper --gradle-version=8.9
```

---

## 🔹 Производительность

```bash
./gradlew build --parallel --build-cache
```

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
```

| Флаг | Эффект |
|------|--------|
| `--parallel` | Параллельные модули |
| `--build-cache` | Reuse outputs |
| `daemon` | JVM остаётся живой (default on) |
| configuration-cache | Кэш конфигурации (Gradle 8+) |

---

## 🔹 Итог

1. Kotlin DSL (`build.gradle.kts`) — современный стандарт.
2. Spring Boot plugin + BOM — версии без конфликтов.
3. `implementation` vs `api` vs `annotationProcessor`.
4. Wrapper — фиксированная версия Gradle.
5. `--parallel`, build cache — ускорение CI.

```
Шпаргалка Gradle:
─────────────────────────────────────────
./gradlew build / test / bootRun / bootJar
implementation / testImplementation
annotationProcessor                  → Lombok, MapStruct
platform(spring-boot-dependencies)   → BOM
libs.versions.toml                   → version catalog
include(":module")                   → multi-module
wrapper                              → reproducible builds
--parallel --build-cache
```
