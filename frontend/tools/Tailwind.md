# Tailwind CSS

> **Теги:** #frontend #css #tailwind #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[CSS_Index]]

---

## 🔹 Концепция Utility-First

Tailwind — **не компонентный** фреймворк. Вместо готовых `.btn`, `.card` — атомарные классы для каждого CSS-свойства.

```html
<!-- Bootstrap подход (компоненты) -->
<button class="btn btn-primary btn-lg">Кнопка</button>

<!-- Tailwind подход (утилиты) -->
<button class="px-6 py-3 bg-blue-600 text-white font-semibold
               rounded-lg hover:bg-blue-700 transition-colors">
  Кнопка
</button>
```

---

## 🔹 Система именования классов

```html
<!-- Spacing (padding/margin) -->
<div class="p-4">        <!-- padding: 1rem (4 × 0.25rem) -->
<div class="px-6 py-3">  <!-- padding-x: 1.5rem, padding-y: 0.75rem -->
<div class="mt-8 mb-4">  <!-- margin-top: 2rem, margin-bottom: 1rem -->

<!-- Размеры -->
<div class="w-full h-64"> <!-- width: 100%, height: 16rem -->
<div class="max-w-lg">    <!-- max-width: 32rem -->

<!-- Цвета -->
<div class="bg-gray-100 text-gray-900 border-gray-300">
<div class="bg-blue-600 hover:bg-blue-700">

<!-- Типографика -->
<p class="text-lg font-bold leading-relaxed tracking-wide">
<p class="text-sm text-gray-500 uppercase">

<!-- Flex/Grid -->
<div class="flex items-center justify-between gap-4">
<div class="grid grid-cols-3 gap-6">
```

---

## 🔹 Responsive Design

Брейкпоинты через префиксы:

```html
<!-- sm: 640px, md: 768px, lg: 1024px, xl: 1280px, 2xl: 1536px -->

<div class="
  grid
  grid-cols-1        <!-- мобайл: 1 колонка -->
  md:grid-cols-2     <!-- tablet: 2 колонки -->
  lg:grid-cols-3     <!-- desktop: 3 колонки -->
">

<p class="text-base md:text-lg lg:text-xl">
  Текст меняет размер по брейкпоинтам
</p>
```

---

## 🔹 State variants

```html
<!-- hover, focus, active, disabled -->
<button class="bg-blue-600 hover:bg-blue-700 active:bg-blue-800
               focus:outline-none focus:ring-2 focus:ring-blue-500
               disabled:opacity-50 disabled:cursor-not-allowed">

<!-- dark mode -->
<div class="bg-white dark:bg-gray-900 text-black dark:text-white">

<!-- group hover -->
<div class="group">
  <p class="text-gray-700 group-hover:text-blue-600">
    Меняется при наведении на родителя
  </p>
</div>
```

---

## 🔹 @apply — переиспользование

```css
/* В CSS файле — для часто используемых комбинаций */
.btn-primary {
  @apply px-6 py-3 bg-blue-600 text-white font-semibold
         rounded-lg hover:bg-blue-700 transition-colors;
}

/* Используй умеренно — теряется смысл utility-first */
```

---

## 🔹 Конфигурация (tailwind.config.js)

```javascript
module.exports = {
  content: ['./src/**/*.{js,jsx,ts,tsx}'],
  theme: {
    extend: {
      colors: {
        brand: { 500: '#c0392b', 600: '#a93226' }
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif']
      },
      spacing: {
        '18': '4.5rem'
      }
    }
  },
  darkMode: 'class', // или 'media'
  plugins: []
}
```

---

## 🔗 Смотри также
> - [[Bootstrap]] — компонентный подход как альтернатива
> - [[CSS_Basics]] — базовые CSS концепции лежащие в основе
> - [[React_Core]] — Tailwind часто используется с React
