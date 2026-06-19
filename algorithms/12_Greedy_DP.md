> **Теги:** #algorithms #dynamic-programming #greedy #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]]

# 🧩 Dynamic Programming & Greedy

---

## 🔹 Часть 1: Dynamic Programming

---

## 🔹 Что такое DP и когда применять

DP — это способ решать задачи, разбивая их на **перекрывающиеся подзадачи** и сохраняя результаты, чтобы не пересчитывать их заново.

Два ключевых признака что задача решается DP:

1. **Optimal Substructure** — оптимальное решение задачи состоит из оптимальных решений подзадач
2. **Overlapping Subproblems** — одни и те же подзадачи встречаются многократно

```
Пример: Fibonacci
fib(5) = fib(4) + fib(3)
fib(4) = fib(3) + fib(2)
fib(3) = fib(2) + fib(1)
            ↑
     fib(2) и fib(3) вычисляются много раз → сохраняем результат
```

**DP vs Greedy:**
- **Greedy:** на каждом шаге выбирает локально оптимальное решение, никогда не пересматривает
- **DP:** рассматривает все возможные решения и выбирает глобально оптимальное

---

## 🔹 Два подхода: мемоизация vs табуляция

### Мемоизация (Top-Down) — рекурсия + кэш

Пишем рекурсию "сверху вниз" как обычно, добавляем кэш результатов:

```java
// Fibonacci: O(2^n) без DP → O(n) с мемоизацией
int fib(int n, int[] memo) {
    if (n <= 1) return n;
    if (memo[n] != 0) return memo[n];  // уже считали
    memo[n] = fib(n-1, memo) + fib(n-2, memo);
    return memo[n];
}
// Вызов: fib(n, new int[n+1])
```

**Преимущества:**
- Код близок к рекурсивному решению — легко понять
- Вычисляет только нужные подзадачи (lazy)

**Недостатки:**
- Overhead рекурсии (стек вызовов)
- StackOverflow на больших n

### Табуляция (Bottom-Up) — итерация + таблица

Заполняем таблицу результатов "снизу вверх" итеративно:

```java
// Fibonacci: Bottom-Up
int fib(int n) {
    if (n <= 1) return n;
    int[] dp = new int[n + 1];
    dp[0] = 0;
    dp[1] = 1;
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i-1] + dp[i-2];
    }
    return dp[n];
}

// Оптимизация памяти: не нужна вся таблица, только 2 предыдущих значения
int fibOptimal(int n) {
    if (n <= 1) return n;
    int prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
```

**Преимущества:**
- Нет рекурсии → нет StackOverflow
- Обычно быстрее (нет overhead стека)
- Можно оптимизировать память

| | Мемоизация (Top-Down) | Табуляция (Bottom-Up) |
|-|----------------------|----------------------|
| Направление | Сверху вниз | Снизу вверх |
| Реализация | Рекурсия + Map/array | Циклы + array |
| Вычисляет | Только нужные подзадачи | Все подзадачи |
| Память | Стек + кэш | Только таблица |
| Когда | Сложная логика, не все подзадачи нужны | Простая зависимость, большие n |

---

## 🔹 Алгоритм решения DP задачи

```
1. Определить состояние (что означает dp[i] или dp[i][j])
2. Найти рекуррентное соотношение (как dp[i] зависит от предыдущих)
3. Определить базовые случаи (начальные значения)
4. Определить порядок вычисления (bottom-up: от чего зависим — считаем раньше)
5. Извлечь ответ (dp[n], dp[m][n] или переменная)
```

---

## 🔹 Классическая задача 1: Knapsack (Рюкзак)

### 0/1 Knapsack — каждый предмет берём один раз или не берём

```
Есть N предметов: каждый имеет вес weight[i] и ценность value[i]
Рюкзак вмещает W кг. Максимизировать суммарную ценность.
```

