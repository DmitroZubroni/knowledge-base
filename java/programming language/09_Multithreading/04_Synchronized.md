> **Теги:** #java #concurrency #synchronized #monitor #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Concurrency_Index]] | [[03_Race_Condition]]

# 04 — synchronized и монитор

---

## 🔹 Идея решения

В [[03_Race_Condition]] мы увидели проблему: несколько потоков одновременно выполняют `count++`, и операции "перемешиваются", теряя инкременты.

Решение очевидно: нужно сделать так, чтобы **только один поток за раз** мог выполнять критическую секцию кода (тот участок, что работает с общими данными). Все остальные потоки должны **подождать своей очереди**.

```
Без synchronized:           С synchronized:

Поток A: READ-ADD-WRITE     Поток A: [READ-ADD-WRITE]  ← никто не вмешается
Поток B:    READ-ADD-WRITE  Поток B:                   [READ-ADD-WRITE] ← ждёт A
       ↑ переплелись, потеря данных      ↑ по очереди, всё корректно
```

`synchronized` — это встроенный в язык механизм для создания таких "по очереди" участков кода.

---

## 🔹 Монитор (Monitor) — что это такое

Каждый объект в Java имеет связанный с ним **монитор** — это невидимый встроенный замок (lock). Когда поток входит в `synchronized` блок — он **захватывает** монитор объекта. Любой другой поток, пытающийся захватить тот же монитор, переходит в состояние `BLOCKED` и ждёт, пока монитор не освободится.

```
Object obj = new Object();
                                 ┌─────────────────┐
synchronized (obj) {             │   Монитор obj    │
    // критическая секция       │  [захвачен: A]   │
}                                └─────────────────┘

Поток A: захватил монитор obj, выполняет код
Поток B: пытается захватить obj → BLOCKED, ждёт
Поток A: вышел из блока → монитор освобождён
Поток B: захватывает монитор, выполняет код
```

> [!note] Главное свойство монитора
> Монитор может быть захвачен **только одним потоком одновременно**. Это и обеспечивает "по очереди".

---

## 🔹 synchronized блок

Самый явный способ — указать конкретный объект, который служит "замком":

```java
class Counter {
    private int count = 0;
    private final Object lock = new Object(); // отдельный объект-замок

    public void increment() {
        synchronized (lock) {
            count++; // только один поток здесь одновременно
        }
    }

    public int getCount() {
        synchronized (lock) {
            return count;
        }
    }
}
```

Теперь повторим эксперимент из [[03_Race_Condition]] — два потока по 1000 инкрементов:

```java
Counter counter = new Counter();
Runnable task = () -> {
    for (int i = 0; i < 1000; i++) counter.increment();
};

Thread t1 = new Thread(task);
Thread t2 = new Thread(task);
t1.start(); t2.start();
t1.join(); t2.join();

System.out.println(counter.getCount()); // всегда 2000, гарантированно
```

> [!warning] Зачем отдельный объект `lock`, а не `this`?
> Можно использовать `synchronized(this)`, но это плохая практика: любой внешний код, имеющий ссылку на твой объект, тоже может сделать `synchronized(myObject)` и непреднамеренно заблокировать твою внутреннюю логику. Приватный финальный объект-замок инкапсулирует синхронизацию полностью.

---

## 🔹 synchronized метод

Если весь метод — критическая секция, можно поставить модификатор прямо на метод:

```java
class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

Это **полностью эквивалентно**:

```java
public void increment() {
    synchronized (this) { // монитор — сам объект Counter
        count++;
    }
}
```

То есть `synchronized` метод использует **сам объект** (`this`) как монитор.

---

## 🔹 static synchronized — монитор класса

Если метод статический, у `this` нет смысла — статические методы не привязаны к конкретному объекту. Вместо этого используется монитор **класса**:

```java
class Counter {
    private static int globalCount = 0;

