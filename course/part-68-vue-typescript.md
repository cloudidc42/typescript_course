# ตอนที่ 68: Vue.js กับ TypeScript

## บทนำ

Vue 3 รองรับ TypeScript อย่างเป็นทางการและมี Composition API ที่ทำงานได้ดีกับ TypeScript อย่างมาก บทนี้จะครอบคลุมการใช้ TypeScript กับ Vue 3 ตั้งแต่พื้นฐานไปจนถึงระดับสูง

## 68.1 Vue 3 + TypeScript Setup

### การสร้างโปรเจกต์

```bash
npm create vue@latest
# เลือก: Add TypeScript? Yes
# เลือก: Add Vue Router? Yes
# เลือก: Add Pinia? Yes
# เลือก: Add ESLint? Yes

cd my-vue-app
npm install
```

### tsconfig.json สำหรับ Vue

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.node.json" },
    { "path": "./tsconfig.app.json" }
  ]
}
```

```json
// tsconfig.app.json
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["env.d.ts", "src/**/*", "src/**/*.vue"],
  "exclude": ["src/**/__tests__/*"]
}
```

## 68.2 Composition API พร้อม Types

### พื้นฐาน setup()

```typescript
// component.vue
<script lang="ts">
import { defineComponent, ref, reactive, computed, watch, onMounted } from 'vue'

interface User {
  id: number
  name: string
  email: string
  age: number
}

export default defineComponent({
  name: 'UserProfile',
  
  setup() {
    // ref - ค่า primitive
    const count = ref<number>(0)
    const username = ref<string>('')
    const isLoading = ref<boolean>(false)
    
    // reactive - object
    const user = reactive<User>({
      id: 1,
      name: 'สมชาย',
      email: 'somchai@example.com',
      age: 30
    })
    
    // computed
    const fullDescription = computed<string>(() => {
      return `${user.name} (${user.age} ปี)`
    })
    
    // methods
    const increment = (): void => {
      count.value++
    }
    
    const updateUser = (newData: Partial<User>): void => {
      Object.assign(user, newData)
    }
    
    // lifecycle
    onMounted(() => {
      console.log('Component mounted')
    })
    
    return {
      count,
      username,
      isLoading,
      user,
      fullDescription,
      increment,
      updateUser
    }
  }
})
</script>
```

### Script Setup (แนะนำ)

```typescript
// UserCard.vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'moderator'
  avatar?: string
}

// ตัวแปร
const currentUser = ref<User | null>(null)
const isEditing = ref<boolean>(false)
const editForm = ref<Partial<User>>({})

// computed
const userDisplayName = computed<string>(() => {
  return currentUser.value?.name ?? 'ผู้ใช้ที่ไม่รู้จัก'
})

const isAdmin = computed<boolean>(() => {
  return currentUser.value?.role === 'admin'
})

// watch
watch(currentUser, (newUser, oldUser) => {
  if (newUser) {
    console.log(`เปลี่ยนผู้ใช้จาก ${oldUser?.name} เป็น ${newUser.name}`)
    editForm.value = { ...newUser }
  }
}, { immediate: true })

// methods
const startEditing = (): void => {
  isEditing.value = true
  editForm.value = { ...currentUser.value }
}

const saveChanges = (): void => {
  if (currentUser.value && editForm.value) {
    currentUser.value = { ...currentUser.value, ...editForm.value }
    isEditing.value = false
  }
}

const cancelEditing = (): void => {
  isEditing.value = false
  editForm.value = {}
}

onMounted(() => {
  // โหลดข้อมูลผู้ใช้
  currentUser.value = {
    id: 1,
    name: 'สมชาย ใจดี',
    email: 'somchai@example.com',
    role: 'admin'
  }
})
</script>