```java
// dp[i][w] = максимальная ценность используя первые i предметов при вместимости w
int knapsack01(int[] weight, int[] value, int W) {
    int n = weight.length;
    int[][] dp = new int[n + 1][W + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            // Не берём i-й предмет
            dp[i][w] = dp[i-1][w];

            // Берём i-й предмет (если помещается)
            if (weight[i-1] <= w) {
                dp[i][w] = Math.max(dp[i][w],
                    dp[i-1][w - weight[i-1]] + value[i-1]);
            }
        }
    }
    return dp[n][W];
}
// Time: O(n*W), Space: O(n*W)

// Оптимизация памяти до O(W): один массив, идём справа налево
int knapsack01Optimized(int[] weight, int[] value, int W) {
    int[] dp = new int[W + 1];

    for (int i = 0; i < weight.length; i++) {
        // Идём СПРАВА НАЛЕВО — чтобы не использовать предмет дважды
        for (int w = W; w >= weight[i]; w--) {
            dp[w] = Math.max(dp[w], dp[w - weight[i]] + value[i]);
        }
    }
    return dp[W];
}
```

**Трассировка примера:**
```
weight = [2, 3, 4], value = [3, 4, 5], W = 5

dp после предмета 1 (w=2, v=3):
  dp[0..1]=0, dp[2..5]=3

dp после предмета 2 (w=3, v=4):
  dp[5]=max(3, dp[2]+4)=max(3,7)=7 ← взяли оба!

Ответ: dp[5] = 7 (предметы 1 и 2)
```

### Unbounded Knapsack — можно брать предмет сколько угодно раз

```java
int unboundedKnapsack(int[] weight, int[] value, int W) {
    int[] dp = new int[W + 1];

    for (int w = 1; w <= W; w++) {
        for (int i = 0; i < weight.length; i++) {
            if (weight[i] <= w) {
                // Идём СЛЕВА НАПРАВО — предмет можно взять снова
                dp[w] = Math.max(dp[w], dp[w - weight[i]] + value[i]);
            }
        }
    }
    return dp[W];
}
```

> [!tip] 0/1 vs Unbounded Knapsack
> **0/1:** внешний цикл по предметам, внутренний по весу **справа налево** — каждый предмет берётся максимум раз.
> **Unbounded:** внешний цикл по весу, внутренний по предметам **слева направо** — предмет можно брать снова.

---

## 🔹 Классическая задача 2: Coin Change

### Минимальное количество монет

```
Дано: монеты [1, 5, 10, 25], сумма amount
Найти: минимальное количество монет для суммы amount
```

```java
// dp[i] = минимальное количество монет для суммы i
int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // инициализация "бесконечностью"
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
// Time: O(amount * coins.length), Space: O(amount)
```

**Трассировка:** coins=[1,2,5], amount=11
```
dp[0]=0
dp[1]=min(dp[0]+1)=1         (монета 1)
dp[2]=min(dp[1]+1,dp[0]+1)=1 (монета 2)
dp[3]=min(dp[2]+1,dp[1]+1)=2 (монета 1+2)
...
dp[11]=min(dp[10]+1,dp[9]+1,dp[6]+1)=3 (5+5+1)
```

### Количество способов набрать сумму

```java
// dp[i] = количество комбинаций монет для суммы i
int changeCount(int amount, int[] coins) {
    int[] dp = new int[amount + 1];
    dp[0] = 1; // один способ набрать 0 — ничего не брать

    for (int coin : coins) {          // внешний цикл по монетам
        for (int i = coin; i <= amount; i++) { // внутренний по сумме
            dp[i] += dp[i - coin];
        }
    }
    return dp[amount];
}
// Порядок важен! Монеты снаружи → каждая комбинация считается один раз
// Если суммы снаружи → считаются перестановки (1+2 и 2+1 — разные)
```

---

## 🔹 Классическая задача 3: Longest Common Subsequence (LCS)

```
Дано: "ABCBDAB" и "BDCAB"
Найти: длину наибольшей общей подпоследовательности → "BCAB" или "BDAB" → длина 4
```

