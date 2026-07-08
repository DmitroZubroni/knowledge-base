# React — Хуки

> **Теги:** #frontend #react #hooks #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[React_Index]]

---

## 🔹 useState

```tsx
const [count, setCount] = useState(0)
const [user, setUser]   = useState<User | null>(null)

// Обновление на основе предыдущего значения
setCount(prev => prev + 1)  // безопасно в event handlers

// Обновление объекта — spread!
setUser(prev => ({ ...prev!, name: 'Новое имя' }))

// Ленивая инициализация (дорогое вычисление)
const [data] = useState(() => computeExpensiveValue())
```

---

## 🔹 useEffect

```tsx
// Зависимости определяют когда эффект перезапускается
useEffect(() => {
  // выполняется после каждого рендера
})

useEffect(() => {
  // выполняется один раз при монтировании
}, [])

useEffect(() => {
  // выполняется при изменении userId
  fetchUser(userId).then(setUser)
}, [userId])

// Cleanup — важно для подписок, таймеров
useEffect(() => {
  const subscription = eventBus.subscribe('update', handler)
  const timer = setInterval(tick, 1000)

  return () => {
    subscription.unsubscribe()
    clearInterval(timer)
  }
}, [])

// Fetch с отменой
useEffect(() => {
  const controller = new AbortController()

  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setUser)
    .catch(err => { if (err.name !== 'AbortError') setError(err) })

  return () => controller.abort()
}, [id])
```

---

## 🔹 useRef

```tsx
// Ссылка на DOM-элемент
const inputRef = useRef<HTMLInputElement>(null)
<input ref={inputRef} />
inputRef.current?.focus()

// Хранение значения без ре-рендера
const timerRef = useRef<number | null>(null)
timerRef.current = setInterval(tick, 1000)
// Не вызывает ре-рендер при изменении (в отличие от useState)

// Хранение предыдущего значения
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>()
  useEffect(() => { ref.current = value })
  return ref.current
}
```

---

## 🔹 useCallback и useMemo

```tsx
// useCallback — мемоизация функции (стабильная ссылка)
const handleSubmit = useCallback(async (data: FormData) => {
  await submitForm(data)
  navigate('/success')
}, [navigate])  // пересоздаётся только если navigate изменился

// useMemo — мемоизация вычисления
const filteredUsers = useMemo(
  () => users.filter(u => u.active && u.role === selectedRole),
  [users, selectedRole]
)

// Когда использовать:
// useCallback — передаёшь функцию в React.memo компонент или deps массив
// useMemo    — дорогое вычисление (>5ms), зависит от пропсов/стейта
// Не оборачивай всё — измерь сначала!
```

---

## 🔹 Кастомные хуки

```tsx
// Логика вынесена из компонента в переиспользуемый хук
function useFetch<T>(url: string) {
  const [data,    setData]    = useState<T | null>(null)
  const [loading, setLoading] = useState(true)
  const [error,   setError]   = useState<Error | null>(null)

  useEffect(() => {
    const controller = new AbortController()
    setLoading(true)

    fetch(url, { signal: controller.signal })
      .then(r => { if (!r.ok) throw new Error(r.statusText); return r.json() })
      .then(setData)
      .catch(err => { if (err.name !== 'AbortError') setError(err) })
      .finally(() => setLoading(false))

    return () => controller.abort()
  }, [url])

  return { data, loading, error }
}

// Использование
const { data: users, loading, error } = useFetch<User[]>('/api/users')
```

---

## 🔗 Смотри также
> - [[React_Core]] — компоненты и пропсы
> - [[React_State]] — глобальное состояние
> - [[JS_Async]] — Promise, async/await на которых строятся хуки
