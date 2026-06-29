# 02 — ООП: Классы и объекты

> **Теги:** #java #programming #oop #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]]

---

## 🔹 Класс и объект

> [!note] Определение
> **Класс** — чертёж/шаблон, описывающий структуру и поведение.
> **Объект** — конкретный экземпляр класса, размещённый в Heap.

```
Класс (шаблон в коде)         Объекты (в памяти Heap)
──────────────────────         ──────────────────────────
class Dog {                    Dog d1 = new Dog("Rex", 3);
    String name;               Dog d2 = new Dog("Buddy", 5);
    int age;                   Dog d3 = new Dog("Max", 1);
    void bark() { ... }
}
```

```java
// Объявление класса
public class Dog {
    // Поля (fields) — состояние объекта
    String name;
    int age;
    String breed;

    // Методы — поведение объекта
    void bark() {
        System.out.println(name + " says: Woof!");
    }

    void info() {
        System.out.println(name + ", " + age + " years, " + breed);
    }
}

// Создание объекта и использование
public class Main {
    public static void main(String[] args) {
        Dog dog = new Dog();   // new → выделение памяти в Heap + конструктор
        dog.name = "Rex";
        dog.age = 3;
        dog.breed = "Husky";

        dog.bark();   // "Rex says: Woof!"
        dog.info();   // "Rex, 3 years, Husky"
    }
}
```

---

## 🔹 Конструкторы

> [!note] Определение
> Конструктор — специальный метод для инициализации объекта. Вызывается при `new`. Не имеет типа возврата, имя совпадает с именем класса.

### Конструктор по умолчанию (default)
```java
public class Cat {
    String name;
    int age;
    // Java сам создаёт: public Cat() { } — если нет ни одного конструктора
}

Cat c = new Cat();  // name = null, age = 0 (значения по умолчанию)
```

### Параметризованный конструктор
```java
public class Cat {
    String name;
    int age;

    // Параметризованный конструктор
    public Cat(String name, int age) {
        this.name = name;  // this.name — поле класса, name — параметр
        this.age = age;
    }
}

Cat c = new Cat("Whiskers", 2);
System.out.println(c.name);  // "Whiskers"
```

### Перегрузка конструкторов
```java
public class Person {
    String name;
    int age;
    String email;

    // Конструктор 1: только имя
    public Person(String name) {
        this.name = name;
        this.age = 0;
        this.email = "";
    }

    // Конструктор 2: имя + возраст
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
        this.email = "";
    }

    // Конструктор 3: все поля
    public Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }
}
```

### Вызов одного конструктора из другого: this(...)
```java
public class Person {
    String name;
    int age;
    String email;

    public Person(String name) {
        this(name, 0, "");  // вызов конструктора с 3 параметрами
    }

    public Person(String name, int age) {
        this(name, age, ""); // вызов конструктора с 3 параметрами
    }

    public Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }
}
```

> [!warning] Подводные камни
> - `this(...)` должен быть **первой строкой** в конструкторе
> - Если определён хотя бы один конструктор — Java **не создаёт** default конструктор
> - Конструктор **не возвращает** значение (даже void не пишется)

---

## 🔹 Ключевое слово this

> [!note] Определение
> `this` — ссылка на текущий объект (тот, у которого вызывается метод или конструктор).

```java
public class BankAccount {
    String owner;
    double balance;

    public BankAccount(String owner, double balance) {
        this.owner = owner;    // this.owner — поле, owner — параметр
        this.balance = balance;
    }

    public void transfer(BankAccount other, double amount) {
        this.balance -= amount;   // у текущего объекта
        other.balance += amount;  // у другого объекта
    }

    public BankAccount getThis() {
        return this;  // вернуть ссылку на себя (метод chaining)
    }
}
```

---

## 🔹 Модификаторы доступа

> [!note] Определение
> Модификаторы доступа определяют, откуда можно обращаться к полям и методам.

| Модификатор | Свой класс | Свой пакет | Подкласс | Все |
|-------------|:---:|:---:|:---:|:---:|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(package-private)* | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

```java
public class Employee {
    public String name;         // доступно везде
    protected int salary;       // доступно в пакете и подклассах
    String department;          // package-private — только в пакете
    private String password;    // только внутри этого класса

    private void authenticate() {  // private метод
        // только внутри класса
    }

    public String getName() {   // public геттер для private поля
        return name;
    }
}
```