```java
// dp[i][j] = LCS для s1[0..i-1] и s2[0..j-1]
int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i-1) == s2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1] + 1;  // символы совпали → берём оба
            } else {
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]); // берём лучшее из пропуска
            }
        }
    }
    return dp[m][n];
}
// Time: O(m*n), Space: O(m*n)
```

**Визуализация таблицы для "ACE" и "ABCDE":**
```
    ""  A  B  C  D  E
""  [0][0][0][0][0][0]
A   [0][1][1][1][1][1]
C   [0][1][1][2][2][2]
E   [0][1][1][2][2][3]  ← ответ: dp[3][5] = 3
```

**LCS как основа других задач:**
- **Edit Distance** (расстояние Левенштейна) — минимум операций для превращения одной строки в другую
- **Diff** (git diff) — построен на LCS
- **Shortest Common Supersequence** — объединить, сохраняя порядок

---

## 🔹 Классическая задача 4: Longest Palindromic Subsequence / Substring

### Longest Palindromic Substring — непрерывная подстрока

```java
// Подход "expand around center": O(n²) time, O(1) space
String longestPalindrome(String s) {
    if (s.length() < 2) return s;
    int start = 0, maxLen = 1;

    for (int i = 0; i < s.length(); i++) {
        // Нечётная длина: центр в i
        int len1 = expandAroundCenter(s, i, i);
        // Чётная длина: центр между i и i+1
        int len2 = expandAroundCenter(s, i, i + 1);

        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = i - (len - 1) / 2;
        }
    }
    return s.substring(start, start + maxLen);
}

int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1;
}
```

### Longest Palindromic Subsequence — не обязательно непрерывная

```java
// dp[i][j] = длина наибольшей палиндромной подпоследовательности в s[i..j]
int longestPalindromicSubseq(String s) {
    int n = s.length();
    int[][] dp = new int[n][n];

    // Базовый случай: каждый символ — палиндром длины 1
    for (int i = 0; i < n; i++) dp[i][i] = 1;

    // Заполняем по возрастанию длины подстроки
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            if (s.charAt(i) == s.charAt(j)) {
                dp[i][j] = dp[i+1][j-1] + 2;  // крайние совпадают
            } else {
                dp[i][j] = Math.max(dp[i+1][j], dp[i][j-1]); // пропускаем один
            }
        }
    }
    return dp[0][n-1];
}
// "bbbab" → dp[0][4] = 4 (подпоследовательность "bbbb")
```

---

## 🔹 Классическая задача 5: Задачи на подмассивы

### Maximum Subarray (Kadane's Algorithm)

```java
// Найти подмассив с максимальной суммой
int maxSubArray(int[] nums) {
    int maxSum = nums[0];
    int currentSum = nums[0];

    for (int i = 1; i < nums.length; i++) {
        // Либо продолжаем текущий подмассив, либо начинаем новый с i
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
// [-2,1,-3,4,-1,2,1,-5,4] → 6 (подмассив [4,-1,2,1])
```

### Longest Increasing Subsequence (LIS)

```java
// O(n²) DP — понятнее
int lisDP(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n]; // dp[i] = длина LIS заканчивающейся в i
    Arrays.fill(dp, 1);

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

// O(n log n) — бинарный поиск (patience sorting)
int lisOptimal(int[] nums) {
    List<Integer> tails = new ArrayList<>(); // tails[i] = наименьший хвост LIS длины i+1

    for (int num : nums) {
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1); // место вставки

        if (pos == tails.size()) tails.add(num); // новая длина
        else tails.set(pos, num);                // заменяем на меньший
    }
    return tails.size();
}
// [10,9,2,5,3,7,101,18] → 4 ([2,3,7,101] или [2,5,7,101])
```

---

## 🔹 Задачи на матрицах (2D DP)

### Unique Paths — количество путей в матрице

