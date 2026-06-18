> **Теги:** #tools #kafka #messaging #конспект
> [!abstract] Связи
> [[main]] | [[main Tools]]

# Apache Kafka

---

## 🔹 Что такое Kafka и зачем она нужна

Представь интернет-магазин: пользователь оформил заказ — нужно уведомить склад, списать деньги, отправить email, обновить аналитику. Если делать всё синхронно — один упавший сервис блокирует всё. Если вызывать каждый сервис напрямую — жёсткая связанность.

Kafka решает это: сервис-отправитель публикует событие в Kafka и забывает о нём. Каждый заинтересованный сервис читает независимо, в своём темпе.

```
Без Kafka:                        С Kafka:
OrderService ──► WarehouseService  OrderService ──► Topic "orders"
             ──► PaymentService                          │
             ──► EmailService       WarehouseService ◄──┤
             ──► AnalyticsService   PaymentService   ◄──┤
    ↑ жёсткая связанность          EmailService     ◄──┤
    ↑ падение одного = проблема     Analytics        ◄──┘
                                    ↑ слабая связанность, независимость
```

**Kafka vs RabbitMQ:**

| | Kafka | RabbitMQ |
|-|-------|----------|
| Модель | Append-only log | Очередь сообщений |
| Хранение | По времени/размеру (дни/недели) | Удаляет после доставки |
| Повторное чтение | ✅ Перемотать offset | ❌ Сложно |
| Пропускная способность | Очень высокая (млн msg/сек) | Средняя |
| Порядок | Гарантирован внутри partition | Ограничен |
| Routing | Простой (topic/partition) | Гибкий (exchanges, bindings) |
| Когда | Event sourcing, CDC, аналитика, большие потоки | Task queues, RPC-style, сложный routing |

---

## 🔹 Архитектура: основные компоненты

```
Producers                Kafka Cluster               Consumers
─────────────────────────────────────────────────────────────────
OrderService ──►        Topic: "orders"          ──► BillingService
                       ┌────────────────────┐         (group: billing)
PaymentService ──►     │ Partition 0: [0..N] │    ──► WarehouseService
                       │ Partition 1: [0..N] │         (group: warehouse)
UserService ──►        │ Partition 2: [0..N] │    ──► AnalyticsService
                       └────────────────────┘         (group: analytics)
                             │
                       Broker Cluster
                       (Leader + Followers)
```

**Topic** — именованная категория событий (`orders`, `payments`, `user-events`). Аналог таблицы в БД, но для потоков.

**Partition** — упорядоченный, неизменяемый лог внутри topic. Сообщения только добавляются в конец. Каждое сообщение получает монотонно растущий **offset**.

**Broker** — сервер Kafka. Хранит партиции. В кластере обычно 3+ брокеров.

**ZooKeeper / KRaft** — хранит метаданные кластера (какой брокер — лидер для партиции). KRaft (Kafka Raft) — встроенная замена ZooKeeper с Kafka 3.x.

---

## 🔹 Партиции и параллелизм — как Kafka масштабируется

Партиции — ключевой механизм масштабирования Kafka. Понять их важно.

### Как сообщение попадает в партицию

```
Producer.send(topic, key, value)
    │
    ├── key == null → round-robin по партициям
    └── key != null → hash(key) % numPartitions → всегда одна и та же партиция
```

**Почему ключ важен:**
```
Без ключа:                          С ключом (orderId):
msg1 → partition 0                  order-123 → всегда partition 1
msg2 → partition 1                  order-124 → всегда partition 0
msg3 → partition 2                  order-123 → всегда partition 1
// порядок для одного заказа не гарантирован    // ВСЕ события заказа 123 в одной партиции
                                                // → порядок гарантирован
```

### Consumer Group и параллелизм

Один consumer group делит партиции между своими members:

```
Topic "orders": 4 партиции

Group "billing" с 2 consumers:
  Consumer-A → partition 0, partition 1  (по 2 партиции на каждого)
  Consumer-B → partition 2, partition 3

Group "billing" с 4 consumers:
  Consumer-A → partition 0
  Consumer-B → partition 1
  Consumer-C → partition 2
  Consumer-D → partition 3  ← оптимально: 1 consumer на партицию

Group "billing" с 5 consumers:
  Consumer-A → partition 0
  Consumer-B → partition 1
  Consumer-C → partition 2
  Consumer-D → partition 3
  Consumer-E → (idle, без работы!)  ← лишний consumer бесполезен
```

