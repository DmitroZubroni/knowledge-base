> **Теги:** #java #functional #streams #collectors #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Functional_Index]]

# 02 — Stream API

---

## 🔹 Что такое Stream и зачем он нужен

Stream — это **конвейер операций** над источником данных (коллекцией, массивом, файлом). Он не хранит данные — он описывает что с ними сделать.

Сравни два подхода для одной задачи: "найти имена сотрудников старше 30 лет, отсортировать и вывести":

```java
// Императивный стиль — описываем КАК делать
List<String> result = new ArrayList<>();
for (Employee e : employees) {
    if (e.getAge() > 30) {
        result.add(e.getName());
    }
}
Collections.sort(result);
for (String name : result) {
    System.out.println(name);
}

// Функциональный стиль (Stream) — описываем ЧТО делать
employees.stream()
    .filter(e -> e.getAge() > 30)
    .map(Employee::getName)
    .sorted()
    .forEach(System.out::println);
```

Функциональный стиль: меньше кода, нет изменяемых переменных, легко распараллелить.

---

## 🔹 Структура pipeline

Любой стрим состоит из трёх частей:

```
Источник → [Промежуточные операции]* → Терминальная операция

list.stream()          — источник
    .filter(...)       — промежуточная (lazy)
    .map(...)          — промежуточная (lazy)
    .sorted()          — промежуточная (lazy)
    .collect(...)      — терминальная → запускает весь конвейер
```

> [!note] Ключевое свойство — LAZY (ленивость)
> Промежуточные операции **не выполняются** пока не вызвана терминальная. Это позволяет оптимизировать pipeline — например, при `findFirst()` после `filter()` поток остановится на первом подходящем элементе, не обрабатывая остальные.

```java
// Демонстрация ленивости
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

Stream<Integer> stream = numbers.stream()
    .filter(n -> {
        System.out.println("filter: " + n);
        return n % 2 == 0;
    })
    .map(n -> {
        System.out.println("map: " + n);
        return n * 10;
    });

// Ничего не напечатано! Поток ещё не запущен.
System.out.println("Before terminal");

stream.findFirst(); // Только теперь запускается

// Вывод:
// Before terminal
// filter: 1
// filter: 2
// map: 2        ← как только нашли первый подходящий — остановились!
// Элементы 3,4,5 вообще не обрабатывались
```

---

## 🔹 Создание стримов

```java
// Из коллекции
List<String> list = List.of("a", "b", "c");
Stream<String> s1 = list.stream();
Stream<String> s2 = list.parallelStream(); // параллельный

// Из массива
String[] arr = {"a", "b", "c"};
Stream<String> s3 = Arrays.stream(arr);
Stream<String> s4 = Stream.of("a", "b", "c");

// Диапазоны чисел (IntStream, LongStream)
IntStream range1 = IntStream.range(0, 10);      // [0, 10) — 0..9
IntStream range2 = IntStream.rangeClosed(1, 5); // [1, 5] — 1..5

// Бесконечные стримы (обязательно с limit!)
Stream<Integer> naturals  = Stream.iterate(1, n -> n + 1).limit(10);
Stream<Integer> evens     = Stream.iterate(0, n -> n + 2).limit(5); // 0,2,4,6,8
Stream<Double>  randoms   = Stream.generate(Math::random).limit(5);

// iterate с предикатом (Java 9+)
Stream<Integer> underHundred = Stream.iterate(1, n -> n < 100, n -> n * 2);
// 1, 2, 4, 8, 16, 32, 64 — останавливается когда n >= 100

// Из строки — по символам (Java 9+)
"hello".chars().forEach(c -> System.out.print((char)c));

// Из файла — по строкам
try (Stream<String> lines = Files.lines(Path.of("file.txt"))) {
    lines.filter(l -> !l.isBlank()).forEach(System.out::println);
}
```

---

## 🔹 Промежуточные операции

### filter — фильтрация

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

numbers.stream()
    .filter(n -> n % 2 == 0)   // только чётные
    .collect(Collectors.toList()); // [2, 4, 6, 8, 10]
```

### map — преобразование каждого элемента

```java
List<String> names = List.of("alice", "bob", "charlie");

names.stream()
    .map(String::toUpperCase)         // String → String
    .collect(Collectors.toList());    // [ALICE, BOB, CHARLIE]

// map меняет ТИП элементов
List<Integer> lengths = names.stream()
    .map(String::length)              // String → Integer
    .collect(Collectors.toList());    // [5, 3, 7]
