> **Теги:** #java #concurrency #thread #lifecycle #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[JAVA]] | [[Concurrency_Index]]

# 02 — Жизненный цикл потока

---

## 🔹 Зачем знать состояния потока

Когда дебажишь зависшее приложение или анализируешь thread dump — нужно понимать, в каком состоянии "застрял" поток и почему. Например, состояние `BLOCKED` означает поток ждёт освобождения монитора (другой поток держит `synchronized`), а `WAITING` означает поток сам решил подождать (`wait()`, `join()`).

Понимание жизненного цикла — это ключ к диагностике deadlock'ов и зависаний.

---

## 🔹 Шесть состояний потока

Java определяет состояния потока через enum `Thread.State`:

```
NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
                       ↑___________________________|
                       (поток может возвращаться в RUNNABLE)
```

```
┌─────────┐  start()   ┌───────────┐
│   NEW   │ ─────────► │  RUNNABLE │ ◄──────────────┐
└─────────┘            └─────┬─────┘                │
                              │                      │
              ┌───────────────┼───────────────┐     │
              ▼                ▼               ▼     │
        ┌──────────┐   ┌──────────┐   ┌──────────────┐
        │ BLOCKED  │   │ WAITING  │   │ TIMED_WAITING│
        └────┬─────┘   └────┬─────┘   └──────┬───────┘
             │              │                  │
             └──────────────┴──────────────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │ TERMINATED  │
                       └─────────────┘
```

### NEW

Поток создан (`new Thread(...)`), но `start()` ещё не вызван. Поток существует как объект Java, но операционная система о нём ничего не знает.

```java
Thread t = new Thread(() -> {});
System.out.println(t.getState()); // NEW
```

### RUNNABLE

Поток либо **выполняется** прямо сейчас, либо **готов к выполнению** и ждёт своей очереди от планировщика OS. Java не различает "выполняется" и "готов" — оба случая это RUNNABLE.

```java
t.start();
System.out.println(t.getState()); // RUNNABLE (даже если процессор сейчас занят другим потоком)
```

### BLOCKED

Поток пытается войти в `synchronized` блок/метод, но монитор уже захвачен другим потоком. Поток ждёт, пока монитор освободится.

```java
synchronized (lock) {
    // другой поток, попытавшийся сюда зайти, будет в состоянии BLOCKED
}
```

### WAITING

Поток ждёт **бесконечно**, пока другой поток не совершит определённое действие. Переходит сюда через:
- `Object.wait()` — без таймаута
- `Thread.join()` — без таймаута
- `LockSupport.park()`

```java
synchronized (lock) {
    lock.wait(); // поток уходит в WAITING, пока кто-то не вызовет lock.notify()
}
```

### TIMED_WAITING

Как WAITING, но с ограничением по времени. Переходит сюда через:
- `Thread.sleep(ms)`
- `Object.wait(ms)`
- `Thread.join(ms)`
- `LockSupport.parkNanos()`

```java
Thread.sleep(1000); // TIMED_WAITING на 1 секунду, потом сам вернётся в RUNNABLE
```

### TERMINATED

Поток завершил выполнение `run()` — либо нормально, либо из-за необработанного исключения. Это финальное состояние, обратно в RUNNABLE поток не вернётся.

```java
t.join(); // ждём завершения
System.out.println(t.getState()); // TERMINATED
```

> [!warning] Поток нельзя перезапустить
> Вызов `start()` на потоке в состоянии TERMINATED бросит `IllegalThreadStateException`. Если нужно повторить задачу — создавай новый объект Thread.

---

## 🔹 sleep() — приостановка потока

`Thread.sleep(ms)` приостанавливает **текущий** поток на указанное время. Поток переходит в `TIMED_WAITING` и **не отдаёт** захваченные блокировки.

```java
synchronized (lock) {
    Thread.sleep(1000); // другие потоки НЕ смогут зайти в synchronized(lock),
                         // даже пока этот поток спит!
}
```

```java
public void printWithDelay() {
    for (int i = 1; i <= 5; i++) {
        System.out.println(i);
        try {
            Thread.sleep(500); // пауза 500мс между выводами
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // см. ниже про interrupt
            return;
        }
    }
}
```

