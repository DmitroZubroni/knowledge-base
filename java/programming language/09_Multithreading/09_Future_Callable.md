> **Теги:** #java #concurrency #future #callable #completablefuture #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Concurrency_Index]] | [[08_ExecutorService]]

# 09 — Future, Callable, CompletableFuture

---

## 🔹 Проблема: Runnable не возвращает результат

`Runnable` — это задача, которая просто выполняется и ничего не возвращает. Но часто нам нужно **получить результат** асинхронной операции: запросить данные из базы, загрузить файл, выполнить вычисление. Для этого нужны `Callable` и `Future`.

---

## 🔹 Callable — задача с результатом

`Callable<T>` — функциональный интерфейс, как `Runnable`, но с возвращаемым значением и возможностью бросить исключение:

```java
import java.util.concurrent.*;

// Runnable — без результата, без исключений
Runnable r = () -> System.out.println("done");

// Callable — с результатом, может бросить checked exception
Callable<Integer> c = () -> {
    Thread.sleep(1000); // может бросить InterruptedException
    return 42;
};
```

---

## 🔹 Future — "обещание" результата

`Future<T>` — это **обёртка над результатом**, который ещё не готов. Ты запускаешь задачу, сразу получаешь `Future`, продолжаешь делать другие дела, а потом забираешь результат когда нужно.

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

// submit() сразу возвращает Future — задача уже выполняется в фоне
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(2000);
    return 42;
});

// Пока задача выполняется — делаем другие дела
System.out.println("Задача запущена, делаю другое...");
doSomethingElse();

// Теперь получаем результат (блокирует если ещё не готов)
Integer result = future.get(); // ждёт максимум... сколько нужно
System.out.println("Результат: " + result); // 42

executor.shutdown();
```

### Методы Future

```java
future.get();                              // получить результат, блокирует до готовности
future.get(5, TimeUnit.SECONDS);           // с таймаутом — бросит TimeoutException
future.isDone();                           // готов ли результат (не блокирует)
future.isCancelled();                      // была ли задача отменена
future.cancel(true);                       // отменить задачу (true = interrupt если выполняется)
```

> [!warning] future.get() — опасные исключения
> `get()` бросает:
> - `InterruptedException` — текущий поток был прерван во время ожидания
> - `ExecutionException` — задача бросила исключение внутри (оборачивается)
> - `TimeoutException` — при использовании `get(timeout, unit)` если не успел
> - `CancellationException` — если задача была отменена
>
> Всегда используй `get(timeout, unit)` — бесконечное ожидание редко является правильным решением.

```java
try {
    Integer result = future.get(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true); // отменяем если слишком долго
    throw new RuntimeException("Задача не завершилась за 5 секунд");
} catch (ExecutionException e) {
    throw new RuntimeException("Ошибка в задаче: " + e.getCause());
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new RuntimeException("Прервано ожидание");
}
```

---

## 🔹 Ограничения Future — почему появился CompletableFuture

`Future` имеет серьёзные ограничения:
- `get()` **блокирует** поток — тратим поток впустую на ожидание
- Нельзя **объединить** несколько Future ("дождись обоих и объедини результат")
- Нельзя **цепочкой** обрабатывать результат ("когда готово — сразу трансформируй и отправь дальше")
- Нельзя указать **callback** ("когда готово — вызови вот этот код")
- Нельзя **вручную завершить** Future с каким-то значением

```java
// Хотим: запросить пользователя И его заказы параллельно, потом объединить
Future<User> userFuture = executor.submit(() -> getUser(id));
Future<List<Order>> ordersFuture = executor.submit(() -> getOrders(id));

// Проблема: get() блокирует — сначала ждём user, потом orders
User user = userFuture.get();        // ждём
List<Order> orders = ordersFuture.get(); // снова ждём
// Итого ждём max(time1, time2), но потоки заняты ожиданием
```

`CompletableFuture` решает все эти проблемы.

---

## 🔹 CompletableFuture — асинхронность без блокировок

`CompletableFuture<T>` — это `Future` на стероидах. Позволяет строить **цепочки асинхронных операций**, объединять результаты, обрабатывать ошибки — всё без блокировки потоков.

### Создание

```java
// Запустить задачу асинхронно (использует ForkJoinPool.commonPool() по умолчанию)
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(1000);
    return 42;
});

// С явным executor (рекомендуется в продакшене)
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(
    () -> compute(),
    executor
);

// Задача без результата
CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> doWork());

// Уже завершённый Future (полезно в тестах и при уже известном результате)
CompletableFuture<Integer> ready = CompletableFuture.completedFuture(42);
```

### Цепочки: thenApply, thenAccept, thenRun

```java
CompletableFuture.supplyAsync(() -> "hello")
    .thenApply(s -> s.toUpperCase())     // трансформация результата String → String
    .thenApply(s -> s + "!")             // ещё трансформация
    .thenAccept(s -> System.out.println(s)) // потребить результат, вернуть Void
    .thenRun(() -> System.out.println("Done")); // выполнить после, без доступа к результату
```

Суффикс `Async` у методов запускает следующий шаг **в другом потоке** пула:

```java
cf.thenApply(s -> transform(s))     // в том же потоке что завершил cf
  .thenApplyAsync(s -> transform(s)) // в потоке из ForkJoinPool.commonPool()
  .thenApplyAsync(s -> transform(s), executor) // в потоке из конкретного executor