```

### flatMap — "разворачивание" вложенных структур

`flatMap` — самая часто непонимаемая операция. Она нужна когда `map` возвращает коллекцию/стрим, и ты хочешь "плоский" результат вместо вложенного.

```java
// Проблема: map возвращает Stream<Stream<String>>
List<List<String>> nested = List.of(
    List.of("a", "b"),
    List.of("c", "d"),
    List.of("e")
);

// ❌ map — получаем Stream<List<String>>, не то что нужно
Stream<List<String>> wrong = nested.stream().map(Collection::stream); // Stream<Stream>

// ✅ flatMap — "разворачивает" каждый внутренний стрим
List<String> flat = nested.stream()
    .flatMap(Collection::stream)          // [[a,b],[c,d],[e]] → [a,b,c,d,e]
    .collect(Collectors.toList());
```

```java
// Реальный пример: все слова из всех предложений
List<String> sentences = List.of(
    "Hello World",
    "Java Stream API",
    "flatMap is useful"
);

List<String> words = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .collect(Collectors.toList());
// [Hello, World, Java, Stream, API, flatMap, is, useful]

// Пример: все заказы всех пользователей
List<Order> allOrders = users.stream()
    .flatMap(user -> user.getOrders().stream())
    .collect(Collectors.toList());
```

> [!tip] Правило для flatMap
> Используй `flatMap` вместо `map` когда функция-маппер возвращает `Collection<T>` или `Stream<T>`, и ты хочешь получить `Stream<T>`, а не `Stream<Collection<T>>`.

### sorted — сортировка

```java
List<String> names = List.of("Charlie", "Alice", "Bob", "Dave");

// Естественный порядок
names.stream().sorted().collect(Collectors.toList());
// [Alice, Bob, Charlie, Dave]

// Обратный порядок
names.stream().sorted(Comparator.reverseOrder()).collect(Collectors.toList());
// [Dave, Charlie, Bob, Alice]

// По длине строки
names.stream()
    .sorted(Comparator.comparingInt(String::length))
    .collect(Collectors.toList());
// [Bob, Alice, Dave, Charlie]

// По длине, при одинаковой — по алфавиту
names.stream()
    .sorted(Comparator.comparingInt(String::length)
        .thenComparing(Comparator.naturalOrder()))
    .collect(Collectors.toList());

// Сортировка объектов
employees.stream()
    .sorted(Comparator.comparing(Employee::getDepartment)
        .thenComparing(Employee::getSalary, Comparator.reverseOrder()))
    .collect(Collectors.toList());
```

### distinct, limit, skip

```java
List<Integer> nums = List.of(1, 2, 2, 3, 3, 3, 4);

nums.stream().distinct().collect(Collectors.toList()); // [1, 2, 3, 4]
nums.stream().limit(3).collect(Collectors.toList());   // [1, 2, 2]
nums.stream().skip(3).collect(Collectors.toList());    // [3, 3, 3, 4]

// Пагинация через skip + limit
int page = 2, pageSize = 10;
list.stream()
    .skip((long)(page - 1) * pageSize)
    .limit(pageSize)
    .collect(Collectors.toList());
```

### peek — отладка конвейера

```java
// peek не меняет элементы, используется для логирования/отладки
List<String> result = names.stream()
    .filter(s -> s.length() > 3)
    .peek(s -> System.out.println("after filter: " + s))
    .map(String::toUpperCase)
    .peek(s -> System.out.println("after map: " + s))
    .collect(Collectors.toList());

// Не используй peek в продакшн-коде для побочных эффектов — только для отладки
```

---

## 🔹 Терминальные операции

### collect — сборка результата

```java
Stream<String> stream = names.stream().filter(s -> s.length() > 3);

List<String> list = stream.collect(Collectors.toList());     // в List
Set<String>  set  = stream.collect(Collectors.toSet());      // в Set (дубликаты удалятся)
LinkedList<String> ll = stream.collect(Collectors.toCollection(LinkedList::new)); // в конкретную реализацию
```

### forEach, count, min, max

```java
// forEach
numbers.stream().forEach(System.out::println); // порядок в parallel не гарантирован
numbers.stream().forEachOrdered(System.out::println); // порядок гарантирован

// count
long count = numbers.stream().filter(n -> n > 5).count(); // 5

