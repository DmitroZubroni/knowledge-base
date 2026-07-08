# Angular — RxJS и Signals

> **Теги:** #frontend #angular #rxjs #signals #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Angular_Index]]

---

## 🔹 Observable — основа RxJS

```typescript
import { Observable, of, from, interval, Subject, BehaviorSubject } from 'rxjs'
import { map, filter, switchMap, debounceTime, takeUntil, catchError } from 'rxjs/operators'

// Создание
const obs1$ = of(1, 2, 3)                          // из значений
const obs2$ = from([1, 2, 3])                      // из массива
const obs3$ = from(fetch('/api/users'))             // из Promise
const timer$ = interval(1000)                       // каждую секунду

// Подписка
const subscription = obs1$.subscribe({
  next: value => console.log(value),
  error: err => console.error(err),
  complete: () => console.log('done')
})
subscription.unsubscribe()  // обязательно отписаться!
```

---

## 🔹 Операторы RxJS

```typescript
// Трансформация
obs$.pipe(
  map(user => user.name),           // преобразование
  filter(name => name.length > 3),  // фильтрация

  // switchMap — для HTTP запросов (отменяет предыдущий)
  switchMap(id => this.http.get(`/api/users/${id}`)),

  // mergeMap — параллельные запросы
  mergeMap(id => this.http.get(`/api/users/${id}`)),

  debounceTime(300),    // ждать 300мс после последнего события (поиск)
  distinctUntilChanged(), // только если значение изменилось

  catchError(err => {
    console.error(err)
    return of(null)     // восстановление после ошибки
  }),

  takeUntil(this.destroy$)  // автоотписка при уничтожении компонента
)
```

---

## 🔹 Subject — Observable + Observer

```typescript
// Subject — multicast, без начального значения
const subject$ = new Subject<string>()
subject$.next('hello')  // emit

// BehaviorSubject — хранит последнее значение
const state$ = new BehaviorSubject<User | null>(null)
state$.getValue()       // текущее значение без подписки
state$.next(user)       // обновить

// ReplaySubject — буферизирует N последних значений
const replay$ = new ReplaySubject<number>(3)
```

---

## 🔹 Angular Signals (Angular 16+)

Современная альтернатива RxJS для простых случаев:

```typescript
import { signal, computed, effect } from '@angular/core'

// Создание
const count = signal(0)
const users = signal<User[]>([])

// computed — реактивное вычисление
const doubleCount = computed(() => count() * 2)
const activeUsers = computed(() => users().filter(u => u.active))

// effect — сайд-эффекты при изменении
effect(() => {
  console.log(`Count: ${count()}`)  // автоматически отслеживает зависимости
})

// Изменение
count.set(5)                        // установить
count.update(v => v + 1)            // обновить на основе предыдущего
users.mutate(list => list.push(newUser))  // мутировать (осторожно)

// В шаблоне — signals вызываются как функции
@Component({
  template: `<p>{{ count() }}</p><p>{{ doubleCount() }}</p>`
})
```

---

## 🔗 Смотри также
> - [[Angular_Core]] — компоненты и DI
> - [[JS_Async]] — Promise и async/await как альтернатива