<template>
  <div class="user-card">
    <div v-if="!isEditing">
      <h2>{{ userDisplayName }}</h2>
      <p>{{ currentUser?.email }}</p>
      <span :class="'role-' + currentUser?.role">{{ currentUser?.role }}</span>
      <span v-if="isAdmin" class="admin-badge">Admin</span>
      <button @click="startEditing">แก้ไข</button>
    </div>
    
    <form v-else @submit.prevent="saveChanges">
      <input v-model="editForm.name" type="text" placeholder="ชื่อ">
      <input v-model="editForm.email" type="email" placeholder="อีเมล">
      <button type="submit">บันทึก</button>
      <button type="button" @click="cancelEditing">ยกเลิก</button>
    </form>
  </div>
</template>
```

## 68.3 defineComponent

```typescript
<script lang="ts">
import { defineComponent, PropType, ref, computed } from 'vue'

interface TableColumn<T> {
  key: keyof T
  label: string
  sortable?: boolean
  formatter?: (value: T[keyof T]) => string
}

interface TableProps<T> {
  columns: TableColumn<T>[]
  data: T[]
  loading?: boolean
  emptyMessage?: string
}

export default defineComponent({
  name: 'DataTable',
  
  props: {
    columns: {
      type: Array as PropType<TableColumn<any>[]>,
      required: true
    },
    data: {
      type: Array as PropType<any[]>,
      required: true
    },
    loading: {
      type: Boolean,
      default: false
    },
    emptyMessage: {
      type: String,
      default: 'ไม่มีข้อมูล'
    }
  },
  
  emits: ['row-click', 'sort-change'],
  
  setup(props, { emit }) {
    const sortColumn = ref<string>('')
    const sortDirection = ref<'asc' | 'desc'>('asc')
    
    const sortedData = computed(() => {
      if (!sortColumn.value) return props.data
      
      return [...props.data].sort((a, b) => {
        const aVal = a[sortColumn.value]
        const bVal = b[sortColumn.value]
        
        const comparison = String(aVal).localeCompare(String(bVal), 'th')
        return sortDirection.value === 'asc' ? comparison : -comparison
      })
    })
    
    const handleSort = (column: string): void => {
      if (sortColumn.value === column) {
        sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
      } else {
        sortColumn.value = column
        sortDirection.value = 'asc'
      }
      
      emit('sort-change', { column, direction: sortDirection.value })
    }
    
    const handleRowClick = (row: any): void => {
      emit('row-click', row)
    }
    
    return {
      sortColumn,
      sortDirection,
      sortedData,
      handleSort,
      handleRowClick
    }
  }
})
</script>
```

## 68.4 Props Typing

```typescript
// แบบที่ 1: defineProps กับ generic
<script setup lang="ts">
interface ButtonProps {
  label: string
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
  loading?: boolean
  icon?: string
  block?: boolean
}

const props = withDefaults(defineProps<ButtonProps>(), {
  variant: 'primary',
  size: 'md',
  disabled: false,
  loading: false,
  block: false
})

// computed จาก props
const buttonClasses = computed<string[]>(() => [
  'btn',
  `btn-${props.variant}`,
  `btn-${props.size}`,
  { 'btn-block': props.block },
  { 'btn-loading': props.loading }
].filter(Boolean) as string[])
</script>
```

```typescript
// แบบที่ 2: PropType
<script lang="ts">
import { defineComponent, PropType } from 'vue'

interface ChartData {
  labels: string[]
  datasets: Array<{
    label: string
    data: number[]
    backgroundColor: string | string[]
    borderColor?: string
  }>
}

interface ChartOptions {
  responsive?: boolean
  maintainAspectRatio?: boolean
  scales?: Record<string, any>
  plugins?: Record<string, any>
}

export default defineComponent({
  props: {
    chartData: {
      type: Object as PropType<ChartData>,
      required: true
    },
    chartOptions: {
      type: Object as PropType<ChartOptions>,
      default: () => ({ responsive: true })
    },
    type: {
      type: String as PropType<'bar' | 'line' | 'pie' | 'doughnut'>,
      default: 'bar'
    },
    height: {
      type: Number,
      default: 300
    }
  }
})
</script>
```

## 68.5 Emits Typing

```typescript
<script setup lang="ts">
interface FormSubmitEvent {
  data: Record<string, any>
  timestamp: Date
}

interface ValidationError {
  field: string
  message: string
}

