# Immutability

> **Теги:** #interviews #java-core #immutability #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Что такое Immutable объект

**Immutable объект** — объект, состояние которого не может быть изменён после создания.

### Преимущества

1. **Thread-safety** — безопасно использовать в многопоточной среде без синхронизации
2. **Ключ в HashMap** — hashCode никогда не меняется
3. **Простота** — легче рассуждать о коде, меньше side effects
4. **Кэширование** — можно безопасно кэшировать и переиспользовать

---

## 🔹 5 правил создания Immutable класса

### 1. Все поля final

```java
public final class User {
    private final String name;
    private final int age;
}
```

### 2. Класс final (нельзя наследоваться)

```java
public final class User {  // final класс
    // ...
}
```

> [!warning] Почему класс должен быть final?
Если наследуемся и переопределим геттер, можем вернуть прямую ссылку на внутреннее состояние → нарушим иммутабельность.

### 3. Все поля тоже immutable

Если поле mutable объект — проблема:
```java
private final List<String> roles;  // mutable!
```

### 4. В конструкторе делать копии mutable объектов

```java
public User(String name, List<String> roles) {
    this.name = name;
    this.roles = new ArrayList<>(roles);  // копия!
}
```

### 5. В геттерах возвращать копии mutable объектов

```java
public List<String> getRoles() {
    return new ArrayList<>(roles);  // копия!
}
```

> [!tip] Альтернатива: не давать геттеры для mutable полей

---

## 🔹 Пример Immutable класса

```java
public final class User {
    private final String name;
    private final int age;
    private final List<String> roles;

    public User(String name, int age, List<String> roles) {
        this.name = name;
        this.age = age;
        this.roles = new ArrayList<>(roles);  // defensive copy
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    public List<String> getRoles() {
        return new ArrayList<>(roles);  // defensive copy
    }
}
```

---

## 🔹 String Immutability

**String** — immutable класс в Java.

### Почему String immutable?

1. **Security** — пароли, пути к файлам безопасны
2. **Thread-safety** — можно безопасно шарить между потоками
3. **String Pool** — кэширование и переиспользование строк
4. **HashCode caching** — hashCode вычисляется один раз и кэшируется

### Пример

```java
String s1 = "Hello";
String s2 = s1.concat(" World");  // создаётся НОВЫЙ объект
// s1 всё ещё "Hello"
```

### StringBuilder vs StringBuffer

| Класс | Mutable | Потокобезопасность | Когда использовать |
|-------|---------|-------------------|-------------------|
| **StringBuilder** | Да | Нет | Однопоточная среда (чаще всего) |
| **StringBuffer** | Да | Да | Многопоточная среда |

```java
// StringBuilder — быстрее, нет синхронизации
StringBuilder sb = new StringBuilder();
sb.append("Hello").append(" World");

// StringBuffer — синхронизированные методы
StringBuffer sb = new StringBuffer();
sb.append("Hello").append(" World");
```

---

## 🔹 Record (Java 14+)

**Record** — готовый immutable класс без boilerplate.

### Синтаксис

```java
public record User(String name, int age) {
    // автоматически генерируется:
    // - final поля
    // - constructor
    // - getters (name(), age())
    // - equals()
    // - hashCode()
    // - toString()
}
```

### Использование

```java
User user = new User("Alice", 30);
System.out.println(user.name());  // Alice
System.out.println(user.age());   // 30
```

### Custom constructor

```java
public record User(String name, int age) {
    public User {
        if (age < 0) {
            throw new IllegalArgumentException("Age must be positive");
        }
    }
}
```

### Ограничения

- Нельзя наследоваться от record
- Record не может наследовать другие классы (кроме java.lang.Record)
- Поля всегда final

> [!tip] Когда использовать Record
- DTO (Data Transfer Objects)
- Значения, которые не меняются
- Вместо Lombok @Value

---

## 🔹 Immutable объект как ключ в HashMap

### Почему immutable ключи важны

Если ключ изменится после добавления в Map:
- hashCode изменится
- При поиске вычисляется новый hashCode → другой бакет
- Элемент не найдётся

### Пример проблемы

```java
// Mutable ключ (ПЛОХО)
class BadKey {
    private int value;
    public BadKey(int value) { this.value = value; }
    public void setValue(int value) { this.value = value; }
    // equals/hashCode по value
}

Map<BadKey, String> map = new HashMap<>();
BadKey key = new BadKey(1);
map.put(key, "value");

key.setValue(2);  // изменили ключ!
map.get(key);     // null — не найдёт!
```

### Правильный подход

```java
// Immutable ключ (ХОРОШО)
record GoodKey(int value) {}

Map<GoodKey, String> map = new HashMap<>();
GoodKey key = new GoodKey(1);
map.put(key, "value");

// key нельзя изменить
map.get(key);  // "value"
```

---

## 🔹 Immutable vs Mutable

| Характеристика | Immutable | Mutable |
|----------------|-----------|---------|
| Состояние | Не меняется | Можно менять |
| Thread-safety | Да | Нужна синхронизация |
| Ключ в Map | Идеально | Проблематично |
| Память | Больше (копии) | Меньше |
| Производительность | Может быть медленнее (копии) | Быстрее |

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **5 правил:** final поля, final класс, immutable поля, копии в конструкторе, копии в геттерах
> - **String** — immutable, StringBuilder/StringBuffer — mutable
> - **Record** — готовый immutable класс (Java 14+)
> - **Ключ в HashMap** — должен быть immutable
> - **Преимущества:** thread-safety, безопасность, простота

```
5 правил immutable:
1. final поля
2. final класс
3. immutable поля
4. копии в конструкторе
5. копии в геттерах

String:
immutable, thread-safe, String Pool

Record:
автоматический immutable класс
конструктор, getters, equals, hashCode, toString

Ключ в HashMap:
immutable → hashCode не меняется
mutable → потеря элемента
```
