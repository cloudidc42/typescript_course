# ตอนที่ 28: State Management ใน React กับ TypeScript

## บทนำ

การจัดการ State เป็นหัวใจสำคัญของแอปพลิเคชัน React ใน TypeScript เราสามารถใช้ประโยชน์จาก type system เพื่อทำให้ state management ปลอดภัยและตรวจสอบได้ในขณะ compile time ในบทนี้เราจะเรียนรู้เครื่องมือและวิธีการต่างๆ ในการจัดการ state ตั้งแต่ built-in hooks ของ React ไปจนถึง library ภายนอก

---

## 1. Context API กับ TypeScript

### 1.1 การสร้าง Context พื้นฐาน

```typescript
import React, { createContext, useContext, useState, ReactNode } from 'react';

// กำหนด type ของ state
interface UserState {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
}

// กำหนด type ของ context value
interface UserContextType {
  user: UserState | null;
  setUser: (user: UserState | null) => void;
  isLoggedIn: boolean;
}

// สร้าง context พร้อม default value
const UserContext = createContext<UserContextType | undefined>(undefined);

// สร้าง Provider component
interface UserProviderProps {
  children: ReactNode;
}

export const UserProvider: React.FC<UserProviderProps> = ({ children }) => {
  const [user, setUser] = useState<UserState | null>(null);

  const value: UserContextType = {
    user,
    setUser,
    isLoggedIn: user !== null,
  };

  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
};

// สร้าง custom hook เพื่อใช้ context
export const useUser = (): UserContextType => {
  const context = useContext(UserContext);
  if (context === undefined) {
    throw new Error('useUser ต้องใช้ภายใน UserProvider');
  }
  return context;
};
```

### 1.2 การใช้ Context ใน Component

```typescript
import React from 'react';
import { useUser } from './UserContext';

const UserProfile: React.FC = () => {
  const { user, setUser, isLoggedIn } = useUser();

  const handleLogout = () => {
    setUser(null);
  };

  if (!isLoggedIn || !user) {
    return <div>กรุณาเข้าสู่ระบบ</div>;
  }

  return (
    <div>
      <h2>โปรไฟล์ผู้ใช้</h2>
      <p>ชื่อ: {user.name}</p>
      <p>อีเมล: {user.email}</p>
      <p>บทบาท: {user.role}</p>
      <button onClick={handleLogout}>ออกจากระบบ</button>
    </div>
  );
};

export default UserProfile;
```

### 1.3 Context ที่ซับซ้อนขึ้น - Theme Context

```typescript
import React, { createContext, useContext, useState, useCallback, ReactNode } from 'react';

type Theme = 'light' | 'dark' | 'system';

interface ThemeColors {
  background: string;
  text: string;
  primary: string;
  secondary: string;
}

const themeColors: Record<Exclude<Theme, 'system'>, ThemeColors> = {
  light: {
    background: '#ffffff',
    text: '#000000',
    primary: '#007bff',
    secondary: '#6c757d',
  },
  dark: {
    background: '#1a1a1a',
    text: '#ffffff',
    primary: '#4da6ff',
    secondary: '#adb5bd',
  },
};

interface ThemeContextType {
  theme: Theme;
  colors: ThemeColors;
  toggleTheme: () => void;
  setTheme: (theme: Theme) => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

export const ThemeProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [theme, setThemeState] = useState<Theme>('light');

  const getColors = (t: Theme): ThemeColors => {
    if (t === 'system') {
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      return prefersDark ? themeColors.dark : themeColors.light;
    }
    return themeColors[t];
  };

  const toggleTheme = useCallback(() => {
    setThemeState(prev => prev === 'light' ? 'dark' : 'light');
  }, []);

  const setTheme = useCallback((newTheme: Theme) => {
    setThemeState(newTheme);
  }, []);

  return (
    <ThemeContext.Provider value={{ theme, colors: getColors(theme), toggleTheme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = (): ThemeContextType => {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme ต้องใช้ภายใน ThemeProvider');
  return context;
};
```

### 1.4 Multiple Contexts และ Composition

```typescript
import React, { ReactNode } from 'react';
import { UserProvider } from './UserContext';
import { ThemeProvider } from './ThemeContext';
import { NotificationProvider } from './NotificationContext';

// รวม providers หลายตัวเข้าด้วยกัน
interface AppProvidersProps {
  children: ReactNode;
}

export const AppProviders: React.FC<AppProvidersProps> = ({ children }) => {
  return (
    <ThemeProvider>
      <UserProvider>
        <NotificationProvider>
          {children}
        </NotificationProvider>
      </UserProvider>
    </ThemeProvider>
  );
};

// ใช้งานใน App
const App: React.FC = () => {
  return (
    <AppProviders>
      <Router />
    </AppProviders>
  );
};
```

---

## 2. useReducer กับ Typed Actions

### 2.1 พื้นฐาน useReducer

