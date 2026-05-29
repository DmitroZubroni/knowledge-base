> **Теги:** #algorithms #linkedlist #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[main Algorithms]]

# 🔗 Связанные списки (Linked Lists)

---

## 🔹 Что такое связанный список

> [!note] Определение
> **Связанный список** — структура данных из **узлов** (Node), где каждый узел хранит значение и **ссылку** на следующий узел. Элементы НЕ расположены в памяти последовательно.

```
[10 | *] → [20 | *] → [30 | *] → [40 | null]
 head                              tail
```

---

## 🔹 Односвязный список (Singly LinkedList)

### Структура узла
```java
class Node {
    int val;
    Node next;

    Node(int val) {
        this.val = val;
        this.next = null;
    }
}
```

### Реализация
```java
class LinkedList {
    Node head;

    // Добавить в начало — O(1)
    void addFirst(int val) {
        Node node = new Node(val);
        node.next = head;
        head = node;
    }

    // Добавить в конец — O(n)
    void addLast(int val) {
        Node node = new Node(val);
        if (head == null) { head = node; return; }
        Node cur = head;
        while (cur.next != null) cur = cur.next;
        cur.next = node;
    }

    // Удалить первый — O(1)
    void removeFirst() {
        if (head != null) head = head.next;
    }

    // Поиск — O(n)
    Node find(int val) {
        Node cur = head;
        while (cur != null) {
            if (cur.val == val) return cur;
            cur = cur.next;
        }
        return null;
    }

    // Вывод списка
    void print() {
        Node cur = head;
        while (cur != null) {
            System.out.print(cur.val + " → ");
            cur = cur.next;
        }
        System.out.println("null");
    }
}
```

---

## 🔹 Двусвязный список (Doubly LinkedList)

```java
class DNode {
    int val;
    DNode prev, next;

    DNode(int val) { this.val = val; }
}

class DoublyLinkedList {
    DNode head, tail;

    // Добавить в конец — O(1) благодаря tail
    void addLast(int val) {
        DNode node = new DNode(val);
        if (tail == null) { head = tail = node; return; }
        tail.next = node;
        node.prev = tail;
        tail = node;
    }

    // Удалить из конца — O(1)
    void removeLast() {
        if (tail == null) return;
        if (head == tail) { head = tail = null; return; }
        tail = tail.prev;
        tail.next = null;
    }
}
```

```
null ← [10] ⇄ [20] ⇄ [30] ⇄ [40] → null
       head                   tail
```

---

## 🔹 Сравнение: Array vs LinkedList

| Операция | Array (ArrayList) | LinkedList |
|----------|-------------------|------------|
| get(i) | O(1) | O(n) |
| add в конец | O(1)* | O(1) двусвязный |
| add в начало | O(n) | O(1) |
| add в середину | O(n) | O(n) поиск + O(1) вставка |
| delete | O(n) | O(n) поиск + O(1) удаление |
| Память | компактно | +8-16 байт на узел (ссылки) |

---

## 🔹 LinkedList в Java

```java
import java.util.LinkedList;
import java.util.Deque;

LinkedList<Integer> list = new LinkedList<>();

// Работа с началом/концом — O(1)
list.addFirst(1);
list.addLast(2);
list.removeFirst();
list.removeLast();
list.getFirst();
list.getLast();

// Как обычный List — get(i) это O(n)!
list.add(5);
list.get(2);     // ❌ медленно — O(n)
list.size();
```

> [!warning] list.get(i) в LinkedList
> `LinkedList.get(i)` — это O(n), потому что приходится идти от head до i-го элемента. Не используй LinkedList там, где нужен частый random access.

---

## 🔹 Классические задачи на LinkedList

### Разворот списка (in-place)
```java
Node reverse(Node head) {
    Node prev = null, curr = head;
    while (curr != null) {
        Node next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev; // новый head
}
// O(n) time, O(1) space
```

### Обнаружение цикла (алгоритм Флойда)
```java
boolean hasCycle(Node head) {
    Node slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true; // встретились — цикл есть
    }
    return false;
}
// Медленный указатель: +1, быстрый: +2
// O(n) time, O(1) space
```

### Нахождение середины списка
```java
Node findMiddle(Node head) {
    Node slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow; // slow — середина
}
```

### Удаление n-го с конца
```java
Node removeNthFromEnd(Node head, int n) {
    Node dummy = new Node(0);
    dummy.next = head;
    Node fast = dummy, slow = dummy;

    for (int i = 0; i <= n; i++) fast = fast.next;

    while (fast != null) {
        slow = slow.next;
        fast = fast.next;
    }
    slow.next = slow.next.next;
    return dummy.next;
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Односвязный** — ссылка только вперёд, O(1) добавление в начало
> - **Двусвязный** — ссылки вперёд и назад, O(1) добавление/удаление с обоих концов
> - `LinkedList.get(i)` — O(n), не использовать для random access
> - **Два указателя** (медленный/быстрый) — стандартный приём для цикла, середины, n-го с конца
> - Разворот списка — классика собеседований, O(n) time, O(1) space
