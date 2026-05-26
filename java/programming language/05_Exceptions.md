# 05 — Исключения (Exceptions)

> **Теги:** #java #programming #exceptions #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[00_JAVA]]

---

## 🔹 Иерархия исключений

```
Throwable
├── Error                          ← НЕ перехватывать (критические ошибки JVM)
│   ├── StackOverflowError
│   ├── OutOfMemoryError
│   └── AssertionError
└── Exception                      ← Перехватывать и обрабатывать
    ├── IOException                 ← Checked (проверяемые)
    ├── SQLException                ← Checked
    ├── ClassNotFoundException      ← Checked
    └── RuntimeException            ← Unchecked (непроверяемые)
        ├── NullPointerException
        ├── ArrayIndexOutOfBoundsException
        ├── ClassCastException
        ├── NumberFormatException
        ├── IllegalArgumentException
        ├── IllegalStateException
        └── ArithmeticException
```

### Checked vs Unchecked

| | Checked | Unchecked |
|-|---------|-----------|
| Наследует от | `Exception` | `RuntimeException` |
| Компилятор | Требует обработки | Не требует |
| Когда | Внешние ресурсы (файлы, БД, сеть) | Ошибки программиста |
| Примеры | `IOException`, `SQLException` | `NullPointerException`, `ClassCastException` |

---

## 🔹 try / catch / finally

> [!note] Определение
> `try` — защищённый блок. `catch` — обработка исключения. `finally` — выполняется **всегда** (даже при исключении или return).

### Базовая структура
```java
try {
    // код, который может выбросить исключение
    int result = 10 / 0;   // ArithmeticException
    System.out.println(result);
} catch (ArithmeticException e) {
    // обработка конкретного исключения
    System.out.println("Ошибка: " + e.getMessage());  // "/ by zero"
} finally {
    // выполняется всегда — закрытие ресурсов
    System.out.println("finally блок");
}
```

### Несколько catch блоков
```java
public static int parseInt(String s) {
    try {
        return Integer.parseInt(s);
    } catch (NumberFormatException e) {
        System.out.println("Не число: " + s);
        return 0;
    } catch (NullPointerException e) {
        System.out.println("null строка");
        return 0;
    }
}
```

### Multi-catch (Java 7+)
```java
try {
    // ...
} catch (NumberFormatException | NullPointerException e) {
    System.out.println("Ошибка формата или null: " + e.getMessage());
}
```

### Порядок catch — от частного к общему
```java
try {
    // ...
} catch (NullPointerException e) {    // 1. конкретнее
    System.out.println("NPE");
} catch (RuntimeException e) {        // 2. шире
    System.out.println("Runtime error");
} catch (Exception e) {               // 3. самый широкий
    System.out.println("General error");
}
```

> [!danger] Опасность
> Ловить `Exception` или `Throwable` — плохая практика. Маскирует реальные ошибки и перехватывает то, что не планировалось.

---

## 🔹 throw и throws

### throw — бросить исключение
```java
public static double divide(double a, double b) {
    if (b == 0) {
        throw new ArithmeticException("Делитель не может быть нулём");
    }
    return a / b;
}

public static void setAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("Некорректный возраст: " + age);
    }
    // ...
}
```

### throws — объявление checked исключений
```java
// Метод объявляет, что МОЖЕТ бросить IOException
public static String readFile(String path) throws IOException {
    // FileReader может бросить IOException — checked исключение
    FileReader reader = new FileReader(path);  // обязательно обрабатывать!
    // ...
    return "";
}

// Вызывающий код ОБЯЗАН обработать
public static void main(String[] args) {
    try {
        String content = readFile("data.txt");
    } catch (IOException e) {
        System.out.println("Файл не найден: " + e.getMessage());
    }
}
```

> [!note] throw vs throws
> `throw` — действие (выбросить исключение прямо сейчас).
> `throws` — декларация в сигнатуре метода (предупреждение что метод может бросить).

---

## 🔹 try-with-resources (Java 7+)

> [!note] Определение
> Автоматически закрывает ресурсы реализующие `AutoCloseable` по завершению блока `try`. Заменяет `finally` для закрытия ресурсов.

```java
// Старый способ — с finally
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("file.txt"));
    String line = br.readLine();
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (br != null) {
        try {
            br.close();  // может тоже бросить исключение!
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

// Новый способ — try-with-resources
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
    System.out.println(line);
} catch (IOException e) {
    System.out.println("Ошибка чтения: " + e.getMessage());
}
// br.close() вызовется автоматически

// Несколько ресурсов
try (
    Connection conn = DriverManager.getConnection(url);
    PreparedStatement stmt = conn.prepareStatement(sql)
) {
    // работа с базой данных
} catch (SQLException e) {
    e.printStackTrace();
}
```

