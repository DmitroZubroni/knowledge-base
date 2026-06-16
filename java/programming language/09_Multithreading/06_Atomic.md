> **Теги:** #java #concurrency #atomic #cas #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Concurrency_Index]] | [[04_Synchronized]] | [[05_Volatile]]

# 06 — Atomic классы и CAS

---

## 🔹 Проблема: synchronized для счётчика — оверхед

Из [[04_Synchronized]] мы знаем, как сделать `count++` потокобезопасным:

```java
class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }
}
```

Это работает, но `synchronized` — относительно дорогая операция: захват монитора, потенциальная блокировка потока, переключение контекста. Для простого счётчика, который инкрементируется миллионы раз в секунду в высоконагруженном приложении, это может стать узким местом.

Java предоставляет более быструю альтернативу — пакет `java.util.concurrent.atomic`, основанный на аппаратной инструкции **Compare-And-Swap (CAS)**.

---

## 🔹 Compare-And-Swap (CAS) — как это работает

CAS — это **атомарная аппаратная инструкция** процессора (не программная конструкция). Она выполняет три действия как одну неделимую операцию:

```
CAS(memoryLocation, expectedValue, newValue):
  1. Прочитать текущее значение по адресу memoryLocation
  2. Если оно равно expectedValue → записать newValue, вернуть true
  3. Если не равно → ничего не менять, вернуть false
```

Всё это происходит **атомарно на уровне процессора** — никакой другой поток не может "влезть" между шагами 1 и 2.

```java
// Псевдокод того, что AtomicInteger делает под капотом
public boolean compareAndSet(int expected, int newValue) {
    if (currentValue == expected) {
        currentValue = newValue;
        return true;
    }
    return false;
}
// Но это выполняется как ОДНА аппаратная инструкция, без возможности прерывания
```

### Как CAS решает инкремент без блокировок

```java
public void increment() {
    while (true) {
        int current = getValue();       // 1. прочитать текущее значение
        int next = current + 1;          // 2. вычислить новое
        if (compareAndSet(current, next)) { // 3. если значение не изменилось — записать
            break; // успех, выходим
        }
        // если другой поток успел изменить значение между шагами 1 и 3 —
        // compareAndSet вернёт false, повторяем попытку (retry)
    }
}
```

```
Поток A                              Поток B
─────────────────────────────────────────────────────
current = 5
next = 6
                                      current = 5
                                      next = 6
CAS(5, 6) → успех, value=6
                                      CAS(5, 6) → FALSE! value уже 6, не 5
                                      retry:
                                      current = 6
                                      next = 7
                                      CAS(6, 7) → успех, value=7

Итог: value=7, корректно! Без единого synchronized или BLOCKED потока.
```

> [!note] Lock-free, но не "бесплатно"
> Это называется **lock-free алгоритм** — никто не блокируется (`BLOCKED`), потоки не ждут друг друга в очереди. Но при высокой конкуренции потоки могут многократно повторять попытку (retry loop) — это называется **spin**. При очень высокой конкуренции CAS может оказаться даже медленнее `synchronized`, но в типичных сценариях он значительно быстрее.

---

## 🔹 AtomicInteger, AtomicLong, AtomicBoolean

Самые часто используемые атомарные типы:

```java
import java.util.concurrent.atomic.AtomicInteger;

AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();   // ++count, возвращает новое значение
counter.getAndIncrement();   // count++, возвращает старое значение
counter.decrementAndGet();   // --count
counter.getAndDecrement();   // count--

counter.addAndGet(5);        // count += 5, возвращает новое значение
counter.getAndAdd(5);        // count += 5, возвращает старое значение

counter.set(10);             // просто установить значение
counter.get();                // прочитать значение

counter.compareAndSet(10, 20); // если текущее==10, установить 20. true/false

// Условное обновление через функцию
counter.updateAndGet(x -> x * 2);            // value = value * 2
counter.getAndUpdate(x -> x * 2);            // возвращает старое, потом обновляет

// Обновление на основе старого значения через функцию (полезно для сложной логики)
counter.accumulateAndGet(5, (current, x) -> current + x);
```

Полный пример многопоточного счётчика:

```java
AtomicInteger counter = new AtomicInteger(0);

Runnable task = () -> {
    for (int i = 0; i < 1000; i++) {
        counter.incrementAndGet();
    }
};

Thread t1 = new Thread(task);
Thread t2 = new Thread(task);
t1.start(); t2.start();
t1.join(); t2.join();

System.out.println(counter.get()); // всегда 2000
```

---

## 🔹 AtomicReference — для объектов

`AtomicReference<T>` позволяет атомарно работать со ссылками на объекты — например, реализовать lock-free стек или обновлять immutable конфигурацию.

```java
import java.util.concurrent.atomic.AtomicReference;

AtomicReference<String> ref = new AtomicReference<>("initial");

ref.set("new value");
ref.get();                                    // "new value"
ref.compareAndSet("new value", "another");    // true, если текущее == "new value"

ref.updateAndGet(s -> s.toUpperCase());       // "ANOTHER"
```

### Пример: lock-free immutable обновление состояния

```java
class Config {
    final String name;
    final int version;

    Config(String name, int version) {
        this.name = name;
        this.version = version;
    }
}

AtomicReference<Config> configRef = new AtomicReference<>(new Config("default", 1));

// Атомарное обновление: увеличить версию, не потеряв конкурентные обновления
public void incrementVersion() {
    Config old, updated;
    do {
        old = configRef.get();
        updated = new Config(old.name, old.version + 1);
    } while (!configRef.compareAndSet(old, updated));
    // если другой поток успел обновить configRef между get() и compareAndSet() —
    // CAS вернёт false, и мы повторим попытку с новым old
}
```

---

## 🔹 LongAdder — для очень высокой конкуренции

`AtomicInteger`/`AtomicLong` отлично работают при умеренной конкуренции. Но при **очень большом** количестве потоков, постоянно соревнующихся за CAS на одной переменной, retry-loop становится узким местом — слишком много потоков пытаются изменить одну и ту же ячейку памяти одновременно.

`LongAdder` решает это, храня **несколько внутренних счётчиков-ячеек** (cells), распределяя нагрузку между потоками — каждый поток обычно пишет в свою ячейку, избегая конкуренции. Финальная сумма вычисляется только при чтении.

```java
import java.util.concurrent.atomic.LongAdder;

LongAdder counter = new LongAdder();

counter.increment();   // ++count
counter.add(5);         // count += 5
counter.decrement();    // --count

long total = counter.sum(); // суммирует все внутренние ячейки — вызывать редко
```

> [!tip] AtomicLong vs LongAdder
> - **AtomicLong** — если нужно **часто читать** текущее значение (например, использовать его в логике сразу после изменения)
> - **LongAdder** — если нужно **часто писать** (инкрементировать) и **редко читать** (например, метрики/статистика, которые читаются раз в минуту для отображения)

---

## 🔹 Сравнение подходов к потокобезопасному счётчику

| | synchronized | AtomicInteger | LongAdder |
|-|---------------|----------------|-----------|
| Механизм | Монитор (блокировка) | CAS (lock-free) | Множество ячеек + CAS |
| Поток может ждать (BLOCKED) | Да | Нет (но может retry) | Нет |
| Производительность при низкой конкуренции | ОК | Лучше | Избыточно |
| Производительность при высокой конкуренции | Плохо (много BLOCKED) | Средне (много retry) | **Лучше всего** |
| Подходит для | Сложная логика, несколько полей | Простой счётчик/флаг | Счётчик с очень частыми инкрементами |

---

## 🔹 Итог

```
CAS (Compare-And-Swap) — атомарная аппаратная инструкция:
  "если значение по адресу == expected, заменить на new, иначе нет"

Atomic* классы — lock-free операции на CAS, без BLOCKED потоков
  AtomicInteger / AtomicLong / AtomicBoolean — простые счётчики и флаги
  AtomicReference<T>                          — атомарная замена ссылок на объекты
  LongAdder                                   — для очень высокой конкуренции

incrementAndGet() vs getAndIncrement() — порядок ++x vs x++ (как с примитивами)

compareAndSet(expected, new) — основа всех lock-free паттернов:
  цикл while: get old → compute new → CAS(old, new), retry если false

Когда что использовать:
  synchronized  — сложная логика, несколько связанных полей
  AtomicInteger — простой счётчик, частое чтение и запись
  LongAdder     — очень высокая конкуренция, редкое чтение (метрики)
  AtomicReference — атомарная замена immutable объектов
```