> [!note] Правило масштабирования
> Максимальный параллелизм = количество партиций. Consumers > partitions — лишние простаивают. Хочешь больше параллелизма — увеличивай количество партиций (но это необратимо!).

### Rebalancing — когда перераспределяются партиции

Rebalancing происходит при:
- Consumer вступает в группу
- Consumer покидает группу (или упал)
- Partition'ов стало больше

**Проблема rebalancing:** во время него **все consumers в группе приостанавливают** чтение. Это Stop-the-World для группы.

**Стратегии назначения партиций:**
```
RangeAssignor     — партиции разбиваются диапазонами (0-3 → consumer A, 4-7 → consumer B)
RoundRobinAssignor — по одной по кругу (более равномерно)
StickyAssignor    — минимизирует перемещения при rebalance (consumer сохраняет свои партиции)
CooperativeStickyAssignor — incremental rebalancing, только затронутые партиции перемещаются
                             (рекомендуется — нет полной остановки)
```

```yaml
spring.kafka.consumer.partition-assignment-strategy:
  org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

---

## 🔹 Offset Management — где мы остановились

Offset — номер позиции сообщения в партиции. Consumer сам отвечает за хранение своего offset.

```
Partition 0:  [0] [1] [2] [3] [4] [5] [6] [7] ...
                              ↑
                         committed offset = 4
                         (сообщения 0-3 обработаны)
                         (следующее к чтению: 4)
```

Offsets хранятся в специальном топике `__consumer_offsets` внутри Kafka (раньше — в ZooKeeper).

### auto-commit vs manual commit

```java
// AUTO COMMIT (enable.auto.commit=true, interval=5000ms)
// Kafka автоматически коммитит каждые N мс
// ПРОБЛЕМА: commit может произойти ДО обработки → at-most-once
// Сообщение прочитано, через 3сек commit, сервис упал до обработки → сообщение потеряно

// MANUAL COMMIT — рекомендуется в продакшене
@KafkaListener(topics = "orders")
public void handle(OrderEvent event, Acknowledgment ack) {
    try {
        processOrder(event);  // сначала обрабатываем
        ack.acknowledge();    // потом коммитим offset → at-least-once
    } catch (Exception e) {
        // не коммитим → сообщение будет перечитано
        log.error("Failed to process order {}", event.getOrderId(), e);
    }
}
```

### auto.offset.reset — что делать при отсутствии committed offset

```yaml
auto-offset-reset: earliest  # читать с самого начала (все исторические сообщения)
auto-offset-reset: latest    # читать только новые (с момента подключения)
auto-offset-reset: none      # бросить исключение если offset не найден
```

`earliest` — при первом запуске нового consumer group берёт все сообщения с начала топика. Осторожно на топиках с большой историей!

---

## 🔹 Гарантии доставки — at-most-once, at-least-once, exactly-once

### At-most-once (максимум один раз)

Сообщение доставляется **не более одного раза** — потеря возможна, дубликаты исключены.

```
Producer → Kafka → Consumer
    ↑ commit offset ДО обработки
    Сервис упал после commit, но до обработки → сообщение потеряно навсегда
```

Когда допустимо: метрики, логи — потеря нескольких записей некритична.

### At-least-once (минимум один раз)

Сообщение доставляется **минимум один раз** — дубликаты возможны, потеря исключена.

```
Producer → Kafka → Consumer
    ↑ commit offset ПОСЛЕ обработки
    Сервис упал ПОСЛЕ обработки, но ДО commit → сообщение перечитается → дубликат
```

Это **стандарт в большинстве систем**. Дубликаты обрабатываются на стороне consumer (идемпотентность).

```java
// Идемпотентный consumer — безопасно обрабатывает дубликаты
@KafkaListener(topics = "orders")
public void handle(OrderEvent event, Acknowledgment ack) {
    // Проверяем: не обрабатывали ли уже это событие?
    if (processedEventRepository.existsById(event.getEventId())) {
        ack.acknowledge(); // уже обработано — пропускаем
        return;
    }

    orderService.process(event);
    processedEventRepository.save(event.getEventId()); // запоминаем
    ack.acknowledge();
}
```

### Exactly-once (ровно один раз)

Сообщение доставляется **ровно один раз** — ни потерь, ни дубликатов.

Требует трёх компонентов:
1. **Idempotent producer** — producer не создаёт дубликаты при retry
2. **Transactions** — атомарная запись в несколько топиков
3. **Isolation level: read_committed** — consumer читает только завершённые транзакции

```java
// Producer side
@Bean
public ProducerFactory<String, Object> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // idempotent
    config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-service-tx-1"); // транзакции
    config.put(ProducerConfig.ACKS_CONFIG, "all");
    return new DefaultKafkaProducerFactory<>(config);
}

