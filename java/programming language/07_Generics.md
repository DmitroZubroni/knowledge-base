# 07 — Generics (Обобщения)

> [!abstract] Связи
> [[00_INDEX]] | [[06_Collections]] | [[08_Functional]] | [[03_OOP_Advanced]]

---

## 🔹 Зачем нужны Generics

> [!note] Определение
> Generics — параметризация типов для написания обобщённого, типобезопасного кода. Ошибки типов выявляются на этапе **компиляции**, а не выполнения.

```java
// БЕЗ Generics (до Java 5) — небезопасно
List listOld = new ArrayList();
listOld.add("text");
listOld.add(42);         // добавить число — компилятор не запретит
String s = (String) listOld.get(0);  // нужен явный cast
String s2 = (String) listOld.get(1); // ❌ ClassCastException в runtime!

// С Generics — типобезопасно
List<String> listNew = new ArrayList<>();
listNew.add("text");
// listNew.add(42);  // ❌ ошибка компиляции — перехвачена заранее!
String s3 = listNew.get(0);  // не нужен cast
```

---

## 🔹 Generic классы

> [!note] Определение
> Generic класс параметризуется типом `<T>` при объявлении. При создании объекта тип указывается конкретно.

```java
// Параметр типа T — можно любую букву, но соглашения:
// T — Type, E — Element, K — Key, V — Value, R — Result, N — Number

public class Box<T> {
    private T value;

    public Box(T value) {
        this.value = value;
    }

    public T getValue() {
        return value;
    }

    public void setValue(T value) {
        this.value = value;
    }

    @Override
    public String toString() {
        return "Box[" + value + "]";
    }
}

// Использование с разными типами
Box<String> strBox = new Box<>("Hello");
Box<Integer> intBox = new Box<>(42);
Box<Double> dblBox = new Box<>(3.14);

System.out.println(strBox.getValue());  // "Hello"
System.out.println(intBox.getValue());  // 42

// Java 7+: Diamond operator <> — тип выводится автоматически
Box<String> box = new Box<>("World");  // не нужно Box<String>(...)
```

### Generic класс с несколькими параметрами
```java
public class Pair<K, V> {
    private K key;
    private V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() { return key; }
    public V getValue() { return value; }

    @Override
    public String toString() {
        return "(" + key + ", " + value + ")";
    }
}

Pair<String, Integer> pair = new Pair<>("age", 25);
System.out.println(pair);  // "(age, 25)"

Pair<String, List<Integer>> complex = new Pair<>("scores", List.of(90, 85, 92));
```

---

## 🔹 Generic методы

> [!note] Определение
> Generic метод объявляет свой параметр типа `<T>` перед типом возвращаемого значения. Тип выводится компилятором автоматически по аргументам.

```java
public class ArrayUtils {
    // Generic метод — <T> перед типом возврата
    public static <T> T getLast(List<T> list) {
        if (list.isEmpty()) return null;
        return list.get(list.size() - 1);
    }

    // Поменять два элемента местами
    public static <T> void swap(List<T> list, int i, int j) {
        T temp = list.get(i);
        list.set(i, list.get(j));
        list.set(j, temp);
    }

    // Сделать копию и вернуть
    public static <T> List<T> repeat(T item, int times) {
        List<T> result = new ArrayList<>();
        for (int i = 0; i < times; i++) {
            result.add(item);
        }
        return result;
    }
}

// Использование — тип выводится автоматически
String last = ArrayUtils.getLast(List.of("a", "b", "c"));  // "c"
List<Integer> threes = ArrayUtils.repeat(3, 5);  // [3, 3, 3, 3, 3]

// Явное указание типа (редко нужно)
String s = ArrayUtils.<String>getLast(List.of("x", "y"));
```

---

## 🔹 Ограниченные параметры типов (Bounded)

> [!note] Определение
> `<T extends SomeClass>` ограничивает тип T — он должен быть `SomeClass` или его подклассом. Позволяет вызывать методы `SomeClass` внутри generic кода.

```java
// Только числа
public static <T extends Number> double sum(List<T> list) {
    double total = 0;
    for (T item : list) {
        total += item.doubleValue();  // метод Number доступен!
    }
    return total;
}

sum(List.of(1, 2, 3));      // 6.0
sum(List.of(1.5, 2.5));     // 4.0
// sum(List.of("a","b"));   // ❌ String не наследует Number

// Интерфейс как ограничение
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

max(10, 20);         // 20
max("apple", "pear"); // "pear"

// Несколько ограничений — & (первый может быть класс, остальные — интерфейсы)
public static <T extends Number & Comparable<T>> T clamp(T value, T min, T max) {
    if (value.compareTo(min) < 0) return min;
    if (value.compareTo(max) > 0) return max;
    return value;
}
```

