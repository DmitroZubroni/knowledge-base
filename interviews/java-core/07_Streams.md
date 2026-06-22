> **Теги:** #interviews #java-core #streams #functional #конспект
> [!abstract] Связи
> [[main]] | [[Interviews]] | [[Functional_Index]] | [[02_StreamAPI]]

# Stream API & Functional — Вопросы на собесе

---

## 🔹 Ключевые свойства Stream

**Stream — это не коллекция.** Он не хранит данные — описывает конвейер операций над источником.

```
Источник → [промежуточные операции (lazy)] → терминальная операция (запускает всё)

list.stream()          ← источник
    .filter(...)       ← lazy, ничего не выполняется
    .map(...)          ← lazy, ничего не выполняется
    .collect(...)      ← терминальная → только здесь pipeline запускается
```

**Три ключевых свойства:**
1. **Lazy** — промежуточные операции выполняются только при вызове терминальной
2. **One-time** — Stream нельзя использовать дважды (`IllegalStateException`)
3. **Non-interfering** — не изменяй источник во время работы Stream

```java
Stream<String> s = list.stream().filter(x -> x.length() > 3);
s.forEach(System.out::println); // OK
s.forEach(System.out::println); // ❌ IllegalStateException: stream already operated upon
```

---

## 🔹 Lazy + Short-circuit — демонстрация

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

Optional<Integer> result = numbers.stream()
    .filter(n -> {
        System.out.println("filter: " + n);
        return n % 2 == 0;
    })
    .map(n -> {
        System.out.println("map: " + n);
        return n * 10;
    })
    .findFirst(); // short-circuit — останавливается на первом

// Вывод:
// filter: 1   ← нечётный, отброшен
// filter: 2   ← чётный, прошёл
// map: 2      ← обработан
// (остановились! 3, 4, 5 не обрабатывались)
// result = Optional[20]
```

**Short-circuit операции** — останавливают конвейер досрочно:
- `findFirst()`, `findAny()` — останавливаются как нашли
- `anyMatch()`, `allMatch()`, `noneMatch()` — останавливаются как решение очевидно
- `limit(n)` — останавливается после n элементов

---

## 🔹 map vs flatMap — главный вопрос на собесе

```java
// map: каждый элемент → один результат (1:1)
List<String> words = List.of("hello", "world");
words.stream().map(String::toUpperCase); // → [HELLO, WORLD]

// map с функцией возвращающей Stream → Stream<Stream<T>>
words.stream().map(s -> Arrays.stream(s.split("")));
// → Stream<Stream<String>> ← ВЛОЖЕНИЕ — обычно не то что нужно

// flatMap: каждый элемент → Stream → результаты объединяются (1:N)
words.stream().flatMap(s -> Arrays.stream(s.split("")));
// → Stream<String> [h, e, l, l, o, w, o, r, l, d] ← ПЛОСКО
```

**Правило:** если функция в `map()` возвращает `Collection<T>` или `Stream<T>` — используй `flatMap()`.

```java
// Все заказы всех пользователей
List<Order> allOrders = users.stream()
    .flatMap(user -> user.getOrders().stream()) // User → Stream<Order>
    .collect(Collectors.toList());

// Все слова из всех предложений
List<String> allWords = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .collect(Collectors.toList());
```

---

## 🔹 reduce — свёртка

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

// reduce(identity, accumulator) — identity: начальное значение
int sum     = nums.stream().reduce(0, Integer::sum);         // 15
int product = nums.stream().reduce(1, (a, b) -> a * b);     // 120
int max     = nums.stream().reduce(Integer.MIN_VALUE, Math::max); // 5

// reduce(accumulator) — без identity → Optional (может быть пустой stream)
Optional<Integer> max2 = nums.stream().reduce(Integer::max); // Optional[5]

// Конкатенация строк через reduce (лучше используй Collectors.joining)
String concat = Stream.of("a","b","c").reduce("", String::concat); // "abc"
```

**Когда reduce, а когда collect:**
- `reduce` — агрегировать в одно значение (sum, max, product)
- `collect` — собрать в коллекцию или сложную структуру

---

## 🔹 Collectors — что спрашивают глубоко

### groupingBy с downstream

```java
List<Employee> employees = ...;

// Простая группировка
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// С downstream — счётчик
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// С downstream — средняя зарплата
Map<String, Double> avgSalary = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// С downstream — только имена
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));

// Двухуровневая группировка
Map<String, Map<String, List<Employee>>> byDeptAndCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getCity)
    ));
```

### partitioningBy — разделение на два

```java
Map<Boolean, List<Employee>> partition = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100_000));

List<Employee> highPaid = partition.get(true);
List<Employee> lowPaid  = partition.get(false);
```

### toMap — подводные камни

```java
// ❌ Дубликаты ключей → IllegalStateException
names.stream().collect(Collectors.toMap(s -> s, String::length));
// если дубликаты есть — упадёт!

// ✅ Передать merge function
names.stream().collect(Collectors.toMap(
    s -> s,
    String::length,
    (v1, v2) -> v1  // при дубликате: оставить первый
));

// ✅ С конкретной реализацией Map
names.stream().collect(Collectors.toMap(
    s -> s,
    String::length,
    (v1, v2) -> v1,
    LinkedHashMap::new  // сохранить порядок вставки
));
```

