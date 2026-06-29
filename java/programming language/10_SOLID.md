# 10 — Принципы SOLID

> **Теги:** #java #programming #solid #design #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]]

> [!note] Что такое SOLID
> SOLID — 5 принципов объектно-ориентированного дизайна, снижающих связанность, повышающих расширяемость и тестируемость кода. Не догма — инструмент мышления.

---

## 🔹 S — Single Responsibility Principle

> [!note] Определение
> Класс должен иметь **одну причину для изменения**. Один класс — одна ответственность.

### ❌ Нарушение SRP
```java
// Один класс делает слишком много
public class Report {
    private String title;
    private String content;

    // ответственность 1: хранить данные отчёта
    public String getTitle() { return title; }
    public String getContent() { return content; }

    // ответственность 2: форматирование
    public String toHtml() {
        return "<h1>" + title + "</h1><p>" + content + "</p>";
    }

    // ответственность 3: сохранение
    public void saveToFile(String path) {
        // запись в файл
    }

    // ответственность 4: отправка
    public void sendByEmail(String recipient) {
        // отправка email
    }
}
// Причины изменения: изменился формат HTML, изменился способ хранения,
// изменился почтовый сервер — всё меняет один класс
```

### ✅ Соблюдение SRP
```java
// Каждый класс — одна ответственность

// 1. Данные
public class Report {
    private final String title;
    private final String content;

    public Report(String title, String content) {
        this.title = title;
        this.content = content;
    }

    public String getTitle() { return title; }
    public String getContent() { return content; }
}

// 2. Форматирование
public class ReportFormatter {
    public String toHtml(Report report) {
        return "<h1>" + report.getTitle() + "</h1><p>" + report.getContent() + "</p>";
    }

    public String toPlainText(Report report) {
        return report.getTitle() + "\n" + report.getContent();
    }
}

// 3. Сохранение
public class ReportRepository {
    public void saveToFile(Report report, String path) {
        // запись в файл
    }

    public void saveToDatabase(Report report) {
        // сохранение в БД
    }
}

// 4. Отправка
public class ReportEmailSender {
    public void send(Report report, String recipient) {
        // отправка email
    }
}
```

---

## 🔹 O — Open/Closed Principle

> [!note] Определение
> Класс **открыт для расширения** (новый функционал через наследование/реализацию) и **закрыт для модификации** (не трогаем существующий код).

### ❌ Нарушение OCP
```java
// При добавлении нового типа фигуры — придётся менять AreaCalculator
public class AreaCalculator {
    public double calculateArea(Object shape) {
        if (shape instanceof Circle) {
            Circle c = (Circle) shape;
            return Math.PI * c.radius * c.radius;
        } else if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle) shape;
            return r.width * r.height;
        }
        // Нужен треугольник? Меняем существующий класс!
        throw new IllegalArgumentException("Unknown shape");
    }
}
```

### ✅ Соблюдение OCP
```java
// Интерфейс — контракт
public interface Shape {
    double area();
}

// Реализации — расширения без изменения контракта
public class Circle implements Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }

    @Override
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double w, double h) { width = w; height = h; }

    @Override
    public double area() { return width * height; }
}

// Добавить треугольник? Просто создаём новый класс, ничего не меняем!
public class Triangle implements Shape {
    private final double base, height;
    public Triangle(double b, double h) { base = b; height = h; }

    @Override
    public double area() { return 0.5 * base * height; }
}

// AreaCalculator не меняется
public class AreaCalculator {
    public double calculateTotal(List<Shape> shapes) {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }
}
```

---

## 🔹 L — Liskov Substitution Principle

> [!note] Определение
> Объект подкласса должен **полностью заменять** объект суперкласса без изменения корректности программы. Не нарушай контракт родителя.

### ❌ Нарушение LSP
```java
public class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
}

// Пингвин — птица, но не летает → нарушение контракта Bird
public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}

public class FlightSimulator {
    public void makeFly(Bird bird) {
        bird.fly();  // сломается на Penguin!
    }
}
```

