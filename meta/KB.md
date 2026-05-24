# Схема базы знаний

```
Obsidian Vault/
│
├── main.md
│
├── meta/
│   └── KB.md
│
├── java/
│   ├── main Java.md  →  00_INDEX | 00_Spring_Index | 00_Library
│   │
│   ├── programming language/
│   │   ├── 00_INDEX.md
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
│       │   ├── 00_Spring_Index.md
│       │   ├── Spring_Core.md
│       │   ├── Spring_Boot.md
│       │   ├── Spring_Web_REST.md
│       │   ├── Hibernate.md
│       │   ├── Spring_Data_JPA.md
│       │   ├── Spring_Security.md
│       │   └── Spring_Test.md
│       └── library/
│           ├── 00_Library.md
│           ├── Jackson.md
│           ├── Lombok.md
│           └── MapStruct.md
│
├── data base/
│   ├── main Data Base.md  →  00_SQL_DB | 00_NoSQL_DB
│   ├── SQL data base/
│   │   ├── 00_SQL.md
│   │   ├── 00_PostgreSQL.md
│   │   ├── JDBC_JPA.md
│   │   └── sql/ ...
│   └── NOSQL/
│       ├── 00_NoSQL_DB.md
│       ├── Redis.md
│       └── Mongo.md
│
├── api/
│   ├── main API.md
│   ├── Rest Api.md
│   ├── OAuth2_JWT.md
│   ├── Grpc.md
│   ├── OpenApi.md
│   └── Swagger.md
│
├── tools/
│   ├── main Tools.md
│   └── tools list/
│       ├── Git.md
│       ├── Nginx.md
│       ├── Gradle.md
│       ├── Kafka.md
│       ├── JUnit_Mockito.md
│       ├── Flyway.md
│       ├── Logging_SLF4J_Logback.md
│       └── GitHub_Actions.md
│
├── docker/
│   ├── main Docker.md
│   └── info/
│       ├── 1 Docker Теория и термины.md
│       ├── 2 Docker Команды.md
│       ├── 3 Docker Dockerfile.md
│       ├── 4 Docker Hub и Registry.md
│       └── 5 Docker Compose.md
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
    └── info/
        ├── 0 general info.md
        ├── 1 OSI TCPIP.md
        ├── 2 HTTP HTTPS.md
        ├── 3 DNS.md
        ├── 4 DHCP.md
        └── 5 Devices.md
```

**Связь разделов с корня:** `main.md` → `[[main Java]]` `[[main Data Base]]` `[[main API]]` `[[main Tools]]` `[[main Docker]]` `[[main Linux]]` `[[main Internet Networks]]`

**Типы файлов:** `main *.md` и `00_*` — навигация · остальные — конспекты

**Связи:** `[[Имя_файла]]` без пути. В начале конспекта — hub и соседние темы.

**Эталон:** `java/programming language/06_Collections.md`, `00_INDEX.md`

---

## Стиль конспекта

**Язык:** русский. Термины на английском.

**Шапка:**
```markdown
# NN — Тема (English)

> [!abstract] Связи
> [[00_INDEX]] | [[сосед1]] | [[сосед2]]
```

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

**Hub** (`00_INDEX`, `main Java`): таблица файлов + порядок `[[тема]] ↓ [[тема]]`, без кода.
