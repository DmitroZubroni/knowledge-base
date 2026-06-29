> **Теги:** #java #collections #treemap #красно-чёрное-дерево #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Collections_Index]]

# TreeMap — отсортированная Map

---

## 🔹 Когда HashMap и LinkedHashMap не подходят

HashMap даёт O(1) и не гарантирует порядок. LinkedHashMap помнит порядок вставки. Но ни один из них не ответит на вопросы вроде:

- "Какой ключ самый маленький / самый большой?"
- "Все пользователи с возрастом от 18 до 30?"
- "Ключ, ближайший к значению X?"

Для таких задач нужна структура, где ключи **постоянно отсортированы**. Это TreeMap.

---

## 🔹 Как устроен TreeMap внутри

Внутри TreeMap — **красно-чёрное дерево** (Red-Black Tree). Это самобалансирующееся бинарное дерево поиска. Каждый узел хранит ключ, значение и ссылки на левого и правого потомка.

```
        "mango"
       /        \
  "apple"      "orange"
       \           \
     "banana"    "peach"
```

Свойство бинарного дерева поиска: всё левее узла — меньше него, всё правее — больше. Из-за этого:
- **Все ключи всегда отсортированы** — не нужно сортировать отдельно
- Поиск, вставка, удаление — O(log n): на каждом шагу отсекается половина дерева

> [!note] Почему красно-чёрное, а не обычное?
> Обычное BST деградирует в список при добавлении уже отсортированных элементов (O(n) поиск). Красно-чёрное дерево автоматически перебалансируется при каждой вставке и удалении, гарантируя высоту O(log n) всегда.

---

## 🔹 Порядок сортировки ключей

По умолчанию TreeMap сортирует ключи по **естественному порядку** — ключи должны реализовывать `Comparable`.

```java
Map<String, Integer> map = new TreeMap<>();
map.put("banana", 2);
map.put("apple", 1);
map.put("orange", 3);

System.out.println(map); // {apple=1, banana=2, orange=3} — алфавитный порядок всегда
```

Если нужен другой порядок — передаём `Comparator` в конструктор:

```java
// Обратный порядок
Map<String, Integer> reversed = new TreeMap<>(Comparator.reverseOrder());
reversed.put("banana", 2);
reversed.put("apple", 1);
reversed.put("orange", 3);
System.out.println(reversed); // {orange=3, banana=2, apple=1}

// Сортировка по длине строки
Map<String, Integer> byLength = new TreeMap<>(
    Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder())
);

// Для числовых ключей — автоматически по возрастанию
Map<Integer, String> nums = new TreeMap<>();
nums.put(5, "five");
nums.put(1, "one");
nums.put(3, "three");
System.out.println(nums); // {1=one, 3=three, 5=five}
```

> [!warning] TreeMap не принимает null ключи
> HashMap и LinkedHashMap допускают один null-ключ. TreeMap — нет, потому что ключи нужно сравнивать между собой, а `compareTo(null)` бросит `NullPointerException`.
>
> ```java
> map.put(null, "value"); // ❌ NullPointerException
> ```

---

## 🔹 Навигационные методы — главная сила TreeMap

Это то, ради чего TreeMap и выбирают. HashMap и LinkedHashMap таких методов не имеют.

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "one");
map.put(3, "three");
map.put(5, "five");
map.put(7, "seven");
map.put(9, "nine");

// --- Крайние элементы ---
map.firstKey();          // 1  — минимальный ключ
map.lastKey();           // 9  — максимальный ключ
map.firstEntry();        // 1=one
map.lastEntry();         // 9=nine

// --- Поиск ближайших ---
map.floorKey(6);         // 5  — наибольший ключ ≤ 6
map.ceilingKey(6);       // 7  — наименьший ключ ≥ 6
map.lowerKey(5);         // 3  — наибольший ключ строго < 5
map.higherKey(5);        // 7  — наименьший ключ строго > 5

map.floorEntry(6);       // 5=five  (те же методы, но возвращают Entry)
map.ceilingEntry(6);     // 7=seven

// --- Диапазоны ---
map.headMap(5);          // {1=one, 3=three}        — ключи строго < 5
map.headMap(5, true);    // {1=one, 3=three, 5=five} — ключи ≤ 5 (включительно)
map.tailMap(5);          // {5=five, 7=seven, 9=nine} — ключи ≥ 5
map.tailMap(5, false);   // {7=seven, 9=nine}         — ключи строго > 5
map.subMap(3, 7);        // {3=three, 5=five}          — от 3 включительно до 7 не включая
map.subMap(3, true, 7, true); // {3=three, 5=five, 7=seven} — оба включительно

// Обратный порядок
map.descendingMap();     // {9=nine, 7=seven, 5=five, 3=three, 1=one}
map.descendingKeySet();  // [9, 7, 5, 3, 1]

