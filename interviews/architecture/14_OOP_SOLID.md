# OOP & SOLID

> **Теги:** #interviews #architecture #oop #solid #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 4 столпа ООП

### 1. Инкапсуляция (Encapsulation)

Скрытие внутренней реализации и предоставление интерфейса.

```java
public class BankAccount {
    private double balance;  // скрыто

    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    public double getBalance() {
        return balance;
    }
}
```

**Преимущества:**
- Защита данных
- Изменение реализации без влияния на внешний код
- Контроль доступа

### 2. Наследование (Inheritance)

Создание новых классов на основе существующих.

```java
public class Animal {
    public void eat() {
        System.out.println("Eating");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println("Barking");
    }
}
```

**Преимущества:**
- Переиспользование кода
- Иерархическая организация
- Полиморфизм

### 3. Полиморфизм (Polymorphism)

Один интерфейс — множество реализаций.

```java
public interface Shape {
    double area();
}

public class Circle implements Shape {
    private double radius;
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle implements Shape {
    private double width, height;
    public double area() { return width * height; }
}

// Использование
List<Shape> shapes = List.of(new Circle(), new Rectangle());
shapes.forEach(shape -> System.out.println(shape.area()));
```

### 4. Абстракция (Abstraction)

Скрывание деталей реализации, показ только существенных характеристик.

```java
public abstract class Vehicle {
    protected String brand;

    public abstract void start();  // абстрактный метод

    public void honk() {
        System.out.println("Honk!");
    }
}
```

---

## 🔹 SOLID принципы

### S - Single Responsibility Principle (SRP)

**Класс должен иметь только одну причину для изменения.**

#### Классическая трактовка

Класс делает только одну вещь.

```java
// ПЛОХО
class User {
    void saveToDB() { }
    void sendEmail() { }
    void validate() { }
}

// ХОРОШО
class UserRepository { void save() { } }
class EmailService { void send() { } }
class UserValidator { void validate() { } }
```

#### Стейкхолдер-трактовка (Actor-based)

**Класс должен отвечать перед одним стейкхолдером (актором).**

```java
// Стейкхолдеры:
// - Менеджер (хочет отчёты)
// - HR (хочет данные о сотрудниках)
// - Разработчик (хочет технические детали)

// ПЛОХО — отвечает перед несколькими стейкхолдерами
class Employee {
    void calculateSalary() { }  // HR
    void generateReport() { }  // Менеджер
    void updateDatabase() { }  // Разработчик
}

// ХОРОШО — каждый класс отвечает перед одним стейкхолдером
class Employee {
    // данные сотрудника (HR)
}

class SalaryCalculator {
    // расчёт зарплаты (HR)
}

class ReportGenerator {
    // генерация отчётов (Менеджер)
}
```

> [!tip] SRP mnemonic
- Классическая: одна ответственность
- Стейкхолдер: один актор (стейкхолдер)

### O - Open/Closed Principle (OCP)

**Программные сущности должны быть открыты для расширения, но закрыты для модификации.**

```java
// ПЛОХО — нужно модифицировать при добавлении новой фигуры
class AreaCalculator {
    double calculate(Object shape) {
        if (shape instanceof Circle) { }
        else if (shape instanceof Rectangle) { }
        // добавляем новый if для каждой новой фигуры
    }
}

// ХОРОШО — открыт для расширения, закрыт для модификации
interface Shape {
    double area();
}

class Circle implements Shape { }
class Rectangle implements Shape { }
class Triangle implements Shape { }  // добавляем новую фигуру без изменения AreaCalculator
```

### L - Liskov Substitution Principle (LSP)

**Объекты подкласса должны заменять объекты базового класса без нарушения работы программы.**

```java
// ПЛОХО — нарушает LSP
class Bird {
    void fly() { }
}

class Penguin extends Bird {
    @Override
    void fly() {
        throw new UnsupportedOperationException("Penguins can't fly");
    }
}

// ХОРОШО
class Bird { }
class FlyingBird extends Bird {
    void fly() { }
}
class Penguin extends Bird { }  // не наследуется от FlyingBird
```

