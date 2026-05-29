# Streams

> **Теги:** #interviews #java-core #streams #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Lazy Evaluation

**Lazy Evaluation** — отложенное выполнение операций в Stream.

### Как работает

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

names.stream()
     .filter(name -> {
         System.out.println("Filtering: " + name);
         return name.length() > 3;
     })
     .map(name -> {
         System.out.println("Mapping: " + name);
         return name.toUpperCase();
     })
     .collect(Collectors.toList());  // только здесь выполняется!
```

**Результат:**
```
Filtering: Alice
Mapping: Alice
Filtering: Bob
Mapping: Bob
Filtering: Charlie
Mapping: Charlie
```

> [!tip] Преимущества Lazy Evaluation
- Не обрабатывает элементы, которые не нужны (short-circuiting)
- Оптимизация цепочки операций
- Возможность работы с бесконечными потоками

---

## 🔹 Промежуточные vs Терминальные операции

### Промежуточные (Intermediate)

Возвращают новый Stream, ленивые.

```java
.filter()    // фильтрация
.map()       // преобразование
flatMap()    // преобразование в поток
sorted()     // сортировка
distinct()   // удаление дубликатов
limit()      // ограничение
skip()       // пропуск
peek()       // отладка
```

### Терминальные (Terminal)

Запускают выполнение Stream, не возвращают Stream.

```java
collect()    // сбор в коллекцию
forEach()    // итерация
reduce()     // свёртка
count()      // подсчёт
anyMatch()   // проверка условия
allMatch()   // все ли соответствуют
noneMatch()  // ни один не соответствует
findFirst()  // первый элемент
findAny()    // любой элемент
min() / max() // минимум/максимум
toArray()    // в массив
```

### Пример

```java
// Промежуточные → можно продолжать цепочку
Stream<String> stream = names.stream()
                            .filter(name -> name.length() > 3)
                            .map(String::toUpperCase);

// Терминальные → завершают Stream
List<String> result = stream.collect(Collectors.toList());
```

> [!warning] Stream можно использовать только один раз
```java
Stream<String> stream = names.stream();
stream.forEach(System.out::println);  // OK
stream.forEach(System.out::println);  // IllegalStateException!
```

---

## 🔹 map vs flatMap

### map

Преобразует каждый элемент в другой элемент (1:1).

```java
List<String> words = List.of("hello", "world");

List<Integer> lengths = words.stream()
                              .map(String::length)  // "hello" → 5
                              .collect(Collectors.toList());
// [5, 5]
```

### flatMap

Преобразует каждый элемент в поток и "выравнивает" (1:N).

```java
List<String> sentences = List.of("hello world", "java streams");

List<String> words = sentences.stream()
                             .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
                             .collect(Collectors.toList());
// ["hello", "world", "java", "streams"]
```

### Сравнение

| Операция | Вход | Выход | Соотношение |
|----------|------|-------|-------------|
| **map** | `Stream<T>` | `Stream<R>` | 1 элемент → 1 элемент |
| **flatMap** | `Stream<T>` | `Stream<R>` | 1 элемент → N элементов |

### Пример с вложенными структурами

```java
List<List<Integer>> nested = List.of(
    List.of(1, 2),
    List.of(3, 4),
    List.of(5)
);

// map → Stream<List<Integer>>
Stream<List<Integer>> mapped = nested.stream().map(list -> list);

// flatMap → Stream<Integer> (выравнивает)
Stream<Integer> flattened = nested.stream()
                                  .flatMap(List::stream);
// [1, 2, 3, 4, 5]
```

---

## 🔹 Optional в Stream

### findFirst()

```java
Optional<String> first = names.stream()
                               .filter(name -> name.startsWith("A"))
                               .findFirst();

if (first.isPresent()) {
    System.out.println(first.get());
}
```

### orElse()

```java
String result = names.stream()
                     .filter(name -> name.startsWith("Z"))
                     .findFirst()
                     .orElse("Default");
