# ตอนที่ 27: React Hooks กับ TypeScript

## บทนำ

React Hooks เป็นฟีเจอร์สำคัญที่เปลี่ยนวิธีการเขียน React components TypeScript ช่วยให้การใช้ Hooks มีความปลอดภัยทางประเภทข้อมูลมากขึ้น ในบทนี้เราจะเรียนรู้วิธีการใช้ React Hooks ทุกตัวกับ TypeScript อย่างละเอียด รวมถึงการสร้าง Custom Hooks

---

## 27.1 useState กับ Generics

### พื้นฐาน useState

```typescript
// src/hooks/examples/useState.tsx
import React, { useState } from 'react';

// ===== Type inference =====
// TypeScript สามารถ infer type ได้จากค่าเริ่มต้น

const NumberState: React.FC = () => {
  const [count, setCount] = useState(0); // inferred: number
  const [name, setName] = useState(''); // inferred: string
  const [isOpen, setIsOpen] = useState(false); // inferred: boolean

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(prev => prev + 1)}>+</button>
    </div>
  );
};

// ===== Explicit generic type =====
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

const UserState: React.FC = () => {
  // ต้องระบุ type เพราะค่าเริ่มต้นเป็น null
  const [user, setUser] = useState<User | null>(null);

  // Union type
  const [status, setStatus] = useState<'idle' | 'loading' | 'success' | 'error'>('idle');

  // Array type
  const [items, setItems] = useState<User[]>([]);

  // Object type
  const [filters, setFilters] = useState<{
    search: string;
    role: string | null;
    page: number;
  }>({ search: '', role: null, page: 1 });

  const loadUser = async (id: string) => {
    setStatus('loading');
    try {
      // const data = await fetchUser(id);
      const data: User = { id, name: 'Test', email: 'test@example.com', role: 'user' };
      setUser(data);
      setStatus('success');
    } catch {
      setStatus('error');
    }
  };

  return (
    <div>
      {status === 'loading' && <p>กำลังโหลด...</p>}
      {status === 'error' && <p>เกิดข้อผิดพลาด</p>}
      {user && <p>User: {user.name}</p>}
      <button onClick={() => loadUser('U001')}>โหลด User</button>
    </div>
  );
};

// ===== Lazy initialization =====
const expensiveInitialState = (): number => {
  // การคำนวณที่ใช้เวลามาก
  return Math.floor(Math.random() * 100);
};

const LazyState: React.FC = () => {
  // ส่ง function ให้ useState เพื่อ lazy initialization
  const [value, setValue] = useState<number>(expensiveInitialState);
  return <div>Initial value: {value}</div>;
};

// ===== Complex state updates =====
interface FormState {
  values: Record<string, string>;
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isSubmitting: boolean;
}

const FormState: React.FC = () => {
  const [form, setForm] = useState<FormState>({
    values: {},
    errors: {},
    touched: {},
    isSubmitting: false
  });

  const updateField = (field: string, value: string) => {
    setForm(prev => ({
      ...prev,
      values: { ...prev.values, [field]: value },
      touched: { ...prev.touched, [field]: true }
    }));
  };

  const setError = (field: string, error: string) => {
    setForm(prev => ({
      ...prev,
      errors: { ...prev.errors, [field]: error }
    }));
  };

  return (
    <form>
      <input
        value={form.values['name'] || ''}
        onChange={(e) => updateField('name', e.target.value)}
        placeholder="ชื่อ"
      />
      {form.errors['name'] && <span>{form.errors['name']}</span>}
    </form>
  );
};

export { NumberState, UserState, LazyState, FormState };
```

---

## 27.2 useEffect กับ Cleanup

```typescript
// src/hooks/examples/useEffect.tsx
import React, { useState, useEffect, useRef } from 'react';

// ===== Basic useEffect =====
const DocumentTitle: React.FC<{ title: string }> = ({ title }) => {
  useEffect(() => {
    const prevTitle = document.title;
    document.title = title;

    // Cleanup function
    return () => {
      document.title = prevTitle;
    };
  }, [title]); // dependency array

  return null;
};

// ===== Event listeners =====
const WindowSize: React.FC = () => {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    };

    window.addEventListener('resize', handleResize);

    // Cleanup: ลบ event listener
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // empty array = เรียกแค่ครั้งเดียว

  return (
    <div>
      ขนาดหน้าจอ: {size.width} x {size.height}
    </div>
  );
};

// ===== Data fetching =====
interface Post {
  id: number;
  title: string;
  body: string;
}

const PostDetail: React.FC<{ postId: number }> = ({ postId }) => {
  const [post, setPost] = useState<Post | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    // AbortController สำหรับ cancel fetch
    const controller = new AbortController();
    const signal = controller.signal;

    const fetchPost = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await fetch(
          `https://jsonplaceholder.typicode.com/posts/${postId}`,
          { signal }
        );

        if (!response.ok) {
          throw new Error(`HTTP Error: ${response.status}`);
        }

        const data = await response.json() as Post;

        if (!signal.aborted) {
          setPost(data);
        }
      } catch (err) {
        if (!signal.aborted) {
          setError(err as Error);
        }
      } finally {
        if (!signal.aborted) {
          setLoading(false);
        }
      }
    };

    void fetchPost();

    // Cleanup: abort request เมื่อ component unmount หรือ postId เปลี่ยน
    return () => {
      controller.abort();
    };
  }, [postId]);

  if (loading) return <div>กำลังโหลด...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!post) return null;

  return (
    <article>
      <h2>{post.title}</h2>
      <p>{post.body}</p>
    </article>
  );
};

// ===== Subscription และ cleanup =====
interface EventEmitter {
  on: (event: string, handler: (data: unknown) => void) => void;
  off: (event: string, handler: (data: unknown) => void) => void;
}

const useEventSubscription = (
  emitter: EventEmitter,
  event: string,
  handler: (data: unknown) => void
): void => {
  useEffect(() => {
    emitter.on(event, handler);
    return () => emitter.off(event, handler);
  }, [emitter, event, handler]);
};

