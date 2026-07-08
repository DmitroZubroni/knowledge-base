# React — Управление состоянием

> **Теги:** #frontend #react #state #redux #zustand #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[React_Index]]

---

## 🔹 Когда что использовать

```
Локальное состояние компонента    → useState
Состояние формы                   → react-hook-form
Состояние нескольких компонентов  → поднять вверх + props
Глобальное: несложное             → Context + useReducer
Глобальное: сложное               → Zustand или Redux Toolkit
Серверное состояние (API)         → TanStack Query (React Query)
```

---

## 🔹 Context API

```tsx
// Создание контекста
interface AuthContextType {
  user: User | null
  login: (credentials: Credentials) => Promise<void>
  logout: () => void
}

const AuthContext = createContext<AuthContextType | null>(null)

// Провайдер
export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const [user, setUser] = useState<User | null>(null)

  const login = async (credentials: Credentials) => {
    const user = await authService.login(credentials)
    setUser(user)
  }

  return (
    <AuthContext.Provider value={{ user, login, logout: () => setUser(null) }}>
      {children}
    </AuthContext.Provider>
  )
}

// Хук для использования
export const useAuth = () => {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth must be inside AuthProvider')
  return ctx
}
```

---

## 🔹 Zustand — лёгкий стейт-менеджер

```tsx
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'

interface CartStore {
  items: CartItem[]
  total: number
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  clear: () => void
}

const useCartStore = create<CartStore>()(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        total: 0,

        addItem: (item) => set(state => ({
          items: [...state.items, item],
          total: state.total + item.price
        })),

        removeItem: (id) => set(state => ({
          items: state.items.filter(i => i.id !== id),
          total: state.items.filter(i => i.id !== id).reduce((s, i) => s + i.price, 0)
        })),

        clear: () => set({ items: [], total: 0 })
      }),
      { name: 'cart-storage' }  // persist в localStorage
    )
  )
)

// Использование — подписка только на нужные поля
const items = useCartStore(state => state.items)
const addItem = useCartStore(state => state.addItem)
```

---

## 🔹 TanStack Query — серверное состояние

```tsx
// Запрос данных
const { data: users, isLoading, error } = useQuery({
  queryKey: ['users', { role: 'admin' }],  // кеш-ключ
  queryFn: () => api.getUsers({ role: 'admin' }),
  staleTime: 5 * 60 * 1000,               // 5 минут кешировать
  retry: 3
})

// Мутация
const { mutate: createUser, isPending } = useMutation({
  mutationFn: (data: CreateUserDto) => api.createUser(data),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] }) // инвалидировать кеш
    toast.success('Пользователь создан')
  },
  onError: (error) => toast.error(error.message)
})

// Вызов мутации
<button onClick={() => createUser({ name: 'Дмитрий' })}>
  Создать
</button>
```

---

## 🔗 Смотри также
> - [[React_Hooks]] — useState, useEffect, useContext
> - [[JS_Async]] — Promise, async/await
