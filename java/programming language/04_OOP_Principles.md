# 04 — Пакеты, static, final, enum

> **Теги:** #java #programming #static #enum #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_JAVA]]

---

## 🔹 Пакеты (packages)

> [!note] Определение
> Пакет — пространство имён для организации классов. Соответствует структуре папок на диске. Предотвращает конфликты имён классов.

```
src/
└── com/
    └── myapp/
        ├── model/
        │   ├── User.java        → пакет com.myapp.model
        │   └── Product.java
        ├── service/
        │   └── UserService.java → пакет com.myapp.service
        └── Main.java            → пакет com.myapp
```

```java
// Объявление пакета — первая строка файла
package com.myapp.model;

public class User {
    private String name;
    // ...
}
```

### Import
```java
package com.myapp.service;

// Импорт конкретного класса
import com.myapp.model.User;

// Импорт всего пакета (*)
import java.util.*;

// Статический импорт — для static полей и методов
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;

public class UserService {
    public void processUser(User user) {  // User доступен без полного имени
        double area = PI * 5 * 5;         // PI — без Math.
        double sq = sqrt(25);             // sqrt — без Math.
    }
}
```

> [!tip] Рекомендации
> - Не используй `import java.util.*` в продакшн-коде — неявно какие классы используются
> - Классы в **одном пакете** не нужно импортировать
> - `java.lang.*` (String, System, Math, Object...) импортируется **автоматически**

> [!warning] Подводные камни
> Имена пакетов — строчными буквами. Конфликт имён: `java.util.Date` vs `java.sql.Date` — нужно использовать полное имя или один из них импортировать явно.

---

## 🔹 Модификатор static

> [!note] Определение
> `static` — принадлежит **классу**, а не конкретному объекту. Одно значение/метод на все объекты класса.

### Статическое поле
```java
public class Counter {
    private static int count = 0;  // одно на весь класс
    private int id;

    public Counter() {
        count++;           // увеличивается при каждом new Counter()
        this.id = count;
    }

    public static int getCount() {
        return count;
    }

    public int getId() {
        return id;
    }
}

Counter c1 = new Counter();  // count = 1
Counter c2 = new Counter();  // count = 2
Counter c3 = new Counter();  // count = 3

System.out.println(Counter.getCount());  // 3 — через имя класса
System.out.println(c1.getId());  // 1
System.out.println(c3.getId());  // 3
```

### Статический метод
```java
public class MathUtils {
    public static final double PI = 3.14159265;

    public static int max(int a, int b) {
        return a > b ? a : b;
    }

    public static double circleArea(double r) {
        return PI * r * r;
    }
}

// Вызов без создания объекта
int m = MathUtils.max(3, 7);            // 7
double area = MathUtils.circleArea(5);  // 78.53...
```

> [!warning] Подводные камни
> Внутри `static` метода **нельзя** обращаться к нестатическим полям и методам — нет ссылки `this`:
> ```java
> public class Broken {
>     int x = 10;
>     public static void broken() {
>         System.out.println(x);  // ❌ нельзя: x принадлежит объекту, а не классу
>     }
> }
> ```

---

## 🔹 Статический блок инициализации

> [!note] Определение
> Статический блок `static { }` — выполняется **один раз** при загрузке класса в JVM, до создания любых объектов. Используется для сложной инициализации статических полей.

```java
public class Config {
    public static final String DB_URL;
    public static final int MAX_CONNECTIONS;

    static {
        // выполняется при загрузке класса
        System.out.println("Config class loaded!");
        DB_URL = System.getenv("DB_URL") != null
            ? System.getenv("DB_URL")
            : "jdbc:postgresql://localhost:5432/mydb";
        MAX_CONNECTIONS = 10;
    }
}

// При первом обращении к классу — блок выполнится
System.out.println(Config.DB_URL);  // "Config class loaded!" затем URL
```

### Порядок инициализации класса
```
1. static поля (в порядке объявления)
2. static { } блоки (в порядке объявления)
3. при new — нестатические поля
4. при new — instance { } блоки
5. при new — конструктор
```

---

## 🔹 Модификатор final

