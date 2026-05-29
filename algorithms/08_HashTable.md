> **Теги:** #algorithms #hashtable #hashmap #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[main Algorithms]]

# 🗂 Хэш-таблицы (Hash Tables)

---

## 🔹 Что такое хэш-таблица

> [!note] Определение
> **Хэш-таблица** — структура данных для хранения пар ключ-значение. Ключ преобразуется хэш-функцией в индекс массива, по которому хранится значение. Доступ — O(1) в среднем.

```
key "cat" → hash("cat") = 3 → bucket[3] = "value"
key "dog" → hash("dog") = 7 → bucket[7] = "value"
```

---

## 🔹 Хэш-функция

> [!note] Хорошая хэш-функция
> 1. **Детерминированная** — один ключ → всегда одинаковый хэш
> 2. **Равномерное распределение** — минимум коллизий
> 3. **Быстрая** — O(1) вычисление

```java
// Java вычисляет hashCode() для каждого объекта
"hello".hashCode()    // -1220993920
42 .hashCode()        // 42
new Integer(42).hashCode() // 42

// Для пользовательских объектов — переопределяй hashCode()
class Point {
    int x, y;

    @Override
    public int hashCode() {
        return Objects.hash(x, y); // стандартный способ
    }

    @Override
    public boolean equals(Object o) {
        // ВСЕГДА переопределяй equals вместе с hashCode!
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
}
```

> [!warning] Правило
> Если переопределяешь `hashCode()`, **обязательно** переопределяй `equals()`. Иначе HashMap / HashSet будут работать некорректно.

---

## 🔹 Коллизии и способы их разрешения

> [!note] Коллизия
> Ситуация, когда два разных ключа дают **одинаковый хэш** (попадают в один bucket).

### Chaining (Цепочки) — Java HashMap
```
bucket[3] → [("cat","A")] → [("act","B")] → null
```
Каждый bucket содержит связанный список (или дерево при большом числе коллизий).

### Open Addressing (Открытая адресация)
При коллизии ищем следующий свободный слот по правилу (linear probing, quadratic probing).

```
// Linear probing: если bucket[i] занят → пробуем bucket[i+1], i+2...
```

---

## 🔹 HashMap в Java

```java
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> map = new HashMap<>();

// Основные операции — O(1) в среднем
map.put("apple", 5);           // добавить / обновить
map.get("apple");              // → 5
map.getOrDefault("pear", 0);   // → 0 (если нет ключа)
map.containsKey("apple");      // → true
map.containsValue(5);          // → true — O(n)!
map.remove("apple");           // удалить
map.size();
map.isEmpty();

// Итерация
for (Map.Entry<String, Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + " = " + e.getValue());
}
for (String key : map.keySet()) { ... }
for (int val : map.values()) { ... }

// Полезные методы
map.putIfAbsent("apple", 3);          // не перезаписывает
map.merge("apple", 1, Integer::sum);  // apple = apple + 1
map.compute("apple", (k, v) -> v == null ? 1 : v + 1);
map.getOrDefault(key, defaultVal);
```

### Подсчёт частоты (частая задача)
```java
String[] words = {"a", "b", "a", "c", "b", "a"};
Map<String, Integer> freq = new HashMap<>();

for (String w : words) {
    freq.merge(w, 1, Integer::sum);
    // или: freq.put(w, freq.getOrDefault(w, 0) + 1);
}
// {a=3, b=2, c=1}
```

---

## 🔹 HashSet в Java

```java
import java.util.HashSet;
import java.util.Set;

Set<String> set = new HashSet<>();

set.add("apple");       // O(1)
set.contains("apple");  // O(1)
set.remove("apple");    // O(1)
set.size();

// Операции над множествами
Set<Integer> a = new HashSet<>(Arrays.asList(1, 2, 3));
Set<Integer> b = new HashSet<>(Arrays.asList(2, 3, 4));

a.retainAll(b);  // пересечение → {2, 3}
a.addAll(b);     // объединение
a.removeAll(b);  // разность
```

---

## 🔹 LinkedHashMap и TreeMap

```java
// LinkedHashMap — сохраняет порядок вставки
Map<String, Integer> linked = new LinkedHashMap<>();
linked.put("b", 2);
linked.put("a", 1);
// итерация даст b, a (порядок вставки)

// TreeMap — отсортирован по ключу, O(log n) все операции
Map<String, Integer> tree = new TreeMap<>();
tree.put("b", 2);
tree.put("a", 1);
// итерация даст a, b (алфавитный порядок)
tree.firstKey();  // "a"
tree.lastKey();   // "b"
```

| Класс | Порядок | Сложность операций | Null ключи |
|-------|---------|-------------------|------------|
| HashMap | нет | O(1) | ✅ один |
| LinkedHashMap | порядок вставки | O(1) | ✅ один |
| TreeMap | сортированный | O(log n) | ❌ |

---

## 🔹 Типичные задачи на HashMap

### Two Sum
```java
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>(); // значение → индекс
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }
        map.put(nums[i], i);
    }
    return new int[]{};
}
// O(n) time, O(n) space
```

### Анаграммы
```java
boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    Map<Character, Integer> map = new HashMap<>();
    for (char c : s.toCharArray()) map.merge(c, 1, Integer::sum);
    for (char c : t.toCharArray()) {
        map.merge(c, -1, Integer::sum);
        if (map.get(c) < 0) return false;
    }
    return true;
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **HashMap** — O(1) put/get/remove, нет порядка
> - **LinkedHashMap** — O(1) + сохраняет порядок вставки
> - **TreeMap** — O(log n) + ключи отсортированы
> - **HashSet** — уникальные элементы, O(1) contains
> - Всегда переопределяй `equals()` + `hashCode()` вместе
> - `getOrDefault` / `merge` / `putIfAbsent` — пиши лаконично
> - Two Sum — классика, HashMap решает за O(n)
