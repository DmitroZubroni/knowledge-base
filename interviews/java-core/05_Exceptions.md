# Exceptions

> **Теги:** #interviews #java-core #exceptions #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Иерархия исключений

```
Throwable
    ├── Error (системные ошибки, не обрабатываются)
    │   ├── OutOfMemoryError
    │   ├── StackOverflowError
    │   └── ...
    └── Exception (исключения, которые можно обрабатывать)
        ├── RuntimeException (unchecked)
        │   ├── NullPointerException
        │   ├── IllegalArgumentException
        │   ├── IndexOutOfBoundsException
        │   └── ...
        └── Checked Exception (проверяемые)
            ├── IOException
            │   ├── FileNotFoundException
            │   └── ...
            ├── SQLException
            └── ...
```

> [!note] Error vs Exception
- **Error** — системные ошибки, которые обычно не обрабатываются (OutOfMemoryError, StackOverflowError)
- **Exception** — исключения, которые можно и нужно обрабатывать

---

## 🔹 Checked vs Unchecked

### Checked Exception

**Checked Exception** — исключение, которое обязательно нужно обработать или объявить в throws.

```java
// Checked — нужно обработать
public void readFile() throws IOException {
    FileReader reader = new FileReader("file.txt");
}
```

**Примеры:**
- `IOException`
- `SQLException`
- `FileNotFoundException`
- `ClassNotFoundException`

### Unchecked Exception

**Unchecked Exception** — наследники `RuntimeException`, не нужно обрабатывать явно.

```java
// Unchecked — можно не обрабатывать
public void divide(int a, int b) {
    return a / b;  // ArithmeticException если b = 0
}
```

**Примеры:**
- `NullPointerException`
- `IllegalArgumentException`
- `IndexOutOfBoundsException`
- `ArithmeticException`

### Сравнение

| Характеристика | Checked | Unchecked |
|----------------|---------|------------|
| Обязательна обработка | Да | Нет |
| Наследует | Exception | RuntimeException |
| Когда использовать | Восстановимые ошибки | Программные ошибки |

> [!tip] Когда использовать Checked vs Unchecked
- **Checked** — внешние факторы (файл не найден, сеть недоступна)
- **Unchecked** — ошибки в коде (NPE, неверные аргументы)

---

## 🔹 try-catch-finally

### Базовый синтаксис

```java
try {
    // код, который может выбросить исключение
} catch (IOException e) {
    // обработка IOException
} catch (Exception e) {
    // обработка остальных исключений
} finally {
    // выполняется всегда
}
```

### Множественный catch (Java 7+)

```java
try {
    // код
} catch (IOException | SQLException e) {
    // обработка обоих исключений
}
```

### finally

**finally** — блок, который выполняется всегда (даже если было исключение или return).

```java
try {
    return 1;
} finally {
    System.out.println("Always executed");  // выполнится
}
```

> [!warning] finally не выполнится если
- `System.exit()`
- Поток убит (`kill -9`)
- Бесконечный цикл в finally

---

## 🔹 try-with-resources (Java 7+)

**try-with-resources** — автоматическое закрытие ресурсов, реализующих `AutoCloseable`.

### Синтаксис

```java
try (FileReader reader = new FileReader("file.txt")) {
    // используем reader
}  // reader.close() вызывается автоматически
```

### Множественные ресурсы

```java
try (FileReader reader = new FileReader("file.txt");
     BufferedReader br = new BufferedReader(reader)) {
    // используем ресурсы
}  // оба закрываются в обратном порядке
```

### До Java 7 (старый способ)

```java
FileReader reader = null;
try {
    reader = new FileReader("file.txt");
    // используем reader
} finally {
    if (reader != null) {
        reader.close();  // может выбросить исключение!
    }
}
```

> [!tip] try-with-resources vs finally
- try-with-resources — автоматически закрывает, даже при исключении
- finally — нужно вручную закрывать, возможны ошибки

---

## 🔹 throw vs throws

### throw

**throw** — выбрасывание исключения в коде.

```java
if (age < 0) {
    throw new IllegalArgumentException("Age must be positive");
}
```

### throws

**throws** — объявление исключений в сигнатуре метода.

```java
public void readFile(String path) throws IOException {
    FileReader reader = new FileReader(path);
}
```

### Сравнение

| Ключевое слово | Назначение | Где используется |
|----------------|------------|-----------------|
| **throw** | Выбросить исключение | В теле метода |
| **throws** | Объявить исключения | В сигнатуре метода |

```java
public void validate(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Invalid age");  // throw
    }
}

public void process() throws IOException {  // throws
    readFile("file.txt");
}
```

---

## 🔹 Создание собственных исключений

### Checked исключение

```java
public class CustomCheckedException extends Exception {
    public CustomCheckedException(String message) {
        super(message);
    }
}

// Использование
public void doSomething() throws CustomCheckedException {
    if (error) {
        throw new CustomCheckedException("Something went wrong");
    }
}
```

### Unchecked исключение

```java
public class CustomUncheckedException extends RuntimeException {
    public CustomUncheckedException(String message) {
        super(message);
    }
}

// Использование
public void doSomething() {
    if (error) {
        throw new CustomUncheckedException("Something went wrong");
    }
}
```

> [!tip] Когда создавать Checked vs Unchecked
- **Checked** — если вызывающий код может восстановиться
- **Unchecked** — если это программная ошибка

---

## 🔹 Common Pitfalls

### 1. Пустой catch

```java
// ПЛОХО — глотаем исключение
try {
    riskyOperation();
} catch (Exception e) {
    // пусто
}
```

### 2. Catch Exception

```java
// ПЛОХО — ловим всё, включая RuntimeException
try {
    riskyOperation();
} catch (Exception e) {
    // слишком широко
}
```

### 3. Игнорирование finally

```java
try {
    return 1;
} finally {
    return 2;  // ПЛОХО — перезаписывает return из try
}
```

### 4. Throw в finally

```java
try {
    throw new Exception1();
} finally {
    throw new Exception2();  // Exception1 будет потерян
}
```

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Checked** — обязательно обработать (IOException, SQLException)
> - **Unchecked** — RuntimeException, не обязательно (NPE, IllegalArgumentException)
> - **try-with-resources** — автоматическое закрытие (Java 7+)
> - **throw** — выбросить исключение в коде
> - **throws** — объявить в сигнатуре метода
> - **finally** — выполняется всегда

```
Иерархия:
Throwable → Error (не обрабатываем)
         → Exception → RuntimeException (unchecked)
                    → Checked (IOException, SQLException)

try-catch-finally:
try → catch → finally (всегда)

try-with-resources:
try (Resource r) { }  // auto-close

throw vs throws:
throw new Exception()      // в теле метода
void method() throws Ex   // в сигнатуре
```
