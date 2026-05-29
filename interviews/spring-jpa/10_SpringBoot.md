# Spring Boot

> **Теги:** #interviews #spring #spring-boot #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 @SpringBootApplication

**@SpringBootApplication** — мета-аннотация, которая объединяет три аннотации:

```java
@SpringBootApplication
// = @SpringBootConfiguration
// + @EnableAutoConfiguration
// + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Составляющие

| Аннотация | Назначение |
|-----------|------------|
| **@SpringBootConfiguration** | Обозначает класс как конфигурацию (аналог @Configuration) |
| **@EnableAutoConfiguration** | Включает автоматическую конфигурацию |
| **@ComponentScan** | Сканирует компоненты в текущем пакете |

### @EnableAutoConfiguration

Автоматически конфигурирует приложение на основе classpath.

```java
// Если в classpath есть H2 → Spring автоматически создаёт DataSource
// Если в classpath есть Spring MVC → автоматически конфигурирует DispatcherServlet
```

---

## 🔹 Spring Boot Starters

**Starter** — набор зависимостей для быстрого старта.

### Примеры

| Starter | Зависимости |
|---------|-------------|
| **spring-boot-starter-web** | Spring MVC, Tomcat, JSON |
| **spring-boot-starter-data-jpa** | Spring Data JPA, Hibernate |
| **spring-boot-starter-security** | Spring Security |
| **spring-boot-starter-test** | JUnit, Mockito, Spring Test |

### BOM (Bill of Materials)

**BOM** — управление версиями зависимостей.

```xml
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
```

Старты уже включают BOM, поэтому версии указывать не нужно:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- версия не нужна! -->
</dependency>
```

---

## 🔹 Auto Configuration

**Auto Configuration** — автоматическая настройка beans на основе classpath.

### Как работает

```
1. @EnableAutoConfiguration
2. Сканирует classpath
3. Загружает candidate configurations (META-INF/spring.factories или META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports)
4. Создаёт beans если условия выполнены
```

### Условия (Conditional)

| Аннотация | Условие |
|-----------|---------|
| **@ConditionalOnClass** | Класс в classpath |
| **@ConditionalOnMissingBean** | Bean отсутствует |
| **@ConditionalOnProperty** | Свойство в application.properties |
| **@ConditionalOnWebApplication** | Web приложение |

### Пример auto-configuration

```java
@Configuration
@ConditionalOnClass(DataSource.class)  // если DataSource в classpath
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean  // если bean не создан вручную
    public DataSource dataSource(DataSourceProperties properties) {
        return createDataSource(properties);
    }
}
```

### Отключение auto-configuration

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class Application {
}
```

Или в `application.properties`:

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## 🔹 Embedded Tomcat

**Embedded Tomcat** — встроенный сервер в Spring Boot.

### Как работает

```java
// spring-boot-starter-web включает Tomcat
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        // Tomcat запускается автоматически на порту 8080
    }
}
```

### Конфигурация

```properties
# application.properties
server.port=8080
server.servlet.context-path=/api
```

### Смена сервера

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

---

## 🔹 WAR vs JAR

### JAR (Java Archive)

**Spring Boot по умолчанию** — исполняемый JAR с embedded Tomcat.

```xml
<packaging>jar</packaging>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

**Преимущества:**
- Самодостаточный (включает Tomcat)
- Простой запуск: `java -jar app.jar`
- Идеален для microservices и Docker

### WAR (Web Application Archive)

Традиционный формат для деплоя на внешний сервер.

```xml
<packaging>war</packaging>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>  <!-- Tomcat предоставляется сервером -->
</dependency>
```

Нужно унаследовать `SpringBootServletInitializer`:

```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}
```

**Когда использовать:**
- Деплой на существующий сервер (Tomcat, WebLogic)
- Требования корпоративной среды

### Сравнение

| Характеристика | JAR | WAR |
|----------------|-----|-----|
| Сервер | Embedded (Tomcat) | Внешний |
| Самодостаточный | Да | Нет |
| Запуск | `java -jar` | Деплой на сервер |
| Microservices | ✅ Идеально | ❌ Нет |
| Legacy | ❌ Нет | ✅ Да |

---

## 🔹 application.properties vs application.yml

### application.properties

```properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/db
spring.datasource.username=user
spring.datasource.password=password
```

### application.yml

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/db
    username: user
    password: password
```

### Profiles

```properties
# application.properties
spring.profiles.active=dev

# application-dev.properties
server.port=8080

# application-prod.properties
server.port=80
```

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
# application-dev.yml
server:
  port: 8080

---
# application-prod.yml
server:
  port: 80
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **@SpringBootApplication** = @Configuration + @EnableAutoConfiguration + @ComponentScan
> - **Starter** — набор зависимостей, BOM управляет версиями
> - **Auto Configuration** — автоматическая настройка по classpath
> - **Embedded Tomcat** — встроенный сервер в JAR
> - **JAR** — самодостаточный (microservices), **WAR** — внешний сервер (legacy)
> - **Profiles** — разные конфигурации для dev/prod

```
@SpringBootApplication:
@SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan

Starter:
набор зависимостей + BOM (версии управляются автоматически)

Auto Configuration:
@EnableAutoConfiguration → сканирует classpath → создаёт beans
@ConditionalOnClass, @ConditionalOnMissingBean

Embedded Tomcat:
spring-boot-starter-web включает Tomcat
server.port=8080

WAR vs JAR:
JAR → embedded Tomcat, самодостаточный
WAR → внешний сервер, legacy

Profiles:
spring.profiles.active=dev
application-dev.yml / application-prod.yml
```