```typescript
import React, { useReducer } from 'react';

// กำหนด State type
interface CounterState {
  count: number;
  step: number;
  history: number[];
}

// กำหนด Action types (Discriminated Union)
type CounterAction =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' }
  | { type: 'SET_STEP'; payload: number }
  | { type: 'SET_COUNT'; payload: number }
  | { type: 'UNDO' };

// Initial state
const initialState: CounterState = {
  count: 0,
  step: 1,
  history: [],
};

// Reducer function ที่ type-safe
function counterReducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case 'INCREMENT':
      return {
        ...state,
        count: state.count + state.step,
        history: [...state.history, state.count],
      };
    case 'DECREMENT':
      return {
        ...state,
        count: state.count - state.step,
        history: [...state.history, state.count],
      };
    case 'RESET':
      return initialState;
    case 'SET_STEP':
      return { ...state, step: action.payload };
    case 'SET_COUNT':
      return {
        ...state,
        count: action.payload,
        history: [...state.history, state.count],
      };
    case 'UNDO': {
      const previousCount = state.history[state.history.length - 1];
      if (previousCount === undefined) return state;
      return {
        ...state,
        count: previousCount,
        history: state.history.slice(0, -1),
      };
    }
    default:
      // TypeScript จะตรวจสอบว่า action ทุกตัวถูก handle แล้ว
      const _exhaustiveCheck: never = action;
      return state;
  }
}

// Component ที่ใช้ reducer
const Counter: React.FC = () => {
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <div>
      <h2>Counter: {state.count}</h2>
      <p>Step: {state.step}</p>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
      <button 
        onClick={() => dispatch({ type: 'UNDO' })}
        disabled={state.history.length === 0}
      >
        Undo
      </button>
      <input
        type="number"
        value={state.step}
        onChange={e => dispatch({ type: 'SET_STEP', payload: Number(e.target.value) })}
      />
    </div>
  );
};
```

### 2.2 useReducer กับ Context

```typescript
import React, { createContext, useContext, useReducer, ReactNode, Dispatch } from 'react';

// Types สำหรับ Todo app
interface Todo {
  id: string;
  title: string;
  completed: boolean;
  priority: 'low' | 'medium' | 'high';
  createdAt: Date;
}

interface TodoState {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  loading: boolean;
  error: string | null;
}

type TodoAction =
  | { type: 'ADD_TODO'; payload: Omit<Todo, 'id' | 'createdAt'> }
  | { type: 'REMOVE_TODO'; payload: string }
  | { type: 'TOGGLE_TODO'; payload: string }
  | { type: 'UPDATE_TODO'; payload: Partial<Todo> & { id: string } }
  | { type: 'SET_FILTER'; payload: TodoState['filter'] }
  | { type: 'SET_LOADING'; payload: boolean }
  | { type: 'SET_ERROR'; payload: string | null }
  | { type: 'LOAD_TODOS'; payload: Todo[] }
  | { type: 'CLEAR_COMPLETED' };

const todoReducer = (state: TodoState, action: TodoAction): TodoState => {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [
          ...state.todos,
          {
            ...action.payload,
            id: crypto.randomUUID(),
            createdAt: new Date(),
          },
        ],
      };
    case 'REMOVE_TODO':
      return {
        ...state,
        todos: state.todos.filter(t => t.id !== action.payload),
      };
    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(t =>
          t.id === action.payload ? { ...t, completed: !t.completed } : t
        ),
      };
    case 'UPDATE_TODO':
      return {
        ...state,
        todos: state.todos.map(t =>
          t.id === action.payload.id ? { ...t, ...action.payload } : t
        ),
      };
    case 'SET_FILTER':
      return { ...state, filter: action.payload };
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    case 'SET_ERROR':
      return { ...state, error: action.payload };
    case 'LOAD_TODOS':
      return { ...state, todos: action.payload, loading: false };
    case 'CLEAR_COMPLETED':
      return {
        ...state,
        todos: state.todos.filter(t => !t.completed),
      };
    default:
      return state;
  }
};

// Context types
interface TodoContextType {
  state: TodoState;
  dispatch: Dispatch<TodoAction>;
  filteredTodos: Todo[];
  addTodo: (todo: Omit<Todo, 'id' | 'createdAt'>) => void;
  removeTodo: (id: string) => void;
  toggleTodo: (id: string) => void;
}

const TodoContext = createContext<TodoContextType | undefined>(undefined);

const initialTodoState: TodoState = {
  todos: [],
  filter: 'all',
  loading: false,
  error: null,
};

export const TodoProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(todoReducer, initialTodoState);

  const filteredTodos = state.todos.filter(todo => {
    switch (state.filter) {
      case 'active': return !todo.completed;
      case 'completed': return todo.completed;
      default: return true;
    }
  });

  const addTodo = (todo: Omit<Todo, 'id' | 'createdAt'>) => {
    dispatch({ type: 'ADD_TODO', payload: todo });
  };

  const removeTodo = (id: string) => {
    dispatch({ type: 'REMOVE_TODO', payload: id });
  };

  const toggleTodo = (id: string) => {
    dispatch({ type: 'TOGGLE_TODO', payload: id });
  };

  return (
    <TodoContext.Provider value={{ state, dispatch, filteredTodos, addTodo, removeTodo, toggleTodo }}>
      {children}
    </TodoContext.Provider>
  );
};

export const useTodo = () => {
  const context = useContext(TodoContext);
  if (!context) throw new Error('useTodo ต้องใช้ภายใน TodoProvider');
  return context;
};
```

---

## 3. Redux Toolkit กับ TypeScript

### 3.1 การติดตั้งและ Setup Store

```typescript
// store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import cartReducer from './slices/cartSlice';
import userReducer from './slices/userSlice';
import productReducer from './slices/productSlice';
import notificationReducer from './slices/notificationSlice';

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    user: userReducer,
    products: productReducer,
    notifications: notificationReducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['user/setLastLogin'],
      },
    }),
});

// Types สำหรับ state และ dispatch
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Typed hooks
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### 3.2 การสร้าง Slice

```typescript
// store/slices/cartSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
  category: string;
}

interface CartItem extends Product {
  quantity: number;
  addedAt: string;
}

interface CartState {
  items: CartItem[];
  isOpen: boolean;
  couponCode: string | null;
  discount: number;
}