// min, max — возвращают Optional
Optional<Integer> min = numbers.stream().min(Integer::compareTo);
Optional<Integer> max = numbers.stream().max(Comparator.naturalOrder());
min.ifPresent(v -> System.out.println("Min: " + v));
```

### reduce — свёртка

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

// reduce(identity, accumulator) — с начальным значением
int sum = nums.stream().reduce(0, Integer::sum);       // 15
int product = nums.stream().reduce(1, (a, b) -> a * b); // 120

// reduce(accumulator) — без начального значения, возвращает Optional
Optional<Integer> max = nums.stream().reduce(Integer::max); // Optional[5]

// Конкатенация строк через reduce (лучше используй Collectors.joining)
String concat = List.of("a","b","c").stream()
    .reduce("", String::concat); // "abc"
```

### findFirst, findAny, anyMatch, allMatch, noneMatch

```java
// find — возвращают Optional
Optional<Integer> first = numbers.stream().filter(n -> n > 5).findFirst();  // Optional[6]
Optional<Integer> any   = numbers.stream().filter(n -> n > 5).findAny();    // любой > 5

// match — возвращают boolean, останавливаются при первом несоответствии (short-circuit)
boolean hasEven   = numbers.stream().anyMatch(n -> n % 2 == 0);   // true — есть хоть один
boolean allPos    = numbers.stream().allMatch(n -> n > 0);          // true — все > 0
boolean noneNeg   = numbers.stream().noneMatch(n -> n < 0);         // true — ни одного < 0
```

---

## 🔹 Collectors — мощные сборщики

### joining — объединение строк

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

String joined  = names.stream().collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"

String wrapped = names.stream().collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie]"
```

### groupingBy — группировка

```java
List<Employee> employees = ...; // каждый сотрудник имеет department, salary, name

// Сгруппировать по отделу
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));
// {"Engineering": [e1, e2], "Sales": [e3, e4, e5]}

// Сгруппировать и подсчитать
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()          // downstream collector
    ));
// {"Engineering": 2, "Sales": 3}

// Сгруппировать и взять среднюю зарплату
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// Сгруппировать по возрасту, получить только имена
Map<Integer, List<String>> namesByAge = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getAge,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));

// Многоуровневая группировка
Map<String, Map<String, List<Employee>>> byDeptAndCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getCity)
    ));
```

### partitioningBy — разделение на два

```java
// Разделить на две группы по предикату
Map<Boolean, List<Employee>> partition = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100_000));

List<Employee> highPaid = partition.get(true);
List<Employee> lowPaid  = partition.get(false);
```

### toMap — в Map

```java
// toMap(keyMapper, valueMapper)
Map<String, Integer> namesToLength = names.stream()
    .collect(Collectors.toMap(
        s -> s,          // ключ — само имя
        String::length   // значение — длина
    ));
// {"Alice": 5, "Bob": 3, "Charlie": 7}

// toMap с обработкой дубликатов (3й аргумент — merge функция)
Map<String, Long> wordFreq = words.stream()
    .collect(Collectors.toMap(
        w -> w,                    // ключ
        w -> 1L,                   // значение
        Long::sum                  // если ключ уже есть — сложить
    ));

// toMap с конкретной реализацией Map
Map<String, Integer> linkedMap = names.stream()
    .collect(Collectors.toMap(
        s -> s,
        String::length,
        (v1, v2) -> v1,        // при дубликате — оставить первый
        LinkedHashMap::new     // использовать LinkedHashMap
    ));
```

### summarizingInt/Double/Long — статистика

```java
IntSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingInt(Employee::getAge));

stats.getCount();   // количество
stats.getSum();     // сумма
stats.getMin();     // минимум
stats.getMax();     // максимум
stats.getAverage(); // среднее
```

### Collectors.teeing — два коллектора в одном (Java 12+)

```java
// Применить два коллектора одновременно и объединить результаты
record MinMax(int min, int max) {}

MinMax minMax = numbers.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Integer::compareTo),   // первый коллектор
        Collectors.maxBy(Integer::compareTo),   // второй коллектор
        (min, max) -> new MinMax(               // объединение результатов
            min.orElse(0),
            max.orElse(0)
        )
    ));
```

---

## 🔹 Числовые стримы — IntStream, LongStream, DoubleStream

Избегают boxing Integer/Long/Double — важно для производительности на больших объёмах.

```java
// Создание
IntStream.of(1, 2, 3, 4, 5)
IntStream.range(0, 10)        // [0, 10)
IntStream.rangeClosed(1, 10)  // [1, 10]

// Специальные операции (нет в обычном Stream)
IntStream ints = IntStream.rangeClosed(1, 10);
int sum     = ints.sum();             // 55
double avg  = ints.average().getAsDouble(); // 5.5
int max     = ints.max().getAsInt();  // 10
IntSummaryStatistics stats = ints.summaryStatistics();

