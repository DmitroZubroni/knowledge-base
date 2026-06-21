> **Теги:** #algorithms #graphs #bfs #dfs #dijkstra #топологическая-сортировка #union-find #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]]

# 🕸 Графы (Graphs)

---

## 🔹 Основные понятия

```
Неориентированный:    Ориентированный:     Взвешенный:
  1 — 2                  1 → 2               1 —(5)— 2
  |   |                  ↑   ↓               |       |
  3 — 4                  4 ← 3              (2)     (3)
                                              3 —(1)— 4
```

| Термин | Описание |
|--------|----------|
| Directed (ориентированный) | Рёбра имеют направление A→B |
| Undirected (неориентированный) | Рёбра без направления A—B |
| Weighted (взвешенный) | Рёбра имеют вес/стоимость |
| DAG | Directed Acyclic Graph — ориентированный без циклов |
| Connected | Все вершины связаны (для undirected) |
| Strongly Connected | Из любой вершины есть путь в любую (для directed) |
| Degree | Степень вершины — количество рёбер |
| In-degree / Out-degree | Количество входящих / исходящих рёбер |

---

## 🔹 Представление графа

### Список смежности — предпочтительный

```java
// Space: O(V + E) — экономично для разреженных графов
// Невзвешенный
Map<Integer, List<Integer>> graph = new HashMap<>();
graph.computeIfAbsent(1, k -> new ArrayList<>()).add(2);
graph.computeIfAbsent(1, k -> new ArrayList<>()).add(3);

// Взвешенный: List<int[]> где int[] = {сосед, вес}
Map<Integer, List<int[]>> weightedGraph = new HashMap<>();
weightedGraph.computeIfAbsent(1, k -> new ArrayList<>()).add(new int[]{2, 5});
weightedGraph.computeIfAbsent(1, k -> new ArrayList<>()).add(new int[]{3, 2});

// Инициализация из массива рёбер (удобно в задачах)
int n = 5; // количество вершин
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());

int[][] edges = {{0,1}, {0,2}, {1,3}, {2,3}};
for (int[] e : edges) {
    adj.get(e[0]).add(e[1]);
    adj.get(e[1]).add(e[0]); // для undirected
}
```

### Матрица смежности

```java
// Space: O(V²) — плохо для разреженных, хорошо для плотных
int V = 5;
int[][] matrix = new int[V][V];
matrix[0][1] = 1;    // невзвешенное ребро 0→1
matrix[0][2] = 5;    // взвешенное ребро 0→2 с весом 5

// Преимущество: проверить ребро (u,v) → O(1) вместо O(degree)
```

| | Список смежности | Матрица смежности |
|--|-----------------|-------------------|
| Space | O(V + E) | O(V²) |
| Проверить ребро (u,v) | O(degree) | **O(1)** |
| Список соседей u | **O(degree)** | O(V) |
| Когда использовать | Разреженные графы (E << V²) | Плотные графы, Floyd-Warshall |

---

## 🔹 BFS (Поиск в ширину)

BFS обходит граф **уровень за уровнем**. Использует Queue. Находит кратчайший путь в **невзвешенном** графе.

```java
// Базовый BFS
void bfs(List<List<Integer>> adj, int start, int V) {
    boolean[] visited = new boolean[V];
    Queue<Integer> queue = new ArrayDeque<>();

    visited[start] = true;
    queue.offer(start);

    while (!queue.isEmpty()) {
        int node = queue.poll();
        System.out.print(node + " ");

        for (int neighbor : adj.get(node)) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue.offer(neighbor);
            }
        }
    }
}
// O(V + E) time, O(V) space

// Кратчайший путь в невзвешенном графе
int shortestPath(List<List<Integer>> adj, int start, int end) {
    boolean[] visited = new boolean[adj.size()];
    Queue<int[]> queue = new ArrayDeque<>(); // [node, distance]

    visited[start] = true;
    queue.offer(new int[]{start, 0});

    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        int node = curr[0], dist = curr[1];

        if (node == end) return dist;

        for (int neighbor : adj.get(node)) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue.offer(new int[]{neighbor, dist + 1});
            }
        }
    }
    return -1; // путь не найден
}
```

---

## 🔹 DFS (Поиск в глубину)

DFS идёт **как можно глубже** по одной ветке, затем откатывается. Использует Stack или рекурсию.

