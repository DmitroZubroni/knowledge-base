# 11 — JVM Internals

> **Теги:** #java #programming #jvm #internals #конспект  

> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]]

---

## 🔹 Компиляция и выполнение Java

```
Исходный код           Компиляция           Выполнение
─────────────          ──────────           ──────────
HelloWorld.java  →  [javac]  →  HelloWorld.class  →  [JVM]  →  Результат
                              (bytecode)
```

> [!note] Почему байткод, а не машинный код?
> Байткод — платформонезависимый. JVM на каждой ОС (Windows, Linux, macOS) интерпретирует тот же `.class` файл. Это принцип **WORA**: Write Once, Run Anywhere.

```java
// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// В терминале:
// javac HelloWorld.java   → создаёт HelloWorld.class
// java HelloWorld         → JVM запускает байткод
```

---

## 🔹 JDK vs JRE vs JVM

```
JDK (Java Development Kit)
├── javac (компилятор)
├── jdb (отладчик)
├── jar (архиватор)
├── javadoc
└── JRE (Java Runtime Environment)
    ├── Стандартные библиотеки (java.lang, java.util, java.io...)
    └── JVM (Java Virtual Machine)
        ├── Class Loader
        ├── Bytecode Verifier
        ├── Interpreter
        ├── JIT Compiler
        └── Garbage Collector
```

| | Для чего | Содержит |
|-|----------|---------|
| **JVM** | Выполнение байткода | Интерпретатор, GC, JIT |
| **JRE** | Запуск Java программ | JVM + стандартные библиотеки |
| **JDK** | Разработка Java программ | JRE + javac + инструменты |

---

## 🔹 JIT-компилятор (Just-In-Time)

> [!note] Определение
> JIT — компилирует "горячие" (часто выполняемые) методы в нативный машинный код во время выполнения. Это делает Java быстрее простой интерпретации.

```
Первые запуски метода:   [Interpreter]  — интерпретация, медленно
После N вызовов:         [JIT Compiler] — компиляция в машинный код
Последующие вызовы:      [Нативный код] — быстро, как C/C++

Порог JIT по умолчанию: ~10 000 вызовов метода
```

---

## 🔹 Память JVM: Stack и Heap

```
┌─────────────────────────────────────────────────────────┐
│                        JVM Memory                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │   Stack      │  │   Stack      │  │      Heap        │ │
│  │  (Thread 1)  │  │  (Thread 2)  │  │  (общий для всех)│ │
│  │─────────────│  │─────────────│  │─────────────────│ │
│  │ frame: main │  │ frame: run  │  │  Object A        │ │
│  │  int x = 5  │  │  int i = 0  │  │  Object B        │ │
│  │  ref → A   ─┼──┼─────────────┼──┼→ Object C        │ │
│  │ frame: foo  │  │             │  │  String Pool     │ │
│  │  int y = 3  │  │             │  │  ...             │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘ │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐│
│  │                  Metaspace (Java 8+)                 ││
│  │  Class metadata, static поля, constant pool         ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Stack (Стек)
```java
// Каждый поток имеет свой стек
// При вызове метода — создаётся новый "фрейм" (stack frame)
// При выходе из метода — фрейм уничтожается

public static void main(String[] args) {           // frame: main
    int x = 42;          // примитив: хранится в стеке
    String s = "hello";  // ссылка в стеке, объект в heap
    Dog d = new Dog();   // ссылка в стеке, объект в heap
    foo(x);              // создаётся новый frame для foo
}