// Define emits with TypeScript
const emit = defineEmits<{
  submit: [event: FormSubmitEvent]
  cancel: []
  error: [errors: ValidationError[]]
  change: [field: string, value: any]
}>()

// หรือแบบ object syntax
const emitAlt = defineEmits({
  submit: (event: FormSubmitEvent) => {
    return event instanceof Object
  },
  cancel: null,
  error: (errors: ValidationError[]) => Array.isArray(errors)
})

// การใช้งาน
const handleSubmit = (formData: Record<string, any>): void => {
  // Validate
  const errors = validateForm(formData)
  if (errors.length > 0) {
    emit('error', errors)
    return
  }
  
  emit('submit', {
    data: formData,
    timestamp: new Date()
  })
}

const validateForm = (data: Record<string, any>): ValidationError[] => {
  const errors: ValidationError[] = []
  
  if (!data.name) {
    errors.push({ field: 'name', message: 'กรุณากรอกชื่อ' })
  }
  
  if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
    errors.push({ field: 'email', message: 'รูปแบบอีเมลไม่ถูกต้อง' })
  }
  
  return errors
}
</script>
```

## 68.6 ref และ reactive Types

```typescript
<script setup lang="ts">
import { ref, reactive, Ref, UnwrapRef, toRef, toRefs } from 'vue'

// ref types
const count: Ref<number> = ref(0)
const name: Ref<string> = ref('')
const user: Ref<User | null> = ref<User | null>(null)

// ref กับ array
const items: Ref<string[]> = ref([])
const selectedIds: Ref<Set<number>> = ref(new Set())

// reactive object
const state = reactive({
  loading: false,
  error: null as string | null,
  data: [] as User[],
  pagination: {
    page: 1,
    perPage: 10,
    total: 0
  }
})

// ชนิดข้อมูลของ state
type AppState = typeof state

// toRef - แปลง reactive property เป็น ref
const loading = toRef(state, 'loading')

// toRefs - แปลง reactive เป็น refs ทั้งหมด
const { loading: loadingRef, error: errorRef } = toRefs(state)

// Generic ref function
function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  const stored = localStorage.getItem(key)
  const initialValue: T = stored ? JSON.parse(stored) : defaultValue
  const value = ref<T>(initialValue)
  
  watch(value, (newVal) => {
    localStorage.setItem(key, JSON.stringify(newVal))
  }, { deep: true })
  
  return value as Ref<T>
}

const theme = useLocalStorage<'light' | 'dark'>('theme', 'light')
const savedUser = useLocalStorage<User | null>('user', null)
</script>
```

## 68.7 Computed Types

```typescript
<script setup lang="ts">
import { computed, ComputedRef, WritableComputedRef } from 'vue'

interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
  discount?: number
}

const cartItems = ref<CartItem[]>([])
const taxRate = ref<number>(0.07)
const couponDiscount = ref<number>(0)

// Read-only computed
const subtotal: ComputedRef<number> = computed(() => {
  return cartItems.value.reduce((sum, item) => {
    const itemPrice = item.discount 
      ? item.price * (1 - item.discount / 100)
      : item.price
    return sum + (itemPrice * item.quantity)
  }, 0)
})

const tax: ComputedRef<number> = computed(() => {
  return subtotal.value * taxRate.value
})

const total: ComputedRef<number> = computed(() => {
  return subtotal.value + tax.value - couponDiscount.value
})

const itemCount: ComputedRef<number> = computed(() => {
  return cartItems.value.reduce((count, item) => count + item.quantity, 0)
})

// Writable computed
const couponCode: WritableComputedRef<string> = computed({
  get: () => couponDiscount.value > 0 ? `DISCOUNT${couponDiscount.value}` : '',
  set: (code: string) => {
    if (code === 'SAVE100') couponDiscount.value = 100
    else if (code === 'SAVE50') couponDiscount.value = 50
    else couponDiscount.value = 0
  }
})

// Computed กับ object
const cartSummary = computed(() => ({
  itemCount: itemCount.value,
  subtotal: subtotal.value,
  tax: tax.value,
  discount: couponDiscount.value,
  total: total.value,
  currency: 'THB'
}))

