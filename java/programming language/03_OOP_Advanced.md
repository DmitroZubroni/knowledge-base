# 03 — ООП: Наследование, полиморфизм, интерфейсы

> **Теги:** #java #programming #oop #inheritance #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]]

---

## 🔹 Наследование (extends)

> [!note] Определение
> Наследование — механизм, при котором класс-потомок получает поля и методы класса-родителя. В Java только **одиночное наследование** (один родитель).

```
Animal (родитель/суперкласс)
    ├── Dog (потомок/подкласс)
    ├── Cat (потомок/подкласс)
    └── Bird (потомок/подкласс)
```

```java
// Родительский класс
public class Animal {
    String name;
    int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// Подкласс наследует ВСЁ из Animal и добавляет своё
public class Dog extends Animal {
    String breed;

    public Dog(String name, int age, String breed) {
        super(name, age);  // вызов конструктора родителя — ОБЯЗАТЕЛЬНО первой строкой
        this.breed = breed;
    }

    public void bark() {
        System.out.println(name + " says: Woof!");  // name унаследовано
    }
}

// Использование
Dog dog = new Dog("Rex", 3, "Husky");
dog.eat();   // метод родителя
dog.sleep(); // метод родителя
dog.bark();  // собственный метод
```

### super — обращение к родителю
```java
public class Cat extends Animal {
    boolean isIndoor;

    public Cat(String name, int age, boolean isIndoor) {
        super(name, age);          // конструктор родителя
        this.isIndoor = isIndoor;
    }

    @Override
    public void eat() {
        super.eat();               // вызов метода родителя
        System.out.println("...and wants more fish");
    }
}
```

> [!warning] Подводные камни
> - `super(...)` — всегда **первая строка** в конструкторе потомка
> - Если в родителе нет конструктора без параметров — подкласс **обязан** явно вызвать `super(...)`
> - `private` поля и методы родителя **не наследуются** (недоступны в подклассе)

---

## 🔹 Переопределение методов (@Override)

> [!note] Определение
> Переопределение (override) — изменение реализации унаследованного метода в подклассе. Сигнатура должна совпадать точно.

```java
public class Animal {
    public String speak() {
        return "...";
    }
}

public class Dog extends Animal {
    @Override
    public String speak() {       // та же сигнатура
        return "Woof!";
    }
}

public class Cat extends Animal {
    @Override
    public String speak() {
        return "Meow!";
    }
}
```

> [!tip] Рекомендация
> Всегда ставь `@Override` при переопределении — компилятор проверит, что такой метод существует в родителе. Без аннотации ошибку не поймаешь.

**Overriding vs Overloading:**

| | Overriding (переопределение) | Overloading (перегрузка) |
|-|------------------------------|--------------------------|
| Где | В подклассе | В том же классе |
| Сигнатура | **Совпадает** с родителем | **Отличается** (параметры) |
| `@Override` | Да | Нет |
| Тип возврата | Тот же (или ковариантный) | Может быть разным |

---

## 🔹 Полиморфизм

> [!note] Определение
> Полиморфизм — возможность обращаться к объектам разных классов через общий тип (родительский класс или интерфейс). Метод вызывается у **фактического** типа объекта в runtime.

```java
Animal a1 = new Dog("Rex", 3, "Husky");   // тип переменной Animal, объект Dog
Animal a2 = new Cat("Whiskers", 2, true); // тип переменной Animal, объект Cat
Animal a3 = new Animal("Generic", 1);

Animal[] animals = {a1, a2, a3};

for (Animal animal : animals) {
    System.out.println(animal.speak());
    // Rex вызовет Dog.speak() → "Woof!"
    // Whiskers вызовет Cat.speak() → "Meow!"
    // Generic вызовет Animal.speak() → "..."
}
```

```
Переменная типа Animal
        ↓
        ┌──────────────────────────────┐
        │ Dog   │ Cat   │ Bird  │ ...  │  ← разные объекты в Heap
        └──────────────────────────────┘
При вызове speak() — JVM смотрит на РЕАЛЬНЫЙ тип объекта (динамическая диспетчеризация)
```

---

## 🔹 Абстрактные классы

> [!note] Определение
> Абстрактный класс — класс, который **нельзя создать напрямую**. Может содержать как обычные методы, так и абстрактные (без реализации). Подклассы обязаны реализовать все абстрактные методы.

```java
public abstract class Shape {
    String color;

    public Shape(String color) {
        this.color = color;
    }

    // Абстрактный метод — нет тела, подкласс ОБЯЗАН реализовать
    public abstract double area();
    public abstract double perimeter();

    // Обычный метод с реализацией
    public void printInfo() {
        System.out.println(color + " shape: area=" + area());
    }
}

public class Circle extends Shape {
    double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
}

public class Rectangle extends Shape {
    double width, height;

    public  (String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() { return width * height; }

    @Override
    public double perimeter() { return 2 * (width + height); }
}

// Shape s = new Shape("red");  // ❌ нельзя создать абстрактный класс
Shape c = new Circle("blue", 5.0);  // ✅
c.printInfo();  // "blue shape: area=78.53..."
```

---

## 🔹 Интерфейсы

> [!note] Определение
> Интерфейс — контракт поведения ("что умеет делать"). До Java 8 — только абстрактные методы. С Java 8 — можно иметь `default` и `static` методы с реализацией.