const initialState: CartState = {
  items: [],
  isOpen: false,
  couponCode: null,
  discount: 0,
};

const cartSlice = createSlice({
  name: 'cart',
  initialState,
  reducers: {
    addToCart: (state, action: PayloadAction<Product>) => {
      const existingItem = state.items.find(item => item.id === action.payload.id);
      if (existingItem) {
        existingItem.quantity += 1;
      } else {
        state.items.push({
          ...action.payload,
          quantity: 1,
          addedAt: new Date().toISOString(),
        });
      }
    },
    removeFromCart: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(item => item.id !== action.payload);
    },
    updateQuantity: (state, action: PayloadAction<{ id: string; quantity: number }>) => {
      const item = state.items.find(i => i.id === action.payload.id);
      if (item) {
        if (action.payload.quantity <= 0) {
          state.items = state.items.filter(i => i.id !== action.payload.id);
        } else {
          item.quantity = action.payload.quantity;
        }
      }
    },
    clearCart: (state) => {
      state.items = [];
      state.couponCode = null;
      state.discount = 0;
    },
    toggleCart: (state) => {
      state.isOpen = !state.isOpen;
    },
    applyCoupon: (state, action: PayloadAction<{ code: string; discount: number }>) => {
      state.couponCode = action.payload.code;
      state.discount = action.payload.discount;
    },
    removeCoupon: (state) => {
      state.couponCode = null;
      state.discount = 0;
    },
  },
});

export const {
  addToCart,
  removeFromCart,
  updateQuantity,
  clearCart,
  toggleCart,
  applyCoupon,
  removeCoupon,
} = cartSlice.actions;

export default cartSlice.reducer;
```

### 3.3 Selectors

```typescript
// store/selectors/cartSelectors.ts
import { RootState } from '../index';
import { createSelector } from '@reduxjs/toolkit';

// Basic selectors
export const selectCartItems = (state: RootState) => state.cart.items;
export const selectCartIsOpen = (state: RootState) => state.cart.isOpen;
export const selectDiscount = (state: RootState) => state.cart.discount;

// Memoized selectors ด้วย createSelector
export const selectCartTotal = createSelector(
  selectCartItems,
  selectDiscount,
  (items, discount) => {
    const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    return subtotal * (1 - discount / 100);
  }
);

export const selectCartItemCount = createSelector(
  selectCartItems,
  (items) => items.reduce((sum, item) => sum + item.quantity, 0)
);

export const selectCartSubtotal = createSelector(
  selectCartItems,
  (items) => items.reduce((sum, item) => sum + item.price * item.quantity, 0)
);

export const selectItemById = (id: string) =>
  createSelector(
    selectCartItems,
    (items) => items.find(item => item.id === id)
  );

export const selectCartByCategory = createSelector(
  selectCartItems,
  (items) => {
    return items.reduce<Record<string, typeof items>>((acc, item) => {
      const category = item.category;
      if (!acc[category]) acc[category] = [];
      acc[category].push(item);
      return acc;
    }, {});
  }
);
```

### 3.4 Async Thunks

```typescript
// store/slices/productSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import { RootState } from '../index';

interface Product {
  id: string;
  name: string;
  price: number;
  description: string;
  imageUrl: string;
  category: string;
  rating: number;
  stock: number;
}

interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  minRating?: number;
  search?: string;
}

interface ProductState {
  items: Product[];
  selectedProduct: Product | null;
  filters: ProductFilters;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  totalCount: number;
  currentPage: number;
  itemsPerPage: number;
}

const initialState: ProductState = {
  items: [],
  selectedProduct: null,
  filters: {},
  status: 'idle',
  error: null,
  totalCount: 0,
  currentPage: 1,
  itemsPerPage: 10,
};

// Async thunk สำหรับ fetch products
export const fetchProducts = createAsyncThunk<
  { products: Product[]; total: number },
  { page: number; filters?: ProductFilters },
  { rejectValue: string; state: RootState }
>(
  'products/fetchProducts',
  async ({ page, filters }, { rejectWithValue, getState }) => {
    try {
      const state = getState();
      const params = new URLSearchParams({
        page: page.toString(),
        limit: state.products.itemsPerPage.toString(),
        ...filters,
      });

      const response = await fetch(`/api/products?${params}`);
      
      if (!response.ok) {
        return rejectWithValue('ไม่สามารถโหลดสินค้าได้');
      }

      const data = await response.json();
      return { products: data.products, total: data.total };
    } catch (error) {
      return rejectWithValue('เกิดข้อผิดพลาดในการเชื่อมต่อ');
    }
  }
);

// Async thunk สำหรับ fetch product detail
export const fetchProductById = createAsyncThunk<
  Product,
  string,
  { rejectValue: string }
>(
  'products/fetchById',
  async (productId, { rejectWithValue }) => {
    try {
      const response = await fetch(`/api/products/${productId}`);
      if (!response.ok) {
        return rejectWithValue(`ไม่พบสินค้า ID: ${productId}`);
      }
      return await response.json();
    } catch (error) {
      return rejectWithValue('เกิดข้อผิดพลาด');
    }
  }
);

// Async thunk สำหรับ create product
export const createProduct = createAsyncThunk<
  Product,
  Omit<Product, 'id'>,
  { rejectValue: string }
>(
  'products/create',
  async (productData, { rejectWithValue }) => {
    try {
      const response = await fetch('/api/products', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(productData),
      });

      if (!response.ok) {
        return rejectWithValue('ไม่สามารถสร้างสินค้าได้');
      }

      return await response.json();
    } catch (error) {
      return rejectWithValue('เกิดข้อผิดพลาด');
    }
  }
);