### ✅ Соблюдение LSP
```java
// Пересмотр иерархии — выделяем интерфейс
public interface Bird {
    void eat();
    void makeSound();
}

public interface FlyingBird extends Bird {
    void fly();
}

public class Sparrow implements FlyingBird {
    @Override public void eat() { System.out.println("Sparrow eating"); }
    @Override public void makeSound() { System.out.println("Tweet!"); }
    @Override public void fly() { System.out.println("Sparrow flying"); }
}

public class Penguin implements Bird {
    @Override public void eat() { System.out.println("Penguin eating fish"); }
    @Override public void makeSound() { System.out.println("Squawk!"); }
    // нет fly() — и не нужен
}

// Теперь симулятор работает только с летающими птицами
public class FlightSimulator {
    public void makeFly(FlyingBird bird) {
        bird.fly();  // Penguin сюда не попадёт
    }
}
```

### Контракт LSP: правила для подклассов
```
✅ Предусловия: подкласс НЕ может усилить (требовать больше от вызывающего)
✅ Постусловия: подкласс НЕ может ослабить (гарантировать меньше вызывающему)
✅ Инварианты: подкласс должен соблюдать все инварианты суперкласса
✅ Исключения: подкласс НЕ должен бросать новые checked исключения
```

---

## 🔹 I — Interface Segregation Principle

> [!note] Определение
> Много специализированных интерфейсов лучше одного "жирного". Клиент не должен зависеть от методов, которые ему не нужны.

### ❌ Нарушение ISP
```java
// "Жирный" интерфейс — многофункциональный документ
public interface IDocument {
    void open();
    void close();
    void save();
    void print();
    void sendByEmail();
    void convertToPdf();
}

// ReadOnly документ вынужден реализовывать ненужные методы
public class ReadOnlyDocument implements IDocument {
    @Override public void open() { /* OK */ }
    @Override public void close() { /* OK */ }

    @Override
    public void save() {
        throw new UnsupportedOperationException("Read only!");
    }
    @Override
    public void print() { /* OK */ }
    @Override
    public void sendByEmail() { /* OK */ }
    @Override
    public void convertToPdf() {
        throw new UnsupportedOperationException("Not supported!");
    }
}
```

### ✅ Соблюдение ISP
```java
// Разделяем на маленькие специализированные интерфейсы
public interface Openable {
    void open();
    void close();
}

public interface Saveable {
    void save();
}

public interface Printable {
    void print();
}

public interface Emailable {
    void sendByEmail();
}

public interface PdfConvertible {
    void convertToPdf();
}

// ReadOnly документ реализует только то что нужно
public class ReadOnlyDocument implements Openable, Printable, Emailable {
    @Override public void open() { /* OK */ }
    @Override public void close() { /* OK */ }
    @Override public void print() { /* OK */ }
    @Override public void sendByEmail() { /* OK */ }
}

// Полноценный документ
public class FullDocument implements Openable, Saveable, Printable, Emailable, PdfConvertible {
    @Override public void open() { /* ... */ }
    @Override public void close() { /* ... */ }
    @Override public void save() { /* ... */ }
    @Override public void print() { /* ... */ }
    @Override public void sendByEmail() { /* ... */ }
    @Override public void convertToPdf() { /* ... */ }
}
```

---

## 🔹 D — Dependency Inversion Principle

> [!note] Определение
> Модули высокого уровня не должны зависеть от модулей низкого уровня. **Оба должны зависеть от абстракций**. Абстракции не должны зависеть от деталей — детали зависят от абстракций.

### ❌ Нарушение DIP
```java
// Высокоуровневый класс зависит от конкретной реализации
public class StudentService {
    // Прямая зависимость от конкретного класса
    private StudentDatabase database = new StudentDatabase();

    public void enrollStudent(Student student) {
        // проверки...
        database.save(student);
    }
}

// Хочу использовать другую БД? Придётся менять StudentService!
```

