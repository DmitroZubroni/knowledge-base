# Redis

> **Теги:** #nosql #redis #cache #конспект  

> [!abstract] Связи
> [[main]] | [[main Data Base]] | [[00_NoSQL_DB]]

---

z # Redis

---

## 🔹 Что такое Redis

> [!note] Определение
> **Redis** (Remote Dictionary Server) — in-memory **key-value** хранилище с rich data structures. Используется как cache, session store, message broker, rate limiter.

| Характеристика | Значение |
|----------------|----------|
| Модель | Key → Value |
| Persistence | Опционально (RDB / AOF) |
| Replication | Master-replica |
| Cluster | Sharding по hash slots |
| SQL | ❌ |

---

## 🔹 Структуры данных

| Тип | Операции | Use case |
|-----|----------|----------|
| **String** | GET, SET, INCR | Cache, counters, flags |
| **Hash** | HGET, HSET | Объект (user profile) |
| **List** | LPUSH, RPOP | Очередь FIFO |
| **Set** | SADD, SMEMBERS | Уникальные теги |
| **Sorted Set (ZSet)** | ZADD, ZRANGE | Leaderboard, ranking |
| **Stream** | XADD, XREAD | Event log (как Kafka-lite) |
| **Bitmap / HyperLogLog** | SETBIT, PFADD | Analytics, unique visitors |

```redis
SET user:42:name "Alice"
EXPIRE user:42:name 3600

HSET user:42 email "a@b.com" tier "premium"
HGETALL user:42

ZADD leaderboard 1500 "player1" 2300 "player2"
ZREVRANGE leaderboard 0 9 WITHSCORES
```

---

## 🔹 TTL и eviction

```redis
SET session:abc123 "data" EX 1800        # TTL 30 min
SET cache:product:1 "json..." PX 60000 # TTL в ms
```

**Политики eviction** (при нехватке RAM):

| Policy | Поведение |
|--------|-----------|
| `noeviction` | Ошибка при записи |
| `allkeys-lru` | Удалить любой least recently used |
| `volatile-lru` | LRU только среди ключей с TTL |

> [!warning] Cache без TTL
> Ключи без `EX`/`PX` живут вечно → memory leak.

---

## 🔹 Persistence — RDB vs AOF

| | RDB (snapshot) | AOF (append-only file) |
|---|----------------|------------------------|
| Как | Периодический dump | Лог каждой write-команды |
| Recovery | Быстрее | Медленнее |
| Потеря данных | До интервала snapshot | Минимальная (`everysec`) |
| Размер файла | Компактный | Больше |

```conf
# redis.conf
save 900 1          # RDB: если 1 key changed за 900s
appendonly yes      # AOF
appendfsync everysec
```

> [!tip] Production
> Часто **оба**: RDB для быстрого cold start + AOF для durability.

---

## 🔹 Spring Data Redis

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 2000ms
```

```java
@Repository
@RequiredArgsConstructor
public class ProductCache {
    private final StringRedisTemplate redis;

    public Optional<Product> get(long id) {
        String json = redis.opsForValue().get("product:" + id);
        return json != null ? Optional.of(parse(json)) : Optional.empty();
    }

    public void put(Product p, Duration ttl) {
        redis.opsForValue().set("product:" + p.getId(), toJson(p), ttl);
    }
}
```

---

## 🔹 @Cacheable (Spring Cache)

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))
            .build();
    }
}

@Service
public class UserService {
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#user.id")
    public User update(User user) { ... }
}
```

---

## 🔹 Pub/Sub

```redis
SUBSCRIBE notifications
PUBLISH notifications "order-123-paid"
```

> [!warning] Pub/Sub не durable
> Подписчик offline — сообщения **теряются**. Для durable — **Redis Streams** или Kafka.

```java
// Spring: RedisMessageListenerContainer
@Component
public class NotificationListener implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] pattern) {
        log.info("Received: {}", new String(message.getBody()));
    }
}
```

---

## 🔹 Use cases

| Задача | Решение Redis |
|--------|---------------|
| Cache DB queries | String + TTL |
| Session store | Hash + TTL |
| Rate limiting | INCR + EXPIRE |
| Leaderboard | Sorted Set |
| Distributed lock | SET NX EX (или Redisson) |
| Real-time counter | INCR |
| Pub/Sub notifications | PUBLISH/SUBSCRIBE |

### ❌ Redis как primary database
Нет сложных запросов, ограниченная durability без AOF, данные в RAM.

### ✅ Redis как cache / fast layer
Поверх PostgreSQL; cache-aside pattern.

---

## 🔹 Итог

1. Redis — in-memory structures, не замена SQL.
2. TTL обязателен для cache-ключей.
3. RDB + AOF — persistence в prod.
4. Spring: `StringRedisTemplate`, `@Cacheable`, Redis CacheManager.
5. Pub/Sub — fire-and-forget; durable events — Streams/Kafka.

```
Шпаргалка Redis:
─────────────────────────────────────────
SET key val EX 3600                → TTL cache
HSET / ZADD / LPUSH                → structures
@Cacheable / @CacheEvict           → Spring Cache
StringRedisTemplate                → programmatic
RDB snapshot + AOF everysec        → persistence
❌ primary DB  ✅ cache/session/rate limit
```
