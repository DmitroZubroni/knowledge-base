> **Теги:** #java #functional #lambda #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Functional_Index]]

# 01 — Лямбды и функциональные интерфейсы

---

## 🔹 Зачем лямбды — проблема анонимных классов

До Java 8 для передачи "поведения" в метод использовались анонимные классы. Это работало, но было громоздко:

```java
// До Java 8 — анонимный класс
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

// Java 8 — лямбда. То же самое, в одну строку
Collections.sort(names, (a, b) -> a.compareTo(b));

// Ещё короче — ссылка на метод
Collections.sort(names, String::compareTo);
```

Лямбды — это не просто синтаксический сахар. Они изменили стиль написания Java-кода: вместо "как делать" (imperatively) стало возможно писать "что делать" (declaratively).

---

## 🔹 Функциональный интерфейс — основа лямбд

Лямбда в Java — это реализация **функционального интерфейса**: интерфейса ровно с **одним абстрактным методом**.

```java
@FunctionalInterface          // необязательно, но хорошая практика
public interface Transformer {
    String transform(String input); // единственный абстрактный метод
}

// Реализация через анонимный класс (старый стиль)
Transformer upper = new Transformer() {
    @Override
    public String transform(String input) {
        return input.toUpperCase();
    }
};

// Реализация через лямбду (Java 8+)
Transformer upper = input -> input.toUpperCase();

// Ещё короче — ссылка на метод
Transformer upper = String::toUpperCase;

System.out.println(upper.transform("hello")); // HELLO
```

> [!note] @FunctionalInterface
> Аннотация не обязательна, но её наличие даёт две вещи:
> - Компилятор **проверит** что в интерфейсе ровно один абстрактный метод
> - Читатель кода сразу понимает намерение автора
>
> Интерфейс может содержать `default` и `static` методы — они не нарушают правило одного абстрактного метода.

---

## 🔹 Синтаксис лямбд

```java
// Базовая форма: (параметры) -> тело

// Без параметров
Runnable r = () -> System.out.println("Hello!");

// Один параметр — скобки необязательны
Consumer<String> print = s -> System.out.println(s);

// Один параметр с явным типом — скобки обязательны
Consumer<String> print = (String s) -> System.out.println(s);

// Несколько параметров — скобки обязательны
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// Тело — одно выражение (return не нужен)
Function<String, Integer> len = s -> s.length();

// Тело — блок кода (нужны {} и return)
Function<String, String> process = s -> {
    String trimmed = s.trim();
    String upper = trimmed.toUpperCase();
    return upper;
};

// Void блок — return не нужен
Runnable multi = () -> {
    System.out.println("Step 1");
    System.out.println("Step 2");
};
```

---

## 🔹 Стандартные функциональные интерфейсы

Java поставляет готовые интерфейсы в пакете `java.util.function`. Создавать свои нужно редко.

### Четыре основных

```java
import java.util.function.*;

// Function<T, R> — принимает T, возвращает R
Function<String, Integer> length = s -> s.length();
length.apply("Hello"); // 5

// Predicate<T> — принимает T, возвращает boolean
Predicate<String> isLong = s -> s.length() > 5;
isLong.test("Hello World"); // true

// Consumer<T> — принимает T, ничего не возвращает
Consumer<String> printer = s -> System.out.println(s);
printer.accept("Hello"); // печатает Hello

// Supplier<T> — ничего не принимает, возвращает T
Supplier<String> greeting = () -> "Hello!";
greeting.get(); // "Hello!"
```

### Расширения основных

```java
// UnaryOperator<T> — это Function<T,T> (тот же тип на входе и выходе)
UnaryOperator<String> trim = s -> s.trim();
UnaryOperator<Integer> doubleIt = n -> n * 2;

// BinaryOperator<T> — это BiFunction<T,T,T>
BinaryOperator<Integer> sum = (a, b) -> a + b;
BinaryOperator<String> concat = (a, b) -> a + b;

// BiFunction<T, U, R> — два параметра, один результат
BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
repeat.apply("ha", 3); // "hahaha"

// BiPredicate<T, U> — два параметра, boolean
BiPredicate<String, Integer> longerThan = (s, n) -> s.length() > n;
longerThan.test("Hello", 3); // true

// BiConsumer<T, U> — два параметра, void
BiConsumer<String, Integer> printN = (s, n) -> {
    for (int i = 0; i < n; i++) System.out.println(s);
};
```

### Примитивные специализации — без boxing

Для примитивов существуют специализированные интерфейсы, избегающие дорогостоящего boxing/unboxing:

```java
// Вместо Function<Integer, Integer> — IntUnaryOperator
IntUnaryOperator doubleInt = n -> n * 2;
doubleInt.applyAsInt(5); // 10 — работает с int напрямую, без Integer

// Вместо Function<Integer, String>
IntFunction<String> intToStr = n -> "Number: " + n;
intToStr.apply(42); // "Number: 42"

// Вместо Function<String, Integer>
ToIntFunction<String> strToInt = s -> s.length();
strToInt.applyAsInt("Hello"); // 5

// Вместо Predicate<Integer>
IntPredicate isEven = n -> n % 2 == 0;
isEven.test(4); // true

// Вместо Supplier<Integer>
IntSupplier randomInt = () -> new Random().nextInt(100);
randomInt.getAsInt(); // случайное число
```

