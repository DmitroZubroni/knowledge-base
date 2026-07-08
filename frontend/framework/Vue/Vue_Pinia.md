# Pinia — State Management

> **Теги:** #frontend #vue #pinia #state #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Vue_Index]]

---

## 🔹 Создание стора

```typescript
// stores/users.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUsersStore = defineStore('users', () => {
  // State
  const users       = ref<User[]>([])
  const loading     = ref(false)
  const currentUser = ref<User | null>(null)

  // Getters (computed)
  const activeUsers = computed(() => users.value.filter(u => u.active))
  const count       = computed(() => users.value.length)

  // Actions
  async function fetchUsers() {
    loading.value = true
    try {
      users.value = await api.getUsers()
    } finally {
      loading.value = false
    }
  }

  async function deleteUser(id: number) {
    await api.deleteUser(id)
    users.value = users.value.filter(u => u.id !== id)
  }

  function $reset() {
    users.value   = []
    currentUser.value = null
  }

  return { users, loading, currentUser, activeUsers, count, fetchUsers, deleteUser, $reset }
})
```

---

## 🔹 Использование в компонентах

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useUsersStore } from '@/stores/users'

const store = useUsersStore()
// storeToRefs сохраняет реактивность при деструктуризации
const { users, loading, activeUsers } = storeToRefs(store)
const { fetchUsers, deleteUser } = store  // actions без storeToRefs

onMounted(fetchUsers)
</script>

<template>
  <div v-if="loading">Загрузка...</div>
  <ul v-else>
    <li v-for="user in activeUsers" :key="user.id">
      {{ user.name }}
      <button @click="deleteUser(user.id)">Удалить</button>
    </li>
  </ul>
</template>
```

---

## 🔗 Смотри также
> - [[Vue_Core]] — реактивность Vue
> - [[Vue_Router]] — навигация
