# TypeScript

> **Теги:** #frontend #typescript #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[JS_Index]]

---

## 🔹 Что такое TypeScript

TypeScript — надмножество JavaScript. Добавляет статическую типизацию, компилируется в JS. Ошибки типов обнаруживаются **при написании кода**, а не в runtime.

```typescript
// JS — ошибка обнаруживается только в runtime
function add(a, b) { return a + b }
add(1, '2')  // '12' — неожиданно, но не ошибка

// TS — ошибка видна сразу в IDE
function add(a: number, b: number): number { return a + b }
add(1, '2')  // ошибка: Argument of type 'string' is not assignable to type 'number'
```

---

## 🔹 Базовые типы

```typescript
// Примитивы
let name: string = 'Дмитрий'
let age: number = 25
let active: boolean = true
let nothing: null = null
let missing: undefined = undefined
let sym: symbol = Symbol('id')
let big: bigint = 9007199n

// Массивы
let nums: number[] = [1, 2, 3]
let strs: Array<string> = ['a', 'b']

// Tuple — фиксированная структура
let pair: [string, number] = ['age', 25]

// any, unknown, never, void
let x: any = 'anything'       // отключает проверки — избегать
let y: unknown = getData()    // нужна проверка типа перед использованием
function throws(): never { throw new Error() }  // никогда не возвращает
function log(): void { console.log() }           // не возвращает значение
```

---

## 🔹 Interface и Type

```typescript
// Interface — для объектов и классов
interface User {
  readonly id: number  // нельзя изменить
  name: string
  email?: string       // опциональное поле
  role: 'admin' | 'user'
}

// Расширение interface
interface Admin extends User {
  permissions: string[]
}

// Type alias — для любых типов
type ID = string | number
type Callback = (err: Error | null, data: unknown) => void
type Point = { x: number; y: number }

// Interface vs Type: interface можно объединять (declaration merging)
// Type предпочтительнее для union/intersection и примитивов
// Interface предпочтительнее для объектов и классов
```

---

## 🔹 Generics

```typescript
// Обобщённые функции
function identity<T>(arg: T): T { return arg }
identity<string>('hello')  // string
identity(42)               // number — вывод типа

// Обобщённые интерфейсы
interface ApiResponse<T> {
  data: T
  status: number
  message: string
}

async function fetchUser(id: number): Promise<ApiResponse<User>> {
  const res = await fetch(`/api/users/${id}`)
  return res.json()
}

// Ограничения generics
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
getProperty(user, 'name')  // string
getProperty(user, 'xyz')   // ошибка — 'xyz' не ключ User
```

---

## 🔹 Utility Types

```typescript
interface User { id: number; name: string; email: string; password: string }

Partial<User>       // все поля опциональными: { id?: number, name?: string, ... }
Required<User>      // все поля обязательными
Readonly<User>      // все поля readonly
Pick<User, 'id' | 'name'>  // только выбранные поля
Omit<User, 'password'>     // исключить поля

Record<string, number>     // { [key: string]: number }
Exclude<'a' | 'b' | 'c', 'b'>  // 'a' | 'c'
Extract<'a' | 'b' | 'c', 'a' | 'c'>  // 'a' | 'c'
ReturnType<typeof fn>      // тип возвращаемого значения
Parameters<typeof fn>      // tuple типов параметров
NonNullable<string | null | undefined>  // string
```

---

## 🔹 Narrowing — сужение типов

```typescript
function process(value: string | number | null) {
  if (value === null) return  // null narrowed out

  if (typeof value === 'string') {
    value.toUpperCase()  // здесь value: string
  } else {
    value.toFixed(2)     // здесь value: number
  }
}

// Discriminated Union
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rect'; width: number; height: number }

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2
    case 'rect':   return shape.width * shape.height
    // TS знает что все случаи покрыты
  }
}
```

---

## 🔗 Смотри також
> - [[JS_Basics]] — базовый JavaScript
> - [[React_Core]] — TypeScript в React компонентах
