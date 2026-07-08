# Vue Router

> **Теги:** #frontend #vue #router #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Vue_Index]]

---

## 🔹 Конфигурация

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('@/layouts/MainLayout.vue'),
      children: [
        { path: '', component: () => import('@/pages/HomePage.vue') },
        { path: 'users', component: () => import('@/pages/UsersPage.vue') },
        { path: 'users/:id', component: () => import('@/pages/UserDetail.vue') },
        {
          path: 'admin',
          component: () => import('@/pages/AdminPage.vue'),
          meta: { requiresAuth: true, role: 'admin' }
        }
      ]
    },
    { path: '/:pathMatch(.*)*', component: () => import('@/pages/NotFound.vue') }
  ]
})

// Navigation Guard
router.beforeEach((to, from) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.isLoggedIn) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
})

export default router
```

---

## 🔹 Использование в компонентах

```vue
<script setup lang="ts">
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route  = useRoute()

// Параметры
const userId = route.params.id as string
const page   = route.query.page as string ?? '1'

// Навигация
router.push('/users')
router.push({ name: 'user-detail', params: { id: 42 } })
router.replace('/home')  // без записи в историю
router.go(-1)            // назад
</script>

<template>
  <RouterLink to="/users">Пользователи</RouterLink>
  <RouterLink :to="{ name: 'user-detail', params: { id: user.id } }">
    {{ user.name }}
  </RouterLink>
  <RouterView />  <!-- куда рендерятся дочерние маршруты -->
</template>
```

---

## 🔗 Смотри также
> - [[Vue_Core]] — компоненты Vue
> - [[Vue_Pinia]] — состояние приложения
