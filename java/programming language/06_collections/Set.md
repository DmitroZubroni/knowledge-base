> **Теги:** #java #collections #set #hashset #treeset #linkedhashset #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Collections_Index]]

# Set — Множество

---

## 🔹 Что такое Set и зачем он нужен

Представь, что нужно хранить список пользователей, которые уже получили письмо. Если использовать ArrayList — один и тот же пользователь может попасть туда дважды, и письмо уйдёт повторно. Set решает эту проблему на уровне структуры данных: он **физически не может хранить дубликаты**.

> [!note] Set — это множество
> Коллекция, которая хранит только **уникальные** элементы. Если попытаться добавить дубликат — он просто игнорируется, без ошибок.

Три ключевых отличия от List:
- **Нет дубликатов** — одно и то же значение хранится только один раз
- **Нет гарантии порядка** (у HashSet) — порядок итерации непредсказуем
- **Нет доступа по индексу** — нельзя сделать `set.get(0)`

---

## 🔹 Set — это интерфейс

`Set` в Java — это интерфейс, а не класс. Его нельзя инстанцировать напрямую. Нужно использовать одну из реализаций:

```java
// ❌ Так нельзя — Set это интерфейс
Set<String> set = new Set<>();

// ✅ Правильно — используем конкретную реализацию
Set<String> set = new HashSet<>();
Set<String> set = new TreeSet<>();
Set<String> set = new LinkedHashSet<>();
```

Три реализации отличаются **порядком элементов** и **скоростью**. Выбор зависит от задачи.

---

## 🔹 HashSet — самый быстрый, без порядка

HashSet — это по сути **урезанный HashMap**. Внутри буквально хранится `HashMap`, где твои элементы — это ключи, а значение у всех одно: константа-заглушка `PRESENT`.

```
HashSet<String>  внутри →  HashMap<String, Object>
  "apple"                     "apple" → PRESENT
  "banana"                    "banana" → PRESENT
  "orange"                    "orange" → PRESENT
```

Именно поэтому HashSet такой же быстрый, как HashMap — O(1) на добавление, удаление и поиск.

```java
Set<String> fruits = new HashSet<>();

// Добавление — O(1)
fruits.add("apple");
fruits.add("banana");
fruits.add("orange");
fruits.add("apple");   // дубликат — игнорируется молча

System.out.println(fruits); // [apple, orange, banana] — порядок непредсказуем!
System.out.println(fruits.size()); // 3

// Проверка наличия — O(1) ← главное преимущество перед List
fruits.contains("apple");   // true
fruits.contains("grape");   // false

// Удаление — O(1)
fruits.remove("apple");

// Итерация
for (String fruit : fruits) {
    System.out.println(fruit);
}
fruits.forEach(System.out::println);  // через лямбду

// Очистка
fruits.clear();
fruits.isEmpty(); // true
```

> [!warning] Как HashSet определяет дубликаты
> Под капотом при `add()` вызывается `equals()`. Если элемент с таким же `equals()` уже есть — новый не добавляется.
>
> Это означает: если используешь свой класс как элемент HashSet — **обязательно переопредели `hashCode()` и `equals()`**. Без этого два логически одинаковых объекта будут считаться разными.

---

## 🔹 TreeSet — отсортированный, но медленнее

TreeSet хранит элементы в **отсортированном порядке**. Внутри — красно-чёрное дерево. Строки сортируются по алфавиту, числа — по возрастанию.

```java
Set<String> sorted = new TreeSet<>();
sorted.add("banana");
sorted.add("apple");
sorted.add("orange");

System.out.println(sorted); // [apple, banana, orange] — всегда алфавитный порядок
```

Сортировку можно изменить, передав `Comparator` в конструктор:

```java
// Обратный порядок
Set<String> reversed = new TreeSet<>(Comparator.reverseOrder());
reversed.add("banana");
reversed.add("apple");
reversed.add("orange");

System.out.println(reversed); // [orange, banana, apple]

// Свой компаратор (например, по длине строки)
Set<String> byLength = new TreeSet<>(Comparator.comparingInt(String::length));
```

TreeSet также даёт дополнительные методы для работы с диапазонами:

```java
TreeSet<Integer> nums = new TreeSet<>(Set.of(1, 3, 5, 7, 9));

nums.first();           // 1  — минимальный
nums.last();            // 9  — максимальный
nums.headSet(5);        // [1, 3]      — всё строго меньше 5
nums.tailSet(5);        // [5, 7, 9]   — всё начиная с 5
nums.subSet(3, 7);      // [3, 5]      — от 3 до 7, не включая 7
nums.floor(6);          // 5  — наибольший элемент ≤ 6
nums.ceiling(6);        // 7  — наименьший элемент ≥ 6
```

