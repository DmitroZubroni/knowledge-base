> **Теги:** #java #concurrency #reentrantlock #readwritelock #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Concurrency_Index]] | [[04_Synchronized]]

# 11 — Явные блокировки: ReentrantLock, ReadWriteLock, StampedLock

---

## 🔹 Зачем нужны явные блокировки если есть synchronized

`synchronized` — простой и надёжный инструмент, но у него есть ограничения:
- Нельзя прервать ожидание блокировки
- Нельзя попробовать захватить с таймаутом и отступить
- Нет разделения читателей и писателей — все ждут в одной очереди
- Нет возможности честной очереди (fair ordering)

Пакет `java.util.concurrent.locks` решает эти ограничения через явные объекты-блокировки.

---

## 🔹 ReentrantLock — гибкая замена synchronized

`ReentrantLock` работает как `synchronized`, но с дополнительными возможностями. Реентерабелен — поток, держащий лок, может взять его повторно.

### Базовое использование

```java
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();       // захватить блокировку
        try {
            count++;
        } finally {
            lock.unlock(); // ВСЕГДА в finally — даже если бросилось исключение
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

> [!warning] unlock() ВСЕГДА в блоке finally
> Если `unlock()` не будет вызван из-за исключения — блокировка останется захваченной навсегда, и все остальные потоки зависнут. Это самая частая ошибка при работе с ReentrantLock.

### tryLock() — попытка с таймаутом

Главное преимущество перед `synchronized` — возможность **не ждать вечно**:

```java
// Попытка захватить немедленно (без ожидания)
if (lock.tryLock()) {
    try {
        // успешно захватили
    } finally {
        lock.unlock();
    }
} else {
    // лок занят — делаем что-то другое
}

