# 09 — Многопоточность (Multithreading)

> **Теги:** #java #programming #concurrency #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_JAVA]]

---

## 🔹 Процессы и потоки

```
Процесс (Process)
├── Собственная память
├── Собственные ресурсы
└── Содержит потоки (Thread)
    ├── Thread 1 (main)       ← Начальный поток
    ├── Thread 2
    └── Thread 3

Потоки одного процесса:
✅ Разделяют Heap (объекты, переменные)
✅ Разделяют static поля
❌ НЕ разделяют Stack (у каждого свой)
```

---

## 🔹 Создание потоков

### Способ 1: extends Thread
```java
public class MyThread extends Thread {
    private String name;

    public MyThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(name + ": " + i);
            try {
                Thread.sleep(100);  // пауза 100мс
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();  // восстановить флаг прерывания
                break;
            }
        }
    }
}

MyThread t1 = new MyThread("Thread-1");
MyThread t2 = new MyThread("Thread-2");
t1.start();  // ❗ start(), не run()! run() — это обычный вызов метода без нового потока
t2.start();
```

### Способ 2: implements Runnable (предпочтительнее)
```java
// Отделяет задачу от управления потоком
public class CountTask implements Runnable {
    private final String label;

    public CountTask(String label) {
        this.label = label;
    }

    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(label + ": " + i);
        }
    }
}

Thread t = new Thread(new CountTask("Worker"));
t.start();

// Или с лямбдой (Runnable — функциональный интерфейс)
Thread t2 = new Thread(() -> {
    for (int i = 0; i < 5; i++) {
        System.out.println("Lambda: " + i);
    }
});
t2.start();
```

> [!tip] Рекомендация
> Предпочитай `Runnable` над `extends Thread` — Java не поддерживает множественное наследование, а задача и поток — разные концепции.

---

## 🔹 Жизненный цикл потока

```
         start()              получил CPU
NEW ──────────→ RUNNABLE ←──────────────→ (выполняется)
                    │                            │
                    │ sleep(), wait(),            │ завершился
                    │ join(), I/O wait            │
                    ↓                            ↓
              BLOCKED/WAITING             TERMINATED
              TIMED_WAITING
                    │
                    │ время истекло /
                    │ notify() / lock освободился
                    ↓
                RUNNABLE
```

| Состояние | Описание |
|-----------|----------|
| `NEW` | Создан, но `start()` не вызван |
| `RUNNABLE` | Выполняется или готов к выполнению |
| `BLOCKED` | Ожидает захвата монитора (`synchronized`) |
| `WAITING` | Ожидает бессрочно (`wait()`, `join()`) |
| `TIMED_WAITING` | Ожидает с таймаутом (`sleep(ms)`, `wait(ms)`) |
| `TERMINATED` | Завершил выполнение |

---

## 🔹 Методы Thread

```java
Thread t = new Thread(() -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
});

t.start();               // запустить поток (создаёт новый поток OS)
t.join();                // ждать завершения t в текущем потоке
t.join(1000);            // ждать не дольше 1000мс
t.interrupt();           // запросить прерывание потока (флаг)
t.isAlive();             // true если поток жив
t.getName();             // имя потока
t.setName("Worker-1");   // установить имя
t.getPriority();         // приоритет 1-10 (default 5)
t.setPriority(Thread.MAX_PRIORITY);  // 10

Thread.sleep(500);       // СТАТИЧЕСКИЙ — усыпить ТЕКУЩИЙ поток на 500мс
Thread.currentThread();  // СТАТИЧЕСКИЙ — ссылка на текущий поток
Thread.currentThread().isInterrupted();  // проверить флаг прерывания
```

> [!warning] Подводные камни
> - `t.run()` — НЕ создаёт новый поток, просто вызывает метод
> - `Thread.sleep()` — прерывается `InterruptedException`, нужно обрабатывать
> - После `interrupt()` сам поток должен проверять флаг и реагировать

---

## 🔹 Проблема гонки данных (Race Condition)

> [!note] Определение
> Race condition — несколько потоков обращаются к общим данным одновременно, и результат зависит от порядка выполнения.

```java
// Пример: атомарность инкремента
public class Counter {
    private int count = 0;

    public void increment() {
        count++;  // НЕ атомарная операция!
        // На самом деле: 1) read count   2) add 1   3) write count
        // Между шагами поток может быть прерван другим потоком!
    }

    public int getCount() { return count; }
}

Counter counter = new Counter();

// Запускаем 2 потока, каждый 1000 раз инкрементирует
Thread t1 = new Thread(() -> { for (int i = 0; i < 1000; i++) counter.increment(); });
Thread t2 = new Thread(() -> { for (int i = 0; i < 1000; i++) counter.increment(); });
t1.start(); t2.start();
t1.join(); t2.join();

System.out.println(counter.getCount()); // НЕ 2000! Может быть 1843, 1756...
```

