> **Теги:** #algorithms #trees #bst #dfs #bfs #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]]

# 🌳 Деревья (Trees)

---

## 🔹 Основные понятия

```
         [10]          ← root (корень)
        /    \
      [5]    [15]      ← внутренние узлы
     /   \      \
   [3]   [7]   [20]   ← листья (leaf nodes)

Height дерева = 2  (число рёбер от корня до самого глубокого листа)
Depth узла [7] = 2  (число рёбер от корня до узла)
```

| Термин | Описание |
|--------|----------|
| Root | Корень — единственный узел без родителя |
| Leaf | Лист — узел без детей |
| Height | Длина пути от корня до самого глубокого листа |
| Depth | Длина пути от корня до данного узла |
| Subtree | Поддерево — узел и все его потомки |
| Level | Уровень = depth + 1 |

```java
// Универсальный узел бинарного дерева
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```

---

## 🔹 Четыре обхода — главное что нужно знать

Понимание обходов — основа всех задач на деревья. Каждый обход отвечает на свой вопрос.

### In-Order: Левый → Корень → Правый

```java
void inOrder(TreeNode root) {
    if (root == null) return;
    inOrder(root.left);
    process(root.val);   // ← обработка МЕЖДУ детьми
    inOrder(root.right);
}
// Для BST даёт элементы в ОТСОРТИРОВАННОМ порядке — ключевое свойство!
// [8] дерево выше → 1 3 4 6 7 8 10 13 14
```

### Pre-Order: Корень → Левый → Правый

```java
void preOrder(TreeNode root) {
    if (root == null) return;
    process(root.val);   // ← обработка ПЕРВОЙ (до детей)
    preOrder(root.left);
    preOrder(root.right);
}
// Применение: сериализация дерева, копирование структуры
// Корень обрабатывается первым → знаем структуру сверху вниз
```

### Post-Order: Левый → Правый → Корень

```java
void postOrder(TreeNode root) {
    if (root == null) return;
    postOrder(root.left);
    postOrder(root.right);
    process(root.val);   // ← обработка ПОСЛЕДНЕЙ (после детей)
}
// Применение: вычисление высоты, суммы поддерева, удаление дерева
// Сначала обрабатываем детей → знаем результат поддерева прежде чем обработать узел
```

### Level-Order: BFS по уровням

```java
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int size = queue.size(); // ВАЖНО: фиксируем размер ТЕКУЩЕГО уровня
        List<Integer> level = new ArrayList<>();

        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null)  queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
// Применение: минимальная глубина, соединение узлов одного уровня,
//             правый вид дерева (последний узел каждого уровня)
```

> [!tip] Как выбрать обход
> - Нужны данные от детей для вычисления в узле → **Post-Order**
> - Нужно передать данные от родителя к детям → **Pre-Order**
> - Отсортированный порядок из BST → **In-Order**
> - Работа по уровням, минимальное расстояние → **Level-Order (BFS)**

---

## 🔹 Бинарное дерево поиска (BST)

**Свойство BST:** для каждого узла N:
- всё в левом поддереве < N.val
- всё в правом поддереве > N.val

```
       [8]
      /    \
   [3]      [10]
   / \         \
 [1] [6]       [14]
     / \       /
   [4] [7]  [13]
```

### Операции

```java
// Поиск — O(log n) avg, O(n) worst (несбалансированное)
TreeNode search(TreeNode root, int val) {
    if (root == null || root.val == val) return root;
    return val < root.val
        ? search(root.left, val)
        : search(root.right, val);
}

// Вставка — O(log n) avg
TreeNode insert(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    if (val < root.val) root.left  = insert(root.left, val);
    else if (val > root.val) root.right = insert(root.right, val);
    return root; // если val == root.val — дубликат, игнорируем
}

// Минимум / максимум
TreeNode findMin(TreeNode root) {
    while (root.left != null) root = root.left;  // самый левый
    return root;
}
TreeNode findMax(TreeNode root) {
    while (root.right != null) root = root.right; // самый правый
    return root;
}

// Удаление — три случая
TreeNode delete(TreeNode root, int val) {
    if (root == null) return null;

    if (val < root.val) {
        root.left = delete(root.left, val);
    } else if (val > root.val) {
        root.right = delete(root.right, val);
    } else {
        // Нашли узел для удаления
        if (root.left == null) return root.right;  // нет левого → заменить правым
        if (root.right == null) return root.left;  // нет правого → заменить левым

        // Два ребёнка: заменить минимальным из правого поддерева (in-order successor)
        TreeNode successor = findMin(root.right);
        root.val = successor.val;
        root.right = delete(root.right, successor.val);
    }
    return root;
}
```

### Проверка: является ли дерево BST

