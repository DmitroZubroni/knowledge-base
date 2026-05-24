# 11 — JVM Internals

> [!abstract] Связи
> [[00_JAVA]] | [[01_Basics]] | [[02_OOP_Core]] | [[09_Multithreading]]

---

## 🔹 Компиляция и выполнение Java

```
Исходный код           Компиляция           Выполнение
─────────────          ──────────           ──────────
HelloWorld.java  →  [javac]  →  HelloWorld.class  →  [JVM]  →  Результат
                              (bytecode)
```

> [!note] Почему байткод, а не машинный код?
> Байткод — платформонезависимый. JVM на каждой ОС (Windows, Linux, macOS) интерпретирует тот же `.class` файл. Это принцип **WORA**: Write Once, Run Anywhere.

```java
// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// В терминале:
// javac HelloWorld.java   → создаёт HelloWorld.class
// java HelloWorld         → JVM запускает байткод
```

---

## 🔹 JDK vs JRE vs JVM

```
JDK (Java Development Kit)
├── javac (компилятор)
├── jdb (отладчик)
├── jar (архиватор)
├── javadoc
└── JRE (Java Runtime Environment)
    ├── Стандартные библиотеки (java.lang, java.util, java.io...)
    └── JVM (Java Virtual Machine)
        ├── Class Loader
        ├── Bytecode Verifier
        ├── Interpreter
        ├── JIT Compiler
        └── Garbage Collector
```

| | Для чего | Содержит |
|-|----------|---------|
| **JVM** | Выполнение байткода | Интерпретатор, GC, JIT |
| **JRE** | Запуск Java программ | JVM + стандартные библиотеки |
| **JDK** | Разработка Java программ | JRE + javac + инструменты |

---

## 🔹 JIT-компилятор (Just-In-Time)

> [!note] Определение
> JIT — компилирует "горячие" (часто выполняемые) методы в нативный машинный код во время выполнения. Это делает Java быстрее простой интерпретации.

```
Первые запуски метода:   [Interpreter]  — интерпретация, медленно
После N вызовов:         [JIT Compiler] — компиляция в машинный код
Последующие вызовы:      [Нативный код] — быстро, как C/C++

Порог JIT по умолчанию: ~10 000 вызовов метода
```

---

## 🔹 Память JVM: Stack и Heap

```
┌─────────────────────────────────────────────────────────┐
│                        JVM Memory                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │   Stack      │  │   Stack      │  │      Heap        │ │
│  │  (Thread 1)  │  │  (Thread 2)  │  │  (общий для всех)│ │
│  │─────────────│  │─────────────│  │─────────────────│ │
│  │ frame: main │  │ frame: run  │  │  Object A        │ │
│  │  int x = 5  │  │  int i = 0  │  │  Object B        │ │
│  │  ref → A   ─┼──┼─────────────┼──┼→ Object C        │ │
│  │ frame: foo  │  │             │  │  String Pool     │ │
│  │  int y = 3  │  │             │  │  ...             │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘ │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │                  Metaspace (Java 8+)                 ││
│  │  Class metadata, static поля, constant pool         ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Stack (Стек)
```java
// Каждый поток имеет свой стек
// При вызове метода — создаётся новый "фрейм" (stack frame)
// При выходе из метода — фрейм уничтожается

public static void main(String[] args) {           // frame: main
    int x = 42;          // примитив: хранится в стеке
    String s = "hello";  // ссылка в стеке, объект в heap
    Dog d = new Dog();   // ссылка в стеке, объект в heap
    foo(x);              // создаётся новый frame для foo
}

public static void foo(int n) {  // frame: foo
    int y = n * 2;       // y — в стеке (frame foo)
}                        // frame foo уничтожается
```

**Stack хранит:**
- Примитивные переменные (`int`, `double`, `boolean`...)
- Ссылки на объекты (но не сами объекты!)
- Параметры методов
- Стек вызовов (call stack)

### Heap (Куча)
```java
// Все объекты создаются в Heap
Dog dog = new Dog("Rex");  // объект Dog — в Heap
int[] arr = new int[100];  // массив — тоже объект, в Heap
String s = new String("Hello");  // в Heap (не в пуле)
```

**Heap хранит:**
- Все объекты (результат `new`)
- Массивы (они тоже объекты)
- Строки (не из String Pool)

| | Stack | Heap |
|-|-------|------|
| Владелец | Каждый поток свой | Один на всё приложение |
| Хранит | Примитивы, ссылки, вызовы | Объекты, массивы |
| Размер | Меньше (~1 МБ по умолчанию) | Больше (настраивается) |
| Скорость | Быстрее (LIFO) | Медленнее |
| Управление памятью | Автоматически (LIFO) | GC |
| Ошибка переполнения | `StackOverflowError` | `OutOfMemoryError` |

> [!warning] StackOverflowError
> Бесконечная рекурсия переполняет стек:
> ```java
> public static void infinite() {
>     infinite();  // каждый вызов — новый frame, стек переполнится
> }
> ```

---

## 🔹 Garbage Collector (GC)

> [!note] Определение
> GC — автоматически освобождает память объектов, на которые больше нет **достижимых ссылок**. Программист не управляет памятью вручную.

```
Объект "живой" (reachable):    есть хотя бы одна ссылка до него от "корней"
Объект "мёртвый" (unreachable): нет ни одной живой ссылки → GC удалит

