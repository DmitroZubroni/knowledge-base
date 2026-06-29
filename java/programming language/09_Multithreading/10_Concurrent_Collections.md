> **Теги:** #java #concurrency #concurrenthashmap #blockingqueue #copyonwrite #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Concurrency_Index]] | [[04_Synchronized]]

# 10 — Потокобезопасные коллекции

---

## 🔹 Почему обычные коллекции опасны в многопоточке

Стандартные `ArrayList`, `HashMap`, `HashSet` — **не потокобезопасны**. При одновременном доступе из нескольких потоков возникают гонки данных.

```java
Map<String, Integer> map = new HashMap<>();

// Два потока одновременно делают put() в один бакет:
// → повреждение внутренней структуры бакета
// → ConcurrentModificationException при итерации
// → потеря данных
// → в редких случаях даже бесконечный цикл (Java 7 и ниже при resize)
```

Первое что приходит в голову — обернуть в `Collections.synchronizedMap()`. Но это грубое решение: **каждый метод синхронизирован целиком** через общий lock, что создаёт огромное количество блокировок и плохо масштабируется при высокой конкуренции.

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// Все операции сериализованы через один lock — узкое горлышко при высокой нагрузке
```

Пакет `java.util.concurrent` предоставляет более умные решения.

---

## 🔹 ConcurrentHashMap — умная сегментная блокировка

`ConcurrentHashMap` — потокобезопасная `HashMap` с высокой производительностью. Ключевое отличие от `synchronizedMap`: **блокирует не всю карту, а только отдельные бакеты**.

```
synchronizedMap:     ConcurrentHashMap:
┌──────────────┐     ┌─────┬─────┬─────┬─────┐
│  ОДИН LOCK   │     │lock │lock │lock │lock │  ← каждый бакет свой lock
│  на всю карту│     │  ↓  │  ↓  │  ↓  │  ↓  │
│              │     │ B0  │ B1  │ B2  │ B3  │
└──────────────┘     └─────┴─────┴─────┴─────┘

Поток A блокирует ALL    Поток A блокирует только B0
Поток B ждёт             Поток B работает с B2 параллельно
```

На практике: в Java 8+ это реализовано через CAS и синхронизацию на уровне **одного узла бакета** — конкуренция возникает только при записи в **один и тот же бакет**.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("key", 1);
map.get("key");
map.remove("key");

// Атомарные составные операции — без дополнительной синхронизации
map.putIfAbsent("key", 1);                        // положить только если нет
map.computeIfAbsent("key", k -> expensiveInit(k)); // вычислить значение если нет ключа
map.computeIfPresent("key", (k, v) -> v + 1);      // обновить если ключ есть
map.compute("key", (k, v) -> v == null ? 1 : v + 1); // безопасный инкремент
map.merge("key", 1, Integer::sum);                  // сложить с существующим

// Атомарный подсчёт частоты — классический паттерн
words.forEach(word -> map.merge(word, 1, Integer::sum));
```

> [!warning] Итерация не бросает ConcurrentModificationException
> В отличие от `HashMap`, `ConcurrentHashMap` не бросает `ConcurrentModificationException` при изменении во время итерации. Но итератор отражает состояние на **момент создания** — изменения после могут быть видны, а могут и нет (weakly consistent итераторы).

> [!warning] Null не допускается
> `ConcurrentHashMap` не разрешает `null` ключи и `null` значения. Если `get()` вернул `null` — ты не можешь понять: ключа нет, или значение равно `null`? В многопоточном контексте такая неопределённость опасна.

---

## 🔹 CopyOnWriteArrayList — для коллекций с редкими записями

`CopyOnWriteArrayList` — потокобезопасный список, где **любое изменение создаёт новую копию** всего массива. Чтения при этом вообще не блокируются.

```
Исходный массив: [A, B, C]

Поток-читатель:  читает [A, B, C] — никаких блокировок
Поток-писатель:  add("D") → создаёт копию [A, B, C, D], заменяет ссылку атомарно
                 Пока копия создаётся — читатели всё ещё работают со старым [A, B, C]
```

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("A");
list.add("B");

// Чтение — очень быстро, без блокировок
String s = list.get(0);

// Запись — дорого: O(n) копирование всего массива
list.add("C");
list.remove("A");

// Итерация БЕЗОПАСНА во время изменений — итерируется по снимку
for (String item : list) {
    list.remove(item); // не бросит ConcurrentModificationException — работает со снимком
}
```

> [!note] Когда использовать CopyOnWriteArrayList
> Только когда **чтений значительно больше, чем записей**:
> - Список подписчиков на события (редко добавляются, часто обходятся)
> - Кэш конфигурации (редко обновляется, часто читается)
> - Список слушателей (listeners) в паттерне Observer
>
> При частых записях — катастрофическая производительность из-за постоянного копирования.

Аналогично существует `CopyOnWriteArraySet`.

---

## 🔹 BlockingQueue — очередь с блокировкой

`BlockingQueue` — это очередь, которая умеет **блокировать поток** при попытке взять элемент из пустой очереди (ждать пока появится) или добавить в полную (ждать места). Это идеальный механизм для паттерна **Producer-Consumer**.

```
Producer поток:   [создаёт задачи] ──put()──► QUEUE ──take()──► [обрабатывает] :Consumer поток
                                    блокирует если          блокирует если
                                    очередь полна           очередь пуста