// Попытка с таймаутом
try {
    if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
        try {
            // захватили за 500мс
        } finally {
            lock.unlock();
        }
    } else {
        System.out.println("Не удалось захватить за 500мс");
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### lockInterruptibly() — прерываемое ожидание

```java
try {
    lock.lockInterruptibly(); // ждёт лок, но может быть прерван через interrupt()
    try {
        // работаем
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    // нас прервали пока ждали лок — обрабатываем
    Thread.currentThread().interrupt();
}
```

Обычный `lock.lock()` **игнорирует** прерывания во время ожидания. `lockInterruptibly()` реагирует на `interrupt()` — это важно для корректной отмены задач.

### Fair mode — честная очередь

```java
ReentrantLock fairLock = new ReentrantLock(true); // fair = true
```

В обычном режиме JVM не гарантирует порядок — поток может "перепрыгнуть" очередь. Fair mode гарантирует FIFO: потоки получают лок **строго в порядке ожидания**. Полезно для предотвращения starvation, но снижает пропускную способность.

### Condition — замена wait/notify

С `synchronized` используют `Object.wait()` и `notify()`. С `ReentrantLock` — `Condition`:

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition(); // можно создать несколько условий
Condition notEmpty = lock.newCondition();

class BoundedBuffer<T> {
    private final Queue<T> buffer = new ArrayDeque<>();
    private final int capacity;

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (buffer.size() == capacity) {
                notFull.await();     // ждём, пока место не освободится
            }
            buffer.add(item);
            notEmpty.signal();       // будим того, кто ждёт элемент
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (buffer.isEmpty()) {
                notEmpty.await();    // ждём, пока появится элемент
            }
            T item = buffer.poll();
            notFull.signal();        // будим того, кто ждёт место
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

> [!tip] Condition vs Object.wait/notify
> `Condition` позволяет создавать **несколько условий ожидания** для одного лока. `Object.wait/notify` привязаны к монитору — одна точка ожидания на всех. Это позволяет будить **конкретную группу** ждущих потоков, а не всех сразу (`notifyAll()`).

---

## 🔹 ReentrantReadWriteLock — раздельные блокировки чтения и записи

Классическая проблема: данные читаются часто, пишутся редко. С обычным `synchronized` или `ReentrantLock` — читатели блокируют друг друга, хотя это совершенно не нужно (несколько потоков могут читать одновременно без риска).

`ReentrantReadWriteLock` решает это:
- **ReadLock** — может держать **несколько потоков одновременно**, пока нет писателя
- **WriteLock** — эксклюзивный: ждёт освобождения всех читателей, блокирует новых

```
Обычный Lock:                    ReadWriteLock:
Читатель1: ══════                Читатель1: ══════
Читатель2:   ══════   ← ждёт    Читатель2: ══════   ← параллельно!
Читатель3:      ══════           Читатель3: ══════   ← параллельно!
Писатель:          ═══           Писатель:        ═══  ← ждёт всех читателей
```

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

class Cache {
    private final Map<String, Object> data = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final ReentrantReadWriteLock.ReadLock  readLock  = rwLock.readLock();
    private final ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

    public Object get(String key) {
        readLock.lock();   // несколько потоков могут читать одновременно
        try {
            return data.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(String key, Object value) {
        writeLock.lock();  // эксклюзивный доступ, все читатели и писатели ждут
        try {
            data.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

### Lock downgrading — понижение с write на read

```java
writeLock.lock();
try {
    updateData();       // изменяем данные
    readLock.lock();    // захватываем read lock не отпуская write
} finally {
    writeLock.unlock(); // отпускаем write lock — теперь держим только read
}
try {
    return readData();  // читаем без риска, что кто-то изменит
} finally {
    readLock.unlock();
}
// Lock downgrading: write → read. Обратное (read → write) — НЕ поддерживается!
```

---

## 🔹 StampedLock — оптимистичное чтение (Java 8+)

`StampedLock` — следующий уровень после `ReadWriteLock`. Добавляет режим **оптимистичного чтения**: читаем **без захвата блокировки вообще**, потом проверяем — не изменились ли данные пока мы читали. Если да — повторяем с настоящей блокировкой.

```java
import java.util.concurrent.locks.StampedLock;

class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // Запись — обычный write lock
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    // Чтение — оптимистичное (без блокировки!)
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead(); // получаем "штамп" — не блокируем
        double curX = x, curY = y;             // читаем данные

        if (!lock.validate(stamp)) {           // проверяем: не было ли записи?
            // Данные изменились — повторяем с настоящей read блокировкой
            stamp = lock.readLock();
            try {
                curX = x;
                curY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(curX * curX + curY * curY);
    }
}
```

```
Поведение оптимистичного чтения:
Нет конкуренции (типичный случай):
  tryOptimisticRead() → читаем → validate() → true → возвращаем результат
  Стоимость: почти нулевая (просто проверка версии)

Есть конкуренция (редкий случай):
  tryOptimisticRead() → читаем → [кто-то записал] → validate() → false
  → readLock() → перечитываем → unlockRead()
  Стоимость: как обычный read lock
```

> [!warning] StampedLock не реентерабелен
> В отличие от `ReentrantLock` и `ReentrantReadWriteLock`, `StampedLock` **не реентерабелен**. Повторный захват того же лока в том же потоке вызовет deadlock.

---

## 🔹 Сравнение механизмов блокировки

| | synchronized | ReentrantLock | ReadWriteLock | StampedLock |
|-|-------------|---------------|---------------|-------------|
| Простота | ✅ Просто | Средне | Сложнее | Сложно |
| tryLock с таймаутом | ❌ | ✅ | ✅ | ✅ |
| Прерываемое ожидание | ❌ | ✅ | ✅ | ✅ |
| Fair mode | ❌ | ✅ | ✅ | ❌ |
| Параллельное чтение | ❌ | ❌ | ✅ | ✅ |
| Оптимистичное чтение | ❌ | ❌ | ❌ | ✅ |
| Реентерабельность | ✅ | ✅ | ✅ | ❌ |
| Когда использовать | Простые случаи | Нужен tryLock/interrupt | Много читателей | Max производительность |

---

## 🔹 Итог

```
ReentrantLock — гибкий synchronized:
  lock() / unlock() — ВСЕГДА unlock() в finally
  tryLock()                   — без ожидания
  tryLock(timeout, unit)      — с таймаутом (борьба с deadlock)
  lockInterruptibly()         — реагирует на interrupt()
  new ReentrantLock(true)     — fair mode (FIFO)
  lock.newCondition()         — несколько условий ожидания (лучше wait/notify)

ReentrantReadWriteLock — раздельные read/write:
  readLock()  — параллельный, несколько потоков одновременно
  writeLock() — эксклюзивный
  Использовать: часто читаем, редко пишем (кэши, справочники)
  Lock downgrading: write → read поддерживается, обратное — нет

StampedLock — оптимистичное чтение (Java 8+):
  tryOptimisticRead() → читаем → validate() → если false → readLock()
  Максимальная производительность при редких записях
  НЕ реентерабелен — deadlock при повторном захвате в том же потоке

Правило выбора:
  synchronized            — простые случаи (большинство ситуаций)
  ReentrantLock           — нужен tryLock или lockInterruptibly
  ReentrantReadWriteLock  — много читателей, мало писателей
  StampedLock             — максимальная производительность, высокая конкуренция
```
