# Vue — Компоненты и реактивность

> **Теги:** #frontend #vue #composition-api #конспект

> [!abstract] Навигация
> [[main]] | [[main Frontend]] | [[Vue_Index]]

---

## 🔹 Single File Component (SFC)

```vue
<!-- UserCard.vue -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

interface Props {
  userId: number
}
const props = defineProps<Props>()
const emit  = defineEmits<{ delete: [id: number] }>()

const user    = ref<User | null>(null)
const loading = ref(true)

const fullName = computed(() => `${user.value?.firstName} ${user.value?.lastName}`)

onMounted(async () => {
  user.value  = await fetchUser(props.userId)
  loading.value = false
})
</script>

<template>
  <div v-if="loading" class="spinner" />
  <div v-else-if="user" class="card">
    <h2>{{ fullName }}</h2>
    <button @click="emit('delete', props.userId)">Удалить</button>
  </div>
</template>

<style scoped>
.card { padding: 1rem; border-radius: 8px; }
</style>
```

---

## 🔹 Реактивность: ref и reactive

```typescript
import { ref, reactive, computed, watch, watchEffect } from 'vue'

// ref — для примитивов и объектов (доступ через .value)
const count = ref(0)
count.value++

// reactive — для объектов (без .value)
const form = reactive({ name: '', email: '', age: 0 })
form.name = 'Дмитрий'

// computed — вычисляемое значение с кешем
const isValid = computed(() => form.name.length > 0 && form.email.includes('@'))

// watch — реакция на изменения
watch(count, (newVal, oldVal) => {
  console.log(`${oldVal} → ${newVal}`)
})
watch(() => form.name, (name) => validateName(name))

// watchEffect — автоматически отслеживает зависимости
watchEffect(() => {
  console.log(`Count: ${count.value}`)  // запустится при изменении count
})
```

---

## 🔹 Директивы

```html
<!-- Условный рендеринг -->
<div v-if="isAdmin">Только для админов</div>
<div v-else-if="isUser">Для пользователей</div>
<div v-else>Для всех</div>
<div v-show="visible"><!-- скрывает через display:none --></div>

<!-- Списки -->
<ul>
  <li v-for="(item, index) in items" :key="item.id">
    {{ index }}. {{ item.name }}
  </li>
</ul>

<!-- Биндинг атрибутов -->
<img :src="imageUrl" :alt="imageAlt">
<button :disabled="isLoading" :class="{ active: isActive, 'text-red': hasError }">
<div :style="{ color: textColor, fontSize: fontSize + 'px' }">

<!-- События -->
<button @click="handleClick">
<input @input="value = $event.target.value" @keyup.enter="submit">
<form @submit.prevent="handleSubmit">

<!-- v-model — двустороннее связывание -->
<input v-model="searchQuery">
<select v-model="selectedRole">
<input type="checkbox" v-model="isAccepted">
```

---

## 🔗 Смотри также
> - [[Vue_Router]] — маршрутизация в Vue
> - [[Vue_Pinia]] — управление состоянием
> - [[TypeScript]] — типизация компонентов
