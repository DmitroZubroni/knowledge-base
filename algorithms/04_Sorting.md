> **Теги:** #algorithms #sorting #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[00_Algorithms]]

# 🔃 Алгоритмы сортировки

---

## 🔹 Сравнительная таблица

| Алгоритм | Best | Average | Worst | Space | Stable |
|----------|------|---------|-------|-------|--------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |

> [!note] Стабильная сортировка
> **Стабильная** — сохраняет относительный порядок одинаковых элементов.

---

## 🔹 Bubble Sort (пузырьковая)

> [!note] Принцип
> Сравниваем соседние элементы, меняем местами если нужно. Наибольший элемент "всплывает" в конец за каждый проход.

```java
void bubbleSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
            }
        }
        if (!swapped) break; // оптимизация: массив уже отсортирован
    }
}
```

```
[5, 3, 1, 4, 2]
Проход 1: [3,1,4,2, 5]  ← 5 встал на место
Проход 2: [1,3,2, 4, 5] ← 4 встал на место
...
```

---

## 🔹 Selection Sort (сортировка выбором)

> [!note] Принцип
> Ищем минимальный элемент в оставшейся части, ставим на текущую позицию.

```java
void selectionSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            if (arr[j] < arr[minIdx]) minIdx = j;
        }
        int temp = arr[i];
        arr[i] = arr[minIdx];
        arr[minIdx] = temp;
    }
}
```

> [!warning] Минус Selection Sort
> Всегда O(n²) — нет оптимизации для уже отсортированных данных. Не стабильная.

---

## 🔹 Insertion Sort (сортировка вставкой)

> [!note] Принцип
> Строим отсортированную часть слева. Каждый новый элемент "вставляем" на правильное место.

```java
void insertionSort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int key = arr[i];
        int j = i - 1;
        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;
    }
}
```

```
[5, 3, 1, 4, 2]
i=1: key=3 → [3, 5, 1, 4, 2]
i=2: key=1 → [1, 3, 5, 4, 2]
i=3: key=4 → [1, 3, 4, 5, 2]
i=4: key=2 → [1, 2, 3, 4, 5]
```

> [!tip] Когда использовать
> Insertion Sort отлично работает на **почти отсортированных** данных — O(n) в лучшем случае. Используется в гибридных алгоритмах (Timsort).

---

## 🔹 Merge Sort (сортировка слиянием)

> [!note] Принцип
> Делим массив пополам рекурсивно до одного элемента, затем **сливаем** отсортированные части.

```java
void mergeSort(int[] arr, int left, int right) {
    if (left >= right) return;

    int mid = left + (right - left) / 2;
    mergeSort(arr, left, mid);
    mergeSort(arr, mid + 1, right);
    merge(arr, left, mid, right);
}

void merge(int[] arr, int left, int mid, int right) {
    int n1 = mid - left + 1;
    int n2 = right - mid;

    int[] L = new int[n1];
    int[] R = new int[n2];

    System.arraycopy(arr, left, L, 0, n1);
    System.arraycopy(arr, mid + 1, R, 0, n2);

    int i = 0, j = 0, k = left;
    while (i < n1 && j < n2) {
        if (L[i] <= R[j]) arr[k++] = L[i++];
        else arr[k++] = R[j++];
    }
    while (i < n1) arr[k++] = L[i++];
    while (j < n2) arr[k++] = R[j++];
}

// Вызов:
mergeSort(arr, 0, arr.length - 1);
```

```
[5, 3, 1, 4, 2]
       ↓ divide
  [5,3,1]  [4,2]
  [5,3][1] [4][2]
  [5][3]
       ↓ merge
  [3,5][1] → [1,3,5]
  [2,4]
       ↓ merge
  [1,2,3,4,5]
```

---

## 🔹 Quick Sort (быстрая сортировка)

> [!note] Принцип
> Выбираем **опорный элемент** (pivot), разбиваем массив на "меньше pivot" и "больше pivot", рекурсивно сортируем части.

```java
void quickSort(int[] arr, int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi - 1);
        quickSort(arr, pi + 1, high);
    }
}

int partition(int[] arr, int low, int high) {
    int pivot = arr[high]; // pivot — последний элемент
    int i = low - 1;

    for (int j = low; j < high; j++) {
        if (arr[j] <= pivot) {
            i++;
            int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
        }
    }
    int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp;
    return i + 1;
}

// Вызов:
quickSort(arr, 0, arr.length - 1);
```

> [!warning] Worst case O(n²)
> Возникает когда pivot всегда минимальный/максимальный (уже отсортированный массив + pivot = последний элемент). Решение: **случайный pivot** или выбор медианы.

```java
// Случайный pivot — защита от worst case
Random rand = new Random();
int randomIdx = low + rand.nextInt(high - low + 1);
int temp = arr[randomIdx]; arr[randomIdx] = arr[high]; arr[high] = temp;
// затем стандартный partition
```

---

## 🔹 Встроенная сортировка Java

```java
import java.util.Arrays;
import java.util.Collections;

// Примитивы — Dual-Pivot QuickSort
int[] arr = {5, 3, 1};
Arrays.sort(arr);                    // по возрастанию
Arrays.sort(arr, 1, 3);             // диапазон [1..3)

// Объекты и коллекции — TimSort (стабильный)
Integer[] boxed = {5, 3, 1};
Arrays.sort(boxed);
Arrays.sort(boxed, Comparator.reverseOrder());

List<Integer> list = Arrays.asList(5, 3, 1);
Collections.sort(list);
list.sort(Comparator.reverseOrder());

// Кастомная сортировка
Arrays.sort(boxed, (a, b) -> b - a); // по убыванию
```

> [!note] TimSort
> Java использует **TimSort** для объектов — гибрид Merge Sort + Insertion Sort. O(n log n) worst case, стабильный, отлично работает на реальных данных с уже отсортированными участками.

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Bubble / Selection / Insertion** — O(n²), только для малых n или учебных целей
> - **Insertion** — лучший из простых, работает быстро на почти отсортированных данных
> - **Merge Sort** — гарантированный O(n log n), стабильный, но O(n) памяти
> - **Quick Sort** — в среднем быстрее Merge Sort, O(log n) памяти, но worst case O(n²)
> - **На практике** — используй `Arrays.sort()` / `Collections.sort()` (TimSort)
