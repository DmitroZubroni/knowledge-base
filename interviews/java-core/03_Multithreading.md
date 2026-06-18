> **Теги:** #interviews #java-core #multithreading #concurrency #конспект
> [!abstract] Связи
> [[main]] | [[Interviews]] | [[Concurrency_Index]]

# Multithreading — Вопросы на собесе

---

## 🔹 Состояния потока

```
NEW → RUNNABLE ⇄ BLOCKED       ← ждёт synchronized монитор
               ⇄ WAITING       ← wait() / join() без таймаута
               ⇄ TIMED_WAITING ← sleep(ms) / wait(ms) / join(ms)
               → TERMINATED
```

**BLOCKED vs WAITING:**
- `BLOCKED` — поток хочет войти в `synchronized`, но монитор занят другим потоком
- `WAITING` — поток сам решил подождать (`wait()`, `join()`)

---

## 🔹 Race Condition — два аспекта

**Atomicity:** `count++` — это READ → ADD → WRITE. Три шага, между которыми другой поток может вмешаться.

**Visibility:** изменение одного потока может не быть видно другому из-за кэшей CPU (L1/L2). Каждое ядро хранит свою копию переменной.

**JMM happens-before** — гарантия видимости:
- Выход из `synchronized(lock)` → вход в `synchronized(lock)` на том же объекте
- Запись в `volatile` → чтение той же `volatile` переменной
- `Thread.start()` → любая операция в запущенном потоке
- Все операции потока → `Thread.join()` в другом потоке

---

## 🔹 synchronized

```java
synchronized (lock) { ... }          // монитор — явный объект (рекомендуется)
public synchronized void method()    // монитор — this
public static synchronized void m() // монитор — SomeClass.class
```

**Реентерабелен** — поток может повторно захватить свой монитор без deadlock.

**Гранулярность:** держи в `synchronized` только чтение/запись общих данных. Сетевые вызовы, I/O, долгие вычисления — выноси наружу.

> [!warning] Подводный камень Spring AOP (self-invocation)
> `@Transactional` / `@Cacheable` работают через Proxy. Вызов `this.method()` минует Proxy → аннотация не сработает.

---

## 🔹 volatile

**Что гарантирует:** visibility (запись сразу видна всем) + запрет переупорядочивания инструкций.

**Что НЕ гарантирует:** atomicity составных операций.

```java
volatile boolean running = true;   // ✅ простой флаг — volatile хватает
volatile int counter = 0;
counter++;                          // ❌ всё ещё не атомарно! (READ-ADD-WRITE)
```

**volatile vs synchronized:**
| | volatile | synchronized |
|-|----------|-------------|
| Visibility | ✅ | ✅ |
| Atomicity составных операций | ❌ | ✅ |
| Блокирует потоки | Нет | Да (BLOCKED) |

**Когда достаточно volatile:**
- Простой флаг остановки (`running = false`)
- Double-Checked Locking в Singleton (`volatile` обязателен — без него возможна публикация наполовину сконструированного объекта)
- Публикация ссылки на immutable объект

---

## 🔹 Atomic классы — CAS без блокировок

