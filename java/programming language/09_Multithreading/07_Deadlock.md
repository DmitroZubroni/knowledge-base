> **Теги:** #java #concurrency #deadlock #livelock #starvation #конспект
> [!abstract] Связи
> [[main]] | [[main Java]] | [[Concurrency_Index]] | [[04_Synchronized]]

# 07 — Deadlock, Livelock, Starvation

---

## 🔹 Deadlock — взаимная блокировка

Deadlock — это ситуация когда два или больше потока **навечно ждут друг друга**. Каждый держит ресурс, который нужен другому, и никто не хочет отпускать свой, пока не получит чужой.

Классическая аналогия: два человека на узкой тропинке. Каждый ждёт, что другой уступит дорогу. Никто не уступает — оба стоят вечно.

### Минимальный пример

```java
Object lockA = new Object();
Object lockB = new Object();

Thread thread1 = new Thread(() -> {
    synchronized (lockA) {
        System.out.println("Thread1: держит lockA, ждёт lockB");
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        synchronized (lockB) { // ← ждёт, пока Thread2 отпустит lockB
            System.out.println("Thread1: держит оба лока");
        }
    }
});

Thread thread2 = new Thread(() -> {
    synchronized (lockB) {
        System.out.println("Thread2: держит lockB, ждёт lockA");
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        synchronized (lockA) { // ← ждёт, пока Thread1 отпустит lockA
            System.out.println("Thread2: держит оба лока");
        }
    }
});

thread1.start();
thread2.start();
// Программа зависнет навечно. Последние два println никогда не выполнятся.
```

```
Thread1:  ───[держит lockA]────────────────────────[ждёт lockB]──►
Thread2:  ───────────────[держит lockB]──[ждёт lockA]──────────────►
                                          ↑
                                     оба ждут вечно
```

---

## 🔹 Четыре условия Coffman — когда возникает deadlock

Deadlock невозможен, если хотя бы одно из четырёх условий не выполняется:

1. **Mutual Exclusion** — ресурс может держать только один поток одновременно (свойство монитора — убрать нельзя)
2. **Hold and Wait** — поток держит один ресурс и ждёт другой. *Можно нарушить: захватывать все нужные ресурсы сразу или отпускать текущий перед захватом нового*
3. **No Preemption** — ресурс нельзя отобрать принудительно, только освободить добровольно. *Можно нарушить через ReentrantLock.tryLock() с таймаутом*
4. **Circular Wait** — есть цикл ожидания: A ждёт B, B ждёт C, C ждёт A. *Самое простое для нарушения: упорядочить захват локов*

---

## 🔹 Способ 1: фиксированный порядок захвата локов

Самое простое и эффективное решение — **всегда захватывать локи в одном и том же порядке** во всех потоках. Если все потоки сначала берут lockA, потом lockB — circular wait невозможен.

```java
// ✅ Оба потока захватывают в одном порядке: A → B
Thread thread1 = new Thread(() -> {
    synchronized (lockA) {
        synchronized (lockB) {
            System.out.println("Thread1: работает");
        }
    }
});

Thread thread2 = new Thread(() -> {
    synchronized (lockA) { // ← тот же порядок, не B → A
        synchronized (lockB) {
            System.out.println("Thread2: работает");
        }
    }
});
// Deadlock невозможен: Thread2 дождётся, пока Thread1 отпустит lockA
```

Для динамических ситуаций (когда нельзя жёстко прописать порядок в коде) — используй системный идентификатор объекта:

```java
// Упорядочиваем по System.identityHashCode()
void transferMoney(Account from, Account to, double amount) {
    Object firstLock  = System.identityHashCode(from) < System.identityHashCode(to) ? from : to;
    Object secondLock = System.identityHashCode(from) < System.identityHashCode(to) ? to : from;

    synchronized (firstLock) {
        synchronized (secondLock) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

---

## 🔹 Способ 2: tryLock() с таймаутом (ReentrantLock)

`synchronized` нельзя "отменить" — поток либо получает монитор, либо ждёт вечно. `ReentrantLock.tryLock(timeout)` позволяет попробовать захватить лок с таймаутом и **отступить**, если не получилось:

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

ReentrantLock lockA = new ReentrantLock();
ReentrantLock lockB = new ReentrantLock();

void doWork() throws InterruptedException {
    while (true) {
        boolean gotA = lockA.tryLock(100, TimeUnit.MILLISECONDS);
        if (!gotA) continue; // не получили lockA — повторяем

        try {
            boolean gotB = lockB.tryLock(100, TimeUnit.MILLISECONDS);
            if (!gotB) continue; // не получили lockB — отпускаем lockA и повторяем

            try {
                // держим оба лока — работаем
                System.out.println("Working with both locks");
                return; // успешно завершили
            } finally {
                lockB.unlock();
            }
        } finally {
            if (!gotB) lockA.unlock(); // важно: отпустить lockA если не взяли lockB
            // На самом деле логику tryLock лучше оформить аккуратнее — см. ниже
        }
    }
}
```

