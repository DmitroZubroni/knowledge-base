# Collections

> **Теги:** #interviews #java-core #collections #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Иерархия коллекций

```
Iterable
    └── Collection
            ├── List
            │   ├── ArrayList
            │   └── LinkedList
            ├── Queue
            │   ├── Deque
            │   │   ├── LinkedList
            │   │   └── ArrayDeque
            │   └── PriorityQueue
            └── Set
                ├── TreeSet
                ├── HashSet
                ├── LinkedHashSet
                └── EnumSet
```

**Map** — не наследуется от Collection, стоит особняком:

```
Map
    ├── TreeMap
    ├── HashMap
    ├── LinkedHashMap
    └── Hashtable
```

> [!note] Map ≠ Collection
> Map не наследуется от Iterable, потому что это не коллекция, а карта пар ключ-значение.

---

## 🔹 ArrayList vs LinkedList

### ArrayList

**Устройство:** динамический массив

```
Создание: массив размером 10
При заполнении: увеличение в 1.5 раза
├── Создаётся новый массив большего размера
├── Элементы копируются из старого в новый
└── Новый элемент добавляется
```

**Асимптотика:**

| Операция | Сложность | Почему |
|----------|-----------|--------|
| `add(e)` (в конец) | O(1)* | Константная, но при resize — O(n) |
| `get(i)` | O(1) | Массив в памяти единым блоком, быстрый доступ по индексу |
| `set(i, e)` | O(1) | Быстрая замена по индексу |
| `remove(i)` | O(n) | Нужно сдвинуть все элементы после удалённого |

*амортизированная

### LinkedList

**Устройство:** двусвязный список узлов

```
Node:
├── value
├── next → следующий узел
└── prev → предыдущий узел
```

**Асимптотика:**

| Операция | Сложность | Почему |
|----------|-----------|--------|
| `add(e)` (в конец) | O(1) | Просто добавляем ссылку от хвоста |
| `get(i)` | O(n) | Нужно пройти по ссылкам от головы до индекса |
| `add(i, e)` | O(n) | Сначала дойти до индекса, потом изменить ссылки |
| `remove(i)` | O(n) | Сначала дойти до индекса, потом перелинковать |

### Сравнение

| Характеристика | ArrayList | LinkedList |
|----------------|-----------|------------|
| Доступ по индексу | O(1) — быстро | O(n) — медленно |
| Вставка/удаление в конец | O(1)* | O(1) |
| Вставка/удаление в середину | O(n) — сдвиг | O(n) — поиск + O(1) перелинковка |
| Память | Единый блок | Разрозненные узлы + ссылки |
| Использование на практике | Почти всегда | Редко (когда нужна Queue/Deque) |

> [!tip] Когда использовать LinkedList
- Когда нужен интерфейс Deque (двусторонняя очередь)
- Когда много операций удаления при итерации
- Но на практике чаще используют ArrayDeque для Deque

---

## 🔹 HashMap под капотом

**Устройство:** массив бакетов → каждый бакет содержит узлы (Node)

```
HashMap:
├── buckets[] (массив)
│   ├── Node [key, value, hash, next]
│   ├── Node → Node → Node (коллизия)
│   └── ...
```

### Процесс добавления элемента

```
1. Вычисляем hashCode ключа
2. Применяем хеш-функцию: index = hash % buckets.length
3. Если бакет пуст → создаём новый Node
4. Если бакет занят (коллизия):
   - До Java 8: linked list в бакете
   - Java 8+: при 8+ элементах → красно-чёрное дерево (O(log n))
5. Если ключ уже существует → перезаписываем value
```

### Коллизии

**До Java 8:** linked list в бакете → O(n) при поиске

**Java 8+:** при 8+ элементах в бакете → дерево → O(log n)

> [!tip] Treeify
- Порог: 8 элементов в бакете
- Обратно в linked list при 6 элементах
- Улучшение для случаев с плохой хеш-функцией

---

## 🔹 equals и hashCode контракт

### Требования к ключу HashMap

1. **Immutable** — объект не должен меняться после добавления в Map
2. **Переопределить equals()** — корректное сравнение по полям
3. **Переопределить hashCode()** — должен использовать те же поля, что и equals

### Контракт

```java
// Если equals() возвращает true для двух объектов,
// то hashCode() должен возвращать одно и то же значение

a.equals(b) == true  →  a.hashCode() == b.hashCode()

// Обратное не обязательно:
a.hashCode() == b.hashCode()  ↛  a.equals(b) == true
```

### Почему immutable ключи?

Если ключ изменится после добавления в Map:
- hashCode изменится
- При поиске вычисляется новый hashCode → другой бакет
- Элемент не найдётся

### Пример реализации

```java
public class User {
    private final String name;
    private final String surname;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(name, user.name) &&
               Objects.equals(surname, user.surname);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, surname);
    }
}
```

> [!warning] String как ключ
- String immutable → идеально подходит
- Но можно использовать и кастомный immutable класс

---

## 🔹 Set реализации

| Реализация | Под капот | Особенности |
|------------|-----------|-------------|
| **TreeSet** | Красно-чёрное дерево | Упорядочен, O(log n) операции |
| **HashSet** | HashMap | Неупорядочен, O(1) операции |
| **LinkedHashSet** | HashMap + связи | Сохраняет порядок вставки |
| **EnumSet** | Bit vector | Оптимизирован для enum |

---

## 🔹 Map реализации

| Реализация | Под капот | Особенности |
|------------|-----------|-------------|
| **TreeMap** | Красно-чёрное дерево | Ключи отсортированы, O(log n) |
| **HashMap** | Hash table | Неупорядочен, O(1) операции |
| **LinkedHashMap** | Hash table + связи | Сохраняет порядок вставки |
| **Hashtable** | Hash table | Синхронизированная, устарела |

> [!tip] Hashtable vs ConcurrentHashMap
- Hashtable — устарела, все методы synchronized
- ConcurrentHashMap — современная, finer-grained locking

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **ArrayList** — O(1) get/set, O(n) remove, используй по умолчанию
> - **LinkedList** — O(n) get, используй только для Deque
> - **HashMap** — массив бакетов, коллизии → tree при 8+ элементах
> - **equals/hashCode** — используй одинаковые поля, immutable ключи
> - **TreeSet/TreeMap** — красно-чёрное дерево, O(log n), упорядочены
> - **HashSet/HashMap** — hash table, O(1), неупорядочены

```
ArrayList vs LinkedList:
get: O(1) vs O(n)
remove: O(n) vs O(n) (но LinkedList быстрее при итерации)

HashMap:
hash → index → bucket
коллизия: linked list → tree (Java 8+, 8+ элементов)

equals/hashCode:
a.equals(b) → a.hashCode() == b.hashCode()
ключ должен быть immutable
```