public static void foo(int n) {  // frame: foo
    int y = n * 2;       // y — в стеке (frame foo)
}                        // frame foo уничтожается
```

**Stack хранит:**
- Примитивные переменные (`int`, `double`, `boolean`...)
- Ссылки на объекты (но не сами объекты!)
- Параметры методов
- Стек вызовов (call stack)

### Heap (Куча)
```java
// Все объекты создаются в Heap
Dog dog = new Dog("Rex");  // объект Dog — в Heap
int[] arr = new int[100];  // массив — тоже объект, в Heap
String s = new String("Hello");  // в Heap (не в пуле)
```

**Heap хранит:**
- Все объекты (результат `new`)
- Массивы (они тоже объекты)
- Строки (не из String Pool)

| | Stack | Heap |
|-|-------|------|
| Владелец | Каждый поток свой | Один на всё приложение |
| Хранит | Примитивы, ссылки, вызовы | Объекты, массивы |
| Размер | Меньше (~1 МБ по умолчанию) | Больше (настраивается) |
| Скорость | Быстрее (LIFO) | Медленнее |
| Управление памятью | Автоматически (LIFO) | GC |
| Ошибка переполнения | `StackOverflowError` | `OutOfMemoryError` |

> [!warning] StackOverflowError
> Бесконечная рекурсия переполняет стек:
> ```java
> public static void infinite() {
>     infinite();  // каждый вызов — новый frame, стек переполнится
> }
> ```

---

## 🔹 Garbage Collector (GC)

> [!note] Определение
> GC — автоматически освобождает память объектов, на которые больше нет **достижимых ссылок**. Программист не управляет памятью вручную.

```
Объект "живой" (reachable):    есть хотя бы одна ссылка до него от "корней"
Объект "мёртвый" (unreachable): нет ни одной живой ссылки → GC удалит

"Корни GC" (GC Roots):
  - локальные переменные в стеках потоков
  - статические поля классов
  - JNI ссылки
```

```java
// Пример создания "мусора"
public void createGarbage() {
    Dog dog = new Dog("Rex");   // создан объект в Heap
    dog = new Dog("Buddy");     // dog теперь ссылается на другой объект
    // первый объект Dog("Rex") — никто на него не ссылается → кандидат для GC
}
// После выхода из метода — dog тоже исчезает из стека → Dog("Buddy") тоже мусор
```

### Поколения (Generational GC)
```
Heap
├── Young Generation (молодое поколение)
│   ├── Eden Space     — новые объекты создаются здесь
│   ├── Survivor S0    — выжившие после Minor GC
│   └── Survivor S1
└── Old Generation (старое поколение / Tenured)
    └── долгоживущие объекты, пережившие несколько Minor GC

Metaspace — метаданные классов (вне Heap, Java 8+)
```

```
Minor GC  — очищает Young Generation (быстро, часто)
Major GC  — очищает Old Generation (медленнее)
Full GC   — очищает всё (медленно, Stop-the-World)
```

### Типы GC в Java
| GC | Описание | Когда использовать |
|----|---------|-------------------|
| Serial GC | Один поток, паузы | Маленькие приложения |
| Parallel GC | Несколько потоков | Throughput (default до Java 8) |
| G1 GC | Предсказуемые паузы | Большой Heap (default Java 9+) |
| ZGC | Паузы < 10ms | Большой Heap, low-latency |
| Shenandoah | Паузы < 10ms | Аналог ZGC |

```java
// Попросить GC запуститься (не гарантия!)
System.gc();  // лишь "подсказка" JVM

// Настройка через JVM флаги:
// java -Xms256m -Xmx1g -XX:+UseG1GC MyApp
//       ↑ начальный heap  ↑ максимальный heap  ↑ использовать G1 GC
```

---

## 🔹 String Pool

> [!note] Определение
> String Pool — область в Heap, где JVM кеширует строковые литералы. Одинаковые литералы указывают на **один объект**.

```java
// Строковые литералы — берутся из пула
String s1 = "hello";  // создаётся в пуле
String s2 = "hello";  // возвращается существующий объект из пула
String s3 = "hello";  // тот же объект

System.out.println(s1 == s2);  // true (одна ссылка!)
System.out.println(s1 == s3);  // true

// new String — создаёт новый объект в Heap, НЕ в пуле
String s4 = new String("hello");  // новый объект в Heap
String s5 = new String("hello");  // ещё один объект в Heap

System.out.println(s1 == s4);      // false (разные объекты!)
System.out.println(s4 == s5);      // false
System.out.println(s4.equals(s5)); // true (содержимое одинаковое)