```

### Объединение нескольких Future

```java
CompletableFuture<User> userCf    = CompletableFuture.supplyAsync(() -> getUser(id));
CompletableFuture<List<Order>> ordersCf = CompletableFuture.supplyAsync(() -> getOrders(id));

// thenCombine — когда ОБА готовы, объединить результаты
CompletableFuture<UserWithOrders> combined = userCf.thenCombine(
    ordersCf,
    (user, orders) -> new UserWithOrders(user, orders)
);

UserWithOrders result = combined.get();
```

```java
// allOf — дождаться ВСЕХ
CompletableFuture<Void> allDone = CompletableFuture.allOf(cf1, cf2, cf3);
allDone.join(); // join() = get() но без checked exceptions

// anyOf — дождаться ПЕРВОГО завершившегося
CompletableFuture<Object> firstDone = CompletableFuture.anyOf(cf1, cf2, cf3);
Object firstResult = firstDone.join();
```

### thenCompose — плоская цепочка (flatMap для Future)

Если следующий шаг сам возвращает `CompletableFuture` — используй `thenCompose`, иначе получишь `CompletableFuture<CompletableFuture<T>>`:

```java
// ❌ thenApply с асинхронным методом — вложенный Future
CompletableFuture<CompletableFuture<Orders>> nested =
    getUser(id).thenApply(user -> getOrders(user.id)); // getOrders возвращает CF

// ✅ thenCompose — "плоский" результат
CompletableFuture<Orders> flat =
    getUser(id).thenCompose(user -> getOrders(user.id));
```

### Обработка ошибок

```java
CompletableFuture.supplyAsync(() -> riskyOperation())
    .exceptionally(ex -> {
        System.err.println("Ошибка: " + ex.getMessage());
        return defaultValue; // возвращаем значение по умолчанию
    })
    .thenAccept(result -> process(result));

// handle — обрабатывает и результат, и ошибку в одном месте
cf.handle((result, ex) -> {
    if (ex != null) {
        log.error("Failed", ex);
        return fallback;
    }
    return result;
});

// whenComplete — выполнить в любом случае (как finally)
cf.whenComplete((result, ex) -> {
    if (ex != null) log.error("Error", ex);
    else log.info("Success: " + result);
    cleanup(); // всегда
});
```

### Реальный пример — параллельные запросы с объединением

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

public CompletableFuture<DashboardData> buildDashboard(long userId) {
    // Запускаем все три запроса параллельно
    CompletableFuture<User> userCf =
        CompletableFuture.supplyAsync(() -> userService.getUser(userId), executor);

    CompletableFuture<List<Order>> ordersCf =
        CompletableFuture.supplyAsync(() -> orderService.getOrders(userId), executor);

    CompletableFuture<List<Notification>> notifCf =
        CompletableFuture.supplyAsync(() -> notifService.getNotifications(userId), executor);

    // Объединяем когда все три готовы
    return CompletableFuture.allOf(userCf, ordersCf, notifCf)
        .thenApply(v -> new DashboardData(
            userCf.join(),    // join() безопасен здесь — allOf гарантировал завершение
            ordersCf.join(),
            notifCf.join()
        ))
        .exceptionally(ex -> {
            log.error("Dashboard build failed", ex);
            return DashboardData.empty();
        });
}
// Общее время ≈ max(time_user, time_orders, time_notif)
// Без параллелизма было бы: time_user + time_orders + time_notif
```

---

## 🔹 get() vs join()

```java
future.get()   // бросает checked: InterruptedException, ExecutionException
future.join()  // бросает unchecked: CompletionException
```

`join()` удобнее в лямбдах и стримах, где checked exceptions неудобны.

---

## 🔹 Итог

```
Callable<T>  — задача с результатом (аналог Runnable но возвращает T)
Future<T>    — "обещание" результата от executor.submit(callable)
  get()      — блокирует до готовности, ВСЕГДА используй с таймаутом
  cancel()   — отменить задачу

Ограничения Future: нет цепочек, нет объединения, нет callbacks, get() блокирует

CompletableFuture<T> — полноценная асинхронность:
  Создание:
    supplyAsync(supplier)      — асинхронная задача с результатом
    runAsync(runnable)         — асинхронная задача без результата
    completedFuture(value)     — уже готовый результат

  Цепочки (каждый метод возвращает новый CF):
    thenApply(fn)      — трансформировать результат (T → U)
    thenAccept(fn)     — потребить результат (T → void)
    thenRun(fn)        — выполнить после (без доступа к результату)
    thenCompose(fn)    — "flatMap", если fn сама возвращает CF

  Объединение:
    allOf(cf1, cf2...) — дождаться всех
    anyOf(cf1, cf2...) — дождаться первого
    thenCombine(other, fn) — объединить два результата

  Ошибки:
    exceptionally(fn)  — восстановление при ошибке
    handle(fn)         — и результат, и ошибка в одном месте
    whenComplete(fn)   — как finally, не меняет результат

get() → checked exceptions
join() → unchecked, удобнее в лямбдах

В продакшене: всегда передавай явный executor в supplyAsync/thenApplyAsync
```
