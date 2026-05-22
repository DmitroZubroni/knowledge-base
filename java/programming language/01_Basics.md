# 01 — Синтаксис и основы Java

> [!abstract] Связи
> [[00_INDEX]] | [[02_OOP_Core]] | [[11_JVM_Internals]]

---

## 🔹 Примитивные типы данных

> [!note] Определение
> Примитивные типы — базовые типы данных Java, хранятся в **стеке** напрямую (не как объекты).

```
Тип       Размер    Диапазон                        Пример
─────────────────────────────────────────────────────────────
byte      1 байт    -128 … 127                      byte b = 100;
short     2 байта   -32 768 … 32 767                short s = 1000;
int       4 байта   -2 147 483 648 … 2 147 483 647  int i = 42;
long      8 байт    ±9.2 × 10¹⁸                     long l = 100L;
float     4 байта   ~±3.4 × 10³⁸ (7 знаков)         float f = 3.14F;
double    8 байт    ~±1.7 × 10³⁰⁸ (15 знаков)       double d = 3.14;
char      2 байта   '\u0000' … '\uFFFF'              char c = 'A';
boolean   1 бит     true / false                    boolean flag = true;
```

> [!warning] Подводные камни
> - `long` требует суффикс `L`: `long x = 10000000000L;` иначе ошибка компиляции
> - `float` требует суффикс `F`: `float f = 3.14F;` иначе присвоится `double`
> - `char` — это **число** от 0 до 65535, можно складывать: `'A' + 1 == 'B'`
> - Деление `int / int` = `int`: `5 / 2 == 2` (не 2.5!)

---

## 🔹 Переменные

> [!note] Определение
> Переменная = именованная ячейка памяти для хранения значения.

```java
// Объявление и инициализация
int age = 25;
String name = "Alice";
double price;      // объявление без инициализации
price = 9.99;      // инициализация позже

// Несколько переменных одного типа
int x = 1, y = 2, z = 3;
```

**Нотация:**
```java
int userName;          // camelCase — переменные и методы
final int MAX_SIZE = 100;  // UPPER_SNAKE_CASE — константы
class MyClass { }      // PascalCase — классы
```

> [!tip] Рекомендации
> - Давай осмысленные имена: `userAge` вместо `ua`
> - Объявляй переменную максимально близко к месту использования
> - Используй `final` для значений, которые не меняются

---

## 🔹 Константы (final)

> [!note] Определение
> `final` переменная — её нельзя переприсвоить после инициализации.

```java
final double PI = 3.14159;
final String APP_NAME = "MyApp";

// PI = 3.0;  // ❌ Ошибка компиляции: cannot assign a value to final variable
```

---

## 🔹 Операторы

### Арифметические
```java
int a = 10, b = 3;
System.out.println(a + b);  // 13
System.out.println(a - b);  // 7
System.out.println(a * b);  // 30
System.out.println(a / b);  // 3  (целочисленное!)
System.out.println(a % b);  // 1  (остаток от деления)

// Инкремент / декремент
int x = 5;
x++;   // x = 6 (постфиксный)
++x;   // x = 7 (префиксный)
x--;   // x = 6
```

### Присваивание
```java
int n = 10;
n += 5;   // n = 15
n -= 3;   // n = 12
n *= 2;   // n = 24
n /= 4;   // n = 6
n %= 4;   // n = 2
```

### Сравнения (возвращают boolean)
```java
int a = 5, b = 10;
a == b   // false
a != b   // true
a > b    // false
a < b    // true
a >= 5   // true
a <= 4   // false
```

> [!warning] Подводные камни
> `==` сравнивает **примитивы по значению**, но **объекты по ссылке**.
> Для строк и объектов всегда используй `.equals()`.

### Логические
```java
boolean a = true, b = false;
a && b   // false  (И — оба true)
a || b   // true   (ИЛИ — хотя бы один true)
!a       // false  (НЕ)

// Short-circuit: правая часть не вычисляется если результат уже известен
// null != null && null.length() > 0  — безопасно, NullPointerException не будет
```

### Тернарный оператор
```java
// condition ? значение_если_true : значение_если_false
int age = 20;
String status = age >= 18 ? "adult" : "minor";
System.out.println(status);  // "adult"

int max = (a > b) ? a : b;
```

### Приоритет операторов (от высокого к низкому)
```
() []                    — скобки (высший)
++ --  (постфикс)
++ --  (префикс)  !  ~
* / %
+ -
< > <= >=
== !=
&&
||
?:                       — тернарный
= += -= *= /= %=         — присваивание (низший)
```

> [!tip] Рекомендация
> Используй скобки для явного указания порядка — это лучше чем полагаться на приоритет.

---

## 🔹 Строки (String)

> [!note] Определение
> `String` — класс (не примитив), строки **иммутабельны** (неизменяемы). Любая операция создаёт новый объект.

