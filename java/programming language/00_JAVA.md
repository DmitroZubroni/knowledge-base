# ☕ Java — Главная карта тем

> **Теги:** #java #hub #programming  

> [!abstract] Навигация
> [[main]] | [[main Java]]

---

## 🗺️ Карта файлов

| Файл                   | Тема                            | Ключевые концепции                                                            |
| ---------------------- | ------------------------------- | ----------------------------------------------------------------------------- |
| [[11_JVM_Internals]]   | JVM и устройство Java           | bytecode, Stack/Heap, GC, String Pool                                         |
| [[01_Basics]]          | Синтаксис и основы              | типы, операторы, строки, циклы, массивы, методы                               |
| [[02_OOP_Core]]        | ООП: классы и объекты           | class, object, constructor, this, encapsulation                               |
| [[03_OOP_Advanced]]    | Наследование и полиморфизм      | extends, interface, abstract, @Override                                       |
| [[04_OOP_Principles]]  | Пакеты, static, final, enum     | package, import, static, final, enum                                          |
| [[05_Exceptions]]      | Исключения                      | try/catch/finally, checked/unchecked, custom                                  |
| [[06_Collections]]     | Коллекции                       | List, Set, Map, Queue                                                         |
| [[07_Generics]]        | Обобщения                       | Generic классы, wildcard, type erasure                                        |
| [[08_Functional]]      | Функциональное программирование | Lambda, Stream API, Optional                                                  |
| [[09_Multithreading]]  | Многопоточность                 | Thread, synchronized, volatile, AtomicInteger                                 |
| [[10_SOLID]]           | Принципы SOLID                  | S, O, L, I, D с примерами                                                     |
| [[12_Design_Patterns]] | Паттерны GoF отдельно от SOLID: | Singleton, Factory, Builder, Strategy, Observer, Repository (идея, не Spring) |

---

## 📚 Рекомендуемый порядок изучения

```
[[11_JVM_Internals]]
    ↓
[[01_Basics]]
    ↓
[[02_OOP_Core]]
    ↓
[[03_OOP_Advanced]]
    ↓
[[04_OOP_Principles]]
    ↓
[[05_Exceptions]]
    ↓
[[06_Collections]]
    ↓
[[07_Generics]]
    ↓
[[08_Functional]]
    ↓
[[09_Multithreading]]
    ↓
[[10_SOLID]]
    ↓
[[12_Design_Patterns]]
```

---

## 📖 Глоссарий

| Термин | Определение |
|--------|-------------|
| **JVM** | Java Virtual Machine — интерпретирует bytecode |
| **JDK** | Java Development Kit = JRE + javac + инструменты |
| **JRE** | Java Runtime Environment = JVM + стандартные библиотеки |
| **Bytecode** | Промежуточный код `.class`, создаётся компилятором javac |
| **Class** | Шаблон/чертёж для создания объектов |
| **Object** | Экземпляр класса, размещается в Heap |
| **Heap** | Область памяти JVM для объектов |
| **Stack** | Область памяти JVM для примитивов, ссылок, вызовов методов |
| **GC** | Garbage Collector — автоматически освобождает память |
| **Encapsulation** | Сокрытие данных: поля private + методы public |
| **Inheritance** | Наследование — extends, передача свойств родителя |
| **Polymorphism** | Один интерфейс — разные реализации |
| **Interface** | Контракт поведения, только абстрактные методы (до Java 8) |
| **Abstract class** | Класс с частичной реализацией, нельзя создать экземпляр |
| **Checked Exception** | Исключение, обязательное для обработки (IOException) |
| **Unchecked Exception** | Исключение не требующее обработки (RuntimeException) |
| **Generic** | Параметризованный тип — `List<String>` |
| **Lambda** | Анонимная функция: `(x) -> x * 2` |
| **Stream** | Конвейер операций над коллекциями |
| **Thread** | Поток выполнения, легковесный процесс |
| **Deadlock** | Взаимная блокировка двух потоков |
| **Race condition** | Несколько потоков меняют данные одновременно |
| **SOLID** | 5 принципов объектно-ориентированного дизайна |
| **Autoboxing** | Автоматическое преобразование `int` → `Integer` |
| **String Pool** | Пул строковых литералов в Heap |
| **Immutable** | Неизменяемый объект (String, Integer) |
| **Overloading** | Перегрузка: одно имя метода, разные параметры |
| **Overriding** | Переопределение метода родителя в подклассе |
| **this** | Ссылка на текущий объект |
| **super** | Ссылка на родительский класс |
| **final** | Константа / нельзя наследовать / нельзя переопределить |
| **static** | Принадлежит классу, не объекту |
| **enum** | Перечисление именованных констант |
| **Optional** | Контейнер для значения, которое может быть null |
| **volatile** | Гарантирует видимость значения переменной между потоками |
| **synchronized** | Блокировка для предотвращения race condition |
| **Type Erasure** | Стирание информации о Generic-типе в runtime |
| **Wildcard** | `<?>` — неизвестный тип в Generics |

---

## 🔑 Ключевые аннотации и ключевые слова

```
Аннотации:        @Override  @FunctionalInterface  @Deprecated  @SuppressWarnings
Доступ:           public  private  protected  (package-private)
Модификаторы:     static  final  abstract  synchronized  volatile  transient
Классы:           class  abstract class  interface  enum  record
Наследование:     extends  implements  super  this
Исключения:       try  catch  finally  throw  throws
Типы:             byte short int long float double char boolean  +  String
```

---

## 📌 Нотации именования

| Стиль | Применение | Пример |
|-------|-----------|--------|
| `PascalCase` | Классы, интерфейсы, enum | `BankAccount`, `Runnable` |
| `camelCase` | Переменные, методы | `userName`, `getBalance()` |
| `UPPER_SNAKE_CASE` | Константы (final static) | `MAX_SIZE`, `PI` |
| `lowercase` | Пакеты | `com.example.myapp` |