### ✅ Соблюдение DIP + Dependency Injection
```java
// 1. Абстракция (интерфейс)
public interface StudentRepository {
    void save(Student student);
    Optional<Student> findById(long id);
    List<Student> findAll();
    void delete(long id);
}

// 2. Конкретные реализации (детали)
public class PostgresStudentRepository implements StudentRepository {
    @Override
    public void save(Student student) { /* SQL INSERT */ }
    @Override
    public Optional<Student> findById(long id) { /* SQL SELECT */ return Optional.empty(); }
    @Override
    public List<Student> findAll() { /* SQL SELECT * */ return List.of(); }
    @Override
    public void delete(long id) { /* SQL DELETE */ }
}

public class InMemoryStudentRepository implements StudentRepository {
    private final Map<Long, Student> storage = new HashMap<>();

    @Override
    public void save(Student student) { storage.put(student.getId(), student); }
    @Override
    public Optional<Student> findById(long id) { return Optional.ofNullable(storage.get(id)); }
    @Override
    public List<Student> findAll() { return new ArrayList<>(storage.values()); }
    @Override
    public void delete(long id) { storage.remove(id); }
}

// 3. Высокоуровневый класс зависит от абстракции (DIP)
//    Зависимость передаётся через конструктор (Dependency Injection)
public class StudentService {
    private final StudentRepository repository;  // интерфейс, не реализация!

    // Dependency Injection через конструктор
    public StudentService(StudentRepository repository) {
        this.repository = repository;
    }

    public void enrollStudent(Student student) {
        // проверки...
        repository.save(student);
    }

    public Optional<Student> getStudent(long id) {
        return repository.findById(id);
    }
}

// 4. Инициализация (IoC Container, или вручную)
// Продакшн:
StudentService service = new StudentService(new PostgresStudentRepository());

// Тесты:
StudentService testService = new StudentService(new InMemoryStudentRepository());
```

### Паттерны DIP
```
Dependency Injection (DI):
  Constructor Injection  — через конструктор (предпочтительно)
  Setter Injection       — через setter метод
  Field Injection        — через @Autowired (Spring, не рекомендуется для тестирования)

Inversion of Control (IoC):
  Фреймворк управляет созданием объектов и передачей зависимостей
  Примеры: Spring Framework, Guice, Jakarta CDI
```

---

## 🔹 Итог SOLID

```
S — Single Responsibility   Один класс = одна ответственность = одна причина для изменения
O — Open/Closed             Расширяй через наследование, не меняй существующий код
L — Liskov Substitution     Подкласс полностью заменяет родителя, не нарушает контракт
I — Interface Segregation   Много маленьких интерфейсов лучше одного большого
D — Dependency Inversion    Зависи от абстракций, передавай зависимости снаружи
```

> [!warning] Минусы и ограничения SOLID
> - **Усложняет код** — больше классов, интерфейсов, файлов
> - **Увеличивает время разработки** — особенно на начальном этапе
> - **Не универсален** — для маленьких скриптов и утилит SOLID излишен
> - **Может привести к over-engineering** — рефакторинг без реальной необходимости
> - Применяй там, где ожидается рост и изменения кода

> [!tip] Когда применять SOLID
> - Код будут читать и поддерживать другие разработчики
> - Компонент будет расширяться с течением времени
> - Нужно покрывать код unit-тестами
> - Работаешь над библиотекой или API

---

## 🔗 Связанные конспекты

> [!abstract] Смотри также
> - [[14_OOP_SOLID]] (интервью) — SOLID, DRY, YAGNI, KISS на собеседовании
> - [[12_Design_Patterns]] — паттерны GoF реализуют принципы SOLID
> - [[15_Patterns]] (интервью) — паттерны проектирования на собеседовании
> - [[16_Microservices]] (интервью) — SOLID в контексте микросервисной архитектуры
