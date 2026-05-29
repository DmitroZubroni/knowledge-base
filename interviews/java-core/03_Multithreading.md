# Multithreading

> **Теги:** #interviews #java-core #multithreading #concurrency #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 synchronized

**synchronized** — ключевое слово для обеспечения потокобезопасности через монитор (lock).

### Варианты использования

```java
// 1. Синхронизация метода (instance)
public synchronized void method() {
    // блокируется this
}

// 2. Синхронизация статического метода (class)
public static synchronized void staticMethod() {
    // блокируется MyClass.class
}

// 3. Синхронизация блока
public void block() {
    synchronized (lock) {
        // блокируется объект lock
    }
}
```

### Как работает

1. Поток пытается захватить монитор (monitor)
2. Если монитор свободен — поток захватывает его
3. Если монитор занят — поток переходит в состояние BLOCKED
4. После выхода из synchronized блока монитор освобождается

### Проблемы

- **Блокировка всего метода** — даже если критическая секция маленькая
- **Deadlock** — взаимная блокировка потоков
- **Производительность** — контекст свитчинг при блокировке

> [!warning] synchronized vs ReentrantLock
- synchronized — встроенный, неявный unlock
- ReentrantLock — явный lock/unlock, tryLock(), fair lock

---

## 🔹 volatile

**volatile** — ключевое слово для обеспечения видимости переменной между потоками.

### Что гарантирует volatile

1. **Visibility** — изменения volatile переменной сразу видны другим потокам
2. **Ordering** — запрещает reordering инструкций

### Что НЕ гарантирует volatile

- **Atomicity** — операции не атомарны (кроме чтения/записи)

```java
volatile int counter;

// НЕ атомарно! race condition
counter++;  // read → modify → write
```

### Когда использовать volatile

- Флаги остановки потока
- Singleton pattern (double-checked locking)
- Переменные с одним потоком записи и несколькими чтения

```java
private volatile boolean running = true;

public void stop() {
    running = false;  // сразу видна другим потокам
}
```

> [!tip] volatile vs synchronized
- volatile — только visibility, без блокировки
- synchronized — visibility + atomicity + mutual exclusion

---

## 🔹 Atomic классы

**Atomic** — классы из `java.util.concurrent.atomic` для атомарных операций без блокировок (CAS — Compare-And-Swap).

### Основные классы

| Класс | Назначение |
|-------|------------|
| `AtomicInteger` | Атомарные операции над int |
| `AtomicLong` | Атомарные операции над long |
| `AtomicBoolean` | Атомарные операции над boolean |
| `AtomicReference<V>` | Атомарные операции над ссылкой |
| `AtomicIntegerArray` | Атомарные операции над массивом int |

### Основные методы

```java
AtomicInteger counter = new AtomicInteger(0);

counter.get();           // чтение
counter.set(5);         // запись
counter.getAndIncrement();  // вернуть старое, затем ++
counter.incrementAndGet();   // ++, затем вернуть новое
counter.compareAndSet(expected, update);  // CAS
```

### Как работает CAS

```
1. Читаем текущее значение
2. Вычисляем новое значение
3. Сравниваем: если текущее == ожидаемое → записываем новое
4. Если не равно → повторяем (retry)
```

> [!tip] Когда использовать Atomic
- Счётчики в многопоточной среде
- Когда нужна атомарность без блокировок
- Когда операции простые (get, set, increment)

---

## 🔹 ConcurrentHashMap

**ConcurrentHashMap** — потокобезопасная реализация HashMap из `java.util.concurrent`.

### Отличия от HashMap

| Характеристика | HashMap | ConcurrentHashMap |
|----------------|---------|-------------------|
| Потокобезопасность | Нет | Да |
| Производительность | Высокая (single-thread) | Высокая (multi-thread) |
| Блокировка | — | Fine-grained (по сегментам/бакетам) |

### Как работает

**Java 7:** сегментация (Segment-based locking)
- Куча разделена на сегменты (16 по умолчанию)
- Каждый сегмент имеет свой lock
- Операции в разных сегментах параллельны

**Java 8+:** блокировка по бакетам (bucket-level locking)
- Lock только на конкретный бакет
- При чтении — без блокировки
- При записи — lock на head бакета

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("key", 1);        // lock на бакет
map.get("key");           // без блокировки
map.computeIfAbsent("key", k -> 0);  // атомарная операция
```

### Атомарные операции

```java
map.putIfAbsent(key, value);      // если нет ключа
map.replace(key, oldValue, newValue);  // CAS
map.compute(key, (k, v) -> v + 1);     // атомарная функция
```

> [!tip] ConcurrentHashMap vs Hashtable
- Hashtable — synchronized на весь map, медленно
- ConcurrentHashMap — fine-grained locking, быстро

---

## 🔹 Executor Service

**ExecutorService** — фреймворк для управления пулом потоков.

### Создание пулов

```java
// 1. Fixed thread pool
ExecutorService fixed = Executors.newFixedThreadPool(10);

