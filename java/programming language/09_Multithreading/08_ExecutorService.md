> **Теги:** #java #concurrency #executorservice #threadpool #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Concurrency_Index]] | [[01_Process_Thread]]

# 08 — ExecutorService и пул потоков

---

## 🔹 Почему создавать потоки вручную — плохая идея

В [[01_Process_Thread]] мы создавали потоки через `new Thread(task).start()`. Это работает для простых примеров, но в реальных приложениях так делать нельзя.

Создание потока — **дорогая операция**. JVM должна запросить у операционной системы системный поток, выделить память под стек (~512KB–1MB по умолчанию), зарегистрировать поток в планировщике. Если приложение получает 1000 запросов в секунду и на каждый создаёт новый поток:

```
1000 запросов/сек
× стоимость создания потока (~1мс)
× память на стек (~1MB)
= 1GB памяти + постоянные задержки на создание/уничтожение
```

Это не масштабируется. Решение — **пул потоков (Thread Pool)**: создать потоки заранее, держать их живыми, переиспользовать для новых задач.

```
Без пула:                          С пулом:
Задача1 → [создать поток] → run → [уничтожить]
Задача2 → [создать поток] → run → [уничтожить]    Потоки созданы заранее:
Задача3 → [создать поток] → run → [уничтожить]    ┌─────────┐
          ↑ каждый раз тратим время                │ Поток 1 │ ◄── Задача1
                                                   │ Поток 2 │ ◄── Задача2
                                                   │ Поток 3 │ ◄── Задача3
                                                   └─────────┘
                                                   переиспользуются
```

---

## 🔹 ExecutorService — интерфейс управления пулом

`ExecutorService` — интерфейс, который:
- принимает задачи (`submit`, `execute`)
- управляет жизненным циклом пула (`shutdown`, `awaitTermination`)
- возвращает результаты (`Future`)

```java
import java.util.concurrent.*;

ExecutorService executor = Executors.newFixedThreadPool(4);

// Отправить задачу (Runnable) — без результата
executor.execute(() -> System.out.println("Task running in: "
    + Thread.currentThread().getName()));

// Отправить задачу (Callable) — с результатом
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});

Integer result = future.get(); // блокирует текущий поток до получения результата
System.out.println(result); // 42

// ВАЖНО: всегда выключать executor
executor.shutdown();
```

> [!warning] Всегда вызывай shutdown()
> Потоки в пуле — **user threads**. JVM не завершится, пока они живы. Если не вызвать `shutdown()` — приложение зависнет после завершения `main()`. Всегда выключай ExecutorService в блоке `finally` или через try-with-resources.

---

## 🔹 Виды пулов через Executors

### newFixedThreadPool(n) — фиксированный пул

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
```

- **n потоков** работают постоянно
- Задачи ждут в очереди (unbounded LinkedBlockingQueue), если все потоки заняты
- Подходит для: CPU-intensive задач (число потоков = число ядер)

```
Потоки: [T1][T2][T3][T4]
Очередь: [Задача5][Задача6][Задача7]...
```

> [!warning] Unbounded очередь — риск OOM
> Если задачи поступают быстрее, чем обрабатываются — очередь растёт бесконечно до `OutOfMemoryError`. Для продакшена используй `ThreadPoolExecutor` с ограниченной очередью.

### newCachedThreadPool() — растущий пул

```java
ExecutorService executor = Executors.newCachedThreadPool();
```

- Создаёт новые потоки по мере необходимости
- Простаивающие потоки живут 60 секунд, потом умирают
- Подходит для: много коротких I/O задач

> [!warning] Может создать неограниченное число потоков
> При взрывном росте нагрузки может создать тысячи потоков — риск `OutOfMemoryError`. В продакшене контролируй через `ThreadPoolExecutor`.

### newSingleThreadExecutor() — один поток

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

- Ровно **один поток**, задачи выполняются строго последовательно
- Если поток упал — автоматически создаётся новый
- Подходит для: задачи, которые нельзя выполнять параллельно (запись в файл, последовательная обработка событий)

### newScheduledThreadPool(n) — с расписанием

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Выполнить один раз через 5 секунд
scheduler.schedule(() -> System.out.println("Once after 5s"), 5, TimeUnit.SECONDS);

// Выполнять каждые 10 секунд, начиная через 0 секунд
scheduler.scheduleAtFixedRate(
    () -> System.out.println("Every 10s"),
    0, 10, TimeUnit.SECONDS
);

// Выполнять через 10 секунд ПОСЛЕ завершения предыдущего
scheduler.scheduleWithFixedDelay(
    () -> System.out.println("10s after previous finishes"),
    0, 10, TimeUnit.SECONDS
);
```

`scheduleAtFixedRate` vs `scheduleWithFixedDelay`:
```
scheduleAtFixedRate(period=10s):
  Старт: t=0  t=10  t=20  t=30  — фиксированный ритм
  Если задача занимает 15с: t=0  t=15  t=20 (пропуска нет, но ритм плавает)

scheduleWithFixedDelay(delay=10s):
  Старт: t=0 → [выполнение 5с] → t=15 → [выполнение 5с] → t=30
  Пауза 10с ПОСЛЕ завершения предыдущего — ритм зависит от времени выполнения
```