### final переменная
```java
final int MAX = 100;
// MAX = 200;  // ❌ нельзя переприсвоить

final List<String> list = new ArrayList<>();
list.add("item");    // ✅ можно изменить СОДЕРЖИМОЕ объекта
// list = new ArrayList<>();  // ❌ нельзя переприсвоить ССЫЛКУ
```

### final поле класса
```java
public class Circle {
    private final double radius;  // должно быть инициализировано в конструкторе

    public Circle(double radius) {
        this.radius = radius;  // инициализация в конструкторе — OK
    }

    // radius = 5;  // нельзя менять после инициализации
}
```

### final метод
```java
public class Template {
    // Шаблонный метод — нельзя переопределить
    public final void process() {
        step1();
        step2();
        step3();
    }

    protected void step1() { System.out.println("Default step1"); }
    protected void step2() { System.out.println("Default step2"); }
    protected void step3() { System.out.println("Default step3"); }
}
```

### final класс
```java
public final class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() { }
    public static Singleton getInstance() { return INSTANCE; }
}
// нельзя наследовать
```

| Применение final | Смысл |
|-----------------|-------|
| `final` переменная | нельзя переприсвоить |
| `final` поле | нельзя менять после конструктора |
| `final` параметр метода | нельзя изменить параметр внутри метода |
| `final` метод | нельзя переопределить в подклассе |
| `final` класс | нельзя наследовать |

---

## 🔹 Enum (перечисления)

> [!note] Определение
> `enum` — тип с фиксированным набором именованных констант. Безопаснее, чем `int` константы. Каждая константа — объект типа `enum`.

### Базовый enum
```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

Day today = Day.WEDNESDAY;
System.out.println(today);           // "WEDNESDAY"
System.out.println(today.name());    // "WEDNESDAY"
System.out.println(today.ordinal()); // 2 (индекс с 0)

// Сравнение
if (today == Day.WEDNESDAY) {        // == работает для enum!
    System.out.println("Midweek");
}

// switch с enum
switch (today) {
    case MONDAY:
    case TUESDAY:
        System.out.println("Early week");
        break;
    case FRIDAY:
        System.out.println("TGIF!");
        break;
    default:
        System.out.println("Regular day");
}

// Все значения
for (Day d : Day.values()) {
    System.out.println(d.ordinal() + ": " + d.name());
}

// Из строки
Day friday = Day.valueOf("FRIDAY");
```

### Enum с полями и методами
```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6);

    private final double mass;    // масса (кг)
    private final double radius;  // радиус (м)

    // конструктор enum всегда private (нельзя создать new Planet())
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    static final double G = 6.67300E-11;

    public double surfaceGravity() {
        return G * mass / (radius * radius);
    }

    public double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}

double earthWeight = 75.0;
double mass = earthWeight / Planet.EARTH.surfaceGravity();
for (Planet p : Planet.values()) {
    System.out.printf("Вес на %s: %.2f%n", p, p.surfaceWeight(mass));
}
```

> [!warning] Подводные камни
> - `enum` конструктор всегда `private` — нельзя `new Planet(...)`
> - `ordinal()` зависит от порядка объявления — не используй его для сохранения в БД, используй `name()` или кастомное поле
> - Не наследуй от `enum` (он уже наследует от `java.lang.Enum`)

---

## 🔹 Record (Java 16+)

> [!note] Определение
> `record` — неизменяемый класс-данных, автоматически генерирующий конструктор, геттеры, `equals()`, `hashCode()`, `toString()`.

```java
// Вместо громоздкого класса:
public class PointOld {
    private final int x;
    private final int y;
    public PointOld(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    // + equals, hashCode, toString...
}

// Record — компактная запись:
public record Point(int x, int y) { }

// Использование
Point p = new Point(3, 7);
System.out.println(p.x());      // 3
System.out.println(p.y());      // 7
System.out.println(p);          // "Point[x=3, y=7]"
p.equals(new Point(3, 7));      // true

// Record с кастомной валидацией
public record Range(int min, int max) {
    // Компактный конструктор (без параметров — они уже объявлены)
    public Range {
        if (min > max) throw new IllegalArgumentException("min > max");
    }
}
```

> [!tip] Рекомендация
> Используй `record` для DTO (Data Transfer Objects), Value Objects, возвращаемых значений из методов — там, где нужна структура данных без логики.