// Async thunk สำหรับ update product
export const updateProduct = createAsyncThunk<
  Product,
  { id: string; data: Partial<Product> },
  { rejectValue: string }
>(
  'products/update',
  async ({ id, data }, { rejectWithValue }) => {
    try {
      const response = await fetch(`/api/products/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });

      if (!response.ok) {
        return rejectWithValue('ไม่สามารถอัปเดตสินค้าได้');
      }

      return await response.json();
    } catch (error) {
      return rejectWithValue('เกิดข้อผิดพลาด');
    }
  }
);

const productSlice = createSlice({
  name: 'products',
  initialState,
  reducers: {
    setFilters: (state, action: PayloadAction<ProductFilters>) => {
      state.filters = action.payload;
      state.currentPage = 1;
    },
    setPage: (state, action: PayloadAction<number>) => {
      state.currentPage = action.payload;
    },
    clearSelectedProduct: (state) => {
      state.selectedProduct = null;
    },
  },
  extraReducers: (builder) => {
    builder
      // fetchProducts
      .addCase(fetchProducts.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload.products;
        state.totalCount = action.payload.total;
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload ?? 'เกิดข้อผิดพลาดที่ไม่ทราบสาเหตุ';
      })
      // fetchProductById
      .addCase(fetchProductById.fulfilled, (state, action) => {
        state.selectedProduct = action.payload;
      })
      // createProduct
      .addCase(createProduct.fulfilled, (state, action) => {
        state.items.push(action.payload);
        state.totalCount += 1;
      })
      // updateProduct
      .addCase(updateProduct.fulfilled, (state, action) => {
        const index = state.items.findIndex(p => p.id === action.payload.id);
        if (index !== -1) {
          state.items[index] = action.payload;
        }
      });
  },
});

export const { setFilters, setPage, clearSelectedProduct } = productSlice.actions;
export default productSlice.reducer;
```

### 3.5 การใช้งานใน Component

```typescript
import React, { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../store';
import { fetchProducts, setFilters } from '../store/slices/productSlice';
import { addToCart } from '../store/slices/cartSlice';
import { selectCartItemCount, selectCartTotal } from '../store/selectors/cartSelectors';

const ProductList: React.FC = () => {
  const dispatch = useAppDispatch();
  const { items, status, error, currentPage, filters } = useAppSelector(
    (state) => state.products
  );
  const cartItemCount = useAppSelector(selectCartItemCount);
  const cartTotal = useAppSelector(selectCartTotal);

  useEffect(() => {
    dispatch(fetchProducts({ page: currentPage, filters }));
  }, [dispatch, currentPage, filters]);

  if (status === 'loading') return <div>กำลังโหลด...</div>;
  if (status === 'failed') return <div>ข้อผิดพลาด: {error}</div>;

  return (
    <div>
      <header>
        <h1>สินค้า</h1>
        <div>ตะกร้า: {cartItemCount} รายการ | ยอดรวม: ฿{cartTotal.toFixed(2)}</div>
      </header>

      <div className="filters">
        <select
          onChange={(e) => dispatch(setFilters({ category: e.target.value || undefined }))}
        >
          <option value="">ทุกหมวดหมู่</option>
          <option value="electronics">อิเล็กทรอนิกส์</option>
          <option value="clothing">เสื้อผ้า</option>
          <option value="food">อาหาร</option>
        </select>
      </div>

      <div className="product-grid">
        {items.map((product) => (
          <div key={product.id} className="product-card">
            <img src={product.imageUrl} alt={product.name} />
            <h3>{product.name}</h3>
            <p>฿{product.price.toFixed(2)}</p>
            <p>คะแนน: {product.rating}/5</p>
            <p>คงเหลือ: {product.stock} ชิ้น</p>
            <button
              onClick={() => dispatch(addToCart(product))}
              disabled={product.stock === 0}
            >
              เพิ่มลงตะกร้า
            </button>
          </div>
        ))}
      </div>
    </div>
  );
};

export default ProductList;
```

---

## 4. Zustand กับ TypeScript

### 4.1 การติดตั้งและ Store พื้นฐาน

```typescript
// store/useCounterStore.ts
import { create } from 'zustand';

interface CounterStore {
  count: number;
  step: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setStep: (step: number) => void;
  incrementBy: (amount: number) => void;
}

const useCounterStore = create<CounterStore>((set, get) => ({
  count: 0,
  step: 1,
  
  increment: () => set((state) => ({ count: state.count + state.step })),
  decrement: () => set((state) => ({ count: state.count - state.step })),
  reset: () => set({ count: 0 }),
  setStep: (step) => set({ step }),
  incrementBy: (amount) => set((state) => ({ count: state.count + amount })),
}));

export default useCounterStore;
```

### 4.2 Zustand Store ที่ซับซ้อน

```typescript
// store/useProductStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface Product {
  id: string;
  name: string;
  price: number;
  category: string;
  stock: number;
}

interface ProductStore {
  products: Product[];
  selectedIds: Set<string>;
  searchQuery: string;
  sortBy: 'name' | 'price' | 'stock';
  sortOrder: 'asc' | 'desc';
  
  // Actions
  setProducts: (products: Product[]) => void;
  addProduct: (product: Product) => void;
  updateProduct: (id: string, updates: Partial<Product>) => void;
  deleteProduct: (id: string) => void;
  deleteSelected: () => void;
  
  // Selection
  toggleSelect: (id: string) => void;
  selectAll: () => void;
  clearSelection: () => void;
  
  // Filtering & Sorting
  setSearchQuery: (query: string) => void;
  setSortBy: (sortBy: ProductStore['sortBy']) => void;
  setSortOrder: (order: ProductStore['sortOrder']) => void;
  
  // Computed (getters)
  getFilteredProducts: () => Product[];
  getTotalValue: () => number;
}

const useProductStore = create<ProductStore>()(
  devtools(
    immer((set, get) => ({
      products: [],
      selectedIds: new Set(),
      searchQuery: '',
      sortBy: 'name',
      sortOrder: 'asc',

      setProducts: (products) => set((state) => {
        state.products = products;
      }),

      addProduct: (product) => set((state) => {
        state.products.push(product);
      }),

      updateProduct: (id, updates) => set((state) => {
        const index = state.products.findIndex(p => p.id === id);
        if (index !== -1) {
          Object.assign(state.products[index], updates);
        }
      }),

      deleteProduct: (id) => set((state) => {
        state.products = state.products.filter(p => p.id !== id);
        state.selectedIds.delete(id);
      }),

      deleteSelected: () => set((state) => {
        const selectedIds = state.selectedIds;
        state.products = state.products.filter(p => !selectedIds.has(p.id));
        state.selectedIds.clear();
      }),

      toggleSelect: (id) => set((state) => {
        if (state.selectedIds.has(id)) {
          state.selectedIds.delete(id);
        } else {
          state.selectedIds.add(id);
        }
      }),

      selectAll: () => set((state) => {
        state.selectedIds = new Set(state.products.map(p => p.id));
      }),

      clearSelection: () => set((state) => {
        state.selectedIds.clear();
      }),

      setSearchQuery: (query) => set({ searchQuery: query }),
      setSortBy: (sortBy) => set({ sortBy }),
      setSortOrder: (order) => set({ sortOrder: order }),

      getFilteredProducts: () => {
        const { products, searchQuery, sortBy, sortOrder } = get();
        let filtered = products.filter(p =>
          p.name.toLowerCase().includes(searchQuery.toLowerCase())
        );
        filtered.sort((a, b) => {
          const aVal = a[sortBy];
          const bVal = b[sortBy];
          const comparison = typeof aVal === 'string'
            ? aVal.localeCompare(bVal as string)
            : (aVal as number) - (bVal as number);
          return sortOrder === 'asc' ? comparison : -comparison;
        });
        return filtered;
      },

      getTotalValue: () => {
        const { products } = get();
        return products.reduce((sum, p) => sum + p.price * p.stock, 0);
      },
    })),
    { name: 'ProductStore' }
  )
);

export default useProductStore;
```

### 4.3 Zustand กับ Persist Middleware

```typescript
// store/useSettingsStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface UserSettings {
  language: 'th' | 'en' | 'ja';
  theme: 'light' | 'dark' | 'system';
  fontSize: 'small' | 'medium' | 'large';
  notifications: {
    email: boolean;
    push: boolean;
    sms: boolean;
  };
  currency: 'THB' | 'USD' | 'EUR';
  timezone: string;
}

interface SettingsStore {
  settings: UserSettings;
  updateSettings: (updates: Partial<UserSettings>) => void;
  updateNotifications: (updates: Partial<UserSettings['notifications']>) => void;
  resetSettings: () => void;
}

const defaultSettings: UserSettings = {
  language: 'th',
  theme: 'system',
  fontSize: 'medium',
  notifications: {
    email: true,
    push: true,
    sms: false,
  },
  currency: 'THB',
  timezone: 'Asia/Bangkok',
};

const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      settings: defaultSettings,
      
      updateSettings: (updates) =>
        set((state) => ({
          settings: { ...state.settings, ...updates },
        })),
      
      updateNotifications: (updates) =>
        set((state) => ({
          settings: {
            ...state.settings,
            notifications: { ...state.settings.notifications, ...updates },
          },
        })),
      
      resetSettings: () => set({ settings: defaultSettings }),
    }),
    {
      name: 'user-settings',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ settings: state.settings }),
    }
  )
);

export default useSettingsStore;
```

### 4.4 การใช้ Zustand ใน Component

```typescript
import React from 'react';
import useProductStore from '../store/useProductStore';

const ProductManager: React.FC = () => {
  const {
    selectedIds,
    searchQuery,
    sortBy,
    getFilteredProducts,
    getTotalValue,
    toggleSelect,
    selectAll,
    clearSelection,
    deleteSelected,
    setSearchQuery,
    setSortBy,
    addProduct,
  } = useProductStore();

  const products = getFilteredProducts();
  const totalValue = getTotalValue();
  const selectedCount = selectedIds.size;

  const handleAddSample = () => {
    addProduct({
      id: crypto.randomUUID(),
      name: `สินค้าใหม่ ${Date.now()}`,
      price: Math.random() * 1000,
      category: 'ทั่วไป',
      stock: Math.floor(Math.random() * 100),
    });
  };

  return (
    <div>
      <div className="toolbar">
        <input
          type="text"
          placeholder="ค้นหาสินค้า..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
        />
        <select value={sortBy} onChange={(e) => setSortBy(e.target.value as any)}>
          <option value="name">เรียงตามชื่อ</option>
          <option value="price">เรียงตามราคา</option>
          <option value="stock">เรียงตามสต็อก</option>
        </select>
        <button onClick={handleAddSample}>เพิ่มสินค้าตัวอย่าง</button>
        <button onClick={selectAll}>เลือกทั้งหมด</button>
        <button onClick={clearSelection} disabled={selectedCount === 0}>
          ยกเลิกการเลือก
        </button>
        <button onClick={deleteSelected} disabled={selectedCount === 0}>
          ลบที่เลือก ({selectedCount})
        </button>
      </div>

      <p>มูลค่ารวมสินค้า: ฿{totalValue.toFixed(2)}</p>

      <table>
        <thead>
          <tr>
            <th>เลือก</th>
            <th>ชื่อ</th>
            <th>ราคา</th>
            <th>หมวดหมู่</th>
            <th>สต็อก</th>
          </tr>
        </thead>
        <tbody>
          {products.map((product) => (
            <tr key={product.id}>
              <td>
                <input
                  type="checkbox"
                  checked={selectedIds.has(product.id)}
                  onChange={() => toggleSelect(product.id)}
                />
              </td>
              <td>{product.name}</td>
              <td>฿{product.price.toFixed(2)}</td>
              <td>{product.category}</td>
              <td>{product.stock}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default ProductManager;
```

---

## 5. Jotai กับ TypeScript

### 5.1 Atoms พื้นฐาน

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// Primitive atoms
const countAtom = atom<number>(0);
const nameAtom = atom<string>('');
const isLoggedInAtom = atom<boolean>(false);

// Array atom
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

const todosAtom = atom<Todo[]>([]);

// Derived (read-only) atom
const completedTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  return todos.filter(todo => todo.completed);
});

const pendingTodosAtom = atom((get) => {
  const todos = get(todosAtom);
  return todos.filter(todo => !todo.completed);
});

const todoStatsAtom = atom((get) => {
  const todos = get(todosAtom);
  const completed = todos.filter(t => t.completed).length;
  return {
    total: todos.length,
    completed,
    pending: todos.length - completed,
    completionRate: todos.length > 0 ? (completed / todos.length) * 100 : 0,
  };
});

// Write atom
const addTodoAtom = atom(
  null,
  (get, set, text: string) => {
    const newTodo: Todo = {
      id: crypto.randomUUID(),
      text,
      completed: false,
    };
    set(todosAtom, [...get(todosAtom), newTodo]);
  }
);

const toggleTodoAtom = atom(
  null,
  (get, set, id: string) => {
    set(
      todosAtom,
      get(todosAtom).map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  }
);
```

### 5.2 Async Atoms

```typescript
import { atom } from 'jotai';
import { atomWithQuery } from 'jotai-tanstack-query';

interface User {
  id: number;
  name: string;
  email: string;
}

// Atom ที่ต้องการ userId
const userIdAtom = atom<number>(1);

// Async atom ที่ fetch data ตาม userId
const userAtom = atom(async (get) => {
  const userId = get(userIdAtom);
  const response = await fetch(`https://api.example.com/users/${userId}`);
  if (!response.ok) throw new Error('ไม่สามารถโหลดข้อมูลผู้ใช้');
  return response.json() as Promise<User>;
});

// Component ที่ใช้ async atom
import React, { Suspense } from 'react';
import { useAtomValue } from 'jotai';

const UserProfile: React.FC = () => {
  const user = useAtomValue(userAtom); // จะ suspend ระหว่างโหลด
  
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
};

const UserProfileWithSuspense: React.FC = () => {
  return (
    <Suspense fallback={<div>กำลังโหลดข้อมูลผู้ใช้...</div>}>
      <UserProfile />
    </Suspense>
  );
};
```

### 5.3 atomWithStorage

```typescript
import { atomWithStorage } from 'jotai/utils';

// Atom ที่บันทึกลง localStorage อัตโนมัติ
const themeAtom = atomWithStorage<'light' | 'dark'>('theme', 'light');
const languageAtom = atomWithStorage<string>('language', 'th');

interface CartItem {
  productId: string;
  quantity: number;
}

const cartAtom = atomWithStorage<CartItem[]>('cart', []);

// Component
const Settings: React.FC = () => {
  const [theme, setTheme] = useAtom(themeAtom);
  const [language, setLanguage] = useAtom(languageAtom);
  
  return (
    <div>
      <select value={theme} onChange={(e) => setTheme(e.target.value as any)}>
        <option value="light">สว่าง</option>
        <option value="dark">มืด</option>
      </select>
      <select value={language} onChange={(e) => setLanguage(e.target.value)}>
        <option value="th">ภาษาไทย</option>
        <option value="en">English</option>
      </select>
    </div>
  );
};
```

---

## 6. การเปรียบเทียบ Approaches

### 6.1 ตารางเปรียบเทียบ

| คุณสมบัติ | Context API | Redux Toolkit | Zustand | Jotai |
|-----------|------------|---------------|---------|-------|
| ขนาด Bundle | ~0 (built-in) | ~10KB | ~1KB | ~2KB |
| Learning Curve | ต่ำ | สูง | ต่ำ | ต่ำ-กลาง |
| DevTools | ไม่มี | มี (ดีมาก) | มี | มี |
| TypeScript Support | ดี | ดีมาก | ดีมาก | ดีมาก |
| Async Support | Manual | ดี (thunks) | Manual | ดี |
| Re-render Optimization | ต้องระวัง | ดี | ดีมาก | ดีมาก |
| Middleware | ไม่มี | มี | มี | มี |
| เหมาะสำหรับ | State เล็ก | App ใหญ่ | ทุกขนาด | Fine-grained |

### 6.2 เมื่อควรเลือกอะไร

```typescript
// Context API - เหมาะสำหรับ global config ที่ไม่ค่อยเปลี่ยน
// เช่น theme, user preferences, localization
const ThemeContext = createContext<Theme>('light');

// Redux Toolkit - เหมาะสำหรับ app ขนาดใหญ่ที่มีทีมหลายคน
// มี time-travel debugging, middleware ecosystem
const store = configureStore({ reducer: { cart, user, products } });

// Zustand - เหมาะสำหรับ state ระดับ component groups
// API ง่าย ไม่ต้อง boilerplate มาก
const useStore = create<State>()((set) => ({ ... }));

// Jotai - เหมาะสำหรับ fine-grained reactivity
// แต่ละ atom เป็นอิสระ ลด re-renders
const countAtom = atom(0);
```

### 6.3 ตัวอย่าง Feature เดียวกันใน 4 วิธี

```typescript
// ===== Shopping Cart State =====

// 1. Context API
interface CartContextType {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  total: number;
}
const CartContext = createContext<CartContextType>(...);

// 2. Redux Toolkit
const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] as CartItem[] },
  reducers: {
    addItem: (state, action: PayloadAction<CartItem>) => {
      state.items.push(action.payload);
    },
    removeItem: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter(i => i.id !== action.payload);
    },
  },
});