```java
String s1 = "Hello";          // из String Pool
String s2 = new String("Hi"); // новый объект в Heap

// Конкатенация
String full = "Hello" + " " + "World";   // "Hello World"
String mixed = "Age: " + 25;             // "Age: 25" (int → String автоматически)
```

### Основные методы String
```java
String s = "  Hello, World!  ";

s.length()                    // 17 — длина строки
s.trim()                      // "Hello, World!" — убрать пробелы по краям
s.strip()                     // "Hello, World!" — Unicode-совместимый trim
s.toUpperCase()               // "  HELLO, WORLD!  "
s.toLowerCase()               // "  hello, world!  "
s.isBlank()                   // false — пустая или только пробелы
s.isEmpty()                   // false — длина == 0

String clean = "Hello, World!";
clean.equals("hello, world!") // false — регистрозависимо
clean.equalsIgnoreCase("hello, world!") // true
clean.contains("World")       // true
clean.startsWith("Hello")     // true
clean.endsWith("!")           // true
clean.indexOf("o")            // 4 — первое вхождение
clean.lastIndexOf("o")        // 8 — последнее вхождение
clean.replace("World", "Java") // "Hello, Java!"
clean.substring(7)             // "World!" — с индекса 7 до конца
clean.substring(7, 12)         // "World" — с 7 до 12 (не включая)
clean.split(", ")              // ["Hello", "World!"]
clean.charAt(0)                // 'H'
clean.toCharArray()            // ['H','e','l','l','o',...]
String.valueOf(42)             // "42" — число в строку
```

> [!warning] Подводные камни
> - `==` для строк сравнивает ссылки: `"abc" == new String("abc")` → `false`
> - Всегда используй `.equals()` для сравнения содержимого строк
> - String immutable: `s.replace("a","b")` не меняет `s`, а создаёт **новую** строку

### StringBuilder — мутабельная строка
```java
// Если много конкатенаций — используй StringBuilder (эффективнее)
StringBuilder sb = new StringBuilder();
sb.append("Hello");
sb.append(", ");
sb.append("World");
String result = sb.toString();  // "Hello, World"
```

---

## 🔹 Ввод данных (Scanner)

```java
import java.util.Scanner;

Scanner sc = new Scanner(System.in);

System.out.print("Введите имя: ");
String name = sc.nextLine();       // читает строку целиком

System.out.print("Введите возраст: ");
int age = sc.nextInt();            // читает int

System.out.print("Введите рост: ");
double height = sc.nextDouble();   // читает double

sc.close();  // закрыть ресурс
```

> [!warning] Подводные камни
> После `nextInt()` или `nextDouble()` в буфере остаётся `\n`.
> Если следующий вызов — `nextLine()`, то он вернёт пустую строку.
> Решение: добавить `sc.nextLine()` после `nextInt()` для очистки буфера.

```java
int age = sc.nextInt();
sc.nextLine(); // очистка буфера
String name = sc.nextLine(); // теперь работает корректно
```

---

## 🔹 Условные операторы

### if / else if / else
```java
int score = 75;

if (score >= 90) {
    System.out.println("Отлично");
} else if (score >= 70) {
    System.out.println("Хорошо");
} else if (score >= 50) {
    System.out.println("Удовлетворительно");
} else {
    System.out.println("Неудовлетворительно");
}
```

### switch (классический)
```java
int day = 3;
switch (day) {
    case 1:
        System.out.println("Понедельник");
        break;  // ❗ Без break — выполнится следующий case!
    case 2:
        System.out.println("Вторник");
        break;
    case 3:
    case 4:
        System.out.println("Среда или четверг"); // два case на один блок
        break;
    default:
        System.out.println("Другой день");
}
```

### switch expression (Java 14+)
```java
String result = switch (day) {
    case 1 -> "Понедельник";
    case 2 -> "Вторник";
    case 3, 4 -> "Среда/четверг";
    default -> "Другой";
};
```

> [!warning] Подводные камни
> Классический `switch` без `break` приведёт к **fall-through** — выполнятся все последующие case. Это частая ошибка.

---

## 🔹 Циклы

### for
```java
// for (инициализация; условие; шаг)
for (int i = 0; i < 5; i++) {
    System.out.println(i);  // 0 1 2 3 4
}

// Обратный порядок
for (int i = 4; i >= 0; i--) {
    System.out.println(i);  // 4 3 2 1 0
}
```

### while
```java
int i = 0;
while (i < 5) {
    System.out.println(i);
    i++;
}
```

### do-while (тело выполняется минимум 1 раз)
```java
int i = 0;
do {
    System.out.println(i);
    i++;
} while (i < 5);
```

