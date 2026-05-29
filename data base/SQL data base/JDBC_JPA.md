# JDBC / JPA (Data Access)

> **Теги:** #database #jdbc #jpa #hikari #middle #конспект  

> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[main SQL DB]] | [[SQL]]

---

## 🔹 JDBC — как работает

```
Application
    ↓
JDBC Driver (postgresql.jar)
    ↓
TCP → PostgreSQL
```

**JDBC URL:** `jdbc:postgresql://localhost:5432/mydb`

| Компонент | Роль |
|-----------|------|
| `Connection` | Сессия с БД |
| `Statement` / `PreparedStatement` | SQL запрос |
| `ResultSet` | Строки результата |

| | DriverManager | DataSource |
|---|---------------|------------|
| Создание connection | Каждый раз новый | Из pool |
| Production | ❌ | ✅ HikariCP |

---

## 🔹 Statement types

### ❌ Statement — конкатенация SQL

```java
stmt.executeQuery("SELECT * FROM users WHERE email = '" + email + "'");
// SQL injection
```

### ✅ PreparedStatement

```java
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE email = ? AND active = ?");
ps.setString(1, email);
ps.setBoolean(2, true);
ResultSet rs = ps.executeQuery();
```

**Преимущества:** параметры экранируются, precompile на сервере.

### CallableStatement

```java
CallableStatement cs = conn.prepareCall("{call calculate_bonus(?)}");
cs.setLong(1, userId);
cs.execute();
```

---

## 🔹 ResultSet и маппинг

```java
while (rs.next()) {
    long id = rs.getLong("id");
    String email = rs.getString("email");
}
```

**RowMapper (Spring JDBC):**

```java
public class UserRowMapper implements RowMapper<User> {
    @Override
    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
        return new User(rs.getLong("id"), rs.getString("email"));
    }
}

List<User> users = jdbcTemplate.query(
    "SELECT id, email FROM users WHERE active = ?",
    new UserRowMapper(),
    true
);
```

---

## 🔹 Connection Pool — зачем

> [!note] Стоимость Connection
> TCP handshake + SSL + auth + memory на стороне БД — **дорого** на каждый запрос.

**Pool:** держит N открытых соединений, выдаёт на время запроса, возвращает обратно.

---

## 🔹 HikariCP — конфигурация

> [!tip] Default в Spring Boot
> HikariCP — быстрый, минималистичный pool.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000      # ms ждать свободный connection
      idle-timeout: 600000           # закрыть idle connection
      max-lifetime: 1800000            # ротация (меньше timeout БД)
```

| Параметр | Default | Production |
|----------|---------|------------|
| `maximumPoolSize` | 10 | по нагрузке (см. формулу) |
| `minimumIdle` | = max | 5–10 или = max |
| `connectionTimeout` | 30s | не увеличивать бесконечно |
| `maxLifetime` | 30m | < DB wait_timeout |

**Формула Hikari (ориентир):**
```
poolSize = (CPU cores * 2) + effective_spindle_count
```
Для SSD `effective_spindle_count` ≈ 1.

> [!warning] Pool too large
> Слишком много connections → contention на БД, context switching.  
> Pool too small → `connectionTimeout`, медленные запросы в очереди.

---

## 🔹 JPA — концепция (без Hibernate)

> [!note] JPA
> Спецификация `jakarta.persistence.*` — API, не реализация.

```
EntityManagerFactory (один на приложение)
    ↓
EntityManager (на транзакцию / запрос)
    ↓
PersistenceContext (кэш 1st level)
```

**Entity states:**

```
transient ──persist()──► persistent ──detach()──► detached
                              │
                         remove()
                              ▼
                           removed
```

---

## 🔹 JDBC vs JPA vs Spring Data

| | JDBC | JPA | Spring Data |
|---|------|-----|-------------|
| SQL контроль | Полный | JPQL + native | Derived + @Query |
| Boilerplate | Высокий | Средний | Низкий |
| Portability | SQL-диалект | JPA provider | JPA |
| Learning curve | Низкий | Высокий | Средний |
| Когда | Reports, batch, сложный SQL | Domain model | CRUD, стандартные API |

---

## 🔹 Spring JdbcTemplate

```java
@Repository
@RequiredArgsConstructor
public class UserJdbcRepository {
    private final JdbcTemplate jdbc;

    public Optional<User> findById(long id) {
        return jdbc.query(
            "SELECT id, email FROM users WHERE id = ?",
            rs -> rs.next() ? Optional.of(map(rs)) : Optional.empty(),
            id
        );
    }

    public int updateEmail(long id, String email) {
        return jdbc.update("UPDATE users SET email = ? WHERE id = ?", email, id);
    }

    public int[] batchInsert(List<User> users) {
        return jdbc.batchUpdate(
            "INSERT INTO users (email) VALUES (?)",
            users,
            users.size(),
            (ps, user) -> ps.setString(1, user.getEmail())
        );
    }
}
```

**NamedParameterJdbcTemplate:**

```java
MapSqlParameterSource params = new MapSqlParameterSource()
    .addValue("email", email)
    .addValue("active", true);
jdbc.query("SELECT * FROM users WHERE email = :email AND active = :active", params, mapper);
```

---

## 🔹 Итог

1. JDBC — низкоуровневый доступ; всегда `PreparedStatement`.
2. DataSource + HikariCP — обязательно в production.
3. JPA — ORM API; Hibernate — реализация в Boot.
4. Spring Data — поверх JPA для репозиториев.
5. JdbcTemplate — когда нужен полный контроль SQL.

```
Шпаргалка JDBC/JPA:
─────────────────────────────────────────
jdbc:postgresql://host:port/db
PreparedStatement + ?              → anti SQL injection
HikariCP maximumPoolSize           → tune by load
JdbcTemplate.query/update/batchUpdate
EntityManager + PersistenceContext
transient/persistent/detached      → entity states
ddl-auto: validate + Flyway        → prod
Spring Data JpaRepository          → CRUD abstraction
```
