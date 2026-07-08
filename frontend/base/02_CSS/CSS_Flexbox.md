# CSS — Flexbox

> **Теги:** #frontend #css #flexbox #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[CSS_Index]]

---

## 🔹 Концепция

Flexbox — одномерная модель: выравнивание по **одной оси** (строка или колонка).

```css
.container {
  display: flex;            /* включить Flexbox */
  flex-direction: row;      /* направление: row | column | row-reverse | column-reverse */
}
```

---

## 🔹 Свойства контейнера

```css
.container {
  display: flex;

  /* Главная ось */
  flex-direction: row;                  /* row (→) | column (↓) */
  justify-content: flex-start;         /* flex-start | center | flex-end
                                          space-between | space-around | space-evenly */

  /* Поперечная ось */
  align-items: stretch;                /* stretch | center | flex-start | flex-end | baseline */

  /* Перенос */
  flex-wrap: nowrap;                   /* nowrap | wrap | wrap-reverse */

  /* Многострочное выравнивание */
  align-content: flex-start;          /* только при flex-wrap: wrap */

  /* Шорткаты */
  flex-flow: row wrap;                 /* direction + wrap */
  gap: 16px;                          /* отступы между элементами */
  gap: 16px 8px;                      /* row-gap column-gap */
}
```

---

## 🔹 Свойства элементов

```css
.item {
  flex-grow: 0;     /* насколько растянуть (0 = не растягивать) */
  flex-shrink: 1;   /* насколько сжать при нехватке места */
  flex-basis: auto; /* начальный размер */

  flex: 1;          /* flex-grow: 1, flex-shrink: 1, flex-basis: 0 */
  flex: 0 0 200px;  /* фиксированный размер 200px */

  align-self: center; /* переопределить align-items для одного */
  order: 2;           /* порядок отображения */
}
```

---

## 🔹 Частые паттерны

```css
/* Центрировать всё */
.center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Прижать последний элемент к правому краю */
.header { display: flex; align-items: center; }
.header .logo { margin-right: auto; } /* занимает всё свободное место */

/* Равномерно распределить */
.nav { display: flex; gap: 24px; }
.nav a { flex: 1; text-align: center; }

/* Карточки с wrap */
.cards {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}
.card { flex: 0 0 calc(33.333% - 11px); } /* 3 в ряд */
```

---

## 🔗 Смотри также
> - [[CSS_Grid]] — двумерная сетка (строки И колонки)
> - [[CSS_Basics]] — позиционирование, box model
