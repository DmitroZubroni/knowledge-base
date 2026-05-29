# Design Patterns

> **Теги:** #interviews #architecture #patterns #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Singleton (thread-safe)

**Singleton** — паттерн, гарантирующий что класс имеет только один экземпляр.

### Плохая реализация (не thread-safe)

```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() { }
    
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();  // race condition!
        }
        return instance;
    }
}
```

### Thread-safe реализации

#### 1. Synchronized method (простой, но медленный)

```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() { }
    
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

#### 2. Double-Checked Locking (рекомендуется)

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() { }
    
    public static Singleton getInstance() {
        if (instance == null) {  // first check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {  // second check (with locking)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

> [!tip] Почему volatile?
Без volatile возможен reordering: объект может быть не полностью инициализирован при возврате.

#### 3. Static holder (Bill Pugh Solution)

```java
public class Singleton {
    private Singleton() { }
    
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Преимущества:**
- Thread-safe без synchronized
- Lazy initialization
- Высокая производительность

#### 4. Enum (простейший)

```java
public enum Singleton {
    INSTANCE;
    
    public void doSomething() {
        // ...
    }
}
```

**Преимущества:**
- Самый простой
- Thread-safe по умолчанию
- Сериализация handled automatically

---

## 🔹 Proxy

**Proxy** — замещающий объект, контролирующий доступ к другому объекту.

### Виды Proxy

| Тип | Назначение |
|-----|------------|
| **Virtual Proxy** | Ленивая загрузка |
| **Protection Proxy** | Контроль доступа |
| **Remote Proxy** | Удалённый доступ |
| **Smart Reference** | Дополнительная логика |

### Пример Virtual Proxy

```java
interface Image {
    void display();
}

class RealImage implements Image {
    private String filename;
    
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();
    }
    
    private void loadFromDisk() {
        System.out.println("Loading " + filename);
    }
    
    public void display() {
        System.out.println("Displaying " + filename);
    }
}

class ProxyImage implements Image {
    private RealImage realImage;
    private String filename;
    
    public ProxyImage(String filename) {
        this.filename = filename;
    }
    
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);  // lazy loading
        }
        realImage.display();
    }
}
```

### Spring @Transactional как Proxy

```java
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        repository.save(user);
    }
}
```

Spring создаёт прокси вокруг `UserService` для управления транзакцией.

---

## 🔹 Builder

**Builder** — пошаговое создание сложных объектов.

### Пример

```java
public class User {
    private final String name;
    private final int age;
    private final String email;
    
    private User(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
    }
    
    public static class Builder {
        private String name;
        private int age;
        private String email;
        
        public Builder name(String name) {
            this.name = name;
            return this;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Использование
User user = new User.Builder()
    .name("Alice")
    .age(30)
    .email("alice@example.com")
    .build();
```

### Преимущества

- Читаемый код
- Неизменяемые объекты (immutable)
- Опциональные параметры
- Валидация в `build()`

### Lombok @Builder

```java
@Builder
public class User {
    private String name;
    private int age;
    private String email;
}
```

---

## 🔹 Adapter

**Adapter** — преобразование интерфейса одного класса в интерфейс другого.

### Пример

```java
// Существующий интерфейс
interface MediaPlayer {
    void play(String audioType, String fileName);
}

// Реализация
class AudioPlayer implements MediaPlayer {
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("Playing mp3: " + fileName);
        } else {
            System.out.println("Invalid media: " + audioType);
        }
    }
}

// Новый интерфейс
interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

// Реализация нового интерфейса
class VlcPlayer implements AdvancedMediaPlayer {
    public void playVlc(String fileName) {
        System.out.println("Playing vlc: " + fileName);
    }
    public void playMp4(String fileName) { }
}

class Mp4Player implements AdvancedMediaPlayer {
    public void playVlc(String fileName) { }
    public void playMp4(String fileName) {
        System.out.println("Playing mp4: " + fileName);
    }
}

// Adapter
class MediaAdapter implements MediaPlayer {
    AdvancedMediaPlayer advancedMusicPlayer;
    
    public MediaAdapter(String audioType) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedMusicPlayer = new VlcPlayer();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedMusicPlayer = new Mp4Player();
        }
    }
    
    public void play(String audioType, String fileName) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedMusicPlayer.playVlc(fileName);
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedMusicPlayer.playMp4(fileName);
        }
    }
}
```

### Когда использовать

- Интеграция с legacy кодом
- Необходимость использовать классы с несовместимыми интерфейсами

---

## 🔹 Decorator (IO pattern)

**Decorator** — динамическое добавление ответственности объекту.

### Пример

```java
interface Coffee {
    double cost();
    String description();
}

class SimpleCoffee implements Coffee {
    public double cost() { return 1.0; }
    public String description() { return "Simple coffee"; }
}

// Decorator
abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
}

class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    public double cost() {
        return coffee.cost() + 0.5;
    }
    
    public String description() {
        return coffee.description() + ", milk";
    }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    public double cost() {
        return coffee.cost() + 0.2;
    }
    
    public String description() {
        return coffee.description() + ", sugar";
    }
}

// Использование
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
// cost: 1.0 + 0.5 + 0.2 = 1.7
```

### Java IO как Decorator pattern

```java
// Базовый поток
InputStream inputStream = new FileInputStream("file.txt");

// Добавляем буферизацию (decorator)
InputStream buffered = new BufferedInputStream(inputStream);

// Добавляем поддержку примитивов (decorator)
DataInputStream data = new DataInputStream(buffered);

int value = data.readInt();
```

### Схема

```
FileInputStream
    ↓ (decorator)
BufferedInputStream
    ↓ (decorator)
DataInputStream
```

> [!tip] Decorator vs Inheritance
- Decorator — динамическое добавление (runtime)
- Inheritance — статическое (compile-time)

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Singleton:** один экземпляр, thread-safe (volatile, double-checked, enum)
> - **Proxy:** замещающий объект, контролирующий доступ (Virtual, Protection, Remote)
> - **Builder:** пошаговое создание сложных объектов (immutable, optional params)
> - **Adapter:** преобразование интерфейса (интеграция legacy)
> - **Decorator:** динамическое добавление ответственности (Java IO)

```
Singleton:
Double-Checked Locking → volatile + synchronized
Enum → самый простой
Static holder → lazy + thread-safe

Proxy:
Virtual → lazy loading
Protection → access control
Spring @Transactional → proxy

Builder:
Пошаговое создание
Immutable объекты
Lombok @Builder

Adapter:
Преобразование интерфейса
Интеграция legacy кода

Decorator:
Динамическое добавление ответственности
Java IO → FileInputStream → BufferedInputStream → DataInputStream
```
