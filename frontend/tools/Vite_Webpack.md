# Vite / Webpack

> **Теги:** #frontend #tools #vite #webpack #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Tools_Index]]

---

## 🔹 Зачем нужен сборщик

Браузер не понимает: TypeScript, SCSS, JSX, `import from './module'` в старых форматах. Сборщик:
- Транспилирует TS/JSX → JS
- Объединяет модули в бандл
- Минифицирует для production
- Разбивает на чанки (code splitting)
- Оптимизирует assets (изображения, шрифты)

---

## 🔹 Vite — современный сборщик

```javascript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': '/src' }  // @/components = src/components
  },
  server: {
    port: 3000,
    proxy: {
      '/api': 'http://localhost:8080'  // проксировать к бэкенду
    }
  },
  build: {
    outDir: 'dist',
    sourcemap: true
  }
})
```

**Почему Vite быстрее Webpack:**
- Dev сервер: отдаёт ES Modules напрямую, без сборки в бандл
- HMR (Hot Module Replacement): обновляет только изменённый модуль
- Build: использует Rollup (быстрее webpack)

```bash
npm create vite@latest my-app -- --template react-ts
npm install && npm run dev
```

---

## 🔹 Code Splitting

```javascript
// Автоматическое разбиение на чанки
// Vite/Webpack разбивают vendor (node_modules) и app код

// Динамический импорт — ленивая загрузка
const LazyComponent = React.lazy(() => import('./HeavyComponent'))

// По роутам (React Router)
const About = lazy(() => import('./pages/About'))
<Route path="/about" element={
  <Suspense fallback={<Spinner />}>
    <About />
  </Suspense>
} />
```

---

## 🔹 Переменные окружения

```bash
# .env
VITE_API_URL=http://localhost:8080
VITE_APP_TITLE=My App

# .env.production
VITE_API_URL=https://api.myapp.com
```

```typescript
// В коде — только VITE_ префикс доступен клиенту
const apiUrl = import.meta.env.VITE_API_URL
const isProd  = import.meta.env.PROD   // boolean
const isDev   = import.meta.env.DEV    // boolean
```

---

## 🔗 Смотри также
> - [[npm_yarn]] — управление зависимостями
> - [[ESLint_Prettier]] — интеграция с линтером