**CAS (Compare-And-Swap)** — атомарная инструкция CPU:
```
если значение == expected → записать new, вернуть true
иначе → ничего не менять, вернуть false
```

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet()           // ++x, возвращает новое
counter.getAndIncrement()           // x++, возвращает старое
counter.compareAndSet(5, 10)        // CAS: если 5 → поставить 10
counter.updateAndGet(x -> x * 2)    // атомарное обновление функцией
```

**LongAdder** — для очень высокой конкуренции. Хранит несколько ячеек, каждый поток пишет в свою, итог суммируется при `sum()`. Быстрее AtomicLong при записи, но `sum()` — не мгновенный снимок.

| | synchronized | AtomicInteger | LongAdder |
|-|-------------|---------------|-----------|
| Механизм | Монитор | CAS | Несколько ячеек + CAS |
| Потоки ждут (BLOCKED) | Да | Нет (retry) | Нет |
| При высокой конкуренции | Плохо | Средне | **Лучше** |
| Когда использовать | Сложная логика | Счётчик, частое чтение | Метрики, редкое чтение |

---

## 🔹 Deadlock, Livelock, Starvation

**Deadlock** — потоки навечно ждут друг друга. Четыре условия Coffman:
1. Mutual Exclusion — ресурс у одного потока
2. Hold and Wait — держит одно, ждёт другое
3. No Preemption — нельзя отобрать принудительно
4. **Circular Wait** — самое частое, нарушить проще всего

**Решения deadlock:**
- Всегда захватывать локи в **одном и том же порядке** во всех потоках
- `tryLock(timeout)` — попробовать с таймаутом и отступить
- `jstack <PID>` → thread dump → раздел "Found Java-level deadlock"

**Livelock** — потоки активны (RUNNABLE), но не продвигаются. Решение: random backoff (`Thread.sleep(random)`).

**Starvation** — поток никогда не получает ресурс. Решение: `new ReentrantLock(true)` — fair mode (FIFO очередь).

---

## 🔹 ReentrantLock и Locks

**Когда ReentrantLock вместо synchronized:**
```java
lock.tryLock(500, TimeUnit.MILLISECONDS) // попытка с таймаутом — борьба с deadlock
lock.lockInterruptibly()                  // можно прервать через interrupt()
new ReentrantLock(true)                   // fair mode — FIFO, против starvation
```

**Правило:** `unlock()` ВСЕГДА в `finally`. Иначе при исключении лок остаётся захваченным навсегда.

**ReentrantReadWriteLock** — несколько читателей параллельно, писатель — эксклюзивно:
```java
readLock.lock()   // много потоков одновременно
writeLock.lock()  // только один, ждёт всех читателей
```
Использовать: часто читаем, редко пишем (кэши, справочники).

**StampedLock** (Java 8+) — оптимистичное чтение без блокировки:
```java
long stamp = lock.tryOptimisticRead();
// читаем данные
if (!lock.validate(stamp)) {      // были ли изменения?
    stamp = lock.readLock();      // да → перечитываем с блокировкой
    try { /* перечитать */ } finally { lock.unlockRead(stamp); }
}
```
Максимальная производительность при редких записях. **Не реентерабелен!**

---

## 🔹 ExecutorService и ThreadPool

**Почему не создавать потоки вручную:** создание потока ~1мс, стек ~512KB–1MB. При 1000 запросах/сек — 1GB памяти только на стеки.

**Виды пулов:**
```java
Executors.newFixedThreadPool(n)      // n потоков, unbounded очередь (риск OOM!)
Executors.newCachedThreadPool()      // динамический, риск тысяч потоков
Executors.newSingleThreadExecutor()  // строго последовательно
Executors.newScheduledThreadPool(n)  // расписание
```

**В продакшене** — явный `ThreadPoolExecutor`:
```java
new ThreadPoolExecutor(
    corePoolSize, maxPoolSize,
    keepAlive, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),   // ограниченная очередь!
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure
)
```

**Алгоритм:** задача пришла → `< core` потоков → создать поток → `< queue` места → в очередь → `< max` потоков → создать временный → иначе → RejectedExecutionHandler.

**Завершение:** `shutdown()` (мягко) → `awaitTermination(30s)` → `shutdownNow()` (interrupt).

---

## 🔹 Future и CompletableFuture

**Future — ограничения:**
- `get()` блокирует поток — тратим поток на ожидание
- Нельзя объединить несколько Future
- Нельзя добавить callback
- Нельзя построить цепочку

**CompletableFuture решает всё:**
```java
// Создание
CompletableFuture.supplyAsync(() -> fetchUser(id), executor)

// Цепочка
    .thenApply(user -> enrich(user))          // трансформация
    .thenAccept(user -> save(user))            // потребление
    .exceptionally(ex -> fallback)             // ошибка

// Объединение параллельных запросов
CompletableFuture.allOf(cf1, cf2, cf3)
    .thenApply(v -> combine(cf1.join(), cf2.join(), cf3.join()))

// Первый готовый
CompletableFuture.anyOf(cf1, cf2, cf3)
```

**get() vs join():** `get()` бросает checked exceptions, `join()` — unchecked `CompletionException`. В лямбдах удобнее `join()`.

**Правило:** всегда передавай явный `executor` в `supplyAsync`. Иначе используется `ForkJoinPool.commonPool()` — общий на всё приложение, можно заблокировать.

---

## 🔹 Потокобезопасные коллекции

| Задача | Что использовать |
|--------|-----------------|
| Thread-safe Map | `ConcurrentHashMap` |
| Map с порядком | `Collections.synchronizedMap(new LinkedHashMap<>())` |
| Список, чтений >> записей | `CopyOnWriteArrayList` |
| Очередь Producer-Consumer | `ArrayBlockingQueue` / `LinkedBlockingQueue` |
| Lock-free очередь | `ConcurrentLinkedQueue` |

**ConcurrentHashMap:** null запрещён, `compute/merge/putIfAbsent` — атомарны, итератор — weakly consistent (не бросает ConcurrentModificationException).

**BlockingQueue методы:**
- `put(e)` — блокирует если полная
- `take()` — блокирует если пустая
- Паттерн Poison Pill — специальный sentinel в очереди для сигнала завершения

---

## 🔹 Virtual Threads (Java 21)

| | Platform Thread | Virtual Thread |
|-|-----------------|----------------|
| Стек | ~1MB | Несколько KB |
| Создание | Дорого | Дёшево |
| Количество | Тысячи | Миллионы |
| При блокировке | OS поток простаивает | Отмонтируется (mount/unmount) |

```java
// Создание
Thread.ofVirtual().start(() -> doWork());

