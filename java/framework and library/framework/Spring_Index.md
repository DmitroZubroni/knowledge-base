# Spring — Index

> **Теги:** #spring #hub #framework  

> [!abstract] Навигация
> [[main]] | [[main Java]]

## 🗺️ Карта файлов

| Файл | Тема | Уровень |
|------|------|---------|
| [[Spring_Core]] | IoC, DI, контейнер, bean lifecycle, scopes, конфигурация | Middle |
| [[Spring_Boot]] | Auto-configuration, starters, properties, Actuator | Middle |
| [[Hibernate]] | ORM, mapping, Session, JPQL, кэш, locking | Middle+ |
| [[Spring_Data_JPA]] | Repository, queries, transactions, projections | Middle+ |
| [[Spring_Web_REST]] | MVC, REST, validation, exception handling, WebClient | Middle+ |
| [[Spring_Security]] | Filter chain, auth, JWT, OAuth2, method security | Middle+ |
| [[Spring_Test]] | Unit, sliced tests, MockMvc, Testcontainers | Middle+ |

---

## 📚 Рекомендуемый порядок изучения

```
[[Spring_Core]]
    ↓
[[Spring_Boot]]
    ↓
[[Hibernate]]
    ↓
[[Spring_Data_JPA]]
    ↓
[[Spring_Web_REST]]
    ↓
[[Spring_Security]]
    ↓
[[Spring_Test]]
```

> [!tip] Параллельные ветки
> **Web-ветка:** Boot → Web → Security → Test  
> **Data-ветка:** Boot → Hibernate → Data JPA → Test  
> Hibernate и Web можно изучать параллельно после Boot.

---

## 🔗 Карта зависимостей

```
Spring_Index
    │
    ├──► Spring_Core ──► Spring_Boot
    │                         │
    │              ┌──────────┴──────────┐
    │              ▼                     ▼
    │         Hibernate            Spring_Web_REST
    │              │                     │
    │              ▼                     ▼
    │      Spring_Data_JPA      Spring_Security
    │              │                     │
    │              └──────────┬──────────┘
    │                         ▼
    │                   Spring_Test
    │
    └── внешние:
         [[JDBC_JPA]]          ← Hibernate, Spring_Data_JPA
         [[PostgreSQL]]        ← Hibernate
         [[OAuth2_JWT]]        ← Spring_Security
         [[Rest Api]]          ← Spring_Web_REST
         [[OpenApi]]           ← Spring_Web_REST
         [[JUnit_Mockito]]     ← Spring_Test
         [[Flyway]]            ← Spring_Data_JPA
         [[12_Design_Patterns]] ← Spring_Core
```

---

## 📖 Глоссарий

| Термин | Определение |
|--------|-------------|
| **IoC** | Inversion of Control — контейнер управляет созданием и связями объектов |
| **DI** | Dependency Injection — внедрение зависимостей через конструктор, setter или field |
| **Bean** | Объект, управляемый Spring-контейнером |
| **ApplicationContext** | Расширенный контейнер: events, i18n, AOP, auto-wiring |
| **Auto-configuration** | Условная настройка бинов на основе classpath и properties |
| **JPA** | Java Persistence API — спецификация ORM |
| **Hibernate** | Реализация JPA (и расширения поверх спецификации) |
| **Repository** | Абстракция доступа к данным в Spring Data |
| **DispatcherServlet** | Front Controller Spring MVC |
| **SecurityFilterChain** | Цепочка фильтров для аутентификации и авторизации |