// ===== setInterval =====
const Timer: React.FC<{ interval?: number }> = ({ interval = 1000 }) => {
  const [seconds, setSeconds] = useState(0);
  const [running, setRunning] = useState(false);

  useEffect(() => {
    if (!running) return;

    const id = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, interval);

    return () => clearInterval(id);
  }, [running, interval]);

  return (
    <div>
      <p>เวลา: {seconds} วินาที</p>
      <button onClick={() => setRunning(r => !r)}>
        {running ? 'หยุด' : 'เริ่ม'}
      </button>
      <button onClick={() => setSeconds(0)}>รีเซ็ต</button>
    </div>
  );
};

// ===== useEffect กับ async =====
const useAsync = <T,>(
  asyncFn: () => Promise<T>,
  deps: React.DependencyList
): { data: T | null; loading: boolean; error: Error | null } => {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: Error | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    let cancelled = false;

    setState({ data: null, loading: true, error: null });

    asyncFn()
      .then(data => {
        if (!cancelled) setState({ data, loading: false, error: null });
      })
      .catch(error => {
        if (!cancelled) setState({ data: null, loading: false, error: error as Error });
      });

    return () => { cancelled = true; };
  // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps);

  return state;
};

export { DocumentTitle, WindowSize, PostDetail, Timer, useAsync };
```

---

## 27.3 useRef Typing

```typescript
// src/hooks/examples/useRef.tsx
import React, { useRef, useEffect, useState } from 'react';

// ===== Ref to DOM element =====
const FocusInput: React.FC = () => {
  const inputRef = useRef<HTMLInputElement>(null);
  const divRef = useRef<HTMLDivElement>(null);
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    // null check จำเป็น
    inputRef.current?.focus();
  }, []);

  const scrollToDiv = () => {
    divRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  const drawOnCanvas = () => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    ctx.fillStyle = '#3b82f6';
    ctx.fillRect(10, 10, 100, 50);
  };

  return (
    <div>
      <input ref={inputRef} placeholder="จะ focus อัตโนมัติ" />
      <button onClick={scrollToDiv}>เลื่อนไป div</button>
      <div ref={divRef} style={{ marginTop: 100 }}>
        Target div
      </div>
      <canvas ref={canvasRef} width={200} height={100} />
      <button onClick={drawOnCanvas}>วาด</button>
    </div>
  );
};

// ===== Mutable ref (เก็บค่าที่ไม่ทำให้ re-render) =====
const StopWatch: React.FC = () => {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  // เก็บ interval id - ไม่ต้องการ re-render เมื่อเปลี่ยน
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const startTimeRef = useRef<number>(0);

  const start = () => {
    setIsRunning(true);
    startTimeRef.current = Date.now() - time;
    intervalRef.current = setInterval(() => {
      setTime(Date.now() - startTimeRef.current);
    }, 10);
  };

  const stop = () => {
    setIsRunning(false);
    if (intervalRef.current !== null) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };

  const reset = () => {
    stop();
    setTime(0);
  };

  useEffect(() => {
    return () => {
      if (intervalRef.current !== null) {
        clearInterval(intervalRef.current);
      }
    };
  }, []);

  const formatTime = (ms: number): string => {
    const seconds = Math.floor(ms / 1000);
    const milliseconds = Math.floor((ms % 1000) / 10);
    return `${String(seconds).padStart(2, '0')}:${String(milliseconds).padStart(2, '0')}`;
  };

  return (
    <div className="stopwatch">
      <div className="display">{formatTime(time)}</div>
      <div className="controls">
        {isRunning
          ? <button onClick={stop}>หยุด</button>
          : <button onClick={start}>เริ่ม</button>
        }
        <button onClick={reset} disabled={isRunning}>รีเซ็ต</button>
      </div>
    </div>
  );
};

// ===== เก็บค่า previous =====
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

const PreviousValueDemo: React.FC = () => {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>ปัจจุบัน: {count}</p>
      <p>ก่อนหน้า: {prevCount ?? 'ไม่มี'}</p>
      <button onClick={() => setCount(c => c + 1)}>เพิ่ม</button>
    </div>
  );
};

// ===== Callback ref =====
const MeasureElement: React.FC = () => {
  const [height, setHeight] = useState<number | null>(null);

  const measuredRef = (node: HTMLDivElement | null) => {
    if (node !== null) {
      setHeight(node.getBoundingClientRect().height);
    }
  };

  return (
    <div>
      <div ref={measuredRef} className="measured-element">
        <p>วัดความสูงของ element นี้</p>
      </div>
      {height !== null && <p>ความสูง: {height}px</p>}
    </div>
  );
};

export { FocusInput, StopWatch, usePrevious, PreviousValueDemo, MeasureElement };
```

---

## 27.4 useContext กับ TypeScript

```typescript
// src/contexts/ThemeContext.tsx
import React, { createContext, useContext, useState } from 'react';

// ===== Theme Context =====
type Theme = 'light' | 'dark' | 'system';

interface ThemeColors {
  background: string;
  foreground: string;
  primary: string;
  secondary: string;
  border: string;
}

const THEMES: Record<Exclude<Theme, 'system'>, ThemeColors> = {
  light: {
    background: '#ffffff',
    foreground: '#111827',
    primary: '#3b82f6',
    secondary: '#6b7280',
    border: '#e5e7eb'
  },
  dark: {
    background: '#111827',
    foreground: '#f9fafb',
    primary: '#60a5fa',
    secondary: '#9ca3af',
    border: '#374151'
  }
};

interface ThemeContextValue {
  theme: Theme;
  colors: ThemeColors;
  setTheme: (theme: Theme) => void;
  toggleTheme: () => void;
  isDark: boolean;
}

// สร้าง context พร้อม default value
const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

// Provider
interface ThemeProviderProps {
  children: React.ReactNode;
  defaultTheme?: Theme;
}