// --- Удаление с края ---
map.pollFirstEntry();    // удалить и вернуть минимальный
map.pollLastEntry();     // удалить и вернуть максимальный
```

---

## 🔹 Практические задачи

### Найти ближайшее значение в словаре

```java
// Есть курсы валют на определённые даты, нужен курс на конкретный день
// (или ближайший предшествующий)
TreeMap<LocalDate, Double> rates = new TreeMap<>();
rates.put(LocalDate.of(2024, 1, 1), 89.5);
rates.put(LocalDate.of(2024, 1, 10), 91.2);
rates.put(LocalDate.of(2024, 1, 20), 88.7);

LocalDate query = LocalDate.of(2024, 1, 15);
Map.Entry<LocalDate, Double> rate = rates.floorEntry(query);
// 2024-01-10 = 91.2 — ближайший известный курс до нужной даты
```

### Подсчёт частоты и топ элементов

```java
String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};
TreeMap<String, Integer> freq = new TreeMap<>();
for (String w : words) {
    freq.merge(w, 1, Integer::sum);
}
System.out.println(freq); // {apple=3, banana=2, cherry=1} — отсортировано по ключу
```

### Расписание: все события в диапазоне времени

```java
TreeMap<Integer, String> schedule = new TreeMap<>();
schedule.put(900,  "standup");
schedule.put(1100, "review");
schedule.put(1400, "lunch");
schedule.put(1600, "demo");
schedule.put(1800, "retro");

// Все события с 10:00 до 15:00
NavigableMap<Integer, String> afternoon = schedule.subMap(1000, true, 1500, true);
System.out.println(afternoon); // {1100=review, 1400=lunch}
```

---

## 🔹 Интерфейсы NavigableMap и SortedMap

TreeMap реализует несколько интерфейсов. Это важно для понимания, какой тип использовать в сигнатуре метода:

```
Map
 └── SortedMap      — firstKey(), lastKey(), headMap(), tailMap(), subMap()
      └── NavigableMap — floor(), ceiling(), lower(), higher(), descendingMap(), pollFirst/Last()
           └── TreeMap  — конкретная реализация
```

```java
// Используй наиболее специфичный тип, который нужен:
SortedMap<K, V> m = new TreeMap<>();      // если нужны только диапазоны
NavigableMap<K, V> m = new TreeMap<>();   // если нужны floor/ceiling/poll
TreeMap<K, V> m = new TreeMap<>();        // если нужны все методы TreeMap
```

---

## 🔹 Сложность операций

| Операция | Сложность | Почему |
|----------|-----------|--------|
| `put` / `get` / `remove` | **O(log n)** | Спуск по дереву высотой log n |
| `firstKey` / `lastKey` | O(log n) | Самый левый / правый узел дерева |
| `floorKey` / `ceilingKey` | O(log n) | Поиск в дереве |
| `headMap` / `tailMap` / `subMap` | O(log n) | Найти границу + итерация по диапазону O(k) |
| Итерация по всем | O(n) | In-order обход дерева |

---

## 🔹 Сравнение HashMap / LinkedHashMap / TreeMap

| | HashMap | LinkedHashMap | TreeMap |
|-|---------|---------------|---------|
| Внутренняя структура | Массив бакетов | Массив бакетов + список | Красно-чёрное дерево |
| Порядок | Нет | Вставки (или доступа) | Отсортированный |
| `get` / `put` | **O(1)** | **O(1)** | O(log n) |
| `firstKey` / `lastKey` | ❌ | ❌ | ✅ O(log n) |
| `floorKey` / `ceilingKey` | ❌ | ❌ | ✅ O(log n) |
| Диапазоны `subMap` | ❌ | ❌ | ✅ |
| Null ключ | ✅ | ✅ | ❌ |

---

## 🔹 Итог

```
TreeMap = красно-чёрное дерево
          ключи всегда отсортированы (естественный порядок или Comparator)

Сложность: O(log n) для всех операций (медленнее HashMap, но это цена за сортировку)

Null ключи — запрещены (нельзя сравнивать с null)

Когда использовать:
  - нужна Map с отсортированными ключами
  - нужно найти ближайший ключ (floor/ceiling)
  - нужна работа с диапазонами ключей (subMap, headMap, tailMap)
  - нужно быстро получить min/max ключ

Интерфейсы: TreeMap → NavigableMap → SortedMap → Map

Главные навигационные методы:
  firstKey / lastKey          — крайние ключи
  floorKey(k) / ceilingKey(k) — ближайший ≤ k / ≥ k
  lowerKey(k) / higherKey(k)  — ближайший строго < k / > k
  subMap / headMap / tailMap  — диапазоны
  pollFirstEntry/pollLastEntry — удалить и вернуть крайний элемент
```
