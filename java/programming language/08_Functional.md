# 08 — Функциональное программирование

> [!abstract] Связи
> [[00_JAVA]] | [[06_Collections]] | [[07_Generics]] | [[03_OOP_Advanced]]

---

## 🔹 Функциональные интерфейсы

> [!note] Определение
> Функциональный интерфейс — интерфейс ровно с **одним абстрактным методом**. Может быть реализован лямбда-выражением.

```java
@FunctionalInterface  // аннотация — проверяет что метод один
public interface MyAction {
    void execute();  // единственный абстрактный метод
    // нельзя добавить ещё один abstract метод
}
```

### Стандартные функциональные интерфейсы (java.util.function)

```
Интерфейс            Сигнатура           Метод         Применение
────────────────────────────────────────────────────────────────────
Function<T,R>        T → R               apply(T)      преобразование
Predicate<T>         T → boolean         test(T)       фильтрация/проверка
Consumer<T>          T → void            accept(T)     действие над T
Supplier<T>          () → T              get()         создание/поставка T
UnaryOperator<T>     T → T               apply(T)      преобразование T в T
BiFunction<T,U,R>    (T,U) → R           apply(T,U)    два аргумента
BiPredicate<T,U>     (T,U) → boolean     test(T,U)     проверка двух аргументов
BiConsumer<T,U>      (T,U) → void        accept(T,U)   действие над двумя
BinaryOperator<T>    (T,T) → T           apply(T,T)    операция над двумя T
```

```java
import java.util.function.*;

// Function<T,R> — принимает T, возвращает R
Function<String, Integer> strLen = s -> s.length();
System.out.println(strLen.apply("Hello"));  // 5

// Composing functions
Function<Integer, Integer> doubleIt = n -> n * 2;
Function<Integer, Integer> addTen = n -> n + 10;
Function<Integer, Integer> doubleThenAdd = doubleIt.andThen(addTen);  // сначала double, потом +10
Function<Integer, Integer> addThenDouble = doubleIt.compose(addTen);  // сначала +10, потом double
System.out.println(doubleThenAdd.apply(5)); // 20
System.out.println(addThenDouble.apply(5)); // 30

// Predicate<T> — принимает T, возвращает boolean
Predicate<String> isLong = s -> s.length() > 5;
Predicate<String> startsWithA = s -> s.startsWith("A");

isLong.test("Hello");      // false
isLong.test("Hello World"); // true

// Composing predicates
Predicate<String> longAndStartsWithA = isLong.and(startsWithA);
Predicate<String> longOrStartsWithA = isLong.or(startsWithA);
Predicate<String> notLong = isLong.negate();

// Consumer<T> — принимает T, ничего не возвращает
Consumer<String> printer = s -> System.out.println(s);
Consumer<String> upperPrinter = s -> System.out.println(s.toUpperCase());
printer.accept("hello");   // "hello"

Consumer<String> printTwice = printer.andThen(upperPrinter);
printTwice.accept("hello"); // "hello" затем "HELLO"

// Supplier<T> — ничего не принимает, возвращает T
Supplier<String> greeting = () -> "Hello, World!";
Supplier<Double> random = () -> Math.random();
System.out.println(greeting.get());  // "Hello, World!"
```

---

## 🔹 Лямбда-выражения

> [!note] Определение
> Лямбда — анонимная функция, реализующая функциональный интерфейс. Компактная замена анонимному классу.

### Синтаксис
```java
// (параметры) -> выражение
// (параметры) -> { блок кода; return ...; }

// Без параметров
Runnable r = () -> System.out.println("Running!");

// Один параметр (скобки необязательны)
Consumer<String> print = s -> System.out.println(s);
Consumer<String> printFull = (String s) -> System.out.println(s);  // с типом

// Несколько параметров
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// Блок кода (несколько строк — нужны фигурные скобки и return)
Function<String, String> process = s -> {
    String trimmed = s.trim();
    String upper = trimmed.toUpperCase();
    return upper;
};

// Без return для void
Runnable multiLine = () -> {
    System.out.println("Line 1");
    System.out.println("Line 2");
};
```

### Замыкания (Closure)
```java
// Лямбда может "захватывать" переменные из внешнего контекста
// Переменная должна быть effectively final (не меняться после объявления)
String prefix = "Hello, ";    // effectively final
Function<String, String> greet = name -> prefix + name;  // захват prefix

System.out.println(greet.apply("Alice"));  // "Hello, Alice"

// prefix = "Hi, ";  // ❌ если раскомментировать — ошибка компиляции
```

> [!warning] Подводные камни
> Лямбда захватывает только **effectively final** переменные. Изменение захваченной переменной после объявления лямбды — ошибка компиляции.

---

## 🔹 Ссылки на методы (Method References)

> [!note] Определение
> Компактный синтаксис для лямбд, которые просто вызывают уже существующий метод.

