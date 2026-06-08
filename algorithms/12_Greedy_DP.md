> **Теги:** #algorithms #greedy #dp #динамическоепрограммирование #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]]

# 🧩 Жадные алгоритмы и Динамическое программирование

---

## 🔹 Жадные алгоритмы (Greedy)

> [!note] Принцип
> На каждом шаге выбираем **локально оптимальное** решение, надеясь прийти к **глобальному оптимуму**. Нет отката назад.

### Когда работает жадный подход
- Задача обладает **свойством жадного выбора**: локальный оптимум → глобальный оптимум
- Задача обладает **оптимальной подструктурой**: оптимальное решение содержит оптимальные решения подзадач

### Классический пример: задача о монетах
```java
// Минимальное количество монет для суммы amount
// (работает для стандартных наборов монет: 1, 5, 10, 25)
int coinChangeGreedy(int[] coins, int amount) {
    Arrays.sort(coins);
    int count = 0;
    for (int i = coins.length - 1; i >= 0 && amount > 0; i--) {
        count += amount / coins[i];
        amount %= coins[i];
    }
    return amount == 0 ? count : -1;
}
```

> [!warning] Жадный не всегда оптимален
> Для монет [1, 3, 4] и суммы 6:
> - Жадный: 4+1+1 = 3 монеты
> - Оптимально: 3+3 = 2 монеты ← DP лучше!

### Пример: интервалы без пересечений
```java
// Максимальное количество непересекающихся интервалов
// Жадный: сортируем по концу, берём раньше заканчивающийся
int eraseOverlapIntervals(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[1] - b[1]);
    int count = 0, end = Integer.MIN_VALUE;
    for (int[] interval : intervals) {
        if (interval[0] >= end) {
            end = interval[1]; // берём этот интервал
        } else {
            count++; // удаляем пересекающийся
        }
    }
    return count;
}
```

---

## 🔹 Динамическое программирование (DP)

> [!note] DP — когда применять
> Задача подходит для DP если:
> 1. **Перекрывающиеся подзадачи** — те же вычисления повторяются
> 2. **Оптимальная подструктура** — решение через решения подзадач

### Два подхода

| Подход | Описание | Риски |
|--------|----------|-------|
| Top-Down (мемоизация) | Рекурсия + кэш | StackOverflow на больших n |
| Bottom-Up (табуляция) | Цикл, заполняем таблицу | Нужно правильно определить порядок |

---

## 🔹 Задача о монетах (DP)

> Минимальное количество монет для набора суммы `amount`.

```java
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // "бесконечность"
    dp[0] = 0;

    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
// dp[6] для [1,3,4]: dp[6] = min(dp[5]+1, dp[3]+1, dp[2]+1) = min(3,1+1,?) = 2 ✅
// O(amount * coins.length) time, O(amount) space
```

---

## 🔹 Рюкзак 0/1 (0/1 Knapsack)

> n предметов с весом `w[i]` и ценностью `v[i]`. Рюкзак вмещает `W`. Максимизировать ценность.

```java
int knapsack(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[][] dp = new int[n + 1][W + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            // не берём предмет i
            dp[i][w] = dp[i-1][w];
            // берём предмет i (если помещается)
            if (weights[i-1] <= w) {
                dp[i][w] = Math.max(dp[i][w],
                    dp[i-1][w - weights[i-1]] + values[i-1]);
            }
        }
    }
    return dp[n][W];
}
// O(n * W) time, O(n * W) space
```

---

## 🔹 Наибольшая общая подпоследовательность (LCS)

> Для строк s и t найти длину наибольшей общей подпоследовательности.

```java
int lcs(String s, String t) {
    int m = s.length(), n = t.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s.charAt(i-1) == t.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
            }
        }
    }
    return dp[m][n];
}
// LCS("ABCBDAB", "BDCAB") = 4 ("BCAB" или "BDAB")
// O(m * n) time, O(m * n) space
```

---

## 🔹 Наибольшая возрастающая подпоследовательность (LIS)

```java
int lis(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1); // каждый элемент — подпоследовательность длиной 1

    int maxLen = 1;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    return maxLen;
}
// [10, 9, 2, 5, 3, 7, 101, 18] → LIS = 4 ([2,3,7,101])
// O(n²) — есть оптимизация до O(n log n) через бинарный поиск
```

---

## 🔹 Максимальная сумма подмассива (Kadane's Algorithm)

```java
int maxSubarraySum(int[] nums) {
    int maxSum = nums[0];
    int currSum = nums[0];

    for (int i = 1; i < nums.length; i++) {
        // либо продолжаем текущий подмассив, либо начинаем новый
        currSum = Math.max(nums[i], currSum + nums[i]);
        maxSum = Math.max(maxSum, currSum);
    }
    return maxSum;
}
// [-2,1,-3,4,-1,2,1,-5,4] → 6 ([4,-1,2,1])
// O(n) time, O(1) space
```

---

## 🔹 Паттерны DP

```
1D DP:  dp[i] зависит от dp[i-1], dp[i-2]...
        → Фибоначчи, монеты, LIS

2D DP:  dp[i][j] зависит от dp[i-1][j], dp[i][j-1]...
        → LCS, рюкзак, расстояние редактирования

Interval DP: dp[i][j] = оптимум для подзадачи [i..j]
        → умножение матриц, разбиение строки

Bitmask DP: состояние — битовая маска посещённых вершин
        → задача коммивояжёра (TSP)
```

---

## 🔹 Сравнение: Greedy vs DP

| | Greedy | DP |
|--|--------|-----|
| Подход | Локальный оптимум | Перебор всех подзадач |
| Скорость | Быстрее | Медленнее |
| Гарантия | Не всегда оптимально | Всегда оптимально |
| Откат | Нет | Через таблицу |
| Применение | Интервалы, Дейкстра, Хаффман | Рюкзак, LCS, монеты |

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Greedy** — быстро, но только если доказана корректность жадного выбора
> - **DP** — всегда корректно при перекрывающихся подзадачах
> - Шаблон DP: определи состояние → формула перехода → base case → порядок заполнения
> - **Kadane** — O(n) максимальная сумма подмассива, классика
> - **0/1 Knapsack** — базовый паттерн 2D DP
> - **LCS** — паттерн сравнения двух строк/последовательностей
> - Top-Down = рекурсия + HashMap/массив кэш
> - Bottom-Up = цикл + массив, нет риска StackOverflow