// 3. Zustand
const useCartStore = create<CartStore>()((set, get) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) => set((state) => ({ items: state.items.filter(i => i.id !== id) })),
  get total() { return get().items.reduce((sum, i) => sum + i.price, 0); },
}));

// 4. Jotai
const cartItemsAtom = atom<CartItem[]>([]);
const cartTotalAtom = atom((get) => 
  get(cartItemsAtom).reduce((sum, i) => sum + i.price, 0)
);
const addItemAtom = atom(null, (get, set, item: CartItem) => 
  set(cartItemsAtom, [...get(cartItemsAtom), item])
);
```

---

## 7. Advanced Patterns

### 7.1 State Machine Pattern กับ TypeScript

```typescript
// ใช้ TypeScript เพื่อสร้าง state machine ที่ปลอดภัย
type AuthState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'authenticated'; user: User }
  | { status: 'error'; message: string };

type AuthEvent =
  | { type: 'LOGIN_START' }
  | { type: 'LOGIN_SUCCESS'; user: User }
  | { type: 'LOGIN_FAILURE'; message: string }
  | { type: 'LOGOUT' };

function authReducer(state: AuthState, event: AuthEvent): AuthState {
  switch (state.status) {
    case 'idle':
      if (event.type === 'LOGIN_START') return { status: 'loading' };
      return state;
    
    case 'loading':
      if (event.type === 'LOGIN_SUCCESS') 
        return { status: 'authenticated', user: event.user };
      if (event.type === 'LOGIN_FAILURE')
        return { status: 'error', message: event.message };
      return state;
    
    case 'authenticated':
      if (event.type === 'LOGOUT') return { status: 'idle' };
      return state;
    
    case 'error':
      if (event.type === 'LOGIN_START') return { status: 'loading' };
      return state;
  }
}