export const ThemeProvider: React.FC<ThemeProviderProps> = ({
  children,
  defaultTheme = 'light'
}) => {
  const [theme, setTheme] = useState<Theme>(defaultTheme);

  const resolvedTheme: Exclude<Theme, 'system'> =
    theme === 'system'
      ? window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
      : theme;

  const toggleTheme = () => {
    setTheme(current => {
      if (current === 'light') return 'dark';
      if (current === 'dark') return 'light';
      return 'light';
    });
  };

  const value: ThemeContextValue = {
    theme,
    colors: THEMES[resolvedTheme],
    setTheme,
    toggleTheme,
    isDark: resolvedTheme === 'dark'
  };

  return (
    <ThemeContext.Provider value={value}>
      <div
        data-theme={resolvedTheme}
        style={{ backgroundColor: THEMES[resolvedTheme].background }}
      >
        {children}
      </div>
    </ThemeContext.Provider>
  );
};

// Custom hook พร้อม type safety
export const useTheme = (): ThemeContextValue => {
  const context = useContext(ThemeContext);
  if (context === undefined) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};

// ===== Auth Context =====
interface AuthUser {
  id: string;
  name: string;
  email: string;
  token: string;
  roles: string[];
}

interface AuthContextValue {
  user: AuthUser | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  hasRole: (role: string) => boolean;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export const AuthProvider: React.FC<React.PropsWithChildren<{}>> = ({ children }) => {
  const [user, setUser] = useState<AuthUser | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const login = async (email: string, password: string): Promise<void> => {
    setIsLoading(true);
    try {
      // Mock API call
      await new Promise(resolve => setTimeout(resolve, 1000));
      setUser({
        id: 'U001',
        name: 'สมชาย ใจดี',
        email,
        token: 'mock-token',
        roles: ['user']
      });
    } finally {
      setIsLoading(false);
    }
  };

  const logout = () => {
    setUser(null);
  };

  const hasRole = (role: string): boolean => {
    return user?.roles.includes(role) ?? false;
  };

  return (
    <AuthContext.Provider value={{
      user,
      isAuthenticated: user !== null,
      isLoading,
      login,
      logout,
      hasRole
    }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = (): AuthContextValue => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

// ===== การใช้งาน =====
const ThemedButton: React.FC = () => {
  const { colors, toggleTheme, isDark } = useTheme();

  return (
    <button
      onClick={toggleTheme}
      style={{
        backgroundColor: colors.primary,
        color: '#fff',
        padding: '8px 16px',
        border: `1px solid ${colors.border}`
      }}
    >
      เปลี่ยนเป็น {isDark ? 'โหมดสว่าง' : 'โหมดมืด'}
    </button>
  );
};

const UserProfile: React.FC = () => {
  const { user, isAuthenticated, logout } = useAuth();

  if (!isAuthenticated || !user) {
    return <p>กรุณาเข้าสู่ระบบ</p>;
  }

  return (
    <div>
      <p>สวัสดี, {user.name}</p>
      <button onClick={logout}>ออกจากระบบ</button>
    </div>
  );
};
```

---

## 27.5 useReducer กับ Typed Actions

```typescript
// src/hooks/examples/useReducer.tsx
import React, { useReducer, useCallback } from 'react';

// ===== Basic useReducer =====

// Action types
type CounterAction =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' }
  | { type: 'SET'; payload: number }
  | { type: 'INCREMENT_BY'; payload: number };

interface CounterState {
  count: number;
  history: number[];
}

const counterReducer = (state: CounterState, action: CounterAction): CounterState => {
  switch (action.type) {
    case 'INCREMENT':
      return {
        count: state.count + 1,
        history: [...state.history, state.count + 1]
      };
    case 'DECREMENT':
      return {
        count: state.count - 1,
        history: [...state.history, state.count - 1]
      };
    case 'RESET':
      return { count: 0, history: [0] };
    case 'SET':
      return {
        count: action.payload,
        history: [...state.history, action.payload]
      };
    case 'INCREMENT_BY':
      return {
        count: state.count + action.payload,
        history: [...state.history, state.count + action.payload]
      };
    default:
      // exhaustive check
      const _exhaustive: never = action;
      return state;
  }
};

const Counter: React.FC = () => {
  const [state, dispatch] = useReducer(counterReducer, {
    count: 0,
    history: [0]
  });

  return (
    <div>
      <p>Count: {state.count}</p>
      <p>History: {state.history.join(' → ')}</p>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'SET', payload: 100 })}>Set 100</button>
      <button onClick={() => dispatch({ type: 'INCREMENT_BY', payload: 5 })}>+5</button>
    </div>
  );
};

// ===== Complex useReducer สำหรับ Shopping Cart =====

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
  image?: string;
}

interface CartState {
  items: CartItem[];
  discount: number;
  couponCode: string | null;
  isLoading: boolean;
}

type CartAction =
  | { type: 'ADD_ITEM'; payload: Omit<CartItem, 'quantity'> }
  | { type: 'REMOVE_ITEM'; payload: { id: string } }
  | { type: 'UPDATE_QUANTITY'; payload: { id: string; quantity: number } }
  | { type: 'CLEAR_CART' }
  | { type: 'APPLY_COUPON'; payload: { code: string; discount: number } }
  | { type: 'REMOVE_COUPON' }
  | { type: 'SET_LOADING'; payload: boolean };

const cartReducer = (state: CartState, action: CartAction): CartState => {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existingIndex = state.items.findIndex(item => item.id === action.payload.id);
      if (existingIndex >= 0) {
        const newItems = [...state.items];
        newItems[existingIndex] = {
          ...newItems[existingIndex],
          quantity: newItems[existingIndex].quantity + 1
        };
        return { ...state, items: newItems };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }]
      };
    }

    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter(item => item.id !== action.payload.id)
      };

    case 'UPDATE_QUANTITY': {
      if (action.payload.quantity <= 0) {
        return {
          ...state,
          items: state.items.filter(item => item.id !== action.payload.id)
        };
      }
      return {
        ...state,
        items: state.items.map(item =>
          item.id === action.payload.id
            ? { ...item, quantity: action.payload.quantity }
            : item
        )
      };
    }

    case 'CLEAR_CART':
      return { ...state, items: [] };

    case 'APPLY_COUPON':
      return {
        ...state,
        couponCode: action.payload.code,
        discount: action.payload.discount
      };

    case 'REMOVE_COUPON':
      return { ...state, couponCode: null, discount: 0 };

    case 'SET_LOADING':
      return { ...state, isLoading: action.payload };

    default:
      const _exhaustive: never = action;
      return state;
  }
};

