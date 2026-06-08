> **Теги:** #algorithms #graphs #bfs #dfs #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]]

# 🕸 Графы (Graphs)

---

## 🔹 Основные понятия

> [!note] Граф
> **Граф** — структура данных из **вершин** (vertices/nodes) и **рёбер** (edges). G = (V, E)

| Термин | Описание |
|--------|----------|
| Directed (ориентированный) | Рёбра имеют направление A→B |
| Undirected (неориентированный) | Рёбра без направления A-B |
| Weighted (взвешенный) | Рёбра имеют вес/стоимость |
| Cyclic | Есть циклы |
| Acyclic | Нет циклов |
| DAG | Directed Acyclic Graph |
| Connected | Все вершины связаны |

---

## 🔹 Представление графа

### Список смежности (Adjacency List) — предпочтительный
```java
// V вершин, E рёбер
// Space: O(V + E)
Map<Integer, List<Integer>> graph = new HashMap<>();

// Добавить вершину
graph.put(1, new ArrayList<>());
graph.put(2, new ArrayList<>());

// Добавить ребро 1 → 2
graph.get(1).add(2);
// Для неориентированного графа:
graph.get(2).add(1); // ← тоже

// Инициализация из списка рёбер
int[][] edges = {{0,1}, {0,2}, {1,3}, {2,3}};
Map<Integer, List<Integer>> g = new HashMap<>();
for (int[] e : edges) {
    g.computeIfAbsent(e[0], k -> new ArrayList<>()).add(e[1]);
    g.computeIfAbsent(e[1], k -> new ArrayList<>()).add(e[0]); // undirected
}
```

### Матрица смежности (Adjacency Matrix)
```java
// Space: O(V²) — плохо для разреженных графов
int V = 5;
int[][] matrix = new int[V][V];
matrix[0][1] = 1; // ребро 0 → 1
// matrix[i][j] = 1 если есть ребро i → j
```

| | Список смежности | Матрица смежности |
|--|-----------------|-------------------|
| Space | O(V + E) | O(V²) |
| Проверить ребро (u,v) | O(degree) | O(1) |
| Список соседей | O(degree) | O(V) |
| Когда лучше | Разреженные графы | Плотные графы |

---

## 🔹 BFS (Поиск в ширину)

> [!note] BFS
> Обходит граф **уровень за уровнем**, начиная от стартовой вершины. Использует **Queue**.
> Находит кратчайший путь в **невзвешенном** графе.

```java
void bfs(Map<Integer, List<Integer>> graph, int start) {
    Set<Integer> visited = new HashSet<>();
    Queue<Integer> queue = new ArrayDeque<>();

    queue.offer(start);
    visited.add(start);

    while (!queue.isEmpty()) {
        int node = queue.poll();
        System.out.print(node + " ");

        for (int neighbor : graph.getOrDefault(node, List.of())) {
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(neighbor);
            }
        }
    }
}
// O(V + E) time, O(V) space
```

### Кратчайший путь через BFS
```java
int shortestPath(Map<Integer, List<Integer>> graph, int start, int end) {
    if (start == end) return 0;
    Set<Integer> visited = new HashSet<>();
    Queue<int[]> queue = new ArrayDeque<>(); // [node, distance]

    queue.offer(new int[]{start, 0});
    visited.add(start);

    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        int node = curr[0], dist = curr[1];

        for (int neighbor : graph.getOrDefault(node, List.of())) {
            if (neighbor == end) return dist + 1;
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(new int[]{neighbor, dist + 1});
            }
        }
    }
    return -1; // путь не найден
}
```

---

## 🔹 DFS (Поиск в глубину)

> [!note] DFS
> Идёт **как можно глубже** по одной ветке, затем откатывается (backtrack). Использует **Stack** или **рекурсию**.

### Рекурсивный DFS
```java
void dfs(Map<Integer, List<Integer>> graph, int node, Set<Integer> visited) {
    visited.add(node);
    System.out.print(node + " ");

    for (int neighbor : graph.getOrDefault(node, List.of())) {
        if (!visited.contains(neighbor)) {
            dfs(graph, neighbor, visited);
        }
    }
}
// Вызов:
Set<Integer> visited = new HashSet<>();
dfs(graph, startNode, visited);
// O(V + E) time, O(V) space (стек рекурсии)
```

### Итеративный DFS (через Stack)
```java
void dfsIterative(Map<Integer, List<Integer>> graph, int start) {
    Set<Integer> visited = new HashSet<>();
    Deque<Integer> stack = new ArrayDeque<>();

    stack.push(start);

    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited.contains(node)) continue;
        visited.add(node);
        System.out.print(node + " ");

        for (int neighbor : graph.getOrDefault(node, List.of())) {
            if (!visited.contains(neighbor)) stack.push(neighbor);
        }
    }
}
```

---

## 🔹 BFS vs DFS

| | BFS | DFS |
|--|-----|-----|
| Структура | Queue | Stack / рекурсия |
| Порядок | По уровням | В глубину |
| Кратчайший путь | ✅ (невзвешенный) | ❌ |
| Поиск цикла | ✅ | ✅ |
| Топологическая сортировка | ✅ (Kahn) | ✅ |
| Компоненты связности | ✅ | ✅ |
| Space | O(V) — ширина | O(V) — глубина |

---

## 🔹 Алгоритм Дейкстры (кратчайший путь во взвешенном графе)

> [!note] Дейкстра
> Находит кратчайший путь от стартовой вершины до всех остальных в **взвешенном** графе с **неотрицательными** весами.

```java
int[] dijkstra(Map<Integer, List<int[]>> graph, int start, int V) {
    int[] dist = new int[V];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;

    // PriorityQueue: [расстояние, вершина]
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, start});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];

        if (d > dist[u]) continue; // устаревшая запись

        for (int[] edge : graph.getOrDefault(u, List.of())) {
            int v = edge[0], w = edge[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
// O((V + E) log V)
```

> [!warning] Дейкстра не работает с отрицательными рёбрами
> Для графов с отрицательными весами — алгоритм Беллмана-Форда.

---

## 🔹 Обнаружение цикла в графе

```java
// Для неориентированного графа — через цвета
// 0 = не посещён, 1 = в процессе, 2 = завершён
boolean hasCycle(Map<Integer, List<Integer>> graph, int V) {
    int[] color = new int[V];
    for (int i = 0; i < V; i++) {
        if (color[i] == 0 && dfsHasCycle(graph, i, color)) return true;
    }
    return false;
}

boolean dfsHasCycle(Map<Integer, List<Integer>> graph, int node, int[] color) {
    color[node] = 1; // в процессе
    for (int neighbor : graph.getOrDefault(node, List.of())) {
        if (color[neighbor] == 1) return true; // цикл!
        if (color[neighbor] == 0 && dfsHasCycle(graph, neighbor, color)) return true;
    }
    color[node] = 2; // завершён
    return false;
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Список смежности** — стандартное представление, O(V + E) память
> - **BFS** → Queue, кратчайший путь в невзвешенном графе
> - **DFS** → Stack / рекурсия, обнаружение цикла, компоненты связности
> - **Дейкстра** → PriorityQueue, кратчайший путь во взвешенном (без отрицательных весов)
> - Всегда отслеживай `visited` чтобы не зациклиться
> - O(V + E) — стандартная сложность BFS и DFS
