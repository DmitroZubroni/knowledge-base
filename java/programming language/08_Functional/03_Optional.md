> **Теги:** #java #functional #optional #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Functional_Index]]

# 03 — Optional

---

## 🔹 Проблема null — "billion dollar mistake"

`null` в Java — источник одной из самых распространённых ошибок: `NullPointerException`. Автор `null` в языках программирования Тони Хоар назвал это решение "ошибкой на миллиард долларов".

Проблема не в самом `null`, а в том что Java никак не сигнализирует: **этот метод может вернуть null**. Ты либо помнишь это, либо получаешь NPE в продакшене.

```java
// Классический сценарий — цепочка вызовов где любой может вернуть null
String city = user.getAddress().getCity().toUpperCase();
//                 ↑ может быть null  ↑ может быть null

// Защитный код — громоздкий и часто забываемый
if (user != null && user.getAddress() != null && user.getAddress().getCity() != null) {
    city = user.getAddress().getCity().toUpperCase();
} else {
    city = "Unknown";
}
```

`Optional<T>` — контейнер, который **явно сигнализирует** в сигнатуре метода: "результат может отсутствовать". Это делает контракт метода очевидным.

---

## 🔹 Создание Optional

```java
import java.util.Optional;

// of() — значение точно не null (NPE если передать null)
Optional<String> opt = Optional.of("Hello");

// ofNullable() — значение может быть null
Optional<String> opt = Optional.ofNullable(someNullableValue);
Optional<String> opt = Optional.ofNullable(null); // → Optional.empty()

// empty() — явно пустой Optional
Optional<String> opt = Optional.empty();
```

> [!warning] Когда использовать of vs ofNullable
> `Optional.of(value)` — когда ты **уверен** что value не null. Если передашь null — получишь NPE немедленно (это хорошо — быстрый fail).
> `Optional.ofNullable(value)` — когда value может быть null (например, возвращается из внешнего API или базы данных).

---

## 🔹 Проверка наличия значения

```java
Optional<String> opt = Optional.of("Hello");
Optional<String> empty = Optional.empty();

opt.isPresent();    // true
empty.isPresent();  // false

opt.isEmpty();      // false (Java 11+)
empty.isEmpty();    // true  (Java 11+)
```

---

## 🔹 Получение значения — правильно и неправильно

```java
Optional<String> opt = Optional.of("Hello");
Optional<String> empty = Optional.empty();

// ❌ get() без проверки — NPE если empty
String val = opt.get(); // "Hello"
String bad = empty.get(); // NoSuchElementException!

// ✅ orElse — значение по умолчанию (всегда вычисляется!)
String s1 = opt.orElse("default");      // "Hello"
String s2 = empty.orElse("default");    // "default"

// ✅ orElseGet — ленивое значение (вычисляется только если empty)
String s3 = empty.orElseGet(() -> computeDefault()); // computeDefault() вызван
String s4 = opt.orElseGet(() -> computeDefault());   // computeDefault() НЕ вызван

// ✅ orElseThrow — бросить исключение если empty
String s5 = opt.orElseThrow();  // "Hello" или NoSuchElementException (Java 10+)
String s6 = empty.orElseThrow(() -> new UserNotFoundException("User not found"));
```

> [!tip] orElse vs orElseGet
> Это важное различие. `orElse(value)` **всегда** вычисляет value, даже если Optional не пуст. Если value — дорогой вызов (запрос в БД, создание объекта), это может быть проблемой:
> ```java
> // ❌ DB запрос выполнится даже если user найден!
> User user = findUser(id).orElse(createDefaultUser()); // createDefaultUser() — запрос
>
> // ✅ DB запрос только если user не найден
> User user = findUser(id).orElseGet(() -> createDefaultUser());
> ```

---

## 🔹 Трансформация — map, flatMap, filter

Вместо "достань значение и проверь null" — применяй операции прямо к Optional:

### map — трансформировать если есть значение

```java
Optional<String> opt = Optional.of("  Hello World  ");

// Цепочка трансформаций — каждый шаг безопасен
Optional<String> result = opt
    .map(String::trim)          // "Hello World"
    .map(String::toUpperCase)   // "HELLO WORLD"
    .filter(s -> s.length() > 5); // Optional["HELLO WORLD"] — длина > 5

// Если в какой-то момент null — весь pipeline возвращает empty
Optional<String> name = findUser(id)           // Optional<User>
    .map(User::getAddress)                      // Optional<Address> (или empty если null)
    .map(Address::getCity)                      // Optional<String>  (или empty если null)
    .map(String::toUpperCase);                  // Optional<String>  (или empty)

// Без Optional — громоздко
String name2 = null;
User user = findUser(id);
if (user != null && user.getAddress() != null && user.getAddress().getCity() != null) {
    name2 = user.getAddress().getCity().toUpperCase();
}
```

### flatMap — для вложенных Optional

Если метод сам возвращает `Optional`, используй `flatMap` чтобы не получить `Optional<Optional<T>>`:

```java
class User {
    Optional<Address> getAddress() { ... } // метод уже возвращает Optional
}

// ❌ map — получаем Optional<Optional<Address>>
Optional<Optional<Address>> bad = findUser(id).map(User::getAddress);

// ✅ flatMap — получаем Optional<Address>
Optional<Address> address = findUser(id).flatMap(User::getAddress);

// Цепочка с flatMap
Optional<String> city = findUser(id)
    .flatMap(User::getAddress)   // User::getAddress возвращает Optional<Address>
    .map(Address::getCity);      // getCity возвращает String (не Optional)
```

### filter — условие на значение

