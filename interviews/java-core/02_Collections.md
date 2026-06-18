> **Теги:** #interviews #java-core #collections #конспект
> [!abstract] Связи
> [[main]] | [[Interviews]] | [[Collections_Index]]

# Collections — Вопросы на собесе

---

## 🔹 Иерархия коллекций

```
Iterable
  └── Collection
        ├── List       — упорядочен, дубликаты разрешены
        │   ├── ArrayList
        │   └── LinkedList
        ├── Set        — уникальные элементы
        │   ├── HashSet
        │   ├── LinkedHashSet
        │   └── TreeSet
        └── Queue      — FIFO / приоритет
            ├── PriorityQueue
            └── Deque
                ├── ArrayDeque
                └── LinkedList

Map (не Collection!)
  ├── HashMap
  ├── LinkedHashMap
  ├── TreeMap
  └── ConcurrentHashMap
```

> Map не наследуется от Collection — это карта пар ключ-значение, а не просто набор элементов.

---

## 🔹 ArrayList vs LinkedList

| | ArrayList | LinkedList |
|-|-----------|------------|
| Внутри | Object[] массив | Двусвязный список Node |
| get(i) | **O(1)** | O(n) |
| add в конец | O(1) амортизированно | O(1) |
| add(i, e) в середину | O(n) — сдвиг | O(n) — найти позицию |
| remove(i) | O(n) — сдвиг | O(n) — найти позицию |
| Память | Компактнее | Больше (2 ссылки на узел) |
| Кэш CPU | Хорошо | Плохо (узлы разбросаны) |

**Вопрос: "Когда LinkedList лучше?"**
Честный ответ: почти никогда. Даже как Deque — `ArrayDeque` быстрее. LinkedList полезен если нужен null-элемент в Deque или реально часто добавляешь в начало с уже известным итератором.

**ArrayList расширение:** при заполнении создаётся массив × 1.5, элементы копируются — O(n). Если знаешь размер заранее — `new ArrayList<>(n)`.

---

## 🔹 HashMap под капотом

**Структура:** `Node[] table` — массив бакетов. Каждый Node хранит `hash, key, value, next`.

**put(key, value):**
1. `hash(key)` — `hashCode() ^ (h >>> 16)` — XOR со сдвигом, чтобы старшие биты влияли на бакет
2. Бакет: `hash & (capacity - 1)` — быстрый аналог `%`, работает т.к. capacity = степень двойки
3. Пуст → новый Node. Тот же ключ → перезапись. Коллизия → в цепочку
4. Список в бакете ≥ 8 нод **И** таблица ≥ 64 элементов → **treeify** (красно-чёрное дерево)

**Параметры по умолчанию:**
- `capacity = 16` (всегда степень двойки)
- `loadFactor = 0.75` → rehash при 12 элементах
- `TREEIFY_THRESHOLD = 8`, `UNTREEIFY_THRESHOLD = 6`

**Rehash:** capacity × 2, все ноды перераспределяются — O(n). Задавай capacity заранее: `new HashMap<>(expectedSize / 0.75 + 1)`.

---

## 🔹 equals() и hashCode() — контракт

```
Правило: a.equals(b) == true  →  a.hashCode() == b.hashCode()
Обратное НЕ обязательно: одинаковый hash ≠ равные объекты (коллизия — нормально)
```

**Почему нужны оба:**
- HashMap сначала ищет бакет по `hashCode()`, потом ищет в цепочке через `equals()`
- Если переопределить только `equals()` — два "равных" объекта попадут в разные бакеты → `get()` не найдёт
- Если переопределить только `hashCode()` — попадут в один бакет, но `equals()` скажет что они разные → дубликаты в Map

**Ключи должны быть effectively immutable** — если ключ изменится, его `hashCode()` изменится, и объект потеряется в Map.

```java
// Правильная реализация через Objects.hash()
@Override public int hashCode() { return Objects.hash(name, age); }
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User u)) return false;
    return age == u.age && Objects.equals(name, u.name);
}
```

---

## 🔹 LinkedHashMap — когда нужен порядок

**Внутри:** HashMap + двусвязный список поверх нод (поля `before`/`after`).

Два режима:
- `new LinkedHashMap<>()` — **порядок вставки**
- `new LinkedHashMap<>(16, 0.75f, true)` — **порядок доступа** (LRU)

**LRU Cache** через `removeEldestEntry`:
```java
new LinkedHashMap<>(capacity, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry e) {
        return size() > capacity; // старейший удаляется автоматически
    }
};
```

Сложность: те же O(1) что у HashMap. Память чуть больше (+2 ссылки на ноду).

---

## 🔹 TreeMap — когда нужна сортировка

**Внутри:** красно-чёрное дерево. Ключи всегда отсортированы.

**Сложность:** O(log n) для всех операций.

**Null ключи:** запрещены (ключи сравниваются, `compareTo(null)` → NPE).

