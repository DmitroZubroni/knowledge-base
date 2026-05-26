# Apache Kafka

> **Теги:** #tools #kafka #конспект  

> [!abstract] Связи
> [[main]] | [[main Tools]]

---

## 🔹 Что такое Kafka

> [!note] Определение
> **Apache Kafka** — распределённый **event streaming** брокер: publish/subscribe, хранение, обработка потоков событий в реальном времени.

| vs | RabbitMQ | Kafka |
|----|----------|-------|
| Модель | Queues, routing | Log (append-only) |
| Throughput | Средний | Очень высокий |
| Retention | После ack удаляется | Хранит по времени/размеру |
| Replay | Сложно | ✅ Offset rewind |
| Когда | Task queues, RPC-style | Event sourcing, analytics, CDC |

---

## 🔹 Архитектура

```
Producer ──► Topic (partition 0, 1, 2, ...) ──► Consumer Group
                │
           Kafka Broker Cluster
                │
           ZooKeeper / KRaft (metadata)
```

| Компонент | Роль |
|-----------|------|
| **Broker** | Сервер, хранит partitions |
| **Topic** | Категория событий (`orders`, `payments`) |
| **Partition** | Упорядоченный log внутри topic |
| **Offset** | Позиция сообщения в partition |
| **Consumer Group** | Несколько consumers делят partitions |
| **Producer** | Пишет в topic |
| **Replication Factor** | Копии partition на разных brokers |

> [!tip] Порядок
> Гарантирован **внутри partition**, не внутри всего topic. Key → одна partition → ordered.

---

## 🔹 Producer

```java
kafkaTemplate.send("orders", order.getId().toString(), orderEvent);
//                    topic      key (partition routing)  value
```

| Параметр | Смысл |
|----------|-------|
| `acks=0` | Fire-and-forget |
| `acks=1` | Leader записал |
| `acks=all` | Все ISR replicas | ✅ durability |
| `retries` | Повтор при transient failure |
| `idempotence=true` | Без дубликатов при retry |

---

## 🔹 Consumer Group

```
Topic "orders" — 3 partitions
Consumer Group "billing-service":
    Consumer-1 → partition 0
    Consumer-2 → partition 1
    Consumer-3 → partition 2

Consumer Group "analytics-service":
    (свои offsets, независимо от billing)
```

> [!note] Масштабирование
> Consumers ≤ partitions. Лишние consumers idle.

**Offset commit:**

```java
@KafkaListener(topics = "orders", groupId = "billing-service")
public void handle(OrderEvent event) {
    process(event);
    // auto-commit или manual commit после успешной обработки
}
```

| Режим | Поведение |
|-------|-----------|
| `enable.auto.commit=true` | Commit периодически (at-most-once risk) |
| Manual commit | Commit после обработки (at-least-once) |

---

## 🔹 Delivery semantics

| Семантика | Как | Trade-off |
|-----------|-----|-------------|
| **At-most-once** | Commit до обработки | Потеря сообщений |
| **At-least-once** | Commit после обработки | Дубликаты возможны |
| **Exactly-once** | Transactions + idempotent producer | Сложнее, overhead |

> [!tip] Практика
> **At-least-once** + **idempotent consumer** (проверка `eventId`) — типичный production паттерн.

---

## 🔹 Spring Kafka

```gradle
implementation 'org.springframework.kafka:spring-kafka'
```

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publish(OrderEvent event) {
        kafkaTemplate.send("orders", event.orderId().toString(), event);
    }
}

@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "orders", groupId = "notification-service")
    public void onOrder(OrderEvent event) {
        notificationService.sendConfirmation(event);
    }

    @KafkaListener(topics = "orders", groupId = "notification-service",
                   containerFactory = "retryKafkaListenerContainerFactory")
    public void onOrderWithRetry(OrderEvent event) { }
}
```

**Dead Letter Topic (DLT):**

```java
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) { }
```

---

## 🔹 Schema Registry (Avro / JSON)

| | JSON | Avro + Schema Registry |
|---|------|------------------------|
| Размер | Больше | Компактный binary |
| Evolution | Ручной контроль | Schema compatibility rules |
| Tooling | Просто | Confluent ecosystem |

> [!tip] Production
> При частых изменениях схемы — **Schema Registry** + backward-compatible evolution.

---

## 🔹 Когда использовать Kafka

| ✅ Kafka | ❌ Не Kafka |
|----------|-------------|
| Event-driven microservices | Простая task queue на 100 msg/day |
| Audit log / event sourcing | Синхронный request-response |
| Real-time analytics pipeline | Нужен routing по сложным правилам (→ RabbitMQ) |
| CDC (Debezium) | Маленький проект без ops capacity |

---

## 🔹 Итог

1. Topic → partitions → ordered log.
2. Consumer Group — параллелизм + независимые offsets.
3. `acks=all` + idempotent producer — durability.
4. At-least-once + idempotent handler — практичный default.
5. Spring: `KafkaTemplate` + `@KafkaListener`.

```
Шпаргалка Kafka:
─────────────────────────────────────────
Topic / Partition / Offset
Consumer Group → 1 consumer per partition max
acks=all + idempotence           → durable produce
@KafkaListener(groupId=...)      → consume
Key → same partition → ordering
At-least-once + idempotent handler
RetryableTopic / DLT             → error handling
```