// Отправка в транзакции
@Transactional
public void processAndPublish(Order order) {
    orderRepository.save(order);               // DB операция
    kafkaTemplate.send("orders", order.toEvent()); // Kafka операция
    // Обе операции либо применяются, либо откатываются
}

// Consumer side
config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

> [!warning] Exactly-once — не серебряная пуля
> - Значительный overhead по производительности (~30-50%)
> - Работает только внутри Kafka (produce + consume в рамках одного кластера)
> - Не покрывает внешние системы (БД, HTTP вызовы)
> - **Рекомендация:** at-least-once + idempotent consumer — проще и надёжнее в большинстве случаев

---

## 🔹 Producer: надёжность и производительность

### acks — гарантии записи

```
acks=0 — fire-and-forget
  Producer → Broker (leader записал?)
  Не ждёт подтверждения → максимальная скорость, потеря при сбое лидера

acks=1 — leader подтвердил
  Producer → Broker (leader) → ✓ ack
  Если follower не успел синхронизироваться и лидер упал → потеря

acks=all (или acks=-1) — все ISR реплики подтвердили
  Producer → Broker (leader) → Followers → ✓ ack
  Максимальная надёжность, немного медленнее
```

**ISR (In-Sync Replicas)** — реплики, которые не отстают от лидера. `acks=all` гарантирует что все ISR подтвердили запись.

### Batching и производительность

```java
// Настройки producer для производительности
config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);      // размер батча (16KB)
config.put(ProducerConfig.LINGER_MS_CONFIG, 5);            // ждать 5мс для накопления батча
config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy"); // сжатие
config.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432); // буфер 32MB
```

`linger.ms` — ждать указанное время перед отправкой батча. Увеличивает latency, но значительно повышает throughput (больше сообщений в одном запросе).

### Idempotent Producer

```java
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// Автоматически включает: acks=all, retries=MAX_INT, max.in.flight=5
// Каждое сообщение получает sequence number
// Broker отбрасывает дубликаты при retry
```

---

## 🔹 Retention и Compaction — сколько хранить

### Time/Size Retention

```yaml
# Хранить 7 дней (по умолчанию)
retention.ms=604800000

# Хранить максимум 10GB на партицию
retention.bytes=10737418240

# Хранить вечно (для event sourcing)
retention.ms=-1
```

Старые сегменты удаляются когда истёк `retention.ms` **или** превышен `retention.bytes`.

### Log Compaction

Compaction — альтернативная стратегия: хранить **только последнее значение для каждого ключа**.

```
Обычный retention:                 Log Compaction:
key=user-1: event1                 key=user-1: event3 (только последнее)
key=user-2: event2                 key=user-2: event4
key=user-1: event3    →            key=user-3: event5
key=user-2: event4
key=user-3: event5
```

```yaml
cleanup.policy=compact        # вместо delete
min.cleanable.dirty.ratio=0.5 # начать compaction когда 50% записей — дубликаты ключей
```

**Когда использовать compaction:** "changelog" топики где важно последнее состояние (профили пользователей, конфигурация), CDC (Change Data Capture) из БД через Debezium.

---

## 🔹 Replication и надёжность кластера

```
Topic "orders", replication-factor=3:

Partition 0:
  Broker 1 (Leader) ← Producer пишет сюда, Consumer читает отсюда
  Broker 2 (Follower) ← реплицирует от лидера
  Broker 3 (Follower) ← реплицирует от лидера

Broker 1 упал → Kafka выбирает нового лидера из ISR (Broker 2 или 3)
→ Producer/Consumer переключаются автоматически
```

**Recommended settings для продакшена:**
```yaml
replication.factor: 3           # 3 копии данных
min.insync.replicas: 2          # минимум 2 реплики в ISR для записи
acks: all                        # ждать подтверждения от всех ISR
unclean.leader.election: false   # не выбирать лидера из отставших реплик
```

---

## 🔹 Spring Kafka — полный пример

### Конфигурация

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false  # ручной коммит
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events"
    listener:
      ack-mode: MANUAL_IMMEDIATE   # коммитим вручную