> [!note] Почему `sleep()` бросает `InterruptedException`
> Если другой поток вызовет `interrupt()` на спящем потоке — `sleep()` немедленно прервётся с исключением `InterruptedException`. Это механизм "вежливой" остановки потока.

---

## 🔹 join() — ждать завершения другого потока

`thread.join()` заставляет **текущий** поток подождать, пока `thread` не завершится (перейдёт в `TERMINATED`).

```java
Thread worker = new Thread(() -> {
    System.out.println("Working...");
    try { Thread.sleep(2000); } catch (InterruptedException e) {}
    System.out.println("Done!");
});

worker.start();
System.out.println("Waiting for worker...");
worker.join(); // main поток блокируется здесь до завершения worker
System.out.println("Worker finished, continuing main");

// Вывод гарантированно:
// Waiting for worker...
// Working...
// Done!
// Worker finished, continuing main
```

Без `join()` главный поток мог бы завершиться раньше, чем worker допишет вывод.

```java
worker.join(1000); // ждать максимум 1 секунду, потом продолжить независимо
```

---

## 🔹 interrupt() — как правильно остановить поток

В Java **нет** безопасного способа форсированно убить поток (метод `Thread.stop()` устарел и опасен — может оставить общие данные в несогласованном состоянии). Вместо этого используется **кооперативный механизм** — interrupt-флаг.

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // выполняем работу, периодически проверяя флаг
        System.out.println("Working...");
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            System.out.println("Interrupted during sleep!");
            Thread.currentThread().interrupt(); // восстанавливаем флаг
            break; // выходим из цикла
        }
    }
    System.out.println("Worker stopped gracefully");
});

worker.start();
Thread.sleep(2000);
worker.interrupt(); // просим поток остановиться
```

Как это работает:
1. `interrupt()` устанавливает внутренний флаг потока в `true`
2. Если поток в этот момент в `sleep()`, `wait()` или `join()` — он немедленно проснётся с `InterruptedException`
3. Если поток выполняет обычный код — флаг просто устанавливается, поток сам должен периодически проверять `isInterrupted()`

> [!warning] Никогда не "съедай" InterruptedException молча
> ```java
> // ❌ Плохо — поток продолжит работу, как будто ничего не произошло
> try {
>     Thread.sleep(1000);
> } catch (InterruptedException e) {
>     // пусто — флаг прерывания потерян навсегда
> }
>
> // ✅ Хорошо — восстанавливаем флаг для вышестоящего кода
> try {
>     Thread.sleep(1000);
> } catch (InterruptedException e) {
>     Thread.currentThread().interrupt(); // восстановить флаг
>     return; // или break, или throw
> }
> ```

---

## 🔹 Приоритеты потоков

Java позволяет задать приоритет потока от 1 (`MIN_PRIORITY`) до 10 (`MAX_PRIORITY`), по умолчанию 5 (`NORM_PRIORITY`).

```java
Thread t = new Thread(() -> {});
t.setPriority(Thread.MAX_PRIORITY); // 10
```

> [!warning] Приоритеты — это лишь подсказка для OS
> Реальное поведение зависит от операционной системы и не гарантировано. На практике приоритеты редко используются — лучше управлять через `ExecutorService` и логику приложения.

---

## 🔹 Итог

```
Состояния потока:
  NEW            — создан, start() не вызван
  RUNNABLE       — выполняется или готов к выполнению
  BLOCKED        — ждёт освобождения synchronized монитора
  WAITING        — ждёт бесконечно (wait/join без таймаута)
  TIMED_WAITING  — ждёт с таймаутом (sleep/wait(ms)/join(ms))
  TERMINATED     — завершён, нельзя перезапустить

sleep(ms)  — пауза текущего потока, БЛОКИРОВКИ НЕ ОТДАЁТ
join()     — ждать завершения другого потока
join(ms)   — ждать с таймаутом

interrupt() — кооперативная остановка:
  - устанавливает флаг
  - "пробуждает" поток из sleep/wait/join с InterruptedException
  - НИКОГДА не глуши InterruptedException молча

Thread.stop() — устарел, не используй
Приоритеты — ненадёжная подсказка для OS, не основной механизм управления
```