```java
// Рекурсивный DFS
void dfs(List<List<Integer>> adj, int node, boolean[] visited) {
    visited[node] = true;
    System.out.print(node + " ");

    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) {
            dfs(adj, neighbor, visited);
        }
    }
}

// Итеративный DFS — когда рекурсия даёт StackOverflow
void dfsIterative(List<List<Integer>> adj, int start) {
    boolean[] visited = new boolean[adj.size()];
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);

    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;
        visited[node] = true;
        System.out.print(node + " ");

        for (int neighbor : adj.get(node)) {
            if (!visited[neighbor]) stack.push(neighbor);
        }
    }
}
```

---

## 🔹 BFS vs DFS — что когда использовать

| Задача | Алгоритм | Почему |
|--------|----------|--------|
| Кратчайший путь в невзвешенном | **BFS** | Гарантирует минимальные уровни |
| Кратчайший путь во взвешенном | **Дейкстра** | BFS не учитывает веса |
| Обнаружение цикла | BFS или DFS | Оба работают |
| Топологическая сортировка | **DFS** или Kahn (BFS) | |
| Компоненты связности | BFS или DFS | Оба работают |
| Проверка двудольности | **BFS** | Раскраска по уровням |
| Поиск в глубину/бэктрекинг | **DFS** | По природе алгоритма |
| Кратчайший путь с отриц. весами | **Беллман-Форд** | Дейкстра не работает |

---

## 🔹 Алгоритм Дейкстры

Кратчайший путь от стартовой вершины до всех остальных во **взвешенном** графе с **неотрицательными** весами.

**Идея:** всегда обрабатываем вершину с минимальным текущим расстоянием → PriorityQueue.

```java
int[] dijkstra(List<List<int[]>> adj, int start) {
    int V = adj.size();
    int[] dist = new int[V];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;

    // PriorityQueue: [расстояние, вершина]
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[]{0, start});

    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];

        if (d > dist[u]) continue; // устаревшая запись — пропускаем

        for (int[] edge : adj.get(u)) {
            int v = edge[0], w = edge[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist; // dist[i] = кратчайшее расстояние от start до i
}
// O((V + E) log V)
```

**Трассировка:**
```
Граф: 0—(4)—1, 0—(1)—2, 2—(2)—1, 1—(1)—3
Start: 0

Шаг 1: pq=[(0,0)]. Poll (0,0). dist=[0,∞,∞,∞]
  0→1: dist[1]=4. 0→2: dist[2]=1
  pq=[(1,2),(4,1)]

Шаг 2: Poll (1,2). dist=[0,4,1,∞]
  2→1: dist[1]=min(4, 1+2)=3. Обновили!
  pq=[(3,1),(4,1 устарел)]

Шаг 3: Poll (3,1). dist=[0,3,1,∞]
  1→3: dist[3]=3+1=4
  pq=[(4,1 устарел),(4,3)]

Результат: dist=[0,3,1,4]
```

> [!warning] Дейкстра не работает с отрицательными рёбрами
> При отрицательных весах уже "закрытая" вершина может получить более короткий путь через отрицательное ребро. Используй **Беллман-Форд**.

---

## 🔹 Алгоритм Беллмана-Форда

Кратчайший путь с **отрицательными весами**. Также обнаруживает **отрицательные циклы**.

**Идея:** релаксировать все рёбра V-1 раз (максимальная длина пути без цикла = V-1 рёбер).

```java
int[] bellmanFord(int V, int[][] edges, int start) {
    // edges[i] = [u, v, weight]
    int[] dist = new int[V];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;

    // V-1 итераций релаксации
    for (int i = 0; i < V - 1; i++) {
        for (int[] edge : edges) {
            int u = edge[0], v = edge[1], w = edge[2];
            if (dist[u] != Integer.MAX_VALUE && dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
            }
        }
    }

    // Проверка отрицательных циклов
    // Если после V-1 итераций ещё можно улучшить — есть отрицательный цикл
    for (int[] edge : edges) {
        int u = edge[0], v = edge[1], w = edge[2];
        if (dist[u] != Integer.MAX_VALUE && dist[u] + w < dist[v]) {
            System.out.println("Обнаружен отрицательный цикл!");
            return null;
        }
    }
    return dist;
}
// O(V * E) — медленнее Дейкстры, но работает с отрицательными весами
```

**Когда применять:**
- Граф с отрицательными весами рёбер
- Нужно обнаружить отрицательный цикл (финансовый арбитраж, routing)
- Кратчайший путь с ограничением на количество рёбер

---

## 🔹 Floyd-Warshall — кратчайшие пути между всеми парами

