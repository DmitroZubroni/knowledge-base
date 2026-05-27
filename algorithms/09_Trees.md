> **Теги:** #algorithms #trees #bst #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[00_Algorithms]]

# 🌳 Деревья (Trees)

---

## 🔹 Основные понятия

> [!note] Дерево
> **Дерево** — иерархическая структура данных из **узлов** (nodes), где каждый узел имеет ноль или более **дочерних** узлов и ровно одного **родителя** (кроме корня).

```
         [10]          ← root (корень)
        /    \
      [5]    [15]      ← внутренние узлы
     /   \      \
   [3]   [7]   [20]   ← листья (leaf)

Высота дерева (height) = 2 (число рёбер от корня до листа)
Глубина узла (depth)   = число рёбер от корня до узла
```

| Термин | Описание |
|--------|----------|
| Root | Корневой узел (нет родителя) |
| Leaf | Листовой узел (нет детей) |
| Height | Длина самого длинного пути от корня до листа |
| Depth | Длина пути от корня до данного узла |
| Subtree | Поддерево — узел и все его потомки |

---

## 🔹 Бинарное дерево (Binary Tree)

Каждый узел имеет **не более двух** дочерних узлов: left и right.

```java
class TreeNode {
    int val;
    TreeNode left, right;

    TreeNode(int val) { this.val = val; }
}
```

---

## 🔹 Бинарное дерево поиска (BST)

> [!note] BST Свойство
> Для каждого узла:
> - все значения в **левом** поддереве < значения узла
> - все значения в **правом** поддереве > значения узла

```
       [8]
      /    \
   [3]      [10]
   / \         \
 [1] [6]       [14]
     / \       /
   [4] [7]  [13]
```

### Операции BST
```java
// Вставка — O(log n) avg, O(n) worst
TreeNode insert(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    if (val < root.val) root.left = insert(root.left, val);
    else if (val > root.val) root.right = insert(root.right, val);
    return root;
}

// Поиск — O(log n) avg
TreeNode search(TreeNode root, int val) {
    if (root == null || root.val == val) return root;
    if (val < root.val) return search(root.left, val);
    return search(root.right, val);
}

// Минимальный элемент
TreeNode findMin(TreeNode root) {
    while (root.left != null) root = root.left;
    return root;
}
```

---

## 🔹 Обходы дерева (Tree Traversals)

### In-Order (Левый → Корень → Правый)
```java
// Для BST даёт элементы в отсортированном порядке!
void inOrder(TreeNode root) {
    if (root == null) return;
    inOrder(root.left);
    System.out.print(root.val + " ");
    inOrder(root.right);
}
// [8] → 1 3 4 6 7 8 10 13 14
```

### Pre-Order (Корень → Левый → Правый)
```java
void preOrder(TreeNode root) {
    if (root == null) return;
    System.out.print(root.val + " ");
    preOrder(root.left);
    preOrder(root.right);
}
// Используется для копирования/сериализации дерева
```

### Post-Order (Левый → Правый → Корень)
```java
void postOrder(TreeNode root) {
    if (root == null) return;
    postOrder(root.left);
    postOrder(root.right);
    System.out.print(root.val + " ");
}
// Используется для удаления дерева, вычисления размера
```

### Level-Order (BFS по уровням)
```java
void levelOrder(TreeNode root) {
    if (root == null) return;
    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int size = queue.size(); // размер текущего уровня
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            System.out.print(node.val + " ");
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        System.out.println(); // переход на следующий уровень
    }
}
```

---

## 🔹 Классические задачи на деревья

### Высота дерева
```java
int height(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(height(root.left), height(root.right));
}
// O(n) — посещаем каждый узел
```

### Проверка сбалансированности
```java
// Сбалансированное: высота левого и правого поддерева отличаются не более чем на 1
boolean isBalanced(TreeNode root) {
    return checkHeight(root) != -1;
}

int checkHeight(TreeNode root) {
    if (root == null) return 0;
    int left = checkHeight(root.left);
    int right = checkHeight(root.right);
    if (left == -1 || right == -1) return -1; // уже не сбалансировано
    if (Math.abs(left - right) > 1) return -1;
    return 1 + Math.max(left, right);
}
```

### Наименьший общий предок (LCA)
```java
TreeNode lca(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lca(root.left, p, q);
    TreeNode right = lca(root.right, p, q);
    if (left != null && right != null) return root; // p и q в разных поддеревьях
    return left != null ? left : right;
}
```

### Проверка: является ли дерево BST
```java
boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val) &&
           validate(node.right, node.val, max);
}
```

---

## 🔹 Сбалансированные деревья

> [!note] Проблема несбалансированного BST
> В худшем случае BST деградирует в связный список (все элементы вставлены по возрастанию) → O(n) на все операции.

| Тип | Балансировка | Операции |
|-----|-------------|----------|
| BST | нет | O(log n) avg, O(n) worst |
| AVL | автоматическая (повороты) | O(log n) гарантированно |
| Red-Black Tree | автоматическая | O(log n) гарантированно |

### TreeMap / TreeSet в Java — Red-Black Tree
```java
TreeMap<Integer, String> treeMap = new TreeMap<>();
treeMap.put(5, "five");
treeMap.put(3, "three");
treeMap.put(7, "seven");

treeMap.firstKey();                // 3 — O(log n)
treeMap.lastKey();                 // 7
treeMap.floorKey(4);               // 3 — ближайший ≤ 4
treeMap.ceilingKey(4);             // 5 — ближайший ≥ 4
treeMap.headMap(5);                // {3=three} — всё до 5
treeMap.tailMap(5);                // {5=five, 7=seven}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **BST** — O(log n) в среднем, O(n) в худшем (несбалансированное)
> - **In-Order обход BST** → отсортированная последовательность
> - **Level-Order** — BFS через Queue, обходит по уровням
> - **Pre-Order** — корень первым, для копирования/сериализации
> - **Post-Order** — корень последним, для удаления/вычисления
> - `TreeMap` / `TreeSet` — Red-Black Tree, O(log n) гарантированно
> - Рекурсия — естественный способ обхода дерева