const initialCartState: CartState = {
  items: [],
  discount: 0,
  couponCode: null,
  isLoading: false
};

// Selectors
const selectSubtotal = (state: CartState): number =>
  state.items.reduce((sum, item) => sum + item.price * item.quantity, 0);

const selectTotal = (state: CartState): number => {
  const subtotal = selectSubtotal(state);
  return subtotal * (1 - state.discount / 100);
};

const selectItemCount = (state: CartState): number =>
  state.items.reduce((sum, item) => sum + item.quantity, 0);

const ShoppingCart: React.FC = () => {
  const [cart, dispatch] = useReducer(cartReducer, initialCartState);

  const addItem = useCallback((item: Omit<CartItem, 'quantity'>) => {
    dispatch({ type: 'ADD_ITEM', payload: item });
  }, []);

  const removeItem = useCallback((id: string) => {
    dispatch({ type: 'REMOVE_ITEM', payload: { id } });
  }, []);

  const updateQuantity = useCallback((id: string, quantity: number) => {
    dispatch({ type: 'UPDATE_QUANTITY', payload: { id, quantity } });
  }, []);

  const subtotal = selectSubtotal(cart);
  const total = selectTotal(cart);
  const itemCount = selectItemCount(cart);

  return (
    <div className="cart">
      <h2>ตะกร้าสินค้า ({itemCount} รายการ)</h2>
      
      {cart.items.length === 0 ? (
        <p>ตะกร้าว่างเปล่า</p>
      ) : (
        <>
          {cart.items.map(item => (
            <div key={item.id} className="cart-item">
              <span>{item.name}</span>
              <span>฿{item.price.toLocaleString()}</span>
              <div className="quantity-controls">
                <button onClick={() => updateQuantity(item.id, item.quantity - 1)}>-</button>
                <span>{item.quantity}</span>
                <button onClick={() => updateQuantity(item.id, item.quantity + 1)}>+</button>
              </div>
              <button onClick={() => removeItem(item.id)}>ลบ</button>
            </div>
          ))}
          
          <div className="cart-summary">
            <p>ยอดรวม: ฿{subtotal.toLocaleString()}</p>
            {cart.discount > 0 && <p>ส่วนลด: {cart.discount}%</p>}
            <p>ยอดสุทธิ: ฿{total.toLocaleString()}</p>
          </div>
        </>
      )}
    </div>
  );
};

export { Counter, ShoppingCart, cartReducer };
export type { CartState, CartAction };
```

---

## 27.6 useCallback และ useMemo

```typescript
// src/hooks/examples/useCallbackMemo.tsx
import React, { useState, useCallback, useMemo, memo } from 'react';

// ===== useCallback =====

interface ButtonProps {
  label: string;
  onClick: () => void;
}

// memo จะ re-render เมื่อ props เปลี่ยน
const MemoButton = memo<ButtonProps>(({ label, onClick }) => {
  console.log(`Rendering button: ${label}`);
  return <button onClick={onClick}>{label}</button>;
});

MemoButton.displayName = 'MemoButton';

const CallbackExample: React.FC = () => {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // ไม่มี useCallback - function ใหม่ทุก render
  const handleIncrement = () => setCount(c => c + 1);

  // มี useCallback - function เดิมถ้า deps ไม่เปลี่ยน
  const handleIncrementMemo = useCallback(() => {
    setCount(c => c + 1);
  }, []); // empty deps = ไม่ recreate

  // useCallback กับ dependencies
  const handleSetText = useCallback((newText: string) => {
    setText(newText.toUpperCase());
    console.log(`Setting text to: ${newText}`);
  }, []);

  // useCallback กับ arguments
  const handleMultiply = useCallback((multiplier: number) => {
    setCount(c => c * multiplier);
  }, []);

  return (
    <div>
      <p>Count: {count}</p>
      <MemoButton label="เพิ่ม (ไม่มี memo)" onClick={handleIncrement} />
      <MemoButton label="เพิ่ม (มี memo)" onClick={handleIncrementMemo} />
      <button onClick={() => handleMultiply(2)}>คูณ 2</button>
      <input
        value={text}
        onChange={(e) => handleSetText(e.target.value)}
        placeholder="พิมพ์..."
      />
    </div>
  );
};

// ===== useMemo =====

interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
  rating: number;
}

interface FilterState {
  category: string;
  minPrice: number;
  maxPrice: number;
  minRating: number;
  sortBy: 'name' | 'price' | 'rating';
  sortOrder: 'asc' | 'desc';
}

const ProductList: React.FC<{ products: Product[] }> = ({ products }) => {
  const [filter, setFilter] = useState<FilterState>({
    category: '',
    minPrice: 0,
    maxPrice: Infinity,
    minRating: 0,
    sortBy: 'name',
    sortOrder: 'asc'
  });

  // คำนวณเฉพาะเมื่อ products หรือ filter เปลี่ยน
  const filteredProducts = useMemo(() => {
    console.log('Filtering products...');
    
    return products
      .filter(product => {
        if (filter.category && product.category !== filter.category) return false;
        if (product.price < filter.minPrice || product.price > filter.maxPrice) return false;
        if (product.rating < filter.minRating) return false;
        return true;
      })
      .sort((a, b) => {
        const multiplier = filter.sortOrder === 'asc' ? 1 : -1;
        if (filter.sortBy === 'price') return (a.price - b.price) * multiplier;
        if (filter.sortBy === 'rating') return (a.rating - b.rating) * multiplier;
        return a.name.localeCompare(b.name) * multiplier;
      });
  }, [products, filter]);

  // คำนวณ statistics
  const stats = useMemo(() => ({
    count: filteredProducts.length,
    avgPrice: filteredProducts.reduce((sum, p) => sum + p.price, 0) / filteredProducts.length || 0,
    avgRating: filteredProducts.reduce((sum, p) => sum + p.rating, 0) / filteredProducts.length || 0,
    categories: [...new Set(filteredProducts.map(p => p.category))]
  }), [filteredProducts]);

  return (
    <div>
      <div className="stats">
        <p>พบ {stats.count} รายการ</p>
        <p>ราคาเฉลี่ย: ฿{stats.avgPrice.toFixed(2)}</p>
        <p>คะแนนเฉลี่ย: {stats.avgRating.toFixed(1)}</p>
      </div>
      
      <ul>
        {filteredProducts.map(product => (
          <li key={product.id}>
            {product.name} - ฿{product.price} ({product.rating}⭐)
          </li>
        ))}
      </ul>
    </div>
  );
};

