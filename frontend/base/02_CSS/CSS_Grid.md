# CSS — Grid

> **Теги:** #frontend #css #grid #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[CSS_Index]]

---

## 🔹 Концепция

CSS Grid — двумерная модель: управление по **строкам И колонкам** одновременно.

```
Flexbox  → одна ось (строка OR колонка)
Grid     → две оси (строки AND колонки)
```

---

## 🔹 Определение сетки

```css
.container {
  display: grid;

  /* Колонки */
  grid-template-columns: 200px 1fr 1fr;     /* 3 колонки */
  grid-template-columns: repeat(3, 1fr);    /* 3 равные */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); /* адаптив */

  /* Строки */
  grid-template-rows: 60px 1fr 40px;       /* header, content, footer */
  grid-auto-rows: minmax(100px, auto);      /* высота автоматических строк */

  /* Отступы */
  gap: 16px;
  column-gap: 24px;
  row-gap: 16px;
}
```

---

## 🔹 Размещение элементов

```css
/* Явное размещение */
.item {
  grid-column: 1 / 3;          /* от линии 1 до линии 3 (span 2) */
  grid-column: 1 / -1;         /* от 1 до последней (вся ширина) */
  grid-column: span 2;         /* занять 2 колонки */

  grid-row: 1 / 3;             /* первые 2 строки */
}

/* Именованные области */
.layout {
  grid-template-areas:
    "header header"
    "sidebar content"
    "footer footer";
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.content { grid-area: content; }
.footer  { grid-area: footer; }
```

---

## 🔹 Выравнивание

```css
.container {
  justify-items: start;    /* выравнивание ячеек по горизонтали */
  align-items: center;     /* выравнивание ячеек по вертикали */
  justify-content: center; /* выравнивание сетки по горизонтали */
  align-content: center;   /* выравнивание сетки по вертикали */
  place-items: center;     /* align-items + justify-items */
}

.item {
  justify-self: end;       /* для конкретного элемента */
  align-self: start;
}
```

---

## 🔹 Адаптивная сетка

```css
/* Автоматически вписывает максимум колонок шириной от 200px */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
}
/* При 600px viewport: 3 колонки по ~180px (min) → 3 × 1fr */
/* При 900px viewport: 4 колонки */
```

---

## 🔗 Смотри также
> - [[CSS_Flexbox]] — для одномерного расположения
> - [[CSS_Basics]] — базовый синтаксис CSS
