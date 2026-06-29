> **Теги:** #java #collections #linkedhashmap #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Collections_Index]]

# LinkedHashMap — HashMap с памятью о порядке

---

## 🔹 Проблема HashMap: порядок непредсказуем

HashMap работает за O(1), но у неё есть особенность — порядок итерации элементов **никак не связан** с порядком их добавления. Под капотом элементы раскладываются по бакетам по хэш-коду, и при итерации ты получаешь их в том порядке, в котором они физически лежат в массиве бакетов.

```java
Map<String, Integer> map = new HashMap<>();
map.put("banana", 2);
map.put("apple", 1);
map.put("orange", 3);

System.out.println(map); // {apple=1, orange=3, banana=2} — порядок случайный
```

Если порядок важен — нужен LinkedHashMap.

---

## 🔹 Как LinkedHashMap решает проблему

LinkedHashMap — это HashMap, в которую добавили **двусвязный список**, пронизывающий все ноды в порядке их вставки. Каждая нода дополнительно хранит ссылки `before` и `after` на соседей по списку.

```
Внутренняя структура:

  table[] (бакеты, как в HashMap):
  [ null | Node("apple") | null | Node("banana") | Node("orange") | ... ]

  Двусвязный список поверх (порядок вставки):
  head ↔ Node("banana") ↔ Node("apple") ↔ Node("orange") ↔ tail
```

При итерации LinkedHashMap идёт **по двусвязному списку**, а не по бакетам. Поэтому порядок всегда совпадает с порядком вставки.

> [!note] Два режима порядка
> LinkedHashMap поддерживает два режима:
> - **Порядок вставки** (по умолчанию) — элементы итерируются в том порядке, в котором были добавлены
> - **Порядок доступа** — элемент перемещается в конец списка каждый раз, когда к нему обращаются через `get()` или `put()`

---

## 🔹 Порядок вставки — использование по умолчанию

```java
Map<String, Integer> map = new LinkedHashMap<>();
map.put("banana", 2);
map.put("apple", 1);
map.put("orange", 3);

System.out.println(map); // {banana=2, apple=1, orange=3} — порядок вставки!

// Обновление значения НЕ меняет позицию ключа в списке
map.put("banana", 99);
System.out.println(map); // {banana=99, apple=1, orange=3} — banana осталась первой
```

---

## 🔹 Порядок доступа — основа для LRU Cache

Второй режим включается третьим параметром конструктора: `accessOrder = true`.

```java
// LinkedHashMap(initialCapacity, loadFactor, accessOrder)
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true);

lru.put("a", 1);
lru.put("b", 2);
lru.put("c", 3);

System.out.println(lru); // {a=1, b=2, c=3}

lru.get("a"); // обращаемся к "a" — она перемещается в конец

System.out.println(lru); // {b=2, c=3, a=1} — "a" теперь в конце (самая "свежая")
```

Элементы в хвосте — самые недавно использованные. В голове — самые давно не использованные (Least Recently Used). Это идеальная основа для **LRU-кэша**.

### LRU Cache — классическая реализация

LRU (Least Recently Used) кэш — кэш фиксированного размера, который при заполнении вытесняет самый давно не использованный элемент.

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {

    private final int capacity;

    public LRUCache(int capacity) {
        // accessOrder=true — порядок доступа, а не вставки
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    // Этот метод вызывается после каждого put()
    // Если вернуть true — самый старый элемент (голова) удаляется автоматически
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}

// Использование
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");

cache.get(1);       // обращаемся к 1 — она перемещается в конец
cache.put(4, "four"); // кэш полон → вытесняется самый старый (2)

System.out.println(cache); // {3=three, 1=one, 4=four}
```

> [!tip] removeEldestEntry
> Это protected метод LinkedHashMap, специально предназначенный для переопределения. По умолчанию возвращает `false` (ничего не удалять). Переопределив его, получаем автоматическое вытеснение старых элементов.

---

## 🔹 API — всё то же, что у HashMap

LinkedHashMap реализует тот же интерфейс `Map`, поэтому API полностью идентичен HashMap. Разница только в поведении при итерации.

```java
Map<String, Integer> map = new LinkedHashMap<>();

map.put("a", 1);
map.get("a");                          // 1
map.getOrDefault("z", 0);             // 0
map.containsKey("a");                  // true
map.remove("a");
map.size();
map.isEmpty();
map.putIfAbsent("b", 2);
map.merge("b", 1, Integer::sum);

// Итерация — гарантированно в порядке вставки
for (Map.Entry<String, Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + " = " + e.getValue());
}
```

---

## 🔹 Сравнение с HashMap и TreeMap

| | HashMap | LinkedHashMap | TreeMap |
|-|---------|---------------|---------|
| Порядок | Не гарантирован | Порядок вставки (или доступа) | Отсортированный |
| `get` / `put` | **O(1)** | **O(1)** | O(log n) |
| Память | Меньше | Чуть больше (+2 ссылки на узел) | Больше (дерево) |
| Null ключ | ✅ | ✅ | ❌ |
| Когда использовать | Просто быстрый поиск | Нужен порядок вставки / LRU-кэш | Нужна сортировка |

---

## 🔹 Итог

```
LinkedHashMap = HashMap + двусвязный список поверх нод

Два режима:
  accessOrder=false (по умолчанию) → порядок вставки
  accessOrder=true                 → порядок доступа (LRU)

Сложности: те же что у HashMap — O(1) для put/get/remove
Память: чуть больше HashMap (+before/after ссылки в каждой ноде)

Главные применения:
  1. Нужна HashMap, но с предсказуемым порядком итерации
  2. Реализация LRU-кэша через переопределение removeEldestEntry()
```