---

## 🔹 Wildcard (подстановочный знак)

> [!note] Определение
> `?` — неизвестный тип. Используется когда конкретный тип не важен или заранее неизвестен.

### Неограниченный wildcard `<?>`
```java
// Метод принимает список любого типа
public static void printAll(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

printAll(List.of("a", "b", "c"));   // OK
printAll(List.of(1, 2, 3));         // OK
printAll(List.of(1.5, 2.5));        // OK

// ❗ Нельзя добавлять в список с <?> — тип неизвестен
// list.add("something");  // ❌ ошибка компиляции
```

### Ковариантный wildcard `<? extends T>` — "Producer"
```java
// Принимает List<Animal> или List<Dog> или List<Cat> (Animal и подклассы)
public static double totalArea(List<? extends Shape> shapes) {
    double total = 0;
    for (Shape shape : shapes) {  // можно читать как Shape
        total += shape.area();
    }
    return total;
}

List<Circle> circles = List.of(new Circle("red", 5), new Circle("blue", 3));
List<Rectangle> rects = List.of(new Rectangle("green", 4, 6));

totalArea(circles);  // OK
totalArea(rects);    // OK

// ❌ Нельзя добавлять — не знаем точного типа
// shapes.add(new Circle());  // ❌
```

### Контравариантный wildcard `<? super T>` — "Consumer"
```java
// Принимает List<Dog>, List<Animal>, List<Object>
public static void addDogs(List<? super Dog> list) {
    list.add(new Dog("Rex", 3, "Husky"));  // ✅ Dog можно добавить
    list.add(new Dog("Buddy", 2, "Lab"));
    // list.add(new Animal("Generic", 1));  // ❌ Animal может не быть Dog
}

List<Animal> animals = new ArrayList<>();
addDogs(animals);   // OK — Animal суперкласс Dog
List<Dog> dogs = new ArrayList<>();
addDogs(dogs);      // OK
```

### PECS — запомнить правило
```
PECS: Producer Extends, Consumer Super

Если получаешь элементы ИЗ структуры → <? extends T>  (producer)
Если кладёшь элементы В структуру    → <? super T>    (consumer)
Если и то и другое                   → <T>

Примеры из JDK:
Collections.copy(List<? super T> dest, List<? extends T> src)
                 ↑ consumer (пишем)        ↑ producer (читаем)
```

---

## 🔹 Стирание типов (Type Erasure)

> [!note] Определение
> В **runtime** информация о generic-типах стирается компилятором. `List<String>` и `List<Integer>` в байткоде — оба просто `List`. Это решение для обратной совместимости с Java до 5.

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();

// В runtime оба — просто ArrayList
System.out.println(strings.getClass() == ints.getClass()); // true!

// Нельзя использовать generic тип в instanceof
// if (list instanceof List<String>) { }  // ❌ ошибка компиляции

// Нельзя создать массив generic типа
// T[] array = new T[10];  // ❌ ошибка компиляции

// Нельзя создать экземпляр T
// T obj = new T();  // ❌ ошибка компиляции
```

### Что происходит при стирании типов
```
// Исходный код
public class Box<T> {
    private T value;
    public T getValue() { return value; }
}

// После компиляции (type erasure)
public class Box {
    private Object value;          // T → Object
    public Object getValue() { return value; }
}

// Для <T extends Number>:
public class NumberBox<T extends Number> {
    private T value;
    public T getValue() { return value; }
}
// После erasure:
public class NumberBox {
    private Number value;          // T → Number (верхняя граница)
    public Number getValue() { return value; }
}
```

> [!warning] Подводные камни
> - Нельзя `new T()` — тип неизвестен в runtime. Обходной путь: передать `Class<T>`
> - Нельзя `new T[size]` — использовать `(T[]) new Object[size]` с кастом (с предупреждением)
> - `instanceof` не работает с параметрами типа
> - При хранении в БД или сериализации generic-тип теряется

```java
// Обходной путь для new T() через Class<T>
public class Factory<T> {
    private final Class<T> type;

    public Factory(Class<T> type) {
        this.type = type;
    }

    public T create() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}

Factory<Dog> dogFactory = new Factory<>(Dog.class);
```