    public static synchronized void increment() {
        globalCount++; // монитор — Counter.class
    }
}
```

Эквивалентно:

```java
public static void increment() {
    synchronized (Counter.class) { // монитор — объект Class<Counter>
        globalCount++;
    }
}
```

> [!warning] Монитор экземпляра и монитор класса — разные замки!
> ```java
> class Example {
>     public synchronized void instanceMethod() { }       // монитор: this
>     public static synchronized void staticMethod() { }  // монитор: Example.class
> }
> ```
> Поток в `instanceMethod()` **не блокирует** другой поток, выполняющий `staticMethod()` — это два разных монитора. Частая ошибка — думать, что `static synchronized` защищает и обычные методы.

---

## 🔹 Реентерабельность (Reentrancy)

Java мониторы **реентерабельны** — поток, который уже держит монитор, может захватить его повторно без блокировки самого себя.

```java
class Example {
    public synchronized void outer() {
        System.out.println("outer");
        inner(); // вызов другого synchronized метода с ТЕМ ЖЕ монитором
    }

    public synchronized void inner() {
        System.out.println("inner"); // не блокируется! тот же поток уже держит монитор
    }
}
```

Без реентерабельности `outer()` заблокировал бы сам себя при вызове `inner()` — это был бы **deadlock с самим собой**. JVM ведёт счётчик "сколько раз текущий поток захватил этот монитор" и освобождает его только когда счётчик достигает нуля.

---

## 🔹 Гранулярность блокировки — баланс производительности

Чем **больше** код внутри `synchronized` — тем дольше другие потоки ждут (хуже производительность). Чем **меньше** — тем выше риск упустить часть логики из защиты (хуже корректность).

```java
class BankAccount {
    private double balance;
    private List<String> transactionLog = new ArrayList<>();

    // ❌ Слишком грубо — блокируем даже логирование, которое не требует синхронизации
    public synchronized void withdraw(double amount) {
        balance -= amount;
        transactionLog.add("Withdraw: " + amount); // это можно делать вне лока
        sendNotification(); // тем более это — может занимать секунды (сеть)!
    }

    // ✅ Точнее — блокируем только то, что действительно требует атомарности
    public void withdrawBetter(double amount) {
        synchronized (this) {
            balance -= amount;
        }
        transactionLog.add("Withdraw: " + amount); // вне лока — параллельно
        sendNotification(); // вне лока — не держит других в очереди
    }
}
```

> [!tip] Правило
> В `synchronized` должно быть **минимально необходимое** количество кода — только то, что напрямую читает/изменяет общие данные. Сетевые вызовы, логирование, вычисления без общих данных — выноси наружу.

---

## 🔹 Deadlock через synchronized — краткое предупреждение

Если внутри одного `synchronized(A)` происходит попытка захватить `synchronized(B)`, а в это же время другой поток держит `B` и пытается захватить `A` — оба потока зависнут навечно. Это **deadlock**, разбирается подробно в [[07_Deadlock]].

```java
// Поток 1: synchronized(A) { synchronized(B) { ... } }
// Поток 2: synchronized(B) { synchronized(A) { ... } }
// ⚠️ Возможен deadlock — оба ждут друг друга вечно
```

---

## 🔹 Стоимость synchronized

`synchronized` — не бесплатен. Захват и освобождение монитора требуют системных вызовов (в худшем случае) и могут включать переключение контекста потока. JVM оптимизирует "лёгкие" случаи (biased locking, lock elision для объектов, которые не escape из потока), но в целом:

- Если критическая секция не нужна (данные не общие) — не используй `synchronized`
- Для простых счётчиков рассмотри `Atomic*` классы — они быстрее благодаря CAS (см. [[06_Atomic]])
- Для сложных сценариев с разделением чтения/записи — `ReentrantReadWriteLock` (см. [[11_Locks]])

---

## 🔹 Итог

```
synchronized = захват монитора объекта на время выполнения блока
              только ОДИН поток может держать монитор одновременно
              остальные → BLOCKED

Три формы:
  synchronized (obj) { ... }     — монитор: явно указанный объект
  synchronized void method()     — монитор: this (объект)
  static synchronized void m()   — монитор: SomeClass.class

Реентерабельность: поток может повторно захватить СВОЙ монитор без deadlock

Гранулярность: минимизируй то, что внутри synchronized
  - сетевые вызовы, I/O, долгие вычисления — НЕ внутри
  - только чтение/запись общих данных

Используй приватный final Object как явный замок,
а не synchronized(this) — для инкапсуляции

Альтернативы:
  Atomic* — для простых счётчиков, быстрее (CAS)
  ReentrantLock / ReadWriteLock — для сложных сценариев (см. [[11_Locks]])
```