```

### Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.from(order);

        // send() возвращает CompletableFuture
        kafkaTemplate.send("orders", order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event: {}", order.getId(), ex);
                } else {
                    log.info("Published order {} to partition {} offset {}",
                        order.getId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }

    // Синхронная отправка (редко нужно)
    public void publishSync(Order order) throws Exception {
        kafkaTemplate.send("orders", order.getId().toString(), OrderCreatedEvent.from(order))
            .get(5, TimeUnit.SECONDS); // ждём подтверждения
    }
}
```

### Consumer — базовый

```java
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final OrderProcessingService processingService;

    @KafkaListener(
        topics = "orders",
        groupId = "billing-service",
        concurrency = "3"  // 3 потока = читает 3 партиции параллельно
    )
    public void handleOrderCreated(
            OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Processing order {} from partition {} offset {}",
            event.getOrderId(), partition, offset);

        try {
            processingService.process(event);
            ack.acknowledge(); // коммитим только после успешной обработки
        } catch (NonRetryableException e) {
            log.error("Non-retryable error for order {}", event.getOrderId(), e);
            ack.acknowledge(); // коммитим чтобы не застрять, идём в DLT
        }
    }
}
```

### Consumer с Retry и Dead Letter Topic

```java
@Component
public class ResilientOrderConsumer {

    @RetryableTopic(
        attempts = "4",                           // 1 попытка + 3 retry
        backoff = @Backoff(
            delay = 1000,                         // первый retry через 1сек
            multiplier = 2.0,                     // экспоненциальный backoff: 1s, 2s, 4s
            maxDelay = 10000                      // максимум 10сек
        ),
        dltStrategy = DltStrategy.FAIL_ON_ERROR,  // в DLT после всех попыток
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE
        // создаёт топики: orders-retry-0, orders-retry-1, orders-retry-2, orders-dlt
    )
    @KafkaListener(topics = "orders", groupId = "billing-service")
    public void handle(OrderCreatedEvent event) {
        billingService.chargeCustomer(event); // если бросит исключение → retry
    }

    // Обработчик Dead Letter Topic
    @DltHandler
    public void handleDlt(OrderCreatedEvent event,
                          @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.error("Message failed all retries, topic={}, orderId={}",
            topic, event.getOrderId());
        alertService.sendAlert("Order processing failed: " + event.getOrderId());
        // Сохранить в БД для ручного разбора
        failedEventsRepository.save(FailedEvent.from(event, topic));
    }
}
```

### Batch Consumer — обработка пачкой

```java
@KafkaListener(topics = "orders", groupId = "analytics",
               batch = "true",          // получаем List сообщений
               concurrency = "2")
public void handleBatch(
        List<OrderCreatedEvent> events,
        Acknowledgment ack) {

    log.info("Processing batch of {} orders", events.size());

    // Обрабатываем все разом — эффективнее для аналитики
    analyticsService.processBatch(events);
    ack.acknowledge(); // коммитим весь батч
}
```

### Programmatic Consumer (без аннотаций)

```java
@Bean
public ConcurrentMessageListenerContainer<String, OrderCreatedEvent> orderContainer(
        ConsumerFactory<String, OrderCreatedEvent> cf) {

    ContainerProperties props = new ContainerProperties("orders");
    props.setMessageListener((MessageListener<String, OrderCreatedEvent>) record -> {
        processOrder(record.value());
    });
    props.setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

    ConcurrentMessageListenerContainer<String, OrderCreatedEvent> container =
        new ConcurrentMessageListenerContainer<>(cf, props);
    container.setConcurrency(3);
    return container;
}
```

### Транзакционная отправка (Outbox Pattern)

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    @Transactional  // DB транзакция
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.from(request);
        orderRepository.save(order);

        // Kafka отправка в той же транзакции (если kafkaTemplate настроен transactionally)
        kafkaTemplate.executeInTransaction(kt ->
            kt.send("orders", order.getId().toString(), OrderCreatedEvent.from(order))
        );

        return order;
        // Если DB commit упал → Kafka сообщение тоже откатится
    }
}
```

---

## 🔹 Monitoring — что смотреть

**Ключевые метрики:**

```
Consumer Lag — главная метрика
  = latest offset - committed offset
  Показывает насколько consumer отстаёт от producer
  Большой lag → consumer не справляется → нужно масштабировать

Producer метрики:
  record-send-rate       — сообщений в секунду
  request-latency-avg    — средняя задержка запроса
  record-error-rate      — процент ошибок

Consumer метрики:
  records-consumed-rate  — сообщений обработано в секунду
  fetch-latency-avg      — задержка получения сообщений
  commit-latency-avg     — задержка коммита offset