---

## 🔹 synchronized — синхронизация

> [!note] Определение
> `synchronized` — гарантирует что в критической секции находится только **один поток** в момент времени. Основан на мониторе объекта.

### Способ 1: synchronized блок по объекту
```java
public class SafeCounter {
    private int count = 0;
    private final Object lock = new Object();  // объект-монитор

    public void increment() {
        synchronized (lock) {
            count++;  // критическая секция — только один поток
        }
    }
}
```

### Способ 2: synchronized метод (монитор = this)
```java
public class SafeCounter {
    private int count = 0;

    public synchronized void increment() {  // монитор = this
        count++;
    }

    public synchronized int getCount() {    // тот же монитор
        return count;
    }
}
```

### Способ 3: synchronized статический метод (монитор = Class объект)
```java
public class Registry {
    private static int nextId = 0;

    public static synchronized int getNextId() {  // монитор = Registry.class
        return ++nextId;
    }
}
```

### Как работает монитор
```
Поток 1 входит в synchronized блок → захватывает монитор
Поток 2 пытается войти → видит монитор занят → переходит в BLOCKED
Поток 1 выходит из synchronized → освобождает монитор
Поток 2 захватывает монитор → входит в критическую секцию
```

> [!warning] Подводные камни
> - `synchronized` снижает производительность — сужай критическую секцию до минимума
> - Не `synchronized` на `this` для публичных объектов — внешний код тоже может синхронизироваться на `this`
> - Вложенные `synchronized` блоки → опасность deadlock

---

## 🔹 volatile

> [!note] Определение
> `volatile` гарантирует, что **все потоки видят актуальное значение** переменной (не кешированное). НЕ решает проблему атомарности составных операций.

```java
public class Worker {
    private volatile boolean running = true;  // видимость между потоками

    public void stop() {
        running = false;  // изменение видно другим потокам сразу
    }

    public void work() {
        while (running) {  // читает актуальное значение из памяти, не из кеша
            doWork();
        }
        System.out.println("Stopped");
    }
}

// volatile НЕДОСТАТОЧНО для атомарности:
private volatile int count = 0;
count++;  // ❌ всё ещё не атомарно: read-increment-write = 3 шага
```

### JMM (Java Memory Model)
```
Без volatile:                    С volatile:
─────────────                    ────────────
CPU Cache L1  →  Thread 1        CPU Cache  →  Thread 1
CPU Cache L2  →  Thread 2        Main Memory (всегда актуально)
Main Memory                      CPU Cache  →  Thread 2
               ↑ потоки могут
               видеть разные значения
```

---

## 🔹 Атомарные типы

> [!note] Определение
> Классы из `java.util.concurrent.atomic` предоставляют атомарные операции без `synchronized`. Основаны на CAS (Compare-And-Swap).

```java
import java.util.concurrent.atomic.*;

AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();          // атомарный ++, возвращает новое значение
counter.getAndIncrement();          // атомарный ++, возвращает старое значение
counter.decrementAndGet();          // атомарный --
counter.addAndGet(5);               // атомарный += 5
counter.get();                      // просто прочитать
counter.set(10);                    // установить значение
counter.compareAndSet(10, 20);      // если значение == 10, то установить 20

// Другие типы
AtomicLong longCounter = new AtomicLong(0L);
AtomicBoolean flag = new AtomicBoolean(false);
AtomicReference<String> ref = new AtomicReference<>("initial");
ref.compareAndSet("initial", "updated");

// Пример безопасного счётчика
public class SafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); }
    public int getCount() { return count.get(); }
}
```

| | `int` | `volatile int` | `AtomicInteger` | `synchronized int` |
|-|-------|----------------|----------------|--------------------|
| Видимость | ❌ | ✅ | ✅ | ✅ |
| Атомарность | ❌ | ❌ | ✅ | ✅ |
| Производительность | Лучшая | Хорошая | Хорошая | Хуже всего |

---

## 🔹 Проблемы многопоточности

### Deadlock (взаимная блокировка)
```java
Object lockA = new Object();
Object lockB = new Object();

// Поток 1: захватывает A, ждёт B
Thread t1 = new Thread(() -> {
    synchronized (lockA) {
        Thread.sleep(100);
        synchronized (lockB) {  // ждёт пока t2 отпустит lockB
            System.out.println("T1 got both");
        }
    }
});

// Поток 2: захватывает B, ждёт A → DEADLOCK
Thread t2 = new Thread(() -> {
    synchronized (lockB) {
        Thread.sleep(100);
        synchronized (lockA) {  // ждёт пока t1 отпустит lockA → зависание
            System.out.println("T2 got both");
        }
    }
});
```