export { CallbackExample, ProductList };
```

---

## 27.7 useLayoutEffect

```typescript
// src/hooks/examples/useLayoutEffect.tsx
import React, { useLayoutEffect, useState, useRef } from 'react';

// ===== useLayoutEffect vs useEffect =====
// useLayoutEffect รัน synchronously หลัง DOM mutations
// ใช้เมื่อต้องการวัด DOM หรือ animate ก่อน paint

const Tooltip: React.FC<{
  children: React.ReactNode;
  content: string;
}> = ({ children, content }) => {
  const [tooltipStyle, setTooltipStyle] = useState<React.CSSProperties>({});
  const [isVisible, setIsVisible] = useState(false);
  const triggerRef = useRef<HTMLSpanElement>(null);
  const tooltipRef = useRef<HTMLDivElement>(null);

  // ใช้ useLayoutEffect เพราะต้องการ measure ก่อน paint
  useLayoutEffect(() => {
    if (!isVisible || !triggerRef.current || !tooltipRef.current) return;

    const triggerRect = triggerRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    const viewportWidth = window.innerWidth;

    let left = triggerRect.left + triggerRect.width / 2 - tooltipRect.width / 2;
    let top = triggerRect.top - tooltipRect.height - 8;

    // ป้องกัน tooltip หลุดขอบจอ
    if (left < 8) left = 8;
    if (left + tooltipRect.width > viewportWidth - 8) {
      left = viewportWidth - tooltipRect.width - 8;
    }
    if (top < 8) {
      top = triggerRect.bottom + 8;
    }

    setTooltipStyle({
      position: 'fixed',
      left: `${left}px`,
      top: `${top}px`,
      zIndex: 1000
    });
  }, [isVisible]);

  return (
    <span style={{ position: 'relative', display: 'inline-block' }}>
      <span
        ref={triggerRef}
        onMouseEnter={() => setIsVisible(true)}
        onMouseLeave={() => setIsVisible(false)}
      >
        {children}
      </span>
      {isVisible && (
        <div
          ref={tooltipRef}
          className="tooltip"
          style={tooltipStyle}
        >
          {content}
        </div>
      )}
    </span>
  );
};

// ===== useLayoutEffect สำหรับ animation =====
const AnimatedCounter: React.FC<{ value: number }> = ({ value }) => {
  const elementRef = useRef<HTMLSpanElement>(null);

  useLayoutEffect(() => {
    const element = elementRef.current;
    if (!element) return;

    // เริ่ม animation ทันทีหลัง DOM update
    element.style.transform = 'scale(1.2)';
    element.style.transition = 'transform 0.15s ease';

    const timeout = setTimeout(() => {
      element.style.transform = 'scale(1)';
    }, 150);

    return () => clearTimeout(timeout);
  }, [value]);

  return (
    <span ref={elementRef} className="animated-counter">
      {value}
    </span>
  );
};

export { Tooltip, AnimatedCounter };
```

---

## 27.8 Custom Hooks กับ TypeScript

### useLocalStorage

```typescript
// src/hooks/useLocalStorage.ts
import { useState, useEffect, useCallback } from 'react';

type UseLocalStorageReturn<T> = [T, (value: T | ((prev: T) => T)) => void, () => void];