// Преобразование Stream<T> → IntStream
List<String> words = List.of("hello", "world");
int totalChars = words.stream()
    .mapToInt(String::length)  // Stream<String> → IntStream
    .sum();                     // 10

// IntStream → Stream<String>
IntStream.range(1, 6)
    .mapToObj(n -> "Item " + n)  // IntStream → Stream<String>
    .forEach(System.out::println);
// Item 1, Item 2, Item 3, Item 4, Item 5

// boxed() — IntStream → Stream<Integer>
List<Integer> list = IntStream.range(1, 6)
    .boxed()
    .collect(Collectors.toList());
```

---

## 🔹 Параллельные стримы

```java
// Создание
list.parallelStream()
list.stream().parallel()

// Пример
long count = LongStream.rangeClosed(1, 1_000_000)
    .parallel()
    .filter(n -> n % 2 == 0)
    .count(); // быстрее на многоядерных CPU
```

> [!warning] Параллельные стримы — не всегда быстрее
> Параллелизм имеет накладные расходы на разбиение данных и объединение результатов. Параллельный стрим **быстрее** при:
> - Большом объёме данных (миллионы элементов)
> - Тяжёлых вычислениях на каждый элемент
> - Независимых операциях (нет состояния, нет синхронизации)
>
> Параллельный стрим **медленнее или некорректен** при:
> - Маленьких коллекциях — overhead перевешивает выгоду
> - Операциях с общим изменяемым состоянием (гонка данных!)
> - Операциях ввода-вывода (I/O bound — поток не работает быстрее)
> - Порядке важен — `forEachOrdered` нивелирует параллелизм

```java
// ❌ Опасно — гонка данных
List<Integer> result = new ArrayList<>();
numbers.parallelStream().forEach(result::add); // ArrayList не потокобезопасен!

// ✅ Безопасно — collect потокобезопасен
List<Integer> result = numbers.parallelStream()
    .filter(n -> n > 5)
    .collect(Collectors.toList()); // collect работает корректно в parallel
```

---

## 🔹 Типичные паттерны

### Подсчёт частоты слов

```java
Map<String, Long> freq = Arrays.stream(text.split("\\s+"))
    .collect(Collectors.groupingBy(
        String::toLowerCase,
        Collectors.counting()
    ));
```

### Топ-N элементов

```java
List<Employee> top5 = employees.stream()
    .sorted(Comparator.comparing(Employee::getSalary).reversed())
    .limit(5)
    .collect(Collectors.toList());
```

### Проверка: все элементы уникальны?

```java
boolean allUnique = list.stream()
    .allMatch(new HashSet<>()::add); // add возвращает false при дубликате
```

### Преобразование List в Map

```java
Map<Long, User> usersById = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u));
```

### Flatten и дедупликация

```java
List<String> unique = departments.stream()
    .flatMap(dept -> dept.getEmployees().stream())
    .map(Employee::getSkill)
    .distinct()
    .sorted()
    .collect(Collectors.toList());
```

---

## 🔹 Итог

```
Stream = конвейер: источник → промежуточные операции → терминальная

Ленивость: промежуточные операции не выполняются без терминальной
Short-circuit: findFirst/anyMatch могут остановить конвейер досрочно

Промежуточные (возвращают Stream):
  filter(pred)         — фильтрация
  map(fn)              — преобразование
  flatMap(fn)          — flatten (fn возвращает Stream)
  sorted(comparator)   — сортировка
  distinct()           — уникальные
  limit(n) / skip(n)   — ограничение / пропуск
  peek(fn)             — отладка

Терминальные (запускают конвейер):
  collect(collector)   — сборка в коллекцию/map/строку
  forEach(fn)          — действие над каждым
  count()              — количество
  min/max(comparator)  — Optional с мин/макс
  reduce(id, fn)       — свёртка
  findFirst/findAny()  — Optional с первым/любым
  anyMatch/allMatch/noneMatch — boolean проверки

Collectors:
  toList/toSet/toCollection
  joining(sep, prefix, suffix)
  groupingBy(classifier, downstream)
  partitioningBy(predicate)
  toMap(keyFn, valueFn, mergeFn)
  counting / summarizingInt / averagingInt

Числовые стримы (IntStream/LongStream/DoubleStream):
  sum(), average(), min(), max(), summaryStatistics()
  mapToInt/mapToObj/boxed()

Параллельные: parallelStream() — только для больших независимых данных
```