Более чистый паттерн с tryLock:

```java
void transfer(Account from, Account to, double amount) throws InterruptedException {
    while (true) {
        if (from.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return; // успех
                    } finally {
                        to.lock.unlock();
                    }
                }
            } finally {
                from.lock.unlock();
            }
        }
        // Оба лока не захвачены — небольшая пауза перед повтором (backoff)
        Thread.sleep(10);
    }
}
```

---

## 🔹 Диагностика deadlock — Thread Dump

Когда приложение зависло, первым делом делают **thread dump** — снимок состояния всех потоков:

```bash
# Узнать PID Java процесса
jps -l

# Получить thread dump
jstack <PID>

# Или послать сигнал (Linux/Mac)
kill -3 <PID>
```

В thread dump deadlock будет явно обозначен:

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x000..., a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00007f... (object 0x000..., a java.lang.Object),
  which is held by "Thread-1"
```

---

## 🔹 Livelock — движение без прогресса

Livelock похож на deadlock, но потоки **не заблокированы** — они активно выполняют код, но не продвигаются к результату. Аналогия: два воспитанных человека в дверях — каждый пытается уступить другому, и оба постоянно меняют направление.

```java
// Потоки постоянно "уступают" друг другу и никто не работает
class Polite {
    volatile boolean needHelp = true;

    void help(Polite other) {
        while (needHelp) {
            if (other.needHelp) {
                System.out.println(Thread.currentThread().getName() + ": пропускаю тебя");
                Thread.yield(); // уступаю процессор
                continue;       // но проблема не решается
            }
            // Никогда не достигаем этой точки — оба вечно "уступают"
            needHelp = false;
        }
    }
}
```

**Решение**: добавить случайный backoff — ждать случайное время перед повторной попыткой. Это нарушает "синхронность" действий:

```java
Thread.sleep(new Random().nextInt(100)); // случайная пауза 0-100мс
```

---

## 🔹 Starvation — голодание потока

Starvation — ситуация когда поток **никогда не получает доступ** к ресурсу, потому что другие потоки постоянно его перехватывают.

Пример: читатели-писатели. Если читателей много и они постоянно держат ReadWriteLock на чтение — поток-писатель может ждать бесконечно, всегда "вытесняемый" новыми читателями.

```
Читатель 1: ══════════╗
Читатель 2:    ═══════╬═════════╗
Читатель 3:        ═══╬═════════╬════════╗
Писатель:   ожидает...╠═════════╬════════╬══... вечно
```

**Решение**: использовать **честные (fair) блокировки** — они гарантируют, что потоки получают доступ в порядке очереди, никто не может постоянно "обгонять":

```java
ReentrantLock fairLock = new ReentrantLock(true); // true = fair mode
```

> [!warning] Fair lock — дороже обычного
> Fair mode снижает пропускную способность, потому что вместо "кто первый успел" используется строгая очередь. Используй только когда starvation действительно проблема.

---

## 🔹 Сравнение трёх проблем

| | Deadlock | Livelock | Starvation |
|-|---------|----------|------------|
| Состояние потока | BLOCKED | RUNNABLE (активен) | BLOCKED / постоянно вытесняется |
| Прогресс | Нет ни у кого | Нет ни у кого | Нет у конкретного потока |
| Видно в thread dump | Да (явно) | Сложно (потоки активны) | Сложно |
| Решение | Порядок локов / tryLock | Random backoff | Fair locks |

---

## 🔹 Итог

```
Deadlock = потоки навечно ждут друг друга
  Условия Coffman (нужны все 4): mutual exclusion, hold&wait, no preemption, circular wait

  Решения:
    1. Фиксированный порядок захвата локов (нарушает circular wait)
    2. tryLock(timeout) — отступить если не захватил за время
    3. Архитектурно — избегать вложенных synchronized

  Диагностика: jstack <PID> → thread dump → секция "Found Java-level deadlock"

Livelock = потоки активны, но не продвигаются
  Решение: random backoff (Thread.sleep(random))

Starvation = один поток никогда не получает ресурс
  Решение: ReentrantLock(true) — fair mode

Правило: никогда не захватывай два лока в разном порядке в разных местах кода.
```