> [!tip] Когда использовать примитивные специализации
> В горячих участках кода (обработка больших массивов, числовые вычисления) примитивные интерфейсы дают ощутимый прирост производительности за счёт исключения boxing. В обычном коде разница незаметна.

---

## 🔹 Композиция функций

Функциональные интерфейсы имеют default методы для объединения в цепочки.

### Function — andThen и compose

```java
Function<Integer, Integer> doubleIt = n -> n * 2;
Function<Integer, Integer> addTen   = n -> n + 10;

// andThen: сначала doubleIt, потом addTen
Function<Integer, Integer> doubleThenAdd = doubleIt.andThen(addTen);
doubleThenAdd.apply(5); // (5*2)+10 = 20

// compose: сначала addTen, потом doubleIt
Function<Integer, Integer> addThenDouble = doubleIt.compose(addTen);
addThenDouble.apply(5); // (5+10)*2 = 30

// Длинная цепочка
Function<String, String> pipeline = Function.<String>identity()
    .andThen(String::trim)
    .andThen(String::toLowerCase)
    .andThen(s -> s.replace(" ", "_"));

pipeline.apply("  Hello World  "); // "hello_world"
```

### Predicate — and, or, negate

```java
Predicate<String> isLong      = s -> s.length() > 5;
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<Integer> isEven     = n -> n % 2 == 0;

Predicate<String> longAndStartsA = isLong.and(startsWithA);
longAndStartsA.test("Alexander"); // true (длина > 5 И начинается с A)
longAndStartsA.test("Anna");      // false (длина ≤ 5)

Predicate<String> longOrStartsA = isLong.or(startsWithA);
longOrStartsA.test("Anna"); // true (начинается с A, хоть и короткая)

Predicate<String> notLong = isLong.negate();
notLong.test("Hi"); // true

// Predicate.not() — Java 11+, удобно для метод-ссылок
List<String> filtered = names.stream()
    .filter(Predicate.not(String::isEmpty)) // убрать пустые строки
    .collect(Collectors.toList());
```

### Consumer — andThen

```java
Consumer<String> print   = s -> System.out.println(s);
Consumer<String> log     = s -> logger.info(s);

Consumer<String> printAndLog = print.andThen(log);
printAndLog.accept("event"); // сначала println, потом log
```

---

## 🔹 Замыкания (Closures) — захват переменных

Лямбда может использовать переменные из внешнего контекста — это называется **захват** (capture). Но с важным ограничением:

```java
String prefix = "Hello, ";     // effectively final — не меняется после объявления
Function<String, String> greet = name -> prefix + name;  // захват prefix

greet.apply("Alice"); // "Hello, Alice"

// prefix = "Hi, ";  // ❌ ошибка компиляции — нельзя менять захваченную переменную
```

**Effectively final** — переменная, которая не переприсваивается после объявления. Явное ключевое слово `final` необязательно.

```java
// ❌ Частая ошибка — попытка изменить переменную в лямбде
int counter = 0;
Runnable task = () -> counter++; // ❌ ошибка компиляции: counter не effectively final

// ✅ Решение через AtomicInteger
AtomicInteger counter = new AtomicInteger(0);
Runnable task = () -> counter.incrementAndGet(); // ✅ объект effectively final, но его состояние меняется
```

> [!warning] Захват this в лямбде
> Лямбда захватывает `this` внешнего класса (не создаёт собственного `this`). Это отличает лямбду от анонимного класса, у которого есть свой `this`.
> ```java
> class MyClass {
>     String name = "outer";
>
>     Runnable lambda = () -> System.out.println(this.name);    // this = MyClass
>     Runnable anon = new Runnable() {
>         @Override
>         public void run() {
>             System.out.println(this); // this = анонимный класс
>         }
>     };
> }
> ```

---

## 🔹 Собственный функциональный интерфейс — когда нужен

Стандартных интерфейсов обычно хватает. Свой интерфейс нужен если:
- Нужно более говорящее имя (self-documenting API)
- Нужно бросать checked исключение

```java
// Пример: обработчик с checked исключением
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
}

// Использование
ThrowingFunction<String, Integer> parse = Integer::parseInt;

// Обёртка для использования в Stream (который не любит checked exceptions)
public static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> fn) {
    return t -> {
        try {
            return fn.apply(t);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    };
}

List<String> numbers = List.of("1", "2", "abc", "4");
numbers.stream()
    .map(wrap(Integer::parseInt)) // теперь работает в stream
    .collect(Collectors.toList());
```

---

## 🔹 Итог

```
Функциональный интерфейс = интерфейс с одним абстрактным методом
@FunctionalInterface — проверяет это на этапе компиляции

Основные интерфейсы java.util.function:
  Function<T,R>     T → R           apply(t)
  Predicate<T>      T → boolean     test(t)
  Consumer<T>       T → void        accept(t)
  Supplier<T>       () → T          get()
  UnaryOperator<T>  T → T           apply(t)    (частный случай Function)
  BinaryOperator<T> (T,T) → T       apply(t,u)
  BiFunction<T,U,R> (T,U) → R       apply(t,u)

Примитивные специализации (IntFunction, ToIntFunction, IntPredicate...) —
  используй в горячем коде для избежания boxing

Композиция:
  Function:  andThen (A→B), compose (B→A по порядку применения)
  Predicate: and, or, negate, Predicate.not()
  Consumer:  andThen

Замыкания:
  Захватывают effectively final переменные
  this лямбды = this внешнего класса (не создаёт своего)
```
