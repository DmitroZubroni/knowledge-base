# Bootstrap

> **Теги:** #frontend #css #bootstrap #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[CSS_Index]]

---

## 🔹 Концепция компонентного фреймворка

Bootstrap — компонентный CSS-фреймворк. Предоставляет готовые **компоненты** (кнопки, карточки, модалки) и **утилиты**.

```html
<!-- Быстрый старт через CDN -->
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
```

---

## 🔹 Grid система (12 колонок)

```html
<div class="container">           <!-- max-width + auto margins -->
  <div class="row g-3">           <!-- flex, g-3 = gap -->
    <div class="col-12 col-md-6 col-lg-4">   <!-- 12 / 6 / 4 колонки -->
      Контент
    </div>
    <div class="col-12 col-md-6 col-lg-8">
      Контент
    </div>
  </div>
</div>

<!-- Брейкпоинты: xs(<576) sm(576) md(768) lg(992) xl(1200) xxl(1400) -->
```

---

## 🔹 Ключевые компоненты

```html
<!-- Кнопки -->
<button class="btn btn-primary">Primary</button>
<button class="btn btn-outline-danger btn-sm">Маленькая</button>

<!-- Карточка -->
<div class="card shadow-sm">
  <img src="img.jpg" class="card-img-top">
  <div class="card-body">
    <h5 class="card-title">Заголовок</h5>
    <p class="card-text">Текст</p>
    <a href="#" class="btn btn-primary">Действие</a>
  </div>
</div>

<!-- Навигация -->
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <div class="container">
    <a class="navbar-brand" href="#">Лого</a>
    <div class="navbar-nav">
      <a class="nav-link active" href="#">Главная</a>
    </div>
  </div>
</nav>

<!-- Модальное окно -->
<button data-bs-toggle="modal" data-bs-target="#myModal">Открыть</button>
<div class="modal fade" id="myModal">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Заголовок</h5>
        <button class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">Контент</div>
    </div>
  </div>
</div>

<!-- Alerts, Badges, Spinners -->
<div class="alert alert-success alert-dismissible fade show">
  Успешно! <button class="btn-close" data-bs-dismiss="alert"></button>
</div>
<span class="badge bg-primary rounded-pill">42</span>
<div class="spinner-border text-primary"></div>
```

---

## 🔹 Утилиты

```html
<!-- Spacing: m/p + t/b/s/e/x/y + 0-5 -->
<div class="mt-3 mb-5 px-4 py-2">

<!-- Display -->
<div class="d-flex justify-content-between align-items-center">
<div class="d-none d-md-block">  <!-- скрыть на мобайле -->

<!-- Текст -->
<p class="text-center fw-bold fs-4 text-muted text-truncate">

<!-- Цвета -->
<div class="bg-primary text-white">
<p class="text-danger">Ошибка</p>

<!-- Border & Shadow -->
<div class="border border-2 border-primary rounded-3 shadow-lg">
```

---

## 🔹 Bootstrap vs Tailwind

| | Bootstrap | Tailwind |
|--|-----------|---------|
| **Подход** | Компоненты | Утилиты |
| **Скорость старта** | Быстрее | Медленнее |
| **Кастомизация** | Сложнее | Проще |
| **Размер** | Больше (можно урезать) | Маленький (только используемое) |
| **Дизайн** | Bootstrap-like | Полностью свой |
| **JS компоненты** | Встроены | Нет |

---

## 🔗 Смотри также
> - [[Tailwind]] — utility-first альтернатива
> - [[CSS_Basics]] — базовые CSS концепции
