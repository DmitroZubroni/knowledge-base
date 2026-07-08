# JavaScript — Модули и DOM API

> **Теги:** #frontend #javascript #modules #dom #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[JS_Index]]

---

## 🔹 ES Modules

```javascript
// math.js — именованный экспорт
export const PI = 3.14159
export function add(a, b) { return a + b }
export class Calculator { }

// utils.js — экспорт по умолчанию
export default function formatDate(date) {
  return date.toLocaleDateString('ru')
}

// main.js — импорт
import formatDate from './utils.js'              // default
import { PI, add } from './math.js'              // named
import { add as sum } from './math.js'           // переименование
import * as MathLib from './math.js'             // всё

// Динамический импорт (lazy loading)
const { add } = await import('./math.js')

// Реэкспорт
export { add, PI } from './math.js'
export { default as formatDate } from './utils.js'
```

---

## 🔹 DOM API — поиск элементов

```javascript
// Современный способ
document.querySelector('.btn')              // первый подходящий
document.querySelectorAll('.card')          // все (NodeList)
document.getElementById('header')          // по id (быстро)

// Навигация по дереву
element.parentElement
element.children                           // только Element дети
element.firstElementChild / lastElementChild
element.nextElementSibling / previousElementSibling
element.closest('.container')             // ближайший предок с selector
element.matches('.active')                // подходит ли под селектор
```

---

## 🔹 DOM API — изменение элементов

```javascript
// Содержимое
el.textContent = 'Текст'          // только текст (безопасно)
el.innerHTML = '<b>HTML</b>'      // HTML (XSS риск!)
el.insertAdjacentHTML('beforeend', '<div>новый</div>')
// позиции: beforebegin, afterbegin, beforeend, afterend

// Атрибуты и классы
el.setAttribute('data-id', '42')
el.getAttribute('data-id')        // '42'
el.removeAttribute('disabled')
el.dataset.id                     // data-id через dataset

el.classList.add('active')
el.classList.remove('active')
el.classList.toggle('active')
el.classList.contains('active')   // boolean
el.classList.replace('old', 'new')

// Стили
el.style.color = 'red'           // инлайн стиль
el.style.cssText = 'color: red; font-size: 16px'
getComputedStyle(el).color        // вычисленный стиль
```

---

## 🔹 Создание и добавление элементов

```javascript
const div = document.createElement('div')
div.className = 'card'
div.textContent = 'Контент'

parent.appendChild(div)               // в конец
parent.prepend(div)                   // в начало
parent.insertBefore(div, reference)   // перед reference
parent.replaceChild(newEl, oldEl)
el.remove()                           // удалить

// Эффективное создание списка (DocumentFragment)
const fragment = document.createDocumentFragment()
items.forEach(item => {
  const li = document.createElement('li')
  li.textContent = item
  fragment.appendChild(li)
})
ul.appendChild(fragment)  // один reflow вместо N
```

---

## 🔹 События

```javascript
// Добавление слушателя
el.addEventListener('click', handler)
el.addEventListener('click', handler, { once: true })    // один раз
el.addEventListener('click', handler, { passive: true }) // для scroll
el.removeEventListener('click', handler)

// Объект события
el.addEventListener('click', (event) => {
  event.target           // элемент на который кликнули
  event.currentTarget    // элемент с обработчиком
  event.preventDefault() // отменить поведение по умолчанию
  event.stopPropagation() // остановить всплытие
})

// Event delegation — один обработчик для динамических элементов
document.querySelector('.list').addEventListener('click', (e) => {
  const item = e.target.closest('.list-item')
  if (!item) return
  const id = item.dataset.id
  // обрабатываем клик по элементу
})
```

---

## 🔗 Смотри также
> - [[HTML_DOM]] — что такое DOM и как браузер его строит
> - [[JS_Async]] — асинхронные операции с DOM
> - [[Vite_Webpack]] — сборщики работающие с модулями