```java
// ❌ Наивный подход: проверять только root.left.val < root.val
// Не работает: [10, [5], [15, [6]]] — 6 < 10 но это нарушение BST

// ✅ Правильно: передавать допустимый диапазон
boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val) &&   // левое: (min, node.val)
           validate(node.right, node.val, max);    // правое: (node.val, max)
}
```

---

## 🔹 Рекурсивное мышление для задач на деревья

Большинство задач на деревья решаются одним из двух способов:

### Паттерн 1: возвращать результат снизу вверх (Post-Order)

```
"Что мне нужно от детей, чтобы вычислить ответ для текущего узла?"
```

```java
// Высота дерева
int height(TreeNode root) {
    if (root == null) return 0;         // base case
    int left  = height(root.left);      // получаем от левого ребёнка
    int right = height(root.right);     // получаем от правого ребёнка
    return 1 + Math.max(left, right);   // используем для текущего узла
}

// Сумма всех узлов
int sumTree(TreeNode root) {
    if (root == null) return 0;
    return root.val + sumTree(root.left) + sumTree(root.right);
}

// Количество узлов
int countNodes(TreeNode root) {
    if (root == null) return 0;
    return 1 + countNodes(root.left) + countNodes(root.right);
}
```

### Паттерн 2: передавать информацию сверху вниз (Pre-Order)

```
"Что мне нужно передать детям из текущего контекста?"
```

```java
// Сумма пути от корня до каждого листа
void pathSum(TreeNode root, int currentSum, List<Integer> leafSums) {
    if (root == null) return;
    currentSum += root.val;                          // передаём накопленную сумму вниз
    if (root.left == null && root.right == null) {   // лист
        leafSums.add(currentSum);
        return;
    }
    pathSum(root.left, currentSum, leafSums);
    pathSum(root.right, currentSum, leafSums);
}
```

### Паттерн 3: глобальная переменная для "пересекающих" путей

Некоторые пути проходят через узел, захватывая оба поддерева. Для таких задач удобна глобальная переменная.

```java
// Диаметр дерева (наибольший путь между двумя узлами)
int maxDiameter = 0;

int diameter(TreeNode root) {
    dfs(root);
    return maxDiameter;
}

int dfs(TreeNode node) {
    if (node == null) return 0;
    int left  = dfs(node.left);
    int right = dfs(node.right);
    maxDiameter = Math.max(maxDiameter, left + right); // путь через текущий узел
    return 1 + Math.max(left, right);                  // возвращаем глубину для родителя
}
```

---

## 🔹 Классические задачи — разбор

### Наименьший общий предок (LCA)

```java
// LCA(p, q) — самый глубокий узел, который является предком обоих
// Идея: если p и q в разных поддеревьях — текущий узел и есть LCA
TreeNode lca(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null) return null;
    if (root == p || root == q) return root; // нашли одного из искомых

    TreeNode left  = lca(root.left, p, q);
    TreeNode right = lca(root.right, p, q);

    if (left != null && right != null) return root; // p и q по разные стороны
    return left != null ? left : right;             // оба в одном поддереве
}
```

### Симметричное дерево

```java
// Является ли дерево зеркально симметричным?
boolean isSymmetric(TreeNode root) {
    return isMirror(root.left, root.right);
}

boolean isMirror(TreeNode left, TreeNode right) {
    if (left == null && right == null) return true;
    if (left == null || right == null) return false;
    return left.val == right.val
        && isMirror(left.left, right.right)   // внешние
        && isMirror(left.right, right.left);  // внутренние
}
```

### Максимальная сумма пути (Max Path Sum)

```java
// Путь может идти через любые два узла (не обязательно через корень)
int maxPathSum = Integer.MIN_VALUE;

int maxPathSum(TreeNode root) {
    dfs(root);
    return maxPathSum;
}

int dfs(TreeNode node) {
    if (node == null) return 0;
    // max(0, ...) — если поддерево даёт отрицательный вклад, не берём его
    int left  = Math.max(0, dfs(node.left));
    int right = Math.max(0, dfs(node.right));

    // Обновляем глобальный максимум (путь через текущий узел)
    maxPathSum = Math.max(maxPathSum, node.val + left + right);

    // Возвращаем родителю максимальную "ветку" (только одна сторона)
    return node.val + Math.max(left, right);
}
```

### Сериализация и десериализация дерева

```java
// Pre-Order с маркером null
String serialize(TreeNode root) {
    if (root == null) return "null,";
    return root.val + "," + serialize(root.left) + serialize(root.right);
}

TreeNode deserialize(String data) {
    Queue<String> nodes = new ArrayDeque<>(Arrays.asList(data.split(",")));
    return buildTree(nodes);
}

TreeNode buildTree(Queue<String> nodes) {
    String val = nodes.poll();
    if (val.equals("null")) return null;
    TreeNode node = new TreeNode(Integer.parseInt(val));
    node.left  = buildTree(nodes);
    node.right = buildTree(nodes);
    return node;
}
```

