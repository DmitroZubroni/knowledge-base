# 06 — Коллекции (Collections)

> **Теги:** #java #programming #collections #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]]

---

## 🔹 Иерархия коллекций

```
Iterable<E>
└── Collection<E>
    ├── List<E>          (упорядочен, допускает дубликаты)
    │   ├── ArrayList
    │   └── LinkedList
    ├── Set<E>           (нет дубликатов)
    │   ├── HashSet          (нет порядка)
    │   ├── LinkedHashSet    (порядок вставки)
    │   └── TreeSet          (отсортирован)
    └── Queue<E>         (очередь FIFO)
        ├── LinkedList
        ├── PriorityQueue
        └── Deque<E>     (двусторонняя очередь)
            ├── ArrayDeque
            └── LinkedList

Map<K,V>                 (ключ-значение, не наследует Collection)
├── HashMap              (нет порядка)
├── LinkedHashMap        (порядок вставки)
└── TreeMap              (отсортирован по ключу)
```

---

## 🔹 List — упорядоченный список

> [!note] Определение
> `List` — коллекция с сохранением порядка элементов, допускает дубликаты. Доступ по индексу.

### ArrayList
```java
import java.util.ArrayList;
import java.util.List;

List<String> names = new ArrayList<>();  // лучше объявлять через интерфейс List

// Добавление
names.add("Alice");
names.add("Bob");
names.add("Charlie");
names.add(1, "Dave");  // вставка по индексу

// Доступ
System.out.println(names.get(0));    // "Alice"
System.out.println(names.size());    // 4
System.out.println(names.isEmpty()); // false

// Проверка
names.contains("Bob");   // true
names.indexOf("Bob");    // 2

// Изменение
names.set(0, "Alicia");  // заменить по индексу
names.remove("Bob");     // удалить по значению
names.remove(0);         // удалить по индексу

// Перебор
for (String name : names) {
    System.out.println(name);
}

// Итерация с индексом
for (int i = 0; i < names.size(); i++) {
    System.out.println(i + ": " + names.get(i));
}

// Создание с начальными данными (Java 9+)
List<String> fixed = List.of("A", "B", "C");  // неизменяемый!

// Изменяемый список из массива
List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));
```

### LinkedList
```java
import java.util.LinkedList;

LinkedList<Integer> ll = new LinkedList<>();
ll.add(1);
ll.add(2);
ll.addFirst(0);     // добавить в начало
ll.addLast(3);      // добавить в конец
ll.getFirst();      // 0
ll.getLast();       // 3
ll.removeFirst();   // удалить из начала
ll.removeLast();    // удалить из конца
```

### ArrayList vs LinkedList

| Операция | ArrayList | LinkedList |
|----------|:---------:|:----------:|
| Доступ по индексу `get(i)` | O(1) ✅ | O(n) ❌ |
| Добавление в конец | O(1) ✅ | O(1) ✅ |
| Добавление в начало/середину | O(n) ❌ | O(1) ✅ |
| Удаление из начала/середины | O(n) ❌ | O(1) ✅ |
| Память | Меньше | Больше (узлы) |
| Когда использовать | Чтение по индексу | Частые вставки/удаления |

---

## 🔹 Set — множество без дубликатов

> [!note] Определение
> `Set` — коллекция без дубликатов. Проверка наличия через `equals()` и `hashCode()`.

### HashSet
```java
import java.util.HashSet;
import java.util.Set;

Set<String> set = new HashSet<>();
set.add("apple");
set.add("banana");
set.add("apple");  // дубликат — не добавится

System.out.println(set.size());        // 2
System.out.println(set.contains("apple")); // true

set.remove("banana");

// Операции над множествами
Set<Integer> a = new HashSet<>(Set.of(1, 2, 3, 4));
Set<Integer> b = new HashSet<>(Set.of(3, 4, 5, 6));

// Пересечение
a.retainAll(b);  // a = {3, 4}

// Объединение
Set<Integer> union = new HashSet<>(a);
union.addAll(b);  // {1, 2, 3, 4, 5, 6}

// Разность
Set<Integer> diff = new HashSet<>(a);
diff.removeAll(b);  // {1, 2}
```

### TreeSet — отсортированный Set
```java
import java.util.TreeSet;

TreeSet<Integer> sorted = new TreeSet<>();
sorted.add(5);
sorted.add(1);
sorted.add(3);
System.out.println(sorted);  // [1, 3, 5] — всегда отсортирован

sorted.first();   // 1
sorted.last();    // 5
sorted.headSet(3); // [1] — элементы < 3
sorted.tailSet(3); // [3, 5] — элементы >= 3
```

### HashSet vs LinkedHashSet vs TreeSet

| | HashSet | LinkedHashSet | TreeSet |
|-|---------|---------------|---------|
| Порядок | Нет | Порядок вставки | Сортировка |
| Скорость add/remove/contains | O(1) | O(1) | O(log n) |
| null | Допускает | Допускает | ❌ (NPE) |
| Когда | Быстрая проверка | Сохранить порядок | Нужна сортировка |

> [!warning] Подводные камни
> Для работы `HashSet` / `HashMap` с кастомными объектами — **обязательно** переопределить `equals()` и `hashCode()`.

---

## 🔹 Map — ключ-значение

> [!note] Определение
> `Map` — хранит пары ключ-значение. Ключи уникальны. Не наследует `Collection`.