```

### orElseThrow()

```java
String result = names.stream()
                     .filter(name -> name.startsWith("A"))
                     .findFirst()
                     .orElseThrow(() -> new NotFoundException());
```

---

## 🔹 IntStream, LongStream, DoubleStream

**Специализированные потоки** для примитивов, избегают boxing/unboxing.

### Создание

```java
IntStream.range(0, 10)           // 0-9
IntStream.rangeClosed(0, 10)    // 0-10
IntStream.of(1, 2, 3, 4, 5)     // конкретные значения
Arrays.stream(new int[]{1, 2, 3}) // из массива
"hello".chars()                   // IntStream символов
```

### Операции

```java
IntStream.range(0, 10)
          .filter(n -> n % 2 == 0)
          .map(n -> n * n)
          .sum();  // сумма

IntStream.range(0, 10)
          .average()  // OptionalDouble
          .ifPresent(System.out::println);

IntStream.range(0, 10)
          .max()  // OptionalInt
          .ifPresent(System.out::println);
```

### Преобразование в Stream

```java
IntStream.range(0, 10)
          .boxed()  // int → Integer
          .collect(Collectors.toList());
```

---

## 🔹 Сортировка

### sorted()

```java
List<String> sorted = names.stream()
                           .sorted()  // естественный порядок
                           .collect(Collectors.toList());

List<String> reversed = names.stream()
                             .sorted(Comparator.reverseOrder())
                             .collect(Collectors.toList());
```

### sorted(Comparator)

```java
List<Person> sorted = people.stream()
                           .sorted(Comparator.comparing(Person::getAge))
                           .collect(Collectors.toList());

// Множественная сортировка
List<Person> multiSort = people.stream()
                                .sorted(Comparator
                                    .comparing(Person::getAge)
                                    .thenComparing(Person::getName))
                                .collect(Collectors.toList());
```

---

## 🔹 Collectors

### toList()

```java
List<String> list = names.stream()
                         .collect(Collectors.toList());
```

### toSet()

```java
Set<String> set = names.stream()
                       .collect(Collectors.toSet());
```

### toMap()

```java
Map<String, Integer> map = names.stream()
                                 .collect(Collectors.toMap(
                                     name -> name,      // key mapper
                                     String::length,   // value mapper
                                     (existing, replacement) -> existing  // merge function
                                 ));
```

### groupingBy()

```java
Map<Integer, List<String>> grouped = names.stream()
                                           .collect(Collectors.groupingBy(String::length));
// {3: ["Bob"], 5: ["Alice"], 7: ["Charlie"]}
```

### joining()

```java
String joined = names.stream()
                     .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"
```

---

## 🔹 Параллельные Stream

**Parallel Stream** — выполняется в нескольких потоках.

### Использование

```java
List<Integer> numbers = IntStream.range(0, 1000000).boxed().collect(Collectors.toList());

long sum = numbers.parallelStream()
                  .mapToInt(Integer::intValue)
                  .sum();
```

### Когда использовать

- **Да:** большие объёмы данных, CPU-bound операции
- **Нет:** маленькие данные, I/O-bound операции (overhead > пользы)

> [!warning] Parallel Stream pitfalls
- Порядок элементов не гарантируется
- Thread-safety операций
- Не всегда быстрее (overhead)

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Lazy Evaluation** — отложенное выполнение, оптимизация
> - **Промежуточные:** filter, map, sorted, distinct
> - **Терминальные:** collect, forEach, reduce, count
> - **map** — 1:1, **flatMap** — 1:N (выравнивание)
> - **IntStream** — примитивы, избегает boxing
> - **Collectors:** toList, toSet, groupingBy, joining

```
Lazy Evaluation:
операции не выполняются до терминальной операции

map vs flatMap:
map: Stream<T> → Stream<R> (1:1)
flatMap: Stream<T> → Stream<R> (1:N, выравнивание)

Промежуточные:
filter, map, sorted, distinct, limit, skip

Терминальные:
collect, forEach, reduce, count, findFirst

IntStream:
range, of, average, max, min, sum
```