// Computed กับ filter
const expensiveItems = computed<CartItem[]>(() => {
  return cartItems.value.filter(item => item.price > 1000)
})

const sortedByPrice = computed<CartItem[]>(() => {
  return [...cartItems.value].sort((a, b) => a.price - b.price)
})
</script>
```

## 68.8 Watch พร้อม Types

```typescript
<script setup lang="ts">
import { watch, watchEffect, WatchStopHandle, WatchOptions } from 'vue'

interface SearchState {
  query: string
  filters: {
    category: string
    minPrice: number
    maxPrice: number
  }
  results: Product[]
  loading: boolean
}

const searchState = reactive<SearchState>({
  query: '',
  filters: {
    category: '',
    minPrice: 0,
    maxPrice: Infinity
  },
  results: [],
  loading: false
})

// watch single ref
watch(
  () => searchState.query,
  async (newQuery: string, oldQuery: string) => {
    if (newQuery.length < 2) return
    
    searchState.loading = true
    try {
      searchState.results = await searchProducts(newQuery)
    } finally {
      searchState.loading = false
    }
  },
  {
    debounce: 300 // หน่วงเวลา 300ms
  }
)

// watch multiple sources
watch(
  [() => searchState.filters.category, () => searchState.filters.minPrice],
  ([newCategory, newMinPrice]: [string, number], [oldCategory, oldMinPrice]: [string, number]) => {
    console.log(`Filter changed: ${oldCategory} -> ${newCategory}`)
    console.log(`MinPrice changed: ${oldMinPrice} -> ${newMinPrice}`)
    refreshSearch()
  }
)

// watch deep
const watchOptions: WatchOptions = {
  deep: true,
  immediate: true,
  flush: 'post'
}

watch(
  () => searchState.filters,
  (newFilters) => {
    console.log('Filters changed:', newFilters)
    refreshSearch()
  },
  watchOptions
)

// watchEffect
const stopEffect: WatchStopHandle = watchEffect(() => {
  document.title = `ค้นหา: ${searchState.query} (${searchState.results.length} ผลลัพธ์)`
})

// Stop watching manually
onUnmounted(() => {
  stopEffect()
})

const refreshSearch = async (): Promise<void> => {
  if (!searchState.query) return
  searchState.results = await searchProducts(searchState.query, searchState.filters)
}
</script>
```

## 68.9 Composables

```typescript
// composables/useApi.ts
import { ref, Ref } from 'vue'

interface UseApiOptions<T> {
  immediate?: boolean
  defaultData?: T
  onSuccess?: (data: T) => void
  onError?: (error: Error) => void
}

interface UseApiReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  execute: (...args: any[]) => Promise<void>
  reset: () => void
}

export function useApi<T>(
  apiFunction: (...args: any[]) => Promise<T>,
  options: UseApiOptions<T> = {}
): UseApiReturn<T> {
  const data = ref<T | null>(options.defaultData ?? null)
  const loading = ref<boolean>(false)
  const error = ref<Error | null>(null)
  
  const execute = async (...args: any[]): Promise<void> => {
    loading.value = true
    error.value = null
    
    try {
      data.value = await apiFunction(...args)
      options.onSuccess?.(data.value as T)
    } catch (err) {
      error.value = err instanceof Error ? err : new Error(String(err))
      options.onError?.(error.value)
    } finally {
      loading.value = false
    }
  }
  
  const reset = (): void => {
    data.value = null
    loading.value = false
    error.value = null
  }
  
  if (options.immediate) {
    execute()
  }
  
  return {
    data: data as Ref<T | null>,
    loading,
    error,
    execute,
    reset
  }
}

// composables/useForm.ts
export interface FormField<T = string> {
  value: T
  error: string
  touched: boolean
  dirty: boolean
}

export type FormFields<T> = {
  [K in keyof T]: FormField<T[K]>
}