```java
// Объявление интерфейса
public interface Flyable {
    // константы — public static final (по умолчанию)
    int MAX_ALTITUDE = 10000;

    // абстрактный метод — public abstract (по умолчанию)
    void fly();

    // default метод (Java 8+) — реализация по умолчанию
    default void land() {
        System.out.println("Landing...");
    }

    // static метод (Java 8+)
    static String info() {
        return "Flyable interface";
    }
}

public interface Swimmable {
    void swim();
}

// Класс может реализовать НЕСКОЛЬКО интерфейсов
public class Duck extends Animal implements Flyable, Swimmable {
    public Duck(String name) {
        super(name, 1);
    }

    @Override
    public void fly() {
        System.out.println(name + " is flying");
    }

    @Override
    public void swim() {
        System.out.println(name + " is swimming");
    }
}

Duck duck = new Duck("Donald");
duck.fly();    // реализация из Duck
duck.swim();   // реализация из Duck
duck.land();   // default реализация из Flyable
Flyable.info(); // статический метод интерфейса
```

### Абстрактный класс vs Интерфейс

| | Абстрактный класс | Интерфейс |
|-|-------------------|-----------|
| Создать экземпляр | ❌ | ❌ |
| Наследование | Один (extends) | Несколько (implements) |
| Поля | Любые | Только `public static final` |
| Методы с реализацией | ✅ | `default` и `static` (Java 8+) |
| Конструктор | ✅ | ❌ |
| Когда использовать | Общее состояние + базовая реализация | Контракт поведения |

> [!tip] Рекомендация
> Используй **интерфейс**, если хочешь описать "что умеет делать" объект.
> Используй **абстрактный класс**, если есть общее состояние и базовая логика.

---

## 🔹 instanceof и приведение типов

> [!note] Определение
> `instanceof` — проверяет, является ли объект экземпляром указанного класса или его подкласса/интерфейса.

```java
Animal animal = new Dog("Rex", 3, "Husky");

// Проверка типа
System.out.println(animal instanceof Animal);  // true
System.out.println(animal instanceof Dog);     // true
System.out.println(animal instanceof Cat);     // false

// Приведение типа (downcast) — от родителя к потомку
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;   // явное приведение
    dog.bark();               // теперь доступны методы Dog
}

// Java 16+ pattern matching для instanceof
if (animal instanceof Dog dog) {  // проверка + приведение в одной строке
    dog.bark();
}
```

> [!danger] Опасность
> Приведение типа без проверки `instanceof` вызывает `ClassCastException` в runtime:
> ```java
> Animal a = new Cat("Tom", 2, true);
> Dog d = (Dog) a;  // ❌ ClassCastException!
> ```

---

## 🔹 Вложенные классы

> [!note] Определение
> Класс, объявленный внутри другого класса. Используется для группировки логически связанных классов.

```java
public class Outer {
    private int value = 10;

    // Нестатический вложенный класс (inner class)
    public class Inner {
        public void show() {
            System.out.println("Outer value: " + value);  // доступ к private внешнего класса
        }
    }

    // Статический вложенный класс
    public static class StaticNested {
        public void show() {
            System.out.println("Static nested");
            // value недоступен — нет ссылки на Outer instance
        }
    }
}

// Использование
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();  // нужен экземпляр внешнего класса
inner.show();

Outer.StaticNested nested = new Outer.StaticNested();  // без экземпляра
nested.show();
```

---

## 🔹 Анонимные классы

> [!note] Определение
> Анонимный класс — одноразовый класс без имени, объявляемый и создаваемый одновременно. Нужен для единственного экземпляра с переопределёнными методами.

```java
// Без анонимного класса — нужно создавать отдельный класс
public class DogSound implements Flyable {
    @Override
    public void fly() { System.out.println("Dog flies somehow"); }
}

// С анонимным классом — прямо на месте
Flyable weirdDog = new Flyable() {
    @Override
    public void fly() {
        System.out.println("Dog flies somehow!");
    }
};
weirdDog.fly();

// Анонимный класс от абстрактного класса
Shape unknownShape = new Shape("purple") {
    @Override
    public double area() { return 42.0; }

    @Override
    public double perimeter() { return 24.0; }
};
unknownShape.printInfo();
```

> [!tip] Рекомендация
> В современном Java (8+) анонимные классы с одним методом заменяются **лямбда-выражениями**. Смотри [[08_Functional]].

---

## 🔹 final класс и final метод

> [!note] Определение
> `final class` — нельзя унаследовать. `final method` — нельзя переопределить в подклассе.

```java
// final класс — нельзя наследовать
public final class ImmutablePoint {
    private final int x;
    private final int y;

    public ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }
}

// class ExtendedPoint extends ImmutablePoint { }  // ❌ ошибка компиляции

// final метод — нельзя переопределить
public class Base {
    public final void mustNotOverride() {
        System.out.println("Это нельзя переопределить");
    }
}

public class Child extends Base {
    // @Override
    // public void mustNotOverride() { }  // ❌ ошибка компиляции
}
```

> [!note] Примеры final классов в Java
> `String`, `Integer`, `Double`, `Boolean` — все они `final`. Поэтому их нельзя наследовать.