### for-each (enhanced for)
```java
int[] numbers = {1, 2, 3, 4, 5};
for (int num : numbers) {
    System.out.println(num);
}

String[] names = {"Alice", "Bob", "Charlie"};
for (String name : names) {
    System.out.println(name);
}
```

### break и continue
```java
for (int i = 0; i < 10; i++) {
    if (i == 5) break;     // выход из цикла
    if (i % 2 == 0) continue;  // переход к следующей итерации
    System.out.println(i); // 1 3
}
```

---

## 🔹 Массивы

> [!note] Определение
> Массив — фиксированный набор элементов одного типа. Размер задаётся при создании и **не изменяется**.

### Одномерный массив
```java
// Объявление и создание
int[] nums = new int[5];    // [0, 0, 0, 0, 0] — значения по умолчанию
nums[0] = 10;
nums[1] = 20;

// Объявление с инициализацией
int[] primes = {2, 3, 5, 7, 11};
String[] names = {"Alice", "Bob", "Charlie"};

// Длина массива
System.out.println(primes.length);  // 5

// Перебор
for (int i = 0; i < primes.length; i++) {
    System.out.println(primes[i]);
}

// Перебор for-each
for (int p : primes) {
    System.out.println(p);
}
```

### Двумерный массив
```java
int[][] matrix = new int[3][3];  // матрица 3×3
matrix[0][0] = 1;

// Инициализация
int[][] grid = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Перебор
for (int i = 0; i < grid.length; i++) {
    for (int j = 0; j < grid[i].length; j++) {
        System.out.print(grid[i][j] + " ");
    }
    System.out.println();
}
```

### Полезные методы Arrays
```java
import java.util.Arrays;

int[] arr = {5, 2, 8, 1, 9};
Arrays.sort(arr);                      // сортировка на месте: [1,2,5,8,9]
System.out.println(Arrays.toString(arr)); // "[1, 2, 5, 8, 9]"
int[] copy = Arrays.copyOf(arr, 3);    // [1, 2, 5]
Arrays.fill(arr, 0);                   // [0, 0, 0, 0, 0]
```

> [!warning] Подводные камни
> - Индексы с `0` до `length - 1`. Выход за границы → `ArrayIndexOutOfBoundsException`
> - `arr.length` (не метод, а поле!) — без `()`
> - `Arrays.toString(arr)` а не `arr.toString()` — иначе печатается адрес объекта

---

## 🔹 Методы

> [!note] Определение
> Метод — именованный блок кода для многократного вызова. Способ организации логики в многоразовые блоки.

### Синтаксис объявления
```java
// модификатор  тип_возврата  имя(параметры) { тело }
public static int add(int a, int b) {
    return a + b;
}

// Метод без возвращаемого значения
public static void printHello(String name) {
    System.out.println("Hello, " + name + "!");
}

// Вызов
int result = add(3, 5);   // result = 8
printHello("Alice");       // "Hello, Alice!"
```

### Перегрузка методов (Overloading)
```java
// Одно имя, разные параметры — компилятор выбирает нужный вариант
public static int sum(int a, int b) {
    return a + b;
}

public static double sum(double a, double b) {
    return a + b;
}

public static int sum(int a, int b, int c) {
    return a + b + c;
}

// Вызов
sum(1, 2);        // int версия → 3
sum(1.5, 2.5);    // double версия → 4.0
sum(1, 2, 3);     // три параметра → 6
```

> [!warning] Подводные камни
> - Перегрузка по **типу возврата** невозможна — компилятор не сможет выбрать нужный метод
> - `return` выходит из метода немедленно — код после него не выполняется
> - void метод может содержать `return;` (без значения) для раннего выхода

### Модификатор static (метод/поле класса)
```java
public class MathUtils {
    public static final double PI = 3.14159;  // статическое поле

    public static double circleArea(double radius) {  // статический метод
        return PI * radius * radius;
    }
}

// Вызов без создания объекта
double area = MathUtils.circleArea(5.0);
System.out.println(MathUtils.PI);
```

> [!tip] Рекомендация
> `static` методы используй для утилитарных функций, которые не зависят от состояния объекта. Смотри подробнее в [[04_OOP_Principles]].

---

## 🔹 Приведение типов (casting)

### Неявное (widening) — безопасное, автоматическое
```java
int i = 42;
long l = i;       // int → long (расширение)
double d = i;     // int → double
float f = i;      // int → float
```

### Явное (narrowing) — может потерять данные
```java
double d = 9.99;
int i = (int) d;   // i = 9 (дробная часть отброшена!)

long l = 1000000000000L;
int i2 = (int) l;  // потеря данных — результат непредсказуем
```

```
byte ← short ← int ← long ← float ← double
                                ↑
                               char
Стрелка ← означает: правый тип расширяется в левый автоматически
```

> [!warning] Подводные камни
> При явном приведении из большего типа в меньший данные **обрезаются**, а не округляются!