---

## 🔹 ThreadPoolExecutor — полный контроль

Все `Executors.new*()` методы — это удобные обёртки над `ThreadPoolExecutor`. В продакшене лучше создавать его явно, чтобы контролировать все параметры:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                          // corePoolSize: минимум потоков (всегда живут)
    10,                         // maximumPoolSize: максимум потоков
    60L, TimeUnit.SECONDS,      // keepAliveTime: сколько живёт лишний поток
    new ArrayBlockingQueue<>(100), // очередь задач (ОГРАНИЧЕННАЯ — 100 задач)
    new ThreadFactory() {       // как создавать потоки (можно задать имена)
        int count = 0;
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "worker-" + count++);
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // политика при переполнении
);
```

### Как ThreadPoolExecutor принимает решения

```
Пришла новая задача:
  ↓
  Активных потоков < corePoolSize?
  → Да: создать новый поток (даже если есть простаивающие)
  → Нет: ↓
  
  Очередь не полна?
  → Да: положить в очередь, ждать свободного потока
  → Нет: ↓
  
  Активных потоков < maximumPoolSize?
  → Да: создать новый (временный) поток
  → Нет: ↓
  
  Применить RejectedExecutionHandler (политику отказа)
```

### Политики отказа (RejectedExecutionHandler)

```java
// AbortPolicy (по умолчанию) — бросить RejectedExecutionException
new ThreadPoolExecutor.AbortPolicy()

// CallerRunsPolicy — выполнить задачу в потоке, который вызвал submit()
// Это замедляет отправителя и тем самым регулирует поток задач (back-pressure)
new ThreadPoolExecutor.CallerRunsPolicy()

// DiscardPolicy — молча выбросить задачу
new ThreadPoolExecutor.DiscardPolicy()

// DiscardOldestPolicy — выбросить самую старую задачу из очереди, принять новую
new ThreadPoolExecutor.DiscardOldestPolicy()
```

> [!tip] CallerRunsPolicy — лучший выбор для back-pressure
> Когда пул переполнен, задача выполняется прямо в потоке отправителя. Это автоматически замедляет поступление новых задач — отправитель занят и не может сразу отправить следующую. Элегантное и безопасное решение.

---

## 🔹 Правильное завершение работы

```java
executor.shutdown(); // запретить новые задачи, дождаться завершения текущих

// Ждать завершения максимум 30 секунд
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    // Если не завершился — принудительно прервать всё
    executor.shutdownNow(); // посылает interrupt всем потокам
    if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
        System.err.println("Executor не завершился!");
    }
}
```

`shutdown()` vs `shutdownNow()`:
- `shutdown()` — мягкое завершение: новые задачи не принимаются, текущие и ожидающие в очереди — выполняются до конца
- `shutdownNow()` — жёсткое: прерывает все выполняющиеся задачи через `interrupt()`, возвращает список задач из очереди, которые не успели запуститься

---

## 🔹 Отправка множества задач

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// invokeAll — отправить все, дождаться всех результатов
List<Callable<Integer>> tasks = List.of(
    () -> compute(1),
    () -> compute(2),
    () -> compute(3)
);

List<Future<Integer>> futures = executor.invokeAll(tasks); // блокирует до завершения всех

for (Future<Integer> f : futures) {
    System.out.println(f.get()); // уже готовы, не блокирует
}

// invokeAny — отправить все, вернуть результат первого завершившегося
Integer first = executor.invokeAny(tasks); // остальные задачи отменяются
System.out.println("Первый результат: " + first);
```

---

## 🔹 Мониторинг пула

```java
ThreadPoolExecutor tpe = (ThreadPoolExecutor) executor;

tpe.getPoolSize();          // текущее число потоков
tpe.getActiveCount();       // число потоков, выполняющих задачи
tpe.getCorePoolSize();       // corePoolSize
tpe.getMaximumPoolSize();    // maximumPoolSize
tpe.getQueue().size();      // задач в очереди
tpe.getCompletedTaskCount(); // всего выполнено задач
```

---

## 🔹 Итог

```
Создавать потоки вручную — дорого и не масштабируется.
Используй ExecutorService + пул потоков.

Виды пулов:
  newFixedThreadPool(n)      — n потоков, unbounded очередь
  newCachedThreadPool()      — динамический, риск OOM при взрывном росте
  newSingleThreadExecutor()  — строго последовательно, автовосстановление
  newScheduledThreadPool(n)  — расписание (schedule/scheduleAtFixedRate)

ThreadPoolExecutor — явное управление:
  corePoolSize   — всегда живые потоки
  maxPoolSize    — максимум при пиковой нагрузке
  queue          — ограниченная очередь (ArrayBlockingQueue в продакшене)
  RejectedPolicy — CallerRunsPolicy для back-pressure

Завершение:
  shutdown()     → мягко (текущие задачи доделываются)
  shutdownNow()  → жёстко (interrupt всем потокам)
  ВСЕГДА вызывай shutdown() иначе JVM не завершится

execute(Runnable) — без результата
submit(Callable)  — с результатом Future<T>
invokeAll(tasks)  — все задачи, ждать всех
invokeAny(tasks)  — все задачи, вернуть первый результат
```
