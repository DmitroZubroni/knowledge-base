> **Теги:** #nosql #redis #cache #конспект
> [!abstract] Связи
> [[main]] | [[main SQL DB]] | [[NoSQL_DB]]

# Redis

---

## 🔹 Что такое Redis и зачем он нужен

Redis (Remote Dictionary Server) — in-memory хранилище данных. Всё хранится в оперативной памяти → операции выполняются за микросекунды.

```
Без Redis:                      С Redis:
Запрос → БД (~5-50ms)          Запрос → Redis (~0.1ms) → ответ
                                Промах → БД (~50ms) → Redis → ответ
                                Следующий запрос → Redis (0.1ms)
```

**Основные применения:**
- **Кэш** — снизить нагрузку на БД, ускорить ответы
- **Session store** — хранить сессии пользователей
- **Rate limiter** — ограничить частоту запросов
- **Distributed lock** — блокировки в распределённых системах
- **Pub/Sub / Streams** — обмен сообщениями между сервисами
- **Leaderboards** — рейтинги в реальном времени
- **Counter** — счётчики просмотров, лайков

---

## 🔹 Структуры данных — подробно

### String — универсальный тип

Самый простой тип. Значение: строка, число, бинарные данные (до 512MB).

```redis
SET user:42:name "Alice" EX 3600    # установить с TTL 1 час
GET user:42:name                     # → "Alice"
MSET k1 v1 k2 v2 k3 v3             # установить несколько
MGET k1 k2 k3                       # получить несколько

# Атомарные счётчики
SET counter:visits:page1 0
INCR counter:visits:page1            # → 1 (атомарно!)
INCRBY counter:visits:page1 10       # → 11
DECR counter:visits:page1            # → 10

# Условная установка — основа distributed lock
SET lock:resource1 "owner-id" NX EX 30  # только если не существует
# NX = Not eXists, EX = EXpire (секунды)
```

**Применение String:**
- Кэш HTML/JSON объектов
- Счётчики просмотров, лайков
- Флаги (`feature:dark-mode:enabled = "true"`)
- Распределённые блокировки через `SET NX EX`

### Hash — объект с полями

Хранит набор пар field-value. Идеален для представления объектов.

```redis
HSET user:42 name "Alice" email "alice@example.com" age "28" tier "premium"
HGET user:42 name             # → "Alice"
HMGET user:42 name email      # → ["Alice", "alice@example.com"]
HGETALL user:42               # → все поля и значения
HDEL user:42 tier             # удалить поле
HEXISTS user:42 email         # → 1 (существует)
HKEYS user:42                 # → [name, email, age]
HLEN user:42                  # → 3

# Атомарный инкремент поля
HINCRBY user:42 age 1         # → 29
```

**Преимущество перед JSON в String:** можно обновить одно поле без десериализации всего объекта.

```
String подход:            Hash подход:
GET user:42              HGET user:42 age
deserialize JSON         → сразу получаем значение
change age
serialize JSON
SET user:42 newJson
```

### List — двусвязный список

```redis
RPUSH queue:emails "email1" "email2"  # добавить в хвост
LPUSH queue:emails "urgent"            # добавить в голову
LRANGE queue:emails 0 -1              # все элементы
RPOP queue:emails                      # извлечь из хвоста (FIFO)
LPOP queue:emails                      # извлечь из головы (LIFO)
LLEN queue:emails                      # длина
LINDEX queue:emails 0                  # элемент по индексу

# Блокирующее извлечение — ждать элемент до N секунд
BRPOP queue:emails 30   # блокирует до 30 сек, пока не появится элемент
BLPOP queue:emails 0    # блокирует бесконечно
```

**Применение:**
- Очередь задач (producer → RPUSH, consumer → BLPOP)
- Лента активности (последние N действий пользователя через LPUSH + LTRIM)
- История просмотров

```redis
# Лента последних 100 действий
LPUSH activity:user:42 "viewed:product:1"
LTRIM activity:user:42 0 99   # оставить только первые 100
```

