> **Теги:** #java #collections #priorityqueue #куча #heap #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Collections_Index]]

# PriorityQueue — Очередь с приоритетом

---

## 🔹 Проблема обычной очереди

Обычная Queue работает по FIFO — кто первый пришёл, тот первый обслужен. Но что если задачи имеют разный приоритет? Например, критическая ошибка сервера должна быть обработана раньше информационного лога, даже если лог пришёл раньше.

PriorityQueue решает это: элементы извлекаются не в порядке добавления, а **в порядке приоритета** — наименьший элемент всегда идёт первым.

```
Обычная Queue:  добавили [3, 1, 4, 1, 5] → извлекаем [3, 1, 4, 1, 5]
PriorityQueue:  добавили [3, 1, 4, 1, 5] → извлекаем [1, 1, 3, 4, 5]
```

---

## 🔹 Как устроена PriorityQueue внутри — Binary Heap

Внутри PriorityQueue — **бинарная куча (binary heap)**. Это особое бинарное дерево, хранящееся в обычном массиве.

По умолчанию — **min-heap**: наименьший элемент всегда на вершине.

```
Дерево:         1
               / \
              3   4
             / \ /
            5  3 5

В массиве: [1, 3, 4, 5, 3, 5]
            ↑ вершина (peek = 1)

Свойство кучи: родитель ≤ обоих детей (для min-heap)
Связь индексов: родитель i → дети 2i+1 и 2i+2
```

Ключевые операции:
- **offer(e)** — добавить в конец массива, затем "всплыть" (sift up) на нужную позицию → O(log n)
- **poll()** — извлечь вершину, поставить последний элемент на место вершины, "утопить" (sift down) → O(log n)
- **peek()** — просто вернуть первый элемент массива → O(1)

> [!note] Важное свойство
> PriorityQueue **не хранит элементы в полностью отсортированном порядке**. Гарантируется только одно: `peek()` и `poll()` всегда вернут минимальный элемент. Если итерировать PriorityQueue через for-each — порядок будет произвольным.

---

## 🔹 Min-heap по умолчанию

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();

pq.offer(5);
pq.offer(1);
pq.offer(3);
pq.offer(2);
pq.offer(4);

pq.peek();   // 1 — минимум, без удаления

// Извлекаем в порядке возрастания
while (!pq.isEmpty()) {
    System.out.print(pq.poll() + " "); // 1 2 3 4 5
}

// Размер и проверка
pq.size();
pq.isEmpty();
pq.contains(3); // O(n) — линейный поиск!
```

---

## 🔹 Max-heap — через Comparator

По умолчанию — min-heap. Для max-heap передаём `Comparator.reverseOrder()`:

```java
PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Comparator.reverseOrder());

maxPQ.offer(5);
maxPQ.offer(1);
maxPQ.offer(3);

maxPQ.peek();  // 5 — максимум наверху

while (!maxPQ.isEmpty()) {
    System.out.print(maxPQ.poll() + " "); // 5 3 1
}
```

---

## 🔹 PriorityQueue с объектами — свой Comparator

```java
class Task {
    String name;
    int priority; // чем меньше число — тем выше приоритет

    Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }
}

// Сортировка по полю priority
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(t -> t.priority)
);

taskQueue.offer(new Task("Low priority log", 3));
taskQueue.offer(new Task("Critical error", 1));
taskQueue.offer(new Task("Warning", 2));