> [!warning] TreeSet требует сравнимости
> Элементы должны реализовывать `Comparable` или нужно передать `Comparator`. Если добавить объект без реализации `Comparable` — `ClassCastException` во время выполнения.

---

## 🔹 LinkedHashSet — порядок вставки сохранён

LinkedHashSet — это HashSet с дополнительным связанным списком, который помнит порядок добавления элементов.

```java
Set<String> linked = new LinkedHashSet<>();
linked.add("banana");
linked.add("apple");
linked.add("orange");
linked.add("apple");  // дубликат — игнорируется

System.out.println(linked); // [banana, apple, orange] — порядок вставки сохранён
```

Полезен, когда нужна уникальность И предсказуемый порядок итерации.

---

## 🔹 Сравнение реализаций

| | HashSet | LinkedHashSet | TreeSet |
|-|---------|---------------|---------|
| Внутренняя структура | HashMap | HashMap + LinkedList | Красно-чёрное дерево |
| Порядок | Не гарантирован | Порядок вставки | Отсортированный |
| `add` / `remove` / `contains` | **O(1)** | **O(1)** | O(log n) |
| Null элемент | ✅ один | ✅ один | ❌ |
| Когда использовать | Быстрая проверка уникальности | Уникальность + порядок вставки | Уникальность + нужна сортировка |

> [!tip] Правило выбора
> - Нет требований к порядку → **HashSet** (99% случаев)
> - Нужен порядок вставки → **LinkedHashSet**
> - Нужна сортировка или работа с диапазонами → **TreeSet**

---

## 🔹 Типичные задачи с Set

### Удаление дубликатов из List

```java
List<Integer> withDupes = Arrays.asList(1, 2, 3, 2, 1, 4, 3);

// Просто передаём List в конструктор Set
Set<Integer> unique = new HashSet<>(withDupes);
System.out.println(unique); // [1, 2, 3, 4]

// Если нужно сохранить порядок
Set<Integer> uniqueOrdered = new LinkedHashSet<>(withDupes);
System.out.println(uniqueOrdered); // [1, 2, 3, 4] — в порядке первого появления
```

### Проверка уникальности

```java
// Быстрая проверка: есть ли дубликаты в списке?
boolean hasDuplicates(List<String> list) {
    return list.size() != new HashSet<>(list).size();
}
```

### Операции над множествами

```java
Set<Integer> a = new HashSet<>(Arrays.asList(1, 2, 3, 4));
Set<Integer> b = new HashSet<>(Arrays.asList(3, 4, 5, 6));

// Пересечение (общие элементы)
Set<Integer> intersection = new HashSet<>(a);
intersection.retainAll(b);  // {3, 4}

// Объединение
Set<Integer> union = new HashSet<>(a);
union.addAll(b);            // {1, 2, 3, 4, 5, 6}

// Разность (есть в a, нет в b)
Set<Integer> difference = new HashSet<>(a);
difference.removeAll(b);    // {1, 2}

// Проверка: является ли a подмножеством b?
b.containsAll(a);           // false
```

### Подсчёт уникальных элементов

```java
String[] words = {"apple", "banana", "apple", "cherry", "banana"};
int uniqueCount = new HashSet<>(Arrays.asList(words)).size(); // 3
```

---

## 🔹 Set vs List — когда что использовать

| Задача | Выбор |
|--------|-------|
| Нужен доступ по индексу | **List** |
| Важен порядок элементов | **List** (или LinkedHashSet) |
| Нужны дубликаты | **List** |
| Нужна быстрая проверка `contains()` | **Set** ← O(1) vs O(n) у List |
| Нужна гарантия уникальности | **Set** |
| Операции над множествами (пересечение, объединение) | **Set** |

---

## 🔹 Итог

```
Set = коллекция уникальных элементов без индексов

Три реализации:
  HashSet        — O(1), порядок не гарантирован         (использовать по умолчанию)
  LinkedHashSet  — O(1), сохраняет порядок вставки
  TreeSet        — O(log n), элементы отсортированы

Уникальность определяется через equals() + hashCode().
Для своих классов — обязательно переопределяй оба метода.

Главное преимущество над List:
  contains() — O(1) у HashSet vs O(n) у ArrayList

Типичные применения:
  - убрать дубликаты из коллекции
  - быстро проверить "уже видели этот элемент?"
  - операции над множествами (пересечение, объединение, разность)
```