```

```java
// Получить consumer lag программно
AdminClient adminClient = AdminClient.create(config);
Map<TopicPartition, OffsetAndMetadata> committed = adminClient
    .listConsumerGroupOffsets("my-group")
    .partitionsToOffsetAndMetadata().get();

Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> latest = adminClient
    .listOffsets(topicPartitions.stream()
        .collect(Collectors.toMap(tp -> tp, tp -> OffsetSpec.latest())))
    .all().get();

// lag = latest.offset - committed.offset
```

**Инструменты мониторинга:** Kafka UI (Provectus), Confluent Control Center, Grafana + Prometheus (JMX exporter).

---

## 🔹 Kafka Streams — обработка потоков

Kafka Streams — библиотека для обработки данных прямо в Kafka, без внешних систем.

```java
@Bean
public KStream<String, OrderEvent> orderStream(StreamsBuilder builder) {
    KStream<String, OrderEvent> orders = builder.stream("orders");

    // Фильтрация + трансформация
    orders
        .filter((key, order) -> order.getAmount().compareTo(BigDecimal.valueOf(1000)) > 0)
        .mapValues(order -> new HighValueOrderAlert(order.getId(), order.getAmount()))
        .to("high-value-orders-alerts");

    // Агрегация: считать заказы по пользователю за окно времени
    orders
        .groupByKey()
        .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
        .count()
        .toStream()
        .to("order-counts-per-user");

    return orders;
}
```

---

## 🔹 Типичные паттерны и антипаттерны

### ✅ Паттерны

**Event-Driven Architecture:**
```
OrderService → "order-created" → BillingService (списать деньги)
                               → WarehouseService (зарезервировать товар)
                               → NotificationService (уведомить пользователя)
```

**Outbox Pattern** — гарантированная публикация при сбоях:
```
1. В одной DB транзакции: сохранить Order + запись в таблицу outbox_events
2. Отдельный процесс читает outbox_events → публикует в Kafka → помечает как sent
Гарантирует: если Order сохранён → событие точно будет опубликовано
```

**CQRS + Event Sourcing:**
```
Command → Kafka → Event Store (все события как source of truth)
                → Read Model (проекция для queries)
```

### ❌ Антипаттерны

```
❌ Kafka как синхронный request-response
   Не используй Kafka там где нужен ответ прямо сейчас → используй gRPC/REST

❌ Слишком маленькие топики (1-2 партиции)
   Ограничивает параллелизм. Партиции не уменьшить → планируй заранее

❌ Огромные сообщения (> 1MB)
   Kafka оптимизирован для маленьких сообщений. Большие файлы → храни в S3,
   в Kafka отправляй только ссылку

❌ enable.auto.commit=true в продакшене
   Риск потери сообщений. Всегда manual commit с явным acknowledge()

❌ Не следить за Consumer Lag
   Lag растёт незаметно → в какой-то момент consumer катастрофически отстаёт
```

---

## 🔹 Итог

```
Kafka = распределённый append-only log

Архитектура:
  Topic → Partitions → Offsets (позиция сообщения)
  Producer → Topic ← Consumer Group
  Несколько Consumer Groups = независимые читатели

Партиции и масштабирование:
  key → hash → одна партиция → порядок гарантирован для ключа
  Consumers ≤ partitions. Хочешь больше параллелизма → больше партиций
  CooperativeStickyAssignor — лучшая стратегия rebalancing

Offset Management:
  enable.auto.commit=false + ack.acknowledge() после обработки
  auto.offset.reset: earliest (новый consumer) / latest (только свежие)

Гарантии доставки:
  at-most-once  — commit ДО обработки. Потери возможны.
  at-least-once — commit ПОСЛЕ обработки. Дубликаты возможны. DEFAULT.
  exactly-once  — idempotent producer + transactions. Overhead.
  Рекомендация: at-least-once + idempotent consumer (проверка eventId)

Producer надёжность:
  acks=all + enable.idempotence=true + retries → без потерь
  linger.ms > 0 + batch.size → throughput

Retention:
  retention.ms  — хранить по времени (default 7 дней)
  retention.bytes — хранить по размеру
  cleanup.policy=compact — только последнее значение для ключа

Spring Kafka:
  KafkaTemplate.send(topic, key, value)      — producer
  @KafkaListener(topics, groupId)            — consumer
  @RetryableTopic + @DltHandler              — retry + DLT
  Acknowledgment.acknowledge()               — ручной commit

Мониторинг: Consumer Lag — главная метрика
```
