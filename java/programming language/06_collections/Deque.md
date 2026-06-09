> **Теги:** #java #collections #deque #двусторонняя-очередь #стек #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Collections_Index]]

# Deque — Двусторонняя очередь

---

## 🔹 Что такое Deque и чем он лучше Queue

Queue — очередь с одним входом и одним выходом. Но часто нужно добавлять и извлекать элементы **с обоих концов**. Именно для этого существует Deque (Double-Ended Queue — двусторонняя очередь).

```
Queue:   добавляем → [A | B | C] → извлекаем
                      хвост  голова

Deque:  ← добавляем/извлекаем → [A | B | C] ← добавляем/извлекаем →
           с обоих концов                        с обоих концов
```

Deque — это **три структуры в одном**:
- **Очередь FIFO** — добавляй в хвост, извлекай из головы
- **Стек LIFO** — добавляй и извлекай из одного конца
- **Двусторонняя очередь** — работай с обоими концами

> [!note] Java рекомендует Deque вместо Stack
> Класс `Stack` в Java устарел. Официальная документация говорит: используй `Deque` и его реализации. `ArrayDeque` как стек работает быстрее `Stack`, который синхронизирован и создаёт лишние накладные расходы.

---

## 🔹 API — методы для обоих концов

Как и Queue, Deque имеет два набора методов для каждой операции:

**Работа с головой (head / first):**

| Действие | С исключением | Без исключения |
|----------|---------------|----------------|
| Добавить в голову | `addFirst(e)` | `offerFirst(e)` |
| Посмотреть голову | `getFirst()` | `peekFirst()` |
| Извлечь голову | `removeFirst()` | `pollFirst()` |

**Работа с хвостом (tail / last):**

| Действие | С исключением | Без исключения |
|----------|---------------|----------------|
| Добавить в хвост | `addLast(e)` | `offerLast(e)` |
| Посмотреть хвост | `getLast()` | `peekLast()` |
| Извлечь хвост | `removeLast()` | `pollLast()` |

**Псевдонимы для использования как Queue и Stack:**

```java
// Как Queue (FIFO):
deque.offer(e)  == deque.offerLast(e)   // добавить в хвост
deque.poll()    == deque.pollFirst()    // извлечь из головы
deque.peek()    == deque.peekFirst()    // посмотреть голову

// Как Stack (LIFO):
deque.push(e)   == deque.addFirst(e)    // положить на вершину стека
deque.pop()     == deque.removeFirst()  // снять с вершины стека
deque.peek()    == deque.peekFirst()    // посмотреть вершину
```

---

## 🔹 Deque как стек (LIFO)

Стек — структура "последний пришёл, первый ушёл". Элементы кладутся и снимаются с одного конца (вершины).

```
push(A) → push(B) → push(C)

Стек:  [C]  ← вершина (top)
       [B]
       [A]

pop() → C
pop() → B
pop() → A
```

```java
Deque<String> stack = new ArrayDeque<>();

// Кладём на вершину
stack.push("A");
stack.push("B");
stack.push("C");
// Стек (сверху вниз): C, B, A

// Смотрим вершину без удаления
stack.peek(); // "C"

// Снимаем с вершины
stack.pop();  // "C"
stack.pop();  // "B"
stack.pop();  // "A"
```

---

## 🔹 Deque как двусторонняя очередь

```java
Deque<String> deque = new ArrayDeque<>();

deque.offerLast("B");   // [B]
deque.offerLast("C");   // [B, C]
deque.offerFirst("A");  // [A, B, C]

deque.peekFirst();  // "A"
deque.peekLast();   // "C"

deque.pollFirst();  // "A" → [B, C]
deque.pollLast();   // "C" → [B]
```

---

## 🔹 Типичные задачи

### Проверка скобок — классика со стеком

```java
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();

    for (char c : s.toCharArray()) {
        if (c == '(' || c == '[' || c == '{') {
            stack.push(c);  // открывающую — кладём
        } else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();  // снимаем последнюю открывающую
            if (c == ')' && top != '(') return false;
            if (c == ']' && top != '[') return false;
            if (c == '}' && top != '{') return false;
        }
    }
    return stack.isEmpty();
}
// "()[]{}"  → true
// "([)]"    → false
```

### Скользящее максимальное окно — Deque как монотонная очередь

```java
// Найти максимум в каждом окне размера k
int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>(); // хранит индексы
    int[] result = new int[nums.length - k + 1];

    for (int i = 0; i < nums.length; i++) {
        // Убираем элементы вне окна
        if (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        // Убираем элементы меньше текущего (они никогда не будут максимумом)
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        deque.offerLast(i);
        // Записываем максимум окна
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    return result;
}
```

### История браузера — Deque как история навигации

```java
Deque<String> history = new ArrayDeque<>();
history.push("google.com");
history.push("github.com");
history.push("stackoverflow.com");

// Нажали "назад"
history.pop();        // убрали текущую страницу
history.peek();       // "github.com" — теперь текущая
```

---

## 🔹 Итог

```
Deque = Double-Ended Queue = добавление/извлечение с обоих концов

Три роли в одном:
  FIFO очередь  → offer/poll (хвост/голова)
  LIFO стек     → push/pop   (голова/голова)
  Дек           → offerFirst/offerLast/pollFirst/pollLast

Методы:
  С исключением:   addFirst/addLast, getFirst/getLast, removeFirst/removeLast
  Без исключения:  offerFirst/offerLast, peekFirst/peekLast, pollFirst/pollLast
  Стек-псевдонимы: push = addFirst, pop = removeFirst

Реализации: ArrayDeque (рекомендуется), LinkedList
Не используй устаревший класс Stack — используй Deque

Типичные задачи: проверка скобок, монотонная очередь, история навигации
```
