# JavaScript — Асинхронность

> **Теги:** #frontend #javascript #async #promise #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[JS_Index]]

---

## 🔹 Event Loop

JS — однопоточный. Event Loop позволяет не блокировать поток при ожидании.

```
Call Stack           Web APIs           Task Queue
──────────           ────────           ──────────
main()               setTimeout         callback1
fetch()       →      fetch API    →     callback2
               (выполняется браузером)

Event Loop: если Call Stack пуст → берёт задачу из Queue
```

```javascript
console.log('1')          // синхронно → сразу
setTimeout(() => {
  console.log('3')        // макрозадача → после всего синхронного
}, 0)
Promise.resolve().then(() => {
  console.log('2')        // микрозадача → раньше макрозадачи
})
console.log('4')          // синхронно → сразу

// Вывод: 1, 4, 2, 3
// Порядок: синхронный → микрозадачи (Promise) → макрозадачи (setTimeout)
```

---

## 🔹 Promise

```javascript
// Создание
const promise = new Promise((resolve, reject) => {
  if (success) resolve(data)
  else reject(new Error('Ошибка'))
})

// Состояния: pending → fulfilled | rejected

// Цепочка
fetch('/api/users')
  .then(response => response.json())    // трансформация
  .then(users => users.filter(u => u.active))
  .catch(error => {                     // любая ошибка в цепочке
    console.error(error)
    return []  // восстановление после ошибки
  })
  .finally(() => setLoading(false))     // всегда

// Комбинаторы
Promise.all([p1, p2, p3])         // ждёт все, падает если один упал
Promise.allSettled([p1, p2, p3])  // ждёт все, возвращает статус каждого
Promise.race([p1, p2, p3])        // возвращает первый resolved/rejected
Promise.any([p1, p2, p3])         // первый fulfilled, игнорирует rejected
```

---

## 🔹 async / await

```javascript
async function loadUserData(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`)

    if (!response.ok) {
      throw new Error(`HTTP error: ${response.status}`)
    }

    const user = await response.json()
    const orders = await fetch(`/api/orders?userId=${userId}`)
      .then(r => r.json())

    return { user, orders }
  } catch (error) {
    console.error('Ошибка загрузки:', error)
    throw error  // пробрасываем выше
  }
}

// Параллельная загрузка (не await друг за другом!)
async function loadAll(ids) {
  const promises = ids.map(id => fetch(`/api/users/${id}`).then(r => r.json()))
  return Promise.all(promises)  // параллельно, не последовательно
}
```

---

## 🔹 Fetch API

```javascript
// GET
const data = await fetch('/api/users').then(r => r.json())

// POST с JSON
const response = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'Дмитрий', email: 'd@example.com' })
})
const created = await response.json()

// С авторизацией
const response = await fetch('/api/profile', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`
  }
})

// Отмена запроса
const controller = new AbortController()
fetch('/api/data', { signal: controller.signal })
controller.abort()  // отменить
```

---

## 🔹 Обработка ошибок

```javascript
// fetch НЕ бросает ошибку при 4xx/5xx!
const response = await fetch('/api/users/999')
response.ok          // false при 404
response.status      // 404
await response.json()// { error: 'Not found' }

// Обёртка для правильной обработки
async function apiFetch(url, options = {}) {
  const response = await fetch(url, options)
  if (!response.ok) {
    const error = await response.json().catch(() => ({}))
    throw Object.assign(new Error(error.message || 'API Error'), {
      status: response.status,
      data: error
    })
  }
  return response.json()
}
```

---

## 🔗 Смотри также
> - [[JS_Functions]] — callback-функции
> - [[React_Hooks]] — useEffect, data fetching в React
