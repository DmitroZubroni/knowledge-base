> **Теги:** #java #collections #hub #layer-mid
> [!abstract] Навигация
> [[main]] | [[main Java]] | [[JAVA]]

# 📦 Collections — Углублённый раздел

---

## 🗺️ Файлы раздела

| Файл | Тема | Ключевые концепции |
|------|------|--------------------|
| [[ArrayList]] | Динамический массив | capacity, расширение ×1.5, сдвиги |
| [[LinkedList]] | Связанный список | Node, head/tail, O(1) на концах, O(n) get(i) |
| [[Set]] | Множество | HashSet / TreeSet / LinkedHashSet, уникальность |
| [[HashMap]] | Хэш-таблица | бакеты, коллизии, treeify, rehash |
| [[LinkedHashMap]] | HashMap с порядком | порядок вставки, порядок доступа, LRU-кэш |
| [[TreeMap]] | Отсортированная Map | красно-чёрное дерево, floor/ceiling, диапазоны |
| [[Queue]] | Очередь FIFO | offer/poll/peek, два набора методов |
| [[Deque]] | Двусторонняя очередь | FIFO + LIFO + дек, push/pop, offerFirst/Last |
| [[ArrayDeque]] | Быстрая реализация Deque | кольцевой массив, быстрее LinkedList и Stack |
| [[PriorityQueue]] | Очередь с приоритетом | binary heap, min/max heap, O(log n) offer/poll |

---

## 📚 Порядок изучения

```
Списки:
  [[ArrayList]] → [[LinkedList]]

Множества:
  [[Set]]

Словари:
  [[HashMap]] → [[LinkedHashMap]] → [[TreeMap]]

Очереди:
  [[Queue]] → [[Deque]] → [[ArrayDeque]] → [[PriorityQueue]]
```