```
Тип ссылки                    Синтаксис                  Эквивалентная лямбда
─────────────────────────────────────────────────────────────────────────────
Статический метод             ClassName::staticMethod    (args) -> ClassName.staticMethod(args)
Метод экземпляра (объект)     object::instanceMethod     (args) -> object.instanceMethod(args)
Метод экземпляра (тип)        ClassName::instanceMethod  (obj, args) -> obj.instanceMethod(args)
Конструктор                   ClassName::new             (args) -> new ClassName(args)
```

```java
// Статический метод
Function<String, Integer> parse = Integer::parseInt;
// эквивалентно: s -> Integer.parseInt(s)
System.out.println(parse.apply("42"));  // 42

// Метод экземпляра конкретного объекта
String prefix = "Hello, ";
Function<String, String> greeter = prefix::concat;
// эквивалентно: s -> prefix.concat(s)
System.out.println(greeter.apply("World"));  // "Hello, World"

// Метод экземпляра произвольного объекта типа
Function<String, String> upper = String::toUpperCase;
// эквивалентно: s -> s.toUpperCase()
System.out.println(upper.apply("hello"));  // "HELLO"

// Конструктор
Supplier<ArrayList<String>> listMaker = ArrayList::new;
// эквивалентно: () -> new ArrayList<>()
ArrayList<String> list = listMaker.get();

// Применение в Stream
List<String> names = List.of("alice", "bob", "charlie");
names.stream()
     .map(String::toUpperCase)       // ClassName::instanceMethod
     .forEach(System.out::println);  // object::instanceMethod
```

---

## 🔹 Stream API

> [!note] Определение
> Stream — конвейер операций над коллекцией данных. Элементы не хранятся. Промежуточные операции **ленивы** (lazy), терминальная запускает весь конвейер.

### Создание стримов
```java
import java.util.stream.*;

// Из коллекции
List<Integer> nums = List.of(1, 2, 3, 4, 5);
Stream<Integer> stream = nums.stream();

// Параллельный стрим
Stream<Integer> parallel = nums.parallelStream();

// Из массива
int[] arr = {1, 2, 3};
IntStream intStream = Arrays.stream(arr);

// Из значений
Stream<String> s = Stream.of("a", "b", "c");

// Диапазон
IntStream range = IntStream.range(0, 10);      // [0, 10) = 0..9
IntStream rangeClosed = IntStream.rangeClosed(1, 5);  // [1, 5] = 1..5

// Бесконечный стрим (с limit!)
Stream<Integer> natural = Stream.iterate(1, n -> n + 1).limit(10);
Stream<Double> randoms = Stream.generate(Math::random).limit(5);
```

### Промежуточные операции (lazy — не выполняются без терминальной)
```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna", "Dave", "Anna");

// filter — отфильтровать
names.stream().filter(s -> s.startsWith("A"));  // Alice, Anna, Anna

// map — преобразовать
names.stream().map(String::toLowerCase);  // alice, bob, charlie...

// distinct — удалить дубликаты
names.stream().distinct();  // Alice, Bob, Charlie, Anna, Dave

// sorted — отсортировать
names.stream().sorted();  // алфавитный порядок
names.stream().sorted(Comparator.reverseOrder()); // обратный
names.stream().sorted(Comparator.comparingInt(String::length)); // по длине

// limit — ограничить количество
names.stream().limit(3);  // Alice, Bob, Charlie

// skip — пропустить первые N
names.stream().skip(2);   // Charlie, Anna, Dave, Anna

// peek — для отладки (не изменяет элементы)
names.stream()
     .filter(s -> s.length() > 3)
     .peek(s -> System.out.println("after filter: " + s))
     .map(String::toUpperCase)
     .peek(s -> System.out.println("after map: " + s))
     .collect(Collectors.toList());

// flatMap — "разворачивает" вложенные коллекции
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4), List.of(5));
List<Integer> flat = nested.stream()
    .flatMap(Collection::stream)  // [[1,2],[3,4],[5]] → [1,2,3,4,5]
    .collect(Collectors.toList());
```

### Терминальные операции (запускают весь конвейер)
```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// collect — собрать результат
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());  // [2, 4, 6, 8, 10]

// toList() — Java 16+ (неизменяемый)
List<Integer> list = numbers.stream().filter(n -> n > 5).toList();

// forEach — выполнить действие
numbers.stream().forEach(System.out::println);

// count — количество элементов
long count = numbers.stream().filter(n -> n > 5).count();  // 5

// reduce — свёртка
int sum = numbers.stream().reduce(0, Integer::sum);  // 55
Optional<Integer> max = numbers.stream().reduce(Integer::max);  // 10

// min / max
Optional<Integer> minimum = numbers.stream().min(Integer::compareTo);
Optional<Integer> maximum = numbers.stream().max(Integer::compareTo);

// findFirst / findAny
Optional<Integer> first = numbers.stream().filter(n -> n > 5).findFirst(); // 6

// anyMatch / allMatch / noneMatch
boolean hasEven = numbers.stream().anyMatch(n -> n % 2 == 0);    // true
boolean allPos = numbers.stream().allMatch(n -> n > 0);           // true
boolean noNeg = numbers.stream().noneMatch(n -> n < 0);           // true

// toArray
Object[] arr = numbers.stream().toArray();
Integer[] typedArr = numbers.stream().toArray(Integer[]::new);
```