```java
// Кратчайший путь между ВСЕМИ парами вершин
int[][] floydWarshall(int[][] graph) {
    int V = graph.length;
    int[][] dist = new int[V][V];

    // Инициализация: копируем граф (INF = нет ребра)
    for (int i = 0; i < V; i++)
        dist[i] = Arrays.copyOf(graph[i], V);

    // Для каждой промежуточной вершины k
    for (int k = 0; k < V; k++) {
        for (int i = 0; i < V; i++) {
            for (int j = 0; j < V; j++) {
                if (dist[i][k] != Integer.MAX_VALUE &&
                    dist[k][j] != Integer.MAX_VALUE &&
                    dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }
    return dist; // dist[i][j] = кратчайшее расстояние от i до j
}
// O(V³) time, O(V²) space
// Когда: плотный граф, нужны расстояния между всеми парами
```

---

## 🔹 Топологическая сортировка

Линейный порядок вершин DAG (ориентированного ациклического графа) такой что для каждого ребра u→v вершина u идёт раньше v.

**Применение:** порядок выполнения задач (build systems, сборка зависимостей), расписания курсов.

### Kahn's Algorithm (BFS — через in-degree)

```java
List<Integer> topologicalSort(List<List<Integer>> adj, int V) {
    int[] inDegree = new int[V];
    for (int u = 0; u < V; u++)
        for (int v : adj.get(u))
            inDegree[v]++;

    // Начинаем с вершин без входящих рёбер
    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < V; i++)
        if (inDegree[i] == 0) queue.offer(i);

    List<Integer> order = new ArrayList<>();
    while (!queue.isEmpty()) {
        int u = queue.poll();
        order.add(u);

        for (int v : adj.get(u)) {
            inDegree[v]--;
            if (inDegree[v] == 0) queue.offer(v);
        }
    }

    // Если обработали не все вершины — есть цикл (не DAG)
    if (order.size() != V) throw new IllegalStateException("Граф содержит цикл!");
    return order;
}
// O(V + E)
```

**Трассировка:**
```
Граф: 0→1, 0→2, 1→3, 2→3
inDegree: [0, 1, 1, 2]

Очередь: [0]  (inDegree=0)
Poll 0: order=[0], уменьшаем inDegree[1]=0, inDegree[2]=0
Очередь: [1, 2]
Poll 1: order=[0,1], уменьшаем inDegree[3]=1
Poll 2: order=[0,1,2], уменьшаем inDegree[3]=0
Очередь: [3]
Poll 3: order=[0,1,2,3] ← топологический порядок
```

### DFS-based топологическая сортировка

```java
List<Integer> topologicalSortDFS(List<List<Integer>> adj, int V) {
    boolean[] visited = new boolean[V];
    Deque<Integer> stack = new ArrayDeque<>();

    for (int i = 0; i < V; i++)
        if (!visited[i])
            dfsTopoSort(adj, i, visited, stack);

    List<Integer> order = new ArrayList<>();
    while (!stack.isEmpty()) order.add(stack.pop());
    return order;
}

void dfsTopoSort(List<List<Integer>> adj, int node, boolean[] visited, Deque<Integer> stack) {
    visited[node] = true;
    for (int neighbor : adj.get(node))
        if (!visited[neighbor])
            dfsTopoSort(adj, neighbor, visited, stack);
    stack.push(node); // добавляем ПОСЛЕ обработки всех потомков
}
// O(V + E)
```

> [!tip] Kahn vs DFS топологическая сортировка
> - **Kahn** — легче обнаруживает цикл (если `order.size() != V`), интуитивнее
> - **DFS** — компактнее, используется когда DFS уже написан
> - Оба дают корректный топологический порядок (может быть несколько валидных)

---

## 🔹 Union-Find (Disjoint Set Union, DSU)

Структура данных для эффективного определения: **принадлежат ли два элемента одному множеству** и **объединения множеств**.

**Применение:** связность графа, цикл в undirected графе, алгоритм Краскала (MST), задачи о компонентах.