export function useForm<T extends Record<string, any>>(initialValues: T) {
  type Fields = FormFields<T>
  
  const fields = reactive<Fields>(
    Object.fromEntries(
      Object.entries(initialValues).map(([key, value]) => [
        key,
        { value, error: '', touched: false, dirty: false }
      ])
    ) as Fields
  )
  
  const isValid = computed(() => {
    return Object.values(fields).every((field: any) => !field.error)
  })
  
  const isDirty = computed(() => {
    return Object.values(fields).some((field: any) => field.dirty)
  })
  
  const setFieldValue = <K extends keyof T>(field: K, value: T[K]): void => {
    (fields[field] as FormField<T[K]>).value = value
    ;(fields[field] as FormField<T[K]>).dirty = true
  }
  
  const setFieldError = (field: keyof T, error: string): void => {
    ;(fields[field] as FormField).error = error
  }
  
  const touchField = (field: keyof T): void => {
    ;(fields[field] as FormField).touched = true
  }
  
  const getValues = (): T => {
    return Object.fromEntries(
      Object.entries(fields).map(([key, field]) => [key, (field as FormField).value])
    ) as T
  }
  
  const reset = (): void => {
    Object.entries(initialValues).forEach(([key, value]) => {
      ;(fields[key as keyof T] as FormField).value = value
      ;(fields[key as keyof T] as FormField).error = ''
      ;(fields[key as keyof T] as FormField).touched = false
      ;(fields[key as keyof T] as FormField).dirty = false
    })
  }
  
  return {
    fields,
    isValid,
    isDirty,
    setFieldValue,
    setFieldError,
    touchField,
    getValues,
    reset
  }
}

// composables/usePagination.ts
export function usePagination<T>(
  dataFetcher: (page: number, perPage: number) => Promise<{ data: T[]; total: number }>,
  options: { perPage?: number } = {}
) {
  const currentPage = ref<number>(1)
  const perPage = ref<number>(options.perPage ?? 10)
  const total = ref<number>(0)
  const items = ref<T[]>([])
  const loading = ref<boolean>(false)
  
  const totalPages = computed(() => Math.ceil(total.value / perPage.value))
  const hasNextPage = computed(() => currentPage.value < totalPages.value)
  const hasPrevPage = computed(() => currentPage.value > 1)
  
  const fetchPage = async (page: number): Promise<void> => {
    loading.value = true
    try {
      const result = await dataFetcher(page, perPage.value)
      items.value = result.data
      total.value = result.total
      currentPage.value = page
    } finally {
      loading.value = false
    }
  }
  
  const nextPage = (): Promise<void> => {
    if (hasNextPage.value) return fetchPage(currentPage.value + 1)
    return Promise.resolve()
  }
  
  const prevPage = (): Promise<void> => {
    if (hasPrevPage.value) return fetchPage(currentPage.value - 1)
    return Promise.resolve()
  }
  
  const goToPage = (page: number): Promise<void> => {
    if (page >= 1 && page <= totalPages.value) return fetchPage(page)
    return Promise.resolve()
  }
  
  // โหลดหน้าแรกทันที
  fetchPage(1)
  
  return {
    items: items as Ref<T[]>,
    currentPage,
    perPage,
    total,
    totalPages,
    loading,
    hasNextPage,
    hasPrevPage,
    fetchPage,
    nextPage,
    prevPage,
    goToPage
  }
}
```

## 68.10 Pinia Store พร้อม TypeScript

```typescript
// stores/userStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
  preferences: UserPreferences
}

interface UserPreferences {
  theme: 'light' | 'dark'
  language: 'th' | 'en'
  notifications: boolean
}

interface AuthState {
  token: string | null
  refreshToken: string | null
  expiresAt: Date | null
}

