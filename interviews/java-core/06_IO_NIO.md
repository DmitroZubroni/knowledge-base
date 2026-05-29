# IO & NIO

> **Теги:** #interviews #java-core #io #nio #конспект

> [!abstract] Связи
> [[main]] | [[Interviews]]

---

## 🔹 Байтовые vs Символьные потоки

### Байтовые потоки (Byte Streams)

Работают с сырыми байтами (8-bit).

```java
InputStream  // чтение байтов
OutputStream // запись байтов
```

**Примеры:**
- `FileInputStream`, `FileOutputStream`
- `BufferedInputStream`, `BufferedOutputStream`
- `DataInputStream`, `DataOutputStream`

### Символьные потоки (Character Streams)

Работают с символами (16-bit Unicode), автоматически кодируют/декодируют.

```java
Reader  // чтение символов
Writer  // запись символов
```

**Примеры:**
- `FileReader`, `FileWriter`
- `BufferedReader`, `BufferedWriter`
- `InputStreamReader`, `OutputStreamWriter`

### Когда использовать

| Тип | Когда использовать |
|-----|-------------------|
| **Байтовые** | Изображения, аудио, видео, бинарные файлы |
| **Символьные** | Текстовые файлы, CSV, JSON, XML |

> [!tip] Байтовые → Символьные
```java
// Байтовый поток → Символьный
InputStreamReader reader = new InputStreamReader(inputStream, StandardCharsets.UTF_8);
```

---

## 🔹 InputStream/OutputStream

### InputStream

```java
try (InputStream is = new FileInputStream("file.txt")) {
    int data = is.read();  // чтение одного байта
    while (data != -1) {
        System.out.print((char) data);
        data = is.read();
    }
}
```

### OutputStream

```java
try (OutputStream os = new FileOutputStream("file.txt")) {
    os.write(65);  // записывает 'A'
    os.write("Hello".getBytes());
}
```

### Буферизированные потоки

```java
// Буферизированное чтение (быстрее)
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("file.txt"))) {
    int data = bis.read();
    while (data != -1) {
        // ...
    }
}
```

---

## 🔹 Reader/Writer

### Reader

```java
try (Reader reader = new FileReader("file.txt")) {
    int data = reader.read();  // чтение одного символа
    while (data != -1) {
        System.out.print((char) data);
        data = reader.read();
    }
}
```

### BufferedReader

```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

### Writer

```java
try (Writer writer = new FileWriter("file.txt")) {
    writer.write("Hello");
    writer.write("World");
}
```

### BufferedWriter

```java
try (BufferedWriter bw = new BufferedWriter(new FileWriter("file.txt"))) {
    bw.write("Hello");
    bw.newLine();
    bw.write("World");
}
```

---

## 🔹 File vs Path (Java 7+)

### File (старый API)

```java
File file = new File("file.txt");
if (file.exists()) {
    System.out.println(file.getAbsolutePath());
}
```

**Проблемы:**
- Много методов возвращают `boolean` (не знают причину ошибки)
- Нет информации о symbolic links
- Неэффективно для больших операций

### Path (новый API, Java 7+)

```java
Path path = Paths.get("file.txt");
if (Files.exists(path)) {
    System.out.println(path.toAbsolutePath());
}
```

**Преимущества:**
- Больше информации об ошибках
- Поддержка symbolic links
- Лучше работает с большими файлами
- Более удобный API

### Files utility class

```java
Path path = Paths.get("file.txt");

// Чтение
List<String> lines = Files.readAllLines(path);
String content = Files.readString(path);

// Запись
Files.writeString(path, "Hello");
Files.write(path, List.of("Line1", "Line2"));

// Копирование/перемещение
Files.copy(source, target);
Files.move(source, target);

// Создание/удаление
Files.createFile(path);
Files.delete(path);
```

---

## 🔹 Decorator Pattern в IO

**Decorator Pattern** — обёртывание потока для добавления функциональности.

### Пример

```java
// Базовый поток
InputStream is = new FileInputStream("file.txt");

// Добавляем буферизацию
InputStream bis = new BufferedInputStream(is);

// Добавляем поддержку примитивов
DataInputStream dis = new DataInputStream(bis);

int value = dis.readInt();
```

### Цепочка декораторов

```
FileInputStream
    ↓
BufferedInputStream  (буферизация)
    ↓
DataInputStream     (чтение примитивов)
```

> [!tip] Порядок важен
- Сначала буферизация, потом функциональность
- Буферизация должна быть ближе к источнику

---

## 🔹 NIO (New I/O)

**NIO** — неблокирующий I/O, введённый в Java 4, улучшен в Java 7 (NIO.2).

### Ключевые отличия NIO от IO

| Характеристика | IO (Blocking) | NIO (Non-blocking) |
|----------------|---------------|-------------------|
| Модель | Блокирующая | Неблокирующая |
| Потоки | Один поток на соединение | Один поток на много соединений |
| Буферизация | Потоковая | Буферная (Buffer) |
| Селекторы | Нет | Есть (Selector) |

### Buffer

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

// Запись
buffer.put("Hello".getBytes());

// Чтение
buffer.flip();  // переключение в режим чтения
while (buffer.hasRemaining()) {
    byte b = buffer.get();
}
```

### Channel

```java
try (FileChannel channel = FileChannel.open(Paths.get("file.txt"), StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    int bytesRead = channel.read(buffer);
}
```

### Selector (для сетевого NIO)

```java
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // блокируется пока есть события
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    // обработка событий...
}
```

> [!tip] Когда использовать NIO
- Высоконкурентные серверы (тысячи соединений)
- Когда нужно масштабируемость
- Когда важна производительность

---

## 🔹 Итог

> [!tip] Шпаргалка
> - **Байтовые:** InputStream/OutputStream — для бинарных данных
> - **Символьные:** Reader/Writer — для текста
> - **File vs Path:** Path — новый API (Java 7+), лучше
> - **Decorator Pattern:** обёртывание потоков (BufferedInputStream)
> - **NIO:** неблокирующий I/O, Buffer, Channel, Selector

```
Байтовые vs Символьные:
InputStream/OutputStream → байты (изображения, бинарные)
Reader/Writer → символы (текст)

File vs Path:
File — старый API
Path — новый API (Java 7+), Files utility class

Decorator Pattern:
FileInputStream → BufferedInputStream → DataInputStream

NIO:
Buffer → Channel → Selector
неблокирующий I/O, высокая производительность
```