### Set — множество уникальных элементов

```redis
SADD tags:article:1 "java" "spring" "redis"
SMEMBERS tags:article:1     # → {java, spring, redis}
SISMEMBER tags:article:1 "java"  # → 1 (есть)
SCARD tags:article:1         # → 3 (размер)
SREM tags:article:1 "redis"  # удалить

# Операции над множествами
SADD tags:article:2 "java" "kafka"
SUNION tags:article:1 tags:article:2      # объединение
SINTER tags:article:1 tags:article:2      # пересечение → {java}
SDIFF tags:article:1 tags:article:2       # разность → {spring}

SUNIONSTORE result tags:article:1 tags:article:2  # сохранить результат
```

**Применение:**
- Уникальные посетители страницы
- Теги статей
- Друзья/подписчики (пересечение = общие друзья)
- Проверка "видел ли пользователь уведомление"

### Sorted Set (ZSet) — отсортированное множество

Как Set, но у каждого элемента есть **score** (число с плавающей точкой). Автоматически сортируется по score.

```redis
ZADD leaderboard 1500.5 "player1" 2300 "player2" 980 "player3"
ZRANK leaderboard "player1"          # → 1 (позиция, 0-based, по возрастанию)
ZREVRANK leaderboard "player1"       # → 1 (позиция по убыванию)
ZSCORE leaderboard "player1"         # → 1500.5
ZINCRBY leaderboard 100 "player1"    # увеличить score

# Диапазоны
ZRANGE leaderboard 0 -1 WITHSCORES          # все, по возрастанию
ZREVRANGE leaderboard 0 9 WITHSCORES        # топ-10, по убыванию
ZRANGEBYSCORE leaderboard 1000 2000         # score в диапазоне [1000, 2000]
ZRANGEBYSCORE leaderboard -inf +inf         # все

ZCARD leaderboard       # количество элементов
ZCOUNT leaderboard 1000 2000  # количество в диапазоне score
```

**Применение:**
- Рейтинги и лидерборды
- Очередь с приоритетами (score = timestamp → работает как delay queue)
- Автодополнение поиска
- Rate limiting по временным окнам

### Stream — журнал событий

Похож на Kafka топик внутри Redis. Каждая запись получает уникальный ID `timestamp-sequence`.

```redis
XADD orders * userId 42 productId 100 amount 99.99
# * → автогенерация ID вида "1699000000000-0"

XLEN orders           # количество записей
XRANGE orders - +     # все записи
XRANGE orders 1699000000000-0 +  # с определённого ID

# Consumer group — как Kafka consumer group
XGROUP CREATE orders billing $ MKSTREAM  # создать группу от текущего момента
XREADGROUP GROUP billing worker1 COUNT 10 STREAMS orders >  # прочитать новые
XACK orders billing 1699000000000-0      # подтвердить обработку
```

### HyperLogLog — приближённый подсчёт уникальных

```redis
PFADD visitors:today "user1" "user2" "user3" "user1"  # дубликат игнорируется
PFCOUNT visitors:today   # → ~3 (погрешность ~0.81%)
PFMERGE visitors:week visitors:mon visitors:tue visitors:wed
```

Хранит ~12KB вместо хранения всех ID. Погрешность 0.81%. Идеален для "сколько уникальных пользователей сегодня".

### Bitmap — битовые операции

```redis
SETBIT user:42:activity:2024-01 0 1   # был активен 1-го января
SETBIT user:42:activity:2024-01 14 1  # был активен 15-го января
GETBIT user:42:activity:2024-01 14    # → 1
BITCOUNT user:42:activity:2024-01     # → 2 (активных дней)

# Пересечение: кто был активен ВСЕ три дня
BITOP AND result day1 day2 day3
BITCOUNT result   # количество таких пользователей
```

---

## 🔹 TTL и Eviction Policy

### TTL стратегии

