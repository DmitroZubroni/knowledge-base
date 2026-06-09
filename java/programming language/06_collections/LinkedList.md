> **Теги:** #java #collections #linkedlist #связанный-список #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Collections_Index]]

# LinkedList — Связанный список

---

## 🔹 Проблема ArrayList при вставке и удалении

ArrayList быстро читает по индексу, но вставка и удаление в середину — O(n), потому что нужно сдвигать все элементы. Если в задаче постоянно вставляют и удаляют элементы в начале или середине — это дорого.

```
ArrayList: удалить элемент с индексом 1
[ A | B | C | D | E ]
      ↑ удаляем
[ A | C | D | E | _ ]  ← C, D, E сдвинулись влево — O(n)
```

LinkedList решает эту проблему по-другому: элементы вообще не лежат подряд в памяти, поэтому никаких сдвигов нет.

---

## 🔹 Структура: узлы со ссылками

В основе LinkedList — **узлы (Node)**. Каждый узел хранит:
- **value** — само значение
- **next** — ссылку на следующий узел

```
Node {         Node {         Node {
  value = 10     value = 20     value = 30
  next  = ──────► next  = ──────► next  = null
}              }              }
↑ head                         ↑ tail
```

Список знает только где **голова (head)** и **хвост (tail)**. Никакого массива внутри нет — узлы разбросаны по памяти и связаны ссылками.

> [!note] Односвязный vs двусвязный список
> - **Односвязный** — каждый узел хранит только `next`. Двигаться можно только вперёд.
> - **Двусвязный** — каждый узел хранит `next` и `prev`. Можно двигаться в обе стороны. Именно так устроен `java.util.LinkedList`.

```
Двусвязный список:
null ◄── Node(10) ◄──► Node(20) ◄──► Node(30) ──► null
         ↑ head                       ↑ tail
```

---

## 🔹 Операции и их сложность

### Добавление в начало — O(1)

Просто создаём новый узел и перебиваем `head`:

```
Было:   [10] → [20] → [30]
                        ↑ tail
head ↑

Добавить 5 в начало:
[5] → [10] → [20] → [30]
 ↑ новый head
```

Ничего не сдвигается — просто новая ссылка. O(1).

### Добавление в конец — O(1)

Аналогично: создаём новый узел, перебиваем `tail.next` и обновляем `tail`:

```
Было:   [10] → [20] → [30]
                        ↑ tail

Добавить 40 в конец:
[10] → [20] → [30] → [40]
                       ↑ новый tail
```

O(1) — потому что мы храним ссылку на хвост.

### Удаление — O(1) при наличии ссылки на узел

Удалить узел = перебить ссылку у предыдущего узла, чтобы он "перепрыгнул" через удаляемый. Удалённый узел потом уберёт сборщик мусора.

```
Удалить 20:
Было:   [10] → [20] → [30]
После:  [10] ────────► [30]
         node.next = node.next.next
```

Само удаление O(1), но **найти узел** — O(n), потому что нужно пройти список с начала.

### Поиск по значению — O(n)

Нет индексов — только последовательный перебор с головы:

```
Найти 30:
head → [10] → нет → [20] → нет → [30] → нашли!
```

В худшем случае — весь список. O(n).

> [!warning] Нет настоящих индексов
> LinkedList не поддерживает быстрый доступ по индексу. `get(i)` в LinkedList — это O(n): нужно пройти i узлов с начала. Это принципиальное отличие от ArrayList.

---

## 🔹 Реализация своего LinkedList (Java)

Понимание устройства через код — лучший способ разобраться:

