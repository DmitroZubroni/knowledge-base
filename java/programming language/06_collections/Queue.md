> **Теги:** #java #collections #queue #очередь #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Collections_Index]]

# Queue — Очередь

---

## 🔹 Что такое очередь и зачем она нужна

Представь очередь в магазине: первый пришёл — первый обслужен. Именно этот принцип лежит в основе структуры данных Queue.

**FIFO (First In — First Out)**: элемент, добавленный первым, будет извлечён первым.

```
Добавляем: A → B → C

Очередь:  [A | B | C]
           ↑ голова    ↑ хвост
           (poll)      (offer)

Извлекаем: A → B → C  (в том же порядке)
```

Где это нужно на практике:
- Обработка задач в порядке поступления (очередь запросов к серверу)
- BFS (обход графа в ширину)
- Буферизация сообщений (Kafka, RabbitMQ — очереди по сути)
- Пул потоков — задачи ожидают свободного потока в очереди

---

## 🔹 Queue — это интерфейс

`Queue` в Java — интерфейс. У него есть несколько реализаций, но сначала важно понять его API. Методы существуют в двух вариантах — это принципиально важно:

| Действие | Бросает исключение | Возвращает null / false |
|----------|--------------------|------------------------|
| Добавить в хвост | `add(e)` | `offer(e)` |
| Посмотреть голову (без удаления) | `element()` | `peek()` |
| Извлечь голову (с удалением) | `remove()` | `poll()` |

> [!tip] Правило выбора метода
> Если очередь **не должна** быть пустой в этот момент — используй `add/element/remove` (бросят исключение если что-то пошло не так, и ты сразу узнаешь об ошибке).
>
> Если очередь **может** быть пустой и это нормально — используй `offer/peek/poll` (вернут `null` или `false`). Это безопаснее в продакшн-коде.

```java
Queue<String> queue = new LinkedList<>();

// Добавление в хвост
queue.offer("Alice");
queue.offer("Bob");
queue.offer("Charlie");
// Очередь: [Alice, Bob, Charlie]

// Посмотреть голову без удаления
queue.peek();     // "Alice" — очередь не изменилась
queue.element();  // "Alice" — то же, но бросит NoSuchElementException если пусто

// Извлечь голову с удалением
queue.poll();     // "Alice" — удалена из очереди
queue.remove();   // "Bob"   — удалена из очереди
// Очередь: [Charlie]

// Утилиты
queue.size();     // 1
queue.isEmpty();  // false
queue.contains("Charlie"); // true
```

---

## 🔹 Основные реализации Queue

```
Queue<E>
├── LinkedList      — двусвязный список, O(1) offer/poll, универсальный
├── ArrayDeque      — массив, O(1) offer/poll, быстрее LinkedList (рекомендуется)
├── PriorityQueue   — куча, O(log n) offer, O(1) peek, O(log n) poll
└── LinkedBlockingQueue, ArrayBlockingQueue — для многопоточки
```

> [!note] Какую реализацию выбрать для обычной очереди FIFO?
> **ArrayDeque** — быстрее LinkedList из-за лучшей кэш-эффективности (элементы в массиве рядом). LinkedList тратит память на ссылки узлов. Подробнее в [[ArrayDeque]].

---

## 🔹 Типичный паттерн: BFS с очередью

Обход графа / дерева в ширину — классическая задача, где Queue незаменима:

```java
// Обход дерева по уровням (Level Order Traversal)
void bfs(TreeNode root) {
    if (root == null) return;

    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        TreeNode node = queue.poll();         // взять следующий узел
        System.out.println(node.val);

        if (node.left != null) queue.offer(node.left);   // добавить детей
        if (node.right != null) queue.offer(node.right);
    }
}
// Порядок обхода гарантирован: сначала все узлы уровня N, потом уровня N+1
```

---

## 🔹 Итог

```
Queue = FIFO (First In — First Out)
        добавляем в хвост, извлекаем из головы

Два набора методов:
  offer/peek/poll    — безопасные (null если пусто)
  add/element/remove — с исключениями (если пусто → NoSuchElementException)

Лучшая реализация для FIFO: ArrayDeque (быстрее LinkedList)
Для приоритетов: PriorityQueue
Для многопоточки: LinkedBlockingQueue / ArrayBlockingQueue

Главное применение: BFS, пул задач, буферизация
```