// Option store (Class-like syntax)
export const useUserStore = defineStore('user', {
  state: (): { 
    currentUser: User | null
    auth: AuthState
    users: User[]
    loading: boolean
    error: string | null
  } => ({
    currentUser: null,
    auth: {
      token: localStorage.getItem('token'),
      refreshToken: localStorage.getItem('refreshToken'),
      expiresAt: null
    },
    users: [],
    loading: false,
    error: null
  }),
  
  getters: {
    isAuthenticated: (state): boolean => !!state.auth.token,
    isAdmin: (state): boolean => state.currentUser?.role === 'admin',
    userCount: (state): number => state.users.length,
    currentTheme: (state): 'light' | 'dark' => 
      state.currentUser?.preferences.theme ?? 'light'
  },
  
  actions: {
    async login(email: string, password: string): Promise<void> {
      this.loading = true
      this.error = null
      
      try {
        const response = await authApi.login({ email, password })
        this.auth = {
          token: response.token,
          refreshToken: response.refreshToken,
          expiresAt: new Date(response.expiresAt)
        }
        this.currentUser = response.user
        
        localStorage.setItem('token', response.token)
        localStorage.setItem('refreshToken', response.refreshToken)
      } catch (err) {
        this.error = err instanceof Error ? err.message : 'เข้าสู่ระบบไม่สำเร็จ'
        throw err
      } finally {
        this.loading = false
      }
    },
    
    logout(): void {
      this.currentUser = null
      this.auth = { token: null, refreshToken: null, expiresAt: null }
      localStorage.removeItem('token')
      localStorage.removeItem('refreshToken')
    },
    
    async updatePreferences(prefs: Partial<UserPreferences>): Promise<void> {
      if (!this.currentUser) return
      
      const updated = await userApi.updatePreferences(this.currentUser.id, prefs)
      this.currentUser = { ...this.currentUser, preferences: updated }
    },
    
    async fetchUsers(): Promise<void> {
      this.loading = true
      try {
        this.users = await userApi.getAll()
      } catch (err) {
        this.error = err instanceof Error ? err.message : 'โหลดข้อมูลไม่สำเร็จ'
      } finally {
        this.loading = false
      }
    }
  }
})

// Setup store (Composition API syntax)
export const useCartStore = defineStore('cart', () => {
  interface CartItem {
    productId: number
    name: string
    price: number
    quantity: number
  }
  
  const items = ref<CartItem[]>([])
  const discount = ref<number>(0)
  
  const itemCount = computed(() => 
    items.value.reduce((sum, item) => sum + item.quantity, 0)
  )
  
  const subtotal = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )
  
  const total = computed(() => subtotal.value - discount.value)
  
  const addItem = (item: CartItem): void => {
    const existing = items.value.find(i => i.productId === item.productId)
    if (existing) {
      existing.quantity += item.quantity
    } else {
      items.value.push({ ...item })
    }
  }
  
  const removeItem = (productId: number): void => {
    const index = items.value.findIndex(i => i.productId === productId)
    if (index !== -1) {
      items.value.splice(index, 1)
    }
  }
  
  const updateQuantity = (productId: number, quantity: number): void => {
    const item = items.value.find(i => i.productId === productId)
    if (item) {
      if (quantity <= 0) {
        removeItem(productId)
      } else {
        item.quantity = quantity
      }
    }
  }
  
  const clearCart = (): void => {
    items.value = []
    discount.value = 0
  }
  
  return {
    items,
    discount,
    itemCount,
    subtotal,
    total,
    addItem,
    removeItem,
    updateQuantity,
    clearCart
  }
})
```

## 68.11 Vue Router พร้อม Types

```typescript
// router/index.ts
import { createRouter, createWebHistory, RouteRecordRaw, RouteLocationNormalized } from 'vue-router'
import { useUserStore } from '@/stores/userStore'

// Route meta types
declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    roles?: Array<'admin' | 'user'>
    title?: string
    breadcrumb?: string
  }
}

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    component: () => import('@/views/HomeView.vue'),
    meta: { title: 'หน้าหลัก' }
  },
  {
    path: '/products',
    component: () => import('@/views/ProductsView.vue'),
    meta: { title: 'สินค้า', requiresAuth: true },
    children: [
      {
        path: ':id',
        component: () => import('@/views/ProductDetailView.vue'),
        props: (route) => ({ id: Number(route.params.id) })
      }
    ]
  },
  {
    path: '/admin',
    component: () => import('@/views/AdminView.vue'),
    meta: { requiresAuth: true, roles: ['admin'], title: 'ผู้ดูแล' }
  }
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition
    return { top: 0 }
  }
})

