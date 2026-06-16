> **Теги:** #java #concurrency #multithreading #hub
> [!abstract] Навигация
> [[main]] | [[main Java]] | [[JAVA]]

# ⚡ Concurrency — Многопоточность

---

## 🗺️ Файлы раздела

| Файл | Тема | Ключевые концепции |
|------|------|--------------------|
| [[01_Process_Thread]] | Процессы и потоки | JVM, heap/stack, зачем многопоточность |
| [[02_Thread_Lifecycle]] | Жизненный цикл потока | состояния, sleep/join/interrupt |
| [[03_Race_Condition]] | Гонка данных | visibility, atomicity, happens-before |
| [[04_Synchronized]] | Монитор и synchronized | блок, метод, static, подводные камни |
| [[05_Volatile]] | Volatile и JMM | кэши CPU, когда хватает, когда нет |
| [[06_Atomic]] | Атомарные типы | CAS, AtomicInteger, AtomicReference |
| [[07_Deadlock]] | Deadlock и другие проблемы | deadlock, livelock, starvation |
| [[08_ExecutorService]] | Пул потоков | ThreadPool, виды пулов, shutdown |
| [[09_Future_Callable]] | Асинхронность | Callable, Future, CompletableFuture |
| [[10_Concurrent_Collections]] | Потокобезопасные коллекции | ConcurrentHashMap, BlockingQueue |
| [[11_Locks]] | Явные блокировки | ReentrantLock, ReadWriteLock, StampedLock |

---

## 📚 Порядок изучения

```
[[01_Process_Thread]]
    ↓
[[02_Thread_Lifecycle]]
    ↓
[[03_Race_Condition]]       ← сначала понять проблему
    ↓
[[04_Synchronized]]         ← первое решение
    ↓
[[05_Volatile]]             ← частный случай видимости
    ↓
[[06_Atomic]]               ← альтернатива synchronized для счётчиков
    ↓
[[07_Deadlock]]             ← что идёт не так
    ↓
[[08_ExecutorService]]      ← правильный способ запускать задачи
    ↓
[[09_Future_Callable]]      ← асинхронные результаты
    ↓
[[10_Concurrent_Collections]]
    ↓
[[11_Locks]]                ← продвинутый уровень
```
