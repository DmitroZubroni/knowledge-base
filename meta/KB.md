# Схема базы знаний

```
Obsidian Vault/
│
├── main.md
│
├── meta/
│   └── KB.md
│
├── algorithms/
│   ├── main Algorithms.md
│   ├── 01_Complexity.md
│   ├── 02_Arrays.md
│   ├── 03_Search.md
│   ├── 04_Sorting.md
│   ├── 05_Recursion.md
│   ├── 06_LinkedList.md
│   ├── 07_Stack_Queue.md
│   ├── 08_HashTable.md
│   ├── 09_Trees.md
│   ├── 10_Graphs.md
│   ├── 11_Combinatorics.md
│   └── 12_Greedy_DP.md
│
├── interviews/
│   ├── Interviews.md
│   ├── java-core/
│   │   ├── 01_Memory_JVM.md
│   │   ├── 02_Collections.md
│   │   ├── 03_Multithreading.md
│   │   ├── 04_Immutability.md
│   │   ├── 05_Exceptions.md
│   │   ├── 06_IO_NIO.md
│   │   ├── 07_Streams.md
│   │   └── 08_Generics.md
│   ├── spring-jpa/
│   │   ├── 09_Spring.md
│   │   ├── 10_SpringBoot.md
│   │   ├── 11_Transactional.md
│   │   └── 12_JPA_Hibernate.md
│   ├── database/
│   │   └── 13_SQL_DB.md
│   ├── architecture/
│   │   ├── 14_OOP_SOLID.md
│   │   ├── 15_Patterns.md
│   │   ├── 16_Microservices.md
│   │   └── 17_SystemDesign.md
│   └── tools/
│       ├── 18_HTTP_Network.md
│       ├── 19_Docker.md
│       ├── 20_Git.md
│       └── 21_Maven_Build.md
│
├── java/
│   ├── main Java.md  →  JAVA | Spring_Index | Library
│   │
│   ├── programming language/
│   │   ├── JAVA.md
│   │   ├── 01_Basics.md
│   │   ├── 02_OOP_Core.md
│   │   ├── 03_OOP_Advanced.md
│   │   ├── 04_OOP_Principles.md
│   │   ├── 05_Exceptions.md
│   │   ├── 06_Collections.md
│   │   ├── 07_Generics.md
│   │   ├── 08_Functional.md
│   │   ├── 09_Multithreading.md
│   │   ├── 10_SOLID.md
│   │   ├── 11_JVM_Internals.md
│   │   └── 12_Design_Patterns.md   ← GoF + Spring mapping
│   │
│   └── framework and library/
│       ├── framework/
│       │   ├── Spring_Index.md
│       │   ├── Spring_Core.md
│       │   ├── Spring_Boot.md
│       │   ├── Spring_Web_REST.md
│       │   ├── Hibernate.md
│       │   ├── Spring_Data_JPA.md
│       │   ├── Spring_Security.md
│       │   └── Spring_Test.md
│       └── library/
│           ├── Library.md
│           ├── Jackson.md
│           ├── Lombok.md
│           └── MapStruct.md
│
├── data base/
│   ├── main SQL DB.md  →  SQL | NoSQL_DB
│   ├── SQL data base/
│   │   ├── SQL.md
│   │   ├── PostgreSQL.md
│   │   ├── MySQL.md
│   │   ├── JDBC_JPA.md
│   │   └── sql/ ...
│   └── NOSQL/
│       ├── NoSQL_DB.md
│       ├── Redis.md
│       └── Mongo.md
│
├── tools/
│   ├── main Tools.md
│   ├── kubernetes/
│   │   ├── main Kubernetes.md
│   │   └── info/
│   │       ├── 1 Kubernetes Теория и архитектура.md
│   │       ├── 2 Kubernetes Workloads.md
│   │       └── 3 Kubernetes Networking и Ingress.md
│   ├── docker/
│   │   ├── main Docker.md
│   │   └── info/
│   │       ├── 1 Docker Теория и термины.md
│   │       ├── 2 Docker Команды.md
│   │       ├── 3 Docker Dockerfile.md
│   │       ├── 4 Docker Hub и Registry.md
│   │       └── 5 Docker Compose.md
│   └── tools list/
│       ├── Git.md
│       ├── Nginx.md
│       ├── Gradle.md
│       ├── Maven.md
│       ├── Kafka.md
│       ├── JUnit_Mockito.md
│       ├── Flyway.md
│       ├── Logging_SLF4J_Logback.md
│       └── GitHub_Actions.md
│
├── linux/
│   ├── main Linux.md
│   ├── info/
│   │   ├── 1 Знакомство_с_Linux.md
│   │   ├── 2 Права_доступа_и_процессы.md
│   │   ├── 3 Управление_сервисами_и_bash.md
│   │   ├── 4 Сеть_и_SSH.md
│   │   ├── 5 Основы_безопасности.md
│   │   ├── 6 Работа_с_устройствами.md
│   │   └── 7 Веб_серверы_и_утилиты.md
│   └── terminal/
│       ├── comand.md
│       └── Terminal Linux Remote Access.md
│
└── internet networks/
    ├── main Internet Networks.md
    ├── api/
    │   ├── main API.md
    │   ├── Rest Api.md
    │   ├── OAuth2_JWT.md
    │   ├── Grpc.md
    │   ├── OpenApi.md
    │   └── Swagger.md
    └── info/
        ├── 1 OSI TCPIP.md
        ├── 2 HTTP HTTPS.md
        ├── 3 DNS.md
        ├── 4 DHCP.md
        ├── 5 Devices.md
        ├── 6 TLS и mTLS.md
        ├── 7 WebSockets.md
        └── 8 Load balancing.md
```