// Component
const AuthComponent: React.FC = () => {
  const [state, dispatch] = useReducer(authReducer, { status: 'idle' });

  const handleLogin = async (credentials: { email: string; password: string }) => {
    dispatch({ type: 'LOGIN_START' });
    try {
      const user = await loginApi(credentials);
      dispatch({ type: 'LOGIN_SUCCESS', user });
    } catch (error) {
      dispatch({ type: 'LOGIN_FAILURE', message: error.message });
    }
  };

  switch (state.status) {
    case 'idle':
      return <LoginForm onSubmit={handleLogin} />;
    case 'loading':
      return <LoadingSpinner />;
    case 'authenticated':
      return <Dashboard user={state.user} />;
    case 'error':
      return <ErrorMessage message={state.message} onRetry={() => {/* ... */}} />;
  }
};
```

### 7.2 Optimistic Updates Pattern

```typescript
import { create } from 'zustand';

interface Comment {
  id: string;
  text: string;
  authorId: string;
  postId: string;
  createdAt: string;
  isPending?: boolean;
  error?: string;
}

interface CommentStore {
  comments: Map<string, Comment[]>;
  addComment: (postId: string, text: string, authorId: string) => Promise<void>;
  deleteComment: (postId: string, commentId: string) => Promise<void>;
}