---

## 🔹 Инкапсуляция

> [!note] Определение
> Инкапсуляция — сокрытие внутреннего состояния объекта и предоставление контролируемого доступа через методы. Поля `private`, доступ через геттеры/сеттеры.

```java
public class BankAccount {
    private String owner;
    private double balance;  // нельзя напрямую изменить снаружи

    public BankAccount(String owner, double initialBalance) {
        this.owner = owner;
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Баланс не может быть отрицательным");
        }
        this.balance = initialBalance;
    }

    // Геттеры — только чтение
    public String getOwner() {
        return owner;
    }

    public double getBalance() {
        return balance;
    }

    // Метод с валидацией (а не просто сеттер)
    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Сумма должна быть положительной");
        }
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) {
            throw new IllegalArgumentException("Недостаточно средств");
        }
        balance -= amount;
    }
}

// Использование
BankAccount acc = new BankAccount("Alice", 1000.0);
// acc.balance = -9999;  // ❌ private — ошибка компиляции
acc.deposit(500);         // ✅ контролируемый доступ
System.out.println(acc.getBalance());  // 1500.0
```

### Геттеры и сеттеры — соглашение об именовании
```java
public class Student {
    private String name;
    private int age;
    private boolean active;

    // Геттер: get + ИмяПоля
    public String getName() { return name; }
    public int getAge() { return age; }
    public boolean isActive() { return active; }  // для boolean — is, а не get

    // Сеттер: set + ИмяПоля
    public void setName(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Имя не может быть пустым");
        }
        this.name = name;
    }

    public void setAge(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Некорректный возраст");
        }
        this.age = age;
    }
}
```

> [!tip] Рекомендация
> Не делай тупые геттеры/сеттеры, которые просто возвращают/устанавливают значение без логики — если объекту нечего скрывать, рассмотри `record` (Java 16+). Смотри [[04_OOP_Principles]].

---

## 🔹 Примитивы vs Ссылочные типы

> [!note] Определение
> Примитивы хранятся **в стеке** напрямую. Объекты создаются **в куче**, а в стеке хранится только **ссылка** на объект.

```
Stack (стек)                 Heap (куча)
──────────────────           ────────────────────────────
int x = 42;                  Dog dog1 ──→ [name="Rex", age=3]
double pi = 3.14;            Dog dog2 ──→ [name="Buddy", age=5]
Dog dog1 (ссылка) ──────────→
Dog dog2 (ссылка) ──────────→
```

```java
// Примитивы передаются по ЗНАЧЕНИЮ (копия)
int a = 10;
int b = a;  // b = копия значения 10
b = 20;
System.out.println(a);  // 10 — a не изменился

// Объекты передаются по ССЫЛКЕ (одни и те же данные)
int[] arr1 = {1, 2, 3};
int[] arr2 = arr1;  // arr2 указывает на тот же массив!
arr2[0] = 99;
System.out.println(arr1[0]);  // 99 — arr1 тоже изменился!
```

> [!warning] Подводные камни
> - `null` — ссылка ни на что. Обращение к методу у `null` → `NullPointerException`
> - Присваивание объекта копирует **ссылку**, не сам объект
> - Чтобы скопировать объект нужно использовать `clone()` или конструктор копирования

### Значения по умолчанию для полей класса
```
Тип           Значение по умолчанию
──────────────────────────────────
int, byte...  0
long          0L
float         0.0f
double        0.0
boolean       false
char          '\u0000'
String        null
any Object    null
```

> [!warning] Подводные камни
> Значения по умолчанию действуют для **полей класса**, но **не** для локальных переменных внутри методов — там переменную нужно инициализировать явно.

---

## 🔹 toString() и вывод объектов

```java
public class Point {
    private int x;
    private int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() {
        return "Point(" + x + ", " + y + ")";
    }
}

Point p = new Point(3, 7);
System.out.println(p);          // "Point(3, 7)" — Java вызывает toString()
System.out.println("Point: " + p); // конкатенация тоже вызывает toString()
```

> [!tip] Рекомендация
> Всегда переопределяй `toString()` в своих классах — упрощает отладку.