```redis
SET key value EX 3600        # истекает через 3600 секунд
SET key value PX 60000       # истекает через 60000 миллисекунд
SET key value EXAT 1893456000 # истекает в unix timestamp
SET key value KEEPTTL        # сохранить текущий TTL при обновлении значения

EXPIRE key 3600              # установить TTL на уже существующий ключ
PEXPIRE key 60000            # в миллисекундах
EXPIREAT key 1893456000      # в unix timestamp
PERSIST key                  # убрать TTL (ключ становится вечным)

TTL key                      # оставшееся время в секундах (-1 = нет TTL, -2 = не существует)
PTTL key                     # в миллисекундах
```

> [!warning] Ключи без TTL = memory leak
> Если кэш-ключи не имеют TTL — они живут вечно и постепенно заполняют память. Всегда устанавливай TTL для кэш-ключей.

### Eviction Policy — что делать когда память заполнена

```conf
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

| Policy | Поведение | Когда использовать |
|--------|-----------|-------------------|
| `noeviction` | Ошибка при записи (OOM error) | Нельзя терять данные |
| `allkeys-lru` | Удалить любой LRU ключ | Чистый кэш — удалять всё старое |
| `allkeys-lfu` | Удалить наименее используемый | Часть данных горячее других |
| `allkeys-random` | Удалить случайный | Редко |
| `volatile-lru` | LRU только среди ключей с TTL | Часть данных нельзя удалять |
| `volatile-lfu` | LFU только среди ключей с TTL | То же, с учётом частоты |
| `volatile-ttl` | Удалить с наименьшим TTL | Предпочесть удалить скоро-истекающие |
| `volatile-random` | Случайный среди ключей с TTL | Редко |

> [!tip] Рекомендации
> - Чистый кэш → `allkeys-lru` или `allkeys-lfu`
> - Смешанное хранение (часть данных нельзя терять) → `volatile-lru` + все важные ключи без TTL
> - `noeviction` + мониторинг памяти → для session store и очередей

---

## 🔹 Паттерны кэширования

### Cache-Aside (Lazy Loading) — самый распространённый

```
Чтение:
  1. Проверить Redis
  2. HIT → вернуть из Redis
  3. MISS → запросить БД → записать в Redis → вернуть

Запись:
  1. Обновить БД
  2. Инвалидировать кэш (delete ключ)
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository repository;
    private final StringRedisTemplate redis;
    private final ObjectMapper mapper;

    private static final Duration TTL = Duration.ofMinutes(30);

    public Product findById(Long id) {
        String key = "product:" + id;
        String cached = redis.opsForValue().get(key);

        if (cached != null) {
            return deserialize(cached); // HIT
        }

        // MISS
        Product product = repository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        redis.opsForValue().set(key, serialize(product), TTL);
        return product;
    }

    public Product update(Product product) {
        Product saved = repository.save(product);
        redis.delete("product:" + product.getId()); // инвалидация
        return saved;
    }
}
```

**Плюсы:** простота, кэшируется только то, что реально запрашивается.
**Минусы:** первый запрос всегда медленный (MISS), возможна краткая рассинхронизация при обновлении.

### Write-Through — запись одновременно в кэш и БД

```
Запись:
  1. Записать в Redis
  2. Записать в БД
  (атомарно или с транзакцией)

Чтение:
  1. Проверить Redis
  2. HIT → вернуть (MISS крайне редки)
```

```java
public Product update(Product product) {
    Product saved = repository.save(product);
    redis.opsForValue().set("product:" + saved.getId(),
        serialize(saved), TTL); // обновляем кэш сразу
    return saved;
}
```

**Плюсы:** кэш всегда актуален, MISS редки.
**Минусы:** записи медленнее (два хранилища), кэшируется всё включая то, что никогда не читают.

### Write-Behind (Write-Back) — сначала кэш, потом БД асинхронно

```
Запись:
  1. Записать в Redis немедленно
  2. Асинхронно (через очередь) → записать в БД