// Через executor (рекомендуется)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> handleRequest());
}
```

**Когда использовать:** I/O-bound задачи (HTTP, БД, файлы). **Не подходит** для CPU-bound (не ускорит вычисления).

**Virtual vs Reactive:** виртуальные потоки — пишем обычный синхронный код. Reactive (WebFlux, Mono/Flux) — сложнее, но больше контроля над backpressure.

---

## 🔹 Типичные вопросы и ответы

**Q: Чем отличается synchronized от ReentrantLock?**
A: synchronized — встроенный, автоматический unlock, нет tryLock. ReentrantLock — явный lock/unlock (в finally!), есть tryLock с таймаутом, lockInterruptibly, fair mode, Condition.

**Q: Что такое happens-before?**
A: Гарантия JMM: если A happens-before B, то все изменения от A видны при выполнении B. Устанавливается через synchronized, volatile, Thread.start/join.

**Q: Почему volatile недостаточно для count++?**
A: `count++` — это READ-ADD-WRITE. volatile гарантирует видимость каждого отдельного чтения и записи, но не атомарность всей операции в целом. Между READ и WRITE другой поток может изменить значение.

**Q: Что произойдёт если не вызвать shutdown() у ExecutorService?**
A: Потоки в пуле — user threads. JVM не завершится пока они живы. Приложение зависнет после завершения main().

**Q: В чём разница put() и offer() у BlockingQueue?**
A: `put()` блокирует поток если очередь полна — ждёт места. `offer()` возвращает false если нет места, не блокирует. `offer(e, timeout, unit)` — с таймаутом.

**Q: Как работает ConcurrentHashMap в Java 8+?**
A: При записи — CAS или lock на один бакет (голову списка). При чтении — без блокировки. В отличие от Java 7 где были сегменты (16 по умолчанию).

**Q: Когда использовать LongAdder вместо AtomicLong?**
A: Когда очень много потоков одновременно инкрементируют (высокая конкуренция) и читать значение нужно редко. LongAdder хранит несколько ячеек → меньше CAS-конкуренции. AtomicLong лучше когда часто читаешь текущее значение.

---

## 🔹 Шпаргалка

```
Состояния: NEW → RUNNABLE ⇄ BLOCKED/WAITING/TIMED_WAITING → TERMINATED
  BLOCKED = ждёт synchronized монитор
  WAITING = сам решил подождать (wait/join)

Race Condition:
  Atomicity: count++ = 3 операции → нужен synchronized / Atomic
  Visibility: кэши CPU → нужен volatile / synchronized

volatile = только visibility. count++ volatile — всё равно не атомарно!

synchronized → Atomic → LongAdder (по возрастанию конкуренции)

Deadlock: захватывать локи в одном порядке / tryLock(timeout)
Livelock: random backoff
Starvation: ReentrantLock(true) — fair mode

ReentrantLock: unlock() в finally ВСЕГДА
ReadWriteLock: много читателей параллельно
StampedLock: оптимистичное чтение, не реентерабелен

ExecutorService: никогда не создавай потоки вручную
  В продакшене: ThreadPoolExecutor с ArrayBlockingQueue
  Завершение: shutdown() → awaitTermination() → shutdownNow()

CompletableFuture vs Future:
  Future.get() блокирует — плохо
  CF: цепочки, allOf/anyOf, exceptionally, без блокировок
  В CF: всегда явный executor в supplyAsync

ConcurrentHashMap: null запрещён, compute/merge атомарны
BlockingQueue: put()/take() блокируют — идеально для Producer-Consumer

Virtual Threads (Java 21): миллионы потоков, I/O-bound, mount/unmount при блокировке
```