**Навигационные методы — главное отличие от HashMap:**
```java
TreeMap<Integer, String> map = new TreeMap<>();
map.firstKey() / map.lastKey()          // min / max
map.floorKey(5)                         // наибольший ключ ≤ 5
map.ceilingKey(5)                       // наименьший ключ ≥ 5
map.headMap(5) / map.tailMap(5)         // всё < 5 / всё ≥ 5
map.subMap(3, 7)                        // [3, 7)
map.pollFirstEntry() / pollLastEntry()  // удалить и вернуть крайний
```

**Когда использовать:** нужна Map с отсортированными ключами, поиск ближайшего ключа, работа с диапазонами.

---

## 🔹 Set реализации

| | HashSet | LinkedHashSet | TreeSet |
|-|---------|---------------|---------|
| Внутри | HashMap | HashMap + список | Красно-чёрное дерево |
| Порядок | Нет | Порядок вставки | Отсортированный |
| contains | **O(1)** | **O(1)** | O(log n) |
| Null | ✅ | ✅ | ❌ |

**Главное преимущество Set перед List:** `contains()` — O(1) у HashSet против O(n) у ArrayList.

**HashSet внутри — это просто HashMap**, где элементы — ключи, а значение — константа `PRESENT`.

---

## 🔹 Queue, Deque, ArrayDeque, PriorityQueue

**Queue — FIFO.** Два набора методов:
- С исключением: `add / element / remove`
- Без исключения: `offer / peek / poll` ← используй в продакшене

**Deque** — двусторонняя очередь. Три роли:
```
FIFO:   offer(e) / poll()              — хвост / голова
LIFO:   push(e) / pop()                — голова / голова
Deque:  offerFirst/Last, pollFirst/Last
```

**ArrayDeque** — кольцевой массив, O(1) на обоих концах. **Быстрее LinkedList** (кэш CPU). Не принимает null. **Используй вместо Stack**.

**PriorityQueue** — бинарная куча (min-heap по умолчанию):
- `offer(e)` — O(log n), `peek()` — O(1), `poll()` — O(log n)
- Max-heap: `new PriorityQueue<>(Comparator.reverseOrder())`
- Итерация через for-each **не даёт** отсортированный порядок — только `poll()` в цикле
- Классика: k наибольших элементов, алгоритм Дейкстры

---

## 🔹 ConcurrentHashMap vs synchronizedMap

| | synchronizedMap | ConcurrentHashMap |
|-|-----------------|-------------------|
| Блокировка | Один lock на всё | CAS + lock на бакет |
| Чтение | Блокируется | Без блокировки |
| Null ключ | ✅ | ❌ |
| Производительность | Плохо при конкуренции | Хорошо |

**Атомарные операции** (без доп. синхронизации):
```java
map.putIfAbsent(key, value)
map.computeIfAbsent(key, k -> new ArrayList<>())
map.merge(key, 1, Integer::sum)  // подсчёт частоты
```

---

## 🔹 Типичные вопросы и ответы

**Q: Что будет если изменить ключ HashMap после добавления?**
A: Объект потеряется — `hashCode()` изменится, HashMap будет искать в другом бакете.

**Q: Почему capacity HashMap всегда степень двойки?**
A: Чтобы заменить дорогой `% capacity` на быстрый `& (capacity-1)` — побитовая операция в разы быстрее.

**Q: Чем отличается fail-fast от fail-safe итератора?**
A: `fail-fast` (ArrayList, HashMap) — бросает `ConcurrentModificationException` при изменении во время итерации. `fail-safe` (ConcurrentHashMap, CopyOnWriteArrayList) — итерирует по снимку, не бросает исключение.

**Q: Как правильно удалять элементы при итерации?**
A: `iterator.remove()`, `list.removeIf(pred)`, или итерация с конца по индексу. Никогда не `list.remove()` внутри for-each.

**Q: В чём разница ArrayList и Vector?**
A: Vector устарел — все методы `synchronized`, медленно. Используй ArrayList + внешняя синхронизация или CopyOnWriteArrayList.

---

## 🔹 Шпаргалка

```
List:
  ArrayList  — get O(1), add конец O(1), insert/remove O(n). Default.
  LinkedList — get O(n). Только для Deque и то хуже ArrayDeque.

Map:
  HashMap        — O(1), порядка нет, null ключ ✅
  LinkedHashMap  — O(1), порядок вставки / LRU-кэш
  TreeMap        — O(log n), отсортирован, floor/ceiling/subMap, null ❌
  ConcurrentHashMap — потокобезопасен, null ❌

Set = Map без value. HashSet/LinkedHashSet/TreeSet — те же правила.

Queue / Deque:
  ArrayDeque    — O(1) оба конца, быстрее LinkedList, null ❌. Default Deque/Stack.
  PriorityQueue — O(log n) offer/poll, O(1) peek. Min-heap по умолчанию.

equals + hashCode:
  equals → одинаковый hashCode (обратное необязательно)
  Ключи HashMap должны быть effectively immutable

HashMap internals:
  capacity=16, loadFactor=0.75, rehash при 12 элементах
  treeify при ≥8 нод в бакете И ≥64 элементов в таблице
  hash = hashCode() ^ (h >>> 16)  — XOR для лучшего распределения
```