// intern() — поместить строку в пул
String s6 = s4.intern();  // вернуть строку из пула (или добавить)
System.out.println(s1 == s6);  // true
```

```
String Pool (в Heap):          Heap:
──────────────────             ────────────────────
"hello" ──────────────────→ s1, s2, s3
"world"                        s4 → [новый объект "hello"]
"java"                         s5 → [ещё один объект "hello"]
```

> [!warning] Подводные камни
> `==` сравнивает **ссылки**, а не содержимое.
> Для строк **всегда** используй `.equals()`:
> ```java
> // ❌ Ненадёжно
> if (input == "yes") { ... }
>
> // ✅ Правильно
> if ("yes".equals(input)) { ... }  // "yes".equals — защита от NPE если input null
> if (input != null && input.equals("yes")) { ... }
> ```

---

## 🔹 Autoboxing / Unboxing

> [!note] Определение
> Автоматическое преобразование примитивного типа в объект-обёртку (boxing) и обратно (unboxing). Нужно для использования примитивов в коллекциях и generics.

```
Примитив    Обёртка (Wrapper)
────────    ─────────────────
byte    ↔   Byte
short   ↔   Short
int     ↔   Integer
long    ↔   Long
float   ↔   Float
double  ↔   Double
char    ↔   Character
boolean ↔   Boolean
```

```java
// Autoboxing — int → Integer (автоматически)
Integer i = 42;          // компилятор делает: Integer.valueOf(42)
List<Integer> list = new ArrayList<>();
list.add(5);             // autoboxing: 5 → Integer.valueOf(5)

// Unboxing — Integer → int (автоматически)
int x = i;              // компилятор делает: i.intValue()
int sum = list.get(0) + 10;  // unboxing: Integer → int, затем +10

// Ручное преобразование
Integer wrapped = Integer.valueOf(100);
int primitive = wrapped.intValue();

// Полезные методы обёрток
Integer.parseInt("42");          // "42" → 42
Integer.toString(42);            // 42 → "42"
Integer.MAX_VALUE;               // 2147483647
Integer.MIN_VALUE;               // -2147483648
Integer.toBinaryString(10);      // "1010"
Integer.toHexString(255);        // "ff"
Double.parseDouble("3.14");      // "3.14" → 3.14
Boolean.parseBoolean("true");    // "true" → true
```

### Ловушка Integer кеша
```java
// Integer кеширует значения от -128 до 127
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (из кеша — один объект)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (не из кеша — разные объекты!)
System.out.println(c.equals(d));  // true (правильное сравнение)
```

> [!danger] Производительность
> Autoboxing в цикле — скрытая проблема производительности:
> ```java
> // ❌ Каждая итерация создаёт объект Long в Heap
> Long sum = 0L;
> for (long i = 0; i < 1_000_000; i++) {
>     sum += i;  // unboxing + boxing на каждой итерации!
> }
>
> // ✅ Используй примитив
> long sum = 0L;
> for (long i = 0; i < 1_000_000; i++) {
>     sum += i;
> }
> ```

---

## 🔹 Версии Java

| Версия | Год | Ключевые фичи |
|--------|-----|----------------|
| **Java 8** (LTS) | 2014 | Lambda, Stream API, Optional, default методы, новый Date/Time API |
| **Java 11** (LTS) | 2018 | `var` в лямбдах, `String.isBlank()`, `Files.readString()`, HTTP Client |
| **Java 17** (LTS) | 2021 | Sealed classes, Pattern Matching instanceof, Records (стабильно), Switch expressions |
| **Java 21** (LTS) | 2023 | Virtual Threads (Project Loom), Record Patterns, Sequenced Collections |

> [!note] LTS (Long Term Support)
> LTS-версии поддерживаются 8+ лет. В продакшне используй LTS: **8, 11, 17, 21**.

### var — локальный вывод типа (Java 10+)
```java
// Компилятор выводит тип из правой части
var list = new ArrayList<String>();    // тип: ArrayList<String>
var name = "Alice";                    // тип: String
var number = 42;                       // тип: int

// var только для локальных переменных
// Нельзя для полей класса, параметров методов, return типа
```

---

## 🔹 Загрузка классов (Class Loading)

```
Bootstrap ClassLoader   — загружает java.lang, java.util (из rt.jar / jmods)
     ↑