Чтение: всегда из Redis (свежие данные)
```

**Плюсы:** очень быстрые записи.
**Минусы:** риск потери данных если Redis упадёт до записи в БД. Сложнее в реализации.

### Read-Through — Redis сам загружает из БД

Redis (с плагином или Redisson) сам обращается к БД при промахе. В Java — через Redisson `RMapCache` с `MapLoader`.

```java
MapLoader<String, Product> loader = new MapLoader<>() {
    @Override
    public Product load(String key) {
        return repository.findById(Long.parseLong(key)).orElse(null);
    }
};
RMapCache<String, Product> cache = redisson.getMapCache("products");
```

---

## 🔹 Cache Stampede — проблема и решения

**Проблема:** популярный кэш-ключ истёк. Сотни запросов одновременно обнаруживают MISS и все идут в БД — "стадо" запросов убивает базу.

```
t=0: key expired
t=1: 500 запросов → все видят MISS → все идут в БД
t=2: БД получает 500 одинаковых запросов → перегрузка / timeout
```

### Решение 1: Mutex Lock (распределённая блокировка)

```java
public Product findById(Long id) {
    String key = "product:" + id;
    String lockKey = "lock:product:" + id;
    String cached = redis.opsForValue().get(key);

    if (cached != null) return deserialize(cached);

    // Пытаемся получить блокировку — только один поток пойдёт в БД
    Boolean locked = redis.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(5)); // NX EX 5

    if (Boolean.TRUE.equals(locked)) {
        try {
            Product product = repository.findById(id).orElseThrow();
            redis.opsForValue().set(key, serialize(product), TTL);
            return product;
        } finally {
            redis.delete(lockKey); // снять блокировку
        }
    } else {
        // Другой поток держит блокировку — немного ждём и повторяем
        Thread.sleep(50);
        return findById(id); // рекурсивный повтор
    }
}
```

### Решение 2: Probabilistic Early Expiration

Случайно продлевать кэш чуть раньше реального истечения — один запрос обновляет заблаговременно:

```java
public Product findById(Long id) {
    String key = "product:" + id;
    String ttlKey = "product:ttl:" + id;

    String cached = redis.opsForValue().get(key);
    if (cached != null) {
        // Если TTL < 20% от исходного — с вероятностью 10% обновить заранее
        Long ttl = redis.getExpire(key, TimeUnit.SECONDS);
        if (ttl != null && ttl < TTL.getSeconds() * 0.2 && Math.random() < 0.1) {
            refreshAsync(id); // асинхронное обновление
        }
        return deserialize(cached);
    }
    return loadAndCache(id);
}
```

### Решение 3: Stale-While-Revalidate

Возвращать устаревшие данные пока в фоне идёт обновление:

```java
public Product findById(Long id) {
    String key = "product:" + id;
    String staleKey = "product:stale:" + id; // с большим TTL

    String cached = redis.opsForValue().get(key);
    if (cached != null) return deserialize(cached);

    // Основной кэш истёк — вернуть stale данные пока обновляем
    String stale = redis.opsForValue().get(staleKey);
    if (stale != null) {
        CompletableFuture.runAsync(() -> loadAndCache(id)); // обновить асинхронно
        return deserialize(stale); // вернуть старое
    }

    return loadAndCache(id); // холодный старт
}
```

---

## 🔹 Distributed Lock — правильная реализация

### Простая блокировка через SET NX EX

```java
@Service
@RequiredArgsConstructor
public class RedisLock {
    private final StringRedisTemplate redis;

    public boolean acquire(String resource, String ownerId, Duration ttl) {
        // SET lock:resource ownerId NX EX ttl
        return Boolean.TRUE.equals(
            redis.opsForValue().setIfAbsent("lock:" + resource, ownerId, ttl)
        );
    }

    public boolean release(String resource, String ownerId) {
        // Lua скрипт: проверить владельца и удалить атомарно
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        Long result = redis.execute(
            new DefaultRedisScript<>(script, Long.class),
            List.of("lock:" + resource),
            ownerId
        );
        return Long.valueOf(1L).equals(result);
    }
}

