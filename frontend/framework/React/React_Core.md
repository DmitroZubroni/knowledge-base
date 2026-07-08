# React — Компоненты и JSX

> **Теги:** #frontend #react #jsx #components #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[React_Index]]

---

## 🔹 Компонент — строительный блок

React-приложение = дерево компонентов. Компонент — функция возвращающая JSX.

```tsx
// Функциональный компонент с TypeScript
interface ButtonProps {
  label: string
  onClick: () => void
  variant?: 'primary' | 'secondary'
  disabled?: boolean
}

const Button = ({ label, onClick, variant = 'primary', disabled = false }: ButtonProps) => {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
    >
      {label}
    </button>
  )
}

export default Button
```

---

## 🔹 JSX — правила

```tsx
// 1. Один корневой элемент (или Fragment)
return (
  <>
    <h1>Заголовок</h1>
    <p>Текст</p>
  </>
)

// 2. className вместо class
<div className="card">

// 3. Выражения в {}
<p>{user.name}</p>
<p>{isActive ? 'Активен' : 'Неактивен'}</p>
<p>{count > 0 && <span>{count}</span>}</p>

// 4. Списки — нужен key
{users.map(user => (
  <UserCard key={user.id} user={user} />
))}

// 5. Стили — объект
<div style={{ color: 'red', fontSize: '16px' }}>

// 6. Events — camelCase
<button onClick={() => console.log('click')}>
<input onChange={e => setValue(e.target.value)}>
```

---

## 🔹 Virtual DOM и Reconciliation

React не работает с реальным DOM напрямую. Он строит **Virtual DOM** (JS-объекты) и сравнивает с предыдущей версией при обновлении.

```
1. State изменился
2. React вызывает рендер компонента → новый Virtual DOM
3. Diffing: сравнивает новый VDOM с предыдущим
4. Патчит только изменённые узлы реального DOM
```

**key** в списках помогает React понять что переиспользовать, а что пересоздать.

---

## 🔹 Props

```tsx
// Передача пропсов
<UserCard user={user} onDelete={handleDelete} showEmail />
// showEmail = shorthand для showEmail={true}

// children — специальный проп
interface CardProps {
  title: string
  children: React.ReactNode
}

const Card = ({ title, children }: CardProps) => (
  <div className="card">
    <h2>{title}</h2>
    <div className="card-body">{children}</div>
  </div>
)

// Использование
<Card title="Профиль">
  <p>Содержимое карточки</p>
</Card>

// Передача всех пропсов (spread)
const Input = ({ className, ...rest }: React.InputHTMLAttributes<HTMLInputElement>) => (
  <input className={`input ${className}`} {...rest} />
)
```

---

## 🔹 Условный рендеринг

```tsx
// Ранний возврат
if (loading) return <Spinner />
if (error) return <ErrorMessage error={error} />

// Тернарный оператор
{isLoggedIn ? <UserMenu /> : <LoginButton />}

// Short-circuit
{hasPermission && <AdminPanel />}

// Switch-like через object
const statusComponents = {
  loading: <Spinner />,
  error: <ErrorMessage />,
  success: <DataTable />
}
return statusComponents[status] ?? null
```

---

## 🔗 Смотри также
> - [[React_Hooks]] — useState, useEffect и другие хуки
> - [[TypeScript]] — типизация пропсов
> - [[CSS_Basics]] — CSS Modules, styled-components