const useCommentStore = create<CommentStore>()((set, get) => ({
  comments: new Map(),
  
  addComment: async (postId, text, authorId) => {
    const tempId = `temp-${Date.now()}`;
    const tempComment: Comment = {
      id: tempId,
      text,
      authorId,
      postId,
      createdAt: new Date().toISOString(),
      isPending: true,
    };

    // Optimistic update
    set((state) => {
      const newComments = new Map(state.comments);
      const existing = newComments.get(postId) ?? [];
      newComments.set(postId, [...existing, tempComment]);
      return { comments: newComments };
    });

    try {
      // จริงๆ call API
      const savedComment = await fetch('/api/comments', {
        method: 'POST',
        body: JSON.stringify({ text, authorId, postId }),
        headers: { 'Content-Type': 'application/json' },
      }).then(r => r.json());

      // Replace temp comment with real one
      set((state) => {
        const newComments = new Map(state.comments);
        const existing = newComments.get(postId) ?? [];
        newComments.set(
          postId,
          existing.map(c => c.id === tempId ? savedComment : c)
        );
        return { comments: newComments };
      });
    } catch (error) {
      // Mark as error
      set((state) => {
        const newComments = new Map(state.comments);
        const existing = newComments.get(postId) ?? [];
        newComments.set(
          postId,
          existing.map(c =>
            c.id === tempId
              ? { ...c, isPending: false, error: 'ส่งความคิดเห็นไม่สำเร็จ' }
              : c
          )
        );
        return { comments: newComments };
      });
    }
  },
  
  deleteComment: async (postId, commentId) => {
    // Optimistic remove
    set((state) => {
      const newComments = new Map(state.comments);
      const existing = newComments.get(postId) ?? [];
      newComments.set(postId, existing.filter(c => c.id !== commentId));
      return { comments: newComments };
    });

    try {
      await fetch(`/api/comments/${commentId}`, { method: 'DELETE' });
    } catch (error) {
      // Rollback - ดึงข้อมูลใหม่
      const response = await fetch(`/api/posts/${postId}/comments`);
      const comments = await response.json();
      set((state) => {
        const newComments = new Map(state.comments);
        newComments.set(postId, comments);
        return { comments: newComments };
      });
    }
  },
}));
```

---

## 8. Best Practices

### 8.1 Selector Composition

```typescript
// การสร้าง selectors แบบ composable
const selectUsers = (state: RootState) => state.users.items;
const selectUserId = (_: RootState, userId: string) => userId;

