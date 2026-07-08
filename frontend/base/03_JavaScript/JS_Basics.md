# JavaScript — Основы

> **Теги:** #frontend #javascript #basics #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[JS_Index]]

---

## 🔹 var / let / const

```javascript
var x = 1      // функциональный scope, hoisting, переобъявление — избегать
let y = 2      // блочный scope, нет hoisting, переприсваивание ✓
const z = 3    // блочный scope, нельзя переприсвоить

const obj = { a: 1 }
obj.a = 2      // ✓ — меняем содержимое
obj = {}       // ✗ — нельзя переприсвоить переменную

// Temporal Dead Zone: let/const не доступны до объявления
console.log(x) // undefined (hoisting var)
console.log(y) // ReferenceError (TDZ)
var x = 1; let y = 2
```

---

## 🔹 Типы данных

```javascript
// Примитивы (immutable, по значению)
typeof 42           // 'number'
typeof 'hello'      // 'string'
typeof true         // 'boolean'
typeof undefined    // 'undefined'
typeof null         // 'object'  ← исторический баг JS
typeof Symbol()     // 'symbol'
typeof 9007199n     // 'bigint'

// Объекты (mutable, по ссылке)
typeof {}           // 'object'
typeof []           // 'object'  ← Array.isArray([]) → true
typeof function(){} // 'function'

// Сравнение
null == undefined   // true  (нестрогое)
null === undefined  // false (строгое — всегда используй ===)
NaN === NaN         // false  ← NaN не равен самому себе
Number.isNaN(NaN)   // true
```

---

## 🔹 Деструктуризация и spread

```javascript
// Массивы
const [a, b, ...rest] = [1, 2, 3, 4, 5]
// a=1, b=2, rest=[3,4,5]

// Объекты
const { name, age = 25, address: { city } } = user
const { name: userName } = user  // переименование

// Spread
const arr2 = [...arr1, 4, 5]          // клон + добавление
const obj2 = { ...obj1, key: 'new' }  // клон + перезапись

// В параметрах функции
function greet({ name, role = 'user' }) { }
greet({ name: 'Dmitry', role: 'admin' })
```

---

## 🔹 Optional chaining и Nullish coalescing

```javascript
// Optional chaining ?.
const city = user?.address?.city         // не падает если address = null
const len  = arr?.length                 // undefined если arr = null
user?.sayHello?.()                       // вызов метода

// Nullish coalescing ??
const name = user.name ?? 'Аноним'      // только null/undefined (не 0, '')
const port = config.port ?? 3000

// Отличие от ||
'' || 'default'   // 'default'  ← '' считается falsy
'' ?? 'default'   // ''         ← только null/undefined

// Optional assignment
user.name ??= 'Аноним'   // присвоить только если null/undefined
counter  ||= 1           // присвоить если falsy
arr      &&= arr.map(x => x * 2) // только если truthy
```

---

## 🔹 Итерация

```javascript
// for...of — по значениям итерируемого
for (const item of array) { }
for (const char of 'string') { }
for (const [key, val] of map) { }

// for...in — по ключам объекта (включая прототип — осторожно)
for (const key in obj) {
  if (obj.hasOwnProperty(key)) { } // проверяй только собственные
}

// Array методы
array.forEach(item => { })              // side effects
array.map(item => item * 2)            // новый массив
array.filter(item => item > 0)         // фильтрация
array.reduce((acc, val) => acc + val, 0) // свёртка
array.find(item => item.id === 5)      // первый подходящий
array.some(item => item > 10)          // хотя бы один
array.every(item => item > 0)          // все
array.flat(Infinity)                   // развернуть вложенные
array.flatMap(item => [item, item * 2]) // map + flat(1)
```

---

## 🔗 Смотри также
> - [[JS_Functions]] — функции, замыкания, this
> - [[JS_Async]] — асинхронность, Promise, async/await
