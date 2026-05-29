> **Теги:** #algorithms #recursion #memoization #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[main Algorithms]]

# 🔄 Рекурсия и Мемоизация

---

## 🔹 Что такое рекурсия

> [!note] Определение
> **Рекурсия** — когда функция вызывает саму себя. Каждый рекурсивный алгоритм содержит:
> 1. **Base case** — условие остановки
> 2. **Recursive case** — вызов себя с меньшим входом

```java
// Классический пример — факториал
int factorial(int n) {
    if (n <= 1) return 1;           // base case
    return n * factorial(n - 1);   // recursive case
}
// factorial(4) = 4 * factorial(3)
//              = 4 * 3 * factorial(2)
//              = 4 * 3 * 2 * factorial(1)
//              = 4 * 3 * 2 * 1 = 24
```

---

## 🔹 Стек вызовов (Call Stack)

Каждый вызов функции создаёт **фрейм** в стеке. При рекурсии глубиной n — n фреймов.

```
factorial(4)
  └─ factorial(3)
       └─ factorial(2)
            └─ factorial(1) → return 1
         return 2 * 1 = 2
    return 3 * 2 = 6
  return 4 * 6 = 24
```

> [!warning] StackOverflowError
> При слишком большой глубине рекурсии JVM переполняет стек и бросает `StackOverflowError`. Дефолтный лимит ~500-1000 вызовов.

```java
// ❌ Бесконечная рекурсия — StackOverflowError
int bad(int n) {
    return bad(n - 1); // нет base case!
}
```

---

## 🔹 Числа Фибоначчи — наивная рекурсия

```java
// ❌ Наивная рекурсия — O(2ⁿ) — экспоненциальная!
int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}
// fib(5) вычисляет fib(3) дважды, fib(2) трижды и т.д.
```

```
            fib(5)
          /        \
       fib(4)      fib(3)
      /    \       /    \
   fib(3) fib(2) fib(2) fib(1)  ← много повторений
```

---

## 🔹 Мемоизация (Top-Down DP)

> [!note] Мемоизация
> Кэшируем результаты вычислений, чтобы не пересчитывать одно и то же. Превращает O(2ⁿ) в O(n).

```java
// ✅ Мемоизация — O(n) time, O(n) space
int[] memo = new int[100];
Arrays.fill(memo, -1);

int fib(int n) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n]; // уже считали
    memo[n] = fib(n - 1) + fib(n - 2);
    return memo[n];
}
```

```java
// Через HashMap для произвольного ключа
Map<Integer, Integer> cache = new HashMap<>();

int fib(int n) {
    if (n <= 1) return n;
    if (cache.containsKey(n)) return cache.get(n);
    int result = fib(n - 1) + fib(n - 2);
    cache.put(n, result);
    return result;
}
```

---

## 🔹 Динамическое программирование Bottom-Up (Tabulation)

> [!note] Bottom-Up DP
> Заполняем таблицу результатов **снизу вверх**, без рекурсии. Нет стека вызовов — эффективнее по памяти.

```java
// ✅ Bottom-Up DP — O(n) time, O(n) space
int fib(int n) {
    if (n <= 1) return n;
    int[] dp = new int[n + 1];
    dp[0] = 0; dp[1] = 1;
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];
}

// ✅ Оптимизация по памяти — O(1) space
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

---

## 🔹 Сравнение подходов

| Подход | Time | Space | Особенность |
|--------|------|-------|-------------|
| Наивная рекурсия | O(2ⁿ) | O(n) стек | Просто, но неприемлемо медленно |
| Мемоизация (Top-Down) | O(n) | O(n) кэш + O(n) стек | Интуитивно, легко писать |
| Tabulation (Bottom-Up) | O(n) | O(n) таблица | Нет риска StackOverflow |
| Bottom-Up оптимизированный | O(n) | O(1) | Лучший вариант |

---

## 🔹 Хвостовая рекурсия (Tail Recursion)

> [!note] Хвостовая рекурсия
> Рекурсивный вызов — **последняя операция** в функции. Компилятор (не JVM) может оптимизировать в цикл.

```java
// Обычная рекурсия — не хвостовая (умножение после вызова)
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1); // операция * выполняется ПОСЛЕ вызова
}

// Хвостовая рекурсия — передаём аккумулятор
int factorial(int n, int acc) {
    if (n <= 1) return acc;
    return factorial(n - 1, n * acc); // вызов — ПОСЛЕДНЯЯ операция
}
// factorial(4, 1) → factorial(3, 4) → factorial(2, 12) → factorial(1, 24) → 24
```

> [!warning] Java и хвостовая рекурсия
> JVM **не оптимизирует** хвостовую рекурсию (в отличие от Scala/Kotlin). Поэтому для глубоких рекурсий — лучше явный цикл.

---

## 🔹 Типичные рекурсивные задачи

### Сумма цифр числа
```java
int sumDigits(int n) {
    if (n < 10) return n;
    return n % 10 + sumDigits(n / 10);
}
```

### Мощность (быстрое возведение в степень)
```java
double power(double base, int exp) {
    if (exp == 0) return 1;
    if (exp % 2 == 0) {
        double half = power(base, exp / 2);
        return half * half; // O(log n)!
    }
    return base * power(base, exp - 1);
}
```

### Обход дерева (см. [[09_Trees]])
```java
void inOrder(Node root) {
    if (root == null) return;
    inOrder(root.left);
    System.out.print(root.val);
    inOrder(root.right);
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - Рекурсия всегда имеет **base case** — иначе StackOverflowError
> - **Мемоизация** — кэшируем результаты, превращаем O(2ⁿ) → O(n)
> - **Bottom-Up DP** — заполняем таблицу циклом, нет рекурсии и стека
> - В Java хвостовая рекурсия не оптимизируется — используй цикл для глубоких задач
> - Правило: если видишь рекурсию с повторяющимися подзадачами → добавь мемоизацию