// 2. Cached thread pool
ExecutorService cached = Executors.newCachedThreadPool();

// 3. Single thread executor
ExecutorService single = Executors.newSingleThreadExecutor();

// 4. Scheduled thread pool
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
```

### Отправка задач

```java
// Runnable — без возвращаемого значения
executor.submit(() -> {
    System.out.println("Task");
});

// Callable — с возвращаемым значением
Future<Integer> future = executor.submit(() -> {
    return 42;
});

Integer result = future.get();  // блокирует до завершения
```

### ScheduledExecutorService

```java
// Запуск с задержкой
scheduled.schedule(task, 1, TimeUnit.SECONDS);

// Периодический запуск (fixed rate)
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);

// Периодический запуск (fixed delay)
scheduled.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);
```

### Завершение работы

```java
executor.shutdown();  // не принимает новые задачи
executor.awaitTermination(1, TimeUnit.MINUTES);
executor.shutdownNow();  // принудительное завершение
```

> [!warning] Executors.newCachedThreadPool()
- Создаёт неограниченное количество потоков
- Может привести к OutOfMemoryError
- Лучше использовать ThreadPoolExecutor с ограничением

---

## 🔹 Виртуальные потоки (Virtual Threads)

**Virtual Threads** — лёгкие потоки, введённые в Java 21 (Project Loom).

### Отличия от platform threads

| Характеристика | Platform Thread | Virtual Thread |
|----------------|-----------------|----------------|
| Вес | Тяжёлый (1MB stack) | Лёгкий (KB stack) |
| Создание | Медленно | Быстро |
| Количество | Ограничено (тысячи) | Миллионы |
| Блокировка | Блокирует OS поток | Mount/Unmount |

### Создание виртуальных потоков

```java
// 1. Через фабрику
Thread vThread = Thread.ofVirtual().start(() -> {
    System.out.println("Virtual thread");
});

// 2. Через ExecutorService
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        System.out.println("Task in virtual thread");
    });
}
```

### Когда использовать

- Высококонкурентные приложения (миллионы запросов)
- I/O-bound операции (HTTP запросы, БД)
- Когда нужно много блокирующих вызовов

> [!tip] Virtual Threads vs Reactive
- Virtual threads — пишем как синхронный код
- Reactive — сложный код с callbacks/mono/flux

---

## 🔹 Race Condition

**Race Condition** — состояние гонки, когда результат зависит от порядка выполнения потоков.

### Пример

```java
int counter = 0;

// Поток 1: counter++
// Поток 2: counter++

// Возможный результат: 1 вместо 2
// counter++ не атомарно: read → modify → write
```

### Решения

1. **synchronized**
```java
synchronized (lock) {
    counter++;
}
```

2. **AtomicInteger**
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
```

3. **LongAdder** (для высоких нагрузок)
```java
LongAdder counter = new LongAdder();
counter.increment();
```

---

## 🔹 Deadlock

**Deadlock** — взаимная блокировка, когда потоки ждут друг друга бесконечно.

### Условия возникновения (4 условия)

1. **Mutual Exclusion** — ресурс может быть занят только одним потоком
2. **Hold and Wait** — поток удерживает ресурс и ждёт другой
3. **No Preemption** — ресурс нельзя отобрать принудительно
4. **Circular Wait** — циклическое ожидание (A ждёт B, B ждёт A)

### Пример

```java
Object lock1 = new Object();
Object lock2 = new Object();

// Поток 1
synchronized (lock1) {
    Thread.sleep(100);
    synchronized (lock2) {
        // ...
    }
}

// Поток 2
synchronized (lock2) {
    Thread.sleep(100);
    synchronized (lock1) {
        // ...
    }
}
```

### Как избежать

- Всегда захватывать блокировки в одном порядке
- Использовать tryLock() с таймаутом
- Минимизировать критические секции

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **synchronized** — монитор, блокировка, deadlock risk
> - **volatile** — visibility, без atomicity, флаги
> - **Atomic** — CAS, без блокировок, счётчики
> - **ConcurrentHashMap** — fine-grained locking, атомарные методы
> - **ExecutorService** — пул потоков, Callable/Future
> - **Virtual Threads** — лёгкие, миллионы, Java 21+

```
synchronized:
метод/блок → монитор → BLOCKED если занят

volatile:
visibility гарантирован, atomicity — нет

Atomic:
CAS (compare-and-set), без блокировок

ConcurrentHashMap:
Java 7: сегменты, Java 8+: бакеты

ExecutorService:
Fixed/Cached/Single/Scheduled пулы

Virtual Threads:
лёгкие, mount/unmount при блокировке
```
