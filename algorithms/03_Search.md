> **Теги:** #algorithms #search #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]]

# 🔍 Алгоритмы поиска

---

## 🔹 Линейный поиск (Linear Search)

> [!note] Принцип
> Перебор элементов **по одному** с начала до конца. Работает на любом массиве — отсортированном и нет.

```java
int linearSearch(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == target) return i; // нашли — вернуть индекс
    }
    return -1; // не нашли
}
```

| Случай | Сложность |
|--------|-----------|
| Best | O(1) — первый элемент |
| Average | O(n/2) → O(n) |
| Worst | O(n) — последний или отсутствует |
| Space | O(1) |

---

## 🔹 Бинарный поиск (Binary Search)

> [!note] Принцип
> Работает только на **отсортированном** массиве. Каждый шаг делит область поиска **пополам**, сравнивая с серединным элементом.

```
arr = [1, 3, 5, 7, 9, 11, 13]
target = 7

Шаг 1: lo=0, hi=6, mid=3 → arr[3]=7 → найден!

target = 11
Шаг 1: lo=0, hi=6, mid=3 → arr[3]=7 < 11 → lo=4
Шаг 2: lo=4, hi=6, mid=5 → arr[5]=11 → найден!
```

### Итеративная реализация
```java
int binarySearch(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2; // безопасно от переполнения

        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

> [!warning] Частая ошибка
> `int mid = (lo + hi) / 2` — может переполнить int при больших lo и hi.
> Безопасный вариант: `mid = lo + (hi - lo) / 2`

### Рекурсивная реализация
```java
int binarySearchRec(int[] arr, int target, int lo, int hi) {
    if (lo > hi) return -1;

    int mid = lo + (hi - lo) / 2;

    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) return binarySearchRec(arr, target, mid + 1, hi);
    else return binarySearchRec(arr, target, lo, mid - 1);
}
```

| Случай | Сложность |
|--------|-----------|
| Best | O(1) — середина массива |
| Average / Worst | O(log n) |
| Space (итеративно) | O(1) |
| Space (рекурсивно) | O(log n) — стек |

---

## 🔹 Встроенный бинарный поиск в Java

```java
import java.util.Arrays;
import java.util.Collections;

int[] arr = {1, 3, 5, 7, 9};
int idx = Arrays.binarySearch(arr, 7);  // возвращает индекс

List<Integer> list = Arrays.asList(1, 3, 5, 7, 9);
int idx2 = Collections.binarySearch(list, 5);
```

> [!warning]
> Если элемент не найден, `Arrays.binarySearch` возвращает `-(insertion_point) - 1`, а не `-1`.

---

## 🔹 Бинарный поиск по условию (шаблон)

Часто на собеседованиях нужен бинарный поиск не по значению, а по **предикату** — найти границу.

```java
// Найти первый индекс, где arr[i] >= target (левая граница)
int lowerBound(int[] arr, int target) {
    int lo = 0, hi = arr.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] < target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}

// Найти первый индекс, где arr[i] > target (правая граница)
int upperBound(int[] arr, int target) {
    int lo = 0, hi = arr.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] <= target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}
```

---

## 🔹 Сравнение алгоритмов поиска

| Алгоритм | Требует сортировки | Time (worst) | Space |
|----------|-------------------|--------------|-------|
| Линейный | ❌ | O(n) | O(1) |
| Бинарный | ✅ | O(log n) | O(1) |

> [!tip] Когда что использовать
> - Данные **не отсортированы** и поиск разовый → линейный
> - Данные **отсортированы** или поиск многократный → бинарный (предварительно отсортировать за O(n log n))
> - Поиск в строке / тексте → `String.contains()`, `indexOf()`

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Линейный** — O(n), не требует сортировки, прост в реализации
> - **Бинарный** — O(log n), только отсортированный массив
> - `mid = lo + (hi - lo) / 2` — всегда так, безопасно
> - Встроенный `Arrays.binarySearch` возвращает отрицательное значение при отсутствии
> - Шаблон lower/upper bound — для поиска границ диапазона