### joining

```java
String result = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie]"
```

### Подсчёт частоты — классический паттерн

```java
String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};
Map<String, Long> freq = Arrays.stream(words)
    .collect(Collectors.groupingBy(s -> s, Collectors.counting()));
// {apple=3, banana=2, cherry=1}

// Через merge — если нужен Integer а не Long
Map<String, Integer> freq2 = new HashMap<>();
Arrays.stream(words).forEach(w -> freq2.merge(w, 1, Integer::sum));
```

---

## 🔹 Функциональные интерфейсы — что спрашивают

```java
// Четыре основных
Function<T, R>    — T → R          apply(t)
Predicate<T>      — T → boolean    test(t)
Consumer<T>       — T → void       accept(t)
Supplier<T>       — () → T         get()

// Производные
UnaryOperator<T>  — T → T          (частный случай Function)
BinaryOperator<T> — (T,T) → T
BiFunction<T,U,R> — (T,U) → R
```

**Композиция:**

```java
// Function: andThen vs compose
Function<Integer, Integer> times2 = n -> n * 2;
Function<Integer, Integer> plus10 = n -> n + 10;

times2.andThen(plus10).apply(5); // (5*2)+10 = 20 — сначала times2, потом plus10
times2.compose(plus10).apply(5); // (5+10)*2 = 30 — сначала plus10, потом times2

// Predicate: and, or, negate
Predicate<String> long5  = s -> s.length() > 5;
Predicate<String> startsA = s -> s.startsWith("A");

long5.and(startsA)      // И
long5.or(startsA)       // ИЛИ
long5.negate()          // НЕ
Predicate.not(String::isEmpty) // Java 11+, для метод-ссылок
```

---

## 🔹 Параллельные стримы — подводные камни

```java
// Создание
list.parallelStream()
list.stream().parallel()
```

**Когда параллельный Stream БЫСТРЕЕ обычного:**
- Большой объём данных (миллионы элементов)
- Тяжёлые CPU-bound вычисления на каждый элемент
- Независимые операции без общего состояния

**Когда параллельный Stream МЕДЛЕННЕЕ или ОПАСЕН:**

```java
// ❌ Маленькая коллекция — overhead ForkJoin > выгода
List<Integer> small = List.of(1, 2, 3, 4, 5);
small.parallelStream().map(n -> n * 2).collect(toList()); // медленнее!

// ❌ Общее изменяемое состояние — race condition!
List<Integer> result = new ArrayList<>();
numbers.parallelStream().forEach(result::add); // ArrayList не потокобезопасен → NPE/потеря данных

// ✅ collect() потокобезопасен в parallel
List<Integer> result = numbers.parallelStream()
    .filter(n -> n > 5)
    .collect(Collectors.toList()); // OK

// ❌ I/O bound операции — потоки ждут I/O, параллелизм не помогает
urls.parallelStream().map(url -> httpClient.get(url)); // не ускорит

// ❌ Порядок не гарантирован в parallel
numbers.parallelStream().forEach(System.out::println); // случайный порядок
numbers.parallelStream().forEachOrdered(System.out::println); // гарантированный порядок, но медленнее
```

**ForkJoinPool:** параллельные стримы используют `ForkJoinPool.commonPool()` — общий на всё JVM. Если заблокировать его в одном месте — всё приложение пострадает.

```java
// Свой пул для изоляции
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() ->
    numbers.parallelStream().map(n -> heavyComputation(n)).collect(toList())
).get();
```

---

## 🔹 Optional — что спрашивают

```java
// Создание
Optional.of(value)          // NPE если null
Optional.ofNullable(value)  // safe
Optional.empty()

// Получение
opt.orElse("default")                    // всегда вычисляется!
opt.orElseGet(() -> computeDefault())    // только если empty (lazy)
opt.orElseThrow(() -> new Ex())
opt.get()  // ❌ NoSuchElementException если empty

// Трансформация без get()
opt.map(String::toUpperCase)             // String → String
opt.flatMap(s -> findInDB(s))            // если метод возвращает Optional
opt.filter(s -> s.length() > 3)

// Действия
opt.ifPresent(s -> save(s))
opt.ifPresentOrElse(s -> save(s), () -> log("empty")) // Java 9+
```

**Антипаттерны Optional:**

```java
// ❌ isPresent() + get() — смысл Optional теряется
if (opt.isPresent()) return opt.get();
else return "default";
// ✅
return opt.orElse("default");

// ❌ Optional как поле класса — не сериализуется
class User { private Optional<String> phone; }
// ✅ поле null, Optional только в возвращаемом значении
class User {
    private String phone;
    public Optional<String> getPhone() { return Optional.ofNullable(phone); }
}

// ❌ Optional как параметр метода — усложняет вызов
void process(Optional<String> name) {}
// ✅ перегрузка или @Nullable
void process(String name) {}
void process() { process(null); }
```

