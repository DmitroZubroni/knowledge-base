# 📋 Подготовка к собеседованиям

> **Теги:** #interviews #hub #index

> [!abstract] Связи
> [[main]]

---

## 🔹 Стратегия подготовки

### Общий подход
- Начинать с фундаментальных тем (Java Core, ООП, Collections)
- Переходить к фреймворкам (Spring, JPA)
- Закреплять архитектурой и System Design
- Отдельно проработать инструменты (Git, Docker, Maven)

### Источники информации
- Официальная документация (JavaDoc, Spring Docs)
- OpenJDK source code и комментарии
- YouTube обзоры новых версий
- Medium, Baeldung, LinkedIn статьи
- Практика через лайфкодинг

---

## 🔹 Топ-темы по уровням

### Junior → Middle
- Java Core: Collections, Multithreading basics, Exceptions
- ООП: 4 столпа, SOLID (особенно SRP)
- Spring: IoC/DI, ComponentScan, basic lifecycle
- SQL: JOIN, базовые запросы, индексы
- Git: basic commands, merge, commit

### Middle → Senior
- JVM: Memory model, GC algorithms, tuning
- Concurrency: synchronized vs volatile, atomic, ExecutorService
- Spring: Transactional propagation, proxy pattern, AOP
- JPA: N+1 problem, lazy loading, locking strategies
- Architecture: Microservices vs Monolith, CAP theorem
- System Design: масштабирование, кэширование, load balancing

---

## 🔹 Структура разделов

| Раздел | Файлы | Темы |
|--------|-------|-------|
| **Java Core** | 01-08 | Memory/JVM, Collections, Multithreading, Immutability, Exceptions, IO/NIO, Streams, Generics |
| **Spring & JPA** | 09-12 | Spring Core, Spring Boot, Transactional, JPA/Hibernate |
| **Базы данных** | 13 | SQL DB, ACID, isolation levels |
| **Архитектура** | 14-17 | OOP/SOLID, Patterns, Microservices, System Design |
| **Инструменты** | 18-21 | HTTP/Network, Docker, Git, Maven/Build |

---

## 🔹 Java Core — частые вопросы

| Файл | Тема |
|------|------|
| [[01_Memory_JVM]] | Heap/Stack/Metaspace, GC алгоритмы, String Pool, примитивы vs объекты |
| [[02_Collections]] | Иерархия, ArrayList vs LinkedList (асимптотика), HashMap под капотом, equals/hashCode контракт |
| [[03_Multithreading]] | synchronized, volatile, atomic, ConcurrentHashMap, Executor, виртуальные потоки |
| [[04_Immutability]] | Immutable-класс (5 правил), String immutability, Record, ключ в HashMap |
| [[05_Exceptions]] | Иерархия, checked/unchecked, try-with-resources, throw vs throws |
| [[06_IO_NIO]] | InputStream/OutputStream, File vs Path, байтовые vs символьные, NIO, Decorator pattern |
| [[07_Streams]] | Lazy evaluation, промежуточные/терминальные, map vs flatMap, count, IntStream |
| [[08_Generics]] | extends vs super (PECS), wildcard, type erasure |

---

## 🔹 Spring & JPA — частые вопросы

| Файл | Тема |
|------|------|
| [[09_Spring]] | IoC/DI, Bean lifecycle, скопы, @Component vs @Bean, ComponentScan, Proxy |
| [[10_SpringBoot]] | Starter (BOM + auto-config), embedded Tomcat, WAR vs JAR, @SpringBootApplication |
| [[11_Transactional]] | @Transactional под капотом, propagation, self-invocation ловушка, rollback |
| [[12_JPA_Hibernate]] | N+1, EntityGraph, LazyInit, оптимистичные/пессимистичные блокировки, ID стратегии |

---

## 🔹 Базы данных — частые вопросы

| Файл | Тема |
|------|------|
| [[13_SQL_DB]] | ACID, уровни изоляции (феномены), блокировки, JOIN vs UNION, FK, constraints |

---

## 🔹 Архитектура и дизайн

| Файл | Тема |
|------|------|
| [[14_OOP_SOLID]] | 4 столпа ООП, SRP (стейкхолдер-трактовка), DIP, полиморфизм, контракт |
| [[15_Patterns]] | Singleton (thread-safe), Proxy, Builder, Adapter, Decorator (IO pattern) |
| [[16_Microservices]] | Монолит vs микросервисы, API Gateway (варианты), Saga, масштабирование |
| [[17_SystemDesign]] | System Design интервью (структура ответа), CAP, горизонтальное vs вертикальное масштабирование |

---

## 🔹 Инструменты — частые вопросы

| Файл | Тема |
|------|------|
| [[18_HTTP_Network]] | HTTP sync vs async IO, REST vs gRPC, WebClient vs RestTemplate, заголовки/тело |
| [[19_Docker]] | Image vs container, слои и кэш, COPY vs ADD, volume, layer ordering |
| [[20_Git]] | gitignore, merge vs cherry-pick, tag, diff, зачем VCS |
| [[21_Maven_Build]] | Lifecycle (package vs install), BOM, артефакты JAR vs WAR, local repo |

---

## 🔹 Итог

> [!tip] Рекомендация
> Начинайте с Java Core (01-08), затем переходите к Spring (09-12). Базы данных (13) можно параллельно с Spring. Архитектуру (14-17) изучайте после освоения фреймворков. Инструменты (18-21) используйте по мере необходимости в проектах.
