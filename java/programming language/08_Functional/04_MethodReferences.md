> **Теги:** #java #functional #method-references #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Functional_Index]]

# 04 — Ссылки на методы (Method References)

---

## 🔹 Зачем нужны ссылки на методы

Лямбды отлично работают, но часто они делают только одно — вызывают уже существующий метод:

```java
// Лямбда просто делегирует вызов существующему методу
names.stream().map(s -> s.toUpperCase());         // вызывает toUpperCase
names.stream().forEach(s -> System.out.println(s)); // вызывает println
numbers.stream().filter(n -> Objects.nonNull(n));   // вызывает nonNull
```

В таких случаях можно использовать **ссылку на метод** — это ещё более краткая запись лямбды:

```java
names.stream().map(String::toUpperCase);
names.stream().forEach(System.out::println);
numbers.stream().filter(Objects::nonNull);
```

Ссылка на метод — это не магия, это просто синтаксический сахар. Компилятор сам разворачивает её в лямбду.

---

## 🔹 Четыре вида ссылок

### 1. Статический метод — `ClassName::staticMethod`

Ссылается на статический метод класса.

```java
// Лямбда                              Ссылка на метод
s -> Integer.parseInt(s)           →   Integer::parseInt
n -> String.valueOf(n)             →   String::valueOf
s -> Objects.isNull(s)             →   Objects::isNull
(a, b) -> Math.max(a, b)           →   Math::max

// Применение
Function<String, Integer> parse = Integer::parseInt;
parse.apply("42"); // 42

List<String> nums = List.of("1", "3", "2");
nums.stream()
    .map(Integer::parseInt)          // String → Integer
    .sorted()
    .forEach(System.out::println);   // 1, 2, 3
```

### 2. Метод экземпляра конкретного объекта — `object::instanceMethod`

Ссылается на метод **конкретного** объекта (object — уже существующий экземпляр).

```java
// Лямбда                              Ссылка на метод
s -> System.out.println(s)         →   System.out::println
s -> myLogger.log(s)               →   myLogger::log
s -> prefix.concat(s)              →   prefix::concat

// Применение
PrintStream out = System.out;
Consumer<String> printer = out::println;
printer.accept("Hello"); // печатает Hello

// В реальном коде
List<String> events = List.of("start", "stop", "error");
events.forEach(logger::info); // вызываем info на конкретном объекте logger
```

### 3. Метод экземпляра произвольного объекта типа — `ClassName::instanceMethod`

Самый неочевидный вид. Первый параметр лямбды становится объектом, на котором вызывается метод.

```java
// Лямбда                              Ссылка на метод
s -> s.toUpperCase()               →   String::toUpperCase
s -> s.isEmpty()                   →   String::isEmpty
s -> s.length()                    →   String::length
(s1, s2) -> s1.compareTo(s2)       →   String::compareTo

// Как это работает:
// String::toUpperCase  эквивалентно  (String s) -> s.toUpperCase()
// Первый аргумент — это объект, на котором вызывается метод

Function<String, String>  toUpper  = String::toUpperCase;
Function<String, Integer> getLen   = String::length;
Predicate<String>         isEmpty  = String::isEmpty;

toUpper.apply("hello");  // "HELLO"
getLen.apply("hello");   // 5
isEmpty.test("");        // true

// В стримах — очень часто
List<String> words = List.of("Hello", "", "World", "");
words.stream()
    .filter(Predicate.not(String::isEmpty))  // убрать пустые
    .map(String::toLowerCase)                // в нижний регистр
    .sorted(String::compareTo)              // отсортировать
    .collect(Collectors.toList());
```

> [!note] Разница между видом 2 и видом 3
> ```java
> String prefix = "Mr. ";
> Function<String, String> f2 = prefix::concat;   // ВИД 2: prefix — конкретный объект
> // f2.apply("Smith") → prefix.concat("Smith") → "Mr. Smith"
>
> Function<String, String> f3 = String::concat;   // ВИД 3: первый аргумент = объект
> // f3.apply("Hello") → "Hello".concat(???) — нет второго аргумента!
> // Правильно через BiFunction:
> BiFunction<String, String, String> f3b = String::concat;
> // f3b.apply("Hello", " World") → "Hello".concat(" World") → "Hello World"
> ```

### 4. Конструктор — `ClassName::new`

Ссылается на конструктор класса.

```java
// Лямбда                              Ссылка на метод
() -> new ArrayList<>()            →   ArrayList::new
s -> new StringBuilder(s)          →   StringBuilder::new
(k, v) -> new AbstractMap.SimpleEntry<>(k, v) → AbstractMap.SimpleEntry::new

// Применение
Supplier<ArrayList<String>> listFactory = ArrayList::new;
ArrayList<String> list = listFactory.get(); // создаёт новый ArrayList

Function<String, StringBuilder> sbFactory = StringBuilder::new;
StringBuilder sb = sbFactory.apply("initial"); // new StringBuilder("initial")

// В стримах
List<String> names = List.of("Alice", "Bob");
List<StringBuilder> builders = names.stream()
    .map(StringBuilder::new)   // для каждого имени создаём StringBuilder
    .collect(Collectors.toList());
```

---

## 🔹 Таблица всех четырёх видов

| Вид | Синтаксис | Эквивалентная лямбда |
|-----|-----------|----------------------|
| Статический метод | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` |
| Метод конкретного объекта | `object::instanceMethod` | `(args) -> object.instanceMethod(args)` |
| Метод произвольного объекта типа | `ClassName::instanceMethod` | `(obj, args) -> obj.instanceMethod(args)` |
| Конструктор | `ClassName::new` | `(args) -> new ClassName(args)` |

---

## 🔹 Когда использовать, а когда нет

```java
// ✅ Хорошо — ссылка читается лучше чем лямбда
names.stream().map(String::toUpperCase)
names.stream().forEach(System.out::println)
numbers.stream().filter(Objects::nonNull)

// ❌ Плохо — лямбда читается лучше
// Если лямбда сложнее одного вызова — пиши лямбду
names.stream().map(s -> s.trim().toLowerCase())  // лучше чем пытаться уместить в ссылку
names.stream().filter(s -> s.length() > 3 && s.startsWith("A")) // тоже лямбда

// ❌ Плохо — ссылка неочевидна
list.stream().map(this::complexTransform)  // что делает complexTransform? не ясно из контекста
// Лучше оставить лямбду с описательным именем переменной или комментарием
```

> [!tip] Правило
> Используй ссылку на метод когда лямбда делает **ровно один вызов** существующего метода. Если нужна логика — пиши лямбду. Цель — читаемость, а не максимальная краткость.

---

## 🔹 Итог

```
Ссылка на метод = сокращённая лямбда, которая просто вызывает метод

Четыре вида:
  ClassName::staticMethod      — статический метод
  object::instanceMethod       — метод конкретного экземпляра
  ClassName::instanceMethod    — метод произвольного объекта типа (1й аргумент = объект)
  ClassName::new               — конструктор

Самый неочевидный — вид 3:
  String::toUpperCase ≡ (String s) -> s.toUpperCase()
  String::compareTo   ≡ (String s1, String s2) -> s1.compareTo(s2)

Используй когда лямбда = один вызов метода
Не используй ради краткости если теряется ясность
```