// Использование
String ownerId = UUID.randomUUID().toString();
if (lock.acquire("order:123", ownerId, Duration.ofSeconds(30))) {
    try {
        processOrder(123L);
    } finally {
        lock.release("order:123", ownerId);
    }
}
```

> [!warning] Проверь владельца перед release!
> Без Lua скрипта возможна race condition: ты проверяешь что ключ твой, потом TTL истекает, другой поток захватывает лок, ты удаляешь чужой лок. Lua скрипт выполняется атомарно — этот сценарий исключён.

### Redlock — для высокой надёжности

Для критичных сценариев (финансовые операции) используй Redlock — алгоритм блокировки на N независимых Redis нодах:

```java
// Через библиотеку Redisson
Config config = new Config();
config.useClusterServers().addNodeAddress("redis://127.0.0.1:7000", "redis://127.0.0.1:7001", "redis://127.0.0.1:7002");
RedissonClient redisson = Redisson.create(config);

RLock lock = redisson.getLock("order:123");
try {
    // Захватить лок, подождать 10сек, автоосвобождение через 30сек
    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
        try {
            processOrder(123L);
        } finally {
            lock.unlock();
        }
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

## 🔹 Rate Limiting

### Фиксированное окно (Fixed Window)

```java
public boolean isAllowed(String userId) {
    String key = "rate:" + userId + ":" + currentMinute();
    Long count = redis.opsForValue().increment(key);
    if (count == 1) redis.expire(key, Duration.ofMinutes(1));
    return count <= 100; // 100 запросов в минуту
}
// Проблема: пользователь может сделать 100 запросов в 00:59 и 100 в 01:00
// = 200 запросов за 2 секунды!
```

### Скользящее окно через Sorted Set (Sliding Window)

```java
public boolean isAllowed(String userId, int limitPerMinute) {
    String key = "rate:sliding:" + userId;
    long now = System.currentTimeMillis();
    long windowStart = now - 60_000; // 1 минута назад

    // Lua скрипт для атомарности
    String script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])
        
        redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window) -- удалить старые
        local count = redis.call('ZCARD', key)                     -- текущее количество
        
        if count < limit then
            redis.call('ZADD', key, now, now)                      -- добавить запрос
            redis.call('EXPIRE', key, window / 1000 + 1)           -- TTL на ключ
            return 1  -- разрешить
        else
            return 0  -- отказать
        end
        """;

    Long result = redis.execute(
        new DefaultRedisScript<>(script, Long.class),
        List.of(key),
        String.valueOf(now),
        String.valueOf(60_000),
        String.valueOf(limitPerMinute)
    );
    return Long.valueOf(1L).equals(result);
}
```

---

## 🔹 Pub/Sub

```redis
# Подписка (блокирует соединение)
SUBSCRIBE channel:notifications
PSUBSCRIBE order:*    # pattern — подписка на все каналы по шаблону

# Публикация
PUBLISH channel:notifications "order-123-paid"
```

```java
// Spring Redis Pub/Sub
@Configuration
public class RedisPubSubConfig {
    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory factory,
            MessageListenerAdapter listenerAdapter) {
        var container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(listenerAdapter,
            new PatternTopic("order:*")); // подписка по шаблону
        return container;
    }

    @Bean
    public MessageListenerAdapter listenerAdapter(OrderEventHandler handler) {
        return new MessageListenerAdapter(handler, "handleMessage");
    }
}

@Component
public class OrderEventHandler {
    public void handleMessage(String message, String channel) {
        log.info("Received from {}: {}", channel, message);
    }
}

// Публикация
@Service
@RequiredArgsConstructor
public class OrderPublisher {
    private final RedisTemplate<String, String> redis;

    public void publish(String orderId, String status) {
        redis.convertAndSend("order:" + orderId, status);
    }
}
```

> [!warning] Pub/Sub не durable
> Если subscriber offline — сообщения теряются навсегда. Для надёжной доставки используй **Redis Streams** или Kafka.

---

## 🔹 Pipeline и транзакции

### Pipeline — пакетная отправка команд

По умолчанию каждая команда — отдельный round-trip (запрос-ответ). Pipeline отправляет много команд за один round-trip.

```java
// Без pipeline: N команд = N round-trips
for (int i = 0; i < 1000; i++) {
    redis.opsForValue().set("key:" + i, "value:" + i);
}

// С pipeline: 1000 команд = 1 round-trip
List<Object> results = redis.executePipelined(connection -> {
    for (int i = 0; i < 1000; i++) {
        connection.set(("key:" + i).getBytes(), ("value:" + i).getBytes());
    }
    return null;
});
// Ускорение: 10-100x при большом количестве команд
```

### MULTI/EXEC — транзакции

```redis
MULTI              # начать транзакцию
SET account:1 900  # команды ставятся в очередь
SET account:2 1100
EXEC               # выполнить все атомарно
# или
DISCARD            # отменить
```

```java
// Spring Redis транзакция
redis.execute(connection -> {
    connection.multi();
    connection.set(key1, val1);
    connection.set(key2, val2);
    connection.exec();
    return null;
});
```

> [!note] Redis транзакции vs SQL транзакции
> Redis `MULTI/EXEC` — **не** как SQL транзакция. Нет rollback при ошибке в команде (другие команды выполнятся). MULTI/EXEC гарантирует только атомарность выполнения (никто не вклинится между командами).

### Lua скрипты — настоящая атомарность

```java
// Lua скрипт выполняется атомарно — как одна операция Redis
String script = """
    local current = redis.call('GET', KEYS[1])
    if current and tonumber(current) >= tonumber(ARGV[1]) then
        redis.call('DECRBY', KEYS[1], ARGV[1])
        return 1  -- успешно списано
    else
        return 0  -- недостаточно средств
    end
    """;

Long result = redis.execute(
    new DefaultRedisScript<>(script, Long.class),
    List.of("balance:" + userId),
    amount.toString()
);
```

---

## 🔹 Persistence — RDB vs AOF

| | RDB (snapshot) | AOF (append-only file) |
|---|----------------|------------------------|
| Как работает | Периодический дамп всей БД в файл | Записывает каждую write-команду в лог |
| Recovery speed | **Быстрый** (загрузить дамп) | Медленнее (воспроизвести все команды) |
| Потеря данных | До последнего snapshot (минуты) | Минимальная (до 1 сек с `everysec`) |
| Размер файла | Компактный | Растёт, нужна компактизация |
| Нагрузка | fork() — кратковременный spike памяти | Постоянная запись на диск |
| Когда | Допустима небольшая потеря данных | Нужна минимальная потеря |

```conf
# redis.conf

# RDB: сохранять если за 900 секунд изменился хотя бы 1 ключ
save 900 1
save 300 10
save 60 10000

# AOF
appendonly yes
appendfsync everysec  # fsync каждую секунду — баланс скорость/надёжность
# appendfsync always   # fsync на каждую команду — медленно но надёжно
# appendfsync no       # fsync по усмотрению OS — быстро но риск потери
```

> [!tip] Рекомендация для продакшена
> Использовать **оба**: RDB для быстрого cold start после перезапуска + AOF для минимальной потери данных.

---

## 🔹 Replication и кластер

### Master-Replica Replication

```
Master: принимает записи → асинхронно реплицирует →
  Replica 1: только чтение
  Replica 2: только чтение
  Replica 3: только чтение
```

```conf
# На реплике (redis.conf):
replicaof 192.168.1.1 6379
replica-read-only yes
```

**Применение:** горизонтальное масштабирование чтения. Если 90% операций — чтение, добавление реплик снижает нагрузку на master.

### Sentinel — автоматический failover

```
Sentinel 1 ─┐
Sentinel 2 ─┼── мониторят Master и Replicas
Sentinel 3 ─┘

Master упал → Sentinels договариваются → выбирают новый Master из реплик
                                          автоматически, без ручного вмешательства
```

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes: "sentinel1:26379,sentinel2:26379,sentinel3:26379"
```

### Redis Cluster — шардирование

```
16384 hash slots разделены между нодами:
  Node 1: slots 0-5460
  Node 2: slots 5461-10922
  Node 3: slots 10923-16383

ключ → CRC16(key) % 16384 → определяет ноду

Каждая нода имеет свои реплики → отказоустойчивость + масштабирование записи
```

```yaml
spring:
  data:
    redis:
      cluster:
        nodes: "node1:7000,node2:7001,node3:7002,node4:7003,node5:7004,node6:7005"
```

> [!warning] Ограничение кластера
> Операции над несколькими ключами (`MGET`, `MSET`, транзакции) работают только если все ключи на одной ноде. Используй hash tags для группировки: `{user:42}:name` и `{user:42}:email` попадут в одну ноду.

---

## 🔹 Spring Data Redis — полный пример

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users",
                config.entryTtl(Duration.ofHours(1)))     // users кэш — 1 час
            .withCacheConfiguration("products",
                config.entryTtl(Duration.ofMinutes(30)))  // products — 30 минут
            .build();
    }
}

@Service
@RequiredArgsConstructor
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "users", key = "#user.id")  // обновить кэш после сохранения
    public User update(User user) {
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#id")     // удалить из кэша
    public void delete(Long id) {
        userRepository.deleteById(id);
    }

    @Caching(evict = {                            // несколько операций с кэшем
        @CacheEvict(value = "users", key = "#id"),
        @CacheEvict(value = "user-lists", allEntries = true)
    })
    public void deleteWithLists(Long id) {
        userRepository.deleteById(id);
    }
}
```

---

## 🔹 Итог

```
Redis = in-memory key-value хранилище, операции за ~0.1ms

Структуры данных:
  String     → кэш, счётчики (INCR), флаги, distributed lock (SET NX EX)
  Hash       → объекты (обновление поля без десериализации)
  List       → очереди (RPUSH/BLPOP), лента активности
  Set        → уникальные элементы, операции над множествами
  Sorted Set → лидерборды, delay queue, rate limiting
  Stream     → надёжный журнал событий (как Kafka-lite)
  HyperLogLog → приближённый подсчёт уникальных (0.81% погрешность, 12KB)
  Bitmap     → активность по дням, bulk аналитика

TTL: всегда ставь на кэш-ключи. Без TTL = memory leak.
Eviction: allkeys-lru для чистого кэша, volatile-lru для смешанного.

Паттерны кэширования:
  Cache-Aside     — читать из Redis, при MISS → БД → Redis (самый частый)
  Write-Through   — запись сразу в Redis и БД
  Write-Behind    — сначала Redis, потом асинхронно БД (риск потери)

Cache Stampede — 100 запросов на истёкший ключ → убивает БД
  Решение: Mutex lock (SET NX) / Probabilistic early expiration / Stale-While-Revalidate

Distributed Lock: SET NX EX + Lua для атомарного release
  Prodation: Redisson Redlock для нескольких нод

Rate Limiting:
  Fixed window  — INCR + EXPIRE (просто, но edge case на границе окна)
  Sliding window — ZSET + ZREMRANGEBYSCORE (точнее)

Pipeline: 1000 команд за 1 round-trip (10-100x быстрее)
Lua скрипты: настоящая атомарность для сложных операций
MULTI/EXEC: нет rollback при ошибке команды (не как SQL)

Persistence: RDB (быстрый restart) + AOF everysec (минимальная потеря)
HA: Sentinel (автофailover) → Cluster (шардирование + HA)

Pub/Sub: не durable (сообщения теряются если subscriber offline)
         → для надёжной доставки: Redis Streams или Kafka
```