### Collectors
```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna", "Dave");

// toList, toSet
List<String> list = names.stream().collect(Collectors.toList());
Set<String> set = names.stream().collect(Collectors.toSet());

// joining — объединить строки
String joined = names.stream().collect(Collectors.joining(", "));  // "Alice, Bob, Charlie, Anna, Dave"
String wrapped = names.stream().collect(Collectors.joining(", ", "[", "]")); // "[Alice, Bob, ...]"

// groupingBy — группировка
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));
// {5=[Alice, Charlie? нет], 3=[Bob], ...}

// partitioningBy — разделить на true/false
Map<Boolean, List<String>> partition = names.stream()
    .collect(Collectors.partitioningBy(s -> s.startsWith("A")));
// {true=[Alice, Anna], false=[Bob, Charlie, Dave]}

// counting
Map<Integer, Long> countByLength = names.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));

// toMap
Map<String, Integer> nameLengths = names.stream()
    .collect(Collectors.toMap(s -> s, String::length));
// {Alice=5, Bob=3, ...}
```

### Числовые специализированные стримы
```java
// IntStream, LongStream, DoubleStream — без boxing overhead
int[] arr = {1, 2, 3, 4, 5};

int sum = IntStream.of(arr).sum();         // 15
double avg = IntStream.of(arr).average().getAsDouble();  // 3.0
int max = IntStream.of(arr).max().getAsInt(); // 5
IntSummaryStatistics stats = IntStream.of(arr).summaryStatistics();
// count=5, sum=15, min=1, max=5, average=3.0

// mapToInt — преобразовать Stream<T> в IntStream
List<String> words = List.of("hello", "world", "java");
int totalChars = words.stream().mapToInt(String::length).sum();  // 14
```

---

## 🔹 Optional<T>

> [!note] Определение
> `Optional` — контейнер для значения, которое **может быть null**. Заменяет явную проверку на null и делает код явным.

```java
import java.util.Optional;

// Создание
Optional<String> present = Optional.of("Hello");      // NPE если null
Optional<String> maybe = Optional.ofNullable(null);   // OK если null
Optional<String> empty = Optional.empty();

// Проверка
present.isPresent();    // true
empty.isPresent();      // false
present.isEmpty();      // false (Java 11+)

// Получение значения
String val = present.get();             // "Hello" (NPE если empty!)
String safe = maybe.orElse("default");  // "default" если empty
String computed = maybe.orElseGet(() -> computeDefault()); // ленивое вычисление
String orThrow = maybe.orElseThrow(() -> new RuntimeException("Нет значения"));

// Преобразование
Optional<Integer> length = present.map(String::length);  // Optional[5]
Optional<String> upper = present.map(String::toUpperCase); // Optional["HELLO"]

// flatMap — для вложенных Optional
Optional<Optional<String>> nested = Optional.of(Optional.of("value"));
Optional<String> flat = nested.flatMap(o -> o);

// filter
Optional<String> longName = present.filter(s -> s.length() > 3);  // Optional["Hello"]
Optional<String> shortName = present.filter(s -> s.length() > 10); // Optional.empty

// ifPresent
present.ifPresent(s -> System.out.println("Value: " + s));

// ifPresentOrElse (Java 9+)
present.ifPresentOrElse(
    s -> System.out.println("Value: " + s),
    () -> System.out.println("No value")
);
```

### Практический пример
```java
// Без Optional
public String getUserEmail(long userId) {
    User user = userRepo.findById(userId);
    if (user == null) return "default@email.com";
    Address address = user.getAddress();
    if (address == null) return "default@email.com";
    return address.getEmail();
}

// С Optional
public String getUserEmail(long userId) {
    return userRepo.findById(userId)           // Optional<User>
        .map(User::getAddress)                  // Optional<Address>
        .map(Address::getEmail)                 // Optional<String>
        .orElse("default@email.com");
}
```

> [!warning] Подводные камни
> - `Optional.get()` без проверки `isPresent()` — то же что NPE
> - Не используй Optional как поле класса — для этого есть `null`
> - Не используй Optional как параметр метода — это усложняет API
> - Optional — для возвращаемых значений методов, где результат может отсутствовать

---

## 🔹 Imperative vs Functional стиль

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna", "Dave", "Barry");

// Задача: получить отсортированные имена длиной > 3, в верхнем регистре

// Imperative (императивный)
List<String> result = new ArrayList<>();
for (String name : names) {
    if (name.length() > 3) {
        result.add(name.toUpperCase());
    }
}
Collections.sort(result);

// Functional (функциональный)
List<String> result2 = names.stream()
    .filter(name -> name.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());

// Оба дают: [ALICE, BARRY, CHARLIE, DAVE]
```

| | Imperative | Functional |
|-|------------|------------|
| Читаемость | Больше кода | Декларативный (что, не как) |
| Изменяемое состояние | Да (accumulator list) | Нет |
| Параллелизация | Сложно | Легко (`.parallelStream()`) |
| Отладка | Проще (точки останова) | Сложнее (`.peek()` помогает) |