---

## 🔹 Ссылки на методы — четыре вида

```java
// 1. Статический метод
Integer::parseInt         ≡  s -> Integer.parseInt(s)
Math::max                 ≡  (a, b) -> Math.max(a, b)

// 2. Метод конкретного объекта
System.out::println       ≡  s -> System.out.println(s)
logger::info              ≡  s -> logger.info(s)

// 3. Метод произвольного объекта типа (самый неочевидный!)
String::toUpperCase       ≡  s -> s.toUpperCase()
String::compareTo         ≡  (s1, s2) -> s1.compareTo(s2)
// Первый аргумент = объект на котором вызывается метод

// 4. Конструктор
ArrayList::new            ≡  () -> new ArrayList<>()
StringBuilder::new        ≡  s -> new StringBuilder(s)
```

---

## 🔹 Типичные вопросы и ответы

**Q: Чем отличается Stream от Collection?**
A: Collection хранит данные в памяти. Stream — конвейер вычислений, данные не хранит. Stream можно использовать только один раз, Collection — многократно. Collection знает свой размер, бесконечный Stream — нет.

**Q: Что такое lazy evaluation и зачем она нужна?**
A: Промежуточные операции не выполняются пока нет терминальной. Это позволяет: 1) short-circuit — остановиться досрочно при `findFirst()`/`anyMatch()`; 2) оптимизировать pipeline — JVM может переставить/объединить операции; 3) работать с бесконечными стримами.

**Q: Почему forEach в параллельном Stream опасен?**
A: `forEach` не гарантирует порядок и не является thread-safe. При записи в `ArrayList` из параллельного `forEach` — race condition. Правильно: `collect(Collectors.toList())` — он thread-safe.

**Q: В чём разница orElse и orElseGet?**
A: `orElse(value)` — value **всегда вычисляется**, даже если Optional не пустой. `orElseGet(supplier)` — вычисляется только если Optional пустой (lazy). При дорогом вычислении default-значения — всегда `orElseGet`.

```java
// orElse — DB запрос выполнится даже если user найден!
findUser(id).orElse(createDefaultUser());

// orElseGet — DB запрос только при отсутствии user
findUser(id).orElseGet(() -> createDefaultUser());
```

**Q: Что будет если вызвать collect(Collectors.toList()) на пустом Stream?**
A: Вернёт пустой список `[]`. Не null, не исключение.

**Q: Чем partitioningBy отличается от groupingBy?**
A: `partitioningBy` разделяет ровно на **две** группы (true/false) по предикату — возвращает `Map<Boolean, List<T>>`. `groupingBy` разделяет на произвольное количество групп по классификатору — возвращает `Map<K, List<T>>`.

**Q: Можно ли изменять элементы source коллекции в Stream?**
A: Нельзя (non-interference). Изменение источника во время итерации может привести к `ConcurrentModificationException` или непредсказуемому поведению.

**Q: Когда использовать reduce, а когда collect?**
A: `reduce` — когда нужно получить одно значение (сумма, максимум, произведение). `collect` — когда нужна коллекция или изменяемый контейнер. Технически `collect` — специализированная форма `reduce`, оптимизированная для изменяемой агрегации (mutable reduction).

---

## 🔹 Шпаргалка

```
Stream ≠ Collection: не хранит данные, одноразовый, может быть бесконечным

Lazy: промежуточные операции не выполняются без терминальной
Short-circuit: findFirst/anyMatch/limit останавливают конвейер досрочно

map vs flatMap:
  map(fn)     — fn возвращает T → Stream<T>
  flatMap(fn) — fn возвращает Stream<T> → Stream<T> (плоско)
  Используй flatMap когда маппер возвращает коллекцию или Stream

reduce(identity, accumulator) — агрегация в одно значение
collect(Collectors.toList())  — сборка в контейнер

Collectors:
  toList/toSet/toCollection
  toMap(keyFn, valueFn, mergeFn) — передавай mergeFn если есть дубликаты!
  groupingBy(classifier, downstream)
  partitioningBy(predicate) → Map<Boolean, List<T>>
  joining(sep, prefix, suffix)
  counting / summarizingInt / averagingInt

Функциональные интерфейсы:
  Function<T,R>: andThen (A→B), compose (обратный порядок)
  Predicate<T>: and, or, negate, Predicate.not()
  Consumer<T>: andThen

Optional:
  orElse    — вычисляется ВСЕГДА (даже если есть значение)
  orElseGet — вычисляется только если empty (используй для дорогих операций)
  map/flatMap/filter — трансформируй не доставая значение
  ❌ Поля, параметры, isPresent+get

Параллельный Stream:
  ✅ Большие данные + CPU-bound + нет общего состояния
  ❌ Маленькие данные, I/O, ArrayList::add в forEach
  collect() — потокобезопасен, forEach — нет
  Использует ForkJoinPool.commonPool() — общий ресурс JVM

Ссылки на методы:
  ClassName::staticMethod
  object::instanceMethod
  ClassName::instanceMethod (1й аргумент = объект)
  ClassName::new
```