> [!danger] Предотвращение Deadlock
> - Всегда захватывай блокировки в **одном порядке**
> - Используй `tryLock()` из `ReentrantLock` с таймаутом
> - Минимизируй вложенные `synchronized` блоки

### Другие проблемы
```
Livelock    — потоки активны, но не прогрессируют (бесконечно уступают друг другу)
Starvation  — поток не получает ресурсы из-за постоянного захвата другими потоками
```

---

## 🔹 ExecutorService и ThreadPool

> [!note] Определение
> `ExecutorService` — управляет пулом потоков, переиспользует их для задач. Лучшая практика для многопоточности в приложениях.

```java
import java.util.concurrent.*;

// Фиксированный пул потоков
ExecutorService executor = Executors.newFixedThreadPool(4);  // 4 потока

// Отправка задачи
executor.submit(() -> {
    System.out.println("Task in: " + Thread.currentThread().getName());
});

// Несколько задач
for (int i = 0; i < 10; i++) {
    final int taskId = i;
    executor.submit(() -> processTask(taskId));
}

// Остановка (дожидается завершения всех задач)
executor.shutdown();
executor.awaitTermination(30, TimeUnit.SECONDS);  // ждать не более 30с

// Немедленная остановка
// executor.shutdownNow();  // прерывает запущенные задачи

// Другие виды пулов
Executors.newSingleThreadExecutor();    // один поток, задачи в очереди
Executors.newCachedThreadPool();        // создаёт потоки по мере надобности
Executors.newScheduledThreadPool(2);    // для периодических задач
```

---

## 🔹 Callable и Future

> [!note] Определение
> `Callable<V>` — как `Runnable`, но возвращает результат и может бросать checked исключения. `Future<V>` — асинхронный результат.

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

// Callable возвращает результат
Callable<Integer> task = () -> {
    Thread.sleep(1000);  // имитация долгой работы
    return 42;
};

Future<Integer> future = executor.submit(task);

// Делаем другую работу пока задача выполняется
System.out.println("Doing other work...");

// Получаем результат (блокирует поток до готовности)
Integer result = future.get();           // ждёт бесконечно
Integer result2 = future.get(2, TimeUnit.SECONDS);  // ждёт не более 2с

future.isDone();       // проверить готовность (не блокирует)
future.isCancelled();
future.cancel(true);   // отменить задачу

executor.shutdown();
```

### CompletableFuture (Java 8+) — асинхронная цепочка
```java
CompletableFuture<Integer> cf = CompletableFuture
    .supplyAsync(() -> fetchData())           // асинхронно получить данные
    .thenApply(data -> processData(data))     // обработать результат
    .thenApply(result -> result * 2)          // ещё преобразование
    .exceptionally(ex -> {                    // обработка ошибок
        System.out.println("Error: " + ex.getMessage());
        return 0;
    });

// Не блокирует поток
cf.thenAccept(result -> System.out.println("Final: " + result));
// Или получить синхронно
int result = cf.get();
```

---

## 🔹 Потокобезопасные коллекции

```java
import java.util.concurrent.*;

// Потокобезопасные варианты
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
BlockingQueue<String> queue = new LinkedBlockingQueue<>();

// Блокирующая очередь (паттерн Producer-Consumer)
BlockingQueue<Integer> buffer = new ArrayBlockingQueue<>(10);

// Producer
Thread producer = new Thread(() -> {
    for (int i = 0; i < 20; i++) {
        buffer.put(i);  // блокирует если очередь полна
    }
});

// Consumer
Thread consumer = new Thread(() -> {
    while (true) {
        Integer item = buffer.take();  // блокирует если очередь пуста
        process(item);
    }
});
```

---

## 🔹 Рекомендации

> [!tip] Лучшие практики
> 1. **Предпочитай `ExecutorService`** вместо ручного управления `Thread`
> 2. **Минимизируй критические секции** — держи `synchronized` блоки как можно меньше
> 3. **Предпочитай атомарные типы** (`AtomicInteger`) вместо `synchronized` для счётчиков
> 4. **Используй `volatile`** только для флагов видимости без составных операций
> 5. **Не разделяй изменяемое состояние** между потоками — лучшая защита от race condition
> 6. **Используй `ConcurrentHashMap`** вместо `HashMap` в многопоточном коде

> [!danger] Антипаттерны
> - `synchronized(this)` в публичных методах — внешний код может тоже синхронизироваться на `this`
> - Блокировка на `String` или boxed примитиве — могут использоваться в пуле
> - `Thread.stop()` — устарел, небезопасен
> - Игнорирование `InterruptedException` без восстановления флага