const selectUserById = createSelector(
  selectUsers,
  selectUserId,
  (users, userId) => users.find(u => u.id === userId)
);

// การใช้งาน
const user = useAppSelector(state => selectUserById(state, '123'));
```

### 8.2 Type Guards สำหรับ Actions

```typescript
// Type guard functions
function isPayloadAction<T>(
  action: unknown
): action is { type: string; payload: T } {
  return (
    typeof action === 'object' &&
    action !== null &&
    'type' in action &&
    'payload' in action
  );
}

// Middleware ที่ใช้ type guard
const loggingMiddleware = (store: any) => (next: any) => (action: unknown) => {
  if (isPayloadAction<unknown>(action)) {
    console.log(`Action: ${action.type}`, action.payload);
  }
  return next(action);
};
```

### 8.3 Testing State Management

```typescript
import { renderHook, act } from '@testing-library/react';
import { useCounterStore } from './useCounterStore';

describe('useCounterStore', () => {
  beforeEach(() => {
    useCounterStore.setState({ count: 0, step: 1 });
  });

  it('ควรเพิ่มค่า count', () => {
    const { result } = renderHook(() => useCounterStore());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('ควรเพิ่มค่า count ตาม step', () => {
    const { result } = renderHook(() => useCounterStore());
    
    act(() => {
      result.current.setStep(5);
      result.current.increment();
    });
    
    expect(result.current.count).toBe(5);
  });

  it('ควร reset ค่าได้', () => {
    const { result } = renderHook(() => useCounterStore());
    
    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });
    
    expect(result.current.count).toBe(0);
  });
});
```

---

## 9. Performance Optimization

### 9.1 การป้องกัน Re-renders ที่ไม่จำเป็น

```typescript
// ใช้ shallow comparison ใน Zustand
import { shallow } from 'zustand/shallow';
import useProductStore from './useProductStore';

// แทนที่จะใช้แบบนี้ (re-render ทุกครั้ง state เปลี่ยน)
// const { products, searchQuery } = useProductStore();

// ใช้แบบนี้แทน (re-render เฉพาะเมื่อ products หรือ searchQuery เปลี่ยน)
const { products, searchQuery } = useProductStore(
  (state) => ({ products: state.products, searchQuery: state.searchQuery }),
  shallow
);

// หรือเลือก field ทีละตัว
const products = useProductStore((state) => state.products);
const searchQuery = useProductStore((state) => state.searchQuery);
```

### 9.2 Splitting Context

```typescript
// แทนที่จะใช้ Context เดียวสำหรับทุกอย่าง
// ให้แยก Context ออกตาม concern

// State context - อัปเดตบ่อย
const CartStateContext = createContext<CartState | undefined>(undefined);

// Actions context - ไม่อัปเดตบ่อย (เพราะ function refs ไม่เปลี่ยน)
const CartActionsContext = createContext<CartActions | undefined>(undefined);

export const CartProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(cartReducer, initialState);
  
  const actions = useMemo<CartActions>(() => ({
    addItem: (item) => dispatch({ type: 'ADD_ITEM', payload: item }),
    removeItem: (id) => dispatch({ type: 'REMOVE_ITEM', payload: id }),
    clearCart: () => dispatch({ type: 'CLEAR' }),
  }), []);

  return (
    <CartActionsContext.Provider value={actions}>
      <CartStateContext.Provider value={state}>
        {children}
      </CartStateContext.Provider>
    </CartActionsContext.Provider>
  );
};

export const useCartState = () => {
  const context = useContext(CartStateContext);
  if (!context) throw new Error('useCartState ต้องใช้ภายใน CartProvider');
  return context;
};

export const useCartActions = () => {
  const context = useContext(CartActionsContext);
  if (!context) throw new Error('useCartActions ต้องใช้ภายใน CartProvider');
  return context;
};
```

---

## 10. สรุป

การเลือก state management solution ขึ้นอยู่กับความต้องการของโปรเจกต์:

- **Context API**: เหมาะสำหรับ global state ง่ายๆ เช่น theme, auth state
- **useReducer**: เหมาะสำหรับ state ที่ซับซ้อนในระดับ component
- **Redux Toolkit**: เหมาะสำหรับ application ขนาดใหญ่ที่ต้องการ predictability สูง
- **Zustand**: เหมาะสำหรับ state management ที่ยืดหยุ่นและง่าย
- **Jotai**: เหมาะสำหรับ fine-grained reactivity

TypeScript ช่วยให้ state management ปลอดภัยมากขึ้นผ่าน:
1. Type-safe actions ด้วย Discriminated Unions
2. Typed selectors
3. Generic types ใน async thunks
4. Compile-time checking ของ state shape

---

*จบบทที่ 28 - State Management ใน React กับ TypeScript*