```java
// Из (0,0) в (m-1,n-1), только вправо и вниз
int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];

    // Первая строка и первый столбец — только один путь
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;

    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = dp[i-1][j] + dp[i][j-1]; // пришли сверху + пришли слева

    return dp[m-1][n-1];
}
```

### Minimum Path Sum

```java
// Найти путь с минимальной суммой от (0,0) до (m-1,n-1)
int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];
    dp[0][0] = grid[0][0];

    for (int i = 1; i < m; i++) dp[i][0] = dp[i-1][0] + grid[i][0];
    for (int j = 1; j < n; j++) dp[0][j] = dp[0][j-1] + grid[0][j];

    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = grid[i][j] + Math.min(dp[i-1][j], dp[i][j-1]);

    return dp[m-1][n-1];
}
```

### Maximal Square — наибольший квадрат из единиц

```java
// dp[i][j] = сторона наибольшего квадрата, заканчивающегося в (i,j)
int maximalSquare(char[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] dp = new int[m][n];
    int maxSide = 0;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (matrix[i][j] == '1') {
                if (i == 0 || j == 0) {
                    dp[i][j] = 1;
                } else {
                    // Квадрат ограничен наименьшим из трёх соседей
                    dp[i][j] = Math.min(dp[i-1][j],
                               Math.min(dp[i][j-1], dp[i-1][j-1])) + 1;
                }
                maxSide = Math.max(maxSide, dp[i][j]);
            }
        }
    }
    return maxSide * maxSide; // площадь
}
```

---

## 🔹 Другие классические задачи

### Jump Game — можно ли добраться до конца

```java
// Жадный подход: отслеживаем максимально достижимую позицию
boolean canJump(int[] nums) {
    int maxReach = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false;        // не можем добраться до i
        maxReach = Math.max(maxReach, i + nums[i]);
    }
    return true;
}
```

### House Robber — ограбить дома без смежных

```java
// dp[i] = максимум ограблений для первых i домов
int rob(int[] nums) {
    if (nums.length == 1) return nums[0];
    int prev2 = nums[0];
    int prev1 = Math.max(nums[0], nums[1]);

    for (int i = 2; i < nums.length; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]); // грабить текущий или пропустить
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
```

### Word Break — можно ли составить строку из словаря

```java
// dp[i] = можно ли составить s[0..i-1] из слов словаря
boolean wordBreak(String s, List<String> wordDict) {
    Set<String> dict = new HashSet<>(wordDict);
    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true; // пустая строка

    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && dict.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    return dp[n];
}
// "leetcode", ["leet","code"] → true
```

---

## 🔹 Часть 2: Greedy (Жадные алгоритмы)

---

## 🔹 Когда работает жадный подход

Жадный алгоритм правильно работает когда есть **greedy choice property**: локально оптимальный выбор ведёт к глобально оптимальному результату.

```
Задача: дать сдачу 41 монетами {25, 10, 5, 1}
Жадный: 25+10+5+1=4 монеты → ПРАВИЛЬНО

Задача: дать сдачу 11 монетами {7, 5, 1}
Жадный: 7+1+1+1 = 4 монеты → НЕПРАВИЛЬНО
  DP: 5+5+1 = 3 монеты → правильный ответ
```

---

## 🔹 Классические жадные задачи

### Activity Selection — максимум несовместимых событий

```java
// Задача: выбрать максимум событий, чтобы они не перекрывались
// Жадная стратегия: сортировать по времени окончания, брать если не конфликтует

int maxActivities(int[] start, int[] end) {
    int n = start.length;
    Integer[] idx = IntStream.range(0, n).boxed().toArray(Integer[]::new);
    Arrays.sort(idx, (a, b) -> end[a] - end[b]); // сортировка по end

    int count = 1;
    int lastEnd = end[idx[0]];

    for (int i = 1; i < n; i++) {
        if (start[idx[i]] >= lastEnd) { // не конфликтует
            count++;
            lastEnd = end[idx[i]];
        }
    }
    return count;
}
```

### Interval Merging — слияние пересекающихся интервалов

