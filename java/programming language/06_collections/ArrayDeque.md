> **Теги:** #java #collections #arraydeque #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Collections_Index]]

# ArrayDeque — Быстрая реализация Deque

---

## 🔹 Зачем отдельный конспект — разве не достаточно Deque?

Deque — это интерфейс. ArrayDeque — его самая быстрая реализация, и именно её нужно использовать в 99% случаев. Важно понимать **почему она быстрее** LinkedList, и что у неё есть одно ограничение, которое часто забывают.

---

## 🔹 Как устроен ArrayDeque внутри

Внутри ArrayDeque — **кольцевой массив** (circular buffer). Это обычный массив, но указатели `head` и `tail` могут "оборачиваться" через начало массива. Благодаря этому добавление и удаление с обоих концов работает за O(1) без сдвигов элементов.

```
Кольцевой массив с head=2, tail=6:

индексы:  [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]
данные:   [ _  |  _ | A  |  B |  C |  D |  _ |  _ ]
                     ↑ head                ↑ tail

offerLast("E"):  tail сдвигается вправо → tail=7
offerFirst("Z"): head сдвигается влево  → head=1

Когда tail достигает конца массива — он "оборачивается" на индекс 0:
[E | _ | Z | A | B | C | D | _ ]
 ↑tail  ↑head
```

**Почему быстрее LinkedList:**
- Элементы лежат рядом в памяти → процессор загружает их в кэш целыми блоками
- Нет объектов-узлов (Node) → меньше расходов на память и сборщик мусора
- Нет разыменования указателей → меньше промахов кэша

---

## 🔹 Расширение массива

Когда массив заполняется, ArrayDeque создаёт новый массив в **2 раза больше** и копирует все элементы. Это O(n), но происходит редко. Если знаешь размер заранее — задавай в конструкторе:

```java
Deque<String> deque = new ArrayDeque<>(100); // начальная ёмкость 100
```

> [!warning] ArrayDeque не принимает null
> Это принципиальное отличие от LinkedList. Попытка добавить `null` бросит `NullPointerException`.
>
> ```java
> deque.offer(null); // ❌ NullPointerException
> ```
>
> Если null-элементы нужны — используй LinkedList. В остальных случаях это ограничение не проблема, а скорее защита от ошибок.

---

## 🔹 ArrayDeque как Queue (FIFO)

```java
Deque<String> queue = new ArrayDeque<>();

queue.offer("first");   // добавить в хвост
queue.offer("second");
queue.offer("third");

queue.peek();   // "first" — посмотреть без удаления
queue.poll();   // "first" — удалить и вернуть
queue.poll();   // "second"
// Осталось: [third]
```

---

## 🔹 ArrayDeque как Stack (LIFO)

```java
Deque<String> stack = new ArrayDeque<>();

stack.push("bottom");  // addFirst — кладём на вершину
stack.push("middle");
stack.push("top");

stack.peek(); // "top"   — вершина без удаления
stack.pop();  // "top"   — снять с вершины
stack.pop();  // "middle"
// Осталось: [bottom]
```

---

## 🔹 Сравнение ArrayDeque vs LinkedList

| | ArrayDeque | LinkedList |
|-|------------|------------|
| Внутренняя структура | Кольцевой массив | Двусвязный список |
| `offerFirst` / `offerLast` | **O(1)** | O(1) |
| `pollFirst` / `pollLast` | **O(1)** | O(1) |
| Память | Меньше (нет Node объектов) | Больше (каждый узел = 2 ссылки) |
| Кэш-эффективность | **Высокая** (элементы рядом) | Низкая (узлы разбросаны) |
| Null элементы | ❌ | ✅ |
| `get(i)` по индексу | ❌ | O(n) |
| Итог | **Быстрее в большинстве задач** | Нужен только если нужен null |

---

## 🔹 Итог

```
ArrayDeque = кольцевой массив + два указателя (head, tail)

O(1) — offerFirst, offerLast, pollFirst, pollLast, peekFirst, peekLast
O(n) — поиск contains(), размер фиксирован → при переполнении удваивается

Null не поддерживается → NullPointerException

Использовать:
  Как Queue  → offer/poll/peek
  Как Stack  → push/pop/peek
  Как Deque  → offerFirst/offerLast/pollFirst/pollLast

Предпочитай ArrayDeque вместо:
  Stack      — устарел, синхронизирован, медленнее
  LinkedList — медленнее из-за кэш-промахов и накладных расходов Node
```
