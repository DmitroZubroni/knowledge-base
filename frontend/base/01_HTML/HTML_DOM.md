# HTML — DOM и браузер

> **Теги:** #frontend #html #dom #browser #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[HTML_Index]]

---

## 🔹 Что такое DOM

**DOM (Document Object Model)** — программное представление HTML-документа. Браузер парсит HTML и строит дерево объектов.

```
HTML строка:                    DOM дерево:
<html>                          Document
  <body>                         └── html
    <h1>Привет</h1>                    └── body
    <p>Текст</p>                             ├── h1 ("Привет")
  </body>                                   └── p ("Текст")
</html>
```

JavaScript взаимодействует именно с DOM-деревом, а не с HTML-строкой.

---

## 🔹 Render Pipeline браузера

```
HTML-текст
    ↓ Parsing
DOM Tree + CSSOM Tree
    ↓ Render Tree (объединение)
Layout (где что расположено, пиксели)
    ↓
Paint (рисует слои)
    ↓
Composite (склеивает слои → экран)
```

**Reflow** — пересчёт Layout (дорогостоящий). Вызывает: изменение размеров, позиции, добавление/удаление элементов.  
**Repaint** — перерисовка без изменения layout (дешевле). Вызывает: изменение цвета, фона, visibility.

---

## 🔹 BOM — Browser Object Model

| Объект | Что предоставляет |
|--------|-------------------|
| `window` | Глобальный объект, размеры окна |
| `navigator` | Информация о браузере/ОС |
| `location` | URL, навигация (`location.href`, `location.reload()`) |
| `history` | История браузера (`history.back()`, `history.pushState()`) |
| `screen` | Размеры экрана |
| `localStorage` / `sessionStorage` | Хранилище данных |

---

## 🔹 Загрузка скриптов

```html
<!-- Блокирует парсинг HTML — избегать в <head> -->
<script src="app.js"></script>

<!-- defer: загружает параллельно, выполняет после HTML -->
<script src="app.js" defer></script>

<!-- async: загружает параллельно, выполняет сразу как загрузил -->
<script src="analytics.js" async></script>
```

**Рекомендация:** `defer` для большинства скриптов. `async` для независимых скриптов (аналитика).

---

## 🔹 События загрузки

```javascript
// DOMContentLoaded — HTML распарсен, DOM готов
// (без ожидания картинок и стилей)
document.addEventListener('DOMContentLoaded', () => {
  console.log('DOM готов')
})

// load — ВСЁ загружено (картинки, стили, шрифты)
window.addEventListener('load', () => {
  console.log('Страница полностью загружена')
})

// beforeunload — пользователь уходит со страницы
window.addEventListener('beforeunload', (e) => {
  e.preventDefault()
  e.returnValue = '' // диалог подтверждения
})
```

---

## 🔗 Смотри также
> - [[JS_Basics]] — работа с DOM через JavaScript
> - [[HTML_Basics]] — базовый синтаксис HTML