### K-й наименьший в BST

```java
// In-Order даёт отсортированный порядок → k-й элемент
int kthSmallest(TreeNode root, int k) {
    int[] count = {0};
    int[] result = {0};
    inOrderKth(root, k, count, result);
    return result[0];
}

void inOrderKth(TreeNode node, int k, int[] count, int[] result) {
    if (node == null) return;
    inOrderKth(node.left, k, count, result);
    if (++count[0] == k) { result[0] = node.val; return; }
    inOrderKth(node.right, k, count, result);
}
```

### Правый вид дерева

```java
// Последний узел каждого уровня при BFS
List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            if (i == size - 1) result.add(node.val); // последний на уровне
            if (node.left != null)  queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }
    return result;
}
```

---

## 🔹 Сбалансированные деревья

**Проблема несбалансированного BST:** при вставке уже отсортированных данных дерево деградирует в список — все операции O(n) вместо O(log n).

```
Вставка 1,2,3,4,5 в обычное BST:   Сбалансированное AVL:
[1]                                       [3]
  \                                      /   \
  [2]                                  [2]   [4]
    \                                  /       \
    [3]   ← список, O(n) поиск       [1]       [5]
      \
      [4]
        \
        [5]
```

| Тип | Балансировка | Операции | В Java |
|-----|-------------|----------|--------|
| BST | Нет | O(log n) avg, O(n) worst | — |
| AVL | Повороты при дисбалансе | O(log n) гарантированно | — |
| Red-Black Tree | Раскраска + повороты | O(log n) гарантированно | TreeMap, TreeSet |
| B-Tree | Для дисков | O(log n) | Индексы БД |

```java
// TreeMap / TreeSet — Red-Black Tree под капотом
TreeMap<Integer, String> map = new TreeMap<>();
map.firstKey() / map.lastKey()         // min/max — O(log n)
map.floorKey(4) / map.ceilingKey(4)    // ближайший ≤4 / ≥4
map.headMap(5) / map.tailMap(5)        // всё <5 / всё ≥5
map.subMap(3, 7)                       // [3, 7)
map.pollFirstEntry() / pollLastEntry() // удалить и вернуть крайний
```

---

## 🔹 Iterative DFS — через явный стек

Рекурсия может дать StackOverflow на глубоких деревьях (~10000+ узлов). Итеративный вариант через явный стек:

```java
// Iterative Pre-Order
List<Integer> preOrderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.add(node.val);
        // Правый первым — чтобы левый обработался раньше
        if (node.right != null) stack.push(node.right);
        if (node.left  != null) stack.push(node.left);
    }
    return result;
}

// Iterative In-Order
List<Integer> inOrderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;

    while (curr != null || !stack.isEmpty()) {
        while (curr != null) {       // идём максимально влево
            stack.push(curr);
            curr = curr.left;
        }
        curr = stack.pop();          // обрабатываем
        result.add(curr.val);
        curr = curr.right;           // переходим вправо
    }
    return result;
}
```

---

## 🔹 Шаблоны решения задач

```
Задача о высоте / глубине         → Post-Order DFS
Задача о путях от корня           → Pre-Order DFS с накопленным состоянием
Задача о диаметре / макс пути     → Post-Order + глобальная переменная
Задача по уровням                 → BFS (Level-Order)
Минимальная глубина               → BFS (останавливается на первом листе)
Симметрия / зеркало               → рекурсия с двумя указателями
Задача на BST (k-й, диапазон)     → In-Order обход
Задача на LCA                     → Post-Order, возвращаем найденный узел
Проверка BST                      → передача диапазона (min, max)
Сериализация                      → Pre-Order с маркерами null
```

---

## 🔹 Итог

```
Четыре обхода:
  In-Order    L→N→R  → отсортированный порядок из BST
  Pre-Order   N→L→R  → сериализация, копирование
  Post-Order  L→R→N  → высота, сумма, удаление (дети раньше родителя)
  Level-Order BFS    → работа по уровням, минимальная глубина

BST: поиск/вставка/удаление O(log n) avg, O(n) worst (несбалансированное)
Валидация BST — передавай диапазон (min, max), не просто сравнивай соседей

Два паттерна рекурсии:
  Снизу вверх (Post-Order): получаем от детей → вычисляем для узла
  Сверху вниз (Pre-Order):  передаём детям накопленное состояние

Глобальная переменная — для путей через узел (диаметр, max path sum)

Red-Black Tree → TreeMap/TreeSet в Java: все операции O(log n) гарантированно
Iterative DFS через Deque → когда рекурсия даёт StackOverflow
```
