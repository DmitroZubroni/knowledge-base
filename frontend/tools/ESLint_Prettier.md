# ESLint / Prettier

> **Теги:** #frontend #tools #eslint #prettier #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Tools_Index]]

---

## 🔹 ESLint — линтер кода

Находит потенциальные ошибки и проблемы стиля. Не форматирует — анализирует.

```json
// eslint.config.js (flat config, ESLint 9+)
import js from '@eslint/js'
import tseslint from 'typescript-eslint'
import reactHooks from 'eslint-plugin-react-hooks'

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    plugins: { 'react-hooks': reactHooks },
    rules: {
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
      '@typescript-eslint/no-unused-vars': 'error',
      '@typescript-eslint/no-explicit-any': 'warn',
      'no-console': 'warn'
    }
  }
]
```

```bash
npx eslint src             # проверить
npx eslint src --fix       # автоисправление
```

---

## 🔹 Prettier — форматирование кода

Единый стиль форматирования. Не анализирует ошибки — только форматирует.

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "printWidth": 100,
  "trailingComma": "es5",
  "arrowParens": "avoid",
  "endOfLine": "lf"
}
```

```bash
npx prettier --write src   # форматировать
npx prettier --check src   # проверить (для CI)
```

---

## 🔹 Pre-commit хук (Husky + lint-staged)

```bash
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{css,json,md}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged
```

Теперь при каждом `git commit` — автоматический lint и форматирование изменённых файлов.

---

## 🔗 Смотри также
> - [[npm_yarn]] — установка ESLint и Prettier
> - [[Vite_Webpack]] — интеграция ESLint плагина в сборщик
