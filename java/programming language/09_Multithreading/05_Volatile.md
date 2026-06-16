> **Теги:** #java #concurrency #volatile #jmm #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Concurrency_Index]] | [[03_Race_Condition]] | [[04_Synchronized]]

# 05 — volatile

---

## 🔹 Когда synchronized — избыточен

В [[03_Race_Condition]] мы разобрали два разных аспекта: **atomicity** (операция должна быть неделимой) и **visibility** (изменение должно быть видно другим потокам).

`synchronized` решает обе проблемы сразу, но иногда нужна **только visibility**, без atomicity. Классический пример — флаг остановки:

```java
class Worker {
    private boolean running = true;

    public void stop() {
        running = false; // простая запись — атомарна сама по себе
    }

    public void run() {
        while (running) { // простое чтение — атомарно само по себе
            // работа
        }
    }
}
```

Запись и чтение `boolean` — атомарны (для всех примитивов кроме `long` и `double` на некоторых платформах). Atomicity тут не проблема. Проблема — **visibility**: поток, выполняющий `run()`, может никогда не увидеть, что `running` стал `false`, из-за кэширования в L1-кэше своего ядра CPU.

Использовать `synchronized` для такого простого случая — избыточно: это создаёт монитор, блокировки, потенциальные задержки — всё ради одного булева флага. Здесь нужен `volatile`.

---

## 🔹 Что делает volatile

`volatile` — модификатор поля, который даёт **две гарантии**:

1. **Visibility** — каждая запись в volatile-поле немедленно видна всем потокам. Каждое чтение берёт самое актуальное значение из основной памяти, минуя кэши CPU.
2. **Запрет переупорядочивания** — компилятор и процессор не могут переставлять операции с volatile-полем относительно других операций (частичная упорядоченность по happens-before).

```java
class Worker {
    private volatile boolean running = true;

    public void stop() {
        running = false; // запись сразу видна всем потокам
    }

    public void run() {
        while (running) { // всегда читает актуальное значение
            // работа
        }
        System.out.println("Stopped!");
    }
}
```

```
Без volatile:                     С volatile:

Core 1 (Поток A):                 Core 1 (Поток A):
  running = false  ──┐              running = false ───────┐
  (записано в кэш L1) │              (немедленно в main mem) │
                       │                                      │
Core 2 (Поток B):      │            Core 2 (Поток B):        │
  while (running) {    │              while (running) {      │
    // running=true    │ ← не видит    // читает свежее ◄────┘
    // вечный цикл!     │   изменения   // выходит из цикла
  }                    ─┘
```

---

## 🔹 Почему volatile НЕ решает проблему atomicity

Самая частая ошибка — использовать `volatile` там, где нужна атомарность составной операции:

```java
class Counter {
    private volatile int count = 0;

    public void increment() {
        count++; // ❌ ВСЁ ЕЩЁ НЕ АТОМАРНО! volatile тут не помогает
    }
}
```

`count++` всё ещё состоит из трёх шагов (READ-ADD-WRITE). `volatile` гарантирует, что каждый отдельный READ и каждый отдельный WRITE будут видны всем потокам **немедленно** — но не гарантирует, что между READ и WRITE другой поток не вмешается.

```
Поток A                          Поток B
─────────────────────────────────────────────────
READ count → 5  (видят оба сразу, т.к. volatile)
                                  READ count → 5
ADD 5+1 = 6
WRITE count = 6  (видно сразу)
                                  ADD 5+1 = 6
                                  WRITE count = 6

Результат: count = 6, должно быть 7. Гонка осталась!
```

> [!warning] volatile = только visibility, НЕ atomicity
> Для составных операций (`x++`, `x += y`, `if (x == null) x = new Foo()`) — `volatile` не спасёт. Нужны `synchronized` или `Atomic*` классы (см. [[06_Atomic]]).

---

## 🔹 Когда volatile достаточно — правильные сценарии

### 1. Флаг остановки/состояния

```java
private volatile boolean shutdownRequested = false;

public void shutdown() {
    shutdownRequested = true; // простая запись
}

public void run() {
    while (!shutdownRequested) { // простое чтение
        doWork();
    }
}
```

### 2. Публикация неизменяемого объекта (immutable)

```java
class Config {
    private volatile ImmutableSettings settings;

    public void updateSettings(ImmutableSettings newSettings) {
        settings = newSettings; // атомарная замена ссылки + видимость
    }

    public ImmutableSettings getSettings() {
        return settings; // всегда видит актуальную ссылку
    }
}
```

Здесь `volatile` гарантирует, что когда поток читает `settings`, он видит **полностью сконструированный** объект `ImmutableSettings`, а не "наполовину инициализированный" (это связано с запретом переупорядочивания при конструировании объекта).

### 3. Double-Checked Locking (классический паттерн Singleton)

```java
class Singleton {
    private static volatile Singleton instance; // volatile ОБЯЗАТЕЛЕН здесь

    public static Singleton getInstance() {
        if (instance == null) {                    // первая проверка (без лока — быстро)
            synchronized (Singleton.class) {
                if (instance == null) {            // вторая проверка (внутри лока)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

> [!note] Почему здесь нужен volatile
> `new Singleton()` — это не одна атомарная операция. Внутри происходит: (1) выделение памяти, (2) вызов конструктора, (3) присвоение ссылки переменной `instance`. Без `volatile` компилятор/процессор могут **переставить шаги 2 и 3** — другой поток может увидеть `instance != null`, но получить доступ к **ещё не инициализированному** объекту. `volatile` запрещает такую перестановку.

---

## 🔹 volatile vs synchronized — сравнение

| | volatile | synchronized |
|-|----------|---------------|
| Visibility | ✅ | ✅ |
| Atomicity составных операций | ❌ | ✅ |
| Блокирует другие потоки | Нет | Да (BLOCKED) |
| Производительность | Быстрее | Дороже (захват монитора) |
| Применимо к | Полям | Блокам кода / методам |
| Реентерабельность | Не применимо | Да |

> [!tip] Правило выбора
> Простое чтение/запись одного поля, где видимость — единственная проблема → `volatile`.
> Составные операции (инкремент, условная замена, несколько связанных полей) → `synchronized` или `Atomic*`.

---

## 🔹 Итог

```
volatile = гарантия visibility + запрет переупорядочивания
           НЕ гарантирует atomicity составных операций

Используй когда:
  - простой флаг состояния (running, shutdownRequested)
  - публикация ссылки на immutable объект
  - Double-Checked Locking (instance в Singleton)

НЕ используй для:
  - count++ (составная операция — нужен synchronized или AtomicInteger)
  - x += y
  - lazy init без правильного паттерна

count++ с volatile — ВСЁ ЕЩЁ НЕ ПОТОКОБЕЗОПАСНО

Сравнение:
  volatile      → быстрее, но только видимость
  synchronized  → дороже, но видимость + атомарность + взаимное исключение
```