```java
class UnionFind {
    private int[] parent;
    private int[] rank;     // для оптимизации по рангу
    private int components; // количество компонентов

    public UnionFind(int n) {
        parent = new int[n];
        rank   = new int[n];
        components = n;
        for (int i = 0; i < n; i++) parent[i] = i; // каждый сам себе родитель
    }

    // Find с path compression — O(α(n)) ≈ O(1)
    public int find(int x) {
        if (parent[x] != x)
            parent[x] = find(parent[x]); // path compression: сразу к корню
        return parent[x];
    }

    // Union по рангу
    public boolean union(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        if (rootX == rootY) return false; // уже в одном компоненте

        // Присоединяем меньшее дерево к большему
        if (rank[rootX] < rank[rootY]) { int tmp = rootX; rootX = rootY; rootY = tmp; }
        parent[rootY] = rootX;
        if (rank[rootX] == rank[rootY]) rank[rootX]++;

        components--;
        return true; // успешно объединили
    }

    public boolean connected(int x, int y) {
        return find(x) == find(y);
    }

    public int getComponents() { return components; }
}
```

### Применение: обнаружение цикла в undirected графе

```java
boolean hasCycle(int V, int[][] edges) {
    UnionFind uf = new UnionFind(V);
    for (int[] edge : edges) {
        // Если оба конца уже в одном компоненте — добавление ребра создаст цикл
        if (!uf.union(edge[0], edge[1])) return true;
    }
    return false;
}
```

### Применение: количество островов (Number of Islands)

```java
int numIslands(char[][] grid) {
    int m = grid.length, n = grid[0].length;
    UnionFind uf = new UnionFind(m * n);
    int water = 0;

    int[] dr = {0, 0, 1, -1};
    int[] dc = {1, -1, 0, 0};

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == '0') { water++; continue; }
            for (int d = 0; d < 4; d++) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
                    uf.union(r * n + c, nr * n + nc);
                }
            }
        }
    }
    return uf.getComponents() - water; // компоненты минус вода
}
```

---

## 🔹 Минимальное остовное дерево (MST)

MST — дерево, соединяющее все вершины с минимальной суммой весов рёбер.

### Алгоритм Краскала (Kruskal) — через Union-Find

```java
int kruskalMST(int V, int[][] edges) {
    // edges[i] = [u, v, weight]
    Arrays.sort(edges, Comparator.comparingInt(e -> e[2])); // сортируем по весу

    UnionFind uf = new UnionFind(V);
    int totalWeight = 0;
    int edgesInMST = 0;

    for (int[] edge : edges) {
        int u = edge[0], v = edge[1], w = edge[2];
        if (uf.union(u, v)) { // объединяем если в разных компонентах (не создаёт цикл)
            totalWeight += w;
            edgesInMST++;
            System.out.println("MST edge: " + u + " - " + v + " weight: " + w);
            if (edgesInMST == V - 1) break; // MST готово (V-1 рёбер)
        }
    }
    return totalWeight;
}
// O(E log E) — из-за сортировки рёбер
```

### Алгоритм Прима (Prim) — через PriorityQueue

```java
int primMST(List<List<int[]>> adj, int V) {
    boolean[] inMST = new boolean[V];
    // PriorityQueue: [вес, вершина]
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[]{0, 0}); // начинаем с вершины 0

    int totalWeight = 0;
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int w = curr[0], u = curr[1];

        if (inMST[u]) continue; // уже в MST
        inMST[u] = true;
        totalWeight += w;

        for (int[] edge : adj.get(u)) {
            int v = edge[0], weight = edge[1];
            if (!inMST[v]) pq.offer(new int[]{weight, v});
        }
    }
    return totalWeight;
}
// O((V + E) log V)
```

| | Краскал | Прим |
|-|---------|------|
| Подход | Сортировать рёбра, добавлять дешёвые | Расти от стартовой вершины |
| Структура | Union-Find | PriorityQueue |
| Лучше для | Разреженных графов | Плотных графов |
| Сложность | O(E log E) | O((V+E) log V) |

---

## 🔹 Обнаружение цикла и компоненты связности

### Обнаружение цикла в directed графе — через цвета

```java
// 0 = не посещён, 1 = в процессе (в текущем пути), 2 = завершён
boolean hasCycleDirected(List<List<Integer>> adj, int V) {
    int[] state = new int[V];
    for (int i = 0; i < V; i++)
        if (state[i] == 0 && dfsHasCycle(adj, i, state))
            return true;
    return false;
}

boolean dfsHasCycle(List<List<Integer>> adj, int node, int[] state) {
    state[node] = 1; // в процессе
    for (int neighbor : adj.get(node)) {
        if (state[neighbor] == 1) return true;  // back edge = цикл!
        if (state[neighbor] == 0 && dfsHasCycle(adj, neighbor, state)) return true;
    }
    state[node] = 2; // завершён
    return false;
}
```

### Компоненты связности (undirected)

