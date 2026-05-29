> **Теги:** #algorithms #combinatorics #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[main Algorithms]]

# 🎲 Комбинаторика

---

## 🔹 Основные понятия

| Понятие | Формула | Порядок важен | Повторения |
|---------|---------|--------------|------------|
| Размещения | A(n,k) = n!/(n-k)! | ✅ | ❌ |
| Перестановки | P(n) = n! | ✅ | ❌ |
| Сочетания | C(n,k) = n!/k!(n-k)! | ❌ | ❌ |
| Размещения с повторениями | nᵏ | ✅ | ✅ |
| Сочетания с повторениями | C(n+k-1, k) | ❌ | ✅ |

---

## 🔹 Перестановки

> [!note] Перестановка
> Все возможные **упорядоченные** последовательности из n элементов. P(n) = n!

```
n=3, элементы [1,2,3]:
[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]
P(3) = 3! = 6
```

### Генерация всех перестановок (backtracking)
```java
List<List<Integer>> permutations(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new ArrayList<>(), new boolean[nums.length], result);
    return result;
}

void backtrack(int[] nums, List<Integer> current, boolean[] used,
               List<List<Integer>> result) {
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        current.add(nums[i]);
        backtrack(nums, current, used, result);
        current.remove(current.size() - 1);
        used[i] = false;
    }
}
// O(n * n!) time — n! перестановок, каждая длиной n
```

---

## 🔹 Сочетания

> [!note] Сочетание
> Выбор k элементов из n **без учёта порядка**. C(n,k) = n! / (k! × (n-k)!)

```
C(4,2) = 4!/(2!×2!) = 6
Из [1,2,3,4] выбрать 2: {1,2},{1,3},{1,4},{2,3},{2,4},{3,4}
```

### Вычисление C(n,k)
```java
// Рекуррентность: C(n,k) = C(n-1,k-1) + C(n-1,k)
long combinations(int n, int k) {
    if (k == 0 || k == n) return 1;
    long[][] dp = new long[n + 1][k + 1];
    for (int i = 0; i <= n; i++) {
        dp[i][0] = 1;
        for (int j = 1; j <= Math.min(i, k); j++) {
            dp[i][j] = dp[i - 1][j - 1] + dp[i - 1][j];
        }
    }
    return dp[n][k];
}
```

### Треугольник Паскаля
```java
int[][] pascal(int n) {
    int[][] triangle = new int[n][n];
    for (int i = 0; i < n; i++) {
        triangle[i][0] = 1;
        for (int j = 1; j <= i; j++) {
            triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j];
        }
    }
    return triangle;
}
```

### Генерация всех сочетаний
```java
List<List<Integer>> combine(int n, int k) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(1, n, k, new ArrayList<>(), result);
    return result;
}

void backtrack(int start, int n, int k, List<Integer> current,
               List<List<Integer>> result) {
    if (current.size() == k) {
        result.add(new ArrayList<>(current));
        return;
    }
    for (int i = start; i <= n; i++) {
        current.add(i);
        backtrack(i + 1, n, k, current, result);
        current.remove(current.size() - 1);
    }
}
```

---

## 🔹 Подмножества (Subsets)

> [!note] Подмножества
> Все подмножества множества из n элементов. Количество = 2ⁿ.

```
[1,2,3] → [], [1], [2], [3], [1,2], [1,3], [2,3], [1,2,3]
2³ = 8 подмножеств
```

### Через backtracking
```java
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

void backtrack(int[] nums, int start, List<Integer> current,
               List<List<Integer>> result) {
    result.add(new ArrayList<>(current)); // добавляем текущее подмножество
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);
        backtrack(nums, i + 1, current, result);
        current.remove(current.size() - 1);
    }
}
```

### Через битовую маску
```java
List<List<Integer>> subsetsBitmask(int[] nums) {
    int n = nums.length;
    List<List<Integer>> result = new ArrayList<>();
    for (int mask = 0; mask < (1 << n); mask++) {
        List<Integer> sub = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask >> i & 1) == 1) sub.add(nums[i]);
        }
        result.add(sub);
    }
    return result;
}
// mask=0b101 → берём элементы на позициях 0 и 2
```

---

## 🔹 Принцип умножения и сложения

> [!note] Принцип умножения
> Если задача состоит из независимых шагов (шаг 1 имеет m вариантов, шаг 2 — n вариантов), то всего m × n комбинаций.

> [!note] Принцип сложения
> Если выбираем из взаимоисключающих вариантов (m штук из группы A или n штук из группы B), то всего m + n вариантов.

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Перестановки** — порядок важен, P(n) = n!, backtracking + used[]
> - **Сочетания** — порядок не важен, C(n,k) = n!/k!(n-k)!, backtracking с start
> - **Подмножества** — 2ⁿ штук, backtracking или битовые маски
> - **Треугольник Паскаля** — быстрый расчёт C(n,k) через DP
> - Временная сложность перебора: O(n!) — только для малых n (≤10-12)