**Связь разделов с корня:** `main.md` → `[[main Java]]` `[[main SQL DB]]` `[[main Algorithms]]` `[[Interviews]]` `[[main Tools]]` `[[main Linux]]` `[[main Internet Networks]]`

**Типы файлов:** `main *.md` — навигация · остальные — конспекты

---

## Граф связей (кровеносная система)

**Принцип:** только **иерархия снизу вверх** — без боковых ссылок между соседними конспектами в шапке. Тогда Graph View в Obsidian — дерево, а не «паутина».

```
                    [[main]]                    ← корень (единственная точка входа)
        ┌───────────┼───────────┬───────────┬───────────┐
   [[main Java]] [[main SQL DB]] [[main Tools]] [[main Linux]] ...
        │               │
   [[JAVA]]    [[main SQL DB]]              ← прослойка (SQL-ветка)
        │          ┌────┴────┐
   [[06_Collections]]  [[SQL]] → [[SQL_DDL]]
```

| Уровень | Файлы | Связи в шапке |
|---------|--------|----------------|
| **0 — корень** | `main.md` | → все `main *` (ветки) |
| **1 — ветка** | `main Java`, `main Internet Networks`, … | → только `[[main]]` |
| **2 — прослойка** | `main SQL DB`, `JAVA`, `Spring_Index` | → `main` + родительская ветка (+ при необходимости mid-layer) |
| **3 — конспект** | `06_Collections`, `Spring_Core`, … | → полная цепочка вверх |

**Примеры цепочек (только в `[!abstract]`):**

- Java: `[[main]] | [[main Java]] | [[JAVA]]`
- Spring: `[[main]] | [[main Java]] | [[Spring_Index]]`
- SQL: `[[main]] | [[main SQL DB]] | [[main SQL DB]] | [[SQL]]`
- Tools / Linux / Networks: `[[main]] | [[main …]]`

**Где ещё бывают ссылки:** в **теле hub-файлов** (таблицы, порядок изучения, списки) — звезда «hub → дети». Порядок изучения — текстом в hub, не в шапке каждого листа.

**Эталон связей:** `java/programming language/06_Collections.md`  
**Эталон hub:** `java/programming language/JAVA.md`, `java/main Java.md`

---

## Теги (лимфатическая система)

Теги **не ведут по графу** — они маркируют смысл для фильтрации и поиска.

| Тип тега | Примеры | Зачем |
|----------|---------|--------|
| Раздел | `#java`, `#database`, `#docker` | Область знаний |
| Роль файла | `#hub`, `#layer-main`, `#layer-mid`, `#конспект` | Тип узла в vault |
| Микротема | `#collections`, `#ioc`, `#ddl` | Точечный фильтр |
| Уровень | `#middle`, `#basics` | Сложность |
| Статус (опционально) | `#черновик`, `#в-работе`, `#готово` | Прогресс, когда понадобится |

**Эталон тегов:** `docker/info/2 Docker Команды.md`

---

## Стиль конспекта

**Язык:** русский. Термины на английском.

**Шапка (конспект):**
```markdown
# NN — Тема (English)

> **Теги:** #раздел #тема1 #тема2

> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]]
```

**Шапка (hub):**
```markdown
> **Теги:** #java #hub #programming
> [!abstract] Навигация
> [[main]] | [[main Java]]
```

**Шапка (ветка `main *`):**
```markdown
> [!abstract] Навигация
> [[main]]
```
Дочерние hub — списком в теле файла (`[[JAVA]]`, …).

**Связи:** только цепочка **вверх** до `main`. Смежные темы — через теги или раздел «См. также» в теле, не в шапке.

**Теги:** раздел + микротема + `#конспект` / `#hub`. Статус прогресса — отдельно, если нужен (`#черновик` и т.д.).

**Тело:**
- разделы: `## 🔹 Название`
- между блоками: `---`
- определения: `> [!note]`
- советы: `> [!tip]` · предупреждения: `> [!warning]`
- схемы: ASCII в ` ``` `
- сравнения: таблицы `| A | B |`
- код: ` ```java ` / `sql` / `yaml`
- плохо/хорошо: `### ❌` и `### ✅`
- в конце: `## 🔹 Итог` + шпаргалка в ` ``` `

**Hub** (`main Java`, `main Algorithms`): таблица файлов + порядок `[[тема]] ↓ [[тема]]`, без кода.