```

### Методы BlockingQueue

| | Бросает исключение | Возвращает false/null | Блокирует | С таймаутом |
|-|-------------------|----------------------|-----------|-------------|
| Добавить | `add(e)` | `offer(e)` | `put(e)` | `offer(e, t, unit)` |
| Извлечь | `remove()` | `poll()` | `take()` | `poll(t, unit)` |
| Посмотреть | `element()` | `peek()` | — | — |

`put()` и `take()` — главные методы. Они **блокируют поток** вместо того чтобы бросать исключение или возвращать null. Именно это делает BlockingQueue идеальным для Producer-Consumer.

### Реализации BlockingQueue

```java
// LinkedBlockingQueue — опционально ограниченная, на основе LinkedList
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100); // ёмкость 100
// Без параметра — unbounded (риск OOM)

// ArrayBlockingQueue — строго ограниченная, на основе массива
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
// Более предсказуемая память (массив фиксированного размера)

// PriorityBlockingQueue — с приоритетом (на основе кучи)
BlockingQueue<Task> queue = new PriorityBlockingQueue<>();
// Порядок по Comparable или Comparator — take() возвращает наименьший

// SynchronousQueue — без буфера вообще
BlockingQueue<Task> queue = new SynchronousQueue<>();
// put() блокирует пока кто-то не вызовет take() — прямая передача между потоками
// Используется в Executors.newCachedThreadPool()

// DelayQueue — элементы извлекаются только после задержки
// Полезен для отложенных задач, планировщиков
```

### Паттерн Producer-Consumer

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(50);

// Producer — производит данные
Runnable producer = () -> {
    try {
        for (int i = 0; i < 100; i++) {
            String task = "task-" + i;
            queue.put(task);         // блокирует если очередь полна
            System.out.println("Произвёл: " + task);
        }
        queue.put("POISON_PILL");    // сигнал завершения для consumer
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
};

// Consumer — потребляет данные
Runnable consumer = () -> {
    try {
        while (true) {
            String task = queue.take(); // блокирует если очередь пуста
            if ("POISON_PILL".equals(task)) break;
            System.out.println("Обработал: " + task);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
};

new Thread(producer).start();
new Thread(consumer).start();
```

> [!tip] Poison Pill — элегантный сигнал завершения
> "Отравленная таблетка" (Poison Pill) — специальный sentinel-элемент в очереди, который сигнализирует consumer'у о завершении работы. Чище, чем volatile флаг или interrupt.

---

## 🔹 ConcurrentLinkedQueue — lock-free очередь

Если блокировка не нужна (producer не должен ждать при полной очереди — очередь неограниченная) — используй `ConcurrentLinkedQueue`. Основана на CAS, без блокировок:

```java
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
queue.offer("task");      // никогда не блокирует
String task = queue.poll(); // возвращает null если пусто, не блокирует
queue.peek();               // посмотреть без удаления
queue.size();               // O(n)! — не кэшируется, считает обходом
```

---

## 🔹 Быстрое сравнение

| Коллекция | Внутренний механизм | Чтение | Запись | Когда |
|-----------|---------------------|--------|--------|-------|
| `synchronizedMap` | Один lock на всё | Медленно | Медленно | Никогда (есть лучше) |
| `ConcurrentHashMap` | CAS + lock на бакет | Быстро | Быстро | Почти всегда |
| `CopyOnWriteArrayList` | Копирование массива | Очень быстро | Очень медленно | Чтений >> Записей |
| `ArrayBlockingQueue` | Lock + условие | — | — | Producer-Consumer |
| `LinkedBlockingQueue` | Два lock (head/tail) | — | — | Очередь задач |
| `ConcurrentLinkedQueue` | CAS (lock-free) | Быстро | Быстро | Очередь без блокировок |

---

## 🔹 Итог

```
Обычные коллекции — не потокобезопасны
synchronizedMap — безопасны, но медленны (один lock)

java.util.concurrent — умные решения:

ConcurrentHashMap:
  - блокирует только отдельные бакеты (или CAS)
  - null запрещён
  - compute/merge/putIfAbsent — атомарные составные операции
  - использовать вместо synchronizedMap почти всегда

CopyOnWriteArrayList:
  - запись = полное копирование массива (дорого!)
  - чтение без блокировок (быстро)
  - только когда чтений >> записей (listeners, конфиги)

BlockingQueue (ArrayBlockingQueue, LinkedBlockingQueue):
  - put() блокирует при полной очереди
  - take() блокирует при пустой
  - идеален для Producer-Consumer
  - Poison Pill — элегантный сигнал завершения

ConcurrentLinkedQueue:
  - lock-free на CAS
  - неограниченная, offer() никогда не блокирует
  - size() — O(n), не вызывай часто
```
