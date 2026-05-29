# Generics

> **Теги:** #interviews #java-core #generics #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Зачем нужны Generics

**Generics** — параметризованные типы для типобезопасности.

### Без Generics

```java
List list = new ArrayList();
list.add("Hello");
list.add(123);  // компилируется, но runtime ошибка

String s = (String) list.get(1);  // ClassCastException!
```

### С Generics

```java
List<String> list = new ArrayList<>();
list.add("Hello");
// list.add(123);  // ошибка компиляции!

String s = list.get(0);  // без приведения типа
```

**Преимущества:**
- Типобезопасность на этапе компиляции
- Устраняет приведения типов (casting)
- Улучшает читаемость кода

---

## 🔹 Generic классы и методы

### Generic класс

```java
public class Box<T> {
    private T value;

    public void set(T value) {
        this.value = value;
    }

    public T get() {
        return value;
    }
}

// Использование
Box<String> stringBox = new Box<>();
stringBox.set("Hello");

Box<Integer> integerBox = new Box<>();
integerBox.set(42);
```

### Generic метод

```java
public class Utils {
    public static <T> T getFirst(List<T> list) {
        return list.get(0);
    }
}

// Использование
String first = Utils.<String>getFirst(names);  // явное указание типа
String first = Utils.getFirst(names);          // вывод типа (type inference)
```

### Несколько параметров типа

```java
public class Pair<K, V> {
    private K key;
    private V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }
}

// Использование
Pair<String, Integer> pair = new Pair<>("age", 30);
```

---

## 🔹 Bounded Type Parameters

### Upper Bound (extends)

Ограничение сверху — тип должен быть наследником указанного класса.

```java
public static <T extends Number> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

// Использование
max(1, 2);           // OK
max(1.5, 2.5);       // OK
// max("a", "b");     // ошибка компиляции
```

### Multiple Bounds

```java
public static <T extends Number & Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}
```

---

## 🔹 Wildcards (?)

**Wildcard** — неизвестный тип, используется для гибкости.

### Unbounded Wildcard (?)

```java
public void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}
```

### Upper Bounded Wildcard (? extends)

```java
public static double sum(List<? extends Number> list) {
    double sum = 0;
    for (Number n : list) {
        sum += n.doubleValue();
    }
    return sum;
}

// Использование
List<Integer> ints = List.of(1, 2, 3);
sum(ints);  // OK

List<Double> doubles = List.of(1.5, 2.5);
sum(doubles);  // OK
```

### Lower Bounded Wildcard (? super)

```java
public static void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}

// Использование
List<Number> numbers = new ArrayList<>();
addNumbers(numbers);  // OK

List<Integer> integers = new ArrayList<>();
addNumbers(integers);  // OK
```

---

## 🔹 PECS Principle

**PECS** — Producer Extends, Consumer Super.

### Producer (производитель)

Если коллекция только **читает** (producer) — используй `? extends`.

```java
public static void printAll(List<? extends Number> list) {
    // Producer — только читаем
    for (Number n : list) {
        System.out.println(n);
    }
}
```

### Consumer (потребитель)

Если коллекция только **записывает** (consumer) — используй `? super`.

```java
public static void addAll(List<? super Integer> list) {
    // Consumer — только записываем
    list.add(1);
    list.add(2);
}
```

### PECS mnemonic

```
P - Producer (производитель) → extends
E - Extends
C - Consumer (потребитель) → super
S - Super
```

### Пример

```java
// Producer — читаем из списка
public static void copy(List<? extends Number> src, List<? super Number> dest) {
    for (Number n : src) {
        dest.add(n);
    }
}
```

> [!warning] Почему нельзя добавлять в ? extends
```java
List<? extends Number> list = new ArrayList<Integer>();
// list.add(1);  // ошибка! не знаем точный тип
```

> [!warning] Почему нельзя читать как конкретный тип из ? super
```java
List<? super Integer> list = new ArrayList<Number>();
// Integer i = list.get(0);  // ошибка! может быть Object
```

---

## 🔹 Type Erasure

**Type Erasure** — стирание типов при компиляции.

### Как работает

```java
// Исходный код
List<String> list = new ArrayList<>();

// После компиляции (bytecode)
List list = new ArrayList();
// тип <String> стирается, добавляются приведения
```

### Пример с методом

```java
// Исходный код
public static <T> T getFirst(List<T> list) {
    return list.get(0);
}

// После компиляции
public static Object getFirst(List list) {
    return list.get(0);
}
```

### Ограничения Type Erasure

1. **Нельзя создать generic массив**
```java
// List<String>[] array = new List<String>[10];  // ошибка!
List<String>[] array = new List[10];  // OK, но warning
```

2. **Нельзя использовать instanceof с generic типами**
```java
if (list instanceof List<String>) { }  // ошибка!
if (list instanceof List<?>) { }      // OK
```

3. **Нельзя создать generic тип**
```java
// T item = new T();  // ошибка!
```

4. **Static поля не могут быть generic**
```java
// private static T value;  // ошибка!
```

---

## 🔹 Generic методы vs Wildcards

### Когда использовать Generic методы

```java
// Generic метод — когда тип T используется в нескольких местах
public static <T> T max(List<T> list) {
    return Collections.max(list);
}
```

### Когда использовать Wildcards

```java
// Wildcard — когда тип используется только один раз
public static void printAll(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Generics** — типобезопасность, устраняет casting
> - **<T extends Number>** — ограничение сверху
> - **<? extends T>** — Producer (чтение)
> - **<? super T>** — Consumer (запись)
> - **PECS** — Producer Extends, Consumer Super
> - **Type Erasure** — стирание типов при компиляции

```
Generic класс:
class Box<T> { T value; }

Bounded:
<T extends Number>  // ограничение сверху

PECS:
Producer → ? extends (чтение)
Consumer → ? super (запись)

Type Erasure:
List<String> → List при компиляции
нельзя: new T[], instanceof List<String>
```