export function useLocalStorage<T>(
  key: string,
  initialValue: T,
  options?: {
    serialize?: (value: T) => string;
    deserialize?: (value: string) => T;
  }
): UseLocalStorageReturn<T> {
  const serialize = options?.serialize ?? JSON.stringify;
  const deserialize = options?.deserialize ?? JSON.parse;

  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item !== null ? (deserialize(item) as T) : initialValue;
    } catch (error) {
      console.warn(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  const setValue = useCallback(
    (value: T | ((prev: T) => T)) => {
      try {
        const valueToStore = value instanceof Function ? value(storedValue) : value;
        setStoredValue(valueToStore);
        window.localStorage.setItem(key, serialize(valueToStore));
        
        // Dispatch event เพื่อ sync ระหว่าง tabs
        window.dispatchEvent(
          new StorageEvent('storage', {
            key,
            newValue: serialize(valueToStore)
          })
        );
      } catch (error) {
        console.warn(`Error setting localStorage key "${key}":`, error);
      }
    },
    [key, serialize, storedValue]
  );

  const removeValue = useCallback(() => {
    try {
      window.localStorage.removeItem(key);
      setStoredValue(initialValue);
    } catch (error) {
      console.warn(`Error removing localStorage key "${key}":`, error);
    }
  }, [key, initialValue]);

  // Sync กับ localStorage changes จาก tab อื่น
  useEffect(() => {
    const handleStorageChange = (event: StorageEvent) => {
      if (event.key === key && event.newValue !== null) {
        try {
          setStoredValue(deserialize(event.newValue) as T);
        } catch {
          // ignore
        }
      }
    };

    window.addEventListener('storage', handleStorageChange);
    return () => window.removeEventListener('storage', handleStorageChange);
  }, [key, deserialize]);

  return [storedValue, setValue, removeValue];
}

// การใช้งาน
const ThemeToggle: React.FC = () => {
  const [theme, setTheme, clearTheme] = useLocalStorage<'light' | 'dark'>(
    'theme',
    'light'
  );

  return (
    <div>
      <p>Theme: {theme}</p>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        เปลี่ยน Theme
      </button>
      <button onClick={clearTheme}>ล้างค่า</button>
    </div>
  );
};
```

### useFetch

```typescript
// src/hooks/useFetch.ts
import { useState, useEffect, useCallback, useRef } from 'react';

interface FetchOptions extends RequestInit {
  immediate?: boolean;
}

interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

interface UseFetchReturn<T> extends FetchState<T> {
  refetch: () => Promise<void>;
  reset: () => void;
}

export function useFetch<T>(
  url: string | null,
  options: FetchOptions = {}
): UseFetchReturn<T> {
  const { immediate = true, ...fetchOptions } = options;

  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: false,
    error: null
  });

  const abortControllerRef = useRef<AbortController | null>(null);
  const isMountedRef = useRef(true);

  useEffect(() => {
    isMountedRef.current = true;
    return () => {
      isMountedRef.current = false;
      abortControllerRef.current?.abort();
    };
  }, []);

  const fetchData = useCallback(async () => {
    if (!url) return;

    abortControllerRef.current?.abort();
    abortControllerRef.current = new AbortController();

    setState(prev => ({ ...prev, loading: true, error: null }));

    try {
      const response = await fetch(url, {
        ...fetchOptions,
        signal: abortControllerRef.current.signal
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const data = await response.json() as T;

      if (isMountedRef.current) {
        setState({ data, loading: false, error: null });
      }
    } catch (error) {
      if (error instanceof Error && error.name === 'AbortError') return;
      
      if (isMountedRef.current) {
        setState({ data: null, loading: false, error: error as Error });
      }
    }
  }, [url, JSON.stringify(fetchOptions)]);

  useEffect(() => {
    if (immediate) {
      void fetchData();
    }
  }, [fetchData, immediate]);

  const reset = useCallback(() => {
    setState({ data: null, loading: false, error: null });
  }, []);

  return { ...state, refetch: fetchData, reset };
}

// การใช้งาน
interface User {
  id: number;
  name: string;
  email: string;
}

const UserList: React.FC = () => {
  const { data: users, loading, error, refetch } = useFetch<User[]>(
    'https://jsonplaceholder.typicode.com/users'
  );

  if (loading) return <div>กำลังโหลด...</div>;
  if (error) return (
    <div>
      Error: {error.message}
      <button onClick={refetch}>ลองใหม่</button>
    </div>
  );

  return (
    <div>
      <button onClick={refetch}>รีเฟรช</button>
      <ul>
        {users?.map(user => (
          <li key={user.id}>{user.name} - {user.email}</li>
        ))}
      </ul>
    </div>
  );
};
```

### useDebounce

```typescript
// src/hooks/useDebounce.ts
import { useState, useEffect, useCallback, useRef } from 'react';

// ===== useDebounce value =====
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// ===== useDebouncedCallback =====
export function useDebouncedCallback<T extends (...args: Parameters<T>) => ReturnType<T>>(
  callback: T,
  delay: number
): [T, () => void] {
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const callbackRef = useRef(callback);

  // อัปเดต callback ref โดยไม่ทำให้ debounce reset
  useEffect(() => {
    callbackRef.current = callback;
  }, [callback]);

  const debouncedCallback = useCallback(
    (...args: Parameters<T>) => {
      if (timerRef.current !== null) {
        clearTimeout(timerRef.current);
      }
      timerRef.current = setTimeout(() => {
        callbackRef.current(...args);
      }, delay);
    },
    [delay]
  ) as T;

  const cancel = useCallback(() => {
    if (timerRef.current !== null) {
      clearTimeout(timerRef.current);
      timerRef.current = null;
    }
  }, []);

  useEffect(() => {
    return cancel;
  }, [cancel]);

  return [debouncedCallback, cancel];
}

// การใช้งาน
const SearchInput: React.FC = () => {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);

  const { data: results, loading } = useFetch<string[]>(
    debouncedQuery ? `/api/search?q=${debouncedQuery}` : null
  );

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="ค้นหา..."
      />
      {loading && <span>กำลังค้นหา...</span>}
      <ul>
        {results?.map((result, i) => <li key={i}>{result}</li>)}
      </ul>
    </div>
  );
};
```

### useToggle

```typescript
// src/hooks/useToggle.ts
import { useState, useCallback } from 'react';

interface UseToggleReturn {
  value: boolean;
  toggle: () => void;
  setTrue: () => void;
  setFalse: () => void;
  setValue: (value: boolean) => void;
}

export function useToggle(initialValue: boolean = false): UseToggleReturn {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse, setValue };
}

// ===== useMultiToggle =====
export function useMultiToggle<T extends string>(
  options: T[],
  initialValue?: T
): {
  active: T | null;
  toggle: (option: T) => void;
  isActive: (option: T) => boolean;
  clear: () => void;
} {
  const [active, setActive] = useState<T | null>(initialValue ?? null);

  const toggle = useCallback((option: T) => {
    setActive(current => current === option ? null : option);
  }, []);

  const isActive = useCallback((option: T) => active === option, [active]);

  const clear = useCallback(() => setActive(null), []);

  return { active, toggle, isActive, clear };
}

// การใช้งาน
const ToggleExample: React.FC = () => {
  const { value: isOpen, toggle, setTrue: open, setFalse: close } = useToggle(false);
  const { active, toggle: toggleTab, isActive } = useMultiToggle(
    ['tab1', 'tab2', 'tab3'],
    'tab1'
  );

  return (
    <div>
      <button onClick={toggle}>{isOpen ? 'ปิด' : 'เปิด'} Modal</button>
      {isOpen && (
        <div className="modal">
          <p>Modal เปิดอยู่</p>
          <button onClick={close}>ปิด</button>
        </div>
      )}

      <div className="tabs">
        {['tab1', 'tab2', 'tab3'].map(tab => (
          <button
            key={tab}
            className={isActive(tab) ? 'active' : ''}
            onClick={() => toggleTab(tab)}
          >
            {tab}
          </button>
        ))}
      </div>
      <p>Active tab: {active}</p>
    </div>
  );
};
```

### useForm

```typescript
// src/hooks/useForm.ts
import { useState, useCallback, useEffect } from 'react';