// Navigation guards
router.beforeEach(async (
  to: RouteLocationNormalized,
  from: RouteLocationNormalized
) => {
  const userStore = useUserStore()
  
  // ตั้ง title
  if (to.meta.title) {
    document.title = `${to.meta.title} | My App`
  }
  
  // ตรวจสอบ auth
  if (to.meta.requiresAuth && !userStore.isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  
  // ตรวจสอบ roles
  if (to.meta.roles && !to.meta.roles.includes(userStore.currentUser?.role as any)) {
    return { path: '/unauthorized' }
  }
})

export default router

// การใช้งาน router ใน component
// views/ProductsView.vue
<script setup lang="ts">
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

// ดึง query params แบบ type-safe
const searchQuery = computed(() => route.query.q as string || '')
const currentPage = computed(() => Number(route.query.page) || 1)

const navigateToProduct = (id: number): void => {
  router.push({ name: 'product-detail', params: { id } })
}

const updateSearch = (query: string): void => {
  router.replace({
    query: { ...route.query, q: query, page: '1' }
  })
}
</script>
```

## 68.12 Nuxt.js พร้อม TypeScript

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  typescript: {
    strict: true,
    typeCheck: true
  },
  modules: ['@pinia/nuxt', '@nuxtjs/i18n'],
  runtimeConfig: {
    apiSecret: process.env.API_SECRET, // server-side only
    public: {
      apiBase: process.env.API_BASE_URL || 'https://api.example.com'
    }
  }
})

// types/index.d.ts
export interface Product {
  id: number
  name: string
  description: string
  price: number
  images: string[]
  category: ProductCategory
  tags: string[]
  stock: number
  rating: number
  reviewCount: number
}

export interface ProductCategory {
  id: number
  name: string
  slug: string
}

// composables/useProducts.ts (Nuxt auto-import)
export function useProducts() {
  const config = useRuntimeConfig()
  
  const fetchProduct = async (id: number): Promise<Product> => {
    const { data, error } = await useFetch<Product>(
      `${config.public.apiBase}/products/${id}`
    )
    
    if (error.value) throw error.value
    if (!data.value) throw new Error('ไม่พบสินค้า')
    
    return data.value
  }
  
  const { data: products, pending, error, refresh } = useLazyFetch<Product[]>(
    `${config.public.apiBase}/products`,
    {
      transform: (data: Product[]) => data.filter(p => p.stock > 0)
    }
  )
  
  return {
    products,
    pending,
    error,
    refresh,
    fetchProduct
  }
}

// pages/products/[id].vue
<script setup lang="ts">
const route = useRoute()
const { fetchProduct } = useProducts()

const productId = computed(() => Number(route.params.id))

const { data: product, pending, error } = await useAsyncData<Product>(
  `product-${productId.value}`,
  () => fetchProduct(productId.value)
)

useHead({
  title: computed(() => product.value?.name ?? 'สินค้า'),
  meta: [
    { name: 'description', content: computed(() => product.value?.description ?? '') }
  ]
})
</script>

<template>
  <div>
    <LoadingSpinner v-if="pending" />
    <ErrorMessage v-else-if="error" :message="error.message" />
    <ProductDetail v-else-if="product" :product="product" />
  </div>
</template>
```

## สรุป

Vue.js กับ TypeScript เป็นการรวมกันที่ทรงพลัง:

1. **Composition API** ทำงานได้ดีกับ TypeScript มากกว่า Options API
2. **Script Setup** ลดโค้ดซ้ำซ้อนและมี better type inference
3. **Pinia** เป็น state management ที่ออกแบบมาสำหรับ TypeScript
4. **Vue Router** รองรับ typed route meta และ params
5. **Composables** ช่วยแยกและ reuse logic แบบ type-safe
6. **Nuxt** ให้ fullstack TypeScript development ที่สมบูรณ์

การใช้ TypeScript กับ Vue 3 จะทำให้พัฒนา application ที่ปลอดภัย บำรุงรักษาง่าย และมี developer experience ที่ดีเยี่ยม