```java
public class LinkedList<T> {

    // Внутренний класс узла
    private static class Node<T> {
        final T value;
        Node<T> next;

        Node(T value) {
            this.value = value;
        }
    }

    private Node<T> head; // ссылка на первый узел

    // Добавить в начало — O(1)
    public void addFirst(T value) {
        Node<T> newNode = new Node<>(value);
        newNode.next = head;   // новый узел смотрит на старую голову
        head = newNode;        // голова теперь новый узел
    }

    // Добавить в конец — O(n) без хранения tail, O(1) с tail
    public void addLast(T value) {
        Node<T> newNode = new Node<>(value);
        if (head == null) {
            head = newNode;    // список был пустой
            return;
        }
        Node<T> current = head;
        while (current.next != null) {
            current = current.next;  // дойти до хвоста
        }
        current.next = newNode;      // прицепить новый узел
    }

    // Найти псевдо-индекс по значению — O(n)
    public int indexOf(T value) {
        if (head == null) return -1;
        if (head.value.equals(value)) return 0;

        Node<T> current = head;
        int index = 0;
        while (current.next != null) {
            index++;
            current = current.next;
            if (current.value.equals(value)) return index;
        }
        return -1; // не найдено
    }

    // Удалить по значению — O(n)
    public void remove(T value) {
        if (head == null) return;

        // удаляем голову
        if (head.value.equals(value)) {
            head = head.next;
            return;
        }

        Node<T> current = head;
        while (current.next != null) {
            if (current.next.value.equals(value)) {
                current.next = current.next.next; // "перепрыгнуть" через удаляемый
                return;
            }
            current = current.next;
        }
    }
}
```

---

## 🔹 LinkedList в Java — java.util.LinkedList

Java предоставляет готовую реализацию — двусвязный список, который реализует и `List`, и `Deque`:

```java
import java.util.LinkedList;

LinkedList<String> list = new LinkedList<>();

// Добавление — O(1)
list.addFirst("A");    // в начало
list.addLast("B");     // в конец
list.add("C");         // тоже в конец

// Доступ к концам — O(1)
list.getFirst();       // "A"
list.getLast();        // "C"
list.peekFirst();      // посмотреть без удаления
list.peekLast();

// Удаление с концов — O(1)
list.removeFirst();    // удалить и вернуть первый
list.removeLast();     // удалить и вернуть последний
list.pollFirst();      // то же, но не бросает исключение если пуст
list.pollLast();

// Как обычный List — но get(i) тут O(n)!
list.get(2);           // O(n) — не используй в цикле
list.size();
list.contains("B");    // O(n)
list.remove("B");      // O(n) — найти + удалить

// Как стек (LIFO)
list.push("X");        // = addFirst
list.pop();            // = removeFirst

// Как очередь (FIFO)
list.offer("X");       // = addLast
list.poll();           // = removeFirst
```

> [!warning] Не вызывай get(i) в цикле у LinkedList
> ```java
> // ❌ O(n²) — катастрофа на больших списках
> for (int i = 0; i < list.size(); i++) {
>     list.get(i); // каждый get — O(n)
> }
>
> // ✅ O(n) — правильно через итератор
> for (String s : list) {
>     // используем s
> }
> ```

---

## 🔹 Сравнение: ArrayList vs LinkedList

| | ArrayList | LinkedList |
|-|-----------|------------|
| Внутренняя структура | `Object[]` массив | Двусвязный список из Node |
| `get(i)` | **O(1)** ✅ | O(n) ❌ |
| `add` в конец | O(1) амортизированно ✅ | **O(1)** ✅ |
| `addFirst` / в начало | O(n) — сдвиг ❌ | **O(1)** ✅ |
| `add(i, e)` в середину | O(n) — сдвиг ❌ | O(n) — поиск позиции ❌ |
| `remove(i)` | O(n) — сдвиг ❌ | O(n) — поиск ❌ |
| Память | Компактнее | Больше (каждый Node — 2 ссылки) |
| Кэш-эффективность | Высокая (элементы рядом) | Низкая (узлы разбросаны) |

> [!tip] Когда что выбрать
> **ArrayList** — почти всегда. Быстрый get(i), компактная память, хорошо работает с кэшем процессора.
>
> **LinkedList** — только если постоянно добавляешь/удаляешь **с концов** (очередь, стек, дек) и `get(i)` не нужен совсем. В остальных случаях ArrayDeque будет лучше даже для этого.

---

## 🔹 Итог

```
LinkedList = цепочка Node { value, next, prev }
             head → Node → Node → Node → null ← tail

Добавление в начало/конец  — O(1)   (перебить ссылку)
Удаление (если есть узел)  — O(1)   (перебить ссылку)
Поиск по значению          — O(n)   (перебор с головы)
get(i) по индексу          — O(n)   (перебор с головы) ← главный минус

Не вызывай get(i) в цикле — получишь O(n²).
Итерируйся только через for-each или Iterator.

Односвязный: next →
Двусвязный:  ← prev | next → (java.util.LinkedList)

На практике: ArrayList почти всегда быстрее из-за кэш-эффективности.
LinkedList полезен как Deque (двусторонняя очередь).
```
