> **Теги:** #algorithms #stack #queue #deque #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[00_Algorithms]]

# 📚 Stack, Queue, Deque

---

## 🔹 Stack (Стек)

> [!note] Определение
> **Stack** — структура данных LIFO (Last In, First Out). Последний добавленный элемент — первый извлечённый. Как стопка тарелок.

```
push(1) → push(2) → push(3)

[3]  ← top
[2]
[1]

pop() → 3
pop() → 2
```

### Stack в Java
```java
import java.util.Deque;
import java.util.ArrayDeque;

// ✅ Современный способ — через Deque
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);       // добавить на вершину — O(1)
stack.push(2);
stack.push(3);
stack.pop();         // извлечь с вершины — O(1) → 3
stack.peek();        // посмотреть вершину без удаления — O(1) → 2
stack.isEmpty();     // проверка на пустоту
stack.size();
```

> [!warning] Класс Stack устарел
> `java.util.Stack` — устаревший класс, наследник `Vector`, потокобезопасный но медленный. Используй `ArrayDeque` как стек.

### Применение Stack
- Проверка скобочной последовательности
- Обход дерева/графа (DFS итеративно)
- Отмена операций (Ctrl+Z)
- Вычисление выражений (RPN)
- История браузера (назад)

### Классическая задача: валидация скобок
```java
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();
            if (c == ')' && top != '(') return false;
            if (c == ']' && top != '[') return false;
            if (c == '}' && top != '{') return false;
        }
    }
    return stack.isEmpty();
}
```

---

## 🔹 Queue (Очередь)

> [!note] Определение
> **Queue** — структура данных FIFO (First In, First Out). Первый добавленный — первый извлечённый. Как очередь в кассу.

```
offer(1) → offer(2) → offer(3)

front → [1][2][3] ← back

poll() → 1
poll() → 2
```

### Queue в Java
```java
import java.util.Queue;
import java.util.ArrayDeque;
import java.util.LinkedList;

Queue<Integer> queue = new ArrayDeque<>(); // ✅ предпочтительно
// Queue<Integer> queue = new LinkedList<>(); — тоже работает

queue.offer(1);      // добавить в конец — O(1)
queue.offer(2);
queue.offer(3);
queue.poll();        // извлечь из начала — O(1) → 1
queue.peek();        // посмотреть первый — O(1) → 2
queue.isEmpty();
queue.size();
```

### Применение Queue
- BFS (обход графа/дерева в ширину)
- Очередь задач / планировщик
- Буфер между потоками (BlockingQueue)
- Кэш с вытеснением (LRU — через LinkedHashMap)

---

## 🔹 Deque (Двусторонняя очередь)

> [!note] Определение
> **Deque** (Double Ended Queue) — можно добавлять и удалять с **обоих концов** за O(1). Обобщение Stack и Queue.

```java
Deque<Integer> deque = new ArrayDeque<>();

// Добавление
deque.addFirst(1);   // в начало
deque.addLast(2);    // в конец
deque.offerFirst(0); // в начало (возвращает false вместо исключения)
deque.offerLast(3);  // в конец

// Извлечение
deque.pollFirst();   // из начала
deque.pollLast();    // из конца
deque.peekFirst();   // посмотреть начало
deque.peekLast();    // посмотреть конец
```

| Метод | Начало | Конец |
|-------|--------|-------|
| Добавить | addFirst / offerFirst | addLast / offerLast |
| Удалить | pollFirst / removeFirst | pollLast / removeLast |
| Просмотр | peekFirst / getFirst | peekLast / getLast |

---

## 🔹 PriorityQueue (Приоритетная очередь)

> [!note] Определение
> **PriorityQueue** — очередь, где элементы извлекаются в порядке **приоритета** (по умолчанию — минимальный первый). Реализована через бинарную кучу (heap).

```java
import java.util.PriorityQueue;
import java.util.Collections;

// Min-heap (по умолчанию) — первым выходит минимум
PriorityQueue<Integer> minPQ = new PriorityQueue<>();
minPQ.offer(5);
minPQ.offer(1);
minPQ.offer(3);
minPQ.poll(); // → 1 (минимальный)

// Max-heap — первым выходит максимум
PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Collections.reverseOrder());
maxPQ.offer(5);
maxPQ.offer(1);
maxPQ.poll(); // → 5 (максимальный)

// Кастомный компаратор
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
```

| Операция | Сложность |
|----------|-----------|
| offer (add) | O(log n) |
| poll (remove min) | O(log n) |
| peek (min) | O(1) |

### Применение PriorityQueue
- Алгоритм Дейкстры (кратчайший путь)
- K наибольших/наименьших элементов
- Merge K sorted arrays
- Планировщик задач по приоритету

```java
// Пример: K наименьших элементов
int[] findKSmallest(int[] arr, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>();
    for (int x : arr) pq.offer(x);
    int[] result = new int[k];
    for (int i = 0; i < k; i++) result[i] = pq.poll();
    return result;
}
// O(n log n) — можно оптимизировать до O(n log k) через max-heap размера k
```

---

## 🔹 Реализация Queue через два Stack

```java
class MyQueue {
    Deque<Integer> in = new ArrayDeque<>();
    Deque<Integer> out = new ArrayDeque<>();

    void push(int x) {
        in.push(x);
    }

    int pop() {
        if (out.isEmpty()) {
            while (!in.isEmpty()) out.push(in.pop());
        }
        return out.pop();
    }

    int peek() {
        if (out.isEmpty()) {
            while (!in.isEmpty()) out.push(in.pop());
        }
        return out.peek();
    }
}
// Амортизированно O(1) на операцию
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Stack** → `Deque<T> stack = new ArrayDeque<>()`, методы: push/pop/peek
> - **Queue** → `Queue<T> queue = new ArrayDeque<>()`, методы: offer/poll/peek
> - **Deque** → универсально, оба конца за O(1)
> - **PriorityQueue** — heap, O(log n) вставка, O(1) peek минимума
> - Старый `Stack` и `LinkedList` — не используй без причины
> - BFS всегда через Queue, DFS — через Stack или рекурсию
