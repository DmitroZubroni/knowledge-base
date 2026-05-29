# Maven & Build

> **Теги:** #interviews #tools #maven #build #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Maven Lifecycle

**Lifecycle** — последовательность фаз сборки проекта.

### Основные фазы

| Фаза | Описание |
|------|----------|
| **validate** | Валидация проекта |
| **compile** | Компиляция исходного кода |
| **test** | Запуск тестов |
| **package** | Упаковка (JAR/WAR) |
| **install** | Установка в local repository |
| **deploy** | Установка в remote repository |

### Порядок выполнения

```
validate → compile → test → package → install → deploy
```

Если запустить `mvn install`, выполняются все предыдущие фазы.

### Примеры

```bash
# Компиляция
mvn compile

# Упаковка
mvn package

# Установка в local repo
mvn install

# Пропустить тесты
mvn install -DskipTests
```

---

## 🔹 Package vs Install

### Package

**Package** — упаковка проекта в артефакт (JAR/WAR).

```bash
mvn package
```

**Результат:**
- Создаёт артефакт в `target/`
- Не устанавливает в local repository

### Install

**Install** — установка артефакта в local repository.

```bash
mvn install
```

**Результат:**
- Создаёт артефакт в `target/`
- Устанавливает в `~/.m2/repository/`
- Другие проекты могут использовать этот артефакт

### Сравнение

| Характеристика | Package | Install |
|----------------|---------|----------|
| Создаёт артефакт | ✅ | ✅ |
| Устанавливает в local repo | ❌ | ✅ |
| Доступен другим проектам | ❌ | ✅ |

---

## 🔹 BOM (Bill of Materials)

**BOM** — POM файл для управления версиями зависимостей.

### Пример BOM

```xml
<!-- bom/pom.xml -->
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>dependencies-bom</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### Использование BOM

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>dependencies-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- версия не нужна! -->
    </dependency>
</dependencies>
```

**Преимущества:**
- Централизованное управление версиями
- Согласованные версии зависимостей
- Упрощение POM файлов

---

## 🔹 Артефакты JAR vs WAR

### JAR (Java Archive)

**JAR** — архив Java классов и ресурсов.

```xml
<packaging>jar</packaging>
```

**Когда использовать:**
- Библиотеки
- Standalone приложения (Spring Boot)
- Утилиты

### WAR (Web Application Archive)

**WAR** — архив веб-приложения.

```xml
<packaging>war</packaging>
```

**Когда использовать:**
- Веб-приложения для деплоя на сервер (Tomcat, WebLogic)
- Традиционные Java EE приложения

### Сравнение

| Характеристика | JAR | WAR |
|----------------|-----|-----|
| Сервер | Embedded (Spring Boot) | Внешний (Tomcat) |
| Структура | Любая | WEB-INF/ |
| Запуск | `java -jar` | Деплой на сервер |
| Spring Boot | ✅ Идеально | ❌ Legacy |

### Пример WAR конфигурации

```xml
<packaging>war</packaging>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <finalName>myapp</finalName>
</build>
```

---

## 🔹 Local Repository

**Local Repository** — локальное хранилище артефактов Maven.

### Расположение

```
~/.m2/repository/
```

### Структура

```
~/.m2/repository/
├── org/
│   └── springframework/
│       └── boot/
│           └── spring-boot-starter-web/
│               ├── 3.2.0/
│               │   ├── spring-boot-starter-web-3.2.0.jar
│               │   ├── spring-boot-starter-web-3.2.0.pom
│               │   └── _remote.repositories
│               ├── maven-metadata.xml
│               └── maven-metadata-local.xml
```

### Команды

```bash
# Очистить local repository
rm -rf ~/.m2/repository/

# Установить артефакт в local repo
mvn install:install-file \
  -Dfile=path/to/artifact.jar \
  -DgroupId=com.example \
  -DartifactId=my-artifact \
  -Dversion=1.0.0
```

### Remote vs Local

| Характеристика | Local | Remote |
|----------------|-------|---------|
| Расположение | `~/.m2/repository/` | Nexus, Artifactory |
| Доступ | Только локально | Команда |
| Скорость | Быстро | Медленнее (сеть) |
| Обновление | `mvn install` | Автоматически |

---

## 🔹 Dependency Scope

**Scope** — область видимости зависимости.

### Типы scope

| Scope | Описание | Пример |
|-------|----------|--------|
| **compile** (default) | Доступно везде | Spring Core |
| **provided** | Компиляция, но не упаковывается | Servlet API |
| **runtime** | Не для компиляции, только runtime | JDBC драйвер |
| **test** | Только для тестов | JUnit |
| **system** | Локальный JAR | Локальные библиотеки |

### Примеры

```xml
<dependencies>
    <!-- compile (default) -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
    </dependency>

    <!-- provided (Tomcat предоставляет) -->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <scope>provided</scope>
    </dependency>

    <!-- test -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 🔹 Multi-module Project

**Multi-module** — проект с несколькими модулями.

### Структура

```
my-project/
├── pom.xml (parent)
├── module-a/
│   ├── pom.xml
│   └── src/
├── module-b/
│   ├── pom.xml
│   └── src/
└── module-c/
    ├── pom.xml
    └── src/
```

### Parent POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>module-a</module>
        <module>module-b</module>
        <module>module-c</module>
    </modules>
</project>
```

### Child POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>module-a</artifactId>
</project>
```

### Команды

```bash
# Сборка всех модулей
mvn clean install

# Сборка конкретного модуля
mvn clean install -pl module-a

# Сборка модуля с зависимостями
mvn clean install -pl module-a -am
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Lifecycle:** validate → compile → test → package → install → deploy
> - **Package** — создаёт артефакт, **Install** — устанавливает в local repo
> - **BOM** — управление версиями зависимостей (dependencyManagement)
> - **JAR** — standalone/libraries, **WAR** — веб-приложения для сервера
> - **Local Repository:** `~/.m2/repository/`, кэш артефактов
> - **Scope:** compile (default), provided, runtime, test, system
> - **Multi-module:** parent POM с modules, child POM с parent

```
Lifecycle:
validate → compile → test → package → install → deploy

Package vs Install:
Package → создаёт артефакт в target/
Install → создаёт + устанавливает в ~/.m2/repository/

BOM:
dependencyManagement → централизованное управление версиями
import scope → импорт BOM

JAR vs WAR:
JAR → standalone, библиотеки, Spring Boot
WAR → веб-приложения, Tomcat, legacy

Local Repository:
~/.m2/repository/ → локальный кэш Maven
Remote → Nexus, Artifactory

Scope:
compile → везде (default)
provided → компиляция, не упаковывается
test → только тесты

Multi-module:
Parent POM → packaging=pom, modules
Child POM → parent ссылка
```
