# React — Router

> **Теги:** #frontend #react #router #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[React_Index]]

---

## 🔹 Настройка (React Router v6+)

```tsx
// main.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom'

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: 'users',
        element: <UsersLayout />,
        children: [
          { index: true, element: <UsersList /> },
          { path: ':id', element: <UserDetail /> }
        ]
      },
      { path: 'about', element: <AboutPage /> },
      { path: '*', element: <NotFound /> }
    ]
  }
])

ReactDOM.createRoot(document.getElementById('root')!).render(
  <RouterProvider router={router} />
)
```

---

## 🔹 Навигация

```tsx
import { Link, NavLink, useNavigate, useParams, useSearchParams } from 'react-router-dom'

// Ссылки
<Link to="/users">Пользователи</Link>
<NavLink to="/about" className={({ isActive }) => isActive ? 'active' : ''}>
  О нас
</NavLink>

// Программная навигация
const navigate = useNavigate()
navigate('/users')
navigate('/users', { state: { from: 'dashboard' } })
navigate(-1)   // назад

// Параметры
const { id } = useParams<{ id: string }>()

// Query строка
const [searchParams, setSearchParams] = useSearchParams()
const query = searchParams.get('q') ?? ''
setSearchParams({ q: 'new query', page: '1' })
```

---

## 🔹 Outlet и Layout

```tsx
// RootLayout.tsx
import { Outlet } from 'react-router-dom'

const RootLayout = () => (
  <div>
    <Header />
    <main>
      <Outlet />  {/* здесь рендерится дочерний маршрут */}
    </main>
    <Footer />
  </div>
)
```

---

## 🔹 Защищённые маршруты

```tsx
const ProtectedRoute = ({ children }: { children: ReactNode }) => {
  const { user } = useAuth()
  const location = useLocation()

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />
  }
  return children
}

// В роутере
{ path: 'admin', element: <ProtectedRoute><AdminPage /></ProtectedRoute> }
```

---

## 🔗 Смотри также
> - [[React_Core]] — компоненты
> - [[React_State]] — глобальное состояние с роутингом
