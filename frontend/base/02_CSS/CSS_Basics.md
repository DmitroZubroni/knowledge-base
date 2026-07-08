# CSS — Основы

> **Теги:** #frontend #css #basics #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[CSS_Index]]

---

## 🔹 Синтаксис

```css
селектор {
  свойство: значение;
  свойство: значение;
}

.button {               /* класс */
  background: #c0392b;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
}
```

---

## 🔹 Типы селекторов

```css
*           { }   /* все элементы */
div         { }   /* тег */
.class      { }   /* класс */
#id         { }   /* id */
[type="text"]{ }  /* атрибут */

/* Комбинаторы */
div p       { }   /* потомок */
div > p     { }   /* прямой потомок */
h1 + p      { }   /* следующий сосед */
h1 ~ p      { }   /* все следующие соседи */

/* Псевдоклассы */
a:hover     { }   /* при наведении */
li:nth-child(2n){ } /* чётные */
p:first-child{ }  /* первый дочерний */
input:focus { }   /* в фокусе */
:not(.active){ }  /* отрицание */

/* Псевдоэлементы */
p::before   { content: ''; }  /* вставить до */
p::after    { content: ''; }  /* вставить после */
::placeholder{ }               /* placeholder input */
::selection { }                /* выделенный текст */
```

---

## 🔹 Специфичность (Specificity)

При конфликте правил побеждает более специфичное:

```
!important  → всегда побеждает (избегать)
#id         → (1, 0, 0)
.class      → (0, 1, 0)
тег         → (0, 0, 1)

Пример:
#nav .item a    → (1, 1, 1)  — победит
.nav-item a     → (0, 1, 1)  — проиграет
```

---

## 🔹 Блочная модель (Box Model)

```
┌─── margin ──────────────────────┐
│  ┌── border ──────────────────┐ │
│  │  ┌── padding ────────────┐ │ │
│  │  │                       │ │ │
│  │  │  content              │ │ │
│  │  │  width × height       │ │ │
│  │  └───────────────────────┘ │ │
│  └────────────────────────────┘ │
└─────────────────────────────────┘
```

```css
/* По умолчанию width = только content */
.box { width: 200px; padding: 20px; } /* реальная = 240px */

/* border-box — width включает padding и border */
* { box-sizing: border-box; } /* рекомендуется глобально */
.box { width: 200px; padding: 20px; } /* реальная = 200px */
```

---

## 🔹 CSS-переменные (Custom Properties)

```css
:root {
  --primary: #c0392b;
  --spacing-md: 16px;
  --font-size-base: 1rem;
}

.button {
  background: var(--primary);
  padding: var(--spacing-md);
}

/* Переопределение для тёмной темы */
@media (prefers-color-scheme: dark) {
  :root {
    --primary: #e74c3c;
  }
}
```

---

## 🔹 Позиционирование

```css
position: static;    /* по умолчанию — в потоке */
position: relative;  /* смещение от нормальной позиции */
position: absolute;  /* позиция относительно ближайшего relative */
position: fixed;     /* фиксированный относительно viewport */
position: sticky;    /* static до threshold, потом fixed */

.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%); /* центрирование */
}
```

---

## 🔗 Смотри также
> - [[CSS_Flexbox]] — расположение элементов
> - [[CSS_Grid]] — двумерная сетка
> - [[Tailwind]] — utility-first альтернатива