```java
int countComponents(int V, int[][] edges) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < V; i++) adj.add(new ArrayList<>());
    for (int[] e : edges) {
        adj.get(e[0]).add(e[1]);
        adj.get(e[1]).add(e[0]);
    }

    boolean[] visited = new boolean[V];
    int components = 0;
    for (int i = 0; i < V; i++) {
        if (!visited[i]) {
            dfs(adj, i, visited); // обходим всю компоненту
            components++;
        }
    }
    return components;
}
// Или через Union-Find: components = uf.getComponents()
```

---

## 🔹 Задача: Number of Islands (классика)

```java
// BFS подход
int numIslands(char[][] grid) {
    int m = grid.length, n = grid[0].length;
    int islands = 0;
    int[] dr = {0, 0, 1, -1};
    int[] dc = {1, -1, 0, 0};

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == '1') {
                islands++;
                // BFS: "затопить" весь остров
                Queue<int[]> queue = new ArrayDeque<>();
                queue.offer(new int[]{r, c});
                grid[r][c] = '0'; // помечаем посещённым

                while (!queue.isEmpty()) {
                    int[] cell = queue.poll();
                    for (int d = 0; d < 4; d++) {
                        int nr = cell[0] + dr[d];
                        int nc = cell[1] + dc[d];
                        if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
                            grid[nr][nc] = '0';
                            queue.offer(new int[]{nr, nc});
                        }
                    }
                }
            }
        }
    }
    return islands;
}
```

---

## 🔹 Задача: Course Schedule (топологическая сортировка)

```java
// Можно ли пройти все курсы? (нет ли циклических зависимостей)
boolean canFinish(int numCourses, int[][] prerequisites) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());
    int[] inDegree = new int[numCourses];

    for (int[] pre : prerequisites) {
        adj.get(pre[1]).add(pre[0]);
        inDegree[pre[0]]++;
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < numCourses; i++)
        if (inDegree[i] == 0) queue.offer(i);

    int completed = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        completed++;
        for (int next : adj.get(course))
            if (--inDegree[next] == 0) queue.offer(next);
    }
    return completed == numCourses; // если все прошли — нет цикла
}
```

---

## 🔹 Паттерны задач на графы

```
Задача                              Алгоритм           Структура
─────────────────────────────────────────────────────────────────────
Кратчайший путь, без весов         BFS                Queue + visited
Кратчайший путь, с весами ≥ 0      Dijkstra           PriorityQueue
Кратчайший путь, с отриц. весами   Bellman-Ford       Итерации V-1 раз
Все пары кратчайших путей          Floyd-Warshall     Матрица O(V³)
Топологический порядок             Kahn (BFS) / DFS   inDegree / Stack
Есть ли цикл (directed)            DFS с цветами      0=белый 1=серый 2=чёрный
Есть ли цикл (undirected)          Union-Find         find/union
Компоненты связности               BFS/DFS / DSU      visited / parent[]
Минимальное остовное дерево        Kruskal / Prim     DSU / PriorityQueue
Острова, регионы в матрице         BFS/DFS            4 направления
Зависимости, расписание            Топо. сортировка   inDegree + queue
```

---

## 🔹 Итог

```
Представление:
  Список смежности — O(V+E) памяти. Стандарт для разреженных графов.
  Матрица смежности — O(V²). Для плотных или Floyd-Warshall.

Обходы:
  BFS → Queue + visited. Кратчайший путь в невзвешенном.
  DFS → Stack/рекурсия. Цикл, компоненты, топосортировка.

Кратчайшие пути:
  Дейкстра   — PriorityQueue, O((V+E)logV), только неотрицательные веса
  Беллман-Форд — O(V*E), отрицательные веса, обнаружение отриц. цикла
  Floyd-Warshall — O(V³), все пары, матрица

Топологическая сортировка (только DAG):
  Kahn (BFS) — inDegree + Queue. Лучше обнаруживает цикл.
  DFS — обратный postorder. Компактнее.

Union-Find (DSU):
  find с path compression → O(α(n)) ≈ O(1)
  union по рангу → O(α(n))
  Применение: цикл в undirected, компоненты, Kruskal MST

MST:
  Kruskal — сортировка рёбер + Union-Find. Лучше для разреженных.
  Prim    — PriorityQueue от вершины. Лучше для плотных.

Обнаружение цикла:
  Directed  → DFS с цветами (0=белый, 1=серый, 2=чёрный)
  Undirected → Union-Find (union вернул false) или BFS/DFS с parent
```
