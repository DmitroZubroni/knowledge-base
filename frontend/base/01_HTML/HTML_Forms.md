# HTML — Формы

> **Теги:** #frontend #html #forms #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[HTML_Index]]

---

## 🔹 Структура формы

```html
<form action="/submit" method="POST" novalidate>
  <label for="email">Email</label>
  <input type="email" id="email" name="email"
         placeholder="user@example.com"
         required autocomplete="email">

  <label for="password">Пароль</label>
  <input type="password" id="password" name="password"
         minlength="8" required>

  <button type="submit">Войти</button>
</form>
```

`label for` = `input id` — связка для accessibility (клик по label = фокус на input).

---

## 🔹 Типы input

| Тип | Назначение | Особенности |
|-----|-----------|-------------|
| `text` | Текстовое поле | |
| `email` | Email адрес | Встроенная валидация формата |
| `password` | Пароль | Скрывает символы |
| `number` | Число | min, max, step атрибуты |
| `tel` | Телефон | Мобильная клавиатура |
| `url` | URL | Валидация формата |
| `date` | Дата | Нативный date picker |
| `checkbox` | Флажок | checked, value |
| `radio` | Одиночный выбор | Одно name = одна группа |
| `file` | Загрузка файла | accept, multiple |
| `range` | Слайдер | min, max, step |
| `hidden` | Скрытое поле | Передаёт данные без UI |
| `search` | Поиск | Иконка очистки |

---

## 🔹 Другие элементы форм

```html
<!-- Выпадающий список -->
<select name="city">
  <optgroup label="Россия">
    <option value="msk">Москва</option>
    <option value="spb" selected>Санкт-Петербург</option>
  </optgroup>
</select>

<!-- Многострочный текст -->
<textarea name="comment" rows="5" cols="40" maxlength="500"></textarea>

<!-- Кнопки -->
<button type="submit">Отправить</button>
<button type="reset">Очистить</button>
<button type="button" onclick="doSomething()">Действие</button>

<!-- Группировка полей -->
<fieldset>
  <legend>Адрес</legend>
  <input type="text" name="street">
  <input type="text" name="city">
</fieldset>
```

---

## 🔹 Атрибуты валидации

```html
<input
  required                     <!-- обязательное поле -->
  minlength="3" maxlength="50" <!-- длина строки -->
  min="18" max="120"           <!-- диапазон числа -->
  pattern="[A-Za-z]{3,}"      <!-- regex -->
  type="email"                 <!-- тип = встроенная валидация -->
>
```

**Кастомная валидация через JS:**
```javascript
input.setCustomValidity('Имя должно содержать только буквы')
input.reportValidity()
```

---

## 🔹 Accessibility форм

```html
<!-- aria-label если нет visible label -->
<input type="search" aria-label="Поиск по сайту">

<!-- aria-describedby для подсказки -->
<input type="password" aria-describedby="pwd-hint">
<span id="pwd-hint">Минимум 8 символов</span>

<!-- aria-required для скринридеров -->
<input required aria-required="true">

<!-- aria-invalid при ошибке -->
<input aria-invalid="true">
<span role="alert">Неверный формат email</span>
```

---

## 🔗 Смотри также
> - [[JS_Basics]] — обработка событий формы в JS
> - [[HTML_DOM]] — доступ к форме через DOM