"Корни GC" (GC Roots):
  - локальные переменные в стеках потоков
  - статические поля классов
  - JNI ссылки
```

```java
// Пример создания "мусора"
public void createGarbage() {
    Dog dog = new Dog("Rex");   // создан объект в Heap
    dog = new Dog("Buddy");     // dog теперь ссылается на другой объект
    // первый объект Dog("Rex") — никто на него не ссылается → кандидат для GC
}
// После выхода из метода — dog тоже исчезает из стека → Dog("Buddy") тоже мусор
```

### Поколения (Generational GC)
```
Heap
├── Young Generation (молодое поколение)
│   ├── Eden Space     — новые объекты создаются здесь
│   ├── Survivor S0    — выжившие после Minor GC
│   └── Survivor S1
└── Old Generation (старое поколение / Tenured)
    └── долгоживущие объекты, пережившие несколько Minor GC

Metaspace — метаданные классов (вне Heap, Java 8+)
```

```
Minor GC  — очищает Young Generation (быстро, часто)
Major GC  — очищает Old Generation (медленнее)
Full GC   — очищает всё (медленно, Stop-the-World)
```

### Типы GC в Java
| GC | Описание | Когда использовать |
|----|---------|-------------------|
| Serial GC | Один поток, паузы | Маленькие приложения |
| Parallel GC | Несколько потоков | Throughput (default до Java 8) |
| G1 GC | Предсказуемые паузы | Большой Heap (default Java 9+) |
| ZGC | Паузы < 10ms | Большой Heap, low-latency |
| Shenandoah | Паузы < 10ms | Аналог ZGC |

```java
// Попросить GC запуститься (не гарантия!)
System.gc();  // лишь "подсказка" JVM

// Настройка через JVM флаги:
// java -Xms256m -Xmx1g -XX:+UseG1GC MyApp
//       ↑ начальный heap  ↑ максимальный heap  ↑ использовать G1 GC
```

---

## 🔹 String Pool

> [!note] Определение
> String Pool — область в Heap, где JVM кеширует строковые литералы. Одинаковые литералы указывают на **один объект**.

```java
// Строковые литералы — берутся из пула
String s1 = "hello";  // создаётся в пуле
String s2 = "hello";  // возвращается существующий объект из пула
String s3 = "hello";  // тот же объект

System.out.println(s1 == s2);  // true (одна ссылка!)
System.out.println(s1 == s3);  // true

// new String — создаёт новый объект в Heap, НЕ в пуле
String s4 = new String("hello");  // новый объект в Heap
String s5 = new String("hello");  // ещё один объект в Heap

System.out.println(s1 == s4);      // false (разные объекты!)
System.out.println(s4 == s5);      // false
System.out.println(s4.equals(s5)); // true (содержимое одинаковое)

// intern() — поместить строку в пул
String s6 = s4.intern();  // вернуть строку из пула (или добавить)
System.out.println(s1 == s6);  // true
```

```
String Pool (в Heap):          Heap:
──────────────────             ────────────────────
"hello" ──────────────────→ s1, s2, s3
"world"                        s4 → [новый объект "hello"]
"java"                         s5 → [ещё один объект "hello"]
```

> [!warning] Подводные камни
> `==` сравнивает **ссылки**, а не содержимое.
> Для строк **всегда** используй `.equals()`:
> ```java
> // ❌ Ненадёжно
> if (input == "yes") { ... }
>
> // ✅ Правильно
> if ("yes".equals(input)) { ... }  // "yes".equals — защита от NPE если input null
> if (input != null && input.equals("yes")) { ... }
> ```

---

## 🔹 Autoboxing / Unboxing

> [!note] Определение
> Автоматическое преобразование примитивного типа в объект-обёртку (boxing) и обратно (unboxing). Нужно для использования примитивов в коллекциях и generics.

```
Примитив    Обёртка (Wrapper)
────────    ─────────────────
byte    ↔   Byte
short   ↔   Short
int     ↔   Integer
long    ↔   Long
float   ↔   Float
double  ↔   Double
char    ↔   Character
boolean ↔   Boolean
```

```java
// Autoboxing — int → Integer (автоматически)
Integer i = 42;          // компилятор делает: Integer.valueOf(42)
List<Integer> list = new ArrayList<>();
list.add(5);             // autoboxing: 5 → Integer.valueOf(5)

