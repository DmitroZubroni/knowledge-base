# npm / yarn / pnpm

> **Теги:** #frontend #tools #npm #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Tools_Index]]

---

## 🔹 package.json — манифест проекта

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest",
    "lint": "eslint src --fix"
  },
  "dependencies": {
    "react": "^18.2.0"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "typescript": "^5.0.0"
  },
  "engines": { "node": ">=18" }
}
```

**dependencies** — нужны в production.  
**devDependencies** — только для разработки (сборщики, линтеры, тесты).

---

## 🔹 Версионирование (semver)

```
"react": "18.2.0"   — точная версия
"react": "^18.2.0"  — 18.x.x (minor и patch можно обновлять)
"react": "~18.2.0"  — 18.2.x (только patch)
"react": "*"        — любая (не использовать)
```

---

## 🔹 Основные команды

```bash
# npm
npm install              # установить все зависимости из package.json
npm install react        # добавить в dependencies
npm install -D vite      # добавить в devDependencies
npm install -g nodemon   # глобально
npm uninstall lodash
npm update               # обновить в рамках semver
npm run dev              # запустить скрипт
npm run build
npx create-react-app .   # запустить без установки

# yarn
yarn / yarn install
yarn add react
yarn add -D vite
yarn remove lodash
yarn dev

# pnpm (быстрее, меньше места на диске)
pnpm install
pnpm add react
pnpm add -D vite
```

---

## 🔹 Lock-файлы

`package-lock.json` (npm) / `yarn.lock` / `pnpm-lock.yaml` — фиксируют точные версии **всех** транзитивных зависимостей.

- **Коммитить в Git** — обязательно
- Гарантирует одинаковое окружение у всей команды и на CI
- `npm ci` — строгая установка из lock-файла (для CI)

---

## 🔹 Полезные инструменты

```bash
npx npm-check-updates     # показать устаревшие зависимости
npx depcheck              # найти неиспользуемые зависимости
npm audit                 # проверить уязвимости
npm audit fix             # исправить автоматически
```

---

## 🔗 Смотри также
> - [[Vite_Webpack]] — сборщики используют npm пакеты
> - [[ESLint_Prettier]] — линтеры устанавливаются через npm