> [!tip] Рекомендация
> Всегда используй `try-with-resources` для файлов, соединений с БД, сетевых сокетов — это надёжнее чем `finally`.

---

## 🔹 Кастомные исключения

> [!note] Определение
> Собственные исключения наследуют от `Exception` (checked) или `RuntimeException` (unchecked). Используются для специфических ошибок домена.

### Unchecked кастомное исключение
```java
public class InsufficientFundsException extends RuntimeException {
    private final double amount;
    private final double balance;

    public InsufficientFundsException(double amount, double balance) {
        super(String.format("Недостаточно средств. Нужно: %.2f, есть: %.2f", amount, balance));
        this.amount = amount;
        this.balance = balance;
    }

    public double getAmount() { return amount; }
    public double getBalance() { return balance; }
}

// Checked кастомное исключение
public class UserNotFoundException extends Exception {
    private final long userId;

    public UserNotFoundException(long userId) {
        super("Пользователь не найден: id=" + userId);
        this.userId = userId;
    }

    public long getUserId() { return userId; }
}
```

### Использование
```java
public class BankAccount {
    private double balance;

    public void withdraw(double amount) {
        if (amount > balance) {
            throw new InsufficientFundsException(amount, balance);  // unchecked — throws не нужен
        }
        balance -= amount;
    }
}

public class UserService {
    public User findById(long id) throws UserNotFoundException {  // checked — обязателен throws
        User user = database.find(id);
        if (user == null) {
            throw new UserNotFoundException(id);
        }
        return user;
    }
}

// Обработка
try {
    User user = userService.findById(999L);
} catch (UserNotFoundException e) {
    System.out.println(e.getMessage());
    System.out.println("ID: " + e.getUserId());
}
```

---

## 🔹 Цепочка исключений (Exception Chaining)

```java
public void processFile(String path) {
    try {
        readFile(path);
    } catch (IOException e) {
        // Оборачиваем в более высокоуровневое исключение
        throw new RuntimeException("Не удалось обработать файл: " + path, e);
        //                                                                   ↑ исходное исключение сохраняется
    }
}

// Получить исходную причину
try {
    processFile("data.txt");
} catch (RuntimeException e) {
    System.out.println("Ошибка: " + e.getMessage());
    System.out.println("Причина: " + e.getCause().getMessage()); // исходный IOException
    e.printStackTrace();  // полный стек вызовов
}
```

---

## 🔹 Полезные методы исключений

```java
try {
    int[] arr = new int[5];
    arr[10] = 1;  // ArrayIndexOutOfBoundsException
} catch (Exception e) {
    e.getMessage()       // строка с описанием ошибки
    e.getClass()         // java.lang.ArrayIndexOutOfBoundsException
    e.getClass().getSimpleName()  // "ArrayIndexOutOfBoundsException"
    e.getCause()         // исходное исключение (если есть)
    e.printStackTrace()  // вывести полный стек вызовов в stderr
    e.getStackTrace()    // массив StackTraceElement[]
}
```

---

## 🔹 Принципы работы с исключениями

> [!tip] Когда бросать исключение
> - Параметр не прошёл валидацию → `IllegalArgumentException`
> - Объект в некорректном состоянии → `IllegalStateException`
> - Значение null там, где не ожидается → `NullPointerException` или проверь сам
> - Ресурс не найден → кастомное `NotFoundException extends RuntimeException`

> [!tip] Когда ловить исключение
> - Только если можешь **обработать** и **восстановиться**
> - Не лови то, что не умеешь обрабатывать — пробрось выше
> - Логируй исключения с контекстом (какой файл, какой ID и т.д.)

> [!danger] Антипаттерны
> ```java
> // ❌ Пустой catch — "проглотить" исключение
> try {
>     riskyOperation();
> } catch (Exception e) {
>     // ничего не делать — НИКОГДА так не делай!
> }
>
> // ❌ Ловить Exception вместо конкретного типа
> catch (Exception e) { ... }
>
> // ❌ Использовать исключения для управления потоком
> try {
>     while (true) { array[i++] = 1; }  // ждём ArrayIndexOutOfBoundsException
> } catch (ArrayIndexOutOfBoundsException e) { }  // это ужасно!
> ```