```java
Optional<Integer> age = Optional.of(25);

Optional<Integer> adult = age.filter(a -> a >= 18); // Optional[25] — условие выполнено
Optional<Integer> teen  = age.filter(a -> a < 18);  // Optional.empty — условие не выполнено

// Практический пример
Optional<User> activeAdmin = findUser(id)
    .filter(u -> u.isActive())
    .filter(u -> u.hasRole("ADMIN"));
```

---

## 🔹 Действия — ifPresent, ifPresentOrElse

```java
Optional<String> opt = Optional.of("Hello");

// ifPresent — выполнить действие если есть значение
opt.ifPresent(s -> System.out.println("Value: " + s)); // "Value: Hello"

// ifPresentOrElse — Java 9+, выполнить одно из двух действий
opt.ifPresentOrElse(
    s -> System.out.println("Found: " + s),
    () -> System.out.println("Not found")
);

// or() — Java 9+, вернуть другой Optional если этот пустой
Optional<String> result = empty.or(() -> Optional.of("fallback")); // Optional["fallback"]
```

---

## 🔹 Optional в Stream

```java
List<Optional<String>> opts = List.of(
    Optional.of("Alice"),
    Optional.empty(),
    Optional.of("Bob"),
    Optional.empty(),
    Optional.of("Charlie")
);

// Java 9+ — stream() на Optional возвращает Stream с 0 или 1 элементом
List<String> names = opts.stream()
    .flatMap(Optional::stream) // пустые Optional просто не добавляются
    .collect(Collectors.toList());
// [Alice, Bob, Charlie]

// Классический паттерн: список возможных значений, взять первое непустое
Optional<String> first = Stream.of(
    tryFirstSource(),
    trySecondSource(),
    tryThirdSource()
).filter(Optional::isPresent)
 .map(Optional::get)
 .findFirst();
```

---

## 🔹 Анти-паттерны — как НЕ нужно использовать Optional

### Анти-паттерн 1: проверка isPresent + get вместо orElse/map

```java
// ❌ Это хуже чем просто проверить null — смысл Optional теряется
if (opt.isPresent()) {
    return opt.get();
} else {
    return "default";
}

// ✅ Правильно
return opt.orElse("default");
```

### Анти-паттерн 2: Optional как поле класса

```java
// ❌ Не нужно — Optional не сериализуется, занимает лишнюю память
class User {
    private Optional<String> middleName; // плохо
}

// ✅ Используй null для полей, Optional только в возвращаемых значениях
class User {
    private String middleName; // может быть null внутри

    public Optional<String> getMiddleName() { // Optional снаружи
        return Optional.ofNullable(middleName);
    }
}
```

### Анти-паттерн 3: Optional как параметр метода

```java
// ❌ Усложняет вызов метода, caller вынужден создавать Optional
public void process(Optional<String> name) { ... }
process(Optional.of("Alice"));    // неудобно
process(Optional.empty());        // некрасиво

// ✅ Лучше перегрузить метод или использовать @Nullable
public void process(String name) { ... }   // name может быть null
public void process() { process(null); }   // перегрузка без параметра
```

### Анти-паттерн 4: Optional в коллекциях

```java
// ❌ List<Optional<String>> — зачем? пустые просто не добавляй
List<Optional<String>> bad = new ArrayList<>();

// ✅ Просто не добавляй null значения или используй пустые строки
List<String> good = new ArrayList<>();
```

### Анти-паттерн 5: orElse с null

```java
// ❌ Полностью теряет смысл Optional
String val = opt.orElse(null); // это просто opt.isPresent() ? opt.get() : null

// ✅ Если нужен null — не используй Optional в этом месте вовсе
```

---

## 🔹 Реальный пример — сервисный метод

```java
// Репозиторий возвращает Optional
public interface UserRepository {
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
}

// Сервис — цепочка операций без явных null-проверок
public class UserService {

    public String getUserCity(Long userId) {
        return userRepository.findById(userId)         // Optional<User>
            .filter(User::isActive)                    // только активные
            .map(User::getAddress)                     // Optional<Address> (null → empty)
            .map(Address::getCity)                     // Optional<String>
            .map(String::toUpperCase)                  // Optional<String>
            .orElse("UNKNOWN");                        // если на любом шаге empty
    }

    public User getOrCreateUser(String email) {
        return userRepository.findByEmail(email)
            .orElseGet(() -> createNewUser(email));    // создаём только если не найден
    }

    public User getExistingUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }
}
```

---

## 🔹 Итог

```
Optional<T> = контейнер, который явно говорит "значение может отсутствовать"
Цель: устранить NPE, сделать контракт метода явным

Создание:
  Optional.of(value)          — не null (NPE если null)
  Optional.ofNullable(value)  — может быть null
  Optional.empty()            — явно пустой

Получение значения:
  get()                       — ❌ только с isPresent(), риск NoSuchElementException
  orElse(default)             — значение по умолчанию (ВСЕГДА вычисляется)
  orElseGet(() -> compute())  — ленивое (вычисляется только если empty)
  orElseThrow(() -> ex)       — бросить исключение если empty

Трансформация (не достаём значение — применяем операции):
  map(fn)         — трансформировать если есть (fn возвращает T)
  flatMap(fn)     — трансформировать если есть (fn возвращает Optional<T>)
  filter(pred)    — оставить только если условие выполнено

Действия:
  ifPresent(fn)              — выполнить если есть
  ifPresentOrElse(fn, else)  — одно из двух (Java 9+)
  or(() -> Optional<T>)      — другой Optional если пустой (Java 9+)

В стримах:
  Optional::stream  — flatMap(Optional::stream) убирает пустые из потока

Правила использования:
  ✅ Возвращаемые значения методов где результат может отсутствовать
  ❌ Поля классов (не сериализуется)
  ❌ Параметры методов (усложняет API)
  ❌ Коллекции (просто не добавляй null)
  ❌ isPresent() + get() — используй map/orElse
```