### HashMap
```java
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> scores = new HashMap<>();

// Добавление / обновление
scores.put("Alice", 95);
scores.put("Bob", 87);
scores.put("Charlie", 92);
scores.put("Alice", 98);   // перезапишет существующее значение

// Получение
System.out.println(scores.get("Alice"));          // 98
System.out.println(scores.get("Nobody"));         // null — ключ не найден
System.out.println(scores.getOrDefault("Nobody", 0));  // 0 — значение по умолчанию

// Проверка
scores.containsKey("Bob");    // true
scores.containsValue(92);     // true
scores.size();                // 3

// Удаление
scores.remove("Charlie");
scores.remove("Bob", 99);  // удалить только если ключ=Bob и значение=99

// putIfAbsent — добавить только если ключа нет
scores.putIfAbsent("Dave", 75);

// computeIfAbsent — создать значение если ключа нет
Map<String, List<String>> groups = new HashMap<>();
groups.computeIfAbsent("team1", k -> new ArrayList<>()).add("Alice");
groups.computeIfAbsent("team1", k -> new ArrayList<>()).add("Bob");
```

### Итерация по Map
```java
Map<String, Integer> map = new HashMap<>(Map.of("a", 1, "b", 2, "c", 3));

// По Entry (ключ + значение)
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// Только ключи
for (String key : map.keySet()) {
    System.out.println(key);
}

// Только значения
for (int value : map.values()) {
    System.out.println(value);
}

// forEach (Java 8+)
map.forEach((key, value) -> System.out.println(key + ": " + value));
```

### HashMap vs LinkedHashMap vs TreeMap

| | HashMap | LinkedHashMap | TreeMap |
|-|---------|---------------|---------|
| Порядок | Нет | Порядок вставки | Сортировка по ключу |
| Скорость get/put | O(1) | O(1) | O(log n) |
| null ключ | 1 null | 1 null | ❌ |
| Когда | Быстрый поиск | Сохранить порядок | Нужна сортировка |

---

## 🔹 Queue — очередь

> [!note] Определение
> `Queue` — структура данных FIFO (First In, First Out). `Deque` — двусторонняя очередь (можно с обоих концов).

```java
import java.util.Queue;
import java.util.LinkedList;
import java.util.ArrayDeque;
import java.util.PriorityQueue;

// Queue — FIFO
Queue<String> queue = new LinkedList<>();
queue.offer("first");   // добавить в хвост (предпочтительнее add)
queue.offer("second");
queue.offer("third");

queue.peek();            // "first" — посмотреть без удаления
queue.poll();            // "first" — извлечь и удалить

// PriorityQueue — с приоритетом (минимальный элемент первым)
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(5);
pq.offer(1);
pq.offer(3);
System.out.println(pq.poll());  // 1 (минимальный)
System.out.println(pq.poll());  // 3

// Deque — стек + очередь (двусторонняя)
ArrayDeque<String> deque = new ArrayDeque<>();
deque.offerFirst("A");   // добавить в начало
deque.offerLast("B");    // добавить в конец
deque.peekFirst();       // посмотреть начало
deque.peekLast();        // посмотреть конец
deque.pollFirst();       // извлечь из начала
deque.pollLast();        // извлечь из конца

// Использование ArrayDeque как стека (LIFO)
ArrayDeque<Integer> stack = new ArrayDeque<>();
stack.push(1);  // = offerFirst
stack.push(2);
stack.push(3);
stack.pop();    // 3 = pollFirst (LIFO)
```

| Метод | При ошибке (пусто/полно) |
|-------|--------------------------|
| `add()` / `remove()` / `element()` | Бросает исключение |
| `offer()` / `poll()` / `peek()` | Возвращает `null` / `false` |

---

## 🔹 Утилитный класс Collections

```java
import java.util.Collections;

List<Integer> list = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6));

Collections.sort(list);           // [1, 1, 2, 3, 4, 5, 6, 9] — сортировка
Collections.reverse(list);        // [9, 6, 5, 4, 3, 2, 1, 1] — переворот
Collections.shuffle(list);        // случайный порядок
Collections.min(list);            // 1 — минимум
Collections.max(list);            // 9 — максимум
Collections.frequency(list, 1);   // 2 — сколько раз встречается 1
Collections.swap(list, 0, 1);     // поменять местами элементы 0 и 1
Collections.fill(list, 0);        // заполнить все нулями

// Неизменяемые обёртки
List<String> immutable = Collections.unmodifiableList(names);
// immutable.add("x");  // ❌ UnsupportedOperationException

// Синхронизированные обёртки (для многопоточности)
List<String> synced = Collections.synchronizedList(new ArrayList<>());

// Создание одноэлементных коллекций
List<String> single = Collections.singletonList("only");
Set<Integer> singleSet = Collections.singleton(42);
Map<String, Integer> singleMap = Collections.singletonMap("key", 1);
```

---

## 🔹 Массив vs ArrayList

| | Массив `int[]` | ArrayList`<Integer>` |
|-|----------------|----------------------|
| Размер | Фиксированный | Динамический |
| Тип | Примитивы + объекты | Только объекты (обёртки) |
| Производительность | Быстрее | Чуть медленнее (boxing) |
| Методы | Минимум | Богатый API |
| Null | Можно | Можно |
| Generic | Ограниченно | Полная поддержка |
| Когда | Фиксированный набор, производительность | Динамический список |

> [!warning] Подводные камни
> - При итерации и удалении из List используй `Iterator` или `removeIf`, иначе `ConcurrentModificationException`
> - `List.of(...)` — **неизменяемый** список (Java 9+), `add` бросит исключение
> - HashMap не гарантирует порядок, и порядок может меняться между запусками

```java
// Безопасное удаление во время итерации
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c", "b"));

// ❌ Неправильно — ConcurrentModificationException
for (String s : list) {
    if (s.equals("b")) list.remove(s);
}

// ✅ Правильно — removeIf
list.removeIf(s -> s.equals("b"));

// ✅ Правильно — Iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();
}
```