```java
int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]); // сортируем по началу
    List<int[]> result = new ArrayList<>();

    int[] current = intervals[0];
    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] <= current[1]) {
            // Пересекаются — расширяем текущий
            current[1] = Math.max(current[1], intervals[i][1]);
        } else {
            result.add(current);
            current = intervals[i];
        }
    }
    result.add(current);
    return result.toArray(new int[0][]);
}
```

### Jump Game II — минимум прыжков

```java
// Жадный: на каждом уровне прыгаем максимально далеко
int jump(int[] nums) {
    int jumps = 0, currentEnd = 0, farthest = 0;

    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);
        if (i == currentEnd) { // достигли границы текущего прыжка
            jumps++;
            currentEnd = farthest;
        }
    }
    return jumps;
}
```

### Meeting Rooms II — минимум комнат для встреч

```java
// Жадный с min-heap: сколько встреч идёт одновременно
int minMeetingRooms(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]); // по времени начала

    PriorityQueue<Integer> ends = new PriorityQueue<>(); // время окончания занятых комнат

    for (int[] interval : intervals) {
        if (!ends.isEmpty() && ends.peek() <= interval[0]) {
            ends.poll(); // освободилась комната
        }
        ends.offer(interval[1]); // занять комнату до interval[1]
    }
    return ends.size();
}
```

---

## 🔹 Шаблоны DP задач

```
Задача на одномерном массиве:
  dp[i] зависит от dp[i-1] или dp[i-k]
  → 1D DP, часто оптимизируется до O(1) памяти (prev1, prev2)
  Примеры: Fibonacci, House Robber, Jump Game

Задача на двух строках/последовательностях:
  dp[i][j] зависит от dp[i-1][j], dp[i][j-1], dp[i-1][j-1]
  → 2D DP таблица
  Примеры: LCS, Edit Distance, Knapsack

Задача на подстроке/подмассиве:
  dp[i][j] = ответ для s[i..j]
  Заполняем по возрастанию длины
  → Interval DP
  Примеры: Longest Palindromic Subsequence, Burst Balloons

Задача на выборе: брать / не брать:
  dp[i][w] — типичный 0/1 Knapsack
  0/1 → внешний цикл предметы, внутренний справа налево
  Unbounded → внутренний слева направо

Задача на подсчёте путей:
  dp[i][j] = dp[i-1][j] + dp[i][j-1]
  Примеры: Unique Paths, Pascal's Triangle
```

---

## 🔹 Итог

```
Dynamic Programming:
  Когда: перекрывающиеся подзадачи + optimal substructure
  
  Мемоизация (Top-Down): рекурсия + кэш
    + Код близок к рекурсивному, считает только нужные подзадачи
    - Overhead рекурсии, StackOverflow на больших n
  
  Табуляция (Bottom-Up): итерация + таблица
    + Быстрее, нет StackOverflow, легко оптимизировать память
    - Нужно явно определить порядок заполнения

Алгоритм решения:
  1. Определить состояние dp[i] / dp[i][j]
  2. Рекуррентное соотношение
  3. Базовые случаи
  4. Порядок вычисления (bottom-up)
  5. Извлечь ответ

Классические задачи:
  0/1 Knapsack    → dp[i][w], справа налево для одного предмета
  Coin Change     → dp[amount], минимум монет
  LCS             → dp[i][j], две строки
  LIS             → dp[i], O(n log n) через бинарный поиск
  Max Subarray    → Kadane's O(n)
  Palindrome      → expand around center / interval DP
  Unique Paths    → dp[i][j] = dp[i-1][j] + dp[i][j-1]

Greedy:
  Когда: local optimal = global optimal (не всегда очевидно!)
  Проверка: если жадный даёт неверный ответ → нужен DP
  
  Activity Selection → сортировка по времени окончания
  Interval Merging   → сортировка по началу
  Jump Game          → отслеживать maxReach
  Meeting Rooms      → min-heap с временами окончания
```