// Извлекаем в порядке приоритета
while (!taskQueue.isEmpty()) {
    Task t = taskQueue.poll();
    System.out.println(t.priority + ": " + t.name);
}
// 1: Critical error
// 2: Warning
// 3: Low priority log
```

---

## 🔹 Сложность операций

| Операция | Сложность | Почему |
|----------|-----------|--------|
| `offer(e)` | **O(log n)** | Добавить в конец + sift up по высоте дерева |
| `poll()` | **O(log n)** | Извлечь вершину + sift down по высоте дерева |
| `peek()` | **O(1)** | Просто первый элемент массива |
| `contains(e)` | O(n) | Линейный поиск, нет индексации |
| `remove(e)` | O(n) | Найти + удалить + перебалансировать |
| Создание из коллекции | O(n) | heapify — специальный алгоритм |

---

## 🔹 Типичные задачи

### K наибольших элементов

```java
// Идея: держим min-heap размера k.
// Если новый элемент больше минимума кучи — заменяем.
// В конце в куче остаются k наибольших.
int[] kLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(); // min-heap

    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // удаляем наименьший
        }
    }
    // В куче k наибольших элементов
    return minHeap.stream().mapToInt(i -> i).toArray();
}
// Сложность: O(n log k) — гораздо лучше O(n log n) сортировки
```

### K-й наибольший элемент

```java
int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();
    }
    return minHeap.peek(); // минимум кучи = k-й наибольший
}
```

### Слияние K отсортированных списков

```java
// Держим в куче по одному элементу из каждого списка
// Всегда берём наименьший и добавляем следующий из того же списка
ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq = new PriorityQueue<>(
        Comparator.comparingInt(n -> n.val)
    );

    for (ListNode node : lists) {
        if (node != null) pq.offer(node);
    }

    ListNode dummy = new ListNode(0);
    ListNode cur = dummy;

    while (!pq.isEmpty()) {
        ListNode node = pq.poll();
        cur.next = node;
        cur = cur.next;
        if (node.next != null) pq.offer(node.next);
    }
    return dummy.next;
}
// O(n log k), где n — всего элементов, k — количество списков
```

### Алгоритм Дейкстры (кратчайший путь)

```java
// PriorityQueue хранит пары [расстояние, вершина]
// Всегда обрабатываем вершину с минимальным расстоянием
int[] dijkstra(int[][] graph, int src) {
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;

    PriorityQueue<int[]> pq = new PriorityQueue<>(
        Comparator.comparingInt(a -> a[0]) // сортируем по расстоянию
    );
    pq.offer(new int[]{0, src}); // [расстояние, вершина]

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];

        if (d > dist[u]) continue; // устаревшая запись

        for (int v = 0; v < n; v++) {
            if (graph[u][v] > 0) {
                int newDist = dist[u] + graph[u][v];
                if (newDist < dist[v]) {
                    dist[v] = newDist;
                    pq.offer(new int[]{newDist, v});
                }
            }
        }
    }
    return dist;
}
```

---

## 🔹 PriorityQueue vs TreeMap для приоритетов

Оба хранят отсортированные элементы, но для разных задач:

| | PriorityQueue | TreeMap |
|-|---------------|---------|
| Основная операция | Быстро извлечь минимум/максимум | Быстро найти по ключу |
| `poll()` минимума | **O(log n)** | O(log n) |
| `peek()` минимума | **O(1)** | O(log n) |
| Дубликаты | ✅ допускает | ❌ ключи уникальны |
| Произвольный доступ | ❌ | ✅ |
| Когда использовать | Нужно быстро брать min/max | Нужен поиск по ключу + сортировка |

---

## 🔹 Итог

```
PriorityQueue = бинарная куча (binary heap) в массиве
                min-heap по умолчанию (наименьший элемент первый)

Сложности:
  offer(e)  — O(log n)  sift up
  poll()    — O(log n)  sift down
  peek()    — O(1)      первый элемент массива
  contains  — O(n)      линейный поиск

Max-heap: new PriorityQueue<>(Comparator.reverseOrder())
Свой порядок: new PriorityQueue<>(Comparator.comparingInt(...))

Итерация через for-each НЕ гарантирует порядок!
Для получения в порядке приоритета — только poll() в цикле.

Главные применения:
  - k наибольших / наименьших элементов
  - k-й наибольший элемент
  - Слияние k отсортированных списков
  - Алгоритм Дейкстры (кратчайший путь)
  - Планировщики задач с приоритетами
```
