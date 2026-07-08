# JavaScript — Функции и замыкания

> **Теги:** #frontend #javascript #functions #closure #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[JS_Index]]

---

## 🔹 Виды объявления функций

```javascript
// Function Declaration — hoisting, доступна до объявления
function add(a, b) { return a + b }

// Function Expression — нет hoisting
const add = function(a, b) { return a + b }

// Arrow Function — нет своего this, нет arguments
const add = (a, b) => a + b
const double = x => x * 2        // один параметр — скобки не нужны
const getObj = () => ({ key: 1 }) // возврат объекта — обернуть в ()

// IIFE — немедленный вызов
;(function() { /* изолированная область */ })()
;(() => { /* модульный паттерн */ })()
```

---

## 🔹 Параметры функций

```javascript
// Значения по умолчанию
function connect(host, port = 3000, protocol = 'http') { }

// Rest parameters
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0)
}
sum(1, 2, 3, 4) // 10

// Деструктуризация в параметрах
function render({ title, body, footer = '' }) { }
```

---

## 🔹 Замыкания (Closures)

Функция "запоминает" лексическое окружение где была создана.

```javascript
function makeCounter(initial = 0) {
  let count = initial  // приватная переменная

  return {
    increment: () => ++count,
    decrement: () => --count,
    value: () => count
  }
}

const counter = makeCounter(10)
counter.increment() // 11
counter.increment() // 12
counter.value()     // 12
// count недоступен снаружи

// Практическое применение: мемоизация
function memoize(fn) {
  const cache = new Map()
  return (...args) => {
    const key = JSON.stringify(args)
    if (cache.has(key)) return cache.get(key)
    const result = fn(...args)
    cache.set(key, result)
    return result
  }
}
```

---

## 🔹 this

```javascript
// Обычная функция: this = контекст вызова
const obj = {
  name: 'Object',
  greet() { return `Привет от ${this.name}` }
}
obj.greet()          // 'Привет от Object'
const fn = obj.greet
fn()                 // this = undefined (strict) или window

// Arrow function: this = лексический (из внешней функции)
class Timer {
  constructor() { this.ticks = 0 }
  start() {
    setInterval(() => {
      this.ticks++  // this = Timer instance (arrow!)
    }, 1000)
  }
}

// Явное связывание
fn.call(context, arg1, arg2)      // вызов с нужным this
fn.apply(context, [arg1, arg2])   // то же, аргументы в массиве
const bound = fn.bind(context)    // новая функция с фиксированным this
```

---

## 🔹 Функции высшего порядка (HOF)

```javascript
// Принимает функцию как аргумент
function repeat(fn, times) {
  for (let i = 0; i < times; i++) fn(i)
}
repeat(console.log, 3)

// Возвращает функцию — каррирование
const multiply = a => b => a * b
const double = multiply(2)
const triple = multiply(3)
double(5)  // 10
triple(5)  // 15

// Композиция функций
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x)
const pipe    = (...fns) => x => fns.reduce((v, f) => f(v), x)

const process = pipe(
  x => x * 2,
  x => x + 1,
  x => `Result: ${x}`
)
process(5) // 'Result: 11'
```

---

## 🔗 Смотри также
> - [[JS_Async]] — асинхронные функции, callback, Promise
> - [[JS_OOP]] — методы классов и this