// Validation rules
type ValidationRule<T> = {
  required?: boolean | string;
  minLength?: { value: number; message: string };
  maxLength?: { value: number; message: string };
  min?: { value: number; message: string };
  max?: { value: number; message: string };
  pattern?: { value: RegExp; message: string };
  validate?: (value: T[keyof T], allValues: T) => string | undefined;
};

type ValidationSchema<T> = {
  [K in keyof T]?: ValidationRule<T>;
};

interface FieldState {
  value: string;
  error: string;
  touched: boolean;
  dirty: boolean;
}

type FormFields<T> = {
  [K in keyof T]: FieldState;
};

interface UseFormOptions<T> {
  initialValues: T;
  validationSchema?: ValidationSchema<T>;
  onSubmit?: (values: T) => void | Promise<void>;
  validateOnChange?: boolean;
  validateOnBlur?: boolean;
}

export function useForm<T extends Record<string, string | number | boolean>>({
  initialValues,
  validationSchema = {} as ValidationSchema<T>,
  onSubmit,
  validateOnChange = false,
  validateOnBlur = true
}: UseFormOptions<T>) {
  const createInitialFields = (): FormFields<T> => {
    const fields = {} as FormFields<T>;
    for (const key in initialValues) {
      fields[key] = {
        value: String(initialValues[key]),
        error: '',
        touched: false,
        dirty: false
      };
    }
    return fields;
  };

  const [fields, setFields] = useState<FormFields<T>>(createInitialFields);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [isSubmitted, setIsSubmitted] = useState(false);

  // Validate a single field
  const validateField = useCallback(
    (name: keyof T, value: string): string => {
      const rules = validationSchema[name];
      if (!rules) return '';

      const allValues = Object.fromEntries(
        Object.entries(fields).map(([k, f]) => [k, (f as FieldState).value])
      ) as T;

      if (rules.required && !value.trim()) {
        return typeof rules.required === 'string'
          ? rules.required
          : `${String(name)} จำเป็นต้องกรอก`;
      }

      if (rules.minLength && value.length < rules.minLength.value) {
        return rules.minLength.message;
      }

      if (rules.maxLength && value.length > rules.maxLength.value) {
        return rules.maxLength.message;
      }

      if (rules.pattern && !rules.pattern.value.test(value)) {
        return rules.pattern.message;
      }

      if (rules.validate) {
        const error = rules.validate(value as T[keyof T], allValues);
        if (error) return error;
      }

      return '';
    },
    [validationSchema, fields]
  );

  // Handle field change
  const handleChange = useCallback(
    (name: keyof T, value: string) => {
      setFields(prev => ({
        ...prev,
        [name]: {
          ...prev[name],
          value,
          dirty: true,
          error: validateOnChange ? validateField(name, value) : prev[name].error
        }
      }));
    },
    [validateField, validateOnChange]
  );

  // Handle field blur
  const handleBlur = useCallback(
    (name: keyof T) => {
      setFields(prev => ({
        ...prev,
        [name]: {
          ...prev[name],
          touched: true,
          error: validateOnBlur
            ? validateField(name, prev[name].value)
            : prev[name].error
        }
      }));
    },
    [validateField, validateOnBlur]
  );

  // Validate all fields
  const validateAll = useCallback((): boolean => {
    let isValid = true;
    const newFields = { ...fields };

    for (const key in fields) {
      const error = validateField(key as keyof T, fields[key].value);
      newFields[key] = { ...fields[key], error, touched: true };
      if (error) isValid = false;
    }

    setFields(newFields);
    return isValid;
  }, [fields, validateField]);

  // Handle form submit
  const handleSubmit = useCallback(
    async (e?: React.FormEvent) => {
      e?.preventDefault();
      setIsSubmitted(true);

      const isValid = validateAll();
      if (!isValid || !onSubmit) return;

      setIsSubmitting(true);
      try {
        const values = Object.fromEntries(
          Object.entries(fields).map(([k, f]) => [k, (f as FieldState).value])
        ) as T;
        await onSubmit(values);
      } finally {
        setIsSubmitting(false);
      }
    },
    [fields, validateAll, onSubmit]
  );

  // Reset form
  const reset = useCallback(() => {
    setFields(createInitialFields());
    setIsSubmitting(false);
    setIsSubmitted(false);
  }, []);

  // Get field props สำหรับ HTML input
  const getFieldProps = useCallback(
    (name: keyof T) => ({
      value: fields[name].value,
      onChange: (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) =>
        handleChange(name, e.target.value),
      onBlur: () => handleBlur(name)
    }),
    [fields, handleChange, handleBlur]
  );

  // Computed values
  const values = Object.fromEntries(
    Object.entries(fields).map(([k, f]) => [k, (f as FieldState).value])
  ) as T;

  const errors = Object.fromEntries(
    Object.entries(fields).map(([k, f]) => [k, (f as FieldState).error])
  ) as Record<keyof T, string>;

  const touched = Object.fromEntries(
    Object.entries(fields).map(([k, f]) => [k, (f as FieldState).touched])
  ) as Record<keyof T, boolean>;

  const isValid = Object.values(errors).every(e => !e);
  const isDirty = Object.values(fields).some(f => (f as FieldState).dirty);

  return {
    values,
    errors,
    touched,
    isSubmitting,
    isSubmitted,
    isValid,
    isDirty,
    handleChange,
    handleBlur,
    handleSubmit,
    getFieldProps,
    reset,
    setFieldValue: handleChange
  };
}

// ===== การใช้งาน useForm =====
interface LoginFormValues {
  email: string;
  password: string;
}