Extension ClassLoader   — загружает ext/*.jar
     ↑
Application ClassLoader — загружает classpath твоего приложения
     ↑
Custom ClassLoader      — можно написать свой

Принцип: сначала спрашивает родителя (Parent Delegation)
```

```java
// Загрузить класс по имени
Class<?> clazz = Class.forName("com.example.MyClass");

// Посмотреть загрузчик класса
System.out.println(String.class.getClassLoader());       // null (Bootstrap)
System.out.println(MyClass.class.getClassLoader());      // AppClassLoader
```

---

## 🔹 JVM флаги (для информации)

```bash
# Размер памяти
java -Xms512m -Xmx2g MyApp    # начальный heap 512МБ, максимум 2ГБ
java -Xss512k MyApp            # размер стека каждого потока

# GC
java -XX:+UseG1GC MyApp
java -XX:+UseZGC MyApp
java -verbose:gc MyApp         # логировать GC

# Отладка
java -XX:+HeapDumpOnOutOfMemoryError MyApp   # дамп при OOM
java -XX:HeapDumpPath=/tmp/dump.hprof MyApp
```

---

## 🔹 GC коллекторы — подробное сравнение

### Serial GC

```bash
java -XX:+UseSerialGC MyApp
```

Один поток для всех GC операций. Stop-the-World на всё время сборки.

```
Minor GC:  Eden → [STW один поток] → Survivor
Major GC:  Old Gen → [STW один поток] → компактизация
```

**Когда:** маленькие приложения (< 100MB heap), однопоточные среды, CLI утилиты.

---

### Parallel GC (Throughput Collector)

```bash
java -XX:+UseParallelGC MyApp
java -XX:ParallelGCThreads=4 MyApp  # число GC потоков
```

Несколько потоков для Minor GC и Full GC. Stop-the-World, но быстрее Serial.

```
Minor GC:  Eden → [STW N потоков параллельно] → Survivor
Full GC:   Old Gen → [STW N потоков] → компактизация
```

**Цель:** максимальный throughput (пропускная способность), паузы не критичны.
**Когда:** batch processing, аналитика, где важна общая скорость, а не latency.

---

### G1 GC (Garbage First) — default с Java 9

```bash
java -XX:+UseG1GC MyApp
java -XX:MaxGCPauseMillis=200 MyApp  # целевое время паузы (мс)
java -XX:G1HeapRegionSize=16m MyApp  # размер региона (1-32MB, степень двойки)
java -XX:G1NewSizePercent=20 MyApp   # min % heap для Young
java -XX:G1MaxNewSizePercent=60 MyApp # max % heap для Young
```

**Ключевая идея:** делит heap на равные **регионы** (~2048 штук). Регионы динамически назначаются под Eden, Survivor, Old, Humongous (большие объекты > 50% региона).

```
Heap разбит на регионы (~1-32MB каждый):

[E][E][S][O][O][H][E][O][S][E][O][H]...
 ↑   ↑   ↑   ↑           ↑
Eden Surv Old Hum      Humongous (большой объект)

G1 GC выбирает регионы с наибольшим количеством мусора (Garbage First)
→ сначала собирает самые "грязные" регионы для выполнения цели по паузе
```

**Фазы G1:**
```
Young GC (Minor):
  Eden + Survivor → [concurrent mark] → Survivor + Old
  STW, параллельные GC потоки

Mixed GC:
  Young + часть Old регионов → одна пауза
  Собирает Old регионы с наибольшим мусором

Full GC (fallback):
  Весь heap, single-thread → ❌ избегать!
  Происходит если G1 не успевает освобождать память
```

**Когда:** heap 4GB+, нужны предсказуемые паузы < 500ms, default для большинства приложений.

---

### ZGC — ultra-low latency (Java 15+ production)

```bash
java -XX:+UseZGC MyApp
java -XX:SoftMaxHeapSize=28g MyApp  # мягкий лимит (ZGC может пойти выше при нужде)
java -XX:ZCollectionInterval=0 MyApp # интервал между GC (0 = адаптивно)
```

**Ключевая идея:** почти всё делается **конкурентно** с работающими потоками приложения. STW паузы только для начальной и финальной разметки — занимают < 1ms независимо от размера heap.

```
Фазы ZGC:
  1. Pause Mark Start     [STW < 1ms] — разметить GC Roots
  2. Concurrent Mark      [concurrent] — обойти граф объектов
  3. Pause Mark End       [STW < 1ms] — финализировать разметку
  4. Concurrent Prepare   [concurrent] — подготовить релокацию
  5. Pause Relocate Start [STW < 1ms] — начать перемещение
  6. Concurrent Relocate  [concurrent] — переместить объекты

Всего STW: 3 паузы × < 1ms = < 3ms суммарно
```

**Как ZGC перемещает объекты конкурентно:**
Использует **load barriers** — специальный код встроенный в каждое чтение ссылки. При чтении "устаревшей" ссылки (объект уже перемещён) barrier автоматически обновляет её на новый адрес.

```
Объект A ссылается на B.
B перемещается в B' пока приложение работает.
Поток читает A.ref → barrier видит что B устарел → возвращает B'
```

**Когда:** heap 8GB-16TB (!), требование паузы < 10ms, real-time системы, trading, игры.

---

### Shenandoah GC (альтернатива ZGC)

```bash
java -XX:+UseShenandoahGC MyApp
```

Аналог ZGC от Red Hat. Тоже конкурентные паузы < 10ms. Отличие: использует **brooks pointers** (дополнительное поле в каждом объекте) вместо load barriers. Работает с Java 12+.

---

### Сравнительная таблица GC

| | Serial | Parallel | G1 | ZGC | Shenandoah |
|-|--------|----------|-----|-----|------------|
| STW паузы | Долгие | Долгие | Средние | **< 1ms** | **< 10ms** |
| Throughput | Низкий | **Высокий** | Хороший | Хороший | Хороший |
| Heap | < 4GB | < 4GB | 4-32GB | **До 16TB** | До 4TB |
| CPU overhead | Минимальный | Средний | Средний | Высокий | Высокий |
| Default с | — | Java 8 | **Java 9** | — | — |
| Когда | CLI, маленькие | Batch, throughput | **Большинство** | Ultra-low latency | Ultra-low latency |

---

## 🔹 Флаги настройки JVM — практика

### Память

```bash
# Heap
-Xms2g                    # начальный размер Heap (ms = minimum size)
-Xmx8g                    # максимальный размер Heap (mx = maximum size)
# Рекомендация: Xms == Xmx в продакшене — исключить GC на рост heap

# Stack
-Xss512k                  # размер стека каждого потока (default ~512KB-1MB)
# Уменьши если много потоков и OOM; увеличь при глубокой рекурсии

# Metaspace (классы, с Java 8)
-XX:MetaspaceSize=256m    # начальный размер Metaspace
-XX:MaxMetaspaceSize=512m # максимум (по умолчанию — не ограничен!)
# ВСЕГДА ограничивай MaxMetaspaceSize!

# G1 специфика
-XX:MaxGCPauseMillis=200  # целевое время паузы G1 (мс) — не гарантия, ориентир
-XX:G1HeapRegionSize=16m  # размер региона (1-32MB, должен быть степенью двойки)
```

### Логирование GC (Java 9+ Unified Logging)

```bash
# Базовое логирование
-Xlog:gc                               # только события GC
-Xlog:gc*                              # все GC сообщения (подробно)
-Xlog:gc:file=/logs/gc.log:time,uptime # писать в файл с временными метками

# Детальное логирование для анализа
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
#           ↑              ↑ декораторы            ↑ ротация: 5 файлов по 20MB

# Для Java 8 (старый формат)
-verbose:gc
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/logs/gc.log
```

### Диагностика и дамп памяти

```bash
# Дамп heap при OutOfMemoryError — ОБЯЗАТЕЛЬНО в продакшене!
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof   # куда писать дамп

# Статистика GC (для мониторинга)
-XX:+PrintGCApplicationStoppedTime    # время STW пауз
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintSafepointStatistics         # все причины пауз

# Для G1 — дополнительная информация
-XX:+UnlockExperimentalVMOptions
-XX:G1LogLevel=finest                  # Java 8, подробный лог G1
```

### Рекомендуемые настройки для продакшена

```bash
# Spring Boot / микросервис (2-4GB heap, G1)
java \
  -Xms2g -Xmx2g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=20m \
  -XX:MaxMetaspaceSize=256m \
  -jar app.jar

# High-throughput сервис (8GB heap, G1)
java \
  -Xms8g -Xmx8g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=500 \
  -XX:G1HeapRegionSize=16m \
  -XX:G1NewSizePercent=20 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=10,filesize=50m \
  -jar app.jar

# Low-latency сервис (16GB heap, ZGC)
java \
  -Xms16g -Xmx16g \
  -XX:+UseZGC \
  -XX:SoftMaxHeapSize=14g \
  -XX:+HeapDumpOnOutOfMemoryError \
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=20m \
  -jar app.jar
```

---

## 🔹 Чтение GC логов

### Unified Logging формат (Java 9+)

```
[2024-01-15T10:23:45.123+0000][0.521s][info][gc,start] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
[2024-01-15T10:23:45.156+0000][0.554s][info][gc      ] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 45M->12M(256M) 33.156ms
[2024-01-15T10:23:45.156+0000][0.554s][info][gc,cpu  ] GC(0) User=0.10s Sys=0.01s Real=0.03s

Разбор строки:
  GC(0)        → номер GC события
  Pause Young  → тип: Young GC (Minor GC)
  Normal       → причина: обычная (не allocation failure)
  45M->12M     → heap до → heap после
  (256M)       → общий размер heap
  33.156ms     → длительность паузы STW
  User/Sys/Real → CPU время
```

```
[gc] GC(5) Pause Full (G1 Compaction Pause) 1234M->456M(2048M) 4567.890ms
            ↑                                                    ↑
     Full GC — это ПЛОХО!                              4.5 секунды! Нужно разобраться
```

### Что искать в логах

```
✅ Нормально:
  Pause Young ... 10-50ms    → Minor GC, быстро
  Pause Young (Concurrent Start) → начало concurrent mark цикла G1

⚠️ Внимание:
  Pause Young ... 200-500ms  → пауза растёт, возможно heap мал
  Pause Remark / Cleanup      → фазы concurrent mark, небольшие паузы

❌ Проблема:
  Pause Full ... 1000ms+     → Full GC! приложение стояло секунды
  Pause Full (Allocation Failure) → heap исчерпан, G1 не успевает
  to-space exhausted          → Survivor переполнен, объекты уходят в Old раньше времени
```

### Инструменты анализа GC логов

```
GCEasy (gcease.io)     — онлайн анализатор, загрузи лог → получи отчёт
GCViewer              — десктопное приложение, графики пауз
JVM MXBeans           — программный доступ к GC статистике из кода
```

```java
// Программный мониторинг GC из кода
import java.lang.management.*;

List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
for (GarbageCollectorMXBean gc : gcBeans) {
    System.out.printf("GC: %-20s Collections: %d Time: %dms%n",
        gc.getName(),
        gc.getCollectionCount(),
        gc.getCollectionTime());
}
// Вывод:
// GC: G1 Young Generation  Collections: 1523  Time: 4823ms
// GC: G1 Old Generation    Collections: 3      Time: 156ms
```

---

## 🔹 Диагностика проблем с памятью

### OutOfMemoryError — виды и причины

```
java.lang.OutOfMemoryError: Java heap space
  → Heap заполнен: утечка памяти или heap слишком мал
  → Решение: анализ heap dump, увеличение -Xmx

java.lang.OutOfMemoryError: GC overhead limit exceeded
  → JVM тратит > 98% времени на GC, освобождает < 2% heap
  → Признак: heap почти полон, GC не помогает → утечка памяти

java.lang.OutOfMemoryError: Metaspace
  → ClassLoader leak: классы загружаются но не выгружаются
  → Решение: -XX:MaxMetaspaceSize, найти утекающий ClassLoader

java.lang.OutOfMemoryError: Direct buffer memory
  → NIO ByteBuffer.allocateDirect() исчерпал off-heap память
  → Решение: -XX:MaxDirectMemorySize=1g

java.lang.StackOverflowError
  → Глубокая рекурсия, циклические вызовы
  → Решение: рефакторинг на итерацию или -Xss увеличить стек
```

### Анализ heap dump

```bash
# Создать дамп вручную (приложение продолжает работать)
jmap -dump:live,format=b,file=/tmp/heap.hprof <PID>

# Или через jcmd (предпочтительнее)
jcmd <PID> GC.heap_dump /tmp/heap.hprof

# Просмотр дампа
jhat /tmp/heap.hprof                    # встроенный (устарел)
# Рекомендуется: Eclipse MAT (Memory Analyzer Tool) — GUI
# Рекомендуется: VisualVM — GUI, бесплатно
```

**Что искать в heap dump через Eclipse MAT:**
```
1. Dominator Tree — кто занимает больше всего памяти
2. Leak Suspects — MAT автоматически предлагает подозреваемых
3. OQL запросы — SELECT * FROM java.util.HashMap WHERE size > 10000

Типичные утечки:
  - HashMap/List в static полях — растут и никогда не очищаются
  - ThreadLocal не очищенный — особенно в thread pools
  - Listener / Observer не отписанный
  - Cache без TTL и eviction policy
```

### Мониторинг в реальном времени

```bash
# jstat — статистика GC в реальном времени
jstat -gc <PID> 1000          # каждую секунду
jstat -gcutil <PID> 1000      # в процентах

# Вывод jstat -gcutil:
#  S0     S1     E      O      M     CCS    YGC   YGCT   FGC   FGCT    GCT
#   0.00  47.76  36.87  65.34  97.54  95.05  2523   8.234    0   0.000   8.234
#   ↑Survivor0  ↑Eden  ↑Old   ↑Meta        ↑Young GC count  ↑Full GC count

# jcmd — универсальный инструмент
jcmd <PID> VM.flags                   # текущие JVM флаги
jcmd <PID> GC.run                     # запустить GC вручную
jcmd <PID> Thread.print               # дамп потоков
jcmd <PID> VM.native_memory           # native memory usage
jcmd <PID> VM.info                    # полная информация о JVM
```

---

## 🔹 GC Tuning — практические советы

```
Шаг 1: Измерь прежде чем менять
  Включи GC логи. Посмотри: сколько пауз, какой длины, как часто Full GC.
  Цель G1: Minor GC < 50ms, Full GC — никогда.

Шаг 2: Heap sizing
  Правило: heap должен быть в 2-3x больше live data set.
  Live data set = сколько памяти после Full GC.
  Пример: после Full GC занято 2GB → ставь -Xmx 4-6GB.

Шаг 3: MaxGCPauseMillis для G1
  Начни с 200ms. Если паузы всё равно большие — увеличь heap.
  Если heap большой но паузы растут — возможно много долгоживущих объектов.

Шаг 4: Избегать Full GC в G1
  Full GC причины:
    - Allocation failure: объекты создаются быстрее чем G1 успевает убирать
      → Увеличить heap или -XX:G1NewSizePercent
    - Humongous allocation: большие объекты > 50% размера региона
      → Увеличить -XX:G1HeapRegionSize

Шаг 5: Переход на ZGC если нужны паузы < 10ms
  ZGC даёт < 1ms паузы при любом размере heap.
  Но: больше CPU overhead (~10-15%), чуть ниже throughput.
  Когда: latency P99/P999 важнее throughput.
```

---

## 🔹 Итог

```
GC коллекторы:
  Serial    — один поток, долгие паузы. Только для маленьких приложений.
  Parallel  — много потоков, долгие паузы, высокий throughput. Batch.
  G1        — регионы, предсказуемые паузы (default Java 9+). Большинство.
  ZGC       — конкурентный, паузы < 1ms, любой heap. Ultra-low latency.

Флаги памяти:
  -Xms/-Xmx     — heap. В продакшене: Xms == Xmx.
  -Xss           — стек потока.
  -XX:MaxMetaspaceSize — ВСЕГДА ограничивай!
  -XX:MaxGCPauseMillis — целевая пауза для G1.

GC логирование (Java 9+):
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=20m

HeapDump: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
Анализ: Eclipse MAT, VisualVM, jmap, jstat, jcmd

OOM виды:
  Java heap space         → утечка или мал heap
  GC overhead exceeded    → почти OOM, GC не справляется
  Metaspace               → ClassLoader leak
  Direct buffer memory    → off-heap утечка

GC tuning алгоритм:
  1. Включи GC логи
  2. Определи live data set (размер heap после Full GC)
  3. Heap = 2-3x live data set, Xms == Xmx
  4. Для G1: MaxGCPauseMillis=200, избегай Full GC
  5. Нужны паузы < 10ms → переходи на ZGC
```
