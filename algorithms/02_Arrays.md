> **Теги:** #algorithms #arrays #конспект
> [!abstract] Связи
> [[main]] | [[main Algorithms]] | [[00_Algorithms]]

# 📦 Массивы (Arrays)

---

## 🔹 Что такое массив

> [!note] Определение
> **Массив** — структура данных, хранящая элементы одного типа в **непрерывной** области памяти. Доступ к элементу — по индексу за O(1).

```
index:  0    1    2    3    4
value: [10] [20] [30] [40] [50]
addr:  100  104  108  112  116   (int = 4 байта)
```

---

## 🔹 Статический массив

```java
// Объявление и инициализация
int[] arr = new int[5];
int[] arr = {10, 20, 30, 40, 50};

// Доступ
int x = arr[2];       // O(1) — чтение
arr[2] = 99;          // O(1) — запись

// Длина
int len = arr.length; // поле, не метод
```

| Операция | Сложность | Причина |
|----------|-----------|---------|
| get(i) | O(1) | прямой адрес = base + i * size |
| set(i) | O(1) | то же |
| search | O(n) | линейный проход |
| insert (середина) | O(n) | сдвиг элементов |
| delete (середина) | O(n) | сдвиг элементов |

---

## 🔹 Динамический массив — ArrayList

> [!note] Принцип работы
> `ArrayList` хранит внутренний статический массив. При заполнении создаёт новый массив в **1.5x** раз больше и копирует элементы.

```java
import java.util.ArrayList;

ArrayList<Integer> list = new ArrayList<>();
list.add(10);           // O(1) амортизированно
list.add(1, 99);        // O(n) — вставка в середину
list.get(2);            // O(1)
list.remove(2);         // O(n) — удаление из середины
list.size();            // O(1)
list.contains(99);      // O(n)
list.indexOf(99);       // O(n)
```

### ❌ Плохо — частая вставка в начало
```java
// Каждый add(0, x) сдвигает все элементы → O(n²) в итоге
for (int i = 0; i < 1000; i++) {
    list.add(0, i);
}
```

### ✅ Хорошо — вставка в конец
```java
// add() в конец — O(1) амортизированно
for (int i = 0; i < 1000; i++) {
    list.add(i);
}
```

---

## 🔹 Амортизированная сложность

> [!note] Амортизация
> Иногда одна операция дорогая (копирование всего массива), но **в среднем по серии операций** — O(1). Это и есть амортизированная сложность.

```
Добавляем 8 элементов в ArrayList (начальная ёмкость = 4):
add(1): обычный  → O(1)
add(2): обычный  → O(1)
add(3): обычный  → O(1)
add(4): обычный  → O(1)
add(5): resize!  → копируем 4 + добавляем 1 = O(5)
add(6): обычный  → O(1)
add(7): обычный  → O(1)
add(8): обычный  → O(1)

Итого: 12 операций на 8 добавлений → ~O(1.5) в среднем → O(1) амортизированно
```

---

## 🔹 Двумерные массивы

```java
int[][] matrix = new int[3][4]; // 3 строки, 4 столбца

int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Обход
for (int i = 0; i < matrix.length; i++) {
    for (int j = 0; j < matrix[i].length; j++) {
        System.out.print(matrix[i][j] + " ");
    }
}
```

---

## 🔹 Полезные методы Arrays

```java
import java.util.Arrays;

int[] arr = {5, 3, 1, 4, 2};

Arrays.sort(arr);                    // O(n log n) — сортировка
Arrays.binarySearch(arr, 3);        // O(log n) — поиск (массив должен быть отсортирован)
Arrays.fill(arr, 0);                 // O(n) — заполнить значением
Arrays.copyOf(arr, arr.length);      // O(n) — копия
Arrays.copyOfRange(arr, 1, 4);       // O(k) — копия диапазона
Arrays.equals(arr1, arr2);           // O(n) — сравнение
System.out.println(Arrays.toString(arr)); // вывод массива
```

---

## 🔹 Типичные задачи на массивы

### Разворот массива на месте
```java
void reverse(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int temp = arr[left];
        arr[left] = arr[right];
        arr[right] = temp;
        left++;
        right--;
    }
}
// O(n) time, O(1) space
```

### Нахождение дубликатов через HashSet
```java
boolean hasDuplicate(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    for (int x : arr) {
        if (!seen.add(x)) return true;
    }
    return false;
}
// O(n) time, O(n) space
```

### Two Pointers — два указателя
```java
// Проверка: является ли строка палиндромом
boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) return false;
        left++;
        right--;
    }
    return true;
}
```

### Sliding Window — скользящее окно
```java
// Максимальная сумма подмассива длиной k
int maxSum(int[] arr, int k) {
    int sum = 0;
    for (int i = 0; i < k; i++) sum += arr[i];
    int max = sum;
    for (int i = k; i < arr.length; i++) {
        sum += arr[i] - arr[i - k];
        max = Math.max(max, sum);
    }
    return max;
}
// O(n) time, O(1) space
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Статический массив** — фиксированный размер, O(1) доступ, O(n) вставка/удаление
> - **ArrayList** — авторесайз (x1.5), O(1) амортизированно добавление в конец
> - **Two Pointers** — для задач с парами/палиндромами/разворотом, O(n) за один проход
> - **Sliding Window** — для задач с подмассивами фиксированной длины
> - Избегай вставок в начало ArrayList — это O(n)