const LoginForm: React.FC = () => {
  const {
    values,
    errors,
    touched,
    isSubmitting,
    handleSubmit,
    getFieldProps
  } = useForm<LoginFormValues>({
    initialValues: { email: '', password: '' },
    validationSchema: {
      email: {
        required: 'กรุณากรอกอีเมล',
        pattern: {
          value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
          message: 'รูปแบบอีเมลไม่ถูกต้อง'
        }
      },
      password: {
        required: 'กรุณากรอกรหัสผ่าน',
        minLength: { value: 8, message: 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร' }
      }
    },
    onSubmit: async (values) => {
      console.log('Submitting:', values);
      await new Promise(resolve => setTimeout(resolve, 1000));
      console.log('Submitted!');
    }
  });

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label>อีเมล</label>
        <input type="email" {...getFieldProps('email')} />
        {touched.email && errors.email && (
          <span className="error">{errors.email}</span>
        )}
      </div>

      <div>
        <label>รหัสผ่าน</label>
        <input type="password" {...getFieldProps('password')} />
        {touched.password && errors.password && (
          <span className="error">{errors.password}</span>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'กำลังเข้าสู่ระบบ...' : 'เข้าสู่ระบบ'}
      </button>
    </form>
  );
};

export { LoginForm };
```

---

## 27.9 Custom Hook Patterns รวม

```typescript
// src/hooks/useWindowScroll.ts
import { useState, useEffect } from 'react';

interface ScrollPosition {
  x: number;
  y: number;
}

export function useWindowScroll(): ScrollPosition {
  const [scroll, setScroll] = useState<ScrollPosition>({
    x: window.scrollX,
    y: window.scrollY
  });

  useEffect(() => {
    const handleScroll = () => {
      setScroll({ x: window.scrollX, y: window.scrollY });
    };

    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => window.removeEventListener('scroll', handleScroll);
  }, []);

  return scroll;
}

// src/hooks/useMediaQuery.ts
export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    setMatches(mediaQuery.matches);

    const handler = (event: MediaQueryListEvent) => setMatches(event.matches);
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// src/hooks/useClipboard.ts
interface UseClipboardReturn {
  copy: (text: string) => Promise<void>;
  copied: boolean;
  error: Error | null;
}

export function useClipboard(resetDelay = 2000): UseClipboardReturn {
  const [copied, setCopied] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const copy = async (text: string): Promise<void> => {
    try {
      await navigator.clipboard.writeText(text);
      setCopied(true);
      setError(null);
      setTimeout(() => setCopied(false), resetDelay);
    } catch (err) {
      setError(err as Error);
      setCopied(false);
    }
  };

  return { copy, copied, error };
}

// src/hooks/useOnClickOutside.ts
import { useEffect, RefObject } from 'react';

export function useOnClickOutside<T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
): void {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) return;
      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);

    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}

// ===== การใช้งาน Custom Hooks =====
import React, { useRef, useCallback } from 'react';
// import { useWindowScroll, useMediaQuery, useClipboard, useOnClickOutside } from './hooks';

const CustomHooksDemo: React.FC = () => {
  const { x: scrollX, y: scrollY } = useWindowScroll();
  const isMobile = useMediaQuery('(max-width: 768px)');
  const { copy, copied } = useClipboard();
  
  const dropdownRef = useRef<HTMLDivElement>(null);
  const [isOpen, setIsOpen] = useState(false);
  
  const handleClickOutside = useCallback(() => {
    setIsOpen(false);
  }, []);
  
  useOnClickOutside(dropdownRef, handleClickOutside);

  return (
    <div>
      <p>Scroll: {scrollX}, {scrollY}</p>
      <p>Device: {isMobile ? 'Mobile' : 'Desktop'}</p>
      
      <button onClick={() => copy('คัดลอกข้อความนี้!')}>
        {copied ? '✓ คัดลอกแล้ว' : 'คัดลอก'}
      </button>
      
      <div ref={dropdownRef} className="dropdown">
        <button onClick={() => setIsOpen(v => !v)}>เมนู ▼</button>
        {isOpen && (
          <ul className="dropdown-menu">
            <li>รายการ 1</li>
            <li>รายการ 2</li>
            <li>รายการ 3</li>
          </ul>
        )}
      </div>
    </div>
  );
};

export { CustomHooksDemo };
```

---

## 27.10 Hook Testing Patterns

```typescript
// src/hooks/__tests__/useLocalStorage.test.ts
import { renderHook, act } from '@testing-library/react';
import { useLocalStorage } from '../useLocalStorage';

describe('useLocalStorage', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('ควรคืน initialValue เมื่อยังไม่มีใน localStorage', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));
    expect(result.current[0]).toBe('default');
  });

  it('ควรเก็บค่าใน localStorage', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', ''));

    act(() => {
      result.current[1]('new value');
    });

    expect(result.current[0]).toBe('new value');
    expect(localStorage.getItem('test-key')).toBe('"new value"');
  });

  it('ควรโหลดค่าจาก localStorage', () => {
    localStorage.setItem('existing-key', '"saved value"');
    const { result } = renderHook(() => useLocalStorage('existing-key', 'default'));
    expect(result.current[0]).toBe('saved value');
  });

  it('ควรลบค่าได้', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 'value'));

    act(() => {
      result.current[1]('stored');
    });

    act(() => {
      result.current[2](); // removeValue
    });

    expect(result.current[0]).toBe('value'); // กลับไป initialValue
    expect(localStorage.getItem('test-key')).toBeNull();
  });
});
```

---

## สรุปบทที่ 27

ในบทนี้เราได้เรียนรู้:
- useState กับ generic types และ lazy initialization
- useEffect กับ cleanup functions และ AbortController
- useRef สำหรับ DOM elements และ mutable values
- useContext กับ TypeScript providers
- useReducer กับ typed actions และ exhaustive checks
- useCallback และ useMemo สำหรับ optimization
- useLayoutEffect สำหรับ DOM measurements
- Custom Hooks:
  - useLocalStorage - เก็บข้อมูลใน localStorage
  - useFetch - ดึงข้อมูลจาก API
  - useDebounce - debounce values และ callbacks
  - useToggle - จัดการ boolean state
  - useForm - form management ครบครัน
  - useWindowScroll, useMediaQuery, useClipboard, useOnClickOutside
- Hook testing ด้วย renderHook

**ขั้นตอนต่อไป**: ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ State Management ด้วย Redux Toolkit และ TypeScript
