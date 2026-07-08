# HTML — Основы

> **Теги:** #frontend #html #basics #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[HTML_Index]]

---

## 🔹 Что такое HTML

**HTML (HyperText Markup Language)** — язык разметки для создания структуры веб-страниц. Это не язык программирования — он описывает **что есть на странице**, а не логику.

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Моя страница</title>
</head>
<body>
    <h1>Заголовок</h1>
    <p>Абзац текста</p>
</body>
</html>
```

---

## 🔹 Структура тега

```
<тег атрибут="значение">Содержимое</тег>
 ↑                              ↑
открывающий тег          закрывающий тег

<img src="photo.jpg" alt="Фото">   ← самозакрывающийся
```

---

## 🔹 Блочные vs строчные элементы

| Тип | Поведение | Примеры |
|-----|-----------|---------|
| **Block** | Занимает всю ширину, начинается с новой строки | `<div>`, `<p>`, `<h1>`–`<h6>`, `<ul>`, `<li>` |
| **Inline** | Занимает только своё содержимое, в строке | `<span>`, `<a>`, `<strong>`, `<em>`, `<img>` |
| **Inline-block** | Как inline, но можно задать размеры | `<button>`, `<input>` |

---

## 🔹 Важные атрибуты

```html
<!-- Глобальные — работают на любом теге -->
<div id="unique-id">        <!-- уникальный идентификатор -->
<div class="my-class">      <!-- для CSS/JS, не уникален -->
<div style="color: red">    <!-- инлайн стиль (избегать) -->
<div data-user-id="42">     <!-- кастомные данные (data-*) -->
<div hidden>                <!-- скрыть элемент -->

<!-- Специфичные -->
<a href="/about" target="_blank">   <!-- ссылка, открыть в новой вкладке -->
<img src="img.jpg" alt="описание">  <!-- alt обязателен для accessibility -->
<input type="text" placeholder="Введите имя" required>
```

---

## 🔹 Мета-теги

```html
<head>
    <meta charset="UTF-8">                              <!-- кодировка -->
    <meta name="description" content="Описание сайта"> <!-- для SEO -->
    <meta name="viewport" content="width=device-width"> <!-- адаптив -->
    <meta property="og:title" content="Заголовок">     <!-- Open Graph -->
    <link rel="stylesheet" href="style.css">
    <script src="app.js" defer></script>               <!-- defer: после HTML -->
</head>
```

---

## 🔹 Чеклист качественного HTML

- [ ] DOCTYPE и lang указаны
- [ ] charset UTF-8
- [ ] viewport meta для мобайла
- [ ] alt у всех `<img>`
- [ ] label у каждого `<input>`
- [ ] Нет инлайновых стилей
- [ ] Семантические теги вместо голых `<div>`

---

## 🔗 Смотри также
> - [[HTML_Semantic]] — семантика и SEO
> - [[HTML_Forms]] — формы, валидация
> - [[CSS_Basics]] — стилизация