### I - Interface Segregation Principle (ISP)

**Клиенты не должны зависеть от интерфейсов, которые они не используют.**

```java
// ПЛОХО — один большой интерфейс
interface Worker {
    void work();
    void eat();
}

class Robot implements Worker {
    void work() { }
    void eat() {
        throw new UnsupportedOperationException("Robots don't eat");
    }
}

// ХОРОШО — разделение интерфейсов
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

class Human implements Workable, Eatable { }
class Robot implements Workable { }
```

### D - Dependency Inversion Principle (DIP)

**Модули верхних уровней не должны зависеть от модулей нижних уровней. Оба должны зависеть от абстракций.**

```java
// ПЛОХО — зависит от конкретной реализации
class LightSwitch {
    private LightBulb bulb = new LightBulb();  // конкретная реализация
    void on() { bulb.on(); }
}

// ХОРОШО — зависит от абстракции
interface Switchable {
    void on();
    void off();
}

class LightBulb implements Switchable { }
class Fan implements Switchable { }

class LightSwitch {
    private Switchable device;  // абстракция
    LightSwitch(Switchable device) {
        this.device = device;
    }
    void on() { device.on(); }
}
```

---

## 🔹 Полиморфизм

### Статический полиморфизм (Compile-time)

Перегрузка методов (overloading).

```java
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
}
```

### Динамический полиморфизм (Runtime)

Переопределение методов (overriding).

```java
class Animal {
    void makeSound() { System.out.println("Animal sound"); }
}

class Dog extends Animal {
    @Override
    void makeSound() { System.out.println("Bark"); }
}

Animal animal = new Dog();
animal.makeSound();  // Bark (динамический полиморфизм)
```

---

## 🔹 Контракт (Contract)

**Контракт** — соглашение между классами/интерфейсами о поведении.

### Пример контракта

```java
// Контракт интерфейса
public interface List<E> {
    // Контракт: add() возвращает true если список изменился
    boolean add(E e);
    
    // Контракт: get() возвращает элемент по индексу
    E get(int index);
    
    // Контракт: выбрасывает IndexOutOfBoundsException если индекс вне диапазона
}
```

### Нарушение контракта

```java
// Нарушение контракта
class MyList implements List<String> {
    public boolean add(String e) {
        // возвращает false даже если добавился
        return false;  // нарушение контракта!
    }
}
```

### LSP как контракт

LSP — это контракт подкласса: подкласс должен соблюдать контракт базового класса.

```java
// Контракт Rectangle: width и height независимы
class Rectangle {
    void setWidth(double w) { }
    void setHeight(double h) { }
    double area() { return width * height; }
}

// Нарушение контракта (LSP violation)
class Square extends Rectangle {
    void setWidth(double w) {
        width = w;
        height = w;  // нарушает контракт Rectangle!
    }
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **4 столпа ООП:** Инкапсуляция, Наследование, Полиморфизм, Абстракция
> - **SRP:** одна ответственность / один стейкхолдер
> - **OCP:** открыт для расширения, закрыт для модификации
> - **LSP:** подкласс заменяет базовый без нарушения работы
> - **ISP:** разделение интерфейсов
> - **DIP:** зависимость от абстракций, не от конкретных реализаций
> - **Полиморфизм:** статический (overloading), динамический (overriding)
> - **Контракт:** соглашение о поведении, LSP — контракт подкласса

```
4 столпа ООП:
Инкапсуляция → скрытие реализации
Наследование → переиспользование кода
Полиморфизм → один интерфейс — множество реализаций
Абстракция → скрытие деталей

SOLID:
S - SRP → одна ответственность / один стейкхолдер
O - OCP → открыт для расширения, закрыт для модификации
L - LSP → подкласс заменяет базовый
I - ISP → разделение интерфейсов
D - DIP → зависимость от абстракций

Полиморфизм:
Статический → overloading (compile-time)
Динамический → overriding (runtime)

Контракт:
Соглашение о поведении
LSP → подкласс соблюдает контракт базового
```
