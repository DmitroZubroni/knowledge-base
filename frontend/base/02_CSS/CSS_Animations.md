# CSS — Анимации и переходы

> **Теги:** #frontend #css #animations #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[CSS_Index]]

---

## 🔹 Transition — плавный переход

```css
.button {
  background: #c0392b;
  transform: scale(1);

  /* transition: свойство длительность easing задержка */
  transition: background 0.3s ease, transform 0.2s ease;
  /* или все свойства: */
  transition: all 0.3s ease;
}

.button:hover {
  background: #e74c3c;
  transform: scale(1.05);
}
```

**Timing functions:**
- `ease` — медленно → быстро → медленно (по умолчанию)
- `linear` — равномерно
- `ease-in` — медленно → быстро
- `ease-out` — быстро → медленно
- `cubic-bezier(x1,y1,x2,y2)` — кастомная кривая

---

## 🔹 Transform — трансформации

```css
.el {
  transform: translate(50px, 20px);   /* смещение */
  transform: scale(1.5);              /* масштаб */
  transform: rotate(45deg);           /* поворот */
  transform: skew(10deg);             /* наклон */

  /* Комбинирование */
  transform: translate(-50%, -50%) scale(1.1);

  transform-origin: center;           /* точка трансформации */
}
```

**Важно:** `transform` не вызывает reflow — только repaint/composite. Гораздо эффективнее изменения `top/left`.

---

## 🔹 @keyframes — сложные анимации

```css
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Несколько точек */
@keyframes pulse {
  0%   { transform: scale(1); }
  50%  { transform: scale(1.1); }
  100% { transform: scale(1); }
}

.card {
  animation: fadeInUp 0.5s ease forwards;
  animation-delay: 0.2s;
}

.badge {
  animation: pulse 1.5s ease infinite;
}
```

---

## 🔹 Свойства animation

```css
.el {
  animation-name: fadeInUp;
  animation-duration: 0.5s;
  animation-timing-function: ease;
  animation-delay: 0.1s;
  animation-iteration-count: 1;     /* infinite | число */
  animation-direction: normal;      /* normal | reverse | alternate */
  animation-fill-mode: forwards;    /* none | forwards | backwards | both */
  animation-play-state: running;    /* running | paused */

  /* Шорткат */
  animation: fadeInUp 0.5s ease 0.1s forwards;
}
```

---

## 🔹 will-change и производительность

```css
/* Подсказка браузеру: оптимизировать этот элемент */
.animated {
  will-change: transform, opacity; /* создаёт отдельный слой */
}
/* ⚠️ Не злоупотреблять — каждый слой потребляет память */

/* Анимируй через transform/opacity — они composite-only */
/* Избегай анимации: width, height, top, left, margin — они reflow */
```

---

## 🔗 Смотри также
> - [[CSS_Basics]] — позиционирование, box model
> - [[CSS_Flexbox]] — расположение анимируемых элементов