// Unboxing — Integer → int (автоматически)
int x = i;              // компилятор делает: i.intValue()
int sum = list.get(0) + 10;  // unboxing: Integer → int, затем +10

// Ручное преобразование
Integer wrapped = Integer.valueOf(100);
int primitive = wrapped.intValue();

// Полезные методы обёрток
Integer.parseInt("42");          // "42" → 42
Integer.toString(42);            // 42 → "42"
Integer.MAX_VALUE;               // 2147483647
Integer.MIN_VALUE;               // -2147483648
Integer.toBinaryString(10);      // "1010"
Integer.toHexString(255);        // "ff"
Double.parseDouble("3.14");      // "3.14" → 3.14
Boolean.parseBoolean("true");    // "true" → true
```

### Ловушка Integer кеша
```java
// Integer кеширует значения от -128 до 127
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (из кеша — один объект)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (не из кеша — разные объекты!)
System.out.println(c.equals(d));  // true (правильное сравнение)
```

> [!danger] Производительность
> Autoboxing в цикле — скрытая проблема производительности:
> ```java
> // ❌ Каждая итерация создаёт объект Long в Heap
> Long sum = 0L;
> for (long i = 0; i < 1_000_000; i++) {
>     sum += i;  // unboxing + boxing на каждой итерации!
> }
>
> // ✅ Используй примитив
> long sum = 0L;
> for (long i = 0; i < 1_000_000; i++) {
>     sum += i;
> }
> ```

---

## 🔹 Версии Java

| Версия | Год | Ключевые фичи |
|--------|-----|----------------|
| **Java 8** (LTS) | 2014 | Lambda, Stream API, Optional, default методы, новый Date/Time API |
| **Java 11** (LTS) | 2018 | `var` в лямбдах, `String.isBlank()`, `Files.readString()`, HTTP Client |
| **Java 17** (LTS) | 2021 | Sealed classes, Pattern Matching instanceof, Records (стабильно), Switch expressions |
| **Java 21** (LTS) | 2023 | Virtual Threads (Project Loom), Record Patterns, Sequenced Collections |

> [!note] LTS (Long Term Support)
> LTS-версии поддерживаются 8+ лет. В продакшне используй LTS: **8, 11, 17, 21**.

### var — локальный вывод типа (Java 10+)
```java
// Компилятор выводит тип из правой части
var list = new ArrayList<String>();    // тип: ArrayList<String>
var name = "Alice";                    // тип: String
var number = 42;                       // тип: int

// var только для локальных переменных
// Нельзя для полей класса, параметров методов, return типа
```

---

## 🔹 Загрузка классов (Class Loading)

```
Bootstrap ClassLoader   — загружает java.lang, java.util (из rt.jar / jmods)
     ↑
Extension ClassLoader   — загружает ext/*.jar
     ↑
Application ClassLoader — загружает classpath твоего приложения
     ↑
Custom ClassLoader      — можно написать свой

Принцип: сначала спрашивает родителя (Parent Delegation)
```

```java
// Загрузить класс по имени
Class<?> clazz = Class.forName("com.example.MyClass");

// Посмотреть загрузчик класса
System.out.println(String.class.getClassLoader());       // null (Bootstrap)
System.out.println(MyClass.class.getClassLoader());      // AppClassLoader
```

---

## 🔹 JVM флаги (для информации)

```bash
# Размер памяти
java -Xms512m -Xmx2g MyApp    # начальный heap 512МБ, максимум 2ГБ
java -Xss512k MyApp            # размер стека каждого потока

# GC
java -XX:+UseG1GC MyApp
java -XX:+UseZGC MyApp
java -verbose:gc MyApp         # логировать GC

# Отладка
java -XX:+HeapDumpOnOutOfMemoryError MyApp   # дамп при OOM
java -XX:HeapDumpPath=/tmp/dump.hprof MyApp
```
